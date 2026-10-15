# 13 — Consistent Hashing

> Part 3 — Caching & the Edge · Prerequisites: topics 09, 11

Once data or work is spread over many machines, every request needs an answer to a simple question: which
machine owns this key? The obvious answer, `hash(key) % N`, works perfectly until N changes. Then almost
every key moves to a different machine at once, which for a cache means a near-total cold start and for a
database means migrating nearly all of the data. Consistent hashing is a family of techniques that answer
"which machine?" in a way that moves only a small, unavoidable fraction of keys when machines are added or
removed. It underpins distributed caches, Dynamo-style databases such as Cassandra and DynamoDB, and
load balancers that need affinity. This topic explains the problem precisely, the hash ring and its virtual
nodes, two important alternatives (rendezvous and jump hashing), and where each shows up.

By the end of this topic you should be able to:
1. Show quantitatively why `hash(key) % N` remaps almost all keys when N changes.
2. Explain the hash ring: how keys are assigned, what moves when nodes join or leave, and why that is minimal.
3. Explain virtual nodes: the load-imbalance problem they fix and their other benefits.
4. Describe rendezvous (highest random weight) hashing and jump consistent hashing, and when to prefer each.
5. Identify where consistent hashing appears: client-side cache sharding, Dynamo-style replication, load balancers.
6. Recognize its limits: it balances keys, not load, and it does not by itself solve hot keys or replication.

---

## 1. The problem with `hash(key) % N`

### 1.1 Modulo placement

The simplest way to spread keys across N servers is to hash the key to a large integer and take it modulo N:

```
server = hash(key) % N
```

With a decent hash function, keys spread evenly and lookup is O(1) with no shared state: every client
computes the same answer. For a fixed N, this is great.

### 1.2 What happens when N changes

Suppose N goes from 4 to 5 (you add a cache node). A key stays put only if `h % 4 == h % 5`. Let's tabulate
hash values 0–19:

```
h      : 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19
h % 4  : 0 1 2 3 0 1 2 3 0 1  2  3  0  1  2  3  0  1  2  3
h % 5  : 0 1 2 3 4 0 1 2 3 4  0  1  2  3  4  0  1  2  3  4
same?  : ✓ ✓ ✓ ✓ . . . . . .  . . . . . .  . . . .        → 4 of 20 stay (20%)
```

Only 20% of keys stay where they were; 80% move. In general, going from N to N+1 keeps only about 1/(N+1) of
keys in place, so for large N almost everything moves. Removing a node is just as bad.

But the *ideal* is very different. When going from N to N+1 nodes, the new node should end up with 1/(N+1) of
the keys, and those keys must come from somewhere, so **at minimum 1/(N+1) of keys must move**, and nothing
else needs to.

| Change | Keys moved with modulo | Minimum necessary |
|--------|------------------------|-------------------|
| 4 → 5 nodes | ~80% | 20% |
| 10 → 11 nodes | ~91% | ~9% |
| 100 → 101 nodes | ~99% | ~1% |

### 1.3 Why that's a disaster

- **Distributed cache**: adding one node to scale up makes ~90%+ of lookups miss at once. That is a cache
  avalanche plus cold start (topic 12) caused by the act of *adding capacity*. A node *crashing* has the same
  effect.
- **Sharded database or storage**: almost all data has to be copied to new owners: hours of network transfer
  and disk I/O for a change that should touch a few percent.
- **Load balancer affinity**: hashing clients to backends for session or cache locality (topic 09); any
  backend change reshuffles nearly every client.

What we want is a **monotone** mapping: when a node is added, keys only move *to* the new node, never between
existing nodes; when a node is removed, only its keys move.

---

## 2. The hash ring

### 2.1 The idea

Consistent hashing (Karger et al., 1997) maps both **nodes** and **keys** into the same hash space, and treats
that space as a circle.

1. Choose a hash function with a large output range, e.g. 0 to 2^32 − 1, and imagine the range bent into a ring.
2. Place each node on the ring at `hash(node_id)` (for example, the hash of its IP address or name).
3. To find a key's owner, compute `hash(key)` and walk **clockwise** from that point until you hit a node. That
   node owns the key.

