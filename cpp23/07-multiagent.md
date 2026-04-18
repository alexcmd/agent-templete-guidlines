# 07 — Multi-Agent Patterns

This document covers the actor model, direct/broadcast messaging, supervisor routing,
AutoGen conversation topologies, MetaGPT SOP pipelines, CAMEL role-playing,
Reflexion split agents, and result aggregation.

---

## 1. Actor Model — Agent as Message-Processing Entity

Each agent is an actor with a mailbox (thread-safe message queue) and a run loop.
Actors communicate only through messages — no shared mutable state between agents.

```cpp
// src/multiagent/actor.hpp
#pragma once
#include <string>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <thread>
#include <atomic>
#include <functional>
#include <nlohmann/json.hpp>
#include "agent/agent_error.hpp"

namespace multiagent {

struct ActorMessage {
    std::string    from_id;
    std::string    to_id;
    std::string    type;       // "task", "result", "error", "control"
    nlohmann::json content;
    std::string    correlation_id;
    std::string    timestamp;
};

// Thread-safe bounded mailbox
class Mailbox {
public:
    explicit Mailbox(std::size_t max_size = 1000);

    bool send(ActorMessage msg);     // Returns false if full
    ActorMessage receive();          // Blocks until message arrives
    std::optional<ActorMessage> try_receive(std::chrono::milliseconds timeout);
    bool empty() const;
    std::size_t size() const;
    void close();  // Unblock waiting receivers

private:
    std::size_t max_size_;
    std::queue<ActorMessage> queue_;
    mutable std::mutex mutex_;
    std::condition_variable cv_;
    bool closed_ = false;
};

// Base class for all agents in the multi-agent system
class Actor {
public:
    explicit Actor(std::string id, std::string role = "");
    virtual ~Actor();

    const std::string& id()   const { return id_; }
    const std::string& role() const { return role_; }

    // Send a message to this actor's mailbox
    bool send(ActorMessage msg) { return mailbox_.send(std::move(msg)); }

    // Start the actor's processing loop in a background thread
    void start();

    // Request graceful shutdown
    void stop();

    // Wait for the actor to finish
    void join();

    bool is_running() const { return running_.load(); }

protected:
    // Override in subclasses: handle a single message
    virtual void handle_message(const ActorMessage& msg) = 0;

    // Send a message to another actor via the runtime
    void send_to(const std::string& target_id, ActorMessage msg);

    // Convenience: reply to a received message
    void reply(const ActorMessage& original, nlohmann::json content);

    std::string id_;
    std::string role_;
    Mailbox     mailbox_;

private:
    std::jthread    thread_;
    std::atomic<bool> running_ { false };

    void run_loop();
};

} // namespace multiagent
```

```cpp
// src/multiagent/actor.cpp
#include "multiagent/actor.hpp"
#include "multiagent/actor_runtime.hpp"
#include <spdlog/spdlog.h>

namespace multiagent {

Mailbox::Mailbox(std::size_t max_size) : max_size_(max_size) {}

bool Mailbox::send(ActorMessage msg) {
    std::lock_guard lock(mutex_);
    if (queue_.size() >= max_size_) return false;
    queue_.push(std::move(msg));
    cv_.notify_one();
    return true;
}

ActorMessage Mailbox::receive() {
    std::unique_lock lock(mutex_);
    cv_.wait(lock, [this] { return !queue_.empty() || closed_; });
    if (closed_ && queue_.empty())
        return { "", "", "control", { {"cmd", "shutdown"} } };
    auto msg = std::move(queue_.front());
    queue_.pop();
    return msg;
}

std::optional<ActorMessage> Mailbox::try_receive(std::chrono::milliseconds timeout) {
    std::unique_lock lock(mutex_);
    if (cv_.wait_for(lock, timeout, [this] { return !queue_.empty(); })) {
        auto msg = std::move(queue_.front());
        queue_.pop();
        return msg;
    }
    return std::nullopt;
}

void Mailbox::close() {
    std::lock_guard lock(mutex_);
    closed_ = true;
    cv_.notify_all();
}

bool Mailbox::empty() const {
    std::lock_guard lock(mutex_);
    return queue_.empty();
}

Actor::Actor(std::string id, std::string role)
    : id_(std::move(id)), role_(std::move(role)) {}

Actor::~Actor() { stop(); join(); }

void Actor::start() {
    running_.store(true);
    thread_ = std::jthread([this](std::stop_token) { run_loop(); });
}

void Actor::stop() {
    running_.store(false);
    mailbox_.close();
}

void Actor::join() {
    if (thread_.joinable()) thread_.join();
}

void Actor::run_loop() {
    spdlog::info("[actor] {} started (role={})", id_, role_);
    while (running_.load()) {
        auto msg = mailbox_.receive();
        if (msg.type == "control" && msg.content.value("cmd", "") == "shutdown") break;
        try {
            handle_message(msg);
        } catch (const std::exception& e) {
            spdlog::error("[actor] {} unhandled exception: {}", id_, e.what());
        }
    }
    spdlog::info("[actor] {} stopped", id_);
}

} // namespace multiagent
```

