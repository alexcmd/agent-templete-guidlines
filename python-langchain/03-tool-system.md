# 03 — Tool System

## 1. @tool Decorator (sync and async)

```python
# src/my_agent/tools/basics.py
from __future__ import annotations

import asyncio
from langchain_core.tools import tool


# --- Sync tool ---
@tool
def add_numbers(a: float, b: float) -> float:
    """Add two numbers together."""
    return a + b


# --- Async tool ---
@tool
async def fetch_url(url: str, timeout: int = 10) -> str:
    """Fetch the content of a URL asynchronously."""
    import httpx
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, timeout=timeout, follow_redirects=True)
        resp.raise_for_status()
        return resp.text[:4000]  # truncate to 4k chars


# --- Tool with return_direct (skip LLM after tool call) ---
@tool(return_direct=True)
def final_answer(answer: str) -> str:
    """Emit a final answer directly to the user."""
    return answer


# --- Tool with tags for filtering ---
@tool(tags=["dangerous", "requires_approval"])
def delete_file(path: str) -> str:
    """Delete a file at the given path."""
    import os
    os.remove(path)
    return f"Deleted {path}"
```

---

## 2. StructuredTool with args_schema (Pydantic BaseModel)

```python
# src/my_agent/tools/structured.py
from __future__ import annotations

from pydantic import BaseModel, Field
from langchain_core.tools import StructuredTool


class SearchInput(BaseModel):
    query: str = Field(description="The search query string")
    max_results: int = Field(default=5, ge=1, le=20, description="Max results to return")
    include_snippets: bool = Field(default=True, description="Include page snippets")


class SearchOutput(BaseModel):
    results: list[dict[str, str]]
    total: int


def _web_search_fn(query: str, max_results: int, include_snippets: bool) -> str:
    """Internal implementation — call Tavily or SerpAPI."""
    from tavily import TavilyClient
    client = TavilyClient()
    resp = client.search(query, max_results=max_results, include_answer=include_snippets)
    results = resp.get("results", [])
    lines = [f"[{i+1}] {r['title']}: {r['url']}\n  {r.get('content','')}" for i, r in enumerate(results)]
    return "\n\n".join(lines)


web_search = StructuredTool.from_function(
    func=_web_search_fn,
    name="web_search",
    description="Search the web for up-to-date information on any topic.",
    args_schema=SearchInput,
    return_direct=False,
)
```

---

## 3. BaseTool for Full Control

```python
# src/my_agent/tools/base_tool_example.py
from __future__ import annotations

from typing import Any, ClassVar
from pydantic import BaseModel, Field
from langchain_core.tools import BaseTool
from langchain_core.callbacks import CallbackManagerForToolRun


class BashInput(BaseModel):
    command: str = Field(description="Shell command to execute")
    timeout: int = Field(default=30, description="Timeout in seconds")
    working_dir: str = Field(default="/tmp", description="Working directory")


class BashTool(BaseTool):
    name: ClassVar[str] = "bash"
    description: ClassVar[str] = (
        "Execute a bash command. Use for file operations, running tests, "
        "or any system-level task. Returns stdout + stderr."
    )
    args_schema: type[BaseModel] = BashInput

    # Allowlist of safe command prefixes
    allowed_prefixes: list[str] = ["ls", "cat", "echo", "python", "pytest", "grep"]
    require_approval: bool = False

    def _run(
        self,
        command: str,
        timeout: int = 30,
        working_dir: str = "/tmp",
        run_manager: CallbackManagerForToolRun | None = None,
    ) -> str:
        import subprocess
        import shlex

        # Safety check
        cmd_base = shlex.split(command)[0] if command else ""
        if self.require_approval and cmd_base not in self.allowed_prefixes:
            return f"ERROR: Command '{cmd_base}' not in allowlist. Request approval."

        try:
            result = subprocess.run(
                command,
                shell=True,
                capture_output=True,
                text=True,
                timeout=timeout,
                cwd=working_dir,
            )
            output = result.stdout + result.stderr
            return output[:8000] if output else "(no output)"
        except subprocess.TimeoutExpired:
            return f"ERROR: Command timed out after {timeout}s"
        except Exception as e:
            return f"ERROR: {type(e).__name__}: {e}"

    async def _arun(self, command: str, timeout: int = 30, working_dir: str = "/tmp") -> str:
        import asyncio
        return await asyncio.get_event_loop().run_in_executor(
            None, self._run, command, timeout, working_dir
        )
```

---

## 4. ToolNode from langgraph.prebuilt

