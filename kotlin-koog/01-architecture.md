# Architecture — Koog Agent System

## Layer Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│  UI / API Layer                                                  │
│  Ktor REST  │  SSE Streaming  │  A2A Server  │  Spring Boot MVC │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│  Strategy Graph (Koog)                                           │
│  nodeLLMRequest → nodeExecuteTool → nodeExecuteMultipleTools     │
│  ReAct Loop │ Reflexion │ SELF-REFINE │ Planner                  │
└──────────┬───────────────┬───────────────────────────────────────┘
           │               │
┌──────────▼──────┐  ┌─────▼───────────────────────────────────── ┐
│  Tool System    │  │  Memory & RAG                               │
│  @Tool funs     │  │  ChatMemory (SQLite/Postgres)               │
│  ToolSet        │  │  LongTermMemory (Qdrant vectors)            │
│  ToolRegistry   │  │  MemGPT 3-tier │ A-MEM note graph           │
│  MCP client     │  │  Modular RAG pipeline                       │
└──────────┬──────┘  └─────┬───────────────────────────────────────┘
           │               │
┌──────────▼───────────────▼───────────────────────────────────────┐
│  Knowledge Graph                                                 │
│  Neo4j (bolt) │ In-memory adjacency list                        │
│  HippoRAG PPR │ LightRAG dual-level │ GraphRAG Leiden            │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│  Multi-Agent Coordination                                        │
│  A2A Protocol (AgentCard, AgentExecutor, A2AServer)              │
│  SupervisorAgent │ AutoGen actor model │ MetaGPT SOP             │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│  Observability & Persistence                                     │
│  EventHandler → OpenTelemetry → Langfuse                         │
│  PostgresJdbcPersistenceStorageProvider │ RollbackToolRegistry   │
│  Event sourcing JSONL │ CQRS StateFlow projections               │
└──────────────────────────────────────────────────────────────────┘
```

---

## Module / Package Structure

```
com.example.agent/
├── Application.kt                   # Entry point, DI wiring
├── config/
│   ├── AppConfig.kt                 # Hoplite config data class
│   └── ProviderConfig.kt            # LLM provider factory
├── agent/
│   ├── SimpleAgent.kt               # Thin wrapper for AIAgent
│   ├── GraphAgent.kt                # Strategy graph definition
│   ├── ReActAgent.kt                # ReAct loop specialization
│   └── PlannerAgent.kt              # Koog Planner archetype
├── tools/
│   ├── ToolRegistryFactory.kt       # Assembles ToolRegistry
│   ├── FileTools.kt                 # @Tool: readFile, writeFile
│   ├── BashTool.kt                  # @Tool: bash execution
│   ├── WebSearchTool.kt             # @Tool: HTTP search
│   └── DatabaseTools.kt             # @Tool: JDBC queries
├── memory/
│   ├── ChatMemoryFactory.kt         # SQLite / Postgres factory
│   ├── LongTermMemory.kt            # Qdrant vector store wrapper
│   ├── MemGPTTiers.kt               # 3-tier MemGPT implementation
│   └── AMemNoteGraph.kt             # A-MEM Zettelkasten
├── rag/
│   ├── RagPipeline.kt               # Modular interface chain
│   ├── QueryRewriter.kt             # HyDE and query expansion
│   ├── HybridRetriever.kt           # Dense + sparse (BGE-M3)
│   ├── Reranker.kt                  # Cross-encoder reranking
│   ├── ContextCompressor.kt         # LLM-based compression
│   ├── RagFusion.kt                 # RRF scoring
│   └── CragRetriever.kt             # Confidence-scored + fallback
├── kg/
│   ├── KgModels.kt                  # Entity, Relation, KnowledgeGraph
│   ├── TripleExtractor.kt           # Koog agent for extraction
│   ├── Neo4jStore.kt                # bolt-java driver wrapper
│   ├── HippoRag.kt                  # PPR-based retrieval
│   └── GraphRag.kt                  # Community detection
├── events/
│   ├── EventModels.kt               # Sealed EventType hierarchy
│   ├── EventStore.kt                # Append-only JSONL / Exposed
│   ├── EventBus.kt                  # Channel<Event> bus
│   └── OtelObservability.kt         # OpenTelemetry exporter
├── multiagent/
│   ├── A2AServerSetup.kt            # AgentCard + A2AServer
│   ├── SupervisorAgent.kt           # Manager + specialist handoff
│   ├── TeamRegistry.kt              # Agent roster + dispatch
│   └── MetaGptSop.kt                # Role-based SOP pipeline
├── persistence/
│   ├── SessionManager.kt            # Koog Persistence.Feature
│   ├── RollbackRegistry.kt          # RollbackToolRegistry config
│   └── CompactionStrategy.kt        # Sliding window + summary
└── prompts/
    ├── SystemPromptBuilder.kt        # Composable prompt builder
    └── PromptTemplates.kt            # ReAct, Reflexion, HyDE strings
