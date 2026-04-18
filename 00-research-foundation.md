# Research Foundation: arXiv Paper Catalog

Comprehensive reference for the 24 peer-reviewed papers that ground the agent implementation
guidelines in this repository. Each entry includes: full citation, problem statement, core
contribution, key algorithm or equation, and practical implementation insight.

---

## Table of Contents

| # | Category | Paper | arXiv |
|---|----------|-------|-------|
| 1 | **Agent Loop** | ReAct | 2210.03629 |
| 2 | **Agent Loop** | Reflexion | 2303.11366 |
| 3 | **Agent Framework** | OpenHands | 2407.16741 |
| 4 | **Memory** | MemGPT | 2310.08560 |
| 5 | **Memory** | CoALA | 2309.02427 |
| 6 | **Memory** | A-MEM | 2502.12110 |
| 7 | **Memory** | Mem0 | 2504.19413 |
| 8 | **KG + RAG** | GraphRAG | 2404.16130 |
| 9 | **KG + RAG** | HippoRAG | 2405.14831 |
| 10 | **KG + RAG** | LightRAG | 2410.05779 |
| 11 | **KG + RAG** | PathRAG | 2502.14902 |
| 12 | **RAG** | Self-RAG | 2310.11511 |
| 13 | **RAG** | CRAG | 2401.15884 |
| 14 | **RAG** | HyDE | 2212.10496 |
| 15 | **RAG** | RAG-Fusion | 2402.03367 |
| 16 | **Architecture** | ESAA | 2602.23193 |
| 17 | **Embeddings** | BGE-M3 | 2402.03216 |
| 18 | **Self-Improvement** | ExpeL | 2308.10144 |
| 19 | **Self-Improvement** | SELF-REFINE | selfrefine.info |
| 20 | **Self-Improvement** | CRITIC | 2310.06825 |
| 21 | **Multi-Agent** | AutoGen | 2308.08155 |
| 22 | **Multi-Agent** | MetaGPT | 2308.00352 |
| 23 | **Efficiency** | StreamingLLM | 2309.17453 |
| 24 | **Evaluation** | LongMemEval | 2410.10813 |

---

## 1. ReAct: Synergizing Reasoning and Acting in Language Models

**Citation:** Yao, S., Zhao, J., Yu, D., Du, N., Shafran, I., Narasimhan, K., & Cao, Y. (2022).
*ReAct: Synergizing Reasoning and Acting in Language Models.*
arXiv:2210.03629. ICLR 2023.

**Problem:** LLMs either reason without acting (chain-of-thought) or act without reasoning
(action prediction), leading to hallucinations or inflexibility.

**Contribution:** Interleave reasoning traces (Thought) with actions (Action/Observation) in
a single LLM prompt. This allows the model to dynamically update plans based on tool feedback.

**Core Algorithm:**
```
Thought:  <reasoning about current state>
Action:   <tool_name>[<tool_args>]
Observation: <tool_result>
... repeat ...
Final Answer: <answer>
```

**Key Results:** +34% on ALFWorld, +10% on WebShop vs. chain-of-thought baseline.
Human interpretability significantly improved vs. action-only baselines.

**Implementation Insight:** Parse the Thought/Action/Observation format with regex or structured
output. Truncate long observations (e.g., >8000 chars) to stay within context. Use at least
`max_iterations=10` to allow multi-hop reasoning. The Thought step is critical — models that
skip it produce lower-quality actions.

---

## 2. Reflexion: Language Agents with Verbal Reinforcement Learning

**Citation:** Shinn, N., Cassano, F., Gopinath, A., Narasimhan, K., & Yao, S. (2023).
*Reflexion: Language Agents with Verbal Reinforcement Learning.*
arXiv:2303.11366. NeurIPS 2023.

**Problem:** RL for LLM agents requires expensive gradient updates. Agents often repeat the
same mistakes across attempts on the same task.

**Contribution:** Replace gradient updates with verbal self-reflection: after each failed
attempt, generate a natural-language critique, store it in an episodic memory buffer, and
prepend it to subsequent attempts as context.

**Core Algorithm:**
```
Episodic memory M = []
For attempt t = 1..max_attempts:
    prompt = system_prompt + format_reflections(M) + task
    response = LLM(prompt)
    score = evaluate(response)
    if score >= threshold: return response
    reflection = LLM("Reflect on failure: " + response + " score=" + score)
    M.append(reflection)
```

**Key Results:** 91% pass@1 on HumanEval (vs. 80% baseline), 130% improvement on AlfWorld
sequential tasks. Verbal reflections outperform numerical rewards in sample efficiency.

**Implementation Insight:** The reflection quality improves with a dedicated "reflector" prompt
separate from the actor prompt. Cap episodic buffer at 3–5 reflections (older ones become
noise). For code tasks, include the error message and failing test in the reflection prompt.

---

## 3. OpenHands: An Open Platform for AI Software Developers as Generalist Agents

