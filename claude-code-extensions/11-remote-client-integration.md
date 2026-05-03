# Remote Client Integration — Third-Party Apps as Bridge Clients

Claude Code exposes a full API that lets any third-party application connect as a remote client: sending prompts, receiving streamed model responses, handling permission decisions, injecting MCP servers and custom agents, and reading session events. This guide covers both connection modes and a complete Python implementation.

---

## Two Connection Modes

```
Mode A: Environment-based (persistent worker)
  ─────────────────────────────────────────
  Third-party app registers as a "bridge environment" on claude.ai.
  Claude.ai dispatches work (sessions) to the worker via pull polling.
  Worker runs one or more REPL sessions. Scales to multi-session.

Mode B: Env-less / per-session (CCR v2)
  ────────────────────────────────────
  Third-party app creates a code session on demand.
  Gets credentials, connects via WebSocket/SSE.
  Single session per connection. No polling needed.
```

| Feature | Mode A (env-based) | Mode B (env-less) |
|---------|-------------------|-------------------|
| Persistent worker | Yes | No |
| Session capacity | configurable `max_sessions` | 1 |
| Work dispatch | Server pulls to worker | App creates sessions |
| Auth | `environment_secret` (HMAC) | `worker_jwt` (JWT) |
| Best for | Always-on agents, background workers | On-demand integrations |

---

## Mode A: Environment-Based (Persistent Worker)

### Step 1 — Register as Bridge Worker

```
POST /v1/environments/bridge
Authorization: Bearer {oauth_token}
anthropic-beta: environments-2025-11-01

{
  "machine_name": "my-agent-server",
  "directory": "/home/user/project",
  "branch": "main",
  "git_repo_url": "https://github.com/org/repo",
  "max_sessions": 3,
  "metadata": {
    "worker_type": "claude_code"
  }
}
```

Response:
```json
{
  "environment_id": "env_01ABC...",
  "environment_secret": "s3cr3t..."
}
```

Store both. `environment_id` is the stable identity; `environment_secret` signs subsequent requests.

### Step 2 — Poll for Work

```
GET /v1/environments/{environment_id}/work/poll
Authorization: {signed_with_environment_secret}
anthropic-beta: environments-2025-11-01
```

Responses:
- `204 No Content` — no pending work, sleep and retry
- `200 OK` — work item available:

```json
{
  "work_id": "work_01XYZ...",
  "session_id": "cse_01...",
  "secret": "{base64url WorkSecret}"
}
```

### WorkSecret Schema

The `secret` field is a base64url-encoded JSON object:

```typescript
type WorkSecret = {
  version: number                              // Currently 1
  session_ingress_token: string                // JWT for session access
  api_base_url: string                         // Session-ingress base URL
  sources: Array<{
    type: string
    git_info?: { type: string; repo: string; ref?: string; token?: string }
  }>
  auth: Array<{ type: string; token: string }> // OAuth token for the session
  claude_code_args?: Record<string, string> | null
  mcp_config?: unknown | null
  environment_variables?: Record<string, string> | null
  use_code_sessions?: boolean                  // true = CCR v2 path
}
```

### Step 3 — Acknowledge Work

```
POST /v1/environments/{environment_id}/work/{work_id}/ack
Authorization: {session_ingress_token}
```

Must be called before starting work. Server extends lease on ACK.

### Step 4 — Heartbeat

```
POST /v1/environments/{environment_id}/work/{work_id}/heartbeat
Authorization: {session_ingress_token}

Response:
{
  "lease_extended": true,
  "state": "running",
  "last_heartbeat": "2026-05-03T10:00:00Z",
  "ttl_seconds": 60
}
```

Send periodically (every 30–45s). If `lease_extended=false`, the session has been revoked.

### Step 5 — Stop Work

```
POST /v1/environments/{environment_id}/work/{work_id}/stop
Authorization: Bearer {oauth_token}

// Optional body:
{ "force": true }
```

### Deregister Worker

```
DELETE /v1/environments/bridge/{environment_id}
Authorization: Bearer {oauth_token}
anthropic-beta: environments-2025-11-01
```

---

## Mode B: Env-less / Per-Session (CCR v2)

### Step 1 — Create Code Session

```
POST /v1/code/sessions
Authorization: Bearer {oauth_token}

{
  "title": "My Agent Session",
  "bridge": {},
  "tags": ["automated", "my-app"]
}

Response:
{
  "session": { "id": "cse_01ABC..." }
}
```

### Step 2 — Get Worker Credentials

