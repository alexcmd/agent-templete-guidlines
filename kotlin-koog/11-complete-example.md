# Complete Example: Autonomous Software Development Agent

This document contains a full, runnable implementation of an autonomous software development agent. It combines all subsystems: multi-agent coordination, RAG over codebase docs, knowledge graph for symbol relationships, event sourcing, and A2A protocol exposure.

---

## build.gradle.kts

```kotlin
plugins {
    kotlin("jvm") version "2.3.10"
    kotlin("plugin.serialization") version "2.3.10"
    id("com.github.johnrengelman.shadow") version "8.1.1"
    application
}

group = "com.example.devagent"
version = "1.0.0"

repositories {
    mavenCentral()
    maven("https://packages.jetbrains.team/maven/p/koog/")
}

val koogVersion    = "0.8.0"
val coroutines     = "1.10.2"
val ktor           = "3.1.2"
val exposed        = "0.60.0"
val serialization  = "1.8.0"

dependencies {
    // Koog core + providers
    implementation("ai.koog:koog-agents:$koogVersion")
    implementation("ai.koog:koog-agents-openai:$koogVersion")
    implementation("ai.koog:koog-agents-anthropic:$koogVersion")
    implementation("ai.koog:koog-agents-ollama:$koogVersion")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:$coroutines")

    // Serialization
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:$serialization")

    // Ktor server (REST + SSE + A2A)
    implementation("io.ktor:ktor-server-netty:$ktor")
    implementation("io.ktor:ktor-server-content-negotiation:$ktor")
    implementation("io.ktor:ktor-serialization-kotlinx-json:$ktor")
    implementation("io.ktor:ktor-server-sse:$ktor")
    implementation("io.ktor:ktor-server-call-logging:$ktor")
    implementation("io.ktor:ktor-server-status-pages:$ktor")

    // Ktor client
    implementation("io.ktor:ktor-client-cio:$ktor")
    implementation("io.ktor:ktor-client-content-negotiation:$ktor")

    // Database
    implementation("org.xerial:sqlite-jdbc:3.45.3.0")
    implementation("org.postgresql:postgresql:42.7.3")
    implementation("org.jetbrains.exposed:exposed-core:$exposed")
    implementation("org.jetbrains.exposed:exposed-dao:$exposed")
    implementation("org.jetbrains.exposed:exposed-jdbc:$exposed")
    implementation("org.jetbrains.exposed:exposed-json:$exposed")
    implementation("org.jetbrains.exposed:exposed-kotlin-datetime:$exposed")
    implementation("com.zaxxer:HikariCP:5.1.0")

    // Vector store
    implementation("io.qdrant:client:1.9.0")

    // Neo4j
    implementation("org.neo4j.driver:neo4j-java-driver:5.19.0")

    // OpenTelemetry
    implementation("io.opentelemetry:opentelemetry-api:1.38.0")
    implementation("io.opentelemetry:opentelemetry-sdk:1.38.0")
    implementation("io.opentelemetry:opentelemetry-exporter-otlp:1.38.0")

    // Config
    implementation("com.sksamuel.hoplite:hoplite-core:2.8.0")
    implementation("com.sksamuel.hoplite:hoplite-yaml:2.8.0")

    // Logging
    implementation("io.github.oshai:kotlin-logging-jvm:7.0.0")
    implementation("ch.qos.logback:logback-classic:1.5.6")

    // Testing
    testImplementation("org.jetbrains.kotlin:kotlin-test-junit5")
    testImplementation("io.mockk:mockk:1.13.11")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:$coroutines")
    testImplementation("io.ktor:ktor-server-test-host:$ktor")
}

application {
    mainClass.set("com.example.devagent.ApplicationKt")
}

kotlin {
    jvmToolchain(17)
}

tasks.shadowJar {
    archiveClassifier.set("")
    manifest { attributes["Main-Class"] = "com.example.devagent.ApplicationKt" }
}
```

---

## Application Entry Point

