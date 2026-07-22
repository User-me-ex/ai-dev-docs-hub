# Implementation Roadmap

> Phased plan from documentation bootstrap to v1.0 general availability. Each phase builds on the previous; no phase is considered complete until its acceptance criteria are met and documented in the [Eval Harness](./EVAL_HARNESS.md).

## Guiding Principles

1. **Spec before code**: Every subsystem ships as a completed doc before its first line of implementation code.
2. **Local-first at every phase**: Each phase must be fully functional offline; cloud integrations are enhancements, not dependencies.
3. **Acceptance criteria are phase gates**: A phase is not done until its acceptance criteria pass in the Eval Harness.
4. **Observability is not optional**: Every phase ships with metrics, traces, and logs from day one.
5. **No heroics**: Phases are sized for steady, reviewable progress. If a phase grows beyond 6 weeks, split it.

---

## Phase 0 — Documentation Bootstrap ✓ (current phase)

**Theme**: Establish the complete specification set so every subsequent phase has unambiguous contracts to implement against.

### Deliverables

- Full spec set under `docs/`, `prompts/`, `diagrams/`, `templates/` — 125+ documents.
- Standard section skeleton enforced across all docs (Overview → Goals → Non-Goals → Requirements → Architecture → Interfaces → Data Model → Failure Modes → Security → Observability → Acceptance Criteria → Open Questions → Related Documents).
- Mermaid diagrams for all major subsystems.
- Master Prompt v0 drafted and reviewed.
- Nine Router spec finalized with complete `/models` discovery contract, canonical schema, and provider endpoint table.
- AI Coding Rules v1.0 finalised.
- All four KB tier specs (`GLOBAL_KB`, `MAIN_KB`, `GROUP_KB`, `INDIVIDUAL_KB`) complete.
- Template set: ADR, RFC, PRD, TRD, AgentSpec, PromptSpec, SubsystemSpec.
- All prompt files: `SYSTEM_PROMPT`, `KERNEL_PROMPT`, `PLANNING_PROMPT`, `ROUTER_PROMPT`, `CRITIC_PROMPT`, `RESEARCH_PROMPT`, `AGENT_PROMPT`, `MASTER_PROMPT`.

### Acceptance criteria

- [ ] Every doc in `docs/` passes the `doc-section-skeleton` Guardian rule.
- [ ] Every doc's Related Documents section contains valid Markdown links.
- [ ] All Mermaid blocks render without parse errors.
- [ ] No document contains executable code (enforced by `no-code-in-doc-repo` rule).

---

## Phase 1 — Kernel & Router MVP

**Theme**: The minimal vertical slice — a user can submit a goal via CLI, the Kernel routes it to a model, and the result comes back. No AI Groups, no memory, no Guardian.

### Deliverables

- **Main AI Kernel skeleton**: intake → plan (flat, single-task) → route → execute (single worker) → deliver loop.
- **Nine Router**: OpenAI, Anthropic, and Ollama adapters; model discovery; role assignment for Kernel, Builder, Critic; CLI-accessible.
- **CLI MVP**: `aidevos init`, `aidevos doctor`, `aidevos run <goal>`, `aidevos models list|refresh|show`, `aidevos router roles|assign|show`.
- **Shared Context Engine**: SQLite backend; `publish`, `subscribe`, `snapshot`; per-topic FIFO ordering.
- **Audit Log**: append-only structured log backed by SQLite; every Kernel event written.
- **Observability**: Prometheus metrics endpoint; structured JSON logs; basic OpenTelemetry trace exporter.

### Target stack

- Runtime: TypeScript / Node.js (or Rust — decision tracked as an ADR).
- DB: SQLite (bundled, zero-config).
- SCE backend: SQLite WAL-mode.
- CLI: compiled binary via `bun build --compile` or `cargo build --release`.

### Acceptance criteria

- [ ] `aidevos init && aidevos doctor` succeeds on a clean machine with only Ollama installed.
- [ ] `aidevos run "write a hello world in TypeScript"` completes against `ollama/llama3.1:8b` with zero network egress.
- [ ] `aidevos models list --json` returns a non-empty catalog with provider grouping matching the Nine Router spec.
- [ ] `aidevos router assign builder openai/gpt-4o` reflects in `aidevos router show`.
- [ ] All Kernel events appear in the Audit Log with `correlation_id`.

---

## Phase 2 — Context Engine & Persistent Memory

**Theme**: Cross-session durability. Results, decisions, and KB entries survive process restarts. Replay works.

### Deliverables

- **Shared Context Engine** full implementation: `snapshot + tail`, durable subscriber cursors, compaction, multi-backend (SQLite + NATS/JetStream).
- **Persistent Memory**: relational tier (SQLite) + vector tier (`usearch` embedded); `memory.write`, `memory.query`, `memory.delete`; retention job.
- **Embeddings**: local embedding via Ollama (`nomic-embed-text`); async backfill queue.
- **Vector Store**: embedded `usearch` index; ANN search; cosine similarity.
- **Agent Memory**: per-agent working memory backed by Persistent Memory.
- **Kernel replay**: `kernel.replay(run_id)` reconstructs task graph and verdicts from stored events.

