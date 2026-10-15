# 11 — Caching Fundamentals

> Part 3 — Caching & the Edge · Prerequisites: topics 01, 02

A cache is a smaller, faster store that holds copies of data whose authoritative home is somewhere slower.
Almost every fast system you have used is fast because of caching: the CPU caches RAM, the OS caches the
disk, the database caches its pages, the application caches the database, the CDN caches the application,
and the browser caches the CDN. Caching is the single most effective performance tool in system design, and
also one of the most common sources of subtle bugs, because the moment you keep a copy of data you have two
versions of the truth and must decide what happens when they disagree. This topic covers why caches work at
all, where they sit, the standard patterns for reading and writing through them, how they decide what to
throw away, how they go stale, and how Redis, the most widely used cache server, works inside.

By the end of this topic you should be able to:
1. Explain why caching works, in terms of locality, skewed access distributions and the latency gaps from topic 02.
2. Name the layers where caches live in a typical request path and what each one is good at.
3. Describe cache-aside, read-through, write-through, write-behind and write-around, with the failure modes of each.
4. Compare LRU, LFU, FIFO and TTL eviction, and describe how LRU is implemented in O(1) and approximated in practice.
5. Reason about invalidation, staleness and the race conditions that leave stale data in a cache indefinitely.
6. Explain Redis's single-threaded event loop, its core data structures, and the RDB vs AOF persistence trade-off.

---

## 1. Why caching works

### 1.1 The latency gap

Recall the numbers from topic 02 (orders of magnitude, not exact):

```
L1 cache reference                  ~1 ns
Main memory reference             ~100 ns
Read 1 MB sequentially from RAM   ~10 µs
SSD random read                   ~100 µs
Round trip within a datacenter    ~500 µs
Read 1 MB from SSD                ~1 ms
Disk seek (spinning)              ~10 ms
Round trip cross-continent      ~100-150 ms
```

Each step down is roughly 10–1000× slower than the one above. A cache exploits this: if you can answer a
request from a faster tier instead of a slower one, you save the difference. A database query that touches
disk and runs a join might take 5–50 ms; a lookup in Redis takes ~0.2–0.5 ms including the network hop; a
lookup in an in-process hash map takes well under a microsecond.

But speed alone does not make a cache useful. A cache is smaller than the data it fronts (otherwise it would
just *be* the database). It only helps if the small subset it holds is the subset people actually ask for.
That is where locality comes in.

### 1.2 Locality

- **Temporal locality**: data accessed recently is likely to be accessed again soon. The user who just loaded
  their profile will load it again on the next page. The product on the homepage banner gets read thousands
  of times per minute.
- **Spatial locality**: data near recently accessed data is likely to be accessed soon. CPU caches load whole
  64-byte cache lines; databases read whole 8 KB pages; an application might cache an entire user object
  instead of a single field because the other fields will be wanted next.

### 1.3 Skewed access

Real workloads are rarely uniform. Access frequencies tend to follow a **power law / Zipf-like distribution**:
a small number of items get most of the traffic. On a shopping site, a few thousand products out of millions
account for most page views; on a social network, a small number of accounts generate most reads.

```
requests
  |█
  |█
  |██
  |███
  |█████
  |████████▄▄▄▄▃▃▃▃▂▂▂▂▂▂▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁
  +-------------------------------------------------------------> items (sorted by popularity)
   ^ head: small, hot           long tail: huge, cold
```

This is why a cache holding 1% of the data can serve 80–95% of requests. The fraction of requests served from
the cache is the **hit ratio** (hits / (hits + misses)).

### 1.4 The arithmetic of hit ratio

Effective latency = hit_ratio × cache_latency + (1 − hit_ratio) × (cache_latency + backend_latency)

(Assuming a miss checks the cache first, then goes to the backend.) With cache = 0.5 ms, backend = 20 ms:

| Hit ratio | Effective latency | Backend load (relative) |
|-----------|-------------------|-------------------------|
| 0%        | 20.5 ms           | 100%                    |
| 80%       | 4.5 ms            | 20%                     |
| 95%       | 1.5 ms            | 5%                      |
| 99%       | 0.7 ms            | 1%                      |

Two lessons hide in this table:

1. **Small hit-ratio changes matter a lot at the high end.** Going from 95% to 99% cuts backend load by 5×.
   Conversely, a drop from 99% to 95% quintuples the load on your database. Systems sized assuming a 99% hit
   ratio can fall over when the hit ratio dips (topic 12 is mostly about this).
2. **Caching is as much about protecting the backend as about latency.** Often the real reason a cache exists
   is that the database could not survive the read load without it. That makes the cache load-bearing, which
   changes how you must treat its failures.

### 1.5 What makes data cacheable

Good candidates: read far more often than written; expensive to compute or fetch; tolerant of being slightly
stale; reasonably small. Poor candidates: data that changes on every read, data that must be exactly current
(account balance during a transfer), data accessed uniformly at random with no hot set, and very large values
that would evict everything else.

---

## 2. Where caches live

