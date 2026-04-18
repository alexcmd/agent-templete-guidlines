# Todo & Task Management: CRUD and Visualization

> Grounded in production source analysis of Claude Code, Gemini CLI, Claw Code, and Claurst  
> Covers: data models, storage patterns, CRUD, state machines, dependency DAGs, visualization, reminder injection

---

## 1. The Dual-System Pattern

Every production coding agent implements **two levels** of task management:

| Level | Tool | Purpose | Persistence |
|-------|------|---------|-------------|
| **Simple checklist** | `TodoWrite` / `write_todos` | Current session multi-step tracking | In-memory or session-scoped file |
| **Structured tasks** | `TaskCreate/Update/Get/List` | Persistent project tasks with dependencies | Files or in-memory store |

Use the simple tool for "what am I doing right now"; use the advanced tool for "what is this project's work queue".

---

## 2. Data Models

### 2.1 Minimal TodoItem (production-proven)

```typescript
// claude-code legacy / gemini-cli write_todos
{
  content: string,                                        // what to do
  status: 'pending' | 'in_progress' | 'completed',
  activeForm: string   // present-continuous label for spinner, e.g. "Running tests"
}
```

### 2.2 Recommended TodoItem (adds id + priority — claurst)

```typescript
{
  id: string,          // stable, unique — required for transition validation
  content: string,
  status: 'pending' | 'in_progress' | 'completed',
  priority?: 'high' | 'medium' | 'low'
}
```

The `id` field is the key upgrade. Without it, state-transition validation is impossible.

### 2.3 Advanced Task (all agents — shared shape)

```typescript
{
  id: string,                // format: UUID v4 (claurst) | seq int (claude-code) | 6-hex (gemini)
  subject: string,
  description: string,
  status: TaskStatus,        // see §2.4
  owner?: string,            // agent ID for multi-agent isolation
  blocks: string[],          // IDs of tasks this task prevents from starting
  blockedBy: string[],       // IDs that must complete before this task can start
  metadata?: Record<string, unknown>,
  output?: string,           // captured result/stdout
  created_at: string,        // ISO timestamp
  updated_at: string,
}
```

### 2.4 Status Values by Agent

| Status | claude-code | gemini-cli | claw-code | claurst |
|--------|:-----------:|:----------:|:---------:|:-------:|
| `pending` / `open` | ✅ | `open` | `Pending` | `Pending` |
| `in_progress` | ✅ | ✅ | `Running` | `InProgress` |
| `completed` / `closed` | ✅ | `closed` | `Completed` | `Completed` |
| `blocked` | ❌ | ✅ | ❌ | ❌ |
| `failed` | ❌ | ❌ | ✅ | ✅ |
| `deleted` | ❌ | ❌ | ❌ | ✅ (triggers removal) |
| `stopped` | ❌ | ❌ | ✅ | ❌ |

**Recommendation**: implement at minimum `pending`, `in_progress`, `completed`, `failed`.

### 2.5 Claw-Code's TaskPacket — Structured Work Order for Sub-Agents

```rust
struct TaskPacket {
    objective: String,
    scope: TaskScope,             // Module | Feature | Repository | Service
    scope_path: Option<String>,
    worktree: Option<String>,     // isolated git worktree
    repo: String,
    branch_policy: String,        // e.g. "create: task/<id>"
    acceptance_tests: Vec<String>,
    commit_policy: String,
    reporting_contract: String,
    escalation_policy: String,
}
```

Use this pattern when dispatching sub-agents — a free-text `description` is insufficient for reliable sub-agent execution.

---

## 3. Storage Patterns

### 3.1 Simple TodoWrite Storage

| Approach | Example | Session-scoped? | Pros | Cons |
|----------|---------|:--------------:|------|------|
| In-memory (AppState) | claude-code, gemini-cli | ✅ | Zero I/O | Lost on crash |
| CWD file (`.clawd-todos.json`) | claw-code | ❌ | Survives restart | Shared across sessions |
| Session-scoped file (`~/.agent/todos/<session>.json`) | claurst | ✅ | Best: isolated + persistent | Small file I/O per write |

**Use the session-scoped file pattern** — it provides isolation and crash recovery at negligible cost.

```python
def todos_path(session_id: str) -> Path:
    return Path.home() / '.myagent' / 'todos' / f'{session_id}.json'
```

