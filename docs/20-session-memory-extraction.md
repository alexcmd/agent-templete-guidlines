# Session Memory Extraction

Claude Code maintains a per-session markdown file that automatically captures key information from the conversation. Extraction runs in a forked subagent without blocking the main conversation, triggered by token and tool-call thresholds rather than wall-clock time.

---

## Architecture

```
Main conversation turn completes
  │
  └─ registerPostSamplingHook(extractSessionMemory)
         │  (fires after every API response, before next turn)
         │
         ├─ querySource !== 'repl_main_thread' → skip (subagents excluded)
         ├─ feature gate 'tengu_session_memory' disabled → skip
         ├─ shouldExtractMemory(messages) → false → skip
         │
         └─ markExtractionStarted()
              │
              ├─ setupSessionMemoryFile(setupContext)
              │    └─ mkdir 0o700, create file 0o600, read current content
              │
              ├─ buildSessionMemoryUpdatePrompt(currentMemory, memoryPath)
              │
              └─ runForkedAgent(...)
                   ├─ querySource: 'session_memory' (deadlock guard)
                   ├─ canUseTool: only FileEditTool on exact memoryPath
                   └─ forkLabel: 'session_memory'
```

The `sequential` wrapper on `extractSessionMemory` ensures only one extraction runs at a time even if multiple turns complete rapidly.

---

## Trigger Conditions (`src/services/SessionMemory/sessionMemory.ts`)

```typescript
export function shouldExtractMemory(messages: Message[]): boolean {
  const currentTokenCount = tokenCountWithEstimation(messages)

  // Initialization: must exceed minimumMessageTokensToInit first
  if (!isSessionMemoryInitialized()) {
    if (!hasMetInitializationThreshold(currentTokenCount)) return false
    markSessionMemoryInitialized()
  }

  // Token threshold: context must have grown by minimumTokensBetweenUpdate
  const hasMetTokenThreshold = hasMetUpdateThreshold(currentTokenCount)

  // Tool call threshold: N tool calls since last extraction
  const toolCallsSince = countToolCallsSince(messages, lastMemoryMessageUuid)
  const hasMetToolCallThreshold = toolCallsSince >= getToolCallsBetweenUpdates()

  // No tool calls in last turn = natural conversation break
  const hasToolCallsInLastTurn = hasToolCallsInLastAssistantTurn(messages)

  // Trigger when:
  // 1. Both token AND tool call thresholds met, OR
  // 2. Token threshold met AND at a natural break (no tool calls in last turn)
  //
  // Token threshold is ALWAYS required — tool call count alone never triggers.
  return (
    (hasMetTokenThreshold && hasMetToolCallThreshold) ||
    (hasMetTokenThreshold && !hasToolCallsInLastTurn)
  )
}
```

**Key invariant**: Token threshold is mandatory. Tool call threshold adds a secondary gate that prevents extraction mid-execution-burst.

---

## Configuration

Config comes from GrowthBook remote config (`tengu_sm_config`), cached non-blocking:

| Field | Default | Description |
|-------|---------|-------------|
| `minimumMessageTokensToInit` | — | Context size before first extraction |
| `minimumTokensBetweenUpdate` | — | Context growth required between extractions |
| `toolCallsBetweenUpdates` | — | Tool calls required between extractions |

Zero values from remote config are ignored (defaults are preserved):

```typescript
const config: SessionMemoryConfig = {
  minimumMessageTokensToInit:
    remoteConfig.minimumMessageTokensToInit > 0
      ? remoteConfig.minimumMessageTokensToInit
      : DEFAULT_SESSION_MEMORY_CONFIG.minimumMessageTokensToInit,
  // ...same pattern for other fields
}
```

---

## Forked Agent Isolation

The extraction agent runs via `runForkedAgent` which creates an isolated context:

```typescript
await runForkedAgent({
  promptMessages: [createUserMessage({ content: userPrompt })],
  cacheSafeParams: createCacheSafeParams(context),
  canUseTool: createMemoryFileCanUseTool(memoryPath),
  querySource: 'session_memory',    // prevents autocompact deadlock
  forkLabel: 'session_memory',
  overrides: { readFileState: setupContext.readFileState },
})
```

The `querySource: 'session_memory'` value makes `shouldAutoCompact` return `false` immediately — a recursion guard preventing the forked extraction agent from trying to compact itself.

The `setupContext` is created via `createSubagentContext(toolUseContext)` which ensures the forked agent's file reads don't pollute the main agent's cache.

