# AI Dev OS — Documentation

This repository is **documentation-only**. It contains the specifications, prompts, diagrams, and templates that define the AI Development Operating System (AI Dev OS). No application code is shipped here — see [AI Coding Rules](./AI_CODING_RULES.md).

## Layout

- [`docs/`](.) — subsystem specifications, one Markdown file per topic (125+ files).
- [`docs/knowledge-bases/`](./knowledge-bases) — the four-tier knowledge base spec (Global, Main, Group, Individual).
- [`../prompts/`](../prompts) — canonical system, kernel, planner, router, critic, research, and agent prompts.
- [`../diagrams/`](../diagrams) — Mermaid diagrams for architecture, data flow, deployment, lifecycle, merge, routing, and research.
- [`../templates/`](../templates) — reusable Markdown templates (ADR, RFC, PRD, TRD, agent spec, prompt spec, subsystem spec).
- [`../assets/`](../assets) — static assets referenced from docs.

## Start Here

1. [Project Vision](./PROJECT_VISION.md)
2. [Product Overview](./PRODUCT_OVERVIEW.md)
3. [PRD](./PRD.md) and [TRD](./TRD.md)
4. [System Overview](./SYSTEM_OVERVIEW.md)
5. [Main AI Kernel](./MAIN_AI_KERNEL.md) — the OS kernel and its loop
6. [Nine Router](./NINE_ROUTER.md) and [Model Discovery](./MODEL_DISCOVERY.md) — `/models` catalog and role assignment
7. [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md) — the single source of truth
8. [Implementation Roadmap](./IMPLEMENTATION_ROADMAP.md)

## Reading Paths

- **Product / PM** → Project Vision → Product Overview → PRD → Roadmap
- **Architect** → System Overview → Main AI Kernel → Shared Context Engine → Nine Router → Architecture Guardian → Merge Manager
- **Agent author** → Master Prompt → AI Groups → Dynamic Workers → Agent Lifecycle → Tool Calling → MCP
- **Ops** → Localhost Architecture → Deployment → Observability → Reliability → Disaster Recovery
- **Integrator** → CLI → API Spec → MCP → Plugin SDK → Model Providers

## Documentation Map

### Orchestration
[Main AI Kernel](./MAIN_AI_KERNEL.md) · [Task Scheduler](./TASK_SCHEDULER.md) · [Planning Engine](./PLANNING_ENGINE.md) · [Task Graph](./TASK_GRAPH.md) · [Multi-Agent Orchestration](./MULTI_AGENT_ORCHESTRATION.md) · [Merge Manager](./MERGE_MANAGER.md) · [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) · [Impact Analysis](./IMPACT_ANALYSIS.md) · [ADRs](./adrs/README.md)

### Agents
[AI Groups](./AI_GROUPS.md) · [AI Group System](./AI_GROUP_SYSTEM.md) · [Dynamic Workers](./DYNAMIC_WORKERS.md) · [Worker Scheduler](./WORKER_SCHEDULER.md) · [Agent Lifecycle](./AGENT_LIFECYCLE.md) · [Agent Memory](./AGENT_MEMORY.md) · [Agent Communication](./AGENT_COMMUNICATION.md) · [Consensus](./CONSENSUS.md) · [Conflict Resolution](./CONFLICT_RESOLUTION.md) · [Self Reflection](./SELF_REFLECTION.md)

### Models
[Nine Router](./NINE_ROUTER.md) · [Model Discovery](./MODEL_DISCOVERY.md) · [Model Providers](./MODEL_PROVIDERS.md) · [Model Routing Policy](./MODEL_ROUTING_POLICY.md) · [OpenAI](./OPENAI_INTEGRATION.md) · [Anthropic](./ANTHROPIC_INTEGRATION.md) · [Google](./GOOGLE_INTEGRATION.md) · [Mistral](./MISTRAL_INTEGRATION.md) · [Ollama](./OLLAMA_INTEGRATION.md) · [Local Models](./LOCAL_MODELS.md)

