# 17 — B-trees & the Write-Ahead Log

> Part 4 — Storing Data · Prerequisites: topics 01, 02, 15

When you run `SELECT * FROM users WHERE id = 42` against a table of a billion rows and get an answer in a
millisecond, something clever is happening on disk. And when the power fails halfway through an
`UPDATE`, and the database comes back up with every committed transaction intact and no half-written
garbage, something else clever happened. This topic opens up the storage engine — the part of a database
that decides how bytes are laid out on disk — and builds up, step by step, the two structures at the heart
of almost every relational database: the **B-tree**, which makes lookups fast, and the **write-ahead
log**, which makes writes durable and crash-safe. Along the way you'll meet the buffer pool, the in-memory
cache that ties them together.

By the end of this topic you should be able to:
1. Explain the progression from an append-only log, to a hash index, to sorted structures, and what each step fixes.
2. Describe how a B-tree is organised in pages, why its fan-out keeps depth at 3–4 levels for billions of rows, and how page splits work.
3. Explain what a write-ahead log is, what fsync does, and walk through crash recovery.
4. Describe the buffer pool, dirty pages, checkpoints, and how they interact with the WAL.
5. Reason about failure modes: torn pages, lying disks, fsync errors, and the cost of full-page writes.

---

## 1. The simplest database: an append-only log

### 1.1 Two functions

Here is a working key-value database in two shell functions (the classic DDIA example):

```bash
db_set () { echo "$1,$2" >> database; }
db_get () { grep "^$1," database | sed -e "s/^$1,//" | tail -n 1; }
```

```
$ db_set 42 '{"name":"Asha"}'
$ db_set 7  '{"name":"Ravi"}'
$ db_set 42 '{"name":"Asha R"}'      # update = append a new version
$ db_get 42
{"name":"Asha R"}                    # tail -n 1: the last write wins
```

The file is a **log**: an append-only sequence of records.

```
database file:
  offset 0:   42,{"name":"Asha"}
  offset 19:  7,{"name":"Ravi"}
  offset 36:  42,{"name":"Asha R"}    <- newer version shadows the older one
```

### 1.2 What's good about it

- **Writes are as fast as they can be.** Appending to the end of a file is **sequential I/O** — the
  cheapest thing a disk can do (topic 02). No seeking, no finding the "right place".
- **Crash safety is simple.** A crash can only damage the last, partially written record; everything
  before it is intact. Add a checksum per record and you can detect and drop a torn tail.
- **Concurrency is simple.** One writer appends; readers can read the immutable prefix freely.

### 1.3 What's bad about it

- **Reads are O(n).** `db_get` scans the entire file. With a billion records, every lookup reads the whole
  database.
- **The file grows forever.** Old versions of a key are never removed.

To fix reads we need an **index**: an additional structure, derived from the primary data, that lets us
find records without scanning. Every index makes the same trade: **faster reads, slower writes** (the
index must be updated on every write) and more space. That trade will follow us through topics 17–19.

---

## 2. Step two: a hash index

### 2.1 Mechanism

Keep the log on disk, and keep an **in-memory hash map** from each key to the **byte offset** of its latest
record.

```
in-memory hash map           on-disk log (append-only)
+------+--------+            +------------------------------------+
| key  | offset |            | 0:  42,{"name":"Asha"}             |
+------+--------+            | 19: 7,{"name":"Ravi"}              |
| 42   | 36     |--------+   | 36: 42,{"name":"Asha R"}  <--------+
| 7    | 19     |        |   +------------------------------------+
+------+--------+        +--> one seek + one read per lookup
```

- **Write:** append the record, then update the hash map entry to the new offset.
- **Read:** look up the offset in the hash map, seek there, read one record.

This is essentially what **Bitcask** (the original storage engine in Riak) does. It's very fast for
workloads with many writes to a modest number of keys.

### 2.2 Compaction and segments

To stop the log growing forever:

1. Break the log into **segments**: close a segment file when it reaches a size limit, start a new one.
2. In the background, **compact** closed segments: rewrite them keeping only the latest value per key,
   and **merge** several segments together.
3. Switch reads to the new merged segment, then delete the old ones.

Deletes are handled by appending a special **tombstone** record; compaction discards the key's older
values when it sees the tombstone.

Crash recovery: the hash map is in memory, so it's lost on restart. Rebuild it by scanning segments, or
store a snapshot of each segment's map on disk for faster startup.

