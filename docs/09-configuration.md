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

---

## `CLAUDE_*` Environment Variables — Complete Reference

All names sourced from `/home/deck/Projects/claude-code/src`. Grouped by category. **Bold** = most commonly used by agent builders.

---

### Hook Subprocess Variables (injected by Claude Code into hook processes)

These are set by Claude Code before spawning hook subprocesses — not user-set.

| Variable | Set When | Description |
|----------|----------|-------------|
| **`CLAUDE_PROJECT_DIR`** | All hooks | Stable project root (never worktree path; POSIX on Windows) |
| **`CLAUDE_PLUGIN_ROOT`** | Plugin/skill hooks | Plugin or skill install path |
| **`CLAUDE_PLUGIN_DATA`** | Plugin hooks | Plugin data directory |
| **`CLAUDE_PLUGIN_OPTION_*`** | Plugin hooks | One var per user config option; key uppercased (`apiKey`→`CLAUDE_PLUGIN_OPTION_APIKEY`). Sensitive values included. See `docs/08-permissions-hooks.md` |
| **`CLAUDE_ENV_FILE`** | SessionStart/Setup/CwdChanged/FileChanged (bash only) | Path to `.sh` file hook can write `VAR=value` exports into; injected into subsequent bash commands |
| `CLAUDE_CODE_SHELL_PREFIX` | bash hooks | Inherited shell command prefix wrapper |

---

### Auth & OAuth

| Variable | Source | Description |
|----------|--------|-------------|
| **`CLAUDE_CODE_OAUTH_TOKEN`** | User-set | OAuth token for API auth (`utils/auth.ts:112`) |
| `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` | User-set | File descriptor for secure OAuth token read |
| `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` | User-set | Refresh token (requires `CLAUDE_CODE_OAUTH_SCOPES`) |
| `CLAUDE_CODE_OAUTH_SCOPES` | User-set | OAuth scopes list (required with REFRESH_TOKEN) |
| `CLAUDE_CODE_OAUTH_CLIENT_ID` | User-set | Override OAuth client ID (e.g. Xcode integration) |
| `CLAUDE_CODE_CUSTOM_OAUTH_URL` | Internal (ant-only) | Custom OAuth base URL for FedStart deployments |
| `CLAUDE_TRUSTED_DEVICE_TOKEN` | Enterprise-injected | Enterprise device token (bypasses enrollment) |
| `CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR` | User-set | Secure file descriptor for API key |
| `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR` | User-set | Secure file descriptor for WebSocket auth |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | User-set | Skip Bedrock authentication |
| `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` | User-set | Skip Foundry authentication |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH` | User-set | Skip Vertex authentication |
| `CLAUDE_CODE_USE_BEDROCK` | User-set | Force Bedrock provider (`utils/model/bedrock.ts`) |
| `CLAUDE_CODE_USE_FOUNDRY` | User-set | Force Foundry provider |
| `CLAUDE_CODE_USE_VERTEX` | User-set | Force Vertex provider |

---

### Config & Paths

| Variable | Source | Description |
|----------|--------|-------------|
| **`CLAUDE_CONFIG_DIR`** | User-set | Override config directory (default: `~/.claude`; `utils/env.ts:25`) |
| `CLAUDE_CODE_TMPDIR` | User-set | Override temp directory |
| `CLAUDE_CODE_MANAGED_SETTINGS_PATH` | Admin-set | Override managed settings path |
| `CLAUDE_CODE_SETTINGS_SCHEMA_URL` | Admin-set | Override settings schema URL |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | User-set | Remote memory directory override |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` | User-set | Additional directories to search for CLAUDE.md |
| `CLAUDE_CODE_PLUGIN_CACHE_DIR` | User-set | Plugin cache directory |
| `CLAUDE_CODE_PLUGIN_SEED_DIR` | User-set | Plugin seed directory |
| `CLAUDE_CODE_SHELL` | User-set | Override shell binary for command execution |
| `CLAUDE_CODE_GIT_BASH_PATH` | User-set (Windows) | Path to Git Bash executable |
| `CLAUDE_CODE_BASE_REF` | CI-set | Base git reference |

