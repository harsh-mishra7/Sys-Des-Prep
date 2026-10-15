# 29 — CAP & PACELC

> Part 6 — Distributed Systems · Prerequisites: topics 24, 26, 28

The CAP theorem is the most quoted and most misquoted result in distributed systems. In its popular
form — "Consistency, Availability, Partition tolerance: pick two" — it is misleading at best. Stated
correctly, it is a narrow but important fact: **when the network partitions, a replicated system must
choose between answering requests and keeping its replicas linearizable**. PACELC extends it with the
trade-off that matters far more often in practice: even when the network is healthy, strong consistency
costs latency. This topic states both precisely, walks through what each choice looks like during a
partition, and explains why sticking "CP" or "AP" labels on real databases usually hides more than it
reveals.

By the end of this topic you should be able to:
1. State CAP precisely, including the exact meaning of C, A and P in the theorem.
2. Explain with a two-node example why a partitioned system can't be both linearizable and available.
3. Identify and correct the common misreadings of CAP ("pick two", "CA systems", "AP means no consistency").
4. State PACELC and explain why the latency-vs-consistency trade-off matters more day to day.
5. Reason about where real systems sit, why their behavior is often configurable per operation, and why single labels mislead.

---

## 1. Background

In 2000 Eric Brewer presented a conjecture: a shared-data system can have at most two of Consistency,
Availability and Partition tolerance. Gilbert and Lynch proved a formal version in 2002. Its historical
value was to legitimize the NoSQL wave: database designers had long assumed strong consistency was
non-negotiable, and CAP made it respectable to explore systems that relaxed consistency to stay
available (Dynamo, Cassandra, Riak). But the slogan form spread faster than the theorem.

---

## 2. CAP stated correctly

### 2.1 The precise definitions (Gilbert & Lynch)

- **C — Consistency** means **linearizability** (topic 28): every read sees the most recent completed
  write; the system behaves like a single copy. *Not* ACID consistency, *not* "replicas eventually agree".
- **A — Availability** means **every request received by a non-failed node must eventually receive a
  non-error response**. Note: *every* non-failed node — including the ones on the minority side of a
  partition. It does not mean "99.99% uptime" or "most requests succeed".
- **P — Partition tolerance** means the system continues to operate despite the network dropping or
  delaying an arbitrary number of messages between nodes.

### 2.2 The theorem

> If the network may partition, then a system cannot guarantee both linearizability and availability
> (in the above sense).

Or, as Kleppmann puts it, CAP is better phrased as **"either Consistent or Available when Partitioned"**.

### 2.3 Why P isn't optional

You don't *choose* partition tolerance. Partitions happen (topic 26) whether you like it or not — a
switch fails, a cable is cut, a GC pause makes a node unreachable for 30 s (from the outside, a paused
node is indistinguishable from a partitioned one). So any system spanning more than one machine must
have *some* behavior during a partition. The real choice is between C and A **during** a partition. When
there's no partition, CAP says nothing — you can have both C and A.

### 2.4 The proof, informally

Two replicas, N1 and N2, both holding `x = 0`. The link between them is cut.

```
          client 1                              client 2
             │ write x=1                            │ read x
             ▼                                      ▼
          ┌──────┐        ✂✂✂ partition ✂✂✂      ┌──────┐
          │  N1  │ ─────────────X──────────────── │  N2  │
          │ x=1  │                                 │ x=0  │
          └──────┘                                 └──────┘
```

Client 1 writes `x=1` to N1. Since N1 must be available, it must eventually respond successfully —
without being able to tell N2. Then client 2 reads `x` from N2. N2 must respond (availability). But it
has never heard of `x=1`, so it returns 0 — a read after a completed write that doesn't see it: **not
linearizable**.

The only way to stay linearizable is for N1 to refuse the write, or N2 to refuse the read (error or wait
until the partition heals, which might be forever) — **not available**. There's no third option: N2
cannot know about a write it has no way of hearing about.

