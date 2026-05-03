# Prompt Caching Deep Dive

Claude Code implements a multi-level prompt caching system that reduces API cost and latency by reusing server-side KV cache across turns. This document reverse-engineers the full implementation from source: how cache markers are placed, what scopes and TTLs apply, how usage is tracked, and how cache breaks are detected and diagnosed.

---

## §1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  Per-Request Cache Marker Budget: 4 slots maximum           │
│                                                             │
│  Slot 1-3: System Prompt Blocks (buildSystemPromptBlocks)   │
│    • attribution header    → cacheScope=null  (no marker)   │
│    • system prefix         → cacheScope='org' (marker)      │
│    • static body           → cacheScope='global'/'org'      │
│    • dynamic suffix        → cacheScope=null  (no marker)   │
│                                                             │
│  Slot 4: Last Message (addCacheBreakpoints)                 │
│    • last content block of last message → cache_control     │
│                                                             │
│  HARD LIMIT: Adding a 5th block returns HTTP 400            │
└─────────────────────────────────────────────────────────────┘
```

**Two cache scopes:**

| Scope | TTL | Who can use it | When applied |
|-------|-----|----------------|--------------|
| `org` | 5 min (default) | All providers | Default for all system blocks and message markers |
| `org` + `ttl:'1h'` | 1 hour | Ant users; claude.ai subscribers not in overage (GrowthBook-gated); Bedrock opt-in | Same as org but with extended TTL |
| `global` | 24 hours | First-party (non-foundry) only | Static portion of system prompt only |

**Cost impact:**
- Cache creation: ~10% surcharge on cached tokens
- Cache read: 90% discount (pay ~10% of normal input cost)
- Minimum cacheable prefix: 1,024 tokens (Opus/Sonnet), 2,048 tokens (Haiku)

---

## §2 The `cache_control` Object

`getCacheControl()` in `src/services/api/claude.ts:358` is the single factory for all cache markers:

```typescript
// Source: src/services/api/claude.ts:358-374
export function getCacheControl({
  scope,
  querySource,
}: {
  scope?: CacheScope
  querySource?: QuerySource
} = {}): {
  type: 'ephemeral'
  ttl?: '1h'
  scope?: CacheScope
} {
  return {
    type: 'ephemeral',
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),
    ...(scope === 'global' && { scope }),
  }
}
```

Three possible outputs:

```json
// Standard 5-minute org cache
{"type": "ephemeral"}

// 1-hour org cache (eligible users, GrowthBook-gated)
{"type": "ephemeral", "ttl": "1h"}

// 24-hour global cache (1P only, system prompt static block)
{"type": "ephemeral", "scope": "global"}
```

---

## §3 System Prompt Cache Architecture

`splitSysPromptPrefix()` in `src/utils/api.ts:321` splits the system prompt array into 2–4 blocks with cache scopes. Three modes activate based on provider and MCP tool state:

### Mode A — Global scope (1P, boundary marker found)

Triggered when: `shouldUseGlobalCacheScope()` is true AND `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` exists in the prompt array AND no MCP tools are being rendered (non-deferred).

```
[attribution header]   → cacheScope: null   (billing header, never cached)
[system prefix]        → cacheScope: null   (CLI version string etc, never cached)
[static body]          → cacheScope: 'global'  ← 24h cache
[dynamic suffix]       → cacheScope: null   (date, cwd, git status — changes each turn)
```

The boundary sentinel is a compile-time constant inserted between the static rules/tools section and the dynamic per-request context.

### Mode B — Tool-based fallback (1P + MCP tools present)

Triggered when: `shouldUseGlobalCacheScope()` is true BUT `needsToolBasedCacheMarker` is true (a non-deferred MCP tool is in the tool list). MCP tools are per-user dynamic content that cannot be included in a shared global cache.

```
[attribution header]   → cacheScope: null
[system prefix]        → cacheScope: 'org'   ← 5m/1h cache
[everything else]      → cacheScope: 'org'   ← 5m/1h cache
```

The boundary marker is stripped/ignored in this mode.

### Mode C — Default (3P providers, no boundary marker)

All other cases (Bedrock, Vertex, custom API keys, 1P without boundary):

```
[attribution header]   → cacheScope: null
[system prefix]        → cacheScope: 'org'   ← 5m/1h cache
[rest joined]          → cacheScope: 'org'   ← 5m/1h cache
```

### `buildSystemPromptBlocks()` — applies markers to blocks

```typescript
// Source: src/services/api/claude.ts:3213-3237
export function buildSystemPromptBlocks(
  systemPrompt: SystemPrompt,
  enablePromptCaching: boolean,
  options?: {
    skipGlobalCacheForSystemPrompt?: boolean  // true when MCP tools present
    querySource?: QuerySource
  },
): TextBlockParam[] {
  // IMPORTANT: Do not add any more blocks for caching or you will get a 400
  return splitSysPromptPrefix(systemPrompt, {
    skipGlobalCacheForSystemPrompt: options?.skipGlobalCacheForSystemPrompt,
  }).map(block => ({
    type: 'text' as const,
    text: block.text,
    ...(enablePromptCaching && block.cacheScope !== null && {
      cache_control: getCacheControl({
        scope: block.cacheScope,
        querySource: options?.querySource,
      }),
    }),
  }))
}
```

---

## §4 Message-Level Cache Markers

`addCacheBreakpoints()` in `src/services/api/claude.ts:3063` places exactly **one** cache marker across the entire message array — on the last content block of a single message.

### Why only one marker

The comment in source explains the KV cache page eviction behavior: with two markers the second-to-last position keeps its local-attention pages alive for an extra turn even though nothing will ever resume from there. One marker frees those pages immediately, reducing server-side memory pressure.

### Normal vs fire-and-forget

```typescript
// Source: src/services/api/claude.ts:3089
const markerIndex = skipCacheWrite
  ? messages.length - 2   // fire-and-forget fork: second-to-last
  : messages.length - 1   // normal: last message
