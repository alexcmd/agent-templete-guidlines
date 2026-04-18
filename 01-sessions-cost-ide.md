# Universal Agent Architecture — Sessions, Cost & IDE Integration

> Sections: Session & Conversation Management · Cost & Rate Limit Management · IDE & Editor Integration

Part of the [Universal Agent Architecture](01-overview.md) series.

## 21. Session & Conversation Management

### 21.1 Session File Format (JSONL)

Each session is a `.jsonl` file — one JSON object per line — stored at:
```
~/.agent/projects/<sha256(realpath(cwd))[:16]>/sessions/<uuid>.jsonl
```

**Message schema:**
```json
{
  "uuid": "msg-uuid-v4",
  "parentUuid": "prev-msg-uuid",
  "role": "user|assistant|system",
  "content": "string or ContentBlock[]",
  "timestamp": 1713456789,
  "toolUseId": "toolu_xyz",
  "sessionId": "session-uuid",
  "metadata": {
    "model": "claude-opus-4-6",
    "usage": {"input_tokens": 1200, "output_tokens": 340},
    "cost_usd": 0.0234,
    "duration_ms": 4200,
    "cache_read_tokens": 800,
    "cache_write_tokens": 400
  }
}
```

The `parentUuid` chain forms a linked list — the canonical conversation history is the path
from any leaf message back to root. This naturally supports branching.

### 21.2 Session Branching

```python
def fork_session(session_path: Path, branch_at_message_id: str) -> Path:
    """Create a new session file branching from a specific message."""
    messages = load_jsonl(session_path)
    # Walk parentUuid chain to find the branch point
    branch_msgs = collect_ancestors(messages, branch_at_message_id)
    
    new_session_id = str(uuid.uuid4())
    new_path = session_path.parent / f"{new_session_id}.jsonl"
    
    # Write the ancestor chain with new session_id, preserving original uuids
    for msg in branch_msgs:
        msg = dict(msg, sessionId=new_session_id)
        append_jsonl(new_path, msg)
    
    return new_path
```

Branch metadata for the session index:
```json
{
  "session_id": "new-uuid",
  "parent_session_id": "orig-uuid",
  "branch_at_message_id": "msg-uuid",
  "created_at": "2026-04-18T10:00:00Z"
}
```

### 21.3 Session Index (SQLite)

For large session archives, maintain a SQLite index alongside the JSONL files:

```sql
CREATE TABLE sessions (
    id          TEXT PRIMARY KEY,
    cwd_hash    TEXT NOT NULL,
    parent_id   TEXT REFERENCES sessions(id),
    title       TEXT,
    created_at  INTEGER,   -- unix timestamp
    updated_at  INTEGER,
    message_count INTEGER DEFAULT 0,
    total_cost  REAL DEFAULT 0.0,
    archived    INTEGER DEFAULT 0
);

CREATE INDEX idx_sessions_cwd ON sessions(cwd_hash, updated_at DESC);
```

### 21.4 Session Resume Flow

```python
def resume_session(session_id: str, cwd: str) -> Session:
    session = Session(cwd, session_id)  # loads JSONL
    # Restore cost state from message metadata
    cost = CostTracker()
    for msg in session.messages:
        if "metadata" in msg and "cost_usd" in msg["metadata"]:
            cost.total += msg["metadata"]["cost_usd"]
    # Inject away summary if session was idle
    if time_since_last_message(session) > AWAY_THRESHOLD:
        inject_away_summary(session)
    return session
```

### 21.5 Session Lifecycle & Cleanup

```
Active         → last message < 1h ago
Idle           → last message 1h–24h ago
Stale          → last message > 24h ago → trigger away summary on re-entry
Archived       → last message > 90 days → moved to archived/
Auto-deleted   → archived + storage quota exceeded (oldest first)
```

Cleanup policy (configurable):
```json
{
  "session": {
    "archive_after_days": 90,
    "auto_delete_after_days": 365,
    "max_sessions_per_project": 100,
    "max_total_size_mb": 500
  }
}
```

---

## 22. Cost & Rate Limit Management

### 22.1 Token Counting

**Estimation (fast, no API call):**
```python
def estimate_tokens(text: str) -> int:
    return len(text) // 4  # ~4 chars per token rule of thumb

def estimate_messages_tokens(messages: list, tools: list = None) -> int:
    total = sum(estimate_tokens(json.dumps(m)) for m in messages)
    if tools:
        total += sum(estimate_tokens(json.dumps(t)) for t in tools)
    return total
```

**Exact count (use for compaction decisions, not every turn):**
```python
def count_tokens_exact(messages: list, model: str, tools: list = None) -> int:
    result = client.messages.count_tokens(
        model=model,
        messages=messages,
        tools=tools or []
    )
    return result.input_tokens
```

