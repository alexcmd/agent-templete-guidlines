# Commands & Skills

## Overview

Claude Code has two distinct command types:
1. **CLI Commands** — invoked at the terminal, before/after conversation
2. **Slash Commands** — invoked mid-conversation with `/command-name`

Slash commands split into three subtypes:
- **Local commands** (run in JS process — auth, plugins, settings)
- **Local JSX commands** (render a React component in the terminal)
- **Prompt commands / Skills** (expand into a prompt the model executes)

---

## Command Registry (`src/commands.ts`)

```typescript
export function getCommands(options: GetCommandsOptions): Command[] {
  return [
    ...getBuiltinCommands(),      // Hardcoded CLI commands
    ...getBundledSkillCommands(),  // Built-in skills from src/skills/bundled/
    ...getPluginCommands(),        // Commands from installed plugins
    ...getMCPCommands(),           // Commands from MCP servers
    ...getUserDefinedCommands(),   // Commands from .claude/commands/
  ]
}
```

---

## Built-in CLI Commands

Located in `src/commands/` — each is a `LocalCommand`, `LocalJSXCommand`, or `PromptCommand`.

Commands marked **Immediate** execute _during_ an active query (not queued). This is driven by `matchingCommand.immediate === true` in `REPL.tsx:3161`.

### Core Commands

| Command | Type | Immediate | Description |
|---------|------|-----------|-------------|
| `/add-dir` | prompt | | Add directory to CLAUDE.md search path |
| `/btw` | local-jsx | ✓ | Quick note injected into active query |
| `/clear` | local-jsx | ✓ | Clear conversation history and free context |
| `/color` | local-jsx | ✓ | Change agent color |
| `/commit` | local | | Create git commit |
| `/commit-push-pr` | local | | Commit + push + create PR |
| `/compact` | local | | Compact/summarize conversation (optional instructions) |
| `/config` | local-jsx | | Edit settings UI |
| `/context` | local | | Show/analyze context window |
| `/cost` | local | | Show session cost tracking |
| `/diff` | local | | Show git diff |
| `/doctor` | local | | Run diagnostic checks |
| `/effort` | local | | Set effort level |
| `/exit` | local-jsx | ✓ | Exit TUI |
| `/export` | local | | Export conversation |
| `/fast` | local | | Toggle fast mode |
| `/feedback` | local-jsx | ✓ | Send feedback |
| `/files` | local | | List tracked files |
| `/help` | local-jsx | ✓ | Show help and command list |
| `/hooks` | local | | Manage hooks |
| `/ide` | local | | IDE integration |
| `/init` | local | | Initialize CLAUDE.md |
| `/install-github-app` | local | | GitHub App setup |
| `/install-slack-app` | local | | Slack app setup |
| `/keybindings` | local-jsx | ✓ | Edit keybindings |
| `/login` | local | | Authenticate (OAuth) |
| `/logout` | local | | Sign out |
| `/mcp` | local | | MCP server management |
| `/memory` | local | | Session memory access |
| `/mobile` | local-jsx | ✓ | Show mobile QR code |
| `/model` | local-jsx | ✓ | Select model |
| `/output-style` | local | | Output formatting |
| `/permissions` | local | | Manage tool permissions |
| `/plan` | local-jsx | ✓ | Toggle plan mode |
| `/plugin` | local | | Plugin management |
| `/pr-comments` | local | | PR comment handling |
| `/privacy-settings` | local | | Privacy configuration |
| `/rate-limit-options` | local | | Rate limit configuration |
| `/release-notes` | local | | Show changelog |
| `/rename` | local | | Rename files |
| `/resume` | local-jsx | ✓ | Resume previous session |
| `/review` | local | | Code review |
| `/rewind` | local | | Undo/rewind operations |
| `/sandbox-toggle` | local | | Toggle sandbox mode |
| `/security-review` | local | | Security analysis |
| `/session` | local-jsx | ✓ | Show remote session QR/URL |
| `/share` | local | | Share conversation |
| `/skills` | local | | List available skills |
| `/stats` | local | | Show statistics |
| `/status` | local | | Show status |
| `/statusline` | local | | Status line configuration |
| `/stickers` | local | | Sticker management |
| `/summary` | local | | Summarize conversation |
| `/tag` | local | | Tag management |
| `/terminalSetup` | local | | Terminal configuration |
| `/theme` | local-jsx | ✓ | Change terminal theme |
| `/upgrade` | local | | Upgrade CLI |
| `/usage` | local | | Show usage info |
| `/vim` | local-jsx | ✓ | Toggle vim mode |

