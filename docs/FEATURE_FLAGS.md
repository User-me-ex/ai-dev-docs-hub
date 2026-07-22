# Feature Flags

> Feature flag system for AI Dev OS — definition, evaluation, storage, lifecycle, and rollout management. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

The feature flag system gates in-development, experimental, and beta functionality behind named flags. Flags are evaluated at compile time for binary-level features (build tags) and at runtime for dynamic features (service behavior). Every flag has an owner, an optional expiration, and defaults to **disabled** for safety.

Flags are stored in a TOML file locally and synced via the SCE for cluster-wide consistency. The system exposes a watch API so subsystems can react to flag changes without restarting.

## FeatureFlag Schema

```toml
[flags.experimental_agent_routing]
description = "Enable latency-optimized agent-to-provider routing"
enabled = false
group = "experimental"
expires_at = "2026-10-01"
owner = "team-core"

[flags.beta_workspace_templates]
description = "Enable workspace creation from templates"
enabled = false
group = "beta"
owner = "team-platform"
expires_at = ""
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | implicit | Flag name (section key under `[flags]`) |
| `description` | string | yes | Human-readable description of what this flag controls |
| `enabled` | bool | yes | Whether the flag is currently active |
| `group` | enum | yes | `experimental`, `beta`, `internal`, `released` |
| `expires_at` | date | no | ISO 8601 date after which the flag MUST be removed or graduated |
| `owner` | string | yes | Team or individual responsible for the flag |

Flags with `expires_at` in the past are treated as **always disabled** and produce a warning on every evaluation. They MUST be removed or updated in the next release cycle.

## Flag Evaluation

### Compile-time flags (binary features)

Flags marked as `compile_time = true` are evaluated during build via Go build tags / Rust features:

```rust
#[cfg(feature = "experimental-agent-routing")]
fn route_agent() { /* new implementation */ }

#[cfg(not(feature = "experimental-agent-routing"))]
fn route_agent() { /* current implementation */ }
```

These flags are not configurable at runtime. They are set during build and baked into the binary.

### Runtime flags (dynamic)

Runtime flags are evaluated on every access through the `flags.isEnabled()` interface. The evaluation is a simple boolean check:

```python
if flags.is_enabled("beta_workspace_templates"):
    show_template_picker()
else:
    show_empty_workspace_form()
```

Runtime flags are cached for 10 s (configurable via `flags.cache_ttl_seconds`). Cache invalidation happens on SCE `flag.changed` events.

### Gradual rollout

Rollout is controlled via two mechanisms:

**Percentage-based:** A flag can specify a rollout percentage between 0 and 100:

```toml
[flags.new_memory_index]
rollout_percentage = 25
```

Evaluation becomes: `hash(workspace_id) % 100 < rollout_percentage`.

**Group-based:** A flag can target specific workspace groups, user tiers, or deployment environments:

```toml
[flags.beta_ui]
enabled_groups = ["staff", "early-access"]
```

Evaluation becomes: `current_group in enabled_groups`.

The final evaluation is: `enabled AND (percentage check OR group check)`. A flag with `enabled = false` is always disabled regardless of rollout settings.

## Flag Storage

### Local TOML file

```
~/.aidevos/flags.toml
```

Loaded at startup. Watched for changes via filesystem watcher (500 ms debounce). Changes are applied dynamically for runtime flags.

### Cluster sync via SCE

In multi-server mode, flag changes are published on the SCE topic `flag.{name}.changed` with the new flag state as payload. All nodes receive the event and update their local cache within 100 ms.

Flag state is stored in the SCE event log for auditability. The SCE compaction policy MUST retain flag change events for the full retention period (see [Data Retention](./DATA_RETENTION.md)).

## Built-in Flags

| Flag | Group | Default | Expires | Owner | Description |
|------|-------|---------|---------|-------|-------------|
| `experimental_agent_routing` | experimental | disabled | 2026-10-01 | team-core | Latency-optimized agent-to-provider routing |
| `experimental_graph_reasoning` | experimental | disabled | 2026-09-01 | team-research | Use knowledge graph for multi-hop reasoning |
| `beta_workspace_templates` | beta | disabled | 2026-12-01 | team-platform | Enable workspace creation from templates |
| `beta_mcp_tool_registry` | beta | disabled | 2026-11-01 | team-integrations | Dynamic MCP tool discovery via registry |
| `internal_detailed_traces` | internal | disabled | — | team-core | Emit verbose span attributes for debugging |
| `released_parallel_tool_calls` | released | enabled | — | team-core | Allow agents to invoke multiple tools concurrently |

Flags in `released` group with no expiration are considered permanent capabilities. They remain in the flag system for observability but should be treated as always-enabled.

## Flag Lifecycle

```
Propose → Evaluate → Ship → Remove
```

| Stage | Action | Artifacts |
|-------|--------|-----------|
| **Propose** | File a flag proposal in the feature tracking system. Define the flag name, description, group, rollout plan, and success criteria. | Flag proposal doc |
| **Evaluate** | Add the flag to `flags.toml` with `enabled = false`. Implement flag checks in code. No user-facing impact yet. | PR with flag definition + code changes |
| **Ship** | Enable for internal group → percentage rollout → full rollout. Monitor metrics, error rates, and SLOs. Communicate in changelog. | SCE events: `flag.enabled`, `flag.rollout_changed` |
| **Remove** | Remove flag checks from code. Remove flag definition from `flags.toml` or archive with `expires_at` = removal date. Ship the cleanup PR. | PR removing flag code; archived flag def |

A flag MUST NOT remain in `experimental` or `beta` group for longer than 6 months without review.

## Interfaces

### `flags.isEnabled(name: str) -> bool`

Returns `true` if the flag is enabled and the current context passes rollout checks.

```
flags.isEnabled("beta_workspace_templates") → false
flags.isEnabled("released_parallel_tool_calls") → true
flags.isEnabled("nonexistent_flag") → false
```

Unknown flags return `false` and log a warning at most once per flag per process lifetime.

### `flags.list() -> list[FeatureFlag]`

Returns all registered flags with their current state:

```
flags.list()
# [
#   FeatureFlag(name="experimental_agent_routing", enabled=false, group="experimental", ...),
#   FeatureFlag(name="released_parallel_tool_calls", enabled=true, group="released", ...),
# ]
```

### `flags.enable(name: str)`

Enables a flag for the current process. If the flag has `expires_at` in the past, this is a no-op with a warning.

### `flags.disable(name: str)`

Disables a flag for the current process. Emits `flag.disabled` on the SCE.

### `flags.watch(name_pattern: str) -> AsyncIterator[FlagChange]`

Returns an async iterator yielding `FlagChange(name, old_state, new_state)` for matching flags:

```python
async for change in flags.watch("experimental_*"):
    if change.new_state.enabled:
        enable_experimental_code_path(change.name)
