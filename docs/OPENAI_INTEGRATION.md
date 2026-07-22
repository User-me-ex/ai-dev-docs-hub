# OpenAI Integration

> Provider adapter implementation guide for the OpenAI and Azure OpenAI APIs. This document extends [Model Providers](./MODEL_PROVIDERS.md) with OpenAI-specific configuration, error handling, streaming behavior, and capabilities mapping. Implementations MUST satisfy every MUST clause below.

## Overview

The OpenAI adapter connects the Nine Router to OpenAI's API and any OpenAI-compatible endpoint (Azure OpenAI, Together AI, Groq, Fireworks). It implements the standard `ProviderAdapter` interface with additional logic for token accounting, response format negotiation, and rate-limit backoff.

## Endpoint Configuration

### Standard OpenAI

```
base_url: https://api.openai.com/v1
auth:     Authorization: Bearer ${OPENAI_API_KEY}
```

### Azure OpenAI

When `openai.use_azure` is `true`, the adapter switches to the Azure endpoint format:

```
base_url: https://{resource}.openai.azure.com/openai/deployments/{deployment}
auth:     api-key: ${AZURE_OPENAI_API_KEY}
api-version: 2024-10-01-preview
```

| Behavior | Azure | Standard |
|----------|-------|----------|
| Model name | Deployment name (not model ID) | Model ID string |
| Auth header | `api-key` | `Authorization: Bearer` |
| Token counting | `prompt_tokens` / `completion_tokens` only | Same + `cached_tokens` on gpt-4o |
| Rate limits | Per-deployment TPM/RPM (Azure portal) | Per-organization tier |
| Regional availability | Per-region deployment | Global |

The adapter **MUST** map Azure deployment names to canonical model IDs via a configured `deployment → model_id` mapping in [Configuration](./CONFIGURATION.md).

### Organization ID

Set `OpenAI-Organization` header when `openai.organization_id` is configured. Organization ID affects rate-limit pooling and billing. The adapter **SHOULD** read it from [Secrets Management](./SECRETS_MANAGEMENT.md).

## Rate Limit Handling

OpenAI enforces rate limits at multiple tiers. The adapter **MUST** implement token-bucket backoff per model-family.

| Tier | Limit pattern | Adapter response |
|------|--------------|------------------|
| Free | 3 RPM, 200K TPM | Retry-After; mark `rate_limited` |
| Tier 1 | 500 RPM, 1M TPM | Exponential backoff × 3, then fallback |
| Tier 2 | 5K RPM, 10M TPM | Minimal backoff; warn at 80% capacity |
| Tier 3 | 50K RPM, 50M TPM | Same as Tier 2 |
| Tier 4+ | 500K+ RPM, 200M+ TPM | Custom |

`429 rate_limit_exceeded` responses include a `Retry-After` header. The adapter **MUST** honor it exactly and back off all requests to that model family. `429 insufficient_quota` **MUST** mark `quota_exceeded` and surface a billing alert with no retry.

## Token Counting

