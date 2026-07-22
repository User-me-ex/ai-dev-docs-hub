# ADR-0002: Cooperative vs Preemptive Scheduling

## Status

Accepted

## Context

The [Task Scheduler](./TASK_SCHEDULER.md) must decide how to handle higher-priority tasks when workers are occupied with lower-priority work. Two approaches exist:

**Cooperative**: Tasks voluntarily yield control at defined checkpoint boundaries. The scheduler cannot forcibly interrupt a running task.

**Preemptive**: The scheduler can forcibly suspend a running task (at any instruction boundary) and assign the worker to a higher-priority task.

AI Dev OS runs agents that make LLM calls, which can take 1–60 seconds. The architecture emphasises checkpointing and budget enforcement.

## Decision

**Cooperative scheduling with checkpoints as preemption points.**

1. The [Agent Lifecycle](AGENT_LIFECYCLE.md) defines explicit checkpoint boundaries (after each tool call, after each model response, every `checkpoint_interval_ms`).
2. The [Task Scheduler](./TASK_SCHEDULER.md) requests preemption at the next checkpoint boundary by sending a `preemption_request` signal via the SCE.
3. The worker completes its current unit of work (tool call or model response), checkpoints its state, and yields.
4. A task that has been preempted more than twice is marked **non-preemptable** — it runs to completion without further interruption to prevent starvation.
5. Preemption is not supported for tasks that hold exclusive resources (file write locks, database transactions). Those tasks declare `preemptable: false` in their task spec.

### Why not preemptive?

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

- [Task Scheduler](./TASK_SCHEDULER.md) — cooperative scheduling algorithm
- [Agent Lifecycle](./AGENT_LIFECYCLE.md) — checkpoint protocol and state serialisation
- [Main AI Kernel](./MAIN_AI_KERNEL.md) — mentions cooperative scheduling in Open Questions
