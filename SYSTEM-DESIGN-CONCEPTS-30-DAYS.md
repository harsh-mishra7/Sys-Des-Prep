# 30 Days of System Design Concepts

**Start:** 2026-10-08 · **End:** 2026-11-06 · **Budget:** ~2–3 hours/day

This plan is about **understanding**, not interviews. There's no estimation drill, no interview framework,
and no "design Twitter" exercise each day. Each day takes one concept area and works through it:
what problem it solves, how it actually works, what it costs, and how it connects to everything else.

---

## How each day works

Each day's folder (`dayNN/`) contains:

| File | What it's for |
|---|---|
| `README.md` | The lesson. The concepts taught properly: the problem, the mechanism, diagrams, trade-offs, failure modes. |
| `questions.md` | Conceptual "why" and "what happens if" questions to test understanding. |
| `answers.md` | Answers to `questions.md`. Read only after attempting them. |
| `flashcards.md` | Short Q/A pairs for spaced repetition. |
| `notes/` | Your scratch pad for the day: an empty folder where you add as many files as you like (notes written from memory, sketches, rough drafts). |

A good day looks like this: read the lesson slowly, attempt the questions without looking, check the
answers, then close everything and write your notes from memory. Use the time however works for you.

**The test of understanding** for every topic: can you explain it to someone else, without notes,
including *why* it exists and *what goes wrong* without it?

---

## The arc

```
Week 1  How requests move and how systems scale     (networking → load balancing → caching)
Week 2  How data is stored and kept correct         (storage engines → transactions → replication → sharding)
Week 3  What makes distributed systems hard         (failures → time → consistency → consensus)
Week 4  How data moves between systems              (messaging → streams → reliability)
Days 29–30  Specialized building blocks + tying it all together
```

Each week builds on the one before. Week 3 makes far more sense after Week 2, so don't skip ahead.

---

# Week 1 — Requests, Networking, Scaling

### Day 1 — Core vocabulary of system design
- Latency vs throughput, and why improving one can hurt the other
- Availability, reliability, durability, fault tolerance: what's actually different
- Scalability: what "scales" means (load parameters, performance under load)
- Percentiles (p50/p99) and tail latency; why averages mislead
- Vertical vs horizontal scaling; stateful vs stateless components
- The basic building blocks and the job each one does: client, DNS, load balancer, app server, cache, database, queue, object storage, CDN

### Day 2 — Networking foundations
- The layers that matter: IP, TCP, UDP, application protocols
- TCP: handshake, reliability, flow control vs congestion control, head-of-line blocking
- UDP: when unreliability is a feature
- DNS: resolution path, recursive vs authoritative, caching and TTLs, GeoDNS, anycast
- TLS: what the handshake establishes, its cost, session resumption
- Connection reuse: keep-alive and connection pooling, and why opening connections is expensive

### Day 3 — Application protocols & communication styles
- HTTP/1.1 → HTTP/2 (multiplexing, header compression) → HTTP/3 (QUIC over UDP): what each fixed
- REST: resources, verbs, safe vs idempotent methods, status code semantics
- RPC and gRPC: protobuf, contracts, streaming; GraphQL: what it trades away
- Server push: short polling, long polling, Server-Sent Events, WebSockets: the mechanism behind each
- Synchronous vs asynchronous communication between services

### Day 4 — Load balancing & proxies
- Why load balancers exist; L4 (transport) vs L7 (application) balancing
- Algorithms: round robin, weighted, least connections, least response time, hashing
- Health checks (active vs passive), connection draining, slow start
- Forward proxy vs reverse proxy vs API gateway vs service mesh (sidecars)
- Sticky sessions and why they fight against statelessness
- Making the load balancer itself highly available (active-passive, floating IPs, DNS-level balancing)

### Day 5 — Caching I: fundamentals
- Why caching works: locality, skewed access, the memory/disk/network latency gaps
- Where caches live: browser, CDN, reverse proxy, application-local, distributed, database buffer pool
- Read patterns: cache-aside, read-through. Write patterns: write-through, write-behind, write-around
- Eviction policies: LRU, LFU, FIFO, TTL, and how LRU is actually implemented
- Invalidation: the "hard problem", and staleness vs consistency trade-offs
- Redis internals: single-threaded event loop, data structures, persistence (RDB vs AOF)

