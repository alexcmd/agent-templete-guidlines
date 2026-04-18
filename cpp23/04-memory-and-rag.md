# 04 — Memory Systems and RAG Pipeline

This document covers the complete memory architecture (MemGPT 3-tier model,
A-MEM note graph, StreamingLLM) and the modular RAG pipeline
(HyDE, RAG-Fusion, CRAG, Self-RAG, parent-document retrieval,
contextual compression, LongMemEval atomic facts).

---

## 1. MemGPT 3-Tier Memory Architecture (arXiv:2310.08560)

```
┌───────────────────────────────────────────┐
│   Tier 1: WorkingMemory (in-context)       │
│   Hot facts, current task state,           │
│   recent messages — always in LLM context  │
│   Capacity: ~4K tokens                     │
└─────────────────┬─────────────────────────┘
                  │ overflow / retrieval
┌─────────────────▼─────────────────────────┐
│   Tier 2: RecallStorage (USearch HNSW)     │
│   Recent episodic memory,                  │
│   conversation summaries, observations     │
│   Capacity: ~100K entries                  │
└─────────────────┬─────────────────────────┘
                  │ deep retrieval
┌─────────────────▼─────────────────────────┐
│   Tier 3: ArchivalStorage (USearch+SQLite) │
│   Long-term knowledge, documents,          │
│   indexed codebases, previous sessions     │
│   Capacity: millions of entries            │
└───────────────────────────────────────────┘
```

### WorkingMemory

```cpp
// src/memory/working_memory.hpp
#pragma once
#include <string>
#include <vector>
#include <deque>
#include "agent/agent_context.hpp"

namespace memory {

struct WorkingMemorySlot {
    std::string key;
    std::string value;
    int         priority = 0;   // higher = evict last
    int         token_estimate = 0;
};

class WorkingMemory {
public:
    explicit WorkingMemory(int max_tokens = 4096);

    void upsert(std::string key, std::string value, int priority = 0);
    std::optional<std::string> get(const std::string& key) const;
    void remove(const std::string& key);

    // Render as text block for injection into system prompt
    std::string render() const;

    // Evict lowest-priority entries until under token limit
    void evict_to_fit();

    int current_tokens() const;

private:
    int max_tokens_;
    std::vector<WorkingMemorySlot> slots_;
};

} // namespace memory
```

```cpp
// src/memory/working_memory.cpp
#include "memory/working_memory.hpp"
#include <algorithm>
#include <format>

namespace memory {

WorkingMemory::WorkingMemory(int max_tokens) : max_tokens_(max_tokens) {}

void WorkingMemory::upsert(std::string key, std::string value, int priority) {
    for (auto& s : slots_) {
        if (s.key == key) {
            s.value = std::move(value);
            s.priority = priority;
            s.token_estimate = static_cast<int>(s.value.size() / 4);
            evict_to_fit();
            return;
        }
    }
    int est = static_cast<int>(value.size() / 4) + 4;
    slots_.push_back({ std::move(key), std::move(value), priority, est });
    evict_to_fit();
}

std::optional<std::string> WorkingMemory::get(const std::string& key) const {
    for (const auto& s : slots_)
        if (s.key == key) return s.value;
    return std::nullopt;
}

void WorkingMemory::remove(const std::string& key) {
    slots_.erase(
        std::remove_if(slots_.begin(), slots_.end(),
            [&key](const auto& s) { return s.key == key; }),
        slots_.end());
}

std::string WorkingMemory::render() const {
    if (slots_.empty()) return "";
    std::string out = "[Working Memory]\n";
    for (const auto& s : slots_)
        out += std::format("  {}: {}\n", s.key, s.value);
    return out;
}

int WorkingMemory::current_tokens() const {
    int total = 0;
    for (const auto& s : slots_) total += s.token_estimate;
    return total;
}

void WorkingMemory::evict_to_fit() {
    // Sort ascending by priority (evict lowest first)
    std::stable_sort(slots_.begin(), slots_.end(),
        [](const auto& a, const auto& b) { return a.priority < b.priority; });
    while (current_tokens() > max_tokens_ && !slots_.empty())
        slots_.erase(slots_.begin());
}

} // namespace memory
```

