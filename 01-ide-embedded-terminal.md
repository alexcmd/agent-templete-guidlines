# IDE Embedded Terminal — Behaviour Changes

> This document covers what changes when Claude Code detects it is running **inside** an IDE's built-in terminal, as opposed to a standalone terminal. The transport/lockfile protocol is in [`01-ide-integration.md`](01-ide-integration.md); JetBrains-specific plugin detection is in [`01-jetbrains-ide-plugin.md`](01-jetbrains-ide-plugin.md).

---

## §1 The Master Gate: `isSupportedTerminal()`

All IDE-terminal-specific behaviour is gated on a single memoized boolean:

```typescript
// src/utils/ide.ts:279-285
export const isSupportedTerminal = memoize(() => {
  return (
    isSupportedVSCodeTerminal() ||       // cursor | windsurf | vscode
    isSupportedJetBrainsTerminal() ||    // all 15 JetBrains IDEs
    Boolean(process.env.FORCE_CODE_TERMINAL)
  )
})
```

```typescript
// sub-gates
export const isSupportedVSCodeTerminal = memoize(() =>
  isVSCodeIde(env.terminal as IdeType),        // reads static env at import time
)
export const isSupportedJetBrainsTerminal = memoize(() =>
  isJetBrainsIde(envDynamic.terminal as IdeType), // reads async-populated cache
)
```

**`FORCE_CODE_TERMINAL`** — environment variable override that forces `isSupportedTerminal()` to `true` regardless of actual terminal. Useful for testing IDE-terminal code paths in CI or non-IDE shells.

The gate is memoized — it resolves once and never re-evaluates. `initJetBrainsDetection()` must complete before the JetBrains branch produces an accurate answer (see [`01-jetbrains-ide-plugin.md §3`](01-jetbrains-ide-plugin.md)).

---

## §2 What Changes — Overview

```
                  isSupportedTerminal()
                  ┌─────────────────────────────────────────┐
           true   │                             false        │
   (IDE terminal) │                    (external terminal)   │
                  │                                          │
  Auto-connect    │ unconditional                explicit opt-in required
  /config option  │ autoInstallIdeExtension      autoConnectIde
  First-run dialog│ suppressed                   shown once
  Ancestry check  │ enabled                      skipped
  IDE name fallback│ terminal name               null
  Ext install     │ triggered                    triggered (same)
  JetBrains notice│ shown if plugin missing      never shown
  /ide errors     │ IDE-specific message         generic message
  /ide tip        │ suppressed                   shown
  Save file hint  │ VS Code only                 never
                  └─────────────────────────────────────────┘
```

---

## §3 Auto-Connect Behaviour

### External terminal (opt-in)

`autoConnectEnabled` requires at least one of these to be truthy:
- `globalConfig.autoConnectIde === true` (user toggled in `/config`)
- `--ide` CLI flag (`autoConnectIdeFlag`)
- `CLAUDE_CODE_SSE_PORT` env var (see §10)
- `CLAUDE_CODE_AUTO_CONNECT_IDE=1`
- `ideToInstallExtension` set (extension install flow)

### IDE terminal (unconditional)

`isSupportedTerminal()` is in the same OR chain:

```typescript
// src/hooks/useIDEIntegration.tsx:33
const autoConnectEnabled =
  (globalConfig.autoConnectIde ||
   autoConnectIdeFlag ||
   isSupportedTerminal() ||   // ← always true in IDE terminal
   process.env.CLAUDE_CODE_SSE_PORT ||
   ideToInstallExtension ||
   isEnvTruthy(process.env.CLAUDE_CODE_AUTO_CONNECT_IDE)) &&
  !isEnvDefinedFalsy(process.env.CLAUDE_CODE_AUTO_CONNECT_IDE)
```

When a lockfile is found and validated, `setDynamicMcpConfig` is called immediately with a `sse-ide` or `ws-ide` entry — no user confirmation needed.

**Opt-out:** `CLAUDE_CODE_AUTO_CONNECT_IDE=false` — `isEnvDefinedFalsy()` converts this to the hard override that defeats all OR conditions.

---

## §4 `/config` — Mutually Exclusive Options

The two IDE config toggles are rendered conditionally and never appear together:

```typescript
// src/components/Settings/Config.tsx:836-870

// External terminal only
...(!isSupportedTerminal() ? [{
  id: 'autoConnectIde',
  label: 'Auto-connect to IDE (external terminal)',
  value: globalConfig.autoConnectIde ?? false,
  // default: false
}] : []),

// IDE terminal only
...(isSupportedTerminal() ? [{
  id: 'autoInstallIdeExtension',
  label: 'Auto-install IDE extension',
  value: globalConfig.autoInstallIdeExtension ?? true,
  // default: true
}] : []),
```

| Setting | Default | Context |
|---------|---------|---------|
| `autoConnectIde` | `false` | external terminal |
| `autoInstallIdeExtension` | `true` | IDE terminal |

Both persist to `~/.claude/settings.json` and emit analytics events on change.

---

## §5 First-Run Dialogs

### Auto-connect dialog (external terminal only)

```typescript
// src/components/IdeAutoConnectDialog.tsx:73-76
export function shouldShowAutoConnectDialog(): boolean {
  const config = getGlobalConfig()
  return (
    !isSupportedTerminal() &&                        // not in IDE terminal
    config.autoConnectIde !== true &&                // not already enabled
    config.hasIdeAutoConnectDialogBeenShown !== true // not seen before
  )
}
```

Rendered as: *"Do you wish to enable auto-connect to IDE?"* with Yes/No. Saves `autoConnectIde` and sets `hasIdeAutoConnectDialogBeenShown: true` to prevent repeating.

In an IDE terminal this dialog never appears because auto-connect is already implicit.

### Disable auto-connect dialog (external terminal only)

`shouldShowDisableAutoConnectDialog()`: `!isSupportedTerminal() && config.autoConnectIde === true` — prompts the user to disable auto-connect when the lockfile-connected IDE disappears. Also suppressed in IDE terminals.

### Onboarding dialog (both contexts, after first install)

`setShowIdeOnboarding(true)` is triggered by `initializeIdeIntegration()` after the extension/plugin is successfully installed for the first time. Appears in both terminal contexts.

---

## §6 Process Ancestry Check for Multi-IDE Disambiguation

When multiple IDE windows share overlapping workspace folders, lockfile workspace matching alone cannot identify the right IDE. Inside an IDE terminal, Claude Code walks the ancestor PID chain to find the specific IDE process:

```typescript
// src/utils/ide.ts:690
const needsAncestryCheck = getPlatform() !== 'wsl' && isSupportedTerminal()
```

When `needsAncestryCheck` is true, `getAncestors()` (lazy, single-shot per `detectIDEs()` call) is invoked to compare lockfile PIDs against the process tree. The match selects the correct IDE window. On WSL the shell-out is unreliable; ancestry checking is skipped there regardless.

In an external terminal there is no ancestor relationship to a running IDE, so the check never runs and disambiguation falls back to workspace path matching only.

---

## §7 IDE Name Fallback in UI

```typescript
// src/utils/ide.ts:1188-1197
export function getIdeClientName(ideClient?: MCPServerConnection): string | null {
  const config = ideClient?.config
  return config?.type === 'sse-ide' || config?.type === 'ws-ide'
    ? config.ideName                              // from connected lockfile
    : isSupportedTerminal()
      ? toIDEDisplayName(envDynamic.terminal)     // from env detection
      : null                                      // external terminal: no name
}
```

Before the MCP connection is established (or if it fails), any UI that calls `getIdeClientName()` gets:
- **IDE terminal**: the display name of the detected IDE (e.g. `"PyCharm"`, `"IntelliJ IDEA"`)
- **External terminal**: `null` — the UI must handle the null case

---

## §8 Extension and Plugin Installation

`initializeIdeIntegration()` runs in both terminal contexts. The branching is on `getTerminalIdeType()`, not on `isSupportedTerminal()` directly:

```
initializeIdeIntegration()
    │
    ├─ getTerminalIdeType() → null         → no installation attempt
    │
    ├─ isVSCodeIde(ideType)                → auto-install via `code --install-extension`
    │     └─ success → setShowIdeOnboarding(true)
    │
    └─ isJetBrainsIde(ideType)             → isJetBrainsPluginInstalledCached()
          ├─ installed → no-op
          └─ not installed → status notice only (no auto-install)
```

