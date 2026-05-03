# Claude Code — Complete Architecture & Reproduction Blueprint

This documentation set reverse-engineers Claude Code from source to give you everything needed to build a comparable AI coding agent for a different business domain.

## Documents

| # | File | Contents |
|---|------|----------|
| 01 | [Architecture Overview](./01-architecture-overview.md) | High-level system design, subsystems, and layering |
| 02 | [Data Flow & Execution](./02-data-flow.md) | End-to-end request lifecycle, streaming, tool loop |
| 03 | [Core Data Structures](./03-data-structures.md) | All key TypeScript types, interfaces, and state shapes |
| 04 | [Tools System](./04-tools-system.md) | Tool interface, all built-in tools, permission model |
| 05 | [Agent Loop & Query Engine](./05-agent-loop.md) | QueryEngine, query.ts, compaction, transitions |
| 06 | [Commands & Skills](./06-commands-skills.md) | Slash commands (~90), immediate vs queued, remote/bridge-safe sets, skills, plugin commands |
| 07 | [Memory & Context](./07-memory-context.md) | Memory directory, system prompt assembly, context injection, prompt cache system (getCacheControl, cache_edits, break detection, session latches) |
| 08 | [Permissions & Hooks](./08-permissions-hooks.md) | Permission modes, rules, lifecycle hooks |
| 09 | [Configuration System](./09-configuration.md) | Settings hierarchy, env vars, MDM, feature flags |
| 10 | [Plugin System](./10-plugin-system.md) | Plugin manifest, lifecycle, MCP integration |
| 11 | [Multi-Agent Orchestration](./11-multi-agent.md) | Coordinator mode, sub-agents, task system |
| 12 | [UI Layer](./12-ui-layer.md) | React/Ink REPL, components, rendering |
| 13 | [Use Cases & Scenarios](./13-use-cases.md) | All major usage scenarios with flow diagrams |
| 14 | [Reproduction Blueprint](./14-reproduction-blueprint.md) | Step-by-step guide to build a domain-specific agent |
| 15 | [Diagrams](./15-diagrams.md) | ASCII architecture, sequence, and component diagrams |
| 16 | [Complete Prompt Documentation](./16-prompts-complete.md) | Every prompt in Claude Code — system prompt sections, all 27 tool descriptions, 5 built-in agent system prompts, compaction/coordinator/proactive prompts, service prompts (SessionMemory, MagicDocs, ExtractMemories), buddy prompt, cache strategy, and dynamic combination matrix |
| 17 | [Speculative Execution](./17-speculative-execution.md) | SpeculationState machine, mutable-ref streaming optimization, prompt suggestion, CompletionBoundary, timeSavedMs, reproduction in Python |
| 18 | [Compaction Deep Dive](./18-compaction-deep-dive.md) | Threshold arithmetic, cached microcompact (cache_edits API), time-based microcompact, session memory compaction, full autocompact, circuit breaker |
| 19 | [Abort Signal Cascade](./19-abort-cascade.md) | Three-level AbortController hierarchy, sibling-abort (Bash-only), permission rejection bubble-up, tool interruptBehavior, synthetic error messages |
| 20 | [Session Memory Extraction](./20-session-memory-extraction.md) | Background forked subagent extraction, token+tool-call thresholds, sequential wrapper, file isolation (0o600/0o700), manual /summary command |
| 21 | [Prompt Caching Deep Dive](./21-prompt-caching.md) | 4-slot budget, system prompt splitting (3 modes: global/tool-based/default), single message-level marker, scope/TTL decision (org 5m/1h, global 24h), usage tracking (cache_creation/cache_read/ephemeral_1h/5m), 2-phase break detection, cache_edits microcompact |
| 22 | [Bridge Protocol](./22-bridge-protocol.md) | Connecting external apps to Claude Code — OAuth registration, long-poll work queue, WorkSecret decode, v1 WebSocket vs v2 SSE transports, all 20 control request subtypes, token refresh, Python client + workflow examples |

## Quick Summary

Claude Code is a **terminal-first AI coding assistant** built on these core principles:

1. **Tool-augmented LLM loop** — Claude calls tools (Bash, file ops, web, agents) in a streaming loop until task is done
2. **Layered permission system** — every tool call passes through configurable allow/deny/ask rules before execution
3. **Hierarchical configuration** — global → project → local → managed → MDM settings cascade
4. **Plugin architecture** — skills, commands, hooks, and MCP servers are all pluggable
5. **Multi-agent orchestration** — spawns sub-agents with isolated contexts and coordinates via task messaging
6. **Memory injection** — loads `.claude/` markdown files into context before each query