**Citation:** Wang, X., et al. (2024).
*OpenHands: An Open Platform for AI Software Developers as Generalist Agents.*
arXiv:2407.16741.

**Problem:** Existing agent platforms are monolithic, hard to extend, and mix execution
logic with agent policy.

**Contribution:** Event-sourced state machine architecture where all agent state is derived
from an append-only event stream. Tools are typed (BashTool, BrowserTool, FileEditTool) and
the controller loop is fully decoupled from the LLM backend.

**Core Architecture:**
```
EventStream (append-only log)
    ↓
AgentController.step()
    ↓ reads all events → reconstructs state
LLM.complete(state) → Action
    ↓
ActionExecutor.run(action) → Observation
    ↓ append both to EventStream
```

**Key Results:** Achieves 37.1% on SWE-Bench Verified, the highest open-source score at
publication time.

**Implementation Insight:** Store every action and observation as a typed event with a UUID,
timestamp, and session ID. This enables: deterministic replay, crash recovery (resume from
last event), parallel execution (fork event streams), and full audit trails. Never mutate
state in place — always append.

---

## 4. MemGPT: Towards LLMs as Operating Systems

**Citation:** Packer, C., Fang, V., Patil, S. G., Rhea, S., & Gonzalez, J. E. (2023).
*MemGPT: Towards LLMs as Operating Systems.*
arXiv:2310.08560. NeurIPS 2023 workshop.

**Problem:** LLMs have fixed context windows that cannot store long-term information beyond
the current conversation.

**Contribution:** Model the LLM as a CPU operating with three memory tiers: (1) main context
(in-context window), (2) recall storage (recent conversation, vector-searchable), and (3)
archival storage (all history, compressed, vector-searchable). Memory CRUD operations are
exposed as tools.

**Core Architecture:**
```
Main context window (in-context, <4K tokens)
  ↓ overflow
Recall storage (recent msgs, BM25 + dense vector search)
  ↓ older msgs
Archival storage (append-only, vector-indexed, no size limit)

Agent tools:
  core_memory_append(key, value)     → write to main context
  recall_memory_search(query, limit) → vector search recent
  archival_memory_search(query, limit) → vector search all history
  archival_memory_insert(content)    → append to archival
```

**Implementation Insight:** The main context budget (~2K tokens for memory blocks) must be
explicitly tracked. Evict oldest recall entries first. Use a fast embedding model (e.g.,
`text-embedding-3-small`) for recall search; use a more powerful one for archival. The key
innovation is that the LLM itself decides when to read/write memory — treat CRUD as normal
tool calls.

---

## 5. CoALA: Cognitive Architectures for Language Agents

**Citation:** Sumers, T. R., Yao, S., Narasimhan, K., & Griffiths, T. L. (2023).
*Cognitive Architectures for Language Agents.*
arXiv:2309.02427. TMLR 2024.

**Problem:** No unified taxonomy exists for memory types in LLM agents, making it hard to
compare architectures or identify gaps.

**Contribution:** Proposes a cognitive architecture framework with four memory stores and
two action spaces:

**Memory Taxonomy:**
```
Working memory    — current context window, active reasoning
Episodic memory   — records of past events, indexed by time
Semantic memory   — general world knowledge, indexed by content
Procedural memory — how-to knowledge (tools, policies, skills)
```

**Action Taxonomy:**
```
External actions  — tool calls, environment interactions
Internal actions  — memory read/write, reasoning, planning
```

**Implementation Insight:** Use this taxonomy to audit your agent's memory design:
- Working = prompt context
- Episodic = conversation history DB + vector search
- Semantic = RAG knowledge base
- Procedural = tool definitions + skill library (ExpeL)

Most production agents implement Working + Semantic well but neglect Episodic (past session
recall) and Procedural (reusable skill patterns).

---

## 6. A-MEM: Agentic Memory System

**Citation:** Xu, W., et al. (2025).
*A-MEM: Agentic Memory System.*
arXiv:2502.12110.

**Problem:** Flat vector stores lose structural relationships between memories. Retrieval is
often semantically off-target because memories lack connections and context.

**Contribution:** Zettelkasten-inspired note graph where each memory is a `MemoryNote` with
forward links to related notes. Uses Ebbinghaus forgetting curve to model memory decay.
New notes trigger automatic link discovery via embedding similarity.

**Core Data Structure:**
```python
@dataclass
class MemoryNote:
    id: str
    content: str            # the memory itself
    description: str        # one-line contextual summary
    keywords: list[str]     # extracted topics
    tags: list[str]         # broad categories
    links: list[str]        # IDs of related notes (forward links)
    strength: float         # Ebbinghaus retention score
    last_accessed: datetime
    access_count: int
```

**Ebbinghaus Decay:**
```
R(t) = e^(-t / S)
where t = days since last access, S = stability (increases with each retrieval)
```

