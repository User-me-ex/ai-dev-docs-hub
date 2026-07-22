# Frequently Asked Questions

## General

**What is AI Dev OS?**
AI Dev OS is an open-source, offline-first operating system for AI-assisted software development. It provides a multi-agent orchestration engine, a sandboxed runtime, a plugin system, and a unified CLI — all designed to run on your own hardware without sending data to third parties unless you explicitly configure it.

**How is it different from other AI coding tools?**
Most AI coding tools are SaaS products or IDE plugins that require internet and send your code to external servers. AI Dev OS runs entirely on your machine, supports any model provider (local or cloud via Nine Router), coordinates multiple specialist agents on complex workflows, and gives you full control over data, security, and cost. It is not an editor plugin — it is a standalone orchestration platform.

**Is it free?**
Yes. AI Dev OS is open source under the Apache 2.0 license. You pay only for the model API keys you choose to use. Running local models via Ollama or llama.cpp is entirely free.

**Do I need internet?**
No. AI Dev OS works fully offline when using local models. Internet is only required when calling cloud model APIs, installing packages, pulling Docker images, or cloning repositories.

**Who maintains it?**
AI Dev OS is maintained by the AI Dev OS Foundation, a community-driven organization with contributors from around the world. Governance is documented in the project's CHARTER.md.

**Can I use it for commercial projects?**
Yes. The Apache 2.0 license permits commercial use, modification, and distribution. No enterprise license is required.

**Does it have a GUI?**
Yes. Run `ai-dev-os ui` to launch a local web dashboard at `http://localhost:8080`. The dashboard provides run history, agent monitoring, memory inspection, and configuration management.

**What operating systems are supported?**
Windows 10+, macOS 12+, and Linux (x86_64 and ARM64). Official builds are provided for all three platforms.

## Setup

**How do I install it?**
```bash
curl -fsSL https://get.ai-dev-os.sh | sh
```
Or download the latest release from GitHub Releases. Windows users can use the MSI installer or `winget install ai-dev-os`.

**What models can I use?**
Any model accessible via the Nine Router gateway (localhost:20128/v1). Local models run through Ollama or llama.cpp by default. Cloud providers (OpenAI, Anthropic, Google, etc.) are optional and configured inside Nine Router.

**Do I need an API key?**
No. By default, AI Dev OS uses local models via Nine Router (port 20128). No API key is required for local inference (Ollama, llama.cpp, etc.). Cloud provider keys are optional and configured entirely within Nine Router's dashboard. The CLI does not store or manage cloud provider keys directly.

**What are the minimum system requirements?**
- 4 GB RAM (16 GB+ recommended when using local models)
- 2 GB free disk space (plus additional space for models)
- A CPU with at least 4 cores
- Docker (optional, required for sandboxed execution)

**How do I uninstall?**
```bash
ai-dev-os uninstall
```
Or manually delete the install directory and `~/.ai-dev-os/`. Use `--purge` to also remove all data, memory, and configuration.

**Can I install it without admin rights?**
Yes. Use the `--prefix` flag to install to a user-local directory. The default install on Linux without sudo places binaries in `~/.local/bin/`.

## Usage

**How do I run a task?**
```bash
ai-dev-os run "Refactor the authentication module to use JWT"
```
The orchestrator parses the request, generates a plan, assigns agents, and executes the workflow. Use `ai-dev-os run --interactive` for step-by-step mode.

**Can multiple agents work together?**
Yes. The built-in orchestrator spawns and coordinates specialist agents — a planner, a coder, a reviewer, a tester, and a documentation agent — on complex tasks. You can define custom agent roles and inter-agent dependencies in `agents.yaml`.

**How do I add custom tools?**
1. Create a tool definition YAML in `~/.ai-dev-os/tools/`
2. Implement the tool's logic as a Python script, shell script, or binary
3. Register it: `ai-dev-os tools register my-tool.yaml`
4. Reference the tool in an agent's `allowed_tools` list

