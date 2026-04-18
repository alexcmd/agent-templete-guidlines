# 03 — Tool System

The tool system provides type-safe registration, permission enforcement, hook lifecycle,
parallel dispatch, and advanced retrieval patterns (AnyTool hierarchical search,
ToolBench DFSDT multi-API planning).

---

## 1. Core Types

```cpp
// src/tools/tool_spec.hpp
#pragma once
#include <string>
#include <vector>
#include <functional>
#include <expected>
#include <nlohmann/json.hpp>
#include "agent/agent_error.hpp"

namespace tools {

enum class ToolPermission {
    ReadOnly,       // reading files, searching, querying
    ReadWrite,      // writing files, modifying state
    Execute,        // running commands, code execution
    Network,        // web search, HTTP calls
    Privileged,     // system-level, unrestricted
};

struct ToolParameter {
    std::string name;
    std::string type;         // "string", "integer", "boolean", "array", "object"
    std::string description;
    bool        required = true;
    nlohmann::json enum_values;  // optional: ["a", "b", "c"]
    nlohmann::json default_value;
};

struct ToolSpec {
    std::string                  name;
    std::string                  description;
    std::vector<ToolParameter>   parameters;
    ToolPermission               permission = ToolPermission::ReadOnly;
    std::string                  category;  // for hierarchical AnyTool retrieval
    bool                         parallel_safe = true; // safe to run concurrently?

    // Generate OpenAI function-calling JSON schema
    nlohmann::json to_openai_schema() const;
};

struct ToolCall {
    std::string    name;
    nlohmann::json arguments;
    std::string    call_id;   // matches tool_call_id in response message
};

struct ToolResult {
    std::string    content;
    bool           is_error = false;
    std::string    error_message;
    nlohmann::json metadata;  // optional structured data

    static ToolResult ok(std::string content, nlohmann::json meta = {}) {
        return { std::move(content), false, "", std::move(meta) };
    }
    static ToolResult error(std::string msg) {
        return { "", true, std::move(msg), {} };
    }
};

// Type-erased tool handler
using ToolHandler = std::function<AgentResult<ToolResult>(
    const nlohmann::json& args,
    const struct ToolContext& ctx)>;

struct ToolContext {
    std::string workspace_root;
    std::string session_id;
    nlohmann::json permissions;   // policy document
    void* user_data = nullptr;    // arbitrary caller-provided context
};

struct ToolEntry {
    ToolSpec    spec;
    ToolHandler handler;
};

} // namespace tools
```

---

## 2. OpenAI Schema Generation

```cpp
// src/tools/tool_spec.cpp
#include "tools/tool_spec.hpp"

namespace tools {

nlohmann::json ToolSpec::to_openai_schema() const {
    nlohmann::json props = nlohmann::json::object();
    std::vector<std::string> required_params;

    for (const auto& p : parameters) {
        nlohmann::json prop = {{"type", p.type}, {"description", p.description}};
        if (!p.enum_values.is_null() && !p.enum_values.empty())
            prop["enum"] = p.enum_values;
        if (!p.default_value.is_null())
            prop["default"] = p.default_value;
        props[p.name] = std::move(prop);
        if (p.required) required_params.push_back(p.name);
    }

    return {
        {"type", "function"},
        {"function", {
            {"name",        name},
            {"description", description},
            {"parameters",  {
                {"type",       "object"},
                {"properties", props},
                {"required",   required_params}
            }}
        }}
    };
}

} // namespace tools
```

---

## 3. ToolRegistry with `std::flat_map`

```cpp
// src/tools/tool_registry.hpp
#pragma once
#include <flat_map>         // C++23
#include <string>
#include <vector>
#include <expected>
#include "tools/tool_spec.hpp"

namespace tools {

class ToolRegistry {
public:
    // Register a tool — overwrites if name already exists
    void register_tool(ToolSpec spec, ToolHandler handler);

    // Convenience: register with lambda
    template<typename F>
    void register_tool(ToolSpec spec, F&& fn) {
        register_tool(std::move(spec), ToolHandler(std::forward<F>(fn)));
    }

    // Lookup — O(log N) binary search on sorted flat_map
    const ToolEntry* find(const std::string& name) const noexcept;

    // All registered names
    std::vector<std::string> names() const;

    // OpenAI tools array
    nlohmann::json to_openai_schema() const;

    // Tools matching a category prefix (e.g., "filesystem/")
    std::vector<const ToolEntry*> by_category(std::string_view category) const;

    std::size_t size() const noexcept { return entries_.size(); }

private:
    // flat_map: two contiguous sorted arrays — cache-friendly lookups
    std::flat_map<std::string, ToolEntry> entries_;
};

} // namespace tools
```

