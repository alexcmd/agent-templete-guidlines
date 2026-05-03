# Hidden Techniques from Source Code

Patterns and capabilities found in the Claude Code source that are undocumented or not prominent in official guides. Sourced from direct source code analysis of the npm sourcemap reconstruction.

---

## 1. asyncRewake Hooks — Background Polling with Model Wakeup

Standard hooks are either synchronous (block Claude while running) or async (fire-and-forget). `asyncRewake` is a third mode: runs in background, but can **wake the model** by exiting with code 2.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/run-tests-background.sh",
            "asyncRewake": true,
            "if": "Bash(npm test *)"
          }
        ]
      }
    ]
  }
}
```

**Hook script** (`run-tests-background.sh`):
```bash
#!/usr/bin/env bash
# Runs after every npm test command, in background

# Wait for test completion (the npm test already ran, this is post-test analysis)
RESULTS=$(cat /tmp/test-results.txt 2>/dev/null)

if echo "$RESULTS" | grep -q "FAIL"; then
    FAILURES=$(echo "$RESULTS" | grep "FAIL")
    echo "{\"systemMessage\": \"Tests failed! Failures:\\n${FAILURES}\"}"
    exit 2   # Exit code 2 = wake the model
fi
exit 0       # Normal exit = don't disturb model
```

Use cases:
- CI pipeline watching (long-running jobs)
- File system monitoring
- External service polling
- Test result watching after code changes

Source: `src/schemas/hooks.ts` — `asyncRewake` field in BashCommandHookSchema.

---

## 2. `once` Hooks — Single-Execution Hooks

Any hook type supports `once: true` — the hook executes exactly once and is then removed from the registry for the session.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/one-time-setup.sh",
            "once": true
          }
        ]
      }
    ]
  }
}
```

Use cases:
- One-time context injection at session start
- Setup operations that should only happen once
- First-run prompts or warnings
- Initialization scripts

Source: `src/schemas/hooks.ts` — `once` field in all hook schemas.

---

## 3. Dynamic Skill Discovery (Sub-directory .claude/skills/)

Not documented but fully implemented: Claude Code automatically discovers `.claude/skills/` directories **nested inside the project** as the model touches files.

```
project/
  src/
    api/
      .claude/
        skills/
          api-review/
            SKILL.md     ← Auto-discovered when model reads src/api/anything.ts
    frontend/
      .claude/
        skills/
          react-patterns/
            SKILL.md     ← Auto-discovered when model reads src/frontend/**
```

**Discovery trigger**: After any `Read`, `Write`, or `Edit` tool call, Claude Code walks up the directory tree from the touched file to `cwd`, checking for `.claude/skills/` at each level.

**Gitignore protection**: `node_modules/`, `.git/`, and other gitignored directories are never discovered (uses `git check-ignore`).

**Depth priority**: Deeper directories win over shallower ones (skill in `src/api/.claude/skills/` overrides same-named skill in `src/.claude/skills/`).

Practical pattern:
```bash
# Create module-specific skills that only activate when working in that module
mkdir -p src/api/.claude/skills/api-patterns
cat > src/api/.claude/skills/api-patterns/SKILL.md << 'EOF'
---
description: "Review API endpoint code for REST conventions and error handling"
---
Review this API code for:
- HTTP method correctness (GET = read-only, POST = create, etc.)
- Proper status codes (201 for create, 404 for not found, etc.)  
- Error response format matching our APIError schema
- Input validation using our validateRequest middleware
- Auth middleware applied to protected routes
EOF
```

Source: `src/skills/loadSkillsDir.ts` — `discoverSkillDirsForPaths()` and `addSkillDirectories()`.

---

## 4. Conditional Skills (Path-Triggered Activation)

Skills with `paths` frontmatter remain **dormant** until the model operates on a matching file:

```markdown
---
description: "React component review checklist"
paths: |
  src/components/**/*.tsx
  src/pages/**/*.tsx
  !src/**/*.test.tsx
---
## React Component Review

When reviewing this component, check:
- Props interface defined with TypeScript
- useCallback for event handlers passed to children
- useMemo for expensive derived values
- Key prop on list items
- Accessible aria attributes
```

