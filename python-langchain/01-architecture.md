# 01 — System Architecture

## 1. Package Structure (src layout)

```
my-agent/
├── pyproject.toml
├── .env
├── langgraph.json                  # LangGraph Platform config
├── Dockerfile
├── docker-compose.yml
└── src/
    └── my_agent/
        ├── __init__.py
        ├── app.py                  # FastAPI entry point
        ├── graph.py                # Top-level StateGraph assembly
        ├── state.py                # TypedDict state definitions
        ├── config.py               # pydantic-settings configuration
        │
        ├── agents/
        │   ├── __init__.py
        │   ├── researcher.py       # Research sub-agent
        │   ├── coder.py            # Code generation sub-agent
        │   └── reviewer.py         # Review sub-agent
        │
        ├── tools/
        │   ├── __init__.py
        │   ├── bash.py
        │   ├── file_io.py
        │   ├── web_search.py
        │   └── code_interpreter.py
        │
        ├── memory/
        │   ├── __init__.py
        │   ├── checkpointer.py     # Postgres checkpointer factory
        │   ├── store.py            # Cross-thread Store factory
        │   └── memgpt.py           # MemGPT 3-tier memory tools
        │
        ├── rag/
        │   ├── __init__.py
        │   ├── pipeline.py         # Modular RAG LCEL chain
        │   ├── retrievers.py       # MultiQuery, Compression, Parent
        │   ├── reranker.py         # FlashRank / CrossEncoder
        │   └── vector_stores.py    # FAISS / Qdrant / pgvector factories
        │
        ├── kg/
        │   ├── __init__.py
        │   ├── models.py           # Entity / Relation Pydantic models
        │   ├── extraction.py       # Triple extraction chain
        │   ├── graph_store.py      # networkx + Neo4j backends
        │   └── lightrag_client.py  # LightRAG wrapper
        │
        ├── prompts/
        │   ├── __init__.py
        │   ├── system.py           # SystemPromptBuilder
        │   └── templates.py        # All prompt templates
        │
        └── observability/
            ├── __init__.py
            └── langsmith.py        # Evaluation helpers
```

### pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-agent"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    # Framework
    "langchain>=0.3.0,<0.4",
    "langgraph>=0.2.0,<0.3",
    "langsmith>=0.2.0,<0.3",
    "langgraph-supervisor>=0.0.10",
    "langgraph-checkpoint-postgres>=2.0.0",
    "langgraph-checkpoint-sqlite>=2.0.0",

    # LLM providers
    "langchain-openai>=0.2.0",
    "langchain-anthropic>=0.2.0",
    "langchain-google-genai>=2.0.0",

    # Vector stores
    "langchain-community>=0.3.0",
    "faiss-cpu>=1.8.0",
    "qdrant-client>=1.12.0",
    "langchain-qdrant>=0.2.0",
    "psycopg[binary]>=3.2.0",
    "langchain-postgres>=0.0.12",
    "pgvector>=0.3.0",

    # RAG
    "rank-bm25>=0.2.2",
    "flashrank>=0.2.9",
    "sentence-transformers>=3.0.0",
    "FlagEmbedding>=1.3.0",

    # KG
    "networkx>=3.3",
    "neo4j>=5.24.0",
    "langchain-neo4j>=0.1.0",
    "lightrag-hku>=1.3.0",

    # Memory
    "mem0ai>=0.1.0",

    # MCP
    "langchain-mcp-adapters>=0.1.0",
    "mcp>=1.0.0",

    # Web tools
    "tavily-python>=0.5.0",
    "langchain-tavily>=0.1.0",

    # API
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.32.0",
    "sse-starlette>=2.1.0",
    "websockets>=13.0",

    # Config
    "pydantic>=2.9.0",
    "pydantic-settings>=2.5.0",
    "python-dotenv>=1.0.0",

    # Observability
    "opentelemetry-sdk>=1.28.0",
    "opentelemetry-exporter-otlp>=1.28.0",

    # Async
    "aioredis>=2.0.0",
    "asyncpg>=0.30.0",
]

[project.optional-dependencies]
dev = [
    "ruff>=0.7.0",
    "mypy>=1.12.0",
    "pytest>=8.3.0",
    "pytest-asyncio>=0.24.0",
    "httpx>=0.27.0",
    "ipykernel>=6.29.0",
]

[tool.hatch.build.targets.wheel]
packages = ["src/my_agent"]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.mypy]
python_version = "3.12"
strict = true

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

---

