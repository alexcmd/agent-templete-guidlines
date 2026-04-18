# 09 — Session and Persistence

## 1. LangGraph Thread Management

Every graph invocation is associated with a `thread_id`. The checkpointer persists state per thread.

```python
# src/my_agent/session/thread_manager.py
from __future__ import annotations

import uuid
from typing import Any

from langgraph.types import StateSnapshot


class ThreadManager:
    """Manages LangGraph thread lifecycle."""

    def __init__(self, graph: Any) -> None:
        self._graph = graph

    def new_thread(self, user_id: str | None = None) -> str:
        """Generate a new unique thread ID."""
        prefix = f"user-{user_id}-" if user_id else ""
        return f"{prefix}{uuid.uuid4()}"

    def build_config(
        self,
        thread_id: str,
        extra: dict | None = None,
    ) -> dict[str, Any]:
        config: dict[str, Any] = {"configurable": {"thread_id": thread_id}}
        if extra:
            config["configurable"].update(extra)
        return config

    async def get_state(self, thread_id: str) -> StateSnapshot | None:
        config = self.build_config(thread_id)
        return await self._graph.aget_state(config)

    async def list_threads(self) -> list[str]:
        """List all known thread IDs (requires PostgreSQL checkpointer)."""
        # With AsyncPostgresSaver you can query the database directly
        raise NotImplementedError("Implement with direct DB query")

    async def delete_thread(self, thread_id: str) -> None:
        """Delete all checkpoints for a thread."""
        # Direct DB operation — checkpointer does not expose this directly
        raise NotImplementedError("Use direct DB DELETE WHERE thread_id=...")
```

---

## 2. AsyncPostgresSaver for Production

```python
# src/my_agent/session/postgres_checkpointer.py
from __future__ import annotations

from contextlib import asynccontextmanager
from typing import AsyncGenerator

from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

from my_agent.config import settings


@asynccontextmanager
async def postgres_checkpointer() -> AsyncGenerator[AsyncPostgresSaver, None]:
    """Async context manager for PostgreSQL checkpointer."""
    async with await AsyncPostgresSaver.from_conn_string(
        settings.postgres_url,
        # Connection pool settings
        pool_config={
            "min_size": 2,
            "max_size": 20,
            "max_queries": 50_000,
        },
    ) as checkpointer:
        await checkpointer.setup()   # creates tables: checkpoints, checkpoint_blobs, checkpoint_writes
        yield checkpointer


# Usage in app startup:
# async with postgres_checkpointer() as cp:
#     graph = builder.compile(checkpointer=cp)
#     app.state.graph = graph
```

### Required PostgreSQL Schema (auto-created by setup())

```sql
-- Created automatically by checkpointer.setup()
-- checkpoints: stores graph state snapshots
-- checkpoint_blobs: stores large binary objects
-- checkpoint_writes: stores pending writes before commit

-- To view threads:
SELECT DISTINCT thread_id, MAX(checkpoint_id) as latest
FROM checkpoints
GROUP BY thread_id
ORDER BY latest DESC;

-- To view state history for a thread:
SELECT checkpoint_id, ts, metadata
FROM checkpoints
WHERE thread_id = 'my-thread-id'
ORDER BY ts DESC;
```

---

## 3. State Inspection and History

```python
# src/my_agent/session/state_inspection.py
from __future__ import annotations

from typing import Any

from langgraph.types import StateSnapshot


async def inspect_thread(graph: Any, thread_id: str) -> dict[str, Any]:
    """Get current state, next nodes, and metadata for a thread."""
    config = {"configurable": {"thread_id": thread_id}}
    state: StateSnapshot = await graph.aget_state(config)

    if state is None:
        return {"error": "Thread not found"}

    return {
        "thread_id": thread_id,
        "current_values": state.values,
        "next_nodes": state.next,
        "config": state.config,
        "metadata": state.metadata,
        "created_at": str(state.created_at) if hasattr(state, "created_at") else None,
        "checkpoint_id": state.config.get("configurable", {}).get("checkpoint_id"),
    }


async def get_state_history(
    graph: Any,
    thread_id: str,
    limit: int = 10,
) -> list[dict[str, Any]]:
    """Get the full history of state snapshots for a thread."""
    config = {"configurable": {"thread_id": thread_id}}
    history = []
    async for snapshot in graph.aget_state_history(config, limit=limit):
        history.append({
            "checkpoint_id": snapshot.config.get("configurable", {}).get("checkpoint_id"),
            "next": snapshot.next,
            "message_count": len(snapshot.values.get("messages", [])),
            "metadata": snapshot.metadata,
        })
    return history
```

---

## 4. Manual State Update with graph.update_state()

