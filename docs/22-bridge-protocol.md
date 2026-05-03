# §22 Bridge Protocol — Connecting External Apps to Claude Code

Reverse-engineered from `src/bridge/` in the Claude Code source tree.

*[← Prompt Caching Deep Dive](./21-prompt-caching.md)*

---

## 22.1 Overview

The Bridge Protocol is a **production OAuth-authenticated, long-poll + streaming** system that lets any external application (a web UI, mobile app, custom dashboard, or your own agent) control a running Claude Code session. Claude's own `claude.ai` web interface uses this exact mechanism.

**What it enables:**
- Dispatch user messages to Claude Code from a remote process
- Receive streamed assistant responses, tool calls, and results
- Send server-side control requests (change model, interrupt, grant tool permissions, inject hooks)
- Run multiple parallel Claude Code sessions from a single bridge server

**CLI entry point:**
```
claude remote-control [--spawn-mode worktree|same-dir|single-session]
```

---

## 22.2 Architecture

```
External App (your code)
       │
       │  1. Register (HTTP POST)         ← OAuth Bearer token
       ▼
 Bridge API (api.anthropic.com)
       │
       │  2. Poll for work (HTTP GET)     ← long-poll, returns WorkResponse
       ▼
 Work Item (WorkSecret)
       │
       │  3. Decode WorkSecret            ← base64url-encoded JSON
       ▼
 Spawn claude-code child process
       │
       │  4a. Connect (v1) WebSocket      ← JWT in auth header
       │  4b. Connect (v2) SSE + HTTP     ← JWT in headers
       ▼
 Session Ingress
       │  ←─── control_request (initialize, set_model, interrupt…)
       │  ────► control_response (success/error)
       │  ←─── user message
       │  ────► assistant / tool_use / tool_result / result
```

---

## 22.3 Connection Phases

### Phase 1 — Register Environment

```http
POST /v1/environments/bridge
Authorization: Bearer {oauth_token}
anthropic-version: 2023-06-01
anthropic-beta: environments-2025-11-01
x-environment-runner-version: {version}

{
  "id": "client-generated-uuid",
  "metadata": {
    "worker_type": "claude_code",
    "machine_name": "my-server",
    "branch": "main",
    "git_repo_url": "https://github.com/...",
    "max_sessions": 1,
    "spawn_mode": "single-session"
  }
}
```

Response:
```json
{
  "environment_id": "env_...",
  "environment_secret": "secret_..."
}
```

### Phase 2 — Poll for Work

```http
GET /v1/environments/{environment_id}/work
Authorization: Bearer {oauth_token}
anthropic-beta: environments-2025-11-01
```

Returns a `WorkResponse` when a user initiates a session, `null` (204) on timeout:

```typescript
type WorkResponse = {
  id: string               // work item ID
  type: 'work'
  environment_id: string
  state: string
  data: { type: 'session' | 'healthcheck'; id: string }
  secret: string           // base64url-encoded WorkSecret JSON
  created_at: string
}

type WorkSecret = {
  version: number
  session_ingress_token: string       // JWT, prefixed sk-ant-si-
  api_base_url: string                // session ingress base URL
  sources: Array<{
    type: string
    git_info?: { type: string; repo: string; ref?: string; token?: string }
  }>
  auth: Array<{ type: string; token: string }>
  claude_code_args?: Record<string, string> | null
  mcp_config?: unknown | null
  environment_variables?: Record<string, string> | null
  use_code_sessions?: boolean         // true → use CCR v2 (SSE) transport
}
```

### Phase 3 — Spawn Session

Decode `work.secret` (base64url → JSON → `WorkSecret`), then:

**v1 (WebSocket):**
```bash
claude \
  --sdk-url "wss://host/v1/session_ingress/ws/{sessionId}" \
  CLAUDE_CODE_SESSION_ACCESS_TOKEN={session_ingress_token}
```

**v2 (SSE + HTTP, when `use_code_sessions=true`):**
```bash
claude \
  --sdk-url "https://host/v1/code/sessions/{sessionId}" \
  CLAUDE_CODE_SESSION_ACCESS_TOKEN={session_ingress_token} \
  CLAUDE_CODE_USE_CODE_SESSIONS=1
```

