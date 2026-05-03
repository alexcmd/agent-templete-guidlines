# External Debugger Plugin for Claude Code — Zero Source Changes

> §1–10: Building a full-featured debugger plugin/agent/skill that attaches to
> Claude Code's tool execution lifecycle using only external configuration surfaces:
> MCP servers, hooks, and skills. No modification of Claude Code source required.
>
> Source-verified against `/home/deck/Projects/claude-code/src` (2026-05-03).

---

## §1 — Integration Surfaces Overview

Claude Code exposes three external integration surfaces that together cover the
full debug lifecycle:

```
┌──────────────────────────────────────────────────────────────────┐
│  MCP Server (stdio/sse/http/ws)                                  │
│  → Expose debugger operations as Claude tools                    │
│  → Claude invokes: attach_debugger, set_breakpoint, inspect_var  │
├──────────────────────────────────────────────────────────────────┤
│  Hooks (PreToolUse / PostToolUse / PostToolUseFailure)           │
│  → Intercept every tool call before and after execution          │
│  → Receive: tool_name, tool_input, tool_response, session_id     │
│  → Can: block execution, modify input, log, trace, diff          │
├──────────────────────────────────────────────────────────────────┤
│  Skills (.claude/skills/SKILL.md)                                │
│  → Slash-command triggered: /debug-session, /trace-run           │
│  → Can call MCP tools, run shell, set allowed-tools allowlist    │
└──────────────────────────────────────────────────────────────────┘
```

**What you can do without source changes:**

| Capability | Surface | Notes |
|---|---|---|
| Launch debugger on a file | MCP tool | Agent calls your `launch_debug` tool |
| Set/clear breakpoints | MCP tool | Stored in your server's state |
| Inspect variables at breakpoint | MCP tool | Reads DAP response, returns to Claude |
| Intercept every Bash/Write call | Hooks (PreToolUse) | Full tool_input JSON |
| Block a tool call | Hook → `"decision":"deny"` | Synchronous hooks only |
| Modify tool input before execution | Hook → `"updatedInput"` | Synchronous hooks only |
| Post-execution diff/audit | Hooks (PostToolUse) | Has tool_response too |
| Trigger debugging via slash command | Skill | User types `/attach-debugger` |
| Execution tracer / audit log | HTTP hook → webhook server | Logs all tool calls |

**What you cannot do without source changes:**

- Pause mid-tool execution (hooks fire before/after, not during)
- Step through individual lines inside a running process from Claude's loop
- Access internal ToolUseContext fields beyond what hooks expose
- Inject tool results into the ongoing message history directly

---

## §2 — Architecture: Recommended Hybrid Pattern

```
┌────────────────────────────────────────────────────────────────┐
│  Claude Code                                                   │
│                                                                │
│  User: "debug main.py"                                         │
│    ↓                                                           │
│  Claude → calls mcp__debugger__launch_debug (MCP)             │
│                        │                                       │
│             ┌──────────▼──────────────┐                       │
│             │  Your MCP Server        │                       │
│             │  (node debugger.js)     │                       │
│             │  ├─ launch_debug        │                       │
│             │  ├─ set_breakpoint      │                       │
│             │  ├─ get_state           │                       │
│             │  └─ evaluate_expr       │                       │
│             └──────────┬──────────────┘                       │
│                        │ spawns                                │
│             ┌──────────▼──────────────┐                       │
│             │  DAP Adapter Process    │                       │
│             │  (debugpy / dlv / lldb) │                       │
│             └─────────────────────────┘                       │
│                                                                │
│  PreToolUse hook → HTTP POST to debugger webhook               │
│    payload: {tool_name, tool_input, session_id}                │
│    response: {decision: "allow"} or block with reason          │
│                                                                │
│  PostToolUse hook → HTTP POST to debugger webhook              │
│    payload: {tool_name, tool_input, tool_response}             │
│    (async: true — non-blocking audit log)                      │
└────────────────────────────────────────────────────────────────┘
```

---

## §3 — MCP Server: Exposing Debug Operations as Claude Tools

The MCP server is the **primary** integration point. Claude Code registers it in
`settings.json` and Claude can invoke its tools like any built-in tool.

### Package Setup

```bash
npm init -y
npm install @modelcontextprotocol/sdk @vscode/debugadapter @vscode/debugprotocol zod
```

### MCP Server Implementation

