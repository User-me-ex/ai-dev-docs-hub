# Model Discovery Sequence

> Sequence diagram of the Nine Router performing model discovery across multiple providers.

```mermaid
sequenceDiagram
    participant UI as User / CLI
    participant NR as Nine Router
    participant RM as Role Manager
    participant CACHE as Model Cache
    participant OAI as OpenAI Adapter
    participant ANT as Anthropic Adapter
    participant OLL as Ollama Adapter
    participant SCE as Shared Context Engine

    UI->>NR: router.refresh_all()

    NR->>CACHE: invalidate(all)
    NR->>SCE: publish("models.discovery_started")

    par OpenAI
        NR->>OAI: discover()
        OAI->>OAI: GET /v1/models
        OAI-->>NR: Model[]
    and Anthropic
        NR->>ANT: discover()
        ANT->>ANT: GET /v1/models
        ANT-->>NR: Model[]
    and Ollama
        NR->>OLL: discover()
        OLL->>OLL: GET /api/tags
        OLL-->>NR: Model[]
    end

    NR->>NR: normalise, deduplicate, group
    NR->>CACHE: store(catalog)
    NR->>SCE: publish("models.discovery_complete", {catalog_summary})

    UI->>NR: router.roles()
    NR->>RM: resolve role assignments
    RM-->>NR: RoleAssignment[]
    NR-->>UI: NineRole[] with current binding

    UI->>NR: router.assign("builder", "gpt-4o", scope="workspace")
    NR->>NR: validate model exists in cache
    NR->>RM: assign("builder", "gpt-4o", scope="workspace")
    RM->>SCE: publish("router.assignments", {role, model_id, scope, actor, ts})
    SCE-->>UI: assignment confirmation
```

## Related Documents

- [Nine Router](../docs/NINE_ROUTER.md) — model discovery and role assignment
- [Model Discovery](../docs/MODEL_DISCOVERY.md) — adapter specifications
- [Model Providers](../docs/MODEL_PROVIDERS.md) — provider integration details
