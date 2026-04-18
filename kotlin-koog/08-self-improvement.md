# Self-Improvement Mechanisms

## Overview

Self-improving agents learn from experience without human intervention:

| Mechanism | Paper | Koog Implementation |
|-----------|-------|---------------------|
| Reflexion | arXiv:2303.11366 | 3-agent strategy: Actor + Evaluator + Reflector |
| ExpeL | arXiv:2308.10144 | Experience pool → Qdrant → insight extraction |
| CRITIC | — | Tool-verified self-correction in strategy graph |
| SELF-REFINE | selfrefine.info | Iterative generate → verify → refine |
| Voyager skill library | — | Qdrant-indexed code/actions retrieval |

---

## Reflexion: 3-Agent Strategy (arXiv:2303.11366)

```kotlin
// selfimprove/ReflexionSystem.kt
package com.example.agent.selfimprove

import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.serialization.Serializable
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json
import java.io.File

@Serializable
data class ReflexionMemory(
    val task: String,
    val attempt: String,
    val score: Double,
    val reflection: String,
    val trial: Int,
    val timestamp: Long = System.currentTimeMillis()
)

class ReflexionSystem(
    private val actorExecutor: PromptExecutor,
    private val actorModel: LLMModel,
    private val evaluatorExecutor: PromptExecutor,
    private val evaluatorModel: LLMModel,
    private val reflectorExecutor: PromptExecutor,
    private val reflectorModel: LLMModel,
    private val toolRegistry: ToolRegistry? = null,
    private val maxTrials: Int = 4,
    private val acceptThreshold: Double = 0.85,
    private val memoryPath: String = "./data/reflexion-memory.jsonl"
) {
    // Episodic memory: verbal feedback from previous trials
    private val episodicBuffer = MutableStateFlow<List<String>>(emptyList())
    private val json = Json { encodeDefaults = true }

    suspend fun run(task: String): Pair<String, List<ReflexionMemory>> {
        val history = mutableListOf<ReflexionMemory>()
        var bestResult = ""
        var bestScore = 0.0

        // Load prior reflections from persistent episodic memory
        loadPriorReflections(task)

        for (trial in 1..maxTrials) {
            println("[Reflexion] Trial $trial/$maxTrials")

            // ACTOR: attempt the task with accumulated reflections
            val result = runActor(task)
            println("[Reflexion] Actor output: ${result.take(150)}")

            // EVALUATOR: score the attempt
            val (score, feedback) = runEvaluator(task, result)
            println("[Reflexion] Evaluator score: $score, feedback: ${feedback.take(100)}")

            if (score > bestScore) {
                bestScore = score
                bestResult = result
            }

            // REFLECTOR: generate verbal RL signal
            val reflection = if (score < acceptThreshold && trial < maxTrials) {
                runReflector(task, result, score, feedback).also {
                    episodicBuffer.update { buf -> buf + it }
                    println("[Reflexion] Reflection: ${it.take(150)}")
                }
            } else ""

            val mem = ReflexionMemory(task, result, score, reflection, trial)
            history.add(mem)
            persistMemory(mem)

            if (score >= acceptThreshold) {
                println("[Reflexion] Accepted at trial $trial with score $score")
                break
            }
        }

        return bestResult to history
    }

    private suspend fun runActor(task: String): String {
        val reflections = episodicBuffer.value
        val actorPrompt = buildString {
            appendLine("You are a capable agent solving tasks.")
            if (reflections.isNotEmpty()) {
                appendLine()
                appendLine("=== SELF-REFLECTIONS FROM PREVIOUS ATTEMPTS ===")
                reflections.forEachIndexed { i, r -> appendLine("${i + 1}. $r") }
                appendLine("=== END REFLECTIONS ===")
                appendLine("Apply these lessons to avoid past mistakes.")
            }
        }
        val actor = AIAgent(
            executor = actorExecutor,
            llmModel = actorModel,
            systemPrompt = actorPrompt,
            toolRegistry = toolRegistry
        )
        return actor.run(task) ?: ""
    }

    private suspend fun runEvaluator(task: String, attempt: String): Pair<Double, String> {
        val evaluator = AIAgent(
            executor = evaluatorExecutor,
            llmModel = evaluatorModel,
            systemPrompt = """
                Evaluate the quality of an agent's response to a task.
                Output EXACTLY in this format:
                SCORE: <0.0-1.0>
                FEEDBACK: <1-2 sentences explaining the score>
            """.trimIndent()
        )
        val raw = evaluator.run("Task: $task\n\nAttempt:\n$attempt") ?: "SCORE: 0.0\nFEEDBACK: No evaluation"
        val score = Regex("SCORE:\\s*([0-9.]+)").find(raw)?.groupValues?.get(1)?.toDoubleOrNull() ?: 0.0
        val feedback = Regex("FEEDBACK:\\s*(.+)", RegexOption.DOT_MATCHES_ALL).find(raw)?.groupValues?.get(1)?.trim() ?: ""
        return score to feedback
    }

    private suspend fun runReflector(
        task: String, attempt: String, score: Double, feedback: String
    ): String {
        val reflector = AIAgent(
            executor = reflectorExecutor,
            llmModel = reflectorModel,
            systemPrompt = """
                You are a self-reflection system. Analyze why an agent's attempt was suboptimal.
                Generate a concise (2-3 sentence) reflection for the agent to learn from.
                Be specific about what went wrong and what to do differently.
            """.trimIndent()
        )
        return reflector.run(
            "Task: $task\nAttempt (score=${score}):\n$attempt\nEvaluator feedback: $feedback"
        ) ?: "Improve accuracy and completeness."
    }

    private fun loadPriorReflections(task: String) {
        val file = File(memoryPath)
        if (!file.exists()) return
        val priorReflections = file.readLines()
            .mapNotNull { runCatching { json.decodeFromString<ReflexionMemory>(it) }.getOrNull() }
            .filter { it.task == task && it.reflection.isNotBlank() }
            .takeLast(5)
            .map { it.reflection }
        episodicBuffer.value = priorReflections
    }

    private fun persistMemory(mem: ReflexionMemory) {
        File(memoryPath).apply {
            parentFile?.mkdirs()
            appendText(json.encodeToString(mem) + "\n")
        }
    }
}
```

