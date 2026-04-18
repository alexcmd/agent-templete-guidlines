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
| 06 | [Commands & Skills](./06-commands-skills.md) | Slash commands, skills, plugin commands |
| 07 | [Memory & Context](./07-memory-context.md) | Memory directory, system prompt assembly, context injection |
| 08 | [Permissions & Hooks](./08-permissions-hooks.md) | Permission modes, rules, lifecycle hooks |
| 09 | [Configuration System](./09-configuration.md) | Settings hierarchy, env vars, MDM, feature flags |
| 10 | [Plugin System](./10-plugin-system.md) | Plugin manifest, lifecycle, MCP integration |
| 11 | [Multi-Agent Orchestration](./11-multi-agent.md) | Coordinator mode, sub-agents, task system |
| 12 | [UI Layer](./12-ui-layer.md) | React/Ink REPL, components, rendering |
| 13 | [Use Cases & Scenarios](./13-use-cases.md) | All major usage scenarios with flow diagrams |
| 14 | [Reproduction Blueprint](./14-reproduction-blueprint.md) | Step-by-step guide to build a domain-specific agent |
| 15 | [Diagrams](./15-diagrams.md) | ASCII architecture, sequence, and component diagrams |

## Quick Summary

Claude Code is a **terminal-first AI coding assistant** built on these core principles:

1. **Tool-augmented LLM loop** — Claude calls tools (Bash, file ops, web, agents) in a streaming loop until task is done
2. **Layered permission system** — every tool call passes through configurable allow/deny/ask rules before execution
3. **Hierarchical configuration** — global → project → local → managed → MDM settings cascade
4. **Plugin architecture** — skills, commands, hooks, and MCP servers are all pluggable
5. **Multi-agent orchestration** — spawns sub-agents with isolated contexts and coordinates via task messaging
6. **Memory injection** — loads `.claude/` markdown files into context before each query
