# System Design Concepts, Topic by Topic

This plan is about **understanding**, not interviews. There's no estimation drill, no interview framework,
and no "design Twitter" exercise. Each topic takes one concept and works through it:
what problem it solves, how it actually works, what it costs, and how it connects to everything else.

There's no calendar. Go topic by topic, at whatever pace the topic needs. A small topic might take an
evening; a dense one (consensus, isolation levels) might take three.

---

## How each topic works

Each topic is a single file in `topics/`, named `<part>-<topic>-<name>.md`
(e.g. `topics/03-13-consistent-hashing.md`), so they sort in reading order. Each file contains:

| Section | What it's for |
|---|---|
| Lesson | The concept taught properly: the problem, the mechanism, diagrams, trade-offs, failure modes. |
| Questions | Conceptual "why" and "what happens if" questions to test understanding. |
| Answers | Collapsed by default. Open only after attempting the questions. |
| Flashcards | Short Q/A pairs for spaced repetition. |

`notes/` is your scratch pad: add as many files as you like (notes written from memory, sketches, rough drafts).

A good session looks like this: read the lesson slowly, attempt the questions without looking, check the
answers, then close everything and write your notes from memory.

**The test of understanding** for every topic: can you explain it to someone else, without notes,
including *why* it exists and *what goes wrong* without it?

---

## The map

```
Part 1   Foundations                       01–02
Part 2   Networking & communication        03–10
Part 3   Caching & the edge                11–14
Part 4   Storing data                      15–22
Part 5   Scaling data                      23–25
Part 6   Distributed systems               26–35
Part 7   Moving data between systems       36–42
Part 8   Reliability & operations          43–48
Part 9   Specialized building blocks       49–54
Part 10  Synthesis                         55
```

Topics are ordered so each one only depends on earlier ones. Parts 4 → 5 → 6 in particular build on
each other, so take them in order. Within a part, small detours are fine.

---

# Part 1 — Foundations

### 01 — Core vocabulary
- Latency vs throughput, and why improving one can hurt the other
- Availability, reliability, durability, fault tolerance: what's actually different
- Scalability: load parameters and performance under load
- Percentiles (p50/p99) and tail latency; why averages mislead
- Vertical vs horizontal scaling; stateful vs stateless components
- The basic building blocks and the job each one does: client, DNS, load balancer, app server, cache, database, queue, object storage, CDN

### 02 — Where time goes: latency hierarchy & queueing
- The latency numbers: CPU cache, RAM, SSD, disk, same-datacenter, cross-region
- Sequential vs random I/O, and why it shapes storage design
- Queueing basics: utilization vs wait time, why systems fall over near 100%
- Little's Law and what it says about concurrency, throughput and latency
- Tail-latency amplification under fan-out

---

# Part 2 — Networking & Communication

### 03 — IP, TCP & UDP
- The layers that matter: IP, TCP, UDP, application protocols
- TCP: handshake, reliability, flow control vs congestion control, head-of-line blocking
- UDP: when unreliability is a feature
- Connection reuse: keep-alive and connection pooling, and why opening connections is expensive

### 04 — DNS
- Resolution path: stub, recursive, root, TLD, authoritative
- Caching and TTLs, and why DNS changes are slow to propagate
- GeoDNS, anycast, DNS-based load balancing and failover

### 05 — TLS
- What the handshake establishes: authentication, key exchange, encryption
- Certificates and the chain of trust
- Handshake cost, session resumption, 0-RTT; TLS termination

### 06 — HTTP: 1.1 → 2 → 3
- HTTP/1.1: persistent connections, pipelining and why it failed
- HTTP/2: multiplexing, streams, header compression; TCP head-of-line blocking remains
- HTTP/3 and QUIC over UDP: what each version fixed

### 07 — API styles: REST, RPC, GraphQL
- REST: resources, verbs, safe vs idempotent methods, status code semantics
- RPC and gRPC: protobuf, contracts, streaming
- GraphQL: what it gives clients and what it trades away
- Synchronous vs asynchronous communication between services

### 08 — Real-time communication
- Short polling, long polling, Server-Sent Events, WebSockets: the mechanism behind each
- Holding millions of open connections: what it costs a server
- Fan-out to connected clients across many servers

### 09 — Load balancing
- Why load balancers exist; L4 (transport) vs L7 (application) balancing
- Algorithms: round robin, weighted, least connections, least response time, hashing, power of two choices
- Health checks (active vs passive), connection draining, slow start
- Sticky sessions and why they fight against statelessness
- Making the load balancer itself highly available (active-passive, floating IPs, DNS-level balancing)