```python
# src/my_agent/session/state_management.py
from __future__ import annotations

from typing import Any

from langchain_core.messages import HumanMessage


async def inject_system_message(
    graph: Any,
    thread_id: str,
    content: str,
) -> None:
    """Inject a message into thread state without running the graph."""
    config = {"configurable": {"thread_id": thread_id}}
    await graph.aupdate_state(
        config,
        values={"messages": [HumanMessage(content=content)]},
        # as_node specifies which node's perspective to use for reducers
        as_node="__start__",
    )


async def correct_last_message(
    graph: Any,
    thread_id: str,
    correction: str,
) -> None:
    """Replace the last AI message with a corrected version."""
    from langchain_core.messages import AIMessage

    config = {"configurable": {"thread_id": thread_id}}
    state = await graph.aget_state(config)
    if state is None:
        return

    messages = state.values.get("messages", [])
    # Remove last AI message and replace
    corrected_messages = [m for m in messages if not (isinstance(m, AIMessage) and m == messages[-1])]
    corrected_messages.append(AIMessage(content=correction))

    await graph.aupdate_state(
        config,
        values={"messages": corrected_messages},
    )


async def reset_to_checkpoint(
    graph: Any,
    thread_id: str,
    checkpoint_id: str,
) -> Any:
    """Roll back to a specific checkpoint and resume from there."""
    config = {
        "configurable": {
            "thread_id": thread_id,
            "checkpoint_id": checkpoint_id,
        }
    }
    state = await graph.aget_state(config)
    return state
```

---

## 5. Cross-Thread Store — Namespace-Based User Memory

```python
# src/my_agent/session/user_memory.py
from __future__ import annotations

from typing import Any

from langgraph.store.base import BaseStore


class UserMemoryManager:
    """Manage per-user memories across threads using LangGraph Store."""

    def __init__(self, store: BaseStore) -> None:
        self._store = store

    def _ns(self, user_id: str, category: str) -> tuple[str, ...]:
        return ("users", user_id, category)

    async def remember_fact(self, user_id: str, key: str, value: Any) -> None:
        ns = self._ns(user_id, "facts")
        await self._store.aput(ns, key, {"value": value})

    async def recall_facts(self, user_id: str, query: str, limit: int = 10) -> list[dict]:
        ns = self._ns(user_id, "facts")
        results = await self._store.asearch(ns, query=query, limit=limit)
        return [{"key": item.key, "value": item.value["value"]} for item in results]

    async def update_preference(self, user_id: str, pref: str, value: Any) -> None:
        ns = self._ns(user_id, "preferences")
        await self._store.aput(ns, pref, {"value": value, "updated_at": str(__import__("datetime").datetime.utcnow())})

    async def get_preferences(self, user_id: str) -> dict[str, Any]:
        ns = self._ns(user_id, "preferences")
        results = await self._store.asearch(ns, query="", limit=100)
        return {item.key: item.value["value"] for item in results}

    async def save_conversation_summary(self, user_id: str, thread_id: str, summary: str) -> None:
        ns = self._ns(user_id, "summaries")
        await self._store.aput(ns, thread_id, {
            "summary": summary,
            "thread_id": thread_id,
            "created_at": str(__import__("datetime").datetime.utcnow()),
        })

    async def get_recent_summaries(self, user_id: str, limit: int = 5) -> list[str]:
        ns = self._ns(user_id, "summaries")
        results = await self._store.asearch(ns, query="recent conversation", limit=limit)
        return [item.value["summary"] for item in results]
```

---

## 6. Session Forking — Copy State Pattern

```python
# src/my_agent/session/forking.py
from __future__ import annotations

import uuid
from typing import Any

from langgraph.types import StateSnapshot


async def fork_thread(
    graph: Any,
    source_thread_id: str,
    checkpoint_id: str | None = None,
) -> str:
    """
    Fork a thread: create a new thread with a copy of the source state.
    Useful for A/B testing different continuation strategies.
    """
    # Get source state
    config = {"configurable": {
        "thread_id": source_thread_id,
        **({"checkpoint_id": checkpoint_id} if checkpoint_id else {}),
    }}
    source_state: StateSnapshot = await graph.aget_state(config)

    if source_state is None:
        raise ValueError(f"Thread {source_thread_id} not found")

    # Create new thread with copied state
    new_thread_id = f"fork-{source_thread_id[:8]}-{uuid.uuid4()}"
    new_config = {"configurable": {"thread_id": new_thread_id}}

    await graph.aupdate_state(
        new_config,
        values=source_state.values,
        as_node="__start__",
    )

    return new_thread_id


async def branch_experiment(
    graph: Any,
    thread_id: str,
    variants: list[dict],
) -> list[str]:
    """
    Create N forks of a thread, each with a different continuation.
    Returns list of new thread IDs.
    """
    fork_ids = []
    for variant in variants:
        new_id = await fork_thread(graph, thread_id)
        # Apply variant-specific state updates
        if variant:
            new_config = {"configurable": {"thread_id": new_id}}
            await graph.aupdate_state(new_config, values=variant)
        fork_ids.append(new_id)
    return fork_ids
```

---

## 7. Auto-Compaction as LangGraph Node

