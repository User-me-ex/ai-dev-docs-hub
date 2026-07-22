# Getting Started

> Get AI Dev OS running in under five minutes — from zero to your first `aidevos run "hello world"`.

## Overview

AI Dev OS is a local-first AI development operating system that orchestrates multiple models, manages agent groups, persists memory, and runs goals through a kernel-based pipeline. This guide walks you through installation, initial setup, and your first run.

## Prerequisites

| Requirement | Details |
|-------------|---------|
| **Operating System** | macOS 13+ (Apple Silicon & Intel), Windows 10/11 (WSL2), Linux (x86_64 & aarch64) |
| **Terminal** | A modern terminal emulator (iTerm2, Windows Terminal, GNOME Terminal) |
| **Ollama (required)** | Install [Ollama](https://ollama.com) for local model inference. Pull at least `llama3.2:3b` for a functional local setup. |
| **Nine Router** | Install and run Nine Router (localhost:20128) — the model gateway. See [INSTALLATION.md](./INSTALLATION.md). |
| **Network** | Outbound HTTPS access for optional cloud provider API calls (not needed with local models) |
| **Disk** | ~150 MB for the binary; additional space for model storage (varies by provider) |

## Step 1: Install Ollama

```bash
# macOS / Linux
curl -fsSL https://ollama.com/install.sh | sh

# Windows: download from https://ollama.com/download
```

Start Ollama and pull a starter model:

```bash
ollama pull llama3.2:3b
```

## Step 2: Install AI Dev OS

Choose one of the following methods:

```bash
# Option A — npm (recommended)
npm install -g aidevos

# Option B — curl/sh (macOS & Linux)
curl -fsSL https://aidevos.dev/install.sh | sh

# Option C — Homebrew (macOS & Linux)
brew install aidevos/tap/aidevos

# Option D — Direct binary download
# Download the latest release for your platform from:
# https://github.com/aidevos/aidevos/releases/latest
# Then move the binary to /usr/local/bin (macOS/Linux) or a PATH directory (Windows)
```

After install, verify the binary is on your PATH:

```bash
aidevos --version
```

## Step 3: Install and Start Nine Router

Nine Router is the model gateway that AI Dev OS uses to route all model requests. See [INSTALLATION.md](./INSTALLATION.md) for setup instructions. Ensure it's running at `http://localhost:20128`.

## Initial Setup: `aidevos init`

Run the setup wizard to scaffold your local environment:

```bash
aidevos init
```

The interactive wizard does the following:

1. **Creates `~/.aidevos/`** — the home directory for config, data, keys, and plugins.
2. **Generates `config.toml`** — with sensible defaults. You are prompted for:
   - Nine Router endpoint (defaults to `http://localhost:20128`).
   - Default model provider (Ollama is the default).
   - Ollama endpoint (defaults to `http://localhost:11434`).
3. **Pulls default models** — if Ollama is detected, `aidevos init` can pull a recommended starter model.
4. **Runs `aidevos doctor`** — a health check that validates your setup.

Example output:

```
$ aidevos init
✔ Created ~/.aidevos
✔ Created ~/.aidevos/config.toml
✔ Detected Nine Router at http://localhost:20128
✔ Detected Ollama at http://localhost:11434
? Pull default model (llama3.2:3b)? [Y/n] Y
✔ Pulled llama3.2:3b
✔ Running doctor...
✔ All checks passed
```

## First Run: `aidevos run "hello world"`

Once setup is complete, submit your first goal:

```bash
aidevos run "hello world"
```

The Kernel processes the goal through its intake → plan → route → execute → critique pipeline. You will see streaming output as agents work:

```
┌─ Intake ──────────────────────────
│ Goal: hello world
├─ Plan ────────────────────────────
│ 1. Parse the greeting intent
│ 2. Generate a friendly response
├─ Route ───────────────────────────
│ Assigned to: general (llama3.2:3b)
├─ Execute ─────────────────────────
│ Hello! How can I help you today?
└─ Result ──────────────────────────
✔ Completed in 1.2s
```

To see all runs:

```bash
aidevos runs list
```

## Configuration Basics

AI Dev OS reads configuration from up to five sources merged in order of precedence:

| Priority | Source | Path |
|----------|--------|------|
| Highest | Environment variables | `AIDEVOS_*` |
| | Project config | `{project_root}/.aidevos.toml` |
| | User config | `~/.aidevos/config.toml` |
| | System config | `/etc/aidevos/config.toml` |
| Lowest | Embedded defaults | Binary |

Edit `~/.aidevos/config.toml` to change providers, router strategy, logging, and more. Changes to watched keys reload automatically.

```toml
[backend]
mode = "server"
host = "127.0.0.1"
port = 8374

[router]
endpoint = "http://localhost:20128"

[providers.ollama]
endpoint = "http://localhost:11434"
model = "llama3.2:3b"
```

All model requests go through Nine Router at `localhost:20128/v1`. See [Configuration](./CONFIGURATION.md) for the full reference.

## Optional: Add Cloud Providers

Cloud providers are optional and configured inside Nine Router's dashboard (`http://localhost:20128`). AI Dev OS does not store cloud API keys directly — Nine Router handles provider authentication, fallback chains, and routing.

To add a cloud provider:
1. Open Nine Router dashboard at `http://localhost:20128`
2. Add your provider (OpenAI, Anthropic, etc.) and API key
3. Configure routing rules to assign cloud models to specific roles

See [Nine Router documentation](./NINE_ROUTER.md) for details.

## Common Next Steps

| Step | Guide |
|------|-------|
| Learn the CLI | [CLI Reference](./CLI.md) |
| Set up agent groups | [AI Groups](./AI_GROUPS.md) |
| Configure the Nine Router | [Nine Router](./NINE_ROUTER.md) |
| Add project-specific config | [Configuration](./CONFIGURATION.md) |
| Explore the knowledge system | [Knowledge System](./KNOWLEDGE_SYSTEM.md) |
| Write custom prompts | [Prompt Governance](./PROMPT_GOVERNANCE.md) |

## Troubleshooting Common Issues

### `aidevos: command not found`

Ensure the install directory is on your `PATH`. On macOS/Linux, the installer defaults to `/usr/local/bin`. Verify with `which aidevos` or re-run the installer.

### `doctor` reports "Ollama not reachable"

Confirm Ollama is running: `ollama list`. Start it with `ollama serve` if needed. Check the endpoint in `~/.aidevos/config.toml` matches your Ollama configuration.

### Run hangs at "Waiting for provider"

The selected model may not be pulled. Run `ollama pull <model>` or switch to an available model in config.

### Config changes not taking effect

Config files are watched for changes; some keys require a restart. Run `aidevos doctor --verbose` to see which config values are active.

### "Connection refused" errors

The backend process may not be running. Start it with `aidevos server start` in a separate terminal or use `aidevos run --mode=direct` to skip the server.

## Quickstart Checklist

Before your first run, complete these steps:

- [ ] Install Ollama and pull a starter model: `ollama pull llama3.2:3b`
- [ ] Install AI Dev OS (choose one method: npm, curl/sh, Homebrew, or binary download)
- [ ] Verify installation: `aidevos --version`
- [ ] Install and start Nine Router (see INSTALLATION.md)
- [ ] Run `aidevos init` to scaffold config and detect Nine Router + Ollama
- [ ] Ensure Ollama is running: `ollama serve` or check service status
- [ ] Run `aidevos doctor` to validate setup
- [ ] Submit your first goal: `aidevos run "hello world"`
- [ ] Explore: `aidevos models list`, `aidevos router show`

## First-Run Wizard Walkthrough

When you run `aidevos init` for the first time, the interactive wizard guides you through:

1. **Welcome screen** — Explains what AI Dev OS is and what the wizard will do.
2. **Config directory** — Creates `~/.aidevos/` with default config.
3. **Nine Router detection** — Verifies Nine Router is reachable at localhost:20128.
4. **Provider setup** — Detects Ollama automatically. Cloud providers are configured separately in Nine Router.
5. **Model pull** — Offers to pull a default model (llama3.2:3b) if Ollama is running.
6. **Health check** — Runs `aidevos doctor` automatically.
7. **Completion** — Shows summary of what was configured and suggests next commands.

```
$ aidevos init
╭─ AI Dev OS Setup Wizard ─────────────────────────╮
│                                                    │
│  Welcome! Let's get you set up in a few steps.     │
│                                                    │
│  Step 1/5: Creating ~/.aidevos/  ✓                │
│  Step 2/5: Detecting Nine Router...  ✓ (localhost:20128)│
│  Step 3/5: Detecting Ollama...  ✓ (localhost:11434)│
│  Step 4/5: Configuring providers  ✓               │
│  Step 5/5: Running health check  ✓                │
│                                                    │
│  All done! Try: aidevos run "hello world"          │
╰────────────────────────────────────────────────────╯
```

## Sample Project Creation

Create your first AI Dev OS project:

```bash
# Create a project directory
mkdir my-first-project
cd my-first-project

# Initialize project-specific config
aidevos config set project.name "My First Project"

# Create a custom agent for this project
cat > .aidevos-agent.yaml << 'EOF'
name: helper
model: llama3.2:3b
temperature: 0.7
allowed_tools:
  - fs.read_file
  - fs.write_file
  - code.search
EOF

# Register the agent
aidevos agent register helper --file .aidevos-agent.yaml

# Run a project-specific goal
aidevos run "Create a hello world Python script"
```

## Common First Tasks

| Task | Command |
|------|---------|
| Run your first goal | `aidevos run "hello world"` |
| List available models | `aidevos models list` |
| Check system health | `aidevos doctor --full` |
| View run history | `aidevos runs list` |
| Create a persistent memory entry | `aidevos memory write --kb main --key "project:hello" --value "My first project"` |
| Search memory | `aidevos memory query "hello world"` |
| View SCE topics | `aidevos context topics` |
| Configure cloud providers | Use Nine Router dashboard at `http://localhost:20128` |

## Learning Path Recommendations

| Level | Focus | Resources |
|-------|-------|-----------|
| **Beginner** | Basic goals, CLI usage, model setup | GETTING_STARTED.md, CLI.md, FAQ.md |
| **Intermediate** | Agent groups, Nine Router, memory | AI_GROUPS.md, NINE_ROUTER.md, PERSISTENT_MEMORY.md |
| **Advanced** | Custom prompts, Guardian rules, plugins | PROMPT_GOVERNANCE.md, ARCHITECTURE_GUARDIAN.md, PLUGIN_SDK.md |
| **Expert** | Multi-agent orchestration, deployment | MULTI_AGENT_ORCHESTRATION.md, DEPLOYMENT.md, MAIN_AI_KERNEL.md |

## Next Steps Guide

After completing the getting started guide:

1. **Explore the CLI** — Run `aidevos --help` to see all commands
2. **Configure your router** — `aidevos router assign builder llama3.2:3b`
3. **Set up agent groups** — `aidevos groups create my-team --members builder reviewer tester`
4. **Write custom prompts** — Create prompt files in `~/.aidevos/prompts/`
5. **Connect an MCP server** — `aidevos mcp connect filesystem`
6. **Deploy in server mode** — `aidevos server start --daemon`

## Related Documents

- [Installation](./INSTALLATION.md) — detailed system requirements and install methods
- [CLI](./CLI.md) — full command reference
- [Local Dev](./LOCAL_DEV.md) — contributing to AI Dev OS itself
- [FAQ](./FAQ.md) — frequently asked questions
- [Configuration](./CONFIGURATION.md) — full config reference
- [Troubleshooting](./TROUBLESHOOTING.md) — deeper diagnostic guide
