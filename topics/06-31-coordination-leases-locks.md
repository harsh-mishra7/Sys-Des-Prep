# 31 — Coordination Services, Leases & Locks

> Part 6 — Distributed Systems · Prerequisites: topics 26, 27, 30

Topic 30 showed how a group of nodes can agree on a sequence of values with Raft or Paxos. Hardly anyone
writes consensus code inside their application, though. Instead they run a small, separate cluster whose
only job is to be the consensus-backed source of truth for a few critical facts: *who is the leader*, *who
holds this lock*, *which nodes are alive*, *what is the current config*. etcd and ZooKeeper are those
clusters. This topic covers what they offer, the main tool they give you (the **lease**), and why the
obvious use of that tool, a **distributed lock**, is far harder to get right than it looks. The central
lesson is that a lock in a distributed system is never a guarantee on its own. It is a hint that has to be
backed by a check at the resource being protected, which is what a **fencing token** does.

By the end of this topic you should be able to:
1. Explain what a coordination service such as etcd or ZooKeeper is for, and what it should not be used for.
2. Explain why ownership in a distributed system must be time-bounded (a lease), and what a lease can and cannot promise.
3. Walk through how a process pause or clock problem lets two clients both believe they hold a lock.
4. Explain how fencing tokens fix this, and what the protected resource must do to support them.
5. Describe Redlock, the Kleppmann–antirez debate about it, and when a Redis lock is good enough.
6. Use ZooKeeper's primitives (ephemeral nodes, sequential nodes, watches) to build leader election and a fair lock without a herd effect.

---

## 1. Coordination services

### 1.1 The problem they solve

Many distributed systems need a few facts that every node must agree on, even during failures:

- **Leader election.** Exactly one instance of the scheduler, or the primary of a database shard, should be
  active at a time (topic 23 needs this for failover).
- **Configuration and metadata.** Which shard lives on which node (topic 25), feature flags, the list of
  brokers in a cluster.
- **Service discovery and membership.** Which instances of a service are currently alive (topic 32).
- **Mutual exclusion.** Only one worker should run the nightly billing job.

Each of these needs linearizable reads and writes (topic 28) and must survive node failures. That requires
consensus (topic 30). Implementing consensus correctly is hard, so the industry factored it out into a
reusable service:

```
   app nodes (many, stateless-ish)
   ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
   │ app1 │ │ app2 │ │ app3 │ │ app4 │
   └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘
      │ sessions/leases, watches  │
      ▼        ▼        ▼        ▼
   ┌─────────────────────────────────┐
   │ coordination cluster (3 or 5)   │  ← Raft (etcd) / Zab (ZooKeeper)
   │  node A   node B   node C       │    linearizable writes, small data
   └─────────────────────────────────┘
```

The coordination cluster is small (3 or 5 nodes, so it tolerates 1 or 2 failures), stores little data, and
everything it stores is replicated by consensus. The application cluster is large and does the real work,
consulting the coordination service only for the critical facts.

### 1.2 etcd and ZooKeeper

| | ZooKeeper | etcd |
|---|---|---|
| Origin | Yahoo, 2008; used by Hadoop, HBase, older Kafka | CoreOS, 2013; Kubernetes' backing store |
| Consensus | Zab (ZooKeeper Atomic Broadcast), Paxos-like | Raft |
| Data model | Tree of "znodes" (like a filesystem), each holding small data | Flat key-value space with ordered keys (prefix ranges act like directories) |
| Liveness tie | **Sessions**: a client session kept alive by heartbeats; ephemeral znodes die with it | **Leases**: a TTL'd object; keys attached to a lease die when it expires |
| Change notification | Watches (one-shot in classic API) | Watch streams (continuous, from a given revision) |
| Versioning | Per-znode version, global zxid | Global monotonically increasing **revision** for every change |
| Conditional write | `setData(path, data, expectedVersion)` | Transactions: `if (compare) then (ops) else (ops)` |

Both give you the same core guarantees:

- **Linearizable writes.** All writes go through the leader and are committed by a majority. There is a
  single total order of all writes.
