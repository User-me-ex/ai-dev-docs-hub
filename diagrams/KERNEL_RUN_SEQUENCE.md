# Kernel Run Sequence

> Full sequence diagram of a Kernel run from goal submission to result delivery.

```mermaid
sequenceDiagram
    participant User as User / CLI
    participant K as Main AI Kernel
    participant SCE as Shared Context Engine
    participant PE as Planning Engine
    participant NR as Nine Router
    participant AGS as AI Group System
    participant DW as Dynamic Worker
    participant CR as Critic
    participant MM as Merge Manager
    participant AG as Architecture Guardian

    User->>K: kernel.submit(goal)
    K->>SCE: publish("run.submitted", {run_id, goal})
    Note over K: Stage 1: Intake
    K->>K: parse goal, authenticate, allocate budget
    K->>SCE: publish("run.intake_complete", {run_id, run_spec})

    Note over K: Stage 2: Plan
    K->>PE: decompose(goal)
    PE->>SCE: publish("plan.started", {run_id})
    PE->>PE: generate TaskGraph
    PE->>SCE: publish("plan.complete", {run_id, task_graph})
    SCE-->>K: task_graph

    Note over K: Stage 3: Route
    K->>NR: binding(role, task)
    NR->>NR: resolve model binding
    NR-->>K: ModelBinding{model, fallbacks}

    Note over K: Stage 4: Execute
    K->>AGS: spawn_worker(task, binding)
    AGS->>DW: execute(task)
    DW->>SCE: publish("worker.task.started", {task_id})
    DW->>DW: invoke model, call tools
    DW->>SCE: publish("worker.task.progress", {task_id, tokens, events})
    DW-->>AGS: artifact
    AGS-->>K: artifact

    Note over K: Stage 5: Critique
    K->>CR: review(artifact)
    CR->>SCE: publish("critique.started", {artifact_id})
    CR-->>K: Verdict{ok, violations?}
    alt Verdict = reject
        CR->>SCE: publish("critique.rejected", {artifact_id, reason})
        K->>PE: replan(reason)
        PE-->>K: revised tasks
    else Verdict = accept
        CR->>SCE: publish("critique.accepted", {artifact_id})
    end

    Note over K: Stage 6: Merge
    K->>MM: merge(artifacts)
    MM->>SCE: publish("merge.started", {artifact_ids})
    MM->>MM: three-way merge
    alt Clean merge
        MM-->>K: MergedArtifact
    else Conflict
        MM-->>K: conflict_escalation
        K->>SCE: publish("merge.conflict", {details})
    end

    Note over K: Stage 7: Guard
    K->>AG: check(MergedArtifact)
    AG->>AG: evaluate rules (parallel)
    AG->>SCE: publish("guardian.evaluating", {artifact_id, rule_count})
    alt All rules pass
        AG-->>K: Verdict{ok: true}
    else Violation detected
        AG-->>K: Verdict{ok: false, violations}
        K->>PE: replan(violations)
        PE-->>K: revised tasks
    end

    Note over K: Stage 8: Deliver
    K-->>User: result
    K->>SCE: publish("run.completed", {run_id, result})
```

## Related Documents

- [Main AI Kernel](../docs/MAIN_AI_KERNEL.md) — stage contracts and kernel loop
- [Shared Context Engine](../docs/SHARED_CONTEXT_ENGINE.md) — SCE event topics
- [Planning Engine](../docs/PLANNING_ENGINE.md)
- [Nine Router](../docs/NINE_ROUTER.md)
- [Dynamic Workers](../docs/DYNAMIC_WORKERS.md)
- [Merge Manager](../docs/MERGE_MANAGER.md)
- [Architecture Guardian](../docs/ARCHITECTURE_GUARDIAN.md)