### 3.2 Advanced Task Storage

| Approach | Example | Persistent? | Concurrent-safe? |
|----------|---------|:-----------:|:----------------:|
| One file per task + file locking | claude-code V2 | ✅ | ✅ (proper-lockfile) |
| One file per task (no locking) | gemini-cli tracker | ✅ | ❌ |
| In-memory HashMap | claw-code | ❌ | ❌ |
| In-memory DashMap (concurrent) | claurst | ❌ | ✅ |

**For single-agent use**: in-memory DashMap/HashMap is fine. **For multi-agent swarms**: use file-per-task with file locking.

```python
# File-per-task pattern (claude-code V2 style)
tasks_dir = Path.home() / '.myagent' / 'tasks' / task_list_id
task_file = tasks_dir / f'{task_id}.json'
# Use filelock library for safe concurrent writes:
with FileLock(str(task_file) + '.lock'):
    task_file.write_text(json.dumps(task, indent=2))
```

---

## 4. CRUD Operations

### 4.1 TodoWrite: Full List Replacement

**All agents use this pattern** — the model sends the complete new list; old list is replaced atomically.

```python
def execute_todo_write(session_id: str, new_todos: list[dict]) -> str:
    old_todos = {t['id']: t for t in load_todos(session_id)}
    
    # Validate
    ids = [t['id'] for t in new_todos]
    assert len(ids) == len(set(ids)), "Duplicate IDs"
    assert all(t.get('content') for t in new_todos), "Empty content"
    
    # State transition validation (claurst pattern)
    VALID_TRANSITIONS = {('pending','in_progress'), ('pending','completed'), ('in_progress','completed')}
    for t in new_todos:
        if t['id'] in old_todos:
            old_s, new_s = old_todos[t['id']]['status'], t['status']
            if old_s != new_s and (old_s, new_s) not in VALID_TRANSITIONS:
                raise ValueError(f"Invalid transition {old_s}→{new_s} for '{t['id']}'")
    
    save_todos(session_id, new_todos)
    
    # Verification nudge (claude-code / claw-code pattern)
    newly_completed = [t for t in new_todos
                       if t['id'] in old_todos
                       and old_todos[t['id']]['status'] != 'completed'
                       and t['status'] == 'completed']
    has_verify = any('verif' in t['content'].lower() for t in new_todos)
    nudge = len(newly_completed) >= 3 and not has_verify
    
    return format_todo_summary(new_todos), nudge

def format_todo_summary(todos: list[dict]) -> str:
    icons = {'pending': '[ ]', 'in_progress': '[~]', 'completed': '[x]'}
    counts = Counter(t['status'] for t in todos)
    header = (f"Todo list ({len(todos)} total: {counts['pending']} pending, "
              f"{counts['in_progress']} in progress, {counts['completed']} completed)")
    lines = [header, ''] + [f"  {icons[t['status']]} {t['content']}" for t in todos]
    return '\n'.join(lines)
```

### 4.2 Advanced Task CRUD

```python
import uuid
from datetime import datetime, timezone

task_store: dict[str, dict] = {}

def task_create(subject: str, description: str, metadata: dict = None) -> str:
    task_id = str(uuid.uuid4())
    task_store[task_id] = {
        'id': task_id, 'subject': subject, 'description': description,
        'status': 'pending', 'owner': None,
        'blocks': [], 'blocked_by': [], 'metadata': metadata, 'output': None,
        'created_at': _now(), 'updated_at': _now()
    }
    return task_id

def task_get(task_id: str) -> dict:
    return task_store[task_id]

def task_list(include_completed: bool = False) -> list[dict]:
    return [t for t in task_store.values()
            if include_completed or t['status'] not in ('completed', 'deleted', 'failed')]

def task_update(task_id: str, **updates) -> dict:
    task = task_store[task_id]
    if updates.get('status') == 'deleted':
        del task_store[task_id]
        return task
    for k, v in updates.items():
        if k == 'add_blocks':
            task['blocks'].extend(v)
        elif k == 'add_blocked_by':
            task['blocked_by'].extend(v)
        else:
            task[k] = v
    task['updated_at'] = _now()
    return task

def _now() -> str:
    return datetime.now(timezone.utc).isoformat()
```

