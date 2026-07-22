# ADR-0004: Role Override Cascading

## Status

Accepted

## Context

The [Nine Router](../NINE_ROUTER.md) allows role-to-model assignments to be overridden at three scopes: workspace, project, and group. The question is whether group overrides should **cascade down** to child groups or be **flat** (each group has its own override, ignoring parent groups).

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

### Scope Resolution Algorithm

```
Algorithm: ResolveEffectiveAssignment
Input:
  - role: string                  // e.g. "builder"
  - group_path: string[]          // e.g. ["web-app", "frontend"]
  - workspace_id: string
Output: AssignedModel | null

01  // Step 1: Try group-level (most specific)
02  for depth = group_path.length - 1 down to 0:
03      current_group = group_path[depth]
04      assignment = lookupAssignment(role, "group", current_group)
05      if assignment != null:
06          log("Resolved: group={current_group}, role={role}, model={assignment.model}")
07          return assignment.model
08
09  // Step 2: Try project-level
10  if group_path.length > 0:
11      project_id = group_path[0]  // Root group is the project
12      assignment = lookupAssignment(role, "project", project_id)
13      if assignment != null:
14          log("Resolved: project={project_id}, role={role}, model={assignment.model}")
15          return assignment.model
16
17  // Step 3: Try workspace-level default
18  assignment = lookupAssignment(role, "workspace", workspace_id)
19  if assignment != null:
20      log("Resolved: workspace={workspace_id}, role={role}, model={assignment.model}")
21      return assignment.model
22
23  // Step 4: No assignment found — return null (Router applies global default)
24  log("No assignment found for role={role}, using global default")
25  return null


// Helper: lookupAssignment
Algorithm: lookupAssignment(role, scope, scope_id)
01  key = (role, scope, scope_id)
02  if assignments.containsKey(key):
03      return assignments[key]
04  return null
```

**Complexity**: O(d) where d = group hierarchy depth. Typical d <= 3. With hash map lookup, each step is O(1). Overall: O(d) ≤ O(3) — constant time in practice.

### Conflict Resolution Strategy

Since assignments are flat (no cascading), conflicts only arise when the same `(role, scope, scope_id)` key is assigned multiple times:

```
Conflict: Two operators assign different models to the same (role, scope, scope_id)

Resolution: Last-writer-wins with operator-level escalation

Algorithm: WriteRoleAssignment

01  existing = lookupAssignment(role, scope, scope_id)
02  if existing != null:
03      if existing.operator_rank >= new_assignment.operator_rank:
04          // Same or higher rank: reject with explanation
05          reject("Conflicting assignment for ({role}, {scope}, {scope_id}).
06                  Existing: {existing.model} by {existing.operator}.
07                  To override, an operator of rank > {existing.operator_rank} must confirm.")
08          return
09      else:
10          // Higher rank operator: allow override, log conflict
11          log("Assignment overridden: ({role}, {scope}, {scope_id}),
12              was={existing.model}, now={new_assignment.model},
13              by={new_assignment.operator}")
14
15  assignments[(role, scope, scope_id)] = new_assignment

Operator ranks: admin (3) > maintainer (2) > operator (1)
```

### Override Precedence Examples

| Scenario | Workspace Assignment | Project Assignment | Group Assignment | Effective Model |
|----------|---------------------|-------------------|------------------|-----------------|
| Group has explicit override | `gpt-4o` (Builder) | `claude-sonnet-4` (Builder) | `gpt-4o` (Builder, frontend) | `gpt-4o` |
| Group has no override | `gpt-4o` (Builder) | `claude-sonnet-4` (Builder) | — (frontend) | `claude-sonnet-4` |
| No project or group override | `gpt-4o` (Builder) | — | — (frontend) | `gpt-4o` |
| No assignments at all | — | — | — | Global default (Router config) |
| Two groups, both with overrides | `gpt-4o` (Builder) | `claude-sonnet-4` (Builder) | `gpt-4o` (frontend), `claude-opus-4` (backend) | frontend: `gpt-4o`, backend: `claude-opus-4` |
| Group override for different role | `gpt-4o` (Builder) | — | `claude-sonnet-4` (Reviewer, frontend) | Builder: `gpt-4o`, Reviewer: `claude-sonnet-4` |

### Group → Project → Workspace Fallback Examples

```
Example 1: Three-level hierarchy
Groups:   "web-app/frontend"  →  "web-app"  →  "my-workspace"
Builder:  [no assignment]     →  claude-4    →  gpt-4o
Result:   claude-4 (inherited from project)

Example 2: Two-level, workspace only
Groups:   "web-app/frontend"  →  "web-app"     →  "my-workspace"
Builder:  [no assignment]     →  [no assignment] →  gpt-4o
Result:   gpt-4o (inherited from workspace)

Example 3: All three levels have assignments
Groups:   "web-app/frontend"  →  "web-app"     →  "my-workspace"
Builder:  o1-mini             →  claude-4        →  gpt-4o
Result:   o1-mini (group-level wins, no cascading)

Example 4: Nested groups, no project override
Groups:   "web-app/frontend/ui-kit"  →  "web-app/frontend"  →  "web-app"  →  "my-workspace"
Builder:  [no assignment]            →  [no assignment]     →  [no assignment]  →  gpt-4o
Result:   gpt-4o (walks up until a match is found: workspace level)
```

### Migration Guide from Previous Model

If migrating from a cascading model (where group assignments propagated to child groups):

```
Step 1: Audit existing assignments
  For each existing cascade-based assignment:
    - List all groups that were implicitly inheriting the assignment
    - These groups now need explicit assignments if they should keep the same model

Step 2: Create explicit assignments
  migration-tool --resolve-cascades
  This tool:
    - Reads all current assignments
    - For each group that had an implicit (cascaded) assignment:
      - Creates an explicit assignment at the group scope with the same model
    - Logs all new assignments for operator review

Step 3: Operator review
  - Review the generated assignments
  - Remove any that are no longer desired (groups that should fall through)
  - Approve the migration batch

Step 4: Enable flat resolution
  - Toggle the resolution algorithm from "cascade" to "flat"
  - All assignments now use explicit scope resolution
  - Old cascade logic is removed

Step 5: Verify
  - Run `router verify --role builder --group web-app/frontend`
  - Confirm the resolved model matches expectations
  - Repeat for all roles and groups

Rollback:
  - Keep cascade logic for one release cycle
  - Rollback by toggling back to "cascade" mode
  - Old cascade-based assignments are still valid in cascade mode
```

### Migration Complexity

| Metric | Value |
|--------|-------|
| Groups affected (typical workspace) | 10–50 |
| Assignments to create (estimated) | 15–75 |
| Migration tool execution time | < 5s |
| Rollback window | 1 release cycle |
| Operator review time | 15–30 minutes |

## Why flat?

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

- [Nine Router](../NINE_ROUTER.md) — role assignment rules (Open Questions tracked here)
- [Model Routing Policy](../MODEL_ROUTING_POLICY.md) — how resolved bindings are used
