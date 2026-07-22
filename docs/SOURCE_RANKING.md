# Source Ranking

> Ranking system for external information sources — authority scoring, freshness weighting, and relevance computation used by the Research Engine and RAG Pipeline to prioritise high-quality sources. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

Source Ranking determines the relative quality and trustworthiness of information sources that the [Research Engine](./RESEARCH_ENGINE.md) crawls and the [RAG Pipeline](./RAG_PIPELINE.md) retrieves from. It produces a normalised score (0.0–1.0) for each source that is used to:

1. **Prioritise crawling**: the Research Engine crawls high-ranked sources before low-ranked ones.
2. **Weight retrieval**: the RAG Pipeline weights results from high-ranked sources higher in relevance scoring.
3. **Filter**: agents can set a minimum authority threshold for knowledge retrieval (`min_authority: 0.5`).

Source Ranking is consumed by the [Citation Engine](./CITATION_ENGINE.md) as one input to the overall authority score.

## Goals

- Every source URI can be scored on demand with no human input.
- The ranking model is transparent: the factors that produce a score are returned alongside the score.
- Source rankings are cached per-domain per-source-type to avoid recomputation on every request.
- Ranking is configurable: operators can override domain scores with custom values.

## Non-Goals

- Ranking model outputs or research results — the research content is ranked after retrieval, not before.
- Blocking sources — Source Ranking scores sources; blocking is a separate access-control function.
- Implementation code — this repo is documentation-only ([AI Coding Rules](./AI_CODING_RULES.md)).

## Ranking Factors

| Factor | Weight | Description | Data Source |
|--------|--------|-------------|-------------|
| **Domain authority** | 0.30 | Pre-computed score for known domains (documentation, academic, community) | Built-in domain registry |
| **Source type** | 0.20 | Official documentation (1.0), GitHub (0.7), npm (0.6), arXiv (0.8), community blog (0.4), forum (0.3) | URI pattern matching |
| **Age** | 0.10 | Decay over time: full score at publish, linear decay to 0.3 after 2 years | Source publication date |
| **Citation count** | 0.15 | Number of other KB entries that reference this source | Citation Engine |
| **Freshness** | 0.10 | Whether the source content has been verified recently | Citation Engine freshness |
| **Cross-validation** | 0.10 | How many independent sources agree on the same information | RAG Pipeline result correlation |
| **Human verification** | 0.05 | Whether a human operator has explicitly verified this source | Citation Engine verify() |

## Domain Authority Registry

Built-in domain authority scores:

| Domain | Score | Category |
|--------|-------|----------|
| `developer.mozilla.org` | 1.0 | Official docs |
| `docs.python.org` | 1.0 | Official docs |
| `nodejs.org` | 1.0 | Official docs |
| `react.dev` | 1.0 | Official docs |
| `github.com` | 0.7 | Code repository |
| `arxiv.org` | 0.8 | Academic |
| `news.ycombinator.com` | 0.4 | Community |
| `stackoverflow.com` | 0.5 | Community |
| `medium.com` | 0.3 | Community blog |
| `dev.to` | 0.3 | Community blog |
| Unknown domain | 0.2 | Default |

Operators can add or override entries in `~/.aidevos/source-ranking.toml`:

```toml
[domain_overrides]
"internal.docs.mycompany.com" = 0.95
"legacy-wiki.mycompany.com" = 0.5
```

## Scoring Algorithm

