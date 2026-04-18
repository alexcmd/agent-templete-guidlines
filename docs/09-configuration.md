# Configuration System

## Settings Hierarchy

Settings are merged from multiple sources, with later sources overriding earlier ones:

```
Priority (lowest → highest):
1. Hard-coded defaults (in TypeScript code)
2. ~/.claude/settings.json       (global user settings)
3. .claude/settings.json         (project settings, committed)
4. .claude/settings.local.json   (local project settings, gitignored)
5. Sync settings (sync.claude.ai, if enabled)
6. Managed settings (remote, via organization policy)
7. MDM policy (enterprise device management)
8. Environment variables         (highest priority)
9. CLI flags (--model, --verbose, etc.)
```

---

## Settings Files

### Global User Settings: `~/.claude/settings.json`

Applies to all projects for this user.

```json
{
  "model": "claude-sonnet-4-6",
  "theme": "dark",
  "verbose": false,
  "preferredNotifChannel": "terminal-bell",
  "language": "en",
  "includeCoAuthoredBy": true,
  "hooks": {
    "Stop": [
      {
        "hooks": [{ "type": "command", "command": "terminal-notifier -message Done" }]
      }
    ]
  },
  "permissions": {
    "allow": ["Bash(git *)"],
    "deny": []
  }
}
```

### Project Settings: `.claude/settings.json`

Should be committed to version control. Shared with the whole team.

```json
{
  "model": "claude-opus-4-6",
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(npx *)",
      "Edit",
      "Write"
    ],
    "deny": ["Bash(npm publish *)"]
  },
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    }
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          { "type": "command", "command": "npx prettier --write $CLAUDE_FILE_PATH || true" }
        ]
      }
    ]
  }
}
```

### Local Settings: `.claude/settings.local.json`

Should be gitignored. User-specific overrides for the project.

```json
{
  "verbose": true,
  "permissions": {
    "allow": [
      "Bash(docker *)",
      "Bash(kubectl *)"
    ]
  }
}
```

---

## Environment Variables

| Variable | Purpose | Default |
|----------|---------|---------|
| `ANTHROPIC_API_KEY` | API key for Claude | (required without OAuth) |
| `ANTHROPIC_BASE_URL` | API base URL | `https://api.anthropic.com` |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Max tokens per response | model default |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Disable telemetry, updates | false |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | Disable background agents | false |
| `CLAUDE_CODE_SKIP_PERMISSIONS` | Skip all permission checks (dangerous) | false |
| `BASH_DEFAULT_TIMEOUT_MS` | Default bash command timeout | 120000 |
| `BASH_MAX_TIMEOUT_MS` | Maximum allowed timeout | 3600000 |
| `CLAUDE_CODE_USE_BEDROCK` | Use AWS Bedrock for API | false |
| `CLAUDE_CODE_USE_VERTEX` | Use Google Vertex AI for API | false |
| `AWS_REGION` | AWS region for Bedrock | us-east-1 |
| `CLAUDE_CODE_PROXY_URL` | HTTP proxy URL | (none) |
| `HTTPS_PROXY` / `HTTP_PROXY` | Standard proxy vars | (none) |
| `NODE_EXTRA_CA_CERTS` | Extra CA certificates | (none) |
| `CLAUDE_DEBUG` | Enable debug logging | false |
| `CLAUDE_CODE_COORDINATOR_MODE` | Enable coordinator mode | false |
| `CLAUDE_AUTO_BACKGROUND_TASKS` | Auto-background long agents | false |

---

## MCP Server Configuration

MCP servers can be configured in settings files:

```json
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"],
      "env": {}
    },
    "postgres": {
      "type": "stdio", 
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://localhost/mydb"
      }
    },
    "remote-server": {
      "type": "sse",
      "url": "https://mcp.example.com/sse",
      "headers": {
        "Authorization": "Bearer ${MY_API_TOKEN}"
      }
    }
  }
}
```

### MCP Server Types

| Type | Description | Config |
|------|-------------|--------|
| `stdio` | Local process via stdin/stdout | `command`, `args`, `env` |
| `sse` | Remote server via Server-Sent Events | `url`, `headers` |
| `http` | Remote server via HTTP | `url`, `headers` |

---

## Feature Flags (`bun:bundle` + GrowthBook)

Claude Code uses compile-time feature flags (Bun bundle) and runtime flags (GrowthBook):

