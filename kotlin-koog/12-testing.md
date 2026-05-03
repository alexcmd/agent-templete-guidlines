# Testing Agent Systems — Kotlin / Koog

Testing Koog agents combines standard JVM testing idioms (JUnit 5, Kotest) with agent-specific concerns: tool schema validation, graph traversal, and real-API integration tests. This guide covers all layers.

---

## §12.1 Testing Strategy

```
Unit Tests (fast, no API)
  ├─ Tool schema validation (Kotlinx Serialization)
  ├─ Tool handler logic (mock execution)
  ├─ Graph edge conditions
  ├─ State reducers
  └─ Token estimation

Integration Tests (slow, real API)
  ├─ Single-turn tool execution
  ├─ Multi-turn context continuity
  └─ Error recovery paths

Evaluation Harnesses (expensive)
  ├─ Task success rate benchmark
  ├─ Regression suite across model versions
  └─ Latency/cost profiling
```

**Key rule**: Do not mock the Anthropic client in integration tests. Real API calls are the only way to catch prompt/schema regressions. Use `@Tag("integration")` to separate unit and integration layers.

---

## §12.2 Project Structure

```
my-agent/
├── src/
│   ├── main/kotlin/com/example/agent/
│   │   ├── tools/
│   │   ├── graph/
│   │   ├── state/
│   │   └── memory/
│   └── test/kotlin/com/example/agent/
│       ├── unit/
│       │   ├── ToolSchemaTest.kt
│       │   ├── GraphTransitionTest.kt
│       │   └── TokenEstimationTest.kt
│       ├── integration/
│       │   ├── AgentLoopTest.kt
│       │   └── ToolE2ETest.kt
│       └── evals/
│           ├── BenchmarkRunner.kt
│           └── Scorers.kt
```

---

## §12.3 Unit Tests

### Tool Schema Validation

```kotlin
// src/test/kotlin/com/example/agent/unit/ToolSchemaTest.kt
import io.kotest.core.spec.style.DescribeSpec
import io.kotest.matchers.shouldBe
import io.kotest.matchers.shouldNotBe
import io.kotest.assertions.throwables.shouldThrow
import kotlinx.serialization.json.*
import com.example.agent.tools.BashTool

class ToolSchemaTest : DescribeSpec({
    val tool = BashTool()

    describe("BashTool schema") {
        it("accepts valid command") {
            val input = buildJsonObject {
                put("command", "echo hello")
                put("timeout", 30)
            }
            val parsed = tool.parseInput(input)
            parsed shouldNotBe null
            parsed.command shouldBe "echo hello"
        }

        it("rejects missing command") {
            val input = buildJsonObject {
                put("timeout", 30)  // no command
            }
            shouldThrow<ToolInputException> {
                tool.parseInput(input)
            }
        }

        it("uses default timeout") {
            val input = buildJsonObject {
                put("command", "ls")
            }
            val parsed = tool.parseInput(input)
            parsed.timeout shouldBe 120  // default
        }
    }
})
```

### Graph Transition Tests

```kotlin
// src/test/kotlin/com/example/agent/unit/GraphTransitionTest.kt
import io.kotest.core.spec.style.DescribeSpec
import io.kotest.matchers.shouldBe
import com.example.agent.graph.*
import com.example.agent.state.AgentState

class GraphTransitionTest : DescribeSpec({
    describe("shouldContinue edge function") {
        it("routes to END when no tool calls") {
            val state = AgentState(
                messages = listOf(
                    AssistantMessage(content = "Done.", toolCalls = emptyList())
                )
            )
            shouldContinue(state) shouldBe GraphNode.END
        }

        it("routes to tool_node when tool calls present") {
            val state = AgentState(
                messages = listOf(
                    AssistantMessage(
                        content = "",
                        toolCalls = listOf(ToolCall(id = "t1", name = "bash", input = mapOf("command" to "ls")))
                    )
                )
            )
            shouldContinue(state) shouldBe GraphNode.TOOL_NODE
        }

        it("routes to END when max iterations reached") {
            val state = AgentState(
                messages = emptyList(),
                iterationCount = 50
            )
            shouldContinue(state) shouldBe GraphNode.END
        }
    }
})
```

### Tool Handler Logic (Mocked Execution)