**Key Results:** 2× improvement over MemGPT on LongMemEval, better than Mem0 on multi-hop
memory reasoning tasks.

**Implementation Insight:** The linking step is the key cost driver — run it asynchronously
after insertion. Embed both content and description for retrieval, then follow links for
context expansion. Boost strength score on each access to implement spaced repetition.

---

## 7. Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory

**Citation:** (Mem0 team, 2025).
*Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory.*
arXiv:2504.19413.

**Problem:** Ad-hoc memory implementations repeat themselves across every agent project.
Production memory requires extraction, deduplication, and smart retrieval at scale.

**Contribution:** Four-stage memory pipeline: (1) extract facts from conversation,
(2) deduplicate against existing memories, (3) store with metadata, (4) retrieve by semantic
similarity. Available as `pip install mem0ai`.

**Core Pipeline:**
```
User message
    ↓ LLM extract
[fact_1, fact_2, ..., fact_n]
    ↓ vector search existing memories
[existing_1, existing_2, ...]
    ↓ LLM deduplicate
[new facts only, merged updates]
    ↓ vector store
Memory DB
    ↓ query time: semantic search
Relevant memories → inject into context
```

**Key Results:** 26% improvement on MemoryBench, 90% reduction in tokens sent to LLM vs.
full conversation history injection.

**Implementation Insight:** The deduplication LLM call is the main latency cost. Run it
async post-response. Use `mem0.add(messages, user_id=...)` after every multi-turn session.
At retrieval time, use `mem0.search(query, user_id=...)` and inject the top-5 memories into
the system prompt.

---

## 8. GraphRAG: From Local to Global: A Graph RAG Approach

**Citation:** Edge, D., Trinh, H., Cheng, N., Bradley, J., Chao, A., Mody, A., Truitt, S.,
& Larson, J. (2024).
*From Local to Global: A Graph RAG Approach to Query-Focused Summarization.*
arXiv:2404.16130. Microsoft Research.

**Problem:** Standard RAG retrieves locally relevant chunks but fails on global queries
("What are the main themes across all documents?") because no single chunk contains the answer.

**Contribution:** Build a knowledge graph from the corpus using LLM-extracted entities and
relations. Apply Leiden community detection to find natural clusters. Generate an LLM summary
for each community. Global queries search community summaries; local queries use entity
neighborhoods.

**Core Algorithm:**
```
Ingest:
  for doc in corpus:
    entities, relations = LLM_extract(doc)
    G.add(entities, relations)
  communities = Leiden_detect(G)
  for c in communities:
    c.summary = LLM_summarize(c.entities + c.relations)

Query:
  if is_global_query:
    relevant = keyword_rank(query, community_summaries)
    answer = LLM(query + top_k_community_summaries)
  else:
    entities = embed_search(query, entity_index)
    context = neighbor_subgraph(entities, hops=2)
    answer = LLM(query + context)
```

**Implementation Insight:** Run community detection once at ingest time. The Leiden algorithm
(via `graspologic` or `igraph`) produces better communities than simple connected components.
Re-run summarization only when the graph changes significantly (>10% new nodes). Use
`text-embedding-3-large` for entity embeddings to maximize retrieval quality.

---

## 9. HippoRAG: Neurobiologically Inspired Long-Term Memory for Large Language Models

**Citation:** Guo, Z., et al. (2024).
*HippoRAG: Neurobiologically Inspired Long-Term Memory for Large Language Models.*
arXiv:2405.14831. NeurIPS 2024.

**Problem:** Standard RAG retrieval is "flat" — each query independently finds the most
similar chunks. This misses indirect connections and multi-hop reasoning.

**Contribution:** Model memory retrieval after hippocampal indexing: build a KG (like a
hippocampal memory index), identify seed entities from the query via embedding search, then
run Personalized PageRank (PPR) to propagate relevance through the KG and surface related
entities the query didn't mention directly.

**Core Algorithm (PPR):**
```
scores = uniform_init(n_entities)
seed_vector = {entity: 1/|seeds| for entity in query_entities}

for iteration in range(max_iter):
    scores = (1 - α) * A * scores + α * seed_vector
    if max_delta(scores) < tolerance: break

# A = column-normalized adjacency matrix (weighted by relation confidence)
# α = 0.15 (teleportation probability back to seeds)
# Typical: max_iter=20, tolerance=1e-6
```

**Key Results:** +20% on multi-hop QA (MuSiQue, 2WikiMultiHopQA) vs. standard dense retrieval.
+14% on PopQA. Particularly strong on questions requiring 3+ hops.

**Implementation Insight:** The bottleneck is building and maintaining the entity embedding
index. Use a fast ANN index (HNSW via USearch or Qdrant) for the initial seed entity lookup.
Pre-compute the column-normalized adjacency matrix at ingest time and store as a sparse CSR
matrix. α=0.15 works well across datasets; don't tune aggressively.

---

## 10. LightRAG: Simple and Fast Retrieval-Augmented Generation

