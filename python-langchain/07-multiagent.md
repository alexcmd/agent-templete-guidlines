# 07 — Multi-Agent Patterns

## 1. LangGraph Supervisor Pattern with create_supervisor

```python
# src/my_agent/agents/supervisor_graph.py
from __future__ import annotations

from langchain_openai import ChatOpenAI
from langgraph.graph import END
from langgraph_supervisor import create_supervisor

from my_agent.agents.researcher import build_researcher_agent
from my_agent.agents.coder import build_coder_agent
from my_agent.agents.reviewer import build_reviewer_agent
from my_agent.memory.checkpointer import get_checkpointer


async def build_supervisor_graph() -> Any:
    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    checkpointer = await get_checkpointer("postgres")

    researcher = build_researcher_agent()
    coder = build_coder_agent()
    reviewer = build_reviewer_agent()

    # create_supervisor wraps all sub-agents and provides handoff tools automatically
    supervisor = create_supervisor(
        agents=[researcher, coder, reviewer],
        model=llm,
        prompt=(
            "You are a software development supervisor. "
            "Route tasks to: researcher (web search, docs), "
            "coder (implementation), reviewer (code review). "
            "When all steps are complete, respond with FINAL ANSWER."
        ),
        # Output mode: "last_message" or "full_history"
        output_mode="last_message",
    )

    return supervisor.compile(checkpointer=checkpointer)
```

---

## 2. Custom Supervisor with Conditional Routing

```python
# src/my_agent/agents/custom_supervisor.py
from __future__ import annotations

import re
from typing import Any, Literal, TypedDict

from langchain_core.messages import AIMessage, BaseMessage, SystemMessage
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from typing import Annotated


SUPERVISOR_SYSTEM = """You are an orchestrator. Decide which specialist to call next.

Available specialists:
- researcher: for information gathering, web search, document retrieval
- coder: for writing, editing, or debugging code
- reviewer: for reviewing code, checking quality, suggesting improvements
- FINISH: when the task is complete

Respond with only one word: researcher, coder, reviewer, or FINISH."""


class SupervisorState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    next_agent: str
    task: str
    iteration: int


def build_custom_supervisor(
    researcher_graph: Any,
    coder_graph: Any,
    reviewer_graph: Any,
    max_iterations: int = 10,
) -> Any:
    llm = ChatOpenAI(model="gpt-4o", temperature=0)

    def supervisor_node(state: SupervisorState) -> dict[str, Any]:
        messages = [SystemMessage(content=SUPERVISOR_SYSTEM)] + state["messages"]
        response: AIMessage = llm.invoke(messages)
        content = response.content.strip().lower()
        # Parse routing decision
        for agent in ("researcher", "coder", "reviewer", "finish"):
            if agent in content:
                return {
                    "next_agent": agent,
                    "iteration": state.get("iteration", 0) + 1,
                }
        return {"next_agent": "finish"}

    def route_supervisor(state: SupervisorState) -> Literal["researcher", "coder", "reviewer", "end"]:
        if state.get("iteration", 0) >= max_iterations:
            return "end"
        match state["next_agent"]:
            case "researcher":
                return "researcher"
            case "coder":
                return "coder"
            case "reviewer":
                return "reviewer"
            case _:
                return "end"

    def make_agent_node(agent_graph: Any, name: str):
        def node(state: SupervisorState) -> dict[str, Any]:
            result = agent_graph.invoke({"messages": state["messages"], "task": state["task"]})
            # Return only the new messages from the sub-agent
            new_msgs = result["messages"][len(state["messages"]):]
            return {"messages": new_msgs}
        node.__name__ = name
        return node

    builder = StateGraph(SupervisorState)
    builder.add_node("supervisor", supervisor_node)
    builder.add_node("researcher", make_agent_node(researcher_graph, "researcher"))
    builder.add_node("coder", make_agent_node(coder_graph, "coder"))
    builder.add_node("reviewer", make_agent_node(reviewer_graph, "reviewer"))

    builder.add_edge(START, "supervisor")
    builder.add_conditional_edges(
        "supervisor",
        route_supervisor,
        {"researcher": "researcher", "coder": "coder", "reviewer": "reviewer", "end": END},
    )
    # All agents return to supervisor after completion
    for agent_name in ("researcher", "coder", "reviewer"):
        builder.add_edge(agent_name, "supervisor")

    return builder.compile()
```

---

## 3. Handoff Tools with Command(goto, update, graph=PARENT)

