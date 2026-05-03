# External Editor Integration & Plan Review Workflow

> Deep-dive: how an agent blocks execution while a user reviews a plan in an
> external editor (VSCode, Cursor, vim, etc.), and how skills / plugins can
> use the same mechanism.
>
> Source: `claude-code/src/utils/editor.ts`, `utils/promptEditor.ts`,
> `commands/plan/plan.tsx`, `tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`,
> `utils/hooks/AsyncHookRegistry.ts`

---

## Overview

Two things must happen when the agent finishes a plan and wants the user to
review it:

1. **Block** — the agent loop must stop processing new LLM turns until the
   human is done reviewing.
2. **Signal** — the agent must resume cleanly once the review is finished,
   consuming any edits the user made.

Claude Code solves this with three composable mechanisms, ranked by fidelity:

| Mechanism | Blocks how | Resumes how | Best for |
|-----------|-----------|-------------|---------|
| Permission dialog (`shouldDefer`) | REPL halts at approval UI | User clicks approve/reject | Built-in plan approval gate |
| `execSync` editor (`--wait` flag) | Process-level sync block | Editor process exits | `/plan open` + terminal editors |
| Async hook (`asyncRewake`) | Hook subprocess runs in background | Hook exits 0 → continue, 2 → re-enter permission | Custom external review tools |

Understanding each mechanism lets you mix them: e.g., auto-open VSCode when
the plan is ready **and** show the approval dialog after the editor closes.

---

## §1 — Editor Detection and Classification

`utils/editor.ts:164` — `getExternalEditor()` (memoized for the session):

```
Priority 1: $VISUAL env var (e.g., "code --wait" or "vim")
Priority 2: $EDITOR env var
Priority 3: PATH scan: code → vi → nano (first available)
Windows fallback: "start /wait notepad"
```

Editors split into two families:

```typescript
// GUI editors — open in a separate window, do not take over the terminal
const GUI_EDITORS = ['code', 'cursor', 'windsurf', 'codium', 'subl', 'atom', 'gedit', 'notepad++', 'notepad']

// Terminal editors — take over stdin/stdout, need alt-screen handoff
// (vim, nvim, nano, emacs, pico, micro, helix, hx)
```

`classifyGuiEditor(editor)` returns the matched family name, or `undefined`
for terminal editors. Classification uses `basename()` so absolute paths
(`/usr/bin/code`) still match.

**Wait-mode overrides** for GUI editors — applied automatically in
`editFileInEditor()`:

```typescript
const EDITOR_OVERRIDES: Record<string, string> = {
  code: 'code -w',      // VS Code: block until the tab is closed
  subl: 'subl --wait',  // Sublime Text: same
}
```

`-w` / `--wait` is the critical flag. Without it, `code <file>` returns
immediately and the agent never sees the edits.

---

## §2 — The Blocking Edit Call (`editFileInEditor`)

`utils/promptEditor.ts:31` — synchronous, truly blocks the Node.js event loop:

```typescript
export function editFileInEditor(filePath: string): EditorResult {
  const inkInstance = instances.get(process.stdout)
  const editor = getExternalEditor()

  const useAlternateScreen = !isGuiEditor(editor)

  if (useAlternateScreen) {
    // vim / nano — take over the whole terminal
    inkInstance.enterAlternateScreen()   // pauses Ink, enters alt buffer
  } else {
    // code / subl — open in separate window, release our terminal
    inkInstance.pause()
    inkInstance.suspendStdin()
  }

  try {
    const editorCommand = EDITOR_OVERRIDES[editor] ?? editor
    execSync(`${editorCommand} "${filePath}"`, { stdio: 'inherit' })
    //       ↑ blocks until the editor process exits

    const editedContent = fs.readFileSync(filePath, { encoding: 'utf-8' })
    return { content: editedContent }
  } catch (err) {
    return { content: null, error: `Editor exited with code ${err.status}` }
  } finally {
    if (useAlternateScreen) {
      inkInstance.exitAlternateScreen()
    } else {
      inkInstance.resumeStdin()
      inkInstance.resume()             // Ink re-renders the REPL
    }
  }
}
```

