# Support

## How to Get Help

### GitHub Issues

Report bugs and request features at
[github.com/ai-dev-os/ai-dev-os/issues](https://github.com/ai-dev-os/ai-dev-os/issues).

Before opening a new issue:
- Search existing issues (both open and closed) to avoid duplicates.
- Use the provided issue templates — they are enforced by GitHub
  Forms and ensure we have all information needed to help you.
- Include the output of `ai-dev-os doctor` and relevant log excerpts
  from `~/.ai-dev-os/runs/`.
- Remove or redact sensitive information such as API keys, tokens,
  and personal data before posting.
- Issues that do not follow the template may be closed without review.

### Discord Community

Join the AI Dev OS Discord server for community support, discussions,
and announcements. This is the fastest way to get help from
maintainers and other users in real time.

The server is organized into several channels:
- `#help` — general assistance and troubleshooting
- `#showcase` — share what you have built with AI Dev OS
- `#development` — contributor discussions and technical deep dives
- `#announcements` — release notes and project updates

Please review the channel descriptions and use the appropriate
channel for your message. [Invite link](https://discord.gg/ai-dev-os)

### Documentation

The full documentation is available at
[docs.ai-dev-os.org](https://docs.ai-dev-os.org) and is organized
into the following sections. Start here before opening an issue or
posting in Discord:

- [Getting Started](GETTING_STARTED.md) — installation, initial
  configuration, and running your first task
- [FAQ](FAQ.md) — answers to common questions by category
- [Troubleshooting](TROUBLESHOOTING.md) — solutions to common
  problems organized by symptom
- [CLI Reference](CLI.md) — complete command reference for the
  `ai-dev-os` CLI including all subcommands and flags
- [API Reference](API.md) — REST and gRPC API documentation for
  programmatic access and integration
- [Architecture Overview](ARCHITECTURE.md) — high-level system
  design and component descriptions

### Stack Overflow

Tag questions with `[ai-dev-os]` on Stack Overflow for community
answers. This is useful for integration questions — CI/CD pipeline
configuration, IDE integration, Docker setup, and deployment
patterns — that benefit from broad visibility and searchability.

## Bug Reports

When filing a bug report, include the following information to help
us diagnose and fix the issue quickly. Incomplete reports may be
delayed or closed:

1. **Environment**: OS version, AI Dev OS version (`ai-dev-os
   version`), model provider(s) and model names, Docker version
   (if used), and hardware specs (CPU, RAM, GPU).
2. **Expected behavior**: A clear, concise description of what you
   expected to happen.
3. **Actual behavior**: A clear, concise description of what
   actually happened, including full error messages, stack traces,
   and unexpected output.
4. **Reproduction steps**: A minimal, complete, and verifiable
   sequence of commands or actions. Include exact task descriptions
   if applicable. Assume a fresh installation with defaults.
5. **Logs and diagnostics**: Full output of `ai-dev-os doctor` and
   relevant run logs (`ai-dev-os logs <run-id> --all`). For crashes
   include the daemon log at `~/.ai-dev-os/daemon.log`.
6. **Configuration**: A sanitized copy of `~/.ai-dev-os/config.yaml`
   with all API keys, tokens, and secrets removed.

## Feature Requests

Feature requests are reviewed by the core team every sprint cycle.
When submitting one:

- Describe the problem you are solving, not just a proposed
  solution. This helps evaluate alternative approaches.
- Explain how the feature benefits the broader community, not just
  your specific use case.
- Include examples, wireframes, or mockups if applicable.
- Tag the issue with the `enhancement` label.

Large or controversial features should be discussed in a GitHub
Discussion before filing a formal issue.

## Security Disclosures

For security vulnerabilities, **do not** open a public issue. Send
details to `security@ai-dev-os.org`. We follow a 90-day coordinated
disclosure timeline:

- Acknowledgment within 48 hours.
- Initial assessment and patch timeline within 5 business days.
- Credit in release notes and security advisory (unless you request
  anonymity).
- No legal action against researchers acting in good faith.

## Commercial Support

Commercial support, SLAs, on-premises deployments, and custom
integrations are available through the AI Dev OS Foundation.
Contact `enterprise@ai-dev-os.org` for pricing and availability.

Available support tiers:

- **Community**: Best-effort via Discord and GitHub. No SLA. Free.
- **Standard**: 48-hour response SLA, email support, quarterly
  update reviews.
- **Enterprise**: 4-hour response SLA, dedicated support engineer,
  priority feature requests, custom integrations, on-premises
  deployment support.

## Related Documents

- [FAQ](FAQ.md) — frequently asked questions
- [Troubleshooting](TROUBLESHOOTING.md) — common problems and solutions
- [Code of Conduct](CODE_OF_CONDUCT.md) — community guidelines
- [Contributing](CONTRIBUTING.md) — how to contribute
