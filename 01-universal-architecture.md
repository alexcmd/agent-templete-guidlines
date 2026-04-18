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

## 5. System Prompt Structure

The system prompt has two sections separated by a **cache boundary**. Everything above the
boundary is invariant within a session and benefits from prompt caching; everything below it
changes per-turn.

```
══ STATIC (cached) ══════════════════════════════════════════
[1] Role introduction
    "You are an interactive agent that helps with X..."

[2] Capability declaration (optional output style section)

[3] Behavioral constraints
    "# Doing tasks\n - Prefer editing existing files..."

[4] Safety rules
    "# Executing actions with care\n..."

[5] Tool usage guidance
    "# Using your tools\n..."

══ CACHE BOUNDARY ═══════════════════════════════════════════

══ DYNAMIC (per-turn) ═══════════════════════════════════════
[6] Environment context
    "# Environment\n - Model: X\n - Date: Y\n - CWD: Z"

[7] Project context (if applicable)
    git status + diff + recent commits (budget: ~2 KB)

[8] Instruction files (CLAUDE.md / AGENT.md equivalent)
    Budget: 4 KB per file, 12 KB total

[9] Runtime config snapshot
    permission mode, allowed tools, MCP server status
```

**Why this matters:** Anthropic's prompt caching saves 90% of input token cost on the static
section. The boundary must be placed after the last static block and before the first dynamic
block. Moving a dynamic value (like a timestamp) above the boundary breaks caching for all
subsequent calls.

---

## 6. Permission Architecture

A three-tiered permission check must run before every tool execution:

```
Tool call arrives
      │
      ▼
┌─────────────────────────┐
│ 1. Deny rules (static)  │──── match ────► DENY immediately
└──────────┬──────────────┘
           │ no match
           ▼
┌─────────────────────────┐
│ 2. Hook override        │──── Deny ─────► DENY
│    (PreToolUse hook)    │──── Ask ──────► ASK user
└──────────┬──────────────┘──── Allow ────► ALLOW
           │ no override
           ▼
┌─────────────────────────┐
│ 3. Policy rules         │──── ask match ► ASK user
│    allow / ask rules    │──── allow match► ALLOW
└──────────┬──────────────┘
           │ no match
           ▼
┌─────────────────────────────────────┐
│ 4. Default: mode >= required_mode?  │──── YES ──► ALLOW
└──────────┬──────────────────────────┘
           │ NO
           ▼
           ASK user → Allow / Deny / Allow-all-session
```

**Permission modes (full list):**

| Mode | What it allows automatically |
|------|------------------------------|
| `readonly` | File reads, searches, queries only. All writes → DENY |
| `acceptEdits` | File writes allowed without prompting. Shell → ASK |
| `default` | Read-only tools → ALLOW. Write/shell → ASK |
| `plan` | Plan-phase enforcement: read-only + write only to plans directory |
| `auto` | Classifier-driven auto-approval (uses model to judge safety) |
| `bypassPermissions` / `yolo` | All tools → ALLOW. No prompts, no hooks (use only for CI/trusted pipelines) |

Every tool declares its `required_permission`. The active mode must satisfy the requirement
for auto-allow; otherwise the permission engine asks the user.

**PermissionRule schema:**

```python
@dataclass
class PermissionRule:
    source: str          # "userSettings"|"projectSettings"|"localSettings"|"cliArg"|"session"
    behavior: str        # "allow"|"deny"|"ask"
    tool_name: str       # e.g. "Bash", "Write"
    rule_content: str    # glob/pattern applied to serialized input, e.g. "git *", "*.py"
```

**Rule expression syntax:** `ToolName(glob)` — e.g. `Bash(git *)`, `Write(src/*)`, `Read(*)`

**Per-directory scoping:** Add `additional_working_directories` in config to grant write
permission only within specified paths. Write attempts outside those paths → DENY or ASK.