### Feature-Gated Commands

| Command | Feature Flag | Description |
|---------|-------------|-------------|
| `/brief` | `KAIROS` | — |
| `/bridge` | `BRIDGE_MODE` | Remote control bridge |
| `/buddy` | `BUDDY` | — |
| `/bughunter` | — | Bug detection |
| `/fork` | `FORK_SUBAGENT` | Fork as subagent |
| `/peers` | `UDS_INBOX` | — |
| `/proactive` | `PROACTIVE`/`KAIROS` | — |
| `/remote-setup` | `CCR_REMOTE_SETUP` | — |
| `/subscribe-pr` | `KAIROS_GITHUB_WEBHOOKS` | — |
| `/torch` | `TORCH` | — |
| `/ultraplan` | `ULTRAPLAN` | — |
| `/workflows` | `WORKFLOW_SCRIPTS` | Workflow scripts |

### ANT-Internal Commands

Commands only available to Anthropic employees:
`/ant-trace`, `/autofix-pr`, `/backfill-sessions`, `/break-cache`, `/env`, `/good-claude`, `/issue`, `/mock-limits`, `/oauth-refresh`, `/onboarding`, `/perf-issue`, `/reset-limits`, `/teleport`, `/version`

---

### Remote-Safe Commands

Commands that work in `--remote` mode:
`/session`, `/exit`, `/clear`, `/help`, `/theme`, `/color`, `/vim`, `/cost`, `/usage`, `/btw`, `/feedback`, `/plan`, `/keybindings`, `/statusline`, `/stickers`, `/mobile`

### Bridge-Safe Commands

Commands that work via the Remote Control bridge (from mobile/web):
`/compact`, `/clear`, `/cost`, `/summary`, `/files`

---

## Built-in Skills (`src/skills/bundled/`)

Skills are `PromptCommand` entries — the model executes them by receiving a detailed prompt.

### Registered Skills

| Skill | Trigger | Description |
|-------|---------|-------------|
| `update-config` | `/update-config` | Configure Claude Code settings via settings.json |
| `keybindings-help` | `/keybindings` | Configure keyboard shortcuts |
| `simplify` | `/simplify` | Review and simplify recently changed code |
| `less-permission-prompts` | `/less-permission-prompts` | Scan transcripts, add allowlist rules |
| `loop` | `/loop` | Run a prompt/command on a recurring interval |
| `schedule` | `/schedule` | Create/manage scheduled remote agent triggers |
| `init` | `/init` | Initialize CLAUDE.md documentation |
| `review` | `/review` | Review a pull request |
| `security-review` | `/security-review` | Security review of branch changes |
| `claude-api` | `/claude-api` | Build/debug Claude API applications |
| `statusline-setup` | `/statusline-setup` | Configure status line |

### Skill File Format

Skills are markdown files with optional YAML frontmatter:

```markdown
---
description: One-line description for the command registry
argument-hint: "[optional] [args] [description]"
---

# Skill Name

Full prompt text that the model receives when this skill is invoked.

The prompt can reference:
- $ARGUMENTS — what the user typed after /command-name
- Current working directory via context
- Any tool the model normally has access to
```

### How Skills Execute

