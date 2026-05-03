# Bridge Protocol — REPL↔Web Bidirectional Sync

Claude Code maintains a persistent connection between the local REPL process and the claude.ai web session. This bridge is bidirectional: the REPL sends conversation events to the web, and the web can send tool invocations, control requests, and configuration back to the REPL.

---

## Architecture

```
Local REPL process                     claude.ai web session
─────────────────                      ─────────────────────
AppState.replBridge*                   Browser session
    │
    ├─ replBridge.ts (ReplBridgeHandle)
    │       │
    │       ├─ V1 Transport (HTTP polling)
    │       │    └─ pollConfig.ts: exponential backoff, jitter
    │       │
    │       └─ V2 Transport (WebSocket)
    │            └─ persistent connection, lower latency
    │
    └─ FlushGate
            └─ ensures ordered message delivery
               (messages queue until gate opens)
```

The transport is abstracted by `HybridTransport` which selects V1 or V2 based on negotiation with the server.

---

## ReplBridgeHandle

The live handle returned after bridge initialization. All outbound operations go through this:

```typescript
// src/bridge/replBridge.ts
export type ReplBridgeHandle = {
  bridgeSessionId: string
  environmentId: string
  sessionIngressUrl: string

  // Send conversation messages (assistant responses, tool results, etc.)
  writeMessages(messages: Message[]): void

  // Send SDK-format messages (structured output, daemon mode)
  writeSdkMessages(messages: SDKMessage[]): void

  // Send a control request to the web side (e.g. request permission)
  sendControlRequest(request: SDKControlRequest): void

  // Send a control response back to a pending web request
  sendControlResponse(response: SDKControlResponse): void

  // Cancel a pending control request
  sendControlCancelRequest(requestId: string): void

  // Signal turn completion
  sendResult(): void

  // Graceful shutdown
  teardown(): Promise<void>
}
```

---

## Bridge State Machine

```typescript
export type BridgeState = 'ready' | 'connected' | 'reconnecting' | 'failed'
```

| State | Meaning | AppState fields |
|-------|---------|----------------|
| `ready` | Bridge registered, env created, awaiting user | `replBridgeConnected=true` |
| `connected` | Web session is active (user on claude.ai) | `replBridgeSessionActive=true` |
| `reconnecting` | Transient drop, backoff in progress | `replBridgeReconnecting=true` |
| `failed` | Permanent failure or reconnects exhausted | `replBridgeError` set |

---

## AppState Bridge Fields

```typescript
// AppState fields (all prefixed replBridge*)
replBridgeEnabled: boolean        // user toggle (config/footer)
replBridgeExplicit: boolean       // true if activated via /remote-control command
replBridgeOutboundOnly: boolean   // forward events to web but reject inbound prompts
replBridgeConnected: boolean      // env registered + session created (= "Ready")
replBridgeSessionActive: boolean  // ingress WebSocket open (= user on claude.ai)
replBridgeReconnecting: boolean   // poll loop in error backoff
replBridgeConnectUrl: string      // connect URL (?bridge=envId) for "Ready" state
replBridgeSessionUrl: string      // session URL on claude.ai (set when connected)
replBridgeEnvironmentId: string   // environment ID for debugging
replBridgeSessionId: string       // session ID for debugging
replBridgeError: string           // error message when connection fails
replBridgeInitialName: string     // session name from /remote-control <name>
showRemoteCallout: boolean        // first-time remote dialog pending
```

---

## BridgeCoreParams

```typescript
// src/bridge/replBridge.ts
export type BridgeCoreParams = {
  dir: string                  // current working directory
  machineName: string          // hostname for display
  branch: string               // git branch name
  gitRepoUrl: string | null    // remote URL (null if no git)
  title: string                // session title shown on web
  baseUrl: string              // claude.ai base URL
  sessionIngressUrl: string    // URL for ingress WebSocket
  workerType: string           // BridgeWorkerType enum value
  workerVersion: string        // build version
}
```

---

## Message Flow: REPL → Web (Outbound)

```
Model response streaming
  │
  ├─ AssistantMessage → writeMessages([...])
  │       │
  │       └─ FlushGate.queue(messages)
  │               │  (gate opens after turn start is confirmed)
  │               └─ transport.send(messages)
  │
  └─ Turn complete → sendResult()
```

