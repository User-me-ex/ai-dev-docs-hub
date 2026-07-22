# Environment Variables

> Complete reference for all environment variables consumed by AI Dev OS. This document is normative — implementations MUST support every variable listed below.

## Overview

AI Dev OS reads environment variables at startup for configuration overrides, secrets injection, and platform detection. Variables use the `AIDEVOS_` prefix for first-party settings. Third-party provider API keys use their standard variable names (e.g., `OPENAI_API_KEY`).

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

### Model providers

| Variable | Type | Default | Description | Config equivalent |
|----------|------|---------|-------------|-------------------|
| `AIDEVOS_DEFAULT_PROVIDER` | string | `openai` | Default provider for unclassified requests | `router.default_provider` |
| `OPENAI_API_KEY` | string | — | OpenAI API key | `providers.openai.api_key` |
| `OPENAI_ORG_ID` | string | — | OpenAI organization ID | `providers.openai.org_id` |
| `ANTHROPIC_API_KEY` | string | — | Anthropic API key | `providers.anthropic.api_key` |
| `GOOGLE_API_KEY` | string | — | Google AI / Gemini API key | `providers.google.api_key` |
| `MISTRAL_API_KEY` | string | — | Mistral API key | `providers.mistral.api_key` |
| `GROQ_API_KEY` | string | — | Groq API key | `providers.groq.api_key` |
| `COHERE_API_KEY` | string | — | Cohere API key | `providers.cohere.api_key` |
| `TOGETHER_API_KEY` | string | — | Together AI API key | `providers.together.api_key` |
| `DEEPSEEK_API_KEY` | string | — | DeepSeek API key | `providers.deepseek.api_key` |
| `OPENAI_BASE_URL` | string | `https://api.openai.com/v1` | Custom base URL for OpenAI-compatible APIs | `providers.openai.base_url` |
| `ANTHROPIC_BASE_URL` | string | `https://api.anthropic.com` | Custom base URL for Anthropic | `providers.anthropic.base_url` |

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

## Related Documents

- [Configuration](./CONFIGURATION.md) — config file format, precedence, dynamic reload
- [Secrets Management](./SECRETS_MANAGEMENT.md) — secret storage backends, encryption
- [CLI](./CLI.md) — command-line flags (higher precedence than env vars)
- [Deployment](./DEPLOYMENT.md) — env var injection in Kubernetes, Docker
- [Backend](./BACKEND.md) — startup sequence, env var loading order
- [Observability](./OBSERVABILITY.md) — tracing, logging configuration via env vars
