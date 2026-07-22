# Local Dev

> Guide for contributing to the AI Dev OS codebase — from cloning to shipping.

## Overview

AI Dev OS is written in TypeScript and compiled to a native binary via Bun. The repository is a monorepo containing the CLI, Kernel, Router, plugin system, and documentation. This guide covers the development workflow, tooling, and conventions.

## Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| **Bun** | 1.2+ | Runtime, package manager, bundler |
| **Node.js** | 20+ | Some tooling scripts |
| **TypeScript** | 5.5+ | Language (handled by Bun) |
| **Git** | 2.40+ | Version control |

Install Bun: `curl -fsSL https://bun.sh/install | bash`

## Repository Setup

```bash
git clone https://github.com/aidevos/aidevos.git
cd aidevos
bun install
bun run build
```

The monorepo structure: `src/` (cli, kernel, router, providers, shared), `tests/`, `docs/`, `scripts/`, and a root `package.json`.

## Development Workflow

The standard loop: **code → lint → test → build**

### 1. Code

Make changes in `src/`. TypeScript strict mode is enforced. Run the dev watcher for fast iteration:

```bash
bun run dev
```

### 2. Lint

```bash
bun run lint          # Check for issues
bun run lint:fix      # Auto-fix
```

Linting covers TypeScript strict checks, import ordering, and formatting via Biome.

### 3. Test

```bash
bun run test                # All tests
bun run test:unit           # Unit tests only
bun run test:integration    # Integration tests (requires Ollama)
bun run test:coverage       # With coverage report
```

Tests use Bun's built-in test runner. Integration tests require an active Ollama instance. Test files use the `.test.ts` convention in `tests/`.

### 4. Build

```bash
bun run build          # TypeScript compilation
bun run build:binary   # Native binary via Bun.compile
bun run build:all      # Both
```

The compiled binary is at `dist/aidevos`.

Run `bun run test` for the full suite. Single file: `bun test tests/kernel/planner.test.ts`. Watch mode: `bun run test:watch`.

Build the binary with `bun run build:binary` — output at `dist/aidevos` with no runtime dependencies. Cross-compile targets via `./scripts/cross-build.sh --target linux-arm64`.

Preview docs locally with `bun run docs:serve` (docsify, opens at `http://localhost:3000`).

## Debugging Tips

| Situation | Approach |
|-----------|----------|
| **Verbose logging** | `AIDEVOS_LOG_LEVEL=debug` or `--verbose` |
| **Kernel traces** | Enable `[tracing]` in config, view at `http://localhost:4318` |
| **Binary crashes** | Run with `bun run dev` for full stack traces |
| **Provider issues** | `aidevos doctor --verbose` |
| **Config not loading** | `aidevos doctor --show-config` |
| **Memory inspector** | `aidevos memory query` |

## Commit Conventions

AI Dev OS follows [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>
```

| Type | Usage |
|------|-------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `refactor` | Code change, no behavior change |
| `test` | Adding/fixing tests |
| `chore` | Build, CI, tooling |

Scopes include: `cli`, `kernel`, `router`, `memory`, `providers`, `config`, `docs`.

```
feat(kernel): add timeout to planner phase
fix(router): handle empty fallback list
docs(cli): document --json flag
```

## CI/CD Overview

| Pipeline | Trigger | What it does |
|----------|---------|--------------|
| **PR checks** | Every PR | `lint → test:unit → test:integration → build:binary` |
| **Main branch** | Push to `main` | All PR checks + publish `:latest` Docker image |
| **Release** | Tag `v*` | Build all platform binaries, publish GitHub release, npm + Homebrew + Docker |

CI is defined in `.github/workflows/`. All PR checks must pass before merge.

## Related Documents

- [Contributing](./CONTRIBUTING.md) — code of conduct and PR process
- [Testing Strategy](./TESTING_STRATEGY.md) — test architecture and coverage
- [Code of Conduct](./CODE_OF_CONDUCT.md) — community standards
- [CLI Reference](./CLI.md) — testing the built binary
- [Installation](./INSTALLATION.md) — installing from source
