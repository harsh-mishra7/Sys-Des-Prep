# 23 — Single-leader replication

> Part 5 — Scaling Data · Prerequisites: topics 01, 17, 21

A single database server is a single point of failure and has a ceiling: one machine's disks, one machine's CPU,
one location on the planet. The first and most common way out is **replication**: keeping a copy of the same data
on several machines. The idea sounds trivial ("just copy the data"), but data keeps *changing*, and the hard part
of replication is entirely about propagating changes: how quickly, in what order, what happens when a copy falls
behind, and what happens when the machine that accepts writes dies. Single-leader replication is the simplest
answer to those questions, used by default in PostgreSQL, MySQL, MongoDB replica sets, Redis, and many others.
Understanding it well also gives you the vocabulary for everything else in this Part: multi-leader and leaderless
replication (topic 24), partitioning (topic 25), and the consensus machinery of Part 6.

By the end of this topic you should be able to:
1. Explain the three reasons to replicate and which ones single-leader replication actually helps with.
2. Describe synchronous, asynchronous and semi-synchronous replication, and what each one risks.
3. Explain how a new follower is brought up to date without stopping the leader.
4. Compare statement-based replication, WAL shipping, and logical (row-based) replication.
5. Walk through a failover step by step and explain how it goes wrong, including split brain.
6. Recognise read-your-writes, monotonic-reads and consistent-prefix anomalies caused by replication lag, and
   know the standard fixes for each.

---

## 1. Why replicate?

Replication means keeping copies of the same data on multiple machines (called **replicas**) connected by a
network. There are three distinct reasons to do it, and it is worth keeping them separate because different
replication setups serve them differently.

| Goal | What replication gives you | Example |
|---|---|---|
| **Availability / durability** | If one machine dies, another has the data and can keep serving | A disk fails at 3am; a standby takes over |
| **Read scaling** | Spread read queries over many copies | A product catalogue read 1000x more than it is written |
| **Latency** | Put a copy geographically near users | A read replica in Europe for European users |

Note what is *not* on the list: **write scaling**. In single-leader replication every write still goes through one
machine. Every replica must eventually apply every write, so adding replicas adds read capacity but no write
capacity. Scaling writes needs partitioning (topic 25) or a multi-leader setup (topic 24).

A further subtlety: replication is about copying data that **fits on one machine**. If the dataset is too large for
one machine, you need partitioning; in practice the two are combined (each partition is replicated), which is
covered in topic 25.

---

## 2. Leaders and followers

### 2.1 The basic mechanism

One replica is designated the **leader** (also called primary or master). The others are **followers** (also called
replicas, secondaries, standbys, or read replicas).

```
                 writes                     reads
   clients ─────────────────┐        ┌─────────────── clients
                            v        v
                       ┌──────────────┐
                       │    LEADER    │  (accepts writes, also can serve reads)
                       └──────┬───────┘
                              │ replication log / change stream
              ┌───────────────┼───────────────┐
              v               v               v
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │ FOLLOWER │    │ FOLLOWER │    │ FOLLOWER │   (read-only; apply changes in log order)
        └──────────┘    └──────────┘    └──────────┘
              ^               ^               ^
              └──── reads ────┴──── reads ────┘
```

1. Clients send all **writes** to the leader. The leader applies the write to its local storage.
2. The leader also appends the change to a **replication log** (or change stream) and sends it to every follower.
3. Each follower applies the changes **in the same order** the leader did. Because they start from the same state
   and apply the same changes in the same order, they end up in the same state.
4. **Reads** can go to the leader or to any follower. Followers reject writes from clients.

The crucial property is the single ordering. Because one machine decides the order of all writes, there are no
write conflicts to resolve: two concurrent updates to the same row are serialized by the leader's own concurrency
control (topic 22), and the followers simply replay the outcome. This is why single-leader is the easiest model to
reason about, and why the other models in topic 24 have to deal with conflict resolution.

### 2.2 Synchronous vs asynchronous replication

The key question: when the leader receives a write, does it wait for followers before telling the client "OK"?

```
Synchronous follower F1, asynchronous follower F2

client        leader            F1 (sync)           F2 (async)
  │  write x=1  │                  │                    │
  │────────────>│ apply locally    │                    │
  │             │──── x=1 ────────>│ apply              │
  │             │──── x=1 ─────────┼───────────────────>│ (arrives whenever)
  │             │<──── ack ────────│                    │
  │<──── OK ────│  (waits for F1, does not wait for F2) │
```

