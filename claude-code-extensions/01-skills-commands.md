# Skills & Slash Commands

Skills are the primary extension mechanism for user-invocable prompt templates. When a user types `/my-skill [args]`, or the model calls `SkillTool` with the skill name, Claude Code loads the skill's markdown body, substitutes arguments, executes any inline shell commands, and injects the result as a user message.

---

## Directory Structure

Skills use a **directory format** — each skill lives in its own folder:

```
~/.claude/skills/              # User scope (all projects)
  my-skill/
    SKILL.md                   # Required: the skill definition
    helper.sh                  # Optional: scripts referenced via ${CLAUDE_SKILL_DIR}

.claude/skills/                # Project scope (current repo, committed)
  my-skill/
    SKILL.md

.claude/settings.local.json    # Local scope (gitignored) — hooks/permissions only

src/some-module/.claude/skills/ # Sub-directory scope (dynamically discovered)
  module-skill/
    SKILL.md
```

> **Legacy format**: Single `.md` files in `.claude/commands/` still work but are deprecated. New skills must use the `skill-name/SKILL.md` directory format under `skills/`.

---

## SKILL.md Format

```markdown
---
# Required
description: "One-line description shown in /help and to the model"

# Optional metadata
name: "Display Name"           # Human-readable name (defaults to directory name)
argument-hint: "<query>"       # Hint shown after the skill name in autocomplete
arguments: "arg1, arg2"        # Named arguments for $arg1/$arg2 substitution
when_to_use: |                 # Multi-line guidance for when model should call this skill
  Use this when the user asks to review code.
  Especially useful for TypeScript files.

# Execution control
context: fork                  # 'fork' = run in isolated sub-agent; omit = inline expansion
agent: my-agent                # Specific agent type to use when context: fork
model: claude-sonnet-4-6       # Override model for this skill; 'inherit' = use session model
effort: normal                 # 'quick' | 'normal' | 'thorough' | integer (token budget)
allowed-tools: "Bash, Read"    # Comma-separated tool whitelist (empty = all tools)
disable-model-invocation: false # true = skill runs without calling the model (pure shell)
user-invocable: true           # false = model-only skill (hidden from /help)
version: "1.0.0"               # Semantic version for cache busting

# Conditional activation (gitignore-style patterns)
paths: |
  src/**.ts
  src/**.tsx

# Hooks (merged with session hooks while skill runs)
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: echo "About to run bash from my-skill"
---

Your skill body goes here. This markdown is injected into the conversation as a user message.

## What I want Claude to do

You can use `$1`, `$2` for positional arguments, or `$arg1`, `$arg2` for named arguments.

Example with positional: review the file at $1
Example with named: the query is "$query"

## Shell injection (trusted skills only)

You can embed shell output using backtick syntax:
Current directory: !`pwd`
Files changed: !`git diff --name-only`

Or fenced code blocks with `!` language:
```!
git log --oneline -5
```

## Special variables

- `${CLAUDE_SKILL_DIR}` — absolute path to this skill's directory
- `${CLAUDE_SESSION_ID}` — current session ID
```

---

## Frontmatter Field Reference

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `description` | string | (from H1 or first line) | Shown in `/help` and tool listings |
| `name` | string | directory name | Display name (does not change invocation name) |
| `argument-hint` | string | — | Autocomplete hint, e.g. `<file-path>` |
| `arguments` | string / list | — | Named arg names, comma-separated or YAML list |
| `when_to_use` | string | — | Model guidance for when to invoke this skill |
| `context` | `"fork"` | inline | `fork` = run in isolated agent sub-context |
| `agent` | string | — | Agent type to use when `context: fork` |
| `model` | string | session model | Model override; `"inherit"` = use session model |
| `effort` | string / int | — | Thinking budget: `quick`/`normal`/`thorough` or token int |
| `allowed-tools` | string | all tools | Comma-separated tool allow-list |
| `disable-model-invocation` | bool | `false` | Skip model call; output only from shell injection |
| `user-invocable` | bool | `true` | `false` = model-only (hidden from user-facing `/help`) |
| `version` | string | — | Version string for cache management |
| `paths` | string / list | — | Gitignore patterns; skill only activates when matching files touched |
| `hooks` | object | — | HooksSettings merged with session hooks while skill is active |

---

## Argument Substitution

Claude Code supports two substitution styles:

### Positional (`$1`, `$2`, ...)
```markdown
Review the code in $1 and focus on $2
```
Invoked as: `/my-skill src/main.ts security`

### Named (`$arg_name`)
```markdown
---
arguments: "filepath, focus_area"
---
Review the code in $filepath and focus on $focus_area
```
Invoked as: `/my-skill src/main.ts security` (positional mapping to names)
Or the model calls it with named kwargs.

### Fallback
If no args are provided, `$1` and `$arg_name` expand to empty strings.

---

## Shell Injection in Skill Body

Skills can embed live shell output directly in the prompt. This runs when the skill is invoked, before the content is sent to the model.

**Inline backtick form:**
```markdown
The current git branch is: !`git branch --show-current`
Changed files: !`git diff --name-only HEAD`
```

**Fenced code block form:**
```markdown
Recent commits:
```!
git log --oneline -10
```
```

**Important security note**: MCP-sourced skills never execute shell injection — only user/project/managed skills can. This is enforced in `loadSkillsDir.ts:374`:
```typescript
if (loadedFrom !== 'mcp') {
  finalContent = await executeShellCommandsInPrompt(finalContent, ...)
}
```