### Knowledge
[Knowledge System](./KNOWLEDGE_SYSTEM.md) · [Persistent Memory](./PERSISTENT_MEMORY.md) · [Memory Synchronization](./MEMORY_SYNCHRONIZATION.md) · [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md) · [Vector Store](./VECTOR_STORE.md) · [Embeddings](./EMBEDDINGS.md) · [RAG Pipeline](./RAG_PIPELINE.md) · [Citation Engine](./CITATION_ENGINE.md) · [Source Ranking](./SOURCE_RANKING.md) · [Obsidian Graph](./OBSIDIAN_GRAPH_ENGINE.md) · [Research Engine](./RESEARCH_ENGINE.md) · [Research Cache](./RESEARCH_CACHE.md) · [Internet Search](./INTERNET_SEARCH.md) · [Web Intelligence](./WEB_INTELLIGENCE.md) · [GitHub Analysis](./GITHUB_ANALYSIS.md)
Knowledge bases: [Global](./knowledge-bases/GLOBAL_KB.md) · [Main](./knowledge-bases/MAIN_KB.md) · [Group](./knowledge-bases/GROUP_KB.md) · [Individual](./knowledge-bases/INDIVIDUAL_KB.md)

### Runtime & Platform
[Localhost Architecture](./LOCALHOST_ARCHITECTURE.md) · [Backend](./BACKEND.md) · [Frontend](./FRONTEND.md) · [Database](./DATABASE.md) · [API Spec](./API_SPEC.md) · [CLI](./CLI.md) · [MCP](./MCP.md) · [IPC](./IPC.md) · [Event Bus](./EVENT_BUS.md) · [Queueing](./QUEUEING.md) · [Job Scheduler](./JOB_SCHEDULER.md) · [Plugin SDK](./PLUGIN_SDK.md) · [Tool Calling](./TOOL_CALLING.md) · [Voice System](./VOICE_SYSTEM.md) · [UI/UX](./UI_UX.md) · [Webhooks](./WEBHOOKS.md) · [Integrations](./INTEGRATIONS.md) · [CI/CD](./CICD.md) · [Docker](./DOCKER.md)

### Reliability & Security
[Security](./SECURITY.md) · [Security Model](./SECURITY_MODEL.md) · [AuthZ / RBAC](./AUTHZ_RBAC.md) · [Auth System](./AUTH_SYSTEM.md) · [Secrets Management](./SECRETS_MANAGEMENT.md) · [Encryption](./ENCRYPTION.md) · [Privacy](./PRIVACY.md) · [Compliance](./COMPLIANCE.md) · [AI Safety](./AI_SAFETY.md) · [Audit Log](./AUDIT_LOG.md) · [Data Retention](./DATA_RETENTION.md) · [Backup Strategy](./BACKUP_STRATEGY.md) · [Disaster Recovery](./DISASTER_RECOVERY.md) · [Reliability](./RELIABILITY.md) · [Rate Limiting](./RATE_LIMITING.md) · [Cost Management](./COST_MANAGEMENT.md)

### Observability & Quality
[Observability](./OBSERVABILITY.md) · [Logging](./LOGGING.md) · [Metrics](./METRICS.md) · [Tracing](./TRACING.md) · [Telemetry](./TELEMETRY.md) · [Reasoning Traces](./REASONING_TRACES.md) · [Benchmarks](./BENCHMARKS.md) · [Eval Harness](./EVAL_HARNESS.md) · [Testing Strategy](./TESTING_STRATEGY.md) · [QA Plan](./QA_PLAN.md) · [Performance](./PERFORMANCE.md) · [Scalability](./SCALABILITY.md) · [Caching](./CACHING_STRATEGY.md) · [BENCHMARKS](./BENCHMARKS.md)

### Governance & Process
[Master Prompt](./MASTER_PROMPT.md) · [Prompt Governance](./PROMPT_GOVERNANCE.md) · [AI Coding Rules](./AI_CODING_RULES.md) · [Class Registry](./CLASS_REGISTRY.md) · [Function Registry](./FUNCTION_REGISTRY.md) · [Symbol Registry](./SYMBOL_REGISTRY.md) · [Variable Registry](./VARIABLE_REGISTRY.md) · [Feature Flags](./FEATURE_FLAGS.md) · [Configuration](./CONFIGURATION.md) · [Environment Variables](./ENVIRONMENT_VARIABLES.md) · [Versioning](./VERSIONING.md) · [Release Process](./RELEASE_PROCESS.md) · [Migration Guide](./MIGRATION_GUIDE.md) · [Upgrade Notes](./UPGRADE_NOTES.md) · [Changelog](./CHANGELOG.md) · [Roadmap](./ROADMAP.md) · [Implementation Roadmap](./IMPLEMENTATION_ROADMAP.md)

