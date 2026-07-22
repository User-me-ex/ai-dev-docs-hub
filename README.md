# AI Development Operating System — Documentation

> The complete specification repository for the **AI Development Operating System (AI Dev OS)** — an open-source multi-agent AI platform for software development.

## What is AI Dev OS?

AI Dev OS is an operating system for AI-assisted software development. It orchestrates multiple AI agents — the Kernel, Planner, Nine Router, Researcher, Builder, Critic, Merger, Guardian, and Voice — through a deterministic loop to produce production-quality code. It is **local-first**, **model-agnostic**, and **observability-built-in**.

This repository contains the **complete specification** for the entire platform. Every subsystem, integration, process, and policy is documented with architecture diagrams, interface definitions, failure mode analysis, and cross-references.

## Quick Links

| Area | Documents |
|------|-----------|
| **Product** | [Vision](./docs/PROJECT_VISION.md) · [PRD](./docs/PRD.md) · [TRD](./docs/TRD.md) · [Roadmap](./docs/IMPLEMENTATION_ROADMAP.md) |
| **Architecture** | [System Overview](./docs/SYSTEM_OVERVIEW.md) · [Main AI Kernel](./docs/MAIN_AI_KERNEL.md) · [Architecture Diagrams](./diagrams/ARCHITECTURE.md) |
| **Agent System** | [AI Groups](./docs/AI_GROUPS.md) · [Dynamic Workers](./docs/DYNAMIC_WORKERS.md) · [Agent Lifecycle](./docs/AGENT_LIFECYCLE.md) · [Agent Communication](./docs/AGENT_COMMUNICATION.md) |
| **Routing** | [Nine Router](./docs/NINE_ROUTER.md) · [Model Discovery](./docs/MODEL_DISCOVERY.md) · [Model Providers](./docs/MODEL_PROVIDERS.md) · [Routing Policy](./docs/MODEL_ROUTING_POLICY.md) |
| **Knowledge** | [Knowledge System](./docs/KNOWLEDGE_SYSTEM.md) · [Persistent Memory](./docs/PERSISTENT_MEMORY.md) · [RAG Pipeline](./docs/RAG_PIPELINE.md) |
| **Quality & Safety** | [Architecture Guardian](./docs/ARCHITECTURE_GUARDIAN.md) · [Impact Analysis](./docs/IMPACT_ANALYSIS.md) · [Merge Manager](./docs/MERGE_MANAGER.md) · [Testing Strategy](./docs/TESTING_STRATEGY.md) |
| **Infrastructure** | [Backend](./docs/BACKEND.md) · [Frontend](./docs/FRONTEND.md) · [Database](./docs/DATABASE.md) · [CLI](./docs/CLI.md) · [Docker](./docs/DOCKER.md) · [CI/CD](./docs/CICD.md) |
| **Operations** | [Deployment](./docs/DEPLOYMENT.md) · [Observability](./docs/OBSERVABILITY.md) · [Logging](./docs/LOGGING.md) · [Metrics](./docs/METRICS.md) · [Tracing](./docs/TRACING.md) |
| **Guides** | [Getting Started](./docs/GETTING_STARTED.md) · [Installation](./docs/INSTALLATION.md) · [Local Dev](./docs/LOCAL_DEV.md) · [Troubleshooting](./docs/TROUBLESHOOTING.md) |

## Repository Map

```
/
├── README.md                    ← This file
├── REPORT.md                    ← Audit report
├── AGENTS.md                    ┬ Agent instructions
├── assets/                      ┤ (currently empty)
├── docs/                        ┤ 130+ specification files
│   ├── knowledge-bases/         ┤ 4-tier KB specs
│   └── adrs/                    ┤ Architecture Decision Records (6)
├── prompts/                     ┤ 8 canonical prompt templates
├── diagrams/                    ┤ 16 Mermaid diagram files
└── templates/                   ┤ 8 reusable document templates
```

## Documentation Standards

All documents follow these conventions:

- **RFC 2119 language**: `MUST`, `SHOULD`, `MAY` indicate requirement levels.
- **Standard skeleton**: Overview → Goals → Non-Goals → Architecture (Mermaid) → Interfaces → Data Model → Failure Modes → Security → Observability → Acceptance Criteria → Related Documents.
- **Cross-references**: Every document links to related subsystems.
- **Mermaid diagrams**: Architecture flowcharts and sequence diagrams for key workflows.

## Quick Start

```bash
# Read the system overview
open docs/SYSTEM_OVERVIEW.md

# Explore the Kernel loop
open docs/MAIN_AI_KERNEL.md

# Understand the Nine Router
open docs/NINE_ROUTER.md

# See the implementation plan
open docs/IMPLEMENTATION_ROADMAP.md
```

## Contributing

See [CONTRIBUTING.md](./docs/CONTRIBUTING.md) and [AI Coding Rules](./docs/AI_CODING_RULES.md) for contribution guidelines. For architecture decisions, see [ADRs](./docs/adrs/README.md).

## License

See [LICENSE_NOTES.md](./docs/LICENSE_NOTES.md).
