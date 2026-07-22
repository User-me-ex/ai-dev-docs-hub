# ADR-0004: Role Override Cascading

## Status

Accepted

## Context

The [Nine Router](./NINE_ROUTER.md) allows role-to-model assignments to be overridden at three scopes: workspace, project, and group. The question is whether group overrides should **cascade down** to child groups or be **flat** (each group has its own override, ignoring parent groups).

Example: a workspace assigns `gpt-4o` to the Builder role. Project `web-app` overrides to `claude-sonnet-4`. Group `frontend` under `web-app` wants `gpt-4o` again — should this be allowed or should the project override be absolute?

## Decision

**Flat assignment with explicit scope resolution. No cascading.**

1. Every `RoleAssignment` has exactly one `scope` and one `scope_id`.
2. When resolving the effective assignment for a task, the Router looks for assignments in this order:
   ```
   group-level → project-level → workspace-level
   ```
3. The first match wins. If a group-level assignment exists, it is used regardless of what its parent project or workspace assigns.
4. There is no implicit inheritance. If group `frontend` has no explicit assignment for Builder, the Router falls through to project `web-app`'s assignment, then to workspace default.
5. Scope is declared at assignment time: `router.assign("builder", "gpt-4o", scope="project", scope_id="web-app")`.

### Why flat?

- Simpler to reason about: "my group uses the model I assigned to my group" — no mental model of cascading.
- Simpler to implement: a single lookup against three keys, no tree traversal.
- Avoids surprises: changing a project-level override does not silently change every group under it.
- The fallback chain (group → project → workspace) provides a sensible default without cascading — groups that don't care inherit their project's default; groups that do care pin their own model.

## Consequences

**Positive:**
- Clear, predictable resolution algorithm
- No cascading side effects
- Simple implementation (3-key map lookup)
- Groups have full autonomy over their model assignments

**Negative:**
- Administrators managing many groups must set assignments per-group if they want to change a model across all groups — no "set once, applies everywhere" except at workspace level
- Groups that truly want to follow their project's default simply omit their own assignment

## Related

- [Nine Router](./NINE_ROUTER.md) — role assignment rules (Open Questions tracked here)
- [Model Routing Policy](./MODEL_ROUTING_POLICY.md) — how resolved bindings are used
