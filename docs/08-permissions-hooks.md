# Permissions & Hooks

## Permission System Overview

Every tool execution must pass through the permission system. This is a layered, rule-based system that classifies each tool invocation as ALLOW, DENY, or ASK.

```
Tool call from model
       │
       ▼
canUseTool(tool, input, permissionContext)
       │
  ┌────┴────────────────────────────────────────┐
  │                                              │
  │  1. Check global mode (bypass → ALLOW all)  │
  │                                              │
  │  2. Check alwaysDenyRules:                  │
  │     "Bash(rm -rf *)" → DENY immediately     │
  │                                              │
  │  3. Check alwaysAllowRules:                 │
  │     "Bash(git *)" → ALLOW immediately       │
  │                                              │
  │  4. Check alwaysAskRules:                   │
  │     "WebFetch(*)" → always ASK              │
  │                                              │
  │  5. Tool-specific heuristics:               │
  │     BashTool: parse command for safety      │
  │     FileEditTool: check path constraints    │
  │     AgentTool: check agent allow list       │
  │                                              │
  │  6. Default by tool sensitivity:            │
  │     Read-only → ALLOW                       │
  │     Modifying → ASK                         │
  └──────────────────────────────────────────────┘
       │
  ┌────┴────────────────────┐
  │ ALLOW: execute now      │
  │ DENY: return error      │  
  │ ASK: show dialog → wait │
  └─────────────────────────┘
```

---

## Permission Modes

```typescript
type PermissionMode =
  | 'default'   // Standard interactive mode
  | 'bypass'    // Skip all prompts (--dangerously-skip-permissions)
  | 'plan'      // Plan mode: read-only until user approves
  | 'auto'      // Automated mode: classifier-based decisions
```

### Default Mode
- Read-only tools (Read, Glob, Grep): auto-allowed
- Modifying tools (Edit, Write, Bash): ask on first use, remember decision
- Dangerous tools (rm, sudo, git push): always ask

### Bypass Mode
Enabled with `--dangerously-skip-permissions`. Skips ALL prompts. For trusted automation only.

### Plan Mode
Entered via `EnterPlanModeTool`. Claude can only read/analyze until the user explicitly approves the plan. Then returns to normal mode.

### Auto Mode
Uses a classifier to auto-approve safe operations without prompting. Configurable sensitivity.

---

## Permission Rules Format

Stored in settings.json under `permissions`:

```json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(npm run *)",
      "Bash(npm test)",
      "Edit",
      "Write",
      "Read"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(sudo *)",
      "Bash(curl * | bash)"
    ],
    "ask": [
      "WebFetch(*)",
      "WebSearch(*)"
    ]
  }
}
```

### Rule Pattern Syntax

| Pattern | Matches |
|---------|---------|
| `Bash` | Any Bash invocation |
| `Bash(git *)` | Bash with command starting with `git ` |
| `Bash(git status)` | Bash with exact command `git status` |
| `Edit` | Any file edit |
| `Edit(/src/*)` | File edits in /src/ |
| `WebFetch(https://api.example.com/*)` | Fetches to specific domain |
| `mcp__server__tool` | Specific MCP tool |
| `Agent(*)` | Any sub-agent spawn |

---

## Permission Rule Sources

Rules can come from different sources, each at a different trust level:

```typescript
type ToolPermissionRulesBySource = {
  globalSettings?: string[]    // ~/.claude/settings.json
  projectSettings?: string[]   // .claude/settings.json
  localSettings?: string[]     // .claude/settings.local.json
  managedSettings?: string[]   // Remote managed settings
  mdm?: string[]               // MDM policy
  session?: string[]           // Granted during current session
}
```

Higher-priority sources can override lower-priority ones.

---

## Interactive Permission Dialog

When a tool invocation requires ASK, the REPL shows a permission dialog:

```
┌─────────────────────────────────────────────┐
│  Claude wants to run a Bash command:         │
│                                              │
│  $ git push origin main --force              │
│                                              │
│  [Allow]  [Always Allow]  [Deny]  [Always Deny]
│  [Allow This Session]                        │
└─────────────────────────────────────────────┘
```

User choices:
- **Allow** — allow this one invocation
- **Always Allow** — add to allow rules in project settings
- **Deny** — deny this one invocation (Claude sees error)
- **Always Deny** — add to deny rules in project settings
- **Allow This Session** — allow for rest of session (not persisted)

