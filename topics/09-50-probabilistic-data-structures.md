# 50 — Probabilistic Data Structures

> Part 9 — Specialized Building Blocks · Prerequisites: topics 11, 12, 19 (helpful: 24, 41)

Some questions are cheap to answer exactly on small data and absurdly expensive on big data. "Have I seen
this URL before?" over ten billion URLs. "How many distinct users visited today?" over a stream of a
billion events. "Which search queries are trending right now?" across every query ever typed. An exact
answer means remembering every item, which means memory proportional to the data.

Probabilistic data structures answer these questions using a small, fixed amount of memory, in exchange
for a controlled, mathematically bounded error. A Bloom filter answers set membership in about 10 bits
per item regardless of how large the items are. A HyperLogLog counts billions of distinct elements in
12 KB with under 1% error. A Count-Min Sketch estimates the frequency of any item in a stream with a few
kilobytes. These structures show up inside databases (LSM-tree reads, topic 19), caches (penetration
protection, topic 12), stream processors (topic 41), and Redis itself.

By the end of this topic you should be able to:
1. Explain how a Bloom filter works, why it has false positives but never false negatives, and derive how to size it for a target false-positive rate.
2. Explain why a plain Bloom filter can't delete, and how counting Bloom filters (and cuckoo filters) fix that, at what cost.
3. Describe the Count-Min Sketch: its structure, why it only overestimates, and how width and depth control the error.
4. Explain the intuition and mechanism of HyperLogLog, why its memory is independent of the number of items, and why HLLs can be merged.
5. Decide when trading exactness for memory is acceptable, and recognise the common failure modes.

---

## 1. The trade: exactness for memory

### 1.1 Why exact answers are expensive

Three common questions and the cost of answering them exactly:

| Question | Exact structure | Memory for 1 billion items |
|---|---|---|
| Is `x` in the set? (membership) | Hash set | Size of all items + overhead: tens of GB for URLs |
| How many distinct items? (cardinality) | Hash set of seen items | Same: you must remember every item to avoid double-counting |
| How often has `x` appeared? (frequency) | Hash map item → count | Every distinct item plus a counter |

All three are linear in the number of distinct items. For a single query on a laptop that's fine. For
per-minute distinct-user counts across 10,000 pages, or a membership check that must fit in RAM next to
every SSTable on every node, it's not.

### 1.2 What we give up

Probabilistic structures exploit two ideas:

1. **Hashing** turns arbitrary items into uniformly distributed bits. A good hash function behaves like a
   random function, so we can reason about collisions with probability.
2. **Lossy summaries**: we keep only a fixed-size summary, and accept that different inputs can produce
   the same summary state. The error is one-sided or bounded, and we pick the memory to hit a target error.

The key properties to check for any such structure:
- **What kind of error?** One-sided (Bloom: only false positives; Count-Min: only overestimates) or
  two-sided (HyperLogLog: can be above or below).
- **How big is the error, and with what probability?** Expressed as an error rate (`ε`) and a failure
  probability (`δ`).
- **Is it mergeable?** Can two summaries computed on different machines be combined into the summary of
  the union? This matters enormously in distributed systems: compute per shard, merge centrally.
- **Can it delete?** Many can't.

---

## 2. Bloom filters in depth

### 2.1 The problem

You want to know if an item is in a set, and a "no" answer is very valuable because it lets you skip
expensive work. Examples:

- An LSM-tree read (topic 19): before reading an SSTable from disk, ask "could this key be in this
  SSTable?" If no, skip the disk read. With dozens of SSTables per key lookup, this is the difference
  between one disk read and dozens.
- Cache penetration (topic 12): requests for IDs that don't exist bypass the cache and hammer the
  database. A Bloom filter of all valid IDs rejects them up front.
- A web crawler: "have I already crawled this URL?"
- A browser's malicious-URL check: a compact local filter; only on a "maybe" does it ask the server.

In all these, an occasional wrong "maybe" just costs one extra lookup. A wrong "no" would be a
correctness bug. Bloom filters give exactly that error profile.

### 2.2 The mechanism

A Bloom filter is an array of `m` bits, all initially 0, plus `k` independent hash functions, each
mapping an item to a position in `[0, m)`.

- **Insert(x)**: compute `h1(x), ..., hk(x)` and set those `k` bits to 1.
- **Query(x)**: compute the same `k` positions. If **any** is 0, `x` is definitely not in the set. If
  **all** are 1, `x` is *probably* in the set.

