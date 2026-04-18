# Event-Driven Architecture with Kotlin Coroutines

## Motivation

Agent systems benefit from event sourcing (arXiv:2602.23193):
- Full audit trail of every decision and tool call
- Replay and debugging support
- CQRS: separate write path (channel) from read path (StateFlow projections)
- Decoupled observability — consumers attach to the event stream without modifying agent logic

---

## Sealed EventType Hierarchy

```kotlin
// events/EventModels.kt
package com.example.agent.events

import kotlinx.serialization.Serializable
import kotlinx.serialization.json.JsonElement
import java.time.Instant
import java.util.UUID

sealed class EventType {
    // Agent lifecycle
    object AgentStarted        : EventType()
    object AgentFinished       : EventType()
    object AgentFailed         : EventType()
    object AgentCancelled      : EventType()

    // LLM interactions
    object LlmRequestSent      : EventType()
    object LlmResponseReceived : EventType()
    object LlmStreamStarted    : EventType()
    object LlmStreamFinished   : EventType()

    // Tool execution
    object ToolCallStarted     : EventType()
    object ToolCallSucceeded   : EventType()
    object ToolCallFailed      : EventType()
    object ToolCallCancelled   : EventType()

    // Memory operations
    object MemoryRead          : EventType()
    object MemoryWrite         : EventType()
    object MemoryCompacted     : EventType()

    // Multi-agent
    object MessageSentToAgent  : EventType()
    object MessageFromAgent    : EventType()

    // RAG
    object RagQueryStarted     : EventType()
    object RagQueryCompleted   : EventType()

    // Session management
    object SessionCreated      : EventType()
    object SessionResumed      : EventType()
    object SessionPersisted    : EventType()

    /** Custom domain events */
    data class Custom(val name: String) : EventType()
}

@Serializable
data class AgentEvent(
    val id: String = UUID.randomUUID().toString(),
    val type: String,               // EventType::class.simpleName
    val sessionId: String,
    val agentId: String,
    val timestamp: Long = Instant.now().epochSecond,
    val payload: Map<String, String> = emptyMap(),
    val traceId: String = "",
    val spanId: String = ""
)

// Helper extension
fun EventType.toEvent(
    sessionId: String,
    agentId: String,
    payload: Map<String, String> = emptyMap()
) = AgentEvent(
    type = this::class.simpleName ?: "Unknown",
    sessionId = sessionId,
    agentId = agentId,
    payload = payload
)
```

---

## Append-Only JSONL Event Store

```kotlin
// events/EventStore.kt
package com.example.agent.events

import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock
import kotlinx.coroutines.withContext
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json
import java.io.File
import java.time.LocalDate

class JsonlEventStore(
    private val logDir: String = "./data/events",
    private val rotateDaily: Boolean = true
) {
    private val json = Json { encodeDefaults = true }
    private val mutex = Mutex()

    private fun currentLogFile(): File {
        val dir = File(logDir).also { it.mkdirs() }
        val fileName = if (rotateDaily) "events-${LocalDate.now()}.jsonl"
                       else "events.jsonl"
        return File(dir, fileName)
    }

    suspend fun append(event: AgentEvent) = mutex.withLock {
        withContext(Dispatchers.IO) {
            currentLogFile().appendText(json.encodeToString(event) + "\n")
        }
    }

    suspend fun replay(
        sessionId: String? = null,
        sinceTimestamp: Long = 0L
    ): List<AgentEvent> = withContext(Dispatchers.IO) {
        File(logDir).listFiles { f -> f.name.endsWith(".jsonl") }
            ?.flatMap { file -> file.readLines() }
            ?.mapNotNull { line -> runCatching { json.decodeFromString<AgentEvent>(line) }.getOrNull() }
            ?.filter { event ->
                event.timestamp >= sinceTimestamp &&
                (sessionId == null || event.sessionId == sessionId)
            }
            ?.sortedBy { it.timestamp }
            ?: emptyList()
    }

    suspend fun replaySession(sessionId: String): List<AgentEvent> =
        replay(sessionId = sessionId)
}
```

---

## EventStore with Kotlin Exposed (SQLite / Postgres)

