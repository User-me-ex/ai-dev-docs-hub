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

## Conventions

- Every subsystem doc follows the same section skeleton: **Overview → Goals → Non-Goals → Requirements → Architecture → Interfaces → Data Model → Failure Modes → Security → Observability → Acceptance Criteria → Open Questions → Related Documents**.
- Requirements language follows RFC 2119 (**MUST / SHOULD / MAY**).
- Diagrams live in [`../diagrams/`](../diagrams) as Mermaid; docs link out rather than inlining large graphs.
- Prompts live in [`../prompts/`](../prompts); docs describing prompt behavior link to the prompt file.
- Templates for new docs live in [`../templates/`](../templates).
- This repository must never contain application code. See [AI Coding Rules](./AI_CODING_RULES.md).