**How do I configure agents?**
Edit `~/.ai-dev-os/agents.yaml` or use `ai-dev-os agent configure <name>`. You can set the model, temperature, max tokens, system prompt, allowed tools, memory settings, and concurrency limits.

**Can I pause and resume a task?**
Yes. Runs are checkpointed at each step. Use `ai-dev-os pause <run-id>` and `ai-dev-os resume <run-id>`. Resuming restores the full agent state, including conversation history and intermediate results.

**How do I share my configuration across teams?**
Export your config with `ai-dev-os config export > team-config.yaml`. Team members import it with `ai-dev-os config import team-config.yaml`. Provider keys are excluded from export by default.

**What is the run queue?**
When multiple tasks are submitted, they enter a queue. Use `ai-dev-os queue list` to view pending runs and `ai-dev-os queue priority <run-id> high` to reorder them.

## Technical

**How does it ensure safety?**
AI Dev OS has a layered safety model:
1. The Guardian agent reviews every tool call against configurable safety policies before execution
2. The sandbox container (via Docker) isolates execution from the host system
3. File system access is restricted to the project directory by default
4. Network access can be restricted per agent

**Can it run completely offline?**
Yes. All core features work offline when using local models. Install plugins and tools while online, then disconnect. Telemetry is opt-in and can be disabled entirely via `ai-dev-os config set telemetry.enabled false`.

**What data is stored and where?**
All data resides in `~/.ai-dev-os/`:
- Config: `config.yaml`
- Agent definitions: `agents.yaml`
- Run history and logs: `runs/`
- Safety policies: `policies.yaml`
- Plugin data: `plugins/`
- Vector store and memory: `memory/`
No data is sent to external services unless you configure a cloud model provider or opt into telemetry.

**How does the agent orchestration work?**
The planner decomposes a task into subtasks and produces a DAG. The dispatcher assigns each subtask to the most capable agent based on tool availability and expertise. The supervisor monitors progress, retries failed steps, handles agent timeouts, and aggregates results into the final output.

**Can I write my own plugins?**
Yes. Plugins are Python or JavaScript modules implementing the Agent or Tool interface. See the Plugin Development Guide. Plugins are sandboxed and versioned per run.

**What is a checkpoint and how often is it created?**
A checkpoint is a snapshot of the run state including agent memory, conversation history, and intermediate artifacts. Checkpoints are created after every tool call or agent handoff. They enable pause/resume and rollback.

**How does memory work across sessions?**
AI Dev OS maintains a vector store for long-term memory. Agents can query past runs, decisions, and code artifacts. Memory is namespaced by project and can be reset selectively.

## Troubleshooting

**Why is the model slow?**
- You may be running a large model on CPU. Use a smaller quantized model (e.g., a 7B Q4_K_M instead of a 13B Q8_0)
- Ensure sufficient system RAM is available
- Cloud provider (if configured via Nine Router) may be rate-limiting your plan tier
- Run `ai-dev-os doctor --benchmark` to diagnose

**Why did my run fail?**
Examine the run logs: `ai-dev-os logs <run-id>`. Common causes include budget exhaustion, the Guardian vetoing a tool call, a tool timeout, a provider API error, or the planner entering a loop. Run `ai-dev-os doctor --repair` for automated recovery suggestions.

**How do I reset my configuration?**
```bash
ai-dev-os config reset              # Reset everything to defaults
ai-dev-os config reset --keep-keys  # Reset but retain provider API keys
```
Or manually edit `~/.ai-dev-os/config.yaml`.

**How do I update AI Dev OS?**
```bash
ai-dev-os update
```
Or download the latest release from the GitHub Releases page. The update command supports `--channel stable|beta|nightly`.

**Where are the logs?**
Run logs live at `~/.ai-dev-os/runs/<run-id>/`. The daemon log is `~/.ai-dev-os/daemon.log`. Use `ai-dev-os logs` with filters for quick access.