`autoInstallIdeExtension: false` in config (or `CLAUDE_CODE_IDE_SKIP_AUTO_INSTALL=1`) skips the entire installation check.

---

## §9 JetBrains Marketplace Status Notice

Only fires when inside a JetBrains terminal:

```typescript
// src/utils/statusNoticeDefinitions.tsx:163-175
isActive: context => {
  if (!isSupportedJetBrainsTerminal()) return false              // must be JetBrains terminal
  if (!(context.config.autoInstallIdeExtension ?? true)) return false  // must not be suppressed
  const ideType = getTerminalIdeType()
  return ideType !== null && !isJetBrainsPluginInstalledCachedSync(ideType)  // plugin absent
}
```

Displays: `↑ Install the {IDE} plugin from the JetBrains Marketplace: https://docs.claude.com/s/claude-code-jetbrains`

Never shown in VS Code terminals or external terminals.

---

## §10 `/ide` Command — Message Differences

**Error when no IDE found** (`src/commands/ide/ide.tsx:131`):

```typescript
if (isSupportedJetBrainsTerminal()) {
  // JetBrains terminal
  "Please install the plugin and restart your IDE"
  + "https://docs.claude.com/s/claude-code-jetbrains"
} else {
  // VS Code terminal or external terminal
  "Make sure your IDE has the Claude Code extension or plugin installed"
}
```

**Tip message** (`ide.tsx:161`) — shown only in external terminals when IDEs are available:

```typescript
if (!isSupportedTerminal() && availableIDEs.length !== 0) {
  "Tip: You can enable auto-connect to IDE in /config or with the --ide flag"
}
```

This tip is suppressed in IDE terminals because auto-connect is already on.

---

## §11 VS Code Terminal Only: "Save File to Continue" Hint

```typescript
// src/components/ShowInIDEPrompt.tsx:60
if (isSupportedVSCodeTerminal()) {
  render("Save file to continue...")
}
```

This hint appears when Claude Code is waiting for the user to save an edited file in the VS Code editor before the tool call can proceed. Not shown in JetBrains terminals (different file sync mechanism) or external terminals.

---

## §12 Edge Cases

### tmux / screen inside an IDE terminal

tmux and screen overwrite `TERM_PROGRAM` (VS Code) or mask `TERMINAL_EMULATOR` (JetBrains), causing `isSupportedTerminal()` to return `false`. However, `CLAUDE_CODE_SSE_PORT` (set by the VS Code extension when it spawns the shell) is inherited through tmux panes:

```typescript
// useIDEIntegration.tsx — comment in source
// tmux/screen overwrite TERM_PROGRAM, breaking terminal detection, but the
// IDE extension's port env var is inherited. If set, auto-connect anyway.
process.env.CLAUDE_CODE_SSE_PORT  // still in the OR chain
```

Auto-connect fires via `CLAUDE_CODE_SSE_PORT` even when `isSupportedTerminal()` is `false`. The result: transport connects correctly but the config toggle shown in `/config` will be `autoConnectIde` (external terminal path) not `autoInstallIdeExtension`.

**JetBrains + tmux**: `TERMINAL_EMULATOR` is also masked. Neither `isSupportedTerminal()` nor `CLAUDE_CODE_SSE_PORT` applies. Use `CLAUDE_CODE_AUTO_CONNECT_IDE=1` as a workaround.

### `FORCE_CODE_TERMINAL`

Forces `isSupportedTerminal()` to `true` in any shell. Use in CI or test environments to exercise IDE-terminal code paths without a real IDE. Does not affect lockfile discovery or transport — you still need a lockfile for the connection.

### `CLAUDE_CODE_AUTO_CONNECT_IDE`

- `=1` or `=true`: forces auto-connect on even in external terminals without `autoConnectIde` config
- `=false` or `=0`: hard-disables auto-connect even in IDE terminals — overrides `isSupportedTerminal()`

---

## §13 Decision Matrix — Which Behaviour Applies?