---

## 2. Actor Runtime — Spawn and Supervise

```cpp
// src/multiagent/actor_runtime.hpp
#pragma once
#include <string>
#include <memory>
#include <unordered_map>
#include <mutex>
#include "multiagent/actor.hpp"

namespace multiagent {

class ActorRuntime {
public:
    static ActorRuntime& instance();

    // Spawn and register an actor
    template<typename ActorT, typename... Args>
    ActorT* spawn(std::string id, Args&&... args) {
        auto actor = std::make_unique<ActorT>(id, std::forward<Args>(args)...);
        auto* ptr  = actor.get();
        {
            std::lock_guard lock(mutex_);
            actors_[id] = std::move(actor);
        }
        ptr->start();
        return ptr;
    }

    // Lookup actor by ID
    Actor* find(const std::string& id) const;

    // Send a message by actor ID
    bool send(const std::string& to_id, ActorMessage msg);

    // Broadcast to all actors
    void broadcast(ActorMessage msg);

    // Terminate an actor
    void terminate(const std::string& id);

    // Terminate all actors and wait
    void shutdown_all();

    std::vector<std::string> actor_ids() const;

private:
    ActorRuntime() = default;
    std::unordered_map<std::string, std::unique_ptr<Actor>> actors_;
    mutable std::mutex mutex_;
};

} // namespace multiagent
```

```cpp
// src/multiagent/actor_runtime.cpp
#include "multiagent/actor_runtime.hpp"
#include <spdlog/spdlog.h>

namespace multiagent {

ActorRuntime& ActorRuntime::instance() {
    static ActorRuntime instance;
    return instance;
}

Actor* ActorRuntime::find(const std::string& id) const {
    std::lock_guard lock(mutex_);
    auto it = actors_.find(id);
    return it != actors_.end() ? it->second.get() : nullptr;
}

bool ActorRuntime::send(const std::string& to_id, ActorMessage msg) {
    auto* actor = find(to_id);
    if (!actor) {
        spdlog::warn("[runtime] send: actor '{}' not found", to_id);
        return false;
    }
    return actor->send(std::move(msg));
}

void ActorRuntime::broadcast(ActorMessage msg) {
    std::lock_guard lock(mutex_);
    for (const auto& [id, actor] : actors_)
        actor->send(msg);
}

void ActorRuntime::terminate(const std::string& id) {
    std::lock_guard lock(mutex_);
    if (auto it = actors_.find(id); it != actors_.end()) {
        it->second->stop();
        it->second->join();
        actors_.erase(it);
    }
}

void ActorRuntime::shutdown_all() {
    std::lock_guard lock(mutex_);
    for (auto& [id, actor] : actors_) actor->stop();
    for (auto& [id, actor] : actors_) actor->join();
    actors_.clear();
}

} // namespace multiagent
```

---

## 3. SupervisorAgent — Route Tasks to Specialists

```cpp
// src/multiagent/supervisor.hpp
#pragma once
#include <string>
#include <vector>
#include <functional>
#include "multiagent/actor.hpp"
#include "agent/agent_context.hpp"
#include "inference/sse_parser.hpp"

namespace multiagent {

struct AgentSpec {
    std::string id;
    std::string role;
    std::string description;  // What tasks this agent handles
    std::vector<std::string> capabilities;
};

// SupervisorAgent: receives tasks, routes to appropriate specialists,
// collects and synthesizes results.
class SupervisorAgent : public Actor {
public:
    SupervisorAgent(std::string id,
                     std::vector<AgentSpec> specialists,
                     agent::AgentConfig cfg,
                     inference::SSEParser& sse);

protected:
    void handle_message(const ActorMessage& msg) override;

private:
    // Select best specialist for a task using LLM routing
    std::string route_task(const std::string& task);

    // Aggregate results from parallel subagents
    std::string aggregate_results(
        const std::string& original_task,
        const std::vector<std::pair<std::string, std::string>>& results);

    std::vector<AgentSpec>    specialists_;
    agent::AgentConfig        cfg_;
    inference::SSEParser&     sse_;

    // Pending tasks: correlation_id → response channel
    std::unordered_map<std::string, std::function<void(const std::string&)>> pending_;
    std::mutex pending_mutex_;
};

} // namespace multiagent
```

