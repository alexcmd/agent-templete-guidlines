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

## §20 — QueryGuard: Concurrent Query Prevention

Prevent re-entrant LLM calls with a synchronous state machine. The generation counter
ensures stale `finally` blocks (from aborted queries) never overwrite fresh state.

```typescript
type QueryState = 'idle' | 'dispatching' | 'running';

class QueryGuard {
  private state: QueryState = 'idle';
  private generation = 0;
  private listeners = new Set<() => void>();

  canStart(): boolean { return this.state === 'idle'; }

  start(): number {
    if (this.state !== 'idle') throw new Error('Query already in progress');
    this.state = 'dispatching';
    this.generation++;
    this.notify();
    return this.generation;
  }

  markRunning(gen: number): void {
    if (gen !== this.generation) return;  // stale — ignore
    this.state = 'running';
    this.notify();
  }

  end(gen: number): void {
    if (gen !== this.generation) return;  // stale — ignore (prevents double-idle)
    this.state = 'idle';
    this.notify();
  }

  // Forcibly end + increment generation (invalidates all in-flight finally blocks)
  forceEnd(): void {
    this.generation++;
    this.state = 'idle';
    this.notify();
  }

  // React useSyncExternalStore compatibility
  subscribe(fn: () => void): () => void {
    this.listeners.add(fn);
    return () => this.listeners.delete(fn);
  }
  getSnapshot(): QueryState { return this.state; }

  private notify(): void { for (const fn of this.listeners) fn(); }
}

// Usage in agent loop
async function runQuery(query: string, guard: QueryGuard): Promise<void> {
  if (!guard.canStart()) return;  // drop concurrent request
  const gen = guard.start();
  try {
    guard.markRunning(gen);
    await executeQuery(query);
  } finally {
    guard.end(gen);  // no-op if gen is stale (forceEnd was called)
  }
}
```

---

## §21 — Token Budget Parsing & Context Window Estimation

### User-Specified Budget Formats

Agents accept token budgets in natural language. Parse these with anchored regex to avoid
false positives in prose.

```typescript
// Supported formats: "+500k", "2M tokens", "spend 1M tokens", "1000000"
function parseTokenBudget(input: string): number | null {
  const patterns: Array<[RegExp, (m: RegExpMatchArray) => number]> = [
    [/^\+?(\d+(?:\.\d+)?)\s*[Kk]$/,  m => Math.round(parseFloat(m[1]) * 1_000)],
    [/^\+?(\d+(?:\.\d+)?)\s*[Mm]$/,  m => Math.round(parseFloat(m[1]) * 1_000_000)],
    [/^spend\s+(\d+(?:\.\d+)?)\s*[Mm]\s+tokens$/i, m => Math.round(parseFloat(m[1]) * 1_000_000)],
    [/^(\d+)$/,                        m => parseInt(m[1])],
  ];

  const trimmed = input.trim();
  for (const [pattern, convert] of patterns) {
    const match = trimmed.match(pattern);
    if (match) return convert(match);
  }
  return null;
}
```

### Accurate Context Window Estimation

The naive approach (sum all messages) is wrong when tool calls are parallel: the model emits
one assistant message split across multiple `tool_use` blocks, each followed by a `tool_result`.
Walk back to the first sibling of the split to count accurately.

```typescript
function estimateContextTokens(
  messages: Message[],
  lastApiUsage: Usage | null,
): number {
  if (!lastApiUsage) {
    // Pre-first-response: rough word-count estimate
    return messages.reduce((sum, m) => sum + roughTokenEstimate(m.content), 0);
  }

  // Find split point: last assistant message that has siblings in the same turn
  const splitIdx = findTurnSplitIndex(messages);

  // Messages after split: estimate
  const newTokens = messages
    .slice(splitIdx)
    .reduce((sum, m) => sum + roughTokenEstimate(m.content), 0);

  // Messages before split: use last API response's reported usage
  // Exclude cache tokens — server-side budget countdown doesn't include them
  const reportedTokens = lastApiUsage.input_tokens - (lastApiUsage.cache_read_input_tokens ?? 0);

  return reportedTokens + newTokens;
}

function roughTokenEstimate(content: MessageContent): number {
  if (typeof content === 'string') return Math.ceil(content.length / 4);
  if (Array.isArray(content)) return content.reduce((sum, b) => sum + roughBlockTokens(b), 0);
  return 0;
}
```

---

## §22 — Read-Only Command Validation

Before executing a shell command in plan mode or restricted contexts, validate it against
allowlists. This is deeper than regex — it parses flags and detects exfiltration vectors.