```python
# src/my_agent/agents/handoff.py
from __future__ import annotations

from typing import Annotated, Any
from langchain_core.messages import ToolMessage
from langchain_core.tools import InjectedToolCallId, tool
from langgraph.prebuilt import InjectedState
from langgraph.types import Command


def make_handoff_tool(target_agent: str, description: str | None = None) -> Any:
    """
    Creates a tool that hands off control to a named sub-agent.
    Uses Command(goto=..., graph=Command.PARENT) to escape current subgraph.
    """
    tool_name = f"transfer_to_{target_agent}"
    tool_description = description or f"Transfer the current task to the {target_agent} agent."

    @tool(tool_name)
    def handoff_tool(
        task: str,
        context: str = "",
        tool_call_id: Annotated[str, InjectedToolCallId] = "",
    ) -> Command:
        """Hand off to another agent."""
        return Command(
            goto=target_agent,
            graph=Command.PARENT,  # escapes current subgraph to parent graph
            update={
                "messages": [
                    ToolMessage(
                        content=f"Transferring to {target_agent}. Task: {task}",
                        tool_call_id=tool_call_id,
                    )
                ],
                "current_task": task,
                "context": context,
            },
        )

    # Override description
    handoff_tool.description = tool_description  # type: ignore
    return handoff_tool


# Pre-built handoff tools
transfer_to_researcher = make_handoff_tool("researcher", "Search for information or research a topic.")
transfer_to_coder = make_handoff_tool("coder", "Write or edit code to implement a feature.")
transfer_to_reviewer = make_handoff_tool("reviewer", "Review and critique code or a solution.")
```

---

## 4. Agent-as-Tool: Wrap Subgraph as @tool

```python
# src/my_agent/agents/agent_as_tool.py
from __future__ import annotations

from typing import Any
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent


def build_researcher_tool() -> Any:
    """Wrap the researcher agent as a LangChain tool callable from a supervisor."""
    from my_agent.tools.web_search import web_search
    from my_agent.tools.file_io import read_file

    researcher_agent = create_react_agent(
        model=ChatOpenAI(model="gpt-4o-mini"),
        tools=[web_search, read_file],
        state_modifier="You are a researcher. Gather information and return a concise summary.",
    )

    @tool
    async def researcher(task: str) -> str:
        """Research a topic and return findings. Handles web search and document retrieval."""
        result = await researcher_agent.ainvoke(
            {"messages": [{"role": "user", "content": task}]}
        )
        return result["messages"][-1].content

    return researcher


def build_coder_tool() -> Any:
    """Wrap the coder agent as a tool."""
    from my_agent.tools.bash import BashTool
    from my_agent.tools.file_io import read_file, write_file

    coder_agent = create_react_agent(
        model=ChatOpenAI(model="gpt-4o"),
        tools=[BashTool(), read_file, write_file],
        state_modifier="You are a senior software engineer. Write clean, tested Python code.",
    )

    @tool
    async def coder(task: str) -> str:
        """Implement a coding task. Returns the code and any test results."""
        result = await coder_agent.ainvoke(
            {"messages": [{"role": "user", "content": task}]}
        )
        return result["messages"][-1].content

    return coder
```

---

## 5. Swarm Pattern: Any Agent Can Hand Off to Any Other

```python
# src/my_agent/agents/swarm.py
from __future__ import annotations

from typing import Any

from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from typing import Annotated, TypedDict
from langchain_core.messages import BaseMessage


class SwarmState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    active_agent: str
    task: str


def build_swarm(agent_configs: dict[str, dict]) -> Any:
    """
    Swarm: each agent has access to handoff tools for all other agents.
    The active_agent field determines which agent runs next.
    """
    from langgraph.prebuilt import create_react_agent
    from my_agent.agents.handoff import make_handoff_tool

    agent_names = list(agent_configs.keys())
    builder = StateGraph(SwarmState)

    for agent_name, config in agent_configs.items():
        # Each agent gets handoff tools to all *other* agents
        handoff_tools = [make_handoff_tool(n) for n in agent_names if n != agent_name]
        all_tools = config.get("tools", []) + handoff_tools

        agent = create_react_agent(
            model=ChatOpenAI(model=config.get("model", "gpt-4o-mini")),
            tools=all_tools,
            state_modifier=config.get("system_prompt", ""),
        )

        def make_node(ag=agent):
            async def node(state: SwarmState) -> dict[str, Any]:
                result = await ag.ainvoke({"messages": state["messages"]})
                new_msgs = result["messages"][len(state["messages"]):]
                return {"messages": new_msgs}
            return node

        builder.add_node(agent_name, make_node())

    def route_initial(state: SwarmState) -> str:
        return state["active_agent"]

    def route_after_agent(state: SwarmState) -> str:
        # Check if the last message is a handoff command
        last = state["messages"][-1] if state["messages"] else None
        if last and hasattr(last, "additional_kwargs"):
            tool_calls = last.additional_kwargs.get("tool_calls", [])
            for tc in tool_calls:
                name = tc.get("function", {}).get("name", "")
                if name.startswith("transfer_to_"):
                    target = name.replace("transfer_to_", "")
                    if target in agent_names:
                        return target
        return END

    builder.add_conditional_edges(START, route_initial, {n: n for n in agent_names})
    for name in agent_names:
        builder.add_conditional_edges(name, route_after_agent, {n: n for n in agent_names} | {END: END})

    return builder.compile()
```