### Day 6 — Caching II: failure modes & distributed caches
- Cache stampede / thundering herd and its fixes (request coalescing, locks, probabilistic early expiry)
- Cache penetration (missing keys) → negative caching, Bloom filters
- Cache avalanche (mass expiry) → TTL jitter
- Hot keys → local caches, key replication
- Distributing a cache: why `hash(key) % N` breaks when N changes
- **Consistent hashing**: the ring, virtual nodes, what moves when nodes join or leave

### Day 7 — CDNs & edge
- How a CDN works: edge PoPs, origin, origin shield, cache hierarchy
- Cache keys, `Cache-Control`, `ETag`, `Vary`, revalidation
- Push vs pull CDNs; purging vs versioned URLs
- Caching dynamic content; edge compute
- Signed URLs for private content
- **Week 1 review:** re-explain consistent hashing and the four cache failure modes from memory

---

# Week 2 — Storing Data and Keeping It Correct

Primary companion: *Designing Data-Intensive Applications* (Kleppmann), Ch. 2, 3, 5, 6, 7.

### Day 8 — Data models
- Relational, document, wide-column, key-value, graph: the shape of data each fits
- Normalization vs denormalization; joins vs embedding
- Schema-on-write vs schema-on-read
- The object-relational mismatch
- Why "SQL vs NoSQL" is the wrong framing: think in access patterns

### Day 9 — Storage engines I: B-trees & indexes
- From an append-only log to a hash index to sorted structures
- B-trees: pages, fan-out, why depth stays at 3–4 for billions of rows, page splits
- Write-ahead log (WAL) and crash recovery
- Clustered vs secondary indexes, covering indexes, composite indexes and the leftmost-prefix rule
- When an index can't be used; the write cost of every index
- How a query planner chooses between seq scan, index scan, and bitmap scan

### Day 10 — Storage engines II: LSM-trees
- Memtable → SSTable → compaction; why sequential writes win
- Bloom filters and how reads stay fast
- Compaction strategies (size-tiered vs leveled) and their costs
- Read, write, and space amplification
- **B-tree vs LSM**: when each wins
- Row-oriented vs column-oriented storage; why analytics databases are columnar

### Day 11 — Transactions & isolation
- ACID, letter by letter, and what each actually promises
- Isolation levels: read uncommitted → read committed → repeatable read / snapshot → serializable
- Anomalies: dirty read/write, non-repeatable read, phantom, lost update, **write skew**
- Which level prevents which anomaly, and why real databases' names don't match the SQL standard

### Day 12 — Concurrency control
- MVCC: snapshots, row versions, visibility rules, vacuum and bloat (with Postgres as the example)
- Pessimistic locking: row locks, `SELECT ... FOR UPDATE`, two-phase locking (2PL)
- Optimistic concurrency: version columns, compare-and-set
- Serializable Snapshot Isolation (SSI)
- Deadlocks: how they form, how databases detect them, how lock ordering prevents them

### Day 13 — Replication
- Why replicate: availability, read scaling, latency
- **Single-leader**: sync vs async vs semi-sync, adding followers, failover and split brain
- Replication log implementations: statement-based, WAL shipping, logical (row-based)
- **Replication lag** and its anomalies: read-your-writes, monotonic reads, consistent prefix
- **Multi-leader**: use cases and the conflict problem
- **Leaderless** (Dynamo-style): quorums (`W + R > N`), read repair, anti-entropy, hinted handoff, sloppy quorums

### Day 14 — Partitioning (sharding)
- Why partition; partitioning combined with replication
- Range vs hash partitioning: the hotspot trade-off
- Skewed workloads and hot keys (the celebrity problem)
- Secondary indexes on partitioned data: local (scatter-gather) vs global
- Rebalancing: fixed partitions, dynamic partitioning, partitions proportional to nodes
- Request routing: client-aware, routing tier, coordination service
- **Week 2 review:** explain an INSERT's journey: WAL → B-tree/LSM → replica → which partition

---

# Week 3 — The Hard Parts of Distributed Systems

Primary companion: DDIA Ch. 8 and 9; MIT 6.5840 (6.824) lectures; Kleppmann's distributed-systems lecture series.

### Day 15 — Faults & partial failure
- Why distributed is fundamentally different from single-machine: partial failure
- Unreliable networks: lost, delayed, duplicated, reordered messages
- Timeouts and failure detection: you can't tell "slow" from "dead"
- Network partitions in practice
- Process pauses (GC, VM suspension) and their consequences
- Fault models: crash-stop, crash-recovery, Byzantine
- The Two Generals problem

