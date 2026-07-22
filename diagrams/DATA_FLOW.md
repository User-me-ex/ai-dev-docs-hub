# Data Flow — End-to-End Request Lifecycle

> Sequence diagram tracing a user goal from submission through delivery, showing every major subsystem interaction, with timing notes, event catalog, and failure handling.

## Full Request Lifecycle

```mermaid
sequenceDiagram
  autonumber
  participant U  as User
  participant SH as Shell (CLI/Web/Desktop)
  participant K  as Main AI Kernel
  participant PL as Planning Engine
  participant RT as Nine Router
  participant SC as Shared Context Engine
  participant AG as AI Group System
  participant W  as Dynamic Worker
  participant P  as Model Provider
  participant T  as Tool (MCP/Plugin/Native)
  participant CR as Critic
  participant MM as Merge Manager
  participant IA as Impact Analysis
  participant GD as Architecture Guardian
  participant MEM as Persistent Memory
  participant AUD as Audit Log

  U->>SH: Submit goal "write feature X"
  SH->>K: kernel.submit(goal)
  K-->>SH: run_id

  K->>SC: publish(run.intake, RunSpec)
  SC->>AUD: append(run.intake)

  K->>PL: plan(RunSpec)
  PL->>MEM: memory.query("similar past tasks")
  MEM-->>PL: MemoryRecord[]
  PL-->>K: TaskGraph { tasks: [A, B, C], deps: A→B, A→C }
  K->>SC: publish(run.planned, TaskGraph)

  K->>RT: binding("builder")
  RT-->>K: ModelBinding { primary: gpt-4o, fallbacks: [...] }
  K->>SC: publish(run.routed, ModelBinding)

  K->>AG: spawn_group("code-builder", task_A)
  AG->>W: WorkerSpec { role: builder, binding, tools, budget }

  W->>P: stream(prompt, tools)
  P-->>W: token stream
  W->>SC: publish(worker.token, ...)  [continuous]

  W->>T: file_read("src/feature.ts")
  T-->>W: file content
  W->>SC: publish(worker.tool_call, { name: file_read, ... })

  W->>P: continue stream with file content
  P-->>W: completion

  W->>SC: publish(worker.completed, Artifact)
  W->>MEM: memory.write({ kind: artifact, text: ... })

  W->>CR: submit artifact for critique
  CR->>P: evaluate(artifact)
  P-->>CR: evaluation result
  CR-->>K: Verdict { ok: true }
  K->>SC: publish(run.critiqued, Verdict)

  K->>MM: merge.begin(["src/feature.ts"])
  MM->>IA: impact.analyze(patch)
  IA-->>MM: Impact { risk: 0.3, affected: [...] }
  MM->>GD: guardian.check(MergedArtifact)
  GD->>SC: publish(guardian.verdicts, Verdict)
  GD->>AUD: append(guardian.verdict)
  GD-->>MM: Verdict { ok: true }
  MM->>MEM: write(merged_artifact)
  MM-->>K: committed

  K->>SC: publish(run.delivered, Response)
  K-->>SH: Response
  SH-->>U: Display result
```

## Parallel Task Execution

```mermaid
sequenceDiagram
  participant K as Kernel
  participant AG as AI Group System
  participant WA as Worker A (task B)
  participant WB as Worker B (task C)
  participant MM as Merge Manager

  Note over K,MM: Tasks B and C are independent — run in parallel

  K->>AG: spawn_group(task_B)
  K->>AG: spawn_group(task_C)

  AG->>WA: WorkerSpec (task B)
  AG->>WB: WorkerSpec (task C)

  WA->>WA: execute task B
  WB->>WB: execute task C

  WA-->>MM: merge.begin(["file1.ts"])
  WB-->>MM: merge.begin(["file2.ts"])

  Note over MM: Different files — no conflict

  MM-->>WA: txn committed
  MM-->>WB: txn committed

  WA->>K: task_B completed
  WB->>K: task_C completed
```

## Model Fallback Flow

