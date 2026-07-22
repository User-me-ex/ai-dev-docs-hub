# Impact Analysis

> The change-impact predictor consulted before edits land — maps proposed changes to affected documents, prompts, agents, and downstream subsystems, and scores risk. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

Impact Analysis (IA) is the pre-flight risk assessment that runs in parallel with the [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md)'s rule evaluation. While the Guardian asks "does this change violate a declared rule?", the Impact Analyser asks "what else will this change affect, and how risky is that blast radius?"

The IA builds its understanding from three sources: the [Obsidian Graph Engine](./OBSIDIAN_GRAPH_ENGINE.md) (structural links between documents, prompts, and code), the [Symbol Registry](./SYMBOL_REGISTRY.md) / [Function Registry](./FUNCTION_REGISTRY.md) (call graphs and dependency maps), and the [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md) (historical change-frequency and past regressions). It produces an `Impact` record that the Guardian includes in every `Verdict` and that the [Merge Manager](./MERGE_MANAGER.md) uses for conflict prioritisation.

## Goals

- Map every proposed change to its direct and transitive affected entities (docs, prompts, agents, APIs, tests).
- Score overall risk on a 0–1 scale with sub-scores and plain-English rationale.
- Surface suggested reviewers based on historical ownership of affected files.
- Feed the Guardian's `Verdict` and the Merge Manager's conflict priority queue.
- Keep analysis latency low: aim for p95 < 500 ms via parallel graph traversal and cached dependency maps.

## Non-Goals

- Enforcing rules — that is the Guardian's job.
- Generating fixes — the IA only describes impact, never modifies artifacts.
- Implementation code — this repository is documentation-only (see [AI Coding Rules](./AI_CODING_RULES.md)).

## Architecture

```mermaid
flowchart LR
  PATCH[Proposed patch\nFilePatch[]] --> IA[Impact Analyser]

  subgraph IA internals
    PARSE[Patch parser\nextracted symbols, links, paths]
    GRAPH[Graph traversal\nObsidian Graph Engine]
    SYM[Symbol lookup\nSymbol / Function Registry]
    HIST[History analyser\nSCE + Audit Log]
    SCORE[Risk scorer]
    REVIEW[Reviewer suggester]
  end

  IA --> PARSE
  PARSE --> GRAPH
  PARSE --> SYM
  PARSE --> HIST

  GRAPH --> SCORE
  SYM --> SCORE
  HIST --> SCORE
  SCORE --> REVIEW

  REVIEW --> IMPACT[Impact record]
  IMPACT --> GUARD[Architecture Guardian]
  IMPACT --> MM[Merge Manager]
  IMPACT --> SCE[(SCE: impact.reports)]
```

## Interface

```
impact.analyze(patch: FilePatch[]) → Impact
impact.analyze_file(path: string, before: string, after: string) → Impact
impact.batch(patches: FilePatch[][]) → Impact[]   # parallel analysis

# Introspection
impact.dependency_map(path: string) → DependencyNode[]
impact.ownership(paths: string[]) → { path, owners: string[] }[]
```

### Impact schema

```
Impact {
  id:              ulid
  patch_id:        string          # hash of the input patch set
  ts:              rfc3339

  affected: {
    path:          string
    kind:          "direct"|"transitive"|"semantic"
    entity_type:   "doc"|"prompt"|"diagram"|"template"|"config"|"test"|"agent"|"api"
    link_path:     string[]        # how we got here (graph path from changed file)
    change_type:   "modified"|"deleted"|"linked"|"semantic"
  }[]

  risk:            number          # 0.0 – 1.0 overall score
  risk_breakdown: {
    blast_radius:  number          # 0–1: how many entities are affected
    centrality:    number          # 0–1: are affected files highly connected?
    volatility:    number          # 0–1: do affected files change often?
    regression_history: number     # 0–1: have similar changes caused regressions?
    test_coverage: number          # 0–1 inverse: how much of the blast radius is untested?
  }
  rationale:       string          # 2–5 sentence plain-English explanation
  suggested_reviewers: {
    id:            string
    reason:        string          # e.g. "authored 70% of affected files in last 90d"
  }[]
  stale:           boolean         # true if graph was rebuilt since analysis started
}
```

## Risk Scoring

The overall risk score is a weighted average of the five sub-scores:

```
risk = 0.30 * blast_radius
     + 0.25 * centrality
     + 0.20 * volatility
     + 0.15 * regression_history
     + 0.10 * test_coverage_inverse
```

### Sub-score computation

**Blast radius** — fraction of the total knowledge graph affected:
```
blast_radius = min(1.0, len(affected) / blast_radius_denominator)
# blast_radius_denominator default: 50 entities
```

**Centrality** — average PageRank of affected nodes, normalised to [0,1]:
```
centrality = mean(graph.pagerank(a.path) for a in affected) / max_pagerank
```

**Volatility** — fraction of affected files that changed more than `volatility_change_threshold` times in the last `volatility_window`:
```
volatility = len([a for a in affected if change_count(a.path, 30d) > 5]) / len(affected)
```

**Regression history** — fraction of affected files that were involved in past regressions (Guardian vetoes):
```
regression_history = len([a for a in affected if has_veto_in(a.path, 90d)]) / len(affected)
```

**Test coverage inverse** — fraction of affected entities with no associated test:
```
test_coverage_inverse = 1.0 - (tested_entities / len(affected))
```

### Risk thresholds used by consumers

| Risk range | Label | Guardian behaviour | Merge Manager behaviour |
|------------|-------|--------------------|------------------------|
| 0.0 – 0.3 | Low | Include in Verdict; no extra checks | Normal priority |
| 0.3 – 0.6 | Medium | Include in Verdict; add hint | Elevated priority in conflict queue |
| 0.6 – 0.8 | High | Include in Verdict; add high-severity warning hint | Block concurrent merges to same files |
| 0.8 – 1.0 | Critical | Add synthetic high violation to Verdict | Require human approval before commit |

