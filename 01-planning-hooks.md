# Universal Agent Architecture — Planning Mode & Hooks

> Sections: Planning Mode · Hooks System (Full Reference)

Part of the [Universal Agent Architecture](01-overview.md) series.

## 19. Planning Mode

Planning mode separates **safe exploration and plan generation** from **write-capable execution**, with an explicit user-approval gate between phases. It is the highest-leverage UX feature for reducing irreversible mistakes.

---

### 19.1 Core Concept

Without planning mode, the agent reads context and immediately starts making changes. With planning mode:

```
Plan Phase (read-only)          Approval Gate          Execute Phase (write-enabled)
──────────────────────    ──────────────────────    ──────────────────────────────────
Read files                 User reviews plan         Write files
Explore codebase      →    Edits plan if needed  →   Run commands
Generate plan file         Approves / rejects         Verify changes
(no writes)                (with feedback)            (read-only)
```

The key invariant: **no write operations occur during plan phase**. The permission system enforces this, not prompts alone.

---

### 19.2 Two Planning Mechanisms

#### A. Tool-Based Plan Mode (rich UX, recommended for interactive agents)

The agent has two tools: `EnterPlanMode` and `ExitPlanMode`.

```
User: "Refactor the auth module"
  │
  ▼
Agent calls EnterPlanMode()
  → permission_mode switches to "plan" (read-only enforcement)
  → exploration sub-agents spin up (parallel codebase reads)
  │
  ▼
Plan Phase loop (read-only tools only)
  → read_file, glob_search, grep_search allowed
  → write_file, bash(write), edit_file → DENIED
  → agent writes plan to plan file (plans directory only exception)
  │
  ▼
Agent calls ExitPlanMode(plan_filename="auth-refactor.md")
  → triggers user approval dialog
  → user reads plan, optionally edits, approves or rejects with feedback
  │
  ┌─── rejected ───► agent revises plan, returns to Plan Phase
  │
  ▼ approved
Execute Phase
  → permission_mode restored to "default"/"yolo"
  → write operations now allowed
  → agent implements following the plan
```

**Plan file schema** (structured markdown):
```markdown
# Plan: Auth Module Refactor

## Phase 1 — Exploration Findings
- `src/auth/middleware.ts` (lines 45–120): session token stored in-memory, not encrypted
- `src/auth/types.ts` (lines 12–34): AuthToken interface missing expiry field

## Phase 2 — Design Approach
Replace in-memory store with Redis. Add expiry to AuthToken. Update middleware.

## Phase 3 — Implementation Steps
- [ ] Add `expiry: Date` to `AuthToken` in `src/auth/types.ts:12`
- [ ] Replace `Map<string, Token>` with `RedisClient` in `src/auth/store.ts`
- [ ] Update `validateToken()` in `src/auth/middleware.ts:67` to check expiry
- [ ] Run: `npm test -- src/auth/`

## Phase 4 — Verification
Single command: `npm test -- src/auth/ && npm run lint`
```

**Tool declarations:**

```python
PLAN_TOOLS = [
    {
        "name": "enter_plan_mode",
        "description": "Switch to planning mode. All write operations will be blocked until the plan is approved. Use when the task is complex enough to warrant exploration before changes.",
        "input_schema": {
            "type": "object",
            "properties": {
                "reason": {"type": "string", "description": "Why planning mode is needed"}
            },
            "required": ["reason"]
        }
    },
    {
        "name": "exit_plan_mode",
        "description": "Submit the plan for user approval. Blocks until user approves or rejects. On rejection, returns feedback and stays in plan mode.",
        "input_schema": {
            "type": "object",
            "properties": {
                "plan_filename": {"type": "string", "description": "Path to the plan file"},
                "summary": {"type": "string", "description": "One-paragraph summary of the plan"}
            },
            "required": ["plan_filename", "summary"]
        }
    }
]
```

#### B. Extended Thinking (lightweight, no separate tools needed)

Enable the API's extended thinking feature. The model reasons internally before emitting tool calls. No permission switching, no plan file, no approval gate — but the reasoning is hidden (or optionally shown).

```python
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=16000,
    thinking={"type": "enabled", "budget_tokens": 10000},  # reserve tokens for thinking
    tools=tools,
    messages=messages
)

# Response contains interleaved thinking and tool_use blocks:
# [ThinkingBlock(thinking="Let me analyze..."), ToolUseBlock(name="read_file",...), ...]

for block in response.content:
    if block.type == "thinking":
        # Optionally display: print(f"[thinking] {block.thinking}")
        pass  # do not inject back into messages — API handles it
    elif block.type == "tool_use":
        result = execute_tool(block.name, block.input)
        tool_results.append({"type": "tool_result", "tool_use_id": block.id, "content": result})
```