### Phase 4 — Message Exchange

Once connected, messages flow bidirectionally as newline-delimited JSON.

**Inbound (server → session, via stdin / WebSocket):**
```typescript
type StdinMessage =
  | SDKUserMessage          // user prompt
  | SDKControlRequest       // server control
  | SDKControlResponse      // permission decisions
  | KeepAlive               // { type: 'keep_alive' }
  | UpdateEnvironmentVars   // { type: 'update_environment_variables', variables: Record<string,string> }
```

**Outbound (session → server, via stdout / WebSocket):**
```typescript
type StdoutMessage =
  | SDKMessage              // assistant / tool_use / tool_result / result / user
  | SDKControlResponse      // response to a server control_request
  | SDKControlRequest       // can_use_tool permission request to server
  | SDKControlCancelRequest // cancel open control request
  | KeepAlive
```

### Phase 5 — Teardown

1. Session exits → bridge receives `result` message
2. Bridge calls `acknowledgeWork(environmentId, workId, sessionToken)`
3. Bridge calls `archiveSession(sessionId)`
4. Bridge returns to polling (or exits in `single-session` mode)

---

## 22.4 Transport Protocols

| | v1 — Hybrid | v2 — SSE + POST |
|--|--|--|
| **Read path** | WebSocket | Server-Sent Events |
| **Write path** | HTTP POST | HTTP POST (CCRClient) |
| **URL pattern** | `wss://host/v1/session_ingress/ws/{id}` | `https://host/v1/code/sessions/{id}` |
| **Sequence resume** | None (server cursor) | SSE `Last-Event-ID` |
| **Max buffer** | 1000 msgs | 500 msgs |
| **Keep-alive** | WS ping every 10 s | SSE `:keepalive` comments |
| **Reconnect** | 1 s → 30 s exponential | 1 s → 30 s exponential |
| **Env var to force** | default | `CLAUDE_CODE_USE_CODE_SESSIONS=1` |

---

## 22.5 Control Protocol

All control messages follow a request/response envelope:

```typescript
// Server → Session
type SDKControlRequest = {
  type: 'control_request'
  request_id: string
  request: SDKControlRequestInner
}

// Session → Server
type SDKControlResponse = {
  type: 'control_response'
  response:
    | { subtype: 'success'; request_id: string; response?: Record<string, unknown> }
    | { subtype: 'error';   request_id: string; error: string;
        pending_permission_requests?: SDKControlRequest[] }
}

// Cancel an open request
type SDKControlCancelRequest = {
  type: 'control_cancel_request'
  request_id: string
}
```

### Control Request Subtypes

| Subtype | Direction | Purpose |
|---------|-----------|---------|
| `initialize` | Server→Session | Inject hooks, MCP servers, system prompt, agents |
| `interrupt` | Server→Session | Abort current turn |
| `set_model` | Server→Session | Switch model mid-session |
| `set_max_thinking_tokens` | Server→Session | Adjust extended thinking budget |
| `set_permission_mode` | Server→Session | Change tool permission mode |
| `can_use_tool` | Session→Server | Request permission for a tool call |
| `mcp_status` | Either | Query MCP server connection states |
| `get_context_usage` | Either | Get token breakdown by category |
| `hook_callback` | Server→Session | Deliver a hook callback payload |
| `mcp_message` | Server→Session | Forward JSON-RPC to an MCP server |
| `mcp_set_servers` | Server→Session | Replace dynamic MCP server list |
| `mcp_reconnect` | Server→Session | Reconnect a failed MCP server |
| `mcp_toggle` | Server→Session | Enable/disable an MCP server |
| `rewind_files` | Server→Session | Undo file changes since a message |
| `cancel_async_message` | Server→Session | Drop a queued async user message |
| `seed_read_state` | Server→Session | Pre-populate file read cache |
| `stop_task` | Server→Session | Stop a running sub-agent task |
| `apply_flag_settings` | Server→Session | Merge settings into flag layer |
| `get_settings` | Either | Return effective+per-source settings |
| `reload_plugins` | Server→Session | Reload commands/agents from disk |
| `elicitation` | Session→Server | MCP elicitation — request user input |