---

## 2. USearch HNSW Integration

USearch provides a single-header HNSW implementation with 10x better recall@10
throughput than FAISS on typical embedding workloads.

```cpp
// src/memory/recall_storage.hpp
#pragma once
#include <string>
#include <vector>
#include <cstdint>
#include <expected>
#include <usearch/index.hpp>
#include <nlohmann/json.hpp>
#include "agent/agent_error.hpp"

namespace memory {

struct MemoryEntry {
    uint64_t    id;
    std::string text;
    nlohmann::json metadata;
    float       score = 0.0f;  // populated during search
};

class RecallStorage {
public:
    explicit RecallStorage(std::size_t dimensions, std::size_t capacity = 100'000);

    // Add a vector with associated text/metadata
    AgentResult<uint64_t> add(const std::vector<float>& embedding,
                               std::string text,
                               nlohmann::json metadata = {});

    // Approximate nearest neighbor search
    AgentResult<std::vector<MemoryEntry>> search(
        const std::vector<float>& query_embedding,
        std::size_t top_k = 10) const;

    // Persist index to disk
    AgentResult<void> save(const std::string& path) const;
    AgentResult<void> load(const std::string& path);

    std::size_t size() const;

private:
    using index_t = unum::usearch::index_dense_t<uint64_t, float>;
    index_t     index_;
    std::size_t dimensions_;
    uint64_t    next_id_ = 0;

    // Text/metadata store alongside the vector index
    // In production, use SQLite; here flat_map for simplicity
    std::flat_map<uint64_t, MemoryEntry> store_;
};

} // namespace memory
```

```cpp
// src/memory/recall_storage.cpp
#include "memory/recall_storage.hpp"
#include <spdlog/spdlog.h>

namespace memory {

RecallStorage::RecallStorage(std::size_t dims, std::size_t capacity)
    : dimensions_(dims)
{
    unum::usearch::index_dense_config_t cfg;
    cfg.connectivity         = 16;   // HNSW M parameter
    cfg.expansion_add        = 200;  // ef_construction
    cfg.expansion_search     = 100;  // ef_search
    index_ = index_t::make(
        unum::usearch::metric_punned_t(dims,
            unum::usearch::metric_kind_t::cos_k,  // cosine similarity
            unum::usearch::scalar_kind_t::f32_k),
        cfg);
    index_.reserve(capacity);
}

AgentResult<uint64_t> RecallStorage::add(
    const std::vector<float>& emb,
    std::string text,
    nlohmann::json meta)
{
    if (emb.size() != dimensions_) {
        return std::unexpected(AgentError{
            AgentErrorCode::VectorSearchFailed,
            std::format("Embedding dim mismatch: expected {}, got {}", dimensions_, emb.size())
        });
    }

    uint64_t id = next_id_++;
    index_.add(id, emb.data());
    store_[id] = MemoryEntry{ id, std::move(text), std::move(meta), 0.0f };
    return id;
}

AgentResult<std::vector<MemoryEntry>> RecallStorage::search(
    const std::vector<float>& query,
    std::size_t top_k) const
{
    if (query.size() != dimensions_) {
        return std::unexpected(AgentError{
            AgentErrorCode::VectorSearchFailed,
            "Query embedding dimension mismatch"
        });
    }

    auto results = index_.search(query.data(), top_k);
    std::vector<MemoryEntry> entries;
    entries.reserve(results.size());

    for (std::size_t i = 0; i < results.size(); ++i) {
        auto it = store_.find(results[i].member.key);
        if (it == store_.end()) continue;
        MemoryEntry e = it->second;
        e.score = results[i].distance;  // cosine distance (lower = closer)
        entries.push_back(std::move(e));
    }
    return entries;
}

AgentResult<void> RecallStorage::save(const std::string& path) const {
    index_.save(path.c_str());
    return {};
}

AgentResult<void> RecallStorage::load(const std::string& path) {
    index_.load(path.c_str());
    return {};
}

std::size_t RecallStorage::size() const { return index_.size(); }

} // namespace memory
```