### 4.3 Dependency Validation (gemini-cli tracker pattern)

```python
def _detect_cycle(task_id: str, new_deps: list[str]) -> bool:
    """DFS cycle detection before adding dependencies."""
    visited = set()
    def dfs(node):
        if node == task_id:
            return True
        if node in visited or node not in task_store:
            return False
        visited.add(node)
        return any(dfs(dep) for dep in task_store[node].get('blocked_by', []))
    return any(dfs(dep) for dep in new_deps)

def task_add_dependency(task_id: str, dependency_id: str):
    if _detect_cycle(task_id, [dependency_id]):
        raise ValueError(f"Adding {dependency_id} as dependency of {task_id} would create a cycle")
    task_store[task_id]['blocked_by'].append(dependency_id)
    task_store[dependency_id]['blocks'].append(task_id)

def can_close(task_id: str) -> bool:
    """Task can only close when all its dependencies are already closed."""
    task = task_store[task_id]
    return all(
        task_store.get(dep_id, {}).get('status') == 'completed'
        for dep_id in task['blocked_by']
    )
```

---

## 5. Reminder Injection into Agent Loop

**Problem**: In long sessions the model loses track of its todo list because it's not visible in recent context.

**Solution** (claude-code pattern): Periodically inject the current todo list as an attachment in the user message.

```python
class TodoReminderMiddleware:
    def __init__(self, threshold_turns: int = 5, interval_turns: int = 10):
        self.turns_since_write = 0
        self.turns_since_reminder = 0
        self.threshold = threshold_turns
        self.interval = interval_turns

    def on_tool_call(self, tool_name: str):
        if tool_name == 'TodoWrite':
            self.turns_since_write = 0

    def on_assistant_turn(self):
        self.turns_since_write += 1
        self.turns_since_reminder += 1

    def get_reminder(self, session_id: str) -> str | None:
        if (self.turns_since_write > self.threshold
                and self.turns_since_reminder > self.interval):
            todos = load_todos(session_id)
            if todos:
                self.turns_since_reminder = 0
                summary = format_todo_summary(todos)
                return f"<todo_reminder>\n{summary}\n</todo_reminder>"
        return None

# In the agent loop, before each user message:
reminder = todo_middleware.get_reminder(session_id)
if reminder:
    user_message = user_message + '\n\n' + reminder
```

---

## 6. System Prompt Integration

### 6.1 Minimal TodoWrite Protocol (in tool description)

The tool description itself serves as the usage protocol — 80+ lines in gemini-cli. Key elements:

```
Use TodoWrite to maintain a living checklist whenever you have more than one step to complete.
- Create the todo list BEFORE starting any multi-step task.
- Mark items `in_progress` when you BEGIN them (only one at a time).
- Mark items `completed` only after you have VERIFIED the result.
- Never mark complete without evidence.
- If a step fails, add a recovery item, do not delete the failed item.
- When ALL items are completed, pass an empty list to clear.
```

### 6.2 Tracker System Prompt Protocol (gemini-cli tracker)

When using an advanced task system, inject a protocol section into the system prompt:

```
## TASK MANAGEMENT PROTOCOL

You MUST use TaskCreate for any request requiring more than 2 distinct actions.
Never maintain a mental task list — all tasks must be persisted via the task tools.

Workflow:
1. PLAN: Call TaskCreate for each sub-task before starting any work.
2. START: Call TaskUpdate(status=in_progress) on the task you are about to execute.
3. COMPLETE: Call TaskUpdate(status=completed, output=<summary>) when done.
4. BLOCK: If blocked, call TaskUpdate(status=failed, output=<reason>).

Dependency rules:
- Use addBlockedBy when task B cannot start until task A finishes.
- Do not mark a task completed if any of its blockedBy tasks are still open.
```

---

## 7. Visualization

### 7.1 Text Summary (all agents — minimum viable)

Every `TodoWrite` response should return a structured text summary:

```
Todo list (4 total: 2 pending, 1 in progress, 1 completed)

  [x] Set up project scaffold
  [~] Implement authentication
  [ ] Add API endpoints
  [ ] Write tests
```

### 7.2 ASCII Dependency Tree (gemini-cli tracker_visualize)

For dependency-aware task systems, provide a `visualize_tasks()` tool:

```python
STATUS_ICON = {
    'pending': '⭕', 'in_progress': '🚧', 'blocked': '⛔',
    'completed': '✅', 'failed': '❌', 'deleted': '🗑️'
}

def visualize_tasks() -> str:
    tasks = list(task_store.values())
    store_ids = set(task_store.keys())
    # Root tasks: not blocked by anything in the current store
    roots = [t for t in tasks if not any(bid in store_ids for bid in t.get('blocked_by', []))]
    
    lines = ['Task Graph:']
    visited = set()

    def render(task, depth=0):
        if task['id'] in visited:
            return
        visited.add(task['id'])
        pad = '  ' * depth
        icon = STATUS_ICON.get(task['status'], '○')
        lines.append(f"{pad}{icon} {task['id'][:6]} [{task['status'].upper()}] {task['subject']}")
        deps = [task_store[bid] for bid in task.get('blocked_by', []) if bid in task_store]
        if deps:
            dep_labels = [d['subject'][:25] for d in deps]
            lines.append(f"{pad}  └─ depends on: {', '.join(dep_labels)}")
        for blocked_id in task.get('blocks', []):
            if blocked_id in task_store:
                render(task_store[blocked_id], depth + 1)

    for root in roots:
        render(root)
    return '\n'.join(lines)
```

Output:
```
Task Graph:
✅ a3f1c2 [COMPLETED] Project Setup
🚧 b4d5e6 [IN_PROGRESS] Implement auth
  └─ depends on: Project Setup
⭕ c6f7a8 [PENDING] Add API endpoints
  └─ depends on: Implement auth
```

### 7.3 Live TUI Panel (claude-code TaskListV2 pattern)

For terminal UI agents (React/Ink, ratatui), render a live task panel that auto-updates:

```
Tasks (5 total: 2 done, 1 in progress, 2 open)               agent-1: Reading auth.ts
──────────────────────────────────────────────────────────────────────────────────────
  ✓  Set up project scaffold
  ✓  Write database schema
  ⟳  Implement authentication     ← current task
  ○  Add API endpoints
  ○  Write tests
  ── +3 pending hidden ──
```

Key behaviors from claude-code's implementation:
- Sort: recently-completed (30s TTL) first → in-progress → pending → older-completed
- Truncate to `min(10, terminal_rows - 14)` — never overflow the screen
- Show `"... +N pending, M in progress"` for hidden overflow
- Show per-agent activity description in swarm mode
- Re-render on every state change (not polling)

---

## 8. Multi-Agent Task Isolation

### 8.1 Key Concern

When multiple agent instances share a task store, you need to prevent cross-contamination.

**Claude-code approach** — key todos by `agentId`:
```python
class AppState:
    todos: dict[str, list[TodoItem]] = {}  # keyed by agentId or sessionId
    
    def get_todos(self, agent_id: str) -> list[TodoItem]:
        return self.todos.get(agent_id, [])
    
    def set_todos(self, agent_id: str, todos: list[TodoItem]):
        self.todos[agent_id] = todos
```

**Claude-code V2 approach** — shared `taskListId` + `owner` field:
- All agents in a team use the same `taskListId` (e.g., team name)
- Each task has an `owner` field set to the claiming agent's ID
- Unowned tasks are auto-claimed by any available agent via file watcher

### 8.2 Task File Watcher (claude-code --tasks mode)

```python
import asyncio
from watchfiles import awatch

async def task_watcher(tasks_dir: Path, agent_id: str, submit_prompt):
    async for changes in awatch(tasks_dir):
        for change_type, file_path in changes:
            if not file_path.endswith('.json'):
                continue
            task = load_task(file_path)
            if (task['status'] == 'pending'
                    and task.get('owner') is None
                    and all_deps_completed(task)):
                claim_task(task, agent_id)
                await submit_prompt(f"{task['subject']}\n\n{task['description']}")
```

This enables external task injection: a CI system, human, or manager agent drops task files into the watched directory and available agents auto-pick them up.

---

## 9. Tool Definitions Reference

### TodoWrite

