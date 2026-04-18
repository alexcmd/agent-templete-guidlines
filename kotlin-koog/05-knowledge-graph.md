# Knowledge Graph Integration

## Data Models

```kotlin
// kg/KgModels.kt
package com.example.agent.kg

import kotlinx.serialization.Serializable
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json
import java.io.File

@Serializable
data class Entity(
    val id: String,
    val name: String,
    val type: String,           // PERSON, ORG, CONCEPT, FILE, FUNCTION, etc.
    val description: String = "",
    val attributes: Map<String, String> = emptyMap()
)

@Serializable
data class Relation(
    val id: String,
    val sourceId: String,
    val targetId: String,
    val type: String,           // CALLS, IMPORTS, DEFINES, RELATES_TO, etc.
    val weight: Double = 1.0,
    val attributes: Map<String, String> = emptyMap()
)

@Serializable
data class KnowledgeGraph(
    val entities: MutableMap<String, Entity> = mutableMapOf(),
    val relations: MutableList<Relation> = mutableListOf(),
    // adjacency: entityId → list of (relationId, neighborId)
    val adjacency: MutableMap<String, MutableList<Pair<String, String>>> = mutableMapOf()
) {
    fun addEntity(entity: Entity) {
        entities[entity.id] = entity
        adjacency.getOrPut(entity.id) { mutableListOf() }
    }

    fun addRelation(relation: Relation) {
        relations.add(relation)
        adjacency.getOrPut(relation.sourceId) { mutableListOf() }.add(
            Pair(relation.id, relation.targetId)
        )
        // For undirected graphs, add reverse edge
        // adjacency.getOrPut(relation.targetId) { mutableListOf() }.add(
        //     Pair(relation.id, relation.sourceId)
        // )
    }

    fun neighbors(entityId: String): List<Entity> =
        adjacency[entityId]?.mapNotNull { (_, nId) -> entities[nId] } ?: emptyList()

    fun getRelationsBetween(srcId: String, dstId: String): List<Relation> =
        relations.filter { it.sourceId == srcId && it.targetId == dstId }

    /** Serialize to JSON file */
    fun saveToFile(path: String) {
        File(path).apply {
            parentFile?.mkdirs()
            writeText(Json { prettyPrint = true }.encodeToString(this@KnowledgeGraph))
        }
    }

    companion object {
        fun loadFromFile(path: String): KnowledgeGraph =
            Json.decodeFromString(File(path).readText())
    }
}
```

---

## Triple Extraction with Koog Agent

```kotlin
// kg/TripleExtractor.kt
package com.example.agent.kg

import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json
import java.util.UUID

@Serializable
data class ExtractedTriple(
    val subject: String,
    val subjectType: String = "CONCEPT",
    val predicate: String,
    val obj: String,           // "object" is reserved in Kotlin
    val objectType: String = "CONCEPT",
    val confidence: Double = 1.0
)

class TripleExtractor(
    private val executor: PromptExecutor,
    private val model: LLMModel
) {
    private val json = Json { ignoreUnknownKeys = true }

    private val EXTRACTION_PROMPT = """
        Extract knowledge graph triples from the given text.
        A triple is: (subject, predicate, object) where:
        - subject: a named entity, concept, or thing
        - predicate: the relationship (use verbs like CALLS, IMPORTS, DEFINES, IS_A, PART_OF, USES, etc.)
        - object: another entity, concept, or thing
        
        Also classify each entity's type: PERSON, ORG, FILE, FUNCTION, CLASS, CONCEPT, LOCATION, DATE
        
        Return ONLY a JSON array:
        [
          {"subject": "...", "subjectType": "...", "predicate": "...", "obj": "...", "objectType": "...", "confidence": 0.9},
          ...
        ]
        
        Extract all significant relationships. Confidence: 0.0-1.0
    """.trimIndent()

    suspend fun extract(text: String): List<ExtractedTriple> {
        val agent = AIAgent(
            executor = executor,
            llmModel = model,
            systemPrompt = EXTRACTION_PROMPT
        )
        val raw = agent.run(text) ?: return emptyList()

        // Extract JSON array from response
        val jsonStr = Regex("\\[.*\\]", RegexOption.DOT_MATCHES_ALL).find(raw)?.value
            ?: return emptyList()

        return runCatching {
            json.decodeFromString<List<ExtractedTriple>>(jsonStr)
                .filter { it.confidence >= 0.5 }
        }.getOrElse { emptyList() }
    }

    /** Extract triples and merge into an existing KnowledgeGraph */
    suspend fun extractAndMerge(text: String, graph: KnowledgeGraph) {
        val triples = extract(text)
        triples.forEach { triple ->
            val srcId = triple.subject.lowercase().replace(" ", "_")
            val dstId = triple.obj.lowercase().replace(" ", "_")

            graph.addEntity(Entity(id = srcId, name = triple.subject, type = triple.subjectType))
            graph.addEntity(Entity(id = dstId, name = triple.obj, type = triple.objectType))
            graph.addRelation(Relation(
                id = UUID.randomUUID().toString(),
                sourceId = srcId,
                targetId = dstId,
                type = triple.predicate,
                weight = triple.confidence
            ))
        }
    }
}
```