**Runtime dialog options (when ASK triggers):**
1. Allow once
2. Deny once
3. Allow for this session
4. Allow permanently (writes to `userSettings`)
5. Edit input in-place before allowing (user can modify the tool's input)

**Multi-agent permission inheritance:**
- Sub-agents inherit the parent's permission mode and rules by default
- Coordinator agents (supervisor role) can grant additional permissions to workers
- Workers cannot self-elevate permissions — they must request via coordinator
- Recommended: workers run in `default` mode; coordinator approves escalations

---

## 7. Memory Architecture (CoALA Taxonomy)

All modern agent memory systems organize into four stores (arXiv:2309.02427):

```
┌──────────────────────────────────────────────────────────────────┐
│  WORKING MEMORY (in-context)                                      │
│  Current conversation messages, active tool results               │
│  Capacity: ~200K tokens (Claude 3.x), managed by compaction      │
├──────────────────────────────────────────────────────────────────┤
│  EPISODIC MEMORY (external)                                       │
│  Past conversation summaries, trajectory records                  │
│  Backend: vector DB (Qdrant / USearch / pgvector)                │
│  Retrieval: top-k ANN by embedding similarity                    │
├──────────────────────────────────────────────────────────────────┤
│  SEMANTIC MEMORY (external)                                       │
│  Facts, entity relationships, knowledge graph                     │
│  Backend: graph DB (Neo4j / SQLite with adjacency table)          │
│  Retrieval: HippoRAG PPR or GraphRAG community summaries          │
├──────────────────────────────────────────────────────────────────┤
│  PROCEDURAL MEMORY (in-weights + tool library)                    │
│  LLM base knowledge + learned skills (ExpeL / Voyager)           │
│  Backend: in-weights (fine-tuning) or vector-indexed skill store  │
└──────────────────────────────────────────────────────────────────┘
```

**MemGPT 3-tier mapping** (arXiv:2310.08560) onto the CoALA taxonomy:

| MemGPT tier | CoALA category | Persistence |
|-------------|----------------|-------------|
| Main context | Working | In-context, lost on compaction |
| Recall storage | Episodic | Vector DB, persists across sessions |
| Archival storage | Semantic | Vector DB, long-term fact store |

### File-Based Memory (memdir pattern)

For agents without a vector DB dependency, structured markdown files provide sufficient
episodic and semantic memory at zero infrastructure cost:

```
~/.agent/memory/
├── user_role.md            (type: user)
├── feedback_testing.md     (type: feedback)
├── project_auth_rewrite.md (type: project)
└── reference_linear.md     (type: reference)
```

Each file carries YAML frontmatter:
```yaml
---
name: "Integration tests must hit real DB"
description: "Test strategy constraint from Q1 incident"
type: feedback
---
Do not mock the database. Prior incident: mock tests passed but prod migration failed.
**Why:** divergence between mock and prod masked a broken migration
**How to apply:** always run integration tests against a real database instance
```

**MEMORY.md index format:**
```markdown
- [Integration tests must hit real DB](feedback_testing.md) — test strategy from Q1 incident
- [User prefers terse responses](user_comms.md) — no trailing summaries
- [Auth rewrite is compliance-driven](project_auth.md) — legal constraint, not tech debt
```

Rules: ≤ 200 entries, one entry ≤ 150 chars. Lines beyond 200 are truncated at injection.
The model reads the index and fetches specific files only when relevant — not all files load
into context on every turn.

**Memory file discovery (upward walk):**
```python
def discover_memory_files(cwd: str, max_files: int = 200) -> list[Path]:
    """Walk upward from cwd collecting memory files, newest-first within each dir."""
    found = []; seen_inodes = set(); path = Path(os.path.realpath(cwd))
    while True:
        mem_dir = path / ".agent" / "memory"
        if mem_dir.is_dir():
            files = sorted(mem_dir.glob("*.md"), key=lambda f: f.stat().st_mtime, reverse=True)
            for f in files:
                inode = f.stat().st_ino
                if inode not in seen_inodes:
                    seen_inodes.add(inode); found.append(f)
                    if len(found) >= max_files: return found
        parent = path.parent
        if parent == path: break
        path = parent
    return found
```

**Memory staleness caveat:** Inject a note when memory files are >24h old so the model
doesn't treat stale information as current:
```python
def memory_freshness_text(mtime: float) -> str:
    age_hours = (time.time() - mtime) / 3600
    if age_hours < 24: return ""
    if age_hours < 168: return f"(written {int(age_hours)}h ago — verify before acting)"
    return f"(written {int(age_hours/24)}d ago — likely stale, verify)"
```

### Session Memory Extraction

After sessions with ≥ 20 messages (or at compaction time), fire a structured API call to
extract atomic facts and append them to the user/project memory files automatically:

```
extraction_categories:
  - UserPreference   (how this user likes to work)
  - ProjectFact      (codebase truths: "uses monorepo", "tests in /test/")
  - CodePattern      (idioms: "all handlers return Result<(), AppError>")
  - Decision         ("chose SQLite over Postgres for simplicity")
  - Constraint       ("never DELETE without explicit user confirmation")

trigger:
  - message_count >= 20
  - OR compact event fires
  - track last_extracted_message_id to avoid re-extraction
```

### AutoDream (Background Consolidation)

A background daemon synthesizes memories across multiple sessions into a consolidated
knowledge base. Run it as a separate low-priority process or scheduled task:

```
Trigger conditions (AND):
  - >= 24 hours since last consolidation
  - >= 5 new sessions since last consolidation
  - No concurrent consolidation (lock file check)

Process:
  1. Read all session summaries from past N sessions
  2. Fire a dedicated consolidation prompt (uses cheaper model)
  3. Extract cross-session patterns and update memory files
  4. Update consolidation_state.json with timestamp + session count
  5. Release lock

Circuit breaker:
  - If lock file is stale (>2h), remove and proceed
  - If consolidation fails 3× in 24h, skip until next day
```

### Away Summary (Re-entry Recap)

When the user returns to an idle session (no activity for > T minutes), generate a 1-3
sentence recap of the current work state using a cheaper/faster model before the user's
next prompt is processed:

```
if time_since_last_message > AWAY_THRESHOLD:
    summary = cheaper_model.complete(
        "Summarize in 1-3 sentences what was being worked on and current status."
        + context_from_last_N_messages
    )
    display_as_status_message(summary)
    # do NOT add to message history — it's informational only
```

---

## 8. RAG Pipeline Stages

Every RAG implementation must traverse the same five stages. The paper references indicate
which algorithm to apply at each stage:

```
Query
  │
  ▼ [1] QUERY TRANSFORMATION
  │     HyDE (arXiv:2212.10496): generate hypothetical answer → embed that
  │     RAG-Fusion (arXiv:2402.03367): generate N sub-queries → RRF merge
  │
  ▼ [2] RETRIEVAL
  │     Dense: embedding ANN (HNSW / Qdrant)
  │     Sparse: BM25 keyword
  │     Hybrid: BGE-M3 (arXiv:2402.03216) unified dense+sparse+multi-vector
  │
  ▼ [3] RELEVANCE FILTERING
  │     CRAG (arXiv:2401.15884): score each chunk Correct/Ambiguous/Incorrect
  │       Correct  → use chunk as-is
  │       Ambiguous → web search to augment
  │       Incorrect → discard, fallback to web search
  │
  ▼ [4] CONTEXT ASSEMBLY
  │     Rerank top-k by cross-encoder or MMR
  │     Trim to token budget
  │
  ▼ [5] GENERATION
        LLM generates answer conditioned on retrieved context
        Self-RAG (arXiv:2310.11511): emit reflection tokens
          [Retrieve] [IsRel] [IsSup] [IsUse] inline during generation
```

---

## 9. Knowledge Graph Architecture

When semantic memory requires structured relationship traversal:

```
┌──────────────────────────────────────────────────────────────────┐
│  EXTRACTION                                                        │
│  LLM triple extractor: sentence → (subject, predicate, object)   │
│  Entity resolver: coreference + deduplication                     │
├──────────────────────────────────────────────────────────────────┤
│  STORAGE                                                           │
│  Small-scale: SQLite adjacency table (entities + edges)           │
│  Production: Neo4j with Cypher                                    │
├──────────────────────────────────────────────────────────────────┤
│  RETRIEVAL — choose by query type:                                │
│                                                                    │
│  Local entity queries → HippoRAG (arXiv:2405.14831)              │
│    seed = top-k ANN of query embedding                           │
│    PPR(seed, damping=0.85) over entity graph                     │
│    return top-m entities by final PPR score                       │
│                                                                    │
│  Global thematic queries → GraphRAG (arXiv:2404.16130)           │
│    Leiden community detection on entity graph                     │
│    LLM generates community summary for each cluster              │
│    retrieve relevant community summaries by query                │
│                                                                    │
│  Path-constrained queries → PathRAG (arXiv:2502.14902)           │
│    enumerate relevant paths, score by flow capacity              │
│    prune low-flow paths before LLM context injection             │
│                                                                    │
│  Dual-level hybrid → LightRAG (arXiv:2410.05779)                 │
│    local mode: entity-level ANN                                   │
│    global mode: community summaries                               │
│    hybrid mode: both, merged                                      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 10. Event-Driven Architecture (ESAA Pattern)

Based on arXiv:2602.23193 and OpenHands (arXiv:2407.16741):

```
                     AGENT ACTION
                          │
                          ▼
              ┌───────────────────────┐
              │  EventStore           │
              │  append-only JSONL    │
              │  event_id, timestamp  │
              │  type, payload, meta  │
              └───────────┬───────────┘
                          │ publish
                          ▼
              ┌───────────────────────┐
              │  EventBus             │
              │  topic routing        │
              └───────────┬───────────┘
                          │
           ┌──────────────┼──────────────┐
           ▼              ▼              ▼
      ┌─────────┐   ┌──────────┐   ┌─────────┐
      │MaterializedView│HookRunner│ │Audit Log│
      │(CQRS read) │ │(pre/post)│   │(append) │
      └─────────┘   └──────────┘   └─────────┘
```

**Event types to implement in all three languages:**

| Event | When emitted |
|-------|-------------|
| `AgentStarted` | Session begins |
| `TurnStarted` | LLM call begins |
| `TextDelta` | Streaming token received |
| `ToolCallRequested` | LLM emits tool_use block |
| `PermissionChecked` | Before tool execution |
| `ToolCallStarted` | Tool execution begins |
| `ToolCallCompleted` | Tool execution ends with result |
| `ToolCallFailed` | Tool execution raises error |
| `TurnCompleted` | LLM call ends (end_turn) |
| `CompactionTriggered` | History summarization begins |
| `CompactionCompleted` | History replaced with summary |
| `AgentStopped` | Session ends |

**Hook runner contract** (runs synchronously on `ToolCallRequested`):

```
pre_hook(event: ToolCallRequested) → HookDecision
  ALLOW   → proceed with permission check
  DENY    → abort tool call, return error to LLM
  ASK     → escalate to user regardless of policy
  REPLACE → substitute a different tool call

post_hook(event: ToolCallCompleted) → void
  (side-effects only: logging, metrics, notifications)
```

---

## 11. Multi-Agent Topologies

Four proven topologies from the literature. Choose based on task structure:

### 11.1 Supervisor (Centralized)

```
User ─► Supervisor Agent
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
 Worker A  Worker B  Worker C
```

**Use when:** Tasks have a clear decomposition; coordinator needs to route based on output.
**Paper:** AutoGen (arXiv:2308.08155), MetaGPT (arXiv:2308.00352)
**Failure mode:** Single point of failure; bottleneck on supervisor context.

### 11.2 Swarm (Peer Handoff)

```
 Agent A ──handoff──► Agent B ──handoff──► Agent C
   ▲                                           │
   └───────────────────────────────────────────┘
```

**Use when:** Tasks flow through specializations in sequence; no fixed routing.
**Paper:** CAMEL (arXiv:2303.17760)
**Implementation:** `handoff_tool(agent_id)` transfers control with shared state.

### 11.3 Parallel Map-Reduce

```
Coordinator
    │ split
    ├─── Sub-task 1 ──► Worker 1
    ├─── Sub-task 2 ──► Worker 2
    └─── Sub-task N ──► Worker N
    │ gather + reduce
    ▼
 Synthesized result
```

**Use when:** Independent subtasks of equal structure (e.g., audit N tables, profile N files).
**Efficiency:** N× throughput on wall-clock time, identical cost.
**Language note:** `cobalt::gather` (C++), `coroutineScope { async { } }` (Kotlin),
`asyncio.gather` / LangGraph BSP (Python).

### 11.4 SOP Pipeline (Sequential)

```
PM Agent ──► Architect Agent ──► Coder Agent ──► Tester Agent
   (PRD)         (Design)           (Code)         (Verified)
```

**Use when:** Output of each stage is the input of the next; strict artifact dependencies.
**Paper:** MetaGPT (arXiv:2308.00352)
**Key feature:** Artifact validation gates: the next agent only runs when the previous
artifact passes a structured check (e.g., code must compile before test agent runs).

### 11.5 Manager-Executor

```
User ──► Manager Agent (large model: Opus, GPT-4o)
              │ plan + delegate via AgentTool
    ┌─────────┼─────────┐
    ▼         ▼         ▼
 Executor  Executor  Executor   (smaller model: Sonnet, GPT-4o-mini)
 (worktree) (worktree) (worktree)
    │         │         │
    └─────────┴─────────┘
         shared Arc<CostTracker>
         total_budget_usd enforced across all executors
```

**Use when:** Tasks benefit from separation between strategic reasoning and implementation;
cost optimization by routing planning to expensive models and execution to cheaper ones.

**Key design decisions:**
- The manager model is explicitly forbidden from using `Bash` directly (enforced by
  tool filtering) — it can only delegate. This prevents the manager from bypassing the
  executor layer.
- Each executor runs in a `git worktree` (isolated copy of the repo) to prevent
  file-write conflicts between concurrent executors.
- A shared `CostTracker` (with atomic counter) enforces a total USD budget cap across
  all executors, not per-executor. When the budget is exhausted, all executors stop.
- `ScratchpadGate` — if you want the manager to reason in a scratchpad before
  delegating, block `Write`/`Edit` tools with a lock until a specific unlock phrase
  appears in the model's output stream. This ensures the manager completes its
  reasoning before any file is touched.
- 6 built-in presets for common workloads: (code-review, bug-fix, feature-dev,
  security-audit, data-analysis, doc-generation). Presets encode the model pair,
  budget split, executor count, and isolation policy.

**Budget split policy options:**
```
Equal          → budget / num_executors per executor
Proportional   → based on task complexity estimate
Manager-first  → fixed manager budget, remainder split among executors
Unlimited-exec → cap only the manager; executors run until done
```

### 11.6 Coordinator Mode (Tool Filtering)

When an orchestrator agent runs in coordinator mode, apply two tool-filtering rules:

1. **Strip coordinator-only tools from workers.** Workers must not have `AgentTool`,
   `TeamCreate`, `SendMessage`, `SyntheticOutput` — otherwise a worker could spawn
   unauthorized sub-agents.

2. **Forbid bash on the coordinator.** The coordinator reasons and delegates; it must not
   execute shell commands directly. Enforced by removing `BashTool` from the coordinator's
   tool set at session initialization.

```python
# coordinator tool set
coordinator_tools = all_tools - {BashTool, PowerShellTool}

# worker tool set
worker_tools = all_tools - {AgentTool, TeamCreateTool, TeamDeleteTool,
                             SendMessageTool, SyntheticOutputTool, CronCreateTool}
```

### 11.7 Bidirectional TUI ↔ Agent Communication (CommandQueue)

In interactive sessions the UI and the agent run on separate threads/tasks. A shared
command queue enables the UI to inject messages into the agent's next turn while a tool
is still executing — without interrupting the current stream:

```
TUI thread                       Agent task
    │                                │
    │ user types during tool run     │
    │                                │
    ├── push(QueuedCommand) ────────►│ drain_queue() at start of next turn
    │   priority: Normal/High        │
    │                                │ inject as user message before API call
```

High-priority commands (e.g., `/stop`, `/compact`) are prepended to the message history;
normal-priority commands are appended. This avoids requiring the user to wait for the
current tool to finish before their input is acknowledged.

---

## 12. Advanced Agent Loop Patterns

### 12.1 Concurrent Tool Execution

When the model emits multiple `tool_use` blocks in a single response, run them in
parallel rather than sequentially. This is safe only when tools are read-only or
independently scoped (no shared write targets).

```
# Sequential (simpler, always safe):
for tool_call in tool_calls:
    result = execute(tool_call)
    results.append(result)

# Concurrent (up to 10× faster for read-heavy turns):
results = parallel_map(execute, tool_calls, max_concurrency=10)

# Rules:
CONCURRENT_SAFE = {read_file, glob, grep, web_fetch, read_mcp_resource}
SEQUENTIAL_ONLY  = {write_file, edit_file, bash, mcp_write_*}

# Implementation by language:
# Python:     asyncio.gather(*[execute(t) for t in tool_calls])  (+ semaphore(10))
# TypeScript: pMap(toolCalls, execute, {concurrency: 10})
# Rust:       tokio::task::JoinSet (spawn all, await all)
# Go:         errgroup.Group with semaphore channel(10)
# Kotlin:     coroutineScope { calls.map { async { execute(it) } }.awaitAll() }
```

**Streaming tool executor:** Begin executing read-only tools as soon as their
`tool_use` block is parsed from the stream — before the full model response
completes. This eliminates latency for file-read operations in long responses.

### 12.2 Tool Output Masking

When a session has many turns, large tool results in message history waste context tokens.
Store them externally and replace with placeholders in the context sent to the LLM:

```
MASKING_THRESHOLD = 10_000  # chars

# On each LLM call, before building context:
for i, message in enumerate(history):
    for block in message.content:
        if block.type == "tool_result" and len(block.content) > MASKING_THRESHOLD:
            key = f"masked_{message.id}_{block.tool_use_id}"
            large_result_store[key] = block.content
            block.content = f"[output stored at {key} — {len(block.content)} chars]"

# The actual output is still available for the agent if it needs to re-read it,
# but the LLM sees only the placeholder in its context window.
```

**Complementary — Tool Distillation:** For very long tool outputs, run an LLM call to
compress the output to a summary before injecting into history. Useful for web_fetch
results and large bash stdout.

### 12.3 Model Routing Strategies

A composable routing pipeline applies strategies in precedence order:

```
1. overrideStrategy       → explicit --model CLI flag wins
2. approvalModeStrategy   → permission mode (e.g. yolo) → cheaper fast model
3. localClassifierStrategy→ local classifier model (no API call)
4. cloudClassifierStrategy→ cloud-based classifier
5. numericalClassifier    → token count → simple/complex routing
6. fallbackStrategy       → on quota/overload → cheaper fallback
7. defaultStrategy        → configured model for session

Routing examples:
  Short question (< 100 tokens)  → haiku / flash (cheap + fast)
  Complex multi-step task        → opus / pro (capable)
  YOLO mode (--yolo)             → sonnet (no interactive confirms)
  Overloaded / 529               → fallback model silently
```

**Minimum viable routing** (most agents only need 2 strategies):
```
strategy = overrideStrategy ?? fallbackStrategy ?? defaultStrategy
```

### 12.4 Loop Detection

Without loop detection, an agent can get stuck calling the same tool repeatedly forever.

**Heuristic approach** (fast, no LLM cost):
```python
class LoopDetector:
    def __init__(self, threshold=5):
        self.threshold = threshold
        self.fingerprints = []  # last N turn fingerprints

    def record(self, tool_calls):
        fp = frozenset(
            (t.name, json.dumps(t.input, sort_keys=True))
            for t in tool_calls
        )
        self.fingerprints.append(fp)
        if len(self.fingerprints) > self.threshold * 2:
            self.fingerprints.pop(0)

    def is_looping(self) -> bool:
        if len(self.fingerprints) < self.threshold:
            return False
        recent = self.fingerprints[-self.threshold:]
        return len(set(recent)) == 1  # all identical
```

**LLM-based approach** (more accurate, kicks in after 30 turns):
```
if turn_count > 30 and turn_count % check_interval in (5..15):
    confidence = cheap_model.classify(
        "Is this agent looping? Review the last 10 tool calls.",
        recent_tool_calls
    )
    if confidence >= 0.9:
        raise LoopDetectedError("Agent appears to be looping")
```

**On loop detection:** inject a meta-message into the conversation explaining the
loop was detected, then either abort or ask the user how to proceed.

### 12.5 IDE Context Injection

When integrated with an IDE via a companion extension:

```
First turn:
  IDE → agent: full context JSON {
    open_files: [{path, content, language}],
    active_file: {path, cursor: {line, col}},
    selection: {text, range}
  }

Subsequent turns (delta-only, saves tokens):
  IDE → agent: delta context {
    changed_files: [{path, new_content}],
    active_file: {path, cursor: {line, col}},
    selection: {text, range}  # only if changed
  }
```

This pattern gives the agent real-time IDE state without repeating the full file
contents on every turn. Implement via a WebSocket or stdin/stdout IPC channel between
the IDE extension and the agent process.

### 12.6 Session Branching (Conversation Tree)

For experimentation and "undo" scenarios, implement full tree branching:

```
Session structure:
  session_id: uuid
  parent_id: uuid | null     # null for root session
  branch_at_message: usize   # which message index this branch diverges from
  name: string               # user-given branch name

Storage:
  Sessions share the same project directory.
  Each branch is a separate JSONL file with its own session_id.
  The parent's messages up to branch_at_message are loaded, then the
  branch's messages are appended.

Operations:
  /fork <name>          → copy current session up to now, new session_id
  Ctrl+B (UI)          → interactive branch selector
  --session <id>        → resume a specific branch
  
SQLite index (optional, for fast browsing):
  CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    parent_id TEXT,
    branch_at_message INTEGER,
    name TEXT,
    created_at INTEGER,
    project_hash TEXT
  );
