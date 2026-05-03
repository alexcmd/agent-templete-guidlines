# Custom Agents

Agents are isolated sub-contexts spawned by the model via `AgentTool`. Each agent gets its own system prompt, tool set, model, permission mode, and optionally its own memory. The model decides when to spawn an agent based on the `description` (whenToUse) field.

---

## Agent vs Skill

| | Skill | Agent |
|--|-------|-------|
| **Invoked by** | User (`/name`) or model (SkillTool) | Model only (AgentTool) |
| **Context** | Same conversation (inline) or forked | Always isolated sub-context |
| **System prompt** | Parent's system prompt | Agent's own system prompt |
| **Tool set** | Parent's tools (filtered by `allowed-tools`) | Custom tool set |
| **Memory** | None | Optional persistent memory |
| **Good for** | Prompt templates, reusable instructions | Specialized autonomous workers |

---

## Directory Structure

```
~/.claude/agents/           # User scope (all projects)
  my-agent.md

.claude/agents/             # Project scope (committed)
  my-agent.md
  my-agent.json             # JSON format also supported
```

Managed agents live at:
```
/path/to/managed/.claude/agents/
```

---

## Markdown Agent Format

The markdown file body becomes the agent's system prompt. Frontmatter controls all configuration.

```markdown
---
# REQUIRED
name: my-agent
description: "When to use this agent: one or more sentences that guide the model to pick this agent for appropriate tasks."

# Tool access
tools: Bash, Read, Write, Edit, Glob, Grep
disallowedTools: WebFetch, WebSearch

# Skills to preload (comma-separated skill names)
skills: simplify, my-skill

# Model and compute
model: claude-sonnet-4-6    # or 'inherit' (uses session model), or specific model ID
effort: normal              # 'quick' | 'normal' | 'thorough' | integer (thinking tokens)

# Permission control
permissionMode: default     # 'default' | 'dontAsk' | 'plan' | 'bypassPermissions' | 'auto'
maxTurns: 20                # Max agentic turns before stopping (default: no limit)

# Display
color: blue                 # Agent color in UI: red/orange/yellow/green/blue/purple/pink/cyan/magenta/teal/lime/amber/violet/rose/sky/indigo/emerald/fuchsia/white/gray

# MCP servers this agent needs
mcpServers:
  - slack                   # Reference to an existing configured server by name
  - github                  # Another existing server
  - my-server:              # Inline server definition
      type: stdio
      command: npx
      args: ["-y", "my-mcp-server"]
      env:
        API_KEY: "${MY_API_KEY}"

# Required MCP servers (agent hidden if unavailable)
# requiredMcpServers: [slack, github]

# Execution control
background: false           # true = always spawn as background task
memory: project             # 'user' | 'project' | 'local' — persistent agent memory
isolation: worktree         # 'worktree' = isolated git worktree clone

# Lifecycle hooks (active only while this agent runs)
hooks:
  PostToolUse:
    - matcher: "Write"
      hooks:
        - type: command
          command: "echo 'Agent wrote a file'"

# Initial prompt (prepended to first user turn, slash commands work)
initialPrompt: "/my-setup-skill"
---

You are a specialized agent for [purpose].

## Your capabilities
- [Capability 1]
- [Capability 2]

## How you work
[System prompt body — instructions, guidelines, persona, output format, etc.]

## Important constraints
- [Constraint 1]
- [Constraint 2]
```

---

## JSON Agent Format

JSON format supports all the same fields as markdown, but without inline shell comments. The body is the `prompt` field:

```json
{
  "description": "When to use this agent",
  "prompt": "You are a specialized agent for...\n\nYour system prompt here.",
  "tools": ["Bash", "Read", "Write"],
  "disallowedTools": ["WebFetch"],
  "model": "claude-haiku-4-5-20251001",
  "effort": "quick",
  "permissionMode": "dontAsk",
  "maxTurns": 10,
  "mcpServers": ["slack"],
  "skills": ["simplify"],
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "echo done" }]
      }
    ]
  },
  "background": false,
  "memory": "local",
  "isolation": "worktree",
  "initialPrompt": "/setup"
}
```

