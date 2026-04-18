# 06 — Event-Driven Architecture

This document covers the full event sourcing, CQRS, pub-sub, and hook lifecycle
subsystem. All agent state mutations are captured as append-only events, enabling
deterministic replay, debugging, and multi-agent coordination.

Reference: Event sourcing for language agents (arXiv:2602.23193).

---

## 1. Event Types and Core Structs

```cpp
// src/events/event.hpp
#pragma once
#include <string>
#include <chrono>
#include <variant>
#include <nlohmann/json.hpp>

namespace events {

enum class EventType : uint32_t {
    // Session lifecycle
    SessionStarted      = 100,
    SessionResumed      = 101,
    SessionCompacted    = 102,
    SessionEnded        = 103,

    // Conversation
    UserMessage         = 200,
    AssistantMessage    = 201,
    SystemPromptUpdated = 202,

    // Tool use
    ToolUsed            = 300,
    ToolError           = 301,
    ToolPermissionDenied = 302,

    // Memory
    MemoryStored        = 400,
    MemoryRetrieved     = 401,
    MemoryEvicted       = 402,

    // RAG / KG
    RAGQueryExecuted    = 500,
    KGTripleAdded       = 501,
    KGCommunityDetected = 502,

    // Agent lifecycle
    AgentStarted        = 600,
    AgentIteration      = 601,
    FinalAnswer         = 602,
    AgentFailed         = 603,
    AgentAborted        = 604,

    // Multi-agent
    MessageSent         = 700,
    MessageReceived     = 701,
    AgentSpawned        = 702,
    AgentCompleted      = 703,

    // Self-improvement
    ReflexionStored     = 800,
    ExperienceAdded     = 801,
    SkillLearned        = 802,
};

std::string event_type_name(EventType t);

struct Event {
    EventType      type;
    nlohmann::json payload;
    std::string    timestamp;    // ISO-8601 UTC
    std::string    session_id;
    std::string    agent_id;     // for multi-agent setups
    uint64_t       sequence = 0; // monotonically increasing per session

    static Event make(EventType t,
                       nlohmann::json payload,
                       const std::string& session_id = "",
                       const std::string& agent_id   = "");
};

// Strongly typed event payloads — use std::variant for type-safe dispatch
struct ToolUsedPayload {
    std::string    tool_name;
    nlohmann::json arguments;
    std::string    result_snippet;
    bool           success;
    double         duration_ms;
};

struct UserMessagePayload {
    std::string content;
    int         estimated_tokens;
};

struct FinalAnswerPayload {
    std::string answer;
    int         total_iterations;
    int         total_tokens;
};

using TypedPayload = std::variant<
    ToolUsedPayload,
    UserMessagePayload,
    FinalAnswerPayload
    // ...extend as needed
>;

} // namespace events
```

```cpp
// src/events/event.cpp
#include "events/event.hpp"
#include <chrono>
#include <format>

namespace events {

std::string event_type_name(EventType t) {
    switch (t) {
        case EventType::SessionStarted:       return "SessionStarted";
        case EventType::UserMessage:          return "UserMessage";
        case EventType::AssistantMessage:     return "AssistantMessage";
        case EventType::ToolUsed:             return "ToolUsed";
        case EventType::ToolError:            return "ToolError";
        case EventType::FinalAnswer:          return "FinalAnswer";
        case EventType::AgentStarted:         return "AgentStarted";
        case EventType::AgentIteration:       return "AgentIteration";
        case EventType::ReflexionStored:      return "ReflexionStored";
        case EventType::MessageSent:          return "MessageSent";
        default: return std::format("Event({})", static_cast<uint32_t>(t));
    }
}

Event Event::make(EventType t, nlohmann::json payload,
                   const std::string& session_id,
                   const std::string& agent_id)
{
    // ISO-8601 timestamp
    auto now = std::chrono::system_clock::now();
    auto ts  = std::format("{:%Y-%m-%dT%H:%M:%SZ}", now);

    return Event{ t, std::move(payload), ts, session_id, agent_id, 0 };
}

} // namespace events
```

---

## 2. Append-Only JSONL Event Store

