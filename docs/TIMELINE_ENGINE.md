# Timeline Engine

> Temporal reasoning across events and runs: tracking when things happened, in what order, and why. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

The Timeline Engine is the temporal-reasoning subsystem of AI Dev OS. It ingests events from multiple sources — the [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md) event log, the [Audit Log](./AUDIT_LOG.md), and agent lifecycle events — and exposes a queryable timeline that supports before/after/between lookups, causal chain traversal, and interval algebra.

While the SCE stores events as an append-only log and the Audit Log stores security-relevant records, the Timeline Engine is a **purpose-built index** optimised for temporal queries. It answers questions like:
- "What happened between 14:30 and 14:35 that caused the RAG pipeline to fail?"
- "Show me all events that preceded `job_abcdef` completing with an error."
- "What events overlap with the deployment window?"
- "Plot a Gantt chart of yesterday's agent coordination sequence."

The Timeline Engine does **not** store its own copy of events — it indexes events from upstream sources by time, source, type, and causal relationships.

## Goals

- Fast temporal queries: before, after, between, interval overlap, causal chains.
- Unified view over SCE events, Audit Log entries, and agent lifecycle events.
- Causal relationship tracking via `causation_id` chains.
- Gantt-chart visualisation for debugging sequences and coordination.
- Support for both point-in-time events and duration intervals.

## Non-Goals

- Long-term archival — use the Audit Log and [Data Retention](./DATA_RETENTION.md) policies for that.
- Real-time stream processing — the Timeline Engine indexes events asynchronously from the SCE.
- Replacing the SCE as the source of truth — the Timeline Engine is a read-optimised projection.

## Event Timeline Schema

Every event in the timeline has this shape:

```
TimelineEvent {
  id:              ulid                 # globally ordered by time
  event_type:      string               # e.g. "job.started", "agent.created"
  source:          "sce" | "audit_log" | "agent_lifecycle"
  source_id:       string               # original event ID from source
  timestamp:       rfc3339nano          # when the event happened
  correlation_id:  string               # propagated from Kernel
  causation_id:    string?              # event that caused this one
  interval_end?:   rfc3339nano          # for duration events (e.g. job.running)
  actor:           { kind: "user"|"agent"|"system", id: string }
  summary:         string               # human-readable one-liner
  metadata:        {}                   # source-specific structured data
  tags:            string[]             # for filtering
}
```

The `id` is a ULID whose timestamp component is the event's occurrence time. This means `ORDER BY id` is equivalent to `ORDER BY timestamp`, and `id` can be used as a cursor for pagination without fetching the timestamp column.

## Sources

### Shared Context Engine (SCE)

The Timeline Engine subscribes to the SCE on the `>event.>` topic pattern. Every event published to the SCE with a `correlation_id` is indexed as a `TimelineEvent` with `source: "sce"`. The `event_type` is derived from the SCE topic (e.g. `research.job.completed` → `"research.job.completed"`).

### Audit Log

Security-relevant events from the [Audit Log](./AUDIT_LOG.md) are indexed with `source: "audit_log"`. These include authentication attempts, permission changes, secret access, and configuration mutations. Audit events are always retained per the Audit Log retention policy, regardless of Timeline Engine pruning settings.

### Agent Lifecycle

Events from [Agent Lifecycle](./AGENT_LIFECYCLE.md) — agent creation, assignment, start, pause, resume, error, completion — are published to the SCE and thus flow through the SCE subscription. However, the Timeline Engine also reads historical lifecycle events from the [Agent Lifecycle](./AGENT_LIFECYCLE.md) store on startup to backfill any missed events.

## Timeline Querying

```
timeline.before(timestamp | event_id, opts?)    → TimelineEvent[]
timeline.after(timestamp | event_id, opts?)     → TimelineEvent[]
timeline.between(start, end, opts?)             → TimelineEvent[]
timeline.causal_chain(event_id)                 → TimelineEvent[]   # walks causation_id DAG
timeline.interval_overlap(interval, opts?)      → TimelineEvent[]   # events with interval_end
timeline.aggregate(query, interval, opts?)      → { bucket, count }[]
```

### Options

```
opts {
  source?:      "sce" | "audit_log" | "agent_lifecycle" | string[]
  event_type?:  string | string[]
  actor_kind?:  "user" | "agent" | "system"
  tags?:        string[]
  correlation_id?: string
  limit?:       number (default 100, max 1000)
  cursor?:      string (ULID, for pagination)
  order?:       "asc" | "desc" (default "desc")
}
```

### Causal Chain Queries

The `causal_chain` query walks the `causation_id` graph backwards from a given event:

```
timeline.causal_chain("event_a")
  → [event_a, event_d, event_g]   # direct causation
  → [event_b, event_e]            # sibling causation (same correlation_id, same source
  →                                # but different causation parent)
```

The engine returns a DAG where edges are `causation_id` links. The caller (typically an agent or the UI) renders the chain as a tree or list.

## Integration with SCE and Audit Log

```
SCE (topic: >event.>)
  │  subscribe
  ▼
Timeline Engine
  │  index → SQLite / Postgres timeline tables
  │
  ├── timeline.query() ←── Agents, UI
  │
  └── timeline.causal_chain() ←── Debugging tools

Audit Log
  │  bulk import (every 60 s)
  ▼
Timeline Engine
  │  index with source: "audit_log"
```