```
POST /v1/code/sessions/{session_id}/bridge
Authorization: Bearer {oauth_token}
X-Trusted-Device-Token: {device_token}   // optional, enables ELEVATED tier

Response:
{
  "worker_jwt": "eyJ...",     // opaque — do not decode
  "api_base_url": "https://api.anthropic.com",
  "expires_in": 3600,
  "worker_epoch": 1
}
```

The `worker_jwt` is the bearer token for all subsequent session operations.

### Step 3 — Register Worker

```
POST {api_base_url}/worker/register
Authorization: Bearer {worker_jwt}

Response:
{ "worker_epoch": 1 }
```

Confirms the worker is live. The epoch must match what the server expects.

### Step 4 — Connect to Session-Ingress

Use `HybridTransport` semantics:
- **Reads**: WebSocket or SSE on `{api_base_url}/v1/sessions/ws/{session_id}/subscribe`
- **Writes**: HTTP POST batches to `{api_base_url}/v1/sessions/{session_id}/events`

WebSocket auth handshake:
```json
{ "type": "auth", "credential": { "type": "oauth", "token": "{worker_jwt}" } }
```

---

## Session Message Protocol

### Send a User Prompt

Write an `SDKUserMessage` to the session events endpoint:

```json
{
  "type": "user",
  "message": {
    "role": "user",
    "content": "List all Python files and show me the imports in each"
  },
  "parent_tool_use_id": null,
  "uuid": "msg-uuid-12345",
  "timestamp": "2026-05-03T10:00:00Z",
  "session_id": "cse_01ABC..."
}
```

### Receive Streamed Response

Messages arrive on the WebSocket as `SDKMessage` objects:

```
→ { "type": "system", "subtype": "status", ... }       // turn started
→ { "type": "stream_event", ... }                       // streaming text delta
→ { "type": "assistant", "message": {...} }             // complete assistant turn
→ { "type": "system", "subtype": "task_started", ... } // tool executing
→ { "type": "user", "message": {...} }                  // tool result
→ { "type": "result", "subtype": "success", ... }       // turn complete
```

### Handle Permission Requests

When a tool needs authorization, you receive a control request:

```json
{
  "type": "control_request",
  "request": {
    "subtype": "can_use_tool",
    "tool_name": "bash",
    "input": { "command": "rm -rf /tmp/cache" },
    "tool_use_id": "toolu_01...",
    "title": "Run Bash Command",
    "description": "Delete temporary cache directory",
    "permission_suggestions": [
      { "type": "allow", "pattern": "rm -rf /tmp/*" }
    ]
  },
  "request_id": "req-01XYZ"
}
```

Respond via:
```
POST /v1/sessions/{session_id}/events
Authorization: Bearer {worker_jwt}

{
  "events": [{
    "type": "control_response",
    "response": {
      "subtype": "success",
      "request_id": "req-01XYZ",
      "response": {
        "behavior": "allow",
        "updatedInput": { "command": "rm -rf /tmp/cache" }
      }
    }
  }]
}
```

To deny:
```json
{ "behavior": "deny", "message": "Not allowed in automated mode" }
```

---

## Initialize Handshake — Injecting Plugins, Agents, and Hooks

After connecting, send an `initialize` control request to inject your configuration:

```json
{
  "type": "control_request",
  "request": {
    "subtype": "initialize",
    "systemPrompt": "You are an automated code review agent. Always respond in structured JSON.",
    "appendSystemPrompt": "Format all findings as {issue, severity, file, line}.",
    "agents": {
      "security-scanner": {
        "name": "security-scanner",
        "description": "Scans code for security vulnerabilities",
        "systemPrompt": "You are a security expert...",
        "tools": ["bash", "read_file", "glob"]
      }
    },
    "sdkMcpServers": ["sqlite", "filesystem"],
    "hooks": {
      "PostToolUse": [{
        "matcher": "bash",
        "hookCallbackIds": ["audit-hook"],
        "timeout": 10
      }]
    },
    "promptSuggestions": false,
    "agentProgressSummaries": true
  },
  "request_id": "init-001"
}
```

Response contains available state:
```json
{
  "type": "control_response",
  "response": {
    "subtype": "success",
    "request_id": "init-001",
    "response": {
      "commands": [...],     // all registered slash commands
      "agents": [...],       // your agents + built-ins
      "models": [...],       // available models
      "account": {...},      // account info
      "output_style": "text",
      "pid": 12345
    }
  }
}
```

---

