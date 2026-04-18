# 08 — Self-Improvement Mechanisms

This document covers the full suite of self-improvement techniques:
Reflexion verbal RL (arXiv:2303.11366), ExpeL experience pool (arXiv:2308.10144),
CRITIC tool verification, SELF-REFINE (selfrefine.info), skill library
(Voyager pattern), and quality gates for experience selection.

---

## 1. Reflexion — Verbal Reinforcement Learning (arXiv:2303.11366)

Reflexion stores short verbal self-critiques in an episodic memory buffer
and prepends them to subsequent attempts. See `02-core-agent-loop.md` for
the full wrapper. This section covers the episodic memory store.

```cpp
// src/agent/episodic_memory.hpp
#pragma once
#include <string>
#include <vector>
#include <chrono>
#include <nlohmann/json.hpp>

namespace agent {

struct EpisodicEntry {
    std::string task;
    std::string outcome;          // "success" | "failure" | "partial"
    std::string reflection;       // verbal self-critique
    double      score = 0.0;
    int         attempt = 0;
    std::chrono::system_clock::time_point created_at;
    nlohmann::json metadata;
};

// In-process episodic memory (persisted via session serialization)
class EpisodicMemory {
public:
    void add(EpisodicEntry entry);

    // Retrieve reflections for a similar task
    std::vector<std::string> get_reflections_for(
        const std::string& task, std::size_t max = 3) const;

    // Get all successful trajectories for a task category
    std::vector<EpisodicEntry> successful_trajectories(
        const std::string& task_category = "") const;

    std::size_t size() const { return entries_.size(); }

    nlohmann::json to_json() const;
    static EpisodicMemory from_json(const nlohmann::json& j);

private:
    std::vector<EpisodicEntry> entries_;
};

} // namespace agent
```

```cpp
// src/agent/episodic_memory.cpp
#include "agent/episodic_memory.hpp"
#include <algorithm>

namespace agent {

void EpisodicMemory::add(EpisodicEntry entry) {
    entries_.push_back(std::move(entry));
}

std::vector<std::string> EpisodicMemory::get_reflections_for(
    const std::string& task, std::size_t max) const
{
    // Simple substring match — replace with embedding similarity in production
    std::vector<std::string> reflections;
    for (auto it = entries_.rbegin(); it != entries_.rend() && reflections.size() < max; ++it) {
        // Check task similarity (very rough heuristic)
        if (it->task.find(task.substr(0, 20)) != std::string::npos ||
            task.find(it->task.substr(0, 20)) != std::string::npos) {
            reflections.push_back(it->reflection);
        }
    }
    return reflections;
}

std::vector<EpisodicEntry> EpisodicMemory::successful_trajectories(
    const std::string& task_category) const
{
    std::vector<EpisodicEntry> result;
    for (const auto& e : entries_) {
        if (e.outcome != "success") continue;
        if (!task_category.empty() &&
            e.task.find(task_category) == std::string::npos) continue;
        result.push_back(e);
    }
    return result;
}

} // namespace agent
```

---

## 2. ExpeL — Experience Pool (arXiv:2308.10144)

ExpeL stores successful trajectories in a vector database and extracts
cross-trajectory insights. New tasks retrieve similar trajectories and
insights to bootstrap performance.

