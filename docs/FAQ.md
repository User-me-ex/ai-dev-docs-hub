# Frequently Asked Questions

## General

**What is AI Dev OS?**
AI Dev OS is an open-source, offline-first operating system for AI-assisted software development. It provides a multi-agent orchestration engine, a sandboxed runtime, a plugin system, and a unified CLI — all designed to run on your own hardware without sending data to third parties unless you explicitly configure it.

**How is it different from other AI coding tools?**
Most AI coding tools are SaaS products or IDE plugins that require internet and send your code to external servers. AI Dev OS runs entirely on your machine, supports any model provider (local or cloud), coordinates multiple specialist agents on complex workflows, and gives you full control over data, security, and cost. It is not an editor plugin — it is a standalone orchestration platform.

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
Any model accessible via an OpenAI-compatible API, plus native support for Ollama, llama.cpp, Anthropic Claude, Google Gemini, Mistral AI, Groq, Together AI, and AWS Bedrock. Local models run through Ollama or llama.cpp.

**Do I need an API key?**
Only for cloud providers. Local models require no API key. Set keys via `ai-dev-os config set PROVIDER_KEY=<value>` or as environment variables. The CLI will prompt you for missing keys on first use.

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
- Cloud provider may be rate-limiting your plan tier
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

---

**Related Documents:** [Getting Started](GETTING_STARTED.md), [Troubleshooting](TROUBLESHOOTING.md), [Support](SUPPORT.md)
