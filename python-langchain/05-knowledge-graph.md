# 05 — Knowledge Graph

## 1. Pydantic Models: Entity, Relation, KnowledgeGraph

```python
# src/my_agent/kg/models.py
from __future__ import annotations

from typing import Any
from pydantic import BaseModel, Field


class Entity(BaseModel):
    id: str = Field(description="Unique identifier, e.g. 'person_elon_musk'")
    name: str = Field(description="Human-readable entity name")
    entity_type: str = Field(description="Type: Person, Organization, Concept, Event, etc.")
    description: str = Field(default="", description="Short description of the entity")
    properties: dict[str, Any] = Field(default_factory=dict)
    embedding: list[float] | None = Field(default=None, exclude=True)


class Relation(BaseModel):
    source_id: str = Field(description="Source entity ID")
    target_id: str = Field(description="Target entity ID")
    relation_type: str = Field(description="Relation: 'founded', 'works_at', 'causes', etc.")
    weight: float = Field(default=1.0, ge=0.0, le=1.0)
    properties: dict[str, Any] = Field(default_factory=dict)


class KnowledgeGraph(BaseModel):
    entities: list[Entity] = Field(default_factory=list)
    relations: list[Relation] = Field(default_factory=list)

    def merge(self, other: "KnowledgeGraph") -> "KnowledgeGraph":
        """Merge two KGs, deduplicating by entity ID."""
        entity_map = {e.id: e for e in self.entities}
        for e in other.entities:
            entity_map[e.id] = e
        rel_set = {(r.source_id, r.target_id, r.relation_type) for r in self.relations}
        new_rels = [r for r in other.relations
                    if (r.source_id, r.target_id, r.relation_type) not in rel_set]
        return KnowledgeGraph(
            entities=list(entity_map.values()),
            relations=self.relations + new_rels,
        )
```

---

## 2. Triple Extraction Chain with Structured Output

```python
# src/my_agent/kg/extraction.py
from __future__ import annotations

from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

from my_agent.kg.models import KnowledgeGraph


EXTRACTION_PROMPT = ChatPromptTemplate.from_messages([
    ("system", (
        "You are an expert knowledge graph builder. Extract entities and relationships "
        "from the given text.\n\n"
        "Rules:\n"
        "- Entity IDs must be snake_case, lowercase, unique\n"
        "- Entity types: Person, Organization, Concept, Technology, Event, Location, Product\n"
        "- Relation types should be verb phrases: 'founded_by', 'located_in', 'uses', etc.\n"
        "- Only extract facts explicitly stated in the text\n"
        "- Weight = confidence (0.0–1.0)"
    )),
    ("human", "Extract a knowledge graph from this text:\n\n{text}"),
])


def build_extraction_chain() -> Any:
    """Returns a chain that extracts a KnowledgeGraph from text."""
    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    structured_llm = llm.with_structured_output(KnowledgeGraph)
    return EXTRACTION_PROMPT | structured_llm


# Usage:
# chain = build_extraction_chain()
# kg: KnowledgeGraph = chain.invoke({"text": "Elon Musk founded Tesla in 2003..."})
```

---

## 3. In-Memory Graph (networkx)

