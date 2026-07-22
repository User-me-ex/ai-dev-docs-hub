# ADRs

> Architecture Decision Records for AI Dev OS. Each ADR documents a significant architectural decision, its context, alternatives considered, and consequences.

## Index

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-0001](./0001-runtime-stack.md) | Runtime Stack Decision (TypeScript + Rust) | Accepted |
| [ADR-0002](./0002-cooperative-scheduling.md) | Cooperative vs Preemptive Scheduling | Accepted |
| [ADR-0003](./0003-auto-fix-consent.md) | Auto-Fix Consent Model | Proposed |
| [ADR-0004](./0004-role-override-cascading.md) | Role Override Cascading (Flat Model) | Accepted |
| [ADR-0005](./0005-discovery-interval.md) | Model Discovery Auto-Refresh Interval | Accepted |
| [ADR-0006](./0006-pick-for-me.md) | "Pick for Me" Implementation Approach | Proposed |

## Creating a New ADR

1. Copy the template from [`templates/ADR.md`](../../templates/ADR.md).
2. Choose the next available number.
3. Fill in: Title, Status, Context, Decision, Consequences, Related.
4. Add the file to this index.

## ADR Lifecycle

| Status | Meaning |
|--------|---------|
| **Proposed** | Decision is under discussion; not yet final. |
| **Accepted** | Decision has been approved and is in effect. |
| **Deprecated** | Decision has been superseded by a later ADR. |
| **Superseded** | This ADR has been replaced by [ADR-NNNN](./NNNN-title.md). |
