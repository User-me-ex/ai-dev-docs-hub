# Upgrade Notes

## Overview

This document contains version-specific upgrade instructions for AI Dev OS. Always consult the relevant notes before upgrading between major or minor versions.

If you are skipping versions, apply each upgrade step sequentially — do not jump directly to the latest version.

---

## Upgrade Checking

```bash
# Check current version and available upgrades
aidevos version --check-upgrade

# Run pre-upgrade diagnostics
aidevos doctor
```

`aidevos doctor` validates your environment, checks for breaking changes, and reports any configuration or data migrations required before upgrading.

---

## v0.1.0 → v0.2.0

*Notes will be added here when v0.2.0 is released.*

---

## Pre-v1 → v1.0 Migration

### Python to Rust Runtime

v1.0 replaces the Python runtime with a Rust native binary. The `aidevos` CLI is now a single statically-linked executable.

**Steps:**

1. Uninstall the Python package: `pip uninstall aidevos`
2. Download the v1.0 binary for your platform from the releases page
3. Verify the binary: `aidevos version`
4. Run migration tool: `aidevos migrate pre-v1-to-v1`

### SQLite Schema Migration

The internal SQLite database schema changed between pre-v1 and v1.0. The migration tool handles this automatically, but manual verification is recommended:

```bash
aidevos migrate pre-v1-to-v1 --dry-run  # preview changes
aidevos migrate pre-v1-to-v1            # apply migration
```

Backup your database before migrating:

```bash
cp ~/.local/share/aidevos/aidevos.db ~/.local/share/aidevos/aidevos.db.backup
```

### Prompt Format Changes

Custom prompt templates (`.aidevos/prompts/`) must be updated:

- `{{ variable }}` syntax changed to `{variable}`
- Tool call format changed from XML tags to JSON blocks
- System prompt sections are now order-independent

Run `aidevos doctor` to detect and report any incompatible prompt files.

### Config File Changes

| Pre-v1 | v1.0 | Notes |
|---|---|---|
| `[agent]` | `[runtime]` | Renamed |
| `memory.backend` | `[memory] backend` | Restructured |
| `logging.level` | `[log] level` | Restructured |
| `provider.api_key` | `[auth] credentials_file` | Moved — keys no longer stored in config |

The migration tool (`aidevos migrate`) automatically rewrites your config file. The original is saved as `config.toml.pre-v1.backup`.

---

## Rollback Instructions

If an upgrade fails or introduces regressions:

1. Restore the previous binary: keep the previous release tarball or use a package manager rollback.
2. Restore database: `cp aidevos.db.backup aidevos.db`
3. Restore config: `cp config.toml.pre-v1.backup config.toml`
4. Verify: `aidevos doctor`

Downgrading across a database schema change may require manual intervention. Contact support if the rollback path is unclear.

---

## Verifying Successful Upgrade

```bash
aidevos version        # confirm expected version
aidevos doctor         # check all systems green
aidevos run --help     # smoke test the CLI
```

Run a minimal test task to confirm the agent runtime works:

```bash
echo "say hello" | aidevos run --stdin
```

---

## Related Documents

- [Migration Guide](./MIGRATION_GUIDE.md)
- [Versioning Policy](./VERSIONING.md)
- [Release Process](./RELEASE_PROCESS.md)
- [Changelog](./CHANGELOG.md)
