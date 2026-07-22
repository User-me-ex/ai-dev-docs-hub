# Glossary

> Comprehensive reference of terms, abbreviations, and concepts used across the AI Development Operating System documentation. Terms are organized alphabetically. Each entry follows the format: **term** (part of speech) — Definition. *Context: usage example or cross-reference.*

## A

**Agent** (n.) — An autonomous AI process that performs a specific role within the Nine Roles architecture. Agents receive tasks via the Kernel, execute using assigned tools and models, and emit results through the SCE. *Context: "Each agent runs in its own sandbox with a bounded context window."*

**Agent Lifecycle** (n.) — The state machine governing an agent from creation through teardown: `idle → assigned → active → paused → completed | failed | timed_out`. The lifecycle is managed by the Kernel and published on the Event Bus. *Context: "The Agent Lifecycle ensures every run is observable and recoverable."*

**AGS** (abbr.) — AI Group System. A subsystem that manages logical groups of agents sharing a common knowledge base, permissions, and execution context. *Context: "The AGS creates a GroupKB for each project team."*

**Architecture Guardian** (n.) — A static-analysis role that enforces architectural rules and invariants across the system. It validates every merge transaction against the rule set defined in [ARCHITECTURE_GUARDIAN.md](./ARCHITECTURE_GUARDIAN.md). *Context: "The Architecture Guardian vetoed the merge because it violated the layering rule."*

**Audit Log** (n.) — An append-only, tamper-evident record of all operations performed by agents and the Kernel. Each entry carries a `correlation_id`, timestamp, actor, action, and payload digest. *Context: "The Audit Log is the source of truth for compliance investigations."*

## B

**Backpressure** (n.) — A flow-control mechanism that prevents downstream systems from being overwhelmed by upstream producers. Implemented via bounded queues and rejection signals. *Context: "When the vector store queue reaches capacity, backpressure causes the embedding pipeline to pause."*

**Blast Radius** (n.) — The set of subsystems affected when a given component fails or is compromised. Used in threat modeling to scope security boundaries. *Context: "OIDC token leakage has a blast radius limited to the issuing workspace."*

**Budget** (n.) — A monetary or token allocation for model inference, enforced by the Cost Meter. Budgets are set per workspace, group, or agent and can trigger alerts or hard stops. *Context: "The monthly inference budget for the development workspace is $200."*

**Builder role** (n.) — One of the Nine Roles. Responsible for implementing code changes, running tests, and producing artifacts. Builders consume Planner specifications and produce deliverables for the Critic. *Context: "The Builder agent generated the implementation from the Planner's spec."*

## C

**Capability Token** (n.) — A signed, short-lived bearer token that grants an agent permission to invoke a specific set of tools or access a specific resource. *Context: "The file-write capability token expires after 60 seconds."*

**Circuit Breaker** (n.) — A resilience pattern that stops requests to a failing provider or subsystem after a threshold of failures, then periodically probes for recovery. *Context: "The OpenAI adapter's circuit breaker tripped after 5 consecutive 429s."*

**Checkpoint** (n.) — A snapshot of an agent's full state (memory, context window, task graph position) at a point in time, enabling pause-resume and rollback. *Context: "The Kernel creates a checkpoint before every tool call."*

**CLI** (abbr.) — Command-Line Interface. The primary user-facing interface for AI Dev OS, used to start sessions, inspect state, and manage configuration. *Context: "Run `aidevos session start --group dev-team` from the CLI."*

**Context Engine** (n.) — See Shared Context Engine (SCE).

**Context Window** (n.) — The maximum number of tokens an agent can process in a single inference request. Managed and partitioned by the Context Window Manager to prevent overflow. *Context: "Claude Sonnet 4 has a 200K token context window."*

**correlation_id** (n.) — A ULID that uniquely identifies a single end-to-end run, propagated across all subsystems, logs, traces, and audit entries. *Context: "Every logged event includes the correlation_id for debugging."*

**Cost Meter** (n.) — A subsystem that tracks token consumption and monetary spend per model, provider, workspace, and agent. Interacts with Budgets to enforce cost limits. *Context: "The Cost Meter logged 5,000 input tokens for the Gemini 2.5 Flash call."*

**Critic role** (n.) — One of the Nine Roles. Reviews Builder output against acceptance criteria, runs the Eval Harness, and either approves or requests revision. *Context: "The Critic flagged a missing edge case in the Builder's implementation."*

