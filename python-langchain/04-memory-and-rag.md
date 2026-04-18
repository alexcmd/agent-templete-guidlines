# 04 — Memory and RAG

## 1. LangGraph Checkpointer as Working Memory

Checkpointers persist the full graph state after every step, enabling resumable sessions.

```python
# src/my_agent/memory/checkpointer.py
from __future__ import annotations

from typing import Any

from langgraph.checkpoint.memory import InMemorySaver
from langgraph.checkpoint.sqlite.aio import AsyncSqliteSaver
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

from my_agent.config import settings


async def get_checkpointer(mode: str = "postgres") -> Any:
    """Factory for the appropriate checkpointer backend."""
    match mode:
        case "memory":
            return InMemorySaver()

        case "sqlite":
            # Good for dev / single-process
            return await AsyncSqliteSaver.from_conn_string("agent_state.db").__aenter__()

        case "postgres":
            # Production: multi-process, durable
            checkpointer = AsyncPostgresSaver.from_conn_string(settings.postgres_url)
            await checkpointer.setup()   # creates tables if not exist
            return checkpointer

        case _:
            raise ValueError(f"Unknown checkpointer mode: {mode}")
```

### Thread configuration

```python
# Every graph invocation needs a thread_id for state isolation
config = {
    "configurable": {
        "thread_id": "user-123-session-456",
        # Optional: checkpoint_ns for sub-graphs
        "checkpoint_ns": "",
    }
}
result = await graph.ainvoke(inputs, config=config)
```

---

## 2. LangGraph Store for Cross-Thread Semantic Memory

```python
# src/my_agent/memory/store.py
from __future__ import annotations

import json
from typing import Any

from langgraph.store.memory import InMemoryStore
from langgraph.store.postgres import AsyncPostgresStore

from my_agent.config import settings


async def get_store(mode: str = "postgres") -> Any:
    if mode == "memory":
        return InMemoryStore()
    store = AsyncPostgresStore.from_conn_string(settings.postgres_url)
    await store.setup()
    return store


# ----------- Usage in a node -----------
async def user_preference_node(state: Any, *, store: Any) -> dict[str, Any]:
    """Read/write user preferences across threads."""
    user_id = state["user_id"]
    namespace = ("users", user_id, "preferences")

    # Read
    items = await store.asearch(namespace, query="notification settings", limit=5)
    prefs = {item.key: item.value for item in items}

    # Write
    await store.aput(namespace, "dark_mode", {"enabled": True})

    return {"scratchpad": {"user_prefs": prefs}}
```

---

## 3. MemGPT 3-Tier Memory as LangGraph Tools (arXiv:2310.08560)

Three tiers: core (in-context), recall (vector search), archival (unlimited storage).

```python
# src/my_agent/memory/memgpt.py
from __future__ import annotations

from typing import Annotated, Any
from pydantic import BaseModel, Field
from langchain_core.tools import tool
from langchain_openai import OpenAIEmbeddings
from langgraph.prebuilt import InjectedState
from my_agent.state import AgentState


# ---- Core memory (in-context, always present) ----
class CoreMemory(BaseModel):
    persona: str = Field(default="", description="Agent's persona and goals")
    human: str = Field(default="", description="What agent knows about the user")


@tool
def update_core_memory(
    section: str,
    content: str,
    state: Annotated[AgentState, InjectedState],
) -> str:
    """Update core memory section ('persona' or 'human')."""
    # Returns state update — ToolNode applies it
    core: dict = state.get("scratchpad", {}).get("core_memory", {})
    core[section] = content
    return f"Core memory '{section}' updated."


# ---- Recall memory (recent episodes, vector search) ----
_recall_store: list[dict[str, Any]] = []   # in-memory placeholder

@tool
async def search_recall_memory(query: str, limit: int = 5) -> str:
    """Search recall memory for relevant past interactions."""
    if not _recall_store:
        return "No recall memories stored yet."
    # In production: replace with Qdrant/pgvector similarity search
    matches = [m for m in _recall_store if query.lower() in m["content"].lower()]
    return "\n".join(m["content"] for m in matches[:limit]) or "No matches found."


@tool
async def add_recall_memory(content: str, importance: float = 0.5) -> str:
    """Store an interaction in recall memory."""
    _recall_store.append({"content": content, "importance": importance})
    return f"Stored in recall memory (total: {len(_recall_store)})."


# ---- Archival memory (unlimited, database-backed) ----
@tool
async def search_archival(query: str, limit: int = 10) -> str:
    """Search archival memory using semantic similarity."""
    # In production: Qdrant MMR search
    return f"[ARCHIVAL SEARCH] Query: '{query}' — implement with Qdrant."


@tool
async def insert_archival(content: str) -> str:
    """Insert content into archival memory for long-term storage."""
    return f"[ARCHIVAL INSERT] Stored {len(content)} chars."


MEMGPT_TOOLS = [update_core_memory, search_recall_memory, add_recall_memory,
                search_archival, insert_archival]
```

