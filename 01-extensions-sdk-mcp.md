# Universal Agent Architecture — LSP, Plugins, SDK & MCP

> Sections: LSP Integration · Plugin System · SDK / Programmatic API · MCP Complete Reference

Part of the [Universal Agent Architecture](01-overview.md) series.

## 25. LSP Integration (Language Server Protocol)

Agents act as **LSP clients** — they spawn language servers and query them to provide
code intelligence (definitions, references, diagnostics, hover) as tool outputs.

### 25.1 Architecture

```
Agent
  │
  ▼ spawn subprocess (lazy — on first file query)
Language Server (clangd, pyright, rust-analyzer, typescript-language-server, …)
  │  stdin/stdout  JSON-RPC 2.0  Content-Length framing
  ▼
LspClient (async reader loop running concurrently)
  ├── pending: Map<id, Promise>   ← request/response correlation
  └── diagnostics: Map<uri, Diagnostic[]>  ← notification cache (1-min TTL)
```

**Transport:** Always stdio. JSON-RPC 2.0 with `Content-Length: N\r\n\r\n` framing —
identical to DAP (see `05-dap-debug-integration.md`).

**Critical:** The server sends `publishDiagnostics` **notifications** at any time —
interleaved with responses. A synchronous blocking reader will corrupt the stream.
You must run a dedicated reader loop that routes messages to pending promises or the
notification cache.

### 25.2 Supported LSP Methods

| Category | Method | Agent use |
|----------|--------|-----------|
| Lifecycle | `initialize` / `initialized` / `shutdown` / `exit` | Session setup/teardown |
| File sync | `textDocument/didOpen` / `didChange` / `didClose` | **Required before any query** |
| Diagnostics | `textDocument/publishDiagnostics` (notification) | Cache and inject into context |
| Navigation | `textDocument/definition` | Resolve symbol to declaration |
| Navigation | `textDocument/references` | Find all usages |
| Navigation | `textDocument/implementation` | Find concrete implementations |
| Symbols | `textDocument/documentSymbol` | File-level symbol tree |
| Symbols | `workspace/symbol` | Project-wide symbol search |
| Call hierarchy | `textDocument/prepareCallHierarchy` + `incomingCalls` / `outgoingCalls` | Callers/callees |
| Hover | `textDocument/hover` | Type info + documentation |
| Completion | `textDocument/completion` | Autocomplete suggestions |
| Formatting | `textDocument/formatting` | Format a whole file |
| Rename | `textDocument/rename` | Project-wide rename |

### 25.3 Async LSP Client (Python)

