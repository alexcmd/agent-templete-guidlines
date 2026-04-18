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

## 4. OAuth 2.0 PKCE Flow (Claude.ai Subscription)

This flow enables subscription-based auth instead of an API key. The access token is then sent as `Authorization: Bearer <token>` and the API beta header `anthropic-beta: oauth-2025-04-20` is required.

### 4.1 Complete Sequence Diagram

```
User                App                      claude.ai
  │                  │                           │
  │ Launch login     │                           │
  │─────────────────>│                           │
  │                  │ generatePKCE()            │
  │                  │  code_verifier = random(32 bytes)
  │                  │  code_challenge = BASE64URL(SHA256(verifier))
  │                  │                           │
  │                  │ Build authorization URL   │
  │                  │  GET /oauth/authorize     │
  │                  │   ?client_id=CLIENT_ID    │
  │                  │   &redirect_uri=localhost:PORT
  │                  │   &response_type=code     │
  │                  │   &code_challenge=...     │
  │                  │   &code_challenge_method=S256
  │                  │   &scope=openid profile email claude-max:inference
  │                  │   &state=RANDOM_STATE     │
  │                  │                           │
  │  Open browser ───>──────────────────────────>│
  │                  │                           │
  │  User logs in ───────────────────────────────│
  │  Grants consent ─────────────────────────────│
  │                  │                           │
  │                  │<── Redirect to localhost ─│
  │                  │    ?code=AUTH_CODE        │
  │                  │    &state=RANDOM_STATE    │
  │                  │                           │
  │                  │ POST /oauth/token         │
  │                  │  code=AUTH_CODE           │
  │                  │  code_verifier=...        │
  │                  │  grant_type=authorization_code
  │                  │                           │
  │                  │<── access_token ──────────│
  │                  │    refresh_token          │
  │                  │    expires_in: 3600       │
  │<── Success ──────│                           │
  │                  │                           │
  │ (Later, token expiry approaching)            │
  │                  │                           │
  │                  │ POST /oauth/token         │
  │                  │  grant_type=refresh_token │
  │                  │  refresh_token=...        │
  │                  │<── new access_token ──────│
```

### 4.2 Implementation (TypeScript)

```typescript
import * as crypto from 'crypto';
import * as http from 'http';

// Constants (from claude-code source: constants/oauth.ts)
const CLIENT_ID = 'CLAUDE_CODE_CLIENT_ID'; // actual ID from Anthropic
const AUTHORIZE_URL = 'https://claude.ai/oauth/authorize';
const TOKEN_URL = 'https://claude.ai/oauth/token';
const SCOPES = ['openid', 'profile', 'email', 'claude-max:inference'];

// Step 1: Generate PKCE pair
function generatePKCE() {
  const codeVerifier = crypto.randomBytes(32).toString('base64url');
  const codeChallenge = crypto
    .createHash('sha256')
    .update(codeVerifier)
    .digest('base64url');
  return { codeVerifier, codeChallenge };
}

// Step 2: Start local callback server
function startCallbackServer(port: number): Promise<string> {
  return new Promise((resolve, reject) => {
    const server = http.createServer((req, res) => {
      const url = new URL(req.url!, `http://localhost:${port}`);
      const code = url.searchParams.get('code');
      res.writeHead(200);
      res.end('Authentication successful! You can close this window.');
      server.close();
      if (code) resolve(code);
      else reject(new Error('No code in callback'));
    });
    server.listen(port);
  });
}

// Step 3: Exchange code for tokens
async function exchangeCodeForTokens(
  code: string,
  codeVerifier: string,
  redirectUri: string
) {
  const response = await fetch(TOKEN_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      grant_type: 'authorization_code',
      code,
      code_verifier: codeVerifier,
      redirect_uri: redirectUri,
      client_id: CLIENT_ID,
    }),
  });
  return response.json() as Promise<{
    access_token: string;
    refresh_token: string;
    expires_in: number;
    token_type: 'Bearer';
  }>;
}