---

## 4. Ebbinghaus Decay Scoring

```python
# src/my_agent/memory/decay.py
from __future__ import annotations

import math
from datetime import datetime, timedelta


def ebbinghaus_retention(
    time_elapsed_hours: float,
    stability: float = 1.0,
    initial_strength: float = 1.0,
) -> float:
    """
    Compute Ebbinghaus forgetting curve retention score.
    R = e^(-t / (S * initial_strength))
    where t = elapsed time, S = memory stability parameter.
    Returns value in [0, 1].
    """
    return math.exp(-time_elapsed_hours / (stability * initial_strength))


def score_memory(created_at: datetime, last_accessed: datetime, access_count: int) -> float:
    """Score a memory item by recency and access frequency."""
    now = datetime.utcnow()
    hours_since_creation = (now - created_at).total_seconds() / 3600
    hours_since_access = (now - last_accessed).total_seconds() / 3600

    # Stability increases with repeated access
    stability = 1.0 + math.log1p(access_count)

    decay = ebbinghaus_retention(hours_since_access, stability=stability)
    return round(decay, 4)
```

---

## 5. Mem0 Integration (arXiv:2504.19413)

```python
# src/my_agent/memory/mem0_client.py
from __future__ import annotations

from mem0 import AsyncMemoryClient
from my_agent.config import settings


_client: AsyncMemoryClient | None = None


def get_mem0_client() -> AsyncMemoryClient:
    global _client
    if _client is None:
        _client = AsyncMemoryClient()  # reads MEM0_API_KEY from env
    return _client


async def remember(user_id: str, content: str) -> dict:
    """Extract and store memories from content."""
    client = get_mem0_client()
    return await client.add(content, user_id=user_id)


async def recall(user_id: str, query: str, limit: int = 5) -> list[dict]:
    """Retrieve relevant memories for a user."""
    client = get_mem0_client()
    results = await client.search(query, user_id=user_id, limit=limit)
    return results


async def get_all_memories(user_id: str) -> list[dict]:
    client = get_mem0_client()
    return await client.get_all(user_id=user_id)
```

---

## 6. Modular RAG Pipeline as LCEL Chain

```
QueryRewriter → HybridRetriever → Reranker → ContextCompressor → Generator
```

```python
# src/my_agent/rag/pipeline.py
from __future__ import annotations

from operator import itemgetter
from typing import Any

from langchain_core.documents import Document
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import (
    RunnableLambda,
    RunnableParallel,
    RunnablePassthrough,
)
from langchain_openai import ChatOpenAI

from my_agent.rag.retrievers import build_hybrid_retriever, rerank_documents


LLM = ChatOpenAI(model="gpt-4o", temperature=0, streaming=True)


QUERY_REWRITE_PROMPT = ChatPromptTemplate.from_messages([
    ("system", "Rewrite the following question to be more specific and searchable."),
    ("human", "{question}"),
])

RAG_PROMPT = ChatPromptTemplate.from_messages([
    ("system", (
        "You are a helpful assistant. Answer the question using ONLY the provided context. "
        "If the answer is not in the context, say 'I don't know.'\n\n"
        "Context:\n{context}"
    )),
    ("human", "{question}"),
])


def format_docs(docs: list[Document]) -> str:
    return "\n\n---\n\n".join(
        f"[Source: {doc.metadata.get('source', 'unknown')}]\n{doc.page_content}"
        for doc in docs
    )


def build_modular_rag_chain(retriever: Any) -> Any:
    """
    Full modular RAG pipeline using LCEL composition.
    Input: {"question": str}
    Output: str (answer)
    """
    query_rewriter = QUERY_REWRITE_PROMPT | LLM | StrOutputParser()

    def retrieve_and_rerank(rewritten_query: str) -> list[Document]:
        docs = retriever.invoke(rewritten_query)
        return rerank_documents(rewritten_query, docs, top_k=5)

    chain = (
        RunnablePassthrough.assign(
            rewritten_query=query_rewriter
        )
        | RunnablePassthrough.assign(
            context=lambda x: format_docs(retrieve_and_rerank(x["rewritten_query"]))
        )
        | RAG_PROMPT
        | LLM
        | StrOutputParser()
    )
    return chain
```

