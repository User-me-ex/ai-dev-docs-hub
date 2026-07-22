# Getting Started

> Get AI Dev OS running in under five minutes — from zero to your first `aidevos run "hello world"`.

## Overview

AI Dev OS is a local-first AI development operating system that orchestrates multiple models, manages agent groups, persists memory, and runs goals through a kernel-based pipeline. This guide walks you through installation, initial setup, and your first run.

## Prerequisites

| Requirement | Details |
|-------------|---------|
| **Operating System** | macOS 13+ (Apple Silicon & Intel), Windows 10/11 (WSL2), Linux (x86_64 & aarch64) |
| **Terminal** | A modern terminal emulator (iTerm2, Windows Terminal, GNOME Terminal) |
| **Ollama (recommended)** | Install [Ollama](https://ollama.com) for local model inference. Pull at least `llama3.2:3b` for a functional local setup. |
| **Network** | Outbound HTTPS access for cloud provider API calls (optional if using only local models) |
| **Disk** | ~150 MB for the binary; additional space for model storage (varies by provider) |

## Quick Install

Choose one of the following methods:

```bash
# Option A — curl/sh (macOS & Linux)
curl -fsSL https://aidevos.dev/install.sh | sh

# Option B — Homebrew (macOS & Linux)
brew install aidevos/tap/aidevos

# Option C — Direct binary download
# Download the latest release for your platform from:
# https://github.com/aidevos/aidevos/releases/latest
# Then move the binary to /usr/local/bin (macOS/Linux) or a PATH directory (Windows)
```

After install, verify the binary is on your PATH:

```bash
aidevos --version
```

## Initial Setup: `aidevos init`

Run the setup wizard to scaffold your local environment:

```bash
aidevos init
```

The interactive wizard does the following:

1. **Creates `~/.aidevos/`** — the home directory for config, data, keys, and plugins.
2. **Generates `config.toml`** — with sensible defaults. You are prompted for:
   - Default model provider (Ollama is the recommended starter).
   - Ollama endpoint (defaults to `http://localhost:11434`).
   - Optional cloud provider API keys.
3. **Pulls default models** — if Ollama is detected, `aidevos init` can pull a recommended starter model.
4. **Runs `aidevos doctor`** — a health check that validates your setup.

Example output:

```
$ aidevos init
✔ Created ~/.aidevos
✔ Created ~/.aidevos/config.toml
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

[providers.ollama]
endpoint = "http://localhost:11434"
model = "llama3.2:3b"
```

See [Configuration](./CONFIGURATION.md) for the full reference.

## Connecting Cloud Providers (Optional)

AI Dev OS supports cloud providers alongside local models. Add API keys via `aidevos init --configure-providers` or by editing `config.toml`:

```toml
[providers.openai]
api_key = "${OPENAI_API_KEY}"
model = "gpt-4o"

[providers.anthropic]
api_key = "${ANTHROPIC_API_KEY}"
model = "claude-sonnet-4-20250514"
```

Environment variables are resolved at runtime. Supported providers are documented in [Model Providers](./MODEL_PROVIDERS.md).

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

## Related Documents

- [Installation](./INSTALLATION.md) — detailed system requirements and install methods
- [CLI](./CLI.md) — full command reference
- [Local Dev](./LOCAL_DEV.md) — contributing to AI Dev OS itself
- [FAQ](./FAQ.md) — frequently asked questions
- [Configuration](./CONFIGURATION.md) — full config reference
- [Troubleshooting](./TROUBLESHOOTING.md) — deeper diagnostic guide
