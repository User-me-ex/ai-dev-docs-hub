# Worker Lifecycle Sequence

> Sequence diagram of the Dynamic Worker lifecycle from spawn to retirement.

```mermaid
sequenceDiagram
    participant TS as Task Scheduler
    participant WS as Worker Scheduler
    participant AGS as AI Group System
    participant DW as Dynamic Worker
    participant SCE as Shared Context Engine
    participant MP as Model Provider
    participant TOOL as External Tool

    TS->>WS: acquire(task)
    WS->>AGS: spawn_worker(group_spec, model_binding)

    Note over DW: WARMING state
    AGS->>DW: init(group_spec, model_binding)
    DW->>DW: load model client, tool bindings
    DW-->>AGS: worker_ready(worker_id)
    AGS-->>WS: Worker
    WS-->>TS: Worker acquired

    Note over DW: RUNNING state
    TS->>DW: execute(task)
    DW->>SCE: publish("worker.task.started", {task_id})
    DW->>DW: inject Master Prompt + Role Prompt + Task

    loop tool calls and model responses
        DW->>MP: model.invoke(prompt)
        MP-->>DW: response{tokens, finish_reason}
        DW->>DW: parse response, extract tool calls

        alt Tool call requested
            DW->>TOOL: tool.call(name, args)
            TOOL-->>DW: result
            DW->>SCE: publish("worker.tool.result", {tool_name, ok, duration_ms})
        end

        opt Every CHECKPOINT_INTERVAL_MS
            DW->>SCE: publish("worker.checkpoint", {task_id, state_snapshot})
        end

        opt Preemption requested
            SCE-->>DW: preemption_request
            DW->>SCE: publish("worker.checkpoint", {task_id, state_snapshot, preempted: true})
            DW-->>TS: yield(worker_id)
            Note over TS,DW: Task rescheduled, worker becomes IDLE
        end
    end

    Note over DW: COMPLETED state
    DW-->>TS: result{ok: true, artifact}
    DW->>SCE: publish("worker.task.completed", {task_id, duration_ms, tokens_used})

    Note over DW: RETURNING → IDLE
    WS->>DW: release()
    alt Pool below min_warm
        WS->>DW: enter IDLE pool
    else Pool at capacity
        WS->>DW: retire()
        DW-->>WS: destroyed
    end
```

## Related Documents

- [Dynamic Workers](../docs/DYNAMIC_WORKERS.md) — worker execution and state machine
- [Worker Scheduler](../docs/WORKER_SCHEDULER.md) — worker pool management
- [Task Scheduler](../docs/TASK_SCHEDULER.md) — task dispatch
- [AI Group System](../docs/AI_GROUP_SYSTEM.md) — worker pool ownership
- [Agent Lifecycle](../docs/AGENT_LIFECYCLE.md) — worker lifecycle and checkpointing
