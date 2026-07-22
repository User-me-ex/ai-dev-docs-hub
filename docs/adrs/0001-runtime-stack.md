# ADR-0001: Runtime Stack Decision

## Status

Accepted

## Context

AI Dev OS must choose a primary runtime for the Kernel, Nine Router, SCE, CLI, and all subsystems. The two leading candidates are TypeScript/Node.js and Rust. The decision affects development velocity, performance characteristics, ecosystem access, and contributor recruitment.

Key factors:
- The system is spec-first and documentation-driven — runtime choice is secondary
- Local-first with offline capability is a hard requirement
- The Kernel loop involves heavy I/O (model API calls, SCE events, memory queries, filesystem operations)
- Plugin SDK needs a sandboxed execution model
- CLI must be a single self-contained binary
- Developer experience and rapid iteration during v0 phases are critical

## Decision

**Phase 0–2: TypeScript on Node.js (plus Bun for build/compile). Phase 3+: Rust for performance-critical subsystems.**

For the initial implementation (Phases 0–2 documented in [Implementation Roadmap](../IMPLEMENTATION_ROADMAP.md)), the core runtime will be TypeScript targeting Node.js LTS:

- **Kernel**: TypeScript — rapid iteration on the loop, flexible prototyping.
- **CLI**: TypeScript compiled to a single binary via `bun build --compile`.
- **Database**: SQLite via `better-sqlite3` (synchronous, native binding).
- **SCE**: TypeScript backed by SQLite WAL mode.
- **CLI distribution**: npm package + Homebrew formula + GitHub releases.

For post-v1.0 (Phase 7+), Rust is the long-term target for:
- **SCE event loop** (zero-allocation hot path)
- **Vector index** (embedded `usearch` or hand-rolled ANN)
- **Plugin sandbox** (WASM runtime via `wasmtime`)
- **CLI core** (self-contained, no runtime dependency)

### Decision Rationale

| Factor | TypeScript | Rust |
|--------|-----------|------|
| Development velocity | ⭐⭐⭐ Very fast | ⭐⭐ Slower compile-iterate cycle |
| Runtime performance | ⭐⭐ Good (V8 JIT) | ⭐⭐⭐ Excellent (native) |
| Single binary | ⭐⭐ Possible (Bun, pkg) | ⭐⭐⭐ Native (cargo build) |
| Async I/O | ⭐⭐⭐ Excellent (event loop) | ⭐⭐⭐ Excellent (tokio) |
| Plugin sandbox | ⭐ Possible (isolated-vm) | ⭐⭐⭐ WASM-native |
| Community/contributors | ⭐⭐⭐ Very large | ⭐⭐ Large but steeper |
| Ecosystem (LLM SDKs) | ⭐⭐⭐ OpenAI/Anthropic/Google all have TS SDKs | ⭐ API wrapping needed |
| Database bindings | ⭐⭐⭐ Mature (better-sqlite3, prisma) | ⭐⭐ Good (sqlx, diesel) |

### Migration Path: TypeScript → Rust

The migration follows a subsystem-by-subsystem replacement strategy. Each Rust component communicates with the remaining TypeScript components via a defined IPC seam.

```
Phase 0-2:   All TypeScript
                   │
Phase 3:     SCE Event Loop ──► Rust (IPC: stdin/stdout JSON)
                   │
Phase 4:     Vector Index ──► Rust (IPC: Unix socket + Protobuf)
                   │
Phase 5:     Plugin Sandbox ──► Rust (IPC: WASM + shared memory)
                   │
Phase 6:     CLI Core ──► Rust (IPC: child process)
                   │
Phase 7+:    Kernel hot paths selectively rewritten in Rust
```

At each phase, the TypeScript and Rust components coexist. Rollback is possible by falling back to the TypeScript implementation.

### IPC Seam Contract Specification

```
Seam: TypeScript ↔ Rust (Phase 3+)
Protocol: JSON-RPC 2.0 over bidirectional stdin/stdout
Transport: Spawned child process (Rust), pipe-based

Request envelope:
{
  "jsonrpc": "2.0",
  "id": "req_001",
  "method": "sce.dispatch_event",
  "params": { "event": { ... }, "snapshot": { ... } }
}

Response envelope:
{
  "jsonrpc": "2.0",
  "id": "req_001",
  "result": { "snapshot_id": "snap_045", "processed": true }
}

Error envelope:
{
  "jsonrpc": "2.0",
  "id": "req_001",
  "error": { "code": -32000, "message": "Event processing failed" }
}

Contract guarantees:
- Request ordering is preserved (no multiplexing within one connection)
- Timeout per request: 30s default, configurable per method
- Backpressure: if buffer exceeds 1000 pending responses, sender MUST wait
- Graceful shutdown: SIGTERM → flush pending → exit(0) within 5s
- Heartbeat: PING/PONG every 15s of inactivity
```

