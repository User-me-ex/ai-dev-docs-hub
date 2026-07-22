# Worker Scheduler

> Worker-level scheduling within a pool — allocation, pooling, health checks, and lifecycle management of Dynamic Workers. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

The Worker Scheduler is the component within the [AI Group System](./AI_GROUP_SYSTEM.md) that manages the pool of [Dynamic Workers](./DYNAMIC_WORKERS.md) available to execute tasks dispatched by the [Task Scheduler](./TASK_SCHEDULER.md). While the Task Scheduler answers "which task should run next?", the Worker Scheduler answers "which worker should run it?"

The Worker Scheduler maintains a pool of warm (pre-warmed) worker processes or coroutines, assigns workers to tasks, monitors their health via heartbeats, and replaces failed or stuck workers.

## Goals

- Minimise cold-start latency: a pool of pre-warmed workers is ready to accept tasks within the configured pool size.
- Workers are reused across tasks within the same run to avoid spawning a new process for every task.
- A worker that crashes or becomes unresponsive is detected and replaced within `heartbeat_grace_ms`.
- The pool scales elastically: workers are added under load and removed during idle periods.
- Each worker is isolated: one worker's crash does not affect other workers or the Kernel.

## Non-Goals

- Task scheduling decisions — handled by [Task Scheduler](./TASK_SCHEDULER.md).
- Cross-run queueing — handled by [Queueing](./QUEUEING.md).
- Model binding resolution — handled by [Nine Router](./NINE_ROUTER.md).
- Implementation code — this repo is documentation-only ([AI Coding Rules](./AI_CODING_RULES.md)).

## Worker Pool Model

```
WorkerPool {
  id:            string                # pool_id = "group:code-builder"
  group:         GroupSpec
  max_size:      int                   # maximum concurrent workers (default: 10)
  min_warm:      int                   # always keep this many warm (default: 2)
  ttl_idle_ms:   int                   # idle worker TTL before retirement (default: 300_000 = 5 min)
  heartbeat_ms:  int                   # worker heartbeat interval (default: 5000 = 5 s)
  heartbeat_grace_ms: int             # missed heartbeats before assumed dead (default: 15000 = 15 s)

  workers:       Map<worker_id, Worker>
  idle_queue:    Queue<Worker>         # warm, available workers
  pending:       Queue<TaskBinding>    # tasks awaiting worker assignment
}
```

## Worker State Machine

```
                  ┌──────────┐
                  │ WARMING  │
                  └────┬─────┘
                       │ init complete
                       v
              ┌────────────────┐
  ┌───────────│     IDLE       │
  │           └───────┬────────┘
  │                   │ assigned task
  │                   v
  │           ┌────────────────┐
  │           │   RUNNING      │
  │           └───────┬────────┘
  │                   │
  │          ┌────────┴────────┐
  │          │                 │
  │          v                 v
  │    ┌──────────┐    ┌──────────┐
  │    │ CRASHED  │    │ CHECKPOINT│ (during preemption)
  │    └────┬─────┘    └────┬─────┘
  │         │               │ task complete?
  │         v               v
  │    ┌──────────┐    ┌──────────┐
  │    │ RETIRED  │    │ RETURNING│
  │    └──────────┘    └────┬─────┘
  │                         │
  └─────────────────────────┘ (back to IDLE)
```

| State | Meaning | Entry | Exit |
|-------|---------|-------|------|
| `WARMING` | Worker process/coroutine being initialised | Pool expansion trigger | Init complete |
| `IDLE` | Warm, awaiting task assignment | `WARMING` complete, or `RETURNING` | Task assigned |
| `RUNNING` | Executing a task | Assigned a `TaskBinding` | Task complete, crash, or preemption |
| `CHECKPOINT` | Preempted, checkpointing state | Preemption request from Task Scheduler | Checkpoint saved → `IDLE` |
| `CRASHED` | Unexpected exit | Heartbeat miss, process death | Detection → `RETIRED` |
| `RETIRED` | Removed from pool | Crash, TTL expiry, pool shrink | Destroyed |
| `RETURNING` | Task completed, returning to pool | Task success/failure | State saved → `IDLE` (or `RETIRED` if pool is full) |

## Scheduling Algorithm

```python
def acquire_task(pool: WorkerPool, task: Task) -> Worker | None:
    """
    Assign a task to the best available worker. Returns None if pool is saturated.
    """
    # 1. Prefer IDLE worker with same model binding (avoid model load time)
    preferred = [w for w in pool.idle_queue
                 if w.model_binding.model_id == task.model_binding.model_id]
    if preferred:
        worker = preferred[0]
        pool.idle_queue.remove(worker)
        worker.assign(task)
        return worker

    # 2. Fall back to any IDLE worker
    if pool.idle_queue:
        worker = pool.idle_queue.pop()
        worker.assign(task)
        return worker

    # 3. Pool saturated: spawn new worker if under max_size
    if len(pool.workers) < pool.max_size:
        worker = spawn_worker(pool.group, task.model_binding)
        pool.workers[worker.id] = worker
        worker.assign(task)
        return worker

    # 4. Pool full: enqueue task as pending
    pool.pending.push(task)
    return None


def release_worker(pool: WorkerPool, worker: Worker):
    """
    Return a worker to the pool or retire it.
    """
    if len(pool.idle_queue) < pool.min_warm:
        worker.state = IDLE
        pool.idle_queue.push(worker)
    else:
        worker.state = RETIRED
        destroy_worker(worker)
        del pool.workers[worker.id]

    # If tasks are pending, assign immediately
    if pool.pending and pool.idle_queue:
        task = pool.pending.pop()
        worker = pool.idle_queue.pop()
        worker.assign(task)
```

