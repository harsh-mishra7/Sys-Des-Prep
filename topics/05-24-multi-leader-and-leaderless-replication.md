# 24 — Multi-leader & leaderless replication

> Part 5 — Scaling Data · Prerequisites: topics 13, 23

Single-leader replication (topic 23) has one great property: a single node orders every write, so there are never
conflicting versions of the data. It also has one great limitation: every write must reach that one node. If the
leader is on another continent, writes are slow. If the leader is unreachable, writes stop. If the "database" is a
phone in airplane mode, there is no leader to reach at all. This topic covers the two families of designs that drop
the single-writer rule: **multi-leader** replication, where several nodes each accept writes and exchange them, and
**leaderless** replication, where any replica accepts writes and clients talk to several replicas at once (the
Dynamo style, used by Cassandra, Riak, ScyllaDB and the original Amazon Dynamo). Both buy availability and latency
by giving up the single ordering, and both therefore must face the **conflict problem**.

By the end of this topic you should be able to:
1. Explain when multi-leader replication is worth its complexity (multi-region, offline clients, collaborative
   editing) and why it makes write conflicts unavoidable.
2. Describe conflict avoidance and the main ways conflicts are resolved (with details deferred to topic 33).
3. Describe multi-leader topologies and the ordering problems they cause.
4. Explain how leaderless reads and writes work, and why `W + R > N` makes reads see the latest write.
5. List the edge cases in which quorums still return stale data.
6. Explain read repair, anti-entropy (with Merkle trees), hinted handoff and sloppy quorums, and the trade-offs of
   each.

---

## 1. Multi-leader replication

### 1.1 The mechanism

In a **multi-leader** (also: master-master, active-active) setup, more than one node accepts writes. Each leader
processes a write locally and then forwards the change asynchronously to all other leaders, which apply it. Each
leader can also have its own followers.

```
          Region EU                                  Region US
  ┌────────────────────────┐                ┌────────────────────────┐
  │  clients ──> LEADER A  │ <── async ───> │  LEADER B <── clients  │
  │               │        │   replication  │     │                  │
  │          followers     │                │  followers             │
  └────────────────────────┘                └────────────────────────┘
  Each region's writes are acknowledged locally and shipped to the other region later.
```

Inside a single datacentre this rarely makes sense: the extra complexity outweighs the benefits. It is used in a few
specific situations.

### 1.2 Use case: multiple datacentres / regions

With single-leader replication across regions, every write from Europe must cross the Atlantic to a US leader.
With one leader per region:

| | Single leader (in US) | Multi-leader (one per region) |
|---|---|---|
| Write latency for EU users | Transatlantic round-trip (~80–150 ms) | Local (~1–5 ms) |
| Region outage | If it's the leader's region, failover needed; writes stop meanwhile | Other regions keep accepting writes |
| Inter-region network problem | EU writes fail | Writes continue; replication catches up later |
| Write conflicts | Impossible | Possible: the same record edited in both regions |

Examples: MySQL with tools like Tungsten, BDR for PostgreSQL, CouchDB, and multi-region "active-active" modes of
several managed databases (e.g. DynamoDB global tables, which resolve conflicts with last-writer-wins).

### 1.3 Use case: clients with offline operation

A calendar app on phone, laptop and tablet must allow edits while offline. Each device has a **local database that
acts as a leader**, accepting writes, and syncs with other devices when it is online. This is multi-leader
replication taken to an extreme: every device is a "datacentre", and the replication lag between them can be
**days**. CouchDB/PouchDB and many mobile sync frameworks are designed around this model.

### 1.4 Use case: collaborative editing

Real-time collaborative editors (several people typing in one document) are a similar problem: each user's local
copy accepts edits immediately and propagates them asynchronously. If you instead required a lock on the document
(effectively single-leader), only one person could type at a time. Fine-grained units of change (a keystroke) and
specialised algorithms, operational transformation or CRDTs, make this work (topic 33).

### 1.5 The conflict problem

Because each leader accepts writes independently and replicates asynchronously, two leaders can accept conflicting
changes to the same data before either knows about the other's:

```
time ───────────────────────────────────────────────────────────>
Leader A (EU):  title = "B"  ──────── replicate ──────>
Leader B (US):  title = "C"  ──────── replicate ──────>
                       both acknowledged to their clients

later:   A receives "C", B receives "B".
         If each simply applies what arrives last:  A ends with "C", B ends with "B".
         The replicas have DIVERGED, permanently.
```

In single-leader replication the second writer would have blocked, or its transaction would have been aborted, and
the user could retry. In multi-leader both writes succeeded, the users have moved on, and the conflict is detected
only later and asynchronously. Making conflict detection synchronous (wait for all leaders to agree before
acknowledging) would throw away the whole point of multi-leader: independent, local writes.

Conflicts are not only "same field, two values". They can be subtle constraint violations: two people in different
regions book the same meeting room for the same slot, each leader sees a free room, both bookings succeed.

### 1.6 Handling conflicts

**Avoid them.** The simplest and most commonly recommended strategy: make sure all writes for a given record go
through the same leader. For example, each user has a "home" region and all their writes go there. From that record's
point of view it is single-leader. Avoidance breaks down when the home leader must change (region failure, user
moves), at which point concurrent writes on the old and new leader become possible again.

**Converge.** Every replica must end up in the same final state when all changes have been applied. Ways to do this:

- **Last write wins (LWW):** give each write a timestamp or unique ID, keep the highest, discard the rest. It
  converges, but it **silently loses data** (the "losing" write was acknowledged to its client), and timestamps from
  unsynchronised clocks may pick the wrong winner (topic 27).
- **Replica-priority:** writes from the higher-numbered replica win. Also loses data.
- **Merge values:** e.g. concatenate them ("B/C"), or for sets take the union.
- **Keep all versions (siblings)** and let application code or the user resolve the conflict later, on read or on
  write. The Dynamo shopping cart is the classic example.
- **Use data structures that merge automatically:** CRDTs (conflict-free replicated data types), such as counters and
  sets designed so that concurrent updates always combine deterministically.

Resolution can run **on write** (a conflict handler called as soon as the database detects a conflict in the
replication stream, e.g. Bucardo's Perl handlers) or **on read** (all versions are stored and returned to the next
reader, who resolves and writes back, e.g. CouchDB). These approaches are the subject of topic 33; here the point is
simply that multi-leader **requires** one of them, and choosing none means divergence.

### 1.7 Replication topologies

With more than two leaders, the **topology** describes which leader sends changes to which.

```
 Circular                 Star (tree)                  All-to-all
  A ──> B                     A                         A ─── B
  ^     │                   / | \                       │ \ / │
  │     v                  B  C  D                      │  X  │
  D <── C              (A forwards between leaves)      │ / \ │
                                                        D ─── C
```

- **Circular and star:** each write passes through several nodes. To avoid infinite loops, each node tags the writes
  it has seen with its identifier and ignores writes it has already processed. The weakness: **one failed node
  interrupts the flow** for others until the topology is reconfigured, often manually.
- **All-to-all:** every leader sends to every other. More fault tolerant (no single forwarding node), but messages
  can **overtake** each other because some network paths are faster than others:

```
Client 1 on leader A: INSERT row x       Client 2 on leader B: UPDATE row x (after seeing the insert)

Leader C receives:   UPDATE x  (via B, fast link)   ... then  INSERT x  (via A, slow link)
                     ^ the row doesn't exist yet!
```

  This is a causality problem (like the consistent-prefix anomaly in topic 23). Timestamps are not enough to order
  these correctly, because clocks are not synchronised (topic 27). The general technique is **version vectors**,
  which track what each node has seen and detect causal order (topics 27, 33). Many multi-leader implementations
  handle this poorly; check before relying on it.

### 1.8 Other costs of multi-leader

