# 02 — Core Agent Loop

The ReAct loop (arXiv:2210.03629) drives all agent behavior. Each iteration produces
a Thought, optionally an Action, and then an Observation that feeds back into the next
iteration. This document covers the complete implementation including streaming, Reflexion
wrapping, SELF-REFINE, abort/timeout, and token budget management.

---

## 1. Core Types

```cpp
// src/agent/agent_error.hpp
#pragma once
#include <string>
#include <optional>

enum class AgentErrorCode {
    LLMTimeout,
    LLMParseError,
    ToolNotFound,
    ToolPermissionDenied,
    ToolExecutionError,
    MemoryFull,
    BudgetExceeded,
    MaxIterationsReached,
    AbortRequested,
    NetworkError,
    SerializationError,
};

struct AgentError {
    AgentErrorCode code;
    std::string    message;
    std::optional<std::string> context; // step index, tool name, etc.

    static AgentError budget_exceeded(int limit) {
        return { AgentErrorCode::BudgetExceeded,
                 std::format("Token budget {} exceeded", limit), std::nullopt };
    }
};

// Convenience alias — used throughout all modules
template<typename T>
using AgentResult = std::expected<T, AgentError>;
```

```cpp
// src/agent/agent_context.hpp
#pragma once
#include <string>
#include <vector>
#include <functional>
#include <chrono>
#include <atomic>
#include <memory>
#include "agent/agent_error.hpp"
#include "tools/tool_registry.hpp"
#include "memory/working_memory.hpp"
#include "events/event_store.hpp"
#include "session/session.hpp"

namespace agent {

enum class Role { System, User, Assistant, Tool };

struct Message {
    Role        role;
    std::string content;
    std::optional<std::string> tool_call_id;  // for Role::Tool observations
    std::optional<std::string> name;          // tool name for Role::Tool
};

struct AgentConfig {
    std::string  model           = "gpt-4o";
    int          max_iterations  = 32;
    int          token_budget    = 120'000;
    int          compact_at      = 100'000;  // trigger compaction
    double       temperature     = 0.0;
    bool         streaming       = true;
    std::chrono::seconds timeout { 120 };
    std::string  llm_base_url    = "http://localhost:8080/v1";
    std::string  api_key;                    // empty = local llama-server
};

struct AgentContext {
    AgentConfig              config;
    std::vector<Message>     messages;
    std::string              system_prompt;
    tools::ToolRegistry*     tool_registry  = nullptr;
    memory::WorkingMemory*   working_memory = nullptr;
    events::EventStore*      event_store    = nullptr;
    session::Session*        session        = nullptr;

    // Abort signal — set from another thread to cancel the loop
    std::atomic<bool>        abort_requested { false };

    // Running token estimate (updated each turn)
    int                      estimated_tokens = 0;

    // Iteration counter
    int                      current_iteration = 0;

    void push(Role role, std::string content) {
        messages.push_back({ role, std::move(content) });
    }
};

} // namespace agent
```

---

## 2. ReAct Step Parsing

```cpp
// src/agent/react.hpp
#pragma once
#include <string>
#include <optional>
#include <expected>
#include "agent/agent_error.hpp"
#include "tools/tool_spec.hpp"

namespace agent::react {

struct ReActStep {
    std::string            thought;
    std::optional<tools::ToolCall> action;  // nullopt = final answer turn
    bool                   is_final_answer = false;
    std::string            final_answer;    // populated when is_final_answer
};

// Parse raw LLM text into a structured ReActStep.
// Expects format:
//   Thought: <reasoning>
//   Action: tool_name
//   Action Input: { "param": "value" }
// OR:
//   Thought: <reasoning>
//   Final Answer: <answer>
AgentResult<ReActStep> parse(const std::string& raw_response);

// Build the observation message from a tool result
std::string build_observation(const tools::ToolResult& result);

} // namespace agent::react
```

