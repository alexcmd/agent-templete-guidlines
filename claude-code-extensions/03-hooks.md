# Hooks System

Hooks are shell commands, HTTP requests, LLM prompts, or agent callbacks that run automatically at lifecycle events. They can inspect, modify, approve, or block operations without changing Claude Code's source code.

> **Source-verified against:** `src/utils/hooks.ts`, `src/types/hooks.ts`, `src/schemas/hooks.ts`, `src/utils/hooks/` (execPromptHook, execAgentHook, execHttpHook, sessionHooks, hooksConfigManager)

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

Sourced from `src/entrypoints/sdk/coreTypes.ts` (HOOK_EVENTS array) and `src/utils/hooks/hooksConfigManager.ts`.

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

Runs a shell command. **Hook input arrives as JSON on stdin.** For small inputs, `CLAUDE_TOOL_INPUT` env var is also set as a convenience; always prefer stdin as it handles large payloads.

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

**Only available for:** `PreToolUse`, `PostToolUse`, `PermissionRequest`

The LLM **must** return: `{"ok": true}` (allow) or `{"ok": false, "reason": "..."}` (block).

```json
{
  "type": "prompt",
  "prompt": "The following tool call is about to run:\n$ARGUMENTS\n\nIs this safe? Respond ONLY with JSON: {\"ok\": true} or {\"ok\": false, \"reason\": \"...\"}",
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
| `timeout` | number | 30s | Timeout in seconds |
| `statusMessage` | string | — | Custom spinner message |
| `once` | bool | `false` | Remove hook after first execution |

### 3. HTTP Hook

POSTs the hook input JSON to an endpoint. Reads response body as hook output.

**Not supported for:** `SessionStart`, `Setup`

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
| `timeout` | number | 600s | Timeout in seconds (10 minutes default) |
| `statusMessage` | string | — | Custom spinner message |
| `once` | bool | `false` | Remove hook after first execution |

**Security:** URL must be in `allowedHttpHookUrls` setting (enterprise). Header values are sanitized to prevent CRLF injection. Resolved IPs are SSRF-guarded unless a proxy is configured.

### 4. Agent Hook (Agentic Verifier)

Spawns a full multi-turn sub-agent to verify/evaluate the hook event. Has access to all tools except nested agent spawning and Stop hooks. Max 50 turns.

**Only available for:** `PreToolUse`, `PostToolUse`, `PermissionRequest`

The agent **must** return: `{"ok": true}` (allow) or `{"ok": false, "reason": "..."}`.

```json
{
  "type": "agent",
  "prompt": "Verify that the bash command:\n$ARGUMENTS\n...does not contain destructive operations like rm -rf, format, or DROP TABLE.\nRespond with: {\"ok\": true} or {\"ok\": false, \"reason\": \"...\"}",
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

## Exit Codes (command hooks)

Exit codes are the primary control mechanism for command hooks:

| Code | Name | Behavior |
|------|------|----------|
| `0` | Success | Hook passed. Stdout shown to user (unless `suppressOutput: true` in JSON). |
| `2` | Blocking error | Hook **blocks** the action. Stderr shown to the **model** as context. JSON on stdout is parsed for `hookSpecificOutput`. |
| Any other | Non-blocking error | Hook failed but action is **not blocked**. Stderr shown to the **user** only (not model). |

**Critical distinction:** exit code `2` sends stderr to the **model**; non-zero codes send stderr to the **user**. Use `2` when you want the model to know why it was blocked.

```bash
#!/usr/bin/env bash
# Exit 2 = blocking: model sees stderr
echo "Blocked: rm -rf is not allowed" >&2
exit 2

# Exit 1 = warning: user sees stderr, action proceeds
echo "Warning: no tests found" >&2
exit 1
```

---

## Matcher and `if` Field

### Matcher

The `matcher` field at the `HookMatcher` level filters which events trigger the hook group. The matched field varies by event type:

| Event(s) | Matched field |
|----------|---------------|
| `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest`, `PermissionDenied` | `tool_name` |
| `SessionStart`, `ConfigChange`, `InstructionsLoaded` | `source` |
| `Setup`, `PreCompact`, `PostCompact` | `trigger` |
| `Notification` | `notification_type` |
| `SessionEnd` | `reason` |
| `StopFailure` | `error` |
| `SubagentStart`, `SubagentStop` | `agent_type` |
| `Elicitation`, `ElicitationResult` | `mcp_server_name` |
| `FileChanged` | `basename(file_path)` |
| `TeammateIdle`, `TaskCreated`, `TaskCompleted` | (no matcher — always fires) |

Matching is **case-insensitive substring** (pipe-separated for OR):

```json
{
  "PreToolUse": [
    { "matcher": "Bash",         "hooks": [...] },
    { "matcher": "Write|Edit",   "hooks": [...] },
    { "matcher": "",             "hooks": [...] }   // empty = all tools
  ]
}
```

### `if` Condition

The `if` field inside each individual hook uses **permission rule syntax** — the same syntax as `permissions.allow`. Evaluated before spawning the process:

```
"Bash(git *)"           # Only when Bash runs commands starting with 'git'
"Bash(rm *)"            # Only when Bash runs rm commands
"Read(*.ts)"            # Only when Read opens .ts files
"Write(src/**)"         # Only when Write touches files under src/
"*"                     # Always (same as omitting)
"mcp__my-server__*"     # Any tool from my-server MCP
```

`if` is supported for: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest`

---

## Hook Input Data (stdin)

Hooks receive a JSON object on stdin. The structure varies by event type.

### Common Fields (all events)

```json
{
  "session_id": "abc123",
  "hook_event_name": "PreToolUse",
  "transcript_path": "/tmp/claude-sessions/abc123/transcript.jsonl",
  "cwd": "/home/user/myproject",
  "permission_mode": "ask"
}
```

| Field | Description |
|-------|-------------|
| `session_id` | Unique session identifier |
| `hook_event_name` | The event name (string) |
| `transcript_path` | Path to JSONL transcript of the current session |
| `cwd` | Current working directory |
| `permission_mode` | Current permission mode: `"ask"` / `"auto"` / `"dontAsk"` |

### PreToolUse / PostToolUse / PostToolUseFailure

```json
{
  "session_id": "abc123",
  "hook_event_name": "PreToolUse",
  "transcript_path": "/tmp/claude-sessions/abc123/transcript.jsonl",
  "cwd": "/home/user/project",
  "permission_mode": "ask",
  "tool_name": "Bash",
  "tool_input": {
    "command": "git status"
  },
  "tool_use_id": "toolu_01abc",
  "agent_id": "subagent-xyz",       // only for sub-agent tool calls
  "agent_type": "explore"           // only for sub-agent tool calls
}
```

### UserPromptSubmit

```json
{
  "session_id": "abc123",
  "hook_event_name": "UserPromptSubmit",
  "transcript_path": "...",
  "cwd": "/home/user/project",
  "permission_mode": "ask",
  "message": "User's message text"
}
```

### SessionStart

```json
{
  "session_id": "abc123",
  "hook_event_name": "SessionStart",
  "transcript_path": "...",
  "cwd": "/home/user/project",
  "permission_mode": "ask"
}
```

### PermissionRequest

```json
{
  "session_id": "abc123",
  "hook_event_name": "PermissionRequest",
  "transcript_path": "...",
  "cwd": "/home/user/project",
  "permission_mode": "ask",
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
  "transcript_path": "...",
  "cwd": "/home/user/project",
  "permission_mode": "ask",
  "file_path": "/absolute/path/to/changed/file.ts"
}
```

---

## Hook Output (stdout)

Hooks communicate back to Claude Code via JSON on stdout.

> **JSON parsing rule:** stdout is only parsed as JSON if it **starts with `{`**. Any other output is treated as plain text. Empty output = success with no action.

### Universal Fields

```json
{
  "continue": true,           // false = stop Claude after this hook
  "suppressOutput": false,    // true = hide hook stdout from transcript
  "stopReason": "...",        // Message shown when continue=false
  "decision": "approve",      // 'approve' | 'block' (legacy; prefer hookSpecificOutput)
  "reason": "Explanation",    // Reason for decision
  "systemMessage": "Warning"  // Warning shown in UI
}
```

**Blocking logic (priority order):**
1. `hookSpecificOutput.permissionDecision == "deny"` (PreToolUse)
2. `decision == "block"` in root JSON
3. Exit code `2`

Any of these blocks the action. `decision: "block"` with exit code `0` works the same as exit code `2`.

### Async Hook (deferred response)

```json
{
  "async": true,
  "asyncTimeout": 300
}
```

Emit this as the **first line** of stdout to move the hook to the background immediately.

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
```

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "deny",
      "message": "This command is not allowed"
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
    "worktreePath": "/path/to/new/worktree"   // Provided by runtime
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

## Environment Variables Available to Hooks

| Variable | Available for | Description |
|----------|--------------|-------------|
| `CLAUDE_PROJECT_DIR` | All hooks | Project root directory |
| `CLAUDE_TOOL_INPUT` | Command hooks (small payloads) | Hook input JSON as convenience copy of stdin |
| `CLAUDE_ENV_FILE` | `SessionStart`, `Setup`, `CwdChanged`, `FileChanged` | Path to a `.sh` file; write `export VAR=value` lines here to inject env vars into subsequent bash commands |
| `CLAUDE_PLUGIN_ROOT` | Hooks inside plugins | Plugin directory |
| `CLAUDE_PLUGIN_DATA` | Hooks inside plugins | Plugin data directory |
| `CLAUDE_PLUGIN_OPTION_<KEY>` | Hooks inside plugins | Plugin config option (key uppercased + sanitized) |

**`CLAUDE_ENV_FILE` example** — hook sets an environment variable for the session:
```bash
#!/usr/bin/env bash
# SessionStart hook: export tool version for use in later commands
echo "export MY_TOOL_VERSION=2.0" >> "$CLAUDE_ENV_FILE"
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
            "prompt": "Evaluate if this bash command is safe:\n$ARGUMENTS\nRespond: {\"ok\": true} or {\"ok\": false, \"reason\": \"...\"}",
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
            "command": "npx prettier --write \"$(echo $CLAUDE_TOOL_INPUT | jq -r '.tool_input.file_path')\" 2>/dev/null || true",
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

### Python Example — PreToolUse blocker with JSON output

```python
#!/usr/bin/env python3
import json
import sys

# Always read from stdin — handles large payloads that don't fit in env var
hook_input = json.loads(sys.stdin.read() or "{}")

tool_name = hook_input.get("tool_name", "")
tool_input = hook_input.get("tool_input", {})

if tool_name == "Bash":
    command = tool_input.get("command", "")
    if "rm -rf /" in command:
        # Exit 2 = blocking; stderr goes to MODEL as context
        print(json.dumps({
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": "Refusing to delete root filesystem"
            }
        }))
        sys.exit(0)  # JSON output handles the block; exit 2 also works

# Allow by default
sys.exit(0)
```

### Shell Example — inject git context at session start

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

### Node.js Example — async HTTP audit hook

```javascript
#!/usr/bin/env node
let raw = "";
process.stdin.on("data", d => raw += d);
process.stdin.on("end", () => {
  const input = JSON.parse(raw || "{}");

  fetch("https://audit.example.com/log", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      timestamp: new Date().toISOString(),
      event: input.hook_event_name,
      tool: input.tool_name,
      input: input.tool_input,
      session: input.session_id
    })
  }).catch(() => {}); // Never fail the hook

  process.exit(0);
});
```

---

## Hook Execution Order

1. All matcher groups for the event are evaluated in config order
2. Within a matcher, hooks execute **sequentially**
3. `decision: "block"` or exit code `2` from any hook stops all subsequent hooks for that event
4. `async: true` hooks run concurrently in background and don't block the main sequence
5. **Deduplication:** hooks with identical `{command, shell, if}` (or `{prompt, if}` / `{url, if}`) from the same source are deduplicated and run only once per event

---

## asyncRewake Pattern

Lets a background process wake the model after it finishes:

```json
{
  "type": "command",
  "command": "~/.claude/hooks/watch-tests.sh",
  "asyncRewake": true
}
```

```bash
#!/usr/bin/env bash
# Run tests in background; wake model if they fail
npm test > /tmp/test-results.txt 2>&1
EXIT_CODE=$?

if [ $EXIT_CODE -ne 0 ]; then
    echo '{"systemMessage": "Tests failed! See /tmp/test-results.txt"}' >&1
    exit 2   # exit 2 = enqueue a wake notification to the model
fi
exit 0
```

**asyncRewake exit code semantics:**
- `0` — background job succeeded; no wake
- `2` — wake the model (stdout JSON forwarded as notification context)
- Other — logged, no wake

---

## Session-End Hook Timeout

`SessionEnd` / `Stop` hooks have a **very short** default timeout of **1.5 seconds** (not 60s). Override with:

```bash
export CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS=5000  # 5 seconds
```

This is intentional — Claude Code exits quickly after the session ends.

---

*[← Custom Agents](02-agents.md) | [Next: Plugins →](04-plugins.md)*
