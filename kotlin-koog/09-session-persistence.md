# Session and Persistence

## Session Data Model

```kotlin
// persistence/SessionManager.kt
package com.example.agent.persistence

import kotlinx.serialization.Serializable
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json
import java.time.Instant
import java.util.UUID

@Serializable
data class SessionState(
    val sessionId: String = UUID.randomUUID().toString(),
    val agentId: String,
    val userId: String = "",
    val createdAt: Long = Instant.now().epochSecond,
    val lastActivityAt: Long = Instant.now().epochSecond,
    val status: SessionStatus = SessionStatus.Active,
    val metadata: Map<String, String> = emptyMap(),
    val messageCount: Int = 0,
    val tokenCount: Int = 0
)

enum class SessionStatus { Active, Idle, Compacted, Archived, Terminated }

@Serializable
data class SessionSnapshot(
    val sessionId: String,
    val state: SessionState,
    val compressedHistory: String = "",   // summarized conversation
    val checkpointAt: Long = Instant.now().epochSecond,
    val version: Int = 1
)
```

---

## Koog Persistence.Feature

Koog ships a `Persistence.Feature` that hooks into the agent lifecycle to auto-save state.

```kotlin
// persistence/KoogPersistence.kt
package com.example.agent.persistence

import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.core.agent.config.AIAgentConfig
import ai.koog.agents.core.agent.config.PromptConfig
import ai.koog.agents.core.features.Persistence
import ai.koog.agents.core.features.persistence.PostgresJdbcPersistenceStorageProvider
import ai.koog.agents.core.features.persistence.SqlitePersistenceStorageProvider
import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel

/** Build a Postgres-backed persistent agent */
fun buildPersistentAgent(
    executor: PromptExecutor,
    model: LLMModel,
    systemPrompt: String,
    toolRegistry: ToolRegistry,
    sessionId: String,
    jdbcUrl: String,
    dbUser: String,
    dbPassword: String
): AIAgent {
    val storageProvider = PostgresJdbcPersistenceStorageProvider(
        jdbcUrl = jdbcUrl,
        username = dbUser,
        password = dbPassword
    )

    return AIAgent(
        executor = executor,
        agentConfig = AIAgentConfig(
            prompt = PromptConfig(systemPrompt = systemPrompt),
            tools = toolRegistry
        ),
        llmModel = model
    ) {
        // Install persistence feature
        install(Persistence) {
            storage = storageProvider
            this.sessionId = sessionId
            enableAutomaticPersistence = true
            persistEveryNMessages = 5    // persist every 5 messages
        }
    }
}

/** SQLite fallback for dev/embedded use */
fun buildSQLitePersistentAgent(
    executor: PromptExecutor,
    model: LLMModel,
    systemPrompt: String,
    toolRegistry: ToolRegistry,
    sessionId: String,
    dbPath: String = "./data/sessions.db"
): AIAgent {
    val storageProvider = SqlitePersistenceStorageProvider(dbPath)

    return AIAgent(
        executor = executor,
        agentConfig = AIAgentConfig(
            prompt = PromptConfig(systemPrompt = systemPrompt),
            tools = toolRegistry
        ),
        llmModel = model
    ) {
        install(Persistence) {
            storage = storageProvider
            this.sessionId = sessionId
            enableAutomaticPersistence = true
        }
    }
}

/** Resume a prior session by sessionId */
suspend fun resumeSession(
    executor: PromptExecutor,
    model: LLMModel,
    systemPrompt: String,
    toolRegistry: ToolRegistry,
    sessionId: String,
    storageProvider: PostgresJdbcPersistenceStorageProvider
): AIAgent {
    val agent = AIAgent(
        executor = executor,
        agentConfig = AIAgentConfig(
            prompt = PromptConfig(systemPrompt = systemPrompt),
            tools = toolRegistry
        ),
        llmModel = model
    ) {
        install(Persistence) {
            storage = storageProvider
            this.sessionId = sessionId
            enableAutomaticPersistence = true
            resumeSession = true      // load prior state on startup
        }
    }
    println("[Session] Resumed session $sessionId")
    return agent
}
```

