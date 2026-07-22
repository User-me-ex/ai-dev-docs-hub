# Context Window Management

> How AI Dev OS composes, budgets, compresses, and overflows LLM context windows across models, workers, and runs. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

Context Window Management is the subsystem responsible for building, maintaining, and compressing the token sequence sent to every model call in AI Dev OS. It operates inside the [Dynamic Worker](./DYNAMIC_WORKERS.md) process, between the worker's reasoning loop and the [Model Providers](./MODEL_PROVIDERS.md) layer. Every model invocation passes through four stages: **compose** (build the message array), **count** (measure token usage), **compress** (reduce if over limit), and **send**.

The subsystem has three core responsibilities:
1. Ensure every model call fits within the model's `context_window` limit, reserving room for the model's output tokens.
2. Maximise the relevancy of content within the available window by prioritising recent task context and high-scoring KB excerpts.
3. Handle overflow gracefully through compression, summarisation, or fallback to a larger model — never by silently dropping content without logging.

## Goals

- Deterministic token budgeting: given a model ID and message array, the token count MUST be reproducible.
- Content prioritisation: the most relevant content (task context > recent conversation > KB excerpts > historical turns) stays in the window; low-relevance content is evicted first.
- Graceful degradation: when compression is insufficient, fallback is triggered automatically with a structured event.
- Zero silent truncation: every truncation, compression, or summarisation step MUST emit an event on the [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md).

## Non-Goals

- Model-specific tokenisation outside the counting interface — token counts use the model provider's own tokenizer where available.
- Persistent context across runs — context is scoped to a single worker's lifetime; [Persistent Memory](./PERSISTENT_MEMORY.md) handles durable knowledge.
- Implementation code — this repository is documentation-only (see [AI Coding Rules](./AI_CODING_RULES.md)).

## Context Window Limits per Model

| Model ID | Context window (tokens) | Max output tokens | Recommended max input |
|----------|------------------------|-------------------|----------------------|
| openai/gpt-4o | 128,000 | 16,384 | 110,000 |
| openai/gpt-4o-mini | 128,000 | 16,384 | 110,000 |
| openai/o1 | 200,000 | 100,000 | 180,000 |
| openai/o3-mini | 200,000 | 100,000 | 180,000 |
| anthropic/claude-3-5-sonnet-20241022 | 200,000 | 8,192 | 190,000 |
| anthropic/claude-3-opus-20240229 | 200,000 | 4,096 | 195,000 |
| anthropic/claude-3-haiku-20240307 | 48,000 | 4,096 | 42,000 |
| google/gemini-2.0-flash | 1,048,576 | 8,192 | 1,000,000 |
| google/gemini-2.5-pro | 1,048,576 | 65,536 | 1,000,000 |
| mistral/mistral-large-2411 | 128,000 | 4,096 | 120,000 |
| ollama/llama3.1:8b | 8,192 | 2,048 | 6,000 |
| ollama/llama3.1:70b | 8,192 | 2,048 | 6,000 |
| ollama/mixtral:8x7b | 32,768 | 4,096 | 28,000 |

The "Recommended max input" column reserves room for the model's maximum output tokens plus a 2,000-token safety margin. Context windows are fetched from the [Model Discovery](./MODEL_DISCOVERY.md) cache and may update as providers change their offerings.

## Context Window Composition Strategy

The context window is assembled from four layers, in priority order:

```
Layer 1 — System Prompt (fixed, ~2,000 tokens)
  ├── Agent identity and role definition
  ├── Safety rules (S-01 through S-08, condensed)
  ├── Tool definitions (selected tools from WorkerSpec)
  └── Output formatting instructions

Layer 2 — Task Context (variable, reserved first)
  ├── Current task description and goal
  ├── Relevant memory records (from KB query)
  ├── Code context (surrounding files, function signatures)
  └── Tool results from current task execution

Layer 3 — Conversation History (variable, sliding window)
  ├── assistant messages (model responses)
  ├── user messages (tool results, system feedback)
  └── Rolling window: keep last N turns, summarise older ones

Layer 4 — Knowledge Base Excerpts (variable, scored)
  ├── Retrieved by RAG pipeline for task relevance
  ├── Scored 0.0–1.0 by hybrid search (BM25 + embedding)
  └── Lowest-score excerpts evicted first under pressure
```