- **Reads.** ZooKeeper reads are served by whichever server the client is connected to and can be slightly
  stale (it guarantees per-client FIFO order and you can call `sync()` before a read to catch up). etcd
  reads are linearizable by default (the leader confirms it is still the leader via a quorum check, the
  "ReadIndex" technique from Raft), with an opt-in "serializable" mode that reads a possibly stale local copy.
- **Monotonic version numbers.** Every write gets a number larger than all earlier writes (ZooKeeper's zxid,
  etcd's revision). This is what makes fencing tokens easy, as section 3 shows.
- **Liveness-linked state.** A key can be tied to a client's session or lease, so it disappears automatically
  when that client stops heartbeating.

### 1.3 What not to use them for

Consensus is expensive: every write needs a disk fsync on a majority of nodes and a network round trip.
And the cluster is deliberately small. So:

- **Not a general database.** etcd has a default size limit of 2 GB (recommended max ~8 GB) and a 1.5 MB
  request limit; ZooKeeper znodes are limited to 1 MB and the entire data set lives in memory. Use them for
  kilobytes to megabytes of metadata, not for user data.
- **Not for high write rates.** Thousands of writes per second is a typical ceiling. Putting a per-request
  counter or a per-user lock in etcd will overload it.
- **Not a message queue.** Watches tell you that something changed, but they are not a durable, high-volume
  delivery mechanism. Use Kafka or a queue (topics 36–37).
- **Not something to scale horizontally by adding nodes.** Adding a node to a consensus cluster makes writes
  *slower* (bigger quorum). Five is usually the max; ZooKeeper's "observers" can add read capacity only.
- **Not on the hot path of every request.** If the coordination service is briefly unavailable (for example
  during a leader election, which takes seconds), your request path should keep working with cached
  values. It should be on the *control* path.

A good mental model: the coordination service is the "small, slow, very reliable brain" that decides who
does what; the data path should not go through it.

---

## 2. Leases: time-bounded ownership

### 2.1 Why ownership must expire

Suppose node A acquires "leadership" by writing `leader = A` to a coordination store, with no expiry. Then A
crashes. From topic 26 we know nobody can tell whether A is dead or just slow. If leadership never expires,
the system waits for A forever. If somebody else can just override it, we lose the exclusivity we wanted.

The answer is a **lease**: ownership granted *for a limited time*. The holder must renew it before it
expires. If it fails to renew, whether because it crashed, was partitioned, or paused, the lease expires
and someone else may take over.

```
time ─────────────────────────────────────────────────────────────►
A:   acquire(ttl=10s)   renew      renew      [crash]
     |──────────────────|──────────|──────────X
lease:  valid until 10    until 15   until 20      expired at 20
B:                                              wait ...  acquire at 20+ ✓
```

The lease is the standard way to turn "I can't tell whether you're dead" into "I'll treat you as dead after
a bounded time, and you agree to treat yourself as dead too".

### 2.2 How leases work in etcd and ZooKeeper

- **etcd:** `LeaseGrant(ttl=10)` returns a lease ID. You `Put` keys attached to the lease. A client sends
  `LeaseKeepAlive` periodically (typically every ttl/3). If the lease expires, etcd deletes all attached keys
  in one revision and watchers see the deletion.
- **ZooKeeper:** each client has a **session** with a timeout (say 10 s). The client library sends
  heartbeats. If the ensemble doesn't hear from the client within the timeout, the session expires and
  every **ephemeral** znode it created is deleted.

Note who decides expiry: the coordination service's leader, using its own clock. The client only knows
"I last renewed successfully at time T, the TTL is 10 s".

### 2.3 What a lease cannot promise

The lease holder's belief that it still holds the lease is based on *its own local view of time*. Topic 26
and topic 27 explained why that view can be wrong:

1. **Process pauses.** A GC stop-the-world pause, a VM being live-migrated, a laptop lid closing, the OS
   swapping the process out. The process checks "lease valid for 2 more seconds", then freezes for 30
   seconds, then resumes and acts, unaware that time passed.
2. **Clock differences.** The server measures expiry on its clock; the client on its own. If the client's
   clock runs slow, or jumps, it may think it has time left when it does not. Using monotonic clocks
   (topic 27) avoids jumps but not rate drift or pauses.
3. **Network delay.** A renewal response, or the client's subsequent write to some resource, can sit in a
   network buffer for an unbounded time.

So a lease gives you: "after expiry, the *coordination service* will let someone else take over". It does
**not** give you: "the old holder will have stopped acting by then". Safety cannot rest on the old holder's
cooperation. This is the root of every distributed-lock bug.

A common mitigation is to have clients treat the lease as expired *earlier* than the server will (for
example consider it dead after ttl minus a safety margin, measured from when the renewal was *sent*). That
narrows the window but does not close it: a pause can still occur between the check and the action.

---

## 3. Distributed locks and why they are dangerous

### 3.1 Two reasons to want a lock

Martin Kleppmann's framing is the most useful:

- **Efficiency.** The lock avoids doing the same work twice (two nodes sending the same report email,
  recomputing the same expensive aggregate). If the lock occasionally fails and both run, the cost is some
  wasted work or a duplicate email. Annoying, not catastrophic.
- **Correctness.** The lock prevents concurrent processes from corrupting data (two writers interleaving on
  the same file, two leaders both accepting writes, double-spending a balance). If the lock fails, data is
  lost or corrupted.

For efficiency locks, a simple single-Redis lock is fine. For correctness locks, a lease-based lock *on its
own* is never enough, no matter which system provides it. You need fencing.

### 3.2 The failure: a paused lock holder

Here is the classic scenario. A storage service holds a file; clients take a lock (lease) before writing.

```
  client 1          lock service           client 2           storage
     │ acquire ───────►│                       │                  │
     │◄──── ok (30s) ──│                       │                  │
     │  ...GC pause starts...                  │                  │
     │  (frozen)       │  lease expires (30s)  │                  │
     │  (frozen)       │◄──────── acquire ─────│                  │
     │  (frozen)       │───── ok (30s) ───────►│                  │
     │  (frozen)       │                       │── write v2 ─────►│ ✓
     │  ...GC pause ends; client 1 still believes it holds lock   │
     │── write v1 ────────────────────────────────────────────────►│ ✓  ← corrupts!
```

Client 1 checked its lease before the pause, so from its own perspective it did nothing wrong. Client 2
legitimately acquired the expired lock. The storage accepted both writes. The lock did not protect anything.

The same outcome arises without GC: client 1's write might simply be delayed in the network for longer than
the lease, or client 1's clock might run slow.

### 3.3 The fix: fencing tokens

The idea is to stop trusting the client and make the **resource** check. Each time the lock service grants
the lock, it returns a **fencing token**: a number that strictly increases with every grant. The client
includes the token with every write. The resource remembers the highest token it has seen and **rejects any
write with a lower token**.

```
  client 1          lock service           client 2           storage (max_token)
     │ acquire ───────►│                       │                  │ (0)
     │◄─ ok token=33 ──│                       │                  │
     │  ...pause...    │◄──────── acquire ─────│                  │
     │                 │──── ok token=34 ─────►│                  │
     │                 │                       │─ write(t=34) ───►│ ✓ max=34
     │── write(t=33) ─────────────────────────────────────────────►│ ✗ 33 < 34 rejected
```

Why it works: the lock service's grants form a total order (it's backed by consensus), so token values
reflect the real order of ownership. The storage enforces that order regardless of pauses or delays.
The client's mistaken belief no longer matters.

