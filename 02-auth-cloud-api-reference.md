# Authentication — Cloud Providers, API Reference & Error Handling

> Sections: AWS Bedrock · Google Vertex AI · API Core Reference · Extended Thinking · Streaming SSE · Error Handling

Part of the [Authentication Guide](02-auth-overview-apikey.md) series.

## 7. AWS Bedrock Integration

### 7.1 Setup

```bash
# 1. Enable model access in AWS Console
# AWS Console → Amazon Bedrock → Model access → Request access to Claude

# 2. Configure AWS credentials (choose one method)
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_DEFAULT_REGION="us-east-1"
# Or: use AWS SSO, IAM roles, or instance profiles (preferred in production)

# 3. Install SDK
pip install anthropic[bedrock]     # Python
# or: pip install boto3
```

### 7.2 API Call via Bedrock (Python boto3)

```python
import boto3
import json

# SigV4 authentication is handled automatically by boto3
bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')

response = bedrock.invoke_model(
    modelId='anthropic.claude-opus-4-6-20250514-v1:0',
    contentType='application/json',
    accept='application/json',
    body=json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1024,
        "messages": [{"role": "user", "content": "Hello"}]
    })
)
result = json.loads(response['body'].read())
print(result['content'][0]['text'])

# Streaming
response = bedrock.invoke_model_with_response_stream(
    modelId='anthropic.claude-opus-4-6-20250514-v1:0',
    contentType='application/json',
    accept='application/json',
    body=json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1024,
        "messages": [{"role": "user", "content": "Tell me a story"}]
    })
)
for event in response['body']:
    chunk = json.loads(event['chunk']['bytes'])
    if chunk.get('type') == 'content_block_delta':
        print(chunk['delta'].get('text', ''), end='', flush=True)
```

### 7.3 Via anthropic-sdk bedrock wrapper

```python
import anthropic

client = anthropic.AnthropicBedrock(
    aws_region="us-east-1",
    # aws_access_key / aws_secret_key optional — falls back to env vars or IAM role
)

message = client.messages.create(
    model="anthropic.claude-opus-4-6-20250514-v1:0",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}]
)
print(message.content[0].text)
```

### 7.4 Bedrock Model IDs

```
anthropic.claude-opus-4-6-20250514-v1:0
anthropic.claude-sonnet-4-5-20251001-v1:0
anthropic.claude-haiku-4-5-20251001-v1:0
anthropic.claude-3-5-sonnet-20241022-v2:0   (legacy)
anthropic.claude-3-haiku-20240307-v1:0      (legacy)
```

---

## 8. Google Vertex AI Integration

### 8.1 Setup

```bash
# 1. Enable Vertex AI API
gcloud services enable aiplatform.googleapis.com

# 2. Authenticate (ADC — Application Default Credentials)
gcloud auth application-default login

# 3. Set project and region
export GOOGLE_CLOUD_PROJECT="my-project"
export GOOGLE_CLOUD_LOCATION="us-east5"

# 4. Install SDK
pip install anthropic[vertex]
```

### 8.2 API Call via Vertex (Python)

```python
import anthropic

client = anthropic.AnthropicVertex(
    project_id="my-project",   # or read from GOOGLE_CLOUD_PROJECT
    region="us-east5",
)

message = client.messages.create(
    model="claude-opus-4-6@20250514",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}]
)
print(message.content[0].text)
```

### 8.3 Vertex AI Model IDs

```
claude-opus-4-6@20250514
claude-sonnet-4-5@20251001
claude-haiku-4-5@20251001
```

---

## 9. Using OAuth Token with Claude API

```bash
# Use Bearer token instead of x-api-key header
curl https://api.anthropic.com/v1/messages \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: oauth-2025-04-20" \
  -H "content-type: application/json" \
  -d '{"model": "claude-opus-4-6", "max_tokens": 1024, "messages": [...]}'

# ANTHROPIC_AUTH_TOKEN env var used by claude-code (includes "Bearer " prefix)
export ANTHROPIC_AUTH_TOKEN="Bearer $ACCESS_TOKEN"
```

Both auth modes are supported by the API:
- `x-api-key: sk-ant-...` — API key mode
- `Authorization: Bearer ...` — OAuth token mode (requires `anthropic-beta: oauth-2025-04-20`)

---

## 10. Anthropic API Core Reference

### Endpoints

