# 11 — Complete Example: Research + Code Generation Agent

## Overview

A multi-agent system that:
1. Receives a software feature request
2. **Researcher** gathers relevant documentation and examples via RAG + web search
3. **Planner** produces a structured implementation plan
4. **Coder** implements the feature, running tests in a loop
5. **Reviewer** provides final code review
6. LangSmith traces every step; PostgreSQL persists sessions; Qdrant stores docs

---

## 1. pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "research-code-agent"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "langchain>=0.3.0,<0.4",
    "langgraph>=0.2.0,<0.3",
    "langsmith>=0.2.0,<0.3",
    "langgraph-supervisor>=0.0.10",
    "langgraph-checkpoint-postgres>=2.0.0",
    "langchain-openai>=0.2.0",
    "langchain-community>=0.3.0",
    "langchain-qdrant>=0.2.0",
    "qdrant-client>=1.12.0",
    "langchain-postgres>=0.0.12",
    "psycopg[binary]>=3.2.0",
    "tavily-python>=0.5.0",
    "langchain-tavily>=0.1.0",
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.32.0",
    "sse-starlette>=2.1.0",
    "pydantic>=2.9.0",
    "pydantic-settings>=2.5.0",
    "python-dotenv>=1.0.0",
    "flashrank>=0.2.9",
    "rank-bm25>=0.2.2",
    "aiofiles>=24.0.0",
    "httpx>=0.27.0",
]

[project.optional-dependencies]
dev = ["ruff", "mypy", "pytest", "pytest-asyncio", "ipykernel"]

[tool.hatch.build.targets.wheel]
packages = ["src/research_code_agent"]
```

---

## 2. FastAPI app.py

```python
# src/research_code_agent/app.py
from __future__ import annotations

import asyncio
import json
import uuid
from collections.abc import AsyncGenerator
from contextlib import asynccontextmanager
from typing import Any

import uvicorn
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from sse_starlette.sse import EventSourceResponse

from research_code_agent.config import settings
from research_code_agent.graph import build_agent_graph
from research_code_agent.memory.checkpointer import get_checkpointer


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    checkpointer = await get_checkpointer("postgres")
    app.state.graph = await build_agent_graph(checkpointer=checkpointer)
    yield


app = FastAPI(title="Research + Code Agent API", version="1.0.0", lifespan=lifespan)
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])


class FeatureRequest(BaseModel):
    description: str
    thread_id: str | None = None
    language: str = "python"


@app.post("/feature/stream")
async def stream_feature_implementation(req: FeatureRequest) -> EventSourceResponse:
    thread_id = req.thread_id or f"feature-{uuid.uuid4()}"
    config: dict[str, Any] = {"configurable": {"thread_id": thread_id}}

    async def generate() -> AsyncGenerator[str, None]:
        graph = app.state.graph
        inputs = {
            "messages": [{"role": "user", "content": req.description}],
            "language": req.language,
            "research_docs": [],
            "plan": "",
            "code": "",
            "test_output": "",
            "review": "",
            "iteration": 0,
        }
        async for event in graph.astream_events(inputs, config=config, version="v2"):
            kind = event["event"]
            if kind == "on_chat_model_stream":
                token = event["data"]["chunk"].content
                if token:
                    yield f"event: token\ndata: {json.dumps({'token': token})}\n\n"
            elif kind == "on_chain_start" and "langgraph_node" in event.get("metadata", {}):
                node = event["metadata"]["langgraph_node"]
                yield f"event: node_start\ndata: {json.dumps({'node': node})}\n\n"
            elif kind == "on_tool_start":
                yield f"event: tool_start\ndata: {json.dumps({'tool': event['name']})}\n\n"
            elif kind == "on_custom_event":
                yield f"event: {event['name']}\ndata: {json.dumps(event['data'])}\n\n"
        yield f"event: done\ndata: {json.dumps({'thread_id': thread_id})}\n\n"

    return EventSourceResponse(generate())


@app.get("/threads/{thread_id}/state")
async def get_state(thread_id: str) -> dict[str, Any]:
    config = {"configurable": {"thread_id": thread_id}}
    state = await app.state.graph.aget_state(config)
    if state is None:
        raise HTTPException(404, "Thread not found")
    return {
        "code": state.values.get("code", ""),
        "review": state.values.get("review", ""),
        "plan": state.values.get("plan", ""),
    }


if __name__ == "__main__":
    uvicorn.run("research_code_agent.app:app", host="0.0.0.0", port=8000, reload=True)
```

---

## 3. LangGraph StateGraph

```python
# src/research_code_agent/graph.py
from __future__ import annotations

import operator
from typing import Annotated, Any, Literal, TypedDict