**Synchronous:** the leader waits until the follower confirms it has received (and usually durably written) the
change before acknowledging the client.
- Good: the follower is guaranteed to have an up-to-date copy. If the leader dies immediately after the OK, the
  write is not lost.
- Bad: every write's latency now includes a network round-trip plus the follower's disk write. Worse, if the
  synchronous follower is down or slow, **writes block**. One sick follower can stall the whole database.

**Asynchronous:** the leader sends the change and acknowledges the client immediately, without waiting.
- Good: write latency is unaffected by followers; the leader keeps accepting writes even if every follower is down.
- Bad: if the leader fails before a change reaches any follower, that **acknowledged write is lost**. The client was
  told "OK" and the data is gone. Durability is weakened in exchange for availability and latency.

**Semi-synchronous:** making *all* followers synchronous is impractical (any one failure halts writes), so the common
compromise is **one** synchronous follower and the rest asynchronous. If the synchronous follower becomes
unavailable or slow, one of the asynchronous followers is promoted to synchronous. This guarantees that at least two
nodes (the leader and one follower) have every acknowledged write. MySQL's "semi-sync replication" and PostgreSQL's
`synchronous_standby_names` with `ANY 1 (...)` are versions of this idea.

| Mode | Write latency | Acknowledged write can be lost? | Leader blocks if follower fails? |
|---|---|---|---|
| Fully synchronous | High (slowest follower) | No | Yes |
| Semi-synchronous (1 sync) | Medium | Only if leader *and* sync follower both lose it | No (another follower is promoted to sync) |
| Asynchronous | Low | Yes, the tail of recent writes | No |

In practice, many production setups run fully asynchronous replication, especially across regions, accepting the
small window of possible data loss. That is a real choice with a real cost, and it is the root of several
problems later in this topic (failover data loss, replication lag anomalies).

A note on terminology: "synchronous replication" here is about *acknowledgement*, not about making followers
linearizable (topic 28). Even with a synchronous follower, a read from that follower can be stale if it has received
the change but not yet applied it, depending on the database's configuration (Postgres distinguishes
`remote_write`, `on`, and `remote_apply` for exactly this reason).

### 2.3 Adding a new follower

You need new followers regularly: to replace a failed machine, to add read capacity, to open a new region. The
naive approach, copying the leader's data files, does not work while the leader is accepting writes, because the
files change during the copy and you get an inconsistent mix. Locking the database for the copy works but means
downtime. The standard procedure avoids both:

```
1. Take a consistent SNAPSHOT of the leader, without locking it
   (e.g. pg_basebackup, or a snapshot at a known transaction boundary).
   The snapshot is associated with an exact position in the replication log:
        Postgres: a log sequence number (LSN)
        MySQL:    binlog file + position, or a GTID set

2. Copy the snapshot to the new follower.

3. The follower connects to the leader and requests every change SINCE that position.

4. Once it has processed the backlog it has "caught up", and from then on it
   streams changes like any other follower.

  leader log:  ... [100] [101] [102] [103] [104] [105] [106] ...
                           ^
                   snapshot taken here (position 101)
  new follower: restore snapshot (state as of 101), then replay 102, 103, ... live
```

The key insight is that the snapshot is tied to a precise log position. Without that link you cannot know which
changes the snapshot already contains and which still need replaying. Two practical consequences:

- The leader must **retain** its log from the snapshot position until the follower has consumed it. If the log is
  recycled first (e.g. Postgres WAL segments removed), the follower can never catch up and must start over.
  Replication slots in Postgres exist to pin the log for a specific follower; the flip side is that a dead follower
  holding a slot can make the leader's disk fill up.
- The same mechanism handles **follower recovery**: a follower that crashes or is disconnected remembers the last
  log position it applied, reconnects, and asks for everything after it ("catch-up recovery").

---

## 3. How the replication log is implemented

"The leader sends changes to followers" hides a design question: *what exactly* is a change? There are three main
answers, and they differ in correctness, coupling to storage internals, and usefulness outside the database.

### 3.1 Statement-based replication

The leader logs every write **statement** it executes (`INSERT`, `UPDATE`, `DELETE`) and sends the SQL text to the
followers, which execute it as if it came from a client.

It is compact and easy to understand, but it breaks whenever the statement is not deterministic:

- **Nondeterministic functions:** `UPDATE orders SET created_at = NOW()` or `INSERT ... VALUES (RAND())` produce a
  different value on each replica.
