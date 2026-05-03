# Clear-Context-and-Execute-Plan · User Interaction Tools · Hooks

> Sections: Clear-Context pattern · AskUserQuestion · User-interaction tools · Hooks full reference · External skill implementation guide

Part of the [Universal Agent Architecture](01-overview.md) series.

---

## 1. The "Clear Context and Execute Plan" Pattern

### 1.1 What it is

"Clear context and execute plan" is a two-phase session management pattern where the agent:

1. **Plans in a read-only context** — explores the codebase, asks clarifying questions, writes a plan file
2. **Clears the entire conversation history** — drops every previous turn so the execution phase starts with a fresh, uncluttered context
3. **Re-enters with only the approved plan** — the plan text becomes the first user message, permissions are elevated, and execution begins

The result: the execution phase sees the task described cleanly in the plan, not buried under pages of exploration chatter. This dramatically reduces the chance of confused or hallucinated execution that references stale plan-phase details.

```
┌─────────────────────── PLAN PHASE ────────────────────────┐
│  EnterPlanMode()                                           │
│    → mode = "plan" (all writes blocked)                   │
│    → context accumulates: Glob, Grep, Read calls,         │
│      AskUserQuestion exchanges, notes, dead ends          │
│                                                            │
│  Agent writes plan to plan file                           │
│  ExitPlanMode()                                           │
│    → user reviews/edits plan, approves                    │
└────────────────────────────────────────────────────────────┘
             │  clearContext = true
             ▼
┌─────────────────────── CONTEXT RESET ─────────────────────┐
│  1. Save plan slug (file path pointer)                     │
│  2. clearConversation() — wipe all messages               │
│  3. regenerateSessionId() — new session UUID               │
│  4. Restore plan slug for new session                      │
│  5. Set permission mode from prePlanMode (default/auto)   │
└────────────────────────────────────────────────────────────┘
             │  initialMessage = plan content
             ▼
┌─────────────────────── EXECUTE PHASE ─────────────────────┐
│  Fresh context, first message = approved plan             │
│  Permissions restored (write-capable)                     │
│  allowedPrompts applied (semantic permission grants)      │
│  Agent implements plan from scratch, no exploration noise  │
└────────────────────────────────────────────────────────────┘
```

### 1.2 Internal implementation (Claude Code source)

**Step 1 — ExitPlanMode tool fires** (`src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`):
```typescript
// After user approves the plan dialog:
context.setAppState(prev => {
  setHasExitedPlanMode(true)
  setNeedsPlanModeExitAttachment(true)
  const restoreMode = prev.toolPermissionContext.prePlanMode ?? 'default'
  return {
    ...prev,
    toolPermissionContext: {
      ...baseContext,
      mode: restoreMode,      // restore default/auto
      prePlanMode: undefined,
    },
  }
})
// Returns: { plan, isAgent, filePath, hasTaskTool, planWasEdited }
```

**Step 2 — REPL.tsx detects clearContext flag** (`src/screens/REPL.tsx:3037`):
```typescript
if (initialMsg.clearContext) {
  // Preserve plan slug BEFORE clearing (new session won't have it)
  const oldPlanSlug = initialMsg.message.planContent
    ? getPlanSlug()
    : undefined

  const { clearConversation } = await import('../commands/clear/conversation.js')
  await clearConversation({
    setMessages,
    readFileState: readFileState.current,
    discoveredSkillNames: discoveredSkillNamesRef.current,
    loadedNestedMemoryPaths: loadedNestedMemoryPathsRef.current,
    getAppState: () => store.getState(),
    setAppState,
    setConversationId,
  })

  haikuTitleAttemptedRef.current = false
  setHaikuTitle(undefined)
  bashTools.current.clear()
  bashToolsProcessedIdx.current = 0

  // Restore plan slug for new session so getPlan() finds the file
  if (oldPlanSlug) {
    setPlanSlug(getSessionId(), oldPlanSlug)
  }
}
```

**Step 3 — regenerateSessionId** (`src/bootstrap/state.ts:435`):
```typescript
export function regenerateSessionId(
  options: { setCurrentAsParent?: boolean } = {},
): SessionId {
  if (options.setCurrentAsParent) {
    STATE.parentSessionId = STATE.sessionId
  }
  // Drop outgoing session's plan-slug from cache
  STATE.planSlugCache.delete(STATE.sessionId)
  STATE.sessionId = randomUUID() as SessionId
  STATE.sessionProjectDir = null
  return STATE.sessionId
}
```

**InitialMessage type** (what carries the plan into the fresh execution context):
```typescript
initialMessage: {
  message: { content: string, planContent?: string, uuid: string, ... }
  clearContext?: boolean           // triggers the wipe
  mode?: PermissionMode            // permission mode to restore
  allowedPrompts?: AllowedPrompt[] // semantic permission grants from plan
}
```

**AllowedPrompt** — a semantic permission grant that the plan declared it needs:
```typescript
type AllowedPrompt = {
  tool: 'Bash'
  prompt: string  // e.g. "run unit tests", "install npm dependencies"
}
```

### 1.3 Plan file persistence

The plan is written to disk during the plan phase so it survives the context clear:

```
~/.config/claude/plans/<session-id>/<plan-slug>.md
```

- `getPlanSlug()` returns the slug for the current session
- `getPlanFilePath(agentId?)` returns the full path
- `getPlan(agentId?)` reads and returns plan content
- The slug is stored in a `Map<sessionId, planSlug>` (bounded cache)
- On context clear: old slug is read → `clearConversation()` wipes session state → slug re-registered under new session ID

### 1.4 Reproducing the pattern in your own agent

```python
import json, os, uuid
from pathlib import Path
from anthropic import Anthropic

client = Anthropic()

PLANS_DIR = Path.home() / ".myagent" / "plans"
PLANS_DIR.mkdir(parents=True, exist_ok=True)


class PlanContextManager:
    """Implements clear-context-and-execute-plan for a custom agent."""

    def __init__(self):
        self.session_id = str(uuid.uuid4())
        self.plan_path: Path | None = None
        self.conversation: list = []

    # ── Phase 1: Plan ────────────────────────────────────────────────────
    def run_plan_phase(self, task: str) -> str:
        """Read-only exploration + plan writing. Returns plan content."""
        system = self._plan_system_prompt(task)
        # ... streaming loop with read-only tools only ...
        # Agent writes plan to self.plan_path
        plan = self.plan_path.read_text() if self.plan_path else ""
        return plan

    def write_plan(self, content: str) -> str:
        """Tool: write plan to disk. Only allowed during plan phase."""
        slug = f"plan-{self.session_id[:8]}"
        self.plan_path = PLANS_DIR / f"{slug}.md"
        self.plan_path.write_text(content)
        return f"Plan saved to {self.plan_path}"

    # ── Context Reset ────────────────────────────────────────────────────
    def clear_context(self):
        """Wipe conversation history, regenerate session ID."""
        old_plan_path = self.plan_path          # preserve
        self.conversation = []                  # clear history
        self.session_id = str(uuid.uuid4())     # new session
        self.plan_path = old_plan_path          # restore pointer

    # ── Phase 2: Execute ─────────────────────────────────────────────────
    def run_execute_phase(self, plan: str):
        """Fresh context; plan is the only prior context."""
        self.clear_context()
        # Seed conversation with the approved plan only
        self.conversation = [{"role": "user", "content": f"Execute this approved plan:\n\n{plan}"}]
        system = self._execute_system_prompt()
        # ... streaming loop with full write-capable tools ...

    def _plan_system_prompt(self, task: str) -> str:
        return f"""You are in PLAN MODE. Task: {task}
Write-operations are blocked. Read files, explore, then write your plan.
When done, call write_plan() with the complete implementation plan."""

    def _execute_system_prompt(self) -> str:
        return "Execute the plan. Write files, run commands. Update todos as you go."
```

**TypeScript version:**
```typescript
class PlanContextManager {
  private sessionId = crypto.randomUUID()
  private planPath: string | null = null
  private conversation: Message[] = []

  async clearContext(): Promise<void> {
    const savedPlan = this.planPath      // preserve disk pointer
    this.conversation = []               // wipe history
    this.sessionId = crypto.randomUUID()
    this.planPath = savedPlan            // restore
  }

  async runExecutePhase(plan: string): Promise<void> {
    await this.clearContext()
    // First message is ONLY the approved plan
    this.conversation = [{ role: 'user', content: `Execute this plan:\n\n${plan}` }]
    // run normal agent loop with write tools enabled
  }
}
```

---

## 2. AskUserQuestion Tool

### 2.1 Purpose and design

`AskUserQuestion` is a blocking interactive tool that presents structured multiple-choice questions to the user and returns their answers. It is the primary mechanism for the agent to clarify requirements, gather preferences, or offer implementation choices **without needing a full plan-mode approval cycle**.

Key design decisions:
- Up to **4 questions per call** (batched to reduce interruptions)
- **2–4 options per question** (keeps choices scannable)
- An **"Other" option** is always appended automatically — never add it yourself
- Optional **preview** field: side-by-side rendering for code/mockup comparisons
- Optional **multiSelect** for non-mutually-exclusive choices
- Disabled when running in channel mode (Telegram/Discord) where no terminal is available

### 2.2 Input schema

```typescript
// Full TypeScript schema from src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx

type QuestionOption = {
  label: string         // Concise (1-5 words), the selectable choice
  description: string   // Explanation of implications / trade-offs
  preview?: string      // Optional: markdown or HTML to render side-by-side
}

type Question = {
  question: string      // Full question text, ends with "?"
  header: string        // Chip label, max 12 chars (e.g., "Auth method")
  options: QuestionOption[] // 2–4 options (Other is automatic)
  multiSelect?: boolean  // Default false; true = comma-separated answers
}

type AskUserQuestionInput = {
  questions: Question[]  // 1–4 questions per call
  // Internal fields (injected by permission component):
  answers?: Record<string, string>      // question text → answer string
  annotations?: Record<string, {        // optional per-question notes
    preview?: string
    notes?: string
  }>
  metadata?: { source?: string }        // analytics tracking only
}

type AskUserQuestionOutput = {
  questions: Question[]
  answers: Record<string, string>  // multi-select answers are comma-separated
  annotations?: ...
}
```