**Key invariant:** `execSync` with `stdio: 'inherit'` hands stdin/stdout
to the child process. The parent Node.js process is literally paused at this
call until the child exits — no timers fire, no other callbacks run.

For VSCode this means: `code -w plan.md` — VSCode opens the file in a new tab,
the Node.js process waits. When the user closes that tab, VSCode exits
(via the `-w` flag), the `execSync` call returns, and the agent reads back the
(possibly edited) content.

---

## §3 — Current Plan Review Workflow

### 3.1 What happens now (two separate steps)

```
Agent calls ExitPlanMode()
    │
    ▼
Permission dialog appears (shouldDefer: true, checkPermissions → 'ask')
    │
    ├── User sees plan in TUI
    ├── User types "/plan open"          ← manual step
    │       └── editFileInEditor(planPath) ← blocks until VSCode tab closes
    │
    ▼
User approves / rejects plan in TUI
    │
    ▼
Agent continues (execute phase)
```

The problem: the user must manually type `/plan open`. The plan review step
and the editor-open step are disconnected.

### 3.2 Auto-open on plan ready (implementation pattern)

To automatically open VSCode when `ExitPlanMode` fires, insert
`editFileInEditor` into the tool's `call()` before the permission flow
concludes:

```typescript
// ExitPlanModeV2Tool.ts — augmented call()
async call(input, context) {
  const filePath = getPlanFilePath(context.agentId)
  const plan = getPlan(context.agentId)

  // Auto-open plan in editor for review before the approval dialog appears.
  // execSync inside editFileInEditor blocks here until the editor exits.
  if (plan && !isTeammate() && !context.agentId) {
    const editorResult = editFileInEditor(filePath)
    // If the user edited the plan while reviewing, use the updated content.
    const reviewedPlan = editorResult.content ?? plan
    // ... continue with permission flow using reviewedPlan
  }

  // ... rest of original call()
}
```

**Sequence with auto-open:**

```
Agent calls ExitPlanMode()
    │
    ▼
ExitPlanMode.call() runs
    ├── editFileInEditor(planPath)  ← VSCode opens, agent BLOCKS
    │       user reviews / edits plan in VSCode
    │       user closes the tab
    │       editFileInEditor returns with (possibly edited) content
    │
    ▼
Permission dialog appears
    ├── Shows plan (with any user edits from VSCode already applied)
    │
    ▼
User approves or rejects
```

---

## §4 — Hook-Based External Review (no source changes required)

If you cannot modify the agent source, the same behavior is achievable with a
`PreToolUse` hook on `ExitPlanMode`.

### 4.1 Shell hook that opens VSCode and waits

`~/.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "ExitPlanMode",
        "hooks": [
          {
            "type": "command",
            "command": "~/.agent/hooks/review_plan.sh",
            "blocking": true,
            "timeout": 600
          }
        ]
      }
    ]
  }
}
```

`~/.agent/hooks/review_plan.sh`:

```bash
#!/usr/bin/env bash
# stdin: JSON with {"event":"PreToolUse","tool":"ExitPlanMode","inputJson":{...}}

INPUT=$(cat)
# The plan file path is injected by normalizeToolInput
PLAN_PATH=$(echo "$INPUT" | jq -r '.inputJson.planFilePath // empty')

if [[ -z "$PLAN_PATH" ]]; then
  # No plan file — let it through
  echo '{"allowed": true}'
  exit 0
fi

# Open the plan in VSCode and block until the user closes the tab
code --wait "$PLAN_PATH"
EDITOR_EXIT=$?

if [[ $EDITOR_EXIT -ne 0 ]]; then
  echo "{\"allowed\": false, \"reason\": \"Editor exited with code $EDITOR_EXIT\"}"
  exit 2
fi

# Allow ExitPlanMode to proceed (user has reviewed)
echo '{"allowed": true}'
exit 0
```