```python
# src/my_agent/kg/graph_store.py
from __future__ import annotations

import json
from pathlib import Path
from typing import Any

import networkx as nx
from my_agent.kg.models import Entity, KnowledgeGraph, Relation


class NetworkXGraphStore:
    """In-memory knowledge graph backed by networkx DiGraph."""

    def __init__(self) -> None:
        self._graph: nx.DiGraph = nx.DiGraph()

    def add_kg(self, kg: KnowledgeGraph) -> None:
        for entity in kg.entities:
            self._graph.add_node(
                entity.id,
                name=entity.name,
                entity_type=entity.entity_type,
                description=entity.description,
                properties=entity.properties,
            )
        for rel in kg.relations:
            self._graph.add_edge(
                rel.source_id,
                rel.target_id,
                relation_type=rel.relation_type,
                weight=rel.weight,
                properties=rel.properties,
            )

    def get_neighbors(
        self,
        entity_id: str,
        depth: int = 2,
        max_nodes: int = 50,
    ) -> nx.DiGraph:
        """Return subgraph within `depth` hops of `entity_id`."""
        nodes = nx.single_source_shortest_path_length(
            self._graph, entity_id, cutoff=depth
        )
        subgraph_nodes = list(nodes.keys())[:max_nodes]
        return self._graph.subgraph(subgraph_nodes).copy()

    def search_entities(self, query: str, entity_type: str | None = None) -> list[dict]:
        results = []
        for node_id, data in self._graph.nodes(data=True):
            if query.lower() in data.get("name", "").lower():
                if entity_type is None or data.get("entity_type") == entity_type:
                    results.append({"id": node_id, **data})
        return results

    def get_paths(self, source_id: str, target_id: str, max_paths: int = 3) -> list[list[str]]:
        """Find simple paths between two entities."""
        try:
            paths = list(nx.all_simple_paths(
                self._graph, source_id, target_id, cutoff=5
            ))
            return paths[:max_paths]
        except nx.NetworkXNoPath:
            return []

    def to_json(self, path: str) -> None:
        data = nx.node_link_data(self._graph)
        Path(path).write_text(json.dumps(data, indent=2))

    @classmethod
    def from_json(cls, path: str) -> "NetworkXGraphStore":
        store = cls()
        data = json.loads(Path(path).read_text())
        store._graph = nx.node_link_graph(data)
        return store

    def subgraph_to_text(self, subgraph: nx.DiGraph) -> str:
        """Convert a subgraph to a text representation for LLM context."""
        lines: list[str] = []
        for node_id, data in subgraph.nodes(data=True):
            lines.append(f"[{data.get('entity_type','?')}] {data.get('name', node_id)}: {data.get('description','')}")
        for src, tgt, data in subgraph.edges(data=True):
            src_name = subgraph.nodes[src].get("name", src)
            tgt_name = subgraph.nodes[tgt].get("name", tgt)
            lines.append(f"  {src_name} --[{data.get('relation_type','?')}]--> {tgt_name}")
        return "\n".join(lines)
```

---

## 4. Neo4j Integration

```python
# src/my_agent/kg/neo4j_store.py
from __future__ import annotations

from typing import Any

from langchain_neo4j import Neo4jGraph, GraphCypherQAChain
from langchain_openai import ChatOpenAI

from my_agent.config import settings
from my_agent.kg.models import KnowledgeGraph


def get_neo4j_graph() -> Neo4jGraph:
    return Neo4jGraph(
        url=settings.neo4j_uri,
        username=settings.neo4j_username,
        password=settings.neo4j_password.get_secret_value(),
    )


def ingest_kg_to_neo4j(kg: KnowledgeGraph, graph: Neo4jGraph) -> None:
    """Batch-ingest a KnowledgeGraph into Neo4j."""
    for entity in kg.entities:
        graph.query(
            "MERGE (e {id: $id}) SET e.name = $name, e.type = $type, e.description = $desc",
            {"id": entity.id, "name": entity.name, "type": entity.entity_type,
             "desc": entity.description},
        )
    for rel in kg.relations:
        cypher = (
            f"MATCH (a {{id: $src}}), (b {{id: $tgt}}) "
            f"MERGE (a)-[r:{rel.relation_type.upper().replace(' ','_')}]->(b) "
            f"SET r.weight = $weight"
        )
        graph.query(cypher, {"src": rel.source_id, "tgt": rel.target_id, "weight": rel.weight})


def build_graph_qa_chain() -> GraphCypherQAChain:
    """Build a chain that answers questions by generating + executing Cypher."""
    graph = get_neo4j_graph()
    graph.refresh_schema()
    return GraphCypherQAChain.from_llm(
        llm=ChatOpenAI(model="gpt-4o", temperature=0),
        graph=graph,
        verbose=True,
        allow_dangerous_requests=True,
    )
```

---

## 5. GraphRAG Pipeline: Extract → Leiden Community → Summarize (arXiv:2404.16130)