### 2.3 Usage rules (from source prompt)

| Rule | Detail |
|------|--------|
| Don't add "Other" | It is injected automatically |
| Recommend an option | Prepend `"(Recommended)"` to the `label` field and make it `options[0]` |
| Plan mode usage | Use to clarify requirements BEFORE writing the plan. NOT for "is the plan ready?" (use `ExitPlanMode` for that) |
| Don't ask about the plan | User can't see the plan until `ExitPlanMode` is called |
| Preview: single-select only | `preview` field is ignored when `multiSelect: true` |
| Preview: markdown format | Set for code snippets, ASCII mockups, config examples |
| Preview: HTML format | Self-contained fragment only; no `<html>`/`<body>`/`<script>`/`<style>` tags |

### 2.4 Implementation examples

**Simple preference question:**
```python
{
  "name": "AskUserQuestion",
  "input": {
    "questions": [
      {
        "question": "Which state management approach should we use?",
        "header": "State mgmt",
        "options": [
          {
            "label": "Redux Toolkit (Recommended)",
            "description": "Battle-tested, DevTools support, works well with the existing auth slice"
          },
          {
            "label": "Zustand",
            "description": "Lighter footprint, simpler API, good for this component's local state"
          },
          {
            "label": "React Context",
            "description": "Zero deps, but re-render scope is wider — may need memoization"
          }
        ]
      }
    ]
  }
}
```

**Multi-select features question:**
```python
{
  "questions": [
    {
      "question": "Which features should be included in the initial release?",
      "header": "Features",
      "multiSelect": True,
      "options": [
        {"label": "Dark mode", "description": "CSS variable theme toggle"},
        {"label": "Offline support", "description": "Service worker + IndexedDB cache"},
        {"label": "Push notifications", "description": "Requires VAPID key setup"},
        {"label": "Export to PDF", "description": "Uses puppeteer; adds ~15 MB to bundle"}
      ]
    }
  ]
}
```

**Code preview comparison:**
```python
{
  "questions": [
    {
      "question": "Which API error handling style should we standardize on?",
      "header": "Error style",
      "options": [
        {
          "label": "Result type",
          "description": "Explicit Ok/Err wrapping, forces callers to handle errors",
          "preview": "async function fetchUser(id: string): Promise<Result<User, ApiError>> {\n  const res = await fetch(`/api/users/${id}`)\n  if (!res.ok) return err({ code: res.status, message: await res.text() })\n  return ok(await res.json())\n}"
        },
        {
          "label": "Throw on error",
          "description": "Conventional try/catch, simpler call sites",
          "preview": "async function fetchUser(id: string): Promise<User> {\n  const res = await fetch(`/api/users/${id}`)\n  if (!res.ok) throw new ApiError(res.status, await res.text())\n  return res.json()\n}"
        }
      ]
    }
  ]
}
```

### 2.5 Python tool declaration for API usage

```python
ASK_USER_QUESTION_TOOL = {
    "name": "AskUserQuestion",
    "description": (
        "Asks the user multiple choice questions to gather information, clarify "
        "ambiguity, understand preferences, make decisions or offer choices. "
        "Use this when you need input from the user during execution."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "questions": {
                "type": "array",
                "minItems": 1,
                "maxItems": 4,
                "description": "Questions to ask (1–4 per call)",
                "items": {
                    "type": "object",
                    "required": ["question", "header", "options"],
                    "properties": {
                        "question": {
                            "type": "string",
                            "description": "Clear question ending with '?'"
                        },
                        "header": {
                            "type": "string",
                            "maxLength": 12,
                            "description": "Short chip label, e.g. 'Auth method'"
                        },
                        "options": {
                            "type": "array",
                            "minItems": 2,
                            "maxItems": 4,
                            "description": "Do NOT include an 'Other' option — it is automatic",
                            "items": {
                                "type": "object",
                                "required": ["label", "description"],
                                "properties": {
                                    "label": {"type": "string"},
                                    "description": {"type": "string"},
                                    "preview": {
                                        "type": "string",
                                        "description": "Markdown content rendered side-by-side"
                                    }
                                }
                            }
                        },
                        "multiSelect": {
                            "type": "boolean",
                            "default": False,
                            "description": "True to allow multiple answers (comma-separated)"
                        }
                    }
                }
            }
        },
        "required": ["questions"]
    }
}


def handle_ask_user_question(questions: list[dict]) -> dict:
    """
    Terminal implementation of AskUserQuestion.
    In a real agent harness replace with your UI component.
    """
    answers = {}
    for q in questions:
        print(f"\n[{q['header']}] {q['question']}")
        for i, opt in enumerate(q['options'], 1):
            print(f"  {i}. {opt['label']} — {opt['description']}")
        print(f"  {len(q['options'])+1}. Other (custom input)")

        if q.get('multiSelect'):
            raw = input("Select (comma-separated numbers): ")
            selected = []
            for n in raw.split(','):
                n = n.strip()
                try:
                    idx = int(n) - 1
                    if 0 <= idx < len(q['options']):
                        selected.append(q['options'][idx]['label'])
                    else:
                        selected.append(input("Custom: "))
                except ValueError:
                    pass
            answers[q['question']] = ', '.join(selected)
        else:
            n = int(input("Select: ").strip())
            if 1 <= n <= len(q['options']):
                answers[q['question']] = q['options'][n-1]['label']
            else:
                answers[q['question']] = input("Custom: ")

    return {"questions": questions, "answers": answers}
```

