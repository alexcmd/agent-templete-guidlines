# Tools System

## Overview

Tools are the primary way Claude interacts with the world. Each tool is a TypeScript module with:
1. A **name** (how Claude calls it)
2. A **JSON Schema** defining valid inputs
3. A **handler** that executes the action
4. Optional **permission check**, **UI renderer**, and **progress streaming**

Tools are registered in `src/tools.ts` and assembled into the `toolPool` passed to each query.

---

## Complete Tool Inventory

### File & Code Tools

| Tool | Name | Description |
|------|------|-------------|
| `FileReadTool` | `Read` | Read file contents by path, with line range support |
| `FileEditTool` | `Edit` | Exact string replacement within a file (requires prior Read) |
| `FileWriteTool` | `Write` | Write/overwrite entire file contents |
| `GlobTool` | `Glob` | Find files by glob pattern |
| `GrepTool` | `Grep` | Search file contents with regex (ripgrep-backed) |
| `NotebookEditTool` | `NotebookEdit` | Edit Jupyter notebook cells |
| `LSPTool` | `LSP` | Query language servers (go-to-definition, hover, etc.) |

### Shell Tools

| Tool | Name | Description |
|------|------|-------------|
| `BashTool` | `Bash` | Execute shell commands with timeout and output capture |
| `PowerShellTool` | `PowerShell` | Execute PowerShell commands (Windows) |
| `REPLTool` | `REPL` | Persistent REPL session (Python/JS/etc.) — ant-only |

### Web Tools

| Tool | Name | Description |
|------|------|-------------|
| `WebFetchTool` | `WebFetch` | Fetch URL content (HTML, JSON, text) |
| `WebSearchTool` | `WebSearch` | Search the web via Brave/other search APIs |

### Agent & Orchestration Tools

| Tool | Name | Description |
|------|------|-------------|
| `AgentTool` | `Agent` | Spawn a sub-agent (isolated QueryEngine instance) |
| `SkillTool` | `Skill` | Invoke a registered skill/slash command |
| `SendMessageTool` | `SendMessage` | Send message to a named running sub-agent |

### Task Management Tools

| Tool | Name | Description |
|------|------|-------------|
| `TaskCreateTool` | `TaskCreate` | Create a new tracked task |
| `TaskUpdateTool` | `TaskUpdate` | Update task status/content |
| `TaskGetTool` | `TaskGet` | Get task details by ID |
| `TaskListTool` | `TaskList` | List all current tasks |
| `TaskOutputTool` | `TaskOutput` | Get output from a running task |
| `TaskStopTool` | `TaskStop` | Stop/cancel a running task |

### Mode Tools

| Tool | Name | Description |
|------|------|-------------|
| `EnterPlanModeTool` | `EnterPlanMode` | Switch to plan mode (requires explicit approval) |
| `ExitPlanModeTool` | `ExitPlanMode` | Exit plan mode |
| `EnterWorktreeTool` | `EnterWorktree` | Create/enter an isolated git worktree |
| `ExitWorktreeTool` | `ExitWorktree` | Exit and optionally clean up worktree |

### MCP Tools

| Tool | Name | Description |
|------|------|-------------|
| `MCPTool` | `mcp__<server>__<tool>` | Proxy to MCP server tool |
| `ListMcpResourcesTool` | `ListMcpResources` | List available MCP server resources |
| `ReadMcpResourceTool` | `ReadMcpResource` | Read an MCP server resource |
| `McpAuthTool` | `McpAuth` | Authenticate with MCP servers requiring OAuth |
| `ToolSearchTool` | `ToolSearch` | Search deferred tool schemas by name/keywords |

### Interaction Tools

| Tool | Name | Description |
|------|------|-------------|
| `AskUserQuestionTool` | `AskUserQuestion` | Prompt user for a response (blocks until answered) |
| `TodoWriteTool` | `TodoWrite` | Write/update the todo list |
| `ConfigTool` | `Config` | Read/write Claude Code configuration |

### Scheduling Tools (feature-gated)

