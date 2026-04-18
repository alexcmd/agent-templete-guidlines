# Universal Agent Architecture — LSP, Plugins, SDK & MCP

> Sections: LSP Integration · Plugin System · SDK / Programmatic API · MCP Complete Reference

Part of the [Universal Agent Architecture](01-overview.md) series.

## 25. LSP Integration (Language Server Protocol)

Agents act as **LSP clients** — they spawn language servers and query them to provide
code intelligence (definitions, references, diagnostics, hover) as tool outputs. No
agent studied acts as an LSP server itself.

### 25.1 Architecture

```
Agent
  │
  ▼ spawn subprocess
Language Server (clangd, pyright, rust-analyzer, etc.)
  │ stdin/stdout JSON-RPC (Content-Length framing)
  ▼
Agent LSP Client
  ├── sends: initialize, textDocument/didOpen, textDocument/hover, ...
  └── receives: publishDiagnostics notifications → injected into context
```

**Transport:** Always stdio (subprocess). The agent spawns the language server as a child
process and communicates via stdin/stdout using JSON-RPC 2.0 with HTTP-style
`Content-Length:` framing headers.

### 25.2 Supported LSP Methods

| Category | Method | Use in agent |
|----------|--------|-------------|
| Lifecycle | `initialize` / `initialized` / `shutdown` | Session setup/teardown |
| Diagnostics | `textDocument/publishDiagnostics` (notification) | Inject errors into context |
| Navigation | `textDocument/definition` | Resolve symbol to declaration |
| Navigation | `textDocument/references` | Find all usages of a symbol |
| Navigation | `textDocument/implementation` | Find concrete implementations |
| Navigation | `textDocument/declaration` | Go to type declaration |
| Symbols | `textDocument/documentSymbol` | File-level symbol tree |
| Symbols | `workspace/symbol` | Project-wide symbol search |
| Call hierarchy | `textDocument/prepareCallHierarchy` | Find callers/callees |
| Call hierarchy | `callHierarchy/incomingCalls` | Who calls this function |
| Call hierarchy | `callHierarchy/outgoingCalls` | What this function calls |
| Hover | `textDocument/hover` | Type info, documentation |
| Completion | `textDocument/completion` | Autocomplete suggestions |
| Formatting | `textDocument/formatting` | Format a file |

### 25.3 LSP Tool Implementation

Expose LSP capabilities as a single `lsp` agent tool with an `operation` parameter:

```python
LSP_TOOL = {
    "name": "lsp",
    "description": "Query the language server for code intelligence: definitions, references, diagnostics, hover info, symbols.",
    "input_schema": {
        "type": "object",
        "properties": {
            "operation": {
                "type": "string",
                "enum": ["go_to_definition", "find_references", "hover",
                         "document_symbols", "workspace_symbols",
                         "go_to_implementation", "incoming_calls", "outgoing_calls",
                         "diagnostics", "format_document"]
            },
            "file_path": {"type": "string", "description": "Absolute path to the file"},
            "line":      {"type": "integer", "description": "0-based line number"},
            "character": {"type": "integer", "description": "0-based character offset"},
            "query":     {"type": "string",  "description": "For workspace_symbols: search query"}
        },
        "required": ["operation", "file_path"]
    }
}
```

### 25.4 JSON-RPC Client (minimal Python implementation)