```
Base URL: https://api.anthropic.com

POST /v1/messages                          → Create message (streaming or blocking)
POST /v1/messages/count_tokens             → Count tokens without inference
POST /v1/messages/batches                  → Create async batch (up to 10K requests)
GET  /v1/messages/batches/<id>             → Get batch status
GET  /v1/messages/batches/<id>/results     → Stream batch results (JSONL)
POST /v1/models                            → List available models
GET  /v1/models/<id>                       → Get model details
```

### Required Headers

```
x-api-key: sk-ant-...         (or Authorization: Bearer ...)
anthropic-version: 2023-06-01  (current stable version identifier)
content-type: application/json
```

### Optional Headers

```
anthropic-beta: <feature1>,<feature2>    (comma-separated beta feature flags)
x-api-key-id: <key-name>                 (for per-key analytics in console)
```

### Current Beta Feature Flags (as of 2026-04-18)

```
oauth-2025-04-20           → Enable OAuth bearer token auth
prompt-caching-2024-07-31  → Enable cache_control on system/messages
interleaved-thinking-2025-05-14 → Extended thinking in tool use loops
```

### Model IDs (as of 2026-04-18)

```
claude-opus-4-6           → Most capable, 200K context, 32K output
claude-sonnet-4-6         → Balanced performance/cost, 200K context
claude-haiku-4-5-20251001 → Fastest/cheapest, 200K context

Aliases (always latest stable):
  claude-opus-latest
  claude-sonnet-latest
  claude-haiku-latest
```

### Pricing (per million tokens, approximate)

```
Model           │ Input  │ Output │ Cache Read │ Cache Write
────────────────┼────────┼────────┼────────────┼────────────
Claude Opus 4   │ $15    │ $75    │ $1.50      │ $18.75
Claude Sonnet 4 │ $3     │ $15    │ $0.30      │ $3.75
Claude Haiku 4  │ $0.80  │ $4     │ $0.08      │ $1.00
```

### Prompt Caching with cache_control

Place `cache_control: {"type": "ephemeral"}` at the boundary between stable and dynamic content. The API caches everything up to (and including) the last cache-control marker.

```json
{
  "system": [
    {
      "type": "text",
      "text": "[Large static system prompt — tools, rules, examples]",
      "cache_control": {"type": "ephemeral"}
    },
    {
      "type": "text",
      "text": "[Dynamic context — date, cwd, git status — NOT cached]"
    }
  ],
  "messages": [...],
  "model": "claude-opus-4-6",
  "max_tokens": 4096
}
```

Cache behavior:
- TTL: 5 minutes per session (ephemeral)
- Minimum cacheable prefix: 1,024 tokens (Opus/Sonnet), 2,048 (Haiku)
- Response usage: `usage.cache_read_input_tokens`, `usage.cache_creation_input_tokens`
- Cache hit: 90% cost reduction on those tokens
- Best practice: put TOOLS, static instructions, and examples before the boundary; date/cwd/git status after

---

## 11. Extended Thinking

Extended thinking lets the model reason step-by-step before producing a final answer. The thinking tokens are billed at the input rate.

### Request Format

```json
{
  "model": "claude-opus-4-6",
  "max_tokens": 16000,
  "thinking": {
    "type": "enabled",
    "budget_tokens": 10000
  },
  "messages": [{"role": "user", "content": "Prove that sqrt(2) is irrational."}]
}
```

### Response Structure

```json
{
  "id": "msg_...",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "thinking",
      "thinking": "Let me reason through this carefully.\n\nAssume sqrt(2) = p/q in lowest terms..."
    },
    {
      "type": "text",
      "text": "Here is the proof by contradiction: ..."
    }
  ],
  "stop_reason": "end_turn",
  "usage": {
    "input_tokens": 15,
    "output_tokens": 3250
  }
}
```

### Constraints

```
budget_tokens: must be >= 1,024
max_tokens: must exceed budget_tokens (model needs room for the final answer)
Thinking tokens billed at: input rate
Incompatible with: temperature, top_p, top_k parameters
Tool use + thinking: supported with anthropic-beta: interleaved-thinking-2025-05-14
```

---

## 12. Streaming SSE

All streaming responses use Server-Sent Events (SSE). Each event has an `event:` line followed by a `data:` JSON line.

### Python SDK (recommended)

```python
import anthropic

client = anthropic.Anthropic()

with client.messages.stream(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Tell me a story"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
    
    message = stream.get_final_message()
    print(f"\nTokens: {message.usage}")
```

