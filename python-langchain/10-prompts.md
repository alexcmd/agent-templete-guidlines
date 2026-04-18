# 10 — Prompt Engineering

## 1. ChatPromptTemplate and Hub Pulling

```python
# src/my_agent/prompts/templates.py
from __future__ import annotations

from langchain import hub
from langchain_core.prompts import (
    ChatPromptTemplate,
    HumanMessagePromptTemplate,
    MessagesPlaceholder,
    SystemMessagePromptTemplate,
    PromptTemplate,
)
from langchain_core.messages import SystemMessage


# --- Basic template ---
basic_prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant. Answer concisely."),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])

# --- Hub pulling ---
react_prompt = hub.pull("hwchase17/react")          # classic ReAct
openai_functions_prompt = hub.pull("hwchase17/openai-functions-agent")

# --- Partial variables ---
from datetime import datetime
dated_prompt = ChatPromptTemplate.from_messages([
    ("system", "Today is {date}. You are a helpful assistant."),
    ("human", "{input}"),
]).partial(date=datetime.now().strftime("%Y-%m-%d"))

# --- From template string ---
simple_template = PromptTemplate.from_template(
    "Summarize the following in {num_words} words:\n\n{text}"
)
```

---

## 2. SystemPromptBuilder Class

```python
# src/my_agent/prompts/system.py
from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime


@dataclass
class SystemPromptBuilder:
    """Builds a structured system prompt with sections."""

    persona: str = "You are a helpful AI assistant."
    capabilities: list[str] = field(default_factory=list)
    constraints: list[str] = field(default_factory=list)
    output_format: str = ""
    examples: list[dict[str, str]] = field(default_factory=list)
    tools_description: str = ""
    memory_instructions: str = ""
    dynamic_context: str = ""

    def build(self, include_date: bool = True) -> str:
        sections: list[str] = []

        # Header
        if include_date:
            sections.append(f"[{datetime.now().strftime('%Y-%m-%d')}]")
        sections.append(self.persona)

        if self.capabilities:
            caps = "\n".join(f"- {c}" for c in self.capabilities)
            sections.append(f"## Capabilities\n{caps}")

        if self.constraints:
            cons = "\n".join(f"- {c}" for c in self.constraints)
            sections.append(f"## Constraints\n{cons}")

        if self.tools_description:
            sections.append(f"## Tools\n{self.tools_description}")

        if self.memory_instructions:
            sections.append(f"## Memory\n{self.memory_instructions}")

        if self.output_format:
            sections.append(f"## Output Format\n{self.output_format}")

        if self.examples:
            ex_text = ""
            for i, ex in enumerate(self.examples, 1):
                ex_text += f"Example {i}:\nInput: {ex['input']}\nOutput: {ex['output']}\n\n"
            sections.append(f"## Examples\n{ex_text.rstrip()}")

        if self.dynamic_context:
            sections.append(f"## Current Context\n{self.dynamic_context}")

        return "\n\n".join(sections)


def build_system_prompt() -> str:
    builder = SystemPromptBuilder(
        persona=(
            "You are an expert software engineering assistant. "
            "You help users design, implement, review, and debug software."
        ),
        capabilities=[
            "Write and review Python, TypeScript, and shell code",
            "Search the web for up-to-date information",
            "Read and write files in the workspace",
            "Execute bash commands and tests",
            "Manage knowledge across multiple documents",
        ],
        constraints=[
            "Never execute destructive commands without explicit confirmation",
            "Always explain your reasoning before taking actions",
            "Cite sources when referencing external information",
            "Return errors clearly with actionable remediation steps",
        ],
        output_format=(
            "Structure responses with:\n"
            "1. Brief summary of what you understood\n"
            "2. Your approach / reasoning\n"
            "3. Implementation or answer\n"
            "4. Next steps or caveats"
        ),
        memory_instructions=(
            "Before responding, check if you have relevant memories from previous sessions. "
            "After completing significant tasks, save key findings to memory."
        ),
    )
    return builder.build()
```

---

## 3. ReAct Thought/Action/Observation Templates (arXiv:2210.03629)

```python
REACT_SYSTEM_PROMPT = """\
You are an AI that reasons step-by-step using the ReAct framework.

For each step, you MUST follow this exact format:

Thought: <your reasoning about what to do next>
Action: <tool_name>
Action Input: <the input to the tool as JSON>

After receiving an Observation, continue with:
Thought: <your updated reasoning>
Action: <next action, OR "Final Answer">
Action Input: <input OR the final answer>

Rules:
- Always start with a Thought
- Only call one tool per step
- Verify tool results before using them
- When you have enough information, use Action: Final Answer

Available tools: {tool_names}

Tool descriptions:
{tools}
"""

REACT_HUMAN_TEMPLATE = """\
Question: {input}
{agent_scratchpad}"""

react_prompt = ChatPromptTemplate.from_messages([
    SystemMessagePromptTemplate.from_template(REACT_SYSTEM_PROMPT),
    HumanMessagePromptTemplate.from_template(REACT_HUMAN_TEMPLATE),
])
```

