# ADR-0006: "Pick for Me" Implementation Approach

## Status

Proposed

## Context

The [Nine Router](../NINE_ROUTER.md) proposes a "Pick for Me" smart-assign feature that recommends a model for a role based on the role description and available models. The question is the implementation approach: a pure rule engine (capability-based filtering + cost/latency scoring) or a model call (Nine Router asks an LLM to recommend). Because Nine Router is the sole model gateway, the recommendation engine has a complete view of all available models — local and cloud — without any subsystem needing direct provider access.

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

### Rule Engine Algorithm

```
Algorithm: PickForMe (v1.0)
Input:
  - role: RoleDefinition          // Name, description, required capabilities
  - available_models: Model[]     // All models from discovered providers
  - preferences: UserPreferences  // Optional user constraints
Output: RecommendedModel

01  // Step 1: Filter by must-have capabilities
02  candidates = []
03  for model in available_models:
04      if model.capabilities ⊇ role.required_capabilities:
05          if not preferences.exclude_providers.contains(model.provider):
06              candidates.add(model)

07  // Step 2: Check for zero candidates
08  if candidates.isEmpty():
09      // Find models with partial capability match
10      partial_candidates = []
11      for model in available_models:
12          match_pct = |model.capabilities ∩ role.required_capabilities|
13                     / |role.required_capabilities|
14          if match_pct >= 0.5:  // At least 50% capability match
15              partial_candidates.add(model, match_pct)
16
17      if partial_candidates.isEmpty():
18          return {
19              model: null,
20              recommendation: "No model found",
21              alternatives: [],
22              explanation: "No model supports the required capabilities:
23                           {role.required_capabilities}"
24          }
25      else:
26          // Return partial match with warning
27          (best_partial, pct) = partial_candidates.maxBy(match_pct)
28          return recommendWithWarning(best_partial, pct)

29  // Step 3: Apply user preference constraints
30  if preferences.max_cost_per_token != null:
31      candidates = candidates.filter(m =>
32          m.cost_per_token <= preferences.max_cost_per_token)
33  if preferences.min_context_window != null:
34      candidates = candidates.filter(m =>
35          m.context_window >= preferences.min_context_window)
36  if preferences.preferred_providers != null:
37      // Boost, don't filter: preferred providers get a score bonus
38      preferred_providers = preferences.preferred_providers

39  // Step 4: Score candidates
40  for candidate in candidates:
41      candidate.score = computeScore(candidate, role, preferences)

42  // Step 5: Return top recommendation
43  best = candidates.maxBy(score)
44  return {
45      model: best,
46      recommendation: best.name,
47      alternatives: candidates.sortedBy(score).take(5),
48      explanation: generateExplanation(best, candidates)
49  }
```

### Scoring Function Formulas

```
ScoringFunction: computeScore(model, role, preferences)

// Dimension 1: Capability Match Score (weight: 0.70)
// Measures what fraction of required capabilities this model supports
capability_score = |model.capabilities ∩ role.required_capabilities|
                 / |role.required_capabilities|

// Bonus: support for nice-to-have capabilities
nice_to_have_bonus = |model.capabilities ∩ role.nice_to_have_capabilities|
                    / |role.nice_to_have_capabilities| * 0.1  // Max +0.1

capability_match = capability_score + nice_to_have_bonus

// Dimension 2: Cost Score (weight: 0.20)
// Normalized cost: $0 = 1.0, $50/M tokens = 0.0
// Uses both input and output costs
effective_cost = (model.cost_per_input_token * 0.3   // Typical input:output ratio 3:1
                + model.cost_per_output_token * 0.7)
cost_score = 1.0 - min(effective_cost / max_acceptable_cost, 1.0)

// Dimension 3: Latency Score (weight: 0.10)
// Normalized latency: 0ms = 1.0, 10000ms = 0.0
latency_score = 1.0 - min(model.p95_latency_ms / 10000, 1.0)

// Preferred provider bonus (if applicable):
provider_bonus = preferences.preferred_providers.contains(model.provider) ? 0.05 : 0.0

// Final score:
total_score = (0.70 * capability_match)
            + (0.20 * cost_score)
            + (0.10 * latency_score)
            + provider_bonus

// Bounds: total_score ∈ [0, 1.15] (provider bonus can push past 1.0)
// Clamp to [0, 1.0] for display purposes
```

### Model Evaluation Prompt Template (v2.0)

For v2.0, when the Router evaluates using an LLM, it uses a structured prompt:

```
System:
You are a model routing evaluator. Your task is to recommend the best model for a
given role based on the available models and the role's requirements.

You MUST respond with a valid JSON object. Do not include any text outside the JSON.

Available models:
{{models_json}}

User:
Role: {{role_name}}
Description: {{role_description}}
Required capabilities: {{required_capabilities}}
Nice-to-have capabilities: {{nice_to_have_capabilities}}

User preferences:
Max cost per token: {{max_cost}}
Preferred providers: {{preferred_providers}}
Min context window: {{min_context_tokens}}

Consider the following factors:
1. The model MUST support all required capabilities
2. Prefer lower cost when capabilities are equal
3. Prefer lower latency for interactive roles
4. Respect user preferences for providers and cost limits
5. Consider non-obvious trade-offs (e.g., a model with fewer capabilities
   but better instruction-following for the specific role)

Recommend the top 3 models with explanation.

Response format:
{
  "recommendation": {
    "model_name": "...",
    "provider": "...",
    "score": <0.0-1.0>,
    "strengths": ["..."],
    "weaknesses": ["..."]
  },
  "alternatives": [
    { "model_name": "...", "provider": "...", "score": <0.0-1.0>,
      "reason_not_top": "..." }
  ],
  "reasoning": "Step-by-step reasoning for the recommendation"
}
```