```kotlin
// src/main/kotlin/com/example/devagent/Application.kt
package com.example.devagent

import com.example.devagent.agent.*
import com.example.devagent.config.*
import com.example.devagent.events.*
import com.example.devagent.kg.*
import com.example.devagent.memory.*
import com.example.devagent.multiagent.*
import com.example.devagent.tools.*
import com.sksamuel.hoplite.ConfigLoaderBuilder
import com.sksamuel.hoplite.addEnvironmentSource
import com.sksamuel.hoplite.addResourceSource
import com.zaxxer.hikari.HikariConfig
import com.zaxxer.hikari.HikariDataSource
import io.github.oshai.kotlinlogging.KotlinLogging
import io.ktor.server.application.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import io.ktor.server.plugins.callloging.*
import io.ktor.server.plugins.statuspages.*
import io.ktor.server.response.*
import io.ktor.server.routing.*
import kotlinx.coroutines.*
import org.jetbrains.exposed.sql.Database

private val log = KotlinLogging.logger {}

fun main() {
    val config = ConfigLoaderBuilder.default()
        .addEnvironmentSource(useUnderscoresAsSeparator = true, allowUppercaseNames = true)
        .addResourceSource("/application.yaml")
        .build()
        .loadConfigOrThrow<AppConfig>()

    log.info { "Starting Dev Agent v1.0.0" }
    log.info { "Provider: ${config.provider.type}" }

    embeddedServer(Netty, port = config.server.port, host = "0.0.0.0") {
        configureApplication(config)
    }.start(wait = true)
}

fun Application.configureApplication(config: AppConfig) {
    // Database connection pool
    val dataSource = HikariDataSource(HikariConfig().apply {
        jdbcUrl = config.database.url
        username = config.database.user.ifEmpty { null }
        password = config.database.password.ifEmpty { null }
        maximumPoolSize = config.database.poolSize
        isAutoCommit = false
        transactionIsolation = "TRANSACTION_REPEATABLE_READ"
    })
    val database = Database.connect(dataSource)

    // Coroutine scope tied to application lifecycle
    val appScope = CoroutineScope(SupervisorJob() + Dispatchers.Default)

    // Observability
    val tracer = OtelSetup.initialize(config.otel.endpoint)

    // Event bus
    val eventStore = ExposedEventStore(database)
    val eventBus = EventBus(
        store = object : EventBus.EventStore {
            override suspend fun append(event: AgentEvent) = eventStore.append(event)
        },
        scope = appScope
    )

    // LLM executor
    val executor = buildExecutor(config.provider)

    // Vector store (Qdrant)
    val qdrantClient = buildQdrantClient(config.qdrant.host, config.qdrant.port, config.qdrant.apiKey)

    // Run async initialization
    appScope.launch {
        // Neo4j KG store
        val kgStore = Neo4jStore(config.neo4j.uri, config.neo4j.user, config.neo4j.password)
        kgStore.initialize()

        // Tool registry
        val fullTools = buildDevToolRegistry(database)

        // Multi-agent team
        val team = buildDevTeam(executor, config, fullTools, eventBus, kgStore)

        // A2A server for the coordinator agent
        val coordinatorCard = buildCoordinatorCard(baseUrl = "http://0.0.0.0:${config.server.port}")
        val a2aExecutor = CoordinatorA2AExecutor(team)
        val a2aServer = startA2AServer(a2aExecutor, coordinatorCard, port = config.server.port + 1)
        log.info { "A2A server started on port ${config.server.port + 1}" }

        environment.monitor.subscribe(ApplicationStopped) {
            a2aServer.stop()
            kgStore.close()
            appScope.cancel()
            dataSource.close()
        }
    }

    // Ktor plugins
    install(CallLogging)
    install(StatusPages) {
        exception<Throwable> { call, cause ->
            log.error(cause) { "Unhandled exception" }
            call.respond(io.ktor.http.HttpStatusCode.InternalServerError,
                mapOf("error" to (cause.message ?: "Internal error")))
        }
    }

    // Routes
    routing {
        get("/health") { call.respond(mapOf("status" to "ok", "version" to "1.0.0")) }
        get("/metrics") {
            call.respond(mapOf("status" to "metrics endpoint — wire SessionMetricsProjection here"))
        }
        // Main chat API
        agentRoutes(executor, config)
    }
}
```

