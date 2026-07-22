# PRD-XXXX: {Product Name}

> Product Requirements Document (PRD) defines what we are building and why.
> It aligns stakeholders, engineers, designers, and QA around a shared
> understanding of scope, priorities, and success criteria.

---

## Metadata

| Field           | Value                     |
|----------------|---------------------------|
| **PRD ID**      | PRD-XXXX                  |
| **Product Name**| {Name of feature / product} |
| **Status**      | draft \| review \| approved \| implemented |
| **Owner**       | {PM / EM name}            |
| **Stakeholders**| {Comma-separated list}    |
| **Target Release** | {Version or date}      |
| **Last Updated**| YYYY-MM-DD                |

---

## 1. Problem Statement

Describe the problem in one or two sentences. Then expand with evidence.

**One-liner:**
> {Current behaviour} makes it hard for {user segment} to {desired outcome}.

**Context:**
- Current state: {What happens today?}
- Pain points: {What is broken, slow, or missing?}
- Supporting data: {Metrics, quotes from user research, support ticket counts}
- Impact: {How many users affected? What is the revenue/cost impact?}

**Example:**
> Users cannot filter the order list by more than one status at a time, forcing them
> to run 3–5 separate searches to get a complete picture of their pending orders.
> Support tickets tagged "order search" have increased 40 % QoQ, and user session
> recordings show an average of 4.2 filter changes per order-management session.

---

## 2. Goals & Non-Goals

### 2.1 Goals

| Goal | Priority (P1–P3) | Success Criterion |
|------|-----------------|-------------------|
| Allow multi-select status filtering | P1 | Users can select ≥1 status; results update in < 500 ms |
| Save recently used filter combinations | P2 | Last 5 filter sets persisted per user |
| Expose filter as a shareable URL | P3 | Filter state encoded in query params |

**Priority conventions:**
- **P1 (Must):** Blocking; without this the feature has no value.
- **P2 (Should):** Important but can ship in a follow-up.
- **P3 (Nice-to-have):** Desirable but lower impact; parked or removed if time runs short.

### 2.2 Non-Goals

These are explicitly out of scope for this PRD:

- {e.g., Full-text search across order notes — deferred to Q4}
- {e.g., Mobile-native filter UI — this release is desktop-only}
- {e.g., Saved filter sharing between users — needs auth scope changes}

---

## 3. Users & Personas

### Primary Persona: {Name}

| Attribute   | Value |
|------------|-------|
| Role       | {e.g., Operations Manager} |
| Technical level | Low / Medium / High |
| Frequency of use | Daily |
| Key need   | {e.g., Find orders quickly by any combination of attributes} |

### Secondary Personas

- **{Persona B}:** {Brief description and key need}
- **{Persona C}:** {Brief description and key need}

---

## 4. Jobs To Be Done (JTBD)

| Job | Trigger | Desired Outcome |
|-----|---------|----------------|
| {e.g., Find a specific order by status + date + customer} | Customer calls asking about an order | Complete the call in < 2 minutes |
| {e.g., Get a weekly snapshot of pending orders} | Monday morning review | Export filtered list as CSV in one click |

---

## 5. Requirements (MoSCoW)

### 5.1 Must Have (MVP)

| ID     | Requirement | Acceptance Criteria |
|--------|-------------|-------------------|
| REQ-01 | User can select multiple status checkboxes | Clicking "Pending" + "Processing" shows orders matching either |
| REQ-02 | Results update without page reload | XHR call completes in < 500 ms p95 |
| REQ-03 | Clear all filters in one click | "Clear" button resets to unfiltered state |
| REQ-04 | Filter state reflected in URL | `/orders?status=pending,processing` round-trips correctly |

### 5.2 Should Have

| ID     | Requirement | Acceptance Criteria |
|--------|-------------|-------------------|
| REQ-05 | Recently used filters shown as chips | Last 5 distinct combinations shown above filter bar |
| REQ-06 | Keyboard navigation for filter list | Arrow keys + Enter operate filter checkboxes |

### 5.3 Could Have

| ID     | Requirement | Acceptance Criteria |
|--------|-------------|-------------------|
| REQ-07 | Drag-and-drop reorder of applied filters | Visual feedback on drag |

### 5.4 Won't Have (this release)

- Saved filter sets shared across team
- Natural-language filter input

---

## 6. User Stories

**Story 1: Multi-select filter**
> As an **operations manager**, I want to **select multiple order statuses at once**
> so that I can **see all orders that need attention without running separate searches**.

**Acceptance criteria:**
- [ ] Checkbox list of statuses shown in filter panel
- [ ] Selecting "Pending" shows only pending orders
- [ ] Selecting "Pending" + "Processing" shows orders in either status
- [ ] Count badge next to filter label shows total matching orders
- [ ] Deselecting all statuses reverts to default (all orders)

**Story 2: Shareable filter URL**
> As an **operations manager**, I want to **share a filtered view with my team**
> so that we can **collaborate on the same subset of orders**.

**Acceptance criteria:**
- [ ] URL updates as filters change
- [ ] Opening the URL reproduces the same filter state
- [ ] Invalid filter values gracefully fall back to unfiltered

---

## 7. Success Metrics

| Metric | Baseline | Target | Measurement Method |
|--------|----------|--------|-------------------|
| Time to find an order | 45 s | ≤ 15 s | RUM / session recording |
| Filter-related support tickets | 120 / month | ≤ 30 / month | Zendesk tag count |
| User satisfaction (CSAT) | 3.2 / 5 | ≥ 4.0 / 5 | In-app survey after 3 uses |
| API calls per order-management session | 8.2 | ≤ 4.0 | Datadog dashboard |

---

## 8. Deployment Architecture

> **Default: Local-First.** All services deploy on local infrastructure by default —
> SQLite for storage, filesystem for assets, Nine Router (localhost:20128/v1) for
> all model access. Cloud deployment is an opt-in override for specific scaling needs.

| Tier | Storage | Model Access | Networking |
|------|---------|-------------|------------|
| Development | SQLite + local filesystem | Nine Router → local inference (Ollama/vLLM) | localhost |
| Production (local) | SQLite WAL + filesystem | Nine Router → local inference | LAN / tailscale |
| Production (cloud) | SQLite + optional cloud replica | Nine Router → hybrid local/cloud | Internet + VPN |

## 9. Milestones

| Milestone | Date       | Deliverable |
|-----------|-----------|-------------|
| Design review | YYYY-MM-DD | Figma mockups approved |
| API ready | YYYY-MM-DD | `/api/orders/filters` endpoint deployed |
| Alpha (internal) | YYYY-MM-DD | Feature-flagged; 10 internal testers |
| Beta (5% users) | YYYY-MM-DD | Ramped to 5 % of traffic |
| GA | YYYY-MM-DD | 100 % rollout; monitoring dashboards live |

---

## 10. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Query performance degrades with multi-select | Medium | High | Index on status column; add Redis cache layer |
| Users find multi-select confusing | Low | Medium | Onboarding tooltip; A/B test label wording |

---

## 11. Version History

| Version | Date       | Author | Changes |
|---------|-----------|--------|---------|
| 0.1     | YYYY-MM-DD| {Name} | Initial draft |
| 0.2     | YYYY-MM-DD| {Name} | Added success metrics per stakeholder review |
| 1.0     | YYYY-MM-DD| {Name} | Approved |

---

*Template version 2.0 — See [README.md](./README.md) for PRD workflow guidance.*