```cpp
// src/agent/react.cpp
#include "agent/react.hpp"
#include <nlohmann/json.hpp>
#include <regex>
#include <format>

namespace agent::react {

AgentResult<ReActStep> parse(const std::string& raw) {
    ReActStep step;

    // Extract Thought:
    static const std::regex thought_re(R"(Thought:\s*(.*?)(?=\nAction:|\nFinal Answer:|$))",
                                        std::regex::dotall);
    if (std::smatch m; std::regex_search(raw, m, thought_re))
        step.thought = m[1].str();

    // Check for Final Answer:
    static const std::regex final_re(R"(Final Answer:\s*(.*))");
    if (std::smatch m; std::regex_search(raw, m, final_re)) {
        step.is_final_answer = true;
        step.final_answer = m[1].str();
        return step;
    }

    // Check for Action:
    static const std::regex action_re(R"(Action:\s*(\w+))");
    static const std::regex input_re(R"(Action Input:\s*(\{.*\}|\[.*\]|".*"))",
                                      std::regex::dotall);
    std::smatch am, im;
    if (!std::regex_search(raw, am, action_re)) {
        // No action and no final answer — treat as thought-only turn
        return step;
    }

    tools::ToolCall call;
    call.name = am[1].str();
    call.call_id = std::format("call_{}", std::hash<std::string>{}(raw));

    if (std::regex_search(raw, im, input_re)) {
        try {
            call.arguments = nlohmann::json::parse(im[1].str());
        } catch (const nlohmann::json::exception& e) {
            return std::unexpected(AgentError{
                AgentErrorCode::LLMParseError,
                std::format("Failed to parse Action Input JSON: {}", e.what()),
                call.name
            });
        }
    } else {
        call.arguments = nlohmann::json::object();
    }

    step.action = std::move(call);
    return step;
}

std::string build_observation(const tools::ToolResult& result) {
    if (result.is_error) {
        return std::format("Observation: ERROR — {}", result.error_message);
    }
    // Truncate very long outputs to stay within token budget
    constexpr std::size_t MAX_OBS_CHARS = 8000;
    const auto& content = result.content;
    if (content.size() > MAX_OBS_CHARS) {
        return std::format("Observation: {}...[truncated {} chars]",
            content.substr(0, MAX_OBS_CHARS),
            content.size() - MAX_OBS_CHARS);
    }
    return "Observation: " + content;
}

} // namespace agent::react
```

---

## 3. SSE Streaming Parser

```cpp
// src/inference/sse_parser.hpp
#pragma once
#include <string>
#include <functional>
#include <generator>   // C++23
#include <httplib.h>

namespace inference {

struct SSEChunk {
    std::string delta;     // token delta text
    bool        done = false;
    std::string finish_reason;
};

// Streams completion tokens from an OpenAI-compatible SSE endpoint.
// Uses cpp-httplib's chunked transfer callback mechanism.
class SSEParser {
public:
    explicit SSEParser(std::string base_url);

    // Returns a C++23 generator — caller iterates token by token
    std::generator<SSEChunk> stream(const nlohmann::json& request_body);

    // Accumulate all chunks into a single string (convenience)
    std::string accumulate(const nlohmann::json& request_body);

private:
    std::string  base_url_;
    httplib::Client client_;
};

} // namespace inference
```

```cpp
// src/inference/sse_parser.cpp
#include "inference/sse_parser.hpp"
#include <nlohmann/json.hpp>
#include <spdlog/spdlog.h>
#include <format>

namespace inference {

SSEParser::SSEParser(std::string base_url)
    : base_url_(std::move(base_url))
    , client_(base_url_)
{
    client_.set_read_timeout(120, 0);
    client_.set_write_timeout(30, 0);
}

std::generator<SSEChunk> SSEParser::stream(const nlohmann::json& body) {
    // cpp-httplib chunked response handler — we accumulate lines here
    // and the generator suspends/resumes for each token chunk.
    // Note: std::generator is synchronous. For fully async streaming,
    // see the Boost.Cobalt variant in section 4.
    std::vector<SSEChunk> chunks;

    auto res = client_.Post("/chat/completions",
        body.dump(),
        "application/json",
        [&](const char* data, std::size_t len) -> bool {
            std::string_view sv(data, len);
            // SSE lines start with "data: "
            for (auto line : sv | std::views::split('\n')) {
                std::string_view lv(line.begin(), line.end());
                if (!lv.starts_with("data: ")) continue;
                lv.remove_prefix(6);
                if (lv == "[DONE]") {
                    chunks.push_back({ "", true, "stop" });
                    return true;
                }
                try {
                    auto j = nlohmann::json::parse(lv);
                    auto delta = j["choices"][0]["delta"];
                    SSEChunk chunk;
                    if (delta.contains("content") && !delta["content"].is_null())
                        chunk.delta = delta["content"].get<std::string>();
                    auto finish = j["choices"][0].value("finish_reason", "");
                    if (!finish.empty() && finish != "null") {
                        chunk.finish_reason = finish;
                        chunk.done = true;
                    }
                    chunks.push_back(std::move(chunk));
                } catch (...) {
                    spdlog::warn("SSE parse error on line: {}", lv);
                }
            }
            return true;
        });

    if (!res || res->status != 200) {
        SSEChunk err;
        err.done = true;
        err.finish_reason = "error";
        chunks.push_back(err);
    }

    for (auto& chunk : chunks) {
        co_yield chunk;
    }
}

std::string SSEParser::accumulate(const nlohmann::json& body) {
    std::string result;
    for (auto chunk : stream(body)) {
        result += chunk.delta;
        if (chunk.done) break;
    }
    return result;
}

} // namespace inference
```