```python
import json, subprocess, threading
from typing import Any

class LspClient:
    def __init__(self, command: list[str]):
        self._proc = subprocess.Popen(
            command, stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.DEVNULL
        )
        self._id = 0
        self._lock = threading.Lock()

    def _send(self, method: str, params: Any) -> Any:
        self._id += 1
        msg = {"jsonrpc": "2.0", "id": self._id, "method": method, "params": params}
        body = json.dumps(msg).encode()
        header = f"Content-Length: {len(body)}\r\n\r\n".encode()
        with self._lock:
            self._proc.stdin.write(header + body)
            self._proc.stdin.flush()
            return self._read_response()

    def _read_response(self) -> Any:
        headers = {}
        while True:
            line = self._proc.stdout.readline().decode().strip()
            if not line:
                break
            k, v = line.split(":", 1)
            headers[k.strip().lower()] = v.strip()
        length = int(headers["content-length"])
        body = self._proc.stdout.read(length)
        return json.loads(body).get("result")

    def initialize(self, root_uri: str) -> None:
        self._send("initialize", {
            "rootUri": root_uri,
            "capabilities": {"textDocument": {"hover": {"contentFormat": ["plaintext"]},
                                               "publishDiagnostics": {}}},
            "clientInfo": {"name": "agent-lsp-client", "version": "1.0"}
        })
        self._notify("initialized", {})

    def _notify(self, method: str, params: Any) -> None:
        msg = {"jsonrpc": "2.0", "method": method, "params": params}
        body = json.dumps(msg).encode()
        header = f"Content-Length: {len(body)}\r\n\r\n".encode()
        with self._lock:
            self._proc.stdin.write(header + body)
            self._proc.stdin.flush()

    def go_to_definition(self, uri: str, line: int, char: int) -> list:
        return self._send("textDocument/definition", {
            "textDocument": {"uri": uri}, "position": {"line": line, "character": char}
        }) or []

    def hover(self, uri: str, line: int, char: int) -> str:
        result = self._send("textDocument/hover", {
            "textDocument": {"uri": uri}, "position": {"line": line, "character": char}
        })
        if result and "contents" in result:
            c = result["contents"]
            return c.get("value", c) if isinstance(c, dict) else str(c)
        return ""

    def document_symbols(self, uri: str) -> list:
        return self._send("textDocument/documentSymbol", {"textDocument": {"uri": uri}}) or []

    def shutdown(self):
        self._send("shutdown", None)
        self._notify("exit", None)
        self._proc.terminate()
```

### 25.5 LSP Server Configuration (Plugin Manifest)

```json
{
  "lspServers": {
    "python": {
      "command": "pyright-langserver",
      "args": ["--stdio"],
      "languages": ["python"],
      "file_extensions": [".py", ".pyi"],
      "root_patterns": ["pyproject.toml", "setup.py", "requirements.txt"]
    },
    "rust": {
      "command": "rust-analyzer",
      "args": [],
      "languages": ["rust"],
      "file_extensions": [".rs"],
      "root_patterns": ["Cargo.toml"]
    },
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"],
      "languages": ["typescript", "javascript"],
      "file_extensions": [".ts", ".tsx", ".js", ".jsx"],
      "root_patterns": ["tsconfig.json", "package.json"]
    }
  }
}
```

### 25.6 Lazy Initialization

Start language servers on first use, not at session start. Map `file_extension → server_id`
and initialize the server the first time a file with that extension is queried:

```python
class LspManager:
    def __init__(self, configs: dict):
        self._configs = configs       # extension → config
        self._clients: dict[str, LspClient] = {}

    def get_client(self, file_path: str) -> LspClient | None:
        ext = Path(file_path).suffix
        cfg = self._configs.get(ext)
        if not cfg:
            return None
        if ext not in self._clients:
            client = LspClient(cfg["command"].split() + cfg.get("args", []))
            client.initialize(f"file://{Path(file_path).parent}")
            self._clients[ext] = client
        return self._clients[ext]
```

### 25.7 Diagnostics Injection

When a file is opened or modified, passively receive `publishDiagnostics` from the
language server and cache them. Inject into tool results or system prompt:

```python
def format_diagnostics_for_context(diagnostics: list[dict], file_path: str) -> str:
    if not diagnostics:
        return ""
    lines = [f"LSP diagnostics for {file_path}:"]
    for d in diagnostics:
        sev = {1: "ERROR", 2: "WARNING", 3: "INFO", 4: "HINT"}.get(d.get("severity", 1), "?")
        r = d["range"]["start"]
        lines.append(f"  [{sev}] line {r['line']+1}:{r['character']+1} — {d['message']}")
    return "\n".join(lines)
```

### 25.8 DAP (Debug Adapter Protocol)

**Not implemented in any agent studied.** DAP requires interactive debugger sessions
(REPL-style step/continue/inspect), which conflicts with the agent's asynchronous tool
execution model. Recommended approach if needed: expose DAP as a shell tool
(`bash: "python -m debugpy --listen 5678 script.py"`) and let the model interact via
separate `debugpy` client calls rather than implementing a full DAP client.

---

## 26. Plugin System

Plugins extend the agent with new tools, slash commands, hooks, MCP servers, LSP servers,
and system prompt fragments — without modifying the core agent code.