---

## 3. What the two choices look like

### 3.1 Choosing consistency (CP behavior)

During a partition, nodes that can't confirm they have the latest data **refuse or block** requests.
Typically the side with a majority of replicas keeps working; the minority side returns errors or
times out.

```
   majority side (3 of 5)          │      minority side (2 of 5)
   - can elect/keep a leader       │      - cannot form a quorum
   - serves reads and writes       │      - rejects writes; rejects or redirects
     linearizably                  │        linearizable reads
```

Examples: etcd, ZooKeeper, Consul (consensus-based, topic 30–31); a single-leader database with
synchronous replication that stops accepting writes when it can't reach the sync replica; HBase,
which is unavailable for a region until it's reassigned. Clients on the minority side see errors —
the system is still "up", but not *available* in the CAP sense.

Use when wrong answers are worse than no answer: locks, leader election, account balances, inventory
that must not oversell, uniqueness.

### 3.2 Choosing availability (AP behavior)

During a partition, every node keeps accepting reads and writes using its local state. Replicas
diverge; when the partition heals, they reconcile — with LWW (losing writes, topic 27), with siblings
the application merges, or with CRDTs (topic 33).

```
   side A                          │      side B
   - accepts writes  x = 1         │      - accepts writes x = 2
   - reads return local value      │      - reads return local value
   ─────────── partition heals ─────────────────────────────────
   anti-entropy: detect x=1 ∥ x=2 → resolve (LWW / siblings / CRDT merge)
```

Examples: Cassandra and Riak at low consistency levels, DynamoDB eventually-consistent reads, multi-leader
replication (topic 24), CouchDB, DNS, caches, offline-first mobile apps (an offline phone *is* a
partitioned replica).

Use when staleness and later reconciliation are acceptable and downtime is costly: shopping carts (the
Dynamo paper's motivating example — never reject "add to cart"), social feeds, metrics, presence.

### 3.3 It's a spectrum, decided per operation

Most systems don't make one global choice:

- Within one database, writes can be CP while reads are AP (e.g. read from any replica, possibly stale).
- Cassandra's per-query consistency level: `ONE` behaves AP-ish; `QUORUM` makes minority-side requests
  fail (but still isn't linearizable — topic 28 — so it isn't "CP" either).
- An application can choose per feature: CP for payments, AP for the product catalog.
- During a partition, a system can degrade gracefully: serve stale reads with a warning, accept writes
  into a queue for later, disable only features that need coordination.

Brewer himself wrote in 2012 ("CAP Twelve Years Later") that the "2 of 3" framing is misleading, and that
the goal should be to maximize combinations of consistency and availability that make sense for the
application, with explicit partition detection, a partition mode, and recovery afterwards.

---

## 4. The common misreadings

### 4.1 "Pick two of three"

Implies three equal options: CA, CP, AP. But P is not a choice — partitions happen. And when there is
no partition, you get both C and A. The real statement is conditional: *if partitioned, C or A.*

### 4.2 "CA systems"

A "CA" system would be one that is linearizable and available but not partition tolerant — i.e. one
that assumes partitions never happen. For a single-node database, that's sort of meaningful (no network
inside it), but then it isn't a distributed system and CAP doesn't apply. For anything distributed, a
"CA" label just means "we haven't specified what happens during a partition" — and in practice, such
systems often do something bad (split brain, lost writes).

### 4.3 "AP means no consistency"

AP systems give up *linearizability* during partitions. They can still provide causal consistency
(topic 28 — the strongest available model), session guarantees, and convergence. "Not linearizable"
≠ "anything goes".

### 4.4 "CP means unavailable"

CP systems are typically available to most clients most of the time; under a partition, the majority
side keeps serving. They're unavailable only to clients that can reach only the minority side (and
briefly during leader elections). Also: CAP-availability is a very strict property; a CP system can
have higher *practical* uptime than an AP system with worse operations.

