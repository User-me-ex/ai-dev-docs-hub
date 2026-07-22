# Caching Strategy

> Caching layers, invalidation policies, and configuration across AI Dev OS. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

Caching reduces latency, saves API costs, and avoids redundant computation throughout AI Dev OS. Caches exist at multiple levels — in-process memory, local SQLite, and optional Redis — with per-cache TTL, max-size, and invalidation policy. Every cache MISS that could have been a HIT is tracked as a metric (see [Observability](./OBSERVABILITY.md)).

The Caching subsystem integrates with the [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md) for event-driven invalidation, the [Cost Management](./COST_MANAGEMENT.md) subsystem for pricing cache cost tracking, and the [Rate Limiting](./RATE_LIMITING.md) subsystem to avoid thundering-herd upstream requests.

## Requirements

- **MUST** expose a `CacheManager` interface with `get`, `set`, `invalidate`, and `warm` operations.
- **MUST** support staleness handling via stale-while-revalidate with separately configurable `serve_stale_secs` and `background_refresh_secs`.
- **MUST** coalesce concurrent MISSes for the same key so that only one caller fetches upstream data.
- **MUST** emit a structured SCE event for every invalidation action.
- **MUST** enforce `max_entries` limits with LRU eviction for in-memory backends.
- **SHOULD** fall back through a backend chain (memory → SQLite → Redis → upstream) on backend failure.
- **SHOULD** support serialization format negotiation (JSON, MessagePack, protobuf) per cache namespace.
- **MAY** distribute cache invalidation across processes via SCE event subscriptions.
- **MUST** prevent cache poisoning by validating that stored values match the expected schema on retrieval.
- **MUST** record hit rate, miss rate, stale-served count, eviction count, and backend latency as metrics.

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
| Pricing rates | `price:<provider>:<model>` | Rate card entry | SQLite | 5 min | 200 entries |
| Tool schemas | `tool:<tool_name>:<version>` | Compiled JSON Schema | In-memory LRU | 1 h | 100 entries |

## Interfaces

### CacheManager

```typescript
interface CacheManager {
  /** Retrieve a value. Returns null on MISS. Optionally serves stale data. */
  get<T>(namespace: string, key: string, options?: CacheGetOptions): Promise<CacheResult<T> | null>;

  /** Store a value with the given TTL. */
  set<T>(namespace: string, key: string, value: T, options?: CacheSetOptions): Promise<void>;

  /** Remove a single entry or all entries in a namespace. */
  invalidate(namespace: string, key?: string): Promise<void>;

  /** Pre-load entries into the cache. */
  warm(namespace: string, entries: Array<{ key: string; value: unknown }>): Promise<void>;

  /** Return the current cache statistics for a namespace. */
  stats(namespace: string): Promise<CacheStats>;
}

interface CacheGetOptions {
  allowStale?: boolean;          // default true (serve stale while refreshing)
  refreshThreshold?: number;     // seconds before TTL expiry to trigger async refresh
  coalesce?: boolean;            // default true (coalesce concurrent MISSes)
}

interface CacheSetOptions {
  ttlSeconds?: number;
  serveStaleSeconds?: number;    // how long past TTL to serve stale values
  backgroundRefreshSeconds?: number; // how long before TTL expiry to trigger async refresh
  format?: SerializationFormat;  // default: negotiated per namespace
}

interface CacheResult<T> {
  value: T;
  stale: boolean;                // true if this was a stale-while-revalidate serve
  refreshedAt: string;           // ISO-8601
  ttlExpiresAt: string;          // ISO-8601
  stalenessExpiresAt: string;    // ISO-8601
  backend: CacheBackend;
}

type SerializationFormat = 'json' | 'msgpack' | 'protobuf';

type CacheBackend = 'memory' | 'sqlite' | 'redis';

interface CacheStats {
  namespace: string;
  hits: number;
  misses: number;
  staleServed: number;
  evictions: number;
  entries: number;
  maxEntries: number;
  backendLatencyMs: number;      // p50 over the last 60 s
  backend: CacheBackend;
}
```

## Cache Key Generation

Every cache key is a string of the form `namespace:segments`. The namespace prevents cross-contamination. The segments are derived from the input parameters, hashed with SHA-256 when the combined key would exceed 128 characters.

```
Algorithm: GENERATE_KEY
Input: namespace (str), params (dict)
Output: cache_key (str)

1.  segments ← []
2.  for each (k, v) in sorted(params.items()):
3.      serialized ← SERIALIZE(v, format="canonical")
4.      segments.append(serialized)

5.  raw_key ← namespace + ":" + ":".join(segments)

6.  if LEN(raw_key) > 128:
7.      hash ← SHA256(raw_key)
8.      return namespace + ":" + hash.hex()[:32]
9.  else:
10.     return raw_key
```

