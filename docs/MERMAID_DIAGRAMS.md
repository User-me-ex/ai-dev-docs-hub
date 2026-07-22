# Mermaid Diagrams

> Diagram conventions for the AI Dev OS documentation.

## Where Diagrams Live

- All source diagrams live under `diagrams/` as Markdown files with fenced ```` ```mermaid ```` blocks.
- Subsystem docs link to a diagram rather than inlining large diagrams.

## When to Use Mermaid vs ASCII

- **Mermaid** for flows, sequences, state machines, ERDs, and class diagrams.
- **ASCII** for tiny inline diagrams (2–5 nodes) where a fenced ```text``` block is clearer.

## Style

- One diagram per file where possible.
- Node IDs UPPER_SNAKE_CASE; labels sentence case.
- Prefer left-to-right flowcharts (`flowchart LR`) for pipelines.
- Colors reserved for status (red = failure path, green = happy path); do not use color for decoration.

## Index

- [Architecture](../diagrams/ARCHITECTURE.md)
- [AI Kernel](../diagrams/AI_KERNEL.md)
- [Context Engine](../diagrams/CONTEXT_ENGINE.md)
- [Data Flow](../diagrams/DATA_FLOW.md)
- [Deployment Topology](../diagrams/DEPLOYMENT_TOPOLOGY.md)
- [Knowledge Graph](../diagrams/KNOWLEDGE_GRAPH.md)
- [Nine Router Flow](../diagrams/NINE_ROUTER_FLOW.md)
