# Hooks System

Hooks are shell commands, HTTP requests, LLM prompts, or agent callbacks that run automatically at lifecycle events. They can inspect, modify, approve, or block operations without changing Claude Code's source code.

---

## Hook Configuration Location

Hooks live in the `hooks` key of any settings file:

```json
// ~/.claude/settings.json  (user scope)
// .claude/settings.json    (project scope, committed)
// .claude/settings.local.json  (local scope, gitignored)
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'About to run bash'"
          }
        ]
      }
    ]
  }
}
```

---

## All 27 Hook Events

Sourced from `src/entrypoints/sdk/coreTypes.ts`:

### Tool Lifecycle
| Event | When | Can block? |
|-------|------|-----------|
| `PreToolUse` | Before any tool runs | Yes — can approve/block/modify input |
| `PostToolUse` | After tool succeeds | No — can add context, modify MCP output |
| `PostToolUseFailure` | After tool throws error | No — observational |

### Session Lifecycle
| Event | When | Can block? |
|-------|------|-----------|
| `SessionStart` | Session begins (before first user message) | No — can inject context + set file watchers |
| `SessionEnd` | Session ends | No — cleanup |
| `Setup` | After session initialized, before `SessionStart` | No — can inject additionalContext |
| `Stop` | Claude stops (after final response) | No — observational |
| `StopFailure` | Claude stops with error | No — observational |

### Sub-Agent Lifecycle
| Event | When | Can block? |
|-------|------|-----------|
| `SubagentStart` | Before sub-agent runs | No — can inject context |
| `SubagentStop` | After sub-agent finishes | No — observational |

### Context Management
| Event | When | Can block? |
|-------|------|-----------|
| `PreCompact` | Before context window compaction | No — observational |
| `PostCompact` | After context window compaction | No — observational |
| `InstructionsLoaded` | When CLAUDE.md instructions are loaded | No — observational |

### Permission Events
| Event | When | Can block? |
|-------|------|-----------|
| `PermissionRequest` | When user permission would be shown | Yes — can approve or deny |
| `PermissionDenied` | When permission is denied | Partial — can set `retry: true` |

### User Interaction
| Event | When | Can block? |
|-------|------|-----------|
| `UserPromptSubmit` | When user submits a message | No — can inject additionalContext |
| `Notification` | When Claude sends a notification | No — can add context |
| `Elicitation` | When Claude requests user input (forms) | Yes — can accept/decline/cancel |
| `ElicitationResult` | After user responds to elicitation | No — observational |

### Task Management
| Event | When | Can block? |
|-------|------|-----------|
| `TaskCreated` | When a task is created | No — observational |
| `TaskCompleted` | When a task completes | No — observational |
| `TeammateIdle` | When a teammate agent becomes idle | No — observational |

### Configuration
| Event | When | Can block? |
|-------|------|-----------|
| `ConfigChange` | When settings change | No — observational |

### Filesystem
| Event | When | Can block? |
|-------|------|-----------|
| `CwdChanged` | When working directory changes | No — can register new file watchers |
| `FileChanged` | When a watched file changes | No — can add context, register more watchers |
| `WorktreeCreate` | When a git worktree is created | No — receives worktreePath |
| `WorktreeRemove` | When a git worktree is removed | No — observational |

---

## 4 Hook Types

### 1. Command Hook (Shell)

Runs a shell command. The hook input is passed as JSON via the `CLAUDE_TOOL_INPUT` env var (or stdin if input is large). Output is read from stdout.

```json
{
  "type": "command",
  "command": "python3 /path/to/my-hook.py",
  "if": "Bash(git *)",
  "shell": "bash",
  "timeout": 30,
  "statusMessage": "Checking git command...",
  "async": false,
  "asyncRewake": false,
  "once": false
}
```

**All fields:**
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `command` | string | required | Shell command to execute |
| `if` | string | — | Permission rule filter (see below) |
| `shell` | `"bash"` / `"powershell"` | `"bash"` | Shell interpreter |
| `timeout` | number | 60s | Timeout in seconds |
| `statusMessage` | string | — | Custom spinner message |
| `async` | bool | `false` | Run in background without blocking |
| `asyncRewake` | bool | `false` | Background; wake model on exit code 2 |
| `once` | bool | `false` | Remove hook after first execution |

