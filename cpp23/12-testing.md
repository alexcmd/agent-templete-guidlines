# Testing Agent Systems — C++23

Testing C++23 agent systems uses GoogleTest for unit/integration layers and a custom harness for LLM evaluation. The core challenge is isolating the LLM call from deterministic components without mocking away real behavior.

---

## §12.1 Testing Strategy

```
Unit Tests (fast, no API)
  ├─ Tool schema validation (nlohmann/json)
  ├─ Tool handler logic (mock execution)
  ├─ ReAct loop state machine
  ├─ Permission rule evaluation
  └─ Token estimation

Integration Tests (slow, real API)
  ├─ Single-turn tool execution
  ├─ Multi-turn conversation context
  ├─ Error recovery
  └─ Compaction triggers

Evaluation Harnesses (expensive)
  ├─ Task success rate benchmark
  ├─ Model regression suite
  └─ Latency/cost profiling
```

**Key rule**: Never mock the HTTP client in integration tests. Use a real `AnthropicClient` pointing to the actual API. Gate integration tests behind `ANTHROPIC_API_KEY` environment variable check.

---

## §12.2 Project Structure

```
my-agent/
├── src/
│   ├── tools/
│   ├── agent/
│   ├── memory/
│   └── utils/
└── tests/
    ├── CMakeLists.txt
    ├── unit/
    │   ├── tool_schema_test.cpp
    │   ├── agent_loop_test.cpp
    │   └── token_estimation_test.cpp
    ├── integration/
    │   ├── agent_e2e_test.cpp
    │   └── tool_e2e_test.cpp
    └── evals/
        ├── benchmark.cpp
        ├── scorers.hpp
        └── fixtures/
            └── tasks.json
```

---

## §12.3 Unit Tests

### Tool Schema Validation

```cpp
// tests/unit/tool_schema_test.cpp
#include <gtest/gtest.h>
#include <nlohmann/json.hpp>
#include "tools/bash_tool.hpp"

using json = nlohmann::json;

class BashToolSchemaTest : public ::testing::Test {
protected:
    BashTool tool_;
};

TEST_F(BashToolSchemaTest, ValidCommandAccepted) {
    json input = {{"command", "echo hello"}, {"timeout", 30}};
    auto parsed = tool_.ParseInput(input);
    ASSERT_TRUE(parsed.has_value());
    EXPECT_EQ(parsed->command, "echo hello");
    EXPECT_EQ(parsed->timeout, 30);
}

TEST_F(BashToolSchemaTest, MissingCommandThrows) {
    json input = {{"timeout", 30}};
    auto parsed = tool_.ParseInput(input);
    EXPECT_FALSE(parsed.has_value());
    EXPECT_EQ(parsed.error().code, ToolError::kMissingRequired);
}

TEST_F(BashToolSchemaTest, DefaultTimeout) {
    json input = {{"command", "ls"}};
    auto parsed = tool_.ParseInput(input);
    ASSERT_TRUE(parsed.has_value());
    EXPECT_EQ(parsed->timeout, 120);  // default
}

TEST_F(BashToolSchemaTest, DangerousCommandDetected) {
    EXPECT_TRUE(tool_.IsDangerous("rm -rf /"));
    EXPECT_TRUE(tool_.IsDangerous("sudo rm -rf /home"));
    EXPECT_FALSE(tool_.IsDangerous("echo hello"));
    EXPECT_FALSE(tool_.IsDangerous("ls -la"));
}
```

### Tool Handler (Mocked Process Execution)

```cpp
// tests/unit/bash_tool_handler_test.cpp
#include <gtest/gtest.h>
#include <gmock/gmock.h>
#include "tools/bash_tool.hpp"
#include "mocks/mock_process_runner.hpp"

using ::testing::Return;
using ::testing::_;

class BashToolHandlerTest : public ::testing::Test {
protected:
    void SetUp() override {
        mock_runner_ = std::make_shared<MockProcessRunner>();
        tool_ = std::make_unique<BashTool>(mock_runner_);
    }

    std::shared_ptr<MockProcessRunner> mock_runner_;
    std::unique_ptr<BashTool> tool_;
};

TEST_F(BashToolHandlerTest, SuccessfulExecution) {
    ProcessResult result{.stdout_output = "hello\n", .stderr_output = "", .exit_code = 0};
    EXPECT_CALL(*mock_runner_, Run("echo hello", _))
        .WillOnce(Return(result));

    auto output = tool_->Execute({.command = "echo hello"});
    EXPECT_EQ(output.content, "hello\n");
    EXPECT_FALSE(output.is_error);
}

TEST_F(BashToolHandlerTest, NonzeroExitIsError) {
    ProcessResult result{.stdout_output = "", .stderr_output = "command not found", .exit_code = 127};
    EXPECT_CALL(*mock_runner_, Run("nosuchcmd", _))
        .WillOnce(Return(result));

    auto output = tool_->Execute({.command = "nosuchcmd"});
    EXPECT_TRUE(output.is_error);
    EXPECT_THAT(output.content, ::testing::HasSubstr("command not found"));
}

TEST_F(BashToolHandlerTest, TimeoutReturnsError) {
    EXPECT_CALL(*mock_runner_, Run(_, _))
        .WillOnce([](...) -> ProcessResult {
            throw ProcessTimeoutException("process timed out");
        });

    auto output = tool_->Execute({.command = "sleep 999", .timeout = 1});
    EXPECT_TRUE(output.is_error);
    EXPECT_THAT(output.content, ::testing::HasSubstr("timeout"));
}
```

