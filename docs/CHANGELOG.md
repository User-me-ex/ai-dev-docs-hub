# Changelog

> All notable changes to AI Dev OS are documented here. This file follows the [Keep a Changelog](https://keepachangelog.com/) format and adheres to the versioning scheme defined in [Versioning.md](./VERSIONING.md).

## Categories

Entries are organized under the following headings:

- **Added** — new features, subsystems, documents, or commands.
- **Changed** — modifications to existing functionality, including non-breaking contract changes.
- **Deprecated** — features that are still supported but should not be used in new work.
- **Removed** — features that have been deleted.
- **Fixed** — bug fixes.
- **Security** — vulnerability fixes.

## Version Format

`[MAJOR.MINOR.PATCH]` — `YYYY-MM-DD`

Pre-release versions are listed as `[MAJOR.MINOR.PATCH-rc.N]` and are excluded from the "latest" stability guarantee.

---

## [Unreleased]

### Added

- Phase 1 implementation in progress: Kernel skeleton, Nine Router adapters, CLI MVP.
- Phase 2 implementation in progress: Persistent Memory, Embeddings, Vector Store.

### Changed

- _(No changes yet.)_

### Deprecated

- _(No deprecations yet.)_

### Removed

- _(No removals yet.)_

### Fixed

- _(No fixes yet.)_

### Security

- _(No security fixes yet.)_

---

## [0.1.0] — 2025-07-22

### Added

- **Documentation bootstrap**: 125+ specification documents under `docs/` covering every subsystem from Kernel to Plugin SDK. Each document follows a standardized section skeleton with Mermaid diagrams, acceptance criteria, and cross-references.
- **Spec set**: Complete PRD, TRD, API Spec, System Overview, Security Model, and subsystem specifications for all Phase 1–7 deliverables.
- **Master Prompt v0**: Initial draft of `MASTER_PROMPT` with role definitions for Kernel, Builder, Planner, Critic, Researcher, and Router.
- **Nine Router spec**: Complete model discovery contract with canonical schema, provider endpoint table, role assignment protocol, and fallback strategy.
- **AI Coding Rules v1.0**: Coding standards for the documentation repository including the `no-code-in-doc-repo` rule and section-skeleton Guardian rule.
- **Knowledge bases**: All four KB tier specs — `GLOBAL_KB`, `MAIN_KB`, `GROUP_KB`, `INDIVIDUAL_KB` — with tier resolution rules and retention policies.
- **Template set**: ADR, RFC, PRD, TRD, AgentSpec, PromptSpec, and SubsystemSpec templates under `templates/`.
- **Prompts**: All prompt files shipped — `SYSTEM_PROMPT`, `KERNEL_PROMPT`, `PLANNING_PROMPT`, `ROUTER_PROMPT`, `CRITIC_PROMPT`, `RESEARCH_PROMPT`, `AGENT_PROMPT`, `MASTER_PROMPT`.
- **Mermaid diagrams**: Architecture diagrams for all major subsystems including Kernel loop, Nine Router routing, Shared Context Engine, Merge Manager, and Knowledge System.
- **CLI spec**: Documented command interface for `aidevos init`, `aidevos doctor`, `aidevos run`, `aidevos models`, `aidevos router`, and `aidevos plugins`.
- **Release infrastructure**: Versioning, Release Process, and Changelog documents established.

### Changed

- _(Initial release — no prior version to compare against.)_

### Deprecated

- _(Initial release.)_

### Removed

- _(Initial release.)_

### Fixed

- _(Initial release.)_

### Security

- _(Initial release.)_

---

## [0.2.0] — _Planned 2025-08_

> Release corresponding to Phase 1 — Kernel & Router MVP completion.

### Added (forecast)

- Main AI Kernel skeleton with intake → plan → route → execute → deliver loop.
- Nine Router: OpenAI, Anthropic, and Ollama adapters with model discovery.
- CLI MVP: `aidevos init`, `aidevos doctor`, `aidevos run <goal>`, `aidevos models list`, `aidevos router assign`.
- Shared Context Engine: SQLite backend with publish/subscribe/snapshot.
- Audit Log: append-only structured log backed by SQLite.
- Observability: Prometheus metrics endpoint, structured JSON logs, OpenTelemetry traces.

---

## [0.3.0] — _Planned 2025-09_

> Release corresponding to Phase 2 — Context Engine & Persistent Memory.

### Added (forecast)

- Persistent Memory: relational (SQLite) + vector (usearch) tiers.
- Embeddings: local embedding via Ollama with async backfill queue.
- Vector Store: embedded usearch index with ANN search.
- Agent Memory: per-agent working memory.
- Kernel replay: `kernel.replay(run_id)` for run reconstruction.

---

## [0.4.0] — _Planned 2025-10_

> Release corresponding to Phase 3 — AI Groups & Dynamic Workers.

### Added (forecast)

- AI Group System with spawn, heartbeat, circuit breaker, elastic autoscaler.
- Dynamic Workers with full WorkerSpec execution loop and tool dispatch.
- Task Graph with parallel execution and dependency resolution.
- Planning Engine with goal-to-TaskGraph decomposition.
- Critic role with verdict emission and retry loop.
- Job Scheduler with cron-based scheduling.
- Built-in groups: `code-builder`, `researcher`, `guardian`.

---

## [0.5.0] — _Planned 2025-11_

> Release corresponding to Phase 4 — Knowledge System & Research Engine.

### Added (forecast)

- Knowledge System: four-tier KB with layered query resolution.
- Obsidian Graph Engine with incremental updates and semantic queries.
- RAG Pipeline: hybrid BM25 + ANN retrieval with context assembly.
- Research Engine: fetch → parse → dedup → diff → provenance → write pipeline.
- Web Intelligence: Playwright-based browser automation.
- Internet Search: pluggable backend (SearXNG, Brave, Google).
- GitHub Analysis: repo structure, issue, PR, and release analysis.

---

## [0.6.0] — _Planned 2025-12_

> Release corresponding to Phase 5 — Merge Manager, Guardian & Impact Analysis.

### Added (forecast)

- Merge Manager: three-way merge with Markdown/YAML/JSON awareness.
- Architecture Guardian: rule engine with built-in rules and hot-reload.
- Impact Analysis: graph traversal with risk scoring and blast radius.
- Custom Guardian rule authoring in `~/.aidevos/rules/`.

---

## [0.7.0] — _Planned 2026-01_

> Release corresponding to Phase 6 — Voice System, Plugin SDK & UX.

### Added (forecast)

- Voice System: Whisper STT, Piper TTS, VAD, wake-word, interruption handling.
- Plugin SDK: manifest format, WASM/subprocess execution, consent flow.
- MCP full implementation: client and server with custom resources.
- Desktop shell (Electron or Tauri) and Web shell (React).
- API Gateway: REST + WebSocket with rate limiting and auth.

---

## [1.0.0] — _Planned 2026-02_

> Release corresponding to Phase 7 — General Availability.

### Added (forecast)

- All reliability targets met (p99 Kernel < 5 s, p99 SCE < 5 ms, etc.).
- Full benchmark suite published.
- Security audit completed and findings addressed.
- Multi-tenant deployment documented.
- Migration guide from pre-v1 snapshots.
- API stability guarantee: all v1.0 API surfaces are semver-stable.

---

## Changelog Maintenance Policy

1. Every PR that adds, changes, or removes user-facing behavior MUST include a corresponding changelog entry.
2. Entries are added under the `[Unreleased]` section during development.
3. At release time, `[Unreleased]` is renamed to the version number, a new `[Unreleased]` section is created, and the release date is added.
4. Entries SHOULD reference relevant issue or PR numbers in parentheses.
5. Entries SHOULD link to the relevant documentation for new or changed features.
6. The changelog is maintained by hand — automated generation tools (e.g., `git log` scraping) are not used. This ensures each entry is written for human readers.

## Related Documents

- [Versioning](./VERSIONING.md)
- [Release Process](./RELEASE_PROCESS.md)
- [Implementation Roadmap](./IMPLEMENTATION_ROADMAP.md)
- [Migration Guide](./MIGRATION_GUIDE.md)
- [Upgrade Notes](./UPGRADE_NOTES.md)