```kotlin
// src/test/kotlin/com/example/agent/unit/BashToolHandlerTest.kt
import io.kotest.core.spec.style.DescribeSpec
import io.kotest.matchers.shouldBe
import io.mockk.*
import com.example.agent.tools.BashTool
import com.example.agent.tools.ProcessRunner

class BashToolHandlerTest : DescribeSpec({
    val processRunner = mockk<ProcessRunner>()
    val tool = BashTool(processRunner = processRunner)

    describe("BashTool handler") {
        it("returns stdout on success") {
            coEvery { processRunner.run("echo hello", any()) } returns ProcessResult(
                stdout = "hello\n",
                stderr = "",
                exitCode = 0,
            )
            val result = tool.execute(command = "echo hello")
            result.output shouldBe "hello\n"
            result.isError shouldBe false
        }

        it("returns error on nonzero exit") {
            coEvery { processRunner.run("nosuchcmd", any()) } returns ProcessResult(
                stdout = "",
                stderr = "command not found",
                exitCode = 127,
            )
            val result = tool.execute(command = "nosuchcmd")
            result.isError shouldBe true
            result.output.contains("command not found") shouldBe true
        }

        it("reports timeout as error") {
            coEvery { processRunner.run(any(), any()) } throws ProcessTimeoutException("timeout")
            val result = tool.execute(command = "sleep 999", timeout = 1)
            result.isError shouldBe true
            result.output.contains("timeout") shouldBe true
        }
    }
})
```

### Token Estimation

```kotlin
// src/test/kotlin/com/example/agent/unit/TokenEstimationTest.kt
import io.kotest.core.spec.style.DescribeSpec
import io.kotest.matchers.ints.shouldBeBetween
import io.kotest.matchers.shouldBe
import com.example.agent.tokens.*

class TokenEstimationTest : DescribeSpec({
    describe("roughTokenEstimate") {
        it("estimates zero for empty content") {
            roughTokenEstimate("") shouldBe 0
        }

        it("estimates ~1000 tokens for 4000 chars") {
            val estimate = roughTokenEstimate("a".repeat(4000))
            estimate.shouldBeBetween(1000, 1500)
        }
    }

    describe("tokenCountWithEstimation") {
        it("uses API usage when available") {
            val messages = listOf(
                Message(role = "user", content = "hello"),
                Message(
                    role = "assistant",
                    content = "world",
                    usage = Usage(inputTokens = 5000, outputTokens = 100)
                )
            )
            tokenCountWithEstimation(messages) shouldBe 5100
        }

        it("falls back to estimation when no API usage") {
            val messages = listOf(
                Message(role = "user", content = "hello world")
            )
            val estimate = tokenCountWithEstimation(messages)
            estimate.shouldBeBetween(2, 10)
        }
    }
})
```

---

## §12.4 Integration Tests

```kotlin
// src/test/kotlin/com/example/agent/integration/AgentLoopTest.kt
import io.kotest.core.spec.style.DescribeSpec
import io.kotest.matchers.shouldNotBe
import io.kotest.matchers.string.shouldContain
import org.junit.jupiter.api.Tag

@Tag("integration")
class AgentLoopTest : DescribeSpec({
    val apiKey = System.getenv("ANTHROPIC_API_KEY")
        ?: return@DescribeSpec  // skip if no API key

    val agent = createTestAgent(apiKey, model = "claude-haiku-4-5-20251001")

    describe("single turn") {
        it("answers simple factual question") {
            val result = agent.invoke(
                messages = listOf(userMessage("What is 2+2?")),
                threadId = newThreadId(),
            )
            val last = result.messages.last()
            last.role shouldBe "assistant"
            last.content shouldContain "4"
        }

        it("executes bash tool") {
            val result = agent.invoke(
                messages = listOf(userMessage("Run: echo 'test-marker-kotlin-99'")),
                threadId = newThreadId(),
            )
            val toolOutputs = result.messages.filter { it.role == "tool" }
            toolOutputs.any { "test-marker-kotlin-99" in it.content } shouldBe true
        }
    }

    describe("multi-turn context") {
        it("preserves conversation history") {
            val threadId = newThreadId()
            agent.invoke(
                messages = listOf(userMessage("Remember: secret code is BLUE-42")),
                threadId = threadId,
            )
            val result = agent.invoke(
                messages = listOf(userMessage("What is the secret code?")),
                threadId = threadId,
            )
            result.messages.last().content shouldContain "BLUE-42"
        }
    }

    describe("error recovery") {
        it("handles tool failure gracefully") {
            val result = agent.invoke(
                messages = listOf(userMessage("Cat this file: /does/not/exist/xyz.txt")),
                threadId = newThreadId(),
            )
            val last = result.messages.last()
            last.role shouldBe "assistant"
            // Model should acknowledge the error
            val hasError = listOf("error", "not found", "doesn't exist", "cannot")
                .any { it in last.content.lowercase() }
            hasError shouldBe true
        }
    }
})
```

---

## §12.5 Evaluation Harness

