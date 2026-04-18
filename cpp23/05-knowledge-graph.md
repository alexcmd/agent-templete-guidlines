# 05 — Knowledge Graph Integration

This document covers the complete knowledge graph subsystem: entity/relation storage,
triple extraction, HippoRAG Personalized PageRank retrieval (arXiv:2405.14831),
LightRAG dual-level retrieval (arXiv:2410.05779), GraphRAG community detection
(arXiv:2404.16130), PathRAG flow-based pruning (arXiv:2502.14902), and persistence.

---

## 1. Core KG Types

```cpp
// src/kg/knowledge_graph.hpp
#pragma once
#include <string>
#include <vector>
#include <unordered_map>
#include <unordered_set>
#include <optional>
#include <flat_map>
#include <nlohmann/json.hpp>

namespace kg {

using EntityId = std::string;  // Stable identifier, e.g., canonical name

struct Entity {
    EntityId    id;
    std::string name;         // Display name
    std::string type;         // "person", "organization", "concept", "file", etc.
    std::string description;  // Free-text description
    nlohmann::json attributes;

    // USearch embedding ID for entity lookup
    std::optional<uint64_t> embedding_id;
};

struct Relation {
    std::string  id;         // Unique relation ID
    EntityId     source;
    EntityId     target;
    std::string  predicate;  // "is_a", "depends_on", "calls", "authored_by", etc.
    float        weight = 1.0f;
    std::string  provenance;  // Source document or session
    nlohmann::json attributes;
};

struct Triple {
    EntityId    subject;
    std::string predicate;
    EntityId    object;
};

class KnowledgeGraph {
public:
    // Entity management
    void         add_entity(Entity e);
    const Entity* find_entity(const EntityId& id) const noexcept;
    Entity*       find_entity(const EntityId& id) noexcept;
    bool          has_entity(const EntityId& id) const noexcept;

    // Relation management
    void         add_relation(Relation r);
    std::vector<const Relation*> relations_from(const EntityId& id) const;
    std::vector<const Relation*> relations_to(const EntityId& id) const;
    std::vector<const Relation*> relations_between(
        const EntityId& src, const EntityId& dst) const;

    // Graph traversal
    std::vector<EntityId> neighbors(const EntityId& id, int max_hops = 1) const;
    std::vector<std::vector<EntityId>> paths(
        const EntityId& src, const EntityId& dst, int max_length = 4) const;

    // Statistics
    std::size_t entity_count() const { return entities_.size(); }
    std::size_t relation_count() const { return relations_.size(); }

    // Serialization
    nlohmann::json to_json() const;
    static KnowledgeGraph from_json(const nlohmann::json& j);
    void save_to_file(const std::string& path) const;
    static KnowledgeGraph load_from_file(const std::string& path);

private:
    // Entity store — flat_map for cache-friendly access
    std::flat_map<EntityId, Entity>  entities_;

    // Relation store
    std::vector<Relation>            relations_;

    // Adjacency index: source → list of relation indices
    std::unordered_map<EntityId, std::vector<std::size_t>> out_edges_;
    // Reverse adjacency: target → relation indices
    std::unordered_map<EntityId, std::vector<std::size_t>> in_edges_;
};

} // namespace kg
```

