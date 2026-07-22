# UI/UX

> Specification for the desktop and web shell frontends — screen inventory, component tree, navigation model, design system tokens, and accessibility requirements. This document is normative — implementations MUST satisfy every MUST clause below.

## Overview

The UI/UX subsystem defines the visual and interactive surface of AI Dev OS across two shells: a **desktop shell** (Electron or Tauri wrapper) and a **web shell** (SPA served by the Backend). Both shells share a common component library, design tokens, and state management pattern.

The UI communicates with the backend exclusively through the [API Spec](./API_SPEC.md) — HTTP + WebSocket for CRUD and streaming, [IPC](./IPC.md) for local desktop calls. No component has direct access to the Kernel or subsystems.

## Goals

- Three-pane layout: **Runs** (left), **Context** (center), **Inspector** (right).
- Keyboard-first navigation: every action reachable via keyboard without a mouse.
- Nine Router surfaced as a first-class picker with search/filter/group-by/refresh.
- Real-time streaming: model tokens, SCE events, and log output appear without polling.
- Accessible: WCAG 2.1 AA minimum, AAA targeted for all primary workflows.
- Offline-capable: the UI functions with stale data when the backend is unreachable, with clear "offline" indicators.

## Non-Goals

- Implementation code — this repo is documentation-only ([AI Coding Rules](./AI_CODING_RULES.md)).
- Mobile-native apps — the web shell is responsive but a native mobile client is post-v1.0.
- Visual design mockups — those live in a design tool (Figma); this doc defines the contract.
- CSS framework choice — the design tokens are framework-agnostic.

## Screen Inventory

| Screen | Route | Shells | Purpose |
|--------|-------|--------|---------|
| **Runs** | `/runs` | desktop, web | List all runs, filter by state, drill into details |
| **Run Detail** | `/runs/:id` | desktop, web | Live streaming view: plan, events, artifacts, verdicts |
| **Plan** | `/runs/:id/plan` | desktop, web | TaskGraph visualization with dependency arrows |
| **Router** | `/router` | desktop, web | Nine Router model picker: search, filter, assign roles |
| **Groups** | `/groups` | desktop, web | AI Group catalog, member roles, health status |
| **Memory** | `/memory` | desktop, web | Persistent Memory browser and query interface |
| **Knowledge** | `/knowledge` | desktop, web | Four-tier KB browser with graph visualization |
| **Settings** | `/settings` | desktop, web | Configuration editor, provider credentials, plugins |
| **Graph** | `/graph` | desktop, web | Obsidian Graph Engine visualizer |
| **Voice** | (overlay) | desktop | Microphone indicator, VAD visualization, mute toggle |

## Layout

```
┌─────────────────────────────────────────────────────┐
│  Header: Logo | Search | Run button | Status | Menu │
├──────────┬──────────────────────────┬────────────────┤
│  Left     │     Center               │  Right         │
│  Panel    │     Panel                │  Panel         │
│           │                          │                │
│  Runs     │  Agent output /          │  Inspector:    │
│  list     │  streaming tokens        │  - Logs        │
│  Active   │  diff views              │  - Trace       │
│  History  │  plan graph              │  - Metrics     │
│  Queue    │                          │  - Events      │
│           │                          │                │
├──────────┴──────────────────────────┴────────────────┤
│  Status bar: correlation_id | budget | model | mode   │
└─────────────────────────────────────────────────────┘
```

### Layout Rules

