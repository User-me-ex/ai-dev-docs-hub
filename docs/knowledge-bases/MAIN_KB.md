# Main Knowledge Base

> Workspace-level knowledge shared by every project inside a workspace.

## Scope

- Organizational conventions, brand voice, preferred libraries.
- Shared architectural decisions (ADRs promoted from projects).
- Workspace-wide secrets metadata (never the secret values themselves).
- Project-specific model routing overrides.
- Codebase patterns and idioms used across the workspace.

## Access

- **Read**: all agents in the workspace.
- **Write**: Kernel, Guardian, workspace maintainers.
- **Scope filter**: `workspace = <workspace_id>`, `project = <project_id>`.

## Record Schema

```
workspace = <workspace_id>
project   = <project_id>
group_id  = null
retention = "forever" (decisions) | "90d" (research results)
```

## Query Example

```typescript
memory.query({
  text: "architecture decision for database choice",
  scope: { workspace: "my-workspace", project: "my-project" },
  kinds: ["decision"],
  k: 3
})
```

## Related Documents

- [Global KB](./GLOBAL_KB.md) — cross-workspace knowledge
- [Group KB](./GROUP_KB.md) — group-specific knowledge
- [Individual KB](./INDIVIDUAL_KB.md) — per-agent knowledge
- [Knowledge System](../KNOWLEDGE_SYSTEM.md)
- [Persistent Memory](../PERSISTENT_MEMORY.md)