```cpp
// src/kg/knowledge_graph.cpp
#include "kg/knowledge_graph.hpp"
#include <fstream>

namespace kg {

void KnowledgeGraph::add_entity(Entity e) {
    entities_[e.id] = std::move(e);
}

const Entity* KnowledgeGraph::find_entity(const EntityId& id) const noexcept {
    auto it = entities_.find(id);
    return it != entities_.end() ? &it->second : nullptr;
}

Entity* KnowledgeGraph::find_entity(const EntityId& id) noexcept {
    auto it = entities_.find(id);
    return it != entities_.end() ? &it->second : nullptr;
}

bool KnowledgeGraph::has_entity(const EntityId& id) const noexcept {
    return entities_.contains(id);
}

void KnowledgeGraph::add_relation(Relation r) {
    std::size_t idx = relations_.size();
    out_edges_[r.source].push_back(idx);
    in_edges_[r.target].push_back(idx);

    // Auto-create entities if they don't exist
    if (!has_entity(r.source))
        add_entity({ r.source, r.source, "unknown", "" });
    if (!has_entity(r.target))
        add_entity({ r.target, r.target, "unknown", "" });

    relations_.push_back(std::move(r));
}

std::vector<const Relation*> KnowledgeGraph::relations_from(const EntityId& id) const {
    std::vector<const Relation*> result;
    auto it = out_edges_.find(id);
    if (it == out_edges_.end()) return result;
    for (auto idx : it->second) result.push_back(&relations_[idx]);
    return result;
}

std::vector<const Relation*> KnowledgeGraph::relations_to(const EntityId& id) const {
    std::vector<const Relation*> result;
    auto it = in_edges_.find(id);
    if (it == in_edges_.end()) return result;
    for (auto idx : it->second) result.push_back(&relations_[idx]);
    return result;
}

std::vector<EntityId> KnowledgeGraph::neighbors(const EntityId& id, int max_hops) const {
    std::unordered_set<EntityId> visited;
    std::vector<EntityId> frontier = { id };
    visited.insert(id);

    for (int hop = 0; hop < max_hops; ++hop) {
        std::vector<EntityId> next;
        for (const auto& node : frontier) {
            for (const auto* r : relations_from(node)) {
                if (!visited.count(r->target)) {
                    visited.insert(r->target);
                    next.push_back(r->target);
                }
            }
            for (const auto* r : relations_to(node)) {
                if (!visited.count(r->source)) {
                    visited.insert(r->source);
                    next.push_back(r->source);
                }
            }
        }
        frontier = std::move(next);
    }

    visited.erase(id);
    return { visited.begin(), visited.end() };
}

nlohmann::json KnowledgeGraph::to_json() const {
    nlohmann::json j;
    for (const auto& [id, e] : entities_) {
        j["entities"].push_back({
            {"id", e.id}, {"name", e.name}, {"type", e.type},
            {"description", e.description}, {"attributes", e.attributes}
        });
    }
    for (const auto& r : relations_) {
        j["relations"].push_back({
            {"id", r.id}, {"source", r.source}, {"target", r.target},
            {"predicate", r.predicate}, {"weight", r.weight},
            {"provenance", r.provenance}
        });
    }
    return j;
}

void KnowledgeGraph::save_to_file(const std::string& path) const {
    std::ofstream f(path);
    f << to_json().dump(2);
}

} // namespace kg
```

---

## 2. Triple Extraction from Text

```cpp
// src/kg/triple_extractor.cpp
#include "kg/triple_extractor.hpp"

namespace kg {

// Extract triples from raw text using an LLM
std::vector<Triple> extract_triples(
    const std::string& text,
    inference::SSEParser& sse,
    const std::string& model)
{
    nlohmann::json req = {
        {"model",  model},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content",
            "Extract all factual triples from the following text. "
            "A triple has: subject (entity), predicate (relationship), object (entity or value).\n\n"
            "Return JSON array: [{\"subject\": \"...\", \"predicate\": \"...\", \"object\": \"...\"}]\n\n"
            "Normalize entity names to canonical form (e.g., 'Barack Obama' not 'Obama').\n\n"
            "Text:\n" + text
        }}}}
    };

    std::string raw = sse.accumulate(req);
    std::vector<Triple> triples;

    try {
        auto j = nlohmann::json::parse(raw);
        for (const auto& t : j) {
            Triple triple;
            triple.subject   = t["subject"].get<std::string>();
            triple.predicate = t["predicate"].get<std::string>();
            triple.object    = t["object"].get<std::string>();
            triples.push_back(std::move(triple));
        }
    } catch (...) {
        spdlog::warn("[kg] triple extraction parse failed for text of {} chars", text.size());
    }

    return triples;
}

// Ingest a document: extract triples and add to KG
void ingest_document(
    const std::string& doc_text,
    const std::string& doc_id,
    KnowledgeGraph& kg,
    inference::SSEParser& sse)
{
    auto triples = extract_triples(doc_text, sse);
    for (std::size_t i = 0; i < triples.size(); ++i) {
        const auto& t = triples[i];

        // Ensure entities exist
        if (!kg.has_entity(t.subject))
            kg.add_entity({ t.subject, t.subject, "unknown", "" });
        if (!kg.has_entity(t.object))
            kg.add_entity({ t.object, t.object, "unknown", "" });

        kg.add_relation({
            std::format("{}_{}_{}", doc_id, i, t.predicate),
            t.subject,
            t.object,
            t.predicate,
            1.0f,
            doc_id
        });
    }
}

} // namespace kg
```

