# Merge Guardian Sequence

> Sequence diagram of the Merge Manager and Architecture Guardian evaluation flow.

```mermaid
sequenceDiagram
    participant DW1 as Dynamic Worker A
    participant DW2 as Dynamic Worker B
    participant MM as Merge Manager
    participant MEM as Persistent Memory
    participant SCE as Shared Context Engine
    participant IA as Impact Analysis
    participant AG as Architecture Guardian

    DW1->>MM: submit_artifact(file_a)
    DW2->>MM: submit_artifact(file_b)
    MM->>MEM: read_base(file)
    MEM-->>MM: base_content

    Note over MM: Three-way merge
    MM->>MM: diff(base, file_a) → delta_a
    MM->>MM: diff(base, file_b) → delta_b
    MM->>MM: merge(delta_a, delta_b) → merged

    alt Clean merge
        MM->>SCE: publish("merge.completed", {clean: true})
    else Conflict detected
        MM->>SCE: publish("merge.conflict", {conflicts: [...]})
        MM->>MM: escalate to human review
        Note over MM: Waiting for human resolution
    end

    MM->>MEM: stage(merged, status="pending_guardian")

    par Guardian evaluation
        AG->>AG: load rules (14 built-in)
        AG->>IA: analyze(merged_artifact)
        IA->>IA: graph traversal, risk scoring
        IA-->>AG: Impact{risk_score, blast_radius}
    and Rule evaluation
        AG->>AG: evaluate critical rules (parallel)
        AG->>AG: evaluate high rules (parallel)
        AG->>AG: evaluate warning rules (parallel)
    end

    alt All rules pass
        AG->>SCE: publish("guardian.verdict", {ok: true})
        AG-->>MM: Verdict{ok: true}
        MM->>MEM: commit(merged)
        MM->>SCE: publish("merge.committed", {artifact_id})
    else Violations found
        AG->>SCE: publish("guardian.verdict", {ok: false, violations: [...]})
        AG-->>MM: Verdict{ok: false, violations}
        MM->>MEM: unstage(merged)
        MM->>SCE: publish("merge.rejected", {artifact_id, violations})
    end
```

## Related Documents

- [Merge Manager](../docs/MERGE_MANAGER.md) — three-way merge algorithm
- [Architecture Guardian](../docs/ARCHITECTURE_GUARDIAN.md) — rule evaluation and veto
- [Impact Analysis](../docs/IMPACT_ANALYSIS.md) — risk assessment
- [diagrams/MERGE_GUARDIAN](./MERGE_GUARDIAN.md) — flowchart diagram