### 2. Prompt Hook (LLM Evaluation)

Runs the hook input through a small LLM call. Use `$ARGUMENTS` placeholder for the input JSON.

```json
{
  "type": "prompt",
  "prompt": "The following tool call is about to run: $ARGUMENTS\n\nDoes this look safe? Respond with JSON: {\"decision\": \"approve\"} or {\"decision\": \"block\", \"reason\": \"...\"} ",
  "if": "Bash",
  "model": "claude-haiku-4-5-20251001",
  "timeout": 30,
  "statusMessage": "Evaluating safety...",
  "once": false
}
```

**All fields:**
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `prompt` | string | required | Prompt text. Use `$ARGUMENTS` for input JSON. |
| `if` | string | — | Permission rule filter |
| `model` | string | fast model | Override model for this hook |
| `timeout` | number | 60s | Timeout in seconds |
| `statusMessage` | string | — | Custom spinner message |
| `once` | bool | `false` | Remove hook after first execution |

### 3. HTTP Hook

POSTs the hook input JSON to an endpoint. Reads response body as hook output.

```json
{
  "type": "http",
  "url": "https://my-audit-server.example.com/hook",
  "if": "Write",
  "headers": {
    "Authorization": "Bearer $MY_TOKEN",
    "Content-Type": "application/json"
  },
  "allowedEnvVars": ["MY_TOKEN"],
  "timeout": 10,
  "statusMessage": "Auditing file write...",
  "once": false
}
```

**All fields:**
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `url` | string | required | URL to POST to |
| `if` | string | — | Permission rule filter |
| `headers` | object | — | HTTP headers. Values can reference `$ENV_VAR` syntax. |
| `allowedEnvVars` | string[] | — | Env vars that may be interpolated in headers. **Required** for env var expansion to work. |
| `timeout` | number | 60s | Timeout in seconds |
| `statusMessage` | string | — | Custom spinner message |
| `once` | bool | `false` | Remove hook after first execution |

### 4. Agent Hook (Agentic Verifier)

Spawns a mini-agent to verify/evaluate the hook event. Use `$ARGUMENTS` placeholder for input.

```json
{
  "type": "agent",
  "prompt": "Verify that the bash command $ARGUMENTS does not contain any destructive operations like rm -rf, format, or DROP TABLE.",
  "if": "Bash",
  "model": "claude-haiku-4-5-20251001",
  "timeout": 60,
  "statusMessage": "Verifying command safety...",
  "once": false
}
```

**All fields:**
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `prompt` | string | required | Verification prompt. Use `$ARGUMENTS` for input. |
| `if` | string | — | Permission rule filter |
| `model` | string | Haiku | Model to use |
| `timeout` | number | 60s | Timeout in seconds |
| `statusMessage` | string | — | Custom spinner message |
| `once` | bool | `false` | Remove hook after first execution |

---

## Matcher and `if` Field

### Matcher

The `matcher` field at the `HookMatcher` level is a string pattern applied to the primary value associated with each event:
- For tool hooks: the tool name (e.g., `"Bash"`, `"Write"`)
- Partial string match (not glob): `"Bash"` matches `Bash` tool; `"Write"` matches `Write` tool

```json
{
  "PreToolUse": [
    {
      "matcher": "Bash",
      "hooks": [...]
    },
    {
      "matcher": "Write",
      "hooks": [...]
    }
  ]
}
```

Omitting `matcher` (or setting it to `""`) applies the hook to ALL events of that type.

### `if` Condition

The `if` field inside each individual hook uses **permission rule syntax** — the same syntax as `permissions.allow`:

```
"Bash(git *)"           # Only when Bash runs commands starting with 'git'
"Bash(rm *)"            # Only when Bash runs rm commands
"Read(*.ts)"            # Only when Read opens .ts files
"Write(src/**)"         # Only when Write touches files under src/
"*"                     # Always (same as omitting)
"mcp__my-server__*"     # Any tool from my-server MCP
```

The `if` condition is evaluated before spawning the hook process, avoiding the overhead of launching a process for non-matching calls.

---

## Hook Input Data (stdin / env)

