# 22 — Concurrency control

> Part 4 — Storing Data · Prerequisites: topics 17, 21 (19 helps for the LSM side of MVCC)

Topic 21 described *what* isolation levels promise and which anomalies they allow. This topic is about *how*
databases actually keep concurrent transactions from stepping on each other. There are two broad philosophies.
**Pessimistic** concurrency control assumes conflicts are likely and prevents them up front by locking: if you
might conflict, wait. **Optimistic** concurrency control assumes conflicts are rare: go ahead, then check at
the end whether anything conflicted, and abort if so. Most modern databases combine **multi-version concurrency
control (MVCC)**, which lets readers and writers stop blocking each other by keeping old versions of rows,
with locks for writes and, at serializable level, either strict locking or a conflict-detection layer like
**serializable snapshot isolation (SSI)**. Knowing the machinery explains everyday operational phenomena: why
Postgres tables bloat, why a forgotten open transaction can hurt a whole database, why `SELECT ... FOR UPDATE`
exists, why deadlocks happen, and why serializable transactions get aborted.

By the end of this topic you should be able to:
1. Explain how MVCC works in Postgres: row versions, `xmin`/`xmax`, snapshots, and the visibility rules.
2. Explain why MVCC creates dead rows, what vacuum does, what bloat is, and how long transactions make it worse.
3. Use row locks and `SELECT ... FOR UPDATE` correctly, and explain two-phase locking and how it prevents phantoms.
4. Implement optimistic concurrency with version columns and compare-and-set, including the retry loop.
5. Explain how SSI detects dangerous patterns and why it aborts transactions.
6. Explain how deadlocks form, how databases detect them, and how consistent lock ordering prevents them.
7. Choose between pessimistic and optimistic approaches for a given contention pattern.

---

## 1. The core problem, restated

Two transactions touch overlapping data at the same time. To produce a result that is correct under the chosen
isolation level (topic 21), the database must decide, for each read and write: can this proceed now, must it
wait, or must some transaction be aborted?

Three families of answers:

| Approach | Idea | Cost when conflicts are rare | Cost when conflicts are common |
|---|---|---|---|
| Pessimistic (locking) | Acquire locks before access; conflicting transactions wait | Lock overhead, some needless waiting | Waiting, convoys, deadlocks, but little wasted work |
| Optimistic (validate at end) | Proceed freely; check for conflicts at commit; abort if any | Very cheap | Many aborts and retries: wasted work, possible livelock |
| Multi-version (MVCC) | Keep old versions so readers see a snapshot and never wait for writers | Storage for versions, cleanup | Reads still fine; writes to the same row still need locks or validation |

MVCC is orthogonal to the other two: it's about reads. Writers still need either locks or optimistic validation
among themselves. Postgres, for example, is MVCC for reads + row locks for writes + (at serializable) SSI.

---

## 2. MVCC: multi-version concurrency control

### 2.1 The idea

Instead of overwriting a row in place, a write creates a **new version** of the row and leaves the old one in
place. Each version is tagged with which transaction created it and which (if any) deleted or replaced it. Each
transaction reads from a **snapshot**: it sees, for each row, the version that was committed as of its snapshot,
ignoring newer versions.

Consequences:
- **Readers never block writers, and writers never block readers.** A long report doesn't stop updates; an update
  doesn't stall readers waiting for its commit.
- Each transaction sees a consistent point-in-time view, which is exactly snapshot isolation (topic 21).
- Old versions must eventually be cleaned up once no snapshot can see them: garbage collection.

### 2.2 Row versions in Postgres

Every row version ("tuple") in a Postgres heap page has hidden system columns, of which two matter most:

- **`xmin`**: the ID of the transaction that created (inserted) this version.
- **`xmax`**: the ID of the transaction that deleted this version (or replaced it with a newer version, or locked
  it). 0 if none.

Transaction IDs (XIDs) are 32-bit integers assigned in increasing order to transactions that write.

An `UPDATE` in Postgres is effectively a `DELETE` of the old version plus an `INSERT` of a new one:

```
 Initial:  account A balance 500, inserted long ago by txid 90

   ctid   xmin   xmax   balance
   (0,1)   90      0      500            ← live

 txid 100 runs: UPDATE accounts SET balance = 400 WHERE id = 'A'; COMMIT;

   (0,1)   90    100      500            ← old version, "deleted" by 100
   (0,2)  100      0      400            ← new version, created by 100

 txid 105 runs: DELETE FROM accounts WHERE id = 'A'; COMMIT;

   (0,1)   90    100      500            ← dead (once nobody can see it)
   (0,2)  100    105      400            ← deleted by 105
```

The old version's header also points to the new version (`t_ctid`), forming a version chain. Indexes point to
tuples (heap locations), so a non-HOT update must also insert new entries into every index on the table
(section 2.6).

(Other engines differ: MySQL InnoDB and Oracle update the row **in place** and keep old versions in a separate
**undo log**; readers that need an old version reconstruct it by applying undo records. LSM engines like RocksDB
store each version as a separate key `key@timestamp`, topic 19. The concepts are the same; the cleanup differs.)

### 2.3 Snapshots