### ReAct Loop State Machine

```cpp
// tests/unit/agent_loop_test.cpp
#include <gtest/gtest.h>
#include "agent/react_loop.hpp"

class ReActLoopTest : public ::testing::Test {};

TEST_F(ReActLoopTest, NoToolCallsTransitionsToEnd) {
    AgentState state;
    state.messages.push_back(Message{
        .role = "assistant",
        .content = "Done.",
        .tool_calls = {},
    });

    auto next = DetermineNextNode(state);
    EXPECT_EQ(next, GraphNode::kEnd);
}

TEST_F(ReActLoopTest, ToolCallsTransitionToToolNode) {
    AgentState state;
    state.messages.push_back(Message{
        .role = "assistant",
        .content = "",
        .tool_calls = {ToolCall{.id = "t1", .name = "bash", .input = {{"command", "ls"}}}},
    });

    auto next = DetermineNextNode(state);
    EXPECT_EQ(next, GraphNode::kToolNode);
}

TEST_F(ReActLoopTest, MaxIterationsEndsLoop) {
    AgentState state;
    state.iteration_count = 50;

    auto next = DetermineNextNode(state);
    EXPECT_EQ(next, GraphNode::kEnd);
}
```

### Token Estimation

```cpp
// tests/unit/token_estimation_test.cpp
#include <gtest/gtest.h>
#include "utils/tokens.hpp"

TEST(TokenEstimation, EmptyStringReturnsZero) {
    EXPECT_EQ(RoughTokenEstimate(""), 0);
}

TEST(TokenEstimation, Approximates1000TokensFor4000Chars) {
    auto estimate = RoughTokenEstimate(std::string(4000, 'a'));
    EXPECT_GE(estimate, 1000);
    EXPECT_LE(estimate, 1500);
}

TEST(TokenEstimation, UsesApiUsageWhenAvailable) {
    std::vector<Message> messages = {
        {.role = "user", .content = "hello"},
        {.role = "assistant", .content = "world",
         .usage = TokenUsage{.input_tokens = 5000, .output_tokens = 100}},
    };
    auto count = TokenCountWithEstimation(messages);
    EXPECT_EQ(count, 5100);
}

TEST(TokenEstimation, FallsBackToEstimationWithNoUsage) {
    std::vector<Message> messages = {
        {.role = "user", .content = "hello world"},
    };
    auto count = TokenCountWithEstimation(messages);
    EXPECT_GE(count, 2);
    EXPECT_LE(count, 10);
}
```

---

## §12.4 Integration Tests