### 2.3 Limits of a hash index

- **All keys must fit in memory.** A hash map on disk performs poorly — random I/O on every probe, hard to
  grow, awkward collision handling.
- **No range queries.** "All keys between `order:1000` and `order:2000`" requires looking up each possible
  key individually; a hash scatters neighbours randomly.

Both limits come from the same root: **the data has no order**. Fix that and much more becomes possible.

---

## 3. Step three: sorted structures

### 3.1 Keep the keys sorted

If records are stored **sorted by key**:

- **Range queries** become a seek plus a sequential scan.
- **You don't need every key in memory.** Keep a **sparse index** — one key every few kilobytes. To find
  `handiwork`, find the nearest indexed keys around it (`handbag` at offset 102,400 and `handsome` at
  offset 106,000), seek to the first, and scan forward a few KB.
- **Merging** sorted files is cheap: it's the merge step of mergesort, done sequentially.

```
sparse in-memory index          sorted file on disk
handbag   -> 102400  ------->   handbag ... handcuff ... handful ... handiwork ...
handsome  -> 106000  ------->   handsome ... hangout ...
```

### 3.2 Two ways to keep data sorted on disk

There are two families of answer to "how do you keep data sorted when writes arrive in random order?":

1. **Log-structured (LSM-trees):** buffer writes in a sorted in-memory structure, flush it to a new
   immutable sorted file, and merge files in the background. Never modify a file in place. That's
   **topic 19**.
2. **Update-in-place (B-trees):** divide the file into fixed-size pages arranged as a balanced tree,
   and modify pages in place when data changes. That's the rest of this topic.

B-trees, introduced in 1970 (Bayer & McCreight), are the standard index structure in almost every
relational database (PostgreSQL, MySQL/InnoDB, Oracle, SQL Server, SQLite) and many non-relational ones.

---

## 4. B-trees

### 4.1 Pages

A B-tree breaks the database into fixed-size blocks called **pages** — typically 4 KB, 8 KB (Postgres) or
16 KB (InnoDB). A page is the unit of reading and writing: the database never reads half a page.

This matches the hardware: disks and SSDs read and write in blocks, and the OS caches files in 4 KB pages.
Reading one byte costs as much as reading the whole block, so a data structure should make every byte of
each page it reads useful.

Each page has an address (a page number, i.e. an offset in the file), and pages refer to each other by
page number — an on-disk equivalent of a pointer.

### 4.2 Structure

- The **root** page holds a sorted list of keys and, between them, references to child pages. Each child
  covers the continuous range of keys between its neighbouring keys.
- **Internal** pages do the same for narrower ranges.
- **Leaf** pages hold the actual keys with either the row itself, or a reference to where the row lives
  (topic 18 covers clustered vs secondary indexes).
- Leaves are usually linked to their siblings (a "B+ tree"), so a range scan can walk sideways across
  leaves without going back up the tree.

```
                         root page
                 [ 100 | 200 | 300 | 400 ]
                /     |      |      |     \
          <100   100-199  200-299  300-399   >=400
                        |
               internal page
         [ 210 | 230 | 250 | 270 | 290 ]
          /    |    |    |    |    \
                    |
             leaf page (keys 250..269)
   [250: row] [251: row] [253: row] ... [269: row]  <-> next leaf
```

### 4.3 Lookup

To find key 251:

1. Read the root. 251 is between 200 and 300 → follow that child reference.
2. Read the internal page. 251 is between 250 and 270 → follow that child.
3. Read the leaf. Binary-search within the page for 251.

The number of pages read equals the **depth** of the tree. Within each page the search is in memory
(binary search over a few hundred keys) and essentially free compared with I/O.

### 4.4 Fan-out and why depth stays at 3–4

The **branching factor** (fan-out) is the number of child references per page. It depends on page size and
key size, and is typically in the **hundreds**:

```
8 KB page, keys + child pointers ~16-20 bytes each  ->  ~400-500 children per page

level   pages            keys reachable (fan-out 500)
root    1                500
2       500              250,000
3       250,000          125,000,000
4       125,000,000      62,500,000,000
```

A tree with fan-out 500 and **depth 4** can address tens of billions of keys. With 4 KB pages and a
branching factor of 500, a 4-level tree can store up to 256 TB. Because a B-tree is always **balanced**
(every leaf is at the same depth), every lookup costs the same: **O(log n)** page reads, with a huge
logarithm base.

