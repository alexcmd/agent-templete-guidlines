# Abort Signal Cascade Architecture

Claude Code uses a three-level `AbortController` hierarchy to manage cancellation across concurrent tool execution. The critical invariant: a Bash tool error kills sibling processes but does NOT abort the query turn — the model continues with error results. Only user interruption or unhandled errors end the turn.

---

## Controller Hierarchy

```
toolUseContext.abortController          ← message-level (one per query turn)
    │  created by QueryEngine for each submitMessage()
    │  aborted by: user Ctrl+C (reason='interrupt'), unrecoverable errors
    │
    └── siblingAbortController          ← per-StreamingToolExecutor
            │  child of toolUseContext.abortController
            │  aborted by: Bash tool error (reason='sibling_error')
            │  DOES NOT propagate upward to parent
            │
            ├── toolAbortController     ← per-tool execution
            │    │  child of siblingAbortController
            │    │  aborted by: permission dialog rejection
            │    │  DOES propagate to toolUseContext.abortController
            │    │  (see bubble-up rule below)
            │    └── Bash subprocess (listens to toolAbortController.signal)
            │
            └── toolAbortController     ← another concurrent tool
                 └── ...
```

---

## `createChildAbortController` (`src/utils/abortController.ts`)

```typescript
export function createChildAbortController(parent: AbortController): AbortController {
  const child = new AbortController()
  parent.signal.addEventListener(
    'abort',
    () => child.abort(parent.signal.reason),
    { once: true }
  )
  return child
}
```

Parent → child propagation is automatic. Child → parent propagation is **manual** and conditional.

---

## Abort Reasons

| Reason string | Set by | Meaning |
|--------------|--------|---------|
| `'interrupt'` | Query engine on Ctrl+C | User typed new message while tools running |
| `'sibling_error'` | `siblingAbortController.abort()` | Bash tool errored; siblings should stop |
| `'streaming_fallback'` | StreamingToolExecutor.discard() | Streaming retry — discard in-progress results |
| `undefined` / other | Permission rejection | Generic abort; bubbles to turn-level controller |

---

## Sibling Abort: Bash Errors Only

Only Bash tool errors trigger `siblingAbortController.abort()`. Read, WebFetch, and other tools do not:

```typescript
// src/services/tools/StreamingToolExecutor.ts
if (isErrorResult) {
  thisToolErrored = true
  // ONLY Bash errors cancel siblings
  // Bash commands often have implicit dependency chains:
  //   mkdir fails → subsequent commands are pointless
  // Read/WebFetch are independent — one failure shouldn't nuke the rest
  if (tool.block.name === BASH_TOOL_NAME) {
    this.hasErrored = true
    this.erroredToolDescription = this.getToolDescription(tool)
    this.siblingAbortController.abort('sibling_error')
  }
}
```

When `siblingAbortController` fires, queued siblings receive a synthetic error message instead of executing:

```
"Cancelled: parallel tool call bash(rm -rf /tmp/work…) errored"
```

---

## Permission Dialog Rejection Bubble-Up

Permission dialog cancellation aborts the `toolAbortController`. A special listener on `toolAbortController` conditionally bubbles the abort up to the message-level controller:

```typescript
// src/services/tools/StreamingToolExecutor.ts
toolAbortController.signal.addEventListener(
  'abort',
  () => {
    if (
      toolAbortController.signal.reason !== 'sibling_error' &&
      !this.toolUseContext.abortController.signal.aborted &&
      !this.discarded
    ) {
      // Bubble up to message-level controller
      this.toolUseContext.abortController.abort(toolAbortController.signal.reason)
    }
  },
  { once: true }
)
```

The bubble-up is suppressed when:
- Reason is `'sibling_error'` (not a user action)
- Parent already aborted (prevent double-abort)
- Executor is discarded (streaming fallback path)