**What happens:**
1. Agent calls `ExitPlanMode()`.
2. The hook fires **before** the permission dialog.
3. The hook shell command runs `code --wait plan.md` — the hook process itself
   blocks while VSCode is open.
4. The agent loop is paused: it receives no new messages while the hook is
   running (because the hook is `blocking: true`).
5. User closes the VSCode tab → hook exits 0 → permission dialog appears.
6. User approves → execute phase begins.

### 4.2 Async hook with `asyncRewake` (non-blocking variant)

If you want the review to happen in the background while the agent continues
other work (e.g., writing a task list), use `asyncRewake`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "ExitPlanMode",
        "hooks": [
          {
            "type": "command",
            "command": "~/.agent/hooks/async_review.sh",
            "blocking": false,
            "asyncRewake": true,
            "timeout": 600
          }
        ]
      }
    ]
  }
}
```

`asyncRewake: true` means: run the hook in the background; if it eventually
exits with code 2, re-enter the permission flow for this tool call. This is
useful for long-running review processes (human-in-the-loop approval systems,
Slack/email notifications, etc.).

---

## §5 — Skill Implementation of External Review

Skills run as injected prompts (not compiled code), so they cannot call
`editFileInEditor()` directly. The patterns available to skills:

### 5.1 Shell command in skill prompt (promptShellExecution)

Skills support `` `!command` `` syntax in their prompt text. When the skill
prompt is executed, any `!`-prefixed code blocks run as shell commands first:

```typescript
// In a BundledSkillDefinition
const reviewSkill: BundledSkillDefinition = {
  name: 'review-plan',
  description: 'Open the current plan in VSCode for review before proceeding',
  async getPromptForCommand(args, context) {
    return [{
      type: 'text',
      text: `
Open the plan in VSCode and wait for the user to finish reviewing it.

\`\`\`!
PLAN_PATH=$(claude code plan-path 2>/dev/null)
if [ -n "$PLAN_PATH" ] && [ -f "$PLAN_PATH" ]; then
  code --wait "$PLAN_PATH"
fi
\`\`\`

After VSCode closes, report the current plan contents back to the user
and ask if they want to proceed with implementation.
      `.trim()
    }]
  }
}
```

The `!`-block runs synchronously before the LLM sees the prompt, so VSCode
opens and blocks execution before the model generates its response.

### 5.2 AskUserQuestion pattern (cleanest for skills)

The `AskUserQuestion` tool creates a blocking dialog identical to a permission
prompt. A skill can instruct Claude to open the editor **and** block:

```typescript
const reviewSkill: BundledSkillDefinition = {
  name: 'review-and-proceed',
  description: 'Open the plan for external review, then wait for user approval',
  async getPromptForCommand(args, context) {
    return [{
      type: 'text',
      text: `
You are about to present the plan for external review. Follow these steps exactly:

1. Use the Bash tool to open the plan file in VSCode:
   \`code --wait <plan-file-path>\`
   
   This command will block until the user closes the VSCode tab.

2. After the Bash tool returns, read the plan file contents back with the Read tool.

3. Call AskUserQuestion with:
   - question: "I have opened the plan in VSCode. Have you finished reviewing it? 
     Type 'yes' to proceed with implementation, or describe any changes you want."
   
4. If the user says yes (or a variant), proceed with implementation.
   If the user requests changes, update the plan file and repeat from step 1.
      `.trim()
    }]
  },
  allowedTools: ['Bash', 'Read', 'AskUserQuestion']
}
```

**Why this works:**
- `Bash("code --wait plan.md")` blocks via the Bash tool's process-wait mechanism
  (same `ShellCommand.result` Promise as any other Bash command).
- `AskUserQuestion` creates a blocking dialog that halts the agent loop until
  the user responds.
- The skill never needs to touch Ink or editor internals.

### 5.3 Hooks in skill definitions

`BundledSkillDefinition` accepts a `hooks` field. Skills can configure hooks
that fire for the duration of the skill's execution:

```typescript
const reviewSkill: BundledSkillDefinition = {
  name: 'plan-review',
  hooks: {
    // This hook fires before any ExitPlanMode call while this skill is active
    PreToolUse: [{
      matcher: 'ExitPlanMode',
      hooks: [{
        type: 'command',
        command: 'code --wait "$PLAN_FILE_PATH"',
        blocking: true,
        timeout: 600
      }]
    }]
  }
}
```

---

## §6 — Blocking Mechanism Deep Dive

### 6.1 Three levels of blocking

```
Level 1: Process-level sync (execSync)
──────────────────────────────────────
Node.js event loop: ████████████[BLOCKED: execSync]████████████
                                  │                 │
                                  └── editor opens  └── editor closes