```typescript
// Per-command safe flag allowlists
const GIT_READ_ONLY: Record<string, string[]> = {
  'log':   ['--oneline', '--graph', '--decorate', '--all', '--stat', '-n', '--follow', '--format'],
  'diff':  ['--stat', '--name-only', '--name-status', '--cached', 'HEAD'],
  'show':  ['--stat', '--name-only', '--format'],
  'status': ['-s', '--short', '--branch', '--porcelain'],
  'ls-files': ['--cached', '--others', '--exclude-standard', '-m'],
};

const GH_READ_ONLY: Record<string, string[]> = {
  'pr':    ['list', 'view', 'status', 'checks', 'diff'],
  'issue': ['list', 'view', 'status'],
  'repo':  ['view', 'list'],
};

function isReadOnlyCommand(cmd: string): boolean {
  const tokens = parseShellTokens(cmd);
  if (tokens.length === 0) return false;

  const [executable, ...rest] = tokens;

  if (executable === 'git') return isReadOnlyGitCommand(rest);
  if (executable === 'gh')  return isReadOnlyGhCommand(rest);

  const SAFE_EXECUTABLES = new Set(['cat', 'head', 'tail', 'grep', 'find', 'ls',
    'pwd', 'echo', 'wc', 'sort', 'uniq', 'diff', 'rg', 'fd']);
  return SAFE_EXECUTABLES.has(executable);
}

// Detect credential-leaking git patterns
function isReadOnlyGitCommand(args: string[]): boolean {
  const subcmd = args[0];
  if (!GIT_READ_ONLY[subcmd]) return false;

  // Block --server-option (git ls-remote exfil vector)
  if (args.some(a => a.startsWith('--server-option'))) return false;

  // Block -S/-G/-O flags that accept arbitrary string args
  if (args.some(a => /^-[SGO]/.test(a) && a.length === 2)) {
    // These consume the next token as a pattern — could be used to filter then exfil
    return false;
  }

  return args.every(a => isAllowedFlag(a, GIT_READ_ONLY[subcmd]));
}

// Detect UNC path exfiltration (Windows credential leakage via NTLM)
function hasUNCPath(cmd: string): boolean {
  return /(?:\\\\|\/\/)(?:[a-zA-Z0-9._-]+)(?:\\|\/)/.test(cmd)
    || /@SSL@|DavWWWRoot/i.test(cmd);
}

// Detect gh repo exfiltration via HOST/OWNER/REPO pattern
function isGhRepoExfil(value: string): boolean {
  if (value.includes('://') || value.includes('@')) return true;
  return value.split('/').length > 2;  // HOST/OWNER/REPO has 2+ slashes
}
```

---

## §23 — Session Storage: Atomic JSONL with Per-File Queues

Transcript writes must be atomic, ordered, and resilient to concurrent writes from
swarm teammates writing to the same session log.

```typescript
class SessionStorage {
  // Per-file write queues — prevents interleaving on concurrent writes
  private writeQueues = new Map<string, WriteQueue>();

  private getQueue(filePath: string): WriteQueue {
    if (!this.writeQueues.has(filePath)) {
      this.writeQueues.set(filePath, new WriteQueue(filePath));
    }
    return this.writeQueues.get(filePath)!;
  }

  async append(filePath: string, entry: SessionEntry): Promise<void> {
    const queue = this.getQueue(filePath);
    return queue.append(entry);
  }
}

class WriteQueue {
  private pending: Array<{ entry: SessionEntry; resolve: () => void }> = [];
  private drainTimer: ReturnType<typeof setTimeout> | null = null;
  private draining = false;

  constructor(private filePath: string) {}

  append(entry: SessionEntry): Promise<void> {
    return new Promise<void>((resolve) => {
      this.pending.push({ entry, resolve });
      if (!this.drainTimer) {
        // Batch writes every 100ms for efficiency
        this.drainTimer = setTimeout(() => this.drain(), 100);
      }
    });
  }

  private async drain(): Promise<void> {
    if (this.draining) return;
    this.draining = true;
    this.drainTimer = null;

    const batch = this.pending.splice(0);
    if (batch.length === 0) { this.draining = false; return; }

    const lines = batch.map(({ entry }) => JSON.stringify(entry) + '\n').join('');
    // Atomic: O_APPEND guarantees no interleaving even with concurrent writers
    await fs.appendFile(this.filePath, lines, { flag: 'a', mode: 0o600 });

    for (const { resolve } of batch) resolve();
    this.draining = false;
  }
}

// Tail-read metadata (64KB) instead of loading the full transcript
async function readLiteMetadata(filePath: string): Promise<SessionMetadata | null> {
  const stat = await fs.stat(filePath).catch(() => null);
  if (!stat) return null;

  const TAIL_SIZE = 64 * 1024;
  const offset = Math.max(0, stat.size - TAIL_SIZE);
  const buffer = Buffer.alloc(Math.min(TAIL_SIZE, stat.size));
  const fd = await fs.open(filePath, 'r');
  await fd.read(buffer, 0, buffer.length, offset);
  await fd.close();

  // Parse JSONL lines from tail, extract metadata entries (last-wins)
  const metadata: Partial<SessionMetadata> = {};
  for (const line of buffer.toString('utf-8').split('\n')) {
    try {
      const entry = JSON.parse(line);
      if (entry.type === 'metadata') Object.assign(metadata, entry.data);
    } catch {}
  }
  return metadata as SessionMetadata;
}
```

---

*[← Event-Driven & Multi-Agent](01-event-driven-multiagent.md) | [Next: Config, Infra & Reference →](01-config-infra-reference.md)*
