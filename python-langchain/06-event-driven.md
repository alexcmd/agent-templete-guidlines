# 06 — Event-Driven Architecture

## 1. Pydantic EventType Enum and Event Model

```python
# src/my_agent/events/models.py
from __future__ import annotations

import uuid
from datetime import datetime
from enum import StrEnum
from typing import Any

from pydantic import BaseModel, Field


class EventType(StrEnum):
    # Agent lifecycle
    AGENT_STARTED = "agent.started"
    AGENT_COMPLETED = "agent.completed"
    AGENT_FAILED = "agent.failed"

    # Node execution
    NODE_STARTED = "node.started"
    NODE_COMPLETED = "node.completed"
    NODE_FAILED = "node.failed"

    # LLM
    LLM_START = "llm.start"
    LLM_TOKEN = "llm.token"
    LLM_END = "llm.end"

    # Tools
    TOOL_START = "tool.start"
    TOOL_END = "tool.end"
    TOOL_ERROR = "tool.error"

    # Human interaction
    HUMAN_INTERRUPT = "human.interrupt"
    HUMAN_RESUME = "human.resume"

    # Memory
    MEMORY_READ = "memory.read"
    MEMORY_WRITE = "memory.write"

    # Custom
    PROGRESS = "progress"
    CUSTOM = "custom"


class Event(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    event_type: EventType
    timestamp: datetime = Field(default_factory=datetime.utcnow)
    thread_id: str
    node_name: str | None = None
    run_id: str | None = None
    data: dict[str, Any] = Field(default_factory=dict)
    metadata: dict[str, Any] = Field(default_factory=dict)

    def to_sse(self) -> str:
        """Format as Server-Sent Event string."""
        import json
        return f"event: {self.event_type}\ndata: {json.dumps(self.model_dump(mode='json'))}\n\n"
```

---

## 2. Append-Only JSONL Event Log

```python
# src/my_agent/events/event_log.py
from __future__ import annotations

import asyncio
import json
from datetime import datetime
from pathlib import Path
from typing import AsyncGenerator

import aiofiles

from my_agent.events.models import Event


class JsonlEventLog:
    """Thread-safe append-only event log backed by a JSONL file."""

    def __init__(self, path: str = "events.jsonl") -> None:
        self._path = Path(path)
        self._path.parent.mkdir(parents=True, exist_ok=True)
        self._lock = asyncio.Lock()

    async def append(self, event: Event) -> None:
        async with self._lock:
            async with aiofiles.open(self._path, "a") as f:
                await f.write(event.model_dump_json() + "\n")

    async def replay(
        self,
        thread_id: str | None = None,
        since: datetime | None = None,
    ) -> AsyncGenerator[Event, None]:
        """Read and yield events, optionally filtered."""
        if not self._path.exists():
            return
        async with aiofiles.open(self._path) as f:
            async for line in f:
                line = line.strip()
                if not line:
                    continue
                try:
                    event = Event.model_validate_json(line)
                except Exception:
                    continue
                if thread_id and event.thread_id != thread_id:
                    continue
                if since and event.timestamp < since:
                    continue
                yield event

    async def count(self, thread_id: str | None = None) -> int:
        count = 0
        async for _ in self.replay(thread_id=thread_id):
            count += 1
        return count
```

---

## 3. CQRS: EventStore (writes) + AgentView (read projections)

