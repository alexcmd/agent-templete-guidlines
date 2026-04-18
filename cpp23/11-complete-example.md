# 11 — Complete Example: Code Assistant Agent

This document provides a full working example wiring all subsystems together:
a code assistant agent with file tools, RAG over docs, KG for symbol relationships,
and multi-agent coordination.

---

## 1. Directory Layout

```
code-assistant/
├── CMakeLists.txt
├── config/
│   └── default.json
├── src/
│   ├── main.cpp
│   ├── setup.hpp/.cpp         # Wires all subsystems
│   ├── code_tools.cpp         # search_code, grep, write_file, run_tests
│   └── tracer.cpp             # LangSmith-compatible tracing
└── third_party/
    └── usearch/               # git submodule
```

---

## 2. CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.28)
project(CodeAssistant VERSION 0.1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# ── Experimental STL modules ───────────────────────────────────────────
set(CMAKE_EXPERIMENTAL_CXX_IMPORT_STD "0e5b6991-d74f-11ee-a958-4e2bf4d9f462")
cmake_policy(SET CMP0155 NEW)

# ── Options ────────────────────────────────────────────────────────────
option(CA_BUILD_TESTS  "Build test suite"         ON)
option(CA_ENABLE_ASAN  "AddressSanitizer"         OFF)
option(CA_ENABLE_LTO   "Link-Time Optimization"   ON)

# ── Compiler flags ─────────────────────────────────────────────────────
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    add_compile_options(-Wall -Wextra -Wpedantic -march=native -O2)
elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    add_compile_options(-Wall -Wextra -march=native -O2 -stdlib=libc++)
endif()

if(CA_ENABLE_ASAN)
    add_compile_options(-fsanitize=address,undefined -fno-omit-frame-pointer)
    add_link_options(-fsanitize=address,undefined)
endif()

if(CA_ENABLE_LTO)
    include(CheckIPOSupported)
    check_ipo_supported(RESULT lto_ok)
    if(lto_ok)
        set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
    endif()
endif()

# ── Dependencies ───────────────────────────────────────────────────────
include(FetchContent)

FetchContent_Declare(nlohmann_json
    GIT_REPOSITORY https://github.com/nlohmann/json.git
    GIT_TAG        v3.11.3 GIT_SHALLOW TRUE)
FetchContent_MakeAvailable(nlohmann_json)

FetchContent_Declare(simdjson
    GIT_REPOSITORY https://github.com/simdjson/simdjson.git
    GIT_TAG        v3.9.0 GIT_SHALLOW TRUE)
FetchContent_MakeAvailable(simdjson)

FetchContent_Declare(glaze
    GIT_REPOSITORY https://github.com/stephenberry/glaze.git
    GIT_TAG        v3.3.0 GIT_SHALLOW TRUE)
FetchContent_MakeAvailable(glaze)

FetchContent_Declare(httplib
    GIT_REPOSITORY https://github.com/yhirose/cpp-httplib.git
    GIT_TAG        v0.16.0 GIT_SHALLOW TRUE)
FetchContent_MakeAvailable(httplib)

FetchContent_Declare(usearch
    GIT_REPOSITORY https://github.com/unum-cloud/usearch.git
    GIT_TAG        v2.14.0 GIT_SHALLOW TRUE)
FetchContent_MakeAvailable(usearch)

FetchContent_Declare(sqlitecpp
    GIT_REPOSITORY https://github.com/SRombauts/SQLiteCpp.git
    GIT_TAG        3.3.2 GIT_SHALLOW TRUE)
set(SQLITECPP_RUN_CPPLINT OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(sqlitecpp)

FetchContent_Declare(spdlog
    GIT_REPOSITORY https://github.com/gabime/spdlog.git
    GIT_TAG        v1.14.0 GIT_SHALLOW TRUE)
FetchContent_MakeAvailable(spdlog)

find_package(Boost 1.87.0 REQUIRED COMPONENTS cobalt system)

# ── Agent library (see 01-architecture.md) ─────────────────────────────
add_subdirectory(../agent-lib ${CMAKE_BINARY_DIR}/agent-lib)

# ── Code assistant executable ──────────────────────────────────────────
add_executable(code_assistant
    src/main.cpp
    src/setup.cpp
    src/code_tools.cpp
    src/tracer.cpp
)

target_include_directories(code_assistant PRIVATE src)

target_link_libraries(code_assistant PRIVATE
    agent_lib
    nlohmann_json::nlohmann_json
    simdjson::simdjson
    glaze::glaze
    httplib::httplib
    usearch
    SQLiteCpp
    spdlog::spdlog
    Boost::cobalt
    Boost::system
)

# Tests
if(CA_BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()
```

---

## 3. Configuration Struct and Loading

```cpp
// src/setup.hpp
#pragma once
#include <string>
#include <memory>
#include "session/config.hpp"
#include "session/session_manager.hpp"
#include "tools/tool_registry.hpp"
#include "memory/recall_storage.hpp"
#include "memory/archival_storage.hpp"
#include "memory/amem.hpp"
#include "rag/pipeline.hpp"
#include "kg/knowledge_graph.hpp"
#include "events/event_store.hpp"
#include "events/event_bus.hpp"
#include "inference/sse_parser.hpp"
#include "agent/agent_context.hpp"
#include "agent/expel.hpp"
#include "agent/skill_library.hpp"
#include "agent/improvement_pipeline.hpp"

struct AppState {
    session::FullConfig             config;
    session::SessionManager         session_mgr;
    tools::ToolRegistry             tool_registry;
    inference::SSEParser            sse;

    // Memory
    std::unique_ptr<memory::RecallStorage>    recall;
    std::unique_ptr<memory::ArchivalStorage>  archival;
    std::unique_ptr<memory::WorkingMemory>    working_mem;
    std::unique_ptr<memory::AMem>             amem;

    // RAG
    std::unique_ptr<rag::RAGPipeline>         rag_pipeline;

    // KG
    std::unique_ptr<kg::KnowledgeGraph>       knowledge_graph;
    std::vector<kg::Community>                kg_communities;

    // Events
    std::unique_ptr<events::EventStore>       event_store;
    std::unique_ptr<events::EventBus>         event_bus;

    // Self-improvement
    std::unique_ptr<agent::ExperiencePool>    experience_pool;
    std::unique_ptr<agent::SkillLibrary>      skill_library;
    std::unique_ptr<agent::ImprovementPipeline> improvement;

    // Embedder function
    std::function<std::vector<float>(const std::string&)> embedder;

    AppState() = delete;
    explicit AppState(const std::string& config_path,
                       const std::string& workspace_root);
};

// Wire all subsystems and return initialized AppState
std::unique_ptr<AppState> setup(const std::string& config_path,
                                  const std::string& workspace_root);

// Build an AgentContext from AppState and user input
agent::AgentContext make_context(AppState& app,
                                   const std::string& user_message,
                                   const std::string& session_id = "");
```

```cpp
// src/setup.cpp
#include "setup.hpp"
#include "tools/builtin/register_all.hpp"
#include "prompts/system_prompt.hpp"
#include <filesystem>
#include <spdlog/spdlog.h>

// Embedder that calls the /v1/embeddings endpoint
static std::function<std::vector<float>(const std::string&)>
make_embedder(inference::SSEParser& sse, const session::FullConfig& cfg)
{
    return [&sse, &cfg](const std::string& text) -> std::vector<float> {
        nlohmann::json req = {
            {"model", cfg.embed_model},
            {"input", text}
        };
        // Call embeddings endpoint directly (not through SSE)
        httplib::Client cli(cfg.agent.llm_base_url);
        cli.set_default_headers({ {"Authorization", "Bearer " + cfg.openai_api_key} });
        auto res = cli.Post("/embeddings", req.dump(), "application/json");
        if (!res || res->status != 200) return std::vector<float>(cfg.embed_dims, 0.0f);
        try {
            auto j = nlohmann::json::parse(res->body);
            auto& data = j["data"][0]["embedding"];
            std::vector<float> emb;
            emb.reserve(data.size());
            for (const auto& v : data) emb.push_back(v.get<float>());
            return emb;
        } catch (...) {
            return std::vector<float>(cfg.embed_dims, 0.0f);
        }
    };
}

AppState::AppState(const std::string& config_path, const std::string& workspace_root)
    : config(session::load_config(config_path))
    , session_mgr(config.sessions_dir)
    , sse(config.agent.llm_base_url)
{
    // Validate config
    auto err = session::validate_config(config);
    if (!err.empty()) {
        spdlog::error("[setup] config error: {}", err);
        throw std::runtime_error(err);
    }

    // Create directories
    for (const auto& dir : {
        config.sessions_dir, config.events_log_dir, config.memory_dir
    }) {
        std::filesystem::create_directories(dir);
    }

    // Embedder (needs SSE initialized first)
    embedder = make_embedder(sse, config);

    // Memory
    recall   = std::make_unique<memory::RecallStorage>(config.embed_dims);
    archival = std::make_unique<memory::ArchivalStorage>(
        config.memory_dir + "/archival.db",
        config.memory_dir + "/archival.usearch",
        config.embed_dims);
    working_mem = std::make_unique<memory::WorkingMemory>(4096);
    amem = std::make_unique<memory::AMem>(*recall, embedder);

    // KG
    if (config.enable_kg) {
        knowledge_graph = std::make_unique<kg::KnowledgeGraph>();
        if (std::filesystem::exists(config.kg_db_path))
            *knowledge_graph = kg::load_from_sqlite(config.kg_db_path);
    }

    // Event system
    event_bus   = std::make_unique<events::EventBus>();
    event_store = std::make_unique<events::EventStore>(
        config.events_log_dir + "/events.jsonl");

    // Wire event store to bus
    event_store->subscribe([&bus = *event_bus](const events::Event& e) {
        bus.publish(e);
    });

    // Self-improvement
    memory::RecallStorage expel_store(config.embed_dims, 50000);
    experience_pool = std::make_unique<agent::ExperiencePool>(
        expel_store, sse, embedder);
    memory::RecallStorage skill_store(config.embed_dims, 10000);
    skill_library = std::make_unique<agent::SkillLibrary>(skill_store, embedder);
    improvement = std::make_unique<agent::ImprovementPipeline>();
    improvement->experience_pool = experience_pool.get();
    improvement->skill_library   = skill_library.get();

    // Tools
    tools::builtin::register_all(tool_registry, config.tavily_api_key);

    spdlog::info("[setup] initialized (model={}, workspace={})",
                 config.agent.model, workspace_root);
}

agent::AgentContext make_context(AppState& app,
                                   const std::string& user_message,
                                   const std::string& session_id)
{
    agent::AgentContext ctx;
    ctx.config        = app.config.agent;
    ctx.tool_registry = &app.tool_registry;
    ctx.working_memory = app.working_mem.get();
    ctx.event_store    = app.event_store.get();

    // Build session
    static auto sess = app.session_mgr.create(".", app.config.agent);
    if (!session_id.empty()) {
        auto loaded = app.session_mgr.load(session_id);
        if (loaded) sess = *loaded;
    }
    ctx.session = &sess;

    // Build system prompt
    prompts::SystemPromptBuilder builder;
    builder
        .with_intro("CodeAssist", "an expert C++23 software engineering assistant")
        .with_tools_overview(app.tool_registry.names())
        .with_react_instructions()
        .with_environment_info()
        .with_date()
        .with_workspace_context(sess.workspace_root);

    // Inject working memory if non-empty
    auto mem_render = app.working_mem->render();
    if (!mem_render.empty())
        builder.with_memory_state(mem_render);

    ctx.system_prompt = builder.build();

    // Add user message
    ctx.push(agent::Role::User, user_message);

    return ctx;
}
```

---

## 4. Code-Specific Tools

```cpp
// src/code_tools.cpp
#include "tools/tool_spec.hpp"
#include <filesystem>
#include <fstream>
#include <format>

namespace code_tools {

tools::ToolEntry make_search_code_tool() {
    tools::ToolSpec spec;
    spec.name        = "search_code";
    spec.description = "Search for code patterns in the workspace using ripgrep. "
                       "Returns matching lines with file:line context.";
    spec.permission  = tools::ToolPermission::ReadOnly;
    spec.category    = "code/search";
    spec.parallel_safe = true;
    spec.parameters  = {
        { "pattern",   "string",  "Regex pattern to search for", true },
        { "file_glob", "string",  "File glob filter (e.g. '*.cpp', '*.hpp')", false },
        { "max_results", "integer", "Maximum results to return (default 50)", false },
    };

    tools::ToolHandler handler = [](const nlohmann::json& args,
                                     const tools::ToolContext& ctx)
        -> AgentResult<tools::ToolResult>
    {
        std::string pattern   = args.at("pattern").get<std::string>();
        std::string glob      = args.value("file_glob", "");
        int         max_res   = args.value("max_results", 50);

        std::string cmd = std::format(
            "cd {} && rg --line-number --no-heading --color=never "
            "-m {} {} {}",
            ctx.workspace_root,
            max_res,
            glob.empty() ? "" : std::format("-g '{}'", glob),
            std::quoted(pattern));

        std::array<char, 8192> buf{};
        std::string output;
        FILE* p = ::popen(cmd.c_str(), "r");
        if (!p) return tools::ToolResult::error("search_code: popen failed");
        while (std::fgets(buf.data(), static_cast<int>(buf.size()), p))
            output += buf.data();
        int exit_code = ::pclose(p);

        if (exit_code == 1) return tools::ToolResult::ok("(no matches found)");
        if (exit_code != 0) return tools::ToolResult::error("rg error: " + output);
        return tools::ToolResult::ok(output);
    };

    return { std::move(spec), std::move(handler) };
}

tools::ToolEntry make_write_file_tool() {
    tools::ToolSpec spec;
    spec.name        = "write_file";
    spec.description = "Write content to a file in the workspace. "
                       "Creates parent directories if needed. Use with care.";
    spec.permission  = tools::ToolPermission::ReadWrite;
    spec.category    = "filesystem/write";
    spec.parallel_safe = false;
    spec.parameters  = {
        { "path",    "string", "File path relative to workspace root", true },
        { "content", "string", "File content to write", true },
        { "append",  "boolean", "Append to existing file (default false)", false },
    };

    tools::ToolHandler handler = [](const nlohmann::json& args,
                                     const tools::ToolContext& ctx)
        -> AgentResult<tools::ToolResult>
    {
        std::filesystem::path rel = args.at("path").get<std::string>();
        std::filesystem::path abs =
            std::filesystem::weakly_canonical(
                std::filesystem::path(ctx.workspace_root) / rel);

        // Path traversal guard
        auto root = std::filesystem::weakly_canonical(ctx.workspace_root);
        if (!abs.string().starts_with(root.string())) {
            return std::unexpected(agent::AgentError{
                agent::AgentErrorCode::ToolPermissionDenied,
                "write_file: path escapes workspace root",
                "write_file"
            });
        }

        std::filesystem::create_directories(abs.parent_path());

        bool append = args.value("append", false);
        std::ofstream f(abs, append ? std::ios::app : std::ios::trunc);
        if (!f) return tools::ToolResult::error("Cannot open file for writing");

        std::string content = args.at("content").get<std::string>();
        f << content;

        return tools::ToolResult::ok(
            std::format("Written {} bytes to {}", content.size(), abs.string()),
            { {"bytes_written", content.size()}, {"path", abs.string()} });
    };

    return { std::move(spec), std::move(handler) };
}

tools::ToolEntry make_run_tests_tool() {
    tools::ToolSpec spec;
    spec.name        = "run_tests";
    spec.description = "Run the project's test suite using ctest or a specified test binary.";
    spec.permission  = tools::ToolPermission::Execute;
    spec.category    = "code/test";
    spec.parallel_safe = false;
    spec.parameters  = {
        { "filter",       "string",  "Test filter pattern (e.g. 'MyTest*')", false },
        { "build_first",  "boolean", "Run cmake --build before tests (default false)", false },
        { "timeout",      "integer", "Timeout in seconds (default 60)", false },
    };

    tools::ToolHandler handler = [](const nlohmann::json& args,
                                     const tools::ToolContext& ctx)
        -> AgentResult<tools::ToolResult>
    {
        bool build_first = args.value("build_first", false);
        std::string filter = args.value("filter", "");
        int timeout = args.value("timeout", 60);

        std::string cmd;
        if (build_first)
            cmd += std::format("cd {} && cmake --build build -j4 && ", ctx.workspace_root);
        else
            cmd += std::format("cd {}/build && ", ctx.workspace_root);

        cmd += std::format("timeout {} ctest --output-on-failure -j4",
                            timeout);
        if (!filter.empty())
            cmd += std::format(" -R '{}'", filter);

        std::array<char, 8192> buf{};
        std::string output;
        FILE* p = ::popen(cmd.c_str(), "r");
        if (!p) return tools::ToolResult::error("run_tests: popen failed");
        while (std::fgets(buf.data(), static_cast<int>(buf.size()), p))
            output += buf.data();
        int exit_code = ::pclose(p);

        return tools::ToolResult::ok(
            output,
            { {"exit_code", exit_code}, {"passed", exit_code == 0} });
    };

    return { std::move(spec), std::move(handler) };
}

void register_code_tools(tools::ToolRegistry& registry) {
    auto search = make_search_code_tool();
    auto write  = make_write_file_tool();
    auto tests  = make_run_tests_tool();
    registry.register_tool(std::move(search.spec), std::move(search.handler));
    registry.register_tool(std::move(write.spec),  std::move(write.handler));
    registry.register_tool(std::move(tests.spec),  std::move(tests.handler));
}

} // namespace code_tools
```

---

## 5. Main Entry Point

```cpp
// src/main.cpp
#include <iostream>
#include <string>
#include <format>
#include <signal.h>
#include "setup.hpp"
#include "code_tools.hpp"
#include "agent/agent_loop.hpp"
#include "agent/reflexion.hpp"
#include "agent/improvement_pipeline.hpp"
#include "events/event_store.hpp"
#include "session/config.hpp"
#include <spdlog/spdlog.h>
#include <spdlog/sinks/stdout_color_sinks.h>

// Global abort flag for SIGINT handling
static std::atomic<bool> g_interrupt { false };

static void sigint_handler(int) {
    g_interrupt.store(true, std::memory_order_relaxed);
    std::println(stderr, "\n[Ctrl-C received — aborting agent]");
}

int main(int argc, char* argv[]) {
    // ── Logging ────────────────────────────────────────────────────────
    auto logger = spdlog::stdout_color_mt("console");
    spdlog::set_default_logger(logger);
    spdlog::set_level(spdlog::level::info);
    spdlog::set_pattern("[%H:%M:%S.%e] [%l] %v");

    // ── Args ───────────────────────────────────────────────────────────
    std::string config_path    = "config/default.json";
    std::string workspace_root = ".";
    std::string resume_session;
    bool        use_reflexion  = false;
    bool        interactive    = false;

    for (int i = 1; i < argc; ++i) {
        std::string_view arg(argv[i]);
        if (arg == "--config"    && i + 1 < argc) config_path    = argv[++i];
        if (arg == "--workspace" && i + 1 < argc) workspace_root = argv[++i];
        if (arg == "--session"   && i + 1 < argc) resume_session = argv[++i];
        if (arg == "--reflexion") use_reflexion = true;
        if (arg == "--interactive") interactive = true;
    }

    // ── Setup ──────────────────────────────────────────────────────────
    std::unique_ptr<AppState> app;
    try {
        app = setup(config_path, workspace_root);
        code_tools::register_code_tools(app->tool_registry);
    } catch (const std::exception& e) {
        spdlog::error("Setup failed: {}", e.what());
        return 1;
    }

    // ── SIGINT ─────────────────────────────────────────────────────────
    std::signal(SIGINT, sigint_handler);

    // ── Main loop ─────────────────────────────────────────────────────
    auto run_once = [&](const std::string& user_input) -> int {
        auto ctx = make_context(*app, user_input, resume_session);

        // Wire global interrupt to agent abort
        ctx.abort_requested.store(g_interrupt.load());
        auto watchdog = std::jthread([&ctx](std::stop_token st) {
            while (!st.stop_requested()) {
                if (g_interrupt.load()) {
                    ctx.abort_requested.store(true);
                    return;
                }
                std::this_thread::sleep_for(std::chrono::milliseconds(100));
            }
        });

        // Emit session started event
        app->event_store->append({
            events::EventType::SessionStarted,
            { {"user_input", user_input.substr(0, 200)},
              {"model",      ctx.config.model} }
        });

        std::println("Running agent (model={}, max_iter={})...",
                     ctx.config.model, ctx.config.max_iterations);

        agent::AgentResult<std::string> result;

        if (use_reflexion) {
            // Wrap in Reflexion for automatic retry on failure
            agent::ReflexionConfig rcfg;
            rcfg.max_attempts      = 3;
            rcfg.success_threshold = 0.7;

            result = agent::run_with_reflexion(
                ctx, rcfg,
                [](const std::string& answer) -> double {
                    // Simple heuristic: longer complete answers score higher
                    if (answer.empty()) return 0.0;
                    if (answer.size() < 50) return 0.5;
                    return 0.8;
                });
        } else {
            result = agent::run_agent(ctx);
        }

        watchdog.request_stop();

        if (result) {
            std::println("\n[Final Answer]\n{}", *result);

            // Save session
            if (auto* sess = ctx.session) {
                sess->messages = ctx.messages;
                if (auto save_r = app->session_mgr.save(*sess); !save_r)
                    spdlog::warn("Session save failed: {}", save_r.error().message);
                std::println("[Session: {}]", sess->id);
            }

            // Self-improvement pipeline (async, non-blocking in production)
            if (app->improvement && !result->empty()) {
                std::vector<std::string> steps;
                for (const auto& m : ctx.messages)
                    if (m.role == agent::Role::Assistant)
                        steps.push_back(m.content.substr(0, 500));

                auto imp_result = app->improvement->process(
                    user_input, *result, steps,
                    app->tool_registry.names(),
                    app->sse,
                    app->tool_registry,
                    tools::ToolContext{ workspace_root },
                    ctx.config.model
                );
                if (!imp_result)
                    spdlog::debug("improvement pipeline: {}", imp_result.error().message);
            }

            return 0;
        } else {
            spdlog::error("[Agent error] {}: {}",
                static_cast<int>(result.error().code),
                result.error().message);
            app->event_store->append({
                events::EventType::AgentFailed,
                { {"error", result.error().message} }
            });
            return 1;
        }
    };

    if (interactive) {
        // Interactive REPL mode
        std::string line;
        std::print("CodeAssist> ");
        while (std::getline(std::cin, line)) {
            if (line == "exit" || line == "quit") break;
            if (line.empty()) { std::print("CodeAssist> "); continue; }
            run_once(line);
            std::print("\nCodeAssist> ");
        }
        return 0;
    }

    // Single-shot mode: read from stdin or first non-flag argument
    std::string user_input;
    for (int i = 1; i < argc; ++i) {
        std::string_view arg(argv[i]);
        if (!arg.starts_with("--")) { user_input = arg; break; }
    }

    if (user_input.empty()) {
        // Read from stdin
        std::string line;
        while (std::getline(std::cin, line)) user_input += line + "\n";
    }

    if (user_input.empty()) {
        std::println(stderr, "Usage: {} [--config path] [--workspace path] "
                            "[--session id] [--reflexion] [--interactive] "
                            "[\"user message\"]", argv[0]);
        return 1;
    }

    return run_once(user_input);
}
```

---

## 6. LangSmith-Compatible Tracing Output

```cpp
// src/tracer.cpp
#include "tracer.hpp"
#include "events/event_bus.hpp"
#include <httplib.h>
#include <spdlog/spdlog.h>
#include <format>

namespace tracer {

void setup_langsmith_tracing(
    events::EventBus& bus,
    const std::string& api_key,
    const std::string& project_name,
    const std::string& session_id)
{
    if (api_key.empty()) {
        spdlog::debug("[tracer] LangSmith API key not set, tracing disabled");
        return;
    }

    struct TraceState {
        std::string run_id;
        std::string project;
        httplib::Client client { "https://api.smith.langchain.com" };
        std::string api_key;
    };

    auto state = std::make_shared<TraceState>();
    state->run_id  = session_id;
    state->project = project_name;
    state->api_key = api_key;
    state->client.set_default_headers({
        {"x-api-key", api_key},
        {"Content-Type", "application/json"}
    });

    // Subscribe to relevant events
    bus.subscribe(events::EventType::AgentStarted,
        [state](const events::Event& e) {
            nlohmann::json body = {
                {"id",           state->run_id},
                {"name",         "agent_run"},
                {"run_type",     "chain"},
                {"start_time",   e.timestamp},
                {"project_name", state->project},
                {"inputs",       e.payload},
            };
            state->client.Post("/runs", body.dump(), "application/json");
        });

    bus.subscribe(events::EventType::ToolUsed,
        [state](const events::Event& e) {
            nlohmann::json body = {
                {"id",           std::format("{}_{}", state->run_id, e.sequence)},
                {"parent_run_id", state->run_id},
                {"name",         e.payload.value("tool", "unknown")},
                {"run_type",     "tool"},
                {"start_time",   e.timestamp},
                {"end_time",     e.timestamp},
                {"inputs",       e.payload.value("args", nlohmann::json{})},
                {"outputs",      { {"result", e.payload.value("result", "")} }},
            };
            state->client.Post("/runs", body.dump(), "application/json");
        });

    bus.subscribe(events::EventType::FinalAnswer,
        [state](const events::Event& e) {
            nlohmann::json body = {
                {"id",       state->run_id},
                {"end_time", e.timestamp},
                {"outputs",  { {"answer", e.payload.value("answer", "")} }},
            };
            state->client.Patch(
                std::format("/runs/{}", state->run_id),
                body.dump(), "application/json");
        });

    spdlog::info("[tracer] LangSmith tracing enabled (project={})", project_name);
}

} // namespace tracer
```

---

## 7. Build and Run Instructions

### Prerequisites

```bash
# Ubuntu / Debian
sudo apt-get install -y \
    build-essential cmake ninja-build \
    libboost-all-dev    # Boost 1.87+ with Cobalt
    libssl-dev          # for cpp-httplib HTTPS
    libsqlite3-dev

# GCC 14 (for full C++23 support)
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt-get install -y gcc-14 g++-14
export CXX=g++-14
export CC=gcc-14

# OR: Clang 17+
sudo apt-get install -y clang-17 libc++-17-dev libc++abi-17-dev
export CXX=clang++-17
export CC=clang-17
```

### Build

```bash
# Clone
git clone https://github.com/your-org/code-assistant
cd code-assistant

# Configure
cmake -B build -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCA_ENABLE_LTO=ON \
    -DCA_BUILD_TESTS=ON

# Build (parallel)
cmake --build build -j$(nproc)

# Run tests
cd build && ctest --output-on-failure
```

### Running with llama-server (local inference)

```bash
# Start llama-server with an OpenAI-compatible API
llama-server \
    --model models/llama-3.1-8b-instruct.Q4_K_M.gguf \
    --host 127.0.0.1 \
    --port 8080 \
    --n-gpu-layers 35 \
    --ctx-size 65536

# Configure to use local server
export AGENT_LLM_URL="http://127.0.0.1:8080/v1"
export AGENT_MODEL="llama-3.1-8b-instruct"

# Run
./build/code_assistant \
    --workspace /path/to/your/project \
    "Add error handling to all functions that return std::expected"
```

### Running with OpenAI

```bash
export OPENAI_API_KEY="sk-..."
./build/code_assistant \
    --workspace /path/to/your/project \
    --config config/default.json \
    "Review the CMakeLists.txt and suggest improvements for C++23"
```

### Interactive mode

```bash
./build/code_assistant --workspace . --interactive
CodeAssist> Explain the KnowledgeGraph class
[Final Answer]
The KnowledgeGraph class...

CodeAssist> Add a method to serialize it to GraphML format
...

CodeAssist> exit
```

### Resume a session

```bash
# The session ID is printed after each run
./build/code_assistant \
    --session 195abc12_deadbeef \
    --workspace . \
    "Continue where we left off"
```

---

## 8. Event Log Inspection

```bash
# View all events for a session
cat events/events.jsonl | jq 'select(.type_name == "ToolUsed")' | head -20

# Count tool calls by type
cat events/events.jsonl | jq -r '.payload.tool // empty' | sort | uniq -c | sort -rn

# Find the final answer
cat events/events.jsonl | jq 'select(.type_name == "FinalAnswer") | .payload.answer'

# Replay and reconstruct state at sequence 42
# (Use the C++ replay API or jq slice)
cat events/events.jsonl | jq 'select(.sequence <= 42)'
```

---

## 9. Testing Strategy

```cpp
// tests/unit/test_react.cpp
#include <gtest/gtest.h>
#include "agent/react.hpp"

TEST(ReactParser, ParsesThoughtAndAction) {
    std::string raw = R"(
Thought: I need to list the files first.
Action: bash
Action Input: {"command": "ls -la"}
)";
    auto result = agent::react::parse(raw);
    ASSERT_TRUE(result.has_value());
    EXPECT_EQ(result->thought, "I need to list the files first.");
    ASSERT_TRUE(result->action.has_value());
    EXPECT_EQ(result->action->name, "bash");
    EXPECT_EQ(result->action->arguments["command"], "ls -la");
    EXPECT_FALSE(result->is_final_answer);
}

TEST(ReactParser, ParsesFinalAnswer) {
    std::string raw = R"(
Thought: I have the answer now.
Final Answer: The directory contains 3 files.
)";
    auto result = agent::react::parse(raw);
    ASSERT_TRUE(result.has_value());
    EXPECT_TRUE(result->is_final_answer);
    EXPECT_EQ(result->final_answer, "The directory contains 3 files.");
}

TEST(ReactParser, HandlesMalformedActionInput) {
    std::string raw = R"(
Thought: Trying invalid JSON.
Action: bash
Action Input: {not valid json}
)";
    auto result = agent::react::parse(raw);
    EXPECT_FALSE(result.has_value());
    EXPECT_EQ(result.error().code, agent::AgentErrorCode::LLMParseError);
}
```

```cpp
// tests/unit/test_tool_registry.cpp
#include <gtest/gtest.h>
#include "tools/tool_registry.hpp"