## Remote Assistant (Viewer Mode)

A third-party app can connect as a **viewer only** — receiving the session stream without sending prompts:

```
WebSocket: wss://api.anthropic.com/v1/sessions/ws/{session_id}/subscribe?organization_uuid={org}
```

Auth:
```json
{ "type": "auth", "credential": { "type": "oauth", "token": "{oauth_token}" } }
```

Viewer characteristics:
- Receives all `SDKMessage` events (read-only)
- Cannot interrupt or send prompts
- Reconnect: 2s initial, exponential backoff, max 5 attempts
- Ping interval: 30s
- Session not found retries: max 3 (session may be compacting)
- Permanent disconnect code: `4003` (unauthorized)

---

## Trusted Device Token

High-security bridge sessions require `ELEVATED` tier auth. Without it, some operations are gated:

```
POST /api/auth/trusted_devices
Authorization: Bearer {oauth_token}

{ "display_name": "My CI Runner" }

Response (within 10-minute enrollment window):
{
  "device_token": "dtok_01...",
  "device_id": "dev_01..."
}
```

Use on all bridge API calls:
```
X-Trusted-Device-Token: dtok_01...
```

Source: `CLAUDE_TRUSTED_DEVICE_TOKEN` env var OR keychain (90-day rolling expiry).

---

## Mid-Session Control Requests

After initialize, use these to mutate the live session:

| Control Subtype | Effect |
|----------------|--------|
| `interrupt` | Abort the current turn |
| `set_model` | Change active model (e.g., `"claude-opus-4-7"`) |
| `set_permission_mode` | `'bypassPermissions'`, `'plan'`, `'acceptEdits'`, `'default'` |
| `set_max_thinking_tokens` | Extended thinking budget (null = disable) |
| `mcp_set_servers` | Replace all MCP servers |
| `mcp_toggle` | Enable/disable a specific MCP server |
| `reload_plugins` | Re-scan and reload plugins from disk |
| `get_settings` | Read current effective settings JSON |
| `apply_flag_settings` | Merge a settings overlay (temporary) |
| `get_context_usage` | Get context window token breakdown |
| `rewind_files` | Undo file changes since a message UUID |
| `cancel_async_message` | Drop a queued prompt (before it starts) |
| `seed_read_state` | Pre-populate file cache to reduce first-read latency |
| `stop_task` | Abort a specific background agent task |

---

## Full Python Example