```python
import asyncio, json, re
from pathlib import Path
from typing import Any

RESPONSE_TIMEOUT = 10.0   # seconds per request
MAX_FILE_BYTES   = 10 * 1024 * 1024  # 10 MB — LSP servers choke on larger files

class LspClient:
    def __init__(self, command: list[str], root_uri: str):
        self._command   = command
        self._root_uri  = root_uri
        self._seq       = 0
        self._pending: dict[int, asyncio.Future] = {}
        self._diagnostics: dict[str, list[dict]] = {}   # uri → diagnostics
        self._diag_ts:    dict[str, float]        = {}   # uri → timestamp
        self._open_files: set[str]               = set()
        self._caps: dict                         = {}   # server capabilities
        self._proc: asyncio.subprocess.Process | None = None
        self._reader_task: asyncio.Task | None         = None

    async def start(self) -> None:
        self._proc = await asyncio.create_subprocess_exec(
            *self._command,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.DEVNULL,
        )
        self._reader_task = asyncio.create_task(self._reader_loop())
        result = await self._request("initialize", {
            "rootUri": self._root_uri,
            "capabilities": {
                "textDocument": {
                    "hover":            {"contentFormat": ["markdown", "plaintext"]},
                    "publishDiagnostics": {"relatedInformation": True},
                    "definition":       {"linkSupport": False},
                    "references":       {},
                    "documentSymbol":   {"hierarchicalDocumentSymbolSupport": True},
                    "completion":       {"completionItem": {"snippetSupport": False}},
                    "formatting":       {},
                    "rename":           {"prepareSupport": True},
                },
                "workspace": {"symbol": {}},
            },
            "clientInfo": {"name": "agent-lsp-client", "version": "1.0"},
        })
        self._caps = result.get("capabilities", {})
        await self._notify("initialized", {})

    # ── reader loop: routes responses to pending futures, caches notifications ──

    async def _reader_loop(self) -> None:
        assert self._proc and self._proc.stdout
        buf = b""
        while True:
            try:
                chunk = await self._proc.stdout.read(4096)
            except Exception:
                break
            if not chunk:
                break
            buf += chunk
            while True:
                m = re.match(rb"Content-Length: (\d+)\r\n\r\n", buf)
                if not m:
                    break
                length = int(m.group(1))
                end = m.end() + length
                if len(buf) < end:
                    break
                body = buf[m.end():end]
                buf  = buf[end:]
                try:
                    msg = json.loads(body)
                except json.JSONDecodeError:
                    continue
                self._dispatch(msg)

    def _dispatch(self, msg: dict) -> None:
        if "id" in msg and ("result" in msg or "error" in msg):
            # Response to a request
            fut = self._pending.pop(msg["id"], None)
            if fut and not fut.done():
                if "error" in msg:
                    fut.set_exception(LspError(msg["error"]))
                else:
                    fut.set_result(msg.get("result"))
        elif msg.get("method") == "textDocument/publishDiagnostics":
            # Notification — cache it
            params = msg.get("params", {})
            uri = params.get("uri", "")
            self._diagnostics[uri] = params.get("diagnostics", [])
            self._diag_ts[uri] = asyncio.get_event_loop().time()

    # ── send helpers ──

    async def _request(self, method: str, params: Any) -> Any:
        self._seq += 1
        seq = self._seq
        msg = {"jsonrpc": "2.0", "id": seq, "method": method, "params": params}
        fut: asyncio.Future = asyncio.get_event_loop().create_future()
        self._pending[seq] = fut
        await self._write(msg)
        return await asyncio.wait_for(fut, timeout=RESPONSE_TIMEOUT)

    async def _notify(self, method: str, params: Any) -> None:
        await self._write({"jsonrpc": "2.0", "method": method, "params": params})

    async def _write(self, msg: dict) -> None:
        assert self._proc and self._proc.stdin
        body = json.dumps(msg).encode()
        frame = f"Content-Length: {len(body)}\r\n\r\n".encode() + body
        self._proc.stdin.write(frame)
        await self._proc.stdin.drain()

    # ── file synchronisation (REQUIRED before any per-file query) ──

    async def open_file(self, path: str) -> None:
        p = Path(path)
        if p.stat().st_size > MAX_FILE_BYTES:
            raise ValueError(f"File too large for LSP: {path}")
        uri = p.as_uri()
        if uri in self._open_files:
            return
        text = p.read_text(errors="replace")
        lang = _language_id(p.suffix)
        await self._notify("textDocument/didOpen", {
            "textDocument": {"uri": uri, "languageId": lang, "version": 1, "text": text}
        })
        self._open_files.add(uri)
        # Give the server time to index the file before querying
        await asyncio.sleep(0.1)

    async def sync_file(self, path: str, text: str, version: int) -> None:
        uri = Path(path).as_uri()
        await self._notify("textDocument/didChange", {
            "textDocument": {"uri": uri, "version": version},
            "contentChanges": [{"text": text}],  # full-document sync (simpler)
        })

    async def close_file(self, path: str) -> None:
        uri = Path(path).as_uri()
        self._open_files.discard(uri)
        await self._notify("textDocument/didClose", {"textDocument": {"uri": uri}})

    # ── queries ──

    async def definition(self, path: str, line: int, char: int) -> list[dict]:
        await self.open_file(path)
        result = await self._request("textDocument/definition", {
            "textDocument": {"uri": Path(path).as_uri()},
            "position": {"line": line, "character": char},
        })
        return _as_list(result)

    async def references(self, path: str, line: int, char: int) -> list[dict]:
        await self.open_file(path)
        return await self._request("textDocument/references", {
            "textDocument": {"uri": Path(path).as_uri()},
            "position": {"line": line, "character": char},
            "context": {"includeDeclaration": True},
        }) or []

    async def hover(self, path: str, line: int, char: int) -> str:
        await self.open_file(path)
        result = await self._request("textDocument/hover", {
            "textDocument": {"uri": Path(path).as_uri()},
            "position": {"line": line, "character": char},
        })
        if not result:
            return ""
        c = result.get("contents", "")
        if isinstance(c, dict):
            return c.get("value", "")
        if isinstance(c, list):
            return "\n".join(i.get("value", i) if isinstance(i, dict) else i for i in c)
        return str(c)

    async def document_symbols(self, path: str) -> list[dict]:
        await self.open_file(path)
        return await self._request("textDocument/documentSymbol",
                                   {"textDocument": {"uri": Path(path).as_uri()}}) or []

    async def workspace_symbols(self, query: str) -> list[dict]:
        return await self._request("workspace/symbol", {"query": query}) or []

    async def get_diagnostics(self, path: str, max_age_s: float = 60.0) -> list[dict]:
        """Return cached diagnostics if fresh, else trigger a re-index by reopening."""
        uri = Path(path).as_uri()
        age = asyncio.get_event_loop().time() - self._diag_ts.get(uri, 0)
        if age > max_age_s:
            # Force server to re-analyse: close + reopen
            await self.close_file(path)
            self._open_files.discard(uri)
            await self.open_file(path)
            await asyncio.sleep(0.5)  # allow publishDiagnostics to arrive
        return self._diagnostics.get(uri, [])

    async def format(self, path: str, tab_size: int = 4, insert_spaces: bool = True) -> list[dict]:
        await self.open_file(path)
        return await self._request("textDocument/formatting", {
            "textDocument": {"uri": Path(path).as_uri()},
            "options": {"tabSize": tab_size, "insertSpaces": insert_spaces},
        }) or []

    # ── lifecycle ──

    async def shutdown(self) -> None:
        try:
            await self._request("shutdown", None)
            await self._notify("exit", None)
        except Exception:
            pass
        if self._reader_task:
            self._reader_task.cancel()
        if self._proc:
            self._proc.kill()


class LspError(Exception):
    pass

def _as_list(r: Any) -> list:
    if r is None:       return []
    if isinstance(r, list): return r
    return [r]

def _language_id(suffix: str) -> str:
    return {
        ".py": "python", ".ts": "typescript", ".tsx": "typescriptreact",
        ".js": "javascript", ".jsx": "javascriptreact",
        ".rs": "rust", ".go": "go", ".cpp": "cpp", ".c": "c",
        ".java": "java", ".cs": "csharp", ".rb": "ruby",
        ".kt": "kotlin", ".swift": "swift", ".zig": "zig",
    }.get(suffix, "plaintext")
```

