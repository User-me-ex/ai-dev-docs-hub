# Global Knowledge Base

> Cross-workspace knowledge shared by every project and every agent on this installation.

## Scope

- Language references, framework docs, standards (RFCs, W3C).
- Model provider docs and capability tables.
- Universal coding rules and safety policies.
- System-wide invariants enforced by the Architecture Guardian.
- Provider registry (available model providers and their configuration).

## Access

- **Read**: all agents across all workspaces.
- **Write**: operators only (via Kernel or direct KB API).
- **Scope filter**: `workspace = "_global"` in memory queries.

## Record Schema

KB entries in this tier follow the standard `MemoryRecord` schema from [Persistent Memory](../PERSISTENT_MEMORY.md) with:

```
workspace = "_global"
project   = null
group_id  = null
retention = "forever"
```

## Update Cadence

- Refreshed by the [Research Engine](../RESEARCH_ENGINE.md) on a scheduled job (default: daily).
- Manual entries reviewed under [Prompt Governance](../PROMPT_GOVERNANCE.md).
- Changes are version-controlled alongside the documentation repo.

## Query Example

```typescript
// Retrieve from Global KB
memory.query({
  text: "TypeScript coding standards",
  scope: { workspace: "_global" },
  kinds: ["kb_entry"],
  k: 5
})
```

## Related Documents

- [Main KB](./MAIN_KB.md) — project-wide knowledge
- [Group KB](./GROUP_KB.md) — group-specific knowledge
- [Individual KB](./INDIVIDUAL_KB.md) — per-agent knowledge
- [Knowledge System](../KNOWLEDGE_SYSTEM.md)
- [Persistent Memory](../PERSISTENT_MEMORY.md)
- [Research Engine](../RESEARCH_ENGINE.md)