---

## 3. ArchivalStorage (USearch + SQLite)

ArchivalStorage combines the USearch vector index for similarity search with
a SQLite table for full-text metadata, timestamps, and provenance.

```cpp
// src/memory/archival_storage.hpp
#pragma once
#include <string>
#include <vector>
#include <SQLiteCpp/SQLiteCpp.h>
#include "memory/recall_storage.hpp"

namespace memory {

class ArchivalStorage {
public:
    ArchivalStorage(const std::string& db_path,
                    const std::string& index_path,
                    std::size_t embedding_dims);

    AgentResult<uint64_t> insert(
        const std::string& text,
        const std::vector<float>& embedding,
        const nlohmann::json& metadata = {});

    AgentResult<std::vector<MemoryEntry>> search_semantic(
        const std::vector<float>& query,
        std::size_t top_k = 20) const;

    // Full-text search via SQLite FTS5
    AgentResult<std::vector<MemoryEntry>> search_fulltext(
        const std::string& query,
        std::size_t limit = 20) const;

    // Hybrid: merge semantic + full-text results (RRF scoring)
    AgentResult<std::vector<MemoryEntry>> search_hybrid(
        const std::vector<float>& query_embedding,
        const std::string& query_text,
        std::size_t top_k = 10) const;

    AgentResult<void> save() const;

private:
    SQLite::Database db_;
    RecallStorage    vector_index_;

    void init_schema();
    static std::vector<MemoryEntry> rrf_merge(
        std::vector<MemoryEntry> semantic,
        std::vector<MemoryEntry> lexical,
        int k = 60);
};

} // namespace memory
```

---

## 4. A-MEM Note Graph (arXiv:2502.12110)

A-MEM implements a Zettelkasten-style note graph where memories are linked
by semantic similarity. Each note includes Ebbinghaus decay tracking so
frequently accessed notes remain more salient.

```cpp
// src/memory/amem.hpp
#pragma once
#include <string>
#include <vector>
#include <chrono>
#include <cmath>
#include <nlohmann/json.hpp>
#include "memory/recall_storage.hpp"

namespace memory {

struct MemoryNote {
    uint64_t    id;
    std::string content;
    std::string context;          // source context
    std::vector<std::string> tags;
    std::vector<uint64_t>    linked_ids;  // bidirectional links

    // Ebbinghaus decay: R = e^(-t/S)
    // R = retention strength [0,1]
    // t = elapsed time since last access (seconds)
    // S = stability (increases with each retrieval)
    double   stability      = 1.0;
    std::chrono::system_clock::time_point last_accessed;
    int      access_count   = 0;

    double retention_at(std::chrono::system_clock::time_point now) const {
        auto elapsed = std::chrono::duration<double>(now - last_accessed).count();
        return std::exp(-elapsed / std::max(stability, 1.0));
    }
};

class AMem {
public:
    explicit AMem(RecallStorage& recall,
                  std::function<std::vector<float>(const std::string&)> embedder);

    // Store a new note, automatically linking to similar existing notes
    AgentResult<uint64_t> store(std::string content,
                                std::string context = "",
                                std::vector<std::string> tags = {});

    // Retrieve notes by semantic similarity, boosted by retention
    AgentResult<std::vector<MemoryNote>> retrieve(
        const std::string& query,
        std::size_t top_k = 5);

    // Access a note — updates retention/stability
    void access(uint64_t id);

    // Prune notes with retention below threshold
    void prune(double min_retention = 0.1);

    // Generate links between a new note and existing similar notes
    std::vector<uint64_t> generate_links(
        const std::vector<MemoryEntry>& similar,
        float link_threshold = 0.75f);

private:
    RecallStorage& recall_;
    std::function<std::vector<float>(const std::string&)> embedder_;
    std::flat_map<uint64_t, MemoryNote> notes_;
    float link_threshold_ = 0.75f;
};

} // namespace memory
```