And in practice, it's even better: the root and the first internal level are tiny (1 page plus a few
hundred pages, a few MB) and are always in memory. So a lookup in a billion-row table typically costs
**one or two actual disk reads** — the leaf, and maybe the level above it.

Why not a binary tree? A balanced binary tree over a billion keys is ~30 levels deep, and each level could
be a separate random disk read. B-trees are shaped for the hardware: wide and shallow, so each expensive
I/O eliminates as many candidates as possible.

### 4.5 Insertion and page splits

To insert a key: find the leaf whose range contains it, and add the key there. If the leaf has free space,
done — one page modified.

If the leaf is **full**, it **splits**:

```
Before: insert 255 into a full leaf
   parent:  [ ... 250 | 270 ... ]
                   |
   leaf:    [250 251 253 256 258 260 262 265]   (full)

After the split:
   parent:  [ ... 250 | 258 | 270 ... ]          <- new separator key pushed up
                   |      \
   left:    [250 251 253 255 256]   right: [258 260 262 265]
```

1. Allocate a new page.
2. Move roughly half the keys into it.
3. Insert a new separator key and child reference into the **parent**.
4. If the parent is full, it splits too — and so on up the tree.
5. If the **root** splits, a new root is created above it, and the tree grows one level taller. This is the
   only way a B-tree's depth increases, which is why all leaves stay at the same depth.

Deletes are the mirror image (pages can be merged when they get too empty), though many databases merge
lazily or not at all and rely on later inserts or maintenance (e.g. `REINDEX`) to reclaim space.

### 4.6 Consequences of splits

- **One logical insert can modify several pages** (the leaf, a new leaf, the parent, possibly more). If the
  database crashes partway through, the tree could be corrupt — e.g. the new page exists but the parent
  doesn't reference it (an orphan). This is why B-trees need a write-ahead log (section 5).
- **Fragmentation.** After splits, pages are around 50–70% full on average; the logical order of leaves
  no longer matches their physical order on disk, so range scans become more random over time.
- **Insert pattern matters.** Inserting monotonically increasing keys (auto-increment IDs, timestamps)
  always hits the rightmost leaf; databases optimise this by splitting unevenly (keeping the left page
  full). Random keys (random UUIDv4 as a primary key) scatter inserts across the whole tree, touching more
  pages, causing more splits and cache misses — one reason time-ordered IDs (UUIDv7, Snowflake; topic 49)
  are popular.

### 4.7 Concurrency inside the tree

Many threads read and modify the tree at once. Databases protect pages with lightweight in-memory locks
called **latches** (distinct from transaction locks, topic 22), using protocols like "latch coupling"
(grab the child's latch, then release the parent's) so that a reader never sees a half-split page. Some
designs (e.g. Lehman–Yao B-link trees, used by Postgres) add right-sibling pointers so readers can recover
if a split happens under them.

### 4.8 Write amplification

To change a single 100-byte row, a B-tree writes at least a whole page (8–16 KB), plus a WAL record, and
possibly several pages on a split. This **write amplification** is a key cost compared with LSM-trees
(topic 19). In exchange, reads are predictable: one path from root to leaf.

---

## 5. The write-ahead log

### 5.1 The problem

Data pages are modified **in memory** first (section 6) and written to disk later. Writing every modified
page to its place on disk at every commit would be:

- **Slow:** a transaction touching 10 pages scattered across the file means 10 random writes before it can
  commit.
- **Unsafe:** if the power fails after writing 3 of the 10, the database is left with a half-applied
  transaction, or a B-tree with a split parent but no child.

### 5.2 The rule

The **write-ahead log** (WAL; called the redo log in InnoDB, journal in some systems) is an append-only file
recording every change. The rule that gives it its name:

> **Before a modified data page is written to disk, the log records describing that modification must
> already be durable on disk.** And a transaction is committed when its commit record is durable in the
> log.

```
 UPDATE accounts SET balance = balance - 100 WHERE id = 1;  COMMIT;

 1. Modify page 812 in the buffer pool (memory)           page 812 now "dirty"
 2. Append WAL record:  LSN 5001  page 812: id=1 balance 500 -> 400
 3. Append WAL record:  LSN 5002  COMMIT txn 77
 4. fsync the WAL up to LSN 5002                          <- the durability point
 5. Return "COMMIT OK" to the client
 ...
 later: write page 812 itself to the data file (checkpoint / background writer)
```