### Acceptance criteria

- [ ] Stopping and restarting the backend with an in-flight run replays the run from the last checkpoint.
- [ ] `memory.query("how does the Nine Router assign fallbacks", { k: 5 })` returns relevant records with score > 0.7 after indexing the docs.
- [ ] Retention job correctly expires records with `retention: "7d"` on schedule.
- [ ] Cold subscriber attaches to a 100k-event topic via `snapshot + tail` in < 2 s.

---

## Phase 3 — AI Groups & Dynamic Workers

**Theme**: Parallel execution. Multiple workers collaborate on a goal decomposed into a task graph.

### Deliverables

- **AI Group System**: `spawn_group`, heartbeat protocol, circuit breaker, elastic autoscaler.
- **Dynamic Workers**: full `WorkerSpec` execution loop; tool dispatch (native + MCP); checkpointing; streaming event emission.
- **Task Graph**: parallel task execution; dependency resolution; sibling parallelism.
- **Planning Engine**: goal → `TaskGraph` decomposition using the Planner role.
- **Critic role**: verdict emission (`accept` / `reject`); retry loop.
- **Job Scheduler**: cron-based job scheduling for Research Engine and Model Discovery.
- **Queueing**: in-memory queue with backpressure signals; Redis backend option.

### Built-in Groups shipped

- `code-builder` (Planner + Builder + Critic)
- `researcher` (Planner + Researcher + Critic)
- `guardian` (Guardian role only)

### Acceptance criteria

- [ ] A goal decomposed into 3 parallel sub-tasks completes faster than sequential execution (measured by wall time).
- [ ] A Worker that misses 3 heartbeats triggers task reassignment within `heartbeat_grace` seconds.
- [ ] Circuit breaker opens after `crash_threshold` crashes and notifies the Kernel.
- [ ] `aidevos runs list --state running` shows all active runs with worker counts.

---

## Phase 4 — Knowledge System & Research Engine

**Theme**: The OS becomes self-improving — it researches, curates, and recalls knowledge across sessions.

### Deliverables

- **Knowledge System**: four-tier KB (Global/Main/Group/Individual) backed by Persistent Memory; layered query with scope resolution.
- **Obsidian Graph Engine**: filesystem watcher; incremental graph updates; `neighbors`, `path`, `similar` queries; semantic edges.
- **RAG Pipeline**: hybrid retrieval (BM25 + ANN); context window assembly; relevance ranking.
- **Research Engine**: full pipeline (fetch → parse → dedup → diff → provenance → write); HTTP, GitHub, npm, arXiv adapters; scheduled jobs; reactive freshness.
- **Web Intelligence**: Playwright-based browser automation for JS-heavy sites.
- **Internet Search**: pluggable search backend (SearXNG local, Brave, Google).
- **GitHub Analysis**: repo structure, issue, PR, and release analysis.

### Acceptance criteria

- [ ] Scheduling a research job against the TanStack Router docs produces KB entries within the first run.
- [ ] `graph.neighbors("docs/MAIN_AI_KERNEL.md", depth=2)` returns all documents reachable within 2 hops in ≤ 100 ms.
- [ ] A `knowledge.stale` event with `urgency: "high"` triggers a re-crawl within 30 s.
- [ ] A RAG query over the full doc vault returns the correct top-3 documents for 5 hand-crafted test queries.

---

## Phase 5 — Merge Manager, Guardian & Impact Analysis

**Theme**: Change safety. Multiple agents can safely edit concurrently; dangerous changes are blocked before they land.

### Deliverables

- **Merge Manager**: full three-way merge; structural awareness for Markdown, YAML, JSON; conflict escalation UI; audit trail.
- **Architecture Guardian**: rule engine; built-in rule set (Table in [ARCHITECTURE_GUARDIAN](./ARCHITECTURE_GUARDIAN.md)); hot-reload; auto-fix for `redact`.
- **Impact Analysis**: graph traversal; risk scoring; suggested reviewers; blast radius calculation.
- **Guardian rule authoring**: `~/.aidevos/rules/` directory; YAML DSL validation; `aidevos guardian rules` CLI.
- **Merge conflict UI**: side-by-side diff in web shell; accept/reject/manual actions.

### Acceptance criteria

- [ ] Two workers concurrently appending to the same Markdown file produce a clean merge.
- [ ] A secret-pattern match in an artifact produces a `critical` Guardian veto.
- [ ] Changing `docs/MAIN_AI_KERNEL.md` produces `impact.risk > 0.6` due to high centrality.
- [ ] A conflict escalated to the user appears in the web shell within 500 ms.
- [ ] Custom rule in `~/.aidevos/rules/` activates without a process restart.

---

## Phase 6 — Voice System, Plugin SDK & UX Polish

**Theme**: The OS is complete as a platform. Third parties can extend it; users can speak to it.

### Deliverables