```python
"""
Third-party app connecting to Claude Code via bridge (env-less / CCR v2 mode).
"""
import asyncio
import json
import time
import uuid
import httpx
import websockets
from dataclasses import dataclass, field
from typing import AsyncIterator, Optional

ANTHROPIC_API = "https://api.anthropic.com"
BETA_HEADER  = "environments-2025-11-01"

@dataclass
class SessionCredentials:
    session_id: str
    worker_jwt: str
    api_base_url: str
    expires_in: int

@dataclass
class BridgeClient:
    credentials: SessionCredentials
    _ws: object = field(default=None, init=False)
    _pending: dict = field(default_factory=dict, init=False)

    # ── Session creation ──────────────────────────────────────────────────────

    @classmethod
    async def create(
        cls,
        oauth_token: str,
        title: str = "Remote Agent",
        trusted_device_token: Optional[str] = None,
    ) -> "BridgeClient":
        headers = {
            "Authorization": f"Bearer {oauth_token}",
            "Content-Type": "application/json",
            "anthropic-version": "2023-06-01",
            "anthropic-beta": BETA_HEADER,
        }
        async with httpx.AsyncClient() as http:
            # 1. Create code session
            r = await http.post(
                f"{ANTHROPIC_API}/v1/code/sessions",
                headers=headers,
                json={"title": title, "bridge": {}},
            )
            r.raise_for_status()
            session_id = r.json()["session"]["id"]

            # 2. Get worker credentials
            bridge_headers = {**headers}
            if trusted_device_token:
                bridge_headers["X-Trusted-Device-Token"] = trusted_device_token
            r = await http.post(
                f"{ANTHROPIC_API}/v1/code/sessions/{session_id}/bridge",
                headers=bridge_headers,
            )
            r.raise_for_status()
            creds_data = r.json()
            creds = SessionCredentials(
                session_id=session_id,
                worker_jwt=creds_data["worker_jwt"],
                api_base_url=creds_data["api_base_url"],
                expires_in=creds_data["expires_in"],
            )

            # 3. Register worker
            worker_headers = {
                "Authorization": f"Bearer {creds.worker_jwt}",
                "Content-Type": "application/json",
            }
            r = await http.post(
                f"{creds.api_base_url}/worker/register",
                headers=worker_headers,
            )
            r.raise_for_status()

        return cls(credentials=creds)

    # ── Connect and initialize ────────────────────────────────────────────────

    async def connect(self):
        ws_url = (
            self.credentials.api_base_url
            .replace("https://", "wss://")
            .replace("http://", "ws://")
            + f"/v1/sessions/ws/{self.credentials.session_id}/subscribe"
        )
        self._ws = await websockets.connect(ws_url)
        # Auth handshake
        await self._ws.send(json.dumps({
            "type": "auth",
            "credential": {"type": "oauth", "token": self.credentials.worker_jwt}
        }))
        # Await auth confirmation
        auth_resp = json.loads(await self._ws.recv())
        if auth_resp.get("type") != "auth_ok":
            raise ConnectionError(f"Auth failed: {auth_resp}")

    async def initialize(
        self,
        system_prompt: Optional[str] = None,
        append_system_prompt: Optional[str] = None,
        agents: Optional[dict] = None,
        mcp_servers: Optional[list[str]] = None,
        hooks: Optional[dict] = None,
    ) -> dict:
        request_id = f"init-{uuid.uuid4()}"
        payload = {"subtype": "initialize"}
        if system_prompt:       payload["systemPrompt"] = system_prompt
        if append_system_prompt: payload["appendSystemPrompt"] = append_system_prompt
        if agents:              payload["agents"] = agents
        if mcp_servers:         payload["sdkMcpServers"] = mcp_servers
        if hooks:               payload["hooks"] = hooks

        return await self.send_control(payload, request_id)

    # ── Messaging ─────────────────────────────────────────────────────────────

    async def send_prompt(self, text: str) -> str:
        """Send a user prompt. Returns the message UUID for tracking."""
        msg_uuid = str(uuid.uuid4())
        await self._post_events([{
            "type": "user",
            "message": {"role": "user", "content": text},
            "parent_tool_use_id": None,
            "uuid": msg_uuid,
            "timestamp": self._now(),
            "session_id": self.credentials.session_id,
        }])
        return msg_uuid

    async def stream(self) -> AsyncIterator[dict]:
        """Yield SDK messages from the WebSocket."""
        async for raw in self._ws:
            msg = json.loads(raw)
            yield msg
            if msg.get("type") == "result":
                break  # turn complete

    # ── Permission handling ───────────────────────────────────────────────────

    async def handle_permission(
        self,
        request: dict,
        decision: str = "allow",  # "allow" | "deny"
        message: str = "",
    ):
        request_id = request.get("request_id") or request["request"].get("request_id")
        tool_use_id = request["request"]["tool_use_id"]

        if decision == "allow":
            response_body = {
                "behavior": "allow",
                "updatedInput": request["request"]["input"],
            }
        else:
            response_body = {
                "behavior": "deny",
                "message": message or "Denied by remote client",
            }

        await self._post_events([{
            "type": "control_response",
            "response": {
                "subtype": "success",
                "request_id": request_id,
                "response": response_body,
            }
        }])

    # ── Control requests ──────────────────────────────────────────────────────

    async def interrupt(self):
        await self.send_control({"subtype": "interrupt"})

    async def set_permission_mode(self, mode: str):
        """mode: 'bypassPermissions' | 'acceptEdits' | 'plan' | 'default'"""
        await self.send_control({"subtype": "set_permission_mode", "mode": mode})

    async def send_control(self, request: dict, request_id: Optional[str] = None) -> dict:
        rid = request_id or str(uuid.uuid4())
        await self._post_events([{
            "type": "control_request",
            "request": request,
            "request_id": rid,
        }])
        # Wait for response on WebSocket
        async for raw in self._ws:
            msg = json.loads(raw)
            if (msg.get("type") == "control_response"
                    and msg.get("response", {}).get("request_id") == rid):
                return msg["response"]

    # ── Internals ─────────────────────────────────────────────────────────────

    async def _post_events(self, events: list):
        async with httpx.AsyncClient() as http:
            r = await http.post(
                f"{self.credentials.api_base_url}/v1/sessions/{self.credentials.session_id}/events",
                headers={"Authorization": f"Bearer {self.credentials.worker_jwt}",
                         "Content-Type": "application/json"},
                json={"events": events},
            )
            r.raise_for_status()

    @staticmethod
    def _now() -> str:
        from datetime import datetime, timezone
        return datetime.now(timezone.utc).isoformat()

    async def teardown(self):
        if self._ws:
            await self._ws.close()


# ── Example usage ──────────────────────────────────────────────────────────────

async def main():
    import os
    token = os.environ["ANTHROPIC_API_KEY"]

    client = await BridgeClient.create(token, title="CI Code Review")
    await client.connect()

    # Inject custom agent and bypass permissions for CI
    await client.initialize(
        system_prompt="You are a CI code review agent. Be concise.",
        agents={
            "reviewer": {
                "name": "reviewer",
                "description": "Reviews diffs for correctness",
                "systemPrompt": "Review this diff and list issues.",
                "tools": ["read_file", "bash", "glob"],
            }
        },
    )
    await client.set_permission_mode("bypassPermissions")

    # Send prompt and stream response
    await client.send_prompt("Review the Python files in /home/user/project for type errors")

    assistant_text = []
    async for msg in client.stream():
        mtype = msg.get("type")

        if mtype == "stream_event":
            delta = msg.get("delta", {}).get("text", "")
            print(delta, end="", flush=True)
            assistant_text.append(delta)

        elif mtype == "control_request":
            req = msg.get("request", {})
            if req.get("subtype") == "can_use_tool":
                # Auto-allow in CI
                await client.handle_permission(msg, decision="allow")

        elif mtype == "result":
            print("\n--- Turn complete ---")
            break

    await client.teardown()


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Security Considerations

| Concern | Mitigation |
|---------|-----------|
| `worker_jwt` expires | `expires_in` seconds; re-call `/bridge` to get fresh credentials |
| Permission bypass in CI | Use `bypassPermissions` mode only for sandboxed environments |
| Sensitive paths | Use `rewind_files` to roll back writes on failure |
| Inbound prompt injection | `outbound_only=true` blocks all inbound prompts — use for viewer apps |
| Session isolation | Each session has independent file state, tool context, and abort controller |

---

## API Endpoint Reference

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| POST | `/v1/environments/bridge` | OAuth | Register env-based worker |
| GET | `/v1/environments/{envId}/work/poll` | env_secret | Poll for sessions to handle |
| POST | `/v1/environments/{envId}/work/{workId}/ack` | session_token | Acknowledge work item |
| POST | `/v1/environments/{envId}/work/{workId}/heartbeat` | session_token | Extend lease |
| POST | `/v1/environments/{envId}/work/{workId}/stop` | OAuth | Stop work item |
| DELETE | `/v1/environments/bridge/{envId}` | OAuth | Deregister worker |
| POST | `/v1/code/sessions` | OAuth | Create per-session (env-less) |
| POST | `/v1/code/sessions/{id}/bridge` | OAuth | Get worker credentials |
| POST | `{api_base_url}/worker/register` | worker_jwt | Register worker with server |
| WS | `/v1/sessions/ws/{id}/subscribe` | worker_jwt | Stream session events |
| POST | `/v1/sessions/{id}/events` | worker_jwt | Send messages and control requests |
| POST | `/api/auth/trusted_devices` | OAuth | Enroll trusted device token |

Required headers on all requests:
```
anthropic-version: 2023-06-01
anthropic-beta: environments-2025-11-01
Content-Type: application/json
```

---

## Related Files

| File | Role |
|------|------|
| `src/bridge/bridgeApi.ts` | `BridgeApiClient` — all HTTP endpoints, auth patterns |
| `src/bridge/bridgeMain.ts` | Environment-based worker orchestration |
| `src/bridge/codeSessionApi.ts` | Env-less session creation and credential fetch |
| `src/bridge/workSecret.ts` | `WorkSecret` decode, SDK URL construction |
| `src/bridge/trustedDevice.ts` | Trusted device token storage and enrollment |
| `src/bridge/capacityWake.ts` | Poll loop wake signal |
| `src/remote/RemoteSessionManager.ts` | Viewer mode WebSocket subscriber |
| `src/remote/SessionsWebSocket.ts` | WebSocket reconnect logic (30s ping, max 5 retries) |
| `src/entrypoints/sdk/controlSchemas.ts` | Zod schemas for all control request/response types |
| `src/entrypoints/sdk/coreSchemas.ts` | Full SDKMessage union (25+ message types) |
| `src/cli/transports/HybridTransport.ts` | Batched write transport (100ms buffer, 500 msg batches) |

---

*[← Bridge Protocol](./10-bridge-protocol.md) | [Skills & Commands →](./01-skills-commands.md)*
