# Claude Code Extension Development — Master Index

Comprehensive guide to building external extensions for Claude Code, derived from deep source code analysis of `/home/deck/Projects/claude-code/src` (the leaked npm sourcemap reconstruction of the Anthropic CLI).

## What "Extension" Means in Claude Code

Claude Code has five distinct extension mechanisms. Each solves a different problem:

| Mechanism | What it does | Who invokes it | Scope |
|-----------|-------------|----------------|-------|
| **Skill** | Markdown prompt template injected into conversation | User (`/skill-name`) or Model (via SkillTool) | user / project / plugin |
| **Agent** | Isolated sub-context with its own system prompt and tool set | Model (via AgentTool) | user / project / plugin |
| **Hook** | Shell/HTTP/LLM callback at lifecycle events | Runtime (automatic) | any settings source |
| **Plugin** | Bundle of skills + hooks + MCP servers, togglable in UI | System at startup | marketplace / builtin |
| **MCP Server** | External process providing tools + resources over a protocol | Runtime (tools available to model) | user / project / settings |

---

## Document Map

### Core Extension Types
- [01-skills-commands.md](01-skills-commands.md) — Skills & slash commands: authoring, frontmatter, argument substitution, conditional activation, shell execution
- [02-agents.md](02-agents.md) — Custom agents: markdown definition, JSON format, all frontmatter fields, tool filtering, memory, isolation
- [03-hooks.md](03-hooks.md) — Hooks system: all 27 events, 4 hook types, full I/O schemas, per-event hookSpecificOutput
- [04-plugins.md](04-plugins.md) — Plugin system: manifest format, components, marketplace, built-in vs loaded plugins, lifecycle
- [05-mcp-servers.md](05-mcp-servers.md) — MCP integration: transport types, resource support, OAuth, tool wrapping, MCP-based skills

### Cross-Cutting Concerns
- [06-settings-permissions.md](06-settings-permissions.md) — Settings hierarchy (5 sources), permission rules syntax, permission modes, deny/allow/ask
- [07-hidden-techniques.md](07-hidden-techniques.md) — Undocumented patterns from source: dynamic skill discovery, conditional skills, asyncRewake hooks, `once` hooks, worktree isolation, agent memory, shell prompt injection

### MCP Deep Dive
- [08-mcp-registration-deep.md](08-mcp-registration-deep.md) — All 7 registration paths, full config type reference, plugin formats (4 kinds), enterprise allowlist/denylist, `.mcp.json` traversal, agent MCP declaration, CLI `mcp add` commands
- [09-mcp-workflow-internals.md](09-mcp-workflow-internals.md) — Full connection lifecycle, tool wrapping pipeline, tool call execution, OAuth flow, MCPB/DXT bundle workflow, elicitation protocol, content size management

---

## Quick Start by Use Case

**"I want to add a reusable prompt I can call with `/`"**  
→ Create a skill: [01-skills-commands.md](01-skills-commands.md)

**"I want a specialized sub-agent with its own personality and tools"**  
→ Create an agent: [02-agents.md](02-agents.md)

**"I want to auto-run a shell command when Claude writes a file"**  
→ Add a PostToolUse hook: [03-hooks.md](03-hooks.md)

**"I want to ship a bundle of skills + hooks to other users"**  
→ Build a plugin: [04-plugins.md](04-plugins.md)

**"I want Claude to access my API/database as a tool"**  
→ Write an MCP server: [05-mcp-servers.md](05-mcp-servers.md)

**"I want to understand the permission system"**  
→ Read [06-settings-permissions.md](06-settings-permissions.md)

**"I want techniques not in the official docs"**  
→ Read [07-hidden-techniques.md](07-hidden-techniques.md)

**"I need to register an MCP server in a plugin, agent, or enterprise config"**  
→ Read [08-mcp-registration-deep.md](08-mcp-registration-deep.md)

**"I want to understand how MCP connections, tool calls, and OAuth work internally"**  
→ Read [09-mcp-workflow-internals.md](09-mcp-workflow-internals.md)

---

## Directory Layout (where files go)

```
~/.claude/                        # User-scope (all projects)
  settings.json                   # User settings: hooks, permissions, mcpServers, enabledPlugins
  skills/
    my-skill/
      SKILL.md                    # Skill definition
  agents/
    my-agent.md                   # Agent definition
  commands/                       # DEPRECATED — use skills/ instead
    my-command.md

.claude/                          # Project-scope (current repo)
  settings.json                   # Project settings (committed)
  settings.local.json             # Local-only settings (gitignored)
  skills/
    my-skill/
      SKILL.md
  agents/
    my-agent.md
  commands/                       # DEPRECATED

# Sub-directory skills (dynamic discovery)
src/.claude/skills/               # Discovered when files in src/ are touched
  ts-skill/
    SKILL.md
```

---

## Source File Reference

Key source files for extension development:

| File | Purpose |
|------|---------|
| `src/skills/loadSkillsDir.ts` | Skill loading, frontmatter parsing, conditional activation |
| `src/tools/AgentTool/loadAgentsDir.ts` | Agent loading, JSON/markdown parsing |
| `src/schemas/hooks.ts` | Hook Zod schemas (canonical source of truth) |
| `src/types/hooks.ts` | Hook output types, HookResult, per-event hookSpecificOutput |
| `src/plugins/builtinPlugins.ts` | Built-in plugin registry |
| `src/entrypoints/sdk/coreTypes.ts` | HOOK_EVENTS array (all 27 events) |
| `src/commands.ts` | Command type definitions, getCommands() |
| `src/Tool.ts` | Tool interface (for MCP tool wrapping) |
| `src/services/mcp/types.ts` | McpServerConfig schemas |