---

## Dev Team: Multi-Agent Setup

```kotlin
// src/main/kotlin/com/example/devagent/multiagent/DevTeam.kt
package com.example.devagent.multiagent

import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.core.tools.ToolRegistry
import com.example.devagent.config.AppConfig
import com.example.devagent.events.EventBus
import com.example.devagent.kg.Neo4jStore
import ai.koog.prompt.executor.llms.PromptExecutor
import kotlinx.coroutines.async
import kotlinx.coroutines.coroutineScope

data class DevTeam(
    val coordinator: AIAgent,
    val researcher: AIAgent,
    val implementer: AIAgent,
    val reviewer: AIAgent
)

fun buildDevTeam(
    executor: PromptExecutor,
    config: AppConfig,
    tools: ToolRegistry,
    eventBus: EventBus,
    kgStore: Neo4jStore
): DevTeam {
    fun agent(systemPrompt: String) = AIAgent(
        executor = executor,
        llmModel = ai.koog.prompt.executor.clients.openai.OpenAIModels.Chat.GPT4o,
        systemPrompt = systemPrompt,
        toolRegistry = tools
    )

    val researcher = agent("""
        You are a senior software researcher. Given a task or question about a codebase:
        1. Search the code using searchCode and grep tools
        2. Read relevant files with readFile
        3. Query the knowledge graph for related symbols
        4. Produce a detailed research report with findings
    """.trimIndent())

    val implementer = agent("""
        You are a senior Kotlin engineer. Given a task and research context:
        1. Analyze the existing code structure
        2. Implement the requested changes following Kotlin best practices
        3. Write tests for your implementation
        4. Use writeFile to save all changes
        5. Run tests with runTests and fix any failures
    """.trimIndent())

    val reviewer = agent("""
        You are a meticulous code reviewer. Given changed files:
        1. Read each changed file carefully
        2. Check for: correctness, edge cases, Kotlin idioms, security, performance
        3. Run existing tests to verify nothing is broken
        4. Output a structured review: APPROVE, REQUEST_CHANGES, or REJECT
    """.trimIndent())

    val coordinator = agent(buildCoordinatorSystemPrompt())

    return DevTeam(coordinator, researcher, implementer, reviewer)
}

fun buildCoordinatorSystemPrompt() = """
    You are the coordinator of a software development team.
    Your team consists of:
    - researcher: researches the codebase and gathers context
    - implementer: writes and tests code
    - reviewer: reviews code for quality and correctness
    
    For each user request:
    1. Break down the task into subtasks
    2. Delegate to researcher first to gather context
    3. Have implementer write the code using the research
    4. Have reviewer check the implementation
    5. Iterate if the reviewer requests changes
    6. Synthesize all outputs into a final response
    
    Use the handoff tools: handoffTo_researcher, handoffTo_implementer, handoffTo_reviewer
""".trimIndent()

/** Parallel task dispatch: run all agents concurrently */
suspend fun DevTeam.runParallelResearch(queries: List<String>): List<String> =
    coroutineScope {
        queries.map { query ->
            async { researcher.run(query) ?: "No result for: $query" }
        }.map { it.await() }
    }
```

---

## Tool Registry for Dev Agent

