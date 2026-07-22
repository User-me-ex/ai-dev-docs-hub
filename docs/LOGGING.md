# Logging

> Structured, correlation-tagged, level-gated logging subsystem — every event carries subsystem, correlation_id, and a machine-parseable schema. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

The Logging subsystem provides structured, level-gated diagnostic output for every component of AI Dev OS. All logs share a common JSON schema so they can be ingested, filtered, and queried uniformly. The subsystem is local-first (rotating files + stdout) with optional remote forwarding (OTLP, syslog, HTTP).

Logging is distinct from [Metrics](./METRICS.md) (aggregated counters) and [Tracing](./TRACING.md) (request-scoped spans). Logs are discrete events that record *what happened, when, and with what context*.

## Goals

- Every log line is a valid JSON object carrying `subsystem`, `correlation_id`, `level`, `message`, and `ts` at minimum.
- Operators can filter logs by level, subsystem, correlation_id, or any structured field without custom parsing.
- PII and secrets are never written to logs — redaction is enforced at the emitting call-site and verified by the [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md).
- Logs survive process restart via rotating file appenders; remote forwarding is best-effort.
- Zero-allocation hot path for high-frequency events (≥1000/s per subsystem).

## Non-Goals

- Aggregated metrics — use [Metrics](./METRICS.md).
- Request tracing — use [Tracing](./TRACING.md).
- User-facing audit trail — use [Audit Log](./AUDIT_LOG.md).
- Implementation code — this repo is documentation-only ([AI Coding Rules](./AI_CODING_RULES.md)).

## Log Levels

| Level | Numeric | Intended use | Typical volume |
|-------|---------|--------------|----------------|
| `FATAL` | 0 | Process will abort; immediate operator attention required | <1/month |
| `ERROR` | 1 | Operation failed; manual intervention likely needed | <1/hour per subsystem |
| `WARN`  | 2 | Operation degraded or unusual; auto-recovery attempted | <1/minute |
| `INFO`  | 3 | Normal lifecycle events (run started, model assigned, veto issued) | ~1/second |
| `DEBUG` | 4 | Development detail; gated behind `--verbose` or config | configurable |
| `TRACE` | 5 | High-frequency loop events; gated behind `--trace` | up to 1000/s |

Logs at `INFO` and above are **always emitted**. `DEBUG` and `TRACE` are opt-in via configuration or CLI flag. Production deployments SHOULD set the default level to `INFO`.

## Log Record Schema

Every log record is a JSON object written on a single line (no pretty-print):

```json
{
  "ts":          "2026-07-22T14:30:00.123Z",
  "level":       "INFO",
  "subsystem":   "nine-router",
  "correlation_id": "abc-123",
  "message":     "Model discovery completed for provider=openai",
  "fields": {
    "provider":  "openai",
    "models_found": 42,
    "duration_ms": 1234
  },
  "error":       null,
  "stack":       null,
  "run_id":      "run_01J...",
  "node":        "node-1"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ts` | RFC3339 string | yes | Event timestamp with millisecond precision |
| `level` | string | yes | One of `FATAL`, `ERROR`, `WARN`, `INFO`, `DEBUG`, `TRACE` |
| `subsystem` | string | yes | Dot-delimited subsystem path (e.g. `kernel.scheduler`, `nine-router.discovery`) |
| `correlation_id` | UUID v7 string | yes | Propagated from the run that caused this event; `_system` for background jobs |
| `message` | string | yes | Human-readable, parameterised with `{field}` placeholders |
| `fields` | object | no | Structured key-value pairs. Values are strings, numbers, booleans, or null |
| `error` | string | no | Error message. Present only when level is ERROR or FATAL |
| `stack` | string | no | Stack trace. Present only when level is FATAL or when DEBUG+error |
| `run_id` | string | no | ULID of the run producing the log, if applicable |
| `node` | string | no | Cluster node identifier in multi-server deployments |

### Redaction

Fields whose values match patterns defined in [Secrets Management](./SECRETS_MANAGEMENT.md) (e.g. `sk-...`, `AKIA...`, `ghp_...`) are replaced with `"[REDACTED]"` at the serialisation layer, before the log line is written. The [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) `no-secrets-in-text` rule independently verifies that no secrets leak.

## Architecture

```mermaid
flowchart LR
  subgraph Producers
    K[Kernel]
    NR[Nine Router]
    PE[Planning Engine]
    DW[Dynamic Workers]
    AG[Architecture Guardian]
    MM[Merge Manager]
    OTHER[... other subsystems]
  end

  Producers --> API[Logger API\nemit(level, message, fields?)]

  API --> DISP[Dispatcher]
  DISP --> FORMAT[JSON Formatter\n+ redaction]
  FORMAT --> STDOUT[stdout / stderr]
  FORMAT --> ROTATE[Rotating File\n< 100 MB each\nmax 10 files]
  FORMAT --> OTLP[OTLP Exporter\noptional]
  FORMAT --> SYSLOG[Syslog / HTTP\noptional]

  DISP --> SCE[(SCE:\nlogs.* topics\noptional mirror)]
```

The Logging API is a thin facade. Every `emit()` call:

1. Builds the log record with mandatory fields (ts, level, subsystem, correlation_id, message).
2. Applies PII/secret redaction to `fields`.
3. Serialises as single-line JSON.
4. Writes to stdout + rotating file synchronously.
5. Optionally forwards via OTLP, syslog, or HTTP exporters (best-effort, non-blocking).

