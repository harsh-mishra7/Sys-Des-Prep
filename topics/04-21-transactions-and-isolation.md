# 21 — Transactions & isolation levels

> Part 4 — Storing Data · Prerequisites: topic 17 (WAL); topic 15 helps

Applications do things that involve several reads and writes that must make sense together: move money from
one account to another, reserve the last seat, place an order and decrement stock. Meanwhile, two things can
go wrong: the process can crash halfway, and other clients can be doing conflicting things at the same moment.
A **transaction** is the database's answer to both: a group of operations that the database promises to treat
as a unit. But "treat as a unit" turns out to hide many different promises, and the strongest version of the
concurrency promise is expensive. So databases offer weaker **isolation levels**, each of which allows certain
**anomalies**. Most production bugs in this area come from developers assuming a stronger guarantee than the
one they're actually running under. This topic gives you the precise vocabulary; topic 22 explains the
machinery that implements it.

By the end of this topic you should be able to:
1. Explain what each letter of ACID actually promises, and what it doesn't.
2. Define the isolation levels from read uncommitted to serializable.
3. Describe each anomaly (dirty read, dirty write, non-repeatable read, phantom, lost update, write skew) with a
   concrete timeline.
4. Say which isolation level prevents which anomaly.
5. Explain why "repeatable read" or "serializable" in one database doesn't mean the same as in another.
6. Look at a piece of application logic and decide what isolation (or explicit locking) it needs.

---

## 1. Why transactions exist

Consider transferring 100 from account A to B:

```sql
UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
```

Things that can go wrong without transactions:
- **Crash between the two statements.** 100 vanishes from A and never appears in B.
- **Constraint violation on the second.** B's account is closed; the second update fails. A was already debited.
- **Concurrent readers.** An auditor summing all balances between the two statements sees 100 missing.
- **Concurrent writers.** Two transfers out of A at once both read balance 150, both decide 100 is affordable,
  and A ends at -50.

Without transactions the application has to handle all of these itself, everywhere, correctly. Transactions move
that burden into the database, which can implement it once, carefully. The cost is performance and, at
higher isolation levels, the need to retry aborted transactions.

---

## 2. ACID, letter by letter

ACID (Härder and Reuter, 1983) is a memorable acronym, but the four letters are uneven, and "ACID compliant"
says less than it seems.

### 2.1 Atomicity: all or nothing, on failure

**Promise:** if a transaction fails partway (crash, error, constraint violation, explicit `ROLLBACK`), all of its
writes are undone, as if it never happened. Either all writes are applied or none are.

**What it is not:** atomicity is *not* about concurrency. "Atomic" in multithreaded programming means other threads
can't see a half-done operation; in ACID that's isolation. A better name would have been "abortability".

**How:** the WAL (topic 17). The database logs changes before applying them; on a crash, recovery uses the log
to roll back uncommitted transactions (or, in MVCC systems like Postgres, uncommitted row versions are simply
never visible). The commit record in the WAL is the single point at which the transaction becomes durable.

**Why it matters for the application:** after an abort you can safely **retry** the whole transaction, because
nothing half-done remains. (Careful: retrying is only safe if the transaction has no external side effects, like
sending an email, and if you know it actually aborted rather than committed with a lost acknowledgment; topic 38.)

### 2.2 Consistency: the application's invariants

**Promise:** the database moves from one valid state to another, where "valid" means satisfying your invariants
(e.g. "credits and debits balance", "no account below zero", "every order has a customer").

**The catch:** the database can enforce only what you declare: `NOT NULL`, `UNIQUE`, `FOREIGN KEY`, `CHECK`
constraints. Most real invariants live in application code. If your transaction's logic is wrong, the database
happily commits an inconsistent state. So consistency is a property of the **application**, achieved by using
atomicity and isolation correctly. Kleppmann notes that the C was arguably added to make the acronym work.

(Don't confuse it with "consistency" in replication or CAP (topics 28 and 29), which means something entirely
different: whether replicas agree / whether reads see the latest write.)

### 2.3 Isolation: concurrent transactions don't interfere

**Promise (ideal):** each transaction behaves as if it were the only one running. Formally, **serializability**:
the result of running transactions concurrently is the same as if they had run one after another, in *some*
serial order.

