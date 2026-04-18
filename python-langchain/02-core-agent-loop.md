# 02 — Core Agent Loop

## 1. State Definition with TypedDict and Reducers

```python
# src/my_agent/state.py
from __future__ import annotations

import operator
from typing import Annotated, Any, TypedDict

from langchain_core.messages import BaseMessage
from langgraph.graph.message import add_messages


class AgentState(TypedDict):
    # add_messages reducer: new messages appended, not overwritten
    messages: Annotated[list[BaseMessage], add_messages]

    # operator.add reducer: list items accumulated across nodes
    observations: Annotated[list[str], operator.add]

    # Last value wins (default reducer — no annotation needed)
    current_task: str
    iteration: int
    should_continue: bool
    error: str | None

    # Arbitrary scratchpad; last write wins
    scratchpad: dict[str, Any]
```

**Reducer rules:**
- `add_messages` — merges message lists, deduplicates by `id`, handles updates
- `operator.add` — concatenates lists; useful for accumulating observations
- No annotation — last write wins (simple assignment)

---

## 2. ReAct Agent as LangGraph Graph (full implementation)

ReAct: Reason + Act interleaved (arXiv:2210.03629)

```python
# src/my_agent/graph.py
from __future__ import annotations

from typing import Any, Literal

from langchain_core.messages import AIMessage, SystemMessage
from langchain_core.tools import BaseTool
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, StateGraph
from langgraph.prebuilt import ToolNode

from my_agent.state import AgentState
from my_agent.prompts.system import build_system_prompt


SYSTEM_PROMPT = build_system_prompt()


def build_react_graph(
    tools: list[BaseTool],
    model_name: str = "gpt-4o",
    checkpointer: Any = None,
) -> Any:
    llm = ChatOpenAI(model=model_name, temperature=0, streaming=True)
    llm_with_tools = llm.bind_tools(tools)

    # ------------------------------------------------------------------ #
    #  Nodes                                                               #
    # ------------------------------------------------------------------ #
    def agent_node(state: AgentState) -> dict[str, Any]:
        messages = [SystemMessage(content=SYSTEM_PROMPT)] + state["messages"]
        response: AIMessage = llm_with_tools.invoke(messages)
        return {"messages": [response]}

    def should_continue(state: AgentState) -> Literal["tools", "end"]:
        last = state["messages"][-1]
        if isinstance(last, AIMessage) and last.tool_calls:
            return "tools"
        return "end"

    # ------------------------------------------------------------------ #
    #  Graph assembly                                                      #
    # ------------------------------------------------------------------ #
    tool_node = ToolNode(tools)

    builder = StateGraph(AgentState)
    builder.add_node("agent", agent_node)
    builder.add_node("tools", tool_node)

    builder.add_edge(START, "agent")
    builder.add_conditional_edges("agent", should_continue, {"tools": "tools", "end": END})
    builder.add_edge("tools", "agent")

    return builder.compile(checkpointer=checkpointer)
```

---

## 3. create_react_agent (prebuilt shortcut)

```python
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.memory import InMemorySaver
from my_agent.tools import get_tools

llm = ChatOpenAI(model="gpt-4o", temperature=0)
tools = get_tools()

agent = create_react_agent(
    model=llm,
    tools=tools,
    checkpointer=InMemorySaver(),
    # Optional: override system prompt
    state_modifier="You are a helpful coding assistant.",
    # Optional: max iterations guard
    recursion_limit=25,
)

# Sync invoke
result = agent.invoke(
    {"messages": [{"role": "user", "content": "What files are in /tmp?"}]},
    config={"configurable": {"thread_id": "t1"}},
)
```

---

## 4. Reflexion as LangGraph (arXiv:2303.11366)

Reflexion adds episodic memory of verbal self-reflections to guide future attempts.