```python
# src/my_agent/tools/tool_node_setup.py
from __future__ import annotations

from langchain_core.messages import AIMessage, ToolMessage
from langgraph.prebuilt import ToolNode
from my_agent.tools import get_all_tools

tools = get_all_tools()
tool_node = ToolNode(tools)

# ToolNode automatically:
# 1. Reads tool_calls from the last AIMessage
# 2. Dispatches to the correct tool by name
# 3. Returns ToolMessage results appended to messages
# 4. Handles errors gracefully with ToolMessage(status="error")

# You can also handle errors explicitly:
tool_node_with_error = ToolNode(
    tools,
    handle_tool_errors=True,           # Return error as ToolMessage instead of raising
    # handle_tool_errors="Custom error message",   # Custom error text
)
```

---

## 5. Permission Enforcement as Pre-Tool Hook

```python
# src/my_agent/tools/permissions.py
from __future__ import annotations

from collections.abc import Callable
from functools import wraps
from typing import Any

from langchain_core.tools import BaseTool


DANGEROUS_TOOLS = {"bash", "delete_file", "write_file", "python_repl"}


def with_permission_check(
    tool: BaseTool,
    approved_tools: set[str],
) -> BaseTool:
    """Wrap a tool so it checks an approved_tools set before running."""
    original_run = tool._run

    @wraps(original_run)
    def checked_run(*args: Any, **kwargs: Any) -> Any:
        if tool.name in DANGEROUS_TOOLS and tool.name not in approved_tools:
            return (
                f"PERMISSION DENIED: '{tool.name}' requires explicit approval. "
                "Ask the user for permission first."
            )
        return original_run(*args, **kwargs)

    tool._run = checked_run  # type: ignore[method-assign]
    return tool


# Usage in graph node:
def build_permission_aware_tool_node(
    tools: list[BaseTool],
    state_getter: Callable[[], set[str]],
) -> Any:
    """Build ToolNode that reads approved_tools from state at runtime."""
    from langgraph.prebuilt import ToolNode

    wrapped = [with_permission_check(t, state_getter()) for t in tools]
    return ToolNode(wrapped)
```

---

## 6. InjectedState — Reading Graph State Inside a Tool

```python
# src/my_agent/tools/stateful_tools.py
from __future__ import annotations

from typing import Annotated, Any
from langchain_core.tools import InjectedToolArg, tool
from langgraph.prebuilt import InjectedState

from my_agent.state import AgentState


@tool
def get_current_task(
    state: Annotated[AgentState, InjectedState],
) -> str:
    """Return the agent's current task from graph state."""
    return state.get("current_task", "No task set.")


@tool
def append_observation(
    observation: str,
    state: Annotated[AgentState, InjectedState],
) -> dict[str, Any]:
    """Add an observation to the state's observation list."""
    # Note: tools cannot directly modify state — they return values
    # The returned dict is treated as a state update by ToolNode
    current = state.get("observations", [])
    return {"observations": current + [observation]}
```

---

## 7. InjectedToolCallId for Handoff Tools

```python
# src/my_agent/tools/handoff_tools.py
from __future__ import annotations

from typing import Annotated
from langchain_core.tools import InjectedToolCallId, tool
from langgraph.types import Command
from langgraph.prebuilt import InjectedState

from my_agent.state import AgentState


def make_handoff_tool(agent_name: str) -> Any:
    """Create a handoff tool that transfers control to another agent."""
    from langchain_core.tools import tool as tool_decorator

    @tool_decorator(f"transfer_to_{agent_name}")
    def handoff(
        task_description: str,
        tool_call_id: Annotated[str, InjectedToolCallId],
        state: Annotated[AgentState, InjectedState],
    ) -> Command:
        """Transfer execution to another agent."""
        from langchain_core.messages import ToolMessage
        return Command(
            goto=agent_name,
            graph=Command.PARENT,
            update={
                "messages": [
                    ToolMessage(
                        content=f"Transferring to {agent_name}: {task_description}",
                        tool_call_id=tool_call_id,
                    )
                ],
                "current_task": task_description,
            },
        )

    return handoff
```

---

## 8. Parallel Tool Execution

LangGraph's ToolNode executes all `tool_calls` in the latest AIMessage concurrently:

```python
# src/my_agent/tools/parallel_demo.py
from __future__ import annotations

import asyncio
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import ToolNode


@tool
async def get_stock_price(symbol: str) -> str:
    """Fetch stock price (simulated)."""
    await asyncio.sleep(0.1)
    prices = {"AAPL": 189.5, "GOOG": 175.2, "MSFT": 415.0}
    return f"{symbol}: ${prices.get(symbol, 0.0)}"


@tool
async def get_news(company: str) -> str:
    """Fetch latest news for a company (simulated)."""
    await asyncio.sleep(0.1)
    return f"Latest news for {company}: strong quarterly earnings reported."


tools = [get_stock_price, get_news]
tool_node = ToolNode(tools)

# The LLM can call both tools in one turn:
# AIMessage(tool_calls=[
#   ToolCall(name="get_stock_price", args={"symbol": "AAPL"}, id="call_1"),
#   ToolCall(name="get_news", args={"company": "Apple"}, id="call_2"),
# ])
# ToolNode dispatches both concurrently and returns two ToolMessages.
```

