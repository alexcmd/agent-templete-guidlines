# Agent Blocking & Waiting Mechanisms — Complete Taxonomy

> Every distinct way the Claude Code agent loop pauses execution and waits for
> an external signal, process, user input, or network event — with analysis of
> which mechanisms are accessible to external plugins, skills, hooks, and agents.
>
> Sources: `utils/editor.ts`, `utils/promptEditor.ts`, `utils/hooks/AsyncHookRegistry.ts`,
> `tools/AskUserQuestionTool/`, `tools/ExitPlanModeTool/`, `hooks/useInboxPoller.ts`,
> `utils/hooks/fileChangedWatcher.ts`, `bridge/replBridge.ts`

---

## Overview

Blocking in an agent loop is not merely a UX detail — it determines correctness.
An agent that resumes before the user finishes editing will read stale content.
An agent that resumes before a hook approves a command will bypass a security
gate. Understanding every blocking path is essential for building reliable
extensions.

The mechanisms fall into eight categories:

```
Category 1 — Process-level sync block       (execSync: halts Node.js event loop)
Category 2 — Tool permission / approval gate (shouldDefer + 'ask': halts REPL)
Category 3 — Hook execution blocking         (sync hook subprocess waits)
  ↳ asyncRewake variant: background hook, exit 2 → system-reminder notification wakes model
Category 4 — Polling loops                   (poll until condition, with sleep)
Category 5 — Filesystem-driven wakeup        (chokidar / fs.watch + hook fire)
Category 6 — Network waiting                 (SSE, WebSocket, HTTP await)
Category 7 — Promise-based coordination      (stored resolve, AbortController)
Category 8 — Keyboard-driven wakeup          (key handler → editor or dialog)
Category 9 — Monitor tool                    (each stdout line wakes model; no stall watchdog)
```

External code accessibility matrix (columns = who can use it):

| Category | Skills | Shell Hooks | Function Hooks | Plugin | Sub-agent |
|----------|--------|-------------|----------------|--------|-----------|
| 1 Process sync | via Bash tool | ✓ direct | ✗ | ✗ | via Bash tool |
| 2 Tool approval | call the tool | trigger via PreToolUse | trigger via PreToolUse | register tool | call the tool |
| 3 Hook blocking | via skill hooks field | ✓ direct | ✓ callback | register hook | ✗ |
| 3 asyncRewake | ✗ | ✓ asyncRewake:true | ✗ | register hook | ✗ |
| 4 Polling | ✗ internal | async flag | ✗ | ✗ | ✗ |
| 5 Filesystem | configure FileChanged | ✓ via exit watchPaths | ✓ | ✓ configure | ✗ |
| 6 Network | RemoteTrigger tool | ✗ | ✗ | ✗ | RemoteTrigger tool |
| 7 Promise coord | ✗ | ✗ | ✗ | ✗ | ✗ |
| 8 Keyboard | configure keybindings | ✗ | ✗ | configure keybindings | ✗ |
| 9 Monitor | ✗ | ✗ | ✗ | ✗ | via Monitor tool call |

---

## Category 1 — Process-Level Sync Block

### Mechanism

`execSync` / `spawnSync` from Node.js `child_process`. These calls are
synchronous — the Node.js event loop is fully paused at the call site until
the child process exits. No timers fire, no other callbacks run.

**Key call sites:**

| File | Context |
|------|---------|
| `utils/promptEditor.ts:69` | `editFileInEditor()` — editor with `--wait` |
| `utils/editor.ts:140,150` | `openFileInExternalEditor()` — terminal editor alt-screen |
| `utils/terminalPanel.ts:64,88,116` | tmux pane setup / cleanup |
| `utils/worktree.ts:1193,1332+` | git worktree tmux operations |
| `utils/ide.ts:969,1002` | IDE process-tree walk (`ps -o ppid= -p {pid}`) |
| `utils/auth.ts:1068,1798` | OAuth helper subprocess |
| `utils/secureStorage/macOsKeychainStorage.ts:39,168` | macOS keychain queries |
| `commands/exit/exit.tsx:20` | tmux detach on `/exit` |

**How it resumes:** Child process exits (or the `-w`/`--wait` flag causes the
GUI editor to exit its CLI wrapper when the user closes the relevant file tab).

**Ink rendering during the block:**

GUI editors (code, cursor, subl) — editor opens in a separate window:
```
inkInstance.pause()          → stop React render loop
inkInstance.suspendStdin()   → release stdin to OS
execSync(code -w file.md)    → blocks here
inkInstance.resumeStdin()    → re-capture stdin
inkInstance.resume()         → full re-render
```

Terminal editors (vim, nano) — editor takes over the terminal entirely:
```
inkInstance.enterAlternateScreen()  → enter alt buffer, pause Ink
spawnSync(vim file.md)              → blocks here
inkInstance.exitAlternateScreen()   → restore main buffer, re-render
```