Suitable for: terminal editors, GUI editors with --wait

Level 2: Permission dialog (shouldDefer + 'ask')
─────────────────────────────────────────────────
REPL loop:     ...message queue empty...
                       ↑
Permission UI: [Plan content shown] [Approve] [Reject]
                                    ↑ user clicks
                                    tool.call() runs, loop continues
Suitable for: built-in approval gates, custom tool approval

Level 3: Async hook (asyncRewake)
──────────────────────────────────
Hook subprocess: [running in background]────────[exits 0 or 2]
REPL loop:        continues processing          if 2: re-enters permission flow
                                                if 0: proceeds
Suitable for: long-running external reviews, notifications, CI/CD gates
```

### 6.2 Ink rendering pause during editor

When `editFileInEditor` runs, it must prevent Ink from redrawing the terminal
while the editor owns the screen:

```
GUI editors (code, subl):
  inkInstance.pause()          → stop rendering loop
  inkInstance.suspendStdin()   → release stdin to the OS
  execSync(code -w ...)        → blocks
  inkInstance.resumeStdin()    → re-capture stdin
  inkInstance.resume()         → trigger a full re-render

Terminal editors (vim, nano):
  inkInstance.enterAlternateScreen()   → enter alt buffer, pause Ink
  spawnSync(vim ...)                   → vim takes over the terminal
  inkInstance.exitAlternateScreen()    → restore main buffer, re-render
```

The difference: GUI editors open in a separate window, so the terminal stays
under Ink's control but stdin must be released. Terminal editors take over the
terminal entirely, requiring the alt-screen handoff.

### 6.3 Process-completion waiting in hooks

`utils/hooks/AsyncHookRegistry.ts` — how the hook runner waits for a
background hook to finish:

```typescript
// Simplified from AsyncHookRegistry.ts:113–268
async function checkForAsyncHookResponses() {
  const pending = getPendingAsyncHooks()

  await Promise.allSettled(pending.map(async hook => {
    if (hook.shellCommand.status !== 'completed') {
      return { type: 'skip' }   // still running — check again next tick
    }

    // Parse JSON response from hook stdout
    const stdout = hook.shellCommand.taskOutput.getOutput()
    const response = JSON.parse(stdout)

    if (response.allowed === false && hook.asyncRewake) {
      // Re-enter permission flow for this tool call
      await reEnterPermissionFlow(hook)
    }
  }))
}
```

This is called from the REPL's main loop on each tick. The hook subprocess
runs independently; the REPL polls until it sees `status === 'completed'`.

---

## §7 — Goto-Line Support

Both `editFileInEditor` and `openFileInExternalEditor` support jumping to a
specific line when opening the file:

```typescript
// utils/editor.ts:57
function guiGotoArgv(guiFamily: string, filePath: string, line?: number): string[] {
  if (!line) return [filePath]
  if (VSCODE_FAMILY.has(guiFamily)) return ['-g', `${filePath}:${line}`]   // code -g file:42
  if (guiFamily === 'subl') return [`${filePath}:${line}`]                  // subl file:42
  return [filePath]
}