Each WAL record gets a **log sequence number (LSN)**, a monotonically increasing position in the log. Each
data page remembers the LSN of the last change applied to it (the "page LSN"), which lets recovery know
whether a given change is already reflected in the page.

### 5.3 Why this is fast

- The WAL is written **sequentially** — the cheapest I/O there is.
- Commit only needs the small WAL records flushed, not the scattered data pages.
- Many transactions' commits can share one fsync (**group commit**): if 50 transactions commit within the
  same millisecond, the database flushes the log once for all of them.
- Data pages are written lazily, so a hot page modified 1,000 times between flushes is written once.

### 5.4 fsync: what "durable" actually means

When a program calls `write()`, the data goes into the **OS page cache** (memory). The OS writes it to the
device later. If the machine loses power before that, the data is gone even though `write()` returned
success.

**`fsync(fd)`** asks the OS to push the file's data (and the needed metadata) all the way to the storage
device, and the device to persist it, returning only when that's done.

```
app write()  -> OS page cache (RAM)  -- fsync() -->  device write cache  -- flush/FUA -->  stable media
                ^ lost on power cut                  ^ lost on power cut unless
                                                       battery/capacitor-backed
```

Practical points:

- fsync is **slow**: on spinning disks, milliseconds; on SSDs, tens of microseconds to milliseconds;
  on network block storage, often ~1 ms or more. That's why group commit matters.
- Some disks and virtualised storage **lie** — acknowledging a flush before the data is truly persistent.
  Enterprise SSDs with **power-loss protection** (capacitors) can safely acknowledge from cache.
- **fsync errors are serious.** In 2018 the PostgreSQL community discovered ("fsyncgate") that on Linux,
  if fsync fails, the kernel may drop the dirty pages and a retried fsync can report success even though
  the data never reached disk. Postgres now treats an fsync failure as fatal (PANIC and recover from WAL)
  instead of retrying.
- Many databases let you **relax** durability for speed: Postgres `synchronous_commit = off`, MySQL
  `innodb_flush_log_at_trx_commit = 2`, Redis `appendfsync everysec`. A crash can then lose the last few
  hundred milliseconds of acknowledged commits — but never corrupts the database, because the WAL ordering
  rule still holds.

### 5.5 Crash recovery

After a crash, the data files may be in any state: some changes from committed transactions may not have
reached data pages, and some changes from uncommitted transactions may have. The WAL fixes both.

The classic algorithm is **ARIES** (IBM, 1992), used in spirit by most databases. Simplified:

```
1. ANALYSIS: Read the WAL from the last checkpoint. Work out which transactions were
             in flight at the crash and which pages might be dirty.

2. REDO:     Replay WAL records forward from the checkpoint. For each record, if the page's
             LSN is older than the record's LSN, apply the change. ("Repeat history.")
             After this, the data pages look exactly as they did at the moment of the crash.

3. UNDO:     Roll back transactions that never committed, using undo information,
             logging the undo actions too (so a crash during recovery is also recoverable).
```

A worked timeline:

```
LSN   WAL record                         data file on disk at crash
100   CHECKPOINT                         (all changes <=100 are in data files)
101   T1: page 5  balance 500 -> 400
102   T2: page 9  stock 10 -> 9
103   T1: COMMIT                          <- fsynced; client told "OK"
104   T2: page 12 stock 3 -> 2
---- crash (T2 never committed) ----

Page 5 on disk still says 500 (never flushed). Page 9 on disk says 9 (flushed early by
the background writer, even though T2 hadn't committed).

Redo:  apply 101 -> page 5 = 400.  apply 102 (already applied, page LSN >= 102, skip).
       apply 104 -> page 12 = 2.
Undo:  T2 has no COMMIT -> roll back 104 and 102: page 12 = 3, page 9 = 10.
Result: T1's committed change is present; T2 left no trace.
```

Two properties this demonstrates, in ARIES' vocabulary:

- **No-force:** committed changes don't have to be in data files at commit time (redo covers them).
- **Steal:** uncommitted changes *may* be written to data files early (undo covers them). This gives the
  buffer pool freedom to evict any page when it needs room.

