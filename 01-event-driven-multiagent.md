# Universal Agent Architecture — Event-Driven & Multi-Agent Topologies

> Sections: Event-Driven Architecture (ESAA) · Multi-Agent Topologies

Part of the [Universal Agent Architecture](01-overview.md) series.

## 10. Event-Driven Architecture (ESAA Pattern)

Based on arXiv:2602.23193 and OpenHands (arXiv:2407.16741):

```
                     AGENT ACTION
                          │
                          ▼
              ┌───────────────────────┐
              │  EventStore           │
              │  append-only JSONL    │
              │  event_id, timestamp  │
              │  type, payload, meta  │
              └───────────┬───────────┘
                          │ publish
                          ▼
              ┌───────────────────────┐
              │  EventBus             │
              │  topic routing        │
              └───────────┬───────────┘
                          │
           ┌──────────────┼──────────────┐
           ▼              ▼              ▼
      ┌─────────┐   ┌──────────┐   ┌─────────┐
      │MaterializedView│HookRunner│ │Audit Log│
      │(CQRS read) │ │(pre/post)│   │(append) │
      └─────────┘   └──────────┘   └─────────┘
```

**Event types to implement in all three languages:**

| Event | When emitted |
|-------|-------------|
| `AgentStarted` | Session begins |
| `TurnStarted` | LLM call begins |
| `TextDelta` | Streaming token received |
| `ToolCallRequested` | LLM emits tool_use block |
| `PermissionChecked` | Before tool execution |
| `ToolCallStarted` | Tool execution begins |
| `ToolCallCompleted` | Tool execution ends with result |
| `ToolCallFailed` | Tool execution raises error |
| `TurnCompleted` | LLM call ends (end_turn) |
| `CompactionTriggered` | History summarization begins |
| `CompactionCompleted` | History replaced with summary |
| `AgentStopped` | Session ends |

**Hook runner contract** (runs synchronously on `ToolCallRequested`):

```
pre_hook(event: ToolCallRequested) → HookDecision
  ALLOW   → proceed with permission check
  DENY    → abort tool call, return error to LLM
  ASK     → escalate to user regardless of policy
  REPLACE → substitute a different tool call

post_hook(event: ToolCallCompleted) → void
  (side-effects only: logging, metrics, notifications)
```

---

## 11. Multi-Agent Topologies

Four proven topologies from the literature. Choose based on task structure:

### 11.1 Supervisor (Centralized)

```
User ─► Supervisor Agent
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
 Worker A  Worker B  Worker C
```

**Use when:** Tasks have a clear decomposition; coordinator needs to route based on output.
**Paper:** AutoGen (arXiv:2308.08155), MetaGPT (arXiv:2308.00352)
**Failure mode:** Single point of failure; bottleneck on supervisor context.

### 11.2 Swarm (Peer Handoff)

```
 Agent A ──handoff──► Agent B ──handoff──► Agent C
   ▲                                           │
   └───────────────────────────────────────────┘
```

**Use when:** Tasks flow through specializations in sequence; no fixed routing.
**Paper:** CAMEL (arXiv:2303.17760)
**Implementation:** `handoff_tool(agent_id)` transfers control with shared state.

### 11.3 Parallel Map-Reduce

```
Coordinator
    │ split
    ├─── Sub-task 1 ──► Worker 1
    ├─── Sub-task 2 ──► Worker 2
    └─── Sub-task N ──► Worker N
    │ gather + reduce
    ▼
 Synthesized result
```

**Use when:** Independent subtasks of equal structure (e.g., audit N tables, profile N files).
**Efficiency:** N× throughput on wall-clock time, identical cost.
**Language note:** `cobalt::gather` (C++), `coroutineScope { async { } }` (Kotlin),
`asyncio.gather` / LangGraph BSP (Python).

### 11.4 SOP Pipeline (Sequential)