1. User types `/skill-name optional args`
2. `getSlashCommandToolSkills(commands)` finds the skill
3. `SkillTool` handler calls `command.getPromptForCommand(args, context)`
4. Returns `ContentBlockParam[]` (the skill's prompt content)
5. Model receives this as a tool result and executes the skill instructions

---

## User-Defined Commands (`.claude/commands/`)

Users can create custom commands by placing markdown files in:
- `~/.claude/commands/` — global commands
- `.claude/commands/` — project-specific commands

```
.claude/
└── commands/
    ├── deploy.md          → /deploy
    ├── format.md          → /format
    └── release/
        └── create.md      → /release:create (or /release/create)
```

### Command File Format

```markdown
---
description: Deploy the application to staging
argument-hint: "[environment]"
---

Deploy the application using the following steps:

1. Run `npm run build`
2. Run `npm test`  
3. Deploy to $ARGUMENTS environment using the deployment script
4. Verify the deployment by checking the health endpoint

If any step fails, stop and report the error.
```

---

## MCP Commands

MCP servers can expose commands (slash commands) alongside tools:

```json
// In MCP server response to 'tools/list':
{
  "tools": [...],
  "commands": [
    {
      "name": "deploy",
      "description": "Deploy using server's capabilities",
      "prompt": "Deploy the application..."
    }
  ]
}
```

---

## Plugin Commands

Plugins can register both tools and commands via their manifest:

```json
// plugin-manifest.json
{
  "name": "my-plugin",
  "commands": [
    {
      "name": "my-command",
      "description": "Does something useful",
      "skillPath": "./skills/my-command.md"
    }
  ]
}
```

---

## Command Execution Flow

### Local Command Flow
```
User types: /clear
       │
       ▼
commands.ts → find command by name
       │
       ▼
command.load() → import command module
       │
       ▼
module.default(args, context) → execute
       │
       ▼
Returns: string | React.Element | undefined
       │
       ▼
REPL.tsx renders result (if any)
```

### Prompt Command (Skill) Flow
```
User types: /review
       │
       ▼
commands.ts → find command (type: 'prompt')
       │
       ▼
SkillTool.handler() is called by model
       │
       ▼
command.getPromptForCommand("", context)
  → returns ContentBlockParam[] (the skill prompt)
       │
       ▼
Tool result injected into conversation
       │
       ▼
Model receives skill instructions, executes them
       │
       ▼
Model can use any available tool (Read, Bash, etc.)
       │
       ▼
Model returns final response to user
```

---

## Skill Tool (`src/tools/SkillTool/`)

The bridge between slash commands and model execution.

```typescript
// SkillTool handler
async function handler(input, context) {
  const { command_name, args } = input
  
  // Find matching command
  const command = context.options.commands.find(
    c => c.name === command_name && c.type === 'prompt'
  )
  
  if (!command) {
    return `Unknown skill: ${command_name}`
  }
  
  // Get the skill's prompt content
  const promptContent = await command.getPromptForCommand(args, context)
  
  // The model will receive this as a tool result
  // and execute the skill instructions
  return formatContentForToolResult(promptContent)
}
```

---

## Command Priority & Resolution

When multiple sources define the same command name:

```
Priority (highest → lowest):
1. User-typed /command (explicit invocation)
2. Local .claude/commands/ (project)
3. ~/.claude/commands/ (global user)
4. Plugin commands
5. Bundled skills
6. MCP commands
7. Built-in CLI commands
```

---

## Context Available to Prompt Commands

When a skill's `getPromptForCommand()` is called, it receives:

```typescript
type SkillContext = {
  // Full tool use context
  options: {
    commands: Command[]
    tools: Tools
    mainLoopModel: string
    mcpClients: MCPServerConnection[]
  }
  getAppState(): AppState
  // Current working directory, git state, settings, etc.
}
```

The skill prompt can reference:
- `$ARGUMENTS` placeholder (replaced with user's args)
- Current files and project structure (via tools the model will use)
- Model context (prior conversation is included)