---

## 4. Async Streaming with Boost.Cobalt

For truly non-blocking streaming (e.g., multiple agents in parallel), use
Boost.Cobalt coroutines:

```cpp
// src/inference/llm_client.hpp  — Boost.Cobalt async variant
#pragma once
#include <boost/cobalt.hpp>
#include <boost/asio.hpp>
#include <nlohmann/json.hpp>
#include <expected>
#include "agent/agent_error.hpp"

namespace inference {

namespace cobalt = boost::cobalt;
namespace asio   = boost::asio;

class AsyncLLMClient {
public:
    explicit AsyncLLMClient(asio::io_context& ioc, std::string base_url, std::string api_key);

    // Async completion — suspends until response arrives
    cobalt::task<AgentResult<std::string>>
    complete(const nlohmann::json& messages, const nlohmann::json& tools = {});

    // Async streaming — yields each SSE chunk
    cobalt::generator<std::string>
    stream(const nlohmann::json& messages, const nlohmann::json& tools = {});

    // Embedding request
    cobalt::task<AgentResult<std::vector<float>>>
    embed(const std::string& text);

private:
    asio::io_context& ioc_;
    std::string       base_url_;
    std::string       api_key_;
};

// Usage example:
// cobalt::main co_main(int, char**) {
//     asio::io_context ioc;
//     AsyncLLMClient client(ioc, "http://localhost:8080/v1", "");
//     auto result = co_await client.complete(messages);
//     ...
// }

} // namespace inference
```

---

## 5. The Main Agent Loop

