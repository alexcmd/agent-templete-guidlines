# §22 Token Budget Management

> Universal guide — applies to all agent implementations.

Token budget management covers three concerns: accurate measurement of context size, threshold-based compaction triggers, and output token capping. Claude Code's implementation is battle-tested against 200K-context models at production scale.

---

## §22.1 The Canonical Token Counter

Use the last API response's actual token count plus a rough estimate of messages added since:

```
tokenCount = lastApiUsage(input + cache_creation + cache_read + output)
           + roughEstimate(messages added after last API call)
```

This avoids:
- **Cumulative counting** — double-counts tokens as context grows turn over turn
- **Output-only counting** — `output_tokens` alone doesn't measure full context size
- **Pure estimation** — estimates alone drift significantly for long sessions

### TypeScript Reference (`src/utils/tokens.ts`)

```typescript
export function tokenCountWithEstimation(messages: readonly Message[]): number {
  let i = messages.length - 1
  while (i >= 0) {
    const usage = getTokenUsage(messages[i])
    if (usage) {
      // Walk back to find first sibling of same API response
      // (parallel tool calls split into multiple records with same message.id)
      const responseId = getAssistantMessageId(messages[i])
      if (responseId) {
        let j = i - 1
        while (j >= 0) {
          const priorId = getAssistantMessageId(messages[j])
          if (priorId === responseId) i = j
          else if (priorId !== undefined) break
          j--
        }
      }
      return getTokenCountFromUsage(usage)
           + roughTokenCountEstimationForMessages(messages.slice(i + 1))
    }
    i--
  }
  return roughTokenCountEstimationForMessages(messages)
}

// getTokenCountFromUsage = input + cache_creation + cache_read + output
```

### Python Implementation

```python
def token_count_with_estimation(messages: list) -> int:
    """
    Canonical context size counter.
    Returns last API usage total + rough estimate of messages since that response.
    """
    # Walk backwards for last usage-bearing assistant message
    last_usage_idx = None
    last_usage = None

    for i in range(len(messages) - 1, -1, -1):
        msg = messages[i]
        if msg.get("role") == "assistant" and "usage" in msg:
            usage = msg["usage"]
            if usage.get("input_tokens") is not None:
                last_usage_idx = i
                last_usage = usage
                break

    if last_usage is None:
        return sum(rough_token_estimate(m) for m in messages)

    api_total = (
        last_usage.get("input_tokens", 0)
        + last_usage.get("cache_creation_input_tokens", 0)
        + last_usage.get("cache_read_input_tokens", 0)
        + last_usage.get("output_tokens", 0)
    )
    messages_since = messages[last_usage_idx + 1:]
    estimated = sum(rough_token_estimate(m) for m in messages_since)
    return api_total + estimated

def rough_token_estimate(message: dict) -> int:
    """
    Fast approximation: ~4 chars per token, padded by 4/3.
    Applied to all content blocks: text, tool_use name+input, tool_result content.
    """
    total_chars = 0
    content = message.get("content", "")

    if isinstance(content, str):
        total_chars = len(content)
    elif isinstance(content, list):
        for block in content:
            btype = block.get("type", "")
            if btype == "text":
                total_chars += len(block.get("text", ""))
            elif btype == "tool_use":
                import json
                total_chars += len(block.get("name", ""))
                total_chars += len(json.dumps(block.get("input", {})))
            elif btype == "tool_result":
                result_content = block.get("content", "")
                if isinstance(result_content, str):
                    total_chars += len(result_content)
                elif isinstance(result_content, list):
                    for item in result_content:
                        if item.get("type") == "text":
                            total_chars += len(item.get("text", ""))
                        elif item.get("type") in ("image", "document"):
                            total_chars += 8000  # ~2000 tokens * 4 chars/token
            elif btype == "thinking":
                total_chars += len(block.get("thinking", ""))

    return int(total_chars / 4 * (4 / 3))
```

---

## §22.2 Threshold Arithmetic

All thresholds derive from `effectiveContextWindow`:

```python
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000   # p99.99 of observed compaction output
AUTOCOMPACT_BUFFER_TOKENS     = 13_000
WARNING_BUFFER_TOKENS         = 20_000
MANUAL_COMPACT_BUFFER_TOKENS  =  3_000

def get_effective_context_window(model_context_window: int, max_output_tokens: int) -> int:
    reserved = min(max_output_tokens, MAX_OUTPUT_TOKENS_FOR_SUMMARY)
    return model_context_window - reserved

def get_thresholds(model_context_window: int, max_output_tokens: int) -> dict:
    effective = get_effective_context_window(model_context_window, max_output_tokens)
    return {
        "effective_window":   effective,
        "autocompact":        effective - AUTOCOMPACT_BUFFER_TOKENS,
        "warning":            effective - AUTOCOMPACT_BUFFER_TOKENS - WARNING_BUFFER_TOKENS,
        "error":              effective - AUTOCOMPACT_BUFFER_TOKENS - WARNING_BUFFER_TOKENS,
        "blocking":           effective - MANUAL_COMPACT_BUFFER_TOKENS,
    }
```

Example for Claude Sonnet (200K context, 64K max output):

```
effectiveContextWindow = 200,000 − min(64,000, 20,000) = 180,000
autoCompactThreshold   = 180,000 − 13,000 = 167,000
warningThreshold       = 167,000 − 20,000 = 147,000
blockingLimit          = 180,000 −  3,000 = 177,000
```

### Warning State Calculator

```python
from dataclasses import dataclass

@dataclass
class TokenWarningState:
    percent_left: int
    above_warning: bool
    above_error: bool
    above_autocompact: bool
    at_blocking_limit: bool

def calculate_token_warning_state(
    token_usage: int,
    model_context_window: int,
    max_output_tokens: int,
    autocompact_enabled: bool = True,
) -> TokenWarningState:
    t = get_thresholds(model_context_window, max_output_tokens)
    threshold = t["autocompact"] if autocompact_enabled else t["effective_window"]

    percent_left = max(0, round((threshold - token_usage) / threshold * 100))

    return TokenWarningState(
        percent_left=percent_left,
        above_warning=token_usage >= t["warning"],
        above_error=token_usage >= t["error"],
        above_autocompact=autocompact_enabled and token_usage >= t["autocompact"],
        at_blocking_limit=token_usage >= t["blocking"],
    )
```

---

## §22.3 Output Token Capping

Claude Code caps `max_tokens` on each API call and escalates on overflow:

```python
DEFAULT_MAX_TOKENS = 16_000    # standard cap for most calls
ESCALATED_MAX_TOKENS = 32_000  # raised after overflow detection

def get_max_tokens(messages: list, model: str, last_max: int) -> int:
    """
    Escalate max_tokens if the last response hit the limit.
    """
    last_response = get_last_assistant_message(messages)
    if last_response:
        stop_reason = last_response.get("stop_reason", "")
        if stop_reason == "max_tokens":
            return ESCALATED_MAX_TOKENS
    return last_max or DEFAULT_MAX_TOKENS
```

The `doesMostRecentAssistantMessageExceed200k` check in `tokens.ts` triggers a model upgrade suggestion when a single response exceeds 200K tokens — indicating the model's context window is probably insufficient.

---

## §22.4 Budget Tracking Across Compaction

Compaction resets the message history, but the model's server-side budget countdown is context-based. After compaction, the remaining budget must account for the pre-compact final window:

```typescript
// src/utils/tokens.ts
export function finalContextTokensFromLastResponse(messages: Message[]): number {
  // Finds last usage with iterations (server tool loops)
  // Falls back to top-level input + output (no server loops)
  // Excludes cache tokens — matches server-side calculate_context_tokens formula
}
```

In Python:

```python
def final_context_tokens_from_last_response(messages: list) -> int:
    """
    Tokens in final server-side context window, for task_budget computation.
    Excludes cache tokens (server formula doesn't count them for budget).
    """
    for msg in reversed(messages):
        if msg.get("role") == "assistant" and "usage" in msg:
            usage = msg["usage"]
            # If server ran tool loops, use last iteration's window
            iterations = usage.get("iterations")
            if iterations and len(iterations) > 0:
                last = iterations[-1]
                return last["input_tokens"] + last["output_tokens"]
            # No server loops — top-level is the final window
            return usage.get("input_tokens", 0) + usage.get("output_tokens", 0)
    return 0
```

---

## §22.5 Circuit Breaker for Compaction

Compaction can fail if context is irrecoverably over limit. Without a circuit breaker, failed attempts repeat every turn. Claude Code observed 250K wasted API calls/day from this pattern:

```python
MAX_CONSECUTIVE_FAILURES = 3

class CompactionCircuitBreaker:
    def __init__(self):
        self.consecutive_failures = 0

    def should_attempt(self) -> bool:
        return self.consecutive_failures < MAX_CONSECUTIVE_FAILURES

    def record_success(self):
        self.consecutive_failures = 0

    def record_failure(self):
        self.consecutive_failures += 1
        if self.consecutive_failures >= MAX_CONSECUTIVE_FAILURES:
            print(f"[compaction] circuit breaker tripped — skipping future attempts")
```

---

## §22.6 Model-Specific Context Windows

| Model | Context Window | Typical max_output |
|-------|---------------|-------------------|
| claude-opus-4-7 | 200,000 | 64,000 |
| claude-sonnet-4-6 | 200,000 | 64,000 |
| claude-haiku-4-5 | 200,000 | 64,000 |

All current Claude models use 200K context. The `effectiveContextWindow` formula reserves 20K for compaction output regardless of `max_output_tokens`, so effective window is always 180K.

---

## §22.7 Cross-Language Quick Reference

### Kotlin (Koog)

```kotlin
data class TokenBudget(
    val contextWindow: Int,
    val maxOutputTokens: Int,
) {
    val reservedForSummary get() = minOf(maxOutputTokens, 20_000)
    val effectiveWindow get() = contextWindow - reservedForSummary
    val autocompactThreshold get() = effectiveWindow - 13_000
    val blockingLimit get() = effectiveWindow - 3_000
}

fun tokenCountWithEstimation(messages: List<Message>): Int {
    val lastUsage = messages.lastOrNull { it.role == "assistant" && it.usage != null }?.usage
        ?: return messages.sumOf { roughEstimate(it) }
    val apiTotal = (lastUsage.inputTokens ?: 0) +
                   (lastUsage.cacheCreationInputTokens ?: 0) +
                   (lastUsage.cacheReadInputTokens ?: 0) +
                   (lastUsage.outputTokens ?: 0)
    val newMessages = messages.dropWhile { it !== messages.last { it.usage != null } }
    return apiTotal + newMessages.sumOf { roughEstimate(it) }
}
```

### C++23

```cpp
struct TokenBudget {
    int context_window;
    int max_output_tokens;

    int reserved_for_summary() const {
        return std::min(max_output_tokens, 20'000);
    }
    int effective_window() const {
        return context_window - reserved_for_summary();
    }
    int autocompact_threshold() const {
        return effective_window() - 13'000;
    }
    int blocking_limit() const {
        return effective_window() - 3'000;
    }
};

// rough estimate: chars / 4 * 4/3
int rough_token_estimate(std::string_view content) {
    return static_cast<int>(content.size() / 4 * 4.0 / 3.0);
}
```

---

## §22.8 Summary: What to Use When

| Need | Function |
|------|---------|
| Check if compaction should trigger | `tokenCountWithEstimation()` |
| Measure context at an API call | `getTokenCountFromUsage(usage)` |
| Fast pre-send estimate | `roughTokenCountEstimation()` |
| Output token budget left | `messageTokenCountFromLastAPIResponse()` |
| Budget after compaction | `finalContextTokensFromLastResponse()` |

---

*See also: [Compaction Deep Dive](docs/18-compaction-deep-dive.md) | [Session Memory Extraction](docs/20-session-memory-extraction.md) | [Agent Loop](docs/05-agent-loop.md)*
