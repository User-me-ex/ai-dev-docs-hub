# Merge Manager

> Concurrency-safe coordinator for parallel agent edits — three-way merge with structural awareness, full audit trail, and deterministic conflict resolution. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

The Merge Manager (MM) is the subsystem that reconciles concurrent edits produced by parallel [Dynamic Workers](./DYNAMIC_WORKERS.md). In a system where many workers may touch the same file simultaneously — one adding a feature, another fixing a bug, a third updating documentation — a naive last-writer-wins strategy loses work. The Merge Manager applies a three-way merge (base + two heads), escalates genuine conflicts to human review, and presents the merged result to the [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) before delivery.

The Merge Manager is intentionally stateless between transactions: it reads the base from [Persistent Memory](./PERSISTENT_MEMORY.md), applies the merge algorithm, writes the result, and releases. All transaction state lives in the SCE `merge.txns` topic and the [Audit Log](./AUDIT_LOG.md).

## Goals

- Serialise conflicting writes (same file, overlapping regions) without stalling non-conflicting ones (different files or non-overlapping regions).
- Three-way merge with structural awareness: Markdown headings, code blocks, YAML front matter, and JSON/TOML sections are treated as semantic units, not raw lines.
- Full audit: every merge decision (auto-resolved, conflict, escalation) is recorded with the base hash, both heads, the merged result, and the resolution strategy.
- Deterministic: given the same base and two heads, the Merge Manager always produces the same merged result (or the same conflict record).
- Escalate cleanly: genuine conflicts produce a side-by-side diff surfaced to the user with explicit accept/reject actions.

## Non-Goals

- Authoring content — the MM only merges; it does not generate.
- Style formatting — that is the Critic's domain.
- Enforcing architectural rules — that is the [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md)'s job.
- Implementation code — this repository is documentation-only (see [AI Coding Rules](./AI_CODING_RULES.md)).

## Architecture

```mermaid
flowchart LR
  subgraph Workers
    W1[Worker A\nedits file X]
    W2[Worker B\nedits file X]
    W3[Worker C\nedits file Y]
  end

  W1 & W2 & W3 -->|merge.begin(paths[])| MM[Merge Manager]

  subgraph MM internals
    LOCK[Optimistic lock / MVCC]
    BASE[Base fetcher\nPersistent Memory]
    ALGO[Merge algorithm\n3-way / structural]
    CONFLICT[Conflict detector]
    ESCALATE[Human escalation]
    RESULT[Result assembler]
  end

  MM --> LOCK
  LOCK --> BASE
  BASE --> ALGO
  ALGO --> CONFLICT
  CONFLICT -->|auto-resolved| RESULT
  CONFLICT -->|genuine conflict| ESCALATE
  ESCALATE -->|human resolves| RESULT
  RESULT --> MEM[(Persistent Memory)]
  RESULT --> GUARD[Architecture Guardian]
  RESULT --> SCE[(SCE: merge.txns)]
  RESULT --> AUDIT[(Audit Log)]
```

## Merge Protocol

### 1. Begin

A worker calls `merge.begin(paths[])` to declare which files it intends to edit. The MM:
- Creates a `MergeTxn` record with a ULID, the actor identity, and the list of paths.
- Reads the current `content_hash` for each path from [Persistent Memory](./PERSISTENT_MEMORY.md) and records it as the `base_hash`.
- Returns the `txn_id` to the worker.

Two transactions that declare overlapping paths are allowed to proceed concurrently; conflicts are detected at commit time (optimistic concurrency).

### 2. Commit

The worker calls `merge.commit(txn_id, patch)` where `patch` is the set of file-level diffs the worker produced.

The MM:
1. Checks for concurrent transactions that committed the same files since `begin` was called.
2. If none: applies the patch directly, writes to Persistent Memory, marks txn `merged_clean`.
3. If concurrent commits exist: runs the three-way merge algorithm with the stored `base_hash` content as BASE, the concurrent commit as HEAD-A, and the current patch as HEAD-B.

### 3. Three-Way Merge Algorithm

For each file that has concurrent changes:

```
BASE  = content at base_hash
HEAD_A = content committed by concurrent txn
HEAD_B = current patch

chunks = diff3(BASE, HEAD_A, HEAD_B)

for chunk in chunks:
  if chunk.type == "same":
    output += chunk.content
  elif chunk.type == "a_only":
    output += chunk.head_a
  elif chunk.type == "b_only":
    output += chunk.head_b
  elif chunk.type == "conflict":
    strategy = resolve(chunk, file_type)
    if strategy == "auto":
      output += apply_auto_strategy(chunk, file_type)
      record_resolution(chunk, strategy)
    else:
      conflicts.append(chunk)
```

### Auto-Resolution Strategies

The MM applies these strategies automatically for specific conflict patterns:

| Pattern | Strategy | Condition |
|---------|----------|-----------|
| Both sides append to end of file | Append both | No content overlap |
| Both sides add different YAML front matter keys | Merge keys | No key collision |
| Both sides add non-overlapping Markdown sections | Append both sections | Sections have distinct headings |
| Both sides modify the same JSON/TOML key | Take later timestamp | `txn_b.ts > txn_a.ts` |
| Both sides add the same line | Deduplicate | Exact string match |
| One side deletes, other side edits | Keep edit + warn | Default; configurable |

### 4. Conflict Escalation

When a chunk cannot be auto-resolved, the MM:
1. Marks the `MergeTxn` as `conflict`.
2. Builds a `ConflictRecord` with a side-by-side diff (HEAD_A vs HEAD_B, highlighted on BASE).
3. Emits `merge.conflict` on the SCE.
4. Surfaces the conflict in the UI and CLI (`aidevos runs show <run_id> --conflicts`).
5. Waits for the user to accept one side, accept the other, or provide a manual resolution.
6. On resolution: records the choice in the audit log, continues the commit.

Conflicts time out after `conflict_timeout` (default 24 h). An unresolved conflict marks the `MergedArtifact` as `partial: true` with the conflicting sections omitted and a clear user-visible note.

### 5. Guardian

After a successful commit (clean or conflict-resolved), the MM passes the `MergedArtifact` to the [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) for invariant checking before delivery.

## Interfaces

```
# Transaction lifecycle
merge.begin(paths: string[], actor?) → MergeTxn
merge.commit(txn_id, patch: FilePatch[]) → CommitResult
merge.abort(txn_id, reason?) → Ack

# Conflict management
merge.conflicts(run_id?) → ConflictRecord[]
merge.resolve(conflict_id, resolution: Resolution) → Ack

# Introspection
merge.txn(txn_id) → MergeTxn
merge.history(path, limit?) → MergeTxn[]
merge.diff(txn_id) → FilePatch[]
```

### Key types

```
MergeTxn {
  id:           ulid
  actor:        { id, role, worker_id }
  paths:        string[]
  base_hashes:  { [path]: sha256 }
  state:        "open"|"committing"|"merged_clean"|"merged_conflict"|"conflict"|"aborted"
  patch:        FilePatch[]?
  conflicts:    ConflictRecord[]
  verdict:      Verdict?         # from Architecture Guardian
  ts: { created, committed?, resolved?, terminated? }
  correlation_id: uuid
}

CommitResult {
  txn_id:        ulid
  state:         "merged_clean"|"conflict"
  conflicts:     ConflictRecord[]
  artifact_id:   ulid?           # present when state == "merged_clean"
}

ConflictRecord {
  id:            ulid
  txn_id:        ulid
  path:          string
  region: { start_line, end_line }
  base:          string
  head_a:        string
  head_b:        string
  auto_strategies_tried: string[]
  resolution?:   { kind: "accept_a"|"accept_b"|"manual", content?, resolved_by, ts }
}
```

## Requirements

