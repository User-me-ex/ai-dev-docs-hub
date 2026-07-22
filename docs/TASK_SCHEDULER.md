# Task Scheduler

> Intra-Kernel task scheduling — allocation of tasks from the TaskGraph to worker pools with priority, dependency ordering, fairness, and preemption. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

The Task Scheduler is the subsystem within the [Main AI Kernel](./MAIN_AI_KERNEL.md) responsible for taking a resolved `TaskGraph` (produced by the [Planning Engine](./PLANNING_ENGINE.md)) and dispatching each task to an available worker in dependency order. It is distinct from the [Job Scheduler](./JOB_SCHEDULER.md), which handles recurring cron-style jobs (model discovery, research crawls, retention). The Task Scheduler handles *run-time* task dispatch within a single Kernel run.

The Task Scheduler answers three questions:
1. **Which task is ready next?** — all dependencies resolved, budget available, resources available.
2. **Which worker should execute it?** — via the [AI Group System](./AI_GROUP_SYSTEM.md) and [Nine Router](./NINE_ROUTER.md).
3. **What happens on failure?** — retry, fallback, or fail the task.

## Goals

- Tasks are executed in topological order: no task starts before all its dependencies complete.
- Independent sibling tasks are executed in parallel up to the available worker pool limit.
- A failed task does not block the entire run — sibling tasks can continue while the failed task is retried or replanned.
- Priority within a run is determined by critical-path analysis: tasks on the critical path are scheduled before non-critical siblings.
- Preemption is cooperative: a running task is checkpointed before a higher-priority task can claim its worker.

## Non-Goals

- Recurring job scheduling — handled by [Job Scheduler](./JOB_SCHEDULER.md).
- Worker lifecycle management — handled by [AI Group System](./AI_GROUP_SYSTEM.md) and [Agent Lifecycle](./AGENT_LIFECYCLE.md).
- Queueing across runs — handled by [Queueing](./QUEUEING.md) subsystem.
- Implementation code — this repo is documentation-only ([AI Coding Rules](./AI_CODING_RULES.md)).

## Task State Machine

```
                         ┌──────────┐
                         │ PENDING  │
                         └────┬─────┘
                              │ dependencies resolved
                              │ budget ok
                              v
                   ┌──────────────────┐
        ┌─────────│    READY          │
        │         └────────┬─────────┘
        │                  │ dispatched
        │                  v
        │         ┌──────────────────┐
        │         │  RUNNING         │
        │         └────────┬─────────┘
        │                  │
        │         ┌────────┴────────┐
        │         │                 │
        │         v                 v
        │   ┌──────────┐    ┌──────────┐
        │   │ COMPLETED │    │ FAILED   │
        │   └──────────┘    └────┬─────┘
        │                        │ retry count < max
        │                        v
        │                  ┌──────────┐
        └──────────────────│  READY   │ (re-queued)
                           └──────────┘
```

| State | Meaning | Entry condition | Exit condition |
|-------|---------|-----------------|----------------|
| `PENDING` | Awaiting dependency resolution | All deps not completed | All deps completed + budget ok |
| `READY` | Eligible for dispatch | Dependencies met | Worker assigned |
| `RUNNING` | Executing on a worker | Worker claimed task | Worker returns/completes/checkpoints |
| `COMPLETED` | Finished successfully | Worker returns success | — |
| `FAILED` | Terminated with error | Worker returns error or timeout | Retry count < max → back to READY; retry exhausted → terminal |

## Scheduling Algorithm

```python
def schedule(tasks: List[Task], pool: WorkerPool, budget: Budget):
    """
    Main scheduling loop. Runs until all tasks are COMPLETED or FAILED (terminal).
    Yields tasks to the worker pool as they become ready.
    """
    remaining = set(tasks)
    running = {}
    completed = set()
    failed = set()

    while remaining:
        # 1. Compute ready set: tasks whose deps are all completed
        ready = {t for t in remaining
                 if all(dep in completed for dep in t.dependencies)
                 and t.budget.tokens <= budget.remaining.tokens
                 and t.budget.wall_ms <= budget.remaining.wall_ms}

        # 2. If no ready tasks and no running tasks → deadlock or all failed
        if not ready and not running:
            for t in remaining:
                t.state = FAILED
                t.reason = "deadlock: dependencies cannot be satisfied"
                failed.add(t)
            break

        # 3. Prioritise ready tasks: critical path first, then most-dependents
        ready_sorted = sorted(ready, key=lambda t: (
            -t.critical_path_length,   # longer path = higher priority
            -len(t.dependents),        # more dependents = higher priority
            t.submitted_at             # FIFO tiebreaker
        ))

        # 4. Dispatch as many as the pool allows
        for task in ready_sorted:
            worker = pool.acquire(task)
            if worker is None:
                break  # pool exhausted
            task.state = RUNNING
            running[task.id] = task
            remaining.remove(task)
            worker.execute(task)

        # 5. Collect completed/failed tasks
        for task_id, result in pool.collect_results(timeout=POLL_INTERVAL_MS):
            task = running.pop(task_id)
            if result.ok:
                task.state = COMPLETED
                completed.add(task)
            else:
                if task.retry_count < task.max_retries:
                    task.retry_count += 1
                    task.state = READY
                    remaining.add(task)
                else:
                    task.state = FAILED
                    task.reason = result.error
                    failed.add(task)

    return RunResult(completed=completed, failed=failed)
```

