# Agent Memory

> **Domain:** Per-Agent Memory Model
> **Applies to:** Kernel, Agent Runtime, Persistent Memory
> **Last updated:** 2026-07-22

## Overview

Every agent in AI Dev OS has a four-tier memory system that balances context cost, recall speed, and persistence. The tiers form a hierarchy: data flows upward through consolidation and downward through retrieval.

```mermaid
flowchart TB
    subgraph TIER1["Tier 1: Working Memory"]
        WM[In-Context\nTransient\nBounded by ctx window]
    end
    subgraph TIER2["Tier 2: Short-Term Memory"]
        STM[Recent turns\nTool results\nObservations\nAuto-summarized]
    end
    subgraph TIER3["Tier 3: Long-Term Memory"]
        LTM[Persistent facts\nSummaries\nPatterns\nBacked by Persistent Memory]
    end
    subgraph TIER4["Tier 4: Shared Memory"]
        SM[Cross-agent notes\nPublished to SCE\nVisible to group]
    end

    WM -->|auto-summary| STM
    STM -->|consolidation| LTM
    LTM -->|publication| SM

    SM -->|retrieval| LTM
    LTM -->|recall| STM
    STM -->|context injection| WM
```

## Memory Tiers

### Working Memory (Tier 1)

- **Storage:** In-context — held in the agent's prompt window.
- **Lifetime:** Transient — lost when the turn ends unless explicitly captured.
- **Bound:** Limited to the model's context window (minus system prompt and tools).
- **Content:** Current task instructions, immediate observations, intermediate reasoning.
- **Persistence:** None unless promoted via `remember()`.

### Short-Term Memory (Tier 2)

- **Storage:** Recent interaction history stored in a ring buffer.
- **Lifetime:** Configurable TTL (default: 30 minutes), evicted by LRU.
- **Capacity:** Default 10,000 tokens of recent turns, tool results, and observations.
- **Auto-summarization:** When the buffer exceeds 70% capacity, `summarize()` produces a compressed digest. The digest replaces the oldest 50% of entries.
- **Indexing:** Each entry is tagged with a turn ID, timestamp, and importance score (0.0–1.0).

### Long-Term Memory (Tier 3)

- **Storage:** Persistent Memory (encrypted SQLite + vector index).
- **Lifetime:** Until explicitly deleted or consolidated away by retention policy.
- **Content:** Cross-session facts, learned patterns, agent summaries, skill definitions.
- **Backing store:** Persistent Memory — survives agent restarts and workspace restarts.
- **Retention policy:** Entries with importance < 0.3 are candidates for pruning after 7 days. Entries with importance > 0.8 are preserved indefinitely.

### Shared Memory (Tier 4)

- **Storage:** Published as named notes on the SCE event bus.
- **Visibility:** All agents in the same group or workspace (depending on the SCE topic scope).
- **Lifetime:** Until explicitly unpublished or the SCE topic is compacted.
- **Use cases:** Shared context (e.g., "database schema loaded by agent-alpha"), coordination markers ("phase-1 complete"), conflict markers ("file-X locked by agent-beta").

## Promotion Path

Data moves through the tiers via a consolidation pipeline:

```
Working Memory
    │  turn ends
    ▼
Short-Term Memory
    │  importance > 0.5 AND (buffer 70% full OR 5 min idle)
    ▼
Long-Term Memory
    │  published_to_scce = true AND group_visible
    ▼
Shared Memory
```

**Consolidation rules:**
1. Every turn end: the agent calls `consolidate()` which scans Working Memory for facts with importance > 0.5.
2. If Short-Term Memory exceeds 70% capacity: `summarize()` runs, creating a digest that is immediately promoted to Long-Term Memory.
3. Long-Term Memory entries tagged `shared: true` are automatically published to the SCE shared topic.
4. Entries with `ttl: <duration>` are evicted from Long-Term Memory after the TTL expires.

## Memory Retrieval

Retrieval is **scope-aware** and **escalating**:

```python
def recall(query, scope="local"):
    # 1. Check Working Memory (fast, exact match)
    results = working_memory.match(query)
    if results and len(results) >= scope.min_results:
        return results

    # 2. Check Short-Term Memory (recent context)
    results = short_term_memory.search(query)
    if results and len(results) >= scope.min_results:
        return results

    # 3. Check Long-Term Memory (semantic search)
    results = long_term_memory.search(query, top_k=10)
    if results and len(results) >= scope.min_results:
        return results

    # 4. Check Shared Memory (cross-agent)
    results = shared_memory.search(query, topic=scope.topic)
    return results
```

Each tier adds latency but expands recall breadth. The agent can set `scope.min_results` to tune the tradeoff.

## Memory Eviction

| Tier | Strategy | Default Limit |
|------|----------|---------------|
| Working Memory | LRU — oldest entries dropped when context window fills. | Model-dependent (typically 8K–128K tokens). |
| Short-Term Memory | TTL-based (entry age > max_ttl) + LRU (buffer full). | 10,000 tokens / 30 min TTL. |
| Long-Term Memory | Retention-based (importance < threshold + age > retention_period). | 100,000 entries / 7 day retention. |
| Shared Memory | Explicit unpublish or SCE topic compaction. | Unlimited (topic compaction by event count). |

## Interfaces

| Interface | Description |
|-----------|-------------|
| `remember(key, value, importance, ttl?)` | Store a fact in Working Memory (auto-promoted on turn end). |
| `recall(query, scope?)` | Retrieve matching facts (escalating tier search). |
| `forget(key_pattern)` | Remove facts matching the pattern from all tiers. |
| `consolidate()` | Manually trigger promotion from Working → Short-Term → Long-Term. |
| `summarize(buffer_id?)` | Generate a summary digest of Short-Term Memory entries. |
| `memory.stats()` | Return counts and token usage per tier. |
| `memory.clear(tier?)` | Clear a specific tier (or all tiers). |

## Usage Example

```typescript
// Agent stores a fact during a coding task
await agent.remember(
  "db_schema.users_table",
  "users table has columns: id (uuid), email (text), created_at (timestamptz)",
  { importance: 0.9, ttl: "7d" }
);

// Later in the same session, recall with escalating scope
const schema = await agent.recall("users table schema", { min_results: 1 });
// 1. Checks Working Memory (miss)
// 2. Checks Short-Term Memory (hit — just consolidated from working)
// 3. Returns immediately with recent context

// Across sessions, after consolidation to Long-Term Memory:
const schema = await agent.recall("users table schema", { min_results: 1 });
// 1. Working Memory: miss
// 2. Short-Term Memory: miss (evicted by TTL)
// 3. Long-Term Memory: hit (semantic search, score: 0.94)
// 4. Returns the persisted fact

// Agent publishes a shared note for sibling agents
await agent.remember(
  "phase.3.complete",
  "Database migration applied. Schema version now 42.",
  { importance: 0.8, shared: true, scope: "group" }
);
// This is automatically published to the SCE shared topic and visible to all agents in the group.
```

## Related Documents

| Document | Description |
|----------|-------------|
| [Persistent Memory](PERSISTENT_MEMORY.md) | Long-term storage backend, encryption, compaction |
| [Context Window Management](CONTEXT_WINDOW_MANAGEMENT.md) | Token budgeting, truncation, sliding window |
| [Knowledge System](KNOWLEDGE_SYSTEM.md) | Main KB, Global KB, FTS5 search |
| [Agent Lifecycle](AGENT_LIFECYCLE.md) | Agent creation, dispatch, teardown |
| [Embeddings](EMBEDDINGS.md) | Vector generation for semantic memory search |