### Day 16 — Time, clocks & ordering
- Wall clocks vs monotonic clocks; NTP and clock skew
- Why timestamps can't order events across machines (and how last-write-wins loses data)
- Happens-before and causality
- **Lamport clocks** and total order
- **Vector clocks / version vectors** and detecting concurrency
- Hybrid logical clocks; Google's TrueTime and commit-wait

### Day 17 — Consistency models
- Linearizability: what it means, and how it differs from serializability
- Sequential consistency, causal consistency, eventual consistency
- Session guarantees: read-your-writes, monotonic reads, writes-follow-reads
- **CAP** stated correctly (behavior during a partition) and its limits
- **PACELC**: the latency vs consistency trade-off when there's no partition
- The cost of strong consistency: coordination, latency, availability

### Day 18 — Consensus
- What consensus solves: leader election, atomic commit, total order broadcast, membership
- FLP impossibility, and how real systems get around it (timeouts, randomization)
- **Raft** in depth: terms, leader election, log replication, commit index, safety
- Paxos: the core idea and why Raft was designed to be understandable
- Quorums and majorities: why 2f+1 nodes tolerate f failures
- Consensus-backed systems: etcd, ZooKeeper, and what they're used for

### Day 19 — Coordination: leader election, locks, leases
- Leases and why time-bounded ownership is needed
- **Distributed locks** and why they're dangerous: pauses, expiry, the fencing-token fix
- Redis-based locks (Redlock) and the debate around them
- ZooKeeper primitives: ephemeral nodes, sequential nodes, watches
- Service discovery and membership; gossip protocols and failure detectors (SWIM, phi-accrual)

### Day 20 — Conflict resolution & CRDTs
- Where conflicts come from (multi-leader, leaderless, offline clients)
- Last-write-wins and what it silently drops
- Siblings / application-level merge (the Dynamo shopping cart)
- **CRDTs**: state-based vs operation-based; G-counter, PN-counter, OR-set, LWW-register
- Operational transformation vs CRDTs for collaborative editing

### Day 21 — Distributed transactions
- The atomic commit problem
- **Two-phase commit**: prepare/commit, the blocking problem, coordinator failure
- Three-phase commit and why it isn't used much
- XA transactions in practice
- **Sagas**: compensating actions; choreography vs orchestration
- TCC (try-confirm-cancel); isolation anomalies in sagas
- **Week 3 review:** explain why consensus, linearizability and atomic commit are closely related

---

# Week 4 — Moving Data Between Systems

### Day 22 — Messaging: queues vs logs
- Why asynchronous messaging: decoupling, buffering, smoothing load
- Message queues (RabbitMQ, SQS): acks, visibility timeouts, redelivery, competing consumers
- Log-based brokers (Kafka): topics, partitions, offsets, consumer groups, retention
- **Queue vs log**: the core distinction and what it enables (replay, multiple readers)
- Ordering guarantees and their limits (per-partition only)
- Kafka internals: ISR replication, leader election, log compaction

### Day 23 — Delivery semantics & idempotency
- At-most-once, at-least-once, exactly-once, and what "exactly-once" really means
- Idempotency: natural keys, idempotency keys, dedup tables, upserts
- **The dual-write problem**
- **Transactional outbox** pattern
- **Change Data Capture (CDC)**: reading the database's log (Debezium, logical replication)
- Poison messages, dead-letter queues, replay safety

### Day 24 — Batch processing
- The Unix philosophy as a data-processing model
- MapReduce: map, shuffle, reduce; why it moves computation to data
- Distributed filesystems (GFS/HDFS) as the foundation
- Joins at scale: sort-merge, broadcast, partitioned hash joins
- Dataflow engines (Spark): DAGs, in-memory intermediate state, lineage-based recovery

### Day 25 — Stream processing
- Streams as unbounded data; events vs state
- **Event time vs processing time**
- Windows: tumbling, sliding (hopping), session
- **Watermarks** and late-arriving data
- Stateful stream processing: state stores, checkpointing (Flink)
- Exactly-once in stream processors
- Lambda vs Kappa architecture; event sourcing and CQRS

### Day 26 — Resilience patterns
- Timeouts: why every remote call needs one, and how to choose them
- Retries: exponential backoff with **jitter**, retry budgets, retry amplification across layers
- **Circuit breakers**: closed, open, half-open
- **Bulkheads**: isolating resources per dependency
- Backpressure and load shedding
- Graceful degradation and fallbacks
- Metastable failures and retry storms

