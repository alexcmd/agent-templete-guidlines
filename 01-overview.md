# Universal Agent Architecture — Overview

> Complete cross-language architectural patterns for production AI coding agents.  
> Grounded in 4 production codebases and 25 arXiv papers.

---

## Document Map

| File | Sections | Topics |
|------|----------|--------|
| [01-core-runtime.md](01-core-runtime.md) | §1–4 | Agent runtime model, component inventory, multi-provider registry, layered architecture |
| [01-prompts-permissions.md](01-prompts-permissions.md) | §5–6 | System prompt construction, permission architecture (6 modes, rule schema) |
| [01-memory-rag-knowledge.md](01-memory-rag-knowledge.md) | §7–9 | Memory tiers (CoALA), memdir format, RAG pipelines, knowledge graph integration |
| [01-event-driven-multiagent.md](01-event-driven-multiagent.md) | §10–11 | Event sourcing, CQRS, local multi-agent, orchestrator/worker patterns |
| [01-advanced-patterns.md](01-advanced-patterns.md) | §12–14 | Self-improvement, session persistence, prompt engineering |
| [01-config-infra-reference.md](01-config-infra-reference.md) | §15–18 | Config schema, Docker infra, research paper reference, cross-language decision guide |
| [01-planning-hooks.md](01-planning-hooks.md) | §19–20 | Planning mode (ScratchpadGate, plan files), hook system (27+ events, wire protocol) |
| [01-clear-context-plan-user-interaction.md](01-clear-context-plan-user-interaction.md) | §1–6 | Clear-context-and-execute pattern (source-verified), AskUserQuestion, all user-interaction tools, hooks full reference, skill implementation guide |
| [01-sessions-cost-ide.md](01-sessions-cost-ide.md) | §21–23 | Session JSONL, cost tracking, IDE integration (LSP, delta context) |
| [01-ide-integration.md](01-ide-integration.md) | §1–12 | IDE integration source-verified deep dive: lockfile protocol, MCP sse-ide/ws-ide transports, RPC (openDiff/closeTab), diff viewer hook, WSL path conversion, reproduction guide |
| [01-jetbrains-ide-plugin.md](01-jetbrains-ide-plugin.md) | §1–9 | JetBrains deep dive: plugin detection (jetbrains.ts), three-tier caching, two-phase terminal detection, all 15 IDE process keywords, notification schemas (ide_connected/selection_changed/log_event), auth token header, status notice logic, plugin-side reproduction guide (Python/Kotlin/C++23) |
| [01-ide-embedded-terminal.md](01-ide-embedded-terminal.md) | §1–14 | `isSupportedTerminal()` master gate, auto-connect unconditional vs opt-in, /config option switching, dialog suppression, ancestry check disambiguation, IDE name fallback, extension/plugin install flow, JetBrains status notice, /ide message differences, VS Code save-file hint, tmux/FORCE_CODE_TERMINAL/CLAUDE_CODE_AUTO_CONNECT_IDE edge cases, decision matrix, reproduction guide (Python/Kotlin/C++23) |
| [01-distributed-agents.md](01-distributed-agents.md) | §24 | A2A protocol, Agent Card, distributed task delegation, JWT auth |
| [01-extensions-sdk-mcp.md](01-extensions-sdk-mcp.md) | §25–28 | LSP client, plugin system, SDK API, MCP (6 transports, OAuth) |
| [05-dap-debug-integration.md](05-dap-debug-integration.md) | §1–7 | DAP client, adapter lifecycle, breakpoints, stepping, variable inspection, agent loop, Claude Code source architecture map |
| [05-dap-claude-code-plugin.md](05-dap-claude-code-plugin.md) | §1–10 | External debugger plugin (zero source changes): MCP server, hooks, skills, full worked example, limitations, decision matrix |
| [01-token-budget-management.md](01-token-budget-management.md) | §22 | Canonical token counter, threshold arithmetic (effective window, autocompact, blocking), output capping, budget across compaction, circuit breaker |
| [claude-code-extensions/10-bridge-protocol.md](claude-code-extensions/10-bridge-protocol.md) | — | REPL↔Web bidirectional bridge: ReplBridgeHandle, V1 polling / V2 WebSocket transports, FlushGate ordering, SDKControlRequest/Response, outbound-only mode |

---

## Universal Patterns

All production agents share these 8 patterns:

1. **Layered System Prompt** — static cacheable prefix + dynamic suffix (git status, memory files, tool list)
2. **Tool Use Loop** — stream → parse tool call → permission gate → pre-hook → execute → post-hook → inject result → repeat
3. **Hierarchical Config** — managed → user → project → local → CLI flags (later wins)
4. **Memory File Discovery** — walk upward from CWD collecting MEMORY.md/CLAUDE.md/GEMINI.md files (cap at 200)
5. **Session JSONL** — append-only log with parentUuid linked list, sha256(realpath(cwd))[:16] project dirs
6. **Hook Lifecycle** — subprocess stdin/stdout JSON protocol with if/once/asyncRewake flags
7. **MCP Tool Namespacing** — `mcp__{server}__{tool}` pattern with 6 transport options
8. **Context Compaction** — split at ToolUse/ToolResult boundaries, summarize old turns

---

## Core Agent Loop

```
User Input
  → Build System Prompt (static prefix + dynamic suffix)
  → LLM Stream
      ├─ Text delta → display live
      └─ Tool use block → validate → check permission
                          → pre-hook
                          → execute (parallel if multiple)
                          → post-hook
                          → inject ToolResult
                          → loop back to LLM Stream
  → Final Response
  → Session Save (append JSONL)
  → Cost Accounting
```

---

## Reading Paths

**"I want to build a minimal working agent"**
→ [01-core-runtime.md](01-core-runtime.md) → [03-minimal-agents.md](03-minimal-agents.md)

**"I need to understand the tool/permission system"**
→ [01-prompts-permissions.md](01-prompts-permissions.md) → [01-planning-hooks.md](01-planning-hooks.md)

**"I'm adding memory and RAG"**
→ [01-memory-rag-knowledge.md](01-memory-rag-knowledge.md) → [00-research-foundation.md](00-research-foundation.md)

**"I'm building a multi-agent system"**
→ [01-event-driven-multiagent.md](01-event-driven-multiagent.md) → [01-distributed-agents.md](01-distributed-agents.md)

**"I need auth (OAuth, API keys, cloud)"**
→ [02-auth-overview-apikey.md](02-auth-overview-apikey.md) → [02-auth-oauth-token-keychain.md](02-auth-oauth-token-keychain.md)

**"I need production-ready implementations"**
→ [03-production-agents.md](03-production-agents.md) → [03-utilities-mcp-checklist.md](03-utilities-mcp-checklist.md)

**"I want to minimize API cost with prompt caching"**
→ [docs/21-prompt-caching.md](docs/21-prompt-caching.md)

**"I want to add IDE/LSP/plugin support"**
→ [01-extensions-sdk-mcp.md](01-extensions-sdk-mcp.md)

**"I want to add debugger support to my agent"**
→ [05-dap-debug-integration.md](05-dap-debug-integration.md) → [05-dap-claude-code-plugin.md](05-dap-claude-code-plugin.md)

---

## Source Codebases

| Codebase | Language | Notable Features |
|----------|----------|-----------------|
| Claude Code | TypeScript | Concurrent tool exec (10×), compaction, subscription auth |
| Gemini CLI | TypeScript | 1M context, 7-strategy model routing, A2A server |
| Claw Code | Rust | 43 tools, recovery recipes, worker lifecycle |
| Claurst | Rust | 35+ providers, AutoDream memory, session branching, TUI |