**Citation:** Edge, D., et al. (2024).
*LightRAG: Simple and Fast Retrieval-Augmented Generation.*
arXiv:2410.05779.

**Problem:** GraphRAG is slow and expensive at query time. Practitioners need a simpler
KG-based RAG that works without heavy community pre-computation.

**Contribution:** Dual-level retrieval — (1) local mode: entity neighborhood lookup for
specific queries, (2) global mode: high-level theme retrieval via simple summary aggregation,
(3) hybrid mode: both combined. Faster than GraphRAG with comparable quality.

**Core Retrieval Modes:**
```
LOCAL mode:
  seeds = embed_search(query, entity_index, top_k=5)
  context = for each seed: entity + 1-hop neighbors + descriptions

GLOBAL mode:
  community_scores = keyword_overlap(query, community_summaries)
  context = top_3_communities.summaries

HYBRID mode:
  context = LOCAL_context + GLOBAL_context
```

**Implementation Insight:** The `lightrag-hku` Python package (`pip install lightrag-hku`)
provides a ready-to-use implementation with OpenAI/Ollama backends. Set `mode="hybrid"` for
best quality. For custom implementations, local retrieval is cheap (O(log n) with HNSW);
global retrieval needs pre-built community summaries.

---

## 11. PathRAG: Pruning Graph-Paths for Retrieval-Augmented Generation

**Citation:** Chen, B., et al. (2025).
*PathRAG: Pruning Graph-Paths for Retrieval-Augmented Generation.*
arXiv:2502.14902.

**Problem:** Entity neighborhood retrieval (local KG-RAG) includes many irrelevant nodes.
Path-based retrieval is more precise but naive BFS produces too many paths.

**Contribution:** Find multi-hop paths between query-relevant seed entities, then prune paths
using a flow-based scoring function that measures information density. Only top-scoring paths
are included in the context.

**Core Scoring:**
```
path_score(p) = Π(edge_weights along p) / len(p)

# Information flow analogy: product of weights models how strongly
# information "flows" from source to target through the path.
# Divide by length to penalize unnecessarily long paths.

prune_threshold = 0.3  # discard paths below this score
```

**Implementation Insight:** BFS with a visited set is sufficient for paths up to length 4.
For longer paths, use bidirectional BFS (start from both endpoints). The pruning threshold
should be tuned on a validation set; 0.2–0.4 works well empirically. Return at most 5–10
paths to avoid context bloat.

---

## 12. Self-RAG: Learning to Retrieve, Generate, and Critique Through Self-Reflection

**Citation:** Asai, A., Wu, Z., Wang, Y., Salehi, A., & Hajishirzi, H. (2023).
*Self-RAG: Learning to Retrieve, Generate, and Critique Through Self-Reflection.*
arXiv:2310.11511. ICLR 2024 (spotlight).

**Problem:** Standard RAG always retrieves even when retrieval is unnecessary (off-topic
documents degrade quality), and never checks if retrieved documents actually support the
generated text.

**Contribution:** Train the LLM to predict four special reflection tokens inline with generation:
- `[Retrieve]` — should retrieval happen at this point?
- `[IsRel]` — is the retrieved document relevant?
- `[IsSup]` — does the document support this claim?
- `[IsUse]` — is the generated passage useful for the query?

**Core Inference Logic:**
```
Generate segment s:
  if P([Retrieve]=YES | s) > threshold:
    docs = retrieve(query + s)
    for doc in docs:
      score_rel = P([IsRel]=RELEVANT | doc, s)
      score_sup = P([IsSup]=FULLY | doc, s)
      if score_rel * score_sup > discard_threshold:
        use doc as context
  score_use = P([IsUse]=YES | final_output)
```

**Key Results:** Outperforms ChatGPT and Llama2-70B on long-form generation, citation
accuracy (ASQA), and bio question answering, while being a much smaller model.

**Implementation Insight:** For production without fine-tuning, approximate Self-RAG by
prompting the LLM to evaluate retrieved documents before using them: "Does this document
directly answer the question? Rate 1-5." Gate on score ≥ 3 before including in context.
This is the CRAG approach (see paper 13).

---

## 13. CRAG: Corrective Retrieval Augmented Generation

**Citation:** Yan, S., et al. (2024).
*Corrective Retrieval Augmented Generation.*
arXiv:2401.15884. ICLR 2024.

**Problem:** RAG systems blindly trust retrieved documents even when they are irrelevant or
contradictory to the query.

**Contribution:** Add a lightweight evaluator that scores retrieved documents as Correct
(high confidence), Ambiguous (medium), or Incorrect (low confidence). The score determines
the fallback strategy.

**Core Decision Tree:**
```
for doc in retrieved_docs:
    score = confidence_evaluator(query, doc)  # 0.0 – 1.0

if max_score > 0.7:      # CORRECT
    context = refine_doc(best_doc)           # extract most relevant passages
elif max_score > 0.3:    # AMBIGUOUS
    context = best_doc + web_search(query)   # supplement with fresh retrieval
else:                    # INCORRECT
    context = web_search(query)              # discard local, go to web
```

