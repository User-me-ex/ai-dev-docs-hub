# API Spec

> Complete HTTP + WebSocket API specification for AI Dev OS — all endpoints, request/response schemas, error codes, versioning, authentication, and rate limits. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

The API is the external-facing interface of the [Backend](./BACKEND.md). It is the contract between the [Frontend](./FRONTEND.md), the [CLI](./CLI.md), external integrations, MCP clients, and the Backend server. All mutations to system state that originate outside the server process MUST go through this API.

The API is versioned under `/v1/`. Breaking changes require a new version prefix. Non-breaking additions (new optional fields, new endpoints) are added under the current version.

## Goals

- Single consistent API for web, desktop, CLI, and external integrations.
- REST + JSON for request/response; WebSocket + SSE for streaming events.
- OpenAPI 3.1 spec machine-generated from route definitions (canonical source is the implementation; this doc is the human-readable companion).
- All mutations are idempotent when identified by `run_id` / `item_id`.
- Error responses follow a consistent structure with machine-readable codes.

## Non-Goals

- gRPC — HTTP/JSON is the API; gRPC may be added in post-v1.0.
- GraphQL — not in scope for v1.0.
- Implementation code — this repository is documentation-only (see [AI Coding Rules](./AI_CODING_RULES.md)).

## Authentication

All endpoints except `/healthz`, `/readyz`, and `/metrics` require authentication.

```
Authorization: Bearer <token>
```

In local single-user mode (`auth.mode: "local"`), the token is a long-lived JWT generated on first install and stored in the OS keychain. The CLI reads it automatically.

In multi-user mode (`auth.mode: "oidc"` or `"jwt"`), tokens are issued by the configured OIDC provider or the system's JWT endpoint.

### Auth error responses

```json
{ "error": "UNAUTHORIZED", "message": "Missing or invalid Authorization header" }
{ "error": "FORBIDDEN",    "message": "Insufficient permissions for this resource" }
```

## Versioning

| Header | Value |
|--------|-------|
| `X-API-Version` (response) | `1.0.0` |
| `X-Deprecation-Date` (response) | Set when an endpoint is deprecated |

Breaking changes bump the `/v1/` prefix to `/v2/`. All v1 endpoints remain available for at least 12 months after a new version is released.

## Rate Limiting

In local mode: no rate limits apply.

In remote/multi-tenant mode:

| Limit | Default | Header |
|-------|---------|--------|
| Requests per minute | 600 | `X-RateLimit-Limit` |
| Runs per hour | 60 | — |
| Memory writes per minute | 300 | — |

Rate limit exceeded response: `HTTP 429` with `Retry-After: <seconds>`.

## Common Response Format

```json
// Success (singular resource)
{ "data": { ... } }

// Success (list)
{ "data": [ ... ], "meta": { "total": 42, "offset": 0, "limit": 20 } }

// Error
{
  "error": "SNAKE_CASE_CODE",
  "message": "Human-readable description",
  "detail": { /* optional structured context */ }
}
```

## Error Code Reference

| Code | HTTP | Meaning |
|------|------|---------|
| `UNAUTHORIZED` | 401 | Missing or invalid token |
| `FORBIDDEN` | 403 | Valid token, insufficient permissions |
| `NOT_FOUND` | 404 | Resource does not exist |
| `CONFLICT` | 409 | State conflict (e.g., run already cancelled) |
| `UNPROCESSABLE` | 422 | Request body fails schema validation |
| `RATE_LIMITED` | 429 | Too many requests |
| `INTERNAL` | 500 | Unexpected server error |
| `SERVICE_UNAVAILABLE` | 503 | Backend starting up or overloaded |
| `ROLE_NOT_ASSIGNED` | 422 | No model bound to the requested role |
| `MODEL_NOT_FOUND` | 404 | Model ID not in discovery cache |
| `QUEUE_FULL` | 503 | Background queue at hard capacity |
| `BUDGET_EXHAUSTED` | 402 | Run budget exceeded |
| `GUARDIAN_VETO` | 422 | Architecture Guardian blocked the change |

