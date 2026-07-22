# ADR-0002: Cooperative vs Preemptive Scheduling

## Status

Accepted

## Context

The [Task Scheduler](../TASK_SCHEDULER.md) must decide how to handle higher-priority tasks when workers are occupied with lower-priority work. Two approaches exist:

**Cooperative**: Tasks voluntarily yield control at defined checkpoint boundaries. The scheduler cannot forcibly interrupt a running task.

**Preemptive**: The scheduler can forcibly suspend a running task (at any instruction boundary) and assign the worker to a higher-priority task.

AI Dev OS runs agents that make LLM calls, which can take 1–60 seconds. The architecture emphasises checkpointing and budget enforcement.

## Decision

**Cooperative scheduling with checkpoints as preemption points.**

1. The [Agent Lifecycle](../AGENT_LIFECYCLE.md) defines explicit checkpoint boundaries (after each tool call, after each model response, every `checkpoint_interval_ms`).
2. The [Task Scheduler](../TASK_SCHEDULER.md) requests preemption at the next checkpoint boundary by sending a `preemption_request` signal via the SCE.
3. The worker completes its current unit of work (tool call or model response), checkpoints its state, and yields.
4. A task that has been preempted more than twice is marked **non-preemptable** — it runs to completion without further interruption to prevent starvation.
5. Preemption is not supported for tasks that hold exclusive resources (file write locks, database transactions). Those tasks declare `preemptable: false` in their task spec.

### Checkpoint Interval Calculation Algorithm

The checkpoint interval is computed dynamically per task based on its characteristics:

```
Algorithm: ComputeCheckpointInterval
Input: task (TaskSpec), model_latency_p50_ms, historical_yield_time_ms
Output: checkpoint_interval_ms

01  base_interval = 5000  // 5 seconds base
02
03  // Scale by task complexity (number of expected steps)
04  if task.estimated_steps > 50:
05      complexity_factor = 0.8
06  elif task.estimated_steps > 20:
07      complexity_factor = 0.9
08  else:
09      complexity_factor = 1.0
10
11  // Scale by model latency (longer calls = fewer checkpoints)
12  if model_latency_p50_ms > 15000:
13      latency_factor = 1.5
14  elif model_latency_p50_ms > 5000:
15      latency_factor = 1.2
16  else:
17      latency_factor = 1.0
18
19  // Scale by priority (higher priority = more frequent checkpoints)
20  priority_factor = 2.0 - (task.priority / 100)  // priority 100 → 1.0, priority 0 → 2.0
21
22  // Max interval bounds
23  interval = min(max(
24      base_interval * complexity_factor * latency_factor * priority_factor,
25      1000   // min: 1 second
26  ), 30000)  // max: 30 seconds
27
28  return round(interval, 100)  // Round to nearest 100ms
```

### Preemption Signal Protocol

The preemption signal flows through the SCE event bus:

```
// Signal: Scheduler → Worker (via SCE)
{
  "event_type": "scheduler.preemption_request",
  "payload": {
    "task_id": "task-042",
    "current_task_id": "task-019",
    "new_task_priority": 85,
    "current_task_priority": 30,
    "reason": "high_priority_interrupt",
    "requested_at": 1735689600000,
    "deadline": 1735689605000  // Worker must respond by this time
  }
}

// Response: Worker → Scheduler (via SCE)
{
  "event_type": "scheduler.preemption_response",
  "payload": {
    "task_id": "task-019",
    "response": "yield" | "non_preemptable" | "busy",
    "current_checkpoint": "checkpoint_008",
    "estimated_remaining_ms": 12000,
    "preemption_count": 2,
    "responded_at": 1735689600500
  }
}

// If no response within deadline, Scheduler treats as "busy" and retries
// after a backoff period: backoff_ms = min(5000 * 2^retry_count, 60000)
```

### Starvation Prevention Proof

**Claim**: Cooperative scheduling with the `max_preemptions = 2` rule prevents task starvation.

**Proof**:
1. Let T be a task of priority p.
2. T can be preempted at most 2 times (by higher-priority tasks H1, H2).
3. After the 2nd preemption, T is marked non-preemptable.
4. T runs to completion without further interruption.
5. During T's execution, H3 (higher priority than T) may arise, but H3 must wait for T to complete.

**Worst-case latency for H3**: T's remaining execution time after its 2nd preemption. With checkpointing every 5s and max preemption loss of 2 checkpoints, T's max execution is:
- Original estimated duration + 2 × checkpoint overhead (state save + restore)
- For a 60s task with 500ms checkpoint overhead: 60s + 1s = 61s worst case.

**Edge case**: If H3 has priority that exceeds a system-defined `critical_threshold` (e.g., priority > 95), it may forcibly preempt even a non-preemptable task. This is reserved for safety-critical operations (e.g., resource exhaustion, security violation).