**Key Results:** +10–15% on TriviaQA, PopQA; significant quality improvements on questions
requiring current information (where the knowledge base is stale).

**Implementation Insight:** The confidence evaluator can be a zero-shot LLM call:
"Does this document contain information that directly answers: {query}? Score 0-10."
Use `gpt-4o-mini` for cost efficiency. Cache evaluator scores keyed on (query_hash, doc_hash)
to avoid re-evaluation on identical query-doc pairs.

---

## 14. HyDE: Precise Zero-Shot Dense Retrieval without Relevance Labels

**Citation:** Gao, L., Ma, X., Lin, J., & Callan, J. (2022).
*Precise Zero-Shot Dense Retrieval without Relevance Labels.*
arXiv:2212.10496. ACL 2023.

**Problem:** Query embeddings often differ semantically from document embeddings (query-doc
mismatch), degrading retrieval precision especially for short queries.

**Contribution:** Instead of embedding the query directly, generate a Hypothetical Document
that would perfectly answer the query using an LLM. Embed the hypothetical document (not the
original query) and use it for retrieval. The hypothetical document is in the same distribution
as real documents.

**Core Algorithm:**
```
# Standard RAG
results = vector_search(embed(query), index)

# HyDE
hypothetical_doc = LLM(f"Write a document that answers: {query}")
results = vector_search(embed(hypothetical_doc), index)
```

**Key Results:** +3–8% on MS-MARCO, TREC-DL, and BEIR benchmarks vs. direct query embedding.
Most effective for short, ambiguous, or domain-specific queries.

**Implementation Insight:** Use a fast, cheap model (gpt-4o-mini) for hypothetical generation.
The hypothetical doc needs only 100–300 words — longer doesn't help. For multi-hop questions,
generate hypothetical docs for each sub-question separately. Optionally combine HyDE with
RAG-Fusion: generate multiple hypothetical docs and aggregate with RRF.

---

## 15. RAG-Fusion with Reciprocal Rank Fusion

**Citation:** (Various, 2024). RAG-Fusion methodology documented in arXiv:2402.03367.

**Problem:** A single query may not capture all relevant dimensions of the user's information
need. Different phrasings retrieve different (all valid) documents.

**Contribution:** Generate N rephrased versions of the original query, retrieve independently
for each, and merge ranked lists using Reciprocal Rank Fusion (RRF).

**RRF Formula:**
```
score(doc, D) = Σ_{q in queries} 1 / (rank_q(doc) + k)

where:
  rank_q(doc) = rank of doc in retrieval results for query q
  k = 60 (smoothing constant, standard value)
  D = set of all queries
```

**Example with 3 queries, k=60:**
```
Query 1: "climate change impacts"    → doc_A rank=1, doc_B rank=3
Query 2: "global warming effects"    → doc_A rank=2, doc_B rank=1
Query 3: "temperature rise outcomes" → doc_A rank=4, doc_B rank=2

doc_A score = 1/(1+60) + 1/(2+60) + 1/(4+60) = 0.01639 + 0.01613 + 0.01563 = 0.04815
doc_B score = 1/(3+60) + 1/(1+60) + 1/(2+60) = 0.01587 + 0.01639 + 0.01613 = 0.04839
```

**Implementation Insight:** Generate query variants with a prompt: "Generate 3 alternative
search queries for: {original_query}. Each should capture a different aspect or phrasing."
Use the same vector store; the overhead is just 2 extra embed+search calls. Works synergistically
with HyDE (generate hypothetical docs for each rephrased query).

---

## 16. ESAA: Event Sourcing Architectures for Autonomous Agents

**Citation:** (2026). arXiv:2602.23193.

**Problem:** Agent state managed as mutable in-memory objects is fragile: crashes lose
state, debugging is hard, and parallel execution causes race conditions.

**Contribution:** Apply Event Sourcing + CQRS to agent architectures. All state changes are
captured as immutable events in an append-only log. The current state is always derived by
replaying events. CQRS separates the write side (commands → events) from the read side
(event projections → views).

**Core Pattern:**
```
WRITE side (CommandHandler):
  AgentStarted(session_id, task, timestamp)
  ToolCalled(session_id, tool_name, args, timestamp)
  ToolSucceeded(session_id, tool_name, result, timestamp)
  LLMQueried(session_id, prompt_tokens, timestamp)
  AgentCompleted(session_id, result, timestamp)

All events → append-only JSONL log

READ side (QueryHandler):
  ProjectedState = replay(all_events_for_session)
  → current_node, messages, tool_results, status
```

**Implementation Insight:** Store events as JSONL (one JSON object per line). Each event needs:
`id` (UUID), `session_id`, `event_type`, `timestamp`, `payload`. Replay is O(n) in event
count — use snapshots every 1000 events for long-running sessions. The CQRS query side can
maintain materialized views (e.g., a `sessions` table with current status) updated by event
handlers.

