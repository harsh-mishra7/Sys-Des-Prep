# 49 — Unique ID Generation

> Part 9 — Specialized Building Blocks · Prerequisites: topics 17, 18, 23, 25, 26, 27

Almost every record a system stores needs an identifier: orders, users, messages, uploaded files, log
events. On a single database this is a solved problem: `SERIAL` / `AUTO_INCREMENT` hands out 1, 2, 3, ...
and you never think about it again. The moment you have more than one machine creating records, the
problem comes back. Who hands out the next number? What if two machines hand out the same one? What if
the machine that hands out numbers is down? And the ID you choose turns out to have consequences far
beyond uniqueness: it decides how fast your B-tree inserts are, whether you can sort by it, whether it
leaks your business volume to competitors, and whether a clock jump can corrupt your data.

This topic is about the design space of ID generation: what properties you might want, the main schemes
(auto-increment, ticket servers, UUIDs, Snowflake-style IDs, ULID/UUIDv7), and what each one costs.

By the end of this topic you should be able to:
1. Explain why a single auto-increment counter becomes a bottleneck and a single point of failure in a distributed system, and why naive fixes (per-shard counters) produce collisions.
2. Describe UUID v4 and UUID v7 at the bit level and explain why v4 hurts B-tree index performance while v7 does not.
3. Explain the Snowflake ID layout (timestamp, worker ID, sequence), how it generates IDs without coordination, and what happens when the clock moves backward.
4. Describe ticket servers and block (range) allocation, and when coordination-based schemes are the right choice.
5. Choose an ID scheme given requirements on size, sortability, coordination, privacy and clock dependence.

---

## 1. What we want from an ID

Before comparing schemes, list the properties an ID can have. No scheme has all of them; every design is
a choice of which ones you give up.

| Property | Meaning | Why you might care |
|---|---|---|
| **Unique** | No two entities ever get the same ID | Non-negotiable. A collision means two records overwrite each other or a foreign key points at the wrong thing. |
| **Decentralized** | Any node can generate an ID without talking to anyone | Availability and latency: no network round trip, no single point of failure (topic 26). |
| **Sortable by time** | Later IDs compare greater than earlier ones (roughly) | "Newest first" pagination, range scans by time, index locality. |
| **Compact** | Small number of bits | 64-bit integers fit in a machine register and in every language's native integer type; smaller indexes. |
| **Dense / sequential** | IDs are consecutive with no gaps | Rarely truly needed (invoice numbers sometimes legally require it). |
| **Unpredictable** | Can't guess other valid IDs | Prevents enumeration attacks (`/orders/1001`, `/orders/1002`...) and hides business volume. |
| **Clock-independent** | Correctness doesn't depend on wall-clock time | Clocks drift and jump (topic 27). |

Two of these pull hard against each other:

- **Sortable vs unpredictable.** If an ID is ordered by time, it reveals *when* it was created, and
  neighboring IDs are guessable. If it's random, it tells nothing but sorts randomly.
- **Decentralized vs dense.** Gap-free numbering requires a single authority deciding "the next one is
  N+1". Any decentralized scheme leaves gaps.

Keep the distinction between *uniqueness* (correctness) and *every other property* (performance,
convenience, privacy) in mind. You can trade away the others; you can't trade away uniqueness.

---

## 2. Why auto-increment breaks

### 2.1 How auto-increment works on one machine

A single database keeps a counter (a sequence object in Postgres, an in-memory counter per table in
MySQL InnoDB). Each insert takes the next value. Because there is exactly one counter and it lives in one
process, there's no question of who decides. Values are compact (64-bit), monotonically increasing, and
inserts always go to the right-most leaf of the primary key B-tree (topic 17), which is the cheapest
possible insert pattern.

Note that even on one machine, auto-increment is **not gap-free**: a transaction that takes value 42 and
then rolls back does not give 42 back. Sequences are deliberately non-transactional so concurrent inserts
don't serialize on them. If you need gap-free numbers (some invoice regulations), you need a separate,
locked counter row, and you accept that it serializes all inserts.

### 2.2 What goes wrong with many machines

Now partition the table across four shards (topic 25), each with its own auto-increment:

```
shard A: 1, 2, 3, 4 ...
shard B: 1, 2, 3, 4 ...      <-- collision: "order 3" exists on every shard
shard C: 1, 2, 3, 4 ...
shard D: 1, 2, 3, 4 ...
```

IDs are unique only *within* a shard. That breaks:
- Any global lookup by ID ("fetch order 3": which one?).
- Moving rows between shards during rebalancing (topic 25): a moved row collides with an existing row.
- Merging data into a warehouse (topic 20) or a search index (topic 51).
- Clients that cache by ID.

The obvious alternatives each fail in a different way:

1. **One central counter that all shards call.** Unique, but every insert now makes a network round trip
   to one machine. That machine is a single point of failure and a throughput ceiling. If it fails over
   asynchronously (topic 23) the new leader may not have seen the last values it handed out, and it
   reissues them: duplicates.