**Why bubble-up matters:** ExitPlanMode uses "clear context + auto" which needs the query turn to actually abort, not just send a REJECT_MESSAGE to the model. Without bubble-up, the model would receive rejection as a tool result and continue, which was a regression bug (#21056).

---

## Tool Interrupt Behavior

Each tool declares how it should respond when the user interrupts (types a new message):

```typescript
// Tool.ts
type Tool = {
  interruptBehavior?: () => 'cancel' | 'block'
}
```

| Value | Behavior |
|-------|---------|
| `'cancel'` | Tool is aborted immediately when user interrupts |
| `'block'` | Tool continues running; user must wait (default if undefined) |

The executor checks `interruptBehavior` when `reason === 'interrupt'`:

```typescript
private getAbortReason(tool: TrackedTool): 'sibling_error' | 'user_interrupted' | 'streaming_fallback' | null {
  if (this.discarded) return 'streaming_fallback'
  if (this.hasErrored) return 'sibling_error'
  if (this.toolUseContext.abortController.signal.aborted) {
    if (this.toolUseContext.abortController.signal.reason === 'interrupt') {
      // Only cancel tools that opt-in to cancellation
      return this.getToolInterruptBehavior(tool) === 'cancel' ? 'user_interrupted' : null
    }
    return 'user_interrupted'
  }
  return null
}
```

---

## Synthetic Error Messages

When a tool is cancelled, it receives a synthetic error result (not a real tool execution):

| Reason | Message content |
|--------|----------------|
| `sibling_error` | `"Cancelled: parallel tool call {name}({args}) errored"` |
| `user_interrupted` | `REJECT_MESSAGE` (shown as "User rejected tool use" / "User rejected edit") |
| `streaming_fallback` | `"Error: Streaming fallback - tool execution discarded"` |

These synthetic messages are valid `tool_result` blocks with `is_error: true`, sent to the model as if the tool had errored naturally.

---

## Interruptible Tool UI State

```typescript
// StreamingToolExecutor updates this so the REPL can show an "interruptible" indicator
private updateInterruptibleState(): void {
  const executing = this.tools.filter(t => t.status === 'executing')
  this.toolUseContext.setHasInterruptibleToolInProgress?.(
    executing.length > 0 &&
    executing.every(t => this.getToolInterruptBehavior(t) === 'cancel'),
  )
}
```

The indicator is only shown when **all** currently-executing tools are `'cancel'`-safe. If any tool is `'block'`, the UI cannot offer interruption.

---

## Reproducing the Pattern

```python
import asyncio
from dataclasses import dataclass, field
from typing import Optional, Callable

@dataclass
class AbortSignal:
    aborted: bool = False
    reason: str = ""
    _listeners: list = field(default_factory=list)

    def abort(self, reason: str = ""):
        if not self.aborted:
            self.aborted = True
            self.reason = reason
            for cb in self._listeners:
                cb()

    def add_listener(self, cb: Callable, once: bool = True):
        if once:
            def wrapper():
                cb()
                if wrapper in self._listeners:
                    self._listeners.remove(wrapper)
            self._listeners.append(wrapper)
        else:
            self._listeners.append(cb)

class AbortController:
    def __init__(self):
        self.signal = AbortSignal()

    def abort(self, reason: str = ""):
        self.signal.abort(reason)

def create_child_abort_controller(parent: AbortController) -> AbortController:
    child = AbortController()
    parent.signal.add_listener(
        lambda: child.abort(parent.signal.reason), once=True
    )
    return child

class StreamingToolExecutor:
    def __init__(self, tool_use_context):
        self.ctx = tool_use_context
        self.has_errored = False
        # Child of message-level controller — sibling errors abort this
        # but do NOT propagate to parent
        self.sibling_ctrl = create_child_abort_controller(
            tool_use_context.abort_controller
        )
        self.tools = []

    async def execute_tool(self, tool):
        # Per-tool controller — child of sibling controller
        tool_ctrl = create_child_abort_controller(self.sibling_ctrl)

        # Permission rejection bubbles to message-level controller
        def on_tool_abort():
            reason = tool_ctrl.signal.reason
            if (reason != "sibling_error"
                    and not self.ctx.abort_controller.signal.aborted):
                self.ctx.abort_controller.abort(reason)

        tool_ctrl.signal.add_listener(on_tool_abort, once=True)

        try:
            result = await self._run_tool(tool, tool_ctrl)

            if result.is_error and tool.name == "bash":
                # Only bash errors cancel siblings
                self.has_errored = True
                self.sibling_ctrl.abort("sibling_error")

            return result
        except asyncio.CancelledError:
            return self._synthetic_error(tool, self._get_abort_reason(tool))

    def _get_abort_reason(self, tool) -> str:
        if self.has_errored:
            return "sibling_error"
        if self.ctx.abort_controller.signal.aborted:
            reason = self.ctx.abort_controller.signal.reason
            if reason == "interrupt":
                behavior = getattr(tool, "interrupt_behavior", lambda: "block")()
                return "user_interrupted" if behavior == "cancel" else None
            return "user_interrupted"
        return None

    def _synthetic_error(self, tool, reason: str) -> dict:
        if reason == "user_interrupted":
            return {"type": "tool_result", "is_error": True,
                    "content": "User rejected tool use"}
        return {"type": "tool_result", "is_error": True,
                "content": f"Cancelled: parallel tool call {tool.name} errored"}
```

---

## Decision Matrix

| Event | Aborts toolAbortController | Aborts siblingAbortController | Aborts turn-level controller | Ends query turn |
|-------|--------------------------|------------------------------|------------------------------|-----------------|
| Bash tool error | Yes | Yes (reason='sibling_error') | No | No |
| Permission rejection | Yes | No | Yes (bubble-up) | Yes |
| User Ctrl+C | Yes (via parent) | Yes (via parent) | Yes (directly) | Yes |
| Streaming fallback | No (discarded flag) | No | No | No (retry) |
| Non-bash tool error | Yes | No | No | No |

---

*[← Compaction Deep Dive](./18-compaction-deep-dive.md) | [Session Memory Extraction →](./20-session-memory-extraction.md)*