The `FlushGate` ensures messages are delivered in order even if the underlying transport reorders sends. Messages queue inside the gate and flush atomically when the gate opens.

---

## Command & Skill Filtering Over Bridge

Not all slash commands are safe to run when the bridge is active. Claude Code applies a three-tier filter (`src/commands.ts`):

### REMOTE_SAFE_COMMANDS
Commands safe in `--remote` mode (TUI-only, no filesystem/git side effects):
```
session, exit, clear, help, theme, color, vim, cost, usage,
copy, btw, feedback, plan, keybindings, statusline, stickers, mobile
```

### BRIDGE_SAFE_COMMANDS (explicit allowlist for local commands)
```
compact, clear, cost, summary, releaseNotes, files
```

### `isBridgeSafeCommand()` Logic
```typescript
export function isBridgeSafeCommand(cmd: Command): boolean {
  if (cmd.type === 'local-jsx') return false   // Ink UI — never safe over bridge
  if (cmd.type === 'prompt')   return true     // Skills always bridge-safe
  return BRIDGE_SAFE_COMMANDS.has(cmd)
}
```

**Skills are always bridge-safe.** A skill (type `'prompt'`) expands to a text prompt — no Ink rendering. When a remote client sends a slash command, `isBridgeSafeCommand` gates it before execution.

### Why `local-jsx` Is Blocked
`local-jsx` commands render Ink React components (model picker, diff viewer, OAuth flow). These only work attached to a local TTY and have no serializable output to send over the bridge.

---

## SDKMessage Types (Complete Union)

The full set of message types flowing across the bridge (`src/entrypoints/sdk/coreSchemas.ts`):

| Type | Subtype | Direction | Purpose |
|------|---------|-----------|---------|
| `user` | — | both | User prompt or tool result |
| `assistant` | — | REPL→Remote | Model response (streaming or complete) |
| `result` | `success`/`error` | REPL→Remote | Turn completion |
| `system` | `init` | REPL→Remote | Session initialization event |
| `system` | `compact_boundary` | REPL→Remote | Compaction happened |
| `system` | `status` | REPL→Remote | Status update (spinner, etc.) |
| `system` | `local_command` | REPL→Remote | Output from bridge-safe command |
| `stream_event` | — | REPL→Remote | Streaming partial assistant message |
| `system` | `hook_started` | REPL→Remote | Hook execution began |
| `system` | `hook_progress` | REPL→Remote | Hook in-progress update |
| `system` | `hook_response` | REPL→Remote | Hook execution result |
| `system` | `task_started` | REPL→Remote | Background task spawned |
| `system` | `task_progress` | REPL→Remote | Background task progress |
| `system` | `task_notification` | REPL→Remote | Task finished or errored |
| `system` | `auth_status` | REPL→Remote | Auth change (login/logout) |
| `system` | `rate_limit_event` | REPL→Remote | Rate limit hit |
| `system` | `files_persisted` | REPL→Remote | File writes committed |
| `system` | `session_state_changed` | REPL→Remote | Session state mutation |
| `system` | `elicitation_complete` | REPL→Remote | MCP elicitation resolved |
| `system` | `prompt_suggestion` | REPL→Remote | Inline autocomplete suggestion |
| `tool_use_summary` | — | REPL→Remote | Tool use summary (compaction boundary) |
| `api_retry` | — | REPL→Remote | API retry in progress |

**Eligibility filter** — only these forward across the bridge:
```typescript
export function isEligibleBridgeMessage(m: Message): boolean {
  if ((m.type === 'user' || m.type === 'assistant') && m.isVirtual) return false
  return (
    m.type === 'user' ||
    m.type === 'assistant' ||
    (m.type === 'system' && m.subtype === 'local_command')  // only this system subtype
  )
}
```

---

## Message Flow: Web → REPL (Inbound)

Three inbound message classes are handled differently:

### 1. Prompt Injection (SDKMessage type `user`)
```typescript
handleIngressMessage(data, recentPostedUUIDs, recentInboundUUIDs,
  onInboundMessage,       // → REPL message queue → new query turn
  onPermissionResponse,   // → clears pending permission dialog
  onControlRequest,       // → handleServerControlRequest()
)
```

### 2. SDKControlRequest (Web → REPL control)

Complete list of control request subtypes (`src/entrypoints/sdk/controlSchemas.ts`):

