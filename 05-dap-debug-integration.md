# DAP — Debug Adapter Protocol Integration

> §1–6: DAP architecture, adapter lifecycle, breakpoints, stepping, variable inspection,
> and agent-driven debugging workflows.
>
> The Debug Adapter Protocol (DAP) is to debuggers what LSP is to language servers:
> a JSON-RPC protocol over stdio that decouples agents from debugger implementations.
> One DAP client can control GDB, LLDB, Python debugpy, Java debugger, Node.js inspector, etc.

---

## §1 — DAP Overview

```
Agent ──JSON-RPC──► DAP Adapter (per language) ──► Runtime (GDB/LLDB/Python/JVM/Node)
       ◄────────────                            ◄───

Adapters:
  Python   → debugpy (pip install debugpy)
  Node.js  → node --inspect (built-in)
  Rust     → lldb-vscode (llvm package)
  Go       → dlv (Delve)
  Java     → com.microsoft.java.debug
  C/C++    → cpptools (codelldb)
```

DAP messages flow over stdin/stdout JSON-RPC (same transport as LSP) or TCP socket.

---

## §2 — DAP Client Architecture

```typescript
interface DapClient {
  // Lifecycle
  launch(config: LaunchConfig): Promise<void>;
  attach(config: AttachConfig): Promise<void>;
  disconnect(): Promise<void>;

  // Breakpoint control
  setBreakpoints(file: string, lines: number[]): Promise<Breakpoint[]>;
  setFunctionBreakpoint(name: string): Promise<Breakpoint>;
  removeBreakpoints(file: string): Promise<void>;

  // Execution control
  continue(threadId: number): Promise<void>;
  stepOver(threadId: number): Promise<void>;
  stepInto(threadId: number): Promise<void>;
  stepOut(threadId: number): Promise<void>;
  pause(threadId: number): Promise<void>;

  // State inspection
  getStackTrace(threadId: number): Promise<StackFrame[]>;
  getScopes(frameId: number): Promise<Scope[]>;
  getVariables(variablesReference: number): Promise<Variable[]>;
  evaluate(expr: string, frameId?: number): Promise<EvaluateResult>;
}
```

### Minimal DAP Client (TypeScript)

```typescript
import { ChildProcess, spawn } from 'child_process';
import { createInterface } from 'readline';

export class DapClient {
  private proc: ChildProcess;
  private seq = 0;
  private pending = new Map<number, { resolve: Function; reject: Function }>();
  private eventHandlers = new Map<string, Function[]>();

  constructor(private adapterPath: string, private args: string[] = []) {}

  async start(): Promise<void> {
    this.proc = spawn(this.adapterPath, this.args, {
      stdio: ['pipe', 'pipe', 'pipe'],
    });

    this.proc.stdout!.on('data', this.handleData.bind(this));
    this.proc.on('exit', () => this.emit('exit', {}));

    // Initialize session
    await this.send('initialize', {
      clientID: 'agent-dap-client',
      clientName: 'Agent DAP',
      adapterID: 'unknown',
      pathFormat: 'path',
      linesStartAt1: true,
      columnsStartAt1: true,
      supportsVariableType: true,
      supportsEvaluateForHovers: true,
      supportsRunInTerminalRequest: false,
    });
  }

  // DAP uses Content-Length headers (same as LSP)
  private buffer = '';
  private handleData(chunk: Buffer): void {
    this.buffer += chunk.toString('utf-8');

    while (true) {
      const headerEnd = this.buffer.indexOf('\r\n\r\n');
      if (headerEnd === -1) break;

      const header = this.buffer.slice(0, headerEnd);
      const lenMatch = header.match(/Content-Length: (\d+)/);
      if (!lenMatch) break;

      const len = parseInt(lenMatch[1]);
      const msgStart = headerEnd + 4;
      if (this.buffer.length < msgStart + len) break;

      const msgStr = this.buffer.slice(msgStart, msgStart + len);
      this.buffer = this.buffer.slice(msgStart + len);
      this.handleMessage(JSON.parse(msgStr));
    }
  }

  private handleMessage(msg: DapMessage): void {
    if (msg.type === 'response') {
      const pending = this.pending.get(msg.request_seq!);
      if (!pending) return;
      this.pending.delete(msg.request_seq!);
      if (msg.success) pending.resolve(msg.body);
      else pending.reject(new Error(msg.message ?? 'DAP request failed'));
    } else if (msg.type === 'event') {
      this.emit(msg.event!, msg.body);
    }
  }

  async send<T = unknown>(command: string, args?: unknown): Promise<T> {
    const seq = ++this.seq;
    const msg = { type: 'request', seq, command, arguments: args };
    const body = JSON.stringify(msg);
    const header = `Content-Length: ${Buffer.byteLength(body)}\r\n\r\n`;

    this.proc.stdin!.write(header + body);

    return new Promise<T>((resolve, reject) => {
      this.pending.set(seq, { resolve, reject });
      // Timeout: DAP requests should respond in <10s
      setTimeout(() => {
        if (this.pending.has(seq)) {
          this.pending.delete(seq);
          reject(new Error(`DAP timeout: ${command}`));
        }
      }, 10_000);
    });
  }

  on(event: string, handler: Function): void {
    this.eventHandlers.set(event, [
      ...(this.eventHandlers.get(event) ?? []),
      handler,
    ]);
  }

  private emit(event: string, body: unknown): void {
    for (const h of this.eventHandlers.get(event) ?? []) h(body);
  }

  async disconnect(): Promise<void> {
    await this.send('disconnect', { terminateDebuggee: true });
    this.proc.kill();
  }
}
```

