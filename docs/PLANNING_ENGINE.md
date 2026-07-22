# Planning Engine

> Turns goals into task graphs through goal decomposition, task graph generation, and dependency resolution. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

The Planning Engine is the second stage in the [Main AI Kernel](./MAIN_AI_KERNEL.md) loop (intake → **plan** → route → execute → critique → merge → guard → deliver). It receives an analysed goal from the Intake stage and produces a `TaskGraph` — a directed acyclic graph of tasks that, when executed in topological order, delivers the goal.

The Engine follows a **minimum viable plan** philosophy: it decomposes only as far as necessary to parallelise independent work and satisfy the goal's acceptance criteria. It does not overspecify implementation details — those are delegated to the executing agents at runtime.

Every plan is versioned and immutable once produced. If the [Critic](./CRITIC.md) rejects an output or the [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) vetoes a change, the Kernel calls `replan()` with the rejection feedback. The Engine produces a new plan that addresses the feedback while preserving as much of the original structure as possible.

## Goal Analysis

The Intake stage produces a structured goal that the Planning Engine consumes:

```
Goal {
  id:                    ulid
  primary_deliverable:   string        # what must be produced
  scope:                 Scope         # in-scope / out-of-scope boundaries
  constraints:           Constraint[]  # hard constraints (budget, time, tech)
  context:               ContextRef[]  # references to relevant KB documents
  acceptance_criteria:   Criterion[]   # testable pass/fail conditions
  correlation_id:        uuid
}
```

Every goal MUST have at least one acceptance criterion. Goals without criteria are rejected at intake.

## Task Decomposition Principles

The Engine applies the following principles when decomposing a goal into tasks:

1. **Minimum viable plan** — produce the smallest DAG that can deliver the goal; avoid over-decomposition.
2. **Parallelise independent tasks** — tasks with no dependency chain MUST run concurrently. Sibling tasks with no data dependency MUST NOT be serialised.
3. **One group per task** — each task is assigned to exactly one [AI Group](./AI_GROUPS.md). Cross-group data flows are expressed as edges.
4. **Data dependencies as edges** — if task B needs task A's output, the edge `A → B` carries the output schema. Implicit ordering (e.g. "run A before B") MUST have a data dependency to justify it.
5. **Budgets are mandatory** — every task declares token, wall-time, and cost budgets. The Engine calculates these from historical data via [Cost Management](./COST_MANAGEMENT.md).

## TaskGraph Schema

```
TaskGraph {
  tasks:         Task[]
  dependencies:  Dependency[]    // edges: { from, to, type, data_schema? }
  metadata: {
    goal_id:     ulid
    run_id:      ulid
    created_at:  rfc3339
    version:     int             // incremented on each replan
    feedback?:   FeedbackRef     // set on replan; references prior rejection
  }
}
```

## TaskNode Schema

```
Task {
  id:          ulid
  group_id:    GroupId           // AI Group responsible for execution
  role:        NineRole           // agent role (Builder, Researcher, etc.)
  description: string             // natural-language task specification
  inputs:      DataSlot[]         // { name, schema, source_task_id? }
  outputs:     DataSlot[]         // { name, schema }
  budget: {
    max_tokens:     int
    max_wall_time:  duration
    max_cost:       decimal       // in USD or token-equivalent
  }
  deps:        TaskId[]           // task IDs that must complete before this one
  state:       TaskState          // set by executor; "pending" at plan time
  artifacts:   ArtifactRef[]      // populated at execution time
}
```

## Planning Algorithm

```
function plan(goal, ctx) -> TaskGraph:
  1.  // Analyse the goal
  2.  validated = validate_goal(goal)
  3.  if not validated.ok: raise InvalidGoal(validated.errors)

  4.  // Identify required groups
  5.  groups = identify_groups(goal.primary_deliverable, goal.context)
  6.  // e.g. ["builder", "researcher", "critic"]

  7.  // Decompose goal into tasks, one per group
  8.  tasks = []
  9.  for group in groups:
 10.    task = decompose(group, goal)
 11.    tasks.append(task)

 12.  // Resolve dependencies between tasks
 13.  dependencies = resolve_dependencies(tasks, goal)
 14.  // e.g. researcher.output → builder.input

 15.  // Assign budgets
 16.  tasks = assign_budgets(tasks, ctx.historical_costs)

 17.  // Validate the graph
 18.  graph = TaskGraph { tasks, dependencies, metadata }
 19.  validation = validate(graph)
 20.  if not validation.ok: raise InvalidPlan(validation.errors)

 21.  // Emit and return
 22.  SCE.emit("planning.plan_ready", graph)
 23.  return graph
```

## Replan Protocol

When the Critic rejects a task's output or the Guardian vetoes a merged artifact, the Kernel calls `replan()`:

```
function replan(run_id, feedback) -> TaskGraph:
  1. original = load_graph(run_id)
  2. if original.metadata.version >= 5: raise MaxReplansExceeded

  3. // Incorporate feedback
  4. if feedback.type == "critic_rejection":
  5.   target_task = find_task(feedback.task_id)
  6.   target_task.description = augment(target_task.description, feedback.detail)
  7.   target_task.budget = increase_budget(target_task.budget, 1.5)

  8. if feedback.type == "guardian_veto":
  9.   add_constraint(feedback.rule_id, feedback.violation)

 10. // Re-resolve dependencies (may have changed)
 11. dependencies = resolve_dependencies(tasks, original.goal)

 12. // Bump version
 13. graph = TaskGraph { tasks, dependencies, metadata: {
 14.   ...original.metadata,
 15.   version: original.metadata.version + 1,
 16.   feedback: FeedbackRef { run_id, type, detail }
 17. }}

 18. validation = validate(graph)
 19. return graph
```

