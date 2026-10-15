# 28 — Consistency Models

> Part 6 — Distributed Systems · Prerequisites: topics 21, 23, 24, 26, 27

When data is replicated (topics 23–24), a read might go to a replica that hasn't yet received the latest
write. So what *should* a read return? "The latest value" sounds obvious, but in a system where
messages take time and clocks disagree (topics 26–27), "latest" is not even well defined without a
precise rule. A **consistency model** is that rule: a contract between the data store and its users
about which values reads may return, given the writes that happened. Strong models make a replicated
system behave like a single copy — easy to program against, but expensive. Weak models allow
surprising results but are fast and stay available during failures. This topic lays out the spectrum
from linearizability down to eventual consistency, the session guarantees that make weak models usable,
and what strong consistency actually costs.

By the end of this topic you should be able to:
1. Define linearizability precisely and decide whether a given history is linearizable.
2. Explain the difference between linearizability and serializability, and what strict serializability is.
3. Describe sequential, causal and eventual consistency, and order the models by strength.
4. Explain the session guarantees (read-your-writes, monotonic reads, monotonic writes, writes-follow-reads) and how they're implemented.
5. Explain why strong consistency costs coordination, latency and availability, and when it's worth paying.

---

## 1. What a consistency model is

### 1.1 The single-copy illusion

On one machine with one copy of the data, there's an obvious answer to "what does a read return?": the
value of the most recent write. Replication breaks this. Suppose `x` is replicated on three nodes and a
client writes `x = 2` (previously 1):

```
 client W ── write x=2 ──▶ replica A  (has 2)
                           replica B  (still 1, update in flight)
                           replica C  (still 1)

 client R reads x from B → gets 1.   Is that wrong?
```

It depends on the contract. Under some models this is fine; under others it's a violation. A consistency
model specifies the set of **allowed histories** — sequences of operations with their invocation times,
response times and returned values.

### 1.2 Data-centric vs client-centric

- **Data-centric models** (linearizable, sequential, causal, eventual) describe what all clients
  collectively can observe about the data.
- **Client-centric / session guarantees** (read-your-writes, monotonic reads, ...) describe what a
  *single client's* session observes, regardless of what others see. They're cheaper, and often what
  users actually care about.

### 1.3 A note on the word "consistency"

"Consistency" means different things in different places:

| Context | Meaning |
|---|---|
| ACID's "C" (topic 21) | Application invariants hold (balances non-negative); the app's job |
| Replica consistency (this topic) | What reads can return across replicas |
| CAP's "C" (topic 29) | Specifically linearizability |
| "Eventually consistent" | Replicas converge if writes stop |

Keep them apart; much confusion in discussions comes from mixing them.

### 1.4 The spectrum

```
 stronger (fewer allowed histories, more coordination)
   ▲   strict serializability  (= serializable + linearizable)
   │   linearizability
   │   sequential consistency
   │   causal consistency   ◀── strongest model that stays available under partition
   │   session guarantees (RYW, monotonic reads, ...)
   │   eventual consistency
   ▼
 weaker (more anomalies allowed, cheaper, more available)
```

Stronger models allow a *subset* of the histories weaker ones allow: every linearizable history is
sequentially consistent, every sequentially consistent history is causally consistent, and so on.

---

## 2. Linearizability

### 2.1 The definition

**Linearizability** (Herlihy & Wing, 1990; also called *atomic consistency*, *strong consistency*, or
*external consistency*) makes a replicated object behave **as if there were only one copy of the data,
and every operation took effect atomically at a single instant** somewhere between the moment it was
invoked and the moment it returned.

Two requirements:

1. **There's a single total order of all operations** consistent with the object's sequential
   behavior (e.g. a register: each read returns the value of the most recent write before it in the order).
2. **That order respects real time:** if operation A *completes* before operation B *starts*, then A
   comes before B in the order.

Operations that overlap in time (concurrent) can be ordered either way — but once any read has
returned the new value, **all later reads (starting after it) must also return the new value**. There
is a point of no return.

### 2.2 Worked example: is this linearizable?

Time flows left to right; each bar is from invocation to response.

```
 Client A:  |── write(x, 1) ─────────────|
 Client B:      |─ read(x) → 1 ─|
 Client C:                         |─ read(x) → 0 ─|
```

