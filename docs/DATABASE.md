# Database

> Schema, migration strategy, indexing rationale, and data-access patterns for the SQLite (and optional Postgres) storage layer. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

AI Dev OS uses SQLite as its default, zero-configuration relational database. All metadata — runs, tasks, agents, model assignments, memory records, audit events, research jobs, plugin consents — lives here. A Postgres alternative is supported for multi-user remote deployments, with schema parity guaranteed.

The database is never accessed directly by application logic. All reads and writes go through the service layer ([Backend](./BACKEND.md), [Persistent Memory](./PERSISTENT_MEMORY.md), [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md)) which are the only components allowed to execute SQL. This keeps the data-access layer auditable and schema changes manageable.

## Goals

- Zero-config on fresh install: SQLite file created automatically at `~/.aidevos/db.sqlite`.
- Schema migrations: every schema change has a numbered migration file; migrations run automatically on startup.
- Postgres parity: all migrations and queries must be compatible with both SQLite and Postgres (no SQLite-only pragmas in shared code).
- Append-only event tables: `sce_events` and `audit_log` are append-only; no UPDATE or DELETE allowed.
- Encryption at rest: entire database encrypted via SQLCipher (SQLite) or column-level encryption (Postgres).

## Non-Goals

- Full-text search as a primary data access pattern — that is the [Vector Store](./VECTOR_STORE.md) and FTS5 virtual tables.
- Large blob storage — objects > 10 MB go to the object store; the DB stores a pointer.
- Implementation code — this repository is documentation-only (see [AI Coding Rules](./AI_CODING_RULES.md)).

## Schema

### `runs`

Kernel run records.

```sql
CREATE TABLE runs (
  id             TEXT PRIMARY KEY,     -- ULID
  goal           TEXT NOT NULL,        -- JSON Goal object
  state          TEXT NOT NULL,        -- intake|planning|routing|executing|critiquing|merging|guarding|delivered|failed|cancelled
  plan           TEXT,                 -- JSON TaskGraph (null until planning complete)
  tasks          TEXT,                 -- JSON Task[] (null until planning complete)
  verdicts       TEXT,                 -- JSON Verdict[]
  artifacts      TEXT,                 -- JSON Artifact[]
  budget         TEXT NOT NULL,        -- JSON { tokens_max, wall_ms_max, usd_max }
  spent          TEXT NOT NULL,        -- JSON { tokens, wall_ms, usd }
  correlation_id TEXT NOT NULL,
  created_at     TEXT NOT NULL,        -- RFC 3339
  updated_at     TEXT NOT NULL,
  terminated_at  TEXT                  -- RFC 3339 | NULL
);

CREATE INDEX idx_runs_state ON runs(state);
CREATE INDEX idx_runs_correlation ON runs(correlation_id);
CREATE INDEX idx_runs_created ON runs(created_at DESC);
```

### `tasks`

Individual task nodes within a run.

```sql
CREATE TABLE tasks (
  id             TEXT PRIMARY KEY,
  run_id         TEXT NOT NULL REFERENCES runs(id),
  parent_id      TEXT REFERENCES tasks(id),     -- NULL for root tasks
  kind           TEXT NOT NULL,
  description    TEXT NOT NULL,
  state          TEXT NOT NULL,                 -- pending|assigned|executing|completed|failed|cancelled
  group_id       TEXT,
  worker_id      TEXT,
  model_id       TEXT,
  artifact_id    TEXT,
  verdict        TEXT,                          -- JSON Verdict | NULL
  depends_on     TEXT,                          -- JSON ulid[] of dependency task IDs
  budget_slice   TEXT,                          -- JSON budget fraction allocated to this task
  correlation_id TEXT NOT NULL,
  created_at     TEXT NOT NULL,
  updated_at     TEXT NOT NULL
);

CREATE INDEX idx_tasks_run ON tasks(run_id, state);
CREATE INDEX idx_tasks_worker ON tasks(worker_id) WHERE worker_id IS NOT NULL;
```

### `sce_events`

Append-only SCE event log.

```sql
CREATE TABLE sce_events (
  id             TEXT PRIMARY KEY,        -- ULID (monotonic per broker)
  topic          TEXT NOT NULL,
  ts             TEXT NOT NULL,           -- RFC 3339 (broker-side monotonic)
  original_ts    TEXT,                    -- client-provided ts (may differ due to clock skew)
  actor_id       TEXT,
  actor_role     TEXT,
  correlation_id TEXT,
  causation_id   TEXT,
  schema_name    TEXT,
  schema_version TEXT,
  payload        BLOB NOT NULL,           -- JSON or encrypted JSON
  sig            TEXT                     -- base64 ed25519 signature
);

-- NOTE: NO UPDATE or DELETE on this table — append only.
CREATE INDEX idx_sce_topic_id ON sce_events(topic, id);       -- primary traversal
CREATE INDEX idx_sce_correlation ON sce_events(correlation_id);
CREATE INDEX idx_sce_ts ON sce_events(ts);
```