Where to get the token:

- **ZooKeeper:** the `zxid` of the lock node's creation, or the znode's version (`cversion`/sequence number).
- **etcd:** the key's `create_revision` or `mod_revision`, which is a global, monotonically increasing number.
- **A database:** a version column that the lock acquisition increments.

What the resource must do. This is the part that is often impossible in practice:

- It must store the max token it has seen, and the check-and-write must be atomic. A database can do this
  with a conditional update: `UPDATE accounts SET balance = ?, fence = 34 WHERE id = ? AND fence <= 34`.
- An object store with conditional writes (compare-and-set on a version or ETag) can be adapted.
- A third-party API (sending an email, charging a card) usually has no fencing support. For those you fall
  back to idempotency keys (topic 38) or accept the risk.

If the resource can do a compare-and-set itself, ask whether you need the lock at all. Often optimistic
concurrency control (topic 22) on the resource is the simpler design: the lock becomes an efficiency
optimisation to reduce conflicts, and correctness comes from the conditional write.

### 3.4 Fencing in leader election

The same pattern protects against "split brain" after leader failover (topic 23). The old leader may still
think it is leader after being paused or partitioned. Raft's **term** number (topic 30) is a fencing token:
followers reject messages from a leader with a stale term. In storage systems, a new primary is often
given a higher "epoch" and the shared storage or replicas refuse writes from lower epochs. Kafka's controller
epoch and HDFS NameNode's fencing work this way.

