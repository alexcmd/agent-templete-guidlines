# MCP Registration: Deep Reference

Complete guide to MCP server registration paths in Claude Code — across plugins, agents, settings, `.mcp.json`, CLI commands, and the enterprise policy system. Sourced from deep analysis of `src/services/mcp/config.ts`, `src/utils/plugins/mcpPluginIntegration.ts`, `src/services/mcp/types.ts`, and `src/entrypoints/mcp.ts`.

---

## Registration Paths Overview

There are **7 distinct registration paths** for MCP servers in Claude Code:

| Path | Where | Scope | Priority |
|------|-------|-------|----------|
| **Enterprise exclusive** | `managed-mcp.json` | Enterprise only | 0 (overrides all) |
| **Settings (user)** | `~/.claude/settings.json` → `mcpServers` | User-global | 1 |
| **Settings (project)** | `.mcp.json` in project hierarchy | Project | 2 |
| **Settings (local)** | `.claude/settings.local.json` → `mcpServers` | Project-local | 3 |
| **Plugin manifest** | plugin `manifest.json` → `mcpServers` | Plugin-scoped | 4 (lowest, deduped) |
| **Agent definition** | agent `.md` → `mcpServers` frontmatter | Agent session | 5 (session only) |
| **Dynamic (CLI flag)** | `--mcp-config` flag | Session | 6 |

The merge order is: **plugin < user < project < local**. Enterprise exclusive mode bypasses all user-level sources.

---

## Full McpServerConfig Type Reference

Every server type shares a discriminated union. All types from `src/services/mcp/types.ts`:

### stdio (most common)
```typescript
{
  type?: 'stdio'   // Optional for backwards compatibility — omitting means stdio
  command: string  // Executable (required, non-empty)
  args?: string[]  // Defaults to []
  env?: Record<string, string>  // Environment variable overrides
}
```

### sse (Server-Sent Events, HTTP/1.1)
```typescript
{
  type: 'sse'
  url: string
  headers?: Record<string, string>
  headersHelper?: string        // Dynamic header injection script (see below)
  oauth?: {
    clientId?: string
    callbackPort?: number       // Fixed port for redirect URI pre-registration
    authServerMetadataUrl?: string  // Must be https://
    xaa?: boolean               // Enable Cross-App Access (SEP-990)
  }
}
```

### http (Streamable HTTP, MCP 2025-03-26 spec)
```typescript
{
  type: 'http'
  url: string
  headers?: Record<string, string>
  headersHelper?: string
  oauth?: { ... }               // Same as SSE
}
```

### ws (WebSocket)
```typescript
{
  type: 'ws'
  url: string
  headers?: Record<string, string>
  headersHelper?: string
}
```

### sdk (In-process, advanced)
```typescript
{
  type: 'sdk'
  name: string                  // Must be unique identifier
}
```
Used for IDE extensions and SDK-managed transports. Never spawns a process. Exempt from enterprise policy filtering.

### sse-ide / ws-ide (IDE bridge transports, internal)
```typescript
{
  type: 'sse-ide'
  url: string
  ideName: string
  ideRunningInWindows?: boolean
}
// ws-ide adds: authToken?: string
```
Internal only — used by VS Code and JetBrains extensions via the bridge.

### claudeai-proxy (Claude.ai connectors, internal)
```typescript
{
  type: 'claudeai-proxy'
  url: string
  id: string
}
```
Internal only — used for Claude.ai connector servers fetched from the web app.

---

## ScopedMcpServerConfig

Every server gets a `scope` field added at config-build time:

```typescript
type ScopedMcpServerConfig = McpServerConfig & {
  scope: 'local' | 'user' | 'project' | 'dynamic' | 'enterprise' | 'claudeai' | 'managed'
  pluginSource?: string   // For plugin servers: plugin's LoadedPlugin.source
}
```

---

## Registration via `.mcp.json`

The `.mcp.json` file is the primary project-scoped MCP config file:

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/",
      "headers": {
        "Authorization": "Bearer ${GITHUB_TOKEN}"
      }
    },
    "my-local-server": {
      "command": "node",
      "args": ["./tools/mcp-server/index.js"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    },
    "sentry": {
      "type": "http",
      "url": "https://mcp.sentry.dev/mcp"
    }
  }
}
```

**Traversal**: Claude Code walks UP from cwd to filesystem root, reading every `.mcp.json` it finds. Closer files override parent files (same server name). This means:

```
/home/user/projects/my-repo/         ← cwd: .mcp.json wins
/home/user/projects/                 ← parent .mcp.json overridden
/home/user/                          ← grandparent .mcp.json overridden
```

**Approval**: Project servers from `.mcp.json` require user approval before connecting. Status tracked in `.claude/settings.local.json`:
```json
{
  "approvedProjectMcpServers": ["github", "my-local-server"],
  "disabledMcpjsonServers": ["old-server"]
}
```

**Adding via CLI**:
```bash
# stdio server (scope: local = .claude/settings.local.json by default)
claude mcp add my-server node /path/to/server.js

