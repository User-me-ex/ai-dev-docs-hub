# Model Discovery

> The pipeline that queries each provider's `/models` endpoint, normalizes the response, groups by provider, and feeds the [Nine Router](./NINE_ROUTER.md).

## Overview

Model Discovery is how AI Dev OS learns what models exist. Users never hand-type model IDs; they see a live, filterable, provider-grouped catalog. Discovery runs on demand from the UI ("Refresh"), on schedule via [Job Scheduler](./JOB_SCHEDULER.md), and on provider-credential change.

## Goals

- Query each provider's `/models` (or documented equivalent) with a single normalized adapter interface.
- Group results by provider in a stable order: OpenAI, Anthropic, Google, Mistral, Ollama, Local (llama.cpp / MLX), MCP-exposed catalogs, user-registered.
- Deduplicate aliases (`gpt-4o` vs `gpt-4o-2024-08-06`) with a canonical id and an `aliases[]` list.
- Emit `models.discovery` events for added/removed/changed models so subscribers (Router UI, [Cost Management](./COST_MANAGEMENT.md)) update automatically.
- Degrade per provider: one provider failing MUST NOT block others.

## Non-Goals

- Model performance benchmarking — belongs in [Benchmarks](./BENCHMARKS.md).
- Cost accounting — belongs in [Cost Management](./COST_MANAGEMENT.md); Discovery only carries advertised pricing hints.
- Fallback/routing decisions — belong in [Model Routing Policy](./MODEL_ROUTING_POLICY.md).

## Requirements

- **MUST** support the following provider endpoints out of the box:

  | Provider  | Endpoint                                              | Auth                       |
  | --------- | ----------------------------------------------------- | -------------------------- |
  | OpenAI    | `GET https://api.openai.com/v1/models`                | `Authorization: Bearer …`  |
  | Anthropic | `GET https://api.anthropic.com/v1/models`             | `x-api-key`, `anthropic-version` |
  | Google    | `GET https://generativelanguage.googleapis.com/v1beta/models` | `?key=…` or ADC |
  | Mistral   | `GET https://api.mistral.ai/v1/models`                | `Authorization: Bearer …`  |
  | Ollama    | `GET http://localhost:11434/api/tags`                 | none (local)               |
  | llama.cpp | `GET http://localhost:8080/v1/models` (OpenAI-compatible) | none (local)           |
  | MLX       | filesystem scan of the model cache                    | none (local)               |
  | MCP       | `tools/list` on connected MCP servers exposing `model` resources | per MCP server  |