// Step 4: Refresh token
async function refreshAccessToken(refreshToken: string) {
  const response = await fetch(TOKEN_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      grant_type: 'refresh_token',
      refresh_token: refreshToken,
      client_id: CLIENT_ID,
    }),
  });
  return response.json();
}

// Full flow
async function authenticateWithClaude() {
  const { codeVerifier, codeChallenge } = generatePKCE();
  const state = crypto.randomBytes(16).toString('hex');
  const port = 54321;
  const redirectUri = `http://localhost:${port}/callback`;

  const authUrl = new URL(AUTHORIZE_URL);
  authUrl.searchParams.set('client_id', CLIENT_ID);
  authUrl.searchParams.set('redirect_uri', redirectUri);
  authUrl.searchParams.set('response_type', 'code');
  authUrl.searchParams.set('code_challenge', codeChallenge);
  authUrl.searchParams.set('code_challenge_method', 'S256');
  authUrl.searchParams.set('scope', SCOPES.join(' '));
  authUrl.searchParams.set('state', state);

  console.log(`Open: ${authUrl.toString()}`);
  // open(authUrl.toString()); // use 'open' package

  const [code] = await Promise.all([
    startCallbackServer(port),
  ]);

  return exchangeCodeForTokens(code, codeVerifier, redirectUri);
}
```

### 4.3 Implementation (Python)

```python
import hashlib
import base64
import secrets
import json
import urllib.parse
import http.server
import threading
from urllib.request import urlopen, Request
from urllib.parse import urlencode

CLIENT_ID = "CLAUDE_CODE_CLIENT_ID"
AUTHORIZE_URL = "https://claude.ai/oauth/authorize"
TOKEN_URL = "https://claude.ai/oauth/token"
SCOPES = ["openid", "profile", "email", "claude-max:inference"]

def generate_pkce():
    code_verifier = secrets.token_urlsafe(32)
    digest = hashlib.sha256(code_verifier.encode()).digest()
    code_challenge = base64.urlsafe_b64encode(digest).rstrip(b'=').decode()
    return code_verifier, code_challenge

def get_auth_code(port=54321):
    """Starts local server and captures OAuth callback code."""
    auth_code = {"value": None}
    
    class Handler(http.server.BaseHTTPRequestHandler):
        def do_GET(self):
            params = dict(urllib.parse.parse_qsl(
                urllib.parse.urlparse(self.path).query
            ))
            auth_code["value"] = params.get("code")
            self.send_response(200)
            self.end_headers()
            self.wfile.write(b"Authentication successful! Close this window.")
            threading.Thread(target=self.server.shutdown).start()
        def log_message(self, *_): pass
    
    server = http.server.HTTPServer(("localhost", port), Handler)
    server.serve_forever()
    return auth_code["value"]

def exchange_code(code, code_verifier, redirect_uri):
    data = urlencode({
        "grant_type": "authorization_code",
        "code": code,
        "code_verifier": code_verifier,
        "redirect_uri": redirect_uri,
        "client_id": CLIENT_ID,
    }).encode()
    req = Request(TOKEN_URL, data=data, 
                  headers={"Content-Type": "application/x-www-form-urlencoded"})
    with urlopen(req) as r:
        return json.loads(r.read())

def authenticate():
    code_verifier, code_challenge = generate_pkce()
    state = secrets.token_hex(16)
    port = 54321
    redirect_uri = f"http://localhost:{port}/callback"
    
    params = {
        "client_id": CLIENT_ID,
        "redirect_uri": redirect_uri,
        "response_type": "code",
        "code_challenge": code_challenge,
        "code_challenge_method": "S256",
        "scope": " ".join(SCOPES),
        "state": state,
    }
    auth_url = f"{AUTHORIZE_URL}?{urlencode(params)}"
    print(f"Open: {auth_url}")
    import webbrowser; webbrowser.open(auth_url)
    
    code = get_auth_code(port)
    tokens = exchange_code(code, code_verifier, redirect_uri)
    return tokens