### 26.1 Plugin Manifest Format (`plugin.json`)

```json
{
  "name": "my-plugin",
  "version": "1.2.0",
  "description": "Adds Docker and Kubernetes tooling",
  "author": {
    "name": "Your Name",
    "email": "you@example.com",
    "url": "https://github.com/you/my-plugin"
  },
  "capabilities": ["read_only", "write", "network"],
  "commands": [
    {
      "name": "docker-status",
      "description": "Show Docker container status",
      "type": "bash",
      "command": "docker ps --format json"
    },
    {
      "name": "k8s-logs",
      "description": "Stream Kubernetes pod logs",
      "type": "prompt",
      "prompt": "Run: kubectl logs $ARGUMENTS --tail=50"
    }
  ],
  "skills": [
    {
      "name": "docker-expert",
      "description": "Docker best practices advisor",
      "systemPromptFragment": "You have deep Docker expertise. Prefer multi-stage builds. Always pin image digests."
    }
  ],
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(docker rm *)",
        "command": ".claude-plugin/hooks/confirm_docker_rm.sh",
        "blocking": true
      }
    ],
    "PostToolUse": [
      {
        "command": ".claude-plugin/hooks/audit.sh"
      }
    ]
  },
  "mcpServers": {
    "docker-mcp": {
      "command": "uvx",
      "args": ["docker-mcp-server"]
    }
  },
  "lspServers": {
    "dockerfile": {
      "command": "docker-langserver",
      "args": ["--stdio"],
      "file_extensions": [".dockerfile", "Dockerfile"]
    }
  }
}
```

### 26.2 Plugin Capability Levels

| Capability | What it allows |
|-----------|---------------|
| `read_only` | File reads, queries, diagnostics |
| `write` | File writes, edits |
| `network` | HTTP requests, MCP over network, OAuth |
| `shell` | Arbitrary shell execution via hooks |
| `agent` | Spawn sub-agents |

Plugins without a `capabilities` field are trusted unconditionally (backwards compatibility).
Always declare capabilities explicitly in new plugins.

### 26.3 Plugin Loading Flow

```
Session start
  │
  ├── 1. Scan built-in plugins (bundled with agent binary)
  ├── 2. Scan ~/.agent/plugins/ (user-installed)
  ├── 3. Scan .agent/plugins/ (project-specific)
  ├── 4. Load each plugin.json → validate manifest → check capabilities
  ├── 5. Register commands, hooks, MCP servers, LSP servers
  └── 6. Inject system prompt fragments from skills

On /reload-plugins:
  → Re-scan all directories, hot-reload without restarting session
```

**Directory structure:**
```
~/.agent/plugins/
└── my-plugin/
    ├── plugin.json          (manifest)
    ├── hooks/
    │   └── confirm_rm.sh   (hook scripts)
    └── assets/             (optional static files)

.agent/plugins/              (project-local, same structure)
```

### 26.4 Plugin Isolation Model

| Plugin component | Isolation |
|-----------------|-----------|
| Hooks (command type) | Subprocess — cannot access agent memory |
| Hooks (prompt type) | LLM call — isolated context |
| MCP servers | Subprocess — separate process, stdio/HTTP |
| LSP servers | Subprocess — separate process, stdio |
| Skills (system prompt) | In-process — injected into main system prompt |
| Commands (bash type) | Subprocess (shell) |
| Commands (prompt type) | In-process LLM call |

### 26.5 Writing a Minimal Plugin

```bash
mkdir -p ~/.agent/plugins/my-hello/hooks
cat > ~/.agent/plugins/my-hello/plugin.json << 'EOF'
{
  "name": "my-hello",
  "version": "0.1.0",
  "description": "Demo plugin",
  "capabilities": ["read_only"],
  "commands": [
    {
      "name": "hello",
      "description": "Say hello",
      "type": "bash",
      "command": "echo 'Hello from plugin!'"
    }
  ],
  "hooks": {
    "SessionStart": [
      { "command": "echo 'Plugin loaded' >> /tmp/agent-plugin.log" }
    ]
  }
}
EOF
```

User runs `/hello` → agent executes `echo 'Hello from plugin!'` and returns output.

### 26.6 Plugin vs MCP Server vs Hook (When to Use Which)

