# 08 — Self-Improvement Mechanisms

## 1. Reflexion with Episodic Memory Store (arXiv:2303.11366)

Full implementation storing reflections in LangGraph Store for cross-session recall.

```python
# src/my_agent/self_improvement/reflexion.py
from __future__ import annotations

import operator
from typing import Annotated, Any, Literal, TypedDict

from langchain_core.messages import AIMessage, BaseMessage, HumanMessage, SystemMessage
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from langgraph.store.base import BaseStore


class ReflexionState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    task: str
    draft: str
    reflections: Annotated[list[str], operator.add]
    score: float
    trial: int
    final_answer: str


ACTOR_SYSTEM = """\
You are an expert problem solver.
Past reflections from similar tasks:
{reflections}

Apply these lessons to produce an excellent response."""

EVALUATOR_SYSTEM = """\
Evaluate the quality of this response for the given task.
Scoring criteria:
- Accuracy (0-0.4): Is the response factually correct?
- Completeness (0-0.3): Does it address all parts of the task?
- Clarity (0-0.3): Is it clear and well-structured?
Return a JSON object: {{"score": 0.0, "reasoning": "..."}}"""

REFLECTOR_SYSTEM = """\
You failed to meet quality standards (score={score}).
Identify what went wrong and how to do better next time.
Be specific, actionable, and concise (2-3 sentences)."""


def build_reflexion_with_store(
    store: BaseStore,
    score_threshold: float = 0.85,
    max_trials: int = 4,
) -> Any:
    actor_llm = ChatOpenAI(model="gpt-4o", temperature=0.4)
    eval_llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
    reflect_llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.3)

    async def actor_node(state: ReflexionState) -> dict[str, Any]:
        # Load past reflections from store
        ns = ("reflexion", "global_reflections")
        past = await store.asearch(ns, query=state["task"], limit=5)
        past_texts = [item.value["reflection"] for item in past]

        # Merge with in-session reflections
        all_reflections = past_texts + state.get("reflections", [])
        reflections_str = "\n".join(f"- {r}" for r in all_reflections[-5:]) or "None"

        system = ACTOR_SYSTEM.format(reflections=reflections_str)
        response = await actor_llm.ainvoke([
            SystemMessage(content=system),
            HumanMessage(content=state["task"]),
        ])
        return {"draft": response.content, "messages": [response]}

    async def evaluator_node(state: ReflexionState) -> dict[str, Any]:
        import json
        prompt = (
            f"Task: {state['task']}\n\n"
            f"Response: {state['draft']}"
        )
        result = await eval_llm.ainvoke([
            SystemMessage(content=EVALUATOR_SYSTEM),
            HumanMessage(content=prompt),
        ])
        try:
            data = json.loads(result.content)
            score = float(data.get("score", 0.0))
        except (json.JSONDecodeError, ValueError):
            score = 0.5
        return {"score": score}

    async def reflector_node(state: ReflexionState) -> dict[str, Any]:
        system = REFLECTOR_SYSTEM.format(score=state["score"])
        result = await reflect_llm.ainvoke([
            SystemMessage(content=system),
            HumanMessage(content=f"Task: {state['task']}\n\nAttempt: {state['draft']}"),
        ])
        reflection = result.content

        # Persist to cross-session store
        ns = ("reflexion", "global_reflections")
        import uuid
        await store.aput(ns, str(uuid.uuid4()), {
            "reflection": reflection,
            "task_summary": state["task"][:100],
            "score": state["score"],
            "trial": state["trial"],
        })

        return {
            "reflections": [reflection],
            "trial": state.get("trial", 0) + 1,
        }

    def route_eval(state: ReflexionState) -> Literal["reflect", "end"]:
        if state["score"] >= score_threshold or state.get("trial", 0) >= max_trials:
            return "end"
        return "reflect"

    builder = StateGraph(ReflexionState)
    builder.add_node("actor", actor_node)
    builder.add_node("evaluator", evaluator_node)
    builder.add_node("reflector", reflector_node)

    builder.add_edge(START, "actor")
    builder.add_edge("actor", "evaluator")
    builder.add_conditional_edges("evaluator", route_eval, {"reflect": "reflector", "end": END})
    builder.add_edge("reflector", "actor")

    return builder.compile(store=store)
```

---

## 2. ExpeL — Experience Pool (arXiv:2308.10144)

Store agent trajectories in a vector store; extract LLM insights for future use.

