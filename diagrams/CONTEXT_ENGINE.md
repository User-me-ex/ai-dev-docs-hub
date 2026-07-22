# Shared Context Engine — Architecture and Data Flow

> How events flow from publishers through the SCE broker to subscribers, snapshots, and the audit log.

## Broker Architecture

```mermaid
flowchart LR
  subgraph Publishers
    K[Kernel]
    W[Workers]
    RT[Nine Router]
    MM[Merge Manager]
    GD[Guardian]
    RES[Research Engine]
    MEM_SVC[Memory Service]
  end

  Publishers -- publish(topic, event) --> BROKER[SCE Broker]

  subgraph Broker_internals["SCE Broker internals"]
    AUTH[Publisher auth\nJWT / signed envelope]
    ACL[Topic ACL check]
    SEQ[Monotonic ULID\nsequencer per topic]
    SIGN[Broker signature\nper event]
    WRITE[Append to event log]
    SNAP[Snapshot updater\nasync]
    FAN[Fan-out to\nlive subscribers]
  end

  BROKER --> AUTH --> ACL --> SEQ --> SIGN --> WRITE
  WRITE --> SNAP
  WRITE --> FAN
  WRITE --> AUDIT[(Audit Log)]

  subgraph Storage
    LOG[(Event Log\nSQLite WAL\nor NATS JetStream)]
    SNAPSHOTS[(Snapshot Store\nlast-writer-wins\nper topic)]
    WAL[(Local WAL buffer\nwhen broker down)]
  end

  WRITE --> LOG
  SNAP --> SNAPSHOTS

  subgraph Subscribers
    CLI_SUB[CLI tail]
    UI_SUB[Web shell\nSSE stream]
    COST[Cost Management]
    RESEARCH_SUB[Research Engine\nreactive triggers]
    PLUGIN_SUB[Plugin middleware]
  end

  FAN -- AsyncIterator<Event> --> Subscribers
  LOG -- snapshot+tail on attach --> Subscribers
```

## Subscriber Attach Protocol

```mermaid
sequenceDiagram
  participant S as New Subscriber
  participant B as SCE Broker
  participant SNAP as Snapshot Store
  participant LOG as Event Log

  S->>B: subscribe(topic, from="snapshot")
  B->>SNAP: load_snapshot(topic)
  SNAP-->>B: { offset, state }
  B-->>S: snapshot { offset, state }
  B->>LOG: tail(topic, from=offset)
  LOG-->>B: events since offset (streaming)
  B-->>S: live event stream
  Note over S,B: Subscriber is now caught up\nand receiving new events
```

## Topic Namespace Conventions

```mermaid
flowchart LR
  TOPICS["SCE Topics"]
  TOPICS --> RUN["run.&lt;run_id&gt;\nKernel loop events per run\nRetention: 90d"]
  TOPICS --> ROUTER["router.assignments\nRole assignment changes\nRetention: forever, compacted"]
  TOPICS --> MODELS["models.discovery\nModel catalog updates\nRetention: 30d"]
  TOPICS --> GUARDIAN["guardian.verdicts\nArchitecture Guardian outputs\nRetention: forever"]
  TOPICS --> GROUP["group.&lt;group_id&gt;\nAI Group coordination\nRetention: 90d"]
  TOPICS --> MEMORY["memory.writes\nPersistent memory write log\nRetention: forever"]
  TOPICS --> MEMORY_DEL["memory.deletes\nDeletion + expiry log\nRetention: forever"]
  TOPICS --> MERGE["merge.txns\nMerge transaction events\nRetention: 90d"]
  TOPICS --> RESEARCH["research.jobs\nResearch Engine job events\nRetention: 30d"]
  TOPICS --> SCHEDULER["scheduler.events\nJob scheduler triggers\nRetention: 30d"]
  TOPICS --> QUEUE["queue.events\nQueueing backpressure signals\nRetention: 7d"]
  TOPICS --> VOICE["voice.events\nVoice session events (no audio)\nRetention: 7d"]
  TOPICS --> GRAPH["graph.updates\nObsidian graph mutations\nRetention: 30d"]
  TOPICS --> DLQ["queue.dlq\nDead-letter queue events\nRetention: 90d"]
```

## Backpressure and Flow Control

```mermaid
flowchart LR
  PUB[Publisher] -->|publish| BROKER[Broker]
  BROKER --> DEPTH{Topic depth\n> soft_cap?}
  DEPTH -->|no| ACK[200 OK\nevent accepted]
  DEPTH -->|yes| SLOW["429 Slow\nbackpressure signal"]
  SLOW --> PUB
  PUB --> THROTTLE[Kernel reduces\ndispatch rate]
```

## Failure and Recovery — Local WAL

```mermaid
sequenceDiagram
  participant P as Publisher (Kernel)
  participant B as SCE Broker
  participant WAL as Local WAL
  participant LOG as Event Log

  P->>B: publish(event)
  B-->>P: Connection refused (broker down)
  P->>WAL: buffer_to_wal(event)

  Note over B,WAL: Broker recovers

  WAL->>B: replay_wal(events_since_last_ack)
  B->>LOG: append(replayed events)
  B-->>WAL: ack(last_event_id)
  WAL->>WAL: clear replayed events
```

## Related Documents

- [Shared Context Engine](../docs/SHARED_CONTEXT_ENGINE.md)
- [Event Bus](../docs/EVENT_BUS.md)
- [Persistent Memory](../docs/PERSISTENT_MEMORY.md)
- [Audit Log](../docs/AUDIT_LOG.md)
- [Main AI Kernel](../docs/MAIN_AI_KERNEL.md)
