# 02 — Where Time Goes: Latency Hierarchy & Queueing

> Part 1 — Foundations · Prerequisites: topic 01

Topic 01 defined latency, throughput and tail latency. This topic asks *where the time actually goes*.
There are two answers. First, **physics and hardware**: touching RAM, an SSD, a spinning disk, or a
machine on another continent costs wildly different amounts of time, spanning about eight orders of
magnitude. Second, **waiting**: most of the latency in a busy system is not work at all, it is requests
standing in line behind other requests. Almost every design technique later in this plan (caching,
append-only storage, batching, replication close to users, load shedding, hedged requests) is a response
to one of these two facts. If you know the numbers and the queueing curve, many design decisions stop
looking arbitrary.

By the end of this topic you should be able to:

1. Recall the rough latency of CPU cache, RAM, SSD, disk, same-datacenter and cross-region operations,
   and the ratios between them.
2. Explain why sequential I/O is so much faster than random I/O on both disks and SSDs, and name the
   storage designs that exist because of it.
3. Explain why waiting time explodes as utilization approaches 100%, and why variability makes it worse.
4. Apply Little's Law to size pools, interpret queue lengths, and predict what a slow dependency does
   to a service.
5. Explain tail-latency amplification under fan-out and the techniques used to fight it.

---

## 1. The latency numbers

### The table

These are order-of-magnitude figures for modern server hardware (the classic list popularized by Jeff
Dean and Peter Norvig, updated). Exact values vary by hardware generation; **the ratios are what you
should memorize**.

| Operation | Approx. time | Notes |
|---|---|---|
| L1 cache reference | ~1 ns | A few CPU cycles |
| Branch mispredict | ~3 ns | Pipeline flush |
| L2 cache reference | ~4 ns | |
| L3 cache reference | ~10–40 ns | Shared across cores |
| Uncontended mutex lock/unlock | ~20 ns | Contended: microseconds or worse |
| Main memory (RAM) reference | ~100 ns | ~100× L1 |
| Compress 1 KB with a fast codec (Snappy/LZ4) | ~2 µs | |
| Read 1 MB sequentially from RAM | ~10–50 µs | Memory bandwidth tens of GB/s |
| Random 4 KB read from NVMe SSD | ~20–100 µs | ~1,000× RAM |
| Round trip within a datacenter / zone | ~0.1–0.5 ms | Same building, a few switches |
| Read 1 MB sequentially from NVMe SSD | ~0.2–1 ms | Several GB/s |
| Round trip between availability zones | ~1–2 ms | Nearby buildings, same region |
| Spinning disk (HDD) seek + rotation | ~5–10 ms | Mechanical |
| Read 1 MB sequentially from HDD | ~5 ms | ~150–250 MB/s |
| Round trip, same continent (e.g. US east ↔ west) | ~60–80 ms | |
| Round trip, cross-ocean (e.g. Europe ↔ US east) | ~70–100 ms | |
| Round trip, far side of the world | ~150–300 ms | |
| TCP + TLS connection setup across an ocean | ~200–300 ms | Several round trips (topics 03, 05) |

### Make it human-sized

Multiply everything by a billion, so 1 ns becomes 1 second:

```
L1 cache            1 s        reaching for a pen on your desk
RAM                 ~2 min     walking to the next room
NVMe SSD read       ~1 day     a trip to another city
Same-DC round trip  ~4 days
HDD seek            ~3 months
Cross-ocean RTT     ~3 years
```

That picture is the reason caches exist. A request that does one cross-region round trip has spent,
in CPU terms, an eternity. A request that does ten sequential database round trips within a datacenter
has spent several milliseconds just on the network, before any work.

### Where the network numbers come from

Light in optical fiber travels at about **200,000 km/s**, roughly two-thirds of its speed in vacuum.
That gives a hard floor:

```
one-way:      ~5 µs per km      →  1 ms per 200 km
round trip:   ~1 ms per 100 km of distance

New York ↔ London ≈ 5,600 km  →  physics floor ≈ 56 ms RTT; real ≈ 70–80 ms
```

Real paths are longer than great-circle distance and pass through routers and switches, so actual
times are higher. **No amount of engineering removes this floor.** The only fixes are to move the data
closer to the user (CDNs, topic 14; multi-region replicas, topic 48) or to do fewer round trips
(connection reuse, topic 03; TLS session resumption, topic 05; batching requests).

Within a datacenter, round trips are dominated not by distance but by the software stack: kernel
network processing, serialization, the remote process waking up and scheduling the work. That's why
an intra-datacenter RPC is ~100s of µs even though light could cross the building in under 1 µs.

### What the numbers imply