---

## 17. BGE-M3: Multi-Linguality, Multi-Functionality, Multi-Granularity

**Citation:** Chen, J., Xiao, S., Zhang, P., Luo, K., Lian, D., & Liu, Z. (2024).
*BGE M3-Embedding: Multi-Linguality, Multi-Functionality, Multi-Granularity Text Embeddings
Through Self-Knowledge Distillation.*
arXiv:2402.03216. ACL 2024.

**Problem:** Production RAG requires (1) multilingual support, (2) dense + sparse retrieval,
and (3) multi-vector (ColBERT-style late interaction) — typically requiring three separate
models.

**Contribution:** A single model that simultaneously supports: dense retrieval (single-vector
cosine similarity), sparse retrieval (BM25-style learned lexical weights), and multi-vector
retrieval (ColBERT late interaction). 100+ languages, 8192-token input.

**Core Architecture:**
```
Input text → XLM-RoBERTa encoder
                ↓
    ┌───────────┼───────────┐
Dense head   Sparse head   ColBERT head
(CLS token)  (token weights) (all token vecs)
    ↓              ↓              ↓
L2-norm      Relu+norm       Stack → MaxSim
```

**Retrieval Fusion (hybrid score):**
```
score = w_dense * cosine(q_dense, d_dense)
      + w_sparse * dot(q_sparse, d_sparse)
      + w_colbert * MaxSim(q_colbert, d_colbert)

Default weights: (0.4, 0.2, 0.4)
```

**Implementation Insight:** Use via `FlagEmbedding` package (`pip install FlagEmbedding`).
For Qdrant, use named vectors: `{"dense": [...], "sparse": {...}}`. BGE-M3 is particularly
valuable for mixed-language corpora or when keyword precision matters (e.g., code identifiers,
proper nouns). For English-only, `text-embedding-3-large` is faster; for multilingual, BGE-M3
is the best open-weight option.

---

## 18. ExpeL: LLM Agents Are Experiential Learners

**Citation:** Zhao, A., Huang, D., Xu, Q., Lin, M., Liu, Y. J., & Huang, G. (2023).
*ExpeL: LLM Agents Are Experiential Learners.*
arXiv:2308.10144. AAAI 2024.

**Problem:** LLM agents make the same class of mistakes repeatedly across different tasks
because they don't learn from experience within a deployment lifetime.

**Contribution:** Maintain an experience pool of successful trajectories (stored in a vector
DB) and extract cross-trajectory insights using an LLM. When a new task arrives, retrieve
similar past trajectories + insights and prepend them as context.

**Core Components:**
```
Experience Pool:
  trajectories: VectorDB<(task, steps, score, tools_used)>
  insights: List<(content, support_count, confidence)>

At task completion (if score >= quality_threshold):
  pool.add(trajectory)
  insights = LLM_extract_insights(pool.sample(10))

At new task start:
  similar = pool.search(task_embedding, top_k=3)
  context = format(insights + similar_trajectories)
  agent.run(task, extra_context=context)
```

**Key Results:** +8–15% success rate on WebArena and ALFWorld vs. zero-shot baselines, with
no fine-tuning. Performance improves as the experience pool grows.

**Implementation Insight:** Use a quality threshold (score ≥ 0.6) before storing trajectories
to avoid polluting the pool with bad examples. Run insight extraction asynchronously (it's
expensive). For insight extraction prompt: "Given these N successful trajectories, what
generalizable patterns or best practices apply across tasks? Return as a list."

---

## 19. SELF-REFINE: Iterative Refinement with Self-Feedback

**Citation:** Madaan, A., Tandon, N., Gupta, P., Hallinan, S., Gao, L., Wiegreffe, S.,
Alon, U., et al. (2023). *SELF-REFINE: Iterative Refinement with Self-Feedback.*
NeurIPS 2023. Website: selfrefine.info.

**Problem:** LLM first-draft outputs are often mediocre — the model doesn't naturally refine
its own outputs without explicit prompting.

**Contribution:** A generate → critique → refine loop using the same LLM. The critique step
uses a separate prompt that evaluates the draft; the refinement step uses the critique to
produce an improved version. Loop terminates when critique says output is acceptable or max
iterations reached.

**Core Loop:**
```
draft = LLM(generate_prompt + task)
for iteration in range(max_iterations):
    critique = LLM(critique_prompt + task + draft)
    if "SATISFACTORY" in critique or "no issues" in critique:
        break
    draft = LLM(refine_prompt + task + draft + critique)
return draft
```

**Key Results:** 15–20% improvement across code generation, math reasoning, sentiment steering,
and constrained generation tasks.