```cpp
// src/agent/agent_loop.cpp
#include "agent/agent_loop.hpp"
#include "agent/react.hpp"
#include "agent/token_budget.hpp"
#include "tools/dispatcher.hpp"
#include "events/event_store.hpp"
#include "session/compaction.hpp"
#include "inference/sse_parser.hpp"
#include <nlohmann/json.hpp>
#include <spdlog/spdlog.h>
#include <chrono>
#include <format>

namespace agent {

// Build the JSON messages array for the OpenAI API
static nlohmann::json build_api_messages(const AgentContext& ctx) {
    nlohmann::json msgs = nlohmann::json::array();
    msgs.push_back({ {"role", "system"}, {"content", ctx.system_prompt} });
    for (const auto& m : ctx.messages) {
        nlohmann::json j;
        switch (m.role) {
            case Role::User:      j["role"] = "user";      break;
            case Role::Assistant: j["role"] = "assistant"; break;
            case Role::Tool:      j["role"] = "tool";      break;
            default: continue;
        }
        j["content"] = m.content;
        if (m.tool_call_id) j["tool_call_id"] = *m.tool_call_id;
        if (m.name)         j["name"]         = *m.name;
        msgs.push_back(std::move(j));
    }
    return msgs;
}

AgentResult<std::string> run_agent(AgentContext& ctx) {
    inference::SSEParser sse(ctx.config.llm_base_url);
    const auto deadline = std::chrono::steady_clock::now() + ctx.config.timeout;

    while (ctx.current_iteration < ctx.config.max_iterations) {
        // ── Abort check ─────────────────────────────────────────────
        if (ctx.abort_requested.load(std::memory_order_relaxed)) {
            return std::unexpected(AgentError{
                AgentErrorCode::AbortRequested, "Agent aborted by caller"
            });
        }

        // ── Timeout check ────────────────────────────────────────────
        if (std::chrono::steady_clock::now() > deadline) {
            return std::unexpected(AgentError{
                AgentErrorCode::LLMTimeout,
                std::format("Agent timeout after {} iterations", ctx.current_iteration)
            });
        }

        // ── Token budget / compaction ─────────────────────────────────
        ctx.estimated_tokens = token_budget::estimate(ctx.messages, ctx.system_prompt);
        if (ctx.estimated_tokens >= ctx.config.compact_at) {
            spdlog::info("[agent] compaction triggered at ~{} tokens", ctx.estimated_tokens);
            auto compact_result = session::compact_session(*ctx.session, ctx, sse);
            if (!compact_result) return std::unexpected(compact_result.error());
        }
        if (ctx.estimated_tokens >= ctx.config.token_budget) {
            return std::unexpected(AgentError::budget_exceeded(ctx.config.token_budget));
        }

        // ── Build request ────────────────────────────────────────────
        nlohmann::json request = {
            {"model",       ctx.config.model},
            {"temperature", ctx.config.temperature},
            {"stream",      ctx.config.streaming},
            {"messages",    build_api_messages(ctx)},
        };

        // Attach tool schemas if registry available
        if (ctx.tool_registry) {
            request["tools"]       = ctx.tool_registry->to_openai_schema();
            request["tool_choice"] = "auto";
        }

        spdlog::debug("[agent] iteration {} — ~{} tokens", ctx.current_iteration, ctx.estimated_tokens);

        // ── LLM call ─────────────────────────────────────────────────
        std::string raw_response = sse.accumulate(request);
        if (raw_response.empty()) {
            return std::unexpected(AgentError{
                AgentErrorCode::LLMParseError, "Empty response from LLM"
            });
        }

        ctx.push(Role::Assistant, raw_response);
        ctx.current_iteration++;

        // Emit event
        if (ctx.event_store) {
            ctx.event_store->append({
                events::EventType::AssistantMessage,
                { {"iteration", ctx.current_iteration}, {"content", raw_response} }
            });
        }

        // ── Parse ReAct step ─────────────────────────────────────────
        auto step_result = react::parse(raw_response);
        if (!step_result) return std::unexpected(step_result.error());
        auto& step = *step_result;

        // Final answer reached
        if (step.is_final_answer) {
            spdlog::info("[agent] final answer after {} iterations", ctx.current_iteration);
            if (ctx.event_store) {
                ctx.event_store->append({
                    events::EventType::FinalAnswer,
                    { {"answer", step.final_answer} }
                });
            }
            return step.final_answer;
        }

        // Thought-only (no action) — continue to next iteration
        if (!step.action) continue;

        // ── Tool dispatch ────────────────────────────────────────────
        auto dispatch_result = tools::dispatch(*ctx.tool_registry, *step.action, ctx);
        std::string observation;

        if (dispatch_result) {
            observation = react::build_observation(*dispatch_result);
            if (ctx.event_store) {
                ctx.event_store->append({
                    events::EventType::ToolUsed,
                    { {"tool", step.action->name}, {"result", dispatch_result->content} }
                });
            }
        } else {
            // Tool error — report as observation, don't abort loop
            observation = std::format("Observation: ERROR — {}",
                                      dispatch_result.error().message);
            spdlog::warn("[agent] tool error: {}", dispatch_result.error().message);
            if (ctx.event_store) {
                ctx.event_store->append({
                    events::EventType::ToolError,
                    { {"tool", step.action->name}, {"error", dispatch_result.error().message} }
                });
            }
        }

        ctx.push(Role::Tool, observation);
        ctx.push(Role::User,  "");  // empty user turn to maintain alternating pattern
    }

    return std::unexpected(AgentError{
        AgentErrorCode::MaxIterationsReached,
        std::format("Max iterations ({}) reached without final answer", ctx.config.max_iterations)
    });
}

} // namespace agent
```

---

## 6. Token Budget and Compaction