### Reference
[Glossary](./GLOSSARY.md) · [FAQ](./FAQ.md) · [Support](./SUPPORT.md) · [Troubleshooting](./TROUBLESHOOTING.md) · [Installation](./INSTALLATION.md) · [Getting Started](./GETTING_STARTED.md) · [Local Dev](./LOCAL_DEV.md) · [Deployment](./DEPLOYMENT.md) · [Folder Structures](./FOLDER_STRUCTURES.md) · [Mermaid Diagrams](./MERMAID_DIAGRAMS.md) · [Third-Party APIs](./THIRD_PARTY_APIS.md) · [License Notes](./LICENSE_NOTES.md) · [Code of Conduct](./CODE_OF_CONDUCT.md) · [Contributing](./CONTRIBUTING.md)

## Document Typology

| Type | Description | Section Skeleton | Example |
|------|-------------|-----------------|---------|
| **Subsystem Specification** | Defines a single subsystem (goals, architecture, interfaces, data model) | Overview → Goals → Non-Goals → Requirements → Architecture → Interfaces → Data Model → Failure Modes → Security → Observability → Acceptance Criteria → Related Documents | [Main AI Kernel](./MAIN_AI_KERNEL.md) |
| **Integration Guide** | Describes how to connect a specific external provider | Endpoint Config → Auth → Capabilities → Error Handling → Streaming → Security | [OpenAI Integration](./OPENAI_INTEGRATION.md) |
| **Process Document** | Defines a workflow, policy, or procedure | Overview → Stages → Roles → Escalation → Failure Modes → Acceptance Criteria | [Conflict Resolution](./CONFLICT_RESOLUTION.md) |
| **Reference** | Alphabetical or categorical reference of terms | A–Z entries | [Glossary](./GLOSSARY.md) |
| **Plan** | Test plan, QA plan, roadmap | Overview → Strategy → Matrix → Gates → Metrics | [QA Plan](./QA_PLAN.md) |
| **Guide** | How-to for developers or operators | Prerequisites → Setup → Workflow → Troubleshooting | [Local Dev](./LOCAL_DEV.md) |

## Cross-Reference Guide

Use this table to navigate between related documents by concern:

| Concern | Primary Document | Related Docs | Used By |
|---------|-----------------|--------------|---------|
| Task execution | [Main AI Kernel](./MAIN_AI_KERNEL.md) | [Planning Engine](./PLANNING_ENGINE.md), [Task Graph](./TASK_GRAPH.md) | Kernel, Router |
| Agent coordination | [Multi-Agent Orchestration](./MULTI_AGENT_ORCHESTRATION.md) | [Agent Communication](./AGENT_COMMUNICATION.md), [Consensus](./CONSENSUS.md), [Conflict Resolution](./CONFLICT_RESOLUTION.md) | Planner, Workers |
| Knowledge management | [Knowledge System](./KNOWLEDGE_SYSTEM.md) | [RAG Pipeline](./RAG_PIPELINE.md), [Vector Store](./VECTOR_STORE.md), [Embeddings](./EMBEDDINGS.md) | Research Engine, Agents |
| Model routing | [Nine Router](./NINE_ROUTER.md) | [Model Discovery](./MODEL_DISCOVERY.md), [Model Routing Policy](./MODEL_ROUTING_POLICY.md) | Kernel, Router |
| Quality assurance | [QA Plan](./QA_PLAN.md) | [Eval Harness](./EVAL_HARNESS.md), [Testing Strategy](./TESTING_STRATEGY.md) | All subsystems |
| Security | [Security Model](./SECURITY_MODEL.md) | [AuthZ/RBAC](./AUTHZ_RBAC.md), [Secrets Management](./SECRETS_MANAGEMENT.md), [Encryption](./ENCRYPTION.md) | All subsystems |
| Observability | [Observability](./OBSERVABILITY.md) | [Logging](./LOGGING.md), [Metrics](./METRICS.md), [Tracing](./TRACING.md), [Telemetry](./TELEMETRY.md) | Operations |
| Deployment | [Deployment](./DEPLOYMENT.md) | [Docker](./DOCKER.md), [CI/CD](./CICD.md), [Disaster Recovery](./DISASTER_RECOVERY.md) | Operations |

