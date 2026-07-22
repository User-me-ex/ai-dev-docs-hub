# Token Budgeting

> How token budgets are calculated, allocated, tracked, and enforced per run and per worker in AI Dev OS. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

Token Budgeting governs the total token consumption of every run from the moment the Kernel accepts a goal to the moment the final artifact is delivered. Budgets are calculated at intake based on the model routing, task complexity, and user-specified limits. They are enforced at three levels: the per-worker level (soft stop), the per-GroupRun level (aggregation), and the per-run level (hard cap).

The budget system is tightly integrated with [Cost Management](./COST_MANAGEMENT.md) (which translates token consumption into USD) and [Context Window Management](./CONTEXT_WINDOW_MANAGEMENT.md) (which ensures each model call fits within the window). A budget is not a context window — the context window is a per-call limit, while the budget is a cumulative limit across all calls in a run.

## Goals

- Transparent accounting: every token spent is traceable to a specific model call, tool invocation, or system operation.
- Multi-dimensional enforcement: tokens, wall-clock time, and cost are enforced independently. Exhausting any dimension stops the run.
- Partial results on exhaustion: when a budget is exhausted, the worker delivers everything completed so far — no work is lost.
- Predictability: given the same goal, group spec, and model, the token cost of a run MUST be within 20% of the historical median (modulo model output variance).

## Non-Goals

- Budgeting for external API calls (GitHub, Jira, Slack) — those are tracked by [Cost Management](./COST_MANAGEMENT.md) separately.
- Preventing all overspend — edge cases like an unexpectedly long model response may push slightly over budget; the system absorbs small overages (≤ 5% of tokens_max) before enforcing a hard stop.
- Implementation code — this repository is documentation-only (see [AI Coding Rules](./AI_CODING_RULES.md)).

## Budget Calculation

The total budget for a run is calculated at intake:

```
budget = {
  tokens_max: tokens_max      # total input + output tokens across all workers in the run
  wall_ms_max: wall_ms_max    # wall-clock duration for the entire run
  usd_max: usd_max            # maximum cost in USD
}
```

Sources (in priority order):
1. **Explicit**: the `budget` field in `POST /v1/runs` (user-specified overrides).
2. **Group defaults**: from `GroupSpec.budget`.
3. **System defaults**: from `config.budget.defaults`.
4. **Estimated**: if none of the above specify a value, the Kernel estimates based on `ModelRouting.complexity_score` and historical averages.

### Input Token Contribution

```
input_tokens = system_prompt_tokens
             + task_context_tokens
             + conversation_history_tokens
             + kb_excerpt_tokens
             + tool_result_tokens (fed back as user messages)
```

### Output Token Contribution

```
output_tokens = sum(model_response_tokens) for all model calls in the run
```

### Tool Token Contribution

Tool calls consume tokens through their arguments (sent to the model) and their results (fed back as context). Each tool invocation and its result are counted in the input token total of the next model call.

## Per-Model Token Costs

Pricing per million tokens (fetched from [Model Discovery](./MODEL_DISCOVERY.md) cache, refreshed hourly):

| Model ID | Input per 1M | Output per 1M | Cached per 1M |
|----------|-------------|--------------|---------------|
| openai/gpt-4o | $2.50 | $10.00 | $1.25 |
| openai/gpt-4o-mini | $0.15 | $0.60 | $0.075 |
| openai/o1 | $15.00 | $60.00 | $7.50 |
| openai/o3-mini | $1.10 | $4.40 | $0.55 |
| anthropic/claude-3-5-sonnet | $3.00 | $15.00 | — |
| anthropic/claude-3-opus | $15.00 | $75.00 | — |
| anthropic/claude-3-haiku | $0.25 | $1.25 | — |
| google/gemini-2.0-flash | $0.10 | $0.40 | $0.025 |
| google/gemini-2.5-pro | $1.25 | $5.00 | $0.3125 |
| mistral/mistral-large | $2.00 | $6.00 | — |
| ollama (any) | $0.00 | $0.00 | — |

Costs are calculated as: `(input_tokens / 1_000_000 * input_price) + (output_tokens / 1_000_000 * output_price)`. Cached input pricing applies when `context.cache` is used (OpenAI Prompt Caching, Anthropic Prompt Caching).

## Budget Allocation

```
BudgetAllocation {
  per_run:    { tokens_max, wall_ms_max, usd_max },
  per_group:  { tokens_max, wall_ms_max, usd_max },    // optional: divides run budget across groups
  per_worker: { tokens_max, wall_ms_max, usd_max }     // optional: divides group budget across workers
}
```

### Default Allocation Strategy

```
budget_per_worker = run_budget / (num_tasks * num_retries_default)
```

This provides each task with an equal slice of the run's budget, with one retry's worth of reserve. The remainder (up to 10% of total) is reserved for the Kernel's coordination overhead.

### Overcommit Protection

The sum of all `per_group` budgets MUST NOT exceed the `per_run` budget. The sum of all `per_worker` budgets within a group MUST NOT exceed the group's allocation. Violations are rejected at intake with `UNPROCESSABLE { reason: "budget_overcommit" }`.

