# Model Routing Policy Flow

> How the Nine Router resolves a model for a given role — capability filtering, scoring, health checking, and fallback.

## Full Routing Policy Flow

```mermaid
flowchart TB
  ROLE_REQ["role request\ne.g. binding('builder')"] --> LOAD_ASSIGN["Load RoleAssignment\nfrom DB / cache"]

  LOAD_ASSIGN --> HAS_ASSIGN{Assignment\nexists?}
  HAS_ASSIGN -->|no| ROLE_ERROR["Return ROLE_NOT_ASSIGNED\nKernel refuses to start run"]
  HAS_ASSIGN -->|yes| PRIMARY["Primary model\nfrom RoleAssignment"]

  PRIMARY --> HEALTH_P{Provider\nhealthy?}
  HEALTH_P -->|yes| BIND["ModelBinding {\nprimary, fallbacks[],\nsnapshot_ts\n}"]
  HEALTH_P -->|no: degraded / auth_error| FALLBACK_CHAIN

  subgraph FALLBACK_CHAIN["Fallback chain evaluation (at task execution time)"]
    FB1[Try primary model]
    FB1 -->|ok| USE[Worker uses model]
    FB1 -->|5xx / timeout| FB2[Try fallback 1]
    FB2 -->|ok| USE
    FB2 -->|error| FB3[Try fallback 2]
    FB3 -->|ok| USE
    FB3 -->|all exhausted| EXHAUST["ExhaustedFallbacks\nTask fails"]
    USE --> SCE_FB[(SCE: router.fallback\nif any fallback used)]
  end

  BIND --> FALLBACK_CHAIN
```

## Routing Policy Rules

```mermaid
flowchart LR
  CANDIDATES["All discovered models\nfor the role's provider scope"] --> MUST_FILTER

  subgraph MUST_FILTER["① Must-have capability filter"]
    TOOL_CHK{requires tools?}
    VIS_CHK{requires vision?}
    AUD_CHK{requires audio?}
    CTX_CHK{requires context\n> N tokens?}
  end

  MUST_FILTER -->|fails any must-have| DROP[Drop model]
  MUST_FILTER -->|all pass| SCORE

  subgraph SCORE["② Score surviving candidates"]
    CAP_SCORE["capability score\n+5 per extra capability beyond must-haves"]
    CTX_SCORE["context score\nnormalised context_window / max_context"]
    LAT_SCORE["latency score\nestimated from provider class"]
    COST_SCORE["cost score\n1 / (input_price_per_M + output_price_per_M)"]
    TOTAL_SCORE["total = 0.40*capability\n+ 0.25*context\n+ 0.20*latency\n+ 0.15*cost"]
  end

  SCORE --> RANK[Rank candidates\nby total score DESC]
  RANK --> PICK_TOP[Pick top = primary\nnext N = fallbacks]
```

## Policy Override Hierarchy

```mermaid
flowchart LR
  WORKSPACE_POLICY[Workspace-level\ndefault policy] --> RESOLVE
  PROJECT_POLICY["Project-level\noverride (if set)"] --> RESOLVE
  GROUP_POLICY["Group-level\noverride (if set)"] --> RESOLVE

  RESOLVE["Resolved policy\nnarrowest scope wins"] --> APPLY["Apply policy\nto role binding"]
```

## Provider Health States

```mermaid
stateDiagram-v2
  [*] --> Healthy : discovery succeeds + HTTP 200
  Healthy --> Degraded : error rate > 20% / 60s\nor HTTP 5xx
  Healthy --> AuthError : HTTP 401
  Healthy --> RateLimited : HTTP 429
  Degraded --> Healthy : error rate < 5% / 60s
  Degraded --> Offline : all requests fail for 120s
  AuthError --> Healthy : credentials updated
  RateLimited --> Healthy : after Retry-After window
  Offline --> Degraded : at least one success
```

## Capability Taxonomy

```mermaid
flowchart LR
  subgraph Capabilities["Model Capability Flags"]
    TOOLS[tools\nFunction calling / tool use]
    VISION[vision\nImage input]
    AUDIO[audio\nAudio input and output]
    JSON[json_mode\nGuaranteed JSON output]
    STREAM[streaming\nToken streaming]
    EMBED[embeddings\nVector representation]
    FT[fine_tune\nFine-tuning supported]
  end
```

## Related Documents

- [Model Routing Policy](../docs/MODEL_ROUTING_POLICY.md)
- [Nine Router](../docs/NINE_ROUTER.md)
- [Model Discovery](../docs/MODEL_DISCOVERY.md)
- [Model Providers](../docs/MODEL_PROVIDERS.md)
- [Dynamic Workers](../docs/DYNAMIC_WORKERS.md)
- [Main AI Kernel](../docs/MAIN_AI_KERNEL.md)
