# Memory and RAG Pipeline

## Memory Architecture Overview

Following CoALA (arXiv:2309.02427), agents benefit from four memory tiers:

| Tier | Implementation | Scope |
|------|---------------|-------|
| Working (in-context) | `ChatMemory` messages list | Current session |
| Episodic (recall) | SQLite `KotlinxSQLiteMemoryStore` | Past conversations |
| Semantic (long-term facts) | Qdrant vector store | Persistent knowledge |
| Procedural (skills) | Tool definitions + prompt templates | Immutable behavior |

---

## ChatMemory with SQLite

```kotlin
// memory/ChatMemoryFactory.kt
package com.example.agent.memory

import ai.koog.agents.core.memory.ChatMemory
import ai.koog.agents.core.memory.KotlinxSQLiteMemoryStore
import ai.koog.agents.core.memory.PostgresChatMemoryStore
import java.util.UUID

object ChatMemoryFactory {

    /** SQLite: lightweight, zero-config, good for dev and single-process */
    fun sqlite(dbPath: String = "file:agent.db"): ChatMemory {
        val store = KotlinxSQLiteMemoryStore(dbPath)
        return ChatMemory(store)
    }

    /** Postgres: production multi-process, persistent across restarts */
    fun postgres(jdbcUrl: String, user: String, password: String): ChatMemory {
        val store = PostgresChatMemoryStore(
            jdbcUrl = jdbcUrl,
            username = user,
            password = password
        )
        return ChatMemory(store)
    }

    /** Per-session memory keyed by session ID */
    fun sessionMemory(baseStore: ChatMemory, sessionId: String = UUID.randomUUID().toString()): ChatMemory {
        return baseStore.forSession(sessionId)
    }
}
```

---

## LongTermMemory with Qdrant

```kotlin
// memory/LongTermMemory.kt
package com.example.agent.memory

import io.qdrant.client.QdrantClient
import io.qdrant.client.QdrantGrpcClient
import io.qdrant.client.grpc.Collections
import io.qdrant.client.grpc.Points
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import kotlinx.serialization.Serializable
import java.util.UUID

@Serializable
data class MemoryEntry(
    val id: String = UUID.randomUUID().toString(),
    val content: String,
    val metadata: Map<String, String> = emptyMap(),
    val timestamp: Long = System.currentTimeMillis()
)

interface EmbeddingProvider {
    suspend fun embed(text: String): FloatArray
    suspend fun embedBatch(texts: List<String>): List<FloatArray>
}

class QdrantLongTermMemory(
    private val client: QdrantClient,
    private val embedder: EmbeddingProvider,
    private val collectionName: String = "agent-memory",
    private val vectorSize: Int = 1536   // OpenAI text-embedding-3-small
) {
    suspend fun initialize() = withContext(Dispatchers.IO) {
        val collections = client.listCollectionsAsync().await().collectionsNamesList
        if (collectionName !in collections) {
            client.createCollectionAsync(
                collectionName,
                Collections.VectorsConfig.newBuilder()
                    .setParams(
                        Collections.VectorParams.newBuilder()
                            .setSize(vectorSize.toLong())
                            .setDistance(Collections.Distance.Cosine)
                            .build()
                    )
                    .build()
            ).await()
        }
    }

    suspend fun store(entry: MemoryEntry) = withContext(Dispatchers.IO) {
        val vector = embedder.embed(entry.content)
        val point = Points.PointStruct.newBuilder()
            .setId(Points.PointId.newBuilder().setUuid(entry.id))
            .setVectors(
                Points.Vectors.newBuilder().setVector(
                    Points.Vector.newBuilder().addAllData(vector.map { it.toDouble().toFloat() })
                )
            )
            .putAllPayload(
                mapOf(
                    "content"   to Points.Value.newBuilder().setStringValue(entry.content).build(),
                    "timestamp" to Points.Value.newBuilder().setIntegerValue(entry.timestamp).build()
                ) + entry.metadata.mapValues {
                    Points.Value.newBuilder().setStringValue(it.value).build()
                }
            )
            .build()

        client.upsertAsync(collectionName, listOf(point)).await()
    }

    suspend fun search(query: String, topK: Int = 5): List<MemoryEntry> = withContext(Dispatchers.IO) {
        val queryVector = embedder.embed(query)
        val results = client.searchAsync(
            Points.SearchPoints.newBuilder()
                .setCollectionName(collectionName)
                .addAllVector(queryVector.map { it.toDouble().toFloat() })
                .setLimit(topK.toLong())
                .setWithPayload(Points.WithPayloadSelector.newBuilder().setEnable(true))
                .build()
        ).await()

        results.map { hit ->
            MemoryEntry(
                id = hit.id.uuid,
                content = hit.payloadMap["content"]?.stringValue ?: "",
                timestamp = hit.payloadMap["timestamp"]?.integerValue ?: 0L,
                metadata = hit.payloadMap
                    .filterKeys { it !in setOf("content", "timestamp") }
                    .mapValues { it.value.stringValue }
            )
        }
    }
}

fun buildQdrantClient(host: String, port: Int, apiKey: String): QdrantClient {
    val builder = QdrantGrpcClient.newBuilder(host, port, false)
    if (apiKey.isNotEmpty()) builder.withApiKey(apiKey)
    return QdrantClient(builder.build())
}
```