| Tool | Name | Description |
|------|------|-------------|
| `ScheduleCronTool` (CronCreate) | `CronCreate` | Create a recurring scheduled task |
| `CronDeleteTool` | `CronDelete` | Delete a scheduled task |
| `CronListTool` | `CronList` | List scheduled tasks |
| `RemoteTriggerTool` | `RemoteTrigger` | Trigger a remote agent |
| `SleepTool` | `Sleep` | Pause execution for N seconds (proactive/Kairos mode) |
| `PushNotificationTool` | `PushNotification` | Send push notification (Kairos mode) |

### Proactive / Kairos Tools (feature-gated)

| Tool | Name | Description |
|------|------|-------------|
| `BriefTool` | `Brief` | Generate brief status (Kairos mode) |
| `SyntheticOutputTool` | (internal) | Inject synthetic output into conversation |

### Team Tools (feature-gated)

| Tool | Name | Description |
|------|------|-------------|
| `TeamCreateTool` | `TeamCreate` | Create a new agent team |
| `TeamDeleteTool` | `TeamDelete` | Delete an agent team |

---

## Tool Implementation Pattern

Every tool follows this structure:

```
src/tools/MyTool/
├── MyTool.tsx          Main tool implementation (handler, schema, UI)
├── UI.tsx              React component for rendering in terminal
├── prompt.ts           Tool description string / constants
├── constants.ts        Tool name constant and other constants
├── permissions.ts      Permission checking logic (if tool-specific)
└── MyTool.test.ts      Unit tests
```

### Minimal Tool Example

```typescript
// src/tools/MyTool/MyTool.tsx
import { buildTool, type ToolDef } from '../../Tool.js'
import { z } from 'zod/v4'

const TOOL_NAME = 'MyTool'

const toolDef: ToolDef<typeof inputSchema> = {
  name: TOOL_NAME,
  description: `Description of what this tool does, when to use it, and
important constraints. This is shown to the model in the system prompt.`,
  inputSchema: z.object({
    param: z.string().describe('What this parameter does'),
    optional_param: z.number().optional().describe('Optional param'),
  }),
  handler: async (input, context) => {
    const { param, optional_param } = input
    // Do the work
    const result = await doSomething(param)
    return `Result: ${result}`
  },
  isEnabled: (context) => {
    // Return false to hide tool from model (e.g., based on settings)
    return context.getAppState().settings.myFeatureEnabled === true
  },
}

export const MyTool = buildTool(toolDef)
```

### Streaming Tool Example

```typescript
const toolDef: ToolDef<typeof inputSchema> = {
  name: TOOL_NAME,
  description: '...',
  inputSchema: z.object({ command: z.string() }),
  supportsStreaming: true,
  
  async *handler(input, context): AsyncGenerator<ToolCallProgress> {
    // Yield progress events as work proceeds
    yield {
      type: 'progress',
      message: 'Starting...',
      status: 'running',
    }
    
    for await (const chunk of streamingOperation(input.command)) {
      yield {
        type: 'progress', 
        message: chunk,
        status: 'running',
      }
    }
    
    // Final result is yielded as the last event
    yield {
      type: 'result',
      message: 'Done!',
      status: 'completed',
    }
  },
}
```

---

## BashTool Deep Dive

BashTool is the most complex tool. Key behaviors:

### Permission Checking
```typescript
// bashPermissions.ts
function bashToolHasPermission(
  command: string,
  permissionContext: ToolPermissionContext
): PermissionResult {
  // 1. Check exact command matches in allow rules
  // 2. Check wildcard patterns (e.g., "Bash(git *)")
  // 3. Check read-only constraints
  // 4. Default to 'ask' for unknown commands
}
```

### Command Classification
Commands are classified for UI display:
- **Search commands** (`grep`, `rg`, `find`, `ag`) → collapsed with "Searched N files"
- **Read commands** (`cat`, `head`, `tail`, `jq`) → collapsed with "Read N files"
- **List commands** (`ls`, `tree`, `du`) → collapsed with "Listed N directories"
- **Silent commands** (`mv`, `cp`, `rm`, etc.) → no output expected

### Execution Model
```typescript
// Runs via LocalShellTask for background support
const taskResult = await spawnShellTask({
  command: input.command,
  timeout: input.timeout ?? getDefaultTimeoutMs(),
  cwd: getCwd(),
  abortController: context.abortController,
})
```