```cpp
// src/memory/amem.cpp
#include "memory/amem.hpp"

namespace memory {

AMem::AMem(RecallStorage& recall,
            std::function<std::vector<float>(const std::string&)> embedder)
    : recall_(recall), embedder_(std::move(embedder)) {}

AgentResult<uint64_t> AMem::store(std::string content,
                                    std::string context,
                                    std::vector<std::string> tags)
{
    auto embedding = embedder_(content);
    auto add_result = recall_.add(embedding, content, { {"context", context} });
    if (!add_result) return std::unexpected(add_result.error());
    uint64_t id = *add_result;

    MemoryNote note;
    note.id           = id;
    note.content      = std::move(content);
    note.context      = std::move(context);
    note.tags         = std::move(tags);
    note.last_accessed = std::chrono::system_clock::now();

    // Find similar notes to link to
    auto similar = recall_.search(embedding, 10);
    if (similar) {
        note.linked_ids = generate_links(*similar, link_threshold_);
        // Add back-links to linked notes
        for (auto lid : note.linked_ids) {
            if (auto it = notes_.find(lid); it != notes_.end())
                it->second.linked_ids.push_back(id);
        }
    }

    notes_[id] = std::move(note);
    return id;
}

AgentResult<std::vector<MemoryNote>> AMem::retrieve(
    const std::string& query, std::size_t top_k)
{
    auto embedding = embedder_(query);
    auto raw = recall_.search(embedding, top_k * 2);
    if (!raw) return std::unexpected(raw.error());

    auto now = std::chrono::system_clock::now();
    std::vector<std::pair<double, MemoryNote*>> scored;

    for (const auto& entry : *raw) {
        auto it = notes_.find(entry.id);
        if (it == notes_.end()) continue;
        auto& note = it->second;
        // Combined score: similarity * retention
        double similarity = 1.0 - entry.score;  // cos distance → similarity
        double retention  = note.retention_at(now);
        double combined   = similarity * retention;
        scored.push_back({ combined, &note });
    }

    std::sort(scored.begin(), scored.end(),
              [](const auto& a, const auto& b) { return a.first > b.first; });

    std::vector<MemoryNote> result;
    for (std::size_t i = 0; i < std::min(top_k, scored.size()); ++i) {
        access(scored[i].second->id);
        result.push_back(*scored[i].second);
    }
    return result;
}

void AMem::access(uint64_t id) {
    auto it = notes_.find(id);
    if (it == notes_.end()) return;
    auto& note = it->second;
    note.access_count++;
    // Stability increases with each retrieval (Ebbinghaus SRS)
    note.stability    = note.stability * 1.5 + 0.5;
    note.last_accessed = std::chrono::system_clock::now();
}

void AMem::prune(double min_retention) {
    auto now = std::chrono::system_clock::now();
    std::erase_if(notes_, [&](const auto& kv) {
        return kv.second.retention_at(now) < min_retention;
    });
}

std::vector<uint64_t> AMem::generate_links(
    const std::vector<MemoryEntry>& similar, float threshold)
{
    std::vector<uint64_t> links;
    for (const auto& e : similar) {
        if (e.score <= (1.0f - threshold))  // distance → similarity
            links.push_back(e.id);
    }
    return links;
}

} // namespace memory
```

---

## 5. Modular RAG Pipeline