```

---

## 13. Self-Improvement Loop

```
                    ┌────────────────────────────────────────┐
                    │           IMPROVEMENT CYCLE             │
                    │                                         │
                    │  1. Execute task                        │
                    │       ↓                                 │
                    │  2. Evaluate outcome                    │
                    │     QualityGate: LLM-as-judge           │
                    │     score = f(task_success, efficiency) │
                    │       ↓                                 │
                    │  3. If score < threshold:               │
                    │     Reflexion: verbal reflection        │
                    │     SELF-REFINE: generate critique      │
                    │     CRITIC: tool-verify claims          │
                    │       ↓                                 │
                    │  4. Store trajectory + reflection       │
                    │     ExpeL experience pool (vector DB)   │
                    │       ↓                                 │
                    │  5. Distill insights across pool        │
                    │     LLM: "What patterns lead to success?"│
                    │       ↓                                 │
                    │  6. Update SkillLibrary (Voyager)       │
                    │     Reusable subtask procedures         │
                    │       ↓                                 │
                    │  7. Next task benefits from:            │
                    │     - Retrieved similar trajectories    │
                    │     - Distilled success patterns        │
                    │     - Reusable skills                   │
                    └────────────────────────────────────────┘
```

**Paper mapping:**

| Step | Paper | arXiv ID |
|------|-------|----------|
| Verbal reflection | Reflexion | 2303.11366 |
| Generate-critique-refine | SELF-REFINE | selfrefine.info |
| Tool-verified correction | CRITIC | 2310.06825 |
| Experience pool storage | ExpeL | 2308.10144 |
| Skill library | Voyager | (2305.16291) |
| LLM-as-judge evaluation | Constitutional AI / alignment work | — |

---

## 14. Session Compaction

The ToolUse/ToolResult boundary guard is the most critical correctness invariant in session
management. Getting this wrong causes 400 errors from API providers:

```
SAFE compaction boundary:

  messages[0..N-4]:  summarize these
  messages[N-3..N]:  preserve verbatim (last 4)

  BUT: if messages[N-4] is a ToolUse, walk back until the
  boundary starts BEFORE that ToolUse:

  messages[N-5] = AssistantMessage (text)    ← start preserve here
  messages[N-4] = ToolUseBlock               ← must be preserved
  messages[N-3] = ToolResultBlock            ← must be preserved with its ToolUse
  messages[N-2] = AssistantMessage
  messages[N-1] = UserMessage

UNSAFE:
  ✗ Summarize up to and including a ToolUse
  ✗ Preserve a ToolResult whose ToolUse was compacted away
  ✗ Split a multi-tool assistant turn (one tool summarized, another preserved)
```

**Three-Strategy Compaction:**

| Token usage | Strategy | Description |
|-------------|----------|-------------|
| < 75% | None | Normal operation |
| 75–89% | **MicroCompact** | Compact only the oldest messages; preserve last N verbatim. No full API call needed — just trim. |
| ≥ 90% | **Full Compact** | Summarize all but the last 10 messages via a dedicated non-agentic API call. Replace head with `<compact-summary>` synthetic user message. |
| Overflow | **ContextCollapse** | Strip individual content blocks (tool result text, large attachments) from individual messages in-place, keeping metadata. Last-resort before hard failure. |

**Circuit breaker:** after `MAX_CONSECUTIVE_FAILURES = 3` compaction failures in a session,
disable the compactor entirely for that session. Log a warning and let the session run until
the model's native context limit is hit. Prevents infinite retry loops from burning budget.

**Tool-result budget:** independently of compaction, evict the *text content* of oldest tool
results when the combined character count of all tool results exceeds a cap (e.g., 50 000
chars). The tool call metadata (name, id, timestamps) is retained; only the result body is
replaced with `"[result truncated — {N} chars]"`. This prevents a single large bash output
from consuming the entire context before compaction can trigger.

**Compaction trigger strategy:**

| Token usage | Action |
|-------------|--------|
| < 75% | Normal operation |
| 75–89% | MicroCompact (oldest messages only) |
| ≥ 90% | Full Compact (dedicated API call) |
| Overflow | ContextCollapse (in-place body stripping) |
| 3 failures | Disable compactor for session |

**Token warning thresholds (for UI):**
- 80% → yellow warning indicator
- 95% → red critical indicator + prompt user that compaction is imminent

**Compaction prompt structure (universal):**

```
"This {domain} session is being continued from a prior context.
The summary below covers the earlier portion.

Summary:
- Scope: {N} earlier messages compacted (user={u}, assistant={a}, tool={t})
- Tools used: {tool_list}
- Recent user requests: {bullet_list}
- Key artifacts / files: {artifact_list}
- Current work in progress: {wip}

Recent messages are preserved verbatim below.
Continue from where it left off without re-summarizing."
```

---

## 15. Configuration Hierarchy

All three implementations use the same precedence order (later sources win):

```
[1] Compiled defaults (lowest priority)
[2] User-level config  (~/.{agent}/settings.json)
[3] Project config     ({cwd}/.{agent}/settings.json)
[4] Machine-local      ({cwd}/.{agent}/settings.local.json)  [gitignored]
[5] CLI flags          (highest priority)
```

**Required config fields (language-agnostic schema):**

```json
{
  "model": "claude-opus-4-7",
  "maxTokens": 16384,
  "permissionMode": "workspace-write",
  "fallbackModel": "claude-haiku-4-5",
  "maxBudgetUsd": null,
  "maxTurns": 10,
  "permissions": {
    "allow": [],
    "deny": [],
    "ask": []
  },
  "tools": {
    "allowed": [],
    "disallowed": [],
    "toolResultBudgetChars": 50000
  },
  "mcp": {
    "servers": {}
  },
  "hooks": {
    "PreToolUse": [],
    "PostToolUse": [],
    "PostModelTurn": [],
    "Stop": []
  },
  "compaction": {
    "microCompactThresholdPercent": 75,
    "fullCompactThresholdPercent": 90,
    "tailMessages": 10,
    "maxConsecutiveFailures": 3
  },
  "providers": {
    "anthropic": { "apiKey": "", "baseUrl": "" },
    "openai": { "apiKey": "", "baseUrl": "" },
    "ollama": { "baseUrl": "http://localhost:11434" }
  },
  "managedAgents": {
    "enabled": false,
    "managerModel": "claude-opus-4-7",
    "executorModel": "claude-sonnet-4-6",
    "executorMaxTurns": 8,
    "maxConcurrentExecutors": 4,
    "executorIsolation": true,
    "totalBudgetUsd": null
  },
  "memory": {
    "autoDream": {
      "enabled": true,
      "minHoursBetween": 24,
      "minNewSessions": 5
    },
    "sessionExtraction": {
      "enabled": true,
      "triggerMessageCount": 20
    },
    "awaySummary": {
      "enabled": true,
      "idleThresholdMinutes": 30,
      "summaryModel": "claude-haiku-4-5"
    }
  }
}
```

---

## 16. Cross-Language Implementation Map

The same logical component in each language:

| Component | C++23 | Kotlin/Koog | Python/LangGraph |
|-----------|-------|-------------|-----------------|
| Agent loop | `run_agent()` coroutine | Strategy graph + `AIAgent.run()` | `StateGraph.compile().stream()` |
| State | `AgentContext` struct | `AgentSession` data class | `TypedDict` + reducers |
| Tool registry | `ToolRegistry` flat_map | `ToolRegistry` + `@Tool` | `ToolNode` + `@tool` |
| Permissions | `PermissionPolicy` | `PermissionPolicy` | custom node + interrupt |
| Pre-hook | `HookRunner::pre()` | `ToolExecutionStrategy.beforeTool()` | LangGraph `NodeInterruptedException` |
| Memory (short) | In-context messages | `ChatMemory` interface | `InMemorySaver` checkpointer |
| Memory (long) | `USearch` HNSW | `QdrantLongTermMemory` | `InMemoryStore` / `AsyncPostgresSaver` |
| Memory (file) | `memdir/` markdown files | `memdir/` markdown files | `memdir/` markdown files |
| Session extraction | post-session LLM call | post-session LLM call | `@traceable` + post-run evaluator |
| Auto-dream | background `tokio::spawn` | background coroutine | background `asyncio.create_task` |
| Away summary | cheaper model on re-entry | cheaper model on re-entry | cheaper model on re-entry |
| Knowledge graph | SQLite adjacency | Neo4j bolt | networkx + Neo4j |
| Event store | Append-only JSONL | Append-only JSONL | JSONL + Redis Streams |
| Compaction | `compact_session()` 3-strategy | `compactIfNeeded()` 3-strategy | `should_compact` node 3-strategy |
| Tool-result budget | char-cap eviction | char-cap eviction | char-cap eviction |
| Fallback model | `fallback_model` config | `fallbackModel` config | `fallback_model` config |
| Multi-agent | `SupervisorAgent` | A2A + `SupervisorAgent` | `create_supervisor` |
| Manager-Executor | `SupervisorAgent` + worktree | `SupervisorAgent` + worktree | `create_supervisor` + interrupt |
| Provider registry | `ProviderRegistry` trait | `LlmProvider` interface | `BaseChatModel` + registry |
| Observability | OpenTelemetry spans | Langfuse + Tracy | LangSmith `@traceable` |
| Session persist | `SessionManager` JSONL | `PostgresJdbcPersistenceStorageProvider` | `AsyncPostgresSaver` |
| Config | JSON + env + CLI | `ClawConfig` Kotlin data class | Pydantic `Settings` |
| Streaming | `std::generator<Event>` | `Flow<AgentEvent>` | `astream()` / `astream_events()` |
| Stream resilience | watchdog + retry + fallback | watchdog + retry + fallback | watchdog + retry + fallback |

### 16.1 Cross-Language Decision Guide

Choose based on your deployment context and team:

| Requirement | C++23 | Kotlin/Koog | Python/LangGraph |
|-------------|-------|-------------|-----------------|
| **Latency-critical inference** | ✅ Best | 🟡 Good | 🟡 Good |
| **Local LLM (llama.cpp)** | ✅ Native | 🟡 Via Ollama | 🟡 Via Ollama |
| **Rapid prototyping** | ❌ Slow build | 🟡 Moderate | ✅ Best |
| **Multiplatform (Android/iOS)** | ❌ No | ✅ KMP native | ❌ No |
| **Spring Boot integration** | ❌ No | ✅ Official starters | ❌ No |
| **LangSmith observability** | ❌ Manual OTEL | 🟡 Via OTEL | ✅ Native |
| **RAG ecosystem breadth** | 🟡 Manual | 🟡 LangChain4j | ✅ Best |
| **Multi-agent A2A** | 🟡 Custom | ✅ Built-in | 🟡 Via supervisor |
| **Async/coroutines** | 🟡 Cobalt | ✅ Native suspend | ✅ asyncio |
| **Memory footprint** | ✅ Minimal | 🟡 JVM overhead | 🟡 Moderate |
| **Enterprise Java ecosystem** | ❌ No | ✅ Full JVM | ❌ No |
| **Production vector search** | ✅ USearch | ✅ Qdrant Java | ✅ Qdrant/pgvector |
| **Event sourcing (ESAA)** | ✅ Yes | ✅ Yes | ✅ Yes |
| **LangGraph Platform deploy** | ❌ No | ❌ No | ✅ Native |

**Pick C++23** when: embedded/edge inference, latency < 50 ms, existing C++ codebase.

**Pick Kotlin/Koog** when: Android/mobile, Spring Boot enterprise, JVM ecosystem reuse,
or compile-time-validated strategy graphs.

**Pick Python/LangGraph** when: fastest time-to-production, richest RAG/tool ecosystem,
LangSmith tracing from day one, LangGraph Platform deployment, or ML/data science teams.

---

## 17. Infrastructure Stack

All three implementations share the same external services:

```yaml
# docker-compose.yml (universal — works for all language implementations)
services:
  postgres:
    image: pgvector/pgvector:pg16   # pgvector extension included
    environment:
      POSTGRES_DB: agentdb
      POSTGRES_USER: agent
      POSTGRES_PASSWORD: secret
    ports: ["5432:5432"]

  qdrant:        # Vector DB for episodic + semantic memory
    image: qdrant/qdrant:v1.9.0
    ports: ["6333:6333", "6334:6334"]
    volumes: ["qdrant_data:/qdrant/storage"]

  redis:         # Event bus for durable pub/sub
    image: redis:7-alpine
    ports: ["6379:6379"]

  neo4j:         # Graph DB for knowledge graph
    image: neo4j:5.19
    environment:
      NEO4J_AUTH: neo4j/secret
    ports: ["7474:7474", "7687:7687"]

  ollama:        # Local LLM serving (llama3, mistral, codestral, etc.)
    image: ollama/ollama:latest
    ports: ["11434:11434"]
    volumes: ["ollama_data:/root/.ollama"]

  langfuse:      # Observability (Python primary, all via OTEL)
    image: langfuse/langfuse:2
    ports: ["3000:3000"]

  jaeger:        # Distributed tracing (C++ / Kotlin via OpenTelemetry)
    image: jaegertracing/all-in-one:1.56
    ports: ["16686:16686", "4317:4317"]

