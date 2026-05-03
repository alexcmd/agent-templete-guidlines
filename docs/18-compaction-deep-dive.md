# Compaction Deep Dive

Claude Code has three distinct compaction strategies with precise thresholds and a circuit breaker. This document covers the full system: threshold arithmetic, microcompact vs. full compaction, cached cache-editing, and time-based clearing.

---

## Compaction Strategies at a Glance

| Strategy | Trigger | What it does | Mutates messages? |
|----------|---------|-------------|-------------------|
| **Cached microcompact** | Per-turn (main thread) | Sends `cache_edits` API blocks to delete old tool results server-side | No (API-layer only) |
| **Time-based microcompact** | Gap since last assistant message | Content-clears old tool results locally | Yes |
| **Session memory compaction** | Same as autocompact threshold | Prunes messages via session memory extraction | Yes |
| **Full autocompact** | Token threshold | Calls API to summarize entire history | Yes (replaces messages) |

---

## Token Threshold Arithmetic (`src/services/compact/autoCompact.ts`)

```
contextWindow              ← model's full context size (e.g. 200,000)
reservedForSummary         ← min(maxOutputTokens, 20_000)
effectiveContextWindow     ← contextWindow − reservedForSummary

autoCompactThreshold       ← effectiveContextWindow − 13_000   [AUTOCOMPACT_BUFFER_TOKENS]
warningThreshold           ← autoCompactThreshold − 20_000     [WARNING_THRESHOLD_BUFFER_TOKENS]
errorThreshold             ← autoCompactThreshold − 20_000     [ERROR_THRESHOLD_BUFFER_TOKENS]
blockingLimit              ← effectiveContextWindow − 3_000    [MANUAL_COMPACT_BUFFER_TOKENS]
```

The `MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000` constant is derived from p99.99 of observed compact summary output lengths.

Override via environment:
```bash
CLAUDE_CODE_AUTO_COMPACT_WINDOW=150000   # cap contextWindow
CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=80       # trigger at 80% of effective window
CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE=180000
DISABLE_COMPACT=1                        # disable all compaction
DISABLE_AUTO_COMPACT=1                   # disable auto only (manual /compact still works)
```

---

## `tokenCountWithEstimation()` — The Canonical Counter

```typescript
// src/utils/tokens.ts
export function tokenCountWithEstimation(messages: readonly Message[]): number {
  // Walk back to find last usage-bearing assistant message
  let i = messages.length - 1
  while (i >= 0) {
    const usage = getTokenUsage(messages[i])
    if (usage) {
      // Walk back further to find first sibling of this API response
      // (parallel tool calls split into multiple assistant records with same message.id)
      const responseId = getAssistantMessageId(messages[i])
      if (responseId) {
        let j = i - 1
        while (j >= 0) {
          const priorId = getAssistantMessageId(messages[j])
          if (priorId === responseId) i = j      // earlier split — anchor here
          else if (priorId !== undefined) break   // different response — stop
          j--
        }
      }
      // API usage (full context at that call) + rough estimate of messages since
      return getTokenCountFromUsage(usage) + roughTokenCountEstimationForMessages(messages.slice(i + 1))
    }
    i--
  }
  // No API response yet — estimate everything
  return roughTokenCountEstimationForMessages(messages)
}

// getTokenCountFromUsage includes ALL token categories:
// input_tokens + cache_creation_input_tokens + cache_read_input_tokens + output_tokens
```

**WARNING:** Do NOT use `messageTokenCountFromLastAPIResponse()` for threshold comparisons — it only counts `output_tokens`, not full context size.

---

## AutoCompact Flow