**The reality:** full serializability costs performance, so most databases default to a weaker level (Postgres,
Oracle, SQL Server: read committed; MySQL InnoDB: repeatable read). Weaker levels permit specific anomalies.
This letter is where most of the subtlety is, and most of this topic.

### 2.4 Durability: committed means it survives

**Promise:** once the database reports a commit as successful, the data won't be lost, even if the machine
crashes immediately after.

**How:** the commit record is written to the WAL and `fsync`ed to stable storage before the commit is
acknowledged (topic 17). In replicated systems, durability may also mean "written to N replicas" (topic 23).

**The fine print:** perfect durability doesn't exist. Disks lie about fsync, firmware has bugs, SSDs can corrupt
data, a whole datacenter can burn, and settings like Postgres's `synchronous_commit = off` or MySQL's
`innodb_flush_log_at_trx_commit = 2` deliberately trade a small window of loss for speed. Durability is a
risk-reduction ladder: fsync to disk, then replicate, then back up offsite.

### 2.5 Single-object vs multi-object

Even without multi-statement transactions, databases provide atomicity and isolation for a **single object**:
writing a 20 KB JSON document won't leave half of it if the power fails, and readers won't see half of it.
Single-object atomic operations like `UPDATE counters SET n = n + 1` or compare-and-set (topic 22) are very useful,
and many NoSQL stores offer only these. Multi-object transactions are needed whenever several pieces of data must
stay in sync: a foreign key relationship, a denormalized counter, a secondary index, a document plus an
"unread count".

### 2.6 BASE

NoSQL systems sometimes describe themselves as **BASE** (Basically Available, Soft state, Eventual consistency),
mostly to say "not ACID". It's a mood rather than a definition. What matters is the specific guarantees a system
gives, which this topic teaches you to ask about.

---

## 3. The anomalies

An **anomaly** (or "phenomenon") is an outcome that couldn't happen if transactions ran one at a time. Each one is
best understood as a timeline. In the timelines, T1 and T2 are concurrent transactions; time flows down.

### 3.1 Dirty write

A transaction overwrites a value that another transaction has written but **not yet committed**.

```
 T1 (buy car, buyer=Alice)            T2 (buy car, buyer=Bob)
 UPDATE listings SET buyer='Alice'
                                       UPDATE listings SET buyer='Bob'
                                       UPDATE invoices SET to='Bob'
                                       COMMIT
 UPDATE invoices SET to='Alice'
 COMMIT
 Result: listing says Bob, invoice says Alice.
```

Also: if T1 later aborts, rolling back its write would wipe out T2's committed value. Dirty writes make rollback
meaningless. **Every practical database prevents dirty writes**, even at the weakest level, using row-level write
locks held until commit.

### 3.2 Dirty read

A transaction reads data another transaction has written but **not yet committed**.

```
 T1 (transfer)                        T2 (report)
 UPDATE A: balance 500 → 400
                                       SELECT balance FROM A   → 400   (uncommitted!)
                                       SELECT balance FROM B   → 200
 UPDATE B: fails, ROLLBACK
                                       report total: 600, but the true total was 700
```

Problems: you may act on data that is later rolled back (it "never existed"), and you see partial state of
another transaction.

### 3.3 Non-repeatable read (read skew)

A transaction reads the same row twice and gets **different committed values**, because another transaction
committed in between. Seen across different rows, it's called **read skew**: the transaction sees part of the
database before another transaction and part after.

```
 Alice has 500 in each of two accounts (total 1000).
 T1 (Alice views balances)                T2 (transfer 100 from acct2 to acct1)
 SELECT bal FROM acct1  → 500
                                           UPDATE acct1 SET bal = 600
                                           UPDATE acct2 SET bal = 400
                                           COMMIT
 SELECT bal FROM acct2  → 400
 Alice sees 900. Where did 100 go?
```

Refreshing fixes it for a human. But it's serious for backups (part of the backup from before, part after),
analytic queries, and integrity checks, which take a long time and must see a consistent picture.

### 3.4 Phantom read

A transaction runs a query with a **search condition** twice, and the *set of rows matching* changes because
another transaction inserted or deleted rows.

