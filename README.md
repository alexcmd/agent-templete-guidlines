# Agent Implementation Guidelines

Production-grade documentation for building modern LLM agent systems across three languages,
grounded in peer-reviewed research (24 arXiv papers) and production frameworks.

---

## Overview

This collection provides complete, runnable implementation guidance for building sophisticated
AI agent systems. Each language section covers the same topics — agent loops, tool systems,
memory, RAG, knowledge graphs, event-driven architecture, multi-agent workflows, self-improvement,
session persistence, and prompt engineering — using the best idiomatic approaches for that
ecosystem.

**Source material analyzed:**
- [Claude Code (TypeScript)](../claude-code/) — 21-section system prompt, compaction, REPL, concurrent tool execution (10×)
- [Gemini CLI (TypeScript)](../gemini-cli/) — 1M context, 7-strategy model routing, LLM loop detection, tool output masking, A2A server
- [Claw Code (Rust)](../claw-code/) — 9-crate workspace, 43 tools, A2A, worker lifecycle, recovery recipes
- [Claurst (Rust)](https://github.com/Kuberwastaken/claurst) — 12-crate workspace, 35+ providers, managed-agent orchestration, AutoDream memory

---

## Directory Structure

```
agent-guidelines/
├── README.md                         ← This file (master index)
├── 00-research-foundation.md         ← 25 arXiv papers with full citations
│
├── 01-overview.md                    ← Universal architecture index + reading paths
├── 01-core-runtime.md                ← §1–4: Agent runtime, component inventory, multi-provider
├── 01-prompts-permissions.md         ← §5–6: System prompt construction, 6-mode permission system
├── 01-memory-rag-knowledge.md        ← §7–9: Memory tiers (CoALA), memdir, RAG pipelines, KG
├── 01-event-driven-multiagent.md     ← §10–11: Event sourcing, CQRS, local multi-agent
├── 01-advanced-patterns.md           ← §12–14: Self-improvement, session persistence, prompts
├── 01-config-infra-reference.md      ← §15–18: Config schema, Docker infra, research papers
├── 01-planning-hooks.md              ← §19–20: Planning mode, hook system (27+ events)
├── 01-sessions-cost-ide.md           ← §21–23: Session JSONL, cost tracking, IDE integration
├── 01-distributed-agents.md          ← §24: A2A protocol, Agent Card, JWT auth
├── 01-extensions-sdk-mcp.md          ← §25–28: LSP, plugin system, SDK API, MCP (6 transports)
│
├── 02-auth-overview-apikey.md        ← §1–3: Auth overview, API key patterns
├── 02-auth-oauth-token-keychain.md   ← §4–6: OAuth 2.0 PKCE, token management, keychain storage
├── 02-auth-cloud-api-reference.md    ← §7–13: Bedrock, Vertex, multi-cloud, error handling
├── 02-auth-multiprovider.md          ← §14: Multi-provider routing and fallback
│
├── 03-minimal-agents.md              ← §1–2: Minimal working agents (Python/TS/Rust/Go/Kotlin)
├── 03-production-agents.md           ← §3–4: Full REPL agent, streaming, MCP integration
├── 03-utilities-mcp-checklist.md     ← §5–13: Utilities, MCP server, production checklist
│
├── 04-todo-task-management.md        ← Todo/Task CRUD, storage, state machines, dependency DAGs
│
├── cpp23/                            ← C++23 implementation (12 files)
│   ├── 00-index.md                   ← C++23 quick-start and package overview
│   ├── 01-architecture.md            ← CMake structure, crate layout, layer diagram
│   ├── 02-core-agent-loop.md         ← ReAct, streaming, Reflexion, SELF-REFINE
│   ├── 03-tool-system.md             ← ToolSpec, ToolRegistry, dispatcher, MCP
│   ├── 04-memory-and-rag.md          ← MemGPT tiers, USearch/FAISS, HyDE, CRAG
│   ├── 05-knowledge-graph.md         ← Entity/Relation, HippoRAG PPR, LightRAG, PathRAG
│   ├── 06-event-driven.md            ← Event sourcing, CQRS, ESAA, hooks
│   ├── 07-multiagent.md              ← Process spawning, AutoGen actor, MetaGPT SOP
│   ├── 08-self-improvement.md        ← ExpeL, Reflexion, CRITIC, SkillLibrary
│   ├── 09-session-persistence.md     ← JSONL session, compaction, SQLite
│   ├── 10-prompts.md                 ← SystemPromptBuilder, ReAct/HyDE templates
│   └── 11-complete-example.md        ← Full software-dev agent with Docker Compose
│
├── kotlin-koog/                      ← Kotlin + Koog v0.8.0 (12 files)
│   ├── 00-index.md                   ← Gradle bootstrap, quick-start, provider config
│   ├── 01-architecture.md            ← Module structure, layer diagram, Spring AI
│   ├── 02-core-agent-loop.md         ← Strategy graph, ReAct, Reflexion, streaming
│   ├── 03-tool-system.md             ← @Tool, ToolSet, ToolRegistry, MCP, parallel
│   ├── 04-memory-and-rag.md          ← ChatMemory, MemGPT, Qdrant, CRAG, BGE-M3
│   ├── 05-knowledge-graph.md         ← Neo4j, HippoRAG PPR, LightRAG, GraphRAG
│   ├── 06-event-driven.md            ← EventHandler, OpenTelemetry, Langfuse, Tracy
│   ├── 07-multiagent.md              ← A2A protocol, SupervisorAgent, AutoGen
│   ├── 08-self-improvement.md        ← ExpeL, CRITIC, SELF-REFINE, SkillLibrary
│   ├── 09-session-persistence.md     ← Koog Persistence, Exposed ORM, compaction
│   ├── 10-prompts.md                 ← SystemPromptBuilder, ReAct/Reflexion/HyDE
│   └── 11-complete-example.md        ← Full code-review agent with Docker Compose
│
└── python-langchain/                 ← Python + LangGraph/LangChain/LangSmith (13 files)
    ├── 00-index.md                   ← pip install, env vars, 30-line quick-start
    ├── 01-architecture.md            ← src layout, FastAPI, layer diagram
    ├── 02-core-agent-loop.md         ← StateGraph, ReAct, Reflexion, SELF-REFINE
    ├── 03-tool-system.md             ← @tool, ToolNode, MCP, parallel calls
    ├── 04-memory-and-rag.md          ← Checkpointer, Store, MemGPT, CRAG, HyDE
    ├── 05-knowledge-graph.md         ← networkx, Neo4j, LightRAG, HippoRAG PPR
    ├── 06-event-driven.md            ← EventStore, CQRS, SSE, WebSocket, Redis Streams
    ├── 07-multiagent.md              ← create_supervisor, handoff tools, swarm
    ├── 08-self-improvement.md        ← Reflexion loop, ExpeL, CRITIC, SELF-REFINE
    ├── 09-session-persistence.md     ← PostgreSQL checkpointer, forking, compaction
    ├── 10-prompts.md                 ← ChatPromptTemplate, Hub versioning, dynamic
    ├── 11-complete-example.md        ← Full research agent with Docker Compose
    └── 12-langsmith-observability.md ← Tracing, evaluation, datasets, LLM-as-judge
```

---

## arXiv Paper Reference Table

All 24 papers referenced across the implementation guides.

| Paper | arXiv ID | Year | Key Contribution |
|-------|----------|------|-----------------|
| **ReAct** | [2210.03629](https://arxiv.org/abs/2210.03629) | 2022 | Thought→Action→Observation loop; +34% ALFWorld |
| **Reflexion** | [2303.11366](https://arxiv.org/abs/2303.11366) | 2023 | Verbal RL via episodic reflection buffer; 91% HumanEval |
| **OpenHands** | [2407.16741](https://arxiv.org/abs/2407.16741) | 2024 | Event-sourced state machine, typed tool system |
| **MemGPT** | [2310.08560](https://arxiv.org/abs/2310.08560) | 2023 | 3-tier memory (in-context / recall / archival) |
| **CoALA** | [2309.02427](https://arxiv.org/abs/2309.02427) | 2023 | Working/episodic/semantic/procedural memory taxonomy |
| **A-MEM** | [2502.12110](https://arxiv.org/abs/2502.12110) | 2025 | Zettelkasten note graph + Ebbinghaus decay; 2× MemGPT |
| **Mem0** | [2504.19413](https://arxiv.org/abs/2504.19413) | 2025 | Extract→deduplicate→store→retrieve; 90% token reduction |
| **GraphRAG** | [2404.16130](https://arxiv.org/abs/2404.16130) | 2024 | Leiden community detection + LLM community summaries |
| **HippoRAG** | [2405.14831](https://arxiv.org/abs/2405.14831) | 2024 | KG + Personalized PageRank; +20% multi-hop QA |
| **LightRAG** | [2410.05779](https://arxiv.org/abs/2410.05779) | 2024 | Dual-level retrieval (local entity + global community) |
| **PathRAG** | [2502.14902](https://arxiv.org/abs/2502.14902) | 2025 | Flow-based path pruning on knowledge graphs |
| **Self-RAG** | [2310.11511](https://arxiv.org/abs/2310.11511) | 2023 | Reflection tokens [Retrieve][IsRel][IsSup][IsUse] |
| **CRAG** | [2401.15884](https://arxiv.org/abs/2401.15884) | 2024 | Confidence-scored retrieval: Correct/Ambiguous/Incorrect |
| **HyDE** | [2212.10496](https://arxiv.org/abs/2212.10496) | 2022 | Hypothetical doc embedding improves retrieval |
| **RAG-Fusion** | [2402.03367](https://arxiv.org/abs/2402.03367) | 2024 | Multi-query + Reciprocal Rank Fusion |
| **ESAA** | [2602.23193](https://arxiv.org/abs/2602.23193) | 2026 | Event Sourcing + CQRS for agents; append-only JSONL |
| **BGE-M3** | [2402.03216](https://arxiv.org/abs/2402.03216) | 2024 | Unified dense+sparse+multi-vector; 100+ languages |
| **ExpeL** | [2308.10144](https://arxiv.org/abs/2308.10144) | 2023 | Experience pool (vector DB) + LLM-distilled insights |
| **SELF-REFINE** | [selfrefine.info](https://selfrefine.info) | 2023 | Generate → critique → refine (capped iterations) |
| **CRITIC** | [2310.06825](https://arxiv.org/abs/2310.06825) | 2023 | Tool-integrated critique loop for self-correction |
| **AutoGen** | [2308.08155](https://arxiv.org/abs/2308.08155) | 2023 | Actor model — typed message handlers, pub-sub topics |
| **MetaGPT** | [2308.00352](https://arxiv.org/abs/2308.00352) | 2023 | SOP pipeline: PRD→Architecture→Code→Tests |
| **StreamingLLM** | [2309.17453](https://arxiv.org/abs/2309.17453) | 2023 | Attention sink tokens + sliding KV cache; 22.2× speedup |
| **LongMemEval** | [2410.10813](https://arxiv.org/abs/2410.10813) | 2024 | Decompose sessions to atomic facts for retrieval |

---

## Quick-Start by Language

### C++23 (15 minutes to running)

```bash
# Prerequisites: CMake ≥ 3.28, Clang 17+ or GCC 13+, vcpkg
git clone https://github.com/your-org/agent-cpp23
cd agent-cpp23
cmake -B build -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake
cmake --build build -j$(nproc)
OPENAI_API_KEY=sk-... ./build/agent "Explain monadic error handling in C++23"
```

**Key dependencies:**
```cmake
# vcpkg.json
{
  "dependencies": [
    "nlohmann-json", "cpp-httplib", "spdlog", "usearch",
    "boost-cobalt", "sqlite3", "sqlitecpp"
  ]
}
```

→ See [cpp23/00-index.md](cpp23/00-index.md)

---

### Kotlin + Koog (5 minutes to running)

```bash
# Prerequisites: JDK 17+, Gradle 8+
git clone https://github.com/your-org/agent-kotlin-koog
cd agent-kotlin-koog
./gradlew run --args="Explain coroutines in 2 sentences"
```

**Single dependency:**
```kotlin
// build.gradle.kts
implementation("ai.koog:koog-agents:0.8.0")
```

**30-second agent:**
```kotlin
suspend fun main() {
    val executor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY"))
    val agent = AIAgent(executor = executor, systemPrompt = "You are a helpful assistant.",
                        llmModel = OpenAIModels.Chat.GPT4o)
    println(agent.run("Hello!"))
}
```

→ See [kotlin-koog/00-index.md](kotlin-koog/00-index.md)

---

### Python + LangGraph (5 minutes to running)

```bash
# Prerequisites: Python 3.12+, uv or pip
pip install langchain langgraph langchain-openai python-dotenv
```

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

@tool
def get_weather(city: str) -> str:
    """Get weather for a city."""
    return f"Sunny, 22°C in {city}"

agent = create_react_agent(ChatOpenAI(model="gpt-4o-mini"), tools=[get_weather])
result = agent.invoke({"messages": [{"role": "user", "content": "Weather in Paris?"}]})
print(result["messages"][-1].content)
```

→ See [python-langchain/00-index.md](python-langchain/00-index.md)

---

## Cross-Language Decision Guide

| Requirement | C++23 | Kotlin/Koog | Python/LangGraph |
|-------------|-------|-------------|-----------------|
| **Latency-critical inference** | ✅ Best | 🟡 Good | 🟡 Good |
| **Local LLM (llama.cpp)** | ✅ Native | 🟡 Via Ollama | 🟡 Via Ollama |
| **Rapid prototyping** | ❌ Slow | 🟡 Moderate | ✅ Best |
| **Multiplatform (Android/iOS)** | ❌ No | ✅ KMP | ❌ No |
| **Spring Boot integration** | ❌ No | ✅ Official starters | ❌ No |
| **LangSmith observability** | ❌ Manual | 🟡 Via OTEL | ✅ Native |
| **RAG ecosystem breadth** | 🟡 Manual | 🟡 LangChain4j | ✅ Best |
| **Multi-agent (A2A)** | 🟡 Custom | ✅ Built-in | 🟡 Via supervisor |
| **Async/coroutines** | 🟡 Cobalt | ✅ Native suspend | ✅ asyncio |
| **Memory footprint** | ✅ Minimal | 🟡 JVM overhead | 🟡 Moderate |
| **Enterprise Java ecosystem** | ❌ No | ✅ Full JVM | ❌ No |
| **Production vector search** | ✅ USearch | ✅ Qdrant Java | ✅ Qdrant/pgvector |
| **Event sourcing (ESAA)** | ✅ Yes | ✅ Yes | ✅ Yes |
| **LangGraph Platform deploy** | ❌ No | ❌ No | ✅ Native |

**When to pick C++23:** Embedded systems, edge inference with llama.cpp, latency <50ms, or
integration into an existing C++ codebase.

**When to pick Kotlin/Koog:** Android/mobile, Spring Boot enterprise apps, JVM ecosystem
reuse, or when you want type-safe strategy graphs with compile-time validation.

**When to pick Python/LangGraph:** Fastest time-to-production, richest RAG/tool ecosystem,
LangSmith tracing from day one, LangGraph Platform deployment, or ML/data science teams.

---

## Universal Architecture Principles

These patterns apply across all three implementations:

### 1. ReAct Loop (arXiv:2210.03629)
Every agent core follows: **Thought → Action → Observation → repeat until Final Answer**.
The loop runs until a terminal condition (final answer, max iterations, timeout, or abort).

### 2. Three-Tier Memory (arXiv:2310.08560)
- **Working memory** — current context window (in-context)
- **Recall memory** — recent sessions, searchable by vector similarity
- **Archival memory** — all historical data, compressed by atomic facts

### 3. Event-Sourced State (arXiv:2602.23193)
State is never mutated in place; it is rebuilt by replaying an append-only event log.
Enables: deterministic replay, audit trails, time-travel debugging, and crash recovery.

### 4. Modular RAG Pipeline
```
Query → [HyDE expand] → [Multi-query] → [Dense + Sparse retrieve]
      → [Rerank] → [CRAG confidence gate] → [Context compress] → LLM
```

### 5. Permission Hierarchy
All tool calls are gated: Deny rules → Hook overrides → Ask rules → Allow rules → Mode default.
Never skip the permission check in production.

### 6. Three-Strategy Compaction
Use the right strategy for each token-pressure level:
- **MicroCompact** (75% fill) — compact only the oldest messages, preserve last N verbatim
- **Full Compact** (90%) — summarize all but the last 10 messages via a dedicated API call
- **ContextCollapse** (overflow) — strip content blocks from individual messages in-place
- **Circuit breaker** — disable compaction after 3 consecutive failures to prevent infinite retry loops
- **Tool-result budget** — evict the text of oldest tool results when combined length exceeds a char cap (e.g., 50 000 chars), keeping only the tool call metadata

### 7. Self-Improvement Pipeline
After each agent run (score-gated):
```
CRITIC verify → SELF-REFINE polish → Quality gate → 
ExpeL pool → SkillLibrary (if score ≥ 0.85)
```

### 8. Loop Detection
Heuristic: if the last N tool-call fingerprints are identical, the agent is looping.
LLM-based: after 30 turns, ask a cheap model with confidence threshold 0.9.
On detection: inject a meta-message, then abort or ask the user.

### 9. Tool Output Masking
When history tool results exceed a char threshold, replace them with placeholders in
the context sent to the LLM. The actual output remains stored locally. Saves tokens
without losing the data.

### 10. Concurrent Tool Execution
Read-only tools can run in parallel (up to 10× throughput on file-read-heavy turns).
Sequential execution is required for write tools to avoid file-write conflicts.

### 11. Model Routing
7-strategy composite router: CLI override → permission mode → local classifier →
cloud classifier → token count → fallback → default. Most agents need only 3:
CLI override → 429/529 fallback → default.

### 12. Planning Mode
Two mechanisms: **tool-based** (EnterPlanMode/ExitPlanMode tools, permission enforcement, plan file, user approval gate) for interactive agents; **extended thinking** (API-level, ephemeral, no approval gate) for automated pipelines. Tool-based plan mode enforces read-only at the permission layer — prompt instructions alone are not sufficient. For Manager-Executor topologies, use a ScratchpadGate to block write tools until the manager emits an unlock phrase in its output stream.

### 13. Hooks Lifecycle
27+ hook event types (PreToolUse, PostToolUse, SessionStart, PreCompact, PermissionRequest, etc.). Four hook types: command (shell stdin/stdout JSON), http (webhook POST), prompt (LLM-evaluated), agent (sub-agent verifier). PreToolUse hooks can ALLOW, DENY, ASK, or replace the tool input. Execution is sequential within a matcher group; first non-allow result wins.

### 14. Session JSONL with Branching
Every message has a `parentUuid` forming a linked list. Fork at any message to create a conversation branch — the branch is a new `.jsonl` file with the ancestor chain replayed. Sessions are indexed by `sha256(realpath(cwd))[:16]`. Archive after 90 days; auto-delete after 365. Use SQLite index for session listing at scale.

### 15. Cost & Budget Control
Track input, output, cache_read, and cache_write tokens separately per turn. Use a `SharedBudget` with an atomic lock across all agents in a multi-agent system. React to 429 with `retry-after` header; react to 529 with exponential backoff (max 60s). Never retry 400/401 — fix the request.

### 16. Distributed Agents (A2A)
Each remote agent publishes an Agent Card (JSON descriptor: capabilities, input/output schema, endpoint, auth type). Delegate tasks via structured JSON with streaming SSE response. Use `A2AResultReassembler` to collect `append: true` artifact chunks. Use short-lived JWT tokens for inter-agent auth. Remote agent failures inject structured error into tool result — never crash the orchestrator.

### 17. LSP Integration
Agents act as LSP **clients** (not servers) — spawn language servers (pyright, rust-analyzer, clangd) as subprocesses and query via JSON-RPC over stdio. Expose as a single `lsp` tool with an `operation` enum. Use lazy initialization (start servers on first file query, not at session start). Passively receive `publishDiagnostics` notifications and inject as context. Max file size: 10 MB.

### 18. Plugin System
Plugins provide: tools, slash commands, hooks, MCP servers, LSP servers, system prompt fragments. Manifest: `plugin.json` with `capabilities` declaration (`read_only`, `write`, `network`, `shell`, `agent`). Isolation: hooks and MCP/LSP servers run as subprocesses; skills (prompt fragments) run in-process. Load order: built-in → user-global → project-local. Hot-reload on `/reload-plugins`.

### 19. SDK Embedding
A public SDK exposes: `createSession()`, `resumeSession()`, `forkSession()`, `query()` (returns async event stream of `text`/`tool_use`/`tool_result`/`cost`/`done` events), custom tool registration via `defineTool()`, and an MCP server wrapper to expose the agent itself as an MCP tool provider. SDK runs headlessly — no REPL, no stdin required.

### 20. MCP Complete Reference
6 transport types (stdio, HTTP, SSE, WebSocket, SDK in-process, managed proxy). Initialization: `initialize` → `initialized` → `tools/list`. Tool naming: `mcp__{server}__{tool}` with underscore normalization. Three resource types: tools (callable), resources (file-like), prompts (templates). OAuth 2.0 via metadata discovery. Status lifecycle: pending → connected → needs-auth → failed → disabled.

---

## Reading Order

**New to agents:**
`00-research-foundation.md` → `{lang}/00-index.md` → `{lang}/02-core-agent-loop.md`

**Building a production RAG system:**
`{lang}/04-memory-and-rag.md` → `{lang}/05-knowledge-graph.md` → `{lang}/09-session-persistence.md`

**Multi-agent system:**
`{lang}/02-core-agent-loop.md` → `{lang}/07-multiagent.md` → `{lang}/06-event-driven.md`

**Self-improving agent:**
`{lang}/08-self-improvement.md` → `{lang}/03-tool-system.md` → `{lang}/11-complete-example.md`

**Production deployment:**
`{lang}/01-architecture.md` → `{lang}/09-session-persistence.md` → `{lang}/06-event-driven.md`
→ `python-langchain/12-langsmith-observability.md`

**Building task/todo tracking into your agent:**
`04-todo-task-management.md` → `{lang}/09-session-persistence.md` → `{lang}/07-multiagent.md`

**Adding planning mode:**
`01-universal-architecture.md` §19 → `03-reference-implementations.md` (permission checker) → `04-todo-task-management.md`

**Production hooks, sessions, cost:**
`01-universal-architecture.md` §20 (hooks) → §21 (sessions) → §22 (cost)

**IDE integration:**
`01-universal-architecture.md` §23 → `03-reference-implementations.md` (tool output masking)

**Distributed multi-agent:**
`01-universal-architecture.md` §24 (A2A) → `{lang}/07-multiagent.md`

**LSP / code intelligence:**
`01-universal-architecture.md` §25 (LSP client, JSON-RPC, diagnostics injection)

**Plugins & extensions:**
`01-universal-architecture.md` §26 (plugin manifest, capabilities, isolation model)

**SDK embedding:**
`01-universal-architecture.md` §27 (query API, custom tools, MCP server wrapper, headless CI)

**MCP complete reference:**
`01-universal-architecture.md` §28 (6 transports, OAuth, resources, prompts, agent-as-server)

---

## Infrastructure Quick Reference

### Minimum Docker Compose Stack

```yaml
# docker-compose.yml (works for all three language agents)
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: agentdb
      POSTGRES_USER: agent
      POSTGRES_PASSWORD: secret
    ports: ["5432:5432"]

  qdrant:
    image: qdrant/qdrant:v1.9.0
    ports: ["6333:6333", "6334:6334"]
    volumes: ["qdrant_data:/qdrant/storage"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  neo4j:
    image: neo4j:5.19
    environment:
      NEO4J_AUTH: neo4j/secret
    ports: ["7474:7474", "7687:7687"]

  ollama:
    image: ollama/ollama:latest
    ports: ["11434:11434"]
    volumes: ["ollama_data:/root/.ollama"]

volumes:
  qdrant_data:
  ollama_data:
```

---

## Contributing

When adding guidelines for a new language or framework:

1. Create `{language}/` directory
2. Follow the same 12-file structure
3. Add all 24 arXiv paper citations in the index
4. Update this README with the new directory entry
5. Update the decision guide table
