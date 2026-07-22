# Nine Router

> Model discovery, grouping, and role assignment across every connected provider.

## Overview

The Nine Router is the model-routing subsystem of the AI Dev OS. It discovers every model available to the workspace, normalizes their metadata, groups them by provider, and lets the user (or the kernel) assign models to the nine canonical roles.

## The Nine Roles

1. **Kernel** — top-level orchestrator.
2. **Planner** — decomposes goals into task graphs.
3. **Router** — selects models per subtask (recursive use of Nine Router).
4. **Researcher** — retrieval and synthesis.
5. **Builder** — code generation and edits.
6. **Critic** — reviews and rejects unsafe or low-quality output.
7. **Merger** — reconciles concurrent edits (see [Merge Manager](./MERGE_MANAGER.md)).
8. **Guardian** — enforces architectural invariants (see [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md)).
9. **Voice** — speech I/O routing (see [Voice System](./VOICE_SYSTEM.md)).

## Model Discovery Requirements

- Discovery MUST query each provider's `/models` endpoint (or the closest documented equivalent).
- Results MUST be grouped by provider: OpenAI, Anthropic, Mistral, Google, Ollama, local (llama.cpp / MLX), and any user-registered provider.
- Each discovered model MUST be normalized into `{ id, provider, display_name, context_window, modalities, capabilities, pricing?, deprecated? }`.
- Discovery results MUST be cached with a TTL and a manual **Refresh** action.
- Discovery MUST degrade gracefully when a provider is unreachable — partial results are surfaced with a per-provider error indicator.

## UI Requirements

The router UI MUST support:

- **Search** across model IDs and display names.
- **Filter** by provider, capability (tools, vision, audio), context window, modality, deprecation status.
- **Refresh** to re-query `/models` on demand, per provider or globally.
- **Assign** any discovered model to any of the nine roles, with per-project overrides.
- **Fallback chains** — ordered lists of models per role, evaluated by the [Model Routing Policy](./MODEL_ROUTING_POLICY.md).

## Data Model

See [Model Discovery](./MODEL_DISCOVERY.md) for the normalized schema, and [Model Providers](./MODEL_PROVIDERS.md) for provider-specific quirks.

## Flow

See [diagrams/NINE_ROUTER_FLOW.md](../diagrams/NINE_ROUTER_FLOW.md).

## Related Documents

- [Model Discovery](./MODEL_DISCOVERY.md)
- [Model Providers](./MODEL_PROVIDERS.md)
- [Model Routing Policy](./MODEL_ROUTING_POLICY.md)
- [Main AI Kernel](./MAIN_AI_KERNEL.md)