---

## 3. HippoRAG — Personalized PageRank Retrieval (arXiv:2405.14831)

HippoRAG models the KG like a hippocampal memory: given query entities,
it runs Personalized PageRank (PPR) on the KG to find the most strongly
associated entities and their surrounding context.

```cpp
// src/kg/hipporag.cpp
#include "kg/hipporag.hpp"
#include <cmath>
#include <numeric>

namespace kg {

// Personalized PageRank on the KG
// Returns entity ID → PPR score map
std::unordered_map<EntityId, double> personalized_pagerank(
    const KnowledgeGraph& graph,
    const std::vector<EntityId>& seed_entities,
    int max_iterations,
    double damping,      // d = 0.85 typical
    double convergence)  // stop when max delta < this
{
    if (seed_entities.empty()) return {};

    // Initialize: uniform seed distribution
    std::unordered_map<EntityId, double> scores;
    double seed_score = 1.0 / seed_entities.size();
    for (const auto& eid : seed_entities)
        scores[eid] = seed_score;

    // Build adjacency: out_degree for normalization
    std::unordered_map<EntityId, double> out_degree;
    for (const auto& id : seed_entities) {
        for (const auto* r : graph.relations_from(id))
            out_degree[id] += r->weight;
    }
    // Include all nodes
    for (const auto& nbr : graph.neighbors(seed_entities[0], 3)) {
        for (const auto* r : graph.relations_from(nbr))
            out_degree[nbr] += r->weight;
        if (!scores.count(nbr)) scores[nbr] = 0.0;
    }

    // PPR iteration
    for (int iter = 0; iter < max_iterations; ++iter) {
        std::unordered_map<EntityId, double> new_scores;
        double max_delta = 0.0;

        for (const auto& [node, _] : scores) {
            // Teleportation to seed
            double teleport = 0.0;
            for (const auto& seed : seed_entities)
                if (node == seed) teleport = (1.0 - damping) * seed_score;

            // Sum incoming contributions
            double incoming = 0.0;
            for (const auto* r : graph.relations_to(node)) {
                double src_score = 0.0;
                if (auto it = scores.find(r->source); it != scores.end())
                    src_score = it->second;
                double deg = out_degree.count(r->source) ? out_degree[r->source] : 1.0;
                incoming += damping * src_score * r->weight / deg;
            }

            new_scores[node] = teleport + incoming;
            max_delta = std::max(max_delta, std::abs(new_scores[node] - scores[node]));
        }

        scores = std::move(new_scores);
        if (max_delta < convergence) break;
    }

    return scores;
}

// HippoRAG retrieval: identify seed entities in query, run PPR, collect context
AgentResult<std::string> hipporag_retrieve(
    const std::string& query,
    const KnowledgeGraph& graph,
    const memory::RecallStorage& entity_vectors,
    const std::function<std::vector<float>(const std::string&)>& embedder,
    inference::SSEParser& sse,
    std::size_t top_k)
{
    // Step 1: Identify query entities via NER or embedding search
    auto query_emb = embedder(query);
    auto seed_hits = entity_vectors.search(query_emb, 3);
    if (!seed_hits || seed_hits->empty()) {
        return std::unexpected(AgentError{
            AgentErrorCode::VectorSearchFailed, "HippoRAG: no seed entities found"
        });
    }

    std::vector<EntityId> seeds;
    for (const auto& hit : *seed_hits)
        seeds.push_back(hit.text);  // entity ID stored as text

    // Step 2: PPR over KG
    auto ppr_scores = personalized_pagerank(graph, seeds, 20, 0.85, 1e-6);

    // Step 3: Collect top-K entities and their relations
    std::vector<std::pair<double, EntityId>> ranked;
    for (const auto& [id, score] : ppr_scores)
        ranked.push_back({ score, id });
    std::sort(ranked.begin(), ranked.end(), std::greater<>());

    // Step 4: Build context string
    std::string context;
    for (std::size_t i = 0; i < std::min(top_k, ranked.size()); ++i) {
        const auto& eid = ranked[i].second;
        if (const auto* entity = graph.find_entity(eid)) {
            context += std::format("Entity: {} ({})\nDescription: {}\n",
                entity->name, entity->type, entity->description);
            for (const auto* rel : graph.relations_from(eid)) {
                context += std::format("  -[{}]-> {}\n",
                    rel->predicate, rel->target);
            }
            context += "\n";
        }
    }

    return context;
}

} // namespace kg
```