---

## Neo4j Integration

```kotlin
// kg/Neo4jStore.kt
package com.example.agent.kg

import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import org.neo4j.driver.AuthTokens
import org.neo4j.driver.Driver
import org.neo4j.driver.GraphDatabase
import org.neo4j.driver.Values.parameters

class Neo4jStore(uri: String, user: String, password: String) : AutoCloseable {
    private val driver: Driver = GraphDatabase.driver(uri, AuthTokens.basic(user, password))

    suspend fun initialize() = withContext(Dispatchers.IO) {
        driver.session().use { session ->
            session.run("CREATE INDEX entity_id IF NOT EXISTS FOR (e:Entity) ON (e.id)")
            session.run("CREATE INDEX entity_name IF NOT EXISTS FOR (e:Entity) ON (e.name)")
        }
    }

    suspend fun upsertEntity(entity: Entity) = withContext(Dispatchers.IO) {
        driver.session().use { session ->
            session.run(
                """
                MERGE (e:Entity {id: ${'$'}id})
                SET e.name = ${'$'}name,
                    e.type = ${'$'}type,
                    e.description = ${'$'}description
                """.trimIndent(),
                parameters(
                    "id", entity.id,
                    "name", entity.name,
                    "type", entity.type,
                    "description", entity.description
                )
            )
        }
    }

    suspend fun upsertRelation(relation: Relation) = withContext(Dispatchers.IO) {
        driver.session().use { session ->
            session.run(
                """
                MATCH (src:Entity {id: ${'$'}srcId})
                MATCH (dst:Entity {id: ${'$'}dstId})
                MERGE (src)-[r:RELATION {id: ${'$'}id}]->(dst)
                SET r.type = ${'$'}type,
                    r.weight = ${'$'}weight
                """.trimIndent(),
                parameters(
                    "id", relation.id,
                    "srcId", relation.sourceId,
                    "dstId", relation.targetId,
                    "type", relation.type,
                    "weight", relation.weight
                )
            )
        }
    }

    suspend fun importGraph(graph: KnowledgeGraph) = withContext(Dispatchers.IO) {
        graph.entities.values.forEach { upsertEntity(it) }
        graph.relations.forEach { upsertRelation(it) }
    }

    /** Cypher search: find paths between entities by relation type */
    suspend fun findPaths(
        fromName: String,
        toName: String,
        maxHops: Int = 4
    ): List<String> = withContext(Dispatchers.IO) {
        driver.session().use { session ->
            val result = session.run(
                """
                MATCH path = (a:Entity {name: ${'$'}from})-[*1..$maxHops]->(b:Entity {name: ${'$'}to})
                RETURN [node IN nodes(path) | node.name] AS names,
                       [rel  IN relationships(path) | rel.type] AS rels
                LIMIT 10
                """.trimIndent(),
                parameters("from", fromName, "to", toName)
            )
            result.list { record ->
                val names = record["names"].asList { it.asString() }
                val rels = record["rels"].asList { it.asString() }
                names.zip(rels + listOf("")).joinToString(" → ") { (n, r) ->
                    if (r.isEmpty()) n else "$n -[$r]→"
                }
            }
        }
    }

    /** Full-text entity search */
    suspend fun searchEntities(query: String, limit: Int = 10): List<Entity> =
        withContext(Dispatchers.IO) {
            driver.session().use { session ->
                val result = session.run(
                    """
                    MATCH (e:Entity)
                    WHERE toLower(e.name) CONTAINS toLower(${'$'}query)
                       OR toLower(e.description) CONTAINS toLower(${'$'}query)
                    RETURN e LIMIT ${'$'}limit
                    """.trimIndent(),
                    parameters("query", query, "limit", limit)
                )
                result.list { record ->
                    val node = record["e"].asNode()
                    Entity(
                        id = node["id"].asString(),
                        name = node["name"].asString(),
                        type = node["type"].asString("CONCEPT"),
                        description = node["description"].asString("")
                    )
                }
            }
        }

    override fun close() = driver.close()
}
```