The adapter uses [tiktoken](https://github.com/openai/tiktoken) for local token estimation:

| Model family | Encoding |
|-------------|----------|
| `gpt-4`, `gpt-3.5-turbo` | `cl100k_base` |
| `gpt-4o`, `gpt-4o-mini`, `o1`, `o3-mini` | `o200k_base` |
| Text models (`davinci-002`) | `p50k_base` |

The adapter **MUST** estimate `prompt_tokens` before every request. If estimated tokens exceed 90% of the model's context window, emit a `CONTEXT_NEARLY_FULL` warning on the SCE.

## Streaming Implementation

OpenAI streaming uses SSE with `data:` frames and a terminal `data: [DONE]` sentinel:

```
data: {"id":"...","choices":[{"index":0,"delta":{"content":"Hello"},"finish_reason":null}]}
data: {"id":"...","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}
data: [DONE]
```

| Edge case | Handling |
|-----------|----------|
| Empty `choices` array | Skip chunk; do not emit `ChatChunk` |
| `finish_reason` on non-final chunk | Accumulate and deduplicate |
| Truncated SSE frame | Buffer partial lines; re-parse on next `\n` |
| Multi-byte character split across chunks | Buffer `delta.content`; emit only on complete UTF-8 codepoint |
| `content_filter` finish reason | Emit `finish_reason: "content_filter"`; Kernel logs the event |
| Missing `[DONE]` sentinel | Treat connection close as normal finish |

## Error Handling

| HTTP | Code | Adapter action |
|------|------|----------------|
| 400 | `context_length_exceeded` | Surface `CONTEXT_TOO_LONG`; Kernel compresses |
| 400 | `invalid_prompt` | Non-retryable; surface malformed input error |
| 401 | `invalid_api_key` | Mark `auth_error`; do not retry |
| 403 | `permission_error` | Mark `auth_error`; check organization ID |
| 404 | `model_not_found` | Surface unavailable; trigger re-discovery |
| 409 | `conflict` | Retry once |
| 429 | `rate_limit_exceeded` | Honor `Retry-After`; exponential backoff |
| 429 | `insufficient_quota` | Mark `quota_exceeded`; billing alert |
| 500 | `server_error` | Retry × 3 with backoff; mark `degraded` |
| 503 | `engine_overloaded` | Retry with 5 s backoff × 3; mark `degraded` |

## Models

| Model | Context | Max output | Pricing (input / output per MTok) | Capabilities |
|-------|---------|-----------|-----------------------------------|--------------|
| `gpt-4o` | 128K | 16,384 | $2.50 / $10.00 | Text, vision, tools, structured outputs |
| `gpt-4o-mini` | 128K | 16,384 | $0.15 / $0.60 | Text, vision, tools, structured outputs |
| `gpt-4-turbo` | 128K | 4,096 | $10.00 / $30.00 | Text, vision, tools |
| `gpt-4` | 8K / 32K | 4,096 | $30.00 / $60.00 | Text, tools |
| `gpt-3.5-turbo` | 16K | 4,096 | $0.50 / $1.50 | Text, tools |
| `o1` | 200K | 100K | $15.00 / $60.00 | Reasoning, vision, structured outputs |
| `o1-mini` | 128K | 65,536 | $1.10 / $4.40 | Reasoning, text |
| `o3-mini` | 200K | 100K | $1.10 / $4.40 | Reasoning, text, structured outputs |
| `text-embedding-3-large` | N/A | N/A | $0.13 / MTok | Embeddings, 3072 dims |
| `text-embedding-3-small` | N/A | N/A | $0.02 / MTok | Embeddings, 1536 dims |

Pricing reflects current rates. The adapter **MUST** load live pricing from OpenRouter or a pricing cache — see [Cost Management](./COST_MANAGEMENT.md).

## Function Calling vs Tools API

OpenAI deprecated the legacy `functions` parameter in favor of `tools`:

| Feature | Legacy `functions` | Current `tools` |
|---------|-------------------|-----------------|
| Schema format | `{ name, description, parameters }` | `{ type: "function", function: { name, description, parameters } }` |
| Parallel calls | No | Yes |
| `tool_choice` | `"auto"` / `"none"` | `"auto"` / `"none"` / `{ type: "function", function: { name } }` |
| Strict mode | No | `{ strict: true }` with JSON Schema (gpt-4o-2024-08-06+) |

The adapter **MUST** normalize `Tool[]` to the `tools` parameter. It **SHOULD** downgrade to legacy `functions` for models that do not support `tools`.

## Vision API Integration

Vision requests use content blocks within the message:

```typescript
messages: [{
  role: "user",
  content: [
    { type: "text", text: "What's in this image?" },
    { type: "image_url", image_url: { url: "data:image/png;base64,<base64>", detail: "auto" } }
  ]
}]
```

| Detail | Tokens per image (1080p) | Use case |
|--------|-------------------------|----------|
| `"low"` | 85 | Thumbnails, diagrams |
| `"high"` | ~1,105 (768 × 2048 tile) | Photographs, documents |
| `"auto"` | Model decides | General purpose |

The adapter **SHOULD** prefer `"low"` for images under 100 KB. It **MUST** reject images exceeding 20 MB.

## Structured Outputs

Supported on gpt-4o-2024-08-06+ and o1-2024-12-17+:

```typescript
response_format: {
  type: "json_schema",
  json_schema: {
    name: "extracted_data",
    strict: true,
    schema: { type: "object", properties: { name: { type: "string" } }, required: ["name"], additionalProperties: false }
  }
}
```

Models that do not support `response_format` **MUST** fall back to a system prompt instructing JSON output, with a warning emitted on the SCE.

## Security Considerations

- API keys **MUST** be read from [Secrets Management](./SECRETS_MANAGEMENT.md); never from `process.env` in production.
- Azure `api-key` and standard `Bearer` tokens are equivalent in sensitivity.
- All requests **MUST** use TLS (HTTPS). The adapter **MUST** reject plaintext endpoints.

## Related Documents

- [Model Providers](./MODEL_PROVIDERS.md) — parent specification
- [Nine Router](./NINE_ROUTER.md) — fallback routing for rate-limited/quota-exceeded providers
- [Cost Management](./COST_MANAGEMENT.md) — pricing cache, budget enforcement
- [Model Discovery](./MODEL_DISCOVERY.md) — model normalization pipeline
- [Secrets Management](./SECRETS_MANAGEMENT.md) — credential storage
- [Token Budgeting](./TOKEN_BUDGETING.md) — context window enforcement
- [Streaming Responses](./STREAMING_RESPONSES.md) — end-to-end streaming contract
- [Model Routing Policy](./MODEL_ROUTING_POLICY.md) — model selection rules