---

## Endpoints

### Health and Readiness

#### `GET /healthz`
Returns 200 OK while the server process is alive. No auth required.

```json
{ "ok": true, "version": "1.0.0", "uptime_s": 3600 }
```

#### `GET /readyz`
Returns 200 OK when all critical subsystems are initialised.

```json
{
  "ok": true,
  "services": {
    "sce":     { "ok": true,  "backend": "sqlite" },
    "memory":  { "ok": true,  "vector": "usearch" },
    "models":  { "ok": true,  "cached": 47 },
    "queue":   { "ok": true,  "depth": 0 },
    "guardian":{ "ok": true,  "rules": 14 }
  }
}
```

Returns 503 with `ok: false` when critical services are unavailable.

#### `GET /metrics`
Prometheus text-format metrics. Auth optionally required (`metrics.auth_required` config).

---

### Kernel Runs

#### `POST /v1/runs`
Submit a new goal to the Kernel.

**Request:**
```json
{
  "goal": "Refactor the authentication module to use JWT",
  "context_ref": "mem:01ABC...",  // optional: MemoryRecord ID for context
  "budget": {                      // optional: defaults from config
    "tokens_max": 100000,
    "wall_ms_max": 300000,
    "usd_max": 2.00
  },
  "group_id": "code-builder",      // optional: force a specific group
  "tags": ["auth", "refactor"]     // optional: metadata tags
}
```

**Response: 202 Accepted**
```json
{
  "data": {
    "run_id": "01ABCDEF...",
    "state": "intake",
    "correlation_id": "01XYZ...",
    "created_at": "2026-07-22T12:00:00Z"
  }
}
```

#### `GET /v1/runs`
List runs with optional filters.

**Query parameters:**
- `state` — filter by state (comma-separated)
- `limit` — max results (default 20, max 100)
- `offset` — pagination offset
- `since` — RFC 3339 timestamp (only runs created after)
- `tag` — filter by tag

**Response: 200 OK**
```json
{
  "data": [
    {
      "run_id": "01ABCDEF...",
      "goal": "Refactor auth...",
      "state": "executing",
      "tasks_total": 3,
      "tasks_completed": 1,
      "budget_spent": { "tokens": 5000, "usd": 0.08 },
      "created_at": "2026-07-22T12:00:00Z",
      "updated_at": "2026-07-22T12:01:00Z"
    }
  ],
  "meta": { "total": 42, "offset": 0, "limit": 20 }
}
```

#### `GET /v1/runs/:id`
Get full run status including task graph and verdicts.

**Response: 200 OK**
```json
{
  "data": {
    "run_id": "01ABCDEF...",
    "goal": "Refactor auth...",
    "state": "executing",
    "plan": { "tasks": [...], "dependencies": [...] },
    "budget": { "tokens_max": 100000, "wall_ms_max": 300000 },
    "budget_spent": { "tokens": 5000, "wall_ms": 12000, "usd": 0.08 },
    "verdicts": [],
    "artifacts": [],
    "correlation_id": "01XYZ...",
    "created_at": "...",
    "updated_at": "..."
  }
}
```

#### `GET /v1/runs/:id/events` (SSE)
Server-Sent Events stream of all SCE events for this run.

```
Content-Type: text/event-stream

event: worker.token
data: {"run_id":"01ABCDEF...","worker_id":"...","text":"Hello","ts":"..."}

event: worker.tool_call
data: {"name":"file_read","args":{"path":"src/auth.ts"},"duration_ms":12}

event: run.completed
data: {"state":"delivered","artifacts":[...],"budget_spent":{...}}
```

Client MUST reconnect with `Last-Event-Id` header to resume.

#### `DELETE /v1/runs/:id`
Cancel an active run.

