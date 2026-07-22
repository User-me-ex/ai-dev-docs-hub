# Research Engine Flow

> End-to-end pipeline from trigger through fetch, parse, deduplication, diff summarisation, and KB write.

## Full Research Pipeline

```mermaid
flowchart TB
  subgraph Triggers["Research Triggers"]
    CRON_T[Cron schedule\nJob Scheduler]
    DEMAND_T[On-demand\nresearch.enqueue]
    REACT_T[Reactive\nSCE: knowledge.stale event]
    CLI_T[CLI: aidevos research]
  end

  Triggers --> QUEUE[(Background Queue)]
  QUEUE --> DISPATCHER[Job Dispatcher]

  DISPATCHER --> FETCH

  subgraph FETCH["① Fetch"]
    HTTP_AD[HttpDocs adapter\nrespects robots.txt]
    GH_AD[GitHub adapter\nAPI v4 GraphQL]
    NPM_AD[npm adapter]
    ARXIV_AD[arXiv adapter]
    RSS_AD[RSS/Atom adapter]
    MCP_AD[MCP tool adapter]
    PLUGIN_AD[Plugin adapter\nPlugin SDK]
  end

  FETCH --> PARSE

  subgraph PARSE["② Parse → Markdown"]
    HTML_P[HTML → Markdown\nreadability extractor]
    JSON_P[JSON → structured MD\nGitHub API responses]
    XML_P[XML → MD\narXiv papers]
    RSS_P[RSS items → MD]
  end

  PARSE --> DEDUP

  subgraph DEDUP["③ Deduplicate"]
    EMBED_Q[Embed parsed content\nnomic-embed-text]
    SIM_CHK{cosine_sim\n> 0.95?}
    EMBED_Q --> SIM_CHK
    SIM_CHK -->|yes, unchanged| TOUCH[Update last_checked_at\nno KB write]
    SIM_CHK -->|no, new or changed| DIFF
  end

  subgraph DIFF["④ Diff Summariser"]
    PREV[Load previous\nsummary from KB]
    MODEL_CALL[Model call\nResearcher role]
    DIFF_SUM[Diff summary\n2–5 sentences]
    PREV & MODEL_CALL --> DIFF_SUM
  end

  DIFF --> PROV

  subgraph PROV["⑤ Provenance Tagger"]
    TAGS_ADD["Add tags:\nresearch_source:URL\nresearch_job:ID\nconfidence:NN"]
    REFS_ADD["Add refs:\n{ kind: research_artifact, id }"]
  end

  PROV --> WRITE

  subgraph WRITE["⑥ KB Writer"]
    UPSERT["memory.upsert(\nkind: research_result\nretention: 30d\n)"]
    GRAPH_UPD[Obsidian Graph\nnode update]
  end

  WRITE --> MEM[(Persistent Memory)]
  WRITE --> GRAPH[(Obsidian Graph Engine)]
  WRITE --> SCE_RES[(SCE: research.jobs\nresearch.completed)]
  WRITE --> KB_M[(Main KB or Group KB)]
```

## Deduplication Detail

```mermaid
flowchart LR
  EXISTING["Existing KB entry?\nmemory.list({ tags: ['research_source:URL'] })"]
  EXISTING -->|not found| NEW[NEW — write full summary]
  EXISTING -->|found| COS{cosine_sim\n(new_embed, existing_embed)\n> 0.95?}
  COS -->|yes| UNCHANGED[UNCHANGED\nupdate last_checked_at only\nno write]
  COS -->|no| CHANGED[CHANGED\nwrite diff_summary + new summary]
```

## Reactive Freshness

```mermaid
sequenceDiagram
  participant A  as Agent (Worker)
  participant SC as SCE
  participant RE as Research Engine
  participant Q  as Background Queue

  A->>SC: publish(knowledge.stale, { kb_entry_id, urgency: "high" })
  SC->>RE: knowledge.stale subscriber fires

  RE->>RE: lookup research_source tag on kb_entry
  RE->>RE: check last_checked_at vs min_recheck_interval

  alt urgency=high OR last_checked > min_recheck_interval
    RE->>Q: enqueue(research_job, priority=high)
    Q-->>RE: job queued
    RE->>SC: publish(research.jobs, { state: scheduled, triggered_by: stale_event })
  else too recent
    RE->>SC: publish(research.jobs, { state: skipped, reason: too_recent })
  end
```

## Source Adapter Interface

```mermaid
flowchart LR
  subgraph AdapterInterface["Source Adapter Interface"]
    FETCH_A["fetch(source) → FetchResult\n{ url, raw_content, content_type, fetched_at, status }"]
    PARSE_A["parse(fetch_result) → ParsedContent\n{ markdown, title, sections[] }"]
    META_A["metadata(source) → SourceMetadata\n{ canonical_url, last_modified?, etag? }"]
  end

  PLUGIN_SDK[Plugin SDK\nSourceAdapter extension point] --> AdapterInterface
  BUILT_IN[Built-in adapters\nHttp, GitHub, npm, arXiv, RSS, MCP] --> AdapterInterface
```

## Related Documents

- [Research Engine](../docs/RESEARCH_ENGINE.md)
- [Knowledge System](../docs/KNOWLEDGE_SYSTEM.md)
- [Persistent Memory](../docs/PERSISTENT_MEMORY.md)
- [Obsidian Graph Engine](../docs/OBSIDIAN_GRAPH_ENGINE.md)
- [Job Scheduler](../docs/JOB_SCHEDULER.md)
- [Internet Search](../docs/INTERNET_SEARCH.md)
- [Web Intelligence](../docs/WEB_INTELLIGENCE.md)