```mermaid
sequenceDiagram
  participant W  as Worker
  participant P1 as Primary Model (gpt-4o)
  participant P2 as Fallback 1 (claude-3-5-sonnet)
  participant P3 as Fallback 2 (ollama/llama3.1:8b)
  participant SC as SCE

  W->>P1: invoke(prompt)
  P1-->>W: HTTP 503 Service Unavailable
  W->>SC: publish(router.fallback, { from: gpt-4o, reason: 503 })

  W->>P2: invoke(prompt)
  P2-->>W: HTTP 429 Too Many Requests
  W->>SC: publish(router.fallback, { from: claude-3-5-sonnet, reason: 429 })

  W->>P3: invoke(prompt)
  P3-->>W: Completion stream
  W->>SC: publish(worker.completed, Artifact)
```

## Timing and Performance Notes

| Step | Operation | Expected Duration | Notes |
|------|-----------|-------------------|-------|
| 1–2 | Submit goal to Kernel | < 10ms | IPC or in-process call |
| 3–4 | Intake + audit | < 100ms | Auth check, budget allocation |
| 5–8 | Plan + memory query | 1–10s | Depends on goal complexity, memory size |
| 9–10 | Route (model binding) | < 50ms | Cache hit (primary path) |
| 11–12 | Spawn worker | < 50ms | Warm pool hit; 500ms-2s cold start |
| 13–24 | Execute (model + tools) | 5–120s | Dominant factor; depends on task size |
| 25–30 | Critique | 2–10s | Model evaluation of artifact |
| 31–38 | Merge + guard | < 5s | Three-way merge + rule evaluation |
| 39–41 | Deliver | < 50ms | Result formatted and returned |

**End-to-end typically**: 10–150s depending on task complexity and model speed.

## Event Catalog Summary

| Event | Publisher | Subscribers | Frequency |
|-------|-----------|-------------|-----------|
| `run.intake` | Kernel | Audit Log, CLI | 1 per run |
| `run.planned` | Kernel | Kernel (self) | 1 per plan |
| `run.routed` | Kernel | Cost Management | 1 per task |
| `worker.token` | Worker | CLI (streaming) | Many per task |
| `worker.tool_call` | Worker | Audit Log, CLI | Per tool call |
| `worker.completed` | Worker | Kernel, Cost Management | 1 per task |
| `run.critiqued` | Kernel | Audit Log | 1 per artifact |
| `guardian.verdicts` | Guardian | Audit Log, CLI | 1 per merge |
| `run.delivered` | Kernel | CLI, Cost Management | 1 per run |

## Performance Characteristics

| Step | Operation | Typical | P99 | Notes |
|------|-----------|---------|-----|-------|
| 1–2 | Submit goal | < 10ms | 50ms | IPC or in-process call |
| 3–4 | Intake + audit | < 100ms | 500ms | Auth + budget allocation |
| 5–8 | Plan + memory query | 2s | 15s | Model call for decomposition |
| 9–10 | Route binding | < 50ms | 200ms | Cache hit |
| 11–12 | Spawn worker | 50ms | 2s | Warm vs cold pool |
| 13–24 | Execute (model + tools) | 15s | 120s | Dominant factor |
| 25–30 | Critique | 3s | 15s | Model evaluation |
| 31–38 | Merge + guard | 2s | 10s | Merge + rule eval |
| 39–41 | Deliver | < 50ms | 200ms | Format result |

**End-to-end**: typically 20–150s. The execute stage dominates (> 80% of total time).

## Configuration Limits Impacting Data Flow

| Setting | Default | Impact |
|---------|---------|--------|
| `kernel.max_concurrent_runs` | 5 | Max parallel run submissions |
| `kernel.max_workers` | 10 | Max concurrent worker goroutines |
| `kernel.stage.timeout` | 300s | Per-stage timeout before fallback |
| `kernel.budget.wall_ms_max` | 300000 | Max wall-clock time per run |
| `kernel.budget.tokens_max` | 100000 | Max tokens per run |

## Related Documents

- [Main AI Kernel](../docs/MAIN_AI_KERNEL.md)
- [Dynamic Workers](../docs/DYNAMIC_WORKERS.md)
- [Nine Router](../docs/NINE_ROUTER.md)
- [Merge Manager](../docs/MERGE_MANAGER.md)
- [Architecture Guardian](../docs/ARCHITECTURE_GUARDIAN.md)
- [Shared Context Engine](../docs/SHARED_CONTEXT_ENGINE.md)
- [Planning Engine](../docs/PLANNING_ENGINE.md)
