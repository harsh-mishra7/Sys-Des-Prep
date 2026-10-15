# 12 — Cache Failure Modes

> Part 3 — Caching & the Edge · Prerequisites: topics 02, 11

Topic 11 showed that a cache with a 99% hit ratio lets the database see only 1% of reads, and that systems
quietly come to depend on that. This topic is about the ways that dependency breaks. Almost none of them are
"the cache crashed". They are patterns where the cache is technically working but the traffic reaching the
backend suddenly multiplies: many requests missing the same key at once, requests for keys that can never
be cached, many keys expiring together, one key receiving more traffic than a single node can serve, or a
cache that starts empty. Each has a recognizable signature and a standard set of defenses. Learning them is
the difference between a cache that makes a system fast and one that makes it fragile.

By the end of this topic you should be able to:
1. Explain the cache stampede (thundering herd) and apply request coalescing, locking and probabilistic early expiry.
2. Explain cache penetration and defend against it with negative caching and Bloom filters.
3. Explain cache avalanche and why TTL jitter (and other spreading techniques) prevents it.
4. Recognize hot keys and mitigate them with local caches and key replication.
5. Plan for cold starts with cache warming and gradual traffic shifting.
6. Reason about the common thread: misses are amplified load on the backend, and the fix is always to bound
   how much of that load can reach it at once.

---

## 1. The common thread: misses are load

Take a backend that can serve 2,000 queries per second and a cache in front of it absorbing 100,000 reads
per second at a 99% hit ratio. The backend sees 1,000 QPS: comfortable. Now consider the ways that number can
change:

```
                 normal         stampede on   penetration    avalanche      cold cache
                                one hot key   attack         (mass expiry)  (restart)
cache hit ratio   99%            ~99%          ~90%           ~70%           0%
backend QPS       1,000          +5,000 burst  10,000         30,000         100,000
                  (fine)         (on 1 row)    (overload)     (overload)     (overload)
```

Recall from topic 02 that as a server's utilization approaches 100%, queueing delay grows without bound. An
overloaded database does not get "a bit slower"; latency explodes, requests time out, clients retry (topic 43),
which adds more load, and the cache cannot be refilled because the refill queries are among the ones timing out.
This is a **metastable failure**: the system stays broken even after the original trigger is gone, because
the overload itself keeps the hit ratio low.

So every defense in this topic is a variant of one idea: **bound how many requests can reach the backend for
reasons the cache should have absorbed.**

---

## 2. Cache stampede (thundering herd / dog-piling)

### 2.1 What happens

A popular key expires (or is invalidated). In the gap between expiry and refill, every request for that key
misses, and in cache-aside each of them independently goes to the database to recompute the same value.

```
 t=0.000   key "homepage" expires
 t=0.001   req 1: miss → query DB (takes 200 ms)
 t=0.002   req 2: miss → query DB
 t=0.003   req 3: miss → query DB
   ...       (5,000 req/s for this key)
 t=0.200   req 1000: miss → query DB         ← 1,000 identical expensive queries in flight
 t=0.201   req 1 finishes, sets cache
 t=0.202+  hits resume ... if the DB survived
```

The number of duplicate queries ≈ request rate for the key × time to recompute. A key read 5,000 times per
second whose value takes 200 ms to compute produces about 1,000 concurrent identical queries. And if those
queries slow the database down, recompute time rises, so even more requests pile up: a feedback loop.

"Thundering herd" is the general name for many waiters woken at once to compete for one thing; in caching it
specifically means this burst of simultaneous misses.

### 2.2 Defense 1: request coalescing (single-flight)

Within one process, make concurrent misses for the same key share one fetch. The first caller does the work;
the others wait for its result.

```
inflight = {}   # key → future

get(key):
    v = cache.get(key)
    if v: return v
    with lock:
        if key in inflight:
            fut = inflight[key]          # someone is already fetching
            owner = False
        else:
            fut = inflight[key] = new Future()
            owner = True
    if owner:
        v = db.load(key); cache.set(key, v, ttl)
        fut.complete(v); remove inflight[key]
    return fut.wait()
```

Go's `singleflight` package, Caffeine's `LoadingCache`, and read-through caches in general do this. Many
reverse proxies and CDNs do the same for HTTP (nginx `proxy_cache_lock`, Varnish request coalescing, often
called "collapsed forwarding" in CDNs; topic 14).

