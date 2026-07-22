# Google Integration

> Provider adapter implementation guide for the Google AI (Gemini) API. Extends [Model Providers](./MODEL_PROVIDERS.md) with Google-specific configuration, safety settings, grounding, and streaming handling.

## Overview

The Google adapter connects the Nine Router to Gemini models via `generativelanguage.googleapis.com`. Google uses a resource-oriented REST API with query-parameter authentication and NDJSON streaming. The adapter normalizes these to the standard `ChatRequest` / `ChatChunk` interface.

## Endpoint Configuration

```
base_url: https://generativelanguage.googleapis.com/v1beta
auth:     ?key=${GOOGLE_API_KEY}
```

API keys **MUST** be provisioned from [Google AI Studio](https://aistudio.google.com/app/apikey) and read from [Secrets Management](./SECRETS_MANAGEMENT.md).

```yaml
providers:
  google:
    api_key: ${GOOGLE_API_KEY}
    base_url: https://generativelanguage.googleapis.com/v1beta
```

## Models

| Model ID | Context | Max output | Pricing (input / output per MTok) | Capabilities |
|----------|---------|-----------|-----------------------------------|--------------|
| `gemini-2.5-pro-exp-03-25` | 1,048,576 | 65,536 | $1.25–$10.00 / $5.00–$40.00 | Text, vision, audio, code exec, grounding, function calling |
| `gemini-2.5-flash-preview-04-17` | 1,048,576 | 65,536 | $0.15–$0.60 / $0.60–$2.40 | Text, vision, audio, grounding, function calling |
| `gemini-2.0-flash-lite-preview-02-05` | 1,048,576 | 8,192 | $0.075–$0.30 / $0.30–$1.20 | Text, vision, function calling |

Pricing tiers: requests up to 200K tokens use the lower price; exceeding 200K uses the higher price.

## API Format

### Request

```
POST /v1beta/models/{model}:streamGenerateContent?key={key}
{
  "contents": [{ "role": "user", "parts": [{ "text": "Hello" }] }],
  "systemInstruction": { "parts": [{ "text": "You are a helpful assistant." }] },
  "generationConfig": { "temperature": 0.7, "maxOutputTokens": 4096 },
  "safetySettings": [
    { "category": "HARM_CATEGORY_HARASSMENT", "threshold": "BLOCK_MEDIUM_AND_ABOVE" },
    { "category": "HARM_CATEGORY_HATE_SPEECH", "threshold": "BLOCK_MEDIUM_AND_ABOVE" },
    { "category": "HARM_CATEGORY_SEXUALLY_EXPLICIT", "threshold": "BLOCK_MEDIUM_AND_ABOVE" },
    { "category": "HARM_CATEGORY_DANGEROUS_CONTENT", "threshold": "BLOCK_MEDIUM_AND_ABOVE" }
  ]
}
```

### Streaming

Response is NDJSON (one JSON object per line). The adapter **MUST**:

1. Parse each line as independent JSON.
2. Extract `candidates[0].content.parts[0].text` as token delta.
3. Track `candidates[0].finishReason` for completion.
4. Emit `ChatChunk { type: "token", delta }` per non-empty text part.
5. On finish, emit `ChatChunk { type: "finish", finish_reason, usage }` from `usageMetadata`.

## System Instruction

Google uses a top-level `systemInstruction` field. The adapter **MUST** extract the first `system`-role message and place it there. Omit if no system message exists.

## Grounding with Google Search

```json
{
  "tools": [{ "googleSearchGrounding": {} }]
}
```

Response includes `groundingMetadata` with source URLs and confidence scores. The adapter **MUST** propagate this as structured metadata on the final `ChatChunk`. Grounding is available on Gemini 2.5 and 2.0 Flash models at higher pricing tiers.

## Safety Settings

| Threshold | Behavior |
|-----------|----------|
| `BLOCK_ONLY_HIGH` | Block only on HIGH probability |
| `BLOCK_MEDIUM_AND_ABOVE` | Block on MEDIUM or HIGH (default) |
| `BLOCK_LOW_AND_ABOVE` | Block on LOW+, MEDIUM+, or HIGH |
| `BLOCK_NONE` | Always show |

Categories: `HARM_CATEGORY_HARASSMENT`, `HARM_CATEGORY_HATE_SPEECH`, `HARM_CATEGORY_SEXUALLY_EXPLICIT`, `HARM_CATEGORY_DANGEROUS_CONTENT`, `HARM_CATEGORY_CIVIC_INTEGRITY`.

When blocked, `finishReason` is `SAFETY`. The adapter surfaces `ChatChunk { type: "error", finish_reason: "content_filter" }`.

## Rate Limits

| Tier | Requests/min | Tokens/min |
|------|-------------|------------|
| Free | 60 | 1,000,000 |
| Pay-as-you-go | 2,000 | 4,000,000 |

The adapter **MUST** track `x-ratelimit-remaining` and `x-ratelimit-reset` headers. Google does not use `Retry-After` — wait for the duration in `x-ratelimit-reset`.

## Error Codes

| HTTP | Status | Adapter action |
|------|--------|----------------|
| 400 | `INVALID_ARGUMENT` | Surface validation error |
| 401 | `UNAUTHENTICATED` | Mark `auth_error` |
| 403 | `PERMISSION_DENIED` | Mark `auth_error`; check key scope |
| 404 | `NOT_FOUND` | Trigger re-discovery |
| 429 | `RESOURCE_EXHAUSTED` | Backoff; mark `rate_limited` |
| 429 | `QUOTA_EXCEEDED` | Mark `quota_exceeded`; billing alert |
| 500+ | Internal | Retry × 3 exponential backoff; mark `degraded` |

## Related Documents

- [Model Providers](./MODEL_PROVIDERS.md)
- [Nine Router](./NINE_ROUTER.md)
- [Cost Management](./COST_MANAGEMENT.md)
- [Model Discovery](./MODEL_DISCOVERY.md)
- [Secrets Management](./SECRETS_MANAGEMENT.md)
- [Streaming Responses](./STREAMING_RESPONSES.md)
- [Tool Calling](./TOOL_CALLING.md)