2. **Auto-increment with offsets.** MySQL's `auto_increment_increment` / `auto_increment_offset`: shard A
   hands out 1, 5, 9...; shard B 2, 6, 10...; and so on. This works, but the number of shards is baked
   into every ID. Adding a fifth shard means changing the step on every shard, carefully, without overlap.
   Used historically in multi-leader MySQL setups (topic 24) for exactly this reason.
3. **Use a timestamp as the ID.** Two inserts in the same millisecond collide. Two machines with the same
   clock collide. Clocks go backwards (topic 27).

So we need a scheme that is unique *by construction* without a single authority on the hot path. There
are two broad families:

- **Coordination-based, but off the hot path**: ticket servers and block allocation (section 3).
- **Coordination-free**: random IDs (UUID v4, section 4) and structured IDs that embed a unique node
  identifier (Snowflake, UUID v7, sections 5–6).

---

## 3. Ticket servers and block allocation

### 3.1 Ticket servers

A ticket server is a dedicated database whose only job is to hand out numbers. The well-known example is
Flickr's ticket server (2010): a MySQL table with a single row, using `REPLACE INTO` to bump an
auto-increment and `LAST_INSERT_ID()` to read it.

```
app server  ── "next id please" ──►  ticket DB (single counter)
            ◄──────── 8812734 ─────
```

To avoid one machine being a SPOF they ran **two** ticket servers, one handing out odd numbers and one
handing out even numbers (the offset trick from 2.2), with clients round-robining between them. If one
dies, the other keeps going alone. IDs are no longer strictly ordered across the two, but they're
roughly ordered and globally unique.

Trade-off: an extra network hop per ID and an operational component to keep alive. In exchange: compact
64-bit, dense-ish, roughly ordered IDs with no clock dependence.

### 3.2 Block (range / hi-lo) allocation

The round trip per ID can be removed by **leasing ranges**. Instead of asking for one number, an app
server asks for a block of, say, 10,000:

```
app server 1:  "give me a block" ──► counter service: block = [1, 10000]
app server 2:  "give me a block" ──► counter service: block = [10001, 20000]

app server 1 now hands out 1, 2, 3 ... 10000 locally, with no network call.
When it runs out, it asks for [20001, 30000].
```

The counter service only needs to durably record "the next free block starts at X". It can live in any
strongly consistent store: a row updated with compare-and-set (topic 22), or a key in etcd/ZooKeeper
(topic 31). This is the "hi/lo" algorithm in ORMs like Hibernate, and it's how many systems that need
compact integer IDs scale.

Properties:
- **Unique**: as long as the block allocation itself is linearizable (topic 28). If the counter store
  loses an acknowledged update (e.g. async failover), two servers can be given the same block. This is
  the one place you must use a consensus-backed or synchronously replicated store.
- **Gaps**: when a server crashes, the unused rest of its block is lost. That's fine; IDs aren't dense.
- **Ordering**: only within a server. Server 1 may still be handing out 9,000-range IDs after server 2
  started on 10,001+. So IDs are not time-ordered across servers.
- **Availability**: the counter service can be down for as long as servers have unused blocks left.
  Bigger blocks = more tolerance but bigger gaps.

---

## 4. UUIDs

A **UUID** (Universally Unique Identifier, RFC 4122, updated by RFC 9562 in 2024) is a 128-bit value,
usually written as 32 hex digits in 5 groups:

```
550e8400-e29b-41d4-a716-446655440000
              ^    ^
              |    variant bits (here "10xx" -> RFC variant)
              version nibble (here 4)
```

There are several versions; the version number is stored in the ID itself. The ones that matter:

| Version | Contents | Notes |
|---|---|---|
| v1 | 60-bit timestamp + clock sequence + 48-bit MAC address | Leaks the machine's MAC and creation time; timestamp bytes stored in a non-sortable order |
| v3 / v5 | Hash (MD5 / SHA-1) of a namespace + name | Deterministic: same input → same UUID. Useful for idempotent IDs from natural keys (topic 38) |
| **v4** | 122 random bits | The default almost everywhere |
| v6 | v1 with timestamp bytes reordered to sort | Mostly superseded by v7 |
| **v7** | 48-bit Unix millisecond timestamp + 74 random bits | Time-sortable; the modern recommendation |

### 4.1 UUID v4: pure randomness

122 of the 128 bits are random (6 are fixed version/variant bits). Any node can generate one with no
coordination and no clock. Is it actually unique? It's probabilistic, but the probability is absurd.
Using the birthday bound, the chance of *any* collision after generating `n` IDs from a space of
`2^122` is about `n² / 2^123`. To reach a 50% chance of one collision you'd need about `2^61` ≈ 2.3 ×
10^18 IDs. Generating a billion per second, that takes ~70 years. Hardware faults are far more likely
than a v4 collision.

The real-world caveat is the **random number generator**. Collisions have happened in practice, not
because the math failed but because of bad randomness: VMs cloned with the same PRNG state, forked
processes sharing a seed, a non-cryptographic PRNG seeded from the time. Use the OS CSPRNG
(`/dev/urandom`, `getrandom()`), which every mainstream UUID library does.