---

## 4. Reflexion Self-Reflection Prompt (arXiv:2303.11366)

```python
REFLEXION_ACTOR_PROMPT = """\
You are an expert problem solver.

## Past Reflections (lessons learned):
{reflections}

## Current Task:
{task}

Attempt the task, incorporating lessons from past reflections.
Be thorough, accurate, and complete."""

REFLEXION_EVALUATOR_PROMPT = """\
Evaluate the quality of this response on a scale 0.0 to 1.0.

Scoring rubric:
- 0.0–0.3: Incorrect or fundamentally flawed
- 0.4–0.6: Partially correct but missing key elements
- 0.7–0.8: Mostly correct with minor issues
- 0.9–1.0: Excellent, complete, and accurate

Task: {task}
Response: {response}

Return a JSON object: {{"score": 0.0, "reasoning": "brief explanation"}}"""

REFLEXION_REFLECTION_PROMPT = """\
Your previous attempt at this task scored {score}/1.0.

Task: {task}
Your attempt: {response}

Reflect on:
1. What specific mistakes did you make?
2. What knowledge or approach was missing?
3. What will you do differently next time?

Be concise (2-4 sentences). Focus on actionable lessons."""
```

---

## 5. HyDE Hypothetical Document Prompt (arXiv:2212.10496)

```python
HYDE_PROMPTS = {
    "general": """\
Write a detailed, informative document that would perfectly answer this question.
Write as if you are the most relevant document in a knowledge base.

Question: {question}

Document:""",

    "code": """\
Write a Python code example with documentation that solves this programming question.
Include imports, type hints, and docstrings.

Question: {question}

```python""",

    "academic": """\
Write an academic abstract that addresses this research question.
Include methodology, findings, and conclusions.

Question: {question}

Abstract:""",
}

def build_hyde_prompt(domain: str = "general") -> ChatPromptTemplate:
    template = HYDE_PROMPTS.get(domain, HYDE_PROMPTS["general"])
    return ChatPromptTemplate.from_messages([
        ("system", "You generate hypothetical documents for retrieval augmentation."),
        ("human", template),
    ])
```

---

## 6. Triple Extraction Prompt

```python
TRIPLE_EXTRACTION_PROMPT = """\
Extract a knowledge graph from the text below.

## Output format:
Return a JSON object with:
- entities: list of {{id, name, entity_type, description}}
- relations: list of {{source_id, target_id, relation_type, weight}}

## Entity types: Person, Organization, Technology, Concept, Event, Location, Product

## Relation types (examples):
- Person → Organization: "founded_by", "works_at", "leads"
- Technology → Technology: "uses", "extends", "replaces"
- Event → Organization: "organized_by", "impacts"
- Concept → Concept: "related_to", "is_a", "part_of"

## Rules:
- Only extract facts explicitly stated in the text
- Entity IDs: snake_case, unique (e.g., "elon_musk", "tesla_inc")
- Weight = confidence (0.5=uncertain, 1.0=explicitly stated)
- Minimum 3 entities, minimum 2 relations

## Text:
{text}

## Knowledge Graph JSON:"""

triple_extraction_prompt = ChatPromptTemplate.from_messages([
    ("system", "You are an expert knowledge graph builder."),
    ("human", TRIPLE_EXTRACTION_PROMPT),
])
```

---

## 7. SELF-REFINE Critique Prompt (selfrefine.info)

```python
SELF_REFINE_CRITIQUE_PROMPT = """\
Critically evaluate the following draft response.

## Task:
{task}

## Draft:
{draft}

## Evaluation criteria:
1. **Accuracy**: Is everything factually correct?
2. **Completeness**: Does it fully address all aspects of the task?
3. **Clarity**: Is it easy to understand?
4. **Conciseness**: Is there unnecessary content?
5. **Format**: Is the formatting appropriate?

## Instructions:
- If the draft is excellent and fully satisfies all criteria: respond with exactly "SATISFACTORY"
- Otherwise: provide specific, actionable critique points as a numbered list
- Be precise about what needs improvement, not vague
- Focus on the most important issues (max 3)"""

SELF_REFINE_REVISION_PROMPT = """\
Revise the following draft based on the critique.

## Task:
{task}

## Original Draft:
{draft}

## Critique:
{critique}

## Instructions:
- Address every critique point explicitly
- Improve the draft, don't just add to it
- Maintain what was good in the original
- Produce the revised version only (no commentary)

## Revised Draft:"""
```

---

## 8. Compaction Continuation Prompt

```python
COMPACTION_PROMPT = """\
The conversation history has been summarized to save context space.

## Summary of previous conversation:
{summary}

## Recent messages (verbatim):
{recent_messages}

You are continuing this conversation. The summary contains all important context
from earlier in the conversation. The recent messages are the most current context.
Proceed naturally as if the full conversation is available."""