```python
# src/my_agent/self_improvement/expel.py
from __future__ import annotations

from typing import Any

from langchain_core.documents import Document
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import FAISS


INSIGHT_EXTRACTION_PROMPT = ChatPromptTemplate.from_messages([
    ("system", (
        "You are analyzing agent trajectories to extract generalizable insights.\n"
        "Given a completed task trajectory (steps taken, tools used, outcome), "
        "extract 3-5 concise, actionable rules that could help future attempts.\n"
        "Format: one rule per line, starting with 'RULE:'"
    )),
    ("human", "Trajectory:\n{trajectory}\n\nOutcome: {outcome}\nScore: {score}"),
])


class ExperiencePool:
    """Vector store of past agent trajectories with extracted insights."""

    def __init__(self, embeddings_model: str = "text-embedding-3-small") -> None:
        self.embeddings = OpenAIEmbeddings(model=embeddings_model)
        self._trajectory_store: FAISS | None = None
        self._insight_store: FAISS | None = None
        self._llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

    def add_trajectory(self, trajectory: str, task: str, outcome: str, score: float) -> None:
        """Add a completed trajectory to the experience pool."""
        doc = Document(
            page_content=trajectory,
            metadata={"task": task, "outcome": outcome, "score": score},
        )
        if self._trajectory_store is None:
            self._trajectory_store = FAISS.from_documents([doc], self.embeddings)
        else:
            self._trajectory_store.add_documents([doc])

        # Extract and store insights if score is high enough
        if score >= 0.7:
            self._extract_and_store_insights(trajectory, outcome, score)

    def _extract_and_store_insights(self, trajectory: str, outcome: str, score: float) -> None:
        chain = INSIGHT_EXTRACTION_PROMPT | self._llm
        result = chain.invoke({"trajectory": trajectory[:2000], "outcome": outcome, "score": score})
        rules = [
            line.replace("RULE:", "").strip()
            for line in result.content.splitlines()
            if line.strip().startswith("RULE:")
        ]
        if not rules:
            return
        docs = [Document(page_content=rule, metadata={"source": "expel"}) for rule in rules]
        if self._insight_store is None:
            self._insight_store = FAISS.from_documents(docs, self.embeddings)
        else:
            self._insight_store.add_documents(docs)

    def get_relevant_insights(self, task: str, k: int = 5) -> list[str]:
        """Retrieve relevant insights for a new task."""
        if self._insight_store is None:
            return []
        docs = self._insight_store.similarity_search(task, k=k)
        return [d.page_content for d in docs]

    def get_similar_trajectories(self, task: str, k: int = 3) -> list[Document]:
        if self._trajectory_store is None:
            return []
        return self._trajectory_store.similarity_search(task, k=k)

    def save(self, path: str) -> None:
        if self._trajectory_store:
            self._trajectory_store.save_local(f"{path}/trajectories")
        if self._insight_store:
            self._insight_store.save_local(f"{path}/insights")
```

---

## 3. CRITIC: Tool-Verified Correction (arXiv:2310.06825)

```python
# src/my_agent/self_improvement/critic.py
from __future__ import annotations

from typing import Any, Literal, TypedDict

from langchain_core.messages import BaseMessage
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.tools import BaseTool
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode
from typing import Annotated


class CriticState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    task: str
    response: str
    critique: str
    is_verified: bool
    iteration: int


CRITIC_SYSTEM = """\
You are a CRITIC agent. Verify the response to the task using tools.
Steps:
1. Identify claims in the response that can be verified
2. Use tools to check each claim
3. If all claims are verified: output VERIFIED
4. If issues found: output CRITIQUE: <specific issues>"""

REVISE_SYSTEM = """\
You received this critique on your response:
{critique}

Revise your response to address all issues. Be precise and accurate."""


def build_critic_graph(verification_tools: list[BaseTool], max_iterations: int = 3) -> Any:
    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    generator_llm = ChatOpenAI(model="gpt-4o", temperature=0.3)
    critic_llm = llm.bind_tools(verification_tools)

    def generator_node(state: CriticState) -> dict[str, Any]:
        if state.get("critique"):
            # Revision pass
            from langchain_core.messages import SystemMessage, HumanMessage
            revision = generator_llm.invoke([
                SystemMessage(content=REVISE_SYSTEM.format(critique=state["critique"])),
                HumanMessage(content=f"Original task: {state['task']}\nPrevious response: {state['response']}"),
            ])
            return {"response": revision.content, "messages": [revision]}
        else:
            result = generator_llm.invoke(f"Task: {state['task']}")
            return {"response": result.content, "messages": [result]}

    def critic_node(state: CriticState) -> dict[str, Any]:
        from langchain_core.messages import SystemMessage, HumanMessage
        response = critic_llm.invoke([
            SystemMessage(content=CRITIC_SYSTEM),
            HumanMessage(content=f"Task: {state['task']}\n\nResponse to verify: {state['response']}"),
        ])
        is_verified = "VERIFIED" in response.content.upper()
        critique = ""
        if not is_verified and "CRITIQUE:" in response.content:
            critique = response.content.split("CRITIQUE:", 1)[-1].strip()
        return {
            "messages": [response],
            "is_verified": is_verified,
            "critique": critique,
            "iteration": state.get("iteration", 0) + 1,
        }

    def route(state: CriticState) -> Literal["tools", "generate", "end"]:
        last = state["messages"][-1]
        from langchain_core.messages import AIMessage
        if isinstance(last, AIMessage) and last.tool_calls:
            return "tools"
        if state["is_verified"] or state.get("iteration", 0) >= max_iterations:
            return "end"
        return "generate"

    tool_node = ToolNode(verification_tools)
    builder = StateGraph(CriticState)
    builder.add_node("generate", generator_node)
    builder.add_node("critic", critic_node)
    builder.add_node("tools", tool_node)

    builder.add_edge(START, "generate")
    builder.add_edge("generate", "critic")
    builder.add_conditional_edges("critic", route, {"tools": "tools", "generate": "generate", "end": END})
    builder.add_edge("tools", "critic")

    return builder.compile()
```

