# 25 — Partitioning (sharding)

> Part 5 — Scaling Data · Prerequisites: topics 13, 18, 23

Replication (topics 23 and 24) puts a *copy* of all the data on each machine. That helps with availability and
reads, but every replica still has to store the whole dataset and apply every write. When the data is too large for
one machine, or the write rate is too high for one machine, you need to split the data itself so that each machine
holds and processes only a part. That is **partitioning**, also called **sharding**. The names vary by system:
a *shard* in MongoDB and Elasticsearch, a *region* in HBase, a *tablet* in Bigtable and Spanner, a *vnode* in
Cassandra and Riak, a *partition* in Kafka. The idea is the same. The difficulties are all about *how* you split:
splitting so that load is even, finding data again, answering queries that don't fit the split, and moving data
when the cluster changes.

By the end of this topic you should be able to:
1. Explain why partitioning is needed and how it combines with replication.
2. Compare key-range and hash partitioning, and explain the hotspot vs range-query trade-off.
3. Explain why even hash partitioning can't fix a single hot key, and describe mitigations.
4. Contrast local (document-partitioned) and global (term-partitioned) secondary indexes.
5. Describe the three main rebalancing strategies and why `hash mod N` is not one of them.
6. Explain the three approaches to request routing and the role of a coordination service.

---

## 1. Why partition?

### 1.1 The goal: scalability

Each piece of data (a row, document, or key-value record) belongs to **exactly one partition**. Each partition is a
small database of its own, and different partitions live on different nodes.

```
                       whole dataset (too big / too busy for one machine)
     ┌──────────────────────────────────────────────────────────────────────────┐
     │ users A–F │ users G–L │ users M–R │ users S–Z │
     └─────┬─────┴─────┬─────┴─────┬─────┴─────┬─────┘
           v           v           v           v
        node 1      node 2      node 3      node 4
```

The benefits:
- **Storage:** a dataset of 40 TB can be spread over 20 machines with 2 TB each.
- **Write throughput:** writes to different partitions go to different machines, so write capacity grows with the
  number of nodes. This is the thing replication alone cannot do.
- **Query throughput:** queries that touch a single partition run independently on separate nodes. Large queries can
  be parallelised across partitions (the basis of MPP analytics databases, topic 20).

The goal is to spread data and load **evenly**. If partitioning is unfair, some partitions get more data or queries
than others; this is called **skew**. A partition with disproportionately high load is a **hotspot**. In the
extreme, all the load lands on one partition and 19 of 20 nodes sit idle: you have paid for 20 machines and get the
throughput of one.

### 1.2 When not to partition

Partitioning adds real complexity: cross-partition queries, cross-partition transactions (topic 34), rebalancing,
routing. A single modern machine with replicas handles a surprisingly large workload, and vertical scaling plus
read replicas plus caching (topic 11) often goes a long way. Partition when the data or write volume genuinely
requires it, and choose the partition key with care, because it is very hard to change later.

### 1.3 Partitioning combined with replication

Partitioning and replication are orthogonal and are almost always used together: each partition is replicated to
several nodes for fault tolerance, and each node stores several partitions. With single-leader replication, each
partition has its own leader, and leaders for different partitions are spread across nodes:

```
               node 1            node 2            node 3            node 4
 partition 1   LEADER            follower          follower
 partition 2                     LEADER            follower          follower
 partition 3   follower                            LEADER            follower
 partition 4   follower          follower                            LEADER

Each node is leader for some partitions and follower for others,
so write load (which goes to leaders) is spread across all nodes.
```

Everything from topic 23 applies **per partition**: each partition has its own replication log, its own lag, and its
own failover. A leaderless store (topic 24) is similar: the N replicas of each key are the N nodes following its
position on the hash ring. One consequence noted in topic 23: there is no global ordering across partitions, which
is why consistent-prefix anomalies arise.

The choice of partitioning scheme is mostly independent of the choice of replication scheme, so the rest of this
topic ignores replication for simplicity.

---

## 2. How to assign keys to partitions

Assume a key-value data model: you always access a record by its primary key. How do you decide which partition a
key belongs to? The naive answer is random assignment, which spreads data perfectly but makes it impossible to find a
record without querying every node. You need a deterministic function from key to partition. There are two main
families.

### 2.1 Partitioning by key range

Assign each partition a **continuous range of keys**, from some minimum to some maximum, like the volumes of a
printed encyclopedia (A–Ba, Bb–Ce, ...). If you know the boundaries, you know which partition holds a key.

```
partition 1: [ "a"      , "fz"     )
partition 2: [ "g"      , "mz"     )
partition 3: [ "n"      , "sz"     )
partition 4: [ "t"      , +∞       )
```