### Raw SSE Event Types

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_abc","type":"message",
       "role":"assistant","content":[],"model":"claude-opus-4-6",
       "stop_reason":null,"usage":{"input_tokens":25,"output_tokens":1}}}

event: content_block_start
data: {"type":"content_block_start","index":0,
       "content_block":{"type":"text","text":""}}

event: ping
data: {"type":"ping"}

event: content_block_delta
data: {"type":"content_block_delta","index":0,
       "delta":{"type":"text_delta","text":"Hello"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,
       "delta":{"type":"text_delta","text":", world!"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: content_block_start
data: {"type":"content_block_start","index":1,
       "content_block":{"type":"tool_use","id":"toolu_xyz","name":"bash","input":{}}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,
       "delta":{"type":"input_json_delta","partial_json":"{\"command\":"}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,
       "delta":{"type":"input_json_delta","partial_json":"\"ls -la\"}"}}

event: content_block_stop
data: {"type":"content_block_stop","index":1}

event: message_delta
data: {"type":"message_delta",
       "delta":{"stop_reason":"tool_use","stop_sequence":null},
       "usage":{"output_tokens":42}}

event: message_stop
data: {"type":"message_stop"}
```

### SSE Parsing Pattern

```python
# Accumulate partial_json for tool inputs
current_tool = None
current_json = ""

for event in sse_stream:
    match event["type"]:
        case "content_block_start":
            cb = event["content_block"]
            if cb["type"] == "tool_use":
                current_tool = {"id": cb["id"], "name": cb["name"]}
                current_json = ""
        case "content_block_delta":
            delta = event["delta"]
            if delta["type"] == "text_delta":
                yield ("text", delta["text"])
            elif delta["type"] == "input_json_delta":
                current_json += delta["partial_json"]
        case "content_block_stop":
            if current_tool:
                current_tool["input"] = json.loads(current_json)
                yield ("tool_use", current_tool)
                current_tool = None
        case "message_delta":
            yield ("stop_reason", event["delta"]["stop_reason"])
```

---

## 13. Error Handling

### Status Codes

```
400 → invalid_request_error    Bad parameters — fix the request, do not retry
401 → authentication_error     Bad or missing API key — check credentials
403 → permission_error         No access to this model or feature
404 → not_found_error          Resource not found (model ID typo?)
413 → request_too_large        Reduce content size (context too large)
422 → invalid_request_error    Unprocessable entity — check request schema
429 → rate_limit_error         Rate limited — use retry-after header
500 → api_error                Anthropic server error — retry with backoff
529 → overloaded_error         High load — retry with longer backoff
```

### Python Retry with Exponential Backoff

```python
import time
import anthropic
from anthropic import APIStatusError, RateLimitError, APIConnectionError

client = anthropic.Anthropic()

def call_with_retry(messages: list, max_retries: int = 5, model: str = "claude-opus-4-6"):
    """Handles 429, 529, 500, and connection errors with proper backoff."""
    for attempt in range(max_retries):
        try:
            return client.messages.create(
                model=model,
                max_tokens=4096,
                messages=messages
            )
        
        except RateLimitError as e:
            # Always respect the retry-after header on 429
            retry_after = int(e.response.headers.get("retry-after", 30))
            print(f"Rate limited (attempt {attempt+1}/{max_retries}). "
                  f"Waiting {retry_after}s...")
            time.sleep(retry_after)
        
        except APIConnectionError:
            # Network error — exponential backoff
            wait = min(2 ** attempt, 60)
            print(f"Connection error (attempt {attempt+1}). Waiting {wait}s...")
            time.sleep(wait)
        
        except APIStatusError as e:
            if e.status_code == 529:
                # Overloaded — longer backoff, optionally switch model
                wait = min(2 ** (attempt + 2), 120)
                print(f"Overloaded (attempt {attempt+1}). Waiting {wait}s...")
                time.sleep(wait)
            elif e.status_code == 500:
                wait = min(2 ** attempt, 30)
                print(f"Server error (attempt {attempt+1}). Waiting {wait}s...")
                time.sleep(wait)
            else:
                # 400, 401, 403, 404, 413, 422 → do not retry
                raise
    
    raise RuntimeError(f"All {max_retries} attempts failed")
```

---


---

*[← OAuth & Token Management](02-auth-oauth-token-keychain.md) | [Next: Multi-Provider Routing →](02-auth-multiprovider.md)*