Postgres is a slight variant: with MVCC (topic 22), old row versions stay in the table, so it doesn't need
a separate undo phase — uncommitted rows are simply invisible because their transaction never committed.
InnoDB keeps a separate **undo log** for rollback and MVCC.

### 5.6 Torn pages and full-page writes

A page is 8–16 KB, but the hardware may only guarantee that 512 B or 4 KB is written atomically. A crash
during a page write can leave a **torn page**: half old, half new. Replaying a WAL record that says "change
byte 400 of page 812" onto a torn page doesn't fix it.

Defences:
- **Postgres full-page writes:** the first time a page is modified after a checkpoint, the WAL record
  includes a full image of the page. Recovery restores the whole page from the WAL, then replays later
  changes. (This is why WAL volume spikes right after a checkpoint.)
- **InnoDB doublewrite buffer:** pages are first written to a sequential doublewrite area, fsynced, then
  written to their real location. If the real write tears, the good copy is in the doublewrite area.
- **Page checksums** let the database detect corruption rather than silently using bad data.

### 5.7 The WAL is useful beyond crashes

Because the WAL is an exact, ordered record of every change, it is reused for:
- **Replication** (topic 23): ship WAL records to replicas, which replay them (Postgres streaming
  replication).
- **Point-in-time recovery**: restore a base backup, then replay archived WAL up to a chosen moment (e.g.
  just before someone ran `DELETE` without a `WHERE`).
- **Change data capture** (topic 39): tools like Debezium read the WAL (Postgres logical decoding, MySQL
  binlog) to stream changes to other systems.

The append-only log we started with in section 1 turns out to be the database's source of truth; the
B-tree pages are, in a sense, a cached, query-friendly materialization of it.

---

## 6. The buffer pool and dirty pages

### 6.1 What it is

Reading a page from disk takes ~100 µs on SSD (topic 02); reading it from RAM takes ~100 ns. So databases
keep a large in-memory cache of pages: the **buffer pool** (InnoDB: buffer pool, often 50–75% of RAM;
Postgres: `shared_buffers`, plus the OS page cache behind it).

```
            queries
               |
        +------v-----------------------------------+
        | BUFFER POOL (RAM)                        |
        |  [p1 clean] [p5 DIRTY] [p9 clean] ...    |
        |  page table: page id -> frame            |
        +----|--------------------------^----------+
   write     | evict / flush            | read on miss
   dirty     v                          |
        +------------------------------------------+
        | data files (B-tree pages)                |       WAL file (append, fsync on commit)
        +------------------------------------------+
```

- **Read path:** look up the page in the buffer pool; on a hit, use it. On a miss, read it from disk into
  a free frame (evicting another page if needed).
- **Write path:** modify the page **in the buffer pool**, mark it **dirty**, write a WAL record. The page
  itself is written back later.

### 6.2 Dirty pages and eviction

A **dirty page** is one whose in-memory version is newer than the on-disk version. Before a dirty page can
be evicted, it must be written to disk — and, by the WAL rule, only after the WAL is durable up to that
page's LSN.

Eviction uses an LRU-like policy, but plain LRU is a bad fit: one full table scan would push every hot page
out of the cache. Databases protect against this:
- **InnoDB** uses a midpoint-insertion LRU: newly read pages enter in the middle ("old" sublist) and only
  move to the "young" end if accessed again after a delay.
- **Postgres** uses a clock-sweep algorithm with usage counts, and uses a small ring buffer for large
  sequential scans so they don't flood `shared_buffers`.

**Background writers** flush dirty pages proactively so that a query needing a free frame doesn't have to
wait for a write.

### 6.3 Checkpoints

If dirty pages were never flushed, recovery would have to replay the WAL from the beginning of time, and
the WAL could never be deleted. A **checkpoint** bounds this:

1. Note the current WAL position.
2. Flush all pages that were dirty at that point to the data files (Postgres spreads this over time to
   avoid an I/O spike: `checkpoint_completion_target`).
3. Write a checkpoint record to the WAL.

After a checkpoint, recovery only needs to start from the checkpoint's position, and older WAL segments can
be recycled (or archived for backups/replicas).

The trade-off:

| Frequent checkpoints | Infrequent checkpoints |
|---|---|
| Faster crash recovery (less WAL to replay) | Slower crash recovery |
| More I/O: hot pages written repeatedly | Hot pages written once per long interval |
| More full-page images in the WAL (Postgres) | Less WAL volume |

