# Nine Router

## Overview

The Nine Router is the model-routing subsystem responsible for discovering, grouping, and assigning models across all connected providers.

## Model Discovery Requirements

- Model discovery MUST query each provider's `/models` endpoint.
- Results MUST be grouped by provider (OpenAI, Anthropic, Mistral, Google, Ollama, local, etc.).
- The UI MUST support:
  - **Search** across model IDs and display names
  - **Filter** by provider, capability, context window, modality
  - **Refresh** to re-query `/models` on demand
  - **Model assignment** to roles (kernel, planner, worker, critic, router, etc.)

## Related Documents

- `MODEL_PROVIDERS.md`
- `MODEL_ROUTING_POLICY.md`
- `MAIN_AI_KERNEL.md`
- `diagrams/NINE_ROUTER_FLOW.md`

_(Stub — full specification to be authored.)_