```cpp
// src/agent/expel.hpp
#pragma once
#include <string>
#include <vector>
#include "memory/recall_storage.hpp"
#include "agent/episodic_memory.hpp"
#include "inference/sse_parser.hpp"

namespace agent {

struct Trajectory {
    std::string task;
    std::vector<std::string> steps;   // Thought/Action/Observation sequence
    std::string final_answer;
    double      score;
    std::vector<std::string> tools_used;
    nlohmann::json metadata;
};

struct Insight {
    std::string content;       // Extracted cross-trajectory pattern
    int         support_count; // How many trajectories support this
    double      confidence;
};

class ExperiencePool {
public:
    ExperiencePool(memory::RecallStorage& storage,
                    inference::SSEParser& sse,
                    std::function<std::vector<float>(const std::string&)> embedder,
                    double quality_threshold = 0.6);

    // Add a trajectory to the pool (only if score >= quality_threshold)
    AgentResult<bool> add_trajectory(Trajectory traj);

    // Retrieve similar trajectories for a new task
    AgentResult<std::vector<Trajectory>> retrieve_similar(
        const std::string& task, std::size_t top_k = 3);

    // Extract insights from the pool using LLM
    AgentResult<std::vector<Insight>> extract_insights(std::size_t max_insights = 5);

    // Format retrieved trajectories + insights as context for a new task
    std::string format_for_context(
        const std::vector<Trajectory>& trajectories,
        const std::vector<Insight>& insights) const;

    std::size_t size() const { return trajectories_.size(); }

private:
    memory::RecallStorage&  storage_;
    inference::SSEParser&   sse_;
    std::function<std::vector<float>(const std::string&)> embedder_;
    double                  quality_threshold_;
    std::vector<Trajectory> trajectories_;  // local cache
};

} // namespace agent
```

```cpp
// src/agent/expel.cpp
#include "agent/expel.hpp"
#include <spdlog/spdlog.h>

namespace agent {

ExperiencePool::ExperiencePool(
    memory::RecallStorage& storage,
    inference::SSEParser& sse,
    std::function<std::vector<float>(const std::string&)> embedder,
    double quality_threshold)
    : storage_(storage)
    , sse_(sse)
    , embedder_(std::move(embedder))
    , quality_threshold_(quality_threshold)
{}

AgentResult<bool> ExperiencePool::add_trajectory(Trajectory traj) {
    if (traj.score < quality_threshold_) {
        spdlog::debug("[expel] rejected trajectory (score {:.2f} < {:.2f})",
                      traj.score, quality_threshold_);
        return false;
    }

    // Create searchable text: task + key steps
    std::string search_text = traj.task + "\n";
    for (const auto& step : traj.steps)
        search_text += step + "\n";

    auto emb    = embedder_(search_text);
    nlohmann::json meta = {
        {"task",      traj.task},
        {"score",     traj.score},
        {"tools",     traj.tools_used},
        {"step_count", traj.steps.size()},
    };

    auto add_result = storage_.add(emb, traj.task, meta);
    if (!add_result) return std::unexpected(add_result.error());

    trajectories_.push_back(std::move(traj));
    spdlog::info("[expel] added trajectory #{} (score={:.2f})", trajectories_.size(), traj.score);
    return true;
}

AgentResult<std::vector<Trajectory>> ExperiencePool::retrieve_similar(
    const std::string& task, std::size_t top_k)
{
    auto query_emb = embedder_(task);
    auto hits      = storage_.search(query_emb, top_k);
    if (!hits) return std::unexpected(hits.error());

    std::vector<Trajectory> result;
    for (const auto& hit : *hits) {
        // Find matching trajectory in local cache by task name
        for (const auto& traj : trajectories_) {
            if (traj.task == hit.text) {
                result.push_back(traj);
                break;
            }
        }
    }
    return result;
}

AgentResult<std::vector<Insight>> ExperiencePool::extract_insights(std::size_t max_insights) {
    if (trajectories_.empty()) return std::vector<Insight>{};

    // Sample trajectories for insight extraction
    std::string traj_summary;
    for (std::size_t i = 0; i < std::min(trajectories_.size(), std::size_t(10)); ++i) {
        const auto& t = trajectories_[i];
        traj_summary += std::format("Task: {}\nSteps: {}\nScore: {:.2f}\n\n",
            t.task,
            t.steps.empty() ? "(none)" : t.steps[0] + "...",
            t.score);
    }

    nlohmann::json req = {
        {"model",  "gpt-4o"},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content",
            std::format(
                "Analyze these {} successful agent trajectories and extract up to {} "
                "generalizable insights or best practices that apply across tasks.\n\n"
                "Return as JSON array: [{{\"content\": \"...\", \"confidence\": 0.0-1.0}}]\n\n"
                "Trajectories:\n{}",
                trajectories_.size(), max_insights, traj_summary)
        }}}}
    };

    std::string raw = sse_.accumulate(req);
    std::vector<Insight> insights;
    try {
        auto j = nlohmann::json::parse(raw);
        for (const auto& item : j) {
            Insight ins;
            ins.content    = item["content"].get<std::string>();
            ins.confidence = item.value("confidence", 0.7);
            ins.support_count = 1;
            insights.push_back(std::move(ins));
        }
    } catch (...) {}

    return insights;
}

std::string ExperiencePool::format_for_context(
    const std::vector<Trajectory>& trajectories,
    const std::vector<Insight>& insights) const
{
    std::string ctx;

    if (!insights.empty()) {
        ctx += "### Learned Insights\n";
        for (const auto& ins : insights)
            ctx += std::format("- {} (confidence: {:.0f}%)\n",
                ins.content, ins.confidence * 100);
        ctx += "\n";
    }

    if (!trajectories.empty()) {
        ctx += "### Similar Past Trajectories\n";
        for (const auto& traj : trajectories) {
            ctx += std::format("**Task**: {}\n**Score**: {:.2f}\n**Steps**:\n",
                traj.task, traj.score);
            for (const auto& step : traj.steps)
                ctx += "  " + step + "\n";
            ctx += "\n";
        }
    }

    return ctx;
}

} // namespace agent
```