### Compile-Time Flags (Dead Code Elimination)
```typescript
import { feature } from 'bun:bundle'

// These are evaluated at build time
const MonitorTool = feature('MONITOR_TOOL')
  ? require('./tools/MonitorTool/MonitorTool.js').MonitorTool
  : null

const cronTools = feature('AGENT_TRIGGERS')
  ? [CronCreateTool, CronDeleteTool, CronListTool]
  : []
```

### Runtime Flags (GrowthBook)
```typescript
import { getFeatureValue_CACHED_MAY_BE_STALE } from './services/analytics/growthbook.js'

const autoBackgroundEnabled = getFeatureValue_CACHED_MAY_BE_STALE(
  'tengu_auto_background_agents',
  false    // default value
)
```

### Key Feature Flags
| Flag | Purpose |
|------|---------|
| `COORDINATOR_MODE` | Multi-agent coordinator |
| `AGENT_TRIGGERS` | Cron/scheduled tasks |
| `REACTIVE_COMPACT` | Pattern-triggered compaction |
| `CONTEXT_COLLAPSE` | Aggressive context reduction |
| `EXPERIMENTAL_SKILL_SEARCH` | Semantic skill discovery |
| `MONITOR_TOOL` | Process monitoring tool |
| `KAIROS` | Proactive assistant features |
| `KAIROS_PUSH_NOTIFICATION` | Push notification support |
| `CACHED_MICROCOMPACT` | Cached micro-compaction |
| `PROACTIVE` | Proactive task features |

---

## Model Configuration

```typescript
type ModelSetting = 
  | string                          // Full model ID
  | 'claude-sonnet-4-6'             // Current Sonnet
  | 'claude-opus-4-6'               // Current Opus  
  | 'claude-haiku-4-5-20251001'     // Current Haiku
  | 'claude-opus-4-7'               // Next Opus (when available)

type EffortValue = 'low' | 'medium' | 'high' | 'max'
```

### Model Resolution Order
1. CLI flag `--model <model>`
2. `ANTHROPIC_MODEL` env var
3. `.claude/settings.json` model
4. `~/.claude/settings.json` model
5. Default model (currently `claude-sonnet-4-6`)

---

## API Provider Configuration

Claude Code supports multiple API providers:

### Anthropic (default)
```bash
ANTHROPIC_API_KEY=sk-ant-...
```

### AWS Bedrock
```bash
CLAUDE_CODE_USE_BEDROCK=1
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
```

### Google Vertex AI
```bash
CLAUDE_CODE_USE_VERTEX=1
GOOGLE_APPLICATION_CREDENTIALS=/path/to/creds.json
GOOGLE_CLOUD_PROJECT=my-project
ANTHROPIC_VERTEX_REGION=us-east5
```

### Custom Base URL (compatible providers)
```bash
ANTHROPIC_BASE_URL=https://my-proxy.example.com
ANTHROPIC_API_KEY=my-key
```

---

## Config Loading (`src/utils/config.ts`)

```typescript
export function getGlobalConfig(): GlobalConfig {
  // Cached singleton
  if (cachedConfig) return cachedConfig

  // Load and merge all settings files
  const globalSettings = loadSettingsFile(getGlobalSettingsPath())
  const projectSettings = loadSettingsFile(getProjectSettingsPath())
  const localSettings = loadSettingsFile(getLocalSettingsPath())
  const managedSettings = getManagedSettings()  // From remote
  const mdmPolicy = getMDMPolicy()              // From OS MDM

  cachedConfig = mergeSettings([
    globalSettings,
    projectSettings,
    localSettings,
    managedSettings,
    mdmPolicy,
  ])

  return cachedConfig
}
```

---

## MDM (Mobile Device Management) Support

For enterprise deployments, IT can enforce policies via MDM:

```xml
<!-- macOS MDM Profile -->
<key>com.anthropic.claude-code</key>
<dict>
  <key>disableAutoUpdates</key>
  <true/>
  <key>allowedModels</key>
  <array>
    <string>claude-sonnet-4-6</string>
  </array>
  <key>disableBypassPermissions</key>
  <true/>
</dict>
```

MDM settings override all user settings except explicit CLI flags.

---

## Config CLI Operations

```bash
# View current config
claude config

# Set a value
claude config set model claude-opus-4-6
claude config set theme light

# In global settings
claude config set --global model claude-opus-4-6

# In project settings
claude config set --project permissions.allow '["Bash(npm *)"]'
```