volumes:
  qdrant_data:
  ollama_data:
```

**Minimum viable stack** (single-machine development): Qdrant + Postgres (pgvector) only.
Everything else can use SQLite and in-process alternatives.

---

## 18. arXiv Paper Reference

| Paper | arXiv ID | Year | Key Contribution |
|-------|----------|------|-----------------|
| **ReAct** | 2210.03629 | 2022 | Thought→Action→Observation loop; +34% ALFWorld vs chain-of-thought |
| **Reflexion** | 2303.11366 | 2023 | Verbal RL via episodic reflection buffer; 91% HumanEval pass@1 |
| **OpenHands** | 2407.16741 | 2024 | Event-sourced state machine, typed tool system, sandboxed runtime |
| **MemGPT** | 2310.08560 | 2023 | 3-tier memory (in-context / recall / archival); unbounded conversation |
| **CoALA** | 2309.02427 | 2023 | Working / episodic / semantic / procedural memory taxonomy |
| **A-MEM** | 2502.12110 | 2025 | Zettelkasten note graph + Ebbinghaus decay scheduling; 2× MemGPT |
| **Mem0** | 2504.19413 | 2025 | Extract → deduplicate → store → retrieve; 90% token reduction |
| **LongMemEval** | 2410.10813 | 2024 | Decompose sessions to atomic facts; benchmark for long-term recall |
| **GraphRAG** | 2404.16130 | 2024 | Leiden community detection + LLM community summaries |
| **HippoRAG** | 2405.14831 | 2024 | KG + Personalized PageRank retrieval; +20% multi-hop QA |
| **LightRAG** | 2410.05779 | 2024 | Dual-level retrieval (local entity + global community) |
| **PathRAG** | 2502.14902 | 2025 | Flow-based path pruning on knowledge graphs |
| **Self-RAG** | 2310.11511 | 2023 | Reflection tokens [Retrieve][IsRel][IsSup][IsUse] |
| **CRAG** | 2401.15884 | 2024 | Confidence-scored retrieval: Correct / Ambiguous / Incorrect |
| **HyDE** | 2212.10496 | 2022 | Hypothetical document embedding improves retrieval |
| **RAG-Fusion** | 2402.03367 | 2024 | Multi-query + Reciprocal Rank Fusion |
| **BGE-M3** | 2402.03216 | 2024 | Unified dense + sparse + multi-vector; 100+ languages |
| **ESAA** | 2602.23193 | 2026 | Event Sourcing + CQRS for agents; append-only JSONL event store |
| **ExpeL** | 2308.10144 | 2023 | Experience pool (vector DB) + LLM-distilled insights |
| **SELF-REFINE** | selfrefine.info | 2023 | Generate → critique → refine (capped iterations) |
| **CRITIC** | 2310.06825 | 2023 | Tool-integrated critique loop for self-correction |
| **AutoGen** | 2308.08155 | 2023 | Actor model — typed message handlers, pub-sub topics |
| **MetaGPT** | 2308.00352 | 2023 | SOP pipeline: PRD → Architecture → Code → Tests |
| **CAMEL** | 2303.17760 | 2023 | Role-playing agent communication; peer handoff pattern |
| **StreamingLLM** | 2309.17453 | 2023 | Attention sink tokens + sliding KV cache; 22.2× throughput |

---

## 19. Planning Mode

Planning mode separates **safe exploration and plan generation** from **write-capable execution**, with an explicit user-approval gate between phases. It is the highest-leverage UX feature for reducing irreversible mistakes.

---

### 19.1 Core Concept

Without planning mode, the agent reads context and immediately starts making changes. With planning mode:

```
Plan Phase (read-only)          Approval Gate          Execute Phase (write-enabled)
──────────────────────    ──────────────────────    ──────────────────────────────────
Read files                 User reviews plan         Write files
Explore codebase      →    Edits plan if needed  →   Run commands
Generate plan file         Approves / rejects         Verify changes
(no writes)                (with feedback)            (read-only)
```

The key invariant: **no write operations occur during plan phase**. The permission system enforces this, not prompts alone.

---

### 19.2 Two Planning Mechanisms

#### A. Tool-Based Plan Mode (rich UX, recommended for interactive agents)

The agent has two tools: `EnterPlanMode` and `ExitPlanMode`.

```
User: "Refactor the auth module"
  │
  ▼
Agent calls EnterPlanMode()
  → permission_mode switches to "plan" (read-only enforcement)
  → exploration sub-agents spin up (parallel codebase reads)
  │
  ▼
Plan Phase loop (read-only tools only)
  → read_file, glob_search, grep_search allowed
  → write_file, bash(write), edit_file → DENIED
  → agent writes plan to plan file (plans directory only exception)
  │
  ▼
Agent calls ExitPlanMode(plan_filename="auth-refactor.md")
  → triggers user approval dialog
  → user reads plan, optionally edits, approves or rejects with feedback
  │
  ┌─── rejected ───► agent revises plan, returns to Plan Phase
  │
  ▼ approved
Execute Phase
  → permission_mode restored to "default"/"yolo"
  → write operations now allowed
  → agent implements following the plan
```

**Plan file schema** (structured markdown):
```markdown
# Plan: Auth Module Refactor

## Phase 1 — Exploration Findings
- `src/auth/middleware.ts` (lines 45–120): session token stored in-memory, not encrypted
- `src/auth/types.ts` (lines 12–34): AuthToken interface missing expiry field

## Phase 2 — Design Approach
Replace in-memory store with Redis. Add expiry to AuthToken. Update middleware.

## Phase 3 — Implementation Steps
- [ ] Add `expiry: Date` to `AuthToken` in `src/auth/types.ts:12`
- [ ] Replace `Map<string, Token>` with `RedisClient` in `src/auth/store.ts`
- [ ] Update `validateToken()` in `src/auth/middleware.ts:67` to check expiry
- [ ] Run: `npm test -- src/auth/`

## Phase 4 — Verification
Single command: `npm test -- src/auth/ && npm run lint`
```

**Tool declarations:**

```python
PLAN_TOOLS = [
    {
        "name": "enter_plan_mode",
        "description": "Switch to planning mode. All write operations will be blocked until the plan is approved. Use when the task is complex enough to warrant exploration before changes.",
        "input_schema": {
            "type": "object",
            "properties": {
                "reason": {"type": "string", "description": "Why planning mode is needed"}
            },
            "required": ["reason"]
        }
    },
    {
        "name": "exit_plan_mode",
        "description": "Submit the plan for user approval. Blocks until user approves or rejects. On rejection, returns feedback and stays in plan mode.",
        "input_schema": {
            "type": "object",
            "properties": {
                "plan_filename": {"type": "string", "description": "Path to the plan file"},
                "summary": {"type": "string", "description": "One-paragraph summary of the plan"}
            },
            "required": ["plan_filename", "summary"]
        }
    }
]
```

#### B. Extended Thinking (lightweight, no separate tools needed)

Enable the API's extended thinking feature. The model reasons internally before emitting tool calls. No permission switching, no plan file, no approval gate — but the reasoning is hidden (or optionally shown).

```python
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=16000,
    thinking={"type": "enabled", "budget_tokens": 10000},  # reserve tokens for thinking
    tools=tools,
    messages=messages
)

# Response contains interleaved thinking and tool_use blocks:
# [ThinkingBlock(thinking="Let me analyze..."), ToolUseBlock(name="read_file",...), ...]

for block in response.content:
    if block.type == "thinking":
        # Optionally display: print(f"[thinking] {block.thinking}")
        pass  # do not inject back into messages — API handles it
    elif block.type == "tool_use":
        result = execute_tool(block.name, block.input)
        tool_results.append({"type": "tool_result", "tool_use_id": block.id, "content": result})
```

**Thinking budget guidelines:**
```
Light reasoning (routing, classification): 1,000–2,000 tokens
Standard tasks (code change, debugging): 5,000–8,000 tokens
Complex tasks (architecture, security audit): 10,000–16,000 tokens

Rule: budget_tokens + max_tokens must not exceed model context limit.
Extended thinking requires min budget_tokens = 1,024.
```

---

### 19.3 Permission Enforcement During Plan Phase

The permission system must enforce read-only during plan mode — prompt instructions alone are not sufficient.

```python
PLAN_MODE_ALLOWED = {"read_file", "glob_search", "grep_search", "list_dir", "web_fetch"}
PLAN_MODE_WRITE_EXCEPTION = {"write_file"}  # only to plans directory

def check_permission_plan_mode(tool: str, inp: dict, plans_dir: str) -> tuple[bool, str]:
    if tool in PLAN_MODE_ALLOWED:
        return True, ""
    if tool == "write_file":
        path = inp.get("path", "")
        if os.path.realpath(path).startswith(os.path.realpath(plans_dir)):
            return True, ""
        return False, "Write blocked in plan mode: only plans directory is writable"
    return False, f"Tool '{tool}' is not allowed in plan mode"
```

---

### 19.4 Multi-Agent Plan Exploration

For large codebases, spawn multiple read-only sub-agents in parallel during plan phase to gather context faster:

```
EnterPlanMode()
     │
     ├── Explore Agent 1: "find all auth-related files"
     ├── Explore Agent 2: "find all tests for auth module"  
     └── Explore Agent 3: "check git log for recent auth changes"
     │
     ▼ (all three complete)
Plan Agent: synthesizes findings → writes plan file
     │
ExitPlanMode(plan_filename="...", summary="...")
```

**Sub-agent count guidelines:**
- 1–3 explore agents for typical codebases
- 1–3 plan agents for complex tasks requiring multiple design options
- Cap at 3 of each to avoid context explosion in the synthesis step

---

### 19.5 Task/Todo Tracking Inside Execute Phase

After plan approval, convert the plan's step list into a tracked task list. The model updates tasks as it works, giving the user live progress visibility.

```python
TASK_SCHEMA = {
    "id": str,           # UUID
    "title": str,        # human-readable description
    "status": Literal["pending", "in_progress", "completed", "failed", "skipped"],
    "priority": Literal["low", "medium", "high", "critical"],
    "created_at": str,   # ISO timestamp
    "completed_at": str, # ISO timestamp or None
}
```

**TodoWrite tool** (model calls this to update its task list):
```python
{
    "name": "todo_write",
    "description": "Update your task list. Call after completing each step to show progress.",
    "input_schema": {
        "type": "object",
        "properties": {
            "todos": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "id": {"type": "string"},
                        "content": {"type": "string"},
                        "status": {"type": "string", "enum": ["pending","in_progress","completed","failed"]},
                        "priority": {"type": "string", "enum": ["low","medium","high","critical"]}
                    },
                    "required": ["id", "content", "status", "priority"]
                }
            }
        },
        "required": ["todos"]
    }
}
```

---

### 19.6 Planning Mode Decision Matrix

| Scenario | Recommended approach |
|----------|---------------------|
| Simple one-file change | No plan mode — direct execution |
| Multi-file refactor (>3 files) | Tool-based plan mode |
| Architecture change or new feature | Tool-based plan mode + parallel explore agents |
| Automated pipeline (no human in loop) | Extended thinking only |
| Security/compliance audit | Extended thinking + plan mode for remediation steps |
| Domain agent with known safe ops | Disable plan mode; use permission rules instead |

---

### 19.7 ScratchpadGate (Reasoning Before Delegation)

In Manager-Executor topologies (see §11.5), block the manager from touching files until it has finished reasoning. Implement by watching the output stream for an unlock phrase:

```python
class ScratchpadGate:
    """Block write tools until model emits the unlock phrase in its text stream."""
    
    UNLOCK_PHRASE = "--- BEGIN EXECUTION ---"
    
    def __init__(self):
        self._unlocked = False
        self._buffer = ""
    
    def feed(self, text_delta: str):
        self._buffer += text_delta
        if self.UNLOCK_PHRASE in self._buffer:
            self._unlocked = True
    
    def is_write_allowed(self) -> bool:
        return self._unlocked
    
    def check(self, tool_name: str, write_tools: set) -> tuple[bool, str]:
        if tool_name not in write_tools:
            return True, ""
        if not self._unlocked:
            return False, "ScratchpadGate: complete reasoning before executing (emit '--- BEGIN EXECUTION ---')"
        return True, ""