## Dependency Map

The IA maintains a cached dependency map derived from:

1. **Graph edges** from the [Obsidian Graph Engine](./OBSIDIAN_GRAPH_ENGINE.md): wikilinks, backlinks, embeds.
2. **Symbol references** from the [Symbol Registry](./SYMBOL_REGISTRY.md): which documents and prompts reference which symbols.
3. **Prompt bindings**: which prompts reference which model roles or KB tiers.
4. **API surface**: changes to [API Spec](./API_SPEC.md) entities propagate to all documented consumers.

The map is rebuilt on every `graph.update` SCE event. When the patch touches a file in the map, the IA traverses up to `transitive_depth` (default 3) hops to collect `transitive` affected entities.

Semantic edges (cosine similarity > `semantic_edge_threshold`) produce `kind: "semantic"` affected entries — these are lower-confidence and weighted at 0.5x in the blast radius calculation.

## Suggested Reviewers

Reviewer suggestion is based on three signals:

1. **Authorship**: agents or users who authored ≥ 20% of lines in affected files in the last 90 d.
2. **Expertise tags**: agents whose `GroupSpec` tags overlap with the affected files' front matter tags.
3. **Past reviewer**: agents who reviewed a Guardian verdict for any affected file in the last 90 d.

Reviewers are de-duplicated and ranked by signal strength. Maximum 5 reviewers suggested per Impact record.

## Requirements

- **MUST** run in parallel with the Guardian's rule evaluation; the Guardian MUST NOT wait for IA before starting rule checks.
- **MUST** complete within `impact_timeout_ms` (default 2000 ms); if the timeout is exceeded, emit a `guardian.impact_timeout` warning and return a partial `Impact` with `stale: true`.
- **MUST** traverse the Obsidian graph for direct (depth 1) and transitive (depth up to `transitive_depth`) affected entities.
- **MUST** include a `rationale` string of 2–5 sentences in plain English.
- **MUST** emit `impact.reports` events on the SCE after every analysis.
- **SHOULD** cache the dependency map with a short TTL (default 60 s); invalidate on `graph.update` events.
- **SHOULD** surface `suggested_reviewers` with a `reason` string for each reviewer.
- **MAY** use a model call to generate the `rationale` when the patch is large and complex.
- **MAY** expose `impact.dependency_map(path)` for interactive exploration via the CLI and UI.

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| Graph stale / rebuilding | `graph.stats().rebuilding == true` | Mark result `stale: true`; use last-known-good snapshot; continue |
| Analysis timeout | Wall time > `impact_timeout_ms` | Return partial result with `stale: true`; emit warning |
| Symbol Registry unavailable | Registry read error | Skip symbol-based traversal; proceed with graph-only analysis |
| Empty patch | No file changes in patch | Return `Impact { affected: [], risk: 0.0, rationale: "No changes detected" }` |
| Graph unavailable | OGE connection error | Return minimal impact with `risk: 0.5` (conservative default); emit warning |

Every failure emits a structured event on the SCE and is recorded in the [Audit Log](./AUDIT_LOG.md).

## Security Considerations

- The IA is read-only: it never modifies any file, registry, or graph node.
- `suggested_reviewers` are suggestions only; the decision to involve a reviewer is always the operator's.
- Dependency maps may reveal system topology; access is governed by [AuthZ/RBAC](./AUTHZ_RBAC.md).
- See [Security Model](./SECURITY_MODEL.md).

## Observability

| Metric | Labels | Description |
|--------|--------|-------------|
| `impact_analysis_total` | `risk_label` | Analyses by risk tier |
| `impact_analysis_seconds` | — | Analysis latency histogram |
| `impact_affected_count` | `kind` | Affected entities per analysis |
| `impact_timeout_total` | — | Timed-out analyses |
| `impact_stale_total` | — | Analyses with stale graph |
| `impact_risk_score` | — | Histogram of overall risk scores |

Traces: one span per `impact.analyze` call; child spans for graph traversal, symbol lookup, and history query. See [Tracing](./TRACING.md).

## Acceptance Criteria

- Changing `docs/NINE_ROUTER.md` produces an `Impact` listing all documents that wikilink to `NINE_ROUTER.md` as `kind: "direct"` affected entities.
- Changing a root document with high PageRank (e.g., `docs/MAIN_AI_KERNEL.md`) produces `risk > 0.6`.
- Analysis completes in ≤ 500 ms (p95) for patches affecting ≤ 10 files on a warmed dependency-map cache.
- An analysis with `risk > 0.8` causes the Merge Manager to require human approval before the commit proceeds.
- The `rationale` field contains a non-empty, human-readable explanation for every non-trivial analysis.

## Open Questions

- Whether semantic edges should contribute to blast radius at full weight or half weight — currently set at 0.5x as a conservative default; tracked in [templates/ADR](../templates/ADR.md).
- Whether `suggested_reviewers` should include agent IDs (for automated review) or only human user IDs.

## Related Documents

- [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) — primary consumer of `Impact`
- [Merge Manager](./MERGE_MANAGER.md) — uses risk score for conflict priority
- [Obsidian Graph Engine](./OBSIDIAN_GRAPH_ENGINE.md) — graph traversal source
- [Symbol Registry](./SYMBOL_REGISTRY.md)
- [Function Registry](./FUNCTION_REGISTRY.md)
- [Audit Log](./AUDIT_LOG.md)
- [System Overview](./SYSTEM_OVERVIEW.md)
- [Main AI Kernel](./MAIN_AI_KERNEL.md)