## 2. Layer Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                        UI / API Layer                            │
│  FastAPI  ──  SSE Stream  ──  WebSocket  ──  REST endpoints      │
└─────────────────────────────┬────────────────────────────────────┘
                              │ invoke / astream_events
┌─────────────────────────────▼────────────────────────────────────┐
│                   LangGraph StateGraph                           │
│  supervisor_node → researcher_node → coder_node → reviewer_node │
│  interrupt() gates ── conditional edges ── stream_mode          │
└──────────┬──────────────────┬──────────────────┬────────────────┘
           │                  │                  │
     ┌─────▼──────┐    ┌──────▼──────┐    ┌─────▼──────┐
     │ Tool System│    │   Memory    │    │  KG Layer  │
     │ @tool      │    │ Checkpointer│    │ networkx   │
     │ ToolNode   │    │ Store       │    │ Neo4j      │
     │ MCP        │    │ MemGPT      │    │ LightRAG   │
     └─────┬──────┘    └──────┬──────┘    └─────┬──────┘
           │                  │                  │
┌──────────▼──────────────────▼──────────────────▼────────────────┐
│                    RAG / Retrieval Layer                         │
│  ModularRAG  ──  HyDE  ──  CRAG  ──  RAG-Fusion  ──  Reranker  │
│  FAISS  ──  Qdrant (hybrid)  ──  pgvector                       │
└─────────────────────────────┬────────────────────────────────────┘
                              │
┌─────────────────────────────▼────────────────────────────────────┐
│                   Observability (LangSmith)                      │
│  Auto-tracing  ──  @traceable  ──  evaluate()  ──  Datasets      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. LangGraph vs Pure LangChain Decision Guide

| Scenario | Recommendation |
|----------|----------------|
| Single LLM call, no state | `ChatOpenAI` + LCEL pipe |
| Chain of prompts, no branching | LCEL `RunnableSequence` |
| Agent with tools, simple loop | `create_react_agent` (prebuilt) |
| Custom branching / retry logic | `StateGraph` with conditional edges |
| Human approval in the loop | `StateGraph` + `interrupt()` |
| Multi-agent coordination | `StateGraph` + `create_supervisor` |
| Long sessions, resumable | `StateGraph` + PostgreSQL checkpointer |
| Cross-user shared memory | `StateGraph` + `Store` |
| Streaming token-by-token | `astream_events(version="v2")` |
| Scheduled / background jobs | LangGraph Platform Cron |

**Rule of thumb:** Use `StateGraph` when you need checkpointing, branching, or human-in-the-loop. Use LCEL for pure pipeline transforms within a node.

---

## 4. Environment Configuration

```python
# src/my_agent/config.py
from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore",
    )

    # LangSmith
    langsmith_tracing: bool = False
    langsmith_api_key: SecretStr = Field(default=SecretStr(""))
    langsmith_project: str = "default"

    # LLM
    openai_api_key: SecretStr = Field(default=SecretStr(""))
    anthropic_api_key: SecretStr = Field(default=SecretStr(""))
    default_model: str = "gpt-4o"
    fallback_model: str = "gpt-4o-mini"

    # Vector stores
    qdrant_url: str = "http://localhost:6333"
    qdrant_api_key: SecretStr = Field(default=SecretStr(""))
    postgres_url: str = "postgresql+psycopg://postgres:postgres@localhost:5432/agentdb"

    # Redis
    redis_url: str = "redis://localhost:6379"

    # Tools
    tavily_api_key: SecretStr = Field(default=SecretStr(""))

    # Neo4j
    neo4j_uri: str = "bolt://localhost:7687"
    neo4j_username: str = "neo4j"
    neo4j_password: SecretStr = Field(default=SecretStr("secret"))

    # Agent
    max_iterations: int = 10
    recursion_limit: int = 50
    token_budget: int = 100_000


settings = Settings()
```

---

## 5. LangGraph Platform Deployment Options

| Option | Cost | Scale | Control | Best For |
|--------|------|-------|---------|----------|
| **Cloud SaaS** | Pay-per-use | Auto | Low | Rapid prototyping, managed infra |
| **Self-Hosted Lite** | Free | 1M nodes/month | High | Internal tools, cost-sensitive |
| **Self-Hosted Enterprise** | License | Unlimited | Full | Regulated industries |
| **BYOC (AWS)** | AWS costs | Auto | Medium | AWS-native, compliance |

### langgraph.json

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/my_agent/graph.py:graph"
  },
  "env": ".env",
  "python_version": "3.12",
  "dockerfile_lines": [
    "RUN apt-get update && apt-get install -y build-essential"
  ]
}
```

### Deployment Commands

```bash
# Local dev (hot reload on file change)
langgraph dev --port 2024

