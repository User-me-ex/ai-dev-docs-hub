# Prompt Spec: {Prompt Name}

> Prompt specifications define the structure, variables, and testing criteria for
> system prompts used by agents. Use this template when creating a new prompt,
> modifying an existing one, or auditing prompt behaviour.

---

## Metadata

| Field           | Value                     |
|----------------|---------------------------|
| **Prompt ID**   | PROMPT-XXXX               |
| **Name**        | {Short, descriptive name} |
| **Version**     | {SemVer — e.g., 1.2.0}   |
| **Owner**       | {Team / person}           |
| **Status**      | active \| deprecated \| experimental |
| **Agents Using**| {List of agent IDs}       |
| **Created**     | YYYY-MM-DD                |
| **Last Updated**| YYYY-MM-DD                |

---

## 1. Purpose

Describe what this prompt is designed to accomplish. What agent behaviour does it
drive? When and why is it invoked?

**Example:**
> The PR Review Prompt instructs the PR-Reviewer agent to analyse GitHub pull
> request diffs and metadata, check for style guide violations, security issues,
> and ADR compliance, and produce structured review comments. It is invoked
> automatically when a PR is opened or updated in the `ai-dev-docs-hub` repo.

---

## 2. Inheritance

| Attribute | Value |
|-----------|-------|
| **Base prompt** | {ID of base prompt, or "none"} |
| **Overrides** | {List of sections / variables this prompt overrides from the base} |
| **Augments** | {List of sections / variables this prompt adds beyond the base} |

**Example:**
> Inherits from `PROMPT-000: Base Code Review Prompt`. Overrides the
> `STYLE_GUIDE_URL` variable. Augments with an `ADR_CHECK` section that is
> specific to this project.

---

## 3. Prompt Template

Below is the full prompt template. Variables are indicated by `{{variable_name}}`.
Comments (in `_italic_`) are instructions for the prompt author.

### 3.1 System Role

```
You are {{AGENT_NAME}}. Your role is {{ROLE_DESCRIPTION}}.

You operate in the {{PROJECT_NAME}} environment.
Model access is provided via Nine Router (localhost:20128/v1)
using role binding {{NINE_ROUTER_ROLE}}.
```

### 3.2 Instructions

```
When you receive a {{EVENT_TYPE}} event, follow these steps:

1. {{STEP_1}}
2. {{STEP_2}}
3. {{STEP_3}}

{{SPECIAL_INSTRUCTIONS}}
```

### 3.3 Formatting Rules

```
Output must follow this structure:

## Review Summary
{Summary of findings}

## Findings
| Category | File | Line | Severity | Message |
|----------|------|------|----------|---------|

## Verdict
{Approve / Changes Requested / Comment}

---
_Use the exact column headers shown above._
```

### 3.4 Constraints

```
- {{CONSTRAINT_1}}
- {{CONSTRAINT_2}}
- Do not {{NEGATIVE_CONSTRAINT}}
- Maximum response length: {{MAX_TOKENS}} tokens
```

---

## 4. Variables

### 4.1 Variable Table

| Variable | Type | Required | Default | Description | Source |
|----------|------|----------|---------|-------------|--------|
| `AGENT_NAME` | string | Yes | — | Name of the agent using this prompt | Agent config |
| `ROLE_DESCRIPTION` | string | Yes | — | One-sentence role description | Agent config |
| `PROJECT_NAME` | string | Yes | — | Full project name | Environment |
| `EVENT_TYPE` | enum | Yes | — | `push`, `pull_request`, `issue_comment` | Webhook event |
| `STEP_1` | string | No | "Read the input" | First instruction step | Prompt author |
| `STEP_2` | string | No | — | Second instruction step | Prompt author |
| `STEP_3` | string | No | — | Third instruction step | Prompt author |
| `SPECIAL_INSTRUCTIONS` | text | No | — | Project-specific override instructions | Prompt author |
| `CONSTRAINT_1` | string | No | — | Behavioural constraint | Prompt author |
| `CONSTRAINT_2` | string | No | — | Behavioural constraint | Prompt author |
| `NEGATIVE_CONSTRAINT` | string | No | — | Explicit prohibition | Prompt author |
| `MAX_TOKENS` | integer | Yes | 4096 | Maximum response tokens | Agent config |
| `STYLE_GUIDE_URL` | URL | Yes | — | Link to project style guide | Environment |
| `ALLOWED_TOOLS` | list | Yes | — | Tools the agent may use | Agent config |
| `NINE_ROUTER_ROLE` | string | Yes | — | Role binding for Nine Router model access | Agent config |