### 4.2 Why random IDs hurt B-tree indexes

v4 is unique and decentralized, but it has a performance cost that becomes significant at scale. It's
about where each insert lands in the primary key index (topic 17, topic 18).

With a monotonically increasing key, every new key is larger than all existing ones, so it goes in the
right-most leaf page:

```
Sequential keys:                      Random keys (UUID v4):

[ ...  ][ ...  ][ ...  ][ new→ ]      [ new ][ ...  ][ new ][ ...  ][ new ]
                          ^hot page       ^        ^         ^
                                       inserts scatter across ALL leaf pages
```

Consequences of random insertion:

1. **Working set = entire index.** With sequential keys, only the right-most page (and its path from the
   root) needs to be in the buffer pool (topic 17) for inserts. With random keys, every leaf is equally
   likely to be touched, so once the index exceeds RAM, many inserts need a disk read first. Insert
   throughput can fall by an order of magnitude when the table outgrows memory.
2. **Page splits everywhere, and half-empty pages.** Inserting into the middle of a full page splits it
   into two half-full pages. Steady-state random insertion leaves pages around 69% full on average
   (versus near 100% for append-only), so the index is bigger and fewer rows fit in cache.
3. **More WAL and more dirty pages.** Each insert dirties a different page; with full-page writes after
   checkpoints (Postgres), that means far more WAL volume (topic 17).
4. **Clustered index makes it worse.** In InnoDB and SQL Server the table *is* the primary key B-tree
   (clustered index, topic 18), so the whole row is scattered, not just an index entry. And every
   secondary index stores the primary key, so a 16-byte PK makes every secondary index larger.
5. **No locality for recent data.** Rows created together are spread across the disk, so "the last
   hour's orders" touches random pages.

This is not a problem for LSM-tree stores (topic 19) in the same way: an LSM buffers writes in the
memtable and writes sorted runs sequentially regardless of key order. Random keys there mostly cost you
range-scan locality, not insert throughput. Random keys are also *good* for hash-partitioned stores,
because they spread load evenly (topic 25).

### 4.3 UUID v7: time-ordered UUIDs

UUID v7 (RFC 9562) keeps the 128-bit format and fixes the locality problem by putting the timestamp
first:

```
 0                   1                   2                   3
|<------------- 48 bits: unix_ts_ms ------------->|ver |<- 12 bits rand_a ->|
|var|<---------------------- 62 bits rand_b ---------------------------->|

  48 bits  Unix epoch milliseconds (good until the year 10889)
   4 bits  version = 0111
  12 bits  random (or a sub-millisecond counter, implementation's choice)
   2 bits  variant
  62 bits  random
```

Because the most significant bits are the timestamp, v7 IDs created later compare greater (byte-wise
and as integers). Inserts go near the right edge of the B-tree, just like auto-increment. And within a
millisecond, the 74 random bits give collision resistance without coordination.

Ordering within the same millisecond: if a single generator creates many IDs per millisecond, pure
random bits would sort those randomly. RFC 9562 allows implementations to use part of `rand_a`/`rand_b`
as a counter that increments within the millisecond, so IDs from one generator are strictly monotonic.
Across generators, same-millisecond IDs are ordered arbitrarily, which is fine: they're "roughly" sorted.

**ULID** (Universally Unique Lexicographically Sortable Identifier) is the same idea that predates v7:
48-bit millisecond timestamp + 80 random bits, encoded as 26 Crockford base32 characters
(`01ARZ3NDEKTSV4RRFFQ69G5FAV`) so the string form sorts correctly. v7 is the standardized equivalent and
fits in existing `uuid` column types.

Costs of v7 compared with v4:
- It **reveals creation time** to anyone who sees the ID (millisecond precision). Usually harmless,
  sometimes not (e.g. revealing when an account was created).
- It **depends on the clock**, but only weakly. If the clock jumps backward, IDs created after the jump
  sort before earlier ones; uniqueness still holds because of the random bits. Order is lost; uniqueness
  isn't. Contrast this with Snowflake (section 5), where a backward jump can cause *duplicates*.
- In a range- or ordered-partitioned store (topic 25), time-ordered keys put all current writes on one
  partition: the same hotspot as auto-increment. Time-ordering is good for one B-tree and bad for
  range-sharded clusters.

---

## 5. Snowflake-style IDs

Twitter published **Snowflake** in 2010 to generate IDs for tweets: 64-bit (fits a `BIGINT`), roughly
time-ordered, generated without coordination at very high rates. The layout:

```
 1 bit   41 bits                         10 bits        12 bits
+---+-----------------------------+--------------+-------------+
| 0 |  timestamp (ms since epoch) |  worker ID   |  sequence   |
+---+-----------------------------+--------------+-------------+
 sign  ~69 years from custom epoch   1024 workers   4096 IDs/ms/worker
```