### 6.4 Why not just use the OS page cache?

The OS already caches file pages, so why do databases build their own? Because the database knows more:
- It must control **when** a page is written (only after its WAL record is durable). The OS might write a
  dirty page whenever it likes.
- It can choose eviction policies aware of access patterns (scans vs. index lookups).
- It can pin pages being modified and coordinate with transactions.

That's also why many databases use `O_DIRECT` to bypass the OS cache for data files (InnoDB typically
does), avoiding double caching. Postgres is a notable exception: it relies partly on the OS cache, which
is why `shared_buffers` is usually set to ~25% of RAM rather than most of it.

---

## 7. Putting it all together: the life of one UPDATE

```
UPDATE products SET stock = stock - 1 WHERE id = 77;   COMMIT;

1. Planner chooses the primary-key B-tree (topic 18).
2. Traverse: root (in buffer pool) -> internal (in buffer pool) -> leaf page 4410 (miss: read from disk).
3. Lock the row (topic 22), latch the page.
4. Change stock 10 -> 9 in page 4410 in memory; mark it dirty; page LSN = 9001.
5. Append WAL record LSN 9001 (if first change since checkpoint, include a full page image).
6. COMMIT: append commit record LSN 9002; fsync WAL (group-committed with other txns).
7. Reply OK.                                                   <- durable from here
8. Seconds later: background writer / checkpoint writes page 4410 to the data file.
9. Next checkpoint passes LSN 9002 -> those WAL segments can be recycled.

Crash between 7 and 8?  Recovery redoes LSN 9001 onto page 4410. Stock = 9. Nothing lost.
Crash before 6's fsync? The client never got OK; the change is either absent or undone. Also correct.
```

---

## 8. How it connects

- **Topic 02 (latency):** the entire design — sequential WAL, page-sized I/O, wide shallow trees, buffer
  pools — follows from the gap between RAM, SSD and disk latencies, and between sequential and random I/O.
- **Topic 18 (indexes & query planning):** B-trees are what indexes are made of; clustered vs secondary
  indexes are about what the leaves contain.
- **Topic 19 (LSM-trees):** the other answer to "keep data sorted on disk", trading read cost for
  sequential writes; also uses a WAL for its memtable.
- **Topic 21 (transactions):** durability ("D" in ACID) is implemented by the WAL; atomicity by undo.
- **Topic 22 (concurrency control):** MVCC and locking operate on top of these pages; latches vs locks.
- **Topic 23 (replication):** WAL shipping is one way leaders feed followers.
- **Topic 39 (CDC):** change data capture reads the WAL/binlog.
- **Topic 42 (event sourcing):** the same idea — a log as the source of truth, state as a derived view.

---

## Common misconceptions

- "Committing a transaction writes the changed rows to the data file" → Commit only makes the WAL durable;
  data pages are written later, lazily.
- "`write()` returning means the data is on disk" → It's in the OS page cache; only `fsync` (and an honest
  device) makes it durable.
- "B-tree lookups in a billion-row table take ~30 disk reads" → That's a binary tree. With fan-out in the
  hundreds, depth is 3–4, and the upper levels are cached, so it's typically 1–2 reads.
- "The WAL is just for crash recovery" → It also drives replication, point-in-time recovery and CDC.
- "If fsync fails, retry it" → On Linux, a failed fsync may have dropped the dirty data; retrying can falsely
  succeed. Databases now crash and recover from the WAL instead.
- "Turning off synchronous commit risks corruption" → It risks losing the last few acknowledged commits,
  not corrupting the database (unlike disabling fsync entirely, which can corrupt it).
- "A bigger buffer pool always fixes slowness" → Only if the working set doesn't fit; it doesn't help
  write-heavy workloads bottlenecked on WAL fsync or checkpoint I/O.

## Further reading (supplements)

- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 3 ("Storage and Retrieval").
- Alex Petrov, *Database Internals*, Part I (B-trees, pages, buffer management, recovery).
- Mohan et al., "ARIES: A Transaction Recovery Method..." (ACM TODS, 1992).
- Bayer & McCreight, "Organization and Maintenance of Large Ordered Indexes" (1970/1972) — the original B-tree.
- PostgreSQL documentation, "Reliability and the Write-Ahead Log" chapter.
- CMU 15-445/645 Database Systems lectures (Andy Pavlo), on YouTube — buffer pools, B+ trees, logging and recovery.
- "PostgreSQL's fsync() surprise" (LWN.net, 2018) — on fsyncgate.

