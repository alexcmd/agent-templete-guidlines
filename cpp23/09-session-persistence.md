# 09 — Session and Persistence

This document covers session management, JSONL serialization, auto-compaction,
SQLite storage, session resume, configuration loading, and permission policies.

---

## 1. Session Struct

```cpp
// src/session/session.hpp
#pragma once
#include <string>
#include <vector>
#include <chrono>
#include <optional>
#include <nlohmann/json.hpp>
#include "agent/agent_context.hpp"

namespace session {

struct CompactionMeta {
    int     compaction_count     = 0;
    int     tokens_before_compact = 0;
    int     tokens_after_compact  = 0;
    std::string last_compact_at;
    std::string summary_text;      // last generated summary
};

struct Session {
    // Identity
    std::string id;               // UUID v4 or timestamp-based
    std::string workspace_root;
    std::string created_at;       // ISO-8601
    std::string updated_at;

    // Conversation history
    std::vector<agent::Message> messages;
    std::string                  system_prompt;

    // Compaction tracking
    CompactionMeta compaction;

    // Token tracking
    int    estimated_tokens   = 0;
    int    total_tokens_used  = 0;  // cumulative across compactions

    // Agent configuration snapshot
    agent::AgentConfig config;

    // Metadata
    nlohmann::json custom_metadata;

    // Generate a new session ID
    static std::string new_id();
};

} // namespace session
```

```cpp
// src/session/session.cpp
#include "session/session.hpp"
#include <random>
#include <format>

namespace session {

std::string Session::new_id() {
    // Simple timestamp + random suffix for uniqueness
    auto now = std::chrono::system_clock::now();
    auto ts  = std::chrono::duration_cast<std::chrono::milliseconds>(
                   now.time_since_epoch()).count();

    std::mt19937 rng(std::random_device{}());
    std::uniform_int_distribution<uint32_t> dist;
    return std::format("{:x}_{:08x}", ts, dist(rng));
}

} // namespace session
```

---

## 2. JSONL Serialization with glaze

glaze (v3.3+) provides zero-boilerplate reflection-based JSON serialization in C++23:

```cpp
// src/session/session_manager.hpp
#pragma once
#include <string>
#include <vector>
#include <expected>
#include "session/session.hpp"
#include "agent/agent_error.hpp"

namespace session {

// Custom glaze metadata for Session structs
// (glaze uses ADL/reflection — no manual field registration needed in C++23)
}

// Tell glaze how to serialize agent::Message::Role enum
template<>
struct glz::meta<agent::Role> {
    static constexpr auto value = glz::enumerate(
        "system",    agent::Role::System,
        "user",      agent::Role::User,
        "assistant", agent::Role::Assistant,
        "tool",      agent::Role::Tool
    );
};

namespace session {

class SessionManager {
public:
    explicit SessionManager(std::string sessions_dir);

    // Create a new session
    Session create(const std::string& workspace_root,
                    const agent::AgentConfig& cfg);

    // Save session to disk as JSONL file
    AgentResult<void> save(const Session& sess) const;

    // Load session from file
    AgentResult<Session> load(const std::string& session_id) const;

    // Append a single message to the JSONL file (streaming write)
    AgentResult<void> append_message(const std::string& session_id,
                                      const agent::Message& msg);

    // List available sessions
    std::vector<std::string> list_sessions() const;

    // Delete session files
    void delete_session(const std::string& session_id);

private:
    std::string sessions_dir_;
    std::string session_path(const std::string& id) const;
};

} // namespace session
```