---

## 4. SELF-REFINE as Iterative LangGraph Graph (selfrefine.info)

(See also 02-core-agent-loop.md for the basic version — this adds persistence)

```python
# src/my_agent/self_improvement/self_refine_persistent.py
from __future__ import annotations

from typing import Any, Literal, TypedDict

from langchain_core.messages import BaseMessage
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import InMemorySaver
from typing import Annotated


class PersistentRefineState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    task: str
    draft: str
    critique: str
    iteration: int
    history: list[dict]     # full draft history for analysis
    is_satisfactory: bool


async def generate(state: PersistentRefineState) -> dict[str, Any]:
    llm = ChatOpenAI(model="gpt-4o", temperature=0.4)
    if state.get("critique"):
        prompt = (
            f"Task: {state['task']}\n"
            f"Current draft: {state['draft']}\n"
            f"Critique: {state['critique']}\n\n"
            "Produce an improved version that addresses all critique points."
        )
    else:
        prompt = f"Task: {state['task']}\n\nProduce a high-quality response."
    result = await llm.ainvoke(prompt)
    history = state.get("history", []) + [{"draft": result.content, "iteration": state.get("iteration", 0)}]
    return {"draft": result.content, "messages": [result], "history": history}


async def critique(state: PersistentRefineState) -> dict[str, Any]:
    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    prompt = (
        f"Task: {state['task']}\n\n"
        f"Draft (iteration {state.get('iteration', 0)}):\n{state['draft']}\n\n"
        "Provide a specific critique. If the draft fully satisfies the task, respond: SATISFACTORY"
    )
    result = await llm.ainvoke(prompt)
    is_satisfactory = "SATISFACTORY" in result.content.upper()
    return {
        "critique": result.content,
        "is_satisfactory": is_satisfactory,
        "iteration": state.get("iteration", 0) + 1,
        "messages": [result],
    }


def route(state: PersistentRefineState) -> Literal["generate", "end"]:
    if state.get("is_satisfactory") or state.get("iteration", 0) >= 5:
        return "end"
    return "generate"


builder = StateGraph(PersistentRefineState)
builder.add_node("generate", generate)
builder.add_node("critique", critique)
builder.add_edge(START, "generate")
builder.add_edge("generate", "critique")
builder.add_conditional_edges("critique", route, {"generate": "generate", "end": END})

self_refine_graph = builder.compile(checkpointer=InMemorySaver())
```

---

## 5. Skill Library — Voyager Pattern (embed code in Qdrant)

```python
# src/my_agent/self_improvement/skill_library.py
from __future__ import annotations

from typing import Any

from langchain_core.documents import Document
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_qdrant import QdrantVectorStore
from pydantic import BaseModel, Field


class Skill(BaseModel):
    name: str = Field(description="Short skill name")
    description: str = Field(description="What this skill does")
    code: str = Field(description="Python implementation")
    dependencies: list[str] = Field(default_factory=list)
    success_count: int = Field(default=0)
    failure_count: int = Field(default=0)

    @property
    def success_rate(self) -> float:
        total = self.success_count + self.failure_count
        return self.success_count / total if total > 0 else 0.0


class SkillLibrary:
    """Qdrant-backed library of reusable agent skills."""

    def __init__(self, collection: str = "skills") -> None:
        self._vs: QdrantVectorStore | None = None
        self._collection = collection
        self._embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
        self._skills: dict[str, Skill] = {}

    def _get_vs(self) -> QdrantVectorStore:
        if self._vs is None:
            self._vs = QdrantVectorStore.from_texts(
                ["__init__"],
                self._embeddings,
                url="http://localhost:6333",
                collection_name=self._collection,
            )
        return self._vs

    def add_skill(self, skill: Skill) -> None:
        self._skills[skill.name] = skill
        doc = Document(
            page_content=f"{skill.name}: {skill.description}\n\nCode:\n{skill.code}",
            metadata={"skill_name": skill.name, "success_rate": skill.success_rate},
        )
        self._get_vs().add_documents([doc])

    def search_skills(self, task: str, k: int = 3) -> list[Skill]:
        docs = self._get_vs().similarity_search(task, k=k)
        result = []
        for doc in docs:
            name = doc.metadata.get("skill_name")
            if name and name in self._skills:
                result.append(self._skills[name])
        return result

    def record_outcome(self, skill_name: str, success: bool) -> None:
        if skill_name in self._skills:
            if success:
                self._skills[skill_name].success_count += 1
            else:
                self._skills[skill_name].failure_count += 1

    async def generate_skill(self, task: str, example_solution: str) -> Skill:
        """Use LLM to generalize a solution into a reusable skill."""
        llm = ChatOpenAI(model="gpt-4o")
        structured = llm.with_structured_output(Skill)
        prompt = (
            f"Generalize the following solution into a reusable skill:\n\n"
            f"Task: {task}\nSolution: {example_solution}\n\n"
            "Extract the reusable pattern as a Skill."
        )
        skill: Skill = await structured.ainvoke(prompt)
        self.add_skill(skill)
        return skill
```