- **Autoincrement and data-dependent statements:** `UPDATE accounts SET tier='gold' WHERE balance > 1000` depends on
  the existing data and on execution order. If concurrent transactions are applied in a slightly different order on a
  follower, the result diverges. This forces followers to execute statements in exactly the leader's order,
  limiting parallelism.
- **Side effects:** triggers, stored procedures, user-defined functions may behave differently per replica.

The leader can work around some of this (for instance by replacing `NOW()` with a literal timestamp before logging),
but there are many edge cases. MySQL used statement-based replication before version 5.1 and now defaults to
row-based, switching to row-based automatically for unsafe statements in "mixed" mode.

### 3.2 Write-ahead log (WAL) shipping

Recall from topic 17 that storage engines write every change to an append-only **write-ahead log** before modifying
their data structures, so they can recover after a crash. That log already describes every change in exact order,
so the leader can simply send it to followers. A follower that replays the WAL builds the exact same on-disk data
structures as the leader. PostgreSQL physical (streaming) replication works this way.

Advantages: the log already exists (no extra work), it is fully deterministic, and replicas are byte-for-byte
identical, which is great for failover.

The disadvantage is **coupling to the storage engine**. A WAL record says something like "in disk block 4711,
change bytes at offset 312 to these values". That is meaningful only to the exact same storage engine at the exact
same version. Consequences:

- Leader and followers usually must run the **same database version**. You cannot upgrade a follower to a new
  version first, fail over to it, then upgrade the old leader: a classic way to get zero-downtime upgrades that WAL
  shipping often rules out. Major upgrades then require downtime or a different mechanism.
- The log is useless to anything that is not that database (you cannot feed it into a search index).
- Everything is replicated: all databases in the cluster, every index update, every vacuum. You cannot replicate one
  table.

### 3.3 Logical (row-based) replication

A **logical log** describes changes at the level of rows, independently of the storage engine's physical layout:

- Insert: the new values of all columns.
- Delete: enough to identify the row (usually the primary key, or all old values if there is none).
- Update: the identifier of the row plus the new values of the changed (or all) columns.
- Transaction boundaries: which row changes belong together, and a commit record.

```
Statement-based:  UPDATE products SET price = price * 1.1 WHERE category = 'books';
Logical/row-based: txn 9812 BEGIN
                   UPDATE products id=17  price: 20.00 -> 22.00
                   UPDATE products id=42  price: 15.00 -> 16.50
                   UPDATE products id=88  price:  9.00 ->  9.90
                   txn 9812 COMMIT
WAL (physical):   page 0x3A1F offset 0x0138: write 8 bytes ...; btree page 0x77C2: insert tuple ...
```

MySQL's row-based binlog and PostgreSQL's logical replication / logical decoding are examples.

Advantages:
- **Decoupled from storage internals**: leader and follower can run different versions or even different storage
  engines, enabling rolling upgrades.
- **Deterministic**: actual resulting values are logged, so `NOW()` or data-dependent updates are not a problem.
- **Usable by external systems**: the logical log is easy to parse, so you can stream it into a data warehouse,
  cache, or search index. This is **change data capture** (CDC), covered in topic 39.
- **Selective**: you can replicate a subset of tables.

Disadvantages: logs can be larger (one statement updating a million rows produces a million row events), and some
things are not captured (in Postgres logical replication, DDL schema changes and sequences are not replicated
automatically).

| | Statement-based | WAL shipping (physical) | Logical (row-based) |
|---|---|---|---|
| Unit of change | SQL statement | Bytes in disk blocks | Row values |
| Deterministic | No | Yes | Yes |
| Version / engine independent | Mostly | No | Yes |
| Size | Small | Large (includes index/page writes) | Medium–large |
| Feeds external systems (CDC) | Awkwardly | No | Yes |
| Example | Old MySQL | Postgres streaming replication | MySQL row binlog, Postgres logical replication |

A fourth, less common option is **trigger-based** replication: application-defined triggers write changes to a
separate table that an external process reads (older tools like Slony and Bucardo). It is flexible but slow and
error-prone, and has mostly been replaced by logical decoding.

---

## 4. Handling node outages

### 4.1 Follower failure: catch-up recovery

Easy, as described in 2.3: the follower knows the last log position it processed, reconnects, and requests the
rest. The only risk is that the leader has discarded the needed log in the meantime, in which case the follower
must be rebuilt from a fresh snapshot.

### 4.2 Leader failure: failover

When the leader fails, one of the followers must be promoted, clients must send writes to the new leader, and the
other followers must start consuming from it. This is **failover**. It can be manual (an operator runs a command) or
automatic. An automatic failover typically has three steps:

```
1. DETECT that the leader has failed
   - No reliable way exists; systems use timeouts on heartbeats (e.g. "no response for 30s").
   - (topic 26: you cannot distinguish "dead" from "slow" or "network partitioned")

2. CHOOSE a new leader
   - Usually the follower with the most up-to-date data (smallest data loss).
   - Either an election among replicas or a decision by a previously elected
     controller / coordination service (topics 30, 31).

3. RECONFIGURE the system
   - Clients (or a proxy / DNS / service discovery) send writes to the new leader.
   - Other followers re-point replication at the new leader.
   - If the old leader comes back, it must be told it is no longer leader and become a follower.
```

Tools that automate this: Patroni (Postgres, using etcd/ZooKeeper/Consul), Orchestrator (MySQL), Redis Sentinel,
and MongoDB's built-in replica set elections.

### 4.3 What goes wrong during failover

Failover is one of the most fragile operations in a database's life. The problems are fundamental, not
implementation bugs.

**Problem 1: Lost writes with asynchronous replication.** The new leader may not have received the last writes the
old leader accepted.

```
time ─────────────────────────────────────────────────────────────>
old leader:   w1  w2  w3  w4  [CRASH]
follower B:   w1  w2  w3            (w4 never arrived)
                                   B promoted → new leader
                                   new writes: w5 w6
old leader returns: has w4, which conflicts with the new history.
Common resolution: discard w4.  The client that got "OK" for w4 has lost its write.
```

Discarding the old leader's unreplicated writes is the usual choice, and it violates the durability the client was
promised. Asynchronous replication + failover = possible loss of acknowledged writes.

**Problem 2: Discarded writes interacting with other systems.** A well-known GitHub incident (2012): an out-of-date
MySQL follower was promoted. It used an autoincrement counter to assign primary keys to new rows; since it was
behind, it reissued primary keys the old leader had already handed out. Those keys were also used in a Redis store,
so some private data was shown to the wrong users. Lesson: data lost from the database does not vanish from caches,
search indexes, or downstream services that already saw it.

**Problem 3: Split brain.** Two nodes both believe they are the leader and both accept writes. With no process for
resolving conflicts (single-leader systems assume there is only one writer), data diverges or gets corrupted.

```
            network partition
                   ║
   ┌────────────┐  ║  ┌────────────┐
   │ old LEADER │  ║  │ follower B │  "leader is dead" → promotes itself
   │ still alive│  ║  │ new LEADER │
   └─────▲──────┘  ║  └─────▲──────┘
         │ writes  ║        │ writes
    client group 1 ║   client group 2
                   ║
 Both accept writes to the same rows. When the partition heals there are two
 incompatible histories, and nothing in the system knows how to merge them.
```

How split brain happens: the old leader is not dead; it is merely slow (a long GC pause, an overloaded disk) or cut
off from the nodes that decided to fail over, but still reachable by some clients.

Defences:
- **Quorum-based leader election.** A node can become leader only with votes from a majority of nodes. Two disjoint
  majorities cannot exist, so two leaders cannot be *elected* in the same term (topic 30).
- **Leader leases.** The leader holds a time-bounded lease and stops accepting writes when it cannot renew it; a new
  leader is only elected after the old lease must have expired (topic 31). This depends on bounded clock drift.
- **Fencing.** Ensure a deposed leader cannot do damage even if it still thinks it is the leader. Either physically
  ("STONITH": shoot the other node in the head, power it off via a management interface) or logically, with
  **fencing tokens / epoch numbers**: every leadership term gets a higher number, and storage or downstream systems
  reject requests carrying an old number (topic 31).

Note the subtle point: preventing two nodes from *being elected* is not enough. The old leader may not know it has
been replaced, so the system also needs the old leader to *stop acting* (lease expiry) or others to *ignore it*
(fencing).

**Problem 4: Choosing the timeout.** A long timeout means a long outage before recovery. A short timeout means
unnecessary failovers: a load spike slows the leader, it misses heartbeats, a failover starts, which adds more load
(the new leader is cold, clients reconnect), which makes things worse. An unnecessary failover is not free: it may
lose async writes and it causes a burst of errors. Many operators prefer manual failover for exactly these reasons,
or set conservative timeouts.