# stdio with args and env
claude mcp add -e API_KEY=xxx -e DB_URL=yyy my-server -- npx my-mcp-server --port 3000

# HTTP server
claude mcp add --transport http github https://api.githubcopilot.com/mcp/

# HTTP with auth header
claude mcp add --transport http corridor https://app.corridor.dev/api/mcp --header "Authorization: Bearer TOKEN"

# SSE server with OAuth
claude mcp add --transport sse my-oauth-server https://mcp.example.com/sse \
  --client-id my-client-id --client-secret

# Specific scope (project = .mcp.json)
claude mcp add --scope project github https://api.githubcopilot.com/mcp/

# User scope (~/.claude/settings.json)
claude mcp add --scope user shared-tools node /tools/shared.js
```

**Available scopes**:
- `local` (default) — `.claude/settings.local.json` (gitignored)
- `project` — `.mcp.json` in cwd (committed, shared)
- `user` — `~/.claude/settings.json` (user-global)

**Server name rules** (from `addMcpConfig()`):
- Only `[a-zA-Z0-9_-]` characters allowed
- `claude-in-chrome` is reserved
- Cannot be added if enterprise MCP config exists

---

## Registration via Plugin Manifest

Plugins declare MCP servers in `manifest.json`. There are **4 formats** for `mcpServers`:

### Format 1: Inline object (direct config)
```json
{
  "mcpServers": {
    "my-api": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/server/index.js"],
      "env": {
        "API_KEY": "${user_config.api_key}",
        "BASE_URL": "${BASE_URL}"
      }
    },
    "my-remote": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${user_config.token}"
      }
    }
  }
}
```

### Format 2: Path to a `.mcp.json` file
```json
{
  "mcpServers": "servers/.mcp.json"
}
```
Relative to `plugin.path`. The file format is the same `{ mcpServers: {...} }` object.

### Format 3: Array of paths/specs
```json
{
  "mcpServers": [
    "servers/main-server.json",
    "servers/backup.json",
    {
      "inline-server": {
        "command": "node",
        "args": ["${CLAUDE_PLUGIN_ROOT}/inline.js"]
      }
    }
  ]
}
```
Merged in order — later entries override same-named servers.

### Format 4: MCPB/DXT binary bundle
```json
{
  "mcpServers": "https://example.com/my-plugin.mcpb"
}
```
or
```json
{
  "mcpServers": "local/path/to/plugin.dxt"
}
```
`.mcpb` and `.dxt` files are zip archives containing a `manifest.json` that defines the MCP server config. Detected by `isMcpbSource()` — any path ending in `.mcpb` or `.dxt`.

**MCPB flow**:
1. Download from URL (or read from path)
2. Compute SHA-256 hash for cache key
3. Unzip to `~/.claude/plugins/data/<plugin-id>/extracted/`
4. Read `manifest.json` from zip
5. If manifest requires `userConfig`, prompt user to configure via `/plugin` menu
6. Convert manifest → McpServerConfig and register

---

## Plugin Server Special Variables

Plugin MCP server configs get three special variable substitutions (in `resolvePluginMcpEnvironment()`):

### 1. `${CLAUDE_PLUGIN_ROOT}`
Absolute path to the plugin's installation directory:
```json
{
  "command": "${CLAUDE_PLUGIN_ROOT}/bin/my-server",
  "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config/default.json"]
}
```

### 2. `${CLAUDE_PLUGIN_DATA}`
Absolute path to the plugin's data directory (`~/.claude/plugins/data/<plugin-source>/`):
```json
{
  "env": {
    "CACHE_DIR": "${CLAUDE_PLUGIN_DATA}/cache",
    "DB_PATH": "${CLAUDE_PLUGIN_DATA}/database.sqlite"
  }
}
```

### 3. `${user_config.<field>}`
Values from user-provided configuration (via `/plugin` configure dialog):
```json
{
  "env": {
    "API_KEY": "${user_config.api_key}",
    "WORKSPACE": "${user_config.workspace_id}"
  },
  "args": ["--token", "${user_config.auth_token}"]
}
```

Substitution order (later wins):
1. `${CLAUDE_PLUGIN_ROOT}` and `${CLAUDE_PLUGIN_DATA}` substituted first
2. `${user_config.X}` substituted from merged plugin options + channel config
3. `${ENV_VAR}` substituted from process environment last

---

## Plugin User Configuration (userConfig)

Plugins that need user-provided values declare a `userConfig` schema in their manifest:

```json
{
  "name": "my-plugin",
  "description": "Requires API credentials",
  "userConfig": {
    "api_key": {
      "type": "string",
      "description": "Your API key",
      "required": true,
      "sensitive": true
    },
    "workspace_id": {
      "type": "string",
      "description": "Workspace ID",
      "required": false,
      "default": "default"
    }
  },
  "mcpServers": {
    "my-api": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/server.js"],
      "env": {
        "API_KEY": "${user_config.api_key}",
        "WORKSPACE": "${user_config.workspace_id}"
      }
    }
  }
}
```

**Channels** (per-server user config for assistant-mode plugins):
```json
{
  "channels": [
    {
      "server": "my-api",
      "displayName": "My API Server",
      "userConfig": {
        "api_token": {
          "type": "string",
          "required": true,
          "sensitive": true,
          "description": "API token for authentication"
        }
      }
    }
  ]
}
```

When required fields are missing, the server is **not loaded** — it enters `needs-config` state. User configures via `/plugin` → Configure.

Sensitive values are stored in the OS keychain (macOS) or settings file (other platforms).

---

## Plugin Server Namespacing

Plugin servers get a `plugin:` prefix to prevent name collisions:

```
Manifest server name: "my-api"
Plugin name: "my-plugin"
→ Scoped name: "plugin:my-plugin:my-api"
→ Tool name prefix: "mcp__plugin_my-plugin_my-api__"
```

This means plugin tools are accessible as:
```
mcp__plugin_my-plugin_my-api__tool-name
```

In permission rules:
```json
{
  "permissions": {
    "allow": ["mcp__plugin_my-plugin_my-api__*"]
  }
}
```

---

## Plugin Server Deduplication

When a plugin server has the same underlying command/URL as a manually configured server, it's **suppressed** (manual wins). Deduplication uses content signatures:

**Signature computation** (`getMcpServerSignature()`):
```
stdio: `stdio:${JSON.stringify([command, ...args])}`
remote: `url:${normalizedUrl}`   # Strips CCR proxy wrapping
sdk/sse-ide/ws-ide: null         # Not deduped (no command or URL)
```

**Dedup rules**:
1. Manual server signature matches plugin server → plugin suppressed
2. Two plugin servers with same signature → first-loaded wins

Suppressed servers appear in `/plugin` UI with a "duplicate of" note but don't connect.

**Claude.ai connectors** are also deduplicated against manual servers by URL signature, since both might point at the same endpoint (e.g., both `mcp__slack__*` and `mcp__claude_ai_Slack__*` connecting to `mcp.slack.com`).

---

## Registration via Agent Definition

Agents can declare MCP servers in their frontmatter. Two forms:

### Reference existing server by name
```markdown
---
name: github-workflow
description: "Manages GitHub PRs and Actions"
mcpServers:
  - github
  - slack