if __name__ == "__main__":
    tokens = authenticate()
    print(f"Access token: {tokens['access_token'][:20]}...")
    print(f"Expires in: {tokens['expires_in']}s")
```

### 4.4 Implementation (Rust)

```rust
use sha2::{Sha256, Digest};
use base64::{Engine, engine::general_purpose::URL_SAFE_NO_PAD};
use rand::Rng;

fn generate_pkce() -> (String, String) {
    let mut rng = rand::thread_rng();
    let code_verifier: String = (0..32)
        .map(|_| rng.sample(rand::distributions::Alphanumeric) as char)
        .collect();
    
    let mut hasher = Sha256::new();
    hasher.update(code_verifier.as_bytes());
    let hash = hasher.finalize();
    let code_challenge = URL_SAFE_NO_PAD.encode(hash);
    
    (code_verifier, code_challenge)
}

// Token exchange via reqwest
async fn exchange_code(
    code: &str,
    code_verifier: &str,
    redirect_uri: &str,
) -> Result<TokenResponse, reqwest::Error> {
    let client = reqwest::Client::new();
    client
        .post("https://claude.ai/oauth/token")
        .json(&serde_json::json!({
            "grant_type": "authorization_code",
            "code": code,
            "code_verifier": code_verifier,
            "redirect_uri": redirect_uri,
            "client_id": CLIENT_ID,
        }))
        .send()
        .await?
        .json::<TokenResponse>()
        .await
}
```

### 4.5 Implementation (Kotlin)

Uses `ktor-client` for HTTP, `java.security.MessageDigest` for SHA-256, and `keychain4j` for OS keychain storage.

```kotlin
// build.gradle.kts dependencies:
// implementation("io.ktor:ktor-client-core:2.3.7")
// implementation("io.ktor:ktor-client-cio:2.3.7")
// implementation("io.ktor:ktor-client-content-negotiation:2.3.7")
// implementation("io.ktor:ktor-serialization-kotlinx-json:2.3.7")
// implementation("com.github.javakeyring:java-keyring:1.0.4")

import io.ktor.client.*
import io.ktor.client.engine.cio.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.http.*
import io.ktor.serialization.kotlinx.json.*
import kotlinx.coroutines.*
import kotlinx.serialization.*
import kotlinx.serialization.json.*
import java.net.ServerSocket
import java.security.MessageDigest
import java.util.Base64
import com.github.javakeyring.Keyring

const val CLIENT_ID = "CLAUDE_CODE_CLIENT_ID"
const val AUTHORIZE_URL = "https://claude.ai/oauth/authorize"
const val TOKEN_URL = "https://claude.ai/oauth/token"
val SCOPES = listOf("openid", "profile", "email", "claude-max:inference")

@Serializable
data class TokenResponse(
    val access_token: String,
    val refresh_token: String,
    val expires_in: Int,
    val token_type: String
)

fun generatePKCE(): Pair<String, String> {
    val verifierBytes = ByteArray(32).also { java.security.SecureRandom().nextBytes(it) }
    val codeVerifier = Base64.getUrlEncoder().withoutPadding().encodeToString(verifierBytes)
    val digest = MessageDigest.getInstance("SHA-256").digest(codeVerifier.toByteArray())
    val codeChallenge = Base64.getUrlEncoder().withoutPadding().encodeToString(digest)
    return Pair(codeVerifier, codeChallenge)
}

fun captureAuthCode(port: Int): String {
    // Open a one-shot HTTP server to capture the OAuth redirect
    ServerSocket(port).use { serverSocket ->
        val socket = serverSocket.accept()
        val request = socket.getInputStream().bufferedReader().readLine() ?: ""
        val code = Regex("code=([^&\\s]+)").find(request)?.groupValues?.get(1) ?: ""
        val response = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n" +
            "<h1>Authentication successful! Close this window.</h1>"
        socket.getOutputStream().write(response.toByteArray())
        socket.close()
        return code
    }
}