from langchain_core.messages import AIMessage, BaseMessage, SystemMessage, HumanMessage
from langchain_core.tools import tool, BaseTool
from langchain_openai import ChatOpenAI
from langgraph.config import get_stream_writer
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode

from research_code_agent.tools.registry import get_all_tools
from research_code_agent.rag.pipeline import build_rag_chain


class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    language: str
    research_docs: Annotated[list[str], operator.add]
    plan: str
    code: str
    test_output: str
    review: str
    iteration: int
    approved: bool


async def build_agent_graph(checkpointer: Any = None) -> Any:
    llm = ChatOpenAI(model="gpt-4o", temperature=0, streaming=True)
    tools = get_all_tools()
    llm_with_tools = llm.bind_tools(tools)
    rag_chain = await build_rag_chain()

    # ---- Research node ----
    async def researcher_node(state: AgentState) -> dict[str, Any]:
        write = get_stream_writer()
        write({"type": "progress", "step": "research", "message": "Searching documentation..."})

        task = state["messages"][-1].content if state["messages"] else ""
        docs = await rag_chain.ainvoke({"question": task})
        write({"type": "progress", "step": "research", "message": f"Found {len(docs)} relevant docs"})

        system = (
            f"You are a senior {state['language']} researcher. "
            "Use the provided documentation to gather relevant context.\n\n"
            f"Relevant docs:\n{docs}"
        )
        response = await llm_with_tools.ainvoke([
            SystemMessage(content=system),
            *state["messages"],
        ])
        return {"messages": [response], "research_docs": [str(docs)]}

    # ---- Planner node ----
    async def planner_node(state: AgentState) -> dict[str, Any]:
        research = "\n\n".join(state["research_docs"][-3:])
        system = (
            f"You are a {state['language']} software architect. "
            "Create a detailed implementation plan based on the research.\n\n"
            f"Research context:\n{research}"
        )
        response = await llm.ainvoke([
            SystemMessage(content=system),
            HumanMessage(content=f"Create implementation plan for: {state['messages'][0].content}"),
        ])
        return {"plan": response.content, "messages": [response]}

    # ---- Coder node ----
    async def coder_node(state: AgentState) -> dict[str, Any]:
        write = get_stream_writer()
        write({"type": "progress", "step": "coding", "message": "Implementing..."})

        system = (
            f"You are a senior {state['language']} engineer. "
            "Implement the plan with clean, tested, production-quality code.\n\n"
            f"Implementation plan:\n{state['plan']}\n\n"
            f"Previous test output (if any):\n{state.get('test_output', 'No tests run yet.')}"
        )
        response = await llm_with_tools.ainvoke([
            SystemMessage(content=system),
            HumanMessage(content=f"Implement: {state['messages'][0].content}"),
        ])
        return {
            "code": response.content,
            "messages": [response],
            "iteration": state.get("iteration", 0) + 1,
        }

    # ---- Reviewer node ----
    async def reviewer_node(state: AgentState) -> dict[str, Any]:
        system = (
            "You are a senior engineer conducting a code review. "
            "Check for: correctness, edge cases, security, performance, readability.\n"
            "If approved, start with 'APPROVED:'. Otherwise start with 'NEEDS WORK:'"
        )
        response = await llm.ainvoke([
            SystemMessage(content=system),
            HumanMessage(content=f"Review this {state['language']} code:\n\n{state['code']}"),
        ])
        approved = response.content.strip().upper().startswith("APPROVED")
        return {"review": response.content, "approved": approved, "messages": [response]}

    # ---- Routing ----
    def route_researcher(state: AgentState) -> Literal["tools", "planner"]:
        last = state["messages"][-1]
        if isinstance(last, AIMessage) and last.tool_calls:
            return "tools"
        return "planner"

    def route_coder(state: AgentState) -> Literal["tools", "reviewer"]:
        last = state["messages"][-1]
        if isinstance(last, AIMessage) and last.tool_calls:
            return "tools"
        return "reviewer"

    def route_reviewer(state: AgentState) -> Literal["coder", "end"]:
        if state.get("approved") or state.get("iteration", 0) >= 3:
            return "end"
        return "coder"

    # ---- Build graph ----
    tool_node = ToolNode(tools)
    builder = StateGraph(AgentState)

    builder.add_node("researcher", researcher_node)
    builder.add_node("planner", planner_node)
    builder.add_node("coder", coder_node)
    builder.add_node("reviewer", reviewer_node)
    builder.add_node("tools", tool_node)

    builder.add_edge(START, "researcher")
    builder.add_conditional_edges("researcher", route_researcher,
                                  {"tools": "tools", "planner": "planner"})
    builder.add_edge("tools", "researcher")
    builder.add_edge("planner", "coder")
    builder.add_conditional_edges("coder", route_coder,
                                  {"tools": "tools", "reviewer": "reviewer"})
    builder.add_edge("tools", "coder")
    builder.add_conditional_edges("reviewer", route_reviewer,
                                  {"coder": "coder", "end": END})

    return builder.compile(checkpointer=checkpointer)
