# Templates

Reusable Markdown templates for the AI Dev OS documentation system. Each template
is a self-contained document with embedded instructions, examples, and conventions
to guide both human authors and AI agents in creating consistent, high-quality
specifications.

> **Architecture:** This system follows a **Local-First** architecture. All data
> is stored locally (SQLite, filesystem). All model access flows through
> [Nine Router](http://localhost:20128/v1). Cloud services are opt-in exceptions,
> never the default.

---

## Quick Reference

| Template | File | Purpose | Best Used When | Typical Length |
|----------|------|---------|----------------|----------------|
| **ADR** | [ADR.md](./ADR.md) | Architecture Decision Record | A significant architectural or infrastructure choice needs a permanent record | 150–300 lines |
| **RFC** | [RFC.md](./RFC.md) | Request For Comments | A design needs structured feedback before committing to implementation | 150–250 lines |
| **PRD** | [PRD.md](./PRD.md) | Product Requirements Document | A feature or product needs product requirements, user stories, and success metrics | 150–300 lines |
| **TRD** | [TRD.md](./TRD.md) | Technical Requirements Document | A system needs non-functional requirements, architecture constraints, and interface specs | 150–350 lines |
| **Agent Spec** | [AGENT_SPEC.md](./AGENT_SPEC.md) | Agent Specification | A new AI agent needs its purpose, tools, prompts, and configuration defined | 150–300 lines |
| **Prompt Spec** | [PROMPT_SPEC.md](./PROMPT_SPEC.md) | Prompt Specification | A system prompt needs versioning, variable documentation, and evaluation criteria | 150–250 lines |
| **Subsystem Spec** | [SUBSYSTEM_SPEC.md](./SUBSYSTEM_SPEC.md) | Subsystem Specification | A distinct subsystem needs a comprehensive boundary, architecture, and interface spec | 200–400 lines |

---

## When to Use Which Template

### ADR (Architecture Decision Record)

**Use when:** You are making a significant architectural choice — choosing a
database, adopting a pattern, defining a service boundary, deciding on a protocol.

**Do NOT use for:** Day-to-day code patterns, minor library upgrades, bug fixes.
Those go in PR descriptions or commit messages.

**Process:** Draft → Review (via PR) → Merge (accepted). Once merged, the ADR is
considered accepted. Superseded ADRs remain in the repository for historical
reference with a `superseded` status and a pointer to the replacement.

### RFC (Request For Comments)

**Use when:** A design is still fluid and you want broad input before locking in
decisions. RFCs are lighter-weight than ADRs and are meant to encourage discussion.

**Do NOT use for:** Decisions you have already made. If the answer is settled,
write an ADR instead.

**Process:** Draft → Comment Period (5 business days) → Revisions → Accept/Reject.
Accepted RFCs may lead to one or more ADRs, PRDs, or TRDs downstream.

### PRD (Product Requirements Document)

**Use when:** You need to define what a feature or product does — problem statement,
goals, user stories, acceptance criteria, and success metrics.

**Do NOT use for:** Technical implementation details (those go in a TRD or
Subsystem Spec). Keep the PRD focused on what, not how.

**Process:** Draft → Stakeholder Review → Approve → Hand off to engineering.
Engineering may produce one or more TRDs from the PRD.

### TRD (Technical Requirements Document)

**Use when:** A system or component needs detailed technical constraints —
performance targets, security requirements, platform support, interfaces, data
model, and migration plan.

**Do NOT use for:** Product requirements. If a requirement has no clear technical
implication, it belongs in the PRD.

**Process:** The TRD should reference a PRD or an RFC. It is reviewed by
engineering leads and approved before implementation begins.

### Agent Spec

**Use when:** Adding a new AI agent to the multi-agent system, or auditing an
existing agent's behaviour, tools, and configuration.

**Do NOT use for:** Prompt-level changes (use Prompt Spec for that) or agent
runtime infrastructure (use Subsystem Spec for the runtime).

**Process:** Draft → Review with agent owners → Register with routing system →
Deploy. The spec becomes the source of truth for the agent's capabilities and
boundaries.

### Prompt Spec

**Use when:** Creating or modifying a system prompt that an agent uses. Every
prompt that is shared across agents or has complex variable injection should be
spec'd.

**Do NOT use for:** One-off prompts in a single agent's configuration. Those can
be documented inline in the agent spec.

**Process:** Version → Test → Review → Deploy. Prompt changes follow a
structured rollout with canary testing and rollback capability.

### Subsystem Spec

**Use when:** Defining a new subsystem or documenting an existing one. This is
the most comprehensive template — it covers architecture, interfaces, data model,
failure modes, security, observability, and acceptance criteria.

**Do NOT use for:** Single-service or single-module documentation. The Subsystem
Spec is for a subsystem that spans multiple components or services.

**Process:** The Subsystem Spec is informed by one or more ADRs (architecture),
PRDs (product intent), and TRDs (technical constraints). It is reviewed by all
teams that interact with the subsystem boundary.

---

## Template Versioning

| Version | Date       | Author | Changes |
|---------|-----------|--------|---------|
| 1.0     | 2025-01-15| Platform Team | Initial release |
| 2.0     | 2025-06-01| Platform Team | Added Prompt Spec; expanded all templates with instructions and examples |

---

## Naming Conventions

- **Filenames:** Use uppercase with underscores: `ADR-0012.md`, `PRD-0042.md`,
  `AGENT_SPEC-003.md`
- **Directory structure:**
  ```
  docs/
  ├── adrs/          # Architecture Decision Records
  ├── rfcs/          # Request For Comments
  ├── prds/          # Product Requirements Documents
  ├── trds/          # Technical Requirements Documents
  ├── agents/        # Agent Specifications
  ├── prompts/       # Prompt Specifications
  └── subsystems/    # Subsystem Specifications
  ```
- **Cross-references:** Always use relative paths: `../../adrs/ADR-0012.md`
- **IDs:** Sequential, zero-padded to 4 digits (ADR-0001, ADR-0002, ...)

---

## Template Conventions

Each template follows these shared conventions:

1. **Header metadata table:** Every document starts with a table containing ID,
   title, status, date, and owner. This enables automated indexing and search.
2. **Section numbering:** Sections are numbered (1., 2., 3.) for precise
   cross-referencing in reviews and AI agent instructions.
3. **Instructions in italics:** Author-facing guidance is rendered in italics or
   as blockquotes. Delete instruction text when filling out the template.
4. **Placeholder brackets:** All placeholders use `{Curly braces}`. Replace the
   entire placeholder including braces with the actual content.
5. **Examples as guidance:** Each template includes one or more filled-in example
   sections. Adapt these to your context; do not leave example content in the
   final document unless it accurately represents your system.
6. **Status field:** Every document has a status line. Valid values are:
   `draft | review | approved | superseded | deprecated | rejected`.
7. **Version history:** Every document ends with a version history table.
   Increment the version for substantive changes, even before approval.
8. **RFC 2119 keywords:** MUST, MUST NOT, SHOULD, SHOULD NOT, MAY — use these
   keywords (uppercase) when expressing normative requirements. See the
   Subsystem Spec for a full guide.

---

## AI Agent Usage Guide

For AI agents consuming these templates:

- **Read the template first** before generating content. Templates contain
  instructions embedded in the document structure.
- **Preserve section numbering** — agents rely on it for navigation.
- **Cross-reference correctly** — use relative markdown links with the document
  ID in the link text, e.g., `[ADR-0012](./adrs/ADR-0012.md)`.
- **Maintain the metadata table** — it is machine-parseable.
- **Respect the template structure** — do not add new top-level sections without
  justification. Use subsections within existing sections if needed.
- **Template versions** — check the template version number at the bottom of each
  template before use. If the template has been updated since the document was
  created, update the document to match.

---

## Quick-Start Workflow

```
Need to make a decision?         → Write an ADR
Need feedback on a design?       → Write an RFC
Defining product requirements?   → Write a PRD
Specifying technical constraints?→ Write a TRD
Creating an AI agent?           → Write an Agent Spec
Documenting a prompt?           → Write a Prompt Spec
Defining subsystem boundaries?  → Write a Subsystem Spec
Not sure which to use?          → Start with an RFC to gather input
```

---

*Template index version 2.0 — Questions? Contact the Platform Team.*
