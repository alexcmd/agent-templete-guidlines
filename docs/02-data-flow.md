# Data Flow & Execution

## End-to-End Request Lifecycle

```
User types message in REPL
         │
         ▼
┌─────────────────────┐
│  REPL.tsx           │  React component handles keypress → submits message
│  screens/REPL.tsx   │
└──────────┬──────────┘
           │ submitMessage(prompt)
           ▼
┌─────────────────────┐
│  QueryEngine        │  Manages conversation state, message history
│  QueryEngine.ts     │  Normalizes input, builds context, calls query()
└──────────┬──────────┘
           │ query(systemPrompt, messages, tools, ...)
           ▼
┌─────────────────────────────────────────────────────┐
│  query.ts — Main Streaming Loop                      │
│                                                       │
│  1. Pre-sampling hooks                               │
│  2. Build API request params                         │
│  3. Stream API call → Anthropic Claude               │
│  4. Process stream events                            │
│  5. On tool_use → execute tool                       │
│  6. Append tool_result → continue loop               │
│  7. On end_turn → yield final message, exit loop     │
│  8. Post-sampling hooks                              │
└──────────┬──────────────────────────────────────────┘
           │ stream events / tool calls
           ▼
┌─────────────────────┐      ┌─────────────────────────┐
│  Tool Execution     │      │  Anthropic API           │
│  src/tools/*/       │◄────►│  (streaming SSE)         │
│  Permission check   │      │  claude-sonnet-4-6       │
└─────────────────────┘      └─────────────────────────┘
           │ yields results/messages
           ▼
┌─────────────────────┐
│  QueryEngine        │  Accumulates messages, tracks usage/cost
│  Persists session   │  Records transcript, flushes storage
└──────────┬──────────┘
           │ yields SDKMessages
           ▼
┌─────────────────────┐
│  REPL.tsx           │  Renders messages in terminal UI
│  React/Ink          │  Updates spinner, status line, costs
└─────────────────────┘
```

---

## Bootstrap / Init Flow

```
main.tsx
  │
  ├─ Fast paths (no module loading):
  │   ├─ --version → print and exit
  │   ├─ --dump-system-prompt → render and exit
  │   ├─ --chrome-native-host → Chrome messaging
  │   ├─ --computer-use-mcp → MCP server
  │   └─ --daemon-worker=<kind> → worker process
  │
  ├─ Bridge mode (--remote-control / --bridge / --sync):
  │   ├─ Auth check
  │   ├─ GrowthBook gate check
  │   └─ bridgeMain()
  │
  └─ Full init:
      ├─ entrypoints/init.ts → enableConfigs()
      │   ├─ applySafeConfigEnvironmentVariables()
      │   ├─ applyExtraCACertsFromConfig()
      │   ├─ setupGracefulShutdown()
      │   ├─ import firstPartyEventLogger (async)
      │   ├─ import growthbook (async)
      │   ├─ populateOAuthAccountInfoIfNeeded()
      │   ├─ initJetBrainsDetection()
      │   ├─ detectCurrentRepository()
      │   ├─ initializeRemoteManagedSettingsLoadingPromise()
      │   ├─ configureGlobalMTLS()
      │   └─ configureGlobalAgents()
      │
      └─ entrypoints/cli.tsx → CLI command handling
          ├─ Parse argv
          ├─ Handle non-interactive modes (--print, -p, --json, etc.)
          └─ Launch REPL with React/Ink
```

---

## QueryEngine.submitMessage() Flow

```typescript
async *submitMessage(prompt: string | ContentBlockParam[]): AsyncGenerator<SDKMessage> {
  // 1. Set working directory for this turn
  setCwd(config.cwd)

  // 2. Process user input (parse /commands, extract attachments, etc.)
  const processUserInputContext = await processUserInput(prompt, {
    commands, tools, mcpClients, ...
  })

  // 3. Fetch and assemble system prompt parts
  const systemPromptParts = await fetchSystemPromptParts({
    tools, commands, mcpClients, settings, memories, ...
  })

  // 4. Normalize messages for API format
  normalizeMessagesForAPI(this.mutableMessages)

  // 5. Call inner query() loop
  for await (const event of query(
    systemPromptParts,
    userContext,
    systemContext,
    toolUseContext,
    this.mutableMessages,
    ...
  )) {
    this.mutableMessages.push(event)   // accumulate
    yield mapToSDKMessage(event)       // emit to caller
  }

  // 6. Session persistence
  if (!isSessionPersistenceDisabled()) {
    recordTranscript(this.mutableMessages)
    flushSessionStorage()
  }
}
```

---

## Inner query() Streaming Loop (query.ts)