```
                        0 / 2^32
                          │
               key k1 ●   │
                     ╲    ▼
            Node D ●──────────────● Node A
                  ╱                  ╲
                 │                    │     ● key k2
                 │      hash ring     │       (walks clockwise → Node B)
                 │                    │
                  ╲                  ╱
            Node C ●──────────────● Node B
                       ● key k3
                       (walks clockwise → Node C)

 k1 lies between D and A → owned by A
```

Equivalently, each node owns the **arc** from its predecessor (exclusive) up to its own position (inclusive).

### 2.2 Implementing lookup

Keep the node positions in a sorted array (or a balanced tree). Lookup is a binary search for the first
position ≥ hash(key), wrapping to the first element if you run off the end:

```
positions = sorted([(hash(n), n) for n in nodes])

owner(key):
    h = hash(key)
    i = first index with positions[i].hash >= h      # binary search, O(log M)
    if i == len(positions): i = 0                    # wrap around
    return positions[i].node
```

M is the number of positions on the ring (with virtual nodes, nodes × vnodes per node). All clients with the
same node list compute the same owner without talking to each other.

### 2.3 What moves when a node joins

Add node E, which hashes to a point between C and D. E takes over the arc from C to E, which previously
belonged to D. **Only the keys in that arc move, all of them from D to E.** Nobody else is affected.

```
Before:   ... C ──────────────────── D ...      arc (C, D] owned by D
After:    ... C ────────── E ─────── D ...      arc (C, E] now owned by E
                                                arc (E, D] still owned by D
```

### 2.4 What moves when a node leaves

Remove node B. Its arc (A, B] is now covered by the next node clockwise, C. **Only B's keys move, all of them
to C.** In a cache, this means only B's keys miss; everything else stays warm.

On average, adding or removing one node among N moves about 1/N of the keys: the theoretical minimum.

### 2.5 The problem with one point per node

With only one position per node, arcs are wildly uneven, because node positions are random:

```
 4 nodes, 1 position each — random positions might give arcs of:
 A: 46%   B: 9%   C: 33%   D: 12%
```

With N random points on a circle, the largest arc is typically about (ln N)/N of the ring, several times the
fair share 1/N. Two more problems:

- **Departures dump load on one neighbor.** When B leaves, all of its keys go to C. C's load might double while
  A and D see nothing.
- **Heterogeneous hardware.** A node with twice the RAM should own twice the keys; one position per node can't
  express that.

---

## 3. Virtual nodes

### 3.1 The fix

Give each physical node many positions on the ring: **virtual nodes (vnodes)**. Hash `"A#0"`, `"A#1"`, ...,
`"A#99"` to place 100 points for node A, and similarly for others. A key's owner is the physical node behind
the first vnode clockwise.

```
ring (each letter = one vnode of that physical node):

   A  C  B  D  A  B  C  D  C  A  D  B  A  C  B  D  ...
   └────────────── interleaved around the ring ──────────────┘
```

### 3.2 What vnodes buy you

1. **Even distribution.** Each node's share is the sum of many small arcs; by the law of large numbers the
   total converges toward 1/N. With V vnodes per node, the standard deviation of a node's load shrinks roughly
   like 1/√V. In practice 100–200 vnodes per node gives load within a few percent of even.
2. **Spread-out redistribution.** When node B leaves, its many small arcs are each taken by whatever node
   follows them, which is a different node each time. B's load spreads across *all* survivors instead of
   landing on one neighbor. Likewise a new node takes a little from everyone.
3. **Weighting.** Give a bigger machine more vnodes (e.g. 200 vs 100) and it owns proportionally more keys.
4. **Faster rebuilds in storage systems.** When a node dies, many nodes each stream a small piece of its data
   to replacements in parallel, rather than one neighbor doing all the work.

### 3.3 The costs

- **Memory and lookup**: the ring has N × V entries (e.g. 100 nodes × 200 = 20,000 positions; binary search
  is ~15 comparisons). Trivial for clients; it matters only at very large scale.