---

## §3 — Adapter Lifecycle

### Launch vs Attach

```typescript
// Launch: start a new process under the debugger
async function launchDebugSession(
  client: DapClient,
  program: string,
  args: string[],
  cwd: string,
): Promise<void> {
  await client.send('launch', {
    type: 'node',          // adapter-specific
    request: 'launch',
    program,
    args,
    cwd,
    stopOnEntry: false,    // stop at first line
    env: {},
  });

  // Wait for 'initialized' event before setting breakpoints
  await new Promise<void>((resolve) => client.on('initialized', () => resolve()));
}

// Attach: connect to a running process
async function attachDebugSession(
  client: DapClient,
  pid: number,
): Promise<void> {
  await client.send('attach', {
    request: 'attach',
    processId: pid,
  });
  await new Promise<void>((resolve) => client.on('initialized', () => resolve()));
}
```

### Standard Initialization Sequence

```
1. Client: initialize request
2. Adapter: initialize response (capabilities)
3. Client: launch or attach request
4. Adapter: initialized event
5. Client: setBreakpoints requests (for each file)
6. Client: setExceptionBreakpoints request
7. Client: configurationDone request        ← signals "ready to run"
8. Adapter: stopped / continued events
```

---

## §4 — Breakpoints & Execution Control

```typescript
export class DebugSession {
  private breakpointsByFile = new Map<string, number[]>();

  async setBreakpoints(file: string, lines: number[]): Promise<Breakpoint[]> {
    this.breakpointsByFile.set(file, lines);
    const result = await this.client.send<{ breakpoints: Breakpoint[] }>(
      'setBreakpoints',
      {
        source: { path: file },
        breakpoints: lines.map(line => ({ line })),
      },
    );
    return result.breakpoints;
  }

  // Listen for stop events and route to the agent
  onStop(cb: (event: StoppedEvent) => void): void {
    this.client.on('stopped', (body: StoppedEvent) => {
      cb(body);
    });
  }

  async getFullState(threadId: number): Promise<DebugState> {
    const frames = await this.client.send<{ stackFrames: StackFrame[] }>(
      'stackTrace',
      { threadId, startFrame: 0, levels: 20 },
    );

    const stateByFrame: FrameState[] = [];
    for (const frame of frames.stackFrames) {
      const scopesResp = await this.client.send<{ scopes: Scope[] }>(
        'scopes',
        { frameId: frame.id },
      );
      const variables = await Promise.all(
        scopesResp.scopes.map(async (scope) => {
          const vars = await this.client.send<{ variables: Variable[] }>(
            'variables',
            { variablesReference: scope.variablesReference },
          );
          return { scope: scope.name, variables: vars.variables };
        }),
      );
      stateByFrame.push({ frame, variables });
    }

    return { threadId, frames: stateByFrame };
  }

  async evaluate(expr: string, frameId: number): Promise<string> {
    const result = await this.client.send<{ result: string; type?: string }>(
      'evaluate',
      { expression: expr, frameId, context: 'watch' },
    );
    return result.type ? `(${result.type}) ${result.result}` : result.result;
  }
}
```

