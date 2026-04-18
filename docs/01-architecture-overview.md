# Architecture Overview

## System Purpose

Claude Code is a CLI-based AI coding agent. It wraps the Anthropic Claude API in a persistent conversation loop that can read/write files, run shell commands, search codebases, spawn sub-agents, and interact with external services — all from the terminal.

**Runtime:** Bun (TypeScript, JSX via React/Ink for terminal UI)
**Frontend:** Ink.js (React renderer for terminal)
**API:** Anthropic Claude API (streaming, beta features)
**Protocol:** Model Context Protocol (MCP) for external tools

---

## Architectural Layers

```
┌─────────────────────────────────────────────────────────────┐
│                      USER INTERFACE                          │
│   Terminal REPL (React/Ink)   │  stdin/stdout (headless)    │
├─────────────────────────────────────────────────────────────┤
│                    COMMAND LAYER                              │
│  CLI Commands  │  Slash Commands  │  Skills  │  MCP Cmds    │
├─────────────────────────────────────────────────────────────┤
│                   SESSION LAYER                               │
│         QueryEngine (conversation state + lifecycle)         │
├─────────────────────────────────────────────────────────────┤
│                    QUERY LAYER                                │
│   query.ts (streaming loop, tool execution, compaction)      │
├─────────────────────────────────────────────────────────────┤
│                    TOOL LAYER                                 │
│  BashTool │ FileTools │ AgentTool │ MCPTool │ WebTools │ ... │
├─────────────────────────────────────────────────────────────┤
│                  PERMISSION LAYER                             │
│   ToolPermissionContext  │  Allow/Deny/Ask Rules  │  Hooks   │
├─────────────────────────────────────────────────────────────┤
│                   SERVICE LAYER                               │
│   API Client  │  MCP  │  Plugins  │  Auth  │  Analytics     │
├─────────────────────────────────────────────────────────────┤
│                  INFRASTRUCTURE                               │
│  Config System │ Memory System │ Task System │ State Store   │
└─────────────────────────────────────────────────────────────┘
```

---

## Subsystem Map

### Core Engine (src/)
| File/Directory | Role |
|---|---|
| `main.tsx` | Process entry point — fast-path routing, daemon workers |
| `entrypoints/cli.tsx` | CLI argument parsing, mode selection |
| `entrypoints/init.ts` | Bootstrap: config, auth, telemetry, proxy, MDM |
| `QueryEngine.ts` | Conversation session object — manages message history, abort, usage |
| `query.ts` | The inner streaming loop — calls Claude API, executes tools, handles compaction |
| `Tool.ts` | Tool interface definition, ToolUseContext, ToolPermissionContext |
| `tools.ts` | Tool registry — assembles the pool of available tools |
| `commands.ts` | Slash command registry — all `/commands` |
| `tasks.ts` | Task registry helpers |
| `Task.ts` | Task type definitions |

### Subsystems
| Directory | Role |
|---|---|
| `tools/` | 40+ tool implementations (each in its own subdirectory) |
| `services/` | API client, MCP, plugins, compaction, auth, analytics, LSP |
| `screens/` | React/Ink UI components (REPL.tsx is the main 5k-line screen) |
| `commands/` | Local command handlers for CLI operations |
| `skills/bundled/` | Built-in skills registered as slash commands |
| `state/` | AppState store (Redux-like immutable state with React Compiler) |
| `components/` | Shared React components (permission dialogs, spinners, etc.) |
| `utils/` | Utilities: auth, git, bash, shell, memory, hooks, permissions, etc. |
| `types/` | TypeScript type definitions |
| `constants/` | System prompts, product URLs, XML tags, betas, OAuth config |
| `hooks/` | Tool permission system hooks |
| `memdir/` | Memory directory (.claude/) — memory types, paths, scanning |
| `bootstrap/` | Global state (sessions, costs, telemetry, MDM) |
| `plugins/` | Plugin system infrastructure |
| `tasks/` | Task management (LocalAgentTask, RemoteAgentTask, LocalShellTask) |
| `coordinator/` | Coordinator mode (multi-agent orchestration) |
| `bridge/` | Remote control / bridge environment (CCR mode) |
| `context/` | Context builders (notifications, user/system context) |
| `query/` | Query sub-modules: config, hooks, token budget, transitions |
| `schemas/` | JSON Schema definitions |
| `outputStyles/` | Custom output style definitions |

---

## Execution Modes

Claude Code runs in several distinct modes:

### 1. Interactive REPL (default)
The primary mode. Launches a React/Ink terminal app with a text input field, message history, and status line. User types messages; agent responds and executes tools.

### 2. Non-interactive / Headless
`claude -p "prompt"` or piped input. Runs a single query without UI, outputs to stdout. Used in scripts and CI.

### 3. Init Mode
`claude init` — initializes CLAUDE.md in the current project.

### 4. MCP Server Mode
Serves Claude Code itself as an MCP server for IDE integrations.

### 5. Bridge / CCR Mode
`--remote-control` / `--bridge` — serves as a bridge for remote cloud execution environments.

### 6. Daemon Worker Mode
`--daemon-worker=<kind>` — spawns background workers for tasks like code indexing.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Runtime | Bun (TypeScript, native JSX) |
| UI framework | React + Ink (terminal renderer) |
| Schema validation | Zod v4 |
| State management | Custom Redux-like store with React Compiler |
| API client | `@anthropic-ai/sdk` |
| MCP integration | `@modelcontextprotocol/sdk` |
| Shell execution | Custom Shell utilities wrapping Node child_process |
| Config persistence | JSON files (~/.claude/, .claude/) |
| Telemetry | OpenTelemetry (OTEL) metrics + first-party event logger |
| Auth | OAuth 2.0 PKCE (claude.ai) or API key |
| Feature flags | GrowthBook |
| Build | Bun bundle with dead-code elimination via `feature()` gates |

---

## Key Design Principles

### 1. Streaming-first
Every API call uses the streaming SDK. Partial text appears in real-time. Tool calls execute as soon as they're fully received from the stream.

### 2. Tool-use loop
Claude doesn't just answer — it calls tools. The query loop continues until Claude stops calling tools (stop_reason = `end_turn`) or the user interrupts.

### 3. Permission before action
Every tool execution is guarded by the `CanUseToolFn`. Rules (allow/deny/ask) are evaluated against the current `ToolPermissionContext` before anything runs.

### 4. Layered configuration
Settings cascade: global → project → local → managed → MDM. Later layers override earlier ones. Each layer has a distinct file location.

### 5. Memory injection
`.claude/` directories (global and project-level) hold markdown memory files that are loaded and injected into the system prompt at query time.

### 6. Plugin extensibility
Skills, commands, MCP servers, hooks, and output styles can all be added via plugins without modifying the core.

### 7. Multi-agent by design
The `AgentTool` is a first-class built-in. Sub-agents run in isolated `QueryEngine` instances, communicate via the `TaskSystem`, and can be backgrounded.
