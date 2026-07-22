# Agent Lifecycle — State Machine and Event Schema

> Complete state machine for a Dynamic Worker, from spawn to retirement, with all events published to the SCE.

## State Machine

```mermaid
stateDiagram-v2
  [*] --> Initialising : AGS calls spawn_group\nWorkerSpec delivered

  Initialising --> Ready : model bound\ntools loaded\nKB scope configured

  Initialising --> Failed : binding failure\nor tool load error

  Ready --> Executing : task assigned\nfrom queue

  Executing --> Checkpointing : checkpoint_interval_ms elapsed\nor explicit checkpoint trigger

  Checkpointing --> Executing : checkpoint saved\nto Persistent Memory\ncontinue task

  Executing --> Completing : model signals done\nartifact ready

  Completing --> Completed : artifact emitted\nbudget recorded

  Executing --> Failed : unrecoverable error\n(e.g., all fallbacks exhausted)

  Executing --> Cancelling : kernel.cancel(run_id)\nor AGS shutdown signal

  Cancelling --> Cancelled : in-flight tool call aborted\ntool handles released

  Completed --> Ready : warm pool — reuse\nfor sibling task

  Completed --> Retiring : no more tasks\nGroupRun draining

  Cancelled --> Retiring : clean up

  Failed --> Retiring : AGS may restart\nif crash_count < threshold

  Retiring --> [*] : tool handles released\nSCE cursors unsubscribed\nbudget reported
```

## Lifecycle Events on SCE

All events are published on `run.<run_id>` topic with `worker_id`, `task_id`, `correlation_id`, and `ts`.

| State transition | SCE Event | Key payload fields |
|-----------------|-----------|-------------------|
| Initialising → Ready | `worker.started` | `role, model_id, tools[]` |
| Ready → Executing | `worker.task_assigned` | `task_id, budget_slice` |
| Executing (stream) | `worker.token` | `text, finish_reason?` |
| Executing (tool) | `worker.tool_call` | `name, args, result?, error?, duration_ms` |
| Tool denied | `worker.tool_denied` | `name, reason` |
| Fallback used | `router.fallback` | `role, from_model, to_model, reason` |
| Checkpointing → Executing | `worker.checkpointed` | `checkpoint_id, budget_spent` |
| Completing → Completed | `worker.completed` | `artifact_id, budget_spent` |
| Executing → Failed | `worker.failed` | `error_code, message, budget_spent` |
| Executing → Cancelling | `worker.cancelling` | `reason` |
| Cancelling → Cancelled | `worker.cancelled` | `reason, budget_spent` |
| Budget exceeded | `worker.budget_exhausted` | `budget_type, spent, limit` |
| Context compressed | `worker.context_compressed` | `tokens_before, tokens_after` |
| Retiring → [*] | `worker.retired` | `total_tasks, total_tokens, wall_ms` |

## Checkpoint Structure

```mermaid
flowchart LR
  subgraph Checkpoint["Checkpoint record\n(stored in Persistent Memory)"]
    W_ID[worker_id: ulid]
    T_ID[task_id: ulid]
    R_ID[run_id: ulid]
    TS[ts: rfc3339]
    CTX_H[context_hash: sha256\nhash of current context window]
    BUDGET[budget_spent: tokens + ms + usd]
    TOOLS[tool_history: ToolCall[]\nall tool calls so far]
    PARTIAL[partial_artifact: string?\npartial output]
    MODEL_S[model_state: object?\nprovider continuation state]
  end

  MEM[(Persistent Memory)] --> Checkpoint
  Checkpoint --> REPLAY[Replay: fresh worker\nloads checkpoint and continues]
```

## Budget Lifecycle

```mermaid
flowchart LR
  KERNEL[Kernel] -->|budget: tokens_max + wall_ms_max + usd_max| WORKER[Worker]
  WORKER --> TRACKER[BudgetTracker]

  TRACKER --> TOK[tokens used\nper API response]
  TRACKER --> WALL[wall_ms elapsed\nmonotonic clock]
  TRACKER --> USD[usd spent\nfrom pricing hints]

  TOK & WALL & USD --> CHECK{budget.exceeded?}
  CHECK -->|no| CONTINUE[Continue execution]
  CHECK -->|yes| CANCEL[Cancel model call\nemit worker.budget_exhausted\ndeliver partial artifact]
```

## Warm Pool Strategy

```mermaid
flowchart LR
  TASK_DONE[Task completed] --> POOL{Warm pool\nhas capacity?}
  POOL -->|yes| WARM[Worker stays warm\nmodel context retained]
  POOL -->|no| RETIRE[Worker retires\nresources freed]

  WARM --> NEXT_TASK{Next sibling task\nfor same group?}
  NEXT_TASK -->|within TTL| REUSE[Reuse warm worker\nno cold start]
  NEXT_TASK -->|TTL expired| RETIRE2[Worker retires]
```

## Related Documents

- [Dynamic Workers](../docs/DYNAMIC_WORKERS.md)
- [AI Group System](../docs/AI_GROUP_SYSTEM.md)
- [Agent Memory](../docs/AGENT_MEMORY.md)
- [Persistent Memory](../docs/PERSISTENT_MEMORY.md)
- [Tool Calling](../docs/TOOL_CALLING.md)
- [Main AI Kernel](../docs/MAIN_AI_KERNEL.md)