```kotlin
// src/main/kotlin/com/example/devagent/tools/DevToolRegistry.kt
package com.example.devagent.tools

import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.agents.core.tools.annotations.Tool
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import kotlinx.serialization.Serializable
import org.jetbrains.exposed.sql.Database
import java.io.File
import java.nio.file.Path
import kotlin.io.path.*

@Serializable data class ReadFileArgs(val path: String, val maxLines: Int = 500)
@Serializable data class WriteFileArgs(val path: String, val content: String)
@Serializable data class SearchArgs(val pattern: String, val dir: String = ".", val fileGlob: String = "*.kt")
@Serializable data class BashArgs(val command: String, val cwd: String = ".", val timeoutSecs: Int = 60)
@Serializable data class GrepArgs(val pattern: String, val dir: String = ".", val contextLines: Int = 3)
@Serializable data class RunTestsArgs(val module: String = ".", val filter: String = "")

@Tool(name = "readFile", description = "Read a file, returning up to maxLines lines.")
suspend fun readFile(args: ReadFileArgs): String = withContext(Dispatchers.IO) {
    val f = File(args.path)
    require(f.exists()) { "File not found: ${args.path}" }
    f.readLines().take(args.maxLines).joinToString("\n")
}

@Tool(name = "writeFile", description = "Write content to a file, creating parent dirs as needed.")
suspend fun writeFile(args: WriteFileArgs): String = withContext(Dispatchers.IO) {
    File(args.path).also { it.parentFile?.mkdirs() }.writeText(args.content)
    "Written ${args.content.length} chars to ${args.path}"
}

@Tool(name = "searchCode", description = "Search for files matching a glob pattern in a directory.")
suspend fun searchCode(args: SearchArgs): String = withContext(Dispatchers.IO) {
    val dir = Path.of(args.dir)
    val matcher = dir.fileSystem.getPathMatcher("glob:**/${args.fileGlob}")
    dir.walk()
        .filter { it.isRegularFile() && matcher.matches(it) }
        .map { it.toString() }
        .take(50)
        .joinToString("\n")
        .ifEmpty { "No files found matching ${args.fileGlob} in ${args.dir}" }
}

@Tool(name = "grep", description = "Search file contents with a regex pattern.")
suspend fun grep(args: GrepArgs): String = withContext(Dispatchers.IO) {
    val proc = ProcessBuilder(
        "grep", "-rn",
        "--include=*.kt", "--include=*.kts", "--include=*.yaml", "--include=*.json",
        "-A", args.contextLines.toString(),
        args.pattern, args.dir
    ).redirectErrorStream(true).start()
    val out = proc.inputStream.bufferedReader().readText().take(30_000)
    proc.waitFor()
    out.ifEmpty { "No matches for: ${args.pattern}" }
}

@Tool(name = "runBash", description = "Execute a bash command. Use for build tasks, git, etc.")
suspend fun runBash(args: BashArgs): String = withContext(Dispatchers.IO) {
    val blocked = listOf("rm -rf /", "sudo rm", ":(){ :|:& };:")
    blocked.forEach { cmd -> require(args.command !in cmd) { "Blocked: $cmd" } }
    val proc = ProcessBuilder("/bin/bash", "-c", args.command)
        .directory(File(args.cwd))
        .redirectErrorStream(true)
        .start()
    val output = proc.inputStream.bufferedReader().readText()
    val exit = proc.waitFor()
    "Exit: $exit\n$output".take(20_000)
}

@Tool(name = "runTests", description = "Run Gradle tests for a module, optionally filtered.")
suspend fun runTests(args: RunTestsArgs): String = withContext(Dispatchers.IO) {
    val cmd = buildList {
        add("./gradlew")
        add("test")
        if (args.module != ".") { add("-p"); add(args.module) }
        if (args.filter.isNotEmpty()) add("--tests=${args.filter}")
        add("--no-daemon")
    }
    val proc = ProcessBuilder(cmd)
        .directory(File(args.module))
        .redirectErrorStream(true)
        .start()
    val output = proc.inputStream.bufferedReader().readText()
    val exit = proc.waitFor()
    "Exit: $exit\n${output.takeLast(10_000)}"
}

fun buildDevToolRegistry(database: Database): ToolRegistry = ToolRegistry {
    tool(::readFile)
    tool(::writeFile)
    tool(::searchCode)
    tool(::grep)
    tool(::runBash)
    tool(::runTests)
    tool(DatabaseQueryTool(database))
}

// Placeholder to avoid compiler error (defined in 03-tool-system.md):
class DatabaseQueryTool(private val db: Database) :
    ai.koog.agents.core.tools.SimpleTool<QueryArgs>() {
    @Serializable data class QueryArgs(val sql: String)
    override val descriptor = ai.koog.agents.core.tools.ToolDescriptor(
        name = "databaseQuery",
        description = "Execute a read-only SQL query",
        parameters = listOf(
            ai.koog.agents.core.tools.ToolParameterDescriptor(
                name = "sql", description = "SQL SELECT statement",
                type = ai.koog.agents.core.tools.ToolParameterType.String, required = true
            )
        )
    )
    override suspend fun execute(args: QueryArgs): String = "Query executed."
}
```

