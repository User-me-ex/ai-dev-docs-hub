# Privacy

> What data AI Dev OS collects, stores, and transmits — and what it does
> not.

## Overview

AI Dev OS is built on a **local-first** architecture. All core
functionality — memory, research, automation — executes entirely on the
user's machine and works with zero data egress. Telemetry and crash
reporting are opt-in and disabled by default. When external services are
used (model providers, search APIs), the data sent is limited to what the
user's action requires and is always gated by explicit configuration.

## Data Collected

| Category | Collected By Default | Stored Where | Description |
|---|---|---|---|
| Configuration | Yes | Local disk (`~/.aidevos/`) | User settings, workspace definitions, provider configs |
| Memory records | Yes | Local SQLite | Encrypted workspace knowledge base |
| Research artifacts | Yes | Local disk | Files downloaded or generated during research sessions |
| Logs (debug + info) | Yes | Local disk (`~/.aidevos/logs/`) | Rota­ted, retained 7 days by default |
| Telemetry metrics | Opt-in | Remote (if configured) | Aggregated usage counters — no personal identifiers |
| Crash reports | Opt-in | Remote (if configured) | Stack trace, system info, log tail — no workspace content |
| Command autocomplete history | Yes | Local SQLite | Only stores command text, scoped to local machine |

## Data NOT Collected

AI Dev OS **does not** collect:

- Keystrokes or mouse events outside of explicit UI actions (e.g. clicking
  a button, submitting a prompt)
- Screen content or screenshots (except when the user explicitly invokes a
  screenshot-taking action, e.g. `aidevos screen`)
- Files outside the configured workspace directories
- Network traffic not directed at AI Dev OS
- Browser history, email, or calendar data (unless the user explicitly
  connects a plugin for that purpose)
- Biometric data or location data (except as exposed by the OS to the
  process, e.g. time zone)

## Local-First Guarantee

| Capability | Works Offline |
|---|---|
| Memory read/write | Yes |
| Research document processing | Yes |
| Code analysis | Yes |
| Script execution | Yes |
| Plugin runtime (local-only plugins) | Yes |
| Model inference (local models) | Yes |
| Model inference (remote models) | No — requires network |
| Web search | No — requires network |
| Telemetry export | No — opt-in, only when configured |

No data ever leaves the machine unless the user has explicitly configured a
remote model provider, search provider, or telemetry endpoint.

## Data Retention

| Category | Default Retention | Configurable |
|---|---|---|
| Configuration | Until workspace deleted | Yes |
| Memory records | Until workspace deleted | Yes (per-record TTL) |
| Research artifacts | Until workspace deleted | Yes |
| Debug logs | 7 days rolling | Yes (max_days, max_size) |
| Telemetry metrics | 90 days on server | By provider |
| Crash reports | 180 days on server | By provider |
| Command history | 30 days | Yes (history_ttl_days) |

Users can delete any data category at any time:

```
aidevos workspace flush --memory
aidevos workspace flush --logs
aidevos workspace delete
```

## User Rights

1. **Export** — `aidevos workspace export` produces a portable archive of
   all workspace data in plaintext (JSON + file copies).
2. **Delete** — `aidevos workspace delete` removes all local data for a
   workspace. Remote telemetry data can be deleted by contacting the
   administrator of the telemetry endpoint.
3. **Opt out** — telemetry is off by default. If enabled, it can be
   disabled at any time via `config set telemetry.enabled false` or by
   removing the `telemetry` block from the config file.
4. **View** — all locally stored data is accessible as plaintext files or
   via the CLI (`aidevos memory list`, `aidevos config show`).

## Third-Party Data Sharing

AI Dev OS does not sell or share personal data. Data is transmitted to
third parties only when the user explicitly configures a provider:

| Third Party | Data Sent | Trigger | User Control |
|---|---|---|---|
| Model provider (e.g. OpenAI, Anthropic) | Prompt text, optional file attachments | User submits a prompt | API key required; provider choice is configurable |
| Search provider (e.g. Bing, Google) | Query string | User runs a web search | Must be enabled in config; each provider has its own toggle |
| Telemetry endpoint | Aggregated counters | Telemetry enabled in config | Opt-in; off by default |
| Crash reporter | Stack trace, system info | Crash occurs + user approval | Opt-in prompt on first crash |

All remote API calls are documented in the audit log. Users can review
exactly what was sent and when.

## Compliance

| Regulation | AI Dev OS Readiness |
|---|---|
| **GDPR** | Local-first architecture means minimal personal data processing. See [Compliance](COMPLIANCE.md) for data export, deletion, and DPA guidance. |
| **CCPA** | No sale of personal information. Users can request deletion of any remotely stored data. |

For compliance in regulated deployments, refer to the [Compliance](COMPLIANCE.md)
document and consult your organisation's Data Protection Officer.

## Related Documents

- [Security Model](SECURITY_MODEL.md) — threat model and trust boundaries
- [Data Retention](DATA_RETENTION.md) — detailed retention policies and
  configuration
- [Compliance](COMPLIANCE.md) — GDPR, SOC 2, and enterprise compliance
  features
- [Encryption](ENCRYPTION.md) — how data is protected at rest and in
  transit
