# Worker Lifecycle Sequence

> Sequence diagram of the Dynamic Worker lifecycle from spawn to retirement, including checkpointing, preemption, warm pool integration, and error recovery.

## Full Worker Lifecycle

```mermaid
sequenceDiagram
    participant TS as Task Scheduler
    participant WS as Worker Scheduler
    participant AGS as AI Group System
    participant DW as Dynamic Worker
    participant SCE as Shared Context Engine
    participant MP as Model Provider
    participant TOOL as External Tool
    participant MEM as Persistent Memory

    TS->>WS: acquire(task)
    WS->>WS: check warm pool for matching group_spec
    alt Warm worker available
        WS->>AGS: reuse_worker(worker_id, new_task)
        AGS->>DW: reassign(task, budget)
        DW-->>AGS: worker_ack(worker_id)
    else No warm worker
        WS->>AGS: spawn_worker(group_spec, model_binding)
    end

    Note over DW: INITIALISING state
    AGS->>DW: init(group_spec, model_binding)
    DW->>DW: load model client, tool bindings
    DW->>MEM: allocate_worker_scope(worker_id, budget)
    MEM-->>DW: scope_allocated
    alt Init success
        DW-->>AGS: worker_ready(worker_id)
    else Init failure (binding error)
        DW-->>AGS: worker_failed(error)
        AGS->>SCE: publish("worker.spawn_failed", {worker_id, error})
        AGS-->>WS: spawn_failed
        WS-->>TS: acquire_failed
    end
    AGS-->>WS: Worker
    WS-->>TS: Worker acquired

    Note over DW: RUNNING state
    TS->>DW: execute(task)
    DW->>SCE: publish("worker.task.started", {task_id, ts})
    DW->>DW: inject Master Prompt + Role Prompt + Task
    DW->>MEM: log_event("task_started", {task_id})

    loop tool calls and model responses
        DW->>MP: model.invoke(prompt)
        MP-->>DW: response{tokens, finish_reason}
        DW->>DW: parse response, extract tool calls
        DW->>MEM: log_event("model_response", {tokens, finish_reason})

        alt Tool call requested
            DW->>TOOL: tool.call(name, args)
            alt Tool success
                TOOL-->>DW: result
                DW->>SCE: publish("worker.tool.result", {tool_name, ok, duration_ms})
            else Tool error
                TOOL-->>DW: error
                DW->>SCE: publish("worker.tool.error", {tool_name, error})
                alt Retryable error
                    DW->>TOOL: tool.call(name, args, retry=1)
                end
            end
        end

        opt Every CHECKPOINT_INTERVAL_MS (default 5000ms)
            DW->>MEM: write_checkpoint({task_id, state_snapshot, budget_spent})
            DW->>SCE: publish("worker.checkpoint", {task_id, checkpoint_id})
        end

        opt Preemption requested
            SCE-->>DW: preemption_request
            DW->>MEM: write_checkpoint({task_id, state_snapshot, preempted: true})
            DW->>SCE: publish("worker.checkpoint", {task_id, preempted: true})
            DW->>MEM: release_worker_scope(worker_id)
            DW-->>TS: yield(worker_id)
            Note over TS,DW: Task rescheduled, worker becomes IDLE
            Note over DW: IDLE state — warm pool candidate
        end

        opt Context compression needed
            DW->>DW: compress_context()
            DW->>SCE: publish("worker.context_compressed", {tokens_before, tokens_after})
        end
    end

    Note over DW: COMPLETING state
    DW->>DW: finalize_artifact()
    DW->>MEM: write_artifact(artifact)
    DW->>MEM: log_budget_final(budget_spent)

    Note over DW: COMPLETED state
    DW-->>TS: result{ok: true, artifact_id}
    DW->>SCE: publish("worker.task.completed", {task_id, duration_ms, tokens_used, artifact_id})

    Note over DW: RETURNING state
    WS->>DW: release()
    alt Warm pool below min_warm
        WS->>DW: enter IDLE pool
        DW->>MEM: mark_idle(worker_id, ttl=60s)
        Note over DW,WS: Worker kept warm with model context retained\nAvailable for sibling task within TTL
    else Warm pool at capacity
        WS->>DW: retire()
        DW->>MEM: release_worker_scope(worker_id)
        DW->>SCE: publish("worker.retired", {worker_id, total_tasks, total_tokens, wall_ms})
        DW-->>WS: destroyed
    end
```

## Checkpoint States

| State | Label | Trigger | Duration | Persistence |
|-------|-------|---------|----------|-------------|
| Periodic | `worker.checkpoint` | `CHECKPOINT_INTERVAL_MS` elapsed | ~50ms | Persistent Memory |
| Preemptive | `worker.checkpoint` + `preempted: true` | External preemption signal | ~100ms | Persistent Memory (high priority) |
| Explicit | `worker.checkpoint` + `trigger: explicit` | Worker calls `checkpoint()` | ~50ms | Persistent Memory |
| Final | `worker.completed` | Task finishes | — | Persistent Memory (artifact + budget) |

## Preemption Flow

```mermaid
flowchart LR
    SCE[SCE: preemption_request] --> DW[Worker receives signal]
    DW --> CHECKPOINT[Write emergency checkpoint\nto Persistent Memory]
    CHECKPOINT --> YIELD[Return yield() to Task Scheduler]
    YIELD --> RESCHEDULE[Task rescheduled\nwith checkpoint_id]
    RESCHEDULE --> NEW_WORKER[New or warm worker\nloads checkpoint and continues]
```

Preemption is triggered when:
- A higher-priority task enters the queue
- Kernel initiates run cancellation
- Budget exhaustion is imminent
- System resources are under pressure (memory, API rate limits)