```

---

## 4. Tool Registry

```python
# src/research_code_agent/tools/registry.py
from __future__ import annotations

import subprocess
from pathlib import Path
from langchain_core.tools import tool, BaseTool
from langchain_community.tools.tavily_search import TavilySearchResults


@tool
def bash(command: str, timeout: int = 30) -> str:
    """Execute a bash command. Returns stdout + stderr."""
    try:
        result = subprocess.run(
            command, shell=True, capture_output=True, text=True, timeout=timeout
        )
        return (result.stdout + result.stderr)[:5000] or "(no output)"
    except subprocess.TimeoutExpired:
        return f"Timed out after {timeout}s"
    except Exception as e:
        return f"Error: {e}"


@tool
def read_file(path: str) -> str:
    """Read a file from the filesystem."""
    try:
        return Path(path).read_text(encoding="utf-8")
    except FileNotFoundError:
        return f"File not found: {path}"


@tool
def write_file(path: str, content: str) -> str:
    """Write content to a file, creating directories as needed."""
    p = Path(path)
    p.parent.mkdir(parents=True, exist_ok=True)
    p.write_text(content, encoding="utf-8")
    return f"Written {len(content)} bytes to {path}"


@tool
async def run_tests(test_path: str = "tests/") -> str:
    """Run pytest on the specified path and return results."""
    import asyncio
    proc = await asyncio.create_subprocess_shell(
        f"python -m pytest {test_path} -v --tb=short 2>&1",
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.STDOUT,
    )
    stdout, _ = await proc.communicate()
    return stdout.decode()[:4000]


web_search = TavilySearchResults(max_results=5)


def get_all_tools() -> list[BaseTool]:
    return [bash, read_file, write_file, run_tests, web_search]
```

---

## 5. RAG Pipeline over Documentation

```python
# src/research_code_agent/rag/pipeline.py
from __future__ import annotations

from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_qdrant import QdrantVectorStore, RetrievalMode, FastEmbedSparse
from langchain_core.documents import Document
from typing import Any


RAG_PROMPT = ChatPromptTemplate.from_messages([
    ("system", (
        "Use the following documentation to help answer the question.\n\n"
        "Documentation:\n{context}"
    )),
    ("human", "{question}"),
])


async def build_rag_chain() -> Any:
    embeddings = OpenAIEmbeddings(model="text-embedding-3-large")
    try:
        vectorstore = QdrantVectorStore.from_existing_collection(
            embedding=embeddings,
            sparse_embedding=FastEmbedSparse(model_name="Qdrant/bm25"),
            url="http://localhost:6333",
            collection_name="agent_docs",
            retrieval_mode=RetrievalMode.HYBRID,
        )
    except Exception:
        # Fallback to FAISS for dev without Qdrant
        from langchain_community.vectorstores import FAISS
        vectorstore = FAISS.from_texts(
            ["No documentation indexed yet."], embeddings
        )

    retriever = vectorstore.as_retriever(search_kwargs={"k": 6})
    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

    def format_docs(docs: list[Document]) -> str:
        return "\n\n---\n\n".join(d.page_content for d in docs)

    chain = (
        RunnablePassthrough.assign(context=lambda x: format_docs(retriever.invoke(x["question"])))
        | RAG_PROMPT
        | llm
        | StrOutputParser()
    )
    return chain
```

---

## 6. LangSmith Tracing Setup

```python
# src/research_code_agent/observability.py
from __future__ import annotations

import os
from langsmith import Client
from langsmith.evaluation import evaluate


def setup_tracing(project_name: str = "research-code-agent") -> None:
    """Configure LangSmith auto-tracing."""
    os.environ["LANGSMITH_TRACING"] = "true"
    os.environ["LANGSMITH_PROJECT"] = project_name


def create_eval_dataset(name: str = "feature-requests") -> str:
    """Create a LangSmith evaluation dataset."""
    client = Client()
    dataset = client.create_dataset(
        name,
        description="Feature request → code generation evaluation set",
    )
    examples = [
        {
            "inputs": {"description": "Implement a binary search function"},
            "outputs": {"code": "def binary_search(arr, target): ..."},
        },
        {
            "inputs": {"description": "Write a function to reverse a linked list"},
            "outputs": {"code": "class ListNode: ..."},
        },
    ]
    for ex in examples:
        client.create_example(
            inputs=ex["inputs"],
            outputs=ex["outputs"],
            dataset_id=str(dataset.id),
        )
    return str(dataset.id)
