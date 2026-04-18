# Multi-Agent Orchestration

## Overview

Claude Code supports spawning sub-agents that run as independent `QueryEngine` instances. Agents can run foreground (blocking) or background (async), and can communicate via the task messaging system.

---

## Agent Taxonomy

```
Main Agent (Claude Code REPL session)
    │
    ├── AgentTool → spawns sub-agents
    │       │
    │       ├── LocalAgentTask (in-process, isolated context)
    │       │       └── Own QueryEngine instance
    │       │           ├── Own message history
    │       │           ├── Own tool pool
    │       │           └── Own abort controller
    │       │
    │       ├── RemoteAgentTask (cloud execution, ant-only)
    │       │       └── Remote QueryEngine
    │       │
    │       └── WorktreeAgent (git worktree isolated)
    │               └── LocalAgentTask + isolated git branch
    │
    └── TaskSystem tracks all agents
            ├── TaskState per agent
            ├── Progress tracking
            └── Notification queue
```

---

## Coordinator Mode (`src/coordinator/coordinatorMode.ts`)

When `CLAUDE_CODE_COORDINATOR_MODE=1` is set, the main agent receives a special system prompt that instructs it to:

1. Break tasks into parallel sub-tasks
2. Spawn workers via `AgentTool`
3. Monitor progress via `TaskListTool`/`TaskGetTool`
4. Send follow-ups via `SendMessageTool`
5. Stop workers via `TaskStopTool`

### Coordinator System Prompt Additions

```typescript
export function getCoordinatorSystemPrompt(): string {
  return `
You are operating in coordinator mode. You orchestrate a team of worker agents.

## Spawning Workers
Use the Agent tool to spawn workers for parallel tasks:
- Give each worker a clear, focused description
- Use run_in_background: true for parallel execution
- Assign meaningful names via the name parameter

## Monitoring Workers  
- Use TaskList to see all running agents
- Use TaskGet to get details on a specific agent
- Workers automatically notify you when complete

## Communication
- Use SendMessage to send follow-up instructions to named agents
- Agents can be addressed by name: SendMessage({to: "agent-name", message: "..."})

## Best Practices
- Spawn workers for independent parallel tasks
- Don't spawn workers for sequential tasks — do those yourself
- Synthesize worker results before presenting to user
`
}
```

---

## AgentTool Execution Flow

```
AgentTool.handler(input, context)
       │
       ├─ Validate input (schema check)
       ├─ Check permission (filterDeniedAgents)
       ├─ Resolve agent definition (built-in or .claude/agents/)
       │
       ├─ Choose execution mode:
       │   ├─ run_in_background: false (default) → foreground
       │   │   └─ registerAgentForeground()
       │   │
       │   ├─ run_in_background: true → background
       │   │   └─ registerAsyncAgent()
       │   │
       │   └─ isolation: 'worktree' → git worktree
       │       └─ createAgentWorktree() then registerAgentForeground()
       │
       ├─ Build agent config:
       │   ├─ Resolve model (subagent_type → agentDefinition → input.model → inherit)
       │   ├─ Build system prompt (agent definition + base prompt)
       │   ├─ Assemble tool pool (filtered by agent definition)
       │   └─ Set permission mode
       │
       ├─ Create QueryEngine for sub-agent
       ├─ Call runAgent(queryEngine, prompt)
       │
       └─ Return result or task ID (if background)
```

---

## Task System (`src/tasks/`)

### LocalAgentTask (`src/tasks/LocalAgentTask/LocalAgentTask.ts`)

Manages a local in-process agent:

```typescript
type LocalAgentTask = {
  taskId: TaskId
  agentId: AgentId
  queryEngine: QueryEngine
  status: 'running' | 'completed' | 'failed' | 'cancelled'
  progress: string[]
  outputLines: string[]
  notificationQueue: AgentNotification[]
  
  // Methods:
  // sendMessage(message) — inject message into agent conversation
  // stop() — cancel execution
  // getProgress() — current status + progress string
}
```

### RemoteAgentTask (`src/tasks/RemoteAgentTask/RemoteAgentTask.ts`)

For cloud-executed agents (ant/enterprise only):

```typescript
// Register an agent to run remotely
async function registerRemoteAgentTask(params: {
  prompt: string
  agentType?: string
  parentSessionId: SessionId
  model?: string
}): Promise<{ taskId: TaskId; sessionUrl: string }>
```

### LocalShellTask (`src/tasks/LocalShellTask/LocalShellTask.ts`)

For long-running shell commands that can be backgrounded:

```typescript
// Spawn a shell command as a background task
async function spawnShellTask(params: {
  command: string
  timeout: number
  cwd: string
  abortController: AbortController
}): Promise<ShellTask>

// Move a foreground shell command to background
function backgroundExistingForegroundTask(taskId: TaskId): void
```

