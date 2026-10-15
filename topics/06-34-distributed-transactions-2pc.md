# 34 — Distributed Transactions & Two-Phase Commit

> Part 6 — Distributed Systems · Prerequisites: topics 17, 21, 25, 26, 30

Topic 21 described transactions on one database: a group of writes that either all happen or none do
(atomicity), with the write-ahead log (topic 17) making that true even if the machine crashes mid-way. Once
data is spread across several machines, either because one database is partitioned (topic 25) or because a
business operation touches two different systems (an orders database and a payments database), that
single-node machinery no longer suffices. Each node can commit or abort its own part, but nothing makes them
agree. This topic is about the **atomic commit problem** and its classic solution, **two-phase commit
(2PC)**: how it works, why it can get stuck, why its "fix" (three-phase commit) is rarely used, how it shows
up in practice (XA), and how modern databases like Spanner combine 2PC with consensus so that it doesn't get
stuck.

By the end of this topic you should be able to:
1. State the atomic commit problem and why it is hard when nodes and networks can fail.
2. Walk through 2PC's prepare and commit phases, including what each participant writes to its log and when.
3. Explain why 2PC blocks when the coordinator fails, and what "in doubt" transactions cost in practice.
4. Explain what three-phase commit tries to fix and why it isn't used much.
5. Describe XA transactions and their practical problems.
6. Explain how Spanner-style systems make 2PC non-blocking by replicating coordinator and participants with consensus.
7. Distinguish atomic commit from consensus, and 2PC from 2PL.

---

## 1. The atomic commit problem

### 1.1 Statement

A transaction writes to several nodes (participants). We want **all participants to commit, or all to
abort**. More precisely, an atomic commit protocol must provide:

- **Agreement:** no two participants decide differently (one commits, another aborts).
- **Validity:** if any participant votes "no" (can't commit), the outcome is abort. If all vote "yes" and
  there are no failures, the outcome is commit.
- **Finality:** once a participant decides, it can't change its mind.
- **Termination:** every participant eventually decides (ideally, even if some nodes fail).

### 1.2 Why it is hard

Consider a shopping site: placing an order inserts into `orders` on shard 1 and decrements stock in
`inventory` on shard 2.

Naive approach: send "commit" to both.

```
 app ── commit ──► shard 1: commits order ✓
     ── commit ──► shard 2: stock constraint fails (out of stock) → aborts ✗
 result: order exists, stock not decremented. Inconsistent.
```

Other ways the naive approach breaks:

- Shard 2 crashes before committing; shard 1 already committed.
- The message to shard 2 is lost; the app can't tell whether it arrived (topic 26).
- The app crashes after sending commit to shard 1 only.

Once a node commits, the change is visible to other transactions and durable; it can't be undone without a
new, compensating transaction (that's the saga approach of topic 35). So a node must not commit until it
knows *every* node will commit. That requires a round of agreement before anyone commits. That round is
the "prepare" phase.

---

## 2. Two-phase commit

### 2.1 Roles

- **Coordinator** (transaction manager): drives the protocol. Often a component of the application server,
  a library, or one of the database nodes.
- **Participants** (resource managers): the databases or shards holding the data.

The transaction first runs normally on each participant (reads, writes, taking locks; topic 22). Only when the
application says "commit" does 2PC start.

### 2.2 The protocol

```
  coordinator                    participant P1            participant P2
      │                               │                         │
      │ [log: txn T started]          │                         │
      │ (transaction work, locks held on each participant)       │
      │                               │                         │
      │ ──── PREPARE(T) ─────────────►│                         │
      │ ──── PREPARE(T) ───────────────────────────────────────►│
      │                               │ check constraints,      │
      │                               │ write all changes + "prepared T"
      │                               │ to WAL, fsync           │ (same)
      │ ◄─────────── YES ─────────────│                         │
      │ ◄─────────── YES ───────────────────────────────────────│
      │                                                          │
      │ [log: COMMIT T, fsync]  ← the commit point               │
      │                                                          │
      │ ──── COMMIT(T) ──────────────►│ commit, release locks    │
      │ ──── COMMIT(T) ────────────────────────────────────────►│ commit, release locks
      │ ◄──────────── ack ────────────│                         │
      │ ◄──────────── ack ──────────────────────────────────────│
      │ [log: T done; may forget it]                             │
```

**Phase 1, prepare (voting):**

1. The coordinator sends PREPARE to every participant.
2. Each participant checks whether it can commit: constraints satisfied, no conflicts, disk space, etc.
3. If yes, it **durably** writes everything needed to commit (the changes and a "prepared" record) to its log,
   and replies YES. From this moment, the participant has **given up the right to abort unilaterally**. It
   must be able to commit even if it crashes and restarts, and it must keep the transaction's locks.
4. If no, it replies NO and may abort immediately.

**Phase 2, commit/abort (decision):**

5. If all participants voted YES, the coordinator writes COMMIT to its own log and fsyncs. **This log write
   is the commit point**: the transaction is now committed, irrevocably, even though no participant has
   committed yet.
6. If any voted NO, or didn't respond within a timeout, the coordinator decides ABORT (and logs it).
7. The coordinator sends the decision to all participants and **retries forever** until each acknowledges.
   A participant that crashed applies the decision after recovery.

### 2.3 The two promises

2PC works because of two points of no return:

- A participant's YES vote is a promise: "I will commit if told to, whatever happens to me."
- The coordinator's logged decision is a promise: "This is the outcome, and I'll keep telling everyone until
  they hear it."

Compare with a wedding ceremony: the officiant asks each party "do you?" (prepare); each says "I do"
(yes vote, and they can't take it back); the officiant pronounces them married (commit point); the
paperwork follows.

### 2.4 Crash recovery cases

| Failure | What happens |
|---|---|
| Participant crashes before voting | Coordinator times out → abort. |
| Participant crashes after voting YES | On restart it finds "prepared T" in its log, re-acquires locks, and asks the coordinator for the outcome. It must not decide alone. |
| Participant votes NO | Abort for everyone. |
| Coordinator crashes before sending PREPARE / before the decision is logged | On restart, there's no commit record → abort (participants that voted YES must wait until then). |
| Coordinator crashes after logging COMMIT | On restart, it reads its log and resends COMMIT to everyone. |
| Commit message lost | Coordinator retries; participants stay prepared until it arrives. |

All these are safe. The problem is *liveness* during the waiting.

---

## 3. The blocking problem

### 3.1 In-doubt participants

Look at a participant that has voted YES and is waiting for the decision. Now the coordinator crashes (or is
partitioned away).

```
coordinator: PREPARE → P1 YES, P2 YES → logs COMMIT → crashes before sending anything
P1: prepared, waiting...     locks held on order rows
P2: prepared, waiting...     locks held on stock rows
```

What can P1 do?

- **Commit on its own?** Unsafe: maybe P2 voted NO and the coordinator decided abort.
- **Abort on its own?** Unsafe: maybe the coordinator logged COMMIT and already told P2, which committed.
- **Ask P2?** Helps only sometimes. If P2 already received a decision, P1 can follow it. If P2 is also just
  prepared, neither knows. If P2 is unreachable, P1 knows nothing.

So P1 is **in doubt** (or "uncertain") and must wait for the coordinator to come back. This is the
**blocking** property of 2PC: a coordinator failure at the wrong moment stops participants from making
progress, potentially for a long time.

### 3.2 Why blocking hurts

An in-doubt transaction keeps its **locks** (it must, because it might still commit, and other
transactions must not see or overwrite its uncommitted writes; topic 22). So:

- Other transactions touching those rows block too, piling up behind the locks.
- If the coordinator's log is lost entirely (disk failure), the in-doubt transactions can stay forever. Database
  administrators then must **heuristically** decide: force commit or force abort by hand
  (`COMMIT PREPARED` / `ROLLBACK PREPARED` in Postgres, "heuristic decisions" in XA), risking atomicity.
- Restarting the database doesn't help: prepared state is durable, so the locks come back after restart.

Postgres exposes this via `PREPARE TRANSACTION 'id'` (stored in `pg_prepared_xacts`) and warns that
forgotten prepared transactions hold locks and prevent VACUUM from cleaning old row versions, and
`max_prepared_transactions` is 0 by default to stop casual use.

### 3.3 Other costs of 2PC

- **Latency:** at least two round trips to all participants plus several fsyncs (participants at prepare,
  coordinator at decision, participants at commit) on the critical path.
- **Lock duration:** locks are held across both phases, including network round trips, which reduces
  throughput on hot rows.
- **Availability:** the transaction succeeds only if *every* participant is up and reachable. With N
  participants each 99.9% available, the transaction path is roughly 0.999^N available. Unlike quorum
  protocols, 2PC needs everyone.
- **Coordinator is a single point of failure** (unless replicated; section 6).

### 3.4 2PC is not consensus (and not 2PL)

- **2PC vs consensus.** Consensus (topic 30) needs only a majority to agree on *some* value proposed by anyone;
  it makes progress if a majority is alive. Atomic commit requires *every* participant to vote yes, because
  each one has a veto (it might be unable to commit). That's why 2PC can't just use a majority. But you can use
  consensus *inside* 2PC to make each role fault-tolerant (section 6).
- **2PC vs 2PL.** Two-phase *locking* (topic 22) is an isolation mechanism on one node: acquire locks while
  growing, release while shrinking, giving serializability. Two-phase *commit* is an atomicity protocol across
  nodes. Distributed databases often use both: 2PL (or MVCC) for isolation within each participant, 2PC for
  atomic commit across them. Similar names, unrelated problems.

---

## 4. Three-phase commit and XA

### 4.1 Three-phase commit (3PC)

3PC (Skeen, 1981) tries to remove blocking by inserting a phase between voting and committing:

```
1. CanCommit?  → participants vote yes/no         (like prepare, but nothing is locked into "must commit" yet)
2. PreCommit   → coordinator tells all "everyone voted yes"; participants ack
3. DoCommit    → participants commit
```

The idea: a participant in the "pre-committed" state knows everyone voted yes; a participant that never got
PreCommit knows no decision to commit could have been made without it. So if the coordinator fails, the
remaining participants can look at each other's states and determine a safe outcome, using timeouts:
if anyone is pre-committed, commit; otherwise abort.

Why it isn't used much:

- **It assumes a synchronous network with bounded delays** and a perfect failure detector (topics 26, 32).
  It relies on timeouts meaning "that node has crashed". In a real network with partitions, a timeout may
  just mean a slow link. Under a partition, participants on one side can decide commit while those on the
  other side decide abort: atomicity violated. Blocking is replaced by the possibility of inconsistency.
- **It adds another round trip** to every transaction.
- **Better solutions exist:** making the coordinator fault-tolerant via consensus (section 6) gives
  non-blocking commit that is correct under partitions, as long as a majority is reachable.

3PC is mainly of historical and educational value.

### 4.2 XA: 2PC across heterogeneous systems

**XA** (X/Open Distributed Transaction Processing standard, 1991) standardises the interface between a
**transaction manager** (coordinator) and **resource managers** (databases, message brokers) so a single
transaction can span, e.g., Postgres, MySQL and a JMS message broker. It's exposed in Java as JTA; Postgres,
MySQL (InnoDB), Oracle, SQL Server and IBM MQ support it.

Typical use: "consume a message from the queue and write to the database, atomically". Both systems are
enlisted in an XA transaction; either the message is acknowledged and the row written, or neither.

XA in practice has a poor reputation:

- **The coordinator usually lives inside the application server** (a library), so its log is on an app
  server disk. If that app server dies or is replaced (common in containerised deployments), in-doubt
  transactions can stay stuck with locks held in the databases until someone recovers that log.
- **Lowest common denominator.** XA can't detect deadlocks across systems (each resource manager sees only its
  own locks), and it doesn't support serializable isolation across systems (SSI needs conflict detection
  across all participants; topic 22).
- **Performance.** Extra round trips and fsyncs; distributed transactions in MySQL have been reported to be
  over ten times slower than single-node ones (a figure DDIA cites).
- **Amplified failures.** Any participant being slow or down makes the whole transaction fail or block, so
  coupling between systems increases.
- **Limited support** in modern infrastructure: Kafka, most NoSQL stores and most cloud-managed services don't
  participate in XA.

Because of this, cross-system atomicity in modern systems is typically achieved differently: the
**transactional outbox** (write the DB row and an outbox row in one local transaction, then publish; topic 39),
**idempotent consumers** (topic 38), and **sagas** (topic 35).

### 4.3 Database-internal distributed transactions

Distributed transactions *within* one distributed database (all participants run the same software) are a
different story. The database can integrate 2PC with its replication, concurrency control and deadlock
detection, and can replicate the coordinator. This is how Spanner, CockroachDB, YugabyteDB, TiDB and
FoundationDB-style systems offer multi-shard ACID transactions with reasonable performance.

---

## 5. How 2PC and consensus combine (Spanner-style)

### 5.1 The idea

2PC blocks because a single machine (the coordinator) holds the decision, and individual participants can
be unavailable. Fix both by making each **role a replicated state machine**:

- Each shard (participant) is a **Raft/Paxos group** of, say, 3 or 5 replicas with a leader. The shard's
  "prepared" state and its commit are entries in its replicated log.
- The coordinator is itself one of the participant groups (the leader of one shard's group acts as the
  coordinator), so its decision is also written to a replicated log.

```
        Paxos group for shard A (coordinator)        Paxos group for shard B (participant)
       ┌────────────────────────────────┐          ┌────────────────────────────────┐
       │ leader A1   replica A2   A3    │          │ leader B1   replica B2   B3    │
       └──────┬─────────────────────────┘          └──────┬─────────────────────────┘
              │  2PC messages between group LEADERS        │
              │◄────────── PREPARED (logged via Paxos in B)┤
              │ log COMMIT via Paxos in A                  │
              ├──────────── COMMIT ──────────────────────►│ log COMMIT via Paxos in B
```

What changes:

- A **"node failure" is now a leader failure inside a group**, which consensus handles in seconds by
  electing a new leader that has the full log, including prepared and decision records.
- The coordinator's decision survives the coordinator machine dying, because it's replicated. A new leader
  of the coordinator group knows the decision and resumes sending it. **No indefinite blocking** unless a
  majority of a group is lost.
- The 2PC protocol logic itself is unchanged; it just runs on top of highly available "virtual nodes".

This is what Gray and Lamport formalised as **Paxos Commit** (2006): use consensus to make the commit
decision fault-tolerant. Spanner (2012) is the best-known production design; CockroachDB uses a variant with
"parallel commits" that reduces latency by letting the transaction be considered committed as soon as all
writes are durably replicated, with the explicit commit record written asynchronously.

### 5.2 Where Spanner's other pieces fit

- **Locks within each group:** the participant leaders hold locks (2PL for read-write transactions) during the
  2PC phases.
- **TrueTime and commit-wait (topic 27):** the coordinator picks a commit timestamp and waits out the clock
  uncertainty before releasing locks, so the commit order matches real time. That gives **external
  consistency** (strict serializability, topic 28) across shards and lets read-only transactions read a
  consistent snapshot at a timestamp without locks or 2PC.
- **Single-shard optimisation:** a transaction touching only one group needs no 2PC, just one Paxos write.
  Schema and key design that keep most transactions within one shard (topic 25) matters for performance.

### 5.3 Costs that remain

- Latency: each 2PC step now involves a consensus round within each group, plus commit-wait. Cross-region
  groups make this tens to hundreds of milliseconds.
- Locks held across those rounds, so contention on hot keys still hurts.
- Everything must be inside one database system. You can't put a third-party payment API into a Paxos group;
  for that, sagas (topic 35).

---

## 6. How it connects

- **Topic 17 (WAL):** prepared and commit records are WAL entries; durability of votes and decisions depends on fsync.
- **Topic 21 (transactions):** 2PC extends atomicity across nodes; isolation is a separate concern.
- **Topic 22 (concurrency control):** participants hold 2PL locks across phases; in-doubt transactions block
  others; 2PC ≠ 2PL.
- **Topic 25 (partitioning):** cross-shard transactions need 2PC; good partitioning keeps most transactions on one shard.
- **Topic 26 (faults):** lost messages and "slow vs dead" cause the in-doubt state; 3PC's synchrony assumption fails here.
- **Topic 27 (clocks):** TrueTime and commit-wait give Spanner external consistency on top of 2PC.
- **Topic 28 (consistency models):** Spanner offers strict serializability across shards.
- **Topic 30 (consensus):** consensus makes coordinators and participants fault-tolerant; atomic commit itself
  needs unanimity, not a majority.
- **Topic 35 (sagas):** the alternative for long-lived or cross-service transactions without distributed locks.
- **Topic 38–39 (idempotency, outbox):** the practical replacements for XA across heterogeneous systems.

---

## Common misconceptions

- "2PC guarantees transactions never get stuck." → 2PC is safe but blocking: a coordinator failure after
  participants vote YES leaves them in doubt, holding locks, until it recovers.
- "A prepared participant can time out and abort." → After voting YES it has given up that right; aborting
  alone could violate atomicity.
- "The transaction commits when participants commit." → It commits when the coordinator durably logs the
  COMMIT decision; participant commits merely carry it out.
- "3PC solves blocking, so it's strictly better." → Only under synchronous-network assumptions; with partitions
  it can violate atomicity. Consensus-replicated 2PC is the practical non-blocking solution.
- "2PC is a consensus algorithm." → Atomic commit requires every participant's yes; consensus needs only a
  majority. They solve different problems (and can be combined).
- "2PC and 2PL are the same thing." → One is atomic commit across nodes, the other is lock-based isolation.
- "Distributed transactions are always too slow to use." → XA across heterogeneous systems often is; well-engineered
  database-internal ones (Spanner, CockroachDB) are practical, especially when most transactions are single-shard.

---

## Further reading (supplements)

- Kleppmann, *Designing Data-Intensive Applications*, Ch. 9 ("Atomic Commit and Two-Phase Commit",
  "Distributed Transactions in Practice").
- Gray & Lamport, "Consensus on Transaction Commit" (ACM TODS, 2006), introduces Paxos Commit.
- Corbett et al., "Spanner: Google's Globally-Distributed Database" (OSDI 2012).
- Bernstein, Hadzilacos & Goodman, *Concurrency Control and Recovery in Database Systems* (1987), Ch. 7 on
  atomic commitment (free online).
- Skeen, "Nonblocking Commit Protocols" (SIGMOD 1981), the 3PC paper.
- PostgreSQL documentation: `PREPARE TRANSACTION` and two-phase transactions.
- CockroachDB blog: "Parallel Commits: An atomic commit protocol for globally distributed transactions".
- MIT 6.5840 (formerly 6.824) lecture notes on two-phase commit and Spanner.

---

## Questions

1. Why can't each shard simply commit its part of a cross-shard transaction when the application says "commit"?
2. Walk through 2PC for a two-shard transaction where both participants vote YES. What gets fsynced, by whom, and when?
3. What exactly is the commit point in 2PC, and why there?
4. A participant has voted YES and then loses contact with the coordinator. Why can't it commit or abort on its own?
5. Explain why an in-doubt transaction can block unrelated transactions on the same database.
6. The coordinator crashes after logging COMMIT but before sending any COMMIT messages. What happens when it restarts?
7. The coordinator's disk is destroyed while transactions are in doubt. What are the options, and what's the risk?
8. What does 3PC add, and why does it fail to guarantee atomicity in real networks?
9. Why do many engineers avoid XA transactions across, e.g., a database and a message broker? What do they do instead?
10. Atomic commit and consensus both involve nodes agreeing. Why can consensus proceed with a majority, while
    2PC needs all participants?
11. How does running each 2PC participant and the coordinator as a Paxos/Raft group remove 2PC's blocking problem?
    What failure would still block it?
12. In Spanner, why are single-shard transactions much cheaper than multi-shard ones?
13. Compare 2PC and 2PL: what problem does each solve?
14. With 5 participants each 99.9% available, roughly how available is the 2PC path, and why does this matter?
15. Why does Postgres ship with `max_prepared_transactions = 0`?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Each shard decides independently. One might commit while another fails (constraint violation, crash,
lost message), leaving a partial result. Once committed, a shard can't undo its change. So no shard should
commit until all are known to be able to commit; that needs a voting round first.

**2.** The coordinator sends PREPARE. Each participant checks it can commit, writes its changes plus a "prepared"
record to its WAL and fsyncs, then replies YES (keeping locks). The coordinator, having all YESes, writes and
fsyncs COMMIT to its log (commit point), then sends COMMIT. Each participant commits (writes and typically
fsyncs a commit record), releases locks and acks. The coordinator then can record completion and forget the
transaction.

**3.** The coordinator's durable write of the COMMIT decision. Before it, the coordinator can still decide abort;
after it, the decision survives coordinator crashes and will be delivered to participants no matter what. It's
the single place the global outcome is decided.

**4.** It promised to commit if told, so aborting might contradict a COMMIT the coordinator already logged and sent
to others. But it doesn't know all votes were YES, so committing might contradict an ABORT. Without the
decision (from the coordinator or a participant that received it), any unilateral choice risks breaking atomicity.

**5.** The in-doubt transaction keeps its locks (and its row versions), because it might still commit. Any other
transaction that needs those rows waits behind the locks, and anything waiting on those transactions waits too.
In Postgres, it also holds back VACUUM.

**6.** On recovery it reads its log, finds COMMIT for T and resends COMMIT to all participants (retrying until
acked). The in-doubt participants then commit and release locks. Nothing is lost; there was just a delay.

**7.** The in-doubt participants can't learn the outcome. An administrator must make a heuristic decision: force
commit or force rollback each prepared transaction (possibly after checking other participants: if any
committed, commit all; if any aborted or never prepared, abort). If no participant knows, a guess can violate
atomicity, which then needs manual reconciliation.

**8.** A PreCommit phase between voting and committing, so that remaining participants can infer a safe outcome
from each other's states after a coordinator failure, using timeouts. It assumes bounded delays and accurate
failure detection. Under a partition, a timeout doesn't mean a crash; the two sides can reach different
decisions (one commits, one aborts), breaking atomicity.

**9.** The coordinator typically lives in the app server, so its failure strands in-doubt transactions holding
locks; XA can't detect cross-system deadlocks or provide cross-system SSI; it's slow; it couples system
availability; and many systems (Kafka, most NoSQL/cloud services) don't support it. Alternatives: the
transactional outbox plus idempotent consumers, change data capture, and sagas.

**10.** Consensus only needs to pick some proposed value, and any majority can do that while preserving safety.
Atomic commit gives every participant a veto: a participant that can't commit (constraint failure, crash) must
force abort, so you need a vote from every participant. A majority deciding "commit" when one member can't
would be incorrect.

**11.** The prepared state and the decision are written to replicated logs. If the machine acting as coordinator
or participant leader dies, a new leader is elected in seconds and has the full log, including the decision,
so it continues the protocol. It still blocks if a group loses its majority (e.g. 2 of 3 replicas down, or a
partition isolating the majority), since that group can't make progress.

**12.** A single-shard transaction is committed with one consensus write in that group: no prepare round, no
cross-group messaging, no coordinator log. Multi-shard ones add 2PC rounds, each a consensus write, plus
locks held longer. Good partitioning keeps related data together.

**13.** 2PC: atomicity across multiple nodes (all commit or all abort). 2PL: isolation (serializability) by locking
on a node: acquire locks during the growing phase, release in the shrinking phase. A distributed database can use
both.

**14.** About 0.999⁵ ≈ 99.5%. Every participant must be up and reachable for the transaction to commit (no
quorum), so availability decreases with each system you enlist, and one slow participant slows all of them.

**15.** Prepared transactions are durable and survive restarts, holding locks and preventing VACUUM; if a
transaction manager forgets them, they cause serious damage silently. Disabling by default ensures only
deployments that actually run a transaction manager enable it deliberately.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** The atomic commit problem?
**A:** Make all nodes involved in a transaction either all commit or all abort, despite failures.

**Q:** 2PC roles?
**A:** Coordinator (transaction manager) and participants (resource managers).

**Q:** Phase 1 of 2PC?
**A:** Prepare: participants durably log changes and a prepared record, then vote yes/no.

**Q:** Phase 2 of 2PC?
**A:** Coordinator logs the decision, sends commit/abort; retries until all ack.

**Q:** What does a YES vote promise?
**A:** The participant will commit if told, even after crash, and can't abort unilaterally.

**Q:** The commit point in 2PC?
**A:** The coordinator's durable log write of the COMMIT decision.

**Q:** What is an in-doubt participant?
**A:** One that voted YES and hasn't received the decision; must wait, holding locks.

**Q:** Why is 2PC called blocking?
**A:** Coordinator failure after votes leaves participants stuck until it recovers.

**Q:** What is a heuristic decision?
**A:** An administrator manually committing or rolling back an in-doubt transaction, risking atomicity.

**Q:** Coordinator crashes before logging decision → outcome?
**A:** Abort.

**Q:** Coordinator crashes after logging COMMIT → outcome?
**A:** On restart, it resends COMMIT; participants commit.

**Q:** 2PC availability compared to quorum protocols?
**A:** Needs every participant; quorums need only a majority.

**Q:** 3PC's extra phase?
**A:** PreCommit, between voting and committing.

**Q:** Why isn't 3PC used?
**A:** Assumes bounded delays/perfect failure detection; partitions can break atomicity; extra round trip.

**Q:** What is XA?
**A:** A standard interface for 2PC across heterogeneous resource managers (DBs, brokers), e.g. via JTA.

**Q:** Main practical XA problems?
**A:** Coordinator in app server, stuck locks, no cross-system deadlock detection or SSI, slow, limited support.

**Q:** Modern alternatives to XA?
**A:** Transactional outbox, idempotent consumers, CDC, sagas.

**Q:** How does Spanner make 2PC non-blocking?
**A:** Coordinator and participants are Paxos groups; decisions are replicated; leader failover continues the protocol.

**Q:** Paxos Commit?
**A:** Gray & Lamport's use of consensus to make the commit decision fault-tolerant.

**Q:** When does consensus-replicated 2PC still block?
**A:** When a group loses its majority.

**Q:** Why are single-shard transactions cheap in Spanner?
**A:** One Paxos write; no 2PC.

**Q:** Why can't atomic commit use majority voting?
**A:** Every participant has a veto; a single "no" must force abort.

**Q:** 2PC vs 2PL?
**A:** 2PC: atomic commit across nodes. 2PL: lock-based isolation.

**Q:** Postgres command for a 2PC participant's prepare?
**A:** `PREPARE TRANSACTION 'id'`, then `COMMIT PREPARED` / `ROLLBACK PREPARED`.