```cpp
// src/tools/tool_registry.cpp
#include "tools/tool_registry.hpp"

namespace tools {

void ToolRegistry::register_tool(ToolSpec spec, ToolHandler handler) {
    std::string name = spec.name;
    entries_[std::move(name)] = ToolEntry{ std::move(spec), std::move(handler) };
}

const ToolEntry* ToolRegistry::find(const std::string& name) const noexcept {
    auto it = entries_.find(name);
    return it != entries_.end() ? &it->second : nullptr;
}

std::vector<std::string> ToolRegistry::names() const {
    std::vector<std::string> ns;
    ns.reserve(entries_.size());
    for (const auto& [k, _] : entries_) ns.push_back(k);
    return ns;
}

nlohmann::json ToolRegistry::to_openai_schema() const {
    nlohmann::json arr = nlohmann::json::array();
    for (const auto& [_, entry] : entries_)
        arr.push_back(entry.spec.to_openai_schema());
    return arr;
}

std::vector<const ToolEntry*> ToolRegistry::by_category(std::string_view cat) const {
    std::vector<const ToolEntry*> result;
    for (const auto& [_, entry] : entries_)
        if (entry.spec.category.starts_with(cat))
            result.push_back(&entry);
    return result;
}

} // namespace tools
```

---

## 4. Permission Enforcement

```cpp
// src/tools/permissions.hpp
#pragma once
#include <string>
#include <vector>
#include <expected>
#include "tools/tool_spec.hpp"
#include "agent/agent_error.hpp"

namespace tools {

struct PermissionPolicy {
    // Allowed permission levels for this session
    std::vector<ToolPermission> allowed_permissions = {
        ToolPermission::ReadOnly
    };
    // Explicit tool blocklist (overrides allowed_permissions)
    std::vector<std::string> blocked_tools;
    // Path prefix whitelist for filesystem tools
    std::vector<std::string> allowed_paths;
};

inline AgentResult<void> check_permission(
    const ToolSpec& spec,
    const PermissionPolicy& policy)
{
    // Blocklist check
    for (const auto& blocked : policy.blocked_tools) {
        if (spec.name == blocked) {
            return std::unexpected(AgentError{
                AgentErrorCode::ToolPermissionDenied,
                std::format("Tool '{}' is explicitly blocked", spec.name),
                spec.name
            });
        }
    }

    // Permission level check
    bool allowed = false;
    for (auto perm : policy.allowed_permissions) {
        if (static_cast<int>(spec.permission) <= static_cast<int>(perm)) {
            allowed = true;
            break;
        }
        if (spec.permission == perm) {
            allowed = true;
            break;
        }
    }

    if (!allowed) {
        return std::unexpected(AgentError{
            AgentErrorCode::ToolPermissionDenied,
            std::format("Tool '{}' requires permission level {} which is not granted",
                        spec.name, static_cast<int>(spec.permission)),
            spec.name
        });
    }

    return {};
}

} // namespace tools
```

---

## 5. Hook System

Hooks allow intercepting tool invocations for logging, rate limiting, caching,
and telemetry without modifying the tool implementations themselves.

```cpp
// src/tools/hooks.hpp
#pragma once
#include <functional>
#include <string>
#include <optional>
#include "tools/tool_spec.hpp"
#include "agent/agent_error.hpp"

namespace tools {

// Return std::nullopt to allow the tool to proceed;
// return ToolResult to short-circuit (e.g., cache hit)
using PreToolUseHook  = std::function<std::optional<ToolResult>(
    const ToolSpec&, const nlohmann::json& args)>;

// Called after successful tool execution
using PostToolUseHook = std::function<void(
    const ToolSpec&, const nlohmann::json& args, const ToolResult&)>;

// Called after tool execution fails
using PostToolUseFailureHook = std::function<void(
    const ToolSpec&, const nlohmann::json& args, const AgentError&)>;

struct HookSet {
    std::vector<PreToolUseHook>         pre_hooks;
    std::vector<PostToolUseHook>        post_hooks;
    std::vector<PostToolUseFailureHook> failure_hooks;

    void add_pre(PreToolUseHook h)          { pre_hooks.push_back(std::move(h)); }
    void add_post(PostToolUseHook h)        { post_hooks.push_back(std::move(h)); }
    void add_failure(PostToolUseFailureHook h) { failure_hooks.push_back(std::move(h)); }

    // Built-in hook factories
    static PreToolUseHook make_logger(spdlog::logger& log) {
        return [&log](const ToolSpec& spec, const nlohmann::json& args) -> std::optional<ToolResult> {
            log.info("[tool] invoke: {} args={}", spec.name, args.dump().substr(0, 200));
            return std::nullopt;  // proceed
        };
    }

    static PostToolUseHook make_tracer(events::EventStore& store) {
        return [&store](const ToolSpec& spec, const nlohmann::json& args, const ToolResult& r) {
            store.append({ events::EventType::ToolUsed,
                           { {"tool", spec.name}, {"args", args}, {"result_len", r.content.size()} }
                         });
        };
    }
};

} // namespace tools
```