| Need | Use |
|------|-----|
| New tool calling external API | MCP server |
| Slash command running a script | Plugin command (bash type) |
| Behavior guard (block dangerous ops) | Plugin hook (PreToolUse, blocking) |
| Domain system prompt fragment | Plugin skill |
| Code intelligence for new language | Plugin LSP server |
| Audit/logging all tool calls | Plugin hook (PostToolUse) |
| Prompt-driven advisor | Plugin skill or command (prompt type) |

---

## 27. SDK / Programmatic API

An agent SDK lets you embed the agent in another application, run it headlessly in CI,
or build a custom UI. The SDK exposes the full agent loop as a programmatic API.

### 27.1 Core SDK Interface

```typescript
// TypeScript — canonical SDK interface

interface AgentSDK {
  // One-shot: runs agent to completion, returns final text
  query(params: QueryParams): AsyncIterable<SDKEvent>

  // Session management
  createSession(options: SessionOptions): Promise<SDKSession>
  resumeSession(sessionId: string, options: SessionOptions): Promise<SDKSession>
  listSessions(options?: ListOptions): Promise<SessionInfo[]>
  getSessionMessages(sessionId: string): Promise<Message[]>
  forkSession(sessionId: string, options?: ForkOptions): Promise<ForkResult>
  renameSession(sessionId: string, title: string): Promise<void>
  tagSession(sessionId: string, tag: string | null): Promise<void>

  // MCP server (expose agent tools to other agents)
  createMcpServer(options: McpServerOptions): McpServer
}

interface QueryParams {
  prompt: string | AsyncIterable<UserMessage>
  sessionId?: string               // resume existing session
  model?: string                   // override model
  maxTurns?: number
  permissionMode?: PermissionMode
  tools?: SdkToolDefinition[]      // additional custom tools
  systemPrompt?: string            // override system prompt
  onText?: (text: string) => void  // streaming text callback
  onToolCall?: (call: ToolCall) => void
  onCost?: (cost: CostUpdate) => void
}

// Event stream from query()
type SDKEvent =
  | { type: "text";     text: string }
  | { type: "tool_use"; name: string; input: Record<string, unknown>; id: string }
  | { type: "tool_result"; tool_use_id: string; content: string; is_error: boolean }
  | { type: "cost";     total_usd: number; turn_usd: number }
  | { type: "done";     final_text: string; session_id: string }
```

### 27.2 Custom Tool Registration

```typescript
function defineTool<T extends Record<string, z.ZodType>>(
  name: string,
  description: string,
  schema: T,
  handler: (args: z.infer<z.ZodObject<T>>) => Promise<string>
): SdkToolDefinition

// Example:
const searchTool = defineTool(
  "search_docs",
  "Search internal documentation",
  { query: z.string(), limit: z.number().optional() },
  async ({ query, limit = 10 }) => {
    const results = await internalSearch(query, limit)
    return JSON.stringify(results)
  }
)

const sdk = new AgentSDK({ apiKey: process.env.ANTHROPIC_API_KEY })
for await (const event of sdk.query({ prompt: "Find docs on OAuth", tools: [searchTool] })) {
  if (event.type === "text") process.stdout.write(event.text)
}
```

### 27.3 Python SDK Embedding

```python
from agent_sdk import AgentSDK, tool, PermissionMode
import asyncio

@tool("search_db", "Search the database")
async def search_db(query: str, limit: int = 10) -> str:
    results = await db.search(query, limit)
    return "\n".join(str(r) for r in results)

async def main():
    sdk = AgentSDK(
        api_key=os.getenv("ANTHROPIC_API_KEY"),
        model="claude-sonnet-4-6",
        permission_mode=PermissionMode.DEFAULT,
        tools=[search_db],
        system_prompt="You are a database assistant.",
    )

    session = await sdk.create_session()
    async for event in session.query("Find all users created last week"):
        if event.type == "text":
            print(event.text, end="", flush=True)
        elif event.type == "cost":
            pass  # track cost

asyncio.run(main())
```

### 27.4 SDK Session Lifecycle

```
create_session()   →  session_id issued, JSONL file created
  │
  ▼
session.query()    →  runs agent turn loop, streams events
  │
  ▼
session.query()    →  subsequent turns use same session_id (conversation continues)
  │
  ▼
fork_session()     →  creates branch from any past message
  │
  ▼
(session auto-archived after idle threshold)
```

