# Core Data Structures

## 1. Tool Interface (`src/Tool.ts`)

The central abstraction — every capability Claude can invoke.

```typescript
export type Tool = {
  name: string                    // Machine name ("Bash", "Edit", "Agent")
  description: string             // Shown to model in system prompt
  displayName?: string            // Human-friendly name for UI
  inputSchema: ToolInputJSONSchema // JSON Schema for input validation
  
  handler: (
    input: Record<string, unknown>,
    context: ToolUseContext,
  ) => Promise<string> | AsyncGenerator<ToolCallProgress>
  
  isEnabled?: (context: ToolUseContext) => boolean | Promise<boolean>
  supportsStreaming?: boolean      // Can emit progress events
  isSensitive?: boolean           // Requires extra permission caution
}

export type ToolInputJSONSchema = {
  type: 'object'
  properties?: { [key: string]: unknown }
  required?: string[]
  [key: string]: unknown
}
```

### ToolUseContext

Passed to every tool handler — provides access to app state, other tools, MCP clients.

```typescript
export type ToolUseContext = {
  options: {
    commands: Command[]
    tools: Tools
    debug: boolean
    mainLoopModel: string
    mcpClients: MCPServerConnection[]
    mcpResources: Record<string, ServerResource[]>
    agentDefinitions: AgentDefinitionsResult
  }
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  readFileState: FileStateCache       // Cached file reads for diffing
  abortController: AbortController
  canUseTool: CanUseToolFn
  agentId?: AgentId
  sessionId?: SessionId
  queryChainTracking?: QueryChainTracking
  thinkingConfig?: ThinkingConfig
  attributionState?: AttributionState
  fileHistoryState?: FileHistoryState
  contentReplacementState?: ContentReplacementState
  notifications: Notification[]
}
```

### ToolPermissionContext

Immutable snapshot of permission rules for the current session.

```typescript
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode           // 'default' | 'bypass' | 'plan' | 'auto'
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource  // Rules to auto-allow
  alwaysDenyRules: ToolPermissionRulesBySource   // Rules to auto-deny
  alwaysAskRules: ToolPermissionRulesBySource    // Rules requiring prompt
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  shouldAvoidPermissionPrompts?: boolean
}>

type PermissionMode = 
  | 'default'   // Standard — ask for sensitive operations
  | 'bypass'    // No prompts — auto-allow everything (--dangerously-skip-permissions)
  | 'plan'      // Plan mode — must confirm before execution
  | 'auto'      // Automated mode — uses classifier to decide
```

---

## 2. Message Types (`src/types/message.ts`, `src/types/`)

### Core Message Union

```typescript
type Message =
  | UserMessage
  | AssistantMessage
  | SystemMessage
  | ToolUseSummaryMessage
  | AttachmentMessage
  | ProgressMessage
  | RequestStartEvent
  | TombstoneMessage
  | SystemLocalCommandMessage
  | StreamEvent
  | SystemAPIErrorMessage
```

### UserMessage
```typescript
type UserMessage = {
  type: 'user'
  message: {
    role: 'user'
    content: Array<
      | TextBlockParam            // Plain text
      | ToolResultBlockParam      // Tool execution result
      | ImageBlockParam           // Pasted image
      | DocumentBlockParam        // Attached document
    >
  }
  uuid: string
}
```

### AssistantMessage
```typescript
type AssistantMessage = {
  type: 'assistant'
  message: {
    role: 'assistant'
    content: Array<
      | TextBlock                 // Model text response
      | ToolUseBlock              // Tool call (name + input)
      | ThinkingBlock             // Extended thinking
    >
    stop_reason: BetaStopReason  // 'end_turn' | 'tool_use' | 'max_tokens' | ...
    usage: BetaUsage
  }
  uuid: string
  costUSD?: number
  durationMs?: number
  requestId?: string
}
```

### SystemMessage
```typescript
type SystemMessage = {
  type: 'system'
  subtype:
    | 'init'            // Session start with metadata
    | 'compact'         // Compact summary marker
    | 'microcompact'    // Micro-compact boundary
    | 'error'           // API or system error
    | 'command_output'  // Local command output
    | 'notification'    // User-visible notification
  content: string | ContentBlock[]
  uuid: string
}
```