```cpp
// src/multiagent/supervisor.cpp
#include "multiagent/supervisor.hpp"
#include "multiagent/actor_runtime.hpp"
#include <nlohmann/json.hpp>
#include <format>

namespace multiagent {

SupervisorAgent::SupervisorAgent(
    std::string id,
    std::vector<AgentSpec> specialists,
    agent::AgentConfig cfg,
    inference::SSEParser& sse)
    : Actor(std::move(id), "supervisor")
    , specialists_(std::move(specialists))
    , cfg_(std::move(cfg))
    , sse_(sse)
{}

std::string SupervisorAgent::route_task(const std::string& task) {
    std::string spec_list;
    for (const auto& s : specialists_)
        spec_list += std::format("  - {} ({}): {}\n", s.id, s.role, s.description);

    nlohmann::json req = {
        {"model",  cfg_.model},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content",
            "Given the following task and list of specialist agents, "
            "select the most appropriate agent ID to handle it.\n\n"
            "Task: " + task + "\n\n"
            "Specialists:\n" + spec_list + "\n"
            "Reply with ONLY the agent ID, nothing else."
        }}}}
    };

    std::string chosen = sse_.accumulate(req);
    // Trim whitespace
    while (!chosen.empty() && std::isspace(chosen.back())) chosen.pop_back();

    // Validate
    for (const auto& s : specialists_)
        if (s.id == chosen) return chosen;

    // Fallback to first specialist
    return specialists_.empty() ? "" : specialists_[0].id;
}

void SupervisorAgent::handle_message(const ActorMessage& msg) {
    if (msg.type == "task") {
        std::string task = msg.content.value("task", "");

        // Route to specialist
        std::string specialist_id = route_task(task);
        if (specialist_id.empty()) {
            reply(msg, { {"error", "No specialist available"} });
            return;
        }

        spdlog::info("[supervisor] routing '{}' to {}", task.substr(0, 50), specialist_id);

        // Register pending callback
        {
            std::lock_guard lock(pending_mutex_);
            pending_[msg.correlation_id] = [this, original_msg = msg](const std::string& result) {
                reply(original_msg, { {"result", result} });
            };
        }

        // Forward to specialist
        ActorRuntime::instance().send(specialist_id, {
            id_, specialist_id, "task",
            { {"task", task} },
            msg.correlation_id,
            msg.timestamp
        });

    } else if (msg.type == "result") {
        // Receive result from specialist
        std::string result = msg.content.value("result", "");
        std::lock_guard lock(pending_mutex_);
        if (auto it = pending_.find(msg.correlation_id); it != pending_.end()) {
            it->second(result);
            pending_.erase(it);
        }
    }
}

std::string SupervisorAgent::aggregate_results(
    const std::string& original_task,
    const std::vector<std::pair<std::string, std::string>>& results)
{
    std::string results_text;
    for (const auto& [agent_id, result] : results)
        results_text += std::format("[{}]: {}\n\n", agent_id, result);

    nlohmann::json req = {
        {"model",  cfg_.model},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content",
            "Synthesize the following results from multiple specialist agents "
            "into a single coherent response for the original task.\n\n"
            "Task: " + original_task + "\n\n"
            "Results:\n" + results_text
        }}}}
    };

    return sse_.accumulate(req);
}

} // namespace multiagent
```

---

## 4. AutoGen Topologies

### Sequential Topology

