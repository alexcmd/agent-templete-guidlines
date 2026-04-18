# Universal Agent Architecture — Core Runtime & Providers

> Sections: Agent Runtime Model · Component Inventory · Multi-Provider Registry · Layered Architecture

Part of the [Universal Agent Architecture](01-overview.md) series.

# Universal Agent Architecture

Cross-language architectural patterns derived from analyzing four production agent CLIs and
24 arXiv papers. Read this document first, then proceed to the language-specific guide.

---

## 1. The Agent Runtime Model

Every production agent system studied converges on the same fundamental runtime shape:

```
┌─────────────────────────────────────────────────────────────────────┐
│                          AGENT RUNTIME                               │
│                                                                       │
│  User Input                                                           │
│      │                                                                │
│      ▼                                                                │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                     TURN LOOP                                 │    │
│  │                                                               │    │
│  │  build_request()                                              │    │
│  │      → system_prompt (static cache boundary)                 │    │
│  │      → dynamic_context (git status, date, config)            │    │
│  │      → message_history (compacted if needed)                 │    │
│  │      → tool_definitions                                       │    │
│  │                                                               │    │
│  │  stream_response()                ←── LLM API                │    │
│  │      → text deltas → user display                            │    │
│  │      → tool_use blocks → collect                             │    │
│  │                                                               │    │
│  │  for each tool_call:                                          │    │
│  │      pre_hook()                                               │    │
│  │      check_permission()                                       │    │
│  │      execute_tool() → ToolResult                             │    │
│  │      post_hook()                                              │    │
│  │                                                               │    │
│  │  append tool_results to history                               │    │
│  │  if stop_reason == end_turn: break                            │    │
│  │  else: continue loop (tool_results feed next request)        │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                        │
│  Output                                                                │
└────────────────────────────────────────────────────────────────────────┘
```

**Key invariants that must hold in every language:**

1. The LLM never sees intermediate tool results until a full API round-trip completes.
2. Every ToolUse message must be followed by its ToolResult in the same conversation turn before compaction can split the history.
3. The system prompt has a static section (cached) and a dynamic section (per-turn); they must be in that order.
4. The loop runs until `stop_reason == end_turn`, not until the first tool call completes.
5. If the stream stalls (no data for N seconds), retry the entire request up to R times before reporting failure.
6. If the API returns overloaded/rate-limited, silently switch to a configured `fallback_model` rather than failing the session.
7. If `stop_reason == max_tokens`, inject a recovery message and retry rather than stopping mid-thought.

---

## 2. Component Inventory

Every agent needs some components and should consider others. Build in this order.

| Component | Priority | Complexity | Notes |
|-----------|----------|------------|-------|
| Agent loop (streaming) | MUST | Medium | Core turn cycle |
| System prompt builder | MUST | Low | Static/dynamic split |
| Tool registry + execution | MUST | Medium | Schema validation + dispatch |
| Permission system | MUST | Medium | Security gate |
| Session JSONL storage | MUST | Low | CWD-hash → file |
| Config loader (layered) | MUST | Low | 5-layer merge |
| Auth handler | MUST | Medium | API key + OAuth |
| Memory file loader | SHOULD | Low | [AGENT].md hierarchy |
| Context compaction | SHOULD | Medium | Long-session survival |
| Hook system | SHOULD | Medium | Extensibility + audit |
| Cost/token tracker | SHOULD | Low | Budget cap + UX |
| Loop detector | SHOULD | Medium | Safety |
| Fallback model | SHOULD | Low | Resilience on 429/529 |
| Stream stall watchdog | SHOULD | Low | Resilience |
| MCP client | OPTIONAL | High | External tool ecosystem |
| Sub-agent spawner | OPTIONAL | High | Parallel work |
| Concurrent tool execution | OPTIONAL | Medium | 10× throughput (read-only tools) |
| Tool output masking | OPTIONAL | Low | Large output history management |
| Model router | OPTIONAL | High | 7-strategy composite routing |
| Plugin system | OPTIONAL | High | Extension marketplace |
| Session branching | OPTIONAL | Medium | Full conversation tree |
| IDE context injection | OPTIONAL | High | VS Code companion (delta-based) |
| Sandbox executor | OPTIONAL | High | Landlock/Docker/Seatbelt |