**Request body (optional):**
```json
{ "reason": "User cancelled" }
```

**Response: 202 Accepted**
```json
{ "data": { "run_id": "...", "state": "cancelling" } }
```

---

### Model Catalog

#### `GET /v1/models`
List all discovered models.

**Query parameters:**
- `provider` — filter by provider ID
- `capability` — filter by capability (e.g., `tools`, `vision`)
- `deprecated=false` — exclude deprecated models (default true)
- `status` — `active | deprecated | all`
- `q` — text search against display_name and ID

**Response: 200 OK**
```json
{
  "data": {
    "providers": [
      {
        "id": "openai",
        "name": "OpenAI",
        "status": "healthy",
        "refreshed_at": "2026-07-22T12:00:00Z",
        "models": [
          {
            "id": "openai/gpt-4o",
            "display_name": "GPT-4o",
            "context_window": 128000,
            "capabilities": ["tools", "vision", "json_mode", "streaming"],
            "pricing": { "input_per_m": 2.50, "output_per_m": 10.00 },
            "deprecated": false
          }
        ]
      }
    ],
    "total": 47,
    "refreshed_at": "2026-07-22T12:00:00Z"
  }
}
```

#### `POST /v1/models/refresh`
Trigger immediate model discovery for all providers (or a specific provider).

**Request:**
```json
{ "provider": "openai" }  // optional: omit to refresh all
```

**Response: 202 Accepted**
```json
{ "data": { "job_id": "...", "providers": ["openai"] } }
```

#### `GET /v1/models/:id`
Get a single model by canonical ID.

---

### Nine Router — Role Assignments

#### `GET /v1/router/roles`
Get all current role assignments.

**Response: 200 OK**
```json
{
  "data": [
    {
      "role": "builder",
      "model_id": "openai/gpt-4o",
      "fallbacks": ["anthropic/claude-3-5-sonnet-20241022", "ollama/llama3.1:8b"],
      "scope": "workspace",
      "assigned_at": "2026-07-22T11:00:00Z"
    }
  ]
}
```

#### `PUT /v1/router/roles/:role`
Assign a model to a role.

**Request:**
```json
{
  "model_id": "anthropic/claude-3-5-sonnet-20241022",
  "fallbacks": ["openai/gpt-4o", "ollama/llama3.1:8b"],
  "scope": "workspace"
}
```

**Response: 200 OK** — updated RoleAssignment.

---

### Persistent Memory

#### `POST /v1/memory`
Write a memory record.

**Request:**
```json
{
  "kind": "kb_entry",
  "text": "The authentication module uses JWT with a 24h expiry.",
  "tags": ["auth", "jwt"],
  "retention": "90d",
  "key": "auth.jwt.expiry"
}
```

**Response: 201 Created** — created MemoryRecord.

#### `GET /v1/memory`
Query memory (hybrid BM25 + ANN).

**Query parameters:**
- `q` — text query (required)
- `k` — max results (default 10)
- `kind` — filter by kind
- `tags` — comma-separated tag filter
- `scope` — `individual | group | main | global | all`
- `min_score` — minimum relevance score (0–1)

**Response: 200 OK**
```json
{
  "data": [
    {
      "id": "01ABCDEF...",
      "kind": "kb_entry",
      "text": "The authentication module uses JWT...",
      "tags": ["auth", "jwt"],
      "score": 0.91,
      "source_type": "human",
      "created_at": "..."
    }
  ]
}
```

#### `GET /v1/memory/:id`
Get single memory record.

#### `DELETE /v1/memory/:id`
Soft-delete a memory record.

---

### Shared Context Engine

#### `GET /v1/context/topics`
List known topics.

#### `GET /v1/context/:topic` (SSE)
Snapshot + live tail of a SCE topic.

```
?from=snapshot   # start from current snapshot (default)
?from=latest     # only new events
?from=<ulid>     # resume from a specific event ID
```