---

## MemGPT 3-Tier Memory (arXiv:2310.08560)

MemGPT manages memory explicitly across three tiers: in-context working memory, external recall (vector search), and archival (full document store).

```kotlin
// memory/MemGPTTiers.kt
package com.example.agent.memory

import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.update

/** In-context window (tier 1): fixed-size sliding window */
class WorkingMemory(private val maxMessages: Int = 20) {
    private val _messages = MutableStateFlow<List<String>>(emptyList())
    val messages: StateFlow<List<String>> = _messages

    fun add(message: String) {
        _messages.update { current ->
            val updated = current + message
            if (updated.size > maxMessages) updated.takeLast(maxMessages)
            else updated
        }
    }

    fun getContext(): String = _messages.value.joinToString("\n")
    fun clear() { _messages.value = emptyList() }
}

/** Recall memory (tier 2): vector search over past interactions */
class RecallMemory(private val vectorStore: QdrantLongTermMemory) {
    suspend fun store(content: String, metadata: Map<String, String> = emptyMap()) {
        vectorStore.store(MemoryEntry(content = content, metadata = metadata))
    }

    suspend fun recall(query: String, topK: Int = 5): List<String> {
        return vectorStore.search(query, topK).map { it.content }
    }
}

/** Archival memory (tier 3): full document storage with pagination */
class ArchivalMemory(private val store: QdrantLongTermMemory) {
    suspend fun archive(content: String, tags: List<String> = emptyList()) {
        store.store(
            MemoryEntry(
                content = content,
                metadata = mapOf("archived" to "true", "tags" to tags.joinToString(","))
            )
        )
    }

    suspend fun search(query: String, topK: Int = 10): List<String> {
        return store.search(query, topK)
            .filter { it.metadata["archived"] == "true" }
            .map { it.content }
    }
}

/** MemGPT agent: wraps the 3-tier memory system */
class MemGPTAgent(
    private val working: WorkingMemory,
    private val recall: RecallMemory,
    private val archival: ArchivalMemory,
    private val executor: ai.koog.prompt.executor.llms.PromptExecutor,
    private val model: ai.koog.prompt.llm.LLMModel
) {
    suspend fun chat(userMessage: String): String {
        // 1. Retrieve relevant context from recall and archival
        val recallContext = recall.recall(userMessage).joinToString("\n")
        val archiveContext = archival.search(userMessage, topK = 3).joinToString("\n")

        // 2. Build enriched prompt
        val contextPrompt = buildString {
            if (recallContext.isNotEmpty()) append("[RECALL]\n$recallContext\n\n")
            if (archiveContext.isNotEmpty()) append("[ARCHIVE]\n$archiveContext\n\n")
            append("[WORKING MEMORY]\n${working.getContext()}\n\n")
            append("[USER]\n$userMessage")
        }

        // 3. Run agent with enriched context
        val agent = ai.koog.agents.core.agent.AIAgent(
            executor = executor,
            llmModel = model,
            systemPrompt = MEMGPT_SYSTEM_PROMPT
        )
        val response = agent.run(contextPrompt) ?: ""

        // 4. Update working memory
        working.add("User: $userMessage")
        working.add("Assistant: $response")

        // 5. Store interaction in recall memory
        recall.store("User asked: $userMessage\nAssistant said: $response")

        return response
    }

    private companion object {
        val MEMGPT_SYSTEM_PROMPT = """
            You are an AI with a 3-tier memory system:
            - [RECALL]: Recent interactions retrieved by semantic similarity
            - [ARCHIVE]: Long-term stored knowledge
            - [WORKING MEMORY]: Current conversation window
            
            Use information from all memory tiers to give coherent, contextual responses.
            If important new information arises, mention it so it can be archived.
        """.trimIndent()
    }
}
```