### 27.5 SDK MCP Server (Expose Agent as MCP Tool Provider)

An agent can itself act as an MCP server, exposing its tools to other agents or
to an IDE that speaks MCP:

```typescript
const server = sdk.createMcpServer({
  name: "my-agent-server",
  version: "1.0.0",
  tools: [
    defineTool("agent_query", "Ask the coding agent", { prompt: z.string() },
      async ({ prompt }) => {
        let result = ""
        for await (const ev of sdk.query({ prompt }))
          if (ev.type === "text") result += ev.text
        return result
      })
  ]
})

server.listen({ transport: "stdio" })  // or { transport: "http", port: 8080 }
```

### 27.6 Headless / CI Mode

For running the agent in CI pipelines without a REPL:

```python
import subprocess, json, sys

def run_agent_headless(prompt: str, cwd: str) -> str:
    """Run agent CLI in headless mode, capture JSON output."""
    result = subprocess.run(
        ["agent", "--output-format", "json", "--max-turns", "20", "--permission-mode", "yolo"],
        input=prompt.encode(),
        capture_output=True,
        cwd=cwd,
        timeout=300
    )
    output = json.loads(result.stdout)
    return output["result"]

# Or via SDK (preferred — no subprocess overhead):
async def run_ci_check(diff: str) -> dict:
    sdk = AgentSDK(permission_mode="yolo", max_cost_usd=2.0)
    session = await sdk.create_session()
    events = []
    async for ev in session.query(f"Review this diff:\n{diff}"):
        events.append(ev)
    return {"text": next(e.text for e in reversed(events) if e.type == "done")}
```

---

## 28. MCP (Model Context Protocol) — Complete Reference

MCP is the standard protocol for connecting agents to external tool providers (databases,
APIs, file systems, code intelligence, etc.). The agent acts as an MCP **client**; external
services act as MCP **servers**.

### 28.1 MCP Architecture

```
Agent (MCP Client)
  │
  ├── stdio transport  ──► local MCP server (subprocess)
  ├── HTTP transport   ──► remote MCP server (REST endpoint)
  ├── SSE transport    ──► remote MCP server (streaming)
  └── WebSocket        ──► remote MCP server (bidirectional)

MCP Server exposes:
  ├── tools      → callable functions (agent invokes via tools/call)
  ├── resources  → file-like data (agent reads via resources/read)
  └── prompts    → prompt templates (agent fetches via prompts/get)
```

### 28.2 MCP Configuration (`.mcp.json`)

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"],
      "env": {}
    },
    "postgres": {
      "command": "uvx",
      "args": ["mcp-server-postgres"],
      "env": { "DATABASE_URL": "${DATABASE_URL}" }
    },
    "github": {
      "type": "http",
      "url": "https://api.github.com/mcp",
      "headers": { "Authorization": "Bearer ${GITHUB_TOKEN}" },
      "toolCallTimeoutMs": 30000
    },
    "internal-api": {
      "type": "http",
      "url": "https://internal.example.com/mcp",
      "oauth": {
        "clientId": "agent-mcp-client",
        "callbackPort": 7777,
        "authServerMetadataUrl": "https://auth.example.com/.well-known/oauth-authorization-server"
      }
    }
  }
}
```

Environment variable expansion: `${VAR}` and `${VAR:-default}` syntax are supported.

### 28.3 Initialization Handshake

```json
// Client → Server
{ "jsonrpc": "2.0", "id": 1, "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "clientInfo": { "name": "agent", "version": "1.0.0" },
    "capabilities": { "tools": {}, "resources": {}, "prompts": {} }
  }
}

// Server → Client
{ "jsonrpc": "2.0", "id": 1, "result": {
    "protocolVersion": "2024-11-05",
    "serverInfo": { "name": "my-server", "version": "0.1.0" },
    "capabilities": { "tools": { "listChanged": true } }
  }
}

// Client → Server (notification, no response)
{ "jsonrpc": "2.0", "method": "initialized", "params": {} }
```

### 28.4 Tool Discovery and Naming

```python
# After initialize, list all tools
tools_result = mcp_client.call("tools/list", {})

for tool in tools_result["tools"]:
    # Namespace: mcp__{server_name}__{tool_name}
    agent_tool_name = f"mcp__{server_name}__{tool['name']}"
    agent_tools.append({
        "name": agent_tool_name,
        "description": tool["description"],
        "input_schema": tool["inputSchema"]
    })