Use estimation for per-turn checks; use the exact API only before compaction decisions
(the API call itself costs ~0.5ms and is not billed).

### 22.2 Cost Tracking

```python
MODEL_RATES_USD_PER_MTok = {
    "claude-opus-4-6":           {"in": 15.0,  "out": 75.0,  "cache_read": 1.5,  "cache_write": 18.75},
    "claude-opus-4-7":           {"in": 15.0,  "out": 75.0,  "cache_read": 1.5,  "cache_write": 18.75},
    "claude-sonnet-4-6":         {"in": 3.0,   "out": 15.0,  "cache_read": 0.3,  "cache_write": 3.75},
    "claude-haiku-4-5-20251001": {"in": 0.80,  "out": 4.0,   "cache_read": 0.08, "cache_write": 1.0},
}

@dataclass
class TurnCost:
    model: str
    input_tokens: int
    output_tokens: int
    cache_read_tokens: int
    cache_write_tokens: int
    cost_usd: float
    duration_ms: int

def compute_cost(usage, model: str) -> float:
    r = MODEL_RATES_USD_PER_MTok.get(model, MODEL_RATES_USD_PER_MTok["claude-opus-4-6"])
    return (
        usage.input_tokens              * r["in"]           / 1_000_000
        + usage.output_tokens           * r["out"]          / 1_000_000
        + getattr(usage, "cache_read_input_tokens", 0)   * r["cache_read"]  / 1_000_000
        + getattr(usage, "cache_creation_input_tokens", 0) * r["cache_write"] / 1_000_000
    )
```

**Prompt cache efficiency metric:**
```python
def cache_efficiency(usage) -> float:
    """0.0 = no cache benefit; 1.0 = entire context from cache."""
    total = usage.input_tokens + getattr(usage, "cache_read_input_tokens", 0)
    if total == 0: return 0.0
    return getattr(usage, "cache_read_input_tokens", 0) / total
```

### 22.3 Budget Enforcement

```python
class BudgetController:
    def __init__(self, max_cost_usd: float, warn_at_pct: float = 0.85):
        self.max_cost = max_cost_usd
        self.warn_pct = warn_at_pct
        self.spent = 0.0

    def record(self, cost: float):
        self.spent += cost
        ratio = self.spent / self.max_cost
        if ratio >= 1.0:
            raise BudgetExhausted(f"Budget ${self.max_cost:.2f} exceeded (spent ${self.spent:.2f})")
        if ratio >= self.warn_pct:
            warn_user(f"Approaching budget limit: ${self.spent:.2f} / ${self.max_cost:.2f}")

    def remaining(self) -> float:
        return max(0.0, self.max_cost - self.spent)
```

For multi-agent systems, use a shared atomic counter (thread-safe) across all agents:

```python
import threading

class SharedBudget:
    def __init__(self, total_usd: float):
        self._lock = threading.Lock()
        self._spent = 0.0
        self._total = total_usd

    def charge(self, cost: float) -> bool:
        """Returns False if budget exhausted."""
        with self._lock:
            if self._spent + cost > self._total:
                return False
            self._spent += cost
            return True
```

### 22.4 Rate Limit Handling

```python
import time, random

def with_retry(fn, max_retries: int = 8):
    for attempt in range(max_retries):
        try:
            return fn()
        except anthropic.RateLimitError as e:
            retry_after = float(e.response.headers.get("retry-after", 2 ** attempt))
            time.sleep(retry_after + random.uniform(0, 0.5))
        except anthropic.APIStatusError as e:
            if e.status_code == 529:  # overloaded
                time.sleep(min(2 ** attempt + random.uniform(0, 1), 60))
            elif e.status_code == 500 and attempt < 3:
                time.sleep(2 ** attempt)
            else:
                raise
    raise RuntimeError(f"Failed after {max_retries} retries")
```

**Rate limit response headers:**
| Header | Meaning |
|--------|---------|
| `retry-after` | Seconds to wait before retrying (429) |
| `x-ratelimit-limit-requests` | Requests per minute ceiling |
| `x-ratelimit-remaining-requests` | Remaining in current window |
| `x-ratelimit-limit-tokens` | Tokens per minute ceiling |
| `x-ratelimit-remaining-tokens` | Remaining tokens in current window |
| `x-ratelimit-reset-requests` | ISO timestamp when request window resets |

### 22.5 Context Window Budget

Monitor and act before the window fills:

```python
CONTEXT_LIMITS = {
    "claude-opus-4-6":           200_000,
    "claude-sonnet-4-6":         200_000,
    "claude-haiku-4-5-20251001": 200_000,
}

def context_budget_check(messages: list, tools: list, model: str, system_prompt_tokens: int = 0):
    limit = CONTEXT_LIMITS.get(model, 200_000)
    used = estimate_messages_tokens(messages, tools) + system_prompt_tokens
    ratio = used / limit
    
    if ratio >= 0.95:
        trigger_compaction(messages)   # hard threshold
    elif ratio >= 0.80:
        warn_user(f"Context {ratio:.0%} full — will compact soon")
    
    return ratio
```

---

## 23. IDE & Editor Integration

### 23.1 Integration Patterns

Two patterns exist for connecting an agent to an IDE:

**Pattern A — IDE Extension calls CLI** (most common)
```
IDE Extension ──subprocess──► Agent CLI
                              (stdin/stdout protocol)
IDE receives:
  - streaming text deltas
  - tool call notifications
  - file diff events
  - session metadata
```

**Pattern B — Agent drives IDE via MCP**
```
Agent ──MCP client──► IDE MCP server
                      (IDE exposes tools)
Agent can call:
  - ide.openFile(path, line)
  - ide.showDiff(old, new)
  - ide.getSelection()
  - ide.getDiagnostics()
```

### 23.2 IDE Context Injection Protocol

To avoid re-sending the full conversation on every keystroke:

```
Turn 1 (full):   [system_prompt] + [full_history] + [file_contents] + user_message
Turn 2+ (delta): [delta_messages_only] + [changed_files_only] + user_message
```

**Delta context format:**
```json
{
  "type": "ide_delta_context",
  "changed_files": [
    {
      "path": "src/auth.py",
      "content": "...full new content...",
      "cursor_line": 42,
      "selection": {"start": 40, "end": 45}
    }
  ],
  "diagnostics": [
    {
      "path": "src/auth.py",
      "line": 42,
      "severity": "error",
      "message": "Undefined name 'token'"
    }
  ],
  "open_files": ["src/auth.py", "src/types.py"]
}
```

Inject this as a system message at the start of each turn. On the first turn, include all
open files; on subsequent turns, include only files that changed since the last turn.

### 23.3 File Change Events

Wire the OS file watcher to feed changes back into the agent's context:

```python
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

class ProjectWatcher(FileSystemEventHandler):
    def __init__(self, on_change):
        self.on_change = on_change
        self._debounce = {}
    
    def on_modified(self, event):
        if event.is_directory: return
        path = event.src_path
        # Debounce: ignore if same file changed in last 100ms
        now = time.monotonic()
        if path in self._debounce and now - self._debounce[path] < 0.1:
            return
        self._debounce[path] = now
        self.on_change(path)

observer = Observer()
observer.schedule(ProjectWatcher(on_file_changed), path=cwd, recursive=True)
observer.start()
```

### 23.4 Diff/Patch Format for IDE Display

When displaying file changes to the user in an IDE:

```python
import difflib

def make_unified_diff(old_content: str, new_content: str, filepath: str) -> str:
    return "".join(difflib.unified_diff(
        old_content.splitlines(keepends=True),
        new_content.splitlines(keepends=True),
        fromfile=f"a/{filepath}",
        tofile=f"b/{filepath}",
        n=3  # context lines
    ))

def diff_stats(old: str, new: str) -> dict:
    added = deleted = 0
    for line in make_unified_diff(old, new, "").splitlines():
        if line.startswith("+") and not line.startswith("+++"): added += 1
        if line.startswith("-") and not line.startswith("---"): deleted += 1
    return {"added": added, "deleted": deleted}
```

### 23.5 MCP Tool Declarations for IDE Integration

```python
IDE_TOOLS = [
    {
        "name": "ide__open_file",
        "description": "Open a file in the editor at a specific line",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "line": {"type": "integer"}
            },
            "required": ["path"]
        }
    },
    {
        "name": "ide__get_selection",
        "description": "Get the currently selected text in the editor",
        "input_schema": {"type": "object", "properties": {}}
    },
    {
        "name": "ide__get_diagnostics",
        "description": "Get linting/type errors for the current file",
        "input_schema": {
            "type": "object",
            "properties": {"path": {"type": "string"}},
            "required": ["path"]
        }
    },
    {
        "name": "ide__show_diff",
        "description": "Show a diff in the editor's diff viewer",
        "input_schema": {
            "type": "object",
            "properties": {
                "old_content": {"type": "string"},
                "new_content": {"type": "string"},
                "filename": {"type": "string"}
            },
            "required": ["old_content", "new_content", "filename"]
        }
    }
]
```

---


---

*[← Planning & Hooks](01-planning-hooks.md) | [Next: Distributed Agents →](01-distributed-agents.md)*