- The left panel is collapsible (default 280px, min 200px, max 400px).
- The right panel is collapsible (default 320px, min 250px, max 500px).
- In "focus mode" (Ctrl+Shift+F), both side panels collapse, leaving only the center.
- The header is always visible. The status bar is always visible.
- Keyboard shortcut `Ctrl+\` toggles the left panel, `Ctrl+Shift+\` toggles the right panel.

## Component Tree

```
App
├── Header
│   ├── Logo
│   ├── SearchInput (Cmd+K palette)
│   ├── RunButton ("Run goal…")
│   ├── StatusIndicator (connected/disconnected/busy)
│   └── Menu (File, Edit, View, Help)
├── MainLayout
│   ├── LeftPanel
│   │   ├── RunsList
│   │   │   ├── RunCard (state, model, duration, preview)
│   │   │   └── RunFilter (state, model, date range)
│   │   ├── QueueView
│   │   │   └── QueueItem (position, priority, model)
│   │   └── GroupsList
│   │       └── GroupCard (name, members, health)
│   ├── CenterPanel
│   │   ├── RunDetail
│   │   │   ├── StreamingOutput (token display)
│   │   │   ├── PlanGraph (TaskGraph SVG)
│   │   │   ├── DiffView (side-by-side / unified)
│   │   │   └── ArtifactBrowser
│   │   ├── RouterPanel
│   │   │   ├── ModelSearch
│   │   │   ├── ModelTable
│   │   │   │   ├── ModelRow (name, provider, capabilities, status)
│   │   │   │   └── RoleBadge (assigned role)
│   │   │   └── AssignmentDrawer (role → model picker + fallback chain)
│   │   ├── MemoryPanel
│   │   │   ├── QueryInput
│   │   │   ├── ResultsList
│   │   │   └── RecordDetail
│   │   ├── KnowledgePanel
│   │   │   ├── TierSelector (Global/Main/Group/Individual)
│   │   │   └── GraphView
│   │   └── SettingsPanel
│   │       ├── ConfigEditor
│   │       ├── ProvidersList
│   │       └── PluginManager
│   └── RightPanel
│       ├── LogViewer (filtered, streaming, searchable)
│       ├── TraceView (span waterfall)
│       ├── MetricChart (subsystem picker + time range)
│       └── EventStream (SCE events, filterable)
└── StatusBar
    ├── CorrelationId (clickable → copy)
    ├── BudgetDisplay (tokens/wall/cost with progress bar)
    ├── ActiveModel (current model: role assignment)
    ├── ModeIndicator (online/offline)
    └── TelemetryIndicator (enabled/disabled)