---

## 3. Other Built-in User-Interaction Tools

### 3.1 Tool inventory

| Tool | Purpose | Blocks execution | Channel-safe |
|------|---------|-----------------|--------------|
| `AskUserQuestion` | Multi-choice Q&A | Yes (awaits answer) | No |
| `EnterPlanMode` | Request read-only plan phase | Yes (awaits consent) | No |
| `ExitPlanMode` | Submit plan for approval | Yes (awaits approve/reject) | No |
| `PushNotification` | Desktop/mobile notification | No (fire-and-forget) | Yes |
| `Monitor` | Stream long-running process events | No (async stream) | Yes |
| `TaskCreate` / `TaskUpdate` / `TaskList` | Progress tracking | No | Yes |
| `SendMessage` | Send message to teammate/agent | No | Yes |

### 3.2 EnterPlanMode

**When to use:** Proactively before non-trivial implementation tasks. Signals the user that you want to explore before coding.

```python
ENTER_PLAN_MODE_TOOL = {
    "name": "EnterPlanMode",
    "description": (
        "Transitions into plan mode where write operations are blocked. "
        "Use proactively for tasks with architectural ambiguity, multiple "
        "valid approaches, or significant multi-file changes."
    ),
    "input_schema": {"type": "object", "properties": {}, "required": []}
}
```

**Decision matrix:**

| Trigger | Use EnterPlanMode? |
|---------|--------------------|
| New feature with architectural choices | Yes |
| Multi-file refactor (>3 files) | Yes |
| Simple bug fix, obvious solution | No |
| User said "just do it" / "can we work on X" | No |
| Research/exploration only | No — use Agent tool with read-only tools |
| You'd call `AskUserQuestion` for approach clarification | Yes — plan mode is better (explore first, then ask) |

### 3.3 ExitPlanMode (V2)

**Input schema:**
```typescript
{
  allowedPrompts?: Array<{
    tool: 'Bash'
    prompt: string  // e.g. "run unit tests", "install npm dependencies"
  }>
}
```

**Behavior:**
1. Reads plan from disk (plan file written during plan phase)
2. Shows plan to user in approval dialog
3. User can edit the plan directly in the dialog (Ctrl+G in terminal)
4. On approval: sets `clearContext=true`, restores `prePlanMode`, returns plan content
5. On rejection: stays in plan mode, returns rejection feedback

**Tool result (on approval):**
```
User has approved your plan. You can now start coding.
Start with updating your todo list if applicable.

Your plan has been saved to: /home/user/.config/claude/plans/session-xyz/plan.md

## Approved Plan:
[full plan content]
```

**Teammate mode:** Instead of showing approval UI, sends a `plan_approval_request` message to the team-lead mailbox and waits for approval via the messaging system.

### 3.4 PushNotification

Available in KAIROS/Channels deployments. Sends a device notification without blocking.

```python
PUSH_NOTIFICATION_TOOL = {
    "name": "PushNotification",
    "description": "Send a proactive notification to the user's device. Use when completing a long task that the user is waiting on.",
    "input_schema": {
        "type": "object",
        "required": ["message", "status"],
        "properties": {
            "message": {"type": "string", "description": "Notification text"},
            "status": {"type": "string", "enum": ["proactive"]}
        }
    }
}
```

### 3.5 Monitor

Streams events from a long-running background process. Each stdout line becomes a notification in the conversation.

```python
MONITOR_TOOL = {
    "name": "Monitor",
    "description": "Watch a long-running process and stream its output as events.",
    "input_schema": {
        "type": "object",
        "required": ["command"],
        "properties": {
            "command": {"type": "string"},
            "description": {"type": "string"},
            "timeout_ms": {"type": "number", "default": 300000},
            "persistent": {"type": "boolean", "default": False}
        }
    }
}
```

---

## 4. Hooks — Full Reference

### 4.1 Event types (complete list from source)

```
src/entrypoints/sdk/coreSchemas.ts — HOOK_EVENTS array
```

