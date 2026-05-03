# LLM Agent System — Python / LangChain / LangGraph / LangSmith
## Master Index

This collection provides production-grade implementation guidelines for building modern LLM
agent systems. All examples target Python 3.12+, LangGraph 0.2+, LangChain 0.3, and
LangSmith with full type hints and async support.

---

## Document Map

| # | File | Topic |
|---|------|-------|
| 01 | [01-architecture.md](01-architecture.md) | Package layout, layer diagram, deployment options |
| 02 | [02-core-agent-loop.md](02-core-agent-loop.md) | StateGraph, ReAct, Reflexion, SELF-REFINE, streaming |
| 03 | [03-tool-system.md](03-tool-system.md) | @tool, StructuredTool, ToolNode, MCP, parallel calls |
| 04 | [04-memory-and-rag.md](04-memory-and-rag.md) | Checkpointer, Store, MemGPT, Modular RAG, CRAG, HyDE |
| 05 | [05-knowledge-graph.md](05-knowledge-graph.md) | Entities, Neo4j, GraphRAG, HippoRAG, LightRAG |
| 06 | [06-event-driven.md](06-event-driven.md) | Event bus, CQRS, SSE, WebSocket, Redis Streams |
| 07 | [07-multiagent.md](07-multiagent.md) | Supervisor, handoff, swarm, CrewAI, AutoGen |
| 08 | [08-self-improvement.md](08-self-improvement.md) | Reflexion, ExpeL, CRITIC, SELF-REFINE, skill library |
| 09 | [09-session-persistence.md](09-session-persistence.md) | Thread mgmt, PostgreSQL, forking, compaction |
| 10 | [10-prompts.md](10-prompts.md) | Templates, ReAct/Reflexion/HyDE prompts, Hub versioning |
| 11 | [11-complete-example.md](11-complete-example.md) | Full research + code-gen agent, Docker Compose |
| 12 | [12-langsmith-observability.md](12-langsmith-observability.md) | Tracing, evaluation, datasets, LLM-as-judge |
| 13 | [13-testing.md](13-testing.md) | Unit tests (tool schema, handler, state machine), integration (real API), eval harness (LLM-as-judge), CI config |

---

## Quick Start

### 1. Install Core Dependencies

```bash
# Core framework
pip install langchain==0.3.* langgraph==0.2.* langsmith==0.2.*

# LLM providers
pip install langchain-openai langchain-anthropic langchain-google-genai

# LangGraph extras
pip install langgraph-supervisor langgraph-checkpoint-postgres langgraph-checkpoint-sqlite

# Vector stores
pip install langchain-community faiss-cpu qdrant-client langchain-qdrant
pip install psycopg[binary] langchain-postgres pgvector

# RAG + retrieval
pip install rank-bm25 flashrank sentence-transformers
pip install FlagEmbedding  # BGE-M3

# Knowledge graph
pip install networkx neo4j langchain-neo4j lightrag-hku

# Memory
pip install mem0ai

# MCP tools
pip install langchain-mcp-adapters mcp

# Web search
pip install tavily-python langchain-tavily

# API layer
pip install fastapi uvicorn[standard] sse-starlette

# Observability
pip install opentelemetry-sdk opentelemetry-exporter-otlp

# Dev tools
pip install pydantic-settings python-dotenv ruff mypy pytest pytest-asyncio
```

### 2. Environment Variables

Create `.env` in your project root:

```bash
# LangSmith (tracing)
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=lsv2_pt_...
LANGSMITH_PROJECT=my-agent
LANGSMITH_ENDPOINT=https://api.smith.langchain.com

# LLM providers
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_API_KEY=...

# Vector stores
QDRANT_URL=http://localhost:6333
QDRANT_API_KEY=                     # leave empty for local

POSTGRES_URL=postgresql+psycopg://user:pass@localhost:5432/agentdb

# Redis
REDIS_URL=redis://localhost:6379

# Tools
TAVILY_API_KEY=tvly-...

# Neo4j (optional)
NEO4J_URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=secret
```

### 3. Minimal Working Agent (30 lines)

```python
# quickstart.py
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.memory import InMemorySaver

load_dotenv()

# Define a tool
@tool
def get_weather(city: str) -> str:
    """Return current weather for a city (stub)."""
    return f"It is sunny and 22°C in {city}."

# Build the agent
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
checkpointer = InMemorySaver()

agent = create_react_agent(
    model=llm,
    tools=[get_weather],
    checkpointer=checkpointer,
)

# Run with a thread
config = {"configurable": {"thread_id": "session-1"}}
result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in Paris?"}]},
    config=config,
)
print(result["messages"][-1].content)
```

### 4. Async Streaming Agent

```python
# streaming_quickstart.py
import asyncio
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.memory import InMemorySaver

async def main() -> None:
    llm = ChatOpenAI(model="gpt-4o", streaming=True)
    agent = create_react_agent(llm, tools=[], checkpointer=InMemorySaver())
    config = {"configurable": {"thread_id": "async-1"}}

    async for event in agent.astream_events(
        {"messages": [{"role": "user", "content": "Explain ReAct in 3 sentences."}]},
        config=config,
        version="v2",
    ):
        if event["event"] == "on_chat_model_stream":
            chunk = event["data"]["chunk"]
            print(chunk.content, end="", flush=True)
    print()

asyncio.run(main())
```

### 5. LangGraph Platform Quick Start

```bash
# Install CLI
pip install langgraph-cli

# Create project scaffold
langgraph new my-agent --template react-agent-python

# Local development server (hot reload)
cd my-agent
langgraph dev

# Build Docker image
langgraph build -t my-agent:latest

# Self-hosted deployment
langgraph up
```

---

## Architecture Decision at a Glance

```
Simple Q&A / one-shot        →  ChatOpenAI + StrOutputParser
Multi-step with tools        →  create_react_agent (prebuilt)
Custom control flow          →  StateGraph (manual graph)
Long sessions + memory       →  StateGraph + checkpointer + Store
Multi-agent coordination     →  create_supervisor + handoff tools
Production deployment        →  LangGraph Platform (Cloud or Self-Hosted)
```

---

## Key Version Matrix

| Package | Min Version | Notes |
|---------|-------------|-------|
| Python | 3.12 | Required for modern type hints |
| langchain | 0.3.0 | Stable LCEL API |
| langgraph | 0.2.0 | StateGraph v2, interrupt(), Store |
| langsmith | 0.2.0 | evaluate() v2, astream_events |
| langchain-openai | 0.2.0 | ChatOpenAI tool_choice |
| pydantic | 2.x | Required by LangChain 0.3 |
| fastapi | 0.115+ | Lifespan, async generator SSE |

---

## Reading Order

**For a new engineer:** 01 → 02 → 03 → 04 → 11 (complete example)

**For RAG/memory work:** 04 → 05 → 09

**For production deployment:** 01 → 06 → 09 → 12 → 11

**For multi-agent systems:** 02 → 03 → 07 → 08