---

## 3. CRITIC — Tool-Verified Self-Correction

CRITIC automatically runs verification tools after generation to check claims:

```cpp
// src/agent/critic.hpp
#pragma once
#include <string>
#include <vector>
#include <functional>
#include "agent/agent_error.hpp"
#include "tools/tool_registry.hpp"

namespace agent {

struct Critique {
    std::string  claim;
    bool         verified;
    std::string  evidence;    // tool output supporting/refuting the claim
    std::string  correction;  // suggested fix if claim is wrong
};

struct CriticConfig {
    std::vector<std::string> verification_tools = { "web_search", "bash", "calculator" };
    int                      max_claims         = 5;
    bool                     auto_correct       = true;
};

// Extract verifiable claims from text and verify them using tools
AgentResult<std::vector<Critique>> critic_verify(
    const std::string& generated_text,
    tools::ToolRegistry& tools,
    inference::SSEParser& sse,
    const CriticConfig& cfg,
    const ToolContext& tool_ctx);

// Apply corrections to produce a revised text
AgentResult<std::string> apply_corrections(
    const std::string& original_text,
    const std::vector<Critique>& critiques,
    inference::SSEParser& sse);

} // namespace agent
```

```cpp
// src/agent/critic.cpp
#include "agent/critic.hpp"
#include "tools/dispatcher.hpp"

namespace agent {

static std::vector<std::string> extract_claims(
    const std::string& text, inference::SSEParser& sse)
{
    nlohmann::json req = {
        {"model",  "gpt-4o-mini"},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content",
            "Extract the factual claims from the following text that can be "
            "verified using external tools (web search, code execution, calculations).\n\n"
            "Return JSON array of claim strings (max 5):\n\n" + text
        }}}}
    };
    std::string raw = sse.accumulate(req);
    std::vector<std::string> claims;
    try {
        auto j = nlohmann::json::parse(raw);
        for (const auto& c : j) claims.push_back(c.get<std::string>());
    } catch (...) {}
    return claims;
}

AgentResult<std::vector<Critique>> critic_verify(
    const std::string& generated_text,
    tools::ToolRegistry& registry,
    inference::SSEParser& sse,
    const CriticConfig& cfg,
    const ToolContext& tool_ctx)
{
    auto claims = extract_claims(generated_text, sse);
    std::vector<Critique> critiques;

    for (std::size_t i = 0;
         i < std::min(claims.size(), static_cast<std::size_t>(cfg.max_claims));
         ++i)
    {
        const auto& claim = claims[i];
        Critique crit;
        crit.claim = claim;

        // Decide which tool to use for verification
        nlohmann::json tool_req = {
            {"model",  "gpt-4o-mini"},
            {"stream", false},
            {"messages", {{{"role","user"}, {"content",
                std::format("To verify the claim: \"{}\"\n\n"
                    "Available tools: {}\n\n"
                    "Respond as JSON: {{\"tool\": \"tool_name\", \"args\": {{...}}}}",
                    claim,
                    [&]{ std::string s;
                         for (const auto& t : cfg.verification_tools) s += t + ", ";
                         return s; }())
            }}}}
        };

        std::string tool_resp = sse.accumulate(tool_req);
        try {
            auto j = nlohmann::json::parse(tool_resp);
            tools::ToolCall tc;
            tc.name      = j["tool"].get<std::string>();
            tc.arguments = j["args"];
            tc.call_id   = std::format("critic_{}", i);

            // Dispatch verification tool
            agent::AgentContext dummy_ctx;
            auto result = tools::dispatch(registry, tc, dummy_ctx);
            if (result) {
                crit.evidence = result->content;

                // Judge whether evidence supports or refutes the claim
                nlohmann::json judge_req = {
                    {"model",  "gpt-4o-mini"},
                    {"stream", false},
                    {"messages", {{{"role","user"}, {"content",
                        std::format("Claim: {}\nEvidence: {}\n\n"
                            "Does the evidence support or refute the claim? "
                            "Respond as JSON: {{\"verified\": true/false, \"correction\": \"...\"}}\n"
                            "If verified, correction can be empty.",
                            claim, result->content.substr(0, 500))
                    }}}}
                };
                std::string judge_raw = sse.accumulate(judge_req);
                try {
                    auto jj = nlohmann::json::parse(judge_raw);
                    crit.verified   = jj.value("verified", true);
                    crit.correction = jj.value("correction", "");
                } catch (...) { crit.verified = true; }
            }
        } catch (...) {
            crit.verified = true;  // cannot verify = assume true
        }

        critiques.push_back(std::move(crit));
    }

    return critiques;
}

AgentResult<std::string> apply_corrections(
    const std::string& original_text,
    const std::vector<Critique>& critiques,
    inference::SSEParser& sse)
{
    // Build correction notes
    std::string corrections;
    for (const auto& c : critiques) {
        if (!c.verified && !c.correction.empty())
            corrections += std::format("- Incorrect: \"{}\" → {}\n", c.claim, c.correction);
    }

    if (corrections.empty()) return original_text;

    nlohmann::json req = {
        {"model",  "gpt-4o"},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content",
            "Revise the following text to fix the identified inaccuracies. "
            "Keep the style and structure the same.\n\n"
            "Original:\n" + original_text + "\n\n"
            "Corrections needed:\n" + corrections
        }}}}
    };

    return sse.accumulate(req);
}

} // namespace agent
```