### 3.5 Other ways locks go wrong

- **Lock released by the wrong client.** Client 1's lease expires, client 2 acquires, then client 1 finishes
  and calls `DEL lock`, deleting client 2's lock. Fix: store a unique random value per acquisition and only
  delete if the value matches (atomically, via a Lua script in Redis or a transaction in etcd).
- **Lock held across a long operation with no renewal.** The TTL must exceed the work, or the holder must
  renew, and renewal can fail silently. Code must check the renewal result and abort work.
- **Deadlock in multi-lock code.** Same as on one machine: acquire locks in a consistent global order, and
  use TTLs so a crashed holder doesn't block forever.
- **Coordination service unavailable.** If acquiring the lock needs a quorum and there is none, nobody can
  make progress. This is the CP choice (topic 29): correctness over availability.

---

## 4. Redis-based locks and the Redlock debate

### 4.1 The single-instance Redis lock

The simplest lock:

```
SET lock:report  <random-value>  NX  PX 30000
   NX     → only set if the key does not exist
   PX     → expire after 30000 ms
returns OK if acquired, nil if someone else holds it

release (Lua script, atomic):
  if redis.call("GET", KEYS[1]) == ARGV[1] then
      return redis.call("DEL", KEYS[1])
  else return 0 end
```

This is fast and simple. Its problems:

- Single point of failure. If Redis dies, no locks.
- If you add a replica for failover, replication is asynchronous (topic 23). The primary grants a lock,
  crashes before replicating it, the replica is promoted without the key, and a second client acquires the
  same lock. Two holders.
- Plus all the pause/expiry problems from section 3 (no fencing token is provided).

For efficiency locks this is usually fine.

### 4.2 Redlock

Salvatore Sanfilippo (antirez), Redis' creator, proposed **Redlock** to avoid the single point of failure
without relying on replication. Use N independent Redis masters (typically 5), no replication between them:

1. Record the current time T1.
2. Try `SET key value NX PX ttl` on all N instances in turn, with a small per-instance timeout.
3. The lock is acquired if it was set on a **majority** (≥ 3 of 5) **and** the elapsed time (now − T1) is
   less than the TTL.
4. The remaining validity time is TTL − elapsed − a clock-drift allowance.
5. If acquisition fails, release on all instances.

The intuition is the same as quorums (topic 24): two clients can't both get a majority at the same time.

### 4.3 The debate

In 2016 Kleppmann published "How to do distributed locking", arguing Redlock is unsafe for correctness
locks. antirez replied in "Is Redlock safe?". The core points:

**Kleppmann's critique:**

1. **No fencing token.** Redlock produces no monotonically increasing number. Even if acquisition were
   perfect, the paused-client scenario of section 3.2 still corrupts data. And because the N instances are
   independent, there's no obvious way to generate one (a random value is not ordered).
2. **Timing assumptions.** Redlock's safety depends on bounded network delay, bounded process pauses, and
   clocks that advance at roughly the correct rate on every Redis node. Example: client 1 gets the lock on
   A, B, C. Node C's clock jumps forward (an NTP step), so C expires the key early. Client 2 acquires on C,
   D, E. Both now hold a majority. Similar outcomes come from a Redis node crashing and restarting without
   having persisted the key.
3. **Wrong tool either way.** If you need only efficiency, a single Redis is simpler and good enough. If you
   need correctness, use a consensus-backed system (ZooKeeper/etcd) plus fencing. Redlock is too heavy for
   the first and not safe enough for the second.

**antirez's response (summarised):**

- If the resource can check fencing tokens, it can usually do compare-and-set itself, so a lock with
  fencing is somewhat redundant. Redlock's random value can be used for a similar check-and-set.
- The clock concern applies to *wall clock jumps*; Redis should use monotonic time and operators should
  avoid stepping clocks. With that, the timing assumptions are reasonable in practice.