### 4.5 "CAP explains why my system is slow / why I chose eventual consistency"

CAP is only about behavior *during partitions*. It says nothing about latency, which is the more common
reason systems relax consistency — that's PACELC (section 5).

### 4.6 "Being CP means the system is linearizable"

Many systems advertised as CP aren't linearizable even when healthy (bugs, stale reads from deposed
leaders, non-linearizable quorums). Jepsen testing has found such violations in many databases. "CP" is
often a design intent, not a verified property.

### 4.7 CAP's narrowness

CAP considers only one consistency model (linearizability) and one fault (network partition). It says
nothing about node crashes, latency, disk failures, or weaker models — and its "availability" ignores how
*fast* responses are (a response after 10 minutes counts). For these reasons Kleppmann argues CAP is of
mostly historical interest for system design, superseded by more precise results (e.g. the
availability/consistency taxonomy in Bailis et al.'s "Highly Available Transactions", and Attiya–Welch
latency bounds).

---

## 5. PACELC

### 5.1 The statement

Daniel Abadi (2010/2012) proposed:

> **If** there is a **P**artition, choose between **A**vailability and **C**onsistency;
> **E**lse (normal operation), choose between **L**atency and **C**onsistency.

```
                 ┌── yes ──▶  A  or  C ?      (CAP)
  partition? ────┤
                 └── no  ──▶  L  or  C ?      (the everyday trade-off)
```

### 5.2 Why the "else" half matters more

Partitions are rare; normal operation is constant. The latency cost of strong consistency (topic 28,
section 7) is paid on **every** request:

- A linearizable write must be acknowledged by a quorum (or a leader that confirmed leadership). If
  replicas span regions, that's ≥ one cross-region round trip (~50–150 ms) per write.
- A linearizable read must contact the leader or a quorum, instead of the nearest replica.
- A weakly consistent system can answer from the local replica in ~1 ms and replicate asynchronously.

This is the real reason most systems with weak consistency chose it: not because of partitions, but
because synchronous coordination on every request is slow. Dynamo's design is primarily about latency
at the tail (p99.9), with partition behavior a related benefit.

The latency trade-off is also *continuous*, unlike CAP's binary: you can wait for 1, 2, or all
replicas; for the local region's quorum or a global one; you can bound staleness ("reads at most 10 s
old") — each is a point on the L/C curve.

### 5.3 PACELC classifications (with caveats)

Abadi's examples, roughly:

| System | If Partition | Else | Notes |
|---|---|---|---|
| Dynamo, Cassandra, Riak (default settings) | PA | EL | Low-latency local reads/writes; reconcile later |
| VoltDB, H-Store, BigTable/HBase | PC | EC | Consistency always; pay in availability and latency |
| MongoDB (classic default) | PA* | EC | Single-primary writes; behavior during failover can lose acknowledged writes depending on write concern |
| PNUTS (Yahoo) | PC | EL | Per-record master; timeline consistency |
| Spanner | PC | EC | Paxos + TrueTime; Google reduces partitions with a private network so it's "effectively CA" |

\* Classifications of MongoDB vary by version and configuration (write concern, read concern); Abadi
originally labeled it PA/EC.

Spanner is an instructive case: Brewer (then at Google) argued Spanner is technically CP but "effectively
CA", because Google's private network makes partitions rare enough that availability exceeds five nines
— the C/A choice is made, but it almost never has to be exercised.

---

## 6. Why the labels are often misleading

1. **Configurable behavior.** Cassandra with `ALL` writes is very different from `ONE`. DynamoDB has
   strongly consistent reads on request. MongoDB's guarantees depend on read and write concern. A single
   label describes, at best, a default.

2. **Different operations, different properties.** A system may be linearizable for single-key writes,
   not for multi-key transactions; consistent for writes but serve stale reads from followers.