## How to Use This Documentation Set

### By Role

| Role | Start With | Then Read | Reference |
|------|-----------|-----------|-----------|
| **Product Manager** | [Project Vision](./PROJECT_VISION.md) → [Product Overview](./PRODUCT_OVERVIEW.md) | [PRD](./PRD.md) → [Roadmap](./ROADMAP.md) | [Glossary](./GLOSSARY.md) |
| **System Architect** | [System Overview](./SYSTEM_OVERVIEW.md) → [Main AI Kernel](./MAIN_AI_KERNEL.md) | [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) → [Merge Manager](./MERGE_MANAGER.md) | All subsystem specs |
| **Agent Developer** | [Master Prompt](./MASTER_PROMPT.md) → [AI Groups](./AI_GROUPS.md) | [Dynamic Workers](./DYNAMIC_WORKERS.md) → [Agent Lifecycle](./AGENT_LIFECYCLE.md) | [Tool Calling](./TOOL_CALLING.md), [MCP](./MCP.md) |
| **DevOps Engineer** | [Localhost Architecture](./LOCALHOST_ARCHITECTURE.md) → [Deployment](./DEPLOYMENT.md) | [Observability](./OBSERVABILITY.md) → [Disaster Recovery](./DISASTER_RECOVERY.md) | [CLI](./CLI.md), [Docker](./DOCKER.md) |
| **Integration Engineer** | [CLI](./CLI.md) → [API Spec](./API_SPEC.md) | [Plugin SDK](./PLUGIN_SDK.md) → [Model Providers](./MODEL_PROVIDERS.md) | [Webhooks](./WEBHOOKS.md), [IPC](./IPC.md) |
| **Contributor** | [Contributing](./CONTRIBUTING.md) → [Local Dev](./LOCAL_DEV.md) | [AI Coding Rules](./AI_CODING_RULES.md) → [QA Plan](./QA_PLAN.md) | [Code of Conduct](./CODE_OF_CONDUCT.md) |

### By Reading Depth

- **Quick orientation (10 min)**: README → Project Vision → Product Overview → Glossary
- **Architecture deep-dive (1 hour)**: System Overview → Main AI Kernel → Nine Router → Shared Context Engine → Architecture Guardian
- **Full understanding (4+ hours)**: All subsystem specs, integration guides, and process documents in the order listed above

## Quick Search Index