**Thinking budget guidelines:**
```
Light reasoning (routing, classification): 1,000–2,000 tokens
Standard tasks (code change, debugging): 5,000–8,000 tokens
Complex tasks (architecture, security audit): 10,000–16,000 tokens

Rule: budget_tokens + max_tokens must not exceed model context limit.
Extended thinking requires min budget_tokens = 1,024.
```

---

### 19.3 Permission Enforcement During Plan Phase

The permission system must enforce read-only during plan mode — prompt instructions alone are not sufficient.

```python
PLAN_MODE_ALLOWED = {"read_file", "glob_search", "grep_search", "list_dir", "web_fetch"}
PLAN_MODE_WRITE_EXCEPTION = {"write_file"}  # only to plans directory

def check_permission_plan_mode(tool: str, inp: dict, plans_dir: str) -> tuple[bool, str]:
    if tool in PLAN_MODE_ALLOWED:
        return True, ""
    if tool == "write_file":
        path = inp.get("path", "")
        if os.path.realpath(path).startswith(os.path.realpath(plans_dir)):
            return True, ""
        return False, "Write blocked in plan mode: only plans directory is writable"
    return False, f"Tool '{tool}' is not allowed in plan mode"
```

---

### 19.4 Multi-Agent Plan Exploration

For large codebases, spawn multiple read-only sub-agents in parallel during plan phase to gather context faster:

```
EnterPlanMode()
     │
     ├── Explore Agent 1: "find all auth-related files"
     ├── Explore Agent 2: "find all tests for auth module"  
     └── Explore Agent 3: "check git log for recent auth changes"
     │
     ▼ (all three complete)
Plan Agent: synthesizes findings → writes plan file
     │
ExitPlanMode(plan_filename="...", summary="...")
```

**Sub-agent count guidelines:**
- 1–3 explore agents for typical codebases
- 1–3 plan agents for complex tasks requiring multiple design options
- Cap at 3 of each to avoid context explosion in the synthesis step

---

### 19.5 Task/Todo Tracking Inside Execute Phase

After plan approval, convert the plan's step list into a tracked task list. The model updates tasks as it works, giving the user live progress visibility.

```python
TASK_SCHEMA = {
    "id": str,           # UUID
    "title": str,        # human-readable description
    "status": Literal["pending", "in_progress", "completed", "failed", "skipped"],
    "priority": Literal["low", "medium", "high", "critical"],
    "created_at": str,   # ISO timestamp
    "completed_at": str, # ISO timestamp or None
}
```

**TodoWrite tool** (model calls this to update its task list):
```python
{
    "name": "todo_write",
    "description": "Update your task list. Call after completing each step to show progress.",
    "input_schema": {
        "type": "object",
        "properties": {
            "todos": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "id": {"type": "string"},
                        "content": {"type": "string"},
                        "status": {"type": "string", "enum": ["pending","in_progress","completed","failed"]},
                        "priority": {"type": "string", "enum": ["low","medium","high","critical"]}
                    },
                    "required": ["id", "content", "status", "priority"]
                }
            }
        },
        "required": ["todos"]
    }
}
```

---

### 19.6 Planning Mode Decision Matrix

| Scenario | Recommended approach |
|----------|---------------------|
| Simple one-file change | No plan mode — direct execution |
| Multi-file refactor (>3 files) | Tool-based plan mode |
| Architecture change or new feature | Tool-based plan mode + parallel explore agents |
| Automated pipeline (no human in loop) | Extended thinking only |
| Security/compliance audit | Extended thinking + plan mode for remediation steps |
| Domain agent with known safe ops | Disable plan mode; use permission rules instead |

---

### 19.7 ScratchpadGate (Reasoning Before Delegation)

In Manager-Executor topologies (see §11.5), block the manager from touching files until it has finished reasoning. Implement by watching the output stream for an unlock phrase:

```python
class ScratchpadGate:
    """Block write tools until model emits the unlock phrase in its text stream."""
    
    UNLOCK_PHRASE = "--- BEGIN EXECUTION ---"
    
    def __init__(self):
        self._unlocked = False
        self._buffer = ""
    
    def feed(self, text_delta: str):
        self._buffer += text_delta
        if self.UNLOCK_PHRASE in self._buffer:
            self._unlocked = True
    
    def is_write_allowed(self) -> bool:
        return self._unlocked
    
    def check(self, tool_name: str, write_tools: set) -> tuple[bool, str]:
        if tool_name not in write_tools:
            return True, ""
        if not self._unlocked:
            return False, "ScratchpadGate: complete reasoning before executing (emit '--- BEGIN EXECUTION ---')"
        return True, ""
```

System prompt instruction to pair with the gate:
```
Before modifying any files, write your complete plan in plain text.
When you have finished planning and are ready to start making changes,
emit exactly: --- BEGIN EXECUTION ---
Only after that line will write operations be permitted.
```

---

## Reading Order