---

## RollbackToolRegistry + RollbackStrategy

The `RollbackToolRegistry` tracks tool call side effects and can undo them with compensating actions.

```kotlin
// persistence/RollbackRegistry.kt
package com.example.agent.persistence

import ai.koog.agents.core.tools.RollbackToolRegistry
import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.agents.core.tools.annotations.Tool
import kotlinx.serialization.Serializable

@Serializable
data class WriteFileArgs(val path: String, val content: String)

@Serializable
data class DeleteFileArgs(val path: String)

// Compensating action: if writeFile fails or is rolled back, delete the file
fun buildRollbackRegistry(): RollbackToolRegistry {
    return RollbackToolRegistry(
        ToolRegistry {
            tool(::writeFile)
            tool(::deleteFile)
        }
    ) {
        // Define compensating actions
        onRollback("writeFile") { args: WriteFileArgs ->
            // Undo: delete the file that was written
            java.io.File(args.path).delete()
            "Rolled back: deleted ${args.path}"
        }

        onRollback("deleteFile") { args: DeleteFileArgs ->
            // Cannot restore a deleted file without a backup — log and alert
            println("[ROLLBACK] Cannot restore deleted file: ${args.path}")
            "Warning: could not restore ${args.path}"
        }
    }
}

@Tool(name = "writeFile", description = "Write content to a file")
suspend fun writeFile(args: WriteFileArgs): String {
    java.io.File(args.path).also { it.parentFile?.mkdirs() }.writeText(args.content)
    return "Written to ${args.path}"
}

@Tool(name = "deleteFile", description = "Delete a file")
suspend fun deleteFile(args: DeleteFileArgs): String {
    java.io.File(args.path).delete()
    return "Deleted ${args.path}"
}
```

---

## Auto-Compaction Implementation

```kotlin
// persistence/CompactionStrategy.kt
package com.example.agent.persistence

import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel

interface CompactionStrategy {
    suspend fun compact(messages: List<String>, keepLast: Int): List<String>
}

/** Sliding window: keep the last N messages unchanged */
class SlidingWindowCompaction(private val windowSize: Int = 20) : CompactionStrategy {
    override suspend fun compact(messages: List<String>, keepLast: Int): List<String> =
        messages.takeLast(keepLast.coerceAtLeast(windowSize))
}

/** Summarization: LLM summarizes old messages, keeps recent ones verbatim */
class SummarizationCompaction(
    private val executor: PromptExecutor,
    private val model: LLMModel,
    private val keepRecentCount: Int = 10,
    private val summarizeCount: Int = 30
) : CompactionStrategy {

    override suspend fun compact(messages: List<String>, keepLast: Int): List<String> {
        if (messages.size <= keepRecentCount + summarizeCount) return messages

        val toSummarize = messages.dropLast(keepRecentCount)
        val toKeep = messages.takeLast(keepRecentCount)

        val summary = summarize(toSummarize)
        return listOf("[COMPACTED HISTORY]\n$summary\n[END COMPACTED HISTORY]") + toKeep
    }

    private suspend fun summarize(messages: List<String>): String {
        val agent = AIAgent(
            executor = executor,
            llmModel = model,
            systemPrompt = """
                Compress the following conversation into a concise summary.
                Preserve: key decisions, important facts, open questions, tool results.
                Output as a structured summary with sections: Facts, Decisions, Context.
            """.trimIndent()
        )
        return agent.run(messages.joinToString("\n")) ?: messages.last()
    }
}

/** Hybrid: sliding window + periodic LLM summarization */
class HybridCompaction(
    private val executor: PromptExecutor,
    private val model: LLMModel,
    private val windowSize: Int = 15,
    private val summaryTrigger: Int = 50    // summarize when history > 50 messages
) : CompactionStrategy {
    private val summarizer = SummarizationCompaction(executor, model, windowSize)
    private val window = SlidingWindowCompaction(windowSize)

    override suspend fun compact(messages: List<String>, keepLast: Int): List<String> {
        return if (messages.size > summaryTrigger)
            summarizer.compact(messages, keepLast)
        else
            window.compact(messages, keepLast)
    }
}
```