```cpp
// src/events/event_store.hpp
#pragma once
#include <string>
#include <vector>
#include <fstream>
#include <mutex>
#include <functional>
#include "events/event.hpp"
#include "agent/agent_error.hpp"

namespace events {

// Append-only event log — each line is a JSON-serialized Event
// Thread-safe for concurrent appends (multi-agent scenario)
class EventStore {
public:
    explicit EventStore(std::string log_path, std::string session_id = "");

    // Append an event — thread-safe
    void append(Event e);

    // Load all events from log file (for replay)
    AgentResult<std::vector<Event>> load_all() const;

    // Load events of a specific type
    AgentResult<std::vector<Event>> load_by_type(EventType t) const;

    // Load events since a sequence number
    AgentResult<std::vector<Event>> load_since(uint64_t seq) const;

    // Subscribe to new events (called in-process)
    using EventHandler = std::function<void(const Event&)>;
    void subscribe(EventHandler handler);

    uint64_t event_count() const { return next_seq_; }
    const std::string& session_id() const { return session_id_; }

private:
    std::string       log_path_;
    std::string       session_id_;
    std::ofstream     log_file_;
    std::mutex        mutex_;
    uint64_t          next_seq_ = 0;
    std::vector<EventHandler> handlers_;
};

} // namespace events
```

```cpp
// src/events/event_store.cpp
#include "events/event_store.hpp"
#include <nlohmann/json.hpp>
#include <spdlog/spdlog.h>
#include <fstream>

namespace events {

EventStore::EventStore(std::string log_path, std::string session_id)
    : log_path_(std::move(log_path))
    , session_id_(std::move(session_id))
    , log_file_(log_path_, std::ios::app)
{
    if (!log_file_.is_open())
        spdlog::error("[event_store] cannot open log: {}", log_path_);

    // Determine next sequence from existing log
    auto existing = load_all();
    if (existing && !existing->empty())
        next_seq_ = existing->back().sequence + 1;
}

void EventStore::append(Event e) {
    std::lock_guard lock(mutex_);

    e.sequence   = next_seq_++;
    e.session_id = session_id_;
    if (e.timestamp.empty())
        e = Event::make(e.type, e.payload, session_id_, e.agent_id);

    nlohmann::json j = {
        {"type",       static_cast<uint32_t>(e.type)},
        {"type_name",  event_type_name(e.type)},
        {"payload",    e.payload},
        {"timestamp",  e.timestamp},
        {"session_id", e.session_id},
        {"agent_id",   e.agent_id},
        {"sequence",   e.sequence},
    };

    log_file_ << j.dump() << '\n';
    log_file_.flush();

    // Notify subscribers (synchronous, in-process)
    for (const auto& h : handlers_) {
        try { h(e); } catch (...) {}
    }
}

AgentResult<std::vector<Event>> EventStore::load_all() const {
    std::ifstream f(log_path_);
    if (!f) return std::vector<Event>{};

    std::vector<Event> events;
    std::string line;
    while (std::getline(f, line)) {
        if (line.empty()) continue;
        try {
            auto j = nlohmann::json::parse(line);
            Event e;
            e.type       = static_cast<EventType>(j["type"].get<uint32_t>());
            e.payload    = j["payload"];
            e.timestamp  = j["timestamp"].get<std::string>();
            e.session_id = j["session_id"].get<std::string>();
            e.agent_id   = j.value("agent_id", "");
            e.sequence   = j["sequence"].get<uint64_t>();
            events.push_back(std::move(e));
        } catch (const std::exception& ex) {
            spdlog::warn("[event_store] parse error on line: {}", ex.what());
        }
    }
    return events;
}

AgentResult<std::vector<Event>> EventStore::load_by_type(EventType t) const {
    auto all = load_all();
    if (!all) return std::unexpected(all.error());
    std::vector<Event> filtered;
    for (const auto& e : *all)
        if (e.type == t) filtered.push_back(e);
    return filtered;
}

AgentResult<std::vector<Event>> EventStore::load_since(uint64_t seq) const {
    auto all = load_all();
    if (!all) return std::unexpected(all.error());
    std::vector<Event> filtered;
    for (const auto& e : *all)
        if (e.sequence >= seq) filtered.push_back(e);
    return filtered;
}

void EventStore::subscribe(EventHandler handler) {
    std::lock_guard lock(mutex_);
    handlers_.push_back(std::move(handler));
}

} // namespace events
```

---

## 3. CQRS — Materialized View (State Projection)

The materialized view reconstructs agent state by replaying events.
This is the read side of CQRS; the write side is the EventStore.

