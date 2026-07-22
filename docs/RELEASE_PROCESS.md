# Release Process

> How AI Dev OS releases are cut, tested, and published. This document covers cadence, phases, build pipeline, artifacts, hotfixes, and LTS. It is the canonical reference for release engineers and CI automation.

## Overview

AI Dev OS follows a time-based release model with a monthly minor release cadence and on-demand patch releases. Every release goes through a defined sequence of phases: feature freeze, release candidate builds, and final release. The process is fully automated through GitHub Actions but requires manual sign-off at each phase gate.

All version numbers follow the scheme defined in [Versioning](./VERSIONING.md). A release is identified by its platform version tag (e.g., `v0.1.0`) and ships with a matching documentation archive.

## Release Cadence

| Release type | Frequency | Bump | Examples |
|-------------|-----------|------|----------|
| Minor | Monthly (first Tuesday) | MINOR | 0.1.0, 0.2.0, 0.3.0 |
| Patch | As needed (critical bugs only) | PATCH | 0.1.1, 0.1.2 |
| LTS | Every 6 months | MAJOR | 1.0.0, 2.0.0 |
| Pre-release | Before each minor | Pre-release tag | 0.1.0-rc.1 |

Patch releases skip the feature freeze phase. LTS releases follow the full process with an extended testing window (2 weeks of RC instead of 1).

## Release Phases

### Feature Freeze (T-7 days)

- The `main` branch is tagged with `v{next_version}-freeze`.
- Only bug fixes, documentation updates, and test additions are permitted on `main`.
- Feature branches for the next release continue development in parallel.
- The release engineer opens a milestone and moves all unresolved issues to the next milestone.

### Release Candidate (T-3 days)

- A release branch `release/v{MAJOR}.{MINOR}` is cut from `main`.
- The version in `package.json` (or equivalent) is bumped to the release candidate version.
- CI runs the full build matrix (see Build Pipeline below).
- Each failed RC produces a new candidate: `rc.1`, `rc.2`, etc.
- The Eval Harness is run against each RC. A passing score >= 90% is required.

### Final Release (T-0)

- The last passing RC is promoted to release.
- The release tag `v{MAJOR}.{MINOR}.{PATCH}` is applied.
- Release artifacts are published (see Release Artifacts below).
- The milestone is closed.
- The release notes are posted to the GitHub Releases page.

## Release Checklist

Every release MUST pass every item on this checklist before promotion:

- [ ] All acceptance criteria for the milestone pass in CI.
- [ ] Eval Harness reports >= 90% pass rate on the default prompt suite.
- [ ] [CHANGELOG](./CHANGELOG.md) updated with the new version entry.
- [ ] Platform version bumped in `package.json`, `Cargo.toml`, or equivalent.
- [ ] `MASTER_PROMPT` version bumped if prompts changed.
- [ ] `docs/README.md` version bumped if docs changed.
- [ ] Migration guide ([MIGRATION_GUIDE.md](./MIGRATION_GUIDE.md)) updated with any schema or API changes.
- [ ] Binaries built and signed for all target platforms.
- [ ] Container image built and pushed.
- [ ] Release notes drafted and reviewed.

## Build Pipeline

The build pipeline is defined in `.github/workflows/release.yml` and consists of three stages:

### Stage 1: Matrix Build

| Target | OS | Architecture | Signing |
|--------|----|-------------|---------|
| macOS Intel | macOS 12 | x86_64 | Apple notarization |
| macOS Apple Silicon | macOS 14 | ARM64 | Apple notarization |
| Windows | Windows Server 2022 | x86_64 | Authenticode |
| Linux | Ubuntu 22.04 | x86_64 | GPG |
| Linux ARM64 | Ubuntu 22.04 (ARM) | ARM64 | GPG |

### Stage 2: Container Image

- Base image: `node:22-alpine` or `debian:12-slim` (determined by runtime ADR).
- The binary is copied into the image at `/usr/local/bin/aidevos`.
- The image is tagged with both the platform version and `latest`.
- Published to GitHub Container Registry (`ghcr.io/anomalyco/ai-dev-os`).

### Stage 3: Artifact Publishing

Artifacts are uploaded to the GitHub Release as assets. See Release Artifacts below for the full list.

## Release Artifacts

| Artifact | Format | Published to |
|----------|--------|-------------|
| CLI binary (macOS Intel) | `.tar.gz` + `.tar.gz.sig` | GitHub Releases |
| CLI binary (macOS ARM64) | `.tar.gz` + `.tar.gz.sig` | GitHub Releases |
| CLI binary (Windows) | `.zip` + `.zip.sig` | GitHub Releases |
| CLI binary (Linux x86_64) | `.tar.gz` + `.tar.gz.sig` | GitHub Releases |
| CLI binary (Linux ARM64) | `.tar.gz` + `.tar.gz.sig` | GitHub Releases |
| Container image | OCI image | ghcr.io |
| Homebrew formula | `.rb` | `anomalyco/homebrew-tap` |
| npm package (SDK) | `.tgz` | npmjs.com |
| Documentation archive | `.tar.gz` | GitHub Releases |

## Release Notes

Every release publishes notes in the following format:

```
## v{MAJOR.MINOR.PATCH} — {YYYY-MM-DD}

### Summary
One-paragraph description of what this release delivers.

### What's New
- Bullet list of major features and subsystems added.

### Breaking Changes
- Bullet list of breaking changes with migration instructions or links.

### Bug Fixes
- Bullet list of fixed issues with issue numbers.

### Upgrade Instructions
Step-by-step upgrade instructions. May reference [Upgrade Notes](./UPGRADE_NOTES.md) for complex upgrades.

### Full Changelog
Link to the CHANGELOG.md entry.
```

Release notes are drafted during the RC phase and finalized at release time.

## Hotfix Process

A hotfix is an emergency patch release for a critical bug (security vulnerability, data loss, or complete feature breakage).

1. An issue is filed with the `hotfix` label and the severity is confirmed by the maintainers.
2. A branch `hotfix/{description}` is cut from the latest release tag, not `main`.
3. The fix is applied, reviewed, and merged to the hotfix branch.
4. The hotfix branch is merged directly into `main` and into any active release branches.
5. A patch release is cut following the same build pipeline — feature freeze and RC phases are skipped.
6. The hotfix is announced via GitHub Releases with the label `hotfix`.

Hotfixes SHOULD be rare. If a subsystem requires frequent hotfixes, its test coverage (see [Testing Strategy](./TESTING_STRATEGY.md)) should be reviewed.

## LTS Releases

Long-term support releases occur every 6 months (approximately January and July).

| Phase | Duration | Support |
|-------|----------|---------|
| Active support | 0–6 months after release | Full: bug fixes, security patches, feature backports |
| Security maintenance | 6–12 months after release | Security patches only |
| End of life | After 12 months | No support; users MUST migrate to the next LTS |

The migration path between LTS releases is documented in [Migration Guide](./MIGRATION_GUIDE.md). A minimum of one minor release of overlap is maintained — LTS `N` and LTS `N+1` are both supported for at least one month.

## Related Documents

- [Versioning](./VERSIONING.md)
- [CHANGELOG](./CHANGELOG.md)
- [Migration Guide](./MIGRATION_GUIDE.md)
- [Upgrade Notes](./UPGRADE_NOTES.md)
- [Testing Strategy](./TESTING_STRATEGY.md)
- [Eval Harness](./EVAL_HARNESS.md)
- [Implementation Roadmap](./IMPLEMENTATION_ROADMAP.md)