---

## 4. SELF-REFINE — Generate → Critique → Refine (selfrefine.info)

Full struct definitions (complementing the loop in 02-core-agent-loop.md):

```cpp
// src/agent/self_refine.hpp
#pragma once
#include <string>
#include <vector>
#include "agent/agent_error.hpp"
#include "inference/sse_parser.hpp"

namespace agent {

struct SelfRefineConfig {
    int         max_iterations = 3;
    std::string critique_prompt;   // optional custom critique prompt
    std::string refine_prompt;     // optional custom refinement prompt
    // Stop early if critique returns these phrases
    std::vector<std::string> stop_phrases = {
        "no issues", "looks good", "correct", "accurate", "STOP"
    };
};

AgentResult<std::string> self_refine_loop(
    const std::string& task,
    const std::string& initial_draft,
    const SelfRefineConfig& cfg,
    inference::SSEParser& sse,
    const std::string& model);

} // namespace agent
```

---

## 5. Skill Library — Voyager Pattern

The Voyager pattern (from Minecraft agents) stores successful behaviors as
"skills" — documented code snippets or prompt templates that are embedded and
retrieved by similarity for new tasks.

```cpp
// src/agent/skill_library.hpp
#pragma once
#include <string>
#include <vector>
#include <nlohmann/json.hpp>
#include "memory/recall_storage.hpp"
#include "agent/agent_error.hpp"

namespace agent {

struct Skill {
    std::string id;
    std::string name;
    std::string description;
    std::string implementation;   // code, prompt template, or procedure
    std::string language;         // "cpp", "bash", "prompt"
    std::vector<std::string> tags;
    double      success_rate = 1.0;
    int         use_count    = 0;
    std::string created_from_task;
};

class SkillLibrary {
public:
    SkillLibrary(memory::RecallStorage& index,
                  std::function<std::vector<float>(const std::string&)> embedder);

    // Store a new skill (from a successful task)
    AgentResult<std::string> store(Skill skill);

    // Retrieve skills relevant to a task
    AgentResult<std::vector<Skill>> retrieve(
        const std::string& task_description, std::size_t top_k = 3);

    // Update skill success rate based on outcome
    void update_success(const std::string& skill_id, bool success);

    // Generate a skill from a successful trajectory
    AgentResult<Skill> synthesize_skill(
        const std::string& task,
        const std::vector<std::string>& steps,
        inference::SSEParser& sse,
        const std::string& model);

    std::size_t size() const { return skills_.size(); }

    nlohmann::json to_json() const;

private:
    memory::RecallStorage& index_;
    std::function<std::vector<float>(const std::string&)> embedder_;
    std::flat_map<std::string, Skill> skills_;
};

} // namespace agent
```