## Budget Tracking During Execution

The `BudgetTracker` runs inside each worker and publishes updates on every token event:

```
BudgetSnapshot {
  worker_id:      ulid
  run_id:         ulid
  ts:             rfc3339
  tokens_spent:   { input, output, total }
  wall_ms_spent:  number
  usd_spent:      number
  tokens_remaining: number   // tokens_max - total_tokens_spent
  wall_ms_remaining: number
  usd_remaining:   number
  pct_exhausted:   number   // 0.0–1.0
}
```

Updates are published as `budget.tick` events on the SCE and included in every `worker.tick` heartbeat.

## Enforcement

### Soft Warning (80% exhausted)

When any dimension of the budget reaches 80% of its limit:

```
emit budget.warning {
  dimension: "tokens" | "wall_ms" | "usd",
  spent, limit, pct: 0.80,
  worker_id, run_id
}
```

The worker continues executing. The Kernel logs the warning and may trigger pre-emptive compression to reduce token consumption.

### Hard Stop (100% exhausted)

When any dimension reaches 100%:

```
emit budget.exhausted {
  dimension,
  spent, limit,
  worker_id, run_id
}
```

The worker's model stream is cancelled immediately. In-flight tool calls are allowed to complete (up to `tool_timeout_ms`). The worker transitions to `Completing` state and delivers partial results.

Multiple dimensions may exhaust simultaneously — the BudgetTracker reports all exhausted dimensions in a single `budget.exhausted` event.

## Partial Artifact Delivery on Exhaustion

When a budget is exhausted, the worker MUST deliver all completed work:

```
PartialArtifact {
  run_id,
  worker_id,
  budget_exhausted: { dimensions: ["tokens"], spent: 95000, limit: 100000 },
  completed_tasks: [TaskResult, ...],
  in_progress_task: {
    task_id,
    partial_output: string,
    completed_tool_calls: [ToolCall, ...],
    interrupted_tool_call: { name, args, partial_result? }
  },
  checkpoint_id: string  // last checkpoint before interruption
}
```

The partial artifact is published as a `run.partial_delivery` event and stored in [Persistent Memory](./PERSISTENT_MEMORY.md). The user can replay from the checkpoint with an increased budget.

## Interfaces

| Function | Signature | Description |
|----------|-----------|-------------|
| `budget.allocate` | `(run_spec: RunSpec) => BudgetAllocation` | Calculate budget allocation for a run, group, and workers. |
| `budget.spend` | `(worker_id: ulid, tokens: TokenUsage, cost: Cost) => BudgetSnapshot` | Record token/cost spend for a model call. Returns the updated snapshot. |
| `budget.remaining` | `(worker_id: ulid) => BudgetSnapshot` | Get the current remaining budget for a worker or run. |
| `budget.reset` | `(run_id: ulid) => BudgetAllocation` | Reset budget tracking (used on replay with increased budget). |

All budget functions emit events on the SCE topic `budget.<run_id>`.

## Integration with Cost Management

Token Budgeting tracks token counts and translates them to cost using per-model pricing. [Cost Management](./COST_MANAGEMENT.md) aggregates these costs across runs, time periods, and projects, adding external API costs (GitHub Actions, MCP tool calls, plugin invocations) that Token Budgeting does not track.

The integration point is the `budget.spend` event: when `budget.spend` is called, it emits a structured event that Cost Management subscribes to for aggregation.

## Acceptance Criteria

- A run with `tokens_max: 50000` that reaches 50,000 tokens MUST stop and deliver a partial artifact within 500 ms of the last token.
- A `budget.warning` event MUST be emitted when any dimension reaches 80% of its limit, with no false positives at 79.9%.
- Multiple dimensions exhausting simultaneously in a single model call MUST produce a single `budget.exhausted` event with all exhausted dimensions.
- The sum of per-worker budgets MUST NOT exceed the per-run budget; an attempt to allocate overcommit MUST be rejected.
- A budget-exhausted worker's partial artifact MUST include the last checkpoint ID so the user can replay.

## Open Questions

- Whether to support budget transfers between dimensions (e.g., unused wall-clock time converted to additional token budget) — tracked in [templates/ADR](../templates/ADR.md).
- Whether per-model pricing should be user-configurable (for enterprise negotiated rates) or always sourced from [Model Discovery](./MODEL_DISCOVERY.md).

## Related Documents

- [Agent Lifecycle](./AGENT_LIFECYCLE.md) — budget enforcement in the worker state machine
- [Cost Management](./COST_MANAGEMENT.md) — USD aggregation across runs, projects, and time periods
- [Context Window Management](./CONTEXT_WINDOW_MANAGEMENT.md) — per-call token limits and compression
- [Model Providers](./MODEL_PROVIDERS.md) — model-specific pricing and configuration
- [Model Discovery](./MODEL_DISCOVERY.md) — how model metadata and pricing are discovered
- [System Overview](./SYSTEM_OVERVIEW.md)
- [Main AI Kernel](./MAIN_AI_KERNEL.md)