---

## Questions

1. Why are writes to an append-only log so fast, and why are reads from it so slow?
2. What two limitations of a hash index lead us toward sorted structures?
3. Why does a sorted file let you keep only a sparse index in memory?
4. Explain why a B-tree over billions of rows needs only 3–4 levels. Why do the top levels almost never
   cause disk reads?
5. Walk through what happens when you insert a key into a full leaf page. How does the tree get taller?
6. Why might using random UUIDv4 values as a B-tree primary key hurt insert performance compared with
   sequential IDs?
7. State the write-ahead rule. What would go wrong if a data page were written to disk before its WAL
   record?
8. Why is committing via the WAL faster than writing the modified data pages at commit time?
9. A transaction is acknowledged as committed, then the machine loses power before its data pages are
   written. Explain step by step how the database recovers the change.
10. What is a torn page, and how do Postgres and InnoDB defend against it?
11. Why do databases maintain their own buffer pool instead of relying entirely on the OS page cache?
12. What does a checkpoint do, and what is the trade-off in how often to run it?
13. A team sets `synchronous_commit = off` for throughput. What can they lose in a crash, and what can't
    happen?
14. Why did Postgres change its behaviour on fsync failure to crash instead of retrying?
15. How is the same WAL used for replication, point-in-time recovery and CDC?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Appending is sequential I/O at the end of the file — no seeks or searching for the right place, and
crash damage is limited to the tail. But there's no index, so finding a key requires scanning the whole
file (O(n)), and you must find the *last* occurrence because updates are new appends.

**2.** All keys must fit in memory (an on-disk hash map performs poorly with random I/O), and it can't do
range queries because hashing scatters adjacent keys.

**3.** If keys are in order, you only need to know roughly where a key is: remember one key per few KB of
file, find the nearest indexed key below the one you want, seek there and scan forward a short way. A
hash index needs every key because unordered data gives no such locality.

**4.** Each page holds hundreds of child references (fan-out ~500 with 8 KB pages), so capacity grows as
500^depth: 4 levels address tens of billions of keys. The root plus the second level is only a few hundred
pages (a few MB), accessed by every lookup, so they stay hot in the buffer pool; only the leaf (and maybe
one level above) typically needs a disk read.

**5.** The leaf is split: a new page is allocated, about half the keys move to it, and a separator key
plus a reference to the new page is inserted into the parent. If the parent is full, it splits too,
recursively. If the root splits, a new root is created above the two halves, increasing depth by one for
all leaves at once, keeping the tree balanced.

**6.** Random keys land on random leaves throughout the tree, so each insert likely needs a different
leaf page — more cache misses, more dirty pages, splits everywhere, pages left half-full, and more WAL
(full-page images). Sequential IDs always insert into the rightmost leaf, which stays in memory, and
splits can be optimised to keep pages full.

**7.** WAL records describing a change must be durable before the modified page is written to disk, and a
transaction commits when its commit record is durable. If a page were written first and the system
crashed before the WAL record, recovery would find changes in data pages that the log knows nothing
about — it couldn't undo an uncommitted change or repair a half-done multi-page operation like a split.

**8.** The WAL is appended sequentially and records are small, and group commit amortises one fsync over
many transactions. Writing data pages would mean several random writes of full pages per transaction,
and hot pages would be written again for every change instead of once per checkpoint.

**9.** At commit the WAL records and commit record were fsynced. On restart, recovery starts from the last
checkpoint, reads the WAL forward and, for each record whose LSN is newer than the target page's LSN,
reapplies the change (redo). The committed change is thus reconstructed in the page. Transactions without
a commit record are rolled back (undo, or in Postgres simply invisible via MVCC).

**10.** A page write interrupted by a crash, leaving the page part old and part new because the device
only writes smaller units atomically. Postgres writes a full image of each page into the WAL on its first
modification after a checkpoint and restores from it during recovery. InnoDB writes pages first to a
doublewrite buffer, fsyncs, then to their real location, so a good copy always exists. Checksums detect
torn pages.

**11.** The database must control when dirty pages are written (only after the WAL is durable up to their
LSN); the OS would write whenever it likes. The database also knows access patterns (protect hot pages
from scans), can pin pages in use, and can avoid double caching with direct I/O.