This skill is **invisible to the model** until the model opens a file matching the path patterns. Then it activates and remains available for the rest of the session.

**Activation log** (visible with `--debug`):
```
[skills] Activated conditional skill 'react-review' (matched path: src/components/Button.tsx)
```

**Pattern syntax**: Uses `ignore` library (same as `.gitignore`):
- `src/**.tsx` — any `.tsx` under `src/`
- `!src/**/*.test.tsx` — exclude test files
- `**/config.json` — any config.json anywhere

Source: `src/skills/loadSkillsDir.ts` — `parseSkillPaths()`, `activateConditionalSkillsForPaths()`.

---

## 5. Skill Shell Injection with Special Variables

Skills have access to two special runtime variables:

### `${CLAUDE_SKILL_DIR}` — Reference co-located scripts

```markdown
---
description: "Run custom analysis"
allowed-tools: "Bash"
---
Run this analysis script: !`${CLAUDE_SKILL_DIR}/analyze.sh`

Then review the output and suggest improvements.
```

Enables shipping self-contained skills with companion scripts:
```
my-skill/
  SKILL.md                    # References ${CLAUDE_SKILL_DIR}/analyze.sh
  analyze.sh                  # Script invoked from SKILL.md
  templates/
    report-template.md
```

### `${CLAUDE_SESSION_ID}` — Session-unique temp files

```markdown
---
description: "Collect metrics for this session"
---
Store metrics in: !`mkdir -p /tmp/claude-${CLAUDE_SESSION_ID} && echo /tmp/claude-${CLAUDE_SESSION_ID}`

This ensures metrics are isolated per Claude session.
```

Source: `src/skills/loadSkillsDir.ts` — `getPromptForCommand()` closure.

---

## 6. Agent Persistent Memory

Agents with `memory` set get a managed memory directory that persists across sessions:

```markdown
---
name: project-tracker
description: "Tracks project progress and context across sessions"
memory: project
tools: Read, Write, Edit
---
You are a project tracking agent with persistent memory.

Your memory files are maintained across sessions. At session start, review your memory to restore context. Update memory when significant progress is made.
```

Memory structure auto-managed:
```
.claude/memory/
  project-tracker/     # 'project' scope
    MEMORY.md          # Agent writes and reads this
    notes.md           # Agent can create additional files

~/.claude/memory/
  project-tracker/     # 'user' scope — survives project switches
    MEMORY.md
```

**Memory injection**: The memory directory's contents are loaded into the agent's system prompt on each invocation. The agent uses `Read`, `Write`, `Edit` to maintain its memory (these tools are auto-added to the allowlist).

**Snapshot system**: Agents can have memory "snapshots" (committed to `.claude/memory-snapshots/`) that are distributed to new contributors. When a contributor first runs an agent with `memory: 'user'` and no local memory exists, the project snapshot is copied to their local memory dir.

Source: `src/tools/AgentTool/agentMemory.ts`, `src/tools/AgentTool/agentMemorySnapshot.ts`.

---

## 7. Worktree Isolation for Agents

Agents with `isolation: worktree` run in a temporary isolated git worktree:

```markdown
---
name: safe-migrator
description: "Performs database migrations in an isolated worktree for safety"
isolation: worktree
tools: Bash, Read, Write, Edit
---
You perform database schema migrations.

Always:
1. Review current schema
2. Create the migration file
3. Run tests in isolated mode
4. Report what changed

Your changes are isolated until reviewed.
```

**What happens**:
1. Git worktree created at temp path (separate checkout of same repo)
2. Agent runs with that worktree as its working directory
3. File operations affect only the worktree clone
4. On completion: if changes were made, parent receives worktree path + branch for review
5. If no changes: worktree is silently deleted

**Return value**: The `AgentTool` result includes `worktree_path` and `branch` when isolation is used.

Source: `src/tools/AgentTool/AgentTool.tsx`, `EnterWorktree`/`ExitWorktree` tools.

---

## 8. Agent Background Mode

Agents with `background: true` are always spawned as background Tasks (non-blocking):

