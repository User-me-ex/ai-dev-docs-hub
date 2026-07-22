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

## J

**Jitter** (n.) — A random delay added to retry or scheduling intervals to prevent thundering-herd patterns. Implemented as ±25% uniform jitter on exponential backoff. *Context: "Adding jitter to the sync interval reduced the load spike from 12 concurrent reconnections."*

## K

**Kernel** (n.) — The central orchestrator of AI Dev OS. The Kernel runs the primary control loop: Intake → Plan → Route → Execute → Critique → Merge → Guard → Deliver. *Context: "The Kernel schedules agents, manages state, and enforces lifecycle."*

**KB (Knowledge Base)** (n.) — A hierarchical storage system for documents, embeddings, and structured data. KBs exist at three levels: Global, Group, and Individual. *Context: "Search the KB for relevant documentation before generating code."*

**Knowledge System** (n.) — The overarching subsystem that manages KB creation, search, ingestion, TTL policies, and cross-KB queries. *Context: "The Knowledge System supports both FTS and semantic search."*

## L

**Leaky Bucket** (n.) — A rate-limiting algorithm where requests enter a bucket and leak out at a constant rate. If the bucket overflows, excess requests are rejected. Used for smoothing bursty traffic. *Context: "The provider adapter uses a leaky bucket to cap outgoing requests to 100 per second."*

**Load Shedding** (n.) — The practice of dropping non-critical work when the system approaches capacity limits. Prevents cascading failures by preserving capacity for essential operations. *Context: "When CPU exceeds 90%, the Kernel begins load shedding: non-critical background jobs are cancelled."*

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

**9Router** (n.) — Alternative shorthand for the Nine Router subsystem. Used in configuration files and CLI commands where brevity is preferred. *Context: "Set the 9Router endpoint in config.toml under [nine_router]."*

**Network Partition** (n.) — A failure condition where two or more nodes in a distributed system lose network connectivity between each other while remaining operational individually. Network partitions require split-brain prevention and reconciliation strategies. *Context: "During a network partition, each node continues operating on local data and queues sync events for replay on reconnection."*

## O

**OGE (Obsidian Graph Engine)** (n.) — A subsystem that builds and queries a graph of Obsidian-style markdown notes with bidirectional links. Supports graph traversal, backlinks, and knowledge visualization. *Context: "The OGE powers the knowledge graph view in the CLI dashboard."*

**OIDC** (abbr.) — OpenID Connect. An identity layer on top of OAuth 2.0 used for authenticating agents and users across workspaces. *Context: "The agent authenticated via OIDC using its workspace-issued identity token."*

## P

**Percentile** (n.) — A statistical measure indicating the value below which a given percentage of observations fall. Used for latency targets (p50, p95, p99) in performance budgets. *Context: "The p99 latency for model inference must stay under 5 seconds."*

**Persistent Memory** (n.) — A durable storage layer for agent state, knowledge base content, and system configuration. Persisted across sessions and backed by SQLite or PostgreSQL. *Context: "The agent's Persistent Memory survived the restart, preserving task progress."*

**Planner role** (n.) — One of the Nine Roles. Receives the parsed intent from Intake and produces a structured plan (task graph, acceptance criteria, dependencies) for execution. *Context: "The Planner decomposed the feature request into 5 subtasks."*

**Planning Engine** (n.) — The subsystem behind the Planner role. Generates task graphs, estimates effort, and identifies dependencies using the Research Engine and Web Intelligence. *Context: "The Planning Engine produced a 12-node DAG for the migration project."*

**Plugin SDK** (n.) — A software development kit for extending AI Dev OS with custom provider adapters, tools, and subsystems. Plugins are loaded at startup and integrated via stable extension points. *Context: "The team built a Jira integration using the Plugin SDK."*

**Post-Mortem** (n.) — A retrospective analysis document generated after a failure or incident. Post-Mortems are stored in the knowledge base and used to update system rules. *Context: "The Kernel generated a Post-Mortem for the timeout incident."*

**Provider Alias** (n.) — A shorthand name configured for a model provider endpoint, used in routing rules and fallback chains. Aliases abstract away connection details (URL, API key reference). *Context: "The 'primary' provider alias points to the local Ollama instance; 'fallback' points to the Anthropic proxy."*

## R

**RAG Pipeline** (n.) — Retrieval-Augmented Generation pipeline. Ingests documents, chunks them, generates embeddings, stores in the Vector Store, and retrieves relevant context at query time. *Context: "The RAG Pipeline retrieved the relevant API spec before generating the code."*

**RBAC** (abbr.) — Role-Based Access Control. The authorization model governing what agents and users can do, backed by capability tokens and workspace-scoped policies. *Context: "RBAC restricted the agent from accessing production secrets."*

