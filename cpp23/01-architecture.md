# 01 — System Architecture

This document describes the full module structure, layer responsibilities, C++23-specific
design choices, and build configuration for a production LLM agent system.

---

## 1. High-Level Layer Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Application Layer                            │
│   main.cpp  ·  config loader  ·  CLI / API gateway  ·  tracing      │
└────────────────────────────┬────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────────┐
│                        Agent Orchestrator                            │
│   AgentContext  ·  ReAct loop  ·  Reflexion wrapper  ·  SELF-REFINE │
│   token budget  ·  abort/timeout  ·  turn management                │
└──────┬─────────────────────┬────────────────────┬───────────────────┘
       │                     │                    │
┌──────▼───────┐   ┌─────────▼────────┐  ┌───────▼──────────────────┐
│  Tool System │   │  Memory System   │  │    Multi-Agent Bus        │
│  ToolRegistry│   │  WorkingMemory   │  │  ActorRuntime  PubSub     │
│  ToolSpec    │   │  RecallStorage   │  │  SupervisorAgent          │
│  Dispatcher  │   │  ArchivalStorage │  │  MessageQueue             │
│  Hook runner │   │  A-MEM NoteGraph │  └──────────────────────────┘
└──────┬───────┘   └────────┬─────────┘
       │                    │
┌──────▼────────────────────▼─────────────────────────────────────────┐
│                      RAG / KG Pipeline                               │
│  QueryRewriter  HybridRetriever  Reranker  ContextCompressor         │
│  KnowledgeGraph  HippoRAG  LightRAG  GraphRAG  PathRAG              │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                      Persistence Layer                               │
│  EventStore (JSONL)  ·  SQLite (sessions, KG)  ·  USearch (vectors) │
│  SessionManager  ·  CQRS materialized view  ·  glaze serialization  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                      Inference Backend                               │
│  openai-cpp client  ·  llama-server (OpenAI-compat)  ·  SSE parser  │
│  Boost.Cobalt async  ·  cpp-httplib  ·  token streaming             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Module Structure

```
src/
├── agent/
│   ├── agent_context.hpp      # AgentContext: messages, memory handle, config
│   ├── agent_loop.hpp/.cpp    # run_agent(): ReAct iteration driver
│   ├── react.hpp/.cpp         # parse_thought_action(), build_observation()
│   ├── reflexion.hpp/.cpp     # ReflexionWrapper: episodic buffer + critique
│   ├── self_refine.hpp/.cpp   # self_refine_loop(): generate→critique→refine
│   └── token_budget.hpp       # estimate_tokens(), compaction triggers
│
├── tools/
│   ├── tool_spec.hpp          # ToolSpec, ToolCall, ToolResult structs
│   ├── tool_registry.hpp/.cpp # ToolRegistry with std::flat_map
│   ├── dispatcher.hpp/.cpp    # parallel_dispatch(), sequential_dispatch()
│   ├── hooks.hpp              # PreToolUse, PostToolUse callbacks
│   ├── builtin/
│   │   ├── bash_tool.cpp
│   │   ├── read_file.cpp
│   │   ├── write_file.cpp
│   │   ├── web_search.cpp
│   │   └── calculator.cpp
│   └── retrieval/
│       ├── dfsdt.hpp/.cpp     # ToolBench depth-first search decision tree
│       └── anytool.hpp/.cpp   # Hierarchical tool retrieval
│
├── memory/
│   ├── working_memory.hpp     # In-context sliding window
│   ├── recall_storage.hpp/.cpp   # USearch HNSW wrapper
│   ├── archival_storage.hpp/.cpp # USearch + SQLite for long-term
│   ├── amem.hpp/.cpp          # A-MEM note graph, Ebbinghaus decay
│   └── streaming_llm.hpp      # Attention sink + KV cache management
│
├── rag/
│   ├── pipeline.hpp           # RAGPipeline: composable stage chain
│   ├── query_rewriter.hpp/.cpp
│   ├── hybrid_retriever.hpp/.cpp  # BM25 + dense vector
│   ├── reranker.hpp/.cpp      # cross-encoder reranking
│   ├── compressor.hpp/.cpp    # ContextualCompressor
│   ├── hyde.hpp/.cpp          # Hypothetical Document Embeddings
│   ├── rag_fusion.hpp/.cpp    # Multi-query + RRF
│   └── crag.hpp/.cpp          # Confidence-scored + web fallback
│
├── kg/
│   ├── knowledge_graph.hpp/.cpp   # Entity, Relation, KG structs
│   ├── triple_extractor.hpp/.cpp  # LLM-based triple extraction
│   ├── hipporag.hpp/.cpp      # Personalized PageRank retrieval
│   ├── lightrag.hpp/.cpp      # Local + global dual retrieval
│   ├── graphrag.hpp/.cpp      # Community detection + summaries
│   └── pathrag.hpp/.cpp       # Flow-based path pruning
│
├── events/
│   ├── event.hpp              # EventType enum, Event struct
│   ├── event_store.hpp/.cpp   # Append-only JSONL log
│   ├── event_bus.hpp          # Typed pub-sub with std::variant
│   ├── materialized_view.hpp  # CQRS projection from event log
│   └── hook_runner.hpp        # PreToolUse/PostToolUse lifecycle
│
├── multiagent/
│   ├── actor.hpp/.cpp         # Actor base: mailbox + run loop
│   ├── actor_runtime.hpp/.cpp # Spawn, supervise, terminate actors
│   ├── pubsub.hpp/.cpp        # Topic-based message routing
│   ├── supervisor.hpp/.cpp    # SupervisorAgent: route + aggregate
│   └── topologies.hpp         # Sequential, parallel, hierarchical
│
├── session/
│   ├── session.hpp            # Session struct, Message types
│   ├── session_manager.hpp/.cpp  # Load, save, resume
│   ├── compaction.hpp/.cpp    # estimate_tokens, compact_session
│   └── config.hpp/.cpp        # Config struct, env/JSON loading
│
├── prompts/
│   ├── system_prompt.hpp/.cpp # SystemPromptBuilder
│   ├── react_templates.hpp    # Thought/Action/Observation templates
│   ├── reflexion_prompt.hpp
│   ├── hyde_prompt.hpp
│   └── kg_prompts.hpp
│
└── inference/
    ├── llm_client.hpp/.cpp    # openai-cpp wrapper with retry
    ├── sse_parser.hpp/.cpp    # cpp-httplib SSE streaming
    └── embedder.hpp/.cpp      # Embedding endpoint wrapper
```