- Redlock checks elapsed time after acquisition, which handles delays *during* acquisition; pauses *after*
  acquisition affect every lease-based lock, including ZooKeeper's.

**Where it lands for a practitioner:**

- Everyone agrees: a lease-based lock can't, by itself, prevent a paused holder from acting. The real
  protection must sit at the resource.
- The disagreement is whether a system's safety should depend on timing assumptions. The distributed-systems
  literature's standard model (asynchronous, topic 26 and 30) says it should not: consensus algorithms are
  safe under any timing and only depend on timing for liveness. Redlock depends on timing for safety.
- Practical rule: efficiency → single Redis lock. Correctness → a consensus-backed store and fencing tokens,
  or redesign around conditional writes / idempotency so you don't need the lock.

---

## 5. ZooKeeper primitives and recipes

ZooKeeper deliberately offers no "lock" API. It gives a few small primitives from which recipes (lock,
leader election, barrier, queue) are built. Libraries such as Apache Curator implement the recipes.

### 5.1 The primitives

- **Znodes.** Nodes in a tree, e.g. `/app/locks/report`. Each has a small data payload, a version, and children.
- **Persistent vs ephemeral nodes.** A persistent node lives until deleted. An **ephemeral** node is bound to
  the session that created it and is deleted automatically when that session ends (cleanly or by
  timeout). Ephemeral nodes cannot have children. They are how ZooKeeper expresses "this client is alive".
- **Sequential nodes.** If you create with the SEQUENTIAL flag, ZooKeeper appends a monotonically increasing,
  10-digit counter, unique within the parent: `/locks/report/lock-0000000041`. This gives a total order of
  creations, i.e. a queue.
- **Watches.** A client can set a watch while reading a node (`exists`, `getData`, `getChildren`). When that
  node (or its child list) changes, ZooKeeper sends the client a one-time notification. In the classic API
  you must re-read and re-register to keep watching, and changes between the notification and your re-read
  are not individually delivered, so you always re-read state rather than trusting the event.
- **Conditional updates.** `setData` and `delete` take an expected version; they fail if the node changed
  (compare-and-set).

Ordering guarantee worth knowing: a client sees a watch event for a change before it sees the new data from
that change. So "notification, then re-read" never misses the change that triggered it.

### 5.2 Recipe: leader election

```
/election
   ├── n_0000000007   (ephemeral, sequential)  ← created by node A  → smallest = leader
   ├── n_0000000008   created by node B        → watches n_...07
   └── n_0000000011   created by node C        → watches n_...08
```

