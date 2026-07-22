# Docker

> Containerised deployment of AI Dev OS — Docker image, Docker Compose, and multi-service deployment. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

Docker is one deployment option for AI Dev OS, best suited for containerized environments, CI/CD, and production server deployment. For local development, the [npm install](./INSTALLATION.md) is recommended as it's simpler and requires fewer resources.

The Docker image packages the backend server (Kernel, SCE, Memory, API), CLI, and all default configuration into a single container. Docker Compose provides the backend server plus optional companion services (Nine Router, Ollama, Prometheus, Grafana).

## Goals

- Single `docker pull ghcr.io/aidevos/aidevos:latest` gives a fully functional backend.
- Docker Compose file starts the complete stack with one command.
- Images are versioned, signed, and rebuilt automatically by CI on every main branch push.
- The container runs as a non-root user with minimal capabilities.
- Multi-architecture images: `linux/amd64` and `linux/arm64`.

## Non-Goals

- Kubernetes deployment — covered in [Deployment](./DEPLOYMENT.md).
- Runtime performance optimisation for containerised workloads — the binary is identical whether run natively or in a container.
- Implementation code — this repo is documentation-only ([AI Coding Rules](./AI_CODING_RULES.md)).

## Image Structure

```
ghcr.io/aidevos/aidevos:latest
├── /app/aidevos              # Single binary (the entire CLI + server)
├── /app/config.toml          # Default configuration
├── /app/prompts/             # Default prompt templates
├── /app/docs/                # Documentation (mounted for KB ingestion)
├── /data/                    # Volume mount point for persistent data
│   ├── aidevos.db           # SQLite database
│   ├── vector-index/        # ANN index files
│   └── logs/                # Log files
└── /app/docker-entrypoint.sh # Entrypoint script
```

### Base Image

| Aspect | Value |
|--------|-------|
| Base | `debian:bookworm-slim` |
| User | `aidevos` (UID 1001, non-root) |
| Shell | `/bin/bash` |
| WORKDIR | `/app` |
| ENTRYPOINT | `["/app/docker-entrypoint.sh"]` |

### Entrypoint Behaviour

```bash
#!/bin/bash
set -e

# Initialise config if not present
if [ ! -f /app/config.toml ]; then
    cp /app/config.toml.default /app/config.toml
fi

# Initialise database if not present
if [ ! -f /data/aidevos.db ]; then
    /app/aidevos server init --db /data/aidevos.db
fi

# Execute command
exec /app/aidevos "$@"
```

## Dockerfile

```dockerfile
FROM debian:bookworm-slim AS base
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

FROM base AS release
WORKDIR /app
COPY --from=builder /build/aidevos /app/aidevos   # from CI build stage
COPY config/docker.toml /app/config.toml.default
COPY prompts/ /app/prompts/
COPY docs/ /app/docs/

RUN groupadd -r aidevos -g 1001 && \
    useradd -r -g aidevos -u 1001 -d /app -s /bin/bash aidevos && \
    chown -R aidevos:aidevos /app /data

USER aidevos
VOLUME ["/data"]
EXPOSE 8374
ENTRYPOINT ["/app/docker-entrypoint.sh"]
CMD ["server", "start"]
```

## Docker Compose

```yaml
version: "3.8"

services:
  # Required: Nine Router (model gateway)
  nine-router:
    image: ghcr.io/nine-router/nine-router:latest
    container_name: nine-router
    ports:
      - "20128:20128"
    volumes:
      - nine_router_data:/data
    restart: unless-stopped

  # Required: Ollama (local model inference)
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    restart: unless-stopped

  # AI Dev OS backend
  aidevos:
    image: ghcr.io/aidevos/aidevos:latest
    container_name: aidevos
    ports:
      - "8374:8374"    # API + Web UI
      - "9090:9090"    # Metrics
    volumes:
      - aidevos_data:/data
      - ~/.aidevos:/home/aidevos/.aidevos:ro  # Config + credentials
    environment:
      - AIDEVOS_HOME=/data
      - AIDEVOS_LOG_LEVEL=info
      - AIDEVOS_ROUTER_ENDPOINT=http://nine-router:20128
    depends_on:
      - nine-router
      - ollama
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "/app/aidevos", "doctor"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Optional: Prometheus for metric scraping
  prometheus:
    image: prom/prometheus:latest
    container_name: aidevos-prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9091:9090"
    profiles:
      - monitoring
      - full

  # Optional: Grafana for dashboards
  grafana:
    image: grafana/grafana:latest
    container_name: aidevos-grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana-dashboards/:/etc/grafana/provisioning/dashboards/
    profiles:
      - monitoring
      - full

volumes:
  nine_router_data:
  ollama_data:
  aidevos_data:
  prometheus_data:
  grafana_data:
```