```cpp
// src/agent/token_budget.hpp
#pragma once
#include <string>
#include <vector>
#include "agent/agent_context.hpp"

namespace agent::token_budget {

// Rough heuristic: 1 token ≈ 4 chars (English prose).
// For precision, use a tiktoken-compatible C++ tokenizer.
inline int estimate(const std::vector<Message>& messages, const std::string& system) {
    std::size_t total = system.size();
    for (const auto& m : messages)
        total += m.content.size();
    // Add 4 tokens per message for role/name overhead
    total += messages.size() * 16;
    return static_cast<int>(total / 4);
}

// Check if adding `new_content` would exceed budget
inline bool would_exceed(const AgentContext& ctx, const std::string& new_content) {
    return (ctx.estimated_tokens + static_cast<int>(new_content.size() / 4))
           >= ctx.config.token_budget;
}

} // namespace agent::token_budget
```

```cpp
// src/session/compaction.cpp  — compact via summarization
#include "session/compaction.hpp"
#include "agent/token_budget.hpp"
#include <nlohmann/json.hpp>
#include <format>

namespace session {

// Compacts the earliest half of messages into a rolling summary.
// Preserves system prompt, last N turns, and injects summary as a
// special user message — matching Claude Code's compaction strategy.
AgentResult<void> compact_session(Session& sess,
                                   agent::AgentContext& ctx,
                                   inference::SSEParser& sse)
{
    const int preserve_last = 8;  // keep last 8 message turns verbatim

    if (static_cast<int>(ctx.messages.size()) <= preserve_last * 2) {
        return {};  // nothing to compact
    }

    // Extract early messages to summarize
    auto split_point = ctx.messages.size() - preserve_last * 2;
    std::string history_text;
    for (std::size_t i = 0; i < split_point; ++i) {
        const auto& m = ctx.messages[i];
        std::string role_str;
        switch (m.role) {
            case agent::Role::User:      role_str = "User";      break;
            case agent::Role::Assistant: role_str = "Assistant"; break;
            case agent::Role::Tool:      role_str = "Tool";      break;
            default: continue;
        }
        history_text += std::format("{}: {}\n\n", role_str, m.content);
    }

    // Request summary from LLM
    nlohmann::json req = {
        {"model",    ctx.config.model},
        {"stream",   false},
        {"messages", {
            {{"role", "user"}, {"content",
                "Summarize the following conversation history concisely, "
                "preserving all important facts, decisions, tool results, "
                "and code changes. This summary will replace the original messages.\n\n"
                + history_text
            }}
        }}
    };

    std::string summary = sse.accumulate(req);
    if (summary.empty()) {
        return std::unexpected(agent::AgentError{
            agent::AgentErrorCode::LLMParseError, "Empty compaction summary"
        });
    }

    // Replace early messages with summary
    std::vector<agent::Message> new_messages;
    new_messages.push_back({
        agent::Role::User,
        "[Conversation Summary]\n" + summary
    });
    for (std::size_t i = split_point; i < ctx.messages.size(); ++i)
        new_messages.push_back(ctx.messages[i]);

    ctx.messages = std::move(new_messages);
    ctx.estimated_tokens = agent::token_budget::estimate(ctx.messages, ctx.system_prompt);
    sess.compaction_count++;

    spdlog::info("[compact] reduced to ~{} tokens", ctx.estimated_tokens);
    return {};
}

} // namespace session
```

---

## 7. Reflexion Wrapper (arXiv:2303.11366)

Reflexion adds a verbal reinforcement loop on top of the base agent: after each
attempt fails (or scores below threshold), a self-reflection is generated and stored
in an episodic buffer. Subsequent attempts receive the reflections as context.

```cpp
// src/agent/reflexion.hpp
#pragma once
#include <string>
#include <vector>
#include <expected>
#include "agent/agent_context.hpp"

namespace agent {

struct ReflexionMemory {
    std::vector<std::string> reflections;   // verbal self-critiques
    int                      attempt = 0;
};

struct ReflexionConfig {
    int    max_attempts       = 3;
    double success_threshold  = 0.7;  // score above which we accept
    std::string evaluator_prompt;
};

// Wraps run_agent() with a reflection loop.
// On failure or low score, generates a reflection and retries.
AgentResult<std::string> run_with_reflexion(
    AgentContext& base_ctx,
    const ReflexionConfig& rcfg,
    std::function<double(const std::string&)> score_fn);

} // namespace agent
```