```kotlin
// events/ExposedEventStore.kt
package com.example.agent.events

import kotlinx.coroutines.Dispatchers
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json
import org.jetbrains.exposed.sql.*
import org.jetbrains.exposed.sql.transactions.experimental.newSuspendedTransaction
import org.jetbrains.exposed.sql.transactions.transaction

object AgentEvents : Table("agent_events") {
    val id          = varchar("id", 36)
    val type        = varchar("type", 100)
    val sessionId   = varchar("session_id", 36).index()
    val agentId     = varchar("agent_id", 36).index()
    val timestamp   = long("timestamp").index()
    val payload     = text("payload")
    val traceId     = varchar("trace_id", 64).default("")

    override val primaryKey = PrimaryKey(id)
}

class ExposedEventStore(database: Database) {
    private val json = Json { encodeDefaults = true }

    init {
        transaction(database) {
            SchemaUtils.createMissingTablesAndColumns(AgentEvents)
        }
    }

    suspend fun append(event: AgentEvent) = newSuspendedTransaction(Dispatchers.IO) {
        AgentEvents.insertIgnore {
            it[id]        = event.id
            it[type]      = event.type
            it[sessionId] = event.sessionId
            it[agentId]   = event.agentId
            it[timestamp] = event.timestamp
            it[payload]   = json.encodeToString(event.payload)
            it[traceId]   = event.traceId
        }
    }

    suspend fun queryBySession(sessionId: String): List<AgentEvent> =
        newSuspendedTransaction(Dispatchers.IO) {
            AgentEvents
                .select { AgentEvents.sessionId eq sessionId }
                .orderBy(AgentEvents.timestamp)
                .map { row -> rowToEvent(row) }
        }

    suspend fun queryByType(type: String, limit: Int = 1000): List<AgentEvent> =
        newSuspendedTransaction(Dispatchers.IO) {
            AgentEvents
                .select { AgentEvents.type eq type }
                .orderBy(AgentEvents.timestamp, SortOrder.DESC)
                .limit(limit)
                .map { rowToEvent(it) }
        }

    private fun rowToEvent(row: ResultRow) = AgentEvent(
        id        = row[AgentEvents.id],
        type      = row[AgentEvents.type],
        sessionId = row[AgentEvents.sessionId],
        agentId   = row[AgentEvents.agentId],
        timestamp = row[AgentEvents.timestamp],
        payload   = runCatching {
            json.decodeFromString<Map<String, String>>(row[AgentEvents.payload])
        }.getOrDefault(emptyMap()),
        traceId   = row[AgentEvents.traceId]
    )
}
```

---

## CQRS: Write Path + Read Path (StateFlow Projections)

```kotlin
// events/EventBus.kt
package com.example.agent.events

import kotlinx.coroutines.*
import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.flow.*

/** Write path: coroutine channel for event ingestion */
class EventBus(
    private val store: EventStore,
    private val scope: CoroutineScope,
    private val bufferSize: Int = 1000
) {
    interface EventStore {
        suspend fun append(event: AgentEvent)
    }

    // Internal buffered channel — non-blocking producers
    private val channel = Channel<AgentEvent>(bufferSize)

    // Broadcast via SharedFlow — multiple consumers can subscribe
    private val _events = MutableSharedFlow<AgentEvent>(
        replay = 100,
        extraBufferCapacity = 1000
    )
    val events: SharedFlow<AgentEvent> = _events.asSharedFlow()

    init {
        // Write path: consume channel → persist → broadcast
        scope.launch(Dispatchers.IO) {
            for (event in channel) {
                runCatching {
                    store.append(event)
                    _events.emit(event)
                }.onFailure { e ->
                    println("[EventBus] Failed to process event: ${e.message}")
                }
            }
        }
    }

    suspend fun publish(event: AgentEvent) {
        channel.send(event)
    }

    fun publishAsync(event: AgentEvent) {
        scope.launch { publish(event) }
    }

    fun close() { channel.close() }
}

/** Read path: StateFlow projections over the event stream */
class SessionMetricsProjection(
    private val bus: EventBus,
    private val scope: CoroutineScope
) {
    data class SessionMetrics(
        val sessionId: String,
        val toolCallCount: Int = 0,
        val llmRequestCount: Int = 0,
        val failureCount: Int = 0,
        val lastEventAt: Long = 0L
    )

    private val _metrics = MutableStateFlow<Map<String, SessionMetrics>>(emptyMap())
    val metrics: StateFlow<Map<String, SessionMetrics>> = _metrics.asStateFlow()

    init {
        scope.launch {
            bus.events.collect { event ->
                _metrics.update { current ->
                    val prev = current[event.sessionId] ?: SessionMetrics(event.sessionId)
                    val updated = when (event.type) {
                        "ToolCallStarted"    -> prev.copy(toolCallCount    = prev.toolCallCount    + 1)
                        "LlmRequestSent"     -> prev.copy(llmRequestCount  = prev.llmRequestCount  + 1)
                        "ToolCallFailed",
                        "AgentFailed"        -> prev.copy(failureCount     = prev.failureCount     + 1)
                        else                 -> prev
                    }.copy(lastEventAt = event.timestamp)
                    current + (event.sessionId to updated)
                }
            }
        }
    }
}
```