```

System prompt instruction to pair with the gate:
```
Before modifying any files, write your complete plan in plain text.
When you have finished planning and are ready to start making changes,
emit exactly: --- BEGIN EXECUTION ---
Only after that line will write operations be permitted.
```

---

## Reading Order

**First implementation (start here):**
1. This document — Sections 1–5 (runtime model, provider registry, layers, prompt, permissions)
2. `00-research-foundation.md` (paper details)
3. Language `00-index.md` (quick-start)
4. Language `02-core-agent-loop.md` (turn loop)
5. Language `03-tool-system.md` (tools + permissions)
6. Language `09-session-persistence.md` (compaction — implement all three strategies)

**Adding intelligence (after basic loop works):**
7. This document — Section 6 (memory: MemGPT + memdir + session extraction + AutoDream)
8. Language `04-memory-and-rag.md`
9. Language `05-knowledge-graph.md`
10. Language `08-self-improvement.md`

**Multi-provider support:**
11. This document — Section 2 (provider registry + stream resilience)

**Multi-agent systems:**
12. This document — Sections 10.1–10.7 (all topologies including Manager-Executor)
13. Language `07-multiagent.md`
14. Language `06-event-driven.md`

**Planning mode:**
15. This document — Section 19 (tool-based plan mode, extended thinking, ScratchpadGate)

**Hooks, sessions, cost:**
16. This document — Sections 20, 21, 22 (hooks full reference, session management, cost tracking)

**IDE integration & distributed agents:**
17. This document — Sections 23, 24 (IDE integration, A2A distributed agents)

**LSP, plugins, SDK, MCP:**
18. This document — Sections 25–28 (LSP, plugins, SDK embedding, MCP reference)

**Production deployment:**
19. Language `01-architecture.md`
20. This document — Section 15 (config schema)
21. Language `10-prompts.md`
22. Language `11-complete-example.md`
23. `python-langchain/12-langsmith-observability.md`

---

## Language Quick-Start

### Python + LangGraph (5 minutes)

```bash
pip install langchain langgraph langchain-openai python-dotenv
```

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

@tool
def bash(command: str) -> str:
    """Run a shell command."""
    import subprocess
    r = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=30)
    return r.stdout + r.stderr

agent = create_react_agent(ChatOpenAI(model="gpt-4o-mini"), tools=[bash])
result = agent.invoke({"messages": [{"role": "user", "content": "List Python files here"}]})
print(result["messages"][-1].content)
```

→ See `python-langchain/00-index.md` for the full guide.

### Kotlin + Koog (5 minutes)

```bash
# Prerequisites: JDK 17+, Gradle 8+
```

```kotlin
// build.gradle.kts
implementation("ai.koog:koog-agents:0.8.0")
```

```kotlin
suspend fun main() {
    val executor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY"))
    val agent = AIAgent(executor = executor,
                        systemPrompt = "You are a coding assistant.",
                        llmModel = OpenAIModels.Chat.GPT4o)
    println(agent.run("List Python files in the current directory"))
}
```

→ See `kotlin-koog/00-index.md` for the full guide.

### C++23 (15 minutes)

```bash
# Prerequisites: CMake ≥ 3.28, Clang 17+ or GCC 13+, vcpkg
cmake -B build -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake
cmake --build build -j$(nproc)
ANTHROPIC_API_KEY=sk-ant-... ./build/agent "List Python files here"
```

```cmake
# vcpkg.json dependencies
{ "dependencies": ["nlohmann-json", "cpp-httplib", "spdlog", "usearch",
                    "boost-cobalt", "sqlite3", "sqlitecpp"] }
```

→ See `cpp23/00-index.md` for the full guide.

---

## Contributing a New Language Guide

When adding guidelines for a new language or framework:

1. Create `{language}/` directory with the same 12-file structure:
   `00-index`, `01-architecture`, `02-core-agent-loop`, `03-tool-system`,
   `04-memory-and-rag`, `05-knowledge-graph`, `06-event-driven`, `07-multiagent`,
   `08-self-improvement`, `09-session-persistence`, `10-prompts`, `11-complete-example`
2. Include all 25 arXiv paper citations in `00-index.md`
3. Add the language to the Cross-Language Implementation Map in Section 16
4. Add the language to the Decision Guide table in Section 16.1
5. Update `README.md` with the new directory entry and quick-start snippet

---

## 20. Hooks System (Full Reference)

Hooks are shell commands, HTTP endpoints, LLM prompts, or function callbacks that fire
at lifecycle events. They allow external tools to observe, validate, modify, or veto
agent actions without changing core agent code.

### 20.1 Hook Event Types

| Event | When it fires |
|-------|---------------|
| `PreToolUse` | Before permission check; can deny or modify input |
| `PostToolUse` | After tool succeeds; side-effects only |
| `PostToolUseFailure` | After tool errors; for alerting/logging |
| `UserPromptSubmit` | When user sends a message; can augment context |
| `SessionStart` | Session initialization |
| `SessionEnd` / `Stop` | Session teardown |
| `StopFailure` | Session ended with unhandled error |
| `PreCompact` | Before context compaction; can inject preservation hints |
| `PostCompact` | After compaction completes |
| `SubagentStart` | Before spawning a sub-agent |
| `SubagentStop` | Sub-agent completes or errors |
| `PermissionRequest` | When permission engine is about to ASK user |
| `PermissionDenied` | After a tool call is denied |
| `TaskCreated` | A task/todo entry is created |
| `TaskCompleted` | A task/todo entry is marked done |
| `Notification` | Agent emits a user notification |
| `FileChanged` | A file in the project changes (requires file watcher) |
| `CwdChanged` | Working directory changes |
| `WorktreeCreate` / `WorktreeRemove` | Git worktree lifecycle |

### 20.2 Hook Types

| Type | Mechanism | Use case |
|------|-----------|----------|
| `command` | Shell command; stdin/stdout JSON | Linting, audit logging, custom deny logic |
| `http` | POST JSON to a URL | Webhook notifications, external approval systems |
| `prompt` | LLM prompt with `$ARGUMENTS` placeholder | AI-driven permission decisions |
| `agent` | Spawn a full sub-agent to evaluate | Complex validation requiring tool use |
| `function` | In-process callback (SDK only) | Programmatic hooks in embedded agents |

### 20.3 Wire Protocol (command and http types)

**stdin payload (all events):**
```json
{
  "event": "PreToolUse",
  "tool": "Bash",
  "inputJson": {"command": "git status"},
  "sessionId": "abc-123",
  "workingDirectory": "/home/user/project"
}
```

**stdout response (PreToolUse only — can control execution):**
```json
{
  "allowed": true,
  "updatedInput": {"command": "git status --short"},
  "reason": "Normalized verbose flag"
}
```

| Response field | Effect |
|---------------|--------|
| `allowed: false` + `blocking: true` | Deny the tool call; inject `reason` as error |
| `allowed: false` + `blocking: false` | Warn user but allow (soft block) |
| `updatedInput: {...}` | Replace tool input with modified version |
| `(no stdout)` | Allow unconditionally |

**Exit code semantics (command type):**
- `0` — allow
- `2` — deny (if hook is blocking)
- Other non-zero — warning only

### 20.4 Hook Configuration

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(rm *)",
        "hooks": [
          {
            "type": "command",
            "command": "~/.agent/hooks/confirm_delete.sh",
            "blocking": true,
            "timeout": 10
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "https://audit.example.com/agent-events",
            "headers": {"Authorization": "Bearer ${AUDIT_TOKEN}"}
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.agent/hooks/session_report.sh",
            "once": true
          }
        ]
      }
    ]
  }
}
```

**Execution order:** Sequential within a matcher group; first non-allow result wins.

**`matcher` field syntax:** Same glob syntax as permission rules: `ToolName(input glob)`.
Omit matcher to run on all tool calls of that event type.

**Special flags:**
- `once: true` — hook fires once then is removed from the registry
- `asyncRewake: true` — hook runs in background; if it exits 2, re-enters the permission flow
- `timeout` — seconds before hook is killed (default varies by type; 10s for command, 60s for agent)

### 20.5 Example Hooks

**Audit logger (PostToolUse):**
```bash
#!/bin/bash
# Append every tool call to an audit log
INPUT=$(cat)
TOOL=$(echo "$INPUT" | jq -r '.tool')
CMD=$(echo "$INPUT" | jq -r '.inputJson.command // "n/a"')
echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) [$TOOL] $CMD" >> ~/.agent/audit.log
```

**Linter gate (PreToolUse on Write):**
```bash
#!/bin/bash
INPUT=$(cat)
PATH=$(echo "$INPUT" | jq -r '.inputJson.path')
if [[ "$PATH" == *.py ]]; then
  python -m py_compile "$PATH" 2>/dev/null || \
    echo '{"allowed":false,"reason":"Syntax error in file"}'; exit 2
fi
```

**Secrets scanner (PreToolUse on Write/Bash):**
```bash
#!/bin/bash
INPUT=$(cat)
CONTENT=$(echo "$INPUT" | jq -r '.inputJson.content // .inputJson.command // ""')
if echo "$CONTENT" | grep -qE 'sk-ant-[A-Za-z0-9]{20,}|ghp_[A-Za-z0-9]{36}'; then
  echo '{"allowed":false,"reason":"Possible secret detected in content"}'; exit 2
fi
```

---

## 21. Session & Conversation Management

### 21.1 Session File Format (JSONL)

Each session is a `.jsonl` file — one JSON object per line — stored at:
```
~/.agent/projects/<sha256(realpath(cwd))[:16]>/sessions/<uuid>.jsonl
```

**Message schema:**
```json
{
  "uuid": "msg-uuid-v4",
  "parentUuid": "prev-msg-uuid",
  "role": "user|assistant|system",
  "content": "string or ContentBlock[]",
  "timestamp": 1713456789,
  "toolUseId": "toolu_xyz",
  "sessionId": "session-uuid",
  "metadata": {
    "model": "claude-opus-4-6",
    "usage": {"input_tokens": 1200, "output_tokens": 340},
    "cost_usd": 0.0234,
    "duration_ms": 4200,
    "cache_read_tokens": 800,
    "cache_write_tokens": 400
  }
}
```

The `parentUuid` chain forms a linked list — the canonical conversation history is the path
from any leaf message back to root. This naturally supports branching.

### 21.2 Session Branching

```python
def fork_session(session_path: Path, branch_at_message_id: str) -> Path:
    """Create a new session file branching from a specific message."""
    messages = load_jsonl(session_path)
    # Walk parentUuid chain to find the branch point
    branch_msgs = collect_ancestors(messages, branch_at_message_id)
    
    new_session_id = str(uuid.uuid4())
    new_path = session_path.parent / f"{new_session_id}.jsonl"
    
    # Write the ancestor chain with new session_id, preserving original uuids
    for msg in branch_msgs:
        msg = dict(msg, sessionId=new_session_id)
        append_jsonl(new_path, msg)
    
    return new_path
```

Branch metadata for the session index:
```json
{
  "session_id": "new-uuid",
  "parent_session_id": "orig-uuid",
  "branch_at_message_id": "msg-uuid",
  "created_at": "2026-04-18T10:00:00Z"
}
```

### 21.3 Session Index (SQLite)

For large session archives, maintain a SQLite index alongside the JSONL files:

```sql
CREATE TABLE sessions (
    id          TEXT PRIMARY KEY,
    cwd_hash    TEXT NOT NULL,
    parent_id   TEXT REFERENCES sessions(id),
    title       TEXT,
    created_at  INTEGER,   -- unix timestamp
    updated_at  INTEGER,
    message_count INTEGER DEFAULT 0,
    total_cost  REAL DEFAULT 0.0,
    archived    INTEGER DEFAULT 0
);

CREATE INDEX idx_sessions_cwd ON sessions(cwd_hash, updated_at DESC);
```

### 21.4 Session Resume Flow

```python
def resume_session(session_id: str, cwd: str) -> Session:
    session = Session(cwd, session_id)  # loads JSONL
    # Restore cost state from message metadata
    cost = CostTracker()
    for msg in session.messages:
        if "metadata" in msg and "cost_usd" in msg["metadata"]:
            cost.total += msg["metadata"]["cost_usd"]
    # Inject away summary if session was idle
    if time_since_last_message(session) > AWAY_THRESHOLD:
        inject_away_summary(session)
    return session
```

### 21.5 Session Lifecycle & Cleanup

```
Active         → last message < 1h ago
Idle           → last message 1h–24h ago
Stale          → last message > 24h ago → trigger away summary on re-entry
Archived       → last message > 90 days → moved to archived/
Auto-deleted   → archived + storage quota exceeded (oldest first)
```

Cleanup policy (configurable):
```json
{
  "session": {
    "archive_after_days": 90,
    "auto_delete_after_days": 365,
    "max_sessions_per_project": 100,
    "max_total_size_mb": 500
  }
}
```

---

## 22. Cost & Rate Limit Management

### 22.1 Token Counting

**Estimation (fast, no API call):**
```python
def estimate_tokens(text: str) -> int:
    return len(text) // 4  # ~4 chars per token rule of thumb

def estimate_messages_tokens(messages: list, tools: list = None) -> int:
    total = sum(estimate_tokens(json.dumps(m)) for m in messages)
    if tools:
        total += sum(estimate_tokens(json.dumps(t)) for t in tools)
    return total
```

**Exact count (use for compaction decisions, not every turn):**
```python
def count_tokens_exact(messages: list, model: str, tools: list = None) -> int:
    result = client.messages.count_tokens(
        model=model,
        messages=messages,
        tools=tools or []
    )
    return result.input_tokens