1. Each candidate creates an ephemeral sequential node under `/election`.
2. It lists the children. If its node has the smallest sequence number, it is the leader.
3. Otherwise it sets a watch on the node **immediately before** its own (not on the leader).
4. When that predecessor is deleted (its owner's session expired), the watcher re-checks: if it now has the
   smallest number, it becomes leader; otherwise it watches its new predecessor.

The leader's "fencing token" can be its sequence number or the creation zxid.

### 5.3 Recipe: a fair lock without a herd effect

The naive lock: everyone tries to create `/lock` as an ephemeral node; losers watch `/lock`. When it is
deleted, **every** waiter wakes up and races to create it. With 1000 waiters, one release triggers 1000
notifications and 1000 create attempts. This is the **herd effect** (thundering herd).

The ZooKeeper lock recipe is the same as leader election above: each waiter creates an ephemeral sequential
child and watches only its predecessor. A release wakes exactly one waiter, and lock order is FIFO by
sequence number (fair). Ephemeral nodes mean a crashed holder's lock is released automatically when its
session expires.

Read-write locks extend this: readers create `read-` nodes and wait only for earlier `write-` nodes; writers
wait for any earlier node.

### 5.4 The subtle part: session expiry and ConnectionLoss

- If the client is disconnected but the session hasn't expired yet, ZooKeeper's client library tries to
  reconnect. The client is in limbo: it might still be leader, it might not. Correct code should stop acting
  as leader (or at least stop starting new critical work) as soon as it sees `Disconnected`, and must give
  up for sure on `Expired`.
- A `create` that fails with ConnectionLoss may actually have succeeded. When using sequential nodes, you
  then don't know your node's name. Curator handles this by embedding a unique GUID in the node name so the
  client can find its own node after reconnecting.
- None of this removes the paused-process problem: a client paused for longer than its session timeout
  resumes believing it is leader. Fencing is still needed for correctness.

### 5.5 The same ideas in etcd

etcd's concurrency package implements the same recipes: a lock is a key under a prefix attached to the
client's lease; waiters watch the key with the next-lower `create_revision` (equivalent to "predecessor");
the revision is the fencing token. Kubernetes controllers use a "Lease" object for leader election with the
same pattern.

---

## 6. How it connects

- **Topic 26 (faults & partial failure):** process pauses and "can't tell slow from dead" are exactly why
  leases exist and why locks fail.
- **Topic 27 (time & clocks):** leases measure time; clock jumps and drift undermine them. Monotonic clocks help
  but don't fix pauses.
- **Topic 28 (consistency models):** a lock service must be linearizable; etcd's default linearizable reads vs
  ZooKeeper's possibly stale reads.
- **Topic 29 (CAP):** coordination services are CP: without a majority they refuse requests.
- **Topic 30 (consensus & Raft):** etcd is Raft, ZooKeeper is Zab. Raft terms are fencing tokens.
- **Topic 22 (concurrency control):** fencing is a form of optimistic concurrency control at the resource;
  often a conditional write replaces the lock.
- **Topic 23 (single-leader replication):** failover needs leader election and fencing of the old leader (split
  brain).
- **Topic 32 (membership):** ephemeral nodes / leases are one way to track membership; gossip is the alternative.
- **Topic 38 (idempotency):** for resources that can't check fencing tokens, idempotency keys are the fallback.
- **Topic 48 (multi-region):** consensus clusters spread across regions pay cross-region latency on every write.

---

## Common misconceptions

- "A distributed lock guarantees only one process is in the critical section." → A lease-based lock only
  guarantees the lock *service* hands it to one client at a time. A paused or delayed old holder can still
  act. Only a check at the resource (fencing) gives mutual exclusion on the resource.
- "Using ZooKeeper/etcd instead of Redis makes locks safe." → It makes the *grant* safe (consensus, no lost
  locks on failover) and provides a token. You still have to use the token at the resource.
- "A random lock value is as good as a fencing token." → A random value only proves identity, not order. The
  resource can't tell whether 0x9f… is newer or older than 0x3a….
- "etcd is a fast, consistent key-value store, so I'll put my data in it." → It's designed for small metadata
  with modest write rates. Large data or high write volume will overwhelm it.
- "Adding more nodes to the etcd cluster scales it." → More voting nodes means larger quorums and slower writes.
- "ZooKeeper watches deliver every change." → Classic watches are one-shot notifications; intermediate
  changes can be missed. Always re-read state.
- "If my code checks that the lease is still valid right before writing, I'm safe." → The pause can happen
  between the check and the write, or the write can be delayed in the network.

---

## Further reading (supplements)

- Kleppmann, *Designing Data-Intensive Applications*, Ch. 8 ("The truth is defined by the majority",
  fencing tokens) and Ch. 9 ("Membership and coordination services").
- Martin Kleppmann, "How to do distributed locking" (blog post, 2016).
- Salvatore Sanfilippo (antirez), "Is Redlock safe?" (blog post, 2016), and the Redis docs page on distributed locks.
- Hunt et al., "ZooKeeper: Wait-free coordination for Internet-scale systems" (USENIX ATC 2010).
- Mike Burrows, "The Chubby lock service for loosely-coupled distributed systems" (OSDI 2006) — introduced
  "sequencers" (fencing tokens) and the coarse-grained lock design.
- Gray & Cheriton, "Leases: An Efficient Fault-Tolerant Mechanism for Distributed File Cache Consistency" (SOSP 1989).
- Apache Curator recipes documentation; etcd documentation on leases and the concurrency API.

---

## Questions

1. Why do systems run a separate small consensus cluster (etcd/ZooKeeper) instead of embedding consensus in
   every application?
2. A teammate proposes storing every user's shopping cart in etcd "because it's strongly consistent". What
   goes wrong?
3. Why must ownership in a distributed system be time-bounded? What would happen with a lock that never
   expires?
4. A lease has a 10-second TTL. The holder checks "I have 6 seconds left" and then writes to storage. How can
   this write still happen after another client has acquired the lease?
5. Explain the difference between a lock for efficiency and a lock for correctness. Which locking approach is
   appropriate for each?
6. Walk through how a fencing token prevents corruption in the GC-pause scenario. What must the storage
   service do?
7. Why can't a random UUID stored in the lock value serve as a fencing token?
8. Single Redis primary with an async replica is used for locks. Describe a failover sequence that produces two
   lock holders.
9. Summarise Kleppmann's two main objections to Redlock. What is antirez's strongest counterpoint?
10. Describe how to implement leader election with ZooKeeper ephemeral sequential nodes. Why watch only the
    predecessor?
11. What is the herd effect, and how does the ZooKeeper lock recipe avoid it?
12. Your ZooKeeper client gets a `Disconnected` event but not yet `Expired`. What should a leader do, and why?
13. A job sends a payment request to a third-party API that doesn't support fencing tokens. What can you do
    to make "only once" hold?
14. How is a Raft term number related to a fencing token?
15. Why are coordination services classified as CP, and what does that mean for your application during a
    network partition?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Consensus is subtle and easy to get wrong, and most applications need it only for a few facts (leader,
config, membership, locks). A shared, battle-tested cluster provides linearizable storage with leases and
watches; apps stay simple and stateless-ish, and the expensive consensus path is limited to small, rare
control-plane operations.

**2.** etcd is built for small metadata and modest write rates: every write is a Raft round trip plus fsync on
a majority, total data size is limited (default 2 GB), and adding nodes doesn't scale writes. Millions of
carts with frequent updates will exceed both size and throughput, and an etcd slowdown will then affect
everything else that relies on it (e.g. leader election). Carts belong in a database.

**3.** You can't distinguish a crashed holder from a slow one. A lock that never expires blocks everyone forever
when the holder crashes. Allowing arbitrary override defeats exclusivity. A lease bounds the wait: after
the TTL without renewal, the system treats the holder as gone.

**4.** The holder may pause (GC, VM migration) between the check and the write for longer than 6 seconds; or
the write may be delayed in the network; or its clock may run slow relative to the lock server's. Meanwhile
the lease expires on the server and another client acquires it. The old write then arrives.

**5.** Efficiency: occasional double execution only wastes work (duplicate email, recomputation). A simple
single-Redis `SET NX PX` lock is fine. Correctness: double execution corrupts data. You need a
consensus-backed lock service *and* fencing tokens checked by the resource, or a design that doesn't need
the lock (conditional writes, idempotency).

**6.** Each grant comes with a strictly increasing token (e.g. 33, then 34). Client 1 (token 33) pauses; client 2
acquires (token 34) and writes; storage records max=34. Client 1 wakes and writes with 33; storage rejects it
because 33 < 34. The storage must atomically compare the incoming token with the stored max and only apply
writes with token ≥ max, updating the max.

**7.** A fencing token must be *ordered* so the resource can tell older from newer holders. Random values only
identify a holder; the resource can't decide which of two random values is stale.

**8.** Client 1 acquires `SET lock NX PX` on the primary. Before the key replicates, the primary crashes. The
replica is promoted without the key. Client 2 runs `SET lock NX` and succeeds. Both believe they hold the lock.

**9.** (a) Redlock has no fencing token, so a paused client can still corrupt data even with perfect acquisition;
(b) its safety depends on timing assumptions (bounded pauses, delays, clock rate), e.g. a clock jump on one
node can let two clients each get a majority. antirez's strongest point: pauses after acquisition affect
every lease-based lock (ZooKeeper too), and if the resource supports compare-and-set you can use that
directly; with monotonic clocks the timing assumptions are reasonable in practice.

**10.** Each candidate creates an ephemeral sequential node under `/election`. The lowest sequence number is the
leader. Every other candidate watches the node just before its own. When it's deleted (session expiry or
resignation), that watcher re-lists children and becomes leader if it's now lowest. Watching only the
predecessor means one deletion wakes one client, instead of everyone.