```cpp
// src/session/session_manager.cpp
#include "session/session_manager.hpp"
#include <fstream>
#include <filesystem>
#include <nlohmann/json.hpp>
#include <spdlog/spdlog.h>

namespace session {

// Serialize agent::Message to JSON
static nlohmann::json message_to_json(const agent::Message& m) {
    nlohmann::json j;
    switch (m.role) {
        case agent::Role::User:      j["role"] = "user";      break;
        case agent::Role::Assistant: j["role"] = "assistant"; break;
        case agent::Role::System:    j["role"] = "system";    break;
        case agent::Role::Tool:      j["role"] = "tool";      break;
    }
    j["content"] = m.content;
    if (m.tool_call_id) j["tool_call_id"] = *m.tool_call_id;
    if (m.name)         j["name"]         = *m.name;
    return j;
}

static agent::Message json_to_message(const nlohmann::json& j) {
    agent::Message m;
    auto role_str = j["role"].get<std::string>();
    if      (role_str == "user")      m.role = agent::Role::User;
    else if (role_str == "assistant") m.role = agent::Role::Assistant;
    else if (role_str == "system")    m.role = agent::Role::System;
    else if (role_str == "tool")      m.role = agent::Role::Tool;
    m.content = j["content"].get<std::string>();
    if (j.contains("tool_call_id")) m.tool_call_id = j["tool_call_id"].get<std::string>();
    if (j.contains("name"))         m.name         = j["name"].get<std::string>();
    return m;
}

SessionManager::SessionManager(std::string sessions_dir)
    : sessions_dir_(std::move(sessions_dir))
{
    std::filesystem::create_directories(sessions_dir_);
}

std::string SessionManager::session_path(const std::string& id) const {
    return (std::filesystem::path(sessions_dir_) / (id + ".jsonl")).string();
}

Session SessionManager::create(const std::string& workspace_root,
                                 const agent::AgentConfig& cfg)
{
    Session sess;
    sess.id              = Session::new_id();
    sess.workspace_root  = workspace_root;
    sess.config          = cfg;

    auto now = std::chrono::system_clock::now();
    sess.created_at = std::format("{:%Y-%m-%dT%H:%M:%SZ}", now);
    sess.updated_at = sess.created_at;

    spdlog::info("[session] created: {}", sess.id);
    return sess;
}

AgentResult<void> SessionManager::save(const Session& sess) const {
    std::ofstream f(session_path(sess.id));
    if (!f) {
        return std::unexpected(agent::AgentError{
            agent::AgentErrorCode::SerializationError,
            std::format("Cannot open session file: {}", sess.id)
        });
    }

    // Line 1: session metadata
    nlohmann::json meta = {
        {"type",            "session_meta"},
        {"id",              sess.id},
        {"workspace_root",  sess.workspace_root},
        {"created_at",      sess.created_at},
        {"estimated_tokens", sess.estimated_tokens},
        {"total_tokens_used", sess.total_tokens_used},
        {"compaction_count", sess.compaction.compaction_count},
        {"system_prompt",   sess.system_prompt},
        {"config",          {
            {"model",          sess.config.model},
            {"max_iterations", sess.config.max_iterations},
            {"token_budget",   sess.config.token_budget},
            {"temperature",    sess.config.temperature},
        }},
    };
    f << meta.dump() << '\n';

    // Subsequent lines: messages
    for (const auto& msg : sess.messages) {
        nlohmann::json line = message_to_json(msg);
        line["type"] = "message";
        f << line.dump() << '\n';
    }

    f.flush();
    return {};
}

AgentResult<Session> SessionManager::load(const std::string& session_id) const {
    std::ifstream f(session_path(session_id));
    if (!f) {
        return std::unexpected(agent::AgentError{
            agent::AgentErrorCode::SerializationError,
            std::format("Session not found: {}", session_id)
        });
    }

    Session sess;
    bool meta_loaded = false;
    std::string line;

    while (std::getline(f, line)) {
        if (line.empty()) continue;
        try {
            auto j = nlohmann::json::parse(line);
            std::string type = j.value("type", "message");

            if (type == "session_meta") {
                sess.id              = j["id"].get<std::string>();
                sess.workspace_root  = j["workspace_root"].get<std::string>();
                sess.created_at      = j.value("created_at", "");
                sess.system_prompt   = j.value("system_prompt", "");
                sess.estimated_tokens = j.value("estimated_tokens", 0);
                sess.total_tokens_used = j.value("total_tokens_used", 0);
                sess.compaction.compaction_count = j.value("compaction_count", 0);

                if (j.contains("config")) {
                    sess.config.model          = j["config"].value("model", "gpt-4o");
                    sess.config.max_iterations = j["config"].value("max_iterations", 32);
                    sess.config.token_budget   = j["config"].value("token_budget", 120000);
                    sess.config.temperature    = j["config"].value("temperature", 0.0);
                }
                meta_loaded = true;
            } else if (type == "message") {
                sess.messages.push_back(json_to_message(j));
            }
        } catch (const std::exception& e) {
            spdlog::warn("[session] parse error loading {}: {}", session_id, e.what());
        }
    }

    if (!meta_loaded) {
        return std::unexpected(agent::AgentError{
            agent::AgentErrorCode::SerializationError,
            "Session file missing metadata line"
        });
    }

    spdlog::info("[session] loaded: {} ({} messages)", sess.id, sess.messages.size());
    return sess;
}

std::vector<std::string> SessionManager::list_sessions() const {
    std::vector<std::string> ids;
    for (const auto& entry : std::filesystem::directory_iterator(sessions_dir_)) {
        if (entry.path().extension() == ".jsonl") {
            ids.push_back(entry.path().stem().string());
        }
    }
    return ids;
}

} // namespace session
```