| Event | When it fires | Can deny/modify |
|-------|--------------|----------------|
| `PreToolUse` | Before permission check; earliest interception point | Yes |
| `PostToolUse` | After tool succeeds | No (side-effects only) |
| `PostToolUseFailure` | After tool errors | No |
| `UserPromptSubmit` | When user sends a message | Can augment context |
| `SessionStart` | Session initialization | No |
| `SessionEnd` | Session graceful teardown | No |
| `Stop` | Before Claude stops responding (fires after each response) | Can reopen |
| `StopFailure` | Session ended with unhandled error | No |
| `SubagentStart` | Before spawning a sub-agent | No |
| `SubagentStop` | Sub-agent completes or errors | No |
| `PreCompact` | Before context compaction | Can inject preservation hints |
| `PostCompact` | After compaction completes | No |
| `PermissionRequest` | Permission engine about to ask user | Can auto-approve |
| `PermissionDenied` | Tool call was denied | No |
| `Setup` | One-time harness setup | No |
| `TeammateIdle` | Multi-agent teammate has nothing to do | No |
| `TaskCreated` | Task/todo created | No |
| `TaskCompleted` | Task/todo marked done | No |
| `Elicitation` | Agent is about to elicit user input | Can provide answer |
| `ElicitationResult` | Elicitation answered | No |
| `ConfigChange` | Settings changed at runtime | No |
| `WorktreeCreate` | Git worktree created | No |
| `WorktreeRemove` | Git worktree removed | No |
| `InstructionsLoaded` | CLAUDE.md / skill instructions loaded | No |
| `CwdChanged` | Working directory changed | No |
| `FileChanged` | File in project changed (requires watcher) | No |

**Important for agents:** When a skill registers a `Stop` hook, it is automatically converted to `SubagentStop` for sub-agents, because sub-agents fire `SubagentStop` (not `Stop`) when they complete.

### 4.2 Hook types

#### command — Shell command hook

```json
{
  "type": "command",
  "command": "~/.agent/hooks/my_hook.sh",
  "if": "Bash(rm *)",
  "shell": "bash",
  "timeout": 10,
  "statusMessage": "Checking deletion...",
  "once": false,
  "async": false,
  "asyncRewake": false
}
```

| Field | Type | Description |
|-------|------|-------------|
| `command` | string | Shell command; receives JSON on stdin |
| `if` | string? | Permission-rule syntax filter (e.g., `"Bash(git *)"`) — skip hook if tool call doesn't match |
| `shell` | `"bash"` \| `"powershell"` | Shell interpreter (default: bash/$SHELL) |
| `timeout` | number? | Seconds before kill (default: 10) |
| `statusMessage` | string? | Spinner text shown to user |
| `once` | bool? | Unregister after first execution |
| `async` | bool? | Non-blocking background execution |
| `asyncRewake` | bool? | Background; if exit 2, re-enters permission flow |

**stdin payload:**
```json
{
  "event": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {"command": "rm -rf /tmp/cache"},
  "session_id": "abc-123",
  "transcript_path": "/home/user/.claude/projects/.../session.jsonl",
  "cwd": "/home/user/project",
  "permission_mode": "default"
}
```

**stdout response (PreToolUse only):**
```json
{
  "allowed": false,
  "reason": "rm with -rf is blocked by policy",
  "blocking": true
}
```

Exit codes: `0` = allow, `2` = deny (if blocking), other non-zero = warning.

#### prompt — LLM evaluation hook

```json
{
  "type": "prompt",
  "prompt": "The agent wants to run: $ARGUMENTS\nIs this safe? Reply with 'ALLOW' or 'DENY: <reason>'",
  "model": "claude-haiku-4-5-20251001",
  "timeout": 30,
  "statusMessage": "AI safety check...",
  "once": false
}
```

Use `$ARGUMENTS` placeholder — it is replaced with the JSON hook input at runtime. The prompt hook's response is parsed for allow/deny signals.

#### http — Webhook hook

```json
{
  "type": "http",
  "url": "https://audit.example.com/agent-events",
  "headers": {
    "Authorization": "Bearer $AUDIT_TOKEN",
    "X-Agent-Session": "$SESSION_ID"
  },
  "allowedEnvVars": ["AUDIT_TOKEN", "SESSION_ID"],
  "timeout": 5,
  "statusMessage": "Logging...",
  "once": false
}
```

**Note on env vars:** Only variables listed in `allowedEnvVars` are interpolated in header values. All other `$VAR` references remain as empty strings. This is a security control.

#### agent — Sub-agent verification hook

```json
{
  "type": "agent",
  "prompt": "Verify that unit tests ran and passed. Check the output for FAIL or ERROR lines.",
  "model": "claude-haiku-4-5-20251001",
  "timeout": 60,
  "statusMessage": "Verifying tests...",
  "once": true
}
```

The hook spawns a full sub-agent with tool access. Use for complex validation that requires actual tool calls (reading files, running checks, etc.).

#### function — In-process callback (SDK embedding only)

```typescript
// Only available when embedding Claude Code as a library (not in settings.json)
const functionHook: FunctionHook = {
  type: 'function',
  id: 'my-gate',
  timeout: 10,
  errorMessage: 'Custom gate blocked this action',
  statusMessage: 'Running custom check...',
  callback: async (messages, signal) => {
    // Return true to allow, false to block
    const lastMsg = messages[messages.length - 1]
    return !JSON.stringify(lastMsg).includes('FORBIDDEN_PATTERN')
  }
}
```

### 4.3 Hook configuration schema

