# Benchmarks

> Benchmarking framework for measuring system performance, model quality, and end-to-end task completion in AI Dev OS.

## Overview

The benchmarking subsystem provides a unified framework for measuring and tracking AI Dev OS performance across hardware, model inference, and task execution dimensions. Benchmarks are automated, reproducible, and published alongside releases in the changelog. Every benchmark run emits structured results to the metrics pipeline defined in [Observability](./OBSERVABILITY.md).

## Benchmark Categories

| Category | What It Measures | Tooling |
|---|---|---|
| System performance | Kernel loop, SCE, memory, vector, graph | `aidevos benchmark run system` |
| Model quality | Response quality, instruction following, tool use | `aidevos benchmark run model` |
| Task completion | End-to-end runs, multi-agent coordination, merges | `aidevos benchmark run task` |
| Prompt quality | Prompt adherence, governance rule coverage | `aidevos benchmark run prompt` |

## System Benchmarks

| Benchmark | Metric | Target | Notes |
|---|---|---|---|
| Kernel loop latency | p99 latency per loop iteration | < 5 s | Full intake → deliver cycle |
| SCE publish throughput | events/s sustained | > 1000/s | Single worker, 1 KB events |
| SCE subscribe latency | p99 delivery from publish | < 10 ms | Same-process subscriber |
| Memory query latency | p95 vector + metadata hybrid query | < 100 ms | 10 K vector index |
| Vector index build | seconds per 10 K embeddings | < 30 s | HNSW, efConstruction=200 |
| Vector index query | p99 recall @ 10 | > 0.95 | On standard benchmark set |
| Graph traversal speed | edges traversed / s | > 500 K/s | Adjacency list, depth-first |

## Model Benchmarks

Benchmarks are run against each [Model Provider](./MODEL_PROVIDERS.md) integration and reported per-model.

| Benchmark | Measurement | Method |
|---|---|---|
| Response quality | Human-annotated score (1–5) | Sample 200 prompts from eval suite |
| Instruction following | Binary pass/fail on constraint tests | Eval Harness governance suite |
| Tool use accuracy | Correct tool selection rate | 50 multi-step tool-use scenarios |
| Output consistency | Semantic similarity across 5 runs | Embedding cosine similarity > 0.92 |

Results feed into the [Model Routing Policy](./MODEL_ROUTING_POLICY.md) to select optimal providers per task class.

## Task Benchmarks

| Benchmark | Description | Target |
|---|---|---|
| E2E run completion | Multi-step coding task from intake to merge | > 85% pass rate |
| Multi-agent coordination | 3-agent task with shared context | < 15% coordination overhead |
| Merge correctness | PR merge without conflicts or regressions | > 95% clean merge rate |
| Regression detection | Existing tests still pass after change | 100% |

## Running Benchmarks

```bash
aidevos benchmark run system      # all system benchmarks
aidevos benchmark run model        # all model benchmarks
aidevos benchmark run task         # all task benchmarks
aidevos benchmark run <suite> --verbose   # detailed per-test output
aidevos benchmark list             # list available suites and tests
```

Results are written to `~/.aidevos/benchmarks/<run_id>/` as JSON and can be compared with `aidevos benchmark diff <run_a> <run_b>`.

## Publishing Results

Benchmark results are published with each release:

1. Run `aidevos benchmark run all` on the reference hardware (see Environment below).
2. Results are stored in `benchmarks/<version>/` in the release artifact.
3. A summary table is included in the [Changelog](./CHANGELOG.md) under the release notes.
4. Historical results are queryable via `aidevos benchmark history --suite <suite>`.

## Benchmark Environment Requirements

| Requirement | Specification |
|---|---|
| CPU | 16+ cores, x86_64 / ARM64 |
| RAM | 32 GB minimum |
| GPU | NVIDIA A100 40 GB or equivalent (for model benchmarks) |
| Disk | NVMe SSD, 500 GB free |
| OS | Ubuntu 22.04 or macOS 14+ |
| Python | 3.11+ |

System benchmarks run on CPU only. Model benchmarks require GPU. All benchmarks MUST be run in an isolated environment with no other load.

## Interpreting Results

Benchmark results are reported as JSON with the following structure:

```json
{
  "suite": "system",
  "run_id": "20260722_153042_abc123",
  "environment": { "cpu": "AMD EPYC 7763", "ram_gb": 64, "gpu": "A100-40GB" },
  "tests": [
    { "name": "kernel_loop_latency", "metric": "p99", "value_ms": 3200, "target_ms": 5000, "pass": true }
  ]
}
```

Use `aidevos benchmark diff <run_a> <run_b>` to compare two runs. A regression is flagged when a metric degrades by > 10% from the baseline run.

## Custom Benchmarks

Benchmark suites are defined as YAML files in `benchmarks/suites/`:

```yaml
name: custom_system
tests:
  - name: kernel_loop_latency
    metric: p99
    target_ms: 5000
  - name: sce_publish_throughput
    metric: events_per_second
    target: 1000
```

Run with `aidevos benchmark run custom_system --suite-file ./benchmarks/suites/custom.yaml`.

## Related Documents

- [Eval Harness](./EVAL_HARNESS.md) — structured evaluation of prompt and model outputs
- [Testing Strategy](./TESTING_STRATEGY.md) — unit, integration, and regression tests
- [Performance](./PERFORMANCE.md) — performance characteristics and optimization guide
- [Reliability](./RELIABILITY.md) — fault tolerance and uptime guarantees
- [Observability](./OBSERVABILITY.md) — metrics pipeline and dashboards
- [Model Routing Policy](./MODEL_ROUTING_POLICY.md) — how benchmark results influence provider selection
