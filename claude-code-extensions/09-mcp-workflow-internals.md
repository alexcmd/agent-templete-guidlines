# MCP Workflow Internals

Complete lifecycle of an MCP server from config → startup → tool call → result, extracted from `src/services/mcp/client.ts`, `src/services/mcp/config.ts`, and related files.

---

## Phase 1: Config Assembly (`getClaudeCodeMcpConfigs`)

At startup, Claude Code assembles the final set of MCP servers:

```
1. Check enterprise exclusive mode (managed-mcp.json exists?)
   └── If yes: only enterprise servers, skip all others

2. Check plugin-only policy (strictPluginOnlyCustomization.mcp)
   └── If yes: skip user/project/local, load only plugin servers

3. Load sources in parallel:
   ├── Enterprise servers (getMcpConfigsByScope('enterprise'))
   │     └── from /etc/claude/managed-mcp.json
   ├── User servers (getMcpConfigsByScope('user'))
   │     └── from ~/.claude/settings.json → mcpServers
   ├── Project servers (getMcpConfigsByScope('project'))
   │     └── walk up from cwd to root, read each .mcp.json
   │         filter to only "approved" servers
   ├── Local servers (getMcpConfigsByScope('local'))
   │     └── from .claude/settings.local.json → mcpServers
   └── Plugin servers (loadAllPluginsCacheOnly → getPluginMcpServers)
         ├── For each enabled plugin:
         │     ├── Check plugin.manifest.mcpServers format
         │     ├── Load from inline/file/array/MCPB
         │     ├── Resolve ${CLAUDE_PLUGIN_ROOT}, ${user_config.*}, ${ENV_VAR}
         │     └── Namespace as "plugin:name:server"
         └── Parallel fetch across all plugins

4. Deduplication:
   ├── Content-based: compute command-array or URL signature
   ├── Plugin vs manual: manual wins (plugin suppressed)
   └── Plugin vs plugin: first-loaded wins

5. Policy filtering:
   ├── Apply deniedMcpServers (always blocks)
   └── Apply allowedMcpServers (if defined)

6. Merge (last writer wins per key):
   plugin → user → approvedProject → local

7. Optional: fetch claude.ai connectors (getAllMcpConfigs)
   └── Deduplicate against manual servers by URL signature
   └── Merge with lowest precedence
```

---

## Phase 2: Connection Establishment

Each configured server goes through the connection manager (`MCPConnectionManager.tsx`):

```
For each server in merged config:
  │
  ├─ isMcpServerDisabled(name)?
  │    Yes → state = 'disabled' (appears in /mcp but skipped)
  │
  ├─ Determine transport type:
  │    stdio   → StdioClientTransport (spawn process)
  │    sse     → SSEClientTransport (HTTP/1.1 streaming)
  │    http    → StreamableHTTPClientTransport (HTTP/2)
  │    ws      → WebSocketTransport (custom implementation)
  │    sse-ide → SSEClientTransport with IDE-specific options
  │    ws-ide  → WebSocketTransport with auth token
  │    sdk     → SdkControlClientTransport (in-process)
  │
  ├─ Expand env vars in config (for stdio: command, args, env)
  │
  ├─ Create Client from @modelcontextprotocol/sdk
  │
  ├─ Auth setup:
  │    Remote (SSE/HTTP): attach ClaudeAuthProvider (OAuth)
  │    XAA enabled: attach XAA IdP auth flow
  │    Manual token: use headers directly
  │
  ├─ client.connect(transport)
  │    ├─ Success → state = 'connected'
  │    │     ├─ Read server capabilities (tools, resources, prompts)
  │    │     ├─ Read server instructions (injected into system prompt)
  │    │     └─ Register cleanup handler
  │    ├─ OAuth 401 → state = 'needs-auth'
  │    │     └─ Wait for user to authenticate via McpAuthTool
  │    └─ Error → state = 'failed'
  │          └─ Schedule reconnect (exponential backoff)
  │
  └─ If connected: list tools, prompts, resources
```

### Reconnection

Failed connections retry with backoff:
```
attempt 1: immediate
attempt 2: 2s
attempt 3: 4s
attempt 4: 8s
...up to maxReconnectAttempts
```

Session expiry (HTTP 404 + JSON-RPC -32001) triggers immediate reconnect with fresh session.

---

## Phase 3: Tool Registration

After connection, MCP tools are wrapped as native `Tool` objects:

```
For each tool in client.listTools():
  │
  ├─ Build tool name: buildMcpToolName(serverName, toolName)
  │    = `mcp__${normalizeNameForMCP(serverName)}__${normalizeNameForMCP(toolName)}`
  │
  ├─ Truncate description to 2048 chars
  │
  ├─ Create MCPTool wrapper:
  │    - name: normalized tool name
  │    - description(): returns tool.description
  │    - inputSchema: parsed from tool.inputSchema
  │    - checkPermissions(): checks permission rules for mcp__server__tool
  │    - call(): delegates to mcpClient.callTool()
  │    - isReadOnly(): heuristic from tool name/description
  │    - mcpInfo: { serverName, toolName }
  │
  └─ Register in tool pool alongside built-in tools
```

