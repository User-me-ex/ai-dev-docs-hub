# Configuration

> Configuration system for AI Dev OS — sources, formats, validation, dynamic reload, and programmatic interfaces. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

AI Dev OS reads configuration from up to five sources merged in order of precedence. Every config key is validated against a JSON Schema at startup. Changes to supported keys are applied dynamically without restart through file watching or SIGHUP.

The config system exposes a read/write API via the CLI and the Kernel, and publishes change events on the SCE for any subsystem that needs to react.

## Config File Locations

| Source | Path | Scope | Created by |
|--------|------|-------|------------|
| User config | `~/.aidevos/config.toml` | All projects on this machine | `aidevos init` |
| Project config | `{project_root}/.aidevos.toml` | Single project | User or template |
| System config | `/etc/aidevos/config.toml` | All users on this machine | Package install |
| Default config | Embedded in binary | Fallback | Build time |

Project config files are discovered by walking up from `CWD` looking for `.aidevos.toml`. The first found ancestor is used. If none is found, only user + system + defaults apply.

## Config File Format (TOML)

All config files use [TOML](https://toml.io/en/v1.0.0) v1.0.0:

```toml
[backend]
mode = "server"
host = "127.0.0.1"
port = 8374

[providers.openai]
api_key = "${OPENAI_API_KEY}"    # $VAR or ${VAR} substitution
model = "gpt-4o"

[router]
strategy = "latency_optimized"
fallback_enabled = true

[auth]
method = "jwt"
jwt_secret = "${AIDEVOS_JWT_SECRET}"
session_ttl_minutes = 480

[logging]
level = "info"
format = "json"
output = "stdout"

[tracing]
enabled = false
endpoint = "http://localhost:4318"

[metrics]
port = 9090
enabled = true

[memory]
max_records_per_workspace = 100000
vector_dimensions = 768

[queue]
soft_cap = 500
hard_cap = 10000

[sce]
backend = "sqlite"
wal_checkpoint_interval = 1000
```

Variable substitution (`${VAR}`) resolves from environment variables at load time. Literal `$` is escaped as `$$`.

## Config Sections

| Section | Required | Description | Dynamic reload |
|---------|----------|-------------|----------------|
| `[backend]` | Yes | Server mode, host, port, gRPC settings | Host/port: restart required |
| `[providers.*]` | No | Model provider definitions (one per provider) | Yes |
| `[router]` | No | Model routing strategy, fallback settings | Yes |
| `[auth]` | No | Auth method, JWT secret, session TTL | Session TTL: yes. Method: restart required |
| `[logging]` | No | Log level, format, output destination | Level and format: SIGHUP |
| `[tracing]` | No | Tracing enablement, OTLP endpoint | Enabled: yes. Endpoint: restart required |
| `[metrics]` | No | Prometheus metrics port, enablement | Yes |
| `[memory]` | No | Memory record limits, vector dimensions | dims: restart required |
| `[queue]` | No | Queue backpressure thresholds | Yes |
| `[sce]` | No | SCE backend selection, checkpoint tuning | Backend: restart required |
| `[secrets]` | No | Secrets backend (local/keychain/vault) | Restart required |
| `[rate_limiting]` | No | Per-provider rate limit overrides | Yes |
| `[plugins.*]` | No | Plugin enablement and per-plugin config | Yes |

Each section has a corresponding JSON Schema that validates types, ranges, and required fields. See [Default config template](#default-config-template) for full schema.

## Environment Variable Overrides

Every config key can be overridden by an environment variable with the `AIDEVOS_` prefix:

| Config path | Env var | Example |
|-------------|---------|---------|
| `backend.host` | `AIDEVOS_BACKEND_HOST` | `AIDEVOS_BACKEND_HOST=0.0.0.0` |
| `backend.port` | `AIDEVOS_BACKEND_PORT` | `AIDEVOS_BACKEND_PORT=8374` |
| `logging.level` | `AIDEVOS_LOG_LEVEL` | `AIDEVOS_LOG_LEVEL=debug` |
| `sce.backend` | `AIDEVOS_SCE_BACKEND` | `AIDEVOS_SCE_BACKEND=nats` |

Mapping: `section.key` → `AIDEVOS_{SECTION}_{KEY}` with uppercase and dots replaced by underscores. Conflicting env vars at startup produce a warning and the env var wins.

## Precedence (highest to lowest)

```
CLI flag (--log-level=debug)
  → Environment variable (AIDEVOS_LOG_LEVEL=debug)
    → Project config (.aidevos.toml in project root)
      → User config (~/.aidevos/config.toml)
        → System config (/etc/aidevos/config.toml)
          → Built-in defaults
```

A key set at a higher precedence completely shadows lower-precedence values — no merging occurs within a single key. However, sections are merged at the key level: setting `AIDEVOS_LOG_LEVEL=debug` does not affect `AIDEVOS_LOG_FORMAT`.

## Dynamic Reload

### SIGHUP for log levels

Sending `SIGHUP` to the aidevos-server process triggers a re-read of the `[logging]` section:

```
kill -HUP $(pidof aidevos-server)
```

This changes the active log level and format without restarting. The change is logged at the old log level as `config.logging_reloaded`. On Windows, use `aidevos config reload` instead.

### Config file change detection

The config system watches the active config file for inotify / `ReadDirectoryChangesW` events. When a change is detected:

1. The modified file is re-parsed and validated.
2. Keys marked as "dynamic reload: yes" are applied in-place.
3. Keys marked as "dynamic reload: restart required" are queued and applied at next restart.
4. A `config.changed` event is published on the SCE with the list of changed keys.

Change debouncing: 500 ms debounce window prevents rapid-reload storms.

## Config Validation

Validation runs at startup for all config sources, and per-change for dynamic reloads:

| Check | What it validates | Error behavior |
|-------|-------------------|----------------|
| Schema conformance | All values match their declared type, enum, or pattern | Startup: fatal. Reload: revert change, log error |
| Required fields | `[backend]` mode, host, port present | Fatal at startup |
| Range checks | Port in 1–65535, rate limits >= 0, etc. | Fatal at startup |
| Provider uniqueness | No duplicate `[providers.*]` section names | Fatal at startup |
| File permissions | Config file not world-writable | Warning (unless dev mode) |
| Secret references | `$VAR` refs resolve to non-empty env vars | Warning on empty; fatal if required field |
| TOML parse | Well-formed TOML v1.0.0 | Fatal with parser error detail |

Validation errors produce structured output with file path, line number, and expected vs actual values.

## Default Config Template

```toml
[backend]
mode = "local"               # "local" | "server"
host = "127.0.0.1"
port = 8374
max_workers_per_pod = 10
max_workers_per_group = 100
worker_concurrent_runs = 3

[router]
strategy = "latency_optimized"
fallback_enabled = true
provider_timeout_seconds = 120

[auth]
method = "jwt"
session_ttl_minutes = 480
token_refresh_minutes = 60

[logging]
level = "info"
format = "text"              # "text" | "json"
output = "stdout"
include_correlation_id = true

[tracing]
enabled = false
endpoint = "http://localhost:4318"
sample_rate = 0.1

[metrics]
port = 9090
enabled = true

[memory]
max_records_per_workspace = 100000
vector_dimensions = 768
vector_max_memory_mb = 2048
context_memory_mb = 512
max_context_tokens = 131072

[queue]
soft_cap = 500
hard_cap = 10000
claim_timeout_seconds = 30
visibility_timeout_seconds = 120

[sce]
backend = "sqlite"           # "sqlite" | "nats"
wal_checkpoint_interval = 1000
partition_count = 64

[secrets]
backend = "local"            # "local" | "keychain" | "vault"

[rate_limiting]
enabled = true
default_rpm = 100

[telemetry]
enabled = true
instance_id = ""
```

## Interfaces

### `config.get(key: str) -> ConfigValue`

Returns the resolved value for a key, applying the full precedence chain. Throws `ConfigKeyError` if the key is unknown.

```
config.get("backend.port")    → 8374
config.get("nonexistent.key") → ConfigKeyError
```

### `config.set(key: str, value: ConfigValue, persist: bool = False)`

Sets a runtime override. If `persist=True`, writes to the active user config file.

### `config.list(prefix: str = "") -> dict`

Returns all key-value pairs under an optional prefix.

### `config.watch(key_pattern: str) -> AsyncIterator[ConfigChange]`

Returns an async iterator yielding `ConfigChange(key, old_value, new_value)` for matching key changes.

## Failure Modes

| Failure | Detection | Response |
|---------|-----------|----------|
| Config file not found | Startup: log warning, use defaults | Continue with built-in defaults |
| Config file unreadable (permissions) | Startup: log error | Fatal; exit with code 2 |
| TOML parse error | Startup: log error with line number | Fatal; exit with code 2 |
| Schema validation failure | Startup: log error with expected vs actual | Fatal; exit with code 2 |
| Dynamic reload parse error | Runtime: log error | Revert to previous config; emit `config.reload_failed` |
| Environment variable missing for `$VAR` | Startup: log warning | Use empty string; error if required field |
| Config file watch limit exceeded | Runtime: log warning | Fall back to periodic poll (30 s interval) |
| Circular `$VAR` reference | Startup: log error | Fatal; exit with code 2 |

## Related Documents

- [Environment Variables](./ENVIRONMENT_VARIABLES.md) — full env var reference, `AIDEVOS_*` prefix
- [CLI](./CLI.md) — command-line flags and their config equivalents
- [Deployment](./DEPLOYMENT.md) — environment-specific config, secrets injection
- [Secrets Management](./SECRETS_MANAGEMENT.md) — secrets backend configuration
- [Backend](./BACKEND.md) — startup sequence, config loading order
- [Observability](./OBSERVABILITY.md) — config change events, metrics, tracing
- [Feature Flags](./FEATURE_FLAGS.md) — runtime feature gating
