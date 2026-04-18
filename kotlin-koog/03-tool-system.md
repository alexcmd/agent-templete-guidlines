# Tool System — Complete Reference

## Three Ways to Define a Tool

Koog offers three authoring styles. Choose based on complexity:

| Style | Best For |
|-------|----------|
| `@Tool` suspend fun | Simple, single-responsibility tools |
| `SimpleTool<Args>` class | Tools needing DI, lifecycle, or state |
| `ToolSet` class | Grouping related tools (e.g., all file ops) |

---

## Style 1: @Tool Suspend Function (Simplest)

```kotlin
// tools/FileTools.kt
package com.example.agent.tools

import ai.koog.agents.core.tools.annotations.Tool
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import kotlinx.serialization.Serializable
import java.io.File
import java.nio.file.Path
import kotlin.io.path.isRegularFile
import kotlin.io.path.walk

@Serializable
data class ReadFileArgs(
    val path: String,
    val maxBytes: Int = 100_000
)

@Tool(
    name = "readFile",
    description = "Read the text content of a file. Returns up to maxBytes characters."
)
suspend fun readFile(args: ReadFileArgs): String = withContext(Dispatchers.IO) {
    val file = File(args.path)
    require(file.exists()) { "File not found: ${args.path}" }
    require(file.isFile)   { "Path is not a file: ${args.path}" }
    file.readText().take(args.maxBytes)
}

@Serializable
data class WriteFileArgs(
    val path: String,
    val content: String,
    val append: Boolean = false
)

@Tool(
    name = "writeFile",
    description = "Write or append text content to a file. Creates parent directories if needed."
)
suspend fun writeFile(args: WriteFileArgs): String = withContext(Dispatchers.IO) {
    val file = File(args.path)
    file.parentFile?.mkdirs()
    if (args.append) file.appendText(args.content)
    else             file.writeText(args.content)
    "Written ${args.content.length} chars to ${args.path}"
}

@Serializable
data class ListFilesArgs(
    val directory: String,
    val glob: String = "**/*",
    val maxResults: Int = 100
)

@Tool(
    name = "listFiles",
    description = "List files in a directory matching an optional glob pattern."
)
suspend fun listFiles(args: ListFilesArgs): String = withContext(Dispatchers.IO) {
    val dir = Path.of(args.directory)
    val matcher = dir.fileSystem.getPathMatcher("glob:${args.glob}")
    dir.walk()
        .filter { it.isRegularFile() && matcher.matches(it.fileName) }
        .take(args.maxResults)
        .map { it.toString() }
        .joinToString("\n")
        .ifEmpty { "No files found." }
}
```

---

## Style 2: SimpleTool<Args> Class (With DI)

```kotlin
// tools/DatabaseQueryTool.kt
package com.example.agent.tools

import ai.koog.agents.core.tools.SimpleTool
import ai.koog.agents.core.tools.ToolDescriptor
import ai.koog.agents.core.tools.ToolParameterDescriptor
import ai.koog.agents.core.tools.ToolParameterType
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import kotlinx.serialization.Serializable
import org.jetbrains.exposed.sql.Database
import org.jetbrains.exposed.sql.transactions.transaction

@Serializable
data class QueryArgs(
    val sql: String,
    val maxRows: Int = 50
)

class DatabaseQueryTool(
    private val database: Database,
    private val allowedPrefixes: List<String> = listOf("SELECT", "WITH")
) : SimpleTool<QueryArgs>() {

    override val descriptor = ToolDescriptor(
        name = "databaseQuery",
        description = "Execute a read-only SQL SELECT query and return results as CSV.",
        parameters = listOf(
            ToolParameterDescriptor(
                name = "sql",
                description = "SQL SELECT statement to execute",
                type = ToolParameterType.String,
                required = true
            ),
            ToolParameterDescriptor(
                name = "maxRows",
                description = "Maximum number of rows to return (default 50)",
                type = ToolParameterType.Integer,
                required = false
            )
        )
    )

    override suspend fun execute(args: QueryArgs): String = withContext(Dispatchers.IO) {
        val upperSql = args.sql.trimStart().uppercase()
        val allowed = allowedPrefixes.any { upperSql.startsWith(it) }
        require(allowed) { "Only SELECT/WITH queries are allowed. Got: ${args.sql.take(20)}" }

        transaction(database) {
            exec(args.sql) { rs ->
                val meta = rs.metaData
                val cols = (1..meta.columnCount).map { meta.getColumnName(it) }
                val rows = mutableListOf(cols.joinToString(","))
                var count = 0
                while (rs.next() && count < args.maxRows) {
                    rows.add(cols.indices.map { rs.getString(it + 1) ?: "NULL" }.joinToString(","))
                    count++
                }
                rows.joinToString("\n")
            } ?: "No results."
        }
    }
}
```