Limit: it only coalesces within one process. With 200 app servers you still get up to 200 queries per key,
which is often fine (200 instead of 1,000), but not always.

### 2.3 Defense 2: a distributed lock / lease

Across servers, let only one fetcher rebuild the key. Using Redis:

```
get(key):
    v = cache.get(key)
    if v: return v
    if redis.SET("lock:"+key, my_id, NX, PX=5000):     # acquired
        v = db.load(key)
        cache.set(key, v, ttl)
        redis.DEL("lock:"+key)   (only if still mine)
        return v
    else:
        sleep briefly, then retry cache.get(key)        # or serve stale, see below
```

Facebook's Memcache paper uses a related idea: on a miss, memcached hands out a **lease** token, and only
gives one out per key every few seconds; other clients get "wait and retry". Details to get right:

- The lock must have an expiry (`PX`), or a crashed holder blocks the key forever.
- Waiters must not spin aggressively; back off, and give up after a bound.
- Waiting adds latency for everyone except the rebuilder. Better: serve the old value while one request
  rebuilds (next defense).

Locks are covered in depth in topic 31; here they only need to be "good enough" because the worst case of a
failed lock is a few duplicate queries, not corruption.

### 2.4 Defense 3: serve stale while revalidating

Keep the expired value around and keep serving it while one request refreshes it in the background. This
removes the latency spike entirely. Implementation: store a **soft expiry** inside the value and a longer
**hard TTL** on the cache entry.

```
value stored = { data, soft_expires_at = now + 60s }      cache TTL = 600s

on read:
    if now < soft_expires_at: return data
    else:
        if acquire refresh lock: refresh in background
        return data          # slightly stale, but instant
```

This is exactly HTTP's `stale-while-revalidate` (topic 14). Trade-off: you serve data past its intended
freshness; acceptable for most read-mostly content.

### 2.5 Defense 4: probabilistic early expiration (XFetch)