---

## HippoRAG — Personalized PageRank (arXiv:2405.14831)

HippoRAG uses the KG structure to propagate relevance from seed entities via Personalized PageRank (PPR), finding non-obvious connections.

```kotlin
// kg/HippoRag.kt
package com.example.agent.kg

import com.example.agent.memory.EmbeddingProvider
import com.example.agent.memory.QdrantLongTermMemory
import kotlin.math.abs

class HippoRag(
    private val graph: KnowledgeGraph,
    private val entityEmbeddings: QdrantLongTermMemory,
    private val embedder: EmbeddingProvider,
    private val dampingFactor: Double = 0.85,
    private val maxIterations: Int = 50,
    private val tolerance: Double = 1e-6
) {
    /** Retrieve relevant subgraph context for a query using PPR */
    suspend fun retrieve(query: String, topK: Int = 5): List<Entity> {
        // 1. Find seed entities: embed query, retrieve top matching entities
        val seedEntities = entityEmbeddings.search(query, topK = 3)
            .mapNotNull { entry -> graph.entities[entry.id] }

        if (seedEntities.isEmpty()) return emptyList()

        // 2. Build personalization vector (uniform over seeds)
        val entityIds = graph.entities.keys.toList()
        val idxMap = entityIds.mapIndexed { i, id -> id to i }.toMap()
        val n = entityIds.size
        val personalization = DoubleArray(n)
        seedEntities.forEach { entity ->
            idxMap[entity.id]?.let { idx -> personalization[idx] = 1.0 / seedEntities.size }
        }

        // 3. Build transition matrix (column-normalized adjacency)
        val matrix = Array(n) { DoubleArray(n) }
        graph.relations.forEach { rel ->
            val si = idxMap[rel.sourceId] ?: return@forEach
            val di = idxMap[rel.targetId] ?: return@forEach
            matrix[di][si] += rel.weight
        }
        // Column-normalize
        for (j in 0 until n) {
            val colSum = matrix.sumOf { row -> row[j] }
            if (colSum > 0) { for (i in 0 until n) matrix[i][j] /= colSum }
        }

        // 4. PPR power iteration
        var scores = DoubleArray(n) { 1.0 / n }
        repeat(maxIterations) {
            val newScores = DoubleArray(n)
            for (i in 0 until n) {
                newScores[i] = (1 - dampingFactor) * personalization[i] +
                    dampingFactor * (0 until n).sumOf { j -> matrix[i][j] * scores[j] }
            }
            val delta = (0 until n).sumOf { abs(newScores[it] - scores[it]) }
            scores = newScores
            if (delta < tolerance) return@repeat
        }

        // 5. Return top-K entities by PPR score
        return entityIds.zip(scores.toList())
            .sortedByDescending { it.second }
            .take(topK)
            .mapNotNull { (id, _) -> graph.entities[id] }
    }
}
```

---

## LightRAG Dual-Level Retrieval (arXiv:2410.05779)

LightRAG supports both local (entity neighborhood) and global (community/theme) retrieval modes.

