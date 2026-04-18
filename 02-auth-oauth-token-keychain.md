# Authentication — OAuth PKCE, Token Manager & Keychain Storage

> Sections: OAuth 2.0 PKCE Flow · Token Manager · OS Keychain Storage

Part of the [Authentication Guide](02-auth-overview-apikey.md) series.

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


---

*[← Auth Overview & API Keys](02-auth-overview-apikey.md) | [Next: Cloud Providers & API Reference →](02-auth-cloud-api-reference.md)*
