# Migration Guide

> Upgrade paths between AI Dev OS versions — steps for data, prompt, and config migrations.

## Overview

This guide covers migrating between major and minor versions. Breaking changes are documented with migration steps, rationale, and rollback procedures.

## Version Compatibility Policy

AI Dev OS maintains **N-2 backward compatibility**:

- **v1.x** releases are backward-compatible with v1.0 data and config formats.
- **v2.0** supports migration from v1.x (N-1) and v1.0 (N-2).
- Patch releases never introduce breaking changes.
- Minor releases maintain backward compatibility within the same major version.

Always upgrade through intermediate versions if skipping more than two major versions.

## Checking Current Version

```bash
aidevos --version
# v1.0.0 (commit abc1234, built 2025-11-15)

aidevos doctor --verbose | grep "Schema version"
# Schema version: 3
```

## Migration Steps

### From Pre-v1 Snapshots to v1.0

v1.0 introduces SQLite-backed data, structured prompt versioning, and TOML config format.

**Before migrating:**

1. Back up `~/.aidevos/`: `cp -r ~/.aidevos ~/.aidevos.backup.$(date +%Y%m%d)`
2. Note your current version: `aidevos --version`
3. Review the changelog.

**Migration command:**

```bash
aidevos migrate --from=snapshot --to=v1
aidevos doctor
```

### Data Migration (SQLite Schema)

The database at `~/.aidevos/data/aidevos.db` uses versioned schemas. Migrations run automatically on `aidevos init` or `aidevos server start`.

| Schema Version | Changes | Auto-migrate |
|----------------|---------|--------------|
| 1 (pre-v1) | Flat file storage | Manual (`aidevos migrate`) |
| 2 (v1.0.0) | Initial SQLite schema | — |
| 3 (v1.1.0) | Added `vector_store` table | Automatic |

Check schema version: `sqlite3 ~/.aidevos/data/aidevos.db "PRAGMA user_version;"`

### Prompt Migration

```bash
aidevos migrate --check-prompts   # Check versions
aidevos migrate --prompts         # Upgrade to latest format
aidevos migrate --prompts --reset # Re-install defaults (overwrites customizations)
```

Custom prompts are preserved during upgrade unless `--reset` is passed.

### Config Migration

```bash
aidevos migrate --config --dry-run    # Preview changes
aidevos migrate --config              # Apply migration
```

Key field changes in v1.0 from pre-v1:

| Pre-v1 field | v1.0 field | Notes |
|--------------|------------|-------|
| `ollama.endpoint` | `providers.ollama.endpoint` | Moved under provider namespace |
| `default_model` | `router.default_model` | Moved to router section |
| `log_level` | `logging.level` | Moved to logging section |
| `jwt_secret` | `auth.jwt_secret` | Moved to auth section |

## Rollback Procedure

1. **Stop the server**: `aidevos server stop`
2. **Restore the backup**:
   ```bash
   rm -rf ~/.aidevos
   cp -r ~/.aidevos.backup.20251115 ~/.aidevos
   ```
3. **Install the previous binary** (see [Installation](./INSTALLATION.md)).
4. **Verify rollback**: `aidevos doctor`
5. **Downgrade schema** (if needed): `aidevos migrate --rollback --to=<previous_version>`

Rollbacks are supported within N-2 compatibility range. Restore from backup for further rollbacks.

## Related Documents

- [Versioning](./VERSIONING.md) — versioning scheme and semantics
- [Release Process](./RELEASE_PROCESS.md) — how releases are cut and published
- [Upgrade Notes](./UPGRADE_NOTES.md) — per-version upgrade notes
- [Changelog](./CHANGELOG.md) — full release history
- [Database](./DATABASE.md) — SQLite schema reference
- [Configuration](./CONFIGURATION.md) — config file format reference
