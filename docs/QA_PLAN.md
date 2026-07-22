# QA Plan

> Quality assurance plan for AI Dev OS covering functional, integration, prompt quality, security, performance, and reliability testing.

## Overview

The QA plan defines the testing strategy, environments, release gates, and quality metrics that ensure AI Dev OS releases are reliable, secure, and performant. QA is integrated into the CI/CD pipeline and enforced by the [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) as a release gate. Every change MUST pass the appropriate QA gates before merging.

## QA Categories

| Category | Focus | Owner |
|---|---|---|
| Functional testing | Core subsystem contracts, API correctness | Development team |
| Integration testing | Cross-subsystem contracts, SCE event flow | Integration team |
| Prompt quality | Instruction following, output consistency, safety | AI quality team |
| Security testing | Auth, encryption, secret handling, injection | Security team |
| Performance testing | Latency targets, throughput, memory, GPU | Performance team |
| Reliability testing | Fault injection, recovery, degradation paths | Reliability team |

### Functional Testing

- Unit tests for every subsystem contract (see [Testing Strategy](./TESTING_STRATEGY.md)).
- API endpoint tests (happy path + error cases) per [API Spec](./API_SPEC.md).
- CLI command smoke tests for every `aidevos` subcommand.

### Integration Testing

- SCE event propagation across Kernel → Worker → Guardian → Memory.
- Model provider integration with mock provider responses.
- Plugin lifecycle: load → init → call → unload.
- End-to-end run: intake → plan → route → execute → critique → merge → guard → deliver.

### Prompt Quality

- Eval suite from [Eval Harness](./EVAL_HARNESS.md) run against every model provider.
- Governance rule compliance (see [Prompt Governance](./PROMPT_GOVERNANCE.md)).
- Safety check: output does not contain prohibited content (see [AI Safety](./AI_SAFETY.md)).

### Security Testing

- Authentication and authorization bypass attempts (see [Auth System](./AUTH_SYSTEM.md)).
- Encryption verification: data at rest and in transit (see [Encryption](./ENCRYPTION.md)).
- Secret injection detection: no secrets in logs or SCE events.
- Dependency vulnerability scan on every build.

### Performance Testing

- Benchmarks from [Benchmarks](./BENCHMARKS.md) run on reference hardware.
- Latency targets from [Performance](./PERFORMANCE.md) — all MUST pass.
- Memory leak detection: 1-hour soak run with stable RSS.

### Reliability Testing

- Fault injection: drop SCE events, kill workers, corrupt cache.
- Recovery: auto-restart, replay, reindex after failure.
- Degradation: verify graceful degradation when dependencies are unavailable.

## Test Environments

| Environment | Purpose | Schedule | Data |
|---|---|---|---|
| Local dev | Developer iteration | On commit | Synthetic |
| CI (GitHub Actions) | Pre-merge validation | Per PR | Synthetic |
| Staging | Pre-release validation | Per release candidate | Anonymized production snapshot |
| Production canary | Canary deployment monitoring | Continuous | Real (read-only metrics) |

## Release QA Checklist

Every release candidate MUST pass the following before shipping:

- [ ] All CI tests pass (functional + integration + security).
- [ ] Eval harness suite passes for all active model providers.
- [ ] Performance benchmarks meet all targets (see [Performance](./PERFORMANCE.md)).
- [ ] Reliability fault-injection suite passes (no data loss, auto-recovery).
- [ ] No critical or high-severity open bugs against the release.
- [ ] Security scan reports zero new vulnerabilities.
- [ ] Changelog reviewed for accuracy.

## Regression Testing Strategy

- Every bug fix includes a test that reproduces the bug (see [Testing Strategy](./TESTING_STRATEGY.md)).
- The full eval harness suite is run against every release candidate and compared to the previous release.
- A regression is defined as any metric that degrades by > 5% (latency, throughput, eval score).
- Regressions block release and require either a fix or a documented trade-off approved by the QA lead.

## Manual Testing Scenarios

Scenarios that require human review, run before each major release:

1. **First-run experience**: Install, configure, run a simple task end-to-end.
2. **Multi-agent workflow**: Run a task that requires 3+ agents with shared context.
3. **Error recovery**: Kill a worker mid-task and verify the run completes.
4. **Large context**: Run a task with 50 K+ tokens of context.
5. **Plugin extension**: Load a custom plugin and verify it participates in the loop.

## Bug Tracking Process

- All bugs are filed in the issue tracker with severity (`critical`, `high`, `medium`, `low`).
- Critical bugs block the release. High bugs require a documented workaround.
- Every bug MUST include: steps to reproduce, expected vs actual behavior, environment, logs.
- Bugs are triaged weekly by the QA lead. See [Error Handling](./ERROR_HANDLING.md) for error taxonomy.

## Quality Metrics and Targets

| Metric | Target | Measurement |
|---|---|---|
| Test pass rate (CI) | 100% | Per build |
| Eval harness score | > 85% | Per model provider |
| Performance targets met | 100% of MUST targets | Per release |
| Regression rate | < 2% metric degradation | Per release |
| Critical bug count | 0 at release | At ship time |
| Security vulns (critical/high) | 0 | Per scan |

## Related Documents

- [Testing Strategy](./TESTING_STRATEGY.md) — unit, integration, and regression test practices
- [Eval Harness](./EVAL_HARNESS.md) — structured evaluation framework for prompts and models
- [Benchmarks](./BENCHMARKS.md) — performance and quality benchmarking
- [Error Handling](./ERROR_HANDLING.md) — error taxonomy, codes, and recovery
- [Release Process](./RELEASE_PROCESS.md) — staging, canary, and production release steps
- [Security Model](./SECURITY_MODEL.md) — security testing scope