```cpp
// tests/integration/agent_e2e_test.cpp
#include <gtest/gtest.h>
#include <cstdlib>
#include "agent/agent.hpp"

class AgentE2ETest : public ::testing::Test {
protected:
    void SetUp() override {
        const char* key = std::getenv("ANTHROPIC_API_KEY");
        if (!key) {
            GTEST_SKIP() << "ANTHROPIC_API_KEY not set";
        }
        agent_ = std::make_unique<Agent>(AgentConfig{
            .api_key = key,
            .model = "claude-haiku-4-5-20251001",
            .max_iterations = 10,
        });
    }

    std::unique_ptr<Agent> agent_;

    std::string NewThreadId() {
        return "test-" + std::to_string(
            std::chrono::system_clock::now().time_since_epoch().count()
        );
    }
};

TEST_F(AgentE2ETest, AnswersSimpleQuestion) {
    auto result = agent_->Invoke({
        .messages = {Message{.role = "user", .content = "What is 2+2?"}},
        .thread_id = NewThreadId(),
    });
    ASSERT_FALSE(result.messages.empty());
    auto& last = result.messages.back();
    EXPECT_EQ(last.role, "assistant");
    EXPECT_THAT(last.content, ::testing::HasSubstr("4"));
}

TEST_F(AgentE2ETest, ExecutesBashTool) {
    auto result = agent_->Invoke({
        .messages = {Message{.role = "user", .content = "Run: echo 'cpp-test-marker-999'"}},
        .thread_id = NewThreadId(),
    });
    bool marker_found = false;
    for (const auto& msg : result.messages) {
        if (msg.role == "tool" && msg.content.find("cpp-test-marker-999") != std::string::npos) {
            marker_found = true;
            break;
        }
    }
    EXPECT_TRUE(marker_found);
}

TEST_F(AgentE2ETest, PreservesMultiTurnContext) {
    auto thread_id = NewThreadId();

    agent_->Invoke({
        .messages = {Message{.role = "user", .content = "Remember: secret is GREEN-77"}},
        .thread_id = thread_id,
    });

    auto result = agent_->Invoke({
        .messages = {Message{.role = "user", .content = "What is the secret?"}},
        .thread_id = thread_id,
    });

    EXPECT_THAT(result.messages.back().content, ::testing::HasSubstr("GREEN-77"));
}

TEST_F(AgentE2ETest, HandlesToolFailureGracefully) {
    auto result = agent_->Invoke({
        .messages = {Message{.role = "user", .content = "Cat this file: /no/such/file.txt"}},
        .thread_id = NewThreadId(),
    });
    auto& last = result.messages.back();
    EXPECT_EQ(last.role, "assistant");
    // Model should describe the error, not crash
    bool has_error_word = false;
    for (const auto& word : {"error", "not found", "cannot", "doesn't exist"}) {
        if (last.content.find(word) != std::string::npos) {
            has_error_word = true;
            break;
        }
    }
    EXPECT_TRUE(has_error_word);
}
```

---

## §12.5 Evaluation Harness

```cpp
// tests/evals/benchmark.cpp
#include <gtest/gtest.h>
#include <nlohmann/json.hpp>
#include <fstream>
#include <chrono>
#include "agent/agent.hpp"
#include "evals/scorers.hpp"

struct EvalTask {
    std::string id;
    std::string category;
    std::string input;
    std::string expected_tool;
    std::vector<std::string> expected_output_contains;
    std::string scorer = "contains_all";
};

struct EvalResult {
    std::string task_id;
    bool passed;
    double score;
    std::string reason;
    long latency_ms;
};

std::vector<EvalTask> LoadTasks(const std::string& path) {
    std::ifstream f(path);
    auto j = nlohmann::json::parse(f);
    std::vector<EvalTask> tasks;
    for (const auto& item : j) {
        tasks.push_back({
            .id = item["id"],
            .category = item["category"],
            .input = item["input"],
            .expected_tool = item.value("expected_tool", ""),
            .expected_output_contains = item.value("expected_output_contains", std::vector<std::string>{}),
            .scorer = item.value("scorer", "contains_all"),
        });
    }
    return tasks;
}

class BenchmarkTest : public ::testing::Test {
protected:
    void SetUp() override {
        const char* key = std::getenv("ANTHROPIC_API_KEY");
        if (!key) GTEST_SKIP() << "ANTHROPIC_API_KEY not set";
        agent_ = std::make_unique<Agent>(AgentConfig{.api_key = key});
    }

    std::unique_ptr<Agent> agent_;
    std::vector<EvalResult> results_;
};

TEST_F(BenchmarkTest, RunBenchmarkSuite) {
    auto tasks = LoadTasks("tests/evals/fixtures/tasks.json");

    for (const auto& task : tasks) {
        auto start = std::chrono::steady_clock::now();

        auto result = agent_->Invoke({
            .messages = {Message{.role = "user", .content = task.input}},
            .thread_id = "eval-" + task.id,
        });

        auto latency = std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::steady_clock::now() - start
        ).count();

        auto last_content = result.messages.back().content;
        auto [passed, score, reason] = [&]() -> std::tuple<bool, double, std::string> {
            if (task.scorer == "contains_all")
                return Scorers::ContainsAll(task.expected_output_contains, last_content);
            if (task.scorer == "tool_used")
                return Scorers::ToolUsed(task.expected_tool, result.messages);
            return Scorers::ContainsAll(task.expected_output_contains, last_content);
        }();

        results_.push_back({task.id, passed, score, reason, latency});
    }

    // Report
    int passed = std::count_if(results_.begin(), results_.end(), [](const auto& r) { return r.passed; });
    double pass_rate = static_cast<double>(passed) / results_.size();
    double avg_score = std::accumulate(results_.begin(), results_.end(), 0.0,
        [](double sum, const auto& r) { return sum + r.score; }) / results_.size();

    std::cout << "\n=== Eval Results ===\n";
    std::cout << "Pass rate: " << static_cast<int>(pass_rate * 100) << "%"
              << "  Avg score: " << std::fixed << std::setprecision(2) << avg_score << "\n";

    for (const auto& r : results_) {
        if (!r.passed) {
            std::cout << "  FAIL " << r.task_id << ": " << r.reason << "\n";
        }
    }

    EXPECT_GE(pass_rate, 0.80) << "Pass rate " << pass_rate << " below 80% baseline";
}
```

