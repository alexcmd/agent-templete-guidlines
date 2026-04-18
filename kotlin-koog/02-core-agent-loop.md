# Core Agent Loop — Strategy Graph

## The Koog Strategy Graph Model

Koog represents agent control flow as a directed graph of **nodes** connected by **typed edges**. Each node is a suspend function that receives the current agent state and returns an edge type that determines the next node. This gives compile-time safety: you cannot wire an edge that no node emits.

```
Start ──→ nodeThink (LLM) ──→ nodeAct (tool) ──→ nodeObserve ──→ nodeThink
                    │                                              │
                    └──────────────── nodeFinish ◄────────────────┘
                                      (when done = true)
```

---

## Simple AIAgent (Prototyping)

```kotlin
// agent/SimpleAgent.kt
package com.example.agent.agent

import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel
import ai.koog.agents.core.agent.config.AIAgentConfig
import ai.koog.agents.core.agent.config.PromptConfig

class SimpleAgent(
    private val executor: PromptExecutor,
    private val model: LLMModel,
    private val systemPrompt: String,
    private val toolRegistry: ToolRegistry? = null
) {
    suspend fun run(userMessage: String): String {
        val agent = AIAgent(
            executor = executor,
            llmModel = model,
            systemPrompt = systemPrompt,
            toolRegistry = toolRegistry
        )
        return agent.run(userMessage) ?: ""
    }

    /** Streaming variant — returns Flow<String> of partial tokens */
    fun stream(userMessage: String): kotlinx.coroutines.flow.Flow<String> {
        val agent = AIAgent(
            executor = executor,
            llmModel = model,
            systemPrompt = systemPrompt,
            toolRegistry = toolRegistry
        )
        return agent.runStreaming(userMessage)
    }
}
```

---

## Graph-Based Strategy: Full Anatomy

```kotlin
// agent/GraphAgent.kt
package com.example.agent.agent

import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.core.agent.config.AIAgentConfig
import ai.koog.agents.core.agent.config.PromptConfig
import ai.koog.agents.core.strategy.*
import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel
import ai.koog.prompt.message.AssistantMessage
import ai.koog.agents.core.agent.context.AIAgentContext

/** Edge types — sealed to enforce exhaustive when() in routing */
sealed class MainEdge {
    object UseTool    : MainEdge()
    object Done       : MainEdge()
    object Error      : MainEdge()
}

fun buildGraphStrategy(toolRegistry: ToolRegistry): AIAgentStrategy {
    return strategy("main") {
        // Entry node: send conversation to LLM
        val nodeThink = nodeLLMRequest("think")

        // Execution node: run whichever tool the LLM requested
        val nodeAct = nodeExecuteTool("act")

        // Error-handling node: log and attempt recovery
        val nodeHandleError = node<AssistantMessage, MainEdge>("handleError") { msg ->
            // log, maybe inject a corrective message
            println("[ERROR] tool call failed, attempting recovery…")
            MainEdge.Done   // or retry with MainEdge.UseTool
        }

        // Terminal node
        val nodeFinish = finish<AssistantMessage>("finish")

        // Wire edges:
        // LLM output → if tool call requested → act, else → finish
        nodeThink {
            onAssistantMessage { msg ->
                if (msg.toolCalls.isNotEmpty()) MainEdge.UseTool else MainEdge.Done
            }
        }

        // Route edges
        edge(nodeThink   to nodeAct)     { it == MainEdge.UseTool }
        edge(nodeThink   to nodeFinish)  { it == MainEdge.Done }
        edge(nodeAct     to nodeThink)   { true }  // loop: observe → think
        edge(nodeHandleError to nodeFinish) { true }

        setStart(nodeThink)
    }
}

fun buildGraphAgent(
    executor: PromptExecutor,
    model: LLMModel,
    systemPrompt: String,
    toolRegistry: ToolRegistry
): AIAgent {
    return AIAgent(
        executor = executor,
        agentConfig = AIAgentConfig(
            prompt = PromptConfig(systemPrompt = systemPrompt),
            tools = toolRegistry
        ),
        strategy = buildGraphStrategy(toolRegistry),
        llmModel = model
    )
}
```

---

## ReAct Pattern as Strategy Graph

ReAct (arXiv:2210.03629) interleaves **Thought**, **Action**, and **Observation** in the prompt. In Koog, the LLM naturally generates Thought + Action in a single `nodeLLMRequest`, and the tool result is the Observation fed back.