### Who can use it

**Shell hooks** — any shell hook command IS a subprocess that uses this
implicitly. When you write `"command": "code --wait $FILE"` in a hook, your
hook shell script blocks at that call, and the hook runner waits for your
script to exit.

**Skills via Bash tool** — instruct Claude to call:
```
Bash("code --wait /path/to/plan.md")
```
The Bash tool spawns a process and `await`s its Promise-based result. This is
not `execSync` (it's async from Node's perspective), but the agent loop cannot
proceed past this tool call until the process exits — functionally identical
blocking from the agent's point of view.

**Skills via `!`-block in prompt** — skill prompt text can embed shell
commands using the `!` prefix syntax, executed by `promptShellExecution.ts`
before the model sees the prompt:
```
```!
code --wait "$PLAN_FILE"
```
```

**Not accessible to:** function hooks, plugins, sub-agents directly (they must
go through tools or shell commands).

---

## Category 2 — Tool Permission / Approval Gate

### Mechanism

The deepest blocking primitive for user interaction. Two tool properties
combine to create a blocking approval dialog:

- `shouldDefer: true` — marks the tool as deferred; the REPL renders a
  permission/approval component and halts the agent loop
- `checkPermissions()` returning `{ behavior: 'ask' }` — triggers the user
  dialog; `behavior: 'allow'` skips it entirely

**Tools with `shouldDefer: true` (25 total, key ones):**

| Tool | Permission behavior | What user sees |
|------|-------------------|----------------|
| `ExitPlanMode` | `'ask'` for non-teammates | Plan approval dialog (full plan content) |
| `AskUserQuestion` | `'ask'` always | Multi-question select dialog (2–4 options each) |
| `EnterPlanMode` | n/a (no approval needed) | Enters plan mode silently |
| `EnterWorktree` | permission check | Worktree creation confirm |
| `ExitWorktree` | permission check | Exit worktree confirm |
| `TaskCreate/Update/Stop` | permission check | Task mutation confirm |
| `NotebookEdit` | permission check | Jupyter cell edit confirm |
| `ConfigTool` | permission check | Settings change confirm |
| `CronCreate/Delete` | permission check | Schedule mutation confirm |
| `TodoWrite` | permission check | Todo update confirm |
| `TeamCreate/Delete` | permission check | Swarm team mutation confirm |
| `WebFetch/WebSearch` | permission check | Network access confirm |
| `SendMessage` | permission check | Message send confirm |
| `RemoteTrigger` | permission check | Remote API call confirm |

All Bash/Edit/Write/Read tool calls also go through the `behavior: 'ask'`
path when the command or path hasn't been pre-approved.

**The gate in `query.ts`:** When `shouldDefer: true`, the tool call is handed
off to `queryHelpers.ts` which renders the permission component via React and
suspends the agent loop. The loop resumes only when the component calls
`onDone()` with the user's decision.

**`requiresUserInteraction() → true`** (subset of above): Hard-guards that
the agent is running interactively. If this returns `true` and the session is
non-TTY (e.g., `--channels` mode, piped stdin), the tool is blocked entirely
without showing any UI. Tools: `AskUserQuestion`, `ExitPlanMode` (non-teammate).

### Who can use it

**Any tool** — a custom tool (plugin or SDK tool) can set `shouldDefer: true`
and return `{ behavior: 'ask' }` from `checkPermissions()` to trigger the
approval gate. The gate is a first-class extension point.

**Skills** — can instruct Claude to call `AskUserQuestion` which creates a
blocking dialog identical to any other permission prompt. This is the primary
mechanism for skills to pause and get structured user input.

**Shell hooks on `PermissionRequest` event** — hooks that fire on
`PermissionRequest` run BEFORE the dialog is shown. A `PreToolUse` blocking
hook runs before permission check even starts.

**Sub-agents** — can call any `shouldDefer: true` tool and the tool call
propagates to the parent REPL for user interaction (sub-agents cannot show UI
themselves, but the tool call bubbles up).

**Full `AskUserQuestion` schema:**
```typescript
{
  questions: [{
    question: string,        // "Which approach should we use?"
    header: string,          // short chip label, ≤ N chars
    options: [{              // 2–4 options
      label: string,
      description: string,
      preview?: string       // optional rich preview (code, mockup, etc.)
    }],
    multiSelect?: boolean    // default false
  }]  // 1–4 questions per call
}
```

---

## Category 3 — Hook Execution Blocking

### Mechanism

Hooks are shell commands (or HTTP/LLM/function callbacks) that fire at lifecycle
events. When a hook is **blocking** (`"blocking": true` or the event type
is `PreToolUse`), the agent loop pauses while the hook runs.

**Sync (blocking) hooks:**

The hook subprocess is spawned and the main loop `await`s its result Promise
(`ShellCommand.result`). Nothing proceeds until the hook exits.