**Implementation Insight:** The critique prompt is the most important component. It should be
specific about evaluation dimensions (correctness, completeness, clarity, edge cases). Use
`gpt-4o` for critique and `gpt-4o-mini` for refinement to balance quality and cost. Cap at
3–5 iterations — diminishing returns after that. For code tasks, include compiler/test output
in the critique.

---

## 20. CRITIC: LLMs Can Self-Correct with Tool-Interactive Critiquing

**Citation:** Gou, Z., et al. (2023).
*CRITIC: Large Language Models Can Self-Correct with Tool-Interactive Critiquing.*
arXiv:2310.06825. ICLR 2024.

**Problem:** LLMs generate confident-sounding but factually wrong outputs. SELF-REFINE-style
critiques without external grounding still hallucinate corrections.

**Contribution:** Ground the critique in tool outputs. After generation, extract verifiable
claims, use tools (web search, calculator, code executor) to check them, and use the tool
output as evidence for corrections.

**Core Loop:**
```
output = LLM(task)
claims = LLM_extract(f"Extract verifiable claims from: {output}")
for claim in claims:
    tool = LLM_choose_tool(claim, available_tools)
    evidence = execute(tool, claim)
    verified, correction = LLM_judge(claim, evidence)
    if not verified:
        output = LLM_correct(output, correction, evidence)
return output
```

**Key Results:** CRITIC reduces factual error rate by 20–40% on TriviaQA, HotpotQA, and
GSM8K (math reasoning) compared to baseline LLM.

**Implementation Insight:** For code agents, the natural tool is a code interpreter — execute
the generated code and use stdout/stderr as the critique evidence. For general text, use web
search. Limit to 3–5 claims per output to control latency. The judge prompt should output
structured JSON: `{"verified": bool, "confidence": 0-1, "correction": "..."}`.

---

## 21. AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation

**Citation:** Wu, Q., Bansal, G., Zhang, J., Wu, Y., Zhang, S., Zhu, E., Li, B.,
Jiang, L., Zhang, X., & Wang, C. (2023).
*AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation Framework.*
arXiv:2308.08155. ICLR 2024.

**Problem:** Single-agent LLMs struggle with complex multi-step tasks that benefit from
specialized roles, parallel execution, and inter-agent critique.

**Contribution:** An actor-model framework where agents are typed message handlers
communicating via a publish-subscribe message bus. Agents can have different tools, personas,
and termination conditions. The framework supports direct agent-to-agent messaging and
broadcast topics.

**Core Architecture (AutoGen v0.4):**
```
AgentRuntime (actor model)
  ├── TopicSubscription registry
  ├── DirectMessageRouter
  └── Agents:
       AssistantAgent(name, tools, system_prompt)
       UserProxyAgent(human_input_mode="NEVER")
       GroupChat(agents, max_round, speaker_selection)

Message flow:
  agent.publish_message(topic, msg)   → broadcast
  agent.send_message(target, msg)     → direct
  await agent.receive_message()       → consume
```

**Key Results:** Complex tasks (coding + testing + debugging) solved with 3–5 agent
collaboration in under 30% of the tokens vs. single-agent approaches.

**Implementation Insight:** For LangGraph implementations, use `create_supervisor` from
`langgraph-supervisor` as the equivalent pattern. For Koog, use `nodeA2AClientSendMessage`
for inter-agent communication. The key design decision is whether agents share a message
history (transparent) or only receive task-specific messages (isolated).

---

## 22. MetaGPT: Meta Programming for a Multi-Agent Collaborative Framework

**Citation:** Hong, S., et al. (2023).
*MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework.*
arXiv:2308.00352. ICLR 2024 (oral).

**Problem:** Multi-agent systems often degenerate into circular discussions without producing
artifacts. Agents need structured roles and validation gates.

**Contribution:** Standard Operating Procedures (SOPs) that enforce artifact-based handoffs:
each agent produces a structured document (PRD, architecture spec, code, test results)
validated before the next agent starts. Agents have roles (ProductManager, Architect,
Engineer, QAEngineer) and a shared message pool.

**SOP Pipeline:**
```
User story
    ↓ ProductManager
PRD document (validated JSON)
    ↓ Architect
System design (module diagram, tech choices)
    ↓ Engineer
Code files (validated by linter)
    ↓ QAEngineer
Test results (validated pass/fail)
    ↓ (if fail → back to Engineer)
```

**Key Results:** Reduces hallucination rate in multi-step software development by 73% vs.
unstructured multi-agent chat. Produces complete, runnable software projects from natural
language specs.

**Implementation Insight:** The key innovation is artifact validation — don't pass raw LLM
output between agents; validate it first (JSON schema, linter, test execution). Implement
as a directed acyclic graph where each node is a (role, tool_set, output_schema) triple.
For LangGraph: each MetaGPT role is a node with a Pydantic output model as the state update.

---

## 23. StreamingLLM: Efficient Streaming Language Models with Attention Sinks

**Citation:** Xiao, G., Tian, Y., Chen, B., Han, S., & Lewis, M. (2023).
*Efficient Streaming Language Models with Attention Sinks.*
arXiv:2309.17453. ICLR 2024.

