# AI Coding Rules

> Mandatory rules every AI agent in AI Dev OS MUST follow when producing or modifying code in a user project. These rules are enforced by the [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) and MUST NOT be bypassed under any circumstances.

## Purpose

These rules exist to make AI-generated changes safe, auditable, and reversible. They are not suggestions or style preferences — they are invariants. When an agent violates a rule, the [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) issues a veto and the Kernel replans. Agents are expected to internalize these rules so that violations are rare, not common.

Rules are divided into three categories: **Core** (always apply), **Style** (apply when modifying code), and **Prohibited** (never do these things in any context).

---

## Core Rules

### 1. Small, reversible steps

Prefer many small diffs over one large rewrite. A diff that touches more than 300 lines SHOULD be split into multiple tasks unless the change is a pure mechanical transformation (e.g., rename across a file). Each step must leave the system in a working state.

**Why**: Large diffs are hard to review, hard to revert, and hard to merge safely. A veto on a large diff wastes more work than a veto on a small one.

### 2. Read before write

Never edit a file without first reading its current contents. Never assume you know what a file contains — always fetch the latest version from the [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md) or the filesystem. This applies even if you wrote the file earlier in the same session.

**Why**: Concurrent agents may have modified the file between your read and your write. Reading before writing detects this and enables the [Merge Manager](./MERGE_MANAGER.md) to handle conflicts correctly.

### 3. Respect the Guardian

The [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) can veto any change. Do not attempt to route around it by splitting a prohibited change across multiple small commits, by modifying Guardian rules, or by disabling rules. A Guardian veto is a signal to replan, not to find a workaround.

**Why**: The Guardian is the trust anchor for change safety. Routing around it undermines the entire governance model.

### 4. Documentation-first

New subsystems, APIs, and significant features MUST have a specification document in `docs/` before any implementation code is written. The spec is the contract; implementation follows the contract, not the other way around.

**Why**: Documenting first forces clarity of thought, enables parallel review, and creates the invariants that the Guardian can enforce before a line of code exists.

### 5. Cite sources

When applying an external pattern, algorithm, or third-party API usage, cite the source in the commit or PR body. Include: source URL, date accessed, and the specific section referenced.

**Why**: Provenance tracking enables future agents to verify that cited sources still reflect current behaviour and to detect when dependencies have changed.

### 6. No hidden state

All persistent state MUST flow through the [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md) or an approved store ([Persistent Memory](./PERSISTENT_MEMORY.md), the database per [DATABASE](./DATABASE.md)). No agent may use in-process global variables, module-level singletons, or filesystem paths outside the approved data directories as a substitute for durable state.

**Why**: Hidden state breaks replay, makes debugging impossible, and prevents the Guardian from reasoning about system state.

### 7. Deterministic tests first

New behaviour requires a failing test (unit, integration, or eval) committed in the same task before the fix or feature code is written. Tests MUST be deterministic: no `Math.random()`, no `Date.now()`, no network calls unless explicitly mocked.

**Why**: Tests-first ensures the change is observable, makes regressions detectable, and prevents the Guardian from accepting a change that has no verification path.

### 8. Least privilege

Request only the tools and scopes actually needed for the task. If a task requires `file_read` but not `file_write`, request only `file_read`. If a task requires reading one directory, do not request access to the entire filesystem.

**Why**: Least privilege limits the blast radius of errors and security incidents. An agent that requests excessive privileges is a signal of poor task decomposition.

### 9. Ask before destructive actions

Deletions, migrations, force-pushes, database schema drops, and bulk overwrites require explicit human confirmation before execution. Agents MUST surface the specific list of entities to be deleted/modified and the rationale before proceeding.

**Why**: Destructive operations are irreversible or expensive to reverse. Human confirmation is the last safety gate before data loss.

### 10. Prefer standards

Use OpenAPI for API specs, JSON Schema for data validation, Mermaid for diagrams, Markdown for documentation, and POSIX shell for scripts where possible. Avoid bespoke formats when a well-supported standard exists.

**Why**: Standards reduce cognitive load, improve tooling compatibility, and make the system easier to reason about for both humans and AI agents.

### 11. Idempotent mutations

Every mutation MUST be idempotent when identified by its `run_id` and `task_id`. Retrying a failed operation MUST NOT produce duplicate records, double-writes, or side effects beyond what the original successful execution would have produced.

**Why**: Retries are fundamental to reliability. Non-idempotent mutations make retries dangerous.

### 12. Propagate correlation_id end-to-end

Every agent call, model call, tool call, and event emission MUST carry the `correlation_id` from the originating Kernel run. Do not generate new UUIDs mid-chain; use the one provided in the `WorkerSpec`.

**Why**: End-to-end correlation is the foundation of distributed tracing and audit. Without it, debugging multi-agent runs is impossible.