```kotlin
// agent/ReActAgent.kt
package com.example.agent.agent

import ai.koog.agents.core.strategy.*
import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel
import ai.koog.prompt.message.AssistantMessage

private const val REACT_SYSTEM_PROMPT = """
You are a ReAct agent. For every step, output:

Thought: <your reasoning about the current state>
Action: <tool name>
Action Input: <JSON args>

When you have the final answer, output:
Thought: I now have enough information.
Final Answer: <answer>
""".trimIndent()

sealed class ReActEdge {
    object Act    : ReActEdge()
    object Answer : ReActEdge()
}

fun buildReActStrategy(toolRegistry: ToolRegistry): AIAgentStrategy {
    return strategy("react") {
        val nodeReason = nodeLLMRequest("reason")
        val nodeExecute = nodeExecuteTool("execute")
        val nodeAnswer = finish<AssistantMessage>("answer")

        nodeReason {
            onAssistantMessage { msg ->
                val text = msg.content ?: ""
                if (text.contains("Final Answer:")) ReActEdge.Answer
                else if (msg.toolCalls.isNotEmpty()) ReActEdge.Act
                else ReActEdge.Answer  // fallback
            }
        }

        edge(nodeReason  to nodeExecute) { it == ReActEdge.Act }
        edge(nodeReason  to nodeAnswer)  { it == ReActEdge.Answer }
        edge(nodeExecute to nodeReason)  { true }

        setStart(nodeReason)
    }
}

fun buildReActAgent(
    executor: PromptExecutor,
    model: LLMModel,
    toolRegistry: ToolRegistry
): ai.koog.agents.core.agent.AIAgent {
    return ai.koog.agents.core.agent.AIAgent(
        executor = executor,
        agentConfig = ai.koog.agents.core.agent.config.AIAgentConfig(
            prompt = ai.koog.agents.core.agent.config.PromptConfig(
                systemPrompt = REACT_SYSTEM_PROMPT
            ),
            tools = toolRegistry
        ),
        strategy = buildReActStrategy(toolRegistry),
        llmModel = model
    )
}
```

---

## Reflexion Wrapper (arXiv:2303.11366)

Reflexion adds a verbal reinforcement loop: after the agent fails or produces a low-quality result, a **Reflector** LLM generates a textual critique that is prepended to the next attempt's context.

```kotlin
// agent/ReflexionAgent.kt
package com.example.agent.agent

import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.core.agent.config.AIAgentConfig
import ai.koog.agents.core.agent.config.PromptConfig
import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.update

data class ReflexionConfig(
    val maxTrials: Int = 3,
    val qualityThreshold: Double = 0.8
)

class ReflexionAgent(
    private val executor: PromptExecutor,
    private val actorModel: LLMModel,
    private val evaluatorModel: LLMModel,
    private val reflectorModel: LLMModel,
    private val toolRegistry: ToolRegistry,
    private val cfg: ReflexionConfig = ReflexionConfig()
) {
    // Episodic memory: accumulate reflections across trials
    private val reflectionMemory = MutableStateFlow<List<String>>(emptyList())

    suspend fun run(task: String): String {
        var result = ""
        for (trial in 1..cfg.maxTrials) {
            // Build system prompt with accumulated reflections
            val reflections = reflectionMemory.value
            val actorPrompt = buildActorPrompt(task, reflections)

            // Actor: attempt the task
            val actor = AIAgent(
                executor = executor,
                llmModel = actorModel,
                systemPrompt = actorPrompt,
                toolRegistry = toolRegistry
            )
            result = actor.run(task) ?: ""

            // Evaluator: score the result
            val score = evaluate(task, result)
            println("[Reflexion] Trial $trial score: $score")
            if (score >= cfg.qualityThreshold) break

            if (trial < cfg.maxTrials) {
                // Reflector: generate critique
                val reflection = reflect(task, result, score)
                reflectionMemory.update { it + reflection }
                println("[Reflexion] Reflection added: ${reflection.take(100)}")
            }
        }
        return result
    }

    private suspend fun evaluate(task: String, result: String): Double {
        val evaluator = AIAgent(
            executor = executor,
            llmModel = evaluatorModel,
            systemPrompt = """
                Score the following response to the task on a scale 0.0-1.0.
                Respond with ONLY the numeric score, e.g.: 0.75
            """.trimIndent()
        )
        val scoreText = evaluator.run("Task: $task\n\nResponse: $result") ?: "0"
        return scoreText.trim().toDoubleOrNull() ?: 0.0
    }

    private suspend fun reflect(task: String, result: String, score: Double): String {
        val reflector = AIAgent(
            executor = executor,
            llmModel = reflectorModel,
            systemPrompt = """
                You are a self-reflection system. Analyze why the agent's response was suboptimal
                and produce a concise reflection (2-3 sentences) that will help it do better next time.
                Focus on what was missing, incorrect, or could be improved.
            """.trimIndent()
        )
        return reflector.run(
            "Task: $task\nResponse (score=$score): $result"
        ) ?: "The response was incomplete. Be more thorough and precise."
    }

    private fun buildActorPrompt(task: String, reflections: List<String>): String {
        val reflectionSection = if (reflections.isEmpty()) "" else """

Previous attempt reflections (learn from these):
${reflections.mapIndexed { i, r -> "${i + 1}. $r" }.joinToString("\n")}

"""
        return "You are a capable assistant.$reflectionSection"
    }
}
```