The `SERIALIZE` function produces a canonical representation: integers as decimal strings, booleans as `"t"` / `"f"`, strings as-is, arrays as comma-separated values, dicts as sorted `k=v` pairs joined by `&`.

## Staleness Handling

Every cache entry has three time-based thresholds:

```
entry.written_at         // when the value was first stored
entry.ttl_expires_at     // written_at + ttl_seconds
entry.stale_expires_at   // written_at + ttl_seconds + serve_stale_seconds
```

Read behavior depends on the current time relative to these thresholds:

| Time Window | Behavior | `stale` Flag |
|---|---|---|
| `now < ttl_expires_at` | Return fresh value. No background action. | `false` |
| `ttl_expires_at <= now < stale_expires_at` | Return stale value immediately. Schedule async background refresh. | `true` |
| `now >= stale_expires_at` | Treat as MISS. Fetch upstream synchronously. | N/A |

### Stale-While-Revalidate Configuration

Configurable per cache namespace in `config.toml`:

```toml
[cache.provider_responses.staleness]
serve_stale_secs = 300           # serve stale for 5 min past TTL
background_refresh_secs = 60     # begin refresh 60 s before TTL expiry

[cache.model_catalog.staleness]
serve_stale_secs = 60
background_refresh_secs = 30
```

If `serve_stale_secs = 0`, stale reads are never served — the entry behaves as hard-TTL. If `background_refresh_secs = 0`, background refresh is disabled and the entry is synchronously re-fetched at TTL expiry.

## Request Coalescing (Thundering Herd Prevention)

When `coalesce = true` (default), concurrent MISSes for the same key are serialized so that only one caller performs the upstream fetch:

```
Algorithm: COALESCE_FETCH
Input: namespace (str), key (str)
Output: value (T or null)

1.  mutex_key ← namespace + ":" + key + ":inflight"

2.  if ACQUIRE_MUTEX(mutex_key, ttl=5s):
3.      // This caller wins the right to fetch
4.      value ← FETCH_UPSTREAM(namespace, key)
5.      if value is not null:
6.          SET(namespace, key, value)
7.      RELEASE_MUTEX(mutex_key)
8.      SIGNAL_COND(key, value)
9.      return value
10. else:
11.     // Another caller is already fetching — wait for the result
12.     value ← WAIT_ON_COND(key, timeout=5s)
13.     if timeout:
14.         return FETCH_UPSTREAM(namespace, key)  // fallback: fetch anyway
15.     return value
```

The mutex (line 2) uses a process-local `Lock` for in-process coalescing, or Redis `SET NX` for distributed coalescing. The `WAIT_ON_COND` (line 12) uses a per-key condition variable with timeout.

## Cache Invalidation

### TTL-Based

Expires entries after a fixed duration from write time. Read during expiry returns the stale value while a background refresh fetches new data (stale-while-revalidate). This is the default for all caches.

### Event-Based (SCE)

Invalidation is triggered by SCE events. When a relevant mutation occurs, the cache subscriber evicts or refreshes affected entries:

| SCE Event | Cache Namespace(s) Affected | Action | Scope |
|---|---|---|---|
| `model.catalog.updated` | `model_catalog`, `pricing` | Evict all | Global |
| `knowledge.base.changed` | `kb_query_results` | Evict entries for affected source | Source-specific |
| `workspace.config.changed` | `config`, `dependency_map` | Evict entries for workspace | Workspace-scoped |
| `plugin.installed` | `model_catalog`, `tool_schemas` | Evict all | Global |
| `cost.pricing.refreshed` | `pricing` | Evict all provider rates | Global |
| `rate_limit.config.changed` | — | No cache action (not cached) | N/A |
| `run.completed` | `provider_responses` | Evict run-scoped entries | Run-scoped |

### Distributed Invalidation via SCE

In multi-process deployments (Redis backend), cache invalidation propagates through SCE subscriptions:

1. Process A calls `invalidate("model_catalog")`.
2. The local entry is evicted immediately.
3. An SCE event `cache.invalidated` is published with payload `{ namespace: "model_catalog", key: null }`.
4. Process B's SCE subscriber receives the event and evicts its local entry.
5. Process B acknowledges with a `cache.invalidated.ack` event.

The subscriber MUST handle events within 100 ms of publication. If the subscriber's local cache does not contain the key, the event is silently dropped (no-op).

### Explicit Refresh

Callers may force a cache refresh by passing `refresh=true` in the query or calling `invalidate(namespace, key)` on the cache API.

## Cache Backend Selection

The active backend for each namespace is determined by a priority chain:

```
Algorithm: SELECT_BACKEND
Input: namespace (str)
Output: backend (CacheBackend)

1.  preferred ← CONFIG["cache." + namespace + ".backend"]
2.  if preferred is set and AVAILABLE(preferred):
3.      return preferred
4.  if preferred == "redis" and not REDIS_AVAILABLE:
5.      LOG_WARN("Redis unavailable for " + namespace + ", falling back to sqlite")
6.      if AVAILABLE("sqlite"):
7.          return "sqlite"
8.  if preferred == "sqlite" and not SQLITE_AVAILABLE:
9.      LOG_WARN("SQLite unavailable for " + namespace + ", falling back to memory")
10.     return "memory"
11. return "memory"  // ultimate fallback
```

This ensures that a Redis cluster failure degrades to SQLite, and an SQLite lock failure degrades to memory-only, rather than failing open (no cache) or failing closed (error).

## Serialization Format Handling

Each namespace negotiates a serialization format at startup:

```toml
[cache.provider_responses.serialization]
format = "msgpack"       # compact binary

[cache.kb_query_results.serialization]
format = "json"          # human-readable, schema-enforced

[cache.sce_snapshots.serialization]
format = "protobuf"      # fastest, smallest — matching SCE wire format
```

The format is stored in the `CacheManager` per-namespace config and applied transparently. On `set`, the value is serialized before storage; on `get`, it is deserialized after retrieval. Format mismatch on retrieval (e.g., upgrading from JSON to MessagePack) is handled by attempting all registered decoders in priority order and logging a warning.

## Cache Backends

| Backend | Use Case | Persistence |
|---|---|---|
| In-memory LRU | Hot caches, sub-ms access | None — lost on restart |
| SQLite (local) | Persistent caches, < 10 ms access | Survives restart, bounded size |
| Redis (optional) | Shared cache across processes | Configurable eviction policy |

In-memory LRU uses an `ordereddict` with O(1) get/set and O(1) eviction of the least-recently-used entry. SQLite cache uses a single table with `(namespace, key, value, expires_at, stale_expires_at, created_at)` and periodic VACUUM every 10,000 writes. Redis, when configured, is used as a shared fallback for SQLite caches to allow multi-process cache sharing, with `EX` (TTL) and `NX` (coalescing) support.

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
5. **Pricing rates** — loaded from SQLite pricing cache into memory.

Cache warming runs before the Kernel loop starts, ensuring the first run does not pay cold-start penalties. Warming is sequential (order above) so that dependencies are resolved before dependents. Each warm phase emits a `cache.warm.completed` SCE event with the namespace and entry count.

## Failure Modes

| Failure | Cause | Effect | Mitigation | SCE Event |
|---|---|---|---|---|
| Stale cache | TTL too long or missed invalidation event | Caller reads outdated data | Reduce TTL, add event-based invalidation, shorten `serve_stale_secs` | `cache.stale.served` |
| Cache miss storm | Cold start or mass eviction | All requests hit upstream | Pre-warming + gradual TTL jitter (±10%) | `cache.miss_storm.detected` |
| Cache backend unavailable | SQLite locked or Redis down | Falls through to upstream via backend chain | Circuit breaker falls back to memory-only | `cache.backend.fallback` |
| Memory pressure | LRU cache grows unbounded | OOM or swap | Enforce `max_entries` with LRU eviction; set `[cache].max_memory_mb` | `cache.memory.pressure` |
| Thundering herd | Concurrent MISS for same key | Multiple upstream calls | Request coalescing via `COALESCE_FETCH` | — (mitigated silently) |
| Cache poisoning | Attacker writes malicious payload to cache | Stored value deserializes to unexpected object | Validate schema on retrieval; sign cache entries | `cache.poisoning.detected` |
| Serialization mismatch | Format change between write and read | Deserialization failure | Try registered decoders in priority order; log warning | `cache.serialization.mismatch` |
| Distributed invalidation lag | SCE event delayed across processes | Process B serves stale data after invalidation | Monitor `cache.invalidation.lag_seconds`; reduce if > 1 s | `cache.invalidation.lag` |

## Security Considerations