## Elastic Scaling

| Condition | Action |
|-----------|--------|
| Pending queue depth > 0 for > 5s | Increase `max_size` by 20% (up to global max) |
| No pending tasks for > 60s | Decrease `max_size` by 20% (down to `min_warm`) |
| Worker pool at max with pending tasks | Emit `worker_pool.saturated` event; Task Scheduler applies backpressure |
| Idle worker count > max_size * 0.5 for > 5 min | Retire excess workers until idle count = target (max_size * 0.2) |

## Heartbeat Protocol

Every active (RUNNING or IDLE) worker sends a heartbeat to the Worker Scheduler every `heartbeat_ms`:

```json
Heartbeat {
  worker_id: string
  state: "running" | "idle"
  task_id: string | null
  tokens_used: int
  wall_ms_used: int
  memory_mb: float
  ts: rfc3339
}
```

The Worker Scheduler tracks the last heartbeat timestamp per worker. If a worker misses `heartbeat_grace_ms / heartbeat_ms` consecutive heartbeats, it is declared `CRASHED`:

1. The worker is removed from the pool (`RETIRED`).
2. Its assigned task (if any) is marked `FAILED(reason="worker_crashed")`.
3. The Task Scheduler retries the task on a new worker.
4. A replacement worker is spawned if the pool is below `min_warm`.

## Circuit Breaker

The Worker Scheduler maintains a circuit breaker per worker process to prevent repeatedly assigning tasks to a failing worker:

| State | Condition | Behaviour |
|-------|-----------|-----------|
| `CLOSED` | Normal operation | Tasks assigned normally |
| `OPEN` | Error rate > 50% in last 10 tasks | No tasks assigned; worker retired |
| `HALF_OPEN` | After `cooldown_ms` (default 30s) | One test task allowed; if it succeeds → CLOSED; if it fails → OPEN |

## Interfaces

```
worker_pool.get(pool_id) → WorkerPool
worker_pool.acquire(task: Task) → Worker | None
worker_pool.release(worker: Worker)
worker_pool.status(pool_id) → PoolStatus
worker_pool.list_pools() → PoolSummary[]
worker_pool.scale(pool_id, delta: int)    # manual scale adjustment
```

```json
PoolStatus {
  pool_id: string
  workers: {
    warming: int,
    idle: int,
    running: int,
    crashed: int,
    retired: int
  }
  pending_queue_depth: int
  max_size: int
  min_warm: int
  circuit_breaker: "closed" | "open" | "half-open"
}
```

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| Worker crash | Heartbeat miss | Mark CRASHED; retry task on new worker; replace worker |
| Worker leak | Process count > max_size * 1.5 | Force-kill oldest IDLE workers; log CRITICAL |
| Pool saturation | Pending queue growing | Emit backpressure; Task Scheduler pauses new dispatching |
| Circuit breaker open | Error rate high | Retire failing worker; spawn replacement; log WARN |
| Fork bomb | Workers spawning faster than retiring | Hard cap on spawn rate (max 1/s); log CRITICAL; alert operator |
| Model load failure | Worker WARMING fails | Log ERROR; retire worker; emit `worker.warm_failure` metric |
| Heartbeat storm | All workers heartbeat simultaneously | Jitter heartbeat timing by ±20%; queue heartbeats |

## Performance Budget

| Operation | p99 Target |
|-----------|------------|
| Pool.acquire (IDLE worker available) | < 100 µs |
| Pool.acquire (spawn new worker) | < 500 ms (warm) |
| Heartbeat processing | < 10 µs per heartbeat |
| Circuit breaker state check | < 5 µs |
| Scale up (spawn 1 worker) | < 500 ms |
| Scale down (retire 1 worker) | < 50 ms |

## Acceptance Criteria

- Submitting 10 tasks to a pool with `max_size = 3` results in at most 3 concurrently RUNNING workers and a pending queue of 7.
- A worker that stops sending heartbeats is declared CRASHED and replaced within `heartbeat_grace_ms` (default 15s).
- Two workers with the same model binding: a task targeting that model is assigned to the IDLE worker, not a new spawn.
- The pool scales up from 2 to 6 workers under sustained load (pending > 0 for 30s) and scales back to 2 during a 5-minute idle period.
- A worker that crashes 5 times in a row triggers the circuit breaker and is not reassigned tasks.
- After all tasks complete, the pool retains exactly `min_warm` workers (default 2) and retires the rest within `ttl_idle_ms`.

## Related Documents

- [Task Scheduler](./TASK_SCHEDULER.md) — consumes workers from this pool
- [AI Group System](./AI_GROUP_SYSTEM.md) — creates, owns, and configures worker pools
- [Dynamic Workers](./DYNAMIC_WORKERS.md) — the worker execution unit
- [Agent Lifecycle](./AGENT_LIFECYCLE.md) — worker state machine and checkpointing
- [Queueing](./QUEUEING.md) — cross-run queuing
- [System Overview](./SYSTEM_OVERVIEW.md)