---

## 6. Tool Dispatcher

```cpp
// src/tools/dispatcher.hpp
#pragma once
#include <vector>
#include <expected>
#include <boost/cobalt.hpp>
#include "tools/tool_registry.hpp"
#include "tools/hooks.hpp"
#include "tools/permissions.hpp"
#include "agent/agent_context.hpp"

namespace tools {

namespace cobalt = boost::cobalt;

// Synchronous single-tool dispatch
AgentResult<ToolResult> dispatch(
    const ToolRegistry&     registry,
    const ToolCall&         call,
    const agent::AgentContext& ctx,
    const PermissionPolicy& policy = {},
    const HookSet*          hooks  = nullptr);

// Parallel dispatch of multiple tool calls using Boost.Cobalt gather
cobalt::task<std::vector<AgentResult<ToolResult>>> dispatch_parallel(
    const ToolRegistry&                registry,
    const std::vector<ToolCall>&       calls,
    const agent::AgentContext&         ctx,
    const PermissionPolicy&            policy = {},
    const HookSet*                     hooks  = nullptr);

} // namespace tools
```

```cpp
// src/tools/dispatcher.cpp
#include "tools/dispatcher.hpp"
#include <spdlog/spdlog.h>

namespace tools {

AgentResult<ToolResult> dispatch(
    const ToolRegistry& registry,
    const ToolCall& call,
    const agent::AgentContext& ctx,
    const PermissionPolicy& policy,
    const HookSet* hooks)
{
    // 1. Lookup
    const ToolEntry* entry = registry.find(call.name);
    if (!entry) {
        return std::unexpected(AgentError{
            AgentErrorCode::ToolNotFound,
            std::format("Tool '{}' not found in registry", call.name),
            call.name
        });
    }

    // 2. Permission check
    if (auto perm_result = check_permission(entry->spec, policy); !perm_result)
        return std::unexpected(perm_result.error());

    // 3. Pre-hooks (may short-circuit with cached result)
    if (hooks) {
        for (const auto& h : hooks->pre_hooks) {
            if (auto shortcircuit = h(entry->spec, call.arguments))
                return *shortcircuit;
        }
    }

    // 4. Build tool context
    ToolContext tctx;
    tctx.workspace_root = ctx.session ? ctx.session->workspace_root : ".";
    tctx.session_id     = ctx.session ? ctx.session->id : "";

    // 5. Invoke
    auto result = entry->handler(call.arguments, tctx);

    // 6. Post-hooks
    if (hooks) {
        if (result) {
            for (const auto& h : hooks->post_hooks)
                h(entry->spec, call.arguments, *result);
        } else {
            for (const auto& h : hooks->failure_hooks)
                h(entry->spec, call.arguments, result.error());
        }
    }

    return result;
}

cobalt::task<std::vector<AgentResult<ToolResult>>> dispatch_parallel(
    const ToolRegistry& registry,
    const std::vector<ToolCall>& calls,
    const agent::AgentContext& ctx,
    const PermissionPolicy& policy,
    const HookSet* hooks)
{
    // Only parallelize tools marked as parallel_safe
    std::vector<cobalt::task<AgentResult<ToolResult>>> tasks;
    tasks.reserve(calls.size());

    for (const auto& call : calls) {
        tasks.push_back([&]() -> cobalt::task<AgentResult<ToolResult>> {
            co_return dispatch(registry, call, ctx, policy, hooks);
        }());
    }

    // Boost.Cobalt gather — runs all concurrently, waits for all
    co_return co_await cobalt::gather(tasks.begin(), tasks.end());
}

} // namespace tools
```