---

## 3. Auto-Compaction

```cpp
// src/session/compaction.hpp
#pragma once
#include "session/session.hpp"
#include "agent/agent_context.hpp"
#include "inference/sse_parser.hpp"

namespace session {

// Estimate token count from messages
int estimate_tokens(const std::vector<agent::Message>& messages,
                     const std::string& system_prompt);

// Compact a session by summarizing early messages
AgentResult<void> compact_session(Session& sess,
                                   agent::AgentContext& ctx,
                                   inference::SSEParser& sse);

// Check for tool-use/tool-result pair integrity before compacting
// (compaction must not split an open tool call)
bool has_open_tool_call(const std::vector<agent::Message>& messages);

// Merge multiple compaction summaries into one
AgentResult<std::string> merge_summaries(
    const std::vector<std::string>& summaries,
    inference::SSEParser& sse,
    const std::string& model);

} // namespace session
```

```cpp
// src/session/compaction.cpp
#include "session/compaction.hpp"
#include <spdlog/spdlog.h>

namespace session {

int estimate_tokens(const std::vector<agent::Message>& messages,
                     const std::string& system_prompt)
{
    std::size_t total = system_prompt.size();
    for (const auto& m : messages) total += m.content.size() + 16;
    return static_cast<int>(total / 4);
}

bool has_open_tool_call(const std::vector<agent::Message>& messages) {
    // Walk backwards — look for an assistant message with a tool_call
    // that has no matching tool result after it
    for (auto it = messages.rbegin(); it != messages.rend(); ++it) {
        if (it->role == agent::Role::Assistant && it->tool_calls &&
            !it->tool_calls->empty()) {
            // Check if any subsequent message has matching tool_call_id
            auto fwd = it.base();
            bool has_result = false;
            while (fwd != messages.end()) {
                if (fwd->role == agent::Role::Tool && fwd->tool_call_id)
                    has_result = true;
                ++fwd;
            }
            if (!has_result) return true;
        }
    }
    return false;
}

AgentResult<void> compact_session(Session& sess,
                                   agent::AgentContext& ctx,
                                   inference::SSEParser& sse)
{
    if (has_open_tool_call(ctx.messages)) {
        spdlog::warn("[compact] skipped — open tool call pair");
        return {};
    }

    const int preserve_last = 8;
    if (static_cast<int>(ctx.messages.size()) <= preserve_last * 2) return {};

    auto split_point = ctx.messages.size() - preserve_last * 2;

    // Build history text for summarization
    std::string history;
    for (std::size_t i = 0; i < split_point; ++i) {
        const auto& m = ctx.messages[i];
        std::string role;
        switch (m.role) {
            case agent::Role::User:      role = "User";      break;
            case agent::Role::Assistant: role = "Assistant"; break;
            case agent::Role::Tool:      role = "Tool";      break;
            default: continue;
        }
        history += std::format("{}: {}\n\n", role, m.content.substr(0, 2000));
    }

    nlohmann::json req = {
        {"model",  ctx.config.model},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content",
            "Summarize this conversation history. Preserve:\n"
            "- All decisions made and their rationale\n"
            "- Tool outputs and their key findings\n"
            "- Code written or modified\n"
            "- Any errors encountered and how they were resolved\n"
            "- Current task state and what was accomplished\n\n"
            + history
        }}}}
    };

    std::string summary = sse.accumulate(req);
    if (summary.empty()) {
        return std::unexpected(agent::AgentError{
            agent::AgentErrorCode::LLMParseError, "Empty compaction summary"
        });
    }

    // Record compaction stats
    int tokens_before = estimate_tokens(ctx.messages, ctx.system_prompt);

    // Replace early messages with summary
    std::vector<agent::Message> new_msgs;
    new_msgs.push_back({
        agent::Role::User,
        "[Previous conversation summary]\n" + summary
    });
    for (std::size_t i = split_point; i < ctx.messages.size(); ++i)
        new_msgs.push_back(ctx.messages[i]);

    ctx.messages = std::move(new_msgs);
    sess.messages = ctx.messages;

    int tokens_after = estimate_tokens(ctx.messages, ctx.system_prompt);

    sess.compaction.compaction_count++;
    sess.compaction.tokens_before_compact = tokens_before;
    sess.compaction.tokens_after_compact  = tokens_after;
    sess.compaction.summary_text          = summary;

    auto now = std::chrono::system_clock::now();
    sess.compaction.last_compact_at = std::format("{:%Y-%m-%dT%H:%M:%SZ}", now);

    spdlog::info("[compact] {} → {} tokens (compaction #{})",
                 tokens_before, tokens_after, sess.compaction.compaction_count);
    return {};
}

AgentResult<std::string> merge_summaries(
    const std::vector<std::string>& summaries,
    inference::SSEParser& sse,
    const std::string& model)
{
    if (summaries.empty()) return "";
    if (summaries.size() == 1) return summaries[0];

    std::string combined;
    for (std::size_t i = 0; i < summaries.size(); ++i)
        combined += std::format("[Summary {}]\n{}\n\n", i + 1, summaries[i]);

    nlohmann::json req = {
        {"model",  model},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content",
            "Merge these conversation summaries into one coherent summary:\n\n" + combined
        }}}}
    };

    return sse.accumulate(req);
}

} // namespace session
```