---

## ExpeL Experience Pool (arXiv:2308.10144)

ExpeL maintains a searchable pool of (task, trajectory, outcome) experiences and extracts cross-task insights.

```kotlin
// selfimprove/ExpeL.kt
package com.example.agent.selfimprove

import ai.koog.agents.core.agent.AIAgent
import com.example.agent.memory.EmbeddingProvider
import com.example.agent.memory.MemoryEntry
import com.example.agent.memory.QdrantLongTermMemory
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel

@kotlinx.serialization.Serializable
data class Experience(
    val id: String = java.util.UUID.randomUUID().toString(),
    val task: String,
    val trajectory: List<String>,   // sequence of (thought, action, observation) steps
    val finalResult: String,
    val success: Boolean,
    val qualityScore: Double,
    val extractedInsights: List<String> = emptyList(),
    val timestamp: Long = System.currentTimeMillis()
)

class ExpeL(
    private val vectorStore: QdrantLongTermMemory,
    private val embedder: EmbeddingProvider,
    private val executor: PromptExecutor,
    private val model: LLMModel,
    private val minQualityToStore: Double = 0.6
) {
    /** Add an experience to the pool after quality gating */
    suspend fun addExperience(exp: Experience) {
        if (exp.qualityScore < minQualityToStore) {
            println("[ExpeL] Dropping low-quality experience (score=${exp.qualityScore})")
            return
        }

        val insights = extractInsights(exp)
        val enriched = exp.copy(extractedInsights = insights)

        // Index the experience text in Qdrant
        val text = buildString {
            appendLine("Task: ${exp.task}")
            appendLine("Trajectory: ${exp.trajectory.joinToString(" | ")}")
            appendLine("Result: ${exp.finalResult}")
            appendLine("Insights: ${insights.joinToString("; ")}")
        }
        vectorStore.store(MemoryEntry(
            id = exp.id,
            content = text,
            metadata = mapOf(
                "success" to exp.success.toString(),
                "score" to exp.qualityScore.toString()
            )
        ))
    }

    /** Retrieve similar experiences for a new task */
    suspend fun retrieve(task: String, topK: Int = 5): List<String> {
        val results = vectorStore.search(task, topK)
        return results.map { entry ->
            buildString {
                appendLine("=== Similar Experience ===")
                appendLine(entry.content)
                appendLine("Success: ${entry.metadata["success"]}, Score: ${entry.metadata["score"]}")
            }
        }
    }

    /** Build an in-context example block from retrieved experiences */
    suspend fun buildFewShotContext(task: String): String {
        val experiences = retrieve(task, topK = 3)
        if (experiences.isEmpty()) return ""
        return buildString {
            appendLine("=== RELEVANT PAST EXPERIENCES ===")
            experiences.forEach { appendLine(it) }
            appendLine("=== Use these experiences to guide your approach ===")
        }
    }

    private suspend fun extractInsights(exp: Experience): List<String> {
        val agent = AIAgent(
            executor = executor,
            llmModel = model,
            systemPrompt = """
                Extract generalizable insights from this agent trajectory.
                Focus on what worked, what didn't, and heuristics that could apply to similar tasks.
                Output 2-3 bullet points starting with "INSIGHT: ".
            """.trimIndent()
        )
        val raw = agent.run(
            "Task: ${exp.task}\nTrajectory: ${exp.trajectory.take(5).joinToString("\n")}\nSuccess: ${exp.success}"
        ) ?: ""
        return raw.lines()
            .filter { it.trimStart().startsWith("INSIGHT:") }
            .map { it.removePrefix("INSIGHT:").trim() }
    }
}
```