### `sce_cursors`

Durable subscriber cursors for `ctx.ack()`.

```sql
CREATE TABLE sce_cursors (
  subscriber_id  TEXT NOT NULL,
  topic          TEXT NOT NULL,
  last_event_id  TEXT NOT NULL,
  updated_at     TEXT NOT NULL,
  PRIMARY KEY (subscriber_id, topic)
);
```

### `memory_records`

[Persistent Memory](./PERSISTENT_MEMORY.md) relational tier.

```sql
CREATE TABLE memory_records (
  id             TEXT PRIMARY KEY,
  workspace      TEXT NOT NULL,
  project        TEXT,
  group_id       TEXT,
  agent          TEXT,
  kind           TEXT NOT NULL,
  key            TEXT,                    -- stable key for upsert
  text           TEXT NOT NULL,
  tags           TEXT,                    -- JSON string[]
  refs           TEXT,                    -- JSON { kind, id }[]
  source_type    TEXT,
  source_id      TEXT,
  source_run_id  TEXT,
  retention      TEXT NOT NULL,
  encrypted      INTEGER NOT NULL DEFAULT 0,
  created_at     TEXT NOT NULL,
  updated_at     TEXT NOT NULL,
  expires_at     TEXT
);

CREATE INDEX idx_memory_workspace_kind ON memory_records(workspace, kind);
CREATE INDEX idx_memory_key ON memory_records(workspace, key) WHERE key IS NOT NULL;
CREATE INDEX idx_memory_expires ON memory_records(expires_at) WHERE expires_at IS NOT NULL;
CREATE INDEX idx_memory_tags ON memory_records(workspace, tags);  -- FTS5 virtual table alternative

-- Full-text search virtual table
CREATE VIRTUAL TABLE memory_fts USING fts5(text, content='memory_records', content_rowid='rowid');
```

### `role_assignments`

[Nine Router](./NINE_ROUTER.md) role assignments.

```sql
CREATE TABLE role_assignments (
  id             TEXT PRIMARY KEY,
  role           TEXT NOT NULL,           -- NineRole
  model_id       TEXT NOT NULL,
  fallbacks      TEXT,                    -- JSON string[]
  scope          TEXT NOT NULL,           -- workspace|project|group
  scope_id       TEXT,                    -- project_id or group_id
  actor_id       TEXT,
  created_at     TEXT NOT NULL,
  superseded_at  TEXT                     -- NULL = currently active
);

-- Get current assignment for a role+scope:
-- SELECT * FROM role_assignments WHERE role=? AND scope=? AND scope_id IS ? AND superseded_at IS NULL
CREATE INDEX idx_role_active ON role_assignments(role, scope, scope_id) WHERE superseded_at IS NULL;
```

### `model_cache`

Cached [Model Discovery](./MODEL_DISCOVERY.md) results.

```sql
CREATE TABLE model_cache (
  id                TEXT PRIMARY KEY,    -- canonical "provider/model-id"
  provider          TEXT NOT NULL,
  provider_model_id TEXT NOT NULL,
  aliases           TEXT,               -- JSON string[]
  display_name      TEXT NOT NULL,
  family            TEXT,
  context_window    INTEGER,
  max_output_tokens INTEGER,
  modalities        TEXT,               -- JSON string[]
  capabilities      TEXT,               -- JSON string[]
  pricing           TEXT,               -- JSON | NULL
  deprecated        INTEGER NOT NULL DEFAULT 0,
  deprecation_date  TEXT,
  status            TEXT NOT NULL,
  discovered_at     TEXT NOT NULL,
  source            TEXT NOT NULL
);

CREATE INDEX idx_model_provider ON model_cache(provider, deprecated);
```

### `audit_log`

Append-only audit records (see [Audit Log](./AUDIT_LOG.md)).

```sql
CREATE TABLE audit_log (
  id             TEXT PRIMARY KEY,
  ts             TEXT NOT NULL,
  actor_id       TEXT,
  actor_role     TEXT,
  action         TEXT NOT NULL,
  resource_type  TEXT,
  resource_id    TEXT,
  outcome        TEXT NOT NULL,         -- ok|denied|error
  detail         TEXT,                  -- JSON
  correlation_id TEXT,
  ip_address     TEXT
);

-- Append only: no UPDATE or DELETE.
CREATE INDEX idx_audit_ts ON audit_log(ts DESC);
CREATE INDEX idx_audit_actor ON audit_log(actor_id, ts DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

### `research_jobs`

[Research Engine](./RESEARCH_ENGINE.md) scheduled jobs.

```sql
CREATE TABLE research_jobs (
  id          TEXT PRIMARY KEY,
  source      TEXT NOT NULL,            -- JSON ResearchJobSource
  cadence     TEXT,                     -- cron expression | NULL (one-shot)
  kb_target   TEXT NOT NULL,
  kb_scope    TEXT NOT NULL,            -- JSON { workspace, project?, group? }
  tags        TEXT,                     -- JSON string[]
  state       TEXT NOT NULL,
  last_run    TEXT,
  next_run    TEXT,
  created_by  TEXT NOT NULL,            -- JSON { id, kind }
  created_at  TEXT NOT NULL,
  updated_at  TEXT NOT NULL
);

