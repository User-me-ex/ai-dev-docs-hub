# Master Prompt

> The canonical system prompt template consumed by the Main AI Kernel. All derived prompts (planner, router, builder, critic, researcher) inherit from it.

## Purpose

The Master Prompt encodes the invariants every agent in the AI Dev OS must obey: safety, honesty, respect for architectural guardians, and the OS's operating principles.

## Source of Truth

The canonical text lives in [`prompts/MASTER_PROMPT.md`](../prompts/MASTER_PROMPT.md). This document describes its structure and governance.

## Structure

1. **Identity** — who the agent is, which role it plays.
2. **Mission** — the current goal and its provenance.
3. **Operating Principles** — local-first, documentation-first, small reversible steps.
4. **Guardrails** — safety, privacy, tool-use limits.
5. **Context Contracts** — how the Shared Context Engine is read and written.
6. **Escalation Rules** — when to defer to the human or to the Critic.
7. **Output Contract** — expected format for the agent's response.

## Governance

- Prompt changes are reviewed under [Prompt Governance](./PROMPT_GOVERNANCE.md).
- All changes must be evaluated with the [Eval Harness](./EVAL_HARNESS.md) before promotion.
- Version and changelog live alongside the prompt file.

## Related Documents

- [Prompt Governance](./PROMPT_GOVERNANCE.md)
- [Main AI Kernel](./MAIN_AI_KERNEL.md)
- [AI Coding Rules](./AI_CODING_RULES.md)