---

## Tool Restriction: Memory File Only

The extraction agent can only edit the exact memory file path:

```typescript
export function createMemoryFileCanUseTool(memoryPath: string): CanUseToolFn {
  return async (tool: Tool, input: unknown) => {
    if (
      tool.name === FILE_EDIT_TOOL_NAME &&
      typeof input === 'object' &&
      'file_path' in input &&
      input.file_path === memoryPath       // exact path match
    ) {
      return { behavior: 'allow', updatedInput: input }
    }
    return {
      behavior: 'deny',
      message: `only ${FILE_EDIT_TOOL_NAME} on ${memoryPath} is allowed`,
      decisionReason: { type: 'other', reason: '...' },
    }
  }
}
```

No other tools (Bash, WebFetch, other file paths) are available to the extraction agent.

---

## File Setup

```typescript
// Memory file is created on first extraction
const sessionMemoryDir = getSessionMemoryDir()
await fs.mkdir(sessionMemoryDir, { mode: 0o700 })   // directory: owner-only

const memoryPath = getSessionMemoryPath()
await writeFile(memoryPath, '', {
  encoding: 'utf-8',
  mode: 0o600,          // file: owner-only read/write
  flag: 'wx',           // O_CREAT|O_EXCL — fail if already exists
})
```

On first creation, a template is loaded (`loadSessionMemoryTemplate()`) and written as the initial content. Subsequent extractions read the current content and update it with new information from the conversation.

The file read clears any cached entry (`toolUseContext.readFileState.delete(memoryPath)`) to ensure the forked agent gets the actual current content rather than a dedup stub.

---

## Token Tracking

After each extraction:

```typescript
// Record current token count so the next threshold check can compute delta
recordExtractionTokenCount(tokenCountWithEstimation(messages))

// Only set lastSummarizedMessageId if last turn has no pending tool calls
// (orphaned tool_results would break compaction that relies on this ID)
if (!hasToolCallsInLastAssistantTurn(messages)) {
  const lastMessage = messages[messages.length - 1]
  if (lastMessage?.uuid) {
    setLastSummarizedMessageId(lastMessage.uuid)
  }
}
```

`lastSummarizedMessageId` is used by session memory compaction to know which messages have already been summarized. Setting it when tool calls are pending would create orphaned `tool_result` messages with no matching `tool_use` in the kept history.

---

## Manual Extraction

The `/summary` command bypasses all threshold checks:

```typescript
export async function manuallyExtractSessionMemory(
  messages: Message[],
  toolUseContext: ToolUseContext,
): Promise<ManualExtractionResult> {
  // No shouldExtractMemory() check — always extracts
  markExtractionStarted()
  try {
    const setupContext = createSubagentContext(toolUseContext)
    const { memoryPath, currentMemory } = await setupSessionMemoryFile(setupContext)
    const userPrompt = await buildSessionMemoryUpdatePrompt(currentMemory, memoryPath)
    await runForkedAgent({ ... })
    return { success: true, memoryPath }
  } catch (error) {
    return { success: false, error: errorMessage(error) }
  } finally {
    markExtractionCompleted()
  }
}
```

---

## Initialization Guard

Session memory only initializes when autocompact is enabled:

```typescript
export function initSessionMemory(): void {
  if (getIsRemoteMode()) return
  if (!isAutoCompactEnabled()) return    // session memory feeds compaction; no point without it
  registerPostSamplingHook(extractSessionMemory)
}
```

The gate check (`'tengu_session_memory'` feature flag) is deferred to hook execution time, not initialization time. This avoids blocking startup on GrowthBook initialization.

---

## Reproducing in Python

