# ADR-0003: Auto-Fix Consent Model

## Status

Proposed

## Context

The [Architecture Guardian](../ARCHITECTURE_GUARDIAN.md) supports an `auto_fix` capability: when a rule violation is detected, the Guardian can attempt to auto-remediate before issuing a veto (e.g. redacting a secret from an artifact). The question is whether auto-fix should be **opt-in per run** (the user consents before any fix is applied) or **on by default** (the Guardian applies fixes automatically and reports what it did).

## Decision

**Auto-fix is opt-in per run.**

1. Auto-fix is disabled by default.
2. A user or agent can opt in by setting `run.governance.auto_fix = true` in the goal specification.
3. When auto-fix is enabled, the Guardian applies all `auto_fix` rules that match the artifact, then re-evaluates the artifact.
4. If the auto-fix resolves all violations, the artifact passes and the fix is recorded in the audit log along with `{ before_hash, after_hash, fix_kind }`.
5. If the auto-fix does not resolve the violation (e.g. the redacted content was critical), the veto is issued with the fix attempt documented.
6. Rules with `severity: critical` never auto-fix — they always veto, even when auto-fix is enabled. This ensures that no automated process can silently modify a critical violation.
7. The `redact` fix kind is the only built-in auto-fix. `remove_file` and `mcp_tool` require explicit operator consent per rule, not per run.

### Why opt-in?

- Auto-fix modifies the user's artifact. Making this opt-in respects user intent — the user should know their artifact will be modified.
- Critical rules should never auto-fix because the stakes are too high (e.g. redacting a secret might give a false sense of security — the secret should have never been there in the first place).
- The first release should be conservative. Auto-fix can become opt-out or default in a future release.
- The consent model gives us operational experience with auto-fix before committing to it as a default behaviour.

## Consequences

**Positive:**
- Respects user intent — no silent modification of artifacts
- Builds trust in the Guardian's auto-fix capability before making it default
- Critical violations always require human attention

**Negative:**
- Most users will not opt in during early releases, reducing the real-world testing surface for auto-fix
- Adds a configuration option (`run.governance.auto_fix`) that must be documented and propagated through the run spec

## Related

- [Architecture Guardian](../ARCHITECTURE_GUARDIAN.md) — auto-fix spec and AutoFixSpec schema
- [Main AI Kernel](../MAIN_AI_KERNEL.md) — RunSpec propagation