**12.** It flushes all pages dirty as of a WAL position and records that position, so recovery can start
there and older WAL can be recycled. Frequent checkpoints mean fast recovery but more I/O (hot pages
written repeatedly, more full-page images); infrequent ones mean less I/O but longer recovery.

**13.** They can lose the most recently acknowledged commits (typically up to a few hundred ms, until the
WAL writer flushes). The database can't become corrupted or end up with a partial transaction, because
the WAL is still written in order and still precedes data pages.

**14.** On Linux, when fsync fails the kernel may mark the dirty pages clean and discard them; a later fsync
then succeeds even though the data never reached disk. Retrying would let Postgres believe data was
durable when it wasn't, e.g. then advance a checkpoint and recycle the WAL. Crashing forces recovery from
the WAL, which still has the changes.

**15.** The WAL is an ordered record of every change. Replicas receive and replay it to stay in sync
(streaming replication); archived WAL replayed onto a base backup reconstructs the database at any point
in time; logical decoding of the WAL (or MySQL's binlog) produces a stream of row changes for other
systems.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Why is an append-only log fast to write?
**A:** Appends are sequential I/O; no seeking or in-place modification.

**Q:** What does a hash index store in memory?
**A:** A map from each key to the byte offset of its latest record in the log.

**Q:** Two limits of a hash index?
**A:** All keys must fit in memory; no efficient range queries.

**Q:** What is a tombstone?
**A:** A special record marking a key as deleted, honoured during compaction.

**Q:** What does sorting keys on disk enable?
**A:** Range queries, sparse in-memory indexes, and cheap sequential merges.

**Q:** Two families of sorted on-disk structures?
**A:** Log-structured (LSM-trees, immutable files merged) and update-in-place (B-trees).

**Q:** What is a B-tree page?
**A:** A fixed-size block (4–16 KB), the unit of reading and writing, referenced by page number.

**Q:** What is fan-out (branching factor)?
**A:** Number of child references per page — typically several hundred.

**Q:** Why is a B-tree only 3–4 levels deep for billions of rows?
**A:** Capacity grows as fan-out^depth; 500^4 ≈ 60 billion.

**Q:** How many disk reads does a typical B-tree lookup cost?
**A:** One or two; the upper levels are cached in the buffer pool.

**Q:** What happens when a leaf page is full on insert?
**A:** It splits in two and a separator key is added to the parent, possibly cascading upward.

**Q:** How does a B-tree get taller?
**A:** Only when the root splits and a new root is created.

**Q:** Why are random UUIDv4 keys bad for B-tree inserts?
**A:** Inserts scatter across all leaves: more cache misses, splits, half-full pages, more WAL.

**Q:** Latch vs lock?
**A:** Latch: short in-memory lock protecting a page structure. Lock: transaction-level protection of logical data.

**Q:** The write-ahead rule?
**A:** WAL records for a change must be durable before the changed page is written; commit = commit record durable.

**Q:** What is an LSN?
**A:** Log sequence number: a record's position in the WAL; pages store the LSN of their last change.

**Q:** What is group commit?
**A:** Flushing the WAL once for many concurrently committing transactions.

**Q:** What does fsync do?
**A:** Forces a file's buffered data from the OS page cache to the device's stable storage.

**Q:** Three phases of ARIES recovery?
**A:** Analysis, redo (repeat history from checkpoint), undo (roll back uncommitted transactions).

**Q:** Steal / no-force?
**A:** Steal: uncommitted changes may be flushed early (undo fixes). No-force: committed changes needn't be flushed at commit (redo fixes).

**Q:** What is a torn page?
**A:** A page only partially written during a crash — part old, part new.

**Q:** Postgres's defence against torn pages?
**A:** Full-page writes: a full page image in the WAL on first modification after a checkpoint.

**Q:** What is the buffer pool?
**A:** The database's in-memory cache of disk pages.

**Q:** What is a dirty page?
**A:** A page modified in memory but not yet written back to disk.

**Q:** What does a checkpoint achieve?
**A:** Flushes dirty pages so recovery can start from that point and older WAL can be recycled.

**Q:** Checkpoint frequency trade-off?
**A:** Frequent: fast recovery, more I/O. Infrequent: less I/O, slower recovery.

**Q:** Besides recovery, what else uses the WAL?
**A:** Replication, point-in-time recovery, change data capture.
