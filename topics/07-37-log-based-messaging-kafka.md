# 37 — Log-based messaging (Kafka)

> Part 7 — Moving Data Between Systems · Prerequisites: topics 17, 23, 25, 30, 36

A classic message queue (topic 36) treats messages as transient: a message is handed to a consumer, acked
and deleted. That works well for "do this job once", but it makes some things impossible: a new consumer
cannot read yesterday's messages, a buggy consumer cannot rewind and reprocess, and ordering degrades as
soon as you add workers. **Log-based messaging** takes a different stance: messages are appended to a
durable, ordered log and are *not* deleted when read. Consumers simply remember how far they've read.
Apache Kafka is the best-known system built on this idea (others include Apache Pulsar, Redpanda, Amazon
Kinesis and Redis Streams). This topic explains the log model — topics, partitions, offsets, consumer
groups and retention — how it differs from a queue, the ordering guarantees it gives and where they
stop, and the internals that make it durable: in-sync-replica replication, leader election and log
compaction.

By the end of this topic you should be able to:
1. Describe topics, partitions, offsets, consumer groups and retention, and how they fit together.
2. Explain the fundamental difference between a queue and a log, and what the log enables (replay, many independent readers, ordering).
3. State Kafka's ordering guarantee precisely (per partition) and the things that can break it.
4. Explain how a partition is replicated: leader, followers, ISR, high-watermark, `acks` and `min.insync.replicas`.
5. Describe what happens when a partition leader fails, including unclean leader election.
6. Explain log compaction and when to use it instead of time-based retention.
7. Reason about consumer rebalancing, offset commits, and consumer lag.

---

## 1. The log as the abstraction

### 1.1 What "log" means here

Not an application log of text lines, but the data structure: an **append-only, totally ordered sequence
of records**, each identified by its position. You already met this idea as the database write-ahead log
(topic 17) and the replication log (topic 23). Kafka makes that log a first-class, shared product.

```
offset:   0      1      2      3      4      5      6      7  ...  (next write goes here)
        ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
        │ rec  │ rec  │ rec  │ rec  │ rec  │ rec  │ rec  │ rec  │ ◄── producers append
        └──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
                    ▲                         ▲
            consumer group "billing"   consumer group "search-indexer"
            is at offset 2             is at offset 5
```

- Writes only ever **append** to the end.
- Records are **immutable** once written.
- Each record gets a monotonically increasing **offset** within its partition.
- Readers don't remove anything. Each reader tracks its own **position** (offset).

### 1.2 Why this is fast