---

## Session Store with Kotlin Exposed

```kotlin
// persistence/SessionStore.kt
package com.example.agent.persistence

import kotlinx.coroutines.Dispatchers
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json
import org.jetbrains.exposed.sql.*
import org.jetbrains.exposed.sql.transactions.experimental.newSuspendedTransaction
import org.jetbrains.exposed.sql.transactions.transaction
import java.time.Instant

object Sessions : Table("sessions") {
    val id               = varchar("id", 36)
    val agentId          = varchar("agent_id", 64).index()
    val userId           = varchar("user_id", 64).default("").index()
    val status           = varchar("status", 20).default("Active")
    val createdAt        = long("created_at")
    val lastActivityAt   = long("last_activity_at").index()
    val messageCount     = integer("message_count").default(0)
    val tokenCount       = integer("token_count").default(0)
    val compressedHistory = text("compressed_history").default("")
    val metadata         = text("metadata").default("{}")

    override val primaryKey = PrimaryKey(id)
}

class SessionStore(private val database: Database) {
    private val json = Json { encodeDefaults = true }

    init {
        transaction(database) {
            SchemaUtils.createMissingTablesAndColumns(Sessions)
        }
    }

    suspend fun create(state: SessionState): SessionState =
        newSuspendedTransaction(Dispatchers.IO) {
            Sessions.insertIgnore {
                it[id]             = state.sessionId
                it[agentId]        = state.agentId
                it[userId]         = state.userId
                it[status]         = state.status.name
                it[createdAt]      = state.createdAt
                it[lastActivityAt] = state.lastActivityAt
                it[metadata]       = json.encodeToString(state.metadata)
            }
            state
        }

    suspend fun update(sessionId: String, block: SessionState.() -> SessionState) =
        newSuspendedTransaction(Dispatchers.IO) {
            val current = findById(sessionId) ?: return@newSuspendedTransaction
            val updated = current.block()
            Sessions.update({ Sessions.id eq sessionId }) {
                it[status]         = updated.status.name
                it[lastActivityAt] = Instant.now().epochSecond
                it[messageCount]   = updated.messageCount
                it[tokenCount]     = updated.tokenCount
                it[metadata]       = json.encodeToString(updated.metadata)
            }
        }

    suspend fun findById(sessionId: String): SessionState? =
        newSuspendedTransaction(Dispatchers.IO) {
            Sessions.select { Sessions.id eq sessionId }
                .firstOrNull()
                ?.let { rowToState(it) }
        }

    suspend fun findByUser(userId: String, limit: Int = 20): List<SessionState> =
        newSuspendedTransaction(Dispatchers.IO) {
            Sessions.select { Sessions.userId eq userId }
                .orderBy(Sessions.lastActivityAt, SortOrder.DESC)
                .limit(limit)
                .map { rowToState(it) }
        }

    suspend fun markCompacted(sessionId: String, compressedHistory: String) =
        newSuspendedTransaction(Dispatchers.IO) {
            Sessions.update({ Sessions.id eq sessionId }) {
                it[status]                  = SessionStatus.Compacted.name
                it[Sessions.compressedHistory] = compressedHistory
                it[lastActivityAt]          = Instant.now().epochSecond
            }
        }

    private fun rowToState(row: ResultRow): SessionState = SessionState(
        sessionId = row[Sessions.id],
        agentId = row[Sessions.agentId],
        userId = row[Sessions.userId],
        status = SessionStatus.valueOf(row[Sessions.status]),
        createdAt = row[Sessions.createdAt],
        lastActivityAt = row[Sessions.lastActivityAt],
        messageCount = row[Sessions.messageCount],
        tokenCount = row[Sessions.tokenCount],
        metadata = runCatching {
            json.decodeFromString<Map<String, String>>(row[Sessions.metadata])
        }.getOrDefault(emptyMap())
    )
}
```