```
PreToolUse hook fires
  → hook subprocess spawned
  → agent loop suspended
  → hook reads stdin (JSON payload), runs, writes stdout, exits
  → stdout parsed as JSON: { allowed, updatedInput, reason, ... }
  → if allowed: tool call proceeds with (possibly modified) input
  → if !allowed + blocking: tool call denied, reason injected as error
```

Exit code semantics:
- `0` — allow (default)
- `2` — deny (requires `"blocking": true`)
- other non-zero — warning only, tool allowed

**Async hooks — two distinct modes:**

**Mode A — Self-declaring async** (hook outputs `{"async": true}` to stdout first):

The hook starts synchronously, writes the JSON header to declare it needs more time,
then continues running. The REPL registers it in `AsyncHookRegistry` and polls each
main-loop tick:

```
Hook subprocess starts → prints {"async": true} → registered in pendingHooks
Agent loop continues doing other work
  ...every tick...
checkForAsyncHookResponses() polls:
  - hook.shellCommand.status === 'completed'? → parse final stdout JSON
  - timeout exceeded? → kill hook, inject timeout error
When final response arrives:
  if exitCode=0: mark success, inject stdout as system message
  if exitCode≠0: inject stderr as error system message
```

**Mode B — `asyncRewake: true` in settings.json (bypass path)**:

asyncRewake hooks are a completely distinct execution path — they **bypass
`AsyncHookRegistry` entirely**. The hook does NOT write `{"async": true}` to
stdout. Instead, stdin is injected with the JSON payload before the process
is backgrounded (`utils/hooks.ts:1006`), and the hook's result is handled
by a direct `.then()` callback:

```
Hook subprocess starts → JSON payload written to stdin → process backgrounded
Agent loop continues (no registry polling)
  ...hook runs independently...
On hook exit (utils/hooks.ts:218-243):
  setImmediate() — flush pending stdio events into in-memory TaskOutput
  read stdout + stderr
  emitHookResponse(exitCode, stdout, stderr)
  if exitCode === 2:
    enqueuePendingNotification({
      value: wrapInSystemReminder(
        `Stop hook blocking error from command "${hookName}": ${stderr || stdout}`
      ),
      mode: 'task-notification'
    })
    → model wakes via useQueueProcessor (idle)
      OR queued_command attachments (busy mid-query)
  if exitCode === 0: success, no model wakeup
```

**Critical distinction:** exit code 2 on an asyncRewake hook does NOT re-enter
a permission dialog. It enqueues a **system reminder** that wakes the model. The
model receives the message content (stderr/stdout) and can react (retry, report
to user, etc.) — but no dialog is shown and no permission gate is re-evaluated.

**Implementation constraints** (`utils/hooks.ts:205-246`):
- Does NOT call `shellCommand.background()` — avoids `spillToDisk()` which
  would break in-memory stdout/stderr capture
- StreamWrappers stay attached and pipe data into in-memory TaskOutput buffers
- `setImmediate()` yields to I/O to flush pending stdio events before reading
- Survives new user prompts (abort reason `'interrupt'` is a no-op for the hook)
- Hard Escape cancel WILL kill the hook — desired behavior for user control

**Available hook events:**

```
PreToolUse      — before permission check; can deny/modify
PostToolUse     — after tool succeeds; side-effects only
PostToolUseFailure — after tool errors
UserPromptSubmit — when user submits a message; can augment context
SessionStart    — once at session init
SessionEnd/Stop — session teardown
StopFailure     — session error
PreCompact      — before context compaction
PostCompact     — after compaction
SubagentStart   — before sub-agent spawn
SubagentStop    — sub-agent done/error
PermissionRequest — before dialog shown (can pre-approve)
PermissionDenied — after denial
TaskCreated     — todo entry created
TaskCompleted   — todo entry completed
Notification    — agent emits a user notification
FileChanged     — file in project changes (requires file watcher)
CwdChanged      — working directory changes
WorktreeCreate/Remove — git worktree lifecycle
```

**Function hooks** (`utils/hooks/sessionHooks.ts`) — in-process callbacks
registered via `addFunctionHook()`. Accept async callbacks returning
`boolean | Promise<boolean>`. Used internally (e.g., plan verification hook).
5s default timeout.

### Who can use it

**Shell hooks** — the primary consumer. Configure in `settings.json`:
```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "ExitPlanMode",
      "hooks": [{"type": "command", "command": "code --wait $PLAN_FILE", "blocking": true}]
    }],
    "FileChanged": [{
      "matcher": ".env|.envrc",
      "hooks": [{"type": "command", "command": "~/.agent/hooks/reload_env.sh"}]
    }]
  }
}
```

**Skills** — can set a `hooks` field in `BundledSkillDefinition` that applies
for the skill's duration:
```typescript
const skill: BundledSkillDefinition = {
  name: 'my-skill',
  hooks: {
    PreToolUse: [{ matcher: 'ExitPlanMode', hooks: [...] }]
  }
}
```