- **Memory is ~1,000× faster than SSD and ~100,000× faster than an HDD seek.** Serving hot data from
  RAM (Redis, a database's buffer pool, the OS page cache) is the single biggest latency lever there
  is (topic 11).
- **A network hop within a datacenter costs about as much as several SSD reads.** Calling a remote
  cache is not free; a local in-process cache can be 1,000× faster than a remote one for small values.
- **Cross-region round trips dwarf everything else.** A design that needs synchronous agreement across
  regions (topics 29, 30) pays ~100 ms per write, at minimum.
- **Round trips in series add up.** The "N+1 queries" problem (one query per item in a list) turns a
  1 ms query into 100 ms for 100 items purely through repeated round trips.
- **Compression is cheap relative to the network.** Spending ~2 µs to compress 1 KB is worth it if it
  saves sending data across a slow or expensive link.

---

## 2. Sequential vs random I/O

The second column of the table hides an important pattern: reading the *same amount* of data can be
fast or slow depending on whether it is **contiguous** (sequential) or **scattered** (random).

### Spinning disks: mechanics

An HDD has a rotating platter and a moving arm. To read a block, the arm must **seek** to the right
track (several ms), then wait for the block to **rotate** under the head (at 7,200 RPM a full rotation
is 8.3 ms, so ~4 ms on average). Only then does data stream off at full speed.

```
random 4 KB reads:      [seek+rotate ~8 ms][read 4KB] [seek+rotate ~8 ms][read 4KB] ...
                        → ~100–200 reads/s → ~0.5 MB/s

sequential read:        [seek+rotate ~8 ms][read read read read read read ...........]
                        → ~200 MB/s
```

Same disk, same data volume, roughly a **400× difference**. For decades this was *the* fact of storage
design.

### SSDs: different reasons, same lesson (mostly)

SSDs have no moving parts, so random **reads** are fast (tens of µs) and the gap between random and
sequential reads is much smaller, especially when many reads are in flight at once (SSDs are internally
parallel across many flash chips). But flash has its own quirks that make **writes** prefer
sequential patterns:

- Flash is read and written in **pages** (typically 4–16 KB) but can only be **erased** in much larger
  **blocks** (often MBs). A page can't be overwritten in place; it must be erased first.
- The SSD's firmware (the flash translation layer, FTL) therefore writes updates to fresh pages and
  marks old ones invalid. Later, **garbage collection** copies the still-valid pages out of a block so
  it can erase it.
- Small random writes scatter invalid pages across many blocks, so garbage collection must copy more
  data: this is **write amplification**. It costs throughput, wears out the flash faster, and causes
  latency spikes when GC runs.
- Large sequential writes fill and invalidate whole blocks together, which is cheap to clean up.

### RAM too

Even memory prefers sequential access. The CPU loads memory in 64-byte **cache lines** and hardware
**prefetchers** detect sequential patterns and fetch ahead. Scanning an array runs at memory bandwidth;
chasing pointers through a linked list pays ~100 ns per hop because each next address is unknown until
the current load finishes.

### Why it shapes storage design

A great deal of storage engineering is the art of **turning random I/O into sequential I/O**:

| Design | How it exploits sequential I/O | Topic |
|---|---|---|
| **Write-ahead log (WAL)** | Every change is appended to a log first (sequential), then applied to data pages lazily. The commit only waits on the append. | 17 |
| **LSM-trees** (Cassandra, RocksDB) | Writes go to memory, then are flushed as large sorted files written sequentially; background compaction merges them, also sequentially. | 19 |
| **B-trees** (Postgres, InnoDB) | Large pages (8–16 KB) and high fan-out keep the tree 3–4 levels deep, minimizing random reads per lookup. | 17 |
| **Column stores** | Store each column contiguously so analytic scans read only the needed columns, sequentially. | 20 |
| **Log-based messaging** (Kafka) | Partitions are append-only files; consumers read sequentially, mostly from the OS page cache. | 37 |
| **Group commit / batching** | Many small writes become one larger sequential write and one fsync. | 01, 17 |
| **Read-ahead** | The OS fetches the next blocks before you ask, betting that access is sequential. | — |

The trade-off: append-only designs must eventually **clean up** (compaction, garbage collection,
checkpointing), which consumes background I/O and can cause latency spikes of its own. You pay later,
in bulk, instead of now, per write.

---

## 3. Queueing basics: utilization vs wait time

### The setup

Any resource that does work (a CPU core, a disk, a database connection, a single-threaded Redis
process, a worker pool) can be modelled as a **queue** in front of one or more **servers** (anything
that processes requests):

```
 arrivals (λ per second)        queue (waiting)          server(s)
 ──▶ ● ──▶ ● ──▶ ● ──▶   [ ● ● ● ● ]   ──▶   [ busy: service time S ]  ──▶ done
```

- **λ (lambda)**: arrival rate, e.g. 80 requests/s.
- **S**: average service time, e.g. 10 ms. The server can therefore complete at most **μ = 1/S** = 100
  requests/s.
- **ρ (rho), utilization** = λ / μ = λ × S. The fraction of time the server is busy. Here 0.8 = 80%.

If ρ ≥ 1, work arrives faster than it can be done and the queue grows without bound. That much is
obvious. The non-obvious part is what happens **below** 100%.

### Why there's waiting even when there's spare capacity

If requests arrived perfectly evenly (one exactly every 12.5 ms) and each took exactly 10 ms, there
would be no queue at 80% utilization: each request would find the server idle. Real arrivals are
**random**: sometimes three arrive close together, then nothing for a while. Real service times vary
too. The server can't "bank" its idle time for later, so idle periods are lost forever while bursts
form queues.

```
time ──────────────────────────────────────────────────────▶
arrivals:   ●  ●●●        ●    ●●   ●            ●●●●
server:     ■  ■■■■■■■■   ■    ■■■■ ■            ■■■■■■■■■■■
                ↑ queue                           ↑ queue
            idle gaps here are wasted capacity that can't absorb the bursts later
```

The busier the server, the less likely there's an idle gap to absorb the next burst, so queues form
more often and last longer.

### The M/M/1 curve

The simplest model, **M/M/1** (random "Markovian" arrivals, random service times, 1 server), gives:

```
Time in system:   W  = S / (1 − ρ)
Time waiting:     Wq = S × ρ / (1 − ρ)
Number in system: L  = ρ / (1 − ρ)
```

Worked example: a server with S = 10 ms (capacity 100 req/s).

| Load | ρ | Avg time in system W | Avg requests in system L |
|---|---|---|---|
| 50 req/s | 50% | 20 ms | 1 |
| 70 req/s | 70% | 33 ms | 2.3 |
| 80 req/s | 80% | 50 ms | 4 |
| 90 req/s | 90% | 100 ms | 9 |
| 95 req/s | 95% | 200 ms | 19 |
| 99 req/s | 99% | 1,000 ms | 99 |
| 101 req/s | 101% | grows forever | grows forever |

Going from 50% to 90% load (1.8× the traffic) makes latency 5× worse. Going from 90% to 99% (10% more
traffic) makes it 10× worse again. Note how *unforgiving* the last stretch is: a small traffic bump or
a small slowdown in service time near the top of the curve produces a huge latency change.

```
W / S
100 │                                          │
    │                                          │
 50 │                                         ╱
    │                                        ╱
 20 │                                      ╱
 10 │                                  _.-'
  5 │                           _..--''
  2 │ ______________....----''''
  1 └──────────────────────────────────────────── ρ
    0%        50%        80%   90%  95%   99%
```

### Variability makes it worse: Kingman's formula

M/M/1 assumes a specific amount of randomness. A more general approximation for a single server,
**Kingman's formula**, separates the three causes of waiting:

```
Wq ≈  ( ρ / (1 − ρ) )  ×  ( (ca² + cs²) / 2 )  ×  S
       utilization        variability            service time

ca = coefficient of variation of inter-arrival times  (std dev / mean)
cs = coefficient of variation of service times
```

- The **utilization** term is the hockey stick above.
- The **variability** term scales it. Bursty traffic (ca high) or erratic service times (cs high: some
  requests hit cache, some do a full table scan; occasional GC pauses) multiply the wait.
- Perfectly constant service times (cs = 0) with random arrivals halve the wait compared to M/M/1.

So there are three levers to reduce waiting: **lower utilization** (add capacity), **reduce
variability** (smooth arrivals, isolate slow request types, avoid GC pauses), and **reduce service
time** (make each request cheaper).

### Multiple servers and pooling

With *c* servers sharing **one** queue, a request only waits if *all* servers are busy, which is much
rarer. Compare at 80% utilization, S = 10 ms:

```
Two separate queues (each M/M/1 at 80%):     W ≈ 50 ms
One shared queue, two servers (M/M/2, 80%):  W ≈ 28 ms
```

Same hardware, same load, nearly half the latency. This is why a single shared work queue feeding a
pool beats pinning requests to specific workers, why supermarkets with one snake line feel faster than
per-checkout lines, and why load balancers that pick idle backends (least connections, power of two
choices, topic 09) beat random assignment. Big pools can also run at higher utilization for the same
wait than small ones; the hockey stick moves right as *c* grows (but it's still a hockey stick).

### Why systems fall over near 100%

The curve alone explains slowness. Collapse needs a feedback loop, and real systems have several:

```
t0  Load at 85%. p99 fine.
t1  Traffic spike pushes ρ to 98%. Queue grows; latency 10× higher.
t2  Clients hit their timeouts (say 1 s) and give up. But the server already
    queued those requests and will still process them: wasted work.
t3  Clients retry. Arrival rate is now original load + retries. ρ > 100%.
t4  The queue grows without bound. Nearly every request now times out before
    the server reaches it. Server is 100% busy doing work nobody is waiting for.
t5  The spike ends, but retries keep the load above capacity. The system stays
    down until load is cut far below normal.
```

The state at t5 is called a **metastable failure**: the trigger is gone, but the system is stuck in a
bad state sustained by its own feedback (retries, cache misses after a restart, timeouts). Throughput
of *useful* work (**goodput**) falls toward zero even though the server is maximally busy.

Other amplifiers of the same shape: garbage collection working harder as memory fills with queued
requests; thread-per-request servers context-switching between thousands of threads; locks held longer
under load, so more contention.

Defences (covered properly later):

- **Headroom**: run latency-sensitive resources at moderate utilization (often 50–70%).
- **Bounded queues**: an unbounded queue doesn't prevent overload, it just converts it into latency
  and memory exhaustion. Reject early instead.
- **Load shedding** and **backpressure** (topic 44): refuse work you can't finish in time, and tell
  upstream to slow down.
- **Timeouts, retry budgets and exponential backoff with jitter** (topic 43): stop clients from
  multiplying the load.
- **Deadline propagation**: drop queued requests whose caller has already given up. Some systems even
  serve the queue **LIFO** under overload, so fresh requests (whose callers are still waiting) succeed.

---

## 4. Little's Law

### The statement

```
L = λ × W

L = average number of items in the system
λ = average arrival rate (= throughput, when the system is stable)
W = average time an item spends in the system
```

It was proved by John Little in 1961, and what makes it powerful is how little it assumes: **any**
stable system, **any** arrival pattern, **any** service-time distribution, any scheduling order. You
can draw the "system" boundary anywhere: a single thread pool, a whole service, a Kafka consumer group,
a checkout line. It's an accounting identity, not a model, so it is always true on average over a long
enough window.

Rearranged, it answers three different questions:

```
L = λ × W     How much concurrency do I need?
λ = L / W     What's my max throughput, given limited concurrency?
W = L / λ     How long will items wait, given a queue length and a drain rate?
```

### Sizing pools

A service makes a downstream call that takes 40 ms on average, at 500 calls/s.

```
needed concurrency  L = 500 × 0.040 = 20 connections in use, on average
```

Give it some headroom for variance (the instantaneous count fluctuates around the average) and a pool of
~30–40 is plenty. A pool of 500 adds nothing but memory and load on the database: extra connections
can't increase throughput if latency is the same, and too many concurrent queries can actually slow a
database down through contention.

### What a slow dependency does

Same service, same pool of 40 connections. The downstream database slows from 40 ms to 2 s per call
(a lock, a bad query plan, a disk problem).

```
max throughput = L / W = 40 / 2 s = 20 calls/s     (was 1,000 calls/s possible)
incoming load  = 500 calls/s
```

480 requests/s can't get a connection. They queue inside the service, holding threads and memory,
which means the service *itself* stops responding to *its* callers, whose pools fill up in turn. This
is how one slow component causes a **cascading failure** across services. Little's Law says it
precisely: **when latency rises and concurrency is capped, throughput must fall**. The defences are
timeouts (so W can't grow unboundedly, topic 43), bulkheads (separate pools per dependency so one slow
dependency can't take all threads) and circuit breakers (topic 44).

Note the asymmetry: a *slow* dependency is often worse than a *dead* one. A dead one fails fast and
frees resources; a slow one holds them.

### Interpreting a queue

A Kafka consumer group has 60,000 messages of lag and processes 1,000 messages/s.

```
W = L / λ = 60,000 / 1,000 = 60 s
```

A message produced now will be processed in about a minute. The same reasoning turns any queue depth
into an expected delay, which is far more meaningful for alerting than the raw depth: "500 items
queued" is fine at 10,000 items/s and terrible at 1 item/s.

### Open vs closed systems

- In an **open** system, new requests arrive regardless of how slow you are (users on the internet,
  incoming events). Overload grows queues.
- In a **closed** system, a fixed number of clients each wait for a response before sending the next
  (a load test with 50 threads, a batch job with a fixed worker pool). Here L is fixed, so if W rises,
  λ *must* fall.

This is why a closed-loop load test can be misleading about a real, open system: when the server slows
down, the test automatically sends less, hiding the overload (this is the coordinated omission problem
from topic 01).

---

## 5. Tail-latency amplification under fan-out

### The problem

Topic 01 introduced the formula. Here's why it matters so much. Many requests are served by
**fanning out** to many servers in parallel and waiting for all of them: a search query touching every
index shard, a page that calls 30 microservices, a scatter-gather query across database partitions
(topic 25).

```
                   ┌──▶ shard 1   10 ms
                   ├──▶ shard 2   12 ms
user ──▶ frontend ─┼──▶ shard 3    9 ms        response time = max(...) = 800 ms
                   ├──▶  ...
                   └──▶ shard 100 800 ms  ← one GC pause
```

The response time is the **maximum** of the parallel calls. If each call independently has probability
*p* of being slow:

```
P(request is slow) = 1 − (1 − p)^N
```

| Fan-out N | p = 1% (backend p99) | p = 0.1% (backend p99.9) |
|---|---|---|
| 1 | 1% | 0.1% |
| 10 | 9.6% | 1% |
| 100 | 63% | 9.5% |
| 1,000 | ~100% | 63% |

At a fan-out of 100, the backend's p99 is roughly the user's *median*. At a fan-out of 1,000, even the
backend's p99.9 dominates. The more you parallelize, the more your user-facing latency is determined
by the *rare* slow events on the backend, not the common fast ones.

The serial case is bad in a different way. If a request calls services **one after another**, their
latencies **add**, and the chance of hitting at least one slow call also grows with the number of calls.
Deep call chains and wide fan-outs both turn rare backend slowness into common user slowness.

### Where the slow responses come from

Tail latency is rarely caused by "the code is slow." It is usually caused by something *temporary* on
one machine:

- Queueing (section 3) during a local burst.
- Garbage collection pauses.
- Background work: LSM compaction (topic 19), checkpoints, log rotation, backups, SSD garbage collection.
- Shared resources: noisy neighbours on the same host, contended locks, network switch buffers.
- Cache misses: a request that misses cache goes to disk.
- Retransmitted packets (a lost TCP packet can add ~200 ms+ for a retransmission timeout, topic 03).
- CPU throttling, power management, or the process simply being descheduled.

You can't eliminate all of these, so large systems learn to be **tail-tolerant**: produce fast
responses even when some components are slow.

### Techniques

- **Hedged requests**: send the request to one replica; if it hasn't answered within, say, the p95
  latency, send a duplicate to another replica and use whichever answers first. Because you only hedge
  the slowest ~5%, extra load is about 5%, but the tail shrinks dramatically. (*The Tail at Scale*
  reports a Google benchmark where hedging after 10 ms cut the p99.9 latency of a 1,000-key read
  across 100 servers from 1,800 ms to 74 ms, while sending only 2% more requests.)
- **Tied requests**: send to two replicas at once, each told about the other; whichever starts
  processing first tells the other to cancel. Cuts queueing delay without doubling work.
- **Timeouts with partial results** ("good enough" answers): a search engine can return results from
  98 of 100 shards rather than waiting for the last two. Acceptable only when completeness isn't
  required.
- **Reduce fan-out**: fetch fewer, larger pieces; co-locate data that is read together; cache the
  aggregated result.
- **Reduce variability at the source**: schedule background work (compaction, GC) carefully, limit
  queue lengths, keep utilization moderate, prioritize interactive requests over batch work.
- **Micro-partitioning and selective replication**: split data into many small partitions so load can
  be rebalanced finely, and keep extra replicas of hot items.
- **Latency-induced probation**: temporarily take a consistently slow server out of rotation.

These come with a cost: hedging only works for **idempotent** reads (or with care, topic 38), and it
adds load, so it must be limited (hedging everything during overload makes overload worse).

### Measuring it

Because of amplification, you need high percentiles (p99, p99.9) **per backend** and the end-to-end
percentiles at the **user** level. Topic 46 covers how to collect them with histograms and tracing so
you can see which fan-out branch produced a slow request.

---

## 6. How it connects

- **Topic 01**: this topic explains the mechanics behind latency vs throughput (the queueing curve),
  Little's Law, and why tail latency matters.
- **Networking (topics 03–06)**: round trips across distance are the dominant cost; TCP and TLS
  handshakes, connection reuse, and HTTP/2–3 are largely about doing fewer round trips.
- **Caching and CDNs (topics 11–14)**: caches exist because RAM is orders of magnitude faster than
  disk and network, and CDNs because the speed of light is fixed.
- **Storage engines (topics 17, 19, 20)**: WAL, B-trees, LSM-trees and column stores are shaped by
  sequential vs random I/O.
- **Replication and partitioning (topics 23–25)**: scatter-gather queries across partitions suffer
  fan-out amplification; synchronous cross-region replication pays the cross-region RTT.
- **Consensus and CAP/PACELC (topics 29–30)**: the latency side of PACELC is exactly the cost of extra
  round trips, often across zones or regions.
- **Messaging (topics 36–37)**: queues are literal queues; Little's Law turns consumer lag into delay;
  Kafka's design is built around sequential I/O.
- **Reliability (topics 43–44)**: timeouts, retries with backoff, load shedding, backpressure,
  bulkheads and circuit breakers are all responses to the queueing collapse described here.
- **Observability (topic 46)**: measuring percentiles correctly and tracing fan-out.

## Common misconceptions

- **"The server is at 80% CPU, so it has 20% capacity left."** In throughput terms maybe, but at 80%
  utilization average wait is already ~5× the service time and small bursts push it much higher.
- **"Queues prevent overload."** A queue absorbs short bursts. Under sustained overload an unbounded
  queue just converts excess load into unbounded latency and memory use. Bound it and shed load.
- **"SSDs made random vs sequential irrelevant."** Random reads got far cheaper, but small random
  writes still cause write amplification and GC pauses, and sequential access still wins in RAM, on
  SSD, and on HDD.
- **"Within a datacenter the network is basically free."** A round trip costs ~0.1–0.5 ms, equivalent
  to thousands of RAM accesses. Chatty designs (N+1 queries) pay it many times.
- **"A bigger connection pool means more throughput."** By Little's Law, throughput = concurrency /
  latency. Extra connections only help if the bottleneck is concurrency; if the database is the
  bottleneck, more concurrent queries make each one slower.
- **"Little's Law only applies to M/M/1 queues."** It holds for any stable system regardless of
  arrival or service distributions.
- **"Parallelizing calls fixes latency."** It turns the sum into a max, which helps the median, but the
  max over many calls is governed by the tail.
- **"Retries improve reliability."** Retries help with transient faults, but during overload they
  multiply load and can lock the system into a metastable failure. Retries need budgets and backoff.

## Further reading (supplements)

- Jeff Dean, *Latency Numbers Every Programmer Should Know* (originally from his talks; widely
  reproduced, with interactive updated versions online)
- Dean & Barroso, *The Tail at Scale* (Communications of the ACM, 2013)
- Martin Kleppmann, DDIA, Chapter 1 (percentiles, tail amplification) and Chapter 3 (storage engines,
  sequential writes)
- Brendan Gregg, *Systems Performance* (2nd ed.), Chapter 2 *Methodologies* (latency scales, queueing,
  utilization vs saturation)
- Mor Harchol-Balter, *Performance Modeling and Design of Computer Systems: Queueing Theory in Action*
- Bronson et al., *Metastable Failures in Distributed Systems* (HotOS 2021)
- John D. C. Little, *A Proof for the Queuing Formula: L = λW* (Operations Research, 1961)

---

## Questions

1. Put these in order from fastest to slowest and give rough magnitudes: an SSD random read, a
   cross-ocean round trip, a RAM access, a same-datacenter round trip, an HDD seek, an L1 cache hit.

2. A team moves their database from the same datacenter as the app to another region 2,000 km away
   "because it's cheaper." Each page load makes 15 sequential queries. Before any other change, what
   happens to page latency, and why can't faster hardware fix it?

3. Why does an HDD deliver ~200 MB/s on sequential reads but less than 1 MB/s on random 4 KB reads?
   What's the equivalent reason SSDs still prefer sequential *writes*?

4. A write-ahead log and an LSM-tree both rely on the same trick. What is it, and what cost do they
   pay later in exchange?

5. A single-threaded service has an average service time of 5 ms. It currently receives 150 req/s.
   What's its utilization, and roughly what average response time would you expect (M/M/1)? What
   happens if traffic grows to 190 req/s?

6. Two services both run at 70% utilization with the same average service time. One serves uniform
   requests; the other serves a mix where 1 in 20 requests is 50× more expensive. Which has longer
   queues, and why? What could you change?

7. Why does one shared queue feeding four workers give lower latency than four separate queues, one per
   worker, at the same total load?

8. Describe how a brief traffic spike can leave a service down for an hour after the spike ends. What
   is this kind of failure called, and what breaks the loop?

9. A service receives 800 req/s; each request holds a database connection for 25 ms. How many
   connections are in use on average? If the pool is capped at 30 and query time rises to 100 ms,
   what is the maximum throughput, and what happens to the remaining requests?

10. Why is a dependency that has become very slow often worse for its callers than one that is
    completely down?

11. Your monitoring shows a queue depth of 12,000. Is that bad? What other number do you need, and
    what does Little's Law let you compute?

12. A closed-loop load test with 100 threads shows the system handling slowdowns gracefully, but
    production collapses under the same slowdown. Explain the difference.

13. A search request fans out to 200 shards and waits for all of them. Each shard has p99 = 50 ms and
    p50 = 5 ms. Roughly what fraction of user requests will take at least 50 ms? Name two ways to
    reduce that.

14. Why does hedging a request after the p95 latency add only ~5% extra load, and why must hedging be
    restricted during overload?

15. List four causes of tail latency on a single server that have nothing to do with how efficient the
    request-handling code is.

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** L1 hit (~1 ns) → RAM (~100 ns) → SSD random read (~20–100 µs) → same-datacenter round trip
(~0.1–0.5 ms) → HDD seek (~5–10 ms) → cross-ocean round trip (~70–150 ms). Each step is roughly one to
three orders of magnitude.

**2.** 2,000 km is ~20 ms of round-trip time from the speed of light in fiber alone, realistically
~25–40 ms. Fifteen *sequential* queries pay it fifteen times: roughly 0.4–0.6 s added to every page,
versus a few ms before. Faster hardware can't help because the cost is propagation delay, bounded by
physics. The fixes are to move the data back near the app (or replicate it there), or to cut the
number of round trips (batch the queries, run them in parallel, cache results).

**3.** On an HDD, each random read pays a mechanical seek plus rotational delay (~8 ms total) before
transferring only 4 KB, so you get ~100–200 reads/s ≈ 0.5 MB/s. A sequential read pays that delay once
and then streams continuously. SSDs can't overwrite flash pages in place and erase in large blocks, so
small random writes leave invalid pages scattered across many blocks; garbage collection must copy the
valid data out before erasing, causing write amplification, wear, and latency spikes. Large sequential
writes invalidate whole blocks together, which is cheap to reclaim.

**4.** Both convert random writes into sequential appends: the WAL appends each change to a log
instead of updating pages in place before acknowledging; the LSM-tree buffers writes in memory and
flushes them as large sorted files. The cost comes later: the WAL needs checkpointing (applying changes
to data pages and truncating the log), and the LSM-tree needs compaction (merging files, discarding
overwritten values), both consuming background I/O and sometimes causing latency spikes. LSM reads may
also have to check several files.

**5.** Capacity = 1/5 ms = 200 req/s; utilization ρ = 150/200 = 75%. W = S / (1 − ρ) = 5 / 0.25 = 20 ms.
At 190 req/s, ρ = 95%, W = 5 / 0.05 = 100 ms: a 27% traffic increase makes latency 5× worse. Any further
burst or slowdown pushes it toward 100% and the queue grows without bound.

**6.** The mixed one. Kingman's formula multiplies wait by a variability term that grows with the
squared coefficient of variation of service time. Expensive requests occupy the server for a long time
and everything behind them waits (head-of-line blocking), so average wait is much longer even at the
same utilization and same mean service time. Fixes: route expensive requests to a separate pool
(isolate variability), make them cheaper or asynchronous, or run at lower utilization.

**7.** With separate queues, a request can wait behind a slow job at its worker while another worker
sits idle. With a shared queue, a request waits only if *all* workers are busy, which is much less
likely. Idle capacity is never wasted while work is waiting. At 80% load with two servers, the shared
queue roughly halves the average wait compared to separate queues; the advantage grows with more
servers.

**8.** The spike pushes utilization near or above 100%, so latency rises past client timeouts. Clients
give up and retry, but the server still processes the abandoned requests. The extra retries keep the
arrival rate above capacity even after the original spike ends, so the queue never drains and nearly
every request times out: the server is fully busy doing useless work. This is a **metastable failure**
(also called congestive collapse or a retry storm). Breaking the loop requires reducing load below
capacity: retry budgets and exponential backoff with jitter, load shedding, bounded queues, dropping
requests whose deadline has passed, or temporarily blocking traffic and letting it back in gradually.

**9.** L = λ × W = 800 × 0.025 = 20 connections on average. With a cap of 30 and W = 0.1 s, max
throughput = L / W = 30 / 0.1 = 300 req/s. The other 500 req/s can't get a connection: they queue in
the service (holding threads and memory, raising latency for everything), time out, or get rejected.
If they queue without limit, the service itself becomes unresponsive to its callers, propagating the
failure upstream.

**10.** A dead dependency fails fast: connection refused, error returned immediately, the thread and
connection are freed. A slow one holds threads, connections and memory for the full duration of each
call, and by Little's Law the concurrency it consumes is throughput × latency, so it can exhaust the
caller's pools and stall unrelated requests too. That's why timeouts, bulkheads and circuit breakers
(which turn "slow" into "fast failure") matter.

**11.** It depends on the drain rate. Little's Law: expected wait W = L / λ. At 12,000 items/s
throughput, 12,000 queued is 1 s of delay (probably fine). At 10 items/s, it's 20 minutes. Alert on
the implied delay (or the age of the oldest item), not raw depth. Also check the trend: a growing queue
means arrivals exceed throughput.

**12.** In a closed system, each of the 100 threads waits for a response before sending the next, so
in-flight requests are capped at 100; when latency rises, the test automatically sends fewer requests
(λ = L / W falls). The load drops exactly when the system struggles, hiding the overload, and slow
periods are under-sampled (coordinated omission). Production is an open system: users keep arriving
whatever the latency, so queues grow and timeouts and retries kick in. Use open-loop (constant arrival
rate) load generators to model production.

**13.** P(at least one of 200 shards is at or above its p99) = 1 − 0.99^200 ≈ 1 − 0.134 ≈ 87%. So most
user requests take at least 50 ms even though the typical shard answers in 5 ms. Ways to reduce it:
hedged or tied requests to replicas of slow shards; return partial results after a deadline; reduce
fan-out (fewer, bigger shards, or pre-aggregated results); attack the causes of shard tail latency
(GC tuning, background-work scheduling, lower utilization).

**14.** By definition only ~5% of requests are still outstanding at the p95 latency, so only those get
a duplicate: ~5% extra requests (and less extra work if the loser is cancelled). Yet the user's latency
becomes roughly the minimum of two independent tries for exactly the slow cases, which cuts the tail
sharply. During overload, though, latency rises across the board, so far more requests cross the hedge
threshold and the duplicates add load to a system that is already saturated, making things worse.
Hedging should be capped (e.g. a budget of a few percent) and disabled when the backend is overloaded.

**15.** Any four of: garbage-collection pauses; queueing behind a local burst; background work such as
compaction, checkpointing, backups or log rotation; SSD garbage collection; noisy neighbours on a
shared host; lock contention; cache misses forcing disk reads; packet loss and TCP retransmission
timeouts; CPU throttling or power-state changes; the process being descheduled by the OS.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** L1 cache vs RAM latency?
**A:** ~1 ns vs ~100 ns: RAM is ~100× slower.

**Q:** Random read latency on an NVMe SSD?
**A:** ~20–100 µs, roughly 1,000× slower than RAM.

**Q:** Round trip within a datacenter vs between availability zones?
**A:** ~0.1–0.5 ms within a datacenter; ~1–2 ms between zones.

**Q:** HDD seek + rotation time?
**A:** ~5–10 ms, giving only ~100–200 random reads/s.

**Q:** Cross-ocean round trip?
**A:** ~70–150 ms; far side of the world up to ~300 ms.

**Q:** Speed-of-light rule of thumb for fiber?
**A:** ~200,000 km/s: about 1 ms of round trip per 100 km of distance.

**Q:** Why can't hardware fix cross-region latency?
**A:** It's propagation delay bounded by the speed of light; only moving data closer or doing fewer round trips helps.

**Q:** Why is sequential I/O faster on an HDD?
**A:** Random access pays a mechanical seek and rotation per read; sequential pays it once, then streams.

**Q:** Why do SSDs still prefer sequential writes?
**A:** Flash erases in large blocks and can't overwrite pages in place; random small writes cause garbage collection, write amplification and wear.

**Q:** Four designs that turn random writes into sequential ones?
**A:** Write-ahead logs, LSM-trees, append-only logs like Kafka, and group commit/batching.

**Q:** Utilization ρ?
**A:** Arrival rate × service time (λ × S): the fraction of time a server is busy.

**Q:** M/M/1 time in system?
**A:** W = S / (1 − ρ): 2× service time at 50%, 10× at 90%, 100× at 99%.

**Q:** Why is there waiting below 100% utilization?
**A:** Arrivals and service times are random; idle time is lost and can't absorb later bursts.

**Q:** Kingman's formula, in words?
**A:** Wait ≈ utilization term ρ/(1−ρ) × variability term (ca² + cs²)/2 × service time.

**Q:** Three levers to reduce queueing delay?
**A:** Lower utilization, reduce variability, reduce service time.

**Q:** Why does one shared queue beat per-worker queues?
**A:** A request waits only when all workers are busy; no worker sits idle while work waits elsewhere.

**Q:** What is a metastable failure?
**A:** A system stuck in a bad state (e.g. retry storm) sustained by its own feedback after the trigger is gone.

**Q:** Goodput?
**A:** Throughput of useful work: requests completed in time for someone to use the result.

**Q:** Why bound queues?
**A:** Unbounded queues turn sustained overload into unbounded latency and memory use; better to reject early.

**Q:** Little's Law and its key property?
**A:** L = λ × W; holds for any stable system regardless of arrival or service distributions.

**Q:** Max throughput with a pool of C connections and latency W?
**A:** C / W.

**Q:** Turn a queue depth into a delay?
**A:** W = L / λ: depth divided by drain rate.

**Q:** Why is a slow dependency worse than a dead one?
**A:** It holds threads and connections for the whole call (concurrency = throughput × latency), exhausting the caller's pools.

**Q:** Open vs closed system?
**A:** Open: arrivals continue regardless of latency. Closed: fixed clients wait for each response, so throughput falls as latency rises.

**Q:** Fan-out tail amplification formula?
**A:** P(slow) = 1 − (1 − p)^N; at p = 1% and N = 100, about 63%.

**Q:** Parallel calls vs sequential calls: how do latencies combine?
**A:** Parallel: the max (governed by the tail). Sequential: the sum.

**Q:** Hedged request?
**A:** If no reply by ~p95, send a duplicate to another replica and take the first answer; ~5% extra load, big tail cut.

**Q:** Tied request?
**A:** Send to two replicas at once; whichever starts first cancels the other.

**Q:** Three non-code causes of tail latency?
**A:** Examples: GC pauses, background compaction, noisy neighbours, queueing bursts, TCP retransmits.