---

## Agent Progress Tracking

```typescript
// AgentTool emits progress events as the sub-agent runs
type AgentToolProgress = {
  type: 'agent-tool-progress'
  agentId: AgentId
  taskId: TaskId
  status: 'running' | 'completed' | 'failed'
  progressMessage?: string
  outputPreview?: string    // Last N lines of output
  tokenCount?: number
  costUSD?: number
}
```

Progress is displayed in the REPL as a collapsible agent view:

```
▶ Agent [explore-codebase] Running...
  └─ Reading src/components/...
  └─ Found 47 React components
  └─ [2.3k tokens | $0.003]
```

---

## SendMessage Tool (`src/tools/SendMessageTool/`)

Allows the coordinator to communicate with running named agents:

```typescript
// Input schema
z.object({
  to: z.string().describe('Agent name or task ID to send message to'),
  message: z.string().describe('Message to inject into agent conversation'),
})

// Handler
async function handler(input, context) {
  const { to, message } = input
  
  // Find agent by name from registry
  const agentId = context.getAppState().agentNameRegistry.get(to)
  const task = getTaskById(agentId)
  
  if (!task || task.status !== 'running') {
    return `Agent "${to}" not found or not running`
  }
  
  // Inject message into agent's conversation
  await task.sendMessage(message)
  
  return `Message sent to agent "${to}"`
}
```

---

## Worktree Isolation (`src/utils/worktree.ts`)

When `isolation: 'worktree'` is specified:

```typescript
async function createAgentWorktree(
  agentId: AgentId,
  taskDescription: string,
): Promise<{
  worktreePath: string
  branchName: string
}> {
  // 1. Create unique branch name
  const branchName = `agent-${agentId}-${slugify(taskDescription)}`
  
  // 2. Create git worktree
  await exec(`git worktree add ${worktreePath} -b ${branchName}`)
  
  return { worktreePath, branchName }
}

// After agent completes:
async function removeAgentWorktree(worktreePath: string): Promise<void> {
  if (await hasWorktreeChanges(worktreePath)) {
    // Keep the worktree — it has uncommitted changes
    notifyUser(`Agent made changes in worktree: ${worktreePath}`)
  } else {
    // Clean up empty worktree
    await exec(`git worktree remove ${worktreePath}`)
  }
}
```

---

## Agent Color System

Each agent gets a unique color for UI identification:

```typescript
// In bootstrap/state.ts
const agentColorMap = new Map<string, AgentColorName>()
// Colors: 'red' | 'blue' | 'green' | 'yellow' | 'magenta' | 'cyan' | ...

function getAgentColor(agentId: AgentId): AgentColorName {
  if (!agentColorMap.has(agentId)) {
    agentColorMap.set(agentId, pickNextColor())
  }
  return agentColorMap.get(agentId)!
}
```

---

## Agent Context Isolation

Each sub-agent gets its own:
- Message history (starts fresh from the task prompt)
- File state cache (does not share parent's cache)
- Abort controller (can be independently cancelled)
- Working directory (can override with `cwd:` parameter)
- Permission context (can be more restrictive than parent)

Sub-agents do NOT inherit:
- Parent's conversation history
- Parent's session ID (gets new session ID)
- Parent's file state cache

Sub-agents DO inherit (by default):
- Model setting (unless overridden)
- Global settings
- MCP server connections
- Tool pool (unless restricted by agent definition)

---

## Multi-Agent Example Flow

```
User: "Analyze the codebase architecture in parallel"

Main Agent:
  1. Spawns AgentTool(name="explore-frontend", run_in_background=true,
       prompt="Explore src/components/, catalog all React components")
  2. Spawns AgentTool(name="explore-backend", run_in_background=true,
       prompt="Explore src/services/, catalog all service modules")
  3. Spawns AgentTool(name="explore-utils", run_in_background=true,
       prompt="Explore src/utils/, catalog utility functions")
  
  [All three run in parallel]
  
  4. TaskList → sees all three running
  5. Waits for completion notifications
  6. Receives results from all three
  7. Synthesizes findings into architecture report
```

---

## Cron / Scheduled Tasks (`src/tools/ScheduleCronTool/`)

Feature-gated (`AGENT_TRIGGERS`). Allows scheduling recurring agent tasks:

```typescript
// CronCreate
z.object({
  schedule: z.string().describe('Cron expression (e.g., "0 9 * * *")'),
  prompt: z.string().describe('Prompt for the scheduled agent'),
  description: z.string().describe('Human-readable description'),
  agentType: z.string().optional(),
})

// Creates a SessionCronTask in bootstrap state
// Triggers agent execution on schedule
// Notifies user via PushNotificationTool (if Kairos enabled)
```