**Plugins** — same `hooks` field available in plugin definitions.

**HTTP hooks** — any URL that accepts POST with JSON payload and returns JSON:
```json
{
  "type": "http",
  "url": "https://your-review-system.com/hooks/plan-approval",
  "headers": {"Authorization": "Bearer ${REVIEW_TOKEN}"},
  "blocking": true,
  "timeout": 120
}
```
The hook URL receives the same stdin JSON payload and must return the same
`{ allowed, updatedInput, reason }` response JSON.

**LLM prompt hooks** — evaluated by Claude Haiku with a 30s timeout:
```json
{
  "type": "prompt",
  "prompt": "Does this command look safe? $ARGUMENTS. Reply with JSON: {\"ok\": true} or {\"ok\": false, \"reason\": \"...\"}",
  "blocking": true
}
```

---

## Category 4 — Polling Loops

### Mechanism

The agent does not truly block — it loops with sleep intervals, checking a
condition each iteration. The distinction from Categories 1–3: other work CAN
proceed between poll ticks.

**Async hook polling** (`AsyncHookRegistry.ts:113`):

Called from the REPL main loop on each turn start:
```typescript
const asyncResponses = await checkForAsyncHookResponses()
// Each pending hook: if status === 'completed', parse response; else skip
// Hooks that haven't completed: stay in pendingHooks, checked next turn
```
Default timeout: 15s (`asyncResponse.asyncTimeout || 15000`).

**Teammate inbox polling** (`hooks/useInboxPoller.ts:107`):

React hook using `useInterval` — fires every 1000ms while the session is alive:
```typescript
const INBOX_POLL_INTERVAL_MS = 1000

useInterval(async () => {
  const msgs = await readUnreadMessages(agentName)
  for (const msg of msgs) {
    if (isPlanApprovalRequest(msg)) { /* show approval dialog */ }
    if (isPermissionRequest(msg)) { /* route to swarm permission bridge */ }
    // ... other message types
  }
}, INBOX_POLL_INTERVAL_MS)
```
This is how sub-agents in swarm mode receive approval responses from the team
lead. The sub-agent blocks (via `setAwaitingPlanApproval`) and the team lead
writes the approval to the mailbox file.

**Bridge poll loop** (`bridge/replBridge.ts:1938`):

For remote (CCR) sessions:
```typescript
while (!signal.aborted) {
  const work = await api.pollForWork({ session_id, capacity })
  if (!work) {
    await sleep(POLL_INTERVAL_MS, signal)  // ~1s between polls
    continue
  }
  // Process work item...
}
```
With capacity backoff: if at capacity, sleeps up to 30s before polling again.

**Sleep tool** — explicitly instructs the agent to sleep:
```
Use: Sleep(seconds=30)
Agent calls Sleep tool → await sleep(ms, signal)
Resumes: timer fires OR user interrupts (Ctrl+C)
```
Signal-aware: `sleep(ms, signal)` resolves early if `signal.abort()` is called.

### Who can use it

**Not directly accessible** — polling is an internal mechanism. External code
cannot inject into a polling loop.

**Indirectly via async hooks** — a hook that returns `{"async": true}` gets
polled by the async hook registry. Your hook subprocess can stay alive for
minutes; the registry polls it every REPL turn.

**Indirectly via mailbox** — a sub-agent in swarm mode can write to the
team-lead mailbox and wait for a response. The team lead's poller picks it up
and routes the response back.

---

## Category 5 — Filesystem-Driven Wakeup

### Mechanism

File watchers fire callbacks when monitored files change. Unlike polling, these
are event-driven (via chokidar/`fs.watch` kernel notifications — inotify on
Linux).

**Watchers active during a session:**

| Watcher | File(s) watched | What happens on change |
|---------|----------------|----------------------|
| `fileChangedWatcher.ts` | paths from hook `matcher` field | Executes `FileChanged` hooks |
| `fileChangedWatcher.ts` | dynamic paths from hook `watchPaths` output | Re-evaluates and re-registers |
| `skillChangeDetector.ts` | skills directory | Reloads skill definitions |
| `keybindings/loadUserBindings.ts` | `~/.claude/keybindings.json` | Hot-reloads key bindings |
| `settings/changeDetector.ts` | `settings.json` | Reloads config in-session |
| `hooks/useTaskListWatcher.ts` | task list file | Re-renders task list UI |
| `hooks/useTasksV2.ts` | task file | Re-fetches tasks, 5s fallback poll |
| `services/teamMemorySync/watcher.ts` | team memory directory | Syncs shared memory across agents |
| `utils/cronScheduler.ts` | cron schedule file | Reloads/reschedules cron jobs |

**`FileChanged` hook configuration:**

```json
{
  "hooks": {
    "FileChanged": [{
      "matcher": ".env|.envrc|.tool-versions",
      "hooks": [{
        "type": "command",
        "command": "~/.agent/hooks/env_changed.sh"
      }]
    }]
  }
}
```