```
 T1                                         T2
 SELECT COUNT(*) FROM bookings
   WHERE room=7 AND day='Mon'   → 0
                                            INSERT INTO bookings (room 7, Mon)
                                            COMMIT
 SELECT COUNT(*) FROM bookings
   WHERE room=7 AND day='Mon'   → 1   (a phantom row appeared)
```

The distinction from a non-repeatable read: there the *same row* changed; here *new rows* appeared. This matters
because locking existing rows can't prevent it: the phantom row didn't exist yet, so there was nothing to lock.
Phantoms become dangerous when the transaction then **writes based on** the result (see write skew).

### 3.5 Lost update

Two transactions both do **read-modify-write** on the same value; one overwrites the other's change.

```
 counter = 10
 T1                               T2
 SELECT n → 10
                                  SELECT n → 10
 UPDATE n = 11
 COMMIT
                                  UPDATE n = 11
                                  COMMIT
 Result: 11. One increment was lost (should be 12).
```

Very common in practice: incrementing counters, updating a JSON document (read, edit field, write back), two
users editing the same wiki page, adjusting stock. Note: neither transaction did a dirty read or a dirty write;
each read a committed value and wrote after the other committed. Read committed allows it.

### 3.6 Write skew

The subtlest one. Two transactions read the **same** data, each makes a decision based on it, and each then
writes to **different** rows. Neither overwrote the other, but the combined result violates an invariant that
each one checked.

The classic example: a hospital requires at least one doctor on call. Alice and Bob are both on call; both feel
sick and click "go off call" at the same moment.

```
 Invariant: at least 1 doctor on call.  Currently on call: Alice, Bob.

 T1 (Alice)                                    T2 (Bob)
 SELECT COUNT(*) FROM doctors
   WHERE on_call → 2  ("ok, someone remains")
                                                SELECT COUNT(*) FROM doctors
                                                  WHERE on_call → 2  ("ok")
 UPDATE doctors SET on_call=false
   WHERE name='Alice'
                                                UPDATE doctors SET on_call=false
                                                  WHERE name='Bob'
 COMMIT                                         COMMIT

 Result: nobody on call. Invariant broken. No row was written by both.
```

The general pattern:
1. `SELECT` checks a condition (enough doctors, room free, username not taken, balance sufficient across accounts).
2. The application decides to proceed.
3. It writes, and **the write changes the result of step 1's query** for the other transaction.

