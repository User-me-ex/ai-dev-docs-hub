# AI Coding Rules

> Rules every AI agent in the AI Dev OS MUST follow when producing or modifying code in a user project.

## Core Rules

1. **Small, reversible steps.** Prefer many small diffs over one large rewrite.
2. **Read before write.** Never edit a file without first reading its current contents.
3. **Respect the guardian.** The [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) can veto a change; do not route around it.
4. **Documentation-first.** New subsystems land as a doc in `docs/` before code exists.
5. **Cite sources.** When applying an external pattern, cite the source in the commit/PR body.
6. **No hidden state.** All persistent state must go through the [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md) or an approved store.
7. **Deterministic tests first.** New behavior requires a failing test before the fix.
8. **Least privilege.** Request only the tools and scopes actually needed for the task.
9. **Ask before destructive actions.** Deletions, migrations, and force-pushes require explicit confirmation.
10. **Prefer standards.** OpenAPI, JSON Schema, Mermaid, Markdown, POSIX shell where possible.

## Style

- Match the surrounding code's style; do not reformat unrelated lines.
- Public APIs MUST have doc comments; private helpers SHOULD.
- Error messages MUST be actionable.

## Prohibited

- Generating application code in this documentation-only repository.
- Committing secrets or `.env` files.
- Bypassing the [Merge Manager](./MERGE_MANAGER.md) for concurrent edits.

## Related Documents

- [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md)
- [Merge Manager](./MERGE_MANAGER.md)
- [Impact Analysis](./IMPACT_ANALYSIS.md)
- [Prompt Governance](./PROMPT_GOVERNANCE.md)