Hook stdin payload:
```json
{
  "event": "FileChanged",
  "file": "/path/to/.env",
  "changeType": "change"
}
```

**Dynamic watch path expansion** — a `FileChanged` hook's stdout can include
new paths to watch:
```json
{
  "watchPaths": ["/path/to/additional/file.conf"]
}
```
The watcher restarts and includes these new paths on the next check.

**`CwdChanged` hooks** — fire when the agent's working directory changes.
Used for reloading env files, updating project-specific config:
```json
{
  "hooks": {
    "CwdChanged": [{
      "hooks": [{
        "type": "command",
        "command": "~/.agent/hooks/setup_project_env.sh"
      }]
    }]
  }
}
```

### Who can use it

**Shell hooks** — configure `FileChanged` / `CwdChanged` in `settings.json`.
This is the correct mechanism for reacting to external tool outputs (build
system writes a status file, linter updates a report file, etc.).

**Plugins** — can register file watchers as part of their initialization.

**Skills** — can include `watchPaths` in their hook output to dynamically
expand the watch list.

**Use cases:**
- Build system writes `build-status.json` → `FileChanged` hook fires → inject
  status into agent context
- `.env` changes (secret rotation, new env var) → `FileChanged` hook reloads
  env → next Bash tool call picks up new values
- CI writes test results to `test-results.json` → hook fires → agent reads
  and continues
- Project switches via `cd` → `CwdChanged` hook loads project-specific setup

---

## Category 6 — Network Waiting

### Mechanism

`await axios.request(...)` or SSE/WebSocket read loops. Truly async — the
event loop is not blocked, but the agent turn cannot complete until the
network response arrives.

**Remote Trigger Tool** (`tools/RemoteTriggerTool/RemoteTriggerTool.ts:135`):
```typescript
const res = await axios.request({
  method, url, headers, data,
  timeout: 20_000,                        // 20s hard timeout
  signal: context.abortController.signal, // abortable by user Ctrl+C
  validateStatus: () => true,             // don't throw on 4xx/5xx
})
```

Actions: `list` | `get` | `create` | `update` | `run`. The `run` action
executes a pre-configured remote trigger and waits for acknowledgement.

**SSE transport** (`cli/transports/SSETransport.ts`) — for remote (CCR)
sessions, the agent receives turns as server-sent events:
```
Reconnect config: base=1s, max=30s, timeout=45s
Frame types: 'message', 'heartbeat' (every 15s), 'close'
Sequence tracking: drops duplicate frames, requests re-sync on gap
```

**HTTP hooks** — when `type: 'http'` hooks fire, an outbound POST is made and
the hook runner awaits the response. The agent loop is paused while this runs
if the hook is `blocking: true`.

### Who can use it

**RemoteTrigger tool** — callable by Claude directly or via skill prompts.
Requires `tengu_surreal_dali` feature flag and OAuth auth.

**HTTP hooks** — the cleanest way for external systems (CI/CD, Slack bots,
approval systems) to gate agent actions. The agent calls a tool, the hook
fires, your server receives the POST, responds with `{ allowed: true/false }`.

**Patterns:**

```
Agent action → hook fires → POST to your server → server responds
                                    ↑
                         your server can:
                         - check Jira ticket status
                         - validate with compliance API
                         - prompt human via Slack and wait for reaction
                         - check test results before allowing deploy
```

---

## Category 7 — Promise-Based Coordination

### Mechanism

Stored `resolve()` callbacks that are called from a different code path,
unblocking an `await new Promise(...)` elsewhere.

**Key usage sites:**

| File | Pattern | What triggers resolve |
|------|---------|----------------------|
| `screens/REPL.tsx:2223` | `resolveShouldAllowHost` | User confirms unknown SSH host |
| `screens/REPL.tsx:2253` | `resolvePromptResponse` | User submits text in prompt input |
| `cli/print.ts:3335` | auth URL promise | Auth flow completes |
| `interactiveHelpers.tsx:40` | dialog response | User confirms dialog |
| `utils/mailbox.ts:63` | teammate message | Message written to mailbox |

**AbortController as synchronization** (`utils/abortController.ts`):

The `AbortController.signal.aborted` flag is checked at every async boundary:
```typescript
// query.ts: agent loop checks abort before each LLM turn
if (context.abortController.signal.aborted) {
  return { reason: 'aborted' }
}

// bridge/replBridge.ts: poll loop exits when aborted
while (!signal.aborted) {
  await pollForWork()
}

// ShellCommand.ts: process killed when signal fires
signal.addEventListener('abort', () => child.kill())
```

The signal fires on:
- User presses Escape (interrupt current tool)
- User sends new message while tool is running (for `interruptBehavior: 'cancel'` tools)
- Session shutdown
- Timeout

### Who can use it