```

## Failure Modes

| Failure | Detection | Response |
|---------|-----------|----------|
| Unknown flag checked | `isEnabled("nonexistent")` | Returns `false`; logs warning once per flag |
| Flags file unreadable | Startup: can't open `flags.toml` | All flags default to `disabled`; emit `flags.file_missing` |
| Flags file parse error | Startup: TOML parse error | All flags default to `disabled`; log error with line number |
| SCE sync failure | `flag.changed` event not received within 1000 ms | Use cached flag state; emit `flags.sync_lag` |
| Expired flag still in code | Startup: warns for each expired flag | Flag treated as always-disabled; owner auto-assigned to ticket |
| Rollout percentage > 100 | Validation on load | Clamped to 100; log warning |
| Flag name collision | Same name in two sections | Last definition wins; log warning |
| Cache stampede | Many parallel `isEnabled` calls on cache miss | Mutex per flag; compute once; broadcast to waiters |

## Flag Evaluation Flow Diagram

```mermaid
flowchart TB
    CHECK[flags.isEnabled(name)] --> EXISTS{Flag defined?}
    EXISTS -->|no| WARN[Log warning once]
    WARN --> RETURN[Return false]
    EXISTS -->|yes| EXPIRED{expires_at\nin the past?}
    EXPIRED -->|yes| WARN_EXPIRED[Log expired warning]
    WARN_EXPIRED --> DISABLED[Return false]
    EXPIRED -->|no| ENABLED{enabled = true?}
    ENABLED -->|no| DISABLED
    ENABLED -->|yes| ROLLOUT{Has rollout\npercentage?}
    ROLLOUT -->|no| GROUP_CHECK{Has enabled\ngroups?}
    ROLLOUT -->|yes| PERCENT{hash(id) % 100\n< rollout_pct?}
    PERCENT -->|yes| GROUP_CHECK
    PERCENT -->|no| DISABLED
    GROUP_CHECK -->|no groups| PASS[Return true]
    GROUP_CHECK -->|has groups| GROUPS{current group\nin enabled_groups?}
    GROUPS -->|yes| PASS
    GROUPS -->|no| DISABLED
```

## Flag Definition Schema (Full)

```toml
[flags.flag_name]
description = "Human-readable description"
enabled = false                              # Master switch
group = "experimental"                       # experimental | beta | internal | released
expires_at = "2026-10-01"                    # ISO 8601 or empty for permanent
owner = "team-name"                          # Responsible team
rollout_percentage = 0                       # 0-100, 0 means disabled for all
enabled_groups = []                          # ["staff", "early-access"]
compile_time = false                         # true = build-time feature gate
depends_on = []                              # ["other_flag_name"] - dependencies
metadata = {}                                # Arbitrary key-value metadata
```

## Targeting Rules

Targeting rules determine which users or workspaces see a flag:

| Rule Type | Configuration | Evaluation |
|-----------|---------------|------------|
| Percentage | `rollout_percentage = 25` | `hash(workspace_id) % 100 < 25` |
| Group | `enabled_groups = ["staff", "beta"]` | `current_group in enabled_groups` |
| Environment | Environment variable `AIDEVOS_ENV` | `env == "staging"` |
| Workspace ID | `target_workspaces = ["ws_abc"]` | `workspace_id in target_workspaces` |

Multiple rules are combined with OR: if ANY rule matches, the flag is enabled for that context.

## Gradual Rollout Algorithm

```
function isEnabled(flag, context):
    if not flag.enabled:
        return false
    if flag.expires_at and flag.expires_at < now():
        return false
    if flag.rollout_percentage > 0:
        hash = hashCode(context.workspace_id)
        if hash % 100 < flag.rollout_percentage:
            return true
    if flag.enabled_groups:
        if context.group in flag.enabled_groups:
            return true
    if flag.target_workspaces:
        if context.workspace_id in flag.target_workspaces:
            return true
    // If no targeting configured and enabled=true, return true
    if flag.rollout_percentage == 0 and not flag.enabled_groups and not flag.target_workspaces:
        return true
    return false