The SCE subscription is **live** — events are indexed within `index_latency_ms` (target < 100 ms). Audit Log entries are imported in batches every 60 s because they are typically queried in bulk for forensics rather than real-time dashboards.

## Use Cases

### Debugging Run Failures

When a job fails, an agent can run:

```
chain = timeline.causal_chain(event_id_of_job_failed)
chain_before = timeline.before(event_id_of_job_failed, { source: "sce", limit: 20 })
```

This returns the causal chain that led to the failure and the 20 preceding SCE events, giving full context for diagnosis.

### Analysing Coordination Sequences

The Timeline Engine can produce a Gantt view of agent coordination:

```
coordination_events = timeline.between(start, end, {
  event_type: ["agent.created", "agent.started", "agent.completed",
                "agent.message_sent", "agent.message_received"],
  order: "asc"
})
```

These events are passed to the visualisation layer to render the Gantt chart.

### Understanding System Evolution

Aggregated queries over time windows reveal system behaviour:

```
hourly_jobs = timeline.aggregate({ source: "sce", event_type: "job.*" }, "1h", { limit: 72 })
```

This returns the last 72 hours of job activity bucketed hourly, useful for spotting load patterns or anomaly spikes.

## Timeline Visualisation

The Timeline Engine provides data for two visualisation modes:

### Mermaid Gantt

```
timeline.to_gantt(events) → mermaid_gantt_string

gantt
    title Agent Coordination — 2026-07-22
    dateFormat  YYYY-MM-DDTHH:mm:ss
    axisFormat  %H:%M

    section Research
    Research Job r1     :done, 2026-07-22T14:00:00, 2026-07-22T14:02:30
    Internet Search     :done, 2026-07-22T14:01:00, 2026-07-22T14:01:15

    section Analysis
    GitHub Analysis     :active, 2026-07-22T14:02:00, 2026-07-22T14:05:00
    Web Intelligence    :done, 2026-07-22T14:03:00, 2026-07-22T14:03:45
```

Interval events (those with `interval_end`) are rendered as Gantt bars. Point events are rendered as milestones.

### Gantt View in UI

The web UI consumes the `timeline.query()` endpoint and renders an interactive Gantt chart (via a library like dhtmlx-gantt or frappe-gantt). The UI supports:
- Zoom: hours / minutes / seconds
- Filter by `source`, `event_type`, `actor`
- Click an event to see its full metadata and causal chain
- Export to Mermaid Markdown or PNG

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| SCE subscription lag | Time since last indexed event > `max_lag_ms` | Emit `timeline.sce_lag` metric; continue processing |
| Backfill missing events | Gap detected in sequence | Query SCE from known-good offset; backfill in batches |
| Index write failure | Database write error | Buffer to in-memory queue (max 10 000 events); retry with backoff |
| Causation chain too deep | `causation_id` walk exceeds `max_chain_depth` (default 100) | Return truncated chain with `truncated: true` flag |
| Query timeout | Query exceeds `query_timeout_ms` (default 5 000) | Return partial results with `timed_out: true` |
| Storage full | Index write fails on disk full | Switch to FIFO eviction; alert operator; emit `timeline.storage_full` |

Every failure emits a structured event on the SCE and is recorded in the [Audit Log](./AUDIT_LOG.md).

## Observability

| Metric | Labels | Description |
|--------|--------|-------------|
| `timeline_events_indexed_total` | `source` | Events written to timeline index |
| `timeline_query_total` | `query_type` | Queries executed |
| `timeline_query_seconds` | `query_type` | Query duration histogram |
| `timeline_causal_chain_depth` | — | Depth of traversed chains |
| `timeline_sce_lag_seconds` | — | SCE event ingestion lag |
| `timeline_index_write_error_total` | `source` | Index write failures |
| `timeline_audit_imported_total` | — | Audit Log entries imported per batch |
| `timeline_eviction_total` | — | Events evicted due to storage pressure |

Traces: one span per query, with child spans for index lookups and causal chain traversal.

## Acceptance Criteria

- Publishing an event to the SCE with `correlation_id` is visible in `timeline.before` / `timeline.after` within `index_latency_ms`.
- `timeline.causal_chain` correctly returns the linked chain for a sequence of events where each event sets `causation_id` to the previous event's ID.
- `timeline.between(t1, t2)` returns only events whose `timestamp` falls within `[t1, t2)`.
- A Gantt chart generated from interval events correctly maps `timestamp` → start and `interval_end` → end.
- Index write failures do not lose more than 10 000 events (in-memory buffer limit).

## Related Documents

- [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md)
- [Audit Log](./AUDIT_LOG.md)
- [Agent Lifecycle](./AGENT_LIFECYCLE.md)
- [Event Bus](./EVENT_BUS.md)
- [Observability](./OBSERVABILITY.md)
- [Data Retention](./DATA_RETENTION.md)
- [Tracing](./TRACING.md)
- [Logging](./LOGGING.md)
- [System Overview](./SYSTEM_OVERVIEW.md)
- [Main AI Kernel](./MAIN_AI_KERNEL.md)