- **Sign bit** is 0 so the value is positive in languages with signed 64-bit integers.
- **Timestamp** is milliseconds since a *custom epoch* (e.g. the system's launch date), not 1970, so 41
  bits last ~69 years from launch.
- **Worker ID** (sometimes split into 5 bits datacenter + 5 bits machine) is unique per generator process.
- **Sequence** counts IDs generated in the current millisecond by this worker; it resets to 0 each new
  millisecond.

### 5.1 The generation algorithm

```
state: last_ts = -1, seq = 0, worker_id (assigned once)

next_id():
    ts = current_time_ms()
    if ts < last_ts:
        # clock moved backwards (see 5.3)
        refuse / wait until ts >= last_ts
    if ts == last_ts:
        seq = (seq + 1) & 0xFFF          # 12 bits
        if seq == 0:                     # 4096 IDs used this ms
            ts = wait_until_next_ms(last_ts)
    else:
        seq = 0
    last_ts = ts
    return (ts - EPOCH) << 22 | worker_id << 12 | seq
```

Why it's unique: two IDs can only be equal if they have the same timestamp, same worker, and same
sequence. Different workers differ in the worker bits. The same worker never reuses (timestamp,
sequence) because it never goes back in time (the `ts < last_ts` check) and the sequence increments
within a millisecond. **Uniqueness is the product of two assumptions: worker IDs are unique, and each
worker's clock never goes backward.**

Throughput: 4096 IDs per ms per worker = ~4 million per second per worker. If the sequence overflows,
the generator spins until the next millisecond.

### 5.2 Assigning worker IDs

The worker ID is the one piece that needs coordination, but only at startup:

- Static config (worker ID in a config file or environment variable). Simple, but a copy-paste mistake
  gives two machines the same ID → silent duplicates.
- Derived from something unique: the last bits of the IP address, a Kubernetes StatefulSet ordinal.
- Leased from a coordination service (topic 31): on startup, create an ephemeral node
  `/snowflake/workers/017` in ZooKeeper or take a lease in etcd. If the process dies, the lease expires
  and the ID can be reused. The original Snowflake used ZooKeeper this way.

The lease approach has the usual lease hazard (topic 31): a process that pauses (GC, VM suspension,
topic 26) past its lease, while another process takes the same worker ID, and then resumes generating
IDs. Both now generate with the same worker ID. Mitigations: stop generating when the lease can't be
confirmed as still held, and add a safety delay before reusing a released worker ID.

### 5.3 The clock problem

Snowflake depends on wall-clock time (topic 27). Wall clocks can move backward: NTP step corrections,
manual changes, VM migration, leap-second handling. Suppose a worker issued IDs at `ts = 1000` with
sequence 0..50, then the clock steps back to `ts = 990`. If the generator naively continues, by the time
the clock reaches 1000 again it will issue `(1000, worker, 0)` again: a **duplicate ID**.

That's why the algorithm refuses to generate when `ts < last_ts`. Options:
- **Wait** until the clock catches up (fine for small jumps of a few ms).
- **Fail** with an error if the jump is large, and alert.
- **Use a logical "last_ts"**: keep issuing with `last_ts` and incrementing the sequence (borrowing from
  the future) until the real clock catches up. This keeps uniqueness at the cost of timestamp accuracy.

But the check only protects within one process lifetime. If the process **restarts** after a backward
jump, `last_ts` is gone. Some implementations persist the last timestamp periodically and refuse to start
until the clock exceeds it. Running NTP in slew mode (adjust gradually, never step) avoids most of this.

Note how UUID v7 behaves differently in the same situation: its random bits make a duplicate
astronomically unlikely even when the clock repeats. Snowflake has only 12 bits of non-time, non-worker
entropy, and they're a deterministic counter, so a repeated clock reading reproduces the same IDs.

### 5.4 What "roughly sorted" means

Snowflake IDs from different workers are ordered by timestamp first, but:
- Clocks on different workers differ by up to the clock skew (often single-digit ms with NTP, sometimes
  far worse, topic 27). An ID from worker A can be numerically smaller than an earlier ID from worker B.
- Within the same millisecond, workers are ordered by worker ID, not by actual time.

So "ORDER BY id" gives you approximately creation order, good for "newest first" feeds and pagination,
but **not** a correct happens-before order (topic 27). Don't use Snowflake IDs to decide which of two
concurrent writes came last.

### 5.5 Variants

- **Instagram (2012)** generated IDs *inside Postgres* with a PL/pgSQL function: 41 bits of time, 13 bits
  of logical shard ID, 10 bits from a per-shard sequence mod 1024. Because the shard ID is in the ID,
  you can find the shard from the ID alone, with no lookup table (topic 25 request routing).
- **Sonyflake**: 39 bits of time in 10 ms units (~174 years), 8 bits sequence, 16 bits machine ID. Fewer
  IDs per second per machine, more machines and longer lifetime.
- **Discord, Mastodon**, and many others use Snowflake-style 64-bit IDs.

Embedding the shard in the ID is a useful general trick: the ID then carries routing information. The
cost is that the row can never move to a different logical shard without changing its ID, so use
*logical* shards (many more than physical machines) and move logical shards between machines (topic 25).

---

## 6. Sortability, index locality and clock dependence

The plan's third bullet pulls together threads from the previous sections. They're worth stating
separately because they're the real reason ID choice matters.

### 6.1 Sortability

A time-sortable ID gives you, for free:
- **Keyset pagination**: `WHERE id < :last_seen ORDER BY id DESC LIMIT 20`. No `OFFSET` scanning, no
  separate `created_at` index needed.
- **Time-range queries via the ID**: you can compute the smallest Snowflake/v7 ID for a timestamp and
  range-scan the primary key.
- **Rough ordering of events** across services when you merge logs or streams.

Limitation: it's *creation time according to some machine's clock*, so it's approximate across machines.

### 6.2 Index locality

Locality means "things inserted together are stored together". Time-ordered keys give insertion
locality (good for B-trees, section 4.2) and temporal locality for reads ("recent" rows are adjacent and
hot pages stay cached, which matches typical access skew, topic 11).

But locality is a double-edged sword in a distributed store:

| Store type | Time-ordered keys | Random keys |
|---|---|---|
| Single B-tree (Postgres, MySQL) | Excellent: append to right edge | Poor: scattered inserts, splits, cache misses |
| Range-partitioned cluster (Bigtable, HBase, Spanner, CockroachDB) | Hotspot: all writes hit the last range | Even spread across ranges |
| Hash-partitioned cluster (Cassandra, DynamoDB) | Fine: hash spreads them anyway | Fine |
| LSM storage engine | Fine; nice for time-range scans | Fine for writes |

Range-partitioned systems often recommend *against* monotonically increasing keys for exactly this
reason: Spanner's documentation warns against them, and Bigtable users "salt" or reverse timestamp
prefixes. One common compromise: prefix the key with a small hash bucket (`bucket = hash(id) % 16`), so
writes spread over 16 ranges while each bucket stays internally ordered.

### 6.3 Clock dependence

| Scheme | Depends on clock for... | If the clock jumps backward |
|---|---|---|
| Auto-increment / ticket / block | nothing | No effect |
| UUID v4 | nothing | No effect |
| UUID v7 / ULID | ordering only | Order is wrong for a while; IDs still unique (random bits) |
| Snowflake | ordering **and uniqueness** | Duplicates unless the generator detects and refuses |

That table is the core lesson: the more of the ID's uniqueness comes from time, the more a bad clock
can hurt you.

---

## 7. Other considerations

### 7.1 Size and representation

- 64-bit integers (Snowflake, ticket servers) fit natively in every database and language. Watch out for
  **JavaScript**: its `Number` is a double with 53 bits of integer precision, so a 64-bit Snowflake ID
  sent as a JSON number gets silently rounded. Send large IDs as strings in JSON APIs.
- 128-bit UUIDs should be stored in a native `uuid` / `BINARY(16)` column, not as a 36-character string.
  A `CHAR(36)` primary key is more than twice as large, compares slower, and bloats every secondary index.

### 7.2 Exposing IDs externally

Sequential IDs in URLs leak information and invite enumeration:
- `order/1000` today and `order/1500` next week tells a competitor you take ~70 orders a day (the
  "German tank problem").
- An authorization bug plus guessable IDs = scraping everyone's data (topic 54). Unpredictable IDs are
  not a substitute for authorization checks, but they reduce the damage of a missing check.

A common pattern: use a compact sequential or time-ordered ID internally (primary key, joins) and a
separate random public ID (or an encrypted/obfuscated form of the internal ID) in URLs and APIs.

### 7.3 Natural keys and deterministic IDs

Sometimes the best ID isn't generated at all: derive it from content or from the request. UUID v5 of a
natural key, or a hash of the content (content addressing, topic 53), gives the same ID for the same
input, which makes retries idempotent (topic 38): two attempts to create "the same thing" produce the
same ID and the second is a no-op, instead of two records.

### 7.4 Client-generated IDs

Because UUIDs and Snowflake-ish IDs need no coordination, the **client** can generate the ID before
sending the request. That makes the create request idempotent (retry with the same ID), lets offline
clients create records and sync later (topic 33), and lets the client reference the new object
immediately. The cost: you must trust or validate client-provided IDs (collision by malice, not chance).

---

## 8. Choosing a scheme

| Scheme | Bits | Coordination | Time-sortable | Clock risk | Guessable | Typical fit |
|---|---|---|---|---|---|---|
| DB auto-increment | 32/64 | One DB | Yes | None | Yes | Single database |
| Ticket server | 64 | Per ID (network hop) | Roughly | None | Yes | Compact IDs, moderate rate |
| Block allocation | 64 | Per block | Per server only | None | Yes | Compact IDs, high rate |
| UUID v4 | 128 | None | No | None | No | Public IDs, hash-partitioned stores, LSM stores |
| UUID v7 / ULID | 128 | None | Yes (ms) | Order only | Partly (time visible) | Default for B-tree primary keys in new systems |
| Snowflake | 64 | Worker ID at startup | Roughly (ms) | Duplicates | Partly | High rate, need 64-bit, time-ordered |

A reasonable default reasoning chain:
1. Single database, IDs not exposed? Auto-increment is fine.
2. Multiple writers, B-tree storage, 128 bits acceptable? UUID v7.
3. Need 64-bit (legacy schema, JS-safe after stringifying, compact indexes) and very high rates?
   Snowflake-style, with careful worker-ID assignment and clock handling.
4. Need compact IDs and no clock dependence? Block allocation backed by a consensus store.
5. IDs in public URLs? Add a random public ID or use v4 for the public-facing one.
6. Range-partitioned distributed database? Avoid monotonic keys as the leading key component, or salt.

---

## 9. How it connects

- **Topic 17 (B-trees & WAL)** and **topic 18 (indexes)**: the index-locality argument for sequential vs
  random keys; clustered indexes amplify it.
- **Topic 19 (LSM-trees)**: why random keys hurt LSM stores much less on writes.
- **Topic 22 (concurrency control)**: compare-and-set for block allocation.
- **Topic 23 (single-leader replication)**: async failover of a central counter can reissue values.
- **Topic 25 (partitioning)**: monotonic keys create hotspots on range-partitioned stores; embedding the
  shard in the ID for routing.
- **Topic 26 (faults)** and **topic 27 (clocks)**: process pauses and clock jumps are the failure modes of
  Snowflake; Snowflake order is not happens-before order.
- **Topic 28 (consistency)** and **topic 31 (coordination)**: the block allocator and worker-ID leases need
  linearizable storage; lease expiry hazards.
- **Topic 38 (idempotency)**: deterministic and client-generated IDs make retries safe.
- **Topic 53 (object storage)**: content addressing, where the ID *is* the hash of the content.
- **Topic 54 (authz)**: guessable IDs and enumeration.

---

## Common misconceptions

- "UUIDs might collide, so I need to check for duplicates." → With a proper CSPRNG, v4 collisions are
  far less likely than hardware corruption. The real risk is broken randomness (cloned VMs, bad seeding),
  which a database unique constraint catches anyway.
- "UUIDs are slow as primary keys." → *Random* UUIDs (v4) are slow as B-tree primary keys at scale
  because of scattered inserts. Time-ordered UUIDs (v7) behave close to auto-increment. The 16-byte size
  is a smaller, secondary cost.
- "Snowflake IDs give a global order of events." → They give approximate creation order, subject to
  clock skew between workers and arbitrary ordering within a millisecond. Not causality.
- "Auto-increment is gap-free." → Rolled-back transactions, crashes and caching of sequence values all
  create gaps, even on one database.
- "Sequential IDs are always better for performance." → Better for a single B-tree; worse for
  range-partitioned clusters, where they concentrate writes on one partition.
- "If the clock goes backward, Snowflake just produces slightly misordered IDs." → Without the
  backward-clock check, it produces *duplicate* IDs.
- "Hiding IDs is a security measure." → Unpredictable IDs reduce the impact of missing authorization
  checks; they don't replace them.

---

## Further reading (supplements)

- RFC 9562, *Universally Unique IDentifiers (UUIDs)* (2024): defines v6, v7, v8 and explains the motivation for time-ordered UUIDs.
- Twitter Engineering blog, "Announcing Snowflake" (2010), and the archived `twitter-archive/snowflake` repository.
- Flickr Code blog, "Ticket Servers: Distributed Unique Primary Keys on the Cheap" (2010).
- Instagram Engineering, "Sharding & IDs at Instagram" (2012).
- Percona blog posts on UUIDs as primary keys in InnoDB (insert performance with random vs ordered keys).
- The ULID specification (github.com/ulid/spec).

---

## Questions

1. Why can't each shard of a partitioned database just use its own auto-increment counter? Name two concrete things that break.
2. A central "next ID" service is replicated with asynchronous replication. The leader crashes and a follower is promoted. Walk through how duplicate IDs can result.
3. In block allocation, what store properties does the "next free block" counter need, and what's the consequence of choosing block size 10 vs 1,000,000?
4. Explain, at the level of B-tree pages and the buffer pool, why inserting UUID v4 primary keys gets dramatically slower once the table no longer fits in memory, while auto-increment keys don't.
5. Why does the clustered-index design of InnoDB make random primary keys worse than in Postgres?
6. Your team switches from UUID v4 to UUID v7 in a Postgres table. What gets better, and what new information do your IDs now reveal?
7. Same switch, but the database is a range-partitioned distributed SQL store (e.g. Spanner or CockroachDB). What new problem appears, and how might you mitigate it?
8. A Snowflake generator's clock is stepped back by 5 ms by NTP. Without special handling, what happens? With the standard handling? What if the process restarts during those 5 ms?
9. Compare the consequence of a backward clock jump for UUID v7 vs Snowflake. Why is one an ordering problem and the other a correctness problem?
10. Two Snowflake workers are accidentally given the same worker ID. When will you notice, and what's the damage?
11. Snowflake IDs from two different services are used to decide which of two conflicting updates "came last". Why is this unsafe?
12. A web frontend receives `{"id": 1541815603606036480}` and later sends it back to fetch the object. The server says "not found". What happened?
13. Why might you use auto-increment internally but expose a different identifier in your public API?
14. How do client-generated IDs make a "create order" endpoint safe to retry? What risk do they introduce?
15. Instagram put the logical shard ID inside its IDs. What does that enable, and what does it constrain?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Each shard's counter starts at 1, so the same ID exists on every shard. Breaks: (a) global lookups by ID are ambiguous unless you also know the shard; (b) rebalancing or merging data (moving rows between shards, loading into a warehouse or search index, combining caches) creates collisions. Foreign keys from other services also become ambiguous.

**2.** The leader hands out IDs 1000–1005 and acknowledges them to clients, but async replication has only shipped the counter state up to 1000 to the follower. The leader crashes; the follower is promoted with counter = 1000 and hands out 1001–1005 again. Those IDs are now in use twice. The root issue: an acknowledged write was lost on failover (topic 23). The counter store needs synchronous replication or consensus (linearizable increments).

**3.** It needs linearizable, durable compare-and-set or increment (no lost updates, no stale reads after failover), so etcd/ZooKeeper, a consensus-backed DB, or a single DB row updated with `UPDATE ... SET next = next + size RETURNING`. Block size 10: frequent trips to the counter service (load on it, less tolerance to its outage), tiny gaps on crash. Block size 1,000,000: rare trips, servers can run for a long time if the counter service is down, but crashes waste up to a million IDs each and IDs are far from time-ordered across servers.

**4.** Auto-increment inserts always hit the right-most leaf page and the pages on its root path, which stay hot in the buffer pool, so inserts need no disk reads. UUID v4 inserts land on a uniformly random leaf. Once the index exceeds the buffer pool, most target leaves aren't cached, so each insert first reads a page from disk (random I/O, topic 02), dirties it, and eventually writes it back. Random inserts into full pages also cause splits that leave pages roughly 2/3 full, inflating the index and lowering the cache hit ratio further. Plus more WAL (full-page images).

**5.** In InnoDB the table rows are stored inside the primary key B-tree (clustered index), so random PKs scatter entire rows, not just small index entries, causing splits and I/O on the main data. Also every secondary index stores the primary key as the row pointer, so a 16-byte random PK enlarges every secondary index. In Postgres the heap is separate (rows appended to heap pages) and only the PK index suffers the scattering.

**6.** Better: inserts become right-edge appends (high buffer-pool hit rate, fewer splits, denser pages, less WAL), recent rows are physically clustered, and you can paginate and time-range by ID. Revealed: the creation timestamp in milliseconds, visible to anyone who sees the ID. They're also somewhat less unpredictable (74 random bits instead of 122, and the time part is guessable), though still infeasible to enumerate.

**7.** All new inserts have the largest keys, so they all go to the last range/partition: a write hotspot on one node while others are idle (topic 25). Mitigations: lead the primary key with a hash/bucket prefix (salting), use UUID v4 for the leading key, or use a composite key with a well-distributed first column (e.g. tenant ID).

**8.** Naively: the generator sees timestamps 5 ms older than ones it already used. When the clock returns to those milliseconds it restarts the sequence at 0 for timestamps it already issued, producing duplicate IDs. Standard handling: detect `ts < last_ts` and wait (for 5 ms that's trivial) or error, so no duplicates. If the process restarts during the window, the in-memory `last_ts` is lost and the check can't catch it; it will reissue IDs for those milliseconds. Fixes: persist a high-water timestamp and refuse to start until the clock passes it, or wait out a maximum-skew interval on startup, and run NTP in slew mode.

**9.** UUID v7's uniqueness comes mainly from 74 random bits; the timestamp is just a prefix for ordering. A repeated timestamp produces IDs with fresh random bits, so they're still unique, merely out of order. Snowflake's non-time part is the worker ID (constant) and a deterministic sequence counter starting from 0, so the same (timestamp, worker, sequence) triple recurs exactly: duplicates.

**10.** Possibly never immediately: as long as they don't generate in the same millisecond with the same sequence, IDs differ. Collisions appear when both generate in the same millisecond (common under load): both produce `(ts, w, 0)`. With a unique constraint, one insert fails (errors you'll see); without one (e.g. IDs used as keys in a key-value store or as event IDs), one record silently overwrites the other or deduplication drops a legitimate event. That's why worker IDs should be leased from a coordination service rather than hand-configured.

**11.** The timestamps come from different machines' wall clocks, which differ by clock skew, and within a millisecond ordering is by worker ID. An update made later in real time can get a smaller ID. Using it for last-write-wins silently discards the actually-newer update (topic 27, topic 33). Causality needs logical clocks or version vectors, or a single serialization point.

**12.** JavaScript numbers are IEEE doubles with 53 bits of integer precision. A 64-bit Snowflake ID above 2^53 is rounded when parsed, so the client sends back a slightly different number. Serialize 64-bit IDs as strings in JSON.

**13.** Internal sequential IDs are compact and fast for joins and indexes. Exposed publicly, they reveal volume and growth (anyone can diff IDs over time) and make enumeration trivial, so any authorization bug becomes a mass data leak. A separate random public ID (or encrypted internal ID) hides that while keeping the efficient internal key.

**14.** The client generates the order ID once and sends it with every retry. The server inserts with that ID; a retry hits a unique-constraint conflict (or an upsert no-op) instead of creating a second order (topic 38). Risk: the server is trusting client input for a primary key, so a malicious or buggy client can collide with someone else's ID or choose patterned IDs. Validate format, scope IDs per client/tenant, and treat conflicts carefully (return the existing object only if it belongs to the same caller).

**15.** Any service can compute the shard from the ID alone (bit shift) and route the request without a lookup table or directory service (topic 25 request routing). Constraint: an object can never move to a different logical shard without changing its ID, so the number of logical shards is fixed up front (Instagram used thousands) and rebalancing moves whole logical shards between physical machines.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Why is uniqueness different from the other ID properties?
**A:** It's a correctness requirement; sortability, compactness, unpredictability are performance/convenience/privacy trade-offs.

**Q:** Why does per-shard auto-increment fail?
**A:** Every shard issues the same values; IDs are unique only within a shard.

**Q:** Is auto-increment gap-free?
**A:** No. Rolled-back transactions and crashes leave gaps; sequences are non-transactional by design.

**Q:** What is a ticket server?
**A:** A dedicated DB that hands out numbers from one counter; often two with odd/even offsets for availability.

**Q:** What is block (hi/lo) allocation?
**A:** Servers lease ranges of IDs from a durable counter and hand them out locally without network calls.

**Q:** What must the block counter store guarantee?
**A:** Linearizable, durable increments; a lost update after failover hands the same block out twice.

**Q:** How many random bits in UUID v4?
**A:** 122 (6 bits are version and variant).

**Q:** Roughly how many v4 UUIDs for a 50% collision chance?
**A:** About 2^61 (~2.3 × 10^18).

**Q:** What actually causes UUID v4 collisions in practice?
**A:** Bad randomness: cloned VM state, shared seeds, non-cryptographic PRNGs.

**Q:** Why do random keys hurt B-tree inserts?
**A:** Inserts scatter across all leaves, so the whole index must be cached, pages split and sit ~2/3 full, and WAL grows.

**Q:** Why do random keys hurt LSM trees much less?
**A:** LSM writes are buffered in the memtable and flushed sequentially regardless of key order.

**Q:** UUID v7 layout?
**A:** 48-bit Unix ms timestamp, version, then 74 random bits (optionally part used as a counter).

**Q:** What does UUID v7 reveal?
**A:** The creation time to millisecond precision.

**Q:** What is ULID?
**A:** 48-bit ms timestamp + 80 random bits, encoded as 26 base32 chars that sort lexicographically.

**Q:** Snowflake layout?
**A:** 1 sign bit, 41 bits ms timestamp (custom epoch), 10 bits worker ID, 12 bits sequence.

**Q:** Snowflake's capacity per worker?
**A:** 4096 IDs per millisecond (~4M/s); on overflow it waits for the next ms.

**Q:** Snowflake's two uniqueness assumptions?
**A:** Worker IDs are unique, and each worker's clock never goes backward.

**Q:** What happens to Snowflake on a backward clock jump without a check?
**A:** It reissues (timestamp, worker, sequence) triples: duplicate IDs.

**Q:** What happens to UUID v7 on a backward clock jump?
**A:** IDs are temporarily misordered but still unique, thanks to random bits.

**Q:** How are Snowflake worker IDs safely assigned?
**A:** Leased from a coordination service (ephemeral nodes/leases), with care for pauses past lease expiry.

**Q:** Are Snowflake IDs a causal order?
**A:** No; only approximate creation order, subject to clock skew and arbitrary within-ms ordering.

**Q:** Why are monotonic keys bad on range-partitioned stores?
**A:** All new writes land on the last range: a single-partition hotspot.

**Q:** What's salting a key?
**A:** Prefixing with a small hash bucket so writes spread over several ranges while staying ordered within a bucket.

**Q:** Why send 64-bit IDs as strings in JSON?
**A:** JavaScript numbers have only 53 bits of integer precision; larger IDs get rounded.

**Q:** Why not store UUIDs as CHAR(36)?
**A:** More than twice the size of BINARY(16)/uuid, slower comparisons, bloated indexes.

**Q:** What do UUID v3/v5 give you?
**A:** Deterministic IDs from a namespace + name hash: same input, same ID (good for idempotency).

**Q:** Why expose a separate public ID?
**A:** Sequential IDs leak volume and enable enumeration.

**Q:** Benefit of embedding a shard ID in the ID?
**A:** Routing from the ID alone; cost is the object can't change logical shard.
