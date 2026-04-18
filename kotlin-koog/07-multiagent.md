# Multi-Agent Patterns with Koog A2A

## Koog A2A Protocol Overview

The Agent-to-Agent (A2A) protocol standardizes how agents discover capabilities and exchange tasks. It has four core concepts:

| Concept | Role |
|---------|------|
| `AgentCard` | JSON capability manifest (name, skills, URL, auth) |
| `AgentExecutor` | Server-side handler that receives and processes tasks |
| `A2AServer` | Ktor/Netty HTTP server hosting the executor |
| `nodeA2AClientSendMessage` | Strategy graph node that sends tasks to remote agents |

---

## Full A2A Server Setup

```kotlin
// multiagent/A2AServerSetup.kt
package com.example.agent.multiagent

import ai.koog.agents.a2a.protocol.*
import ai.koog.agents.a2a.server.A2AServer
import ai.koog.agents.a2a.server.transport.HttpJSONRPCServerTransport
import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel

/** Build an AgentCard describing this agent's capabilities */
fun buildResearcherCard(baseUrl: String): AgentCard = AgentCard(
    name = "researcher",
    description = "Searches the web and retrieves relevant documents for a given query.",
    url = "$baseUrl/a2a/researcher",
    version = "1.0",
    capabilities = AgentCapabilities(
        streaming = true,
        pushNotifications = false
    ),
    skills = listOf(
        AgentSkill(
            id = "web-research",
            name = "Web Research",
            description = "Searches the web for information on a topic",
            tags = listOf("research", "web", "retrieval"),
            examples = listOf("Research recent developments in Kotlin coroutines")
        )
    ),
    authentication = AgentAuthentication(
        schemes = listOf("Bearer")
    )
)

/** AgentExecutor: handles incoming tasks from other agents */
class ResearcherAgentExecutor(
    private val executor: PromptExecutor,
    private val model: LLMModel,
    private val toolRegistry: ToolRegistry
) : AgentExecutor {

    override suspend fun execute(task: A2ATask): A2ATaskResult {
        val userMessage = task.message.parts
            .filterIsInstance<A2AMessagePart.Text>()
            .joinToString("\n") { it.text }

        val agent = AIAgent(
            executor = executor,
            llmModel = model,
            systemPrompt = """
                You are a research specialist. Given a research question:
                1. Search the web for relevant information
                2. Retrieve and summarize the most relevant content
                3. Return a concise, factual summary with sources
            """.trimIndent(),
            toolRegistry = toolRegistry
        )

        val result = runCatching { agent.run(userMessage) ?: "No result" }
            .getOrElse { "Research failed: ${it.message}" }

        return A2ATaskResult(
            id = task.id,
            status = A2ATaskStatus(
                state = TaskState.Completed,
                message = A2AMessage(
                    role = "agent",
                    parts = listOf(A2AMessagePart.Text(result))
                )
            )
        )
    }

    override suspend fun cancel(taskId: String) {
        // Implement cancellation if needed
    }
}

/** Start the A2A server */
fun startA2AServer(
    executor: AgentExecutor,
    card: AgentCard,
    port: Int = 8081
): A2AServer {
    val transport = HttpJSONRPCServerTransport(port = port)
    val server = A2AServer(
        agentExecutor = executor,
        agentCard = card,
        transport = transport
    )
    server.start()
    return server
}

// Usage:
/*
val card = buildResearcherCard("http://localhost:8081")
val researchExecutor = ResearcherAgentExecutor(executor, model, toolRegistry)
val server = startA2AServer(researchExecutor, card, port = 8081)
*/
```

---

## A2A Client Node in Strategy Graph