---

## 7. HyDE — Hypothetical Document Embeddings (arXiv:2212.10496)

```python
# src/my_agent/rag/hyde.py
from __future__ import annotations

from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnableLambda
from langchain_openai import ChatOpenAI, OpenAIEmbeddings


HYDE_PROMPT = ChatPromptTemplate.from_messages([
    ("system", "Write a concise document that would perfectly answer this question."),
    ("human", "{question}"),
])


def build_hyde_retriever(vector_store: Any) -> Any:
    """Use hypothetical document for retrieval, then answer with real docs."""
    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.3)

    hypothetical_doc_chain = HYDE_PROMPT | llm | StrOutputParser()

    def retrieve_with_hyde(question: str) -> list:
        hyp_doc = hypothetical_doc_chain.invoke({"question": question})
        # Embed the hypothetical doc and search
        return vector_store.similarity_search_by_vector(
            OpenAIEmbeddings().embed_query(hyp_doc),
            k=5,
        )

    return RunnableLambda(retrieve_with_hyde)
```

---

## 8. RAG-Fusion with RRF (arXiv:2402.03367)

```python
# src/my_agent/rag/rag_fusion.py
from __future__ import annotations

from collections import defaultdict
from langchain_core.documents import Document
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI


MULTI_QUERY_PROMPT = ChatPromptTemplate.from_messages([
    ("system", (
        "Generate {num_queries} different search queries for the given question. "
        "Return only the queries, one per line."
    )),
    ("human", "{question}"),
])


def reciprocal_rank_fusion(
    ranked_lists: list[list[Document]],
    k: int = 60,
) -> list[Document]:
    """RRF scoring: score(d) = Σ 1/(rank(d) + k)."""
    scores: dict[str, float] = defaultdict(float)
    doc_map: dict[str, Document] = {}
    for ranked in ranked_lists:
        for rank, doc in enumerate(ranked, 1):
            doc_id = doc.metadata.get("id", doc.page_content[:100])
            scores[doc_id] += 1.0 / (rank + k)
            doc_map[doc_id] = doc
    return [doc_map[doc_id] for doc_id in sorted(scores, key=scores.__getitem__, reverse=True)]


def build_rag_fusion_retriever(base_retriever: Any, num_queries: int = 3) -> Any:
    from langchain_core.runnables import RunnableLambda

    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.3)
    query_chain = MULTI_QUERY_PROMPT | llm | StrOutputParser()

    def fused_retrieve(question: str) -> list[Document]:
        raw = query_chain.invoke({"question": question, "num_queries": num_queries})
        queries = [q.strip() for q in raw.strip().splitlines() if q.strip()]
        ranked_lists = [base_retriever.invoke(q) for q in queries]
        return reciprocal_rank_fusion(ranked_lists)

    return RunnableLambda(fused_retrieve)
```

---

## 9. CRAG with Confidence Threshold + Tavily Fallback (arXiv:2401.15884)

```python
# src/my_agent/rag/crag.py
from __future__ import annotations

from langchain_core.documents import Document
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_community.tools.tavily_search import TavilySearchResults


RELEVANCE_PROMPT = ChatPromptTemplate.from_messages([
    ("system", (
        "Rate the relevance of the document to the query on a scale 0.0–1.0. "
        "Return only the float."
    )),
    ("human", "Query: {query}\n\nDocument: {document}"),
])


async def crag_retrieve(
    query: str,
    retriever: Any,
    confidence_threshold: float = 0.5,
) -> list[Document]:
    """
    CRAG: retrieve → score → fallback to web search if confidence is low.
    """
    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
    docs = retriever.invoke(query)

    # Score top-3 documents
    scores: list[float] = []
    for doc in docs[:3]:
        result = await llm.ainvoke(
            RELEVANCE_PROMPT.format_messages(query=query, document=doc.page_content[:500])
        )
        try:
            scores.append(float(result.content.strip()))
        except ValueError:
            scores.append(0.0)

    avg_confidence = sum(scores) / len(scores) if scores else 0.0

    if avg_confidence >= confidence_threshold:
        return docs

    # Low confidence → fall back to web search
    search = TavilySearchResults(max_results=5)
    web_results = search.invoke(query)
    web_docs = [
        Document(
            page_content=r["content"],
            metadata={"source": r["url"], "type": "web"},
        )
        for r in web_results
    ]
    # Return blend: local docs + web docs
    return docs[:2] + web_docs
```

