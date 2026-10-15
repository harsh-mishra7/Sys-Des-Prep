# 55 — Reading Real Systems

> Part 10 — Synthesis · Prerequisites: topics 01–54 (especially 13, 17, 19, 21–34, 50, 53)

Fifty-four topics taught one mechanism at a time: consistent hashing in isolation, quorums in isolation,
vector clocks in isolation. Real systems don't come in pieces. A production database is dozens of these
mechanisms chosen together, each one compensating for the weaknesses of another, all bent around one or
two overriding requirements. Understanding a system means seeing *why this combination*: which requirement
forced which choice, and which problem each later mechanism exists to patch.

The best training for that is reading the papers in which real systems explained themselves. This topic
is a guided reading. We walk through Amazon's **Dynamo** paper (2007) section by section and map every
mechanism to the topic where it was taught; then apply the same lens, faster, to Google's **Spanner**
(2012) and **Bigtable** (2006), which made almost opposite choices. Then you build a concept map of the
whole plan and assess where your understanding is weakest.

Read this topic with the papers open beside it. All three are free online and readable in an evening each.

By the end of this topic you should be able to:
1. Read a systems paper actively: identify its requirements, its key design decisions, and the trade-offs it accepts.
2. Trace every major mechanism in Dynamo (consistent hashing, virtual nodes, quorums, vector clocks, hinted handoff, Merkle-tree anti-entropy, gossip) to its requirement and to the topic that taught it.
3. Do the same for Spanner (Paxos groups, 2PC, TrueTime, commit wait) and Bigtable (tablets, LSM storage, Chubby), and explain why their choices differ from Dynamo's.
4. Build a concept map linking every topic to the topics it depends on.
5. Assess your own understanding honestly and plan targeted revision.

---

## 1. How to read a systems paper

### 1.1 Three passes

S. Keshav's well-known "How to Read a Paper" suggests three passes; adapted for systems papers:

1. **Skim (15 minutes).** Abstract, introduction, section headings, figures, conclusion. Answer: what
   problem, what's the one-sentence design, what's claimed?
2. **Read for mechanisms (1–2 hours).** Each design section: what does it do, step by step, and *which
   requirement or failure does it address*? Skip proofs and dense evaluation details for now.
3. **Read critically.** What did they give up? What happens in the failure cases they don't discuss? What
   would break at 10× scale? What did later systems (or later papers from the same company) change?

### 1.2 Questions to ask every paper

Keep these open while reading; they're the lens used for all three papers below.

| Question | Why it matters |
|---|---|
| What is the **one requirement** that dominates every other? | It explains most of the design. Dynamo: writes must always succeed. Spanner: externally consistent transactions across the globe. Bigtable: huge, sorted, sparse tables at low cost. |
| What is the **data model and API**? | It constrains what consistency means and what the system must coordinate (topic 15). |
| How is data **partitioned**, and how are requests **routed**? | Topic 13, topic 25. |
| How is data **replicated**, and who can accept writes? | Single-leader, multi-leader, leaderless, consensus (topics 23, 24, 30). |
| What **consistency** does a read see? | Topics 28, 29. |
| How are **conflicts or concurrency** handled? | Topics 22, 27, 33. |
| What does the system **assume about failures and clocks**? | Topics 26, 27. |
| How are **membership** and **failure detection** done? | Topics 31, 32. |
| What does the **storage engine** look like on one node? | Topics 17, 19. |
| What did they **give up**, and do they admit it? | Every design trades something (topics 29, 48). |

---

## 2. Dynamo (2007): a guided reading

**Paper**: DeCandia, Hastorun, Jampani, Kakulapati, Lakshman, Pilchin, Sivasubramanian, Vosshall, Vogels,
"Dynamo: Amazon's Highly Available Key-value Store", SOSP 2007.

Context: Amazon's e-commerce platform, mid-2000s. Services like the shopping cart and session state needed
a store that **never rejects a write**, even when disks fail, networks partition, or a whole datacenter
has trouble, because every rejected "add to cart" was lost revenue. Note: Dynamo is **not** the same system
as Amazon DynamoDB (section 2.10).

### 2.1 Section 1 — Introduction: the dominant requirement

The paper opens with "reliability at massive scale" and the claim that the slightest outage has
financial consequences. The key sentence for understanding everything that follows: the shopping cart
must be able to accept writes "even if disks are failing, network routes are flapping, or data centers are
being destroyed by tornados".

Mapping:
- **"Always writeable"** is availability chosen over consistency during failures: AP in CAP terms
  (topic 29). Every later mechanism exists to keep writes flowing despite failures, then clean up the
  consequences.
- Failure as the normal case at scale: **topic 26** (partial failure) and **topic 01** (availability vs
  reliability vs durability).

### 2.2 Section 2 — Background: requirements and SLAs

**2.1 System assumptions and requirements.**
- **Query model**: simple reads and writes of a single item identified by a key; values are binary blobs
  (usually under 1 MB); no operations span multiple items, no relational schema. → **Topic 15** (key-value
  data model chosen for its access pattern).
- **ACID properties**: Dynamo deliberately offers **weaker consistency** and **no isolation guarantees**,
  and only single-key updates, because "data stores that provide ACID guarantees tend to have poor
  availability". → **Topic 21** (what each ACID letter costs), **topic 28** (consistency models).
- **Efficiency**: latency requirements measured at the **99.9th percentile**. → **Topic 02** and **topic
  46**.
- **Other assumptions**: internal use only, in a trusted environment; no authentication or authorization
  inside Dynamo. → Contrast with **topic 54**: security was explicitly out of scope.

**2.2 Service Level Agreements.** The paper's famous figure shows a single page request fanning out to
over 150 services. Amazon's SLAs are expressed at the 99.9th percentile because averages hide the
experience of the most valuable customers (those with the longest histories, hence the most data).
→ **Topic 02** (tail latency amplification under fan-out), **topic 46** (SLIs/SLOs/SLAs, why averages
mislead), **topic 01** (percentiles).

**2.3 Design considerations.** This subsection is the heart of the paper. Read it slowly.
- **Optimistic replication**: replicate changes in the background, tolerate disconnected operation,
  accept that replicas diverge temporarily. → **Topic 24** (leaderless replication), **topic 28**
  (eventual consistency).
