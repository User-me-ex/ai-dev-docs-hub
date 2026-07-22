# RAG Pipeline

> **Domain:** Retrieval-Augmented Generation
> **Applies to:** Kernel, Agents, Knowledge System, Vector Store, Obsidian Graph Engine
> **Last updated:** 2026-07-22

## Overview

AI Dev OS uses a **hybrid retrieval** strategy that combines three complementary signals: keyword (BM25), semantic (ANN), and relational (graph). Each signal captures a different aspect of relevance; their results are fused via reciprocal rank fusion to produce a single ranked list.

```
                    ┌──────────┐
                    │  Query   │
                    └─────┬────┘
                          │
               ┌──────────┼──────────┐
               ▼          ▼          ▼
           ┌──────┐  ┌──────┐  ┌────────┐
           │ BM25 │  │ ANN  │  │ Graph  │
           │FTS5  │  │HNSW  │  │Neighbor│
           │SQLite│  │cosine│  │Depth 1 │
           └──┬───┘  └──┬───┘  └───┬────┘
              │         │          │
              └─────────┼──────────┘
                        ▼
                ┌──────────────┐
                │ Rank Fusion  │
                │ (RRF, conf)  │
                └──────┬───────┘
                       ▼
                ┌──────────────┐
                │  Context     │
                │  Assembly    │
                └──────┬───────┘
                       ▼
                ┌──────────────┐
                │  Prompt +    │
                │  Generate    │
                └──────────────┘
```

## Query Flow

```
rag.retrieve(query, { top_k: 20, weights: { bm25: 0.4, ann: 0.4, graph: 0.2 } })
  │
  ├─ 1. embed(query) → query_vector (768-d, normalized)
  │
  ├─ 2. BM25 search: SQLite FTS5, top 20
  │
  ├─ 3. ANN search: Vector Store, top 20, cosine similarity
  │
  ├─ 4. Graph expansion: Obsidian Graph Engine neighbors (depth 1), top 10
  │
  ├─ 5. RRF fusion: Reciprocal Rank Fusion on all candidates
  │
  └─ 6. Context assembly: select top-K within token budget
```

## BM25 (Keyword)

| Component | Detail |
|-----------|--------|
| **Engine** | SQLite FTS5 (Full-Text Search) |
| **Table** | `fts_content(content_id, title, body, metadata)` |
| **Weights** | Title: 3.0, Body: 1.0, Metadata: 0.5 |
| **Tokenization** | Unicode61 tokenizer with tokenchars |
| **Ranking** | BM25 with default parameters (k1=1.2, b=0.75) |

BM25 is optimized for exact keyword matching. It is particularly effective for code, identifiers, and structured terms.

## ANN (Semantic)

| Component | Detail |
|-----------|--------|
| **Engine** | usearch HNSW index (see [Vector Store](VECTOR_STORE.md)) |
| **Metric** | Cosine similarity |
| **Default top-K** | 20 |
| **Query vector** | 768-d L2-normalized `float32` array |
| **Filters applied** | Optional metadata filters passed through from the caller |

ANN captures semantic similarity — documents that use different words but mean the same thing.

## Graph Expansion

| Component | Detail |
|-----------|--------|
| **Engine** | Obsidian Graph Engine |
| **Traversal** | Depth 1 (direct neighbors of any document matched by BM25 or ANN) |
| **Edge types** | `references`, `related`, `contained_in`, `derived_from` |
| **Max neighbors** | 10 per seed document |
| **Score** | Edge weight (0.0–1.0) assigned by the graph engine |

Graph expansion pulls in structurally related content — e.g., a function's callers, a document's parent section, or an agent memory's related context.

## Rank Fusion

Reciprocal Rank Fusion (RRF) combines the three ranked lists:

```
RRF_score(d) = w_bm25 * (1 / (k + rank_bm25(d)))
             + w_ann  * (1 / (k + rank_ann(d)))
             + w_graph * (1 / (k + rank_graph(d)))
```

**Default weights:**

| Signal | Weight | Notes |
|--------|--------|-------|
| BM25 | 0.4 | Strongest for code and exact terminology. |
| ANN | 0.4 | Equal weight — semantic match is as important as keyword. |
| Graph | 0.2 | Lower weight — graph is supplementary context, not primary. |