```kotlin
// kg/LightRag.kt
package com.example.agent.kg

import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel

class LightRag(
    private val graph: KnowledgeGraph,
    private val executor: PromptExecutor,
    private val model: LLMModel
) {
    enum class RetrievalMode { LOCAL, GLOBAL, HYBRID }

    suspend fun retrieve(
        query: String,
        mode: RetrievalMode = RetrievalMode.HYBRID,
        topK: Int = 5
    ): String = when (mode) {
        RetrievalMode.LOCAL  -> localRetrieval(query, topK)
        RetrievalMode.GLOBAL -> globalRetrieval(query)
        RetrievalMode.HYBRID -> buildString {
            append("=== LOCAL CONTEXT ===\n")
            append(localRetrieval(query, topK))
            append("\n=== GLOBAL CONTEXT ===\n")
            append(globalRetrieval(query))
        }
    }

    /** Local: find most relevant entities + their 1-hop neighborhood */
    private fun localRetrieval(query: String, topK: Int): String {
        val queryTokens = query.lowercase().split(Regex("\\W+")).toSet()

        // Score entities by name overlap with query
        val scored = graph.entities.values.map { entity ->
            val nameTokens = entity.name.lowercase().split(Regex("\\W+")).toSet()
            val overlap = nameTokens.intersect(queryTokens).size.toDouble() / (queryTokens.size + 1)
            entity to overlap
        }.sortedByDescending { it.second }.take(topK)

        return scored.joinToString("\n\n") { (entity, _) ->
            val neighbors = graph.neighbors(entity.id)
            val rels = graph.relations.filter { it.sourceId == entity.id }
            buildString {
                append("Entity: ${entity.name} (${entity.type})\n")
                if (entity.description.isNotBlank()) append("  ${entity.description}\n")
                rels.forEach { rel ->
                    val target = graph.entities[rel.targetId]?.name ?: rel.targetId
                    append("  → [${rel.type}] → $target\n")
                }
            }
        }
    }

    /** Global: summarize high-level themes/communities in the graph */
    private suspend fun globalRetrieval(query: String): String {
        // Use entity type distribution as community summary
        val typeCounts = graph.entities.values.groupBy { it.type }
            .mapValues { it.value.size }
            .entries.sortedByDescending { it.value }

        val summary = typeCounts.joinToString(", ") { "${it.key}: ${it.value} entities" }
        val agent = AIAgent(
            executor = executor,
            llmModel = model,
            systemPrompt = "Summarize what topics this knowledge graph covers based on its entity distribution."
        )
        return agent.run("Entity distribution: $summary\nQuery context: $query") ?: summary
    }
}
```

---

## GraphRAG Community Detection (arXiv:2404.16130)

GraphRAG uses Leiden community detection; here we implement a simpler connected-components approach with summary generation.