```typescript
async function* query(
  systemPrompt, userContext, systemContext,
  toolUseContext, messages, ...
): AsyncGenerator<Message | StreamEvent | TombstoneMessage> {

  // ─── PRE-SAMPLING ────────────────────────────────────────
  // 1. Handle pre-sampling hooks (plugin-registered)
  yield* runPreSamplingHooks(toolUseContext, messages)

  // ─── CONTEXT MANAGEMENT ──────────────────────────────────
  // 2. Check token budget; compact if needed
  if (shouldAutoCompact(messages, tokenBudget)) {
    messages = await buildPostCompactMessages(messages)
  }

  // ─── API CALL ────────────────────────────────────────────
  // 3. Build and fire the API request
  const apiParams = {
    model: resolvedModel,
    system: buildSystemPromptBlocks(systemPrompt),
    messages: normalizeMessagesForAPI(messages),
    tools: tools.map(toolToAPISchema),
    max_tokens: calculateMaxTokens(model, budget),
    thinking: shouldEnableThinking ? { type: 'enabled', budget_tokens } : undefined,
    betas: getMergedBetas(model, features),
  }
  const stream = client.messages.stream(apiParams)
  yield { type: 'request_start', request_id }

  // ─── STREAM PROCESSING ───────────────────────────────────
  // 4. Process stream events
  let currentToolUses: ToolUseBlock[] = []
  for await (const event of stream) {
    if (event.type === 'content_block_start') {
      if (event.content_block.type === 'tool_use') {
        currentToolUses.push(event.content_block)
      }
    }
    if (event.type === 'content_block_delta') {
      // Accumulate text deltas, stream to UI
      yield { type: 'stream_event', event }
    }
    if (event.type === 'message_stop') {
      // Emit full assistant message
      yield createAssistantMessage(accumulatedContent)
    }
  }

  // ─── TOOL EXECUTION ──────────────────────────────────────
  // 5. Execute all tool calls from this turn
  if (currentToolUses.length > 0) {
    const toolResults = []
    for (const toolUse of currentToolUses) {
      const tool = findToolByName(toolUse.name, tools)
      const canUse = await canUseTool(tool, toolUse.input, toolUseContext)

      if (canUse === 'allow') {
        const result = await executeToolWithProgress(tool, toolUse.input, toolUseContext)
        toolResults.push({ tool_use_id: toolUse.id, content: result })
        yield createProgressMessage(result)
      } else if (canUse === 'deny') {
        toolResults.push({ tool_use_id: toolUse.id, content: 'Permission denied' })
      } else {
        // 'ask' — show permission dialog, wait for user
        const userDecision = await promptUserForPermission(tool, toolUse)
        // ... handle decision
      }
    }

    // 6. Append tool results and recurse (continue loop)
    messages.push(createUserMessage({ content: toolResults }))
    yield* query(systemPrompt, userContext, systemContext, toolUseContext, messages, ...)
  }

  // ─── POST-SAMPLING ───────────────────────────────────────
  // 7. Handle stop hooks
  yield* runPostSamplingHooks(toolUseContext, messages)
}
```

---

## Tool Execution Detail

```
AgentTool / BashTool / FileEditTool / ...
       │
       ▼
canUseTool(tool, input, context)
       │
  ┌────┴────┐
  │ Rules   │  Evaluate alwaysAllow / alwaysDeny / alwaysAsk arrays
  │ Check   │  Match tool name + input patterns (e.g., "Bash(rm -rf *)")
  └────┬────┘
       │
  ┌────┴────────────────────────────────────────────┐
  │ ALLOW → execute immediately                      │
  │ DENY  → return error message, no execution       │
  │ ASK   → show permission dialog in REPL           │
  │          user clicks Allow/Deny/Always Allow/etc.│
  └──────────────────────────────────────────────────┘
       │ ALLOW
       ▼
tool.handler(input, context)
       │
  ┌────┴────────────────────────────────────────────┐
  │ Sync: return string result                        │
  │ Async: return Promise<string>                     │
  │ Streaming: yield* progress events                 │
  └──────────────────────────────────────────────────┘
       │
       ▼
ToolResultBlockParam  →  appended to messages  →  next iteration
```

---

## Compaction Flow (Context Window Management)

When the conversation approaches the model's context limit, Claude Code compacts (summarizes) old messages:

```
Token count approaches limit
       │
       ▼
autoCompact.shouldCompact(messages, tokenBudget)
       │ yes
       ▼
buildPostCompactMessages(messages)
       │
  ┌────┴──────────────────────────────────────┐
  │ 1. Find compact boundary marker           │
  │ 2. Keep recent messages (last N turns)    │
  │ 3. Summarize older messages via Claude    │
  │ 4. Insert compact summary as system msg  │
  │ 5. Return truncated + summary messages   │
  └───────────────────────────────────────────┘
       │
       ▼
Continue query with compacted messages
```

### Compaction Variants
- **Standard compaction** — summaries old turns, keeps recent ones
- **Micro-compaction** — lighter summarization for smaller contexts
- **Reactive compaction** — triggered by specific content patterns (feature-gated)
- **Context collapse** — aggressive truncation (feature-gated)

---

## Streaming Message Types

Messages flowing through the system:

| Type | Direction | Purpose |
|---|---|---|
| `UserMessage` | User→Model | User input, tool results |
| `AssistantMessage` | Model→User | Text responses, tool calls |
| `SystemMessage` | System→Model | Context injections |
| `ProgressMessage` | Internal | Streaming tool progress (shown in UI) |
| `AttachmentMessage` | Internal | Memory/file attachments |
| `ToolUseSummaryMessage` | Internal | Collapsed tool use for display |
| `StreamEvent` | Stream | Raw stream deltas (content_block_delta, etc.) |
| `RequestStartEvent` | Stream | API request metadata |
| `TombstoneMessage` | Internal | Replaced/deleted messages |
| `SDKCompactBoundaryMessage` | SDK | Compact boundary marker |

---

## Session Persistence

After each completed query, Claude Code persists the session:

```
recordTranscript(messages)
  → Writes to ~/.claude/projects/<project-hash>/sessions/<session-id>.jsonl

flushSessionStorage()
  → Writes pending file state, memory, metadata
```

Sessions can be resumed with `claude --resume <session-id>`.
