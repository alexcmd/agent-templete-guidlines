# Architecture Diagrams

## 1. High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CLAUDE CODE SYSTEM                            │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    USER INTERFACE LAYER                       │    │
│  │                                                               │    │
│  │   ┌──────────────┐   ┌──────────────┐   ┌───────────────┐  │    │
│  │   │  Terminal     │   │  Web Browser  │   │  IDE Plugin   │  │    │
│  │   │  (Ink/React) │   │  (claude.ai) │   │ (VSCode/JB)  │  │    │
│  │   └──────┬───────┘   └──────┬───────┘   └───────┬───────┘  │    │
│  └──────────┼─────────────────┼─────────────────────┼───────────┘    │
│             │                  │                      │                │
│  ┌──────────▼──────────────────▼──────────────────────▼───────────┐  │
│  │                     SESSION LAYER                                │  │
│  │                                                                  │  │
│  │   ┌────────────────────────────────────────────────────────┐   │  │
│  │   │                  QueryEngine                             │   │  │
│  │   │  mutableMessages[] | AbortController | totalUsage       │   │  │
│  │   │  submitMessage() → normalizes input → calls query()     │   │  │
│  │   └────────────────────────┬───────────────────────────────┘   │  │
│  └───────────────────────────┼─────────────────────────────────────┘  │
│                               │                                        │
│  ┌────────────────────────────▼──────────────────────────────────┐    │
│  │                    QUERY LOOP (query.ts)                        │    │
│  │                                                                  │    │
│  │  [Pre-hooks] → [API Call] → [Stream] → [Tool Exec] → [Post]   │    │
│  │       ↑                                     │                   │    │
│  │       └─────────────────────────────────────┘                   │    │
│  │                   (loop until end_turn)                          │    │
│  └────────┬────────────────────────────────────────────────────────┘    │
│           │                                                              │
│  ┌────────▼────────────────────────────────────────────────────────┐    │
│  │                      TOOL LAYER                                   │    │
│  │                                                                   │    │
│  │  ┌────────┐ ┌────────┐ ┌─────────┐ ┌────────┐ ┌────────────┐  │    │
│  │  │  Bash  │ │  File  │ │  Agent  │ │  Web   │ │  MCP Proxy │  │    │
│  │  │  Tool  │ │  Tools │ │  Tool   │ │  Tools │ │  Tools     │  │    │
│  │  └────────┘ └────────┘ └────┬────┘ └────────┘ └─────┬──────┘  │    │
│  └─────────────────────────────┼──────────────────────────┼────────┘    │
│                                 │                           │             │
│  ┌──────────────────────────────▼───────────────────────────▼──────┐    │
│  │                  EXTERNAL SERVICES                                │    │
│  │                                                                   │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │    │
│  │  │ Anthropic   │  │  MCP Servers │  │  File System / Shell    │ │    │
│  │  │ Claude API  │  │  (external) │  │  (local machine)        │ │    │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────┘ │    │
│  └───────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Request Sequence Diagram

```
User          REPL.tsx       QueryEngine      query.ts        Claude API      Tools
 │               │                │               │                │             │
 │ type message  │                │               │                │             │
 ├──────────────►│                │               │                │             │
 │               │ submitMessage()│               │                │             │
 │               ├───────────────►│               │                │             │
 │               │                │ processInput()│                │             │
 │               │                │ fetchSysPrompt│               │             │
 │               │                │───────────────►               │             │
 │               │                │                │               │             │
 │               │                │                │ stream(params)│             │
 │               │                │                ├──────────────►│             │
 │               │                │                │               │             │
 │ [streaming]   │ text_delta     │ stream_event   │◄─ text delta ─┤             │
 │◄──────────────┤◄───────────────┤◄───────────────┤               │             │
 │               │                │                │               │             │
 │               │                │                │◄─ tool_use ───┤             │
 │               │                │  tool_use_block│               │             │
 │               │◄───────────────┤◄───────────────┤               │             │
 │               │                │                │               │             │
 │ [perm dialog] │                │                │               │             │
 │◄──────────────┤                │               │                │             │
 │ Allow         │                │               │                │             │
 ├──────────────►│ canUseTool()   │               │                │             │
 │               │                │               │                │             │
 │               │                │               │ tool.handler() │             │
 │               │                │               ├────────────────────────────►│
 │               │                │               │                │             │
 │ [progress]    │  progress      │ progress_event│◄─── progress ──────────────┤
 │◄──────────────┤◄───────────────┤◄──────────────┤                │             │
 │               │                │               │                │             │
 │               │                │               │◄─── result ────────────────┤
 │               │                │               │                │             │
 │               │                │               │ stream(params  │             │
 │               │                │               │ + tool_result) │             │
 │               │                │               ├──────────────►│             │
 │               │                │               │◄── end_turn ──┤             │
 │               │                │               │               │             │
 │ final message │ message        │ SDKMessage    │               │             │
 │◄──────────────┤◄───────────────┤◄──────────────┤               │             │
```

