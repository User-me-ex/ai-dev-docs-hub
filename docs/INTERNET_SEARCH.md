# Internet Search

> Pluggable web-search backend abstraction with a unified result schema. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

Internet Search provides a uniform interface over multiple search backends. Agents call a single `search.query(q, opts?)` method and receive results in a standard schema regardless of which backend is active. This decouples agent reasoning from backend choice and allows operators to switch or combine backends without changing any agent prompt.

The subsystem is used in three contexts:
- **Direct** — an agent calls `search.query` with a search string and optional filters.
- **Research Engine integration** — the [Research Engine](./RESEARCH_ENGINE.md) uses search results as a discovery mechanism to find new source URLs.
- **Web Intelligence integration** — search results that point to JS-heavy pages are handed to [Web Intelligence](./WEB_INTELLIGENCE.md) for rendering.

## Goals

- Abstract over multiple search backends with a stable, unified result schema.
- Support backend-specific configuration (API endpoints, rate limits, auth) per environment.
- Cache recent results to reduce API cost and latency.
- Respect rate limits and handle backoff transparently.
- Support standard search operators (site:, filetype:, intitle:, etc.) where the backend allows them.
- Operate through firewalls and proxies common in enterprise environments.

## Non-Goals

- Web crawling or content extraction — that is [Web Intelligence](./WEB_INTELLIGENCE.md).
- Document ranking beyond what the backend provides — ranking is the search engine's responsibility.
- Indexing local content — use [Vector Store](./VECTOR_STORE.md) for local semantic search.

## Supported Backends

| Backend | Type | API Key Required | Rate Limit | Notes |
|---------|------|------------------|------------|-------|
| **SearXNG** (local) | Self-hosted meta-search | No | Configurable | Preferred for privacy; no external dependency |
| **Brave Search** | SaaS | Yes | 20 000 req/month (free) | Good web + news + image results |
| **Google Programmable Search** | SaaS (API) | Yes | 100 queries/day (free tier) | Best result quality; expensive at scale |
| **Bing Search** | SaaS (Azure) | Yes | 1 000 req/month (free) | Enterprise-friendly; good for news |
| **DuckDuckGo** | Web scraping | No | ~1 req/s (unbounded) | No official API; uses lightweight HTML scraping |

Backend selection is configured in the environment via one of these mechanisms:
```
# Active backend (workspace-wide)
SEARCH_BACKEND = "brave"

# Per-backend credentials
SEARCH__BRAVE__API_KEY = "..."
SEARCH__GOOGLE__API_KEY = "..."
SEARCH__GOOGLE__CX = "..."      # custom search engine ID
SEARCH__BING__API_KEY = "..."
SEARCH__SEARXNG__BASE_URL = "http://searxng:8080"
```

All API keys are read via [Secrets Management](./SECRETS_MANAGEMENT.md); they MUST NOT appear in config files, environment dumps, or agent prompts.

## Unified Search Result Schema

Every backend maps to this standard result type:

```
SearchResult {
  index:        number        # rank position (1-based)
  title:        string
  url:          string
  snippet:      string        # ~200 char excerpt
  source:       string        # backend identifier ("brave", "google", etc.)
  published?:   rfc3339       # publication date (if available)
  site_name?:   string        # e.g. "GitHub", "Wikipedia"
  content_type: string        # "web" | "news" | "image" | "video"
  thumbnail?:   string        # URL to thumbnail (news, image results)
  extra?:       { }           # backend-specific fields
}

SearchResponse {
  query:          string
  results:        SearchResult[]
  total_estimated: number     # estimated total results (backend-provided)
  page:           number      # current page
  backend:        string
  took_ms:        number
  cached:         boolean     # true if served from cache
  truncated:      boolean     # true if results exceeded max_results
}
```

## Result Ranking

By default results are returned in the backend's native order. The Internet Search layer MAY apply post-processing when `ranking` is specified:

```
search.query("kubernetes ingress", { ranking: "recency" })
```

Supported ranking modes:
- `relevance` — backend-native (default).
- `recency` — sort by `published` descending; results without dates go to the end.
- `source_priority` — boost results from specific `site_name` values (e.g. GitHub > dev.to > medium).

Ranking is always applied after the backend returns results; it never modifies the backend's own ranking signal for pagination.

## Caching

Search results are cached in [Persistent Memory](./PERSISTENT_MEMORY.md) with a configurable TTL:

| Cache Level | TTL | Description |
|-------------|-----|-------------|
| `aggressive` | 1 h | Same query returns cached results within TTL |
| `default` | 10 min | Balance of freshness and cost |
| `none` | 0 | Always hits the backend |

