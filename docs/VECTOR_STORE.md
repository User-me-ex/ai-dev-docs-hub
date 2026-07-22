# Vector Store

> **Domain:** Vector Index Specification and Management
> **Applies to:** Persistent Memory, RAG Pipeline, Embeddings
> **Last updated:** 2026-07-22

## Overview

The Vector Store is an Approximate Nearest Neighbor (ANN) index that enables semantic search across all embedded content in AI Dev OS. By default it uses **usearch** — a single-file, zero-configuration HNSW index that lives on disk. For production deployments, alternative backends are supported.

## Supported Backends

| Backend | Default | Type | Persistence | Notes |
|---------|---------|------|-------------|-------|
| **usearch** | ✅ Default | Embedded (single-file) | `vector_index.usearch` + `vectors.db` (SQLite) | Zero-config. Ships with the binary. Best for local-first. |
| **Chroma** | — | Embedded | `~/.aidevos/data/chroma/` | Local alternative; memory-friendly |
| **pgvector** | — | PostgreSQL extension | External PG instance | Optional; for multi-workspace shared indexes |
| **Qdrant** | — | Standalone service | Docker / cloud | Optional; for HA deployments |
| **Pinecone** | — | Managed cloud | Pinecone account | Optional; for serverless vector search at scale |

## Local Default: usearch

The embedded usearch index is the default and recommended backend for local-first operation:

- **File:** `~/.aidevos/data/<workspace_id>/vector_index.usearch`
- **Config:** Zero — created on first write, loaded on startup.
- **Durability:** Memory-mapped writes are flushed to disk every 100 write operations.
- **Backup:** The `.usearch` file can be copied while the index is open (safe snapshots).

## Index Configuration

Local defaults (usearch/Chroma) are zero-config — created on first write, loaded on startup. Optional tuning:

```toml
[vector_store]
backend = "usearch"        # usearch | chroma | pgvector | qdrant | pinecone

[index]
metric = "cosine"          # One of: cosine, l2, ip
dimensions = 768           # Must match the embedding model
M = 16                     # HNSW M parameter (connections per node)
ef_construction = 200      # HNSW ef_construction (build quality)
ef_search = 64             # Default ef_search for ANN queries

[persistence]
save_interval = 100        # Flush to disk every N writes
auto_rebuild_interval = "24h"  # Full rebuild from relational store daily

[limits]
max_vectors = 10_000_000   # Max vectors per index
max_vector_bytes = 10_737_418_240  # ~10 GB
```

> **Note:** Chroma and usearch are the default local backends. pgvector is an optional alternative for multi-workspace deployments. Cloud vector stores (Pinecone, Qdrant cloud) are optional and configured independently.

### HNSW Parameters

| Parameter | Default | Effect |
|-----------|---------|--------|
| `M` | 16 | Number of bi-directional links per node. Higher = better recall, more memory. Range: 4–64. |
| `ef_construction` | 200 | Dynamic candidate list during build. Higher = better quality, slower build. Range: 100–800. |
| `ef_search` | 64 | Candidate list during search. Higher = better recall, slower query. Can be overridden per query. |

## CRUD Operations

| Operation | Interface | Description |
|-----------|-----------|-------------|
| **Insert** | `store.upsert(vectors, metadata[])` | Adds vectors. If an ID exists, updates in place. |
| **Read** | `store.get(id)` | Returns vector + metadata by ID. |
| **Update** | `store.upsert(vectors, metadata[])` | Same as insert — upsert semantics. |
| **Delete** | `store.delete(ids)` | Marks as deleted in the index and removes from the relational store. |
| **List** | `store.list(filter, limit, offset)` | Paginated list with optional metadata filter. |

### Insert / Upsert

```typescript
await store.upsert([
  { id: "doc-001", vector: float32[768], metadata: { source: "main_kb", chunk_id: 42 } },
  { id: "doc-002", vector: float32[768], metadata: { source: "agent_memory", agent: "alpha" } },
]);
```

- Vectors must be L2-normalized `float32` arrays.
- Metadata is a flat key-value map (values: string, number, boolean, null).
- Upsert is atomic within a batch: all succeed or all fail.

## ANN Search