```

---

## 7. Docker Compose

```yaml
# docker-compose.yml
version: "3.9"

services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_USER: agent
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: agentdb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U agent -d agentdb"]
      interval: 10s
      timeout: 5s
      retries: 5

  qdrant:
    image: qdrant/qdrant:v1.12.0
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - qdrant_data:/qdrant/storage
    environment:
      QDRANT__SERVICE__HTTP_PORT: 6333

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

  agent-api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - POSTGRES_URL=postgresql+psycopg://agent:secret@postgres:5432/agentdb
      - QDRANT_URL=http://qdrant:6333
      - REDIS_URL=redis://redis:6379
      - LANGSMITH_TRACING=true
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
      qdrant:
        condition: service_started
      redis:
        condition: service_started
    volumes:
      - ./workspace:/workspace

volumes:
  postgres_data:
  qdrant_data:
  redis_data:
```

---

## 8. Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# System deps
RUN apt-get update && apt-get install -y \
    build-essential git curl \
    && rm -rf /var/lib/apt/lists/*

# Install dependencies
COPY pyproject.toml .
RUN pip install --no-cache-dir -e .

# Copy source
COPY src/ ./src/
COPY .env.example .env

EXPOSE 8000
CMD ["python", "-m", "uvicorn", "research_code_agent.app:app", \
     "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

---

## 9. langgraph.json for LangGraph Platform

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/research_code_agent/graph.py:build_agent_graph",
    "researcher": "./src/research_code_agent/agents/researcher.py:researcher_graph",
    "coder": "./src/research_code_agent/agents/coder.py:coder_graph"
  },
  "env": ".env",
  "python_version": "3.12",
  "dockerfile_lines": [
    "RUN apt-get update && apt-get install -y build-essential"
  ]
}
```

---

## 10. Complete Run Instructions

```bash
# 1. Clone and install
git clone https://github.com/your-org/research-code-agent
cd research-code-agent
pip install -e ".[dev]"

# 2. Configure environment
cp .env.example .env
# Edit .env: add OPENAI_API_KEY, LANGSMITH_API_KEY, TAVILY_API_KEY

# 3. Start infrastructure
docker-compose up -d postgres qdrant redis
# Wait for postgres health check to pass

# 4. Local dev with LangGraph Platform
langgraph dev --port 2024
# Visit http://localhost:2024 for the LangGraph Studio UI

# 5. OR run as FastAPI
python -m uvicorn research_code_agent.app:app --reload --port 8000

# 6. Send a feature request
curl -X POST http://localhost:8000/feature/stream \
  -H "Content-Type: application/json" \
  -d '{"description": "Implement a rate limiter using a token bucket algorithm"}' \
  --no-buffer

# 7. Run evaluation
python -c "
from research_code_agent.observability import create_eval_dataset, setup_tracing
setup_tracing()
dataset_id = create_eval_dataset()
print(f'Dataset: {dataset_id}')
"

# 8. Docker production build
docker-compose up --build agent-api

# 9. LangGraph Platform deployment
langgraph build -t research-code-agent:1.0.0
langgraph deploy --project-id your-project-id
```

---

## 11. Integration Test

```python
# tests/test_integration.py
import asyncio
import pytest
from research_code_agent.graph import build_agent_graph
from langgraph.checkpoint.memory import InMemorySaver


@pytest.mark.asyncio
async def test_feature_request_end_to_end():
    graph = await build_agent_graph(checkpointer=InMemorySaver())
    config = {"configurable": {"thread_id": "test-1"}}

    result = await graph.ainvoke(
        {
            "messages": [{"role": "user", "content": "Write a Python function that checks if a string is a palindrome"}],
            "language": "python",
            "research_docs": [],
            "plan": "",
            "code": "",
            "test_output": "",
            "review": "",
            "iteration": 0,
            "approved": False,
        },
        config=config,
    )

    assert result["code"], "Should generate code"
    assert result["plan"], "Should generate a plan"
    assert "def " in result["code"] or "class " in result["code"], "Code should contain a function"


@pytest.mark.asyncio
async def test_streaming():
    graph = await build_agent_graph(checkpointer=InMemorySaver())
    config = {"configurable": {"thread_id": "test-stream-1"}}
    tokens = []

    async for event in graph.astream_events(
        {
            "messages": [{"role": "user", "content": "Write hello world in Python"}],
            "language": "python",
            "research_docs": [],
            "plan": "",
            "code": "",
            "test_output": "",
            "review": "",
            "iteration": 0,
            "approved": False,
        },
        config=config,
        version="v2",
    ):
        if event["event"] == "on_chat_model_stream":
            token = event["data"]["chunk"].content
            if token:
                tokens.append(token)

    assert len(tokens) > 0, "Should stream tokens"
```