```cpp
// src/multiagent/topologies.hpp
#pragma once
#include <vector>
#include <string>
#include <functional>
#include "multiagent/actor.hpp"

namespace multiagent {

// Sequential: Agent A → Agent B → Agent C → ...
// Each agent receives the previous output as input
struct SequentialTopology {
    std::vector<std::string> agent_ids;  // ordered pipeline

    AgentResult<std::string> execute(
        const std::string& initial_input,
        ActorRuntime& runtime,
        std::chrono::seconds timeout = std::chrono::seconds(300));
};

// Parallel: all agents receive the same input, results are aggregated
struct ParallelTopology {
    std::vector<std::string> agent_ids;

    AgentResult<std::vector<std::string>> execute(
        const std::string& input,
        ActorRuntime& runtime,
        std::chrono::seconds timeout = std::chrono::seconds(300));
};

// Hierarchical: coordinator delegates subtasks to workers
struct HierarchicalTopology {
    std::string              coordinator_id;
    std::vector<std::string> worker_ids;

    AgentResult<std::string> execute(
        const std::string& input,
        ActorRuntime& runtime,
        std::chrono::seconds timeout = std::chrono::seconds(600));
};

} // namespace multiagent
```

---

## 5. MetaGPT SOP Pipeline

MetaGPT defines Standardized Operating Procedures (SOPs) where each role
produces a specific artifact that feeds the next stage.

```cpp
// src/multiagent/metagpt.hpp
#pragma once
#include <string>
#include <vector>
#include <functional>
#include "agent/agent_context.hpp"

namespace multiagent::metagpt {

struct Artifact {
    std::string type;       // "PRD", "Architecture", "Code", "Tests", "Review"
    std::string content;
    std::string author_role;
};

struct SopRole {
    std::string name;       // "ProductManager", "Architect", "Engineer", "QA"
    std::string system_prompt;
    std::string produces;   // artifact type this role produces
    std::string consumes;   // artifact type this role consumes

    // Generate artifact from previous artifact
    Artifact execute(const Artifact& input,
                      inference::SSEParser& sse,
                      const agent::AgentConfig& cfg) const;
};

// Full MetaGPT-style pipeline
struct SopPipeline {
    std::vector<SopRole> roles;

    // Run through all roles sequentially
    std::vector<Artifact> execute(
        const std::string& initial_requirement,
        inference::SSEParser& sse,
        const agent::AgentConfig& cfg);
};

// Built-in role definitions
SopRole make_product_manager_role();
SopRole make_architect_role();
SopRole make_engineer_role();
SopRole make_qa_role();
SopPipeline make_software_dev_pipeline();

} // namespace multiagent::metagpt
```

```cpp
// src/multiagent/metagpt.cpp
#include "multiagent/metagpt.hpp"

namespace multiagent::metagpt {

Artifact SopRole::execute(const Artifact& input,
                            inference::SSEParser& sse,
                            const agent::AgentConfig& cfg) const
{
    nlohmann::json req = {
        {"model",  cfg.model},
        {"stream", false},
        {"messages", {
            {{"role", "system"}, {"content", system_prompt}},
            {{"role", "user"},   {"content",
                std::format("Input ({}):\n{}\n\nProduce: {}",
                    input.type, input.content, produces)
            }}
        }}
    };

    std::string output = sse.accumulate(req);
    return Artifact{ produces, output, name };
}

std::vector<Artifact> SopPipeline::execute(
    const std::string& initial_requirement,
    inference::SSEParser& sse,
    const agent::AgentConfig& cfg)
{
    std::vector<Artifact> artifacts;
    Artifact current{ "Requirement", initial_requirement, "User" };

    for (const auto& role : roles) {
        spdlog::info("[metagpt] {} processing...", role.name);
        current = role.execute(current, sse, cfg);
        artifacts.push_back(current);
    }

    return artifacts;
}

SopRole make_product_manager_role() {
    return SopRole{
        "ProductManager",
        "You are a Product Manager. Given user requirements, produce a detailed "
        "Product Requirements Document (PRD) with user stories, acceptance criteria, "
        "and technical constraints.",
        "PRD",
        "Requirement"
    };
}

SopRole make_architect_role() {
    return SopRole{
        "Architect",
        "You are a Software Architect. Given a PRD, produce a detailed technical "
        "architecture document including component diagram, data flow, API design, "
        "and technology stack decisions.",
        "Architecture",
        "PRD"
    };
}

SopRole make_engineer_role() {
    return SopRole{
        "Engineer",
        "You are a Senior Software Engineer. Given an architecture document, "
        "implement the code. Write clean, production-ready C++23 code with "
        "proper error handling using std::expected.",
        "Code",
        "Architecture"
    };
}

SopRole make_qa_role() {
    return SopRole{
        "QA",
        "You are a QA Engineer. Given code, write comprehensive tests using "
        "Google Test. Cover unit tests, integration tests, and edge cases.",
        "Tests",
        "Code"
    };
}

SopPipeline make_software_dev_pipeline() {
    return SopPipeline{{
        make_product_manager_role(),
        make_architect_role(),
        make_engineer_role(),
        make_qa_role()
    }};
}

} // namespace multiagent::metagpt
```