Worked example with `m = 16`, `k = 3`:

```
start:                  0000 0000 0000 0000

insert "cat"  -> 2, 7, 13
                        0010 0001 0000 0100
                          ^    ^       ^
insert "dog"  -> 4, 7, 11
                        0010 1001 0001 0100
                             ^ (7 already set)

query "cat"   -> 2, 7, 13: all 1 -> "probably present"   (true positive)
query "cow"   -> 1, 7, 9 : bit 1 = 0 -> "definitely absent"
query "emu"   -> 4, 11, 13: all 1 -> "probably present"  (FALSE POSITIVE: bits set by cat and dog)
```

Why no false negatives: inserting `x` set all its bits to 1, and bits are never cleared. So a later
query for `x` finds all of them set.

Why false positives: bits are shared. As the filter fills, the chance that some unseen item's `k`
positions are all already 1 (set by other items) grows.

Note what the filter does *not* store: the items themselves. You cannot list the members, and the size
doesn't depend on item size: a 2 KB URL costs the same as an 8-byte integer.

### 2.3 The false-positive rate

After inserting `n` items into `m` bits with `k` hash functions, the probability that a particular bit is
still 0 is:

```
P(bit is 0) = (1 - 1/m)^(kn)  ≈  e^(-kn/m)
```

A false positive needs all `k` positions of a non-member to be 1:

```
FPR ≈ (1 - e^(-kn/m))^k
```

Two forces act on `k`:
- More hash functions = more bits that must all be 1 for a false positive (good).
- More hash functions = more bits set per insert, so the filter fills faster (bad).

The optimum is:

```
k_opt = (m / n) · ln 2  ≈  0.693 · (m / n)
```

At the optimum, about half the bits are 1 (the filter carries the most information when it's a coin
flip), and the false-positive rate becomes:

```
FPR ≈ (0.6185)^(m/n)
```

### 2.4 Sizing a Bloom filter

Usually you know `n` (expected items) and the target false-positive rate `p`, and you want `m` and `k`:

```
m / n = -ln(p) / (ln 2)^2  ≈  1.44 · log2(1/p)   bits per item
k     = log2(1/p)          (rounded)
```

| Target FPR `p` | Bits per item | Hash functions `k` |
|---|---|---|
| 10% | 4.8 | 3 |
| 1% | 9.6 | 7 |
| 0.1% | 14.4 | 10 |
| 0.01% | 19.2 | 13 |

Rules of thumb that follow:
- **~10 bits per item gives 1%.** Every additional ~4.8 bits per item cuts the FPR by 10×.
- Memory depends on `n` and `p`, never on item size. 1 billion items at 1% ≈ 1.2 GB; a hash set of
  1 billion URLs would be ~100 GB.
- **Overfilling degrades fast.** If you size for `n` but insert `2n`, the FPR at 1% design goes to
  roughly 15%. A Bloom filter does not fail loudly; it just gets less useful. Size for the expected peak
  or use a scalable Bloom filter (2.7).

### 2.5 Hash functions in practice

You don't need `k` truly independent hash functions. The standard trick (Kirsch and Mitzenmacher, 2006)
derives them from two:

```
g_i(x) = h1(x) + i · h2(x)   mod m,    for i = 0 .. k-1
```

with no measurable loss in accuracy. Implementations typically compute one 128-bit fast hash (e.g.
MurmurHash3, xxHash) and split it into `h1` and `h2`. Cryptographic hashes aren't needed unless an
adversary chooses the inputs (see 2.8).

Cache behaviour matters too: `k` random bit positions means `k` cache misses on a large filter. **Blocked
Bloom filters** (used by RocksDB) first hash to one 64-byte cache line and set all `k` bits within it:
one cache miss per lookup, at a small cost in accuracy.

### 2.6 Deletion and counting Bloom filters

You can't delete from a standard Bloom filter. Clearing `x`'s bits would also clear bits shared by other
items, creating false negatives, which is the one error the filter promises never to make. In the
example above, deleting "cat" by clearing bit 7 would make "dog" disappear too.

A **counting Bloom filter** replaces each bit with a small counter (typically 4 bits):

```
insert(x):  increment the k counters
delete(x):  decrement the k counters
query(x):   all k counters > 0  -> probably present
```