The pipeline is a chain of composable stages, each taking a query and returning
enriched results. Stages are `std::function` objects to allow hot-swapping.

```cpp
// src/rag/pipeline.hpp
#pragma once
#include <string>
#include <vector>
#include <functional>
#include <expected>
#include "agent/agent_error.hpp"

namespace rag {

struct Document {
    std::string   id;
    std::string   content;
    nlohmann::json metadata;
    float          score = 0.0f;
};

struct RAGQuery {
    std::string              original;
    std::vector<std::string> expanded;  // rewritten/expanded queries
};

// Each pipeline stage transforms query + docs
using RetrievalStage  = std::function<
    AgentResult<std::vector<Document>>(const RAGQuery&)>;

using RerankStage     = std::function<
    AgentResult<std::vector<Document>>(const RAGQuery&, std::vector<Document>)>;

using CompressionStage = std::function<
    AgentResult<std::string>(const RAGQuery&, const std::vector<Document>&)>;

struct RAGPipeline {
    std::vector<RetrievalStage>  retrieval_stages;  // run all, merge
    std::vector<RerankStage>     rerank_stages;     // applied in order
    CompressionStage             compressor;

    AgentResult<std::string> run(const std::string& query) const;
};

} // namespace rag
```

```cpp
// src/rag/pipeline.cpp
#include "rag/pipeline.hpp"
#include <spdlog/spdlog.h>

namespace rag {

AgentResult<std::string> RAGPipeline::run(const std::string& query) const {
    RAGQuery rq{ query, { query } };

    // 1. Retrieve from all sources
    std::vector<Document> all_docs;
    for (const auto& stage : retrieval_stages) {
        auto docs = stage(rq);
        if (!docs) {
            spdlog::warn("[rag] retrieval stage failed: {}", docs.error().message);
            continue;
        }
        all_docs.insert(all_docs.end(), docs->begin(), docs->end());
    }

    if (all_docs.empty()) {
        return std::unexpected(AgentError{
            AgentErrorCode::VectorSearchFailed,
            "RAG pipeline: no documents retrieved"
        });
    }

    // 2. Rerank
    for (const auto& stage : rerank_stages) {
        auto reranked = stage(rq, all_docs);
        if (!reranked) continue;
        all_docs = std::move(*reranked);
    }

    // 3. Compress/format
    if (compressor) {
        return compressor(rq, all_docs);
    }

    // Default: concatenate top docs
    std::string context;
    for (std::size_t i = 0; i < std::min(std::size_t(5), all_docs.size()); ++i)
        context += std::format("[{}]\n{}\n\n", i + 1, all_docs[i].content);
    return context;
}

} // namespace rag
```

---

## 6. HyDE — Hypothetical Document Embeddings (arXiv:2212.10496)

```cpp
// src/rag/hyde.cpp
#include "rag/hyde.hpp"

namespace rag {

// HyDE: generate a hypothetical ideal document, embed it, search real corpus
AgentResult<std::vector<Document>> hyde_retrieve(
    const std::string& query,
    const memory::RecallStorage& storage,
    inference::SSEParser& sse,
    const std::function<std::vector<float>(const std::string&)>& embedder,
    std::size_t top_k)
{
    // Step 1: Generate hypothetical answer document
    nlohmann::json req = {
        {"model",  "gpt-4o-mini"},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content",
            "Write a detailed paragraph that would answer the following question. "
            "Write as if you are a document that contains the answer:\n\n" + query
        }}}}
    };
    std::string hypothetical_doc = sse.accumulate(req);

    // Step 2: Embed hypothetical doc and search real corpus
    auto hyp_embedding = embedder(hypothetical_doc);

    auto results = storage.search(hyp_embedding, top_k);
    if (!results) return std::unexpected(results.error());

    std::vector<Document> docs;
    for (const auto& entry : *results)
        docs.push_back({ std::to_string(entry.id), entry.text, entry.metadata, entry.score });

    return docs;
}

} // namespace rag
```