```python
# src/my_agent/agents/reflexion.py
from __future__ import annotations

import operator
from typing import Annotated, Any, Literal, TypedDict

from langchain_core.messages import BaseMessage, HumanMessage, SystemMessage
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.tools import BaseTool
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from langgraph.store.memory import InMemoryStore


class ReflexionState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    reflections: Annotated[list[str], operator.add]
    trial: int
    score: float
    task: str


ACTOR_PROMPT = ChatPromptTemplate.from_messages([
    ("system", (
        "You are a helpful assistant. Past reflections on similar tasks:\n"
        "{reflections}\n\n"
        "Attempt the task carefully, incorporating these lessons."
    )),
    ("human", "{task}"),
])

EVALUATOR_PROMPT = ChatPromptTemplate.from_messages([
    ("system", "Evaluate the response quality on a scale 0.0-1.0. Return only the float."),
    ("human", "Task: {task}\nResponse: {response}"),
])

REFLECTION_PROMPT = ChatPromptTemplate.from_messages([
    ("system", "Reflect on what went wrong and how to improve. Be concise."),
    ("human", "Task: {task}\nAttempt: {response}\nScore: {score}"),
])


def build_reflexion_graph(
    tools: list[BaseTool],
    score_threshold: float = 0.85,
    max_trials: int = 3,
    store: Any | None = None,
) -> Any:
    llm = ChatOpenAI(model="gpt-4o", temperature=0.3)
    fast_llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

    store = store or InMemoryStore()

    def actor_node(state: ReflexionState) -> dict[str, Any]:
        reflections_text = "\n".join(state.get("reflections", [])) or "None yet."
        chain = ACTOR_PROMPT | llm
        response = chain.invoke({"reflections": reflections_text, "task": state["task"]})
        return {"messages": [response]}

    def evaluator_node(state: ReflexionState) -> dict[str, Any]:
        last_response = state["messages"][-1].content
        result = (EVALUATOR_PROMPT | fast_llm).invoke(
            {"task": state["task"], "response": last_response}
        )
        try:
            score = float(result.content.strip())
        except ValueError:
            score = 0.0
        return {"score": score}

    def reflection_node(state: ReflexionState) -> dict[str, Any]:
        last_response = state["messages"][-1].content
        result = (REFLECTION_PROMPT | fast_llm).invoke({
            "task": state["task"],
            "response": last_response,
            "score": state["score"],
        })
        reflection = result.content

        # Persist reflection to cross-thread store for future use
        namespace = ("reflexion", "reflections")
        store.put(namespace, f"trial-{state['trial']}", {"reflection": reflection})

        return {
            "reflections": [reflection],
            "trial": state["trial"] + 1,
        }

    def route_after_eval(
        state: ReflexionState,
    ) -> Literal["reflection", "end"]:
        if state["score"] >= score_threshold or state["trial"] >= max_trials:
            return "end"
        return "reflection"

    builder = StateGraph(ReflexionState)
    builder.add_node("actor", actor_node)
    builder.add_node("evaluator", evaluator_node)
    builder.add_node("reflection", reflection_node)

    builder.add_edge(START, "actor")
    builder.add_edge("actor", "evaluator")
    builder.add_conditional_edges(
        "evaluator",
        route_after_eval,
        {"reflection": "reflection", "end": END},
    )
    builder.add_edge("reflection", "actor")

    return builder.compile()
```

---

## 5. SELF-REFINE as Iterative Conditional Graph (selfrefine.info)

```python
# src/my_agent/agents/self_refine.py
from __future__ import annotations

from typing import Any, Literal, TypedDict

from langchain_core.messages import BaseMessage
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from typing import Annotated


class RefineState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    draft: str
    critique: str
    iteration: int
    is_satisfactory: bool
    task: str


GENERATOR_PROMPT = ChatPromptTemplate.from_messages([
    ("system", "Generate a high-quality response to the task."),
    ("human", "{task}"),
])

CRITIQUE_PROMPT = ChatPromptTemplate.from_messages([
    ("system", (
        "Critique the following draft. Identify specific improvements needed. "
        "If the draft is satisfactory, reply with exactly: SATISFACTORY"
    )),
    ("human", "Task: {task}\nDraft: {draft}"),
])

REFINE_PROMPT = ChatPromptTemplate.from_messages([
    ("system", "Revise the draft based on the critique. Produce an improved version."),
    ("human", "Task: {task}\nDraft: {draft}\nCritique: {critique}"),
])


def build_self_refine_graph(max_iterations: int = 5) -> Any:
    llm = ChatOpenAI(model="gpt-4o", temperature=0.4)

    def generate_node(state: RefineState) -> dict[str, Any]:
        if state.get("draft"):
            # Refinement pass
            result = (REFINE_PROMPT | llm).invoke({
                "task": state["task"],
                "draft": state["draft"],
                "critique": state["critique"],
            })
        else:
            # Initial generation
            result = (GENERATOR_PROMPT | llm).invoke({"task": state["task"]})
        return {"draft": result.content, "messages": [result]}

    def critique_node(state: RefineState) -> dict[str, Any]:
        result = (CRITIQUE_PROMPT | llm).invoke({
            "task": state["task"],
            "draft": state["draft"],
        })
        is_satisfactory = "SATISFACTORY" in result.content.upper()
        return {
            "critique": result.content,
            "is_satisfactory": is_satisfactory,
            "iteration": state.get("iteration", 0) + 1,
        }

    def should_refine(state: RefineState) -> Literal["generate", "end"]:
        if state["is_satisfactory"] or state["iteration"] >= max_iterations:
            return "end"
        return "generate"

    builder = StateGraph(RefineState)
    builder.add_node("generate", generate_node)
    builder.add_node("critique", critique_node)

    builder.add_edge(START, "generate")
    builder.add_edge("generate", "critique")
    builder.add_conditional_edges("critique", should_refine, {"generate": "generate", "end": END})

    return builder.compile()
```