## Interfaces

```
logger.emit(level, message, fields?, error?)
logger.info(message, fields?)           // shorthand for emit("INFO", ...)
logger.warn(message, fields?)
logger.error(message, fields?, error?)
logger.fatal(message, fields?, error?)  // SHOULD call process.exit after emit

logger.set_level(level)                 // runtime level change
logger.flush()                          // block until all buffered writes complete
logger.subsystem(subsystem) → Logger    // returns scoped logger with pre-filled subsystem
```

The scoped logger returned by `logger.subsystem("nine-router.discovery")` prepopulates the `subsystem` field on every call. All other arguments are forwarded unchanged.

## Log Sampling

High-volume subsystems (Model Discovery polling, SCE event dispatch) MUST support rate-limited sampling:

| Strategy | Trigger | Behaviour |
|----------|---------|-----------|
| **Rate limit** | >100 logs/s at same level | Drop excess; emit periodic "dropped N messages" summary |
| **Burst** | >1000 logs/s at same level | Switch to aggregate counters in [Metrics](./METRICS.md); emit single summary log after burst |
| **Sample** | TRACE level | Configurable sample rate (default 1:100); always keep ERROR+ |

Sampling is implemented at the Dispatcher level, before formatting.

## Logger API by Subsystem

Every subsystem spec in this repo that has a "Logging" or "Observability" section MUST declare its log event schema in a table. Example from the [Main AI Kernel](./MAIN_AI_KERNEL.md):

| Event | Level | Fields | Frequency |
|-------|-------|--------|-----------|
| `run.started` | INFO | `{run_id, goal_preview}` | per run |
| `run.stage_completed` | INFO | `{run_id, stage, duration_ms}` | per stage |
| `run.failed` | ERROR | `{run_id, stage, reason}` | per failure |
| `budget.exhausted` | WARN | `{run_id, budget_type, spent, max}` | per budget type |
| `scheduler.tick` | TRACE | `{runs_active, queue_depth}` | every 100ms |

## Configuration

```
[AIDEVOS_LOG]
level = "INFO"                    # default log level
format = "json"                   # json (default) or text (human-readable, for CLI)
path = "~/.aidevos/logs/"         # log directory
max_size_mb = 100                 # per-file max
max_files = 10                    # rotation count
enable_otlp = false               # OTLP exporter
otlp_endpoint = "localhost:4318"  # OTLP HTTP/gRPC endpoint
```

All config keys are settable via environment variables following the pattern in [Environment Variables](./ENVIRONMENT_VARIABLES.md).

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| Disk full | Write error | Fall back to stdout-only; emit periodic `disk.full` warning |
| OTLP endpoint unreachable | Connection refused | Log once at WARN, then silently skip OTLP forwarding; retry every 60s |
| Rate limiter saturated | Drop rate > 50% | Elevate a WARN with "excessive log volume from <subsystem>" |
| Redaction engine failure | Regex exception | Log the event with a `[REDACTION_FAILED]` tag; never crash the producer |
| File rotation failure | Rename error (Windows: file locked) | Continue writing to current file; retry rotation on next write |

## Performance Budget

| Operation | p99 Target | Measurement |
|-----------|------------|-------------|
| `emit(INFO, "msg", {k: v})` — no OTLP | < 10 µs | Wall-clock |
| `emit(ERROR, "msg", {k: v}, err)` — with stack | < 50 µs | Wall-clock |
| File rotation | < 100 ms | Wall-clock |
| JSON serialisation of 10-field record | < 2 µs | CPU |

## Security Considerations

- Redaction runs before serialisation, before any exporter. No exporter can see unredacted values.
- The `fields` object is filtered through an allow-list of types: strings, numbers, booleans, null. Objects and arrays are rejected at compile-time in typed languages, or stripped at runtime.
- Sensitive subsystem names (e.g. `secrets-management`) SHOULD be aliased in production logs.
- Log files inherit the OS file permissions of the `aidevos` process (default: `600` on Unix, restricted ACL on Windows).

## Acceptance Criteria

- A Python script that reads all log files in `~/.aidevos/logs/` can parse every line as valid JSON and extract `{ts, level, subsystem, correlation_id, message}` without errors.
- Setting `level = "ERROR"` suppresses all WARN/INFO/DEBUG/TRACE output while still writing ERROR and FATAL events.
- An `emit()` call with `fields = {"key": "sk-proj-abc123..."}` produces a log line where the value is `"[REDACTED]"`.
- The OTLP exporter, when configured, forwards a structured log to an OpenTelemetry Collector without data loss under 100 msg/s.
- File rotation creates a new file and preserves 10 rotated files when the current file exceeds 100 MB.

## Related Documents

- [Metrics](./METRICS.md) — aggregated counters and histograms
- [Tracing](./TRACING.md) — request-scoped span traces
- [Observability](./OBSERVABILITY.md) — overall observability architecture
- [Audit Log](./AUDIT_LOG.md) — append-only security audit trail
- [Secrets Management](./SECRETS_MANAGEMENT.md) — redaction pattern source
- [Environment Variables](./ENVIRONMENT_VARIABLES.md) — config key mapping
- [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) — enforces no-secrets-in-text
- [System Overview](./SYSTEM_OVERVIEW.md)
- [Main AI Kernel](./MAIN_AI_KERNEL.md)