---

## 4. LightRAG — Dual-Level Retrieval (arXiv:2410.05779)

LightRAG retrieves at two levels: local (entity neighborhood) and global (community):

```cpp
// src/kg/lightrag.cpp
#include "kg/lightrag.hpp"

namespace kg {

// Local retrieval: find entities matching query, return their neighborhoods
AgentResult<std::string> lightrag_local(
    const std::string& query,
    const KnowledgeGraph& graph,
    const memory::RecallStorage& entity_vectors,
    const std::function<std::vector<float>(const std::string&)>& embedder,
    int neighborhood_hops)
{
    auto emb  = embedder(query);
    auto hits = entity_vectors.search(emb, 5);
    if (!hits) return std::unexpected(hits.error());

    std::string context;
    for (const auto& hit : *hits) {
        const auto& eid = hit.text;
        if (const auto* e = graph.find_entity(eid)) {
            context += std::format("## {}\n{}\n", e->name, e->description);
            for (const auto& nbr : graph.neighbors(eid, neighborhood_hops)) {
                if (const auto* ne = graph.find_entity(nbr)) {
                    context += std::format("  Related: {} — {}\n", ne->name, ne->description);
                }
            }
            context += "\n";
        }
    }
    return context;
}

// Global retrieval: summarize communities relevant to the query
AgentResult<std::string> lightrag_global(
    const std::string& query,
    const std::vector<Community>& communities,
    inference::SSEParser& sse)
{
    // Score communities by query relevance (simplified: keyword overlap)
    std::vector<std::pair<int, const Community*>> scored;
    for (const auto& c : communities) {
        int overlap = 0;
        for (const auto& keyword : c.keywords)
            if (query.find(keyword) != std::string::npos) ++overlap;
        scored.push_back({ overlap, &c });
    }
    std::sort(scored.begin(), scored.end(), std::greater<>());

    std::string context;
    for (std::size_t i = 0; i < std::min(scored.size(), std::size_t(3)); ++i)
        context += std::format("[Community {}]\n{}\n\n",
            scored[i].second->id, scored[i].second->summary);

    return context;
}

// Combined dual-level retrieval
AgentResult<std::string> lightrag_retrieve(
    const std::string& query,
    const KnowledgeGraph& graph,
    const memory::RecallStorage& entity_vectors,
    const std::vector<Community>& communities,
    const std::function<std::vector<float>(const std::string&)>& embedder,
    inference::SSEParser& sse)
{
    auto local_ctx  = lightrag_local(query, graph, entity_vectors, embedder, 2);
    auto global_ctx = lightrag_global(query, communities, sse);

    std::string combined;
    if (local_ctx)  combined += "### Local Context\n" + *local_ctx + "\n";
    if (global_ctx) combined += "### Global Context\n" + *global_ctx + "\n";

    if (combined.empty()) {
        return std::unexpected(AgentError{
            AgentErrorCode::VectorSearchFailed, "LightRAG: no context found"
        });
    }
    return combined;
}

} // namespace kg
```

---

## 5. GraphRAG — Community Detection + Summaries (arXiv:2404.16130)

GraphRAG applies Leiden community detection to the KG and generates a summary
for each community. Queries are answered at the community level.