```cpp
// src/events/materialized_view.hpp
#pragma once
#include <string>
#include <vector>
#include <optional>
#include "events/event.hpp"
#include "agent/agent_context.hpp"

namespace events {

// Projected agent state — rebuilt by replaying events
struct AgentMaterializedView {
    std::string session_id;
    std::string agent_id;

    // Conversation state
    std::vector<agent::Message> messages;
    std::string                  last_answer;

    // Statistics
    int     total_iterations = 0;
    int     total_tool_calls = 0;
    int     failed_tool_calls = 0;
    double  total_llm_latency_ms = 0.0;

    // Tools used (name → count)
    std::unordered_map<std::string, int> tool_call_counts;

    // Reflexion memory
    std::vector<std::string> reflections;

    // Current status
    enum class Status { Running, Completed, Failed, Aborted } status = Status::Running;
};

// Replay events to reconstruct state
AgentMaterializedView project(const std::vector<Event>& events);

// Incremental update: apply a single event to an existing view
void apply_event(AgentMaterializedView& view, const Event& e);

} // namespace events
```

```cpp
// src/events/materialized_view.cpp
#include "events/materialized_view.hpp"
#include <spdlog/spdlog.h>

namespace events {

void apply_event(AgentMaterializedView& view, const Event& e) {
    switch (e.type) {
        case EventType::SessionStarted:
            view.session_id = e.session_id;
            view.status     = AgentMaterializedView::Status::Running;
            break;

        case EventType::UserMessage:
            view.messages.push_back({
                agent::Role::User,
                e.payload.value("content", "")
            });
            break;

        case EventType::AssistantMessage:
            view.messages.push_back({
                agent::Role::Assistant,
                e.payload.value("content", "")
            });
            view.total_iterations++;
            break;

        case EventType::ToolUsed:
            view.total_tool_calls++;
            view.tool_call_counts[e.payload.value("tool", "unknown")]++;
            if (e.payload.contains("duration_ms"))
                view.total_llm_latency_ms += e.payload["duration_ms"].get<double>();
            break;

        case EventType::ToolError:
            view.failed_tool_calls++;
            break;

        case EventType::FinalAnswer:
            view.last_answer = e.payload.value("answer", "");
            view.status      = AgentMaterializedView::Status::Completed;
            break;

        case EventType::AgentFailed:
            view.status = AgentMaterializedView::Status::Failed;
            break;

        case EventType::AgentAborted:
            view.status = AgentMaterializedView::Status::Aborted;
            break;

        case EventType::ReflexionStored:
            if (e.payload.contains("reflection"))
                view.reflections.push_back(e.payload["reflection"].get<std::string>());
            break;

        default:
            break;  // Ignore unknown events — forward compatibility
    }
}

AgentMaterializedView project(const std::vector<Event>& events) {
    AgentMaterializedView view;
    for (const auto& e : events)
        apply_event(view, e);
    return view;
}

} // namespace events
```

---

## 4. Event Bus — Typed Pub/Sub with std::variant

```cpp
// src/events/event_bus.hpp
#pragma once
#include <functional>
#include <unordered_map>
#include <vector>
#include <mutex>
#include <typeindex>
#include "events/event.hpp"

namespace events {

// Typed event bus — subscribers register for specific EventType values.
// Handlers are called synchronously in the same thread that publishes.
// For async cross-thread delivery, pair with a message queue (see multiagent/).
class EventBus {
public:
    using Handler = std::function<void(const Event&)>;

    // Subscribe to a specific event type
    void subscribe(EventType type, Handler handler) {
        std::lock_guard lock(mutex_);
        handlers_[static_cast<uint32_t>(type)].push_back(std::move(handler));
    }

    // Subscribe to ALL events
    void subscribe_all(Handler handler) {
        std::lock_guard lock(mutex_);
        wildcard_handlers_.push_back(std::move(handler));
    }

    // Publish an event — calls all registered handlers
    void publish(const Event& e) {
        std::lock_guard lock(mutex_);
        auto key = static_cast<uint32_t>(e.type);
        if (auto it = handlers_.find(key); it != handlers_.end())
            for (const auto& h : it->second) h(e);
        for (const auto& h : wildcard_handlers_) h(e);
    }

    // Helper: subscribe with a typed handler using std::visit pattern
    template<typename PayloadT, typename HandlerT>
    void subscribe_typed(EventType type, HandlerT&& handler) {
        subscribe(type, [h = std::forward<HandlerT>(handler)](const Event& e) {
            // Attempt to deserialize payload to PayloadT
            // In production: use glaze or nlohmann adl_serializer
            try {
                PayloadT payload = e.payload.get<PayloadT>();
                h(payload);
            } catch (...) {}
        });
    }

private:
    std::mutex mutex_;
    std::unordered_map<uint32_t, std::vector<Handler>> handlers_;
    std::vector<Handler> wildcard_handlers_;
};

// Global singleton bus — use sparingly; prefer passing by reference
EventBus& global_bus();

} // namespace events
```