**First implementation (start here):**
1. This document — Sections 1–5 (runtime model, provider registry, layers, prompt, permissions)
2. `00-research-foundation.md` (paper details)
3. Language `00-index.md` (quick-start)
4. Language `02-core-agent-loop.md` (turn loop)
5. Language `03-tool-system.md` (tools + permissions)
6. Language `09-session-persistence.md` (compaction — implement all three strategies)

**Adding intelligence (after basic loop works):**
7. This document — Section 6 (memory: MemGPT + memdir + session extraction + AutoDream)
8. Language `04-memory-and-rag.md`
9. Language `05-knowledge-graph.md`
10. Language `08-self-improvement.md`

**Multi-provider support:**
11. This document — Section 2 (provider registry + stream resilience)

**Multi-agent systems:**
12. This document — Sections 10.1–10.7 (all topologies including Manager-Executor)
13. Language `07-multiagent.md`
14. Language `06-event-driven.md`

**Planning mode:**
15. This document — Section 19 (tool-based plan mode, extended thinking, ScratchpadGate)

**Hooks, sessions, cost:**
16. This document — Sections 20, 21, 22 (hooks full reference, session management, cost tracking)

**IDE integration & distributed agents:**
17. This document — Sections 23, 24 (IDE integration, A2A distributed agents)

**LSP, plugins, SDK, MCP:**
18. This document — Sections 25–28 (LSP, plugins, SDK embedding, MCP reference)

**Production deployment:**
19. Language `01-architecture.md`
20. This document — Section 15 (config schema)
21. Language `10-prompts.md`
22. Language `11-complete-example.md`
23. `python-langchain/12-langsmith-observability.md`

---

## Language Quick-Start

### Python + LangGraph (5 minutes)

```bash
pip install langchain langgraph langchain-openai python-dotenv
```

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

@tool
def bash(command: str) -> str:
    """Run a shell command."""
    import subprocess
    r = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=30)
    return r.stdout + r.stderr

agent = create_react_agent(ChatOpenAI(model="gpt-4o-mini"), tools=[bash])
result = agent.invoke({"messages": [{"role": "user", "content": "List Python files here"}]})
print(result["messages"][-1].content)
```

→ See `python-langchain/00-index.md` for the full guide.

### Kotlin + Koog (5 minutes)

```bash
# Prerequisites: JDK 17+, Gradle 8+
```

```kotlin
// build.gradle.kts
implementation("ai.koog:koog-agents:0.8.0")
```

```kotlin
suspend fun main() {
    val executor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY"))
    val agent = AIAgent(executor = executor,
                        systemPrompt = "You are a coding assistant.",
                        llmModel = OpenAIModels.Chat.GPT4o)
    println(agent.run("List Python files in the current directory"))
}
```

→ See `kotlin-koog/00-index.md` for the full guide.

### C++23 (15 minutes)

```bash
# Prerequisites: CMake ≥ 3.28, Clang 17+ or GCC 13+, vcpkg
cmake -B build -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake
cmake --build build -j$(nproc)
ANTHROPIC_API_KEY=sk-ant-... ./build/agent "List Python files here"
```

```cmake
# vcpkg.json dependencies
{ "dependencies": ["nlohmann-json", "cpp-httplib", "spdlog", "usearch",
                    "boost-cobalt", "sqlite3", "sqlitecpp"] }