In `settings.json` (project: `.claude/settings.json`, user: `~/.config/claude/settings.json`):

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
            "timeout": 10
          }
        ]
      },
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "~/.agent/hooks/secrets_scan.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "https://audit.example.com/events",
            "headers": {"Authorization": "Bearer $TOKEN"},
            "allowedEnvVars": ["TOKEN"]
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
            "once": true,
            "async": true
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.agent/hooks/inject_context.sh"
          }
        ]
      }
    ]
  }
}
```

**`matcher` field syntax:** Same glob syntax as permission rules: `ToolName(input glob)`. Omit to match all tool calls of that event.

**Execution order:** Sequential within a matcher group; first deny result wins.

**`if` vs `matcher`:** Both filter which hook runs, but at different levels:
- `matcher` is on the `HookMatcher` object (coarse: which tool class)
- `if` is on the individual `HookCommand` (fine: tool + input pattern)
- Use `if` for precise input-level filtering without extra matcher entries

### 4.4 Production hook examples

**Audit logger (PostToolUse):**
```bash
#!/bin/bash
INPUT=$(cat)
TOOL=$(echo "$INPUT" | jq -r '.tool_name')
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // .tool_input.path // "n/a"')
SESSION=$(echo "$INPUT" | jq -r '.session_id')
echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) [$SESSION] [$TOOL] $CMD" >> ~/.agent/audit.log
```

**Secrets scanner (PreToolUse on Write and Bash):**
```bash
#!/bin/bash
INPUT=$(cat)
CONTENT=$(echo "$INPUT" | jq -r '.tool_input.content // .tool_input.command // ""')
if echo "$CONTENT" | grep -qE \
  'sk-ant-[A-Za-z0-9]{20,}|ghp_[A-Za-z0-9]{36}|AKIA[A-Z0-9]{16}|-----BEGIN.*PRIVATE KEY'; then
  echo '{"allowed":false,"reason":"Possible secret detected in output","blocking":true}'
  exit 2
fi
```

**Linter gate (PreToolUse on Write — Python files):**
```bash
#!/bin/bash
INPUT=$(cat)
FILE=$(echo "$INPUT" | jq -r '.tool_input.path // ""')
CONTENT=$(echo "$INPUT" | jq -r '.tool_input.content // ""')
if [[ "$FILE" == *.py ]]; then
  echo "$CONTENT" | python -m py_compile - 2>/dev/null
  if [ $? -ne 0 ]; then
    echo '{"allowed":false,"reason":"Python syntax error in file","blocking":true}'
    exit 2
  fi
fi
```

**Context injector (UserPromptSubmit) — adds current git status:**
```bash
#!/bin/bash
INPUT=$(cat)
GIT_STATUS=$(git status --short 2>/dev/null | head -20)
if [ -n "$GIT_STATUS" ]; then
  # Inject as additional context — harness reads this from stdout
  echo "{\"contextToAdd\": \"Current git status:\n${GIT_STATUS}\"}"
fi
```

**Post-compact memory preservation (PreCompact):**
```bash
#!/bin/bash
# Write a memory hint that will survive compaction
SESSION=$(cat /dev/stdin | jq -r '.session_id')
HINTS_FILE=~/.agent/compaction-hints.md
if [ -f "$HINTS_FILE" ]; then
  cat "$HINTS_FILE"
fi
```

---

## 5. External Skill Implementation Guide

### 5.1 How skills interact with user-interaction tools

**The key constraint:** Skills are prompts, not code. A skill's `getPromptForCommand()` returns text content that is injected into the model's context. The model (Claude) then decides to call tools. **Skills cannot directly invoke tools.**

```
User types: /my-skill
     │
     ▼
SkillTool.call()
  → skill.getPromptForCommand(args, context)
  → returns: [{ type: 'text', text: PROMPT_TEXT }]
     │
     ▼ PROMPT_TEXT injected into model context
     │
     ▼
Model processes prompt, decides to call:
  → AskUserQuestion(...)        ✓ model calls this
  → EnterPlanMode()             ✓ model calls this
  → Bash("git diff")            ✓ model calls this
  → ExitPlanMode(...)           ✓ model calls this
```

So to use `AskUserQuestion` from a skill: write a prompt that instructs Claude to call it.

### 5.2 Skill definition for bundled (in-process) skills

```typescript
// src/skills/bundled/my-workflow.ts
import { AGENT_TOOL_NAME } from '../../tools/AgentTool/constants.js'
import { registerBundledSkill } from '../bundledSkills.js'

const PROMPT = `# My Workflow Skill

## Step 1: Gather requirements
Call AskUserQuestion with these questions:
1. question: "Which database should we target?", header: "Database",
   options: [{label:"PostgreSQL",...},{label:"SQLite",...},{label:"MongoDB",...}]
2. question: "What performance tier do you need?", header: "Perf tier",
   options: [{label:"Development",...},{label:"Production",...}]

## Step 2: Enter plan mode
After gathering requirements, call EnterPlanMode() to begin exploration.

## Step 3: Plan implementation
Read the relevant source files, then write your plan to the plan file.
Call ExitPlanMode() when the plan is ready.

## Step 4: Execute
Implement the plan following the approved steps.
`