### 10 — Proxies, API gateways & service mesh
- Forward proxy vs reverse proxy
- API gateway: routing, auth, aggregation, and the risk of a "god" gateway
- Service mesh: sidecars, data plane vs control plane, mTLS between services
- Service discovery: client-side vs server-side

---

# Part 3 — Caching & the Edge

### 11 — Caching fundamentals
- Why caching works: locality, skewed access, the latency gaps from topic 02
- Where caches live: browser, CDN, reverse proxy, application-local, distributed, database buffer pool
- Read patterns: cache-aside, read-through. Write patterns: write-through, write-behind, write-around
- Eviction policies: LRU, LFU, FIFO, TTL, and how LRU is actually implemented
- Invalidation and staleness vs consistency trade-offs
- Redis internals: single-threaded event loop, data structures, persistence (RDB vs AOF)

### 12 — Cache failure modes
- Cache stampede / thundering herd: request coalescing, locks, probabilistic early expiry
- Cache penetration (missing keys): negative caching, Bloom filters
- Cache avalanche (mass expiry): TTL jitter
- Hot keys: local caches, key replication
- Cold starts and cache warming

### 13 — Consistent hashing
- Why `hash(key) % N` breaks when N changes
- The ring, virtual nodes, what moves when nodes join or leave
- Alternatives: rendezvous (HRW) hashing, jump hash
- Where it shows up: caches, Dynamo-style stores, load balancers

### 14 — CDNs & the edge
- Edge PoPs, origin, origin shield, cache hierarchy
- Cache keys, `Cache-Control`, `ETag`, `Vary`, revalidation
- Push vs pull CDNs; purging vs versioned URLs
- Caching dynamic content; edge compute
- Signed URLs for private content

---

# Part 4 — Storing Data

Primary companion: *Designing Data-Intensive Applications* (Kleppmann), Ch. 2–4, 7.

### 15 — Data models
- Relational, document, wide-column, key-value, graph: the shape of data each fits
- Normalization vs denormalization; joins vs embedding
- Schema-on-write vs schema-on-read
- Why "SQL vs NoSQL" is the wrong framing: think in access patterns

### 16 — Encoding & schema evolution
- Text formats (JSON, XML) vs binary formats (Protobuf, Thrift, Avro)
- Forward and backward compatibility, and why rolling deploys need both
- Schema registries; evolving schemas in databases, APIs and message streams

### 17 — B-trees & the write-ahead log
- From an append-only log to a hash index to sorted structures
- B-trees: pages, fan-out, why depth stays at 3–4 for billions of rows, page splits
- Write-ahead log (WAL), fsync and crash recovery
- Buffer pools and dirty pages

### 18 — Indexes & query planning
- Clustered vs secondary indexes, covering indexes, composite indexes and the leftmost-prefix rule
- When an index can't be used; the write cost of every index
- How a query planner chooses between seq scan, index scan, and bitmap scan
- Join algorithms: nested loop, hash join, merge join

### 19 — LSM-trees
- Memtable → SSTable → compaction; why sequential writes win
- Bloom filters and how reads stay fast
- Compaction strategies (size-tiered vs leveled) and their costs
- Read, write, and space amplification
- B-tree vs LSM: when each wins

### 20 — Column storage, OLTP vs OLAP
- Transaction processing vs analytics: different access patterns, different engines
- Row-oriented vs column-oriented storage; compression and vectorized execution
- Data warehouses, star schemas, materialized views

### 21 — Transactions & isolation levels
- ACID, letter by letter, and what each actually promises
- Isolation levels: read uncommitted → read committed → repeatable read / snapshot → serializable
- Anomalies: dirty read/write, non-repeatable read, phantom, lost update, write skew
- Which level prevents which anomaly, and why real databases' names don't match the SQL standard

### 22 — Concurrency control
- MVCC: snapshots, row versions, visibility rules, vacuum and bloat (Postgres as the example)
- Pessimistic locking: row locks, `SELECT ... FOR UPDATE`, two-phase locking (2PL)
- Optimistic concurrency: version columns, compare-and-set
- Serializable Snapshot Isolation (SSI)
- Deadlocks: how they form, how databases detect them, how lock ordering prevents them

---

# Part 5 — Scaling Data

Primary companion: DDIA Ch. 5–6.