| Scenario | `isSupportedTerminal()` | Auto-connect | Config option | Dialog |
|----------|------------------------|--------------|---------------|--------|
| VS Code built-in terminal | `true` | unconditional | `autoInstallIdeExtension` | suppressed |
| JetBrains built-in terminal | `true` | unconditional | `autoInstallIdeExtension` | suppressed |
| External terminal, `autoConnectIde: true` | `false` | on | `autoConnectIde` | disable dialog shown |
| External terminal, `autoConnectIde: false` | `false` | off | `autoConnectIde` | enable dialog shown (once) |
| tmux inside VS Code | `false` | via `CLAUDE_CODE_SSE_PORT` | `autoConnectIde` | suppressed if already connected |
| `FORCE_CODE_TERMINAL=1` | `true` | unconditional | `autoInstallIdeExtension` | suppressed |
| `CLAUDE_CODE_AUTO_CONNECT_IDE=false` | any | **disabled** | unchanged | suppressed |

---

## §14 Implementing Terminal-Aware Behaviour in Your Agent

If you are building an agent that embeds a terminal or wraps Claude Code, replicate the env vars Claude Code reads:

```python
# Python — simulate IDE terminal environment

import subprocess, os

env = os.environ.copy()

# Option A: VS Code-style (synchronous, instant detection)
env["TERM_PROGRAM"] = "vscode"           # or set VSCODE_GIT_ASKPASS_MAIN

# Option B: Cursor
env["CURSOR_TRACE_ID"] = "any-value"

# Option C: JetBrains-style (requires lockfile + parent process match)
env["TERMINAL_EMULATOR"] = "JetBrains-JediTerm"
# The CLI will then walk ancestor PIDs looking for a JetBrains process name.
# Since it won't find one, it falls back to 'pycharm' as the generic type.

# Option D: Force all IDE-terminal behaviour without terminal detection
env["FORCE_CODE_TERMINAL"] = "1"

# Option E: Force auto-connect without IDE terminal detection
env["CLAUDE_CODE_AUTO_CONNECT_IDE"] = "1"
# Write a lockfile first (see 01-ide-integration.md §11)

result = subprocess.run(["claude", "--print", "hello"], env=env)
```

```kotlin
// Kotlin — simulate IDE terminal

val pb = ProcessBuilder("claude", "--print", "hello")
pb.environment().apply {
    // VS Code terminal path (instant detection)
    put("TERM_PROGRAM", "vscode")

    // Or: force all IDE-terminal behaviour
    put("FORCE_CODE_TERMINAL", "1")

    // Or: force auto-connect only
    put("CLAUDE_CODE_AUTO_CONNECT_IDE", "1")
    put("CLAUDE_CODE_SSE_PORT", "54321")  // must match your lockfile
}
val proc = pb.start()
```

```cpp
// C++23 — simulate IDE terminal

#include <cstdlib>
#include <array>
#include <string>

int simulate_ide_terminal() {
    // VS Code terminal (sync detection)
    setenv("TERM_PROGRAM", "vscode", 1);

    // Or force-enable without terminal detection
    setenv("FORCE_CODE_TERMINAL", "1", 1);

    // Or force auto-connect (pair with a lockfile)
    setenv("CLAUDE_CODE_AUTO_CONNECT_IDE", "1", 1);
    setenv("CLAUDE_CODE_SSE_PORT", "54321", 1);

    return system("claude --print hello");
}
```

**Key env vars controlling terminal-aware behaviour:**

| Variable | Effect |
|----------|--------|
| `TERM_PROGRAM=vscode` | `isSupportedVSCodeTerminal()` → true |
| `TERMINAL_EMULATOR=JetBrains-JediTerm` | `isSupportedJetBrainsTerminal()` → true (after async detection) |
| `FORCE_CODE_TERMINAL=1` | `isSupportedTerminal()` → true unconditionally |
| `CLAUDE_CODE_AUTO_CONNECT_IDE=1` | forces auto-connect regardless of terminal type |
| `CLAUDE_CODE_AUTO_CONNECT_IDE=false` | disables auto-connect unconditionally |
| `CLAUDE_CODE_SSE_PORT=PORT` | auto-connect even when `isSupportedTerminal()` is false (tmux case) |

---

*[← JetBrains IDE Plugin](01-jetbrains-ide-plugin.md) | [Distributed Agents →](01-distributed-agents.md)*
