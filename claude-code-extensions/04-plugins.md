# Plugin System

Plugins are bundles of skills, hooks, MCP servers, LSP servers, and agents that can be installed, enabled/disabled, and shared via a marketplace. They appear in the `/plugin` UI.

---

## Plugin Types

| Type | ID Format | Source | Toggleable in UI |
|------|-----------|--------|-----------------|
| Built-in | `name@builtin` | Ships with CLI | Yes |
| Marketplace | `name@marketplace-name` | External registry | Yes |
| Managed | — | Enterprise policy | No (admin-controlled) |

---

## Plugin Manifest

Every plugin has a manifest describing its metadata and components:

```json
{
  "name": "my-plugin",
  "description": "What this plugin does",
  "version": "1.2.3",
  "skills": [...],
  "hooks": { ... },
  "mcpServers": { ... },
  "lspServers": { ... },
  "agents": [...]
}
```

The manifest is validated at load time. Missing required fields cause the plugin to be skipped with a logged error.

---

## Plugin Components

A plugin can provide any combination of:

### Skills
```json
{
  "skills": [
    {
      "name": "my-plugin-skill",
      "description": "What the skill does",
      "prompt": "The skill prompt content...",
      "userInvocable": true,
      "allowedTools": ["Bash", "Read"],
      "argumentHint": "<arg>",
      "whenToUse": "Use when...",
      "model": "claude-haiku-4-5-20251001",
      "context": "fork",
      "agent": "general-purpose",
      "hooks": { ... }
    }
  ]
}
```

### Hooks
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/plugins/my-plugin/hooks/pre-bash.sh"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/plugins/my-plugin/hooks/session-start.sh"
          }
        ]
      }
    ]
  }
}
```

### MCP Servers
```json
{
  "mcpServers": {
    "my-mcp": {
      "type": "stdio",
      "command": "node",
      "args": ["~/.claude/plugins/my-plugin/server/index.js"],
      "env": {
        "API_KEY": "${MY_PLUGIN_API_KEY}"
      }
    }
  }
}
```

### LSP Servers
```json
{
  "lspServers": {
    "my-lsp": {
      "command": "my-language-server",
      "args": ["--stdio"],
      "filetypes": ["mylang"]
    }
  }
}
```

### Agents
```json
{
  "agents": [
    {
      "name": "my-plugin-agent",
      "description": "When to use this agent",
      "prompt": "System prompt...",
      "tools": ["Bash", "Read"],
      "model": "claude-haiku-4-5-20251001"
    }
  ]
}
```

---

## Plugin Directory Structure

Installed marketplace plugins live in `~/.claude/plugins/`:

```
~/.claude/plugins/
  installed_plugins.json          # Registry of installed plugins + metadata
  known_marketplaces.json         # Known marketplace URLs
  install-counts-cache.json       # Download stats cache
  cache/                          # Cached manifests
  data/                           # Plugin data directories
    my-plugin@marketplace/
      manifest.json               # Plugin manifest
      skills/                     # Skill .md files
        my-skill.md
      agents/                     # Agent .md files
        my-agent.md
      hooks/                      # Hook scripts
        pre-bash.sh
      server/                     # MCP server files
        index.js
  marketplaces/
    my-marketplace/               # Marketplace index
```

---

## Built-in Plugin API

Built-in plugins are registered programmatically in the CLI source. While you can't create new built-in plugins externally (requires source changes), understanding the API helps when building similar patterns:

```typescript
import { registerBuiltinPlugin } from './plugins/builtinPlugins.js'

registerBuiltinPlugin({
  name: 'my-builtin',
  description: 'A built-in plugin that ships with the CLI',
  version: '1.0.0',
  defaultEnabled: true,          // Enabled by default; user can disable
  isAvailable: () => {           // Optional: conditional availability
    return process.platform !== 'win32'
  },
  skills: [
    {
      name: 'my-builtin-skill',
      description: 'Does something useful',
      getPromptForCommand: async (args) => [
        { type: 'text', text: `Skill content with args: ${args}` }
      ]
    }
  ],
  hooks: {
    SessionStart: [
      {
        hooks: [
          {
            type: 'command',
            command: 'echo "Session started from built-in plugin"'
          }
        ]
      }
    ]
  },
  mcpServers: {
    'my-server': {
      type: 'stdio',
      command: 'node',
      args: ['/path/to/server.js']
    }
  }
})
```

Built-in plugin IDs use the format `name@builtin`. User preference is persisted to `settings.json` under `enabledPlugins`:
```json
{
  "enabledPlugins": {
    "my-builtin@builtin": true,
    "other-plugin@builtin": false
  }
}
```

---

## Plugin Load Lifecycle

```
Startup
  └── loadAllPlugins()
        ├── Load installed_plugins.json
        ├── For each plugin entry:
        │     ├── Read manifest.json from data/
        │     ├── Validate manifest
        │     ├── Check enabled state (settings.json > manifest.defaultEnabled > true)
        │     └── If enabled:
        │           ├── Load skills from skillsPaths
        │           ├── Load agents from agentsPaths
        │           ├── Register hooks (hooksConfig)
        │           └── Initialize MCP servers (mcpServers)
        └── Return LoadedPlugin[] split into enabled/disabled