## Warm Pool Integration

```mermaid
flowchart LR
    COMPLETED[Task completed] --> POOL{Warm pool\nhas capacity?}
    POOL -->|yes, pool_size < max_warm| WARM[Worker enters IDLE\nTTL = 60s]
    POOL -->|no, pool at capacity| RETIRE[Worker retires]

    WARM --> SIBLING{Next task for\nsame group_spec?}
    SIBLING -->|within TTL| REUSE[Reuse worker\nno cold start]
    SIBLING -->|TTL expired| RETIRE

    RETIRE --> FREE[Resources freed\nmodel client closed]
```

- Default warm pool size: `min(5, max_workers * 0.2)`
- Per-group warm pool: `max_warm = 2` (group-specific configuration)
- TTL extensions: each reassignment resets the 60s TTL

## Error Recovery Transitions

```mermaid
stateDiagram-v2
    RUNNING --> Recovery : model call error (retryable)
    Recovery --> RUNNING : retry succeeds
    Recovery --> ERROR : all retries exhausted

    RUNNING --> ERROR : unrecoverable error

    ERROR --> RESTARTING : crash_count < threshold (default 3)
    RESTARTING --> INITIALISING : restart with same group_spec
    RESTARTING --> RETIRED : crash_count >= threshold

    ERROR --> RETIRED : AGS marks worker as failed
```

## Resource Cleanup Guarantee

When a worker enters `RETIRING` (either through normal retirement, error, or cancellation):

1. **Tool handles** are released — all MCP/plugin connections are closed with a 5s graceful shutdown.
2. **SCE cursors** are unsubscribed — the worker's topic subscriptions are removed.
3. **Budget** is reported — final `budget_spent` is written to Persistent Memory.
4. **Warm pool slot** is freed — the slot becomes available for a new worker.
5. **Checkpoint** is written if not already — ensures no work is lost on error retirement.

## Lifecycle Event Catalog

| Event | Publisher | Payload | Retention |
|-------|-----------|---------|-----------|
| `worker.spawn_failed` | AGS | `{worker_id, error, group_spec}` | 90d |
| `worker.started` | Worker | `{worker_id, role, model_id, tools[]}` | 90d |
| `worker.task_assigned` | AGS | `{task_id, budget_slice}` | 90d |
| `worker.task.started` | Worker | `{task_id, ts}` | 90d |
| `worker.token` | Worker | `{text, finish_reason}` | 7d (high volume) |
| `worker.tool_call` | Worker | `{name, args}` | 90d |
| `worker.tool.result` | Worker | `{tool_name, ok, duration_ms}` | 90d |
| `worker.tool.error` | Worker | `{tool_name, error, retryable}` | 90d |
| `worker.checkpoint` | Worker | `{checkpoint_id, budget_spent}` | 90d |
| `worker.context_compressed` | Worker | `{tokens_before, tokens_after}` | 30d |
| `worker.task.completed` | Worker | `{artifact_id, duration_ms, tokens_used}` | 90d |
| `worker.failed` | Worker | `{error_code, message, budget_spent}` | 90d |
| `worker.cancelling` | Worker | `{reason}` | 90d |
| `worker.cancelled` | Worker | `{reason, budget_spent}` | 90d |
| `worker.budget_exhausted` | Worker | `{budget_type, spent, limit}` | 90d |
| `worker.retired` | Worker | `{total_tasks, total_tokens, wall_ms}` | 90d |

## Failure Scenarios

| Scenario | Detection | Effect | Recovery |
|----------|-----------|--------|----------|
| Model provider 5xx | Worker receives error | Immediate retry with same model (max 3) | If exhausted → fallback model chain |
| Model provider timeout | Request exceeds timeout (default 120s) | Same as 5xx; checkpoint saved before retry | Fallback chain if primary times out > 3x |
| Tool call error (retryable) | Tool returns retryable error | Worker retries with backoff: 1s, 5s, 15s | If exhausted → mark failure, continue without tool result |
| Tool call error (fatal) | Tool returns non-retryable error | Worker continues with partial result | Tool result excluded from artifact |
| Context overflow | Token count > context_window | Worker compresses and continues | Compression logged; sliding window applied |
| Checkpoint write failure | Persistent Memory unavailable | Worker continues without checkpoint | Last checkpoint used for recovery (if any) |
| Worker crash (OOM, panic) | AGS detects heartbeat loss | Worker marked as failed; checkpoint restored | New worker spawned from last checkpoint |

## Implementation Notes

- Checkpoint interval is configurable per group via `checkpoint_interval_ms` (default 5000ms).
- Preemption is signalled through an SCE topic the worker subscribes to during `INITIALISING`.
- Warm pool TTL is a global config with per-group override (`warm_pool_ttl_ms`, default 60000).
- The `crash_count` threshold is configurable via `AGS.max_worker_restarts` (default 3).
- All lifecycle events are written to Persistent Memory in addition to SCE for audit log durability.

## Related Documents

- [Dynamic Workers](../docs/DYNAMIC_WORKERS.md) — worker execution and state machine
- [Worker Scheduler](../docs/WORKER_SCHEDULER.md) — worker pool management
- [Task Scheduler](../docs/TASK_SCHEDULER.md) — task dispatch
- [AI Group System](../docs/AI_GROUP_SYSTEM.md) — worker pool ownership
- [Agent Lifecycle](../docs/AGENT_LIFECYCLE.md) — worker lifecycle and checkpointing
- [Tool Calling](../docs/TOOL_CALLING.md) — tool dispatch and error handling
