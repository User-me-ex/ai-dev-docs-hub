# Reliability

> Reliability targets, failure mode analysis, resilience patterns, and SLA commitments for AI Dev OS. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

AI Dev OS is built on a **graceful degradation over hard failure** philosophy. Every subsystem is designed to detect when its dependencies are unhealthy, enter a degraded-but-operational state, and recover without data loss when health is restored. A local developer workstation should never crash because NATS is unreachable; a production cluster should never serve stale data because a vector index rebuild is in progress.

The reliability architecture has four layers:

1. **Resilience patterns** — circuit breakers, bulkheads, retries, timeouts built into every subsystem.
2. **Degradation modes** — defined degraded states per subsystem with automatic recovery.
3. **Error budgets** — measurable SLO violations that trigger operational responses.
4. **Disaster recovery** — RPO/RTO targets with tested recovery procedures.

## SLO Targets

All latency targets are measured from the server's perspective (wall-clock time between request arrival and response). Uptime is measured in calendar months.

| Operation | Target | Measurement period | Severity |
|-----------|--------|--------------------|----------|
| Kernel loop (goal → delivered) p99 | < 60 s | 1 day | Critical |
| SCE publish p99 | < 50 ms (NATS), < 200 ms (SQLite) | 1 hour | Critical |
| Model discovery refresh p99 | < 30 s | 1 day | High |
| Memory query (relational) p95 | < 100 ms | 1 hour | High |
| Memory query (semantic/vector) p95 | < 200 ms | 1 hour | High |
| Guardian check p95 | < 5 s | 1 day | High |
| Merge commit p95 | < 10 s | 1 day | High |
| Queue claim latency p99 | < 100 ms | 1 hour | Medium |
| Object store PUT p99 | < 1 s (local), < 5 s (S3) | 1 day | Medium |
| Uptime (multi-server) | 99.9% | 1 month | Critical |
| Uptime (single-server) | 99.5% | 1 month | Medium |

SLOs are enforced by the [Observability](./OBSERVABILITY.md) subsystem. Violations are recorded as structured events and contribute to the error budget.

## Resilience Patterns

### Circuit breakers

Every outbound dependency (database, NATS, S3, model provider, MCP server) is wrapped in a circuit breaker with three states:

| State | Behavior | Transition |
|-------|----------|------------|
| **Closed** | Requests pass through normally | On failure count > `failure_threshold` (default 5, 60 s window) → **Open** |
| **Open** | Requests fail immediately (fast rejection) | After `reset_timeout_ms` (default 30 s) → **Half-open** |
| **Half-open** | Limited requests allowed (default 1) | On success → **Closed**; on failure → **Open** |

Circuit breaker events are published on the SCE: `circuit_breaker.{closed|open|half_open}`.

### Bulkheads

Critical subsystems run in isolated resource pools:

```
HTTP server     → fixed thread pool (default: 4 * CPU cores)
Kernel runs     → separate goroutine/process per run
Model I/O       → bounded connection pool per provider
SCE broker      → dedicated event loop
Queue consumer  → worker pool with configurable concurrency
```

A model provider that hangs MUST NOT block the HTTP server. A kernel run that enters an infinite loop MUST NOT starve the queue consumer.

### Retry with backoff

All transient operations use truncated exponential backoff:

```
delay_ms = min(base * 2^attempt + jitter, max_delay)
jitter    = uniform(0, base * 0.1)

base      = 100 ms (default, per-operation configurable)
max_delay = 30 s
max_attempts = 5
```

Operations that are idempotent (SCE publish, memory write, object PUT) use at-most-once delivery with retry. Operations that are non-idempotent carry an idempotency key and use at-least-once delivery.

### Timeout envelopes

Every external call has a timeout and a "timeout margin" for circuit breaker tripping:

```
sqlite_query_timeout_ms   = 5,000
postgres_query_timeout_ms = 10,000
nats_publish_timeout_ms   = 2,000
s3_put_timeout_ms         = 30,000
model_inference_timeout_s = 120
health_check_timeout_ms   = 3,000
```

If a call exceeds its timeout, the caller receives a `DEADLINE_EXCEEDED` error and the circuit breaker records a failure.

### Fallback chains

When the primary implementation of a subsystem fails, a fallback is tried:

| Subsystem | Primary | Fallback 1 | Fallback 2 |
|-----------|---------|------------|------------|
| SCE | NATS JetStream | SQLite WAL | Local buffer (in-memory) |
| Vector index | pgvector | usearch (embedded) | FTS (SQLite full-text search) |
| Object store | S3 | Local filesystem | Temp directory + background sync |
| Model provider | Primary (e.g., OpenAI) | Fallback (e.g., Anthropic) | Local model (Ollama) |
| Auth | OIDC | JWT local | OS keychain (dev only) |

Each fallback MUST maintain the same interface contract. The subsystem emits `{subsystem}.fallback_activated` on the SCE.

### Checkpoint/replay

Kernel runs checkpoint their state at each lifecycle transition (see [Agent Lifecycle](./AGENT_LIFECYCLE.md)):