```python
# src/my_agent/kg/graphrag.py
from __future__ import annotations

from typing import Any

import networkx as nx
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

from my_agent.kg.extraction import build_extraction_chain
from my_agent.kg.graph_store import NetworkXGraphStore
from my_agent.kg.models import KnowledgeGraph


COMMUNITY_SUMMARY_PROMPT = ChatPromptTemplate.from_messages([
    ("system", "Summarize the following entities and their relationships into a coherent paragraph."),
    ("human", "{community_text}"),
])


def leiden_communities(graph: nx.DiGraph, resolution: float = 1.0) -> dict[str, int]:
    """
    Approximate Leiden via greedy modularity (networkx).
    In production, use the `leidenalg` or `cdlib` library for true Leiden.
    """
    undirected = graph.to_undirected()
    communities = nx.community.greedy_modularity_communities(undirected)
    node_to_community: dict[str, int] = {}
    for i, comm in enumerate(communities):
        for node in comm:
            node_to_community[node] = i
    return node_to_community


async def build_graphrag_index(documents: list[str]) -> dict[str, str]:
    """
    GraphRAG indexing pipeline.
    Returns: {community_id: summary_text}
    """
    extraction_chain = build_extraction_chain()
    store = NetworkXGraphStore()
    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
    summary_chain = COMMUNITY_SUMMARY_PROMPT | llm

    # 1. Extract KG from all documents
    for doc in documents:
        kg: KnowledgeGraph = await extraction_chain.ainvoke({"text": doc})
        store.add_kg(kg)

    # 2. Detect communities
    communities = leiden_communities(store._graph)

    # 3. Group nodes by community
    community_groups: dict[int, list[str]] = {}
    for node_id, comm_id in communities.items():
        community_groups.setdefault(comm_id, []).append(node_id)

    # 4. Summarize each community
    summaries: dict[str, str] = {}
    for comm_id, node_ids in community_groups.items():
        subgraph = store._graph.subgraph(node_ids)
        text = store.subgraph_to_text(subgraph)
        result = await summary_chain.ainvoke({"community_text": text})
        summaries[str(comm_id)] = result.content

    return summaries


async def graphrag_query(
    question: str,
    community_summaries: dict[str, str],
    top_k: int = 3,
) -> str:
    """
    GraphRAG query: retrieve relevant community summaries, then answer.
    """
    from langchain_community.vectorstores import FAISS
    from langchain_openai import OpenAIEmbeddings
    from langchain_core.documents import Document

    docs = [
        Document(page_content=summary, metadata={"community_id": cid})
        for cid, summary in community_summaries.items()
    ]
    vectorstore = FAISS.from_documents(docs, OpenAIEmbeddings())
    relevant = vectorstore.similarity_search(question, k=top_k)
    context = "\n\n".join(d.page_content for d in relevant)

    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    prompt = f"Context:\n{context}\n\nQuestion: {question}\n\nAnswer:"
    result = await llm.ainvoke(prompt)
    return result.content
```

---

## 6. HippoRAG: KG + Personalized PageRank (arXiv:2405.14831)

```python
# src/my_agent/kg/hipporag.py
from __future__ import annotations

import numpy as np
import networkx as nx
from typing import Any

from my_agent.kg.graph_store import NetworkXGraphStore


def personalized_pagerank(
    graph: nx.DiGraph,
    seed_nodes: list[str],
    alpha: float = 0.15,
    max_iter: int = 100,
) -> dict[str, float]:
    """
    Personalized PageRank (PPR) from seed nodes.
    Higher alpha = more influence from seed nodes.
    Returns node -> score dict.
    """
    personalization = {node: 0.0 for node in graph.nodes()}
    for node in seed_nodes:
        if node in personalization:
            personalization[node] = 1.0 / len(seed_nodes)

    # Normalize so values sum to 1
    total = sum(personalization.values())
    if total > 0:
        personalization = {k: v / total for k, v in personalization.items()}

    scores = nx.pagerank(
        graph,
        alpha=1 - alpha,  # networkx uses alpha for damping
        personalization=personalization,
        max_iter=max_iter,
    )
    return scores


async def hipporag_retrieve(
    query: str,
    store: NetworkXGraphStore,
    entity_vectorstore: Any,
    top_k_entities: int = 5,
    top_k_docs: int = 10,
) -> list[str]:
    """
    HippoRAG retrieval:
    1. Find seed entities via vector similarity to query
    2. Run PPR from seed entities
    3. Return top-k nodes as context
    """
    # 1. Find seed entities
    seed_results = entity_vectorstore.similarity_search(query, k=top_k_entities)
    seed_ids = [doc.metadata["entity_id"] for doc in seed_results if "entity_id" in doc.metadata]

    if not seed_ids:
        return []

    # 2. PPR
    scores = personalized_pagerank(store._graph, seed_ids)

    # 3. Top-k nodes by PPR score
    top_nodes = sorted(scores.items(), key=lambda x: x[1], reverse=True)[:top_k_docs]
    node_ids = [nid for nid, _ in top_nodes]

    subgraph = store._graph.subgraph(node_ids)
    return [store.subgraph_to_text(subgraph)]
```

---

## 7. LightRAG Integration (arXiv:2410.05779)

