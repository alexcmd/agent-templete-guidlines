# Agent Loop & Query Engine

## QueryEngine Class

`src/QueryEngine.ts` is the session container. One `QueryEngine` per conversation (including sub-agents).

### Responsibilities
- Maintains `mutableMessages` array (the full conversation history)
- Manages `AbortController` for cancellation
- Tracks `totalUsage` (tokens, costs) across the session
- Loads memory prompts from `.claude/` on first query
- Handles session persistence (transcript recording)
- Tracks discovered skill names for context injection

### Key State
```typescript
class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private discoveredSkillNames = new Set<string>()
  private loadedNestedMemoryPaths = new Set<string>()
  private fileStateCache: FileStateCache
  private fileHistoryState: FileHistoryState
  private processUserInputContext?: ProcessUserInputContext
}
```

### QueryEngineConfig
```typescript
type QueryEngineConfig = {
  sessionId: SessionId
  cwd: string
  initialMessages?: Message[]         // Resume from prior session
  customSystemPrompt?: string
  appendSystemPrompt?: string
  permissionMode: PermissionMode
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  mcpResources: Record<string, ServerResource[]>
  agentDefinitions: AgentDefinitionsResult
  mainLoopModel: ModelSetting
  parentSessionId?: SessionId         // Sub-agent context
  agentId?: AgentId
  isNonInteractive: boolean
  verbose: boolean
  debug: boolean
  querySource?: QuerySource
  isCoordinator?: boolean
}
```

---

## Inner Loop: query.ts

The `query()` function is a generator that yields messages/events as the model responds and tools execute. It's recursive — tool results are appended as user messages and the function calls itself to continue.

### Full Loop Logic

```
ENTER query()
  │
  ├─ 1. Start relevant memory prefetch (async, non-blocking)
  ├─ 2. Check for reactive compaction trigger
  ├─ 3. Handle pre-sampling hooks (yields hook progress events)
  │
  ├─ 4. Build API request
  │     ├─ resolve model (main loop model, fast mode, fallback)
  │     ├─ prependUserContext(messages, userContext)
  │     ├─ appendSystemContext(systemPrompt, systemContext)
  │     ├─ splitSysPromptPrefix → cache scope optimization
  │     └─ getMergedBetas(model) → feature flags
  │
  ├─ 5. Fire streaming API call → client.messages.stream(params)
  │     yield RequestStartEvent
  │
  ├─ 6. Process stream:
  │     for each event in stream:
  │       ├─ content_block_start (tool_use) → record tool call
  │       ├─ content_block_delta (text) → stream text to UI
  │       ├─ content_block_stop → finalize content block
  │       ├─ message_delta → update stop_reason, usage
  │       └─ message_stop → yield complete AssistantMessage
  │
  ├─ 7. If stop_reason === 'tool_use':
  │     ├─ Classify tool calls (executeNow vs deferred)
  │     ├─ For each tool call:
  │     │   ├─ canUseTool() → ALLOW / DENY / ASK
  │     │   ├─ If ASK → yield PermissionRequest, wait for resolution
  │     │   ├─ execute handler() → yield progress events
  │     │   └─ collect ToolResultBlockParam
  │     ├─ yield UserMessage(toolResults)
  │     └─ RECURSE → query() with updated messages
  │
  ├─ 8. If stop_reason === 'end_turn':
  │     ├─ Check token budget (warn if approaching limit)
  │     └─ EXIT loop
  │
  ├─ 9. If stop_reason === 'max_tokens':
  │     ├─ Trigger compaction if auto-compact enabled
  │     └─ RECURSE with compacted messages
  │
  └─ 10. Post-sampling hooks (yields hook progress events)
```

---

## Compaction System (`src/services/compact/`)

### Auto-Compact Detection
```typescript
// autoCompact.ts
function shouldCompact(
  messages: Message[],
  usage: BetaUsage,
  model: string,
): boolean {
  const contextLimit = getModelContextLimit(model)
  const usedTokens = usage.input_tokens + usage.output_tokens
  const threshold = contextLimit * 0.85  // 85% threshold
  return usedTokens > threshold
}
```

### Compaction Process
```typescript
// compact.ts
async function buildPostCompactMessages(
  messages: Message[],
  systemPrompt: SystemPrompt,
  model: string,
): Promise<Message[]> {
  // 1. Find compact boundary (keeps messages after last boundary)
  const boundary = findCompactBoundary(messages)
  const recentMessages = messages.slice(boundary)
  const oldMessages = messages.slice(0, boundary)

  // 2. Summarize old messages via a separate Claude call
  const summary = await generateSummary(oldMessages, model)

  // 3. Build new message list
  return [
    createSystemMessage({ subtype: 'compact', content: summary }),
    ...recentMessages
  ]
}
```

### Compaction Variants

| Type | Trigger | Approach |
|------|---------|----------|
| Standard | Token limit approaching | Full summary of old turns |
| Micro-compact | Intermediate boundary | Lighter summarization |
| Reactive | Content patterns (feature-gated) | Pattern-specific condensation |
| Context collapse | Aggressive limit (feature-gated) | Maximum truncation |