- **When to resolve conflicts: at read time, not write time.** Traditional systems resolve at write time
  (reject a write if replicas can't agree), which costs availability. Dynamo pushes complexity to reads so
  writes are never rejected. → **Topic 33** (where conflicts come from and when to resolve them),
  **topic 29**.
- **Who resolves: the application or the data store.** The store can only apply simple policies like
  last-write-wins; the application knows the data's meaning and can merge (the shopping cart). → **Topic
  33** (LWW vs application-level merge).
- **Incremental scalability, symmetry, decentralization, heterogeneity**: add one node at a time; no
  special nodes (no master); prefer peer-to-peer techniques; let capacity vary per node. → **Topic 13**
  (consistent hashing for incremental scaling), **topic 32** (decentralized membership), **topic 09**
  contrast (no central load balancer in the data path).

### 2.3 Section 3 — Related work

Dynamo positions itself against peer-to-peer systems (Chord, Pastry: multi-hop routing), distributed file
systems and databases (Bayou, Coda, Ficus: conflict resolution; GFS: single master), and Bigtable.
Its distinctive choice: a **zero-hop DHT**, where every node knows enough routing information to send a
request directly to the right node, because multi-hop routing adds latency variance that hurts the 99.9th
percentile. → **Topic 13** (consistent hashing came from DHTs), **topic 25** (request routing), **topic
02** (variance at the tail).

### 2.4 Section 4 — System architecture: the mechanisms

The paper's **Table 1** is the best one-glance summary. Here it is with topic mappings:

| Problem | Dynamo's technique | Advantage claimed | Taught in |
|---|---|---|---|
| Partitioning | Consistent hashing | Incremental scalability | Topic 13, 25 |
| High availability for writes | Vector clocks with reconciliation during reads | Version size is decoupled from update rates | Topics 27, 33 |
| Handling temporary failures | Sloppy quorum and hinted handoff | High availability and durability when some replicas are unavailable | Topic 24 |
| Recovering from permanent failures | Anti-entropy using Merkle trees | Synchronizes divergent replicas in the background | Topics 24, 53 |
| Membership and failure detection | Gossip-based membership protocol and failure detection | Symmetry; no centralized registry | Topic 32 |

Now each subsection.

**4.1 System interface.** Two operations: `get(key)` returns an object, **or a list of conflicting
versions**, plus a **context**; `put(key, context, object)` writes, passing back the context from a
previous read. The context carries the object's vector clock. Keys are hashed with MD5 to a 128-bit
identifier that determines placement.
→ **Topic 07** (API design: the API exposes conflicts to the caller, which is unusual and deliberate),
**topic 27** (the context is a version vector), **topic 22** (the context works like an optimistic
concurrency version, except a stale context doesn't fail: it creates a sibling).

**4.2 Partitioning algorithm.** Consistent hashing: the output range of the hash is a ring; each node is
assigned a random position; a key belongs to the first node clockwise from its hash. The paper names the
two problems of basic consistent hashing: random positions give **non-uniform load**, and the scheme is
**oblivious to node heterogeneity**. The fix is **virtual nodes**: each physical node owns many positions
on the ring. Benefits listed: when a node fails, its load spreads evenly over the remaining nodes; when a
node joins, it takes roughly equal load from all others; and the number of virtual nodes per physical node
can follow its capacity.
→ **Topic 13** (the ring, vnodes, what moves when nodes join or leave), **topic 25** (rebalancing).

**4.3 Replication.** Each key is stored on `N` nodes: the coordinator (first node clockwise) plus the
`N - 1` next distinct nodes clockwise. The list of nodes responsible for a key is its **preference list**.
Because of virtual nodes, the successors on the ring might be the same physical machine, so the preference
list skips positions to ensure **`N` distinct physical nodes**; it's also built to span datacenters.
→ **Topic 24** (replication factor N), **topic 13** (successor replication on the ring), **topic 48**
(spanning failure domains and datacenters).

**4.4 Data versioning.** Dynamo is eventually consistent: a `put` may return before all replicas have it,
and a subsequent `get` may return an older version. Every modification produces a new immutable version.
Most of the time new versions subsume old ones (**syntactic reconciliation**: the system can tell which
is newer). When versions branch because of failures plus concurrent updates, the system cannot pick, and
returns all of them for the client to merge (**semantic reconciliation**).

The paper's example: the shopping cart. "Add to cart" must never be lost, so when two versions of a cart
diverge, the application merges them by **union**. Known consequence: **deleted items can reappear**
(a removal in one branch is undone by the union with the other branch).

Causality is tracked with **vector clocks**: a list of `(node, counter)` pairs per version. If every
counter in version A's clock is ≤ the corresponding counter in B's, A is an ancestor of B and can be
discarded; otherwise they're concurrent siblings. The figure walks through writes handled by nodes Sx,
Sy, Sz producing `D3([Sx,2],[Sy,1])` and `D4([Sx,2],[Sz,1])`, which are concurrent, then reconciled by
a client into `D5([Sx,3],[Sy,1],[Sz,1])`.

A practical issue: vector clocks grow if many different nodes coordinate writes for a key (which happens
under failures). Dynamo **truncates** the clock: each entry stores a timestamp of when the node last
updated it, and when the clock exceeds a threshold (e.g. 10 entries), the oldest pair is dropped. The paper
admits this can lose causality information and produce false conflicts, but says the problem "has not
surfaced in production".
→ **Topic 27** (happens-before, vector clocks/version vectors; why timestamps can't order events),
**topic 33** (siblings, application merge, the shopping cart, why deleted items resurface, and why CRDTs
like an OR-set fix exactly this), **topic 28** (eventual consistency).

**4.5 Execution of get() and put().** A client reaches a node either through a generic load balancer
(which forwards to a node in the preference list) or with a **partition-aware client library** that routes
directly to the coordinator. The coordinator for a `put` generates the new vector clock, writes locally,
sends the write to the other `N - 1` nodes, and succeeds when **`W`** of them respond. A `get` asks all
`N` nodes and returns once **`R`** respond; if they return divergent versions, it returns all causally
unrelated ones.

Setting `R + W > N` yields a quorum-like system: the read set and write set overlap. Latency is dictated
by the slowest of the `R` (or `W`) replicas, so `R` and `W` are usually less than `N`.
→ **Topic 24** (quorums and their edge cases), **topic 25** (client-aware routing vs routing tier),
**topic 10** (client-side vs server-side discovery), **topic 02** (latency = slowest of the replicas you
wait for).

**4.6 Handling failures: hinted handoff.** A strict quorum would make writes fail when some of the key's
`N` home nodes are unreachable. Dynamo uses a **sloppy quorum**: reads and writes go to the **first `N`
healthy nodes** on the preference list, which may include nodes outside the key's home set. A write meant
for an unreachable node A is sent to node D with a **hint** in its metadata saying "this belongs to A". D
keeps hinted replicas in a separate local store and, when it detects A has recovered, delivers them and
deletes its copy. Preference lists spanning datacenters let this survive whole-datacenter failures.

The critical consequence (the one topic 24 stresses): with a sloppy quorum, `R + W > N` **no longer
guarantees** that a read sees the latest write, because the `W` nodes that took the write and the `R` nodes
that serve the read may not overlap. Sloppy quorums buy write availability, not consistency.
→ **Topic 24** (sloppy quorums, hinted handoff), **topic 29** (choosing A over C), **topic 48**
(cross-datacenter replication for disaster tolerance).

**4.7 Handling permanent failures: replica synchronization.** Hints can be lost (the hint holder itself
may fail before handing off). To repair divergence in the background, Dynamo uses **anti-entropy with
Merkle trees**: each node keeps a Merkle tree per key range it hosts (the set of keys covered by a virtual
node); two nodes compare roots, descend into differing subtrees, and synchronize only the differing keys.
The paper states the benefit (minimal data transferred, fewer disk reads) and the drawback: **when a node
joins or leaves, key ranges change, and many trees must be recalculated**, a problem section 6.2's
partitioning strategy later addresses.
→ **Topic 53** (Merkle trees, comparison in O(d log n), identical range requirement), **topic 24**
(anti-entropy alongside read repair and hinted handoff).

**4.8 Membership and failure detection.**
- *Ring membership*: an administrator explicitly adds or removes a node with a command-line tool; the
  change is written to persistent storage and then **propagated by gossip**: every second each node
  contacts a random peer and they reconcile their persisted membership change histories. Partitioning and
  placement information propagates the same way, so each node can route directly (zero-hop).
  Explicit add/remove (rather than inferring membership from failures) exists because Amazon's outages are
  mostly transient and shouldn't trigger expensive rebalancing.
- *External discovery*: gossip alone can create a **logically partitioned ring** (two administrators add
  nodes A and B separately; they don't know about each other). Fix: **seed nodes**, known to all nodes via
  configuration, through which everyone eventually reconciles.
- *Failure detection*: purely **local** and **request-driven**: node A considers B failed if B doesn't
  respond to A's messages, routes around it, and periodically retries. Because explicit membership handles
  permanent changes, there's no need for a globally consistent view of failure; gossip-based detection was
  used in early designs and found unnecessary.
→ **Topic 32** (membership, gossip convergence, failure detectors), **topic 26** (can't distinguish slow
from dead; timeouts), **topic 43** (timeouts and retries), **topic 31** (contrast: no coordination service
like ZooKeeper; Dynamo is decentralized by choice).

**4.9 Adding/removing storage nodes.** A new node takes over key ranges from existing nodes; those nodes
transfer the keys and stop being responsible. Because of consistent hashing, only the ranges adjacent to
the new node's positions move.
→ **Topic 13** (minimal movement), **topic 25** (rebalancing).

### 2.5 Section 5 — Implementation

Each node has three components: request coordination, membership and failure detection, and a **local
persistence engine** that is **pluggable**: Berkeley DB Transactional Data Store, BDB Java Edition, MySQL,
or an in-memory buffer with persistent backing store, chosen per application by object size and access
pattern. Request coordination is an event-driven state machine built on Java NIO channels.

Two details worth mapping:
- **Read repair**: after returning to the client, the coordinator waits briefly for late responses; any
  replica that returned a stale version is updated with the latest version. → **Topic 24**.
- **Write coordinator choice**: the coordinator for a write is chosen as **the node that replied fastest
  to the preceding read** (whose context the write carries), increasing the chance of read-your-writes.
  → **Topic 23** (read-your-writes), **topic 28** (session guarantees).
→ Storage engines: **topic 17** (B-tree/WAL engines like BDB and InnoDB) and **topic 19** (alternatives).

### 2.6 Section 6 — Experiences and lessons learned

This section is where the paper is most valuable: what happened in production.

- **Configurable (N, R, W)**: the common configuration is **(3, 2, 2)**. Setting `W = 1` gives very
  available writes but risks durability; `R = 1, W = N` gives a high-performance read engine. Different
  services (shopping cart, session management, product catalog cache) tune them differently.
  → **Topic 24**, **topic 28** (consistency as a tunable cost).

**6.1 Balancing performance and durability.** A latency graph shows 99.9th percentile write latency far
above the average, with diurnal patterns. For services needing better latency, Dynamo can use an
**in-memory object buffer**: writes go to memory and a writer thread flushes to disk periodically; reads
check the buffer first. This lowers 99.9th percentile latency by a factor of 5 at peak, but risks losing
buffered writes on a crash. Mitigation: the coordinator asks **one of the N replicas to perform a durable
write**, while the others buffer.
→ **Topic 01** (durability vs performance), **topic 17** (fsync cost; why write-ahead logging matters),
**topic 11** (write-behind caching and its risk), **topic 02**.

**6.2 Ensuring uniform load distribution.** The paper measures load imbalance and compares **three
partitioning strategies**:
1. *T random tokens per node, partition by token value* (the original scheme): ranges vary in size,
   bootstrapping a new node requires scanning other nodes' data to find keys, and Merkle trees must be
   recalculated when membership changes; archiving is hard.
2. *T random tokens per node and equal-sized partitions*: decouples partitioning from placement.
3. *Q/S tokens per node, equal-sized partitions*: the hash space is divided into **Q fixed, equal-sized
   partitions**, and each of S nodes is assigned Q/S of them. When nodes join or leave, whole partitions
   move. Best load-balancing efficiency, simpler bootstrapping and recovery (partitions are files that can
   be transferred whole), and simpler archival.
→ **Topic 25** (rebalancing: *fixed number of partitions* is exactly strategy 3, the approach also used by
Riak, Elasticsearch, Couchbase), **topic 13** (vnodes), **topic 53** (Merkle trees per fixed partition no
longer need recomputing when membership changes).

**6.3 Divergent versions: when and how many.** Over 24 hours of shopping-cart traffic, **99.94% of
requests saw exactly one version**, 0.00057% saw 2, and fewer saw more. Divergence was mostly caused by
**concurrent writers**, often automated clients (robots), rather than by failures.
→ **Topic 33** (how often conflicts actually happen), **topic 27** (concurrency vs causality).

**6.4 Client-driven or server-driven coordination.** Moving coordination into the client library
(which pulls membership state every 10 seconds) removes a network hop and the load balancer, improving
99.9th percentile latency by at least 30 ms. Cost: clients may route with stale membership for a while.
→ **Topic 25** (client-aware routing), **topic 10** (client-side discovery), **topic 09** (removing the load
balancer from the path).

**6.5 Balancing background vs foreground tasks.** Background tasks (replica synchronization, data handoff)
compete with foreground gets/puts. An **admission controller** monitors foreground performance (disk
latency, lock contention, transaction timeouts) and allocates time slices to background tasks only when
foreground latency is healthy.
→ **Topic 44** (admission control, load shedding, bulkheads), **topic 46** (monitoring the signals that
drive it).

**6.6 Discussion.** Dynamo's main lesson: an eventually consistent store can be the building block for
highly available applications, but applications must be designed for it (merging, idempotency, tolerating
staleness). The full-membership routing table (each node knows every other) is noted as a scalability
limit for very large clusters (thousands of nodes), which a hierarchical scheme would be needed to address.
→ **Topic 38** (idempotency), **topic 33**, **topic 32** (membership state size).

### 2.7 Checklist: the plan's Dynamo concepts

The plan asked you to find these in the paper. Where each lives:

| Concept | Paper section | Topic |
|---|---|---|
| Consistent hashing | 4.2 | 13 |
| Virtual nodes | 4.2, 6.2 | 13, 25 |
| Quorums (N, R, W) | 4.5, 6 | 24 |
| Vector clocks | 4.4 | 27, 33 |
| Hinted handoff / sloppy quorum | 4.6 | 24 |
| Merkle-tree anti-entropy | 4.7 | 53, 24 |
| Gossip | 4.8 | 32 |
| Read repair | 5 | 24 |
| Shopping-cart merge | 4.4, 6.3 | 33 |
| 99.9th-percentile SLAs | 2.2, 6.1 | 02, 46 |
| Admission control | 6.5 | 44 |
| Fixed partitions for rebalancing | 6.2 | 25 |

### 2.8 Concepts in the plan that Dynamo deliberately avoids

Equally instructive is what Dynamo does **not** use, and why:

| Not used | Why not | Topic |
|---|---|---|
| A leader per key / single-leader replication | Leader failover stalls writes | 23 |
| Consensus (Paxos/Raft) | Requires a majority; unavailable in minority partitions | 30 |
| Distributed transactions / 2PC | Blocking; Dynamo only needs single-key ops | 34 |
| Linearizability | Requires coordination on every operation | 28 |
| A coordination service for membership | Avoids a central dependency; uses explicit admin + gossip | 31 |
| Last-write-wins by wall clock | Silently drops cart additions; clocks are unreliable | 27, 33 |
| Security | Assumed trusted environment | 54 |

### 2.9 The critical read

Questions a critical reader should ask after finishing:
- **Who pays for availability?** Application developers, who must write merge logic and live with
  resurrected deletes. Later systems moved toward CRDTs (Riak data types) to make merging automatic and
  correct (topic 33).
- **Does `R + W > N` mean strong consistency?** Not with sloppy quorums, and not even with strict quorums
  under all timing conditions (topic 24's edge cases: concurrent writes, failed writes partially applied,
  read repair races).
- **Vector clock truncation** sacrifices correctness (false siblings or, worse, lost causality) for bounded
  size, with only production experience as justification.
- **Anti-entropy cost**: Merkle trees per range must be maintained and recomputed on membership change,
  addressed by fixed partitions.

### 2.10 Legacy: Cassandra, Riak, and DynamoDB

- **Apache Cassandra** (Facebook, 2008) took Dynamo's **distribution** (consistent hashing, vnodes, tunable
  quorums, hinted handoff, gossip, Merkle-tree repair) and Bigtable's **data model and storage engine**
  (column families, memtable/SSTable LSM-tree). It resolves conflicts with **last-write-wins** by timestamp
  per cell, not vector clocks: simpler, but with the data-loss risks of topic 27 and topic 33.
- **Riak** followed the Dynamo paper closely, including vector clocks and siblings, and later added
  CRDT-based data types.
- **Amazon DynamoDB** (2012) shares the name and the key-value/document API, but its architecture differs:
  per the 2022 USENIX ATC paper on DynamoDB, each partition is replicated with **Multi-Paxos** and has a
  **leader** that serves writes and strongly consistent reads. It traded Dynamo's leaderless design for
  predictable performance and simpler semantics for customers. A good example of how the "right" design
  depends on who the users are (internal teams vs external customers).

---

## 3. Spanner (2012) through the same lens

**Paper**: Corbett et al., "Spanner: Google's Globally-Distributed Database", OSDI 2012.

**Dominant requirement**: a globally distributed database that offers **externally consistent**
(linearizable / strictly serializable) **transactions** with SQL-like semantics, because application teams
inside Google (the advertising backend, F1) found that building on weakly consistent stores like Bigtable
pushed too much complexity onto them. It's Dynamo's mirror image: where Dynamo gives up consistency for
availability, Spanner gives up some availability and latency for consistency.

### 3.1 Section 2 — Implementation: architecture

- A deployment is a **universe**; it's divided into **zones** (units of administrative deployment and
  physical isolation, roughly a datacenter or part of one). Each zone has a **zonemaster** and hundreds to
  thousands of **spanservers**. → **Topic 48** (failure domains, multi-region).
- **Spanserver stack (2.1)**: each spanserver serves 100–1000 **tablets**; a tablet is a bag of mappings
  `(key, timestamp) → value` (multi-versioned), stored in B-tree-like files plus a **write-ahead log** on
  **Colossus** (successor to GFS). → **Topic 17** (WAL), **topic 22** (multi-version storage, MVCC),
  **topic 40** (GFS/Colossus as the distributed filesystem underneath), **topic 53** (storage separated from
  serving).
- **One Paxos state machine per tablet**: each tablet's replicas (across zones) form a **Paxos group**; writes
  go through the group's **leader**, which holds a time-based **leader lease** (default 10 s). Reads at a
  timestamp can be served by any sufficiently up-to-date replica. → **Topic 30** (consensus, leader
  election, 2f+1), **topic 31** (leases), **topic 23** (leader-based replication; here every leader change is
  a consensus decision, so no split brain).
- **Lock table at the leader** for concurrency control within the group: **two-phase locking**. → **Topic
  22** (2PL).
- **Transaction manager** at each leader: if a transaction involves multiple Paxos groups, their leaders run
  **two-phase commit**; one group is the coordinator. → **Topic 34** — and this is the exact case the
  topic's last bullet describes: **2PC whose participants are Paxos groups** rather than single machines,
  so a participant (or the coordinator) failing doesn't block the commit forever: the group elects a new
  leader that has the full 2PC state in its replicated log. The classic 2PC blocking problem is solved by
  making each participant highly available.
- **Directories and placement (2.2)**: a **directory** is a set of contiguous keys sharing a prefix, the unit
  of data placement and movement. **Movedir** moves directories between Paxos groups in the background
  (e.g. to rebalance load or move data closer to users), without blocking ongoing operations. Applications
  can specify placement constraints (which regions, how many replicas). → **Topic 25** (range partitioning,
  rebalancing), **topic 48** (data placement for latency and fault tolerance).
- **Data model (2.3)**: schematized **semi-relational tables** with primary keys, a SQL-based query language,
  and **interleaving**: child tables declared `INTERLEAVE IN PARENT` are stored with their parent rows
  (e.g. a user's albums with the user), keeping related data in the same directory for locality and
  single-group transactions. → **Topic 15** (embedding/locality vs normalization), **topic 16** (schemas and
  schema changes, which Spanner performs as timestamped transactions without blocking), **topic 25**
  (choosing partition keys).

### 3.2 Section 3 — TrueTime

The paper's most famous contribution. `TT.now()` returns an **interval** `[earliest, latest]` guaranteed to
contain the true absolute time. Google keeps the uncertainty `ε` small (typically **1–7 ms** in the paper)
using **GPS receivers and atomic clocks** in every datacenter (time masters), with daemons polling several
masters and applying worst-case drift assumptions between polls. `TT.after(t)` and `TT.before(t)` answer
whether `t` has definitely passed.

The insight: instead of pretending clocks are synchronized (and silently being wrong, topic 27's
last-write-wins problem), **expose the uncertainty explicitly** and make the system wait it out when
correctness depends on it.
→ **Topic 27** (wall clocks, NTP skew, why timestamps normally can't order events, TrueTime and
commit-wait), **topic 26** (the time assumptions are part of the fault model: if `ε` is violated,
correctness is violated).

### 3.3 Section 4 — Concurrency control

**Timestamps and external consistency (4.1).**
- **Read-write transactions** use 2PL. At commit, the coordinator leader picks a commit timestamp `s` ≥
  `TT.now().latest`, then performs **commit wait**: it doesn't let clients see the commit until
  `TT.after(s)` is true, i.e. until `s` is guaranteed to be in the past everywhere. Expected wait is about
  `2ε`. Consequence: if transaction T1 commits before T2 starts (in real time), then `s1 < s2`. That's
  **external consistency**, equivalent to strict serializability (serializable + linearizable, topic 28's
  distinction between the two).
- **Paxos leader leases are disjoint** in time, so timestamps assigned by successive leaders are monotonic.
  → **Topic 31** (leases), **topic 30**.
- **Read-only transactions** and **snapshot reads** are **lock-free**: pick a timestamp (`TT.now().latest` for
  "read the latest"), and read the MVCC versions as of that timestamp from any replica that's caught up to
  it (each replica tracks a **safe time** `t_safe`, the timestamp up to which it's guaranteed to have all
  writes). → **Topic 22** (MVCC snapshots), **topic 21** (snapshot isolation for reads), **topic 23**
  (reading from followers safely, not just "possibly stale").

**Details (4.2).**
- Read-write transactions buffer writes at the client; reads acquire read locks at the group leaders.
  Deadlock avoidance uses **wound-wait**: an older transaction wounds (aborts) a younger one holding a lock it
  needs; a younger one waits for an older one. → **Topic 22** (deadlocks: prevention by ordering, here by
  transaction age).
- **Schema changes** are transactions with a future timestamp; operations depending on the schema
  synchronize on that timestamp, so a schema change across thousands of servers doesn't block traffic.
  → **Topic 47** (zero-downtime schema changes).

### 3.4 Evaluation, F1, and the CAP question

The evaluation and the F1 case study (Google's ad backend migrated from sharded MySQL, topic 25's pain of
manual resharding) show the latencies involved: commit wait and cross-region Paxos make writes cost
tens of milliseconds. → **Topic 29 (PACELC)**: Spanner chooses **consistency over latency** in normal
operation (EC) and **consistency over availability** during partitions (PC). Eric Brewer's 2017 note
"Spanner, TrueTime and the CAP Theorem" argues it's technically CP but *effectively CA* in practice, because
Google's private network makes partitions rare enough that availability exceeds five nines. → **Topic 29**
(classifying real systems and why labels mislead).

Later systems built on its ideas without atomic clocks: **CockroachDB** and **YugabyteDB** use hybrid
logical clocks with a bounded max offset and uncertainty intervals (restarting reads that fall in the
uncertainty window) instead of commit wait (topic 27). And Zanzibar (topic 54) stores its tuples in Spanner
and builds its zookies from Spanner timestamps.

### 3.5 Spanner mechanism map

| Mechanism | Paper section | Topic |
|---|---|---|
| Zones, multi-region placement | 2, 2.2 | 48 |
| Tablets, MVCC (key, timestamp) → value | 2.1 | 22 |
| WAL on Colossus | 2.1 | 17, 40 |
| Paxos group per tablet; leader leases | 2.1, 4.1 | 30, 31 |
| 2PL lock table; wound-wait | 2.1, 4.2 | 22 |
| 2PC across Paxos groups | 2.1, 4.2 | 34 |
| Directories, movedir, range partitioning | 2.2 | 25 |
| Interleaved tables, schemas | 2.3 | 15, 16 |
| TrueTime, commit wait | 3, 4.1 | 27 |
| External consistency | 4.1 | 28 |
| Lock-free snapshot reads at safe time | 4.1 | 21, 22, 23 |
| Non-blocking schema changes | 4.2 | 47 |
| CP/EC classification | evaluation, Brewer 2017 | 29 |

One trap from topic 49 shows up in Spanner's documentation: **monotonically increasing primary keys**
(timestamps, auto-increment) create a write hotspot on the last range-partitioned split. Spanner recommends
UUIDv4 keys or bit-reversed sequences.

---

## 4. Bigtable (2006) through the same lens

**Paper**: Chang, Dean, Ghemawat, Hsieh, Wallach, Burrows, Chandra, Fikes, Gruber, "Bigtable: A
Distributed Storage System for Structured Data", OSDI 2006.

**Dominant requirement**: store petabytes of structured data (web pages, Google Earth imagery, analytics)
across thousands of commodity servers, for both high-throughput batch processing and low-latency serving,
on top of the existing GFS. Bigtable is not about multi-region availability (Dynamo) or transactions
(Spanner): it's about **scale and throughput on a sorted, sparse map**, with consistency within a single
cluster made easy by having one server per tablet.

### 4.1 Section 2 — Data model

"A Bigtable is a sparse, distributed, persistent multi-dimensional sorted map", indexed by
`(row: string, column: string, timestamp: int64) → string`.
- **Rows** are kept in **lexicographic order**; contiguous row ranges are partitioned into **tablets**. Reads
  of short row ranges are efficient, so clients choose row keys for locality: the **Webtable** example uses
  reversed hostnames (`com.cnn.www`) so pages from the same domain are adjacent. → **Topic 25** (range
  partitioning; key design for locality vs hotspots), **topic 49** (sequential keys and hotspots on
  range-partitioned stores).
- **Every read or write under a single row key is atomic**, no matter how many columns: single-row
  transactions only. → **Topic 21** (atomicity scope), contrast with Spanner.
- **Column families** (e.g. `anchor:`, `contents:`) are the unit of access control and of storage
  configuration; columns within a family are unbounded and sparse. → **Topic 15** (the wide-column model,
  which Bigtable defined), **topic 20** (column-family grouping as a storage layout).
- **Timestamps** version each cell; garbage collection keeps the last `n` versions or versions newer than an
  age. → **Topic 22** (multi-versioning), **topic 27** (timestamps assigned by Bigtable or the client).

### 4.2 Section 4 — Building blocks

- **GFS** stores logs and data files. → **Topic 40** (distributed filesystems), **topic 53** (replicated chunk
  storage).
- **SSTable**: a persistent, ordered, immutable map from keys to values, with a block index loaded into
  memory, so a lookup is one disk seek. → **Topic 19** (SSTables), **topic 17** (from hash index to sorted
  structures).
- **Chubby**: a highly available, persistent **distributed lock service** with five replicas using **Paxos**;
  it provides small files usable as locks, with **sessions** that expire (leases). Bigtable uses it to ensure
  there's at most one active master, store the bootstrap location of data, discover tablet servers and
  finalize their deaths, and store schemas and ACLs. If Chubby is unavailable for an extended period,
  Bigtable becomes unavailable. → **Topic 31** (coordination services, leases, locks; ZooKeeper is the
  open-source descendant), **topic 30** (Paxos), **topic 32** (membership via a coordination service, the
  opposite of Dynamo's gossip).

### 4.3 Section 5 — Implementation

**5.1 Tablet location.** A **three-level hierarchy**, like a B-tree (topic 17) for locating data: a Chubby
file points to the **root tablet**, which indexes all **METADATA tablets**, which index all user tablets.
Clients **cache** tablet locations and only walk the hierarchy on a miss or a stale entry (at most ~6 round
trips if the cache is wrong). → **Topic 25** (request routing via a lookup table; client-side caching of
routing), **topic 11** (caching), **topic 17** (B+-tree-like indexing).

**5.2 Tablet assignment.** Each tablet is served by **exactly one tablet server** at a time. Tablet servers
create and hold an exclusive lock on a uniquely named file in a Chubby directory; the master monitors that
directory to discover servers. If a server loses its lock (e.g. a network partition makes its Chubby session
expire), it **stops serving** its tablets. The master periodically asks servers about their locks, and if a
server is unreachable, the master tries to acquire that server's lock itself: if it succeeds, Chubby is
alive and the server is cut off, so the master deletes the file and reassigns the tablets. The master itself
kills itself if its own Chubby session expires.
→ **Topic 31** (leases and the rule "stop acting when your lease may have expired"; fencing), **topic 26**
(slow vs dead), **topic 23** (single owner per partition avoids conflicts; failover), **topic 32** (failure
detection through a coordination service).

**5.3 Tablet serving.** The write path:

```
write -> commit log (in GFS, redo records) -> memtable (sorted, in memory)
memtable full -> minor compaction -> new SSTable in GFS
merging compaction: memtable + a few SSTables -> one SSTable
major compaction: all SSTables -> one SSTable, deleted data purged
read = merged view of memtable + SSTables
```

This is the **LSM-tree** in its canonical form, and it's one of the main sources for topic 19. Deletions
are written as special deletion entries (tombstones) and removed at major compaction.
→ **Topic 19** (memtable → SSTable → compaction; why sequential writes win; read amplification),
**topic 17** (commit log as WAL, crash recovery by replaying the log).

**5.4 Compactions** (minor, merging, major) are described in detail; they bound memory use, shorten log
replay on recovery, and reclaim deleted data. Schemas themselves live in Chubby. → **Topic 19**, **topic 31**.

### 4.4 Section 6 — Refinements

- **Locality groups**: column families grouped into separate SSTables, so a scan of one group doesn't read
  others; a group can be declared **in-memory**. → **Topic 20** (column-oriented layout; reading only the
  columns you need).
- **Compression**: per-locality-group, two-pass schemes (Bentley-McIlroy over long windows, then a fast
  compressor), achieving large ratios on Webtable because similar pages are stored together. → **Topic 20**
  (compression benefits from storing similar data together).
- **Caching**: a **scan cache** (key-value pairs, for temporal locality) and a **block cache** (SSTable
  blocks from GFS, for spatial locality). → **Topic 11** (where caches live; locality).
- **Bloom filters**: per locality group, to ask whether an SSTable might contain data for a given row/column
  pair, so most lookups for non-existent data never touch disk. → **Topic 50** and **topic 19** — the textbook
  use case.
- **Commit-log implementation**: one commit log **per tablet server** (not per tablet), to avoid many
  concurrent files in GFS and to get group-commit benefits. Recovery complication: tablets from a failed
  server are spread over many servers, each needing only its tablets' entries; so the log is **sorted by
  (table, row, sequence number)** in parallel first, making each server's read sequential. → **Topic 17**
  (group commit, log-structured recovery), **topic 02** (sequential vs random I/O).
- **Speeding up tablet recovery**: before moving a tablet, the source server does a minor compaction so the
  new server needn't replay the log. → **Topic 25** (rebalancing cost).
- **Exploiting immutability**: since SSTables are immutable, reads need no synchronization on them, and
  removing deleted data becomes **garbage collection of obsolete SSTables** (mark-and-sweep over METADATA).
  Tablet splits are fast because children share the parent's SSTables. → **Topic 19**, **topic 53**
  (immutability simplifies distributed storage).

### 4.5 Section 9 — Lessons

The paper's lessons are some of the most quoted in systems:
- Large distributed systems are vulnerable to **many kinds of failure**, not just the clean network
  partitions and fail-stop failures assumed in protocols: memory and network corruption, large clock skew,
  hung machines, extended asymmetric partitions, bugs in other systems (Chubby), GFS quota overflow,
  planned and unplanned maintenance. → **Topic 26** (fault models, partial failure), **topic 27** (clock skew).
- **Delay adding features until it's clear how they'll be used** (they skipped general multi-row
  transactions; most applications needed only single-row ones; later, Percolator and then Spanner added
  them).
- **System-level monitoring is essential** (they traced RPCs to find latency problems). → **Topic 46**
  (observability, distributed tracing: Google's Dapper came out of the same environment).
- **Simple designs** are worth the most: they report a complex, abandoned protocol for tablet server
  membership in favour of a simpler one relying on Chubby. → A theme across topics 30 and 31.

### 4.6 Bigtable mechanism map

| Mechanism | Paper section | Topic |
|---|---|---|
| Sorted sparse map (wide-column) | 2 | 15 |
| Row-range tablets; key design (reversed domains) | 2, 5 | 25, 49 |
| Single-row atomicity | 2 | 21 |
| Cell timestamps / versions | 2 | 22, 27 |
| GFS underneath | 4 | 40, 53 |
| SSTables | 4, 5.3 | 19 |
| Chubby (Paxos, locks, sessions) | 4, 5.2 | 30, 31 |
| Three-level location hierarchy, client cache | 5.1 | 25, 11, 17 |
| One server per tablet, lock-based assignment | 5.2 | 23, 26, 31 |
| Memtable, commit log, compactions | 5.3 | 17, 19 |
| Locality groups, compression | 6 | 20 |
| Scan and block caches | 6 | 11 |
| Bloom filters | 6 | 50 |
| Per-server commit log, sorted recovery | 6 | 17, 02 |
| Lessons on failures and monitoring | 9 | 26, 46 |

### 4.7 Three systems side by side

| | Dynamo | Bigtable | Spanner |
|---|---|---|---|
| Dominant requirement | Writes always succeed | Scale + throughput on sorted data | Global transactions with strong consistency |
| Data model | Key → opaque blob | Sorted wide-column map | Semi-relational tables, SQL |
| Partitioning | Hash (consistent hashing, vnodes) | Range (tablets) | Range (directories/splits) |
| Routing | Zero-hop, gossip-propagated ring | METADATA hierarchy + client cache | Location via zones/placement driver |
| Replication | Leaderless, N replicas, sloppy quorums | One serving server per tablet; durability from GFS replication | Paxos group per tablet, leader + leases |
| Consistency | Eventual; client-visible conflicts | Strong per row in one cluster | External consistency (strict serializability) |
| Transactions | None (single key) | Single-row | Multi-row, multi-shard (2PL + 2PC over Paxos) |
| Conflict handling | Vector clocks + app merge | Avoided (single owner per tablet) | Avoided (locks + consensus) |
| Clocks | Not trusted for correctness | Cell timestamps only | TrueTime is central to correctness |
| Membership/failures | Gossip, local failure detection | Chubby locks, master | Paxos, leases, zonemasters |
| Storage engine | Pluggable (BDB, MySQL) | LSM (memtable/SSTable) on GFS | B-tree-like files + WAL on Colossus |
| CAP / PACELC | PA/EL | Single-cluster, CP-ish (depends on Chubby/GFS) | PC/EC |

Reading this table well is the synthesis this topic is after. Each row is a topic, each column is a
coherent set of answers driven by one requirement. Change the requirement and the whole column changes.

---

## 5. Building your concept map

### 5.1 The exercise

On one page (paper, whiteboard, or a diagram tool), draw every topic as a node and an arrow from each topic
to the topics it **depends on** (needs in order to be understood). Then add a few **cross-links**:
mechanisms that reappear in a different form elsewhere (e.g. "WAL" in topics 17, 23, 37, 39, 51).

Do it **from memory first**, then compare with the skeleton below and with the topics' "How it connects"
sections. The gaps between your map and the skeleton are your weak spots.

### 5.2 A skeleton to check against

Major dependency spines (not every edge; your map should have more):

```
FOUNDATIONS
01 vocabulary ─► 02 latency & queueing ─► (everything performance-related: 11, 43, 44, 46)

NETWORK
03 TCP/UDP ─► 05 TLS ─► 06 HTTP versions ─► 07 API styles ─► 08 real-time
03 ─► 04 DNS ─► 09 load balancing ─► 10 proxies/gateways/mesh ─► 54 authn/authz (edge), 14 CDNs

CACHING
02 ─► 11 caching ─► 12 cache failure modes ─► 50 probabilistic (Bloom vs penetration)
11 ─► 13 consistent hashing ─► 14 CDNs, 24 leaderless, 25 partitioning

STORAGE (single node)
15 data models ─► 16 encoding/evolution
17 B-tree + WAL ─► 18 indexes ─► 51 search, 52 geospatial, 49 IDs (index locality)
17 ─► 19 LSM ─► 50 (Bloom filters), 51 (segments)
15, 19 ─► 20 column storage/OLAP
21 transactions/isolation ─► 22 concurrency control (MVCC, 2PL, SSI)

SCALING DATA
17 (WAL) ─► 23 single-leader ─► 24 multi-leader & leaderless ─► 33 conflicts/CRDTs
13 ─► 25 partitioning ─► 51 (shards), 52 (geo partitioning), 53 (object store metadata)

DISTRIBUTED SYSTEMS
26 faults ─► 27 clocks ─► 28 consistency models ─► 29 CAP/PACELC
26, 28 ─► 30 consensus/Raft ─► 31 coordination, leases, locks ─► 32 membership & gossip
27 ─► 33 conflicts & CRDTs
21, 30 ─► 34 2PC (+ consensus) ─► 35 sagas

MOVING DATA
36 queues ─► 37 Kafka (log) ─► 38 delivery semantics & idempotency ─► 39 outbox & CDC
40 batch ─► 41 streaming ─► 42 event sourcing & CQRS
17 (log) ─► 37, 39, 42           (the log as the unifying abstraction)

RELIABILITY
26 ─► 43 timeouts & retries ─► 44 circuit breakers, load shedding ─► 45 rate limiting
02, 01 ─► 46 observability ─► 47 safe deployments ─► 48 fault isolation & DR

BUILDING BLOCKS
25, 27 ─► 49 IDs
19, 12 ─► 50 probabilistic structures
17, 25 ─► 51 search, 52 geospatial
24, 40 ─► 53 object storage, erasure coding, Merkle trees
05, 10 ─► 54 authn/authz

SYNTHESIS
13, 24, 27, 32, 33, 53 ─► Dynamo
17, 19, 25, 30, 31, 40, 50 ─► Bigtable
21, 22, 27, 28, 30, 31, 34 ─► Spanner
```

### 5.3 Cross-cutting ideas to draw as links

The concepts that recur in different costumes are the ones worth seeing explicitly:

| Idea | Appears in topics |
|---|---|
| **The append-only log** | 17 (WAL), 19 (LSM), 23 (replication log), 30 (Raft log), 37 (Kafka), 39 (CDC), 42 (event sourcing), 51 (translog) |
| **Immutability simplifies distribution** | 19 (SSTables), 37, 42, 49 (content IDs), 51 (segments), 53 (objects, content addressing) |
| **Quorums / majorities and overlap** | 24 (`R + W > N`), 30 (majority commit), 31 (lock quorums) |
| **Leases and time-bounded ownership** | 31, 49 (worker IDs), Bigtable tablet locks, Spanner leader leases |
| **Staleness vs coordination** | 11 (TTL caches), 23 (replica lag), 28, 29, 54 (token revocation) |
| **Hashing to spread load / summarize** | 09 (hash LB), 13, 25, 50, 53 (Merkle, content hashes) |
| **Idempotency makes retries safe** | 07 (methods), 38, 43, 49 (client IDs), 53 (content-addressed writes) |
| **Fan-out amplifies the tail** | 02, 25 (scatter-gather), 43 (hedging), 51 (shard fan-out), 53 (degraded reads) |
| **Filter then refine** | 19/50 (Bloom then read), 51 (retrieve then re-rank), 52 (cells then exact distance) |
| **Single owner avoids conflicts; many writers need merging** | 23 vs 24, 33, Bigtable vs Dynamo |

---

## 6. Self-assessment

### 6.1 The test

For each topic, the plan's test of understanding is: **can you explain it to someone else, without notes,
including why it exists and what goes wrong without it?** Score yourself honestly, topic by topic:

| Score | Meaning |
|---|---|
| 3 | I can explain the mechanism step by step, its trade-offs, and its failure modes, and connect it to other topics, without notes. |
| 2 | I can explain the idea and the main trade-off, but I'm shaky on a step or a failure mode. |
| 1 | I recognise the terms; I couldn't explain the mechanism. |
| 0 | I don't remember it. |

Quick probes by part (if you can't answer one in two minutes out loud, score that topic ≤ 2):

- **Part 1**: Why do averages mislead about latency? What does Little's Law say about a system with 100 ms
  latency at 1,000 req/s?
- **Part 2**: Why does HTTP/2 still suffer head-of-line blocking, and how does QUIC fix it? Why do DNS changes
  propagate slowly?
- **Part 3**: Walk through a cache stampede and two fixes. What moves when a node joins a consistent-hash ring?
- **Part 4**: Why does a B-tree stay 3–4 levels deep? Which isolation level prevents write skew, and how does
  Postgres implement it?
- **Part 5**: What anomalies does replication lag cause? Why don't sloppy quorums guarantee reading the latest
  write? Local vs global secondary indexes?
- **Part 6**: Why can't you distinguish slow from dead? Why do vector clocks detect concurrency where Lamport
  clocks can't? Walk through a Raft leader election and why a stale leader can't commit. Why do
  distributed locks need fencing tokens? Why does 2PC block?
- **Part 7**: What does "exactly-once" actually mean? How does the outbox pattern avoid dual writes? What's a
  watermark?
- **Part 8**: Why does retrying make outages worse, and what are the fixes? Explain token bucket. What are RPO
  and RTO?
- **Part 9**: Why do random UUIDs hurt B-trees? Size a Bloom filter for 1%. Explain BM25's `k1`. Why search 9
  geohash cells? Why does erasure coding raise repair traffic? Why are JWTs hard to revoke?
- **Part 10**: Explain to someone why Dynamo, Bigtable and Spanner made such different choices.

### 6.2 Your five weakest concepts

After scoring, list your **five lowest-scoring concepts** (concepts, not whole topics: "write skew" rather than
"topic 21"). For each, write down:

1. **The concept** and the topic number.
2. **What exactly you couldn't explain** (the mechanism step? a failure mode? why it exists?).
3. **Where to revisit**: the section of that topic's lesson, plus one supplement from its "Further reading"
   (e.g. a DDIA chapter, the relevant paper section from this topic, a lecture).
4. **A concrete re-test**: the question from that topic's question list you'll answer out loud after revisiting.

Example entry:

```
Concept:    Sloppy quorums vs strict quorums (topic 24)
Gap:        Couldn't explain why R + W > N fails to guarantee freshness with hinted handoff
Revisit:    Topic 24 section on sloppy quorums; Dynamo paper §4.6; DDIA ch. 5 "Sloppy Quorums and Hinted Handoff"
Re-test:    Topic 24 question on a write landing on stand-in nodes and a read hitting home nodes
```

Revisit in a week, re-score, and repeat. Weak spots usually cluster (e.g. clocks, consistency models and
CAP together), so if several of your five are in Part 6, re-read topics 26–29 in order rather than jumping.

### 6.3 Going further

More papers that reward the same lens, in rough order of approachability: GFS (2003), MapReduce (2004),
Chubby (2006), ZooKeeper (2010), Kafka (2011), Raft (2014), Facebook's TAO (2013), Amazon Aurora (2017),
DynamoDB (2022). For each, fill in the side-by-side table of section 4.7 as a new column.

---

## 7. How it connects

This topic connects to everything by design. The densest links:
- **Topics 13, 24, 27, 32, 33, 53**: the Dynamo toolkit (consistent hashing, quorums, vector clocks, gossip,
  conflict resolution, Merkle trees).
- **Topics 17, 19, 25, 30, 31, 40, 50**: the Bigtable toolkit (WAL, LSM, range partitioning, Chubby/Paxos,
  GFS, Bloom filters).
- **Topics 21, 22, 27, 28, 30, 34**: the Spanner toolkit (transactions, MVCC/2PL, TrueTime, external
  consistency, Paxos, 2PC over consensus).
- **Topics 29 and 48**: how to classify the three and what each gives up.
- **Topics 02, 44, 46**: the operational lessons (tail latency SLAs, admission control, monitoring).

---

## Common misconceptions

- "Dynamo and DynamoDB are the same system." → DynamoDB took the name and the key-value idea; its
  architecture uses a leader and Multi-Paxos per partition, unlike Dynamo's leaderless design.
- "Dynamo provides strong consistency when `R + W > N`." → Sloppy quorums and hinted handoff break the
  overlap guarantee; Dynamo is explicitly eventually consistent and exposes conflicting versions.
- "Spanner beats CAP." → It's a CP system whose availability is high because partitions are rare on
  Google's network; during a partition, the minority side can't commit.
- "TrueTime means Google's clocks are perfectly synchronized." → TrueTime exposes bounded uncertainty and
  Spanner waits it out (commit wait); correctness depends on the bound holding.
- "Bigtable replicates by running multiple tablet servers per tablet." → Each tablet has exactly one serving
  server at a time; durability comes from GFS replication of its logs and SSTables (cross-cluster
  replication was added later).
- "Reading a paper means reading it front to back once." → Skim for structure, read for mechanisms, then
  read critically for what was given up.
- "A concept map is just the table of contents." → Its value is in the dependency arrows and the cross-links
  between mechanisms that recur in different topics.

---

## Further reading (supplements)

- DeCandia et al., "Dynamo: Amazon's Highly Available Key-value Store" (SOSP 2007).
- Corbett et al., "Spanner: Google's Globally-Distributed Database" (OSDI 2012); Eric Brewer, "Spanner, TrueTime and the CAP Theorem" (2017).
- Chang et al., "Bigtable: A Distributed Storage System for Structured Data" (OSDI 2006).
- Elhemali et al., "Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL Database Service" (USENIX ATC 2022).
- Lakshman and Malik, "Cassandra: A Decentralized Structured Storage System" (2010).
- S. Keshav, "How to Read a Paper" (ACM SIGCOMM CCR, 2007).
- DDIA chapters 5–9 (the background for nearly every mechanism in these papers).
- MIT 6.5840 (Distributed Systems) lecture notes and videos on Spanner and other papers.

---

## Questions

1. What single requirement explains most of Dynamo's design, and name three mechanisms that exist because of it.
2. Dynamo resolves conflicts "at read time" rather than "at write time". What does each option mean, and what does Dynamo's choice cost the application developer?
3. Explain why virtual nodes were introduced in Dynamo, citing both problems the paper names with basic consistent hashing.
4. A Dynamo cluster uses (N, R, W) = (3, 2, 2). Node A, a home replica for key k, is unreachable during a write. Trace the write, the hint, and a later read that hits A after it recovers but before the hint is delivered. What does the client see?
5. Why did Dynamo's shopping cart sometimes resurrect deleted items, and which later technique from topic 33 addresses it?
6. Why does Dynamo use Merkle trees in addition to hinted handoff and read repair? What problem with them did partitioning strategy 3 (fixed, equal-sized partitions) relieve?
7. Dynamo's membership changes are explicit, but failure detection is local. Why separate the two?
8. List three concepts from the plan that Dynamo deliberately does *not* use, and why.
9. In Spanner, how does running 2PC over Paxos groups remove 2PC's classic blocking problem?
10. Explain commit wait. Why does it guarantee that if T1 commits before T2 starts, T1's timestamp is smaller?
11. Why can Spanner serve lock-free read-only transactions from follower replicas without returning stale data?
12. In Bigtable, why does a tablet server stop serving when it loses its Chubby lock, and what failure is this protecting against?
13. Trace a Bigtable write and a read through the memtable, commit log and SSTables. Which refinements make reads fast?
14. Using the side-by-side table, explain why Dynamo is hash-partitioned and Bigtable/Spanner are range-partitioned.
15. Classify Dynamo and Spanner using PACELC and justify each letter.

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Writes must always succeed ("always writeable"), even during disk, node, network and datacenter failures. Mechanisms that follow: leaderless replication with sloppy quorums and hinted handoff (any healthy node can take a write), vector clocks with read-time reconciliation (accept concurrent writes and sort them out later), and anti-entropy with Merkle trees (repair the divergence that accepting writes everywhere creates). Gossip-based decentralized membership also follows (no central component that can block writes).

**2.** Resolving at write time means a write is rejected unless replicas can agree on it (requires coordination; failures reduce availability). Resolving at read time means writes are always accepted, divergent versions may exist, and reads must reconcile them. Dynamo chose read time; the cost is that `get` can return multiple sibling versions and the application must merge them semantically (and accept anomalies such as resurrected deletes in the cart).

**3.** Basic consistent hashing assigns each node one random position, giving (a) non-uniform data and load distribution (random arc sizes) and (b) no way to account for heterogeneous node capacity. Virtual nodes give each physical node many positions: load spreads evenly, a failed node's load disperses across many nodes rather than one successor, a joining node takes load from many nodes, and stronger machines can get more virtual nodes.

**4.** The coordinator walks the preference list and finds A unreachable, so it sends the write to the first N healthy nodes: B, C and stand-in D (with a hint "belongs to A"). W = 2 acks arrive; success. A recovers but hasn't received the hint. A read with R = 2 might contact A and B: A returns the old version, B the new one. The new version's vector clock descends from the old, so the coordinator returns the new version (syntactic reconciliation) and read-repairs A. But if the read had hit A and some other node that also missed the write (possible with sloppy quorums), the client would see the old value: R + W > N doesn't guarantee overlap when the write landed on a stand-in.

**5.** Divergent cart versions were merged by union. If one branch removed an item and the concurrent branch still contained it, the union contains it again: the deletion is lost. An observed-remove set (OR-set) CRDT tracks unique tags for each add and removes only observed tags, so a remove beats the adds it saw while concurrent adds survive, merging correctly without resurrecting removed items.

**6.** Hinted handoff fails if the node holding the hint dies before delivering it; read repair only fixes keys that are read. Merkle-tree anti-entropy finds and repairs any divergence in the background, including rarely read keys, while transferring little data. Problem: trees are per key range, and with random tokens, ranges change whenever nodes join or leave, forcing expensive tree recomputation. With fixed, equal-sized partitions, the ranges never change (only their placement), so trees remain valid across membership changes.

**7.** At Amazon most outages are transient; treating a temporary failure as a membership change would trigger expensive data movement and rebalancing that then has to be undone. So permanent changes are explicit administrative actions propagated by gossip, while temporary failures are handled locally: a node that can't reach a peer routes around it (hinted handoff) and retries later. No globally consistent failure view is needed.

**8.** Any three of: single-leader replication (leader failover blocks writes), consensus such as Paxos/Raft (needs a majority; minority partitions can't write), distributed transactions/2PC (blocking and unnecessary for single-key access), linearizability (coordination per operation), a central coordination service (a central dependency), wall-clock last-write-wins (silently loses writes), security (trusted environment assumed).

**9.** In classic 2PC, if the coordinator crashes after participants have voted yes, they're stuck holding locks until it recovers. In Spanner, each participant and the coordinator is a Paxos group: the 2PC state (prepared, commit decision) is written to the group's replicated log. If a leader fails, the group elects a new leader that has that state and continues the protocol. The protocol only blocks if a whole group loses its majority.

**10.** The coordinator assigns commit timestamp s ≥ TT.now().latest, then waits until TT.after(s) (the true time is certainly past s) before releasing locks and acknowledging. So by the time any client learns T1 committed, real time is past s1. Any T2 that starts afterwards gets s2 ≥ TT.now().latest at a later moment, which is ≥ the true time at that moment, which is > s1. Hence real-time order implies timestamp order: external consistency.

**11.** All data is multi-versioned, and each replica tracks a safe time t_safe: the timestamp up to which it's guaranteed to have applied all writes (from its Paxos state and pending prepared transactions). A read-only transaction picks a timestamp (e.g. TT.now().latest), and any replica whose t_safe has reached that timestamp can serve a consistent snapshot without locks; otherwise it waits briefly. This gives linearizable reads from followers.

**12.** If a tablet server is partitioned from Chubby, its session (lease) expires and the master may reassign its tablets to another server. If the old server kept serving, two servers would serve the same tablet: split brain, with conflicting writes. So a server that can't confirm its lock stops serving, just as topic 31's lease rule says ("stop acting before your lease might have expired").

**13.** Write: append to the tablet server's commit log in GFS (durability), then insert into the in-memory sorted memtable; when full, a minor compaction writes it as an immutable SSTable; merging and major compactions combine SSTables and purge deletions. Read: merge the memtable with the relevant SSTables (newest first). Fast reads come from in-memory SSTable block indexes, Bloom filters to skip SSTables that don't contain the row/column, scan and block caches, locality groups (read only needed column families), and compaction keeping the number of SSTables small.

**14.** Dynamo's access pattern is single-key get/put; hash partitioning spreads load evenly with no hotspots and supports incremental scaling via consistent hashing; range scans weren't needed. Bigtable and Spanner need efficient range scans and locality (all pages of a domain, a user's interleaved rows, SQL range queries), which requires keeping keys sorted, so they range-partition and split/move ranges dynamically, accepting hotspot risk from sequential keys.

**15.** Dynamo: PA/EL. During a partition it chooses availability (sloppy quorums, writes accepted anywhere); in normal operation it chooses latency (R, W < N; returns after the fastest replicas, eventual consistency). Spanner: PC/EC. During a partition, a minority side can't reach a Paxos majority and can't commit (consistency over availability); in normal operation it pays latency for consistency (cross-replica Paxos, commit wait).

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Three passes for reading a paper?
**A:** Skim for structure; read for mechanisms and the requirement each serves; read critically for what was given up.

**Q:** Dynamo's dominant requirement?
**A:** Writes must always succeed, even under failures.

**Q:** Dynamo partitioning technique (topic)?
**A:** Consistent hashing with virtual nodes (topic 13).

**Q:** Two problems vnodes solved in Dynamo?
**A:** Uneven load from random positions, and no way to reflect heterogeneous node capacity.

**Q:** What is a Dynamo preference list?
**A:** The nodes responsible for a key: coordinator plus successors, skipping to get N distinct physical nodes.

**Q:** Dynamo's typical (N, R, W)?
**A:** (3, 2, 2).

**Q:** How does Dynamo track causality (topic)?
**A:** Vector clocks per version, passed as the get/put context (topic 27).

**Q:** Syntactic vs semantic reconciliation?
**A:** Syntactic: the system sees one version descends from another. Semantic: concurrent versions merged by the application.

**Q:** Why did deleted cart items reappear?
**A:** Siblings were merged by union, undoing removals; OR-set CRDTs fix it.

**Q:** What does hinted handoff do?
**A:** A stand-in node stores a write for an unreachable replica and delivers it on recovery.

**Q:** Why doesn't R + W > N guarantee freshness in Dynamo?
**A:** Sloppy quorums let writes land on stand-ins, so read and write sets may not overlap.

**Q:** Dynamo's permanent-failure repair?
**A:** Anti-entropy using Merkle trees per key range (topic 53).

**Q:** Dynamo membership and failure detection?
**A:** Explicit admin changes propagated by gossip; seed nodes; local request-driven failure detection.

**Q:** Dynamo's partitioning strategy 3?
**A:** Q fixed equal-sized partitions, Q/S per node; whole partitions move (topic 25).

**Q:** What fraction of cart requests saw one version?
**A:** 99.94%; divergence mostly came from concurrent writers, not failures.

**Q:** What did client-driven coordination gain?
**A:** Removed a hop and the load balancer, cutting 99.9th percentile latency by ≥ 30 ms.

**Q:** Dynamo admission controller?
**A:** Gives background tasks time only when foreground latency is healthy (topic 44).

**Q:** Is DynamoDB Dynamo?
**A:** No; DynamoDB uses a leader and Multi-Paxos per partition.

**Q:** Spanner's dominant requirement?
**A:** Globally distributed, externally consistent transactions.

**Q:** Spanner replication unit?
**A:** A Paxos group per tablet, with a leased leader.

**Q:** How does Spanner make 2PC non-blocking in practice?
**A:** Participants and coordinator are Paxos groups, so 2PC state survives leader failure.

**Q:** What does TrueTime return?
**A:** An interval [earliest, latest] containing true time; ε typically 1–7 ms.

**Q:** What is commit wait?
**A:** Waiting until the commit timestamp is certainly past before acknowledging; ~2ε.

**Q:** What is external consistency?
**A:** If T1 commits before T2 starts, T1's timestamp is smaller (strict serializability).

**Q:** How are Spanner read-only transactions served?
**A:** Lock-free MVCC snapshot reads at a timestamp, from any replica whose safe time has caught up.

**Q:** Spanner deadlock avoidance?
**A:** Wound-wait by transaction age.

**Q:** Bigtable data model?
**A:** Sparse, sorted map (row, column, timestamp) → string; single-row atomicity.

**Q:** Bigtable's lock service?
**A:** Chubby: Paxos-replicated, sessions/leases; one master, tablet server liveness, bootstrap location.

**Q:** Bigtable tablet location lookup?
**A:** Chubby file → root tablet → METADATA tablets → user tablet, with client caching.

**Q:** Bigtable write path?
**A:** Commit log → memtable → minor compaction to SSTable → merging/major compactions.

**Q:** Why does a Bigtable server stop serving on lock loss?
**A:** To prevent two servers serving the same tablet after reassignment (split brain).

**Q:** Bigtable read accelerators?
**A:** Bloom filters, block and scan caches, locality groups, compactions.

**Q:** PACELC for Dynamo and Spanner?
**A:** Dynamo PA/EL; Spanner PC/EC.

**Q:** Concept map test?
**A:** Draw every topic with dependency arrows and cross-links from memory, then compare.

**Q:** Self-assessment test per topic?
**A:** Explain it without notes, including why it exists and what goes wrong without it.