| Failover risk | Cause | Mitigation |
|---|---|---|
| Lost acknowledged writes | Async replication; new leader was behind | Semi-sync; promote most up-to-date follower |
| Inconsistent external systems | Reused IDs, caches saw discarded data | Don't derive IDs from replica-local counters; reconcile downstream |
| Split brain | Old leader still alive and reachable | Majority election, leases, fencing tokens, STONITH |
| Flapping failovers | Timeout too short under load | Conservative timeouts, manual approval, hysteresis |

---

## 5. Replication lag and its anomalies

### 5.1 Read scaling means reading stale data

The read-scaling promise of replication is: add followers, send reads to them. For that to scale, replication must be
asynchronous; you cannot make a write wait for 50 followers. So followers are **behind** the leader by some amount,
the **replication lag**. Normally this is a fraction of a second. Under load, during a network problem, or while a
follower replays a huge transaction, it can grow to seconds or minutes.

If you stop writing, followers eventually catch up and become consistent: this is **eventual consistency**. But
"eventually" has no upper bound, and in the meantime users can observe strange things. Three named anomalies cover
most of them; each has a corresponding guarantee and standard fixes. (These guarantees are discussed more formally
as session guarantees in topic 28.)

### 5.2 Reading your own writes

**Anomaly:** a user submits a change, then reloads the page, and the change is gone.

```
user          leader              follower (lagging)
 │ update profile │                   │
 │───────────────>│ write ok          │
 │<──── OK ───────│                   │
 │ GET profile ───┼──────────────────>│  (change not applied yet)
 │<── old profile ┼───────────────────│
 "My edit was lost!"   ...a second later the follower catches up
```

**Guarantee: read-your-writes (read-after-write) consistency.** A user always sees their own updates. It makes no
promise about other users' updates.

Techniques:
1. **Read things the user may have modified from the leader.** E.g. a user's own profile is read from the leader;
   other users' profiles from followers. Requires knowing what could have been modified without querying.
2. **Time window.** For a period after a user's last write (say one minute, based on the observed maximum lag),
   route all of that user's reads to the leader. Also: monitor lag and refuse to read from followers that are more
   than a minute behind.
3. **Track the write position.** The client remembers the log position (Postgres LSN, MySQL GTID) or timestamp of its
   last write. A read is served only by a replica that has applied at least that position; otherwise the read waits
   or goes to another replica or the leader. MySQL's `WAIT_FOR_EXECUTED_GTID_SET` and similar mechanisms support
   this.
4. **Cross-device complications.** If the user writes on a laptop and reads on a phone, a position stored in the
   laptop's cookie is not visible to the phone; it must be stored server-side. Different devices may also be routed
   to different datacentres, so "route to the leader" must be global, not per-region.

### 5.3 Monotonic reads

**Anomaly:** a user sees data, then refreshes and it has gone back in time, because the two reads went to different
followers with different lag.

```
time ──────────────────────────────────────────────>
leader:      comment C inserted
follower 1:  ......C applied (low lag)
follower 2:  .......................C applied (high lag)

user read #1 → follower 1: sees comment C
user read #2 → follower 2: comment C has disappeared
```

**Guarantee: monotonic reads.** If a user has seen data at some point in time, later reads will never see an older
state. It is weaker than strong consistency (you may still see stale data) but stronger than eventual consistency
(time never goes backwards for one user).

Techniques:
- **Sticky routing:** each user always reads from the same replica, e.g. by hashing the user ID to choose the
  replica. If that replica fails, the user must be rerouted, ideally to one at least as up to date.
- **Track the highest position seen** by the session and read only from replicas at or past it (the same mechanism
  as for read-your-writes).

### 5.4 Consistent prefix reads

**Anomaly:** an observer sees causally related writes in the wrong order: the answer before the question.

```
Mr. Poons:    "How far into the future can you see, Mrs. Cake?"   (write 1)
Mrs. Cake:    "About ten seconds usually, Mr. Poons."             (write 2, a reply to 1)

Observer reads via replicas with different lag:
   sees write 2 ... then write 1
   "About ten seconds usually." → "How far into the future can you see?"
```

(The example is from DDIA.) In a single non-partitioned database with a single replication log, followers apply
writes in leader order, so this cannot happen on one follower. It arises when data is **partitioned** (topic 25):
the question lives in one partition and the answer in another, each replicated independently with its own lag. There
is no global ordering of writes across partitions.

**Guarantee: consistent prefix reads.** If a sequence of writes happens in a certain order, anyone reading them sees
them in that order: you see some *prefix* of the history, never a version with gaps.

Techniques:
- Write causally related data to the **same partition** (e.g. all messages in one conversation share a partition
  key), so one replication log orders them.