Other examples: double-booking a meeting room (both see no conflicting booking, both insert one); claiming a
username (both see it free, both insert; prevented only if there's a unique constraint); spending from a
balance spread across several rows; a multiplayer game where two players move pieces to the same square.

When the write in step 3 is an **insert** (the booking case), the conflict is via a phantom: there's no existing
row both could lock. That's what makes it hard to prevent with row locks alone.

Write skew is a generalization of lost update: in a lost update, the two writes hit the same row; in write
skew, different rows.

---

## 4. The isolation levels

The SQL standard (SQL-92) defines four levels by which anomalies (it calls them phenomena) they *forbid*.

### 4.1 Read uncommitted

Prevents dirty writes only. Dirty reads are allowed. Rarely useful; Postgres accepts the setting but treats it
as read committed.

### 4.2 Read committed

Two guarantees:
1. **No dirty reads:** you only see committed data.
2. **No dirty writes:** you only overwrite committed data.

Still allowed: non-repeatable reads / read skew, phantoms, lost updates, write skew.

Default in Postgres, Oracle and SQL Server. Implementation: write locks held until commit; for reads, either
read locks (rare, slow) or, more commonly, MVCC returning the last committed version (topic 22). In Postgres,
each **statement** gets a fresh snapshot, so two `SELECT`s in the same transaction can see different data.

### 4.3 Repeatable read and snapshot isolation

The standard's **repeatable read** forbids non-repeatable reads: rows you've read won't change under you.
Phantoms are still allowed by the standard's definition.

In practice, most databases implement this level with **snapshot isolation (SI)**: each transaction reads from a
consistent **snapshot** of the database taken when it starts (its first statement), seeing all data committed
before that point and nothing committed after, for the entire transaction. Writes go to the latest version and
are checked for conflicts.

Snapshot isolation prevents:
- Dirty reads, non-repeatable reads, read skew.
- Phantoms **for reads**: re-running a query gives the same rows, since new inserts aren't in your snapshot.
- Lost updates, **if** the implementation detects write-write conflicts (Postgres does: the second updater of a
  row gets "could not serialize access due to concurrent update" and must retry; the original SI definition
  calls this "first committer wins").

Snapshot isolation still allows **write skew** (and write skew via phantoms), because the two transactions write
different rows, so no write-write conflict exists, and each one's read was consistent with its own snapshot.

SI is very popular because readers never block writers and writers never block readers. Long-running reads
(backups, reports) get a perfectly consistent view.

### 4.4 Serializable

The strongest level: the outcome is equivalent to *some* serial execution. All anomalies above are prevented,
including write skew and phantoms.

Three main implementation strategies (details in topic 22):
1. **Actual serial execution:** run transactions one at a time on a single thread (VoltDB, Redis scripts,
   Datomic's transactor). Works when transactions are short and the data fits in memory.
2. **Two-phase locking (2PL):** readers and writers block each other, with predicate or index-range locks to
   handle phantoms. Used by MySQL InnoDB's SERIALIZABLE, SQL Server, and historically most databases.
3. **Serializable snapshot isolation (SSI):** run optimistically on snapshots, track read/write dependencies,
   abort transactions that could produce a non-serializable result. Postgres's SERIALIZABLE (since 9.1),
   CockroachDB's default.

The price: lower throughput (2PL blocks; serial execution limits to one core per partition) or aborted
transactions that must be retried (SSI, and deadlock victims under 2PL). **Applications using serializable must
be written to retry** on serialization failures (SQLSTATE `40001`).

Note: serializable promises *some* serial order, not the order in which transactions started or committed in
real time. A guarantee that also respects real-time order is **strict serializability**, which combines
serializability with linearizability (topic 28); it matters in distributed databases (Spanner provides it).

---

## 5. Which level prevents which anomaly

### 5.1 The table

`✓` = prevented, `✗` = possible.

| Anomaly | Read uncommitted | Read committed | Snapshot isolation ("repeatable read" in Postgres) | Serializable |
|---|---|---|---|---|
| Dirty write | ✓ | ✓ | ✓ | ✓ |
| Dirty read | ✗ | ✓ | ✓ | ✓ |
| Non-repeatable read / read skew | ✗ | ✗ | ✓ | ✓ |
| Phantom read (re-running a query) | ✗ | ✗ | ✓ | ✓ |
| Lost update | ✗ | ✗ | ✓ in Postgres (detects and aborts); ✗ in MySQL InnoDB RR | ✓ |
| Write skew (including via phantoms) | ✗ | ✗ | ✗ | ✓ |

The SQL standard's own table only lists dirty read, non-repeatable read and phantom, and says the standard
"repeatable read" level may allow phantoms. Lost update and write skew aren't in the standard's definitions at
all, which is part of the problem described next.

### 5.2 Choosing a level by the shape of your logic

- **Single-row read-modify-write** (counter, stock, edit a document): lost update risk. Use an atomic update
  (`SET n = n + 1`), `SELECT ... FOR UPDATE`, compare-and-set, or a level that detects lost updates (topic 22).
- **Check-then-write across rows** (doctors on call, room booking, balance spread across rows, unique-ish
  constraints the schema can't express): write skew risk. Use serializable, explicit locks on the rows (or a
  materialized "lock row") read in the check, or a database constraint (unique, exclusion constraint).
- **Read-only report or backup needing a consistent picture:** snapshot isolation / repeatable read.
- **Simple reads and independent single-statement writes:** read committed is fine.

---

## 6. Why real databases' names don't match the standard

### 6.1 The standard was defined badly

SQL-92 defined levels by forbidding three phenomena (dirty read, non-repeatable read, phantom), described
informally and with lock-based implementations in mind. In 1995, Berenson et al. ("A Critique of ANSI SQL
Isolation Levels") showed:
- The definitions are ambiguous (strict and loose interpretations exist).
- They omit dirty write, lost update, read skew and write skew.
- **Snapshot isolation doesn't fit anywhere** in the hierarchy: it prevents phantoms for reads (stronger than
  standard repeatable read in that respect) yet allows write skew (weaker than serializable). It also forbids
  all three standard phenomena, so by the standard's literal wording SI would qualify as "serializable", even
  though it isn't.

Vendors then chose names for whatever they implemented.

### 6.2 What the names actually mean

| You ask for | Postgres | MySQL (InnoDB) | Oracle | SQL Server |
|---|---|---|---|---|
| READ UNCOMMITTED | Read committed | Read uncommitted | not offered | Read uncommitted |
| READ COMMITTED | Read committed (snapshot per statement) **default** | Read committed (snapshot per statement) | Read committed (statement snapshot) **default** | Read committed, lock-based by default (snapshot-based if `READ_COMMITTED_SNAPSHOT` is on) **default** |
| REPEATABLE READ | Snapshot isolation (prevents lost updates by aborting) | **default**; snapshot for plain reads, but locking reads and `UPDATE`s act on the *latest* committed row, so lost updates and write skew can happen | not offered | Lock-based repeatable read (phantoms possible) |
| SNAPSHOT | (that's its RR) | n/a | n/a | Snapshot isolation (separate level) |
| SERIALIZABLE | True serializable via SSI | Serializable via 2PL (plain SELECTs become locking reads) | **Snapshot isolation** (write skew possible!) | Serializable via 2PL with key-range locks |

Key takeaways:
- **Oracle's "SERIALIZABLE" is snapshot isolation.** It allows write skew.
- **MySQL's default "REPEATABLE READ" is not snapshot isolation in the strict sense.** A plain `SELECT` reads the
  snapshot, but `UPDATE ... SET n = <value computed in app>` overwrites the latest version without a conflict
  check, so classic lost updates happen. A `SELECT ... FOR UPDATE` reads the latest committed data, not the
  snapshot, so in one transaction you can see two different "truths".
- **Postgres's "REPEATABLE READ"** is real SI, stronger than the standard requires (no phantoms on reads), with
  lost-update detection, but still allows write skew.
- Only "SERIALIZABLE" in Postgres, MySQL, SQL Server, CockroachDB actually prevents write skew.

Rule: **never trust the name; read your database's documentation for what it actually prevents**, and test
the anomalies that matter to your logic (projects like Jepsen and Hermitage do exactly this).

---

## 7. Worked example: one bug, four levels

A shopping site sells limited-edition sneakers: stock = 1. Two customers check out at once. The application code:

```
BEGIN;
  SELECT stock FROM products WHERE sku='S1';       -- 1
  if stock >= 1:
      UPDATE products SET stock = stock_seen - 1 WHERE sku='S1';   -- writes 0
      INSERT INTO orders (...) VALUES (...);
COMMIT;
```

- **Read committed:** both read 1, both write 0, both insert an order. Two orders, one pair of sneakers.
  (Lost update; the `stock_seen - 1` from the application makes it worse than `stock - 1`.)
- **MySQL REPEATABLE READ:** same outcome. Plain reads come from the snapshot, and the update overwrites
  without detecting the conflict.
- **Postgres REPEATABLE READ:** the second transaction's `UPDATE` hits a row modified by a concurrent committed
  transaction, so it fails with a serialization error. Retry sees stock 0 and refuses. Correct, with a retry loop.
- **Serializable (any real one):** correct; one transaction aborts and retries.
- **Fix at any level:** `UPDATE products SET stock = stock - 1 WHERE sku='S1' AND stock >= 1` and check the
  affected row count (atomic conditional update), or `SELECT ... FOR UPDATE` first. Topic 22 covers both.

Now change the invariant: "the total reserved across all variants (rows) of a limited product must not exceed
100". Each checkout reads the sum across rows and inserts a reservation row. This is write skew via phantoms:
even Postgres REPEATABLE READ allows overselling. You need SERIALIZABLE, or lock a single "product quota" row
with `FOR UPDATE` that every checkout must touch (materializing the conflict).

---

## 8. How it connects

- **Topic 17 (WAL):** the mechanism behind atomicity and durability.
- **Topic 22 (concurrency control):** how isolation levels are implemented: MVCC, 2PL, optimistic CAS, SSI,
  deadlocks.
- **Topic 23 (replication):** durability across machines; read replicas may lag, giving reads that break
  even the snapshot guarantees you have on the primary.
- **Topic 28 (consistency models):** linearizability vs serializability; strict serializability combines both.
- **Topic 34 (distributed transactions & 2PC):** atomic commit when a transaction spans multiple databases or
  partitions.
- **Topic 35 (sagas):** what to do when you can't have one ACID transaction across services.
- **Topic 38 (idempotency):** safely retrying a transaction whose outcome you don't know.

---

## Common misconceptions

- "Atomicity means other transactions can't see my intermediate state." → That's isolation. Atomicity means
  all-or-nothing on failure.
- "ACID compliant means my data is always consistent." → The database enforces only declared constraints;
  application invariants are your responsibility, and the default isolation level allows many anomalies.
- "My database is ACID, so transactions are serializable." → Most databases default to read committed or
  repeatable read, not serializable.
- "Repeatable read means the same thing everywhere." → Postgres RR is snapshot isolation with lost-update
  detection; MySQL RR allows lost updates; SQL Server RR is lock-based and allows phantoms.
- "SERIALIZABLE in Oracle prevents all anomalies." → It's snapshot isolation; write skew is possible.
- "Wrapping code in a transaction makes it safe from races." → Only if the isolation level (or explicit
  locking) prevents the specific anomaly your logic is exposed to.
- "Serializable means transactions run in the order they arrived." → It means equivalent to *some* serial order;
  real-time order is strict serializability.
- "Durability means the data can never be lost." → It means surviving a crash after an acknowledged commit, with
  correct fsync and hardware. Disks fail; replication and backups extend the guarantee.

## Further reading (supplements)

- *Designing Data-Intensive Applications*, Ch. 7 ("Transactions"), the best single treatment.
- Berenson, Bernstein, Gray, Melton, O'Neil, O'Neil, "A Critique of ANSI SQL Isolation Levels" (SIGMOD 1995).
- Adya, "Weak Consistency: A Generalized Theory and Optimistic Implementations for Distributed Transactions"
  (PhD thesis, MIT 1999): precise, implementation-independent definitions of anomalies.
- PostgreSQL documentation, "Transaction Isolation" chapter.
- MySQL documentation, "Transaction Isolation Levels" and "Consistent Nonlocking Reads".
- Martin Kleppmann's Hermitage project on GitHub: tests of isolation anomalies across databases.
- Jepsen analyses (jepsen.io), e.g. of Postgres, MySQL and distributed SQL databases.
- Peter Bailis et al., "Highly Available Transactions: Virtues and Limitations" (VLDB 2014) and Bailis's blog
  post "When is ACID ACID? Rarely." (2013), on default isolation levels.

---

## Questions

1. Why is "abortability" a better name than "atomicity" for the A in ACID? What does it let the application do?
2. Why does Kleppmann argue that consistency is a property of the application rather than the database?
3. Dirty writes are prevented even at read uncommitted. Why is preventing them so important?
4. A nightly backup job runs a long series of `SELECT`s under read committed while the site is live. What can go
   wrong with the backup, and what isolation level fixes it?
5. Explain the difference between a non-repeatable read and a phantom read. Why can't row locks prevent phantoms?
6. Two users increment a "likes" counter at the same moment using `SELECT likes` then `UPDATE ... SET likes = 43`.
   Name the anomaly and give three ways to prevent it.
7. Describe the doctors-on-call write skew. Why doesn't snapshot isolation prevent it?
8. A meeting room booking service checks for overlapping bookings, then inserts a new booking. Under Postgres
   REPEATABLE READ, can two people book the same room for the same slot? How would you fix it?
9. Your team migrates from Postgres to MySQL and keeps `REPEATABLE READ`. Code that worked now occasionally
   loses updates. Why?
10. An Oracle user says "we use SERIALIZABLE, so write skew is impossible." Respond.
11. What must an application do differently when running at serializable isolation?
12. Where does snapshot isolation fit relative to the SQL standard's four levels, and why didn't it fit cleanly?
13. A username signup flow checks `SELECT ... WHERE username = 'x'`, then inserts. What anomaly is possible,
    and what's the simplest robust fix?
14. Why is serializability not the same as "transactions execute in the order they were submitted"?
15. Is it safe to blindly retry a transaction after the client sees a network timeout on `COMMIT`? Why or why not?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Because the guarantee is about what happens when a transaction can't complete: it is aborted and all its
writes are discarded. It's not about concurrency (that's isolation). Abortability lets the application simply
retry the whole transaction after an error, knowing no partial state was left behind.

**2.** Because the invariants that define "consistent" (balances add up, someone is on call, stock isn't negative)
are mostly known only to the application. The database enforces only declared constraints. A transaction
whose logic is wrong will commit an invalid state. The database supplies atomicity and isolation as tools; the
application uses them to maintain consistency.

**3.** Without it, two transactions' writes to related rows interleave, leaving mixed results (listing says Bob,
invoice says Alice), and rollback becomes impossible to define: undoing T1's write would erase T2's committed
data or restore a value T2 overwrote. Every isolation level needs a well-defined notion of "this transaction's
writes", which requires no dirty writes.

**4.** Read committed takes a fresh snapshot per statement (or reads latest committed), so tables dumped at
different times reflect different moments: an order may be in the dump while its order_items aren't, or money
moved between accounts may appear twice or not at all (read skew). Restoring such a backup gives an inconsistent
database. Snapshot isolation (repeatable read in Postgres) gives the entire dump one consistent point-in-time view;
this is what `pg_dump` uses.

**5.** Non-repeatable read: the *same row* returns different committed values when read twice. Phantom: the *set of
rows matching a condition* changes because rows were inserted or deleted. Row locks lock rows that exist; a
phantom is a row that didn't exist when you read, so there was nothing to lock. Preventing phantoms needs locks
on the *condition* (predicate locks) or on index ranges (next-key/gap locks), or snapshot-based reads plus
serializability checks.

**6.** Lost update. Fixes: an atomic update (`UPDATE posts SET likes = likes + 1 WHERE id = ...`); pessimistic
locking (`SELECT likes ... FOR UPDATE` then update); optimistic compare-and-set with a version column; or an
isolation level that detects the conflict (Postgres REPEATABLE READ or any serializable), with a retry.

**7.** Both doctors' transactions read that two doctors are on call, each concludes it's safe to leave, and each
updates *its own* row. Snapshot isolation only detects write-write conflicts on the same row; here the write
sets are disjoint, and each transaction's reads were consistent with its own snapshot. The anomaly is that each
transaction's write invalidated the premise of the other's read, which SI doesn't track.

**8.** Yes. Both transactions query for overlaps in their snapshots, both see none, both insert a new row; no
existing row was written by both, so there's no conflict to detect. This is write skew via phantoms. Fixes:
run at SERIALIZABLE (Postgres SSI will abort one) with retries; use a Postgres exclusion constraint on
(room, time range) so the database rejects overlaps; or materialize the conflict, e.g. pre-create a row per
(room, slot) and `SELECT ... FOR UPDATE` it.

**9.** MySQL InnoDB's REPEATABLE READ serves plain reads from a snapshot but applies `UPDATE`s to the latest
committed row without checking whether it changed since the snapshot. So read-modify-write where the new
value is computed in the application silently overwrites concurrent changes. Postgres's REPEATABLE READ would
have aborted the second transaction with a serialization error. Fix with atomic updates, `FOR UPDATE`, version
columns, or SERIALIZABLE.

**10.** Oracle's SERIALIZABLE level is actually snapshot isolation. It prevents dirty reads, non-repeatable reads
and lost updates, but write skew is possible. They need explicit locking (`SELECT ... FOR UPDATE` on the rows
the decision depends on, or a materialized lock row) or constraints to protect cross-row invariants.

**11.** Expect serialization failures (and deadlock aborts) as a normal event, and retry the whole transaction,
including re-reading data, rather than surfacing an error to the user. Keep transactions short to reduce
conflicts, avoid external side effects inside them (or make them idempotent), and monitor abort rates.

**12.** SI prevents dirty reads, non-repeatable reads, and phantoms on reads, so by the standard's literal
definitions it would qualify as serializable, yet it allows write skew. And it doesn't map to standard
repeatable read either (it prevents phantoms, which that level allows). The standard defined levels via a few
informally stated phenomena, assuming lock-based implementations, and omitted lost update and write skew;
Berenson et al. pointed this out in 1995.

**13.** Two signups for the same name both see no row and both insert: write skew via a phantom (or under weaker
levels, simply a race). The simplest robust fix is a `UNIQUE` constraint on username; the second insert fails,
and the application reports "name taken". The constraint is enforced by the index regardless of isolation level.

**14.** Serializability only requires that the outcome equals *some* serial order of the committed transactions.
That order may differ from start or commit time: a transaction that committed later in real time can be
ordered "before" one that committed earlier, as long as the result is equivalent. The guarantee that also
respects real-time ordering is strict serializability.

**15.** Not blindly. The commit may have succeeded and only the acknowledgment was lost; retrying would apply the
transaction twice (two orders, two charges). The transaction must be idempotent, e.g. by including a unique
request ID with a unique constraint, or by checking whether it committed before retrying (topic 38).

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What does atomicity promise?
**A:** If a transaction fails partway, all of its writes are discarded: all or nothing.

**Q:** Is atomicity about concurrency?
**A:** No. Concurrency is isolation; atomicity is about failure (abortability).

**Q:** Who is responsible for consistency in ACID?
**A:** Mostly the application; the database enforces only declared constraints.

**Q:** What is the ideal isolation guarantee?
**A:** Serializability: the result equals some serial execution of the transactions.

**Q:** What does durability promise, and how?
**A:** Committed data survives crashes; via WAL fsync before acknowledging commit (plus replication/backups).

**Q:** Dirty write?
**A:** Overwriting another transaction's uncommitted write. Prevented at every level.

**Q:** Dirty read?
**A:** Reading another transaction's uncommitted write.

**Q:** Non-repeatable read / read skew?
**A:** Seeing different committed values for the same data within one transaction because another committed in between.

**Q:** Phantom read?
**A:** The set of rows matching a query changes because another transaction inserted or deleted rows.

**Q:** Lost update?
**A:** Two concurrent read-modify-write cycles; one overwrites the other's change.

**Q:** Write skew?
**A:** Two transactions read the same data, then write different rows based on it, together breaking an invariant.

**Q:** Classic write skew example?
**A:** Two doctors both go off call after each sees two on call; nobody is left.

**Q:** What does read committed prevent?
**A:** Dirty reads and dirty writes only.

**Q:** What does snapshot isolation prevent?
**A:** Dirty reads, non-repeatable reads, read phantoms, and (if detected) lost updates. Not write skew.

**Q:** Which level prevents write skew?
**A:** Serializable only.

**Q:** Default isolation in Postgres and MySQL?
**A:** Postgres: read committed. MySQL InnoDB: repeatable read.

**Q:** What is Oracle's SERIALIZABLE really?
**A:** Snapshot isolation; write skew is possible.

**Q:** What is Postgres's REPEATABLE READ really?
**A:** Snapshot isolation that aborts on concurrent updates to the same row (prevents lost updates).

**Q:** What's the gotcha in MySQL's REPEATABLE READ?
**A:** Updates and locking reads act on the latest row, not the snapshot, so lost updates can happen.

**Q:** Three ways to implement serializable?
**A:** Actual serial execution, two-phase locking, serializable snapshot isolation.

**Q:** What must apps do under serializable?
**A:** Retry transactions that fail with serialization errors (SQLSTATE 40001).

**Q:** Why don't the SQL standard levels fit real databases?
**A:** Defined informally around three phenomena with locking in mind; omit lost update and write skew; SI doesn't fit (Berenson et al., 1995).

**Q:** Serializable vs strict serializable?
**A:** Serializable: equivalent to some serial order. Strict: that order also respects real-time commit order.

**Q:** Simplest fix for "check username free, then insert"?
**A:** A UNIQUE constraint.

**Q:** Why is retrying after a commit timeout dangerous?
**A:** The commit may have succeeded; a retry duplicates it unless the transaction is idempotent.