requiredMcpServers:
  - github
---
```

The string `"github"` references a server named `github` in the merged MCP config. If that server isn't configured, it's silently ignored (unless listed in `requiredMcpServers`).

### Inline server definition
```markdown
---
name: api-specialist
description: "Works with the internal API"
mcpServers:
  - internal-api:
      type: http
      url: "https://internal.company.com/mcp"
      headers:
        Authorization: "Bearer ${INTERNAL_API_TOKEN}"
  - my-tools:
      command: node
      args: ["/shared/tools/mcp-server.js"]
      env:
        ENV: production
---
```

The inline server (`{ name: config }` object) is started only for this agent's session. It does not persist beyond the agent's lifetime.

### `requiredMcpServers` filter

If listed servers aren't available, the agent is hidden from the model entirely:
```yaml
requiredMcpServers:
  - github    # Agent hidden unless 'github' server is connected
  - jira      # Case-insensitive substring match: matches 'jira', 'jira-prod', 'jira-dev'
```

Matching is **case-insensitive substring**: `"github"` matches any configured server whose name contains "github".

---

## Enterprise Policy: Allowlist and Denylist

Configured in `managed-settings.json` or `policySettings`:

```json
{
  "allowedMcpServers": [
    { "serverName": "github" },
    { "serverName": "slack" },
    { "serverCommand": ["npx", "-y", "@modelcontextprotocol/server-filesystem"] },
    { "serverUrl": "https://mcp.approved-vendor.com/*" }
  ],
  "deniedMcpServers": [
    { "serverName": "untrusted-server" },
    { "serverCommand": ["python", "malicious.py"] },
    { "serverUrl": "https://untrusted.example.com/*" }
  ],
  "allowManagedMcpServersOnly": true
}
```

**Three entry types**:
- `{ serverName: string }` — match by name
- `{ serverCommand: string[] }` — match by exact command array (stdio only)
- `{ serverUrl: string }` — match by URL pattern with `*` wildcard (remote servers only)

**Precedence rules**:
- Denylist always wins over allowlist
- `allowedMcpServers: []` (empty array) blocks ALL servers
- `allowedMcpServers: undefined` allows all (no restriction)
- If allowlist has command entries, stdio servers MUST match one
- If allowlist has URL entries, remote servers MUST match one
- SDK-type servers are exempt from policy (they're SDK-managed transport stubs)

**`allowManagedMcpServersOnly: true`**: The `allowedMcpServers` allowlist is read only from managed settings. Users can still add servers to their own config, but only admin-approved servers are actually allowed. Users can still deny servers for themselves.

### Enterprise Exclusive Mode

When `managed-mcp.json` exists, it takes **exclusive control**:
```
/etc/claude/managed-mcp.json   (Linux)
/Library/Application Support/Claude/managed-mcp.json  (macOS)
```

File format:
```json
{
  "mcpServers": {
    "company-api": { ... },
    "approved-github": { ... }
  }
}
```

When this file exists:
- User, project, local, and plugin servers are **all ignored**
- Only the enterprise servers are available
- `addMcpConfig()` throws an error if user tries to add servers

Exception: SDK-type servers from IDE extensions still work (hardcoded exemption for `claude-vscode`).

---

## Per-Session Server Disable/Enable

Users can disable specific servers without removing them:

```json
// .claude/settings.local.json
{
  "disabledMcpServers": ["noisy-server", "slow-server"],
  "enabledMcpServers": ["computer-use"]   // For built-in default-disabled servers
}
```

`disabledMcpServers` — servers to disable (opt-out)
`enabledMcpServers` — servers to enable that default to disabled (opt-in for built-ins like `computer-use`)

Via the `/mcp` command:
```bash
# In Claude Code session
/mcp  # Opens MCP management UI with enable/disable toggles
```

---

## Tool Naming and Normalization

MCP tool names must match `^[a-zA-Z0-9_-]{1,64}$`. Server names undergo normalization:

```
Raw server name: "claude.ai Slack"
→ normalizeNameForMCP() → "claude_ai_Slack"
→ Final tool prefix: "mcp__claude_ai_Slack__"
```

Normalization (`src/services/mcp/normalization.ts`):
- Replace any `[^a-zA-Z0-9_-]` with `_`
- For `claude.ai `-prefixed names: also collapse consecutive `_` and strip leading/trailing `_`

**Name deduplication**: If two tools from different servers have the same name after normalization, one gets a numeric suffix (`__2`, `__3`, etc.) to prevent collisions.

**Tool description cap**: Descriptions are capped at **2048 characters** (`MAX_MCP_DESCRIPTION_LENGTH`). OpenAPI-generated servers sometimes dump 15-60KB of endpoint documentation — the cap prevents that from bloating every conversation.

---

## MCP Connection States

Each server progresses through connection states:

```
pending → connected     (success)
        → failed        (connection error)
        → needs-auth    (OAuth required / 401 error)
        → disabled      (user-disabled)
