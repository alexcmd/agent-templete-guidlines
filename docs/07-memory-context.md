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
- Everything before the boundary uses `cache_control: { type: 'ephemeral' }` with `scope: 'global'` — can be shared across users/organizations (for Anthropic-controlled prompts)
- Everything after uses per-session caching

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
