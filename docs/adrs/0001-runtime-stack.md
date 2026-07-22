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

### Consequences

**Positive:**
- Rapid prototyping for all core subsystems
- Access to mature SDKs for every model provider (OpenAI, Anthropic, Google, Mistral)
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