---

## 5. Hook Runner — PreToolUse / PostToolUse Lifecycle

```cpp
// src/events/hook_runner.hpp
#pragma once
#include "tools/hooks.hpp"
#include "events/event_bus.hpp"

namespace events {

// HookRunner bridges tool hooks and the event bus
// so tool lifecycle events are automatically published
class HookRunner {
public:
    explicit HookRunner(EventBus& bus, tools::HookSet& hooks);

    // Wires hook callbacks to emit events on the bus
    void wire();

    // Pre-hook: publishes ToolInvoking event, checks rate limits
    std::optional<tools::ToolResult> pre(
        const tools::ToolSpec& spec,
        const nlohmann::json& args);

    // Post-hook: publishes ToolUsed event with duration
    void post(const tools::ToolSpec& spec,
               const nlohmann::json& args,
               const tools::ToolResult& result,
               double duration_ms);

    // Failure hook
    void failure(const tools::ToolSpec& spec,
                  const nlohmann::json& args,
                  const agent::AgentError& error);

private:
    EventBus&        bus_;
    tools::HookSet&  hooks_;

    // Rate limiter: tool name → last call timestamp
    std::flat_map<std::string, std::chrono::steady_clock::time_point> last_call_;
    std::flat_map<std::string, std::chrono::milliseconds> rate_limits_;
};

} // namespace events
```

---

## 6. Topic-Based Pub/Sub for Multi-Agent

```cpp
// src/events/topic_bus.hpp
#pragma once
#include <string>
#include <functional>
#include <unordered_map>
#include <vector>
#include <mutex>
#include <nlohmann/json.hpp>

namespace events {

// Topic-based message routing for multi-agent communication.
// Topics follow a hierarchical path convention:
//   "agent/{agent_id}/task"       — direct to agent
//   "broadcast/all"               — all agents
//   "coordinator/results"         — coordinator inbox
struct TopicMessage {
    std::string    topic;
    std::string    sender_id;
    nlohmann::json content;
    std::string    timestamp;
    std::string    correlation_id;  // for request/reply patterns
};

using TopicHandler = std::function<void(const TopicMessage&)>;

class TopicBus {
public:
    // Subscribe to exact topic or prefix (ends with "/*")
    void subscribe(const std::string& topic, TopicHandler handler);

    // Unsubscribe
    void unsubscribe(const std::string& topic);

    // Publish to a topic — calls all matching subscribers
    void publish(TopicMessage msg);

    // Request/reply: publish and wait for response on correlation topic
    // (blocks calling thread — use cobalt::task in async context)
    std::optional<TopicMessage> request(
        const std::string& topic,
        const nlohmann::json& content,
        const std::string& sender_id,
        std::chrono::milliseconds timeout = std::chrono::seconds(30));

private:
    std::mutex mutex_;
    std::unordered_map<std::string, std::vector<TopicHandler>> exact_handlers_;
    std::vector<std::pair<std::string, TopicHandler>>          prefix_handlers_;

    bool matches(const std::string& pattern, const std::string& topic) const;
};

} // namespace events
```

```cpp
// src/events/topic_bus.cpp
#include "events/topic_bus.hpp"

namespace events {

void TopicBus::subscribe(const std::string& topic, TopicHandler handler) {
    std::lock_guard lock(mutex_);
    if (topic.ends_with("/*")) {
        std::string prefix = topic.substr(0, topic.size() - 2);
        prefix_handlers_.push_back({ prefix, std::move(handler) });
    } else {
        exact_handlers_[topic].push_back(std::move(handler));
    }
}

void TopicBus::publish(TopicMessage msg) {
    std::lock_guard lock(mutex_);

    // Exact match
    if (auto it = exact_handlers_.find(msg.topic); it != exact_handlers_.end())
        for (const auto& h : it->second) h(msg);

    // Prefix match
    for (const auto& [prefix, handler] : prefix_handlers_)
        if (msg.topic.starts_with(prefix)) handler(msg);
}

bool TopicBus::matches(const std::string& pattern, const std::string& topic) const {
    if (pattern.ends_with("/*"))
        return topic.starts_with(pattern.substr(0, pattern.size() - 2));
    return pattern == topic;
}

} // namespace events
```

---

## 7. Event Replay for Debugging