```python
# src/my_agent/events/cqrs.py
from __future__ import annotations

import asyncio
from collections import defaultdict
from typing import Any

from my_agent.events.models import Event, EventType


class EventStore:
    """Write side: accepts commands, emits events."""

    def __init__(self) -> None:
        self._events: list[Event] = []
        self._subscribers: list[asyncio.Queue] = []

    async def publish(self, event: Event) -> None:
        self._events.append(event)
        for queue in self._subscribers:
            await queue.put(event)

    def subscribe(self) -> asyncio.Queue:
        q: asyncio.Queue = asyncio.Queue()
        self._subscribers.append(q)
        return q

    def unsubscribe(self, queue: asyncio.Queue) -> None:
        self._subscribers.remove(queue)

    def get_events(self, thread_id: str) -> list[Event]:
        return [e for e in self._events if e.thread_id == thread_id]


class AgentView:
    """Read side: projects events into queryable state."""

    def __init__(self, event_store: EventStore) -> None:
        self._store = event_store
        self._state: dict[str, dict[str, Any]] = defaultdict(dict)

    def project(self, thread_id: str) -> dict[str, Any]:
        """Rebuild agent view by replaying all events for a thread."""
        state: dict[str, Any] = {
            "status": "idle",
            "tool_calls": [],
            "token_count": 0,
            "current_node": None,
            "errors": [],
        }
        for event in self._store.get_events(thread_id):
            match event.event_type:
                case EventType.AGENT_STARTED:
                    state["status"] = "running"
                    state["started_at"] = event.timestamp
                case EventType.AGENT_COMPLETED:
                    state["status"] = "completed"
                    state["completed_at"] = event.timestamp
                case EventType.AGENT_FAILED:
                    state["status"] = "failed"
                    state["errors"].append(event.data.get("error", "unknown"))
                case EventType.NODE_STARTED:
                    state["current_node"] = event.node_name
                case EventType.TOOL_START:
                    state["tool_calls"].append(event.data.get("tool_name"))
                case EventType.LLM_TOKEN:
                    state["token_count"] += event.data.get("token_count", 1)
        return state
```

---

## 4. LangGraph Streaming as Event Bus: astream_events v2

```python
# src/my_agent/events/stream_adapter.py
from __future__ import annotations

from typing import Any, AsyncGenerator

from my_agent.events.models import Event, EventType
from my_agent.events.cqrs import EventStore


async def graph_events_to_bus(
    graph: Any,
    inputs: dict,
    config: dict,
    event_store: EventStore,
) -> None:
    """
    Consume astream_events v2 and publish typed Event objects to the EventStore.
    """
    thread_id = config.get("configurable", {}).get("thread_id", "unknown")

    await event_store.publish(Event(
        event_type=EventType.AGENT_STARTED,
        thread_id=thread_id,
        data={"inputs": str(inputs)[:200]},
    ))

    try:
        async for raw_event in graph.astream_events(inputs, config=config, version="v2"):
            kind = raw_event["event"]
            name = raw_event.get("name", "")
            run_id = raw_event.get("run_id", "")

            match kind:
                case "on_chain_start" if raw_event.get("metadata", {}).get("langgraph_node"):
                    await event_store.publish(Event(
                        event_type=EventType.NODE_STARTED,
                        thread_id=thread_id,
                        node_name=raw_event["metadata"]["langgraph_node"],
                        run_id=run_id,
                    ))

                case "on_chat_model_stream":
                    token = raw_event["data"]["chunk"].content
                    if token:
                        await event_store.publish(Event(
                            event_type=EventType.LLM_TOKEN,
                            thread_id=thread_id,
                            data={"token": token, "token_count": 1},
                        ))

                case "on_tool_start":
                    await event_store.publish(Event(
                        event_type=EventType.TOOL_START,
                        thread_id=thread_id,
                        data={"tool_name": name, "inputs": raw_event.get("data", {})},
                    ))

                case "on_tool_end":
                    await event_store.publish(Event(
                        event_type=EventType.TOOL_END,
                        thread_id=thread_id,
                        data={"tool_name": name},
                    ))

                case "on_custom_event":
                    await event_store.publish(Event(
                        event_type=EventType.CUSTOM,
                        thread_id=thread_id,
                        data={"name": name, "payload": raw_event.get("data", {})},
                    ))

        await event_store.publish(Event(
            event_type=EventType.AGENT_COMPLETED,
            thread_id=thread_id,
        ))

    except Exception as e:
        await event_store.publish(Event(
            event_type=EventType.AGENT_FAILED,
            thread_id=thread_id,
            data={"error": str(e)},
        ))
        raise
```

---

## 5. get_stream_writer() for Custom Progress Events

```python
# src/my_agent/agents/nodes_with_events.py
from __future__ import annotations

from typing import Any
from langgraph.config import get_stream_writer


async def research_node(state: Any) -> dict[str, Any]:
    """Research node that emits fine-grained progress via custom events."""
    write = get_stream_writer()

    write({"type": "progress", "step": "search", "message": "Searching knowledge base..."})
    docs = await search_knowledge_base(state["messages"][-1].content)

    write({"type": "progress", "step": "rerank", "message": f"Reranking {len(docs)} docs...", "pct": 50})
    ranked = await rerank_documents(docs)

    write({"type": "progress", "step": "done", "message": "Research complete", "pct": 100,
           "doc_count": len(ranked)})

    return {"scratchpad": {"research_docs": ranked}}
```