CREATE INDEX idx_research_state ON research_jobs(state, next_run);
```

### `plugin_consents`

Plugin capability consents (see [Plugin SDK](./PLUGIN_SDK.md)).

```sql
CREATE TABLE plugin_consents (
  plugin_id      TEXT NOT NULL,
  capability     TEXT NOT NULL,
  granted        INTEGER NOT NULL,      -- 0 | 1
  granted_by     TEXT,
  granted_at     TEXT,
  manifest_version TEXT,
  PRIMARY KEY (plugin_id, capability)
);
```

## Migrations

Migrations are numbered SQL files in `migrations/`. Every migration file MUST:
- Be named `NNNN_<description>.sql` where `NNNN` is a zero-padded 4-digit sequence number.
- Be idempotent (`CREATE TABLE IF NOT EXISTS`, `CREATE INDEX IF NOT EXISTS`).
- Never modify `sce_events` or `audit_log` rows (these tables are append-only forever).
- Include a comment block at the top: `-- Migration NNNN: description | author | date`.

The server applies pending migrations automatically at startup using a `schema_migrations` table:

```sql
CREATE TABLE schema_migrations (
  version    INTEGER PRIMARY KEY,
  applied_at TEXT NOT NULL
);
```

Rollback is not supported: every migration must be forward-only. Breaking changes require a new additive migration; data backfills run as scheduled jobs, not in the migration itself.

## Requirements

- **MUST** use parameterised queries for all SQL; string concatenation into SQL is prohibited (enforced by the `no-sql-injection` Guardian rule).
- **MUST** run in WAL mode (`PRAGMA journal_mode=WAL`) on SQLite.
- **MUST** never UPDATE or DELETE rows in `sce_events` or `audit_log`.
- **MUST** apply migrations idempotently at startup before serving any request.
- **MUST** encrypt the SQLite database at rest using SQLCipher with a key from [Secrets Management](./SECRETS_MANAGEMENT.md).
- **MUST** keep schema parity between SQLite and Postgres targets.
- **SHOULD** expose a `db.stats()` endpoint reporting WAL size, page count, and index health.
- **MAY** run `PRAGMA optimize` (SQLite) or `ANALYZE` (Postgres) nightly via the [Job Scheduler](./JOB_SCHEDULER.md).

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| Database file missing on startup | `SQLITE_CANTOPEN` | Create new database; run all migrations; log `db.created` |
| Schema migration failure | Migration SQL error | Halt startup; log which migration failed; do not partially apply |
| Write failure during run | `SQLITE_FULL` / I/O error | Buffer to WAL; emit `db.write_error`; block new runs; alert |
| Database corruption | `PRAGMA integrity_check` fails | Halt; restore from backup; see [Disaster Recovery](./DISASTER_RECOVERY.md) |
| FTS index out of sync | FTS5 triggers fail | Rebuild FTS index via `INSERT INTO memory_fts(memory_fts) VALUES('rebuild')` |

## Observability

| Metric | Description |
|--------|-------------|
| `db_query_total{table,op}` | Query count by table and operation |
| `db_query_seconds{table,op}` | Query latency histogram |
| `db_wal_pages` | SQLite WAL page count |
| `db_migration_version` | Current applied migration version |
| `db_table_rows{table}` | Row count per table (sampled, not exact) |

## Acceptance Criteria

- `aidevos-server` starts and auto-creates the SQLite database with all migrations applied on a fresh machine in < 2 s.
- `PRAGMA integrity_check` returns `ok` after 10,000 concurrent writes from a stress test.
- A new migration file causes the migration to apply automatically on the next server start without manual intervention.
- Decrypting the database file without the correct key returns `SQLITE_NOTADB`.
- Query `SELECT * FROM memory_records WHERE workspace=? AND kind=?` with 100k rows completes in < 10 ms with the index in place.

## Open Questions

- Whether to vendor SQLCipher into the binary or depend on the system-installed version — tracked in [templates/ADR](../templates/ADR.md).
- Schema versioning strategy for the Postgres target: Flyway vs. Liquibase vs. custom.

## Related Documents

- [Backend](./BACKEND.md)
- [Persistent Memory](./PERSISTENT_MEMORY.md)
- [Shared Context Engine](./SHARED_CONTEXT_ENGINE.md)
- [Audit Log](./AUDIT_LOG.md)
- [Encryption](./ENCRYPTION.md)
- [Disaster Recovery](./DISASTER_RECOVERY.md)
- [Backup Strategy](./BACKUP_STRATEGY.md)
- [System Overview](./SYSTEM_OVERVIEW.md)
- [Main AI Kernel](./MAIN_AI_KERNEL.md)