---

## §5 — Agent-Driven Debugging Workflow

Expose the debugger as a single `debug` tool that encapsulates the full DAP lifecycle.

```typescript
// Tool definition
const debugTool = defineTool({
  name: 'debug',
  description: 'Control a debugger: launch, set breakpoints, step, inspect variables',
  inputSchema: z.discriminatedUnion('operation', [
    z.object({ operation: z.literal('launch'), program: z.string(), args: z.array(z.string()).optional(), cwd: z.string() }),
    z.object({ operation: z.literal('attach'), pid: z.number() }),
    z.object({ operation: z.literal('set_breakpoint'), file: z.string(), line: z.number() }),
    z.object({ operation: z.literal('remove_breakpoints'), file: z.string() }),
    z.object({ operation: z.literal('continue'), threadId: z.number().optional() }),
    z.object({ operation: z.literal('step_over'), threadId: z.number().optional() }),
    z.object({ operation: z.literal('step_into'), threadId: z.number().optional() }),
    z.object({ operation: z.literal('step_out'), threadId: z.number().optional() }),
    z.object({ operation: z.literal('state') }),
    z.object({ operation: z.literal('evaluate'), expression: z.string(), frameId: z.number().optional() }),
    z.object({ operation: z.literal('stop') }),
  ]),
  execute: async (input, ctx) => {
    const session = ctx.debugSessions.getOrCreate(ctx.sessionId);

    switch (input.operation) {
      case 'launch': {
        const adapter = adapterForLanguage(detectLanguage(input.program));
        await session.launch(adapter, input.program, input.args ?? [], input.cwd);
        return toolResult('Debug session started. Use set_breakpoint then continue.');
      }
      case 'set_breakpoint': {
        const bps = await session.setBreakpoints(input.file, [input.line]);
        const verified = bps.filter(b => b.verified).length;
        return toolResult(`Set ${verified}/${bps.length} breakpoints verified`);
      }
      case 'state': {
        const threads = await session.getThreads();
        const states = await Promise.all(threads.map(t => session.getFullState(t.id)));
        return toolResult(formatDebugState(states));
      }
      case 'evaluate': {
        const result = await session.evaluate(input.expression, input.frameId ?? 0);
        return toolResult(result);
      }
      case 'stop': {
        await session.disconnect();
        return toolResult('Debug session terminated');
      }
      default:
        return toolResult(await session.executeControl(input));
    }
  },
});

// Adapter selection by language/file extension
function adapterForLanguage(lang: string): AdapterConfig {
  const adapters: Record<string, AdapterConfig> = {
    python:     { path: 'python', args: ['-m', 'debugpy.adapter'], port: 5678 },
    javascript: { path: 'node', args: ['--inspect-brk'], protocol: 'inspector' },
    typescript: { path: 'node', args: ['--inspect-brk', '-r', 'ts-node/register'] },
    rust:       { path: 'lldb-vscode', args: [] },
    go:         { path: 'dlv', args: ['dap', '--listen', ':38697'] },
  };
  return adapters[lang] ?? adapters['javascript'];
}
```

### Typical Agent Debugging Loop