### Conflict with Long-Lived Tasks

Long-lived tasks (LLM calls, file operations) present a challenge because they do not naturally hit checkpoints:

| Task Type | Duration | Preemptable? | Strategy |
|-----------|----------|-------------|----------|
| LLM API call | 1–60s | No (in-flight) | Wait for response; checkpoint on return |
| File write (large) | 0.5–5s | Yes | Checkpoint every 1MB written |
| File read (large) | 0.1–2s | Yes | Checkpoint every 10MB read |
| Database transaction | 0.01–1s | No (atomic) | Complete; checkpoint on commit |
| Plugin execution | 0.1–30s | Plugin-defined | Respect plugin's `preemptable` flag |
| Vector index rebuild | 10–300s | Yes | Checkpoint every 1000 vectors indexed |
| Memory summarization | 5–60s | Yes | Checkpoint after each model call |

For LLM calls specifically: the Kernel sets a `max_llm_wait_ms` parameter. If the call exceeds this and a preemption is pending, the Kernel can cancel the in-flight LLM call (wasting tokens but freeing the worker). This is a last resort — the system prefers to wait for completion.

### Comparative Analysis of Preemptive Approaches

| Approach | Complexity | Starvation Risk | Work Loss | Node.js Compat | Token Waste |
|----------|-----------|-----------------|-----------|----------------|-------------|
| Cooperative (chosen) | Low | Low (2-preemption limit) | None (checkpointed) | Yes | None |
| OS-level thread preemption | High | Low | High (no checkpoint) | No (requires Rust) | High |
| Green thread preemption (Node.js Workers) | Medium | Medium | Medium (partial state) | Partial | Medium |
| Time-sliced (fixed quantum) | Medium | Low | Medium (quantum boundary) | Partial | Medium |
| Priority inversion aware | Very high | Very low | None | No | None |

### Implementation Phases

#### Phase 1 (v0.1): Basic Cooperative Scheduling

- Fixed checkpoint interval per task (default 5s)
- Preemption requests sent via SCE; worker yields at next checkpoint
- Non-preemptable flag (`preemptable: false`)
- Simple priority queue in Task Scheduler

**Acceptance Criteria:**
- A task with `preemptable: true` yields within 5s + current unit of work latency
- A task with `preemptable: false` runs to completion without interruption
- Preemption count is tracked and tasks become non-preemptable after 2 preemptions

#### Phase 2 (v0.2): Adaptive Checkpoint Intervals

- Dynamic interval based on task complexity, model latency, priority
- Preemption backoff and retry logic
- Scheduler exposes `checkpoint_interval_override` API

**Acceptance Criteria:**
- High-priority tasks (priority > 80) preempt low-priority tasks within 2s
- Starvation prevention proof is validated in simulation (1000+ random tasks)
- No task is preempted more than 3 times (enforced by hard limit)

#### Phase 3 (v1.0): Advanced Preemption

- Critical priority threshold (> 95) overrides non-preemptable status
- LLM call cancellation as last resort (controlled via `max_llm_wait_ms`)
- Long-lived task progress checkpointing (per operation, not per time)

**Acceptance Criteria:**
- Critical tasks (priority > 95) preempt any task, including non-preemptable ones
- LLM call cancellation wastes at most 5% of total tokens across all runs
- Long-lived tasks report checkpoint progress for observability

## Why not preemptive?

- Language-level preemption (OS threads) would require Rust or C++. Node.js does not support thread preemption.
- Preemptive interruption of an LLM call wastes tokens — the call must be abandoned mid-way and retried from scratch.
- The 5–15 second latency of LLM calls makes cooperative yield at checkpoint boundaries negligible in practice.
- Checkpointing ensures that preemption does not lose work — the preempted task can resume from its last checkpoint, not from scratch.

## Consequences

**Positive:**
- Simpler implementation (no OS thread management, no signal handlers)
- Compatible with Node.js event loop model
- No wasted tokens from mid-call preemption
- Checkpointing provides durability benefits beyond preemption (crash recovery)

**Negative:**
- A long-running task with no natural checkpoint boundary (e.g. a single model call that takes 60s) cannot be preempted until the call returns
- Starvation is possible if a constant stream of higher-priority tasks prevents a low-priority task from making progress (mitigated by the non-preemptable threshold)
- The preemption mechanism adds complexity to the agent execution loop (checkpoint, yield, resume)

## Related

- [Task Scheduler](../TASK_SCHEDULER.md) — cooperative scheduling algorithm
- [Agent Lifecycle](../AGENT_LIFECYCLE.md) — checkpoint protocol and state serialisation
- [Main AI Kernel](../MAIN_AI_KERNEL.md) — mentions cooperative scheduling in Open Questions
