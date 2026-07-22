# Model Routing Policy

> Deterministic policy that maps role → model using capability-first routing, then latency, then cost. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

The Model Routing Policy is the rule engine the [Nine Router](./NINE_ROUTER.md) consults when resolving a `ModelBinding` for a given role. Unlike the Router, which handles discovery, assignment, and fallback-chain management, the Policy encapsulates the *logic* of model selection: given a role and a runtime context, it returns the single best model.

Selection follows a three-stage cascade: **filter by must-have capabilities → score by nice-to-have capabilities → sort by preference ordering → pick top**. The Policy is deterministic: identical inputs always produce identical outputs, enabling reproducible runs and testable fallback behaviour.

The Policy is stateless. It reads its rules from a versioned `Policy` document stored in the [Knowledge System](./KNOWLEDGE_SYSTEM.md) and projected through the [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md). Every `choose()` call fetches the latest policy snapshot for the active scope.

## Policy Schema

```
Policy {
  role:                        string          # canonical role name
  must_have_capabilities[]:    Capability[]    # required — filter stage
  nice_to_have_capabilities[]: Capability[]    # scored — ranking stage
  prefer_ordering:             ("latency" | "cost" | "capability_score")[]
  fallback_count?:             int             # default 2
  scope:                       "workspace" | "project" | "group"
  scope_id?:                   string
  version:                     semver
}
```