### 25.4 TypeScript Async Client

```typescript
import { spawn, ChildProcess } from 'child_process';

const RESPONSE_TIMEOUT_MS = 10_000;
const MAX_FILE_BYTES = 10 * 1024 * 1024;

export class LspClient {
  private proc: ChildProcess;
  private seq = 0;
  private pending = new Map<number, { resolve: (v: unknown) => void; reject: (e: Error) => void }>();
  private diagnostics = new Map<string, LspDiagnostic[]>();
  private diagTimestamps = new Map<string, number>();
  private openFiles = new Set<string>();
  private buf = Buffer.alloc(0);
  caps: LspCapabilities = {};

  constructor(command: string[]) {
    this.proc = spawn(command[0], command.slice(1), {
      stdio: ['pipe', 'pipe', 'ignore'],
    });
    this.proc.stdout!.on('data', (chunk: Buffer) => {
      this.buf = Buffer.concat([this.buf, chunk]);
      this.drain();
    });
    this.proc.on('exit', () => {
      for (const { reject } of this.pending.values()) {
        reject(new Error('LSP server exited'));
      }
      this.pending.clear();
    });
  }

  private drain(): void {
    while (true) {
      const header = this.buf.toString('utf-8', 0, Math.min(200, this.buf.length));
      const m = header.match(/Content-Length: (\d+)\r\n\r\n/);
      if (!m) break;
      const headerEnd = this.buf.indexOf('\r\n\r\n') + 4;
      const length = parseInt(m[1]);
      if (this.buf.length < headerEnd + length) break;
      const body = this.buf.slice(headerEnd, headerEnd + length);
      this.buf = this.buf.slice(headerEnd + length);
      this.dispatch(JSON.parse(body.toString('utf-8')));
    }
  }

  private dispatch(msg: LspMessage): void {
    if ('id' in msg && ('result' in msg || 'error' in msg)) {
      const p = this.pending.get(msg.id as number);
      if (!p) return;
      this.pending.delete(msg.id as number);
      if ('error' in msg) p.reject(new Error(JSON.stringify(msg.error)));
      else p.resolve(msg.result);
    } else if (msg.method === 'textDocument/publishDiagnostics') {
      const { uri, diagnostics } = msg.params as { uri: string; diagnostics: LspDiagnostic[] };
      this.diagnostics.set(uri, diagnostics);
      this.diagTimestamps.set(uri, Date.now());
    }
  }

  async request<T = unknown>(method: string, params: unknown): Promise<T> {
    const id = ++this.seq;
    const msg = JSON.stringify({ jsonrpc: '2.0', id, method, params });
    const frame = `Content-Length: ${Buffer.byteLength(msg)}\r\n\r\n${msg}`;
    this.proc.stdin!.write(frame);

    return new Promise<T>((resolve, reject) => {
      const timer = setTimeout(() => {
        this.pending.delete(id);
        reject(new Error(`LSP timeout: ${method}`));
      }, RESPONSE_TIMEOUT_MS);

      this.pending.set(id, {
        resolve: (v) => { clearTimeout(timer); resolve(v as T); },
        reject:  (e) => { clearTimeout(timer); reject(e); },
      });
    });
  }

  private notify(method: string, params: unknown): void {
    const msg = JSON.stringify({ jsonrpc: '2.0', method, params });
    this.proc.stdin!.write(`Content-Length: ${Buffer.byteLength(msg)}\r\n\r\n${msg}`);
  }

  async initialize(rootUri: string): Promise<void> {
    const result = await this.request<{ capabilities: LspCapabilities }>('initialize', {
      rootUri,
      capabilities: {
        textDocument: {
          hover:            { contentFormat: ['markdown', 'plaintext'] },
          publishDiagnostics: { relatedInformation: true },
          definition:       { linkSupport: false },
          documentSymbol:   { hierarchicalDocumentSymbolSupport: true },
          formatting:       {},
        },
        workspace: { symbol: {} },
      },
      clientInfo: { name: 'agent-lsp-client', version: '1.0' },
    });
    this.caps = result.capabilities;
    this.notify('initialized', {});
  }

  async openFile(filePath: string): Promise<void> {
    const { size } = await import('fs/promises').then(fs => fs.stat(filePath));
    if (size > MAX_FILE_BYTES) throw new Error(`File too large for LSP: ${filePath}`);

    const uri = `file://${filePath}`;
    if (this.openFiles.has(uri)) return;

    const text = await import('fs/promises').then(fs => fs.readFile(filePath, 'utf-8'));
    const ext = filePath.slice(filePath.lastIndexOf('.'));
    this.notify('textDocument/didOpen', {
      textDocument: { uri, languageId: languageId(ext), version: 1, text },
    });
    this.openFiles.add(uri);
    await new Promise(r => setTimeout(r, 100));  // let server index
  }

  getDiagnostics(filePath: string): LspDiagnostic[] {
    return this.diagnostics.get(`file://${filePath}`) ?? [];
  }

  async shutdown(): Promise<void> {
    try { await this.request('shutdown', null); } catch {}
    this.notify('exit', null);
    this.proc.kill();
  }
}
```

### 25.5 Connection Manager (Lazy Init + Reconnect)

```typescript
interface LspServerConfig {
  command: string[];
  extensions: string[];
  rootPatterns: string[];   // detect project root by walking up for these files
}