```cpp
// src/kg/graphrag.hpp
#pragma once
#include <string>
#include <vector>
#include <unordered_map>
#include "kg/knowledge_graph.hpp"

namespace kg {

struct Community {
    std::string              id;
    std::vector<EntityId>    members;
    std::string              summary;    // LLM-generated
    std::vector<std::string> keywords;
    float                    level = 0;  // hierarchy level
};

// Simple union-find for connected components (approximation of Leiden)
class UnionFind {
public:
    explicit UnionFind(int n) : parent_(n), rank_(n, 0) {
        std::iota(parent_.begin(), parent_.end(), 0);
    }

    int find(int x) {
        if (parent_[x] != x) parent_[x] = find(parent_[x]);
        return parent_[x];
    }

    void unite(int x, int y) {
        int px = find(x), py = find(y);
        if (px == py) return;
        if (rank_[px] < rank_[py]) std::swap(px, py);
        parent_[py] = px;
        if (rank_[px] == rank_[py]) rank_[px]++;
    }

private:
    std::vector<int> parent_, rank_;
};

// Detect communities using union-find on weighted edges
std::vector<Community> detect_communities(
    const KnowledgeGraph& graph,
    float weight_threshold = 0.5f);

// Generate LLM summary for a community
std::string summarize_community(
    const Community& community,
    const KnowledgeGraph& graph,
    inference::SSEParser& sse,
    const std::string& model = "gpt-4o-mini");

// Build community index from KG (run once at ingest time)
std::vector<Community> build_community_index(
    const KnowledgeGraph& graph,
    inference::SSEParser& sse);

} // namespace kg
```

```cpp
// src/kg/graphrag.cpp
#include "kg/graphrag.hpp"
#include <numeric>

namespace kg {

std::vector<Community> detect_communities(
    const KnowledgeGraph& graph,
    float weight_threshold)
{
    // Assign sequential IDs to entities
    std::vector<EntityId> entity_ids;
    std::unordered_map<EntityId, int> id_to_idx;
    for (const auto& eid : graph.neighbors("", 100)) {  // crude iteration
        id_to_idx[eid] = static_cast<int>(entity_ids.size());
        entity_ids.push_back(eid);
    }
    // Also add entities from relations
    // (In production, KG would expose an iterator)

    int N = static_cast<int>(entity_ids.size());
    if (N == 0) return {};

    UnionFind uf(N);

    // Unite entities connected by high-weight relations
    for (const auto& eid : entity_ids) {
        for (const auto* rel : graph.relations_from(eid)) {
            if (rel->weight >= weight_threshold) {
                auto it1 = id_to_idx.find(rel->source);
                auto it2 = id_to_idx.find(rel->target);
                if (it1 != id_to_idx.end() && it2 != id_to_idx.end())
                    uf.unite(it1->second, it2->second);
            }
        }
    }

    // Collect components
    std::unordered_map<int, std::vector<EntityId>> components;
    for (int i = 0; i < N; ++i)
        components[uf.find(i)].push_back(entity_ids[i]);

    std::vector<Community> communities;
    int cid = 0;
    for (auto& [root, members] : components) {
        Community c;
        c.id      = std::format("community_{}", cid++);
        c.members = std::move(members);
        communities.push_back(std::move(c));
    }
    return communities;
}

std::string summarize_community(
    const Community& community,
    const KnowledgeGraph& graph,
    inference::SSEParser& sse,
    const std::string& model)
{
    // Build entity descriptions
    std::string entities_text;
    for (const auto& eid : community.members) {
        if (const auto* e = graph.find_entity(eid)) {
            entities_text += std::format("- {} ({}): {}\n",
                e->name, e->type, e->description);
            for (const auto* r : graph.relations_from(eid))
                entities_text += std::format("  [{} -> {}]\n", r->predicate, r->target);
        }
    }

    nlohmann::json req = {
        {"model",  model},
        {"stream", false},
        {"messages", {{{"role","user"}, {"content",
            "Summarize the following group of related entities and their relationships "
            "in 2-3 sentences. Focus on what this group represents as a whole.\n\n"
            + entities_text
        }}}}
    };

    return sse.accumulate(req);
}

std::vector<Community> build_community_index(
    const KnowledgeGraph& graph,
    inference::SSEParser& sse)
{
    auto communities = detect_communities(graph);
    for (auto& c : communities) {
        c.summary = summarize_community(c, graph, sse);

        // Extract keywords from summary (simplified)
        std::istringstream ss(c.summary);
        std::string word;
        while (ss >> word) {
            // Filter stop words in production
            if (word.size() > 4) c.keywords.push_back(word);
        }
    }
    return communities;
}

} // namespace kg
```

---

## 6. PathRAG — Flow-Based Path Pruning (arXiv:2502.14902)

PathRAG finds paths between query-relevant entities and prunes them based on
information flow (analogous to max-flow in a network).