```

Fire-and-forget forks (spawned subagents that run once and exit) mark the second-to-last message. Since these agents share a prefix with the parent, the write is a no-op merge on the server — the fork doesn't leave its own tail in the KV cache.

### User message conversion

```typescript
// Source: src/services/api/claude.ts:588-631
export function userMessageToMessageParam(
  message: UserMessage,
  addCache = false,
  enablePromptCaching: boolean,
  querySource?: QuerySource,
): MessageParam {
  if (addCache) {
    // String content: wrap in array so cache_control can be attached
    if (typeof message.message.content === 'string') {
      return {
        role: 'user',
        content: [{
          type: 'text',
          text: message.message.content,
          ...(enablePromptCaching && {
            cache_control: getCacheControl({ querySource }),
          }),
        }],
      }
    }
    // Array content: attach cache_control to last block only
    return {
      role: 'user',
      content: message.message.content.map((block, i) => ({
        ...block,
        ...(i === message.message.content.length - 1
          ? enablePromptCaching
            ? { cache_control: getCacheControl({ querySource }) }
            : {}
          : {}),
      })),
    }
  }
  // No cache marker: clone array to prevent mutation side-effects
  return {
    role: 'user',
    content: Array.isArray(message.message.content)
      ? [...message.message.content]
      : message.message.content,
  }
}
```

### Assistant message conversion

Assistant messages exclude `thinking` and `redacted_thinking` blocks from receiving the cache marker — the marker goes on the last **non-thinking** block:

```typescript
// Source: src/services/api/claude.ts:633-674
// Key predicate: last block AND not thinking AND not redacted_thinking
i === message.message.content.length - 1 &&
_.type !== 'thinking' &&
_.type !== 'redacted_thinking' &&
(feature('CONNECTOR_TEXT') ? !isConnectorTextBlock(_) : true)
```

---

## §5 TTL Decision Logic

### `should1hCacheTTL()` — session-stable per-source TTL

```
Priority order:
1. Bedrock + ENABLE_PROMPT_CACHING_1H_BEDROCK env var → true (no GrowthBook needed)
2. USER_TYPE=ant → eligible=true (Anthropic internal)
3. isClaudeAISubscriber() && !currentLimits.isUsingOverage → eligible=true
4. Otherwise → eligible=false → return false

If eligible:
5. Fetch GrowthBook feature 'tengu_prompt_cache_1h_config' → { allowlist: string[] }
6. Match querySource against allowlist (supports trailing * prefix matching)
   e.g. "repl_main_thread*" matches "repl_main_thread" and "repl_main_thread_compact"
```

Both `userEligible` and `allowlist` are **latched in session state** (`STATE.promptCache1hEligible`, `STATE.promptCache1hAllowlist`) after first evaluation. This prevents mid-session overage flips or GrowthBook disk cache updates from changing the TTL mid-conversation, which would bust the server-side cache (~20K tokens per flip).

### `shouldUseGlobalCacheScope()` — 1P only

Returns true only for first-party Claude.ai users (not Bedrock, not Vertex, not custom API key). Also requires the `prompt_caching_scope` beta header to be present in the request.

### Global cache strategy decision

```typescript
// Source: src/services/api/claude.ts:1207-1229
const useGlobalCacheFeature = shouldUseGlobalCacheScope()
const needsToolBasedCacheMarker =
  useGlobalCacheFeature &&
  filteredTools.some(t => t.isMcp === true && !willDefer(t))