```cpp
// tests/evals/scorers.hpp
#pragma once
#include <string>
#include <vector>
#include <algorithm>
#include <tuple>

namespace Scorers {

std::tuple<bool, double, std::string>
ContainsAll(const std::vector<std::string>& expected, const std::string& output) {
    if (expected.empty()) return {true, 1.0, "no keywords required"};

    std::string lower_output = output;
    std::transform(lower_output.begin(), lower_output.end(), lower_output.begin(), ::tolower);

    int matches = 0;
    for (const auto& kw : expected) {
        std::string lower_kw = kw;
        std::transform(lower_kw.begin(), lower_kw.end(), lower_kw.begin(), ::tolower);
        if (lower_output.find(lower_kw) != std::string::npos) ++matches;
    }

    double score = static_cast<double>(matches) / expected.size();
    std::string reason = std::to_string(matches) + "/" + std::to_string(expected.size()) + " keywords found";
    return {score == 1.0, score, reason};
}

std::tuple<bool, double, std::string>
ToolUsed(const std::string& expected_tool, const std::vector<Message>& messages) {
    for (const auto& msg : messages) {
        if (msg.role == "assistant") {
            for (const auto& tc : msg.tool_calls) {
                if (tc.name == expected_tool)
                    return {true, 1.0, "found " + expected_tool};
            }
        }
    }
    return {false, 0.0, "expected " + expected_tool + " not found"};
}

} // namespace Scorers
```

---

## §12.6 CMake Configuration

```cmake
# tests/CMakeLists.txt
cmake_minimum_required(VERSION 3.27)

include(FetchContent)
FetchContent_Declare(googletest
    URL https://github.com/google/googletest/archive/v1.14.0.zip
)
FetchContent_MakeAvailable(googletest)

# Unit tests
add_executable(unit_tests
    unit/tool_schema_test.cpp
    unit/bash_tool_handler_test.cpp
    unit/agent_loop_test.cpp
    unit/token_estimation_test.cpp
)
target_link_libraries(unit_tests
    agent_lib
    GTest::gtest_main
    GTest::gmock
)
gtest_discover_tests(unit_tests)

# Integration tests (separate target, requires API key)
add_executable(integration_tests
    integration/agent_e2e_test.cpp
    integration/tool_e2e_test.cpp
)
target_link_libraries(integration_tests
    agent_lib
    GTest::gtest_main
)

# Benchmark / eval (separate target, expensive)
add_executable(eval_benchmark
    evals/benchmark.cpp
)
target_link_libraries(eval_benchmark
    agent_lib
    GTest::gtest_main
)
```

---

## §12.7 CI Configuration

```yaml
# .github/workflows/test.yml
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Configure
        run: cmake -B build -DCMAKE_BUILD_TYPE=Release
      - name: Build
        run: cmake --build build --target unit_tests -j$(nproc)
      - name: Run unit tests
        run: ./build/tests/unit_tests

  integration:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    env:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    steps:
      - uses: actions/checkout@v4
      - name: Configure
        run: cmake -B build -DCMAKE_BUILD_TYPE=Release
      - name: Build
        run: cmake --build build --target integration_tests -j$(nproc)
      - name: Run integration tests
        run: ./build/tests/integration_tests
```

---

## §12.8 Mock Implementations

```cpp
// tests/mocks/mock_process_runner.hpp
#pragma once
#include <gmock/gmock.h>
#include "tools/process_runner.hpp"

class MockProcessRunner : public ProcessRunner {
public:
    MOCK_METHOD(ProcessResult, Run,
        (const std::string& command, const ProcessOptions& options),
        (override));
};
```

```cpp
// tests/mocks/mock_http_client.hpp
#pragma once
#include <gmock/gmock.h>
#include "api/http_client.hpp"

class MockHttpClient : public HttpClient {
public:
    MOCK_METHOD(HttpResponse, Post,
        (const std::string& url, const nlohmann::json& body),
        (override));
};
```

---

*[← Complete Example](./11-complete-example.md) | [Index →](./00-index.md)*