---

## 7. Built-in Tools

### bash_tool

```cpp
// src/tools/builtin/bash_tool.cpp
#include "tools/tool_spec.hpp"
#include <cstdio>
#include <array>
#include <stdexcept>

namespace tools::builtin {

ToolEntry make_bash_tool() {
    ToolSpec spec;
    spec.name        = "bash";
    spec.description = "Execute a bash command in the workspace shell. "
                       "Returns stdout+stderr. Timeout: 30s.";
    spec.permission  = ToolPermission::Execute;
    spec.category    = "system/exec";
    spec.parallel_safe = false;  // shell commands are stateful
    spec.parameters  = {
        { "command", "string", "The bash command to execute", true },
        { "timeout_seconds", "integer", "Timeout in seconds (default 30)", false }
    };

    ToolHandler handler = [](const nlohmann::json& args, const ToolContext& ctx)
        -> AgentResult<ToolResult>
    {
        std::string cmd = args.at("command").get<std::string>();
        int timeout_s   = args.value("timeout_seconds", 30);

        // Restrict to workspace
        std::string safe_cmd = std::format(
            "cd {} && timeout {} bash -c {}",
            ctx.workspace_root,
            timeout_s,
            std::quoted(cmd)
        );

        std::array<char, 4096> buffer{};
        std::string output;
        FILE* pipe = ::popen(safe_cmd.c_str(), "r");
        if (!pipe) {
            return ToolResult::error("Failed to open shell pipe");
        }

        while (std::fgets(buffer.data(), static_cast<int>(buffer.size()), pipe))
            output += buffer.data();

        int exit_code = ::pclose(pipe);
        if (exit_code != 0) {
            // Non-zero exit — return as content with note, not error
            // (agent can read the error output and decide what to do)
            return ToolResult::ok(
                std::format("Exit code {}\n{}", exit_code, output),
                { {"exit_code", exit_code} }
            );
        }
        return ToolResult::ok(output, { {"exit_code", 0} });
    };

    return { std::move(spec), std::move(handler) };
}

} // namespace tools::builtin
```

### read_file

```cpp
// src/tools/builtin/read_file.cpp
#include "tools/tool_spec.hpp"
#include <fstream>
#include <filesystem>
#include <format>

namespace tools::builtin {

ToolEntry make_read_file_tool() {
    ToolSpec spec;
    spec.name        = "read_file";
    spec.description = "Read a file from the workspace. Returns file contents as text. "
                       "Supports optional line range.";
    spec.permission  = ToolPermission::ReadOnly;
    spec.category    = "filesystem/read";
    spec.parallel_safe = true;
    spec.parameters  = {
        { "path",       "string",  "File path (relative to workspace root)", true },
        { "start_line", "integer", "First line to read (1-indexed, inclusive)", false },
        { "end_line",   "integer", "Last line to read (inclusive)", false },
    };

    ToolHandler handler = [](const nlohmann::json& args, const ToolContext& ctx)
        -> AgentResult<ToolResult>
    {
        std::filesystem::path rel = args.at("path").get<std::string>();
        std::filesystem::path abs = std::filesystem::path(ctx.workspace_root) / rel;

        // Path traversal guard
        abs = std::filesystem::weakly_canonical(abs);
        auto root = std::filesystem::weakly_canonical(ctx.workspace_root);
        if (!abs.string().starts_with(root.string())) {
            return std::unexpected(AgentError{
                AgentErrorCode::ToolPermissionDenied,
                "Path escapes workspace root",
                "read_file"
            });
        }

        if (!std::filesystem::exists(abs))
            return ToolResult::error(std::format("File not found: {}", abs.string()));

        std::ifstream f(abs);
        if (!f) return ToolResult::error("Cannot open file");

        int start = args.value("start_line", 1);
        int end   = args.value("end_line", INT_MAX);

        std::string result;
        std::string line;
        int lineno = 0;
        while (std::getline(f, line)) {
            ++lineno;
            if (lineno < start) continue;
            if (lineno > end)   break;
            result += std::format("{:6}: {}\n", lineno, line);
        }

        return ToolResult::ok(result, { {"lines_read", lineno - start + 1} });
    };

    return { std::move(spec), std::move(handler) };
}

} // namespace tools::builtin
```