**Research Engine** (n.) — A subsystem that performs deep research on a topic using web search, knowledge base queries, and multi-step reasoning. Produces structured research reports. *Context: "The Research Engine compiled a report on the latest LLM safety benchmarks."*

**Role (Nine Roles)** (n.) — A defined set of responsibilities, capabilities, and model bindings within the Nine Roles architecture. Each role is implemented by one or more agents. *Context: "The Critic role requires a model with strong reasoning capabilities."*

**Role Assignment** (n.) — The process by which the Nine Router maps a task to an agent, model, and provider based on capability requirements, cost constraints, latency targets, and current load. Role assignment considers the fallback chain if the primary binding is unavailable. *Context: "Role assignment selected a Builder agent with Claude Sonnet 4 after the primary GPT-4o binding was rate-limited."*

**Router role** (n.) — One of the Nine Roles. Assigns tasks to the appropriate agent based on capability, load, and cost. Works in tandem with the Nine Router subsystem. *Context: "The Router role directed the task to the least-loaded Builder agent."*

## S

**SCE (Shared Context Engine)** (n.) — The central communication and state-sharing hub. All subsystems publish state changes to the SCE and subscribe to relevant topics. The SCE is the backbone of inter-agent coordination. *Context: "The Builder published the MergeTxn to the SCE for the Critic to review."*

**Secrets Management** (n.) — A subsystem for storing, rotating, and accessing credentials (API keys, tokens, certificates) securely. All provider credentials MUST be read from Secrets Management, never from environment variables. *Context: "The OpenAI adapter reads its API key from Secrets Management at startup."*

**Semantic Edge** (n.) — A typed, directed relationship between two knowledge graph nodes, carrying a label and optional properties. Used by the OGE for graph traversal. *Context: "The 'depends_on' semantic edge connects the Planning Engine to the Vector Store."*

**Semantic Versioning** (n.) — A versioning scheme (MAJOR.MINOR.PATCH) where MAJOR indicates breaking changes, MINOR indicates backward-compatible additions, and PATCH indicates backward-compatible bug fixes. All AI Dev OS components follow SemVer. *Context: "The model binding API was bumped to v2.0.0 with the breaking provider interface change."*

**Session Token** (n.) — A signed JWT or capability token that authenticates a CLI session. Carries workspace, role, and expiration claims. *Context: "The CLI session token expires after 8 hours by default."*

**SLI (Service Level Indicator)** (n.) — A quantifiable measure of service performance, such as request latency, error rate, or throughput. SLIs are the raw measurements used to compute SLO compliance. *Context: "The p99 latency SLI for model inference is collected from the Nine Router metrics endpoint."*

**SLO (Service Level Objective)** (n.) — A target value or range for an SLI over a specified time window. SLOs define the acceptable performance threshold for a subsystem. *Context: "The model inference SLO requires p99 latency under 5 seconds over a 30-day rolling window."*

**Snapshot (SCE)** (n.) — A point-in-time capture of all active SCE topics and their latest values, used for checkpointing and recovery. *Context: "The Kernel took an SCE snapshot before initiating the multi-agent GroupRun."*

**SSE Streaming** (n.) — Server-Sent Events streaming. Used to stream model inference tokens from the Nine Router to the Kernel and from the Kernel to the CLI or Dashboard in real time. *Context: "The model response is delivered token-by-token over SSE, enabling the dashboard to show a live typing effect."*

## T

**Tail Latency** (n.) — The latency experienced by the slowest fraction of requests, typically measured at the p99 or p99.9 percentile. Tail latency is critical for user-facing systems where a single slow request can degrade the perceived experience. *Context: "P99 tail latency for the routing decision must stay under 200 ms."*

**Task Graph** (n.) — A directed acyclic graph (DAG) representing the plan for a run. Each node is a task with dependencies, inputs, outputs, and an assigned agent. *Context: "The Planner's task graph had 8 nodes with 3 parallel branches."*

**Task Graph (DAG)** (n.) — See Task Graph. The DAG structure enables parallel execution of independent tasks and sequential execution of dependent ones. *Context: "The Kernel executes independent DAG branches concurrently."*

**Thundering Herd** (n.) — A failure pattern where many clients simultaneously attempt the same operation after a cache expiry or service recovery, overwhelming the backend. Mitigated with jitter, pre-warming, and rate limiting. *Context: "The database was hit by a thundering herd when all agent caches expired simultaneously after the TTL reset."*

**Tier Routing** (n.) — A routing strategy that classifies models into tiers (e.g., "fast", "balanced", "quality") and assigns tasks to tiers based on latency and accuracy requirements. *Context: "Simple formatting tasks are routed to the 'fast' tier (local Llama); complex code generation goes to the 'quality' tier (Claude Sonnet)."*

