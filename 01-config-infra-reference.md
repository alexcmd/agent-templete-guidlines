# Universal Agent Architecture — Config, Infrastructure & Reference

> Sections: Configuration Hierarchy · Cross-Language Map · Infrastructure Stack · arXiv Papers

Part of the [Universal Agent Architecture](01-overview.md) series.

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


---

*[← Advanced Patterns](01-advanced-patterns.md) | [Next: Planning & Hooks →](01-planning-hooks.md)*