### Progressive Enhancement Path: v1.0 → v2.0

```
v1.0: Pure Rule Engine
  └─ Capability filtering + weighted scoring
  └─ Deterministic, offline, no cost
  └─ Template-based explanation
  └─ Complexity: O(n) where n = model count
  │
  │ Collect feedback signals:
  │ - How often does the user accept/reject the recommendation?
  │ - Manual override rate per role
  │ - Post-hoc satisfaction survey ("was this a good recommendation?")
  │
  v
v1.5: Rule Engine + Feedback Learning
  └─ Score weights tuned by historical acceptance rate
  └─ Role-specific weight profiles (Builder roles weight capability higher,
      Reviewer roles weight cost higher)
  └─ Still deterministic given the same weights
  └─ Still no LLM call
  │
  │ Evaluate if v2.0 is needed:
  │ - Override rate > 30% → v2.0 may improve recommendations
  │ - Rule engine produces counter-intuitive results for specific roles
  │ - User requests more nuanced reasoning
  │
  v
v2.0: Hybrid (Rule Engine + Model Call)
  └─ Rule engine narrows to top-10 candidates (reduces LLM context)
  └─ LLM evaluates top-10 with nuanced reasoning prompt
  └─ LLM result weighted against rule engine score (50:50 blend)
  └─ LLM call cost: ~200 tokens per recommendation (~$0.001 at GPT-4o pricing)
  └─ Fallback: if LLM call fails, use pure rule engine result
  │
  │ Monitor:
  │ - Override rate dropped to < 15% → hybrid is effective
  │ - Override rate unchanged → increase LLM weight or improve prompt
  │ - Cost per recommendation > $0.01 → optimize prompt size or cache
  │
  v
v2.5+: Agentic Evaluation
  └─ Router maintains a persistent agent that learns from feedback
  └─ Fine-tuned small model for routing decisions
  └─ Local inference, no cloud dependency
  └─ Complexity: O(1) inference, but training pipeline required
```

### Fallback Behavior

The "Pick for Me" feature degrades gracefully at every level:

```
Level 1: Rule Engine Active (Online)
  └─ All models discovered, capabilities indexed
  └─ Returns recommendation with explanation
  └─ Latency: < 50ms

Level 2: Rule Engine Active (Degraded)
  └─ Some providers unreachable, partial model catalog
  └─ Returns recommendation with warning:
      "Recommendation based on partial model catalog.
       Providers unavailable: [Anthropic, Mistral]"
  └─ Latency: < 50ms

Level 3: Rule Engine Active (No Candidates)
  └─ No models meet the required capabilities
  └─ Returns closest partial match with warning:
      "No model fully supports all required capabilities.
       Closest match: gpt-4o (supports 3/5 capabilities).
       Consider reducing role requirements."
  └─ If even partial matches found:
      "No model in the catalog supports any of the required capabilities.
       Check provider connectivity."

Level 4: Rule Engine Active (Offline/No Catalog)
  └─ No model catalog available (first boot, no network)
  └─ Returns:
      "Model catalog not available. 'Pick for Me' requires at least
       one successful model discovery. Run discovery or assign a model manually."
  └─ User can still assign models manually (manual override always available)

Level 5: v2.0 Hybrid — LLM Call Fails
  └─ Fall back to v1.0 rule engine result
  └─ Log the LLM failure for diagnosis
  └─ Return rule engine result with notice:
      "Enhanced evaluation unavailable. Recommendation based on capability scoring."

Level 6: v2.0 Hybrid — LLM Returns Malformed Response
  └─ Validate JSON structure
  └─ If invalid: retry once with stricter prompt
  └─ If still invalid: fall back to rule engine result
  └─ Log malformed response for prompt improvement
```

## Why rule engine first?

- No additional API cost for recommendations
- Deterministic and testable behaviour
- Works fully offline
- Transparent: users can see why a model was recommended

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

---

## Local-First Amendment

Under the Local-First architecture:

- **"Pick for Me" is a Nine Router feature.** Only Nine Router runs the
  recommendation engine. Agents and subsystems consume the result via Nine
  Router's API — they never perform their own model selection.
- **Local models are first-class citizens.** The rule engine scores Ollama,
  vLLM, and llama.cpp models alongside cloud models using the same capability,
  cost, and latency metrics.
- **Cost scoring is relative.** For local models, "cost" defaults to zero (no
  per-token charge) unless the operator configures a compute budget.
- **Offline by design.** The v1.0 rule engine works without any network
  connectivity. The v2.0 hybrid mode's LLM call also goes through Nine Router,
  ensuring a single audit trail.
- **Scope unchanged.** The rule engine algorithm, scoring formulas, fallback
  behavior, and progressive enhancement path remain as specified above.