### 23 — Single-leader replication
- Why replicate: availability, read scaling, latency
- Sync vs async vs semi-sync; adding new followers
- Replication log implementations: statement-based, WAL shipping, logical (row-based)
- Failover and split brain
- Replication lag anomalies: read-your-writes, monotonic reads, consistent prefix

### 24 — Multi-leader & leaderless replication
- Multi-leader: use cases (multi-region, offline clients) and the conflict problem
- Leaderless (Dynamo-style): quorums (`W + R > N`) and their edge cases
- Read repair, anti-entropy, hinted handoff, sloppy quorums

### 25 — Partitioning (sharding)
- Why partition; partitioning combined with replication
- Range vs hash partitioning: the hotspot trade-off
- Skewed workloads and hot keys (the celebrity problem)
- Secondary indexes on partitioned data: local (scatter-gather) vs global
- Rebalancing: fixed partitions, dynamic partitioning, partitions proportional to nodes
- Request routing: client-aware, routing tier, coordination service

---

# Part 6 — Distributed Systems

Primary companion: DDIA Ch. 8–9; MIT 6.5840 lectures; Kleppmann's distributed-systems lecture series.

### 26 — Faults & partial failure
- Why distributed is fundamentally different from single-machine: partial failure
- Unreliable networks: lost, delayed, duplicated, reordered messages
- Timeouts and failure detection: you can't tell "slow" from "dead"
- Process pauses (GC, VM suspension) and their consequences
- Fault models: crash-stop, crash-recovery, Byzantine
- The Two Generals problem

### 27 — Time, clocks & ordering
- Wall clocks vs monotonic clocks; NTP and clock skew
- Why timestamps can't order events across machines (and how last-write-wins loses data)
- Happens-before and causality
- Lamport clocks; vector clocks / version vectors
- Hybrid logical clocks; Google's TrueTime and commit-wait

### 28 — Consistency models
- Linearizability: what it means, and how it differs from serializability
- Sequential, causal and eventual consistency
- Session guarantees: read-your-writes, monotonic reads, writes-follow-reads
- The cost of strong consistency: coordination, latency, availability

### 29 — CAP & PACELC
- CAP stated correctly (behavior during a partition) and its common misreadings
- PACELC: latency vs consistency when there's no partition
- Classifying real systems, and why the labels are often misleading

### 30 — Consensus & Raft
- What consensus solves: leader election, total order broadcast, membership
- FLP impossibility, and how real systems get around it
- Raft in depth: terms, leader election, log replication, commit index, safety
- Paxos: the core idea and why Raft was designed to be understandable
- Why 2f+1 nodes tolerate f failures

### 31 — Coordination services, leases & locks
- etcd and ZooKeeper: what they're for and what not to use them for
- Leases and why time-bounded ownership is needed
- Distributed locks and why they're dangerous: pauses, expiry, the fencing-token fix
- Redis-based locks (Redlock) and the debate around them
- ZooKeeper primitives: ephemeral nodes, sequential nodes, watches

### 32 — Membership, failure detection & gossip
- Cluster membership and why it's hard
- Gossip protocols: how information spreads, convergence time
- Failure detectors: heartbeats, SWIM, phi-accrual

### 33 — Conflict resolution & CRDTs
- Where conflicts come from (multi-leader, leaderless, offline clients)
- Last-write-wins and what it silently drops
- Siblings / application-level merge (the Dynamo shopping cart)
- CRDTs: state-based vs operation-based; G-counter, PN-counter, OR-set, LWW-register
- Operational transformation vs CRDTs for collaborative editing

### 34 — Distributed transactions & two-phase commit
- The atomic commit problem
- Two-phase commit: prepare/commit, the blocking problem, coordinator failure
- Three-phase commit and why it isn't used much; XA in practice
- How Spanner-style systems combine 2PC with consensus

### 35 — Sagas
- Long-lived business transactions without distributed locks
- Compensating actions; choreography vs orchestration
- TCC (try-confirm-cancel)
- Isolation anomalies in sagas and countermeasures (semantic locks, commutative updates)

---

# Part 7 — Moving Data Between Systems

### 36 — Message queues
- Why asynchronous messaging: decoupling, buffering, smoothing load
- Queue brokers (RabbitMQ, SQS): acks, visibility timeouts, redelivery, competing consumers
- Pub/sub, fan-out, routing; priority and delay queues

### 37 — Log-based messaging (Kafka)
- Topics, partitions, offsets, consumer groups, retention
- Queue vs log: the core distinction and what it enables (replay, multiple readers)
- Ordering guarantees and their limits (per-partition only)
- Kafka internals: ISR replication, leader election, log compaction