---

## Configuration Loading

```kotlin
// config/AppConfig.kt (full Hoplite config with env var fallback)
package com.example.agent.config

import com.sksamuel.hoplite.ConfigLoaderBuilder
import com.sksamuel.hoplite.addEnvironmentSource
import com.sksamuel.hoplite.addResourceSource

fun loadConfig(): AppConfig {
    return ConfigLoaderBuilder.default()
        .addEnvironmentSource(useUnderscoresAsSeparator = true, allowUppercaseNames = true)
        .addResourceSource("/application.yaml")
        .build()
        .loadConfigOrThrow()
}
```

---

## LangSmith-Compatible Trace Export

```kotlin
// persistence/TraceExporter.kt
package com.example.agent.persistence

import com.example.agent.events.AgentEvent
import io.ktor.client.*
import io.ktor.client.engine.cio.*
import io.ktor.client.plugins.auth.*
import io.ktor.client.plugins.auth.providers.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.client.request.*
import io.ktor.http.*
import io.ktor.serialization.kotlinx.json.*
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json

@Serializable
data class LangSmithRun(
    val id: String,
    val name: String,
    val run_type: String,           // "chain", "llm", "tool"
    val inputs: Map<String, String>,
    val outputs: Map<String, String>,
    val start_time: Long,
    val end_time: Long,
    val session_name: String,
    val tags: List<String> = emptyList()
)

class LangSmithExporter(
    private val apiKey: String,
    private val projectName: String,
    private val baseUrl: String = "https://api.smith.langchain.com"
) {
    private val client = HttpClient(CIO) {
        install(ContentNegotiation) { json(Json { ignoreUnknownKeys = true }) }
        install(Auth) {
            bearer { loadTokens { BearerTokens(apiKey, "") } }
        }
    }

    suspend fun exportEvents(events: List<AgentEvent>) {
        val runs = events.mapIndexedNotNull { i, event ->
            when (event.type) {
                "LlmRequestSent" -> {
                    val responseEvent = events.getOrNull(i + 1)?.takeIf {
                        it.type == "LlmResponseReceived" && it.sessionId == event.sessionId
                    }
                    LangSmithRun(
                        id = event.id,
                        name = "LLM Request",
                        run_type = "llm",
                        inputs = mapOf("prompt" to (event.payload["promptLength"] ?: "")),
                        outputs = mapOf("response" to (responseEvent?.payload?.get("responseLength") ?: "")),
                        start_time = event.timestamp * 1000,
                        end_time = (responseEvent?.timestamp ?: event.timestamp) * 1000,
                        session_name = projectName
                    )
                }
                "ToolCallStarted" -> {
                    val resultEvent = events.drop(i + 1).firstOrNull {
                        (it.type == "ToolCallSucceeded" || it.type == "ToolCallFailed") &&
                        it.sessionId == event.sessionId &&
                        it.payload["tool"] == event.payload["tool"]
                    }
                    LangSmithRun(
                        id = event.id,
                        name = event.payload["tool"] ?: "tool",
                        run_type = "tool",
                        inputs = mapOf("args" to (event.payload["args"] ?: "")),
                        outputs = mapOf(
                            "result" to (resultEvent?.payload?.get("resultPreview") ?: ""),
                            "status" to (resultEvent?.type ?: "unknown")
                        ),
                        start_time = event.timestamp * 1000,
                        end_time = (resultEvent?.timestamp ?: event.timestamp) * 1000,
                        session_name = projectName
                    )
                }
                else -> null
            }
        }

        if (runs.isEmpty()) return
        client.post("$baseUrl/runs/batch") {
            contentType(ContentType.Application.Json)
            setBody(mapOf("post" to runs))
        }
    }
}
```
