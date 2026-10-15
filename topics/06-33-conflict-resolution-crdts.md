# 33 — Conflict Resolution & CRDTs

> Part 6 — Distributed Systems · Prerequisites: topics 24, 27, 28

Single-leader replication (topic 23) avoids write conflicts by funnelling all writes for a piece of data
through one node. As soon as you allow two places to accept writes to the same data independently, whether
two datacenter leaders, several leaderless replicas, or a phone that edits while offline, two writes can
happen *concurrently*: neither knew about the other. When the replicas later exchange updates, the system
has to decide what the result is. This topic covers where conflicts come from, the easy-but-lossy answer
(last-write-wins), the Dynamo answer (keep both and let the application merge), and the modern answer:
**CRDTs**, data types designed so that concurrent updates always merge automatically and deterministically.
It ends with collaborative editing, where CRDTs compete with the older operational transformation approach.

By the end of this topic you should be able to:
1. Explain where write conflicts come from and how version vectors detect them.
2. Explain exactly what last-write-wins drops, and when it is acceptable anyway.
3. Describe siblings and application-level merge, using the Dynamo shopping cart, including its "deleted items reappear" anomaly.
4. Define a CRDT and explain the difference between state-based and operation-based CRDTs and what each requires.
5. Walk through G-counter, PN-counter, LWW-register and OR-set, including how merge works.
6. Compare operational transformation and CRDTs for collaborative editing.

---

## 1. Where conflicts come from

### 1.1 Concurrent writes

Two writes are **concurrent** if neither happened-before the other (topic 27): the node performing each did
not know about the other. Time doesn't matter; two writes a minute apart are concurrent if the second writer
hadn't seen the first. Conflicts arise from concurrent writes to the same data on different replicas.

Three common sources:

**Multi-leader replication (topic 24).** Each datacenter has a leader that accepts writes and replicates
asynchronously to the others.

```
 DC-Europe leader                          DC-US leader
 t=0  title = "Draft"                      title = "Draft"
 t=1  user A: title = "Plan"               user B: title = "Roadmap"
 t=2  ── replicate "Plan" ──────────────►  conflict: "Roadmap" vs "Plan"
      ◄────────────── replicate "Roadmap" ─ conflict: "Plan" vs "Roadmap"
```

**Leaderless replication (topic 24).** In Dynamo-style stores (Cassandra, Riak, DynamoDB's original
design) any replica can accept a write. Two clients writing the same key through different coordinators,
or a sloppy quorum during a partition, produce divergent replicas.

**Offline clients.** A mobile app, a note-taking app or a calendar on a laptop is effectively a leader that
can be partitioned for hours. Two devices edit the same note offline, then both sync. Collaborative editors
(Google Docs, Figma) are the extreme case: many users with tiny "offline" windows of network latency.

### 1.2 Detecting conflicts: version vectors

To resolve a conflict, you first must know whether two versions are concurrent or one supersedes the other.
Timestamps can't tell you (topic 27). **Version vectors** can: each replica keeps a counter per replica that
has written the value.

```
v1 = {A:2, B:1}     v2 = {A:2, B:2}    → v1 ≤ v2 in every entry: v2 supersedes v1, keep v2
v1 = {A:3, B:1}     v2 = {A:2, B:2}    → neither dominates: concurrent → CONFLICT
```

Comparing vectors gives one of: equal, one dominates (happened-before: just take the newer), or concurrent
(genuine conflict). Every strategy below is about what to do in the third case.

### 1.3 Avoiding conflicts

The cheapest strategy is not to have them:

- Route all writes for a given record to the same leader (e.g. a user's "home" datacenter). Conflicts only
  arise when that routing changes, e.g. on failover.
- Use single-leader replication or consensus for the data that can't tolerate conflicts.

When you can't avoid them, you need **convergence**: all replicas must end up with the same value once
they've seen the same set of updates. Different replicas resolving the conflict differently would be the
worst outcome (permanent divergence).

---

## 2. Last-write-wins (LWW)

### 2.1 Mechanism

Attach a timestamp to every write. On conflict, keep the write with the highest timestamp and discard the
others. Ties are broken deterministically, e.g. by replica ID. Every replica applies the same rule, so all
converge. Cassandra uses LWW per column (cell) by default; many multi-leader setups and sync tools do too.

### 2.2 What it silently drops

LWW guarantees convergence by **throwing away data**:

1. **Concurrent writes.** Of two concurrent writes, one is discarded. The client was told "write
   succeeded", but its data is gone, with no error or notification.

   ```
   client X: SET cart = [book]        ts=100  → ack ✓
   client Y: SET cart = [lamp]        ts=101  → ack ✓
   after merge: cart = [lamp]                   book silently lost
   ```

2. **Causally later writes with smaller timestamps.** With clock skew (topic 27), a write that happened
   *after* another (and saw it) can carry an earlier timestamp and lose. Example: node 1's clock is 3 s
   fast. A writes x=1 on node 1 at real time 10:00:00 (timestamp 10:00:03). B reads x=1 and writes x=2 on
   node 2 at 10:00:01 (timestamp 10:00:01). LWW keeps x=1, discarding the newer write that was
   intentionally overwriting it.

3. **Whole-object granularity.** If the unit is the whole object, two writes that change different fields
   still clash: one user changes the email, another changes the address; one change disappears. Per-field
   LWW (as in Cassandra) shrinks this but doesn't remove it.

4. **Read-modify-write updates.** Two concurrent "increment the counter" writes, each reading 5 and writing
   6, end up as 6, not 7.

### 2.3 When LWW is fine

- Data that is **written once and never modified** (immutable records under unique keys, e.g. a key that is
  a UUID). There are no conflicting versions.
- Values where the latest value really is all that matters and lost concurrent updates are acceptable: a
  "last seen at" timestamp, a cache entry, sensor readings, a user's current profile picture.
- When paired with logical or hybrid clocks (topic 27) to at least preserve causal order, losing only truly
  concurrent writes.

Rule of thumb: LWW is a reasonable choice when you can say out loud "if two people change this at the same
time, it's OK if one change vanishes".

---

## 3. Siblings and application-level merge

### 3.1 Keep all the versions

The alternative to discarding: when versions are concurrent, the database keeps **all of them** as
**siblings** and returns them all on the next read. The application merges them and writes back the result,
which supersedes all the siblings. Riak (with `allow_mult`) and the original Dynamo work this way.

```
read cart  → siblings: { [milk, eggs] (vv {A:2,B:1}),  [milk, flour] (vv {A:1,B:2}) }
app merges → [milk, eggs, flour]
write back with context {A:2,B:2} → single version again
```

The client must send the version vector ("causal context") it read along with its write, so the database
knows which siblings the write replaces.

### 3.2 The Dynamo shopping cart

The Dynamo paper (2007) used the shopping cart as the motivating example. The business priority was: *never
reject an "add to cart"*. The cart had to be writable even during partitions and failures (AP, topic 29).
So concurrent versions of the cart can appear, and Dynamo's resolution was to **merge by union**: the cart
contains every item present in any sibling.

```
            cart = {milk}   vv {A:1}
           /                         \
  replica A: add eggs               replica B: add flour       (partition)
  {milk, eggs} vv {A:2}             {milk, flour} vv {A:1,B:1}
           \                         /
     concurrent → siblings → union → {milk, eggs, flour}   ✓ no add lost
```

**The anomaly: deleted items reappear.** Union handles additions well but has no way to express "this was
removed":

```
 cart = {milk, eggs}       vv {A:2}
 replica A: remove eggs  → {milk}         vv {A:3}
 replica B (concurrent): add flour → {milk, eggs, flour}   vv {A:2,B:1}
 union merge → {milk, eggs, flour}        ← eggs is back!
```

Replica A's version just *lacks* eggs; union can't tell "never added" from "removed". The Dynamo paper
explicitly accepts this ("deleted items can resurface"), since an extra item in a cart is a much cheaper
mistake than a lost one. Fixing it properly needs the merge to know about removals, which leads to tombstones
and to the OR-set in section 4.

### 3.3 Costs of application-level merge

- **Every reader must handle siblings.** Easy to forget, and the merge logic must be correct for each data type.
- **Merge must be deterministic, commutative, associative, idempotent.** If two clients merge the same
  siblings in a different order and get different results, you've created new conflicts. Writing such merge
  functions by hand is exactly what CRDTs systematise.
- **Sibling explosion.** If clients write without passing the causal context (blind writes), every write
  creates a new sibling. Riak deployments hit this in practice.
- **Deletes need tombstones.** To merge removals correctly, you must remember that something was deleted.

### 3.4 Other resolution strategies

- **Custom merge on write.** Some databases let you register a conflict handler that runs when replication
  detects a conflict (e.g. some multi-leader setups with conflict triggers). It can't prompt a user, so it
  must be automatic.
- **Ask the user.** Git merge conflicts, a calendar showing "conflicting copies". Works only when a human can
  be involved and the data is small.
- **Field-level merging / three-way merge.** Compare both versions to a common ancestor; take each side's
  changes to non-overlapping fields; only real overlaps are conflicts. This is what Git and many sync engines do.

---

## 4. CRDTs

### 4.1 The idea

A **CRDT** (Conflict-free Replicated Data Type, Shapiro et al., 2011) is a data type whose operations are
designed so that replicas can be updated independently, without coordination, and **always converge** to the
same state once they've received the same updates, no matter the order or duplication of delivery. This
property is called **strong eventual consistency**: eventual consistency plus a guarantee that replicas that
saw the same updates are in the same state (no "pending conflict" state, no lost updates by design).

Instead of "resolve conflicts after the fact", CRDTs make conflicts impossible by choosing semantics that
are well-defined for every concurrent combination.

### 4.2 State-based vs operation-based

**State-based CRDTs (CvRDTs, convergent).** Replicas periodically send their *whole state* to each other.
The receiver merges with a **merge** (join) function. Requirements on merge:

- **Commutative:** merge(a, b) = merge(b, a)
- **Associative:** merge(a, merge(b, c)) = merge(merge(a, b), c)
- **Idempotent:** merge(a, a) = a

Mathematically, states form a **join-semilattice** and merge computes the least upper bound; every update
only moves state "upward". The consequence: you can deliver states in any order, any number of times, over
an unreliable network (e.g. via gossip, topic 32), and replicas still converge. The cost: sending whole
states can be large. **Delta-state CRDTs** reduce this by sending only the recently changed part, which is
itself mergeable.

**Operation-based CRDTs (CmRDTs, commutative).** Replicas send *operations* ("increment by 1", "add x with
tag t"). Each replica applies the operations it receives. Requirements:

- Concurrent operations must **commute** (applying them in either order gives the same state).
- The delivery layer must deliver **every operation exactly once to every replica**, and usually in
  **causal order** (an operation arrives after operations it depends on). This requires a reliable causal
  broadcast layer, often implemented with version vectors and deduplication.

| | State-based | Operation-based |
|---|---|---|
| What is sent | Full state (or deltas) | Individual operations |
| Network requirements | Weak: any order, duplicates OK, loss OK if resent later | Strong: exactly-once, causal delivery |
| Message size | Proportional to state | Small |
| Natural fit | Gossip / anti-entropy between replicas (Riak, Redis Enterprise CRDBs) | Systems with a reliable log/broadcast (collaborative editors) |

The two styles are equivalent in expressive power; each can emulate the other.

### 4.3 G-counter (grow-only counter)

Problem: count page views across replicas without coordination. A single integer with LWW loses increments.
Instead, keep **one counter per replica** in a vector; each replica only increments its own entry.

```
state: {A: nA, B: nB, C: nC}
increment at replica A:  nA += 1
value:                   nA + nB + nC
merge(x, y):             entry-wise max
```

Example:

```
A: {A:3, B:0, C:0}   B: {A:1, B:2, C:0}
merge → {A:max(3,1), B:max(0,2), C:0} = {A:3, B:2, C:0}   value = 5
```

Why max, not sum? Because merge must be idempotent: receiving the same state twice must not double-count.
Each entry is only ever increased by its owner, so the max of two copies of an entry is simply the newer one.

### 4.4 PN-counter (positive-negative counter)

A G-counter can't decrement: decreasing your entry would be undone by max with an older, higher copy. So use
**two G-counters**: P for increments, N for decrements.

```
increment at A: P[A] += 1
decrement at A: N[A] += 1
value: sum(P) − sum(N)
merge: merge P and N separately (entry-wise max)
```

Example: likes on a post. A: P={A:5}, N={A:1}. B: P={B:3}, N={B:2}. Merged value = (5+3) − (1+2) = 5.

Note a PN-counter can go negative; it can't enforce "stock ≥ 0" because that invariant needs coordination
(two replicas each sell the last unit). Invariants that span replicas are beyond what CRDTs can guarantee
alone; see topic 35 for countermeasures and topic 34 for coordination.

### 4.5 LWW-register

A **register** holds a single value. An **LWW-register** stores (value, timestamp) and merge keeps the one
with the higher timestamp (ties broken by replica ID). It is a valid CRDT: merge is commutative, associative
and idempotent, so replicas converge. But it has exactly the data-loss semantics of section 2: concurrent
assignments lose all but one. CRDTs don't make data loss disappear; they make the resolution *well-defined*.

The alternative is a **multi-value register (MV-register)**: keep all concurrent values (like Dynamo
siblings), tracked with version vectors, and let the reader decide. A later write that has seen all values
replaces them.

### 4.6 Sets: from G-set to OR-set

- **G-set (grow-only set):** add only; merge = union. This is the Dynamo cart without removals.
- **2P-set (two-phase set):** a G-set of adds plus a G-set of removes (tombstones). An element is present if
  it's in adds and not in removes. Flaw: once removed, an element can **never be re-added** (the tombstone
  wins forever).

- **OR-set (observed-remove set):** the standard practical set CRDT, and the fix for the Dynamo cart.
  - **Add(x):** create a unique tag (e.g. replica ID + counter), store (x, tag).
  - **Remove(x):** remove all (x, tag) pairs **this replica currently observes**, i.e. tombstone those tags.
  - **Contains(x):** there's at least one (x, tag) with tag not tombstoned.
  - **Merge:** union of add-tags, union of tombstones.

  Rule in plain words: a remove only cancels the adds it has *seen*. An add concurrent with a remove has a
  new tag the remove didn't see, so it survives. This is called **add-wins** semantics.

Replaying the cart example with an OR-set:

```
cart: {(milk, a1), (eggs, a2)}

replica A: remove eggs → tombstones {a2}
replica B (concurrent): add flour → (flour, b1)

merge: adds {(milk,a1),(eggs,a2),(flour,b1)}   tombstones {a2}
contents: {milk, flour}        ✓ eggs stays removed, flour kept
```

And concurrent add/remove of the same element:

```
replica A: remove eggs (observed tag a2) → tombstone a2
replica B: add eggs again                → (eggs, b7)
merge: eggs present via b7 → add wins
```

Re-adding after removal works too: a fresh add creates a fresh tag. The cost is metadata: tags and
tombstones accumulate. Optimised OR-sets (using version vectors instead of storing every tombstone) keep
this bounded, and garbage collecting tombstones needs knowledge that all replicas have seen them.

### 4.7 Composing CRDTs

CRDTs compose: a map whose values are CRDTs is a CRDT (merge key-by-key). This gives JSON-like documents:
a cart as an OR-map from product ID to a PN-counter quantity, a profile as a map of LWW-registers per field.
Riak's data types (counters, sets, maps, flags, registers), Redis Enterprise's Active-Active databases, and
libraries like Automerge and Yjs are built this way.

### 4.8 Limits of CRDTs

- **Not every semantics is expressible.** Invariants such as "balance ≥ 0", "username unique", or "at most
  10 seats" need coordination between replicas. CRDTs give convergence, not arbitrary invariants.
- **Metadata overhead.** Per-replica vectors and tombstones grow with replicas and history; garbage collection
  is hard because it requires knowing every replica has caught up.
- **Semantics may surprise.** Add-wins vs remove-wins, LWW in registers. "Converges" doesn't mean "the result
  a user expected".

---

## 5. Collaborative editing: OT vs CRDTs

Text editing is the hardest common case: many users insert and delete characters concurrently, and they
expect to see each other's edits within milliseconds while also being able to type during network hiccups.
Each user's editor applies its own edits locally immediately (otherwise typing would lag), so concurrent
edits are the norm.

### 5.1 The problem with positions

```
text: "abc"
user 1: insert "X" at position 1 → "aXbc"
user 2 (concurrent): delete position 2 (the "c") → "ab"

naive apply on user 1's side: "aXbc" then delete pos 2 → "aXc"   ✗ deleted "b" instead of "c"
```

Positions are relative to the document state at the time of the edit. Once another edit has changed the
document, the positions are wrong.

### 5.2 Operational transformation (OT)

OT (developed from the late 1980s; used by Google Docs, and the original Google Wave) **transforms**
incoming operations against concurrent ones that have already been applied, adjusting their positions.

```
transform(delete(2), against insert(1, "X")) → delete(3)
user 1 applies delete(3) to "aXbc" → "aXb"   ✓
user 2 applies insert(1,"X") to "ab" → "aXb"   ✓ converged
```

Properties:

- Works on plain sequences of operations with positions; documents stay compact.
- Correct transformation functions for all pairs of operation types are notoriously hard to get right in a
  fully peer-to-peer setting (several published algorithms were later shown to be incorrect).
- Practical OT therefore uses a **central server** that imposes a single order on operations (Google Docs,
  Jupiter-style protocol). Clients transform against the server's sequence. That makes it simpler and correct,
  but requires the server for syncing; peer-to-peer and long offline periods are awkward.

### 5.3 Sequence CRDTs

A sequence CRDT (RGA, Logoot, LSEQ, the ones in Yjs and Automerge) avoids positions entirely by giving
**every character a unique, stable identifier** and expressing edits relative to identifiers:

- Insert "X" **after character with ID (user1, 4)**, getting new ID (user1, 5).
- Delete **character with ID (user2, 7)** (it becomes a tombstone, still used as an anchor for later inserts).
- A deterministic tie-break orders concurrent inserts at the same spot (e.g. by ID), so all replicas place
  them identically.

Since IDs never change, concurrent edits don't invalidate each other; there's nothing to transform. Replicas
can sync peer-to-peer, in any order, after any amount of offline time.

### 5.4 Comparison

| | Operational transformation | Sequence CRDTs |
|---|---|---|
| Core trick | Transform positions of concurrent ops | Stable unique IDs per element |
| Topology | Practically needs a central server for ordering | Any: peer-to-peer, offline, server-relayed |
| Correctness | Hard to prove for arbitrary peer-to-peer orders | Convergence follows from CRDT properties |
| Metadata | Small; document is plain text | IDs and tombstones per character; modern implementations compress heavily |
| Offline / local-first | Awkward for long offline divergence | Natural fit |
| Examples | Google Docs, older collaborative editors | Yjs, Automerge, many newer editors |

Historically CRDTs were considered too memory-hungry for text; implementations like Yjs and Automerge 2 cut
the overhead dramatically (run-length encoding of IDs, compact binary formats), which is why "local-first"
software has adopted them. Both approaches have a known semantic issue: **interleaving**. Two users typing
words at the same spot concurrently can have their characters interleaved in some algorithms; newer
algorithms specifically avoid it.

Note that neither approach solves *intent* conflicts: if two users rewrite the same sentence differently, the
result converges but may be gibberish. Convergence is a technical property, not editorial agreement.

---

## 6. How it connects

- **Topic 24 (multi-leader & leaderless):** the sources of concurrent writes; read repair and anti-entropy
  propagate state that this topic then merges.
- **Topic 27 (time & clocks):** happens-before defines "concurrent"; version vectors detect it; clock skew is
  why LWW loses causally later writes; HLCs reduce that.
- **Topic 28 (consistency models):** CRDTs give strong eventual consistency, compatible with availability
  during partitions, and can be combined with causal consistency.
- **Topic 29 (CAP):** CRDTs are the main tool for building AP systems that don't lose updates.
- **Topic 32 (gossip):** state-based CRDTs are designed to be spread by gossip and anti-entropy.
- **Topic 34–35 (transactions, sagas):** invariants CRDTs can't keep require coordination (2PC, consensus) or
  compensation.
- **Topic 42 (event sourcing):** operation-based CRDTs resemble replaying a log of commutative events.
- **Topic 8 (real-time communication):** collaborative editors relay operations over WebSockets.

---

## Common misconceptions

- "Last-write-wins means the latest write survives." → It means the write with the highest *timestamp*
  survives. With clock skew that may not be the latest, and concurrent writes are discarded silently either way.
- "Conflicts only happen when writes occur at the same instant." → Concurrency is about knowledge, not time.
  Two writes hours apart are concurrent if an offline device made one without seeing the other.
- "CRDTs eliminate data loss." → They make merges deterministic. An LWW-register CRDT still discards
  concurrent values; the guarantee is convergence.
- "CRDTs can enforce any business rule." → Rules like uniqueness or non-negative balances require coordination.
- "A G-counter merge sums the vectors." → It takes the entry-wise max; summing would double-count on redelivery.
- "Removing from an OR-set removes concurrent adds too." → It only removes the tags it observed; concurrent adds survive (add-wins).
- "OT is obsolete." → It's widely deployed and efficient with a central server; CRDTs win mainly in
  peer-to-peer and offline-heavy settings.

---

## Further reading (supplements)

- Kleppmann, *Designing Data-Intensive Applications*, Ch. 5 ("Handling write conflicts", "Detecting concurrent writes").
- Shapiro, Preguiça, Baquero & Zawirski, "Conflict-free Replicated Data Types" (SSS 2011) and the companion
  technical report "A comprehensive study of Convergent and Commutative Replicated Data Types" (INRIA, 2011).
- DeCandia et al., "Dynamo: Amazon's Highly Available Key-value Store" (SOSP 2007), section on versioning and the shopping cart.
- Martin Kleppmann, "CRDTs: The Hard Parts" (talk, 2020).
- Kleppmann, Wiggins, van Hardenberg & McGranaghan, "Local-first software" (Ink & Switch essay, 2019).
- Ellis & Gibbs, "Concurrency control in groupware systems" (SIGMOD 1989), the origin of OT.
- crdt.tech (community site listing papers and implementations).

---

## Questions

1. Two users edit the same document field from different datacenters one minute apart. Under what condition
   is this a conflict, and how would the system detect it?
2. Give three different ways last-write-wins can lose a write that a client was told succeeded.
3. For which kinds of data is LWW a reasonable choice? Give two examples and one counter-example.
4. Explain the Dynamo shopping cart's merge strategy, why it was chosen, and the anomaly it causes.
5. Why must the client send the version vector (causal context) it read along with its write in a
   sibling-based store? What happens if it doesn't?
6. Define strong eventual consistency and explain how it differs from plain eventual consistency.
7. What three properties must a state-based CRDT's merge function have, and why does each matter on an unreliable network?
8. What does an operation-based CRDT demand from the network layer that a state-based one doesn't?
9. Why does a G-counter merge with max instead of sum? Walk through a redelivery example.
10. Why can't you decrement a G-counter, and how does a PN-counter fix it?
11. Replay the "remove eggs vs add flour" cart scenario with an OR-set. Why do deleted items no longer reappear?
12. An OR-set receives a remove(x) on replica A and a concurrent add(x) on replica B. What's the result, and why?
13. A team wants to use a PN-counter to track remaining inventory and guarantee it never goes below zero. Why won't that work?
14. Why do naive position-based concurrent text edits break, and how does OT fix it?
15. Why do sequence CRDTs fit offline/peer-to-peer editing better than OT, and what do they pay for it?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** It's a conflict if the edits are concurrent: the second edit was made without having seen the first
(e.g. replication lag or a partition meant DC-2 hadn't received DC-1's write yet). Elapsed time is
irrelevant. The system detects it by comparing version vectors: if neither dominates, they're concurrent.

**2.** (a) A concurrent write with a higher timestamp discards it. (b) Clock skew: a causally later write gets
a smaller timestamp than the write it overwrote and loses. (c) Whole-object LWW: a concurrent write to a
different field of the same object replaces the whole object. (Also: concurrent read-modify-write increments
collapse into one.)

**3.** Reasonable for immutable write-once data under unique keys, or values where only the latest matters and
losing a concurrent update is acceptable: a "last seen" timestamp, a cache entry, a current location reading.
Counter-example: a shopping cart, a counter, or a bank balance, where concurrent updates must all count.

**4.** Concurrent cart versions are kept as siblings and merged by union. It was chosen because "add to cart"
must always succeed and must never be lost (availability over consistency). The anomaly: union can't
represent removal, so an item removed on one replica reappears if a concurrent version still contains it.

**5.** The context tells the database which siblings the new write has seen and therefore replaces. Without it,
the database must treat the write as concurrent with all existing siblings, so it adds yet another sibling.
Sibling count grows without bound (sibling explosion) and old values never go away.

**6.** Eventual consistency only says replicas converge eventually if writes stop; it allows temporary
unresolved conflicts and says nothing about how. Strong eventual consistency adds: any two replicas that have
received the same set of updates are in the same state immediately, with no conflict-resolution step, no
rollback and no coordination.

**7.** Commutative (order of state exchange doesn't matter, since messages arrive in any order), associative
(grouping doesn't matter, so states can be merged via intermediaries such as gossip relays), idempotent
(receiving the same state twice changes nothing, so duplicates and retries are harmless).

**8.** Every operation must be delivered exactly once to every replica (non-idempotent ops like "increment"
would double-count or be lost), usually in causal order (e.g. a remove must arrive after the add it refers
to). State-based CRDTs tolerate loss, duplication and reordering.

**9.** Each entry is owned and only increased by one replica, so the max of two copies is the latest value.
Summing would double-count: A has {A:3}, sends it to B, B sums to get {A:3}; then a retry delivers the same
{A:3} again and summing gives {A:6}. With max, the second delivery leaves {A:3}. Merge must be idempotent.

**10.** Merge takes the max per entry, so a decremented entry is replaced by any older, higher copy and the
decrement disappears. A PN-counter keeps two G-counters (increments P and decrements N), both only ever
growing, and reports sum(P) − sum(N).

**11.** Cart adds are tagged: (milk,a1), (eggs,a2). A removes eggs by tombstoning a2. B concurrently adds
(flour,b1). Merge unions the adds and the tombstones: eggs' only tag a2 is tombstoned, so eggs is absent; flour
is present. Deleted items don't reappear because removal is represented explicitly (tombstones) rather than by
absence.

**12.** x is present. The remove only tombstoned the tags A had observed; B's add created a new, unseen tag, which
survives the merge. Add wins under concurrency.

**13.** Two replicas can each see stock = 1 and each decrement locally (each sells the last unit) without
coordination; after merge stock = −1. Keeping a non-negativity invariant requires coordination before the
decrement (consensus, a single owner, reservations/escrow splitting stock between replicas) or compensation
after (topic 35).

**14.** Positions refer to the document state when the op was created; a concurrent insert or delete shifts
characters, so applying the other user's op at the original position hits the wrong character. OT transforms
an incoming op against the concurrent ops already applied (e.g. a delete after an earlier insert has its index
shifted by one), so all replicas converge.

**15.** Sequence CRDTs give every character a stable unique ID and express inserts/deletes relative to IDs, so
concurrent ops never invalidate each other; there's no need for a central ordering server, and replicas can
merge after arbitrary offline periods. The cost is per-character metadata (IDs, tombstones), which modern
implementations compress but can't eliminate, and tombstone garbage collection is hard.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** When are two writes concurrent?
**A:** When neither happened-before the other: each was made without knowledge of the other.

**Q:** Three sources of write conflicts?
**A:** Multi-leader replication, leaderless replication, offline clients.

**Q:** How are concurrent versions detected?
**A:** Version vectors: if neither vector dominates the other, the versions are concurrent.

**Q:** What does last-write-wins do?
**A:** Keeps the write with the highest timestamp, silently discarding the others.

**Q:** Two ways LWW loses data beyond plain concurrent writes?
**A:** Clock skew makes causally later writes lose; whole-object LWW drops changes to other fields.

**Q:** When is LWW acceptable?
**A:** Immutable data, or values where losing a concurrent update is acceptable (last-seen, caches).

**Q:** What are siblings?
**A:** Concurrent versions the database keeps and returns together for the application to merge.

**Q:** Dynamo cart merge rule and its anomaly?
**A:** Union of siblings; deleted items can reappear.

**Q:** What is sibling explosion?
**A:** Writes without causal context each add a new sibling, so siblings grow without bound.

**Q:** What is a CRDT?
**A:** A replicated data type whose concurrent updates always merge deterministically, so replicas converge without coordination.

**Q:** Strong eventual consistency?
**A:** Replicas that received the same updates have the same state, with no conflict resolution step.

**Q:** State-based CRDT merge requirements?
**A:** Commutative, associative, idempotent (a join-semilattice).

**Q:** Operation-based CRDT requirements?
**A:** Concurrent ops commute; exactly-once, typically causal delivery.

**Q:** Delta-state CRDT?
**A:** A state-based CRDT that ships only recently changed parts of the state, which are themselves mergeable.

**Q:** G-counter structure and merge?
**A:** One count per replica; value = sum; merge = entry-wise max.

**Q:** PN-counter?
**A:** Two G-counters (P, N); value = sum(P) − sum(N).

**Q:** LWW-register?
**A:** Value plus timestamp; merge keeps the higher timestamp. A CRDT that still drops concurrent writes.

**Q:** MV-register?
**A:** Keeps all concurrent values, like siblings.

**Q:** 2P-set flaw?
**A:** Once removed, an element can never be re-added.

**Q:** OR-set mechanism?
**A:** Adds get unique tags; remove tombstones the observed tags; element present if any untombstoned tag exists.

**Q:** OR-set semantics under concurrent add and remove?
**A:** Add wins.

**Q:** What can't CRDTs guarantee?
**A:** Cross-replica invariants like uniqueness or non-negative balances.

**Q:** OT core idea?
**A:** Transform the positions of incoming concurrent operations against ones already applied.

**Q:** Sequence CRDT core idea?
**A:** Every character has a stable unique ID; edits reference IDs, not positions.

**Q:** OT vs CRDT topology?
**A:** OT practically needs a central server; CRDTs work peer-to-peer and offline.

**Q:** Main cost of sequence CRDTs?
**A:** Per-character metadata and tombstones.
