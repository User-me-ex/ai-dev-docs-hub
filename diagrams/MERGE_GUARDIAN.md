# Merge and Guardian Flow

> Detailed sequence of events from a worker submitting changes through three-way merge, impact analysis, Guardian rule evaluation, and final delivery or veto.

## Full Merge + Guardian Sequence

```mermaid
sequenceDiagram
  autonumber
  participant WA  as Worker A
  participant WB  as Worker B (concurrent)
  participant MM  as Merge Manager
  participant IA  as Impact Analysis
  participant GD  as Architecture Guardian
  participant SC  as Shared Context Engine
  participant MEM as Persistent Memory
  participant AUD as Audit Log
  participant K   as Kernel

  WA->>MM: merge.begin(["src/feature.ts"])
  MM-->>WA: txn_id = txn-A

  WB->>MM: merge.begin(["src/feature.ts"])
  MM-->>WB: txn_id = txn-B

  WA->>MM: merge.commit(txn-A, patch_A)
  MM->>MEM: load base_hash(feature.ts)
  MEM-->>MM: base content

  Note over MM: No concurrent commit yet for this path → clean commit

  MM->>MEM: write merged_artifact
  MM->>SC: publish(merge.txns, { txn: txn-A, state: merged_clean })

  WB->>MM: merge.commit(txn-B, patch_B)
  MM->>MEM: load base_hash(feature.ts)
  MEM-->>MM: base content

  Note over MM: Concurrent: txn-A already committed this path

  MM->>MM: 3-way merge(base, head_A, head_B)

  alt Auto-resolvable conflict
    MM->>MM: apply auto-strategy (e.g., append both)
    MM->>SC: publish(merge.txns, { txn: txn-B, state: merged_clean, auto_resolved: true })
  else Genuine conflict
    MM->>SC: publish(merge.txns, { txn: txn-B, state: conflict })
    MM->>K: notify conflict — run surface to UI
    K-->>WB: ConflictRecord (side-by-side diff)
    Note over WB,K: Human or agent resolves conflict
    K->>MM: merge.resolve(conflict_id, accept_B)
    MM->>SC: publish(merge.txns, { resolved_by: human })
  end

  MM->>IA: impact.analyze(merged_patch)
  IA->>SC: graph traversal + history query (parallel)
  IA-->>MM: Impact { risk: 0.4, affected: [...], rationale: "..." }

  MM->>GD: guardian.check(MergedArtifact + Impact)

  par Rule evaluation (parallel)
    GD->>GD: evaluate no-secrets-in-text
    GD->>GD: evaluate no-code-in-doc-repo
    GD->>GD: evaluate correlation-id-required
    GD->>GD: evaluate doc-section-skeleton (warning)
  end

  alt All critical/high rules pass
    GD->>SC: publish(guardian.verdicts, { ok: true, hints: [...] })
    GD->>AUD: append(verdict ok)
    GD-->>MM: Verdict { ok: true }
    MM-->>K: MergedArtifact committed
    K->>SC: publish(run.delivered, Response)
  else Critical/high rule violated
    GD->>SC: publish(guardian.verdicts, { ok: false, violations: [...] })
    GD->>AUD: append(verdict veto)
    GD-->>MM: Verdict { ok: false, violations: [{rule: "no-secrets", message: "..."}] }
    MM-->>K: vetoed
    K->>K: replan (replan_count++)
  end
```

## Three-Way Merge Algorithm Detail

```mermaid
flowchart LR
  BASE[BASE\nContent at base_hash] --> DIFF3[diff3 algorithm]
  HEAD_A[HEAD_A\nWorker A's changes] --> DIFF3
  HEAD_B[HEAD_B\nWorker B's changes] --> DIFF3

  DIFF3 --> SAME[same chunks\n→ output as-is]
  DIFF3 --> A_ONLY[a_only chunks\n→ output HEAD_A]
  DIFF3 --> B_ONLY[b_only chunks\n→ output HEAD_B]
  DIFF3 --> CONFLICT[conflict chunks\n→ auto-strategy?]

  CONFLICT --> AUTO{Auto-resolvable?}
  AUTO -->|append both| MERGE_OUT[output both sections]
  AUTO -->|yaml key merge| MERGE_OUT
  AUTO -->|later timestamp wins| MERGE_OUT
  AUTO -->|no strategy applies| ESCALATE[Escalate to human\nConflictRecord]
  MERGE_OUT --> RESULT[Merged output]
  ESCALATE --> HUMAN[Human resolution\nor timeout → partial]
  HUMAN --> RESULT
```

## Auto-Resolution Strategies

```mermaid
flowchart LR
  CONFLICT_CHUNK[Conflict chunk] --> CHECK1{Both sides\nappend to EOF?}
  CHECK1 -->|yes| APPEND[Append both\norder: A then B]

  CHECK1 -->|no| CHECK2{Different YAML\nfront matter keys?}
  CHECK2 -->|yes| YAML_MERGE[Merge keys\nno collision]

  CHECK2 -->|no| CHECK3{Non-overlapping\nMarkdown sections?}
  CHECK3 -->|yes| MD_APPEND[Append both sections]

  CHECK3 -->|no| CHECK4{Same JSON/TOML key,\ndifferent values?}
  CHECK4 -->|yes| TIMESTAMP[Take later timestamp]

  CHECK4 -->|no| CHECK5{Both add same line?}
  CHECK5 -->|yes| DEDUP[Deduplicate]

  CHECK5 -->|no| ESCALATE2[Escalate to human]
```

## Guardian Rule Evaluation

```mermaid
flowchart LR
  ARTIFACT[MergedArtifact] --> ENGINE[Rule Engine]

  subgraph PARALLEL["Parallel rule evaluation"]
    R1["no-secrets-in-text\ncritical — regex scan"]
    R2["no-code-in-doc-repo\ncritical — file extension check"]
    R3["correlation-id-required\nhigh — event payload check"]
    R4["doc-section-skeleton\nwarning — heading structure"]
    RN["custom rules\n~/.aidevos/rules/"]
  end

  ENGINE --> PARALLEL

  R1 & R2 & R3 -->|violations| VIOLATIONS[Collect violations[]]
  R4 -->|hints| HINTS[Collect hints[]]
  RN -->|violations or hints| VIOLATIONS

  VIOLATIONS --> VERDICT{Any critical\nor high violations?}
  VERDICT -->|yes| VETO["Verdict { ok: false }\nVeto — Kernel replans"]
  VERDICT -->|no| OK["Verdict { ok: true, hints[] }\nOK — deliver with warnings"]
```

## Related Documents

- [Merge Manager](../docs/MERGE_MANAGER.md)
- [Architecture Guardian](../docs/ARCHITECTURE_GUARDIAN.md)
- [Impact Analysis](../docs/IMPACT_ANALYSIS.md)
- [Main AI Kernel](../docs/MAIN_AI_KERNEL.md)
- [Shared Context Engine](../docs/SHARED_CONTEXT_ENGINE.md)
