# Agent Spec: {Agent Name}

> Agent specifications define the purpose, boundaries, and runtime configuration
> of an AI agent in the multi-agent system. Use this template when creating a new
> agent, onboarding it into the router, or auditing an existing agent's behaviour.

---

## Metadata

| Field           | Value                     |
|----------------|---------------------------|
| **Agent ID**    | AGENT-XXXX                |
| **Name**        | {Short, memorable name}   |
| **Role**        | {One-line role summary}   |
| **Group**       | {e.g., Code, Docs, Ops}  |
| **Owner**       | {Team / person}           |
| **Status**      | active \| inactive \| deprecated |
| **Created**     | YYYY-MM-DD                |
| **Last Updated**| YYYY-MM-DD                |

---

## 1. Purpose

Describe the single responsibility of this agent. What problem does it solve?
What scope does it operate within?

**Example:**
> The PR-Reviewer Agent reviews pull requests in the `ai-dev-docs-hub` repository.
> It checks for adherence to the project's style guide, detects common security
> antipatterns, validates that the ADR template was used for architecture changes,
> and leaves structured inline feedback on the diff.

---

## 2. Behaviour & Guardrails

### 2.1 Core Behaviour

List the agent's primary behaviours in priority order:

1. {Behaviour 1 — e.g., Review PR diff within 30 seconds of `opened` or `synchronize` event}
2. {Behaviour 2 — e.g., Post review summary as a PR comment using the format defined in §6}
3. {Behaviour 3 — e.g., Assign labels (`needs-review`, `approved`, `changes-requested`) based on confidence threshold}

### 2.2 Guardrails

Explicit boundaries the agent must not cross:

| Guardrail | Enforcement |
|-----------|------------|
| {e.g., Never auto-approve PRs — human always decides} | System prompt + post-hoc audit |
| {e.g., Never modify files outside the PR diff} | Sandboxed filesystem |
| {e.g., Never access secrets or credentials} | Tool restrictions; env vars filtered |
| {e.g., Maximum 1 review per PR per hour} | Rate limit in coordinator |

### 2.3 Error Handling

Describe what the agent does when it encounters an unexpected situation:

| Scenario | Behaviour |
|----------|-----------|
| PR diff exceeds token limit | Summarize file-level changes only; skip inline comments |
| Downstream API (GitHub API) unavailable | Retry 3 times with exponential backoff; log and skip |
| Ambiguous request | Request clarification via PR comment with structured options |

---

## 3. Inputs & Outputs

### 3.1 Inputs

| Input | Source | Format | Example |
|-------|--------|--------|---------|
| PR diff | GitHub webhook | Unified diff (text) | `diff --git a/file.ts b/file.ts` |
| PR metadata | GitHub API | JSON | `{ "number": 42, "title": "..." }` |
| Repository rules | AGENTS.md | Markdown | Local file read |
| Style guide | `/style-guide/` directory | Markdown + YAML | `style-guide/typescript.md` |

### 3.2 Outputs

| Output | Destination | Format | Example |
|--------|------------|--------|---------|
| Review summary | PR comment | Markdown with checklists | See §6 |
| Inline comments | PR review thread | GitHub Review API | `Line 42: This variable shadows...` |
| Labels | PR metadata | GitHub Issue Labels | `needs-review` |
| Metrics | Local file (JSONL) | JSON metrics | `{ "agent": "pr-reviewer", "duration_ms": 1234 }` |

---

## 4. Tools Allowed

### 4.1 Approved Tools

| Tool | Purpose | Rate Limit | Requires Approval |
|------|---------|-----------|-------------------|
| `read_file` | Read files in the PR diff | 100 calls / minute | No |
| `grep_search` | Search codebase for patterns | 50 calls / minute | No |
| `write_comment` | Post PR comments | 10 calls / PR | No |
| `apply_label` | Add labels to PR | 5 calls / PR | No |
| `github_api_get` | Fetch PR metadata | 30 calls / minute | No |
| `github_api_post` | Write to PR (reviews, status checks) | 10 calls / minute | No |

### 4.2 Blocked Tools

| Tool | Reason |
|------|--------|
| `delete_file` | Destructive operation — never allowed for review agents |
| `write_file` | Writing files outside PR diff is out of scope |
| `merge_pr` | Human-in-the-loop; agent may not merge |
| `exec_shell` | Security risk — arbitrary command execution |

### 4.3 Tool Configuration

```json
{
  "allowed_tools": ["read_file", "grep_search", "write_comment", "apply_label", "github_api_get", "github_api_post"],
  "max_calls_per_minute": 100,
  "sandbox_access": "read-only on repo, read-write on /tmp",
  "timeout_per_call_ms": 30000
}
```

---

## 5. System Prompts

### 5.1 Role Prompt

