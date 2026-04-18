# Kotlin Koog Agent System — Implementation Guidelines

**Framework**: Koog v0.8.0 (JetBrains)  
**Language**: Kotlin 2.3.10+  
**Runtime**: JDK 17+  
**Last Updated**: April 2026

---

## Document Index

| # | File | Topics |
|---|------|--------|
| 01 | [architecture.md](./01-architecture.md) | Module structure, Gradle build, provider config, layer diagram |
| 02 | [core-agent-loop.md](./02-core-agent-loop.md) | Strategy graph, ReAct, Reflexion, SELF-REFINE, streaming |
| 03 | [tool-system.md](./03-tool-system.md) | @Tool, ToolSet, ToolRegistry, parallel execution, MCP |
| 04 | [memory-and-rag.md](./04-memory-and-rag.md) | ChatMemory, MemGPT, Qdrant, HyDE, RAG-Fusion, CRAG |
| 05 | [knowledge-graph.md](./05-knowledge-graph.md) | Neo4j, HippoRAG, LightRAG, GraphRAG, PathRAG |
| 06 | [event-driven.md](./06-event-driven.md) | Event sourcing, CQRS, Koog EventHandler, OpenTelemetry |
| 07 | [multiagent.md](./07-multiagent.md) | A2A protocol, SupervisorAgent, AutoGen, MetaGPT SOP |
| 08 | [self-improvement.md](./08-self-improvement.md) | Reflexion, ExpeL, CRITIC, SELF-REFINE, skill library |
| 09 | [session-persistence.md](./09-session-persistence.md) | Koog Persistence, RollbackRegistry, compaction, resume |
| 10 | [prompts.md](./10-prompts.md) | SystemPromptBuilder, ReAct templates, HyDE, KG extraction |
| 11 | [complete-example.md](./11-complete-example.md) | Full software-dev agent, Docker Compose, observability |

---

## Quick Reference

### Gradle Dependency (single line)

```kotlin
implementation("ai.koog:koog-agents:0.8.0")
```

### Minimal Agent (30 seconds to running)

```kotlin
import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.core.agent.config.AIAgentConfig
import ai.koog.prompt.executor.clients.openai.OpenAILLMClient
import ai.koog.prompt.executor.llms.all.simpleOpenAIExecutor

suspend fun main() {
    val executor = simpleOpenAIExecutor(System.getenv("OPENAI_API_KEY"))
    val agent = AIAgent(
        executor = executor,
        systemPrompt = "You are a helpful assistant.",
        llmModel = OpenAIModels.Chat.GPT4o,
    )
    val result = agent.run("Explain coroutines in 2 sentences.")
    println(result)
}
```

### Provider Quick-Config

```kotlin
// OpenAI
val executor = simpleOpenAIExecutor(apiKey = System.getenv("OPENAI_API_KEY"))

// Anthropic Claude
val executor = simpleAnthropicExecutor(apiKey = System.getenv("ANTHROPIC_API_KEY"))

// Google Gemini
val executor = simpleGoogleExecutor(apiKey = System.getenv("GOOGLE_API_KEY"))

// Ollama (local)
val executor = simpleOllamaExecutor(baseUrl = "http://localhost:11434")

// DeepSeek
val executor = simpleDeepSeekExecutor(apiKey = System.getenv("DEEPSEEK_API_KEY"))

// Azure OpenAI
val executor = simpleAzureOpenAIExecutor(
    apiKey = System.getenv("AZURE_OPENAI_API_KEY"),
    endpoint = System.getenv("AZURE_OPENAI_ENDPOINT"),
    deploymentName = "gpt-4o"
)

// OpenRouter (access 100+ models)
val executor = simpleOpenRouterExecutor(apiKey = System.getenv("OPENROUTER_API_KEY"))
```

---

## Gradle Bootstrap (build.gradle.kts)