// Terminal editors that accept +N:
const PLUS_N_EDITORS = /\b(vi|vim|nvim|nano|emacs|pico|micro|helix|hx)\b/
// spawn: vim +42 file.md
```

For plan review this is useful to jump directly to a failing section when
re-presenting a rejected plan.

---

## §8 — Platform Differences

| Platform | GUI editor invocation | Terminal editor | Notes |
|----------|----------------------|-----------------|-------|
| Linux/macOS | `spawn(base, args, {detached: true})` | `spawnSync(base, args, {stdio:'inherit'})` | POSIX argv array, no shell injection |
| Windows | `spawn(`${editor} ${args}`, {shell:true})` | `spawnSync(`${editor} "${file}"`, {shell:true})` | `.cmd` wrappers need `shell:true`; explicit quoting |
| WSL | Same as Linux | Same as Linux | IDE detection checks WSL distro match |

**Windows gotcha:** `code.cmd` (the VS Code CLI) needs `shell: true` on
Windows because `CreateProcess` cannot execute `.cmd` files directly. The
agent explicitly requests `code.cmd` on Windows to avoid resolving to
`Code.exe` (the GUI binary) which opens a new editor window instead of the
CLI.

---

## §9 — Wiring It All Together: Recommended Pattern

For a new agent or skill that wants plan-review-in-external-editor:

```
Agent finishes exploration → writes plan.md to plans directory
         │
         ▼
ExitPlanMode tool called (or equivalent)
         │
         ▼
[Optional] PreToolUse hook fires
    └── runs: code --wait plan.md        ← user reviews in VSCode
    └── hook exits 0 when tab closed
         │
         ▼
Permission dialog shows (with any user edits already applied)
    ├── User approves → execute phase
    └── User rejects (with feedback) → agent revises plan → loops back
         │
         ▼
Execute phase: write-capable, following the approved plan
```

**Configuration (settings.json):**

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "ExitPlanMode",
        "hooks": [
          {
            "type": "command",
            "command": "code --wait \"$(echo $HOOK_INPUT | jq -r '.inputJson.planFilePath // empty')\"",
            "blocking": true,
            "timeout": 600
          }
        ]
      }
    ]
  }
}
```

Or, for a more robust hook with proper JSON parsing:

```bash
#!/usr/bin/env bash
# ~/.agent/hooks/plan_review.sh
# Called with full hook JSON on stdin

INPUT=$(cat)
PLAN_PATH=$(echo "$INPUT" | jq -r '.inputJson.planFilePath // empty')

if [[ -z "$PLAN_PATH" ]] || [[ ! -f "$PLAN_PATH" ]]; then
  exit 0   # No plan file, allow unconditionally
fi

# Prefer $VISUAL or $EDITOR if set, fall back to code
EDITOR_CMD="${VISUAL:-${EDITOR:-code}}"

# Add --wait for known GUI editors
case "$EDITOR_CMD" in
  code|cursor|windsurf|codium) EDITOR_CMD="$EDITOR_CMD --wait" ;;
  subl) EDITOR_CMD="$EDITOR_CMD --wait" ;;
esac

$EDITOR_CMD "$PLAN_PATH"
```

---

## §10 — Key Source Locations (Claude Code)

| File | What it contains |
|------|-----------------|
| `utils/editor.ts:164` | `getExternalEditor()` — detection + classification |
| `utils/editor.ts:81` | `openFileInExternalEditor()` — detached/blocking open |
| `utils/promptEditor.ts:31` | `editFileInEditor()` — blocking edit with Ink pause/resume |
| `commands/plan/plan.tsx:104` | `/plan open` command handler |
| `utils/plans.ts:79` | `getPlansDirectory()` + `getPlanFilePath()` |
| `tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts:221` | `checkPermissions → 'ask'` (permission gate) |
| `utils/hooks/AsyncHookRegistry.ts:113` | Async hook polling loop |
| `utils/hooks/execPromptHook.ts` | LLM-evaluated hooks (Haiku) |
| `skills/bundledSkills.ts:15` | `BundledSkillDefinition` type (hooks field) |

---

*Related: [Planning Mode & Hooks](01-planning-hooks.md) · [OS & Shell Integration](01-os-shell-integration.md) · [Sessions, Cost & IDE](01-sessions-cost-ide.md)*