- **MUST** cache the last successful response per provider with a configurable TTL (default `10m`) and expose a manual **Refresh** action per provider and globally.
- **MUST** normalize every entry into the [canonical schema](#canonical-schema).
- **MUST** publish a `DiscoveryReport` event on `models.discovery` after every run.
- **MUST** never store provider credentials — pull from [Secrets Management](./SECRETS_MANAGEMENT.md) at call time.
- **SHOULD** support pluggable providers via the [Plugin SDK](./PLUGIN_SDK.md).
- **MAY** enrich entries with community metadata (context window, deprecation) when the provider omits it.

## Architecture

```mermaid
flowchart LR
  TRIG[Trigger: UI · Cron · Cred change] --> DISP[Dispatcher]
  DISP --> OAI[OpenAI Adapter]
  DISP --> ANT[Anthropic Adapter]
  DISP --> GOO[Google Adapter]
  DISP --> MIS[Mistral Adapter]
  DISP --> OLL[Ollama Adapter]
  DISP --> LOC[Local Adapter]
  DISP --> MCP[MCP Adapter]
  OAI & ANT & GOO & MIS & OLL & LOC & MCP --> NORM[Normalizer]
  NORM --> DEDUP[Alias Resolver]
  DEDUP --> CACHE[(TTL Cache)]
  DEDUP --> BUS[(SCE: models.discovery)]
  BUS --> ROUTER[Nine Router UI]
  BUS --> COST[Cost Management]
```

## Interfaces

```
discover(provider_id) → Model[]
refresh_all(force?: bool) → DiscoveryReport
list(filter?: ModelFilter) → Model[]        // reads cache; never hits providers
get(model_id) → Model
subscribe() → AsyncIterator<DiscoveryReport>
```

Errors follow [API Spec](./API_SPEC.md).

## Canonical Schema

```
Model {
  id:             string     # canonical: "openai/gpt-4o"
  provider:       "openai"|"anthropic"|"google"|"mistral"|"ollama"|"local"|"mcp"|string
  provider_model_id: string  # raw id returned by the provider
  aliases:        string[]   # e.g. ["gpt-4o-2024-08-06", "gpt-4o-latest"]
  display_name:   string
  family:         string?    # "gpt-4o", "claude-3.5", "gemini-1.5"
  context_window: number?    # tokens
  max_output_tokens: number?
  modalities:     ("text"|"image"|"audio"|"video")[]
  capabilities:   ("tools"|"vision"|"json_mode"|"streaming"|"embeddings"|"fine_tune")[]
  pricing: {                 # per 1M tokens, provider-advertised, USD
    input?: number
    output?: number
    cached_input?: number
  } | null
  deprecated:     boolean
  deprecation_date: rfc3339?
  status:         "available"|"limited"|"preview"|"retired"
  discovered_at:  rfc3339
  source:         "api"|"catalog"|"filesystem"|"mcp"|"user"
}

DiscoveryReport {
  ts:        rfc3339
  providers: { id, ok, duration_ms, count, error? }[]
  added:     Model[]
  removed:   { id, reason }[]
  changed:   { id, before, after }[]
  errors:    { provider, code, message }[]
}
```

### Grouping & sort order in the UI

The Nine Router UI MUST render providers in this order and MUST NOT hide a provider that returned an error — it renders empty with an error badge and a Retry action:

1. **Local** (Ollama, llama.cpp, MLX)  — because AI Dev OS is local-first
2. **OpenAI**
3. **Anthropic**
4. **Google**
5. **Mistral**
6. **MCP-exposed catalogs**
7. **User-registered** providers

Within a provider, sort by `family`, then `deprecated asc`, then `display_name`.

## Failure Modes

| Mode                        | Response                                                                 |
| --------------------------- | ------------------------------------------------------------------------ |
| HTTP 429 / rate limit       | Exponential backoff with full jitter; surface `provider.rate_limited`    |
| HTTP 5xx                    | Retry twice; on final failure keep last-known-good cache; badge Provider as degraded |
| Malformed response          | Skip provider for this cycle; publish `provider.error`; alert            |
| Auth failure                | Do not retry; surface a clear "Reconnect provider" action in UI          |
| Local endpoint unreachable  | Mark local provider "offline"; do not error the whole refresh            |
| Timeout (default 10s)       | Abort adapter; keep others; report timeout in `DiscoveryReport.errors`   |

## Security

- Credentials are read from [Secrets Management](./SECRETS_MANAGEMENT.md) per call; never logged, never persisted with the cache.
- Discovery responses may include experimental/preview model names — treat as public.
- All external calls go through the Kernel-proxied HTTP client so they show up in the [Audit Log](./AUDIT_LOG.md).

## Observability

- Metrics: `discovery_run_total{provider,ok}`, `discovery_duration_seconds{provider}`, `discovery_models_total{provider,status}`, `discovery_delta_total{kind=added|removed|changed}`.
- Traces: one span per adapter call; parent span per `refresh_all`.

## Acceptance Criteria

- Fresh install with only Ollama running yields a non-empty catalog with a single group.
- Revoking the OpenAI key surfaces a clear reconnect action and leaves other providers intact.
- A retired provider model appears in the next `DiscoveryReport.removed` within one refresh cycle.
- Adding a user-registered OpenAI-compatible endpoint (base URL + key) makes its models appear under **User-registered** without a code change.

## Related Documents

- [Nine Router](./NINE_ROUTER.md) · [Model Providers](./MODEL_PROVIDERS.md) · [Model Routing Policy](./MODEL_ROUTING_POLICY.md) · [Cost Management](./COST_MANAGEMENT.md) · [Secrets Management](./SECRETS_MANAGEMENT.md) · [Plugin SDK](./PLUGIN_SDK.md) · [diagrams/NINE_ROUTER_FLOW](../diagrams/NINE_ROUTER_FLOW.md)