---

## 6. MetaGPT SOP as LangGraph Pipeline

```python
# src/my_agent/agents/metagpt_pipeline.py
from __future__ import annotations

from typing import Any, TypedDict

from langchain_core.messages import BaseMessage
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from typing import Annotated


class SoftwareProjectState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    requirements: str
    prd: str            # Product Requirements Document
    architecture: str   # System design
    code: str           # Implementation
    tests: str          # Test suite
    qa_report: str


def make_role_node(role: str, system_prompt: str, output_field: str):
    llm = ChatOpenAI(model="gpt-4o", temperature=0.2)
    prompt = ChatPromptTemplate.from_messages([
        ("system", system_prompt),
        ("human", "{context}"),
    ])
    chain = prompt | llm

    async def node(state: SoftwareProjectState) -> dict[str, Any]:
        context = build_context(state)
        result = await chain.ainvoke({"context": context})
        return {output_field: result.content}

    node.__name__ = role
    return node


def build_context(state: SoftwareProjectState) -> str:
    parts = [f"Requirements: {state.get('requirements', '')}"]
    if state.get("prd"):
        parts.append(f"PRD:\n{state['prd']}")
    if state.get("architecture"):
        parts.append(f"Architecture:\n{state['architecture']}")
    if state.get("code"):
        parts.append(f"Code:\n{state['code']}")
    return "\n\n".join(parts)


PM_PROMPT = "You are a Product Manager. Write a detailed PRD for the given requirements."
ARCH_PROMPT = "You are a Software Architect. Design the system architecture based on the PRD."
DEV_PROMPT = "You are a Senior Developer. Implement the architecture as clean Python code."
QA_PROMPT = "You are a QA Engineer. Write pytest tests and identify any issues in the code."


def build_metagpt_pipeline() -> Any:
    builder = StateGraph(SoftwareProjectState)
    builder.add_node("product_manager", make_role_node("product_manager", PM_PROMPT, "prd"))
    builder.add_node("architect", make_role_node("architect", ARCH_PROMPT, "architecture"))
    builder.add_node("developer", make_role_node("developer", DEV_PROMPT, "code"))
    builder.add_node("qa_engineer", make_role_node("qa_engineer", QA_PROMPT, "qa_report"))

    builder.add_edge(START, "product_manager")
    builder.add_edge("product_manager", "architect")
    builder.add_edge("architect", "developer")
    builder.add_edge("developer", "qa_engineer")
    builder.add_edge("qa_engineer", END)

    return builder.compile()
```

---

## 7. CrewAI Integration

```python
# src/my_agent/agents/crewai_integration.py
from __future__ import annotations

from crewai import Agent, Crew, Process, Task
from crewai.tools import BaseTool
from langchain_openai import ChatOpenAI


def build_research_crew(topic: str) -> str:
    """Build a CrewAI crew with sequential process for research."""
    llm = ChatOpenAI(model="gpt-4o", temperature=0.3)

    researcher = Agent(
        role="Senior Research Analyst",
        goal=f"Research {topic} thoroughly and provide comprehensive findings",
        backstory="Expert at gathering and analyzing information from multiple sources",
        llm=llm,
        verbose=True,
        allow_delegation=False,
    )

    writer = Agent(
        role="Technical Writer",
        goal="Transform research findings into clear, actionable reports",
        backstory="Expert at communicating complex technical information clearly",
        llm=llm,
        verbose=True,
    )

    research_task = Task(
        description=f"Research the topic: {topic}. Gather key facts, trends, and insights.",
        expected_output="A detailed research report with key findings and citations.",
        agent=researcher,
    )

    writing_task = Task(
        description="Transform the research into a concise executive summary.",
        expected_output="A 500-word executive summary suitable for a non-technical audience.",
        agent=writer,
        context=[research_task],
    )

    crew = Crew(
        agents=[researcher, writer],
        tasks=[research_task, writing_task],
        process=Process.sequential,
        verbose=True,
    )

    result = crew.kickoff()
    return str(result)
```