```
PM Agent ──► Architect Agent ──► Coder Agent ──► Tester Agent
   (PRD)         (Design)           (Code)         (Verified)
```

**Use when:** Output of each stage is the input of the next; strict artifact dependencies.
**Paper:** MetaGPT (arXiv:2308.00352)
**Key feature:** Artifact validation gates: the next agent only runs when the previous
artifact passes a structured check (e.g., code must compile before test agent runs).

### 11.5 Manager-Executor

```
User ──► Manager Agent (large model: Opus, GPT-4o)
              │ plan + delegate via AgentTool
    ┌─────────┼─────────┐
    ▼         ▼         ▼
 Executor  Executor  Executor   (smaller model: Sonnet, GPT-4o-mini)
 (worktree) (worktree) (worktree)
    │         │         │
    └─────────┴─────────┘
         shared Arc<CostTracker>
         total_budget_usd enforced across all executors
```

**Use when:** Tasks benefit from separation between strategic reasoning and implementation;
cost optimization by routing planning to expensive models and execution to cheaper ones.

**Key design decisions:**
- The manager model is explicitly forbidden from using `Bash` directly (enforced by
  tool filtering) — it can only delegate. This prevents the manager from bypassing the
  executor layer.
- Each executor runs in a `git worktree` (isolated copy of the repo) to prevent
  file-write conflicts between concurrent executors.
- A shared `CostTracker` (with atomic counter) enforces a total USD budget cap across
  all executors, not per-executor. When the budget is exhausted, all executors stop.
- `ScratchpadGate` — if you want the manager to reason in a scratchpad before
  delegating, block `Write`/`Edit` tools with a lock until a specific unlock phrase
  appears in the model's output stream. This ensures the manager completes its
  reasoning before any file is touched.
- 6 built-in presets for common workloads: (code-review, bug-fix, feature-dev,
  security-audit, data-analysis, doc-generation). Presets encode the model pair,
  budget split, executor count, and isolation policy.

**Budget split policy options:**
```
Equal          → budget / num_executors per executor
Proportional   → based on task complexity estimate
Manager-first  → fixed manager budget, remainder split among executors
Unlimited-exec → cap only the manager; executors run until done
```

### 11.6 Coordinator Mode (Tool Filtering)

When an orchestrator agent runs in coordinator mode, apply two tool-filtering rules:

1. **Strip coordinator-only tools from workers.** Workers must not have `AgentTool`,
   `TeamCreate`, `SendMessage`, `SyntheticOutput` — otherwise a worker could spawn
   unauthorized sub-agents.

2. **Forbid bash on the coordinator.** The coordinator reasons and delegates; it must not
   execute shell commands directly. Enforced by removing `BashTool` from the coordinator's
   tool set at session initialization.

```python
# coordinator tool set
coordinator_tools = all_tools - {BashTool, PowerShellTool}

# worker tool set
worker_tools = all_tools - {AgentTool, TeamCreateTool, TeamDeleteTool,
                             SendMessageTool, SyntheticOutputTool, CronCreateTool}
```

### 11.7 Bidirectional TUI ↔ Agent Communication (CommandQueue)

In interactive sessions the UI and the agent run on separate threads/tasks. A shared
command queue enables the UI to inject messages into the agent's next turn while a tool
is still executing — without interrupting the current stream:

```
TUI thread                       Agent task
    │                                │
    │ user types during tool run     │
    │                                │
    ├── push(QueuedCommand) ────────►│ drain_queue() at start of next turn
    │   priority: Normal/High        │
    │                                │ inject as user message before API call
```

High-priority commands (e.g., `/stop`, `/compact`) are prepended to the message history;
normal-priority commands are appended. This avoids requiring the user to wait for the
current tool to finish before their input is acknowledged.

---


---

*[← Memory, RAG & Knowledge Graphs](01-memory-rag-knowledge.md) | [Next: Advanced Patterns →](01-advanced-patterns.md)*