```python
def score_source(uri: str, metadata: SourceMetadata) -> SourceRank:
    factors = []

    # 1. Domain authority
    domain = extract_domain(uri)
    domain_score = DOMAIN_REGISTRY.get(domain, 0.2)
    factors.append(Factor("domain_authority", 0.30, domain_score))

    # 2. Source type
    source_type = classify_source(uri)
    type_score = SOURCE_TYPE_SCORES.get(source_type, 0.3)
    factors.append(Factor("source_type", 0.20, type_score))

    # 3. Age decay
    if metadata.published_at:
        age_days = (now() - metadata.published_at).days
        age_score = max(0.3, 1.0 - (age_days / 730))  # linear decay over 2 years
    else:
        age_score = 0.5  # unknown age → moderate score
    factors.append(Factor("age", 0.10, age_score))

    # 4. Citation count
    citation_count = citation_engine.count_for_uri(uri)
    citation_score = min(1.0, citation_count / 10)  # 10+ citations = max score
    factors.append(Factor("citation_count", 0.15, citation_score))

    # 5. Freshness
    freshness_score = citation_engine.freshness_for_uri(uri)  # 0.0 = stale, 1.0 = fresh
    factors.append(Factor("freshness", 0.10, freshness_score))

    # 6. Cross-validation
    cross_val_score = compute_cross_validation(uri)  # 0.0–1.0
    factors.append(Factor("cross_validation", 0.10, cross_val_score))

    # 7. Human verification
    verified = citation_engine.is_verified(uri)
    factors.append(Factor("human_verification", 0.05, 1.0 if verified else 0.0))

    total = sum(f.weight * f.score for f in factors)
    return SourceRank(score=total, factors=factors)
```

## Ranking Tiers

| Score Range | Tier | Meaning | Research Priority |
|-------------|------|---------|-------------------|
| 0.8–1.0 | **Authoritative** | Official documentation, peer-reviewed, human-verified | Highest priority |
| 0.6–0.79 | **Trusted** | Well-known community sources, cross-validated | Normal priority |
| 0.4–0.59 | **Adequate** | Community blogs, unverified sources, recent | Lower priority |
| 0.2–0.39 | **Low confidence** | Unknown domains, stale content, unvalidated | Crawl only if necessary |
| 0.0–0.19 | **Unverified** | Never encountered, no metadata | Skip by default |

## Usage in Research Engine

The Research Engine uses source ranking to:

1. **Order crawl queue**: crawl authoritative sources first (schedule within 5 minutes), trusted sources next (within 1 hour), adequate sources on next schedule (within 6 hours), low-confidence sources on demand only.
2. **Filter results**: by default, return only sources with score ≥ 0.4 (Adequate+). The calling agent can override with `min_rank`.
3. **Weight synthesis**: when synthesising results from multiple sources, higher-ranked sources get proportionally more weight in the final summary.

## Interfaces

```
source_ranking.score(uri: string, metadata?: SourceMetadata) → SourceRank
source_ranking.batch(uris: string[]) → Map<string, SourceRank>
source_ranking.domain(domain: string) → float  # cached domain score
source_ranking.override(domain: string, score: float) → Ack  # operator override
source_ranking.pattern(source_type: string) → float  # source type baseline score
```

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| Unknown domain | Not in registry, no metadata | Return 0.2 default score; create `source_ranking.unknown_domain` event |
| Metadata unavailable | No published_at, no author | Score with partial factors (weight-adjusted); log DEBUG |
| Override file corrupt | TOML parse error | Fall back to built-in registry; log ERROR |
| Cross-validation slow | Computation > 1s | Serve score without cross-validation factor; compute asynchronously |
| Score instability | Same URI returns different score within 5 minutes | Log WARN; freeze score for 5 minutes; investigate factor volatility |

## Acceptance Criteria

- `source_ranking.score("https://docs.python.org/3/tutorial/")` returns a SourceRank with `score ≥ 0.8` and factors including `domain_authority = 1.0`.
- `source_ranking.score("https://unknown-personal-blog.com/random-post")` returns a SourceRank with `score ≈ 0.2` (unknown domain default).
- Setting `[domain_overrides]` in config overrides the built-in domain score for the matching domain.
- A URI with 15 citations scores higher on the `citation_count` factor than one with 2 citations.
- A source that has been human-verified through `citation.verify()` has a higher rank than an equivalent unverified source.

## Related Documents

- [Citation Engine](./CITATION_ENGINE.md) — provides citation count and freshness data used by ranking
- [Research Engine](./RESEARCH_ENGINE.md) — consumes rankings for crawl prioritisation
- [RAG Pipeline](./RAG_PIPELINE.md) — uses rankings for weighted retrieval
- [Research Cache](./RESEARCH_CACHE.md) — caching layer that feeds source metadata
- [Knowledge System](./KNOWLEDGE_SYSTEM.md) — stores ranked KB records
- [System Overview](./SYSTEM_OVERVIEW.md)
