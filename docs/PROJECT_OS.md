# Project OS

> **Domain:** Workspace & Project Management
> **Applies to:** Kernel, CLI, Persistent Memory
> **Last updated:** 2026-07-22

## Overview

AI Dev OS organizes all work into a two-level hierarchy: **workspaces** (isolated tenants) and **projects** (sub-units within a workspace). This model provides strong isolation between unrelated contexts while allowing controlled sharing within a workspace.

```
ai-dev-os data
  └── ~/.aidevos/data/
       ├── <workspace_id>/
       │   ├── config.toml
       │   ├── main.kb/          # Main Knowledge Base (SQLite)
       │   ├── global.kb/        # Global Knowledge Base (cross-project)
       │   ├── persistent_memory/
       │   ├── vector_index.usearch
       │   └── projects/
       │       ├── <project_id>/
       │       │   ├── config.toml
       │       │   ├── main.kb/  # Project-level Main KB
       │       │   └── artifacts/
       │       └── ...
       └── ...
```

## Workspace Model

A **workspace** is the top-level isolation boundary:

- **Filesystem:** `~/.aidevos/data/<workspace_id>/` — encrypted at rest with a workspace-specific key.
- **Process:** Each workspace runs in a dedicated backend process with its own SCE event bus, agent pool, and memory stores.
- **Identity:** Workspaces have their own Ed25519 identity key used for inter-workspace communications (opt-in).
- **Resources:** Memory limits, CPU quotas, and disk caps are configured per workspace.

A workspace cannot access another workspace's data unless cross-workspace sharing is explicitly enabled via the Global KB bridge.

## Project Model

A **project** is a sub-unit within a workspace:

- **Filesystem:** `~/.aidevos/data/<workspace_id>/projects/<project_id>/`
- **Knowledge base:** Every project has its own **Main KB** (SQLite FTS5-backed knowledge store). See [Knowledge System](KNOWLEDGE_SYSTEM.md).
- **Agent scope:** Agents spawned within a project can read/write that project's Main KB by default.
- **Artifacts:** Build outputs, logs, and generated files live under `artifacts/`.

## Workspace Lifecycle

```
Created ──► Configured ──► Active ──► Deleted
                │                        ▲
                └──► Archived ───────────┘
```

| Stage | Trigger | Effect |
|-------|---------|--------|
| **Create** | `workspace.create(name)` | Generates identity key, provisions encrypted directory, initializes Main KB and Global KB. |
| **Configure** | `workspace.update(id, config)` | Sets resource limits, model provider bindings, agent pool size. |
| **Active** | Automatic after create | Backend process starts, agents available. |
| **Archive** | `workspace.archive(id)` | Backend process stops, data remains on disk, key is sealed. |
| **Delete** | `workspace.delete(id)` | Data directory is zeroed and removed. **Irreversible.** |

## Project Lifecycle

```
Created ──► Active ──► Archived ──► Deleted
```

| Stage | Trigger | Effect |
|-------|---------|--------|
| **Create** | `project.create(workspace_id, name)` | Creates project directory, initializes project-level Main KB. |
| **Active** | Automatic | Agents can be dispatched to this project. |
| **Archive** | `project.archive(workspace_id, project_id)` | KB is compacted and frozen. No new agent work. |
| **Delete** | `project.delete(workspace_id, project_id)` | Project data is removed. Archives are preserved if they were exported. |

## Cross-Project References

Projects within the same workspace can opt in to sharing knowledge:

1. A project publishes a fact to the **Global KB** via `kb.publish(namespace, key, value)`.
2. Other projects subscribe via `kb.subscribe(namespace, key_pattern)`.
3. The Global KB is stored at the workspace level (`~/.aidevos/data/<ws_id>/global.kb/`).
4. Cross-project references are **not automatic** — agents must explicitly publish and subscribe.

## Multi-Workspace Considerations

- **Data isolation:** Workspaces are fully isolated at the filesystem and process level.
- **Cross-workspace bridging:** Must be explicitly configured via the Global KB bridge (requires workspace admin approval on both sides).
- **Resource contention:** The Kernel enforces per-workspace CPU/memory quotas to prevent noisy-neighbor issues.
- **Model provider sharing:** API keys can be shared across workspaces via the Secrets Vault with workspace-scoped access policies.

## Interfaces

### Workspace Operations

| Interface | Description |
|-----------|-------------|
| `workspace.create(name, config?)` | Creates a new workspace. Returns `WorkspaceId`. |
| `workspace.list()` | Lists all workspaces with metadata. |
| `workspace.get(id)` | Returns workspace config and status. |
| `workspace.update(id, config)` | Updates workspace configuration. |
| `workspace.archive(id)` | Archives a workspace (stops processes, seals data). |
| `workspace.delete(id)` | Permanently deletes a workspace. |
| `workspace.stats(id)` | Returns resource usage statistics. |

### Project Operations

| Interface | Description |
|-----------|-------------|
| `project.create(workspace_id, name)` | Creates a new project. Returns `ProjectId`. |
| `project.list(workspace_id)` | Lists projects in a workspace. |
| `project.get(workspace_id, project_id)` | Returns project metadata. |
| `project.archive(workspace_id, project_id)` | Archives a project (freezes KB). |
| `project.delete(workspace_id, project_id)` | Permanently deletes a project. |

## Related Documents

| Document | Description |
|----------|-------------|
| [Folder Structures](FOLDER_STRUCTURES.md) | Complete filesystem layout reference |
| [Configuration](CONFIGURATION.md) | Workspace and project configuration schema |
| [Knowledge System](KNOWLEDGE_SYSTEM.md) | Main KB, Global KB architecture |
| [Backend](BACKEND.md) | Workspace process architecture |
| [Security](SECURITY.md) | Workspace isolation and encryption |