```

---

## Koog vs LangChain4j Decision Guide

Use **Koog** when:
- You want idiomatic Kotlin coroutines throughout (suspend funs, Flow, Channel)
- You need multiplatform support (Android, iOS, JS)
- You want type-safe strategy graphs with compile-time edge validation
- You require the A2A protocol for inter-agent communication
- You are integrating with Spring AI via the official starters
- You need the Koog Planner archetype (LLM or GOAP)

Use **LangChain4j** when:
- Your team already has LangChain4j expertise
- You need one of the 30+ embedding store integrations not yet in Koog
- You need the full LangChain4j MCP client ecosystem
- You want Java interop with existing enterprise Java code
- You need more mature RAG chain abstractions out of the box

**Hybrid approach** (recommended for production):
- Koog handles agent strategy, tool execution, A2A, and session persistence
- LangChain4j provides embedding models, vector store clients (Qdrant, pgvector), and MCP tools
- Both coexist on JVM without conflict

---

## Provider Configuration

```kotlin
// config/ProviderConfig.kt
package com.example.agent.config

import ai.koog.prompt.executor.clients.anthropic.AnthropicLLMClient
import ai.koog.prompt.executor.clients.google.GoogleLLMClient
import ai.koog.prompt.executor.clients.openai.OpenAILLMClient
import ai.koog.prompt.executor.llms.MultiLLMPromptExecutor
import ai.koog.prompt.executor.llms.all.*
import ai.koog.prompt.llm.LLMProvider

enum class ProviderType { OPENAI, ANTHROPIC, GOOGLE, OLLAMA, DEEPSEEK, OPENROUTER, AZURE }

data class ProviderConfig(
    val type: ProviderType,
    val apiKey: String = "",
    val baseUrl: String = "",
    val deploymentName: String = "",  // Azure
    val model: String = ""
)

fun buildExecutor(cfg: ProviderConfig) = when (cfg.type) {
    ProviderType.OPENAI       -> simpleOpenAIExecutor(cfg.apiKey)
    ProviderType.ANTHROPIC    -> simpleAnthropicExecutor(cfg.apiKey)
    ProviderType.GOOGLE       -> simpleGoogleExecutor(cfg.apiKey)
    ProviderType.OLLAMA       -> simpleOllamaExecutor(cfg.baseUrl.ifEmpty { "http://localhost:11434" })
    ProviderType.DEEPSEEK     -> simpleDeepSeekExecutor(cfg.apiKey)
    ProviderType.OPENROUTER   -> simpleOpenRouterExecutor(cfg.apiKey)
    ProviderType.AZURE        -> simpleAzureOpenAIExecutor(
        apiKey = cfg.apiKey,
        endpoint = cfg.baseUrl,
        deploymentName = cfg.deploymentName
    )
}

/** Multi-provider executor: route by model prefix */
fun buildMultiExecutor(vararg configs: ProviderConfig): MultiLLMPromptExecutor {
    val clients = configs.map { cfg ->
        when (cfg.type) {
            ProviderType.OPENAI    -> OpenAILLMClient(cfg.apiKey)
            ProviderType.ANTHROPIC -> AnthropicLLMClient(cfg.apiKey)
            ProviderType.GOOGLE    -> GoogleLLMClient(cfg.apiKey)
            else -> throw IllegalArgumentException("Multi-executor: unsupported provider ${cfg.type}")
        }
    }
    return MultiLLMPromptExecutor(clients)
}
```

---

## Spring Boot Integration

Koog ships four Spring AI starters. Add to your Spring Boot project:

```kotlin
// build.gradle.kts (Spring Boot additions)
plugins {
    id("org.springframework.boot") version "3.3.2"
    id("io.spring.dependency-management") version "1.1.5"
}

dependencies {
    // Koog Spring AI starters
    implementation("ai.koog:koog-spring-ai-model-chat:0.8.0")
    implementation("ai.koog:koog-spring-ai-model-embedding:0.8.0")
    implementation("ai.koog:koog-spring-ai-chat-memory:0.8.0")
    implementation("ai.koog:koog-spring-ai-vector-store:0.8.0")

    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-webflux")  // for streaming
}
```

```yaml
# application.yml
spring:
  ai:
    koog:
      provider: openai
      api-key: ${OPENAI_API_KEY}
      model: gpt-4o
    koog-memory:
      store: sqlite
      path: ./data/agent.db
```

```kotlin
// Spring controller using Koog chat
@RestController
@RequestMapping("/api/agent")
class AgentController(
    private val chatModel: ChatModel,         // injected by koog-spring-ai-model-chat
    private val chatMemory: ChatMemory        // injected by koog-spring-ai-chat-memory
) {
    @PostMapping("/chat")
    suspend fun chat(@RequestBody request: ChatRequest): ChatResponse {
        val response = chatModel.call(
            Prompt(
                listOf(UserMessage(request.message)),
                ChatOptions.builder().model("gpt-4o").build()
            )
        )
        return ChatResponse(response.result.output.content)
    }
}
```

---

## Standalone Ktor Application

```kotlin
// Application.kt (standalone, no Spring)
package com.example.agent

