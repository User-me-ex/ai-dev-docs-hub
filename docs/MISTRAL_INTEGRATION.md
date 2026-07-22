# Mistral Integration

> Provider adapter implementation guide for the Mistral AI API. Extends [Model Providers](./MODEL_PROVIDERS.md) with Mistral-specific configuration, streaming, embeddings, function calling, and JSON mode.

## Overview

The Mistral adapter connects the Nine Router to Mistral's OpenAI-compatible API. Mistral uses the same `/v1/chat/completions` endpoint and SSE streaming as OpenAI. The adapter supports chat completions, embeddings, function calling, and JSON mode.

## Endpoint Configuration

```
base_url: https://api.mistral.ai/v1
auth:     Authorization: Bearer ${MISTRAL_API_KEY}
```

```yaml
providers:
  mistral:
    api_key: ${MISTRAL_API_KEY}     # resolved via Secrets Management
    base_url: https://api.mistral.ai/v1
```

API keys are provisioned from the [Mistral AI Platform](https://console.mistral.ai/).

## Models

| Model ID | Context | Max output | Pricing (input / output per MTok) | Capabilities |
|----------|---------|-----------|-----------------------------------|--------------|
| `mistral-large-2503` | 128K | 4,096 | $2.00 / $6.00 | Text, function calling, JSON mode |
| `mistral-small-2503` | 32K | 4,096 | $0.10 / $0.30 | Text, function calling, JSON mode |
| `codestral-2505` | 256K | 8,192 | $0.30 / $0.90 | Code gen, FIM, function calling |
| `ministral-3b-2503` | 32K | 4,096 | $0.04 / $0.04 | Lightweight edge inference |

Free-tier models (`open-mistral-7b`, `open-mixtral-8x7b`) are available for experimentation.

## Chat Completions

Request format follows OpenAI's schema:

```
POST /v1/chat/completions
{ "model": "mistral-large-2503", "messages": [...], "stream": true }
```

SSE events follow OpenAI conventions:

```
data: {"choices":[{"index":0,"delta":{"content":"Here"},"finish_reason":null}]}
data: {"choices":[{"index":0,"delta":{"content":" is"},"finish_reason":null}]}
data: [DONE]
```

The adapter **MUST** parse `data:` lines, extract `choices[0].delta.content`, and emit `ChatChunk { type: "token", delta }` per content chunk. On `finish_reason`, emit `ChatChunk { type: "finish" }`. Mistral does not include usage in streamed chunks — estimate via token counting.

## Embeddings API

```
POST /v1/embeddings
{ "model": "mistral-embed", "input": ["Text to embed"] }
```

Response: `{ data: [{ index, embedding: float[] }] }`. Supports batched input (up to 32). Produces 1024-dimensional embeddings.

## Function Calling

Mistral supports the OpenAI function calling schema:

```json
{
  "tools": [{
    "type": "function",
    "function": { "name": "get_weather", "parameters": { ... } }
  }],
  "tool_choice": "auto"
}
```

Tool calls appear in `choices[0].delta.tool_calls`. The adapter **MUST** accumulate by `index` and emit complete `ChatChunk { type: "tool_call" }` only on `finish_reason: "tool_calls"`.

## JSON Mode

Set `response_format: { type: "json_object" }` to constrain output to valid JSON. The adapter **MUST** validate JSON parseability after `[DONE]`.

## Rate Limits

| Tier | Requests/min | Tokens/min |
|------|-------------|------------|
| Free | 30 | 500,000 |
| Pro | 500 | 2,000,000 |
| Business | 2,000 | 10,000,000 |

Parse `x-ratelimit-remaining-*` headers. Mistral returns `Retry-After` in seconds on 429.

## Error Codes

| HTTP | Code | Adapter action |
|------|------|----------------|
| 400 | `bad_request` | Surface validation error |
| 401 | `unauthorized` | Mark `auth_error` |
| 403 | `forbidden` | Mark `auth_error` |
| 404 | `not_found` | Trigger re-discovery |
| 429 | `too_many_requests` | Backoff; mark `rate_limited` |
| 429 | `quota_exceeded` | Mark `quota_exceeded`; billing alert |
| 5xx | Server error | Retry × 3; mark `degraded` |

## Related Documents

- [Model Providers](./MODEL_PROVIDERS.md)
- [Nine Router](./NINE_ROUTER.md)
- [Cost Management](./COST_MANAGEMENT.md)
- [Model Discovery](./MODEL_DISCOVERY.md)
- [Secrets Management](./SECRETS_MANAGEMENT.md)
- [Streaming Responses](./STREAMING_RESPONSES.md)
- [Tool Calling](./TOOL_CALLING.md)
- [Local Models](./LOCAL_MODELS.md)