suspend fun exchangeCode(
    code: String,
    codeVerifier: String,
    redirectUri: String
): TokenResponse {
    val client = HttpClient(CIO) {
        install(ContentNegotiation) { json() }
    }
    return client.post(TOKEN_URL) {
        contentType(ContentType.Application.Json)
        setBody(mapOf(
            "grant_type" to "authorization_code",
            "code" to code,
            "code_verifier" to codeVerifier,
            "redirect_uri" to redirectUri,
            "client_id" to CLIENT_ID
        ))
    }.body()
}

suspend fun authenticate(): TokenResponse {
    val (codeVerifier, codeChallenge) = generatePKCE()
    val state = java.util.UUID.randomUUID().toString().replace("-", "")
    val port = 54321
    val redirectUri = "http://localhost:$port/callback"

    val params = mapOf(
        "client_id" to CLIENT_ID,
        "redirect_uri" to redirectUri,
        "response_type" to "code",
        "code_challenge" to codeChallenge,
        "code_challenge_method" to "S256",
        "scope" to SCOPES.joinToString(" "),
        "state" to state
    )
    val query = params.entries.joinToString("&") { "${it.key}=${it.value}" }
    val authUrl = "$AUTHORIZE_URL?$query"

    println("Open: $authUrl")
    // Desktop: java.awt.Desktop.getDesktop().browse(java.net.URI(authUrl))

    val code = captureAuthCode(port)
    return exchangeCode(code, codeVerifier, redirectUri)
}

// Store tokens in OS keychain via keychain4j
fun storeTokens(tokens: TokenResponse) {
    val keyring = Keyring.create()
    val json = Json.encodeToString(tokens)
    keyring.setPassword("my-agent", "anthropic-tokens", json)
}

fun loadTokens(): TokenResponse? {
    return try {
        val keyring = Keyring.create()
        val json = keyring.getPassword("my-agent", "anthropic-tokens")
        Json.decodeFromString(json)
    } catch (e: Exception) { null }
}
```

### 4.6 Implementation (C++23)

Uses `cpp-httplib` for the local callback server and `libcurl` for token exchange. PKCE SHA-256 + base64url via OpenSSL.

```cpp
// Dependencies:
//   libcurl-dev, libssl-dev
//   cpp-httplib (header-only: github.com/yhirose/cpp-httplib)
//   nlohmann/json (header-only)
//
// Compile: g++ -std=c++23 pkce_auth.cpp -lcurl -lssl -lcrypto -o pkce_auth

#include <httplib.h>
#include <nlohmann/json.hpp>
#include <openssl/sha.h>
#include <openssl/evp.h>
#include <curl/curl.h>
#include <random>
#include <string>
#include <span>
#include <expected>
#include <print>

using json = nlohmann::json;

static constexpr std::string_view CLIENT_ID   = "CLAUDE_CODE_CLIENT_ID";
static constexpr std::string_view AUTHORIZE_URL = "https://claude.ai/oauth/authorize";
static constexpr std::string_view TOKEN_URL   = "https://claude.ai/oauth/token";
static constexpr std::string_view SCOPES      =
    "openid profile email claude-max:inference";

// ── Base64url (no padding) ────────────────────────────────────────────────