**Toil** (n.) — Manual, repetitive, automatable operational work. The AI Dev OS aims to eliminate toil through automated self-healing, auto-scaling, and self-remediation. *Context: "Manual config file editing after every upgrade is toil — the migration command should handle it."*

**Token Bucket** (n.) — A rate-limiting algorithm where tokens are added at a fixed rate and each request consumes one token. Requests are blocked or queued when the bucket is empty. *Context: "The API adapter uses a token bucket with a capacity of 60 and a refill rate of 1 per second."*

**TTL** (abbr.) — Time-To-Live. The maximum lifespan of a cache entry, session token, checkpoint, or knowledge base document before it is invalidated or deleted. *Context: "Knowledge base entries have a default TTL of 30 days."*

## U

**ULID** (n.) — Universally Unique Lexicographically Sortable Identifier. A 26-character, time-sortable identifier used for correlation IDs, task IDs, and audit log entries. *Context: "The correlation_id `01HXYZ...` is a ULID that encodes the timestamp of the run."*

## V

**Vector Store** (n.) — A database optimized for storing and querying embedding vectors via similarity search. AI Dev OS supports multiple Vector Store backends (SQLite with vector extension, Qdrant). *Context: "The RAG Pipeline queries the Vector Store for the top 5 most similar documents."*

## W

**WAL (Write-Ahead Log)** (n.) — A durability mechanism used by SQLite and other storage engines. All writes go to the WAL first before being checkpointed to the main database. *Context: "The SQLite WAL enabled concurrent reads during the write-heavy merge operation."*

**Warm Cache** (n.) — A cache that contains frequently accessed data and returns hit rates above 90%. Warm caches result from steady-state operation after an initial warm-up period. *Context: "The model catalog cache is warm after the first discovery cycle, reducing subsequent lookup latency to under 1 ms."*

**Warm Pool** (n.) — A pre-allocated set of worker processes kept ready to accept tasks immediately, reducing cold-start latency. *Context: "The Kernel maintains a Warm Pool of 3 Dynamic Workers."*

**Web Intelligence** (n.) — A subsystem that performs live web searches, fetches pages, and extracts structured information. Used by the Research Engine and during the Intake phase. *Context: "Web Intelligence fetched the latest API documentation from the vendor website."*

**Worker** (n.) — A process that executes tasks on behalf of an agent. Workers may be long-running (daemon) or dynamic (spawned per task). *Context: "The Worker loaded the model and began streaming the response."*

**WorkerSpec** (n.) — The declarative configuration for a worker: model binding, context size, tool list, timeout, and resource limits. *Context: "The WorkerSpec for the Builder agent includes the file-edit tool and a 5-minute timeout."*

**Workspace** (n.) — An isolated environment with its own agents, knowledge bases, secrets, and configuration. Workspaces map to projects or teams. *Context: "Each workspace has its own vector store namespace and cost budget."*

## X-Z

**Exponential Backoff** (n.) — A retry strategy where the wait time between retries increases exponentially (e.g., 1s, 2s, 4s, 8s) to reduce load on recovering systems. Always combined with jitter to prevent thundering herds. *Context: "The provider adapter retries with exponential backoff starting at 1 second, doubling each attempt, capped at 60 seconds."*

**Error Budget** (n.) — The acceptable amount of service unavailability or degraded performance over a defined window, calculated as 100% minus the SLO target. Error budgets are consumed by incidents and replenished over time. *Context: "The 99.9% SLO gives a 43-minute monthly error budget; the team has consumed 12 minutes so far."*

**OAuth Flow** (n.) — The OAuth 2.0 authorization framework used for obtaining delegated access to provider APIs. AI Dev OS supports the authorization code flow for web-based provider configuration. *Context: "The Anthropic provider adapter uses the OAuth authorization code flow to obtain a refresh token on first setup."*

**Device Code Flow** (n.) — An OAuth 2.0 grant type designed for CLI and headless environments. The user is shown a code and URL, authenticates in their browser, and the CLI polls for the access token. *Context: "On first run, the CLI starts a device code flow: visit https://anthropic.com/activate and enter code XYZ-ABC."*

**Format Translation** (n.) — The process of converting between provider-specific model request/response formats (e.g., OpenAI chat format to Anthropic messages format). Performed by the Nine Router's format translation layer. *Context: "The format translation layer converts the OpenAI-style tool call into Anthropic's tool_use block format."*

**Content Addressing** (n.) — A storage model where content is identified by a cryptographic hash of its data rather than a location (URL or path). Enables deduplication, integrity verification, and immutable references. *Context: "Knowledge base chunks are content-addressed using SHA-256 hashes, ensuring the same document never occupies duplicate storage."*