Costs and caveats:
- **4× the memory** (4-bit counters instead of 1-bit).
- **Counter overflow**: 4 bits saturate at 15. Once a counter saturates it must stay stuck (never
  decremented), or it could underflow later. With optimal sizing, overflow is extremely rare.
- **Deleting something never inserted** decrements other items' counters and can cause false negatives.
  You must only delete items you know are present.

Uses: tracking a changing set, e.g. cache summaries exchanged between proxies (the original "Summary
Cache" paper used counting Bloom filters for exactly this).

A modern alternative is the **cuckoo filter** (Fan et al., 2014): it stores short fingerprints of items
in a cuckoo hash table. It supports deletion, often uses less space than a Bloom filter at low FPRs
(below ~3%), and does lookups in two memory accesses. It can fail to insert when very full.

### 2.7 Other variants and operations

- **Union**: two Bloom filters with the same `m`, `k` and hash functions combine by bitwise OR; the result
  is exactly the filter of the union. This is what makes them handy in distributed settings.
- **Intersection** via bitwise AND works approximately (its FPR is worse than building the filter for the
  intersection directly).
- **Scalable Bloom filters** grow by adding new, larger filters with tighter FPRs as old ones fill;
  queries check all of them. Useful when `n` isn't known up front (RedisBloom's `BF.RESERVE` with
  expansion works this way).
- **Estimating `n`** from the fraction of set bits is possible: `n ≈ -(m/k)·ln(1 - X/m)` where `X` is the
  number of 1 bits.

### 2.8 Failure modes

- **Overfilling** (2.4): FPR silently climbs.
- **Adversarial inputs**: if an attacker knows your hash function, they can craft items that are false
  positives (or that fill specific bits). Use a keyed hash when inputs are attacker-controlled.
- **Rebuilding**: since you can't delete, a filter for a set with lots of churn must be periodically
  rebuilt from the source of truth. In an LSM-tree, each SSTable's filter is built once when the SSTable
  is written and is immutable, so this problem never arises: compaction writes new SSTables with new
  filters.
- **Wrong use of "maybe"**: treating "probably present" as "present" without checking the real data
  turns a performance optimization into a correctness bug.

---

## 3. Count-Min Sketch: frequencies

### 3.1 The problem

In a stream of events (search queries, IP addresses hitting a server, products viewed), you want to
estimate how many times a given item appeared, and find the **heavy hitters** (top-k most frequent).
An exact hash map item→count over billions of distinct items is too large, especially when you want one
per minute, per region, per service.

### 3.2 The structure

A Count-Min Sketch (Cormode and Muthukrishnan, 2005) is a 2-D array of counters with `d` rows and `w`
columns, and one hash function per row.

```
         w columns
       +---+---+---+---+---+---+---+---+
row 1  | 0 | 3 | 0 | 1 | 0 | 7 | 0 | 2 |   h1
row 2  | 4 | 0 | 0 | 2 | 0 | 0 | 5 | 2 |   h2
row 3  | 0 | 1 | 8 | 0 | 2 | 0 | 0 | 2 |   h3   d rows
row 4  | 2 | 0 | 0 | 0 | 6 | 3 | 0 | 2 |   h4
       +---+---+---+---+---+---+---+---+
```

- **Update(x, c)**: for each row `i`, add `c` to `count[i][h_i(x)]`.
- **Query(x)**: return `min over i of count[i][h_i(x)]`.

Why the minimum: each counter that `x` maps to holds `x`'s true count **plus** the counts of every other
item that collided into that cell. Collisions only ever add. So every row's counter is an overestimate,
and the least-polluted one (the minimum) is the best estimate.

```
true count(x) <= estimate(x)          always (never underestimates)
estimate(x)  <= true(x) + ε·N         with probability >= 1 - δ
                (N = total of all counts in the stream)
```

### 3.3 Sizing

```
w = ceil(e / ε)          (e ≈ 2.718)
d = ceil(ln(1 / δ))
```

Example: you want each estimate within 0.1% of the total stream size (`ε = 0.001`) with 99.9% confidence
(`δ = 0.001`): `w = 2719`, `d = 7`. That's ~19,000 counters; at 4 bytes each, ~76 KB, regardless of
whether the stream has a thousand or a billion distinct items.