A Postgres snapshot records, at the moment it is taken:
- **`xmin`** (snapshot's): the lowest transaction ID still running. Everything below it has finished.
- **`xmax`** (snapshot's): one past the highest transaction ID assigned so far. Everything at or above it
  hadn't started; treat as invisible.
- **`xip` list**: the transaction IDs between those two that were **in progress** at that moment.

When is it taken?
- **Read committed:** a new snapshot at the start of **each statement**.
- **Repeatable read / serializable:** one snapshot at the first statement of the transaction, used throughout.

That one difference is the whole mechanical distinction between Postgres's read committed and repeatable read.

### 2.4 The visibility rules

For a given row version and a snapshot, the version is visible if:

1. **Its creator is visible:** the transaction `xmin` **committed**, and it committed "before the snapshot",
   meaning `xmin < snapshot.xmax` and `xmin` is not in the snapshot's in-progress list.
   (Or `xmin` is the current transaction itself: you see your own writes.)
2. **Its deleter is not visible:** `xmax` is 0, or the transaction `xmax` **aborted**, or `xmax` was still in
   progress / started after the snapshot (so from this snapshot's point of view the deletion hasn't happened).
   (If `xmax` is the current transaction, the row is deleted from your own point of view.)

Worked example, using the table above. Suppose transaction 103 took its snapshot when 100 was still running:
`snapshot = {xmin: 100, xmax: 104, xip: [100]}`.

```
 Version (0,1): xmin=90 (committed, < 100) → creator visible
                xmax=100 (in xip, not visible to this snapshot) → deletion not visible
                → VISIBLE: transaction 103 sees balance 500

 Version (0,2): xmin=100 (in xip) → creator not visible
                → NOT VISIBLE
```

Even after 100 commits, transaction 103 (at repeatable read) keeps seeing 500, because its snapshot is fixed.
A transaction that starts after 100 commits gets a snapshot where 100 is not in progress, so it sees 400.

Commit status of each XID is looked up in the commit log (`pg_xact`), and cached on the tuple via **hint bits**
so later readers don't need the lookup. (This is why the first read of freshly written rows can cause writes:
setting hint bits dirties the page.)

### 2.5 Writes under MVCC: conflicts still need locks

Two transactions updating the **same row** can't both create a new version from the same old one. When T2 tries
to update a row whose `xmax` is set by an in-progress T1, T2 **waits** for T1 to finish (this is a row lock;
Postgres stores row locks in the tuple's `xmax`, not in a separate lock table). Then:

- If T1 **aborted**: T2 proceeds on the original version.
- If T1 **committed**:
  - **Read committed:** T2 re-reads the *new* version, re-checks its `WHERE` clause against it, and applies its
    update on top ("EvalPlanQual"). This is why `SET n = n + 1` is safe under read committed, but read-then-write
    in application code is not (the app's earlier read isn't re-evaluated).
  - **Repeatable read / serializable:** T2 gets `ERROR: could not serialize access due to concurrent update` and
    must retry. That's how Postgres prevents lost updates at repeatable read (topic 21).

### 2.6 Vacuum and bloat

Dead versions don't disappear by themselves. A version is **dead** when its deleting transaction committed and no
running snapshot could still need it (its `xmax` is older than the oldest snapshot's `xmin` across the whole
database, called the **xmin horizon**).

**VACUUM** (run automatically by the **autovacuum** daemon based on how many rows changed) does:
1. Scan the table for dead tuples.
2. Remove index entries pointing to them.
3. Mark the space in heap pages as free for reuse (recorded in the free space map).
4. Update the **visibility map** (pages where all tuples are visible to everyone, enabling index-only scans,
   topic 18, and letting later vacuums skip them).
5. **Freeze** old tuples (see 2.7).

Plain VACUUM does **not** shrink the file; it makes space reusable within it. `VACUUM FULL` rewrites the table
compactly but takes an exclusive lock blocking all reads and writes for the duration (tools like `pg_repack`
do it online).

**Bloat** is when a table or index takes much more space than its live data: lots of dead tuples or reusable-but-
empty space. Causes:
- **Autovacuum can't keep up** with a high update rate (default settings are conservative for big tables).
- **A long-running transaction** (or an "idle in transaction" session someone forgot to close, a long report,
  a stale replication slot, or a standby with `hot_standby_feedback`) holds back the xmin horizon. Vacuum can't
  remove *any* version newer than that horizon, across the whole database, no matter how much it runs.

```
  time ─────────────────────────────────────────────────────────────▶
  T_long: BEGIN; SELECT ...  (left open for 6 hours, idle in transaction)
  Meanwhile: 50 million updates on hot tables → 50 million dead versions
  Vacuum runs every minute: "cannot remove: still visible to T_long's snapshot"
  Effects: tables and indexes grow; scans read more pages; caches hold dead data;
           index scans follow longer version chains; queries slow down everywhere.
```

Effects of bloat: more I/O per query, worse cache hit rates, bigger backups, slower index scans. Fixes: kill
long transactions (`idle_in_transaction_session_timeout`), tune autovacuum to be more aggressive on hot tables,
and design to avoid huge update rates on the same rows.

**HOT updates (heap-only tuples).** If an update doesn't change any indexed column and the new version fits on the
same page, Postgres stores it there and chains to it from the old one *without* adding new index entries. This
greatly reduces index bloat and write amplification. It's a reason to avoid indexing frequently updated columns
and to leave free space on pages (`fillfactor`).

### 2.7 Transaction ID wraparound

XIDs are 32-bit, so they wrap around after ~4 billion. Postgres compares XIDs modulo 2³², treating ~2 billion as
"past" and ~2 billion as "future". A row with a very old `xmin` would suddenly appear to be from the future and
become invisible: data would seem to vanish. To prevent this, vacuum **freezes** old tuples, marking them as
"committed long ago, visible to everyone". If freezing falls too far behind (often because a long transaction or
replication slot blocks it), Postgres warns, and eventually refuses new write transactions to protect the data.
This is a real outage pattern in very write-heavy Postgres systems.

### 2.8 MVCC summary

| Aspect | Postgres (versions in heap) | InnoDB / Oracle (undo log) | LSM (RocksDB etc.) |
|---|---|---|---|
| Where old versions live | Same table pages | Undo segments | Separate keys in SSTables |
| Update cost | New tuple + index entries (unless HOT) | In place + undo record | New key version |
| Cleanup | VACUUM | Purge of undo log | Compaction |
| Long transaction impact | Table/index bloat | Undo log growth, slow reads reconstructing versions | Versions retained through compaction |

---

## 3. Pessimistic locking

### 3.1 Lock modes

The basic idea: before touching data, acquire a lock; others wanting a conflicting lock wait.

- **Shared (S) lock:** for reading. Many transactions can hold S locks on the same object at once.
- **Exclusive (X) lock:** for writing. Only one holder; conflicts with S and X.

```
            Held: S     Held: X
 Want S:    ok          wait
 Want X:    wait        wait
```

Real databases have more modes (intention locks for hierarchies of table → page → row; Postgres has four
row-lock strengths and eight table-lock modes), but S/X is the essential model.

Under MVCC, plain reads don't take row locks at all. Writes always take an exclusive row lock, held until the
transaction ends. That's how dirty writes are prevented at every level (topic 21).

### 3.2 `SELECT ... FOR UPDATE`

Since MVCC reads take no locks, a read-then-write sequence is unprotected (the lost update). `FOR UPDATE` turns a
read into a locking read: it acquires exclusive row locks on the returned rows, held until commit or rollback.

```sql
BEGIN;
SELECT stock FROM products WHERE sku = 'S1' FOR UPDATE;   -- locks the row
-- app logic: if stock >= 1 ...
UPDATE products SET stock = stock - 1 WHERE sku = 'S1';
INSERT INTO orders ...;
COMMIT;                                                    -- releases the lock
```

A second checkout running the same code blocks on the `SELECT ... FOR UPDATE` until the first commits, then
reads the updated stock (in Postgres read committed, it sees the latest committed version after waiting). The
read-modify-write cycle is serialized for that row.

Variants:
- **`FOR SHARE`**: shared lock; others can read-lock too, but not update. Useful to stop a referenced row from
  being deleted while you insert a child row. (Postgres also has `FOR NO KEY UPDATE` and `FOR KEY SHARE`, used
  internally for foreign keys.)
- **`NOWAIT`**: fail immediately instead of waiting if the row is locked.
- **`SKIP LOCKED`**: skip rows that are locked. The classic use is a job queue in a table: each worker runs
  `SELECT ... WHERE status='pending' ORDER BY id LIMIT 1 FOR UPDATE SKIP LOCKED`, so workers grab different jobs
  without blocking each other.

Using `FOR UPDATE` to fix **write skew** (topic 21): lock every row your decision depends on. In the doctors
example, `SELECT * FROM doctors WHERE on_call AND shift = 1234 FOR UPDATE` locks both doctors' rows, so the second
transaction waits and then sees the first's change. But if the decision depends on the **absence** of rows (no
booking exists for this room and slot), there's nothing to lock. Options: **materialize the conflict** (create a
table of (room, slot) rows ahead of time and lock the relevant one), use a constraint, or use serializable.

