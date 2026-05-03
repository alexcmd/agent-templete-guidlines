# C++23 LLM Agent System — Implementation Guidelines Index

> Production-grade reference for building modern LLM agent systems in C++23.
> Covers the full stack: ReAct loops, tool dispatch, RAG pipelines, knowledge graphs,
> multi-agent coordination, event sourcing, and self-improvement mechanisms.

---

## Document Map

| File | Topic | Key Concepts |
|------|-------|-------------|
| [01-architecture.md](01-architecture.md) | System Architecture | Module layout, CMake, dependency graph, C++23 rationale |
| [02-core-agent-loop.md](02-core-agent-loop.md) | ReAct Agent Loop | AgentContext, std::expected chains, SSE streaming, Reflexion, SELF-REFINE |
| [03-tool-system.md](03-tool-system.md) | Tool System | ToolRegistry, type-erasure, parallel dispatch, DFSDT, AnyTool |
| [04-memory-and-rag.md](04-memory-and-rag.md) | Memory & RAG | MemGPT tiers, USearch HNSW, HyDE, RAG-Fusion, CRAG, A-MEM |
| [05-knowledge-graph.md](05-knowledge-graph.md) | Knowledge Graph | Triple extraction, HippoRAG PPR, LightRAG, GraphRAG, PathRAG |
| [06-event-driven.md](06-event-driven.md) | Event-Driven Architecture | Event sourcing, CQRS, pub-sub, hook lifecycle, event replay |
| [07-multiagent.md](07-multiagent.md) | Multi-Agent Patterns | Actor model, AutoGen topology, MetaGPT SOP, CAMEL, Reflexion split |
| [08-self-improvement.md](08-self-improvement.md) | Self-Improvement | ExpeL, CRITIC, SELF-REFINE, skill library, Voyager pattern |
| [09-session-persistence.md](09-session-persistence.md) | Sessions & Persistence | JSONL serialization, compaction, SQLite, config loading |
| [10-prompts.md](10-prompts.md) | Prompt Engineering | SystemPromptBuilder, ReAct templates, HyDE, KG extraction, caching |
| [11-complete-example.md](11-complete-example.md) | Full Working Example | main.cpp, CMakeLists.txt, all subsystems wired, build instructions |
| [12-testing.md](12-testing.md) | Testing | GoogleTest unit tests, GMock tool handlers, integration tests (ANTHROPIC_API_KEY gate), eval benchmark (ContainsAll/ToolUsed scorers), CMake config |

---

## Quick Reference

### Core Types

```cpp
// Primary error-propagation type throughout the system
using AgentResult<T> = std::expected<T, AgentError>;

// Event log entry (append-only)
struct Event { EventType type; nlohmann::json payload; std::string timestamp; };

// Tool invocation descriptor
struct ToolCall { std::string name; nlohmann::json arguments; std::string call_id; };

// Single conversation turn
struct Message { Role role; std::string content; std::vector<ToolCall> tool_calls; };
```

### Key Architectural Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Error propagation | `std::expected<T,E>` | Zero-cost on happy path; monadic composition via `.and_then()` |
| Tool map | `std::flat_map` | Cache-friendly sorted array; 2–4x faster lookup for small registries |
| Async runtime | Boost.Cobalt over Asio | `co_await`-native; structured concurrency; no manual thread management |
| JSON | nlohmann/json + simdjson | nlohmann for mutation; simdjson for high-throughput parsing of LLM responses |
| HTTP / SSE | cpp-httplib | Header-only; built-in SSE chunked-transfer support |
| Vector search | USearch HNSW | Single-header; 10x faster than FAISS on recall@10 benchmarks |
| Serialization | glaze (reflection) | C++23 reflection-based; zero-boilerplate round-trip |

### Dependency Versions (as of 2026-04)

```
llama.cpp          >= b4500   (OpenAI-compatible server: llama-server)
openai-cpp (olrea) >= 1.4.0
nlohmann/json      >= 3.11.3
simdjson           >= 3.9.0
glaze              >= 3.3.0
cpp-httplib        >= 0.16.0
Boost              >= 1.87.0  (Cobalt + Asio)
USearch            >= 2.14.0
SQLiteCpp          >= 3.3.2
spdlog             >= 1.14.0
```

### Compiler Requirements

```cmake
set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
# GCC >= 14 or Clang >= 17 or MSVC >= 19.38
```

---

## Reading Order

**For a new implementation**, read in order 01 → 02 → 03 → 11 (build the skeleton first),
then layer in 04–09 as needed.

**For specific subsystems:**
- Adding a new tool: 03-tool-system.md § ToolSpec and registry registration
- Wiring memory: 04-memory-and-rag.md § MemGPT 3-tier model
- Debugging agent state: 06-event-driven.md § Event replay
- Scaling to multiple agents: 07-multiagent.md § Actor model

---

## Research Paper Citations

All major design decisions are grounded in peer-reviewed work:

- ReAct (arXiv:2210.03629) — Thought/Action/Observation loop
- Reflexion (arXiv:2303.11366) — Verbal reinforcement learning via episodic memory
- MemGPT (arXiv:2310.08560) — Hierarchical memory tiers
- CoALA (arXiv:2309.02427) — Cognitive architecture for language agents
- ExpeL (arXiv:2308.10144) — Experience pool from trajectories
- SELF-REFINE (selfrefine.info) — Generate → critique → refine
- HyDE (arXiv:2212.10496) — Hypothetical document embeddings
- RAG-Fusion (arXiv:2402.03367) — Multi-query + Reciprocal Rank Fusion
- CRAG (arXiv:2401.15884) — Corrective RAG with confidence scoring
- Self-RAG (arXiv:2310.11511) — Reflection tokens for retrieval decisions
- HippoRAG (arXiv:2405.14831) — KG + Personalized PageRank
- LightRAG (arXiv:2410.05779) — Dual-level KG retrieval
- GraphRAG (arXiv:2404.16130) — Community detection + summaries
- PathRAG (arXiv:2502.14902) — Flow-based KG path pruning
- A-MEM (arXiv:2502.12110) — Zettelkasten note graph with decay
- MemoryBank (arXiv:2305.10250) — Ebbinghaus forgetting curve
- LongMemEval (arXiv:2410.10813) — Atomic fact decomposition
- StreamingLLM (arXiv:2309.17453) — Attention sink + sliding window KV
- Event sourcing for agents (arXiv:2602.23193) — Append-only state log

---

## Project Layout (Top-Level)

```
agent-system/
├── CMakeLists.txt
├── cmake/
│   ├── FetchDependencies.cmake
│   └── CompilerFlags.cmake
├── src/
│   ├── main.cpp
│   ├── agent/          # Core loop, context, ReAct
│   ├── tools/          # Registry, built-in tools
│   ├── memory/         # MemGPT tiers, USearch wrapper
│   ├── rag/            # RAG pipeline stages
│   ├── kg/             # Knowledge graph
│   ├── events/         # Event store, CQRS
│   ├── multiagent/     # Actor model, pub-sub
│   ├── session/        # Persistence, compaction
│   └── prompts/        # Prompt builders
├── tests/
│   ├── unit/
│   └── integration/
├── config/
│   └── default.json
└── third_party/        # Vendored single-header libs
```
