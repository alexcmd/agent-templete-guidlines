# Universal Agent Architecture — Distributed Multi-Agent & A2A Protocol

> Section: Distributed Multi-Agent & A2A Protocol

Part of the [Universal Agent Architecture](01-overview.md) series.

## 24. Distributed Multi-Agent & A2A Protocol

### 24.1 A2A vs Local Multi-Agent

| | Local (same process) | Distributed (A2A) |
|-|---------------------|-------------------|
| Communication | In-memory / function call | Network (HTTP/WebSocket/SSE) |
| Latency | Microseconds | Milliseconds–seconds |
| Failure modes | Exception propagation | Network timeout, partial failure |
| Auth | Inherited from parent | Separate per-agent credentials |
| Scale | Limited by single machine | Horizontal |
| Use case | Parallel tool execution | Cross-service agent delegation |

### 24.2 Agent Card (Capability Advertisement)

Each distributed agent publishes an **Agent Card** — a JSON descriptor that other agents
use to discover and invoke it:

```json
{
  "agent_id": "code-reviewer-v2",
  "name": "Code Review Agent",
  "version": "2.1.0",
  "description": "Reviews code diffs for bugs, security issues, and style",
  "capabilities": ["code_review", "security_audit", "style_check"],
  "input_schema": {
    "type": "object",
    "properties": {
      "diff": {"type": "string", "description": "Unified diff to review"},
      "language": {"type": "string"},
      "focus": {"type": "array", "items": {"type": "string"},
                "description": "Aspects to focus on: security|performance|style|bugs"}
    },
    "required": ["diff"]
  },
  "output_schema": {
    "type": "object",
    "properties": {
      "issues": {"type": "array"},
      "summary": {"type": "string"},
      "approved": {"type": "boolean"}
    }
  },
  "endpoint": "https://agents.example.com/code-reviewer",
  "auth": {"type": "bearer"},
  "streaming": true,
  "max_concurrent_tasks": 10
}
```

### 24.3 Task Delegation Protocol

**Request:**
```json
{
  "task_id": "task-uuid-v4",
  "agent_id": "code-reviewer-v2",
  "input": {
    "diff": "...",
    "language": "python",
    "focus": ["security", "bugs"]
  },
  "callback_url": "https://orchestrator.example.com/results/task-uuid-v4",
  "timeout_seconds": 120,
  "streaming": true
}
```

**Streaming response (Server-Sent Events):**
```
event: status
data: {"task_id":"task-uuid","status":"running","progress":0.2}

event: artifact
data: {"task_id":"task-uuid","type":"partial_result","content":"Found 2 issues...","append":true}

event: artifact
data: {"task_id":"task-uuid","type":"partial_result","content":" reviewing auth module","append":true}

event: status
data: {"task_id":"task-uuid","status":"completed","final":true}
```

**Terminal task states:**
- `completed` — success, result available
- `failed` — agent encountered an unrecoverable error
- `killed` — timed out or cancelled by orchestrator
- `auth-required` — agent needs additional credentials (pauses, waits for re-auth)

### 24.4 Result Reassembly

For streaming artifact updates with `append: true`:

```python
class A2AResultReassembler:
    def __init__(self):
        self._artifacts: dict[str, str] = {}
        self._status = "pending"
        self._final = False

    def feed(self, event_type: str, data: dict):
        if event_type == "status":
            self._status = data["status"]
            self._final = data.get("final", False)
        elif event_type == "artifact":
            key = data.get("artifact_id", "default")
            if data.get("append"):
                self._artifacts[key] = self._artifacts.get(key, "") + data["content"]
            else:
                self._artifacts[key] = data["content"]

    def is_done(self) -> bool:
        return self._final or self._status in ("completed", "failed", "killed")

    def result(self) -> dict:
        return {"status": self._status, "artifacts": self._artifacts}
```

### 24.5 Inter-Agent Authentication

```python
# Agent-to-agent calls use short-lived JWT tokens issued by an orchestrator

def make_agent_token(caller_id: str, callee_id: str, task_id: str,
                     secret: str, ttl_seconds: int = 300) -> str:
    import jwt, time
    payload = {
        "sub": caller_id,
        "aud": callee_id,
        "task_id": task_id,
        "iat": int(time.time()),
        "exp": int(time.time()) + ttl_seconds,
    }
    return jwt.encode(payload, secret, algorithm="HS256")

def verify_agent_token(token: str, expected_caller: str, secret: str) -> dict:
    import jwt
    payload = jwt.decode(token, secret, algorithms=["HS256"],
                         options={"require": ["sub", "aud", "task_id", "exp"]})
    if payload["sub"] != expected_caller:
        raise ValueError("Token caller mismatch")
    return payload
```

### 24.6 Graceful Degradation for Remote Agent Failures

```python
async def call_remote_agent(agent_card: dict, input: dict, fallback_fn=None):
    try:
        result = await a2a_client.invoke(
            endpoint=agent_card["endpoint"],
            input=input,
            timeout=agent_card.get("timeout_seconds", 60)
        )
        return result
    except asyncio.TimeoutError:
        if fallback_fn:
            return await fallback_fn(input)   # local fallback
        raise
    except (ConnectionError, aiohttp.ClientError) as e:
        # Inject error into agent context rather than crashing
        return {
            "status": "failed",
            "error": f"Remote agent unavailable: {e}",
            "artifacts": {}
        }
```

**Design principle:** Remote agent failures should inject a structured error result into the
orchestrator's tool result slot — never crash the orchestrator session. The orchestrator can
then decide to retry, use a fallback, or ask the user.

---


---

*[← Sessions, Cost & IDE](01-sessions-cost-ide.md) | [Next: Extensions, SDK & MCP →](01-extensions-sdk-mcp.md)*