**Mostly internal** — Promise coordination is not a public extension API.
External code cannot inject `resolve()` callbacks.

**Indirectly via tools** — any tool that involves user input (permission
dialogs, `AskUserQuestion`) uses this pattern internally. External code
triggers it by calling the tool.

**AbortController in function hooks** — function hooks receive the abort
signal as a parameter and should respect it:
```typescript
addFunctionHook('PreToolUse', async (messages, signal) => {
  if (signal?.aborted) return true  // cancelled, allow
  const result = await doValidation(messages, { signal })
  return result.valid
})
```

---

## Category 8 — Keyboard-Driven Wakeup

### Mechanism

Keyboard shortcuts that trigger blocking actions directly from the input
handler, bypassing the agent loop entirely.

**Default bindings** (`keybindings/defaultBindings.ts`):

| Binding | Action | Blocks how |
|---------|--------|-----------|
| `ctrl+g` | `chat:externalEditor` | calls `editPromptInEditor()` → `execSync` |
| `ctrl+c` | interrupt | fires `abortController.abort()` |
| `ctrl+z` | suspend process | `process.kill(process.pid, 'SIGTSTP')` |
| `ctrl+r` | history search | inline TUI search |
| tmux swarm bindings | toggle swarm panel | `spawnSync(tmux)` |

**`ctrl+g` flow** (`components/PromptInput/PromptInput.tsx:1320`):
```
User presses ctrl+g
→ handler calls editPromptInEditor(currentPrompt)
→ → editFileInEditor(tempFile)   [writes current prompt to temp file]
→ → → execSync(editor tempFile)  [blocks]
→ → → read edited content
→ return to PromptInput with edited prompt text
→ user can submit or keep editing
```

The agent is NOT running during this — the user is editing a prompt before
submitting it. But the key insight: the same `editFileInEditor` mechanism used
for plan review is reused here.

**Custom keybindings** (`~/.claude/keybindings.json`):
```json
[
  {
    "key": "ctrl+e",
    "action": "chat:externalEditor"
  }
]
```

Chord bindings are supported:
```json
[
  {
    "key": "ctrl+x ctrl+e",
    "action": "chat:externalEditor"
  }
]
```

Hot-reload: the keybindings file is watched by chokidar and reloads without
restarting Claude Code.

### Who can use it

**Users** — configure `~/.claude/keybindings.json`.

**Plugins** — can register custom actions and bind keys to them.

**Not skills/hooks** — keyboard bindings are outside the tool/hook system.

---

## Category 9 — Monitor Tool (Stream-per-Line Wakeup)

The Monitor tool (`feature('MONITOR_TOOL')`) is a specialized background process
wrapper where **each stdout line from the monitored process becomes a separate
notification** that wakes the model. Unlike `run_in_background` (single wakeup
on completion), Monitor gives the model a continuous stream of events.

### Mechanism

Monitor creates a `LocalShellTask` with `kind: 'monitor'` and streams output
lines as `task-notification` events (via `enqueueStreamEvent`) with no `<status>`
tag — these are progress pings, not terminal events. The model is woken on each
line and can react in real-time.

```
Monitor("docker events --filter type=container", description="Watch containers")
  │
  ├── Spawns process as LocalShellTask with kind='monitor'
  ├── Each stdout line → enqueueStreamEvent → no <status> tag
  │       → model wakes (progress ping, NOT task completion)
  │       → model reads the line and can act
  │
  └── On process exit:
        completed → "Monitor '<description>' stream ended"
        failed    → "Monitor '<description>' script failed (exit N)"
        killed    → "Monitor '<description>' stopped"
```

**Key differences from `Bash(run_in_background=True)`:**

| | `run_in_background` | `Monitor` |
|---|---|---|
| Wakeup events | One (on completion) | One per stdout line |
| Model sees | Final status + output file path | Each line as it arrives |
| Use case | Wait for process to finish | React to streaming events |
| Stall watchdog | Yes (detects prompt hang) | No (streaming-only, no stall) |
| Completion message | "background command completed" | "stream ended / script failed / stopped" |
| Collapses with others | Yes (N background commands) | No (distinct summary prefix) |
| Leading sleep ≥ 2s | Blocked by BashTool | Irrelevant (not a shell command) |

### Until-loop pattern

Monitor is the correct tool for condition polling:

```
Monitor("until condition; do sleep 2; done", description="Wait for service")
```

The until-loop exits (stream ends) when the condition is met. The model is
notified "stream ended" and can proceed. Compare to the wrong pattern:

```bash
# Wrong: sleep loop in Bash blocks and wastes a tool call per iteration
while ! curl -sf http://service/health; do sleep 2; done  # blocks Claude Code

# Right: use Monitor for the polling, Bash for the action after
Monitor("until curl -sf http://service/health; do sleep 2; done")
# model wakes when stream ends → then runs next step
```

### Use cases