```kotlin
// src/test/kotlin/com/example/agent/evals/BenchmarkRunner.kt
import com.example.agent.evals.Scorers
import io.kotest.core.spec.style.DescribeSpec
import io.kotest.matchers.doubles.shouldBeGreaterThanOrEqualTo
import kotlinx.serialization.json.*
import java.nio.file.Files
import java.nio.file.Path
import kotlin.time.measureTimedValue

data class EvalTask(
    val id: String,
    val category: String,
    val input: String,
    val expectedTool: String? = null,
    val expectedOutputContains: List<String> = emptyList(),
    val scorer: String = "contains_all",
)

data class EvalResult(
    val taskId: String,
    val passed: Boolean,
    val score: Double,
    val reason: String,
    val latencyMs: Long,
)

@Tag("eval")
class BenchmarkRunner : DescribeSpec({
    val apiKey = System.getenv("ANTHROPIC_API_KEY") ?: return@DescribeSpec
    val agent = createTestAgent(apiKey)
    val tasks = loadTasks("src/test/resources/evals/tasks.json")
    val results = mutableListOf<EvalResult>()

    for (task in tasks) {
        it("eval: ${task.id}") {
            val (result, duration) = measureTimedValue {
                agent.invoke(
                    messages = listOf(userMessage(task.input)),
                    threadId = "eval-${task.id}",
                )
            }
            val lastContent = result.messages.last().content
            val (passed, score, reason) = when (task.scorer) {
                "contains_all" -> Scorers.containsAll(task, lastContent)
                "tool_used"    -> Scorers.toolUsed(task, result.messages)
                else           -> Scorers.containsAll(task, lastContent)
            }
            results += EvalResult(task.id, passed, score, reason, duration.inWholeMilliseconds)
        }
    }

    afterSpec {
        val passRate = results.count { it.passed }.toDouble() / results.size
        val avgScore = results.map { it.score }.average()
        println("\n=== Eval Results ===")
        println("Pass rate: ${(passRate * 100).toInt()}%  Avg score: ${"%.2f".format(avgScore)}")
        results.filter { !it.passed }.forEach {
            println("  FAIL ${it.taskId}: ${it.reason}")
        }
        passRate shouldBeGreaterThanOrEqualTo 0.80
    }
})

fun loadTasks(path: String): List<EvalTask> {
    val json = Json { ignoreUnknownKeys = true }
    val text = Files.readString(Path.of(path))
    return json.decodeFromString(text)
}
```

```kotlin
// src/test/kotlin/com/example/agent/evals/Scorers.kt
object Scorers {
    fun containsAll(task: EvalTask, output: String): Triple<Boolean, Double, String> {
        val lower = output.lowercase()
        val matches = task.expectedOutputContains.map { it.lowercase() in lower }
        val score = if (matches.isEmpty()) 1.0 else matches.count { it }.toDouble() / matches.size
        val reason = "${matches.count { it }}/${matches.size} keywords found"
        return Triple(score == 1.0, score, reason)
    }

    fun toolUsed(task: EvalTask, messages: List<Message>): Triple<Boolean, Double, String> {
        val toolCalls = messages
            .filter { it.role == "assistant" }
            .flatMap { m -> m.toolCalls ?: emptyList() }
            .map { it.name }
        val used = task.expectedTool in toolCalls
        return Triple(used, if (used) 1.0 else 0.0, "Expected ${task.expectedTool}, saw $toolCalls")
    }
}
```

---

## §12.6 Test Fixtures

```kotlin
// src/test/kotlin/com/example/agent/TestFixtures.kt
import java.util.UUID

fun userMessage(content: String) = Message(role = "user", content = content)

fun newThreadId() = UUID.randomUUID().toString()

fun createTestAgent(apiKey: String, model: String = "claude-haiku-4-5-20251001") =
    Agent(
        apiKey = apiKey,
        model = model,
        maxIterations = 10,  // lower for tests
        timeout = 60_000,    // 60 seconds per turn
    )
```

---

## §12.7 Gradle Configuration

```kotlin
// build.gradle.kts
tasks.withType<Test> {
    useJUnitPlatform()
    testLogging { events("passed", "skipped", "failed") }
}

// Separate integration test source set
sourceSets {
    create("integrationTest") {
        kotlin.srcDir("src/test/kotlin")
        compileClasspath += sourceSets["main"].output + configurations["testRuntimeClasspath"]
        runtimeClasspath += output + compileClasspath
    }
}

task<Test>("integrationTest") {
    description = "Runs integration tests"
    group = "verification"
    testClassesDirs = sourceSets["integrationTest"].output.classesDirs
    classpath = sourceSets["integrationTest"].runtimeClasspath
    systemProperty("ANTHROPIC_API_KEY", System.getenv("ANTHROPIC_API_KEY") ?: "")
    useJUnitPlatform { includeTags("integration") }
}

dependencies {
    testImplementation("io.kotest:kotest-runner-junit5:5.8.0")
    testImplementation("io.kotest:kotest-assertions-core:5.8.0")
    testImplementation("io.mockk:mockk:1.13.9")
}
```

---

## §12.8 CI Configuration

```yaml
# .github/workflows/test.yml
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: "17", distribution: "temurin" }
      - run: ./gradlew test -x integrationTest

  integration:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    env:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: "17", distribution: "temurin" }
      - run: ./gradlew integrationTest
```

---

*[← Complete Example](./11-complete-example.md) | [Index →](./00-index.md)*
