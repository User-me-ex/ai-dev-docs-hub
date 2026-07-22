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

### Consent Flow

```
User/Agent                         Guardian                       Audit Log
    │                                 │                               │
    │  1. Submit artifact             │                               │
    │ ───────────────────────────────►│                               │
    │                                 │                               │
    │  2. Evaluate rules              │                               │
    │  3. Violations found           │                               │
    │                                 │                               │
    │  4. Check auto_fix flag        │                               │
    │     auto_fix = false? → veto   │                               │
    │     auto_fix = true?  → continue                               │
    │                                 │                               │
    │  5. For each fixable violation: │                               │
    │     a. Is severity = critical?  │                               │
    │        Yes → veto (no fix)     │                               │
    │        No  → continue          │                               │
    │     b. Apply fix_kind          │                               │
    │                                │                               │
    │  6. Re-evaluate artifact       │                               │
    │     Pass → pass + log fix      │                               │
    │     Fail → veto + log fix      │                               │
    │                                │                               │
    │  7. Return result              │                               │
    │ ◄──────────────────────────────│                               │
    │                                │   8. Write audit record       │
    │                                │ ─────────────────────────────►│
    │                                │    { actor, artifact_id,      │
    │                                │      before_hash, after_hash, │
    │                                │      fix_kinds[], result }    │
```

### Consent Persistence Model

Consent can be persisted at multiple levels, with the most specific winning:

```typescript
interface AutoFixConsent {
  level: "run" | "workspace" | "project" | "global";
  scope_id: string;           // run ID, workspace ID, etc.
  enabled: boolean;
  rules_whitelist?: string[];   // Which rules allow auto-fix
  rules_blacklist?: string[];   // Which rules never auto-fix
  fix_kinds_allowed: FixKind[]; // Which fix kinds are permitted
  expires_at?: number;          // Optional TTL for consent
  granted_by: string;           // Who granted the consent
  granted_at: number;
}

type FixKind = "redact" | "remove_file" | "mcp_tool" | "rewrite";

// Resolution order:
// 1. Run-level consent (if present, use it)
// 2. Project-level consent (if present, use it)
// 3. Workspace-level consent (if present, use it)
// 4. Global default (auto_fix = false)

// Merge: if multiple levels exist, run-level rules_blacklist merges with
// project-level rules_whitelist — the most restrictive combination wins.
function resolveConsent(run, project, workspace): AutoFixConsent {
  const levels = [run, project, workspace].filter(Boolean);
  return levels.reduce((merged, level) => ({
    enabled: level.enabled,
    rules_whitelist: intersect(merged.rules_whitelist, level.rules_whitelist),
    rules_blacklist: union(merged.rules_blacklist, level.rules_blacklist),
    fix_kinds_allowed: intersect(merged.fix_kinds_allowed, level.fix_kinds_allowed),
  }));
}
```

### Capability Token Scoping for Auto-Fix

Auto-fix operates under a scoped capability token that limits what the Guardian can modify:

```typescript
interface AutoFixCapabilityToken {
  token_id: string;
  issued_to: "guardian";
  expires_at: number;
  scope: {
    artifact_types: string[];     // e.g. ["code", "config", "docs"]
    max_changes: number;          // Max characters changed
    fix_kinds: FixKind[];         // Allowed fix kinds
    paths_allowlist: string[];    // Glob patterns for writable paths
    paths_denylist: string[];     // Glob patterns for protected paths
  };
  audit_hook: string;             // URL for real-time audit callback
}

// Token validation before each fix:
function validateToken(token, fix): boolean {
  if (token.expires_at < Date.now()) return false;
  if (!token.scope.artifact_types.includes(fix.artifact_type)) return false;
  if (cumulative_changes + fix.change_size > token.scope.max_changes) return false;
  if (!token.scope.fix_kinds.includes(fix.kind)) return false;
  if (!pathMatches(fix.path, token.scope.paths_allowlist)) return false;
  if (pathMatches(fix.path, token.scope.paths_denylist)) return false;
  return true;
}
```

### Denied Action Audit Trail

Every auto-fix action (applied or denied) is recorded:

```typescript
interface AutoFixAuditRecord {
  id: string;
  timestamp: number;
  actor: string;                    // Who submitted the artifact
  guardian_instance: string;        // Which Guardian instance
  artifact_id: string;
  artifact_type: string;
  rule_id: string;                  // Which rule triggered
  rule_severity: "info" | "warning" | "critical";

  // Fix details
  fix_kind: FixKind;
  fix_applied: boolean;
  before_hash: string | null;       // null if fix not applied
  after_hash: string | null;        // null if fix not applied
  fix_content_diff: string | null;  // Unified diff of changes

  // Consent details
  consent_level: "run" | "workspace" | "project" | "global" | "none";
  consent_id: string | null;
  consent_granted_by: string | null;

  // Denied actions
  denial_reason?: string;           // e.g. "severity=critical", "consent_expired",
                                    // "capability_token_exceeded", "rule_blacklisted"

  // Result
  final_verdict: "pass" | "veto" | "fix_attempted_veto";
  veto_reason?: string;             // If final verdict is veto

  // Chain of custody
  guardian_signature: string;       // HMAC of the record
}
```

### Human-in-the-Loop Timeout Behavior

When auto-fix encounters an action requiring operator consent (e.g., `remove_file` or `mcp_tool`), it can request real-time human approval:

```typescript
interface HITLRequest {
  request_id: string;
  artifact_id: string;
  rule_id: string;
  fix_kind: FixKind;
  fix_description: string;
  impact_assessment: string;        // What happens if this fix is applied
  required_role: string;            // "operator" | "admin" | "super_admin"
  timeout_ms: number;               // Max wait time
  escalation_after_ms: number;      // Escalate to next role tier
}

// Timeout behavior:
// timeout_ms (default: 300000 = 5 min):
//   - No response → fix is DENIED, violation is recorded as veto
//   - Reason: assume denial to be safe; operator can re-submit manually
//
// escalation_after_ms (default: 120000 = 2 min):
//   - No response from primary role → notify backup role (next tier)
//   - Backup role can approve (override timeout) or ignore
//
// If HITL is not configured (no operator connected):
//   - All operator-consent actions are DENIED
//   - Artifact is vetoed with clear message: "requires operator consent"
```

### Critical Rules Catalog

The following rules have `severity: critical` and **never auto-fix**, regardless of consent settings:

| Rule ID | Description | Why Never Auto-Fix |
|---------|-------------|-------------------|
| `secret-in-artifact` | Credential or secret value detected in artifact output | Auto-redaction gives false sense of security; root cause must be addressed |
| `file-delete-outside-scope` | Attempt to delete a file outside the workspace | Destructive operation requires human verification |
| `env-mutation` | Attempt to modify environment variables | Could affect other processes; must be intentional |
| `network-exfiltration` | Artifact contains internal URLs or hostnames | Information disclosure risk; every occurrence must be reviewed |
| `dependency-pin-bypass` | Using unpinned dependency version in production config | Supply chain security; must be explicitly approved |
| `exec-code-in-artifact` | Artifact contains executable code in unexpected location (e.g., config file) | Potential injection; always needs human review |
| `permission-escalation` | Artifact requests broader permissions than its declared scope | Least-privilege violation; needs explicit justification |
| `data-destroy-command` | DROP TABLE, rm -rf, or similar in artifact | Destructive; always needs confirmation |

New rules added to this catalog require Security Team approval and are reviewed quarterly.

## Why opt-in?

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