### Security Checks
- `parseForSecurity(command)` — AST-based bash parsing to detect dangerous patterns
- Blocks certain constructs in read-only validation mode
- `parseSedEditCommand()` — special handling for sed-based file edits

---

## AgentTool Deep Dive

The AgentTool spawns isolated sub-agents (nested QueryEngine instances).

### Input Schema
```typescript
z.object({
  description: z.string(),            // Short task description (3-5 words)
  prompt: z.string(),                  // Full task prompt
  subagent_type: z.string().optional(), // Built-in or custom agent type
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
  run_in_background: z.boolean().optional(),
  isolation: z.enum(['worktree', 'remote']).optional(),
  cwd: z.string().optional(),
  name: z.string().optional(),         // Named agent for SendMessage
  mode: permissionModeSchema().optional(),
})
```

### Execution Modes
1. **Foreground** (default) — blocks until complete, streams progress
2. **Background** (`run_in_background: true`) — returns immediately, runs async
3. **Worktree isolation** (`isolation: 'worktree'`) — creates git worktree branch
4. **Remote** (`isolation: 'remote'`) — runs in cloud (ant-only)

### Built-in Agent Types
- `general-purpose` — default agent with full tool access
- `explore` — specialized for codebase exploration (limited tools)
- `plan` — planning agent (Plan mode)
- `code-reviewer` — code review specialist
- `Explore`, `Plan` — exported as named types for the main harness

### Custom Agents
Defined in `.claude/agents/<name>.md` with YAML frontmatter:
```yaml
---
name: my-agent
description: Does something specific
model: sonnet
tools: [Read, Grep, Glob]
---
System prompt for the agent goes here...
```

---

## FileEditTool Deep Dive

Performs exact string replacement in files. Designed to be precise and auditable.

### Input Schema
```typescript
z.object({
  file_path: z.string(),      // Absolute path
  old_string: z.string(),     // Exact text to find (must be unique)
  new_string: z.string(),     // Replacement text
  replace_all: z.boolean().optional(), // Replace all occurrences
})
```

### Behavior
1. Reads file (or uses cached `readFileState`)
2. Finds `old_string` in content
3. Fails if `old_string` is not found or not unique (unless `replace_all`)
4. Replaces and writes file
5. Notifies VS Code via MCP if applicable

---

## Tool Assembly (tools.ts)

```typescript
export function assembleToolPool(
  toolUseContext: ToolUseContext,
  options: AssembleOptions,
): Tool[] {
  const basTools = [
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    GlobTool,
    GrepTool,
    BashTool,
    WebFetchTool,
    WebSearchTool,
    AgentTool,
    SkillTool,
    TaskCreateTool,
    TaskUpdateTool,
    TaskGetTool,
    TaskListTool,
    AskUserQuestionTool,
    EnterPlanModeTool,
    ExitPlanModeTool,
    EnterWorktreeTool,
    ExitWorktreeTool,
    TodoWriteTool,
    ConfigTool,
    // ... conditionally added tools
  ]

  // Add MCP tools (dynamic, from connected servers)
  const mcpTools = buildMCPTools(toolUseContext.options.mcpClients)

  // Filter by isEnabled()
  const enabledTools = [...baseTools, ...mcpTools]
    .filter(tool => tool.isEnabled?.(toolUseContext) ?? true)

  return enabledTools
}
```

---

## Permission System for Tools

```
canUseTool(tool, input, context): Promise<PermissionResult>
       │
       ▼
1. Check alwaysDenyRules — if matched, return DENY immediately
       │
       ▼
2. Check alwaysAllowRules — if matched, return ALLOW immediately
       │
       ▼
3. Check alwaysAskRules — if matched, always ASK user
       │
       ▼
4. Check permission mode:
   - 'bypass' → ALLOW all
   - 'plan'   → ASK for execution tools (allow read-only)
   - 'default' → apply heuristics
       │
       ▼
5. Tool-specific logic:
   - BashTool: parse command, check safety rules
   - FileEditTool: check path vs. working directory
   - AgentTool: check agent allow/deny list
   - MCPTool: check server-specific rules
       │
       ▼
6. Return ALLOW | DENY | ASK
```

### Permission Rule Format

Rules stored in settings as patterns:
```json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(npm run *)",
      "Edit",
      "Read"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(sudo *)"
    ]
  }
}
```
