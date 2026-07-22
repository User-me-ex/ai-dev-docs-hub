# Streaming Responses

> How AI Dev OS streams model tokens, tool results, SCE events, and progress updates to clients over SSE, WebSocket, and MCP transports. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

Streaming is the primary delivery mechanism for AI Dev OS outputs. Every model response, tool call, state transition, and log line is streamed to subscribed clients in real time. The streaming layer sits between the [Dynamic Worker](./DYNAMIC_WORKERS.md) execution loop and the client-facing transport layer ([API Spec](./API_SPEC.md)), providing a unified event pipeline regardless of whether the client is the CLI, the Web UI, or an MCP peer.

Three types of content are streamed:

1. **Model token streams**: individual tokens from the LLM response, delivered as they are generated.
2. **Tool call streams**: structured notifications when a tool is invoked, including its arguments, progress, and result.
3. **SCE event streams**: all Shared Context Engine events published during a run, replayed as a continuous stream.

The streaming system is built on three transports: Server-Sent Events (SSE) for HTTP clients, WebSocket for bidirectional clients, and MCP streaming for MCP peers. All three transports deliver the same event format, differing only in framing.

## Goals

- Unified event schema across all transports — a subscriber receives the same events regardless of transport.
- Ordered delivery: events from a single worker arrive in the order they were produced.
- Reconnection resilience: clients can resume interrupted streams from the last received event.
- Backpressure: the streaming layer applies backpressure to prevent memory exhaustion when a client is slow to consume.
- Graceful degradation: a transport failure mid-stream delivers all buffered events and a terminal error event.

## Non-Goals

- Reliable delivery semantics (exactly-once) — stream delivery is at-least-once; duplicate events are handled by the client via `event_id` deduplication.
- Long-term event storage — streams are ephemeral; durable event history is in the [Audit Log](./AUDIT_LOG.md).
- Implementation code — this repository is documentation-only (see [AI Coding Rules](./AI_CODING_RULES.md)).

## Streaming Models

### SSE (Server-Sent Events)

Used for HTTP clients (CLI, browser). The client opens a single GET request to an SSE endpoint (e.g., `GET /v1/runs/:id/events`) and receives events as a text/event-stream.

```
GET /v1/runs/:id/events HTTP/1.1
Accept: text/event-stream

Response:
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no

event: worker.token
id: evt_01ABC1
data: {"run_id":"run_01XYZ","worker_id":"wkr_001","text":"Hello","index":0,"ts":"2026-07-22T12:00:00Z"}

event: worker.token
id: evt_01ABC2
data: {"run_id":"run_01XYZ","worker_id":"wkr_001","text":" world","index":1,"ts":"2026-07-22T12:00:00.100Z"}
```

SSE supports reconnection via the `Last-Event-Id` header. When the client reconnects, the server replays events from that ID onward.

### WebSocket Streaming

Used for bidirectional clients (Web UI, integrated tools). The client connects to `ws://<host>/v1/ws` and subscribes to topics. Events are pushed as JSON-framed messages.

```
← { "event": "run.completed", "id": "evt_01XYZ", "data": { ... } }
```

WebSocket supports both subscription-based and pattern-based topic matching (`run.*`).

### MCP Streaming

Used for MCP client peers. Events are delivered as MCP `notifications/events` messages over the MCP transport (stdio or HTTP/SSE). See [MCP](./MCP.md) for the full protocol.

### Summary Comparison

| Feature | SSE | WebSocket | MCP Streaming |
|---------|-----|-----------|---------------|
| Direction | Server → Client | Bidirectional | Bidirectional |
| Reconnection | Built-in (`Last-Event-Id`) | Manual (client tracks `event_id`) | MCP session layer |
| Framing | `text/event-stream` | JSON messages | MCP JSON-RPC notifications |
| Best for | CLI, browser GET | Web UI, real-time dashboards | MCP client peers |
| Backpressure | HTTP/2 flow control | TCP backpressure | MCP transport dependent |

## Stream Chunk Schema

Every streamed event follows the same schema, regardless of transport:

```
StreamEvent {
  id:        string          # monotonic event ID (ULID)
  event:     EventType       # dot-separated event name
  data:      object          # event payload (varies by event type)
  retry?:    number          # milliseconds to wait before reconnecting (SSE only)
  ts:        rfc3339         # event timestamp
  run_id:    string
  worker_id?: string         # absent for Kernel-level events
  correlation_id: string
}
```

The `id` field is monotonic within a run — clients can use it for ordering and deduplication. The `event` field follows the same namespacing as SCE events: `worker.token`, `tool.call`, `run.completed`, etc.

## Stream Types

### Model Token Stream

Emitted as the model generates output tokens:

```
{
  "id": "evt_01ABC1",
  "event": "worker.token",
  "data": {
    "text": "Hello",
    "index": 0,
    "finish_reason": null
  },
  "run_id": "run_01XYZ",
  "worker_id": "wkr_001",
  "ts": "2026-07-22T12:00:00Z"
}
```

The final token in a stream has a non-null `finish_reason` (`"stop"`, `"length"`, `"tool_calls"`, or `"content_filtered"`). Clients SHOULD accumulate `text` fields in `index` order to reconstruct the full response.

### Tool Call Stream

Emitted for each tool invocation during execution:

```
// Tool call started
{
  "id": "evt_01ABC5",
  "event": "tool.call",
  "data": {
    "call_id": "call_001",
    "name": "file_write",
    "args": {"path": "src/hello.py", "content": "..."},
    "status": "started",
    "ts": "2026-07-22T12:00:01.000Z"
  },
  "run_id": "run_01XYZ",
  "worker_id": "wkr_001"
}

// Tool call completed
{
  "id": "evt_01ABC6",
  "event": "tool.result",
  "data": {
    "call_id": "call_001",
    "name": "file_write",
    "status": "success",
    "duration_ms": 45,
    "result": {"size_bytes": 1024}
  }
}

// Tool call failed
{
  "id": "evt_01ABC7",
  "event": "tool.error",
  "data": {
    "call_id": "call_002",
    "name": "file_read",
    "status": "error",
    "duration_ms": 12,
    "error": {"code": "FILE_NOT_FOUND", "message": "src/missing.py does not exist"}
  }
}
```

### Log Stream

Structured log lines emitted during execution:

```
{
  "id": "evt_01ABC3",
  "event": "log.line",
  "data": {
    "level": "info",
    "message": "Context compressed: 45000 -> 32000 tokens",
    "module": "context_compressor",
    "ts": "2026-07-22T12:00:00.500Z"
  }
}
```

Log levels: `debug`, `info`, `warn`, `error`. Enabled by default; clients can filter by level via query parameter (`?log_level=warn`).

### Progress Stream

High-level progress updates for the run:

```
{
  "id": "evt_01ABC4",
  "event": "run.progress",
  "data": {
    "tasks_total": 5,
    "tasks_completed": 2,
    "pct_complete": 40,
    "budget_spent": {"tokens": 45000, "usd": 0.90, "pct_budget": 45}
  }
}
```

Emitted on every task completion and on every `worker.tick` heartbeat (default every 10 seconds).

## Backpressure Handling

The streaming layer applies backpressure at three levels:

1. **Client buffer**: each client connection has a configurable buffer (`stream.buffer_max_events`, default 10,000). When the buffer exceeds this limit, the streaming layer pauses event production for that client and emits `stream.backpressure_warning` internally.
2. **Worker throttling**: when a worker's event publication rate exceeds the SCE's capacity (default 500 events/s per topic), the worker's model stream is throttled via a token bucket (`stream.token_bucket_rate`, default 200 events/s per worker). Throttled workers continue generating tokens internally but events are queued.
3. **Provider backpressure**: when the model provider's streaming response slows (network congestion, server-side throttling), the streaming layer does not buffer aggressively — it passes the reduced throughput directly to the client, maintaining backpressure transparency.

### Slow Consumer Protection