### ProgressMessage
```typescript
type ProgressMessage = {
  type: 'progress'
  toolUseId: string
  data: ToolProgressData   // Tool-specific progress payload
  uuid: string
}

type ToolProgressData =
  | BashProgress
  | AgentToolProgress
  | SkillToolProgress
  | MCPProgress
  | WebSearchProgress
  | TaskOutputProgress
  | REPLToolProgress
  | HookProgress
```

---

## 3. AppState (`src/state/AppState.tsx`)

Global immutable state accessed via React Context and the state store.

```typescript
type AppState = {
  // ─── Settings & Model ───────────────────────────────────
  settings: SettingsJson
  mainLoopModel: ModelSetting
  verbose: boolean
  isBriefOnly: boolean
  
  // ─── UI State ────────────────────────────────────────────
  statusLineText: string | undefined
  expandedView: 'none' | 'tasks' | 'teammates'
  selectedIPAgentIndex: number
  coordinatorTaskIndex: number
  viewSelectionMode: 'none' | 'selecting-agent' | 'viewing-agent'
  footerSelection: FooterItem | null
  
  // ─── Permissions ─────────────────────────────────────────
  toolPermissionContext: ToolPermissionContext
  
  // ─── Agent Identity ──────────────────────────────────────
  agent: string | undefined
  
  // ─── Remote / Bridge ─────────────────────────────────────
  kairosEnabled: boolean
  remoteSessionUrl: string | undefined
  remoteConnectionStatus: 'connecting' | 'connected' | 'reconnecting' | 'disconnected'
  remoteBackgroundTaskCount: number
  replBridgeEnabled: boolean
  replBridgeConnected: boolean
  
  // ─── Tasks (Mutable sub-object) ──────────────────────────
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>
  
  // ─── MCP ────────────────────────────────────────────────
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
    commands: Command[]
    resources: Record<string, ServerResource[]>
    pluginReconnectKey: number
  }
  
  // ─── Plugins ─────────────────────────────────────────────
  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
    commands: Command[]
    errors: PluginError[]
    installationStatus: {
      inProgress: string[]
      errors: Record<string, string>
    }
  }
}
```

---

## 4. Bootstrap/Global State (`src/bootstrap/state.ts`)

Process-level singletons (not React state).

```typescript
type State = {
  // ─── Session Identity ────────────────────────────────────
  originalCwd: string
  projectRoot: string
  sessionId: SessionId

  // ─── Cost Tracking ───────────────────────────────────────
  totalCostUSD: number
  modelUsage: {
    [modelName: string]: {
      inputTokens: number
      outputTokens: number
      cacheReadInputTokens: number
      cacheCreationInputTokens: number
    }
  }

  // ─── Telemetry ───────────────────────────────────────────
  meter: Meter | null               // OpenTelemetry meter
  sessionCounter: AttributedCounter | null

  // ─── Multi-Agent ─────────────────────────────────────────
  agentColorMap: Map<string, AgentColorName>  // Color per agent for UI

  // ─── Debugging ───────────────────────────────────────────
  lastAPIRequest: Omit<BetaMessageStreamParams, 'messages'> | null

  // ─── Scheduled Tasks ─────────────────────────────────────
  scheduledTasksEnabled: boolean
  sessionCronTasks: SessionCronTask[]
  sessionCreatedTeams: Set<string>
}
```

---

## 5. Command Types (`src/types/command.ts`)

```typescript
type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)

type CommandBase = {
  name: string                        // e.g., "review", "update-config"
  aliases?: string[]
  description: string
  isEnabled?: () => boolean
  isHidden?: boolean
  availability?: CommandAvailability[] // 'claude-ai' | 'console'
  userInvocable?: boolean
  loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin' | 'bundled' | 'mcp'
}

// Prompt-based skills (model invokes them via SkillTool)
type PromptCommand = {
  type: 'prompt'
  progressMessage: string
  contentLength: number
  model?: string
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  context?: 'inline' | 'fork'
  getPromptForCommand(
    args: string, 
    context: ToolUseContext
  ): Promise<ContentBlockParam[]>
}

// Traditional CLI commands (run synchronously)
type LocalCommand = {
  type: 'local'
  load: () => Promise<LocalCommandModule>
}

// React-based UI commands (render a component)
type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<LocalJSXCommandModule>
}
```

---

## 6. Settings Schema (`src/utils/settings/settings.ts`)