---

## CRITIC: Tool-Verified Self-Correction

CRITIC uses external tools (code execution, web search, calculators) to verify LLM outputs before accepting them.

```kotlin
// selfimprove/CriticNode.kt
package com.example.agent.selfimprove

import ai.koog.agents.core.agent.AIAgent
import ai.koog.agents.core.strategy.*
import ai.koog.agents.core.tools.ToolRegistry
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel
import ai.koog.prompt.message.AssistantMessage

sealed class CriticEdge {
    object Verified  : CriticEdge()
    object Critique  : CriticEdge()
    object Abandon   : CriticEdge()
}

/** Adds a verification loop after the LLM generates a response */
fun buildCriticStrategy(toolRegistry: ToolRegistry, maxCritiques: Int = 2): AIAgentStrategy {
    return strategy("critic") {
        var critiqueCount = 0

        // Initial generation
        val nodeGenerate = nodeLLMRequest("generate")

        // Tool-augmented verification (runs code, looks up facts, etc.)
        val nodeVerifyTools = nodeExecuteMultipleTools("verifyTools", parallelTools = true)

        // LLM critique using tool results
        val nodeCritique = nodeLLMRequest("critique")

        // Correction based on critique
        val nodeCorrect = nodeLLMRequest("correct")

        val nodeDone = finish<AssistantMessage>("done")

        nodeCritique {
            onAssistantMessage { msg ->
                critiqueCount++
                val verdict = msg.content ?: ""
                when {
                    verdict.contains("VERIFIED",   ignoreCase = true) -> CriticEdge.Verified
                    verdict.contains("INCORRECT",  ignoreCase = true) &&
                    critiqueCount < maxCritiques                       -> CriticEdge.Critique
                    else                                               -> CriticEdge.Abandon
                }
            }
        }

        edge(nodeGenerate    to nodeVerifyTools) { true }
        edge(nodeVerifyTools to nodeCritique)    { true }
        edge(nodeCritique    to nodeCorrect)     { it == CriticEdge.Critique }
        edge(nodeCritique    to nodeDone)        { it == CriticEdge.Verified || it == CriticEdge.Abandon }
        edge(nodeCorrect     to nodeVerifyTools) { true }

        setStart(nodeGenerate)
    }
}
```

