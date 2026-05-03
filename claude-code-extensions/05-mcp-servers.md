# MCP Servers

MCP (Model Context Protocol) servers extend Claude Code with external tools, resources, and skills. Each server runs as a separate process and communicates over a defined protocol. Claude Code wraps MCP tools into its native `Tool` interface so they behave identically to built-in tools.

---

## Configuration

MCP servers are configured in settings.json at any scope level:

```json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "node",
      "args": ["/path/to/server/index.js"],
      "env": {
        "API_KEY": "my-key",
        "DATABASE_URL": "${DATABASE_URL}"
      }
    }
  }
}
```

The server name (`my-server`) becomes a namespace prefix for its tools: `mcp__my-server__tool-name`.

---

## Transport Types

### stdio (most common)
```json
{
  "type": "stdio",
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-filesystem", "/allowed/path"],
  "env": {
    "SOME_VAR": "value"
  }
}
```

The server process reads MCP protocol messages from stdin and writes responses to stdout.

### SSE (Server-Sent Events)
```json
{
  "type": "sse",
  "url": "https://my-mcp-server.example.com/sse",
  "headers": {
    "Authorization": "Bearer ${SSE_TOKEN}"
  },
  "oauth": {
    "clientId": "my-client-id",
    "clientSecret": "${OAUTH_SECRET}",
    "tokenEndpoint": "https://auth.example.com/token",
    "scopes": ["read", "write"]
  }
}
```

### HTTP (streamable HTTP, newer spec)
```json
{
  "type": "http",
  "url": "https://my-mcp-server.example.com/mcp",
  "headers": {
    "X-API-Key": "${API_KEY}"
  }
}
```

### WebSocket
```json
{
  "type": "ws",
  "url": "wss://my-mcp-server.example.com/ws"
}
```

### SDK (in-process, advanced)
```json
{
  "type": "sdk",
  "instance": { ... }
}
```
Used for embedding MCP servers directly in code without a subprocess. Primarily for SDK entrypoints.

### SSE-IDE
```json
{
  "type": "sse-ide",
  "url": "http://localhost:PORT/sse"
}
```
Special transport for IDE extensions (VS Code, JetBrains bridge).

---

## Full McpServerConfig Schema

```typescript
type McpServerConfig =
  | {
      type: 'stdio'
      command: string          // Executable name or path
      args?: string[]          // Command arguments
      env?: Record<string, string>  // Environment variables (support ${VAR} expansion)
    }
  | {
      type: 'sse' | 'http'
      url: string              // Server URL
      headers?: Record<string, string>  // HTTP headers
      oauth?: OAuthConfig      // OAuth2 configuration
    }
  | {
      type: 'ws'
      url: string              // WebSocket URL
    }
  | {
      type: 'sdk'
      instance: McpSdkServerConfig
    }
```

---

## MCP Resources

MCP servers can expose resources (files, data, etc.) in addition to tools:

```typescript
// Resources are accessible via ListMcpResourcesTool and ReadMcpResourceTool
// Example resource URI format: mcp://server-name/resource-path
```

Built-in tools for MCP resource access:
- `ListMcpResourcesTool` — list available resources from connected MCP servers
- `ReadMcpResourceTool` — read a specific resource by URI

---

## MCP-Based Skills

MCP servers can expose **skills** (prompt templates) in addition to tools. These appear in Claude Code's skill registry and can be invoked with `/server-name:skill-name`.

Skill discovery happens via `mcpSkillBuilders.ts` which maps MCP prompt endpoints to `PromptCommand` objects using the same `createSkillCommand` function as file-based skills.

```typescript
// In your MCP server, expose prompts:
server.addPrompt({
  name: "code-review",
  description: "Perform a thorough code review",
  arguments: [
    { name: "filepath", description: "File to review", required: true }
  ],
  load: async ({ filepath }) => ({
    messages: [
      {
        role: "user",
        content: { type: "text", text: `Please review ${filepath}...` }
      }
    ]
  })
})
```

**Security note**: MCP skills never execute inline shell injection (`!`backtick`` syntax) even if their content includes it. Enforced in `loadSkillsDir.ts`.

---

## Tool Naming Convention

MCP tools are namespaced to avoid collisions:
```
mcp__<server-name>__<tool-name>
```

Examples:
- `mcp__slack__send-message`
- `mcp__github__create-pr`
- `mcp__filesystem__read-file`

Permission rules use this naming:
```json
{
  "permissions": {
    "allow": [
      "mcp__slack__*",         // All slack tools
      "mcp__github__read-*"    // Only read tools from github
    ]
  }
}
```

---

## OAuth for MCP Servers

SSE and HTTP servers support OAuth2:

```json
{
  "type": "sse",
  "url": "https://api.example.com/mcp",
  "oauth": {
    "clientId": "my-oauth-client",
    "clientSecret": "${OAUTH_SECRET}",
    "tokenEndpoint": "https://auth.example.com/oauth/token",
    "authorizationEndpoint": "https://auth.example.com/oauth/authorize",
    "scopes": ["read", "write"],
    "redirectUri": "http://localhost:PORT/callback"
  }
}
```

Claude Code handles the OAuth flow including:
- Authorization code flow with PKCE
- Token storage (persisted per server)
- Token refresh on expiry

---

## XAA (Cross-App Access)

Claude Code supports XAA for MCP servers that need access across applications. This is an Anthropic-internal protocol for trusted MCP server authentication. Configuration is server-specific and requires server-side support.

---

## Configuring MCP in Agent Definitions

Agents can reference MCP servers by name (must already be configured) or define inline servers:

```markdown
---
name: github-agent
description: "Use for GitHub PR and issue operations"
mcpServers:
  - github                 # References existing mcpServers.github config
  - slack                  # References existing mcpServers.slack config
requiredMcpServers:
  - github                 # Agent hidden if 'github' server is not configured
---
```

Or inline server definition:
```yaml
mcpServers:
  - my-api:
      type: stdio
      command: npx
      args: ["-y", "my-api-mcp-server"]
      env:
        API_KEY: "${MY_API_KEY}"
```

---

## Writing an MCP Server

### Node.js / TypeScript

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'
import { z } from 'zod'

const server = new McpServer({
  name: 'my-server',
  version: '1.0.0'
})

// Add a tool
server.addTool({
  name: 'query-database',
  description: 'Query the application database',
  inputSchema: z.object({
    sql: z.string().describe('SQL query to execute'),
    limit: z.number().optional().describe('Max rows to return')
  }),
  handler: async ({ sql, limit }) => {
    const results = await db.query(sql, { limit: limit ?? 100 })
    return {
      content: [
        { type: 'text', text: JSON.stringify(results, null, 2) }
      ]
    }
  }
})

// Add a resource
server.addResource({
  uri: 'mcp://my-server/schema',
  name: 'Database Schema',
  mimeType: 'application/json',
  load: async () => ({
    contents: [
      { type: 'text', text: JSON.stringify(await db.getSchema()) }
    ]
  })
})

// Add a skill/prompt
server.addPrompt({
  name: 'optimize-query',
  description: 'Analyze and optimize a SQL query for performance',
  arguments: [
    { name: 'query', description: 'SQL query to optimize', required: true }
  ],
  load: async ({ query }) => ({
    messages: [
      {
        role: 'user',
        content: {
          type: 'text',
          text: `Analyze this SQL query for performance issues and suggest optimizations:\n\n\`\`\`sql\n${query}\n\`\`\``
        }
      }
    ]
  })
})

// Start server
const transport = new StdioServerTransport()
await server.connect(transport)
```

Register in settings.json:
```json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "node",
      "args": ["/path/to/my-server/dist/index.js"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    }
  }
}
```

### Python

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

app = Server("my-python-server")

@app.list_tools()
async def list_tools():
    return [
        types.Tool(
            name="analyze-logs",
            description="Analyze application log files",
            inputSchema={
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "Path to log file"},
                    "level": {"type": "string", "enum": ["error", "warn", "info"]}
                },
                "required": ["path"]
            }
        )
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "analyze-logs":
        path = arguments["path"]
        level = arguments.get("level", "error")
        # ... analyze logs
        return [types.TextContent(type="text", text=f"Analysis: {results}")]

async def main():
    async with stdio_server() as streams:
        await app.run(*streams, app.create_initialization_options())

import asyncio
asyncio.run(main())
```

---

## MCP Tool Wrapping

When Claude Code loads MCP tools, they're wrapped in the native `Tool` interface. The wrapper handles:

1. **Permission checking**: Uses `mcp__server-name__tool-name` pattern for rule matching
2. **Input validation**: MCP tool's inputSchema is converted to Zod
3. **Error handling**: MCP errors wrapped in standard error format
4. **Deduplication**: Same tool from multiple servers gets deduplicated
5. **Tool search**: MCP tools are searchable via `ToolSearch`

The wrapping code is in `src/services/mcp/client.ts` and `tools.ts::assembleToolPool()`.

---

## MCP Debugging

```bash
# List connected MCP servers and their tools
# (in Claude Code session)
> What MCP servers are connected and what tools do they provide?

# Test MCP server directly
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | node /path/to/server.js

# Enable MCP debug logging
CLAUDE_MCP_DEBUG=1 claude

# Check server startup errors
claude --debug 2>&1 | grep -i mcp
```

Common issues:
- Server command not found: check `command` path in config, use full path or npx
- Auth failures: verify OAuth config, check token storage in `~/.claude/`
- Tool name conflicts: two servers with same tool name → one is deduplicated
- Startup timeout: server takes too long to start → increase timeout or optimize startup