Note what the error is relative to: **`N`, the total stream volume**, not the item's own count. For a
heavy hitter with 5% of all traffic, an error of 0.1% of `N` is small. For a rare item with 0.001% of
traffic, the error can be 100× its true count. **Count-Min is accurate for frequent items and useless for
rare ones.** That's fine for its main use, finding heavy hitters.

Width controls the size of the error (more columns → fewer collisions per cell). Depth controls the
probability of a bad estimate (more rows → more independent chances that at least one row has few
collisions).

### 3.4 Improvements and uses

- **Conservative update**: when adding to `x`, only increase each of its counters up to
  `estimate(x) + c`, not blindly by `c`. Reduces overestimation noticeably; the cost is that the sketch
  no longer supports deletions/negative updates.
- **Heavy hitters / top-k**: keep a Count-Min Sketch plus a small min-heap of the current top-k items.
  For each event, update the sketch, query the estimate, and if it exceeds the heap's minimum, insert or
  update in the heap. Memory: sketch + k entries. RedisBloom's `TOPK` (based on a variant called
  HeavyKeeper) and `CMS.*` commands provide these.
- **Mergeable**: two sketches with identical dimensions and hash functions add cell by cell. Compute
  per-partition sketches in a stream job (topic 41) and sum them.
- **Uses**: trending queries, detecting heavy IPs for rate limiting or DDoS mitigation (topic 45),
  identifying hot keys in a cache or partition (topic 12, topic 25), database query planners estimating
  value frequencies.

---

## 4. HyperLogLog: cardinality

### 4.1 The problem

"How many **distinct** users visited this page today?" The difficulty is duplicates: to avoid counting a
user twice, an exact method must remember every user it has seen. 100 million distinct users per day per
page × thousands of pages is a lot of memory. HyperLogLog (Flajolet et al., 2007) estimates distinct
counts in the billions using about 12 KB, with ~0.81% standard error.

### 4.2 The intuition: rare patterns imply many trials

Hash each item to a uniformly random bit string. Look at the number of **leading zeros**:

```
P(hash starts with "1")      = 1/2
P(hash starts with "01")     = 1/4
P(hash starts with "001")    = 1/8
P(hash starts with k zeros then 1) = 1/2^(k+1)
```

If the longest run of leading zeros you've seen across all items is `k`, you've probably seen around
`2^k` distinct items. It's like coin flipping: if someone tells you their longest streak of heads was 10,
you'd guess they flipped about a thousand times.

Crucially, **duplicates don't matter**: the same item always produces the same hash, so it can't make the
maximum bigger by appearing again. The structure only needs to remember one number (the max), which
fits in a few bits. That's where the tiny memory comes from.

But a single maximum is a terribly noisy estimator: one lucky hash with 30 leading zeros makes you guess
a billion. HyperLogLog fixes the variance by averaging many independent estimates.

### 4.3 The mechanism

Use `m = 2^p` registers (Redis uses `p = 14`, so `m = 16,384`).

```
hash(x) = 64 bits
          |<-- p bits -->|<---------- remaining 64-p bits ---------->|
            register index      rank = position of first 1 bit
                                        (leading zeros + 1)

register[index] = max(register[index], rank)
```

Each item updates exactly one register: the first `p` bits pick which, and the rest give the "rank". This
is *stochastic averaging*: it simulates `m` independent experiments with only one hash per item.

Estimate:

```
E = α_m · m² / Σ_j 2^(-register[j])
```

That's `α_m · m` times the **harmonic mean** of the per-register estimates `2^register[j]`. The harmonic
mean is used because it's much less sensitive to the occasional huge outlier register than the
arithmetic mean (the key improvement over the earlier LogLog algorithm). `α_m` (≈ 0.7213 for large `m`)
corrects a systematic bias.

Corrections:
- **Small cardinalities**: when many registers are still 0, the formula is biased. HLL switches to
  **linear counting**: `m · ln(m / V)` where `V` is the number of zero registers. (Google's HyperLogLog++
  improves this range further with empirical bias correction and a sparse representation.)
- **Large cardinalities**: with a 64-bit hash, hash collisions are not a concern until far beyond any
  practical count.

### 4.4 Accuracy and memory

```
standard error ≈ 1.04 / sqrt(m)
```

| Registers `m` | Memory (6-bit registers) | Standard error |
|---|---|---|
| 1,024 (p=10) | 768 B | 3.25% |
| 16,384 (p=14) | 12 KB | 0.81% |
| 65,536 (p=16) | 48 KB | 0.41% |