Composition algorithm:

```
compose_context(worker_spec, task, kb_results, history):
  1. system = build_system_prompt(worker_spec.role, worker_spec.tools, safety_rules)
  2. task_msgs = build_task_messages(task, kb_results)
  3. history_msgs = prune_history(history, max_history_tokens)
  4. full_context = [system] + task_msgs + history_msgs

  5. total = token_count(full_context)
  6. max_input = model.context_window - model.max_output_tokens - SAFETY_MARGIN

  7. if total > max_input:
       full_context = compress(full_context, target=max_input)

  8. return full_context
```

## Token Counting Methods

| Provider | Method | Implementation |
|----------|--------|----------------|
| OpenAI | `tiktoken` | `encoding = tiktoken.encoding_for_model(model_id); len(encoding.encode(text))` |
| Anthropic | Client SDK | `client.count_tokens(messages)` — uses server-side counting |
| Google | `vertexai` SDK | `model.count_tokens(content)` — server-side |
| Mistral | `mistral_common` | `tokenizer.encode(text, return_length=True)` |
| Ollama (local) | Estimate | `len(text) // 4` fallback; uses provider API count if available |
| Fallback | Heuristic | `len(text) / 3.5` tokens per character (conservative) |

All token counts are cached per text fragment with an LRU cache (max 10,000 entries, TTL 60 s). Server-side counting (Anthropic, Google) is NOT cached.

## Context Compression Strategies

When `total_tokens > max_input`, the compressor applies strategies in order from least to most destructive:

### Strategy 1: Structured Truncation (fastest, lowest quality loss)

Remove structured elements that are least critical:
1. Drop tool call argument values (keep names, drop values) for completed calls.
2. Truncate code context to function signatures only (remove function bodies).
3. Remove KB excerpts below `relevance_score < 0.3`.

Each removal emits a `context.dropped` event with `reason` and `tokens_saved`.

### Strategy 2: Conversation Summarisation (medium speed, moderate loss)

When truncation is insufficient, summarise oldest conversation turns:

```
summarize_oldest_turns(history, target_tokens):
  1. Identify turns older than the last K turns (K = max(5, len(history) * 0.3))
  2. Concatenate oldest turns into a single text block
  3. Call model_providers.summarize(text_block, max_length=1000 tokens)
  4. Replace with <summary> tag and original turn count
  5. Emit context.summarized { original_tokens, summarized_tokens, turns_count }
```

Summarisation preserves tool call names and their success/failure status but drops the full tool call arguments and outputs. The summary is clearly labelled in the context so the model knows it is reading a compressed representation.

### Strategy 3: Critical Drop (last resort, highest loss)

If compression target is still not met:
1. Drop all KB excerpts (emit `context.dropped { reason: "kb_excerpts", count, tokens_saved }`).
2. Drop all but the last 3 conversation turns.
3. As a final measure, truncate the oldest turn unconditionally, keeping the system prompt and latest turn intact.

After any compression strategy, if the context still exceeds `max_input`, the system MUST NOT send the truncated context — it MUST fall back to overflow handling.

## Handling Context Overflow

### CONTEXT_TOO_LONG Error