Cache key: `sha256(normalize(query) + backend + page)`. Normalisation lowercases the query, strips excess whitespace, and sorts operators.

## Integration with Research Engine and Web Intelligence

```
Research Engine ──→ source URL unknown ──→ search.query("latest react router docs")
                      │
                      ▼
              SearchResult[] ──→ discovered URLs added as ResearchJob sources
                      │
                      ▼ (JS-heavy page detected)
              Web Intelligence ──→ web.browse(playbook)
```

When the Research Engine discovers a new topic to research, it uses Internet Search to find candidate source URLs. Each discovered URL is evaluated: if the URL's content type requires JS rendering (SPA patterns, client-side frameworks), the source is configured with `web_intelligence` and delegated to [Web Intelligence](./WEB_INTELLIGENCE.md).

## Search Operators

Operators are passed through to the backend when supported. The unified interface recognises these:

| Operator | Example | Backend Support |
|----------|---------|-----------------|
| `site:` | `site:react.dev` | All |
| `filetype:` | `filetype:pdf` | Google, Bing |
| `intitle:` | `intitle:API reference` | Google, SearXNG |
| `inurl:` | `inurl:changelog` | Google, SearXNG |
| `before:` / `after:` | `after:2025-01-01` | Google, Brave, Bing |
| `source:` | `source:news` | Brave, Bing |
| `-` (exclude) | `kubernetes -helm` | All |

Backends that do not support a given operator silently ignore it. The response includes a `query_effective` field showing what was actually sent.

## Firewall and Proxy Configuration

Enterprise deployments often require outbound search traffic through a proxy:

```
SEARCH_PROXY = "http://proxy.company.com:8080"
SEARCH_PROXY_BYPASS = ["*.internal.com", "10.*"]
```

When `SEARCH_PROXY` is set, all backend requests (including SearXNG if it is remote) go through the proxy. The proxy is configured once at startup from the environment; it cannot be changed per-request. See [Network Security](./SECURITY_MODEL.md) for proxy trust requirements.

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| Backend unavailable | HTTP connection error / DNS failure | Failover to next backend in priority list; emit `search.backend_failover` |
| Rate limited | HTTP 429 | Wait for `Retry-After` header (max 60 s) then retry once; fail if still 429 |
| API key expired / invalid | HTTP 401 / 403 | Fail with `auth_error`; alert operator via `search.auth_error` metric |
| No results | Empty `results[]` | Return empty response; set `total_estimated: 0` |
| Backend timeout | No response within `backend_timeout_ms` (default 5 000) | Failover to next backend; emit `search.timeout` |
| Query too long | Backend returns 400 or truncates | Retry with truncated query; set `truncated: true` |
| Proxy failure | Proxy connection refused | Fall back to direct connection if `SEARCH_PROXY_FAILOVER=true` |

Every failure emits a structured event on the SCE and is recorded in the [Audit Log](./AUDIT_LOG.md).

## Observability

| Metric | Labels | Description |
|--------|--------|-------------|
| `search_query_total` | `backend`, `status=success\|failure` | Query outcomes |
| `search_query_seconds` | `backend` | Query duration histogram |
| `search_cache_hit_total` | `backend` | Cache hits |
| `search_cache_miss_total` | `backend` | Cache misses |
| `search_rate_limited_total` | `backend` | 429 responses |
| `search_backend_failover_total` | `from_backend`, `to_backend` | Failover events |
| `search_result_count` | `backend` | Results per query histogram |
| `search_auth_error_total` | `backend` | Auth failures |

Traces: one span per `search.query` call, with a child span for the backend HTTP request. Cached responses skip the backend span.

## Acceptance Criteria

- `search.query("react hooks")` returns at least one result with a `url`, `title`, and `snippet` regardless of active backend.
- The same query repeated within cache TTL returns `cached: true` and does not affect rate-limit counters.
- When the primary backend returns 429, the system transparently failovers to the secondary backend and records the event in metrics.
- Setting `SEARCH_BACKEND` to an unrecognised value fails at startup with a clear error.

## Related Documents

- [Web Intelligence](./WEB_INTELLIGENCE.md)
- [Research Engine](./RESEARCH_ENGINE.md)
- [GitHub Analysis](./GITHUB_ANALYSIS.md)
- [Secrets Management](./SECRETS_MANAGEMENT.md)
- [Security Model](./SECURITY_MODEL.md)
- [Observability](./OBSERVABILITY.md)
- [Persistent Memory](./PERSISTENT_MEMORY.md)
- [System Overview](./SYSTEM_OVERVIEW.md)
- [Main AI Kernel](./MAIN_AI_KERNEL.md)
