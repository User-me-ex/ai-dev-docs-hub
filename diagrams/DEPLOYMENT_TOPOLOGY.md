# Deployment Topology

> Supported deployment configurations from single-developer laptop to multi-node production cluster.

## Topology 1 — Single Developer Machine (Default)

The standard installation: one binary, one SQLite database, local models via Ollama, optional cloud provider keys. All model access flows through Nine Router.

```mermaid
flowchart TB
  subgraph DEV_MACHINE["Developer Machine (macOS / Linux / Windows WSL2)"]

    subgraph PROCESS["aidevos-server process"]
      KERNEL[Main AI Kernel]
      SCE_SVC[SCE Broker\nSQLite WAL]
      MEM_SVC[Persistent Memory\nSQLite + usearch]
      QUEUE_SVC[Queue + Scheduler\nSQLite]
      PLUGIN_HOST[Plugin Host]
      MCP_SRV[MCP Server\nstdio + SSE]
      HTTP_SRV[HTTP + WS Server\n127.0.0.1:7700]
    end

    subgraph STORAGE["Local Storage"]
      SQLITE_DB[("~/.aidevos/db.sqlite\nRuns, Tasks, Memory,\nSCE events, Audit Log")]
      VEC_IDX[("~/.aidevos/vectors/\nusearch index")]
      OBJ_STORE[("~/.aidevos/objects/\nBlobs > 10 MB")]
      VAULT[("Workspace vault\nMarkdown files")]
    end

    subgraph GATEWAY["Model Gateway"]
      NR[Nine Router\nlocalhost:20128/v1]
    end

    subgraph LOCAL_MODELS["Local AI (default)"]
      OLLAMA[Ollama\nlocalhost:11434]
      WHISPER[Whisper\nlocal STT]
      PIPER[Piper\nlocal TTS]
    end

    subgraph CLIENTS["Client Surfaces (same machine)"]
      CLI[aidevos CLI\nUnix socket IPC]
      DESKTOP[Desktop Shell\nTauri app]
      WEB_LOCAL[Web Shell\nbrowser → localhost:7700]
    end

    CLIENTS -- IPC/HTTP --> HTTP_SRV
    HTTP_SRV --> KERNEL
    KERNEL -->|"POST /v1/chat/completions"| NR
    NR --> LOCAL_MODELS
    KERNEL --> STORAGE
  end

  subgraph CLOUD_OPT["Cloud Providers (optional, configured in Nine Router)"]
    OAI[OpenAI]
    ANT[Anthropic]
    GOO[Google]
    MIS[Mistral]
  end

  NR -. HTTPS .-> CLOUD_OPT
```

**Characteristics:**
- Zero cloud dependencies in default config.
- All data stays on the local machine.
- Single SQLite file — trivial backup (`cp db.sqlite db.backup.sqlite`).
- Suitable for one developer or a small team sharing a machine.

---

## Topology 2 — Team Server (LAN / VPN)

One shared `aidevos-server` accessible to multiple developers on the same network. NATS replaces SQLite as the SCE backend for concurrent write performance. All model access flows through Nine Router.

```mermaid
flowchart TB
  subgraph SERVER["Team Server (dedicated host)"]

    subgraph AIDEVOS_PROC["aidevos-server"]
      KERNEL2[Main AI Kernel]
      SCE_NATS[SCE Broker\nNATS JetStream]
      MEM_PG[Persistent Memory\nPostgres + pgvector]
      HTTP2[HTTP + WS Server\n0.0.0.0:7700 + TLS]
      MCP2[MCP Server]
    end

    NATS[(NATS / JetStream\ndedicated process)]
    PG[("PostgreSQL\nRunning separately\nor managed Postgres")]

    SCE_NATS --> NATS
    MEM_PG --> PG
  end

  subgraph DEV_A["Developer A"]
    CLI_A[aidevos CLI\n--endpoint https://team-server:7700]
    WEB_A[Web Shell]
  end

  subgraph DEV_B["Developer B"]
    CLI_B[aidevos CLI]
    WEB_B[Web Shell]
  end

  DEV_A -- HTTPS --> HTTP2
  DEV_B -- HTTPS --> HTTP2

  subgraph GATEWAY2["Model Gateway"]
    NR2[Nine Router\nlocalhost:20128/v1]
  end
  KERNEL2 -->|"POST /v1/chat/completions"| NR2

  subgraph LOCAL_AI["Local AI on server"]
    OLLAMA2[Ollama]
  end
  NR2 --> OLLAMA2

  subgraph CLOUD2["Cloud Providers (optional)"]
    OAI2[OpenAI]
    ANT2[Anthropic]
  end
  NR2 -. HTTPS .-> CLOUD2
```