std::string base64url_encode(std::span<const unsigned char> data) {
    static constexpr char table[] =
        "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_";
    std::string out;
    out.reserve((data.size() * 4 + 2) / 3);
    size_t i = 0;
    for (; i + 2 < data.size(); i += 3) {
        uint32_t v = (data[i] << 16) | (data[i+1] << 8) | data[i+2];
        out += table[(v >> 18) & 63]; out += table[(v >> 12) & 63];
        out += table[(v >>  6) & 63]; out += table[v & 63];
    }
    if (i + 1 == data.size()) {
        uint32_t v = data[i] << 16;
        out += table[(v >> 18) & 63]; out += table[(v >> 12) & 63];
    } else if (i + 2 == data.size()) {
        uint32_t v = (data[i] << 16) | (data[i+1] << 8);
        out += table[(v >> 18) & 63]; out += table[(v >> 12) & 63];
        out += table[(v >>  6) & 63];
    }
    return out;
}

// ── PKCE Generation ────────────────────────────────────────────────────────

struct PKCEPair { std::string verifier, challenge; };

PKCEPair generate_pkce() {
    // 32 random bytes → base64url → code_verifier
    std::array<unsigned char, 32> buf{};
    std::mt19937_64 rng{std::random_device{}()};
    for (auto& b : buf)
        b = static_cast<unsigned char>(rng() & 0xFF);
    std::string verifier = base64url_encode(buf);

    // SHA-256(verifier) → base64url → code_challenge
    std::array<unsigned char, SHA256_DIGEST_LENGTH> hash{};
    SHA256(reinterpret_cast<const unsigned char*>(verifier.data()),
           verifier.size(), hash.data());
    std::string challenge = base64url_encode(hash);

    return {verifier, challenge};
}

// ── Local Callback Server ─────────────────────────────────────────────────

std::string capture_auth_code(int port) {
    httplib::Server svr;
    std::string code;

    svr.Get("/callback", [&](const httplib::Request& req, httplib::Response& res) {
        code = req.get_param_value("code");
        res.set_content("Authentication successful! Close this window.", "text/plain");
        // Shut down after first request
        svr.stop();
    });

    svr.listen("localhost", port);
    return code;
}

// ── Token Exchange via libcurl ─────────────────────────────────────────────

static size_t curl_write_cb(char* ptr, size_t size, size_t nmemb, std::string* out) {
    out->append(ptr, size * nmemb);
    return size * nmemb;
}

std::expected<json, std::string> exchange_code(
    std::string_view code,
    std::string_view verifier,
    std::string_view redirect_uri)
{
    CURL* curl = curl_easy_init();
    if (!curl) return std::unexpected("curl_easy_init failed");

    std::string body = std::format(
        "grant_type=authorization_code&code={}&code_verifier={}"
        "&redirect_uri={}&client_id={}",
        code, verifier, redirect_uri, CLIENT_ID);

    std::string response_body;
    curl_easy_setopt(curl, CURLOPT_URL, TOKEN_URL.data());
    curl_easy_setopt(curl, CURLOPT_POSTFIELDS, body.c_str());
    curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, curl_write_cb);
    curl_easy_setopt(curl, CURLOPT_WRITEDATA, &response_body);

    struct curl_slist* headers = nullptr;
    headers = curl_slist_append(headers, "Content-Type: application/x-www-form-urlencoded");
    curl_easy_setopt(curl, CURLOPT_HTTPHEADER, headers);

    CURLcode res = curl_easy_perform(curl);
    curl_slist_free_all(headers);
    curl_easy_cleanup(curl);

    if (res != CURLE_OK)
        return std::unexpected(curl_easy_strerror(res));

    return json::parse(response_body);
}

int main() {
    auto [verifier, challenge] = generate_pkce();
    int port = 54321;
    std::string redirect_uri = std::format("http://localhost:{}/callback", port);

    std::string auth_url = std::format(
        "{}?client_id={}&redirect_uri={}&response_type=code"
        "&code_challenge={}&code_challenge_method=S256&scope={}",
        AUTHORIZE_URL, CLIENT_ID, redirect_uri, challenge, SCOPES);

    std::println("Open: {}", auth_url);
    // system(("xdg-open '" + auth_url + "'").c_str());  // Linux
    // system(("open '" + auth_url + "'").c_str());      // macOS

    std::string code = capture_auth_code(port);
    auto result = exchange_code(code, verifier, redirect_uri);

    if (!result)
        std::println(stderr, "Error: {}", result.error());
    else
        std::println("access_token: {}...", result->at("access_token").get<std::string>().substr(0, 20));
}
```

---

## 5. Token Manager (Auto-Refresh)

### TypeScript

```typescript
class TokenManager {
  private accessToken: string;
  private refreshToken: string;
  private expiresAt: number;  // milliseconds