**Max replans:** 5. After the 5th replan, the Kernel marks the run as `failed` and escalates to the operator.

## Architecture

```mermaid
flowchart TB
  INTAKE[Intake Stage] -->|Goal| PLAN[Planning Engine]
  PLAN -->|analyse| ANALYSIS[Goal Analysis]
  ANALYSIS -->|validated goal| DECOMP[Task Decomposition]
  DECOMP -->|tasks[]| DEPS[Dependency Resolution]
  DEPS -->|dependencies[]| BUDGET[Budget Assignment]
  BUDGET -->|tasks[] + budgets| VALIDATE[Graph Validation]
  VALIDATE -->|TaskGraph| ROUTER[Route Stage]

  CRITIC[Critic] -->|rejection| KERNEL[Main AI Kernel]
  GUARD[Architecture Guardian] -->|veto| KERNEL
  KERNEL -->|replan(run_id, feedback)| PLAN

  PLAN -->|emit| SCE[(SCE: planning.*)]
  SCE --> COST[Cost Management]
  SCE --> OBS[Observability]
  SCE --> AUDIT[Audit Log]
```

## Interfaces

```
plan(goal: Goal, ctx: PlanningContext) → TaskGraph
replan(run_id: ulid, feedback: Feedback) → TaskGraph
plan.validate(graph: TaskGraph) → ValidationResult
plan.explain(graph_id: ulid) → PlanExplanation   // human-readable plan summary
plan.diff(from_graph_id: ulid, to_graph_id: ulid) → PlanDiff
```

`ValidationResult` shape:

```
ValidationResult {
  ok:       boolean
  errors:   ValidationError[]
  warnings: string[]
  stats: {
    task_count:      int
    parallel_depth: int     // longest chain of serial dependencies
    total_budget:    Budget // sum of all task budgets
  }
}
```

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| Cycle detected | Graph contains a directed cycle | Reject plan; surface cycle edges to operator; suggest breaking the cycle by merging tasks |
| Unreachable task | Task has no inbound path from any root | Remove task or add missing dependency; emit `planning.unreachable_task` warning |
| Max replans exceeded | `version >= 5` | Mark run `failed`; escalate to operator with full feedback history |
| Invalid goal (no criteria) | `goal.acceptance_criteria` empty | Return `InvalidGoal`; refuse to plan |
| Budget exceeds limit | `total_budget > ctx.max_run_budget` | Return `BudgetExceeded`; suggest reducing scope or increasing limit |
| Unresolvable dependency | No model satisfies the required role's must-haves | Return `PlanBlocked`; surface the missing capability to operator |
| Group not found | `identify_groups()` returns empty | Return `NoGroupForGoal`; check AI Group configuration |

## Security Considerations

- Plan data (goal, context, feedback) is treated as sensitive; all reads require authorisation (see [AuthZ/RBAC](./AUTHZ_RBAC.md)).
- Replan feedback includes Critic/Guardian verdicts which are signed; the Engine MUST verify signatures before accepting feedback.
- Budget caps are enforced at the Engine level, not delegated to executors, to prevent budget-busting replan loops.
- See [Security Model](./SECURITY_MODEL.md).

## Observability

| Metric | Labels | Description |
|--------|--------|-------------|
| `planning_plan_total` | `ok` | Plan generation attempts |
| `planning_replan_total` | `version`, `feedback_type` | Replan events |
| `planning_plan_seconds` | — | Plan generation latency histogram |
| `planning_task_count` | — | Task count per plan |
| `planning_parallel_depth` | — | Longest dependency chain |
| `planning_cycle_detected_total` | — | Cycle detection events |
| `planning_max_replan_exceeded_total` | — | Runs terminated after 5 replans |

Traces: one span per `plan()` with child spans for each decomposition step; one span per `replan()`. See [Tracing](./TRACING.md).

## Acceptance Criteria

- A goal "Add a search bar to the React app" with acceptance criteria produces a `TaskGraph` with at least 3 tasks (Researcher → Builder → Critic).
- A cycle between tasks A → B → C → A is detected and the plan is rejected with `errors` containing the cycle edges.
- Calling `replan()` with `critic_rejection` feedback increments the plan version and preserves all non-rejected tasks.
- The 6th call to `replan()` for the same run raises `MaxReplansExceeded`.

## Related Documents

- [Task Graph](./TASK_GRAPH.md) — the DAG structure the Engine produces
- [Main AI Kernel](./MAIN_AI_KERNEL.md) — calls plan() and replan() in the loop
- [Multi-Agent Orchestration](./MULTI_AGENT_ORCHESTRATION.md) — executor that runs planned tasks
- [AI Groups](./AI_GROUPS.md) — groups assigned to tasks
- [Cost Management](./COST_MANAGEMENT.md) — budget calculation from historical data
- [Critic](./CRITIC.md) — rejection feedback source
- [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) — veto feedback source
- [System Overview](./SYSTEM_OVERVIEW.md)
