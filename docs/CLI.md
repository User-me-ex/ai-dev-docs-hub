# CLI

> `aidevos` — one binary, subcommand tree, both a friendly TTY UI and a machine-readable `--json` mode. Every UI action is reachable from the CLI.

## Overview

The CLI is the primary local surface for AI Dev OS. It is a thin adapter over the [Main AI Kernel](./MAIN_AI_KERNEL.md) syscalls, so anything the desktop UI can do can be scripted. The CLI is also the offline-first fallback when the desktop app is unavailable.

## Goals

- One binary, discoverable subcommand tree.
- First-class `--json` mode for scripting; TTY-aware pretty output otherwise.
- Every UI action reachable from the CLI.
- Zero-config against a local backend (`~/.aidevos/config.toml`).
- Stable exit codes.

## Non-Goals

- Implementation code (this repo is documentation-only).
- Provider-specific commands — provider knobs live in [Model Providers](./MODEL_PROVIDERS.md) and are surfaced through generic subcommands.

## Requirements

- **MUST** read config from `~/.aidevos/config.toml` and env vars (`AIDEVOS_*`), with env taking precedence.
- **MUST** support `--json`, `--quiet`, `--verbose`, `--profile <name>`, `--no-color`.
- **MUST** exit non-zero with a documented code on any failure; codes stable across minor versions.
- **MUST** stream Kernel events to stdout when a subcommand blocks on a run.
- **SHOULD** ship shell completions for bash, zsh, fish.
- **MAY** expose a REPL (`aidevos shell`) that maintains a Kernel session across commands.

## Command Tree

```
aidevos
├── init                                # scaffold ~/.aidevos and connect to local backend
├── doctor                              # environment + provider health check
├── run <goal> [--group ID] [--budget]  # submit a goal to the Kernel
├── runs
│   ├── list [--state ...]
│   ├── show <run_id>
│   ├── stream <run_id>
│   ├── cancel <run_id>
│   └── replay <run_id> [--from EVENT]
├── models                              # Nine Router / Model Discovery
│   ├── list [--provider P] [--capability C] [--json]
│   ├── refresh [--provider P]
│   ├── show <model_id>
│   └── search <query>
├── router
│   ├── roles                           # list the nine roles + current assignment
│   ├── assign <role> <model_id> [--project P]
│   ├── fallbacks <role> <model_id> ...
│   └── show
├── groups
│   ├── list
│   ├── create <name> [--members ...]
│   └── show <group_id>
├── memory
│   ├── query <q> [--kb global|main|group|individual]
│   ├── write --kb <kb> --key <k> --value <v>
│   └── export <path>
├── context
│   ├── topics
│   ├── tail <topic>
│   └── snapshot <topic>
├── research <query> [--depth N]        # Research & Web Intelligence
├── github
│   ├── analyze <repo>
│   └── watch <repo>
├── plugins
│   ├── list
│   ├── install <ref>
│   └── enable|disable <name>
├── mcp
│   ├── servers
│   ├── connect <ref>
│   └── call <server> <tool> --args @file.json
├── voice
│   ├── listen                          # push-to-talk STT session
│   └── say <text>
├── secrets
│   ├── set <NAME>
│   └── list
├── config
│   ├── get <path>
│   └── set <path> <value>
└── version
```

## Interfaces

Every command supports `--json`; JSON output is a single object with `{ ok, data?, error? }` and never mixes with human text.

## Data Model

- Config: `~/.aidevos/config.toml`
  - `[backend]` `endpoint`, `token_file`
  - `[profiles.<name>]` overrides
  - `[router]` default role assignments
  - `[providers.<id>]` base URLs and secret refs
- Cache: `~/.cache/aidevos/` (models, prompts, MCP)
- State: `~/.local/state/aidevos/` (WAL, cursors)

## Exit codes

| Code | Meaning                              |
| ---- | ------------------------------------ |
| 0    | Success                              |
| 1    | Generic error                        |
| 2    | Usage error (bad flags/args)         |
| 3    | Config error                         |
| 4    | Auth error                           |
| 5    | Backend unreachable                  |
| 6    | Guardian veto (run refused)          |
| 7    | Budget exhausted                     |
| 8    | Cancelled                            |
| 9    | Timeout                              |

## Failure Modes

- Backend down → `aidevos doctor` explains next step; commands that don't need the backend still work (`models list --cache`).
- Provider outage → `aidevos models refresh` reports per-provider status and keeps last-known-good cache.

## Security

- Tokens are stored via the OS keychain when available; fall back to `chmod 600` file.
- `aidevos secrets set` never echoes; secrets never appear in `--json` output.
- All commands are audited into the [Audit Log](./AUDIT_LOG.md) with the correlation id.

## Observability

`aidevos --verbose` prints the correlation id for every command; every command emits a `cli.command` event on the Shared Context Engine.

## Acceptance Criteria

- `aidevos init && aidevos doctor` succeeds on a fresh machine with only Ollama installed.
- `aidevos models list --json | jq '.data | group_by(.provider)'` produces the same grouping as the UI.
- `aidevos router assign builder ollama/llama3.1:8b` immediately reflects in `aidevos router show`.

## Related Documents

- [Backend](./BACKEND.md) · [API Spec](./API_SPEC.md) · [Getting Started](./GETTING_STARTED.md) · [Nine Router](./NINE_ROUTER.md) · [Model Discovery](./MODEL_DISCOVERY.md) · [MCP](./MCP.md) · [Secrets Management](./SECRETS_MANAGEMENT.md) · [Main AI Kernel](./MAIN_AI_KERNEL.md)
