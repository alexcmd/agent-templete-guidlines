# Plugin System

## Overview

The plugin system extends Claude Code without modifying core code. Plugins can add:
- **Slash commands** (new `/my-command` entries)
- **Skills** (prompt-based commands the model executes)
- **Hooks** (lifecycle event handlers)
- **MCP servers** (new tools via Model Context Protocol)
- **LSP servers** (language server integrations)
- **Output styles** (custom response formatting)

---

## Plugin Directory Structure

```
~/.claude/plugins/
├── installed_plugins.json        Manifest of installed plugins
└── cache/
    └── my-plugin@1.0.0/
        ├── manifest.json          Plugin manifest
        ├── skills/
        │   └── my-skill.md        Skill prompt files
        └── commands/
            └── my-command.md      Command prompt files
```

---

## Plugin Manifest Format

```json
{
  "name": "my-plugin",
  "version": "1.2.0",
  "description": "Adds deployment capabilities to Claude Code",
  "author": {
    "name": "Jane Smith",
    "email": "jane@example.com",
    "url": "https://example.com"
  },
  
  "skills": {
    "deploy": {
      "description": "Deploy the application",
      "argumentHint": "[environment]",
      "promptFile": "./skills/deploy.md"
    },
    "rollback": {
      "description": "Roll back the last deployment",
      "promptFile": "./skills/rollback.md"
    }
  },
  
  "commands": [
    {
      "name": "status",
      "type": "local",
      "description": "Check deployment status",
      "entrypoint": "./commands/status.js"
    }
  ],
  
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ./hooks/post-session.js"
          }
        ]
      }
    ]
  },
  
  "mcpServers": {
    "my-mcp": {
      "type": "stdio",
      "command": "node",
      "args": ["./mcp/server.js"],
      "env": {}
    }
  },
  
  "outputStyles": {
    "compact": {
      "name": "compact",
      "prompt": "Be very concise. Use bullet points. No preamble."
    }
  }
}
```

---

## Plugin Lifecycle

### Installation

```bash
# From npm
claude plugins install @company/claude-deploy-plugin

# From local path (development)
claude plugins install /path/to/my-plugin

# From git URL
claude plugins install https://github.com/example/claude-plugin
```

Installation process:
1. Download/copy plugin files to `~/.claude/plugins/cache/<name>@<version>/`
2. Validate manifest schema
3. Register in `~/.claude/plugins/installed_plugins.json`
4. Optionally enable immediately

### Enabling/Disabling

```bash
claude plugins enable my-plugin
claude plugins disable my-plugin
```

Stored in settings:
```json
{
  "enabledPlugins": {
    "my-plugin": true,
    "other-plugin": false
  }
}
```

### Loading (`src/utils/plugins/pluginLoader.ts`)

At startup, enabled plugins are loaded:

```typescript
async function loadAllPlugins(
  settings: SettingsJson,
): Promise<LoadedPlugin[]> {
  const installedPlugins = await getInstalledPlugins()
  const enabledPlugins = installedPlugins.filter(
    p => settings.enabledPlugins?.[p.name] !== false
  )
  
  const loaded: LoadedPlugin[] = []
  for (const plugin of enabledPlugins) {
    try {
      const manifest = await loadManifest(plugin.cachePath)
      const commands = await buildPluginCommands(manifest)
      const hooks = buildPluginHooks(manifest)
      const mcpConfigs = buildMCPConfigs(manifest)
      
      loaded.push({ manifest, commands, hooks, mcpConfigs, status: 'loaded' })
    } catch (error) {
      loaded.push({ manifest: plugin, status: 'error', error })
    }
  }
  
  return loaded
}
```

### Uninstallation

```bash
claude plugins uninstall my-plugin
```

1. Disable plugin (update settings)
2. Remove from installed_plugins.json
3. Delete cache directory

---

## Built-in Plugins (`src/plugins/builtinPlugins.ts`)

Some plugins ship with Claude Code but are optionally enabled:

| Plugin | Purpose | Default |
|--------|---------|---------|
| `chrome` | Browser integration via Chrome DevTools | disabled |
| `code-index` | Semantic code indexing | disabled |
| `github` | GitHub API integration | disabled |
| `linear` | Linear issue tracking | disabled |

---

## Plugin Command Implementation

### Prompt-based Skill (most common)

```markdown
<!-- skills/deploy.md -->
---
description: Deploy the application to an environment
argument-hint: "[staging|production]"
---

# Deploy Command

Deploy the application to the specified environment.

**Environment**: $ARGUMENTS (defaults to "staging" if not specified)

Steps to follow:
1. First verify the current git status with `git status`
2. Run the test suite: `npm test`
3. Build: `npm run build`
4. Deploy using the deployment script: `./scripts/deploy.sh $ARGUMENTS`
5. Verify the deployment by checking health endpoint
6. Report the deployment URL and any warnings
```

### Local JS Command (for programmatic operations)

```typescript
// commands/status.js
export default async function handler(args, context) {
  const { exec } = await import('child_process')
  const { promisify } = await import('util')
  const execAsync = promisify(exec)
  
  const { stdout } = await execAsync('kubectl get pods')
  
  return {
    type: 'text',
    content: `Current pod status:\n${stdout}`,
  }
}
```

---

## MCP Server Integration

MCP (Model Context Protocol) is the primary extension mechanism for external tools.

### MCP Server in Plugin

```json
// In plugin manifest
{
  "mcpServers": {
    "database": {
      "type": "stdio",
      "command": "node",
      "args": ["./mcp/database-server.js"],
      "env": {
        "DB_URL": "${DATABASE_URL}"
      }
    }
  }
}
```

### What MCP Servers Can Provide

1. **Tools** — Functions Claude can call (appear as `mcp__<server>__<tool>`)
2. **Resources** — Data sources Claude can read
3. **Prompts** — Pre-built prompt templates
4. **Commands** — Slash commands

### MCP Tool Implementation (Node.js)

```typescript
// mcp/database-server.js
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'

const server = new McpServer({
  name: 'database',
  version: '1.0.0',
})

server.tool(
  'query',
  'Execute a SQL query against the database',
  {
    sql: { type: 'string', description: 'SQL query to execute' },
    params: { type: 'array', items: { type: 'string' }, optional: true },
  },
  async ({ sql, params }) => {
    const result = await db.query(sql, params)
    return {
      content: [{ type: 'text', text: JSON.stringify(result.rows, null, 2) }],
    }
  }
)

const transport = new StdioServerTransport()
await server.connect(transport)
```

---

## Plugin Security Model

Plugins run with the same permissions as Claude Code itself. There is no sandboxing between plugins and the main process.

Security considerations:
- Only install plugins from trusted sources
- Review plugin manifest before enabling
- MCP servers run as separate processes (partial isolation)
- Hooks run as shell commands (full system access)

---

## Plugin Error Handling

When a plugin fails to load, it's recorded in `AppState.plugins.errors`:

```typescript
type PluginError = {
  pluginName: string
  error: string
  phase: 'manifest' | 'load' | 'command' | 'hook' | 'mcp'
}
```

The user sees a warning in the REPL footer, and the plugin is skipped. Other plugins continue to load.

---

## Output Styles

Plugins can define custom output styles that change how Claude formats responses:

```json
{
  "outputStyles": {
    "json-only": {
      "name": "json-only",
      "prompt": "Always respond with valid JSON only. No prose, no markdown. The JSON should be compact (no extra whitespace). For errors, return {\"error\": \"message\"}."
    },
    "verbose-teaching": {
      "name": "verbose-teaching",
      "prompt": "Explain everything thoroughly. Define technical terms. Include 'Why this matters' sections. Assume the reader is learning."
    }
  }
}
```

Users select an output style with:
```bash
claude config set outputStyle json-only
```