- **Metadata and operations**: in storage systems, more vnodes means more token ranges to track, repair and
  stream. Cassandra historically defaulted to 256 vnodes per node; newer versions recommend fewer (e.g. 16)
  with a smarter token allocation algorithm, because very many ranges made repair and some operations slow.
- **Replication placement gets subtler**: see section 5.2.

---

## 4. Alternatives

### 4.1 Rendezvous hashing (highest random weight, HRW)

Proposed by Thaler and Ravishankar (1996). For each key, compute a score for **every** node, combining the key
and the node, and pick the node with the highest score:

```
owner(key) = argmax over nodes n of  hash(key, n)
```

```
key "user:42":
   hash("user:42", A) = 0.71
   hash("user:42", B) = 0.93   ← highest → B owns it
   hash("user:42", C) = 0.18
   hash("user:42", D) = 0.55
```

Why it is consistent:
- **Node removed**: only keys whose highest score was the removed node change owner; each goes to its
  second-highest node, which is effectively random, so its load spreads evenly across survivors. Every other
  key's winner is unchanged.
- **Node added**: a key moves only if the new node's score beats its current winner, which happens for about
  1/(N+1) of keys, all moving to the new node.

Strengths:
- **Even distribution without vnodes**: each key's winner is an independent uniform choice.
- **Trivially gives ranked replicas**: the top k scores are the k replicas, with no ring-walking subtleties.
- **Simple**: no ring structure, no sorted array; just the node list.
- **Weighting** is possible with a weighted scoring formula.

Weakness: lookup is **O(N)** per key (score every node). For tens or a few hundred nodes this is negligible
compared to a network hop; for thousands, use a hierarchy (pick a group, then a node) or a ring.

Where it shows up: client-side sharding libraries, CDN and proxy selection of which cache server holds an
object (it was designed for distributed web caching), Microsoft's Cache Array Routing Protocol (CARP), and
GitHub's load balancer (GLB) design.

### 4.2 Jump consistent hash

From Lamping and Veach (Google, 2014). A tiny function that maps a 64-bit key to a bucket number in [0, N):

```
int32 jump_hash(uint64 key, int32 num_buckets):
    b = -1; j = 0
    while j < num_buckets:
        b = j
        key = key * 2862933555777941757 + 1          # LCG step: pseudo-random from the key
        j = floor((b + 1) * (2^31 / ((key >> 33) + 1)))
    return b
```

The idea: imagine growing the number of buckets from 1 to N. When going from n to n+1 buckets, each key should
jump to the new bucket with probability 1/(n+1), otherwise stay. A key-seeded pseudo-random sequence decides
these jumps, and the algorithm skips ahead directly to the next jump rather than iterating every n, giving
**O(log N)** expected time.

Properties:
- **No memory**: no ring, no table, just arithmetic.
- **Near-perfect balance** and **minimal movement**: going from N to N+1 buckets moves exactly the ~1/(N+1)
  keys that should move.
- **Big limitation: buckets are numbered 0..N−1, and you can only add or remove at the end.** You cannot
  remove bucket 3 out of 10 (a crashed server in the middle) without renumbering, which moves lots of keys.
  It also does not support weights directly.

So jump hash suits cases where the "buckets" are logical shards that grow in a controlled way and are
themselves made reliable (e.g. each shard is a replicated group), rather than individual servers that fail
arbitrarily.

### 4.3 Other variants worth knowing of

