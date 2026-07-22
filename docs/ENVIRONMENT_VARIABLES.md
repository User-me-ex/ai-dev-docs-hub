# Environment Variables

> Complete reference for all environment variables consumed by AI Dev OS. This document is normative — implementations MUST support every variable listed below.

## Overview

AI Dev OS reads environment variables at startup for configuration overrides, secrets injection, and platform detection. Variables use the `AIDEVOS_` prefix for first-party settings.

**Architecture note:** Provider API keys (e.g., `OPENAI_API_KEY`) are configured within the [Nine Router](NINE_ROUTER_INTEGRATION.md) dashboard, not as AI Dev OS environment variables. The AI Dev OS never reads provider API keys from its environment — Nine Router handles all credential management. The variables listed below under "Model providers" are documented for reference compatibility but are NOT used by the AI Dev OS Kernel directly.

Environment variables take precedence over config file values but are overridden by CLI flags. See [Configuration](./CONFIGURATION.md#precedence) for the full precedence chain.

## Variables Reference

### Core paths

| Variable | Type | Default | Description | Config equivalent |
|----------|------|---------|-------------|-------------------|
| `AIDEVOS_HOME` | string | `~/.aidevos` | Root data directory for databases, config, logs | — |
| `AIDEVOS_CONFIG` | string | `$AIDEVOS_HOME/config.toml` | Path to the user config file | — |
| `AIDEVOS_DB_PATH` | string | `$AIDEVOS_HOME/data/aidevos.db` | SQLite database file path | `backend.db_path` |
| `AIDEVOS_VECTOR_DB_PATH` | string | `$AIDEVOS_HOME/data/vectors/` | Vector index directory (usearch) | `memory.vector_path` |

### Server / backend

| Variable | Type | Default | Description | Config equivalent |
|----------|------|---------|-------------|-------------------|
| `AIDEVOS_BACKEND_HOST` | string | `127.0.0.1` | Bind address for the HTTP/gRPC server | `backend.host` |
| `AIDEVOS_BACKEND_PORT` | integer | `8374` | Server listen port | `backend.port` |
| `AIDEVOS_METRICS_PORT` | integer | `9090` | Prometheus metrics server port | `metrics.port` |
| `AIDEVOS_MAX_WORKERS` | integer | `10` | Maximum concurrent worker agents per pod | `backend.max_workers_per_pod` |
| `AIDEVOS_RUN_TIMEOUT_MS` | integer | `300000` | Maximum wall-clock time for a single agent run (ms) | `backend.run_timeout_ms` |

### Logging and observability

| Variable | Type | Default | Description | Config equivalent |
|----------|------|---------|-------------|-------------------|
| `AIDEVOS_LOG_LEVEL` | enum | `info` | Log level: `trace`, `debug`, `info`, `warn`, `error`, `fatal` | `logging.level` |
| `AIDEVOS_LOG_FORMAT` | enum | `text` | Log format: `text` or `json` | `logging.format` |
| `AIDEVOS_TRACING_ENABLED` | bool | `false` | Enable OpenTelemetry tracing | `tracing.enabled` |
| `AIDEVOS_TRACING_ENDPOINT` | string | `http://localhost:4318` | OTLP HTTP endpoint for trace export | `tracing.endpoint` |
| `AIDEVOS_TELEMETRY_ENABLED` | bool | `true` | Enable anonymous product telemetry | `telemetry.enabled` |

### Nine Router gateway

| Variable | Type | Default | Description | Config equivalent |
|----------|------|---------|-------------|-------------------|
| `AIDEVOS_NINE_ROUTER_ENDPOINT` | string | `http://localhost:20128/v1` | Nine Router API base URL | `nine_router.endpoint` |
| `AIDEVOS_NINE_ROUTER_API_KEY` | string | — | Nine Router API key (if configured) | `nine_router.api_key` |

### Model providers (configured within Nine Router)

> These variables are NOT read by AI Dev OS. They are documented here for reference. Provider API keys and endpoints are configured inside the [Nine Router](NINE_ROUTER_INTEGRATION.md) dashboard at `http://localhost:20128/dashboard`.

| Variable | Type | Description | Scope |
|----------|------|-------------|-------|
| `OPENAI_API_KEY` | string | OpenAI API key | Configure in Nine Router dashboard |
| `ANTHROPIC_API_KEY` | string | Anthropic API key | Configure in Nine Router dashboard |
| `GOOGLE_API_KEY` | string | Google AI / Gemini API key | Configure in Nine Router dashboard |
| `MISTRAL_API_KEY` | string | Mistral API key | Configure in Nine Router dashboard |
| `GROQ_API_KEY` | string | Groq API key | Configure in Nine Router dashboard |
| `COHERE_API_KEY` | string | Cohere API key | Configure in Nine Router dashboard |
| `TOGETHER_API_KEY` | string | Together AI API key | Configure in Nine Router dashboard |
| `DEEPSEEK_API_KEY` | string | DeepSeek API key | Configure in Nine Router dashboard |

### Secrets and security

| Variable | Type | Default | Description | Config equivalent |
|----------|------|---------|-------------|-------------------|
| `AIDEVOS_SECRETS_BACKEND` | enum | `local` | Secrets storage: `local`, `keychain`, `vault` | `secrets.backend` |
| `AIDEVOS_JWT_SECRET` | string | — | JWT signing secret (auto-generated if empty) | `auth.jwt_secret` |
| `AIDEVOS_ENCRYPTION_KEY` | string | — | Key for encrypting secrets at rest (age format) | `secrets.encryption_key` |

### SCE and messaging

| Variable | Type | Default | Description | Config equivalent |
|----------|------|---------|-------------|-------------------|
| `AIDEVOS_SCE_BACKEND` | enum | `sqlite` | SCE broker: `sqlite` or `nats` | `sce.backend` |
| `NATS_URL` | string | `nats://localhost:4222` | NATS server URL (used when SCE backend is NATS) | `sce.nats_url` |
| `NATS_CREDS` | string | — | NATS credentials file path | `sce.nats_creds` |

### Queue and tuning

| Variable | Type | Default | Description | Config equivalent |
|----------|------|---------|-------------|-------------------|
| `AIDEVOS_QUEUE_SOFT_CAP` | integer | `500` | Queue depth that triggers backpressure warning | `queue.soft_cap` |
| `AIDEVOS_QUEUE_HARD_CAP` | integer | `10000` | Queue depth that causes new tasks to be rejected | `queue.hard_cap` |
| `AIDEVOS_WAL_CHECKPOINT_INTERVAL` | integer | `1000` | SCE events between WAL checkpoints | `sce.wal_checkpoint_interval` |
| `AIDEVOS_VECTOR_MAX_MEMORY_MB` | integer | `2048` | Max memory for in-process vector index (MB) | `memory.vector_max_memory_mb` |
| `AIDEVOS_MAX_CONTEXT_TOKENS` | integer | `131072` | Max tokens per context window | `memory.max_context_tokens` |

### Database connection (server mode)

| Variable | Type | Default | Description | Config equivalent |
|----------|------|---------|-------------|-------------------|
| `AIDEVOS_DB_URL` | string | — | Postgres connection string (server mode only) | `backend.db_url` |
| `AIDEVOS_DB_POOL_SIZE` | integer | `10` | Postgres connection pool size (server mode) | `backend.db_pool_size` |
| `AIDEVOS_DB_READ_REPLICA_URL` | string | — | Postgres read replica connection string | `backend.db_read_replica_url` |

### Proxy / network

| Variable | Type | Default | Description | Config equivalent |
|----------|------|---------|-------------|-------------------|
| `HTTP_PROXY` | string | — | HTTP proxy for outbound requests | — |
| `HTTPS_PROXY` | string | — | HTTPS proxy for outbound requests | — |
| `NO_PROXY` | string | — | Comma-separated hosts to bypass proxy | — |
| `AIDEVOS_PROXY_ENABLED` | bool | `false` | Enable proxy support for model API calls | `network.proxy_enabled` |

### Example usage

```bash
# Minimal local setup
export AIDEVOS_HOME="$HOME/.aidevos"
export OPENAI_API_KEY="sk-..."
export ANTHROPIC_API_KEY="sk-ant-..."
export AIDEVOS_LOG_LEVEL="debug"

# Server mode with Postgres
export AIDEVOS_BACKEND_MODE="server"
export AIDEVOS_DB_URL="postgres://user:pass@host:5432/aidevos"
export AIDEVOS_SCE_BACKEND="nats"
export NATS_URL="nats://nats-cluster:4222"

# Production tracing
export AIDEVOS_TRACING_ENABLED="true"
export AIDEVOS_TRACING_ENDPOINT="http://otel-collector:4318"
```

## Variable Loading Sequence

```mermaid
sequenceDiagram
    participant Process
    participant Env as Environment
    participant Config as Config File
    participant CLI as CLI Flags
    Process->>Env: Read AIDEVOS_* vars
    Process->>Config: Load config.toml
    Process->>CLI: Parse CLI flags
    Process->>Process: Merge (CLI > Env > Config > Defaults)
    Process->>Process: Validate variable types & ranges
    Process->>Process: Resolve secret references
    Process->>Process: Build effective configuration
```

## Algorithm: Variable Loading and Resolution

```
function loadConfiguration():
    defaults = loadEmbeddedDefaults()
    config   = readConfigFile(resolvePath("AIDEVOS_CONFIG", "~/.aidevos/config.toml"))
    env      = readEnvironment("AIDEVOS_*")
    cli      = parseCLIFlags()

    effective = merge(defaults, config, env, cli)
    effective = validate(effective)
    effective = resolveSecrets(effective)
    return effective

function resolvePath(varName, fallback):
    if varName is set in env:
        return env[varName]
    return fallback
```

## Scope Resolution

Variables are resolved in the following priority order (highest first):

1. **CLI flags** (`--db-path`, `--log-level`) — override everything.
2. **Environment variables** (`AIDEVOS_*`) — override config file values.
3. **Project-local config** (`{project_root}/.aidevos.toml`) — per-project overrides.
4. **User config** (`~/.aidevos/config.toml`) — user-level defaults.
5. **System config** (`/etc/aidevos/config.toml`) — system-wide defaults.
6. **Embedded defaults** — baked into the binary.

## Secret Masking Flow

```mermaid
flowchart LR
    A[Env Var Read] --> B{Is secret?}
    B -->|Yes - API key| C[Load from keychain]
    B -->|Yes - JWT secret| D[Auto-generate if empty]
    B -->|No| E[Use as-is]
    C --> F[Mask in logs]
    D --> F
    E --> G[Log value]
    F --> H[Store in memory]
    G --> H
    H --> I[Scoped to process lifetime]
```

Variables containing the substrings `KEY`, `SECRET`, `TOKEN`, `PASSWORD`, or `CREDENTIAL` are automatically masked in all log output. The raw value is stored in process memory and never serialized to disk unless explicitly configured.

## Validation Pipeline

Every variable is validated at startup:

```mermaid
flowchart LR
    VAR[Variable] --> TYPE{Type check}
    TYPE -->|string| RANGE{Range check?}
    TYPE -->|integer| MINMAX[Min/max check]
    TYPE -->|enum| ALLOWED[Allowed values check]
    TYPE -->|bool| PARSE[Parse bool]
    RANGE --> PASS([Pass])
    MINMAX --> PASS
    ALLOWED --> PASS
    PARSE --> PASS
    TYPE --> FAIL([Fail with error])
```

Validation failures print a diagnostic message and, for non-critical variables, fall back to the default value. Critical variables (e.g., `OPENAI_API_KEY` when using OpenAI) cause the process to exit with code 3.

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| Unknown variable | Startup warning | Ignored; logged at debug level |
| Type mismatch | Validation failure | Fall back to default; log error |
| Missing required variable | Provider call fails | Emit `config.missing_var` event |
| Invalid enum value | Validation failure | Fall back to default; log warn |
| Integer out of range | Min/max validation | Clamp to range; log warn |
| Secret reference not found | Keychain lookup fails | Fall back to empty; log error |
| Proxy variable conflict | Both `HTTP_PROXY` and `NO_PROXY` | Respect `NO_PROXY` precedence |

## Observability / Metrics

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `env_vars_loaded_total` | Counter | source (env/config/cli/default) | Count of variables loaded per source |
| `env_vars_validation_failures` | Counter | var_name | Variables that failed validation |
| `env_vars_secrets_masked` | Counter | — | Secrets masked in log output |
| `env_vars_resolution_duration_ms` | Histogram | — | Time to resolve all variables |

## Acceptance Criteria

- Every variable listed in the reference table is consumed by the implementation and produces the documented effect.
- Setting `AIDEVOS_LOG_LEVEL=debug` produces debug-level log output regardless of the config file value.
- An API key set via environment variable is masked in all logs and never written to disk.
- Setting an invalid value for `AIDEVOS_LOG_LEVEL` (e.g., `superdebug`) falls back to `info` with a warning.
- The loading sequence respects the documented priority order (CLI > Env > Config > Defaults).

## Related Documents

- [Configuration](./CONFIGURATION.md) — config file format, precedence, dynamic reload
- [Secrets Management](./SECRETS_MANAGEMENT.md) — secret storage backends, encryption
- [CLI](./CLI.md) — command-line flags (higher precedence than env vars)
- [Deployment](./DEPLOYMENT.md) — env var injection in Kubernetes, Docker
- [Backend](./BACKEND.md) — startup sequence, env var loading order
- [Observability](./OBSERVABILITY.md) — tracing, logging configuration via env vars