```
1. agent: debug({ operation: "launch", program: "main.py", cwd: "/repo" })
2. agent: debug({ operation: "set_breakpoint", file: "main.py", line: 42 })
3. agent: debug({ operation: "continue" })
   ← stopped event fires when breakpoint hit
4. agent: debug({ operation: "state" })   → reads stack + variables
5. agent: debug({ operation: "evaluate", expression: "user.email" })
6. agent: debug({ operation: "step_over" })
7. agent: debug({ operation: "state" })   → reads updated state
8. agent: debug({ operation: "stop" })
```

---

## §6 — Session Management & Error Handling

```typescript
class DebugSessionManager {
  private sessions = new Map<string, DebugSession>();

  getOrCreate(sessionId: string): DebugSession {
    if (!this.sessions.has(sessionId)) {
      this.sessions.set(sessionId, new DebugSession());
    }
    return this.sessions.get(sessionId)!;
  }

  async cleanup(sessionId: string): Promise<void> {
    const session = this.sessions.get(sessionId);
    if (!session) return;
    try { await session.disconnect(); } catch {}
    this.sessions.delete(sessionId);
  }
}

// Error categories for DAP
enum DapErrorKind {
  AdapterNotFound   = 'ADAPTER_NOT_FOUND',   // executable missing
  LaunchFailed      = 'LAUNCH_FAILED',        // process couldn't start
  AttachFailed      = 'ATTACH_FAILED',        // PID not found / permission
  Timeout           = 'TIMEOUT',              // request exceeded 10s
  ProtocolError     = 'PROTOCOL_ERROR',       // malformed DAP message
  RuntimeError      = 'RUNTIME_ERROR',        // debuggee crashed
  SessionNotActive  = 'SESSION_NOT_ACTIVE',   // command sent before launch
}

// Error handling in tool executor
async function withDapErrorHandling<T>(
  op: () => Promise<T>,
  fallback: string,
): Promise<T | string> {
  try {
    return await op();
  } catch (e) {
    const msg = (e as Error).message;
    if (msg.includes('ENOENT')) return `${fallback}: adapter executable not found`;
    if (msg.includes('timeout')) return `${fallback}: debugger did not respond within 10s`;
    if (msg.includes('ESRCH')) return `${fallback}: target process no longer exists`;
    return `${fallback}: ${msg}`;
  }
}
```

### Capability Negotiation

Not all adapters support all features — check capabilities from `initialize` response:

```typescript
interface DapCapabilities {
  supportsConfigurationDoneRequest?: boolean;
  supportsFunctionBreakpoints?: boolean;
  supportsConditionalBreakpoints?: boolean;
  supportsHitConditionalBreakpoints?: boolean;
  supportsEvaluateForHovers?: boolean;
  supportsSetVariable?: boolean;
  supportsGotoTargetsRequest?: boolean;
  supportsCompletionsRequest?: boolean;
  supportsRestartRequest?: boolean;
  supportsExceptionOptions?: boolean;
}

// Always check before using optional features
if (caps.supportsConditionalBreakpoints) {
  await client.send('setBreakpoints', {
    source: { path: file },
    breakpoints: [{ line, condition: 'user.role === "admin"' }],
  });
}
```

---

## Integration Checklist

- [ ] Choose DAP adapter per language (debugpy, dlv, lldb-vscode, node inspect)
- [ ] Implement Content-Length header framing (same as LSP)
- [ ] Follow init sequence: initialize → launch/attach → wait for `initialized` event → setBreakpoints → configurationDone
- [ ] Scope debug sessions to agent session ID; clean up on session end
- [ ] Check `DapCapabilities` before using optional features
- [ ] Expose as a single `debug` tool with discriminated union for operations
- [ ] Handle `stopped` events asynchronously — inject as context for the agent's next turn
- [ ] Timeout all DAP requests at 10s; return error, don't block agent loop
- [ ] Graceful disconnect with `terminateDebuggee: true` on session end