---

## Koog EventHandler for Agent Lifecycle

```kotlin
// events/KoogEventHandlerSetup.kt
package com.example.agent.events

import ai.koog.agents.core.feature.handler.EventHandler
import ai.koog.agents.core.feature.model.ToolCall
import io.github.oshai.kotlinlogging.KotlinLogging

private val log = KotlinLogging.logger {}

fun buildObservabilityHandler(
    bus: EventBus,
    sessionId: String,
    agentId: String
): EventHandler {
    return EventHandler {

        onAgentStarted { input ->
            log.info { "[Agent $agentId] Started. Input: ${input.take(100)}" }
            bus.publishAsync(
                EventType.AgentStarted.toEvent(sessionId, agentId,
                    mapOf("inputPreview" to input.take(200)))
            )
        }

        onAgentFinished { output ->
            log.info { "[Agent $agentId] Finished. Output: ${output?.take(100)}" }
            bus.publishAsync(
                EventType.AgentFinished.toEvent(sessionId, agentId,
                    mapOf("outputPreview" to (output?.take(200) ?: "")))
            )
        }

        onPreToolUse { toolCall: ToolCall ->
            log.debug { "[Agent $agentId] Tool → ${toolCall.toolName}" }
            bus.publishAsync(
                EventType.ToolCallStarted.toEvent(sessionId, agentId,
                    mapOf("tool" to toolCall.toolName, "args" to toolCall.args.take(500)))
            )
        }

        onPostToolUse { toolCall, result ->
            log.debug { "[Agent $agentId] Tool ✓ ${toolCall.toolName}" }
            bus.publishAsync(
                EventType.ToolCallSucceeded.toEvent(sessionId, agentId,
                    mapOf("tool" to toolCall.toolName, "resultPreview" to result.take(200)))
            )
        }

        onPostToolUseFailure { toolCall, error ->
            log.error { "[Agent $agentId] Tool ✗ ${toolCall.toolName}: ${error.message}" }
            bus.publishAsync(
                EventType.ToolCallFailed.toEvent(sessionId, agentId,
                    mapOf("tool" to toolCall.toolName, "error" to (error.message ?: "")))
            )
        }

        onLlmRequest { prompt ->
            bus.publishAsync(
                EventType.LlmRequestSent.toEvent(sessionId, agentId,
                    mapOf("promptLength" to prompt.length.toString()))
            )
        }

        onLlmResponse { response ->
            bus.publishAsync(
                EventType.LlmResponseReceived.toEvent(sessionId, agentId,
                    mapOf("responseLength" to response.length.toString()))
            )
        }
    }
}
```

---

## OpenTelemetry + Langfuse Integration

