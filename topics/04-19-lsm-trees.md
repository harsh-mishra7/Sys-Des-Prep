# 19 — LSM-trees

> Part 4 — Storing Data · Prerequisites: topics 17, 18 (02 helps for the disk-latency intuition)

Topic 17 showed the B-tree: a structure that keeps data sorted on disk by updating fixed-size pages in place.
That design is excellent for reads, but every write lands on some random page somewhere in the tree, and
random writes are the most expensive thing you can ask of a storage device. The **log-structured merge-tree
(LSM-tree)** takes the opposite stance: never update anything in place. Buffer writes in memory, flush them
to disk as large sorted immutable files, and clean up later in the background. This trades some read work and
some background work for very high write throughput. It is the engine behind Cassandra, ScyllaDB, HBase,
Bigtable, RocksDB, LevelDB, and many others (RocksDB is embedded inside CockroachDB, TiKV, Kafka Streams state
stores, and more). Understanding it explains why these systems behave the way they do: why writes are cheap,
why deletes are strange, why "compaction" shows up in every operations runbook.

By the end of this topic you should be able to:
1. Trace a write and a read through an LSM-tree: WAL, memtable, SSTables, levels.
2. Explain why turning random writes into sequential writes increases throughput so much.
3. Explain how Bloom filters, sparse indexes, and block caches keep reads fast despite many files.
4. Compare size-tiered and leveled compaction and say which workloads each suits.
5. Define read, write, and space amplification and use them to reason about any storage engine.
6. Decide whether a B-tree or an LSM-tree fits a given workload, with reasons.

---

## 1. The problem: random writes are expensive

### 1.1 Why "in place" hurts

A B-tree (topic 17) stores keys in pages of, say, 8 KB. To insert a key you find the right leaf page, modify it,
and eventually write the whole page back. If your inserts are spread across the key space (user IDs, UUIDs,
hashes) each one touches a different page. Writing one 100-byte row ends up costing an 8 KB page write
(plus the WAL record). That is **write amplification**: the device writes far more bytes than the application
asked for.

And the writes are *random*: page 9,184, then page 211, then page 77,302.

- On a spinning disk, a random write costs a seek (~5–10 ms). A sequential write of the same data costs a tiny
  fraction of that. Sequential throughput can be 100x or more higher than random-write throughput.
