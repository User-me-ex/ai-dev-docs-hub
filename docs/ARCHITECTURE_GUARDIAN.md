# Architecture Guardian

> The invariant-enforcement authority with final veto power over every change that reaches the merge stage. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

The Architecture Guardian (AG) is the last mandatory gate in the [Main AI Kernel](./MAIN_AI_KERNEL.md) loop. Before any `MergedArtifact` is delivered to the user, the Guardian evaluates it against a declarative rule set and issues one of two verdicts: **ok** (change is delivered) or **veto** (change is rejected and the Kernel replans). The veto is final: no subsystem — including the Kernel itself — may route around it.

The Guardian is intentionally narrow in scope. It does not understand business intent; it enforces structural, security, and quality invariants that have been declared in advance by the operator or the AI coding rules. It is analogous to a compiler's type checker: it cannot tell you if the program is correct, but it guarantees that it respects a known set of constraints.

Rules are first-class documents. Adding, modifying, or retiring a rule is a documentation change (a new or updated `Rule` block), not a code change. The Guardian reloads its rule set on every evaluation (or on a file-watch event), so rule changes take effect without a restart.

## Goals

- Enforce a declarative rule set against every proposed change before delivery.
- Explain every veto in actionable terms: which rule was violated, what the violation was, and how to fix it.
- Block regressions before they reach the user: catches issues that the Critic missed or accepted.
- Reload rules dynamically: rule changes take effect on the next evaluation without a restart.
- Operate fail-closed for `critical` severity rules: if the rule engine crashes while evaluating a critical rule, the change is vetoed.

## Non-Goals

- Understanding business intent — the Guardian enforces declared constraints, not correctness.
- Running tests — that is the Critic's domain.
- Implementation code — this repository is documentation-only (see [AI Coding Rules](./AI_CODING_RULES.md)).
- Duplicating contracts of the [Merge Manager](./MERGE_MANAGER.md) — the Guardian evaluates the merged result, not the merge process.

## Architecture

```mermaid
flowchart TB
  MERGE[Merge Manager] -->|MergedArtifact| GUARD[Architecture Guardian]

  subgraph Guardian internals
    RELOAD[Rule loader\nfile-watch / hot-reload]
    EVAL[Rule evaluator\nper-rule parallel]
    IMPACT[Impact Analysis call]
    EXPLAIN[Explanation generator]
    AUDIT_W[Audit writer]
  end

  GUARD --> RELOAD
  RELOAD -->|Rule[]| EVAL
  GUARD --> IMPACT
  IMPACT -->|Impact| EVAL
  EVAL -->|violations[]| EXPLAIN
  EXPLAIN --> VERDICT{Verdict}
  VERDICT -->|ok| DELIVER[Kernel delivers result]
  VERDICT -->|veto| REPLAN[Kernel replans]
  GUARD --> AUDIT_W
  AUDIT_W --> AUDITLOG[(Audit Log)]
  GUARD --> SCE[(SCE: guardian.verdicts)]
```

The Guardian is stateless between evaluations: it loads rules, evaluates the artifact, emits the verdict, and releases all state. Persistent records live in the [Audit Log](./AUDIT_LOG.md) and the `guardian.verdicts` SCE topic.

## Rule Schema

```
Rule {
  id:          string       # stable kebab-case; e.g. "no-secrets-in-text"
  name:        string       # human display name
  description: string       # what this rule enforces
  scope:       RuleScope    # see below
  expr:        RuleExpr     # declarative expression (see DSL)
  severity:    "critical"   # veto; fail-closed if engine crashes
             | "high"       # veto; fail-open if engine crashes
             | "warning"    # append to hints[] but do not veto
  auto_fix?:   AutoFixSpec  # optional: how to auto-remediate
  tags:        string[]
  version:     semver
}

RuleScope {
  kinds:    ("code"|"doc"|"prompt"|"config"|"any")[]
  paths?:   GlobPattern[]   # file path filters
  groups?:  GroupId[]       # only enforce when change originates from these Groups
}
```

## Built-in Rules

The Guardian ships with the following built-in rules. Operators MAY add custom rules via the `~/.aidevos/rules/` directory or the [Plugin SDK](./PLUGIN_SDK.md).