---

## 6. interrupt() for Human Approval Gate

```python
# src/my_agent/agents/human_in_loop.py
from __future__ import annotations

from typing import Any, Literal, TypedDict

from langchain_core.messages import BaseMessage, HumanMessage
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import Command, interrupt
from typing import Annotated


class HumanGateState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    proposed_action: str
    approved: bool


def plan_node(state: HumanGateState) -> dict[str, Any]:
    # ... agent plans an action ...
    return {"proposed_action": "DELETE all files in /tmp"}


def human_approval_node(state: HumanGateState) -> Command:
    """Pause execution and wait for human input via interrupt()."""
    human_response: str = interrupt(
        value={
            "message": "Please approve or reject this action:",
            "action": state["proposed_action"],
        }
    )
    approved = human_response.strip().lower() in ("yes", "y", "approve", "approved")
    return Command(update={"approved": approved})


def execute_node(state: HumanGateState) -> dict[str, Any]:
    if state["approved"]:
        return {"messages": [HumanMessage(content=f"Executed: {state['proposed_action']}")]}
    return {"messages": [HumanMessage(content="Action rejected by user.")]}


def route_after_approval(state: HumanGateState) -> Literal["execute", "end"]:
    return "execute" if state.get("approved") else "end"


checkpointer = InMemorySaver()
builder = StateGraph(HumanGateState)
builder.add_node("plan", plan_node)
builder.add_node("human_approval", human_approval_node)
builder.add_node("execute", execute_node)

builder.add_edge(START, "plan")
builder.add_edge("plan", "human_approval")
builder.add_conditional_edges(
    "human_approval",
    route_after_approval,
    {"execute": "execute", "end": END},
)
builder.add_edge("execute", END)

graph = builder.compile(checkpointer=checkpointer)


# --- Usage ---
async def run_with_approval() -> None:
    config = {"configurable": {"thread_id": "approval-1"}}

    # First run — agent plans and pauses at interrupt
    result = await graph.ainvoke(
        {"messages": [], "proposed_action": "", "approved": False},
        config=config,
    )
    # result["__interrupt__"] contains the interrupt value

    # Human reviews and resumes
    final = await graph.ainvoke(
        Command(resume="yes"),
        config=config,
    )
    print(final["messages"][-1].content)
```

---

## 7. Streaming — 7 Stream Modes

