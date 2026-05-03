# IDE Integration — Source-Verified Deep Dive

> Sections: Architecture overview · IDE detection · Lockfile protocol · MCP transport · RPC calls · Diff viewer · Auto-connect · WSL path conversion · Environment variables · Reproduction guide

Part of the [Universal Agent Architecture](01-overview.md) series.

*Source-verified against `/home/deck/Projects/claude-code/src` — all file:line references are authoritative.*

---

## 1. Architecture Overview

Claude Code's IDE integration is **bidirectional over MCP**. The IDE extension acts as an MCP server; the CLI registers it as a special internal MCP client. This gives the agent full RPC access to the editor at tool-call granularity.

```
┌─────────────────────────────────────────────────┐
│                  USER'S MACHINE                  │
│                                                  │
│  ┌──────────────────┐    lockfile (~/.claude/    │
│  │   IDE Extension  │◄── ide/{port}.lock)        │
│  │  (VS Code /      │                            │
│  │   JetBrains)     │    JSON: port, transport,  │
│  │                  │    workspace, pid, token    │
│  │  MCP Server      │                            │
│  │  ┌────────────┐  │                            │
│  │  │ /sse       │  │◄──── SSE transport         │
│  │  │ ws://      │  │◄──── WebSocket transport   │
│  │  └────────────┘  │                            │
│  └──────────────────┘                            │
           ▲                                        │
           │ TCP :port                              │
           │                                        │
  ┌────────┴──────────┐                            │
  │  Claude Code CLI  │                            │
  │                   │                            │
  │  reads lockfile   │                            │
  │  → resolves host  │                            │
  │  → TCP check      │                            │
  │  → registers as   │                            │
  │    sse-ide / ws-  │                            │
  │    ide MCP server │                            │
  │                   │                            │
  │  callIdeRpc()     │                            │
  │  → openDiff       │                            │
  │  → closeAllDiff   │                            │
  │  → closeTab       │                            │
  └───────────────────┘                            │
└─────────────────────────────────────────────────┘
```

Key design decisions:
- The IDE extension owns the **server**; the CLI owns the **client** — inverted from what you might expect
- Communication is via the **MCP protocol** (JSON-RPC), not a bespoke IPC
- IDE instances are discovered via **lockfiles**, not service discovery or config

---

## 2. IDE Detection

**Source:** `utils/ide.ts`, `utils/env.ts`

Detection uses two independent strategies that are OR'd together: terminal environment variables (fast, zero-cost) and process-list scanning (slower, fallback).

### 2.1 Terminal Environment Detection

When Claude Code is launched from within an IDE's integrated terminal, the IDE sets environment variables that survive into child processes:

| Env Variable | IDE |
|---|---|
| `CURSOR_TRACE_ID` | Cursor |
| `VSCODE_GIT_ASKPASS_MAIN` | VS Code, Cursor, Windsurf (any code-family) |
| `TERMINAL_EMULATOR=JetBrains-JediTerm` | All JetBrains IDEs |
| `__CFBundleIdentifier` (macOS) | VS Code, Cursor, Windsurf, Android Studio, JetBrains |
| `TERM_PROGRAM` | Generic IDE/terminal |

These are checked first. If matched, no process scan is needed.

### 2.2 Process-List Scanning

When not launched from an IDE terminal, Claude Code scans running processes:

**Linux / macOS:**
```bash
ps aux | grep -i "<pattern>"
```

Patterns per IDE:
- VS Code: `"Visual Studio Code"` / `"code"`
- Cursor: `"Cursor Helper"` / `"cursor"`
- Windsurf: `"windsurf"`
- JetBrains: `"idea"`, `"pycharm"`, `"webstorm"`, `"phpstorm"`, `"rubymine"`, `"clion"`, `"goland"`, `"rider"`, `"datagrip"`, `"appcode"`, `"dataspell"`, `"aqua"`, `"gateway"`, `"fleet"`, `"androidstudio"`

**Windows:**
```cmd
tasklist | findstr /i "Code.exe cursor.exe windsurf.exe idea64.exe ..."
```

### 2.3 Supported IDE Enum

