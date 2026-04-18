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

Located in `src/commands/` — each is a `LocalCommand` or `LocalJSXCommand`.

| Command | Type | Description |
|---------|------|-------------|
| `/help` | local-jsx | Show help overlay |
| `/clear` | local | Clear conversation history |
| `/compact` | local | Manually compact conversation |
| `/config` | local-jsx | Open config settings UI |
| `/cost` | local | Show session cost breakdown |
| `/doctor` | local-jsx | Diagnose configuration issues |
| `/exit` | local | Exit Claude Code |
| `/history` | local-jsx | Browse conversation history |
| `/ide` | local-jsx | IDE integration settings |
| `/init` | local | Initialize CLAUDE.md in project |
| `/install-github-app` | local | Install GitHub App integration |
| `/login` | local | Authenticate (OAuth) |
| `/logout` | local | Remove credentials |
| `/mcp` | local-jsx | MCP server management |
| `/memory` | local-jsx | Memory system management |
| `/model` | local | Show/switch model |
| `/permissions` | local-jsx | Manage permission rules |
| `/plugins` | local-jsx | Plugin management |
| `/pr_comments` | local | Fetch GitHub PR comments |
| `/reset` | local | Reset to fresh session |
| `/resume` | local-jsx | Resume prior session |
| `/review` | prompt | Review a pull request (skill) |
| `/status` | local | Show connection status |
| `/terminal-setup` | local | Configure terminal integration |
| `/vim` | local | Toggle vim keybindings |

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