- **Voice System**: Whisper local STT; Piper local TTS; VAD; wake-word (OpenWakeWord); streaming TTS; interruption handling.
- **Plugin SDK**: manifest format; in-process (WASM) and subprocess execution models; capability gate; consent flow; crash recovery; `aidevos plugins` CLI.
- **MCP full implementation**: client (stdio + SSE + streamable-http) and server (`memory://`, `context://`, `knowledge://`, `graph://` resources + tools).
- **Desktop shell** (Electron or Tauri): model panel, run viewer, memory explorer, graph visualiser.
- **Web shell** (React): parity with desktop for remote/cloud deployments.
- **API Gateway**: REST + WebSocket API as documented in [API Spec](./API_SPEC.md); rate limiting; auth.

### Acceptance criteria

- [ ] `aidevos voice say "Hello"` produces audio within 500 ms on a machine with Piper installed.
- [ ] A third-party plugin installs from a local path and its tool is callable via `aidevos mcp call`.
- [ ] The desktop shell renders the Nine Router model panel with live refresh.
- [ ] Web shell and desktop shell have feature parity for all Phase 1–5 capabilities.

---

## Phase 7 — v1.0 General Availability

**Theme**: Production-ready. Reliability targets met; documentation complete; migration guides written; eval harness passing on all default prompts.

### Deliverables

- All reliability targets from [Reliability](./RELIABILITY.md) met and measured.
- Eval Harness green on all default prompts across all built-in Groups.
- Full benchmark suite from [Benchmarks](./BENCHMARKS.md) run and results published.
- Security audit completed; findings addressed; [Security Model](./SECURITY_MODEL.md) updated.
- Multi-tenant deployment documented in [Deployment](./DEPLOYMENT.md).
- Migration guide from pre-v1 snapshots in [Migration Guide](./MIGRATION_GUIDE.md).
- Complete [CHANGELOG](./CHANGELOG.md) from Phase 0 to v1.0.
- API stability guarantee: all v1.0 API surfaces are semver-stable.

### Reliability targets (from [Reliability](./RELIABILITY.md))

| Target | Value |
|--------|-------|
| Kernel loop p99 latency (local) | < 5 s for single-task goal |
| SCE publish p99 (SQLite) | < 5 ms |
| Model discovery refresh p99 | < 3 s for all providers |
| Memory query p95 (10k records) | < 100 ms |
| Guardian check p95 | < 500 ms |
| Merge commit p95 (clean) | < 200 ms |
| Uptime (local mode) | 99.9% (no cloud dependencies) |

### Acceptance criteria

- [ ] All Phase 0–6 acceptance criteria pass in CI.
- [ ] Eval Harness reports ≥ 90% pass rate on the default prompt suite.
- [ ] Zero critical Guardian violations in the v1.0 release artifact.
- [ ] `aidevos doctor` passes on macOS (Apple Silicon), macOS (Intel), Ubuntu 22.04, and Windows 11 (WSL2).
- [ ] All `MUST` clauses in all 125+ docs have a corresponding acceptance criterion or open ADR.

---

## Post-v1.0 Backlog (Unscheduled)

The following capabilities are explicitly out of scope for v1.0 but are documented for future phases:

| Capability | Relevant doc |
|------------|-------------|
| Multi-tenant cloud deployment | [Deployment](./DEPLOYMENT.md) |
| Remote sync and collaboration | [Localhost Architecture](./LOCALHOST_ARCHITECTURE.md) |
| Fine-tuning pipeline | [Model Providers](./MODEL_PROVIDERS.md) |
| Agent marketplace | [Plugin SDK](./PLUGIN_SDK.md) |
| Real-time voice conversation (< 100 ms) | [Voice System](./VOICE_SYSTEM.md) |
| Video/screen input (Vision role) | [Nine Router](./NINE_ROUTER.md) |
| Custom embedding models | [Embeddings](./EMBEDDINGS.md) |
| Federated knowledge bases | [Knowledge System](./KNOWLEDGE_SYSTEM.md) |

---

## Timeline Guidance

| Phase | Estimated duration | Key risk |
|-------|--------------------|----------|
| Phase 0 | 2 weeks | Scope creep in spec writing |
| Phase 1 | 4 weeks | Runtime stack decision (TS vs Rust) |
| Phase 2 | 3 weeks | Vector index performance at scale |
| Phase 3 | 5 weeks | Distributed worker scheduling correctness |
| Phase 4 | 4 weeks | Web scraping reliability; copyright compliance |
| Phase 5 | 4 weeks | Three-way merge correctness for edge cases |
| Phase 6 | 5 weeks | Cross-platform audio API differences |
| Phase 7 | 3 weeks | Security audit findings scope |

Total estimated: **~30 weeks** from Phase 1 start to v1.0.

---

## Related Documents

- [PRD](./PRD.md)
- [TRD](./TRD.md)
- [Roadmap](./ROADMAP.md)
- [Release Process](./RELEASE_PROCESS.md)
- [Reliability](./RELIABILITY.md)
- [Eval Harness](./EVAL_HARNESS.md)
- [Benchmarks](./BENCHMARKS.md)
- [Changelog](./CHANGELOG.md)
- [Migration Guide](./MIGRATION_GUIDE.md)