```kotlin
// events/OtelObservability.kt
package com.example.agent.events

import io.opentelemetry.api.GlobalOpenTelemetry
import io.opentelemetry.api.trace.Span
import io.opentelemetry.api.trace.SpanKind
import io.opentelemetry.api.trace.Tracer
import io.opentelemetry.context.Context
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter
import io.opentelemetry.sdk.OpenTelemetrySdk
import io.opentelemetry.sdk.trace.SdkTracerProvider
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor

object OtelSetup {
    fun initialize(
        otlpEndpoint: String = "http://localhost:4317",
        serviceName: String = "koog-agent"
    ): Tracer {
        val exporter = OtlpGrpcSpanExporter.builder()
            .setEndpoint(otlpEndpoint)
            .build()

        val tracerProvider = SdkTracerProvider.builder()
            .addSpanProcessor(BatchSpanProcessor.builder(exporter).build())
            .setResource(
                io.opentelemetry.sdk.resources.Resource.create(
                    io.opentelemetry.api.common.Attributes.of(
                        io.opentelemetry.semconv.resource.attributes.ResourceAttributes.SERVICE_NAME,
                        serviceName
                    )
                )
            )
            .build()

        val otel = OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .buildAndRegisterGlobal()

        return otel.getTracer(serviceName)
    }
}

class OtelAgentTracer(private val tracer: Tracer) {
    private val activeSpans = java.util.concurrent.ConcurrentHashMap<String, Span>()

    fun startAgentSpan(sessionId: String, input: String): Span {
        val span = tracer.spanBuilder("agent.run")
            .setSpanKind(SpanKind.SERVER)
            .startSpan()
        span.setAttribute("session.id", sessionId)
        span.setAttribute("input.length", input.length.toLong())
        activeSpans[sessionId] = span
        return span
    }

    fun startToolSpan(sessionId: String, toolName: String, args: String): Span {
        val parentSpan = activeSpans[sessionId]
        val spanBuilder = tracer.spanBuilder("tool.$toolName")
            .setSpanKind(SpanKind.INTERNAL)
        if (parentSpan != null) {
            spanBuilder.setParent(Context.current().with(parentSpan))
        }
        val span = spanBuilder.startSpan()
        span.setAttribute("tool.name", toolName)
        span.setAttribute("tool.args", args.take(1000))
        return span
    }

    fun endAgentSpan(sessionId: String, output: String) {
        activeSpans.remove(sessionId)?.also { span ->
            span.setAttribute("output.length", output.length.toLong())
            span.end()
        }
    }
}

/** Langfuse trace export via OTLP-compatible endpoint */
fun buildLangfuseExporter(
    secretKey: String,
    publicKey: String,
    host: String = "https://cloud.langfuse.com"
): OtlpGrpcSpanExporter {
    val credentials = java.util.Base64.getEncoder()
        .encodeToString("$publicKey:$secretKey".toByteArray())
    return OtlpGrpcSpanExporter.builder()
        .setEndpoint("$host/api/public/otel")
        .addHeader("Authorization", "Basic $credentials")
        .build()
}
```

---

## Replay from Event Log

```kotlin
// events/EventReplay.kt
package com.example.agent.events

import kotlinx.coroutines.flow.*

/** Replay events from JSONL and reconstruct session state */
class EventReplayer(private val store: JsonlEventStore) {

    /** Reconstruct conversation messages from events for a session */
    suspend fun reconstructSession(sessionId: String): List<Map<String, String>> {
        val events = store.replaySession(sessionId)
        val messages = mutableListOf<Map<String, String>>()

        events.forEach { event ->
            when (event.type) {
                "LlmRequestSent" -> messages.add(
                    mapOf("role" to "user", "content" to (event.payload["prompt"] ?: ""))
                )
                "LlmResponseReceived" -> messages.add(
                    mapOf("role" to "assistant", "content" to (event.payload["response"] ?: ""))
                )
                "ToolCallSucceeded" -> messages.add(
                    mapOf(
                        "role" to "tool",
                        "tool" to (event.payload["tool"] ?: ""),
                        "result" to (event.payload["resultPreview"] ?: "")
                    )
                )
            }
        }
        return messages
    }

    /** Stream events as a cold Flow for debugging */
    fun replayAsFlow(sessionId: String): Flow<AgentEvent> = flow {
        store.replaySession(sessionId).forEach { emit(it) }
    }

    /** Compute session statistics from replay */
    suspend fun sessionStats(sessionId: String): Map<String, Any> {
        val events = store.replaySession(sessionId)
        return mapOf(
            "totalEvents"    to events.size,
            "toolCalls"      to events.count { it.type == "ToolCallStarted" },
            "toolFailures"   to events.count { it.type == "ToolCallFailed" },
            "llmRequests"    to events.count { it.type == "LlmRequestSent" },
            "durationSecs"   to if (events.size >= 2)
                events.last().timestamp - events.first().timestamp else 0L
        )
    }
}
```

---

## SharedFlow Broadcast to Multiple Consumers

```kotlin
// Attach multiple independent consumers to the same event stream
val bus = EventBus(store, scope)

// Consumer 1: metrics projection
val metrics = SessionMetricsProjection(bus, scope)

// Consumer 2: alert on failures
scope.launch {
    bus.events
        .filter { it.type == "ToolCallFailed" || it.type == "AgentFailed" }
        .collect { event ->
            sendAlert("Agent failure in session ${event.sessionId}: ${event.payload}")
        }
}

// Consumer 3: cost tracking
scope.launch {
    bus.events
        .filter { it.type == "LlmResponseReceived" }
        .collect { event ->
            val tokens = event.payload["tokens"]?.toIntOrNull() ?: 0
            costTracker.record(event.sessionId, tokens)
        }
}

// Consumer 4: live dashboard via SSE
scope.launch {
    bus.events
        .filter { it.sessionId == "active-session-id" }
        .collect { event ->
            dashboardSseEmitter.emit(event)
        }
}
```