### Usage

```bash
# Start basic stack (Nine Router + Ollama + AI Dev OS)
docker compose up -d

# Start full stack (with monitoring)
docker compose --profile full up -d

# Run a one-off command (ensure Nine Router is accessible)
docker run --rm ghcr.io/aidevos/aidevos:latest doctor

# Run with local config
docker run --rm -v ~/.aidevos:/home/aidevos/.aidevos:ro \
  -e AIDEVOS_ROUTER_ENDPOINT=http://host.docker.internal:20128 \
  ghcr.io/aidevos/aidevos:latest run "hello world"
```

## Configuration

The Docker image reads configuration from:

1. `/app/config.toml` (embedded default).
2. Mounted config at `/home/aidevos/.aidevos/config.toml` (user overrides).
3. Environment variables (override both, following [Environment Variables](./ENVIRONMENT_VARIABLES.md) naming).

### Environment Variables for Docker

| Variable | Default | Description |
|----------|---------|-------------|
| `AIDEVOS_HOME` | `/data` | Persistent data directory |
| `AIDEVOS_LOG_LEVEL` | `info` | Log level |
| `AIDEVOS_PORT` | `8374` | HTTP API port |
| `AIDEVOS_METRICS_PORT` | `9090` | Prometheus metrics port |
| `AIDEVOS_DB_PATH` | `/data/aidevos.db` | SQLite database path |
| `AIDEVOS_DB_URL` | — | Postgres connection string (overrides SQLite) |

## Security

- The container runs as non-root user `aidevos` (UID 1001).
- Capabilities are dropped; `--cap-drop=ALL` is recommended in production.
- The `/data` volume is owned by `aidevos` and should not be shared with other containers.
- Credentials are passed via environment variables or mounted config, never embedded in the image.
- The `latest` tag is updated on every main branch build. Pin to a specific version for production (`ghcr.io/aidevos/aidevos:v0.1.0`).

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| Database volume missing | `aidevos server init` runs on every start | Safe; data directory is created if not present |
| Port conflict | `EADDRINUSE` | Log ERROR; suggest `docker compose down` or change port mapping |
| Memory limit hit | Container OOM-killed | Increase Docker memory limit; add `--memory=4g` |
| Config file permissions | Config file not readable by aidevos user | Log ERROR with permission fix hint |
| Postgres not available | Connection refused | Retry with backoff (5 attempts); fail with clear error |

## Acceptance Criteria

- `docker pull ghcr.io/aidevos/aidevos:latest` completes in under 60 seconds on a 100 Mbps connection.
- `docker run --rm ghcr.io/aidevos/aidevos:latest doctor` exits with code 0.
- `docker compose up -d` starts Nine Router, Ollama, and the backend; `curl http://localhost:8374/api/v1/health` returns `{"status":"ok"}`.
- The container passes `docker scout quickview` with zero critical CVEs.
- Stopping and restarting the container preserves data in `/data` (runs, KB entries, router assignments).
- The `--profile full` stack starts Prometheus and Grafana alongside the backend and all services communicate.

## Related Documents

- [CI/CD](./CICD.md) — Docker image build and publish in CI
- [Deployment](./DEPLOYMENT.md) — deployment topologies, Kubernetes support
- [Installation](./INSTALLATION.md) — Docker installation instructions
- [Configuration](./CONFIGURATION.md) — config file reference
- [Environment Variables](./ENVIRONMENT_VARIABLES.md) — env var reference
- [System Overview](./SYSTEM_OVERVIEW.md)