#### `POST /v1/context/:topic`
Publish an event to a topic.

**Request:**
```json
{
  "kind": "event",
  "schema_name": "custom.alert",
  "payload": { "message": "Custom alert" }
}
```

---

### Research Engine

#### `POST /v1/research`
Enqueue a research job.

**Request:**
```json
{
  "source": { "type": "http", "url": "https://docs.example.com" },
  "cadence": "0 * * * *",
  "kb_target": "main",
  "tags": ["docs", "example"]
}
```

#### `GET /v1/research`
List research jobs.

#### `GET /v1/research/:id`
Get job status including artifact list.

---

### Plugin Manager

#### `GET /v1/plugins`
List installed plugins.

#### `POST /v1/plugins`
Install a plugin.

**Request:**
```json
{ "ref": "~/.aidevos/plugins/my-plugin" }   // or registry URL
```

#### `PUT /v1/plugins/:id/enable`
Enable a plugin.

#### `PUT /v1/plugins/:id/disable`
Disable a plugin.

#### `DELETE /v1/plugins/:id`
Uninstall a plugin.

---

### Guardian

#### `GET /v1/guardian/rules`
List active Guardian rules.

#### `GET /v1/guardian/verdicts`
List recent verdicts.

**Query:** `run_id`, `ok=false`, `limit`, `since`.

---

## WebSocket API

Connect at `ws://<host>/v1/ws` with the same `Authorization` header.

### Subscribe to a run

```json
→ { "action": "subscribe", "topic": "run.01ABCDEF..." }
← { "event": "subscribed", "topic": "run.01ABCDEF..." }
← { "event": "worker.token", "data": { ... } }
```

### Publish an event

```json
→ { "action": "publish", "topic": "run.01ABCDEF...", "payload": { ... } }
← { "event": "published", "id": "..." }
```

### Unsubscribe

```json
→ { "action": "unsubscribe", "topic": "run.01ABCDEF..." }
← { "event": "unsubscribed" }
```

---

## Requirements

- **MUST** implement all endpoints listed; omitting any is a spec violation.
- **MUST** return consistent error format for all 4xx and 5xx responses.
- **MUST** support `idempotency-key` header for mutating endpoints; duplicate requests with the same key return the original response.
- **MUST** validate all request bodies against their JSON Schema; invalid bodies return `UNPROCESSABLE` with `detail.fields` listing failing fields.
- **MUST** include `X-Request-Id` (ULID) in every response; use this for support trace-back.
- **MUST** return `X-API-Version` in every response.
- **MUST** support SSE reconnection via `Last-Event-Id` on all SSE endpoints.
- **SHOULD** support cursor-based pagination (`after=<last_id>`) in addition to offset-based.
- **MAY** support `Accept: application/x-ndjson` on list endpoints for streaming large result sets.

## Acceptance Criteria

- `POST /v1/runs` with a valid goal returns 202 with a `run_id` within 200 ms.
- `GET /v1/runs/:id/events` delivers every `worker.token` event within 50 ms of publication on the SCE.
- An invalid request body to any endpoint returns 422 with `detail.fields` listing all failing fields.
- `PUT /v1/router/roles/builder` with a model not in the discovery cache returns 404 `MODEL_NOT_FOUND`.
- Two requests with the same `Idempotency-Key` header return identical responses; only one mutation is applied.

## Open Questions

- Whether to add a GraphQL endpoint for flexible queries on memory and run data — tracked in [templates/ADR](../templates/ADR.md).

## Related Documents

- [Backend](./BACKEND.md)
- [Frontend](./FRONTEND.md)
- [CLI](./CLI.md)
- [IPC](./IPC.md)
- [MCP](./MCP.md)
- [Auth System](./AUTH_SYSTEM.md)
- [Nine Router](./NINE_ROUTER.md)
- [Persistent Memory](./PERSISTENT_MEMORY.md)
- [System Overview](./SYSTEM_OVERVIEW.md)