---

## SELF-REFINE: Iterative Refinement (selfrefine.info)

```kotlin
// selfimprove/SelfRefine.kt
package com.example.agent.selfimprove

import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel

data class RefinementConfig(
    val maxIterations: Int = 4,
    val acceptKeyword: String = "LGTM",
    val critiquePrompt: String = DEFAULT_CRITIQUE_PROMPT,
    val refinePrompt: String = DEFAULT_REFINE_PROMPT
) {
    companion object {
        const val DEFAULT_CRITIQUE_PROMPT = """
            Critique the given output for the task. Be specific and actionable.
            Identify: factual errors, logical gaps, missing information, poor structure.
            If the output is excellent, respond with just: LGTM
            Otherwise start your critique with: ISSUES:
        """.trimIndent()

        const val DEFAULT_REFINE_PROMPT = """
            Refine the given output based on the critique.
            Address each issue mentioned. Improve quality substantially.
            Output the refined version only, no preamble.
        """.trimIndent()
    }
}

class SelfRefineAgent(
    private val executor: PromptExecutor,
    private val model: LLMModel,
    private val cfg: RefinementConfig = RefinementConfig()
) {
    suspend fun run(task: String, initialOutput: String? = null): String {
        val generator = AIAgent(executor, llmModel = model,
            systemPrompt = "You are a high-quality content generator.")
        val critiquer = AIAgent(executor, llmModel = model,
            systemPrompt = cfg.critiquePrompt)
        val refiner   = AIAgent(executor, llmModel = model,
            systemPrompt = cfg.refinePrompt)

        // Generate initial output if not provided
        var current = initialOutput ?: (generator.run(task) ?: "")
        println("[SELF-REFINE] Initial output: ${current.take(100)}")

        for (iteration in 1..cfg.maxIterations) {
            // Critique
            val critique = critiquer.run("Task: $task\n\nOutput:\n$current") ?: ""
            println("[SELF-REFINE] Iteration $iteration critique: ${critique.take(100)}")

            // Accept if critique says OK
            if (critique.trim().startsWith(cfg.acceptKeyword, ignoreCase = true)) {
                println("[SELF-REFINE] Accepted at iteration $iteration")
                break
            }

            if (!critique.contains("ISSUES:", ignoreCase = true) && iteration > 1) break

            // Refine
            val refined = refiner.run(
                "Task: $task\n\nCurrent output:\n$current\n\nCritique:\n$critique"
            ) ?: current
            current = refined
            println("[SELF-REFINE] Refined output: ${current.take(100)}")
        }

        return current
    }
}
```

---

## Koog LLM-as-a-Judge Verification

