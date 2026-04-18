# Authentication — Overview, API Keys & Subscription Plans

> Sections: Auth Methods Overview · API Key Authentication · Subscription Plans & Rate Limits

Part of the [Authentication Guide](02-auth-oauth.md) series.

# Authentication & API Reference for Agent Builders

> Complete authentication reference for Anthropic API, OAuth 2.0 PKCE, AWS Bedrock, Google Vertex AI, and multi-provider abstraction.
> Covers: API keys, OAuth PKCE flows in TypeScript/Python/Rust/Kotlin/C++23, token management, OS keychain storage, rate limits, streaming SSE, error handling, extended thinking.

---

## 1. Auth Methods Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Anthropic Auth Methods                           │
├─────────────────┬───────────────────────────────────────────────────┤
│ Method          │ Use Case                                          │
├─────────────────┼───────────────────────────────────────────────────┤
│ API Key         │ Direct API access, scripts, servers, CI/CD        │
│                 │ console.anthropic.com → API Keys → Create         │
├─────────────────┼───────────────────────────────────────────────────┤
│ OAuth 2.0 PKCE  │ User-facing apps, Claude.ai subscription          │
│                 │ claude.ai → Settings → Subscription               │
├─────────────────┼───────────────────────────────────────────────────┤
│ AWS Bedrock     │ Enterprise, AWS infrastructure                     │
│                 │ AWS Console → Bedrock → Model Access               │
├─────────────────┼───────────────────────────────────────────────────┤
│ Google Vertex   │ Enterprise, GCP infrastructure                    │
│                 │ GCP Console → Vertex AI → Claude models           │
└─────────────────┴───────────────────────────────────────────────────┘
```

### Decision Table

| Scenario | Recommended Auth |
|---|---|
| Personal scripts, automation, CI/CD | API Key (env var) |
| SaaS app using user's Claude.ai subscription | OAuth 2.0 PKCE |
| Enterprise app on AWS | AWS Bedrock + IAM role |
| Enterprise app on GCP | Google Vertex AI + service account |
| Internal tool, shared team key | API Key (secrets manager) |
| CLI tool distributed to end users | OAuth 2.0 PKCE |
| Offline / air-gapped environment | AWS Bedrock (private endpoint) |

---

## 2. API Key Authentication

### 2.1 Getting an API Key

1. Go to `https://console.anthropic.com`
2. Sign up / log in
3. Navigate to **API Keys** in the left sidebar
4. Click **Create Key**
5. Name it and copy the key (`sk-ant-api03-...`)

> **Security**: Keys start with `sk-ant-`. Never commit to git. Use environment variables or OS keychain.

### 2.2 Using the API Key

```bash
# Environment variable (recommended)
export ANTHROPIC_API_KEY="sk-ant-api03-..."

# Direct curl call
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-opus-4-6",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

### 2.3 API Key Storage Best Practices

```bash
# ~/.bashrc or ~/.zshrc (user-level, no version control)
export ANTHROPIC_API_KEY="sk-ant-..."

# .env file (project-level — always add to .gitignore)
ANTHROPIC_API_KEY=sk-ant-...

# Docker / CI secrets
docker run -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY myapp

# macOS Keychain (most secure for interactive tools)
security add-generic-password -s "anthropic" -a "api-key" -w "sk-ant-..."
# Retrieve:
security find-generic-password -s "anthropic" -a "api-key" -w

# AWS Secrets Manager (production servers)
aws secretsmanager create-secret \
  --name anthropic/api-key \
  --secret-string '{"ANTHROPIC_API_KEY":"sk-ant-..."}'

# HashiCorp Vault
vault kv put secret/anthropic api_key=sk-ant-...
```

### 2.4 Key Rotation Strategy

```
Rotation schedule:
  - Personal keys: every 90 days
  - Production keys: every 30 days or on any suspected compromise
  - CI/CD keys: rotate on every team member departure

Zero-downtime rotation:
  1. Create new key at console.anthropic.com
  2. Deploy new key to secrets manager
  3. Update all services (rolling deploy or config reload)
  4. Verify new key is active (check logs for 401s)
  5. Delete old key (wait 24h to confirm no more traffic)
  6. Update rotation date in key inventory
```

---

## 3. Subscription Plans & Rate Limits

```
Plan          │ Context  │ Rate Limits              │ Price
──────────────┼──────────┼──────────────────────────┼──────────────
Free (API)    │ 200K     │ 60 req/min (Tier 1)       │ Pay-per-token
              │          │ 40K input / 8K output tok │
──────────────┼──────────┼──────────────────────────┼──────────────
Claude Pro    │ 200K     │ 5× free limits            │ $20/month
(claude.ai)   │          │ Via OAuth subscription    │
──────────────┼──────────┼──────────────────────────┼──────────────
Claude Max    │ 200K     │ 20×-50× higher            │ $100-$200/mo
(claude.ai)   │          │ Via OAuth subscription    │
──────────────┼──────────┼──────────────────────────┼──────────────
Claude Team   │ 200K     │ Per-member limits         │ $30/user/mo
(claude.ai)   │          │ Shared workspace          │
──────────────┼──────────┼──────────────────────────┼──────────────
API Tier 1    │ 200K     │ 60 req/min                │ Pay-per-token
(console)     │          │ 40K input / 8K output tok │
──────────────┼──────────┼──────────────────────────┼──────────────
API Tier 2    │ 200K     │ 2,000 req/min             │ After $40 spend
(console)     │          │ 400K input / 80K output   │
──────────────┼──────────┼──────────────────────────┼──────────────
API Tier 3    │ 200K     │ 4,000 req/min             │ After $200 spend
(console)     │          │ 800K input / 160K output  │
──────────────┼──────────┼──────────────────────────┼──────────────
API Tier 4    │ 200K     │ 4,000 req/min             │ After $4,000 spend
(console)     │          │ Custom negotiated         │
──────────────┼──────────┼──────────────────────────┼──────────────
Enterprise    │ 200K+    │ Custom limits             │ Custom pricing
(Bedrock/     │          │ SOC2, HIPAA, BAA          │
 Vertex)      │          │                           │
```

### Rate Limit Headers (returned with every API response)

```
x-ratelimit-limit-requests: 60
x-ratelimit-limit-tokens: 40000
x-ratelimit-remaining-requests: 59
x-ratelimit-remaining-tokens: 39000
x-ratelimit-reset-requests: 2026-04-18T10:00:01Z
x-ratelimit-reset-tokens: 2026-04-18T10:00:01Z
retry-after: 30          ← only present on 429 responses
```

### Retry-After Handling

Always read `retry-after` before deciding how long to sleep. Fall back to exponential backoff if the header is missing.

```python
import time

def get_retry_delay(headers: dict, attempt: int) -> float:
    retry_after = headers.get("retry-after")
    if retry_after:
        return float(retry_after)
    return min(2 ** attempt, 60)   # cap at 60 seconds
```

---


---

*[Next: OAuth & Token Management →](02-auth-oauth-token-keychain.md)*
