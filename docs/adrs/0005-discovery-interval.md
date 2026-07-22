# ADR-0005: Model Discovery Auto-Refresh Interval

## Status

Accepted

## Context

The [Nine Router](../NINE_ROUTER.md) is the sole model gateway: it polls every configured provider's `/models` endpoint on behalf of all subsystems. No subsystem calls any provider directly. The poll interval determines how quickly new models appear in the UI and how quickly deprecated models are surfaced. Too frequent: wasted API calls and rate-limit risk. Too infrequent: stale model catalog.

## Decision

**Fixed 10-minute interval, with targeted refresh on credential change.**

1. The default discovery interval is 10 minutes (configurable via `discovery_interval_sec`).
2. When a provider's credential is added, updated, or removed, a **targeted refresh** runs for that provider only within 5 seconds.
3. The interval is fixed (not adaptive) for v1.0. Adaptive intervals based on error rate or API response time are tracked as a future improvement.
4. Discovery results are cached per-provider with the same TTL as the interval.
5. The `last_success` timestamp per provider is exposed in the UI and API.

### Adaptive Interval Calculation (v2.0 Feature)

For v2.0, the fixed interval will be replaced with an adaptive algorithm:

```
Algorithm: ComputeAdaptiveInterval
Input:
  - provider_id: string
  - error_rate: float              // Ratio of failed polls in sliding window
  - api_latency_p95_ms: float      // P95 response time
  - model_list_stability: float    // Rate of model list changes (0 = stable, 1 = churning)
  - rate_limit_remaining: int      // Remaining requests in rate limit window
  - rate_limit_window_sec: int     // Rate limit window duration
Output: interval_sec: int

01  // Base: start with configured default (600s = 10 min)
02  base_interval = 600
03
04  // Factor 1: Error rate (backoff on failures)
05  if error_rate > 0.5:
06      error_factor = 5.0          // Heavy backoff: 50 min
07  elif error_rate > 0.2:
08      error_factor = 3.0          // Moderate backoff: 30 min
09  elif error_rate > 0.05:
10      error_factor = 1.5          // Light backoff: 15 min
11  else:
12      error_factor = 1.0
13
14  // Factor 2: API latency (slower API → poll less often)
15  if api_latency_p95_ms > 5000:
16      latency_factor = 3.0
17  elif api_latency_p95_ms > 2000:
18      latency_factor = 1.5
19  else:
20      latency_factor = 1.0
21
22  // Factor 3: Model list stability (stable list → poll less often)
23  if model_list_stability < 0.01:
24      stability_factor = 2.0      // Very stable: poll every 20 min
25  elif model_list_stability < 0.1:
26      stability_factor = 1.0      // Normal: poll every 10 min
27  else:
28      stability_factor = 0.5      // Churning: poll every 5 min
29
30  // Factor 4: Rate limit headroom
31  if rate_limit_window_sec > 0 and rate_limit_remaining > 0:
32      rate_factor = max(0.5, (base_interval * rate_limit_remaining) / rate_limit_window_sec)
33  else:
34      rate_factor = 1.0
35
36  // Compute final interval
37  interval = base_interval * error_factor * latency_factor * stability_factor * rate_factor
38
39  // Clamp to bounds
40  interval = min(max(interval, 60), 3600)  // Between 1 min and 60 min
41
42  return round(interval, 10)  // Round to nearest 10 seconds
```

### Targeted Refresh Trigger Catalog

Targeted refresh is triggered by the following events (not just credential changes):

| Trigger | Event Source | Response Time | Refresh Scope |
|---------|-------------|---------------|---------------|
| Credential added | SCE: `provider.credential_added` | Within 5s | Single provider |
| Credential updated | SCE: `provider.credential_updated` | Within 5s | Single provider |
| Credential removed | SCE: `provider.credential_removed` | Within 5s | Single provider |
| Credential expired | SCE: `provider.credential_expired` | Within 5s | Single provider |
| Provider config changed | SCE: `provider.config_updated` | Within 5s | Single provider |
| Manual refresh requested | API: `POST /discovery/refresh` | Immediate | As specified |
| Rate limit reset | SCE: `provider.rate_limit_reset` | Next scheduled poll | Single provider |
| Network reconnected | SCE: `system.network_reconnected` | Within 30s | All providers |
| Startup/first poll | System boot | Immediate | All providers |

### SCE Event Schema for Discovery Events

```typescript
// discovery.poll_started
{
  "event_type": "discovery.poll_started",
  "payload": {
    "provider_id": "openai",
    "provider_type": "openai",
    "poll_type": "scheduled" | "targeted" | "manual",
    "trigger_event_id": string | null,  // If targeted, the triggering event ID
    "scheduled_interval_sec": 600,
    "last_successful_poll_at": 1735689000000,
    "attempt": 1
  }
}

// discovery.poll_completed
{
  "event_type": "discovery.poll_completed",
  "payload": {
    "provider_id": "openai",
    "provider_type": "openai",
    "poll_type": "scheduled" | "targeted" | "manual",
    "result": "success" | "partial" | "failed",
    "models_discovered": 45,
    "models_added": 2,         // New models since last poll
    "models_removed": 1,       // Deprecated models removed
    "models_changed": 3,       // Models with updated capabilities
    "duration_ms": 1234,
    "api_status_code": 200,
    "error_message": string | null
  }
}

// discovery.poll_failed
{
  "event_type": "discovery.poll_failed",
  "payload": {
    "provider_id": "openai",
    "provider_type": "openai",
    "poll_type": "scheduled" | "targeted" | "manual",
    "failure_reason": "rate_limited" | "auth_error" | "timeout"
                   | "network_error" | "parse_error" | "server_error",
    "status_code": 429,
    "retry_count": 2,
    "next_retry_at": 1735689605000,
    "backoff_sec": 60
  }
}

// discovery.credential_trigger
{
  "event_type": "discovery.credential_trigger",
  "payload": {
    "provider_id": "anthropic",
    "credential_id": "cred-003",
    "change_type": "added" | "updated" | "removed" | "expired",
    "changed_by": "operator-alice",
    "changed_at": 1735689600000,
    "scheduled_refresh_in_ms": 5000  // Targeted refresh scheduled within 5s
  }
}
```