```cpp
// src/agent/reflexion.cpp
#include "agent/reflexion.hpp"
#include "agent/agent_loop.hpp"
#include "inference/sse_parser.hpp"
#include <spdlog/spdlog.h>
#include <format>

namespace agent {

static std::string generate_reflection(
    const std::string& task,
    const std::string& attempt_result,
    const std::string& failure_reason,
    inference::SSEParser& sse,
    const std::string& model)
{
    nlohmann::json req = {
        {"model",  model},
        {"stream", false},
        {"messages", {
            {{"role", "user"}, {"content", std::format(
                "You are a self-reflective agent reviewing a failed attempt.\n\n"
                "Task: {}\n\n"
                "Attempt result: {}\n\n"
                "Failure reason: {}\n\n"
                "Write a concise reflection (2-4 sentences) identifying what went wrong "
                "and how to improve in the next attempt. Be specific.",
                task, attempt_result, failure_reason
            )}}
        }}
    };
    return sse.accumulate(req);
}

AgentResult<std::string> run_with_reflexion(
    AgentContext& base_ctx,
    const ReflexionConfig& rcfg,
    std::function<double(const std::string&)> score_fn)
{
    ReflexionMemory memory;
    inference::SSEParser sse(base_ctx.config.llm_base_url);
    std::string task = base_ctx.messages.empty() ? "" : base_ctx.messages.back().content;

    for (int attempt = 0; attempt < rcfg.max_attempts; ++attempt) {
        memory.attempt = attempt;

        // Inject past reflections into context before each attempt
        AgentContext attempt_ctx = base_ctx;  // copy base config
        if (!memory.reflections.empty()) {
            std::string reflection_block = "Past attempt reflections:\n";
            for (std::size_t i = 0; i < memory.reflections.size(); ++i)
                reflection_block += std::format("  [Attempt {}] {}\n", i + 1, memory.reflections[i]);
            // Prepend to system prompt
            attempt_ctx.system_prompt = reflection_block + "\n\n" + attempt_ctx.system_prompt;
        }

        spdlog::info("[reflexion] attempt {}/{}", attempt + 1, rcfg.max_attempts);
        auto result = run_agent(attempt_ctx);

        if (!result) {
            // Hard error (timeout, abort) — don't retry
            if (result.error().code == AgentErrorCode::AbortRequested ||
                result.error().code == AgentErrorCode::LLMTimeout)
                return result;

            // Soft error — generate reflection and retry
            auto refl = generate_reflection(task, "", result.error().message, sse, base_ctx.config.model);
            memory.reflections.push_back(refl);
            spdlog::info("[reflexion] reflection: {}", refl);
            continue;
        }

        double score = score_fn(*result);
        spdlog::info("[reflexion] attempt {} score: {:.2f}", attempt + 1, score);

        if (score >= rcfg.success_threshold) {
            return *result;
        }

        // Score too low — reflect and retry
        auto failure_reason = std::format("Score {:.2f} below threshold {:.2f}",
                                           score, rcfg.success_threshold);
        auto refl = generate_reflection(task, *result, failure_reason, sse, base_ctx.config.model);
        memory.reflections.push_back(refl);
    }

    return std::unexpected(AgentError{
        AgentErrorCode::MaxIterationsReached,
        std::format("Reflexion: failed after {} attempts", rcfg.max_attempts)
    });
}

} // namespace agent
```

---

## 8. SELF-REFINE Loop

SELF-REFINE (selfrefine.info) runs a tighter generate→critique→refine cycle
within a single agent invocation rather than across full ReAct runs.