import com.example.agent.config.AppConfig
import com.example.agent.config.buildExecutor
import com.example.agent.tools.ToolRegistryFactory
import io.ktor.server.application.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import io.ktor.server.routing.*
import com.sksamuel.hoplite.ConfigLoaderBuilder
import com.sksamuel.hoplite.addResourceSource

fun main() {
    val config = ConfigLoaderBuilder.default()
        .addResourceSource("/application.yaml")
        .build()
        .loadConfigOrThrow<AppConfig>()

    embeddedServer(Netty, port = config.server.port) {
        configureRouting(config)
    }.start(wait = true)
}

fun Application.configureRouting(config: AppConfig) {
    val executor = buildExecutor(config.provider)
    val toolRegistry = ToolRegistryFactory.build()

    routing {
        agentRoutes(executor, toolRegistry)
    }
}
```

---

## AppConfig with Hoplite

```kotlin
// config/AppConfig.kt
package com.example.agent.config

import com.sksamuel.hoplite.ConfigAlias

data class AppConfig(
    val server: ServerConfig,
    val provider: ProviderConfig,
    val database: DatabaseConfig,
    val qdrant: QdrantConfig,
    val neo4j: Neo4jConfig,
    val otel: OtelConfig
)

data class ServerConfig(val port: Int = 8080)

data class DatabaseConfig(
    val url: String,
    val user: String,
    val password: String,
    @ConfigAlias("pool-size") val poolSize: Int = 10
)

data class QdrantConfig(
    val host: String = "localhost",
    val port: Int = 6334,
    @ConfigAlias("api-key") val apiKey: String = "",
    val collection: String = "agent-memory"
)

data class Neo4jConfig(
    val uri: String = "bolt://localhost:7687",
    val user: String = "neo4j",
    val password: String
)

data class OtelConfig(
    val endpoint: String = "http://localhost:4317",
    @ConfigAlias("langfuse-secret") val langfuseSecret: String = "",
    @ConfigAlias("langfuse-public") val langfusePublic: String = ""
)
```

```yaml
# src/main/resources/application.yaml
server:
  port: 8080

provider:
  type: OPENAI
  api-key: ${OPENAI_API_KEY}

database:
  url: ${POSTGRES_URL:jdbc:sqlite:./data/agent.db}
  user: ${POSTGRES_USER:}
  password: ${POSTGRES_PASSWORD:}
  pool-size: 5

qdrant:
  host: ${QDRANT_HOST:localhost}
  port: ${QDRANT_PORT:6334}
  api-key: ${QDRANT_API_KEY:}
  collection: agent-memory

neo4j:
  uri: ${NEO4J_URI:bolt://localhost:7687}
  user: ${NEO4J_USER:neo4j}
  password: ${NEO4J_PASSWORD:secret}

otel:
  endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT:http://localhost:4317}
  langfuse-secret: ${LANGFUSE_SECRET_KEY:}
  langfuse-public: ${LANGFUSE_PUBLIC_KEY:}
```

---

## Four Agent Archetypes (Summary)

```kotlin
// 1. Simple AIAgent — prototyping, one-shot tasks
val agent = AIAgent(executor, systemPrompt = "...", llmModel = OpenAIModels.Chat.GPT4o)
val result = agent.run(userInput)

// 2. Functional Agent — thin wrapper with custom logic
val agent = functionalAgent(executor, model = AnthropicModels.Claude35Sonnet) {
    // custom pre/post hooks
}

// 3. Graph-Based Agent — production multi-step (see 02-core-agent-loop.md)
val agent = AIAgent(
    executor = executor,
    agentConfig = AIAgentConfig(prompt = PromptConfig(systemPrompt), tools = toolRegistry),
    strategy = buildReActStrategy(toolRegistry)
)

// 4. Planner Agent — task decomposition
val agent = plannerAgent(
    executor = executor,
    tools = toolRegistry,
    plannerType = PlannerType.LLM   // or PlannerType.GOAP
)
```

---

## Key Interfaces to Know

```kotlin
// The core interfaces you implement or compose:

interface PromptExecutor          // wraps LLM client calls
interface AIAgentStrategy         // defines the strategy graph
interface Tool                    // single callable tool
interface ToolSet                 // grouped set of tools
interface ChatMemory              // conversation history store
interface LongTermMemory          // persistent factual memory
interface EmbeddingProvider       // turns text → float vector
interface VectorStore             // ANN search store
interface EventHandler            // lifecycle event callbacks
```

All are suspend-friendly; most accept `CoroutineContext` for cancellation propagation.