### 4.2 Variable Dependency Graph

```
AGENT_NAME ──┐
ROLE_DESCRIPTION ──┼──► SYSTEM_ROLE
PROJECT_NAME ──┘
                   
EVENT_TYPE ──┐
STEP_1 ──────┤
STEP_2 ──────┼──► INSTRUCTIONS
STEP_3 ──────┤
SPECIAL_INSTRUCTIONS ──┘
```

### 4.3 Variable Injection Order

Variables are resolved in the following order (later overrides earlier):

1. **Default values** (hardcoded in prompt definition)
2. **Agent-level overrides** (configured in agent spec)
3. **Runtime context** (injected by the coordinator at invocation time)
4. **User overrides** (explicitly passed by the calling agent or user)

---

## 5. Evaluation

### 5.1 Evaluation Dataset

| Test Case | Input | Expected Output | Notes |
|-----------|-------|----------------|-------|
| TC-01 | PR with formatting-only diff | "no issues found" verdict | Verifies no false positives |
| TC-02 | PR with hardcoded secret | "security" finding at `critical` severity | Verifies security detection |
| TC-03 | PR with missing ADR | "ADR required" finding | Verifies ADR compliance check |
| TC-04 | Empty diff (whitespace only) | Graceful handling, no crash | Verifies edge-case robustness |

### 5.2 Evaluation Metrics

| Metric | Target | Method |
|--------|--------|--------|
| Precision (findings) | ≥ 0.95 | Human review of 100 sampled outputs |
| Recall (critical findings) | ≥ 0.99 | Red-team injection of known issues |
| Format compliance | 100 % | Regex check on output structure |
| Latency p95 | ≤ 10 s | Production monitoring |

### 5.3 Running Evaluations

```
# Run the evaluation suite
$ python eval/eval_prompt.py --prompt-id PROMPT-XXXX --suite full

# Compare two prompt versions
$ python eval/compare_prompts.py --baseline PROMPT-XXXX@1.0 --candidate PROMPT-XXXX@1.1
```

---

## 6. Version History

| Version | Date       | Author | Changes |
|---------|-----------|--------|---------|
| 0.1     | YYYY-MM-DD| {Name} | Initial draft |
| 0.2     | YYYY-MM-DD| {Name} | Added variable dependency graph |
| 0.3     | YYYY-MM-DD| {Name} | Updated evaluation dataset — added TC-04 |
| 1.0     | YYYY-MM-DD| {Name} | First stable release |
| 1.1     | YYYY-MM-DD| {Name} | Added `SPECIAL_INSTRUCTIONS` variable; updated injection order |
| 2.0     | YYYY-MM-DD| {Name} | Major rewrite — split role and instructions into separate sections |

---

## 7. Rollout & Rollback Plan

| Action | Steps | Success Criteria |
|--------|-------|-----------------|
| **Rollout** | 1. Deploy to staging 2. Run eval suite 3. Shadow-deploy to 5 % traffic 4. Ramp to 100 % | All eval metrics at or above target; no P1 alerts in 48 h |
| **Rollback** | Revert to previous version via coordinator config | Previous eval metrics restored within 10 min |

---

## 8. Related Documents

- [AGENT_SPEC-001: PR-Reviewer Agent](../agents/AGENT_SPEC-001.md)
- [AGENT_SPEC-002: Code-Generator Agent](../agents/AGENT_SPEC-002.md)
- [ADR-0023: Prompt Versioning Strategy](../adrs/ADR-0023.md)

---

*Template version 2.0 — See [README.md](./README.md) for prompt specification workflow guidance.*