```kotlin
// kg/GraphRag.kt
package com.example.agent.kg

import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel

data class Community(
    val id: Int,
    val entityIds: Set<String>,
    val summary: String = ""
)

class GraphRag(
    private val graph: KnowledgeGraph,
    private val executor: PromptExecutor,
    private val model: LLMModel
) {
    /** Simple connected-components community detection (Leiden approximation) */
    fun detectCommunities(): List<Community> {
        val visited = mutableSetOf<String>()
        val communities = mutableListOf<Community>()
        var communityId = 0

        graph.entities.keys.forEach { startId ->
            if (startId !in visited) {
                val component = mutableSetOf<String>()
                val queue = ArrayDeque<String>()
                queue.add(startId)

                while (queue.isNotEmpty()) {
                    val current = queue.removeFirst()
                    if (current in visited) continue
                    visited.add(current)
                    component.add(current)
                    graph.neighbors(current).forEach { neighbor ->
                        if (neighbor.id !in visited) queue.add(neighbor.id)
                    }
                    // Also traverse reverse edges
                    graph.relations.filter { it.targetId == current }.forEach { rel ->
                        if (rel.sourceId !in visited) queue.add(rel.sourceId)
                    }
                }

                communities.add(Community(communityId++, component))
            }
        }
        return communities
    }

    /** Generate LLM summary for each community */
    suspend fun buildCommunitySummaries(communities: List<Community>): List<Community> {
        return communities.map { community ->
            val entityNames = community.entityIds
                .mapNotNull { graph.entities[it]?.name }
                .take(20)
            val relSummary = graph.relations
                .filter { it.sourceId in community.entityIds }
                .take(15)
                .joinToString("; ") { rel ->
                    val src = graph.entities[rel.sourceId]?.name ?: rel.sourceId
                    val dst = graph.entities[rel.targetId]?.name ?: rel.targetId
                    "$src ${rel.type} $dst"
                }

            val agent = AIAgent(
                executor = executor,
                llmModel = model,
                systemPrompt = "Summarize what this knowledge graph community represents in 2-3 sentences."
            )
            val summary = agent.run(
                "Entities: ${entityNames.joinToString(", ")}\nRelationships: $relSummary"
            ) ?: "Community of ${community.entityIds.size} entities."

            community.copy(summary = summary)
        }
    }

    /** Query: find the most relevant communities, then entities within them */
    suspend fun query(userQuery: String, topCommunities: Int = 3): String {
        val communities = detectCommunities()
        val withSummaries = buildCommunitySummaries(communities)

        // Rank communities by keyword overlap with query
        val queryTokens = userQuery.lowercase().split(Regex("\\W+")).toSet()
        val ranked = withSummaries.sortedByDescending { community ->
            val tokens = community.summary.lowercase().split(Regex("\\W+")).toSet()
            tokens.intersect(queryTokens).size
        }.take(topCommunities)

        return ranked.joinToString("\n\n") { community ->
            val entities = community.entityIds
                .mapNotNull { graph.entities[it] }
                .take(10)
                .joinToString(", ") { "${it.name} (${it.type})" }
            """
            Community ${community.id}: ${community.summary}
            Key entities: $entities
            """.trimIndent()
        }
    }
}
```

---

## PathRAG Flow-Based Pruning (arXiv:2502.14902)

PathRAG selects the most information-rich paths through the graph using flow-based pruning.

```kotlin
// kg/PathRag.kt
package com.example.agent.kg

data class KgPath(
    val entities: List<Entity>,
    val relations: List<Relation>,
    val flowScore: Double
)

class PathRag(private val graph: KnowledgeGraph) {

    /** Find and prune paths using max-flow scoring */
    fun retrievePaths(
        seedEntityIds: List<String>,
        maxPathLength: Int = 4,
        topK: Int = 5
    ): List<KgPath> {
        val paths = mutableListOf<KgPath>()

        seedEntityIds.forEach { seedId ->
            dfs(
                currentId = seedId,
                path = mutableListOf(),
                rels = mutableListOf(),
                visited = mutableSetOf(),
                maxDepth = maxPathLength,
                paths = paths
            )
        }

        // Score paths by flow: product of edge weights, normalized by length
        return paths
            .map { path ->
                val flowScore = if (path.relations.isEmpty()) 0.0
                else path.relations.map { it.weight }.fold(1.0) { acc, w -> acc * w } /
                     path.relations.size
                path.copy(flowScore = flowScore)
            }
            .sortedByDescending { it.flowScore }
            .take(topK)
    }

    private fun dfs(
        currentId: String,
        path: MutableList<Entity>,
        rels: MutableList<Relation>,
        visited: MutableSet<String>,
        maxDepth: Int,
        paths: MutableList<KgPath>
    ) {
        val entity = graph.entities[currentId] ?: return
        visited.add(currentId)
        path.add(entity)

        if (path.size > 1) {
            paths.add(KgPath(path.toList(), rels.toList(), 0.0))
        }

        if (path.size < maxDepth) {
            graph.relations.filter { it.sourceId == currentId && it.targetId !in visited }
                .forEach { rel ->
                    rels.add(rel)
                    dfs(rel.targetId, path, rels, visited, maxDepth, paths)
                    rels.removeLast()
                }
        }

        path.removeLast()
        visited.remove(currentId)
    }

    fun formatPaths(paths: List<KgPath>): String =
        paths.joinToString("\n") { path ->
            path.entities.mapIndexed { i, entity ->
                if (i < path.relations.size)
                    "${entity.name} -[${path.relations[i].type}]→"
                else entity.name
            }.joinToString(" ") + " (score: %.3f)".format(path.flowScore)
        }
}
```
