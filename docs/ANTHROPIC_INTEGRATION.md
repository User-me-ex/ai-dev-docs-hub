# Anthropic Integration

> Provider adapter implementation guide for the Anthropic API. This document extends [Model Providers](./MODEL_PROVIDERS.md) with Anthropic-specific message formatting, tool use, extended thinking, streaming event handling, and capabilities mapping. Implementations MUST satisfy every MUST clause below.

## Overview

The Anthropic adapter connects the Nine Router to Anthropic's Messages API. Unlike OpenAI's flat message format, Anthropic uses structured content blocks and a distinct streaming event protocol. The adapter normalizes these to the standard `ChatRequest` / `ChatChunk` interfaces while preserving Anthropic-specific features.

## Endpoint Configuration

```
base_url: https://api.anthropic.com/v1
auth:     x-api-key: ${ANTHROPIC_API_KEY}
          anthropic-version: 2023-06-01
```

The `anthropic-version` header **MUST** be set on every request. The adapter **SHOULD** allow overriding the version to access beta features (e.g., `betas: extended-thinking-2025-01-15`).

## Message Format

Anthropic uses a content-block structure within each message:

```typescript
{
  model: "claude-sonnet-4-20250514",
  max_tokens: 4096,
  system: [{ type: "text", text: "System prompt here." }],
  messages: [
    { role: "user", content: [{ type: "text", text: "Hello" }] },
    { role: "assistant", content: [{ type: "text", text: "Hi!" }] }
  ],
  stream: true
}
```

| Content block type | Description |
|--------------------|-------------|
| `text` | Plain text |
| `tool_use` | Tool invocation (id, name, input) |
| `tool_result` | Tool output (tool_use_id, content, is_error) |
| `image` | Base64 image (media_type, data) |
| `document` | PDF document (media_type, data) |

The adapter **MUST** normalize flat `messages: [{ role, content: string }]` into Anthropic's content-block format and flatten incoming content blocks back to `ChatChunk` events.

## Tool Use

Anthropic represents tool calls as `tool_use` content blocks within the assistant response:

```typescript
tools: [{
  name: "get_weather",
  description: "Get the current weather",
  input_schema: { type: "object", properties: { location: { type: "string" } }, required: ["location"] }
}]
```

| Streaming event | ChatChunk mapping |
|----------------|-------------------|
| `content_block_start` with `type: "tool_use"` | `{ type: "tool_call", id, name, arguments: "" }` |
| `content_block_delta` with `type: "input_json_delta"` | Accumulate `partial_json` into `arguments` |
| `content_block_stop` for tool_use block | Emit completed `tool_call` with full arguments JSON |

The adapter **MUST** accumulate `input_json_delta` partial JSON tokens and emit only on content block completion. Tool results are sent as `user` messages with `tool_result` content blocks.

## Extended Thinking (Beta)

Enable via the `betas` header and `thinking` parameter:

```typescript
headers: { "anthropic-beta": "extended-thinking-2025-01-15" }
body: { thinking: { type: "enabled", budget_tokens: 16000 } }
```

Two new content block types appear in the stream:

| Type | Adapter handling |
|------|------------------|
| `thinking` | Emit `ChatChunk { type: "thinking", delta }` |
| `redacted_thinking` | Emit `ChatChunk { type: "thinking", delta: "[REDACTED]" }`; log redaction |

The `budget_tokens` is subtracted from `max_tokens`. The adapter **MUST** ensure `max_tokens > budget_tokens` or reject the request.

## Rate Limit Handling

| Tier | Requests/min | Tokens/min |
|------|-------------|------------|
| Free | 5 | 40K |
| Build | 50 | 200K |
| Pro | 1,000 | 800K |
| Max | 5,000 | 4M |
| Enterprise | 10,000+ | 10M+ |

The adapter **MUST** parse rate limit headers (`anthropic-ratelimit-requests-limit`, `anthropic-ratelimit-requests-remaining`, `anthropic-ratelimit-tokens-limit`, `anthropic-ratelimit-tokens-remaining`, `retry-after-ms`).

On `429`, back off for exactly `retry-after-ms` milliseconds and publish `rate_limited` state to the SCE. Unlike OpenAI, Anthropic uses `retry-after-ms` (integer ms), not `Retry-After` (HTTP-date).

## Token Counting

Anthropic does not expose a public tokenizer. The adapter **SHOULD** prefer the vendor package `@anthropic-ai/token-counter` for pre-request estimation and post-hoc server-reported usage from `message_delta.usage` for accounting.

```typescript
import { countTokens } from '@anthropic-ai/token-counter'
countTokens(messages, model)
```