- The ranges are not necessarily even: they must adapt to the actual data distribution. If many keys start with "s",
  that range needs to be split more finely. Boundaries are chosen manually or automatically by the database.
- Within each partition, keys are kept **sorted** (e.g. in an LSM-tree or B-tree, topics 17 and 19).

The great advantage is **efficient range scans**. Because adjacent keys live together, "all sensor readings between
10:00 and 11:00" or "all orders for customer 42" (with a compound key like `customer_id#order_date`) is a read of one
contiguous range on one (or a few) partitions. Used by Bigtable, HBase, Spanner, CockroachDB, TiKV, and MongoDB with
ranged sharding.

The disadvantage is **hotspots from access patterns**. The classic case is a timestamp key:

```
key = timestamp of a sensor reading

partition for "yesterday":  idle
partition for "today":      ALL writes go here  ← hotspot
```

Every write lands in the partition holding the current time while others sit idle. Monotonically increasing keys
(timestamps, autoincrement IDs) always do this with range partitioning. The fix is to prefix the key with something
that distributes writes, e.g. `sensor_name#timestamp`. Writes then spread across sensors, at the cost that "all
sensors between 10:00 and 11:00" now requires one range query per sensor.

### 2.2 Partitioning by hash of key

Apply a **hash function** to the key and assign each partition a range of **hashes** instead of a range of keys.

```
hash("alice")   = 0x1A..   → partition 1   [0x00.., 0x40..)
hash("alicf")   = 0xC3..   → partition 4   [0xC0.., 0xFF..]
hash("bob")     = 0x7E..   → partition 2   [0x40.., 0x80..)

Similar keys scatter: the hash destroys ordering.
```

A good hash function turns skewed input into a uniform distribution: even sequential timestamps are scattered evenly
across partitions. The hash doesn't need to be cryptographically strong, but it must be **stable across processes
and versions**; e.g. Java's `Object.hashCode()` or Python's randomised string `hash()` are unsuitable because the
same key may hash differently in another process. Cassandra uses Murmur3, MongoDB uses MD5 for hashed shard keys.

The cost: **range queries become inefficient**. Adjacent keys are now scattered over all partitions, so a range
query must be sent to every partition (MongoDB with hash-based sharding does exactly this), and some systems don't
support primary-key range queries at all on hash partitioning (Riak, Voldemort, Couchbase).

### 2.3 The trade-off, and a compound-key compromise

| | Key-range partitioning | Hash partitioning |
|---|---|---|
| Distribution of keys | Depends on data; boundaries must adapt | Uniform (for distinct keys) |
| Hotspots from sequential keys | Yes (timestamps, autoincrement) | No |
| Range queries on the key | Efficient: one or few partitions | Scattered: query all partitions |
| Locality of related keys | Kept | Destroyed |
| Typical users | HBase, Bigtable, Spanner, CockroachDB | Cassandra, DynamoDB, Riak, Redis Cluster |

Cassandra offers a middle ground with a **compound primary key**: the first part (the *partition key*) is hashed to
pick the partition; the remaining columns (*clustering columns*) determine the sorted order **within** the partition.

```
PRIMARY KEY ((user_id), posted_at)

hash(user_id) → which partition
within that partition, rows for this user are sorted by posted_at

"All posts by user 42 between June and August" → one partition, one sorted range scan.
"All posts by anyone between June and August"   → not efficient (scatter).
```

This fits one-to-many relationships well: pick the entity whose data you read together (the user, the conversation,
the device) as the partition key, and the ordering inside it as clustering columns. DynamoDB's partition key + sort
key works the same way.

A note on terminology: **consistent hashing** (topic 13) is one specific way of choosing hash-range boundaries so
that few keys move when nodes are added. Many databases using "hash partitioning" actually use a fixed number of
partitions (section 4.2) rather than a ring, but the goal of minimising data movement is the same.

---

## 3. Skewed workloads and hot keys

### 3.1 The celebrity problem

Hashing distributes **keys** evenly. It cannot distribute load when **one key** receives a large fraction of all
requests, because every request for that key hashes to the same partition.

```
Social network, posts partitioned by hash(user_id)

hash(ordinary_user_1) → partition 3:    20 writes/s
hash(ordinary_user_2) → partition 7:    15 writes/s
hash(celebrity)       → partition 5: 50,000 writes/s (comments, likes on their posts)

Partition 5's node is saturated; the rest of the cluster is idle.
Adding nodes doesn't help: that key still lives on one partition.
```

This is the **celebrity** or **hot key** problem. Examples beyond social media: a flash-sale product, a viral video's
view counter, a global "configuration" row everyone reads, a tenant that is 100x larger than all others in a
multi-tenant system.

### 3.2 Mitigations

Most databases can't automatically compensate for a highly skewed key, so it is usually the application's job.