```typescript
// debugger-mcp/src/server.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  ListToolsRequestSchema,
  CallToolRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';
import { z } from 'zod';
import { DapClient } from './dap-client.js';
import { SessionManager } from './session-manager.js';

const sessions = new SessionManager();

const server = new Server(
  { name: 'claude-debugger', version: '1.0.0' },
  { capabilities: { tools: {} } }
);

// ── Tool registry ────────────────────────────────────────────────

const TOOLS = [
  {
    name: 'launch_debug',
    description: 'Launch a debug session for a program. Returns session_id.',
    inputSchema: {
      type: 'object',
      properties: {
        program:  { type: 'string', description: 'Absolute path to program' },
        args:     { type: 'array', items: { type: 'string' } },
        cwd:      { type: 'string' },
        language: { type: 'string', enum: ['python', 'node', 'go', 'rust', 'c'] },
      },
      required: ['program', 'language'],
    },
  },
  {
    name: 'set_breakpoint',
    description: 'Set a line breakpoint. Returns verified status.',
    inputSchema: {
      type: 'object',
      properties: {
        session_id: { type: 'string' },
        file:       { type: 'string' },
        line:       { type: 'number' },
        condition:  { type: 'string', description: 'Optional conditional expression' },
      },
      required: ['session_id', 'file', 'line'],
    },
  },
  {
    name: 'debug_continue',
    description: 'Continue execution until next breakpoint or program end.',
    inputSchema: {
      type: 'object',
      properties: { session_id: { type: 'string' } },
      required: ['session_id'],
    },
  },
  {
    name: 'debug_step',
    description: 'Step over/into/out of current line.',
    inputSchema: {
      type: 'object',
      properties: {
        session_id: { type: 'string' },
        kind: { type: 'string', enum: ['over', 'into', 'out'] },
      },
      required: ['session_id', 'kind'],
    },
  },
  {
    name: 'get_debug_state',
    description: 'Get current stack frames, scopes, and local variables.',
    inputSchema: {
      type: 'object',
      properties: { session_id: { type: 'string' } },
      required: ['session_id'],
    },
  },
  {
    name: 'evaluate_expression',
    description: 'Evaluate an expression in the current debug frame.',
    inputSchema: {
      type: 'object',
      properties: {
        session_id:  { type: 'string' },
        expression:  { type: 'string' },
        frame_index: { type: 'number', default: 0 },
      },
      required: ['session_id', 'expression'],
    },
  },
  {
    name: 'stop_debug',
    description: 'Terminate the debug session.',
    inputSchema: {
      type: 'object',
      properties: { session_id: { type: 'string' } },
      required: ['session_id'],
    },
  },
];

server.setRequestHandler(ListToolsRequestSchema, async () => ({ tools: TOOLS }));

// ── Tool dispatcher ──────────────────────────────────────────────

server.setRequestHandler(CallToolRequestSchema, async ({ params }) => {
  const { name, arguments: args = {} } = params;

  try {
    switch (name) {
      case 'launch_debug': {
        const adapter = adapterConfig(args.language as string);
        const client = new DapClient(adapter.bin, adapter.args);
        await client.start();
        await client.launch({
          program: args.program as string,
          args: (args.args as string[]) ?? [],
          cwd: (args.cwd as string) ?? process.cwd(),
          stopOnEntry: false,
        });
        const sessionId = sessions.register(client);
        return { content: [{ type: 'text', text: `Debug session started. session_id=${sessionId}` }] };
      }

      case 'set_breakpoint': {
        const client = sessions.get(args.session_id as string);
        const bp = await client.setBreakpoints(args.file as string, [{
          line: args.line as number,
          condition: args.condition as string | undefined,
        }]);
        const verified = bp.breakpoints.filter(b => b.verified).length;
        return { content: [{ type: 'text', text: `Breakpoints set: ${verified} verified` }] };
      }

      case 'debug_continue': {
        const client = sessions.get(args.session_id as string);
        const stopped = await client.continueUntilStop();
        return { content: [{ type: 'text', text: formatStopReason(stopped) }] };
      }

      case 'debug_step': {
        const client = sessions.get(args.session_id as string);
        const cmd = { over: 'next', into: 'stepIn', out: 'stepOut' }[args.kind as string]!;
        const stopped = await client.step(cmd as any);
        return { content: [{ type: 'text', text: formatStopReason(stopped) }] };
      }

      case 'get_debug_state': {
        const client = sessions.get(args.session_id as string);
        const state = await client.getFullState();
        return { content: [{ type: 'text', text: JSON.stringify(state, null, 2) }] };
      }

      case 'evaluate_expression': {
        const client = sessions.get(args.session_id as string);
        const result = await client.evaluate(
          args.expression as string,
          args.frame_index as number ?? 0,
        );
        return { content: [{ type: 'text', text: result }] };
      }

      case 'stop_debug': {
        const client = sessions.get(args.session_id as string);
        await client.disconnect();
        sessions.remove(args.session_id as string);
        return { content: [{ type: 'text', text: 'Debug session terminated.' }] };
      }

      default:
        return { content: [{ type: 'text', text: `Unknown tool: ${name}` }], isError: true };
    }
  } catch (e) {
    return { content: [{ type: 'text', text: `Error: ${(e as Error).message}` }], isError: true };
  }
});

// ── Adapter configs ──────────────────────────────────────────────

function adapterConfig(language: string): { bin: string; args: string[] } {
  const adapters: Record<string, { bin: string; args: string[] }> = {
    python:     { bin: 'python',     args: ['-m', 'debugpy.adapter'] },
    node:       { bin: 'node',       args: ['--inspect-brk=0'] },
    typescript: { bin: 'node',       args: ['--inspect-brk=0', '-r', 'ts-node/register'] },
    go:         { bin: 'dlv',        args: ['dap', '--listen=:0'] },
    rust:       { bin: 'lldb-vscode', args: [] },
    c:          { bin: 'lldb-vscode', args: [] },
  };
  return adapters[language] ?? adapters['node'];
}

function formatStopReason(event: any): string {
  if (!event) return 'Program continued (no stop event received).';
  return `Stopped: reason=${event.reason}, thread=${event.threadId}` +
    (event.description ? `, description=${event.description}` : '');
}

// ── Entry point ──────────────────────────────────────────────────

const transport = new StdioServerTransport();
await server.connect(transport);
```