## Streaming Event Types

Anthropic emits typed SSE events with `event:` and `data:` on separate lines:

| Event | When | Adapter action |
|-------|------|---------------|
| `message_start` | Start | Initialize accumulator with `id`, `model`, `usage` |
| `content_block_start` | New block | Begin accumulating by type (text, tool_use, thinking) |
| `content_block_delta` | Delta within block | Append `delta.text` or `delta.partial_json` |
| `content_block_stop` | Block end | Finalize; emit `ChatChunk` for block |
| `message_delta` | Metadata | Update `stop_reason`, `usage.output_tokens` |
| `message_stop` | End | Emit `ChatChunk { type: "finish", usage, finish_reason }` |
| `ping` | Keepalive | Ignore; reset idle timer |

The adapter **MUST** maintain a content block state machine and handle interleaved text/tool_use blocks within a single response.

## Models

| Model ID | Context | Max output | Pricing (input / output per MTok) | Capabilities |
|----------|---------|-----------|-----------------------------------|--------------|
| `claude-sonnet-4-20250514` | 200K | 8,192 | $3.00 / $15.00 | Text, vision, tools, extended thinking |
| `claude-3-5-sonnet-20241022` | 200K | 8,192 | $3.00 / $15.00 | Text, vision, tools, PDF |
| `claude-3-opus-20240229` | 200K | 4,096 | $15.00 / $75.00 | Text, vision, tools, PDF |
| `claude-3-haiku-20240307` | 200K | 4,096 | $0.25 / $1.25 | Text, vision, tools, PDF |
| `claude-3-5-haiku-20241022` | 200K | 8,192 | $0.80 / $4.00 | Text, vision, tools, PDF |

All Claude models support 200K context. The adapter **MUST** respect per-model `max_tokens` limits.

## Vision (Image Input)

```typescript
content: [{
  type: "image",
  source: { type: "base64", media_type: "image/jpeg", data: "<base64>" }
}]
```

| Constraint | Value |
|-----------|-------|
| Max image size | 100 MB |
| Max dimensions | 8,000 × 8,000 px |
| Supported formats | JPEG, PNG, WebP, GIF |
| Token cost | ~1,600 tokens per 1024 × 1024 px tile |

The adapter **MUST** reject images exceeding size or dimension limits.

## PDF Input Support

Claude 3.5+ models accept PDFs as `document` content blocks:

```typescript
content: [{
  type: "document",
  source: { type: "base64", media_type: "application/pdf", data: "<base64>" }
}]
```

| Constraint | Value |
|-----------|-------|
| Max file size | 32 MB |
| Max pages | 100 |
| Max tokens per page | ~1,500 |

The adapter **MUST** warn when PDF page count exceeds 50 pages.

## Error Codes

| HTTP | Code | Adapter action |
|------|------|----------------|
| 400 | `invalid_request_error` | Surface validation error |
| 400 | `context_length_exceeded` | Surface `CONTEXT_TOO_LONG`; Kernel compresses |
| 401 | `authentication_error` | Mark `auth_error`; do not retry |
| 403 | `permission_error` | Mark `auth_error`; check API key scope |
| 404 | `not_found_error` | Model unavailable; trigger re-discovery |
| 429 | `rate_limit_error` | Parse `retry-after-ms`; backoff |
| 429 | `quota_exceeded_error` | Mark `quota_exceeded`; billing alert |
| 500 | `api_error` | Retry × 3 with exponential backoff; mark `degraded` |
| 529 | `overload_error` | Retry with 5 s backoff × 3; mark `degraded` |

529 is unique to Anthropic — treat it as a 503.

## Security Considerations

- API keys **MUST** be read from [Secrets Management](./SECRETS_MANAGEMENT.md); never from `process.env`.
- All requests **MUST** use TLS (HTTPS).
- PDF and image content carries the same sensitivity as message text.

## Related Documents

- [Model Providers](./MODEL_PROVIDERS.md) — parent specification
- [Nine Router](./NINE_ROUTER.md) — fallback routing for rate-limited providers
- [Cost Management](./COST_MANAGEMENT.md) — pricing cache, budget enforcement
- [Model Discovery](./MODEL_DISCOVERY.md) — model normalization pipeline
- [Secrets Management](./SECRETS_MANAGEMENT.md) — credential storage
- [Token Budgeting](./TOKEN_BUDGETING.md) — context window enforcement
- [Streaming Responses](./STREAMING_RESPONSES.md) — end-to-end streaming contract
- [Tool Calling](./TOOL_CALLING.md) — cross-provider tool use abstraction