const globalCacheStrategy: GlobalCacheStrategy = useGlobalCacheFeature
  ? needsToolBasedCacheMarker
    ? 'none'           // MCP tools block global — fall back to org
    : 'system_prompt'  // global 24h cache for static system prompt
  : 'none'
```

`PROMPT_CACHING_SCOPE_BETA_HEADER` is pushed into the betas array when global cache is enabled.

---

## §6 Usage Tracking

The API returns cache token counts in `usage`. Claude Code tracks three distinct buckets:

```typescript
// Source: src/services/api/claude.ts:2924-3038
// updateUsage() — per-streaming-event update (only overwrite if value > 0)
cache_creation_input_tokens  // tokens written to cache this turn
cache_read_input_tokens      // tokens read from cache this turn (90% discount)

// Sub-breakdown (for 1h vs 5m analysis):
cache_creation.ephemeral_1h_input_tokens
cache_creation.ephemeral_5m_input_tokens

// When CACHED_MICROCOMPACT feature enabled:
cache_deleted_input_tokens   // KV cache pages deleted via cache_edits
```

`accumulateUsage()` sums all these across multi-turn conversations for session-level cost accounting.

**Important:** `updateUsage` only overwrites cache fields when the new value is `!= null && > 0`. `message_delta` events can send explicit 0 values that should not overwrite the real `message_start` values.

---

## §7 Cache Break Detection

`promptCacheBreakDetection.ts` implements a two-phase monitoring system that warns users when the cache is unexpectedly busted.

### Phase 1 — `recordPromptState(snapshot)` (pre-call)

Records 12 hashed state dimensions before each API call:

| Field | What's hashed | Why tracked |
|-------|---------------|-------------|
| `systemHash` | System blocks, cache_control stripped | Detects content changes |
| `cacheControlHash` | cache_control objects only | Catches scope/TTL flips (global↔org, 1h↔5m) |
| `toolsHash` | All tool schemas, cache_control stripped | Detects tool changes |
| `perToolHashes` | Per-tool schema hash map | Identifies which tool changed |
| `model` | Model string | Model changes bust cache |
| `fastMode` | bool | Fast mode adds/removes beta header |
| `globalCacheStrategy` | `'system_prompt'/'none'` | MCP tool add/remove flips strategy |
| `betas` | Sorted beta header list | Any beta change busts cache |
| `autoModeActive` | bool | Now latched sticky-on, tracked to verify fix |
| `isUsingOverage` | bool | Now latched session-stable, tracked to verify fix |
| `cachedMCEnabled` | bool | Cache editing beta toggle |
| `effortValue` | Resolved effort string | Goes into `output_config` / `effort_override` |
| `extraBodyHash` | `getExtraBodyParams()` hash | `CLAUDE_CODE_EXTRA_BODY` changes |

### Phase 2 — `checkResponseForCacheBreak(querySource, cacheReadTokens, ...)` (post-call)

```
Detection threshold:
  cache_read_tokens < prev_cache_read * 0.95   (>5% drop)
  AND token_drop >= 2,000                        (MIN_CACHE_MISS_TOKENS)

Exclusions:
  • First call (no previous baseline)
  • Haiku model (excluded: different caching behavior)
  • cache_deletion_pending=true (expected drop from cache_edits)
  • TTL expiration (5min or 1h elapsed since last assistant message)
