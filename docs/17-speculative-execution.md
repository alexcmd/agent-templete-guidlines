# Speculative Execution

Claude Code pre-computes the model's next response while the user is still typing, then reuses the result if the submitted input matches. This section documents the `SpeculationState` machine, mutable-ref optimizations, and how to reproduce the pattern.

---

## Overview

```
User types...
  │
  ├─ promptSuggestion fires (text autocomplete, user_intent/stated_intent)
  │
  └─ speculation starts (full model call in background)
          │
          ├─ User submits matching input → reuse results (timeSavedMs recorded)
          └─ User submits different input → abort speculation, start fresh
```

The speculation system has two related but distinct sub-systems:

| System | Purpose | State field |
|--------|---------|------------|
| `promptSuggestion` | Short text autocomplete shown inline | `AppState.promptSuggestion` |
| `speculation` | Full model pre-computation | `AppState.speculation` |

---

## Core Types (`src/state/AppStateStore.ts`)

```typescript
export type CompletionBoundary =
  | { type: 'complete'; completedAt: number; outputTokens: number }
  | { type: 'bash'; command: string; completedAt: number }
  | { type: 'edit'; toolName: string; filePath: string; completedAt: number }
  | { type: 'denied_tool'; toolName: string; detail: string; completedAt: number }

export type SpeculationResult = {
  messages: Message[]
  boundary: CompletionBoundary | null
  timeSavedMs: number
}

export type SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string
      abort: () => void
      startTime: number
      // Mutable ref — avoids O(n) array spread on every streaming message
      messagesRef: { current: Message[] }
      // Mutable ref — tracks paths written to speculation overlay
      writtenPathsRef: { current: Set<string> }
      boundary: CompletionBoundary | null
      suggestionLength: number
      toolUseCount: number
      isPipelined: boolean
      contextRef: { current: REPLHookContext }
      pipelinedSuggestion?: {
        text: string
        promptId: 'user_intent' | 'stated_intent'
        generationRequestId: string | null
      } | null
    }
```

Session-level tracking:
```typescript
// In AppState
speculation: SpeculationState                   // current run
speculationSessionTimeSavedMs: number           // cumulative savings this session
promptSuggestion: {
  text: string | null
  promptId: 'user_intent' | 'stated_intent' | null
  shownAt: number
  acceptedAt: number
  generationRequestId: string | null
}
```

---

## How It Works

### Phase 1 — Prompt Suggestion (text autocomplete)

When the user starts typing, two prompt IDs are tried:

| `promptId` | Description |
|-----------|-------------|
| `user_intent` | Predicts the full intent behind partial input |
| `stated_intent` | Repeats back what the user is literally typing |

The suggestion appears inline in `PromptInput.tsx`. Accepting it (Tab) sets `acceptedAt` and populates `promptSuggestion.text`.

### Phase 2 — Speculation (full model pre-computation)

A full query is launched with the predicted input before the user submits:

1. A unique `id` (UUID) is assigned to the speculation run
2. `messagesRef` and `writtenPathsRef` are mutable refs (not copied arrays) — critical for streaming performance
3. The model streams responses into `messagesRef.current`
4. `boundary` records what stopped the run (`complete`, `bash`, `edit`, `denied_tool`)
5. `isPipelined` is `true` when a pipelined suggestion has been queued

### Phase 3 — Match or Abort

On user submit:
- **Match**: speculation result is promoted to the real conversation. `timeSavedMs = Date.now() - startTime`
- **No match**: `abort()` is called, speculation is discarded, fresh query starts

### CompletionBoundary

The boundary captures what halted pre-computation:

| Type | Meaning |
|------|---------|
| `complete` | Model finished naturally (no pending tool calls) |
| `bash` | Stopped at a Bash tool call (needs permission) |
| `edit` | Stopped at a file edit tool call (needs permission) |
| `denied_tool` | Tool was denied; model continued past it |

---

## Mutable Ref Optimization

```typescript
// WRONG — O(n) array copy per streaming message
const messages = [...state.speculation.messages, newMessage]

// CORRECT — O(1) mutation through ref
messagesRef.current.push(newMessage)
```

