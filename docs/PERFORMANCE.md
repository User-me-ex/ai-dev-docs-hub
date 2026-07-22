# Performance

> Performance characteristics, targets, and optimization strategies for AI Dev OS.

## Overview

This document defines the performance budget for AI Dev OS — the latency, throughput, and resource utilization targets every subsystem MUST meet. Performance is monitored continuously via the [Observability](./OBSERVABILITY.md) pipeline and enforced as part of the CI gate (see [Testing Strategy](./TESTING_STRATEGY.md)).

## Performance Targets

| Component | Metric | Target | Measurement |
|---|---|---|---|
| Kernel loop | p99 latency | < 5 s | Full intake → deliver cycle |
| SCE publish | p99 latency | < 5 ms | Single event, in-process |
| SCE subscribe | p99 delivery | < 10 ms | First subscriber receives event |
| Memory query | p95 latency | < 100 ms | Hybrid vector + metadata, 10 K index |
| Vector index build | time per 10 K embeddings | < 30 s | HNSW, efConstruction=200 |
| Guardian check | p95 latency | < 500 ms | Full rule set against run context |
| Model API call | p95 time to first token | < 2 s | Excludes provider network latency |
| Worker spawn | p99 cold start | < 3 s | Containerized worker, image cached |
| Graph traversal | p99 10-hop BFS | < 200 ms | 100 K node graph |
| File I/O (config) | p99 read | < 10 ms | TOML/YAML parse into memory |
| SQLite query | p99 simple SELECT | < 5 ms | Indexed, < 1 K rows returned |

## Bottleneck Analysis

Identified bottleneck sources and their impact:

| Bottleneck | Impact | Frequency |
|---|---|---|
| SCE write throughput | Limits multi-agent event throughput | Under high concurrency |
| Vector index rebuild | Blocks memory queries during rebuild | On model catalog changes |
| Model API latency | Dominates total run time | Every run |
| Filesystem I/O (metadata) | Slows startup and config reload | On cold start |
| SQLite write lock | Contention under concurrent writes | Multi-worker sessions |
| Serialization overhead | JSON serialize/deserialize of large contexts | Large SCE events |

## Optimization Strategies

| Strategy | Applied To | Effect |
|---|---|---|
| In-memory LRU cache | Model catalog, KB queries, SCE snapshots | Avoids redundant I/O and recomputation |
| Connection pooling | Model provider HTTP clients | Reduces TLS handshake overhead |
| Async batched writes | SCE events, audit log, metrics | Groups small writes into batch commits |
| Batch vector operations | Embedding insert, index build | Reduces per-vector overhead |
| SQLite WAL mode | Persistent memory, audit log | Concurrent reads during write |
| HNSW index tuning | Vector store | `efConstruction` / `efSearch` tradeoff |
| Response streaming | Model inference | Lowers time-to-first-token |
| Pre-warming | Model catalog, plugin registry | Eliminates cold-start penalty |

## Profiling

Built-in profiling is available via the CLI:

```bash
aidevos doctor --profile              # run profile and print summary
aidevos doctor --profile --flamegraph  # generate flamegraph SVG
aidevos doctor --profile --output profile.json  # raw trace data
```

Profiling runs the Kernel loop and all subsystem entry points on a synthetic workload. Results include per-function CPU time, call count, and allocation size.

Flamegraphs are written to `~/.aidevos/profiles/` and can be viewed in any browser.

## Memory Usage

| Component | Typical Footprint | Notes |
|---|---|---|
| Main Kernel process | 150–300 MB | Includes SCE, scheduler, router |
| Worker process (each) | 80–200 MB | Per-agent session memory |
| SQLite (persistent store) | File-size + 32 MB WAL | Scales with KB size |
| Vector index (10 K embeddings) | ~40 MB | HNSW, 768-dim, float32 |
| Vector index (100 K embeddings) | ~400 MB | Same parameters |
| Plugin host process | 50–150 MB | Per loaded plugin |

## GPU Utilization

| Workload | GPU Memory | Notes |
|---|---|---|
| Model inference (7B param) | 14–18 GB | Half-precision, batch=1 |
| Model inference (70B param) | 40–80 GB | Requires quantization or multi-GPU |
| Embedding generation (batch 32) | 2–4 GB | BERT-family, 768-dim |
| Vector index (HNSW build) | CPU only | GPU not used |

GPU memory is allocated lazily. Use `--gpu` flags on per-command basis to control which operations use the GPU. See [Model Providers](./MODEL_PROVIDERS.md) for provider-specific GPU configuration.

## Optimization Workflow

The recommended workflow for performance optimization follows a measure → identify → optimize → verify loop:

1. **Measure**: Run `aidevos doctor --profile` against a representative workload to establish baseline.
2. **Identify**: Examine the profile output for the top-5 bottlenecks by CPU time or wall-clock time. Compare against the performance targets table above.
3. **Optimize**: Apply one strategy from the optimization table (caching, batching, pooling, index tuning). Avoid making multiple changes simultaneously.
4. **Verify**: Re-run the profile and diff against the baseline with `aidevos doctor --profile --diff <baseline.json>`.
5. **Gate**: Ensure all MUST targets still pass before committing.

Regressions that exceed 10% on any MUST target require a documented trade-off approved by the performance team lead.

## Scalability Bottlenecks

When scaling from single-user to multi-workspace deployments, watch for these bottlenecks:

| Bottleneck | Signs | Remedy |
|---|---|---|
| SQLite write contention | `SQLITE_BUSY` errors in logs | Switch to PostgreSQL for persistent memory |
| SCE event ordering | Out-of-order delivery under load | Enable partitioned topics per workspace |
| Model API rate limits | 429 responses increase with worker count | Distribute across provider API keys |
| Cache churn | Cache miss rate > 30% | Increase cache sizes or add Redis backend |
| Worker startup time | Cold start > 10 s | Pre-pull container images, use worker pools |

## Related Documents

- [Benchmarks](./BENCHMARKS.md) — benchmarking framework and results
- [Scalability](./SCALABILITY.md) — horizontal scaling and throughput
- [Caching Strategy](./CACHING_STRATEGY.md) — caching layers and invalidation
- [Reliability](./RELIABILITY.md) — fault tolerance and degradation
- [Observability](./OBSERVABILITY.md) — metrics collection and dashboards
- [Tracing](./TRACING.md) — distributed trace propagation