- **Maglev hashing** (Google's software load balancer): builds a fixed-size lookup table from per-backend
  preference lists. O(1) lookup, very even, near-minimal disruption on backend changes.
- **Consistent hashing with bounded loads** (Mirrokni, Thorup & Zadimoghaddam, 2016): like the ring, but a node
  that is already above (1 + ε) × average load is skipped and the key goes to the next node. Limits the
  overload on any single node; used for HAProxy's consistent-hash balancing and by Vimeo for video caching.
- **Fixed partitions / hash slots**: rather than mapping keys to nodes directly, hash keys into a fixed number
  of partitions (Redis Cluster: 16,384 slots; Elasticsearch/Riak style fixed shard counts) and keep a separate,
  explicit table of partition → node. Rebalancing moves whole partitions. This decouples *key placement*
  (never changes) from *partition placement* (changes under operator control). Topic 25 discusses this as a
  partitioning strategy.

### 4.4 Comparison

| | Modulo | Ring + vnodes | Rendezvous (HRW) | Jump hash | Fixed slots + table |
|--|--------|---------------|------------------|-----------|---------------------|
| Keys moved on change | ~all | ~1/N | ~1/N | ~1/N | Only moved slots |
| Balance | Good | Good with enough vnodes | Good | Excellent | Depends on slot assignment |
| Lookup cost | O(1) | O(log(N·V)) | O(N) | O(log N) | O(1) |
| Memory | None | N·V entries | Node list | None | Slot table |
| Remove arbitrary node | Bad | Yes | Yes | No (only last) | Yes (reassign slots) |
| Weights | No | Via vnode count | Via weighted score | No | Via slot count |
| Replica selection | Awkward | Walk the ring | Top-k scores | Awkward | In the table |

---

## 5. Where it shows up

### 5.1 Distributed caches

Memcached servers do not know about each other; clients decide where each key lives. Libraries such as
**ketama** (the classic consistent-hashing implementation for Memcached) build a ring with ~100–160 points per
server. When a cache server dies, only its share of keys misses and is refilled from the database; when you add
one, only ~1/N of keys miss. This directly limits the avalanche and cold-start effects from topic 12.

Proxies such as twemproxy (nutcracker) and mcrouter do the same on behalf of clients, so the node list lives in
one place. A subtle issue is **clients disagreeing about membership**: if half the clients think node C is
alive and half think it's dead, the same key is read and written on two different nodes, producing stale data.
Getting membership consistent is its own problem (topic 32).

Redis Cluster uses the fixed-slot variant: `slot = CRC16(key) mod 16384`, with slots assigned to nodes and moved
explicitly during resharding. Clients learn the map and get `MOVED` redirects when it changes.

### 5.2 Dynamo-style databases

Amazon's Dynamo paper (2007) made consistent hashing famous in databases; Cassandra, Riak and others follow it.

- **Partitioning**: each node (or vnode) owns a range of the token ring; a row's partition key is hashed
  (Cassandra uses Murmur3) to find its position.
- **Replication**: with replication factor 3, a key is stored on the owner and the next two **distinct physical
  nodes** clockwise; this list is the key's **preference list**. With vnodes, the walk must skip vnodes
  belonging to physical nodes already chosen (and, in practice, try to pick nodes in different racks or
  availability zones), or two replicas might land on the same machine.

```
     key k hashes here
          │
          ▼
 ──●──────┼──●────────●────────●────────●────────●──
   D         B        B        A        B        C
             ^        ^        ^                 ^
         replica 1   skip   replica 2   skip   replica 3
                   (B again)          (B again)

 Walk clockwise collecting distinct physical nodes until you have RF = 3:
 preference list [B, A, C]. Taking "the next three vnodes" would give [B, B, A].
```

- **Membership changes**: a joining node takes over token ranges and streams the corresponding data from the
  current owners; a leaving node streams its ranges to the new owners. Only ~1/N of data moves.
- **Leaderless reads/writes and quorums** on top of this placement are covered in topic 24; gossip-based
  membership that tells every node the ring layout is topic 32.

DynamoDB (the managed service) hashes partition keys onto internal partitions and splits partitions by size or
throughput; the details are internal, but the hash-partitioned model and the "hot partition key" warnings follow
directly from these ideas.

### 5.3 Load balancers

Load balancers use hashing when the same client or request should keep landing on the same backend (topic 09):

- **Cache affinity**: route requests for a URL to the same cache server so each object is cached once in the
  fleet instead of on every server (CDNs do this inside a PoP; HAProxy and nginx support consistent hashing on
  URI or header).
- **Connection affinity at L4**: a packet-level balancer must send all packets of a TCP connection to the same
  backend, even when the balancer fleet itself changes. Google's Maglev and GitHub's GLB use consistent hashing
  (Maglev hashing and rendezvous respectively) so that adding or removing a balancer or backend breaks as few
  in-flight connections as possible.
- **Sticky sessions** via hashing client IP: consistent hashing limits how many users lose their session when a
  backend is added or removed (but statelessness is still the better fix).

### 5.4 Others

Sharded message brokers and stream processors assign partitions to consumers with consistent schemes; peer-to-
peer systems (Chord, Kademlia DHTs) use ring- or distance-based placement; object stores place objects on
storage nodes with hashing variants (Ceph's CRUSH is a hierarchical, weighted cousin of rendezvous hashing;
topic 53).

---

## 6. Limits and pitfalls

- **Balances keys, not load.** Consistent hashing spreads *keys* evenly. If one key is hot (topic 12) or some
  keys are far larger than others, load is still uneven. Hot keys need local caching or replication; skewed
  partition keys need better key design (topic 25). Bounded-load consistent hashing helps for request routing.
- **Every client must agree on membership.** Two clients with different node lists compute different owners.
  Use a single source of truth (a coordination service, topic 31, or gossip, topic 32), or a proxy.
- **Flapping nodes cause churn.** If a node is repeatedly marked dead and alive, its keys bounce back and forth,
  each time missing (cache) or streaming (storage). Use failure detection with hysteresis rather than removing a
  node at the first timeout.
- **The hash function matters.** It must be well-distributed (Murmur3, xxHash, MD5 for ketama). It need not be
  cryptographic. Do not use language built-in hash functions that are randomized per process (e.g. Python's
  `hash()` for strings), or every process builds a different ring.
- **Range queries are lost.** Hashing destroys key order, so "all orders between two dates" must touch every
  node. Range partitioning keeps order at the cost of hotspot risk (topic 25).
- **Replication still needs care.** The ring tells you an order of nodes; turning that into replicas on distinct
  machines and failure domains is extra logic.

---

## 7. How it connects

- **Topic 09**: hashing-based load balancing and sticky sessions use these schemes to limit disruption.
- **Topic 11, 12**: client-side sharding of Memcached/Redis; consistent hashing turns "a cache node died" from an
  avalanche into a 1/N miss bump.
- **Topic 14**: CDNs route objects to cache servers inside a PoP with consistent or rendezvous hashing so each
  object is stored once per PoP.
- **Topic 24**: Dynamo-style leaderless replication builds quorums on top of ring placement.
- **Topic 25**: partitioning strategies (hash vs range, fixed partitions, rebalancing) in databases generalize
  this topic.
- **Topic 32**: gossip protocols distribute ring membership in Cassandra-like systems.
- **Topic 53**: object storage placement (CRUSH, erasure-coded fragments) uses related ideas.

## Common misconceptions

- "Consistent hashing means no keys move." → About 1/N of keys must move when membership changes; consistent
  hashing just ensures that's *all* that moves.
- "Consistent hashing guarantees even load." → One point per node is badly uneven; even with vnodes it balances
  keys, not hot keys or large values.
- "Virtual nodes are about replication." → They are about balance, spreading the impact of joins/leaves, and
  weighting. Replication is a separate walk to distinct physical nodes.
- "Rendezvous hashing is too slow." → O(N) hashes per lookup is microseconds for tens to hundreds of nodes,
  negligible next to a network call.
- "Jump hash is a drop-in replacement for the ring." → It can only add or remove buckets at the end, so it
  doesn't handle arbitrary node failures by itself.
- "Redis Cluster uses a hash ring." → It uses 16,384 fixed hash slots with an explicit slot-to-node map.

## Further reading (supplements)

- Karger et al., *Consistent Hashing and Random Trees: Distributed Caching Protocols for Relieving Hot Spots on
  the World Wide Web* (STOC 1997).
- DeCandia et al., *Dynamo: Amazon's Highly Available Key-value Store* (SOSP 2007), sections on partitioning and
  replication.
- Thaler & Ravishankar, *Using Name-Based Mappings to Increase Hit Rates* (IEEE/ACM ToN 1998), the rendezvous
  hashing paper.
- Lamping & Veach, *A Fast, Minimal Memory, Consistent Hash Algorithm* (2014), jump consistent hash.
- Eisenbud et al., *Maglev: A Fast and Reliable Software Network Load Balancer* (NSDI 2016).
- Mirrokni, Thorup & Zadimoghaddam, *Consistent Hashing with Bounded Loads* (SODA 2018), and the Google
  Research / Vimeo blog posts about it.
- Kleppmann, *Designing Data-Intensive Applications*, Ch. 6 (Partitioning), especially rebalancing strategies.

---

## Questions

1. With `hash(key) % N`, roughly what fraction of keys move when going from 9 to 10 cache servers? What is the
   minimum fraction that must move, and why?
2. On a hash ring, node E is added between C and D. Exactly which keys move, from where, and to where?
3. On a ring with one position per node, node B crashes. Which node absorbs its load, and why is that a problem?
   How do virtual nodes change the outcome?
4. Why does adding more virtual nodes per server improve balance? What does it cost?
5. You have servers with 64 GB and 128 GB of RAM in the same cache tier. How do you make the larger ones hold
   about twice as many keys with a ring? With rendezvous hashing?
6. Explain rendezvous hashing. When node C is removed, where do C's keys go, and why does that spread evenly?
7. Why is jump consistent hash a poor fit for a pool of individual cache servers that can crash at any time,
   but a good fit for a fixed set of replicated shards?
8. In a Dynamo-style store with vnodes and replication factor 3, why can't you simply take "the next three
   vnodes clockwise" as the replicas?
9. Half your application servers think cache node C is alive and half think it's dead. What goes wrong?
10. A celebrity's profile key is hammered and its cache node saturates. Will switching from modulo hashing to
    consistent hashing help? What would?
11. Redis Cluster uses 16,384 hash slots rather than a ring. What does an explicit slot table give operators
    that a pure ring does not?
12. Why is it a mistake to use a per-process randomized hash function (such as Python's built-in `hash()` on
    strings) to build a consistent-hashing ring?
13. A load balancer hashes request URLs to a pool of caching proxies. Why does consistent hashing matter here
    more than for a pool of stateless app servers?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** A key stays only if `h % 9 == h % 10`, which happens for about 1/10 of keys, so ~90% move. The minimum is
1/10: the new server should end up with 1/10 of the keys, and those must come from the existing servers; nothing
else needs to move.

**2.** Only keys whose hash falls in the arc (C, E] move. Previously the first node clockwise from them was D;
now it's E. So they move from D to E. Keys on every other arc are unaffected.

**3.** The next node clockwise, C, takes all of B's keys, possibly doubling C's load while others see no change,
and if C was already the biggest arc it may tip over (cascading failure). With vnodes, B's many small arcs are
each followed by different physical nodes, so B's load is spread roughly evenly across all survivors.

**4.** Each node's share becomes the sum of many independent random arcs, so its total concentrates around 1/N
(variance falls roughly as 1/V). Costs: larger ring (N·V entries, slightly slower lookups, more memory), and in
storage systems many more token ranges to track, stream and repair.

**5.** Ring: give the 128 GB servers twice as many virtual nodes (e.g. 200 vs 100). Rendezvous: use a weighted
score (e.g. `-weight / ln(hash(key, node))` normalized to (0,1)) so larger-weight nodes win proportionally more
often.

**6.** For each key compute `hash(key, node)` for every node and choose the highest. Removing C only changes keys
whose winner was C; each now goes to its second-highest scorer. Since scores are independent per key, the
second-place node is effectively random, so C's keys spread evenly across all remaining nodes. All other keys
keep their winner.

**7.** Jump hash maps to buckets 0..N−1 and only supports adding/removing the last bucket. If server 3 of 10
crashes, you can't remove bucket 3 without renumbering and moving many keys. With a fixed set of shards that are
each internally replicated (so a shard never disappears), and growth happens by appending shards, jump hash's
zero-memory, perfectly balanced, minimal-movement mapping is ideal.

**8.** Consecutive vnodes may belong to the same physical node (or the same rack/zone), so two or three "replicas"
could live on one machine and die together. The walk must skip vnodes of physical nodes already chosen, and
ideally ensure distinct failure domains.

**9.** For keys owned by C, the two groups compute different owners: one reads and writes C, the other the next
node. Writes and invalidations reach only one copy, so the other group reads stale data, and the hit ratio drops.
Membership must come from a consistent source (config service, gossip, proxy).

**10.** No. Both schemes map one key to one node; consistent hashing only changes how keys move when membership
changes. Fixes: in-process local caching of the hot key with a short TTL, replicating the key under several names
across nodes (or read replicas), shrinking the value.

**11.** Key → slot never changes; only slot → node does, and operators control it explicitly. That allows
deliberate, incremental rebalancing (move specific slots, at chosen times, with migration of exactly those keys),
weighting by slot count, and a simple O(1) lookup table that clients cache and update via redirects.

**12.** Every process seeds the hash differently, so each process places nodes and keys at different positions
and computes different owners. Clients would disagree about where keys live, causing misses and stale data. Use a
stable, deterministic hash (Murmur3, xxHash, MD5).

**13.** Caching proxies are stateful: each holds a different set of cached objects. If adding or removing a proxy
reshuffles most URLs, nearly every request lands on a proxy that doesn't have the object, so hit ratio collapses
and origin load spikes. Stateless app servers can serve any request equally well, so remapping costs nothing.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Why does `hash(key) % N` break when N changes?
**A:** Almost every key's remainder changes, so ~N/(N+1) of keys move instead of the necessary ~1/(N+1).

**Q:** Minimum fraction of keys that must move when adding one node to N?
**A:** About 1/(N+1).

**Q:** How does the hash ring assign a key?
**A:** Hash the key onto the ring and walk clockwise to the first node.

**Q:** What moves when a node joins the ring?
**A:** Only the keys in the arc between its predecessor and itself, all from its successor.

**Q:** What moves when a node leaves?
**A:** Only its keys, all to the next node clockwise (or spread across many with vnodes).

**Q:** Ring lookup implementation?
**A:** Sorted array of positions plus binary search with wrap-around: O(log M).

**Q:** Problem with one ring position per node?
**A:** Very uneven arcs, and a departed node's load lands entirely on one neighbor.

**Q:** What are virtual nodes?
**A:** Many ring positions per physical node.

**Q:** Three benefits of virtual nodes?
**A:** Even balance, spreading joins/leaves across all nodes, weighting by capacity.

**Q:** Cost of many vnodes?
**A:** Bigger ring and more token ranges to manage, stream and repair.

**Q:** Rendezvous (HRW) hashing in one line?
**A:** Score every node with hash(key, node) and pick the highest.

**Q:** Rendezvous lookup cost?
**A:** O(N) per key.

**Q:** Rendezvous bonus feature?
**A:** The top-k scores give a natural ranked replica list.

**Q:** Jump consistent hash properties?
**A:** No memory, O(log N), excellent balance, minimal movement; buckets can only be added or removed at the end.

**Q:** When to use jump hash?
**A:** For numbered shards that grow by appending and never disappear individually (each internally replicated).

**Q:** Consistent hashing with bounded loads?
**A:** Skip nodes already above (1+ε) × average load, limiting the maximum load per node.

**Q:** How does Redis Cluster place keys?
**A:** CRC16(key) mod 16384 slots, plus an explicit slot-to-node table.

**Q:** How does Dynamo-style replication pick replicas?
**A:** Walk clockwise from the key collecting RF distinct physical nodes (the preference list).

**Q:** Why do load balancers use consistent hashing?
**A:** Cache affinity and connection affinity while minimizing disruption when backends or balancers change.

**Q:** Does consistent hashing fix hot keys?
**A:** No: it balances keys, not load per key.

**Q:** What must all clients agree on?
**A:** The same membership list and the same deterministic hash function.

**Q:** What capability does hashing remove compared with range partitioning?
**A:** Efficient range queries, since key order is destroyed.

**Q:** What is ketama?
**A:** The classic Memcached client consistent-hashing scheme, with ~100–160 ring points per server.