B's read overlaps the write and returns 1 — fine, the write could have "taken effect" before B's read.
But C's read *starts after B's read completed*, and returns 0, the old value. Since B already saw 1,
the write's linearization point must be before B's read, which is before C's read — so C must see 1.
**Not linearizable.**

```
 Client A:  |── write(x, 1) ─────────────|
 Client B:      |─ read(x) → 0 ─|
 Client C:                         |─ read(x) → 1 ─|
```

B saw 0 (write hadn't taken effect yet), C later saw 1. A valid order: B's read, write, C's read.
**Linearizable.**

The second history is also what you'd see with a single register on one machine; the first is the
classic symptom of reading from a lagging replica after another client already saw the new value. In
a real system it looks like: Alice refreshes a page and sees the final score; she tells Bob, Bob
refreshes and sees the game still in progress — his request hit a stale replica.

### 2.3 What linearizability is good for

You need it when the system must make a decision that everyone agrees on, right now:

- **Leader election and locks.** Every node must agree who holds the lock. A lock service that lets two
  nodes both "acquire" the lock because they read from different replicas is useless. ZooKeeper and etcd
  provide linearizable writes (and linearizable reads, optionally) for this reason (topic 31).
- **Uniqueness constraints.** "Username alice is taken" must be seen by everyone the moment it's
  claimed — the claim is effectively a compare-and-set, which requires linearizability.
- **Cross-channel timing dependencies.** A web server stores an uploaded image in a file store, then
  enqueues a message "resize image X". If the resizer reads from a replica of the file store that
  hasn't received the image yet, it fails or resizes an old version. The message travelled via a
  *different channel* (the queue) faster than replication. Linearizable storage prevents this.

### 2.4 How systems implement it

| Replication approach | Linearizable? |
|---|---|
| Single-leader, reads from leader (and leader knows it's still leader) | Potentially yes |
| Single-leader, reads from async followers | No |
| Consensus (Raft, Paxos, Zab) | Yes, if reads also go through consensus or use read-index/leases (topic 30) |
| Multi-leader | No — concurrent writes on different leaders |
| Leaderless (Dynamo-style) with quorums | Generally no — even with W + R > N |

Why are quorums not enough? With `N=3, W=2, R=2`:

```
 writer: write x=1, sent to replicas 1,2,3; arrives at replica 3 first.
 reader A: reads replicas 2,3 → sees {0, 1} → returns 1   (latest timestamp)
 reader B (starts after A finishes): reads replicas 1,2 → sees {0, 0} → returns 0
```

B starts after A finished and sees the older value: a linearizability violation, despite `W + R > N`.
It can be fixed by having readers perform **synchronous read repair** (write back the newest value to
a quorum before returning) and writers read a quorum before writing (the ABD algorithm), at a
performance cost — and that still only gives linearizable reads/writes, not compare-and-set, which
needs consensus. LWW conflict resolution with skewed clocks (topic 27) breaks it further.

Also beware: "reads from the leader" is only linearizable if the leader is *really* still the leader.
A deposed leader that's partitioned (or paused) and doesn't know it can serve stale reads. Raft handles
this with a read-index round (confirm leadership with a majority) or leader leases (topic 30).

### 2.5 Linearizability vs serializability

These are constantly confused:

| | Serializability | Linearizability |
|---|---|---|
| About | **Transactions** (groups of operations over multiple objects) | **Single operations** on a single object (register) |
| Guarantee | Result equals *some* serial order of transactions | Each operation appears to take effect atomically at an instant, in real-time order |
| Real-time order required? | **No** — the serial order can differ from real time | **Yes** |
| Topic | Isolation (topic 21) | Recency / replica consistency |

Serializability alone allows strange things in real time: a serializable database may order a
transaction that started later *before* one that committed earlier — e.g. a read-only transaction could
legally see a snapshot from before a write that had already committed. (Snapshot-based implementations
can do this if a client's read lands on a stale replica.)

**Strict serializability** (also "strong one-copy serializability") is both: transactions are
serializable *and* the serial order respects real time. Spanner's external consistency (topic 27) and
FoundationDB give this. Actual serial execution and strict 2PL on a single node are strictly
serializable; snapshot-based schemes, especially when reads may be served by lagging replicas, often
are only serializable.

---

## 3. Sequential consistency

**Sequential consistency** (Lamport, 1979) relaxes linearizability by dropping the real-time requirement:

- All operations appear in **some single total order**, which all clients agree on,
- and that order respects **each client's own program order**,
- but it need not match real time across clients.

```
 Client A:  write(x,1) ─────────────── (done)
 Client B:                                   read(x) → 0
```

Under linearizability, B (starting after A finished) must read 1. Under sequential consistency, the
order "B's read, then A's write" is allowed, since all clients see this same order and each client's
own order is preserved. B's view lags — but consistently: no client sees the operations in a different
order from anyone else.

Where it shows up:

- It's the classic model for **multiprocessor memory** (what programmers intuitively expect of shared
  memory; real CPUs offer even weaker models without fences).
- A system where all writes go through a single ordered log, and replicas apply the log in order but
  clients may read from a replica that's behind, roughly gives sequential consistency for the data
  (plus a per-client constraint for reads to be in program order — e.g. sticky to one replica).
  ZooKeeper's reads (without `sync`) are close to this: writes are linearizable, reads may be stale
  but each client sees updates in order.

Sequential consistency isn't "composable": two objects each sequentially consistent don't make the
combined system sequentially consistent. Linearizability is composable (local), which is one reason
it's the preferred strong model.

---

## 4. Causal consistency

### 4.1 Definition

**Causal consistency** requires only that operations which are **causally related** (happens-before,
topic 27) are seen by every client in that order. **Concurrent** operations may be seen in different
orders by different clients.

Causally related means: same client's operations in order; a read that returned a value written by a
write comes after that write; transitivity.

### 4.2 Example

```
 Alice: posts "I lost my phone!"                      (write w1)
 Bob:   reads w1, replies "Oh no, use my old one"     (write w2, depends on w1)
 Carol: posts "Lunch at noon?"                         (write w3, unrelated)

 Allowed for some reader:   w1, w2, w3   or  w3, w1, w2   or  w1, w3, w2
 Forbidden for every reader: w2 before w1  (seeing the reply without the question)
```

Causal consistency forbids the anomaly users find nonsensical (an answer before its question; a
comment on a photo you can't see yet; "unfriend Bob" applied after "post something Bob shouldn't see"),
while letting unrelated updates propagate in any order, without global coordination.

### 4.3 Why it matters: strongest available model

Causal consistency is the **strongest consistency model that can remain available during a network
partition** (shown formally by Mahajan, Alvisi & Dahlin, 2011, for a large class of models): a replica
cut off from the rest can still accept reads and writes, and enforce causality using only the metadata
it has. Anything stronger (sequential, linearizable) requires coordination that a partition prevents
(topic 29).

### 4.4 How it's implemented

The system must track dependencies and delay making a write visible until everything it depends on is
visible:

- Each write carries its **causal dependencies** — e.g. a version vector (topic 27), or a list of the
  writes the client had read.
- A replica that receives w2 depending on w1 **buffers w2** until w1 has arrived and been applied.
- Clients carry a causal "token" (what they've seen) so a new replica can wait until it's caught up.

Examples: MongoDB's *causally consistent sessions* (using cluster time), research systems like COPS and
Eiger, and many CRDT-based systems (topic 33). Total order broadcast or a single-leader log also gives
causal order as a side effect, because a total order consistent with causality is in particular causal.

Costs: dependency metadata on every write, and visibility delays when a dependency is slow to arrive.

---

## 5. Eventual consistency

**Eventual consistency**: if no new writes are made to an object, eventually all replicas will
converge to the same value, and all reads will return it.

That's all. It's a **liveness** guarantee (topic 26) with no safety component: it says nothing about
what reads return in the meantime, or how long "eventually" is. Under eventual consistency, a client
can:

- Write a value and immediately read the old one.
- See a value, then on the next read see an older value (going back in time).
- See a reply before the message it replies to.

Variations:

- **Strong eventual consistency (SEC)**: any two replicas that have received the same *set* of updates
  are in the same state — no need to wait for a quiescent period. CRDTs (topic 33) provide this.
- Convergence also requires a deterministic conflict resolution rule — LWW (which loses data, topic 27),
  siblings, or CRDT merge.

Most leaderless stores (Cassandra, Riak, DynamoDB's default reads) and asynchronous replicas (topic 23)
are eventually consistent by default; DNS (topic 04) and CDN caches (topic 12) are eventually
consistent systems too. In practice, "eventually" is usually milliseconds to seconds — but under faults
it can be minutes or more, and the application must tolerate that.

---

## 6. Session guarantees

Eventual consistency is often fine *globally* but confusing *for the user who just acted*. Session
guarantees (Terry et al., "Session Guarantees for Weakly Consistent Replicated Data", 1994 — the Bayou
project) define what one client's session observes. They're cheap because they need only per-client
tracking, not global coordination. Topic 23 introduced the first two as replication-lag anomalies.

### 6.1 Read-your-writes (RYW)

**After a client writes a value, its subsequent reads return that value or a newer one.**

Anomaly without it: a user edits their profile, the page reloads from a lagging follower, and the
edit appears lost. They edit again; now there are duplicate changes.

Implementations:

- Read anything the user may have modified from the leader (e.g. their own profile) and the rest from
  followers.
- Track the time/LSN of the user's last write; for a window after it (say 1 minute), read from the
  leader — or from a follower that has applied at least that LSN.
- The client remembers the **timestamp/version of its last write** and sends it with reads; the
  replica waits (or redirects) until it's caught up to that version.
- Cross-device RYW (write on phone, read on laptop) needs the version stored server-side, since the
  laptop doesn't know the phone's token; and multi-datacenter setups must route both devices to the same
  region.

### 6.2 Monotonic reads

**If a client has read a value, later reads won't return an older value.** No "moving backwards in time".

Anomaly: a user loads comments and sees a new comment (replica B, up to date), refreshes and it vanishes
(replica C, lagging), refreshes again and it's back.

Implementations: route each user to the same replica (e.g. hash user ID to a replica — but if that
replica fails, rerouting must respect versions), or track the highest version read and require replicas
to be at least that fresh.

### 6.3 Monotonic writes

**A client's writes are applied in the order the client issued them**, on every replica.

Anomaly: a user sets their password to P1, then P2; a replica applies P2 then P1 and ends with P1.

Implementation: per-client sequence numbers; replicas apply a client's write only after its predecessors.

### 6.4 Writes-follow-reads (session causality)

**If a client reads a value written by w1, and then makes write w2, then w2 is ordered after w1 on
every replica.** The client's write is placed after whatever it based its decision on.

Anomaly: Bob reads Alice's post and replies; on another replica, the reply arrives first and readers
see a reply to a post that doesn't exist (yet). Related to the **consistent prefix reads** guarantee
from topic 23: readers see writes in an order that makes sense (a prefix of some causal order), not
answers before questions.

Implementation: the write carries the versions the client has read as dependencies (exactly the causal
consistency mechanism).

### 6.5 Putting them together

| Guarantee | Protects against | Implemented by |
|---|---|---|
| Read-your-writes | Own write disappears | Track last-write version; read from fresh enough replica |
| Monotonic reads | Going back in time | Sticky replica, or track last-read version |
| Monotonic writes | Own writes reordered | Per-client sequence numbers |
| Writes-follow-reads | Reply before question | Write carries read dependencies |

**All four together ≈ causal consistency within a session.** Causal consistency extends this across
clients: if Bob's write depends on Alice's, Carol sees them in order too.

---

## 7. The cost of strong consistency

### 7.1 Coordination

Strong models require replicas to **agree** before answering. A linearizable write must reach (at
least) a majority, or a leader that has confirmed it's still the leader, before it's acknowledged; a
linearizable read must make sure no newer value exists anywhere. That agreement means extra round trips
between nodes on the critical path of every operation. Weak models let each replica answer from local
state and sync in the background.

### 7.2 Latency

Coordination costs round trips, and round trips cost latency — especially across regions:

- Same datacenter: ~0.5 ms per round trip; consensus adds a millisecond or two.
- Cross-region: 50–150 ms per round trip (topic 02). A linearizable write in a multi-region cluster
  pays at least one cross-region round trip — every time, even with no failures.

Attiya and Welch proved a lower bound: in a network with delay variance, the response time of
linearizable reads and writes is at least proportional to the delay uncertainty. **Weaker models can be
faster, and linearizable ones can't escape this.** This is the "else, latency vs consistency" half of
PACELC (topic 29).

There's also a cost even without distribution: modern multi-core CPUs aren't linearizable by default
(each core has its own cache and store buffers); memory fences that restore ordering are expensive. Even
hardware designers drop linearizability for performance.

### 7.3 Availability

If replicas must coordinate to answer, then **when they can't communicate, they can't answer** — at
least not on the side of a partition that lacks a majority. The minority side must refuse requests (or
return errors) to avoid serving stale or conflicting data. A weakly consistent system can keep serving
on both sides and reconcile later. This is CAP (topic 29).

### 7.4 Throughput and scalability

All writes in a linearizable register are ordered through one leader / one consensus group — a
throughput ceiling. Systems scale by partitioning (topic 25) into many independent consensus groups,
but then operations *across* partitions need more coordination (distributed transactions, topic 34).

### 7.5 When is it worth it?

| Need | Usually choose |
|---|---|
| Locks, leader election, unique constraints, account balance can't go negative | Linearizable / strictly serializable (on the small piece of state that needs it) |
| Social feeds, comments, messaging | Causal + session guarantees |
| Counters, view counts, likes, shopping cart | Eventual (with CRDTs/merge) |
| User's own profile edits | Read-your-writes on top of eventual |
| Caches, DNS, CDN | Eventual with bounded staleness (TTL) |

A common design: keep a small core of strongly consistent metadata (in a consensus system) and serve
the bulk of the data with weaker, faster models.

Many databases let you choose **per operation**: Cassandra's consistency levels (`ONE`, `QUORUM`,
`ALL`, `LOCAL_QUORUM`), DynamoDB's `ConsistentRead`, MongoDB's read/write concerns, etcd's serializable
vs linearizable reads. Note that tunable quorum levels still don't automatically yield linearizability
(section 2.4).

---

## 8. How it connects

- **Topic 21**: serializability is an isolation property; contrasted with linearizability here.
- **Topic 22**: MVCC snapshots — another place where "which version can I see?" is defined.
- **Topic 23**: replication-lag anomalies are session-guarantee violations.
- **Topic 24**: quorums don't guarantee linearizability; leaderless stores are eventually consistent.
- **Topic 27**: causal consistency is built on happens-before; version vectors track dependencies;
  Spanner's TrueTime gives strict serializability.
- **Topic 29** (next): CAP is the availability cost of linearizability; PACELC adds the latency cost.
- **Topic 30**: consensus is how linearizability is implemented in practice.
- **Topic 31**: coordination services provide linearizable operations for locks and leader election.
- **Topic 33**: CRDTs provide strong eventual consistency.
- **Topic 12–14**: caches are a common source of staleness — effectively an eventually consistent replica.

---

## Common misconceptions

- **"Linearizable and serializable are the same thing."** → Linearizability is a recency guarantee on
  single operations in real time; serializability is an isolation guarantee on multi-object
  transactions with no real-time requirement. Strict serializability is both.
- **"W + R > N gives strong consistency."** → It gives overlap, but concurrent writes, sloppy quorums,
  failed writes, and LWW conflict resolution all allow non-linearizable histories.
- **"Reading from the leader is always linearizable."** → Not if the leader has been deposed and
  doesn't know it (partition, pause). It must confirm leadership or hold a lease.
- **"Eventual consistency means the data is wrong."** → It means reads may be stale for a while; with
  session guarantees and good conflict handling it's correct for many applications.
- **"Eventual consistency means 'consistent within a few ms'."** → No time bound is promised; under
  faults convergence can take much longer.
- **"Sequential consistency is the same as linearizability."** → Sequential drops the real-time
  constraint; a client can read a stale value after another client's write completed.
- **"Causal consistency requires a global clock."** → It requires tracking dependencies (logical
  clocks), not physical time.
- **"Strong consistency is just a little slower."** → It costs round trips on every operation (huge
  cross-region) and availability on the minority side of partitions.

---

## Further reading (supplements)

- DDIA, Chapter 9 "Consistency and Consensus" (linearizability, ordering guarantees); Chapter 5 "Problems with Replication Lag".
- Herlihy & Wing, "Linearizability: A Correctness Condition for Concurrent Objects" (1990).
- Terry et al., "Session Guarantees for Weakly Consistent Replicated Data" (1994).
- Peter Bailis, "Linearizability versus Serializability" (blog post, 2014).
- Kyle Kingsbury, "Consistency Models" map at jepsen.io/consistency.
- Werner Vogels, "Eventually Consistent" (ACM Queue / CACM, 2008).
- Doug Terry, "Replicated Data Consistency Explained Through Baseball" (2011).

---

## Questions

1. State the two requirements of linearizability. What does "point of no return" mean for reads?
2. Client A writes x=1 (from t=0 to t=10). Client B reads x at t=2–4 and gets 1. Client C reads at
   t=6–8 and gets 0. Is the history linearizable? Why?
3. Why is linearizability needed for a distributed lock but not for a "likes" counter?
4. Explain the cross-channel problem with the image-upload + resize-queue example. Which model prevents it?
5. With N=3, W=2, R=2, construct a history that isn't linearizable. How could it be fixed?
6. What's the difference between linearizability and serializability? Give an example of a serializable
   but not strictly serializable execution.
7. How does sequential consistency differ from linearizability? Give a history allowed by one but not
   the other.
8. Why is causal consistency described as "the strongest model available under partitions"?
9. How does a replica enforce causal consistency when it receives a write whose dependency hasn't
   arrived yet?
10. What does eventual consistency guarantee, and what does it not guarantee? Is it safety or liveness?
11. A user changes their profile picture, refreshes, and sees the old picture. Which guarantee is
    violated, and how would you implement it across a user's phone and laptop?
12. A user sees a new comment, refreshes, and it disappears. Which guarantee? How do you fix it?
13. Why does a linearizable write in a three-region cluster cost at least one cross-region round trip
    even when nothing has failed?
14. Your system stores user sessions, a global username registry, and a news feed. Which consistency
    model would you pick for each and why?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** (1) All operations appear in one total order consistent with the object's semantics (a read
returns the latest write before it in the order); (2) the order respects real time — if A finishes
before B starts, A precedes B. "Point of no return": once any read returns the new value, every read
that starts afterwards must also return it (or a newer one).

**2.** No. B's read returned 1, so the write's linearization point is before B's read completes (t ≤ 4).
C's read starts at t=6, after B's completed, so it must come after the write in the order and return 1.
It returned 0, which is a stale read after the new value was already observed.

**3.** A lock requires all nodes to agree right now on who holds it; if two nodes read different
replicas and both see "free", both acquire it, breaking mutual exclusion. A likes counter tolerates
briefly stale or out-of-order views; nothing breaks if one user sees 1,001 and another 1,002, as long as
the count converges (e.g. a CRDT counter).

**4.** The web server writes the image to the file store and enqueues "resize X". The resizer picks up
the message quickly — faster than the file store replicates — and reads from a replica that doesn't have
the image (or has the old one). Two channels (store replication, queue) race. Linearizable storage
prevents it: once the upload write completed, any later read sees it. Alternatives: include the version
in the message and have the reader wait for it.

**5.** A write x=1 reaches replica 3 first. Reader A reads {2,3}, sees 1 (newer) and returns 1. Reader B
starts after A finishes, reads {1,2}, both still 0, returns 0 — non-linearizable. Fix: readers perform
synchronous read repair (write the newest value they saw to a quorum before returning), and writers read
the latest version from a quorum first (ABD). Even then, compare-and-set needs consensus.

**6.** Linearizability: single-object operations appear atomic and in real-time order. Serializability:
multi-object transactions produce the same result as some serial order, not necessarily the real-time
order. Example: T1 commits a write to x; afterwards T2 (read-only) starts and reads from a stale
snapshot, seeing the old x. Serial order "T2 then T1" is valid, so it's serializable — but it violates
real time, so not strictly serializable.

**7.** Sequential consistency requires a single total order respecting each client's program order but
not real time across clients. History: A writes x=1 and completes; afterwards B reads x and gets 0.
Sequentially consistent (order: B's read, A's write) but not linearizable (B started after A's write
finished).

**8.** A replica can enforce causal order using only metadata attached to writes it has received: it
can always accept local reads and writes during a partition, and buffer remote writes until their
dependencies arrive. No cross-partition coordination is needed. Stronger models (sequential,
linearizable) require agreement on a total order, which is impossible without communication, so a
partitioned replica would have to refuse requests.

**9.** The write carries its dependencies (e.g. a version vector or list of writes it depends on). The
replica checks whether all dependencies have been applied locally; if not, it buffers the write and
makes it visible only after the dependencies arrive and are applied.

**10.** It guarantees that if writes stop, all replicas eventually converge to the same value. It does
not say how long that takes, nor anything about what reads return in the meantime (stale values,
going back in time, effects before causes are all allowed). It's purely a liveness property.

**11.** Read-your-writes. Implementation: when the write completes, record its version (LSN/timestamp)
server-side keyed by user ID. On reads, the server looks up the user's last-write version and serves
from a replica that has applied at least that version (or from the leader). Storing it server-side lets
the laptop benefit from a write done on the phone; in multi-region setups route the user to the same
region for both devices.

**12.** Monotonic reads. Fix: pin the user to one replica (e.g. hash user ID), or track the highest
version the user has read and only serve them from replicas at least that fresh.

**13.** Linearizability requires that once the write is acknowledged, every subsequent read anywhere
sees it. To guarantee that, the write must be durable on (and agreed by) a majority of replicas before
acknowledgement — and with replicas in three regions, a majority includes at least one other region.
That's a cross-region round trip on every write, even without failures.

**14.** Username registry: linearizable (uniqueness requires compare-and-set agreed by everyone), stored
in a consensus-backed store. Sessions: read-your-writes/monotonic reads within the user's session is
enough (sticky routing or version tokens); global strong consistency is unnecessary. News feed: causal
consistency (replies after posts) with eventual propagation; staleness of seconds is fine, and
availability/latency matter more.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What is a consistency model?
**A:** A contract stating which values reads may return given the writes that occurred.

**Q:** Linearizability in one sentence?
**A:** The system behaves as if there's one copy and each operation takes effect atomically at an instant between its start and end, in real-time order.

**Q:** "Point of no return" in linearizability?
**A:** Once any read returns a new value, all later-starting reads must return it too.

**Q:** Three uses that need linearizability?
**A:** Locks/leader election, uniqueness constraints, cross-channel timing dependencies.

**Q:** Does W + R > N guarantee linearizability?
**A:** No — concurrent reads during an in-flight write can see new then old.

**Q:** Is reading from the leader always linearizable?
**A:** Only if it confirms it's still leader (read-index or lease).

**Q:** Linearizability vs serializability?
**A:** Linearizability: single objects, real-time order. Serializability: multi-object transactions, some serial order, no real-time requirement.

**Q:** Strict serializability?
**A:** Serializable + linearizable: the serial order respects real time.

**Q:** Sequential consistency?
**A:** One total order agreed by all, respecting each client's program order, but not real time.

**Q:** Causal consistency?
**A:** Causally related operations are seen in the same order by everyone; concurrent ones may differ.

**Q:** Strongest consistency model that stays available under partition?
**A:** Causal consistency.

**Q:** How is causal consistency implemented?
**A:** Writes carry dependencies; replicas buffer writes until dependencies are visible.

**Q:** Eventual consistency guarantees?
**A:** If writes stop, replicas eventually converge — nothing about reads in between.

**Q:** Strong eventual consistency?
**A:** Replicas that received the same set of updates are in the same state (CRDTs).

**Q:** Read-your-writes?
**A:** A client always sees its own completed writes.

**Q:** Monotonic reads?
**A:** A client never sees older data after seeing newer data.

**Q:** Monotonic writes?
**A:** A client's writes are applied in the order issued.

**Q:** Writes-follow-reads?
**A:** A write is ordered after the writes the client had read before making it.

**Q:** All four session guarantees together give?
**A:** Roughly causal consistency within one session.

**Q:** How to implement read-your-writes across devices?
**A:** Store last-write version server-side per user; read only from replicas caught up to it.

**Q:** How to implement monotonic reads?
**A:** Sticky routing to one replica, or track the highest version read.

**Q:** Three costs of strong consistency?
**A:** Coordination (round trips), latency (esp. cross-region), availability during partitions.

**Q:** Which "consistency" is CAP's C?
**A:** Linearizability.

**Q:** Common architecture for mixing models?
**A:** Small strongly consistent core (consensus store) + bulk data on weaker, faster models.

**Q:** Is linearizability composable?
**A:** Yes (local property); sequential consistency is not.