```

Tracked query sources (capped at 10 per session to limit memory growth):
- `repl_main_thread*` (main thread + compact share same tracking state)
- `sdk`
- `agent:custom`, `agent:default`, `agent:builtin`

Untracked sources (speculative, session_memory, prompt_suggestion, etc.) still log metrics to analytics but skip break detection — they're short-lived forks that run 1–3 turns each.

---

## §8 Advanced: Cached Microcompact (`cache_edits`)

When the `CACHED_MICROCOMPACT` feature is enabled and the model supports cache editing, Claude Code can selectively delete previously cached KV pages via `cache_edits` blocks:

### `cache_edits` block structure

```json
{
  "type": "cache_edits",
  "edits": [
    {"type": "delete", "cache_reference": "<tool_use_id>"}
  ]
}
```

These blocks are inserted into the last user message (after tool result blocks) and pinned so they're resent at the same position in future calls.

### `cache_reference` on `tool_result` blocks

When `useCachedMC=true`, `addCacheBreakpoints()` also adds `cache_reference: block.tool_use_id` to every `tool_result` block that precedes the last cache_control marker. This allows the server to know which tool invocations are in the cached prefix so it can evict them on demand.

### `cache_deleted_input_tokens`

The API returns this field when pages are deleted. Claude Code skips break-detection for the turn immediately following a cache deletion (the drop is expected, not a break).

---

## §9 Configuration & Environment Variables

| Variable | Effect |
|----------|--------|
| `DISABLE_PROMPT_CACHING` | Global kill switch — disables all cache_control markers |
| `DISABLE_PROMPT_CACHING_HAIKU` | Disable for the small/fast model only |
| `DISABLE_PROMPT_CACHING_SONNET` | Disable for default Sonnet only |
| `DISABLE_PROMPT_CACHING_OPUS` | Disable for default Opus only |
| `ENABLE_PROMPT_CACHING_1H_BEDROCK` | Enable 1h TTL for Bedrock users (bypasses GrowthBook) |
| `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` | Strips `defer_loading`, `scope`, `ttl` from all requests |
| `USER_TYPE=ant` | Forces 1h TTL eligibility (Anthropic internal) |

GrowthBook feature flag: `tengu_prompt_cache_1h_config` → `{ allowlist: string[] }` with prefix matching (`*` suffix).

---

## §10 Implementation Guide

### Pattern 1 — Static system prompt with cache boundary

The core pattern: everything stable before the boundary gets cached; dynamic per-request context after it does not.

```python
import anthropic

client = anthropic.Anthropic()

STATIC_SYSTEM = """You are a senior software engineer assistant.

[Large static rules, tool documentation, examples — ~5,000+ tokens]
These never change between requests."""

def build_system(dynamic_context: str) -> list[dict]:
    return [
        {
            "type": "text",
            "text": STATIC_SYSTEM,
            "cache_control": {"type": "ephemeral"},  # cache this stable block
        },
        {
            "type": "text",
            "text": dynamic_context,
            # no cache_control — changes each turn, must not be cached
        },
    ]

def add_cache_breakpoint(messages: list[dict]) -> list[dict]:
    """Add a single cache marker to the last message's last content block."""
    if not messages:
        return messages
    msgs = [m.copy() for m in messages]
    last = msgs[-1]
    content = last["content"]
    if isinstance(content, str):
        content = [{"type": "text", "text": content}]
    else:
        content = list(content)
    # Attach cache marker to last block
    last_block = {**content[-1], "cache_control": {"type": "ephemeral"}}
    last["content"] = content[:-1] + [last_block]
    msgs[-1] = last
    return msgs

def chat(history: list[dict], user_input: str, dynamic_ctx: str) -> tuple[str, dict]:
    history.append({"role": "user", "content": user_input})
    messages_with_cache = add_cache_breakpoint(history)

    response = client.beta.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=8096,
        system=build_system(dynamic_ctx),
        messages=messages_with_cache,
        betas=["prompt-caching-2024-07-31"],
    )

    usage = response.usage
    cache_read    = getattr(usage, "cache_read_input_tokens", 0) or 0
    cache_created = getattr(usage, "cache_creation_input_tokens", 0) or 0
    cache_hit = cache_read > 0

    assistant_text = response.content[0].text
    history.append({"role": "assistant", "content": assistant_text})

    return assistant_text, {
        "input_tokens":   usage.input_tokens,
        "output_tokens":  usage.output_tokens,
        "cache_read":     cache_read,
        "cache_creation": cache_created,
        "cache_hit":      cache_hit,
    }

# Usage
history = []
dynamic = f"Current date: 2026-05-03\nWorking dir: /home/deck/projects/myapp"

reply, stats = chat(history, "Explain the auth flow", dynamic)
print(f"Cache hit: {stats['cache_hit']} | read={stats['cache_read']} created={stats['cache_creation']}")

reply2, stats2 = chat(history, "How do I add OAuth?", dynamic)
# Second turn: cache_read > 0, cache_creation = 0 → full cache hit
```

### Pattern 2 — Per-model TTL selection

```python
def get_cache_control(user_eligible: bool, query_source: str = "") -> dict:
    """
    Mirrors Claude Code's getCacheControl() / should1hCacheTTL() logic.
    user_eligible: True if ant user, claude.ai subscriber not in overage,
                   or Bedrock with ENABLE_PROMPT_CACHING_1H_BEDROCK set.
    """
    ALLOWLIST = ["repl_main_thread", "sdk", "agent:"]  # configure via feature flag
    
    use_1h = user_eligible and any(
        query_source.startswith(p.rstrip("*")) if p.endswith("*") else query_source == p
        for p in ALLOWLIST
    )
    
    ctrl = {"type": "ephemeral"}
    if use_1h:
        ctrl["ttl"] = "1h"
    return ctrl