Instead of everyone missing at the same instant, let each request independently decide, with probability
rising as expiry approaches, to refresh early. One request will almost always refresh before the deadline, and
the herd never forms. The algorithm from Vattani, Chierichetti & Lowenstein ("Optimal Probabilistic Cache
Stampede Prevention", VLDB 2015):

```
stored with each value: delta  = how long the last recompute took
                        expiry = absolute expiry time

on read:
    if now - delta * beta * ln(random()) >= expiry:     # ln(random()) is negative
        recompute and reset expiry
    else:
        return cached value
```

`-ln(random())` is an exponentially distributed random number, usually small, occasionally large. So far from
expiry nobody refreshes; very close to expiry, with expiry minus now comparable to `delta`, someone almost
certainly does. Values that take longer to recompute (`delta` larger) are refreshed earlier. `beta` (default
1) tunes eagerness. No locks, no coordination: an elegant fit for hot keys.

### 2.6 Defense 5: don't expire hot keys on reads at all

For a small set of known-hot, expensive values (the homepage, a config blob), have a background job refresh
them on a schedule and give them no TTL (or a very long one). Reads never miss. The cost is that the job is a
single point of staleness if it dies, so monitor it.

| Defense | Scope | Latency at expiry | Complexity |
|---------|-------|-------------------|------------|
| Coalescing | Per process | Waiters wait | Low |
| Distributed lock/lease | Cluster | Waiters wait (or serve stale) | Medium |
| Stale-while-revalidate | Cluster | None | Medium |
| Probabilistic early expiry | Cluster, no coordination | None (almost always) | Low |
| Background refresh | Known hot keys | None | Low, plus a job to run |

---

## 3. Cache penetration (requests for keys that don't exist)

### 3.1 What happens

Cache-aside only caches values it found. If a request asks for something that does not exist, the database
returns nothing, nothing is cached, and the next identical request goes to the database again. Every such
request **passes through** the cache.

```
GET /product/99999999   → cache miss → DB: no row → nothing cached
GET /product/99999999   → cache miss → DB: no row → nothing cached
... forever
```

Sources: a buggy client looping on a deleted ID; crawlers following stale links; and deliberately, attackers
requesting random non-existent IDs to bypass the cache and hit the database directly. Random keys also defeat
negative caching (each key is new), which is why the second defense exists.

### 3.2 Defense 1: negative caching

Cache the "not found" result too, with a sentinel value and a **short TTL**:

```
v = db.load(id)
if v is None:
    cache.set(key, NOT_FOUND, ttl=60s)       # short
else:
    cache.set(key, v, ttl=1h)
```

Why short: if the item is created later, you don't want the cache to keep claiming it doesn't exist (or
invalidate the negative entry on create). DNS does the same: NXDOMAIN answers are cached for the zone's
negative TTL (topic 04). HTTP caches can cache 404 responses.

Weakness: an attacker using a fresh random key each time never repeats a key, so negative caching never hits,
and it fills your cache with junk sentinel entries, evicting useful data.

### 3.3 Defense 2: a Bloom filter in front

A **Bloom filter** is a compact bit array that answers "is X in the set?" with either **definitely not** or
**probably yes** (topic 50 covers the math; topic 19 shows how LSM-trees use the same trick).

Mechanism in brief:
- An array of m bits, all zero, and k independent hash functions.
- **Add(x)**: compute the k hashes, set those k bit positions to 1.
- **Check(x)**: compute the k hashes; if any of those bits is 0, x was definitely never added. If all are 1,
  x was *probably* added (the bits might have been set by other items: a false positive).

```
bits:   0 1 0 0 1 1 0 0 1 0 1 0
add("p42"): h1=1, h2=5, h3=10 → set bits 1, 5, 10
check("p77"): h1=1, h2=3, h3=8 → bit 3 is 0 → definitely not present → reject without touching cache or DB
```

Use: load all valid product IDs into a Bloom filter (around 10 bits per element gives ~1% false positives, so
100 million IDs fit in ~120 MB, far smaller than the IDs themselves). On every request, check the filter
first; "definitely not" returns 404 immediately.

Properties and caveats:
- **No false negatives**, so legitimate requests are never rejected, *provided every created ID is added to
  the filter before it can be requested.* Adding must be part of the create path.
- False positives are fine: they just fall through to the normal path (cache, then DB).
- **Standard Bloom filters cannot delete.** Deleted IDs remain "probably present", which slowly raises the
  false-positive rate; rebuild periodically, or use counting Bloom filters or cuckoo filters, which support
  deletion.
- It must be kept in sync across servers (stored in Redis via a Bloom module, or rebuilt from the DB).

### 3.4 Other defenses

- **Validate input** before any lookup: IDs outside a valid range or format can be rejected outright.
- **Rate limit** per client (topic 45): random-key scans from one source become obvious.
- **Use unguessable IDs** where enumeration matters (topic 49), so scanning is pointless.

---

## 4. Cache avalanche (mass expiry)

### 4.1 What happens

Many keys expire at the same moment, so many misses hit the backend together. A stampede is one hot key; an
avalanche is many keys at once. Typical causes:

- A batch job or warm-up script populated a million keys at 03:00 with `TTL = 1h`. At 04:00 they all expire
  together, and again every hour if they are refilled at once.
- A deploy changed the key prefix or serialization format, invalidating everything at once.
- The cache cluster (or a large part of it) restarted or failed over and came back empty.

```
expirations per second
    |                 █
    |                 █
    |                 █
    |▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁█▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁   ← all keys set at 03:00 with TTL 1h
    +------------------------------------> time
                     04:00
```

### 4.2 Defense: TTL jitter

Add randomness to each TTL so expirations spread out:

```
ttl = base_ttl + random(0, base_ttl * 0.1)      # e.g. 3600 + [0, 360) seconds
```

Instead of a million expirations in one second you get roughly 2,800 per second spread over six minutes, and
since refills now happen at different times, subsequent cycles stay spread out too. This is cheap and almost
always worth doing.

```
expirations per second (with jitter)
    |
    |                 ▃▃▃▃▃▃
    |▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁██████▁▁▁▁▁▁▁▁▁▁▁▁
    +------------------------------------> time
                     04:00–04:06
```

Jitter is the same idea used for retry backoff (topic 43): randomness breaks synchronization.

### 4.3 Defenses against the other causes

- **High availability of the cache itself**: replicas and automatic failover (Redis Sentinel/Cluster), and
  spreading keys over many nodes (topic 13) so that losing one node loses only 1/N of the keys.
- **Graceful degradation**: if the cache is unavailable, protect the database with **circuit breakers and
  load shedding** (topic 44): serve degraded responses or errors for a fraction of requests rather than
  letting all of them pile onto the DB. A slow DB helps no one.
- **Rate-limit backend refills**: cap concurrent DB queries from the cache-miss path (a semaphore/bulkhead).
- **Change key formats gradually**: dual-read or roll out to a percentage of traffic, never flip all keys at once.
- The stampede defenses (serve stale, coalescing) also blunt an avalanche.

---

## 5. Hot keys

### 5.1 What happens

In a distributed cache, each key lives on one node (or one primary, topic 13). If one key receives an enormous
share of traffic, say a celebrity's profile, the item in a flash sale, or a global config read on every
request, that single node is overwhelmed while the others idle. Adding more nodes does not help: the key still
maps to one place.

```
          node A     node B     node C     node D
load:     ███        ███        ███████████████████████   ███
                                ^ "product:flash-sale" lives here
```

Symptoms: one cache node at 100% CPU or saturating its network link, high latency only for requests involving
that node. Redis being single-threaded per process makes the ceiling per key sharp (on the order of 100k+
simple ops/s per instance). Large values make it worse: a 500 KB hot key at 20k reads/s is 10 GB/s of
network, far beyond one NIC.

Finding them: Redis `--hotkeys` (with LFU policy), `MONITOR` sampling (carefully), client-side counting,
per-key metrics in proxies.

### 5.2 Defense 1: local (in-process) cache in front

Cache the hot key in each app server's memory for a short time (one to a few seconds). With 200 app servers
and a 1 s local TTL, the distributed cache sees at most ~200 reads/s for that key, regardless of user traffic.

```
request → [local cache, TTL 1-5 s] → [Redis] → [DB]
           absorbs the hot key       sees ~N_servers reads/s per hot key
```

Trade-off: up to the local TTL of staleness, and possible disagreement between servers (topic 11). For
hot read-mostly keys this is almost always acceptable. Some client libraries detect hot keys automatically and
cache only those locally. Redis 6+ offers **client-side caching** with server-assisted invalidation
(`CLIENT TRACKING`): the server tells clients when keys they cached change.

### 5.3 Defense 2: key replication (spreading one key over many nodes)

Store N copies of the hot key under different names that hash to different nodes, and have readers pick one at
random:

```
write:  for i in 0..N-1: cache.set("product:42#" + i, value)
read:   cache.get("product:42#" + random(0, N-1))
```

Read load divides by N. Cost: writes and invalidations must touch all N copies (and can be briefly
inconsistent between them). Alternatively, add read replicas to the cache shard holding the key and spread
reads across them, which achieves the same with infrastructure instead of key naming.

### 5.4 Other mitigations

- **Split big values**: break a large hot value into pieces, or cache only the fields actually needed.
- **Precompute and push**: for truly global data (feature flags, config), push it to every server on change
  rather than reading it per request.
- **Hot keys on the write side** (a single counter incremented by everyone) are a different problem: shard
  the counter into N sub-counters and sum on read, or batch increments locally (topic 25 covers hot partitions
  in databases).

---

## 6. Cold starts and cache warming

### 6.1 What happens

A cache starts empty: after a fresh deploy of a new cluster, after a restart with no persistence, after a
failover to a replica that held nothing, after a region failover where the standby region's cache was idle,
or after a key-format change. Hit ratio is 0% and climbs only as traffic fills it. During that time the
backend sees the full read load it was never sized for: an avalanche of the entire keyspace.

```
hit ratio
100% |                       ______________
     |                 _____/
     |            ____/
     |       ____/          ← the backend carries everything in this region
     |   ___/
  0% |__/
     +-------------------------------------> time after restart
```

### 6.2 Defenses

- **Persistence**: Redis with RDB/AOF (topic 11) restarts with its data. Stale-ish data is far better than none
  (pair with TTLs).
- **Warm before taking traffic**: replay a sample of recent production reads, or preload the known top-N keys
  (from access logs or a hit-tracking table), before the node joins the pool.
- **Warm from a peer**: Facebook's "cold cluster warmup": a new cache cluster, on a miss, reads from an already
  warm cluster instead of the database, until its own hit ratio is high enough.
- **Shift traffic gradually**: send 1%, 5%, 20%... of traffic to the new cache (or new region), letting it
  fill while the backend only sees a fraction of the miss load. This is the same principle as canary
  deploys (topic 47).
- **Avoid flushing everything**: prefer rolling restarts of cache nodes; with consistent hashing (topic 13)
  each restarted node loses only its own share of keys.
- **Protect the backend regardless**: concurrency limits on refills, load shedding (topic 44), and coalescing,
  so a cold cache causes slowness rather than collapse.

### 6.3 Capacity planning implication

If a cold start would take the backend down, the system's real capacity is the backend's capacity, and the
cache is a single point of failure. Either size the backend to survive some plausible cold-start rate (for
example, losing one cache node out of N), or invest in the warming and admission-control mechanisms above.
Knowing which you have chosen is the point.

---

## 7. Putting it together

| Failure mode | Trigger | Signature | Primary defenses |
|--------------|---------|-----------|------------------|
| Stampede | One hot key expires | Burst of identical queries | Coalescing, locks/leases, serve stale, probabilistic early expiry |
| Penetration | Requests for non-existent keys | Steady misses that never become hits | Negative caching, Bloom filter, validation, rate limiting |
| Avalanche | Many keys expire / cache lost | Hit ratio cliff | TTL jitter, cache HA, circuit breakers, gradual key changes |
| Hot key | One key with extreme traffic | One node saturated | Local cache, key replication, read replicas |
| Cold start | Empty cache | Hit ratio climbs from zero | Persistence, warming, gradual traffic shift |

A sensible default setup for a cache-aside Redis cache: TTLs with jitter on every key; negative caching with
short TTLs; single-flight coalescing in the client library; serve-stale or early refresh for known expensive
keys; a small in-process cache for the hottest keys; and a concurrency limit plus circuit breaker on the miss
path to the database. Monitor hit ratio, miss rate to the backend, per-node load, and evictions.

---

## 8. How it connects

- **Topic 02**: queueing theory explains why a modest increase in backend load becomes a latency explosion.
- **Topic 11**: the patterns (cache-aside, read-through) and Redis behavior these failures arise from.
- **Topic 13**: consistent hashing limits how many keys move or vanish when cache nodes change, which limits
  avalanches and cold starts; it also explains why a hot key stays on one node.
- **Topic 14**: CDNs face the same problems at the edge: request collapsing, `stale-while-revalidate`, origin
  shields, cache warming.
- **Topic 25**: hot partitions in databases are the storage-layer version of hot keys.
- **Topic 31**: distributed locks and leases, done properly.
- **Topic 43, 44**: retries amplify these failures; jitter, backoff, circuit breakers and load shedding contain them.
- **Topic 45**: rate limiting to stop penetration attacks.
- **Topic 50**: Bloom, counting Bloom and cuckoo filters in depth.

## Common misconceptions

- "If the cache is up, the cache is fine." → Most of these failures happen with the cache healthy; the damage
  is in the miss traffic reaching the backend.
- "A distributed lock fully solves stampedes." → It limits duplicate queries but makes waiters wait, needs
  expiry to survive crashed holders, and adds a dependency. Serving stale often works better.
- "Negative caching stops penetration attacks." → Not with random keys: each key is new. A Bloom filter or
  input validation is needed.
- "Bloom filters can give false negatives." → They cannot (if every valid key was added); they give false
  positives, and standard ones cannot delete.
- "Adding cache nodes fixes a hot key." → A single key lives on one node; more nodes don't spread it. You need
  local caching or replication of that key.
- "TTL jitter is a micro-optimization." → It is the main defense against synchronized mass expiry and costs
  one line of code.
- "A restarted cache just means a few slow minutes." → If the backend is sized for the warm hit ratio, a cold
  cache can take it down and keep it down (metastable failure).

## Further reading (supplements)

- Nishtala et al., *Scaling Memcache at Facebook* (NSDI 2013): leases for stampedes and stale sets, gutter
  pools, cold cluster warmup.
- Vattani, Chierichetti & Lowenstein, *Optimal Probabilistic Cache Stampede Prevention* (VLDB 2015).
- Bronson et al., *Metastable Failures in Distributed Systems* (HotOS 2021).
- RFC 5861, *HTTP Cache-Control Extensions for Stale Content* (`stale-while-revalidate`, `stale-if-error`).
- Bloom, *Space/Time Trade-offs in Hash Coding with Allowable Errors* (1970); Kleppmann DDIA Ch. 3 for Bloom
  filters in storage engines.
- Go `golang.org/x/sync/singleflight` documentation (request coalescing in practice).

---

## Questions

1. A key is read 3,000 times per second and takes 300 ms to rebuild. Roughly how many duplicate database
   queries does a naive cache-aside setup issue when it expires? Why can the number be even worse in practice?
2. Compare request coalescing within a process with a distributed lock across the cluster. When is
   per-process coalescing enough?
3. Explain how probabilistic early expiration prevents a stampede without any coordination between servers.
   Why does it refresh expensive values earlier?
4. Why is "serve stale while one request refreshes" often better than "everyone waits for the lock holder"?
   What do you give up?
5. An attacker sends requests for random product IDs that don't exist. Your cache hit ratio for those is zero.
   Why doesn't negative caching help, and what does?
6. You add a Bloom filter of valid product IDs. A merchant creates a product and the first request for it
   returns 404. What went wrong? And what happens over months as products are deleted?
7. A warm-up script loads 2 million keys at 02:00 with a 1-hour TTL. Describe what the database sees over the
   following night and fix it.
8. During a flash sale, one Redis node hits 100% CPU while the other eleven are at 10%. Why didn't
   partitioning spread the load, and what are two fixes?
9. You put a 2-second in-process cache in front of Redis for hot keys. What new consistency behavior do users
   see, and why is it usually acceptable?
10. Your cache cluster is replaced by a new one with no data. Describe a rollout that avoids overloading the
    database.
11. Why can a system stay broken after a cache problem even once the cause is fixed? What breaks the loop?
12. Which of the five failure modes does TTL jitter help with, and which does it not help with at all?
13. A team argues the cache doesn't need monitoring because "it's just a cache". Which metrics would have
    warned them about each failure mode?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** About rate × rebuild time = 3,000 × 0.3 = ~900 concurrent identical queries. In practice the burst slows
the database, so rebuild time increases, letting even more requests miss before the first refill completes;
clients may also time out and retry, adding more.

**2.** Coalescing makes concurrent misses in one process share a single fetch; a distributed lock (or lease)
ensures one fetch across all servers. Per-process coalescing caps duplicates at roughly the number of app
servers; when that number of queries per expiry is cheap for the database, it is enough and avoids the
complexity and latency of a distributed lock. This is also why read-through caches and proxies with request
collapsing are so effective.

**3.** Each request independently decides to refresh with a probability that grows as expiry approaches
(`now − delta·beta·ln(rand) ≥ expiry`). Far from expiry the chance is negligible; near expiry some single
request almost certainly refreshes first, resetting the expiry before the herd forms. The threshold scales
with `delta`, the recompute time, so slow-to-compute values start refreshing earlier, leaving enough time to
finish before the real expiry.

**4.** With serve-stale, nobody waits: all requests get the previous value instantly while one refresh runs in
the background, so latency doesn't spike and there's no lock contention or retry loop. You give up freshness:
for a short time after nominal expiry users see the old value.

**5.** Each random key is requested once; a negative entry for it is never reused, and caching them all fills
the cache with junk that evicts real data. Fixes: a Bloom filter of valid IDs to reject definitely-missing
keys before cache and DB; input validation; rate limiting per client; unguessable IDs.

**6.** The new ID was not added to the Bloom filter before it became requestable (or the filter on that server
was not updated), causing a false negative. The create path must add to the filter first. Over months, deleted
IDs remain set in the filter (standard Bloom filters can't delete), so the false-positive rate creeps up and
more junk requests fall through; rebuild periodically or use a counting/cuckoo filter.

**7.** At 03:00 all 2 million keys expire together and every subsequent read misses: a huge spike of DB queries
(an avalanche), and since they're refilled together, it repeats roughly every hour. Fix: add jitter to TTLs
(e.g. 3600 + random 0–600 s), and spread the warm-up itself over time.

**8.** A single key maps to exactly one node, so all traffic for the flash-sale item lands there; partitioning
spreads *keys*, not load within a key. Fixes: an in-process local cache with a short TTL in the app servers;
replicating the key under N suffixed names across nodes (or adding read replicas for that shard); reducing
value size.

**9.** Up to 2 seconds of staleness after a change, and different servers may briefly disagree, so a user
could see a new value then an old one. Usually fine because hot keys are read-mostly data (product details,
config) where a couple of seconds of staleness is invisible; correctness-critical paths (checkout) re-read
authoritative data.

**10.** Keep the old cluster serving; warm the new one by replaying recent reads or loading top-N keys; or make
the new cluster fall back to the old cluster on misses. Then shift traffic gradually (1% → 10% → 50% → 100%)
watching hit ratio and DB load. Keep concurrency limits and circuit breakers on the DB miss path throughout.

**11.** Metastability: overload makes refill queries time out, so the cache stays empty, so the database stays
overloaded; client retries add even more load. Breaking the loop requires shedding load: circuit breakers,
rejecting a fraction of traffic, limiting concurrent DB queries, stopping retries, or warming the cache from
elsewhere, so the backend can recover and refill the cache.

**12.** It helps against avalanches caused by synchronized expiry (and somewhat reduces simultaneous refill
after warm-up). It does nothing for penetration (keys never cached), hot keys (load on one node), stampedes on
a single key (that key still expires at some moment; you need coalescing/early refresh), or a cold start from
an empty cache.

**13.** Hit ratio and miss rate to the backend (avalanche, cold start, penetration); DB QPS and latency
from the miss path (all); per-key or per-node request rate and CPU/network (hot keys); rate of not-found
results (penetration); expirations per second (avalanche); evictions (cache too small); duplicate in-flight
loads per key (stampede).

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What is the common thread in all cache failure modes?
**A:** A spike of miss traffic reaching a backend that was sized for the warm hit ratio.

**Q:** Define cache stampede.
**A:** A hot key expires and many concurrent requests miss and recompute it simultaneously.

**Q:** Approximate number of duplicate queries in a stampede?
**A:** Request rate for the key × time to recompute.

**Q:** What is request coalescing (single-flight)?
**A:** Concurrent misses for the same key in one process share a single backend fetch.

**Q:** What must a stampede lock always have?
**A:** An expiry, so a crashed holder doesn't block the key forever.

**Q:** What is a lease in Facebook's Memcache?
**A:** A token issued on a miss, rate-limited per key, that lets one client refill; deletes invalidate it so stale sets fail.

**Q:** Stale-while-revalidate?
**A:** Keep serving the expired value while one request refreshes in the background.

**Q:** Probabilistic early expiration in one line?
**A:** Each read refreshes early with probability rising near expiry, scaled by recompute time, so one request refreshes before the herd.

**Q:** Define cache penetration.
**A:** Requests for keys that don't exist, which are never cached and always hit the database.

**Q:** Negative caching?
**A:** Caching a "not found" sentinel with a short TTL.

**Q:** Why does negative caching fail against random-key attacks?
**A:** Each key is new so nothing is reused, and the junk entries evict real data.

**Q:** Bloom filter answers?
**A:** "Definitely not in the set" or "probably in the set."

**Q:** Can a Bloom filter give false negatives?
**A:** No, as long as every member was added. It gives false positives only.

**Q:** Main operational caveat of a standard Bloom filter?
**A:** It can't delete; new items must be added before they're requested; rebuild or use counting/cuckoo filters.

**Q:** Define cache avalanche.
**A:** Many keys expire or vanish at once, causing a cliff in hit ratio and a flood of backend queries.

**Q:** Main defense against synchronized expiry?
**A:** TTL jitter: add a random offset to each TTL.

**Q:** Define hot key.
**A:** A single key with so much traffic that the one cache node holding it saturates.

**Q:** Why doesn't adding nodes fix a hot key?
**A:** A key maps to one node; partitioning spreads keys, not the load of one key.

**Q:** Two hot-key fixes?
**A:** Short-TTL in-process local cache; replicate the key under N names (or read replicas) and read a random copy.

**Q:** Hot write key fix?
**A:** Shard the counter into sub-counters summed on read, or batch increments locally.

**Q:** Define cold start.
**A:** An empty cache after restart, new cluster, failover or key change, so the backend carries all reads.

**Q:** Three cache warming techniques?
**A:** Preload top-N keys or replay recent reads; read from a warm peer cluster on miss; shift traffic gradually.

**Q:** What is a metastable failure?
**A:** The system stays overloaded after the trigger is gone, because overload itself keeps the cache empty and retries add load.

**Q:** How do you break a metastable cache failure?
**A:** Shed load (circuit breakers, rejecting traffic, limiting DB concurrency, stopping retries) so the backend recovers and the cache refills.
