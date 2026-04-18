# 10 — Prompt Engineering for C++ Agents

This document covers all prompt construction patterns: SystemPromptBuilder,
dynamic boundary caching, ReAct templates, Reflexion critique, HyDE, KG extraction,
GraphRAG community summaries, SELF-REFINE critique, and OpenAI tool description format.

---

## 1. SystemPromptBuilder

The system prompt is assembled from multiple sections. Sections before the
"dynamic boundary" can be cached (e.g., Anthropic prompt caching); sections
after are freshly inserted each turn.

```cpp
// src/prompts/system_prompt.hpp
#pragma once
#include <string>
#include <vector>
#include <functional>
#include "session/session.hpp"

namespace prompts {

enum class SectionKind {
    Static,   // Stable content — suitable for prompt caching
    Dynamic,  // Changes each turn (memory state, date, workspace info)
};

struct PromptSection {
    std::string  name;
    SectionKind  kind;
    std::string  content;
    int          priority = 0;  // lower = appears earlier
};

class SystemPromptBuilder {
public:
    SystemPromptBuilder& add_section(PromptSection section);

    // Add built-in sections
    SystemPromptBuilder& with_intro(const std::string& agent_name,
                                     const std::string& agent_purpose);
    SystemPromptBuilder& with_date();
    SystemPromptBuilder& with_workspace_context(const std::string& workspace_root);
    SystemPromptBuilder& with_tools_overview(const std::vector<std::string>& tool_names);
    SystemPromptBuilder& with_memory_state(const std::string& memory_content);
    SystemPromptBuilder& with_react_instructions();
    SystemPromptBuilder& with_output_format(const std::string& format_spec);
    SystemPromptBuilder& with_environment_info();
    SystemPromptBuilder& with_project_context(const std::string& context_text);

    // Build final prompt string
    // dynamic_boundary_after: section name after which content is dynamic
    std::string build(const std::string& dynamic_boundary_after = "tools_overview") const;

    // Build with cache markers (for Anthropic API with prompt caching)
    std::string build_with_cache_markers() const;

private:
    std::vector<PromptSection> sections_;
};

} // namespace prompts
```

```cpp
// src/prompts/system_prompt.cpp
#include "prompts/system_prompt.hpp"
#include <algorithm>
#include <format>
#include <chrono>

namespace prompts {

SystemPromptBuilder& SystemPromptBuilder::add_section(PromptSection section) {
    sections_.push_back(std::move(section));
    return *this;
}

SystemPromptBuilder& SystemPromptBuilder::with_intro(
    const std::string& agent_name,
    const std::string& agent_purpose)
{
    return add_section({
        "intro", SectionKind::Static,
        std::format(
            "You are {}, {}.\n\n"
            "You are a highly capable AI assistant that operates in a ReAct loop: "
            "you reason step by step (Thought), take actions using tools (Action), "
            "and observe the results (Observation) until you reach a final answer.",
            agent_name, agent_purpose),
        0
    });
}

SystemPromptBuilder& SystemPromptBuilder::with_date() {
    auto now = std::chrono::system_clock::now();
    std::string date = std::format("{:%Y-%m-%d}", now);
    return add_section({
        "date", SectionKind::Dynamic,
        std::format("Today's date is {}.", date),
        100
    });
}

SystemPromptBuilder& SystemPromptBuilder::with_workspace_context(
    const std::string& workspace_root)
{
    return add_section({
        "workspace", SectionKind::Dynamic,
        std::format(
            "Your workspace root is: {}\n"
            "All file operations are relative to this directory.\n"
            "Do not access files outside this root.",
            workspace_root),
        110
    });
}

SystemPromptBuilder& SystemPromptBuilder::with_react_instructions() {
    return add_section({
        "react_format", SectionKind::Static,
        R"(## ReAct Format

For each step, follow this format exactly:

Thought: [Your reasoning about what to do next]
Action: [tool_name]
Action Input: {"param1": "value1", "param2": "value2"}

After observing the tool result:

Thought: [Your reasoning about the observation]
Action: [next tool or final answer]

When you have a complete answer:

Thought: [Your final reasoning]
Final Answer: [Your complete response to the user]

Rules:
- Always include a Thought before an Action or Final Answer
- Action Input must be valid JSON
- Only use tools that are available in your tool registry
- If a tool fails, reason about why and try an alternative approach)",
        50
    });
}

SystemPromptBuilder& SystemPromptBuilder::with_memory_state(
    const std::string& memory_content)
{
    if (memory_content.empty()) return *this;
    return add_section({
        "memory_state", SectionKind::Dynamic,
        "## Current Working Memory\n\n" + memory_content,
        120
    });
}

SystemPromptBuilder& SystemPromptBuilder::with_tools_overview(
    const std::vector<std::string>& tool_names)
{
    std::string tools_list;
    for (const auto& t : tool_names)
        tools_list += "  - " + t + "\n";

    return add_section({
        "tools_overview", SectionKind::Static,
        "## Available Tools\n\n" + tools_list +
        "\nUse `Action: tool_name` to invoke a tool.",
        40
    });
}

SystemPromptBuilder& SystemPromptBuilder::with_project_context(
    const std::string& context_text)
{
    return add_section({
        "project_context", SectionKind::Static,
        "## Project Context\n\n" + context_text,
        30
    });
}

SystemPromptBuilder& SystemPromptBuilder::with_environment_info() {
    return add_section({
        "environment", SectionKind::Static,
        "## Environment\n"
        "- OS: Linux (x86_64)\n"
        "- Shell: bash\n"
        "- C++ Standard: C++23\n"
        "- Build system: CMake 3.28+\n",
        60
    });
}

std::string SystemPromptBuilder::build(const std::string& dynamic_boundary_after) const {
    // Sort by priority
    auto sorted = sections_;
    std::sort(sorted.begin(), sorted.end(),
              [](const auto& a, const auto& b) { return a.priority < b.priority; });

    std::string result;
    bool past_boundary = false;

    for (const auto& section : sorted) {
        if (!past_boundary && section.name == dynamic_boundary_after)
            past_boundary = true;

        if (!result.empty()) result += "\n\n";
        result += section.content;
    }

    return result;
}

} // namespace prompts
```

