# Nine Router Flow — Model Discovery, Grouping, and Role Assignment

> Complete flow from discovery trigger through to a resolved ModelBinding delivered to the Kernel.

## Model Discovery Pipeline

```mermaid
flowchart TB
  subgraph Triggers
    UI_BTN[UI: Refresh button]
    CRON_JOB[Cron: every 10 min]
    CRED_CHG[Credential change event]
    CLI_CMD[CLI: aidevos models refresh]
  end

  Triggers --> DISP[Discovery Dispatcher]

  subgraph Adapters["Provider Adapters (all run in parallel)"]
    OLL[Ollama\nGET /api/tags\nlocalhost:11434]
    LLC[llama.cpp\nGET /v1/models\nlocalhost:8080]
    MLX[MLX\nFilesystem scan]
    OAI[OpenAI\nGET /v1/models]
    ANT[Anthropic\nGET /v1/models]
    GOO[Google\nGET /v1beta/models]
    MIS[Mistral\nGET /v1/models]
    MCP_A[MCP servers\ntools/list]
    USR[User-registered\ncustom base URL]
  end

  DISP --> Adapters

  subgraph Normalise["Normalization + Deduplication"]
    NORM[Schema normalizer\n→ canonical Model\{\}]
    ALIAS[Alias resolver\ngpt-4o vs gpt-4o-2024-08-06]
    SORT[Sort: family ASC, deprecated ASC,\ndisplay_name ASC]
  end

  Adapters -->|ok: raw models[]| NORM
  Adapters -->|error| ERR_BADGE[Mark provider degraded\nerror badge in UI]

  NORM --> ALIAS --> SORT

  SORT --> CACHE[(TTL Cache\n10 min per provider)]
  SORT --> SCE_BUS[(SCE: models.discovery\nDiscoveryReport)]

  CACHE --> UI_PANEL[Router UI Panel]
  SCE_BUS --> COST[Cost Management]
  SCE_BUS --> CLI_OUT[CLI output]
```

## Provider Grouping (UI Render Order)

```mermaid
flowchart LR
  CATALOG[Full model catalog] --> G1["① Local\nOllama · llama.cpp · MLX"]
  CATALOG --> G2["② OpenAI\ngpt-4o · o1 · text-embedding-3-*"]
  CATALOG --> G3["③ Anthropic\nclaude-3-5-* · claude-opus-4-5"]
  CATALOG --> G4["④ Google\ngemini-2.5-pro · gemini-2.5-flash"]
  CATALOG --> G5["⑤ Mistral\nmistral-large · codestral"]
  CATALOG --> G6["⑥ MCP-exposed\ncatalog from MCP server"]
  CATALOG --> G7["⑦ User-registered\ncustom providers"]

  G1 & G2 & G3 & G4 & G5 & G6 & G7 --> PANEL[Nine Router UI\nsearch · filter · assign]
```

## Role Assignment Flow

```mermaid
flowchart LR
  PANEL[Router UI\nor CLI assign] --> ASSIGN["router.assign(role, model_id, scope)"]

  ASSIGN --> VALID{Model in cache?}
  VALID -->|no| ERROR[Return MODEL_NOT_FOUND]
  VALID -->|yes| WRITE[Write RoleAssignment\nto DB]

  WRITE --> SCE_RA[(SCE: router.assignments\nRoleAssignment event)]
  WRITE --> POLICY[Update routing policy\nfor role]

  SCE_RA --> KERNEL[Kernel subscription\nnext run uses new binding]
  SCE_RA --> COST_MGR[Cost Management\nupdate cost forecast]
  SCE_RA --> CLI_SHOW[CLI: aidevos router show\nreflects immediately]
```

## Fallback Resolution (at task-start time)

```mermaid
flowchart TD
  TASK[Task assigned\nrole=builder] --> BIND["router.binding(role)"]

  BIND --> PRIMARY[Try primary model\ne.g. openai/gpt-4o]
  PRIMARY -->|ok| USE[Worker uses model]
  PRIMARY -->|error 5xx / timeout| FB1

  FB1[Try fallback 1\ne.g. anthropic/claude-3-5-sonnet]
  FB1 -->|ok| USE
  FB1 -->|error| FB2

  FB2[Try fallback N\ne.g. ollama/llama3.1:8b]
  FB2 -->|ok| USE
  FB2 -->|all exhausted| EXHAUST[ExhaustedFallbacks\nKernel marks task failed]

  USE --> SCE_FB[(SCE: router.fallback event\nif any fallback was used)]
```

## The Nine Roles — Visual Summary

```mermaid
flowchart LR
  subgraph NINE["Nine Canonical Roles"]
    R1[1. Kernel\nOrchestrator]
    R2[2. Planner\nDecomposes goals]
    R3[3. Router\nSelects models]
    R4[4. Researcher\nRetrieval + synthesis]
    R5[5. Builder\nCode generation]
    R6[6. Critic\nQuality review]
    R7[7. Merger\nConcurrent edit reconciler]
    R8[8. Guardian\nInvariant enforcer]
    R9[9. Voice\nSTT / TTS]
  end

  WORKSPACE[Workspace-level\nassignment] --> NINE
  PROJECT[Project-level\noverride] --> NINE
  GROUP[Group-level\noverride] --> NINE
```

## Related Documents

- [Nine Router](../docs/NINE_ROUTER.md)
- [Model Discovery](../docs/MODEL_DISCOVERY.md)
- [Model Providers](../docs/MODEL_PROVIDERS.md)
- [Model Routing Policy](../docs/MODEL_ROUTING_POLICY.md)
- [Main AI Kernel](../docs/MAIN_AI_KERNEL.md)
