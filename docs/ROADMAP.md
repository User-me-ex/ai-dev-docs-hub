# Product Roadmap

## Overview

This document outlines the high-level strategic roadmap for AI Dev OS — a multi-agent AI operating system for software development. The roadmap covers three horizons: near-term (0–3 months), mid-term (3–6 months), and long-term (6–12 months), culminating in a v1.0 release.

---

## Current Status

| Phase | Status |
|---|---|
| Phase 0: Foundation | Complete |
| Phase 1: Kernel MVP | In development |

---

## Near-Term (Next 3 Months)

### Phase 1 — Kernel MVP

| Item | Status | Priority | Dependencies |
|---|---|---|---|
| Agent lifecycle management | In progress | P0 | Runtime |
| Task queue and scheduling | In progress | P0 | Agent lifecycle |
| Inter-agent messaging bus | Planned | P1 | Task queue |
| CLI tooling (run, logs, config) | In progress | P0 | — |
| Plugin architecture | Planned | P1 | Kernel core |

### Phase 2 — Memory & Persistence

| Item | Status | Priority | Dependencies |
|---|---|---|---|
| Working memory (short-term) | In progress | P0 | Phase 1 |
| Persistent memory (long-term) | Planned | P0 | Phase 1 |
| Memory indexing and retrieval | Planned | P1 | Persistent memory |

---

## Mid-Term (3–6 Months)

### Phase 3 — AI Groups

| Item | Status | Priority | Dependencies |
|---|---|---|---|
| Group formation protocols | Planned | P1 | Agent lifecycle |
| Role-based agent specialization | Planned | P1 | Group protocols |
| Group-level coordination | Planned | P1 | Phase 2 |

### Phase 4 — Knowledge System

| Item | Status | Priority | Dependencies |
|---|---|---|---|
| Knowledge graph ingestion | Planned | P1 | Phase 2 |
| Semantic search and retrieval | Planned | P1 | Knowledge graph |
| Cross-session knowledge retention | Planned | P1 | Phase 2 |

---

## Long-Term (6–12 Months)

### Phase 5 — Merge & Guardian

| Item | Status | Priority | Dependencies |
|---|---|---|---|
| PR merge automation | Planned | P1 | Phase 3 |
| Guardian (automated review) | Planned | P1 | Phase 3 |
| Conflict resolution | Planned | P2 | Merge |

### Phase 6 — Voice & Multimodal

| Item | Status | Priority | Dependencies |
|---|---|---|---|
| Voice input (speech-to-text) | Planned | P2 | Phase 1 |
| Voice output (text-to-speech) | Planned | P2 | Voice input |
| Image/video understanding | Planned | P2 | Phase 4 |

### Phase 7 — v1.0 Release

| Item | Status | Priority | Dependencies |
|---|---|---|---|
| API stability guarantee | Planned | P0 | All phases |
| Production hardening | Planned | P0 | All phases |
| Documentation and onboarding | Planned | P1 | v1.0 features |

---

## Vision Items

| Item | Status | Priority | Dependencies |
|---|---|---|---|
| Multi-tenant cloud | Backlog | P2 | Phase 7 |
| Agent marketplace | Backlog | P2 | Phase 7, Plugin arch |
| Fine-tuning pipeline | Backlog | P2 | Phase 4 |
| Federated knowledge base | Backlog | P2 | Phase 4 |
| Video input | Backlog | P3 | Phase 6 |

---

## Notes

- **Phase 0** completed all infrastructure scaffolding (repo, CI, build system).
- **Phase 1** is the primary focus; nothing ships before the kernel is stable.
- Strategic priority is **correctness over velocity** — we invest in eval harness, tracing, and observability early.
- Memory (Phase 2) is the highest-risk dependency for all later phases. We plan to prototype working memory alongside Phase 1 to de-risk.
- Vision items are not committed — they will be re-evaluated at each phase boundary.

---

## Related Documents

- [Implementation Roadmap](./IMPLEMENTATION_ROADMAP.md)
- [Project Vision](./PROJECT_VISION.md)
- [Product Overview](./PRODUCT_OVERVIEW.md)
- [PRD](./PRD.md)