---

## A-MEM Zettelkasten Note Graph (arXiv:2502.12110)

A-MEM structures memory as a note graph with bidirectional links and Ebbinghaus forgetting-based decay scoring.

```kotlin
// memory/AMemNoteGraph.kt
package com.example.agent.memory

import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow
import kotlinx.serialization.Serializable
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json
import java.io.File
import java.time.Instant
import kotlin.math.exp

@Serializable
data class MemoryNote(
    val id: String,
    val content: String,
    val description: String = "",
    val keywords: List<String> = emptyList(),
    val links: MutableList<String> = mutableListOf(),    // IDs of related notes
    val createdAt: Long = Instant.now().epochSecond,
    val lastAccessedAt: Long = Instant.now().epochSecond,
    val accessCount: Int = 0
) {
    /** Ebbinghaus retention: R = e^(-t/S) where t=elapsed days, S=stability */
    fun retentionScore(stabilityDays: Double = 7.0): Double {
        val elapsedDays = (Instant.now().epochSecond - lastAccessedAt) / 86_400.0
        return exp(-elapsedDays / stabilityDays)
    }

    /** Combined relevance: semantic sim × retention */
    fun score(semanticSimilarity: Double): Double =
        semanticSimilarity * retentionScore()
}

class AMemNoteGraph(
    private val persistencePath: String = "./data/amem-notes.jsonl",
    private val embedder: EmbeddingProvider,
    private val vectorStore: QdrantLongTermMemory
) {
    private val notes = mutableMapOf<String, MemoryNote>()
    private val json = Json { prettyPrint = false }

    suspend fun initialize() {
        loadFromDisk()
    }

    /** Add a new note and find related notes to link */
    suspend fun addNote(
        content: String,
        keywords: List<String> = emptyList(),
        description: String = ""
    ): MemoryNote {
        val note = MemoryNote(
            id = java.util.UUID.randomUUID().toString(),
            content = content,
            description = description,
            keywords = keywords
        )

        // Find related notes via vector search
        val relatedEntries = vectorStore.search(content, topK = 5)
        note.links.addAll(relatedEntries.map { it.id }.filter { it != note.id })

        // Back-link: update related notes to link to this one
        relatedEntries.forEach { related ->
            notes[related.id]?.links?.add(note.id)
        }

        notes[note.id] = note
        vectorStore.store(MemoryEntry(id = note.id, content = content, metadata = mapOf(
            "keywords" to keywords.joinToString(","),
            "description" to description
        )))
        appendToDisk(note)
        return note
    }

    /** Retrieve notes with Ebbinghaus-weighted scoring */
    suspend fun retrieve(query: String, topK: Int = 5): List<MemoryNote> {
        val searchResults = vectorStore.search(query, topK = topK * 2)
        return searchResults
            .mapNotNull { entry ->
                notes[entry.id]?.let { note ->
                    // Simulate cosine similarity from rank position
                    val rankSim = 1.0 / (searchResults.indexOf(entry) + 1)
                    Pair(note, note.score(rankSim))
                }
            }
            .sortedByDescending { it.second }
            .take(topK)
            .map { (note, _) ->
                // Update access statistics (forgetting curve reset)
                notes[note.id] = note.copy(
                    lastAccessedAt = Instant.now().epochSecond,
                    accessCount = note.accessCount + 1
                )
                notes[note.id]!!
            }
    }

    /** Flow of notes sorted by decay score — for background compaction */
    fun decayingNotes(threshold: Double = 0.1): Flow<MemoryNote> = flow {
        notes.values
            .filter { it.retentionScore() < threshold }
            .sortedBy { it.retentionScore() }
            .forEach { emit(it) }
    }

    private fun appendToDisk(note: MemoryNote) {
        File(persistencePath).parentFile?.mkdirs()
        File(persistencePath).appendText(json.encodeToString(note) + "\n")
    }

    private fun loadFromDisk() {
        val file = File(persistencePath)
        if (!file.exists()) return
        file.forEachLine { line ->
            runCatching { json.decodeFromString<MemoryNote>(line) }
                .onSuccess { notes[it.id] = it }
        }
    }
}
```