- On an SSD the gap is smaller but real. SSDs can't overwrite a page in place: they write to fresh flash pages
  and erase whole blocks (hundreds of KB to MBs) later. Small random writes fragment blocks, forcing the drive's
  own garbage collector to copy live data around (the SSD's *internal* write amplification), which costs
  throughput and wears the flash out faster. Large sequential writes fill whole erase blocks cleanly.

### 1.2 The log is the fastest possible write

Topic 17 started from the simplest storage engine: append every write to the end of a file. Appending is purely
sequential and needs no lookup before writing. The problem was reads: to find a key you'd scan the whole log.
A hash index in memory fixes that but requires all keys to fit in RAM and can't do range queries.

The LSM-tree keeps the append-only write path but adds **sorting**, which gives you efficient lookups and range
scans without in-place updates. The core idea:

```
  Write path: never seek. Accumulate in memory, sort there, write out big sorted chunks sequentially.
  Read path:  look in a small number of sorted chunks, newest first.
  Background: merge chunks to keep their number (and garbage) under control.
```

---

## 2. The mechanism: memtable → SSTable → compaction

### 2.1 The pieces

```
                    write(k, v)
                        │
           ┌────────────┴─────────────┐
           ▼                          ▼
   ┌───────────────┐          ┌───────────────┐
   │  WAL (on disk)│          │   MEMTABLE    │   in memory, sorted
   │ append-only   │          │ (skip list or │   (e.g. 64 MB)
   │ for durability│          │  red-black    │
   └───────────────┘          │  tree)        │
                              └──────┬────────┘
                                     │ when full: freeze, flush
                                     ▼
   DISK  ─────────────────────────────────────────────────────────
     Level 0:  [SST] [SST] [SST]            (fresh flushes, may overlap)
     Level 1:  [SST a–f][SST g–m][SST n–z]  (non-overlapping)
     Level 2:  [..][..][..][..][..][..][..][..][..][..]   (10x bigger)
     ...
```

- **WAL (write-ahead log).** Same purpose as in topic 17: the memtable lives in RAM, so a crash would lose it.
  Every write is first appended to the WAL. On restart, replay the WAL to rebuild the memtable. Once a memtable
  has been flushed to an SSTable, its WAL segment can be deleted.
- **Memtable.** An in-memory sorted structure (a skip list in RocksDB/LevelDB, for example). Inserts are cheap
  because they're in memory. Because it is sorted, it can be written out in key order.
- **SSTable (Sorted String Table).** An immutable file of key-value pairs sorted by key. Once written it is never
  modified, only eventually deleted after being merged into a newer file.
- **Compaction.** A background process that merges SSTables, discarding overwritten values and deleted keys.

### 2.2 The write path, step by step

1. Append `(k, v)` to the WAL (and, depending on the durability setting, fsync it; see topic 17).
2. Insert `(k, v)` into the memtable. Done: acknowledge the client.
3. When the memtable reaches its size threshold, freeze it (it becomes read-only, a new empty memtable takes
   writes) and flush it in the background: iterate in key order and write a new SSTable sequentially.
4. Delete the WAL segment that covered the flushed memtable.

Notice what is *not* in the write path: no reading of existing data, no lookup of where the key lives, no
random I/O. An update to an existing key is just a new entry with the same key; the old version still sits in
some older SSTable until compaction removes it. This is why LSM writes are "blind": the engine doesn't need to
know if the key already exists.

(Caveat: if the application needs a read before write, e.g. "insert only if absent" or a uniqueness check,
that read costs the same as any other read. The blind-write advantage applies to pure puts.)

### 2.3 Inside an SSTable

A typical SSTable file:

```
 ┌──────────────────────────────────────────────────────────────┐
 │ data block 0:  apple→..  apricot→..  avocado→.. ...  (~4 KB) │
 │ data block 1:  banana→.. berry→.. ...                         │
 │ ...                                                           │
 │ data block N                                                  │
 ├──────────────────────────────────────────────────────────────┤
 │ index block:  first key of each data block + its offset      │
 │                apple → 0, banana → 4096, cherry → 8192 ...   │
 ├──────────────────────────────────────────────────────────────┤
 │ Bloom filter block (for all keys in this file)               │
 ├──────────────────────────────────────────────────────────────┤
 │ footer: offsets of index and filter, min/max key, metadata   │
 └──────────────────────────────────────────────────────────────┘
```

- Data blocks are usually **compressed** (Snappy, LZ4, Zstd). Because files are written once, compression is
  easy; in a B-tree, compressing a page that is later updated in place is awkward.
- The **sparse index** stores only one key per block, not every key. Because the file is sorted, to find
  `blueberry` you find the last index entry `≤ blueberry` (here `banana`), read that one block, and search
  inside it. The index is small enough to keep in memory.
- The **Bloom filter** answers "is this key possibly in this file?" (section 3).

### 2.4 Deletes: tombstones

You can't delete from an immutable file. So a delete is a write of a special marker, a **tombstone**:
`(k, DELETED)`. Reads that find the tombstone first report "not found", even though older SSTables still
contain a value for `k`. Compaction eventually drops both the old value and, once it is sure no older copy
can exist below it, the tombstone itself.

Consequences worth knowing:
- Deletes *add* data before they remove it. A mass delete can temporarily increase disk usage.
- Reads that scan over ranges full of tombstones get slow: the engine must read and skip each one. Cassandra
  famously warns ("tombstone overwhelming") when a query scans too many, and queue-like workloads (insert,
  then delete shortly after) are a classic anti-pattern for LSM stores.
- In replicated systems tombstones must live long enough to reach every replica (Cassandra's `gc_grace_seconds`);
  dropped too early, a replica that missed the delete can "resurrect" the data during repair (topic 24).

### 2.5 The read path, step by step

To read key `k`, look from newest to oldest and stop at the first hit:

```
 get(k):
   1. active memtable           → found? return (or "not found" if tombstone)
   2. immutable memtable(s)     → (being flushed)
   3. Level 0 SSTables, newest first (they may overlap, so check each one)
   4. Level 1: only the ONE file whose key range covers k
   5. Level 2: only the ONE file whose key range covers k
   ...
   not found anywhere → key does not exist
```

Newest-first is what makes updates and deletes correct: the latest version shadows older ones.

The worst case is a key that doesn't exist: you have to check every place. Without help, that's one disk read
per SSTable consulted. Section 3 is about that help.

### 2.6 Range scans

A range scan (`k BETWEEN 'm' AND 'p'`) opens an iterator on the memtable and on every SSTable whose range
overlaps, and does a **k-way merge** of the sorted streams (like the merge step of merge sort), with the newest
source winning ties and tombstones hiding older values. This works because everything is sorted. It's more
work than a B-tree range scan, which walks one chain of leaf pages, because several sources are merged.

### 2.7 Why sequential writes win, summarized

| | B-tree insert | LSM insert |
|---|---|---|
| Reads before write | Yes, walk the tree to the leaf (often cached) | No |
| Disk writes per insert | WAL record + eventually a whole page (random location) | WAL record; data later written in big sequential flushes |
| I/O pattern | Random page writes | Sequential appends and sequential file writes |
| Batching | Limited; each dirty page written separately | Thousands of writes batched into one flush |
| Compression | Awkward (pages mutate) | Natural (files immutable) |

The LSM still writes each byte more than once (flush, then each compaction rewrites it), but every one of those
writes is large and sequential, which devices handle far better.

---

## 3. Keeping reads fast

An LSM-tree may have dozens of SSTables. A naive read would check them all. Several tricks prevent that.

### 3.1 Bloom filters

A **Bloom filter** is a compact probabilistic set (topic 50 covers it in depth). It answers "is `k` in this set?"
with either **"definitely not"** or **"probably yes"**. It never gives a false negative; it sometimes gives a
false positive.

How it works:
- A bit array of `m` bits, all zero, and `k` independent hash functions.
- To add a key: compute the `k` hashes, set those `k` bits to 1.
- To query: compute the `k` hashes. If *any* of those bits is 0, the key was never added (definitely not).
  If all are 1, it was probably added, but those bits might have been set by other keys.

```
 bits:     0 1 0 0 1 0 1 0 0 1 0 0
 add "cat":  h1=1, h2=4, h3=9      → set bits 1,4,9
 add "dog":  h1=4, h2=6, h3=9      → set bits 4,6,9
 query "eel": h1=1, h2=3, h3=6     → bit 3 is 0 → DEFINITELY NOT present
 query "fox": h1=1, h2=6, h3=9     → all 1 → "probably" (false positive!)
```

With about **10 bits per key** the false-positive rate is about 1%. Each SSTable carries a Bloom filter over its
keys, and the filters are held in memory. Now a point read does:

```
 for each candidate SSTable (newest first):
     if bloom.mightContain(k) == false: skip (no disk I/O)
     else: read index → read one data block → search
```

For a key that doesn't exist, ~99% of SSTables are skipped without touching disk. For a key that does exist,
you usually read exactly one data block from the file that has it. Bloom filters only help **point lookups**;
they can't answer "any key between m and p?" (RocksDB adds *prefix* Bloom filters for prefix-bounded scans).

### 3.2 Other read helpers

- **Key range per file.** Each SSTable's footer records its min and max key. In levels ≥ 1, files in a level
  don't overlap, so a binary search over file ranges finds the single candidate file per level.
- **Sparse index in memory.** Finding the right block inside a file costs no disk I/O.
- **Block cache.** Recently read (decompressed) data blocks are cached in memory, just like a B-tree's buffer
  pool (topic 17). Hot keys never hit disk.
- **Compaction itself.** Fewer files means fewer places to look. Compaction is as much about read performance as
  about space.

### 3.3 What the read cost looks like

With leveled compaction (section 4) and Bloom filters, a typical point read costs:
- Memtable check: in memory.
- One Bloom check per L0 file and one per level: in memory.
- About one data block read from disk (often a cache hit).

That's competitive with a B-tree point read. Where LSMs remain worse: range scans (merging many sources), reads
of keys with many versions or tombstones, and tail latency when compaction competes for disk bandwidth.

---

## 4. Compaction strategies and their costs

Without compaction, SSTables accumulate forever: reads slow down (more files), and disk fills with overwritten
and deleted data. Compaction merges files: read several sorted inputs, k-way merge, keep only the newest version
of each key, drop data shadowed by tombstones, write new sorted outputs, delete the inputs. Because inputs and
outputs are sorted, compaction is a purely sequential streaming operation, and it needs little memory.

*How* you choose what to merge is the big design decision. Two families dominate.

### 4.1 Size-tiered compaction (STCS)

Group SSTables by similar size. When there are enough files of roughly the same size (e.g. 4), merge them into
one larger file.

```
  flushes:      [8][8][8][8]           → merge → [32]
                [32][32][32][32]       → merge → [128]
                [128][128][128][128]   → merge → [512]
```

- **Write-friendly.** Each byte is rewritten only about once per tier (log of the data size, base 4 here). Low
  write amplification.
- **Read-unfriendly.** Files of different tiers overlap in key range, so a key can be in any of them. Many files
  to check; Bloom filters help point reads but range scans and keys that have been updated many times suffer.
- **Space-hungry.** A merge needs room for inputs and output simultaneously. Merging the largest tier can
  temporarily need roughly double that tier's size. Also, an overwritten key can sit in several tiers for a long
  time, so stale data lingers. Worst case space amplification is around 2x.
- Default in Cassandra historically; good for write-heavy, insert-mostly data.

### 4.2 Leveled compaction (LCS)

Organize data into levels L1, L2, L3, ..., each about **10x** larger than the previous. Within each level
(except L0) SSTables are small (e.g. 64 MB) and have **non-overlapping key ranges**: each level is one sorted run
split into files.

```
 L0:  [a–z] [c–x] [b–y]                      fresh flushes, overlapping
 L1:  [a–d][e–h][i–m][n–r][s–z]              total  ~ 256 MB
 L2:  [a][b][c][d][e]...(50 files)...        total  ~ 2.5 GB
 L3:  ...                                     total  ~ 25 GB
```

When a level exceeds its target size, pick one file from it and merge it with the files in the next level that
overlap its key range (about 10 of them, since the next level is 10x denser), writing new non-overlapping files
into the next level.

- **Read-friendly.** At most one file per level can contain a key. With ~5–7 levels, a point read checks at most
  a handful of Bloom filters and usually reads one block.
- **Space-efficient.** Since ~90% of data lives in the last level and each level holds at most one version of a
  key, stale data is bounded at roughly 10%. Space amplification ~1.1x.
- **Write-expensive.** Moving one file down a level rewrites ~10 overlapping files in the next level. Each byte
  can be rewritten ~10 times *per level*. Write amplification can be 10–30x in practice.
- Default in RocksDB and LevelDB; Cassandra offers it as LCS. Good for read-heavy or update-heavy workloads
  where the same keys are overwritten often.

### 4.3 Others worth knowing

- **Time-window compaction (TWCS)** in Cassandra: group SSTables by the time window their data was written in and
  never merge across windows. For time-series with TTLs, an entire window's file expires at once and is simply
  deleted, no merge needed. Bad if you update old data.
- **Hybrids**: RocksDB's "universal" compaction is size-tiered-like; many systems use tiered at upper levels and
  leveled at the bottom to balance write and space costs.

### 4.4 Comparison

| | Size-tiered | Leveled |
|---|---|---|
| Write amplification | Low (~log of data size) | High (~10 per level) |
| Read amplification | Higher (many overlapping files) | Low (1 file per level) |
| Space amplification | High (up to ~2x; temporary doubling during big merges) | Low (~1.1x) |
| Best for | Write-heavy, append-mostly, rarely-updated data | Read-heavy, frequently updated keys, tight disk |

### 4.5 When compaction goes wrong

Compaction uses the same disk bandwidth and CPU as foreground traffic. Failure modes:

- **Compaction falls behind.** If sustained write rate exceeds what compaction can merge, L0 fills with files.
  Reads slow down (more overlapping files to check). Engines protect themselves with **write stalls**: RocksDB
  slows down, then stops, writes when L0 has too many files or too many bytes are "pending compaction". Your
  write latency suddenly spikes from microseconds to seconds. This is the LSM version of backpressure.
- **Latency spikes from compaction I/O.** A big merge saturates the disk and foreground reads queue behind it
  (topic 02's queueing). Mitigations: rate-limit compaction, run it on separate threads with priority, use
  faster disks.
- **Disk-full during compaction.** Size-tiered merges need temporary headroom. Running at 80% disk utilization
  with STCS can make the final big merge impossible.
- **Tombstone build-up.** As in 2.4, scans crawl over deletion markers.

---

## 5. Read, write, and space amplification

These three numbers are the vocabulary for comparing any storage engine.

- **Write amplification (WA)** = bytes written to the device / bytes written by the application.
  Includes WAL, flushes, compaction rewrites, and (on SSDs) the drive's internal GC.
- **Read amplification (RA)** = number of disk reads (or bytes read) per logical read.
- **Space amplification (SA)** = bytes on disk / bytes of live data.

A well-known observation (sometimes called the **RUM conjecture**: Read, Update, Memory) is that you can optimize
any two of the three only at the expense of the third. Every engine design picks a point:

```
           low read amp
               ▲
               │    B-tree (in-place, 1 location per key)
               │    Leveled LSM
               │
               │               Size-tiered LSM
               │
               └──────────────────────────────▶ low write amp
                                    Pure append log (no compaction):
                                    WA≈1 but RA and SA unbounded
```

Rough figures (they vary hugely with configuration and workload):

| Engine | Write amp | Read amp (point) | Space amp |
|---|---|---|---|
| B-tree | Page size / row size for random updates (can be 10–100x for small rows), plus WAL | ~1 (inner pages cached) | ~1.3–1.5x (pages partly empty after splits) |
| LSM leveled | ~10–30x but sequential | ~1 with Bloom filters + cache | ~1.1x |
| LSM size-tiered | ~4–10x | Several files | up to ~2x |

Why these matter in practice:
- WA determines how many writes per second a given disk sustains, and how fast an SSD wears out.
- RA determines read latency and how much memory you need for caches.
- SA determines how much disk (and money) you need, and whether you risk running out during compaction.

Note the subtlety in comparing WA: a B-tree's write amplification is made of *random* page writes; an LSM's
(even if numerically larger) is made of *sequential* writes. Bytes alone don't tell the whole story; the access
pattern matters (topic 02).

---

## 6. B-tree vs LSM: when each wins

### 6.1 Rules of thumb

**LSM-trees tend to win when:**
- The workload is write-heavy (ingest, logging, time series, event data, metrics, messaging, IoT).
- Writes are spread randomly across a large key space (random UUID keys hurt B-trees more).
- Disk space or SSD wear matters and you want compression.
- You can tolerate some background I/O and occasional latency variance.

**B-trees tend to win when:**
- The workload is read-heavy, especially with range scans.
- You need predictable latency: no background compaction storms.
- You need strong transactional semantics with in-place locking. Each key lives in exactly one place, so
  row/range locks can attach to the tree structure itself (topic 22). LSM engines can do transactions
  (RocksDB, CockroachDB, TiKV do), but they implement them on top (e.g. with MVCC keys and separate lock tables).
- Data is updated in place often with stable sizes.

### 6.2 Who uses what

| Engine | Structure |
|---|---|
| Postgres, MySQL InnoDB, SQL Server, Oracle, SQLite | B-tree (B+tree) |
| Cassandra, ScyllaDB, HBase, Bigtable | LSM |
| RocksDB, LevelDB, Pebble | LSM (embedded libraries) |
| CockroachDB, TiKV/TiDB, YugabyteDB | LSM underneath a distributed SQL layer |
| MyRocks (MySQL on RocksDB) | LSM, built to cut space and write amplification versus InnoDB |
| DynamoDB | B-tree-based internals per partition (publicly described) |

The MyRocks story is a good real-world anchor: a large deployment moved MySQL from InnoDB to RocksDB mainly to
reduce disk space (compression, low SA) and SSD wear (lower WA for their workload), accepting somewhat slower
range reads.

### 6.3 Benchmark before believing

These are tendencies, not laws. A B-tree with a large buffer pool and a write-sequential key (e.g. an
auto-increment primary key, where all inserts hit the rightmost page) can ingest very fast. An LSM with a
read-heavy workload, good Bloom filters, and a warm block cache can read very fast. Real decisions should be
made by testing the actual workload, especially at the p99 and at steady state (after compaction has caught up
with weeks of data, not in the first five minutes of an empty database).

---

## 7. How it connects

- **Topic 02 (latency hierarchy & queueing):** sequential vs random I/O is the entire motivation; compaction
  competing with foreground I/O is a queueing problem.
- **Topic 17 (B-trees & WAL):** the alternative design; the WAL plays the same durability role in both.
- **Topic 18 (indexes):** secondary indexes in LSM stores are just more LSM key spaces; the write cost of each
  index still applies, though LSM makes it cheaper per index.
- **Topic 20 (column storage):** column stores often borrow the LSM pattern: buffer writes in a row-oriented
  in-memory store, then write immutable sorted column files and merge them in the background.
- **Topic 22 (concurrency control):** MVCC in LSM engines stores multiple versions as distinct keys
  (`key@timestamp`); compaction doubles as garbage collection of old versions, the LSM equivalent of Postgres's
  vacuum.
- **Topics 24 & 33 (leaderless replication, conflict resolution):** Cassandra's last-write-wins, tombstones and
  `gc_grace_seconds` interact with repair and resurrection.
- **Topic 50 (probabilistic data structures):** Bloom filters in depth.
- **Topic 51 (search):** Lucene's segments are an LSM-like design: immutable segments plus background merging.

---

## Common misconceptions

- "LSM-trees write each byte once, B-trees write it many times." → LSMs rewrite data repeatedly during
  compaction, sometimes 10–30x. Their advantage is that the writes are sequential and batched, not that there
  are fewer of them.
- "LSM-trees are slow to read." → With Bloom filters, leveled compaction and a block cache, point reads are
  typically about one disk read. Range scans and tombstone-heavy reads are the weak spots.
- "A delete frees space immediately." → A delete writes a tombstone; space is reclaimed only after compaction
  merges the tombstone with the older data (and, in replicated stores, after a grace period).
- "Bloom filters make every read one I/O." → They eliminate files that definitely don't contain the key; a
  ~1% false-positive rate still causes occasional wasted reads, and they don't help range scans at all.
- "Compaction is just housekeeping." → It is central to read performance, space usage, and write stalls.
  Most LSM production incidents are compaction incidents.
- "LSM is for NoSQL, B-tree is for SQL." → Many distributed SQL databases (CockroachDB, TiDB, YugabyteDB)
  run on LSM engines. The data model (topic 15) and the storage engine are independent choices.

## Further reading (supplements)

- *Designing Data-Intensive Applications*, Martin Kleppmann, Ch. 3 ("SSTables and LSM-Trees", "Comparing B-Trees
  and LSM-Trees").
- O'Neil et al., "The Log-Structured Merge-Tree (LSM-Tree)" (1996), the original paper.
- Chang et al., "Bigtable: A Distributed Storage System for Structured Data" (OSDI 2006): memtable/SSTable in a
  real system.
- RocksDB wiki on GitHub: "Leveled Compaction", "Universal Compaction", and "RocksDB Tuning Guide".
- Athanassoulis et al., "Designing Access Methods: The RUM Conjecture" (EDBT 2016).
- Bloom, "Space/Time Trade-offs in Hash Coding with Allowable Errors" (1970).

---

## Questions

1. Why does an LSM-tree need a WAL if the memtable is going to be flushed to disk anyway?
2. Walk through what happens on disk when you update the same key 1,000 times in an LSM-tree, and then what
   compaction does with those versions.
3. Why must a read check the memtable and SSTables from newest to oldest? What bug appears if you check oldest
   first?
4. A Bloom filter says "probably present" for a key in 5 SSTables, but the key is only in one. What happened,
   and what does it cost?
5. Why can't Bloom filters help with a query like `SELECT * WHERE key BETWEEN 'm' AND 'p'`?
6. A team runs a queue on Cassandra: producers insert a row per job, consumers read the oldest job and delete
   it. Over a week, "fetch next job" gets slower and slower. Explain.
7. Why does leveled compaction have higher write amplification but lower space amplification than size-tiered?
8. Your RocksDB-backed service has a write-latency spike to several seconds every few minutes during a bulk
   import, though the disk is not full. What is the likely cause, and what are your options?
9. Under size-tiered compaction, why can you run out of disk at 60% utilization?
10. A time-series metrics store writes data once, never updates it, and expires it after 30 days. Which
    compaction strategy fits best, and why?
11. Compare the I/O a B-tree and an LSM-tree perform when inserting 1 million rows with random UUID keys.
12. Why is it natural to compress SSTables but harder to compress B-tree pages?
13. Explain read, write and space amplification, and why improving one usually worsens another.
14. A read-heavy workload with lots of range scans and strict p99 latency targets: B-tree or LSM? Defend it.
15. In a replicated LSM store, why can a deleted row come back if tombstones are purged too early?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** The memtable is in RAM. Between a write being acknowledged and the memtable being flushed (which can be
minutes), a crash would lose those writes. The WAL is a sequential append on disk that records every write
before it's acknowledged; on restart the engine replays it to rebuild the memtable. After a flush, the WAL
segment for that memtable is no longer needed and is deleted.

**2.** Each update appends a WAL record and replaces the entry in the memtable (the memtable keeps only the
latest for that key, though MVCC-enabled engines may keep several). If flushes happen between updates,
multiple SSTables end up holding different versions of the same key. Reads return the newest. Compaction, when
it merges files containing several versions, keeps only the newest one (or the versions still needed by open
snapshots) and drops the rest, reclaiming space.

**3.** Because the newest version must shadow older ones. Updates and deletes are new entries, not modifications.
Checking oldest first would return stale values for updated keys and would return values for keys that were
deleted (the tombstone in a newer place would never be seen).

**4.** False positives: in those 4 files the key's hash bits all happened to be set by other keys. Each false
positive costs an unnecessary index lookup and a data block read (possibly a disk read) that finds nothing.
With ~10 bits per key the rate is ~1%, so this is rare; seeing it on many files suggests filters are too small
or missing.

**5.** A Bloom filter only answers membership of an exact key by hashing it. Hashing destroys ordering, so there
is no way to ask "is any key in this interval present?" The engine must use the files' min/max key ranges to
choose candidate files and then merge iterators over them. (Prefix Bloom filters help only when the range is
bounded by a shared prefix.)

**6.** Each delete writes a tombstone; the deleted rows and tombstones remain until compaction (and the grace
period) removes them. "Oldest job" sits at the start of the partition's sorted range, exactly where all the
tombstones accumulate. Every fetch has to scan past thousands of tombstones before finding a live row, so it
gets slower as tombstones build up. LSM stores are a poor fit for queue patterns; use a real queue (topic 36).

**7.** In leveled compaction, pushing one file into the next level means rewriting the ~10 overlapping files there,
and this happens at every level, so a byte is rewritten many times (high WA). But each level is a single
non-overlapping sorted run holding at most one version per key, and ~90% of data is in the last level, so at
most ~10% of space is stale (low SA). Size-tiered rewrites each byte only once per tier (low WA) but lets
overlapping files with old versions coexist for a long time and needs temporary room for big merges (high SA).

**8.** Write stalls: the import writes faster than compaction can merge, so L0 file count or pending compaction
bytes cross thresholds and RocksDB slows or stops writes until compaction catches up. Options: give compaction
more threads/bandwidth; larger memtables and L0 thresholds (buys time, costs reads); use a write-friendlier
compaction (universal/tiered) for the import; throttle the import; or for bulk loads, build sorted SSTables
externally and ingest them directly (RocksDB supports SST file ingestion).

**9.** A size-tiered merge of the largest tier writes a new file roughly the size of all inputs before deleting
the inputs. If that tier holds most of the data, you temporarily need nearly double its size. Plus stale versions
and tombstones add to the live data. So "60% full" can be insufficient headroom for the next big merge.

**10.** Time-window compaction (or a similar time-bucketed scheme). Data is grouped into files by write time,
never merged across windows, and since it's never updated there's nothing to reconcile across windows. When a
window passes the 30-day TTL, its whole file is dropped, with no tombstone processing or rewriting. Size-tiered
or leveled would repeatedly rewrite data that's about to expire.

**11.** B-tree: each insert walks the tree to a random leaf; most leaves aren't in cache once the tree is large,
so many inserts cause a random page read, then later a random page write of a full page (plus page splits
as leaves fill), plus the WAL. LSM: each insert is a WAL append and a memtable insert; data reaches disk in large
sequential flushes and later sequential compaction rewrites. Total bytes written may be comparable or higher for
the LSM, but the B-tree does random I/O and per-row page writes, so the LSM sustains far higher insert rates.

**12.** SSTables are written once and never modified, so you compress each block as it's written and never have
to recompress. B-tree pages are modified in place; a compressed page that grows after an update may no longer
fit its slot, forcing complex handling (InnoDB's compressed tables do this and pay for it). Also, LSM files are
sorted and dense, which compresses well.

**13.** WA: device bytes written per logical byte written. RA: device reads per logical read. SA: disk bytes per
live byte. They trade off because the work of organizing data has to happen somewhere: keeping data perfectly
organized for reads (one location per key, no stale data) requires rewriting it eagerly (WA); writing lazily
leaves data scattered (RA) and duplicated (SA). The RUM conjecture states this formally.

**14.** Usually a B-tree. Range scans walk one chain of sorted leaf pages rather than merging iterators over
several files; there's no background compaction to cause tail-latency spikes; and each key has one location.
An LSM could work with leveled compaction, rate-limited compaction and big caches, but you'd be fighting its
nature. The decision should still be confirmed with a benchmark at steady state.

**15.** If replica A processed a delete and later purged the tombstone (and the data) during compaction, but
replica B never received the delete, B still has the old value. When anti-entropy repair compares A and B, A has
no record that the key was deleted, so B's value looks like data A is missing and gets copied back: the
deleted row is resurrected. Tombstones must outlive the maximum time replicas can stay out of sync, which is
what Cassandra's `gc_grace_seconds` enforces (and why repairs must run within that window).

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What are the three main components of an LSM-tree?
**A:** Memtable (sorted, in memory), SSTables (immutable sorted files on disk), and compaction (background merging). Plus a WAL for durability.

**Q:** What does SSTable stand for and what's its key property?
**A:** Sorted String Table; an immutable file of key-value pairs sorted by key.

**Q:** Why are LSM writes fast?
**A:** No read before write and no random I/O: writes go to a WAL append and an in-memory memtable, and reach disk in large sequential flushes.

**Q:** What is the WAL for in an LSM-tree?
**A:** Rebuilding the memtable after a crash, since the memtable lives only in RAM.

**Q:** What happens when the memtable fills up?
**A:** It's frozen, a new memtable takes writes, and the frozen one is flushed sequentially to a new SSTable.

**Q:** In what order does a read search the LSM?
**A:** Newest to oldest: memtable, immutable memtables, L0 files, then one file per level downward. Stop at the first hit.

**Q:** How is a delete represented?
**A:** As a tombstone: a write marking the key deleted. Compaction later removes it and the older data.

**Q:** What does a Bloom filter answer?
**A:** "Definitely not in the set" or "probably in the set". No false negatives, some false positives.

**Q:** Rough Bloom filter false-positive rate at 10 bits per key?
**A:** About 1%.

**Q:** Do Bloom filters help range scans?
**A:** No. They work on exact-key hashes only (prefix Bloom filters help prefix-bounded scans).

**Q:** What is a sparse index in an SSTable?
**A:** One index entry per data block (the block's first key), enough to locate the one block that might contain a key.

**Q:** What does compaction do?
**A:** Merges sorted SSTables, keeps only the newest version of each key, drops deleted data, and writes new files.

**Q:** Size-tiered compaction in one line?
**A:** Merge several similar-sized files into a bigger one: low write amp, higher read and space amp.

**Q:** Leveled compaction in one line?
**A:** Levels each ~10x larger, non-overlapping files within a level: low read and space amp, high write amp.

**Q:** Which compaction suits write-once time-series with TTL?
**A:** Time-window compaction: whole windows expire as files are dropped.

**Q:** Define write amplification.
**A:** Bytes written to the device divided by bytes written by the application.

**Q:** Define space amplification.
**A:** Bytes on disk divided by bytes of live data.

**Q:** What is the RUM conjecture?
**A:** Among Read, Update (write) and Memory (space) overheads, optimizing two costs the third.

**Q:** What is a write stall?
**A:** The engine throttling or stopping writes because compaction has fallen behind (too many L0 files or pending bytes).

**Q:** Why are queue-like workloads bad on LSM stores?
**A:** Constant deletes pile up tombstones at the head of the range, so reads must scan past them.

**Q:** Why is compression natural in LSMs?
**A:** Files are immutable, so blocks are compressed once when written and never rewritten in place.

**Q:** When does a B-tree usually beat an LSM?
**A:** Read-heavy and range-scan-heavy workloads, predictable latency needs, in-place locking.

**Q:** When does an LSM usually beat a B-tree?
**A:** Write-heavy workloads, random-key inserts, compression and SSD-wear concerns.

**Q:** Name three LSM-based systems.
**A:** Cassandra, RocksDB, HBase (also LevelDB, ScyllaDB, Bigtable, CockroachDB's Pebble).

**Q:** Why must tombstones outlive replica divergence?
**A:** Otherwise repair copies the old value from a replica that missed the delete, resurrecting the data.