```
You are PR-Reviewer, an AI agent that reviews pull requests for the
ai-dev-docs-hub project. You follow the project's style guide and conventions.

Your job is to:
1. Read the PR diff and description
2. Check for style guide violations
3. Check ADR compliance (architecture changes must include an ADR)
4. Identify security issues (hardcoded secrets, injection risks)
5. Leave structured, actionable feedback
```

### 5.2 Instructions Prompt

```
When reviewing a PR:
- Start with a summary table (files changed, lines added/deleted, categories)
- Group feedback by category: Style, Security, Architecture, Correctness
- For each issue, include: file path, line number, severity (critical/warning/nit), suggestion
- End with a verdict: Approve (no blockers), Changes Requested (blockers found), or Comment (nits only)

Never auto-approve a PR that modifies:
- CI/CD configuration
- Authentication/authorization logic
- Database migrations
- Dependency manifests
```

### 5.3 System Prompt References

The agent uses prompt specifications defined in [PROMPT_SPEC-001](./prompts/PR_REVIEW.md)
for its main review prompt, and [PROMPT_SPEC-002](./prompts/SUMMARIZE.md) for the
summarization step. See the Prompt Specs directory for full variable documentation.

---

## 6. Memory Scope

| Memory Store | Access | Retention | Purpose |
|-------------|--------|-----------|---------|
| Ephemeral (conversation history) | Read/Write | Duration of task | Tracking current PR context |
| Long-term (vector store) | Read only | Indefinite | Retrieving past review patterns |
| Agent state (local SQLite) | Read/Write | TTL 24 hours | Rate limiting, deduplication |
| Shared context (local SQLite) | Read only | Indefinite | Reading team conventions, glossary |

### 6.1 Memory Configuration

```yaml
memory:
  ephemeral:
    provider: in_memory
    max_tokens: 64000
  long_term:
    provider: sqlite-vec
    path: ${LOCAL_DATA_DIR}/vectors.db
    collection: pr_review_patterns
  state:
    provider: sqlite
    path: ${LOCAL_DATA_DIR}/agent_state.db
    ttl_seconds: 86400
```

---

## 7. Model Binding (via Nine Router)

| Attribute | Value |
|-----------|-------|
| **Nine Router endpoint** | `http://localhost:20128/v1` |
| **Role binding** | `reviewer` (maps to model via Nine Router role config) |
| **Fallback role** | `reviewer-fallback` |
| **Reasoning role** | `reviewer-reasoning` (for complex reviews) |
| **Max output tokens** | 4096 |
| **Temperature** | 0.2 (low for consistent structured output) |
| **Routing hint** | `group: docs` (Nine Router may use this for locality) |

> All model access flows through Nine Router. The agent never calls a provider
> directly. Model assignments are configured in Nine Router's role-to-model
> bindings, not in the agent spec.

---

## 8. Acceptance Tests

| Test ID | Scenario | Expected Outcome |
|---------|----------|-----------------|
| AT-01 | PR with formatting-only changes | Agent posts "no issues found" comment; no labels applied |
| AT-02 | PR with hardcoded API key | Agent flags as `critical` security issue; applies `security` label |
| AT-03 | PR with architecture change but no ADR | Agent requests ADR; applies `needs-adr` label |
| AT-04 | PR diff exceeds 2000 lines | Agent summarizes at file level; skips inline comments |
| AT-05 | GitHub API returns 429 (rate limit) | Agent retries with backoff; logs warning |
| AT-06 | Agent invoked on non-PR event | Agent returns no-op; logs "unexpected event type" |

### 8.1 Test Harness

```
# Run acceptance tests locally
$ python tests/test_agent.py --agent pr-reviewer --suite smoke
$ python tests/test_agent.py --agent pr-reviewer --suite full

# Expected: all tests pass (exit code 0)
```

---

## 9. Changelog

| Version | Date       | Author | Changes |
|---------|-----------|--------|---------|
| 0.1     | YYYY-MM-DD| {Name} | Initial draft |
| 0.2     | YYYY-MM-DD| {Name} | Added memory scope configuration |
| 0.3     | YYYY-MM-DD| {Name} | Updated tool list — removed `exec_shell` |
| 1.0     | YYYY-MM-DD| {Name} | Approved for production |

---

## 10. Related Documents

- [PROMPT_SPEC-001: PR Review Prompt](./prompts/PR_REVIEW.md)
- [PROMPT_SPEC-002: Summarization Prompt](./prompts/SUMMARIZE.md)
- [SUBSYSTEM_SPEC-001: Agent Runtime](./SUBSYSTEM_SPEC-001.md)
- [ADR-0012: Agent Tool Security Model](../adrs/ADR-0012.md)

---

*Template version 2.0 — See [README.md](./README.md) for agent specification workflow guidance.*