---

## 7. RAG-Fusion with Reciprocal Rank Fusion (arXiv:2402.03367)

```cpp
// src/rag/rag_fusion.cpp
#include "rag/rag_fusion.hpp"

namespace rag {

// Generate N diverse query variants
std::vector<std::string> generate_queries(
    const std::string& original,
    int n,
    inference::SSEParser& sse)
{
    nlohmann::json req = {
        {"model",  "gpt-4o-mini"},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content",
            std::format("Generate {} diverse search queries that cover different "
                "aspects of the following question. Return as JSON array of strings:\n\n{}",
                n, original)
        }}}}
    };
    std::string raw = sse.accumulate(req);
    std::vector<std::string> queries = { original };
    try {
        auto j = nlohmann::json::parse(raw);
        for (const auto& q : j)
            queries.push_back(q.get<std::string>());
    } catch (...) {}
    return queries;
}

// Reciprocal Rank Fusion: score = Σ 1/(k + rank_i) for each result list
std::vector<Document> reciprocal_rank_fusion(
    const std::vector<std::vector<Document>>& result_lists,
    int k = 60)
{
    std::unordered_map<std::string, double> scores;
    std::unordered_map<std::string, Document> doc_map;

    for (const auto& list : result_lists) {
        for (std::size_t rank = 0; rank < list.size(); ++rank) {
            const auto& doc = list[rank];
            scores[doc.id]  += 1.0 / (k + static_cast<int>(rank) + 1);
            doc_map[doc.id]  = doc;
        }
    }

    std::vector<std::pair<double, std::string>> ranked;
    for (const auto& [id, score] : scores)
        ranked.push_back({ score, id });

    std::sort(ranked.begin(), ranked.end(), std::greater<>());

    std::vector<Document> result;
    for (const auto& [score, id] : ranked) {
        auto doc   = doc_map[id];
        doc.score  = static_cast<float>(score);
        result.push_back(std::move(doc));
    }
    return result;
}

// Full RAG-Fusion pipeline
AgentResult<std::vector<Document>> rag_fusion_retrieve(
    const std::string& query,
    const memory::RecallStorage& storage,
    inference::SSEParser& sse,
    const std::function<std::vector<float>(const std::string&)>& embedder,
    int num_queries,
    std::size_t top_k)
{
    auto queries = generate_queries(query, num_queries - 1, sse);

    std::vector<std::vector<Document>> all_results;
    for (const auto& q : queries) {
        auto emb  = embedder(q);
        auto hits = storage.search(emb, top_k);
        if (!hits) continue;
        std::vector<Document> docs;
        for (const auto& e : *hits)
            docs.push_back({ std::to_string(e.id), e.text, e.metadata, e.score });
        all_results.push_back(std::move(docs));
    }

    if (all_results.empty()) {
        return std::unexpected(AgentError{
            AgentErrorCode::VectorSearchFailed, "RAG-Fusion: all query variants failed"
        });
    }

    return reciprocal_rank_fusion(all_results);
}

} // namespace rag
```

---

## 8. CRAG — Corrective RAG (arXiv:2401.15884)

CRAG evaluates retrieved document relevance with a confidence score.
If confidence is low, it falls back to web search.