---

## 4. Configuration Loading

```cpp
// src/session/config.hpp
#pragma once
#include <string>
#include <optional>
#include <nlohmann/json.hpp>
#include "agent/agent_context.hpp"

namespace session {

struct FullConfig {
    // LLM settings
    agent::AgentConfig agent;

    // Paths
    std::string sessions_dir   = "./sessions";
    std::string events_log_dir = "./events";
    std::string memory_dir     = "./memory";
    std::string kg_db_path     = "./kg.db";

    // Embedding
    std::string embed_model    = "text-embedding-3-small";
    int         embed_dims     = 1536;

    // API keys
    std::string openai_api_key;
    std::string tavily_api_key;
    std::string anthropic_api_key;

    // Feature flags
    bool enable_memory     = true;
    bool enable_rag        = true;
    bool enable_kg         = false;
    bool enable_multiagent = false;
    bool enable_tracing    = false;

    // Permission policy
    std::vector<std::string> allowed_tools;
    std::vector<std::string> blocked_tools;
    std::vector<std::string> allowed_paths;
    bool allow_network_tools = false;
    bool allow_execute_tools = false;
};

// Load config from:
//   1. Environment variables (highest priority)
//   2. JSON config file
//   3. Defaults (lowest priority)
FullConfig load_config(const std::string& config_file_path = "config/default.json");

// Parse environment overrides
void apply_env_overrides(FullConfig& cfg);

// Validate config — returns error message or empty string if valid
std::string validate_config(const FullConfig& cfg);

} // namespace session
```