Features that assume a single writer become dangerous: autoincrement keys (two leaders hand out the same ID;
workarounds include interleaved ranges or globally unique IDs, topic 49), triggers, and integrity constraints
(uniqueness can't be checked locally when another leader might be inserting the same value). Multi-leader is often
retrofitted onto databases designed for single leaders, so these are common sources of surprises.

---

## 2. Leaderless replication (Dynamo-style)

### 2.1 The idea

Leaderless replication abandons the concept of a leader entirely. **Any replica can accept a write**, and the
**client (or a coordinator node acting on the client's behalf) sends each write to several replicas in parallel**
and each read to several replicas too. Amazon's Dynamo paper (2007) popularised it; Riak, Cassandra, ScyllaDB and
Voldemort followed. (Note: Amazon DynamoDB, the managed service, is a different design that uses leaders per
partition, despite the name.)

There is no failover, because there is no leader to fail. If one replica is down, the write still succeeds on the
others.

### 2.2 Writing and reading with N, W, R

Each key is stored on **N** replicas (often chosen with consistent hashing, topic 13: the N nodes following the
key's position on the ring). Two parameters control each request:

- **W**: a write is considered successful once **W** replicas have acknowledged it.
- **R**: a read queries replicas and waits for **R** responses.

```
N = 3, W = 2, R = 2

WRITE key=cart:42, value=v2 (version 7)
  client/coordinator ──> replica 1  ack ✓
                     ──> replica 2  ack ✓     2 acks = W → success
                     ──> replica 3  (down) ✗

READ key=cart:42
  client/coordinator ──> replica 1  returns v2 (version 7)
                     ──> replica 2  (slow, ignored once R reached)
                     ──> replica 3  (recovered) returns v1 (version 6)
  Got R=2 responses: one says v2@7, one says v1@6 → return the newer, v2.
```

Two things to notice:

1. Reads go to several replicas precisely because some replicas may have missed writes (replica 3 was down). The
   client compares the responses and picks the newest using **version numbers**, not by trusting any one node.
2. Requests are typically *sent* to all N replicas; W and R are about how many responses to *wait* for.

### 2.3 Why `W + R > N` works

If `W + R > N`, the set of W replicas that acknowledged the latest write and the set of R replicas you read from
**must overlap in at least one replica**. That replica holds the latest write, so (given version numbers to identify
it) the read can return it.

```
N = 5 replicas:  [1] [2] [3] [4] [5]

W = 3: write acknowledged by  [1] [2] [3]
R = 3: read answered by               [3] [4] [5]
                                       ^ overlap guaranteed: 3 + 3 = 6 > 5

If W = 2, R = 2 (sum 4, not > 5):
   write on [1] [2], read from [4] [5]  → no overlap → read can miss the write entirely
```

Reads and writes that obey this rule are called **quorum reads and writes**. A common choice is N odd (3 or 5) with
`W = R = (N + 1) / 2`, i.e. majorities.

Tuning W and R:

| Setting (N=3) | Reads | Writes | Tolerates |
|---|---|---|---|
| W=2, R=2 | Medium cost | Medium cost | 1 node down for reads and writes |
| W=3, R=1 | Fast, cheap | Slow; any node down blocks writes | 0 down for writes, 2 for reads |
| W=1, R=3 | Slow; any node down blocks reads | Fast | 2 down for writes, 0 for reads |
| W=1, R=1 | Fast | Fast | Stale reads likely; `W+R ≤ N`, no overlap guarantee |

In general, writes tolerate `N − W` unavailable nodes and reads tolerate `N − R`. Choosing `W + R ≤ N` deliberately
trades the overlap guarantee for lower latency and higher availability: reads are more likely to be stale, but they
succeed even with many nodes down. Cassandra exposes this per query through consistency levels (`ONE`, `QUORUM`,
`ALL`, `LOCAL_QUORUM`...).

### 2.4 The edge cases: when quorums still return stale data

It is tempting to read `W + R > N` as "guarantees the latest value". It does not, in practice. Cases where a quorum
read can be stale or behave strangely:

1. **Sloppy quorums** (section 3.4): the W writes may land on nodes outside the key's normal N replicas, so the
   overlap argument no longer holds.
2. **Concurrent writes.** If two writes happen concurrently, it is unclear which "happened first"; the outcome depends
   on conflict resolution. With last-write-wins by timestamp, one write is silently dropped, and clock skew can drop
   the "newer" one (topics 27, 33).
3. **A write concurrent with a read.** The write may have reached some replicas but not others; whether the read sees
   it is undetermined.
4. **Partially failed writes are not rolled back.** If a write succeeded on fewer than W replicas, the client is told
   it failed, but the replicas that did accept it **keep it**. Subsequent reads may or may not return that "failed"
   value.
5. **A replica carrying the new value fails** and is restored from a replica with an old value. The number of
   replicas storing the new value can drop below W, breaking the overlap.
6. **Non-monotonic reads across clients (no linearizability).** Even with perfect timing, a quorum system can show a
   new value to one reader and then the old value to a later reader:

```
N=3, W=3 in progress, R=2.  Write x=1 being sent to replicas A, B, C. Old value x=0.

  t1: A has x=1, B and C still x=0
  reader 1 queries A, B → sees {1, 0} → returns 1 (newer)
  t2: reader 2 queries B, C → sees {0, 0} → returns 0
Reader 2 started after reader 1 finished, yet saw an older value.
```

   This violates linearizability (topic 28). It can be fixed by having readers perform **synchronous read repair**
   before returning (write back the newest value to the stale replicas) and writers read the latest state before
   writing (as in the ABD algorithm), but Dynamo-style databases typically do not, for performance. Linearizable
   compare-and-set operations are not possible with quorums alone; they need consensus (topic 30).

So quorums are best understood as **making stale reads less likely and tunable**, not as a strong consistency
guarantee. Leaderless databases are designed for workloads that tolerate eventual consistency.

### 2.5 Monitoring staleness

In single-leader systems you can measure lag directly: "the follower is 4 MB / 2 seconds behind the leader's log
position". Leaderless systems have no shared log order, so staleness is harder to measure. Approaches include
probabilistic models of how stale reads tend to be ("probabilistically bounded staleness", PBS) and measuring how
many read-repair or hinted-handoff operations are happening. If you only use read repair (no anti-entropy), there is
no upper bound on how old a rarely-read value can be.

---

## 3. Keeping replicas in sync: repair mechanisms

Without a leader shipping a log in order, replicas miss writes whenever they are down, slow, or partitioned. Several
complementary mechanisms bring them back in line.

### 3.1 Read repair

When a client (or coordinator) reads from several replicas and notices that one returned a stale value, it **writes
the newer value back** to that replica.

```
read cart:42 from replicas 1, 2, 3:
   replica 1 → v2 (version 7)
   replica 2 → v2 (version 7)
   replica 3 → v1 (version 6)   ← stale
coordinator returns v2 to the client, and sends "cart:42 = v2 @7" to replica 3
```

- Good: no extra background work; repair effort is proportional to reads; frequently read keys stay consistent.
- Limitation: **only repairs data that is read**. A key nobody reads stays stale on that replica indefinitely, which
  can bite durability: if the replicas holding the up-to-date copies fail, the stale value is all that remains.
- Can be done in the background after the response is returned, or synchronously before returning (stronger
  guarantees, more latency).

### 3.2 Anti-entropy

A **background process** continually compares the data on replicas and copies any missing data from one to another.
Unlike the replication log in single-leader systems, it does not copy writes in any particular order, and there can
be significant delay before data is copied.

Comparing a large dataset row by row would be far too expensive over the network, so replicas use **Merkle trees**
(hash trees, topic 53):

```
Each replica builds a tree over a key range:

                    root = H(H12 + H34)
                   /                  \
           H12 = H(H1+H2)        H34 = H(H3+H4)
            /       \             /       \
          H1        H2          H3        H4       (leaf = hash of the rows in a sub-range)
       keys 0-249 250-499    500-749   750-999

Replicas exchange root hashes. Equal → whole range in sync, done.
Different → compare children, descend only into differing subtrees,
            until the few differing leaf ranges are found; transfer only those rows.
```

The amount of data exchanged is proportional to the size of the *difference*, not the size of the dataset, plus a
logarithmic number of hash comparisons. Cassandra's `nodetool repair` and Riak's active anti-entropy work this way.
Anti-entropy is what eventually fixes rarely-read data that read repair never touches. Its cost is background I/O
and CPU (building trees, streaming), so it is often run on a schedule.

Gossip protocols (topic 32) use a closely related idea to spread cluster membership information.

### 3.3 Hinted handoff

When a replica that should receive a write is temporarily unreachable, another node **accepts the write on its
behalf** and stores it with a **hint**: "this belongs to node C". When C comes back, the node holding the hint
**hands off** the write to C and deletes its copy.

```
Key K belongs on nodes A, B, C. C is down.

write K ──> A ✓
        ──> B ✓
        ──> C ✗   → node D stores (K, value, hint: "for C")
...C recovers...
D ──> C: "here are the writes you missed" → C applies them, D deletes the hints
```

This speeds up recovery after short outages (C does not need full anti-entropy to catch up on those writes). Hints
are usually kept for a limited time (Cassandra's default is a few hours); for longer outages you fall back to
anti-entropy repair. Hints are a best-effort optimisation, not a durability guarantee: if the hint holder also fails,
the hint is gone.

### 3.4 Sloppy quorums

Now combine hinted handoff with the quorum rule. Normally a write needs W acknowledgements from the key's designated
N replicas (its "home" nodes). What if a network partition cuts the client off from most of them, but other nodes in
the cluster are reachable?

- **Strict quorum:** return an error. The write is unavailable.
- **Sloppy quorum:** accept writes on **any reachable nodes**, even ones outside the key's home set, and count them
  toward W. Those nodes store the write with hints and hand it off when the home nodes are reachable again.

```
Key K's home nodes: A, B, C.   N=3, W=2.
Partition: client can reach A, D, E but not B, C.

Strict quorum: only A is a home node → 1 < W → write FAILS
Sloppy quorum: A ✓, D ✓ (hint for B) → counts as 2 = W → write SUCCEEDS
Later read with R=2 from home nodes B, C → neither has the write → STALE read,
even though W + R = 4 > 3.
```

The trade-off:

| | Strict quorum | Sloppy quorum |
|---|---|---|
| Write availability during partitions | Lower | Higher: any W reachable nodes suffice |
| `W + R > N` overlap guarantee | Holds | Does **not** hold until hinted handoff completes |
| What it really guarantees | Overlap with recent writes (modulo edge cases) | Durability: the write is stored on W nodes *somewhere* |

The original Dynamo used sloppy quorums for "always writable" shopping carts. In Riak they are on by default; in
Cassandra and Voldemort they are off by default (Cassandra's hinted handoff does not count toward the consistency
level, except `ANY`, which accepts a write stored only as a hint).

### 3.5 How the mechanisms fit together

| Mechanism | Triggered by | Fixes | Gap it leaves |
|---|---|---|---|
| Read repair | A read that sees divergent values | Frequently read keys | Rarely read keys stay stale |
| Hinted handoff | A write to an unreachable replica | Writes missed during short outages | Long outages; lost hint holders |
| Anti-entropy (Merkle trees) | Background schedule | Everything, eventually | Costs I/O; delay before repair |
| Sloppy quorum | Partition from home replicas | Write availability | Breaks quorum overlap temporarily |

---

## 4. Concurrent writes in leaderless systems

Leaderless systems have the same conflict problem as multi-leader ones, for the same reason: there is no single
node ordering writes. Two clients may write the same key at the same time, and different replicas may receive the
writes in different orders (due to network delays and partial failures). If each replica just overwrites with the
latest arrival, replicas become permanently inconsistent.

To converge, a leaderless database needs to know whether two writes are **concurrent** or whether one **happened
before** the other (it overwrote a value it had seen). Two writes are concurrent if neither knew about the other,
not if they happened at the same clock time. Systems track this with per-key **version numbers** and, across
replicas, **version vectors**: the server returns a version with every read, the client sends it back with the next
write, and the server can then tell whether the new write supersedes a value or is concurrent with it. Concurrent
values are kept as **siblings** for the application to merge, or are resolved automatically with LWW or CRDTs.

This is the heart of topic 27 (happens-before) and topic 33 (conflict resolution, siblings, CRDTs). The important
link here: leaderless and multi-leader replication **move ordering and conflict resolution from the database's
single leader to the application or to data-type design**.

---

## 5. Comparing the three replication models

| | Single-leader (23) | Multi-leader | Leaderless |
|---|---|---|---|
| Who accepts writes | One node | One per region/device | Any replica (client writes to W of N) |
| Write conflicts | None | Yes | Yes |
| Write latency (geo) | Distance to leader | Local | Depends on W and replica placement |
| Behaviour on node failure | Failover required | Other leaders continue | Nothing special if quorum reachable |
| Consistency | Linearizable if reading the leader | Eventual | Tunable, eventual; not linearizable |
| Repair | Replication log | Async replication log between leaders | Read repair, hinted handoff, anti-entropy |
| Examples | Postgres, MySQL, MongoDB | Multi-region active-active, CouchDB, offline apps | Cassandra, Riak, ScyllaDB, Dynamo |

A useful rule of thumb: use single-leader unless you have a concrete reason (multi-region local writes, offline
operation, extreme write availability) that justifies taking on conflict resolution.

---

## 6. How it connects

- **Topic 13 (consistent hashing):** how leaderless systems choose the N home replicas for a key, and what moves
  when nodes join.
- **Topic 23 (single-leader replication):** the baseline; replication lag anomalies apply here too, often worse.
- **Topic 25 (partitioning):** Dynamo-style stores are partitioned *and* replicated; the ring determines both.
- **Topic 27 (time, clocks & ordering):** why timestamps can't order concurrent writes; version vectors.
- **Topic 28 (consistency models):** why quorum reads are not linearizable; eventual and causal consistency.
- **Topic 29 (CAP & PACELC):** leaderless with sloppy quorums is the canonical "AP" design; W and R tune the
  latency-vs-consistency trade-off in PACELC.
- **Topic 30 (consensus):** majority quorums appear again, but consensus adds ordering and terms so it *can*
  provide linearizable operations.
- **Topic 32 (membership & gossip):** how nodes in a leaderless cluster learn who is alive.
- **Topic 33 (conflict resolution & CRDTs):** LWW, siblings, and CRDTs in depth.
- **Topic 48 (multi-region):** active-active vs active-passive regional architectures.
- **Topic 53 (Merkle trees):** the data structure behind anti-entropy.

---

## Common misconceptions

- "`W + R > N` gives strong consistency." → It ensures overlap between write and read sets, which makes stale reads
  unlikely, but concurrent writes, failed writes not rolled back, sloppy quorums, restored replicas and in-flight
  writes all break it. Quorum systems are not linearizable without extra work.
- "Multi-leader just means more write throughput." → Its main purposes are locality and availability; the price is
  conflicts. Every leader still applies every write.
- "Conflicts only happen when two users click at the same instant." → With offline clients or a cross-region
  partition, "concurrent" can span hours: two edits are concurrent if neither knew about the other.
- "Last write wins resolves conflicts." → It picks a survivor; the other acknowledged write is silently thrown
  away, and with skewed clocks the wrong one may win.
- "A failed write in a leaderless store left no trace." → Replicas that accepted it keep it; later reads may return
  it.
- "Read repair keeps all replicas consistent." → Only for data that is read. Anti-entropy is needed for the rest.
- "Sloppy quorums are just quorums that are more available." → They give up the overlap guarantee until hinted
  handoff completes; they guarantee durability, not freshness.
- "DynamoDB is a Dynamo-style leaderless database." → The managed service uses a leader per partition; the leaderless
  design comes from the 2007 Dynamo paper.

## Further reading (supplements)

- Martin Kleppmann, *Designing Data-Intensive Applications*, Ch. 5, sections "Multi-Leader Replication" and
  "Leaderless Replication".
- Giuseppe DeCandia et al., "Dynamo: Amazon's Highly Available Key-value Store" (SOSP 2007).
- Avinash Lakshman and Prashant Malik, "Cassandra: A Decentralized Structured Storage System" (2009).
- Peter Bailis et al., "Probabilistically Bounded Staleness for Practical Partial Quorums" (VLDB 2012).
- Apache Cassandra documentation: "Hinted Handoff", "Read Repair" and "Repair" (anti-entropy) pages.

---

## Questions

1. Why does a multi-leader setup inside a single datacentre rarely make sense, while one leader per region often
   does?
2. A note-taking app lets users edit on phone and laptop while offline. Explain why this is multi-leader replication,
   and what the "replication lag" is.
3. Two users in different regions rename the same document at the same moment. Walk through what happens in a
   multi-leader system that has no conflict resolution.
4. "Conflict avoidance" routes each user's writes to their home region. When does this stop protecting you?
5. In an all-to-all multi-leader topology, how can an update arrive before the insert it modifies? Why don't
   timestamps fix it?
6. With N=5, which (W, R) pairs guarantee overlap? Which choice would you make for a write-heavy workload that must
   stay writable with two nodes down?
7. Explain, with an example, how `W + R > N` can still return a stale value after a write that "failed".
8. Two readers query a quorum system one after the other; the first sees the new value and the second sees the old.
   How is this possible, and what property is violated?
9. A key was written once a year ago and is almost never read. One of its three replicas missed that write. Which
   repair mechanism will fix it, and which won't?
10. How do Merkle trees let two replicas with a billion keys each find their differences without sending a billion
    keys?
11. During a partition, a client can reach only one of a key's three home replicas but several other nodes. Contrast
    what happens with strict and sloppy quorums, and what each guarantees.
12. Why is hinted handoff not a durability guarantee?
13. Why do leaderless systems need version numbers (or version vectors) rather than just "the latest arrival wins"?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Inside one datacentre the network is fast and a failover is quick, so single-leader already gives low latency
and good availability; multi-leader adds conflicts for little gain. Across regions, a single leader makes every
remote write pay an inter-region round-trip and stops writes if that region or the link fails; one leader per region
gives local write latency and lets each region keep writing independently, which can be worth the conflict
complexity.

**2.** Each device has a local database that accepts writes without contacting anyone (it's a leader), and devices
sync asynchronously when online, which is multi-leader replication. The replication lag is the time until devices
sync, which can be hours or days, so conflicting edits are routine.

**3.** Each region's leader accepts its rename and acknowledges it locally. Each then replicates its change to the
other. If each leader simply applies incoming writes, leader A ends with B's title and leader B ends with A's title
(or each keeps the one it received last): the replicas permanently diverge, and different users see different
titles forever.

**4.** When a record's home leader changes: the home region fails and traffic moves to another, the user relocates,
or routing is reconfigured. During the transition, writes may land on both the old and new leader concurrently.
It also doesn't help for data shared by users with different home regions (a shared document, a meeting room).

**5.** Different network paths have different delays. Client 1 inserts on A; client 2 sees it on B and updates it;
B's message to C travels faster than A's, so C receives the update for a row it doesn't have. Timestamps don't help
because clocks on different nodes aren't synchronised well enough to order events reliably, and a timestamp can't
express "this write depends on that one". Version vectors track causal dependencies.

**6.** Need W + R > 5: (1,5), (2,4), (3,3), (4,2), (5,1) and anything larger. Writes tolerate N − W failures, so to stay
writable with two nodes down, W ≤ 3. A write-heavy workload prefers a small W; W=3, R=3 keeps the overlap and
tolerates two failures for both reads and writes. W=2, R=4 makes writes cheaper but reads fail if two nodes are down.

**7.** N=3, W=2. A write reaches replica A only; the others time out, so the client is told it failed. But A keeps
the value (no rollback). A later read with R=2 that includes A sees the "failed" value as the newest and returns it;
another read of B and C returns the old value. The result depends on which replicas answer.

**8.** A write is in flight and has reached only some replicas. Reader 1's quorum includes an updated replica, so it
returns the new value; reader 2 (starting later) happens to query two replicas not yet updated and returns the old
value. Linearizability is violated: once any read has returned the new value, all later reads should too. Fixes
include synchronous read repair before returning, which Dynamo-style stores usually skip for performance.

**9.** Read repair won't, because nobody reads it. Hinted handoff only helps if the replica was down when the write
happened and the hint still exists, and hints usually expire after hours. Anti-entropy (background Merkle-tree
comparison) will eventually find and copy the missing value.

**10.** Each replica builds a hash tree over its key ranges: leaves hash small ranges, parents hash their children.
They compare root hashes; if equal, the whole range is identical. If not, they compare children and descend only into
subtrees whose hashes differ. They exchange a logarithmic number of hashes per differing range and transfer only the
rows in differing leaf ranges, proportional to the difference rather than the dataset.

**11.** Strict quorum: only one home node is reachable, which is fewer than W=2, so the write fails; but the
`W + R > N` overlap holds for writes that did succeed. Sloppy quorum: the write is accepted on the reachable home node
plus a non-home node holding a hint, counted toward W, so it succeeds. It guarantees the write is durably stored on W
nodes, but reads from the home replicas may not see it until hinted handoff delivers it; the overlap guarantee is
temporarily lost.

**12.** The hint lives on a substitute node. If that node fails or the hint expires before the home replica returns,
the hinted write never reaches its home replica. Durability then relies on other replicas that accepted the write and
on anti-entropy.

**13.** Replicas receive writes in different orders, so "latest arrival" differs per replica and they diverge.
Version numbers let a replica tell whether a new write supersedes a value it already knew (happened-after) or was made
without knowledge of it (concurrent). Superseding writes can overwrite; concurrent ones must be kept as siblings or
merged, so nothing is lost silently and all replicas converge.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What is multi-leader replication?
**A:** Several nodes accept writes and replicate them asynchronously to each other.

**Q:** Three use cases for multi-leader?
**A:** Multi-region local writes, offline-capable clients, real-time collaborative editing.

**Q:** Why are conflicts unavoidable in multi-leader?
**A:** Leaders accept writes independently and replicate asynchronously, so concurrent changes to the same data both succeed.

**Q:** Simplest conflict strategy?
**A:** Avoid them: route all writes for a record to the same leader.

**Q:** What does last-write-wins cost?
**A:** It silently discards acknowledged writes; clock skew can pick the wrong winner.

**Q:** Weakness of circular/star topologies?
**A:** One failed node interrupts replication flow for others.

**Q:** Weakness of all-to-all topology?
**A:** Messages can overtake each other, violating causality; needs version vectors.

**Q:** What is leaderless replication?
**A:** Any replica accepts writes; clients send writes and reads to several replicas in parallel.

**Q:** Meaning of N, W, R?
**A:** N replicas per key; W acks to succeed a write; R responses for a read.

**Q:** Why does W + R > N help?
**A:** The write set and read set must overlap in at least one replica holding the latest write.

**Q:** Failures tolerated for writes and reads?
**A:** Writes: N − W. Reads: N − R.

**Q:** Name four ways quorums still return stale data.
**A:** Sloppy quorums, concurrent writes (LWW), failed writes not rolled back, restored replicas, in-flight writes.

**Q:** Are quorum reads linearizable?
**A:** No, not without synchronous read repair and more; linearizable CAS needs consensus.

**Q:** What is read repair?
**A:** On a read, write the newest value back to replicas that returned stale data.

**Q:** Limitation of read repair?
**A:** Only repairs keys that are read.

**Q:** What is anti-entropy?
**A:** Background process comparing replicas and copying missing data.

**Q:** Why Merkle trees for anti-entropy?
**A:** Compare hashes top-down to find differing ranges, transferring data proportional to the difference.

**Q:** What is hinted handoff?
**A:** A substitute node stores writes for an unreachable replica and delivers them when it returns.

**Q:** What is a sloppy quorum?
**A:** Counting writes on any reachable nodes (not just home replicas) toward W during partitions.

**Q:** What does a sloppy quorum guarantee and not guarantee?
**A:** Guarantees durability on W nodes; does not guarantee reads see the write until handoff.

**Q:** When are two writes concurrent?
**A:** When neither knew about the other, regardless of clock time.

**Q:** How do leaderless stores detect concurrency?
**A:** Version numbers per key and version vectors across replicas.

**Q:** Is AWS DynamoDB leaderless?
**A:** No; the 2007 Dynamo paper was. DynamoDB uses a leader per partition.