```

### Pattern 3 — Cache usage monitoring

```python
class CacheMonitor:
    """Mirrors Claude Code's promptCacheBreakDetection.ts logic."""
    
    MIN_MISS_TOKENS = 2_000
    MISS_THRESHOLD = 0.95  # >5% drop triggers alert
    
    def __init__(self):
        self.prev_cache_read: int | None = None

    def check(self, cache_read: int, cache_creation: int) -> str | None:
        """Returns a warning string if a cache break is detected."""
        if self.prev_cache_read is None:
            self.prev_cache_read = cache_read
            return None
        
        drop = self.prev_cache_read - cache_read
        if cache_read >= self.prev_cache_read * self.MISS_THRESHOLD:
            self.prev_cache_read = cache_read
            return None
        if drop < self.MIN_MISS_TOKENS:
            self.prev_cache_read = cache_read
            return None
        
        msg = f"Cache break detected: {self.prev_cache_read} → {cache_read} ({drop:,} tokens lost)"
        self.prev_cache_read = cache_read
        return msg

monitor = CacheMonitor()
for turn in conversation:
    _, stats = chat(history, turn, dynamic)
    warning = monitor.check(stats["cache_read"], stats["cache_creation"])
    if warning:
        print(f"WARNING: {warning}")
```

### TypeScript equivalents

```typescript
// System prompt block with cache marker
const systemBlocks: TextBlockParam[] = [
  {
    type: 'text',
    text: STATIC_SYSTEM,
    cache_control: { type: 'ephemeral' },
  },
  {
    type: 'text',
    text: dynamicContext,
    // no cache_control
  },
]

// Add cache breakpoint to last message
function addCacheBreakpoint(messages: MessageParam[]): MessageParam[] {
  if (!messages.length) return messages
  const result = [...messages]
  const last = { ...result[result.length - 1]! }
  const content = Array.isArray(last.content)
    ? [...last.content]
    : [{ type: 'text' as const, text: last.content as string }]
  const lastBlock = { ...content[content.length - 1]!, cache_control: { type: 'ephemeral' as const } }
  last.content = [...content.slice(0, -1), lastBlock]
  result[result.length - 1] = last
  return result
}

// Read cache usage
const { cache_read_input_tokens, cache_creation_input_tokens } = response.usage
const cacheHit = (cache_read_input_tokens ?? 0) > 0
```

---

## §11 Common Pitfalls

| Pitfall | Root cause | Fix |
|---------|-----------|-----|
| HTTP 400 on caching | More than 4 `cache_control` blocks in system | Keep system at 2–3 blocks max; one for message history |
| Cache never hits | Dynamic content included in cached block | Move date/cwd/git status to a block without `cache_control` |
| Cache hits on turn 2 but not turn 3 | 5-minute TTL exceeded | Use `ttl: '1h'` if eligible, or keep turns under 5 minutes |
| Thinking blocks getting cache marker | Forgot `type !== 'thinking'` exclusion | Only add cache_control to the last non-thinking, non-redacted_thinking block |
| Cache break on MCP tool add | Global→org strategy flip | Expected: MCP tools are per-user dynamic; cache restores next turn |
| Mixed 5m/1h TTLs in session | Eligibility/allowlist evaluated mid-session | Latch both values on first call and reuse for entire session |

---

## Related Files

| File | Role |
|------|------|
| `src/services/api/claude.ts` | `getCacheControl()`, `getPromptCachingEnabled()`, `should1hCacheTTL()`, `userMessageToMessageParam()`, `assistantMessageToMessageParam()`, `addCacheBreakpoints()`, `buildSystemPromptBlocks()`, `updateUsage()`, `accumulateUsage()` |
| `src/utils/api.ts` | `splitSysPromptPrefix()` — splits system prompt into cacheable/uncacheable blocks |
| `src/services/api/promptCacheBreakDetection.ts` | `recordPromptState()`, `checkResponseForCacheBreak()` — 2-phase break detection |
| `src/bootstrap/state.ts` | `promptCache1hEligible`, `promptCache1hAllowlist` — session-stable latches |
| `src/services/compact/cachedMicrocompact.ts` | `cache_edits` block insertion for KV cache editing |
| `src/constants/querySource.ts` | `QuerySource` type — determines TTL eligibility per call site |

---

*[← Session Memory Extraction](./20-session-memory-extraction.md) | [Architecture Overview →](./01-architecture-overview.md)*
