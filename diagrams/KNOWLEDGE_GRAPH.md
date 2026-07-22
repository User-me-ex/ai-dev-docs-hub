# Knowledge Graph — Structure, Node Types, Edge Types, and Query Patterns

> How the Obsidian Graph Engine models the vault as a graph, and how the RAG Pipeline traverses it.

## Graph Structure

```mermaid
flowchart TB
  subgraph Vault["Markdown Vault"]
    DOC_A["docs/MAIN_AI_KERNEL.md\nkind: doc\ntags: [kernel, orchestration]"]
    DOC_B["docs/NINE_ROUTER.md\nkind: doc\ntags: [routing, models]"]
    DOC_C["docs/SHARED_CONTEXT_ENGINE.md\nkind: doc"]
    PROMPT_A["prompts/KERNEL_PROMPT.md\nkind: prompt"]
    DIAG_A["diagrams/AI_KERNEL.md\nkind: diagram"]
    KB_A["docs/knowledge-bases/GLOBAL_KB.md\nkind: kb_entry"]
  end

  DOC_A -- "wikilink\n[[NINE_ROUTER]]" --> DOC_B
  DOC_A -- "wikilink\n[[SHARED_CONTEXT_ENGINE]]" --> DOC_C
  DOC_B -- "backlink" --> DOC_A
  DOC_C -- "backlink" --> DOC_A
  DOC_A -- "embed\n![[diagrams/AI_KERNEL]]" --> DIAG_A
  DOC_A -- "tag: kernel" --> TAG_K((#kernel))
  PROMPT_A -- "tag: kernel" --> TAG_K
  TAG_K -- "shared tag" --> DOC_A
  TAG_K -- "shared tag" --> PROMPT_A

  DOC_A -. "semantic similarity 0.87" .-> DOC_B
  DOC_B -. "semantic similarity 0.74" .-> KB_A
```

## Node Types

```mermaid
flowchart LR
  subgraph NodeTypes["Node Types"]
    DOC[doc\nMain documentation files\n.md in docs/]
    PROMPT_N[prompt\nSystem / role prompts\n.md in prompts/]
    DIAG[diagram\nMermaid diagrams\n.md in diagrams/]
    TMPL[template\nReusable doc templates\n.md in templates/]
    KB[kb_entry\nKnowledge base entries\nany KB tier]
    PLAY[playbook\nStep-by-step procedures\ndocs/playbooks/]
  end
```

## Edge Types

```mermaid
flowchart LR
  subgraph EdgeTypes["Edge Types and Sources"]
    WL["wikilink\n[[target]] in source\nextracted by parser"]
    MD_L["md_link\n[text](path)\nextracted by parser"]
    EMB["embed\n![[target]]\nextracted by parser"]
    BL["backlink\ncomputed reverse\nof wikilink/md_link"]
    TAG_E["tag\nshared #hashtag\nboth nodes have same tag"]
    SEM["semantic\ncosine_sim > 0.75\ncomputed from embeddings"]
    PARENT["parent_dir\nfile is in subdirectory\nof another node's dir"]
  end
```

## Four-Tier KB Hierarchy

```mermaid
flowchart TB
  GLOBAL[Global KB\nSystem-wide constants\nshared across all workspaces]
  MAIN[Main KB\nProject-level knowledge\ncode patterns, decisions]
  GROUP[Group KB\nGroup playbooks, prompts\nshared within one Group]
  INDIVIDUAL[Individual KB\nAgent working memory\nper session]

  GLOBAL --> MAIN --> GROUP --> INDIVIDUAL

  QUERY[Knowledge query] --> INDIVIDUAL
  INDIVIDUAL -->|not found: escalate| GROUP
  GROUP -->|not found: escalate| MAIN
  MAIN -->|not found: escalate| GLOBAL
  GLOBAL -->|not found| MISS[Cache miss\nno result]
```

## RAG Pipeline — Graph-Augmented Retrieval

```mermaid
flowchart LR
  QUERY_TEXT[User query text] --> EMBED[Embed query\nnomic-embed-text]
  EMBED --> VEC_SEARCH[ANN search\nVector Store]
  VEC_SEARCH -->|top-K seed nodes| SEEDS[Seed node set]

  SEEDS --> GRAPH_EXP["graph.neighbors(seed, depth=1)\nExpand via wikilinks + backlinks"]
  GRAPH_EXP --> EXPANDED[Expanded node set]

  EXPANDED --> RANK["Rank nodes:\n0.6 * semantic_score\n+ 0.2 * backlink_count\n+ 0.2 * recency"]
  RANK --> TOP_N[Top-N nodes]
  TOP_N --> EXCERPT[Extract text excerpts\ncontext window slices]
  EXCERPT --> CONTEXT[Assembled RAG context\nfor model prompt]
```

## Graph Update Pipeline

```mermaid
flowchart LR
  FS[Filesystem event\ncreate / modify / delete] --> DEDUP[Deduplicate\ncoalesce buffer]
  DEDUP -->|debounce 200ms| PARSE[Parse document\nextract links, tags, aliases]

  PARSE --> DELTA[Compute delta\nadded / removed edges]
  DELTA --> DB_WRITE[Write to graph store\nSQLite adjacency tables]
  DB_WRITE --> VEC_QUEUE[Queue node\nfor embedding update]
  DB_WRITE --> SCE_EV[(SCE: graph.updates)]

  VEC_QUEUE --> EMBED_SVC[Embedding service\nasync backfill]
  EMBED_SVC --> VEC_IDX[(Vector index\nusearch)]
  VEC_IDX --> SEM_EDGES[Recompute\nsemantic edges\nfor updated node]
```

## Cypher-Like Query Examples

```
# All documents reachable from MAIN_AI_KERNEL.md within 2 wikilink hops
MATCH (n:doc)-[:wikilink*1..2]->(m) WHERE n.id = "docs/MAIN_AI_KERNEL.md" RETURN m

# Documents sharing any tag with NINE_ROUTER.md
MATCH (n)-[:tag]->(t)<-[:tag]-(m) WHERE n.id = "docs/NINE_ROUTER.md" AND n <> m RETURN m

# Top semantic neighbours of a document
MATCH (n)-[e:semantic]->(m) WHERE n.id = "docs/PERSISTENT_MEMORY.md" AND e.weight > 0.8
RETURN m, e.weight ORDER BY e.weight DESC LIMIT 10

# Shortest path between two documents
MATCH p = shortestPath((a)-[*]-(b))
WHERE a.id = "docs/CLI.md" AND b.id = "docs/NINE_ROUTER.md" RETURN p
```

## Related Documents

- [Obsidian Graph Engine](../docs/OBSIDIAN_GRAPH_ENGINE.md)
- [Knowledge System](../docs/KNOWLEDGE_SYSTEM.md)
- [RAG Pipeline](../docs/RAG_PIPELINE.md)
- [Vector Store](../docs/VECTOR_STORE.md)
- [Persistent Memory](../docs/PERSISTENT_MEMORY.md)
- [Research Engine](../docs/RESEARCH_ENGINE.md)
