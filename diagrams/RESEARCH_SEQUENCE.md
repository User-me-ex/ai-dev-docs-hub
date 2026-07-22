# Research Engine Sequence

> Sequence diagram of the Research Engine performing a scheduled crawl with caching and citation tracking.

```mermaid
sequenceDiagram
    participant JS as Job Scheduler
    participant RE as Research Engine
    participant RC as Research Cache
    participant ADAPT as Source Adapter
    participant EXT as External Source
    participant CE as Citation Engine
    participant KB as Knowledge System
    participant SCE as Shared Context Engine

    JS->>RE: trigger_job(source_config)
    RE->>CE: check_freshness(source)
    CE-->>RE: {stale: true, last_checksum}

    RE->>RC: fetch(source.uri)
    RC->>RC: lookup in cache

    alt Cache miss or stale
        RC->>ADAPT: fetch(source.uri)
        ADAPT->>EXT: HTTP GET / API call
        EXT-->>ADAPT: response
        ADAPT-->>RC: raw_content + metadata
        RC->>RC: store(content, checksum, metadata)
    end

    RC-->>RE: content

    RE->>RE: parse content → structured data
    RE->>RE: deduplicate against existing KB entries
    RE->>RE: diff against previous version (if re-crawl)

    alt New content found
        RE->>CE: citation.create(source, capture_info)
        CE-->>RE: citation_id
        RE->>KB: write(citation_id, parsed_content, scope)
        KB-->>RE: kb_record_id
        RE->>SCE: publish("research.completed", {source, new_entries: 1, updated_entries: 0})
    else Content unchanged
        RE->>CE: update_freshness(source, checksum)
        RE->>SCE: publish("research.no_change", {source})
    end
```

## Related Documents

- [Research Engine](../docs/RESEARCH_ENGINE.md) — full pipeline spec
- [Research Cache](../docs/RESEARCH_CACHE.md) — caching layer
- [Citation Engine](../docs/CITATION_ENGINE.md) — provenance tracking
- [Source Ranking](../docs/SOURCE_RANKING.md) — source authority scoring
- [Knowledge System](../docs/KNOWLEDGE_SYSTEM.md) — KB record storage