  constructor(tokens: { access_token: string; refresh_token: string; expires_in: number }) {
    this.accessToken  = tokens.access_token;
    this.refreshToken = tokens.refresh_token;
    this.expiresAt    = Date.now() + tokens.expires_in * 1000;
  }

  isExpired(): boolean {
    // Refresh 5 minutes before actual expiry to avoid race conditions
    return Date.now() >= this.expiresAt - 5 * 60 * 1000;
  }

  async getValidToken(): Promise<string> {
    if (this.isExpired()) {
      await this.refresh();
    }
    return this.accessToken;
  }

  async refresh(): Promise<void> {
    const response = await fetch(TOKEN_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        grant_type: 'refresh_token',
        refresh_token: this.refreshToken,
        client_id: CLIENT_ID,
      }),
    });
    const data = await response.json();
    this.accessToken = data.access_token;
    this.expiresAt   = Date.now() + data.expires_in * 1000;
    if (data.refresh_token) {
      // Rotate refresh token if server provides a new one
      this.refreshToken = data.refresh_token;
    }
  }

  authHeader(): string {
    return `Bearer ${this.accessToken}`;
  }
}
```

### Python Equivalent

```python
import time
import json
from urllib.request import urlopen, Request
from urllib.parse import urlencode

class TokenManager:
    REFRESH_BUFFER = 5 * 60  # seconds before expiry to refresh

    def __init__(self, tokens: dict):
        self.access_token  = tokens["access_token"]
        self.refresh_token = tokens["refresh_token"]
        self.expires_at    = time.time() + tokens["expires_in"]

    def is_expired(self) -> bool:
        return time.time() >= self.expires_at - self.REFRESH_BUFFER

    def get_valid_token(self) -> str:
        if self.is_expired():
            self.refresh()
        return self.access_token

    def refresh(self) -> None:
        data = urlencode({
            "grant_type": "refresh_token",
            "refresh_token": self.refresh_token,
            "client_id": CLIENT_ID,
        }).encode()
        req = Request(TOKEN_URL, data=data,
                      headers={"Content-Type": "application/x-www-form-urlencoded"})
        with urlopen(req) as r:
            resp = json.loads(r.read())
        self.access_token = resp["access_token"]
        self.expires_at   = time.time() + resp["expires_in"]
        if "refresh_token" in resp:
            self.refresh_token = resp["refresh_token"]  # rotate

    def auth_header(self) -> str:
        return f"Bearer {self.get_valid_token()}"
```

---

## 6. OS Keychain Storage

### TypeScript (keytar / node-keytar)

```typescript
import keytar from 'keytar';

const SERVICE = 'my-agent';
const ACCOUNT = 'anthropic-tokens';

async function saveTokens(tokens: object) {
  await keytar.setPassword(SERVICE, ACCOUNT, JSON.stringify(tokens));
}

async function loadTokens(): Promise<object | null> {
  const raw = await keytar.getPassword(SERVICE, ACCOUNT);
  return raw ? JSON.parse(raw) : null;
}

async function deleteTokens() {
  await keytar.deletePassword(SERVICE, ACCOUNT);
}
```

### Python (keyring)

```python
import keyring, json

SERVICE = "my-agent"
ACCOUNT = "anthropic-tokens"

def save_tokens(tokens: dict) -> None:
    keyring.set_password(SERVICE, ACCOUNT, json.dumps(tokens))

