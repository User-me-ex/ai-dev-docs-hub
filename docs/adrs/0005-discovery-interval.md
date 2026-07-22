# ADR-0005: Model Discovery Auto-Refresh Interval

## Status

Accepted

## Context

The [Nine Router](./NINE_ROUTER.md) polls every configured provider's `/models` endpoint to discover available models. The poll interval determines how quickly new models appear in the UI and how quickly deprecated models are surfaced. Too frequent: wasted API calls and rate-limit risk. Too infrequent: stale model catalog.

## Decision

**Fixed 10-minute interval, with targeted refresh on credential change.**

1. The default discovery interval is 10 minutes (configurable via `discovery_interval_sec`).
2. When a provider's credential is added, updated, or removed, a **targeted refresh** runs for that provider only within 5 seconds.
3. The interval is fixed (not adaptive) for v1.0. Adaptive intervals based on error rate or API response time are tracked as a future improvement.
4. Discovery results are cached per-provider with the same TTL as the interval.
5. The `last_success` timestamp per provider is exposed in the UI and API.

### Why fixed 10 minutes?

- Simple to implement and reason about
- 10 minutes is fast enough that a user waiting for a new model is not frustrated (>6 refreshes per hour)
- 10 minutes is slow enough to not be a burden on provider APIs (most model lists change at most daily)
- Targeted refresh on credential change ensures that adding a new provider key shows models within seconds, not minutes
- Adaptive intervals add complexity without clear benefit for v1.0 — provider APIs don't change model lists frequently enough to warrant it

### Risk: Rate limiting

Some providers (notably Anthropic and Mistral) throttle `/models` requests at higher rates. With a 10-minute interval per provider, even with 10 configured providers, we make at most 1,440 requests/day — well within typical rate limits.

## Consequences

**Positive:**
- Simple implementation (cron job with fixed interval)
- Predictable API call volume
- Credential changes reflect immediately
- Configurable for power users

**Negative:**
- If a model is launched and deprecated within the same 10-minute window, the short window could be missed (unlikely in practice — models are deprecated over weeks/months)
- All providers are polled at the same interval, even if some change rarely (e.g. local Ollama models)

## Related

- [Nine Router](./NINE_ROUTER.md) — discovery trigger details
- [Job Scheduler](./JOB_SCHEDULER.md) — the cron engine that drives the interval