```markdown
---
name: audit-logger
description: "Logs all file operations to the audit trail"
background: true
tools: Write, Read
maxTurns: 5
---
Log this operation to the audit trail at ~/.claude/audit.log.

Include timestamp, operation type, file path, and session ID.
```

The parent agent receives a `Task` object rather than waiting for completion. The parent can continue working while the background agent runs.

Useful for:
- Audit logging
- Async notification systems
- Non-blocking analysis
- Parallelizable independent work

Source: `src/tools/AgentTool/loadAgentsDir.ts` — `background` field in AgentDefinition.

---

## 9. Initial Prompt with Slash Commands

The `initialPrompt` field in agent definitions runs slash commands before the first user turn:

```markdown
---
name: pre-configured-agent
description: "Agent that starts with a specific context"
initialPrompt: "/doctor\n/reload-plugins"
---
You start your session by running diagnostics and reloading plugins.
```

Slash commands in `initialPrompt` are fully resolved — they expand to their skill content before being injected into the conversation.

Useful for:
- Pre-loading context skills
- Running setup checks
- Injecting dynamic environment information at agent start

Source: `src/tools/AgentTool/loadAgentsDir.ts` — `initialPrompt` field.

---

## 10. requiredMcpServers — Conditional Agent Availability

Agents can be hidden from the model when required MCP servers are not configured:

```markdown
---
name: linear-manager
description: "Manages Linear issues and projects"
requiredMcpServers:
  - linear
mcpServers:
  - linear
---
You manage Linear issues...
```

If the `linear` MCP server is not in `mcpServers` config, this agent is filtered out of the agent list before the model sees it. Prevents the model from trying to use an agent that would fail.

Pattern matching is case-insensitive substring: `"linear"` matches server names containing "linear" (e.g., `linear-prod`, `linear-dev`).

Source: `src/tools/AgentTool/loadAgentsDir.ts` — `hasRequiredMcpServers()`, `filterAgentsByMcpRequirements()`.

---

## 11. PermissionRequest Hook — Intercept Permission Prompts

Intercept and auto-approve/deny permission requests before showing them to the user:

```json
{
  "hooks": {
    "PermissionRequest": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/auto-approve-git.sh",
            "if": "Bash(git *)"
          }
        ]
      }
    ]
  }
}
```

**Hook script** (`auto-approve-git.sh`):
```bash
#!/usr/bin/env bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('tool_input',{}).get('command',''))")

# Auto-approve safe git operations
if [[ "$COMMAND" =~ ^git\ (status|log|diff|show|branch|fetch|pull).*$ ]]; then
    echo '{
      "hookSpecificOutput": {
        "hookEventName": "PermissionRequest",
        "decision": {
          "behavior": "allow"
        }
      }
    }'
fi
```

Also supports adding permanent rules:
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow",
      "updatedPermissions": [
        {
          "rule": "Bash(git status)",
          "behavior": "allow"
        }
      ]
    }
  }
}
```

Source: `src/types/hooks.ts` — `PermissionRequest` in `syncHookResponseSchema`.

---

## 12. PostToolUse MCP Output Modification

PostToolUse hooks can modify the output that MCP tools return to the model:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "mcp__my-database__query",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/sanitize-db-output.py",
            "if": "mcp__my-database__query"
          }
        ]
      }
    ]
  }
}
```

**Hook script** (sanitize-db-output.py):
```python
import json, sys, os

data = json.loads(os.environ.get('CLAUDE_TOOL_INPUT', sys.stdin.read()))
original_output = data.get('tool_output', {})

# Redact sensitive fields
if isinstance(original_output, dict):
    for key in ['password', 'token', 'secret', 'ssn']:
        if key in original_output:
            original_output[key] = '[REDACTED]'

print(json.dumps({
    "hookSpecificOutput": {
        "hookEventName": "PostToolUse",
        "updatedMCPToolOutput": original_output,
        "additionalContext": "Sensitive fields were redacted from database output"
    }
}))
```

Source: `src/types/hooks.ts` — `PostToolUse` in `syncHookResponseSchema` with `updatedMCPToolOutput`.

---