## Critical Path Analysis

The Task Scheduler computes a critical path for each task graph at submission time:

```
For each task T in topological order (reverse):
    T.critical_path_length = max(
        T.estimated_duration_ms,
        max(T.dependents, key=lambda d: d.critical_path_length).critical_path_length + T.estimated_duration_ms
    )
```

Tasks on the critical path (`T.critical_path_length == max_critical_path_length`) are scheduled before all non-critical tasks, even if their dependencies are met later.

## Priority and Preemption

| Priority level | Source | Preempts |
|----------------|--------|----------|
| `CRITICAL` | Critical path task, Guardian veto replan | Any non-critical |
| `HIGH` | User-facing interaction, dependent-blocking task | `NORMAL`, `LOW` |
| `NORMAL` | Default task priority | `LOW` |
| `LOW` | Background research, KB indexing | — |

Preemption is cooperative: the scheduler requests a checkpoint from the running worker via the [Agent Lifecycle](./AGENT_LIFECYCLE.md) `checkpoint()` API. The worker checkpoints and yields; the scheduler assigns the worker to the higher-priority task. The preempted task is re-queued as READY.

A task that has checkpointed more than twice is not preemptable (to prevent starvation).

## Interfaces

```
scheduler.submit(task_graph: TaskGraph, run_id: string) → RunHandle
scheduler.status(run_id) → SchedulingStatus
scheduler.pause(run_id) → Ack        # suspend all scheduling for this run
scheduler.resume(run_id) → Ack       # resume suspended run
scheduler.cancel(run_id, task_id?) → Ack  # cancel run or single task
scheduler.priority(run_id, task_id, priority) → Ack  # runtime priority change
```

```json
SchedulingStatus {
  run_id: string
  state: "scheduling" | "paused" | "completed" | "failed"
  tasks: {
    pending: int,
    ready: int,
    running: int,
    completed: int,
    failed: int
  }
  workers_used: int
  workers_available: int
  estimated_remaining_ms: int
}
```

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| Deadlocked tasks | No READY tasks, no RUNNING tasks, PENDING tasks remain | Mark all remaining as FAILED(reason="deadlock"); notify Kernel for replan |
| Worker pool exhausted | All workers busy, tasks READY | Wait; emit `scheduler.pool_exhausted` event; backpressure to Queue |
| Task timeout | Task runs longer than budget.wall_ms | Checkpoint and cancel; mark FAILED(reason="timeout"); retry if retries remain |
| Preemption loop | Same task preempted > 3 times | Mark task non-preemptable; allow it to complete; log WARN |
| Budget exceeded | Run budget exhausted mid-scheduling | Cancel all READY tasks; let RUNNING tasks complete; deliver partial results |
| Stalled scheduler | No progress for > 60s | Heartbeat check on all workers; if worker(s) unresponsive, reassign their tasks |

## Performance Budget

| Operation | p99 Target |
|-----------|------------|
| `scheduler.submit` (100-task graph) | < 10 ms |
| Per-scheduling tick (idle) | < 100 µs |
| Per-scheduling tick (50 tasks, 10 ready) | < 500 µs |
| Critical path computation (100 tasks) | < 5 ms |
| Preemption handover (checkpoint + reassign) | < 1 s |

## Acceptance Criteria

- A TaskGraph with three independent tasks completes in approximately the same wall time as the slowest task (parallel execution).
- A TaskGraph where Task C depends on tasks A and B schedules A and B first, and C only after both complete.
- A task that fails twice and succeeds on the third attempt is correctly recorded with `retry_count: 2`.
- Preempting a LOW-priority task with a CRITICAL task completes the critical task first, then resumes the preempted task from its checkpoint.
- Submitting 100 tasks with 5 available workers results in at most 5 concurrent RUNNING tasks at any point.
- The scheduler correctly detects a deadlock when two tasks each depend on the other (cycle in dependencies) and marks both as FAILED.

## Related Documents

- [Worker Scheduler](./WORKER_SCHEDULER.md) — worker-level scheduling within a pool
- [Job Scheduler](./JOB_SCHEDULER.md) — cron-style recurring job scheduling
- [Planning Engine](./PLANNING_ENGINE.md) — produces the TaskGraph the scheduler consumes
- [Task Graph](./TASK_GRAPH.md) — TaskGraph data model
- [AI Group System](./AI_GROUP_SYSTEM.md) — worker pool management
- [Queueing](./QUEUEING.md) — cross-run queueing
- [System Overview](./SYSTEM_OVERVIEW.md)
- [Main AI Kernel](./MAIN_AI_KERNEL.md)