```json
{
  "name": "TodoWrite",
  "description": "Manage a structured todo list for tracking multi-step work. Call with the complete updated list — previous state is replaced. Use activeForm for present-continuous descriptions (e.g., 'Running tests'). Clear the list by passing an empty array when all tasks are done.",
  "input_schema": {
    "type": "object",
    "properties": {
      "todos": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "id":         { "type": "string", "description": "Stable unique identifier" },
            "content":    { "type": "string", "description": "Task description" },
            "status":     { "type": "string", "enum": ["pending", "in_progress", "completed"] },
            "priority":   { "type": "string", "enum": ["high", "medium", "low"] }
          },
          "required": ["id", "content", "status"]
        }
      }
    },
    "required": ["todos"]
  }
}
```

### TaskCreate

```json
{
  "name": "TaskCreate",
  "description": "Create a persistent task with subject, description, and optional metadata. Returns the task ID for use with other task tools.",
  "input_schema": {
    "type": "object",
    "properties": {
      "subject":     { "type": "string" },
      "description": { "type": "string" },
      "metadata":    { "type": "object" }
    },
    "required": ["subject", "description"]
  }
}
```

### TaskUpdate

```json
{
  "name": "TaskUpdate",
  "description": "Update an existing task. Partial updates — only provided fields are changed. Set status=deleted to remove the task.",
  "input_schema": {
    "type": "object",
    "properties": {
      "taskId":      { "type": "string" },
      "subject":     { "type": "string" },
      "description": { "type": "string" },
      "status":      { "type": "string", "enum": ["pending", "in_progress", "completed", "failed", "deleted"] },
      "owner":       { "type": "string" },
      "addBlocks":   { "type": "array", "items": { "type": "string" } },
      "addBlockedBy":{ "type": "array", "items": { "type": "string" } },
      "output":      { "type": "string" },
      "metadata":    { "type": "object" }
    },
    "required": ["taskId"]
  }
}
```

---

## 10. Best Practices Summary

1. **Always add `id` to TodoItems** — enables transition validation and stable references
2. **Use session-scoped file storage** — not CWD-relative (shared pollution) or pure in-memory (crash loss)
3. **Enforce state transitions** — completed is immutable; prevent `in_progress → pending`
4. **Inject todo reminders** every N turns — the model will lose context without periodic re-injection
5. **Use file locking** for disk-based tasks in multi-agent scenarios — prevents corruption
6. **Validate circular dependencies** before adding them — DFS at update time
7. **Gate closure on dependencies** — task cannot close until all its `blockedBy` tasks are closed
8. **Return a text summary** from every `TodoWrite` call — counts and per-item icons
9. **Use `TaskPacket` format** for sub-agent work orders — include scope, branch policy, acceptance tests
10. **Separate the two levels** — `TodoWrite` for current-session checklist; `TaskCreate` for persistent project tracking

---

## 11. Cross-Agent Feature Matrix

| Feature | claude-code | gemini-cli | claw-code | claurst |
|---------|:-----------:|:----------:|:---------:|:-------:|
| Todo `id` field | ❌/✅(V2) | ❌ | ❌ | ✅ |
| Todo `priority` field | ❌ | ❌ | ❌ | ✅ |
| State transition enforcement | ❌ | ❌ | ❌ | ✅ |
| Session-scoped todo file | N/A | N/A | ❌ | ✅ |
| Verification nudge | ✅ | ❌ | ✅ | ❌ |
| Periodic reminder injection | ✅ | ❌ | ❌ | ❌ |
| System prompt task protocol | ❌ | ✅ | ❌ | ❌ |
| File-locked task files | ✅ (V2) | ❌ | ❌ | ❌ |
| Task file watcher mode | ✅ | ❌ | ❌ | ❌ |
| TaskPacket work order | ❌ | ❌ | ✅ | ❌ |
| Dependency DAG + validation | ✅ | ✅ | ❌ | ❌ |
| Circular dependency detection | ❌ | ✅ | ❌ | ❌ |
| Close gate (deps must close) | ❌ | ✅ | ❌ | ❌ |
| Live TUI panel | ✅ React/Ink | ✅ TodoTray | ❌ | ❌ |
| ASCII dependency visualization | ❌ | ✅ | ❌ | ❌ |
| Sub-agent task isolation | ✅ | ❌ | `team_id` | ❌ |
| `/tasks` slash command | ❌ | ✅ (bg pane) | ❌ | ✅ |
| Multi-agent swarm support | ✅ | ❌ | Partial | ❌ |
