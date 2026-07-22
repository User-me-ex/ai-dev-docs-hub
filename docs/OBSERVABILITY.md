# Observability

> Unified metrics, traces, and logging — OpenTelemetry-based, local-first, with optional cloud export. Every signal carries a correlation_id so the full lifecycle of a run can be reconstructed from any entry point.

## Overview

Observability is built on three pillars — **metrics**, **traces**, and **logs** — atop the [OpenTelemetry](https://opentelemetry.io) SDK. The subsystem is **local-first**: all signals write to stdout and rotating files by default with zero external dependencies. Optional exporters forward data to Prometheus, Jaeger, Tempo, SigNoz, or any OTLP-compatible backend. It supports both **push** (OTLP, stdout) and **pull** (Prometheus /metrics) models. Every subsystem SHALL produce its own metric, trace, and log streams; the Observability subsystem aggregates, correlates, and forwards them. Every event MUST carry subsystem and correlation_id — this single invariant enables cross-subsystem debugging without a centralized store.

## Goals

- **Single unified API** (obs.*) consumed by every subsystem so instrumentation code is identical regardless of backend.
- **Local-first by default**: zero external dependencies; all signals emit to stdout and rotating files.
- **Optional cloud export**: enable Prometheus, OTLP gRPC/HTTP, file shipping via configuration alone.
- **correlation_id everywhere**: every metric, span, and log event carries the correlation ID.
- **Structured logging everywhere**: no freeform text — every log event conforms to LogEvent as NDJSON.
- **Head-based sampling with error amplification**: 10 % default; 100 % for traces with an error.
- **Standardised metric namespace**: idevos_<subsystem>_<name> with required labels.

## Non-Goals

- Replacing the audit log — see [Audit Log](./AUDIT_LOG.md).
- Real-user monitoring or browser instrumentation — server and agent telemetry only.
- Long-term storage or retention — see [Data Retention](./DATA_RETENTION.md).
- Vendor-specific dashboards or alerting.
- Full APM — the subsystem exports signals, it does not host a query plane.

## Architecture

`mermaid
flowchart TB
  subgraph Subsystems
    K[Kernel] R[Router] W[Workers] C[Critic] M[Merge] G[Guardian] P[Planner] RS[Researcher]
  end
  subgraph OTelSDK[OpenTelemetry SDK]
    MTR[Metrics] TRC[Traces] LOG[Logs]
  end
  subgraph Exporters
    STDOUT[Stdout NDJSON] PROM[Prometheus /metrics] OTLP[OTLP gRPC/HTTP] FILE[File rotation]
  end
  Subsystems -->|obs.*| MTR & TRC & LOG
  MTR --> STDOUT & PROM & OTLP
  TRC --> STDOUT & OTLP
  LOG --> STDOUT & FILE & OTLP
  PROM --> PROM_SRV[Prometheus]
  OTLP --> JAEGER[Jaeger/Tempo] & SIGNOZ[SigNoz]
  FILE --> FILES[(Log files)]
  HEALTH[/health :9600] --> Exporters
`

Each pipeline is independently configured. Exporters are hot-reloadable via SIGHUP.

## Metrics

Prometheus exposition format on http://localhost:9600/metrics (pull-only, port configurable). All metrics follow idevos_<subsystem>_<name> — lowercase snake_case.

### Types

| Type | OTel instrument | Behaviour |
|------|----------------|-----------|
| **Counter** | Int64Counter | Monotonically increasing (total requests, errors) |
| **UpDown Counter** | Int64UpDownCounter | Increases and decreases (active runs, queue depth) |
| **Gauge** | Int64Gauge / DoubleGauge | Point-in-time value (memory, temperature) |
| **Histogram** | Int64Histogram / DoubleHistogram | Value distribution (latency, batch size) |

### Required labels

| Label | Type | Description |
|-------|------|-------------|
| subsystem | string | Originating subsystem (snake_case) |
| correlation_id | uuid | Active correlation ID; "" outside a run |

### Common metrics (every subsystem)

| Metric | Type | Description |
|--------|------|-------------|
| idevos_<subsystem>_run_active | UpDown Counter | Currently active runs |
| idevos_<subsystem>_error_total | Counter | Total errors by error_code label |
| idevos_<subsystem>_duration_seconds | Histogram | Wall-clock duration by status (ok\|error); buckets 0.01–300 |

### Per-subsystem examples

| Subsystem | Metrics |
|-----------|---------|
| Kernel | kernel_loop_iterations_total (counter), kernel_loop_duration_seconds (histogram) |
| Router | outer_discovery_seconds (histogram), outer_fallback_total (counter) |
| Workers | worker_task_seconds (histogram), worker_token_total (counter) |
| Critic | critic_verdict_total (counter, label erdict) |
| Guardian | guardian_rule_eval_seconds (histogram), guardian_veto_total (counter) |

### Lifecycle

Metrics register at subsystem init. A metric MUST NOT be deleted or renamed without one minor version deprecation (see [Versioning](./VERSIONING.md)). Deprecated metrics are prefixed deprecated_ for one release cycle.

## Tracing

All traces conform to the [OpenTelemetry Tracing API](https://opentelemetry.io/docs/specs/otel/trace/api/). Each subsystem creates spans via the global TracerProvider.

### Span model

`	ext
Span {
  trace_id:       uuid                   # == correlation_id
  span_id:        ulid                   # time-sortable
  parent_span_id: ulid?                  # nil for root spans
  name:           string                 # low-cardinality
  kind:           SpanKind
  start_time:     rfc3339nano
  end_time:       rfc3339nano
  status:         { code, message? }
  attributes:     map[string] any
  events:         SpanEvent[]
  links:          SpanLink[]
}
`

### Parent-child rules

- Every public API endpoint creates a root span.
- Every Kernel loop iteration creates a root span kernel.loop.iteration.
- Subsystem calls within an iteration are child spans of the Kernel root.
- Worker invocations create a child span under the Kernel span.
- Cross-subsystem calls propagate context via W3C Trace Context (	raceparent).

### Required span attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| subsystem | string | Originating subsystem |
| correlation_id | uuid | Trace ID (duplicated for query convenience) |
| un_id | ulid? | Run ULID if inside a run |

### Sampling (head-based)

| Condition | Rate |
|-----------|------|
| Default | 10 % |
| Any span with status = ERROR | 100 % (overrides) |
| /health or /metrics requests | 0 % |

Per-subsystem overrides:
`yaml
observability:
  traces:
    sampling:
      default: 0.1
      overrides:
        kernel: 1.0
        worker: 0.05
`

### correlation_id as trace_id

Assigned by the Kernel at run start, correlation_id SHALL be the OTel 	race_id, ensuring every signal from a run shares a correlation ID without manual join logic. Outside a run, a new UUID is generated.

## Logging

NDJSON only — no plain-text output.

### Log levels

| Level | Value | Description |
|-------|-------|-------------|
| debug | 0 | Diagnostic; off by default in production |
| info | 1 | Normal operational events |
| warn | 2 | Unexpected but non-fatal |
| error | 3 | Recoverable failures |
| critical | 4 | Unrecoverable; operator intervention required |

### Per-subsystem level configuration

`yaml
observability:
  logs:
    level: info
    overrides:
      kernel: debug
      worker: warn
`

### Sensitive data redaction

Applied at the exporter boundary before emission:

| Pattern | Redaction |
|---------|-----------|
| sk-[a-zA-Z0-9]{20,} | sk-***REDACTED*** |
| Bearer [a-zA-Z0-9._-]+ | Bearer ***REDACTED*** |
| Authorization: .+ | Authorization: ***REDACTED*** |
| password=, secret=, 	oken= | ***REDACTED*** |

Extensible via configuration.

## Log Schema

`	ext
LogEvent {
  ts:             rfc3339
  level:          "debug"|"info"|"warn"|"error"|"critical"
  message:        string                 # Static — no interpolation
  subsystem:      string                 # Matches metrics/traces identifier
  correlation_id: uuid                   # "" outside a run
  run_id?:        ulid
  worker_id?:     ulid
  error?:         { code, message }
  metadata?:      object
}
`

### Invariants

- message MUST be static — dynamic values go in metadata or error.message.
- subsystem MUST match the identifier used in metrics and traces.
- correlation_id MUST be present and non-empty during a run.
- metadata MUST NOT contain sensitive data.
- error MUST be present when level is error or critical.

## Health Check

`	ext
GET http://localhost:9600/health
`

### Readiness vs. liveness

| Mode | Query | Response |
|------|-------|----------|
| **Readiness** (default) | ?probe=readiness | 200 OK when all exporters connected |
| **Liveness** | ?probe=liveness | 200 OK if process running and initialised |

### Response format

`json
{
  "status": "ok" | "degraded" | "unhealthy",
  "probe": "readiness" | "liveness",
  "ts": "2026-07-22T14:30:00.123456789Z",
  "components": {
    "prometheus": { "status": "ok" },
    "otlp": { "status": "degraded", "reason": "endpoint unreachable" }
  }
}
`

### Aggregation

All ok → ok. Any degraded → degraded. Any unhealthy → unhealthy. degraded SHALL NOT cause process exit. Excluded from sampling; no auth when bound to localhost.

## Interfaces

`	ext
obs.metrics(name, value, attributes?) → void
obs.traces.start(name, attributes?) → Span
obs.logs.event(level, message, metadata?) → void
obs.health() → HealthStatus
`

Every subsystem SHALL use these four functions. Direct OTel SDK calls are discouraged — the obs.* wrapper ensures consistent label injection, correlation_id propagation, and redaction.

## Exporters

### stdout (default)

NDJSON to stdout: one line per metric, span, and log event.

### Prometheus

Pull-only /metrics on port 9600 (configurable):

`yaml
observability:
  exporters:
    prometheus:
      enabled: true
      port: 9600
      path: /metrics
`

### OTLP

Exports all signals via OTLP gRPC or HTTP. Compatible: Jaeger, Tempo, SigNoz, OTel Collector.

`yaml
observability:
  exporters:
    otlp:
      enabled: false
      protocol: grpc
      endpoint: http://localhost:4318
      compression: gzip
      timeout: 10s
`

### File rotation

`yaml
observability:
  exporters:
    file:
      enabled: true
      path: /var/log/aidevos/
      rotation:
        max_size_mb: 100
        max_age_days: 7
        max_files: 10
      compress: true
`

File names: idevos-<subsystem>-<YYYY-MM-DD>-<NN>.jsonl.

## Failure Modes

| Mode | Detection | Effect | Response |
|------|-----------|--------|----------|
| Prometheus port unavailable | Bind error | Scraping unavailable | Log warning; continue; degraded |
| OTLP endpoint unreachable | Connection failure | Forwarding blocked | Buffer 5 MB; drain on reconnect; degraded |
| OTLP buffer full | Memory threshold | Drop oldest batches | Log critical; increment idevos_obs_dropped_total |
| Disk full | Write error | Rotation fails | Switch to stdout; unhealthy |
| Sampling overhead > 5 % | CPU monitor | None | Log warn |
| Metric name collision | Duplicate name | Second ignored | Log warn |

All failures emit an SCE event and are recorded in the [Audit Log](./AUDIT_LOG.md).

## Security

- **No PII in log messages**: message is a static template; dynamic values go in metadata and are redacted before emission.
- **Redaction patterns** run at the exporter boundary per the table in Logging. Extensible via config.
- **Access control**: /metrics and /health SHALL bind to 127.0.0.1 by default. When bound to  .0.0.0, operator MUST configure network ACL, basic auth, or mTLS.
- **Audit trail**: changes to observability configuration SHALL be recorded in the Audit Log with the actor identity.

## Acceptance Criteria

- A fresh install produces NDJSON metric, trace, and log lines on stdout for every subsystem action.
- A Prometheus scrape of http://localhost:9600/metrics returns valid exposition with the three common metrics.
- A run with an error produces a trace with status = ERROR recorded even when default sampling is 0 %.
- Every log line conforms to LogEvent — jq 'has("ts") and has("level") and has("subsystem")' passes.
- obs.metrics injects subsystem and correlation_id automatically.
- Configuring OTLP with a valid SigNoz endpoint forwards traces and logs with correlation_id as the trace ID.
- Setting worker: critical log override suppresses worker logs below critical while other subsystems run at default.
- A log line containing sk-abc123... is redacted to sk-***REDACTED***.
- /health?probe=readiness returns degraded when the OTLP endpoint is unreachable.
- A duplicate metric registration does not crash the process.

## Related Documents

[Metrics](./METRICS.md), [Tracing](./TRACING.md), [Logging](./LOGGING.md), [Telemetry](./TELEMETRY.md), [Reasoning Traces](./REASONING_TRACES.md), [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md), [Audit Log](./AUDIT_LOG.md), [Data Retention](./DATA_RETENTION.md), [Configuration](./CONFIGURATION.md), [Security Model](./SECURITY_MODEL.md), [System Overview](./SYSTEM_OVERVIEW.md), [Main AI Kernel](./MAIN_AI_KERNEL.md)