**What should I do if I encounter a bug?**
Check existing GitHub Issues. If none match, file a new one using the bug report template. Include the output of `ai-dev-os doctor`, relevant run logs, and steps to reproduce.

## Installation

**How do I install on Windows?**
Use the MSI installer from GitHub Releases, or `winget install ai-dev-os`. Alternatively, download the `aidevos-x86_64-windows.zip` and extract to a PATH directory.

**How do I install on macOS?**
`brew install aidevos/tap/aidevos` or `curl -fsSL https://aidevos.dev/install.sh | sh`.

**How do I install on Linux?**
Use the install script, download the `.tar.gz` from GitHub Releases, or use your distribution's package manager if available.

**Can I run AI Dev OS in Docker?**
Yes. A container image is published at `ghcr.io/anomalyco/ai-dev-os:latest`. Run with `docker run -v ~/.aidevos:/home/user/.aidevos ghcr.io/anomalyco/ai-dev-os`.

**How do I install without internet?**
Download the release archive on a connected machine, transfer it to the air-gapped machine, and extract to a PATH directory. Local models must also be pre-downloaded.

**What if I get a "permission denied" error during install?**
Ensure the install directory is writable. Use `--prefix ~/.local` to install to a user-local directory.

## Configuration

**How do I change the default model?**
Configure the default model in Nine Router's dashboard or edit `~/.aidevos/config.toml`: `[router] default_provider = "ollama"`. All model routing goes through Nine Router.

**Where are provider API keys stored?**
Cloud provider keys are stored in Nine Router's configuration. AI Dev OS itself only stores local provider settings (Ollama endpoint, etc.). See Nine Router documentation for key management.

**Can I use different models for different tasks?**
Yes. The Nine Router assigns models to roles. Edit the `[router]` section in config to map specific models to specific roles.

**How do I configure multiple providers?**
Configure cloud providers inside Nine Router's dashboard at `http://localhost:20128`. Nine Router handles all provider routing — AI Dev OS simply calls Nine Router's unified API at `localhost:20128/v1`.

**How do I disable telemetry?**
`aidevos config set telemetry.enabled false` or set `AIDEVOS_TELEMETRY_ENABLED=false`.

## Models

**What local models are supported?**
Any model running via Ollama or llama.cpp with an OpenAI-compatible API. Tested models: Llama 3.2, Mistral, Phi-4, Qwen 2.5, DeepSeek.

**Can I use GPT-4?**
Yes. Configure your OpenAI key inside Nine Router's dashboard. Nine Router then assigns GPT-4 to specific roles based on routing policy. No direct OpenAI API configuration is needed in AI Dev OS.

**How do I add a custom model endpoint?**
Add a provider entry in Nine Router's configuration with the custom `base_url` and model name. AI Dev OS discovers available models from Nine Router automatically.

**What happens if a model provider is unavailable?**
The Nine Router falls back through the configured fallback chain (e.g., cloud → local). If all providers fail, the run fails with a clear error.

**How are costs tracked?**
Each model call records token usage, which is aggregated in the metrics endpoint. Set budget caps per run in `AIDEVOS_RUN_BUDGET_USD`.

## Agents

**How many agents can run concurrently?**
Default is 10, configurable via `AIDEVOS_MAX_WORKERS`. Each agent runs in its own worker process.

**Can I create custom agent roles?**
Yes. Define custom roles in `agents.yaml` with specific system prompts, allowed tools, and model assignments.

**How do agents communicate?**
All inter-agent communication flows through the Shared Context Engine (SCE). Agents publish and subscribe to topic-scoped events.

**Can I run agents without the orchestrator?**
Yes. Use `aidevos agent run <agent-name>` to run a single agent outside the pipeline.

**How are agent timeouts handled?**
If an agent does not respond within its configured timeout (default 300s), the Kernel marks it as failed and retries or escalates.