**Constant `k`:** 60 (standard RRF parameter to dampen high-rank outliers).

Weights are configurable per query in `rag.retrieve(query, { weights: {...} })`.

## Context Assembly

After fusion, the top-ranked chunks are assembled into a prompt context:

```typescript
function assemble(chunks: Chunk[], budget: number): string {
  // 1. Sort by RRF score descending
  chunks.sort((a, b) => b.rrf_score - a.rrf_score);

  // 2. Deduplicate by content hash
  chunks = deduplicate(chunks);

  // 3. Greedily fill token budget
  let context = "";
  for (const chunk of chunks) {
    if (countTokens(context + chunk.text) > budget) break;
    context += chunk.text + "\n\n---\n\n";
  }

  return context;
}
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `budget` | 4096 tokens | Max tokens for assembled context. |
| `max_chunks` | 15 | Hard limit on number of chunks (even if within budget). |
| `deduplicate` | true | Hash-based dedup with SHA-256 of chunk text. |

## Chunking Strategy

Documents are segmented before indexing:

| Parameter | Default | Description |
|-----------|---------|-------------|
| **Max chunk tokens** | 512 | Approximate — chunk at sentence boundaries. |
| **Overlap** | 64 tokens | Sliding window overlap between adjacent chunks. |
| **Strategy** | Recursive text split | Attempts to split at paragraph → sentence → token boundaries. |
| **Code chunks** | 128 tokens | Smaller chunks for code content (functions, methods). |

Chunks are stored in the Knowledge System and referenced by the BM25 FTS5 index and the Vector Store.

## Interfaces

| Interface | Description |
|-----------|-------------|
| `rag.retrieve(query, opts?)` | Full hybrid retrieval. Returns `Chunk[]` with RRF scores. |
| `rag.rerank(chunks, query)` | Re-rank a list of chunks against a query using cross-encoder (if available) or RRF. |
| `rag.assemble(chunks, budget)` | Assemble top chunks into a prompt-ready context string. |
| `rag.search_mode(mode)` | Switch between `hybrid` (default), `keyword_only`, `semantic_only`, or `graph_only`. |
| `rag.stats()` | Return query latency breakdown, cache hit rates, chunk counts. |

## Failure Modes

| Failure | Symptom | Recovery |
|---------|---------|----------|
| **No results** | All three retrievers return empty. | Return a clear "no relevant context found" message. The agent can fall back to its own knowledge. |
| **Embedding unavailable** | ANN step fails (model down, timeout). | Skip ANN, run BM25 + Graph only. Log the embedding failure. |
| **Context overflow** | Assembled context exceeds token budget. | Enforce hard truncation at the token budget. The earliest (highest-scored) chunks are preserved. |
| **Graph engine unavailable** | Graph expansion step fails. | Skip graph expansion, run BM25 + ANN only. |
| **BM25 index stale** | FTS5 index hasn't been updated after a write. | Trigger a manual `kb.fts.rebuild()` or rely on the 60-second auto-refresh. |

## Observability Metrics

| Metric | Type | Labels |
|--------|------|--------|
| `rag_queries_total` | Counter | mode (hybrid/keyword/semantic/graph), status (success/partial/failure) |
| `rag_retrieval_latency_seconds` | Histogram | step (bm25/ann/graph/fusion/assembly) |
| `rag_chunks_retrieved` | Histogram | source (bm25/ann/graph) |
| `rag_context_tokens` | Histogram | status (truncated/full) |
| `rag_no_results_total` | Counter | reason (all_empty/budget_exceeded) |

## Related Documents

| Document | Description |
|----------|-------------|
| [Knowledge System](KNOWLEDGE_SYSTEM.md) | Main KB and Global KB architecture |
| [Vector Store](VECTOR_STORE.md) | ANN vector index specification |
| [Embeddings](EMBEDDINGS.md) | Embedding generation pipeline |
| [Obsidian Graph Engine](OBSIDIAN_GRAPH.md) | Graph-based knowledge navigation |
| [Persistent Memory](PERSISTENT_MEMORY.md) | Long-term memory backed by RAG |
| [Agent Memory](AGENT_MEMORY.md) | Per-agent memory tiers |
