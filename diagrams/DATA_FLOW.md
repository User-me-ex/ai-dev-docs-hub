# Data Flow — End-to-End Request Lifecycle

> Sequence diagram tracing a user goal from submission through delivery, showing every major subsystem interaction.

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

## Related Documents

- [Main AI Kernel](../docs/MAIN_AI_KERNEL.md)
- [Dynamic Workers](../docs/DYNAMIC_WORKERS.md)
- [Nine Router](../docs/NINE_ROUTER.md)
- [Merge Manager](../docs/MERGE_MANAGER.md)
- [Architecture Guardian](../docs/ARCHITECTURE_GUARDIAN.md)
- [Shared Context Engine](../docs/SHARED_CONTEXT_ENGINE.md)