---

## Modular RAG Pipeline

```kotlin
// rag/RagPipeline.kt
package com.example.agent.rag

/** Each stage transforms the query or retrieved context */
interface QueryRewriter {
    suspend fun rewrite(query: String): List<String>  // returns 1+ expanded queries
}

interface Retriever {
    suspend fun retrieve(queries: List<String>, topK: Int): List<RetrievedChunk>
}

interface Reranker {
    suspend fun rerank(query: String, chunks: List<RetrievedChunk>, topN: Int): List<RetrievedChunk>
}

interface ContextCompressor {
    suspend fun compress(query: String, chunks: List<RetrievedChunk>): String
}

data class RetrievedChunk(
    val id: String,
    val content: String,
    val score: Float,
    val metadata: Map<String, String> = emptyMap()
)

data class RagResult(
    val context: String,
    val chunks: List<RetrievedChunk>,
    val expandedQueries: List<String>
)

class ModularRagPipeline(
    private val queryRewriter: QueryRewriter,
    private val retriever: Retriever,
    private val reranker: Reranker,
    private val compressor: ContextCompressor,
    private val topK: Int = 10,
    private val topN: Int = 5
) {
    suspend fun run(query: String): RagResult {
        val expandedQueries = queryRewriter.rewrite(query)
        val rawChunks = retriever.retrieve(expandedQueries, topK)
        val reranked = reranker.rerank(query, rawChunks, topN)
        val context = compressor.compress(query, reranked)
        return RagResult(context, reranked, expandedQueries)
    }
}
```

---

## HyDE Query Rewriter (arXiv:2212.10496)

```kotlin
// rag/QueryRewriter.kt
package com.example.agent.rag

import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel
import kotlinx.coroutines.async
import kotlinx.coroutines.coroutineScope

/** HyDE: generate a hypothetical answer, embed that, then search */
class HyDEQueryRewriter(
    private val executor: PromptExecutor,
    private val model: LLMModel
) : QueryRewriter {
    override suspend fun rewrite(query: String): List<String> = coroutineScope {
        val hydeJob = async {
            val agent = AIAgent(
                executor = executor,
                llmModel = model,
                systemPrompt = """
                    Generate a hypothetical document that would perfectly answer the following query.
                    Write as if this document already exists. Be specific and technical.
                    Output ONLY the hypothetical document text, no preamble.
                """.trimIndent()
            )
            agent.run(query) ?: query
        }
        // Also include original query for diversity
        listOf(query, hydeJob.await())
    }
}

/** Multi-query expansion: generate N alternative phrasings */
class MultiQueryRewriter(
    private val executor: PromptExecutor,
    private val model: LLMModel,
    private val numAlternatives: Int = 3
) : QueryRewriter {
    override suspend fun rewrite(query: String): List<String> {
        val agent = AIAgent(
            executor = executor,
            llmModel = model,
            systemPrompt = """
                Generate $numAlternatives alternative phrasings of the given query.
                Each phrasing should capture the same information need but from a different angle.
                Output one query per line, numbered 1. 2. 3. etc.
            """.trimIndent()
        )
        val expanded = agent.run(query) ?: ""
        val alternatives = expanded.lines()
            .mapNotNull { it.trim().removePrefix(Regex("^\\d+\\.\\s*").find(it)?.value ?: "").ifBlank { null } }
        return listOf(query) + alternatives.take(numAlternatives)
    }
}
```