```typescript
// utils/ide.ts ~lines 102-120
type SupportedIDE =
  | 'cursor'
  | 'windsurf'
  | 'vscode'
  | 'pycharm'
  | 'intellij'
  | 'webstorm'
  | 'phpstorm'
  | 'rubymine'
  | 'clion'
  | 'goland'
  | 'rider'
  | 'datagrip'
  | 'appcode'
  | 'dataspell'
  | 'aqua'
  | 'gateway'
  | 'fleet'
  | 'androidstudio'
```

---

## 3. Lockfile Discovery Protocol

**Source:** `utils/ide.ts` lines 73–90, 346–393

### 3.1 Lockfile Location

```
~/.claude/ide/{port}.lock
```

On WSL with a Windows IDE, additional paths are also checked to find the Windows-side lockfile.

### 3.2 Lockfile Schema

```typescript
interface IDELockfile {
  workspaceFolders: string[]    // absolute paths open in IDE
  pid: number                   // IDE process PID
  ideName: string               // human-readable IDE name
  transport: 'ws' | 'sse'      // WebSocket or SSE
  runningInWindows: boolean     // true when IDE is on Windows host (WSL)
  authToken?: string            // optional bearer token
}
```

The filename encodes the port: `8765.lock` means the MCP server listens on `:8765`.

### 3.3 Lockfile Validation

Before connecting, Claude Code validates:

1. **Workspace match** — `workspaceFolders` must contain the current working directory (or a parent). Case-insensitive on Windows drive letters. NFC-normalized on macOS.
2. **Process liveness** — `pid` is checked to confirm the IDE process is still running.
3. **TCP reachability** — opens a TCP socket to `host:port` with a **500 ms timeout**. Fails silently (returns null connection) if the port is not listening.

The env var `CLAUDE_CODE_IDE_SKIP_VALID_CHECK=1` bypasses workspace validation (useful in testing).

### 3.4 Multiple IDE Instances

If multiple lockfiles exist (e.g., two VS Code windows), the `/ide` command (`commands/ide/ide.tsx`) shows each instance with its workspace folder list so the user can select one. The disambiguator renders workspace folder names in the TUI.

---

## 4. Network Transport

**Source:** `services/mcp/types.ts` lines 68–87, 128–129

Two MCP transports are supported, selected per lockfile:

### 4.1 SSE Transport (`sse-ide`)

```typescript
interface SSEIDEServerConfig {
  type: 'sse-ide'
  url: string                    // http://host:port/sse
  ideName: string
  ideRunningInWindows?: boolean  // WSL awareness
}
```

The agent connects to `http://<host>:<port>/sse` and receives a standard SSE stream of MCP messages.

### 4.2 WebSocket Transport (`ws-ide`)

```typescript
interface WSIDEServerConfig {
  type: 'ws-ide'
  url: string                    // ws://host:port
  ideName: string
  authToken?: string             // sent as Authorization: Bearer <token>
  ideRunningInWindows?: boolean
}
```

### 4.3 Host Resolution for WSL

When `runningInWindows: true` in the lockfile and Claude Code is running in WSL, the host is resolved to the Windows host IP (obtained via `ip route show default` or `/etc/resolv.conf nameserver`) rather than `localhost`. This allows the WSL CLI to reach the IDE running on the Windows side.

`CLAUDE_CODE_IDE_HOST_OVERRIDE` overrides the resolved host entirely.

### 4.4 Internal-Only Server Types

Both `sse-ide` and `ws-ide` are **internal types** — they never appear in user-facing MCP server lists (`/mcp list`) and are excluded from serialization to config files. They are created transiently in memory when a lockfile is found.

---

## 5. RPC Protocol

**Source:** `services/mcp/client.ts` lines 2116–2130, `hooks/useDiffInIDE.ts`, `utils/ide.ts`

### 5.1 Core RPC Function

```typescript
// services/mcp/client.ts
export async function callIdeRpc(
  toolName: string,
  args: Record<string, unknown>,
  client: ConnectedMCPServer,
): Promise<string | ContentBlockParam[] | undefined>
```

This wraps a standard MCP tool call but targets the IDE MCP server instead of a user-configured server. Errors are swallowed and return `undefined`; callers treat undefined as "IDE unavailable, skip."

### 5.2 RPC Methods

**`openDiff`** — opens the IDE's built-in diff viewer for a file edit:
```typescript
await callIdeRpc('openDiff', {
  old_file_path: string,     // original file path (may be temp file)
  new_file_path: string,     // new file path
  new_file_contents: string, // proposed content
  tab_name: string,          // UUID-suffixed tab identifier
}, ideClient)
```

