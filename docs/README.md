# AI Dev OS — Documentation

This repository is **documentation-only**. It contains the specifications, prompts, diagrams, and templates that define the AI Development Operating System (AI Dev OS). No application code is shipped here.

## Layout

- `docs/` — subsystem specifications, one Markdown file per topic.
- `docs/knowledge-bases/` — the four-tier knowledge base spec (Global, Main, Group, Individual).
- `prompts/` — canonical system, kernel, planner, router, critic, research, and agent prompts.
- `diagrams/` — Mermaid diagrams for architecture, data flow, deployment, and the Nine Router.
- `templates/` — reusable Markdown templates (ADR, RFC, PRD, TRD, agent spec, prompt spec).
- `assets/` — static assets referenced from docs.

## Start Here

1. [Project Vision](./PROJECT_VISION.md)
2. [Product Overview](./PRODUCT_OVERVIEW.md)
3. [PRD](./PRD.md) and [TRD](./TRD.md)
4. [System Overview](./SYSTEM_OVERVIEW.md)
5. [Main AI Kernel](./MAIN_AI_KERNEL.md)
6. [Nine Router](./NINE_ROUTER.md) and [Model Discovery](./MODEL_DISCOVERY.md)
7. [Implementation Roadmap](./IMPLEMENTATION_ROADMAP.md)

## Conventions

- Every subsystem doc follows the same section skeleton (Overview → Goals → Non-Goals → Requirements → Architecture → Interfaces → Data Model → Failure Modes → Security → Open Questions → Related Documents).
- Diagrams live in `diagrams/` as Mermaid; docs link to them rather than inlining.
- Prompts live in `prompts/`; docs describing prompt behavior link to the prompt file.
- This repository must never contain application code. See [AI Coding Rules](./AI_CODING_RULES.md).