- Track **causal dependencies** explicitly (topics 27 and 28: happens-before, causal consistency).

### 5.5 Summary of anomalies

| Anomaly | What the user sees | Guarantee | Typical fix |
|---|---|---|---|
| Lost own write | My edit vanished after reload | Read-your-writes | Read own data from leader; track last-write LSN |
| Time going backwards | Comment appears then disappears | Monotonic reads | Sticky replica per user; track highest LSN seen |
| Causality violated | Answer before question | Consistent prefix | Co-locate related writes; causal tracking |

### 5.6 Designing for lag rather than hoping

The tempting approach is to pretend replication is synchronous and assume lag is negligible. That works until a
lag spike, and then bugs appear that are rare, hard to reproduce, and hard to explain. Better:

- Ask, for each read path, "what happens if this read is 30 seconds stale?" If the answer is unacceptable, give that
  path a stronger guarantee explicitly (leader read, LSN wait).
- **Monitor replication lag** (bytes and seconds behind) and alert on it; remove badly lagging replicas from the
  read pool.
- Remember that transactions (topic 21) on the leader do not extend to followers: a transaction that reads from a
  follower sees a consistent snapshot of *the follower's* (possibly old) state.

---

## 6. How it connects

- **Topic 01 (core vocabulary):** replication is the main tool for availability and durability; this topic shows
  exactly where those promises leak.
- **Topic 17 (B-trees & the WAL):** WAL shipping reuses the crash-recovery log as a replication stream.
- **Topic 21 (transactions):** isolation applies on the leader; follower reads may observe older committed states.
- **Topic 24 (multi-leader & leaderless):** the alternatives when one write leader is not enough, and the conflicts
  they introduce.
- **Topic 25 (partitioning):** each partition typically has its own leader and followers; partitioning is why
  consistent-prefix violations happen.
- **Topic 26 (faults & partial failure):** failure detection by timeout and its fundamental ambiguity.
- **Topic 28 (consistency models):** read-your-writes and monotonic reads as session guarantees; why reading from
  async followers is not linearizable.
- **Topic 29 (CAP & PACELC):** sync vs async replication is a concrete latency-vs-consistency choice.
- **Topics 30 and 31 (consensus, leases, fencing):** the rigorous solutions to leader election and split brain. A
  Raft group is single-leader replication with safe failover built in.
- **Topic 39 (outbox & CDC):** logical replication logs as a change feed for other systems.
- **Topic 48 (multi-region & disaster recovery):** cross-region async replicas, RPO (data you can lose) and RTO
  (time to recover) are precisely the async-lag and failover-time trade-offs here.

---

## Common misconceptions

- "Adding replicas scales my database." → It scales **reads**. Every replica applies every write, so write
  throughput is bounded by one leader.
- "Replication is a backup." → Replication faithfully copies mistakes too: a `DROP TABLE` or a bad migration is
  replicated within milliseconds. Backups (point-in-time snapshots) protect against logical errors; replicas protect
  against machine failure.
- "Synchronous replication means followers are always readable with the latest data." → It means the write is
  durable on the follower before ack; whether it is *applied and visible* there depends on settings, and a read
  may still race.
- "Asynchronous replication only risks stale reads." → It also risks **losing acknowledged writes** on failover.
- "Failover is a solved, push-button operation." → It involves timeout tuning, possible data loss, split-brain risk,
  and external systems that saw discarded data. It needs consensus-based election plus fencing to be safe.
- "Replication lag is always milliseconds." → Usually, but it is unbounded: large transactions, load spikes, network
  issues or a slow follower disk can push it to minutes.
- "Eventual consistency means it is consistent most of the time, so I can ignore it." → The anomalies are rare but
  visible, confusing to users, and hard to debug. Decide per read path which guarantee is needed.
- "Statement-based replication is fine since it's just SQL." → Nondeterministic functions and data-dependent
  statements make replicas silently diverge.

## Further reading (supplements)

- Martin Kleppmann, *Designing Data-Intensive Applications*, Ch. 5 "Replication" (sections on leaders and followers,
  and problems with replication lag).
- PostgreSQL documentation: "High Availability, Load Balancing, and Replication" and "Logical Replication" chapters.
- MySQL Reference Manual: "Replication Formats" (statement-based, row-based, mixed) and "Semisynchronous Replication".
- GitHub Engineering blog, "Downtime last Saturday" (December 2012), on the MySQL failover incident.
- Douglas Terry et al., "Session Guarantees for Weakly Consistent Replicated Data" (1994): the origin of
  read-your-writes and monotonic reads.

