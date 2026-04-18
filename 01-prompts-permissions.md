# Universal Agent Architecture — System Prompt & Permissions

> Sections: System Prompt Structure · Permission Architecture

Part of the [Universal Agent Architecture](01-overview.md) series.

## 5. System Prompt Structure

The system prompt has two sections separated by a **cache boundary**. Everything above the
boundary is invariant within a session and benefits from prompt caching; everything below it
changes per-turn.

```
══ STATIC (cached) ══════════════════════════════════════════
[1] Role introduction
    "You are an interactive agent that helps with X..."

[2] Capability declaration (optional output style section)

[3] Behavioral constraints
    "# Doing tasks\n - Prefer editing existing files..."

[4] Safety rules
    "# Executing actions with care\n..."

[5] Tool usage guidance
    "# Using your tools\n..."

══ CACHE BOUNDARY ═══════════════════════════════════════════

══ DYNAMIC (per-turn) ═══════════════════════════════════════
[6] Environment context
    "# Environment\n - Model: X\n - Date: Y\n - CWD: Z"

[7] Project context (if applicable)
    git status + diff + recent commits (budget: ~2 KB)

[8] Instruction files (CLAUDE.md / AGENT.md equivalent)
    Budget: 4 KB per file, 12 KB total

[9] Runtime config snapshot
    permission mode, allowed tools, MCP server status
```

**Why this matters:** Anthropic's prompt caching saves 90% of input token cost on the static
section. The boundary must be placed after the last static block and before the first dynamic
block. Moving a dynamic value (like a timestamp) above the boundary breaks caching for all
subsequent calls.

---

## 6. Permission Architecture

A three-tiered permission check must run before every tool execution:

```
Tool call arrives
      │
      ▼
┌─────────────────────────┐
│ 1. Deny rules (static)  │──── match ────► DENY immediately
└──────────┬──────────────┘
           │ no match
           ▼
┌─────────────────────────┐
│ 2. Hook override        │──── Deny ─────► DENY
│    (PreToolUse hook)    │──── Ask ──────► ASK user
└──────────┬──────────────┘──── Allow ────► ALLOW
           │ no override
           ▼
┌─────────────────────────┐
│ 3. Policy rules         │──── ask match ► ASK user
│    allow / ask rules    │──── allow match► ALLOW
└──────────┬──────────────┘
           │ no match
           ▼
┌─────────────────────────────────────┐
│ 4. Default: mode >= required_mode?  │──── YES ──► ALLOW
└──────────┬──────────────────────────┘
           │ NO
           ▼
           ASK user → Allow / Deny / Allow-all-session
```

**Permission modes (full list):**

| Mode | What it allows automatically |
|------|------------------------------|
| `readonly` | File reads, searches, queries only. All writes → DENY |
| `acceptEdits` | File writes allowed without prompting. Shell → ASK |
| `default` | Read-only tools → ALLOW. Write/shell → ASK |
| `plan` | Plan-phase enforcement: read-only + write only to plans directory |
| `auto` | Classifier-driven auto-approval (uses model to judge safety) |
| `bypassPermissions` / `yolo` | All tools → ALLOW. No prompts, no hooks (use only for CI/trusted pipelines) |

Every tool declares its `required_permission`. The active mode must satisfy the requirement
for auto-allow; otherwise the permission engine asks the user.

**PermissionRule schema:**

```python
@dataclass
class PermissionRule:
    source: str          # "userSettings"|"projectSettings"|"localSettings"|"cliArg"|"session"
    behavior: str        # "allow"|"deny"|"ask"
    tool_name: str       # e.g. "Bash", "Write"
    rule_content: str    # glob/pattern applied to serialized input, e.g. "git *", "*.py"
```

**Rule expression syntax:** `ToolName(glob)` — e.g. `Bash(git *)`, `Write(src/*)`, `Read(*)`

**Per-directory scoping:** Add `additional_working_directories` in config to grant write
permission only within specified paths. Write attempts outside those paths → DENY or ASK.

**Runtime dialog options (when ASK triggers):**
1. Allow once
2. Deny once
3. Allow for this session
4. Allow permanently (writes to `userSettings`)
5. Edit input in-place before allowing (user can modify the tool's input)

**Multi-agent permission inheritance:**
- Sub-agents inherit the parent's permission mode and rules by default
- Coordinator agents (supervisor role) can grant additional permissions to workers
- Workers cannot self-elevate permissions — they must request via coordinator
- Recommended: workers run in `default` mode; coordinator approves escalations

---


---

*[← Core Runtime](01-core-runtime.md) | [Next: Memory, RAG & Knowledge Graphs →](01-memory-rag-knowledge.md)*
