# Integrations — External System Catalog

> The master catalog and integration guide for every external system AI Dev OS connects to. Each integration is a configured bridge — MCP server, Plugin SDK extension, webhook, or direct API client — governed by the rules in this document. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

AI Dev OS connects to external systems through four integration patterns. Every integration is registered in the Integration Registry so the Kernel can discover, authenticate, and monitor it. Integrations are distinct from third-party API calls (see [Third-Party APIs](./THIRD_PARTY_APIS.md)) — an integration is a **configured bridge** with its own lifecycle, while an API call is a stateless request.

Integration lifecycle: `discovered → registered → authenticated → active → (degraded) → deactivated`.

## Integration Categories

| Category | Description | Example Integrations |
|----------|-------------|---------------------|
| **Model Providers** | LLM inference APIs and self-hosted models | OpenAI, Anthropic, Google, Mistral, Ollama |
| **Version Control** | Git hosting and code review platforms | GitHub, GitLab |
| **Communication** | Messaging and collaboration tools | Slack, Discord |
| **Issue Tracking** | Bug and task trackers | GitHub Issues, Linear, Jira |
| **CI/CD** | Continuous integration and deployment | GitHub Actions, Jenkins |
| **Monitoring** | Observability and alerting platforms | Datadog, Sentry, Grafana |
| **Secret Stores** | Secrets management systems | Vault, AWS Secrets Manager, 1Password |
| **Knowledge Bases** | Document stores and wikis | Obsidian, Confluence, Notion |
| **Search** | Web and code search engines | SearXNG, Brave Search |

## Catalog

### Model Providers

| Integration | Type | Auth | Capabilities | Config Doc |
|-------------|------|------|-------------|------------|
| OpenAI | Direct API | API key | Chat, embeddings, vision, STT | [OpenAI](./OPENAI_INTEGRATION.md) |
| Anthropic | Direct API | API key | Chat, extended thinking, tool use | [Anthropic](./ANTHROPIC_INTEGRATION.md) |
| Google | Direct API | API key / OAuth | Gemini chat, embeddings | [Google](./GOOGLE_INTEGRATION.md) |
| Mistral | Direct API | API key | Chat, embeddings | [Mistral](./MISTRAL_INTEGRATION.md) |
| Ollama | MCP / Direct API | None (localhost) | Local models, custom models | [Ollama](./OLLAMA_INTEGRATION.md) |

### Version Control & Issue Tracking

| Integration | Type | Auth | Capabilities | Config Doc |
|-------------|------|------|-------------|------------|
| GitHub | MCP + Webhook | OAuth / PAT | PR review, issue management, search, webhook events | [GitHub](./GITHUB_ANALYSIS.md) |

### Communication & Monitoring

| Integration | Type | Auth | Capabilities |
|-------------|------|------|-------------|
| Slack | Webhook | Webhook URL | Send notifications to channels |
| Discord | Webhook | Webhook URL | Send notifications to channels |
| Datadog | Direct API | API + App keys | Push metrics, traces, logs |
| Sentry | Direct API | DSN / Auth token | Capture errors, performance |

### Secret Stores

| Integration | Type | Capabilities |
|-------------|------|-------------|
| Vault | Plugin SDK | Dynamic secrets, leasing, rotation |
| AWS Secrets Manager | Direct API | Static secret retrieval, rotation |
| 1Password | Plugin SDK | Secret reference resolution |

## Supported Integration Patterns

### MCP Server

The primary pattern. External tools expose their capabilities as MCP tools and resources. AI Dev OS connects as an MCP client. See [MCP](./MCP.md).

**Use when**: the external system has a rich API surface and needs two-way interaction (read + write). Examples: GitHub (PR review, issue creation), filesystem (read + write files).

### Plugin SDK

For custom or complex integrations that run as subprocesses. The Plugin SDK provides a JSON-RPC bridge with capability-based security. See [Plugin SDK](./PLUGIN_SDK.md).

**Use when**: the integration needs local binary execution, hardware access, or custom business logic. Examples: Vault secret leasing, image processing.

### Webhook

Outgoing event notifications. The external system receives POST requests when events occur. See [Webhooks](./WEBHOOKS.md).

**Use when**: the external system only needs to be notified (fire-and-forget with retry). Examples: Slack notifications, Discord alerts.

### Direct API

Stateless HTTP client calls to external REST or GraphQL APIs. Managed through the [Model Providers](./MODEL_PROVIDERS.md) and [Third-Party APIs](./THIRD_PARTY_APIS.md) layers.

**Use when**: simple request-response without persistent connection. Examples: OpenAI inference, SearXNG search.

## How to Add a New Integration

1. **Choose a pattern**: MCP, Plugin SDK, Webhook, or Direct API. Base the choice on the interaction model and security requirements.
2. **Register in the Integration Registry**: a new entry in the integrations config with `{ name, pattern, auth, endpoints }`.
3. **Add auth**: store credentials via [Secrets Management](./SECRETS_MANAGEMENT.md). NEVER inline secrets in config.
4. **Define event bindings**: if the integration emits or consumes events, register them in the [Event Bus](./EVENT_BUS.md).
5. **Add observability**: each integration MUST export metrics (call count, latency, error rate) and log every call.
6. **Write a config doc**: a markdown file in `docs/` following the pattern of existing integration docs (e.g., [GitHub](./GITHUB_ANALYSIS.md), [OpenAI](./OPENAI_INTEGRATION.md)).
7. **Pass the Guardian**: every integration MUST pass [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md) validation before activation.

## Related Documents

- [MCP](./MCP.md)
- [Plugin SDK](./PLUGIN_SDK.md)
- [Webhooks](./WEBHOOKS.md)
- [Third-Party APIs](./THIRD_PARTY_APIS.md)
- [Model Providers](./MODEL_PROVIDERS.md)
- [Secrets Management](./SECRETS_MANAGEMENT.md)
- [Architecture Guardian](./ARCHITECTURE_GUARDIAN.md)
- [System Overview](./SYSTEM_OVERVIEW.md)