**Problem:** Transformer KV caches grow linearly with sequence length. For long-running
agents with many tool calls, the cache exceeds GPU memory and generation slows to a crawl.

**Contribution:** Observation that LLMs attend disproportionately to the first 4–8 tokens
(attention sinks) and the most recent tokens. By keeping a fixed window of recent tokens
plus the initial attention sinks, the KV cache size is bounded and generation speed is
constant regardless of sequence length.

**Core Algorithm:**
```
KV_cache = [sink_tokens (4–8)] + [sliding_window (recent 1000–4000 tokens)]

When new token is generated:
  if len(KV_cache) > max_size:
    drop oldest non-sink tokens
  append new KV to cache

# sink_tokens = first 4 tokens (typically BOS and first few content tokens)
# These attract "sink attention" and stabilize generation even if far from current position
```

**Key Results:** 22.2× throughput improvement vs. naive KV cache. Enables infinite-length
generation with constant memory footprint. <1% quality degradation on perplexity benchmarks.

**Implementation Insight:** Relevant for local LLM deployments (llama.cpp, vLLM) or custom
inference servers. Most cloud LLM APIs handle this server-side. For C++ llama.cpp: use
`n_ctx` as the sliding window size and `--no-kv-offload` for best throughput. The sink
token concept explains why prepending a brief "attention anchor" (like system prompt
keywords) to long conversations can stabilize generation.

---

## 24. LongMemEval: Benchmarking Chat Assistants on Long-Term Interactive Memory

**Citation:** Wu, D., et al. (2024).
*LongMemEval: Benchmarking Chat Assistants on Long-Term Interactive Memory.*
arXiv:2410.10813. NeurIPS 2024.

**Problem:** Commercial AI assistants (ChatGPT, Claude, Gemini) show 30%+ accuracy drops
on questions requiring retrieval of information from earlier in long conversations.

**Contribution:** A benchmark of 500 questions requiring memory across 1–15 session turns.
Key finding: decomposing sessions into atomic facts (one claim per sentence) and indexing
them separately dramatically improves retrieval vs. full-sentence or paragraph indexing.

**Core Finding:**
```
# NAIVE: index full conversation turns
results = vector_search("What is Alice's job?", turn_index)
# → 30% accuracy (relevant info buried in long turns)

# ATOMIC FACTS: decompose first
facts = LLM_decompose(turn)
# "Alice works at Google" → indexed separately from "Alice likes hiking"
results = vector_search("What is Alice's job?", fact_index)
# → 64% accuracy (+34 points)
```

**Atomic Fact Extraction Prompt:**
```
Split the following text into atomic facts.
An atomic fact is a single, self-contained, verifiable statement.
Output one fact per line, starting with "FACT: ".
Do not include opinions or uncertain statements.
```

**Commercial Assistant Results:** GPT-4o: 71% → 94% with atomic fact RAG.
Claude 3.5: 68% → 93%. Baseline without memory: 61%.

**Implementation Insight:** Run atomic decomposition asynchronously after each conversation
turn. Store the source turn ID as metadata with each fact for provenance. At retrieval time,
use hybrid search (dense + BM25) on the fact index. Merge facts from the same source turn
to avoid redundant context. This is more important than the embedding model choice — the
same model performs much better with atomic facts vs. full turns.

---

## Summary: Which Paper for Which Problem?

| Problem | Recommended Paper(s) |
|---------|---------------------|
| Agent won't use tools correctly | ReAct (2210.03629) |
| Agent repeats the same mistakes | Reflexion (2303.11366) |
| Context window fills up too fast | MemGPT (2310.08560), StreamingLLM (2309.17453) |
| Agent forgets cross-session info | MemGPT (2310.08560), Mem0 (2504.19413), A-MEM (2502.12110) |
| RAG retrieves irrelevant chunks | CRAG (2401.15884), Self-RAG (2310.11511), HyDE (2212.10496) |
| Need global document understanding | GraphRAG (2404.16130), LightRAG (2410.05779) |
| Multi-hop Q&A fails | HippoRAG (2405.14831), PathRAG (2502.14902) |
| Agent generates factual errors | CRITIC (2310.06825), SELF-REFINE (selfrefine.info) |
| Want to learn from past runs | ExpeL (2308.10144), Reflexion (2303.11366) |
| Multi-agent coordination | AutoGen (2308.08155), MetaGPT (2308.00352) |
| Structured software development | MetaGPT (2308.00352) |
| Crash recovery / audit trail | ESAA (2602.23193), OpenHands (2407.16741) |
| Multilingual retrieval | BGE-M3 (2402.03216) |
| Long conversation memory accuracy | LongMemEval (2410.10813) → atomic facts |
| Query coverage improvement | RAG-Fusion (2402.03367) |
| Memory taxonomy design | CoALA (2309.02427) |