```python
# src/my_agent/streaming_examples.py
from __future__ import annotations

import asyncio
from typing import Any

from my_agent.graph import build_react_graph
from my_agent.tools import get_tools

graph = build_react_graph(get_tools())
config: dict[str, Any] = {"configurable": {"thread_id": "stream-demo"}}
inputs = {"messages": [{"role": "user", "content": "Search the web for LangGraph news."}]}


async def demo_stream_modes() -> None:

    # 1. "values" — full state snapshot after each step
    async for state_snapshot in graph.astream(inputs, config, stream_mode="values"):
        print("SNAPSHOT:", state_snapshot["messages"][-1])

    # 2. "updates" — partial state dicts (only changed keys)
    async for node_name, update in graph.astream(inputs, config, stream_mode="updates"):
        print(f"NODE={node_name} UPDATE={update}")

    # 3. "messages" — token-by-token from LLM nodes
    async for message_chunk, metadata in graph.astream(
        inputs, config, stream_mode="messages"
    ):
        print(message_chunk.content, end="", flush=True)

    # 4. "custom" — explicit get_stream_writer() events
    async for event in graph.astream(inputs, config, stream_mode="custom"):
        print("CUSTOM EVENT:", event)

    # 5. "checkpoints" — checkpoint saved events
    async for checkpoint_data in graph.astream(
        inputs, config, stream_mode="checkpoints"
    ):
        print("CHECKPOINT:", checkpoint_data["id"])

    # 6. "debug" — every internal event (verbose)
    async for debug_event in graph.astream(inputs, config, stream_mode="debug"):
        print("DEBUG:", debug_event["type"])

    # 7. Multiple modes simultaneously
    async for event_tuple in graph.astream(
        inputs, config, stream_mode=["values", "messages"]
    ):
        mode, data = event_tuple
        print(f"[{mode}]", data)

    # --- astream_events (unified v2 format) ---
    async for event in graph.astream_events(inputs, config, version="v2"):
        match event["event"]:
            case "on_chat_model_stream":
                print(event["data"]["chunk"].content, end="", flush=True)
            case "on_tool_start":
                print(f"\n[TOOL START] {event['name']}")
            case "on_tool_end":
                print(f"[TOOL END] {event['name']}")
            case "on_custom_event":
                print(f"[CUSTOM] {event['name']}: {event['data']}")
```

---

## 8. Custom Events with get_stream_writer()

```python
# Used inside a node to emit custom progress events
from langgraph.config import get_stream_writer
from typing import Any


async def long_running_node(state: Any) -> dict[str, Any]:
    write = get_stream_writer()

    write({"progress": "Starting data fetch...", "pct": 0})
    data = await fetch_data()

    write({"progress": "Processing results...", "pct": 50})
    processed = await process(data)

    write({"progress": "Done", "pct": 100})
    return {"scratchpad": {"result": processed}}
```

---

## 9. Async Graph with Async Nodes

```python
# src/my_agent/async_graph.py
from __future__ import annotations

import asyncio
from typing import Any

import httpx
from langchain_core.messages import AIMessage
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, StateGraph
from langgraph.checkpoint.memory import InMemorySaver

from my_agent.state import AgentState


async def async_fetch_node(state: AgentState) -> dict[str, Any]:
    """Async node — can use await freely."""
    async with httpx.AsyncClient() as client:
        resp = await client.get("https://api.example.com/data", timeout=10)
        return {"scratchpad": {"api_data": resp.json()}}


async def async_llm_node(state: AgentState) -> dict[str, Any]:
    llm = ChatOpenAI(model="gpt-4o", streaming=True)
    response = await llm.ainvoke(state["messages"])
    return {"messages": [response]}


builder = StateGraph(AgentState)
builder.add_node("fetch", async_fetch_node)
builder.add_node("llm", async_llm_node)
builder.add_edge(START, "fetch")
builder.add_edge("fetch", "llm")
builder.add_edge("llm", END)

graph = builder.compile(checkpointer=InMemorySaver())


async def run_async() -> None:
    config = {"configurable": {"thread_id": "async-1"}}
    result = await graph.ainvoke(
        {"messages": [{"role": "user", "content": "Summarize the fetched data."}]},
        config=config,
    )
    print(result["messages"][-1].content)
```

---

## 10. Token Tracking and Auto-Compaction Trigger

```python
# src/my_agent/compaction.py
from __future__ import annotations

from typing import Any

from langchain_core.messages import BaseMessage, SystemMessage
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from typing import Annotated, TypedDict


TOKEN_BUDGET = 80_000  # trigger compaction at 80k tokens


def count_tokens(messages: list[BaseMessage]) -> int:
    """Rough token estimate: 1 token ≈ 4 characters."""
    return sum(len(m.content) // 4 for m in messages if hasattr(m, "content"))


class CompactionState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    token_count: int


def token_check_node(state: CompactionState) -> dict[str, Any]:
    count = count_tokens(state["messages"])
    return {"token_count": count}


async def compaction_node(state: CompactionState) -> dict[str, Any]:
    """Summarize old messages to free context window."""
    llm = ChatOpenAI(model="gpt-4o-mini")
    old_messages = state["messages"][:-5]  # keep last 5 fresh
    summary_prompt = (
        "Summarize the following conversation history concisely, "
        "preserving all important facts and decisions:\n\n"
        + "\n".join(f"{m.type}: {m.content}" for m in old_messages)
    )
    summary = await llm.ainvoke([SystemMessage(content=summary_prompt)])
    # Replace old messages with summary
    new_messages = [summary] + state["messages"][-5:]
    return {"messages": new_messages, "token_count": count_tokens(new_messages)}


def should_compact(state: CompactionState) -> str:
    return "compact" if state["token_count"] > TOKEN_BUDGET else "continue"
```