Hooks receive input as JSON either via `CLAUDE_TOOL_INPUT` environment variable (small inputs) or stdin (large inputs). Structure depends on the event:

### PreToolUse / PostToolUse / PostToolUseFailure
```json
{
  "session_id": "abc123",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "git status"
  }
}
```

### UserPromptSubmit
```json
{
  "session_id": "abc123",
  "hook_event_name": "UserPromptSubmit",
  "message": "User's message text"
}
```

### SessionStart
```json
{
  "session_id": "abc123",
  "hook_event_name": "SessionStart"
}
```

### PermissionRequest
```json
{
  "session_id": "abc123",
  "hook_event_name": "PermissionRequest",
  "tool_name": "Bash",
  "tool_input": { "command": "rm -rf build/" },
  "permission_request": { "message": "Allow rm?" }
}
```

### FileChanged
```json
{
  "session_id": "abc123",
  "hook_event_name": "FileChanged",
  "file_path": "/absolute/path/to/changed/file.ts"
}
```

---

## Hook Output (stdout)

Hooks communicate back to Claude Code via JSON on stdout. Empty output or non-JSON output = success with no action.

### Universal Fields
```json
{
  "continue": true,           // false = stop Claude after this hook
  "suppressOutput": false,    // true = hide hook stdout from transcript
  "stopReason": "...",        // Message shown when continue=false
  "decision": "approve",      // 'approve' | 'block' (for blocking hooks)
  "reason": "Explanation",    // Reason for decision
  "systemMessage": "Warning"  // Warning shown to user
}
```

### Async Hook (deferred response)
```json
{
  "async": true,
  "asyncTimeout": 300
}
```
When `asyncRewake: true` in hook config and hook exits with code 2, the model wakes up.

### Per-Event `hookSpecificOutput`

#### PreToolUse
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",       // 'allow' | 'deny' | 'ask'
    "permissionDecisionReason": "...",   // Reason shown to user
    "updatedInput": { "command": "git status --short" },  // Modify tool input
    "additionalContext": "Context injected into conversation"
  }
}
```

#### PostToolUse
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "The output shows 3 errors",
    "updatedMCPToolOutput": { "modified": "output" }  // Replace MCP tool result
  }
}
```

#### UserPromptSubmit
```json
{
  "hookSpecificOutput": {
    "hookEventName": "UserPromptSubmit",
    "additionalContext": "User is asking about topic X. Relevant docs: ..."
  }
}
```

#### SessionStart
```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "Repository context: this is a TypeScript monorepo with 3 packages",
    "initialUserMessage": "Please start by running /doctor",
    "watchPaths": ["/absolute/path/to/watch", "/another/path"]
  }
}
```

#### Setup
```json
{
  "hookSpecificOutput": {
    "hookEventName": "Setup",
    "additionalContext": "Additional system context injected at setup"
  }
}
```

#### SubagentStart
```json
{
  "hookSpecificOutput": {
    "hookEventName": "SubagentStart",
    "additionalContext": "Parent context for sub-agent"
  }
}
```

#### PermissionRequest
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow",
      "updatedInput": { "command": "git status" },    // Optional: modify input
      "updatedPermissions": []                         // Optional: add permanent rules
    }
  }
}
// OR
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "deny",
      "message": "This command is not allowed",
      "interrupt": false
    }
  }
}
```

#### PermissionDenied
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionDenied",
    "retry": true    // true = re-attempt the operation after hook runs
  }
}
```

#### Elicitation / ElicitationResult
```json
{
  "hookSpecificOutput": {
    "hookEventName": "Elicitation",
    "action": "accept",          // 'accept' | 'decline' | 'cancel'
    "content": { "field": "value" }   // Form field values
  }
}
```

#### CwdChanged / FileChanged
```json
{
  "hookSpecificOutput": {
    "hookEventName": "CwdChanged",
    "watchPaths": ["/new/path/to/watch"]   // Add more file watchers
  }
}
```

#### WorktreeCreate
```json
{
  "hookSpecificOutput": {
    "hookEventName": "WorktreeCreate",
    "worktreePath": "/path/to/new/worktree"   // Always included by runtime
  }
}
```