```cpp
// src/session/config.cpp
#include "session/config.hpp"
#include <fstream>
#include <cstdlib>
#include <spdlog/spdlog.h>

namespace session {

static std::optional<std::string> getenv_s(const char* name) {
    const char* val = std::getenv(name);
    if (!val || std::strlen(val) == 0) return std::nullopt;
    return std::string(val);
}

FullConfig load_config(const std::string& config_file_path) {
    FullConfig cfg;

    // Load from JSON file if it exists
    std::ifstream f(config_file_path);
    if (f.is_open()) {
        try {
            nlohmann::json j;
            f >> j;

            // Agent settings
            if (j.contains("agent")) {
                cfg.agent.model          = j["agent"].value("model", cfg.agent.model);
                cfg.agent.max_iterations = j["agent"].value("max_iterations", cfg.agent.max_iterations);
                cfg.agent.token_budget   = j["agent"].value("token_budget", cfg.agent.token_budget);
                cfg.agent.compact_at     = j["agent"].value("compact_at", cfg.agent.compact_at);
                cfg.agent.temperature    = j["agent"].value("temperature", cfg.agent.temperature);
                cfg.agent.llm_base_url   = j["agent"].value("llm_base_url", cfg.agent.llm_base_url);
            }

            // Paths
            cfg.sessions_dir   = j.value("sessions_dir",   cfg.sessions_dir);
            cfg.events_log_dir = j.value("events_log_dir", cfg.events_log_dir);
            cfg.memory_dir     = j.value("memory_dir",     cfg.memory_dir);
            cfg.kg_db_path     = j.value("kg_db_path",     cfg.kg_db_path);

            // API keys
            cfg.openai_api_key    = j.value("openai_api_key",    "");
            cfg.tavily_api_key    = j.value("tavily_api_key",    "");
            cfg.anthropic_api_key = j.value("anthropic_api_key", "");

            // Embedding
            cfg.embed_model = j.value("embed_model", cfg.embed_model);
            cfg.embed_dims  = j.value("embed_dims",  cfg.embed_dims);

            // Features
            cfg.enable_memory     = j.value("enable_memory",     cfg.enable_memory);
            cfg.enable_rag        = j.value("enable_rag",        cfg.enable_rag);
            cfg.enable_kg         = j.value("enable_kg",         cfg.enable_kg);
            cfg.enable_multiagent = j.value("enable_multiagent", cfg.enable_multiagent);
            cfg.enable_tracing    = j.value("enable_tracing",    cfg.enable_tracing);

            // Permissions
            if (j.contains("allowed_tools"))
                for (const auto& t : j["allowed_tools"])
                    cfg.allowed_tools.push_back(t.get<std::string>());
            if (j.contains("blocked_tools"))
                for (const auto& t : j["blocked_tools"])
                    cfg.blocked_tools.push_back(t.get<std::string>());
            if (j.contains("allowed_paths"))
                for (const auto& p : j["allowed_paths"])
                    cfg.allowed_paths.push_back(p.get<std::string>());

            cfg.allow_network_tools = j.value("allow_network_tools", cfg.allow_network_tools);
            cfg.allow_execute_tools = j.value("allow_execute_tools", cfg.allow_execute_tools);

            spdlog::info("[config] loaded from {}", config_file_path);
        } catch (const std::exception& e) {
            spdlog::warn("[config] parse error in {}: {}", config_file_path, e.what());
        }
    } else {
        spdlog::debug("[config] no config file at {}, using defaults", config_file_path);
    }

    // Environment variable overrides (highest priority)
    apply_env_overrides(cfg);

    return cfg;
}

void apply_env_overrides(FullConfig& cfg) {
    if (auto v = getenv_s("OPENAI_API_KEY"))    cfg.openai_api_key    = *v;
    if (auto v = getenv_s("ANTHROPIC_API_KEY")) cfg.anthropic_api_key = *v;
    if (auto v = getenv_s("TAVILY_API_KEY"))    cfg.tavily_api_key    = *v;
    if (auto v = getenv_s("AGENT_MODEL"))       cfg.agent.model       = *v;
    if (auto v = getenv_s("AGENT_LLM_URL"))     cfg.agent.llm_base_url = *v;
    if (auto v = getenv_s("AGENT_MAX_ITER"))    cfg.agent.max_iterations = std::stoi(*v);
    if (auto v = getenv_s("AGENT_TOKEN_BUDGET")) cfg.agent.token_budget = std::stoi(*v);
    if (auto v = getenv_s("AGENT_SESSIONS_DIR")) cfg.sessions_dir = *v;
}

std::string validate_config(const FullConfig& cfg) {
    if (cfg.agent.model.empty())
        return "agent.model must not be empty";
    if (cfg.agent.max_iterations < 1)
        return "agent.max_iterations must be >= 1";
    if (cfg.agent.token_budget < 1000)
        return "agent.token_budget must be >= 1000";
    if (cfg.agent.compact_at >= cfg.agent.token_budget)
        return "agent.compact_at must be < agent.token_budget";
    return "";  // valid
}

} // namespace session
```

---

## 5. Default Config File

```json
// config/default.json
{
    "agent": {
        "model":           "gpt-4o",
        "max_iterations":  32,
        "token_budget":    128000,
        "compact_at":      100000,
        "temperature":     0.0,
        "llm_base_url":    "https://api.openai.com/v1"
    },
    "sessions_dir":   "./sessions",
    "events_log_dir": "./events",
    "memory_dir":     "./memory",
    "kg_db_path":     "./knowledge_graph.db",
    "embed_model":    "text-embedding-3-small",
    "embed_dims":     1536,
    "enable_memory":      true,
    "enable_rag":         true,
    "enable_kg":          false,
    "enable_multiagent":  false,
    "enable_tracing":     false,
    "allow_network_tools": false,
    "allow_execute_tools": false,
    "blocked_tools": [],
    "allowed_paths":  ["."]
}
```

---

## 6. Permission Policy