A single request from a browser to a database can pass through half a dozen caches:

```
 Browser        CDN edge       Reverse proxy     App server            Distributed       Database
 cache    -->   (topic 14) --> (Varnish/nginx)-> [in-process cache] --> cache (Redis) --> [buffer pool]
                                                                                           |
 private,       shared,        shared,          per-process,           shared across     disk
 per user       per region     per datacenter   fastest, small         all app servers
```

| Layer | What it caches | Strengths | Weaknesses |
|-------|----------------|-----------|------------|
| Browser / client | HTTP responses, assets, API data | Zero network cost; offloads everything behind it | You cannot reliably purge it; only as fresh as headers allow |
| CDN (topic 14) | Static assets, cacheable HTTP responses | Close to users, absorbs huge traffic | Coarse control; shared, so personalized data is tricky |
| Reverse proxy (topic 10) | Whole HTTP responses | Shields app servers; simple | Only works for full responses keyed by URL/headers |
| Application-local (in-process) | Objects, computed results | Nanosecond-to-microsecond access, no network | Each server has its own copy: duplicated memory, inconsistent between servers, lost on restart |
| Distributed cache (Redis, Memcached) | Objects, query results, sessions, counters | Shared by all app servers, large, one copy to invalidate | Network hop (~0.2–1 ms); another system to operate |
| Database buffer pool (topic 17) | Disk pages | Automatic, transparent, consistent with the DB | Only saves disk I/O, not query execution, parsing or network |
| OS page cache | File blocks | Free, automatic | Same as above |

A few points worth drawing out:

- **Private vs shared caches.** The browser cache is private to one user; a CDN or reverse proxy is shared by
  everyone. Anything user-specific must never land in a shared cache keyed only by URL, or one user sees
  another's data. `Cache-Control: private` exists for exactly this (topic 14).
- **Local vs distributed.** An in-process cache is ~1000× faster than a network hop to Redis, but with 50 app
  servers you have 50 independent copies that expire at different times. Many systems use both: a tiny local
  cache with a short TTL (seconds) in front of a distributed cache (a **two-tier** or **near cache**). This is
  also the main defense against hot keys (topic 12).
- **The database already caches.** Postgres's shared buffers and the OS page cache keep hot pages in RAM. If
  your working set fits in RAM, a "slow query" is often slow because of CPU work (joins, sorts, serialization),
  not disk. An application cache then saves the work, not the I/O. It is worth knowing which you are saving.
