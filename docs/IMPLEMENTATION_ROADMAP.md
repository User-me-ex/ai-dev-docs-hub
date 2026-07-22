# Implementation Roadmap

> Phased plan from bootstrap to v1.0.

## Phase 0 — Documentation Bootstrap (this repo)

- Full spec set under `docs/`, `prompts/`, `diagrams/`, `templates/`.
- Master Prompt v0 drafted and reviewed.
- Nine Router spec finalized with `/models` discovery contract.

## Phase 1 — Kernel & Router MVP

- Main AI Kernel skeleton with role dispatch.
- Nine Router with OpenAI, Anthropic, and Ollama providers.
- CLI shell with `router list|refresh|assign`.

## Phase 2 — Context & Memory

- Shared Context Engine with pub/sub.
- Persistent Memory (episodic + semantic).
- Obsidian Graph Engine read/write.

## Phase 3 — Groups & Workers

- AI Group System with Planner/Builder/Critic default group.
- Dynamic Workers with autoscaling.
- Task Graph + Job Scheduler.

## Phase 4 — Knowledge & Research

- Four-tier KBs (Global/Main/Group/Individual).
- Research Engine with web + GitHub sources.
- RAG pipeline over the graph.

## Phase 5 — Merge, Guardian, Impact

- Merge Manager for concurrent agent edits.
- Architecture Guardian enforcing invariants.
- Impact Analysis pre-flight for all edits.

## Phase 6 — Voice, Plugins, UX Polish

- Voice System (STT/TTS/wake-word).
- Plugin SDK GA.
- Desktop shell + web shell parity.

## Phase 7 — v1.0

- Reliability targets met (see [Reliability](./RELIABILITY.md)).
- Eval Harness passing on all default prompts.
- Docs, changelog, migration guides complete.

## Related Documents

- [PRD](./PRD.md)
- [TRD](./TRD.md)
- [Roadmap](./ROADMAP.md)
- [Release Process](./RELEASE_PROCESS.md)