The tab name contains a UUID (`${basename}-${uuid4()}`) so multiple concurrent diffs don't collide.

**`closeAllDiffTabs`** — batch-closes every open diff tab:
```typescript
await callIdeRpc('closeAllDiffTabs', {}, ideClient)
```
Called when the user presses ESC or the agent finishes a multi-file edit.

**`closeTab`** — closes one specific tab by name:
```typescript
await callIdeRpc('closeTab', { tab_name: string }, ideClient)
```

### 5.3 IDE → CLI Notification

When the IDE extension connects, it sends an MCP notification:
```json
{
  "method": "ide_connected",
  "params": { "pid": 12345 }
}
```

The CLI uses this PID for process liveness checks on subsequent turns.

---

## 6. Diff Viewer Integration

**Source:** `hooks/useDiffInIDE.ts`

This hook intercepts file-write tool calls when an IDE is connected and `diffTool: "auto"` is configured.

### 6.1 Flow

```
Agent proposes file edit
  │
  ▼
useDiffInIDE detects IDE connection
  │
  ▼
Write new content to temp file
  │
  ▼
callIdeRpc('openDiff', {
  old_file_path: original,
  new_file_path: tempFile,
  new_file_contents: newContent,
  tab_name: `${basename}-${uuid}`
})
  │
  ├─ User accepts in IDE diff viewer
  │     → IDE sends back content (or signals accept)
  │     → Claude Code applies the edit
  │     → callIdeRpc('closeTab', tab_name)
  │
  └─ User rejects / ESC
        → Edit discarded
        → callIdeRpc('closeAllDiffTabs', {})
```

### 6.2 Concurrent Edits

Multiple files can be in-flight simultaneously (parallel tool calls). Each gets a distinct UUID tab name. Tabs are cleaned up individually on accept or all at once on ESC.

---

## 7. Auto-Connect & Extension Installation

**Source:** `utils/ide.ts` lines 1288–1348, `hooks/useIDEIntegration.tsx`

### 7.1 Auto-Connect Triggers

Auto-connect fires when **any** of these conditions is true:
- `autoConnectIde: true` in global config
- `--ide` CLI flag passed
- `CLAUDE_CODE_AUTO_CONNECT_IDE=1` env var set
- `CLAUDE_CODE_SSE_PORT` env var is set (means Claude Code was launched from inside an IDE terminal)

When triggered, `useIDEIntegration` reads all lockfiles in `~/.claude/ide/`, validates them, and registers matching ones as `sse-ide` or `ws-ide` MCP servers for the session.

### 7.2 Extension Installation

**VS Code / Cursor / Windsurf:**
```bash
code --install-extension anthropic.claude-code
```
Called programmatically when the user has VS Code in PATH and the extension is not yet installed.

**JetBrains:**
Manual installation from JetBrains Marketplace — no CLI support. Claude Code detects if the plugin is installed by checking for the lockfile; if absent, it shows a link to the marketplace.

### 7.3 Version Checking

The lockfile or extension handshake may include a version. Claude Code compares against a minimum supported extension version and warns (but does not block) if the extension is outdated.

---

## 8. WSL / Windows Path Conversion

**Source:** `utils/idePathConversion.ts`

When the CLI runs in WSL and the IDE runs on Windows, file paths must be translated in both directions.

### 8.1 Converter Interface

```typescript
interface PathConverter {
  toLocalPath(idePath: string): string    // IDE (Windows) → CLI (WSL)
  toIDEPath(localPath: string): string    // CLI (WSL) → IDE (Windows)
}
```

### 8.2 `WindowsToWSLConverter`

Uses `wslpath` utility:
```typescript
// Windows → WSL
exec(`wslpath -u "${windowsPath}"`)
// e.g. C:\Users\alice\proj  →  /mnt/c/Users/alice/proj

// WSL → Windows
exec(`wslpath -w "${wslPath}"`)
// e.g. /mnt/c/Users/alice/proj  →  C:\Users\alice\proj
```