---

## 8. AutoGen v0.4 ConversableAgent Integration

```python
# src/my_agent/agents/autogen_integration.py
from __future__ import annotations

from autogen import ConversableAgent, UserProxyAgent


def build_autogen_pair(task: str) -> str:
    """Two-agent AutoGen setup: assistant + user proxy with code execution."""
    assistant = ConversableAgent(
        name="assistant",
        system_message=(
            "You are a helpful assistant that writes and runs Python code. "
            "Always verify your solution works."
        ),
        llm_config={"model": "gpt-4o", "temperature": 0},
    )

    user_proxy = UserProxyAgent(
        name="user_proxy",
        human_input_mode="NEVER",   # fully automated
        max_consecutive_auto_reply=10,
        code_execution_config={"work_dir": "/tmp/autogen", "use_docker": False},
        is_termination_msg=lambda x: "TERMINATE" in (x.get("content") or ""),
    )

    result = user_proxy.initiate_chat(assistant, message=task)
    return result.summary
```

---

## 9. Parallel Execution with LangGraph BSP

LangGraph executes nodes in the same "super-step" concurrently (Pregel BSP model):

```python
# src/my_agent/agents/parallel_agents.py
from __future__ import annotations

from typing import Any, TypedDict

from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from typing import Annotated
from langchain_core.messages import BaseMessage
import operator


class ParallelState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    research_result: str
    analysis_result: str
    combined_report: str


async def research_node(state: ParallelState) -> dict[str, Any]:
    llm = ChatOpenAI(model="gpt-4o-mini")
    result = await llm.ainvoke(f"Research: {state['messages'][-1].content}")
    return {"research_result": result.content}


async def analysis_node(state: ParallelState) -> dict[str, Any]:
    llm = ChatOpenAI(model="gpt-4o-mini")
    result = await llm.ainvoke(f"Analyze: {state['messages'][-1].content}")
    return {"analysis_result": result.content}


async def combine_node(state: ParallelState) -> dict[str, Any]:
    llm = ChatOpenAI(model="gpt-4o")
    prompt = (
        f"Research findings: {state['research_result']}\n"
        f"Analysis: {state['analysis_result']}\n"
        "Synthesize into a final report."
    )
    result = await llm.ainvoke(prompt)
    return {"combined_report": result.content}


def build_parallel_graph() -> Any:
    builder = StateGraph(ParallelState)
    builder.add_node("research", research_node)
    builder.add_node("analysis", analysis_node)
    builder.add_node("combine", combine_node)

    # Both research and analysis start simultaneously from START
    builder.add_edge(START, "research")
    builder.add_edge(START, "analysis")

    # Both must complete before combine runs
    builder.add_edge("research", "combine")
    builder.add_edge("analysis", "combine")
    builder.add_edge("combine", END)

    return builder.compile()
```

---

## 10. Human-in-the-Loop Multi-Agent Approval Gates

```python
# src/my_agent/agents/hitl_multiagent.py
from __future__ import annotations

from typing import Any, TypedDict

from langchain_core.messages import BaseMessage
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from langgraph.types import Command, interrupt
from typing import Annotated


class TeamState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    plan: str
    code: str
    approved: bool
    reviewer_feedback: str


def planning_node(state: TeamState) -> dict[str, Any]:
    # ... generate plan ...
    return {"plan": "Implement feature X using approach Y."}


def coding_node(state: TeamState) -> dict[str, Any]:
    # ... generate code based on plan ...
    return {"code": "def feature_x():\n    pass  # TODO"}


def human_review_gate(state: TeamState) -> Command:
    """Pause for human review of the generated code."""
    response: dict = interrupt({
        "instruction": "Review the proposed code and approve or provide feedback.",
        "plan": state["plan"],
        "code": state["code"],
    })
    approved = response.get("approved", False)
    feedback = response.get("feedback", "")
    return Command(update={"approved": approved, "reviewer_feedback": feedback})


def route_after_review(state: TeamState) -> str:
    return "done" if state["approved"] else "coding"  # retry if rejected


def build_hitl_pipeline() -> Any:
    builder = StateGraph(TeamState)
    builder.add_node("plan", planning_node)
    builder.add_node("code", coding_node)
    builder.add_node("review", human_review_gate)

    builder.add_edge(START, "plan")
    builder.add_edge("plan", "code")
    builder.add_edge("code", "review")
    builder.add_conditional_edges(
        "review",
        route_after_review,
        {"done": END, "coding": "code"},
    )

    from langgraph.checkpoint.memory import InMemorySaver
    return builder.compile(checkpointer=InMemorySaver())
```