### Session Manager

```typescript
// debugger-mcp/src/session-manager.ts
import { randomUUID } from 'crypto';
import { DapClient } from './dap-client.js';

export class SessionManager {
  private sessions = new Map<string, DapClient>();

  register(client: DapClient): string {
    const id = randomUUID();
    this.sessions.set(id, client);
    return id;
  }

  get(sessionId: string): DapClient {
    const c = this.sessions.get(sessionId);
    if (!c) throw new Error(`No debug session: ${sessionId}`);
    return c;
  }

  remove(sessionId: string): void {
    this.sessions.delete(sessionId);
  }
}
```

### Minimal DapClient (reuse §2 from `05-dap-debug-integration.md`)

```typescript
// debugger-mcp/src/dap-client.ts
// Full implementation: see 05-dap-debug-integration.md §2
// Additional helpers needed here:

export class DapClient {
  // ... (from §2) ...

  async launch(config: Record<string, unknown>): Promise<void> {
    await this.send('launch', config);
    await this.waitForEvent('initialized');
    await this.send('configurationDone', {});
  }

  async continueUntilStop(): Promise<unknown> {
    const threads = await this.send<{ threads: { id: number }[] }>('threads', {});
    const threadId = threads.threads[0]?.id ?? 1;
    await this.send('continue', { threadId });
    return this.waitForEvent('stopped');
  }

  async step(command: 'next' | 'stepIn' | 'stepOut'): Promise<unknown> {
    const threads = await this.send<{ threads: { id: number }[] }>('threads', {});
    const threadId = threads.threads[0]?.id ?? 1;
    await this.send(command, { threadId });
    return this.waitForEvent('stopped');
  }

  async setBreakpoints(file: string, bps: { line: number; condition?: string }[]) {
    return this.send<{ breakpoints: { verified: boolean; line: number }[] }>(
      'setBreakpoints',
      { source: { path: file }, breakpoints: bps },
    );
  }

  async getFullState(): Promise<object> {
    const threads = await this.send<{ threads: { id: number; name: string }[] }>('threads', {});
    const result: object[] = [];
    for (const thread of threads.threads) {
      const frames = await this.send<{ stackFrames: { id: number; name: string; source?: { path?: string }; line: number }[] }>(
        'stackTrace', { threadId: thread.id, startFrame: 0, levels: 5 },
      );
      const frameStates = await Promise.all(frames.stackFrames.map(async frame => {
        const scopes = await this.send<{ scopes: { name: string; variablesReference: number }[] }>('scopes', { frameId: frame.id });
        const variables = await Promise.all(scopes.scopes.map(async scope => {
          const vars = await this.send<{ variables: { name: string; value: string; type?: string }[] }>(
            'variables', { variablesReference: scope.variablesReference },
          );
          return { scope: scope.name, variables: vars.variables };
        }));
        return { frame: { name: frame.name, file: frame.source?.path, line: frame.line }, variables };
      }));
      result.push({ thread: thread.name, frames: frameStates });
    }
    return result;
  }

  async evaluate(expression: string, frameIndex: number): Promise<string> {
    const threads = await this.send<{ threads: { id: number }[] }>('threads', {});
    const threadId = threads.threads[0]?.id ?? 1;
    const frames = await this.send<{ stackFrames: { id: number }[] }>(
      'stackTrace', { threadId, startFrame: 0, levels: frameIndex + 1 },
    );
    const frameId = frames.stackFrames[frameIndex]?.id ?? 0;
    const result = await this.send<{ result: string; type?: string }>(
      'evaluate', { expression, frameId, context: 'watch' },
    );
    return result.type ? `(${result.type}) ${result.result}` : result.result;
  }

  private waitForEvent(event: string): Promise<unknown> {
    return new Promise((resolve, reject) => {
      const timer = setTimeout(() => reject(new Error(`Timeout waiting for ${event}`)), 30_000);
      this.on(event, (body: unknown) => { clearTimeout(timer); resolve(body); });
    });
  }
}
```