## D

**Dynamic Worker** (n.) — A worker process that is spawned on demand for a specific task and terminated after completion. Dynamic Workers are stateless and receive their configuration via WorkerSpec. *Context: "The Nine Router assigned the task to a Dynamic Worker running Mistral Large."*

## E

**Ed25519** (n.) — A public-key signature system used for signing capability tokens, merge transactions, and audit log entries. Ed25519 provides high security with small key sizes and fast verification. *Context: "Every MergeTxn is signed with the agent's Ed25519 key."*

**Embedding** (n.) — A dense vector representation of text, used for semantic search and clustering. AI Dev OS generates embeddings via configurable provider adapters and stores them in the Vector Store. *Context: "The RAG Pipeline converts each document chunk into a 1536-dimensional embedding."*

**Event Bus** (n.) — A publish-subscribe message bus that carries system events (state changes, errors, metrics) between subsystems. The Event Bus is the backbone of observability. *Context: "The Architecture Guardian publishes validation results to the Event Bus."*

**Eval Harness** (n.) — A testing framework for evaluating agent outputs against predefined acceptance criteria. Used by the Critic role and in CI pipelines. *Context: "The Eval Harness runs the test suite and compares actual vs expected outputs."*

## F

**Fallback Chain** (n.) — An ordered list of providers or models that the Nine Router tries sequentially when the primary choice fails or is rate-limited. *Context: "The fallback chain is: Anthropic → OpenAI → Google → Mistral."*

**FTS (Full Text Search)** (n.) — Keyword-based search over knowledge base entries, complementing semantic (embedding) search. Typically backed by SQLite FTS5. *Context: "The knowledge base query uses FTS for exact matches and embeddings for semantic similarity."*

## G

**Global KB** (n.) — The top-level knowledge base accessible to all agents and workspaces. Contains system-wide documentation, policies, and reference data. *Context: "Global KB stores the company coding standards."*

**Group KB** (n.) — A knowledge base scoped to a specific AGS group. Inherits from Global KB but may override or extend. *Context: "The frontend-team Group KB contains component library documentation."*

**GroupRun** (n.) — A coordinated execution of multiple agents within an AGS group, orchestrated by the Kernel. GroupRuns have a shared task graph and synchronized context. *Context: "The weekly security audit runs as a GroupRun across all agents."*

**GroupSpec** (n.) — The declarative configuration for an AGS group: members, roles, knowledge base bindings, permissions, and routing rules. *Context: "The GroupSpec declares three agents with Planner, Builder, and Critic roles."*

**Guardian** (n.) — See Architecture Guardian.

## H

**Heartbeat** (n.) — A periodic signal emitted by a worker or agent to indicate liveness. Missed heartbeats trigger health checks and potential replacement. *Context: "The Kernel terminated the worker after three missed heartbeats."*

## I

**IA (Impact Analysis)** (n.) — Impact Analysis. A subsystem that analyzes proposed changes to determine affected components, services, and documentation. *Context: "Before merging, the Impact Analysis reported 6 files might be affected."*

**Individual KB** (n.) — A knowledge base scoped to a single agent, holding session-local context and ephemeral data. Not persisted across agent restarts unless explicitly written to Group KB. *Context: "The agent stored the intermediate analysis in its Individual KB."*

**Intake phase** (n.) — The first phase of the Kernel loop. Raw user input is parsed, validated, classified, and enriched before being handed to the Planner. *Context: "During the Intake phase, the Kernel extracts intent and entities from the prompt."*

**IPC** (abbr.) — Inter-Process Communication. The mechanism by which agents and subsystems communicate across process boundaries, typically via Unix domain sockets or Windows named pipes. *Context: "Dynamic Workers use IPC over Unix sockets to stream tokens back to the Kernel."*

## K

**Kernel** (n.) — The central orchestrator of AI Dev OS. The Kernel runs the primary control loop: Intake → Plan → Route → Execute → Critique → Merge → Guard → Deliver. *Context: "The Kernel schedules agents, manages state, and enforces lifecycle."*

**KB (Knowledge Base)** (n.) — A hierarchical storage system for documents, embeddings, and structured data. KBs exist at three levels: Global, Group, and Individual. *Context: "Search the KB for relevant documentation before generating code."*