```typescript
const results = await store.search(query, {
  top_k: 10,
  ef_search: 128,       // Override default (64)
  filters: { source: "main_kb" },
  min_score: 0.7,       // Cosine similarity threshold
});
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `top_k` | 10 | Number of results to return. |
| `ef_search` | 64 | HNSW search breadth. Higher = more accurate, slower. |
| `filters` | `{}` | Metadata filter (exact match on key-value pairs). |
| `min_score` | 0.0 | Minimum cosine similarity threshold. |

**Search result:**

```typescript
{
  id: "doc-001",
  score: 0.92,           // Cosine similarity
  metadata: { source: "main_kb", chunk_id: 42 },
  vector: float32[768],  // Omitted by default; include with include_vector: true
}
```

## Index Persistence

```
┌─────────────────────────────────────────────────────────────┐
│                    Vector Store                              │
├──────────────────────┬──────────────────────────────────────┤
│  usearch index       │  SQLite relational store              │
│  (vector_index.      │  (vectors.db)                         │
│   usearch)           │                                       │
│                      │  Tables:                              │
│  HNSW graph          │  ─────────────────                    │
│  on disk,            │  vectors(id, vector_blob, metadata,   │
│  memory-mapped       │           created_at, updated_at)     │
│                      │  index_metadata(key, value)           │
│  Fast ANN search     │                                       │
│  No metadata         │  Used for:                            │
│  No filtering        │  - Metadata storage/filtering         │
│                      │  - Index rebuild source               │
│                      │  - Backup/recovery                    │
└──────────────────────┴──────────────────────────────────────┘
```

- **usearch index:** Memory-mapped, single-file. Holds the HNSW graph structure only. No metadata.
- **Relational store:** SQLite database at `~/.aidevos/data/<ws_id>/vectors.db`. Stores the raw vectors, metadata, and index metadata.

## Index Rebuild

A full rebuild is triggered automatically every 24 hours or manually via `store.rebuild()`:

1. Read all vectors from the relational store (`vectors.db`).
2. Build a new HNSW index from scratch using current `M` and `ef_construction`.
3. Swap the active index atomically (rename + remap).
4. Old index is unmapped and deleted.

Rebuilds are **read-safe** — queries continue on the old index until the swap completes.

## Concurrency Model

- **Single writer:** All upserts and deletes go through a single writer goroutine. Concurrent write calls are serialized.
- **Multiple readers:** The usearch index is memory-mapped, so multiple threads can query simultaneously without locks.
- **Read-write safety:** Writes append to the memory-mapped file; readers may see stale data but never inconsistent data.

## Capacity Limits

| Limit | Default | Maximum | Effect When Exceeded |
|-------|---------|---------|----------------------|
| `max_vectors` | 10,000,000 | 100,000,000 | `store.upsert()` returns `ErrStoreFull`. Relational store still accepts writes. |
| Dimensions | 768 | 2048 | Must match the embedding model. Index creation fails on mismatch. |
| File size | — | ~10 GB | OS-dependent. usearch file is limited by virtual address space. |
| Metadata size | 4 KB per entry | 64 KB per entry | Larger metadata is truncated at write time. |

## Interfaces

| Interface | Description |
|-----------|-------------|
| `store.upsert(entries)` | Insert or update vectors + metadata. |
| `store.delete(ids)` | Delete vectors by ID. |
| `store.search(query_vector, opts)` | ANN search with optional filters and ef_search override. |
| `store.get(id)` | Retrieve a single vector + metadata by ID. |
| `store.rebuild()` | Trigger a full index rebuild from the relational store. |
| `store.stats()` | Return vector count, index size, dimension, last rebuild time. |
| `store.export(path)` | Export the index to a portable format (for migration). |

## Failure Modes

| Failure | Symptom | Recovery |
|---------|---------|----------|
| **Index corruption** | usearch fails to load or returns garbage scores. | Rebuild from relational store (`store.rebuild()`). If relational store is also corrupt, restore from backup. |
| **Disk full** | Write operations fail with `EIO` or `ENOSPC`. | The writer goroutine stops and enters a retry loop (30s interval). Queries continue to work. |
| **Dimension mismatch** | Upsert vector has wrong dimension. | Rejected with `ErrDimensionMismatch`. The entire batch fails atomically. |
| **Memory pressure** | mmap fails or the kernel OOM-kills the process. | Reduce `max_vectors` or switch to pgvector (no mmap). |

## Observability Metrics

| Metric | Type | Labels |
|--------|------|--------|
| `vector_store_vectors_total` | Gauge | backend |
| `vector_store_index_size_bytes` | Gauge | backend |
| `vector_store_search_latency_seconds` | Histogram | backend, filtered (true/false) |
| `vector_store_upserted_total` | Counter | status (success/failure) |
| `vector_store_rebuild_duration_seconds` | Gauge | status |

## Related Documents

| Document | Description |
|----------|-------------|
| [Embeddings](EMBEDDINGS.md) | Vector generation pipeline |
| [Persistent Memory](PERSISTENT_MEMORY.md) | Long-term storage backed by vector search |
| [RAG Pipeline](RAG_PIPELINE.md) | Retrieval combining ANN + BM25 + graph |
| [Knowledge System](KNOWLEDGE_SYSTEM.md) | Main KB and Global KB architecture |