---

## §4 — Hooks: Intercepting Tool Execution Events

Hooks let you observe (and conditionally block) every tool call without
launching an MCP server. They are the best fit for execution tracing, audit
logging, and simple pre-flight validation.

### Hook Input Payloads (source-verified)

**PreToolUse** (`services/tools/toolExecution.ts:800`):
```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "npm test", "timeout": 30 },
  "tool_use_id": "toolu_01abc…",
  "session_id": "sess_xyz…",
  "permission_mode": "default",
  "timestamp_ms": 1746300000000,
  "user_type": "external"
}
```

**PostToolUse** (`services/tools/toolExecution.ts:1483`):
```json
{
  "hook_event_name": "PostToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "npm test" },
  "tool_response": { "output": "PASS\n3 suites, 12 tests" },
  "tool_use_id": "toolu_01abc…",
  "session_id": "sess_xyz…"
}
```

**PostToolUseFailure** (`services/tools/toolExecution.ts:1700`):
```json
{
  "hook_event_name": "PostToolUseFailure",
  "tool_name": "Bash",
  "tool_input": { "command": "npm test" },
  "error": "Exit code 1",
  "is_interrupt": false,
  "tool_use_id": "toolu_01abc…",
  "session_id": "sess_xyz…"
}
```

### Hook Output (what your hook can return)

```json
{
  "decision": "allow",           // or "deny" (blocks tool), "block" (blocks all)
  "decisionReason": "string",    // shown to user when denied
  "updatedInput": { ... },       // replace tool_input with this (PreToolUse only)
  "blockingError": "message",    // stops agent loop with this error
  "additionalContexts": ["msg"], // injected into next LLM turn
  "preventContinuation": false   // stop after this tool result
}
```

### Hook Types

#### Command Hook (shell script)

```json
{
  "type": "command",
  "command": "node /path/to/trace-hook.js",
  "shell": "bash",
  "timeout": 10,
  "async": true
}
```

Your script reads hook input from **stdin** as JSON:

```javascript
// trace-hook.js
const chunks = [];
process.stdin.on('data', d => chunks.push(d));
process.stdin.on('end', () => {
  const event = JSON.parse(Buffer.concat(chunks).toString());
  appendFileSync('/tmp/claude-trace.ndjson', JSON.stringify(event) + '\n');
  // Return empty → allow; return JSON → modify behavior
  process.stdout.write(JSON.stringify({ decision: 'allow' }));
});
```

#### HTTP Hook (webhook)

```json
{
  "type": "http",
  "url": "http://localhost:9876/hook",
  "headers": { "X-Secret": "$HOOK_SECRET" },
  "allowedEnvVars": ["HOOK_SECRET"],
  "timeout": 5
}
```

Your webhook server receives a POST body with the hook JSON and should return
the hook output JSON. For fire-and-forget audit logs, use `"async": true` in
the hook config — Claude Code will not wait for your response.