## 13. SessionStart File Watchers

The `SessionStart` hook can register file paths to watch for changes:

```bash
#!/usr/bin/env bash
# SessionStart hook — register files to watch

echo '{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "Watching config files for changes",
    "watchPaths": [
      "/home/user/project/.env",
      "/home/user/project/config/settings.yaml",
      "/home/user/project/docker-compose.yml"
    ]
  }
}'
```

When watched files change, `FileChanged` hooks fire. The `FileChanged` hook receives the path and can inject context:

```bash
#!/usr/bin/env bash
# FileChanged hook

INPUT=$(cat)
FILE=$(echo "$INPUT" | python3 -c "import json,sys; print(json.load(sys.stdin).get('tool_input',{}).get('file_path',''))")

echo "{
  \"hookSpecificOutput\": {
    \"hookEventName\": \"FileChanged\",
    \"additionalContext\": \"File changed: ${FILE}. The model should re-read it if relevant.\"
  }
}"
```

Source: `src/types/hooks.ts` — `SessionStart` and `FileChanged` in `syncHookResponseSchema`.

---

## 14. Undercover Mode Detection

The source code contains an "undercover mode" that prevents Claude from revealing internal codenames when accessed via certain channels. In source (`utils/undercover.ts`):

- Blocks model from saying internal names: "Capybara", "Tengu", "Kairos", "Kestrel"
- Active in certain entrypoints (IDE bridge, remote sessions)
- Does not affect external API usage or standard CLI

This is relevant if you're building tools that inspect model responses for metadata — the model won't reveal internal version names in certain contexts.

---

## 15. Coordinator Mode for Agent Swarms

A coordinator mode exists for distributed agent execution (`CLAUDE_CODE_COORDINATOR_MODE=1`):

```bash
CLAUDE_CODE_COORDINATOR_MODE=1 claude
```

In coordinator mode, the agent set is replaced by "coordinator agents" from `src/coordinator/workerAgent.ts`. This enables:
- Distributed work across multiple Claude instances
- Worker pool management
- Result aggregation
- Cross-process agent communication

This is primarily an Anthropic-internal / enterprise feature. The code path is:
```typescript
if (isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)) {
  const { getCoordinatorAgents } = require('../../coordinator/workerAgent.js')
  return getCoordinatorAgents()
}
```

---

## 16. Skill `disable-model-invocation`

Skills with `disable-model-invocation: true` run their shell injection but **don't send anything to the model**:

```markdown
---
description: "Run pre-commit checks without model involvement"
disable-model-invocation: true
allowed-tools: "Bash"
---
```!
npm run lint && npm test && echo "All checks passed"
```
```

This makes the skill a pure shell executor — useful for:
- Running automated checks
- Generating reports without model interpretation
- Triggering side effects

The shell output is shown to the user but not processed by the model.

Source: `src/skills/loadSkillsDir.ts` — `disableModelInvocation` field in `createSkillCommand`.

---

## 17. Hook System Interaction with Permission Rules

Hook `if` conditions use **permission rule syntax** — the same language as `permissions.allow`. This is evaluated **before spawning the hook process**, as a fast filter.

The evaluation uses the same matcher as the permission system:
```
Bash(git *)       → True when tool is 'Bash' AND command starts with 'git '
Write(**.ts)      → True when tool is 'Write' AND path ends with .ts
mcp__github__*    → True when tool name matches mcp__github__ prefix
```

Since the filter runs synchronously before process spawn, complex patterns save significant overhead — you avoid spawning Python/Node.js processes for every file write when you only care about TypeScript files.

Source: `src/schemas/hooks.ts` — `IfConditionSchema`.

---

## 18. Agent Color System

All 20 named colors are available for custom agents:

```
red, orange, yellow, green, blue, purple, pink, cyan,
magenta, teal, lime, amber, violet, rose, sky, indigo,
emerald, fuchsia, white, gray
```

Colors are auto-assigned if not specified (cycles through the palette). Custom colors let you build visually organized swarm dashboards where each agent has a distinct color.

Source: `src/tools/AgentTool/agentColorManager.ts` — `AGENT_COLORS` array.