export function registerMyWorkflowSkill(): void {
  registerBundledSkill({
    name: 'my-workflow',
    description: 'Interactive workflow with requirement gathering and planning.',
    userInvocable: true,
    allowedTools: ['AskUserQuestion', 'EnterPlanMode', 'ExitPlanMode', 'Read', 'Bash', 'Write'],
    // Optional: register hooks that fire while this skill runs
    hooks: {
      PostToolUse: [{
        matcher: 'Write',
        hooks: [{
          type: 'command',
          command: '~/.agent/hooks/post_write_lint.sh',
        }]
      }]
    },
    async getPromptForCommand(args, _context) {
      let prompt = PROMPT
      if (args) prompt += `\n\n## User context\n${args}`
      return [{ type: 'text', text: prompt }]
    },
  })
}
```

### 5.3 Disk-based skills with hooks in frontmatter

Skills stored as `.md` files in `.claude/skills/` or `~/.config/claude/skills/` can declare hooks in YAML frontmatter:

```markdown
---
name: secure-writer
description: Write files with automatic secrets scanning
hooks:
  PreToolUse:
    - matcher: "Write"
      hooks:
        - type: command
          command: ~/.agent/hooks/secrets_scan.sh
          statusMessage: "Scanning for secrets..."
  PostToolUse:
    - hooks:
        - type: http
          url: https://audit.example.com/events
          headers:
            Authorization: "Bearer $AUDIT_TOKEN"
          allowedEnvVars: ["AUDIT_TOKEN"]
---

# Secure Writer

Always scan files for secrets before writing them. When you detect a potential
secret, use AskUserQuestion to confirm whether it's intentional before proceeding.

## Steps
1. Read the content you plan to write
2. Check for patterns like API keys, tokens, private keys
3. If suspicious patterns found: ask the user with AskUserQuestion
4. Write only after confirmation
```

Hook registration from frontmatter (`src/utils/hooks/registerFrontmatterHooks.ts`):
- Hooks are scoped to the current session
- `Stop` hooks are automatically converted to `SubagentStop` for sub-agents
- Hooks are cleaned up when the session/agent ends
- The skill's directory is available as `$CLAUDE_PLUGIN_ROOT` env var in hook commands

### 5.4 Replicating AskUserQuestion in a custom agent

If you're building an agent without the Claude Code harness, implement `AskUserQuestion` as a tool with its own handler:

```python
class UserInteractionLayer:
    """
    Drop-in replacement for AskUserQuestion in custom agents.
    Replace render_question() with your UI (web, Slack, terminal, etc.)
    """

    def handle_ask_user_question(self, tool_input: dict) -> dict:
        questions = tool_input['questions']
        answers: dict[str, str] = {}

        for q in questions:
            answer = self.render_question(q)
            answers[q['question']] = answer

        return {
            "questions": questions,
            "answers": answers
        }

    def render_question(self, q: dict) -> str:
        """Override for your UI. Default: terminal prompt."""
        print(f"\n{'─'*50}")
        print(f"[{q['header']}] {q['question']}")
        opts = q['options']
        for i, opt in enumerate(opts, 1):
            print(f"  {i}. {opt['label']}")
            if opt.get('description'):
                print(f"     {opt['description']}")
        print(f"  {len(opts)+1}. Other")

        if q.get('multiSelect'):
            raw = input("Select numbers (comma-separated): ")
            parts = [p.strip() for p in raw.split(',')]
            selected = []
            for p in parts:
                try:
                    idx = int(p) - 1
                    if 0 <= idx < len(opts):
                        selected.append(opts[idx]['label'])
                    else:
                        selected.append(input("Custom: ").strip())
                except ValueError:
                    selected.append(p)
            return ', '.join(selected)
        else:
            n = int(input("Select: ").strip())
            if 1 <= n <= len(opts):
                return opts[n-1]['label']
            return input("Custom: ").strip()
```

### 5.5 Replicating the clear-context-and-execute pattern in a skill prompt

A skill cannot call the actual REPL `clearContext` mechanism (that's a UI-layer operation). But it can simulate the same intent by instructing the model to use a sub-agent for the execution phase, which runs in an isolated context:

```typescript
const PLAN_THEN_EXECUTE_PROMPT = `# Plan-then-Execute Workflow

## Phase 1: Plan (current context)
1. Use Read, Grep, Glob to explore the codebase
2. Use AskUserQuestion if you need to clarify requirements or approach
3. Write your implementation plan — be specific about files, functions, and steps
4. Call EnterPlanMode before exploring, then ExitPlanMode when the plan is ready

## Phase 2: Execute (fresh sub-agent context)
After the plan is approved, spawn a sub-agent using the ${AGENT_TOOL_NAME} tool:

{
  "type": "execute_approved_plan",
  "prompt": "Execute this plan:\n\n[paste approved plan here]\n\nImplement each step. Update todos as you go.",
  "tools": ["Read", "Write", "Edit", "Bash", "TodoWrite"]
}

This gives the execution agent a clean context containing only the plan,
eliminating all the exploration noise from the planning phase.
`
```

### 5.6 Hook dispatch flow (for framework implementers)

When building a custom harness that supports hooks, this is the execution flow:

```
Tool call arrives
       │
       ▼
 Run PreToolUse hooks (sequential)
       │
   ┌───┴─────────────────────────────────────────┐
   │ For each HookMatcher in PreToolUse:         │
   │   if matcher matches tool name/input:       │
   │     For each hook in matcher.hooks:         │
   │       if hook.if condition matches:         │
   │         execute hook                        │
   │         if result.allowed == false          │
   │           → deny tool call                  │
   │           → return error to model           │
   │         if result.updatedInput:             │
   │           → replace tool input              │
   └─────────────────────────────────────────────┘
       │ (all allow)
       ▼
 Permission check (allow/ask/deny rules)
       │
       ▼
 Execute tool
       │
       ▼
 Run PostToolUse hooks (async-safe, side-effects only)