#### Agent Hook (LLM-based decision)

```json
{
  "type": "agent",
  "prompt": "Analyze this tool call: $ARGUMENTS\n\nIs it safe to execute? Return JSON with decision field.",
  "model": "claude-haiku-4-5-20251001",
  "timeout": 30
}
```

### Hook Configuration in settings.json

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "http",
            "url": "http://localhost:9876/pre-tool",
            "timeout": 5
          }
        ]
      },
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "node /home/user/.claude/hooks/write-guard.js",
            "timeout": 5
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "http",
            "url": "http://localhost:9876/post-tool",
            "async": true
          }
        ]
      }
    ],
    "PostToolUseFailure": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "node /home/user/.claude/hooks/failure-logger.js",
            "async": true
          }
        ]
      }
    ]
  }
}
```

**Matcher syntax** (from `schemas/hooks.ts`):
- `"Bash"` — exact tool name
- `"Bash(git *)"` — tool name + input pattern (glob on serialized input)
- `"*"` — all tools
- `"Write|Edit"` — alternation (pipe-separated)

### Execution Tracer Webhook Server

```typescript
// hook-server/src/index.ts
import Fastify from 'fastify';
import { appendFileSync } from 'fs';
import { randomUUID } from 'crypto';

const app = Fastify({ logger: false });
const TRACE_FILE = process.env.TRACE_FILE ?? '/tmp/claude-trace.ndjson';

app.post<{ Body: Record<string, unknown> }>('/pre-tool', async (req, reply) => {
  const event = req.body;
  const traceId = randomUUID();

  // Log the call
  appendFileSync(TRACE_FILE, JSON.stringify({ traceId, ...event }) + '\n');

  // Example: block dangerous patterns
  if (event.tool_name === 'Bash') {
    const cmd = (event.tool_input as any)?.command ?? '';
    if (/\brm\s+-rf\s+\//.test(cmd)) {
      return reply.send({
        decision: 'deny',
        decisionReason: 'Blocked: rm -rf / is not allowed by debugger policy.',
      });
    }
  }

  return reply.send({ decision: 'allow' });
});

app.post('/post-tool', async (req, reply) => {
  const event = req.body as Record<string, unknown>;
  appendFileSync(TRACE_FILE, JSON.stringify({ type: 'post', ...event }) + '\n');
  return reply.send({});
});

await app.listen({ port: 9876, host: '127.0.0.1' });
console.error('Hook server listening on :9876');
```

---

## §5 — Skills: Slash-Command Triggered Debugging

Skills let users invoke your debugger with a typed command like `/attach-debugger`.
They are the best fit for operator-initiated actions (not Claude-initiated).

### Skill File Location

```
~/.claude/skills/attach-debugger/SKILL.md     ← user-global
.claude/skills/attach-debugger/SKILL.md       ← project-local (takes precedence)
```

### Example: Debug Session Skill

```markdown
---
name: Attach Debugger
description: Launch a DAP debug session for the target file and configure breakpoints
argument-hint: <file> [breakpoint_line]
arguments: [file, line]
allowed-tools:
  - mcp__claude-debugger__launch_debug
  - mcp__claude-debugger__set_breakpoint
  - mcp__claude-debugger__get_debug_state
  - mcp__claude-debugger__evaluate_expression
  - mcp__claude-debugger__stop_debug
  - Read
  - Bash
user-invocable: true
---

When invoked with `/attach-debugger <file> [line]`:

1. Launch a debug session for the file using `launch_debug`.
   Detect language from file extension: .py → python, .ts → typescript, .go → go, .rs → rust.

2. If a line number is provided, call `set_breakpoint` on that line.

3. Call `debug_continue` to run until the breakpoint is hit.

4. Call `get_debug_state` and show the current stack frames and local variables.

5. Ask the user: "Continue, step over, step into, evaluate expression, or stop?"

6. Loop until the user types "stop" or the program exits.
```

### Example: Execution Tracer Skill

```markdown
---
name: Trace Execution
description: Run a command under the execution tracer and summarize tool calls
argument-hint: <prompt describing what to do>
allowed-tools:
  - Bash
  - Read
user-invocable: true
---

When invoked with `/trace-execution <prompt>`:

1. Start the hook server if not running:
   `!pgrep -f hook-server || node ~/.claude/hooks/hook-server.js &`