### web_search (via SerpAPI / Tavily)

```cpp
// src/tools/builtin/web_search.cpp
#include "tools/tool_spec.hpp"
#include <httplib.h>
#include <nlohmann/json.hpp>

namespace tools::builtin {

ToolEntry make_web_search_tool(std::string api_key, std::string provider = "tavily") {
    ToolSpec spec;
    spec.name        = "web_search";
    spec.description = "Search the web for current information. Returns top results with "
                       "titles, URLs, and snippets.";
    spec.permission  = ToolPermission::Network;
    spec.category    = "web/search";
    spec.parallel_safe = true;
    spec.parameters  = {
        { "query",       "string",  "Search query",                    true },
        { "max_results", "integer", "Number of results to return (default 5)", false },
    };

    ToolHandler handler = [key = std::move(api_key), prov = std::move(provider)]
        (const nlohmann::json& args, const ToolContext&) -> AgentResult<ToolResult>
    {
        std::string query    = args.at("query").get<std::string>();
        int max_results      = args.value("max_results", 5);

        httplib::Client cli("https://api.tavily.com");
        cli.set_connection_timeout(10, 0);

        nlohmann::json body = {
            {"api_key",          key},
            {"query",            query},
            {"max_results",      max_results},
            {"search_depth",     "basic"},
            {"include_answer",   true},
        };

        auto res = cli.Post("/search", body.dump(), "application/json");
        if (!res || res->status != 200)
            return ToolResult::error("Web search request failed");

        try {
            auto j = nlohmann::json::parse(res->body);
            std::string output;
            if (j.contains("answer") && !j["answer"].is_null())
                output += "Summary: " + j["answer"].get<std::string>() + "\n\n";
            for (const auto& r : j["results"]) {
                output += std::format("[{}] {}\n{}\n\n",
                    r["url"].get<std::string>(),
                    r["title"].get<std::string>(),
                    r["content"].get<std::string>().substr(0, 500));
            }
            return ToolResult::ok(output);
        } catch (...) {
            return ToolResult::error("Failed to parse search response");
        }
    };

    return { std::move(spec), std::move(handler) };
}

} // namespace tools::builtin
```

### calculator

```cpp
// src/tools/builtin/calculator.cpp
// Simple expression evaluator — replace with a proper library for production
#include "tools/tool_spec.hpp"
#include <cmath>

namespace tools::builtin {

ToolEntry make_calculator_tool() {
    ToolSpec spec;
    spec.name        = "calculator";
    spec.description = "Evaluate a mathematical expression. Supports +,-,*,/,^,sqrt,log,sin,cos.";
    spec.permission  = ToolPermission::ReadOnly;
    spec.category    = "math";
    spec.parallel_safe = true;
    spec.parameters  = {
        { "expression", "string", "Math expression to evaluate", true }
    };

    ToolHandler handler = [](const nlohmann::json& args, const ToolContext&)
        -> AgentResult<ToolResult>
    {
        // In production: use exprtk or muparser library
        // Here we show a minimal approach using popen to bc
        std::string expr = args.at("expression").get<std::string>();
        std::string cmd  = std::format("echo '{}' | bc -l 2>&1", expr);
        std::array<char, 256> buf{};
        FILE* p = ::popen(cmd.c_str(), "r");
        if (!p) return ToolResult::error("Calculator exec failed");
        std::string result;
        while (std::fgets(buf.data(), static_cast<int>(buf.size()), p))
            result += buf.data();
        ::pclose(p);
        // Trim trailing newline
        if (!result.empty() && result.back() == '\n') result.pop_back();
        return ToolResult::ok(result);
    };

    return { std::move(spec), std::move(handler) };
}

} // namespace tools::builtin
```

---

## 8. ToolBench DFSDT — Multi-API Planning

DFSDT (Depth-First Search Decision Tree) enables planning chains of tool calls
for complex tasks requiring multiple API interactions. Rather than calling tools
one at a time based on reactive outputs, DFSDT expands a decision tree of possible
API call sequences and selects the most promising branch.

