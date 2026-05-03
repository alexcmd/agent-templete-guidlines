# JetBrains IDE Plugin — Deep Dive

> **Prerequisite:** Read [`01-ide-integration.md`](01-ide-integration.md) first. That document covers the lockfile discovery protocol, MCP transports (`sse-ide`/`ws-ide`), RPC methods (`openDiff`/`closeAllDiffTabs`), WSL path conversion, and the base reproduction guide. This document covers everything JetBrains-specific that sits on top of that foundation.

---

## §1 Why JetBrains Is Different

VS Code-family IDEs (Cursor, Windsurf) set unique env vars at shell spawn time, enabling synchronous compile-time detection. JetBrains IDEs expose only `TERMINAL_EMULATOR=JetBrains-JediTerm` — a single value shared across all 15 JetBrains products — so fine-grained detection requires async parent-process inspection at startup. The plugin also cannot be auto-installed; the CLI only nudges users to the JetBrains Marketplace.

```
                    ┌─────────────────────────────────────────┐
                    │          CLI startup sequence            │
                    │                                          │
                    │  1. env.ts detectTerminal()  (sync)      │
                    │     __CFBundleIdentifier → IDE type      │
                    │     TERMINAL_EMULATOR=JediTerm → 'pycharm'│
                    │                   │                      │
                    │  2. initJetBrainsDetection()  (async)    │
                    │     getAncestorCommandsAsync(pid, 10)    │
                    │     → specific IDE type, cache result    │
                    │                   │                      │
                    │  3. getTerminalWithJetBrainsDetection()  │
                    │     returns cached fine-grained type     │
                    └─────────────────────────────────────────┘
```

**Source files involved:**

| File | Role |
|------|------|
| `src/utils/env.ts` | Sync terminal detection (Phase 1) |
| `src/utils/envDynamic.ts` | Async parent-process detection (Phase 2) |
| `src/utils/jetbrains.ts` | Plugin installation detection |
| `src/utils/ide.ts` | `IdeType`, `IdeConfig`, `isJetBrainsIde()` |
| `src/utils/statusNoticeDefinitions.tsx` | Marketplace install prompt |
| `src/hooks/useIdeSelection.ts` | `selection_changed` notification handler |
| `src/hooks/useIdeLogging.ts` | `log_event` notification handler |

---

## §2 Plugin Detection — `utils/jetbrains.ts`

Claude Code checks whether the JetBrains plugin is installed by scanning well-known JetBrains configuration directories for a folder named `claude-code-jetbrains-plugin`.

### 2.1 IDE directory name mapping

```typescript
// src/utils/jetbrains.ts:9-25
const ideNameToDirMap: { [key: string]: string[] } = {
  pycharm:       ['PyCharm'],
  intellij:      ['IntelliJIdea', 'IdeaIC'],   // Ultimate + Community
  webstorm:      ['WebStorm'],
  phpstorm:      ['PhpStorm'],
  rubymine:      ['RubyMine'],
  clion:         ['CLion'],
  goland:        ['GoLand'],
  rider:         ['Rider'],
  datagrip:      ['DataGrip'],
  appcode:       ['AppCode'],
  dataspell:     ['DataSpell'],
  aqua:          ['Aqua'],
  gateway:       ['Gateway'],
  fleet:         ['Fleet'],
  androidstudio: ['AndroidStudio'],
}
```

IntelliJ has two directory variants because Ultimate (`IntelliJIdea`) and Community (`IdeaIC`) install to different paths. Android Studio gets extra paths under `Google/` on all platforms.

### 2.2 Per-OS base directories

```
macOS
  ~/Library/Application Support/JetBrains/{IDE*}/plugins/claude-code-jetbrains-plugin
  ~/Library/Application Support/{IDE*}/plugins/claude-code-jetbrains-plugin
  ~/Library/Application Support/Google/{IDE*}/...        ← Android Studio only

Windows
  %APPDATA%\JetBrains\{IDE*}\plugins\claude-code-jetbrains-plugin
  %LOCALAPPDATA%\JetBrains\{IDE*}\plugins\...
  %LOCALAPPDATA%\Google\{IDE*}\...                       ← Android Studio only

Linux
  ~/.config/JetBrains/{IDE*}/claude-code-jetbrains-plugin    ← NO plugins/ subdir
  ~/.local/share/JetBrains/{IDE*}/...                        ← NO plugins/ subdir
  ~/.{DirName}/claude-code-jetbrains-plugin                  ← legacy dotdir, e.g. ~/.PyCharm
  ~/.config/Google/{IDE*}/...                                ← Android Studio only
```