### Day 27 — Rate limiting & flow control
- Why rate limit: protection, fairness, cost
- Algorithms: **token bucket, leaky bucket, fixed window, sliding window log, sliding window counter**, with the mechanism and trade-offs of each
- Distributed rate limiting: shared counters, local vs global accuracy, race conditions
- Fail-open vs fail-closed
- Admission control and queue-based flow control

### Day 28 — Observability & operational concepts
- Metrics, logs, traces: what each answers
- Distributed tracing: trace IDs, spans, context propagation
- SLI, SLO, SLA, error budgets: concepts and meaning
- Golden signals; RED and USE methods
- Deployment strategies: rolling, blue-green, canary, feature flags
- Zero-downtime schema changes (expand/contract)
- Failure domains, blast radius, cell-based architecture

---

# Days 29–30 — Specialized Building Blocks & Synthesis

### Day 29 — Specialized data structures & storage
- **Probabilistic structures**: Bloom filters (revisited in depth), Count-Min Sketch, HyperLogLog
- **Search**: inverted indexes, tokenization, relevance scoring (TF-IDF, BM25), how Elasticsearch shards
- **Tries** and prefix search
- **Geospatial indexing**: geohash, quadtrees, S2 cells
- **Object storage**: flat namespace, metadata vs data, content-addressing, chunking, deduplication
- **Erasure coding** vs replication
- Merkle trees and efficient data comparison

### Day 30 — Synthesis: reading real systems
- Read the **Amazon Dynamo paper** (2007) and identify every concept from this month in it:
  consistent hashing, vnodes, quorums, vector clocks, hinted handoff, Merkle-tree anti-entropy, gossip
- Skim Google's Spanner or Bigtable papers with the same lens
- Build your own **concept map**: one page linking every topic you studied to the ones it depends on
- List your 5 weakest concepts and note where to revisit them

---

## Progress Tracker

| Day | Date | Topic | Done | Understanding (1–5) | Revisit |
|---|---|---|---|---|---|
| 1 | 2026-10-08 | Core vocabulary | ☑ | | |
| 2 | | Networking foundations | ☐ | | |
| 3 | | Protocols & communication styles | ☐ | | |
| 4 | | Load balancing & proxies | ☐ | | |
| 5 | | Caching I: fundamentals | ☐ | | |
| 6 | | Caching II: failure modes & consistent hashing | ☐ | | |
| 7 | | CDNs & edge (+ week review) | ☐ | | |
| 8 | | Data models | ☐ | | |
| 9 | | B-trees & indexes | ☐ | | |
| 10 | | LSM-trees & columnar storage | ☐ | | |
| 11 | | Transactions & isolation | ☐ | | |
| 12 | | Concurrency control | ☐ | | |
| 13 | | Replication | ☐ | | |
| 14 | | Partitioning (+ week review) | ☐ | | |
| 15 | | Faults & partial failure | ☐ | | |
| 16 | | Time, clocks & ordering | ☐ | | |
| 17 | | Consistency models & CAP | ☐ | | |
| 18 | | Consensus & Raft | ☐ | | |
| 19 | | Coordination, locks, leases | ☐ | | |
| 20 | | Conflict resolution & CRDTs | ☐ | | |
| 21 | | Distributed transactions (+ week review) | ☐ | | |
| 22 | | Queues vs logs | ☐ | | |
| 23 | | Delivery semantics, outbox, CDC | ☐ | | |
| 24 | | Batch processing | ☐ | | |
| 25 | | Stream processing | ☐ | | |
| 26 | | Resilience patterns | ☐ | | |
| 27 | | Rate limiting & flow control | ☐ | | |
| 28 | | Observability & operations | ☐ | | |
| 29 | | Specialized data structures & storage | ☐ | | |
| 30 | | Synthesis: Dynamo paper + concept map | ☐ | | |

---

## If you fall behind

Keep the order and push the end date back; don't skip ahead. If a day is too dense, split it across two.
Understanding one topic properly is worth more than skimming two.

## Reference shelf (supplements, not substitutes)

- **DDIA**, Martin Kleppmann — the backbone for Weeks 2–4
- **Martin Kleppmann's Distributed Systems lectures** (Cambridge, free on YouTube) — Week 3
- **MIT 6.5840** lectures — consensus and replication
- **Hussein Nasser** (YouTube) — networking, protocols, database internals
- **ByteByteGo** — quick visual refreshers
- Papers: Dynamo, Bigtable, GFS, MapReduce, Raft ("In Search of an Understandable Consensus Algorithm"), Spanner