**Knowledge System** (n.) — The overarching subsystem that manages KB creation, search, ingestion, TTL policies, and cross-KB queries. *Context: "The Knowledge System supports both FTS and semantic search."*

## M

**Main KB** (n.) — The primary knowledge base for a workspace, combining Global KB and Group KB into a unified view. Agents query the Main KB by default. *Context: "The Main KB returned 12 relevant documents for the query."*

**MCP** (abbr.) — Model Context Protocol. An open protocol for exposing tools and data sources to AI models. AI Dev OS acts as an MCP host, connecting to MCP servers for tool execution. *Context: "The GitLab MCP server exposes merge request APIs to agents."*

**Merge Manager** (n.) — The subsystem responsible for reviewing, validating, and committing changes produced by agents. Works with the Architecture Guardian and Impact Analysis. *Context: "The Merge Manager rejected the diff because it lacked test coverage."*

**MergeTxn** (n.) — A signed, atomic transaction containing one or more file changes, metadata, and the agent's Ed25519 signature. MergeTxns are the unit of work submitted to the Merge Manager. *Context: "The MergeTxn contained edits to three files and a test file."*

**Model Binding** (n.) — The configuration that maps a role or task type to a specific model and provider, including parameters like temperature, max tokens, and tool selection. *Context: "The Planner role has a model binding to Claude Sonnet 4 with temperature 0.3."*

**Model Discovery** (n.) — The subsystem that periodically queries all configured providers for available models, normalizes their metadata, and publishes the unified model catalog. *Context: "Model Discovery runs every 5 minutes and on demand via `aidevos models refresh`."*

## N

**Nine Roles** (n.) — The nine agent role archetypes: Intake, Planner, Router, Builder, Critic, Merge, Guardian, Deliver, Monitor. Each role has distinct responsibilities, capabilities, and model bindings. *Context: "The Nine Roles architecture separates concerns across the agent pipeline."*

**Nine Router** (n.) — The role assignment and provider routing engine. Given a task and available agents, the Nine Router selects the optimal agent, provider, and model based on capability, cost, and latency. *Context: "The Nine Router assigned the coding task to a Builder agent on Claude Sonnet 4."*

## O

**OGE (Obsidian Graph Engine)** (n.) — A subsystem that builds and queries a graph of Obsidian-style markdown notes with bidirectional links. Supports graph traversal, backlinks, and knowledge visualization. *Context: "The OGE powers the knowledge graph view in the CLI dashboard."*

**OIDC** (abbr.) — OpenID Connect. An identity layer on top of OAuth 2.0 used for authenticating agents and users across workspaces. *Context: "The agent authenticated via OIDC using its workspace-issued identity token."*

## P

**Persistent Memory** (n.) — A durable storage layer for agent state, knowledge base content, and system configuration. Persisted across sessions and backed by SQLite or PostgreSQL. *Context: "The agent's Persistent Memory survived the restart, preserving task progress."*

**Planner role** (n.) — One of the Nine Roles. Receives the parsed intent from Intake and produces a structured plan (task graph, acceptance criteria, dependencies) for execution. *Context: "The Planner decomposed the feature request into 5 subtasks."*

**Planning Engine** (n.) — The subsystem behind the Planner role. Generates task graphs, estimates effort, and identifies dependencies using the Research Engine and Web Intelligence. *Context: "The Planning Engine produced a 12-node DAG for the migration project."*

**Plugin SDK** (n.) — A software development kit for extending AI Dev OS with custom provider adapters, tools, and subsystems. Plugins are loaded at startup and integrated via stable extension points. *Context: "The team built a Jira integration using the Plugin SDK."*

**Post-Mortem** (n.) — A retrospective analysis document generated after a failure or incident. Post-Mortems are stored in the knowledge base and used to update system rules. *Context: "The Kernel generated a Post-Mortem for the timeout incident."*

## R

**RAG Pipeline** (n.) — Retrieval-Augmented Generation pipeline. Ingests documents, chunks them, generates embeddings, stores in the Vector Store, and retrieves relevant context at query time. *Context: "The RAG Pipeline retrieved the relevant API spec before generating the code."*

**RBAC** (abbr.) — Role-Based Access Control. The authorization model governing what agents and users can do, backed by capability tokens and workspace-scoped policies. *Context: "RBAC restricted the agent from accessing production secrets."*