```python
# src/my_agent/kg/lightrag_client.py
from __future__ import annotations

import asyncio
import os
from pathlib import Path

from lightrag import LightRAG, QueryParam
from lightrag.llm import openai_complete_if_cache, openai_embedding
from lightrag.utils import EmbeddingFunc


WORKING_DIR = "./lightrag_data"


async def _llm_func(prompt: str, **kwargs: Any) -> str:
    return await openai_complete_if_cache(
        model="gpt-4o",
        prompt=prompt,
        api_key=os.environ["OPENAI_API_KEY"],
        **kwargs,
    )


async def _embed_func(texts: list[str]) -> np.ndarray:
    return await openai_embedding(
        texts,
        model="text-embedding-3-large",
        api_key=os.environ["OPENAI_API_KEY"],
    )


def get_lightrag() -> LightRAG:
    Path(WORKING_DIR).mkdir(exist_ok=True)
    return LightRAG(
        working_dir=WORKING_DIR,
        llm_model_func=_llm_func,
        embedding_func=EmbeddingFunc(
            embedding_dim=3072,
            max_token_size=8192,
            func=_embed_func,
        ),
    )


async def lightrag_insert(documents: list[str]) -> None:
    rag = get_lightrag()
    for doc in documents:
        await rag.ainsert(doc)


async def lightrag_query(
    question: str,
    mode: str = "hybrid",  # "naive" | "local" | "global" | "hybrid" | "mix"
) -> str:
    """
    Query LightRAG with different modes:
    - naive: simple chunk retrieval
    - local: entity-focused subgraph
    - global: community-level summaries
    - hybrid: local + global combined
    - mix: all modes merged
    """
    rag = get_lightrag()
    result = await rag.aquery(
        question,
        param=QueryParam(mode=mode),
    )
    return result
```

---

## 8. KG Persistence and Neo4j Export

```python
# src/my_agent/kg/persistence.py
from __future__ import annotations

import json
from pathlib import Path
from my_agent.kg.graph_store import NetworkXGraphStore
from my_agent.kg.models import Entity, KnowledgeGraph, Relation


def export_to_neo4j_cypher(store: NetworkXGraphStore, output_path: str) -> None:
    """Export the graph as a Cypher script for batch import."""
    lines: list[str] = []
    for node_id, data in store._graph.nodes(data=True):
        name = data.get("name", node_id).replace("'", "\\'")
        etype = data.get("entity_type", "Unknown")
        desc = data.get("description", "").replace("'", "\\'")
        lines.append(
            f"MERGE (n:{etype} {{id: '{node_id}'}}) "
            f"SET n.name = '{name}', n.description = '{desc}';"
        )
    for src, tgt, data in store._graph.edges(data=True):
        rel = data.get("relation_type", "RELATED_TO").upper().replace(" ", "_")
        weight = data.get("weight", 1.0)
        lines.append(
            f"MATCH (a {{id: '{src}'}}), (b {{id: '{tgt}'}}) "
            f"MERGE (a)-[r:{rel}]->(b) SET r.weight = {weight};"
        )
    Path(output_path).write_text("\n".join(lines))


def export_to_jsonl(store: NetworkXGraphStore, output_path: str) -> None:
    with open(output_path, "w") as f:
        for node_id, data in store._graph.nodes(data=True):
            f.write(json.dumps({"type": "entity", "id": node_id, **data}) + "\n")
        for src, tgt, data in store._graph.edges(data=True):
            f.write(json.dumps({"type": "relation", "source": src, "target": tgt, **data}) + "\n")
```

---

## 9. Entity Embeddings in Qdrant

```python
# src/my_agent/kg/entity_embeddings.py
from __future__ import annotations

from langchain_core.documents import Document
from langchain_qdrant import QdrantVectorStore
from langchain_openai import OpenAIEmbeddings
from qdrant_client import QdrantClient

from my_agent.kg.models import Entity


def index_entities(entities: list[Entity], collection: str = "kg_entities") -> QdrantVectorStore:
    """Embed entity descriptions and store in Qdrant for similarity search."""
    docs = [
        Document(
            page_content=f"{e.name} ({e.entity_type}): {e.description}",
            metadata={"entity_id": e.id, "entity_type": e.entity_type, "name": e.name},
        )
        for e in entities
        if e.description  # only index entities with descriptions
    ]
    return QdrantVectorStore.from_documents(
        docs,
        embedding=OpenAIEmbeddings(model="text-embedding-3-large"),
        url="http://localhost:6333",
        collection_name=collection,
    )


async def find_similar_entities(
    query: str,
    entity_vectorstore: QdrantVectorStore,
    entity_type: str | None = None,
    k: int = 5,
) -> list[Document]:
    """Find entities similar to a query, optionally filtered by type."""
    search_kwargs: dict = {"k": k}
    if entity_type:
        search_kwargs["filter"] = {"entity_type": entity_type}
    return entity_vectorstore.similarity_search(query, **search_kwargs)
```
