# Caching Strategy

> Caching layers, invalidation policies, and configuration across AI Dev OS.

## Overview

Caching reduces latency, saves API costs, and avoids redundant computation throughout AI Dev OS. Caches exist at multiple levels — in-process memory, local SQLite, and optional Redis — with per-cache TTL, max-size, and invalidation policy. Every cache MISS that could have been a HIT is tracked as a metric (see [Observability](./OBSERVABILITY.md)).

## Where Caching Is Used

Caching is applied to data that is expensive to compute or fetch and changes infrequently relative to read frequency. Write-heavy data (audit log, events) is not cached.

## Cache Locations

| Cache | Key | Value | Backend | TTL | Max Size |
|---|---|---|---|---|---|
| Model catalog | `model:<provider>:<id>` | Model metadata + capabilities | In-memory LRU | 5 min | 500 entries |
| Embedding cache | `embed:<text_hash>` | Embedding vector | SQLite | 24 h | 100 K entries |
| SCE snapshots | `sce:<topic>:<sequence>` | Serialized event batch | In-memory LRU | 30 s | 200 entries |
| Graph traversal | `graph:<query_hash>` | Path result set | In-memory LRU | 60 s | 100 entries |
| KB query results | `kb:<query_hash>:<filters>` | Retrieved chunks + scores | SQLite | 10 min | 5 K entries |
| Dependency map | `dep:<workspace_id>` | Resolved dependency graph | In-memory LRU | 60 s | 50 entries |
| Config (parsed) | `config:<path>` | Parsed TOML/YAML | In-memory LRU | 30 s | 20 entries |
| Provider responses | `provider:<model>:<prompt_hash>` | Cached LLM response | SQLite | 1 h | 10 K entries |

## Cache Invalidation

Three invalidation mechanisms are used, applied per cache as listed in the table above:

### TTL-Based

Expires entries after a fixed duration from write time. Read during expiry returns the stale value while a background refresh fetches new data (stale-while-revalidate). This is the default for all caches.

### Event-Based

Invalidation is triggered by SCE events. When a relevant mutation occurs, the cache subscriber evicts or refreshes affected entries:

| SCE Event | Cache Action |
|---|---|
| `model.catalog.updated` | Evict all model catalog entries |
| `knowledge.base.changed` | Evict KB query results for affected source |
| `workspace.config.changed` | Evict config and dependency map entries |
| `plugin.installed` | Evict model catalog (if plugin adds models) |

### Explicit Refresh

Callers may force a cache refresh by passing `refresh=true` in the query or calling `invalidate(namespace, key)` on the cache API.

## Cache Backends

| Backend | Use Case | Persistence |
|---|---|---|
| In-memory LRU | Hot caches, sub-ms access | None — lost on restart |
| SQLite (local) | Persistent caches, < 10 ms access | Survives restart, bounded size |
| Redis (optional) | Shared cache across processes | Configurable eviction policy |

In-memory LRU uses `ordereddict` with O(1) get/set. SQLite cache uses a single table with `(namespace, key, value, expires_at)` and periodic VACUUM. Redis, when configured, is used as a shared fallback for SQLite caches to allow multi-process cache sharing.

## Cache Configuration

Per-cache configuration is set in `config.toml`:

```toml
[cache.embedding]
backend = "sqlite"
ttl_seconds = 86400
max_entries = 100000

[cache.model_catalog]
backend = "memory"
ttl_seconds = 300
max_entries = 500

[cache.provider_responses]
backend = "sqlite"
ttl_seconds = 3600
max_entries = 10000
```

Global defaults can be overridden per environment:

```toml
[cache]
backend = "redis"  # override all caches to use Redis
redis_url = "redis://localhost:6379/0"
```

## Cache Warming

On startup, the following caches are pre-loaded:

1. **Model catalog** — queried from SQLite persistent store and loaded into the in-memory LRU cache.
2. **Plugin registry** — loaded into memory from disk.
3. **Configuration** — all config files parsed and cached.
4. **Dependency map** — resolved and cached for each active workspace.

Cache warming runs before the Kernel loop starts, ensuring the first run does not pay cold-start penalties.

## Failure Modes

| Failure | Cause | Effect | Mitigation |
|---|---|---|---|
| Stale cache | TTL too long or missed invalidation event | Caller reads outdated data | Reduce TTL, add event-based invalidation |
| Cache miss storm | Cold start or mass eviction | All requests hit upstream | Pre-warming + gradual TTL jitter |
| Cache backend unavailable | SQLite locked or Redis down | Falls through to upstream | Circuit breaker falls back to memory-only |
| Memory pressure | LRU cache grows unbounded | OOM or swap | Enforce max_entries with eviction |
| Thundering herd | Concurrent MISS for same key | Multiple upstream calls | Request coalescing (first caller fetches, others wait) |

## Related Documents

- [Performance](./PERFORMANCE.md) — performance targets and optimization strategies
- [Model Discovery](./MODEL_DISCOVERY.md) — how the model catalog is populated
- [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md) — SCE event-driven invalidation
- [Knowledge System](./KNOWLEDGE_SYSTEM.md) — KB query result caching
- [Configuration](./CONFIGURATION.md) — config file parsing and caching