## Memory

**What types of memory does AI Dev OS support?**
Working memory (per-run), long-term memory (persistent SQLite + vector store), and knowledge bases (GLOBAL_KB, MAIN_KB, GROUP_KB, INDIVIDUAL_KB).

**How long does memory persist?**
Working memory persists within a run. Long-term memory and KB entries persist indefinitely until evicted by retention policy.

**Can I reset memory selectively?**
Yes. `aidevos memory reset --kb <name>` resets a specific knowledge base without affecting others.

**How does the vector store work?**
Text embeddings are generated by the embedding pipeline and stored in a usearch HNSW index. Queries use ANN search with cosine similarity.

**Is memory encrypted at rest?**
Yes. Persistent memory and vector stores are encrypted using the configured encryption key (AES-256-GCM).

## Troubleshooting

**Why is the startup slow?**
First startup may be slower due to model downloads and index building. Subsequent startups should be under 500ms.

**Why are tool calls failing?**
Check that the tool is registered (`aidevos tools list`) and the agent has permission to use it (`agents.yaml` `allowed_tools`).

**Why is the Guardian blocking operations?**
Run `aidevos doctor --guardian` to see active rule violations. Adjust rules in `~/.aidevos/policies.yaml` or add exceptions.

**How do I view real-time logs?**
`aidevos logs --follow` streams all subsystem logs. Use `--level debug` for verbose output.

**What does exit code 6 mean?**
The Architecture Guardian vetoed the operation. Check the Guardian log for the specific rule that was violated.

## Security

**How is data isolated between projects?**
AI Dev OS uses workspace-level isolation. Each workspace has its own database, memory, and configuration.

**Can AI Dev OS access the internet?**
Only model provider calls through Nine Router (if cloud providers are configured) and web research (if enabled). All network access is configurable per agent. By default, with local models, no internet access is required.

**How are API keys protected?**
Keys are stored in the OS keychain or encrypted at rest. They are never logged and never included in telemetry.

**Is there an audit log?**
Yes. Every action is recorded in the append-only Audit Log with correlation_id, agent identity, and timestamp.

**Can I run AI Dev OS in a sandbox?**
Yes. Use `--sandbox docker` to run agent execution inside a Docker container with restricted capabilities.

## Performance

**What is the typical run latency?**
A simple goal with a local model completes in 1-5 seconds. Complex multi-agent workflows may take 30-120 seconds.

**How does it perform under load?**
The priority queue disciplines concurrent runs. The system is tested at 50 concurrent runs with graceful degradation.

**What is the maximum database size?**
SQLite databases are tested up to 50 GB. Vector indexes are tested up to 100k records.

**Does it support read replicas?**
For multi-workspace deployments, optional Postgres read replicas are supported via `AIDEVOS_DB_READ_REPLICA_URL`. The local default (SQLite) does not use replicas.

## Deployment

**Can I run AI Dev OS as a service?**
Yes. Ensure Nine Router is running first, then `aidevos server start --daemon` runs the backend as a background service. Use `aidevos server stop` to shut it down.

**How do I deploy to Kubernetes?**
A Helm chart is available in the repository. See [Deployment](./DEPLOYMENT.md) for Kubernetes configuration.

**Can I run multiple instances?**
Yes. Use different `AIDEVOS_HOME` paths for each instance. Each instance connects to its own Nine Router or shares a central one. The default SQLite store is per-instance; shared backends (Postgres, etc.) are optional.

**What monitoring is available?**
Prometheus metrics at `:9090/metrics`, structured JSON logs, and OpenTelemetry tracing.

**How do I upgrade AI Dev OS?**
`aidevos update` or download the latest release. Database migrations are applied automatically on startup.

---

**Related Documents:** [Getting Started](GETTING_STARTED.md), [Troubleshooting](TROUBLESHOOTING.md), [Support](SUPPORT.md)