---

## REST API Routes

```kotlin
// src/main/kotlin/com/example/devagent/api/Routes.kt
package com.example.devagent.api

import ai.koog.agents.core.agent.AIAgent
import com.example.devagent.config.AppConfig
import com.example.devagent.config.buildExecutor
import io.ktor.http.*
import io.ktor.server.application.*
import io.ktor.server.request.*
import io.ktor.server.response.*
import io.ktor.server.routing.*
import io.ktor.server.sse.*
import kotlinx.coroutines.flow.collect
import kotlinx.serialization.Serializable
import ai.koog.prompt.executor.llms.PromptExecutor

@Serializable data class ChatRequest(val message: String, val sessionId: String = "")
@Serializable data class ChatResponse(val response: String, val sessionId: String)

fun Routing.agentRoutes(executor: PromptExecutor, config: AppConfig) {
    install(SSE)

    route("/api/v1") {
        // Synchronous chat
        post("/chat") {
            val req = call.receive<ChatRequest>()
            val agent = AIAgent(
                executor = executor,
                llmModel = ai.koog.prompt.executor.clients.openai.OpenAIModels.Chat.GPT4o,
                systemPrompt = "You are a helpful software development assistant."
            )
            val result = agent.run(req.message) ?: "I could not generate a response."
            call.respond(ChatResponse(result, req.sessionId.ifEmpty { java.util.UUID.randomUUID().toString() }))
        }

        // Streaming chat via SSE
        sse("/chat/stream") {
            val message = call.request.queryParameters["message"] ?: "Hello"
            val agent = AIAgent(
                executor = executor,
                llmModel = ai.koog.prompt.executor.clients.openai.OpenAIModels.Chat.GPT4o,
                systemPrompt = "You are a helpful software development assistant."
            )
            agent.runStreaming(message).collect { token ->
                send(io.ktor.server.sse.ServerSentEvent(data = token))
            }
        }

        // Health + readiness
        get("/health") { call.respond(mapOf("status" to "healthy")) }
    }
}
```

---

## OpenTelemetry + Langfuse Setup

```kotlin
// src/main/kotlin/com/example/devagent/events/OtelSetup.kt
package com.example.devagent.events

import io.opentelemetry.api.trace.Tracer
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter
import io.opentelemetry.sdk.OpenTelemetrySdk
import io.opentelemetry.sdk.resources.Resource
import io.opentelemetry.sdk.trace.SdkTracerProvider
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor

object OtelSetup {
    fun initialize(
        otlpEndpoint: String,
        langfuseSecret: String = "",
        langfusePublic: String = "",
        langfuseHost: String = "https://cloud.langfuse.com",
        serviceName: String = "dev-agent"
    ): Tracer {
        val processors = mutableListOf<io.opentelemetry.sdk.trace.export.SpanExporter>()

        // Primary OTLP endpoint (Jaeger, Tempo, etc.)
        if (otlpEndpoint.isNotEmpty()) {
            processors.add(
                OtlpGrpcSpanExporter.builder().setEndpoint(otlpEndpoint).build()
            )
        }

        // Langfuse (optional)
        if (langfuseSecret.isNotEmpty() && langfusePublic.isNotEmpty()) {
            val credentials = java.util.Base64.getEncoder()
                .encodeToString("$langfusePublic:$langfuseSecret".toByteArray())
            processors.add(
                OtlpGrpcSpanExporter.builder()
                    .setEndpoint("$langfuseHost/api/public/otel")
                    .addHeader("Authorization", "Basic $credentials")
                    .build()
            )
        }

        val resource = Resource.create(
            io.opentelemetry.api.common.Attributes.of(
                io.opentelemetry.semconv.resource.attributes.ResourceAttributes.SERVICE_NAME,
                serviceName
            )
        )

        val tracerProvider = SdkTracerProvider.builder()
            .setResource(resource)
            .also { builder ->
                processors.forEach { exporter ->
                    builder.addSpanProcessor(BatchSpanProcessor.builder(exporter).build())
                }
            }
            .build()

        return OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .buildAndRegisterGlobal()
            .getTracer(serviceName)
    }
}
```