If `wslpath` is unavailable, a manual fallback converts `C:\` → `/mnt/c/` by regex.

### 8.3 Activation Condition

The converter is instantiated when `lockfile.runningInWindows === true && isWsl()`. All paths sent to the IDE go through `toIDEPath()`; all paths received from the IDE go through `toLocalPath()`.

---

## 9. Environment Variables Reference

| Variable | Type | Purpose |
|---|---|---|
| `CLAUDE_CODE_SSE_PORT` | string (port) | Set by IDE extension when launching Claude Code from its terminal; triggers auto-connect |
| `CLAUDE_CODE_IDE_SKIP_VALID_CHECK` | `"1"` | Skip workspace-folder validation against CWD |
| `CLAUDE_CODE_IDE_HOST_OVERRIDE` | string (host) | Override resolved host for IDE connection |
| `CLAUDE_CODE_IDE_SKIP_AUTO_INSTALL` | `"1"` | Suppress automatic extension installation prompts |
| `CLAUDE_CODE_AUTO_CONNECT_IDE` | `"1"` | Force auto-connect even without `--ide` flag |
| `WSL_DISTRO_NAME` | string | Set by WSL; used to match lockfile distro scope |
| `VSCODE_GIT_ASKPASS_MAIN` | string (path) | VS Code family detection; presence signals IDE terminal |
| `CURSOR_TRACE_ID` | string | Cursor-specific trace; presence signals Cursor terminal |
| `TERMINAL_EMULATOR` | `"JetBrains-JediTerm"` | JetBrains terminal detection |

---

## 10. Key Source Files

| File | Role |
|---|---|
| `utils/ide.ts` | Core: lockfile read, IDE detection, TCP validation, RPC dispatch |
| `utils/idePathConversion.ts` | WSL ↔ Windows path conversion |
| `utils/jetbrains.ts` | JetBrains plugin install detection |
| `utils/env.ts` | Terminal-based IDE detection via env vars (lines 115–200) |
| `services/mcp/types.ts` | `sse-ide` / `ws-ide` server config schemas |
| `services/mcp/client.ts` | `callIdeRpc()` implementation |
| `hooks/useDiffInIDE.ts` | Diff viewer lifecycle hook |
| `hooks/useIDEIntegration.tsx` | Auto-detect + register IDE as MCP config |
| `commands/ide/ide.tsx` | `/ide` command — TUI for IDE selection |

---

## 11. Reproduction Guide

### 11.1 Minimal IDE Server (Python)

Implement the MCP server side — what the IDE extension must expose:

```python
# ide_mcp_server.py — minimal MCP server for IDE integration
import json, uuid, asyncio
from pathlib import Path

LOCKFILE_DIR = Path.home() / ".claude" / "ide"
PORT = 8765

async def handle_open_diff(args: dict) -> dict:
    """Show diff in editor. Return accepted content or signal rejection."""
    old_path = args["old_file_path"]
    new_contents = args["new_file_contents"]
    tab_name = args["tab_name"]
    # YOUR IDE API: open diff viewer, return accepted content
    ...
    return {"accepted": True, "content": new_contents}

async def handle_close_tab(args: dict) -> dict:
    tab_name = args["tab_name"]
    # YOUR IDE API: close the tab
    return {}

async def handle_close_all_diff_tabs(args: dict) -> dict:
    # YOUR IDE API: close all diff tabs
    return {}

TOOL_HANDLERS = {
    "openDiff": handle_open_diff,
    "closeTab": handle_close_tab,
    "closeAllDiffTabs": handle_close_all_diff_tabs,
}

def write_lockfile(port: int, workspace_folders: list[str], pid: int):
    LOCKFILE_DIR.mkdir(parents=True, exist_ok=True)
    lockfile = LOCKFILE_DIR / f"{port}.lock"
    lockfile.write_text(json.dumps({
        "workspaceFolders": workspace_folders,
        "pid": pid,
        "ideName": "MyEditor",
        "transport": "sse",            # or "ws"
        "runningInWindows": False,
        "authToken": None,
    }))
    return lockfile

def remove_lockfile(port: int):
    (LOCKFILE_DIR / f"{port}.lock").unlink(missing_ok=True)
```

### 11.2 Kotlin (Koog / JetBrains plugin)

```kotlin
// IdeMcpServer.kt
import kotlinx.serialization.json.*
import java.nio.file.Path

data class LockfileContent(
    val workspaceFolders: List<String>,
    val pid: Long,
    val ideName: String,
    val transport: String,  // "sse" | "ws"
    val runningInWindows: Boolean,
    val authToken: String? = null
)