**Characteristics:**
- Multiple concurrent users supported (RBAC via [AuthZ/RBAC](../docs/AUTHZ_RBAC.md)).
- NATS provides high-throughput SCE; Postgres provides concurrent-safe memory.
- TLS required when `listen != 127.0.0.1`.
- Local Ollama keeps most inference on-prem; cloud providers optional.

---

## Topology 3 — Cloud / Multi-Tenant Production

Horizontally scalable deployment with multiple `aidevos-server` replicas, a managed database, managed NATS, and an object store.

```mermaid
flowchart TB
  subgraph CDN_LB["Load Balancer / CDN"]
    LB[HTTPS Load Balancer\nL7, sticky sessions for WS]
  end

  subgraph AIDEVOS_CLUSTER["aidevos-server cluster (N replicas)"]
    S1[Server replica 1]
    S2[Server replica 2]
    SN[Server replica N]
  end

  subgraph MANAGED_INFRA["Managed Infrastructure"]
    PG_CLOUD[("Managed PostgreSQL\n+ pgvector extension")]
    NATS_CLOUD[(NATS / JetStream\nmanaged cluster)]
    S3_OBJ[("Object Store\nS3 / GCS / Azure Blob")]
    REDIS_QUEUE[(Redis\nqueue backend)]
  end

  subgraph GATEWAY["Model Gateway"]
    NR_CLUSTER[Nine Router\nlocalhost:20128/v1]
  end

  subgraph AI_BACKENDS["AI Backends"]
    OLLAMA_FARM[GPU Server Farm\nOllama cluster]
    OAI3[OpenAI]
    ANT3[Anthropic]
  end

  subgraph OBSERVABILITY["Observability Stack"]
    PROM[Prometheus]
    GRAF[Grafana]
    TEMPO[Tempo / Jaeger]
    LOKI[Loki / Elasticsearch]
  end

  LB --> AIDEVOS_CLUSTER
  AIDEVOS_CLUSTER --> MANAGED_INFRA
  AIDEVOS_CLUSTER -->|"POST /v1/chat/completions"| NR_CLUSTER
  NR_CLUSTER --> AI_BACKENDS
  AIDEVOS_CLUSTER --> OBSERVABILITY
```

**Characteristics:**
- Stateless application servers; all state in managed services.
- NATS cluster for SCE; PostgreSQL + pgvector for memory.
- Redis for queue (vs. SQLite in single-node).
- Horizontal scaling: add replicas to handle more concurrent runs.
- See [Deployment](../docs/DEPLOYMENT.md) for full production configuration reference.

---

## Port Reference

| Service | Default port | Protocol | TLS |
|---------|-------------|----------|-----|
| HTTP API | 7700 | HTTP/1.1, HTTP/2 | optional (required for remote) |
| WebSocket | 7700 | WS over HTTP | same as HTTP |
| MCP stdio | — | stdio (local subprocess) | N/A |
| MCP SSE | 7701 | HTTP SSE | optional |
| Ollama | 11434 | HTTP | none (local) |
| llama.cpp | 8080 | HTTP | none (local) |
| NATS | 4222 | TCP | optional |
| PostgreSQL | 5432 | TCP | optional |
| Prometheus | 9090 | HTTP | optional |

## Related Documents

- [Backend](../docs/BACKEND.md)
- [Localhost Architecture](../docs/LOCALHOST_ARCHITECTURE.md)
- [Deployment](../docs/DEPLOYMENT.md)
- [Security Model](../docs/SECURITY_MODEL.md)
- [Auth System](../docs/AUTH_SYSTEM.md)
- [Reliability](../docs/RELIABILITY.md)
- [Scalability](../docs/SCALABILITY.md)