```python
import asyncio
import os
import time
from pathlib import Path
from typing import Optional, Callable, Awaitable

class SessionMemoryExtractor:
    """
    Background session notes file, updated after model turns based on thresholds.
    """

    def __init__(
        self,
        memory_path: Path,
        model_client,
        min_tokens_to_init: int = 5_000,
        min_tokens_between_updates: int = 3_000,
        tool_calls_between_updates: int = 5,
    ):
        self.memory_path = memory_path
        self.client = model_client
        self.min_tokens_to_init = min_tokens_to_init
        self.min_tokens_between_updates = min_tokens_between_updates
        self.tool_calls_between_updates = tool_calls_between_updates

        self._initialized = False
        self._last_token_count = 0
        self._tool_calls_since_update = 0
        self._extracting = False
        self._lock = asyncio.Lock()

    def should_extract(self, token_count: int, tool_calls_in_last_turn: bool) -> bool:
        if not self._initialized:
            if token_count < self.min_tokens_to_init:
                return False
            self._initialized = True

        token_delta = token_count - self._last_token_count
        met_token = token_delta >= self.min_tokens_between_updates
        met_tool_calls = self._tool_calls_since_update >= self.tool_calls_between_updates

        return met_token and (met_tool_calls or not tool_calls_in_last_turn)

    async def after_turn(self, messages: list, token_count: int):
        """
        Call this after each model turn completes.
        Returns immediately — extraction runs in background.
        """
        tool_calls_last = self._count_tool_calls_last_turn(messages)
        self._tool_calls_since_update += self._count_all_tool_calls(messages)

        if not self.should_extract(token_count, tool_calls_last > 0):
            return

        # Fire and forget — don't await
        asyncio.create_task(self._extract(messages, token_count))

    async def _extract(self, messages: list, token_count: int):
        async with self._lock:  # sequential — only one extraction at a time
            if self._extracting:
                return
            self._extracting = True

        try:
            self.memory_path.parent.mkdir(mode=0o700, parents=True, exist_ok=True)
            current = self._read_current() or self._initial_template()

            prompt = self._build_prompt(current, messages)

            # Run isolated extraction agent
            new_content = await self._run_extraction_agent(prompt, current)

            # Atomic write
            tmp = self.memory_path.with_suffix('.tmp')
            tmp.write_text(new_content, encoding='utf-8')
            tmp.replace(self.memory_path)

            self._last_token_count = token_count
            self._tool_calls_since_update = 0
        finally:
            self._extracting = False

    def _read_current(self) -> Optional[str]:
        if self.memory_path.exists():
            return self.memory_path.read_text(encoding='utf-8')
        return None

    def _initial_template(self) -> str:
        return "# Session Notes\n\n## Key Decisions\n\n## Progress\n\n## Context\n"

    def _build_prompt(self, current: str, messages: list) -> str:
        conversation_summary = self._summarize_messages(messages[-20:])  # last 20
        return (
            f"Current session notes:\n{current}\n\n"
            f"Recent conversation:\n{conversation_summary}\n\n"
            f"Update the session notes to capture important new information. "
            f"Be concise. Preserve existing content unless it's outdated."
        )

    async def _run_extraction_agent(self, prompt: str, current: str) -> str:
        # Restricted agent: can only update the memory file
        response = await self.client.complete(
            system="You are a session notes assistant. Update the notes file with key information.",
            messages=[{"role": "user", "content": prompt}],
        )
        return response.text

    def _count_tool_calls_last_turn(self, messages: list) -> int:
        for msg in reversed(messages):
            if msg.get("role") == "assistant":
                return sum(1 for b in msg.get("content", []) if b.get("type") == "tool_use")
        return 0

    def _count_all_tool_calls(self, messages: list) -> int:
        return sum(
            sum(1 for b in m.get("content", []) if b.get("type") == "tool_use")
            for m in messages if m.get("role") == "assistant"
        )

    def _summarize_messages(self, messages: list) -> str:
        lines = []
        for m in messages:
            role = m.get("role", "?")
            content = m.get("content", "")
            if isinstance(content, list):
                text = " | ".join(
                    b.get("text", b.get("name", "")) for b in content
                    if b.get("type") in ("text", "tool_use")
                )
            else:
                text = str(content)
            lines.append(f"{role}: {text[:200]}")
        return "\n".join(lines)
```

---

## Related Files

| File | Role |
|------|------|
| `src/services/SessionMemory/sessionMemory.ts` | Main extraction logic, `shouldExtractMemory`, `initSessionMemory` |
| `src/services/SessionMemory/sessionMemoryUtils.ts` | Threshold state, `markExtractionStarted/Completed`, config |
| `src/services/SessionMemory/prompts.ts` | Extraction prompt builder, template loader |
| `src/utils/forkedAgent.ts` | `runForkedAgent`, `createSubagentContext`, `createCacheSafeParams` |
| `src/utils/hooks/postSamplingHooks.ts` | `registerPostSamplingHook`, `executePostSamplingHooks` |
| `src/services/compact/sessionMemoryCompact.ts` | Uses session memory for lighter compaction |

---

*[← Abort Cascade](./19-abort-cascade.md) | [Architecture Overview →](./01-architecture-overview.md)*