---

## Questions

1. Why does adding followers increase read capacity but not write capacity?
2. A database uses fully synchronous replication to three followers. One follower's disk starts failing and becomes
   very slow. What happens to the system, and how does semi-synchronous replication avoid this?
3. A client receives "OK" for a write, and one second later the leader's machine catches fire. Under what replication
   setup is the write lost? Under what setup is it safe?
4. Why can't you set up a new follower simply by copying the leader's data directory while it runs? What makes the
   snapshot-plus-log approach correct?
5. A Postgres leader's disk fills up even though the data hasn't grown. A follower was decommissioned a week ago.
   What is a likely cause?
6. Give two concrete statements that would make replicas diverge under statement-based replication, and explain why.
7. You want to upgrade your database's major version with near-zero downtime by upgrading a follower first and
   failing over to it. Why might WAL shipping prevent this, and what replication method enables it?
8. Why is a logical (row-based) log useful to systems other than database replicas?
9. Walk through how split brain can arise even though the old leader never crashed. What two different kinds of
   mechanism are needed to prevent harm?
10. Your automatic failover timeout is 5 seconds. During a traffic spike the leader's latency rises and a failover
    is triggered, followed by more errors. Explain the feedback loop and the trade-off in choosing the timeout.
11. A user edits their shipping address, then the checkout page shows the old address. Which guarantee is violated,
    and give two ways to fix it.
12. A user's notification count shows 5, then 3, then 5 on successive refreshes. What is happening and how would you
    fix it?
13. Why can't consistent-prefix violations happen when reading from a single follower of an unpartitioned database,
    but can when data is partitioned?
14. Your team says "our replicas are basically never behind, so we read everything from them". What questions
    would you ask?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Every write must be applied on every replica to keep them identical, so each follower does the same write
work as the leader. Adding followers spreads reads (each read goes to only one replica) but the total write work
grows with the number of replicas; and all writes still funnel through the single leader, so write throughput is
capped by one machine.

**2.** Every write waits for acknowledgement from all synchronous followers, so write latency becomes the slow
follower's latency; if it stops responding, writes block entirely. One sick node degrades or halts the whole
database. Semi-sync requires acknowledgement from only one synchronous follower (or any one of a set); if that
follower is slow or fails, another follower takes over the synchronous role, so no single follower can stall writes.

**3.** With fully asynchronous replication the write may not have reached any follower; if a follower is promoted,
the write is gone. With synchronous or semi-synchronous replication, at least one follower durably had the write
before the client got "OK", and failover should promote that follower (the most up-to-date one), so the write
survives.

**4.** The files change while you copy them, so the copy mixes pages from different points in time and is
internally inconsistent. The snapshot approach takes a consistent snapshot tied to an exact log position (LSN /
binlog position / GTID). The follower restores it, then replays every change after that position. Because the
position precisely marks what the snapshot includes, nothing is missed or applied twice.

**5.** A replication slot (or equivalent log-retention setting) for the decommissioned follower still exists. The
leader keeps all WAL from that follower's last confirmed position, waiting for it to reconnect, so the WAL grows
without bound. Drop the slot and monitor slot lag.

**6.** `UPDATE t SET updated_at = NOW()` gives each replica a different timestamp. `INSERT ... VALUES (RAND())`
differs too. `UPDATE accounts SET tier='gold' WHERE balance > 1000` depends on existing data; if concurrent
transactions apply in a different order on a follower, a different set of rows matches. Autoincrement IDs can also
differ if inserts interleave differently. Each replica re-executes the logic instead of copying the result.

**7.** WAL records describe byte-level changes to a specific storage format; a new major version may use a different
on-disk format, so it cannot replay the old version's WAL (and versions usually must match). Logical replication
describes row-level changes independent of storage format, so a newer-version follower can consume it, enabling
upgrade-then-failover.

**8.** It describes changes as rows and values with transaction boundaries, which any program can parse. That makes
it a change data capture feed: you can keep a search index, cache, data warehouse, or analytics pipeline in sync
with the database by consuming the stream (topic 39), without dual writes from the application.

**9.** The old leader is alive but slow (GC pause, overloaded) or partitioned from the nodes that monitor it, yet
still reachable by some clients. The others time out, promote a follower, and now two nodes accept writes. You need
(a) a safe *election*: majority votes / consensus so two leaders can't be elected for the same term; and (b) a way
to stop the *old* leader acting: leases that expire so it stops by itself, and fencing tokens / epochs (or STONITH)
so that storage and other nodes reject its requests even if it doesn't know it's been deposed.

