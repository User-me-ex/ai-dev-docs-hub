# AI Kernel — Detailed Internal Flow

> Zoomed-in view of the Main AI Kernel's internal loop, showing all eight stages, their inputs/outputs, and how each stage publishes to the Shared Context Engine.

## Kernel Loop

```mermaid
flowchart TB
  GOAL([User Goal\n+ ContextRef?]) --> INTAKE

  subgraph INTAKE["① Intake"]
    INT_PARSE[Parse Goal]
    INT_AUTH[Authenticate actor]
    INT_BUDGET[Allocate budget\ntokens / wall-ms / USD]
    INT_SPEC[Emit RunSpec]
    INT_PARSE --> INT_AUTH --> INT_BUDGET --> INT_SPEC
  end

  INT_SPEC --> PLAN

  subgraph PLAN["② Planning Engine"]
    PL_DECOMP[Decompose goal\ninto TaskGraph]
    PL_DEP[Resolve task\ndependencies]
    PL_SCHED[Schedule parallel\nvs. sequential]
    PL_DECOMP --> PL_DEP --> PL_SCHED
  end

  PL_SCHED --> ROUTE

  subgraph ROUTE["③ Nine Router"]
    RT_DISC[Model Discovery\n/models cache]
    RT_POLICY[Routing Policy\nmust-haves + scoring]
    RT_BIND[Emit ModelBinding\nper task role]
    RT_DISC --> RT_POLICY --> RT_BIND
  end

  RT_BIND --> EXEC

  subgraph EXEC["④ Dynamic Workers\nvia AI Group System"]
    EX_SPAWN[Spawn workers\nper group]
    EX_TOOL[Tool dispatch\nNative / MCP / Plugin]
    EX_STREAM[Stream events\nto SCE]
    EX_CHKPT[Checkpoint\nevery N ms]
    EX_SPAWN --> EX_TOOL --> EX_STREAM
    EX_STREAM --> EX_CHKPT
  end

  EX_STREAM --> CRIT

  subgraph CRIT["⑤ Critic"]
    CR_EVAL[Evaluate artifact\nquality + safety]
    CR_VERDICT[Emit Verdict\naccept / reject]
    CR_EVAL --> CR_VERDICT
  end

  CR_VERDICT -->|reject\nreplan_count < MAX| PLAN
  CR_VERDICT -->|accept| MERGE

  subgraph MERGE["⑥ Merge Manager"]
    MG_TXN[Open MergeTxn]
    MG_3WAY[Three-way merge\nstructural awareness]
    MG_CONFLICT[Conflict escalation\nor auto-resolve]
    MG_TXN --> MG_3WAY --> MG_CONFLICT
  end

  MG_CONFLICT --> GUARD

  subgraph GUARD["⑦ Architecture Guardian"]
    GD_RULES[Evaluate rules\nin parallel]
    GD_IMPACT[Impact Analysis\nin parallel]
    GD_VERDICT[Emit Verdict\nok / veto]
    GD_RULES & GD_IMPACT --> GD_VERDICT
  end

  GD_VERDICT -->|veto\nreplan_count < MAX| PLAN
  GD_VERDICT -->|ok| DELIVER

  DELIVER([Result delivered\nto entry surface])

  %% SCE writes (all stages)
  INTAKE & PLAN & ROUTE & EXEC & CRIT & MERGE & GUARD & DELIVER -.->|events| SCE[(Shared\nContext\nEngine)]
  SCE -.-> MEM[(Persistent\nMemory)]
  SCE -.-> AUDIT[(Audit Log)]
```

## Stage Contracts

| Stage | Input type | Output type | Max replans |
|-------|-----------|-------------|-------------|
| Intake | `Goal` | `RunSpec` | — |
| Planning | `RunSpec` \| `ReplanSpec` | `TaskGraph` | 5 |
| Route | `Task` | `ModelBinding` | — |
| Execute | `Task + ModelBinding` | `Artifact[]` | — |
| Critique | `Artifact` | `Verdict` | (triggers replan) |
| Merge | `Verdict[]` | `MergedArtifact` | — |
| Guard | `MergedArtifact` | `Verdict (ok\|veto)` | (triggers replan) |
| Deliver | `Artifact` | `Response` | — |

## Replan Guard Rail

```mermaid
flowchart LR
  REPLAN{replan\ncount} -->|< MAX_REPLANS=5| PLAN[Planning Engine]
  REPLAN -->|>= MAX_REPLANS| ESCALATE[Escalate to human\nmark run failed]
```

## Related Documents

- [Main AI Kernel](../docs/MAIN_AI_KERNEL.md)
- [Planning Engine](../docs/PLANNING_ENGINE.md)
- [Nine Router](../docs/NINE_ROUTER.md)
- [Dynamic Workers](../docs/DYNAMIC_WORKERS.md)
- [Merge Manager](../docs/MERGE_MANAGER.md)
- [Architecture Guardian](../docs/ARCHITECTURE_GUARDIAN.md)
- [Shared Context Engine](../docs/SHARED_CONTEXT_ENGINE.md)