---

## Hooks System

Hooks are shell commands that execute at lifecycle events.

### Hook Events

| Event | When | Common Uses |
|-------|------|-------------|
| `PreToolUse` | Before each tool call | Logging, blocking, input modification |
| `PostToolUse` | After each tool call | Post-processing, notifications |
| `Stop` | When model stops (end_turn) | Notifications, auto-commits, CI triggers |
| `SubagentStop` | When sub-agent stops | Coordination, cleanup |
| `Notification` | On system notification | Alert forwarding |
| `PreCompact` | Before context compaction | Save state before summary |

### Hook Configuration (settings.json)

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "notify-send 'Claude is done'"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Running bash: $CLAUDE_TOOL_INPUT' >> /tmp/audit.log"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write $CLAUDE_FILE_PATH 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

### Hook Execution Environment

Hooks run in **embedded Node.js subprocesses** (`spawn()`), completely separate from the user's terminal. The user never sees hook I/O directly.

| Platform | Shell |
|----------|-------|
| Unix/macOS | `/bin/sh` via `spawn(cmd, [], { shell: true })` |
| Windows | Git Bash (Cygwin) — paths must be POSIX |
| `shell: 'powershell'` | `pwsh -NoProfile -NonInteractive -Command <cmd>` |

### Hook Environment Variables

The subprocess inherits the parent `process.env` plus these injected variables:

| Variable | Set When | Description |
|----------|----------|-------------|
| `CLAUDE_PROJECT_DIR` | Always | Stable project root (never worktree path; POSIX on Windows) |
| `CLAUDE_PLUGIN_ROOT` | Plugin/skill hooks | Plugin or skill install path |
| `CLAUDE_PLUGIN_DATA` | Plugin hooks | Plugin data directory |
| `CLAUDE_PLUGIN_OPTION_*` | Plugin hooks | Per-option values (e.g. `CLAUDE_PLUGIN_OPTION_API_KEY`) |
| `CLAUDE_ENV_FILE` | SessionStart/Setup/CwdChanged/FileChanged (bash only) | Path to `.sh` file hook writes `VAR=value` exports into; injected into subsequent bash commands |
| `CLAUDE_CODE_SHELL_PREFIX` | bash only | Inherited command prefix wrapper |

**Note:** Tool context (tool name, input, output) is **not** in env vars — it is sent via stdin JSON.

In GitHub Actions only, secrets are scrubbed from the inherited env (`ANTHROPIC_API_KEY`, `AWS_SECRET_ACCESS_KEY`, etc.).

#### `CLAUDE_PLUGIN_OPTION_*` — Full Details

Each key in the plugin's saved options is exposed as `CLAUDE_PLUGIN_OPTION_{KEY}` where `{KEY}` is the option key uppercased (e.g. `apiKey` → `CLAUDE_PLUGIN_OPTION_APIKEY`). Key sanitization: `key.replace(/[^A-Za-z0-9_]/g, '_').toUpperCase()`. The manifest schema already constrains keys to `/^[A-Za-z_]\w*$/` so this is belt-and-suspenders.

**Storage split at save time** (`pluginOptionsStorage.ts`):

| Option schema `sensitive` | Storage location |
|--------------------------|-----------------|
| `false` / absent | `settings.json → pluginConfigs[pluginId].options` |
| `true` | macOS keychain or `.credentials.json` → `secureStorage.pluginSecrets[pluginId]` |

At load time both are merged (`secureStorage` wins on collision). **Sensitive values ARE included** in `CLAUDE_PLUGIN_OPTION_*` env vars — hooks run user code, same trust boundary as reading the keychain directly.

Options are memoized per-`pluginId` for the session lifetime (cleared by `/reload-plugins` or any settings change).

**Template substitution in hook command strings** — before spawn, the command string is processed for `${user_config.KEY}` references (throws on missing key — plugin authoring bug). In skill/agent prose, `${user_config.KEY}` for a sensitive field renders as a placeholder instead of the real value so secrets don't go into the model context.

**Plugin manifest `userConfig` schema** (field per option):