```cpp
// src/session/permissions.hpp
#pragma once
#include <string>
#include <vector>
#include "tools/permissions.hpp"
#include "session/config.hpp"

namespace session {

// Build a PermissionPolicy from the loaded FullConfig
tools::PermissionPolicy build_policy(const FullConfig& cfg) {
    tools::PermissionPolicy policy;

    policy.blocked_tools  = cfg.blocked_tools;
    policy.allowed_paths  = cfg.allowed_paths;

    // Always allow read-only
    policy.allowed_permissions = { tools::ToolPermission::ReadOnly };

    if (cfg.allow_execute_tools) {
        policy.allowed_permissions.push_back(tools::ToolPermission::Execute);
        policy.allowed_permissions.push_back(tools::ToolPermission::ReadWrite);
    }
    if (cfg.allow_network_tools) {
        policy.allowed_permissions.push_back(tools::ToolPermission::Network);
    }

    // Whitelist overrides blocklist for explicitly listed tools
    // (handled in check_permission by inverting blocklist logic)

    return policy;
}

} // namespace session
```

---

## 7. SQLite Session Storage

For production deployments, sessions are stored in SQLite alongside the JSONL files:

```cpp
// src/session/sqlite_session_store.hpp
#pragma once
#include <SQLiteCpp/SQLiteCpp.h>
#include "session/session.hpp"

namespace session {

class SQLiteSessionStore {
public:
    explicit SQLiteSessionStore(const std::string& db_path);

    void             save(const Session& sess);
    std::optional<Session> load(const std::string& session_id) const;
    std::vector<std::string> list() const;
    void             delete_session(const std::string& session_id);

    // Search sessions by metadata
    std::vector<std::string> search(const std::string& query_text) const;

private:
    SQLite::Database db_;
    void init_schema();
};

} // namespace session
```

```cpp
// src/session/sqlite_session_store.cpp
#include "session/sqlite_session_store.hpp"
#include <nlohmann/json.hpp>

namespace session {

SQLiteSessionStore::SQLiteSessionStore(const std::string& db_path)
    : db_(db_path, SQLite::OPEN_READWRITE | SQLite::OPEN_CREATE)
{
    init_schema();
}

void SQLiteSessionStore::init_schema() {
    db_.exec(
        "CREATE TABLE IF NOT EXISTS sessions ("
        "  id TEXT PRIMARY KEY,"
        "  workspace_root TEXT,"
        "  created_at TEXT,"
        "  updated_at TEXT,"
        "  message_count INTEGER,"
        "  total_tokens INTEGER,"
        "  compaction_count INTEGER,"
        "  config TEXT,"         // JSON blob
        "  messages TEXT"        // JSON blob
        ")"
    );
    db_.exec(
        "CREATE INDEX IF NOT EXISTS idx_sessions_created ON sessions(created_at)"
    );
}

void SQLiteSessionStore::save(const Session& sess) {
    // Serialize messages to JSON
    nlohmann::json msgs = nlohmann::json::array();
    for (const auto& m : sess.messages) {
        nlohmann::json j;
        switch (m.role) {
            case agent::Role::User:      j["role"] = "user";      break;
            case agent::Role::Assistant: j["role"] = "assistant"; break;
            case agent::Role::Tool:      j["role"] = "tool";      break;
            default: j["role"] = "system"; break;
        }
        j["content"] = m.content;
        msgs.push_back(j);
    }

    nlohmann::json config_j = {
        {"model",          sess.config.model},
        {"max_iterations", sess.config.max_iterations},
        {"token_budget",   sess.config.token_budget},
    };

    auto now = std::format("{:%Y-%m-%dT%H:%M:%SZ}", std::chrono::system_clock::now());

    SQLite::Statement stmt(db_,
        "INSERT OR REPLACE INTO sessions "
        "(id, workspace_root, created_at, updated_at, message_count, "
        " total_tokens, compaction_count, config, messages) "
        "VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)");

    stmt.bind(1, sess.id);
    stmt.bind(2, sess.workspace_root);
    stmt.bind(3, sess.created_at);
    stmt.bind(4, now);
    stmt.bind(5, static_cast<int>(sess.messages.size()));
    stmt.bind(6, sess.total_tokens_used);
    stmt.bind(7, sess.compaction.compaction_count);
    stmt.bind(8, config_j.dump());
    stmt.bind(9, msgs.dump());
    stmt.exec();
}

std::vector<std::string> SQLiteSessionStore::list() const {
    std::vector<std::string> ids;
    SQLite::Statement q(db_, "SELECT id FROM sessions ORDER BY created_at DESC");
    while (q.executeStep())
        ids.push_back(q.getColumn(0).getString());
    return ids;
}

} // namespace session
```