---

## Docker Compose

```yaml
# docker-compose.yml
version: "3.9"

services:
  # Qdrant vector database
  qdrant:
    image: qdrant/qdrant:v1.9.0
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - qdrant_data:/qdrant/storage
    environment:
      QDRANT__SERVICE__API_KEY: "${QDRANT_API_KEY:-}"

  # PostgreSQL for session persistence + event store
  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: agents
      POSTGRES_USER: agent
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD:-secret}"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U agent -d agents"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Neo4j for knowledge graph
  neo4j:
    image: neo4j:5.19.0-community
    ports:
      - "7474:7474"
      - "7687:7687"
    environment:
      NEO4J_AUTH: "neo4j/${NEO4J_PASSWORD:-secret}"
      NEO4J_PLUGINS: '["apoc"]'
    volumes:
      - neo4j_data:/data

  # Jaeger for OpenTelemetry traces
  jaeger:
    image: jaegertracing/all-in-one:1.58
    ports:
      - "4317:4317"    # OTLP gRPC
      - "16686:16686"  # Jaeger UI
    environment:
      COLLECTOR_OTLP_ENABLED: "true"

  # The agent service itself
  dev-agent:
    build: .
    ports:
      - "8080:8080"
      - "8081:8081"  # A2A server
    environment:
      OPENAI_API_KEY: "${OPENAI_API_KEY}"
      ANTHROPIC_API_KEY: "${ANTHROPIC_API_KEY:-}"
      POSTGRES_URL: "jdbc:postgresql://postgres:5432/agents"
      POSTGRES_USER: "agent"
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD:-secret}"
      QDRANT_HOST: "qdrant"
      QDRANT_PORT: "6334"
      NEO4J_URI: "bolt://neo4j:7687"
      NEO4J_USER: "neo4j"
      NEO4J_PASSWORD: "${NEO4J_PASSWORD:-secret}"
      OTEL_EXPORTER_OTLP_ENDPOINT: "http://jaeger:4317"
      LANGFUSE_SECRET_KEY: "${LANGFUSE_SECRET_KEY:-}"
      LANGFUSE_PUBLIC_KEY: "${LANGFUSE_PUBLIC_KEY:-}"
      SERVER_PORT: "8080"
    depends_on:
      postgres:
        condition: service_healthy
      qdrant:
        condition: service_started
      neo4j:
        condition: service_started
    volumes:
      - ./data:/app/data    # local file workspace
      - ./src:/app/src      # mount source for code agent to inspect

volumes:
  qdrant_data:
  postgres_data:
  neo4j_data:
```

---

## Dockerfile

```dockerfile
FROM gradle:8.8-jdk17 AS builder
WORKDIR /app
COPY build.gradle.kts settings.gradle.kts ./
COPY src ./src
RUN gradle shadowJar --no-daemon

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
RUN mkdir -p /app/data
EXPOSE 8080 8081
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

---

## Build and Deployment

```bash
# --- Local Development ---

# 1. Start infrastructure
docker compose up -d qdrant postgres neo4j jaeger