```kotlin
plugins {
    kotlin("jvm") version "2.3.10"
    kotlin("plugin.serialization") version "2.3.10"
    application
}

group = "com.example.agent"
version = "1.0.0"

repositories {
    mavenCentral()
    // Koog snapshots if needed
    maven("https://packages.jetbrains.team/maven/p/koog/")
}

val koogVersion = "0.8.0"
val coroutinesVersion = "1.10.2"
val ktorVersion = "3.1.2"
val exposedVersion = "0.60.0"

dependencies {
    // Core Koog
    implementation("ai.koog:koog-agents:$koogVersion")

    // Provider clients (include only what you need)
    implementation("ai.koog:koog-agents-openai:$koogVersion")
    implementation("ai.koog:koog-agents-anthropic:$koogVersion")
    implementation("ai.koog:koog-agents-google:$koogVersion")
    implementation("ai.koog:koog-agents-ollama:$koogVersion")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:$coroutinesVersion")

    // Serialization
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.8.0")

    // Ktor server (for A2A, streaming, REST API)
    implementation("io.ktor:ktor-server-netty:$ktorVersion")
    implementation("io.ktor:ktor-server-content-negotiation:$ktorVersion")
    implementation("io.ktor:ktor-serialization-kotlinx-json:$ktorVersion")
    implementation("io.ktor:ktor-server-sse:$ktorVersion")

    // Ktor client (for outbound HTTP)
    implementation("io.ktor:ktor-client-cio:$ktorVersion")
    implementation("io.ktor:ktor-client-content-negotiation:$ktorVersion")

    // Database — SQLite (lightweight, dev/embedded)
    implementation("org.xerial:sqlite-jdbc:3.45.3.0")

    // Database — Postgres (production)
    implementation("org.postgresql:postgresql:42.7.3")

    // Kotlin Exposed ORM
    implementation("org.jetbrains.exposed:exposed-core:$exposedVersion")
    implementation("org.jetbrains.exposed:exposed-dao:$exposedVersion")
    implementation("org.jetbrains.exposed:exposed-jdbc:$exposedVersion")
    implementation("org.jetbrains.exposed:exposed-json:$exposedVersion")
    implementation("org.jetbrains.exposed:exposed-kotlin-datetime:$exposedVersion")

    // Qdrant vector database client
    implementation("io.qdrant:client:1.9.0")

    // Neo4j (knowledge graph)
    implementation("org.neo4j.driver:neo4j-java-driver:5.19.0")

    // OpenTelemetry (observability)
    implementation("io.opentelemetry:opentelemetry-api:1.38.0")
    implementation("io.opentelemetry:opentelemetry-sdk:1.38.0")
    implementation("io.opentelemetry:opentelemetry-exporter-otlp:1.38.0")

    // Logging
    implementation("io.github.oshai:kotlin-logging-jvm:7.0.0")
    implementation("ch.qos.logback:logback-classic:1.5.6")

    // Config
    implementation("com.sksamuel.hoplite:hoplite-core:2.8.0")
    implementation("com.sksamuel.hoplite:hoplite-yaml:2.8.0")

    // Testing
    testImplementation("org.jetbrains.kotlin:kotlin-test-junit5")
    testImplementation("io.mockk:mockk:1.13.11")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:$coroutinesVersion")
}

application {
    mainClass.set("com.example.agent.ApplicationKt")
}

kotlin {
    jvmToolchain(17)
}
```

---

## Environment Variables Reference

```bash
# LLM Providers
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_API_KEY=AIza...
DEEPSEEK_API_KEY=...
OPENROUTER_API_KEY=sk-or-...
AZURE_OPENAI_API_KEY=...
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/

# Databases
POSTGRES_URL=jdbc:postgresql://localhost:5432/agents
POSTGRES_USER=agent
POSTGRES_PASSWORD=secret
SQLITE_PATH=./data/agent.db

# Vector Store
QDRANT_HOST=localhost
QDRANT_PORT=6334
QDRANT_API_KEY=          # empty for local dev

# Knowledge Graph
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=secret

# Observability
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_HOST=https://cloud.langfuse.com
```

---

## Architecture Decision Quick Reference

| Decision | Recommendation |
|----------|---------------|
| Simple Q&A agent | `AIAgent` (Simple archetype) |
| Multi-step reasoning | Graph-based strategy |
| Planning with decomposition | Planner archetype (LLM or GOAP) |
| Multi-agent coordination | A2A protocol + `nodeA2AClientSendMessage` |
| Local embedding/inference | Ollama provider |
| Production vector search | Qdrant (Java client or langchain4j) |
| Session persistence | `PostgresJdbcPersistenceStorageProvider` |
| Lightweight / dev persistence | `KotlinxSQLiteMemoryStore` |
| MCP tool integration | Koog MCP client or LangChain4j MCP |
| Spring Boot app | Koog Spring AI starters |
| Standalone JVM app | Koog + Ktor |

---

## Paper Citations

All architectural patterns implemented in these guidelines are grounded in peer-reviewed research:

- ReAct: arXiv:2210.03629 (Yao et al., 2022)
- Reflexion: arXiv:2303.11366 (Shinn et al., 2023)
- MemGPT: arXiv:2310.08560 (Packer et al., 2023)
- CoALA: arXiv:2309.02427 (Sumers et al., 2023)
- A-MEM: arXiv:2502.12110 (Xu et al., 2025)
- ExpeL: arXiv:2308.10144 (Zhao et al., 2023)
- SELF-REFINE: selfrefine.info (Madaan et al., 2023)
- HyDE: arXiv:2212.10496 (Gao et al., 2022)
- RAG-Fusion + RRF: arXiv:2402.03367
- CRAG: arXiv:2401.15884 (Yan et al., 2024)
- Self-RAG: arXiv:2310.11511 (Asai et al., 2023)
- BGE-M3: arXiv:2402.03216 (Chen et al., 2024)
- HippoRAG: arXiv:2405.14831 (Guo et al., 2024)
- LightRAG: arXiv:2410.05779 (Edge et al., 2024)
- GraphRAG: arXiv:2404.16130 (Edge et al., 2024)
- PathRAG: arXiv:2502.14902 (Chen et al., 2025)
- Event sourcing agents: arXiv:2602.23193 (2026)