---

## Token Budget System (`src/query/tokenBudget.ts`)

```typescript
type TokenBudgetState = {
  inputTokensUsed: number
  outputTokensUsed: number
  cacheReadTokensUsed: number
  warningState: 'none' | 'low' | 'critical' | 'exceeded'
  shouldCompact: boolean
  shouldStop: boolean
}

function calculateTokenWarningState(
  usage: BetaUsage,
  model: string,
): TokenBudgetState {
  const maxContext = getModelMaxContextWindow(model)
  const ratio = usage.input_tokens / maxContext
  
  return {
    warningState: ratio > 0.95 ? 'critical' 
                : ratio > 0.85 ? 'low'
                : 'none',
    shouldCompact: ratio > 0.85 && isAutoCompactEnabled(),
    shouldStop: ratio > 0.99,
    // ...
  }
}
```

---

## Query Configuration (`src/query/config.ts`)

```typescript
type QueryConfig = {
  sessionId: SessionId
  
  gates: {
    streamingToolExecution: boolean  // Execute tools while streaming
    emitToolUseSummaries: boolean    // Collapse tool output in UI
    isAnt: boolean                   // Anthropic employee mode
    fastModeEnabled: boolean         // Fast mode (Opus 4.6 fast)
  }
  
  maxTurns?: number                  // Hard limit on tool call rounds
  stopSequences?: string[]           // Custom stop sequences
}
```

---

## Query Hooks (`src/query/hooks/`)

Hooks execute at specific points in the query lifecycle:

### Pre-Sampling Hooks
Run before the API call. Can modify messages or inject context.

```typescript
// Registered via plugin manifest or settings hooks config
async function* runPreSamplingHooks(
  toolUseContext: ToolUseContext,
  messages: Message[],
): AsyncGenerator<HookProgress> {
  for (const hook of getPreSamplingHooks()) {
    yield { type: 'hook_start', hookName: hook.name }
    await executeHookCommand(hook.command, { messages })
    yield { type: 'hook_complete', hookName: hook.name }
  }
}
```

### Post-Sampling Hooks (Stop Hooks)
Run after the model finishes (end_turn). Used for notifications, auto-commits, etc.

```typescript
// Common use: run 'git commit' after Claude finishes coding
// settings.json:
{
  "hooks": {
    "Stop": [{ "hooks": [{ "type": "command", "command": "notify-send Done" }] }]
  }
}
```

### PreToolUse Hooks
Run before a specific tool executes. Can block execution or modify input.

### PostToolUse Hooks
Run after a specific tool completes. Gets tool name, input, and output.

---

## Streaming Tool Executor (`src/services/tools/StreamingToolExecutor.ts`)

When `streamingToolExecution` gate is enabled, tools can yield progress before returning final result:

```typescript
async function* executeToolWithStreaming(
  tool: Tool,
  input: Record<string, unknown>,
  context: ToolUseContext,
): AsyncGenerator<ToolCallProgress | string> {
  if (tool.supportsStreaming) {
    // Tool is a generator — stream its output
    const gen = tool.handler(input, context) as AsyncGenerator<ToolCallProgress>
    for await (const progress of gen) {
      yield progress
    }
  } else {
    // Tool returns a promise — no streaming
    const result = await (tool.handler(input, context) as Promise<string>)
    yield result
  }
}
```

---

## Message Normalization

Before every API call, messages are normalized:

```typescript
function normalizeMessagesForAPI(messages: Message[]): BetaMessageParam[] {
  return messages
    .filter(m => m.type === 'user' || m.type === 'assistant')
    .map(m => {
      if (m.type === 'user') {
        return {
          role: 'user',
          content: m.message.content,
        }
      }
      return {
        role: 'assistant',
        content: m.message.content.filter(
          b => b.type !== 'thinking' || isThinkingEnabled()
        ),
      }
    })
    // Ensure alternating user/assistant messages
    // Merge consecutive same-role messages
    .reduce(mergeConsecutiveRoles, [])
}
```

---

## Error Handling in the Loop

```typescript
try {
  for await (const event of stream) {
    // ...
  }
} catch (error) {
  if (error instanceof FallbackTriggeredError) {
    // Switch to fallback model and retry
    yield* query({ ...params, model: getFallbackModel() })
  } else if (isRetryableError(error)) {
    // Exponential backoff retry
    await sleep(getRetryDelay(attempt))
    yield* query(params)
  } else if (isPromptTooLongMessage(error)) {
    yield createAssistantAPIErrorMessage(PROMPT_TOO_LONG_ERROR_MESSAGE)
  } else {
    yield createAssistantAPIErrorMessage(errorMessage(error))
  }
}
```

---

## Turn Counting & Limits

```typescript
// Each tool call round = one "turn"
let turnCount = 0
const maxTurns = config.maxTurns ?? DEFAULT_MAX_TURNS  // typically unlimited

while (stop_reason === 'tool_use') {
  turnCount++
  if (maxTurns && turnCount >= maxTurns) {
    yield createSystemMessage('Max turns reached')
    break
  }
  // execute tools, recurse
}
```