#### `initialize` (most important at session start)

```typescript
{
  subtype: 'initialize',
  hooks?: Record<HookEvent, SDKHookCallbackMatcher[]>,
  sdkMcpServers?: string[],
  jsonSchema?: Record<string, unknown>,
  systemPrompt?: string,
  appendSystemPrompt?: string,
  agents?: Record<string, AgentDefinition>,
  promptSuggestions?: boolean,
  agentProgressSummaries?: boolean,
}
```

Response carries available commands, models, account info, and PID:
```typescript
{
  commands: SlashCommand[],
  agents: AgentInfo[],
  output_style: string,
  available_output_styles: string[],
  models: ModelInfo[],
  account: AccountInfo,
  pid?: number,        // CLI process PID (for tmux socket isolation)
  fast_mode_state?: FastModeState,
}
```

#### `can_use_tool` (permission request, Session→Server)

```typescript
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

---

## 22.6 Authentication

### OAuth Flow (Bridge)

1. Bridge sends `Authorization: Bearer {oauth_token}` on all API calls.
2. Required headers:
   ```
   anthropic-version: 2023-06-01
   anthropic-beta: environments-2025-11-01
   x-environment-runner-version: {semver}
   X-Trusted-Device-Token: {device_token}   # optional, elevated security
   ```
3. On `401`, bridge calls refresh handler and retries once.

### Session Ingress Token (JWT)

- Obtained from `WorkSecret.session_ingress_token`
- Prefix: `sk-ant-si-`
- Passed as `CLAUDE_CODE_SESSION_ACCESS_TOKEN` env var to child process
- Contains `exp` (Unix seconds) — bridge refreshes **5 minutes before expiry**
- Fallback refresh interval: 30 minutes

### Token Lookup Priority (Child Process)

1. `CLAUDE_CODE_SESSION_ACCESS_TOKEN` env var
2. File descriptor: `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR`
3. File path: `CLAUDE_SESSION_INGRESS_TOKEN_FILE` or `/home/claude/.claude/remote/.session_ingress_token`

---

## 22.7 Environment Variables

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | Session ingress JWT (bridge-injected at spawn) |
| `CLAUDE_CODE_ENVIRONMENT_KIND` | Set to `'bridge'` in child processes |
| `CLAUDE_CODE_FORCE_SANDBOX` | Bridge-injected; forces sandbox mode for tools |
| `CLAUDE_BRIDGE_OAUTH_TOKEN` | Dev override: OAuth access token instead of keychain |
| `CLAUDE_BRIDGE_BASE_URL` | Dev override: bridge API base URL |
| `CLAUDE_BRIDGE_SESSION_INGRESS_URL` | Dev override: session ingress URL |
| `CLAUDE_BRIDGE_USE_CCR_V2` | Dev override: force CCR v2 (SSE) transport |
| `CLAUDE_TRUSTED_DEVICE_TOKEN` | Trusted device enrollment token |
| `CLAUDE_CODE_USE_CODE_SESSIONS` | Enable CCR v2 in child (set when `use_code_sessions=true`) |
| `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2` | Use HybridTransport (WS reads + HTTP POST writes) |

---

## 22.8 TypeScript Interfaces (Source-Accurate)

Full type definitions from `src/bridge/types.ts`:

```typescript
export type SpawnMode = 'single-session' | 'worktree' | 'same-dir'
export type BridgeWorkerType = 'claude_code' | 'claude_code_assistant'

export type BridgeConfig = {
  dir: string
  machineName: string
  branch: string
  gitRepoUrl: string | null
  maxSessions: number
  spawnMode: SpawnMode
  verbose: boolean
  sandbox: boolean
  bridgeId: string          // client-generated UUID (idempotent registration)
  workerType: string        // 'claude_code' | 'claude_code_assistant' | any
  environmentId: string     // client-generated UUID
  reuseEnvironmentId?: string  // backend-issued ID for resume (--session-id)
  apiBaseUrl: string
  sessionIngressUrl: string
  debugFile?: string
  sessionTimeoutMs?: number // default: 86_400_000 (24h)
}