```cpp
// src/agent/skill_library.cpp
#include "agent/skill_library.hpp"
#include <format>

namespace agent {

SkillLibrary::SkillLibrary(
    memory::RecallStorage& index,
    std::function<std::vector<float>(const std::string&)> embedder)
    : index_(index), embedder_(std::move(embedder)) {}

AgentResult<std::string> SkillLibrary::store(Skill skill) {
    std::string search_text = skill.name + ": " + skill.description;
    auto emb = embedder_(search_text);

    nlohmann::json meta = {
        {"name",        skill.name},
        {"language",    skill.language},
        {"description", skill.description},
    };

    auto add_result = index_.add(emb, skill.id, meta);
    if (!add_result) return std::unexpected(add_result.error());

    skill.id = skill.id.empty()
        ? std::format("skill_{}", skills_.size())
        : skill.id;

    skills_[skill.id] = std::move(skill);
    return skill.id;
}

AgentResult<std::vector<Skill>> SkillLibrary::retrieve(
    const std::string& task_description, std::size_t top_k)
{
    auto emb  = embedder_(task_description);
    auto hits = index_.search(emb, top_k);
    if (!hits) return std::unexpected(hits.error());

    std::vector<Skill> result;
    for (const auto& hit : *hits) {
        auto it = skills_.find(hit.text);  // hit.text = skill ID
        if (it != skills_.end()) result.push_back(it->second);
    }
    return result;
}

void SkillLibrary::update_success(const std::string& skill_id, bool success) {
    auto it = skills_.find(skill_id);
    if (it == skills_.end()) return;
    auto& skill = it->second;
    skill.use_count++;
    // Exponential moving average of success rate
    double alpha = 0.1;
    skill.success_rate = alpha * (success ? 1.0 : 0.0) + (1.0 - alpha) * skill.success_rate;
}

AgentResult<Skill> SkillLibrary::synthesize_skill(
    const std::string& task,
    const std::vector<std::string>& steps,
    inference::SSEParser& sse,
    const std::string& model)
{
    std::string steps_text;
    for (std::size_t i = 0; i < steps.size(); ++i)
        steps_text += std::format("  {}. {}\n", i + 1, steps[i]);

    nlohmann::json req = {
        {"model",  model},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content",
            "Extract a reusable skill from this successful task execution.\n\n"
            "Task: " + task + "\n\n"
            "Steps:\n" + steps_text + "\n\n"
            "Create a skill that captures the generalizable pattern. "
            "Return JSON: {\"name\": \"...\", \"description\": \"...\", "
            "\"implementation\": \"...\", \"language\": \"prompt\", \"tags\": [...]}"
        }}}}
    };

    std::string raw = sse.accumulate(req);
    try {
        auto j = nlohmann::json::parse(raw);
        Skill skill;
        skill.id             = std::format("skill_{}", skills_.size() + 1);
        skill.name           = j.value("name", "unnamed_skill");
        skill.description    = j.value("description", "");
        skill.implementation = j.value("implementation", "");
        skill.language       = j.value("language", "prompt");
        if (j.contains("tags"))
            for (const auto& t : j["tags"])
                skill.tags.push_back(t.get<std::string>());
        skill.created_from_task = task;
        return skill;
    } catch (...) {
        return std::unexpected(AgentError{
            AgentErrorCode::LLMParseError, "Failed to synthesize skill"
        });
    }
}

} // namespace agent
```

