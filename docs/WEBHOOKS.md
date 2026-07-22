# Webhooks — Outgoing Event Notifications

> The managed outbound webhook dispatch system that delivers real-time event notifications to external services with HMAC-signed payloads, exponential-backoff retries, and dead-letter observability. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

Webhooks let external systems subscribe to events emitted by AI Dev OS. When an event matching a registered webhook's filter is flushed to the [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md), the Webhooks subsystem delivers a signed HTTP POST to the configured endpoint. Retries follow an exponential-backoff schedule; permanently failed deliveries move to a dead-letter queue for manual inspection.

Webhooks are **outgoing only**. Incoming webhooks (external services calling into AI Dev OS) are handled by the public HTTP API defined in [API Spec](./API_SPEC.md).

## Goals

- Reliable at-least-once delivery with configurable retry.
- HMAC-SHA256 signing of every payload so receivers can verify authenticity.
- Per-webhook event type filtering — a webhook only receives events it registered for.
- Dead-letter queue for deliveries that exhaust all retries.
- Full audit trail of every delivery attempt.

## Non-Goals

- Incoming webhook ingestion — use the public API.
- Webhook management UI — CLI and API only (see [CLI](./CLI.md)).
- Custom payload transformation — payloads are standardised envelopes.

## Webhook Registration Schema

```
WebhookRegistration {
  id:            string          # auto-generated unique ID
  url:           string          # HTTPS endpoint (HTTP rejected at registration)
  events:        string[]        # event type patterns, e.g. ["run.*", "guardian.veto"]
  secret:        string          # HMAC signing secret (stored hashed; returned only on create)
  retry_policy: {
    max_attempts:  u32           # default 5, max 10
    base_delay_ms: u32           # default 1000
    max_delay_ms:  u32           # default 60000
    multiplier:    f64           # default 2.0 (exponential)
  }
  headers:       Record<string, string>  # custom HTTP headers, e.g. { "X-Environment": "prod" }
  active:        boolean         # true to receive deliveries; false to pause
  created_at:    rfc3339
  updated_at:    rfc3339
}
```

`secret` is required. It is used to compute the HMAC signature. The subsystem stores only a SHA-256 hash of the secret; the plaintext secret is returned exactly once in the registration response and never again.

## Supported Event Types

| Event Type | Description | Payload Highlights |
|------------|-------------|--------------------|
| `run.completed` | A run finished (success or failure) | `{ run_id, status, duration_ms, agent_id }` |
| `guardian.veto` | The Architecture Guardian vetoed a plan | `{ plan_id, reason, rule_ids[] }` |
| `group.circuit_open` | A group's circuit breaker tripped | `{ group_id, failure_rate, window_seconds }` |
| `research.completed` | A research operation finished | `{ research_id, source_count, token_usage }` |
| `memory.updated` | Persistent memory was written | `{ memory_id, agent_id, operation }` |
| `error.critical` | An unrecoverable error occurred | `{ error_code, subsystem, message }` |

Custom event types added by plugins flow through automatically; the webhook dispatcher matches them against the subscriber's `events` patterns.

## Delivery Protocol

Every delivery is an HTTPS POST:

```
POST <webhook.url> HTTP/1.1
Content-Type: application/json
X-Aidevos-Signature-256: <HMAC-SHA256 hex digest>
X-Aidevos-Event-Type: <event.type>
X-Aidevos-Delivery-Id: <uuid v7>
User-Agent: aidevos-webhook/1.0
```

Body (JSON):

```
{
  "specversion": "1.0",
  "id": "<event.id>",
  "source": "<event.source>",
  "type": "<event.type>",
  "datacontenttype": "application/json",
  "time": "<event.timestamp>",
  "correlation_id": "<event.correlation_id>",
  "data": <event.payload>           # original event payload
}
```

The `X-Aidevos-Signature-256` header is the hex-encoded HMAC-SHA256 of the raw request body, computed with the webhook's `secret`. Recipients MUST recompute and compare this signature before trusting the payload.

The `X-Aidevos-Delivery-Id` header is unique per delivery attempt (not per event) — a retry of the same event gets a new delivery ID.

## Retry Policy

When a delivery fails (non-2xx status, network error, or timeout > 10 s), the dispatcher retries according to the webhook's `retry_policy`:

1. **Schedule**: `delay = min(base_delay_ms * multiplier^(attempt-1), max_delay_ms)` with jitter (±20%).
2. **Attempts**: up to `max_attempts` (default 5). Attempt 1 is the initial delivery; max 4 retries by default.
3. **Dead letter**: after `max_attempts` failures, the event moves to the dead-letter queue. A dead-letter event is stored with all delivery attempt logs:
   ```
   DeadLetterEvent {
     webhook_id:    string
     event_id:      string
     attempts:      DeliveryAttempt[]
     first_failed:  rfc3339
     last_failed:   rfc3339
     reason:        string   # "max_retries_exceeded"
   }
   ```
4. **Manual replay**: dead-letter events can be replayed via `webhooks.replay(webhook_id, event_id)` or bulk-replayed.
5. **Backpressure**: if the delivery queue exceeds 10,000 pending events, new events for that webhook are dropped with a `webhook.backpressure` warning event.

## Security

| Measure | Detail |
|---------|--------|
| HMAC signing | HMAC-SHA256 of the JSON body, hex-encoded in `X-Aidevos-Signature-256` |
| Secret storage | Only SHA-256 hash stored; plaintext returned once on creation |
| URL validation | Only HTTPS URLs accepted (HTTP is rejected at registration) |
| IP allowlist | Optional; if configured, the delivering IP range is published so receivers can allowlist |
| Payload limits | Max 1 MB per payload; larger payloads are truncated with `"truncated": true` in data |
| Sensitive data | Webhook payloads MUST NOT contain secrets; the [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) filters secret-like patterns before dispatch |

## Interfaces

```
webhooks.register(params: {
  url: string
  events: string[]
  retry_policy?: RetryPolicy
  headers?: Record<string, string>
}) → WebhookRegistration  # includes secret (returned once)
webhooks.unregister(webhook_id: string) → Ack
webhooks.list(filter?: { event_type?: string, active?: boolean }) → WebhookRegistration[]
webhooks.get(webhook_id: string) → WebhookRegistration  # secret NOT included
webhooks.update(webhook_id: string, changes: Partial<WebhookRegistration>) → WebhookRegistration
webhooks.rotate_secret(webhook_id: string) → { new_secret }
webhooks.logs(webhook_id: string, opts?: { limit, since }) → DeliveryAttempt[]
webhooks.replay(webhook_id: string, event_id: string) → DeliveryAttempt
webhooks.dead_letter.list() → DeadLetterEvent[]
webhooks.dead_letter.replay(event_id: string) → Ack
webhooks.dead_letter.delete(event_id: string) → Ack
```

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| Network timeout ( > 10 s) | Response not received | Retry with backoff |
| Non-2xx status | Status code not in 2xx range | Retry; 4xx permanent errors (401, 403, 410) skip retry and dead-letter immediately |
| DNS resolution failure | DNS NXDOMAIN | Retry; after max attempts, dead-letter |
| TLS error | Certificate expired, host mismatch | Retry once (ephemeral errors); dead-letter if persistent |
| Rate-limited (429) | 429 response with Retry-After | Wait `Retry-After` seconds before next attempt |
| Secret mismatch | Recipient reports HMAC mismatch | Not detectable by sender; logged as successful delivery |

## Observability

| Metric | Type | Description |
|--------|------|-------------|
| `webhook_deliveries_total{webhook_id,status}` | Counter | Delivery attempts by outcome (2xx, 4xx, 5xx, error) |
| `webhook_delivery_seconds{webhook_id}` | Histogram | End-to-end delivery latency |
| `webhook_retries_total{webhook_id}` | Counter | Retry attempts |
| `webhook_dead_letter_total{webhook_id}` | Counter | Events moved to dead-letter queue |
| `webhook_queue_depth` | Gauge | Pending deliveries in the dispatch queue |

All delivery attempts, retries, and dead-letter events are recorded in the [Audit Log](./AUDIT_LOG.md). Each delivery generates a trace span with the webhook ID and delivery ID as attributes. See [Tracing](./TRACING.md).

## Acceptance Criteria

- Registering a webhook with `events: ["run.completed"]` and publishing a `run.completed` event results in an HTTPS POST to the registered URL within 500 ms.
- A webhook delivering to a 500-ing endpoint retries with exponential backoff and dead-letters after `max_attempts` failures.
- Rotating the secret immediately invalidates the old signature; the next delivery uses the new secret.
- Dead-letter events can be replayed via `webhooks.replay()` and the replay delivery appears in the logs.

## Related Documents

- [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md)
- [Event Bus](./EVENT_BUS.md)
- [API Spec](./API_SPEC.md)
- [Security Model](./SECURITY_MODEL.md)
- [Observability](./OBSERVABILITY.md)
- [Audit Log](./AUDIT_LOG.md)
- [Main AI Kernel](./MAIN_AI_KERNEL.md)