const SERVER_CONFIGS: Record<string, LspServerConfig> = {
  python:     { command: ['pyright-langserver', '--stdio'],            extensions: ['.py', '.pyi'],               rootPatterns: ['pyproject.toml', 'setup.py', 'requirements.txt'] },
  typescript: { command: ['typescript-language-server', '--stdio'],    extensions: ['.ts', '.tsx', '.js', '.jsx'], rootPatterns: ['tsconfig.json', 'package.json'] },
  rust:       { command: ['rust-analyzer'],                            extensions: ['.rs'],                        rootPatterns: ['Cargo.toml'] },
  go:         { command: ['gopls'],                                    extensions: ['.go'],                        rootPatterns: ['go.mod'] },
  cpp:        { command: ['clangd', '--background-index'],             extensions: ['.cpp', '.c', '.h', '.hpp'],  rootPatterns: ['compile_commands.json', 'CMakeLists.txt'] },
  java:       { command: ['jdtls'],                                    extensions: ['.java'],                     rootPatterns: ['pom.xml', 'build.gradle'] },
  zig:        { command: ['zls'],                                      extensions: ['.zig'],                      rootPatterns: ['build.zig'] },
};

class LspConnectionManager {
  private clients = new Map<string, LspClient>();
  private starting = new Map<string, Promise<LspClient>>();