---

## 6. CAMEL Role-Playing — Inception Prompting

```cpp
// src/multiagent/camel.cpp
#include "multiagent/camel.hpp"

namespace multiagent::camel {

// CAMEL inception prompt: makes agents adopt specific roles
std::string make_inception_prompt(
    const std::string& assistant_role,
    const std::string& user_role,
    const std::string& task)
{
    return std::format(
        "You are playing the role of {0}. The user is playing the role of {1}.\n\n"
        "Task: {2}\n\n"
        "As {0}, you must:\n"
        "1. Understand your role's constraints and expertise\n"
        "2. Collaborate with the {1} to accomplish the task\n"
        "3. Stay in character throughout the conversation\n"
        "4. Only provide assistance within your role's domain\n\n"
        "Begin the conversation to accomplish the task.",
        assistant_role, user_role, task
    );
}

// Run a CAMEL role-play session for N rounds
std::vector<std::pair<std::string, std::string>> camel_session(
    const std::string& user_role,
    const std::string& assistant_role,
    const std::string& task,
    int max_rounds,
    inference::SSEParser& sse,
    const agent::AgentConfig& cfg)
{
    std::vector<std::pair<std::string, std::string>> conversation;

    // Initialize both agents with inception prompts
    std::string user_sys = make_inception_prompt(user_role, assistant_role, task);
    std::string asst_sys = make_inception_prompt(assistant_role, user_role, task);

    // User initiates
    nlohmann::json user_req = {
        {"model",  cfg.model},
        {"stream", false},
        {"messages", {
            {{"role","system"}, {"content", user_sys}},
            {{"role","user"},   {"content", "Begin working on the task."}}
        }}
    };
    std::string user_msg = sse.accumulate(user_req);

    for (int round = 0; round < max_rounds; ++round) {
        // Assistant responds
        nlohmann::json asst_req = {
            {"model",  cfg.model},
            {"stream", false},
            {"messages", {
                {{"role","system"}, {"content", asst_sys}},
                {{"role","user"},   {"content", user_msg}}
            }}
        };
        std::string asst_msg = sse.accumulate(asst_req);
        conversation.push_back({ user_role, user_msg });
        conversation.push_back({ assistant_role, asst_msg });

        // Check task completion signal
        if (asst_msg.find("<TASK_DONE>") != std::string::npos) break;

        // User continues
        nlohmann::json next_user_req = {
            {"model",  cfg.model},
            {"stream", false},
            {"messages", {
                {{"role","system"}, {"content", user_sys}},
                {{"role","user"},   {"content", asst_msg}}
            }}
        };
        user_msg = sse.accumulate(next_user_req);
    }

    return conversation;
}

} // namespace multiagent::camel
```

---

## 7. Reflexion Multi-Agent Split

Reflexion (arXiv:2303.11366) works best when Actor, Evaluator, and Self-Reflection
are separate agents, each with a focused role:

```cpp
// src/multiagent/reflexion_agents.hpp
#pragma once
#include "multiagent/actor.hpp"
#include "agent/agent_context.hpp"

namespace multiagent {

// Actor agent: executes the task using the base ReAct loop
class ActorAgent : public Actor {
public:
    ActorAgent(std::string id, agent::AgentContext base_ctx);
protected:
    void handle_message(const ActorMessage& msg) override;
private:
    agent::AgentContext ctx_;
};

// Evaluator agent: scores the Actor's output against a rubric
class EvaluatorAgent : public Actor {
public:
    EvaluatorAgent(std::string id, std::string rubric,
                    agent::AgentConfig cfg, inference::SSEParser& sse);
protected:
    void handle_message(const ActorMessage& msg) override;
private:
    std::string        rubric_;
    agent::AgentConfig cfg_;
    inference::SSEParser& sse_;
};

// SelfReflection agent: generates verbal reflection given task+output+eval
class SelfReflectionAgent : public Actor {
public:
    SelfReflectionAgent(std::string id,
                         agent::AgentConfig cfg,
                         inference::SSEParser& sse);
protected:
    void handle_message(const ActorMessage& msg) override;
private:
    agent::AgentConfig cfg_;
    inference::SSEParser& sse_;
    std::vector<std::string> memory_;  // episodic reflection buffer
};

// Orchestrator: ties Actor + Evaluator + SelfReflection into a loop
AgentResult<std::string> run_reflexion_multiagent(
    const std::string& task,
    ActorRuntime& runtime,
    const std::string& actor_id,
    const std::string& evaluator_id,
    const std::string& reflector_id,
    int max_attempts,
    double success_threshold);

} // namespace multiagent
```