---

## Style 3: ToolSet Class (Grouped Tools)

```kotlin
// tools/WebSearchToolSet.kt
package com.example.agent.tools

import ai.koog.agents.core.tools.ToolSet
import ai.koog.agents.core.tools.annotations.Tool
import io.ktor.client.*
import io.ktor.client.engine.cio.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.serialization.kotlinx.json.*
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json

@Serializable
data class WebSearchArgs(val query: String, val numResults: Int = 5)

@Serializable
data class FetchUrlArgs(val url: String)

@Serializable
data class ExtractTextArgs(val url: String, val selector: String = "body")

class WebSearchToolSet(
    private val searchApiKey: String,
    private val searchEngineId: String
) : ToolSet {

    private val httpClient = HttpClient(CIO) {
        install(ContentNegotiation) { json(Json { ignoreUnknownKeys = true }) }
    }

    @Tool(name = "webSearch", description = "Search the web using Google Custom Search API.")
    suspend fun webSearch(args: WebSearchArgs): String = withContext(Dispatchers.IO) {
        val response = httpClient.get(
            "https://www.googleapis.com/customsearch/v1"
        ) {
            parameter("key", searchApiKey)
            parameter("cx", searchEngineId)
            parameter("q", args.query)
            parameter("num", args.numResults.coerceIn(1, 10))
        }
        response.bodyAsText()
    }

    @Tool(name = "fetchUrl", description = "Fetch raw HTML/text content from a URL.")
    suspend fun fetchUrl(args: FetchUrlArgs): String = withContext(Dispatchers.IO) {
        httpClient.get(args.url).bodyAsText().take(50_000)
    }

    @Tool(name = "extractText", description = "Fetch a URL and extract visible text content.")
    suspend fun extractText(args: ExtractTextArgs): String = withContext(Dispatchers.IO) {
        // Simple text extraction — use jsoup for production
        val html = httpClient.get(args.url).bodyAsText()
        html.replace(Regex("<[^>]+>"), " ")
            .replace(Regex("\\s+"), " ")
            .trim()
            .take(20_000)
    }
}
```

---

## ToolRegistry Construction and Filtering

```kotlin
// tools/ToolRegistryFactory.kt
package com.example.agent.tools

import ai.koog.agents.core.tools.ToolRegistry
import org.jetbrains.exposed.sql.Database

object ToolRegistryFactory {

    /** Full registry — all tools available */
    fun buildFull(database: Database, searchApiKey: String, searchEngineId: String): ToolRegistry {
        return ToolRegistry {
            // @Tool suspend funs
            tool(::readFile)
            tool(::writeFile)
            tool(::listFiles)

            // SimpleTool instances
            tool(DatabaseQueryTool(database))

            // ToolSet instances
            toolSet(WebSearchToolSet(searchApiKey, searchEngineId))
        }
    }

    /** Read-only registry — no writes, no shell */
    fun buildReadOnly(database: Database): ToolRegistry {
        return ToolRegistry {
            tool(::readFile)
            tool(::listFiles)
            tool(DatabaseQueryTool(database))
        }
    }

    /** Filter to a named subset at runtime */
    fun filterByRole(registry: ToolRegistry, role: String): ToolRegistry {
        val allowedNames = when (role) {
            "researcher" -> setOf("webSearch", "fetchUrl", "readFile")
            "developer"  -> setOf("readFile", "writeFile", "runBash", "runTests")
            "analyst"    -> setOf("databaseQuery", "readFile", "listFiles")
            else         -> emptySet()
        }
        return if (allowedNames.isEmpty()) registry
               else registry.filterByNames(allowedNames)
    }
}
```

---

## Permission Enforcement via EventHandler