```

**Tool name normalization rules:**
- Server name: lowercase, spaces/hyphens → underscores
- Tool name: same normalization
- Separator: double underscore `__`
- Example: server `"my-database"`, tool `"query tables"` → `mcp__my_database__query_tables`

### 28.5 Tool Execution

```python
# Agent calls tool
response = mcp_client.call("tools/call", {
    "name": "query",
    "arguments": { "sql": "SELECT * FROM users LIMIT 10" }
})

# MCP server returns
# { "content": [{"type": "text", "text": "id,name\n1,Alice\n2,Bob"}], "isError": false }
```

Tool result content types:
| Type | Description |
|------|-------------|
| `text` | Plain text output |
| `image` | Base64-encoded image with MIME type |
| `resource` | URI reference to a resource |

### 28.6 Resources and Prompts

```python
# List available resources (file-like data)
resources = mcp_client.call("resources/list", {})
# → [{"uri": "db://users/schema", "name": "Users schema", "mimeType": "application/json"}]

# Read a resource
content = mcp_client.call("resources/read", {"uri": "db://users/schema"})
# → {"contents": [{"uri": "...", "mimeType": "application/json", "text": "{...}"}]}

# List prompt templates
prompts = mcp_client.call("prompts/list", {})
# → [{"name": "code-review", "description": "Review code for issues", "arguments": [...]}]

# Get a prompt
prompt = mcp_client.call("prompts/get", {
    "name": "code-review",
    "arguments": { "code": "def foo(): pass", "language": "python" }
})
# → {"messages": [{"role": "user", "content": {"type": "text", "text": "Review this Python..."}}]}
```

### 28.7 MCP over OAuth

For MCP servers requiring OAuth 2.0 authentication:

```python
# OAuth discovery
import httpx, json

async def discover_oauth_config(metadata_url: str) -> dict:
    async with httpx.AsyncClient() as client:
        r = await client.get(metadata_url)
        return r.json()
    # Returns: authorization_endpoint, token_endpoint, scopes_supported, ...

# PKCE flow (same as agent OAuth — see §02 auth guide)
# After obtaining access_token, inject as Authorization header:
mcp_client = McpClient(
    transport="http",
    url="https://api.example.com/mcp",
    headers={"Authorization": f"Bearer {access_token}"}
)
```

### 28.8 MCP Server Status Lifecycle

```
pending     → initializing (transport connecting)
connected   → healthy, tools available
needs-auth  → server responded with auth challenge (OAuth flow required)
failed      → connection or initialize error
disabled    → user or policy disabled this server
```

On `needs-auth`: pause tool calls to this server, trigger OAuth flow, refresh token,
retry initialize.

### 28.9 Writing a Minimal MCP Server (Python)

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent
import mcp.types as types

app = Server("my-tools")

@app.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="search_docs",
            description="Search internal documentation",
            inputSchema={
                "type": "object",
                "properties": {"query": {"type": "string"}},
                "required": ["query"]
            }
        )
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "search_docs":
        results = await search(arguments["query"])
        return [TextContent(type="text", text="\n".join(results))]
    raise ValueError(f"Unknown tool: {name}")

async def main():
    async with stdio_server() as (read, write):
        await app.run(read, write, app.create_initialization_options())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

### 28.10 Agent-as-MCP-Server (Bidirectional)

An agent can itself act as an MCP server, making its session accessible to IDE extensions
or orchestrator agents:

```python
# Expose agent session as MCP server
from mcp.server import Server
from mcp.server.stdio import stdio_server

agent_server = Server("coding-agent")

@agent_server.list_tools()
async def list_tools():
    return [Tool(name="ask_agent", description="Ask the coding agent",
                 inputSchema={"type":"object","properties":{"prompt":{"type":"string"}},"required":["prompt"]})]

@agent_server.call_tool()
async def call_tool(name: str, args: dict):
    if name == "ask_agent":
        result = await run_agent_query(args["prompt"])
        return [TextContent(type="text", text=result)]
```

This pattern lets an IDE that supports MCP (e.g., via an extension) talk directly to
the running agent session without a custom protocol.

---

*[← Distributed Agents](01-distributed-agents.md) | [↑ Overview](01-overview.md)*