---

### Session & Bridge (mostly internal / cloud-set)

| Variable | Source | Description |
|----------|--------|-------------|
| `CLAUDE_CODE_SESSION_ID` | Internal | Session identifier |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | Bridge-injected | Access token for session ingress API |
| `CLAUDE_CODE_ENVIRONMENT_KIND` | Bridge-injected | Environment context: `'bridge'`, `'local'`, etc. |
| `CLAUDE_CODE_REMOTE` | Cloud-set | Boolean: running in remote/cloud mode |
| `CLAUDE_CODE_MESSAGING_SOCKET` | Internal | UDS socket path for concurrent session messaging |
| `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` | User-set | Timeout for SessionEnd/Stop hooks (default: 1500 ms) |
| `CLAUDE_CODE_PARENT_SESSION_ID` | Internal | Parent session identifier (subagents) |
| `CLAUDE_CODE_ENTRYPOINT` | Internal | Entrypoint type: `mcp`, `claude-code-github-action`, etc. |
| `CLAUDE_CODE_ENVIRONMENT_RUNNER_VERSION` | Internal | Environment runner version |

---

### Transport & Network

| Variable | Source | Description |
|----------|--------|-------------|
| `CLAUDE_CODE_API_BASE_URL` | User-set | Override Claude API base URL |
| `CLAUDE_CODE_SSE_PORT` | Internal | Port for SSE endpoint (survives tmux) |
| `CLAUDE_CODE_USE_CCR_V2` | Internal | Enable SSETransport v2 |
| `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2` | Internal | Enable HybridTransport (WS reads + POST writes) |
| `CLAUDE_CODE_MAX_RETRIES` | User-set | Maximum API retries (`services/api/withRetry.ts`) |
| `CLAUDE_CODE_STREAM_IDLE_TIMEOUT_MS` | User-set | Idle timeout for API streams |
| `CLAUDE_CODE_PROXY_RESOLVES_HOSTS` | User-set | Proxy resolves hostnames |
| `CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK` | User-set | Don't fall back to non-streaming |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | User-set | Minimize non-essential API calls |

---

### Model & Context

| Variable | Source | Description |
|----------|--------|-------------|
| **`CLAUDE_CODE_MAX_OUTPUT_TOKENS`** | User-set | Override max output tokens (`query.ts:1202`) |
| `CLAUDE_CODE_MAX_CONTEXT_TOKENS` | User-set | Override max context window |
| `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` | User-set | Max tokens for file read tool |
| `CLAUDE_CODE_DISABLE_THINKING` | User-set | Disable extended thinking |
| `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` | User-set | Disable adaptive thinking depth |
| `CLAUDE_CODE_DISABLE_FAST_MODE` | User-set | Disable fast mode |
| `CLAUDE_CODE_EFFORT_LEVEL` | User-set | Override effort level (ignores settings) |
| `CLAUDE_CODE_SUBAGENT_MODEL` | User-set | Model override for subagents |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | User-set | Message window for auto-compaction |
| `CLAUDE_CODE_AUTOCOMPACT_PCT_OVERRIDE` | User-set | Override auto-compact percentage |
| `CLAUDE_CODE_DISABLE_PRECOMPACT_SKIP` | User-set | Don't skip precompaction |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | User-set | Disable 1M context model option |
| `CLAUDE_CODE_SM_COMPACT` | User-set | Small model compaction mode |
| `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` | User-set | Max concurrent tool uses |

---

### Features & Behavior Toggles