```cpp
// src/kg/pathrag.cpp
#include "kg/pathrag.hpp"
#include <queue>

namespace kg {

// Find paths between seed entities using BFS
std::vector<std::vector<EntityId>> find_paths(
    const KnowledgeGraph& graph,
    const std::vector<EntityId>& seeds,
    int max_path_length,
    std::size_t max_paths)
{
    std::vector<std::vector<EntityId>> all_paths;

    for (std::size_t i = 0; i < seeds.size() && all_paths.size() < max_paths; ++i) {
        for (std::size_t j = i + 1; j < seeds.size() && all_paths.size() < max_paths; ++j) {
            // BFS from seeds[i] to seeds[j]
            using Path = std::vector<EntityId>;
            std::queue<Path> q;
            q.push({ seeds[i] });

            while (!q.empty() && all_paths.size() < max_paths) {
                auto path = q.front(); q.pop();
                const auto& current = path.back();

                if (current == seeds[j]) {
                    all_paths.push_back(path);
                    continue;
                }
                if (static_cast<int>(path.size()) >= max_path_length) continue;

                for (const auto* r : graph.relations_from(current)) {
                    // Avoid cycles
                    bool visited = std::any_of(path.begin(), path.end(),
                        [&](const auto& n) { return n == r->target; });
                    if (!visited) {
                        auto new_path = path;
                        new_path.push_back(r->target);
                        q.push(std::move(new_path));
                    }
                }
            }
        }
    }

    return all_paths;
}

// Score a path by its total edge weight (information flow proxy)
float score_path(const std::vector<EntityId>& path, const KnowledgeGraph& graph) {
    if (path.size() < 2) return 0.0f;
    float flow = 1.0f;
    for (std::size_t i = 0; i + 1 < path.size(); ++i) {
        auto rels = graph.relations_between(path[i], path[i + 1]);
        float max_w = 0.0f;
        for (const auto* r : rels) max_w = std::max(max_w, r->weight);
        flow *= max_w > 0 ? max_w : 0.1f;
    }
    return flow;
}

// PathRAG: retrieve by finding and pruning paths between query entities
AgentResult<std::string> pathrag_retrieve(
    const std::string& query,
    const KnowledgeGraph& graph,
    const memory::RecallStorage& entity_vectors,
    const std::function<std::vector<float>(const std::string&)>& embedder,
    int max_path_length,
    float prune_threshold)
{
    // Identify query entities
    auto query_emb = embedder(query);
    auto seed_hits = entity_vectors.search(query_emb, 4);
    if (!seed_hits || seed_hits->empty())
        return std::unexpected(AgentError{
            AgentErrorCode::VectorSearchFailed, "PathRAG: no seed entities"
        });

    std::vector<EntityId> seeds;
    for (const auto& h : *seed_hits) seeds.push_back(h.text);

    // Find paths between seed entities
    auto paths = find_paths(graph, seeds, max_path_length, 20);

    // Score and prune paths
    std::vector<std::pair<float, const std::vector<EntityId>*>> scored;
    for (const auto& path : paths) {
        float s = score_path(path, graph);
        if (s >= prune_threshold) scored.push_back({ s, &path });
    }
    std::sort(scored.begin(), scored.end(), std::greater<>());

    // Render paths as context
    std::string context;
    for (std::size_t i = 0; i < std::min(scored.size(), std::size_t(5)); ++i) {
        const auto& path = *scored[i].second;
        context += std::format("[Path, score={:.3f}]\n", scored[i].first);
        for (std::size_t k = 0; k + 1 < path.size(); ++k) {
            auto rels = graph.relations_between(path[k], path[k + 1]);
            std::string pred = rels.empty() ? "→" : rels[0]->predicate;
            context += std::format("  {} -[{}]-> ", path[k], pred);
        }
        if (!path.empty()) context += path.back();
        context += "\n\n";
    }

    if (context.empty())
        return std::unexpected(AgentError{
            AgentErrorCode::VectorSearchFailed, "PathRAG: no paths above threshold"
        });

    return context;
}

} // namespace kg
```

---

## 7. Entity Embeddings with USearch

Entity embeddings allow fast semantic lookup of entities by natural-language query:

```cpp
// src/kg/entity_embedder.cpp
#include "kg/entity_embedder.hpp"

namespace kg {

// Build entity embedding index from a KG
AgentResult<void> build_entity_embeddings(
    const KnowledgeGraph& graph,
    memory::RecallStorage& index,
    const std::function<std::vector<float>(const std::string&)>& embedder)
{
    // For each entity, embed "name: description" to capture type info
    for (std::size_t i = 0; i < graph.entity_count(); ++i) {
        // In production, KG exposes an iterator — simplified here
        // Iterate via neighbor traversal or maintain a flat list
    }

    // Example for a known entity:
    // auto embed_text = entity.name + ": " + entity.description;
    // auto emb = embedder(embed_text);
    // co_await index.add(emb, entity.id, {{"name", entity.name}});

    return {};
}

} // namespace kg
```

---

## 8. KG Persistence to SQLite

```cpp
// src/kg/kg_sqlite.cpp
#include "kg/kg_sqlite.hpp"
#include <SQLiteCpp/SQLiteCpp.h>

namespace kg {

void save_to_sqlite(const KnowledgeGraph& graph, const std::string& db_path) {
    SQLite::Database db(db_path, SQLite::OPEN_READWRITE | SQLite::OPEN_CREATE);

    db.exec("CREATE TABLE IF NOT EXISTS entities ("
            "  id TEXT PRIMARY KEY,"
            "  name TEXT,"
            "  type TEXT,"
            "  description TEXT,"
            "  attributes TEXT"
            ")");

    db.exec("CREATE TABLE IF NOT EXISTS relations ("
            "  id TEXT PRIMARY KEY,"
            "  source TEXT,"
            "  target TEXT,"
            "  predicate TEXT,"
            "  weight REAL,"
            "  provenance TEXT"
            ")");

    // Save entities
    auto kg_json = graph.to_json();
    SQLite::Transaction txn(db);

    SQLite::Statement ins_entity(db,
        "INSERT OR REPLACE INTO entities (id, name, type, description, attributes) "
        "VALUES (?, ?, ?, ?, ?)");

    for (const auto& e : kg_json["entities"]) {
        ins_entity.bind(1, e["id"].get<std::string>());
        ins_entity.bind(2, e["name"].get<std::string>());
        ins_entity.bind(3, e["type"].get<std::string>());
        ins_entity.bind(4, e["description"].get<std::string>());
        ins_entity.bind(5, e["attributes"].dump());
        ins_entity.exec();
        ins_entity.reset();
    }

    SQLite::Statement ins_rel(db,
        "INSERT OR REPLACE INTO relations (id, source, target, predicate, weight, provenance) "
        "VALUES (?, ?, ?, ?, ?, ?)");

    for (const auto& r : kg_json["relations"]) {
        ins_rel.bind(1, r["id"].get<std::string>());
        ins_rel.bind(2, r["source"].get<std::string>());
        ins_rel.bind(3, r["target"].get<std::string>());
        ins_rel.bind(4, r["predicate"].get<std::string>());
        ins_rel.bind(5, r["weight"].get<double>());
        ins_rel.bind(6, r["provenance"].get<std::string>());
        ins_rel.exec();
        ins_rel.reset();
    }

    txn.commit();
}

KnowledgeGraph load_from_sqlite(const std::string& db_path) {
    SQLite::Database db(db_path, SQLite::OPEN_READONLY);
    KnowledgeGraph graph;

    // Load entities
    SQLite::Statement q_entities(db, "SELECT id, name, type, description, attributes FROM entities");
    while (q_entities.executeStep()) {
        Entity e;
        e.id          = q_entities.getColumn(0).getString();
        e.name        = q_entities.getColumn(1).getString();
        e.type        = q_entities.getColumn(2).getString();
        e.description = q_entities.getColumn(3).getString();
        try {
            e.attributes = nlohmann::json::parse(q_entities.getColumn(4).getString());
        } catch (...) {}
        graph.add_entity(std::move(e));
    }

    // Load relations
    SQLite::Statement q_rels(db,
        "SELECT id, source, target, predicate, weight, provenance FROM relations");
    while (q_rels.executeStep()) {
        Relation r;
        r.id         = q_rels.getColumn(0).getString();
        r.source     = q_rels.getColumn(1).getString();
        r.target     = q_rels.getColumn(2).getString();
        r.predicate  = q_rels.getColumn(3).getString();
        r.weight     = static_cast<float>(q_rels.getColumn(4).getDouble());
        r.provenance = q_rels.getColumn(5).getString();
        graph.add_relation(std::move(r));
    }

    return graph;
}

} // namespace kg
```