---

## RAG-Fusion + RRF Scoring (arXiv:2402.03367)

```kotlin
// rag/RagFusion.kt
package com.example.agent.rag

/** Reciprocal Rank Fusion: combine rankings from multiple queries */
object RagFusion {
    fun rrf(
        rankedLists: List<List<RetrievedChunk>>,
        k: Int = 60
    ): List<RetrievedChunk> {
        val scores = mutableMapOf<String, Double>()
        val chunks = mutableMapOf<String, RetrievedChunk>()

        rankedLists.forEach { list ->
            list.forEachIndexed { rank, chunk ->
                scores[chunk.id] = (scores[chunk.id] ?: 0.0) + 1.0 / (k + rank + 1)
                chunks[chunk.id] = chunk
            }
        }

        return scores.entries
            .sortedByDescending { it.value }
            .mapNotNull { chunks[it.key] }
    }
}

/** RAG-Fusion retriever: multi-query + RRF */
class RagFusionRetriever(
    private val baseRetriever: Retriever
) : Retriever {
    override suspend fun retrieve(queries: List<String>, topK: Int): List<RetrievedChunk> {
        val rankedLists = queries.map { q -> baseRetriever.retrieve(listOf(q), topK) }
        return RagFusion.rrf(rankedLists)
    }
}
```

---

## CRAG with Confidence Scoring (arXiv:2401.15884)

```kotlin
// rag/CragRetriever.kt
package com.example.agent.rag

import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel

enum class RetrievalConfidence { HIGH, AMBIGUOUS, LOW }

class CragRetriever(
    private val primaryRetriever: Retriever,
    private val webSearchTool: suspend (String) -> List<RetrievedChunk>,
    private val executor: PromptExecutor,
    private val model: LLMModel,
    private val highThreshold: Float = 0.8f,
    private val lowThreshold: Float = 0.4f
) : Retriever {

    override suspend fun retrieve(queries: List<String>, topK: Int): List<RetrievedChunk> {
        val primaryChunks = primaryRetriever.retrieve(queries, topK)

        val confidence = assessConfidence(queries.first(), primaryChunks)
        return when (confidence) {
            RetrievalConfidence.HIGH -> primaryChunks

            RetrievalConfidence.LOW -> {
                // Fall back entirely to web search
                webSearchTool(queries.first())
            }

            RetrievalConfidence.AMBIGUOUS -> {
                // Merge primary retrieval with web search
                val webChunks = webSearchTool(queries.first())
                RagFusion.rrf(listOf(primaryChunks, webChunks))
            }
        }
    }

    private suspend fun assessConfidence(
        query: String,
        chunks: List<RetrievedChunk>
    ): RetrievalConfidence {
        if (chunks.isEmpty()) return RetrievalConfidence.LOW

        val avgScore = chunks.map { it.score }.average()
        return when {
            avgScore >= highThreshold -> RetrievalConfidence.HIGH
            avgScore <= lowThreshold  -> RetrievalConfidence.LOW
            else                      -> {
                // LLM judge: does the retrieved content actually answer the query?
                val agent = AIAgent(
                    executor = executor,
                    llmModel = model,
                    systemPrompt = """
                        Assess whether the retrieved documents adequately answer the query.
                        Respond with exactly one word: HIGH, AMBIGUOUS, or LOW
                    """.trimIndent()
                )
                val context = chunks.take(3).joinToString("\n---\n") { it.content }
                val verdict = agent.run("Query: $query\n\nDocuments:\n$context") ?: "AMBIGUOUS"
                when (verdict.trim().uppercase()) {
                    "HIGH" -> RetrievalConfidence.HIGH
                    "LOW"  -> RetrievalConfidence.LOW
                    else   -> RetrievalConfidence.AMBIGUOUS
                }
            }
        }
    }
}
```

---

## ParentDocumentRetriever