---

## 3. Permission System Decision Tree

```
                    Tool Call Arrives
                           │
                           ▼
                   ┌───────────────┐
                   │ Mode=bypass?   │──── YES ────► ALLOW
                   └───────┬───────┘
                           │ NO
                           ▼
                   ┌───────────────────┐
                   │ Matches denyRules? │──── YES ────► DENY
                   └───────┬───────────┘
                           │ NO
                           ▼
                   ┌────────────────────┐
                   │ Matches allowRules? │──── YES ────► ALLOW
                   └───────┬────────────┘
                           │ NO
                           ▼
                   ┌────────────────────┐
                   │ Matches askRules?  │──── YES ────► ASK
                   └───────┬────────────┘
                           │ NO
                           ▼
                   ┌─────────────────────┐
                   │ Tool category=read? │──── YES ────► ALLOW
                   └───────┬─────────────┘
                           │ NO
                           ▼
                   ┌─────────────────────────┐
                   │ Mode=plan?               │──── YES ────► DENY (non-read)
                   └───────┬─────────────────┘
                           │ NO
                           ▼
                   ┌─────────────────────────┐
                   │ Tool-specific check      │
                   │ (bash safety parse,      │
                   │  path constraints, etc.) │
                   └───────┬─────────────────┘
                           │
                       ┌───┴───┐
                       │       │
                      ALLOW   ASK
```

---

## 4. Memory System Flow

```
Startup
  │
  ▼
findMemoryFilePaths(cwd)
  │
  ├── Walk up directory tree
  │   ├── /home/user/projects/myapp/.claude/CLAUDE.md
  │   ├── /home/user/projects/.claude/CLAUDE.md  [if exists]
  │   └── /home/user/.claude/CLAUDE.md           [global]
  │
  ▼
loadMemoryPrompt(paths)
  │
  ├── Read each CLAUDE.md
  ├── Parse markdown content
  └── Return MemoryPrompt[]
  │
  ▼
fetchSystemPromptParts()
  │
  ├── [STATIC, globally cached]
  │   ├── Core instructions
  │   ├── Tool descriptions
  │   └── ─── DYNAMIC BOUNDARY ─────────────
  │
  └── [DYNAMIC, per-session]
      ├── Language preference
      ├── Output style
      ├── <memory path="...">content</memory>  ← injected here
      ├── Custom system prompt
      ├── Environment context (git, cwd, OS)
      └── Append prompt
  │
  ▼
Claude API call with assembled system prompt
```

---

## 5. Multi-Agent Orchestration

```
                          Main Agent
                              │
                              │ AgentTool(name="worker-a", background=true)
                              │ AgentTool(name="worker-b", background=true)
                              │ AgentTool(name="worker-c", background=true)
                              │
              ┌───────────────┼────────────────┐
              ▼               ▼                ▼
         Worker A         Worker B          Worker C
       (own QueryEngine) (own QueryEngine) (own QueryEngine)
              │               │                │
              │ [runs]        │ [runs]          │ [runs]
              │               │                │
              ▼               ▼                ▼
      [task complete]  [task complete]  [task complete]
              │               │                │
              └───────────────┼────────────────┘
                              │
                              ▼ (notifications via TaskSystem)
                          Main Agent
                              │
                        TaskList() → all complete
                              │
                        synthesize results
                              │
                         report to user
```