```kotlin
// multiagent/A2AClientStrategy.kt
package com.example.agent.multiagent

import ai.koog.agents.a2a.client.nodeA2AClientSendMessage
import ai.koog.agents.core.strategy.*
import ai.koog.prompt.message.AssistantMessage

sealed class CoordinatorEdge {
    object Research    : CoordinatorEdge()
    object Implement   : CoordinatorEdge()
    object Review      : CoordinatorEdge()
    object Done        : CoordinatorEdge()
}

fun buildCoordinatorStrategy(
    researcherUrl: String,
    implementerUrl: String,
    reviewerUrl: String
): AIAgentStrategy {
    return strategy("coordinator") {
        // Plan the work
        val nodePlan = nodeLLMRequest("plan")

        // Delegate to specialist agents
        val nodeResearch   = nodeA2AClientSendMessage("research",   agentUrl = researcherUrl)
        val nodeImplement  = nodeA2AClientSendMessage("implement",  agentUrl = implementerUrl)
        val nodeReview     = nodeA2AClientSendMessage("review",     agentUrl = reviewerUrl)

        val nodeDone = finish<AssistantMessage>("done")

        nodePlan {
            onAssistantMessage { msg ->
                val content = msg.content ?: ""
                when {
                    content.contains("RESEARCH:")   -> CoordinatorEdge.Research
                    content.contains("IMPLEMENT:")  -> CoordinatorEdge.Implement
                    content.contains("REVIEW:")     -> CoordinatorEdge.Review
                    content.contains("DONE:")       -> CoordinatorEdge.Done
                    else                            -> CoordinatorEdge.Done
                }
            }
        }

        edge(nodePlan      to nodeResearch)  { it == CoordinatorEdge.Research }
        edge(nodePlan      to nodeImplement) { it == CoordinatorEdge.Implement }
        edge(nodePlan      to nodeReview)    { it == CoordinatorEdge.Review }
        edge(nodePlan      to nodeDone)      { it == CoordinatorEdge.Done }
        edge(nodeResearch  to nodePlan)      { true }   // report back to coordinator
        edge(nodeImplement to nodePlan)      { true }
        edge(nodeReview    to nodePlan)      { true }

        setStart(nodePlan)
    }
}
```

---

## Direct Agent Composition (Subgraph)

```kotlin
// multiagent/SubgraphComposition.kt
package com.example.agent.multiagent

import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.core.strategy.*
import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel
import ai.koog.prompt.message.AssistantMessage

/** Embed one agent as a callable sub-strategy within another */
fun buildPipelineStrategy(
    researchExecutor: PromptExecutor,
    researchModel: LLMModel,
    writeExecutor: PromptExecutor,
    writeModel: LLMModel,
    tools: ToolRegistry
): AIAgentStrategy {
    return strategy("pipeline") {
        // Phase 1: research sub-agent
        val nodeResearch = subgraphWithAgent<String, String>("research") { input ->
            val researchAgent = AIAgent(
                executor = researchExecutor,
                llmModel = researchModel,
                systemPrompt = "You research topics thoroughly.",
                toolRegistry = tools
            )
            researchAgent.run(input) ?: "No research result"
        }

        // Phase 2: writing sub-agent consumes research output
        val nodeWrite = subgraphWithAgent<String, String>("write") { researchResult ->
            val writerAgent = AIAgent(
                executor = writeExecutor,
                llmModel = writeModel,
                systemPrompt = "You write clear, structured reports based on research."
            )
            writerAgent.run("Based on this research, write a report:\n$researchResult") ?: ""
        }

        val nodeDone = finish<AssistantMessage>("done")

        edge(nodeResearch to nodeWrite) { true }
        edge(nodeWrite    to nodeDone)  { true }

        setStart(nodeResearch)
    }
}
```

---

## SupervisorAgent with Handoff Tools

```kotlin
// multiagent/SupervisorAgent.kt
package com.example.agent.multiagent

import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.agents.core.tools.annotations.Tool
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel
import kotlinx.serialization.Serializable

data class SpecialistAgent(
    val name: String,
    val description: String,
    val agent: AIAgent
)

class SupervisorAgent(
    private val executor: PromptExecutor,
    private val model: LLMModel,
    private val specialists: List<SpecialistAgent>
) {
    @Serializable
    data class HandoffArgs(val agentName: String, val task: String)

    // Build a ToolRegistry with one handoff tool per specialist
    private val handoffRegistry: ToolRegistry by lazy {
        ToolRegistry {
            specialists.forEach { specialist ->
                // Dynamically register handoff tool
                tool(
                    name = "handoffTo_${specialist.name}",
                    description = "Hand off task to ${specialist.name}: ${specialist.description}",
                    execute = { task: String -> specialist.agent.run(task) ?: "No response" }
                )
            }
        }
    }

    private val SUPERVISOR_PROMPT = buildString {
        appendLine("You are a supervisor agent. Available specialists:")
        specialists.forEach { s ->
            appendLine("- ${s.name}: ${s.description}")
        }
        appendLine()
        appendLine("Delegate tasks to appropriate specialists using handoff tools.")
        appendLine("After all specialists complete their work, synthesize the results.")
    }

    suspend fun run(task: String): String {
        val supervisor = AIAgent(
            executor = executor,
            llmModel = model,
            systemPrompt = SUPERVISOR_PROMPT,
            toolRegistry = handoffRegistry
        )
        return supervisor.run(task) ?: "Supervisor failed."
    }
}
```

