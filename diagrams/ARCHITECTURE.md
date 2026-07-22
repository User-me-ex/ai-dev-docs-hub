# Architecture — High-Level System Diagram

> Complete view of every major subsystem and how they interconnect. See individual subsystem docs for component-level diagrams.

## Top-Level Flowchart

```mermaid
flowchart TB
  subgraph Surfaces["Entry surfaces"]
    CLI[CLI — aidevos]
    DESKTOP[Desktop shell\nTauri / Electron]
    WEB[Web shell\nReact SPA]
    VOICE[Voice input\nSTT wake-word]
    MCP_IN[External MCP client\nIDE / agent]
  end

  subgraph Kernel["Main AI Kernel"]
    INTAKE[Intake]
    PLAN[Planning Engine]
    ROUTE[Nine Router]
    EXEC[Dynamic Workers]
    CRIT[Critic]
    MERGE[Merge Manager]
    GUARD[Architecture Guardian]
    DELIVER[Deliver]
  end

  subgraph Groups["AI Group System"]
    GRP_CODE[Code Builder Group]
    GRP_RES[Research Group]
    GRP_GUARD[Guardian Group]
    GRP_OTHER[Other Groups…]
  end

  subgraph Knowledge["Knowledge Layer"]
    MEM[(Persistent Memory\nSQLite + usearch)]
    GRAPH[(Obsidian Graph Engine)]
    RAG[RAG Pipeline]
    KB_G[(Global KB)]
    KB_M[(Main KB)]
    KB_GRP[(Group KB)]
    KB_IND[(Individual KB)]
  end

  subgraph Providers["Model Providers"]
    LOCAL[Local\nOllama / Whisper / Piper]
    OAI[OpenAI]
    ANT[Anthropic]
    GOO[Google]
    MIS[Mistral]
    CUSTOM[User-registered]
  end

  subgraph Platform["Platform Services"]
    SCE[(Shared Context Engine\nEvent log + snapshot)]
    AUDIT[(Audit Log)]
    QUEUE[Queueing]
    SCHED[Job Scheduler]
    PLUGIN[Plugin SDK]
    MCP_SRV[MCP Server]
    IPC[IPC — Unix socket]
    DB[(Database — SQLite)]
  end

  subgraph Research["Research & Intelligence"]
    RESEARCH[Research Engine]
    WEB_INTEL[Web Intelligence]
    INET[Internet Search]
    GH[GitHub Analysis]
  end

  %% Entry → Kernel
  Surfaces --> INTAKE
  INTAKE --> PLAN
  PLAN --> ROUTE
  ROUTE --> Groups
  Groups --> EXEC
  EXEC --> CRIT
  CRIT -->|reject| PLAN
  CRIT -->|accept| MERGE
  MERGE --> GUARD
  GUARD -->|veto| PLAN
  GUARD -->|ok| DELIVER
  DELIVER --> Surfaces

  %% Kernel ↔ SCE (all stages)
  INTAKE & PLAN & ROUTE & EXEC & CRIT & MERGE & GUARD --> SCE

  %% Workers ↔ Providers
  EXEC --> Providers
  ROUTE --> Providers

  %% Knowledge access
  EXEC --> Knowledge
  PLAN --> MEM
  RESEARCH --> MEM
  RESEARCH --> GRAPH
  GRAPH --> RAG
  RAG --> EXEC

  KB_G --> KB_M --> KB_GRP --> KB_IND

  %% Platform wiring
  SCE --> AUDIT
  SCE --> DB
  QUEUE --> EXEC
  SCHED --> QUEUE
  SCHED --> RESEARCH
  PLUGIN --> EXEC
  MCP_SRV --> EXEC
  MCP_IN --> MCP_SRV
  IPC --> INTAKE

  %% Research sources
  RESEARCH --> WEB_INTEL
  RESEARCH --> INET
  RESEARCH --> GH
```

## Subsystem Relationship Summary

| Subsystem | Calls | Called by |
|-----------|-------|-----------|
| Main AI Kernel | Nine Router, Planning Engine, AI Group System, Merge Manager, Architecture Guardian | All entry surfaces |
| Nine Router | Model Providers, Persistent Memory (cache) | Kernel, Dynamic Workers |
| AI Group System | Dynamic Workers, Nine Router, Knowledge Layer | Kernel |
| Dynamic Workers | Model Providers, Tool Calling, MCP, Plugin SDK | AI Group System |
| Merge Manager | Architecture Guardian, Impact Analysis, Persistent Memory | Kernel |
| Architecture Guardian | Impact Analysis, Audit Log | Kernel, Merge Manager |
| Shared Context Engine | Database, Audit Log | Every subsystem |
| Persistent Memory | Vector Store, Embeddings, Database | Memory clients |
| Research Engine | Web Intelligence, Internet Search, GitHub Analysis, Persistent Memory | Job Scheduler, Kernel |
| Obsidian Graph Engine | Persistent Memory, Vector Store | RAG Pipeline, Research Engine, MCP |

## Related Documents

- [System Overview](../docs/SYSTEM_OVERVIEW.md)
- [Main AI Kernel](../docs/MAIN_AI_KERNEL.md)
- [Nine Router](../docs/NINE_ROUTER.md)
- [AI Groups](../docs/AI_GROUPS.md)
- [Shared Context Engine](../docs/SHARED_CONTEXT_ENGINE.md)