export type SessionHandle = {
  sessionId: string
  done: Promise<'completed' | 'failed' | 'interrupted'>
  kill(): void
  forceKill(): void
  activities: SessionActivity[]       // ring buffer ~10 entries
  currentActivity: SessionActivity | null
  accessToken: string                 // current session_ingress_token
  lastStderr: string[]                // ring buffer of stderr lines
  writeStdin(data: string): void
  updateAccessToken(token: string): void
}

export type SessionActivity = {
  type: 'tool_start' | 'text' | 'result' | 'error'
  summary: string   // e.g. "Editing src/foo.ts"
  timestamp: number
}

export type PermissionResponseEvent = {
  type: 'control_response'
  response: {
    subtype: 'success'
    request_id: string
    response: Record<string, unknown>
  }
}

export type BridgeApiClient = {
  registerBridgeEnvironment(config: BridgeConfig): Promise<{
    environment_id: string
    environment_secret: string
  }>
  pollForWork(
    environmentId: string,
    environmentSecret: string,
    signal?: AbortSignal,
    reclaimOlderThanMs?: number,
  ): Promise<WorkResponse | null>
  acknowledgeWork(environmentId: string, workId: string, sessionToken: string): Promise<void>
  stopWork(environmentId: string, workId: string, force: boolean): Promise<void>
  deregisterEnvironment(environmentId: string): Promise<void>
  sendPermissionResponseEvent(
    sessionId: string,
    event: PermissionResponseEvent,
    sessionToken: string,
  ): Promise<void>
  archiveSession(sessionId: string): Promise<void>
  reconnectSession(environmentId: string, sessionId: string): Promise<void>
  heartbeatWork(
    environmentId: string,
    workId: string,
    sessionToken: string,
  ): Promise<{ lease_extended: boolean; state: string }>
}
```

---

## 22.9 Example: Python Bridge Client

A minimal bridge that registers, polls, and relays messages to a WebSocket session:

```python
import asyncio
import base64
import json
import uuid
import httpx
import websockets

ANTHROPIC_BASE = "https://api.anthropic.com"
INGRESS_BASE   = "https://session-ingress.anthropic.com"  # adjust for env
BETA_HEADER    = "environments-2025-11-01"
RUNNER_VERSION = "1.0.0"

HEADERS = lambda token: {
    "Authorization": f"Bearer {token}",
    "Content-Type": "application/json",
    "anthropic-version": "2023-06-01",
    "anthropic-beta": BETA_HEADER,
    "x-environment-runner-version": RUNNER_VERSION,
}