If a client fails to read events (TCP buffer fills), the streaming layer:
1. Buffers up to `stream.buffer_max_bytes` (default 50 MB) per connection.
2. When buffer is exceeded, drops the oldest events (starting with `log.line`, then `worker.token`).
3. If all event types are dropped and the buffer is still over limit, the connection is closed with `stream.consumer_too_slow` reason.
4. The client can reconnect with `Last-Event-Id` to recover dropped events.

## Client-Side Streaming

### CLI

The CLI connects to `GET /v1/runs/:id/events` via SSE. It renders:
- Token events as incremental text (typewriter effect).
- Tool call events as formatted JSON blocks.
- Progress events as a live progress bar.
- Log events as coloured output (warn= yellow, error= red).

Connection: the CLI uses `curl`-compatible SSE or the Node.js `eventsource` library. Reconnection is automatic via `Last-Event-Id`.

### Web UI

The Web UI connects via WebSocket to `ws://<host>/v1/ws`. It subscribes to `run.<id>` and renders events in a real-time log panel:
- Token events in a streaming text area.
- Tool calls as collapsible cards with start/end timestamps.
- A progress bar at the top of the run view.

### MCP Client

MCP clients receive events as MCP notifications over their transport. The client subscribes via `resources/subscribe` and receives `notifications/events` for each stream event. See [MCP](./MCP.md) for subscription mechanics.

## Error Handling Mid-Stream

### Graceful Degradation

When an error occurs mid-stream:
1. The streaming layer delivers all buffered events up to the point of error.
2. A terminal error event is emitted:
```
{
  "id": "evt_01FFF",
  "event": "stream.error",
  "data": {
    "code": "WORKER_CRASHED",
    "message": "Worker wkr_001 failed: all model fallbacks exhausted",
    "partial_output": "The refactored code should...",
    "last_event_id": "evt_01FFE",
    "checkpoint_id": "chk_001"
  }
}
```
3. The stream is closed cleanly on the server side.

### Partial Results

If the worker crashes mid-token, the streaming layer delivers all tokens produced before the crash as the final `worker.token` event with `finish_reason: "crashed"`. The partial output is included in the `stream.error` event.

### Transport Failure

| Failure | Behaviour |
|---------|-----------|
| Client disconnects | Stream continues server-side; events buffered up to `buffer_max_events`. When buffer is full, oldest events are dropped. |
| Server restarts | All streams are terminated. Clients reconnect with `Last-Event-Id`; server resumes from the last checkpoint. |
| Transport timeout | Client detects `timeout` (30s no data), reconnects with `Last-Event-Id`. Server replays from that ID. |
| WebSocket ping failure | Client sends WebSocket ping at 15s intervals. No pong within 10s → reconnect. |

## Acceptance Criteria

- A client subscribing via SSE to `GET /v1/runs/:id/events` MUST receive every `worker.token` event within 100 ms of its emission on the SCE.
- A WebSocket client that disconnects and reconnects with the last received `event_id` MUST receive all missed events, in order, starting from that ID.
- A worker that generates events faster than the SCE rate limit (500 events/s) MUST be throttled: events MUST be queued, not dropped, up to `stream.buffer_max_events`.
- A `stream.error` event MUST be the last event in a stream; no events after it are delivered.
- The same event delivered over SSE, WebSocket, and MCP transports MUST have identical `id`, `event`, `data`, `correlation_id`, and `ts` fields.

## Open Questions

- Whether to support a compressed binary stream format (CBOR or msgpack) for high-throughput MCP streaming — tracked in [templates/ADR](../templates/ADR.md).
- Whether to add a `stream.batch` event that bundles multiple small events (log lines, token fragments) to reduce transport overhead.

## Related Documents

- [API Spec](./API_SPEC.md) — endpoint definitions for SSE and WebSocket streaming
- [Frontend](./FRONTEND.md) — Web UI streaming client implementation
- [MCP](./MCP.md) — MCP notification streaming for peer clients
- [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md) — event publication that feeds the streaming layer
- [Dynamic Workers](./DYNAMIC_WORKERS.md) — worker lifecycle and event emission
- [System Overview](./SYSTEM_OVERVIEW.md)
- [Main AI Kernel](./MAIN_AI_KERNEL.md)