| Variable | Source | Description |
|----------|--------|-------------|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | User-set | Disable automatic memory management |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | User-set | Disable background task execution |
| `CLAUDE_CODE_DISABLE_GIT_INSTRUCTIONS` | User-set | Disable git-specific instructions in system prompt |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | User-set | Hard-off CLAUDE.md loading |
| `CLAUDE_CODE_DISABLE_CRON` | User-set | Disable cron scheduling |
| `CLAUDE_CODE_DISABLE_ATTACHMENTS` | User-set | Disable file attachments |
| `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING` | User-set | Disable file checkpointing |
| `CLAUDE_CODE_DISABLE_SESSION_DATA_UPLOAD` | User-set | Skip session data upload |
| `CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY` | User-set | Disable feedback survey |
| `CLAUDE_CODE_DISABLE_ADVISOR_TOOL` | User-set | Disable advisor tool suggestions |
| `CLAUDE_CODE_DISABLE_OFFICIAL_MARKETPLACE_AUTOINSTALL` | User-set | Skip auto-install official plugins |
| `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION` | User-set | Enable/disable prompt suggestion feature |
| `CLAUDE_CODE_ENABLE_TASKS` | User-set | Enable task list feature |
| `CLAUDE_CODE_SKIP_PROMPT_HISTORY` | User-set | Skip loading prompt history on startup |
| `CLAUDE_CODE_DONT_INHERIT_ENV` | User-set | Don't inherit parent environment in subprocesses |
| `CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING` | User-set | Enable SDK-side file checkpointing |
| `CLAUDE_CODE_PLAN_MODE_REQUIRED` | User-set | Require plan mode for write operations |

---

### Sandbox & Security

| Variable | Source | Description |
|----------|--------|-------------|
| `CLAUDE_CODE_FORCE_SANDBOX` | Bridge-injected | Force sandbox mode for tools |
| `CLAUDE_CODE_BUBBLEWRAP` | User-set | Use bubblewrap for tool sandboxing |
| `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` | Auto (GHA) | Scrub sensitive env vars from subprocesses (auto-set in GitHub Actions) |
| `CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK` | User-set | Disable command injection detection in BashTool |

---

### IDE & Terminal Integration

| Variable | Source | Description |
|----------|--------|-------------|
| **`CLAUDE_CODE_AUTO_CONNECT_IDE`** | User-set | Auto-connect to detected IDE |
| `CLAUDE_CODE_IDE_SKIP_AUTO_INSTALL` | User-set | Skip auto IDE extension install |
| `CLAUDE_CODE_IDE_SKIP_VALID_CHECK` | User-set | Skip IDE validity check |
| `CLAUDE_CODE_TMUX_SESSION` | User-set | Named tmux session |
| `CLAUDE_CODE_TMUX_PREFIX` | User-set | tmux key binding prefix |
| `CLAUDE_CODE_TMUX_TRUECOLOR` | User-set | Enable truecolor in tmux |
| `CLAUDE_CODE_HOST_PLATFORM` | Container-set | Override detected platform |
| `CLAUDE_CODE_SHELL_PREFIX` | User-set | Shell command prefix |
| `CLAUDE_CODE_PS_SHELL_PREFIX` | User-set | PowerShell command prefix |

---

### UI & Display

| Variable | Source | Description |
|----------|--------|-------------|
| `CLAUDE_CODE_SYNTAX_HIGHLIGHT` | User-set | Enable syntax highlighting |
| `CLAUDE_CODE_NO_FLICKER` | User-set | Reduce UI flicker |
| `CLAUDE_CODE_DISABLE_VIRTUAL_SCROLL` | User-set | Disable virtual scroll optimization |
| `CLAUDE_CODE_DISABLE_MOUSE` | User-set | Disable mouse input |
| `CLAUDE_CODE_DISABLE_TERMINAL_TITLE` | User-set | Don't set terminal title |
| `CLAUDE_CODE_FORCE_FULL_LOGO` | User-set | Force full ASCII logo render |
| `CLAUDE_CODE_ACCESSIBILITY` | User-set | Enable accessibility mode |
| `CLAUDE_CODE_STREAMLINED_OUTPUT` | User-set | Streamlined output format (non-interactive) |

---

### Glob & File Search

| Variable | Source | Description |
|----------|--------|-------------|
| `CLAUDE_CODE_GLOB_TIMEOUT_SECONDS` | User-set | File glob timeout |
| `CLAUDE_CODE_GLOB_HIDDEN` | User-set | Include hidden files in glob |
| `CLAUDE_CODE_GLOB_NO_IGNORE` | User-set | Ignore `.gitignore` in glob |
| `CLAUDE_CODE_USE_NATIVE_FILE_SEARCH` | User-set | Use native file search instead of ripgrep |
| `CLAUDE_CODE_PWSH_PARSE_TIMEOUT_MS` | User-set | PowerShell parse timeout |