---

## 6. Quality Gate

The quality gate scores agent outputs before storing them in the experience pool
or skill library. Only outputs above threshold are persisted.

```cpp
// src/agent/quality_gate.hpp
#pragma once
#include <string>
#include <functional>
#include <vector>
#include "agent/agent_error.hpp"
#include "inference/sse_parser.hpp"

namespace agent {

struct QualityScore {
    double  overall;       // 0.0 - 1.0
    double  correctness;
    double  completeness;
    double  efficiency;
    std::string feedback;
};

struct QualityGateConfig {
    double  pass_threshold  = 0.7;
    bool    use_llm_judge   = true;
    bool    use_heuristics  = true;
    // Custom scoring functions
    std::vector<std::function<double(const std::string&)>> heuristic_fns;
};

// Score an output using LLM-as-judge + optional heuristics
AgentResult<QualityScore> score_output(
    const std::string& task,
    const std::string& output,
    inference::SSEParser& sse,
    const QualityGateConfig& cfg);

// Determine if an output passes the quality gate
bool passes_gate(const QualityScore& score, const QualityGateConfig& cfg);

} // namespace agent
```

```cpp
// src/agent/quality_gate.cpp
#include "agent/quality_gate.hpp"

namespace agent {

AgentResult<QualityScore> score_output(
    const std::string& task,
    const std::string& output,
    inference::SSEParser& sse,
    const QualityGateConfig& cfg)
{
    QualityScore score{};

    if (cfg.use_llm_judge) {
        nlohmann::json req = {
            {"model",  "gpt-4o"},
            {"stream", false},
            {"messages", {{{"role","user"}, {"content",
                "Evaluate the quality of this agent output on a scale of 0-10 "
                "for each dimension.\n\n"
                "Task: " + task + "\n\n"
                "Output: " + output.substr(0, 1000) + "\n\n"
                "Return JSON: {\"correctness\": 0-10, \"completeness\": 0-10, "
                "\"efficiency\": 0-10, \"feedback\": \"...\"}"
            }}}}
        };

        std::string raw = sse.accumulate(req);
        try {
            auto j = nlohmann::json::parse(raw);
            score.correctness  = j.value("correctness",  7.0) / 10.0;
            score.completeness = j.value("completeness", 7.0) / 10.0;
            score.efficiency   = j.value("efficiency",   7.0) / 10.0;
            score.feedback     = j.value("feedback", "");
            score.overall = (score.correctness + score.completeness + score.efficiency) / 3.0;
        } catch (...) {
            score.overall = 0.7;  // default if judge fails
        }
    }

    // Apply heuristics
    if (cfg.use_heuristics && !cfg.heuristic_fns.empty()) {
        double heuristic_total = 0.0;
        for (const auto& fn : cfg.heuristic_fns)
            heuristic_total += fn(output);
        double heuristic_score = heuristic_total / cfg.heuristic_fns.size();
        // Blend LLM score with heuristics
        score.overall = 0.7 * score.overall + 0.3 * heuristic_score;
    }

    return score;
}

bool passes_gate(const QualityScore& score, const QualityGateConfig& cfg) {
    return score.overall >= cfg.pass_threshold;
}

} // namespace agent
```