---

## 3. C++23 Design Rationale

### `std::expected<T, E>` for Error Propagation

Every function that can fail returns `std::expected<T, AgentError>` instead of
throwing exceptions. This is the single most impactful C++23 feature for agent code
because agent loops are inherently error-prone (network failures, malformed JSON,
tool timeouts) and exception-based control flow obscures the happy path.

```cpp
// Monadic composition — reads left-to-right, no try/catch noise
auto result = llm_client.complete(prompt)
    .and_then(parse_react_response)
    .and_then([&](const ReActStep& step) { return dispatch_tool(step.action); })
    .transform([](const ToolResult& r) { return build_observation(r); })
    .or_else([](const AgentError& e) -> std::expected<std::string, AgentError> {
        log_error(e);
        return std::unexpected(e);
    });
```

Error categories map cleanly to an enum:

```cpp
enum class AgentErrorCode {
    LLMTimeout, LLMParseError, ToolNotFound, ToolPermissionDenied,
    ToolExecutionError, MemoryFull, VectorSearchFailed, KGExtractionFailed,
    SerializationError, NetworkError, BudgetExceeded
};

struct AgentError {
    AgentErrorCode code;
    std::string message;
    std::optional<std::string> context; // tool name, step index, etc.
};
```

### `std::flat_map<K, V>` for the Tool Registry

`std::flat_map` stores keys and values in two contiguous sorted arrays. For a
tool registry with 20–200 entries, this is 2–4x faster than `std::map` due to
cache locality during lookup. The sorted invariant also enables binary search
and prefix-range iteration (useful for hierarchical tool categories).

```cpp
// Tool registry — sorted array layout, cache-friendly lookup
std::flat_map<std::string, ToolEntry> registry_;
// vs std::map: pointer-chasing red-black tree nodes scattered in heap
```

### `std::generator<T>` for Token Streaming

C++23 `std::generator<T>` is the natural type for an SSE token stream:
the coroutine suspends after each token chunk, yielding control back to
the caller without blocking a thread.

```cpp
std::generator<std::string> stream_completion(const CompletionRequest& req) {
    for (auto& chunk : sse_parser_.stream(req)) {
        co_yield chunk.delta;
    }
}
```

### `std::print` / `std::println`

All diagnostic output uses `std::println` rather than `std::cout <<` chains.
This avoids the interleaving issues of multiple `<<` operations in multi-threaded
contexts (each `std::println` call is a single formatted write).

### `import std;` Modules

The codebase uses `import std;` as a precompiled module for the full standard library,
cutting rebuild times on headers-only changes by 60–80% in practice.

---

## 4. CMake Project Structure

```
agent-system/
├── CMakeLists.txt              # Root: options, subdirs, install
├── cmake/
│   ├── FetchDependencies.cmake # FetchContent declarations
│   ├── CompilerFlags.cmake     # C++23 flags, sanitizers, LTO
│   └── Modules/
│       └── FindUSearch.cmake
├── src/
│   └── CMakeLists.txt          # agent_lib static library
├── app/
│   └── CMakeLists.txt          # agent_app executable
└── tests/
    └── CMakeLists.txt          # GTest/Catch2 suites
```