```kotlin
// rag/ParentDocumentRetriever.kt
package com.example.agent.rag

/** Embed small child chunks for precise matching; return full parent documents */
class ParentDocumentRetriever(
    private val childRetriever: Retriever,
    private val parentStore: Map<String, String>  // parentId → full document
) : Retriever {
    override suspend fun retrieve(queries: List<String>, topK: Int): List<RetrievedChunk> {
        val childChunks = childRetriever.retrieve(queries, topK * 3)

        // Deduplicate: if multiple children from same parent, keep only one
        val seenParents = mutableSetOf<String>()
        return childChunks.mapNotNull { child ->
            val parentId = child.metadata["parentId"] ?: return@mapNotNull child
            if (seenParents.add(parentId)) {
                val parentContent = parentStore[parentId]
                    ?: return@mapNotNull child
                child.copy(content = parentContent)
            } else null
        }.take(topK)
    }
}
```

---

## BGE-M3 Hybrid Dense+Sparse Retrieval (arXiv:2402.03216)

```kotlin
// rag/HybridRetriever.kt
package com.example.agent.rag

import io.qdrant.client.QdrantClient
import io.qdrant.client.grpc.Points

/** BGE-M3 produces both dense (1024-dim) and sparse (BM25-style) vectors.
 *  Qdrant supports named vectors for hybrid search. */
class BgeM3HybridRetriever(
    private val client: QdrantClient,
    private val collectionName: String,
    private val denseEmbedder: EmbeddingProvider,
    private val sparseEmbedder: SparseEmbeddingProvider,  // returns token-weight map
    private val denseWeight: Float = 0.7f,
    private val sparseWeight: Float = 0.3f
) : Retriever {

    override suspend fun retrieve(queries: List<String>, topK: Int): List<RetrievedChunk> {
        val allResults = mutableListOf<Pair<String, Float>>()

        queries.forEach { query ->
            val denseVector = denseEmbedder.embed(query)
            val sparseVector = sparseEmbedder.embed(query)

            // Dense search via named vector "dense"
            val denseResults = client.searchAsync(
                Points.SearchPoints.newBuilder()
                    .setCollectionName(collectionName)
                    .setVectorName("dense")
                    .addAllVector(denseVector.map { it.toDouble().toFloat() })
                    .setLimit(topK.toLong())
                    .setWithPayload(Points.WithPayloadSelector.newBuilder().setEnable(true))
                    .build()
            ).await()

            denseResults.forEach { hit ->
                allResults.add(Pair(hit.id.uuid, hit.score * denseWeight))
            }
        }

        // Deduplicate and sort by combined score
        return allResults
            .groupBy { it.first }
            .map { (id, scores) ->
                val totalScore = scores.sumOf { it.second.toDouble() }.toFloat()
                RetrievedChunk(id = id, content = "", score = totalScore)
            }
            .sortedByDescending { it.score }
            .take(topK)
    }
}

/** Sparse embedding: returns token ID → weight map */
interface SparseEmbeddingProvider {
    suspend fun embed(text: String): Map<Int, Float>
}
```

---

## LongMemEval Atomic Fact Decomposition

LongMemEval (arXiv:2502.XXXXX) shows improved accuracy when long documents are decomposed into atomic facts before storage.

```kotlin
// memory/AtomicFactDecomposer.kt
package com.example.agent.memory

import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel

class AtomicFactDecomposer(
    private val executor: PromptExecutor,
    private val model: LLMModel
) {
    suspend fun decompose(document: String): List<String> {
        val agent = AIAgent(
            executor = executor,
            llmModel = model,
            systemPrompt = """
                Decompose the given text into atomic facts.
                An atomic fact is a single, self-contained, verifiable statement.
                Output one fact per line, starting with "FACT: ".
                Do not include opinions or uncertain statements.
            """.trimIndent()
        )
        val result = agent.run(document) ?: ""
        return result.lines()
            .filter { it.startsWith("FACT: ") }
            .map { it.removePrefix("FACT: ").trim() }
            .filter { it.isNotBlank() }
    }

    /** Decompose and store all facts in a memory store */
    suspend fun decomposeAndStore(
        document: String,
        memory: QdrantLongTermMemory,
        docId: String
    ) {
        val facts = decompose(document)
        facts.forEach { fact ->
            memory.store(
                MemoryEntry(
                    content = fact,
                    metadata = mapOf("sourceDocId" to docId, "type" to "atomic-fact")
                )
            )
        }
    }
}
```