3. **Neither C nor A in the CAP sense.** Many systems are neither linearizable (so not "C") nor
   available on every node during a partition (so not "A"). A single-leader database with asynchronous
   replication: the minority side (cut off from the leader) can't write — not A; followers serve stale
   reads and failover may lose acknowledged writes — not C. Cassandra at `QUORUM`: minority-side
   requests fail, and quorums aren't linearizable. These systems are just "P", making neither guarantee.

4. **Partial partitions and real faults.** Real failures are messy: asymmetric partitions, one slow
   node, a flapping link. Behavior under these is determined by timeouts and implementation details, not
   by a label.

5. **The label says nothing about correctness.** Jepsen tests have shown systems that claimed CP losing
   acknowledged writes during partitions or leader changes, and "AP" systems losing data via LWW. What
   matters is the system's *documented and tested* behavior under specific failures.

6. **The application layer changes the picture.** An AP database with an application that does
   compare-and-set via a lightweight-transaction (Paxos) path, or a CP database fronted by a cache, has
   different end-to-end properties than the label suggests.

Better questions to ask about a system than "is it CP or AP?":

- What consistency model does each operation provide (linearizable, sequential, causal, RYW, eventual)?
- What happens to a write on the minority side of a partition: rejected, accepted and later merged,
  or accepted and possibly lost?
- Can an acknowledged write be lost during failover? Under which settings?
- What does a read cost (which replicas, how many round trips), and what staleness can it see?
- How are conflicts resolved (LWW, siblings, CRDTs)?

---

## 7. A worked scenario: shopping site across two regions

A shopping site runs replicas in region East and region West. The inter-region link fails for 20 minutes.

**Cart service, AP choice.** Both regions accept "add to cart". A user whose requests bounce between
regions might add item A in East and item B in West. On heal, the carts are merged (union — topic 33),
so both items appear. Risk: a removed item reappears (the classic Dynamo anomaly), which is an
acceptable annoyance.

**Inventory for a limited sale, CP choice.** Stock counts live in a consensus group with 3 replicas in
East and 2 in West. East has a majority, so it keeps selling; West cannot reach a quorum and returns
"try again later" for purchases. No overselling. West users see errors for 20 minutes.

**Product catalog, AP with bounded staleness.** Both regions keep serving their local copy; price
changes made during the partition propagate after heal. A cache TTL means staleness is bounded anyway
during normal operation.

**Else (no partition), PACELC.** The cart writes locally in ~2 ms (EL). Inventory decrements need a
cross-region quorum round trip (~70 ms) on every purchase (EC) — acceptable for checkout, unacceptable
for every page view. That's why the catalog doesn't go through consensus.

The point: the CAP/PACELC choice is made **per data item and operation**, according to the cost of
being wrong versus the cost of being slow or unavailable.

---

## 8. How it connects

- **Topic 01**: "availability" as uptime vs CAP's strict definition.
- **Topic 23**: single-leader failover and split brain — what happens when partition handling is
  undefined.
- **Topic 24**: leaderless and multi-leader systems are the classic PA/EL designs; sloppy quorums are an
  availability-over-consistency choice.
- **Topic 26**: partitions and pauses are why P isn't optional; you can't distinguish them.
- **Topic 28**: C means linearizability; causal consistency is the strongest model achievable while
  available; the cost-of-consistency discussion is PACELC's "EL vs EC".
- **Topic 30** (next): consensus is how CP systems keep the majority side working.
- **Topic 33**: CRDTs and merges are how AP systems reconcile after a partition.
- **Topic 34–35**: distributed transactions (CP-style) vs sagas (availability-friendly) make the same
  trade-off at the application level.

---

## Common misconceptions

- **"CAP: pick any two."** → Partitions aren't optional; the choice is C vs A *during* a partition. Without
  one, you can have both.
- **"My database is CA."** → For a distributed system, that means partition behavior is unspecified, not
  that partitions can't happen.
- **"C in CAP is the C in ACID."** → CAP's C is linearizability; ACID's C is application invariants.
- **"A in CAP means high uptime."** → It means every non-failed node returns a non-error response —
  much stricter, and not the same as SLA availability.