---

## 8. Parallel Map-Reduce

```cpp
// src/multiagent/mapreduce.hpp
#pragma once
#include <vector>
#include <string>
#include <boost/cobalt.hpp>
#include "agent/agent_error.hpp"

namespace multiagent {

namespace cobalt = boost::cobalt;

// Parallel map: send each subtask to a different agent concurrently
cobalt::task<std::vector<AgentResult<std::string>>> parallel_map(
    const std::vector<std::string>& subtasks,
    const std::vector<std::string>& agent_ids,
    ActorRuntime& runtime,
    std::chrono::seconds timeout);

// Reduce: aggregate results using an LLM summarizer
cobalt::task<AgentResult<std::string>> reduce(
    const std::string& original_task,
    const std::vector<std::string>& results,
    inference::SSEParser& sse,
    const agent::AgentConfig& cfg);

// Full map-reduce pipeline
cobalt::task<AgentResult<std::string>> map_reduce(
    const std::string& task,
    std::function<std::vector<std::string>(const std::string&)> decompose_fn,
    const std::vector<std::string>& agent_ids,
    ActorRuntime& runtime,
    inference::SSEParser& sse,
    const agent::AgentConfig& cfg);

} // namespace multiagent
```

---

## 9. Agent Lifecycle State Machine

```
         spawn()
            │
            ▼
         [Spawned]
            │ start()
            ▼
         [Running] ──── abort_requested ──── [Aborted]
            │                                    │
            │ max_iterations / error             │
            ▼                                    │
         [Completing]                            │
            │                                    │
            ├── success → [Completed]            │
            └── failure → [Failed] ◄─────────────┘

All terminal states notify the supervisor and publish to EventBus.
```

```cpp
enum class ActorLifecycleState {
    Spawned,
    Running,
    Completing,
    Completed,
    Failed,
    Aborted
};

struct ActorStatus {
    std::string         id;
    ActorLifecycleState state;
    std::string         last_error;
    std::chrono::system_clock::time_point started_at;
    std::chrono::system_clock::time_point finished_at;
    nlohmann::json      result;
};
```

---

## 10. Shared Context and Blackboard

For agents that need to share intermediate results (blackboard architecture):

```cpp
// src/multiagent/blackboard.hpp
#pragma once
#include <string>
#include <any>
#include <unordered_map>
#include <mutex>
#include <functional>

namespace multiagent {

// Thread-safe shared blackboard for multi-agent coordination
class Blackboard {
public:
    template<typename T>
    void write(const std::string& key, T value) {
        std::lock_guard lock(mutex_);
        data_[key] = std::move(value);
        notify_watchers(key);
    }

    template<typename T>
    std::optional<T> read(const std::string& key) const {
        std::lock_guard lock(mutex_);
        auto it = data_.find(key);
        if (it == data_.end()) return std::nullopt;
        try { return std::any_cast<T>(it->second); }
        catch (...) { return std::nullopt; }
    }

    bool has(const std::string& key) const {
        std::lock_guard lock(mutex_);
        return data_.count(key) > 0;
    }

    // Watch for changes to a key
    void watch(const std::string& key, std::function<void(const std::string&)> cb) {
        std::lock_guard lock(mutex_);
        watchers_[key].push_back(std::move(cb));
    }

private:
    void notify_watchers(const std::string& key) {
        if (auto it = watchers_.find(key); it != watchers_.end())
            for (const auto& cb : it->second) cb(key);
    }

    mutable std::mutex mutex_;
    std::unordered_map<std::string, std::any> data_;
    std::unordered_map<std::string, std::vector<std::function<void(const std::string&)>>> watchers_;
};

} // namespace multiagent
```