---

## 6. LangSmith Evaluation → Fine-Tuning Data Export

```python
# src/my_agent/self_improvement/finetune_export.py
from __future__ import annotations

import json
from pathlib import Path
from langsmith import Client


def export_high_quality_examples(
    dataset_name: str,
    min_score: float = 0.8,
    output_path: str = "finetune_data.jsonl",
) -> int:
    """
    Export high-scoring LangSmith examples as OpenAI fine-tuning JSONL.
    Returns number of examples exported.
    """
    client = Client()
    examples = list(client.list_examples(dataset_name=dataset_name))

    # Get feedback scores for each example
    exported = 0
    with open(output_path, "w") as f:
        for example in examples:
            # Get feedback for this run
            feedbacks = list(client.list_feedback(run_ids=[str(example.id)]))
            scores = [fb.score for fb in feedbacks if fb.score is not None]
            avg_score = sum(scores) / len(scores) if scores else 0.0

            if avg_score >= min_score:
                # Format for OpenAI fine-tuning
                messages = []
                if example.inputs.get("system"):
                    messages.append({"role": "system", "content": example.inputs["system"]})
                messages.append({"role": "user", "content": str(example.inputs.get("input", ""))})
                messages.append({"role": "assistant", "content": str(example.outputs.get("output", ""))})

                f.write(json.dumps({"messages": messages}) + "\n")
                exported += 1

    print(f"Exported {exported}/{len(examples)} examples (min_score={min_score})")
    return exported
```

---

## 7. Quality Scoring with LLM-as-Judge (LangSmith)

```python
# src/my_agent/self_improvement/llm_judge.py
from __future__ import annotations

from typing import Any

from langsmith.evaluation import evaluate
from langsmith.schemas import Example, Run


def build_quality_evaluator(criteria: list[str]) -> Any:
    """Build a LangSmith LLM-as-judge evaluator for multiple criteria."""
    from langsmith.evaluation import LangChainStringEvaluator

    evaluators = []
    for criterion in criteria:
        evaluators.append(
            LangChainStringEvaluator(
                "criteria",
                config={"criteria": criterion},
            )
        )
    return evaluators


# Custom evaluator function
def code_correctness_evaluator(run: Run, example: Example) -> dict:
    """Evaluate if generated code is syntactically correct Python."""
    import ast
    output = run.outputs.get("output", "") if run.outputs else ""

    # Extract code blocks
    code_blocks = []
    in_block = False
    current_block: list[str] = []
    for line in output.splitlines():
        if line.strip().startswith("```python"):
            in_block = True
            current_block = []
        elif line.strip() == "```" and in_block:
            in_block = False
            code_blocks.append("\n".join(current_block))
        elif in_block:
            current_block.append(line)

    if not code_blocks:
        return {"key": "code_correctness", "score": 0.5, "comment": "No code blocks found"}

    errors = []
    for block in code_blocks:
        try:
            ast.parse(block)
        except SyntaxError as e:
            errors.append(str(e))

    score = 1.0 if not errors else 0.0
    return {
        "key": "code_correctness",
        "score": score,
        "comment": f"Errors: {errors}" if errors else "All code blocks are valid Python",
    }


# Run evaluation
def run_quality_evaluation(
    dataset_name: str,
    agent_function: Any,
    experiment_prefix: str = "quality-eval",
) -> Any:
    return evaluate(
        agent_function,
        data=dataset_name,
        evaluators=[
            code_correctness_evaluator,
            *build_quality_evaluator(["correctness", "helpfulness", "conciseness"]),
        ],
        experiment_prefix=experiment_prefix,
        metadata={"model": "gpt-4o"},
    )
```