```
Stream build output line by line         Monitor("npm run build 2>&1")
Watch docker/k8s events in real-time    Monitor("kubectl get events -w")
Poll until condition met                 Monitor("until check; do sleep 2; done")
Tail a log file for errors               Monitor("tail -f /var/log/app.log | grep ERROR")
Watch filesystem for file appearance     Monitor("inotifywait -m /tmp/output.json")
```

### Who can use it

**Model (Claude)** — calls the Monitor tool directly when `MONITOR_TOOL` feature
is enabled. The system prompt explicitly says: _"Use the Monitor tool to stream
events from a background process (each stdout line is a notification). For
one-shot 'wait until done,' use Bash with run_in_background instead."_

**Not available to:** hooks, skills, or plugins directly. Hooks can spawn their
own long-running processes, but those don't feed back into the Monitor stream.

### Implementation notes

- `startStallWatchdog()` skips monitor tasks: `if (kind === 'monitor') return () => {}` (`LocalShellTask.tsx:47`)
- `collapseBackgroundBashNotifications()` never collapses Monitor events (no `<status>` tag, distinct summary prefix)
- Status pill counts monitors separately from bash tasks (`pillLabel.ts:19`)
- UI shows `description` instead of raw command (`BackgroundTasksDialog.tsx:498`)

---

## Comparative Analysis: Which Mechanism to Use When

### Choosing the right block type for external review/approval

```
Scenario                              Recommended mechanism
─────────────────────────────────────────────────────────────────────────
Plan review in external editor        PreToolUse hook (blocking) on ExitPlanMode
                                      → "command": "code --wait $PLAN_FILE"

Ask user to choose between options    AskUserQuestion tool (via skill prompt)
                                      → structured 2–4 option dialog

Wait for CI test results              FileChanged hook on test-results.json
                                      OR Monitor("until test-results.json exists; do sleep 2; done")
                                      OR asyncRewake hook on Bash(run tests) → model woken with result

External approval system (Slack, etc) HTTP hook on PreToolUse
                                      → your server waits for Slack reaction, then responds

Wait for human to finish a task       AskUserQuestion with "Done?" option
                                      OR PostToolUse hook that writes to mailbox

Guard a dangerous command             PreToolUse hook (blocking) on Bash
                                      → exit 2 to deny, exit 0 to allow

React to env/config change            FileChanged hook on .env / config files

Remote agent scheduling               RemoteTrigger tool (list/create/run)

React to streaming events in real-time Monitor tool (each stdout line wakes model)
                                      → docker events, kubectl -w, log tail, etc.

Poll until condition                  Monitor("until <check>; do sleep 2; done")
                                      → model woken when stream ends (condition met)

Sub-agent waiting for team lead       setAwaitingPlanApproval + mailbox polling
                                      (swarm mode only)
```

### Blocking hierarchy (strongest to weakest)

```
1. execSync (process-level)
   └── halts Node.js event loop entirely
   └── no callbacks, no timers, no GC
   └── strongest: OS-enforced

2. Permission dialog (shouldDefer + 'ask')
   └── halts agent loop in REPL
   └── React still runs (renders the dialog)
   └── user must take action to unblock

3. Sync hook (blocking: true)
   └── halts permission check phase
   └── hook subprocess must exit before tool call proceeds

4. Async hook (async: true)
   └── agent continues other work
   └── hook polled by AsyncHookRegistry each main-loop tick
   └── weakest: non-deterministic timing

4b. asyncRewake hook
   └── agent continues other work (bypasses registry)
   └── exit code 2 → system-reminder notification wakes model
   └── does NOT re-enter permission dialog — model receives message and decides
```

### Timeout reference

| Mechanism | Default timeout | Configurable |
|-----------|----------------|-------------|
| `execSync` editor | none (process exit) | via editor's own timeout |
| Permission dialog | none (user must act) | N/A |
| Sync hook | 10s (command), 60s (agent) | `"timeout": N` in hook config |
| Async hook | 15s | `"asyncTimeout": N` in hook JSON response |
| LLM prompt hook | 30s | fixed |
| HTTP hook | varies | depends on server |
| Function hook | 5s | configurable in `addFunctionHook()` |
| RemoteTrigger HTTP | 20s | fixed |
| Sleep tool | user-specified | N/A |

---

## Complete Hook Event → Blocking Matrix

