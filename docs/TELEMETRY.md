# Telemetry

## Overview

Telemetry in AI Dev OS is **opt-in only** and **disabled by default**. No data is collected or transmitted unless the user explicitly enables telemetry.

Telemetry helps the core team understand usage patterns, prioritize features, and fix bugs. It is designed with privacy as a first principle — collected data is minimal, anonymized, and never includes user content.

---

## What Is Collected

When enabled, the following anonymized data points are collected:

| Data Point | Description |
|---|---|
| OS version | e.g. `Windows 10.0.19045`, `macOS 14.5`, `Linux 6.8` |
| AI Dev OS version | e.g. `0.2.0` |
| Command usage counts | e.g. `{ run: 42, config: 15, eval: 7 }` |
| Error types | Aggregated error categories, **not** stack traces or messages |
| Model provider usage | e.g. `{ anthropic: 120, openai: 45 }` — no model names or API keys |
| Session duration | Uptime in seconds (rounded to nearest minute) |

---

## What Is NOT Collected

- Code content, file contents, or project structure
- Prompts sent to or responses from AI models
- Memory contents or knowledge graph data
- Personal information (name, email, IP address — IPs are discarded at ingestion)
- System environment variables
- Network configuration or hostnames

---

## How to Enable / Disable

### Configuration File

```toml
# ~/.config/aidevos/config.toml

[telemetry]
enabled = false  # set to true to opt in
```

### Environment Variable

```bash
# Overrides config.toml
export AIDEVOS_TELEMETRY_ENABLED=true
```

Setting `AIDEVOS_TELEMETRY_ENABLED=false` explicitly disables telemetry even if the config file says `true`.

---

## Data Transmission

Telemetry data is buffered locally and uploaded in periodic batch requests to `telemetry.aidevos.dev/v1/ping`. Batch size: up to 100 events. Frequency: every 60 minutes while the daemon is running.

Transmission uses HTTPS with TLS 1.3. No data is sent to third-party services — the telemetry endpoint is self-hosted by the AI Dev OS project.

If the endpoint is unreachable (network offline, DNS failure), events are queued and retried. The queue caps at 1000 events; older events are dropped (FIFO).

---

## Privacy Guarantees

1. Telemetry is opt-in. No data leaves the machine without user consent.
2. All collected data is anonymized at the source — no PII is ever transmitted.
3. Telemetry payloads are encrypted in transit (TLS 1.3).
4. Data on the telemetry server is retained for 90 days, then aggregated and anonymized permanently.
5. Users can delete their telemetry data by contacting the project team with their installation ID.
6. The telemetry module is open source — inspect it at `src/telemetry/`.

---

## Related Documents

- [Privacy Policy](./PRIVACY.md)
- [Configuration Reference](./CONFIGURATION.md)
- [Environment Variables](./ENV_VARS.md)