# Production Docker image
langgraph build -t my-agent:1.0.0

# Self-hosted (requires Docker + Postgres + Redis)
langgraph up --port 8123

# Cloud deploy
langgraph deploy --project-id proj_abc123
```

---

## 6. FastAPI Integration

```python
# src/my_agent/app.py
from __future__ import annotations

import json
import uuid
from collections.abc import AsyncGenerator
from contextlib import asynccontextmanager
from typing import Any

import uvicorn
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
from sse_starlette.sse import EventSourceResponse

from my_agent.config import settings
from my_agent.graph import build_graph


# --------------------------------------------------------------------------- #
#  Startup / shutdown                                                           #
# --------------------------------------------------------------------------- #
@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    app.state.graph = await build_graph()
    yield
    # teardown if needed


app = FastAPI(title="Agent API", version="1.0.0", lifespan=lifespan)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)


# --------------------------------------------------------------------------- #
#  Request / response models                                                    #
# --------------------------------------------------------------------------- #
class ChatRequest(BaseModel):
    message: str
    thread_id: str | None = None
    stream: bool = True


class ChatResponse(BaseModel):
    thread_id: str
    content: str


# --------------------------------------------------------------------------- #
#  SSE streaming endpoint                                                       #
# --------------------------------------------------------------------------- #
@app.post("/chat/stream")
async def chat_stream(req: ChatRequest) -> EventSourceResponse:
    thread_id = req.thread_id or str(uuid.uuid4())
    config: dict[str, Any] = {"configurable": {"thread_id": thread_id}}

    async def event_generator() -> AsyncGenerator[dict[str, str], None]:
        graph = app.state.graph
        async for event in graph.astream_events(
            {"messages": [{"role": "user", "content": req.message}]},
            config=config,
            version="v2",
        ):
            kind = event["event"]
            if kind == "on_chat_model_stream":
                token = event["data"]["chunk"].content
                if token:
                    yield {"event": "token", "data": token}
            elif kind == "on_tool_start":
                yield {
                    "event": "tool_start",
                    "data": json.dumps({"tool": event["name"]}),
                }
            elif kind == "on_tool_end":
                yield {"event": "tool_end", "data": json.dumps({"tool": event["name"]})}
        yield {"event": "done", "data": json.dumps({"thread_id": thread_id})}

    return EventSourceResponse(event_generator())


# --------------------------------------------------------------------------- #
#  Synchronous endpoint                                                         #
# --------------------------------------------------------------------------- #
@app.post("/chat", response_model=ChatResponse)
async def chat(req: ChatRequest) -> ChatResponse:
    thread_id = req.thread_id or str(uuid.uuid4())
    config: dict[str, Any] = {"configurable": {"thread_id": thread_id}}
    graph = app.state.graph
    result = await graph.ainvoke(
        {"messages": [{"role": "user", "content": req.message}]},
        config=config,
    )
    last_message = result["messages"][-1]
    return ChatResponse(thread_id=thread_id, content=last_message.content)


# --------------------------------------------------------------------------- #
#  State inspection endpoint                                                    #
# --------------------------------------------------------------------------- #
@app.get("/threads/{thread_id}/state")
async def get_thread_state(thread_id: str) -> dict[str, Any]:
    config: dict[str, Any] = {"configurable": {"thread_id": thread_id}}
    state = await app.state.graph.aget_state(config)
    if state is None:
        raise HTTPException(status_code=404, detail="Thread not found")
    return {"values": state.values, "next": state.next}


if __name__ == "__main__":
    uvicorn.run("my_agent.app:app", host="0.0.0.0", port=8000, reload=True)
```

---

## 7. Key Architectural Principles

1. **State is the single source of truth.** All agent data flows through the LangGraph `State` TypedDict. Avoid side-channel state in global variables.

2. **Nodes are pure functions.** Each node takes `State` and returns a partial `State` dict. Side effects (DB writes, API calls) belong in tools, not nodes directly.

3. **Checkpoints at every edge.** Use a real checkpointer (SQLite in dev, PostgreSQL in prod) so any step can be replayed after a crash.

4. **Async all the way.** Define async nodes (`async def`) and use `ainvoke` / `astream` / `astream_events` at the API layer for non-blocking IO.

5. **Observability from day one.** Set `LANGSMITH_TRACING=true` and name every node, tool, and chain. Debugging without traces costs orders of magnitude more time than adding them upfront.

6. **Fail fast with typed state.** Use `Annotated` reducers so LangGraph validates state merges at runtime.