The `messagesRef.current` pattern avoids triggering React re-renders or Zustand re-subscriptions while streaming. The ref is only promoted to the immutable `AppState` tree when speculation completes or is accepted.

The same pattern applies to `writtenPathsRef.current` for tracking file system overlay writes.

---

## Reproducing Speculative Execution

```python
import asyncio
import time
from dataclasses import dataclass, field
from typing import Literal, Optional

@dataclass
class CompletionBoundary:
    type: Literal["complete", "bash", "edit", "denied_tool"]
    completed_at: float
    # For bash: command string; for edit: file_path; for denied_tool: detail
    detail: str = ""

@dataclass
class SpeculationRun:
    id: str
    start_time: float
    messages: list = field(default_factory=list)        # mutable, not copied
    written_paths: set = field(default_factory=set)     # mutable
    boundary: Optional[CompletionBoundary] = None
    tool_use_count: int = 0
    cancelled: bool = False

class SpeculativeExecutor:
    """
    Pre-computes model response while user types.
    Reuses result if submitted input matches the prediction.
    """

    def __init__(self, model_client):
        self.client = model_client
        self.current: Optional[SpeculationRun] = None
        self._task: Optional[asyncio.Task] = None
        self.session_time_saved_ms = 0

    async def start(self, predicted_input: str, context: dict) -> str:
        """Begin speculative computation for predicted_input. Returns run ID."""
        if self._task and not self._task.done():
            self._task.cancel()

        run_id = f"spec-{time.time_ns()}"
        self.current = SpeculationRun(id=run_id, start_time=time.monotonic())
        self._task = asyncio.create_task(
            self._run(self.current, predicted_input, context)
        )
        return run_id

    async def _run(self, run: SpeculationRun, input_text: str, context: dict):
        try:
            async for message in self.client.stream(input_text, context):
                if run.cancelled:
                    return
                run.messages.append(message)         # O(1) mutation
                if message.get("type") == "tool_use":
                    run.tool_use_count += 1
                    run.boundary = CompletionBoundary(
                        type="bash" if "bash" in message.get("name","") else "edit",
                        completed_at=time.monotonic(),
                        detail=str(message.get("input", "")),
                    )
                    return  # stop at permission boundary
            run.boundary = CompletionBoundary(
                type="complete", completed_at=time.monotonic()
            )
        except asyncio.CancelledError:
            run.cancelled = True

    async def accept(self, submitted_input: str, predicted_input: str):
        """
        Try to reuse speculation result.
        Returns (messages, time_saved_ms) on hit, (None, 0) on miss.
        """
        if not self.current or self.current.cancelled:
            return None, 0

        if submitted_input.strip() != predicted_input.strip():
            # Miss — abort speculation
            if self._task and not self._task.done():
                self._task.cancel()
            self.current = None
            return None, 0

        # Wait for completion
        if self._task and not self._task.done():
            await self._task

        if not self.current or self.current.cancelled:
            return None, 0

        time_saved_ms = (time.monotonic() - self.current.start_time) * 1000
        self.session_time_saved_ms += time_saved_ms
        messages = self.current.messages
        self.current = None
        return messages, time_saved_ms
```

---

## Key Invariants

1. Only one speculation run is ever active. Starting a new one cancels the previous.
2. Mutable refs (`messagesRef`, `writtenPathsRef`) are never diffed against prior state during streaming — only read on accept/discard.
3. `timeSavedMs` is wall-clock delta from start to accept; it accounts for partial pre-computation even if the model didn't finish.
4. The speculation boundary (`bash`, `edit`) tells the REPL UI whether to show an inline permission prompt or full tool dialog when promoting the result.
5. File writes during speculation go to an overlay; they are committed or rolled back on accept/discard.

---

## Related Files

| File | Role |
|------|------|
| `src/state/AppStateStore.ts` | `SpeculationState`, `SpeculationResult`, `CompletionBoundary` types |
| `src/services/PromptSuggestion/promptSuggestion.ts` | `shouldEnablePromptSuggestion()`, suggestion generation |
| `src/QueryEngine.ts` | Speculation launch and promote logic |
| `src/components/PromptInput.tsx` | UI integration, Tab to accept |

---

*[← Diagrams](./15-diagrams.md) | [Compaction Deep Dive →](./18-compaction-deep-dive.md)*