```

```python
class HookDispatcher:
    def __init__(self, config: dict):
        self.hooks_config: dict = config.get('hooks', {})

    async def run_pre_tool_use(
        self, tool_name: str, tool_input: dict, session_id: str
    ) -> tuple[bool, dict, str]:
        """Returns (allowed, updated_input, reason)."""
        matchers = self.hooks_config.get('PreToolUse', [])
        updated_input = tool_input

        for matcher_config in matchers:
            if not self._matcher_matches(matcher_config.get('matcher'), tool_name, tool_input):
                continue
            for hook in matcher_config.get('hooks', []):
                if not self._if_matches(hook.get('if'), tool_name, tool_input):
                    continue
                allowed, updated_input, reason = await self._exec_hook(
                    hook, tool_name, updated_input, session_id, 'PreToolUse'
                )
                if not allowed:
                    return False, updated_input, reason

        return True, updated_input, ''

    async def _exec_hook(self, hook, tool_name, tool_input, session_id, event):
        hook_type = hook['type']
        payload = {
            'event': event,
            'tool_name': tool_name,
            'tool_input': tool_input,
            'session_id': session_id,
        }
        if hook_type == 'command':
            return await self._exec_command_hook(hook, payload)
        elif hook_type == 'http':
            return await self._exec_http_hook(hook, payload)
        elif hook_type == 'prompt':
            return await self._exec_prompt_hook(hook, payload)
        return True, tool_input, ''

    async def _exec_command_hook(self, hook, payload):
        import asyncio, json
        proc = await asyncio.create_subprocess_shell(
            hook['command'],
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )
        timeout = hook.get('timeout', 10)
        try:
            stdout, _ = await asyncio.wait_for(
                proc.communicate(json.dumps(payload).encode()),
                timeout=timeout
            )
        except asyncio.TimeoutError:
            proc.kill()
            return True, payload.get('tool_input', {}), ''

        if proc.returncode == 2:  # deny
            try:
                result = json.loads(stdout)
                return False, payload.get('tool_input', {}), result.get('reason', 'Hook denied')
            except Exception:
                return False, payload.get('tool_input', {}), 'Hook denied'

        if stdout:
            try:
                result = json.loads(stdout)
                updated = result.get('updatedInput', payload.get('tool_input', {}))
                allowed = result.get('allowed', True)
                reason = result.get('reason', '')
                return allowed, updated, reason
            except Exception:
                pass

        return True, payload.get('tool_input', {}), ''

    def _matcher_matches(self, matcher: str | None, tool_name: str, tool_input: dict) -> bool:
        if not matcher:
            return True  # no matcher = match all
        # Simple name-only match; extend with glob for "Bash(git *)" syntax
        base = matcher.split('(')[0]
        return base == tool_name

    def _if_matches(self, condition: str | None, tool_name: str, tool_input: dict) -> bool:
        if not condition:
            return True
        # Parse "ToolName(input_pattern)" and match against tool_input
        return self._matcher_matches(condition, tool_name, tool_input)
```

---

## 6. Quick Reference

### 6.1 When to use which user-interaction mechanism

| Situation | Use |
|-----------|-----|
| Need 1–4 discrete choices before acting | `AskUserQuestion` |
| Task is complex; want approval before writing any code | `EnterPlanMode` → `ExitPlanMode` |
| Want fresh context for execution after planning | `ExitPlanMode` (triggers `clearContext`) |
| Task completed; user may be away | `PushNotification` |
| Running a long process; want live output | `Monitor` |
| Tracking implementation steps | `TaskCreate` / `TaskUpdate` |
| Need to block on a yes/no in plan mode | `AskUserQuestion` then `ExitPlanMode` |

### 6.2 Hook selection guide

| Requirement | Hook type |
|-------------|-----------|
| Run a shell script | `command` |
| Send to webhook/Slack/PagerDuty | `http` |
| AI-driven decision (is this safe?) | `prompt` |
| Complex validation needing tool access | `agent` |
| Programmatic in-process logic (SDK only) | `function` |
| Non-blocking audit logging | `command` + `async: true` |
| One-time setup action | any + `once: true` |
| Background check that can re-block | `command` + `asyncRewake: true` |

### 6.3 Skill context modes

| Mode | Execution | Use when |
|------|-----------|----------|
| `inline` (default) | Prompt expands in current conversation | Skill builds on existing context |
| `fork` | Runs as isolated sub-agent | Skill needs clean context; long-running; parallel |

---

*[← Advanced Patterns](01-advanced-patterns.md) | [Next: Planning & Hooks (Original)](01-planning-hooks.md)*