---

## 7. Incremental Improvement Pipeline

```cpp
// src/agent/improvement_pipeline.hpp
#pragma once
#include "agent/expel.hpp"
#include "agent/skill_library.hpp"
#include "agent/episodic_memory.hpp"
#include "agent/quality_gate.hpp"
#include "agent/critic.hpp"
#include "agent/self_refine.hpp"

namespace agent {

// Wires all self-improvement mechanisms together.
// Call after each agent run to update the learning state.
struct ImprovementPipeline {
    ExperiencePool*  experience_pool  = nullptr;
    SkillLibrary*    skill_library    = nullptr;
    EpisodicMemory*  episodic_memory  = nullptr;
    QualityGateConfig quality_cfg;
    SelfRefineConfig  refine_cfg;
    CriticConfig      critic_cfg;

    // Process a completed agent run
    AgentResult<void> process(
        const std::string& task,
        const std::string& output,
        const std::vector<std::string>& steps,
        const std::vector<std::string>& tools_used,
        inference::SSEParser& sse,
        tools::ToolRegistry& tool_registry,
        const ToolContext& tool_ctx,
        const std::string& model);
};

} // namespace agent
```

```cpp
// src/agent/improvement_pipeline.cpp
#include "agent/improvement_pipeline.hpp"
#include <spdlog/spdlog.h>

namespace agent {

AgentResult<void> ImprovementPipeline::process(
    const std::string& task,
    const std::string& output,
    const std::vector<std::string>& steps,
    const std::vector<std::string>& tools_used,
    inference::SSEParser& sse,
    tools::ToolRegistry& tool_registry,
    const ToolContext& tool_ctx,
    const std::string& model)
{
    // 1. CRITIC: verify factual claims
    auto critiques = critic_verify(output, tool_registry, sse, critic_cfg, tool_ctx);
    std::string verified_output = output;
    if (critiques) {
        auto corrected = apply_corrections(output, *critiques, sse);
        if (corrected) verified_output = *corrected;
    }

    // 2. SELF-REFINE: polish the output
    auto refined = self_refine_loop(task, verified_output, refine_cfg, sse, model);
    if (refined) verified_output = *refined;

    // 3. Quality gate
    auto score_result = score_output(task, verified_output, sse, quality_cfg);
    if (!score_result) return std::unexpected(score_result.error());

    double score = score_result->overall;
    spdlog::info("[improvement] quality score: {:.2f} (threshold: {:.2f})",
                 score, quality_cfg.pass_threshold);

    bool passed = passes_gate(*score_result, quality_cfg);
    std::string outcome = passed ? "success" : "partial";

    // 4. Store in episodic memory
    if (episodic_memory) {
        episodic_memory->add({
            task, outcome,
            score_result->feedback,
            score, 0,
            std::chrono::system_clock::now()
        });
    }

    // 5. Add to experience pool (only if quality sufficient)
    if (experience_pool && passed) {
        Trajectory traj;
        traj.task         = task;
        traj.steps        = steps;
        traj.final_answer = verified_output;
        traj.score        = score;
        traj.tools_used   = tools_used;
        auto add_r = experience_pool->add_trajectory(std::move(traj));
        if (!add_r) spdlog::warn("[improvement] experience pool add failed");
    }

    // 6. Synthesize skill if output quality is high
    if (skill_library && score >= 0.85) {
        auto skill = skill_library->synthesize_skill(task, steps, sse, model);
        if (skill) {
            auto store_r = skill_library->store(std::move(*skill));
            if (!store_r) spdlog::warn("[improvement] skill store failed");
        }
    }

    return {};
}

} // namespace agent
```