TEST(ToolRegistry, LookupFound) {
    tools::ToolRegistry reg;
    tools::ToolSpec spec;
    spec.name = "test_tool";
    spec.description = "A test tool";
    reg.register_tool(spec, [](const nlohmann::json&, const tools::ToolContext&) {
        return tools::ToolResult::ok("ok");
    });

    EXPECT_NE(reg.find("test_tool"), nullptr);
    EXPECT_EQ(reg.find("nonexistent"), nullptr);
}

TEST(ToolRegistry, CategoryFilter) {
    tools::ToolRegistry reg;
    for (auto name : {"fs/read", "fs/write", "web/search"}) {
        tools::ToolSpec s; s.name = name; s.category = std::string(name).substr(0, 2);
        reg.register_tool(s, [](auto&, auto&){ return tools::ToolResult::ok(""); });
    }
    auto fs_tools = reg.by_category("fs");
    EXPECT_EQ(fs_tools.size(), 2u);
}
```

```cpp
// tests/integration/test_agent_loop.cpp
// Requires a running llama-server or mock server
#include <gtest/gtest.h>
#include "setup.hpp"
#include "agent/agent_loop.hpp"

TEST(AgentLoop, CompletesSingleStepTask) {
    // This test requires AGENT_LLM_URL to be set
    const char* url = std::getenv("AGENT_LLM_URL");
    if (!url) GTEST_SKIP() << "AGENT_LLM_URL not set";

    auto app = setup("config/test.json", "/tmp/test_workspace");
    auto ctx = make_context(*app, "What is 2 + 2?");
    ctx.config.max_iterations = 5;

    auto result = agent::run_agent(ctx);
    ASSERT_TRUE(result.has_value()) << result.error().message;
    EXPECT_FALSE(result->empty());
}
```

---

## 10. Performance Notes

- **Tool dispatch**: `std::flat_map` lookup is O(log N). For 50 tools, this is ~6 comparisons.
- **Streaming**: `std::generator<SSEChunk>` allocates one coroutine frame — negligible overhead.
- **USearch HNSW**: 10M 1536-dim vectors, ~99% recall@10, ~500μs per query on modern hardware.
- **Event store**: JSONL append is O(1). Load-all is O(N·parse). Keep hot-path reads in MaterializedView.
- **Token estimation**: The 1-token/4-chars heuristic has ~15% error. Use tiktoken C++ bindings for precision.
- **Multi-agent parallelism**: Boost.Cobalt `gather` runs N tool calls concurrently with zero thread overhead — all on the io_context thread pool.
- **Compaction**: Triggers at `compact_at` tokens (default 100K). The summarization LLM call is ~$0.01 and takes ~2s. Enable async compaction for latency-sensitive paths.