```

Use estimation for per-turn checks; use the exact API only before compaction decisions
(the API call itself costs ~0.5ms and is not billed).

### 22.2 Cost Tracking

```python
MODEL_RATES_USD_PER_MTok = {
    "claude-opus-4-6":           {"in": 15.0,  "out": 75.0,  "cache_read": 1.5,  "cache_write": 18.75},
    "claude-opus-4-7":           {"in": 15.0,  "out": 75.0,  "cache_read": 1.5,  "cache_write": 18.75},
    "claude-sonnet-4-6":         {"in": 3.0,   "out": 15.0,  "cache_read": 0.3,  "cache_write": 3.75},
    "claude-haiku-4-5-20251001": {"in": 0.80,  "out": 4.0,   "cache_read": 0.08, "cache_write": 1.0},
}

@dataclass
class TurnCost:
    model: str
    input_tokens: int
    output_tokens: int
    cache_read_tokens: int
    cache_write_tokens: int
    cost_usd: float
    duration_ms: int

def compute_cost(usage, model: str) -> float:
    r = MODEL_RATES_USD_PER_MTok.get(model, MODEL_RATES_USD_PER_MTok["claude-opus-4-6"])
    return (
        usage.input_tokens              * r["in"]           / 1_000_000
        + usage.output_tokens           * r["out"]          / 1_000_000
        + getattr(usage, "cache_read_input_tokens", 0)   * r["cache_read"]  / 1_000_000
        + getattr(usage, "cache_creation_input_tokens", 0) * r["cache_write"] / 1_000_000
    )
```

**Prompt cache efficiency metric:**
```python
def cache_efficiency(usage) -> float:
    """0.0 = no cache benefit; 1.0 = entire context from cache."""
    total = usage.input_tokens + getattr(usage, "cache_read_input_tokens", 0)
    if total == 0: return 0.0
    return getattr(usage, "cache_read_input_tokens", 0) / total
```

### 22.3 Budget Enforcement

```python
class BudgetController:
    def __init__(self, max_cost_usd: float, warn_at_pct: float = 0.85):
        self.max_cost = max_cost_usd
        self.warn_pct = warn_at_pct
        self.spent = 0.0

    def record(self, cost: float):
        self.spent += cost
        ratio = self.spent / self.max_cost
        if ratio >= 1.0:
            raise BudgetExhausted(f"Budget ${self.max_cost:.2f} exceeded (spent ${self.spent:.2f})")
        if ratio >= self.warn_pct:
            warn_user(f"Approaching budget limit: ${self.spent:.2f} / ${self.max_cost:.2f}")

    def remaining(self) -> float:
        return max(0.0, self.max_cost - self.spent)
```

For multi-agent systems, use a shared atomic counter (thread-safe) across all agents:

```python
import threading

class SharedBudget:
    def __init__(self, total_usd: float):
        self._lock = threading.Lock()
        self._spent = 0.0
        self._total = total_usd

    def charge(self, cost: float) -> bool:
        """Returns False if budget exhausted."""
        with self._lock:
            if self._spent + cost > self._total:
                return False
            self._spent += cost
            return True
```

### 22.4 Rate Limit Handling

```python
import time, random

def with_retry(fn, max_retries: int = 8):
    for attempt in range(max_retries):
        try:
            return fn()
        except anthropic.RateLimitError as e:
            retry_after = float(e.response.headers.get("retry-after", 2 ** attempt))
            time.sleep(retry_after + random.uniform(0, 0.5))
        except anthropic.APIStatusError as e:
            if e.status_code == 529:  # overloaded
                time.sleep(min(2 ** attempt + random.uniform(0, 1), 60))
            elif e.status_code == 500 and attempt < 3:
                time.sleep(2 ** attempt)
            else:
                raise
    raise RuntimeError(f"Failed after {max_retries} retries")
```

**Rate limit response headers:**
| Header | Meaning |
|--------|---------|
| `retry-after` | Seconds to wait before retrying (429) |
| `x-ratelimit-limit-requests` | Requests per minute ceiling |
| `x-ratelimit-remaining-requests` | Remaining in current window |
| `x-ratelimit-limit-tokens` | Tokens per minute ceiling |
| `x-ratelimit-remaining-tokens` | Remaining tokens in current window |
| `x-ratelimit-reset-requests` | ISO timestamp when request window resets |

### 22.5 Context Window Budget

Monitor and act before the window fills:

```python
CONTEXT_LIMITS = {
    "claude-opus-4-6":           200_000,
    "claude-sonnet-4-6":         200_000,
    "claude-haiku-4-5-20251001": 200_000,
}

def context_budget_check(messages: list, tools: list, model: str, system_prompt_tokens: int = 0):
    limit = CONTEXT_LIMITS.get(model, 200_000)
    used = estimate_messages_tokens(messages, tools) + system_prompt_tokens
    ratio = used / limit
    
    if ratio >= 0.95:
        trigger_compaction(messages)   # hard threshold
    elif ratio >= 0.80:
        warn_user(f"Context {ratio:.0%} full — will compact soon")
    
    return ratio
```

---

## 23. IDE & Editor Integration

### 23.1 Integration Patterns

Two patterns exist for connecting an agent to an IDE:

**Pattern A — IDE Extension calls CLI** (most common)
```
IDE Extension ──subprocess──► Agent CLI
                              (stdin/stdout protocol)
IDE receives:
  - streaming text deltas
  - tool call notifications
  - file diff events
  - session metadata
```

**Pattern B — Agent drives IDE via MCP**
```
Agent ──MCP client──► IDE MCP server
                      (IDE exposes tools)
Agent can call:
  - ide.openFile(path, line)
  - ide.showDiff(old, new)
  - ide.getSelection()
  - ide.getDiagnostics()
```

### 23.2 IDE Context Injection Protocol

To avoid re-sending the full conversation on every keystroke:

```
Turn 1 (full):   [system_prompt] + [full_history] + [file_contents] + user_message
Turn 2+ (delta): [delta_messages_only] + [changed_files_only] + user_message
```

**Delta context format:**
```json
{
  "type": "ide_delta_context",
  "changed_files": [
    {
      "path": "src/auth.py",
      "content": "...full new content...",
      "cursor_line": 42,
      "selection": {"start": 40, "end": 45}
    }
  ],
  "diagnostics": [
    {
      "path": "src/auth.py",
      "line": 42,
      "severity": "error",
      "message": "Undefined name 'token'"
    }
  ],
  "open_files": ["src/auth.py", "src/types.py"]
}
```

Inject this as a system message at the start of each turn. On the first turn, include all
open files; on subsequent turns, include only files that changed since the last turn.

### 23.3 File Change Events

Wire the OS file watcher to feed changes back into the agent's context:

```python
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

class ProjectWatcher(FileSystemEventHandler):
    def __init__(self, on_change):
        self.on_change = on_change
        self._debounce = {}
    
    def on_modified(self, event):
        if event.is_directory: return
        path = event.src_path
        # Debounce: ignore if same file changed in last 100ms
        now = time.monotonic()
        if path in self._debounce and now - self._debounce[path] < 0.1:
            return
        self._debounce[path] = now
        self.on_change(path)

observer = Observer()
observer.schedule(ProjectWatcher(on_file_changed), path=cwd, recursive=True)
observer.start()
```

### 23.4 Diff/Patch Format for IDE Display

When displaying file changes to the user in an IDE:

```python
import difflib

def make_unified_diff(old_content: str, new_content: str, filepath: str) -> str:
    return "".join(difflib.unified_diff(
        old_content.splitlines(keepends=True),
        new_content.splitlines(keepends=True),
        fromfile=f"a/{filepath}",
        tofile=f"b/{filepath}",
        n=3  # context lines
    ))

def diff_stats(old: str, new: str) -> dict:
    added = deleted = 0
    for line in make_unified_diff(old, new, "").splitlines():
        if line.startswith("+") and not line.startswith("+++"): added += 1
        if line.startswith("-") and not line.startswith("---"): deleted += 1
    return {"added": added, "deleted": deleted}
```

### 23.5 MCP Tool Declarations for IDE Integration

```python
IDE_TOOLS = [
    {
        "name": "ide__open_file",
        "description": "Open a file in the editor at a specific line",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "line": {"type": "integer"}
            },
            "required": ["path"]
        }
    },
    {
        "name": "ide__get_selection",
        "description": "Get the currently selected text in the editor",
        "input_schema": {"type": "object", "properties": {}}
    },
    {
        "name": "ide__get_diagnostics",
        "description": "Get linting/type errors for the current file",
        "input_schema": {
            "type": "object",
            "properties": {"path": {"type": "string"}},
            "required": ["path"]
        }
    },
    {
        "name": "ide__show_diff",
        "description": "Show a diff in the editor's diff viewer",
        "input_schema": {
            "type": "object",
            "properties": {
                "old_content": {"type": "string"},
                "new_content": {"type": "string"},
                "filename": {"type": "string"}
            },
            "required": ["old_content", "new_content", "filename"]
        }
    }
]
```

---

## 24. Distributed Multi-Agent & A2A Protocol

### 24.1 A2A vs Local Multi-Agent

| | Local (same process) | Distributed (A2A) |
|-|---------------------|-------------------|
| Communication | In-memory / function call | Network (HTTP/WebSocket/SSE) |
| Latency | Microseconds | Milliseconds–seconds |
| Failure modes | Exception propagation | Network timeout, partial failure |
| Auth | Inherited from parent | Separate per-agent credentials |
| Scale | Limited by single machine | Horizontal |
| Use case | Parallel tool execution | Cross-service agent delegation |

### 24.2 Agent Card (Capability Advertisement)

Each distributed agent publishes an **Agent Card** — a JSON descriptor that other agents
use to discover and invoke it:

```json
{
  "agent_id": "code-reviewer-v2",
  "name": "Code Review Agent",
  "version": "2.1.0",
  "description": "Reviews code diffs for bugs, security issues, and style",
  "capabilities": ["code_review", "security_audit", "style_check"],
  "input_schema": {
    "type": "object",
    "properties": {
      "diff": {"type": "string", "description": "Unified diff to review"},
      "language": {"type": "string"},
      "focus": {"type": "array", "items": {"type": "string"},
                "description": "Aspects to focus on: security|performance|style|bugs"}
    },
    "required": ["diff"]
  },
  "output_schema": {
    "type": "object",
    "properties": {
      "issues": {"type": "array"},
      "summary": {"type": "string"},
      "approved": {"type": "boolean"}
    }
  },
  "endpoint": "https://agents.example.com/code-reviewer",
  "auth": {"type": "bearer"},
  "streaming": true,
  "max_concurrent_tasks": 10
}
```

### 24.3 Task Delegation Protocol

**Request:**
```json
{
  "task_id": "task-uuid-v4",
  "agent_id": "code-reviewer-v2",
  "input": {
    "diff": "...",
    "language": "python",
    "focus": ["security", "bugs"]
  },
  "callback_url": "https://orchestrator.example.com/results/task-uuid-v4",
  "timeout_seconds": 120,
  "streaming": true
}
```

**Streaming response (Server-Sent Events):**
```
event: status
data: {"task_id":"task-uuid","status":"running","progress":0.2}

event: artifact
data: {"task_id":"task-uuid","type":"partial_result","content":"Found 2 issues...","append":true}

event: artifact
data: {"task_id":"task-uuid","type":"partial_result","content":" reviewing auth module","append":true}

event: status
data: {"task_id":"task-uuid","status":"completed","final":true}
```

**Terminal task states:**
- `completed` — success, result available
- `failed` — agent encountered an unrecoverable error
- `killed` — timed out or cancelled by orchestrator
- `auth-required` — agent needs additional credentials (pauses, waits for re-auth)

### 24.4 Result Reassembly

For streaming artifact updates with `append: true`:

```python
class A2AResultReassembler:
    def __init__(self):
        self._artifacts: dict[str, str] = {}
        self._status = "pending"
        self._final = False

    def feed(self, event_type: str, data: dict):
        if event_type == "status":
            self._status = data["status"]
            self._final = data.get("final", False)
        elif event_type == "artifact":
            key = data.get("artifact_id", "default")
            if data.get("append"):
                self._artifacts[key] = self._artifacts.get(key, "") + data["content"]
            else:
                self._artifacts[key] = data["content"]

    def is_done(self) -> bool:
        return self._final or self._status in ("completed", "failed", "killed")

    def result(self) -> dict:
        return {"status": self._status, "artifacts": self._artifacts}
```

### 24.5 Inter-Agent Authentication

```python
# Agent-to-agent calls use short-lived JWT tokens issued by an orchestrator

def make_agent_token(caller_id: str, callee_id: str, task_id: str,
                     secret: str, ttl_seconds: int = 300) -> str:
    import jwt, time
    payload = {
        "sub": caller_id,
        "aud": callee_id,
        "task_id": task_id,
        "iat": int(time.time()),
        "exp": int(time.time()) + ttl_seconds,
    }
    return jwt.encode(payload, secret, algorithm="HS256")