A log is the friendliest possible workload for storage (topic 02's sequential vs random I/O):

- Producers append sequentially to the end of a segment file.
- Consumers read sequentially from some offset — and recent data is usually still in the OS page cache.
- Kafka relies heavily on the page cache rather than its own cache, and uses zero-copy transfer
  (`sendfile`) to ship bytes from page cache to socket without copying through user space.
- Records are batched (and compressed per batch) end to end: producer batches, stored as batches,
  fetched as batches.

The result is that one broker can sustain hundreds of MB/s, and retaining data for days or weeks costs
only disk space — there is no per-message bookkeeping of "who has acked what".

---

## 2. Topics, partitions, offsets

### 2.1 Topics

A **topic** is a named stream of records of one kind: `orders`, `page-views`, `user-profile-changes`.
Records are key/value pairs with a timestamp and optional headers. The key is optional but, as we'll
see, central to ordering.

### 2.2 Partitions

A single log on one machine can't scale indefinitely, so a topic is split into **partitions** — each an
independent ordered log, stored and replicated on brokers. This is partitioning exactly as in topic 25,
applied to a log.

```
topic "orders" (3 partitions)

partition 0:  [0][1][2][3][4][5]          ◄─ leader on broker 1
partition 1:  [0][1][2][3]                ◄─ leader on broker 2
partition 2:  [0][1][2][3][4][5][6][7]    ◄─ leader on broker 3
```

How does a record choose a partition?
- **With a key:** `partition = hash(key) mod numPartitions` (Kafka's default uses murmur2). All records
  with the same key land in the same partition — and therefore stay in order relative to each other.
- **Without a key:** the producer spreads records across partitions (newer clients use a "sticky"
  partitioner that fills a batch for one partition before moving on, for better batching).
- **Custom partitioner:** you can supply your own function.

Note the consequence of `hash mod N` (topic 13): **changing the number of partitions remaps keys**. If
you go from 6 to 12 partitions, a key's future records may land in a different partition than its past
records, so per-key ordering across the change is broken. Pick partition counts with growth in mind.

Partitions are also the **unit of parallelism**: within one consumer group, at most one consumer reads
each partition. Twelve partitions means at most twelve useful consumers in a group.

### 2.3 Offsets

An **offset** is a record's position within its partition: a 64-bit integer, unique only *within* that
partition. "Offset 42" means nothing without saying which partition. A consumer's progress is therefore
a map: `{partition 0 → 1500, partition 1 → 980, partition 2 → 2210}`.

### 2.4 Segments and retention

Physically, each partition is a directory of **segment files** (e.g. 1 GB each) plus index files mapping
offsets and timestamps to byte positions. Only the newest segment (the *active* segment) is written.

**Retention** decides when old data is removed — by deleting whole old segments, never individual records:
- **Time-based:** keep 7 days (`retention.ms`).
- **Size-based:** keep at most N bytes per partition (`retention.bytes`).
- **Compaction:** keep the latest record per key (section 6).
- **Infinite / tiered:** some deployments keep data forever, offloading old segments to object storage
  (tiered storage) so brokers' local disks hold only recent data.

Retention is *independent of consumption*. A record is deleted when it ages out, whether or not anyone
read it. A consumer that falls behind by more than the retention window will find its offset no longer
exists and must reset (to the earliest or latest available offset, per `auto.offset.reset`) — data loss
from that consumer's point of view.

---

## 3. Consumer groups

### 3.1 The model

A **consumer group** is a set of consumers that cooperatively read a topic, identified by a `group.id`.
Kafka assigns each partition to **exactly one** consumer in the group.

```
topic "orders": P0 P1 P2 P3 P4 P5

group "billing" (3 consumers)            group "search-indexer" (2 consumers)
  c1 ◄─ P0, P1                             s1 ◄─ P0, P1, P2
  c2 ◄─ P2, P3                             s2 ◄─ P3, P4, P5
  c3 ◄─ P4, P5
```

Two properties follow:
- **Within a group:** partitions are divided up, so the group behaves like competing consumers
  (topic 36) — each record processed by one member — but with *partition-level* granularity.
- **Across groups:** every group reads every record independently. That's pub/sub for free: adding a
  new consumer group doesn't affect any existing one and doesn't require any change by producers.

If a group has more consumers than partitions, the extras sit idle. If it has fewer, some consumers
read several partitions.

### 3.2 Committing offsets

A consumer periodically **commits** its position: "for group billing, partition 3, the next offset to
read is 1501". Kafka stores commits in an internal compacted topic (`__consumer_offsets`). On restart,
or when a partition moves to another consumer, reading resumes from the last committed offset.

The order of "process" vs "commit" determines delivery semantics — exactly the ack question of topic 36:

```
commit, then process:   crash after commit → records skipped        ⇒ at-most-once
process, then commit:   crash before commit → records re-processed  ⇒ at-least-once
```

Auto-commit (`enable.auto.commit=true`) commits periodically in the background, which may commit offsets
for records you haven't finished processing yet (if your processing is asynchronous) — be deliberate
about it. Topic 38 covers how Kafka's transactions extend this to exactly-once for read-process-write
pipelines.

A neat property of the log: committing is cheap. It's one number per partition, not one ack per
message. The flip side: you can't ack individual records out of order. If record 100 fails and 101–150
succeed, committing 151 also "acks" 100. Consumers that need per-record retry move failures to a
separate retry topic or DLQ (topic 38).

### 3.3 Rebalancing

When consumers join, leave or crash (detected via heartbeats and `session.timeout.ms`, or a consumer
exceeding `max.poll.interval.ms` between polls), the group **rebalances**: partitions are reassigned. A
group coordinator broker manages membership; one consumer (the leader) computes the assignment.

Rebalances have costs:
- In the classic *eager* protocol, all consumers stop and give up all partitions during the rebalance —
  a "stop the world" pause.
- Records processed since the last commit by a consumer that lost a partition will be re-processed by
  the new owner (duplicates).
- Per-partition local state (caches, aggregates) must move.

Improvements: **cooperative (incremental) rebalancing** only moves the partitions that need to move;
**static membership** (`group.instance.id`) lets a consumer restart quickly without triggering a
rebalance. Newer Kafka versions move assignment logic to the broker side (the "next generation"
consumer rebalance protocol).

A consumer stuck processing a slow batch for longer than `max.poll.interval.ms` is considered dead and
kicked out — then rejoins, triggering *another* rebalance. Slow processing can thus produce rebalance
storms; keep per-poll batches small enough to finish in time.

### 3.4 Consumer lag

**Lag** = (latest offset in partition) − (consumer group's committed offset). It's the log equivalent of
queue depth (topic 36) and the primary health metric for consumers. Monitor it per partition: one hot
partition with growing lag can hide behind a healthy average.

---

## 4. Queue vs log: the core distinction

### 4.1 Side by side

| | Classic queue (RabbitMQ, SQS) | Log (Kafka, Kinesis, Pulsar) |
|---|---|---|
| What happens on read | Message is leased, then deleted on ack | Nothing; reader advances its offset |
| Retention | Until acked | By time/size/compaction, independent of reads |
| Multiple independent readers | Need a copy per subscriber | Free: one consumer group per reader |
| Replay / rewind | Impossible once acked | Reset offset to any retained position |
| Unit of parallelism | Message — any worker takes any message | Partition — one consumer per partition per group |
| Ordering | Lost with competing consumers | Guaranteed within a partition |
| Per-message ack / retry | Yes, natively | No; commit a position; failures need separate handling |
| Head-of-line blocking | Rarely — others take next message | Yes — a slow record holds up its partition |
| Per-message routing, priority, delay | Rich (exchanges, priorities, TTL) | Minimal; topics + keys |
| Throughput | Good | Very high (sequential I/O, batching) |

### 4.2 What the log enables

**Replay.** Because records stay, a consumer can rewind: deploy a fixed version of a buggy consumer, reset
its group's offsets to yesterday, and reprocess. Rebuild a search index from scratch by reading the
topic from the beginning (with compaction, the beginning is the full current state; see section 6).

**Many independent readers.** Billing, search, analytics, fraud detection and a data-lake loader all
read the same `orders` topic at their own pace. A slow analytics consumer does not slow billing. The
producer has no idea any of them exist.

**Ordering by key.** All events for order #123 go to one partition, read by one consumer, in order —
while thousands of other orders are processed in parallel in other partitions.

**A shared source of truth for derived data.** The log becomes the integration backbone: databases emit
changes into it (topic 39), stream processors transform it (topic 41), and downstream stores
(caches, search, warehouses) are all derived from it. This idea — "the log as the unifying abstraction"
— underlies event sourcing (topic 42) as well.

### 4.3 What the log costs

- **Head-of-line blocking.** A consumer processes a partition sequentially. One slow or failing record
  stalls everything behind it in that partition. (Contrast topic 36, where other workers keep going.)
- **Parallelism is capped by partition count**, which is awkward to change later.
- **No per-message features** like selective ack, priority or delay; these are built on top with extra
  topics.
- **Storage** grows with retention, not with backlog.

### 4.4 Choosing

A rough guide:
- Job/task distribution where each message is independent, may take varying time, and needs per-message
  retry/priority/delay → **queue**.
- Event streams with many consumers, high throughput, ordering per key, or replay/history → **log**.

Many systems use both: events flow through Kafka; a consumer turns some events into tasks on a work queue.

---

## 5. Ordering guarantees and their limits

### 5.1 The guarantee

**Kafka guarantees order within a partition only.** Records in one partition are appended in the order
the leader receives them and are read in offset order. There is no ordering across partitions — a
consumer reading P0 and P1 interleaves them arbitrarily, and two consumer groups may see cross-partition
events in different relative orders.

So "ordered by key" holds *if* all of a key's records go to the same partition. If you need a global
total order across all events, you need a single partition — capping throughput at what one partition
leader and one consumer can handle.

### 5.2 Things that can break per-key ordering

1. **Changing the partition count** (section 2.2) remaps keys.
2. **Producer retries with multiple in-flight requests.** Suppose the producer sends batch A then batch B
   to the same partition without waiting. A fails transiently, B succeeds, A is retried and lands *after*
   B. Fix: the **idempotent producer** (`enable.idempotence=true`, default in modern clients) — each
   producer has an ID and per-partition sequence numbers; the broker rejects out-of-sequence or
   duplicate batches, preserving order with up to 5 in-flight requests. (Older advice was
   `max.in.flight.requests.per.connection=1`.)
3. **Consumer-side parallelism.** If a consumer hands records from one partition to a thread pool, it
   has reintroduced competing consumers and lost ordering. Parallelise *by key* inside the consumer if
   needed.
4. **Retry topics.** Moving a failed record to a retry topic and continuing means later records for the
   same key are processed before the retried one.
5. **Multiple producers for the same key.** Order in the partition is arrival order at the leader, which
   may not be the order events happened in the world (topic 27). If two services write updates for the
   same entity, the log order is just "whoever got there first".
6. **Unclean leader election** (section 7.4) can drop records, creating gaps.

### 5.3 Hot partitions

Keying by `customer_id` puts all of a huge customer's events in one partition — the hot-key problem from
topic 25. One partition's consumer becomes the bottleneck and its lag grows. Options: a finer key (if
ordering isn't needed at customer level), key salting (splitting a key across N partitions and accepting
order only within each sub-key), or a dedicated topic for the outlier.

---

## 6. Log compaction

### 6.1 The idea

Time-based retention throws away old records regardless of content. For some topics, what you want is
**"the latest value for every key, forever"** — like a table. That is **log compaction**
(`cleanup.policy=compact`).

```
before compaction (key:value)
offset: 0      1      2      3      4      5      6      7
       [A:1]  [B:1]  [A:2]  [C:1]  [B:2]  [A:3]  [C:null] [D:1]

after compaction (A:1, A:2, B:1, C:1 removed — each superseded by a later value for its key)
offset: 4      5      6         7
       [B:2]  [A:3]  [C:null]  [D:1]
       (offsets are preserved; gaps 0–3 are fine)
```

- The log cleaner periodically rewrites old segments keeping only the **latest record per key**.
- **Offsets are preserved**; compacted logs have gaps in offsets.
- The **active segment** is never compacted, so recent duplicates remain until it rolls.
- A record with a **null value** is a **tombstone**: it means "delete this key". Tombstones are kept for
  a while (`delete.retention.ms`) so consumers can see the delete, then removed.

### 6.2 What it's for

- **Changelog / table topics:** a compacted topic of `user_id → latest profile` is a complete,
  replayable snapshot. A new service reads it from offset 0 and ends up with the full current state,
  without the topic growing with every update ever made. (This is the "stream–table duality" used by
  stream processors, topic 41.)
- **CDC streams** (topic 39): row-level changes keyed by primary key; compaction keeps current rows.
- **Kafka's own metadata**: `__consumer_offsets` is compacted (latest offset per group/partition).
- **State-store backups** for stream processors (Kafka Streams changelogs).

Compaction doesn't guarantee only one record per key at any moment; consumers must treat it as "at least
the latest value per key, possibly with some older ones".

---

## 7. Kafka internals: replication and leader election

### 7.1 Replicas, leaders, followers

Each partition has a **replication factor** (commonly 3). One replica is the **leader**; the others are
**followers**. This is single-leader replication (topic 23) applied per partition:

- Producers write to the partition leader.
- Followers continuously *fetch* from the leader (pull-based replication), appending to their own copy.
- Consumers read from the leader by default (follower fetching is possible for rack-local reads).

Leadership for different partitions is spread across brokers, so every broker is the leader for some
partitions and a follower for others — load is balanced.

### 7.2 The in-sync replica set (ISR)

The leader tracks which followers are **in sync** — caught up to within `replica.lag.time.max.ms`
(default 30 s) of the leader's log end. That set, including the leader, is the **ISR**.

```
partition orders-2, replication factor 3

leader (broker 3):    [0 .. 1000]          log end offset = 1001
follower (broker 1):  [0 .. 1000]          in ISR
follower (broker 5):  [0 ..  640]          fell behind > 30 s → removed from ISR

ISR = {3, 1}
```

The ISR is dynamic. A slow or dead follower is dropped; when it catches up, it rejoins. Contrast with a
fixed-quorum system (topic 24, 30): Kafka doesn't need a majority, it needs *the current ISR*.

### 7.3 Durability: acks, min.insync.replicas, high-watermark

The producer's `acks` setting says when a write is acknowledged:

| `acks` | Ack when | Risk |
|---|---|---|
| `0` | Immediately, no response awaited | Loss on any failure |
| `1` | Leader has written to its log | Loss if leader dies before followers copy |
| `all` (`-1`) | All replicas currently in the ISR have the record | Durable as long as one ISR member survives |

`acks=all` alone has a gap: if the ISR has shrunk to just the leader, "all ISR" means "just the leader".
So pair it with the topic setting **`min.insync.replicas`** (e.g. 2 with replication factor 3): if fewer
than 2 replicas are in sync, the leader *rejects* writes (`NotEnoughReplicas`) rather than accepting
weakly durable ones. RF=3, min ISR=2, acks=all is the standard "don't lose data" configuration: it
tolerates one broker failure without losing availability or acknowledged writes.

The **high-watermark (HW)** is the highest offset replicated to all ISR members. Records above the HW
exist on the leader but aren't yet "committed". **Consumers can read only up to the HW**, so they never
see a record that could later vanish in a failover.

```
leader log:    0 1 2 3 4 5 6 7 8
follower A:    0 1 2 3 4 5 6
follower B:    0 1 2 3 4 5
                         ▲
                 high-watermark = 6 (offsets 0..5 committed; consumers see only these)
```

(Precisely, the HW is the offset of the first record *not* yet replicated everywhere in the ISR.)

### 7.4 Leader election

When a partition leader's broker fails:

1. The **controller** notices. (Historically the controller was a broker elected via ZooKeeper, and
   cluster metadata lived in ZooKeeper — topic 31. Since Kafka 3.x, **KRaft** mode replaces ZooKeeper
   with a Raft-based quorum of controller nodes storing metadata in an internal log — topic 30. ZooKeeper
   mode was removed in Kafka 4.0.)
2. It picks a new leader **from the ISR** — any ISR member has every committed record.
3. It bumps the partition's **leader epoch** and tells brokers; producers and consumers refresh metadata
   and redirect.
4. Followers truncate any uncommitted suffix that diverges from the new leader (using the leader epoch
   to find the divergence point) and resume fetching. Records above the old HW that weren't replicated
   may be discarded — but they were never acknowledged under `acks=all`, so no acknowledged write is lost.

Leader epochs play the same role as Raft's terms or fencing tokens (topics 30, 31): a deposed leader
that comes back can't confuse anyone because its epoch is stale.

**Unclean leader election.** What if *every* ISR member is dead and only an out-of-sync replica
survives? Two choices:
- Wait for an ISR member to come back: **consistent but unavailable** (default:
  `unclean.leader.election.enable=false`).
- Elect the out-of-sync replica: **available but loses committed data** — and the data loss also causes
  offsets to be reused, confusing consumers.

This is the CAP/PACELC trade-off (topic 29) made into a config flag.

### 7.5 Putting the write path together

```
producer (acks=all, idempotent)
   │ batch for orders-2, seq=17
   ▼
leader broker 3: append to active segment (page cache)
   │
   ├──► follower broker 1 fetches → appends → next fetch reports its offset
   └──► follower broker 5 fetches → appends
   │
leader sees all ISR at ≥ offset → advances HW → responds to producer "ok"
   │
consumers fetching from orders-2 may now read the batch
```

Note Kafka does not fsync every write by default; durability comes from **replication across machines**
(ideally across racks/AZs) rather than per-write fsync on one disk. The bet is that several replicas
won't all lose power before flushing.

### 7.6 Exactly-once hooks (preview)

Kafka adds two mechanisms used by topic 38: the **idempotent producer** (dedupe retries via producer ID
+ sequence numbers) and **transactions** (atomically write to several partitions *and* commit consumer
offsets; consumers with `isolation.level=read_committed` skip aborted records). Together these give
exactly-once *within Kafka* for read-process-write pipelines.

---

## 8. Other log-based systems

- **Amazon Kinesis Data Streams:** managed; "shards" instead of partitions; retention 24 h by default
  (extendable up to 365 days); per-shard throughput limits.
- **Apache Pulsar:** separates serving (brokers) from storage (Apache BookKeeper), supports both
  streaming and queue-style per-message acks on the same topic, built-in tiered storage.
- **Redpanda:** Kafka-API-compatible, written in C++, uses Raft per partition instead of ISR.
- **Redis Streams:** an append-only stream type in Redis with consumer groups and per-message acks
  (a hybrid of log and queue), bounded by memory.

The concepts — partitioned append-only logs, offsets, groups, retention — carry over.

---

## 9. How it connects

- **Topic 17 (WAL):** the same append-only log idea, used as a database's durability mechanism.
- **Topic 23 (single-leader replication):** each partition is a single-leader replicated log; ISR is a
  dynamic flavour of semi-synchronous replication.
- **Topic 25 (partitioning):** key-hash partitioning, hot keys, and the pain of changing N.
- **Topic 27 (ordering):** the log gives a total order per partition — and nothing across partitions.
- **Topic 29/30/31:** unclean election is a CAP choice; KRaft uses Raft; leader epochs are fencing tokens.
- **Topic 36 (queues):** the contrast case.
- **Topic 38 (delivery semantics):** offsets commit order, idempotent producer, transactions.
- **Topic 39 (CDC):** databases stream their changes into Kafka topics, often compacted.
- **Topic 41 (stream processing):** Kafka is the usual input/output of stream processors.
- **Topic 42 (event sourcing):** the log as the source of truth.

---

## Common misconceptions

- **"Kafka is just a faster queue."** → It's a replicated, retained log; consumption doesn't delete. That
  enables replay and many independent readers but removes per-message ack, priority and delay.
- **"Kafka guarantees ordering."** → Only within a partition. Across partitions there's no order.
- **"More consumers = more throughput."** → Only up to the number of partitions; extra consumers in a
  group sit idle.
- **"acks=all means written to all replicas."** → It means all replicas *currently in the ISR*, which may
  be just the leader unless `min.insync.replicas` is set.
- **"Data is kept until consumed."** → Retention is time/size/compaction based. A consumer that falls
  behind the retention window loses data.
- **"Compaction keeps exactly one record per key."** → It keeps at least the latest; the active segment
  and not-yet-cleaned segments can hold older values.
- **"Adding partitions later is a harmless scaling step."** → It remaps keys, breaking per-key ordering
  across the change and moving state for stateful consumers.
- **"Kafka is durable because it fsyncs."** → By default it relies on replication across machines, not
  per-write fsync.

## Further reading (supplements)

- Jay Kreps, "The Log: What every software engineer should know about real-time data's unifying
  abstraction" (LinkedIn Engineering blog, 2013).
- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 11, "Partitioned Logs" and
  "Databases and Streams".
- Kreps, Narkhede, Rao, "Kafka: a Distributed Messaging System for Log Processing" (NetDB 2011).
- Apache Kafka documentation: "Design" section (persistence, efficiency, replication, log compaction).
- KIP-500 (replace ZooKeeper with a self-managed metadata quorum) for KRaft background.

---

## Questions

1. Why can a Kafka topic retain a week of data cheaply, while a RabbitMQ queue with a week of backlog is
   a problem?
2. A topic has 8 partitions. You run a consumer group with 12 instances. How many are doing work, and
   what would you change to use all 12?
3. A service consumes `orders`, and a bug corrupts its output for the last 6 hours. How does the log
   model help you recover, and what precondition must hold?
4. Events for the same order must be processed in order. What must the producer do, and name two things
   that could still break the ordering.
5. You increase partitions from 6 to 24 on a keyed topic. What happens to per-key ordering, and why?
6. A consumer commits offsets before processing each batch. It crashes mid-batch. What happens to the
   unprocessed records? Which semantics is that?
7. Your consumer takes up to 10 minutes per batch, and `max.poll.interval.ms` is 5 minutes. Describe the
   failure pattern you'd observe.
8. Replication factor 3, `acks=all`, `min.insync.replicas=1`. Two followers fall out of the ISR, then the
   leader's disk dies. What happened to writes acknowledged meanwhile? How should the config change?
9. Why can consumers only read up to the high-watermark rather than the leader's log end?
10. All ISR replicas for a partition are down; one stale replica is alive. What are the two options and
    what does each cost?
11. You need a topic from which any new service can bootstrap the current profile of every user. Which
    cleanup policy fits, and how are deleted users represented?
12. One customer produces 40% of all events, and you key by customer ID. What symptom appears, and what
    are the options?
13. When would you still choose a classic queue over Kafka?
14. A consumer hands each record of a partition to a 16-thread pool for speed. What guarantee has it
    lost, and how could it parallelise without losing it?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Kafka stores records in sequential segment files and tracks only one offset per partition per
group; retention is deleting old segment files. Old data costs only disk. A classic broker tracks
per-message state and is optimised for short queues; long queues cause memory pressure, slow
restarts and degraded performance.

**2.** Only 8 — each partition goes to at most one consumer in the group; 4 sit idle as hot spares. To
use 12, the topic needs at least 12 partitions (with the key-remapping caveat).

**3.** Fix the bug, reset the consumer group's offsets to before the bug (by timestamp), and reprocess;
outputs must be overwritable/idempotent. Precondition: the needed records are still within retention
(and the consumer's writes can be safely redone).

**4.** Use the order ID as the record key so all its events go to one partition (and use the idempotent
producer). Breakers: changing partition count; retries with multiple in-flight requests without
idempotence; consumer-side multithreading; retry topics; multiple producers racing for the same key.

**5.** `hash(key) mod 24` differs from `mod 6`, so many keys now map to new partitions. New records for a
key go to a different partition than old ones; a consumer may process the new ones before the old ones
are drained, so ordering across the change isn't guaranteed.

**6.** The committed offset is already past them; on restart, the consumer (or its replacement) starts
after them, so they are never processed — at-most-once.

**7.** The consumer exceeds the poll interval, is considered dead and removed from the group; a
rebalance reassigns its partitions; its uncommitted work is redone by another consumer; it then rejoins,
triggering another rebalance. You see repeated rebalances, duplicates and growing lag. Fix: smaller
batches (`max.poll.records`), faster processing, or longer interval.

**8.** With ISR = {leader}, acks=all is satisfied by the leader alone, so writes were acknowledged
with a single copy; when its disk dies they're lost. Set `min.insync.replicas=2` so the leader refuses
writes when fewer than 2 replicas are in sync — trading availability for durability.

**9.** Records above the HW aren't on all ISR replicas yet; if the leader fails, the new leader may not
have them and they'd be truncated. A consumer that had read them would have seen data that then
"disappears" — so reads stop at committed data.

**10.** Wait for an ISR member to return (no data loss, partition unavailable meanwhile), or allow an
unclean election of the stale replica (available immediately, but committed records it lacks are lost
and offsets get reused). Consistency vs availability.

**11.** A compacted topic keyed by user ID. The latest profile per key survives; a new service reads
from the beginning. Deleted users are tombstones — a record with the key and a null value, retained for
`delete.retention.ms` so consumers see it, then removed.

**12.** A hot partition: its consumer can't keep up, lag grows on that partition only, while others are
idle. Options: key more finely (e.g. customer + order if per-customer order isn't required), salt the
key across several partitions (accepting order only within sub-keys), or isolate the big customer to
its own topic with more partitions.

**13.** For task distribution where messages are independent, take variable time, and need per-message
ack/retry, priority or delay, without needing replay or many readers — e.g. image resize jobs, emails.
Also when head-of-line blocking per partition would hurt.

**14.** Per-partition (per-key) ordering, and safe offset commits (a later offset may complete before
an earlier one fails). Parallelise by key: hash each record's key to one of 16 single-threaded lanes, and
commit only up to the lowest offset completed across lanes.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What is a log in this context?
**A:** An append-only, totally ordered, immutable sequence of records addressed by offset.

**Q:** What is a partition?
**A:** An independent ordered log; a topic is split into partitions for scale and parallelism.

**Q:** How is a keyed record's partition chosen?
**A:** hash(key) mod number-of-partitions.

**Q:** What is an offset?
**A:** A record's position within one partition — unique only within that partition.

**Q:** What does retention depend on?
**A:** Time, size, or compaction — not on whether records were consumed.

**Q:** What is a consumer group?
**A:** Consumers sharing a group.id; each partition is assigned to exactly one member.

**Q:** How do you get pub/sub in Kafka?
**A:** Use multiple consumer groups; each reads every record independently.

**Q:** Max useful consumers in a group?
**A:** The number of partitions.

**Q:** Commit-then-process vs process-then-commit?
**A:** At-most-once vs at-least-once.

**Q:** What is consumer lag?
**A:** Latest offset minus the group's committed offset, per partition.

**Q:** What triggers a rebalance?
**A:** Consumers joining/leaving, missed heartbeats, or exceeding max.poll.interval.ms.

**Q:** Core difference between a queue and a log?
**A:** Queue deletes on ack; log retains and each reader tracks its own position.

**Q:** What does the log enable that a queue can't?
**A:** Replay, many independent readers, per-key ordering at scale.

**Q:** Kafka's ordering guarantee?
**A:** Within a partition only.

**Q:** Why does changing partition count break ordering?
**A:** hash mod N changes, so keys move to different partitions.

**Q:** What does the idempotent producer do?
**A:** Producer ID + per-partition sequence numbers let the broker drop duplicate/out-of-order retries.

**Q:** Log-specific downside vs a queue?
**A:** Head-of-line blocking within a partition; no per-message ack/priority/delay.

**Q:** What is log compaction?
**A:** Keep at least the latest record per key; older values for that key are removed.

**Q:** What is a tombstone?
**A:** A record with a null value — deletes the key in a compacted topic.

**Q:** What is the ISR?
**A:** In-sync replicas: leader plus followers caught up within replica.lag.time.max.ms.

**Q:** acks=all means?
**A:** Acknowledged once all current ISR members have the record.

**Q:** Why set min.insync.replicas=2?
**A:** So acks=all can't be satisfied by the leader alone; writes are rejected if ISR is too small.

**Q:** What is the high-watermark?
**A:** The committed boundary — replicated to all ISR; consumers read only below it.

**Q:** Where is a new leader chosen from?
**A:** The ISR (with a new leader epoch).

**Q:** Unclean leader election trade-off?
**A:** Availability at the cost of losing committed records.

**Q:** What replaced ZooKeeper in Kafka?
**A:** KRaft — a Raft quorum of controllers storing metadata in a log.

**Q:** How does Kafka mostly achieve durability?
**A:** Replication across brokers, not per-write fsync.