| Subtype | Purpose |
|---------|---------|
| `initialize` | Session handshake — send hooks, MCP servers, system prompt, agents |
| `interrupt` | Abort the currently running turn |
| `set_model` | Change the active model |
| `set_permission_mode` | Change permission enforcement mode |
| `set_max_thinking_tokens` | Extended thinking budget |
| `can_use_tool` | Permission prompt from server (async decision) |
| `mcp_status` | Query MCP server connection status |
| `get_context_usage` | Context window usage breakdown |
| `rewind_files` | Undo file changes since a specific message |
| `cancel_async_message` | Drop a queued user message |
| `seed_read_state` | Prime the file cache (avoids first-read latency) |
| `hook_callback` | Deliver hook callback result to REPL |
| `mcp_message` | Send JSON-RPC message to an MCP server |
| `mcp_set_servers` | Replace the active MCP server set |
| `reload_plugins` | Reload plugins from disk |
| `mcp_reconnect` | Reconnect a specific MCP server |
| `mcp_toggle` | Enable or disable an MCP server |
| `stop_task` | Abort a background task |
| `apply_flag_settings` | Merge a new settings layer |
| `get_settings` | Read effective resolved settings |
| `elicitation` | Request user input (MCP elicitation flow) |

**Outbound-only guard:** when `outboundOnly=true`, all subtypes except `initialize` are rejected with `OUTBOUND_ONLY_ERROR`.

### Initialize Handshake

The `initialize` control request is the first message sent by a remote client. The REPL responds with available state:

```typescript
// Remote → REPL (request)
SDKControlInitializeRequest {
  subtype: 'initialize'
  hooks?: Record<HookEvent, HookCallbackMatcher[]>
  sdkMcpServers?: string[]
  jsonSchema?: Record<string, unknown>
  systemPrompt?: string
  appendSystemPrompt?: string
  agents?: Record<string, AgentDefinition>
  promptSuggestions?: boolean
  agentProgressSummaries?: boolean
}

// REPL → Remote (response)
SDKControlInitializeResponse {
  commands: SlashCommand[]        // all registered commands with descriptions
  agents: AgentInfo[]             // available agent definitions
  output_style: string            // current output style
  available_output_styles: string[]
  models: ModelInfo[]             // available models with context windows
  account: AccountInfo            // current account (email, plan, org)
  pid?: number                    // CLI process PID (for tmux socket isolation)
  fast_mode_state?: FastModeState
}
```

Remote clients use the initialize response to populate their command/agent/model pickers without a separate API call.

### Permission Request (`can_use_tool`)

When a tool needs permission and no local handler is available, the REPL sends a control request to the remote:

```typescript
// REPL → Remote (outbound control request)
{
  subtype: 'can_use_tool',
  tool_name: string,
  input: Record<string, unknown>,
  tool_use_id: string,
  permission_suggestions?: PermissionUpdate[],
  blocked_path?: string,
  decision_reason?: string,
  title?: string,
  display_name?: string,
  agent_id?: string,
  description?: string,
}
```

Remote responds via `POST /v1/sessions/{sessionId}/events` with a `control_response`:
```typescript
{
  type: 'control_response',
  response: {
    subtype: 'success',
    request_id: string,
    response: { behavior: 'allow' | 'deny', ... }
  }
}
```

### SDKControlResponse (REPL → Web)
```typescript
{
  type: 'control_response',
  response: {
    subtype: 'success' | 'error',
    request_id: string,
    response?: Record<string, unknown>,  // success payload
    error?: string,                       // error message
    pending_permission_requests?: SDKControlRequest[],
  }
}
```

---

## Outbound-Only Mode

When `replBridgeOutboundOnly=true`, the bridge forwards events to the web but rejects all inbound prompts and control requests. This is used when the REPL is the source of truth and the web is just a viewer:

```typescript
// Only events matching these predicates are forwarded outbound
isEligibleBridgeMessage(message): boolean
```

---

## V1 vs V2 Transport

| Feature | V1 (HTTP Polling) | V2 (WebSocket) |
|---------|------------------|----------------|
| Latency | Higher (poll interval) | Lower (push) |
| Reconnection | Automatic (next poll) | Backoff + reconnect |
| Availability | Everywhere | Requires WS support |
| Config | `pollConfigDefaults.ts` | `capacityWake.ts` |

### HybridTransport Batching (`src/cli/transports/HybridTransport.ts`)