**10.** The spike slows the leader, it misses heartbeats, failover starts. Failover causes errors, client reconnects
and a cold new leader (empty caches), which increases load and latency, possibly triggering further failovers.
Each failover may also lose async writes. Short timeouts give faster recovery from real failures but more false
positives; long timeouts give fewer false positives but longer outages. Conservative timeouts or human approval are
common.

**11.** Read-your-writes. Fixes: read data the user can modify (their own profile/address) from the leader;
for a time window after a write, route that user's reads to the leader; or record the LSN of the write and only
read from a replica that has applied at least that position.

**12.** Successive requests hit different followers with different lag (monotonic reads violated): one has applied
the latest updates, another hasn't. Fix: sticky routing of each user to one replica (e.g. hash of user ID), or track
the highest log position the session has seen and only read from replicas at or past it.

**13.** A single follower applies one replication log in the leader's order, so it always reflects a prefix of the
history. With partitioning, each partition has its own log and its own lag; the question may be on a partition
that's behind while the answer is on one that's ahead, and there is no global order across partitions.

**14.** What is the maximum observed lag, and is it monitored and alerted on? What happens on each read path if a read
is stale by 30 seconds or a few minutes (large transaction, failover, network issue)? Do any flows read immediately
after writing (read-your-writes)? Do users bounce between replicas (monotonic reads)? Is data partitioned
(consistent prefix)? Are replicas with too much lag removed from the pool?

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Three reasons to replicate?
**A:** Availability/durability, read scaling, lower latency via geographic proximity.

**Q:** Does single-leader replication scale writes?
**A:** No. All writes go through one leader and every replica applies every write.

**Q:** Why are there no write conflicts in single-leader replication?
**A:** One node orders all writes; followers replay that order.

**Q:** Synchronous replication's main downside?
**A:** Higher write latency, and a slow or dead sync follower blocks writes.

**Q:** Asynchronous replication's main downside?
**A:** Acknowledged writes can be lost if the leader fails before they replicate.

**Q:** What is semi-synchronous replication?
**A:** One follower is synchronous, the rest async; if the sync one fails, another is promoted to sync.

**Q:** How do you add a follower without downtime?
**A:** Consistent snapshot tied to a log position, restore it, then replay the log from that position.

**Q:** What is catch-up recovery?
**A:** A follower reconnects and requests changes after the last log position it applied.

**Q:** Why does statement-based replication break?
**A:** Nondeterminism (NOW(), RAND()), data/order-dependent statements, triggers and side effects.

**Q:** What is WAL shipping?
**A:** Sending the storage engine's write-ahead log to followers, which replay it byte-for-byte.

**Q:** Main drawback of WAL shipping?
**A:** Coupled to the storage format: versions must usually match, no rolling major upgrades, unusable externally.

**Q:** What does a logical (row-based) log record?
**A:** Row-level changes (new values, row identifiers) plus transaction boundaries.

**Q:** Two benefits of logical replication?
**A:** Version/engine independence (rolling upgrades) and use as a CDC feed for other systems.

**Q:** Three steps of automatic failover?
**A:** Detect failure (timeout), choose a new leader (most up to date), reconfigure clients and followers.

**Q:** What is split brain?
**A:** Two nodes both believe they are leader and accept writes, so histories diverge.

**Q:** Defences against split brain?
**A:** Majority-based election, leases, fencing tokens/epochs, STONITH.

**Q:** Trade-off in failover timeout?
**A:** Short: fast recovery but false failovers under load. Long: fewer false alarms but longer outages.

**Q:** What is replication lag?
**A:** How far a follower is behind the leader; usually small, but unbounded.

**Q:** Read-your-writes guarantee?
**A:** A user always sees their own previous writes.

**Q:** Monotonic reads guarantee?
**A:** A user never sees older data after having seen newer data.

**Q:** Consistent prefix guarantee?
**A:** Writes are seen in the order they happened; no answer before its question.

**Q:** Why does consistent prefix break in partitioned databases?
**A:** Each partition replicates independently; there is no global order across partitions.

**Q:** Common fix for monotonic reads?
**A:** Route each user to the same replica, e.g. by hashing user ID.

**Q:** Common fix for read-your-writes?
**A:** Read own data from the leader, or wait for a replica to reach the write's LSN.

**Q:** Is a replica a backup?
**A:** No. It replicates mistakes like DROP TABLE immediately.
