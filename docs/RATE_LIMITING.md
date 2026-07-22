# Rate Limiting

> Rate limiting strategy across AI Dev OS, covering model providers, SCE topics, API endpoints, and tool calls.

## Overview

Rate limiting protects AI Dev OS from resource exhaustion, ensures fair sharing across concurrent runs, and enforces contractual limits with external [Model Providers](./MODEL_PROVIDERS.md). The system applies token-bucket rate limits at multiple tiers and communicates backpressure through HTTP headers, SCE events, and queue signals.

## Where Rate Limits Apply

| Layer | What Is Limited | Limiting Factor |
|---|---|---|
| Model providers | API calls per model per minute | Provider contract + cost budget |
| SCE topics | Publish events per topic per second | SCE throughput capacity |
| API endpoints | REST/gRPC requests per route | Server capacity |
| Tool calls | Tool invocations per run per second | Worker execution capacity |
| Worker spawns | Concurrent workers per workspace | Resource pool size |
| File operations | Disk writes per second | I/O bandwidth |

## Rate Limit Tiers

Each tier is independently configurable. The effective rate is the minimum of all applicable tiers.

| Tier | Scope | Default | Configured In |
|---|---|---|---|
| Per-provider | Per model provider API key | Provider-defined | `config.toml` → `[rate_limits.providers]` |
| Per-workspace | All runs in a workspace | 100 req/s burst | `config.toml` → `[rate_limits.workspace]` |
| Per-user | All runs by a user identity | 50 req/s sustained | `config.toml` → `[rate_limits.user]` |
| Per-run | Single run execution | 20 req/s sustained | `config.toml` → `[rate_limits.run]` |
| Global | System-wide aggregate | 500 req/s burst | `config.toml` → `[rate_limits.global]` |

## Algorithm

Every rate limit uses a **token bucket** with configurable capacity (burst) and refill rate (sustained). Tokens are consumed per request. If the bucket is empty, the request is either queued or rejected based on the policy.

```
capacity = burst_limit
refill   = sustained_limit / second
cost     = 1 per request (or weighted by request cost)
```

Weighted costs allow expensive operations (embedding, file write) to consume more tokens than cheap ones (config read).

## Backpressure Signals

| Signal | Source | Consumer Action |
|---|---|---|
| `429 Too Many Requests` | Model provider HTTP response | Queue or fail with retry-After |
| `X-RateLimit-*` headers | AI Dev OS API | Client-side backoff |
| SCE `rate_limit.backpressure` event | Kernel rate limiter | Workers pause or throttle |
| Worker queue depth notification | Job scheduler | Kernel delays new work dispatch |
| Provider `retry-after` header | Model provider | Respect before retry |

## Configuration

Rate limits are defined in `config.toml`:

```toml
[rate_limits]
mode = "queue"  # "queue" | "reject" | "degrade"

[rate_limits.global]
burst = 500
sustained = 200

[rate_limits.providers.openai]
burst = 60
sustained = 30

[rate_limits.providers.anthropic]
burst = 40
sustained = 20

[rate_limits.topics]
"system.events" = { burst = 200, sustained = 100 }
"agent.messages" = { burst = 100, sustained = 50 }
```

## Rate Limit Headers

All HTTP API responses include standard rate limit headers:

| Header | Description |
|---|---|
| `X-RateLimit-Limit` | Maximum requests per window |
| `X-RateLimit-Remaining` | Requests remaining in current window |
| `X-RateLimit-Reset` | Unix timestamp when the window resets |
| `Retry-After` | Seconds to wait before retry (on 429 only) |

## Queueing When Rate Limited

When `mode = "queue"`, rate-limited requests are placed in a per-tier FIFO queue (see [Queueing](./QUEUEING.md)). Queue capacity defaults to 1000 items per tier. When the queue is full, new requests receive a 429 response.

## Failure Modes

| Failure | Cause | Effect | Mitigation |
|---|---|---|---|
| Rate limit exceeded | Transient burst exceeds capacity | Request queued or 429 | Client retries with backoff |
| Backpressure cascade | Upstream provider throttles | All dependent operations slow | Circuit breaker on provider |
| Token bucket starvation | Sustained max throughput | Latency increases linearly | Scale out workers |
| Misconfigured limit | Limit set too low | False positives | Validation on config load |
| Queue overflow | Sustained overload | Requests dropped with 429 | Alert on queue depth |

## Related Documents

- [Queueing](./QUEUEING.md) — queue architecture and backpressure propagation
- [Model Providers](./MODEL_PROVIDERS.md) — provider-specific rate limit contracts
- [Cost Management](./COST_MANAGEMENT.md) — cost-aware rate limiting
- [Scalability](./SCALABILITY.md) — horizontal scaling to handle increased load
- [Observability](./OBSERVABILITY.md) — rate limit metrics and dashboards