- **"AP systems have no consistency guarantees."** → They can offer causal consistency, session
  guarantees, and convergence.
- **"CP systems go down during partitions."** → The majority side keeps serving; only minority-side
  clients are affected.
- **"We chose eventual consistency because of CAP."** → Usually the real reason is latency (PACELC's
  "else" branch), which affects every request.
- **"Labelling a system CP or AP tells you how it behaves."** → Behavior depends on configuration and
  per-operation settings, and many systems are neither C nor A; check the documented, tested behavior.

---

## Further reading (supplements)

- DDIA, Chapter 9, sections "Linearizability" and "The Cost of Linearizability" (including the CAP discussion).
- Seth Gilbert & Nancy Lynch, "Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services" (2002).
- Eric Brewer, "CAP Twelve Years Later: How the 'Rules' Have Changed" (IEEE Computer, 2012).
- Daniel Abadi, "Consistency Tradeoffs in Modern Distributed Database System Design" (IEEE Computer, 2012) — PACELC.
- Martin Kleppmann, "Please stop calling databases CP or AP" (blog post, 2015).
- Eric Brewer, "Spanner, TrueTime and the CAP Theorem" (Google, 2017).
- Jepsen analyses (jepsen.io) for real systems' behavior under partitions.

---

## Questions

1. Define C, A and P as used in the CAP theorem. How does CAP's "A" differ from "99.99% availability"?
2. Walk through the two-node proof of CAP. What exactly must each node do to be "available", and why does
   that break linearizability?
3. Why is "pick two of three" a misleading formulation? Why is P not optional?
4. What does it mean if a vendor says its distributed database is "CA"?
5. A five-node CP cluster partitions into groups of 3 and 2. What happens to clients on each side?
6. An "AP" system: what consistency guarantees can it still offer during a partition?
7. State PACELC. Why does the "else" branch affect users more than the "partition" branch?
8. A single-leader database with asynchronous replication: is it CP or AP? Justify.
9. Why might Spanner be described as "effectively CA"? Is that a contradiction of CAP?
10. Why is Cassandra with QUORUM reads and writes neither "C" nor "A" in the CAP sense?
11. List four questions about a database's behavior that are more useful than "is it CP or AP?".
12. A shopping site has a cart, a payment ledger, and product reviews. For each, which side of CAP and
    PACELC would you choose, and why?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** C: linearizability — reads see the latest completed write, as if there were one copy. A: every
request to a non-failed node eventually gets a non-error response. P: the system keeps operating
despite arbitrary message loss between nodes. CAP-A is a per-node, every-request guarantee during a
partition; "99.99% availability" is a statistical uptime measure that allows some requests (e.g. those
hitting a minority side) to fail.

**2.** N1 and N2 both have x=0 and can't communicate. A client writes x=1 to N1; to be available N1 must
acknowledge it, without informing N2. Another client then reads x from N2; to be available N2 must
answer, and it only knows x=0. A read after a completed write returns the old value — not linearizable.
Staying linearizable would require N1 or N2 to refuse or block, violating availability.

**3.** It suggests three symmetric choices. But network partitions happen regardless of preference, so
any distributed system must handle them; the actual choice is only between C and A while a partition is
happening. Without a partition you can have both C and A.

**4.** Either it's really a single-node system (not distributed, CAP irrelevant), or it hasn't specified
what happens during a partition — which typically means something undesirable like split brain or lost
writes when a partition does occur.

**5.** The 3-node side has a majority: it keeps (or elects) a leader and serves reads and writes
linearizably. The 2-node side can't form a quorum: writes are rejected or time out, and linearizable
reads are rejected or redirected (stale reads might be allowed if explicitly requested). When the
partition heals, the minority nodes catch up from the leader.

**6.** Causal consistency (the strongest model available under partition), session guarantees such as
read-your-writes and monotonic reads (if clients stick to one side), and eventual convergence via
version vectors, siblings or CRDTs.