---

## Style Rules

These rules apply when modifying existing code or documentation. They are enforced as `warning` severity by the Guardian (surfaces a hint but does not veto).

- **Match surroundings**: Match the surrounding file's indentation, naming conventions, and comment style. Do not reformat unrelated lines; keep diffs minimal.
- **Public API doc comments**: All public functions, classes, and types MUST have doc comments. Private helpers SHOULD.
- **Actionable error messages**: Error messages MUST explain what happened, what state the system is in, and (if possible) what the user or caller can do next.
- **Section skeleton for docs**: New documents MUST follow the standard section skeleton: Overview → Goals → Non-Goals → Requirements → Architecture → Interfaces → Data Model → Failure Modes → Security → Observability → Acceptance Criteria → Open Questions → Related Documents.
- **RFC 2119 language**: Requirements sections MUST use `MUST`, `MUST NOT`, `SHOULD`, `SHOULD NOT`, `MAY` with the semantics defined in RFC 2119.
- **Cross-links**: Any reference to another subsystem in a doc MUST be a Markdown link. Never reference a subsystem by name alone.
- **Mermaid diagrams in docs**: Architecture sections MUST include a Mermaid `flowchart` or `stateDiagram` block. The diagram lives inline in the doc or in the `diagrams/` directory with a link.

---

## Prohibited Actions

These are absolute prohibitions. The Guardian enforces all of these as `critical` severity — a violation vetoes the change unconditionally.

| # | Prohibition | Reason |
|---|-------------|--------|
| P1 | **No application code in a documentation-only repository** | This repo is spec-only; code lives in the implementation repo |
| P2 | **No secrets, credentials, or tokens in any committed file** | Secrets in files are a critical security incident |
| P3 | **No direct model provider calls** | All provider I/O must go through the Kernel-proxied [Model Providers](./MODEL_PROVIDERS.md) |
| P4 | **No bypassing the Merge Manager** | Concurrent edits without the [Merge Manager](./MERGE_MANAGER.md) cause data loss |
| P5 | **No bypassing the Architecture Guardian** | The Guardian is non-negotiable; routing around it is a prohibited action |
| P6 | **No storing sensitive data in SCE event payloads** | SCE events are broadly readable; sensitive data must be tokenised or stored in encrypted Persistent Memory |
| P7 | **No ad-hoc network calls from workers** | All outbound network access must go through the Kernel-proxied HTTP client |
| P8 | **No modifying Guardian rule files from within a run** | Rules must be reviewed and committed by a human, not auto-generated by a run |
| P9 | **No `.env` files or `process.env` credential access** | Credentials come from [Secrets Management](./SECRETS_MANAGEMENT.md) only |
| P10 | **No force-push to main/production branches** | Protected branches require PR review; force-push destroys history |

---

## Guidance for Common Situations

### When you are unsure of scope

If you cannot determine whether a change falls within your task's scope, emit a `task.clarification_needed` event on the SCE and wait for the Kernel to provide clarification. Do not guess and proceed — a wrong guess that violates a rule wastes a full Kernel loop.

### When a Guardian veto surprises you

Read the `Verdict.violations[].message` and `fix_hint` fields. They are written to be actionable. Do not retry the same change without addressing the specific violation. If the violation is a false positive, flag it via the `impact.stale` channel so a human can review the rule.

### When tests are hard to write

A test that is hard to write is a signal that the interface is hard to use. Before skipping the test, consider whether the API or data model should be redesigned. If a test is genuinely infeasible (e.g., testing hardware I/O), document why in the spec and open a question in the doc's "Open Questions" section.

### When you need to touch many files

If a single task requires touching > 10 files, split it into sub-tasks in the [Task Graph](./TASK_GRAPH.md). Each sub-task should have a coherent, reviewable scope. Use the [Impact Analysis](./IMPACT_ANALYSIS.md) output to understand blast radius before committing.

---

## Rule Versioning and Updates

These rules are version-controlled at the top level of this document. Changes require:
1. An ADR explaining the motivation (see [templates/ADR](../templates/ADR.md)).
2. A corresponding update to the Guardian's built-in rule set.
3. Review by at least one human maintainer.
4. A changelog entry in [CHANGELOG](./CHANGELOG.md).

Current rule set version: `1.0.0`.

---

## Related Documents

- [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) — enforces these rules
- [Merge Manager](./MERGE_MANAGER.md) — enforces rule P4
- [Impact Analysis](./IMPACT_ANALYSIS.md) — blast-radius pre-flight for rule 9
- [Prompt Governance](./PROMPT_GOVERNANCE.md) — prompt-level equivalent rules
- [Security Model](./SECURITY_MODEL.md)
- [Audit Log](./AUDIT_LOG.md)
- [templates/ADR](../templates/ADR.md) — for rule change proposals