- **Memoization** (caching a function's result by its arguments) is the same idea inside a single program.

---

## 3. Read and write patterns

The cache and the database are two separate stores. The patterns below describe who is responsible for
moving data between them, and in which order.

### 3.1 Cache-aside (lazy loading)

The application talks to both the cache and the database. The cache is passive.

```
read(key):
  value = cache.get(key)
  if value is not null: return value          # hit
  value = db.query(key)                        # miss
  cache.set(key, value, ttl)
  return value

write(key, value):
  db.update(key, value)
  cache.delete(key)                            # invalidate, don't update
```

```
   App ──1. get──> Cache        (miss)
   App ──2. query─> Database
   App ──3. set───> Cache
```

- **Pros**: simple; the cache only holds data that has been requested; if the cache is down, reads fall
  back to the database (slower, but correct); works with any cache and any database.
- **Cons**: every miss costs three steps; the first request for each key is slow; consistency is entirely
  the application's job; many code paths each have to remember to invalidate.

This is by far the most common pattern with Redis or Memcached.

**Why delete on write rather than update?** Two reasons. First, the cached value may be a derived object
(joined from several tables, rendered to JSON); recomputing it on every write is wasteful if it is not read
again. Second, updating has a worse race. Two concurrent writers:

```
 time  Writer A                     Writer B
  t1   db.update(x = 1)
  t2                                db.update(x = 2)
  t3                                cache.set(x = 2)
  t4   cache.set(x = 1)             
       → DB says 2, cache says 1, and it stays wrong until TTL
```

With delete, the worst outcome of this interleaving is an extra miss. Delete is not race-free either
(see section 5.3), but it narrows the window.

### 3.2 Read-through

The cache sits in front of the database and loads missing data itself, via a loader function or plugin you
configure. The application only talks to the cache.

```
   App ──get──> Cache ──(on miss) load──> Database
```

Same behavior as cache-aside, but the loading logic lives in one place (the cache library or a caching layer
such as a Guava/Caffeine `LoadingCache`, or a DAX-style front for DynamoDB). Advantages: the application code
is cleaner and the cache can coalesce concurrent misses for the same key into one load (important for
topic 12). Disadvantage: the cache must understand how to load your data, which couples it to your schema.

### 3.3 Write-through

Every write goes to the cache, and the cache (or a layer wrapping it) synchronously writes to the database
before acknowledging.

```
   App ──write──> Cache ──write (sync)──> Database
        <── ack only after both succeed ──
```

- **Pros**: cache is always populated with fresh data for anything written; reads after writes hit.
- **Cons**: every write pays both latencies; the cache fills with data that may never be read (often paired
  with a TTL to limit this); if the two writes are not atomic, a failure between them leaves them out of sync.

### 3.4 Write-behind (write-back)

Writes go to the cache and are acknowledged immediately; the cache flushes them to the database
asynchronously, often in batches.

```
   App ──write──> Cache ──ack──> App
                    │
                    └── later, batched ──> Database
```

- **Pros**: very fast writes; batching and coalescing (ten updates to one counter become one DB write)
  massively reduce database load. This is how CPU caches and the OS page cache work, and how a database
  buffer pool treats dirty pages (topic 17).
- **Cons**: **data loss** if the cache dies before flushing; the database is behind, so anything reading it
  directly sees old data; ordering and failure handling of the flush are complicated. Use only when the cache
  is itself durable/replicated, or when losing recent writes is acceptable (view counters, analytics).

### 3.5 Write-around

Writes go straight to the database, bypassing the cache entirely; the cache is filled only on reads
(cache-aside style). Usually the write also invalidates the cached entry.

- **Pros**: the cache is not polluted with data that is written but rarely read (logs, bulk imports).
- **Cons**: a read right after a write misses.

### 3.6 Summary

| Pattern | Who loads on miss | Write path | Read-after-write | Main risk |
|---------|------------------|------------|------------------|-----------|
| Cache-aside | Application | DB, then invalidate cache | Miss, then fresh | Stale data from races; forgotten invalidation |
| Read-through | Cache | (combine with a write pattern) | Depends | Coupling cache to data source |
| Write-through | — | Cache + DB synchronously | Hit, fresh | Slower writes; partial failure |
| Write-behind | — | Cache now, DB later | Hit, fresh | Losing acknowledged writes |
| Write-around | — | DB only | Miss | Extra misses on recent writes |

Read and write patterns are combined: "read-through + write-through" or "cache-aside + write-around" are
common pairings.

### 3.7 Refresh-ahead

A variant worth knowing: the cache refreshes an entry *before* it expires if it has been accessed recently,
so hot keys never take a miss. Caffeine's `refreshAfterWrite` does this. It trades some wasted refreshes for
removing the latency spike at expiry (topic 12 uses a probabilistic version of the same idea).

---

## 4. Eviction policies

A cache has finite memory. When it is full and a new item arrives, something must go. The **eviction policy**
decides what. The goal is to evict the item least likely to be needed again; since we cannot see the future,
every policy uses the past as a predictor.

### 4.1 The policies

- **FIFO (first in, first out)**: evict the oldest inserted item regardless of use. Trivial to implement (a
  queue), but evicts popular items just because they are old.
- **LRU (least recently used)**: evict the item whose last access is oldest. Bets on temporal locality.
  The default almost everywhere.
- **LFU (least frequently used)**: evict the item with the fewest accesses. Bets on long-term popularity.
  Weaknesses: needs counters; an item that was hugely popular yesterday can squat forever unless counts
  decay over time; new items start with low counts and get evicted before they can prove themselves.
- **TTL (time to live)**: each item expires after a fixed time regardless of space. Strictly speaking this
  is a *freshness* policy rather than a *capacity* policy: it bounds staleness. Most caches combine TTL with
  one of the capacity policies.
- **Random**: evict a random item. Surprisingly competitive, and cheap. Used as a building block for
  sampled approximations.

| Policy | Good at | Bad at |
|--------|---------|--------|
| FIFO | Simplicity | Keeping hot items |
| LRU | Recency-driven workloads | **Scans**: one big sequential read flushes the whole hot set |
| LFU | Stable popularity | Changing popularity; new items |
| TTL | Bounding staleness | Managing memory |

### 4.2 How LRU is implemented: hash map + doubly linked list

The textbook O(1) LRU combines two structures:

- a **hash map** from key to a node, for O(1) lookup;
- a **doubly linked list** of nodes ordered by recency, most recent at the head.

```
 hash map                 doubly linked list (most recent → least recent)

 "a" ──────────────────>  head ─> [a] <──> [c] <──> [b] <─ tail
 "c" ──────────────────────────────────────^         ^
 "b" ────────────────────────────────────────────────┘

 Each map entry points directly at its node; evictions take the tail.
```

Operations:
- **get(k)**: look up the node in the map (O(1)); unlink it from its current position and move it to the
  head (O(1), because the list is doubly linked so you have both neighbors).
- **put(k, v)**: if present, update and move to head. Otherwise create a node at the head and insert into
  the map. If over capacity, remove the tail node (O(1)) and delete its key from the map (the node stores its
  key for exactly this reason).

Walkthrough, capacity 3:

```
put a      list: a
put b      list: b a
put c      list: c b a
get a      list: a c b         (a moves to head)
put d      list: d a c         (b was tail → evicted)
get b      miss
```

In Java, `LinkedHashMap` with `accessOrder=true` and an overridden `removeEldestEntry` is exactly this.

**Why real systems approximate.** Exact LRU requires updating a shared linked list on *every read*. Under
concurrency, that list becomes a point of contention (every reader takes a lock to move a node), and each
node costs two extra pointers of memory. So production caches approximate:

- **Redis** does not keep a list. Each object stores a 24-bit "last access" clock. When it needs to evict,
  it samples a handful of keys (`maxmemory-samples`, default 5) and evicts the oldest among them, keeping a
  small pool of good candidates between rounds. With 10 samples it is very close to true LRU, at a fraction
  of the cost. Redis's LFU mode uses an 8-bit logarithmic counter per key that decays over time.
- **CLOCK** (used by operating systems and many databases' buffer pools): items sit in a circular array with
  a "referenced" bit. Access sets the bit (cheap, no list moves). To evict, a "hand" sweeps the circle:
  if the bit is set, clear it and move on (second chance); if clear, evict.
- **Memcached** uses segmented LRU per slab class, and avoids bumping an item more than once per short
  interval to reduce lock traffic.

### 4.3 Scan resistance and smarter policies

Plain LRU's worst case is a **scan**: a batch job reads a million keys once each, and every one of them becomes
"most recent", pushing out the genuinely hot set. Solutions:

- **Segmented LRU / 2Q**: new items enter a probationary segment; only items accessed a second time get
  promoted to the protected segment. A scan only churns the probationary area. Postgres and InnoDB use
  related ideas for their buffer pools (InnoDB inserts new pages at the midpoint of its LRU list).
- **ARC (adaptive replacement cache)**: balances a recency list and a frequency list, adapting the split.
- **W-TinyLFU** (Caffeine): a small LRU window admits new items; an approximate frequency sketch (a
  count-min sketch, topic 50) decides whether a newcomer is more valuable than the eviction victim. It
  achieves near-optimal hit ratios on many real traces.

The general lesson: the eviction policy matters most when the cache is small relative to the working set.
If the hot set fits comfortably, any reasonable policy works.

---

## 5. Invalidation, staleness and consistency

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

### 5.1 The fundamental problem

The moment data is written to the database, every cached copy of it is stale. You have three broad options
for dealing with that:

1. **Expire it (TTL)**: accept staleness up to T seconds. Simple and robust: even if every other mechanism
   fails, the error heals itself within T. The cost is choosing T: short TTLs mean more misses; long TTLs
   mean more staleness.
2. **Invalidate it explicitly**: on write, delete (or update) the cached entry. Fresh data almost immediately,
   but every write path must remember to do it, and races exist (section 5.3).
3. **Never mutate; version the key**: put a version in the key (`product:42:v17`). A write creates a new
   version; old entries are simply never read again and age out. Clean for immutable or versioned data,
   and the same trick CDNs use with fingerprinted asset URLs (topic 14).

In practice you combine them: explicit invalidation for freshness, plus a TTL as a safety net so that
any missed invalidation or race heals eventually. **Always set a TTL** unless you have a very good reason.

### 5.2 Staleness is a consistency choice

A cache is a replica of the data, and everything you will learn about replication lag (topic 23) and
consistency models (topic 28) applies to it. Questions to ask per piece of data:

- How stale can it be? A product description: minutes. Stock count on the product page: seconds (and the
  checkout re-checks the database anyway). An account balance used to authorize a payment: zero, so do not
  serve it from cache.
- Does the user who made a change need to see it immediately (**read-your-writes**)? A user who edits their
  profile and sees the old name thinks the save failed. A common trick: after a write, bypass or update the
  cache for that user for a short time, or read from the primary.
- Can different users see different versions at the same time? With local caches on 50 servers, yes, and
  a user bouncing between servers can see data go "back in time" (a monotonic-reads violation).

### 5.3 The race that leaves stale data forever (until TTL)

Cache-aside with delete-on-write still has a race between a reader filling the cache and a writer
invalidating it:

```
 time  Reader R                          Writer W
  t1   cache.get(x) → miss
  t2   db.read(x) → old value (v1)
  t3                                     db.update(x = v2)
  t4                                     cache.delete(x)      (nothing to delete)
  t5   cache.set(x = v1)                 
       → cache holds v1, DB holds v2. Stale until TTL expires.
```

The reader read before the write but populated the cache after the invalidation. It is less likely than the
update race (the read must be slow and straddle the whole write), but at scale it happens. Mitigations:

- **TTL** bounds the damage. This is the main reason TTLs are non-negotiable.
- **Leases** (from Facebook's Memcache paper): on a miss, the cache hands the reader a lease token. A
  delete invalidates outstanding leases, so the reader's later `set` with a stale token is rejected.
- **Versioned / conditional sets**: store a version number with the value and only overwrite if the new
  version is higher (Redis Lua script or `WATCH`/`MULTI`, Memcached `cas`).
- **Delayed double delete**: delete, then delete again after a short delay longer than a typical read. Crude
  but common.
- **Invalidate from the database's change stream**: rather than application code deleting keys, a consumer
  reads the database's replication log (change data capture, topic 39) and invalidates cache keys for every
  committed change. This removes the "forgot to invalidate on this code path" class of bugs and orders
  invalidations by commit order.

### 5.4 Other invalidation pitfalls

- **Dual-write failure**: the DB update succeeds but the cache delete fails (network blip). Retry the delete,
  or rely on CDC, or fall back on TTL.
- **Invalidating derived data**: if `user:42` changes, which cached objects embed it? A rendered page, a
  search result, a "friends of" list... Tracking dependencies is hard; often you choose shorter TTLs for
  derived data or use tag-based invalidation (store a set of keys per tag and delete them together).
- **Replica lag**: if the reader repopulates the cache from a read replica that has not yet received the
  write, it caches old data right after you invalidated. Read from the primary on misses that follow a
  recent write, or delay invalidation until replicas have caught up.
- **Invalidating many local caches**: with in-process caches on every server, a delete must be broadcast
  (Redis pub/sub, a message bus) or you accept TTL-bounded staleness.

---

## 6. Redis internals

Redis is an in-memory data structure server. Understanding a few internals explains both its speed and its
sharp edges.

### 6.1 The single-threaded event loop

Redis executes commands on **one main thread**. It uses I/O multiplexing (`epoll` on Linux, `kqueue` on BSD)
to watch thousands of client sockets at once:

```
loop forever:
    ready = epoll_wait(all client sockets, timers)
    for each ready socket:
        read bytes, parse complete commands
        execute command against in-memory data   (microseconds)
        queue reply
    write replies
    run timers (expire keys, background bookkeeping)
```

Why one thread is enough, and even an advantage:

- Commands operate on RAM and mostly take microseconds; the bottleneck is usually network and system calls,
  not CPU. A single core can serve on the order of 100k+ simple operations per second, more with pipelining.
- **Every command is atomic** with no locks: no other command runs in between. `INCR`, `LPUSH`, `SETNX`
  are safe under concurrency for free. `MULTI/EXEC` transactions and Lua scripts also run without
  interleaving.
- No lock contention, no context switches, simple and predictable code.

The flip side: **one slow command blocks everyone**. `KEYS *` on ten million keys, `SMEMBERS` on a huge set,
`DEL` of a giant hash, or a long Lua script freezes every other client for its duration. Hence `SCAN`
instead of `KEYS`, `UNLINK` (frees memory in a background thread) instead of `DEL` for big values, and
watching the slow log. Since Redis 6, network reads/writes can use I/O threads, but command execution
remains single-threaded. To use many cores you run many Redis processes (shards, as in Redis Cluster,
which partitions keys into 16,384 hash slots, related to topic 13).

### 6.2 Data structures

Redis is valued less as "a cache" than as shared data structures with server-side operations:

| Type | Typical uses | Notes |
|------|--------------|-------|
| String | Cached blobs, counters (`INCR`), flags, locks (`SET key val NX PX 30000`) | Binary-safe, up to 512 MB |
| Hash | Objects with fields (`user:42` → name, email) | Update one field without rewriting the whole value |
| List | Queues, recent-items lists | Fast push/pop at the ends; `BLPOP` for blocking consumers |
| Set | Tags, unique visitors, membership | Union/intersection on the server |
| Sorted set (ZSET) | Leaderboards, priority queues, time-indexed data, sliding-window rate limiting (topic 45) | Hash table + skip list: O(log n) insert, range by score or rank |
| Stream | Append-only log with consumer groups | Kafka-like semantics in miniature (topic 37) |
| HyperLogLog, Bitmap, Geo | Approximate counts, compact flags, geo radius queries | Topics 50, 52 |

Small collections use compact encodings (listpack, intset) that store elements contiguously to save memory,
switching to full hash tables or skip lists once they grow past configurable thresholds.

Expiry: each key can have a TTL. Redis removes expired keys **lazily** (when accessed, it checks and deletes)
and **actively** (a periodic task samples keys with TTLs and deletes expired ones, repeating if many were
expired). So memory for expired keys is reclaimed gradually, not instantly.

When `maxmemory` is reached, `maxmemory-policy` decides: `noeviction` (writes fail, the default), `allkeys-lru`,
`allkeys-lfu`, `volatile-lru`/`volatile-lfu`/`volatile-ttl` (only keys with a TTL), or random variants.
For a pure cache, `allkeys-lru` or `allkeys-lfu` is usual.

### 6.3 Persistence: RDB vs AOF

Redis keeps everything in memory, but can persist to disk so a restart does not start empty.

**RDB (snapshots)**: periodically, Redis calls `fork()`. The child process writes a compact binary snapshot of
the whole dataset to disk while the parent keeps serving. The OS's **copy-on-write** means the child sees a
frozen view of memory; pages are only duplicated when the parent modifies them.

- Compact files, fast restarts, good for backups.
- You lose everything since the last snapshot (minutes of writes).
- `fork()` on a large heap can pause the parent for milliseconds to longer, and a write-heavy workload during
  the snapshot can nearly double memory usage via copy-on-write.

**AOF (append-only file)**: every write command is appended to a log file. On restart, Redis replays the log.
The `appendfsync` setting picks the durability/speed trade-off (fsync is explained in topic 17):

| `appendfsync` | Behavior | Loss on power failure |
|---------------|----------|-----------------------|
| `always` | fsync after every write | ~nothing, but slow |
| `everysec` (default) | fsync once per second in background | up to ~1 second |
| `no` | let the OS decide | possibly ~30 seconds |

The log grows forever, so Redis periodically **rewrites** it (again via fork) into the minimal set of
commands that recreate the current state. Modern Redis can write the rewritten AOF with an RDB preamble,
combining fast loading with the AOF tail.

| | RDB | AOF |
|--|-----|-----|
| Data loss window | Since last snapshot (minutes) | ~1 s with `everysec` |
| File size | Small | Larger |
| Restart speed | Fast | Slower (replay) |
| Runtime cost | Periodic fork | Continuous writes + periodic rewrite |

Many deployments use both. For a pure cache, persistence may be off entirely, since the data can be rebuilt
from the database (at the cost of a cold start, topic 12).

**Replication** is separate from persistence: a Redis primary streams commands to replicas asynchronously,
so a failover can lose recently acknowledged writes (topic 23). Redis Sentinel or Redis Cluster automate
failover. This is why Redis is best treated as a cache or as a store where losing a second of writes is
acceptable, unless you configure and verify otherwise.

### 6.4 Redis vs Memcached briefly

Memcached is multi-threaded, stores only opaque strings, has no persistence or replication, and uses a slab
allocator to avoid fragmentation. It is simpler and scales across cores in a single process. Redis offers
rich data structures, persistence, replication and scripting. For a plain key-value cache either works.

---

## 7. How it connects

- **Topic 01, 02**: caching exists because of the latency hierarchy and because averages and tails matter.
  A cache miss is often what produces tail latency.
- **Topic 12**: what happens when the cache misbehaves: stampedes, penetration, avalanches, hot keys, cold
  starts. Read it as the second half of this topic.
- **Topic 13**: how a distributed cache spreads keys over many nodes without remapping everything when a
  node joins or leaves.
- **Topic 14**: HTTP caching and CDNs apply the same ideas at the edge, with headers instead of code.
- **Topic 17**: the database buffer pool is itself a cache with write-behind semantics (dirty pages flushed
  later, protected by the WAL).
- **Topic 23, 28**: a cache is an asynchronously updated replica; staleness is replication lag, and the
  consistency guarantees you get are those of an eventually consistent system.
- **Topic 39**: change data capture is the robust way to drive cache invalidation.
- **Topic 45, 50**: Redis sorted sets for rate limiting; count-min sketches inside TinyLFU.

## Common misconceptions

- "A cache is just an optimization; if it fails we're only slower." → Once the backend is sized for the
  cached hit ratio, the cache is load-bearing. Losing it can take the database down (topic 12).
- "Update the cache on write so it's always fresh." → Updating on write has a worse race than deleting and
  wastes work for data that is not read. Delete and let the next read refill.
- "If I invalidate correctly I don't need TTLs." → Races, failed deletes and replica lag will leave stale
  entries. TTL is the backstop that makes the system self-healing.
- "LRU is always the best policy." → LRU collapses under scans and ignores frequency. Segmented LRU, ARC or
  W-TinyLFU do better on many real workloads; what matters most is whether the hot set fits.
- "Redis is single-threaded, so it's slow." → It is fast because it is single-threaded over in-memory data:
  no locks, atomic commands. The real risk is a single slow command blocking everyone.
- "Redis is a database, so my data is safe." → With default async replication and `everysec` AOF, a crash
  or failover can lose acknowledged writes.
- "Caching more is always better." → Caching low-reuse data evicts useful data, wastes memory and adds
  invalidation complexity. Cache what is hot, expensive and tolerant of staleness.

## Further reading (supplements)

- Nishtala et al., *Scaling Memcache at Facebook* (NSDI 2013): leases, regional invalidation, cold cluster warmup.
- Redis documentation: "Key eviction", "Redis persistence", "Redis keyspace notifications" and the latency/slow log pages.
- Kleppmann, *Designing Data-Intensive Applications*, Ch. 1 (load and performance) and Ch. 11 (derived data, keeping caches in sync via change streams).
- Megiddo & Modha, *ARC: A Self-Tuning, Low Overhead Replacement Cache* (FAST 2003).
- Einziger, Friedman & Manes, *TinyLFU: A Highly Efficient Cache Admission Policy* (and Ben Manes's Caffeine wiki on W-TinyLFU).
- AWS whitepaper / docs: "Caching strategies" for ElastiCache (cache-aside vs write-through in practice).

---

## Questions

1. A cache holds 2% of the products in a catalog yet serves 90% of product-page reads. What property of the
   workload makes this possible, and what kind of workload would make the same cache nearly useless?
2. Your database is sized to handle 5% of read traffic because the cache hit ratio is normally 95%. A deploy
   accidentally changes the cache key format. What happens, and what does this tell you about the role of
   the cache?
3. In cache-aside, why do most teams delete the cache entry on write rather than writing the new value into
   the cache? Walk through an interleaving that shows the problem with updating.
4. Describe the read/write interleaving that leaves a stale value in the cache even with delete-on-write.
   Which mechanisms limit or prevent it?
5. When is write-behind an acceptable choice, and what exactly do you risk? Name a non-cache system you know
   that uses write-behind internally.
6. A nightly report job reads every row of a large table through the same LRU cache that serves user traffic.
   What happens to the hit ratio the next morning, and what policy or design change fixes it?
7. Explain how an O(1) LRU cache works. Why does the list need to be doubly linked, and why does each node
   store its own key?
8. Why does Redis approximate LRU by sampling instead of keeping an exact LRU list?
9. You run 40 app servers, each with an in-process cache in front of Redis. A product price changes. What do
   different users see over the next minute, and how could you reduce the inconsistency?
10. A user updates their display name, refreshes, and sees the old name. List two plausible causes involving
    caching and one fix for each.
11. Redis is single-threaded. Why is it nonetheless fast, and what operational mistake turns that design into
    an outage?
12. Compare RDB and AOF persistence. If you can tolerate losing at most one second of writes, which do you
    choose and with which setting? What does `fork()` cost you?
13. Postgres already keeps hot pages in memory. Under what circumstances does an application cache in front
    of it still help significantly?
14. Why should a cache entry almost always have a TTL even when you have explicit invalidation?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Skewed (Zipf-like) access: a small head of popular items receives most requests, so caching that head
captures most traffic. A uniform-random workload over a huge keyspace (every item equally likely, little
reuse) would give a hit ratio close to the cache's fraction of the data, about 2%.

**2.** After the deploy every lookup uses a new key, so every lookup misses: the hit ratio drops to near 0%
and the database suddenly receives ~20× the load it was sized for. It likely saturates, latency explodes and
requests time out, possibly cascading. The cache was load-bearing: it was a capacity component, not an
optional speed-up. Key-format changes should be rolled out gradually or with warming (topic 12).

**3.** Updating requires computing the cached form on every write (wasted if not read) and races between
writers can leave the older value in the cache: A writes DB=1, B writes DB=2, B sets cache=2, A sets cache=1.
The DB says 2, the cache says 1 until TTL. With delete, the same interleaving just produces two deletes and a
later miss that loads the correct value.

**4.** A reader misses, reads the old value from the DB, then stalls; a writer updates the DB and deletes the
(absent) cache key; the reader then sets the old value into the cache. Stale until TTL. Mitigations: TTL to
bound it; leases (a delete invalidates the reader's token, so its set is rejected); version-checked sets;
delayed double delete; CDC-driven invalidation that orders deletes after commits.

**5.** When losing the most recent writes is tolerable (view counts, metrics, non-critical aggregates) or
when the cache layer is itself durable/replicated. The risk is that an acknowledged write never reaches the
database if the cache crashes before flushing, plus anyone reading the DB directly sees stale data. Examples:
the OS page cache, a CPU cache, a database buffer pool with dirty pages (protected by the WAL).

**6.** The scan touches each row once, making all of them "most recent" and evicting the hot set. Next morning
users miss en masse until the cache refills. Fixes: scan-resistant policies (segmented LRU/2Q, ARC,
W-TinyLFU, which require a second access before protecting an item); or route batch reads around the cache
(write-around/bypass), or to a replica.

**7.** A hash map maps key → list node; a doubly linked list orders nodes by recency. Get: find via map,
unlink, move to head. Put: insert at head; if over capacity, remove the tail and delete its key from the map.
Doubly linked so a node can be unlinked in O(1) given only a pointer to it (you need its predecessor). The node
stores its key so that when you evict the tail you know which map entry to delete.

**8.** Exact LRU requires updating a shared structure on every read (contention, extra pointers per key,
memory overhead across millions of keys). Sampling a few keys and evicting the one with the oldest access
clock is cheap and, with a candidate pool, gets very close to true LRU hit ratios.

**9.** Servers whose local entry has not expired still serve the old price; others have refreshed. A user
whose requests land on different servers might see the new price, then the old one again. Reduce by: short
local TTLs (seconds), broadcasting invalidations (pub/sub) to all servers, using only the shared Redis cache
for volatile data, or sticky routing for read-your-writes. Checkout should always use the authoritative price.

**10.** (a) The write path updated the DB but the cache delete failed or the code path forgot to invalidate →
fix with retried deletes or CDC-based invalidation. (b) The cache was refilled from a lagging read replica
right after invalidation → read from the primary after a recent write, or delay invalidation. (c) A local
in-process cache or browser/CDN cache still holds the old version → short TTLs, bypass the cache for the
author for a while, or versioned keys.

**11.** It operates on in-memory data with microsecond commands, uses epoll to multiplex thousands of
connections, and avoids locks and context switches; the bottleneck is usually the network. Every command is
atomic. The mistake: running an O(N) command on a large structure (`KEYS *`, `SMEMBERS` on a huge set, `DEL`
of a huge hash, a long Lua script), which blocks all other clients for its duration and looks like an outage.

**12.** RDB: periodic forked snapshots, compact, fast restarts, but lose everything since the last snapshot.
AOF: logs every write; with `appendfsync everysec` you lose at most about a second; bigger files and slower
replay, plus periodic rewrites. For ≤1 s loss choose AOF with `everysec` (optionally plus RDB for backups).
`fork()` can pause the main thread on large heaps and copy-on-write can grow memory significantly under heavy
writes during the snapshot or rewrite.

**13.** When the expensive part is not disk I/O but query work: joins, aggregations, sorting, serialization,
many round trips to assemble an object, or connection limits on the database. An application cache stores the
final computed result, saving CPU and connections on the database and network round trips, which the buffer
pool cannot do. It also helps when the read volume simply exceeds what the database CPU can serve.

**14.** Explicit invalidation can fail: races (question 4), failed or forgotten deletes, replica lag, bugs in a
new code path. A TTL guarantees any such error heals within a bounded time, and also limits memory spent on
entries that are no longer read.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What two properties of workloads make caching effective?
**A:** Locality (temporal and spatial) and skewed, Zipf-like access where a small hot set gets most traffic.

**Q:** Define hit ratio.
**A:** Hits / (hits + misses): the fraction of requests served from the cache.

**Q:** Why does going from 99% to 95% hit ratio matter so much?
**A:** Backend load goes from 1% to 5% of traffic: a 5× increase.

**Q:** What is a "load-bearing" cache?
**A:** One the backend cannot survive without, because capacity was sized assuming the cache's hit ratio.

**Q:** Name six places a cache can live in a web request path.
**A:** Browser, CDN, reverse proxy, in-process app cache, distributed cache (Redis/Memcached), database buffer pool (plus OS page cache).

**Q:** Main downside of an in-process cache across many servers?
**A:** Each server has its own copy, so data is duplicated and servers disagree until entries expire or are invalidated.

**Q:** Cache-aside in one sentence?
**A:** The app checks the cache, on miss reads the DB and populates the cache; on write it updates the DB and deletes the cache entry.

**Q:** Read-through vs cache-aside?
**A:** In read-through the cache itself loads missing data via a configured loader; in cache-aside the application does.

**Q:** Write-through?
**A:** Writes go to cache and DB synchronously before acknowledging; fresh cache, slower writes.

**Q:** Write-behind and its risk?
**A:** Write to cache, acknowledge, flush to DB asynchronously; risk is losing acknowledged writes if the cache dies.

**Q:** Write-around?
**A:** Writes bypass the cache and go to the DB; cache filled only on reads; avoids polluting the cache with write-only data.

**Q:** Why delete instead of update the cache on write?
**A:** Concurrent writers can leave the older value in the cache, and recomputing on every write wastes work.

**Q:** What race leaves stale data with delete-on-write?
**A:** A reader reads the old DB value, a writer updates and deletes, then the reader sets the old value into the cache.

**Q:** Name three mitigations for that race.
**A:** TTL, leases, version-checked sets (also delayed double delete, CDC-driven invalidation).

**Q:** LRU's classic weakness?
**A:** Scans: a one-time sequential read evicts the entire hot set.

**Q:** LFU's classic weaknesses?
**A:** Stale popularity squats without decay; new items get evicted before building up counts.

**Q:** O(1) LRU data structures?
**A:** Hash map (key → node) plus doubly linked list ordered by recency; evict from the tail.

**Q:** How does Redis approximate LRU?
**A:** Per-key access clock; on eviction, sample a few keys (default 5) and evict the oldest, keeping a candidate pool.

**Q:** What is the CLOCK algorithm?
**A:** Circular buffer with a referenced bit per item; a sweeping hand clears set bits and evicts the first item whose bit is clear.

**Q:** What does W-TinyLFU add?
**A:** A frequency sketch that decides whether a new item is worth admitting over the eviction victim, plus a small LRU window.

**Q:** Three ways to handle staleness?
**A:** TTL expiry, explicit invalidation, versioned (immutable) keys.

**Q:** Why is Redis single-threaded execution an advantage?
**A:** No locks, no context switches, and every command is atomic.

**Q:** Biggest operational risk of single-threaded Redis?
**A:** One slow O(N) command (`KEYS *`, huge `DEL`, long Lua) blocks every client.

**Q:** How does Redis expire keys?
**A:** Lazily on access plus an active periodic sampling task.

**Q:** RDB in one line?
**A:** Forked, copy-on-write point-in-time snapshots; compact and fast to load, but lose writes since the last snapshot.

**Q:** AOF in one line?
**A:** Log of every write command replayed on restart; `everysec` fsync loses at most about one second.

**Q:** Redis vs Memcached?
**A:** Redis: rich data structures, persistence, replication, single-threaded execution. Memcached: plain strings, multi-threaded, no persistence.

**Q:** What sorted-set implementation does Redis use?
**A:** A hash table plus a skip list, giving O(log n) inserts and range queries by score.