```typescript
type SettingsJson = {
  // ─── Model ───────────────────────────────────────────────
  model?: ModelSetting              // e.g., 'claude-sonnet-4-6'
  effort?: 'low' | 'medium' | 'high' | 'max'
  
  // ─── Display ─────────────────────────────────────────────
  theme?: ThemeName
  verbose?: boolean
  preferredNotifChannel?: 'terminal-bell' | 'iterm2' | 'system'
  
  // ─── Permissions ─────────────────────────────────────────
  permissions?: ToolPermissionRulesBySource
  
  // ─── Hooks ───────────────────────────────────────────────
  hooks?: HooksSettings
  
  // ─── Plugins & MCP ───────────────────────────────────────
  plugins?: PluginConfig
  enabledPlugins?: Record<string, boolean>
  mcpServers?: Record<string, ScopedMcpServerConfig>
  
  // ─── Features ────────────────────────────────────────────
  autoUpdaterStatus?: 'enabled' | 'disabled'
  hasCompletedProjectOnboarding?: boolean
  language?: string
  outputStyle?: string
  
  // ─── Session ─────────────────────────────────────────────
  customSystemPrompt?: string
  includeCoAuthoredBy?: boolean
}

type SettingSource =
  | 'globalSettings'    // ~/.claude/settings.json
  | 'projectSettings'   // .claude/settings.json
  | 'userSettings'      // ~/.claude/settings.json (alias)
  | 'localSettings'     // .claude/settings.local.json
  | 'managedSettings'   // sync.claude.ai (remote)
  | 'mdm'               // MDM policy (enterprise)
```

---

## 7. Task State (`src/tasks/`)

```typescript
type TaskState = {
  id: TaskId
  type: 'local-agent' | 'remote-agent' | 'local-shell' | 'cron'
  status: 'running' | 'completed' | 'failed' | 'cancelled'
  description: string
  progress?: string
  outputLines: string[]
  error?: string
  agentId?: AgentId
  parentSessionId?: SessionId
  startedAt: number         // Unix ms
  completedAt?: number
  tokenCount?: number
  costUSD?: number
}
```

---

## 8. MCP Types (`src/services/mcp/types.ts`)

```typescript
type MCPServerConnection = {
  name: string
  client: Client                     // MCP SDK client
  status: 'connected' | 'connecting' | 'failed'
  tools: MCPTool[]
  resources?: ServerResource[]
  error?: string
  serverInfo?: { name: string; version: string }
}

type ServerResource = {
  uri: string
  name: string
  description?: string
  mimeType?: string
}
```

---

## 9. Plugin Manifest (`src/utils/plugins/schemas.ts`)

```typescript
type PluginManifest = {
  name: string
  version: string
  description: string
  author?: {
    name: string
    email?: string
    url?: string
  }
  skills?: {
    [skillName: string]: BundledSkillDefinition
  }
  commands?: CommandMetadata[]
  hooks?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  lspServers?: Record<string, LspServerConfig>
  outputStyles?: { [styleName: string]: unknown }
}
```

---

## 10. Agent Definition (`src/tools/AgentTool/loadAgentsDir.ts`)

Custom agent types defined in `.claude/agents/` or loaded from built-ins.

```typescript
type AgentDefinition = {
  name: string             // Unique identifier (e.g., "explore", "code-reviewer")
  description: string      // What the agent does (shown to model)
  model?: ModelSetting     // Override model for this agent type
  tools?: string[]         // Restricted tool subset
  systemPrompt?: string    // Custom system prompt prefix
  temperature?: number
  requiresMcp?: string[]   // Required MCP server names
}
```

---

## 11. Hooks Settings (`src/types/hooks.ts`)

```typescript
type HooksSettings = {
  PreToolUse?: HookConfig[]     // Before each tool call
  PostToolUse?: HookConfig[]    // After each tool call  
  Stop?: HookConfig[]           // When model stops (end_turn)
  SubagentStop?: HookConfig[]   // When sub-agent stops
  Notification?: HookConfig[]   // On system notification
  PreCompact?: HookConfig[]     // Before context compaction
}

type HookConfig = {
  matcher?: string              // Tool name pattern to match
  hooks: HookItem[]
}

type HookItem = {
  type: 'command'
  command: string               // Shell command to execute
  timeout?: number
}
```