- **Cache poisoning**: An attacker with write access to the cache backend could inject malicious values. Mitigations: (a) validate stored values against the expected TypeScript schema on every `get()`, (b) optionally sign cache entries with an HMAC keyed by a secret from [Secrets Management](./SECRETS_MANAGEMENT.md) — entries with invalid or missing signatures are treated as MISS, (c) the HMAC key is rotated every 24 hours.
- **Side-channel via cache timing**: Cache HIT vs. MISS timing differs by design (~50 µs vs. ~5 ms). To prevent timing side-channels that could leak cache contents, `get()` always adds a random jitter of up to 100 µs to the response time. For security-sensitive namespaces (`config`, `dependency_map`), the jitter floor is 500 µs.
- **Namespace isolation**: One namespace MUST NOT be able to evict entries from another namespace. The `invalidate(namespace)` call validates that the caller holds a capability token for that namespace.
- **Stale data serving**: Stale-while-revalidate MUST respect a maximum staleness ceiling (`[cache.staleness].max_serve_stale_secs`, default 600). Operators can set this to 0 to disable stale serving entirely for security-critical caches.
- **Audit trail**: Every invalidation event and backend fallback is recorded in the [Audit Log](./AUDIT_LOG.md).
- All external calls go through [Model Providers](./MODEL_PROVIDERS.md) or the [Plugin SDK](./PLUGIN_SDK.md) — no ad-hoc network access.

## Observability

All metrics, traces, and logs conform to [Observability](./OBSERVABILITY.md), [Tracing](./TRACING.md), and [Logging](./LOGGING.md). Every cache operation carries a `correlation_id` propagated from the caller.

### Metrics

| Metric Name | Type | Labels | Description |
|---|---|---|---|
| `cache.hits_total` | Counter | `namespace`, `backend` | Count of cache HITs |
| `cache.misses_total` | Counter | `namespace`, `backend` | Count of cache MISSes |
| `cache.hit_ratio` | Gauge | `namespace` | hit / (hit + miss) over the last 60 s |
| `cache.stale_served_total` | Counter | `namespace` | Count of stale values served |
| `cache.evictions_total` | Counter | `namespace`, `backend` | Count of LRU evictions |
| `cache.backend_latency_ms` | Histogram | `namespace`, `backend` | Latency of backend get/set operations |
| `cache.entries` | Gauge | `namespace`, `backend` | Current number of cached entries |
| `cache.invalidation_events_total` | Counter | `namespace` | Count of invalidation events processed |
| `cache.invalidation_lag_seconds` | Gauge | `namespace` | Max lag between SCE event and local eviction |
| `cache.warm_duration_ms` | Histogram | `namespace` | Time to warm a namespace on startup |
| `cache.serialization_mismatch_total` | Counter | `namespace` | Count of deserialization mismatches |

Hit ratio alerting: `cache.hit_ratio < 0.8` for more than 5 minutes triggers a WARNING-level alert.

## Acceptance Criteria

1. **Hit ratio**: Under normal load (no cold starts), every cache namespace maintains hit_ratio >= 0.80 measured over a 5-minute sliding window.
2. **Stale serving**: A read 30 seconds past TTL (but within `serve_stale_secs`) returns the stale value with `stale = true` and triggers a background refresh that completes within 1 second.
3. **Request coalescing**: 50 concurrent requests for the same missing key result in exactly 1 upstream fetch and 49 waiting callers receiving the same result within 100 ms of the fetch completing.
4. **Backend fallback**: When the configured Redis backend is unavailable, the system transparently falls back to SQLite (or memory) and emits a `cache.backend.fallback` event.
5. **Cache invalidation**: Publishing an SCE `model.catalog.updated` event results in all model catalog entries being evicted in all processes within 200 ms.
6. **Max entries enforcement**: When a cache reaches `max_entries`, the next `set()` evicts the least-recently-used entry and increments the eviction counter.
7. **Security — poisoning prevention**: A cache entry containing a value that fails schema validation returns `null` and emits a `cache.poisoning.detected` event.
8. **Security — namespace isolation**: Calling `invalidate` on cache namespace `"A"` does not affect any entries in namespace `"B"`.
9. **Serialization round-trip**: A value stored with `format = "msgpack"` deserializes to an identical value on retrieval, including nested objects and arrays.

All acceptance criteria are testable via the [Eval Harness](./EVAL_HARNESS.md). A change to this document requires a matching update to any dependent doc listed in *Related Documents*.

## Related Documents

- [Performance](./PERFORMANCE.md) — performance targets and optimization strategies
- [Model Discovery](./MODEL_DISCOVERY.md) — how the model catalog is populated
- [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md) — SCE event-driven invalidation
- [Knowledge System](./KNOWLEDGE_SYSTEM.md) — KB query result caching
- [Configuration](./CONFIGURATION.md) — config file parsing and caching
- [Cost Management](./COST_MANAGEMENT.md) — pricing cache cost tracking
- [Rate Limiting](./RATE_LIMITING.md) — upstream rate limiting for cache MISS fetches
- [Security Model](./SECURITY_MODEL.md) — cache poisoning threat model
- [Secrets Management](./SECRETS_MANAGEMENT.md) — HMAC key for signed cache entries