---

## 6. FastAPI SSE Endpoint

```python
# src/my_agent/api/sse_endpoint.py
from __future__ import annotations

import asyncio
import json
import uuid
from typing import AsyncGenerator

from fastapi import APIRouter
from fastapi.responses import StreamingResponse
from pydantic import BaseModel

from my_agent.events.models import Event, EventType
from my_agent.events.cqrs import EventStore


router = APIRouter()
_event_store = EventStore()


class RunRequest(BaseModel):
    message: str
    thread_id: str | None = None


@router.post("/run/stream")
async def run_stream(req: RunRequest) -> StreamingResponse:
    thread_id = req.thread_id or str(uuid.uuid4())

    async def event_generator() -> AsyncGenerator[str, None]:
        from my_agent.graph import build_react_graph
        from my_agent.tools import get_tools

        graph = build_react_graph(get_tools())
        config = {"configurable": {"thread_id": thread_id}}

        # Subscribe to events BEFORE starting the graph
        queue = _event_store.subscribe()

        # Run graph in background task
        async def run_graph() -> None:
            from my_agent.events.stream_adapter import graph_events_to_bus
            await graph_events_to_bus(
                graph,
                {"messages": [{"role": "user", "content": req.message}]},
                config,
                _event_store,
            )

        task = asyncio.create_task(run_graph())

        # Stream events from queue
        try:
            while True:
                try:
                    event: Event = await asyncio.wait_for(queue.get(), timeout=30.0)
                    yield event.to_sse()
                    if event.event_type in (EventType.AGENT_COMPLETED, EventType.AGENT_FAILED):
                        break
                except asyncio.TimeoutError:
                    yield "event: heartbeat\ndata: {}\n\n"
        finally:
            _event_store.unsubscribe(queue)
            task.cancel()

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",
            "Connection": "keep-alive",
        },
    )
```

---

## 7. WebSocket Endpoint for Bidirectional Event Stream

```python
# src/my_agent/api/websocket_endpoint.py
from __future__ import annotations

import asyncio
import json
import uuid
from typing import Any

from fastapi import APIRouter, WebSocket, WebSocketDisconnect

from my_agent.events.cqrs import EventStore
from my_agent.events.models import Event, EventType
from my_agent.events.stream_adapter import graph_events_to_bus


router = APIRouter()


@router.websocket("/ws/agent")
async def agent_websocket(websocket: WebSocket) -> None:
    await websocket.accept()
    event_store = EventStore()
    current_task: asyncio.Task | None = None

    try:
        while True:
            raw = await websocket.receive_text()
            try:
                message = json.loads(raw)
            except json.JSONDecodeError:
                await websocket.send_text(json.dumps({"error": "Invalid JSON"}))
                continue

            action = message.get("action")

            if action == "run":
                thread_id = message.get("thread_id") or str(uuid.uuid4())
                user_message = message.get("message", "")

                # Cancel previous task if running
                if current_task and not current_task.done():
                    current_task.cancel()

                queue = event_store.subscribe()

                async def run_and_stream() -> None:
                    from my_agent.graph import build_react_graph
                    from my_agent.tools import get_tools

                    graph = build_react_graph(get_tools())
                    config = {"configurable": {"thread_id": thread_id}}
                    send_task = asyncio.create_task(
                        _send_events_from_queue(websocket, queue, event_store)
                    )
                    try:
                        await graph_events_to_bus(
                            graph,
                            {"messages": [{"role": "user", "content": user_message}]},
                            config,
                            event_store,
                        )
                    finally:
                        send_task.cancel()
                        event_store.unsubscribe(queue)

                current_task = asyncio.create_task(run_and_stream())

            elif action == "resume":
                # Resume a paused graph after interrupt
                # send Command(resume=...) to the graph
                pass

            elif action == "cancel":
                if current_task:
                    current_task.cancel()
                    await websocket.send_text(json.dumps({"event": "cancelled"}))

    except WebSocketDisconnect:
        if current_task:
            current_task.cancel()


async def _send_events_from_queue(
    websocket: WebSocket,
    queue: asyncio.Queue,
    event_store: EventStore,
) -> None:
    while True:
        event: Event = await queue.get()
        await websocket.send_text(event.model_dump_json())
        if event.event_type in (EventType.AGENT_COMPLETED, EventType.AGENT_FAILED):
            break
```