**Tool pool merge** (`assembleToolPool()`):
1. Built-in tools
2. MCP tools from all connected clients
3. Dedup: same name → log warning, last wins (per client connection order)
4. Filter by permission context (`isEnabled()`, permission mode)

---

## Phase 4: MCP Tool Call Execution

When the model invokes a MCP tool:

```
Model sends tool_use block:
  { id: "toolu_123", name: "mcp__github__create-pr", input: {...} }
  │
  ├─ MCPTool.validateInput(input)
  │    └─ Zod parse against tool's inputSchema
  │
  ├─ MCPTool.checkPermissions(input, context)
  │    ├─ Check permission rules for "mcp__github__create-pr"
  │    ├─ Check permission rules for "mcp__github__*"
  │    └─ Return allow/deny/ask
  │
  ├─ PreToolUse hooks fire (if any match mcp__github__*)
  │
  ├─ MCPTool.call(args, context, canUseTool, parentMessage)
  │    │
  │    ├─ Find client for "github" server
  │    │    └─ isMcpServerDisabled("github")? → abort
  │    │
  │    ├─ Check state (connected? needs-auth? failed?)
  │    │    └─ Not connected → throw with reconnect suggestion
  │    │
  │    ├─ client.request({ method: 'tools/call', params: { name: 'create-pr', arguments: {...} } })
  │    │    ├─ With timeout: getMcpToolTimeoutMs() (default ~27.8 hours)
  │    │    ├─ OAuth token refresh if expired
  │    │    └─ XAA step-up if needed
  │    │
  │    ├─ Handle response:
  │    │    ├─ isError: true → throw McpToolCallError
  │    │    ├─ Content too large → truncate or persist to file
  │    │    ├─ Binary content (images) → resize/downsample, save to disk
  │    │    └─ Success → return MCPToolResult
  │    │
  │    └─ On auth error (401):
  │         └─ Update client state to 'needs-auth'
  │         └─ Return error asking user to re-authenticate
  │
  ├─ PostToolUse hooks fire
  │    └─ updatedMCPToolOutput: replace tool result if hook provides one
  │
  └─ Return tool_result to model
```

### Content Size Management

MCP tool outputs can be large. Claude Code applies several protections:

**Truncation** (`mcpContentNeedsTruncation()`):
- Text content > threshold → truncated with `[Content truncated...]`
- Multi-part content → each part checked independently

**Binary persistence** (`persistBinaryContent()`):
- Image content → resized/downsampled (max dimensions enforced)
- Saved to temp file
- Model receives file path instead of raw binary

**Tool result storage** (`persistToolResult()`):
- Very large outputs → stored to disk
- Model receives "Output stored at /path/to/result" message

---

## Phase 5: MCP Prompt/Skill Fetching

When `MCP_SKILLS` feature is enabled, Claude Code fetches prompts from connected MCP servers:

```
fetchMcpSkillsForClient(client, serverName):
  │
  ├─ client.listPrompts() → ListPromptsResult
  │
  └─ For each prompt:
       ├─ Convert to PromptCommand via createSkillCommand()
       │    ├─ name: "${serverName}:${promptName}" (namespaced)
       │    ├─ description: from prompt.description
       │    ├─ argumentHint: from prompt.arguments
       │    └─ getPromptForCommand: calls client.getPrompt(name, args)
       │
       └─ Register in skill registry with loadedFrom: 'mcp'
            └─ Security: NO shell injection execution (loadedFrom !== 'mcp')
```

MCP skills appear as `/server-name:skill-name` in the skill list.

---

## Phase 6: Resource Access

MCP resources are listed but not automatically fetched:

```
client.listResources() → ServerResource[]
  └─ Stored in AppState.mcpResources[serverName]

Available via built-in tools:
  ListMcpResourcesTool:
    → Lists all resources from all connected servers
    → Returns: { server, uri, name, mimeType, description }[]

  ReadMcpResourceTool:
    → Fetches specific resource by URI
    → client.readResource({ uri })
    → Returns content (text or blob)
```

Resources don't appear in the model's tool list — the model uses `ListMcpResourcesTool` and `ReadMcpResourceTool` to access them.

---

## Phase 7: Elicitation Protocol

MCP servers can request structured user input via the elicitation protocol:

```
MCP server sends: ElicitRequest
  {
    message: "Please provide your API credentials",
    requestedSchema: {
      type: "object",
      properties: {
        api_key: { type: "string", description: "API key" },
        region: { type: "string", enum: ["us-east", "eu-west"] }
      }
    }
  }
  │
  ├─ runElicitationHooks(hookInput) fires 'Elicitation' hooks
  │    └─ Hook can auto-fill or cancel
  │
  ├─ If not handled by hook:
  │    └─ Show form to user in UI
  │
  └─ runElicitationResultHooks(result) fires 'ElicitationResult' hooks
       └─ Hook can inspect/log the user's response
```