**`allowed-tools` interacts with shell injection**: Tools listed in `allowed-tools` are added to the `alwaysAllowRules` during shell execution, so shell commands only need permission for those tools.

---

## Special Variables Available in Skill Body

```
${CLAUDE_SKILL_DIR}    → Absolute path to skill's own directory
                          Use to reference co-located scripts:
                          !`${CLAUDE_SKILL_DIR}/helper.sh`

${CLAUDE_SESSION_ID}   → Current session UUID
                          Useful for creating session-unique temp files
```

---

## Conditional Skills (Path-Triggered)

Skills with `paths` frontmatter are **dormant at startup** and activate automatically when the model operates on matching files:

```markdown
---
description: "TypeScript-specific review checklist"
paths: |
  src/**.ts
  src/**.tsx
  !src/**/*.test.ts
---
## TypeScript Review

Check for:
- Proper type annotations
- No `any` casts without comment
...
```

**How it works** (from `loadSkillsDir.ts`):
1. On startup, skills with `paths` are stored in `conditionalSkills` map instead of being loaded
2. After each `Read`, `Write`, or `Edit` tool call, `activateConditionalSkillsForPaths()` checks if any touched file matches
3. On match, skill moves from `conditionalSkills` → `dynamicSkills` (immediately available to model)
4. Uses `ignore` library (same as `.gitignore`) for pattern matching
5. Once activated, stays activated for the session (even after cache clears)

Pattern syntax follows gitignore rules:
- `src/**.ts` — all `.ts` files under `src/`
- `!src/**/*.test.ts` — exclude test files
- `**/config.json` — any `config.json` anywhere

---

## Dynamic Skill Discovery (Sub-Directory Skills)

When the model touches a file, Claude Code walks up from that file's directory looking for `.claude/skills/` folders that weren't loaded at startup. This enables **skills scoped to subdirectories**:

```
my-project/
  src/
    api/
      .claude/skills/
        endpoint-review/
          SKILL.md    ← Only activated when model touches files in src/api/
    frontend/
      .claude/skills/
        component-review/
          SKILL.md    ← Only activated when model touches files in src/frontend/
  .claude/skills/
    project-review/
      SKILL.md        ← Always available (loaded at startup)
```

**Discovery rules** (from `loadSkillsDir.ts:discoverSkillDirsForPaths`):
- Starts from the touched file's directory, walks up to `cwd` (not including `cwd` itself)
- Skills at `cwd` level are already loaded; only sub-directory skills are discovered dynamically
- Gitignored directories are skipped (security: prevents `node_modules` skills from loading)
- Deepest directories take precedence over shallower ones

---

## Execution Flow When a Skill is Invoked

```
User types /my-skill arg1 arg2
  → parseSlashCommand() → { commandName: 'my-skill', args: 'arg1 arg2' }
  → findCommand('my-skill', commands)
  → getPromptForCommand(args, toolUseContext):
      1. Prepend baseDir if skill has a directory
      2. Substitute $1, $2, ${CLAUDE_SKILL_DIR}, ${CLAUDE_SESSION_ID}
      3. Execute !`shell` injection blocks (if not MCP-sourced)
      4. Return ContentBlockParam[]
  → Content injected as user turn in conversation
  → Model sees skill content and responds
```

For `context: fork` skills:
```
  → SkillTool spawns forkSubagent() with skill content as initial prompt
  → Sub-agent runs in isolated context with parent's tools
  → Result returned to parent agent as tool result
```

---

## Bundled Skills (Shipped with CLI)

Claude Code ships with several built-in skills registered at startup via `registerBundledSkill()`. These demonstrate the programmatic API:

```typescript
import { registerBundledSkill } from '../bundledSkills.js'

registerBundledSkill({
  name: 'simplify',
  description: 'Review changed code for reuse, quality, and efficiency',
  userInvocable: true,
  async getPromptForCommand(args) {
    let prompt = SIMPLIFY_PROMPT
    if (args) prompt += `\n\n## Additional Focus\n\n${args}`
    return [{ type: 'text', text: prompt }]
  },
})
```

You cannot define bundled skills externally — they require source code changes. Use file-based skills or plugin skills for external extensions.

---

## Skill Loading Priority

When skills with the same name exist in multiple locations, the **last writer wins** per source group. Source groups load in this order (later groups override earlier):

1. `policySettings` (managed/enterprise)
2. `userSettings` (`~/.claude/skills/`)
3. `projectSettings` (`.claude/skills/` dirs from cwd up to `$HOME`)
4. Additional dirs (`--add-dir` flag)
5. Legacy commands dirs

Within a group, deeper `.claude/skills/` directories take precedence.

Duplicate detection uses `realpath()` (resolves symlinks) so two paths to the same file don't load twice.

---

## Testing Skills

```bash
# Verify skill is discovered
claude --debug 2>&1 | grep "Loaded.*skills"

# Test skill output without sending to model
# (Use disable-model-invocation: true in SKILL.md temporarily)

# Check conditional skill activation
claude --debug 2>&1 | grep "Activated conditional skill"

# Check dynamic discovery
claude --debug 2>&1 | grep "Dynamically discovered"
```

Environment variable to disable managed skills:
```bash
CLAUDE_CODE_DISABLE_POLICY_SKILLS=1 claude
```

Environment variable to disable project settings:
```bash
# Not directly supported; use --bare flag
claude --bare  # Skips user/project skill discovery
```