| Rule ID | Severity | Description |
|---------|----------|-------------|
| `no-secrets-in-text` | critical | Detect credentials, API keys, tokens in any text artifact |
| `no-code-in-doc-repo` | critical | Reject executable code files in a documentation-only repo |
| `no-direct-provider-call` | critical | Agents MUST NOT call model providers directly; must go through the Kernel |
| `no-hidden-state` | high | All persistent state must flow through SCE or an approved store |
| `context-engine-write-required` | high | Every subsystem mutation must emit to SCE |
| `correlation-id-required` | high | Every event must carry a `correlation_id` |
| `doc-section-skeleton` | warning | Doc files should have the standard section skeleton |
| `rfc2119-language` | warning | Requirements should use MUST/SHOULD/MAY language |
| `cross-link-required` | warning | References to other subsystems should use Markdown links |
| `test-before-feature` | warning | New behaviour should have a failing test in the task graph |
| `budget-cap-declared` | high | Every run must declare token, wall-time, and cost budgets |
| `least-privilege-tools` | high | Workers must declare only the tools they actually use |
| `audit-log-write` | high | All failures must be recorded in the Audit Log |
| `guardian-cannot-bypass` | critical | No code path may skip the Guardian stage |

## Rule Expression DSL

Rules are expressed using a minimal, safe DSL (no Turing-complete scripting):

```
# Pattern matching on artifact content
content_matches(pattern: Regex)
content_not_matches(pattern: Regex)

# File/path predicates
path_matches(glob: Glob)
file_extension_in(exts: string[])
has_front_matter_key(key: string)

# Structural predicates (for code artifacts)
imports(module: string)
exports(symbol: string)
calls(function: string)

# SCE predicates
event_published(topic: string, event_type: string)

# Composite
all(rules: RuleExpr[])
any(rules: RuleExpr[])
not(rule: RuleExpr)
```

Example: the `no-secrets-in-text` rule:

```yaml
id: no-secrets-in-text
severity: critical
scope:
  kinds: ["any"]
expr:
  not:
    content_matches: "(AKIA[0-9A-Z]{16}|sk-[a-zA-Z0-9]{48}|ghp_[a-zA-Z0-9]{36}|-----BEGIN (RSA |EC )?PRIVATE KEY-----)"
```

## Interfaces

```
# Evaluation
guardian.check(artifact: MergedArtifact) → Verdict
guardian.check_batch(artifacts: MergedArtifact[]) → Verdict[]   # parallel evaluation

# Rule management
guardian.rules() → Rule[]
guardian.get_rule(rule_id) → Rule
guardian.reload() → { count: number, errors: RuleError[] }      # hot-reload from disk

# Introspection
guardian.verdicts(filter?) → Verdict[]        # historical verdicts from Audit Log
guardian.explain(verdict_id) → VerdictDetail  # full explanation with rule trace

# Subscribe
guardian.subscribe() → AsyncIterator<Verdict>
```

### Verdict shape

```
Verdict {
  id:           ulid
  artifact_id:  ulid
  run_id:       ulid
  ts:           rfc3339
  ok:           boolean
  violations: {
    rule_id:    string
    severity:   "critical"|"high"|"warning"
    message:    string      # actionable explanation
    location?:  { path, line_start, line_end }
    fix_hint?:  string      # how to resolve
  }[]
  hints: {
    rule_id:    string
    message:    string
  }[]
  impact:       Impact?     # from Impact Analysis
  correlation_id: uuid
}
```

A `Verdict` with `ok: false` has at least one violation with severity `critical` or `high`. Warnings do not prevent delivery but are surfaced in the UI and CLI.

## Evaluation Flow

```
1. Load rules (from cache if hot-reload not triggered)
2. Call impact.analyze(artifact) → Impact (in parallel with rule evaluation)
3. For each rule in rules, evaluate rule.expr against artifact:
   - critical/high rules: evaluate in parallel; collect all violations
   - warning rules: evaluate in parallel; collect all hints
4. If any critical rule engine crashes: add synthetic critical violation "rule_engine_crash:<rule_id>"
5. If any high rule engine crashes: add synthetic high violation + emit alert
6. Assemble Verdict { ok: violations.length == 0, violations, hints, impact }
7. Emit Verdict on SCE guardian.verdicts topic
8. Write to Audit Log
9. Return Verdict to Kernel
```

## Auto-Fix

Rules with an `auto_fix` spec instruct the Guardian to attempt remediation before issuing a veto. The auto-fix runs once; if it does not resolve the violation, the veto is issued:

```
AutoFixSpec {
  kind:    "redact"         # replace matched content with [REDACTED]
         | "remove_file"    # delete the offending file from the artifact
         | "prepend_note"   # prepend a human-visible warning to the artifact
         | "mcp_tool"       # call an MCP tool to fix
  config:  object
}
```

Auto-fixes are audited in the `guardian.verdicts` record as `{ auto_fix_applied: true, fix_kind, before_hash, after_hash }`.

## Requirements