The `HybridTransport` uses WS for reads and HTTP POST for writes with strict batching:

| Parameter | Value |
|-----------|-------|
| Buffer time before flush | 100ms |
| Max messages per batch | 500 |
| Max queue depth | 100,000 |
| Write concurrency | 1 (serial) |
| Backoff range | 500ms → 8s |
| Jitter | 1s |
| Failure cap | optional `maxConsecutiveFailures` |

```typescript
new HybridTransport(
  url: URL,
  headers?: Record<string, string>,
  sessionId?: string,
  refreshHeaders?: () => Record<string, string>,  // token refresh callback
  options?: {
    maxConsecutiveFailures?: number   // replBridge sets this; 1P does not
    onBatchDropped?: (batchSize: number, failures: number) => void
  }
)
```

Non-stream writes flush buffered stream events first to preserve message ordering within a turn.

### CapacityWake (`src/bridge/capacityWake.ts`)

Used by V2 to signal the poll loop to wake early (instead of sleeping until poll interval):

```typescript
type CapacityWake = {
  signal(): CapacitySignal   // merged AbortSignal (outer loop + capacity wake)
  wake(): void               // abort current sleep, arm fresh controller
}

// Usage pattern:
const capacityWake = createCapacityWake(loopSignal)
while (!loopSignal.aborted) {
  const { signal, cleanup } = capacityWake.signal()
  await sleep(POLL_INTERVAL, signal)  // returns on outer abort OR capacity wake
  cleanup()
  // poll for work...
  // session done → capacityWake.wake()
}
```

---

## Deduplication

Inbound messages are deduplicated using a bounded UUID set:

```typescript
// src/bridge/bridgeMessaging.ts
export class BoundedUUIDSet {
  // Tracks recently-seen message UUIDs to prevent duplicate processing
  // on reconnect or poll overlap
}
```

---

## Debug and Fault Injection

```typescript
// src/bridge/bridgeDebug.ts
registerBridgeDebugHandle(handle)    // register for fault injection
injectBridgeFault(faultType, ...)    // inject artificial errors/delays
clearBridgeDebugHandle()             // unregister
```

Available in `--verbose` mode and diagnostic tooling.

---

## Reproducing in Your Agent

The bridge pattern is useful for any agent that needs a local-process ↔ remote-UI connection. Key design decisions to replicate:

### 1. FlushGate for Ordering

```python
import asyncio
from collections import deque

class FlushGate:
    """
    Queue messages until the gate opens, then flush in order.
    """
    def __init__(self):
        self._open = False
        self._queue: deque = deque()
        self._lock = asyncio.Lock()

    def queue(self, messages: list):
        async def _enqueue():
            async with self._lock:
                if self._open:
                    await self._send_immediately(messages)
                else:
                    self._queue.extend(messages)
        return asyncio.create_task(_enqueue())

    def open(self):
        async def _flush():
            async with self._lock:
                self._open = True
                while self._queue:
                    msg = self._queue.popleft()
                    await self._send_immediately([msg])
        return asyncio.create_task(_flush())

    async def _send_immediately(self, messages: list):
        raise NotImplementedError
```

### 2. Transport Abstraction

```python
from abc import ABC, abstractmethod
from enum import Enum

class BridgeState(Enum):
    READY = "ready"
    CONNECTED = "connected"
    RECONNECTING = "reconnecting"
    FAILED = "failed"

class BridgeTransport(ABC):
    @abstractmethod
    async def send_messages(self, messages: list) -> None: ...

    @abstractmethod
    async def send_control_response(self, response: dict) -> None: ...

    @abstractmethod
    async def receive(self) -> dict | None:
        """Poll or await next inbound message."""
        ...

    @abstractmethod
    async def teardown(self) -> None: ...

class PollingTransport(BridgeTransport):
    """V1: HTTP polling with exponential backoff."""
    def __init__(self, poll_url: str, session: object):
        self.poll_url = poll_url
        self.session = session
        self.min_interval = 0.5
        self.max_interval = 30.0
        self.backoff = 1.0

    async def receive(self) -> dict | None:
        try:
            response = await self.session.get(self.poll_url)
            data = await response.json()
            self.backoff = self.min_interval  # reset on success
            return data if data else None
        except Exception:
            self.backoff = min(self.backoff * 2, self.max_interval)
            await asyncio.sleep(self.backoff)
            return None

class WebSocketTransport(BridgeTransport):
    """V2: persistent WebSocket."""
    def __init__(self, ws_url: str):
        self.ws_url = ws_url
        self._ws = None

    async def connect(self):
        import websockets
        self._ws = await websockets.connect(self.ws_url)

    async def receive(self) -> dict | None:
        if not self._ws:
            return None
        import json
        msg = await self._ws.recv()
        return json.loads(msg)
```