### 38 — Delivery semantics & idempotency
- At-most-once, at-least-once, exactly-once, and what "exactly-once" really means
- Idempotency: natural keys, idempotency keys, dedup tables, upserts
- Poison messages, dead-letter queues, replay safety

### 39 — Outbox & change data capture
- The dual-write problem
- Transactional outbox pattern
- CDC: reading the database's log (Debezium, logical replication)
- Keeping caches, search indexes and other derived data in sync

### 40 — Batch processing
- The Unix philosophy as a data-processing model
- MapReduce: map, shuffle, reduce; moving computation to data
- Distributed filesystems (GFS/HDFS) as the foundation
- Joins at scale: sort-merge, broadcast, partitioned hash joins
- Dataflow engines (Spark): DAGs, in-memory intermediate state, lineage-based recovery

### 41 — Stream processing
- Streams as unbounded data; events vs state
- Event time vs processing time
- Windows: tumbling, sliding (hopping), session
- Watermarks and late-arriving data
- Stateful stream processing: state stores, checkpointing (Flink); exactly-once in stream processors
- Lambda vs Kappa architecture

### 42 — Event sourcing & CQRS
- Events as the source of truth; rebuilding state from the log
- Snapshots, event versioning, and the cost of an immutable history
- CQRS: separate write and read models, and eventual consistency between them

---

# Part 8 — Reliability & Operations

### 43 — Timeouts, retries & backoff
- Why every remote call needs a timeout, and how to choose it
- Exponential backoff with jitter; retry budgets
- Retry amplification across layers; hedged requests
- Metastable failures and retry storms

### 44 — Circuit breakers, bulkheads & load shedding
- Circuit breakers: closed, open, half-open
- Bulkheads: isolating resources per dependency
- Backpressure vs load shedding; admission control
- Graceful degradation and fallbacks

### 45 — Rate limiting
- Why rate limit: protection, fairness, cost
- Token bucket, leaky bucket, fixed window, sliding window log, sliding window counter: mechanism and trade-offs
- Distributed rate limiting: shared counters, local vs global accuracy, race conditions
- Fail-open vs fail-closed

### 46 — Observability
- Metrics, logs, traces: what each answers
- Distributed tracing: trace IDs, spans, context propagation
- SLI, SLO, SLA, error budgets
- Golden signals; RED and USE methods

### 47 — Deployments & safe change
- Rolling, blue-green, canary; feature flags
- Zero-downtime schema changes (expand/contract)
- Config changes as a leading cause of outages

### 48 — Fault isolation, multi-region & disaster recovery
- Failure domains and blast radius; cell-based architecture; shuffle sharding
- Active-passive vs active-active multi-region
- RPO and RTO; backups vs replication

---

# Part 9 — Specialized Building Blocks

### 49 — Unique ID generation
- Why auto-increment breaks in a distributed system
- UUID v4 vs v7, Snowflake-style IDs, ticket servers
- Sortability, index locality, and clock dependence

### 50 — Probabilistic data structures
- Bloom filters in depth (false positives, sizing, counting Bloom filters)
- Count-Min Sketch for frequencies; HyperLogLog for cardinality
- Trading exactness for memory: when that's acceptable

### 51 — Search & inverted indexes
- Inverted indexes, tokenization, analyzers
- Relevance scoring: TF-IDF, BM25
- Tries and prefix search (autocomplete)
- How a search cluster shards and replicates (Elasticsearch as the example)

### 52 — Geospatial indexing
- Why a plain index on (lat, lng) doesn't work
- Geohash, quadtrees, S2 cells
- Proximity queries and edge cases at cell boundaries

### 53 — Object storage, erasure coding & Merkle trees
- Object storage: flat namespace, metadata vs data, chunking, content-addressing, deduplication
- Erasure coding vs replication: durability, cost, repair traffic
- Merkle trees and efficient data comparison

### 54 — Authentication & authorization
- Sessions vs tokens; where state lives
- JWT: structure, signing, and the revocation problem
- OAuth 2.0 and OpenID Connect flows: what each party knows
- Authorization models: RBAC, ABAC, relationship-based (Zanzibar)

---

# Part 10 — Synthesis

### 55 — Reading real systems
- Read the **Amazon Dynamo paper** (2007) and find every concept from this plan in it:
  consistent hashing, vnodes, quorums, vector clocks, hinted handoff, Merkle-tree anti-entropy, gossip