def verify_agent_token(token: str, expected_caller: str, secret: str) -> dict:
    import jwt
    payload = jwt.decode(token, secret, algorithms=["HS256"],
                         options={"require": ["sub", "aud", "task_id", "exp"]})
    if payload["sub"] != expected_caller:
        raise ValueError("Token caller mismatch")
    return payload
```

### 24.6 Graceful Degradation for Remote Agent Failures

```python
async def call_remote_agent(agent_card: dict, input: dict, fallback_fn=None):
    try:
        result = await a2a_client.invoke(
            endpoint=agent_card["endpoint"],
            input=input,
            timeout=agent_card.get("timeout_seconds", 60)
        )
        return result
    except asyncio.TimeoutError:
        if fallback_fn:
            return await fallback_fn(input)   # local fallback
        raise
    except (ConnectionError, aiohttp.ClientError) as e:
        # Inject error into agent context rather than crashing
        return {
            "status": "failed",
            "error": f"Remote agent unavailable: {e}",
            "artifacts": {}
        }
```

**Design principle:** Remote agent failures should inject a structured error result into the
orchestrator's tool result slot — never crash the orchestrator session. The orchestrator can
then decide to retry, use a fallback, or ask the user.

---

## 25. LSP Integration (Language Server Protocol)

Agents act as **LSP clients** — they spawn language servers and query them to provide
code intelligence (definitions, references, diagnostics, hover) as tool outputs. No
agent studied acts as an LSP server itself.

### 25.1 Architecture

```
Agent
  │
  ▼ spawn subprocess
Language Server (clangd, pyright, rust-analyzer, etc.)
  │ stdin/stdout JSON-RPC (Content-Length framing)
  ▼
Agent LSP Client
  ├── sends: initialize, textDocument/didOpen, textDocument/hover, ...
  └── receives: publishDiagnostics notifications → injected into context
```

**Transport:** Always stdio (subprocess). The agent spawns the language server as a child
process and communicates via stdin/stdout using JSON-RPC 2.0 with HTTP-style
`Content-Length:` framing headers.

### 25.2 Supported LSP Methods

| Category | Method | Use in agent |
|----------|--------|-------------|
| Lifecycle | `initialize` / `initialized` / `shutdown` | Session setup/teardown |
| Diagnostics | `textDocument/publishDiagnostics` (notification) | Inject errors into context |
| Navigation | `textDocument/definition` | Resolve symbol to declaration |
| Navigation | `textDocument/references` | Find all usages of a symbol |
| Navigation | `textDocument/implementation` | Find concrete implementations |
| Navigation | `textDocument/declaration` | Go to type declaration |
| Symbols | `textDocument/documentSymbol` | File-level symbol tree |
| Symbols | `workspace/symbol` | Project-wide symbol search |
| Call hierarchy | `textDocument/prepareCallHierarchy` | Find callers/callees |
| Call hierarchy | `callHierarchy/incomingCalls` | Who calls this function |
| Call hierarchy | `callHierarchy/outgoingCalls` | What this function calls |
| Hover | `textDocument/hover` | Type info, documentation |
| Completion | `textDocument/completion` | Autocomplete suggestions |
| Formatting | `textDocument/formatting` | Format a file |

### 25.3 LSP Tool Implementation

Expose LSP capabilities as a single `lsp` agent tool with an `operation` parameter:

```python
LSP_TOOL = {
    "name": "lsp",
    "description": "Query the language server for code intelligence: definitions, references, diagnostics, hover info, symbols.",
    "input_schema": {
        "type": "object",
        "properties": {
            "operation": {
                "type": "string",
                "enum": ["go_to_definition", "find_references", "hover",
                         "document_symbols", "workspace_symbols",
                         "go_to_implementation", "incoming_calls", "outgoing_calls",
                         "diagnostics", "format_document"]
            },
            "file_path": {"type": "string", "description": "Absolute path to the file"},
            "line":      {"type": "integer", "description": "0-based line number"},
            "character": {"type": "integer", "description": "0-based character offset"},
            "query":     {"type": "string",  "description": "For workspace_symbols: search query"}
        },
        "required": ["operation", "file_path"]
    }
}
```

### 25.4 JSON-RPC Client (minimal Python implementation)

```python
import json, subprocess, threading
from typing import Any

class LspClient:
    def __init__(self, command: list[str]):
        self._proc = subprocess.Popen(
            command, stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.DEVNULL
        )
        self._id = 0
        self._lock = threading.Lock()

    def _send(self, method: str, params: Any) -> Any:
        self._id += 1
        msg = {"jsonrpc": "2.0", "id": self._id, "method": method, "params": params}
        body = json.dumps(msg).encode()
        header = f"Content-Length: {len(body)}\r\n\r\n".encode()
        with self._lock:
            self._proc.stdin.write(header + body)
            self._proc.stdin.flush()
            return self._read_response()

    def _read_response(self) -> Any:
        headers = {}
        while True:
            line = self._proc.stdout.readline().decode().strip()
            if not line:
                break
            k, v = line.split(":", 1)
            headers[k.strip().lower()] = v.strip()
        length = int(headers["content-length"])
        body = self._proc.stdout.read(length)
        return json.loads(body).get("result")

    def initialize(self, root_uri: str) -> None:
        self._send("initialize", {
            "rootUri": root_uri,
            "capabilities": {"textDocument": {"hover": {"contentFormat": ["plaintext"]},
                                               "publishDiagnostics": {}}},
            "clientInfo": {"name": "agent-lsp-client", "version": "1.0"}
        })
        self._notify("initialized", {})

    def _notify(self, method: str, params: Any) -> None:
        msg = {"jsonrpc": "2.0", "method": method, "params": params}
        body = json.dumps(msg).encode()
        header = f"Content-Length: {len(body)}\r\n\r\n".encode()
        with self._lock:
            self._proc.stdin.write(header + body)
            self._proc.stdin.flush()

    def go_to_definition(self, uri: str, line: int, char: int) -> list:
        return self._send("textDocument/definition", {
            "textDocument": {"uri": uri}, "position": {"line": line, "character": char}
        }) or []

    def hover(self, uri: str, line: int, char: int) -> str:
        result = self._send("textDocument/hover", {
            "textDocument": {"uri": uri}, "position": {"line": line, "character": char}
        })
        if result and "contents" in result:
            c = result["contents"]
            return c.get("value", c) if isinstance(c, dict) else str(c)
        return ""

    def document_symbols(self, uri: str) -> list:
        return self._send("textDocument/documentSymbol", {"textDocument": {"uri": uri}}) or []

    def shutdown(self):
        self._send("shutdown", None)
        self._notify("exit", None)
        self._proc.terminate()
```

### 25.5 LSP Server Configuration (Plugin Manifest)

```json
{
  "lspServers": {
    "python": {
      "command": "pyright-langserver",
      "args": ["--stdio"],
      "languages": ["python"],
      "file_extensions": [".py", ".pyi"],
      "root_patterns": ["pyproject.toml", "setup.py", "requirements.txt"]
    },
    "rust": {
      "command": "rust-analyzer",
      "args": [],
      "languages": ["rust"],
      "file_extensions": [".rs"],
      "root_patterns": ["Cargo.toml"]
    },
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"],
      "languages": ["typescript", "javascript"],
      "file_extensions": [".ts", ".tsx", ".js", ".jsx"],
      "root_patterns": ["tsconfig.json", "package.json"]
    }
  }
}
```

### 25.6 Lazy Initialization

Start language servers on first use, not at session start. Map `file_extension → server_id`
and initialize the server the first time a file with that extension is queried:

```python
class LspManager:
    def __init__(self, configs: dict):
        self._configs = configs       # extension → config
        self._clients: dict[str, LspClient] = {}

    def get_client(self, file_path: str) -> LspClient | None:
        ext = Path(file_path).suffix
        cfg = self._configs.get(ext)
        if not cfg:
            return None
        if ext not in self._clients:
            client = LspClient(cfg["command"].split() + cfg.get("args", []))
            client.initialize(f"file://{Path(file_path).parent}")
            self._clients[ext] = client
        return self._clients[ext]
```

### 25.7 Diagnostics Injection

When a file is opened or modified, passively receive `publishDiagnostics` from the
language server and cache them. Inject into tool results or system prompt:

```python
def format_diagnostics_for_context(diagnostics: list[dict], file_path: str) -> str:
    if not diagnostics:
        return ""
    lines = [f"LSP diagnostics for {file_path}:"]
    for d in diagnostics:
        sev = {1: "ERROR", 2: "WARNING", 3: "INFO", 4: "HINT"}.get(d.get("severity", 1), "?")
        r = d["range"]["start"]
        lines.append(f"  [{sev}] line {r['line']+1}:{r['character']+1} — {d['message']}")
    return "\n".join(lines)
```

### 25.8 DAP (Debug Adapter Protocol)

**Not implemented in any agent studied.** DAP requires interactive debugger sessions
(REPL-style step/continue/inspect), which conflicts with the agent's asynchronous tool
execution model. Recommended approach if needed: expose DAP as a shell tool
(`bash: "python -m debugpy --listen 5678 script.py"`) and let the model interact via
separate `debugpy` client calls rather than implementing a full DAP client.

---

## 26. Plugin System

Plugins extend the agent with new tools, slash commands, hooks, MCP servers, LSP servers,
and system prompt fragments — without modifying the core agent code.

### 26.1 Plugin Manifest Format (`plugin.json`)

```json
{
  "name": "my-plugin",
  "version": "1.2.0",
  "description": "Adds Docker and Kubernetes tooling",
  "author": {
    "name": "Your Name",
    "email": "you@example.com",
    "url": "https://github.com/you/my-plugin"
  },
  "capabilities": ["read_only", "write", "network"],
  "commands": [
    {
      "name": "docker-status",
      "description": "Show Docker container status",
      "type": "bash",
      "command": "docker ps --format json"
    },
    {
      "name": "k8s-logs",
      "description": "Stream Kubernetes pod logs",
      "type": "prompt",
      "prompt": "Run: kubectl logs $ARGUMENTS --tail=50"
    }
  ],
  "skills": [
    {
      "name": "docker-expert",
      "description": "Docker best practices advisor",
      "systemPromptFragment": "You have deep Docker expertise. Prefer multi-stage builds. Always pin image digests."
    }
  ],
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(docker rm *)",
        "command": ".claude-plugin/hooks/confirm_docker_rm.sh",
        "blocking": true
      }
    ],
    "PostToolUse": [
      {
        "command": ".claude-plugin/hooks/audit.sh"
      }
    ]
  },
  "mcpServers": {
    "docker-mcp": {
      "command": "uvx",
      "args": ["docker-mcp-server"]
    }
  },
  "lspServers": {
    "dockerfile": {
      "command": "docker-langserver",
      "args": ["--stdio"],
      "file_extensions": [".dockerfile", "Dockerfile"]
    }
  }
}
```

### 26.2 Plugin Capability Levels

| Capability | What it allows |
|-----------|---------------|
| `read_only` | File reads, queries, diagnostics |
| `write` | File writes, edits |
| `network` | HTTP requests, MCP over network, OAuth |
| `shell` | Arbitrary shell execution via hooks |
| `agent` | Spawn sub-agents |

Plugins without a `capabilities` field are trusted unconditionally (backwards compatibility).
Always declare capabilities explicitly in new plugins.

### 26.3 Plugin Loading Flow

```
Session start
  │
  ├── 1. Scan built-in plugins (bundled with agent binary)
  ├── 2. Scan ~/.agent/plugins/ (user-installed)
  ├── 3. Scan .agent/plugins/ (project-specific)
  ├── 4. Load each plugin.json → validate manifest → check capabilities
  ├── 5. Register commands, hooks, MCP servers, LSP servers
  └── 6. Inject system prompt fragments from skills

On /reload-plugins:
  → Re-scan all directories, hot-reload without restarting session
```

**Directory structure:**
```
~/.agent/plugins/
└── my-plugin/
    ├── plugin.json          (manifest)
    ├── hooks/
    │   └── confirm_rm.sh   (hook scripts)
    └── assets/             (optional static files)