---

## 2. ReAct Templates

```cpp
// src/prompts/react_templates.hpp
#pragma once
#include <string>
#include <vector>

namespace prompts::react {

// Few-shot ReAct example for task context
inline std::string few_shot_example() {
    return R"(
## Example ReAct Interaction

User: What files are in the current directory?

Thought: The user wants to list the directory contents. I'll use the bash tool.
Action: bash
Action Input: {"command": "ls -la"}

Observation: total 48
drwxr-xr-x  8 user user 4096 Apr 17 10:00 .
drwxr-xr-x 12 user user 4096 Apr 17 09:00 ..
-rw-r--r--  1 user user 1234 Apr 17 10:00 CMakeLists.txt
drwxr-xr-x  4 user user 4096 Apr 17 10:00 src
drwxr-xr-x  2 user user 4096 Apr 17 10:00 tests

Thought: I can see the directory structure. The user's question is answered.
Final Answer: The current directory contains:
- CMakeLists.txt (1234 bytes)
- src/ directory
- tests/ directory
)";
}

// Template for when the agent needs to search before answering
inline std::string search_first_template(const std::string& query) {
    return std::format(
        "To answer: \"{}\"\n\n"
        "First search for relevant information before responding.",
        query);
}

// Template for multi-step coding tasks
inline std::string coding_task_template(
    const std::string& task,
    const std::string& language)
{
    return std::format(
        "Programming task: {}\n\n"
        "Language: {}\n\n"
        "Approach:\n"
        "1. Read relevant existing files first\n"
        "2. Understand the codebase structure\n"
        "3. Write or modify code\n"
        "4. Test the implementation\n"
        "5. Report the result",
        task, language);
}

} // namespace prompts::react
```

---

## 3. Reflexion Self-Reflection Prompt