Costs: lock waits add latency; hot rows become bottlenecks (every checkout of a popular product queues on one
row); holding locks across slow work (a network call to a payment provider inside the transaction) blocks
others for that whole time; and locks can deadlock (section 6).

### 3.3 Two-phase locking (2PL)

**Two-phase locking** is the classic algorithm for serializability with locks. Rules:

1. A transaction must acquire an S lock before reading an object and an X lock before writing it.
2. **Growing phase:** it may acquire locks but not release any.
3. **Shrinking phase:** once it releases any lock, it may not acquire any more.

In practice databases use **strict 2PL** (or strong strict 2PL): hold all locks until commit or abort, then
release them all at once. This avoids other transactions seeing data written by a transaction that could still
abort (cascading aborts).

```
 locks held
    ▲            ┌──────────────── commit: release all
    │         ┌──┘
    │      ┌──┘
    │   ┌──┘
    │───┘  growing phase              (strict 2PL: no shrinking until the end)
    └────────────────────────────────────────▶ time
```

Why it gives serializability: if two transactions conflict on any object, one must wait until the other has
finished with all its locks. So the order in which they pass through conflicting objects is the same for every
object; the conflict graph has no cycles, which is the definition of conflict-serializability.

Note the difference from MVCC: under 2PL, **readers block writers and writers block readers**. A long read holds
S locks on everything it read, blocking all writes to that data. This is why 2PL has worse throughput and much
worse tail latency under contention.

(Two-phase *locking* is unrelated to two-phase *commit* (topic 34) apart from the name.)

### 3.4 Preventing phantoms: predicate and index-range locks

Locking the rows you read doesn't stop someone inserting a new row that matches your query (topic 21's
phantom). Serializable 2PL needs to lock the **condition**:

- **Predicate locks:** lock "all rows matching `room_id = 7 AND time overlaps 10:00–11:00`", including rows that
  don't exist yet. An insert checks whether it matches any held predicate lock. Precise but expensive to check.
- **Index-range locks** (next-key locks, gap locks): a practical approximation. If there's an index on `room_id`,
  lock the index range for `room_id = 7` (or the gap where such entries would go). Any insert of a matching row
  must touch that index range, so it blocks. It may lock more than necessary (e.g. all bookings of room 7, not
  just the overlapping time), but that's safe. Without a suitable index, the database falls back to locking the
  whole table.