```

→ See `cpp23/00-index.md` for the full guide.

---

## Contributing a New Language Guide

When adding guidelines for a new language or framework:

1. Create `{language}/` directory with the same 12-file structure:
   `00-index`, `01-architecture`, `02-core-agent-loop`, `03-tool-system`,
   `04-memory-and-rag`, `05-knowledge-graph`, `06-event-driven`, `07-multiagent`,
   `08-self-improvement`, `09-session-persistence`, `10-prompts`, `11-complete-example`
2. Include all 25 arXiv paper citations in `00-index.md`
3. Add the language to the Cross-Language Implementation Map in Section 16
4. Add the language to the Decision Guide table in Section 16.1
5. Update `README.md` with the new directory entry and quick-start snippet

---

## 20. Hooks System (Full Reference)

Hooks are shell commands, HTTP endpoints, LLM prompts, or function callbacks that fire
at lifecycle events. They allow external tools to observe, validate, modify, or veto
agent actions without changing core agent code.

### 20.1 Hook Event Types

| Event | When it fires |
|-------|---------------|
| `PreToolUse` | Before permission check; can deny or modify input |
| `PostToolUse` | After tool succeeds; side-effects only |
| `PostToolUseFailure` | After tool errors; for alerting/logging |
| `UserPromptSubmit` | When user sends a message; can augment context |
| `SessionStart` | Session initialization |
| `SessionEnd` / `Stop` | Session teardown |
| `StopFailure` | Session ended with unhandled error |
| `PreCompact` | Before context compaction; can inject preservation hints |
| `PostCompact` | After compaction completes |
| `SubagentStart` | Before spawning a sub-agent |
| `SubagentStop` | Sub-agent completes or errors |
| `PermissionRequest` | When permission engine is about to ASK user |
| `PermissionDenied` | After a tool call is denied |
| `TaskCreated` | A task/todo entry is created |
| `TaskCompleted` | A task/todo entry is marked done |
| `Notification` | Agent emits a user notification |
| `FileChanged` | A file in the project changes (requires file watcher) |
| `CwdChanged` | Working directory changes |
| `WorktreeCreate` / `WorktreeRemove` | Git worktree lifecycle |

### 20.2 Hook Types

| Type | Mechanism | Use case |
|------|-----------|----------|
| `command` | Shell command; stdin/stdout JSON | Linting, audit logging, custom deny logic |
| `http` | POST JSON to a URL | Webhook notifications, external approval systems |
| `prompt` | LLM prompt with `$ARGUMENTS` placeholder | AI-driven permission decisions |
| `agent` | Spawn a full sub-agent to evaluate | Complex validation requiring tool use |
| `function` | In-process callback (SDK only) | Programmatic hooks in embedded agents |

### 20.3 Wire Protocol (command and http types)

**stdin payload (all events):**
```json
{
  "event": "PreToolUse",
  "tool": "Bash",
  "inputJson": {"command": "git status"},
  "sessionId": "abc-123",
  "workingDirectory": "/home/user/project"
}
```

**stdout response (PreToolUse only — can control execution):**
```json
{
  "allowed": true,
  "updatedInput": {"command": "git status --short"},
  "reason": "Normalized verbose flag"
}
```

| Response field | Effect |
|---------------|--------|
| `allowed: false` + `blocking: true` | Deny the tool call; inject `reason` as error |
| `allowed: false` + `blocking: false` | Warn user but allow (soft block) |
| `updatedInput: {...}` | Replace tool input with modified version |
| `(no stdout)` | Allow unconditionally |

**Exit code semantics (command type):**
- `0` — allow
- `2` — deny (if hook is blocking)
- Other non-zero — warning only

### 20.4 Hook Configuration

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(rm *)",
        "hooks": [
          {
            "type": "command",
            "command": "~/.agent/hooks/confirm_delete.sh",
            "blocking": true,
            "timeout": 10
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "https://audit.example.com/agent-events",
            "headers": {"Authorization": "Bearer ${AUDIT_TOKEN}"}
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.agent/hooks/session_report.sh",
            "once": true
          }
        ]
      }
    ]
  }
}
```

**Execution order:** Sequential within a matcher group; first non-allow result wins.

**`matcher` field syntax:** Same glob syntax as permission rules: `ToolName(input glob)`.
Omit matcher to run on all tool calls of that event type.

**Special flags:**
- `once: true` — hook fires once then is removed from the registry
- `asyncRewake: true` — hook runs in background; if it exits 2, re-enters the permission flow
- `timeout` — seconds before hook is killed (default varies by type; 10s for command, 60s for agent)

### 20.5 Example Hooks

**Audit logger (PostToolUse):**
```bash
#!/bin/bash
# Append every tool call to an audit log
INPUT=$(cat)
TOOL=$(echo "$INPUT" | jq -r '.tool')
CMD=$(echo "$INPUT" | jq -r '.inputJson.command // "n/a"')
echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) [$TOOL] $CMD" >> ~/.agent/audit.log
```

**Linter gate (PreToolUse on Write):**
```bash
#!/bin/bash
INPUT=$(cat)
PATH=$(echo "$INPUT" | jq -r '.inputJson.path')
if [[ "$PATH" == *.py ]]; then
  python -m py_compile "$PATH" 2>/dev/null || \
    echo '{"allowed":false,"reason":"Syntax error in file"}'; exit 2
fi
```

**Secrets scanner (PreToolUse on Write/Bash):**
```bash
#!/bin/bash
INPUT=$(cat)
CONTENT=$(echo "$INPUT" | jq -r '.inputJson.content // .inputJson.command // ""')
if echo "$CONTENT" | grep -qE 'sk-ant-[A-Za-z0-9]{20,}|ghp_[A-Za-z0-9]{36}'; then
  echo '{"allowed":false,"reason":"Possible secret detected in content"}'; exit 2
fi
```

---


---

**See also:** [Clear-Context-and-Execute · AskUserQuestion · Hooks Full Reference →](01-clear-context-plan-user-interaction.md) — source-verified deep dive into the clear-context workflow, AskUserQuestion schema, all user-interaction tools, and complete hook dispatch implementation guide.

*[← Config, Infra & Reference](01-config-infra-reference.md) | [Next: Sessions, Cost & IDE →](01-sessions-cost-ide.md)*