---

## 10. Advanced Retrievers

### MultiQueryRetriever

```python
from langchain.retrievers.multi_query import MultiQueryRetriever
from langchain_openai import ChatOpenAI

retriever = MultiQueryRetriever.from_llm(
    retriever=base_retriever,
    llm=ChatOpenAI(model="gpt-4o-mini"),
    # Generates 3 variants by default, deduplicates results
)
```

### ContextualCompressionRetriever

```python
from langchain.retrievers.contextual_compression import ContextualCompressionRetriever
from langchain.retrievers.document_compressors.chain_extract import LLMChainExtractor
from langchain_openai import ChatOpenAI

compressor = LLMChainExtractor.from_llm(ChatOpenAI(model="gpt-4o-mini"))
compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=base_retriever,
)
```

### ParentDocumentRetriever

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore
from langchain_text_splitters import RecursiveCharacterTextSplitter

parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000)
child_splitter = RecursiveCharacterTextSplitter(chunk_size=400)
store = InMemoryStore()

parent_retriever = ParentDocumentRetriever(
    vectorstore=child_vectorstore,
    docstore=store,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)
# Adds parent + child chunks; returns parent docs on retrieval
```

### EnsembleRetriever (BM25 + dense)

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

bm25 = BM25Retriever.from_documents(docs, k=5)
dense = vectorstore.as_retriever(search_kwargs={"k": 5})

ensemble = EnsembleRetriever(
    retrievers=[bm25, dense],
    weights=[0.4, 0.6],
)
```

---

## 11. Vector Store Integrations

### FAISS (local dev)

```python
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-large")

# Build from documents
vectorstore = FAISS.from_documents(documents, embeddings)
vectorstore.save_local("faiss_index")

# Load
vectorstore = FAISS.load_local("faiss_index", embeddings, allow_dangerous_deserialization=True)

# MMR retrieval
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={"k": 6, "fetch_k": 20, "lambda_mult": 0.7},
)
```

### Qdrant with Hybrid Search

```python
from langchain_qdrant import QdrantVectorStore, RetrievalMode, FastEmbedSparse
from langchain_openai import OpenAIEmbeddings
from qdrant_client import QdrantClient

client = QdrantClient(url="http://localhost:6333")

vectorstore = QdrantVectorStore.from_documents(
    documents,
    embedding=OpenAIEmbeddings(model="text-embedding-3-large"),
    sparse_embedding=FastEmbedSparse(model_name="Qdrant/bm25"),
    location="http://localhost:6333",
    collection_name="my_docs",
    retrieval_mode=RetrievalMode.HYBRID,
)

retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 10, "score_threshold": 0.5},
)
```

### pgvector

```python
from langchain_postgres import PGVector
from langchain_openai import OpenAIEmbeddings

vectorstore = PGVector(
    embeddings=OpenAIEmbeddings(model="text-embedding-3-large"),
    collection_name="documents",
    connection=settings.postgres_url,
    use_jsonb=True,
)

# Filtered search
retriever = vectorstore.as_retriever(
    search_type="similarity_score_threshold",
    search_kwargs={
        "k": 5,
        "score_threshold": 0.7,
        "filter": {"source": "internal_docs"},
    },
)
```

---

## 12. BGE-M3 Hybrid Dense + Sparse (arXiv:2402.03216)

```python
# src/my_agent/rag/bge_m3.py
from __future__ import annotations

from FlagEmbedding import BGEM3FlagModel


class BGEM3Embedder:
    """BGE-M3 hybrid dense + sparse embeddings."""

    def __init__(self, model_name: str = "BAAI/bge-m3") -> None:
        self.model = BGEM3FlagModel(model_name, use_fp16=True)

    def embed_documents(self, texts: list[str]) -> list[list[float]]:
        out = self.model.encode(texts, return_dense=True, return_sparse=False)
        return out["dense_vecs"].tolist()

    def embed_query(self, text: str) -> list[float]:
        return self.embed_documents([text])[0]

    def encode_hybrid(self, texts: list[str]) -> dict:
        return self.model.encode(
            texts,
            return_dense=True,
            return_sparse=True,
            return_colbert_vecs=False,
        )
```