```
query loop (each turn)
  │
  ├─ shouldAutoCompact(messages, model)?
  │    ├─ querySource === 'session_memory' → false  (deadlock guard)
  │    ├─ querySource === 'compact'        → false  (deadlock guard)
  │    ├─ !isAutoCompactEnabled()          → false
  │    ├─ feature('REACTIVE_COMPACT') && GrowthBook flag → false
  │    └─ tokenCountWithEstimation(messages) >= autoCompactThreshold → true
  │
  └─ autoCompactIfNeeded(messages, toolUseContext, cacheSafeParams)
       │
       ├─ circuit breaker: consecutiveFailures >= 3 → skip
       │
       ├─ trySessionMemoryCompaction(messages, agentId, threshold)
       │    ├─ success → runPostCompactCleanup(), return
       │    └─ null   → fall through
       │
       └─ compactConversation(messages, ...)
            ├─ success → runPostCompactCleanup(), reset consecutiveFailures=0
            └─ failure → consecutiveFailures++, log circuit breaker warning at 3
```

The circuit breaker (`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`) prevents thrashing in sessions where context is irrecoverably over limit — observed to generate 250K wasted API calls/day without it.

---

## Microcompact: Compactable Tools

Only results from these tools are eligible for microcompaction:

```typescript
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,     // file read results
  ...SHELL_TOOL_NAMES,     // bash, shell variants
  GREP_TOOL_NAME,
  GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
])
```

Non-compactable results (agent outputs, permission dialogs, etc.) are always preserved.

---

## Cached Microcompact (Preferred Path)

Cached MC uses the Anthropic **cache editing API** to delete old tool results server-side without invalidating the cached prefix. The local message array is **not mutated**.

```
Turn N: tool_result A (cache_reference)
         tool_result B (cache_reference)
Turn N+1: trigger fires — queue cache_edits block to delete A
Turn N+2: cache_edits block sent → server removes A from cached prefix
           Local messages still show A; server doesn't see it
```

State management (module-level in `microCompact.ts`):
```typescript
let cachedMCState: CachedMCState | null = null   // registered tool IDs, pinned edits
let pendingCacheEdits: CacheEditsBlock | null = null  // queued for next API call

// Called after successful API response
markToolsSentToAPIState()

// Called by API layer before building request
consumePendingCacheEdits() → CacheEditsBlock | null

// Re-sent on every subsequent request (cache coherence)
getPinnedCacheEdits() → PinnedCacheEdits[]
```

**Only runs for `querySource.startsWith('repl_main_thread')`** — subagents, teammates, session_memory forks are excluded to prevent cross-contamination of the shared module state.

---

## Time-Based Microcompact