def load_tokens() -> dict | None:
    raw = keyring.get_password(SERVICE, ACCOUNT)
    return json.loads(raw) if raw else None

def delete_tokens() -> None:
    keyring.delete_password(SERVICE, ACCOUNT)
```

### Rust (keyring crate)

```rust
use keyring::Entry;
use serde::{Deserialize, Serialize};

fn save_tokens<T: Serialize>(tokens: &T) -> keyring::Result<()> {
    let entry = Entry::new("my-agent", "anthropic-tokens")?;
    entry.set_password(&serde_json::to_string(tokens).unwrap())?;
    Ok(())
}

fn load_tokens<T: for<'de> Deserialize<'de>>() -> keyring::Result<T> {
    let entry = Entry::new("my-agent", "anthropic-tokens")?;
    let raw = entry.get_password()?;
    Ok(serde_json::from_str(&raw).unwrap())
}

fn delete_tokens() -> keyring::Result<()> {
    let entry = Entry::new("my-agent", "anthropic-tokens")?;
    entry.delete_password()
}
```

### Kotlin (keychain4j / java-keyring)

```kotlin
import com.github.javakeyring.Keyring
import kotlinx.serialization.encodeToString
import kotlinx.serialization.decodeFromString
import kotlinx.serialization.json.Json

private const val SERVICE = "my-agent"
private const val ACCOUNT = "anthropic-tokens"

fun saveTokens(tokens: TokenResponse) {
    Keyring.create().setPassword(SERVICE, ACCOUNT, Json.encodeToString(tokens))
}

fun loadTokens(): TokenResponse? {
    return try {
        val raw = Keyring.create().getPassword(SERVICE, ACCOUNT)
        Json.decodeFromString<TokenResponse>(raw)
    } catch (_: Exception) { null }
}

fun deleteTokens() {
    try { Keyring.create().deletePassword(SERVICE, ACCOUNT) } catch (_: Exception) {}
}
```

### C++ (libsecret on Linux / Security.framework on macOS)

The simplest cross-platform approach for C++ agents is to shell out to the platform keychain CLI, or use `libsecret` on Linux and the `Security.framework` on macOS.

```cpp
#include <cstdlib>
#include <string>
#include <optional>
#include <print>

// macOS: uses 'security' CLI (ships with every Mac)
// Linux: uses 'secret-tool' from libsecret-tools package

std::optional<std::string> keychain_get(const std::string& service,
                                         const std::string& account) {
#ifdef __APPLE__
    std::string cmd = "security find-generic-password -s '" + service +
                      "' -a '" + account + "' -w 2>/dev/null";
#else
    std::string cmd = "secret-tool lookup service '" + service +
                      "' account '" + account + "' 2>/dev/null";
#endif
    FILE* pipe = popen(cmd.c_str(), "r");
    if (!pipe) return std::nullopt;
    char buf[4096] = {};
    fgets(buf, sizeof(buf), pipe);
    pclose(pipe);
    std::string result(buf);
    if (!result.empty() && result.back() == '\n')
        result.pop_back();
    return result.empty() ? std::nullopt : std::make_optional(result);
}

void keychain_set(const std::string& service, const std::string& account,
                   const std::string& value) {
#ifdef __APPLE__
    std::string cmd = "security add-generic-password -U -s '" + service +
                      "' -a '" + account + "' -w '" + value + "' 2>/dev/null";
#else
    // secret-tool reads the secret from stdin
    std::string cmd = "echo '" + value + "' | secret-tool store --label='my-agent'" +
                      " service '" + service + "' account '" + account + "' 2>/dev/null";
#endif
    std::system(cmd.c_str());
}
```

> For production C++ apps, link against `libsecret` (Linux) or `Security.framework` (macOS) directly instead of shelling out.

---

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

## 14. Multi-Provider Auth Abstraction

When building an agent that supports many providers (30+), define a `ProviderDescriptor` and select providers by scanning environment variables.

### ProviderDescriptor Schema

```python
from dataclasses import dataclass

