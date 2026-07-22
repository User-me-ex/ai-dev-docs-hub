# Ollama Integration

> Provider adapter implementation guide for Ollama's local API. This document extends [Model Providers](./MODEL_PROVIDERS.md) with Ollama-specific configuration, model management, API interaction patterns, and performance tuning for local inference. Implementations MUST satisfy every MUST clause below.

## Overview

The Ollama adapter connects the Nine Router to a local Ollama instance, providing zero-cost inference with no external API dependencies. Ollama serves models over a local HTTP REST API, manages model lifecycle (pull, list, delete), and supports both its native `/api/chat` and an OpenAI-compatible `/v1/chat/completions` endpoint.

## Endpoint Configuration

```
base_url: http://127.0.0.1:11434  (default; configurable)
```

```typescript
interface OllamaConfig {
  base_url: string            // default: "http://127.0.0.1:11434"
  default_keep_alive: string  // default: "5m"
  request_timeout: number     // default: 300000 (5 min for model loads)
  max_concurrent: number      // default: 1
  auto_pull: boolean          // default: false; pull missing models automatically
}
```

The adapter **MUST** default to `http://127.0.0.1:11434` when `base_url` is not configured.

## Model Management

### List models

```
GET /api/tags
→ { models: [{ name, model, size, digest, details: { family, parameter_size } }] }
```

Normalize: `id = "ollama/<name>"`, `provider = "ollama"`, `family = details.family`.

### Pull model

```
POST /api/pull { "name": "llama3.2", "stream": false }
→ { "status": "success" }
```

With `stream: true`, the response emits progress lines:
```
{"status":"pulling manifest"}
{"status":"downloading", "digest":"sha256:...", "total":4720987342, "completed":1048576}
{"status":"success"}
```

### Delete model

```
DELETE /api/delete { "name": "llama3.2" }
→ 200 OK
```

The adapter **SHOULD** remove deleted models from its cache and trigger re-discovery.

## Chat API

Ollama's native chat endpoint uses NDJSON streaming.

### Request

```
POST /api/chat
{
  "model": "llama3.2",
  "messages": [{ "role": "user", "content": "Hello" }],
  "stream": true,
  "tools": [...],        // Ollama 0.2+
  "options": { "num_ctx": 4096, "temperature": 0.7 }
}
```

### Response stream

Each line is a complete JSON object. The `done: true` line carries final metadata:

```
{"model":"llama3.2","message":{"role":"assistant","content":"Hello"},"done":false}
{"model":"llama3.2","message":{"role":"assistant","content":"!"},"done":false}
{"model":"llama3.2","message":{"role":"assistant","content":""},"done":true,
 "total_duration":...,"eval_count":2,"eval_duration":...,"prompt_eval_count":5}
```

The adapter **MUST** accumulate messages until `done: true` and emit `ChatChunk` events for each `content` delta, then a final `ChatChunk { type: "finish" }` with usage from `eval_count` and `prompt_eval_count`.

Ollama also exposes `/v1/chat/completions` (OpenAI-compatible). The adapter **SHOULD** prefer the native endpoint for full metadata but **MAY** use the OpenAI-compatible endpoint for generic tools.

## Embeddings API

```
POST /api/embeddings { "model": "llama3.2", "prompt": "Text to embed" }
→ { "embedding": [0.012, -0.034, ...] }
```

Normalize to `float32[][]`. Ollama does not support batched embedding natively — the adapter **SHOULD** send concurrent single-prompt requests.

## Tool Calling

Ollama 0.2+ supports the OpenAI tools schema:

```typescript
tools: [{
  type: "function",
  function: { name: "get_weather", description: "...", parameters: { ... } }
}]
```

Tool calls appear in `message.tool_calls`:

```typescript
"message": { "role": "assistant", "content": "", "tool_calls": [
  { "function": { "name": "get_weather", "arguments": { "location": "London" } } }
]}
```

The adapter **MUST** map `tool_calls` to `ChatChunk { type: "tool_call", id, name, arguments }`. Tool support varies by model family — the adapter **SHOULD** check `details.family` (Llama 3.1+ and Mistral families have reliable tool calling).

## Modelfile and Custom Models

Ollama uses Modelfiles to define custom models:

```
FROM llama3.2
SYSTEM You are a helpful assistant specialized in code review.
PARAMETER temperature 0.2
PARAMETER num_ctx 8192
```

Create via API:

```
POST /api/create { "name": "code-reviewer", "modelfile": "FROM llama3.2\n..." }
→ {"status":"success"}
```

Custom models appear in `/api/tags` and **MUST** be treated identically to pulled models.

## Health Check

The adapter **MUST** implement `health()` using two probes:

| Probe | Endpoint | Success | Failure |
|-------|----------|---------|---------|
| Connectivity | `GET /api/version` | 200 + version string | Connection refused or timeout |
| Model availability | `GET /api/tags` | 200 with `models` array | Timeout |

The adapter **SHOULD** cache the model list with a 30-second TTL to avoid hammering `/api/tags`.

## Model Availability Detection

Models listed in `/api/tags` are available. When a model is requested but absent:

1. Emit `model_unavailable` on the SCE.
2. If `auto_pull` is `true`, invoke `/api/pull` and wait.
3. If `auto_pull` is `false` or pull fails, surface the error for fallback.

## Concurrent Request Handling

| Scenario | Behavior | Adapter strategy |
|----------|----------|-----------------|
| Same model, concurrent | Ollama queues internally | Queue up to `max_concurrent`; reject excess with `degraded` |
| Different models, sequential | Model swapped on each request | Warn on SCE; match `num_ctx` to smallest available |
| Model not loaded | First request loads model (5-30 s) | Set timeout to 300 s for first request; emit `status: "loading"` |

The adapter **MUST** maintain a per-model semaphore (default `max_concurrent: 1`). Additional requests **MUST** be queued with a timeout.

## Performance Considerations

| Factor | Impact | Recommendation |
|--------|--------|---------------|
| GPU vs CPU | GPU offers 10-50× speedup | Default GPU; fall back to CPU if CUDA/Vulkan unavailable |
| Context size (`num_ctx`) | Larger = linear latency increase | Set per-model based on VRAM; default 2048 |
| Quantization | Q4_K_M offers ~4× memory reduction vs FP16 | Prefer `:latest` tags (usually Q4_K_M) |
| Keep-alive | Keeps model loaded (no reload latency) | Set `keep_alive: "5m"`; extend during active workloads |
| Model swaps | Different models trigger reload | Avoid concurrent requests to different models |

The adapter **SHOULD** expose `eval_duration` and `load_duration` as performance metrics.

## Security Considerations

- Ollama requires no authentication by default. The adapter **MUST** warn if `base_url` is not `127.0.0.1` with no auth configured.
- Remote deployments **SHOULD** use TLS and a reverse proxy with authentication.
- Modelfiles are user-defined code. The adapter **MUST NOT** execute untrusted Modelfiles without review.
- The adapter **MUST** restrict access to localhost unless explicitly configured otherwise.

## Related Documents

- [Model Providers](./MODEL_PROVIDERS.md) — parent specification
- [Local Models](./LOCAL_MODELS.md) — model selection and quantization guidance
- [Nine Router](./NINE_ROUTER.md) — fallback routing when Ollama is offline
- [Model Discovery](./MODEL_DISCOVERY.md) — model normalization pipeline
- [Performance](./PERFORMANCE.md) — GPU/CPU inference benchmarks
- [Cost Management](./COST_MANAGEMENT.md) — local inference has zero per-token cost
- [Streaming Responses](./STREAMING_RESPONSES.md) — end-to-end streaming contract