---

## 5. Root CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.28)
project(AgentSystem VERSION 0.1.0 LANGUAGES CXX)

# ── C++23 ──────────────────────────────────────────────────────────────
set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Experimental STL modules support (GCC 14+ / Clang 17+)
set(CMAKE_EXPERIMENTAL_CXX_IMPORT_STD "0e5b6991-d74f-11ee-a958-4e2bf4d9f462")
cmake_policy(SET CMP0155 NEW)

# ── Options ────────────────────────────────────────────────────────────
option(AGENT_USE_FAISS   "Use FAISS instead of USearch"       OFF)
option(AGENT_ENABLE_ASAN "Enable AddressSanitizer"            OFF)
option(AGENT_BUILD_TESTS "Build test suite"                   ON)
option(AGENT_ENABLE_LTO  "Enable Link-Time Optimization"      ON)

# ── Compiler flags ─────────────────────────────────────────────────────
include(cmake/CompilerFlags.cmake)

# ── Dependencies ───────────────────────────────────────────────────────
include(cmake/FetchDependencies.cmake)

# ── Subdirectories ─────────────────────────────────────────────────────
add_subdirectory(src)
add_subdirectory(app)
if(AGENT_BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()
```

---

## 6. FetchDependencies.cmake

```cmake
include(FetchContent)

# nlohmann/json ─────────────────────────────────────────────────────────
FetchContent_Declare(nlohmann_json
    GIT_REPOSITORY https://github.com/nlohmann/json.git
    GIT_TAG        v3.11.3
    GIT_SHALLOW    TRUE)
FetchContent_MakeAvailable(nlohmann_json)

# simdjson ──────────────────────────────────────────────────────────────
FetchContent_Declare(simdjson
    GIT_REPOSITORY https://github.com/simdjson/simdjson.git
    GIT_TAG        v3.9.0
    GIT_SHALLOW    TRUE)
FetchContent_MakeAvailable(simdjson)

# glaze ─────────────────────────────────────────────────────────────────
FetchContent_Declare(glaze
    GIT_REPOSITORY https://github.com/stephenberry/glaze.git
    GIT_TAG        v3.3.0
    GIT_SHALLOW    TRUE)
FetchContent_MakeAvailable(glaze)

# cpp-httplib ───────────────────────────────────────────────────────────
FetchContent_Declare(httplib
    GIT_REPOSITORY https://github.com/yhirose/cpp-httplib.git
    GIT_TAG        v0.16.0
    GIT_SHALLOW    TRUE)
FetchContent_MakeAvailable(httplib)

# openai-cpp (olrea) ────────────────────────────────────────────────────
FetchContent_Declare(openai_cpp
    GIT_REPOSITORY https://github.com/olrea/openai-cpp.git
    GIT_TAG        v1.4.0
    GIT_SHALLOW    TRUE)
FetchContent_MakeAvailable(openai_cpp)

# USearch ───────────────────────────────────────────────────────────────
FetchContent_Declare(usearch
    GIT_REPOSITORY https://github.com/unum-cloud/usearch.git
    GIT_TAG        v2.14.0
    GIT_SHALLOW    TRUE)
FetchContent_MakeAvailable(usearch)

# SQLiteCpp ─────────────────────────────────────────────────────────────
FetchContent_Declare(sqlitecpp
    GIT_REPOSITORY https://github.com/SRombauts/SQLiteCpp.git
    GIT_TAG        3.3.2
    GIT_SHALLOW    TRUE)
set(SQLITECPP_RUN_CPPLINT OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(sqlitecpp)

# spdlog ────────────────────────────────────────────────────────────────
FetchContent_Declare(spdlog
    GIT_REPOSITORY https://github.com/gabime/spdlog.git
    GIT_TAG        v1.14.0
    GIT_SHALLOW    TRUE)
FetchContent_MakeAvailable(spdlog)

# Boost (Cobalt + Asio) — expects system install or vcpkg ───────────────
find_package(Boost 1.87.0 REQUIRED COMPONENTS cobalt system)
```

---

## 7. src/CMakeLists.txt

```cmake
add_library(agent_lib STATIC
    agent/agent_loop.cpp
    agent/react.cpp
    agent/reflexion.cpp
    agent/self_refine.cpp
    tools/tool_registry.cpp
    tools/dispatcher.cpp
    tools/builtin/bash_tool.cpp
    tools/builtin/read_file.cpp
    tools/builtin/write_file.cpp
    tools/builtin/web_search.cpp
    memory/recall_storage.cpp
    memory/archival_storage.cpp
    memory/amem.cpp
    rag/pipeline.cpp
    rag/query_rewriter.cpp
    rag/hybrid_retriever.cpp
    rag/reranker.cpp
    rag/hyde.cpp
    rag/rag_fusion.cpp
    rag/crag.cpp
    kg/knowledge_graph.cpp
    kg/triple_extractor.cpp
    kg/hipporag.cpp
    kg/lightrag.cpp
    kg/graphrag.cpp
    events/event_store.cpp
    events/event_bus.cpp
    multiagent/actor.cpp
    multiagent/actor_runtime.cpp
    multiagent/pubsub.cpp
    session/session_manager.cpp
    session/compaction.cpp
    session/config.cpp
    prompts/system_prompt.cpp
    inference/llm_client.cpp
    inference/sse_parser.cpp
    inference/embedder.cpp
)

target_include_directories(agent_lib PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})

target_link_libraries(agent_lib PUBLIC
    nlohmann_json::nlohmann_json
    simdjson::simdjson
    glaze::glaze
    httplib::httplib
    openai_cpp
    usearch
    SQLiteCpp
    spdlog::spdlog
    Boost::cobalt
    Boost::system
)

# C++23 STL modules (experimental; gate behind detection)
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU" AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL "14")
    target_compile_options(agent_lib PUBLIC -fmodules-ts)
endif()
```

---

## 8. CompilerFlags.cmake

```cmake
# Compiler detection
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    set(BASE_FLAGS -Wall -Wextra -Wpedantic -Wconversion -Wshadow
                   -fno-rtti  # disable RTTI for performance
                   -march=native)
    if(AGENT_ENABLE_ASAN)
        list(APPEND BASE_FLAGS -fsanitize=address,undefined -fno-omit-frame-pointer)
        link_libraries(-fsanitize=address,undefined)
    endif()
elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    set(BASE_FLAGS -Wall -Wextra -Wconversion -Wshadow
                   -fno-rtti -march=native
                   -stdlib=libc++)
endif()

add_compile_options(${BASE_FLAGS})

# LTO
if(AGENT_ENABLE_LTO)
    include(CheckIPOSupported)
    check_ipo_supported(RESULT lto_supported)
    if(lto_supported)
        set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
    endif()
endif()

# Release optimizations
if(CMAKE_BUILD_TYPE STREQUAL "Release")
    add_compile_options(-O3 -DNDEBUG)
endif()
```

---

## 9. Dependency Graph

```
agent_lib
├── nlohmann_json        (JSON DOM manipulation)
├── simdjson             (High-throughput JSON parsing of LLM responses)
├── glaze                (Reflection-based serialization of structs)
├── cpp-httplib          (HTTP client + SSE streaming)
├── openai-cpp           (OpenAI API typed client)
├── USearch              (HNSW vector index for recall + archival memory)
├── SQLiteCpp            (Sessions, KG persistence, experience pool)
├── spdlog               (Structured logging)
└── Boost.Cobalt + Asio  (Async coroutine runtime for parallel tool calls)
    └── system
```

---

## 10. Data Flow: Single Agent Turn

```
User Input
    │
    ▼
session_manager.append_message(user_msg)
    │
    ▼
system_prompt_builder.build()          ← dynamic boundary for prompt caching
    │
    ▼
llm_client.complete_stream(messages)   ← openai-cpp + SSE parser (cpp-httplib)
    │
    ▼
react::parse_thought_action(response)  ← std::expected<ReActStep, AgentError>
    │
    ├── [Thought only] → append to context, continue
    │
    └── [Action] ──────────────────────────────────────────────────────────┐
                                                                            │
                                                   tool_registry.lookup()  │
                                                        │                  │
                                                   permission_check()      │
                                                        │                  │
                                                   hook_runner.pre()       │
                                                        │                  │
                                                   tool_handler.invoke()   │
                                                        │                  │
                                                   hook_runner.post()      │
                                                        │                  │
                                              build_observation(result) ◄──┘
                                                        │
                                                        ▼
                                              event_store.append(ToolUsed)
                                                        │
                                                        ▼
                                              session.append(observation)
                                                        │
                                                        ▼
                                              [next ReAct iteration]
```

---

## 11. Interface Segregation Principles

Each subsystem exposes a clean interface through a header with no implementation
leaking into the public API. Forward declarations are used aggressively to keep
compile times low.

```cpp
// agent/agent_loop.hpp — pure interface, no heavy includes
#pragma once
#include <expected>
#include <string>
#include "agent/agent_context.hpp"
#include "agent/agent_error.hpp"

namespace agent {
    // Primary entry point — runs until final answer or budget exhausted
    auto run_agent(AgentContext& ctx) -> std::expected<std::string, AgentError>;
}
```

Implementation files (`agent_loop.cpp`) include the heavy headers
(openai-cpp, nlohmann/json, tool registry) and are compiled once.
This keeps rebuild times manageable even as the codebase grows.