When the gap since the last assistant message exceeds the threshold, the server's cached prefix has expired. Cached MC's cache-editing assumption is invalid (there's no warm cache to edit), so a different strategy fires: directly clear old tool results in the local message array.

```typescript
// src/services/compact/microCompact.ts
function maybeTimeBasedMicrocompact(messages, querySource) {
  const trigger = evaluateTimeBasedTrigger(messages, querySource)
  if (!trigger) return null

  const { gapMinutes, config } = trigger  // config from GrowthBook: gapThresholdMinutes, keepRecent

  const compactableIds = collectCompactableToolIds(messages)
  const keepSet = new Set(compactableIds.slice(-Math.max(1, config.keepRecent)))
  const clearSet = new Set(compactableIds.filter(id => !keepSet.has(id)))

  // Mutate matching tool_result blocks
  const result = messages.map(message => {
    if (message.type !== 'user') return message
    return {
      ...message,
      message: {
        ...message.message,
        content: message.message.content.map(block =>
          block.type === 'tool_result' && clearSet.has(block.tool_use_id)
            ? { ...block, content: '[Old tool result content cleared]' }
            : block
        )
      }
    }
  })

  resetMicrocompactState()  // cachedMCState is stale after content change
  return { messages: result }
}
```

The `keepRecent` floor of 1 prevents clearing ALL results (edge case where `keepRecent=0` would call `slice(-0)` which returns the full array — an unintended no-op).

---

## Session Memory Compaction (Lighter Alternative)

Before falling back to full `compactConversation`, `autoCompactIfNeeded` tries `trySessionMemoryCompaction`. This uses the session memory file (maintained by the background extraction hook) to prune the message history without calling the API for a full summarization.

On success it:
1. Resets `lastSummarizedMessageId` (pruned messages are gone)
2. Runs `runPostCompactCleanup`
3. Calls `notifyCompaction` to prevent false cache break alerts
4. Returns without calling `compactConversation`

---

## ContentReplacement and Tool Result Budget

Full compaction uses `ContentReplacement` to track what was replaced:

```typescript
// From src/utils/sessionStorage.ts
recordContentReplacement(toolUseId, replacedContent)
```

`applyToolResultBudget` trims the oldest tool results to fit within token limits before sending to the API. References in the message array to trimmed results are replaced with a stub so the API still sees valid `tool_result` blocks.

---

## Implementing Compaction in Your Agent

```python
# Token budget constants (from Claude Code source)
AUTOCOMPACT_BUFFER_TOKENS = 13_000
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000

def get_effective_context_window(model: str, raw_window: int, max_output: int) -> int:
    reserved = min(max_output, MAX_OUTPUT_TOKENS_FOR_SUMMARY)
    return raw_window - reserved

def get_autocompact_threshold(model: str, raw_window: int, max_output: int) -> int:
    return get_effective_context_window(model, raw_window, max_output) - AUTOCOMPACT_BUFFER_TOKENS

def token_count_with_estimation(messages: list, last_usage: dict | None) -> int:
    """Mirrors tokenCountWithEstimation: last API usage + rough estimate of newer messages."""
    if last_usage:
        api_count = (
            last_usage.get("input_tokens", 0)
            + last_usage.get("cache_creation_input_tokens", 0)
            + last_usage.get("cache_read_input_tokens", 0)
            + last_usage.get("output_tokens", 0)
        )
        # Estimate messages added after last API call
        new_messages = get_messages_after_last_response(messages)
        estimated = sum(rough_estimate(m) for m in new_messages)
        return api_count + estimated
    return sum(rough_estimate(m) for m in messages)

def rough_estimate(message: dict) -> int:
    """~4 chars per token, padded by 4/3."""
    text = extract_text_content(message)
    return int(len(text) / 4 * (4 / 3))

class CompactionController:
    MAX_CONSECUTIVE_FAILURES = 3

    def __init__(self):
        self.consecutive_failures = 0

    async def maybe_compact(self, messages: list, model: str, last_usage: dict | None,
                             raw_window: int, max_output: int) -> tuple[bool, list]:
        if self.consecutive_failures >= self.MAX_CONSECUTIVE_FAILURES:
            return False, messages

        token_count = token_count_with_estimation(messages, last_usage)
        threshold = get_autocompact_threshold(model, raw_window, max_output)

        if token_count < threshold:
            return False, messages

        try:
            compacted = await self.compact_conversation(messages)
            self.consecutive_failures = 0
            return True, compacted
        except Exception as e:
            self.consecutive_failures += 1
            if self.consecutive_failures >= self.MAX_CONSECUTIVE_FAILURES:
                print(f"Compaction circuit breaker tripped after {self.consecutive_failures} failures")
            return False, messages

    async def compact_conversation(self, messages: list) -> list:
        # Call model to summarize, return replacement messages
        raise NotImplementedError
```

---

## Related Files

| File | Role |
|------|------|
| `src/services/compact/autoCompact.ts` | Threshold constants, `shouldAutoCompact`, `autoCompactIfNeeded` |
| `src/services/compact/microCompact.ts` | Cached MC, time-based MC, `COMPACTABLE_TOOLS` |
| `src/services/compact/compact.ts` | Full `compactConversation` implementation |
| `src/services/compact/sessionMemoryCompact.ts` | Session memory compaction path |
| `src/services/compact/postCompactCleanup.ts` | Post-compaction cleanup (reset state) |
| `src/utils/tokens.ts` | `tokenCountWithEstimation`, `getTokenCountFromUsage` |
| `src/utils/toolResultStorage.ts` | `applyToolResultBudget`, `ContentReplacement` |

---

*[← Speculative Execution](./17-speculative-execution.md) | [Abort Cascade →](./19-abort-cascade.md)*