---

## AutoGen-Style Typed Message Passing via Kotlin Channel

```kotlin
// multiagent/AutoGenActors.kt
package com.example.agent.multiagent

import kotlinx.coroutines.*
import kotlinx.coroutines.channels.Channel

sealed class AgentMessage {
    data class UserTask(val task: String, val fromAgent: String = "user") : AgentMessage()
    data class AgentResponse(val result: String, val fromAgent: String) : AgentMessage()
    data class ToolResult(val toolName: String, val result: String) : AgentMessage()
    data class Terminate(val reason: String = "complete") : AgentMessage()
    data class Error(val message: String, val fromAgent: String) : AgentMessage()
}

abstract class AutoGenActor(val name: String) {
    protected val inbox = Channel<AgentMessage>(Channel.UNLIMITED)
    protected val outbox = Channel<AgentMessage>(Channel.UNLIMITED)

    suspend fun send(msg: AgentMessage) = inbox.send(msg)
    fun subscribe(): Channel<AgentMessage> = outbox

    abstract suspend fun run(): Unit

    protected suspend fun reply(msg: AgentMessage) = outbox.send(msg)
}

class ResearchActor(
    private val executor: ai.koog.prompt.executor.llms.PromptExecutor,
    private val model: ai.koog.prompt.llm.LLMModel
) : AutoGenActor("researcher") {
    override suspend fun run() {
        for (msg in inbox) {
            when (msg) {
                is AgentMessage.UserTask -> {
                    val agent = ai.koog.agents.core.agent.AIAgent(
                        executor = executor,
                        llmModel = model,
                        systemPrompt = "You are a research specialist."
                    )
                    val result = runCatching { agent.run(msg.task) ?: "No result" }
                        .getOrElse { "Research error: ${it.message}" }
                    reply(AgentMessage.AgentResponse(result, fromAgent = name))
                }
                is AgentMessage.Terminate -> {
                    outbox.close()
                    break
                }
                else -> Unit
            }
        }
    }
}

/** Orchestrator: routes messages between actors */
class ActorOrchestrator(
    private val actors: Map<String, AutoGenActor>,
    private val scope: CoroutineScope
) {
    fun start() {
        actors.values.forEach { actor ->
            scope.launch { actor.run() }
        }
    }

    suspend fun runConversation(task: String): String {
        val researcher = actors["researcher"] ?: error("No researcher actor")
        researcher.send(AgentMessage.UserTask(task))
        val response = researcher.subscribe().receive()
        researcher.send(AgentMessage.Terminate())
        return when (response) {
            is AgentMessage.AgentResponse -> response.result
            is AgentMessage.Error         -> "Error: ${response.message}"
            else                          -> "Unexpected response"
        }
    }
}
```

---

## MetaGPT SOP Pipeline

```kotlin
// multiagent/MetaGptSop.kt
package com.example.agent.multiagent

import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.update

/** SOP artifact state — each role reads from previous, writes to next */
data class SopArtifacts(
    val requirements: String = "",
    val systemDesign: String = "",
    val implementationPlan: String = "",
    val code: String = "",
    val testReport: String = ""
)

abstract class SopRole(val roleName: String) {
    abstract suspend fun execute(artifacts: SopArtifacts): SopArtifacts
}

class ProductManagerRole(executor: PromptExecutor, model: LLMModel) : SopRole("ProductManager") {
    private val agent = AIAgent(executor, llmModel = model,
        systemPrompt = "You are a product manager. Define clear, actionable requirements.")
    override suspend fun execute(artifacts: SopArtifacts): SopArtifacts {
        val req = agent.run("Define requirements for: ${artifacts.requirements}") ?: ""
        return artifacts.copy(requirements = req)
    }
}

class ArchitectRole(executor: PromptExecutor, model: LLMModel) : SopRole("Architect") {
    private val agent = AIAgent(executor, llmModel = model,
        systemPrompt = "You are a software architect. Design clean, scalable systems.")
    override suspend fun execute(artifacts: SopArtifacts): SopArtifacts {
        val design = agent.run("Design a system for:\n${artifacts.requirements}") ?: ""
        return artifacts.copy(systemDesign = design)
    }
}

class EngineerRole(executor: PromptExecutor, model: LLMModel) : SopRole("Engineer") {
    private val agent = AIAgent(executor, llmModel = model,
        systemPrompt = "You are a senior Kotlin engineer. Write production-grade code.")
    override suspend fun execute(artifacts: SopArtifacts): SopArtifacts {
        val code = agent.run(
            "Implement:\nRequirements: ${artifacts.requirements}\nDesign: ${artifacts.systemDesign}"
        ) ?: ""
        return artifacts.copy(code = code)
    }
}

class QaEngineerRole(executor: PromptExecutor, model: LLMModel) : SopRole("QAEngineer") {
    private val agent = AIAgent(executor, llmModel = model,
        systemPrompt = "You are a QA engineer. Write thorough test cases.")
    override suspend fun execute(artifacts: SopArtifacts): SopArtifacts {
        val tests = agent.run("Write tests for:\n${artifacts.code}") ?: ""
        return artifacts.copy(testReport = tests)
    }
}

class MetaGptPipeline(private val roles: List<SopRole>) {
    private val _artifacts = MutableStateFlow(SopArtifacts())
    val artifacts: StateFlow<SopArtifacts> = _artifacts

    suspend fun run(initialTask: String): SopArtifacts {
        _artifacts.value = SopArtifacts(requirements = initialTask)
        roles.forEach { role ->
            println("[MetaGPT SOP] Running role: ${role.roleName}")
            val result = role.execute(_artifacts.value)
            _artifacts.update { result }
        }
        return _artifacts.value
    }
}

// Assembly
fun buildMetaGptPipeline(executor: PromptExecutor, model: LLMModel): MetaGptPipeline =
    MetaGptPipeline(
        listOf(
            ProductManagerRole(executor, model),
            ArchitectRole(executor, model),
            EngineerRole(executor, model),
            QaEngineerRole(executor, model)
        )
    )
```