- Skim Google's Spanner or Bigtable papers with the same lens
- Build your own **concept map**: one page linking every topic to the ones it depends on
- List your 5 weakest concepts and note where to revisit them

---

## Progress Tracker

| # | Topic | Done | Date | Understanding (1–5) | Revisit |
|---|---|---|---|---|---|
| 01 | Core vocabulary | ☑ | 2026-10-08 | | |
| 02 | Latency hierarchy & queueing | ☐ | | | |
| 03 | IP, TCP & UDP | ☑ | 2026-10-08 | | |
| 04 | DNS | ☑ | 2026-10-08 | | |
| 05 | TLS | ☑ | 2026-10-08 | | |
| 06 | HTTP 1.1 → 2 → 3 | ☑ | 2026-10-08 | | |
| 07 | API styles: REST, RPC, GraphQL | ☑ | 2026-10-08 | | |
| 08 | Real-time communication | ☑ | 2026-10-08 | | |
| 09 | Load balancing | ☐ | | | |
| 10 | Proxies, API gateways & service mesh | ☐ | | | |
| 11 | Caching fundamentals | ☐ | | | |
| 12 | Cache failure modes | ☐ | | | |
| 13 | Consistent hashing | ☐ | | | |
| 14 | CDNs & the edge | ☐ | | | |
| 15 | Data models | ☐ | | | |
| 16 | Encoding & schema evolution | ☐ | | | |
| 17 | B-trees & the WAL | ☐ | | | |
| 18 | Indexes & query planning | ☐ | | | |
| 19 | LSM-trees | ☐ | | | |
| 20 | Column storage, OLTP vs OLAP | ☐ | | | |
| 21 | Transactions & isolation levels | ☐ | | | |
| 22 | Concurrency control | ☐ | | | |
| 23 | Single-leader replication | ☐ | | | |
| 24 | Multi-leader & leaderless replication | ☐ | | | |
| 25 | Partitioning (sharding) | ☐ | | | |
| 26 | Faults & partial failure | ☐ | | | |
| 27 | Time, clocks & ordering | ☐ | | | |
| 28 | Consistency models | ☐ | | | |
| 29 | CAP & PACELC | ☐ | | | |
| 30 | Consensus & Raft | ☐ | | | |
| 31 | Coordination services, leases & locks | ☐ | | | |
| 32 | Membership, failure detection & gossip | ☐ | | | |
| 33 | Conflict resolution & CRDTs | ☐ | | | |
| 34 | Distributed transactions & 2PC | ☐ | | | |
| 35 | Sagas | ☐ | | | |
| 36 | Message queues | ☐ | | | |
| 37 | Log-based messaging (Kafka) | ☐ | | | |
| 38 | Delivery semantics & idempotency | ☐ | | | |
| 39 | Outbox & change data capture | ☐ | | | |
| 40 | Batch processing | ☐ | | | |
| 41 | Stream processing | ☐ | | | |
| 42 | Event sourcing & CQRS | ☐ | | | |
| 43 | Timeouts, retries & backoff | ☐ | | | |
| 44 | Circuit breakers, bulkheads & load shedding | ☐ | | | |
| 45 | Rate limiting | ☐ | | | |
| 46 | Observability | ☐ | | | |
| 47 | Deployments & safe change | ☐ | | | |
| 48 | Fault isolation, multi-region & DR | ☐ | | | |
| 49 | Unique ID generation | ☐ | | | |
| 50 | Probabilistic data structures | ☐ | | | |
| 51 | Search & inverted indexes | ☐ | | | |
| 52 | Geospatial indexing | ☐ | | | |
| 53 | Object storage, erasure coding & Merkle trees | ☐ | | | |
| 54 | Authentication & authorization | ☐ | | | |
| 55 | Synthesis: reading real systems | ☐ | | | |

---

## Pacing

Keep the order; don't skip ahead into a part whose prerequisites you haven't done. If a topic is too
dense, split it across sessions. Understanding one topic properly is worth more than skimming two.

## Reference shelf (supplements, not substitutes)

- **DDIA**, Martin Kleppmann — the backbone for Parts 4–7
- **Martin Kleppmann's Distributed Systems lectures** (Cambridge, free on YouTube) — Part 6
- **MIT 6.5840** lectures — consensus and replication
- **Hussein Nasser** (YouTube) — networking, protocols, database internals
- **ByteByteGo** — quick visual refreshers
- Papers: Dynamo, Bigtable, GFS, MapReduce, Raft ("In Search of an Understandable Consensus Algorithm"), Spanner