fun writeLockfile(port: Int, content: LockfileContent) {
    val dir = Path.of(System.getProperty("user.home"), ".claude", "ide")
    dir.toFile().mkdirs()
    val file = dir.resolve("$port.lock").toFile()
    file.writeText(Json.encodeToString(content))
}

// MCP tool handler — implement in your MCP server
fun handleToolCall(name: String, args: JsonObject): JsonElement = when (name) {
    "openDiff" -> {
        val oldPath = args["old_file_path"]!!.jsonPrimitive.content
        val newContents = args["new_file_contents"]!!.jsonPrimitive.content
        val tabName = args["tab_name"]!!.jsonPrimitive.content
        // Open JetBrains diff tool
        // ApplicationManager.getApplication().invokeLater { ... }
        buildJsonObject { put("accepted", true) }
    }
    "closeTab" -> {
        val tabName = args["tab_name"]!!.jsonPrimitive.content
        // Close the specific editor tab
        buildJsonObject { }
    }
    "closeAllDiffTabs" -> buildJsonObject { }
    else -> throw IllegalArgumentException("Unknown tool: $name")
}
```

### 11.3 C++23

```cpp
// ide_mcp_server.hpp
#include <filesystem>
#include <nlohmann/json.hpp>

struct LockfileContent {
    std::vector<std::string> workspace_folders;
    int pid;
    std::string ide_name;
    std::string transport;  // "sse" | "ws"
    bool running_in_windows = false;
    std::optional<std::string> auth_token;
};

void write_lockfile(int port, const LockfileContent& content) {
    namespace fs = std::filesystem;
    auto dir = fs::path(std::getenv("HOME")) / ".claude" / "ide";
    fs::create_directories(dir);
    auto path = dir / (std::to_string(port) + ".lock");
    
    nlohmann::json j{
        {"workspaceFolders", content.workspace_folders},
        {"pid", content.pid},
        {"ideName", content.ide_name},
        {"transport", content.transport},
        {"runningInWindows", content.running_in_windows},
    };
    if (content.auth_token)
        j["authToken"] = *content.auth_token;
    
    std::ofstream(path) << j.dump();
}

// Tool dispatch — wire to your MCP server implementation
nlohmann::json handle_tool(std::string_view name, const nlohmann::json& args) {
    if (name == "openDiff") {
        auto old_path = args.at("old_file_path").get<std::string>();
        auto new_contents = args.at("new_file_contents").get<std::string>();
        auto tab_name = args.at("tab_name").get<std::string>();
        // Open diff in your editor
        return {{"accepted", true}};
    }
    if (name == "closeTab") {
        auto tab_name = args.at("tab_name").get<std::string>();
        return {};
    }
    if (name == "closeAllDiffTabs") return {};
    throw std::runtime_error("Unknown tool: " + std::string(name));
}
```

### 11.4 Integration Checklist

- [ ] Start an SSE or WebSocket MCP server on a free port
- [ ] Write `~/.claude/ide/{port}.lock` with workspace folders and transport type
- [ ] Remove lockfile on extension shutdown / IDE close
- [ ] Implement MCP tools: `openDiff`, `closeTab`, `closeAllDiffTabs`
- [ ] (Optional) On connect, send `ide_connected` notification with PID
- [ ] (WSL) Set `runningInWindows: true` when IDE is on Windows host
- [ ] (Auth) Include `authToken` in lockfile and validate `Authorization: Bearer` header

---

## 12. Interaction with the Broader Agent Loop

IDE integration hooks into two points in the agent loop:

1. **Pre-tool-call (diff preview):** `useDiffInIDE` intercepts `Write`/`Edit` tool calls before execution, redirects the proposed change to the IDE diff viewer, and waits for user accept/reject. This happens *inside* the tool-use loop, blocking the agent turn until resolved.

2. **Session startup (auto-connect):** `useIDEIntegration` runs once at CLI launch, scans lockfiles, and registers matching IDE connections as MCP servers. These servers are available to `callIdeRpc()` for the entire session.

The IDE MCP server is treated like any other MCP server internally — it participates in parallel tool execution, respects the same permission gates, and can be listed via `/mcp debug` — except it is never serialized to config and never shown in user-facing lists.

---

*[← Sessions & Cost](01-sessions-cost-ide.md) | [Extensions & SDK →](01-extensions-sdk-mcp.md)*