**11.** When many clients watch the same node, a single change wakes all of them; they all race to re-acquire,
flooding the server, and all but one fail. The recipe chains waiters (each watches its predecessor), so a
release wakes exactly one waiter and order is FIFO.

**12.** Stop doing leader-only work (or at least stop starting new critical operations) immediately, because
the session might expire and another node may become leader while this one is disconnected. If it
reconnects before expiry, it can resume; on `Expired` it must definitely abdicate. Even then, fencing at the
resource is needed for pauses.

**13.** Use an idempotency key (topic 38): send a stable key per logical payment so the provider deduplicates
retries or duplicate attempts. Additionally record intent and outcome in your own database with a
conditional write so a second worker sees the payment already attempted. Without provider support,
"exactly once" can't be guaranteed; you reconcile afterwards.

**14.** A term number increases with each election and every leader message carries it. Followers reject messages
with an older term. That's exactly a fencing token: a monotonically increasing number from an ordered
grant, checked by the recipient, so a deposed leader's messages are rejected.

**15.** They use majority quorums: during a partition, the minority side cannot commit writes (and in etcd,
linearizable reads), so it refuses requests rather than return possibly wrong answers. Your app on the
minority side cannot acquire locks or change config; it should keep running on cached state where safe and
stop leader-only work.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What is a coordination service for?
**A:** A small consensus-backed cluster storing critical metadata: leader election, locks, config, membership, with linearizable writes, leases and watches.