```json
{
  "userConfig": {
    "apiKey": {
      "type": "string",
      "title": "API Key",
      "description": "Your API key — get one at example.com",
      "required": true,
      "sensitive": true
    },
    "maxRetries": {
      "type": "number",
      "title": "Max Retries",
      "description": "How many times to retry on failure",
      "default": 3,
      "min": 1,
      "max": 10
    },
    "outputDir": {
      "type": "directory",
      "title": "Output directory",
      "description": "Where to write output files"
    }
  }
}
```

Supported `type` values: `string`, `number`, `boolean`, `directory`, `file`. The `multiple: true` flag (string type) allows an array of strings.

### Hook Wire Protocol

**stdin** — single JSON line terminated with `\n`:

```json
{"session_id":"abc-123","transcript_path":"/abs/path/transcript.jsonl","cwd":"/home/user/project","hook_event_name":"PreToolUse","tool_name":"Bash","tool_input":{"command":"git status"},"tool_use_id":"toolu_01abc"}
```

**stdout** — either JSON (if starts with `{`) or plain text:

```json
{
  "continue": true,
  "decision": "approve",
  "reason": "string shown to user",
  "systemMessage": "string injected into model context",
  "suppressOutput": false
}
```

Plain text stdout is shown as a message in the transcript.

**Exit code semantics:**

| Code | Behavior |
|------|----------|
| `0` | Success. Stdout consumed normally. |
| `1` | Non-blocking error. Execution continues; stderr shown to user. |
| `2` | **Blocking.** Action blocked; stderr becomes the block reason shown to user. |
| `>2` | Same as `1` (non-blocking). |

Note: exit 0 with `{"decision":"block"}` in stdout is also treated as a block.

**Default timeout:** 10 minutes (`TOOL_HOOK_EXECUTION_TIMEOUT_MS`). Per-hook override: `hook.timeout` (seconds). `SessionEnd`/`Stop` default is **1.5 s** (override via `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS`).

### Async Hooks

A hook can background itself by emitting `{"async": true}` as the **first line** of stdout — Claude resumes immediately while the hook continues running. Alternatively set `async: true` or `asyncRewake: true` in config.

- `async: true` — fire-and-forget; exit code ignored
- `asyncRewake: true` — background hook; if it exits with code `2`, the result is enqueued as a task-notification that wakes the model

### Hook Output to Model

When stdout is plain text, it is injected into the conversation as a system message:

```bash
#!/bin/bash
INPUT=$(cat)   # stdin is the authoritative JSON source
TOOL=$(echo "$INPUT" | jq -r '.tool_name')
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // "n/a"')
echo "Audit: [$TOOL] $CMD" >> ~/.agent/audit.log
# Any echo to stdout here becomes a system message in the conversation
```

---

## Denial Tracking (`src/utils/permissions/denialTracking.ts`)

When a tool is denied, Claude Code tracks the denial to:
1. Avoid repeatedly asking the same question
2. Provide better context to the model about what's not allowed
3. Inform the permission learning system

```typescript
type DenialTrackingState = {
  deniedTools: Map<string, {
    toolName: string
    input: Record<string, unknown>
    deniedAt: number
    reason: 'user-denied' | 'rule-denied'
  }>
}
```

---

## Read-Only Mode / Plan Mode Constraints

In plan mode (`EnterPlanMode`), the system enforces read-only operations:

```typescript
function checkReadOnlyConstraints(
  tool: Tool,
  input: Record<string, unknown>,
): boolean {
  const readOnlyTools = new Set(['Read', 'Glob', 'Grep', 'Bash'])
  
  if (!readOnlyTools.has(tool.name)) {
    return false  // Non-read-only tool: blocked
  }
  
  if (tool.name === 'Bash') {
    // Only allow read-only bash commands
    return isReadOnlyBashCommand(input.command as string)
  }
  
  return true
}
```

---

## Permission Dialog Components (`src/components/`)

Each tool type has a specialized permission dialog:

- `BashPermissionRequest.tsx` — Shows command with syntax highlighting
- `FileEditPermissionRequest.tsx` — Shows before/after diff
- `FileWritePermissionRequest.tsx` — Shows file content preview
- `WebFetchPermissionRequest.tsx` — Shows URL
- `SkillPermissionRequest.tsx` — Shows skill description
- `EnterPlanModePermissionRequest.tsx` — Shows plan summary
- `AgentPermissionRequest.tsx` — Shows agent prompt preview
- `McpPermissionRequest.tsx` — Shows MCP tool + input