```cpp
// src/tools/retrieval/dfsdt.hpp
#pragma once
#include <string>
#include <vector>
#include <memory>
#include "tools/tool_spec.hpp"
#include "agent/agent_error.hpp"

namespace tools::dfsdt {

struct PlanNode {
    ToolCall                        call;
    std::optional<ToolResult>       result;
    std::vector<std::unique_ptr<PlanNode>> children;
    float                           score = 0.0f;
    bool                            pruned = false;
};

struct DFSDTConfig {
    int   max_depth    = 4;
    int   branching    = 3;   // candidate actions per node
    float prune_threshold = 0.3f;
};

// Generate candidate next tool calls given current context and results so far
std::vector<ToolCall> generate_candidates(
    const std::string& task,
    const std::vector<ToolCall>& executed_so_far,
    const ToolRegistry& registry,
    inference::SSEParser& sse,
    int branching);

// Score a candidate plan node (LLM-based evaluation)
float score_node(
    const std::string& task,
    const PlanNode& node,
    inference::SSEParser& sse);

// Main DFSDT search — returns the best tool call sequence
AgentResult<std::vector<ToolCall>> plan(
    const std::string& task,
    const ToolRegistry& registry,
    const DFSDTConfig& cfg,
    inference::SSEParser& sse);

} // namespace tools::dfsdt
```

```cpp
// src/tools/retrieval/dfsdt.cpp
#include "tools/retrieval/dfsdt.hpp"
#include <format>

namespace tools::dfsdt {

std::vector<ToolCall> generate_candidates(
    const std::string& task,
    const std::vector<ToolCall>& history,
    const ToolRegistry& registry,
    inference::SSEParser& sse,
    int branching)
{
    // Build history description
    std::string hist_str;
    for (const auto& c : history)
        hist_str += std::format("  - {}({})\n", c.name, c.arguments.dump());

    // Available tools
    std::string tools_str;
    for (const auto& name : registry.names())
        if (auto* e = registry.find(name))
            tools_str += std::format("  - {}: {}\n", name, e->spec.description);

    nlohmann::json req = {
        {"model",  "gpt-4o"},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content", std::format(
            "Task: {}\n\n"
            "Available tools:\n{}\n"
            "Calls made so far:\n{}\n"
            "Suggest {} different next tool calls that would make progress.\n"
            "Respond as JSON array: [{{\"name\":\"tool\",\"arguments\":{{}}}},...]\n"
            "Each entry must be a valid tool from the list above.",
            task, tools_str, hist_str, branching
        )}}}}
    };

    std::string raw = sse.accumulate(req);
    std::vector<ToolCall> candidates;
    try {
        auto j = nlohmann::json::parse(raw);
        for (const auto& item : j) {
            ToolCall tc;
            tc.name      = item["name"].get<std::string>();
            tc.arguments = item.value("arguments", nlohmann::json::object());
            tc.call_id   = std::format("dfsdt_{}", candidates.size());
            candidates.push_back(std::move(tc));
        }
    } catch (...) { /* ignore parse errors */ }
    return candidates;
}

AgentResult<std::vector<ToolCall>> plan(
    const std::string& task,
    const ToolRegistry& registry,
    const DFSDTConfig& cfg,
    inference::SSEParser& sse)
{
    // Iterative DFS using explicit stack
    struct Frame {
        std::vector<ToolCall> path;
        int depth;
    };

    std::vector<Frame> stack = { { {}, 0 } };
    std::vector<ToolCall> best_path;
    float best_score = -1.0f;

    while (!stack.empty()) {
        auto [path, depth] = stack.back();
        stack.pop_back();

        if (depth >= cfg.max_depth) {
            // Evaluate this complete path
            // In production: execute the path and measure task success
            float score = static_cast<float>(path.size()) / cfg.max_depth;
            if (score > best_score) {
                best_score = score;
                best_path  = path;
            }
            continue;
        }

        auto candidates = generate_candidates(task, path, registry, sse, cfg.branching);
        for (auto& cand : candidates) {
            auto new_path = path;
            new_path.push_back(std::move(cand));
            stack.push_back({ std::move(new_path), depth + 1 });
        }
    }

    if (best_path.empty()) {
        return std::unexpected(AgentError{
            AgentErrorCode::ToolNotFound,
            "DFSDT: no valid plan found"
        });
    }
    return best_path;
}

} // namespace tools::dfsdt
```

---

## 9. AnyTool — Hierarchical Tool Retrieval

For registries with hundreds of tools, AnyTool uses a two-stage retrieval:
first select the category, then select the specific tool within that category.
This reduces the context size sent to the LLM.