```cpp
// src/agent/self_refine.cpp
#include "agent/self_refine.hpp"
#include "inference/sse_parser.hpp"
#include <spdlog/spdlog.h>

namespace agent {

static std::string critique_output(const std::string& task,
                                    const std::string& draft,
                                    inference::SSEParser& sse,
                                    const std::string& model,
                                    const std::string& critique_prompt_template)
{
    std::string prompt = critique_prompt_template.empty()
        ? std::format("Task: {}\n\nDraft output:\n{}\n\n"
                       "Critique this output. List specific issues and improvements needed.",
                       task, draft)
        : std::format(critique_prompt_template, task, draft);

    nlohmann::json req = {
        {"model",    model},
        {"stream",   false},
        {"messages", {{{"role","user"},{"content", prompt}}}}
    };
    return sse.accumulate(req);
}

static std::string refine_output(const std::string& task,
                                  const std::string& draft,
                                  const std::string& critique,
                                  inference::SSEParser& sse,
                                  const std::string& model)
{
    nlohmann::json req = {
        {"model",  model},
        {"stream", false},
        {"messages", {{{"role","user"},{"content", std::format(
            "Task: {}\n\nPrevious output:\n{}\n\nCritique:\n{}\n\n"
            "Produce an improved version addressing the critique.",
            task, draft, critique
        )}}}}
    };
    return sse.accumulate(req);
}

AgentResult<std::string> self_refine_loop(
    const std::string& task,
    const std::string& initial_draft,
    const SelfRefineConfig& cfg,
    inference::SSEParser& sse,
    const std::string& model)
{
    std::string current = initial_draft;

    for (int i = 0; i < cfg.max_iterations; ++i) {
        auto critique = critique_output(task, current, sse, model, cfg.critique_prompt);
        spdlog::debug("[self-refine] iteration {} critique: {}", i + 1, critique.substr(0, 200));

        // Check stopping condition: if critique says output is acceptable
        if (critique.find("no issues") != std::string::npos ||
            critique.find("looks good") != std::string::npos ||
            critique.find("STOP") != std::string::npos) {
            spdlog::info("[self-refine] converged at iteration {}", i + 1);
            break;
        }

        current = refine_output(task, current, critique, sse, model);
    }

    return current;
}

} // namespace agent
```

---

## 9. Abort and Timeout Handling

The `abort_requested` atomic flag can be set from any thread (e.g., a signal handler
or a supervising thread). The loop checks it at the top of each iteration:

```cpp
// From another thread or signal handler:
ctx.abort_requested.store(true, std::memory_order_relaxed);

// With a deadline enforced externally using std::jthread:
auto watchdog = std::jthread([&ctx, timeout = cfg.timeout](std::stop_token st) {
    if (!st.wait_for(timeout))
        ctx.abort_requested.store(true, std::memory_order_relaxed);
});
auto result = run_agent(ctx);
watchdog.request_stop();  // cancel watchdog if agent finished early
```

For Boost.Cobalt-based async agents, use `cobalt::timeout`:

```cpp
cobalt::task<AgentResult<std::string>> run_agent_async(AgentContext& ctx) {
    auto [res] = co_await cobalt::race(
        run_agent_coroutine(ctx),
        cobalt::sleep(ctx.config.timeout)
    );
    if (!res) {
        co_return std::unexpected(AgentError{AgentErrorCode::LLMTimeout, "Timeout"});
    }
    co_return *res;
}
```

---

## 10. OpenAI Function-Calling Mode

When using the modern OpenAI function-calling API (tools array) instead of
ReAct text parsing, the response structure differs:

```cpp
// Parse OpenAI function-call response (alternative to text-based ReAct)
AgentResult<std::vector<tools::ToolCall>> parse_function_calls(const std::string& json_response) {
    try {
        auto j = nlohmann::json::parse(json_response);
        auto& choice = j["choices"][0];
        auto& msg    = choice["message"];

        if (!msg.contains("tool_calls") || msg["tool_calls"].is_null()) {
            return std::vector<tools::ToolCall>{};
        }

        std::vector<tools::ToolCall> calls;
        for (const auto& tc : msg["tool_calls"]) {
            tools::ToolCall call;
            call.call_id  = tc["id"].get<std::string>();
            call.name     = tc["function"]["name"].get<std::string>();
            call.arguments = nlohmann::json::parse(
                tc["function"]["arguments"].get<std::string>());
            calls.push_back(std::move(call));
        }
        return calls;
    } catch (const nlohmann::json::exception& e) {
        return std::unexpected(AgentError{
            AgentErrorCode::LLMParseError,
            std::format("parse_function_calls: {}", e.what())
        });
    }
}
```

The main loop can branch on whether the LLM response is JSON (function-calling mode)
or plain text (ReAct text mode):

```cpp
bool is_json_response(const std::string& s) {
    return !s.empty() && (s.front() == '{' || s.front() == '[');
}
```
