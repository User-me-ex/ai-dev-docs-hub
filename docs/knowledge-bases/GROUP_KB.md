# Group Knowledge Base

> Per-AI-Group knowledge shared by every agent within a group.

## Scope

- Role-specific playbooks (e.g. the Builder group's refactor recipes).
- Shared prompt fragments and few-shot examples for the group.
- Group-scoped memory summarizations.
- Group-specific tool configurations and MCP server bindings.
- Task templates commonly used by the group.

## Access

- **Read**: all agents in the group.
- **Write**: group lead agent and the Kernel.
- **Scope filter**: `workspace = <workspace_id>`, `group = <group_id>`.

## Record Schema

```
workspace = <workspace_id>
project   = <project_id>
group_id  = <group_id>
retention = "90d" (default) | "30d" (research results)
```

## Query Example

```typescript
memory.query({
  text: "playbook for code review",
  scope: { workspace: "my-workspace", group: "code-reviewer" },
  kinds: ["playbook"],
  k: 5
})
```

## Related Documents

- [Global KB](./GLOBAL_KB.md) — cross-workspace knowledge
- [Main KB](./MAIN_KB.md) — project-wide knowledge
- [Individual KB](./INDIVIDUAL_KB.md) — per-agent knowledge
- [Knowledge System](../KNOWLEDGE_SYSTEM.md)
- [AI Groups](../AI_GROUPS.md)
- [Persistent Memory](../PERSISTENT_MEMORY.md)