.agent/plugins/              (project-local, same structure)
```

### 26.4 Plugin Isolation Model

| Plugin component | Isolation |
|-----------------|-----------|
| Hooks (command type) | Subprocess — cannot access agent memory |
| Hooks (prompt type) | LLM call — isolated context |
| MCP servers | Subprocess — separate process, stdio/HTTP |
| LSP servers | Subprocess — separate process, stdio |
| Skills (system prompt) | In-process — injected into main system prompt |
| Commands (bash type) | Subprocess (shell) |
| Commands (prompt type) | In-process LLM call |

### 26.5 Writing a Minimal Plugin

```bash
mkdir -p ~/.agent/plugins/my-hello/hooks
cat > ~/.agent/plugins/my-hello/plugin.json << 'EOF'
{
  "name": "my-hello",
  "version": "0.1.0",
  "description": "Demo plugin",
  "capabilities": ["read_only"],
  "commands": [
    {
      "name": "hello",
      "description": "Say hello",
      "type": "bash",
      "command": "echo 'Hello from plugin!'"
    }
  ],
  "hooks": {
    "SessionStart": [
      { "command": "echo 'Plugin loaded' >> /tmp/agent-plugin.log" }
    ]
  }
}
EOF
```

User runs `/hello` → agent executes `echo 'Hello from plugin!'` and returns output.

### 26.6 Plugin vs MCP Server vs Hook (When to Use Which)

| Need | Use |
|------|-----|
| New tool calling external API | MCP server |
| Slash command running a script | Plugin command (bash type) |
| Behavior guard (block dangerous ops) | Plugin hook (PreToolUse, blocking) |
| Domain system prompt fragment | Plugin skill |
| Code intelligence for new language | Plugin LSP server |
| Audit/logging all tool calls | Plugin hook (PostToolUse) |
| Prompt-driven advisor | Plugin skill or command (prompt type) |

---

## 27. SDK / Programmatic API

An agent SDK lets you embed the agent in another application, run it headlessly in CI,
or build a custom UI. The SDK exposes the full agent loop as a programmatic API.

### 27.1 Core SDK Interface

```typescript
// TypeScript — canonical SDK interface

interface AgentSDK {
  // One-shot: runs agent to completion, returns final text
  query(params: QueryParams): AsyncIterable<SDKEvent>

  // Session management
  createSession(options: SessionOptions): Promise<SDKSession>
  resumeSession(sessionId: string, options: SessionOptions): Promise<SDKSession>
  listSessions(options?: ListOptions): Promise<SessionInfo[]>
  getSessionMessages(sessionId: string): Promise<Message[]>
  forkSession(sessionId: string, options?: ForkOptions): Promise<ForkResult>
  renameSession(sessionId: string, title: string): Promise<void>
  tagSession(sessionId: string, tag: string | null): Promise<void>

  // MCP server (expose agent tools to other agents)
  createMcpServer(options: McpServerOptions): McpServer
}

interface QueryParams {
  prompt: string | AsyncIterable<UserMessage>
  sessionId?: string               // resume existing session
  model?: string                   // override model
  maxTurns?: number
  permissionMode?: PermissionMode
  tools?: SdkToolDefinition[]      // additional custom tools
  systemPrompt?: string            // override system prompt
  onText?: (text: string) => void  // streaming text callback
  onToolCall?: (call: ToolCall) => void
  onCost?: (cost: CostUpdate) => void
}

// Event stream from query()
type SDKEvent =
  | { type: "text";     text: string }
  | { type: "tool_use"; name: string; input: Record<string, unknown>; id: string }
  | { type: "tool_result"; tool_use_id: string; content: string; is_error: boolean }
  | { type: "cost";     total_usd: number; turn_usd: number }
  | { type: "done";     final_text: string; session_id: string }
```

### 27.2 Custom Tool Registration

```typescript
function defineTool<T extends Record<string, z.ZodType>>(
  name: string,
  description: string,
  schema: T,
  handler: (args: z.infer<z.ZodObject<T>>) => Promise<string>
): SdkToolDefinition

// Example:
const searchTool = defineTool(
  "search_docs",
  "Search internal documentation",
  { query: z.string(), limit: z.number().optional() },
  async ({ query, limit = 10 }) => {
    const results = await internalSearch(query, limit)
    return JSON.stringify(results)
  }
)

const sdk = new AgentSDK({ apiKey: process.env.ANTHROPIC_API_KEY })
for await (const event of sdk.query({ prompt: "Find docs on OAuth", tools: [searchTool] })) {
  if (event.type === "text") process.stdout.write(event.text)
}
```

### 27.3 Python SDK Embedding

```python
from agent_sdk import AgentSDK, tool, PermissionMode
import asyncio

@tool("search_db", "Search the database")
async def search_db(query: str, limit: int = 10) -> str:
    results = await db.search(query, limit)
    return "\n".join(str(r) for r in results)

async def main():
    sdk = AgentSDK(
        api_key=os.getenv("ANTHROPIC_API_KEY"),
        model="claude-sonnet-4-6",
        permission_mode=PermissionMode.DEFAULT,
        tools=[search_db],
        system_prompt="You are a database assistant.",
    )

    session = await sdk.create_session()
    async for event in session.query("Find all users created last week"):
        if event.type == "text":
            print(event.text, end="", flush=True)
        elif event.type == "cost":
            pass  # track cost

asyncio.run(main())
```

### 27.4 SDK Session Lifecycle

```
create_session()   →  session_id issued, JSONL file created
  │
  ▼
session.query()    →  runs agent turn loop, streams events
  │
  ▼
session.query()    →  subsequent turns use same session_id (conversation continues)
  │
  ▼
fork_session()     →  creates branch from any past message
  │
  ▼
(session auto-archived after idle threshold)
```

### 27.5 SDK MCP Server (Expose Agent as MCP Tool Provider)

An agent can itself act as an MCP server, exposing its tools to other agents or
to an IDE that speaks MCP:

```typescript
const server = sdk.createMcpServer({
  name: "my-agent-server",
  version: "1.0.0",
  tools: [
    defineTool("agent_query", "Ask the coding agent", { prompt: z.string() },
      async ({ prompt }) => {
        let result = ""
        for await (const ev of sdk.query({ prompt }))
          if (ev.type === "text") result += ev.text
        return result
      })
  ]
})

server.listen({ transport: "stdio" })  // or { transport: "http", port: 8080 }
```

### 27.6 Headless / CI Mode

For running the agent in CI pipelines without a REPL:

```python
import subprocess, json, sys

def run_agent_headless(prompt: str, cwd: str) -> str:
    """Run agent CLI in headless mode, capture JSON output."""
    result = subprocess.run(
        ["agent", "--output-format", "json", "--max-turns", "20", "--permission-mode", "yolo"],
        input=prompt.encode(),
        capture_output=True,
        cwd=cwd,
        timeout=300
    )
    output = json.loads(result.stdout)
    return output["result"]

# Or via SDK (preferred — no subprocess overhead):
async def run_ci_check(diff: str) -> dict:
    sdk = AgentSDK(permission_mode="yolo", max_cost_usd=2.0)
    session = await sdk.create_session()
    events = []
    async for ev in session.query(f"Review this diff:\n{diff}"):
        events.append(ev)
    return {"text": next(e.text for e in reversed(events) if e.type == "done")}
```

---

## 28. MCP (Model Context Protocol) — Complete Reference

MCP is the standard protocol for connecting agents to external tool providers (databases,
APIs, file systems, code intelligence, etc.). The agent acts as an MCP **client**; external
services act as MCP **servers**.

### 28.1 MCP Architecture

```
Agent (MCP Client)
  │
  ├── stdio transport  ──► local MCP server (subprocess)
  ├── HTTP transport   ──► remote MCP server (REST endpoint)
  ├── SSE transport    ──► remote MCP server (streaming)
  └── WebSocket        ──► remote MCP server (bidirectional)

MCP Server exposes:
  ├── tools      → callable functions (agent invokes via tools/call)
  ├── resources  → file-like data (agent reads via resources/read)
  └── prompts    → prompt templates (agent fetches via prompts/get)
```

### 28.2 MCP Configuration (`.mcp.json`)

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"],
      "env": {}
    },
    "postgres": {
      "command": "uvx",
      "args": ["mcp-server-postgres"],
      "env": { "DATABASE_URL": "${DATABASE_URL}" }
    },
    "github": {
      "type": "http",
      "url": "https://api.github.com/mcp",
      "headers": { "Authorization": "Bearer ${GITHUB_TOKEN}" },
      "toolCallTimeoutMs": 30000
    },
    "internal-api": {
      "type": "http",
      "url": "https://internal.example.com/mcp",
      "oauth": {
        "clientId": "agent-mcp-client",
        "callbackPort": 7777,
        "authServerMetadataUrl": "https://auth.example.com/.well-known/oauth-authorization-server"
      }
    }
  }
}
```

Environment variable expansion: `${VAR}` and `${VAR:-default}` syntax are supported.

### 28.3 Initialization Handshake

```json
// Client → Server
{ "jsonrpc": "2.0", "id": 1, "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "clientInfo": { "name": "agent", "version": "1.0.0" },
    "capabilities": { "tools": {}, "resources": {}, "prompts": {} }
  }
}

// Server → Client
{ "jsonrpc": "2.0", "id": 1, "result": {
    "protocolVersion": "2024-11-05",
    "serverInfo": { "name": "my-server", "version": "0.1.0" },
    "capabilities": { "tools": { "listChanged": true } }
  }
}

// Client → Server (notification, no response)
{ "jsonrpc": "2.0", "method": "initialized", "params": {} }
```

### 28.4 Tool Discovery and Naming

```python
# After initialize, list all tools
tools_result = mcp_client.call("tools/list", {})

for tool in tools_result["tools"]:
    # Namespace: mcp__{server_name}__{tool_name}
    agent_tool_name = f"mcp__{server_name}__{tool['name']}"
    agent_tools.append({
        "name": agent_tool_name,
        "description": tool["description"],
        "input_schema": tool["inputSchema"]
    })
```

**Tool name normalization rules:**
- Server name: lowercase, spaces/hyphens → underscores
- Tool name: same normalization
- Separator: double underscore `__`
- Example: server `"my-database"`, tool `"query tables"` → `mcp__my_database__query_tables`

### 28.5 Tool Execution

```python
# Agent calls tool
response = mcp_client.call("tools/call", {
    "name": "query",
    "arguments": { "sql": "SELECT * FROM users LIMIT 10" }
})

# MCP server returns
# { "content": [{"type": "text", "text": "id,name\n1,Alice\n2,Bob"}], "isError": false }
```

Tool result content types:
| Type | Description |
|------|-------------|
| `text` | Plain text output |
| `image` | Base64-encoded image with MIME type |
| `resource` | URI reference to a resource |

### 28.6 Resources and Prompts

```python
# List available resources (file-like data)
resources = mcp_client.call("resources/list", {})
# → [{"uri": "db://users/schema", "name": "Users schema", "mimeType": "application/json"}]

# Read a resource
content = mcp_client.call("resources/read", {"uri": "db://users/schema"})
# → {"contents": [{"uri": "...", "mimeType": "application/json", "text": "{...}"}]}

# List prompt templates
prompts = mcp_client.call("prompts/list", {})
# → [{"name": "code-review", "description": "Review code for issues", "arguments": [...]}]

# Get a prompt
prompt = mcp_client.call("prompts/get", {
    "name": "code-review",
    "arguments": { "code": "def foo(): pass", "language": "python" }
})
# → {"messages": [{"role": "user", "content": {"type": "text", "text": "Review this Python..."}}]}
```

### 28.7 MCP over OAuth

For MCP servers requiring OAuth 2.0 authentication:

```python
# OAuth discovery
import httpx, json

async def discover_oauth_config(metadata_url: str) -> dict:
    async with httpx.AsyncClient() as client:
        r = await client.get(metadata_url)
        return r.json()
    # Returns: authorization_endpoint, token_endpoint, scopes_supported, ...

# PKCE flow (same as agent OAuth — see §02 auth guide)
# After obtaining access_token, inject as Authorization header:
mcp_client = McpClient(
    transport="http",
    url="https://api.example.com/mcp",
    headers={"Authorization": f"Bearer {access_token}"}
)
```

### 28.8 MCP Server Status Lifecycle

```
pending     → initializing (transport connecting)
connected   → healthy, tools available
needs-auth  → server responded with auth challenge (OAuth flow required)
failed      → connection or initialize error
disabled    → user or policy disabled this server
```

On `needs-auth`: pause tool calls to this server, trigger OAuth flow, refresh token,
retry initialize.

### 28.9 Writing a Minimal MCP Server (Python)

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent
import mcp.types as types

app = Server("my-tools")

@app.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="search_docs",
            description="Search internal documentation",
            inputSchema={
                "type": "object",
                "properties": {"query": {"type": "string"}},
                "required": ["query"]
            }
        )
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "search_docs":
        results = await search(arguments["query"])
        return [TextContent(type="text", text="\n".join(results))]
    raise ValueError(f"Unknown tool: {name}")

async def main():
    async with stdio_server() as (read, write):
        await app.run(read, write, app.create_initialization_options())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

### 28.10 Agent-as-MCP-Server (Bidirectional)

An agent can itself act as an MCP server, making its session accessible to IDE extensions
or orchestrator agents:

```python
# Expose agent session as MCP server
from mcp.server import Server
from mcp.server.stdio import stdio_server

agent_server = Server("coding-agent")

@agent_server.list_tools()
async def list_tools():
    return [Tool(name="ask_agent", description="Ask the coding agent",
                 inputSchema={"type":"object","properties":{"prompt":{"type":"string"}},"required":["prompt"]})]

@agent_server.call_tool()
async def call_tool(name: str, args: dict):
    if name == "ask_agent":
        result = await run_agent_query(args["prompt"])
        return [TextContent(type="text", text=result)]
```

This pattern lets an IDE that supports MCP (e.g., via an extension) talk directly to
the running agent session without a custom protocol.