- **MUST** use optimistic concurrency: transactions proceed without locks; conflicts are detected at commit time.
- **MUST** record `base_hash` at `begin` time and verify it at `commit` time to detect stale base.
- **MUST** apply three-way merge for all concurrent edits to the same file.
- **MUST** attempt auto-resolution strategies before escalating to a human.
- **MUST** record every merge decision (auto-resolved and manual) in the [Audit Log](./AUDIT_LOG.md).
- **MUST** pass every committed `MergedArtifact` through the [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) before delivery.
- **MUST** surface conflicts in the UI and CLI within `conflict_surface_ms` (default 500 ms) of detection.
- **MUST** support conflict timeout: unresolved conflicts after `conflict_timeout` produce a `partial` artifact.
- **SHOULD** be structurally aware for Markdown (headings, sections), code (AST-based for supported languages), and YAML/TOML (key-level merging).
- **SHOULD** notify the relevant workers when their transaction is aborted due to an unresolvable conflict.
- **MAY** use a model call to suggest a resolution for semantic conflicts (both sides changed the meaning of a sentence).

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| Unresolvable conflict | No auto-strategy applies | Escalate to human; emit `merge.conflict`; await resolution |
| Conflict timeout | `conflict_timeout` expires | Produce `partial` artifact; notify user; abort conflicting txn |
| Stale base (optimistic conflict) | `base_hash` mismatch at commit | Retry merge with updated base; if still conflicting, escalate |
| Persistent Memory unavailable | Read/write error | Buffer txn; retry with backoff; block new commits until recovered |
| Guardian veto | `Verdict.ok == false` | Return artifact to Kernel as `vetoed`; Kernel replans |
| Actor disconnect during conflict | Heartbeat miss on txn | Mark txn `abandoned`; auto-abort after `abandon_timeout` |

Every failure emits a structured event on the SCE and is recorded in the [Audit Log](./AUDIT_LOG.md).

## Security Considerations

- Every transaction is authenticated: only the originating worker (or an operator) may commit or abort it.
- Merge results are signed by the Merge Manager before being passed to the Guardian (see [Security Model](./SECURITY_MODEL.md)).
- The audit trail for every merge decision is append-only and tamper-evident.
- Manual conflict resolutions by operators are logged with their identity and timestamp.
- See [AuthZ/RBAC](./AUTHZ_RBAC.md) for who may resolve conflicts.

## Observability

| Metric | Labels | Description |
|--------|--------|-------------|
| `merge_txn_total` | `state` | Transactions by terminal state |
| `merge_txn_seconds` | `state` | Transaction duration histogram |
| `merge_conflict_total` | `resolved_by=auto\|human\|timeout` | Conflict outcomes |
| `merge_auto_resolve_total` | `strategy` | Auto-resolution strategy usage |
| `merge_concurrent_txn_count` | — | Concurrent open transactions |
| `merge_file_patch_total` | — | Files patched across all transactions |

Traces: one span per `merge.begin`→`merge.commit`, with child spans for base fetch, algorithm, and guardian handoff. See [Tracing](./TRACING.md).

## Acceptance Criteria

- Two workers concurrently appending non-overlapping sections to the same Markdown file produce a clean merge with both sections present.
- Two workers concurrently editing the same YAML front matter key triggers a conflict, which is surfaced to the user within 500 ms.
- Resolving a conflict by accepting HEAD_A causes the final artifact to match HEAD_A for the conflicting region.
- An unresolved conflict after `conflict_timeout` produces a `partial: true` artifact with the conflicting section explicitly marked and a user-visible note.
- Every merge transaction (clean or conflicted) produces a corresponding entry in the [Audit Log](./AUDIT_LOG.md).

## Open Questions

- Whether to use AST-based merging for code files (e.g., TypeScript, Rust) by default or only when explicitly configured — tracked in [templates/ADR](../templates/ADR.md).
- The appropriate `conflict_timeout` default: 24 h is generous for human review but may be too long for fully-automated pipelines.

## Related Documents

- [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) — receives `MergedArtifact` after commit
- [Impact Analysis](./IMPACT_ANALYSIS.md) — consulted for risk scoring before merge
- [Dynamic Workers](./DYNAMIC_WORKERS.md) — actors that open merge transactions
- [Audit Log](./AUDIT_LOG.md)
- [Persistent Memory](./PERSISTENT_MEMORY.md) — stores base content and merged results
- [System Overview](./SYSTEM_OVERVIEW.md)
- [Main AI Kernel](./MAIN_AI_KERNEL.md)
- [diagrams/MERGE_GUARDIAN](../diagrams/MERGE_GUARDIAN.md)