JSON agents can also be defined as a record (multiple agents in one file), though this is primarily used internally:
```json
{
  "agent-one": { "description": "...", "prompt": "..." },
  "agent-two": { "description": "...", "prompt": "..." }
}
```

---

## Complete Frontmatter Reference

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Agent type identifier. Used in `AgentTool` `subagent_type` parameter. Must be unique. |
| `description` | string | **Critical**: This is `whenToUse` — the model reads this to decide when to spawn your agent. Write it as guidance: "Use this agent when..." |

### Tool Access

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `tools` | list / `"*"` | `"*"` (all tools) | Tool allowlist. Comma-separated string or YAML list. `"*"` = all tools. |
| `disallowedTools` | list | `[]` | Tools explicitly blocked (applied on top of `tools` allowlist). |

Tool names are the same as shown in permission prompts: `Bash`, `Read`, `Write`, `Edit`, `Glob`, `Grep`, `WebFetch`, `WebSearch`, `Agent`, `Skill`, `NotebookEdit`, `TaskCreate`, `TaskUpdate`, `TaskList`, `TaskStop`, `TaskOutput`, `TaskGet`, `Monitor`, `CronCreate`, `CronDelete`, `CronList`, `EnterPlanMode`, `ExitPlanMode`, `EnterWorktree`, `ExitWorktree`, `RemoteTrigger`, `PushNotification`, `AskUserQuestion`, `ScheduleWakeup`, and MCP tool names (`mcp__server-name__tool-name`).

### Model and Compute

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `model` | string | session model | Model ID or `"inherit"`. Example: `claude-haiku-4-5-20251001` |
| `effort` | string / int | — | Thinking budget. Levels: `quick`, `normal`, `thorough`. Or integer = token count. |

### Permission Control

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `permissionMode` | string | `"default"` | Permission mode for this agent. See modes below. |
| `maxTurns` | int | unlimited | Max agentic turns. Agent stops gracefully when reached. |

Permission modes:
- `default` — Normal permission prompts
- `dontAsk` — Never prompt; allow everything allowed by settings
- `plan` — Planning mode: Read/Glob/Grep only, no writes
- `bypassPermissions` — Bypass all permission checks (requires user opt-in)
- `auto` — Uses Bash security classifier; auto-allows safe commands

### MCP Servers

| Field | Type | Description |
|-------|------|-------------|
| `mcpServers` | list | MCP servers for this agent. Can be server names (strings) or inline `{name: config}` objects. |
| `requiredMcpServers` | list | Pattern list. Agent is hidden from model if none of the configured servers match. |

MCP server reference (use existing server by name):
```yaml
mcpServers:
  - slack
  - github
```

MCP server inline definition:
```yaml
mcpServers:
  - my-server:
      type: stdio
      command: node
      args: ["/path/to/server.js"]
      env:
        TOKEN: "${TOKEN}"
```

### Advanced Execution

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `background` | bool | `false` | If true, always spawn as a background Task (non-blocking to parent). |
| `memory` | string | — | `user`, `project`, or `local`. Enables persistent memory in `~/.claude/memory/<name>/`. |
| `isolation` | string | — | `worktree` = run in a temporary isolated git worktree clone. Worktree is cleaned if no changes made. |
| `initialPrompt` | string | — | Prepended to first user turn in agent. Slash commands work here (e.g., `/setup-skill`). |
| `color` | string | auto-assigned | UI color for agent display in swarm views. |
| `hooks` | object | — | HooksSettings active only during this agent's session. |
| `skills` | list | — | Skill names to preload for this agent. |

---

## Built-In Agents

Claude Code ships these built-in agents:

| Agent Type | `subagent_type` | Description |
|-----------|----------------|-------------|
| General Purpose | `general-purpose` | Multi-step research and execution tasks |
| Explore | `Explore` | Fast read-only search (find/grep; reads excerpts, not full files) |
| Plan | `Plan` | Software architect for implementation plans |
| Claude Code Guide | `claude-code-guide` | Answers questions about Claude Code itself |
| Statusline Setup | `statusline-setup` | Configures Claude Code status line setting |
| Verification | `verificationAgent` | Feature-flagged verifier agent |

Disable all built-in agents for SDK use:
```bash
CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=1 claude
```

---

## Agent Loading Priority

Like skills, later sources override earlier ones for the same `name`:
1. Built-in agents (lowest priority)
2. Plugin agents
3. User agents (`~/.claude/agents/`)
4. Project agents (`.claude/agents/`, scanned up to `$HOME`)
5. Flag agents (highest priority)
6. Managed agents (enterprise, overrides all)

Within project agents, the most-specific path (deepest from cwd up) wins.

---

## How the Model Selects an Agent

The model sees all agent `description` strings when deciding whether to call `AgentTool`. Write descriptions that clearly distinguish when to use each agent:

```markdown
---
description: |
  Use this agent when the user asks to review database queries or optimize SQL.
  Especially effective for PostgreSQL-specific optimizations.
  DO NOT use for general code review — use the simplify skill instead.
---
```

The `whenToUse` is injected into the `AgentTool` description block so the model reads it during tool selection.

---

## Agent Memory

Agents with `memory` set get a persistent memory directory:

```
~/.claude/memory/
  my-agent/         # memory: 'user'
    MEMORY.md       # Agent-managed memory files
    notes.md

.claude/memory/
  my-agent/         # memory: 'project'
    MEMORY.md

# 'local' scope: ephemeral, project-local, gitignored
```

Memory is injected into the agent's system prompt on each invocation. The agent reads and writes its memory files using `Read`, `Write`, `Edit` tools (automatically added to tool allowlist when memory is enabled).

---

## Worktree Isolation

Agents with `isolation: worktree` run in a temporary git worktree clone:

```markdown
---
name: safe-refactor
description: "Performs risky refactoring in an isolated worktree"
isolation: worktree
---
```

**Behavior**:
- A fresh git worktree is created at a temporary path
- Agent works in the worktree (reads/writes affect only the clone)
- If agent makes no changes: worktree is silently cleaned up
- If agent makes changes: worktree path and branch are returned to parent for review

Source: `EnterWorktree` / `ExitWorktree` tools + worktree management in `AgentTool`.

---

## Spawning Agents Programmatically

The model calls `AgentTool` to spawn agents. You can reference custom agents in skill prompts:

```markdown
---
name: parallel-review
description: "Run multiple review agents in parallel"
---
Use the Agent tool to spawn three agents concurrently in a single message:

1. subagent_type: sql-reviewer — focus: database queries in the diff
2. subagent_type: security-reviewer — focus: security vulnerabilities  
3. subagent_type: general-purpose — focus: code quality

Provide each agent the full git diff as context.
```

Within skills, reference the tool name via the `AGENT_TOOL_NAME` constant (in source). In markdown, just say "use the Agent tool" or reference `AgentTool`.

---

## Debugging Agent Loading

```bash
# See what agents were loaded and any parse failures
claude --debug 2>&1 | grep -E "agent|Agent"

# Check for parse errors in your agent files
# Failed files are logged: "Failed to parse agent from /path/to/agent.md: error message"

# Test agent description by asking Claude to describe available agents
# (works in interactive session)
```

Common parse failures:
- Missing `name` field in frontmatter → silently skipped (assumed to be non-agent doc)
- Missing `description` field → logged as parse error
- Invalid `permissionMode` → field ignored, logged
- Invalid `isolation` value → field ignored, logged
- Invalid `mcpServers` item → item skipped, logged