```cpp
// src/prompts/reflexion_prompt.hpp
#pragma once
#include <string>
#include <vector>

namespace prompts::reflexion {

inline std::string make_reflection_prompt(
    const std::string& task,
    const std::string& failed_attempt,
    const std::string& failure_reason,
    const std::vector<std::string>& prior_reflections = {})
{
    std::string prior_str;
    if (!prior_reflections.empty()) {
        prior_str = "\nPrior reflections:\n";
        for (std::size_t i = 0; i < prior_reflections.size(); ++i)
            prior_str += std::format("  [{}] {}\n", i + 1, prior_reflections[i]);
    }

    return std::format(
        "You are a self-reflective AI agent analyzing a failed attempt.\n\n"
        "Task: {}\n\n"
        "Failed attempt summary:\n{}\n\n"
        "Failure reason: {}{}\n\n"
        "Write a concise reflection (3-5 sentences) that:\n"
        "1. Identifies the specific mistake or gap in reasoning\n"
        "2. Explains why the approach failed\n"
        "3. Proposes a concrete alternative strategy\n"
        "4. Notes any information that was missing or misunderstood\n\n"
        "Be specific and actionable. This reflection will be prepended to the next attempt.",
        task, failed_attempt, failure_reason, prior_str);
}

inline std::string make_evaluator_prompt(
    const std::string& task,
    const std::string& output,
    const std::string& rubric)
{
    return std::format(
        "Evaluate this agent output against the rubric.\n\n"
        "Task: {}\n\n"
        "Output:\n{}\n\n"
        "Rubric:\n{}\n\n"
        "Score each dimension 0-10 and explain. "
        "Return JSON: {{\"correctness\": N, \"completeness\": N, "
        "\"efficiency\": N, \"overall\": N, \"feedback\": \"...\"}}",
        task, output, rubric);
}

} // namespace prompts::reflexion
```

---

## 4. HyDE — Hypothetical Document Prompt (arXiv:2212.10496)

```cpp
// src/prompts/hyde_prompt.hpp
#pragma once
#include <string>

namespace prompts::hyde {

inline std::string make_hypothetical_doc_prompt(
    const std::string& query,
    const std::string& domain = "")
{
    std::string domain_ctx = domain.empty()
        ? ""
        : std::format("This document is from the {} domain. ", domain);

    return std::format(
        "{}Write a detailed, factual paragraph that would directly answer "
        "the following question. Write as if you are an authoritative document "
        "that contains the exact answer. Be specific and include relevant technical "
        "details, numbers, names, and dates where appropriate.\n\n"
        "Question: {}",
        domain_ctx, query);
}

// Multi-hypothetical: generate N different hypothetical docs
inline std::string make_multi_hyp_prompt(
    const std::string& query, int n)
{
    return std::format(
        "Generate {} different hypothetical document paragraphs that could "
        "each answer the question from a different angle or perspective. "
        "Return as JSON array of strings.\n\n"
        "Question: {}",
        n, query);
}

} // namespace prompts::hyde
```

---

## 5. Knowledge Graph Triple Extraction

```cpp
// src/prompts/kg_prompts.hpp
#pragma once
#include <string>
#include <vector>

namespace prompts::kg {

inline std::string make_triple_extraction_prompt(
    const std::string& text,
    const std::vector<std::string>& known_entity_types = {})
{
    std::string entity_types_hint;
    if (!known_entity_types.empty()) {
        entity_types_hint = "Known entity types: ";
        for (const auto& t : known_entity_types)
            entity_types_hint += t + ", ";
        entity_types_hint += "\n\n";
    }

    return std::format(
        "Extract all factual triples from the following text.\n\n"
        "{}Rules:\n"
        "- Each triple: (subject_entity, predicate, object_entity)\n"
        "- Normalize entity names to canonical form\n"
        "- Use specific predicates: 'is_a', 'has_property', 'located_in', "
        "'authored_by', 'depends_on', 'calls', 'implements', 'part_of'\n"
        "- Include only verifiable facts, not opinions\n\n"
        "Return JSON array:\n"
        "[{{\"subject\": \"...\", \"predicate\": \"...\", \"object\": \"...\"}}]\n\n"
        "Text:\n{}",
        entity_types_hint, text);
}

inline std::string make_entity_description_prompt(
    const std::string& entity_name,
    const std::string& context_text)
{
    return std::format(
        "Based on the following context, write a 1-2 sentence description "
        "of the entity '{}'. Focus on what it is and its primary role.\n\n"
        "Context:\n{}",
        entity_name, context_text.substr(0, 1000));
}

inline std::string make_community_summary_prompt(
    const std::vector<std::string>& entity_descriptions,
    const std::vector<std::string>& key_relations)
{
    std::string entities_str;
    for (const auto& e : entity_descriptions)
        entities_str += "- " + e + "\n";

    std::string rels_str;
    for (const auto& r : key_relations)
        rels_str += "- " + r + "\n";

    return std::format(
        "Summarize this group of related entities as a coherent knowledge cluster "
        "in 2-3 sentences. What is the unifying theme or purpose of this group?\n\n"
        "Entities:\n{}\n"
        "Key relationships:\n{}",
        entities_str, rels_str);
}

} // namespace prompts::kg
```