MySQL InnoDB uses next-key locks (row lock + lock on the gap before it) for locking reads and updates, even at
REPEATABLE READ. That's why InnoDB sometimes blocks inserts into "gaps" in surprising ways.

---

## 4. Optimistic concurrency

### 4.1 The idea

Don't lock. Read freely, do your work, and at write time check that what you read hasn't changed. If it has,
fail and retry. Wins when conflicts are rare; loses when they're frequent (many retries, wasted work).

### 4.2 Version columns

Add a `version` integer (or an `updated_at` timestamp, though integers are safer than clock values, topic 27)
to the row. Every update increments it and is conditional on the version you read:

```sql
-- read
SELECT id, title, body, version FROM docs WHERE id = 42;     -- version = 7

-- user edits for a while, no transaction or lock held ...

-- write: only if nobody else changed it
UPDATE docs
SET    title = 'New title', body = '...', version = version + 1
WHERE  id = 42 AND version = 7;

-- check affected row count:
--   1 → success
--   0 → someone else updated it first (version is now 8): conflict
```

On conflict, the application can re-read and retry (if the change can be reapplied automatically), merge, or show
the user "this document was changed by someone else". ORMs build this in (JPA's `@Version`, Rails'
`lock_version`, Django via libraries). HTTP exposes the same idea with `ETag` and `If-Match` (topic 14):
a `PUT` with a stale ETag gets `412 Precondition Failed`.

A key advantage: **no locks are held while the user thinks**. Holding a database lock for the minutes a human
spends editing would be impossible; the version check works across separate requests and transactions.

### 4.3 Compare-and-set (CAS)

The general primitive: "set X to new value **only if** it is currently equal to the value I expect". The version
column is one form. Others:

```sql
-- conditional update on the value itself
UPDATE products SET stock = stock - 1 WHERE sku = 'S1' AND stock >= 1;   -- check rowcount

-- CAS on a field
UPDATE wiki SET content = 'new' WHERE id = 1 AND content = 'old';
```

Many non-SQL stores offer CAS natively: DynamoDB's conditional writes (`ConditionExpression`), etcd's
transactions comparing `mod_revision`, Redis `WATCH`/`MULTI`/`EXEC` (the transaction fails if a watched key
changed), memcached `cas`. CPUs offer the same instruction for lock-free data structures.

Caveat: CAS must be atomic at the database. If the database evaluates the `WHERE` against a stale snapshot instead
of the current row, the check is meaningless. In SQL databases, `UPDATE ... WHERE` evaluates against the latest
committed row (with a row lock), so it is safe.

Also beware **ABA**: if a value goes A → B → A between your read and your CAS, a CAS on the value succeeds even
though things changed. A monotonically increasing version number avoids this.

### 4.4 The retry loop

```
 for attempt in 1..N:
     row = read()
     new = compute(row)
     if cas_write(new, expected=row.version): return ok
     sleep(backoff(attempt) + jitter)     # topic 43
 return conflict_error
```

Under heavy contention (many clients updating the same hot row), most attempts fail and retry, wasting work and
possibly never succeeding for some clients. That's when pessimistic locking (a queue of waiters) or a redesign
(sharded counters, atomic increments, batching) is better.

### 4.5 Atomic operations: the simplest option

Often you don't need either approach: let the database do the read-modify-write atomically in one statement.

```sql
UPDATE counters SET n = n + 1 WHERE id = 'likes:42';
UPDATE accounts SET balance = balance - 100 WHERE id = 'A' AND balance >= 100;
```

The database takes the row lock internally for the duration of the statement. Redis `INCR`, MongoDB `$inc`, and
DynamoDB `ADD` are the same idea. The limitation: the logic must be expressible in the statement.

### 4.6 Pessimistic vs optimistic: choosing

| Situation | Better choice |
|---|---|
| Low contention, short transactions | Optimistic (no lock overhead) |
| User edits spanning seconds to minutes | Optimistic (version column); can't hold locks |
| High contention on the same rows | Pessimistic (waiting beats repeated aborts), or redesign |
| Expensive work between read and write | Pessimistic (avoid redoing it on conflict) |
| Logic expressible as one statement | Atomic update |
| Distributed stores without locks | CAS / conditional writes |

---

## 5. Serializable Snapshot Isolation (SSI)

### 5.1 The motivation

2PL gives serializability but blocks readers and writers against each other. Snapshot isolation is fast (readers
never block) but allows write skew. **SSI** (Cahill, Röhm, Fekete, 2008) keeps SI's non-blocking reads and adds
**optimistic** detection of the specific patterns that can cause non-serializable results, aborting a
transaction when one appears. Postgres implements it as its SERIALIZABLE level (Ports and Grittner, 2012);
CockroachDB and others use related techniques.

### 5.2 What it tracks

The key concept is the **rw-antidependency** (read-write conflict) between concurrent transactions: T1 **read** a
version of some data, and concurrent T2 **wrote** a newer version of that same data that T1's snapshot didn't
see. In any equivalent serial order, T1 must come **before** T2 (T1 behaved as if T2's write hadn't happened).

```
   T1 ──rw──▶ T2      "T1 read something that T2 later overwrote; T1 must be ordered before T2"
```

Write skew is exactly a **cycle** of these. Doctors example:

```
   T_alice read Bob's on_call row (true); T_bob then wrote it      → T_alice ──rw──▶ T_bob
   T_bob read Alice's on_call row (true); T_alice then wrote it   → T_bob   ──rw──▶ T_alice

   Cycle: Alice must come before Bob AND Bob before Alice. No serial order exists.
```

### 5.3 How SSI detects it

