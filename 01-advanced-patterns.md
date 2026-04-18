# Universal Agent Architecture — Advanced Loop Patterns & Compaction

> Sections: Advanced Agent Loop Patterns · Self-Improvement Loop · Session Compaction

Part of the [Universal Agent Architecture](01-overview.md) series.

## 12. Advanced Agent Loop Patterns

### 12.1 Concurrent Tool Execution

When the model emits multiple `tool_use` blocks in a single response, run them in
parallel rather than sequentially. This is safe only when tools are read-only or
independently scoped (no shared write targets).

```
# Sequential (simpler, always safe):
for tool_call in tool_calls:
    result = execute(tool_call)
    results.append(result)

# Concurrent (up to 10× faster for read-heavy turns):
results = parallel_map(execute, tool_calls, max_concurrency=10)

# Rules:
CONCURRENT_SAFE = {read_file, glob, grep, web_fetch, read_mcp_resource}
SEQUENTIAL_ONLY  = {write_file, edit_file, bash, mcp_write_*}

# Implementation by language:
# Python:     asyncio.gather(*[execute(t) for t in tool_calls])  (+ semaphore(10))
# TypeScript: pMap(toolCalls, execute, {concurrency: 10})
# Rust:       tokio::task::JoinSet (spawn all, await all)
# Go:         errgroup.Group with semaphore channel(10)
# Kotlin:     coroutineScope { calls.map { async { execute(it) } }.awaitAll() }
```

**Streaming tool executor:** Begin executing read-only tools as soon as their
`tool_use` block is parsed from the stream — before the full model response
completes. This eliminates latency for file-read operations in long responses.

### 12.2 Tool Output Masking

When a session has many turns, large tool results in message history waste context tokens.
Store them externally and replace with placeholders in the context sent to the LLM:

```
MASKING_THRESHOLD = 10_000  # chars

# On each LLM call, before building context:
for i, message in enumerate(history):
    for block in message.content:
        if block.type == "tool_result" and len(block.content) > MASKING_THRESHOLD:
            key = f"masked_{message.id}_{block.tool_use_id}"
            large_result_store[key] = block.content
            block.content = f"[output stored at {key} — {len(block.content)} chars]"

# The actual output is still available for the agent if it needs to re-read it,
# but the LLM sees only the placeholder in its context window.
```

**Complementary — Tool Distillation:** For very long tool outputs, run an LLM call to
compress the output to a summary before injecting into history. Useful for web_fetch
results and large bash stdout.

### 12.3 Model Routing Strategies

A composable routing pipeline applies strategies in precedence order:

```
1. overrideStrategy       → explicit --model CLI flag wins
2. approvalModeStrategy   → permission mode (e.g. yolo) → cheaper fast model
3. localClassifierStrategy→ local classifier model (no API call)
4. cloudClassifierStrategy→ cloud-based classifier
5. numericalClassifier    → token count → simple/complex routing
6. fallbackStrategy       → on quota/overload → cheaper fallback
7. defaultStrategy        → configured model for session

Routing examples:
  Short question (< 100 tokens)  → haiku / flash (cheap + fast)
  Complex multi-step task        → opus / pro (capable)
  YOLO mode (--yolo)             → sonnet (no interactive confirms)
  Overloaded / 529               → fallback model silently
```

**Minimum viable routing** (most agents only need 2 strategies):
```
strategy = overrideStrategy ?? fallbackStrategy ?? defaultStrategy
```

### 12.4 Loop Detection

Without loop detection, an agent can get stuck calling the same tool repeatedly forever.

**Heuristic approach** (fast, no LLM cost):
```python
class LoopDetector:
    def __init__(self, threshold=5):
        self.threshold = threshold
        self.fingerprints = []  # last N turn fingerprints

    def record(self, tool_calls):
        fp = frozenset(
            (t.name, json.dumps(t.input, sort_keys=True))
            for t in tool_calls
        )
        self.fingerprints.append(fp)
        if len(self.fingerprints) > self.threshold * 2:
            self.fingerprints.pop(0)

    def is_looping(self) -> bool:
        if len(self.fingerprints) < self.threshold:
            return False
        recent = self.fingerprints[-self.threshold:]
        return len(set(recent)) == 1  # all identical
```

**LLM-based approach** (more accurate, kicks in after 30 turns):
```
if turn_count > 30 and turn_count % check_interval in (5..15):
    confidence = cheap_model.classify(
        "Is this agent looping? Review the last 10 tool calls.",
        recent_tool_calls
    )
    if confidence >= 0.9:
        raise LoopDetectedError("Agent appears to be looping")
```

**On loop detection:** inject a meta-message into the conversation explaining the
loop was detected, then either abort or ask the user how to proceed.

### 12.5 IDE Context Injection

When integrated with an IDE via a companion extension:

```
First turn:
  IDE → agent: full context JSON {
    open_files: [{path, content, language}],
    active_file: {path, cursor: {line, col}},
    selection: {text, range}
  }

Subsequent turns (delta-only, saves tokens):
  IDE → agent: delta context {
    changed_files: [{path, new_content}],
    active_file: {path, cursor: {line, col}},
    selection: {text, range}  # only if changed
  }
```

This pattern gives the agent real-time IDE state without repeating the full file
contents on every turn. Implement via a WebSocket or stdin/stdout IPC channel between
the IDE extension and the agent process.

### 12.6 Session Branching (Conversation Tree)

For experimentation and "undo" scenarios, implement full tree branching:

```
Session structure:
  session_id: uuid
  parent_id: uuid | null     # null for root session
  branch_at_message: usize   # which message index this branch diverges from
  name: string               # user-given branch name

Storage:
  Sessions share the same project directory.
  Each branch is a separate JSONL file with its own session_id.
  The parent's messages up to branch_at_message are loaded, then the
  branch's messages are appended.

Operations:
  /fork <name>          → copy current session up to now, new session_id
  Ctrl+B (UI)          → interactive branch selector
  --session <id>        → resume a specific branch
  
SQLite index (optional, for fast browsing):
  CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    parent_id TEXT,
    branch_at_message INTEGER,
    name TEXT,
    created_at INTEGER,
    project_hash TEXT
  );
```

---

## 13. Self-Improvement Loop

```
                    ┌────────────────────────────────────────┐
                    │           IMPROVEMENT CYCLE             │
                    │                                         │
                    │  1. Execute task                        │
                    │       ↓                                 │
                    │  2. Evaluate outcome                    │
                    │     QualityGate: LLM-as-judge           │
                    │     score = f(task_success, efficiency) │
                    │       ↓                                 │
                    │  3. If score < threshold:               │
                    │     Reflexion: verbal reflection        │
                    │     SELF-REFINE: generate critique      │
                    │     CRITIC: tool-verify claims          │
                    │       ↓                                 │
                    │  4. Store trajectory + reflection       │
                    │     ExpeL experience pool (vector DB)   │
                    │       ↓                                 │
                    │  5. Distill insights across pool        │
                    │     LLM: "What patterns lead to success?"│
                    │       ↓                                 │
                    │  6. Update SkillLibrary (Voyager)       │
                    │     Reusable subtask procedures         │
                    │       ↓                                 │
                    │  7. Next task benefits from:            │
                    │     - Retrieved similar trajectories    │
                    │     - Distilled success patterns        │
                    │     - Reusable skills                   │
                    └────────────────────────────────────────┘
```

**Paper mapping:**

| Step | Paper | arXiv ID |
|------|-------|----------|
| Verbal reflection | Reflexion | 2303.11366 |
| Generate-critique-refine | SELF-REFINE | selfrefine.info |
| Tool-verified correction | CRITIC | 2310.06825 |
| Experience pool storage | ExpeL | 2308.10144 |
| Skill library | Voyager | (2305.16291) |
| LLM-as-judge evaluation | Constitutional AI / alignment work | — |

---

## 14. Session Compaction

The ToolUse/ToolResult boundary guard is the most critical correctness invariant in session
management. Getting this wrong causes 400 errors from API providers:

```
SAFE compaction boundary:

  messages[0..N-4]:  summarize these
  messages[N-3..N]:  preserve verbatim (last 4)

  BUT: if messages[N-4] is a ToolUse, walk back until the
  boundary starts BEFORE that ToolUse:

  messages[N-5] = AssistantMessage (text)    ← start preserve here
  messages[N-4] = ToolUseBlock               ← must be preserved
  messages[N-3] = ToolResultBlock            ← must be preserved with its ToolUse
  messages[N-2] = AssistantMessage
  messages[N-1] = UserMessage

UNSAFE:
  ✗ Summarize up to and including a ToolUse
  ✗ Preserve a ToolResult whose ToolUse was compacted away
  ✗ Split a multi-tool assistant turn (one tool summarized, another preserved)
```

**Three-Strategy Compaction:**

| Token usage | Strategy | Description |
|-------------|----------|-------------|
| < 75% | None | Normal operation |
| 75–89% | **MicroCompact** | Compact only the oldest messages; preserve last N verbatim. No full API call needed — just trim. |
| ≥ 90% | **Full Compact** | Summarize all but the last 10 messages via a dedicated non-agentic API call. Replace head with `<compact-summary>` synthetic user message. |
| Overflow | **ContextCollapse** | Strip individual content blocks (tool result text, large attachments) from individual messages in-place, keeping metadata. Last-resort before hard failure. |

**Circuit breaker:** after `MAX_CONSECUTIVE_FAILURES = 3` compaction failures in a session,
disable the compactor entirely for that session. Log a warning and let the session run until
the model's native context limit is hit. Prevents infinite retry loops from burning budget.

**Tool-result budget:** independently of compaction, evict the *text content* of oldest tool
results when the combined character count of all tool results exceeds a cap (e.g., 50 000
chars). The tool call metadata (name, id, timestamps) is retained; only the result body is
replaced with `"[result truncated — {N} chars]"`. This prevents a single large bash output
from consuming the entire context before compaction can trigger.

**Compaction trigger strategy:**

| Token usage | Action |
|-------------|--------|
| < 75% | Normal operation |
| 75–89% | MicroCompact (oldest messages only) |
| ≥ 90% | Full Compact (dedicated API call) |
| Overflow | ContextCollapse (in-place body stripping) |
| 3 failures | Disable compactor for session |

**Token warning thresholds (for UI):**
- 80% → yellow warning indicator
- 95% → red critical indicator + prompt user that compaction is imminent

**Compaction prompt structure (universal):**

```
"This {domain} session is being continued from a prior context.
The summary below covers the earlier portion.

Summary:
- Scope: {N} earlier messages compacted (user={u}, assistant={a}, tool={t})
- Tools used: {tool_list}
- Recent user requests: {bullet_list}
- Key artifacts / files: {artifact_list}
- Current work in progress: {wip}

Recent messages are preserved verbatim below.
Continue from where it left off without re-summarizing."
```

---


---

*[← Event-Driven & Multi-Agent](01-event-driven-multiagent.md) | [Next: Config, Infra & Reference →](01-config-infra-reference.md)*