2. Execute the user's prompt in a clean context.
   The hook server will log all tool calls to /tmp/claude-trace.ndjson.

3. After completion, read and summarize the trace:
   `!cat /tmp/claude-trace.ndjson`

4. Present a table: tool_name | command_summary | duration_ms | success
```

---

## §6 — Settings.json Complete Reference

Place in `~/.claude/settings.json` (user-global) or `.claude/settings.json`
(project-local, project-local takes precedence):

```json
{
  "mcpServers": {
    "claude-debugger": {
      "type": "stdio",
      "command": "node",
      "args": ["/home/user/.claude/plugins/debugger-mcp/dist/server.js"],
      "env": {
        "DEBUG_LEVEL": "verbose",
        "TRACE_FILE": "/tmp/claude-debug.ndjson"
      }
    }
  },

  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "http",
            "url": "http://localhost:9876/pre-tool",
            "headers": { "Authorization": "Bearer $HOOK_SECRET" },
            "allowedEnvVars": ["HOOK_SECRET"],
            "timeout": 5
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "http",
            "url": "http://localhost:9876/post-tool",
            "async": true
          }
        ]
      }
    ],
    "PostToolUseFailure": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "node /home/user/.claude/hooks/failure-alert.js",
            "async": true
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "node /home/user/.claude/hooks/session-start.js",
            "statusMessage": "Initializing debug plugin…"
          }
        ]
      }
    ],
    "SessionEnd": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "node /home/user/.claude/hooks/session-end.js",
            "async": true
          }
        ]
      }
    ]
  },

  "allowedHttpHookUrls": ["http://localhost:9876/*"],

  "env": {
    "HOOK_SECRET": "changeme",
    "DEBUGGER_MODE": "enabled"
  }
}
```

### MCP Server Transport Options

| type | When to use | Config fields |
|---|---|---|
| `stdio` | Local process, most common | `command`, `args`, `env` |
| `sse` | Remote HTTP server (SSE) | `url`, `headers` |
| `http` | Remote HTTP server (POST) | `url`, `headers` |
| `ws` | WebSocket server | `url`, `headers` |
| `sdk` | In-process (tests) | `module` |

### Hook Policy Keys

| Key | Default | Effect |
|---|---|---|
| `disableAllHooks` | `false` | Disables entire hook system |
| `allowManagedHooksOnly` | `false` | Only managed (policy) hooks run |
| `allowedHttpHookUrls` | `[]` | Allowed URL patterns for http hooks |
| `httpHookAllowedEnvVars` | `[]` | Env vars allowed in http hook headers |

---

## §7 — Full Project Layout

```
debugger-plugin/
├── mcp-server/
│   ├── src/
│   │   ├── server.ts          # MCP server entry (§3)
│   │   ├── dap-client.ts      # DAP protocol client (§3 + §2 of 05-dap-debug-integration.md)
│   │   └── session-manager.ts # Session registry (§3)
│   ├── package.json
│   └── tsconfig.json
│
├── hook-server/
│   ├── src/
│   │   └── index.ts           # HTTP webhook server (§4)
│   └── package.json
│
├── skills/                    # Copy to ~/.claude/skills/
│   ├── attach-debugger/
│   │   └── SKILL.md           # §5
│   └── trace-execution/
│       └── SKILL.md           # §5
│
├── hooks/                     # Copy scripts to ~/.claude/hooks/
│   ├── session-start.js       # Start hook server on session init
│   └── session-end.js         # Clean up on session end
│
└── settings-template.json     # Paste into ~/.claude/settings.json (§6)
```

### Auto-Start Hook Server via SessionStart

```javascript
// hooks/session-start.js
const { execSync } = require('child_process');
const { existsSync } = require('fs');

const SERVER_PID_FILE = '/tmp/claude-hook-server.pid';

try {
  if (existsSync(SERVER_PID_FILE)) {
    const pid = parseInt(require('fs').readFileSync(SERVER_PID_FILE, 'utf-8'));
    process.kill(pid, 0); // throws if not running
    process.exit(0);      // already running, nothing to do
  }
} catch {}

// Start server in background
const child = require('child_process').spawn(
  'node',
  ['/home/user/.claude/plugins/debugger-plugin/hook-server/dist/index.js'],
  { detached: true, stdio: 'ignore' }
);
child.unref();
require('fs').writeFileSync(SERVER_PID_FILE, String(child.pid));
process.exit(0);
```

---

## §8 — Typical Agent Debug Workflow (End-to-End)

```
User: /attach-debugger src/api.ts 142

  ↓ Skill invokes MCP tool:
Claude → mcp__claude-debugger__launch_debug {program: "src/api.ts", language: "typescript"}
  ← session_id=sess-abc123

Claude → mcp__claude-debugger__set_breakpoint {session_id, file: "src/api.ts", line: 142}
  ← 1 verified breakpoint

Claude → mcp__claude-debugger__debug_continue {session_id}
  ← Stopped: reason=breakpoint, thread=1

Claude → mcp__claude-debugger__get_debug_state {session_id}
  ← {thread: "main", frames: [{frame: {name: "handleRequest", file: "src/api.ts", line: 142},
      variables: [{scope: "Local", variables: [{name: "req", value: "{method:'GET',...}", type: "Request"}]}]}]}

Claude: "Stopped at line 142 in handleRequest(). Local variable `req.method` is 'GET'.
         What would you like to do? (continue / step over / evaluate expression / stop)"

User: evaluate req.headers['x-auth-token']

Claude → mcp__claude-debugger__evaluate_expression {session_id, expression: "req.headers['x-auth-token']"}
  ← "(string) Bearer eyJhb..."

Claude: "The auth token is present: Bearer eyJhb..."

User: stop
Claude → mcp__claude-debugger__stop_debug {session_id}
  ← "Debug session terminated."
```

**Concurrent: Hook tracer running in background**

```
[hook-server] PRE  Bash {"command":"npx ts-node src/api.ts"} → allow
[hook-server] POST Bash {"command":"npx ts-node src/api.ts"} output: "listening on :3000"
[hook-server] PRE  mcp__claude-debugger__launch_debug {...}  → allow
[hook-server] POST mcp__claude-debugger__launch_debug {...}  output: "session_id=sess-abc123"
```

---

## §9 — Limitations and Workarounds

| Limitation | Workaround |
|---|---|
| Hooks fire before/after, not mid-execution | Use hooks for audit; use MCP tool for interactive stepping |
| Cannot pause Bash mid-execution | Wrap long scripts in a watched subprocess; poll from MCP tool |
| Hook `updatedInput` only works for synchronous hooks | Do not use `async: true` if you need to modify input |
| MCP tool namespace: `mcp__{server}__{tool}` | Claude must call the tool by that name; document it in system prompt or skill |
| No direct write-back to Claude's message history | Use `additionalContexts` in hook response to inject context |
| Session ID is not predictable | Read from hook input `session_id` field at runtime |
| http hooks blocked unless URL is in `allowedHttpHookUrls` | Add `"allowedHttpHookUrls": ["http://localhost:9876/*"]` to settings |

---

## §10 — Decision Matrix

| Goal | Best Approach |
|---|---|
| Claude autonomously debugs a program | MCP server (§3) |
| Audit/log all tool calls | HTTP PostToolUse hook, async (§4) |
| Block dangerous commands | HTTP PreToolUse hook, synchronous (§4) |
| User manually starts a debug session | Skill (§5) |
| Modify tool input before execution | Command PreToolUse hook returning `updatedInput` (§4) |
| Notify external system on failure | Command PostToolUseFailure hook (§4) |
| Full DAP step/inspect loop | MCP server wrapping DapClient (§3) |
| Run on remote machine | MCP server with type: "sse" or "http" (§6) |

---

## Integration Checklist

- [ ] MCP server registered in `settings.json` under `mcpServers`
- [ ] Hook server URL added to `allowedHttpHookUrls`
- [ ] SessionStart hook auto-starts webhook server
- [ ] Skills copied to `~/.claude/skills/` or `.claude/skills/`
- [ ] MCP server tool names match skill `allowed-tools` list (`mcp__{server}__{tool}`)
- [ ] DAP adapter binary installed: `pip install debugpy` / `go install dlv` / etc.
- [ ] DapClient timeouts at 10s per request (see §6 of `05-dap-debug-integration.md`)
- [ ] Sessions cleaned up on `stop_debug` or SessionEnd hook
- [ ] Hook server handles graceful shutdown (SIGTERM from SessionEnd hook)

---

*[← 05-dap-debug-integration.md](05-dap-debug-integration.md) | [Extensions & MCP →](01-extensions-sdk-mcp.md)*