Capabilities are drawn from the [Model Discovery](./MODEL_DISCOVERY.md#capability-matrix) canonical set:

```
Capability = "reasoning_high" | "reasoning_medium" | "reasoning_low"
           | "context_200k+" | "context_128k" | "context_32k" | "context_8k"
           | "tools" | "tool_use_complex" | "json_mode" | "structured_output"
           | "vision" | "audio" | "streaming" | "code_generation"
           | "code_review" | "instruction_following" | "embedding"
           | "function_calling" | "parallel_tool_calls" | "fast" | "cheap"
```

## Capability Requirements Per Role

Each of the nine canonical roles declares a fixed set of capability requirements. These are the workspace defaults; projects MAY override via the Main KB.

| Role | Must-Have Capabilities | Nice-to-Have Capabilities | Preference Order |
|------|------------------------|---------------------------|------------------|
| **Kernel** | `reasoning_high`, `context_128k`, `tools`, `json_mode` | `context_200k+`, `streaming`, `parallel_tool_calls` | capability, latency, cost |
| **Planner** | `instruction_following`, `structured_output`, `reasoning_medium` | `context_128k`, `tools`, `json_mode` | capability, cost, latency |
| **Router** | `fast`, `cheap`, `json_mode`, `tools` | `reasoning_low`, `streaming` | latency, cost, capability |
| **Researcher** | `tools`, `context_128k`, `tool_use_complex` | `context_200k+`, `reasoning_medium`, `vision` | capability, latency, cost |
| **Builder** | `code_generation`, `tools`, `context_32k` | `context_128k`, `reasoning_high`, `parallel_tool_calls`, `streaming` | capability, latency, cost |
| **Critic** | `reasoning_high`, `code_review`, `json_mode`, `instruction_following` | `context_128k`, `structured_output` | capability, latency, cost |
| **Merger** | `structured_output`, `json_mode`, `tools`, `code_review` | `context_32k`, `reasoning_medium` | capability, cost, latency |
| **Guardian** | `rule_following`, `structured_output`, `json_mode`, `fast` | `instruction_following`, `reasoning_low` | latency, capability, cost |
| **Voice** | `audio`, `streaming`, `fast` | `tools`, `json_mode` | latency, capability, cost |

## Model Selection Algorithm

```
function choose(role, ctx) -> ModelBinding:
  1. policy = resolve_policy(role, ctx.scope)   // scope-aware lookup
  2. candidates = router.list(ctx.filter)         // all reachable models

  3. // Stage 1: Filter by must-have capabilities
  4. filtered = candidates.filter(m =>
  5.   policy.must_have_capabilities.all(cap => m.capabilities.contains(cap))
  6. )
  7. if filtered.empty(): raise NoCandidateSatisfiesMustHaves

  8. // Stage 2: Score by nice-to-have capabilities
  9. scored = filtered.map(m => {
 10.   score = policy.nice_to_have_capabilities
 11.     .count(cap => m.capabilities.contains(cap))
 12.   return { model: m, score }
 13. })

 14. // Stage 3: Sort by preference ordering
 15. sorted = scored.sort((a, b) => {
 16.   for axis in policy.prefer_ordering:
 17.     cmp = compare_by_axis(a.model, b.model, axis)
 18.     if cmp != 0: return cmp
 19.   return a.score - b.score   // tie-break by capability score
 20. })

 21. // Stage 4: Pick top + build fallback chain
 22. primary = sorted[0].model
 23. fallbacks = sorted[1..policy.fallback_count].map(s => s.model)
 24.
 25. return ModelBinding {
 26.   role, primary, fallbacks,
 27.   scope: policy.scope,
 28.   snapshot_ts: now(),
 29.   correlation_id: ctx.correlation_id
 30. }
```

## Fallback Chain Construction

The Policy constructs an ordered fallback chain for every role. The chain is consumed by [Dynamic Workers](./DYNAMIC_WORKERS.md) during execution when the primary model returns a provider error.

```
FallbackChain {
  primary:   Model
  secondary: Model    // tried when primary returns ProviderError
  tertiary:  Model    // tried when primary + secondary both fail
}

// If the policy has fewer candidates than fallback_count:
// - secondary = primary (retry with backoff)
// - tertiary  = first model that passed the must-have filter
// If no model passes must-haves, the chain is empty and the
// Kernel MUST refuse to start the run.
```

## Per-Project Overrides

Projects MAY override the workspace-level policy for any role via the [Main KB](./knowledge-bases/MAIN_KB.md). The override document follows the same `Policy` schema and is projected into the SCE when a project context is active.

```
// Example override in Main KB:
policy_override:
  role: "Builder"
  must_have_capabilities: ["code_generation", "tools", "context_128k"]
  nice_to_have_capabilities: ["reasoning_high", "streaming"]
  prefer_ordering: ["capability", "latency"]
  scope: "project"
  scope_id: "my-monorepo"
```

## Policy Resolution

The resolution order follows the same cascade as the Nine Router's role assignment:

1. **Workspace default** — the canonical requirement table above
2. **Project override** — read from `project_kb.policy_overrides[role]`
3. **Group override** — read from `group_kb.policy_overrides[role]`

A `policy.choose()` call with a group context resolves group → project → workspace. A call with only a project context resolves project → workspace. A call with neither resolves to the workspace default.

## Architecture

```mermaid
flowchart TB
  KERNEL[Main AI Kernel] -->|choose(role, ctx)| POLICY[Model Routing Policy]
  POLICY -->|resolve scope| SCE[(SCE: policy.registry)]
  SCE -->|workspace default| WKB[(Knowledge System\nworkspace KB)]
  SCE -->|project override| PKB[(Main KB\nproject overrides)]
  SCE -->|group override| GKB[(Group KB\ngroup overrides)]
  POLICY -->|list candidates| ROUTER[Nine Router]
  ROUTER -->|Model[]| POLICY
  POLICY -->|filter| F1[Stage 1: Must-have filter]
  F1 -->|score| F2[Stage 2: Nice-to-have score]
  F2 -->|sort| F3[Stage 3: Preference sort]
  F3 -->|pick| F4[Stage 4: Select primary + fallbacks]
  F4 -->|ModelBinding| KERNEL
  POLICY -->|emit| SCE2[(SCE: policy.choices)]
  SCE2 --> COST[Cost Management]
  SCE2 --> OBS[Observability]
```

## Interfaces

```
policy.choose(role: NineRole, ctx: PolicyContext) → ModelBinding
policy.list(role: NineRole, scope?: Scope) → Policy
policy.set(role: NineRole, policy: Policy, scope?: Scope)
policy.resolve(run_ctx: RunContext) → { [role]: ModelBinding }
```

All interfaces follow [Agent Communication](./AGENT_COMMUNICATION.md) and [API Spec](./API_SPEC.md).

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| No candidate satisfies must-haves | Filter returns empty set | `NoCandidateSatisfiesMustHaves` error; Kernel refuses run; operator alerted via SCE |
| Role has no policy defined | Policy registry miss | Fall back to workspace default for that role; emit `policy.missing_default` warning |
| Capability mismatch (model advertises but cannot deliver) | Reported by Critic or worker error | Adjust model's reported capabilities; re-trigger discovery; emit `policy.capability_mismatch` |
| Policy document parse error | Validation failure on `set()` | Reject the update; keep previous policy; surface parse error |
| Circular override resolution | Depth > 3 | Terminate resolution; use workspace default; emit `policy.circular_override` alert |

## Security Considerations

- Policy documents are parsed from validated YAML; never `eval`'d.
- Override scope is checked against the caller's authorization boundary (see [AuthZ/RBAC](./AUTHZ_RBAC.md)).
- Policy snapshots are immutable once used in a `ModelBinding`; a malicious late mutation cannot affect an in-flight run.
- See [Security Model](./SECURITY_MODEL.md).

## Observability

| Metric | Labels | Description |
|--------|--------|-------------|
| `policy_choose_total` | `role`, `ok` | Policy selection calls |
| `policy_choose_seconds` | `role` | Selection latency histogram |
| `policy_override_total` | `role`, `scope` | Override resolution count |
| `policy_no_candidate_total` | `role` | Empty candidate set events |
| `policy_fallback_chain_length` | `role` | Histogram of chain lengths |

Traces: one span per `choose()` with child spans for filter, score, and sort phases. See [Tracing](./TRACING.md).

## Acceptance Criteria

- Calling `policy.choose("Builder", ctx)` with an empty model catalog raises `NoCandidateSatisfiesMustHaves`.
- A model with `capabilities: ["code_generation", "tools", "context_32k"]` scores higher for the Builder role than one missing `code_generation`.
- A project override that changes `must_have_capabilities` for the Kernel role to `["reasoning_high", "context_200k+"]` narrows the candidate set accordingly.
- The resolved fallback chain always has exactly three entries (primary, secondary, tertiary) even when the catalog has fewer models.

## Open Questions

- Whether `prefer_ordering` should support weighted scoring instead of cascade sort — tracked in [templates/ADR](../templates/ADR.md).
- Whether per-role capability requirements should be configurable at the workspace level or remain hard-coded.

## Related Documents

- [Nine Router](./NINE_ROUTER.md) — model discovery, role assignment, fallback-chain storage
- [Model Providers](./MODEL_PROVIDERS.md) — per-provider integration details
- [Model Discovery](./MODEL_DISCOVERY.md) — capability matrix and canonical schema
- [Cost Management](./COST_MANAGEMENT.md) — cost-aware sorting in preference ordering
- [Dynamic Workers](./DYNAMIC_WORKERS.md) — consumers of `ModelBinding` with fallback chain
- [AI Groups](./AI_GROUPS.md) — group-scoped policy overrides
- [Main AI Kernel](./MAIN_AI_KERNEL.md) — calls `policy.choose()` during the route stage
- [Knowledge System](./KNOWLEDGE_SYSTEM.md) — policy document storage