When the model provider returns a `CONTEXT_TOO_LONG` error (indicates our token count was inaccurate or the provider's limit is stricter than declared):

```
on context_too_long_error:
  1. Log the discrepancy between our count and the provider's count
  2. Increment a backoff counter (max 3 retries)
  3. Apply compression strategy 2 (summarisation) with 20% stricter target
  4. Retry the call
  5. If all 3 retries fail, emit model.context_too_long → fallback
```

### Automatic Compression Schedule

Between every model call, the compressor checks the running context size:

```
if (running_token_count >= max_input * 0.85):
    schedule_compression = true   # compress before next model call
    emit context.warning { token_count, max_input, pct_full: 0.85 }
```

### Fallback to Larger-Context Model

When compression is exhausted and the context still does not fit:

1. The worker emits `context.fallback_required` with the current token count and model ID.
2. The [Nine Router](./NINE_ROUTER.md) is queried for a fallback model with a larger context window (e.g., 128K → 200K, or 200K → 1M for Gemini).
3. If a fallback is available, the worker rebinds to the new model and retries the compose cycle.
4. If no fallback is available, the worker emits `worker.failed` with `reason: "context_overflow"` and delivers partial results.

## Budgeting Tokens Within Context

Tokens are budgeted per layer. The following defaults can be overridden in `GroupSpec.context_budget`:

| Layer | Default budget (tokens) | Notes |
|-------|------------------------|-------|
| System prompt | 2,000 | Fixed; includes role + tools + safety rules |
| Task context | 40% of max_input | Reserved first; never compressed unless overflow |
| Conversation history | 30% of max_input | Sliding window; oldest N turns summarised |
| KB excerpts | 20% of max_input | Pruned by relevance score |
| Safety margin | 10% of max_input | Reserved for model output growth |

## Interfaces

All context operations are exposed through the [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md) and the internal API:

| Function | Signature | Description |
|----------|-----------|-------------|
| `context.compose` | `(spec: WorkerSpec, task: Task, kb: KBDoc[], history: Message[]) => Message[]` | Build the full message array for a model call. |
| `context.compress` | `(messages: Message[], target_tokens: number, strategy?: CompressionStrategy) => Message[]` | Compress context to fit within target_tokens. Returns compressed messages and emits events for every removed or summarised segment. |
| `context.token_count` | `(messages: Message[], model_id: string) => number` | Count tokens for the given message array using the model's tokenizer. |
| `context.summarize` | `(text: string, max_tokens: number) => string` | Summarise a text block using the configured summarisation model. |
| `context.budget_status` | `() => { used: number, max_input: number, pct_full: number, layers: LayerBudget[] }` | Return the current budget breakdown per layer. |

## Acceptance Criteria

- `context.compose` for a model with a 128K window and 100K input MUST produce a message array within 110K tokens (reserving 16K output + 2K margin).
- `context.compress` with target_tokens=50K on a 70K input MUST produce an array ≤ 50K tokens and emit at least one `context.dropped` or `context.summarized` event.
- A CONTEXT_TOO_LONG error from the provider MUST trigger fallback logic within 1 second, with no more than 3 retry attempts.
- Token counts for OpenAI models using `context.token_count` MUST match the server-side count within 1% tolerance.
- After compression, the system prompt and the most recent user message MUST always be present (never summarised or dropped).

## Open Questions

- Whether context compression should be delegated to a specialised "Compressor" worker to avoid blocking the executing worker — tracked in [templates/ADR](../templates/ADR.md).
- Whether to support a persistent context cache (EPOCH in OpenAI, Prompt Caching in Anthropic) and how to expose it through the compose API.

## Related Documents

- [Agent Memory](./AGENT_MEMORY.md) — per-agent, per-run, and persistent memory stores
- [Token Budgeting](./TOKEN_BUDGETING.md) — token budget enforcement per run and per worker
- [RAG Pipeline](./RAG_PIPELINE.md) — knowledge base retrieval that feeds into context composition
- [Model Providers](./MODEL_PROVIDERS.md) — model-specific configuration and tokenizer access
- [Dynamic Workers](./DYNAMIC_WORKERS.md) — worker lifecycle and checkpoint protocol
- [Agent Lifecycle](./AGENT_LIFECYCLE.md) — full state machine for worker execution
- [System Overview](./SYSTEM_OVERVIEW.md)
- [Main AI Kernel](./MAIN_AI_KERNEL.md)