```
intake → [checkpoint] → plan → [checkpoint] → route → [checkpoint]
→ execute → [checkpoint] → critique → [checkpoint] → merge → [checkpoint]
→ guard → [checkpoint] → deliver
```

On restart, the Kernel replays from the last checkpoint. If a crash occurs during `execute`, the run resumes at the `execute` checkpoint and re-invokes the agent group. Checkpoints are stored in the Persistent Memory subsystem.

### WAL buffering

SQLite WAL buffers writes before flushing to disk. The server triggers a WAL checkpoint:

- Every 1000 SCE events (configurable `wal_checkpoint_interval`).
- On graceful shutdown (see [Backend](./BACKEND.md#graceful-shutdown)).
- When `wal_size_pages` exceeds `wal_max_size_pages` (default 10,000).

## Degradation Modes

| Degraded subsystem | What still works | What degrades | Recovery action |
|--------------------|------------------|---------------|-----------------|
| SCE (NATS down, on SQLite fallback) | All operations | SCE publish latency increases; cross-server subscriptions lost; no SSE tail across pods | NATS reconnects automatically; buffer replays to NATS when healthy |
| Vector index (rebuilding) | Relational memory queries; FTS memory queries | Semantic/vector queries return empty results or fall back to FTS | Index completes; `vector.ready` event emitted |
| Object store (S3 down, on local FS) | All operations | Objects stored locally, not replicated; risk of data loss if pod dies | Background sync to S3 when healthy; emit `objects.syncing` |
| Model provider (rate-limited) | All other providers | Requests to rate-limited provider queue; fallback provider handles role | Backoff timer expires; provider marked healthy |
| Database (connection pool saturated) | Cached reads; SCE events (in buffer) | New writes block; some reads fail | Connection returns to pool; pool size scales |
| Postgres replica lagging | Reads from primary | Read queries may return stale data (up to `max_replica_lag_ms`) | Read pool falls back to primary for lag-sensitive queries |
| Disk space warning (>80%) | Reads; writes to other disks | Writes to affected disk may fail; WAL checkpoint paused | Alert; auto-cleanup old SCE events; archive to object store |

## Error Budgets

Error budget is calculated monthly:

```
error_budget = (1 - slo_target) * total_operations
burn_rate    = slo_violations / error_budget
```

| SLO target | Monthly error budget (1M operations) | Daily budget |
|------------|--------------------------------------|--------------|
| 99.9%      | 1,000 violations | ~33 violations |
| 99.5%      | 5,000 violations | ~167 violations |
| 99.0%      | 10,000 violations | ~333 violations |

### Budget depletion actions

| Burn rate | Window | Action |
|-----------|--------|--------|
| > 100% / day | 1 hour | Page on-call; freeze non-critical deployments |
| > 50% / day | 6 hours | Create P1 incident; begin root cause analysis |
| > 10% / day | 24 hours | Log for weekly reliability review |
| < 5% / day | 30 days | No action (healthy) |

When error budget is exhausted for a critical SLO, all feature deployments are blocked until the budget recovers (next month) or a remediation plan is approved by the Architecture Guardian.

## Disaster Recovery

| Data store | RPO | RTO | Recovery method |
|------------|-----|-----|-----------------|
| SQLite (local mode) | 5 min | 10 min | WAL file + snapshot restore from backup |
| Postgres (primary) | 1 min | 5 min | WAL streaming replica promote |
| Postgres (read replicas) | 5 s | 30 s | Load balancer failover to another replica |
| NATS JetStream | 0 (synchronous replication) | 1 min | Leader election among 3-node cluster |
| S3 objects | 0 (versioning enabled) | 5 min | S3 Cross-Region Replication or bucket restore |
| Vector index (pgvector) | 1 min | 15 min | Rebuild from embeddings in Postgres |
| Embedded vector index (usearch) | 5 min | 30 min | Rebuild from embeddings in SQLite |

### Recovery procedure (catastrophic loss of primary Postgres)

```
1. Promote standby to primary (pg_ctl promote / kubectl label)
2. Point aidevos-server to new primary via AIDEVOS_DB_URL
3. Verify SCE event continuity from NATS JetStream (no events lost)
4. Rebuild pgvector index from stored embeddings
5. Verify object store access (S3 unaffected)
6. Run `aidevos-server doctor --full` to validate all subsystems
7. Emit `dr.recovery_complete` event on SCE
```

## Reliability Testing

### Chaos engineering

The following chaos experiments are run weekly against the staging environment:

| Experiment | Description | Success criteria |
|------------|-------------|-----------------|
| NATS cluster kill | Stop 2 of 3 NATS nodes simultaneously | SCE falls back to SQLite; no events lost; service remains available |
| Postgres primary failover | Force pg_ctl promote on standby, kill primary | Read replicas serve; writes queue briefly; no data loss |
| Pod random kill | Kill 30% of aidevos-server pods every 60 s for 5 min | HPA replaces pods; in-flight runs resume from checkpoint |
| Network partition | Block traffic between worker pods and database for 120 s | Circuit breakers open; workers enter drain state; DB reconnection succeeds |
| Disk fill | Fill 95% of the data volume | Alerts fire; auto-cleanup runs; writes to S3 unaffected |
| CPU throttling | Limit worker pods to 0.1 CPU | Workers slow but don't crash; queue backpressure engages |
| Model provider latency | Inject 60 s delay on model API responses | Circuit breaker opens after 5 failures; fallback provider used |

### Fault injection API

The server exposes a `/__chaos` endpoint (disabled in production) for programmatic fault injection:

```
POST /__chaos/inject
{
  "subsystem": "database",
  "fault": "connection_timeout",
  "probability": 0.1,
  "duration_ms": 30000
}
```

### Jepsen-style consistency tests

The [Eval Harness](./EVAL_HARNESS.md) includes linearizability tests for:

- SCE event ordering within a partition (single-producer, single-consumer).
- Memory record read-after-write consistency.
- Queue at-least-once delivery under partition.

## Failure Modes

| Mode | Detection | Response | SLO impact |
|------|-----------|----------|------------|
| SCE event loss (NATS disk full) | NATS `disk_full` alert | Switch to SQLite SCE; replay buffered events on recovery | SCE publish latency SLO violated |
| Postgres WAL corruption | Replication lag spikes; CHECKSUM errors | Promote standby; run pg_verify_checksums | DB query latency SLO violated; RPO may be exceeded |
| Memory record corruption | CRC mismatch on read | Return error; log corrupted record ID; repair from SCE event replay | Memory query SLO violated for affected records |
| Worker goroutine leak | RSS grows unbounded; `num_goroutines` metric | Kill and restart worker; emit `worker.leak_detected` | Worker capacity reduced; queue depth grows |
| Deadlock in queue pop | Queue depth unchanged; in_flight stuck | `std_syscall` timeout of 10 s triggers crash recovery; queue items re-queued via visibility timeout | Queue latency SLO violated |
| MCP server disconnect | MCP ping timeout | Reconnect with backoff; emit `mcp.disconnected`; serve cached tools/resources | Tools unavailable for duration; agent falls back to tool-less plan |
| Clock skew (> 1 s) | SCE event timestamps inconsistent | Log warning; use monotonic clock for ordering; reject events with future timestamps | Event ordering ambiguity; SCE publish SLO may violate |
| OOM kill (Kubernetes) | Container exit code 137 | HPA restarts pod; in-flight runs resume from checkpoint | Uptime SLO impacted; run latency SLO may violate |
| TLS certificate expiry | /healthz reports `tls_expiry` in < 30 days | Alert; auto-renew via cert-manager | Uptime SLO violated if expiry causes connection drops |
| Concurrent schema migration race | Migration version mismatch between pods | Locks via Postgres advisory lock; second pod waits up to 5 min then exits | Startup sequence delayed; uptime SLO may violate |

## Observability for Reliability

### Alerting thresholds

| Alert | Condition | Severity | Response |
|-------|-----------|----------|----------|
| High SCE latency | SCE publish p99 > 100 ms for 5 min | Critical | Check NATS cluster; tune partitions |
| Error budget depletion | Burn rate > 100% / hour | Critical | Page on-call |
| Database connection pool saturation | `pool_idle_connections` = 0 for 2 min | High | Increase `pool_size`; check for slow queries |
| Queue depth growing | `queue_depth` > `soft_cap` and rising for 5 min | High | Scale workers; check for stuck consumer |
| Circuit breaker open | Any circuit breaker open for > 1 min | Warning | Investigate failing dependency |
| Disk space < 20% free | `disk_free_bytes / disk_total_bytes` < 0.2 | Critical | Alert on-call; trigger cleanup |
| Replica lag > 5 s | `pg_replica_lag_seconds` > 5 | High | Fall back to primary; investigate replication |
| Process RSS > 80% limit | `process_rss_bytes / memory_limit_bytes` > 0.8 | Warning | Tune worker limits; scale up pod |

### On-call runbooks

Each runbook covers:

1. Symptoms (dashboard links, metric queries).
2. Immediate mitigation steps (fast button).
3. Root cause diagnosis procedure.
4. Permanent fix deployment.
5. Post-mortem template.

Runbooks are maintained in the `runbooks/` directory and linked from alert notifications.

## Related Documents

- [Scalability](./SCALABILITY.md) — scaling dimensions, limits, load testing
- [Performance](./PERFORMANCE.md) — latency benchmarks, throughput profiling
- [Disaster Recovery](./DISASTER_RECOVERY.md) — RPO/RTO procedures, backup verification
- [Backup Strategy](./BACKUP_STRATEGY.md) — backup schedule, retention, restore testing
- [Observability](./OBSERVABILITY.md) — metrics, traces, logs, alerting
- [Deployment](./DEPLOYMENT.md) — topology, health checks, TLS, migration
- [Backend](./BACKEND.md) — startup/shutdown sequence, graceful drain
- [Error Handling](./ERROR_HANDLING.md) — error types, propagation, user-facing messages
- [Testing Strategy](./TESTING_STRATEGY.md) — chaos engineering, Jepsen tests, load tests
- [System Overview](./SYSTEM_OVERVIEW.md)
- [Main AI Kernel](./MAIN_AI_KERNEL.md)
- [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md)