```kotlin
// tools/PermissionEnforcer.kt
package com.example.agent.tools

import ai.koog.agents.core.feature.handler.EventHandler
import ai.koog.agents.core.feature.model.ToolCall
import io.github.oshai.kotlinlogging.KotlinLogging

private val log = KotlinLogging.logger {}

data class PermissionPolicy(
    val allowedTools: Set<String> = emptySet(),   // empty = all allowed
    val deniedTools:  Set<String> = emptySet(),
    val auditLog: Boolean = true
)

fun buildPermissionHandler(policy: PermissionPolicy): EventHandler {
    return EventHandler {
        // Pre-tool: block disallowed tools before execution
        onPreToolUse { toolCall: ToolCall ->
            val name = toolCall.toolName
            if (policy.deniedTools.contains(name)) {
                throw SecurityException("Tool '$name' is explicitly denied by policy")
            }
            if (policy.allowedTools.isNotEmpty() && !policy.allowedTools.contains(name)) {
                throw SecurityException("Tool '$name' is not in the allowed list")
            }
            if (policy.auditLog) {
                log.info { "[AUDIT] Tool called: $name args=${toolCall.args}" }
            }
        }

        // Post-tool: log results (redact sensitive output)
        onPostToolUse { toolCall, result ->
            if (policy.auditLog) {
                val safeResult = result.take(200).replace(Regex("[A-Za-z0-9+/=]{40,}"), "<REDACTED>")
                log.info { "[AUDIT] Tool result: ${toolCall.toolName} → $safeResult" }
            }
        }

        // Post-failure: alert and circuit-break
        onPostToolUseFailure { toolCall, error ->
            log.error { "[AUDIT] Tool FAILED: ${toolCall.toolName} → ${error.message}" }
        }
    }
}

// Wire into agent:
val agent = AIAgent(
    executor = executor,
    llmModel = model,
    systemPrompt = systemPrompt,
    toolRegistry = toolRegistry,
    eventHandler = buildPermissionHandler(
        PermissionPolicy(
            deniedTools = setOf("runBash"),  // never allow bash in production
            auditLog = true
        )
    )
)
```

---

## BashTool (With Sandboxing)

```kotlin
// tools/BashTool.kt
package com.example.agent.tools

import ai.koog.agents.core.tools.annotations.Tool
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import kotlinx.coroutines.withTimeoutOrNull
import kotlinx.serialization.Serializable

@Serializable
data class BashArgs(
    val command: String,
    val workingDir: String = ".",
    val timeoutSeconds: Int = 30
)

private val BLOCKED_COMMANDS = listOf("rm -rf /", "sudo rm", "dd if=", "mkfs", "> /dev/")

@Tool(name = "runBash", description = "Execute a bash command and return stdout+stderr.")
suspend fun runBash(args: BashArgs): String = withContext(Dispatchers.IO) {
    // Safety check
    BLOCKED_COMMANDS.forEach { blocked ->
        require(!args.command.contains(blocked)) {
            "Blocked command pattern detected: $blocked"
        }
    }

    val result = withTimeoutOrNull(args.timeoutSeconds * 1_000L) {
        val proc = ProcessBuilder("/bin/bash", "-c", args.command)
            .directory(java.io.File(args.workingDir))
            .redirectErrorStream(true)
            .start()
        val output = proc.inputStream.bufferedReader().readText()
        val exitCode = proc.waitFor()
        "Exit: $exitCode\n$output".take(50_000)
    }
    result ?: "Command timed out after ${args.timeoutSeconds}s"
}
```

---

## MCP Tool Integration

Koog supports the Model Context Protocol for discovering and calling tools exposed by external servers.

```kotlin
// tools/McpTools.kt
package com.example.agent.tools

import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.agents.mcp.McpToolsProvider
import ai.koog.agents.mcp.transport.StdioMcpTransport

/** Integrate tools from an MCP server via stdio transport */
suspend fun buildMcpToolRegistry(
    mcpServerCommand: List<String>  // e.g. ["npx", "@modelcontextprotocol/server-filesystem"]
): ToolRegistry {
    val transport = StdioMcpTransport(command = mcpServerCommand)
    val provider = McpToolsProvider(transport)
    val mcpTools = provider.discoverTools()

    return ToolRegistry {
        mcpTools.forEach { tool(it) }
    }
}

// LangChain4j MCP client (alternative, more mature for complex MCP servers)
/*
import dev.langchain4j.agent.tool.ToolSpecification
import dev.langchain4j.mcp.client.DefaultMcpClient
import dev.langchain4j.mcp.client.transport.stdio.StdioMcpTransport as LC4jStdioTransport

fun buildLc4jMcpClient(): DefaultMcpClient {
    val transport = LC4jStdioTransport(
        command = listOf("npx", "@modelcontextprotocol/server-github"),
        environment = mapOf("GITHUB_TOKEN" to System.getenv("GITHUB_TOKEN"))
    )
    return DefaultMcpClient.builder()
        .transport(transport)
        .toolExecutionTimeout(java.time.Duration.ofSeconds(30))
        .build()
}
*/
```

