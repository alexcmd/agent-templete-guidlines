# Universal Agent Architecture — Memory, RAG & Knowledge Graphs

> Sections: Memory Architecture (CoALA) · RAG Pipeline Stages · Knowledge Graph Architecture

Part of the [Universal Agent Architecture](01-overview.md) series.

## 7. Memory Architecture (CoALA Taxonomy)

All modern agent memory systems organize into four stores (arXiv:2309.02427):

```
┌──────────────────────────────────────────────────────────────────┐
│  WORKING MEMORY (in-context)                                      │
│  Current conversation messages, active tool results               │
│  Capacity: ~200K tokens (Claude 3.x), managed by compaction      │
├──────────────────────────────────────────────────────────────────┤
│  EPISODIC MEMORY (external)                                       │
│  Past conversation summaries, trajectory records                  │
│  Backend: vector DB (Qdrant / USearch / pgvector)                │
│  Retrieval: top-k ANN by embedding similarity                    │
├──────────────────────────────────────────────────────────────────┤
│  SEMANTIC MEMORY (external)                                       │
│  Facts, entity relationships, knowledge graph                     │
│  Backend: graph DB (Neo4j / SQLite with adjacency table)          │
│  Retrieval: HippoRAG PPR or GraphRAG community summaries          │
├──────────────────────────────────────────────────────────────────┤
│  PROCEDURAL MEMORY (in-weights + tool library)                    │
│  LLM base knowledge + learned skills (ExpeL / Voyager)           │
│  Backend: in-weights (fine-tuning) or vector-indexed skill store  │
└──────────────────────────────────────────────────────────────────┘
```

**MemGPT 3-tier mapping** (arXiv:2310.08560) onto the CoALA taxonomy:

| MemGPT tier | CoALA category | Persistence |
|-------------|----------------|-------------|
| Main context | Working | In-context, lost on compaction |
| Recall storage | Episodic | Vector DB, persists across sessions |
| Archival storage | Semantic | Vector DB, long-term fact store |

### File-Based Memory (memdir pattern)

For agents without a vector DB dependency, structured markdown files provide sufficient
episodic and semantic memory at zero infrastructure cost:

```
~/.agent/memory/
├── user_role.md            (type: user)
├── feedback_testing.md     (type: feedback)
├── project_auth_rewrite.md (type: project)
└── reference_linear.md     (type: reference)
```

Each file carries YAML frontmatter:
```yaml
---
name: "Integration tests must hit real DB"
description: "Test strategy constraint from Q1 incident"
type: feedback
---
Do not mock the database. Prior incident: mock tests passed but prod migration failed.
**Why:** divergence between mock and prod masked a broken migration
**How to apply:** always run integration tests against a real database instance
```

**MEMORY.md index format:**
```markdown
- [Integration tests must hit real DB](feedback_testing.md) — test strategy from Q1 incident
- [User prefers terse responses](user_comms.md) — no trailing summaries
- [Auth rewrite is compliance-driven](project_auth.md) — legal constraint, not tech debt
```

Rules: ≤ 200 entries, one entry ≤ 150 chars. Lines beyond 200 are truncated at injection.
The model reads the index and fetches specific files only when relevant — not all files load
into context on every turn.

**Memory file discovery (upward walk):**
```python
def discover_memory_files(cwd: str, max_files: int = 200) -> list[Path]:
    """Walk upward from cwd collecting memory files, newest-first within each dir."""
    found = []; seen_inodes = set(); path = Path(os.path.realpath(cwd))
    while True:
        mem_dir = path / ".agent" / "memory"
        if mem_dir.is_dir():
            files = sorted(mem_dir.glob("*.md"), key=lambda f: f.stat().st_mtime, reverse=True)
            for f in files:
                inode = f.stat().st_ino
                if inode not in seen_inodes:
                    seen_inodes.add(inode); found.append(f)
                    if len(found) >= max_files: return found
        parent = path.parent
        if parent == path: break
        path = parent
    return found
```

**Memory staleness caveat:** Inject a note when memory files are >24h old so the model
doesn't treat stale information as current:
```python
def memory_freshness_text(mtime: float) -> str:
    age_hours = (time.time() - mtime) / 3600
    if age_hours < 24: return ""
    if age_hours < 168: return f"(written {int(age_hours)}h ago — verify before acting)"
    return f"(written {int(age_hours/24)}d ago — likely stale, verify)"
```

### Session Memory Extraction

After sessions with ≥ 20 messages (or at compaction time), fire a structured API call to
extract atomic facts and append them to the user/project memory files automatically:

```
extraction_categories:
  - UserPreference   (how this user likes to work)
  - ProjectFact      (codebase truths: "uses monorepo", "tests in /test/")
  - CodePattern      (idioms: "all handlers return Result<(), AppError>")
  - Decision         ("chose SQLite over Postgres for simplicity")
  - Constraint       ("never DELETE without explicit user confirmation")

trigger:
  - message_count >= 20
  - OR compact event fires
  - track last_extracted_message_id to avoid re-extraction
```

### AutoDream (Background Consolidation)

A background daemon synthesizes memories across multiple sessions into a consolidated
knowledge base. Run it as a separate low-priority process or scheduled task:

```
Trigger conditions (AND):
  - >= 24 hours since last consolidation
  - >= 5 new sessions since last consolidation
  - No concurrent consolidation (lock file check)

Process:
  1. Read all session summaries from past N sessions
  2. Fire a dedicated consolidation prompt (uses cheaper model)
  3. Extract cross-session patterns and update memory files
  4. Update consolidation_state.json with timestamp + session count
  5. Release lock

Circuit breaker:
  - If lock file is stale (>2h), remove and proceed
  - If consolidation fails 3× in 24h, skip until next day
```

### Away Summary (Re-entry Recap)

When the user returns to an idle session (no activity for > T minutes), generate a 1-3
sentence recap of the current work state using a cheaper/faster model before the user's
next prompt is processed:

```
if time_since_last_message > AWAY_THRESHOLD:
    summary = cheaper_model.complete(
        "Summarize in 1-3 sentences what was being worked on and current status."
        + context_from_last_N_messages
    )
    display_as_status_message(summary)
    # do NOT add to message history — it's informational only
```

---

## 8. RAG Pipeline Stages

Every RAG implementation must traverse the same five stages. The paper references indicate
which algorithm to apply at each stage:

```
Query
  │
  ▼ [1] QUERY TRANSFORMATION
  │     HyDE (arXiv:2212.10496): generate hypothetical answer → embed that
  │     RAG-Fusion (arXiv:2402.03367): generate N sub-queries → RRF merge
  │
  ▼ [2] RETRIEVAL
  │     Dense: embedding ANN (HNSW / Qdrant)
  │     Sparse: BM25 keyword
  │     Hybrid: BGE-M3 (arXiv:2402.03216) unified dense+sparse+multi-vector
  │
  ▼ [3] RELEVANCE FILTERING
  │     CRAG (arXiv:2401.15884): score each chunk Correct/Ambiguous/Incorrect
  │       Correct  → use chunk as-is
  │       Ambiguous → web search to augment
  │       Incorrect → discard, fallback to web search
  │
  ▼ [4] CONTEXT ASSEMBLY
  │     Rerank top-k by cross-encoder or MMR
  │     Trim to token budget
  │
  ▼ [5] GENERATION
        LLM generates answer conditioned on retrieved context
        Self-RAG (arXiv:2310.11511): emit reflection tokens
          [Retrieve] [IsRel] [IsSup] [IsUse] inline during generation
```

---

## 9. Knowledge Graph Architecture

When semantic memory requires structured relationship traversal:

```
┌──────────────────────────────────────────────────────────────────┐
│  EXTRACTION                                                        │
│  LLM triple extractor: sentence → (subject, predicate, object)   │
│  Entity resolver: coreference + deduplication                     │
├──────────────────────────────────────────────────────────────────┤
│  STORAGE                                                           │
│  Small-scale: SQLite adjacency table (entities + edges)           │
│  Production: Neo4j with Cypher                                    │
├──────────────────────────────────────────────────────────────────┤
│  RETRIEVAL — choose by query type:                                │
│                                                                    │
│  Local entity queries → HippoRAG (arXiv:2405.14831)              │
│    seed = top-k ANN of query embedding                           │
│    PPR(seed, damping=0.85) over entity graph                     │
│    return top-m entities by final PPR score                       │
│                                                                    │
│  Global thematic queries → GraphRAG (arXiv:2404.16130)           │
│    Leiden community detection on entity graph                     │
│    LLM generates community summary for each cluster              │
│    retrieve relevant community summaries by query                │
│                                                                    │
│  Path-constrained queries → PathRAG (arXiv:2502.14902)           │
│    enumerate relevant paths, score by flow capacity              │
│    prune low-flow paths before LLM context injection             │
│                                                                    │
│  Dual-level hybrid → LightRAG (arXiv:2410.05779)                 │
│    local mode: entity-level ANN                                   │
│    global mode: community summaries                               │
│    hybrid mode: both, merged                                      │
└──────────────────────────────────────────────────────────────────┘
```

---


---

*[← System Prompt & Permissions](01-prompts-permissions.md) | [Next: Event-Driven & Multi-Agent →](01-event-driven-multiagent.md)*