```

## Design Tokens

All visual properties are expressed as design tokens — named values that map to CSS custom properties. Tokens are organized by category:

### Colors

```css
--color-bg-primary: #1a1a2e;        /* Dark background */
--color-bg-secondary: #16213e;       /* Panel backgrounds */
--color-bg-tertiary: #0f3460;        /* Active selection */
--color-text-primary: #e0e0e0;       /* Primary text */
--color-text-secondary: #a0a0b0;     /* Secondary / muted text */
--color-accent: #00d4ff;             /* Links, focus indicators */
--color-success: #00e676;            /* Run passed, connected */
--color-warning: #ffc107;            /* Warning states */
--color-error: #ff5252;              /* Errors, vetoes */
--color-border: #2a2a4a;             /* Panel borders */
--color-surface: #1e1e3a;            /* Cards, elevated surfaces */
```

### Typography

```css
--font-mono: 'JetBrains Mono', 'Fira Code', 'Cascadia Code', monospace;
--font-ui: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
--font-size-xs: 11px;
--font-size-sm: 13px;
--font-size-base: 14px;
--font-size-lg: 16px;
--font-size-xl: 20px;
--font-size-2xl: 24px;
--font-weight-normal: 400;
--font-weight-medium: 500;
--font-weight-bold: 600;
--line-height-tight: 1.25;
--line-height-normal: 1.5;
```

### Spacing

```css
--space-xs: 4px;
--space-sm: 8px;
--space-md: 16px;
--space-lg: 24px;
--space-xl: 32px;
--space-2xl: 48px;
```

All text rendered in the center and right panels uses `--font-mono`. UI chrome (headers, sidebars, labels) uses `--font-ui`.

## Navigation

| Shortcut | Action |
|----------|--------|
| `Cmd+K` / `Ctrl+K` | Command palette (search screens, actions) |
| `Cmd+Enter` | Submit current goal (from anywhere) |
| `Ctrl+Tab` | Cycle between open runs |
| `Ctrl+Shift+T` | Open Router panel |
| `Ctrl+Shift+M` | Open Memory panel |
| `Ctrl+Shift+K` | Open Knowledge panel |
| `Ctrl+Shift+,` | Open Settings |
| `Ctrl+\` | Toggle left panel |
| `Ctrl+Shift+\` | Toggle right panel |
| `Escape` | Close current panel / clear selection |
| `Ctrl+Shift+F` | Focus mode (hide both side panels) |

## State Management

The UI subscribes to three data sources:

1. **SCE events** (via WebSocket): real-time updates to runs, verdicts, logs, and metrics.
2. **API responses** (via HTTP): CRUD operations for settings, router assignments, groups.
3. **IPC calls** (desktop only): local file access, keychain read, plugin subprocess I/O.

State is managed as a single store (Zustand / Pinia / similar):

```typescript
Store {
  runs: Run[]
  activeRunId: string | null
  router: { models: Model[], assignments: RoleAssignment[] }
  groups: Group[]
  settings: Settings
  memory: { query: string, results: MemoryRecord[] }
  ui: { leftPanelOpen, rightPanelOpen, mode, theme }
}
```

The store is a write-through cache: mutations go to the API, the API confirms, and the store updates from the confirmation response or SCE event.

## Accessibility

- All interactive elements have visible focus indicators (minimum 2px outline, 3:1 contrast against adjacent color).
- All icons have `aria-label` or `aria-labelledby` attributes.
- All streaming content (token output, log streams) uses `aria-live="polite"` regions.
- The command palette supports full keyboard navigation (arrow keys, Enter, Escape).
- Color is never the sole indicator of state (e.g. run status is shown as text + icon + color).
- All panels are reachable via sequential keyboard navigation (Tab order follows visual order).
- Screen reader announcements are emitted for: run started/completed, Guardian veto, model assignment change.

## Performance Budget

| Metric | Target | Measurement |
|--------|--------|-------------|
| Time to interactive (cold) | < 3 s | Lighthouse |
| Input latency (keypress → visible) | < 50 ms | RAIL model |
| Token streaming latency (first token) | < 200 ms from backend emit | wall clock |
| Panel switch | < 100 ms | wall clock |
| Model search (1000 models) | < 200 ms | filter + render |
| Memory query results render (100 items) | < 300 ms | render |

## Error States

| State | Display | Action |
|-------|---------|--------|
| Backend disconnected | Yellow "Disconnected" badge in status bar + banner | Retry connection (auto with backoff) |
| API error (4xx/5xx) | Toast notification + inline error on affected panel | Retry or dismiss |
| Streaming interrupted | Streamed text preserved; "Resume" button shown | Click to resubscribe |
| Empty state (no runs) | Illustration + "Run your first goal" CTA | Click to focus goal input |
| Loading state | Skeleton placeholders (never spinners alone) | — |

## Failure Modes

| Mode | Detection | Response |
|------|-----------|----------|
| WebSocket reconnection loop | Server returns 101×4 timeout | Fall back to polling (GET /api/v1/runs?poll) with 2s interval |
| GPU-accelerated rendering crash | Canvas context lost | Fall back to software rendering; emit user-visible toast |
| Localisation file missing | `lang.json` not found | Fall back to `en-US`; log WARN |
| Plugin panel crash | iframe or component error | Isolate crashed panel (show "Plugin crashed" placeholder); log ERROR |
| Token display backlog | Accumulated tokens > 100k without render frame | Throttle render to 60fps; batch commit to DOM every 100ms |

## Security Considerations

- The UI never displays raw secrets, API keys, or tokens. All secret fields are masked (`********`) in the settings panel.
- The command palette does not expose system commands (no `aidevos server` commands, no file system access).
- The web shell enforces Content Security Policy headers served by the Backend.
- Plugin panels are sandboxed in iframes with `sandbox` attribute restrictions.

## Acceptance Criteria

- A user can complete a full run workflow without touching a mouse: `Cmd+K` → type goal → Enter → watch output → `Escape` → `Ctrl+Shift+T` to inspect router → `Ctrl+Shift+\` to view logs.
- The Router panel renders 200 models across 7 provider groups with search/filter in under 500ms from page load.
- A WebSocket disconnection and reconnection preserves the current run's streaming output (no tokens lost, no duplicate lines).
- `aria-label` audit (axe-core) passes with zero violations on every screen.
- The layout renders correctly at 1024×768 minimum viewport without horizontal scroll.

## Related Documents

- [Frontend](./FRONTEND.md) — implementation architecture, component library, build tooling
- [API Spec](./API_SPEC.md) — all endpoints consumed by the UI
- [IPC](./IPC.md) — local communication channel (desktop only)
- [Streaming Responses](./STREAMING_RESPONSES.md) — SSE/WebSocket event format
- [Voice System](./VOICE_SYSTEM.md) — Voice overlay UI
- [Accessibility](./MERMAID_DIAGRAMS.md) — diagram conventions
- [System Overview](./SYSTEM_OVERVIEW.md)