---

## Tool Serialization for A2A Protocol

When exposing an agent via A2A, tool schemas are automatically serialized as part of the AgentCard capabilities declaration.

```kotlin
// multiagent/AgentCardBuilder.kt
package com.example.agent.multiagent

import ai.koog.agents.a2a.protocol.AgentCapabilities
import ai.koog.agents.a2a.protocol.AgentCard
import ai.koog.agents.a2a.protocol.AgentSkill
import ai.koog.agents.core.tools.ToolRegistry

fun buildAgentCard(
    name: String,
    description: String,
    toolRegistry: ToolRegistry,
    baseUrl: String
): AgentCard {
    val skills = toolRegistry.tools.map { tool ->
        AgentSkill(
            id = tool.descriptor.name,
            name = tool.descriptor.name,
            description = tool.descriptor.description,
            tags = listOf("tool")
        )
    }

    return AgentCard(
        name = name,
        description = description,
        url = baseUrl,
        version = "1.0",
        capabilities = AgentCapabilities(
            streaming = true,
            pushNotifications = false
        ),
        skills = skills
    )
}
```

---

## ToolBench DFSDT Pattern for Multi-Step Tool Planning

The Depth-First Search with Decision Trees (DFSDT) pattern from ToolBench enables systematic multi-step tool use planning:

```kotlin
// tools/DfsdtPlanner.kt
package com.example.agent.tools

import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel

data class ToolNode(
    val toolName: String,
    val args: String,
    val result: String = "",
    val children: MutableList<ToolNode> = mutableListOf(),
    val depth: Int = 0
)

class DfsdtPlanner(
    private val executor: PromptExecutor,
    private val model: LLMModel,
    private val toolRegistry: ToolRegistry,
    private val maxDepth: Int = 5,
    private val maxBranches: Int = 3
) {
    private val DFSDT_PROMPT = """
        You are a tool-use planner. Given a task, reason about which tools to call and in what order.
        You can backtrack if a tool call fails or returns insufficient information.
        Always explain your reasoning before each tool call.
        When the task is complete, output: FINAL ANSWER: <answer>
    """.trimIndent()

    suspend fun plan(task: String): String {
        val agent = AIAgent(
            executor = executor,
            llmModel = model,
            systemPrompt = DFSDT_PROMPT,
            toolRegistry = toolRegistry
        )
        // DFSDT emerges from the ReAct loop with backtracking prompts
        return agent.run(
            """
            Task: $task
            
            Think step by step. If a tool call doesn't give useful results, 
            try a different approach (backtrack). Max depth: $maxDepth.
            """.trimIndent()
        ) ?: "Planning failed."
    }
}
```

---

## Complete: FileReadTool, BashTool, WebSearchTool, DatabaseQueryTool Wired Together

```kotlin
// tools/FullToolRegistry.kt
package com.example.agent.tools

import ai.koog.agents.core.tools.ToolRegistry
import org.jetbrains.exposed.sql.Database

fun buildFullToolRegistry(
    db: Database,
    searchApiKey: String,
    searchEngineId: String,
    enableBash: Boolean = false   // disabled by default for safety
): ToolRegistry = ToolRegistry {
    // File operations
    tool(::readFile)
    tool(::writeFile)
    tool(::listFiles)

    // Database (read-only enforced in the tool itself)
    tool(DatabaseQueryTool(db, allowedPrefixes = listOf("SELECT", "WITH")))

    // Web search + fetch
    toolSet(WebSearchToolSet(searchApiKey, searchEngineId))

    // Bash — gated by flag
    if (enableBash) {
        tool(::runBash)
    }
}

// Usage in main:
/*
val registry = buildFullToolRegistry(
    db = database,
    searchApiKey = config.searchApiKey,
    searchEngineId = config.searchEngineId,
    enableBash = config.featureFlags.enableBash
)
val filteredRegistry = registry.filterByNames(setOf("readFile", "webSearch", "databaseQuery"))
*/
```
