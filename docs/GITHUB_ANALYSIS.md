# GitHub Analysis

> Automated repository analysis via the GitHub REST and GraphQL APIs. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

GitHub Analysis is the repository-intelligence subsystem of AI Dev OS. It retrieves and analyses public (and authorised private) GitHub repositories on demand or on a schedule. Analysis results are written as structured Markdown into the [Knowledge System](./KNOWLEDGE_SYSTEM.md), where agents can query them via [RAG Pipeline](./RAG_PIPELINE.md) lookups.

The subsystem answers questions like:
- "What is the directory structure of this repo?"
- "What open issues have the `good first issue` label?"
- "What did the last 3 releases contain?"
- "Which files changed the most in the last 30 days?"
- "What is the bus factor for the main module?"

## Goals

- Analyse six dimensions of a repository: structure, issues, PRs, releases, commits, and codebase metrics.
- Use both REST (for lists and paginated resources) and GraphQL (for batched, nested queries) to stay within GitHub API rate limits.
- Authenticate via a GitHub token provisioned through [Secrets Management](./SECRETS_MANAGEMENT.md).
- Write analysis results as structured, queryable KB entries with provenance.
- Support periodic re-analysis via the [Job Scheduler](./JOB_SCHEDULER.md) to keep repository knowledge fresh.
- Stay within GitHub API rate limits; degrade gracefully when limits are exceeded.

## Non-Goals

- Performing CI/CD operations (creating PRs, merging, deploying) — use external tooling for that.
- Mirroring the full repository database — only analysed summaries are stored.
- Writing code changes — see [AI Coding Rules](./AI_CODING_RULES.md).

## Analysis Types

| Type | REST Endpoint | GraphQL Query | Output |
|------|---------------|---------------|--------|
| **Repository structure** | `GET /repos/{owner}/{repo}/git/trees/{branch}?recursive=1` | `repository.object` (Tree) | Directory tree Markdown with file sizes |
| **Issue analysis** | `GET /repos/{owner}/{repo}/issues?state=open&labels=...` | `repository.issues(first:100)` | Table of issues with labels, age, assignee |
| **PR analysis** | `GET /repos/{owner}/{repo}/pulls?state=open` | `repository.pullRequests(first:100)` | Table of PRs with status, review count, mergeability |
| **Release analysis** | `GET /repos/{owner}/{repo}/releases?per_page=10` | `repository.releases(last:10)` | Release notes, tags, dates, changelog summaries |
| **Commit history** | `GET /repos/{owner}/{repo}/commits?since={date}` | `repository.ref.target.history` | Commit list with authors, files changed, stats |
| **Codebase metrics** | Aggregate from above | `repository.languages, repository.object` | Language breakdown, file count, bus factor estimation |

## GitHub API Integration

All API calls go through a shared HTTP client that:
- Uses **REST** for single-resource fetches and large paginated lists (issues, commits).
- Uses **GraphQL** for batched queries (e.g. repository metadata + languages + default branch in one call).
- Respects `X-RateLimit-Remaining` and pauses when the remaining budget drops below `min_remaining` (default 10).
- Follows `Link` headers for REST pagination automatically up to `max_pages` (default 5).

The GraphQL endpoint is `https://api.github.com/graphql`. The REST endpoint is `https://api.github.com`. For GitHub Enterprise, the base URL is configurable:

```
GITHUB_API_BASE_URL = "https://github.company.com/api/v3"
GITHUB_GRAPHQL_URL  = "https://github.company.com/api/graphql"
```

## Authentication

Authentication uses a personal access token (classic or fine-grained) with appropriate scopes:

```
GITHUB_TOKEN = "ghp_..."       # public repos + private if repo scope
```

For fine-grained tokens, the required permissions depend on the analysis type:

| Permission | Required For |
|------------|-------------|
| `Contents: read` | Repository structure, commit history |
| `Issues: read` | Issue analysis |
| `Pull requests: read` | PR analysis |
| `Metadata: read` | All (implicit for public repos) |

The token is provisioned through [Secrets Management](./SECRETS_MANAGEMENT.md) and **never** logged, included in KB entries, or exposed to agents.

## Analysis Output Schema

Every analysis produces a KB entry with the following structure:

```
KbEntry {
  kind:     "github_analysis"
  key:      f"github:{owner}/{repo}:{analysis_type}:{ref}"
  text:     markdown_string      # formatted analysis
  tags: [
    "github_analysis",
    "github_repo:{owner}/{repo}",
    "github_analysis_type:{analysis_type}",
    "github_ref:{ref}"
  ]
  refs: [
    { kind: "github_api_response", id: response_hash }
  ]
  metadata: {
    analysis_type: string
    owner:         string
    repo:          string
    ref:           string          # branch or tag
    run_at:        rfc3339
    cached_until:  rfc3339         # next scheduled re-analysis
    api_calls_used: number
    token_scopes:  string[]
  }
  retention: "30d"                 # re-analysis refreshes the entry
}
```