---

## SELF-REFINE as a Node Chain (selfrefine.info)

```kotlin
// In strategy graph: generate → verify → refine loop
sealed class RefineEdge {
    object Refine   : RefineEdge()
    object Accept   : RefineEdge()
}

fun buildSelfRefineStrategy(maxIterations: Int = 3): AIAgentStrategy {
    return strategy("selfRefine") {
        var iterations = 0

        // Generate initial response
        val nodeGenerate = nodeLLMRequest("generate")

        // LLM-as-judge: critique the output
        val nodeVerify = nodeLLMRequest("verify")

        // Incorporate critique and regenerate
        val nodeRefine = nodeLLMRequest("refine")

        val nodeDone = finish<ai.koog.prompt.message.AssistantMessage>("done")

        nodeVerify {
            onAssistantMessage { msg ->
                iterations++
                val verdict = msg.content ?: ""
                when {
                    verdict.contains("ACCEPT", ignoreCase = true) -> RefineEdge.Accept
                    verdict.contains("REFINE", ignoreCase = true) && iterations < maxIterations -> RefineEdge.Refine
                    else -> RefineEdge.Accept  // max iterations reached
                }
            }
        }

        edge(nodeGenerate to nodeVerify) { true }
        edge(nodeVerify   to nodeRefine) { it == RefineEdge.Refine }
        edge(nodeVerify   to nodeDone)   { it == RefineEdge.Accept }
        edge(nodeRefine   to nodeVerify) { true }

        setStart(nodeGenerate)
    }
}
```

---

## Parallel Tool Execution

```kotlin
// Execute multiple tools concurrently in a single node
val nodeParallelTools = nodeExecuteMultipleTools(
    name = "parallelSearch",
    parallelTools = true    // default is false (sequential)
)
```

When `parallelTools = true`, Koog launches all tool calls from a single LLM message concurrently using coroutines and waits for all results before feeding observations back to the LLM.

---

## Error Handling with Kotlin Result

```kotlin
// agent/SafeAgent.kt
package com.example.agent.agent

import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel
import kotlinx.coroutines.CancellationException
import kotlinx.coroutines.withTimeout
import io.github.oshai.kotlinlogging.KotlinLogging

private val log = KotlinLogging.logger {}

class SafeAgent(
    private val executor: PromptExecutor,
    private val model: LLMModel,
    private val systemPrompt: String,
    private val timeoutMs: Long = 60_000L
) {
    suspend fun run(input: String): Result<String> = runCatching {
        withTimeout(timeoutMs) {
            val agent = AIAgent(
                executor = executor,
                llmModel = model,
                systemPrompt = systemPrompt
            )
            agent.run(input) ?: throw IllegalStateException("Agent returned null response")
        }
    }.onFailure { e ->
        when (e) {
            is CancellationException -> log.warn { "Agent cancelled for input: ${input.take(50)}" }
            is kotlinx.coroutines.TimeoutCancellationException ->
                log.error { "Agent timed out after ${timeoutMs}ms" }
            else -> log.error(e) { "Agent failed: ${e.message}" }
        }
    }
}

// Usage
suspend fun main() {
    val agent = SafeAgent(executor, model, "You are helpful.")
    agent.run("What is 2+2?").fold(
        onSuccess = { println("Result: $it") },
        onFailure = { println("Error: ${it.message}") }
    )
}
```

