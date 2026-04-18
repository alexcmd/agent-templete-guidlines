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

### Hook Environment Variables

Available in hook commands:

| Variable | Available In | Description |
|----------|-------------|-------------|
| `CLAUDE_TOOL_NAME` | Pre/PostToolUse | Tool being called |
| `CLAUDE_TOOL_INPUT` | Pre/PostToolUse | JSON input to tool |
| `CLAUDE_TOOL_OUTPUT` | PostToolUse | Tool's output |
| `CLAUDE_FILE_PATH` | Edit/Write hooks | File path being modified |
| `CLAUDE_SESSION_ID` | All | Current session ID |
| `CLAUDE_AGENT_ID` | SubagentStop | Sub-agent's ID |

### Hook Execution Model

```typescript
// hooks.ts
async function runHooks(
  event: HookEvent,
  context: HookContext,
): Promise<void> {
  const hooks = getHooksForEvent(event)
  
  for (const hookConfig of hooks) {
    // Check matcher
    if (hookConfig.matcher && !matchesPattern(hookConfig.matcher, context)) {
      continue
    }
    
    for (const hook of hookConfig.hooks) {
      if (hook.type === 'command') {
        const env = buildHookEnvironment(context)
        const result = await exec(hook.command, {
          env,
          timeout: hook.timeout ?? 60_000,
        })
        
        // Hook output is shown to model as system message
        if (result.stdout) {
          yield createSystemMessage({ content: result.stdout })
        }
        
        // Non-zero exit = hook blocked the action
        if (result.exitCode !== 0) {
          throw new HookBlockedError(result.stderr)
        }
      }
    }
  }
}
```

### Hook Output to Model

When a hook produces stdout, it's injected into the conversation as a system message. This allows hooks to communicate with the model:

```bash
# This hook output tells Claude there's an issue
#!/bin/bash
if git diff --cached | grep -q "TODO"; then
  echo "WARNING: Committing with TODO comments. Please resolve them first."
  exit 1
fi
```

The model sees: `WARNING: Committing with TODO comments. Please resolve them first.`

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