---

## 11. Complete Example: Code Review Agent

```python
# src/my_agent/agents/code_reviewer.py
from __future__ import annotations

import operator
from typing import Annotated, Any, Literal, TypedDict

from langchain_core.messages import AIMessage, BaseMessage, HumanMessage, SystemMessage
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.memory import InMemorySaver


# --------------------------------------------------------------------------- #
#  State                                                                        #
# --------------------------------------------------------------------------- #
class ReviewState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    code_snippet: str
    review_comments: Annotated[list[str], operator.add]
    severity: str          # "low" | "medium" | "high"
    iteration: int
    approved: bool


# --------------------------------------------------------------------------- #
#  Tools                                                                        #
# --------------------------------------------------------------------------- #
@tool
def check_syntax(code: str, language: str = "python") -> str:
    """Check code for syntax errors using ast.parse."""
    import ast
    try:
        ast.parse(code)
        return "No syntax errors found."
    except SyntaxError as e:
        return f"SyntaxError at line {e.lineno}: {e.msg}"


@tool
def check_security(code: str) -> str:
    """Scan code for obvious security issues (stub — integrate bandit in prod)."""
    issues = []
    if "eval(" in code:
        issues.append("HIGH: Use of eval() detected — potential code injection.")
    if "exec(" in code:
        issues.append("HIGH: Use of exec() detected — potential code injection.")
    if "password" in code.lower() and "=" in code:
        issues.append("MEDIUM: Possible hardcoded credential.")
    return "\n".join(issues) if issues else "No obvious security issues."


# --------------------------------------------------------------------------- #
#  LLM + nodes                                                                  #
# --------------------------------------------------------------------------- #
tools = [check_syntax, check_security]
llm = ChatOpenAI(model="gpt-4o", temperature=0).bind_tools(tools)

REVIEW_SYSTEM = """You are a senior software engineer conducting a code review.
Use your tools to check syntax and security, then provide a structured review.
Return severity as one of: low / medium / high."""


def reviewer_node(state: ReviewState) -> dict[str, Any]:
    messages = [
        SystemMessage(content=REVIEW_SYSTEM),
        HumanMessage(content=f"Review this code:\n```\n{state['code_snippet']}\n```"),
        *state["messages"],
    ]
    response: AIMessage = llm.invoke(messages)
    # Extract severity from response text
    content_lower = response.content.lower()
    severity = "low"
    for level in ("high", "medium", "low"):
        if level in content_lower:
            severity = level
            break
    return {
        "messages": [response],
        "severity": severity,
        "iteration": state.get("iteration", 0) + 1,
    }


def summarize_node(state: ReviewState) -> dict[str, Any]:
    last = state["messages"][-1]
    return {"review_comments": [last.content], "approved": state["severity"] == "low"}


def route(state: ReviewState) -> Literal["tools", "summarize"]:
    last = state["messages"][-1]
    if isinstance(last, AIMessage) and last.tool_calls:
        return "tools"
    return "summarize"


# --------------------------------------------------------------------------- #
#  Graph                                                                        #
# --------------------------------------------------------------------------- #
tool_node = ToolNode(tools)
builder = StateGraph(ReviewState)
builder.add_node("reviewer", reviewer_node)
builder.add_node("tools", tool_node)
builder.add_node("summarize", summarize_node)

builder.add_edge(START, "reviewer")
builder.add_conditional_edges("reviewer", route, {"tools": "tools", "summarize": "summarize"})
builder.add_edge("tools", "reviewer")
builder.add_edge("summarize", END)

code_review_graph = builder.compile(checkpointer=InMemorySaver())


# --- Run it ---
if __name__ == "__main__":
    config = {"configurable": {"thread_id": "review-1"}}
    result = code_review_graph.invoke(
        {
            "messages": [],
            "code_snippet": "password = 'admin123'\neval(user_input)",
            "review_comments": [],
            "severity": "low",
            "iteration": 0,
            "approved": False,
        },
        config=config,
    )
    print("Severity:", result["severity"])
    print("Approved:", result["approved"])
    print("Comments:", result["review_comments"])
```