---

## 8. interrupt() as Synchronization Point in Event Stream

```python
# src/my_agent/events/interrupt_handler.py
from __future__ import annotations

from typing import Any
from langgraph.types import Command


async def handle_interrupt(
    graph: Any,
    config: dict,
    interrupt_payload: dict,
    websocket: Any,
) -> Any:
    """
    When a graph hits interrupt(), pause and wait for human input via WebSocket.
    Then resume with Command(resume=...).
    """
    import json

    # Notify client about interrupt
    await websocket.send_text(json.dumps({
        "event": "human.interrupt",
        "data": interrupt_payload,
    }))

    # Wait for human response
    raw_response = await websocket.receive_text()
    human_input = json.loads(raw_response).get("value", "")

    # Resume the graph
    result = await graph.ainvoke(Command(resume=human_input), config=config)
    return result
```

---

## 9. Redis Streams as Durable Event Bus

```python
# src/my_agent/events/redis_bus.py
from __future__ import annotations

import json
from typing import AsyncGenerator

import aioredis

from my_agent.config import settings
from my_agent.events.models import Event


class RedisEventBus:
    """Durable event bus using Redis Streams."""

    STREAM_KEY = "agent:events"
    GROUP_NAME = "agent_consumers"

    def __init__(self) -> None:
        self._redis: aioredis.Redis | None = None

    async def connect(self) -> None:
        self._redis = await aioredis.from_url(settings.redis_url, decode_responses=True)
        # Create consumer group (ignore if exists)
        try:
            await self._redis.xgroup_create(self.STREAM_KEY, self.GROUP_NAME, id="0", mkstream=True)
        except aioredis.ResponseError:
            pass  # group already exists

    async def publish(self, event: Event) -> str:
        """Append event to Redis stream. Returns message ID."""
        assert self._redis is not None
        msg_id = await self._redis.xadd(
            self.STREAM_KEY,
            {"event": event.model_dump_json()},
            maxlen=10_000,  # keep last 10k events
        )
        return msg_id

    async def consume(
        self,
        consumer_name: str,
        batch_size: int = 10,
        block_ms: int = 1000,
    ) -> AsyncGenerator[Event, None]:
        """Consume events as a named consumer in the group."""
        assert self._redis is not None
        while True:
            messages = await self._redis.xreadgroup(
                self.GROUP_NAME,
                consumer_name,
                {self.STREAM_KEY: ">"},
                count=batch_size,
                block=block_ms,
            )
            if not messages:
                continue
            for _, records in messages:
                for msg_id, fields in records:
                    event = Event.model_validate_json(fields["event"])
                    yield event
                    # Acknowledge after processing
                    await self._redis.xack(self.STREAM_KEY, self.GROUP_NAME, msg_id)

    async def close(self) -> None:
        if self._redis:
            await self._redis.close()
```

---

## 10. LangSmith Callback Integration as Event Consumer

```python
# src/my_agent/events/langsmith_consumer.py
from __future__ import annotations

import asyncio
from typing import Any

from langsmith import Client
from my_agent.events.models import Event, EventType
from my_agent.events.cqrs import EventStore


class LangSmithEventConsumer:
    """Consume events from EventStore and send metadata to LangSmith."""

    def __init__(self, event_store: EventStore, project_name: str = "default") -> None:
        self._store = event_store
        self._client = Client()
        self._project = project_name

    async def run(self) -> None:
        queue = self._store.subscribe()
        try:
            while True:
                event: Event = await queue.get()
                await self._handle(event)
        finally:
            self._store.unsubscribe(queue)

    async def _handle(self, event: Event) -> None:
        # Log custom metadata to LangSmith as feedback
        if event.event_type == EventType.AGENT_COMPLETED:
            loop = asyncio.get_event_loop()
            await loop.run_in_executor(
                None,
                self._client.create_feedback,
                event.thread_id,
                "agent_completed",
                {"score": 1.0, "metadata": event.data},
            )
        elif event.event_type == EventType.AGENT_FAILED:
            loop = asyncio.get_event_loop()
            await loop.run_in_executor(
                None,
                self._client.create_feedback,
                event.thread_id,
                "agent_failed",
                {"score": 0.0, "metadata": event.data},
            )
```