#### Notification
```json
{
  "hookSpecificOutput": {
    "hookEventName": "Notification",
    "additionalContext": "Extra context about this notification"
  }
}
```

---

## Complete Settings.json Hook Example

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/.claude/hooks/bash-safety-check.py",
            "if": "Bash(rm *)",
            "timeout": 10,
            "statusMessage": "Checking rm safety..."
          },
          {
            "type": "agent",
            "prompt": "Evaluate if this bash command is safe: $ARGUMENTS. Respond approve or block.",
            "if": "Bash(sudo *)",
            "model": "claude-haiku-4-5-20251001",
            "timeout": 30
          }
        ]
      },
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "http",
            "url": "https://audit.example.com/log",
            "headers": { "Authorization": "Bearer $AUDIT_TOKEN" },
            "allowedEnvVars": ["AUDIT_TOKEN"],
            "async": true
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write \"$CLAUDE_TOOL_INPUT_PATH\" 2>/dev/null || true",
            "if": "Write(**.ts)",
            "async": true
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "cat ~/.claude/context/global-context.md",
            "statusMessage": "Loading context..."
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/.claude/hooks/inject-repo-context.py",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

---

## Writing Hook Scripts

### Python Example (command hook with JSON output)

```python
#!/usr/bin/env python3
import json
import os
import sys

# Read hook input
hook_input_json = os.environ.get('CLAUDE_TOOL_INPUT', '')
if not hook_input_json:
    hook_input_json = sys.stdin.read()

try:
    hook_input = json.loads(hook_input_json)
except json.JSONDecodeError:
    hook_input = {}

tool_name = hook_input.get('tool_name', '')
tool_input = hook_input.get('tool_input', {})

# Example: block dangerous rm commands
if tool_name == 'Bash':
    command = tool_input.get('command', '')
    if 'rm -rf /' in command:
        output = {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": "Refusing to delete root filesystem"
            }
        }
        print(json.dumps(output))
        sys.exit(0)

# Allow by default (empty output = success)
sys.exit(0)
```

### Shell Example (inject git context at session start)

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/inject-git-context.sh
# Hook event: SessionStart

if git rev-parse --git-dir > /dev/null 2>&1; then
    BRANCH=$(git branch --show-current)
    LAST_COMMIT=$(git log --oneline -1)
    CHANGED=$(git diff --name-only HEAD | head -20)
    
    cat << EOF
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "Git context: branch=${BRANCH}, last commit: ${LAST_COMMIT}, changed files: ${CHANGED}"
  }
}
EOF
fi
```

### Node.js Example (async HTTP audit hook)

```javascript
#!/usr/bin/env node
const input = JSON.parse(process.env.CLAUDE_TOOL_INPUT || '{}');

// Fire-and-forget audit log
fetch('https://audit.example.com/log', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    timestamp: new Date().toISOString(),
    event: input.hook_event_name,
    tool: input.tool_name,
    input: input.tool_input,
    session: input.session_id
  })
}).catch(() => {}); // Never fail the hook

// Exit immediately (async: true in config handles non-blocking)
process.exit(0);
```

---

## Hook Execution Order

When multiple hooks match the same event:
1. All matching matchers for the event are evaluated
2. Within a matcher, hooks execute sequentially
3. Multiple matchers execute in config order
4. `decision: block` from any hook stops subsequent hooks for that event

For `async: true` hooks, they run concurrently in background and don't affect the execution order of non-async hooks.

---

## asyncRewake Pattern

The `asyncRewake` pattern allows a background process to wake up the model:

```json
{
  "type": "command",
  "command": "~/.claude/hooks/watch-tests.sh",
  "asyncRewake": true
}
```

In the hook script:
```bash
#!/usr/bin/env bash
# Run tests in background, wake model if they fail
npm test > /tmp/test-results.txt 2>&1
EXIT_CODE=$?

if [ $EXIT_CODE -ne 0 ]; then
    # Exit code 2 = wake the model
    echo '{"systemMessage": "Tests failed! See /tmp/test-results.txt"}'
    exit 2
fi
exit 0
```

Exit codes:
- `0` — Success, no action
- `1` — Non-blocking error (logged, hook continues)
- `2` (with `asyncRewake`) — Wake the model with optional output