---

## 6. SELF-REFINE Critique Prompt

```cpp
// src/prompts/self_refine_prompt.hpp
#pragma once
#include <string>

namespace prompts::self_refine {

// Generic critique prompt — can be specialized per domain
inline std::string make_critique_prompt(
    const std::string& task,
    const std::string& draft,
    const std::string& domain = "general")
{
    std::string domain_focus;
    if (domain == "code") {
        domain_focus =
            "Focus on:\n"
            "- Correctness (does it compile and run correctly?)\n"
            "- Edge cases not handled\n"
            "- Performance issues\n"
            "- Security vulnerabilities\n"
            "- Missing error handling\n"
            "- Code style and readability\n";
    } else if (domain == "analysis") {
        domain_focus =
            "Focus on:\n"
            "- Logical consistency\n"
            "- Missing evidence or sources\n"
            "- Faulty assumptions\n"
            "- Alternative interpretations not considered\n"
            "- Incomplete coverage of the topic\n";
    } else {
        domain_focus =
            "Focus on:\n"
            "- Accuracy and correctness\n"
            "- Completeness\n"
            "- Clarity and conciseness\n"
            "- Any missing important aspects\n";
    }

    return std::format(
        "Review the following draft output for the given task.\n\n"
        "Task: {}\n\n"
        "Draft:\n{}\n\n"
        "{}\n"
        "If the output is satisfactory, respond with: 'STOP'\n"
        "Otherwise, provide a specific critique listing what needs improvement.",
        task, draft, domain_focus);
}

inline std::string make_refine_prompt(
    const std::string& task,
    const std::string& draft,
    const std::string& critique)
{
    return std::format(
        "Improve the following draft based on the critique provided.\n\n"
        "Task: {}\n\n"
        "Original draft:\n{}\n\n"
        "Critique:\n{}\n\n"
        "Produce an improved version that addresses all critique points. "
        "Maintain the same format and length unless the critique requires changes.",
        task, draft, critique);
}

} // namespace prompts::self_refine
```

---

## 7. CRAG Relevance Scoring Prompt

```cpp
// src/prompts/crag_prompts.hpp
#pragma once
#include <string>

namespace prompts::crag {

inline std::string make_relevance_prompt(
    const std::string& query,
    const std::string& document_snippet)
{
    return std::format(
        "Rate the relevance of this document snippet to the query.\n\n"
        "Query: {}\n\n"
        "Document snippet: {}\n\n"
        "Rate 0-10 where:\n"
        "  0-3: Irrelevant or misleading\n"
        "  4-6: Partially relevant, tangential\n"
        "  7-10: Directly relevant and useful\n\n"
        "Respond with ONLY a single number (0-10).",
        query, document_snippet.substr(0, 600));
}

inline std::string make_knowledge_refinement_prompt(
    const std::string& query,
    const std::string& web_search_results)
{
    return std::format(
        "Given these web search results, extract and organize the key facts "
        "relevant to answering the query. Discard irrelevant information.\n\n"
        "Query: {}\n\n"
        "Web search results:\n{}\n\n"
        "Organized key facts:",
        query, web_search_results);
}

} // namespace prompts::crag
```

---

## 8. Tool Description Templates