Each register only needs to store a rank up to ~64, so 6 bits suffice. Memory depends on `m` only, **not
on the cardinality**: the same 12 KB counts ten distinct items or ten billion. Redis's `PFADD`/`PFCOUNT`
use exactly this (p=14, 12 KB dense), with a sparse encoding for small sets to save memory.

Standard error is a standard deviation: with 0.81%, about 68% of estimates are within ±0.81% and about
95% within ±1.6%. An estimate can be above or below the truth.

### 4.5 Merging: the killer feature

Two HLLs with the same `p` and hash function merge by taking the **element-wise maximum** of registers.
The result is *exactly* the HLL you'd have built from the union of both streams, because each register
is a max over items, and `max` is associative, commutative and idempotent.

```
HLL(Monday) ∪ HLL(Tuesday) = register-wise max  = HLL(Monday ∪ Tuesday)
```

Consequences:
- **Distributed counting**: each shard or each stream partition keeps its own HLL; merge them centrally.
  No need to shuffle user IDs between machines (topic 40, topic 41).
- **Roll-ups**: store one HLL per hour; weekly unique users = merge 168 hourly HLLs. Exact distinct counts
  can't do this: you can't add daily unique counts to get weekly uniques because the same user appears
  on many days.
- **Idempotent**: re-adding the same item or re-merging the same HLL changes nothing, so at-least-once
  delivery (topic 38) doesn't inflate the count. This makes HLL essentially a CRDT (topic 33): state-based,
  merge = max.

What you can't do well: **intersections**. `|A ∩ B|` can be estimated by inclusion–exclusion
(`|A| + |B| - |A ∪ B|`), but the absolute error is relative to the union, so small intersections of large
sets come out as noise. Use MinHash-based sketches (or Theta sketches) when intersections matter.

---

## 5. Trading exactness for memory: when it's acceptable

### 5.1 Good fits

- **The approximate answer drives a decision that's robust to small errors.** "Is this query trending?"
  "Roughly how many unique visitors?" Dashboards, analytics, capacity planning. Nobody acts differently
  on 1,003,000 vs 1,000,000.
- **The structure is a filter in front of an exact check.** Bloom filters in LSM-trees, cache
  penetration filters, crawler dedup with a fallback lookup. A false positive costs a bit of work; the
  exact answer is still available behind it.
- **The error is one-sided in the safe direction.** Count-Min overestimates: if you use it to detect
  heavy users for rate limiting, you might flag a borderline user, never miss a real heavy one.
- **Memory or network is the real constraint**: per-key, per-window, per-shard summaries that must be
  shipped and merged. Summaries of KB instead of GB make the system design possible at all.

### 5.2 Bad fits

- **Money, billing, inventory, quotas with contractual meaning.** "Approximately how many API calls did
  this customer make" can't be invoiced.
- **Security decisions with the wrong error direction.** A Bloom filter of revoked tokens (topic 54) is
  fine *only* if a "maybe revoked" triggers an exact check; using "not in filter = not revoked" is safe
  (no false negatives), but "in filter = revoked" would reject valid tokens.
- **Small data.** If the exact set fits comfortably in memory, use it. Probabilistic structures pay off
  at scale.
- **Rare items or small intersections** where the error bound, relative to the total, swamps the answer.

### 5.3 A summary table

| Structure | Question | Error | Memory | Delete? | Merge |
|---|---|---|---|---|---|
| Bloom filter | Membership | False positives only | ~1.44·log2(1/p) bits/item | No | Bitwise OR |
| Counting Bloom | Membership with deletes | False positives only | ~4× Bloom | Yes | Add counters |
| Cuckoo filter | Membership with deletes | False positives only | ≈ Bloom (better at low p) | Yes | Not simply |
| Count-Min Sketch | Frequency of an item | Overestimate ≤ εN w.p. 1-δ | (e/ε)·ln(1/δ) counters | With plain update | Cell-wise add |
| HyperLogLog | Distinct count | ±1.04/√m (two-sided) | m × 6 bits (12 KB typical) | No | Register-wise max |

