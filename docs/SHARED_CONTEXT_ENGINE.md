# Shared Context Engine

> The durable, subscribable **source of truth** shared by every agent, subsystem, and UI. If it isn't on the Context Engine, it didn't happen.

## Overview

The Shared Context Engine (SCE) is the append-only event log plus snapshot store that carries every meaningful state transition in AI Dev OS. It replaces ad-hoc shared memory, per-agent scratchpads, and cross-service RPC state. Every subsystem in this repo publishes to and reads from the SCE — the [Main AI Kernel](./MAIN_AI_KERNEL.md) enforces this as an invariant.

## Goals

- One log, strong per-topic ordering, at-least-once delivery.
- Snapshots for fast recovery and cheap catch-up subscribers.
- Pluggable backends behind a stable interface: SQLite (local, default), Postgres (remote), NATS/JetStream (streaming), Kafka (large deployments).
- Fully local-first — SCE works with zero network dependencies.
- Deterministic replay for any single topic or correlation id.

## Non-Goals

- Free-form key/value storage — use [Persistent Memory](./PERSISTENT_MEMORY.md) for durable knowledge.
- Large-blob transport — publish a pointer to [Vector Store](./VECTOR_STORE.md) or object storage instead.
- Cross-tenant fan-out — tenants get isolated topic namespaces; there is no global bus.

## Requirements

- **MUST** guarantee per-topic FIFO ordering.
- **MUST** deliver every accepted event at least once to every live subscriber.
- **MUST** support snapshot + tail: a new subscriber gets `(snapshot, offset)` and then a live stream from `offset`.
- **MUST** attach `correlation_id` and `causation_id` to every event.
- **MUST** authenticate publishers and enforce topic ACLs.
- **SHOULD** dedupe by `event.id` on the read path.
- **MAY** offer topic compaction for last-writer-wins projections.

## Architecture

```mermaid
flowchart LR
  A[Agent A] -- publish --> BUS[[SCE Broker]]
  B[Agent B] -- publish --> BUS
  BUS --> LOG[(Event Log)]
  BUS --> SNAP[(Snapshot Store)]
  BUS -- subscribe --> A
  BUS -- subscribe --> B
  LOG --> MEM[(Persistent Memory)]
  LOG --> GRAPH[(Obsidian Graph)]
  BUS --> AUDIT[(Audit Log)]
```

See also [diagrams/CONTEXT_ENGINE](../diagrams/CONTEXT_ENGINE.md).

## Interfaces

```
ctx.publish(topic, event) → { id, offset }
ctx.subscribe(topic, from?: offset|"snapshot") → AsyncIterator<Event>
ctx.snapshot(topic) → { offset, state }
ctx.query(topic, filter) → Event[]        // bounded, indexed
ctx.ack(topic, offset)                    // durable cursor for a subscriber
ctx.compact(topic, keyFn)                 // last-writer-wins projection
```

Envelope and error rules come from [Agent Communication](./AGENT_COMMUNICATION.md) and [API Spec](./API_SPEC.md).

## Data Model

```
Event {
  id:             ulid            # monotonic per broker
  topic:          string          # e.g. "run.<ulid>", "router.assignments"
  ts:             rfc3339
  actor:          { id, role }
  correlation_id: uuid            # groups events for one user goal
  causation_id:   ulid|null       # id of the event that caused this one
  schema:         { name, version } # JSON Schema handle
  payload:        object
  sig:            base64          # Kernel or agent signature
}
```

### Topic conventions

| Topic pattern          | Purpose                                    | Retention             |
| ---------------------- | ------------------------------------------ | --------------------- |
| `run.<run_id>`         | Kernel loop events for a single run        | 90d, then compacted   |
| `router.assignments`   | Model role assignments and changes         | forever, compacted    |
| `models.discovery`     | Model discovery reports                    | 30d                   |
| `guardian.verdicts`    | Architecture Guardian outputs              | forever               |
| `group.<group_id>`     | AI Group coordination events               | 90d                   |
| `memory.writes`        | Persistent memory write log                | forever               |

Retention is governed by [Data Retention](./DATA_RETENTION.md).

## Delivery & Ordering

- Per-topic FIFO. Cross-topic order is not guaranteed.
- At-least-once; consumers MUST be idempotent by `event.id`.
- Subscriber cursors are durable via `ctx.ack`; unacked events replay on reconnect.
- Backpressure: publishers see `429 Slow` when a topic exceeds its soft cap; the Kernel converts this into worker throttling.

## Failure Modes

| Mode                     | Response                                                                 |
| ------------------------ | ------------------------------------------------------------------------ |
| Backend unavailable      | Publishers write to local WAL; broker replays on recovery                |
| Slow subscriber          | Cursor lags; broker warns after `slow_subscriber_seconds` threshold      |
| Corrupt event            | Quarantine to `deadletter.<topic>`; alert; do not block topic            |
| Snapshot drift           | Rebuild snapshot from log; mark snapshot `provisional` during rebuild    |
| Clock skew across nodes  | Broker rewrites `ts` with broker-side monotonic clock; original preserved |

Every failure is mirrored to the [Audit Log](./AUDIT_LOG.md).

## Security

- Publishers authenticate via [Auth System](./AUTH_SYSTEM.md); topic ACLs via [AuthZ/RBAC](./AUTHZ_RBAC.md).
- Payloads are encrypted at rest per [Encryption](./ENCRYPTION.md); sensitive fields MAY be tokenized.
- Signatures on every event enable end-to-end verification even across pluggable backends.
- Tenants are isolated at the topic-namespace level; no cross-tenant subscribe.

## Observability

Metrics: `sce_publish_total{topic}`, `sce_deliver_lag_seconds{topic,subscriber}`, `sce_deadletter_total{topic}`, `sce_snapshot_build_seconds{topic}`. Full guidance in [Observability](./OBSERVABILITY.md) and [Metrics](./METRICS.md).

## Acceptance Criteria

- 10k events/s on SQLite backend on commodity laptop, p99 publish < 5 ms.
- Cold subscriber attaches to a 1 M-event topic via `snapshot + tail` in < 2 s.
- Killing the broker mid-publish loses no acknowledged event on restart.
- Replay of a `correlation_id` reproduces the exact ordered sequence.

## Open Questions

- Whether to expose transactional multi-topic publish or keep single-topic atomicity only.
- Compaction policy for `run.*` topics after `terminated` — hard delete vs. cold archive.

## Related Documents

- [Event Bus](./EVENT_BUS.md) · [Persistent Memory](./PERSISTENT_MEMORY.md) · [Audit Log](./AUDIT_LOG.md) · [Agent Communication](./AGENT_COMMUNICATION.md) · [Main AI Kernel](./MAIN_AI_KERNEL.md) · [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) · [diagrams/CONTEXT_ENGINE](../diagrams/CONTEXT_ENGINE.md)