**Split the hot key (key salting).** Append a random suffix, e.g. a number from 00 to 99, so writes to one logical
key spread over 100 physical keys on different partitions.

```
write like for post P:  key = "P#" + random(0..99)   → spread over up to 100 partitions
read like count for P:  read "P#00" .. "P#99" and SUM  → 100 reads instead of 1
```

The costs: reads must fetch and combine all the sub-keys, and you need bookkeeping to know **which** keys are split
(splitting every key multiplies read cost everywhere for no benefit). This only works for data that can be combined:
counters, append-only lists, sets.

**Other approaches:**
- **Cache hot reads** (topic 11) in front of the database, possibly with a small in-process cache on each app server
  for the very hottest keys; this handles read-heavy hot keys well (and topic 12 covers hot-key cache failure modes).
- **Buffer and aggregate writes:** accumulate increments in memory or a stream for a second and write one batched
  update (topic 41).
- **Give the hot entity its own partition or dedicated hardware** (common for very large tenants).
- **Some systems split hot ranges automatically:** e.g. DynamoDB's adaptive capacity and "split for heat", and
  range-partitioned stores that split a range by load rather than size. A single key still can't be split by
  the database.

Detecting hot keys requires per-partition and ideally per-key metrics; hotspots often show up only as one node with
much higher CPU or latency than its peers.

---

## 4. Secondary indexes on partitioned data

### 4.1 The problem

So far, every query uses the primary key, so you can compute the partition. But many queries search by something
else: "all red cars", "all orders with status = pending", "all users in city X". Secondary indexes (topic 18) are the
bread and butter of relational databases and search engines, and they **don't map neatly to partitions**: the
records matching "color = red" are spread over every partition. There are two main approaches.

### 4.2 Local indexes (document-partitioned): scatter-gather reads

Each partition maintains a secondary index covering **only its own documents**.

```
Cars partitioned by car_id

partition 0 (ids 0–499)                 partition 1 (ids 500–999)
 data: 191 red Honda                     data: 515 red Ford
       214 black Dodge                         768 silver Audi
       306 red Ford                            893 silver VW
 local index:                            local index:
   color:red    → [191, 306]               color:red    → [515]
   color:black  → [214]                    color:silver → [768, 893]
   make:Ford    → [306]                    make:Ford    → [515]

Query "color = red":
   router ──> partition 0 → [191, 306]
          ──> partition 1 → [515]
          ──> partition N → ...
   combine results  ("scatter-gather")
```

- **Writes are cheap and local:** adding a record updates only the index on the partition that holds it, in the same
  local transaction. The index is always consistent with the data.
- **Reads by secondary key are expensive:** the query must go to **all** partitions and the results combined. Even
  with parallel requests, the slowest partition determines latency, so **tail latency amplification** applies: if
  each partition has a 1% chance of being slow, a query over 100 partitions is slow most of the time (topic 02).
  Adding partitions adds work to every such query rather than spreading it.

Used by MongoDB, Cassandra, Elasticsearch (each shard is a self-contained Lucene index), Riak, and VoltDB. Systems
recommend designing the partition key so that common queries can be served from one partition when possible (e.g.
partition by user so "this user's red cars" hits one partition).

### 4.3 Global indexes (term-partitioned)

Instead of each partition indexing its own data, build **one global index** covering all data, and **partition the
index itself**, by the indexed term rather than by the record's key.

```
Data partitioned by car_id (as before).
Global index on color, partitioned by term:

 index partition A (colors a–r):          index partition B (colors s–z):
   color:black  → [214]                     color:silver → [768, 893]
   color:red    → [191, 306, 515]           color:yellow → [...]

Query "color = red":  go only to index partition A → [191, 306, 515]
                      then fetch those records from data partitions (by car_id)
```

The index can be partitioned by term range (good for range queries like "price between 10k and 20k") or by hash of
the term (better load distribution).

- **Reads are efficient:** a query for one term goes to one index partition rather than all of them.
- **Writes are slower and more complicated:** writing a single record may update **several index partitions** (one
  for each indexed term: color, make, ...), which may live on different nodes from the data and from each other.
  Keeping them atomically consistent would need a distributed transaction (topic 34).
- So in practice global secondary indexes are usually **updated asynchronously**: a read of the index shortly after a
  write may not reflect it. DynamoDB global secondary indexes, for instance, are eventually consistent and typically
  propagate within a fraction of a second, but can lag more under faults. (Some systems, like Spanner and
  CockroachDB, do update global indexes transactionally, paying the cost of distributed commits.)

### 4.4 Comparison