```python
# src/my_agent/session/compaction.py
from __future__ import annotations

from typing import Any

from langchain_core.messages import AIMessage, BaseMessage, SystemMessage
from langchain_openai import ChatOpenAI


TOKEN_LIMIT = 80_000
COMPACTION_TARGET = 20_000   # tokens to target after compaction


def estimate_tokens(messages: list[BaseMessage]) -> int:
    """Rough estimate: 1 token ≈ 4 characters."""
    return sum(len(getattr(m, "content", "")) // 4 for m in messages)


async def maybe_compact_node(state: Any) -> dict[str, Any]:
    """
    Node that compacts message history if it exceeds TOKEN_LIMIT.
    Should be inserted at the top of the agent loop.
    """
    messages = state.get("messages", [])
    token_count = estimate_tokens(messages)

    if token_count <= TOKEN_LIMIT:
        return {}   # No compaction needed

    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

    # Keep the last N recent messages fresh
    recent_count = 5
    old_messages = messages[:-recent_count]
    recent_messages = messages[-recent_count:]

    # Summarize old messages
    history_text = "\n".join(
        f"{m.type.upper()}: {getattr(m, 'content', '')[:500]}"
        for m in old_messages
    )
    summary_prompt = (
        "You are compacting a conversation history. "
        "Preserve all key facts, decisions, code snippets, and action items.\n\n"
        f"History to summarize:\n{history_text}"
    )
    summary = await llm.ainvoke([SystemMessage(content=summary_prompt)])

    compacted_messages = [
        SystemMessage(content=f"[CONVERSATION SUMMARY]\n{summary.content}"),
        *recent_messages,
    ]

    print(f"Compacted {len(messages)} → {len(compacted_messages)} messages "
          f"({token_count} → {estimate_tokens(compacted_messages)} tokens)")

    return {"messages": compacted_messages}
```

---

## 8. Conversation Summary Memory

```python
# src/my_agent/session/summary_memory.py
from __future__ import annotations

from langchain.memory import ConversationSummaryBufferMemory
from langchain_openai import ChatOpenAI


def build_summary_memory(max_token_limit: int = 4000) -> ConversationSummaryBufferMemory:
    """
    Summary + buffer memory: keeps recent messages verbatim,
    summarizes older ones when buffer exceeds max_token_limit.
    """
    return ConversationSummaryBufferMemory(
        llm=ChatOpenAI(model="gpt-4o-mini"),
        max_token_limit=max_token_limit,
        return_messages=True,
        memory_key="chat_history",
    )


# LangGraph-native alternative: use the compaction node above
# ConversationSummaryBufferMemory is better suited for pure LCEL chains
```

---

## 9. Configuration with pydantic-settings and .env

```python
# src/my_agent/config.py  (extended)
from __future__ import annotations

from functools import lru_cache
from pydantic import Field, SecretStr, field_validator
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore",
    )

    # === Tracing ===
    langsmith_tracing: bool = False
    langsmith_api_key: SecretStr = Field(default=SecretStr(""))
    langsmith_project: str = "default"

    # === LLM ===
    openai_api_key: SecretStr = Field(default=SecretStr(""))
    anthropic_api_key: SecretStr = Field(default=SecretStr(""))
    default_model: str = "gpt-4o"
    fallback_model: str = "gpt-4o-mini"
    model_temperature: float = 0.0
    model_max_tokens: int = 4096

    # === Vector Stores ===
    qdrant_url: str = "http://localhost:6333"
    qdrant_api_key: SecretStr = Field(default=SecretStr(""))
    qdrant_collection: str = "agent_docs"

    # === Database ===
    postgres_url: str = "postgresql+psycopg://postgres:postgres@localhost:5432/agentdb"
    postgres_pool_min: int = 2
    postgres_pool_max: int = 20

    # === Redis ===
    redis_url: str = "redis://localhost:6379"
    redis_ttl_seconds: int = 86_400  # 24 hours

    # === Session ===
    max_session_tokens: int = 100_000
    compaction_target_tokens: int = 20_000
    max_iterations: int = 10
    recursion_limit: int = 50

    # === Tools ===
    tavily_api_key: SecretStr = Field(default=SecretStr(""))
    bash_allowed_prefixes: list[str] = Field(
        default=["ls", "cat", "echo", "python", "pytest", "git", "pip"]
    )

    @field_validator("postgres_url")
    @classmethod
    def validate_postgres_url(cls, v: str) -> str:
        if not v.startswith("postgresql"):
            raise ValueError("postgres_url must start with postgresql://")
        return v


@lru_cache(maxsize=1)
def get_settings() -> Settings:
    return Settings()


settings = get_settings()
```

### .env Template

```bash
# === LangSmith ===
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=lsv2_pt_your_key_here
LANGSMITH_PROJECT=my-agent-prod

# === LLM ===
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
DEFAULT_MODEL=gpt-4o
FALLBACK_MODEL=gpt-4o-mini

# === Databases ===
POSTGRES_URL=postgresql+psycopg://agent:secret@localhost:5432/agentdb
REDIS_URL=redis://localhost:6379

# === Vector Stores ===
QDRANT_URL=http://localhost:6333

# === Tools ===
TAVILY_API_KEY=tvly-...

# === Session ===
MAX_SESSION_TOKENS=100000
MAX_ITERATIONS=10
```
