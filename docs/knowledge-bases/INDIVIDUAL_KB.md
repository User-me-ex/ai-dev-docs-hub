# Individual Knowledge Base

> Per-agent private knowledge — the agent's own notes, scratchpads, and reflections.

## Scope

- Working memory promoted to durable notes.
- Self-reflection outputs (see [Self Reflection](../SELF_REFLECTION.md)).
- Personal few-shot library the agent maintains for itself.
- Intermediate findings and partial artifact drafts.
- Task-specific tool call histories and observations.

## Access

- **Read/write**: the owning agent only.
- **Read**: the Kernel and Critic for review.
- **Scope filter**: `workspace = <workspace_id>`, `agent = <agent_id>`.

## Record Schema

```
workspace = <workspace_id>
project   = <project_id>
group_id  = <group_id>
agent     = <agent_id>
retention = "7d" (default) | "session" (conversation) | "30d" (summaries)
```

## Query Example

```typescript
memory.query({
  text: "what I learned about the test suite",
  scope: { workspace: "my-workspace", agent: "worker-abc123" },
  kinds: ["summary", "fact"],
  k: 5
})
```

## Promotion Path

Individual KB entries can be promoted to higher tiers by the agent or Kernel:

1. Agent writes note to Individual KB (retention: session/7d).
2. Agent or Kernel identifies high-value entry.
3. Entry is summarized and re-written to Group KB or Main KB with longer retention.
4. Original Individual entry expires per its retention policy.

## Related Documents

- [Global KB](./GLOBAL_KB.md) — cross-workspace knowledge
- [Main KB](./MAIN_KB.md) — project-wide knowledge
- [Group KB](./GROUP_KB.md) — group-specific knowledge
- [Knowledge System](../KNOWLEDGE_SYSTEM.md)
- [Agent Memory](../AGENT_MEMORY.md)
- [Persistent Memory](../PERSISTENT_MEMORY.md)