```kotlin
// selfimprove/LlmJudge.kt
package com.example.agent.selfimprove

import ai.koog.agents.core.agent.AIAgent
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel

data class JudgmentResult(
    val score: Double,      // 0.0 - 1.0
    val reasoning: String,
    val verdict: String     // "PASS" or "FAIL"
)

class LlmJudge(
    private val executor: PromptExecutor,
    private val model: LLMModel,
    private val criteria: List<String> = listOf(
        "Factual accuracy",
        "Completeness",
        "Clarity and structure",
        "Actionability"
    )
) {
    suspend fun judge(task: String, response: String): JudgmentResult {
        val criteriaText = criteria.mapIndexed { i, c -> "${i + 1}. $c" }.joinToString("\n")
        val agent = AIAgent(
            executor = executor,
            llmModel = model,
            systemPrompt = """
                You are an impartial judge evaluating AI responses.
                Evaluation criteria:
                $criteriaText
                
                Score format (respond EXACTLY like this):
                SCORE: <0.0-1.0>
                VERDICT: <PASS if score >= 0.7, else FAIL>
                REASONING: <2-3 sentences>
            """.trimIndent()
        )
        val raw = agent.run("Task: $task\n\nResponse:\n$response") ?: ""
        val score = Regex("SCORE:\\s*([0-9.]+)").find(raw)?.groupValues?.get(1)?.toDoubleOrNull() ?: 0.0
        val verdict = if (Regex("VERDICT:\\s*PASS").containsMatchIn(raw)) "PASS" else "FAIL"
        val reasoning = Regex("REASONING:\\s*(.+)", RegexOption.DOT_MATCHES_ALL)
            .find(raw)?.groupValues?.get(1)?.trim() ?: ""
        return JudgmentResult(score, reasoning, verdict)
    }
}
```

---

## Voyager Skill Library

```kotlin
// selfimprove/SkillLibrary.kt
package com.example.agent.selfimprove

import ai.koog.agents.core.agent.AIAgent
import com.example.agent.memory.EmbeddingProvider
import com.example.agent.memory.MemoryEntry
import com.example.agent.memory.QdrantLongTermMemory
import ai.koog.prompt.executor.llms.PromptExecutor
import ai.koog.prompt.llm.LLMModel
import kotlinx.serialization.Serializable

@Serializable
data class Skill(
    val id: String = java.util.UUID.randomUUID().toString(),
    val name: String,
    val description: String,
    val code: String,           // Kotlin code or pseudo-code for the skill
    val language: String = "kotlin",
    val successCount: Int = 0,
    val usageCount: Int = 0,
    val tags: List<String> = emptyList()
) {
    val successRate: Double get() = if (usageCount == 0) 0.0 else successCount.toDouble() / usageCount
}

class SkillLibrary(
    private val vectorStore: QdrantLongTermMemory,
    private val embedder: EmbeddingProvider,
    private val executor: PromptExecutor,
    private val model: LLMModel
) {
    /** Store a new skill (after quality verification) */
    suspend fun storeSkill(skill: Skill) {
        val text = "${skill.name}: ${skill.description}\nCode:\n${skill.code}"
        vectorStore.store(MemoryEntry(
            id = skill.id,
            content = text,
            metadata = mapOf(
                "name" to skill.name,
                "language" to skill.language,
                "tags" to skill.tags.joinToString(",")
            )
        ))
    }

    /** Retrieve most relevant skills for a task */
    suspend fun retrieveSkills(task: String, topK: Int = 3): List<String> {
        return vectorStore.search(task, topK).map { entry ->
            "Skill: ${entry.metadata["name"]}\n${entry.content}"
        }
    }

    /** Extract a reusable skill from a successful trajectory */
    suspend fun extractSkill(
        task: String,
        trajectory: List<String>,
        result: String
    ): Skill? {
        val agent = AIAgent(
            executor = executor,
            llmModel = model,
            systemPrompt = """
                Extract a reusable skill from this successful agent trajectory.
                Output EXACTLY in this JSON format:
                {
                  "name": "<short skill name>",
                  "description": "<what this skill does>",
                  "code": "<pseudocode or actual code>",
                  "tags": ["tag1", "tag2"]
                }
                If no generalizable skill can be extracted, output: NOT_EXTRACTABLE
            """.trimIndent()
        )
        val raw = agent.run(
            "Task: $task\nTrajectory:\n${trajectory.joinToString("\n")}\nResult: $result"
        ) ?: return null

        if (raw.contains("NOT_EXTRACTABLE")) return null

        return runCatching {
            val jsonStr = Regex("\\{.*\\}", RegexOption.DOT_MATCHES_ALL).find(raw)?.value ?: return null
            val json = kotlinx.serialization.json.Json { ignoreUnknownKeys = true }
            val parsed = json.decodeFromString<Map<String, kotlinx.serialization.json.JsonElement>>(jsonStr)
            Skill(
                name = parsed["name"]?.toString()?.trim('"') ?: "unnamed",
                description = parsed["description"]?.toString()?.trim('"') ?: "",
                code = parsed["code"]?.toString()?.trim('"') ?: ""
            )
        }.getOrNull()
    }

    /** Build few-shot skill context for a new task */
    suspend fun buildSkillContext(task: String): String {
        val skills = retrieveSkills(task)
        if (skills.isEmpty()) return ""
        return "=== AVAILABLE SKILLS ===\n${skills.joinToString("\n---\n")}\n=== END SKILLS ==="
    }
}
```

