# Compliance

> Compliance considerations for AI Dev OS deployments.

## Overview

AI Dev OS is a local-first development tool. It does not itself process
personal data on a central server, and it does not require a cloud service
to operate. However, compliance with regulations such as GDPR, SOC 2, and
CCPA depends on how the tool is deployed, configured, and used within an
organisation. This document describes the built-in features that support
compliance programmes and the limitations that deployers must address
independently.

## Built-in Compliance Features

| Feature | Description | Relevant Standard |
|---|---|---|
| **Audit log** | Append-only, hash-chained log of all security-relevant operations. Tamper-evident: any modification breaks the hash chain. | SOC 2, GDPR (Art. 30) |
| **RBAC** | Role-based access control with fine-grained capabilities. Every action is tied to an authenticated actor. | SOC 2 (CC6, CC7) |
| **Tenant isolation** | Workspaces are cryptographically isolated. No workspace can read another's data without the corresponding key. | SOC 2 (CC6) |
| **Data retention controls** | Per-category TTLs, automated pruning, and manual flush commands. | GDPR (Art. 5(1)(e)) |
| **Encryption at rest** | AES-256-GCM for memory records, age for config, SQLite cipher for databases. | SOC 2 (CC6) |
| **Encryption in transit** | TLS 1.3 for all remote communication. | SOC 2 (CC6) |
| **Export API** | `workspace export` produces a portable archive of all workspace data in JSON. | GDPR (Art. 20) |
| **Data deletion** | `workspace delete` removes all local data for a workspace. Telemetry data can be deleted on the server side. | GDPR (Art. 17) |

## GDPR Readiness

| Requirement | AI Dev OS Support |
|---|---|
| **Right to be informed** | [Privacy](PRIVACY.md) document describes exactly what data is collected and why. |
| **Right of access** | Users can view all stored data via the CLI or by reading local files directly. |
| **Right to rectification** | All config and memory records are editable. |
| **Right to erasure** | `workspace delete` removes all local data. Telemetry data deletion is handled by the telemetry backend. |
| **Right to data portability** | `workspace export` produces a JSON archive that can be imported into any system. |
| **Data Processing Record** | The audit log serves as a record of processing activities (GDPR Art. 30). Every data access, modification, and export is recorded. |
| **DPA readiness** | AI Dev OS does not act as a processor; the user's chosen model providers and cloud infrastructure providers may require a DPA. |

### Practical Steps for GDPR Compliance

1. Publish the [Privacy](PRIVACY.md) document to end users.
2. Configure telemetry as disabled (default) or obtain explicit consent
   before enabling.
3. Review the model provider's DPA for any remote model APIs used.
4. Set data retention TTLs in `workspace config` to match the
   organisation's retention schedule.
5. Use `workspace export` to fulfil data portability requests.

## SOC 2 Readiness

| Trust Service Criteria | AI Dev OS Support | Gap / Action Required |
|---|---|---|
| **CC6 — Logical and Physical Access** | RBAC, encryption at rest and in transit, OS keychain integration | Deployers must manage OS-level access controls and keychain backup policies |
| **CC7 — System Operations** | Audit log, health check API, metrics endpoint | Deployers must configure monitoring and alerting on audit log anomalies |
| **CC8 — Change Management** | Configuration versioning, workspace snapshots, rollback support | Deployers must establish a change approval process for production workspaces |
| **CC9 — Risk Mitigation** | Vendor risk is limited to model/search providers the user connects | Deployers must assess third-party providers independently |

The audit log is the primary evidence source for SOC 2 examinations. It
should be exported to a SIEM and retained according to the organisation's
retention policy.

## Enterprise Features

| Feature | Availability | Compliance Benefit |
|---|---|---|
| **SSO / SAML** | Via OIDC-compatible identity provider | Centralised access control and user lifecycle management |
| **Audit log export to SIEM** | JSON streaming to stdout or file; syslog forwarding | Real-time monitoring and alerting |
| **Legal hold** | Workspace snapshots can be frozen (read-only) to preserve data for litigation | Prevents spoliation during legal proceedings |
| **Region-specific deployment** | Data stays on local infrastructure; no mandatory cloud dependency | Data residency control |

## Limitations

> **AI Dev OS is a tool. Compliance is a practice.**

1. **Third-party model providers.** When AI Dev OS sends a prompt to a
   remote model provider, that provider processes the data on its own
   infrastructure. The organisation is responsible for ensuring that the
   provider has appropriate safeguards (DPA, data region, etc.). AI Dev OS
   recommends using providers that offer data retention opt-out and
   zero-data-retention APIs.

2. **Search providers.** Web search queries are sent to the configured
   search provider. Organisations in regulated industries should use a
   self-hosted or enterprise search endpoint.

3. **No built-in data classification.** AI Dev OS does not inspect or label
   data for sensitivity. Deployers should implement data classification at
   the policy level (e.g. restrict which workspaces can connect to which
   providers).

4. **Audit log is local by default.** For compliance in multi-user or
   server deployments, the audit log must be forwarded to a central SIEM or
   immutable storage.

## Related Documents

- [Privacy](PRIVACY.md) — what data is collected and transmitted
- [Security Model](SECURITY_MODEL.md) — threat model and trust boundaries
- [Data Retention](DATA_RETENTION.md) — retention policies and
  configuration
- [Audit Log](AUDIT_LOG.md) — audit log schema and export
- [Encryption](ENCRYPTION.md) — encryption at rest and in transit
- [Auth System](AUTH_SYSTEM.md) — RBAC and SSO integration