Theory (Fekete et al., 2005) shows that every non-serializable execution under snapshot isolation contains a
**"dangerous structure"**: a transaction T_pivot with both an incoming and an outgoing rw-antidependency, among
concurrent transactions:

```
   T_in ──rw──▶ T_pivot ──rw──▶ T_out
```

(In the doctors case, each doctor transaction is a pivot, with T_in and T_out being the same other transaction.)

So SSI only needs to track rw-antidependencies and abort a transaction when a pivot appears. Mechanism in Postgres:
- **Reads take SIREAD "locks"** (predicate locks that don't block anything): records of what each serializable
  transaction read, at row, page, or relation granularity (coarsened when there are many, to save memory).
  Index-range reads lock the relevant index pages, which covers phantoms.
- **Writes check SIREAD locks:** when a transaction writes data that a concurrent transaction has read, an
  rw-antidependency is recorded.
- **When a transaction gets both an in and an out edge** with concurrent transactions (and some commit-order
  conditions hold), one transaction in the structure is aborted with
  `ERROR: could not serialize access due to read/write dependencies among transactions` (SQLSTATE `40001`).

The application retries the aborted transaction; on retry it takes a new snapshot that includes the other
transaction's commit, sees "only one doctor on call", and refuses.

### 5.4 Trade-offs

- **No blocking for reads**, like SI: predictable latency, good for read-heavy workloads.
- **False positives:** the dangerous structure is necessary but not sufficient for an anomaly, and lock
  coarsening adds more. Some aborted transactions would have been fine. Rates are usually low but rise with
  contention and long transactions.
- **Long read-write transactions** are more likely to be aborted (they accumulate dependencies). Declaring
  read-only transactions (`READ ONLY`, or `DEFERRABLE` to wait for a safe snapshot that can't be aborted) helps.
- **Retries are mandatory**: every serializable transaction must be wrapped in retry logic.
- Overhead: memory and CPU to track SIREAD locks and dependencies.

Compared with 2PL: SSI aborts where 2PL would block (and 2PL also aborts on deadlock). Compared with plain SI:
SSI costs a bit more and occasionally aborts, but removes write skew.

---

## 6. Deadlocks

### 6.1 How they form

A **deadlock** is a cycle of transactions each waiting for a lock held by the next. Classic two-way case: two
transfers in opposite directions.

```
 T1: transfer A → B                        T2: transfer B → A
 UPDATE accounts ... WHERE id='A'  (lock A)
                                           UPDATE accounts ... WHERE id='B'  (lock B)
 UPDATE accounts ... WHERE id='B'
   → waits for T2 (holds B)
                                           UPDATE accounts ... WHERE id='A'
                                             → waits for T1 (holds A)
 Neither can ever proceed.
```

Other common shapes:
- **Lock upgrade:** two transactions both hold S locks (e.g. `FOR SHARE`, or reads under 2PL) on the same row, then
  both want X. Each waits for the other's S lock.
- **Different statement orders** touching the same set of rows, e.g. one batch updates rows by ascending ID,
  another by a different index order.
- **Foreign key and gap locks** that the developer didn't realize were taken (InnoDB's gap locks are a frequent
  source).

Deadlocks are mostly a pessimistic-locking problem: they need waiting. Pure optimistic schemes don't deadlock
(they can livelock with endless retries instead).

### 6.2 Detection: the wait-for graph

The database maintains (or builds on demand) a **wait-for graph**: a node per transaction, an edge `T1 → T2` when
T1 waits for a lock T2 holds. A deadlock is a cycle.

```
   T1 ──waits for──▶ T2
    ▲                 │
    └───waits for─────┘          cycle → deadlock
```

- **Postgres:** when a lock wait exceeds `deadlock_timeout` (default 1 second), it checks the graph for a cycle
  (the check is costly, so it's only done for waits that last a while). If found, it aborts one transaction with
  `ERROR: deadlock detected` (SQLSTATE `40P01`), releasing its locks so the others proceed.
- **MySQL InnoDB:** checks immediately when a wait starts (configurable), and picks a victim, preferring the
  transaction that has modified fewer rows (cheaper to roll back).
- **Timeouts** (`lock_timeout` in Postgres, `innodb_lock_wait_timeout`) are a cruder backstop: give up after waiting
  too long, whether or not there's a cycle. Distributed databases often rely on timeouts or on schemes like
  **wait-die / wound-wait** (decide by transaction age who may wait and who aborts), because building a global
  wait-for graph across machines is hard.

The victim's transaction is rolled back; the application must **retry it**. Deadlocks are not data corruption:
the database resolves them safely; the cost is latency and wasted work.

### 6.3 Prevention: lock ordering

If every transaction acquires locks in the **same global order**, no cycle can form: a transaction holding lock
k only ever waits for locks greater than k, so the waits can't loop back.

For the transfer example: always lock the account with the smaller ID first.

```sql
BEGIN;
SELECT * FROM accounts WHERE id IN ('A', 'B') ORDER BY id FOR UPDATE;   -- locks A then B, always
UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
COMMIT;
```

T2 (B → A) also locks A first, so it waits for T1 at the very first lock instead of grabbing B. (Note: the
`ORDER BY` makes the lock order deterministic in practice; for full certainty some teams lock rows one by one in
sorted order.)

Other prevention techniques:
- **Keep transactions short** and do slow work (HTTP calls, user think time) outside them. Fewer locks held for
  less time means fewer cycles.
- **Acquire the strongest lock you'll need up front** (`FOR UPDATE` instead of reading and later upgrading) to
  avoid upgrade deadlocks.
- **Batch updates in a consistent order** (sort by primary key).
- **Use atomic single-statement updates** where possible.
- **Retry on deadlock** anyway: prevention reduces frequency, but in a complex system some deadlocks will occur.

---

## 7. Putting it together: one database, all mechanisms

What Postgres does at each isolation level, as a summary of this topic and topic 21:

| | Read committed | Repeatable read | Serializable |
|---|---|---|---|
| Snapshot taken | Per statement | Per transaction | Per transaction |
| Reads block writes? | No (MVCC) | No | No |
| Concurrent update of same row | Wait, then re-evaluate on new version | Wait; if other committed → abort (40001) | Same as repeatable read |
| Write skew / phantoms in writes | Possible | Possible | Detected by SSI → abort (40001) |
| Explicit locking available | `FOR UPDATE/SHARE`, `NOWAIT`, `SKIP LOCKED`, advisory locks | Same | Same (rarely needed) |
| Deadlocks possible | Yes | Yes | Yes |

And the toolbox for application developers, roughly in order of preference for a given problem:
1. A **constraint** (unique, foreign key, check, exclusion) if the invariant can be declared.
2. An **atomic statement** if the read-modify-write fits in one SQL statement.
3. **Optimistic CAS / version column** for low-contention or long, user-driven edits.
4. **`SELECT ... FOR UPDATE`** for high-contention or expensive critical sections, with consistent lock ordering.
5. **SERIALIZABLE + retry** for complex cross-row invariants, especially those depending on absence of rows.

---

## 8. How it connects

- **Topic 17 (WAL, buffer pool):** row versions and locks live in pages managed by the buffer pool; MVCC writes go
  through the WAL.
- **Topic 18 (indexes):** HOT updates, index bloat, visibility maps and index-only scans; index-range locks need a
  usable index.
- **Topic 19 (LSM-trees):** MVCC versions as separate keys; compaction is the garbage collector.
- **Topic 21 (isolation levels):** the guarantees these mechanisms implement.
- **Topic 23 (replication):** standbys' queries can hold back vacuum (`hot_standby_feedback`), and replication slots
  can too.
- **Topic 27 (time & clocks):** why version counters beat timestamps for optimistic checks; distributed MVCC uses
  timestamps (hybrid logical clocks, TrueTime).
- **Topic 31 (coordination, leases & locks):** distributed locks and fencing tokens extend these ideas across
  machines; fencing tokens are CAS on a monotonic version.
- **Topic 34 (distributed transactions):** 2PL combined with 2PC across nodes, and distributed deadlock handling.
- **Topic 38 (idempotency):** retries after deadlocks and serialization failures must be safe.
- **Topic 43 (retries & backoff):** backoff with jitter for optimistic retry loops.

---

## Common misconceptions

- "MVCC means no locks." → Reads take no row locks, but writes still take exclusive row locks, and conflicting
  writers wait (or abort).
- "Postgres UPDATE modifies the row in place." → It creates a new tuple version and marks the old one; the old
  one is dead space until vacuum.
- "VACUUM shrinks the table." → Plain VACUUM makes space reusable inside the file; only `VACUUM FULL` (or
  `pg_repack`) returns space to the OS.
- "An idle connection is harmless." → An idle *in transaction* session holds back the xmin horizon and blocks
  cleanup across the whole database, causing bloat.
- "Optimistic locking is always faster." → Under high contention it causes repeated aborts and wasted work;
  pessimistic locking or a redesign is better there.
- "`SELECT ... FOR UPDATE` prevents all race conditions." → It locks rows that exist; it can't lock rows that
  don't exist yet (phantoms), so "check absence, then insert" still races.
- "Deadlocks mean the database is broken." → They're a normal consequence of locking; the database detects and
  breaks them, and the application retries. Lock ordering reduces them.
- "Two-phase locking and two-phase commit are related." → Different problems entirely: serializability on one
  node vs atomic commit across nodes.
- "SSI aborts only when there really was an anomaly." → It aborts on a dangerous structure, which includes false
  positives; retries are part of the contract.

## Further reading (supplements)

- *Designing Data-Intensive Applications*, Ch. 7 ("Implementing Snapshot Isolation", "Preventing Lost Updates",
  "Write Skew and Phantoms", "Serializability").
- PostgreSQL documentation: "Concurrency Control" (MVCC, explicit locking), "Routine Vacuuming".
- Cahill, Röhm, Fekete, "Serializable Isolation for Snapshot Databases" (SIGMOD 2008).
- Ports & Grittner, "Serializable Snapshot Isolation in PostgreSQL" (VLDB 2012).
- Fekete et al., "Making Snapshot Isolation Serializable" (ACM TODS 2005).
- Bernstein, Hadzilacos, Goodman, *Concurrency Control and Recovery in Database Systems* (1987, free online).
- Hironobu Suzuki, *The Internals of PostgreSQL* (interdb.jp), chapters on concurrency control and vacuum.
- MySQL documentation: "InnoDB Locking" (record, gap and next-key locks) and "Deadlocks in InnoDB".

---

## Questions

1. Under Postgres MVCC, what physically happens when you `UPDATE` a row, and why does that create work for vacuum?
2. Transaction 103's snapshot is `{xmin: 100, xmax: 104, xip: [100, 102]}`. Is a row version with `xmin = 101`,
   `xmax = 102` visible to it, assuming 101 committed? Explain using the rules.
3. What is the mechanical difference between Postgres's read committed and repeatable read?
4. A developer leaves a `psql` session open with `BEGIN; SELECT 1;` over a weekend. On Monday the database is
   slower and much larger. Explain the chain of cause and effect, and how to prevent it.
5. Why is `UPDATE counters SET n = n + 1` safe under read committed, but `SELECT n` followed by
   `UPDATE ... SET n = <n+1 computed in the app>` is not?
6. What is a HOT update, and how would you design a table to benefit from it?
7. What does strict two-phase locking require, and why does it guarantee serializability?
8. How does a serializable 2PL database prevent the meeting-room double booking? What happens if there's no index
   on `room_id`?
9. Design an optimistic concurrency scheme for a wiki page edited by multiple users in a browser. What does the
   user experience on conflict?
10. A flash sale has 10,000 buyers hitting the same product row per second. Would you use optimistic CAS with
    retries or `SELECT ... FOR UPDATE`? Or something else? Discuss.
11. Explain how SSI detects the doctors-on-call write skew. Why must the application retry?
12. Why can SSI abort a transaction that would actually have been fine?
13. Two background jobs update the same set of rows, one ordered by `id`, one ordered by `created_at`. They
    occasionally deadlock. Why, and how do you fix it?
14. How does Postgres detect deadlocks, and why does it wait `deadlock_timeout` before checking?
15. Build a job queue on a SQL table so that multiple workers can each take a different job without blocking
    each other. Which locking feature do you use?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Postgres sets `xmax` on the existing tuple to the updating transaction's ID and inserts a new tuple with
`xmin` = that ID (and, unless it's a HOT update, new index entries pointing to it). The old tuple stays in the
page. Once the update has committed and no snapshot can still see the old version, it is dead space; vacuum must
find it, remove index entries pointing to it, and mark the space reusable.

**2.** Check the creator: `xmin = 101` is below the snapshot's `xmax` (104), not in the in-progress list, and
committed, so the creator is visible. Check the deleter: `xmax = 102` is in the in-progress list, so from this
snapshot's view the deletion hasn't happened. Therefore the version **is visible**.

**3.** When the snapshot is taken: read committed takes a new snapshot for each statement, so later statements see
newly committed data; repeatable read takes one snapshot at the transaction's first statement and uses it
throughout. Also, on a concurrent update of the same row, read committed re-evaluates against the new version and
continues, while repeatable read aborts with a serialization error.

**4.** The open transaction holds a snapshot, which pins the database's xmin horizon. Vacuum cannot remove any row
version deleted after that point, in any table. Over the weekend, all updates and deletes create dead versions
that can't be cleaned, so tables and indexes bloat; queries read more pages, caches fill with dead data, and
performance drops. Long enough, it can also block freezing and approach XID wraparound. Prevention:
`idle_in_transaction_session_timeout`, monitoring of `pg_stat_activity` for old `xact_start`, alerts on
replication slots and transaction age.

**5.** In the atomic update, the database reads and writes the row under its row lock; if a concurrent transaction
updated it first, Postgres waits, then re-evaluates `n + 1` against the newly committed version. In the two-step
version, the application's `SELECT` read an old value without a lock; the database only sees `SET n = 43`, a
constant, and has no idea it was derived from a stale read, so it overwrites the other increment.

**6.** A heap-only tuple update: when no indexed column changes and the new version fits on the same page, Postgres
puts the new version there and links it from the old one, without new index entries. Design: don't index columns
that change frequently (status counters, `last_seen_at`) unless needed, and set a lower `fillfactor` on heavily
updated tables so pages have free space for new versions.

**7.** Every read takes a shared lock, every write an exclusive lock, and all locks are held until the transaction
commits or aborts (no lock is released earlier). Any two conflicting transactions are therefore forced into an
order: the second waits for the first to finish entirely, on every object they share. The order of conflicts is
consistent across objects, so the conflict graph is acyclic and the execution is equivalent to a serial one.

**8.** The booking transaction's query (`WHERE room_id = 7 AND time overlaps ...`) takes a predicate lock or, in
practice, an index-range lock on the `room_id = 7` range of the index. The second transaction's query (shared) can
coexist, but its insert of a room-7 booking needs an exclusive lock on that range, which conflicts with the
first transaction's shared range lock, so it waits (or deadlocks, and one aborts). Without an index on `room_id`,
the database has no range to lock and must lock the whole table, serializing all bookings.

**9.** Store a `version` column. When the editor loads the page, send `version` along (or as an ETag). On save, run
`UPDATE pages SET body = ..., version = version + 1 WHERE id = ? AND version = ?` (or HTTP `If-Match`). If one row
was updated, the save succeeds. If zero, someone else saved in between; show the user "this page was changed",
offer a diff or merge, and let them resubmit against the new version. No lock is held while the user edits.

**10.** Both struggle. Optimistic CAS would have most attempts fail and retry, wasting work and adding load (retry
storms). `FOR UPDATE` would serialize everyone on one row lock: correct, but throughput is limited to one
transaction at a time on that row and latency grows with queue length. Better: an atomic conditional update
(`SET stock = stock - 1 WHERE stock > 0`, shortest possible lock hold), splitting stock across several rows
(sharded counters, buyers pick a random shard), or taking orders into a queue (topic 36) and allocating
stock asynchronously. Pessimistic or atomic beats optimistic under this much contention.

**11.** Each doctor's transaction reads both doctors' rows (SIREAD locks record this) and then writes its own row.
When Alice's transaction writes her row, the database sees Bob's concurrent transaction read it, recording
Bob → Alice (rw). Bob's write similarly records Alice → Bob. Each transaction now has both an incoming and an
outgoing rw edge: a dangerous structure. SSI aborts one with SQLSTATE 40001. The application retries; the new
snapshot shows only one doctor on call, so the retried transaction correctly refuses.

**12.** The dangerous structure (a pivot with incoming and outgoing rw-antidependencies) is necessary for an
anomaly but not sufficient: the full dependency graph might have no cycle. SSI doesn't check for actual cycles
(too expensive), only the structure. Also, SIREAD locks may be coarsened from rows to pages or whole tables to save
memory, creating dependencies that don't really exist at row level.

**13.** Each job locks rows as it updates them in its own order. Job 1 may lock row X then wait for row Y, while
job 2, iterating in a different order, has locked Y and waits for X: a cycle. Fix: make both acquire locks in the
same order (e.g. sort by `id` before updating, or `SELECT ... ORDER BY id FOR UPDATE` first), keep batches small
and transactions short, and retry on deadlock errors.

**14.** It builds the wait-for graph from the lock table and looks for a cycle involving the waiting transaction; if
found, it aborts one transaction with a deadlock error. Checking is relatively expensive, and most lock waits
resolve quickly on their own, so Postgres only runs the check for waits longer than `deadlock_timeout` (1 s by
default). The cost is that a real deadlock lasts at least that long before being broken.

**15.** Each worker runs, in a short transaction:
`SELECT id FROM jobs WHERE status = 'pending' ORDER BY id LIMIT 1 FOR UPDATE SKIP LOCKED;` then marks it
`in_progress` (or processes it) and commits. `FOR UPDATE` locks the job so no one else takes it; `SKIP LOCKED` makes
other workers skip locked rows instead of waiting, so each grabs a different job. Add a timeout/lease field to
recover jobs from crashed workers, and keep in mind that a queue table with heavy updates and deletes needs good
vacuuming (topic 36 covers dedicated queues).

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Pessimistic vs optimistic concurrency control in one line each?
**A:** Pessimistic: lock first, others wait. Optimistic: proceed, validate at the end, abort and retry on conflict.

**Q:** Core idea of MVCC?
**A:** Writes create new row versions; each transaction reads a consistent snapshot; readers and writers don't block each other.

**Q:** What are `xmin` and `xmax` on a Postgres tuple?
**A:** `xmin`: transaction that created the version. `xmax`: transaction that deleted/replaced (or locked) it.

**Q:** What does a Postgres snapshot contain?
**A:** xmin (oldest running XID), xmax (next XID to assign), and the list of in-progress XIDs.

**Q:** When is a version visible to a snapshot?
**A:** Its creator committed before the snapshot, and its deleter (if any) did not commit before the snapshot.

**Q:** Read committed vs repeatable read in Postgres, mechanically?
**A:** Snapshot per statement vs one snapshot per transaction.

**Q:** What does VACUUM do?
**A:** Removes dead tuples and their index entries, marks space reusable, updates the visibility map, freezes old tuples.

**Q:** What is bloat?
**A:** Tables or indexes much larger than their live data, due to dead versions and unreclaimed space.

**Q:** Why do long transactions cause bloat?
**A:** They hold back the xmin horizon, so vacuum can't remove dead versions anywhere in the database.

**Q:** What is a HOT update?
**A:** An update with no indexed-column change, stored on the same page without new index entries.

**Q:** Why does Postgres freeze tuples?
**A:** To avoid 32-bit transaction ID wraparound making old rows look like they're from the future.

**Q:** What does `SELECT ... FOR UPDATE` do?
**A:** Takes exclusive row locks on the selected rows until the transaction ends.

**Q:** What is `SKIP LOCKED` used for?
**A:** Letting workers skip rows locked by others, e.g. a job queue table.

**Q:** Two-phase locking rule?
**A:** Acquire locks in a growing phase, release in a shrinking phase, never acquire after releasing. Strict 2PL holds all until commit.

**Q:** How does 2PL handle phantoms?
**A:** Predicate locks or index-range (next-key/gap) locks on the query's condition.

**Q:** Version-column optimistic update?
**A:** `UPDATE ... SET version = version + 1 WHERE id = ? AND version = <read version>`; 0 rows means conflict.

**Q:** What is compare-and-set?
**A:** Write a new value only if the current value matches the expected one, atomically.

**Q:** What is the ABA problem?
**A:** A value changes A → B → A, so a value-based CAS wrongly succeeds; monotonic versions avoid it.

**Q:** When is optimistic concurrency a bad fit?
**A:** High contention on the same rows: many aborts and wasted retries.

**Q:** What does SSI add to snapshot isolation?
**A:** Tracking of read-write dependencies and aborting transactions that form a dangerous structure, preventing write skew.

**Q:** What is an rw-antidependency?
**A:** T1 read data that a concurrent T2 later wrote, so T1 must be ordered before T2.

**Q:** SSI's dangerous structure?
**A:** A pivot transaction with both an incoming and an outgoing rw-antidependency among concurrent transactions.

**Q:** SQLSTATE for serialization failures?
**A:** 40001. Retry the transaction.

**Q:** What is a deadlock?
**A:** A cycle of transactions each waiting for a lock held by the next.

**Q:** How do databases detect deadlocks?
**A:** By finding cycles in a wait-for graph (Postgres after `deadlock_timeout`), then aborting a victim.

**Q:** How does lock ordering prevent deadlocks?
**A:** If all transactions acquire locks in the same global order, waits can't form a cycle.

**Q:** Name three other deadlock-reduction techniques.
**A:** Short transactions, take the strongest lock up front (avoid upgrades), sort batch updates; and retry anyway.

**Q:** Preferred order of tools for a race condition?
**A:** Constraint, atomic statement, optimistic CAS, `FOR UPDATE` with ordering, serializable with retry.