```cpp
// src/rag/crag.cpp
#include "rag/crag.hpp"

namespace rag {

// Score retrieved documents for relevance to the query [0.0, 1.0]
float score_relevance(
    const std::string& query,
    const Document& doc,
    inference::SSEParser& sse)
{
    nlohmann::json req = {
        {"model",  "gpt-4o-mini"},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content",
            std::format("Rate the relevance of this document to the query on a scale 0-10.\n"
                "Query: {}\nDocument: {}\nRespond with only a number 0-10.",
                query, doc.content.substr(0, 500))
        }}}}
    };
    std::string raw = sse.accumulate(req);
    try {
        float score = std::stof(raw) / 10.0f;
        return std::clamp(score, 0.0f, 1.0f);
    } catch (...) {
        return 0.5f;  // uncertain
    }
}

AgentResult<std::vector<Document>> crag_retrieve(
    const std::string& query,
    const memory::RecallStorage& storage,
    const std::function<std::vector<float>(const std::string&)>& embedder,
    inference::SSEParser& sse,
    tools::ToolRegistry& tool_registry,
    float confidence_threshold)
{
    // Step 1: Retrieve from vector store
    auto emb  = embedder(query);
    auto hits = storage.search(emb, 5);
    if (!hits) return std::unexpected(hits.error());

    std::vector<Document> docs;
    for (const auto& e : *hits)
        docs.push_back({ std::to_string(e.id), e.text, e.metadata, e.score });

    // Step 2: Score top result
    float confidence = docs.empty() ? 0.0f : score_relevance(query, docs[0], sse);

    if (confidence >= confidence_threshold) {
        // CORRECT — use retrieved docs as-is
        return docs;
    }

    if (confidence < 0.3f) {
        // INCORRECT — discard and use web search only
        docs.clear();
    }
    // else AMBIGUOUS — combine both

    // Step 3: Web search fallback
    auto* web_tool = tool_registry.find("web_search");
    if (web_tool) {
        tools::ToolContext tctx;
        auto web_result = web_tool->handler(
            { {"query", query}, {"max_results", 5} }, tctx);
        if (web_result) {
            docs.push_back({
                "web_0",
                web_result->content,
                { {"source", "web"} },
                confidence
            });
        }
    }

    return docs;
}

} // namespace rag
```

---

## 9. Self-RAG Reflection Tokens (arXiv:2310.11511)

Self-RAG introduces special reflection tokens to control retrieval:

```cpp
// src/rag/self_rag.hpp
#pragma once
#include <string>
#include <optional>

namespace rag {

// Self-RAG reflection token values
enum class RetrieveToken { Retrieve, NoRetrieve };
enum class IsRelToken    { Relevant, Irrelevant };
enum class IsSupToken    { Fully, Partially, No };
enum class IsUseToken    { Useful, NotUseful };

struct SelfRAGTokens {
    RetrieveToken retrieve;
    std::optional<IsRelToken>  is_relevant;
    std::optional<IsSupToken>  is_supported;
    std::optional<IsUseToken>  is_useful;
};

// Parse Self-RAG reflection tokens from model output
SelfRAGTokens parse_reflection_tokens(const std::string& text);

// Prompt that instructs the model to emit reflection tokens
std::string build_self_rag_prompt(const std::string& task,
                                   const std::vector<Document>& retrieved_docs);

// Decision logic: given tokens, should we retrieve/refine?
bool should_retrieve(const SelfRAGTokens& tokens);
bool should_refine(const SelfRAGTokens& tokens);

} // namespace rag
```

---

## 10. Contextual Compressor

Reduces retrieved context to only the sentences/paragraphs relevant to the query:

```cpp
// src/rag/compressor.cpp
#include "rag/compressor.hpp"

namespace rag {

std::string contextual_compress(
    const std::string& query,
    const std::vector<Document>& docs,
    inference::SSEParser& sse,
    std::size_t max_chars)
{
    std::string all_content;
    for (std::size_t i = 0; i < std::min(docs.size(), std::size_t(5)); ++i)
        all_content += std::format("[Doc {}]\n{}\n\n", i + 1, docs[i].content);

    nlohmann::json req = {
        {"model",  "gpt-4o-mini"},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content",
            std::format("Given the following documents and a query, extract only the "
                "sentences directly relevant to answering the query. "
                "Be concise. Maximum {} characters.\n\n"
                "Query: {}\n\nDocuments:\n{}",
                max_chars, query, all_content)
        }}}}
    };

    return sse.accumulate(req);
}

} // namespace rag
```