---

## 6. Plugin Architecture

```
Plugin Manifest (manifest.json)
         │
         ▼
┌────────────────────────────────────────────────────────┐
│                    Plugin Components                     │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │    Skills    │  │     Hooks    │  │  MCP Servers │ │
│  │  (markdown   │  │   (shell     │  │  (separate   │ │
│  │   prompts)   │  │  commands)   │  │  processes)  │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘ │
└─────────┼─────────────────┼─────────────────┼──────────┘
          │                  │                  │
          ▼                  ▼                  ▼
   Slash commands      Event callbacks      New tool types
   /plugin:skill       PreToolUse           mcp__server__tool
                       PostToolUse
                       Stop
```

---

## 7. Configuration Hierarchy

```
┌─────────────────────────────────────────────────────┐
│                   EFFECTIVE CONFIG                   │
│           (all sources merged, later wins)           │
├─────────────────────────────────────────────────────┤
│  CLI Flags (--model, --verbose, ...)       [9: HIGH] │
├─────────────────────────────────────────────────────┤
│  Environment Variables (ANTHROPIC_*, etc.) [8]       │
├─────────────────────────────────────────────────────┤
│  MDM Policy (enterprise device mgmt)       [7]       │
├─────────────────────────────────────────────────────┤
│  Managed Settings (sync.claude.ai)         [6]       │
├─────────────────────────────────────────────────────┤
│  .claude/settings.local.json (gitignored)  [5]       │
├─────────────────────────────────────────────────────┤
│  .claude/settings.json (project)           [4]       │
├─────────────────────────────────────────────────────┤
│  ~/.claude/settings.json (global user)     [3]       │
├─────────────────────────────────────────────────────┤
│  Default values (TypeScript code)          [2: LOW]  │
└─────────────────────────────────────────────────────┘
```

---

## 8. Context Window Management

```
Context window (e.g., 200k tokens)

[████████████████████░░░░░░░░░░░░░░░] 50% — normal operation
[██████████████████████████████░░░░░] 85% — compaction triggered
[█████████████████████████████████░░] 95% — warning shown to user
[████████████████████████████████████] 100% — hard stop

Compaction process:
  Old messages           Summary              Recent messages
  [msg1][msg2][msg3] → [▶ Summary: ...] + [msg8][msg9][msg10]
  
  Result: ~40% of original token count, key context preserved
```

---

## 9. Tool Output Flow

```
Tool executes
     │
     ▼
output = tool.handler(input, context)
     │
     ├─ If output < INLINE_THRESHOLD (8KB):
     │       embed in ToolResultBlockParam directly
     │
     └─ If output > INLINE_THRESHOLD:
             write to ~/.claude/tmp/<session>/tool-results/<id>
             return reference with preview
             │
             ▼
     Model receives: 
       "[Large output stored. Preview: first 2KB...]
        Full output available at path: /tmp/..."
```

---

## 10. Component Dependency Map

```
main.tsx
    └── entrypoints/init.ts
            └── entrypoints/cli.tsx
                    └── screens/REPL.tsx
                            ├── QueryEngine.ts
                            │       ├── query.ts
                            │       │       ├── services/api/claude.ts
                            │       │       ├── services/compact/
                            │       │       └── tools/* (via tools.ts)
                            │       ├── utils/queryContext.ts
                            │       │       └── constants/prompts.ts
                            │       ├── memdir/memdir.ts
                            │       └── utils/processUserInput/
                            │
                            ├── hooks/useCanUseTool.ts
                            │       └── utils/permissions/
                            │
                            ├── state/AppStateStore.ts
                            │       └── state/AppState.tsx
                            │
                            └── services/mcp/
                                    └── (MCP SDK client)
```