| Event | Blocking? | Can deny? | Can modify input? | Accessible to |
|-------|----------|-----------|------------------|--------------|
| `PreToolUse` | ✓ | ✓ | ✓ | shell, HTTP, prompt, function |
| `PostToolUse` | ✗ (side-effects) | ✗ | ✗ | shell, HTTP, prompt, function |
| `PostToolUseFailure` | ✗ | ✗ | ✗ | shell, HTTP, prompt, function |
| `UserPromptSubmit` | ✓ | ✓ | ✓ (inject context) | shell, HTTP, prompt, function |
| `SessionStart` | ✓ async | ✗ (async only) | ✗ | shell, HTTP |
| `SessionEnd/Stop` | ✗ | ✗ | ✗ | shell, HTTP, function |
| `PermissionRequest` | ✓ | ✓ (pre-approve) | ✓ (modify permissions) | shell, HTTP |
| `FileChanged` | ✗ | ✗ | ✗ | shell, HTTP |
| `CwdChanged` | ✗ | ✗ | ✗ | shell, HTTP |
| `SubagentStart` | ✓ | ✓ | ✓ (modify system prompt) | shell, HTTP |
| `SubagentStop` | ✗ | ✗ | ✗ | shell, HTTP |
| `PreCompact` | ✓ | ✗ | ✓ (inject hints) | shell, HTTP, function |
| `WorktreeCreate` | ✓ | ✓ | ✗ | shell, HTTP |
| `Notification` | ✗ | ✗ | ✗ | shell, HTTP |

---

## Code Template: External Review Gate (all patterns)

### Pattern A — Blocking shell hook (simplest)
```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "ExitPlanMode",
      "hooks": [{
        "type": "command",
        "command": "~/.agent/hooks/review_gate.sh",
        "blocking": true,
        "timeout": 600
      }]
    }]
  }
}
```
```bash
#!/usr/bin/env bash
INPUT=$(cat)
PLAN=$(echo "$INPUT" | jq -r '.inputJson.planFilePath // empty')
[ -z "$PLAN" ] && exit 0
code --wait "$PLAN" && exit 0 || echo '{"allowed":false,"reason":"Review aborted"}' && exit 2
```

### Pattern B — HTTP approval gateway
```json
{
  "type": "http",
  "url": "https://approval.internal/agent-gate",
  "headers": {"Authorization": "Bearer ${APPROVAL_TOKEN}"},
  "blocking": true,
  "timeout": 300
}
```
Your server receives the hook payload, can take up to 300s to respond with
`{"allowed": true/false}`. Use for Slack/email approval flows.

### Pattern C — asyncRewake: background hook that wakes the model
```json
{
  "type": "command",
  "command": "~/.agent/hooks/async_review.sh",
  "asyncRewake": true,
  "timeout": 600
}
```
```bash
#!/usr/bin/env bash
# asyncRewake hooks: stdin is pre-loaded with JSON before background launch.
# Do NOT output {"async": true} — asyncRewake bypasses the async registry.
INPUT=$(cat)
PLAN_FILE=$(echo "$INPUT" | jq -r '.inputJson.planFilePath // empty')
[ -z "$PLAN_FILE" ] && exit 0

code --wait "$PLAN_FILE"

# User closed VSCode. Did they approve?
if grep -q 'APPROVED' "$PLAN_FILE"; then
  exit 0                   # success — model is NOT woken up
else
  # exit 2 → enqueuePendingNotification wrapping stderr/stdout as system reminder
  # The model wakes and reads this message — no permission dialog is shown.
  echo "Plan review failed: user did not mark APPROVED in $PLAN_FILE"
  exit 2
fi
```

**Note:** exit 2 wakes the model with a `<system-reminder>` containing the hook's
stderr/stdout. This is NOT a permission re-entry — the model receives the message
and decides how to proceed (retry, ask user, abort, etc.).

### Pattern D — AskUserQuestion via skill (structured dialog)
```typescript
const reviewSkill: BundledSkillDefinition = {
  name: 'review-plan',
  allowedTools: ['AskUserQuestion', 'Bash', 'Read'],
  async getPromptForCommand() {
    return [{
      type: 'text',
      text: `
1. Use Bash: \`code --wait "$(plan-path)"\`
2. Use Read to get the (possibly edited) plan contents
3. Use AskUserQuestion:
   question: "Plan reviewed. How should we proceed?"
   header: "Plan review"
   options:
     - label: "Approve & implement"
       description: "Start coding now"
     - label: "Request changes"
       description: "I've edited the plan — regenerate approach"
     - label: "Reject"
       description: "Start over with a different plan"
4. Act on the answer.
      `
    }]
  }
}
```

### Pattern E — FileChanged trigger (CI-driven)
```json
{
  "hooks": {
    "FileChanged": [{
      "matcher": "test-results.json|build-status.json",
      "hooks": [{
        "type": "command",
        "command": "~/.agent/hooks/inject_results.sh"
      }]
    }]
  }
}
```
```bash
#!/usr/bin/env bash
INPUT=$(cat)
FILE=$(echo "$INPUT" | jq -r '.file')
CONTENT=$(cat "$FILE")
# Output content to be injected as a system message
echo "File changed: $FILE"
echo "$CONTENT"
```

---

*Related: [External Editor & Plan Review](01-external-editor-plan-review.md) · [Planning Mode & Hooks](01-planning-hooks.md) · [OS & Shell Integration](01-os-shell-integration.md)*