### 3. Outbound-Only Guard

```python
class BridgeHandle:
    def __init__(self, transport: BridgeTransport, outbound_only: bool = False):
        self.transport = transport
        self.outbound_only = outbound_only
        self._gate = FlushGate()

    async def write_messages(self, messages: list):
        await self._gate.queue(messages)

    async def handle_inbound(self, message: dict) -> bool:
        """Returns True if message was accepted, False if rejected."""
        if self.outbound_only and message.get("type") == "user_prompt":
            return False
        await self._dispatch(message)
        return True

    async def _dispatch(self, message: dict):
        msg_type = message.get("type")
        if msg_type == "control_request":
            await self._handle_control(message)
        elif msg_type == "user_prompt":
            await self._inject_prompt(message)
```

### 4. Deduplication

```python
from collections import OrderedDict

class BoundedUUIDSet:
    def __init__(self, max_size: int = 1000):
        self._seen = OrderedDict()
        self.max_size = max_size

    def add_and_check(self, uuid: str) -> bool:
        """Returns True if UUID is new (not a duplicate)."""
        if uuid in self._seen:
            return False
        self._seen[uuid] = True
        if len(self._seen) > self.max_size:
            self._seen.popitem(last=False)  # evict oldest
        return True
```

---

## Plugin & Agent Injection via Initialize

Plugins and custom agents are NOT registered over the bridge message stream. Instead, they are injected via the `initialize` control request when the remote client connects:

```typescript
// Remote client sends on first connect:
{
  subtype: 'initialize',
  agents: {
    "code-reviewer": {
      name: "code-reviewer",
      description: "Reviews code for bugs and style",
      systemPrompt: "You are a senior code reviewer...",
      tools: ["read_file", "bash"],
    }
  },
  sdkMcpServers: ["sqlite", "filesystem"],
  hooks: {
    "PostToolUse": [{
      matcher: "bash",
      hookCallbackIds: ["my-hook-id"],
      timeout: 30,
    }]
  },
  appendSystemPrompt: "Always respond in JSON.",
}
```

The REPL merges these into its live session:
- `agents` → available to AgentTool, listed in initialize response
- `sdkMcpServers` → started as in-process MCP connections
- `hooks` → registered for current session only (not persisted to settings.json)
- `systemPrompt` / `appendSystemPrompt` → injected into system prompt for this session

Subsequent `reload_plugins`, `mcp_set_servers`, and `mcp_toggle` control requests let the remote client mutate these registrations mid-session.

---

## Related Files

| File | Role |
|------|------|
| `src/bridge/replBridge.ts` | Main bridge — `ReplBridgeHandle`, `BridgeCoreParams`, `BridgeState` |
| `src/bridge/bridgeMain.ts` | Bridge orchestration, session setup |
| `src/bridge/replBridgeTransport.ts` | V1/V2 transport implementations |
| `src/bridge/bridgeMessaging.ts` | `handleIngressMessage`, `isEligibleBridgeMessage`, `BoundedUUIDSet` |
| `src/bridge/flushGate.ts` | `FlushGate` implementation |
| `src/bridge/pollConfigDefaults.ts` | V1 polling intervals and backoff |
| `src/bridge/capacityWake.ts` | V2 capacity signal handling |
| `src/bridge/bridgeDebug.ts` | Fault injection and debug handles |
| `src/bridge/bridgePermissionCallbacks.ts` | Permission response event posting |
| `src/entrypoints/sdk/controlSchemas.ts` | Full Zod schemas for all `SDKControlRequest` subtypes |
| `src/entrypoints/sdk/coreSchemas.ts` | `SDKMessage` union (25+ types) |
| `src/commands.ts` | `BRIDGE_SAFE_COMMANDS`, `REMOTE_SAFE_COMMANDS`, `isBridgeSafeCommand` |

---

*[← MCP Workflow Internals](./09-mcp-workflow-internals.md) | [Remote Client Integration →](./11-remote-client-integration.md)*
