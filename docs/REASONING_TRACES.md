# Reasoning Traces

## Overview

Reasoning traces capture the internal chain-of-thought of AI agents during task execution. Each trace records the input, intermediate reasoning, and output for every step an agent takes, enabling debugging, audit, and transparency.

Traces are distinct from logs — they capture model-level reasoning rather than system events. They are the primary mechanism for understanding *why* an agent made a particular decision.

---

## Trace Schema

| Field | Type | Description |
|---|---|---|
| `trace_id` | `uuid` | Unique trace identifier |
| `run_id` | `uuid` | Associated run ID |
| `worker_id` | `string` | Agent or worker that generated the trace |
| `step` | `integer` | Step number within the run |
| `input` | `string` | Input prompt or context at this step |
| `reasoning` | `string` | Raw chain-of-thought text from the model |
| `output` | `string` | Agent output (action, tool call, or response) |
| `confidence` | `float` | Model-reported confidence (0.0–1.0), if available |
| `tokens_used` | `integer` | Token count for this step |
| `timestamp` | `datetime` | UTC timestamp of the step |

---

## Trace Levels

| Level | Description |
|---|---|
| `none` | No tracing. Zero overhead, no insight. |
| `concise` | Stores only final reasoning summary and output for each step. Low overhead. |
| `full` | Stores complete input, raw reasoning, and output. Maximum insight, higher overhead. |

Configure via `aidevos.toml`:

```toml
[traces]
level = "concise"  # none | concise | full
```

---

## Storage

Traces are written to a dedicated SCE (System Communication Event) topic under `aidevos.traces.*`. This enables real-time streaming and integration with observability pipelines.

For long-lived debugging, traces may also be persisted to Persistent Memory with session-based retention (default: 7 days). Retention policy is configurable:

```toml
[traces.storage]
backend = "sce"          # sce | memory
retention_days = 7       # 0 = indefinite
```

---

## Viewing Traces

### CLI

```bash
aidevos run show --traces <run-id>
aidevos run show --traces --level full <run-id>
```

### API

```
GET /v1/runs/:id/traces
```

Query parameters: `?level=concise&step_gt=5&limit=50`

---

## Privacy Considerations

Traces may contain sensitive reasoning — including user data, API keys, or internal business logic encountered during agent execution. Access to traces MUST be access-controlled:

- Only run owners and admins can view traces by default.
- Traces are encrypted at rest.
- Trace export should strip sensitive fields (configurable filter).

---

## Performance Impact

Full tracing adds per-step overhead for serialization and I/O. Sampling reduces cost:

```toml
[traces.sampling]
rate = 0.1  # record 10% of all steps
```

Production systems should use `concise` level or sampling unless actively debugging.

---

## Integration with Eval Harness

When an evaluation fails, the corresponding trace is automatically captured and linked to the eval result. This allows developers to replay the agent's reasoning at the point of failure:

```bash
aidevos eval show <eval-id> --trace
```

Failed eval traces are retained even if the retention policy would otherwise expire them.

---

## Related Documents

- [Observability](./OBSERVABILITY.md)
- [Logging](./LOGGING.md)
- [Self-Reflection](./SELF_REFLECTION.md)
- [Agent Communication](./AGENT_COMMUNICATION.md)
