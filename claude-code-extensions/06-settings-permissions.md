# Settings, Configuration & Permissions

Understanding the settings system is essential for all extension types — hooks, MCP servers, permissions, and plugins all live in settings files.

---

## Settings File Hierarchy

Claude Code merges settings from 5 sources, in priority order (higher = overrides lower):

| Priority | Source | File | Scope |
|----------|--------|------|-------|
| 5 (highest) | `policySettings` | `/etc/claude/managed-settings.json` (Linux) | Enterprise — users cannot override |
| 4 | `flagSettings` | `--settings /path/to/file` | CLI flag |
| 3 | `localSettings` | `.claude/settings.local.json` | Project-local (gitignored) |
| 2 | `projectSettings` | `.claude/settings.json` | Project (committed to repo) |
| 1 (lowest) | `userSettings` | `~/.claude/settings.json` | User-global |

Also special sources:
- `dynamic` — Runtime mutations (from SDK, `/config` tool, feature flags)
- `enterprise` — Enterprise remote config via API
- `claudeai` — Claude.ai web app settings
- `managed` — Alias for policySettings

Merging strategy:
- Scalar values: higher priority wins
- `permissions.allow/deny/ask`: **merged** (all sources contribute rules)
- `hooks`: **merged by event** (all hooks from all sources run)
- `mcpServers`: higher priority wins per server name
- `enabledPlugins`: user setting wins over default

---

## Settings JSON Structure

Full settings.json schema:

```json
{
  // Model selection
  "model": "claude-sonnet-4-6",

  // Per-file model overrides (gitignore-style patterns → model)
  "modelOverrides": {
    "**.ts": "claude-opus-4-7",
    "**.md": "claude-haiku-4-5-20251001"
  },

  // Permission rules
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Read",
      "mcp__github__*"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(sudo *)"
    ],
    "ask": [
      "Write(**.json)"
    ]
  },

  // Default permission mode for new sessions
  "defaultPermissionMode": "default",

  // Hooks (all sources merged)
  "hooks": {
    "PreToolUse": [...],
    "PostToolUse": [...],
    "SessionStart": [...],
    "UserPromptSubmit": [...],
    "Stop": [...],
    "PermissionRequest": [...]
  },

  // MCP servers
  "mcpServers": {
    "server-name": {
      "type": "stdio",
      "command": "...",
      "args": [...]
    }
  },

  // Plugin enable/disable state
  "enabledPlugins": {
    "my-plugin@marketplace": true,
    "other@builtin": false
  },

  // Environment variables injected into tool calls
  "env": {
    "MY_API_KEY": "secret",
    "NODE_ENV": "development"
  },

  // UI
  "theme": "dark",
  "verbosityLevel": 1,

  // Context window
  "contextWindowOverride": {
    "maxTokens": 100000
  },

  // Output styles
  "outputStyles": {
    "default": "standard"
  },

  // Compact settings
  "compactThreshold": 0.8,

  // Feature flags (internal, may change)
  "featureFlags": {
    "FEATURE_NAME": true
  }
}
```

---

## Permission Rule Syntax

Used in `permissions.allow`, `permissions.deny`, `permissions.ask`, and hook `if` conditions.

### Basic Forms

```
ToolName                          # Match tool by name
ToolName(pattern)                 # Match tool + input pattern
*                                 # Match all tools
mcp__server-name                  # All tools from MCP server
mcp__server-name__tool-name       # Specific MCP tool
```

### Tool-Specific Pattern Matching

For `Bash`, the pattern matches the **command string**:
```
Bash(git *)              # Any git command
Bash(git commit *)       # Only git commit
Bash(npm *)              # Any npm command
Bash(rm *)               # Any rm command (usually in deny list)
Bash(sudo *)             # Any sudo command
```

For `Read`, `Write`, `Edit`, `Glob`, `Grep`, the pattern matches the **file path**:
```
Read(*.ts)               # Read any .ts file
Write(src/**)            # Write any file under src/
Edit(**.json)            # Edit any .json file
Read(/etc/**)            # Read anything under /etc (usually denied)
```

For MCP tools, patterns match the tool input:
```
mcp__github__create-pr   # Specific GitHub tool
mcp__slack__*            # All Slack tools
```

### Glob Patterns

Uses micromatch/minimatch:
- `*` — any characters except `/`
- `**` — any characters including `/`
- `?` — single character
- `[abc]` — character class
- `!pattern` — negation (in paths frontmatter, not permissions)

---

## Permission Modes

Set per-session or as default:

| Mode | Behavior |
|------|---------|
| `default` | Prompt for each new permission; remember within session |
| `dontAsk` | Never prompt; allow if allowed by rules, deny if denied |
| `plan` | Read-only mode: only Bash(read-only), Read, Glob, Grep |
| `bypassPermissions` | Skip ALL permission checks (requires explicit user opt-in) |
| `auto` | Uses Bash security classifier; auto-allows benign commands |
| `acceptEdits` | Auto-accept all file edits (Write, Edit) without prompting |

Set default mode in settings:
```json
{
  "defaultPermissionMode": "dontAsk"
}
```

Set via CLI flag:
```bash
claude --dangerously-skip-permissions   # bypassPermissions
claude --permission-mode dontAsk
```

---

## Permission Decision Flow

When a tool is about to run:

```
1. bypassPermissions mode? → Allow immediately
2. plan mode? → Check if tool is read-only (Bash read commands, Read, Glob, Grep)
3. Check deny rules (any source) → Deny if matched
4. Check allow rules (any source) → Allow if matched  
5. Check ask rules → Ask user if matched
6. Tool-specific checkPermissions() → May allow/deny/ask
7. PreToolUse hooks → May approve/block/modify
8. auto mode? → Run Bash security classifier
9. Prompt user → Remember choice for session
```

**First-match wins** within each rule list. But deny, allow, and ask lists are checked as separate groups (deny first, then allow, then ask).

---

## Permissions in Extension Context

### Granting tools to skills

In a skill's `allowed-tools` frontmatter — tools listed here get auto-allowed during skill's shell injection:
```markdown
---
allowed-tools: "Bash, Read, Glob"
---
```

### Granting tools to agents

In an agent's `tools` frontmatter — only listed tools are available:
```markdown
---
tools: Bash, Read, Write, Glob
disallowedTools: WebFetch
---
```

### Hook permissions

Hooks run with the permission context of the session, but hooks themselves don't need to declare permissions. However, if a command hook uses `Bash`, it bypasses the normal tool permission flow (hooks are trusted system code).

---

## Environment Variables in Settings

The `env` key injects environment variables into tool execution contexts:

```json
{
  "env": {
    "MY_SECRET": "value",
    "DATABASE_URL": "postgres://localhost/mydb",
    "NODE_ENV": "development"
  }
}
```

These are available in:
- Shell hooks' `command` execution environment
- MCP server processes (`env` in server config)
- Bash tool command execution

**Note**: CLI-level env vars (`CLAUDE_*`) are different from settings-level `env`. The former configure Claude Code itself; the latter are injected into tool environments.

---

## Important CLI Environment Variables

These control Claude Code behavior at the process level:

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_DISABLE_POLICY_SKILLS=1` | Skip loading managed/enterprise skills |
| `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=1` | Disable built-in agents (SDK mode) |
| `CLAUDE_CODE_SIMPLE=1` | Simple mode: skip custom agents, only built-ins |
| `CLAUDE_CODE_COORDINATOR_MODE=1` | Enable coordinator mode for agent swarms |
| `CLAUDE_CODE_ENTRYPOINT=sdk-ts` | Declare SDK entrypoint type (affects agent loading) |
| `CLAUDE_MCP_DEBUG=1` | Enable MCP debug logging |
| `ANTHROPIC_MODEL` | Override default model |
| `ANTHROPIC_BASE_URL` | Override API endpoint |
| `ANTHROPIC_API_KEY` | API key |

---

## Settings Source Inspection

```bash
# See which source each setting comes from
claude doctor

# Debug setting loading
claude --debug 2>&1 | grep -i settings

# Check effective permissions
# (in session)
> What permissions do I currently have set?
```

---

## CLAUDE.md (Instructions File)

CLAUDE.md files are not settings — they're natural language instructions injected into Claude's system prompt. They support:

```markdown
# Project Instructions

When working in this repository, always:
- Run tests after changes
- Follow our commit message format

<!-- Conditional rules using paths -->
<user_info>
paths: src/**.ts
</user_info>
TypeScript files: use strict mode, no any casts.

<user_info>
paths: **.py
</user_info>
Python files: follow PEP 8, use type hints.
```

CLAUDE.md files are loaded from:
1. `~/.claude/CLAUDE.md` (user global)
2. `.claude/CLAUDE.md` (project)
3. All parent `.claude/CLAUDE.md` up to home
4. Additional dirs (`--add-dir`)

They combine with settings but serve a different purpose: settings configure behavior, CLAUDE.md informs the model about conventions and constraints.