  // Extension → language key
  private extMap = new Map<string, string>(
    Object.entries(SERVER_CONFIGS).flatMap(([lang, cfg]) =>
      cfg.extensions.map(ext => [ext, lang])
    )
  );

  async getClient(filePath: string): Promise<LspClient | null> {
    const ext  = filePath.slice(filePath.lastIndexOf('.'));
    const lang = this.extMap.get(ext);
    if (!lang) return null;

    if (this.clients.has(lang)) return this.clients.get(lang)!;
    if (this.starting.has(lang)) return this.starting.get(lang)!;

    const p = this.startClient(lang, filePath);
    this.starting.set(lang, p);
    try {
      const client = await p;
      this.clients.set(lang, client);
      return client;
    } finally {
      this.starting.delete(lang);
    }
  }

  private async startClient(lang: string, filePath: string): Promise<LspClient> {
    const cfg = SERVER_CONFIGS[lang];
    const rootUri = `file://${this.findRoot(filePath, cfg.rootPatterns)}`;
    const client = new LspClient(cfg.command);
    await client.initialize(rootUri);
    return client;
  }

  private findRoot(filePath: string, patterns: string[]): string {
    const { existsSync } = require('fs');
    const { dirname, join } = require('path');
    let dir = dirname(filePath);
    while (dir !== dirname(dir)) {
      if (patterns.some(p => existsSync(join(dir, p)))) return dir;
      dir = dirname(dir);
    }
    return dirname(filePath);  // fallback: file's directory
  }

  // Reconnect with exponential backoff after server crash
  async reconnect(lang: string, filePath: string): Promise<LspClient | null> {
    this.clients.delete(lang);
    const BACKOFF = [1000, 2000, 4000, 8000];
    for (const delay of BACKOFF) {
      await new Promise(r => setTimeout(r, delay));
      try {
        return await this.getClient(filePath);
      } catch {}
    }
    return null;
  }