| Keyword | Relevant Documents |
|---------|-------------------|
| Agent, worker, lifecycle | [Agent Lifecycle](./AGENT_LIFECYCLE.md), [Dynamic Workers](./DYNAMIC_WORKERS.md), [Worker Scheduler](./WORKER_SCHEDULER.md) |
| Auth, security, encryption | [Auth System](./AUTH_SYSTEM.md), [AuthZ/RBAC](./AUTHZ_RBAC.md), [Encryption](./ENCRYPTION.md), [Security Model](./SECURITY_MODEL.md) |
| Cache, caching | [Caching Strategy](./CACHING_STRATEGY.md), [Research Cache](./RESEARCH_CACHE.md) |
| CI/CD, build, release | [CI/CD](./CICD.md), [Release Process](./RELEASE_PROCESS.md), [Deployment](./DEPLOYMENT.md) |
| CLI, commands, flags | [CLI](./CLI.md) |
| Consensus, conflict | [Consensus](./CONSENSUS.md), [Conflict Resolution](./CONFLICT_RESOLUTION.md) |
| Context, memory, recall | [Context Window Management](./CONTEXT_WINDOW_MANAGEMENT.md), [Agent Memory](./AGENT_MEMORY.md), [Persistent Memory](./PERSISTENT_MEMORY.md) |
| Cost, budget, pricing | [Cost Management](./COST_MANAGEMENT.md) |
| Database, storage | [Database](./DATABASE.md), [Object Store](./OBJECT_STORE.md), [Vector Store](./VECTOR_STORE.md) |
| Docker, container | [Docker](./DOCKER.md) |
| Embeddings, vectors | [Embeddings](./EMBEDDINGS.md), [Vector Store](./VECTOR_STORE.md) |
| Error handling | [Error Handling](./ERROR_HANDLING.md) |
| Eval, testing, benchmarks | [Eval Harness](./EVAL_HARNESS.md), [QA Plan](./QA_PLAN.md), [Benchmarks](./BENCHMARKS.md), [Testing Strategy](./TESTING_STRATEGY.md) |
| Event bus, IPC | [Event Bus](./EVENT_BUS.md), [IPC](./IPC.md) |
| Feature flags | [Feature Flags](./FEATURE_FLAGS.md) |
| Guardian, rules, governance | [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md), [AI Coding Rules](./AI_CODING_RULES.md), [Prompt Governance](./PROMPT_GOVERNANCE.md) |
| Integrations, providers | [Integrations](./INTEGRATIONS.md), [Model Providers](./MODEL_PROVIDERS.md) |
| Kernel, main loop | [Main AI Kernel](./MAIN_AI_KERNEL.md) |
| Knowledge base, RAG | [Knowledge System](./KNOWLEDGE_SYSTEM.md), [RAG Pipeline](./RAG_PIPELINE.md) |
| Logging, observability | [Logging](./LOGGING.md), [Observability](./OBSERVABILITY.md), [Telemetry](./TELEMETRY.md) |
| MCP, tool calling | [MCP](./MCP.md), [Tool Calling](./TOOL_CALLING.md) |
| Memory, persistence | [Agent Memory](./AGENT_MEMORY.md), [Persistent Memory](./PERSISTENT_MEMORY.md) |
| Merge, conflict resolution | [Merge Manager](./MERGE_MANAGER.md), [Conflict Resolution](./CONFLICT_RESOLUTION.md) |
| Metrics, monitoring | [Metrics](./METRICS.md) |
| Migrations, versioning | [Migration Guide](./MIGRATION_GUIDE.md), [Versioning](./VERSIONING.md) |
| Model routing, discovery | [Nine Router](./NINE_ROUTER.md), [Model Discovery](./MODEL_DISCOVERY.md), [Model Routing Policy](./MODEL_ROUTING_POLICY.md) |
| Multi-agent, groups | [Multi-Agent Orchestration](./MULTI_AGENT_ORCHESTRATION.md), [AI Group System](./AI_GROUP_SYSTEM.md) |
| Observability, tracing | [Observability](./OBSERVABILITY.md), [Tracing](./TRACING.md) |
| Plugin SDK, extensions | [Plugin SDK](./PLUGIN_SDK.md) |
| Privacy, compliance | [Privacy](./PRIVACY.md), [Compliance](./COMPLIANCE.md) |
| Queue, scheduler | [Queueing](./QUEUEING.md), [Job Scheduler](./JOB_SCHEDULER.md) |
| Research, source ranking | [Research Engine](./RESEARCH_ENGINE.md), [Source Ranking](./SOURCE_RANKING.md), [Citation Engine](./CITATION_ENGINE.md) |
| Secrets, credentials | [Secrets Management](./SECRETS_MANAGEMENT.md) |
| SCE, shared context | [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md) |
| Security, audit | [Security Model](./SECURITY_MODEL.md), [Audit Log](./AUDIT_LOG.md) |
| UI, frontend, voice | [Frontend](./FRONTEND.md), [Voice System](./VOICE_SYSTEM.md) |
| Webhooks, web intelligence | [Webhooks](./WEBHOOKS.md), [Web Intelligence](./WEB_INTELLIGENCE.md) |

### Navigation Tips

- Every document links to related documents at the bottom in a "Related Documents" section.
- Use the [Glossary](./GLOSSARY.md) as a quick lookup for unfamiliar terms.
- Diagrams are stored in `../diagrams/` as standalone Mermaid files; subsystem docs may inline small diagrams.
- Templates for new documents (ADR, RFC, PRD, subsystem spec) are in `../templates/`.

## Reading Guide

This index table helps you quickly find the right document based on what you need to do:

| If you want to... | Start here | Then read |
|-------------------|------------|-----------|
| Understand the product vision | [Project Vision](./PROJECT_VISION.md) | [Product Overview](./PRODUCT_OVERVIEW.md), [PRD](./PRD.md) |
| Set up a local instance | [Installation](./INSTALLATION.md) | [Getting Started](./GETTING_STARTED.md), [Local Dev](./LOCAL_DEV.md) |
| Configure providers | [Model Providers](./MODEL_PROVIDERS.md) | [Nine Router](./NINE_ROUTER.md), [Local Models](./LOCAL_MODELS.md) |
| Write agent prompts | [Master Prompt](./MASTER_PROMPT.md) | [Prompt Governance](./PROMPT_GOVERNANCE.md), [AI Groups](./AI_GROUPS.md) |
| Debug a failed run | [Troubleshooting](./TROUBLESHOOTING.md) | [Logging](./LOGGING.md), [Observability](./OBSERVABILITY.md) |
| Add a new subsystem spec | [templates/ADR](../templates/ADR.md) | [AI Coding Rules](./AI_CODING_RULES.md) |
| Upgrade from an older version | [Upgrade Notes](./UPGRADE_NOTES.md) | [Migration Guide](./MIGRATION_GUIDE.md), [Changelog](./CHANGELOG.md) |
| Understand the glossary | [Glossary](./GLOSSARY.md) | [FAQ](./FAQ.md) |
| Contribute to the project | [Contributing](./CONTRIBUTING.md) | [Code of Conduct](./CODE_OF_CONDUCT.md), [Local Dev](./LOCAL_DEV.md) |

## Related Documents

### By Category

**Core Architecture**
- [System Overview](./SYSTEM_OVERVIEW.md) — high-level architecture diagram and component descriptions
- [Main AI Kernel](./MAIN_AI_KERNEL.md) — the OS kernel and its primary control loop
- [Nine Router](./NINE_ROUTER.md) — role assignment and provider routing engine
- [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md) — inter-component communication and state hub
- [Project Vision](./PROJECT_VISION.md) — long-term goals and guiding principles

**Agent System**
- [AI Groups](./AI_GROUPS.md) — agent group configuration
- [AI Group System](./AI_GROUP_SYSTEM.md) — group management subsystem
- [Dynamic Workers](./DYNAMIC_WORKERS.md) — worker process lifecycle
- [Agent Lifecycle](./AGENT_LIFECYCLE.md) — agent state machine
- [Agent Communication](./AGENT_COMMUNICATION.md) — inter-agent messaging protocol
- [Multi-Agent Orchestration](./MULTI_AGENT_ORCHESTRATION.md) — coordination patterns for multiple agents
- [Consensus](./CONSENSUS.md) — multi-agent agreement protocol
- [Conflict Resolution](./CONFLICT_RESOLUTION.md) — resolving disagreements between agents
- [Self Reflection](./SELF_REFLECTION.md) — agent self-evaluation and refinement

**Quality and Testing**
- [QA Plan](./QA_PLAN.md) — test levels, release gates, quality metrics
- [Testing Strategy](./TESTING_STRATEGY.md) — detailed test approach
- [Eval Harness](./EVAL_HARNESS.md) — automated evaluation framework
- [Benchmarks](./BENCHMARKS.md) — performance benchmark methodology

**Operations**
- [Deployment](./DEPLOYMENT.md) — deployment topologies
- [Self-Hosting](./SELF_HOSTING.md) — running AI Dev OS on your own hardware
- [Offline Mode](./OFFLINE_MODE.md) — fully offline operation specification
- [Disaster Recovery](./DISASTER_RECOVERY.md) — backup and restore procedures
- [Docker](./DOCKER.md) — container build and deployment

**Security and Compliance**
- [Security Model](./SECURITY_MODEL.md) — overall security architecture
- [Auth System](./AUTH_SYSTEM.md) — authentication mechanisms
- [AuthZ/RBAC](./AUTHZ_RBAC.md) — authorization and role-based access control
- [Secrets Management](./SECRETS_MANAGEMENT.md) — credential storage and rotation
- [Privacy](./PRIVACY.md) — privacy policy and data handling
- [Compliance](./COMPLIANCE.md) — regulatory compliance framework

## Conventions

- Every subsystem doc follows the same section skeleton: **Overview → Goals → Non-Goals → Requirements → Architecture → Interfaces → Data Model → Failure Modes → Security → Observability → Acceptance Criteria → Open Questions → Related Documents**.
- Requirements language follows RFC 2119 (**MUST / SHOULD / MAY**).
- Diagrams live in [`../diagrams/`](../diagrams) as Mermaid; docs link out rather than inlining large graphs.
- Prompts live in [`../prompts/`](../prompts); docs describing prompt behavior link to the prompt file.
- Templates for new docs live in [`../templates/`](../templates).
- This repository must never contain application code. See [AI Coding Rules](./AI_CODING_RULES.md).