```

---

## LoadedPlugin Structure

The runtime representation of an installed plugin:

```typescript
type LoadedPlugin = {
  name: string
  manifest: PluginManifest
  path: string                     // Filesystem path (sentinel 'builtin' for built-ins)
  source: string                   // e.g., "my-plugin@my-marketplace"
  repository: string               // Registry URL or 'builtin'
  enabled?: boolean
  isBuiltin?: boolean
  commandsPaths?: string[]         // Paths to skill markdown directories
  skillsPaths?: string[]
  agentsPaths?: string[]
  hooksConfig?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  lspServers?: Record<string, LspServerConfig>
}
```

---

## Plugin Commands / Skills Loading

Plugin skills can come from two sources:

1. **Inline skill definitions** in the manifest (for programmatic/bundled skills)
2. **Markdown files** in the plugin's `skills/` directory

For markdown-based skills, the plugin sets `skillsPaths` pointing to its `skills/` directory. The `loadSkillsDir` function loads `.md` files from those paths with `loadedFrom: 'plugin'` and `source: 'plugin'`.

---

## Plugin Installation Flow

```
/install <plugin-id-or-url>
  └── Resolve plugin from marketplace
  └── Fetch manifest.json
  └── Validate manifest
  └── Check for policy blocks
  └── Write to ~/.claude/plugins/installed_plugins.json
  └── Download plugin files to ~/.claude/plugins/data/<plugin-id>/
  └── Reload plugins (/reload-plugins or next session)
```

---

## Plugin Enable/Disable

Via `/plugin` UI:
- User toggles enable/disable
- Persisted to `settings.json` → `enabledPlugins["plugin-id"] = true/false`
- Takes effect after `/reload-plugins` or next session startup

Via settings.json directly:
```json
{
  "enabledPlugins": {
    "my-plugin@my-marketplace": false,
    "other-plugin@builtin": true
  }
}
```

---

## Policy Plugin Management

Enterprise administrators can deploy plugins via managed settings. Managed plugins:
- Cannot be disabled by users (no toggle in UI)
- Can block marketplace plugin installation
- Deployed via `policySettings` source

Policy settings file location (platform-dependent):
```
# macOS
/Library/Application Support/Claude/managed-settings.json

# Linux  
/etc/claude/managed-settings.json
```

---

## Error Types in Plugin Loading

The validator reports 22 distinct error types. Common ones:

| Error | Meaning |
|-------|---------|
| `MISSING_MANIFEST` | manifest.json not found in plugin data dir |
| `INVALID_MANIFEST` | JSON parse failed or schema validation failed |
| `MISSING_REQUIRED_FIELD` | Required field (name, description) missing |
| `INVALID_HOOK_CONFIG` | hooks field doesn't match HooksSchema |
| `INVALID_MCP_CONFIG` | mcpServers entry doesn't match McpServerConfigSchema |
| `POLICY_BLOCKED` | Plugin blocked by enterprise policy |
| `DUPLICATE_SKILL` | Two plugins provide skills with the same name |
| `VERSION_MISMATCH` | Plugin requires CLI version not satisfied |

---

## Plugin vs Settings-Level Configuration

Some capabilities can be configured at both plugin level and settings level:

| Capability | In Plugin Manifest | In settings.json |
|-----------|-------------------|-----------------|
| Hooks | `hooks: {...}` | `hooks: {...}` |
| MCP servers | `mcpServers: {...}` | `mcpServers: {...}` |
| Skills | `skills: [...]` | N/A (use skills/ dir) |
| Agents | `agents: [...]` | N/A (use agents/ dir) |
| Enable/disable | Auto via marketplace | `enabledPlugins: {...}` |

When the same hook event has both plugin hooks and settings hooks, all fire (settings hooks first, then plugin hooks).

---

## Creating a Minimal Distributable Plugin

A minimal filesystem plugin that can be installed:

```
my-plugin/
├── manifest.json
├── skills/
│   └── my-skill.md
├── agents/
│   └── my-agent.md
└── hooks/
    └── on-session-start.sh
```

**manifest.json:**
```json
{
  "name": "my-plugin",
  "description": "My useful plugin for Claude Code",
  "version": "0.1.0",
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/absolute/path/to/my-plugin/hooks/on-session-start.sh"
          }
        ]
      }
    ]
  }
}
```

Note: paths in hook commands must be absolute (plugin data dir is not in PATH).