| | Local (document-partitioned) | Global (term-partitioned) |
|---|---|---|
| Where the index for a record lives | Same partition as the record | Index partition determined by the term |
| Write cost | One partition, local, synchronous | May touch several partitions; usually async |
| Read by secondary key | Scatter-gather to all partitions | One (or few) index partitions, then fetch records |
| Index consistency | Consistent with data | Often eventually consistent |
| Tail latency on reads | Amplified by fan-out | Lower |
| Examples | MongoDB, Cassandra, Elasticsearch | DynamoDB GSIs, Spanner, CockroachDB, Riak Search (term-partitioned) |

Search engines (topic 51) are essentially secondary indexes at large scale and face exactly this choice.

---

## 5. Rebalancing

### 5.1 Why and what's required

Over time, things change: query throughput grows (add CPUs), the dataset grows (add disks), machines fail (others
take over their partitions). All require moving data and requests from one node to another, a process called
**rebalancing**. Requirements for any strategy:

1. Afterwards, load (data, reads, writes) should be shared **fairly** between nodes.
2. The database should **keep accepting reads and writes** while rebalancing.
3. **No more data than necessary** should move, to minimise network and disk I/O.

### 5.2 How not to do it: hash mod N

It is tempting to assign `partition = hash(key) mod N` where N is the number of nodes. The problem: when N changes,
**most keys move**.

```
hash(key) = 123456
  N = 10 → 123456 mod 10 = 6
  N = 11 → 123456 mod 11 = 3
  N = 12 → 123456 mod 12 = 0

Going from N to N+1 nodes moves about N/(N+1) of all keys, e.g. ~91% when going from 10 to 11,
even though only ~9% need to move to fill the new node.
```

Rebalancing becomes a near-total reshuffle. Topic 13 covers this failure in detail as the motivation for consistent
hashing. The strategies below all avoid it by decoupling **keys→partitions** (stable) from **partitions→nodes**
(changes on rebalancing).

### 5.3 Strategy 1: a fixed number of partitions

Create **many more partitions than nodes** up front, e.g. 1,000 partitions for a 10-node cluster (about 100
partitions per node). The mapping from key to partition never changes; only whole partitions are assigned to nodes.

```
Before (4 nodes, 20 partitions, 5 each):
 node 1: p0  p1  p2  p3  p4
 node 2: p5  p6  p7  p8  p9
 node 3: p10 p11 p12 p13 p14
 node 4: p15 p16 p17 p18 p19

Add node 5: it steals a few partitions from each existing node
 node 1: p0  p1  p2  p3
 node 2: p5  p6  p7  p8
 node 3: p10 p11 p12 p13
 node 4: p15 p16 p17 p18
 node 5: p4  p9  p14 p19
Only 4 of 20 partitions (20% of data) moved, exactly what the new node needs.
```

- Moving a partition means copying it in the background; the old assignment serves requests until the transfer
  completes, then routing switches.
- More powerful nodes can be given more partitions.
- Used by Riak, Elasticsearch (number of primary shards fixed at index creation), Couchbase, Voldemort, and Redis
  Cluster (16,384 hash slots).

The catch is **choosing the number**. It is fixed at creation and usually hard to change (Elasticsearch requires
reindexing or split/shrink operations). It is the **maximum number of nodes** you can ever use, so you want it high;
but each partition has fixed management overhead, so too many partitions is wasteful. If the dataset size is
uncertain, picking the right number is hard: partitions that are too large make rebalancing and recovery slow; too
small and overhead dominates.

### 5.4 Strategy 2: dynamic partitioning

Range-partitioned databases would be awkward with fixed boundaries: a bad boundary guess can put all data in one
partition. So they create partitions **dynamically**:

- When a partition grows beyond a configured size (HBase's default is 10 GB), it is **split** into two halves of
  roughly equal size, and one half can be moved to another node.
- When a partition shrinks below a threshold (lots of deletes), it can be **merged** with an adjacent one.

This is like how a B-tree splits pages (topic 17), applied at the cluster level.

```
partition [a, z) grows past 10 GB
   → split at median key "m":   [a, m)   [m, z)
   → [m, z) moved to a less loaded node
```

- The number of partitions **adapts to the data volume**: small datasets have few partitions, large ones many.
- The problem: an empty database starts with **one partition**, so all writes hit one node until the first split.
  HBase and MongoDB allow **pre-splitting**: configuring initial boundaries if you know the key distribution.
- Works with key-range partitioning (HBase, Bigtable tablets, CockroachDB/TiKV ranges, Spanner splits) and can also be
  used with hash-range partitioning (MongoDB supports both).
- Some systems also split by **load**, not only size, to deal with hot ranges.

### 5.5 Strategy 3: partitions proportional to nodes

A third approach makes the number of partitions proportional to the number of **nodes**: a fixed number of
partitions **per node**. Cassandra used this with 256 token ranges ("vnodes") per node by default (newer versions
default to fewer, e.g. 16, with a smarter token allocation algorithm).

- While the node count is constant, partitions grow with the dataset. Adding nodes makes partitions smaller again.
- When a node joins, it picks a fixed number of existing partitions **at random**, splits each, and takes over one
  half of each split. This is essentially consistent hashing with virtual nodes (topic 13).
- Random splitting can produce unfair splits; averaged over many partitions per node, the new node ends up with a
  fair share. Newer algorithms choose split points more carefully.
- Requires hash-based partitioning, so boundaries can be picked from the hash space.

### 5.6 Comparison

| | Fixed partitions | Dynamic partitioning | Proportional to nodes |
|---|---|---|---|
| Number of partitions | Fixed at creation | Grows/shrinks with data | Fixed per node, grows with cluster |
| Partition size | Grows with data | Bounded (split threshold) | Grows with data, shrinks with nodes |
| Key scheme | Usually hash | Range (or hash) | Hash |
| What happens on node add | Moves whole partitions | Moves split partitions | Splits random partitions, takes halves |
| Main weakness | Hard to pick the number | Single partition at the start (pre-split) | Random splits can be uneven |
| Examples | Riak, Elasticsearch, Redis Cluster, Couchbase | HBase, Bigtable, MongoDB, CockroachDB | Cassandra, ScyllaDB |

### 5.7 Automatic or manual rebalancing?

Fully automatic rebalancing is convenient but risky. Rebalancing is expensive: it moves large amounts of data and
reroutes requests, and can overload nodes and the network while it runs. Combined with automatic failure detection it
can cascade:

```
node 3 overloaded → slow → failure detector thinks it's dead
   → rebalancer moves its partitions to nodes 1, 2, 4 (huge data transfer)
   → transfer adds load to nodes 1, 2, 4 and to node 3 (still alive, serving)
   → more nodes slow down → more "dead" nodes → more rebalancing ...
```

So many systems take a middle ground: the system **suggests** a new assignment, and a human approves it (Couchbase,
Riak), or automatic rebalancing is rate-limited. Keeping a human in the loop is slower but avoids operational
surprises.

---

## 6. Request routing

### 6.1 The problem

Once data is partitioned, when a client wants key `"user:42"`, how does it know which node (IP and port) to contact?
And as partitions move during rebalancing, someone must keep that knowledge up to date. This is an instance of
**service discovery** and has three common shapes.

```
 (1) Any node / forward         (2) Routing tier               (3) Partition-aware client
                                                                
  client                         client                          client (knows the map)
    │ (any node)                   │                                │
    v                              v                                │ direct to owner
  node A ──forward──> node C    ┌─────────┐                         v
  (if A doesn't own the key)    │ router  │──> node C             node C
                                └─────────┘
```

1. **Contact any node** (e.g. through a round-robin load balancer). If that node owns the partition it handles the
   request; otherwise it forwards it to the right node and relays the reply. Cassandra coordinators work this way.
   Redis Cluster is a variant: a node that doesn't own the slot replies with a `MOVED` redirect telling the client
   where to go.
2. **Routing tier:** all requests go to a partition-aware proxy layer, which forwards them. The routing tier owns no
   data. MongoDB's `mongos`, Vitess's `vtgate`, and many proxies for sharded MySQL/Redis are examples.
3. **Partition-aware clients:** the client library holds the partition map and connects directly to the right node,
   saving a hop. Smart clients for Redis Cluster, Kafka producers/consumers (which fetch partition leadership
   metadata from brokers), and the Cassandra drivers' token-aware routing do this.

In all three cases the key question is the same: **how does the component making the routing decision learn about
changes in the assignment of partitions to nodes?** Getting this wrong means requests go to the wrong node, where at
best they are redirected and at worst a node that has lost a partition serves stale data or accepts writes it no
longer owns.

### 6.2 Coordination service

Many systems keep the authoritative partition map in a separate **coordination service** such as ZooKeeper or etcd
(topic 31), which is itself replicated with consensus (topic 30) so all observers see the same map.

```
             ┌───────────────────────────────────────┐
             │   ZooKeeper / etcd                    │
             │   partition → node map                │
             │   p0 → node1, p1 → node3, p2 → node2   │
             └──────▲───────────────┬────────────────┘
       register /   │               │ notify (watch) on change
       heartbeat    │               v
  ┌───────┐ ┌───────┐ ┌───────┐   ┌──────────────┐
  │node 1 │ │node 2 │ │node 3 │   │ routing tier │ ← or partition-aware clients
  └───────┘ └───────┘ └───────┘   └──────────────┘
```

- Nodes register themselves in the coordination service; it maintains the authoritative mapping.
- Routers (or clients) **subscribe** (ZooKeeper watches, etcd watch) and are notified when partitions move or a leader
  changes.
- Examples: HBase, SolrCloud, and Kafka (historically) use ZooKeeper; Kafka's newer KRaft mode replaces it with an
  internal Raft-based controller quorum. MongoDB uses its own config server replica set; Kubernetes-based systems
  often use etcd.

The alternative is **gossip** (topic 32): Cassandra and Riak nodes spread cluster state among themselves, so any node
can route a request. This avoids an external dependency at the cost of weaker consistency of the routing view
(different nodes may briefly disagree about who owns what).

Even with a coordination service, routing information is distributed and can be briefly stale. Robust systems
therefore have nodes **verify ownership**: a node that receives a request for a partition it no longer owns rejects
or redirects it, and leadership changes are guarded by epochs/fencing (topics 23, 31) so a deposed partition leader
can't accept writes.

Finding the routing tier or the nodes themselves typically still relies on DNS (topic 04), which changes rarely
enough that it is adequate for that purpose.

### 6.3 Queries that span partitions

Single-key operations route to one partition. Everything else is harder:
- **Secondary-index queries:** scatter-gather or global index (section 4).
- **Analytics queries:** massively parallel processing (MPP) engines split a query into stages that run on partitions
  in parallel and combine results (topic 20, topic 40).
- **Multi-partition writes:** updating records on two partitions atomically needs a distributed transaction (topic 34)
  or a saga (topic 35). Choosing the partition key so that data changed together lives together is often the most
  effective fix.

---

## 7. Choosing a partition key: the practical summary

The partition key decision is the most consequential and least reversible one in a partitioned system. Useful
questions:

1. **What is the most common access pattern?** Choose a key such that it touches one partition (e.g. `user_id` if most
   queries are "this user's data").
2. **Is the key's cardinality high and its load even?** Low-cardinality keys (country, status) produce few, uneven
   partitions.
3. **Are keys monotonic?** With range partitioning, timestamps and sequential IDs create write hotspots.
4. **Are there celebrity keys?** Plan for splitting or caching the heavy hitters.
5. **What must be updated atomically together?** Co-locate it to avoid cross-partition transactions.
6. **Which queries will scatter?** Make sure they are rare or served by a different system (a search index, a
   warehouse fed by CDC, topic 39).

---

## 8. How it connects

- **Topic 02 (latency & queueing):** tail latency amplification in scatter-gather queries.
- **Topic 11 and 12 (caching):** caches absorb hot-key reads; hot keys are a classic cache failure mode.
- **Topic 13 (consistent hashing):** why `hash mod N` fails; the ring and virtual nodes behind strategy 3.
- **Topics 17–19 (B-trees, indexes, LSM-trees):** each partition is a local storage engine; dynamic splitting mirrors
  B-tree page splits; local secondary indexes are ordinary indexes per partition.
- **Topics 23–24 (replication):** each partition is replicated; per-partition leaders; no cross-partition ordering.
- **Topics 30–31 (consensus & coordination):** the partition map and partition leadership are managed via consensus
  and leases.
- **Topic 32 (gossip):** the decentralised alternative for spreading routing information.
- **Topics 34–35 (distributed transactions & sagas):** what you need when a write spans partitions.
- **Topic 37 (Kafka):** topic partitions are the unit of ordering and parallelism, with the same key-choice issues.
- **Topic 51 (search):** local vs global index partitioning in search engines.

---

## Common misconceptions

- "Sharding and replication are alternatives." → They solve different problems (size/write throughput vs
  availability/read scaling) and are used together: each partition is replicated.
- "Hash partitioning eliminates hotspots." → It evens out *keys*, not *load per key*. A single hot key still lands on
  one partition.
- "Consistent hashing means `hash(key) mod N`." → `mod N` is exactly what consistent hashing avoids: changing N moves
  almost every key.
- "More partitions is always better." → Each partition has overhead, scatter-gather queries fan out wider, and
  tail latency gets worse with fan-out.
- "Secondary indexes work the same on a sharded database." → Either every query fans out to all partitions (local
  indexes) or index updates become cross-partition and usually asynchronous (global indexes).
- "Automatic rebalancing is always desirable." → Rebalancing is expensive; combined with false failure detection it
  can cascade into an outage.
- "The partition key can be changed later if needed." → Changing it means rewriting and moving all data; choose it
  based on access patterns from the start.
- "A range-partitioned database starts out spread across the cluster." → With dynamic partitioning, an empty table
  starts as one partition on one node unless pre-split.

## Further reading (supplements)

- Martin Kleppmann, *Designing Data-Intensive Applications*, Ch. 6 "Partitioning".
- Fay Chang et al., "Bigtable: A Distributed Storage System for Structured Data" (OSDI 2006): tablets and dynamic
  splitting.
- Giuseppe DeCandia et al., "Dynamo: Amazon's Highly Available Key-value Store" (SOSP 2007): hash partitioning with
  virtual nodes.
- Apache Cassandra documentation, "Dynamo" architecture page (partitioning, vnodes, token allocation).
- Redis documentation, "Redis cluster specification" (hash slots, `MOVED`/`ASK` redirects).
- AWS documentation, "Best practices for designing and using partition keys effectively" (DynamoDB), including write
  sharding for hot keys.

---

## Questions

1. Replication already gives you more machines. Why doesn't it solve the problem of a dataset or write load that
   exceeds one machine?
2. An IoT system stores readings keyed by timestamp in a range-partitioned database. Ingestion is slow even though
   the cluster has 20 nodes. What is happening and how would you change the key?
3. Why does hash partitioning make range queries expensive? How does a compound key like
   `PRIMARY KEY ((user_id), posted_at)` recover some of the benefit?
4. A video's view counter receives 100,000 increments per second and the partition holding it is overloaded. Why
   won't adding nodes help? Describe a fix and its cost.
5. Why must the hash function used for partitioning be stable across processes, and why is a language's built-in
   object hash often a bad choice?
6. A sharded product catalogue uses local secondary indexes. Searching by category got slower after scaling from 8 to
   64 partitions. Why?
7. With a global secondary index, a user creates a listing and immediately searches for it, but it doesn't appear.
   Why might this happen?
8. Why is `hash(key) mod N` a bad rebalancing scheme? Roughly what fraction of keys move when going from 10 to 11
   nodes?
9. With a fixed number of partitions, why is choosing that number hard? What goes wrong if it is too small or too
   large?
10. A brand-new HBase table receives a burst of writes, and only one node is busy. Why, and what's the fix?
11. Describe how a node joins under "partitions proportional to nodes".
12. Explain how fully automatic rebalancing combined with automatic failure detection can cause a cascading failure.
13. Compare the three request-routing approaches. What problem do all of them share, and how does a coordination
    service address it?
14. A partition was moved from node 2 to node 4, but a client with a stale routing table still sends writes to node
    2. What should happen, and what mechanisms ensure it?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Each replica holds the full dataset and applies every write. So storage per machine doesn't shrink, and the
write load per machine doesn't shrink either (single-leader funnels all writes through one node). Only by splitting
the data, so each machine stores and writes a subset, do storage and write capacity grow with the number of nodes.

**2.** All new readings have the current timestamp, so they all fall into the one partition covering "now": a write
hotspot, with the other 19 nodes idle. Prefix the key with something that spreads writes, such as
`sensor_id#timestamp`. Writes spread across sensors; the cost is that a time-range query over all sensors needs one
range scan per sensor (or a scatter).

**3.** The hash scatters adjacent keys across all partitions, so a range of keys can be anywhere and the query must
go to every partition. With a compound key, only `user_id` is hashed to pick the partition; within it, rows are
sorted by `posted_at`, so a range query for one user's posts is a single sorted scan on one partition. Range queries
across users still scatter.

**4.** All increments target one key, which hashes to one partition on one node (plus its replicas); more nodes
don't change where that key lives. Fix: split the counter into, say, 100 sub-keys (`video#00`..`video#99`), increment
a random one, and sum them to read. Cost: reads touch 100 keys, and you must track which keys are split. Alternatives:
buffer increments and write batched updates; cache reads.

**5.** Every client, router and node must compute the same partition for the same key, now and after restarts and
upgrades. Built-in hashes like Java's `hashCode()` may differ across implementations or versions, and Python's string
hash is randomised per process, so the same key could map to different partitions. Use a well-defined function
(Murmur3, MD5, xxHash).

**6.** Each partition indexes only its own data, so a category search must query all partitions and combine the
results (scatter-gather). Going from 8 to 64 partitions means 64 requests per query; latency is determined by the
slowest of 64, so tail latency amplification makes the query slow more often, and each partition does work for every
query.

**7.** Global index entries live on index partitions determined by the term, often on different nodes from the data,
and are usually updated asynchronously to avoid distributed transactions. The write succeeded on the data partition
but the index partition hasn't been updated yet, so the search misses it. It's a read-your-writes violation on the
index.

**8.** When N changes, `hash mod N` changes for almost all keys, so most data must move even though the new node
needs only a small share. From 10 to 11 nodes, about 10/11 ≈ 91% of keys change partition, while only ~9% need to
move to fill the new node.

**9.** The number is fixed at creation and is the maximum number of nodes you can spread data over, but each partition
has overhead. Too few: you can't scale beyond that many nodes and partitions get huge, making rebalancing and
recovery slow. Too many: overhead in metadata, file handles, and per-partition work, and wider fan-out for
scatter-gather queries. With an uncertain growth curve, there's no good value.

**10.** Dynamic partitioning starts a new table with a single partition, so all writes go to one node until the
partition reaches the split threshold and splits. Fix: pre-split the table with initial boundaries based on the
expected key distribution (or use keys that don't concentrate writes).

**11.** Each node has a fixed number of partitions (e.g. vnodes/tokens). A new node chooses that many existing
partitions at random (in hash space), splits each, and takes over half of each split, while the old nodes keep the
other halves. Over many partitions the new node gets roughly a fair share. This is consistent hashing with virtual
nodes.

**12.** An overloaded node responds slowly and is declared dead. The system starts moving its partitions to other
nodes, a large data transfer that loads both the receivers and the "dead" node (still alive and serving). Those nodes
slow down, more are declared dead, more rebalancing starts, until the cluster collapses. Requiring human approval or
rate-limiting rebalancing breaks the loop.

**13.** Any-node forwarding: simple for clients, extra hop. Routing tier: clients stay simple, extra hop and another
component to run. Partition-aware clients: no extra hop, but clients must hold and refresh the map. All share the
problem of keeping the partition-to-node map accurate as partitions move. A coordination service (ZooKeeper, etcd)
holds the authoritative map, replicated with consensus, and notifies routers/clients via watches when it changes.
Gossip is the decentralised alternative.

**14.** Node 2 should not accept the write: it should reject it or redirect (like Redis `MOVED`), and the client
should refresh its routing info and retry at node 4. This requires nodes to check ownership against the current
assignment on every request, and leadership/ownership changes to be guarded by epochs or fencing tokens, so a node
with an outdated view (e.g. after a pause) can't act as owner.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What is partitioning (sharding)?
**A:** Splitting data so each record belongs to exactly one partition, with partitions on different nodes.

**Q:** What does partitioning scale that replication doesn't?
**A:** Storage capacity and write throughput.

**Q:** How do partitioning and replication combine?
**A:** Each partition is replicated; each node holds leaders for some partitions and followers for others.

**Q:** What are skew and a hotspot?
**A:** Skew: uneven data/load across partitions. Hotspot: a partition with disproportionately high load.

**Q:** Advantage of key-range partitioning?
**A:** Efficient range scans; related keys stay together.

**Q:** Disadvantage of key-range partitioning?
**A:** Hotspots from sequential keys like timestamps.

**Q:** Advantage of hash partitioning?
**A:** Uniform distribution of keys, even for sequential ones.

**Q:** Disadvantage of hash partitioning?
**A:** Range queries on the key must hit all partitions.

**Q:** What is a compound primary key in Cassandra?
**A:** A hashed partition key plus clustering columns sorted within the partition.

**Q:** What is the celebrity problem?
**A:** One key gets huge load; hashing can't spread a single key.

**Q:** How do you split a hot key?
**A:** Add a random suffix to spread writes; reads fetch and combine all sub-keys.

**Q:** What is a local (document-partitioned) index?
**A:** Each partition indexes only its own records.

**Q:** Cost of local secondary indexes?
**A:** Reads by secondary key scatter-gather to all partitions; tail latency amplification.

**Q:** What is a global (term-partitioned) index?
**A:** One index over all data, itself partitioned by the indexed term.

**Q:** Cost of global secondary indexes?
**A:** Writes touch multiple partitions; usually updated asynchronously, so reads may be stale.

**Q:** Three goals of rebalancing?
**A:** Fair load afterwards, stay online during it, move no more data than necessary.

**Q:** Why not hash mod N?
**A:** Changing N moves almost all keys.

**Q:** Fixed number of partitions strategy?
**A:** Many more partitions than nodes; move whole partitions when nodes change.

**Q:** Weakness of a fixed partition count?
**A:** Hard to choose; caps the number of nodes, too many adds overhead.

**Q:** Dynamic partitioning?
**A:** Split partitions that grow too large, merge small ones.

**Q:** Weakness of dynamic partitioning?
**A:** Starts with one partition; needs pre-splitting.

**Q:** Partitions proportional to nodes?
**A:** Fixed partitions per node; new node splits random partitions and takes halves.

**Q:** Why be careful with automatic rebalancing?
**A:** It's expensive and combined with false failure detection can cascade.

**Q:** Three request routing approaches?
**A:** Any node forwards, a routing tier, or partition-aware clients.

**Q:** Role of ZooKeeper/etcd in routing?
**A:** Holds the authoritative partition map and notifies routers of changes.

**Q:** Alternative to a coordination service for routing?
**A:** Gossip among nodes (Cassandra, Riak).

**Q:** What should a node do with a request for a partition it no longer owns?
**A:** Reject or redirect it; ownership guarded by epochs/fencing.