---

## Parallel Agent Execution

```kotlin
// multiagent/ParallelAgents.kt
package com.example.agent.multiagent

import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel
import kotlinx.coroutines.async
import kotlinx.coroutines.awaitAll
import kotlinx.coroutines.coroutineScope

data class AgentTask(val agentName: String, val systemPrompt: String, val input: String)
data class AgentResult(val agentName: String, val output: String, val error: String? = null)

/** Run multiple agents in parallel; collect all results */
suspend fun runAgentsInParallel(
    tasks: List<AgentTask>,
    executor: PromptExecutor,
    model: LLMModel
): List<AgentResult> = coroutineScope {
    tasks.map { task ->
        async {
            runCatching {
                val agent = AIAgent(
                    executor = executor,
                    llmModel = model,
                    systemPrompt = task.systemPrompt
                )
                AgentResult(task.agentName, agent.run(task.input) ?: "")
            }.getOrElse { e ->
                AgentResult(task.agentName, "", error = e.message)
            }
        }
    }.awaitAll()
}

/** Team registry: named agents with their capabilities */
class TeamRegistry(
    private val executor: PromptExecutor,
    private val model: LLMModel
) {
    private val agents = mutableMapOf<String, Pair<String, LLMModel?>>() // name → (systemPrompt, model)

    fun register(name: String, systemPrompt: String, model: LLMModel? = null) {
        agents[name] = systemPrompt to model
    }

    suspend fun dispatch(agentName: String, task: String): String {
        val (prompt, agentModel) = agents[agentName]
            ?: throw IllegalArgumentException("Unknown agent: $agentName")
        val agent = AIAgent(
            executor = executor,
            llmModel = agentModel ?: model,
            systemPrompt = prompt
        )
        return agent.run(task) ?: ""
    }

    suspend fun dispatchAll(task: String): Map<String, String> = coroutineScope {
        agents.entries.map { (name, config) ->
            async {
                name to runCatching {
                    AIAgent(executor, llmModel = config.second ?: model, systemPrompt = config.first)
                        .run(task) ?: ""
                }.getOrElse { "Error: ${it.message}" }
            }
        }.awaitAll().toMap()
    }
}
```

---

## Worker Lifecycle State Machine