**Linux note:** JetBrains on Linux stores plugins directly under the config root — there is no `plugins/` subdirectory. `detectPluginDirectories()` omits the `plugins/` join when `platform() === 'linux'`.

**Symlink note:** `detectPluginDirectories()` accepts both directories and symbolic links (`dirent.isSymbolicLink()`). This supports [GNU stow](https://www.gnu.org/software/stow/) users who symlink their JetBrains config dirs. (`dirent.isDirectory()` is `false` for symlinks, so accepting `isSymbolicLink()` is required.)

### 2.3 Three-tier caching architecture

```
Tier 1 — Promise cache (pluginInstalledPromiseCache: Map<IdeType, Promise<boolean>>)
  Deduplicates concurrent callers: second caller gets the same in-flight promise.

Tier 2 — Result cache (pluginInstalledCache: Map<IdeType, boolean>)
  Populated once the promise resolves. isJetBrainsPluginInstalledCachedSync() reads here.

Tier 3 — Sync fallback
  Returns false until Tier 2 is populated. Used by status notice isActive() checks
  which cannot be async.
```

```typescript
// src/utils/jetbrains.ts:171-191

// Async, with optional force-refresh
export async function isJetBrainsPluginInstalledCached(
  ideType: IdeType,
  forceRefresh = false,
): Promise<boolean>

// Sync, returns false if result not yet resolved
export function isJetBrainsPluginInstalledCachedSync(
  ideType: IdeType,
): boolean {
  return pluginInstalledCache.get(ideType) ?? false
}
```

Call `isJetBrainsPluginInstalledCached(ideType)` (async) during startup to warm the cache, then use `isJetBrainsPluginInstalledCachedSync(ideType)` anywhere a synchronous check is needed.

---

## §3 Terminal Detection Pipeline

Terminal detection runs in two phases. Phase 1 is synchronous and runs at module import time; Phase 2 is async and must be awaited during startup.

### 3.1 Phase 1 — synchronous (`src/utils/env.ts`)

```typescript
// env.ts detectTerminal() — abbreviated JetBrains-relevant paths

// macOS: check __CFBundleIdentifier
const bundleId = process.env.__CFBundleIdentifier?.toLowerCase()
if (bundleId?.includes('com.google.android.studio')) return 'androidstudio'
if (bundleId) {
  for (const ide of JETBRAINS_IDES) {
    if (bundleId.includes(ide)) return ide  // e.g. 'pycharm', 'intellij'
  }
}

// Linux/Windows: JediTerm env var (single value for all JetBrains IDEs)
if (process.env.TERMINAL_EMULATOR === 'JetBrains-JediTerm') {
  return 'pycharm'  // placeholder; Phase 2 refines this
}
```

`JETBRAINS_IDES` (`src/utils/env.ts:115-132`) is the authoritative list used for bundle ID matching:

```typescript
export const JETBRAINS_IDES = [
  'pycharm', 'intellij', 'webstorm', 'phpstorm', 'rubymine', 'clion',
  'goland', 'rider', 'datagrip', 'appcode', 'dataspell', 'aqua',
  'gateway', 'fleet', 'jetbrains', 'androidstudio',
]
```

Phase 1 returns `'pycharm'` as a placeholder on Linux/Windows when `JetBrains-JediTerm` is detected. This allows synchronous code to proceed; Phase 2 overwrites the cached value with the specific IDE.

### 3.2 Phase 2 — async parent-process scan (`src/utils/envDynamic.ts`)

```typescript
// src/utils/envDynamic.ts:64-96
async function detectJetBrainsIDEFromParentProcessAsync(): Promise<string | null> {
  if (process.platform === 'darwin') {
    return null  // macOS: Phase 1 (bundle ID) already gave the right answer
  }

  const commands = await getAncestorCommandsAsync(process.pid, 10)  // up to 10 ancestors

  for (const command of commands) {
    const lowerCommand = command.toLowerCase()
    for (const ide of JETBRAINS_IDES) {
      if (lowerCommand.includes(ide)) {
        jetBrainsIDECache = ide
        return ide
      }
    }
  }

  jetBrainsIDECache = null
  return null  // fallback: getTerminalWithJetBrainsDetection() returns 'pycharm'
}
```

**Detection trigger:** Only runs when `TERMINAL_EMULATOR === 'JetBrains-JediTerm'`. On macOS, bundle IDs from Phase 1 are already definitive, so the parent-process scan is skipped.

**Fallback chain:**
1. Phase 2 found a specific IDE → use it
2. Phase 2 returned null (scan failed or macOS) → `'pycharm'` (generic JetBrains placeholder)

### 3.3 Initialization call

```typescript
// src/utils/envDynamic.ts:136-140
export async function initJetBrainsDetection(): Promise<void> {
  if (process.env.TERMINAL_EMULATOR === 'JetBrains-JediTerm') {
    await detectJetBrainsIDEFromParentProcessAsync()
  }
}
```

Call `initJetBrainsDetection()` **once, early in app startup** (before any UI renders). After it resolves, the synchronous `getTerminalWithJetBrainsDetection()` returns the fine-grained IDE type from cache.

---

## §4 JetBrains IDE Reference Table

All 15 JetBrains `IdeType` values with their process detection keywords and directory names.

| `IdeType` | `displayName` | macOS process | Windows exe | Linux keyword | Dir names |
|-----------|--------------|---------------|-------------|---------------|-----------|
| `pycharm` | PyCharm | `PyCharm` | `pycharm64.exe` | `pycharm` | `PyCharm` |
| `intellij` | IntelliJ IDEA | `IntelliJ IDEA` | `idea64.exe` | `idea`, `intellij` | `IntelliJIdea`, `IdeaIC` |
| `webstorm` | WebStorm | `WebStorm` | `webstorm64.exe` | `webstorm` | `WebStorm` |
| `phpstorm` | PhpStorm | `PhpStorm` | `phpstorm64.exe` | `phpstorm` | `PhpStorm` |
| `rubymine` | RubyMine | `RubyMine` | `rubymine64.exe` | `rubymine` | `RubyMine` |
| `clion` | CLion | `CLion` | `clion64.exe` | `clion` | `CLion` |
| `goland` | GoLand | `GoLand` | `goland64.exe` | `goland` | `GoLand` |
| `rider` | Rider | `Rider` | `rider64.exe` | `rider` | `Rider` |
| `datagrip` | DataGrip | `DataGrip` | `datagrip64.exe` | `datagrip` | `DataGrip` |
| `appcode` | AppCode | `AppCode` | `appcode.exe` | `appcode` | `AppCode` |
| `dataspell` | DataSpell | `DataSpell` | `dataspell64.exe` | `dataspell` | `DataSpell` |
| `aqua` | Aqua | _(disabled)_ | `aqua64.exe` | _(disabled)_ | `Aqua` |
| `gateway` | Gateway | _(disabled)_ | `gateway64.exe` | _(disabled)_ | `Gateway` |
| `fleet` | Fleet | _(disabled)_ | `fleet.exe` | _(disabled)_ | `Fleet` |
| `androidstudio` | Android Studio | `Android Studio` | `studio64.exe` | `android-studio` | `AndroidStudio` |

**"disabled" process keywords** (Aqua, Gateway, Fleet on Mac/Linux): process name matching is intentionally disabled because the process names are too generic. These IDEs are detected only via bundle ID (macOS) or Windows exe. On Linux, only the `TERMINAL_EMULATOR` env var + parent-process fallback applies.

---

## §5 Notification Protocol

Three notifications flow between the plugin and the CLI. All follow standard MCP JSON-RPC notification format (`method` + `params`, no `id`).

### 5.1 CLI → Plugin: `ide_connected`

Sent immediately after the MCP connection is established. The plugin uses this to confirm the CLI is running and capture its PID.

```typescript
// src/utils/ide.ts:829-836
export async function maybeNotifyIDEConnected(client: Client) {
  await client.notification({
    method: 'ide_connected',
    params: { pid: process.pid },
  })
}
```

**Schema:**
```typescript
{
  method: 'ide_connected',
  params: { pid: number }
}
```

**Plugin-side handling:** Record the PID. If the PID disappears from the process table, the CLI session has ended — clean up the lockfile.

### 5.2 Plugin → CLI: `selection_changed`

Sent whenever the user changes text selection in the IDE editor. The CLI uses this to populate context for the next prompt.

```typescript
// src/hooks/useIdeSelection.ts:32-53
{
  method: 'selection_changed',
  params: {
    selection?: {               // null or absent = cursor position only
      start: { line: number, character: number },  // 0-indexed
      end:   { line: number, character: number },  // 0-indexed
    } | null,
    text?: string,             // selected text (may be empty string)
    filePath?: string,         // absolute path on the IDE host
  }
}
```

**Line-count edge case** (`useIdeSelection.ts:94-99`): When `end.character === 0`, the end line is not counted as selected (the cursor is at the start of a line, not past any characters on it). The hook decrements `lineCount` by 1 in this case.

**Two notification shapes the handler accepts:**
1. `selection` object present + non-null → user has selected a range → compute `lineCount`
2. `selection` absent/null, `text` defined → cursor moved but no range selected → forward `text` + `filePath` with `lineCount: 0`

### 5.3 Plugin → CLI: `log_event`

A generic telemetry bridge. The plugin sends named events; the CLI forwards them to its analytics pipeline prefixed with `tengu_ide_`.

```typescript
// src/hooks/useIdeLogging.ts:8-16
{
  method: 'log_event',
  params: {
    eventName: string,               // becomes tengu_ide_{eventName}
    eventData: Record<string, unknown>,  // arbitrary key/value payload
  }
}
```

```typescript
// CLI side — useIdeLogging.ts:30-36
logEvent(`tengu_ide_${eventName}`, eventData)
```

Use this for plugin-side events that should appear in Claude Code telemetry (e.g., `plugin_loaded`, `diff_accepted`, `diff_rejected`).

---

## §6 WebSocket Auth Token

When the lockfile contains an `authToken`, the CLI includes it in every WebSocket upgrade request via a custom header:

```
X-Claude-Code-Ide-Authorization: {authToken}
```

```typescript
// src/services/mcp/client.ts (ws-ide transport setup)
if (serverRef.type === 'ws-ide') {
  const wsHeaders = {
    'User-Agent': getMCPUserAgent(),
    ...(serverRef.authToken && {
      'X-Claude-Code-Ide-Authorization': serverRef.authToken,
    }),
  }
  // passed to Bun or Node.js WebSocket constructor as headers
}
```

The plugin generates the token at startup and writes it to the lockfile. On upgrade, it validates the `X-Claude-Code-Ide-Authorization` header and rejects connections with an invalid or missing token. SSE transport has no auth mechanism.

---

## §7 Status Notice — Plugin Install Prompt

When Claude Code detects it is running inside a JetBrains terminal but the plugin is not installed, it displays an install prompt in the status bar.

```typescript
// src/utils/statusNoticeDefinitions.tsx:160-189
const jetbrainsPluginNotice: StatusNoticeDefinition = {
  id: 'jetbrains-plugin-install',
  type: 'info',
  isActive: context => {
    if (!isSupportedJetBrainsTerminal()) return false
    if (!(context.config.autoInstallIdeExtension ?? true)) return false
    const ideType = getTerminalIdeType()
    return ideType !== null && !isJetBrainsPluginInstalledCachedSync(ideType)
  },
  render: () => (
    <Box>
      Install the {ideName} plugin from the JetBrains Marketplace:{' '}
      https://docs.claude.com/s/claude-code-jetbrains
    </Box>
  ),
}
```

**Activation conditions (all must be true):**
1. `isSupportedJetBrainsTerminal()` — the current terminal is a JetBrains IDE
2. `config.autoInstallIdeExtension` is `true` (default) — not explicitly disabled
3. `isJetBrainsPluginInstalledCachedSync(ideType)` returns `false` — plugin not found

**Suppression:** Set `autoInstallIdeExtension: false` in `~/.claude/settings.json` or `CLAUDE_CODE_IDE_SKIP_AUTO_INSTALL=1` env var.

Unlike VS Code, the CLI cannot install JetBrains plugins automatically — this notice is informational only, pointing to the JetBrains Marketplace.

---

## §8 Implementing the Plugin Side

This section shows how to implement a minimal JetBrains plugin MCP server (the IDE side). The base lockfile + MCP server setup is in [`01-ide-integration.md §11`](01-ide-integration.md). The examples here focus on the JetBrains-specific notifications.

### 8.1 Plugin registration sequence

```
Plugin starts
    │
    ├─ Start MCP server (SSE on :PORT or WebSocket on :PORT)
    ├─ Generate authToken (UUID, WebSocket only)
    ├─ Write ~/.claude/ide/{PORT}.lock
    │     { workspaceFolders, pid, ideName, transport, authToken? }
    │
    └─ Wait for ide_connected notification from CLI
           │
           └─ Begin sending selection_changed notifications
                 (on every editor selection change event)
```

### 8.2 Python — implementing notifications

```python
import asyncio, json, os, signal
from pathlib import Path
from mcp.server import Server
from mcp.server.sse import SseServerTransport
from starlette.applications import Starlette
from starlette.routing import Route

PORT = 54321
IDE_NAME = "PyCharm"
WORKSPACE = [str(Path.cwd())]
LOCKFILE = Path.home() / ".claude" / "ide" / f"{PORT}.lock"

app_server = Server("ide")
connected_clients: list = []  # track active SSE sessions

# ── Startup: write lockfile ──────────────────────────────────────────────────

def write_lockfile():
    LOCKFILE.parent.mkdir(parents=True, exist_ok=True)
    LOCKFILE.write_text(json.dumps({
        "workspaceFolders": WORKSPACE,
        "pid": os.getpid(),
        "ideName": IDE_NAME,
        "transport": "sse",
    }))

def remove_lockfile():
    LOCKFILE.unlink(missing_ok=True)

# ── Handle ide_connected notification from CLI ───────────────────────────────

@app_server.notification_handler("ide_connected")
async def on_ide_connected(params: dict):
    cli_pid = params.get("pid")
    print(f"[plugin] CLI connected, pid={cli_pid}")
    # Optionally: monitor cli_pid with os.kill(cli_pid, 0) to detect disconnection

# ── Expose RPC tools the CLI can call ───────────────────────────────────────

@app_server.call_tool()
async def handle_call_tool(name: str, arguments: dict):
    if name == "openDiff":
        old_path = arguments["old_file_path"]
        new_path = arguments["new_file_path"]
        contents = arguments["new_file_contents"]
        tab_name = arguments["tab_name"]
        print(f"[plugin] openDiff: {tab_name}")
        # Open diff viewer in IDE here
        return [{"type": "text", "text": "ok"}]

    if name == "closeAllDiffTabs":
        print("[plugin] closeAllDiffTabs")
        return [{"type": "text", "text": "ok"}]

    if name == "close_tab":
        print(f"[plugin] close_tab: {arguments.get('tab_name')}")
        return [{"type": "text", "text": "ok"}]

    return [{"type": "text", "text": f"unknown tool: {name}"}]

@app_server.list_tools()
async def list_tools():
    from mcp.types import Tool
    return [
        Tool(name="openDiff", description="Open diff in IDE",
             inputSchema={"type": "object", "properties": {
                 "old_file_path": {"type": "string"},
                 "new_file_path": {"type": "string"},
                 "new_file_contents": {"type": "string"},
                 "tab_name": {"type": "string"},
             }, "required": ["old_file_path", "new_file_path", "new_file_contents", "tab_name"]}),
        Tool(name="closeAllDiffTabs", description="Close all diff tabs",
             inputSchema={"type": "object", "properties": {}}),
        Tool(name="close_tab", description="Close a specific tab",
             inputSchema={"type": "object", "properties": {
                 "tab_name": {"type": "string"},
             }, "required": ["tab_name"]}),
    ]

# ── Send selection_changed to CLI ────────────────────────────────────────────

async def send_selection_changed(
    session,            # active MCP session
    file_path: str,
    start_line: int, start_char: int,
    end_line: int, end_char: int,
    selected_text: str,
):
    await session.send_notification(
        method="selection_changed",
        params={
            "selection": {
                "start": {"line": start_line, "character": start_char},
                "end":   {"line": end_line,   "character": end_char},
            },
            "text": selected_text,
            "filePath": file_path,
        },
    )

# ── Send log_event to CLI ────────────────────────────────────────────────────

async def send_log_event(session, event_name: str, event_data: dict):
    # Results in tengu_ide_{event_name} in CLI analytics
    await session.send_notification(
        method="log_event",
        params={"eventName": event_name, "eventData": event_data},
    )

# ── Starlette SSE app ────────────────────────────────────────────────────────

transport = SseServerTransport("/messages/")

async def handle_sse(request):
    async with transport.connect_sse(request.scope, request.receive, request._send) as streams:
        await app_server.run(streams[0], streams[1], app_server.create_initialization_options())

starlette_app = Starlette(routes=[Route("/sse", endpoint=handle_sse)])

if __name__ == "__main__":
    import uvicorn
    write_lockfile()
    try:
        uvicorn.run(starlette_app, host="127.0.0.1", port=PORT)
    finally:
        remove_lockfile()
```

### 8.3 Kotlin — implementing notifications

```kotlin
import io.modelcontextprotocol.kotlin.sdk.*
import io.modelcontextprotocol.kotlin.sdk.server.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import kotlinx.serialization.json.*
import java.io.File
import java.nio.file.Files

const val PORT = 54321
const val IDE_NAME = "IntelliJ IDEA"

fun writeLockfile() {
    val lockDir = File(System.getProperty("user.home"), ".claude/ide")
    lockDir.mkdirs()
    File(lockDir, "$PORT.lock").writeText(
        """{"workspaceFolders":["${System.getProperty("user.dir")}"],"pid":${ProcessHandle.current().pid()},"ideName":"$IDE_NAME","transport":"ws"}"""
    )
}

fun removeLockfile() {
    File(System.getProperty("user.home"), ".claude/ide/$PORT.lock").delete()
}

fun buildMcpServer(): Server {
    val server = Server(
        Implementation(name = "ide", version = "1.0.0"),
        ServerOptions(capabilities = ServerCapabilities(tools = ServerCapabilities.Tools(listChanged = false)))
    )

    // ── Handle ide_connected notification ────────────────────────────────────
    server.setNotificationHandler("ide_connected") { notification ->
        val pid = notification.params?.get("pid")?.jsonPrimitive?.intOrNull
        println("[plugin] CLI connected, pid=$pid")
    }

    // ── Expose RPC tools ──────────────────────────────────────────────────────
    server.addTool(
        name = "openDiff",
        description = "Open diff in IDE",
        inputSchema = Tool.Input(
            properties = mapOf(
                "old_file_path"      to JsonObject(mapOf("type" to JsonPrimitive("string"))),
                "new_file_path"      to JsonObject(mapOf("type" to JsonPrimitive("string"))),
                "new_file_contents"  to JsonObject(mapOf("type" to JsonPrimitive("string"))),
                "tab_name"           to JsonObject(mapOf("type" to JsonPrimitive("string"))),
            ),
            required = listOf("old_file_path", "new_file_path", "new_file_contents", "tab_name")
        )
    ) { request ->
        val args = request.arguments
        println("[plugin] openDiff: ${args["tab_name"]?.jsonPrimitive?.content}")
        // Invoke IDE diff viewer here via IntelliJ Platform API
        CallToolResult(content = listOf(TextContent("ok")))
    }

    server.addTool(
        name = "closeAllDiffTabs",
        description = "Close all diff tabs",
        inputSchema = Tool.Input()
    ) { _ ->
        println("[plugin] closeAllDiffTabs")
        CallToolResult(content = listOf(TextContent("ok")))
    }

    server.addTool(
        name = "close_tab",
        description = "Close a specific tab",
        inputSchema = Tool.Input(
            properties = mapOf("tab_name" to JsonObject(mapOf("type" to JsonPrimitive("string")))),
            required = listOf("tab_name")
        )
    ) { request ->
        println("[plugin] close_tab: ${request.arguments["tab_name"]?.jsonPrimitive?.content}")
        CallToolResult(content = listOf(TextContent("ok")))
    }

    return server
}

// ── Send selection_changed ────────────────────────────────────────────────────

suspend fun sendSelectionChanged(
    server: Server,
    filePath: String,
    startLine: Int, startChar: Int,
    endLine: Int, endChar: Int,
    text: String,
) {
    server.sendNotification(
        method = "selection_changed",
        params = buildJsonObject {
            put("selection", buildJsonObject {
                put("start", buildJsonObject {
                    put("line", startLine); put("character", startChar)
                })
                put("end", buildJsonObject {
                    put("line", endLine); put("character", endChar)
                })
            })
            put("text", text)
            put("filePath", filePath)
        }
    )
}

// ── Send log_event ────────────────────────────────────────────────────────────

suspend fun sendLogEvent(server: Server, eventName: String, eventData: JsonObject) {
    // CLI forwards this as tengu_ide_{eventName}
    server.sendNotification(
        method = "log_event",
        params = buildJsonObject {
            put("eventName", eventName)
            put("eventData", eventData)
        }
    )
}

fun main() {
    writeLockfile()
    Runtime.getRuntime().addShutdownHook(Thread { removeLockfile() })

    // WebSocket transport (ws-ide) — use ktor-websockets or similar
    // See 01-ide-integration.md §11 for the WebSocket server setup
    println("[plugin] MCP server listening on ws://127.0.0.1:$PORT")
}
```

### 8.4 C++23 — notification payloads

The C++23 base server setup is in [`01-ide-integration.md §11`](01-ide-integration.md). Add these notification dispatchers on top:

```cpp
// selection_changed payload builder
nlohmann::json make_selection_changed(
    std::string_view file_path,
    int start_line, int start_char,
    int end_line,   int end_char,
    std::string_view text)
{
    return {
        {"method", "selection_changed"},
        {"params", {
            {"selection", {
                {"start", {{"line", start_line}, {"character", start_char}}},
                {"end",   {{"line", end_line},   {"character", end_char}}}
            }},
            {"text",     text},
            {"filePath", file_path}
        }}
    };
}

// log_event payload builder — CLI forwards as tengu_ide_{event_name}
nlohmann::json make_log_event(
    std::string_view event_name,
    nlohmann::json   event_data)
{
    return {
        {"method", "log_event"},
        {"params", {
            {"eventName", event_name},
            {"eventData", std::move(event_data)}
        }}
    };
}

// ide_connected handler (incoming from CLI)
void on_notification(const nlohmann::json& notif) {
    if (notif["method"] == "ide_connected") {
        int cli_pid = notif["params"]["pid"];
        std::println("[plugin] CLI connected, pid={}", cli_pid);
    }
}
```

---

## §9 Key Behavioral Differences vs VS Code Integration

| Behaviour | VS Code | JetBrains |
|-----------|---------|-----------|
| Plugin install | Auto-installed by CLI | Manual via Marketplace |
| Terminal detection | Sync env var (unique per IDE) | Two-phase: bundle ID + async parent scan |
| Auth token | Not used | Optional `authToken` in lockfile → `X-Claude-Code-Ide-Authorization` header |
| Process keywords | One entry per platform | `{IDE}64.exe` on Windows; Aqua/Gateway/Fleet disabled on Mac/Linux |
| `selection_changed` | Implemented same way | Implemented same way |
| `log_event` | Implemented same way | Implemented same way |
| Plugin dir (Linux) | N/A | No `plugins/` subdir — config root directly |

---

*[← IDE Integration](01-ide-integration.md) | [Distributed Agents →](01-distributed-agents.md)*