**Q:** Consensus algorithm in etcd? In ZooKeeper?
**A:** etcd: Raft. ZooKeeper: Zab.

**Q:** Three things not to use etcd/ZooKeeper for.
**A:** Bulk data storage, high write rates, message queuing (and anything on every request's hot path).

**Q:** Why doesn't adding nodes scale a consensus cluster?
**A:** Writes need a majority; more voters means bigger quorums and slower commits.

**Q:** What is a lease?
**A:** Ownership granted for a bounded time that the holder must renew; if not renewed it expires and someone else can take over.

**Q:** What does a lease guarantee, and what doesn't it?
**A:** The service won't grant it to anyone else before expiry. It doesn't guarantee the old holder has stopped acting.

**Q:** Three ways a lease holder can act after expiry.
**A:** Process pause (GC/VM), delayed network message, clock drift/jump.

**Q:** Efficiency lock vs correctness lock?
**A:** Efficiency: failure only wastes work. Correctness: failure corrupts data; needs consensus + fencing.

**Q:** What is a fencing token?
**A:** A strictly increasing number issued with each lock grant; the resource rejects operations with a token lower than the highest seen.

**Q:** Where do etcd and ZooKeeper fencing tokens come from?
**A:** etcd revision (create/mod revision); ZooKeeper zxid or sequential node number.

**Q:** Why isn't a random lock value a fencing token?
**A:** It isn't ordered, so the resource can't tell stale from current.

**Q:** Correct single-Redis lock acquire/release?
**A:** `SET key rand NX PX ttl`; release via Lua script deleting only if value matches.

**Q:** Why is async replication bad for Redis locks?
**A:** Primary can grant a lock and crash before replicating; promoted replica lets another client acquire it.

**Q:** Redlock in one sentence?
**A:** Acquire the same key on a majority of N independent Redis masters within the TTL.

**Q:** Kleppmann's two objections to Redlock?
**A:** No fencing token; safety depends on timing assumptions (pauses, delays, clocks).

**Q:** Practical rule for Redis vs ZooKeeper locks?
**A:** Efficiency → single Redis. Correctness → consensus store + fencing, or conditional writes/idempotency.

**Q:** Ephemeral znode?
**A:** A node deleted automatically when its creating session ends.

**Q:** Sequential znode?
**A:** Name gets a monotonically increasing suffix, giving a total order of creation.

**Q:** ZooKeeper watch semantics (classic)?
**A:** One-shot notification of a change; re-read and re-register; intermediate changes may be missed.

**Q:** ZooKeeper leader election recipe?
**A:** Ephemeral sequential nodes; lowest is leader; others watch their predecessor.

**Q:** Herd effect?
**A:** One change wakes all waiters who stampede; avoided by watching only your predecessor.

**Q:** What should a ZooKeeper leader do on Disconnected?
**A:** Stop leader-only work until reconnected; abdicate on Expired.

**Q:** Raft's built-in fencing token?
**A:** The term number.

**Q:** Coordination services: CP or AP?
**A:** CP: minority side refuses requests during a partition.