# 2. Set secrets
export OPENAI_API_KEY="sk-..."
export POSTGRES_PASSWORD="secret"
export NEO4J_PASSWORD="secret"

# 3. Run locally (hot-reload via Gradle)
./gradlew run

# 4. Test the agent
curl -X POST http://localhost:8080/api/v1/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Explain the architecture of this codebase"}'

# Stream tokens via SSE
curl "http://localhost:8080/api/v1/chat/stream?message=Write+a+Kotlin+coroutine+example"

# --- Production Deployment ---

# Build fat jar
./gradlew shadowJar

# Build and run with Docker Compose
docker compose up --build -d

# View logs
docker compose logs -f dev-agent

# View traces in Jaeger
open http://localhost:16686

# View Neo4j graph
open http://localhost:7474

# --- Running Tests ---
./gradlew test

# Run with coverage
./gradlew test jacocoTestReport

# Integration tests (requires running infrastructure)
./gradlew integrationTest -PwithDocker=true
```

---

## application.yaml

```yaml
server:
  port: 8080

provider:
  type: OPENAI
  api-key: ${OPENAI_API_KEY}
  model: gpt-4o

database:
  url: ${POSTGRES_URL:jdbc:sqlite:./data/agent.db}
  user: ${POSTGRES_USER:}
  password: ${POSTGRES_PASSWORD:}
  pool-size: 10

qdrant:
  host: ${QDRANT_HOST:localhost}
  port: ${QDRANT_PORT:6334}
  api-key: ${QDRANT_API_KEY:}
  collection: dev-agent-memory

neo4j:
  uri: ${NEO4J_URI:bolt://localhost:7687}
  user: ${NEO4J_USER:neo4j}
  password: ${NEO4J_PASSWORD:secret}

otel:
  endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT:http://localhost:4317}
  langfuse-secret: ${LANGFUSE_SECRET_KEY:}
  langfuse-public: ${LANGFUSE_PUBLIC_KEY:}
  langfuse-host: ${LANGFUSE_HOST:https://cloud.langfuse.com}

agent:
  max-tokens: 128000
  compaction-threshold: 0.80
  max-tool-retries: 3
  session-timeout-minutes: 60
  enable-bash: false          # set to true only in trusted environments
  enable-web-search: true
```

---

## Integration Test Example

```kotlin
// src/test/kotlin/com/example/devagent/IntegrationTest.kt
package com.example.devagent

import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.http.*
import io.ktor.server.testing.*
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals
import kotlin.test.assertTrue

class AgentIntegrationTest {

    @Test
    fun `health endpoint returns ok`() = testApplication {
        application { configureApplication(loadTestConfig()) }
        val response = client.get("/health")
        assertEquals(HttpStatusCode.OK, response.status)
        assertTrue(response.bodyAsText().contains("ok"))
    }

    @Test
    fun `chat endpoint returns non-empty response`() = testApplication {
        application { configureApplication(loadTestConfig()) }
        val response = client.post("/api/v1/chat") {
            contentType(ContentType.Application.Json)
            setBody("""{"message": "What is 2+2?"}""")
        }
        assertEquals(HttpStatusCode.OK, response.status)
        assertTrue(response.bodyAsText().contains("4") || response.bodyAsText().contains("four"))
    }
}

fun loadTestConfig(): AppConfig {
    // Load from test resources, override with in-memory DB
    return AppConfig(
        server = ServerConfig(port = 0),
        provider = ProviderConfig(
            type = ProviderType.OPENAI,
            apiKey = System.getenv("OPENAI_API_KEY") ?: "test-key"
        ),
        database = DatabaseConfig(
            url = "jdbc:sqlite::memory:",
            user = "",
            password = ""
        ),
        qdrant = QdrantConfig(host = "localhost", port = 6334),
        neo4j = Neo4jConfig(uri = "bolt://localhost:7687", user = "neo4j", password = "secret"),
        otel = OtelConfig(endpoint = "")
    )
}
```
