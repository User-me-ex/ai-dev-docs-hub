# Knowledge Graph

```mermaid
flowchart LR
  NOTE[Markdown Note] -- backlink --> NOTE2[Related Note]
  NOTE -- tag --> TAG((#topic))
  NOTE -- embed --> ASSET[Asset]
  NOTE --> INDEX[(Vector Index)]
  INDEX --> RAG[RAG Pipeline]
  RAG --> AGENT[Agent]
```