---

## 11. LongMemEval — Atomic Fact Decomposition (arXiv:2410.10813)

For long sessions, decompose each session into atomic indexable facts:

```cpp
// src/memory/longmemeval.cpp
#include "memory/longmemeval.hpp"

namespace memory {

std::vector<std::string> decompose_to_atomic_facts(
    const std::string& session_text,
    inference::SSEParser& sse)
{
    nlohmann::json req = {
        {"model",  "gpt-4o-mini"},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content",
            "Decompose the following conversation/text into atomic facts. "
            "Each fact should be a single, self-contained statement. "
            "Return as a JSON array of strings.\n\n" + session_text
        }}}}
    };

    std::string raw = sse.accumulate(req);
    std::vector<std::string> facts;
    try {
        auto j = nlohmann::json::parse(raw);
        for (const auto& fact : j)
            facts.push_back(fact.get<std::string>());
    } catch (...) {
        // Fallback: split by newlines
        std::istringstream ss(raw);
        std::string line;
        while (std::getline(ss, line))
            if (!line.empty()) facts.push_back(line);
    }
    return facts;
}

// Index atomic facts into archival storage
AgentResult<void> index_session_facts(
    const std::string& session_text,
    ArchivalStorage& archive,
    inference::SSEParser& sse,
    const std::function<std::vector<float>(const std::string&)>& embedder,
    const std::string& session_id)
{
    auto facts = decompose_to_atomic_facts(session_text, sse);
    for (const auto& fact : facts) {
        auto emb = embedder(fact);
        auto r = archive.insert(fact, emb, { {"session_id", session_id}, {"type", "atomic_fact"} });
        if (!r) return std::unexpected(r.error());
    }
    return {};
}

} // namespace memory
```

---

## 12. StreamingLLM — Attention Sink + Sliding Window (arXiv:2309.17453)

For very long contexts, StreamingLLM keeps a fixed set of "sink" tokens at the
start of the KV cache (the first 4 tokens of the prompt) plus a sliding window
of the most recent tokens. This enables infinite-length generation without
recomputing the full attention.

```cpp
// src/memory/streaming_llm.hpp
#pragma once
#include <vector>
#include <deque>
#include <string>

namespace memory {

struct KVCacheWindow {
    // Sink tokens — always kept (first K tokens of original prompt)
    std::vector<std::string> sink_tokens;
    int sink_count = 4;

    // Sliding window of recent tokens
    std::deque<std::string> window;
    int window_size = 2048;  // tokens

    // Add new tokens, evicting old ones if window is full
    void push(const std::string& token) {
        window.push_back(token);
        while (static_cast<int>(window.size()) > window_size)
            window.pop_front();
    }

    // Reconstruct the effective context as: sink_tokens + window
    std::string effective_context() const {
        std::string ctx;
        for (const auto& t : sink_tokens) ctx += t;
        for (const auto& t : window)      ctx += t;
        return ctx;
    }

    int total_tokens() const {
        return static_cast<int>(sink_tokens.size() + window.size());
    }
};

// Initialize sink tokens from a prompt (first `sink_count` tokens)
inline KVCacheWindow init_streaming_window(
    const std::string& initial_prompt,
    int sink_count = 4,
    int window_size = 2048)
{
    KVCacheWindow w;
    w.sink_count  = sink_count;
    w.window_size = window_size;

    // Approximate tokenization: split by whitespace for demonstration
    // In production: use a proper tokenizer
    std::istringstream ss(initial_prompt);
    std::string tok;
    int count = 0;
    while (ss >> tok) {
        if (count++ < sink_count)
            w.sink_tokens.push_back(tok + " ");
        else
            w.window.push_back(tok + " ");
    }
    // Trim window to size
    while (static_cast<int>(w.window.size()) > window_size)
        w.window.pop_front();

    return w;
}

} // namespace memory
```
