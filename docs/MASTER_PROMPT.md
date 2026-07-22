# Master Prompt

> Normative specification for the Master Prompt — the base system prompt injected into every agent's context window. This document defines the prompt's structure, variables, governance, and versioning contract. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

The Master Prompt is the foundational prompt from which all agent prompts derive. It encodes the invariants, safety rules, and operating principles that every agent in AI Dev OS must obey. It is injected as the first system message in every agent's context window, before any role-specific prompt, task description, or knowledge base excerpt.

The Master Prompt template lives at `prompts/MASTER_PROMPT.md`. This spec document defines what the prompt must contain, how it is versioned, and how it is injected.

## Goals

- Every agent, regardless of role, receives a consistent set of operating principles and constraints.
- The Master Prompt enforces safety, honesty, and architectural boundaries before any task-specific instructions are added.
- Prompt versioning follows the governance process in [Prompt Governance](./PROMPT_GOVERNANCE.md) so that changes are auditable and reversible.
- The prompt is parameterised via template variables so that run-specific context (run_id, role, budget, tools) is injected at context assembly time, not baked into the prompt file.

## Non-Goals

- Role-specific instructions — those belong in the Kernel, Planner, Router, Builder, Critic, Researcher, and Agent prompts.
- Task-specific instructions — those are generated dynamically by the Planning Engine.
- Knowledge base content — that is injected by the Context Window Manager after the prompt.
- Implementation code — this repo is documentation-only ([AI Coding Rules](./AI_CODING_RULES.md)).

## Prompt Template Structure

The Master Prompt template is a Markdown file with Jinja2-style `{{ variable }}` placeholders. It has these sections in order:

```markdown
# System Prompt

You are an AI agent running inside AI Dev OS — an intelligent development operating system.

## Your Context
- Run ID:           {{ run_id }}
- Task ID:          {{ task_id }}
- Role:             {{ role }}
- Group:            {{ group_id }}
- Correlation ID:   {{ correlation_id }}
- Budget (tokens):  {{ budget.tokens_max }} total
- Budget (time):    {{ budget.wall_ms_max / 1000 }} seconds
- Tools available:  {{ tool_list | join(", ") }}

## Operating Instructions
...

## Safety Rules
...

## Output Contract
...

## Quality Rules
...

## Termination Conditions
...
```

### Section Content Rules

| Section | Required | Content |
|---------|----------|---------|
| Identity | yes | Agent role, group, run context. Must include all template variables. |
| Operating Instructions | yes | Documentation-only repo rules, correlation_id propagation, SCE event emission, checkpointing, graceful degradation. |
| Safety Rules | yes | No code execution outside sandbox, no secret leakage, no direct provider calls, no bypassing Guardian, no hidden state. |
| Output Contract | yes | Expected output format for the agent's response: structured JSON with `{ summary, artifacts, events, verdict? }`. |
| Quality Rules | yes | RFC 2119 language, cross-references, minimum acceptance criteria, self-review before delivery. |
| Termination Conditions | yes | Budget exhaustion, Guardian veto, task completion, manual cancel. How to signal each. |

## Template Variables

| Variable | Type | Source | Required | Description |
|----------|------|--------|----------|-------------|
| `run_id` | string (ULID) | Kernel | yes | Unique run identifier |
| `task_id` | string (ULID) | Planning Engine | yes | Unique task identifier within run |
| `role` | string | Nine Router binding | yes | Agent's role (kernel, planner, router, researcher, builder, critic, merger, guardian, voice) |
| `group_id` | string | AI Group System | yes | AI Group the agent belongs to |
| `correlation_id` | string (UUID v7) | Kernel | yes | Propagated across all agents in the run |
| `budget.tokens_max` | integer | Kernel (Cost Management) | yes | Maximum tokens for this task |
| `budget.wall_ms_max` | integer | Kernel (Cost Management) | yes | Maximum wall-clock time in ms |
| `budget.usd_max` | float | Kernel (Cost Management) | no | Maximum cost in USD (optional) |
| `tool_list` | string[] | AI Group System | yes | Tools available to this agent |
| `kb_scope` | string | Knowledge System | no | Knowledge base scope hint |
| `checkpoint_interval_ms` | integer | Agent Lifecycle | no | How often to checkpoint |

## Injection Order

When the Kernel assembles an agent's context window, it applies prompts in this order:

```
1. Master Prompt (always first — system message)
2. Role Prompt (e.g. Kernel Prompt, Planner Prompt — second system message)
3. Task Description (from TaskGraph node — user message)
4. Knowledge Base Excerpts (from RAG Pipeline — interspersed as needed)
5. Conversation History (previous turns, tool call results)
6. Current Turn (agent's response)
```