```

**Reconnection**: Failed servers retry up to `maxReconnectAttempts` times with exponential backoff.

**Session expiry**: HTTP servers return 404 + JSON-RPC code -32001 when a session is expired. Claude Code detects this (`isMcpSessionExpiredError()`) and reconnects transparently.

---

## Claude Code as an MCP Server

Claude Code can expose itself as an MCP server, making all its built-in tools available to external agents:

```bash
claude mcp   # Start Claude Code as MCP server (stdio transport)
```

**Implementation** (`src/entrypoints/mcp.ts`):
- Creates an `@modelcontextprotocol/sdk` Server named `claude/tengu`
- Exposes all built-in tools via `ListToolsRequestSchema` handler
- Executes tools via `CallToolRequestSchema` handler using the same permission context
- Uses `StdioServerTransport` for communication

This enables:
- External orchestrators to use Claude Code's tools (Bash, Read, Write, Edit, etc.)
- Other AI agents to delegate tool calls to Claude Code
- Custom pipelines that use Claude Code tools programmatically

---

## Dynamic MCP Server Registration

For SDK and programmatic use, servers can be registered dynamically:

```bash
# Via CLI flag (session-only, not persisted)
claude --mcp-config /path/to/custom.mcp.json

# Via environment variable
MCP_CONFIG=/path/to/custom.mcp.json claude
```

The `--mcp-config` flag accepts a `.mcp.json` formatted file. Servers from this file are subject to policy filtering but go through the same deduplication flow.

In SDK contexts, servers can be registered via the `mcp_set_servers` control message:
```typescript
// SDK V2 Query.setMcpServers()
// Programmatically set the MCP servers for the session
```

---

## Official MCP Registry

Claude Code queries the Anthropic official MCP registry at startup (fire-and-forget):

```
GET https://api.anthropic.com/mcp-registry/v0/servers?version=latest&visibility=commercial
```

This populates an `officialUrls` set used by `isOfficialMcpUrl()`. The registry data influences:
- Trust signals for connected servers
- Permission prompt UI (official servers may get different treatment)
- Analytics and telemetry

Disable the registry fetch:
```bash
CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1 claude
```

---

## headersHelper (Dynamic Header Injection)

A `headersHelper` field exists on SSE, HTTP, and WebSocket configs for dynamic header generation:

```json
{
  "type": "http",
  "url": "https://api.example.com/mcp",
  "headersHelper": "/path/to/header-generator.sh"
}
```

The `headersHelper` is a script path that, when invoked, returns headers as JSON. This enables:
- Token refresh flows
- Time-based auth signatures
- Dynamic credentials from external vaults
- Per-request header generation

Implementation is in `src/services/mcp/headersHelper.ts`.

---

## Complete MCP Configuration Decision Tree

```
Request to use server "X"
  │
  ├─ Enterprise managed-mcp.json exists?
  │    Yes → Only enterprise servers available (no others)
  │    No → Continue
  │
  ├─ strictPluginOnlyCustomization(mcp) set in policy?
  │    Yes → Only plugin + enterprise servers (no user/project/local)
  │    No → Load user/project/local servers
  │
  ├─ Server "X" in deniedMcpServers?
  │    Yes → BLOCKED (always, regardless of allowlist)
  │    No → Continue
  │
  ├─ allowedMcpServers defined?
  │    No (undefined) → ALLOWED
  │    Empty [] → BLOCKED
  │    Has entries → Must match at least one entry
  │        If cmd entries exist: stdio server must match command
  │        If URL entries exist: remote server must match URL pattern
  │        Otherwise: name-based match
  │
  ├─ Server type = 'sdk'?
  │    Yes → ALLOWED (exempt from policy)
  │
  ├─ Server in disabledMcpServers?
  │    Yes → DISABLED (appears in /mcp but doesn't connect)
  │
  └─ PROCEED TO CONNECTION
       ↓
       Attempt connection (pending → connected/failed/needs-auth)
```

---

## Debugging MCP Registration

```bash
# Show all configured servers and their connection status
claude mcp list

# Show specific server details
claude mcp get server-name

# Remove a server
claude mcp remove server-name --scope user

# Test server registration without starting full Claude session
claude --debug 2>&1 | grep -E "mcp|MCP|server"

# Check plugin MCP loading
claude --debug 2>&1 | grep "Loaded.*MCP servers from plugin"

# Check deduplication
claude --debug 2>&1 | grep "Suppressing plugin MCP server"

# Check policy filtering
claude --debug 2>&1 | grep "not allowed by enterprise policy"

# See which .mcp.json files are being loaded
claude --debug 2>&1 | grep "mcp.json"

# Disable nonessential network traffic (registry fetch, etc.)
CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1 claude
```

Common errors and fixes:
- `"Already exists"` → server with that name already in that scope; use `--scope` to add to different scope or remove first
- `"Name contains invalid characters"` → use only `[a-zA-Z0-9_-]`
- `"Enterprise policy"` → contact admin to add server to allowlist
- `"Missing environment variables"` → set the env vars before starting Claude
- `"needs-config"` → configure the plugin via `/plugin` → Configure menu
