# Folder Structures

> Reference for the directory layouts used by AI Dev OS — the home config vault, workspace vaults, and the project repository itself.

## Overview

AI Dev OS maintains three distinct directory trees: the **user config directory** (`~/.aidevos/`), **workspace vaults** (project-level knowledge stores), and the **repository itself**. Understanding these layouts helps with configuration, debugging, and contributing.

## The `~/.aidevos/` Directory

Created by `aidevos init`. This is the user-level home for all persistent state.

```
~/.aidevos/
├── config.toml           # User configuration (TOML)
├── config.lock.json      # Merged + resolved config (auto-generated)
├── data/
│   ├── aidevos.db        # SQLite database (runs, memory, sessions)
│   └── vector.db         # Vector store (embeddings index)
├── keys/
│   └── ed25519.pem       # Local signing key (auto-generated)
├── secrets/
│   └── (encrypted provider keys, managed by CLI)
├── plugins/
│   ├── index.json        # Installed plugin manifest
│   └── installed/        # Plugin installations (one subdirectory each)
├── rules/
│   └── (user-defined AI coding rules, loaded at startup)
├── mcp.d/
│   └── (MCP server configurations, one file per server)
├── cache/
│   ├── models/           # Cached model metadata
│   └── responses/        # Cached run outputs (TTL-based)
└── logs/
    ├── aidevos.log       # Rotating application log
    └── access.log        # API access log (server mode)
```

### Directory Purposes

| Directory | Purpose |
|-----------|---------|
| `config.toml` | User-level configuration merged with project and system configs. See [Configuration](./CONFIGURATION.md). |
| `config.lock.json` | The resolved, validated config after merging all sources. Read by subsystems. Do not edit manually. |
| `data/` | Persistent storage. `aidevos.db` holds runs, sessions, agent groups, and the fact/entity knowledge graph. `vector.db` holds embedding vectors for semantic search. |
| `keys/` | Auto-generated Ed25519 key pair for signing local requests and envelopes. Regenerated on loss (old signatures become unverifiable). |
| `secrets/` | Encrypted provider API keys and secrets. Managed exclusively through CLI commands; never edit manually. |
| `plugins/` | Plugin registry and installations. `index.json` tracks installed plugins and their versions. See [Plugin SDK](./PLUGIN_SDK.md). |
| `rules/` | User-authored AI coding rules loaded into the Kernel context. Rules are merged with project-level rules at runtime. |
| `mcp.d/` | MCP (Model Context Protocol) server configurations. Each file declares one MCP server endpoint. |
| `cache/` | Ephemeral caches with TTL-based eviction. Safe to delete; re-populated on demand. |
| `logs/` | Application logs. Rotated automatically. Log level and format controlled by `[logging]` config. |

## The Workspace Vault Structure

Each project or workspace gets a vault directory — either `.aidevos/` inside the project root or a dedicated vault path set in config.

```
project-root/
└── .aidevos/                  # Workspace vault (hidden, in project root)
    ├── vault.toml             # Vault metadata (name, description)
    ├── knowledge/
    │   ├── entities/          # Entity definitions (YAML)
    │   ├── facts/             # Fact assertions (YAML)
    │   └── vectors/           # Local vector index
    ├── sessions/
    │   └── (session snapshots, one per active session)
    ├── runs/
    │   └── (recent run logs, synced to global DB)
    └── rules/
        └── (project-specific AI coding rules)
```

Workspace vaults are optional. When present, they layer on top of the global `~/.aidevos/` — data is merged, with the workspace taking precedence for entity/fact resolution.

## The `ai-dev-docs-hub` Repository Structure

This repository — the one containing this document — follows this layout:

```
ai-dev-docs-hub/
├── docs/                      # All documentation (this directory)
│   ├── README.md              # Root index
│   ├── GETTING_STARTED.md     # Getting started guide
│   ├── INSTALLATION.md        # Installation guide
│   ├── CLI.md                 # CLI reference
│   ├── CONFIGURATION.md       # Configuration reference
│   ├── ... (100+ subsystem docs)
│   └── knowledge-bases/       # Knowledge base markdown
├── prompts/                   # Prompt templates and governance
│   ├── templates/             # Reusable prompt fragments
│   ├── system/                # System-level prompts
│   └── governance/            # Prompt policy definitions
├── diagrams/                  # Mermaid and other diagram sources
├── templates/                 # Templates (ADRs, issue templates, etc.)
│   ├── ADR.md                 # Architecture Decision Record template
│   └── issue-template.md      # GitHub issue template
├── src/                       # Source code reference (spec stubs, validation schemas)
│   ├── schemas/               # JSON Schema files for config validation
│   └── examples/              # Example config files and outputs
├── scripts/                   # Utility scripts (lint, validate links, cross-ref)
├── tests/                     # Documentation tests (link checks, schema validation)
├── .github/                   # GitHub workflows, issue templates
├── AGENTS.md                  # Guidance for AI agents editing this repo
└── README.md                  # Repository root README
```

## Related Documents

- [Configuration](./CONFIGURATION.md) — config file format and key reference
- [CLI](./CLI.md) — command reference including vault management
- [Local Dev](./LOCAL_DEV.md) — setting up a development environment
- [Database](./DATABASE.md) — SQLite schema reference
- [Plugin SDK](./PLUGIN_SDK.md) — plugin directory structure and manifest format
- [Secrets Management](./SECRETS_MANAGEMENT.md) — secrets storage encryption