```

## A/B Test Configuration

Feature flags can be used for A/B testing by configuring split percentages:

```toml
[flags.new_ui_layout]
description = "New UI layout for workspace dashboard"
enabled = true
group = "experimental"
owner = "team-ux"
rollout_percentage = 50   # 50% of workspaces see the new layout
```

A/B test results are collected via the metrics pipeline:
- `flag_ab_exposed_total{flag_name, variant}` — counter of exposures
- `flag_ab_conversion_total{flag_name, variant, event}` — counter of conversion events

## Flag Lifecycle (Detailed)

| Stage | Duration | Actions | Artifacts |
|-------|----------|---------|-----------|
| **Propose** | 1 day | File proposal; define name, group, rollout plan, success criteria | Flag proposal doc |
| **Evaluate** | 1-2 sprints | Add flag to code; implement checks; test in dev | PR with flag def + code |
| **Ship** | Gradual (days-weeks) | Internal → 10% → 25% → 50% → 100% → GA | SCE events per rollout change |
| **Remove** | 1 sprint | Remove flag checks; archive flag def; ship cleanup PR | PR removing flag code |

## Kill Switch Procedure

When a flag causes production issues, the kill switch procedure is:

```
1. Identify the problematic flag (e.g., experimental_agent_routing)
2. Disable immediately:
   aidevos flags disable experimental_agent_routing
3. Verify the disable took effect:
   aidevos flags list | grep experimental_agent_routing
4. The SCE event flag.experimental_agent_routing.disabled is emitted
5. All nodes pick up the change within 100 ms (SCE sync)
6. Code paths revert to the previous behavior automatically
7. Root cause investigation begins; flag is not re-enabled until fixed
```

## Failure Modes (Expanded)

| Failure | Detection | Response |
|---------|-----------|----------|
| Unknown flag checked | `isEnabled("nonexistent")` | Returns `false`; logs warning once per flag |
| Flags file unreadable | Startup: can't open `flags.toml` | All flags default to `disabled`; emit `flags.file_missing` |
| Flags file parse error | Startup: TOML parse error | All flags default to `disabled`; log error with line number |
| SCE sync failure | `flag.changed` event not received within 1000 ms | Use cached flag state; emit `flags.sync_lag` |
| Expired flag still in code | Startup: warns for each expired flag | Flag treated as always-disabled; owner auto-assigned to ticket |
| Rollout percentage > 100 | Validation on load | Clamped to 100; log warning |
| Flag name collision | Same name in two sections | Last definition wins; log warning |
| Cache stampede | Many parallel `isEnabled` calls on cache miss | Mutex per flag; compute once; broadcast to waiters |
| Circular dependency | Flag A depends on B, B depends on A | Detection at load; break circle by treating both as disabled |

## Observability / Metrics

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `flag_evaluations_total` | Counter | flag_name, result (enabled/disabled) | Flag evaluation count |
| `flag_cache_hit_ratio` | Gauge | flag_name | Cache effectiveness |
| `flag_sync_lag_seconds` | Gauge | flag_name | SCE sync latency |
| `flag_expired_warnings_total` | Counter | flag_name | Expired flag warnings |

## Acceptance Criteria

- An unknown flag always returns `false` and logs a warning at most once.
- Disabling a flag via CLI takes effect on all cluster nodes within 100 ms.
- A flag with `enabled = false` is always disabled regardless of targeting rules.
- A flag with `expires_at` in the past is treated as disabled with a warning.
- Rollout percentage targeting correctly distributes across workspace IDs.
- Kill switch disable propagates within 100 ms and reverts behavior.

## Related Documents

- [Configuration](./CONFIGURATION.md) — config file format, dynamic reload, precedence
- [Release Process](./RELEASE_PROCESS.md) — flag lifecycle stages, changelog entries for flag changes
- [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md) — SCE topics for flag sync
- [CLI](./CLI.md) — `aidevos flags` subcommands for enable/disable/list
- [Observability](./OBSERVABILITY.md) — flag change events, metrics on flag evaluation
- [Deployment](./DEPLOYMENT.md) — flag file injection in containerized deployments