```kotlin
// multiagent/WorkerLifecycle.kt
package com.example.agent.multiagent

import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow

enum class WorkerState {
    Spawning,
    TrustRequired,      // waiting for permission grant
    ReadyForPrompt,
    Processing,
    WaitingForTool,
    Finished,
    Failed,
    Cancelled
}

data class WorkerStatus(
    val workerId: String,
    val state: WorkerState,
    val currentTask: String = "",
    val progressPercent: Int = 0,
    val errorMessage: String = ""
)

class WorkerLifecycleManager {
    private val _workers = MutableStateFlow<Map<String, WorkerStatus>>(emptyMap())
    val workers: StateFlow<Map<String, WorkerStatus>> = _workers

    fun spawn(workerId: String, task: String): WorkerStatus {
        val status = WorkerStatus(workerId, WorkerState.Spawning, task)
        update(status)
        return status
    }

    fun transition(workerId: String, newState: WorkerState, message: String = "") {
        val current = _workers.value[workerId] ?: return
        val allowed = allowedTransitions[current.state] ?: emptySet()
        require(newState in allowed) {
            "Invalid transition: ${current.state} → $newState for worker $workerId"
        }
        update(current.copy(state = newState, errorMessage = message))
    }

    private fun update(status: WorkerStatus) {
        _workers.value = _workers.value + (status.workerId to status)
    }

    private val allowedTransitions = mapOf(
        WorkerState.Spawning         to setOf(WorkerState.TrustRequired, WorkerState.ReadyForPrompt, WorkerState.Failed),
        WorkerState.TrustRequired    to setOf(WorkerState.ReadyForPrompt, WorkerState.Cancelled),
        WorkerState.ReadyForPrompt   to setOf(WorkerState.Processing, WorkerState.Cancelled),
        WorkerState.Processing       to setOf(WorkerState.WaitingForTool, WorkerState.Finished, WorkerState.Failed),
        WorkerState.WaitingForTool   to setOf(WorkerState.Processing, WorkerState.Failed, WorkerState.Cancelled),
        WorkerState.Finished         to emptySet(),
        WorkerState.Failed           to emptySet(),
        WorkerState.Cancelled        to emptySet()
    )
}
```

---

## CAMEL Role-Playing with Inception Prompting

```kotlin
// multiagent/CamelRolePlaying.kt
package com.example.agent.multiagent

import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel

data class RolePlayConfig(
    val task: String,
    val aiUserRole: String = "User",
    val aiAssistantRole: String = "Assistant",
    val maxRounds: Int = 8
)

class CamelRolePlaying(
    private val executor: PromptExecutor,
    private val model: LLMModel
) {
    suspend fun run(config: RolePlayConfig): List<Pair<String, String>> {
        val conversation = mutableListOf<Pair<String, String>>()

        // Inception prompt: instruct each agent to play its role
        val userSystemPrompt = """
            You are playing the role of a ${config.aiUserRole}.
            Your goal is to accomplish: ${config.task}
            You are collaborating with a ${config.aiAssistantRole}.
            Ask specific questions and give concrete instructions to drive the task forward.
            When the task is complete, say "TASK COMPLETE."
        """.trimIndent()

        val assistantSystemPrompt = """
            You are playing the role of a ${config.aiAssistantRole}.
            Your counterpart is a ${config.aiUserRole} working on: ${config.task}
            Respond to instructions with specific, actionable solutions.
            When you've completed the task, say "TASK COMPLETE."
        """.trimIndent()

        val userAgent = AIAgent(executor, llmModel = model, systemPrompt = userSystemPrompt)
        val assistantAgent = AIAgent(executor, llmModel = model, systemPrompt = assistantSystemPrompt)

        var currentMessage = "Let's work on the task: ${config.task}"
        var isUser = true

        for (round in 1..config.maxRounds) {
            val speaker = if (isUser) config.aiUserRole else config.aiAssistantRole
            val agent = if (isUser) userAgent else assistantAgent

            val response = runCatching { agent.run(currentMessage) ?: "..." }.getOrElse { "Error: ${it.message}" }
            conversation.add(speaker to response)

            if (response.contains("TASK COMPLETE", ignoreCase = true)) break

            currentMessage = response
            isUser = !isUser
        }

        return conversation
    }
}
```

---

## Scheduled/Recurring Agent Tasks

```kotlin
// multiagent/CronAgent.kt
package com.example.agent.multiagent

import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel
import kotlinx.coroutines.*

class CronAgent(
    private val executor: PromptExecutor,
    private val model: LLMModel,
    private val task: String,
    private val systemPrompt: String,
    private val intervalMs: Long = 60_000L     // default: every minute
) {
    private var job: Job? = null

    fun start(scope: CoroutineScope) {
        job = scope.launch {
            while (isActive) {
                runCatching {
                    val agent = AIAgent(executor, llmModel = model, systemPrompt = systemPrompt)
                    val result = agent.run(task)
                    println("[CronAgent] Result at ${System.currentTimeMillis()}: ${result?.take(100)}")
                }.onFailure { e ->
                    println("[CronAgent] Error: ${e.message}")
                }
                delay(intervalMs)
            }
        }
    }

    fun stop() { job?.cancel() }
}
```
