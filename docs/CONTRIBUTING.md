# Contributing to AI Dev OS

Thank you for considering contributing to AI Dev OS. This document describes the processes and standards for all types of contributions.

## How to Contribute

- **Documentation**: Improve or expand guides, API references, tutorials, and code comments
- **Code**: Fix bugs, add features, optimize performance, or improve test coverage
- **Prompts**: Contribute system prompts, few-shot examples, and agent definitions that improve model output quality
- **Bug reports**: File detailed, reproducible bug reports with logs and reproduction steps
- **Feature requests**: Propose well-scoped enhancements that align with the project roadmap
- **Community support**: Answer questions on Discord and Stack Overflow, triage issues, and review documentation

## Development Setup

### Prerequisites
- Python 3.10+ or Node.js 18+ (depending on the component)
- Rust toolchain (for native extensions and performance-critical modules)
- Docker (required for running sandbox integration tests)
- Git LFS (for large test fixtures)

### Local Setup
```bash
git clone https://github.com/ai-dev-os/ai-dev-os.git
cd ai-dev-os
make install-dev       # Install all dependencies in development mode
make build             # Build all components
make test              # Run the full test suite to verify your setup
```

### Running Tests
```bash
make test              # Run all test suites
make test-unit         # Run unit tests only (fast, for quick iteration)
make test-integration  # Run integration tests (requires Docker)
make test-e2e          # Run end-to-end tests (requires Docker and a model)
make test-coverage     # Run tests and generate coverage reports
```

## Pull Request Process

1. Fork the repository and create a feature branch from `main`
2. Make your changes following the coding and documentation standards below
3. Write or update tests to cover your changes — new code without tests will not be merged
4. Run the full test suite and ensure it passes
5. Commit using the required commit message format
6. Push your branch and open a pull request against `main`. Fill out the PR template completely
7. Address reviewer feedback. Keep the branch updated with `main`
8. A maintainer will merge once all CI checks pass and at least one approval is received

## Coding Standards

- **Python**: Follow PEP 8. Type hints are required for all public APIs and recommended for internal ones. Run `ruff check .` before committing
- **TypeScript/JavaScript**: Follow the project's ESLint and Prettier configuration. Prefer typed interfaces over `any`
- **Rust**: Follow `rustfmt` conventions. Run `cargo clippy` before committing. Document all public items
- **Shell scripts**: Use shellcheck. Prefer POSIX sh for maximum portability
- All new code must include tests — unit tests for logic, integration tests for workflows
- Keep functions focused on a single responsibility. Modules should have clear boundaries
- Use descriptive variable names. Avoid abbreviations unless they are universally understood

## Documentation Standards

- All public APIs must include docstrings (Google style for Python, JSDoc for TypeScript, rustdoc for Rust)
- Every user-facing feature must include CLI help text and a documentation page in `docs/`
- Documentation is written in Markdown with a maximum line length of 100 characters
- Code examples must be runnable. Add them to the test suite if feasible
- Include screenshots or terminal recordings for UI or CLI changes
- Cross-reference related documentation pages using relative Markdown links

## Commit Message Format

We follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body]
[optional footer]
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`, `revert`

Scope is the component or module being changed (e.g., `agent`, `cli`, `orchestrator`, `docs`).

Examples:
- `feat(agent): add retry logic for tool execution failures`
- `fix(cli): handle SIGTERM during long-running tasks`
- `docs: update installation instructions for Windows ARM64`

The description must be lowercase, imperative, and 72 characters or fewer.

## Review Process

- All PRs require at least one approval from a core maintainer
- Maintainers review within 48 hours on average during business days
- PRs introducing breaking changes must include a `BREAKING CHANGE` footer in the commit and a migration guide
- Large changes should be preceded by a GitHub Discussion or RFC
- Automated checks (lint, test, build, security scan) must pass before human review begins

## Related Documents

- [Local Development Guide](DEVELOPMENT.md)
- [Code of Conduct](CODE_OF_CONDUCT.md)
- [Testing Strategy](TESTING.md)
- [Architecture Overview](ARCHITECTURE.md)
