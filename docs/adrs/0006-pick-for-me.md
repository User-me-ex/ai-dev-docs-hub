# ADR-0006: "Pick for Me" Implementation Approach

## Status

Proposed

## Context

The [Nine Router](../NINE_ROUTER.md) proposes a "Pick for Me" smart-assign feature that recommends a model for a role based on the role description and available models. The question is the implementation approach: a pure rule engine (capability-based filtering + cost/latency scoring) or a model call (the Router asks an LLM to recommend).

## Decision

**Pure rule engine for v1.0; model call evaluated for v2.0.**

The "Pick for Me" feature for v1.0 will use a deterministic rule engine:

1. Read the role's required capabilities from the role definition (e.g. Builder needs `code_generation`, `tool_use`).
2. Filter models by must-have capabilities.
3. Score remaining models by: capability match (70%), cost (20%), latency (10%).
4. Return the top model with an explanation of why it was chosen.

The rule-based approach:
- Is deterministic and auditable: same input always produces same recommendation.
- Does not add cost: no LLM call required.
- Works offline: no cloud dependency.
- Can be overridden by the user without confusion.

A model-call approach would be re-evaluated for v2.0 when:
- The model catalog has grown beyond what rule-based scoring handles well.
- The Router has enough historical data to ground recommendations.
- The cost of the recommendation call is negligible compared to the model use.

## Consequences

**Positive:**
- No additional API cost for recommendations
- Deterministic and testable behaviour
- Works fully offline
- Transparent: users can see why a model was recommended

**Negative:**
- Rule-based scoring is less flexible than an LLM — it cannot reason about nuanced trade-offs (e.g. "this model has worse benchmarks but better instruction-following for this specific project")
- The explanation is generated from a template, not from natural language

## Related

- [Nine Router](../NINE_ROUTER.md) — "Pick for me" feature in Open Questions
- [Model Routing Policy](../MODEL_ROUTING_POLICY.md) — the scoring cascade this feature reuses