---

## 9. MCP Tools via langchain-mcp-adapters

```python
# src/my_agent/tools/mcp_tools.py
from __future__ import annotations

import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient


async def get_mcp_tools() -> list:
    """Load tools from one or more MCP servers."""
    client = MultiServerMCPClient({
        "filesystem": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"],
            "transport": "stdio",
        },
        "github": {
            "url": "https://mcp.example.com/github/sse",
            "transport": "sse",
        },
    })
    tools = await client.get_tools()
    return tools


# Integration in graph build:
async def build_graph_with_mcp() -> Any:
    from langgraph.prebuilt import create_react_agent
    from langchain_openai import ChatOpenAI

    mcp_tools = await get_mcp_tools()
    llm = ChatOpenAI(model="gpt-4o")
    return create_react_agent(llm, tools=mcp_tools)
```

---

## 10. Complete Tool Examples

### bash tool
```python
bash_tool = BashTool(allowed_prefixes=["ls", "cat", "python", "pytest", "git"])
```

### read_file and write_file
```python
from langchain_core.tools import tool
from pathlib import Path


@tool
def read_file(path: str) -> str:
    """Read the contents of a file."""
    try:
        return Path(path).read_text(encoding="utf-8")
    except FileNotFoundError:
        return f"ERROR: File not found: {path}"
    except PermissionError:
        return f"ERROR: Permission denied: {path}"


@tool
def write_file(path: str, content: str) -> str:
    """Write content to a file, creating parent dirs if needed."""
    p = Path(path)
    p.parent.mkdir(parents=True, exist_ok=True)
    p.write_text(content, encoding="utf-8")
    return f"Written {len(content)} bytes to {path}"
```

### python_repl
```python
@tool
def python_repl(code: str) -> str:
    """Execute Python code in an isolated namespace. Returns stdout output."""
    import io
    import sys
    import contextlib
    namespace: dict = {}
    buf = io.StringIO()
    with contextlib.redirect_stdout(buf):
        try:
            exec(compile(code, "<string>", "exec"), namespace)  # noqa: S102
        except Exception as e:
            return f"ERROR: {type(e).__name__}: {e}"
    return buf.getvalue() or "(no output)"
```

---

## 11. Hierarchical Tool Retrieval (AnyTool Pattern)

When the agent has 100+ tools, embed tool descriptions and retrieve relevant ones per query:

```python
# src/my_agent/tools/tool_retrieval.py
from __future__ import annotations

from langchain_core.tools import BaseTool
from langchain_core.vectorstores import VectorStore
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS


class ToolRetriever:
    """Retrieve top-k relevant tools from a large tool pool by embedding similarity."""

    def __init__(self, tools: list[BaseTool], top_k: int = 5) -> None:
        self.tools = {t.name: t for t in tools}
        self.top_k = top_k
        embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
        docs = [f"{t.name}: {t.description}" for t in tools]
        self._vs: VectorStore = FAISS.from_texts(docs, embeddings)

    def get_relevant_tools(self, query: str) -> list[BaseTool]:
        results = self._vs.similarity_search(query, k=self.top_k)
        names = [doc.page_content.split(":")[0].strip() for doc in results]
        return [self.tools[n] for n in names if n in self.tools]


# Usage in a graph node:
# retriever = ToolRetriever(all_tools, top_k=5)
# relevant = retriever.get_relevant_tools(state["messages"][-1].content)
# llm_with_dynamic_tools = llm.bind_tools(relevant)
```

---

## 12. Tool Allowlist and Filtering

```python
# src/my_agent/tools/__init__.py
from __future__ import annotations

from langchain_core.tools import BaseTool


_ALL_TOOLS: list[BaseTool] = []   # populated at import


def register_tool(tool: BaseTool) -> BaseTool:
    _ALL_TOOLS.append(tool)
    return tool


def get_tools(
    allowed: list[str] | None = None,
    tags_filter: list[str] | None = None,
) -> list[BaseTool]:
    """Return tools filtered by name allowlist and/or tags."""
    result = list(_ALL_TOOLS)
    if allowed is not None:
        result = [t for t in result if t.name in allowed]
    if tags_filter is not None:
        result = [t for t in result if any(tag in (t.tags or []) for tag in tags_filter)]
    return result
```