---

## Streaming with Flow<String>

```kotlin
// Ktor SSE endpoint streaming agent tokens
import io.ktor.server.application.*
import io.ktor.server.response.*
import io.ktor.server.routing.*
import io.ktor.server.sse.*
import kotlinx.coroutines.flow.collect

fun Route.streamingRoute(executor: PromptExecutor, model: LLMModel) {
    install(SSE)

    sse("/agent/stream") {
        val userMessage = call.request.queryParameters["q"] ?: "Hello"
        val agent = AIAgent(
            executor = executor,
            llmModel = model,
            systemPrompt = "You are a helpful assistant."
        )
        agent.runStreaming(userMessage).collect { token ->
            send(ServerSentEvent(data = token))
        }
    }
}

// Or collect into a channel for custom processing
import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.launch

suspend fun streamWithChannel(agent: AIAgent, input: String): Channel<String> {
    val channel = Channel<String>(Channel.UNLIMITED)
    kotlinx.coroutines.coroutineScope {
        launch {
            agent.runStreaming(input).collect { token ->
                channel.send(token)
            }
            channel.close()
        }
    }
    return channel
}
```

---

## Token Tracking and Compaction Trigger

```kotlin
// Track token usage and trigger context compaction when needed
data class TokenBudget(
    val maxContextTokens: Int = 128_000,
    val compactionThreshold: Double = 0.80   // compact at 80% full
)

class TokenAwareAgent(
    private val executor: PromptExecutor,
    private val model: LLMModel,
    private val budget: TokenBudget = TokenBudget(),
    private val systemPrompt: String
) {
    private var estimatedTokensUsed = 0

    // Rough heuristic: 1 token ≈ 4 characters
    private fun estimateTokens(text: String) = text.length / 4

    suspend fun runWithCompaction(
        messages: MutableList<String>,
        newMessage: String
    ): String {
        estimatedTokensUsed += estimateTokens(newMessage)

        if (estimatedTokensUsed.toDouble() / budget.maxContextTokens > budget.compactionThreshold) {
            val summary = summarizeHistory(messages)
            messages.clear()
            messages.add(summary)
            estimatedTokensUsed = estimateTokens(summary)
        }

        messages.add(newMessage)
        val agent = AIAgent(
            executor = executor,
            llmModel = model,
            systemPrompt = systemPrompt
        )
        val result = agent.run(messages.joinToString("\n")) ?: ""
        estimatedTokensUsed += estimateTokens(result)
        messages.add(result)
        return result
    }

    private suspend fun summarizeHistory(messages: List<String>): String {
        val summarizer = AIAgent(
            executor = executor,
            llmModel = model,
            systemPrompt = """
                Summarize the following conversation history into a compact context summary.
                Preserve key facts, decisions, and open questions.
                Output: [SUMMARY] <concise summary> [/SUMMARY]
            """.trimIndent()
        )
        return summarizer.run(messages.joinToString("\n")) ?: messages.last()
    }
}
```

---

## Complete Example: Code Review Agent