class BridgeClient:
    def __init__(self, oauth_token: str):
        self.token = oauth_token
        self.client = httpx.AsyncClient(base_url=ANTHROPIC_BASE, timeout=65)
        self.env_id: str | None = None
        self.env_secret: str | None = None

    # ── Phase 1: Register ────────────────────────────────────────────────────

    async def register(self) -> None:
        bridge_id = str(uuid.uuid4())
        env_id    = str(uuid.uuid4())
        payload = {
            "id": bridge_id,
            "metadata": {
                "worker_type": "claude_code",
                "machine_name": "my-bridge",
                "branch": "main",
                "git_repo_url": None,
                "max_sessions": 1,
                "spawn_mode": "single-session",
            },
        }
        r = await self.client.post(
            "/v1/environments/bridge",
            headers=HEADERS(self.token),
            json=payload,
        )
        r.raise_for_status()
        data = r.json()
        self.env_id     = data["environment_id"]
        self.env_secret = data["environment_secret"]
        print(f"Registered: env_id={self.env_id}")

    # ── Phase 2: Poll ────────────────────────────────────────────────────────

    async def poll_for_work(self) -> dict | None:
        r = await self.client.get(
            f"/v1/environments/{self.env_id}/work",
            headers=HEADERS(self.token),
        )
        if r.status_code == 204:
            return None
        r.raise_for_status()
        return r.json()

    # ── Phase 3: Decode WorkSecret ────────────────────────────────────────────

    @staticmethod
    def decode_secret(b64: str) -> dict:
        # base64url, add padding if needed
        padded = b64 + "=" * (-len(b64) % 4)
        return json.loads(base64.urlsafe_b64decode(padded))

    # ── Phase 4: Connect session via WebSocket (v1) ───────────────────────────

    async def run_session(self, work: dict) -> None:
        secret    = self.decode_secret(work["secret"])
        session_id = work["data"]["id"]
        jwt_token  = secret["session_ingress_token"]
        ingress_url = secret["api_base_url"]

        ws_url = f"{ingress_url}/v1/session_ingress/ws/{session_id}"
        ws_url = ws_url.replace("https://", "wss://").replace("http://", "ws://")

        extra_headers = {"Authorization": f"Bearer {jwt_token}"}
        async with websockets.connect(ws_url, extra_headers=extra_headers) as ws:
            print(f"Connected to session {session_id}")
            await self._exchange(ws, session_id, jwt_token)

    async def _exchange(self, ws, session_id: str, jwt_token: str) -> None:
        # Send initialize control_request
        init_req = {
            "type": "control_request",
            "request_id": str(uuid.uuid4()),
            "request": {
                "subtype": "initialize",
                "systemPrompt": "You are a helpful assistant.",
            },
        }
        await ws.send(json.dumps(init_req))

        # Wait for initialize response, then send user message
        initialized = False
        async for raw in ws:
            msg = json.loads(raw)
            mtype = msg.get("type")

            if mtype == "control_response":
                sub = msg["response"].get("subtype")
                rid = msg["response"].get("request_id")
                print(f"  control_response: subtype={sub} request_id={rid}")
                if not initialized and sub == "success":
                    initialized = True
                    user_msg = {
                        "type": "user",
                        "message": {
                            "role": "user",
                            "content": [{"type": "text", "text": "Hello! What can you do?"}],
                        },
                    }
                    await ws.send(json.dumps(user_msg))

            elif mtype == "assistant":
                for block in msg.get("message", {}).get("content", []):
                    if block.get("type") == "text":
                        print(f"  Assistant: {block['text']}")

            elif mtype == "result":
                print(f"  Session result: {msg.get('result')}")
                break

            elif mtype == "keep_alive":
                await ws.send(json.dumps({"type": "keep_alive"}))

    # ── Phase 5: Acknowledge + Archive ───────────────────────────────────────

    async def acknowledge(self, work_id: str, session_token: str) -> None:
        await self.client.post(
            f"/v1/environments/{self.env_id}/work/{work_id}/acknowledge",
            headers=HEADERS(self.token),
            json={"session_token": session_token},
        )

    async def archive(self, session_id: str, session_token: str) -> None:
        await self.client.post(
            f"/v1/sessions/{session_id}/archive",
            headers={**HEADERS(self.token), "Authorization": f"Bearer {session_token}"},
        )

    async def deregister(self) -> None:
        await self.client.delete(
            f"/v1/environments/{self.env_id}",
            headers=HEADERS(self.token),
        )
        print("Deregistered environment")


# ── Main Loop ────────────────────────────────────────────────────────────────

async def main(oauth_token: str) -> None:
    bridge = BridgeClient(oauth_token)
    await bridge.register()

    try:
        while True:
            print("Polling for work…")
            work = await bridge.poll_for_work()
            if work is None:
                await asyncio.sleep(1)
                continue

            secret = bridge.decode_secret(work["secret"])
            session_id = work["data"]["id"]
            jwt_token  = secret["session_ingress_token"]

            await bridge.run_session(work)
            await bridge.acknowledge(work["id"], jwt_token)
            await bridge.archive(session_id, jwt_token)
            break  # single-session mode

    finally:
        await bridge.deregister()


if __name__ == "__main__":
    import os
    asyncio.run(main(os.environ["CLAUDE_OAUTH_TOKEN"]))
```

---

## 22.10 Example: Workflow — Automated Code Review Bot

This workflow dispatches a code review task to Claude Code via the bridge and collects results:

```python
import asyncio, json, uuid, base64, subprocess, os
import httpx, websockets