def build_compaction_continuation_prompt(
    summary: str,
    recent_messages: list,
) -> str:
    recent_text = "\n".join(
        f"{m.type.upper()}: {getattr(m, 'content', '')}"
        for m in recent_messages
    )
    return COMPACTION_PROMPT.format(
        summary=summary,
        recent_messages=recent_text,
    )
```

---

## 9. Dynamic Boundary Pattern

Inject runtime context (user info, available tools, permissions) into the system prompt:

```python
# src/my_agent/prompts/dynamic.py
from __future__ import annotations

from langchain_core.messages import SystemMessage


def build_dynamic_system_message(
    user_id: str,
    user_role: str,
    approved_tools: list[str],
    user_memories: list[str],
    current_date: str,
) -> SystemMessage:
    """Build a system message with dynamic user context."""

    memories_block = ""
    if user_memories:
        memories_block = (
            "\n## What I remember about you:\n"
            + "\n".join(f"- {m}" for m in user_memories[:5])
        )

    tools_block = (
        f"\n## Approved tools: {', '.join(approved_tools)}"
        if approved_tools
        else ""
    )

    role_instructions = {
        "admin": "You have full access to all system capabilities.",
        "developer": "You can read/write code files and execute tests.",
        "analyst": "You can read data and generate reports, but not modify production systems.",
        "viewer": "You have read-only access. Suggest but do not execute changes.",
    }.get(user_role, "You have standard access.")

    content = f"""\
[{current_date}]
You are an expert AI assistant.

## User Context:
- User ID: {user_id}
- Role: {user_role}
- Permissions: {role_instructions}
{tools_block}
{memories_block}

## Core Instructions:
- Always be helpful, accurate, and concise
- Respect the user's role and permission boundaries
- Cite your sources when making factual claims
- Ask for clarification before taking irreversible actions"""

    return SystemMessage(content=content)
```

---

## 10. LangSmith Hub Prompt Versioning

```python
# src/my_agent/prompts/hub_management.py
from __future__ import annotations

from langchain import hub
from langsmith import Client


def push_prompt_to_hub(prompt_name: str, prompt_template: Any) -> str:
    """Push a prompt to LangSmith Hub and return the URL."""
    url = hub.push(
        f"my-org/{prompt_name}",
        prompt_template,
        new_repo_is_public=False,
        tags=["production", "v1"],
    )
    print(f"Pushed to: {url}")
    return url


def pull_versioned_prompt(prompt_name: str, commit_hash: str | None = None) -> Any:
    """Pull a specific version of a prompt from LangSmith Hub."""
    if commit_hash:
        return hub.pull(f"my-org/{prompt_name}:{commit_hash}")
    return hub.pull(f"my-org/{prompt_name}")


def list_prompt_versions(prompt_name: str) -> list[dict]:
    """List all versions of a Hub prompt."""
    client = Client()
    versions = list(client.list_commits(prompt_identifier=f"my-org/{prompt_name}"))
    return [{"hash": v.commit_hash, "created_at": v.created_at} for v in versions]


# Canonical prompt registry — pull once at startup, cache
_PROMPT_CACHE: dict[str, Any] = {}


def get_prompt(name: str, fallback: Any | None = None) -> Any:
    """Get prompt from cache or Hub, with optional local fallback."""
    if name not in _PROMPT_CACHE:
        try:
            _PROMPT_CACHE[name] = hub.pull(f"my-org/{name}")
        except Exception:
            if fallback is not None:
                _PROMPT_CACHE[name] = fallback
            else:
                raise
    return _PROMPT_CACHE[name]
```

---

## 11. PromptTemplate with Partial Variables

```python
from langchain_core.prompts import PromptTemplate
from datetime import datetime


# Partial at template definition time
analysis_prompt = PromptTemplate(
    template="As of {date}, analyze {topic} with focus on {aspect}.",
    input_variables=["topic", "aspect"],
    partial_variables={"date": datetime.now().strftime("%B %Y")},
)
# Usage: analysis_prompt.format(topic="AI", aspect="safety")


# Partial at runtime
base_prompt = PromptTemplate(
    template="You are a {persona}. {instruction}",
    input_variables=["persona", "instruction"],
)
coding_prompt = base_prompt.partial(persona="senior Python engineer")
# Usage: coding_prompt.format(instruction="Review this code: ...")


# ChatPromptTemplate partial
from langchain_core.prompts import ChatPromptTemplate

chat_base = ChatPromptTemplate.from_messages([
    ("system", "You are {persona}. Language: {language}."),
    ("human", "{question}"),
])
french_assistant = chat_base.partial(language="French")
# Usage: french_assistant.invoke({"persona": "tutor", "question": "What is Python?"})
```