```kotlin
// agent/CodeReviewAgent.kt
package com.example.agent.agent

import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.core.agent.config.AIAgentConfig
import ai.koog.agents.core.agent.config.PromptConfig
import ai.koog.agents.core.strategy.*
import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.agents.core.tools.annotations.Tool
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel
import ai.koog.prompt.message.AssistantMessage
import kotlinx.serialization.Serializable

// --- Tools ---

@Serializable
data class ReadFileArgs(val path: String)

@Tool(name = "readFile", description = "Read file contents from the repository")
suspend fun readFile(args: ReadFileArgs): String {
    return java.io.File(args.path).readText()
}

@Serializable
data class SearchCodeArgs(val pattern: String, val directory: String = ".")

@Tool(name = "searchCode", description = "Search for code patterns using grep")
suspend fun searchCode(args: SearchCodeArgs): String {
    val result = Runtime.getRuntime()
        .exec(arrayOf("grep", "-r", args.pattern, args.directory))
        .inputStream.bufferedReader().readText()
    return result.ifEmpty { "No matches found." }
}

@Serializable
data class RunTestsArgs(val module: String = ".")

@Tool(name = "runTests", description = "Run tests for a module and return results")
suspend fun runTests(args: RunTestsArgs): String {
    val proc = ProcessBuilder("./gradlew", "test", "-p", args.module)
        .redirectErrorStream(true)
        .start()
    val output = proc.inputStream.bufferedReader().readText()
    val exitCode = proc.waitFor()
    return "Exit code: $exitCode\n$output"
}

// --- Strategy ---

sealed class ReviewEdge {
    object Investigate : ReviewEdge()
    object WriteReview : ReviewEdge()
    object Done        : ReviewEdge()
}

fun buildCodeReviewStrategy(toolRegistry: ToolRegistry): AIAgentStrategy {
    return strategy("codeReview") {
        // Phase 1: Investigate — read files, run tests
        val nodeInvestigate = nodeLLMRequest("investigate")
        val nodeRunTools = nodeExecuteMultipleTools("runTools", parallelTools = true)

        // Phase 2: Write — produce structured review
        val nodeWriteReview = nodeLLMRequest("writeReview")

        // Terminal
        val nodeDone = finish<AssistantMessage>("done")

        nodeInvestigate {
            onAssistantMessage { msg ->
                when {
                    msg.toolCalls.isNotEmpty()                          -> ReviewEdge.Investigate
                    msg.content?.contains("REVIEW:")    == true         -> ReviewEdge.Done
                    msg.content?.contains("WRITE REVIEW") == true       -> ReviewEdge.WriteReview
                    else                                                 -> ReviewEdge.Done
                }
            }
        }

        edge(nodeInvestigate to nodeRunTools)   { it == ReviewEdge.Investigate }
        edge(nodeInvestigate to nodeWriteReview){ it == ReviewEdge.WriteReview }
        edge(nodeInvestigate to nodeDone)       { it == ReviewEdge.Done }
        edge(nodeRunTools    to nodeInvestigate){ true }
        edge(nodeWriteReview to nodeDone)       { true }

        setStart(nodeInvestigate)
    }
}

// --- Agent ---

private const val CODE_REVIEW_PROMPT = """
You are an expert Kotlin code reviewer. When given a pull request or file to review:

1. Read the relevant files using readFile
2. Search for related usage patterns using searchCode
3. Run the test suite with runTests
4. When you have gathered enough information, say "WRITE REVIEW" to produce your review

Structure your final review as:
REVIEW:
## Summary
## Issues Found (severity: Critical/Major/Minor)
## Suggestions
## Verdict (Approve/Request Changes/Reject)
""".trimIndent()

fun buildCodeReviewAgent(
    executor: PromptExecutor,
    model: LLMModel,
    toolRegistry: ToolRegistry
): AIAgent {
    return AIAgent(
        executor = executor,
        agentConfig = AIAgentConfig(
            prompt = PromptConfig(systemPrompt = CODE_REVIEW_PROMPT),
            tools = toolRegistry
        ),
        strategy = buildCodeReviewStrategy(toolRegistry),
        llmModel = model
    )
}

// Usage:
suspend fun reviewPullRequest(
    executor: PromptExecutor,
    model: LLMModel,
    changedFiles: List<String>
): String {
    val toolRegistry = ToolRegistry {
        tool(::readFile)
        tool(::searchCode)
        tool(::runTests)
    }
    val agent = buildCodeReviewAgent(executor, model, toolRegistry)
    return agent.run(
        "Please review these changed files: ${changedFiles.joinToString(", ")}"
    ) ?: "Review failed."
}
```

---

## Abort and Cancellation

Koog agents respect Kotlin structured concurrency. Cancel the job to abort:

```kotlin
import kotlinx.coroutines.*

val job = scope.launch {
    val result = agent.run(userInput)
    // …
}

// From another coroutine or HTTP handler:
job.cancel()              // cooperative cancellation
job.cancelAndJoin()       // wait for cleanup
```

For user-initiated cancellation in a REST endpoint:

```kotlin
post("/agent/cancel/{id}") {
    val id = call.parameters["id"]!!
    activeJobs[id]?.cancel()
    call.respond(HttpStatusCode.OK, mapOf("cancelled" to id))
}
```

The agent's tool calls also respect cancellation — any blocking `java.*` calls inside tools should use `withContext(Dispatchers.IO)` and check `isActive` where possible.