---

## 3. Multi-Provider Registry

Production agents must support multiple LLM backends behind a single interface. The pattern
is a trait-based provider registry:

```
┌─────────────────────────────────────────────────────────────────────┐
│  LlmProvider (trait/interface)                                        │
│  + create_message(request) → response                                │
│  + create_message_stream(request) → Stream<StreamEvent>              │
│  + list_models() → Vec<ModelInfo>                                    │
│  + health_check() → ProviderStatus                                   │
│  + capabilities() → ProviderCapabilities                             │
├──────────────────┬──────────────────────────────────────────────────┤
│  Native adapters │ OpenAI-compat factory                             │
│  ─────────────── │ ─────────────────────                             │
│  Anthropic       │ ollama, lm-studio, llama-cpp                      │
│  OpenAI          │ deepseek, groq, xai, mistral                      │
│  Google Gemini   │ openrouter, togetherai, fireworks                 │
│  Amazon Bedrock  │ huggingface, nebius, 20+ more                     │
│  Azure OpenAI    │                                                   │
│  GitHub Copilot  │ Custom base URL (any OAI-compatible endpoint)     │
└──────────────────┴──────────────────────────────────────────────────┘
                              │
                    ProviderRegistry
                    (selected at runtime via config/env/CLI flag)
```

**Provider selection precedence** (highest wins):
1. CLI flag `--provider` / `--api-base`
2. Environment variable (e.g. `AGENT_PROVIDER` / `LLM_BASE_URL` / `OLLAMA_HOST`)
3. Project settings `{cwd}/.agent/settings.json → provider_configs`
4. User settings `~/.agent/settings.json → provider_configs`
5. Compiled default

**Per-provider thinking/reasoning options** (must be negotiated at request time):
```
Anthropic Opus/Sonnet   → thinking { type, budget_tokens }
Google Gemini           → thinkingConfig { thinkingBudget }
Amazon Bedrock          → reasoningConfig { budgetTokens }
OpenAI o-series         → reasoningEffort: "low" | "medium" | "high"
Ollama / local          → (none — omit option block)
```

**No-API-key providers** (Ollama, lm-studio, llama-cpp) should set `no_api_key_required: true`
in the provider descriptor so the agent doesn't prompt for a key on startup.

**Stream resilience pattern:**
```
attempt = 0
while attempt < max_retries:
    start_stream()
    watch watchdog_timer(45s)
    if stall detected:
        attempt++; retry
    if status 429 or 529:
        switch to fallback_model
        attempt++; retry
    if stop_reason == max_tokens:
        inject recovery_message("Resume directly — no apology, no recap.")
        continue turn
    if stop_reason == end_turn:
        break
```

---

## 4. Layered Architecture

All three language implementations organize into the same five layers:

```
┌─────────────────────────────────────────────────────────────┐
│  LAYER 5: Interface                                          │
│  Terminal REPL / REST API / SSE / WebSocket / CI headless   │
├─────────────────────────────────────────────────────────────┤
│  LAYER 4: Orchestration                                      │
│  Multi-agent supervisor / Task registry / Cron scheduler    │
├─────────────────────────────────────────────────────────────┤
│  LAYER 3: Runtime                                            │
│  Turn loop / Permission engine / Hook runner / Compaction   │
├─────────────────────────────────────────────────────────────┤
│  LAYER 2: Intelligence                                       │
│  Tool system / Memory / RAG / Knowledge graph / Prompts     │
├─────────────────────────────────────────────────────────────┤
│  LAYER 1: Foundation                                         │
│  LLM API client / Session persistence / Config / Logging    │
└─────────────────────────────────────────────────────────────┘
```

**Dependency rule:** each layer only imports from layers below it. Layer 3 (Runtime) never
imports from Layer 4 (Orchestration). This enables the runtime to be reused unchanged across
single-agent and multi-agent deployments.

---


---

*[← Overview](01-overview.md) | [Next: System Prompt & Permissions →](01-prompts-permissions.md)*