ANTHROPIC_BASE = "https://api.anthropic.com"
BETA = "environments-2025-11-01"


async def code_review_bot(oauth_token: str, repo_path: str, diff: str) -> str:
    """Register a bridge, wait for a work item, run Claude on the diff, return review."""
    headers = lambda t: {
        "Authorization": f"Bearer {t}",
        "Content-Type": "application/json",
        "anthropic-version": "2023-06-01",
        "anthropic-beta": BETA,
        "x-environment-runner-version": "1.0.0",
    }

    async with httpx.AsyncClient(base_url=ANTHROPIC_BASE, timeout=65) as http:
        # 1. Register
        reg = await http.post("/v1/environments/bridge", headers=headers(oauth_token), json={
            "id": str(uuid.uuid4()),
            "metadata": {
                "worker_type": "claude_code",
                "machine_name": "review-bot",
                "branch": "main",
                "git_repo_url": None,
                "max_sessions": 1,
                "spawn_mode": "single-session",
            },
        })
        reg.raise_for_status()
        env = reg.json()
        env_id, env_secret = env["environment_id"], env["environment_secret"]

        # 2. Launch claude remote-control as a subprocess in the repo dir
        proc = subprocess.Popen(
            ["claude", "remote-control", "--session-id", env_id],
            cwd=repo_path,
            env={**os.environ, "CLAUDE_BRIDGE_OAUTH_TOKEN": oauth_token},
        )

        review_text = ""
        try:
            # 3. Poll for the work item that the subprocess registers
            work = None
            for _ in range(30):
                r = await http.get(f"/v1/environments/{env_id}/work", headers=headers(oauth_token))
                if r.status_code == 200:
                    work = r.json()
                    break
                await asyncio.sleep(2)

            if not work:
                raise TimeoutError("No work item received")

            secret = json.loads(base64.urlsafe_b64decode(work["secret"] + "=="))
            session_id = work["data"]["id"]
            jwt = secret["session_ingress_token"]
            ingress = secret["api_base_url"]

            ws_url = ingress.replace("https://", "wss://") + f"/v1/session_ingress/ws/{session_id}"

            # 4. Connect + drive the review
            async with websockets.connect(ws_url, extra_headers={"Authorization": f"Bearer {jwt}"}) as ws:
                # initialize
                await ws.send(json.dumps({
                    "type": "control_request",
                    "request_id": str(uuid.uuid4()),
                    "request": {
                        "subtype": "initialize",
                        "systemPrompt": (
                            "You are a senior code reviewer. "
                            "Review diffs for correctness, security, and style. "
                            "Be concise and actionable."
                        ),
                    },
                }))

                init_done = False
                async for raw in ws:
                    msg = json.loads(raw)
                    if msg["type"] == "control_response" and not init_done:
                        if msg["response"]["subtype"] == "success":
                            init_done = True
                            # send the diff as the user message
                            await ws.send(json.dumps({
                                "type": "user",
                                "message": {
                                    "role": "user",
                                    "content": [{"type": "text", "text": f"Review this diff:\n\n```diff\n{diff}\n```"}],
                                },
                            }))

                    elif msg["type"] == "assistant":
                        for block in msg.get("message", {}).get("content", []):
                            if block.get("type") == "text":
                                review_text += block["text"]

                    elif msg["type"] == "result":
                        break

                    elif msg["type"] == "keep_alive":
                        await ws.send(json.dumps({"type": "keep_alive"}))

            # 5. Acknowledge
            await http.post(
                f"/v1/environments/{env_id}/work/{work['id']}/acknowledge",
                headers=headers(oauth_token),
                json={"session_token": jwt},
            )

        finally:
            proc.terminate()
            await http.delete(f"/v1/environments/{env_id}", headers=headers(oauth_token))

    return review_text


# Usage
if __name__ == "__main__":
    diff = """
--- a/src/auth.py
+++ b/src/auth.py
@@ -12,7 +12,7 @@
-def check_password(user_input):
-    return user_input == SECRET_PASSWORD
+def check_password(user_input, stored_hash):
+    return bcrypt.checkpw(user_input.encode(), stored_hash)
"""
    review = asyncio.run(code_review_bot(
        oauth_token=os.environ["CLAUDE_OAUTH_TOKEN"],
        repo_path="/path/to/repo",
        diff=diff,
    ))
    print(review)