The `text` field is a structured Markdown document with tables, code blocks (for directory trees), and summary paragraphs. The format is designed to be both human-readable and easily parseable by agents via RAG chunking.

## Scheduling Periodic Analysis

Analysis can be scheduled via the [Job Scheduler](./JOB_SCHEDULER.md):

```
Job {
  type:     "github_analysis"
  config: {
    owner:    "vercel"
    repo:     "next.js"
    types:    ["structure", "issues", "releases"]
    ref:      "canary"
    cadence:  "@daily"
    kb_target: "group"
  }
}
```

When a scheduled analysis runs, it always overwrites the previous entry for the same `key`. This ensures agents always see the freshest analysis for a given `{owner}/{repo}:{analysis_type}:{ref}`.

## Integration with Research Engine and Knowledge System

```
Research Engine ──→ discovers new repo ──→ GitHub Analysis ──→ KB entry
                                                                    │
Agent query ╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌→ RAG Pipeline
                                                                    │
                                                                    ▼
                                                            Structured answer
```

The [Research Engine](./RESEARCH_ENGINE.md) can trigger GitHub Analysis as a specialised source type (`source.type: "github_analysis"`). When a research job targets a GitHub repository, the Research Engine calls GitHub Analysis rather than attempting a raw HTTP crawl of the repo's pages.

## Rate Limit Handling

GitHub imposes rate limits:
- **Unauthenticated**: 60 requests/hour (REST) — not recommended for analysis.
- **Authenticated**: 5 000 requests/hour (REST), 5 000 points/hour (GraphQL).

The client tracks remaining budget and adapts:

```
if remaining_requests < min_remaining (default 10):
    pause until reset_at + 1 second
if remaining_requests < critical_threshold (default 50):
    switch to low-throughput mode: 1 request per 2 seconds
```

When rate-limited mid-analysis, partial results are saved with a `rate_limited: true` flag in the KB entry. The next scheduled run will continue from where it left off (for paginated analyses) using the `cursor` stored in the entry metadata.

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| Token expired / invalid | HTTP 401 | Fail with `auth_error`; alert operator via `github.auth_error` |
| Repository not found | HTTP 404 | Fail with `repo_not_found`; no KB entry written |
| API rate limit | HTTP 403 + `rate limit` in body | Save partial results; set `rate_limited: true`; schedule retry at `reset_at` |
| GraphQL query too complex | HTTP 422 + "Something went wrong" | Split query into smaller (REST) requests; degrade gracefully |
| Network error | Connection timeout / DNS failure | Retry with 3 s / 10 s / 30 s backoff; fail on 4th attempt |
| Repository empty | API returns empty tree | Write KB entry with `empty_repo: true` and minimal metadata |
| Analysis type not supported | Invalid `analysis_type` in config | Fail with `invalid_analysis_type`; no KB entry written |
| Branch not found | `GET /repos/{owner}/{repo}/branches/{ref}` returns 404 | Fall back to default branch; include warning in KB entry |

Every failure emits a structured event on the SCE and is recorded in the [Audit Log](./AUDIT_LOG.md).

## Observability

| Metric | Labels | Description |
|--------|--------|-------------|
| `github_analysis_total` | `analysis_type`, `status=success\|failure` | Analysis outcomes |
| `github_analysis_seconds` | `analysis_type` | Duration histogram |
| `github_api_call_total` | `api=rest\|graphql`, `endpoint` | API calls made |
| `github_api_error_total` | `status_code`, `endpoint` | API errors |
| `github_rate_limited_total` | — | Rate limit encounters |
| `github_token_expired_total` | — | Auth failures |
| `github_analysis_cache_hit_total` | `analysis_type` | KB entries served from stale cache |
| `github_items_fetched` | `analysis_type` | Issues/PRs/commits per analysis |

Traces: one span per analysis call, with child spans for each API request and the final KB write.

## Acceptance Criteria

- Analysing `vercel/next.js` for `structure` produces a KB entry with a directory tree containing at least `packages/next/`.
- Analysing `vercel/next.js` for `issues` returns only open issues with their labels and ages.
- An expired token causes a clear `auth_error` failure with no partial KB entry written.
- Scheduling `@daily` analysis on a repo produces exactly one new KB entry per day for each analysis type.

## Related Documents

- [Research Engine](./RESEARCH_ENGINE.md)
- [Internet Search](./INTERNET_SEARCH.md)
- [Web Intelligence](./WEB_INTELLIGENCE.md)
- [Knowledge System](./KNOWLEDGE_SYSTEM.md)
- [Secrets Management](./SECRETS_MANAGEMENT.md)
- [Job Scheduler](./JOB_SCHEDULER.md)
- [Persistent Memory](./PERSISTENT_MEMORY.md)
- [RAG Pipeline](./RAG_PIPELINE.md)
- [Observability](./OBSERVABILITY.md)
- [System Overview](./SYSTEM_OVERVIEW.md)
- [Main AI Kernel](./MAIN_AI_KERNEL.md)