Other members of the family you'll meet: **t-digest** and **KLL sketches** for approximate quantiles
(p99 latency across many servers, topic 46; you can't average percentiles, but you can merge digests),
**MinHash** for set similarity, **reservoir sampling** for uniform samples of a stream.

---

## 6. How it connects

- **Topic 11–12 (caching, cache failure modes)**: Bloom filters against cache penetration; Count-Min to
  detect hot keys.
- **Topic 19 (LSM-trees)**: per-SSTable Bloom filters are what keep LSM reads fast; their bits-per-key
  setting is a read-amplification vs memory knob.
- **Topic 24 (leaderless replication)** and **topic 53 (Merkle trees)**: other compact summaries used to
  compare data between replicas.
- **Topic 25 (partitioning)**: detecting hot keys and skew.
- **Topic 33 (CRDTs)**: HLL merge is a join-semilattice like a state-based CRDT.
- **Topic 40–41 (batch and stream processing)**: mergeable sketches let you compute per partition and
  combine; windowed distinct counts and top-k.
- **Topic 45 (rate limiting)**: Count-Min for approximate per-client counters at very high cardinality.
- **Topic 46 (observability)**: quantile sketches for percentiles across hosts.
- **Topic 51 (search)**: Bloom filters and similar summaries appear in index structures.

---

## Common misconceptions

- "A Bloom filter might say an inserted item is missing." → Never. False negatives are impossible (unless
  you delete from a counting filter something never inserted, or the implementation is buggy).
- "Bloom filter size depends on how big the items are." → Only on the number of items and the target
  FPR; item size is irrelevant.
- "More hash functions always means fewer false positives." → Beyond `k = (m/n)·ln 2`, extra hash
  functions fill the filter faster and the FPR rises.
- "You can remove items from a Bloom filter by clearing their bits." → That clears bits shared with
  other items, producing false negatives.
- "Count-Min Sketch error is a percentage of the item's count." → It's a fraction of the *total stream*
  count, so rare items get relatively huge errors.
- "HyperLogLog memory grows with the number of distinct items." → It's fixed by the number of registers;
  12 KB for 0.81% error whether there are 10 or 10 billion distinct items.
- "You can add up daily unique-visitor counts to get weekly uniques." → Not with exact counts (double
  counting); you *can* merge daily HLLs to get the weekly HLL.
- "Approximate means unreliable." → The errors are mathematically bounded and predictable; the design
  question is whether the bound is acceptable for the decision being made.

---

## Further reading (supplements)

- Burton H. Bloom, "Space/Time Trade-offs in Hash Coding with Allowable Errors" (1970).
- Broder and Mitzenmacher, "Network Applications of Bloom Filters: A Survey" (2004).
- Kirsch and Mitzenmacher, "Less Hashing, Same Performance: Building a Better Bloom Filter" (2006).
- Fan et al., "Cuckoo Filter: Practically Better Than Bloom" (2014).
- Cormode and Muthukrishnan, "An Improved Data Stream Summary: The Count-Min Sketch and its Applications" (2005).
- Flajolet, Fusy, Gandouet, Meunier, "HyperLogLog: the analysis of a near-optimal cardinality estimation algorithm" (2007); Heule, Nunkesser, Hall, "HyperLogLog in Practice" (Google, 2013, the HLL++ paper).
- Salvatore Sanfilippo (antirez), "Redis new data structure: the HyperLogLog" blog post (2014).
- DDIA chapter 3 (Bloom filters in LSM storage).

---

## Questions

1. Why can a Bloom filter produce false positives but never false negatives? Explain in terms of what insert and query do to bits.
2. You need a Bloom filter for 100 million keys at a 1% false-positive rate. Roughly how much memory and how many hash functions? What happens if 300 million keys are actually inserted?
3. Why does adding more hash functions eventually make the false-positive rate worse?
4. Explain what goes wrong if you "delete" an item from a standard Bloom filter by clearing its bits. How does a counting Bloom filter avoid it, and what new hazard does it introduce?
5. In an LSM-tree with 20 SSTables per level-path, a point lookup for a non-existent key costs how many disk reads with and without Bloom filters? Why don't LSM-trees need counting Bloom filters?
6. A service uses a Bloom filter of valid product IDs to block cache-penetration attacks. An attacker learns the hash function. What can they do?
7. In a Count-Min Sketch, why do we take the minimum across rows rather than the average?
8. A Count-Min Sketch with `ε = 0.001` processes 1 billion events. Item A truly appeared 50 million times; item B 200 times. Discuss the estimates you might get for each.
9. What do the width and the depth of a Count-Min Sketch each control?
10. Explain the coin-flip intuition behind HyperLogLog and why duplicate items don't inflate the estimate.
11. Why does HyperLogLog use many registers and a harmonic mean instead of one maximum?
12. You store one HLL per hour for each page. How do you get daily unique visitors for a page? Why is this impossible with exact hourly distinct counts alone?
13. Your stream processor delivers events at-least-once. Which of the three structures (Bloom, Count-Min, HLL) give the same result despite duplicates, and which don't?
14. Give one situation where an approximate structure is clearly appropriate and one where it would be wrong, and say what property decides it.
15. Why are mergeable summaries particularly valuable in distributed systems?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Insert sets the item's `k` bits to 1, and nothing ever sets bits back to 0. So querying an inserted item always finds all its bits set: no false negatives. But bits are shared between items, so a never-inserted item may find all its `k` positions already set by other items: a false positive.

**2.** About 9.6 bits per item × 100 M ≈ 960 M bits ≈ 120 MB, with `k = 7`. At 300 M items, `kn/m` triples: FPR ≈ (1 - e^(-7·3/9.6))^7 ≈ (1 - e^-2.19)^7 ≈ 0.888^7 ≈ 0.44. The filter now answers "maybe" for almost half of absent keys: it fails silently by becoming useless.

**3.** Each hash function sets another bit per insert, so the fraction of 1-bits grows. Initially, requiring more bits to match lowers the FPR; past `k = (m/n)·ln 2` the filter is so full that each extra position is very likely 1 anyway, and the faster filling dominates. FPR is minimized when about half the bits are set.

**4.** Cleared bits may also belong to other inserted items; those items now have a 0 among their positions and are reported "definitely absent": a false negative, breaking the filter's one guarantee. A counting filter keeps a counter per position and decrements; a position only reaches 0 when no remaining item uses it. New hazards: 4× memory, counter saturation (must freeze at max), and deleting an item that was never inserted decrements other items' counters, creating false negatives.

**5.** Without filters the lookup may read (an index block of) each of the 20 SSTables: ~20 disk reads to conclude "not found". With 1% filters, the expected number of SSTables wrongly checked is 20 × 0.01 = 0.2, so typically zero disk reads. SSTables are immutable: each filter is built once when the SSTable is written and discarded when compaction replaces the file, so no deletion is ever needed.

**6.** They can search offline for nonexistent IDs whose hash positions are all already 1 (false positives), and send only those, so every request passes the filter and hits the database again. They could also probe to learn the filter state. Mitigation: keyed/secret hash functions (e.g. SipHash with a secret key), plus negative caching and rate limiting behind it.

**7.** Every counter equals the item's true count plus non-negative contributions from colliding items. So every row overestimates; the minimum is the row with the least collision noise and is always ≥ the true count. Averaging would include badly polluted rows and gives no one-sided guarantee.

**8.** Error bound εN = 0.001 × 1 B = 1 M (with probability 1-δ). A: estimate between 50 M and ~51 M: within 2% — good. B: estimate between 200 and ~1,000,200 — possibly thousands of times too high. Count-Min is reliable for heavy hitters and meaningless for rare items.

**9.** Width (`w = e/ε`) controls the magnitude of the error: more columns → fewer collisions per cell → smaller overestimate. Depth (`d = ln(1/δ)`) controls the confidence: more independent rows → lower probability that all rows are heavily polluted for a given item.

**10.** A uniformly random hash starts with `k` zeros with probability `2^-(k+1)`. Seeing a hash with many leading zeros is rare, so if the maximum observed run is `k`, you've likely seen ~`2^k` distinct values, like guessing the number of coin-flip sessions from the longest streak of heads. The same item always has the same hash, so repeating it can't produce a new maximum; the structure only records maxima, so duplicates are invisible.

**11.** One maximum has huge variance: a single unlucky hash gives an estimate off by orders of magnitude. Splitting items across `m` registers (by hash prefix) gives `m` independent estimates from one hash per item; averaging reduces error as 1/√m. The harmonic mean dampens the influence of outlier registers with abnormally high ranks.

**12.** Merge the 24 hourly HLLs by register-wise max and query the result; it equals the HLL of all the day's visitors. With only exact hourly distinct counts you can't: a user who visited in 5 different hours is counted in 5 hourly numbers, and the counts carry no information about overlap. You'd need the raw sets.

**13.** Bloom filter: duplicates set the same bits; unaffected. HLL: same hash, same register, same max; unaffected (idempotent). Count-Min Sketch: each duplicate adds to the counters again; the frequency estimate is inflated. If you need frequency counts under at-least-once delivery, deduplicate first (topic 38).

**14.** Appropriate: unique visitors per page on an analytics dashboard (decision tolerant to ~1% error, huge cardinality, needs roll-ups), or a Bloom filter in front of a disk lookup (false positive only costs one extra read). Wrong: billing a customer per API call or enforcing a contractual quota, where the number has legal/financial meaning. Deciding property: whether the decision is robust to the bounded error, and whether the error direction is safe or backed by an exact check.

**15.** In a distributed system data is computed in pieces (per shard, per partition, per time window, per region). Mergeable summaries let each piece compute a small summary locally and combine them later without moving raw data, and combine across time windows for roll-ups. Shipping KB-sized sketches instead of raw item sets makes the network and memory cost independent of data volume. Idempotent merges (HLL max, Bloom OR) also tolerate duplicate delivery and retries.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Bloom filter: what errors can it make?
**A:** False positives only ("probably present" for an absent item); never false negatives.

**Q:** Bloom filter insert and query?
**A:** Insert sets k hashed bits to 1; query says "absent" if any of the k bits is 0, else "probably present".

**Q:** Bloom FPR formula?
**A:** (1 - e^(-kn/m))^k.

**Q:** Optimal number of hash functions?
**A:** k = (m/n)·ln 2; at that point about half the bits are set.

**Q:** Bits per item for 1% FPR?
**A:** About 9.6 (~10), with k = 7.

**Q:** Each extra ~4.8 bits per item does what?
**A:** Cuts the false-positive rate by 10×.

**Q:** Does Bloom filter size depend on item size?
**A:** No; only on item count and target FPR.

**Q:** What happens when a Bloom filter is overfilled?
**A:** FPR rises silently, possibly to near-useless levels.

**Q:** How do implementations get k hash functions cheaply?
**A:** Double hashing: g_i = h1 + i·h2 (Kirsch–Mitzenmacher).

**Q:** Why can't a standard Bloom filter delete?
**A:** Clearing bits would affect other items that share them, causing false negatives.

**Q:** Counting Bloom filter costs?
**A:** ~4× memory, counter saturation, and false negatives if you delete something never inserted.

**Q:** Cuckoo filter in one line?
**A:** Fingerprints in a cuckoo hash table: supports deletes, compact at low FPR, two memory accesses per lookup.

**Q:** How do you union two Bloom filters?
**A:** Bitwise OR (same m, k and hashes).

**Q:** Count-Min Sketch structure?
**A:** d rows × w counters, one hash per row; update adds to one cell per row; query takes the min.

**Q:** Count-Min error direction?
**A:** Overestimates only; estimate ≥ true count.

**Q:** Count-Min error bound?
**A:** ≤ ε·N (N = total stream count) with probability 1 - δ; w = e/ε, d = ln(1/δ).

**Q:** Why is Count-Min bad for rare items?
**A:** The error is relative to the whole stream, which can dwarf a rare item's true count.

**Q:** What is conservative update?
**A:** Only raise counters up to the new minimum estimate, reducing overestimation.

**Q:** How do you find heavy hitters with Count-Min?
**A:** Sketch plus a min-heap of the current top-k by estimated count.

**Q:** HyperLogLog core intuition?
**A:** Max leading zeros k in random hashes suggests ~2^k distinct items.

**Q:** Why are duplicates harmless to HLL?
**A:** Same item, same hash, same register update; max doesn't change.

**Q:** HLL register update?
**A:** First p hash bits pick the register; register = max(register, leading zeros of remaining bits + 1).

**Q:** HLL standard error?
**A:** About 1.04/√m; Redis uses m = 16,384, ~0.81%, 12 KB.

**Q:** Why the harmonic mean in HLL?
**A:** It limits the effect of outlier registers with abnormally large values.

**Q:** HLL merge?
**A:** Register-wise max; yields exactly the HLL of the union.

**Q:** What can't HLL do well?
**A:** Small intersections of large sets (inclusion–exclusion error is relative to the union).

**Q:** When is approximation acceptable?
**A:** When decisions tolerate bounded error, the error direction is safe, or an exact check sits behind it.

**Q:** When is approximation wrong?
**A:** Billing, inventory, contractual quotas, and small data that fits exactly anyway.