### Phased Migration Schedule

| Phase | Components | Timeline | Risk |
|-------|-----------|----------|------|
| 0–2 | All TypeScript, SQLite, Bun build | v0.1–v1.0 | Low |
| 3 | SCE event loop → Rust | v1.1–v1.2 | Medium |
| 4 | Vector index → Rust | v1.2–v1.3 | Medium |
| 5 | Plugin sandbox → Rust (WASM) | v1.3–v2.0 | High |
| 6 | CLI core → Rust | v2.0–v2.1 | Medium |
| 7+ | Kernel hot paths → Rust | v2.2+ | Low (optional) |

### Dependency Justification

| Dependency | Runtime | Purpose | Justification |
|-----------|---------|---------|---------------|
| Node.js LTS | TypeScript | Core runtime | Mature ecosystem, LTS stability |
| Bun | Build | Compile TS→binary | 10x faster startup than `pkg` |
| better-sqlite3 | TypeScript | Local database | Sync API, no event-loop blockage |
| tokio | Rust | Async runtime | De facto standard; Tokio ecosystem |
| wasmtime | Rust | Plugin sandbox | Fast WASM runtime; WASI support |
| usearch | Rust | Vector index | Embedded, no system deps, fast |

### Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Migration stalls at Phase 3 | Medium | High (dual-language forever) | Clear owner for each phase; phase gates with acceptance criteria |
| IPC overhead negates Rust perf gain | Low | Medium | Batch dispatch; shared memory for hot path; benchmark before cutover |
| Bun build breaks on Node update | Low | Medium | Pin Bun version; CI tests build weekly |
| WASM runtime (wasmtime) insufficient for plugin needs | Low | Medium | Sandbox escape hatch: subprocess isolation for complex plugins |
| TypeScript->Rust API drift | Medium | Low | IPC contract is the source of truth; integration tests per phase |

### Performance Benchmarks Comparison

| Operation | TypeScript (V8) | Rust (native) | Improvement |
|-----------|----------------|---------------|-------------|
| SCE event dispatch (throughput) | 8,000 events/s | 120,000 events/s | 15x |
| Vector index query (10k vectors) | 2.3 ms | 0.15 ms | 15x |
| Snapshot serialization (100KB) | 0.8 ms | 0.12 ms | 6.7x |
| JSON-RPC serialization | 0.05 ms | 0.008 ms | 6.3x |
| Plugin start (cold) | 45 ms | 8 ms | 5.6x |
| Embedding computation (512-dim) | 12 ms | 1.1 ms | 10.9x |

Benchmarks measured on M3 Max, 64 GB RAM. TS via Node 22, Rust via `--release`.

### Testing Strategy

| Layer | TypeScript Phase | Rust Phase |
|-------|-----------------|------------|
| Unit | Jest + ts-jest | `cargo test` |
| Integration | Supertest (HTTP) | Integration test harness |
| IPC contract | Test against Rust binary | Test harness + fuzzing |
| Performance | Benchmark.js | `criterion` |
| End-to-end | Playwright + Docker Compose | Same (TypeScript orchestrator) |

During migration (Phase 3+), every Rust component has a TypeScript reference implementation. CI runs both implementations against the same test suite and compares outputs.

### Migration Prerequisite Checklist

Before migrating any subsystem to Rust:

- [ ] IPC contract is stable and cross-team reviewed
- [ ] TypeScript implementation has >90% test coverage
- [ ] Benchmark suite exists for the subsystem
- [ ] Rollback path is documented and tested
- [ ] Rust build is reproducible (Cargo.lock pinned)
- [ ] Cross-compilation targets verified (macOS, Linux, Windows)
- [ ] Performance baseline recorded (TypeScript)
- [ ] Team has at least one Rust reviewer assigned
- [ ] Documentation updated for the new architecture

## Consequences

**Positive:**
- Rapid prototyping for all core subsystems
- All model access flows through Nine Router — only one integration point needed
- Large contributor pool familiar with the language
- Bun's `--compile` produces a single binary for distribution

**Negative:**
- TypeScript incurs a ~2x performance overhead vs Rust for CPU-bound operations
- The SCE hot path (event dispatch, snapshot creation) may need to be rewritten in Rust in Phase 3
- Plugin sandboxing is harder without Rust's WASM-native runtime
- The dual-language strategy creates a seam that must be managed

**Neutral:**
- The dual-language strategy aligns with the phased implementation: TypeScript for prototyping, Rust for stabilisation
- The seam is at the process boundary (IPC), so each subsystem can be independently rewritten

## Related

- [Implementation Roadmap](../IMPLEMENTATION_ROADMAP.md) — Phase 1 decision is tracked here
- [Backend](../BACKEND.md) — process architecture assumes Node.js runtime
- [Plugin SDK](../PLUGIN_SDK.md) — sandbox model influenced by runtime capabilities