- **MUST** evaluate all `critical` and `high` rules on every `MergedArtifact` before delivery.
- **MUST** fail-closed for `critical` rules: a rule engine crash produces a synthetic critical violation that blocks delivery.
- **MUST** emit `guardian.verdicts` on the SCE for every evaluation.
- **MUST** record every verdict in the [Audit Log](./AUDIT_LOG.md).
- **MUST** include an actionable `message` and optional `fix_hint` for every violation.
- **MUST** reload rules from disk on a `guardian.reload()` call without a process restart.
- **MUST** never be bypassable — the Kernel's delivery stage MUST require an `ok: true` Verdict.
- **SHOULD** run rule evaluation in parallel across all rules to minimise latency.
- **SHOULD** call [Impact Analysis](./IMPACT_ANALYSIS.md) in parallel with rule evaluation and include the result in the Verdict.
- **MAY** attempt auto-fix for rules that declare an `AutoFixSpec` before issuing a veto.
- **MAY** support custom rules loaded from `~/.aidevos/rules/` without code changes.

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| Rule engine crash (critical rule) | Exception in rule evaluator | Fail-closed: synthetic critical violation; block delivery |
| Rule engine crash (high rule) | Exception in rule evaluator | Fail-open: synthetic high violation; block delivery; alert |
| Rule file invalid (parse error) | YAML/DSL parse failure on reload | Keep previous rule set; surface parse error to operator |
| Impact Analysis timeout | Call takes > `impact_timeout_ms` | Proceed without impact in Verdict; emit `guardian.impact_timeout` warning |
| Guardian storm (≥3 vetoes in 10s) | Veto rate counter | Freeze non-critical routes; alert on-call; keep read paths live (see [Main AI Kernel](./MAIN_AI_KERNEL.md)) |
| SCE write failure | Event publish NAK | Buffer verdict; retry; if repeated, halt delivery until flush |

Every failure is recorded in the [Audit Log](./AUDIT_LOG.md) and follows [Error Handling](./ERROR_HANDLING.md).

## Security Considerations

- The Guardian is the system's trust anchor for change safety. Its rule set must be version-controlled alongside the documentation.
- Custom rules loaded from disk are parsed by the DSL engine, never `eval`'d, to prevent rule injection.
- Audit records of verdicts are append-only and cryptographically signed; they cannot be altered after the fact.
- The Guardian's own code paths are excluded from auto-fix — no rule may auto-modify the Guardian's rule files.
- See [Security Model](./SECURITY_MODEL.md).

## Observability

| Metric | Labels | Description |
|--------|--------|-------------|
| `guardian_check_total` | `ok=true\|false` | Verdicts issued |
| `guardian_check_seconds` | — | Evaluation latency histogram |
| `guardian_violation_total` | `rule_id`, `severity` | Violations per rule |
| `guardian_veto_total` | `reason=rule\|engine_crash` | Veto count |
| `guardian_rule_count` | `severity` | Active rule count |
| `guardian_reload_total` | `ok` | Rule reload events |
| `guardian_auto_fix_total` | `rule_id`, `applied` | Auto-fix attempts |

Traces: one span per `guardian.check` call; one child span per rule evaluation. See [Tracing](./TRACING.md).

## Acceptance Criteria

- Submitting an artifact that contains a string matching the AWS access key pattern causes a `critical` violation and blocks delivery.
- Submitting a code file to the documentation-only repository triggers the `no-code-in-doc-repo` critical rule and vetoes the change.
- Adding a new custom rule to `~/.aidevos/rules/my-rule.yaml` and calling `guardian.reload()` makes the rule active on the next `guardian.check` without a process restart.
- A Guardian veto with `severity: critical` forces the Kernel to replan; the second plan must produce a different artifact.
- Issuing three vetoes within 10 s triggers the guardian storm response: new routes are frozen and an alert is raised.

## Open Questions

- Whether the auto-fix feature should require explicit opt-in per run or be on by default for `redact` fixes — tracked in [templates/ADR](../templates/ADR.md).
- Whether warning-only verdicts should be surfaced inline in the UI diff view or only in a separate panel.

## Related Documents

- [Merge Manager](./MERGE_MANAGER.md) — produces the `MergedArtifact` the Guardian evaluates
- [Impact Analysis](./IMPACT_ANALYSIS.md) — called in parallel during evaluation
- [AI Coding Rules](./AI_CODING_RULES.md) — the human-readable policy the Guardian enforces
- [Prompt Governance](./PROMPT_GOVERNANCE.md)
- [Audit Log](./AUDIT_LOG.md)
- [Main AI Kernel](./MAIN_AI_KERNEL.md)
- [System Overview](./SYSTEM_OVERVIEW.md)
- [diagrams/MERGE_GUARDIAN](../diagrams/MERGE_GUARDIAN.md)