```cpp
// src/prompts/tool_prompts.hpp
#pragma once
#include <string>
#include "tools/tool_spec.hpp"

namespace prompts::tools {

// Generate a human-readable description for a tool (for system prompt inclusion)
inline std::string describe_tool(const ::tools::ToolSpec& spec) {
    std::string params;
    for (const auto& p : spec.parameters) {
        params += std::format("  - {} ({}{}): {}\n",
            p.name, p.type,
            p.required ? ", required" : ", optional",
            p.description);
    }

    return std::format(
        "### {}\n"
        "{}\n"
        "Parameters:\n"
        "{}"
        "Category: {}\n"
        "Permission: {}",
        spec.name,
        spec.description,
        params,
        spec.category,
        static_cast<int>(spec.permission));
}

// Generate a compact tool list for few-shot examples
inline std::string tool_list_compact(const std::vector<::tools::ToolSpec>& specs) {
    std::string result;
    for (const auto& spec : specs)
        result += std::format("- **{}**: {}\n", spec.name, spec.description);
    return result;
}

// Generate tool usage example
inline std::string tool_usage_example(
    const std::string& tool_name,
    const nlohmann::json& example_args,
    const std::string& example_output)
{
    return std::format(
        "Example:\n"
        "Action: {}\n"
        "Action Input: {}\n"
        "Observation: {}",
        tool_name,
        example_args.dump(2),
        example_output.substr(0, 200));
}

} // namespace prompts::tools
```

---

## 9. Compaction Continuation Prompt

When resuming a compacted session, the agent needs a special prompt to
reorient itself based on the summary:

```cpp
// src/prompts/compaction_prompts.hpp
#pragma once
#include <string>

namespace prompts::compaction {

// Injected as the first message after a compaction summary
inline std::string continuation_prompt(
    const std::string& summary,
    const std::string& current_task)
{
    return std::format(
        "The previous conversation has been summarized to save space.\n\n"
        "Summary of prior work:\n{}\n\n"
        "Current task: {}\n\n"
        "Continue from where we left off, using the summary above as context. "
        "Do not repeat work already completed.",
        summary, current_task);
}

// System prompt addendum for compacted sessions
inline std::string compacted_session_notice(int compaction_count) {
    return std::format(
        "Note: This conversation has been compacted {} time(s). "
        "The current context contains a summary of earlier work, "
        "followed by the most recent messages.",
        compaction_count);
}

} // namespace prompts::compaction
```

---

## 10. Full System Prompt Example

```cpp
// Usage example: assembling a complete system prompt for a code assistant agent
#include "prompts/system_prompt.hpp"
#include "prompts/react_templates.hpp"

std::string build_code_assistant_prompt(
    const std::string& workspace_root,
    const std::vector<std::string>& tool_names,
    const std::string& project_context,
    const std::string& memory_state)
{
    prompts::SystemPromptBuilder builder;

    builder
        .with_intro("CodeAssist",
            "an expert software engineering assistant specializing in C++23")
        .with_project_context(project_context)
        .with_tools_overview(tool_names)
        .with_react_instructions()
        .with_environment_info()
        .with_date()
        .with_workspace_context(workspace_root);

    if (!memory_state.empty())
        builder.with_memory_state(memory_state);

    // Add custom sections
    builder.add_section({
        "coding_principles", prompts::SectionKind::Static,
        "## Coding Principles\n\n"
        "- Use C++23 features: std::expected, std::flat_map, std::print, import std;\n"
        "- Prefer value semantics and const-correctness\n"
        "- Use std::expected<T,E> instead of exceptions for error propagation\n"
        "- Write self-documenting code with clear variable names\n"
        "- Always check return values and handle errors\n"
        "- Test changes after making them",
        25
    });

    return builder.build("tools_overview");
}
```

---

## 11. Prompt Caching Strategy

For Anthropic API with prompt caching, the dynamic boundary determines
which portion of the prompt is sent fresh each turn:

```
[STATIC — cached after first call]
├── intro
├── coding_principles
├── project_context
├── tools_overview
├── react_format
├── environment
└── ---- CACHE BOUNDARY ----

[DYNAMIC — regenerated each turn]
├── date
├── workspace_context
└── memory_state (changes as agent runs)
```

For OpenAI API (no explicit cache control), structure prompts so the longest
stable prefix maximizes KV cache reuse on repeated calls within a session.

```cpp
// Emit cache_control markers for Anthropic API
inline nlohmann::json make_anthropic_system_with_cache(
    const std::string& static_prefix,
    const std::string& dynamic_suffix)
{
    return nlohmann::json::array({
        {
            {"type",  "text"},
            {"text",  static_prefix},
            {"cache_control", {{"type", "ephemeral"}}}
        },
        {
            {"type", "text"},
            {"text", dynamic_suffix}
        }
    });
}
```