**Research Engine** (n.) — A subsystem that performs deep research on a topic using web search, knowledge base queries, and multi-step reasoning. Produces structured research reports. *Context: "The Research Engine compiled a report on the latest LLM safety benchmarks."*

**Role (Nine Roles)** (n.) — A defined set of responsibilities, capabilities, and model bindings within the Nine Roles architecture. Each role is implemented by one or more agents. *Context: "The Critic role requires a model with strong reasoning capabilities."*

**Router role** (n.) — One of the Nine Roles. Assigns tasks to the appropriate agent based on capability, load, and cost. Works in tandem with the Nine Router subsystem. *Context: "The Router role directed the task to the least-loaded Builder agent."*

## S

**SCE (Shared Context Engine)** (n.) — The central communication and state-sharing hub. All subsystems publish state changes to the SCE and subscribe to relevant topics. The SCE is the backbone of inter-agent coordination. *Context: "The Builder published the MergeTxn to the SCE for the Critic to review."*

**Secrets Management** (n.) — A subsystem for storing, rotating, and accessing credentials (API keys, tokens, certificates) securely. All provider credentials MUST be read from Secrets Management, never from environment variables. *Context: "The OpenAI adapter reads its API key from Secrets Management at startup."*

**Semantic Edge** (n.) — A typed, directed relationship between two knowledge graph nodes, carrying a label and optional properties. Used by the OGE for graph traversal. *Context: "The 'depends_on' semantic edge connects the Planning Engine to the Vector Store."*

**Session Token** (n.) — A signed JWT or capability token that authenticates a CLI session. Carries workspace, role, and expiration claims. *Context: "The CLI session token expires after 8 hours by default."*

**Snapshot (SCE)** (n.) — A point-in-time capture of all active SCE topics and their latest values, used for checkpointing and recovery. *Context: "The Kernel took an SCE snapshot before initiating the multi-agent GroupRun."*

## T

**Task Graph** (n.) — A directed acyclic graph (DAG) representing the plan for a run. Each node is a task with dependencies, inputs, outputs, and an assigned agent. *Context: "The Planner's task graph had 8 nodes with 3 parallel branches."*

**Task Graph (DAG)** (n.) — See Task Graph. The DAG structure enables parallel execution of independent tasks and sequential execution of dependent ones. *Context: "The Kernel executes independent DAG branches concurrently."*

**TTL** (abbr.) — Time-To-Live. The maximum lifespan of a cache entry, session token, checkpoint, or knowledge base document before it is invalidated or deleted. *Context: "Knowledge base entries have a default TTL of 30 days."*

## U

**ULID** (n.) — Universally Unique Lexicographically Sortable Identifier. A 26-character, time-sortable identifier used for correlation IDs, task IDs, and audit log entries. *Context: "The correlation_id `01HXYZ...` is a ULID that encodes the timestamp of the run."*

## V

**Vector Store** (n.) — A database optimized for storing and querying embedding vectors via similarity search. AI Dev OS supports multiple Vector Store backends (SQLite with vector extension, Qdrant). *Context: "The RAG Pipeline queries the Vector Store for the top 5 most similar documents."*

## W

**WAL (Write-Ahead Log)** (n.) — A durability mechanism used by SQLite and other storage engines. All writes go to the WAL first before being checkpointed to the main database. *Context: "The SQLite WAL enabled concurrent reads during the write-heavy merge operation."*

**Warm Pool** (n.) — A pre-allocated set of worker processes kept ready to accept tasks immediately, reducing cold-start latency. *Context: "The Kernel maintains a Warm Pool of 3 Dynamic Workers."*

**Web Intelligence** (n.) — A subsystem that performs live web searches, fetches pages, and extracts structured information. Used by the Research Engine and during the Intake phase. *Context: "Web Intelligence fetched the latest API documentation from the vendor website."*

**Worker** (n.) — A process that executes tasks on behalf of an agent. Workers may be long-running (daemon) or dynamic (spawned per task). *Context: "The Worker loaded the model and began streaming the response."*

**WorkerSpec** (n.) — The declarative configuration for a worker: model binding, context size, tool list, timeout, and resource limits. *Context: "The WorkerSpec for the Builder agent includes the file-edit tool and a 5-minute timeout."*

**Workspace** (n.) — An isolated environment with its own agents, knowledge bases, secrets, and configuration. Workspaces map to projects or teams. *Context: "Each workspace has its own vector store namespace and cost budget."*