```cpp
// src/tools/retrieval/anytool.hpp
#pragma once
#include <string>
#include <vector>
#include "tools/tool_registry.hpp"
#include "inference/sse_parser.hpp"

namespace tools::anytool {

// Stage 1: identify the category most relevant to the task
std::string select_category(
    const std::string& task,
    const std::vector<std::string>& categories,
    inference::SSEParser& sse);

// Stage 2: select specific tool within the category
AgentResult<ToolCall> select_tool(
    const std::string& task,
    const std::vector<const ToolEntry*>& candidates,
    inference::SSEParser& sse);

// Combined: category → specific tool selection
AgentResult<ToolCall> retrieve(
    const std::string& task,
    const ToolRegistry& registry,
    inference::SSEParser& sse);

} // namespace tools::anytool
```

```cpp
// src/tools/retrieval/anytool.cpp
#include "tools/retrieval/anytool.hpp"
#include <set>

namespace tools::anytool {

std::string select_category(
    const std::string& task,
    const std::vector<std::string>& categories,
    inference::SSEParser& sse)
{
    std::string cats;
    for (const auto& c : categories) cats += "  - " + c + "\n";

    nlohmann::json req = {
        {"model",  "gpt-4o"},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content",
            "Task: " + task + "\n\n"
            "Tool categories:\n" + cats + "\n"
            "Reply with only the category name that best matches the task."
        }}}}
    };
    auto raw = sse.accumulate(req);
    // Trim whitespace
    while (!raw.empty() && std::isspace(raw.back())) raw.pop_back();
    return raw;
}

AgentResult<ToolCall> retrieve(
    const std::string& task,
    const ToolRegistry& registry,
    inference::SSEParser& sse)
{
    // Collect unique categories
    std::set<std::string> cat_set;
    for (const auto& name : registry.names())
        if (auto* e = registry.find(name))
            if (!e->spec.category.empty())
                cat_set.insert(e->spec.category);

    std::vector<std::string> categories(cat_set.begin(), cat_set.end());

    // Stage 1
    auto category = select_category(task, categories, sse);

    // Stage 2
    auto candidates = registry.by_category(category);
    if (candidates.empty()) {
        // Fallback: search all tools
        candidates.clear();
        for (const auto& n : registry.names())
            if (auto* e = registry.find(n)) candidates.push_back(e);
    }

    return select_tool(task, candidates, sse);
}

AgentResult<ToolCall> select_tool(
    const std::string& task,
    const std::vector<const ToolEntry*>& candidates,
    inference::SSEParser& sse)
{
    std::string tool_list;
    for (const auto* e : candidates)
        tool_list += std::format("  - {}: {}\n", e->spec.name, e->spec.description);

    nlohmann::json req = {
        {"model",  "gpt-4o"},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content",
            "Task: " + task + "\n\n"
            "Available tools:\n" + tool_list + "\n"
            "Respond as JSON: {\"name\": \"tool_name\", \"arguments\": {...}}"
        }}}}
    };

    std::string raw = sse.accumulate(req);
    try {
        auto j = nlohmann::json::parse(raw);
        ToolCall tc;
        tc.name      = j["name"].get<std::string>();
        tc.arguments = j.value("arguments", nlohmann::json::object());
        tc.call_id   = "anytool_0";
        return tc;
    } catch (...) {
        return std::unexpected(AgentError{
            AgentErrorCode::LLMParseError,
            "AnyTool: failed to parse tool selection response"
        });
    }
}

} // namespace tools::anytool
```

---

## 10. Registering All Built-in Tools

```cpp
// src/tools/builtin/register_all.hpp
#pragma once
#include "tools/tool_registry.hpp"

namespace tools::builtin {

inline void register_all(ToolRegistry& registry,
                          const std::string& tavily_api_key = "") {
    registry.register_tool(make_bash_tool().spec,      make_bash_tool().handler);
    registry.register_tool(make_read_file_tool().spec, make_read_file_tool().handler);
    // ... avoid double-constructing in production — use ToolEntry directly:
    auto bash     = make_bash_tool();
    auto read_f   = make_read_file_tool();
    auto web      = make_web_search_tool(tavily_api_key);
    auto calc     = make_calculator_tool();

    registry.register_tool(std::move(bash.spec),   std::move(bash.handler));
    registry.register_tool(std::move(read_f.spec), std::move(read_f.handler));
    registry.register_tool(std::move(web.spec),    std::move(web.handler));
    registry.register_tool(std::move(calc.spec),   std::move(calc.handler));
}

} // namespace tools::builtin
```