---

## Quality Scoring Before Storing

```kotlin
// selfimprove/QualityGate.kt
package com.example.agent.selfimprove

data class QualityGateConfig(
    val minScore: Double = 0.65,
    val minLength: Int = 20,
    val requiresSuccess: Boolean = false
)

class QualityGate(
    private val judge: LlmJudge,
    private val cfg: QualityGateConfig = QualityGateConfig()
) {
    suspend fun evaluate(
        task: String,
        response: String,
        succeeded: Boolean = true
    ): Pair<Boolean, JudgmentResult> {
        if (response.length < cfg.minLength) {
            return false to JudgmentResult(0.0, "Response too short", "FAIL")
        }
        if (cfg.requiresSuccess && !succeeded) {
            return false to JudgmentResult(0.0, "Task marked as failed", "FAIL")
        }
        val judgment = judge.judge(task, response)
        return (judgment.score >= cfg.minScore) to judgment
    }
}
```

---

## Improvement Metrics Tracking

```kotlin
// selfimprove/ImprovementTracker.kt
package com.example.agent.selfimprove

import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.serialization.Serializable

@Serializable
data class ImprovementMetrics(
    val totalAttempts: Int = 0,
    val successfulAttempts: Int = 0,
    val averageScore: Double = 0.0,
    val averageTrialsToSuccess: Double = 0.0,
    val skillsAcquired: Int = 0,
    val experiencesStored: Int = 0
) {
    val successRate: Double get() = if (totalAttempts == 0) 0.0
                                    else successfulAttempts.toDouble() / totalAttempts
}

class ImprovementTracker {
    private val _metrics = MutableStateFlow(ImprovementMetrics())
    val metrics: StateFlow<ImprovementMetrics> = _metrics

    private val scoreHistory = mutableListOf<Double>()
    private val trialsHistory = mutableListOf<Int>()

    fun recordAttempt(score: Double, trials: Int, succeeded: Boolean) {
        scoreHistory.add(score)
        if (succeeded) trialsHistory.add(trials)
        _metrics.value = _metrics.value.copy(
            totalAttempts = _metrics.value.totalAttempts + 1,
            successfulAttempts = if (succeeded) _metrics.value.successfulAttempts + 1
                                 else _metrics.value.successfulAttempts,
            averageScore = scoreHistory.average(),
            averageTrialsToSuccess = if (trialsHistory.isEmpty()) 0.0 else trialsHistory.average()
        )
    }

    fun recordSkillAcquired() {
        _metrics.value = _metrics.value.copy(skillsAcquired = _metrics.value.skillsAcquired + 1)
    }

    fun recordExperienceStored() {
        _metrics.value = _metrics.value.copy(experiencesStored = _metrics.value.experiencesStored + 1)
    }

    fun printSummary() {
        val m = _metrics.value
        println("""
            === Improvement Metrics ===
            Success rate:           ${"%.1f".format(m.successRate * 100)}%
            Average score:          ${"%.3f".format(m.averageScore)}
            Avg trials to success:  ${"%.1f".format(m.averageTrialsToSuccess)}
            Skills acquired:        ${m.skillsAcquired}
            Experiences stored:     ${m.experiencesStored}
        """.trimIndent())
    }
}
```