  async shutdownAll(): Promise<void> {
    await Promise.allSettled([...this.clients.values()].map(c => c.shutdown()));
    this.clients.clear();
  }
}
```

### 25.6 LSP Agent Tool

```typescript
export const lspTool = defineTool({
  name: 'lsp',
  description: 'Code intelligence via LSP: definitions, references, diagnostics, hover, symbols, formatting',
  inputSchema: z.discriminatedUnion('operation', [
    z.object({ operation: z.literal('definition'),        file: z.string(), line: z.number(), character: z.number() }),
    z.object({ operation: z.literal('references'),        file: z.string(), line: z.number(), character: z.number() }),
    z.object({ operation: z.literal('hover'),             file: z.string(), line: z.number(), character: z.number() }),
    z.object({ operation: z.literal('document_symbols'),  file: z.string() }),
    z.object({ operation: z.literal('workspace_symbols'), query: z.string() }),
    z.object({ operation: z.literal('diagnostics'),       file: z.string() }),
    z.object({ operation: z.literal('format'),            file: z.string() }),
  ]),
  execute: async (input, ctx) => {
    const client = await ctx.lspManager.getClient(input.file ?? '');
    if (!client) return toolResult(`No LSP server configured for this file type`);

    try {
      switch (input.operation) {
        case 'definition': {
          const locs = await client.definition(input.file, input.line, input.character);
          return toolResult(formatLocations(locs));
        }
        case 'hover': {
          const text = await client.hover(input.file, input.line, input.character);
          return toolResult(text || 'No hover info');
        }
        case 'diagnostics': {
          const diags = client.getDiagnostics(input.file);
          return toolResult(formatDiagnostics(diags, input.file));
        }
        // … other operations
      }
    } catch (e) {
      // Server crash → attempt reconnect
      const ext = input.file.slice(input.file.lastIndexOf('.'));
      const lang = ctx.lspManager.extMap.get(ext);
      if (lang) await ctx.lspManager.reconnect(lang, input.file);
      return toolResult(`LSP error: ${(e as Error).message}`);
    }
  },
});
```

### 25.7 Diagnostics Injection

Format diagnostics for the LLM context and auto-inject after file edits:

```typescript
function formatDiagnostics(diags: LspDiagnostic[], filePath: string): string {
  if (diags.length === 0) return `No diagnostics for ${filePath}`;
  const SEV = { 1: 'ERROR', 2: 'WARN', 3: 'INFO', 4: 'HINT' };
  const lines = diags.map(d => {
    const sev = SEV[d.severity as 1 | 2 | 3 | 4] ?? '?';
    const { line, character } = d.range.start;
    return `  [${sev}] ${line + 1}:${character + 1}  ${d.message}` +
           (d.relatedInformation?.length ? ` (see ${d.relatedInformation[0].location.uri})` : '');
  });
  return `LSP diagnostics — ${filePath}:\n${lines.join('\n')}`;
}

// Auto-inject diagnostics into the tool result after a file write
async function postWriteHook(filePath: string, ctx: AgentContext): Promise<string> {
  const client = await ctx.lspManager.getClient(filePath);
  if (!client) return '';
  // Re-sync with updated content
  const text = await fs.readFile(filePath, 'utf-8');
  await client.syncFile(filePath, text, Date.now());
  await new Promise(r => setTimeout(r, 300));  // let server reanalyze
  const diags = client.getDiagnostics(filePath).filter(d => d.severity === 1);
  if (diags.length === 0) return '';
  return '\n\n' + formatDiagnostics(diags, filePath);
}
```

### 25.8 DAP (Debug Adapter Protocol)

→ See [`05-dap-debug-integration.md`](05-dap-debug-integration.md) for a full DAP client
implementation. DAP uses **identical** Content-Length framing as LSP and exposes as a
single `debug` tool — the same pattern as the `lsp` tool above.

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