The Master Prompt is always at the start. No other content is placed before it.

## Versioning

The Master Prompt follows semantic versioning as defined in [Prompt Governance](./PROMPT_GOVERNANCE.md):

| Level | Example | When to bump |
|-------|---------|--------------|
| MAJOR | `1.0.0` → `2.0.0` | Breaking change to output contract or safety rules |
| MINOR | `0.1.0` → `0.2.0` | New section added, new variable, non-breaking change |
| PATCH | `0.1.0` → `0.1.1` | Typo, formatting, example rewording |

The version is stored in the YAML front matter of `prompts/MASTER_PROMPT.md`:

```yaml
---
prompt_id: master
version: 0.1.0
description: "Master system prompt v0.1 — initial release"
last_reviewed: "2026-07-22"
---
```

## Template Rendering

Prompt variables are rendered by the Kernel at context-assembly time:

```
1. Kernel reads prompts/MASTER_PROMPT.md from disk
2. Parses YAML front matter (metadata, version)
3. Resolves {{ variable }} placeholders from the RunSpec and TaskSpec
4. Handles missing variables: required variables → error; optional variables → empty string
5. Handles filters: e.g. {{ tool_list | join(", ") }} → "read_file, write_file, search_code"
6. Returns rendered string as the first system message
```

## Configuration

The Master Prompt file path is configurable:

```toml
[AIDEVOS_PROMPT]
master_prompt_path = "prompts/MASTER_PROMPT.md"                # default
role_prompt_dir = "prompts/"                                   # default
```

## Related Prompts

| Prompt | File | Relationship |
|--------|------|--------------|
| System Prompt | `prompts/SYSTEM_PROMPT.md` | Subset of Master Prompt focused on CLI/tool mode |
| Kernel Prompt | `prompts/KERNEL_PROMPT.md` | Extends Master Prompt with Kernel-specific orchestration instructions |
| Planner Prompt | `prompts/PLANNING_PROMPT.md` | Extends with task decomposition and TaskGraph generation |
| Router Prompt | `prompts/ROUTER_PROMPT.md` | Extends with model discovery and role assignment |
| Critic Prompt | `prompts/CRITIC_PROMPT.md` | Extends with verdict generation and quality assessment |
| Researcher Prompt | `prompts/RESEARCH_PROMPT.md` | Extends with web search, crawling, and synthesis |
| Agent Prompt | `prompts/AGENT_PROMPT.md` | Generic worker role — extends with tool-use and checkpointing |

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| Template variable missing at render | `{{ undefined_var }}` left unrendered | Kernel logs WARN; leaves placeholder visible (degraded but safe) |
| Prompt file corrupt | YAML front matter parse error | Kernel falls back to last-known-good cached version; logs CRITICAL |
| Variable type mismatch | `budget.tokens_max` is a string | Coerce to int; log WARN |
| Unicode/injection in variable | Suspicious characters | Kernel sanitises: strips control characters, truncates at 1024 chars |
| Prompt governance disabled | governance registry not found | Kernel uses prompt file as-is with no versioning enforcement |

## Security Considerations

- The Master Prompt is the first defence against prompt injection. Safety rules in the prompt instruct agents to reject instructions that conflict with the Master Prompt.
- Template variables are rendered by the Kernel, not by the LLM. An agent cannot read or modify the template.
- The prompt file is read-only from the agent's perspective. Agents cannot modify prompts.
- The version checksum in the governance registry prevents tampering with prompt files outside the change workflow.

## Acceptance Criteria

- `kernel.submit({goal: "say hello"})` produces an agent context window whose first message is the rendered Master Prompt with all required variables populated.
- Setting `AIDEVOS_PROMPT.master_prompt_path` to a non-existent file causes the Kernel to log a CRITICAL error and use the last-known-good cached version.
- A rendered prompt contains `Run ID: run_01J...` with the correct ULID format.
- The `tool_list | join(", ")` filter produces a comma-separated list from the array `["read_file", "write_file"]` → `"read_file, write_file"`.
- Removing a required section (e.g. "Safety Rules") from the prompt file causes the CI lint gate to fail.

## Related Documents

- [Prompt Governance](./PROMPT_GOVERNANCE.md) — versioning, review, rollout process
- [Eval Harness](./EVAL_HARNESS.md) — prompt evaluation and regression testing
- [Context Window Management](./CONTEXT_WINDOW_MANAGEMENT.md) — how multiple prompt sources are assembled
- [AI Safety](./AI_SAFETY.md) — safety principles encoded in the prompt
- [System Overview](./SYSTEM_OVERVIEW.md)
- [Main AI Kernel](./MAIN_AI_KERNEL.md) — prompt injection point