```cpp
// src/events/replay.hpp
#pragma once
#include <string>
#include <functional>
#include <chrono>
#include "events/event.hpp"
#include "events/materialized_view.hpp"

namespace events {

struct ReplayConfig {
    // Only replay events of these types (empty = all)
    std::vector<EventType> filter_types;

    // Time range filter
    std::optional<std::string> start_timestamp;
    std::optional<std::string> end_timestamp;

    // Slow-motion: delay between events for debugging
    std::chrono::milliseconds step_delay { 0 };

    // Stop at this sequence number
    std::optional<uint64_t> stop_at_sequence;
};

// Replay events from a log file, calling `handler` for each event.
// Returns the final materialized view.
AgentMaterializedView replay(
    const std::string& log_path,
    const ReplayConfig& cfg,
    std::function<void(const Event&, const AgentMaterializedView&)> handler = {});

// Reconstruct agent state at a specific sequence number
AgentMaterializedView state_at(const std::string& log_path, uint64_t sequence);

// Find events matching a payload predicate
std::vector<Event> find_events(
    const std::string& log_path,
    std::function<bool(const Event&)> predicate);

} // namespace events
```

```cpp
// src/events/replay.cpp
#include "events/replay.hpp"
#include "events/event_store.hpp"
#include <thread>

namespace events {

AgentMaterializedView replay(
    const std::string& log_path,
    const ReplayConfig& cfg,
    std::function<void(const Event&, const AgentMaterializedView&)> handler)
{
    EventStore store(log_path);
    auto all_events = store.load_all();
    if (!all_events) return {};

    AgentMaterializedView view;

    for (const auto& e : *all_events) {
        // Sequence filter
        if (cfg.stop_at_sequence && e.sequence > *cfg.stop_at_sequence) break;

        // Type filter
        if (!cfg.filter_types.empty()) {
            bool match = std::any_of(cfg.filter_types.begin(), cfg.filter_types.end(),
                [&e](EventType t) { return t == e.type; });
            if (!match) continue;
        }

        apply_event(view, e);

        if (handler) handler(e, view);

        if (cfg.step_delay.count() > 0)
            std::this_thread::sleep_for(cfg.step_delay);
    }

    return view;
}

AgentMaterializedView state_at(const std::string& log_path, uint64_t sequence) {
    ReplayConfig cfg;
    cfg.stop_at_sequence = sequence;
    return replay(log_path, cfg);
}

std::vector<Event> find_events(
    const std::string& log_path,
    std::function<bool(const Event&)> predicate)
{
    EventStore store(log_path);
    auto all = store.load_all();
    if (!all) return {};

    std::vector<Event> result;
    for (const auto& e : *all)
        if (predicate(e)) result.push_back(e);

    return result;
}

} // namespace events
```

---

## 8. Complete Event Flow Diagram

```
User request arrives
         │
         ▼
EventStore.append(UserMessage)
         │
         ▼
EventBus.publish(UserMessage)
         │         │
         │     ───────────────────────────────────────────────
         │    │  TopicBus (multi-agent)                       │
         │    │  Subscribers: coordinator, logger, tracer     │
         │    └──────────────────────────────────────────────
         │
         ▼
Agent loop iteration
         │
         ├── EventStore.append(AgentIteration)
         │
         ├── Tool call dispatched
         │     │
         │     ├── HookRunner.pre()
         │     │     └── publishes ToolInvoking
         │     │
         │     ├── Tool executes
         │     │
         │     └── HookRunner.post() or .failure()
         │           └── publishes ToolUsed / ToolError
         │
         └── EventStore.append(AssistantMessage)

Final answer reached
         │
         ▼
EventStore.append(FinalAnswer)
         │
         ▼
EventBus.publish(FinalAnswer)
         │
         ├── MaterializedView updated (in-memory CQRS read model)
         │
         └── TopicBus notifies coordinator (multi-agent)

At any time: replay log_file → rebuild view deterministically
```

---

## 9. LangSmith-Compatible Tracing

Output events in LangSmith trace format for observability:

```cpp
// src/events/langsmith_tracer.hpp
#pragma once
#include "events/event_bus.hpp"
#include <httplib.h>

namespace events {

// Subscribe to EventBus and forward events to LangSmith tracing API
class LangSmithTracer {
public:
    LangSmithTracer(EventBus& bus,
                     std::string api_key,
                     std::string project_name);

    void start();  // Subscribe and begin tracing

private:
    EventBus&   bus_;
    std::string api_key_;
    std::string project_name_;
    std::string run_id_;
    httplib::Client client_;

    void on_event(const Event& e);
    void post_run_create(const Event& e);
    void post_run_update(const std::string& run_id, const Event& e);
};

} // namespace events
```