### Staleness Window Calculation

The system tracks staleness to warn operators when discovery data may be outdated:

```
Algorithm: ComputeStalenessWindow

Input:
  - last_successful_poll_at: number (Unix ms)
  - configured_interval_sec: number
  - adaptive_interval_sec: number | null  // null if using fixed
  - error_backoff_active: boolean

Output: staleness: StalenessLevel
        staleness_window_ms: number

01  elapsed_ms = now() - last_successful_poll_at
02  expected_interval_ms = (adaptive_interval_sec ?? configured_interval_sec) * 1000
03
04  if elapsed_ms <= expected_interval_ms:
05      return { level: "fresh", window_ms: 0 }
06
07  overage_ms = elapsed_ms - expected_interval_ms
08
09  if overage_ms <= expected_interval_ms * 0.5:
10      // Less than 1.5x the interval
11      return { level: "stale", window_ms: overage_ms }
12
13  if overage_ms <= expected_interval_ms * 2.0:
14      // Between 1.5x and 3x the interval
15      return { level: "degraded", window_ms: overage_ms }
16
17  // More than 3x the interval
18  if error_backoff_active:
19      return { level: "backoff", window_ms: overage_ms }
20  else:
21      return { level: "critical", window_ms: overage_ms }

// Staleness levels trigger different behaviors:
// fresh   → no action
// stale   → warn in UI: "Model data may be stale"
// degraded → warn + suggest manual refresh
// backoff → show "Backoff active: X errors" with next retry time
// critical → alert + auto-trigger refresh (unless in error backoff)
```

### Credentials Change Detection Mechanism

The system detects credential changes through multiple channels:

```
1. SCE Event Subscription (primary)
   └─ The DiscoveryService subscribes to SCE events matching:
        event_type IN ["provider.credential_*", "provider.config_updated"]
   └─ On event: schedule targeted refresh within 5 seconds
   └─ De-duplicate: if 3 credential events arrive in 10 seconds,
        run 1 refresh instead of 3 (coalesce by provider_id)

2. API Webhook (secondary)
   └─ External credential management systems can POST to:
        POST /api/v1/discovery/webhook/credential-change
   └─ Body: { provider_id, credential_id, change_type, timestamp }
   └─ Response: { scheduled: true, refresh_within_ms: 5000 }

3. Polling-based detection (fallback)
   └─ If SCE is unavailable, the scheduled poll compares
        credential metadata (last_rotated_at) with last poll timestamp
   └─ If credential changed since last poll: full refresh
   └─ Rate: checked every 60 seconds (lightweight, no API call)

4. Startup detection
   └─ On system start: check all credential modified_at timestamps
   └─ If any credential changed while system was offline: targeted refresh
   └─ Also runs if system clock jumps > 5 minutes (NTP correction, hibernation)
```

### v1.0 Verification Criteria

- [ ] Fixed 10-minute interval polls each provider on schedule
- [ ] Targeted refresh triggers within 5 seconds of credential change event
- [ ] Credential change event is de-duplicated (3 events in 10s → 1 refresh)
- [ ] Poll results are cached per-provider with correct TTL
- [ ] `last_success` timestamp is accurate and exposed in API/UI
- [ ] Rate limiting (1,440 requests/day with 10 providers) is validated with integration tests
- [ ] Poll failure triggers retry with exponential backoff (1s, 2s, 4s, 8s, 16s, 60s cap)
- [ ] Manual refresh via API works and resets the next scheduled poll timer

## Why fixed 10 minutes?

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

- [Nine Router](../NINE_ROUTER.md) — discovery trigger details
- [Job Scheduler](../JOB_SCHEDULER.md) — the cron engine that drives the interval

---

## Local-First Amendment

This ADR was originally framed around direct provider polling. Under the
Local-First architecture:

- **Discovery is a Nine Router responsibility.** Only Nine Router polls provider
  endpoints. Subsystems query Nine Router's model catalog — they never call
  providers directly.
- **Local providers are treated identically** to remote ones. Ollama, vLLM, and
  llama.cpp expose `/models` endpoints that Nine Router discovers on the same
  10-minute cycle.
- **Targeted refresh still applies.** Adding a local provider (e.g., starting
  Ollama) triggers a targeted refresh within 5 seconds, just like adding a cloud
  credential.
- **Scope unchanged.** The 10-minute interval, adaptive algorithm (v2.0), and
  staleness tracking all apply as specified above. Only the ownership of the
  poll shifts: Nine Router is the single poller.
