# Memory & Context System

## Overview

Claude Code uses two orthogonal systems to inject long-term context into conversations:

1. **Memory System** — loads `.claude/` markdown files into the system prompt
2. **Context System** — assembles dynamic session state (git, env, tools) into system prompt sections

---

## Memory Directory Structure

```
~/.claude/
├── CLAUDE.md              Global user memory (always loaded)
├── memory/                Memory storage directory
│   ├── user_role.md
│   ├── feedback_testing.md
│   └── project_foo.md
└── settings.json

<project>/
├── .claude/
│   ├── CLAUDE.md          Project memory (loaded when in project)
│   ├── commands/          User-defined slash commands
│   ├── agents/            Custom agent definitions
│   └── settings.json      Project settings
```

### Memory File Loading Order

```
1. ~/.claude/CLAUDE.md (global)
2. <project>/.claude/CLAUDE.md (project-level)
3. <subdir>/.claude/CLAUDE.md (directory-level, if in subdirectory)
4. Any .claude/CLAUDE.md found walking up the directory tree
```

---

## Memory System (`src/memdir/`)

### Memory Loading

```typescript
// memdir.ts
export async function loadMemoryPrompt(options: {
  cwd: string
  settings: SettingsJson
  loadedPaths: Set<string>
}): Promise<MemoryPrompt[]> {
  const paths = findMemoryFilePaths(options.cwd)
  
  const memories: MemoryPrompt[] = []
  for (const path of paths) {
    if (options.loadedPaths.has(path)) continue
    
    const content = await readFile(path, 'utf-8')
    const parsed = parseMemoryFile(content)
    memories.push(parsed)
    options.loadedPaths.add(path)
  }
  
  return memories
}
```

### Memory Injection

Memories are injected into the system prompt as attachment messages:

```typescript
// attachments.ts
function getAttachmentMessages(memories: MemoryPrompt[]): AttachmentMessage[] {
  return memories.map(memory => ({
    type: 'attachment',
    content: [{
      type: 'text',
      text: `<memory path="${memory.path}">\n${memory.content}\n</memory>`,
    }],
    uuid: generateUUID(),
  }))
}
```

### CLAUDE.md Format

Claude Code reads CLAUDE.md files as plain markdown:

```markdown
# Project Context

This is a Node.js backend service for processing payments.

## Architecture
- Express.js REST API
- PostgreSQL via Prisma ORM  
- Redis for caching
- Jest for testing

## Key Conventions
- All API handlers are in src/handlers/
- Use async/await, not callbacks
- All errors should be logged with context

## Important Notes
- Never commit .env files
- Run `npm test` before committing
- The payments module requires PCI compliance review
```

---

## Session Memory Service (`src/services/SessionMemory/`)

A separate service that manages structured memory extraction and storage during sessions:

```typescript
// Extracts key facts from conversations and saves them to memory files
type ExtractedMemory = {
  type: 'user' | 'feedback' | 'project' | 'reference'
  name: string
  description: string
  content: string
  filePath: string    // Where to save it
}
```

### Auto-Memory Features
When enabled, Claude Code can:
- Extract important information from conversations
- Save to structured memory files in `~/.claude/projects/<project-hash>/memory/`
- Load relevant memories at conversation start

---

## System Prompt Assembly (`src/utils/queryContext.ts`)

The system prompt is assembled fresh for each query from multiple sources:

```typescript
export async function fetchSystemPromptParts(options: {
  tools: Tools
  commands: Command[]
  settings: SettingsJson
  mcpClients: MCPServerConnection[]
  memories: MemoryPrompt[]
  cwd: string
  isNonInteractive: boolean
  customSystemPrompt?: string
  appendSystemPrompt?: string
  outputStyleConfig?: OutputStyleConfig
  coordinatorMode?: boolean
}): Promise<SystemPrompt> {
  const sections = await resolveSystemPromptSections({
    // 1. Core instructions (huge static section)
    mainPrompt: getSystemPrompt(options),
    
    // 2. Tool documentation (dynamic, based on enabled tools)
    toolDocs: getToolDescriptions(options.tools),
    
    // 3. Memory files (from .claude/)
    memories: options.memories,
    
    // 4. Custom/append sections
    customPrompt: options.customSystemPrompt,
    appendPrompt: options.appendSystemPrompt,
    
    // 5. Output style
    outputStyle: options.outputStyleConfig,
    
    // 6. Feature-specific sections
    coordinatorSection: options.coordinatorMode ? getCoordinatorSystemPrompt() : null,
  })
  
  return asSystemPrompt(sections)
}
```

---

## System Prompt Structure (`src/constants/prompts.ts`)

The main system prompt is built from these sections in order:

```
[STATIC — globally cacheable] ─────────────────────────────────
1. Role declaration ("You are Claude Code, Anthropic's official CLI...")
2. Core behavioral rules
   - Task approach
   - Code quality standards
   - Security guidelines
   - Comment guidelines
   - Tool usage rules
3. Available tool descriptions
4. Skill/command documentation
5. __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__  ← Cache split point

[DYNAMIC — per-session] ────────────────────────────────────────
6. Current model identity
7. Language preference (if set)
8. Output style (if configured)
9. Memory file contents
10. Hooks documentation (if hooks configured)
11. Custom system prompt (user-provided prefix)
12. Environment context (git, platform, working directory)
13. Coordinator section (if coordinator mode)
14. Append prompt (user-provided suffix)
```

### Cache Optimization

Claude Code splits the system prompt at `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`:
- Everything before the boundary uses `cache_control: { type: 'ephemeral', scope: 'global' }` — shared across all sessions and users (Anthropic-controlled prompts only)
- Everything after uses per-session caching (`scope` omitted)

---

## Prompt Cache System

All details sourced from `src/services/api/claude.ts`, `src/utils/api.ts`, and `src/bootstrap/state.ts`.

### `getCacheControl()` — `services/api/claude.ts:358`

Every API message block gets a `cache_control` directive built by this function:

```typescript
export function getCacheControl({ scope, querySource } = {}): {
  type: 'ephemeral'
  ttl?: '1h'
  scope?: CacheScope
} {
  return {
    type: 'ephemeral',
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),
    ...(scope === 'global' && { scope }),
  }
}
```

**TTL options:**
- Default: `5m` (omitted, API default)
- Extended: `1h` — eligible users only (ANT employees, Claude AI subscribers not in overage, Bedrock with `ENABLE_PROMPT_CACHING_1H_BEDROCK`, GrowthBook gate `tengu_prompt_cache_1h_config`)

**Eligibility is latched** at session start in `bootstrap/state.ts:225` (`promptCache1hEligible`). This prevents a mid-session eligibility change (e.g., entering overage) from switching TTL and busting the cache.

**Cache scopes:**

| Scope | Meaning |
|-------|---------|
| `'global'` | Shared across all sessions and users (static system prompt blocks) |
| `'org'` | Organization-level sharing |
| `null` (omitted) | Per-session only |

### System Prompt Split Caching — `utils/api.ts:321`

`splitSysPromptPrefix()` divides the system prompt at `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`:

```
[Before boundary]  → cache_control { type:'ephemeral', scope:'global' }
[After boundary]   → cache_control { type:'ephemeral' }  (no scope)
```

The strategy for global caching (`tool_based` vs `system_prompt`) is selected by `shouldUseGlobalCacheScope()`. When MCP tools are present, the strategy switches to `tool_based` (tools are the stable prefix) rather than `system_prompt`.

### Cache Editing (`cache_edits`) — `services/api/claude.ts:3050`

When the context window fills up, stale cached content can be deleted without invalidating the rest of the cache prefix:

```typescript
type CacheEditsBlock = {
  type: 'cache_edits'
  edits: { type: 'delete'; cache_reference: string }[]
}
```

**How it works:**
1. `cache_reference` fields are added to `tool_result` blocks inside the cached prefix
2. A `cache_edits` block with `{ type: 'delete', cache_reference }` entries is inserted into the last user message
3. Deduplication prevents the same reference being deleted twice across multiple `cache_edits` blocks
4. Previously-pinned `cache_edits` are re-inserted at their original positions in the message array
5. **Constraint:** a `cache_reference` must appear _before_ the last `cache_control` marker in message order

The `cacheEditingHeaderLatched` flag in `bootstrap/state.ts:237` ensures the beta header is only set once — toggling it mid-session would bust the cache.

### Cache Break Detection — `services/api/promptCacheBreakDetection.ts`

Tracks a hash of the full cache-relevant API state on each request and logs when a change would bust the cache. Monitored fields:

```typescript
type PreviousState = {
  systemHash: number          // System prompt content
  cacheControlHash: number    // scope/TTL values
  toolsHash: number           // Tool schema set
  perToolHashes: Record<string, number>
  model: string
  fastMode: boolean
  globalCacheStrategy: string // 'tool_based' | 'system_prompt'
  betas: string[]             // Beta headers
  autoModeActive: boolean
  isUsingOverage: boolean     // Latched — won't flip mid-session
  cachedMCEnabled: boolean
  effortValue: string
  extraBodyHash: number
  callCount: number
  cacheDeletionsPending: boolean
}
```

### Bootstrap State — Session Cache Latches

`bootstrap/state.ts` holds session-stable cache variables that are **set once and never changed** to avoid mid-session cache busts:

| State field | Line | Purpose |
|-------------|------|---------|
| `promptCache1hEligible` | 225 | 1h TTL eligibility — latched on first eval |
| `promptCache1hAllowlist` | 221 | GrowthBook allowlist — cached for session |
| `afkModeHeaderLatched` | 229 | Sticky-on: AFK_MODE_BETA_HEADER |
| `fastModeHeaderLatched` | 233 | Sticky-on: FAST_MODE_BETA_HEADER |
| `cacheEditingHeaderLatched` | 237 | Sticky-on: cache-editing beta header |
| `thinkingClearLatched` | 242 | Sticky-on: thinking cache clear (>1h idle) |
| `systemPromptSectionCache` | 203 | Map of section name → cached content |
| `lastMainRequestId` | 248 | Last API requestId for cache eviction hints |
| `lastApiCompletionTimestamp` | 252 | Correlates cache misses with idle time |

**Latch pattern** — flags that are `false` by default and flip to `true` permanently when a feature is first used:

```typescript
// If fast mode is turned on mid-session, the beta header stays on
// for the rest of the session (preventing a cache bust on toggle-off)
if (isFastMode && !getFastModeHeaderLatched()) {
  setFastModeHeaderLatched(true)
}
const includeFastHeader = isFastMode || getFastModeHeaderLatched()
```

### Session Memory Service — `services/SessionMemory/sessionMemory.ts`

Auto-extracts key facts from the conversation and saves them to structured memory files. Gated by GrowthBook feature flag `tengu_session_memory`.

- **Config source:** `tengu_sm_config` from GrowthBook (cached, may be stale across sessions)
- **Thresholds:** init threshold (context window tokens), update threshold (tokens since last extraction), tool-call count between updates
- **Extraction:** runs as a background forked subagent — does not block the main conversation turn

### Token Accounting

`bootstrap/state.ts:67` tracks cache token usage in `modelUsage`:

```typescript
type ModelUsage = {
  inputTokens: number
  outputTokens: number
  cacheReadInputTokens: number      // Tokens served from cache
  cacheCreationInputTokens: number  // Tokens written to cache
}
```

Surfaced in `/cost` output.

---

## User Context (`src/context.ts`)

Dynamic context injected into the conversation as a user message:

```typescript
export async function getSystemContext(options): Promise<{ [key: string]: string }> {
  return {
    // Environment
    platform: `${osType()} ${osRelease()}`,
    shell: getShell(),
    cwd: getCwd(),
    
    // Git state
    isGit: String(await getIsGit()),
    gitBranch: await getGitBranch(),
    gitDirtyFiles: await getGitDirtyFiles(),
    
    // Session
    sessionStartDate: getSessionStartDate(),
    
    // Worktree (if active)
    worktreeSession: getCurrentWorktreeSession(),
  }
}

export async function getUserContext(options): Promise<{ [key: string]: string }> {
  return {
    // Model info
    mainLoopModel: getCanonicalName(options.mainLoopModel),
    marketingModelName: getMarketingNameForModel(options.mainLoopModel),
    
    // Coordinator info
    coordinatorContext: options.isCoordinator 
      ? getCoordinatorUserContext(options.mcpClients) 
      : undefined,
    
    // Teammate context (if sub-agent)
    teammateContext: isTeammate() ? getTeammateContext() : undefined,
    
    // Memory
    hasMemories: String(options.memories.length > 0),
  }
}
```

---

## Context Injection Points

The assembled context is injected at specific positions in the conversation:

```typescript
// query.ts
// 1. Prepend user context (added as first message)
const messages = prependUserContext(messages, userContext)

// 2. Append system context (added to system prompt)
const systemPrompt = appendSystemContext(systemPrompt, systemContext)
```

---

## Memory Prefetching

For performance, relevant memories are prefetched before the API call:

```typescript
// Start prefetch in background (non-blocking)
const prefetchPromise = startRelevantMemoryPrefetch({
  messages: currentMessages,
  cwd: getCwd(),
})

// ... API call starts while prefetch runs ...

// By the time we need memories, they may already be loaded
const relevantMemories = await prefetchPromise
```

---

## CLAUDE.md Hierarchy and Nesting

When a user navigates to a subdirectory, Claude Code loads ALL CLAUDE.md files from:
1. The filesystem root
2. Each directory going down to current working directory
3. Global `~/.claude/CLAUDE.md`

This creates a nested memory hierarchy where:
- Global memories apply everywhere
- Project memories apply within the project
- Subdirectory memories apply to specific module contexts

```typescript
// memdir.ts
function findMemoryFilePaths(cwd: string): string[] {
  const paths: string[] = []
  
  // Walk up from cwd to filesystem root
  let dir = cwd
  while (dir !== path.dirname(dir)) {
    const claudeMd = path.join(dir, '.claude', 'CLAUDE.md')
    if (existsSync(claudeMd)) {
      paths.unshift(claudeMd)  // Prepend (ancestor first)
    }
    dir = path.dirname(dir)
  }
  
  // Add global memory
  const globalMd = path.join(homedir(), '.claude', 'CLAUDE.md')
  if (existsSync(globalMd)) {
    paths.unshift(globalMd)
  }
  
  return paths
}
```