---

### Telemetry & Monitoring

| Variable | Source | Description |
|----------|--------|-------------|
| `CLAUDE_CODE_ENABLE_TELEMETRY` | User-set | Enable/disable telemetry reporting |
| `CLAUDE_CODE_METRICS_ENDPOINT` | Admin-set | Datadog metrics endpoint |
| `CLAUDE_CODE_OTEL_FLUSH_TIMEOUT_MS` | Admin-set | OpenTelemetry flush timeout |
| `CLAUDE_CODE_OTEL_SHUTDOWN_TIMEOUT_MS` | Admin-set | OpenTelemetry shutdown timeout |
| `CLAUDE_CODE_DIAGNOSTICS_FILE` | User-set | Write diagnostics to file |
| `CLAUDE_CODE_SESSION_LOG` | User-set | Session log file path |
| `CLAUDE_CODE_JSONL_TRANSCRIPT` | User-set | JSONL transcript file path |

---

### Debug & Development

| Variable | Source | Description |
|----------|--------|-------------|
| `CLAUDE_CODE_DEBUG_LOG_LEVEL` | Dev-set | Debug logging level (`debug`/`info`/`warn`/`error`) |
| `CLAUDE_CODE_DEBUG_LOGS_DIR` | Dev-set | Directory for debug log files |
| `CLAUDE_CODE_PROFILE_STARTUP` | Dev-set | Profile startup performance |
| `CLAUDE_CODE_PROFILE_QUERY` | Dev-set | Profile query execution |
| `CLAUDE_CODE_PERFETTO_TRACE` | Dev-set | Enable Perfetto tracing |
| `CLAUDE_CODE_FRAME_TIMING_LOG` | Dev-set | File path for frame timing measurements |
| `CLAUDE_CODE_SLOW_OPERATION_THRESHOLD_MS` | Dev-set | Threshold for slow operations (`utils/slowOperations.ts`) |
| `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER` | Test-set | Exit after first render (CI testing) |
| `CLAUDE_CODE_STALL_TIMEOUT_MS_FOR_TESTING` | Test-set | Stall timeout for testing |
| `CLAUDE_CODE_OVERRIDE_DATE` | Test-set | Override current date (`utils/dates.ts`) |

---

### TypeScript Exported Constants

These are compile-time constants, not env vars:

| Name | File | Value |
|------|------|-------|
| `CLAUDE_AI_BASE_URL` | `constants/product.ts:4` | `'https://claude.ai'` |
| `CLAUDE_AI_INFERENCE_SCOPE` | `constants/oauth.ts:33` | `'user:inference'` |
| `CLAUDE_AI_PROFILE_SCOPE` | `constants/oauth.ts:34` | `'user:profile'` |
| `CLAUDE_AI_OAUTH_SCOPES` | `constants/oauth.ts:45` | Array of all OAuth scopes |
| `CLAUDE_CODE_20250219_BETA_HEADER` | `constants/betas.ts:3` | `'claude-code-20250219'` |
| `CLAUDE_FOLDER_PERMISSION_PATTERN` | `tools/FileEditTool/constants.ts:5` | `'/.claude/**'` |
| `CLAUDE_IN_CHROME_MCP_SERVER_NAME` | `utils/claudeInChrome/common.ts:12` | `'claude-in-chrome'` |
| `CLAUDE_CODE_GUIDE_AGENT_TYPE` | `tools/AgentTool/built-in/claudeCodeGuideAgent.ts:21` | `'claude-code-guide'` |
| `CLAUDE_SOCKET_PREFIX` | `utils/tmuxSocket.ts:36` | `'claude'` |
| `CLAUDE_ALIAS_REGEX` | `utils/shellConfig.ts:12` | Regex for shell `alias claude=` |
| `CLAUDE_3_7_SONNET_CONFIG` … `CLAUDE_SONNET_4_6_CONFIG` | `utils/model/configs.ts` | Per-model config objects |