**7.** If partitioned, choose availability or consistency; else, choose latency or consistency. Partitions
are rare, but normal operation is constant: linearizability costs coordination round trips on every
request (quorum or leader contact, possibly cross-region), so the latency trade-off affects every user,
every request.

**8.** Neither, strictly. During a partition, nodes cut off from the leader can't accept writes — not
available. Followers serve stale reads, and on failover asynchronously replicated writes acknowledged
by the old leader can be lost — not linearizable. It's a P system that guarantees neither C nor A in
the CAP sense, though it may have high practical uptime.

**9.** Spanner chooses consistency during partitions (it's CP), but Google's private, redundant network
makes partitions extremely rare, so the system's measured availability is very high — users rarely
experience the C-over-A choice. It's not a contradiction: CAP still applies; it's just that the
partition branch is rarely exercised.

**10.** Not C: quorum reads during an in-flight write can return new then old values (topic 28), and LWW
with clock skew can drop writes. Not A: nodes on a side without enough replicas for a quorum return
errors for QUORUM requests.

**11.** For example: which consistency model each operation offers; what happens to writes on a
minority side (rejected, merged later, possibly lost); whether acknowledged writes can be lost during
failover and under which settings; what a read costs in replicas/round trips and how stale it can be;
how conflicts are resolved.

**12.** Cart: AP and EL — always accept adds, merge carts later (union/CRDT), low latency matters for
UX. Payment ledger: CP and EC — wrong balances or double charges are worse than temporarily rejecting a
payment; accept the latency of quorum writes. Reviews: AP and EL — staleness of seconds or minutes is
harmless; serve locally, propagate asynchronously.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** CAP stated correctly?
**A:** When the network is partitioned, a system can't be both linearizable and available.

**Q:** What does C mean in CAP?
**A:** Linearizability.

**Q:** What does A mean in CAP?
**A:** Every request to a non-failed node eventually gets a non-error response.

**Q:** Why isn't P a choice?
**A:** Partitions happen anyway; you only choose behavior during them.

**Q:** What does CAP say when there's no partition?
**A:** Nothing — you can have both C and A.

**Q:** Who proved CAP formally, and when?
**A:** Gilbert and Lynch, 2002 (from Brewer's 2000 conjecture).

**Q:** CP behavior during a partition?
**A:** Majority side serves; minority side rejects or blocks requests.

**Q:** AP behavior during a partition?
**A:** All sides accept requests; replicas diverge and reconcile after healing.

**Q:** Problem with "CA" label?
**A:** It means partition behavior is unspecified (or the system isn't distributed).

**Q:** Strongest consistency an AP system can keep?
**A:** Causal consistency.

**Q:** PACELC?
**A:** If Partition: A vs C; Else: Latency vs Consistency.

**Q:** Why does PACELC's else branch matter more?
**A:** Coordination latency is paid on every request; partitions are rare.

**Q:** PACELC class of Cassandra/Dynamo defaults?
**A:** PA/EL.

**Q:** PACELC class of Spanner / HBase?
**A:** PC/EC.

**Q:** Why is Spanner "effectively CA"?
**A:** It's CP, but Google's network makes partitions so rare that availability is extremely high.

**Q:** Is async single-leader replication CP or AP?
**A:** Neither — minority can't write (not A), stale reads and lost writes on failover (not C).

**Q:** Is Cassandra QUORUM linearizable?
**A:** No.

**Q:** Why do CP/AP labels mislead?
**A:** Behavior is configurable per operation, many systems are neither, and labels don't reflect tested behavior.

**Q:** Who wrote "CAP Twelve Years Later"?
**A:** Eric Brewer (2012) — says "2 of 3" is misleading.

**Q:** What's the real reason most systems relax consistency?
**A:** Latency, not partitions.

**Q:** Better question than "CP or AP?"
**A:** What consistency does each operation give, and what happens to writes during partitions and failover?