@dataclass
class ProviderDescriptor:
    id: str              # e.g. "anthropic", "openai", "ollama"
    name: str            # human display name
    env_var: str | None  # env var containing the API key (None if no key needed)
    base_url: str        # API base URL
    no_api_key_required: bool = False  # True for local providers (ollama, lm_studio)
```

### Provider Registry (Python)

```python
PROVIDERS: list[ProviderDescriptor] = [
    ProviderDescriptor("anthropic",  "Anthropic",        "ANTHROPIC_API_KEY",
                       "https://api.anthropic.com"),
    ProviderDescriptor("openai",     "OpenAI",           "OPENAI_API_KEY",
                       "https://api.openai.com/v1"),
    ProviderDescriptor("google",     "Google Gemini",    "GOOGLE_API_KEY",
                       "https://generativelanguage.googleapis.com/v1beta"),
    ProviderDescriptor("azure",      "Azure OpenAI",     "AZURE_OPENAI_API_KEY",
                       ""),  # base_url from config
    ProviderDescriptor("groq",       "Groq",             "GROQ_API_KEY",
                       "https://api.groq.com/openai/v1"),
    ProviderDescriptor("deepseek",   "DeepSeek",         "DEEPSEEK_API_KEY",
                       "https://api.deepseek.com/v1"),
    ProviderDescriptor("mistral",    "Mistral",          "MISTRAL_API_KEY",
                       "https://api.mistral.ai/v1"),
    ProviderDescriptor("openrouter", "OpenRouter",       "OPENROUTER_API_KEY",
                       "https://openrouter.ai/api/v1"),
    ProviderDescriptor("ollama",     "Ollama (local)",   None,
                       "http://localhost:11434",
                       no_api_key_required=True),
    ProviderDescriptor("lm_studio",  "LM Studio (local)", None,
                       "http://localhost:1234/v1",
                       no_api_key_required=True),
    # ... 25+ more
]
```

### Provider Selection Precedence

```python
import os

def select_provider(preferred: str | None = None) -> ProviderDescriptor | None:
    """
    Provider selection order:
      1. Explicit --provider flag or AGENT_PROVIDER env var
      2. First provider whose env_var is set in the environment
      3. First local provider (no_api_key_required=True)
      4. None — prompt user to configure
    """
    if preferred:
        for p in PROVIDERS:
            if p.id == preferred:
                return p

    # Check configured env var
    provider_id = os.environ.get("AGENT_PROVIDER")
    if provider_id:
        for p in PROVIDERS:
            if p.id == provider_id:
                return p

    # Auto-detect from env vars
    for p in PROVIDERS:
        if p.env_var and os.environ.get(p.env_var):
            return p

    # Fall back to local providers
    for p in PROVIDERS:
        if p.no_api_key_required:
            return p

    return None

def get_api_key(provider: ProviderDescriptor) -> str | None:
    if provider.no_api_key_required:
        return None
    return os.environ.get(provider.env_var or "") if provider.env_var else None
```

### Using Bearer Token Instead of API Key (OAuth Mode)

```python
def get_auth_headers(provider: ProviderDescriptor,
                     token_manager: TokenManager | None = None) -> dict:
    """Returns the correct auth headers for a provider."""
    if provider.id == "anthropic":
        if token_manager:
            # OAuth mode
            return {
                "Authorization": token_manager.auth_header(),
                "anthropic-beta": "oauth-2025-04-20",
                "anthropic-version": "2023-06-01",
            }
        else:
            return {
                "x-api-key": get_api_key(provider),
                "anthropic-version": "2023-06-01",
            }
    else:
        # OpenAI-compatible providers
        return {"Authorization": f"Bearer {get_api_key(provider)}"}
```

---

*Last updated: 2026-04-18. Model IDs and pricing reflect availability at that date. Check console.anthropic.com for the latest.*