The elicitation result is sent back to the MCP server as `ElicitResult`.

---

## MCPb/DXT Bundle Workflow

For marketplace plugins using MCPB format:

```
Plugin manifest: { "mcpServers": "https://example.com/plugin.mcpb" }
  │
  1. isMcpbSource("https://...mcpb") → true
  │
  2. computeHash(source URL)
  │
  3. Check cache (~/.claude/plugins/data/<plugin>/extracted/)
  │    Hit → use cached extraction, verify hash
  │    Miss → download
  │
  4. Download (if needed):
  │    └─ axios.get(url, { responseType: 'arraybuffer' })
  │    └─ SHA-256 hash for integrity
  │
  5. Unzip to extraction path:
  │    └─ parseZipModes(buffer) → file list
  │    └─ unzipFile(buffer, extractedPath)
  │
  6. parseAndValidateManifestFromBytes(buffer)
  │    └─ Read manifest.json from zip
  │    └─ Validate against McpbManifest schema
  │
  7. Check if userConfig required:
  │    └─ validateUserConfig(savedConfig, manifest.userConfig)
  │    └─ If validation fails → return { status: 'needs-config' }
  │         └─ Server not loaded, user prompted via /plugin
  │
  8. Convert manifest → McpServerConfig:
  │    └─ command: manifest.server.entry_point
  │    └─ args: manifest.server.args
  │    └─ env: merged env + ${user_config.*} substituted
  │
  9. Return { manifest, mcpConfig, extractedPath, contentHash }
  │
  10. Register as normal McpServerConfig
```

**Cache invalidation**: If the source URL changes or hash changes, extraction is redone.
**Auto-update**: Official marketplace plugins with `autoUpdate: true` check for new MCPB versions at startup.

---

## OAuth Flow for Remote MCP Servers

```
Server configured with oauth section:
  │
  ├─ Startup: ClaudeAuthProvider attached to SSE/HTTP transport
  │
  ├─ First connection:
  │    ├─ Request goes out without token
  │    ├─ Server returns 401 / OAuth challenge
  │    └─ hasMcpDiscoveryButNoToken() → true
  │         └─ Client state: 'needs-auth'
  │         └─ McpAuthTool appears in tool list
  │
  ├─ Model calls McpAuthTool({ serverName: "my-server" })
  │    ├─ Opens browser to OAuth authorization URL
  │    ├─ User approves
  │    └─ Browser redirects to localhost:callbackPort/callback
  │
  ├─ OAuth callback received:
  │    ├─ Exchange code for access token
  │    ├─ Store tokens in keychain (macOS) or settings
  │    └─ Set client state: 'connected'
  │
  └─ Subsequent requests:
       ├─ checkAndRefreshOAuthTokenIfNeeded() before each call
       └─ Auto-refresh when token nearing expiry
```

**XAA (Cross-App Access / SEP-990)**: For enterprise single sign-on across apps:
```json
{
  "type": "sse",
  "url": "https://corp-mcp.example.com/sse",
  "oauth": {
    "clientId": "my-client-id",
    "xaa": true
  }
}
```
Requires prior setup: `claude mcp xaa setup` to configure the IdP.

---

## MCP Client Lifecycle Hooks

The client-level hooks that run during MCP operations (in addition to tool-level hooks):

**At connection**: `MCPConnectionManager` emits state changes to React state
**At tool call**: `PreToolUse` / `PostToolUse` hooks apply to `mcp__server__tool` pattern
**At elicitation**: `Elicitation` / `ElicitationResult` hooks
**At OAuth**: `PermissionRequest` hooks (if the auth prompt is treated as a permission)

---

## Environment Variables Affecting MCP

| Variable | Effect |
|----------|--------|
| `MCP_TOOL_TIMEOUT` | Override tool call timeout in ms (default: ~100M ms) |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Skip official registry fetch |
| `MCP_CLIENT_SECRET` | OAuth client secret (used by `claude mcp add --client-secret`) |
| `CLAUDE_MCP_DEBUG` | Enable verbose MCP debug logging |

---

## Summary: All MCP Registration Patterns

```
╔══════════════════════════════════════════════════════════════╗
║  SCOPE         PATH                    FORMAT               ║
╠══════════════════════════════════════════════════════════════╣
║  enterprise    managed-mcp.json        { mcpServers: {} }   ║
║  user          ~/.claude/settings.json  mcpServers key      ║
║  project       .mcp.json               { mcpServers: {} }   ║
║  local         .claude/settings.local.json  mcpServers key  ║
║  plugin        manifest.json           4 formats (see doc)  ║
║  agent         frontmatter             string or {name:cfg} ║
║  dynamic       --mcp-config flag       { mcpServers: {} }   ║
╚══════════════════════════════════════════════════════════════╝

Merge precedence: enterprise > local > project > user > plugin
Plugin servers: namespaced "plugin:name:server", deduplicated
Agent servers: session-lifetime only, optional reference to existing
```