**Merkle Tree** (n.) — A tree data structure where each leaf node is labelled with a data block's hash and each non-leaf node is labelled with the cryptographic hash of its child nodes. Used for efficient data integrity verification in distributed systems. *Context: "The sync protocol uses a Merkle tree to efficiently compare record sets between nodes."*

**Bloom Filter** (n.) — A space-efficient probabilistic data structure used to test set membership. May produce false positives but never false negatives. Used for cache existence checks and deduplication. *Context: "The write-ahead log uses a bloom filter to quickly check whether a record ID has already been processed."*

**Coordinated Omission** (n.) — A measurement bias where requests that are delayed or dropped due to system overload are excluded from latency statistics, making the system appear faster than it actually is. Mitigated by keeping request timing independent of server response. *Context: "To avoid coordinated omission, the load tester records request start times before sending and includes all results in the latency histogram."*

**Head-of-Line Blocking** (n.) — A performance degradation where a slow or pending request at the front of a queue blocks subsequent requests from being processed, even if they could be served faster. *Context: "The streaming response adapter mitigates head-of-line blocking by reading from multiple provider connections concurrently."*

**Circuit State** (n.) — The current state of a circuit breaker: CLOSED (normal operation), OPEN (failures exceeded threshold, requests blocked), or HALF_OPEN (probing for recovery). *Context: "The OpenAI adapter's circuit breaker transitioned from OPEN to HALF_OPEN after the 30-second cooldown period."*

**Cold Cache** (n.) — A cache that has just been initialized or flushed, resulting in near-zero hit rates until it is warmed by actual usage. Cold caches cause elevated latency for the first requests. *Context: "After the restart, all caches were cold; the first 100 requests experienced 5x normal latency."*

**Cache Stampede** (n.) — A failure pattern where a popular cache entry expires and multiple concurrent requests all attempt to recompute it simultaneously, overwhelming the backend. Mitigated with mutex locks, probabilistic early expiration, and background refresh. *Context: "The database CPU spiked to 100% due to a cache stampede when the model catalog TTL expired."*

**Concurrency Limit** (n.) — The maximum number of concurrent operations allowed for a resource. Exceeding the concurrency limit causes additional requests to be queued or rejected. *Context: "The Ollama provider has a concurrency limit of 4 simultaneous inference requests."*

**Deadline Propagation** (n.) — The practice of carrying a request's timeout deadline across service boundaries so that downstream services can cancel work when the overall deadline has passed. *Context: "The Kernel sets a 30-second deadline on the run; the Nine Router propagates this deadline to the model provider call."*

**Causal Consistency** (n.) — A consistency model where causally related operations are seen by all nodes in the same order, while concurrent operations may be seen in different orders. Weaker than sequential consistency but stronger than eventual consistency. *Context: "The SCE event log provides causal consistency: if event A caused event B, all agents see A before B."*

**Vector Clock** (n.) — A data structure for tracking the causal history of a distributed value. Each node maintains a counter; the vector clock is the set of (node, counter) pairs. Used for detecting concurrent updates in conflict resolution. *Context: "The sync protocol uses vector clocks to detect concurrent writes to the same memory record."*

**Gossip Protocol** (n.) — A communication protocol where each node periodically exchanges state information with a random subset of peers. Information spreads exponentially. Used for cluster membership and metadata propagation. *Context: "The cluster uses a gossip protocol to propagate Nine Router availability information within seconds."*

**Raft Consensus** (n.) — A distributed consensus algorithm for managing a replicated log across a cluster of nodes. Provides strong consistency through leader election and log replication. Used for critical metadata that requires linearizability. *Context: "The SCE snapshot metadata is replicated using Raft consensus across the three controller nodes."*

**CRDT (Conflict-Free Replicated Data Type)** (n.) — A data structure that can be concurrently updated by multiple nodes without coordination, and where all replicas converge to the same state after applying all updates. *Context: "The agent heartbeat tracking uses a CRDT counter so that any node can count heartbeats without central coordination."*

**Bulkhead** (n.) — A resilience pattern that isolates system resources into independent pools so that a failure in one pool does not cascade to others. Named after ship bulkhead compartments. *Context: "Each provider adapter has its own connection pool bulkhead, preventing one provider's latency spike from exhausting all connections."*

**Rate Limit** (n.) — A restriction on the number of requests that can be made to a resource within a given time window. Enforced at multiple levels: provider API limits, per-agent limits, and per-workspace limits. *Context: "The Anthropic API enforces a rate limit of 1,000 requests per minute on the tier 3 plan."*