```

---

## 22.11 Multi-Session Spawn Modes

| Mode | Behavior | Use Case |
|------|----------|---------|
| `single-session` | One session; bridge exits when done | CI jobs, one-shot tasks |
| `worktree` | Each session gets an isolated git worktree | Parallel PR reviews |
| `same-dir` | All sessions share cwd (can conflict) | Interactive dashboard |

```typescript
// BridgeConfig.spawnMode controls this:
type SpawnMode = 'single-session' | 'worktree' | 'same-dir'
```

In `worktree` mode, each `WorkResponse` triggers creation of a new git worktree under the base directory. Sessions are fully isolated at the filesystem level.

---

## 22.12 Error Handling

### WebSocket Close Codes (v1)

| Code | Meaning | Action |
|------|---------|--------|
| `1002` | Protocol error / session reaped | Permanent close, no retry |
| `4001` | Session expired or not found | Permanent close |
| `4003` | Unauthorized | Permanent close, refresh token |

### HTTP Status (Bridge API)

| Status | Meaning | Action |
|--------|---------|--------|
| `204` | No work available (poll timeout) | Continue polling |
| `401` | OAuth token stale | Refresh and retry once |
| `400` | Bad request (e.g. invalid ID format) | `BridgeFatalError`, no retry |
| `403` | Permission denied | `BridgeFatalError` |
| `404` | Environment not found | `BridgeFatalError` |
| `5xx` | Server error | Retry with exponential backoff (500 ms → 30 s) |

### ID Safety

Server-provided IDs must match `/^[a-zA-Z0-9_-]+$/` before interpolating into URL paths — prevents path traversal. The source enforces this in `validateBridgeId()` (`src/bridge/bridgeApi.ts:48`).

---

## 22.13 Token Refresh

```
JWT expiry detected (exp field in token payload)
    ↓
Schedule refresh at (exp − 5 minutes)
    ↓
GET new session_ingress_token via OAuth
    ↓
Call sessionHandle.updateAccessToken(newToken)
    ↓
Schedule next refresh in 30 minutes (fallback)
```

The bridge proactively refreshes; the child process receives the new token via `update_environment_variables` message:

```json
{
  "type": "update_environment_variables",
  "variables": {
    "CLAUDE_CODE_SESSION_ACCESS_TOKEN": "sk-ant-si-..."
  }
}
```

---

## 22.14 Image Content Normalization

Web/mobile clients sometimes send camelCase image fields. The bridge normalizes on ingest:

```typescript
// Malformed (iOS/web client)
{ type: 'image', source: { type: 'base64', mediaType: 'image/png', data: '...' } }

// Normalized to (Anthropic API canonical)
{ type: 'image', source: { type: 'base64', media_type: 'image/png', data: '...' } }
```

---

## 22.15 AppState Fields

Two fields in the REPL `AppState` reflect bridge connectivity:

```typescript
replBridgeEnabled:   boolean   // true when --remote-control / --bridge flag set
replBridgeConnected: boolean   // true when actively connected to ingress
```

---

## 22.16 Cross-References

- **§01** — Architecture Overview: bridge subsystem entry, execution mode 5
- **§02** — Data Flow: `bridgeMain()` bootstrap path
- **§03** — Core Data Structures: `replBridgeEnabled`, `replBridgeConnected` in AppState
- **§06** — Commands & Skills: bridge-safe command set, `/remote-control` command
- **§08** — Permissions & Hooks: `set_permission_mode` control request, hook injection via `initialize`
- **§09** — Configuration: `CLAUDE_CODE_SESSION_ACCESS_TOKEN`, `CLAUDE_CODE_ENVIRONMENT_KIND`, `CLAUDE_CODE_FORCE_SANDBOX`
- **§11** — Multi-Agent: bridge sessions spawn child processes using the same agent loop

*[← Prompt Caching Deep Dive](./21-prompt-caching.md)*
