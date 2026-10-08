# Day 1: Core Vocabulary of System Design

Every later topic in this plan (caching, replication, consensus, queues) is a tool for moving one of a
small number of dials: **how fast**, **how much**, **how often it works**, **whether data survives**, and
**how it behaves as load grows**. Today is about naming those dials precisely, because most bad design
arguments come from two people using the same word for different things.

By the end of today you should be able to:

1. Explain latency vs throughput and why pushing one can hurt the other.
2. Say exactly how availability, reliability, durability and fault tolerance differ.
3. Define scalability in terms of *load parameters* and *performance under load*, not as a vibe.
4. Explain why p99 matters more than the average, and why tail latency gets worse as systems grow.
5. Contrast vertical and horizontal scaling, and explain why statelessness makes horizontal scaling easy.
6. Name the standard building blocks and the one job each does.

---

## 1. Latency vs throughput

### Definitions

- **Latency**: how long one unit of work takes. Strictly, *latency* is the time a request spends
  *waiting* to be handled; *response time* is what the client sees end to end (network + queueing +
  service time). In everyday use people say "latency" for both. Be precise when it matters.
- **Throughput**: how many units of work complete per unit time (requests/s, MB/s, messages/s).
- **Bandwidth** is the *capacity* of a channel (the maximum possible throughput). Throughput is what
  you actually achieve.

A highway analogy: latency is how long one car takes to drive from A to B. Throughput is how many cars
arrive at B per hour. A wider highway raises throughput without making any single car faster.

### They are related, but not opposites

The link between them is **Little's Law**, which holds for any stable system:

```
L = λ × W

L = average number of requests in the system (concurrency, "in flight")
λ = arrival rate (throughput, in steady state)
W = average time each request spends in the system (latency)
```

If a service handles 1,000 req/s and each takes 200 ms, there are on average 200 requests in flight.
If you want more throughput at the same latency, you need more concurrency (more workers, threads,
connections). If concurrency is capped (say, a pool of 50 DB connections), then latency and throughput
are directly traded: throughput can't exceed `50 / W`.

### Why improving one can hurt the other

**1. Batching.** Grouping work (writing 100 rows in one disk flush, sending 50 messages in one network
packet) amortizes fixed costs, so throughput rises. But the first item in a batch waits for the batch
to fill, so its latency rises. Kafka producers, database group commit, and TCP's Nagle algorithm all
make this trade explicitly.

**2. Queueing and utilization.** This is the most important intuition of the day. As a resource gets
busier, the time requests spend *waiting in line* grows, and it grows non-linearly. For a simple
single-server queue (M/M/1 model), average time in system is:

```
W = S / (1 − ρ)        S = service time, ρ = utilization (0..1)

ρ = 50%  → W = 2 × S
ρ = 80%  → W = 5 × S
ρ = 90%  → W = 10 × S
ρ = 99%  → W = 100 × S
```

```
latency
  │                                  │
  │                                 ╱
  │                               ╱
  │                            ╱
  │                       _.-'
  │              __..--''
  │ ____...---'''
  └──────────────────────────────────── utilization
  0%                 70%        90%  100%
```

So driving a server to 99% utilization maximizes throughput per machine, but latency explodes. This is
why well-run systems keep headroom (often targeting 50–70% utilization on latency-sensitive paths) and
why "the box is only at 85% CPU" can still mean users see terrible latency.

**3. Pipelining and parallelism** usually help both, until a shared resource (a lock, a disk, a single
leader) saturates. Then you are back in the queueing curve above.

**Takeaway:** throughput is a *capacity* question ("how much can we push through?"); latency is an
*experience* question ("how long does each request wait?"). You tune for throughput in batch/background
work and for latency on user-facing paths.

---

## 2. Percentiles and tail latency

### Why averages mislead

Latency distributions are not bell curves. They are **right-skewed**: most requests are fast, a few are
very slow (a GC pause, a cache miss, a retransmitted packet, a lock wait, a noisy neighbour).

Take 100 requests: 98 take 10 ms, 2 take 2,000 ms.

- Mean = (98 × 10 + 2 × 2000) / 100 = **49.8 ms**. Describes no actual request.
- Median (p50) = **10 ms**. What the typical request sees.
- p99 = **2,000 ms**. What 1 in 100 requests sees.

The mean is dragged around by outliers yet still hides how bad they are. Two very different systems can
have the same mean.

### Percentiles

- **pN** is the value below which N% of observations fall. p50 = median, p95, p99, p99.9.
- p50 tells you the typical experience. p99/p99.9 tell you about the **tail**.

### Why the tail matters more than it looks

1. **Your heaviest users hit the tail most.** Users with the most data (biggest carts, most followers,
   largest histories) often make the slowest requests. These are frequently your most valuable users.

2. **One user, many requests.** A single page load can trigger dozens of requests. If each has a 1%
   chance of being slow, a user loading many pages a day sees the tail constantly.

3. **Fan-out amplifies the tail (tail-latency amplification).** If one user request fans out to N
   backend calls in parallel and must wait for all of them, the user request is as slow as the
   *slowest* call:

   ```
   P(user request hits at least one p99-slow call) = 1 − 0.99^N

   N = 1    → 1%
   N = 10   → ~10%
   N = 100  → ~63%
   ```

   At N = 100, the backend's p99 has become the user's *median*. This is why large systems invest so
   heavily in tail latency, and why techniques like **hedged requests** (send a duplicate request to
   another replica if the first is slow, take whichever returns first) exist.

### Practical rules

- **You cannot average percentiles.** The average of each server's p99 is not the fleet's p99. To
  combine, merge the underlying distributions (histograms or sketches such as HDR histograms,
  t-digest), then compute the percentile.
- **Measure from the client's side** where possible. A server measuring only its own processing time
  misses queueing that happens before the request is picked up.
- **Coordinated omission**: if a load generator waits for a slow response before sending the next
  request, it sends fewer requests exactly when the system is slow, so it under-counts the slow
  period. Real users don't politely wait for each other.
- SLOs are usually stated on percentiles: "p99 < 300 ms over 30 days", not "average < 100 ms".

---

## 3. Availability, reliability, durability, fault tolerance

These four get used interchangeably. They are different questions.

| Term | Question it answers | Measured as |
|---|---|---|
| **Availability** | Is the system up and responding *right now*? | % of time (or % of requests) served successfully |
| **Reliability** | Does it keep working *correctly* over time? | Probability of correct operation over a period; MTBF |
| **Durability** | Once data is acknowledged as stored, will it still be there later? | Probability of not losing an object per year |
| **Fault tolerance** | Can it keep working *while* parts of it are broken? | Which faults, and how many, it survives |

### Availability

```
Availability = uptime / (uptime + downtime)
             ≈ MTBF / (MTBF + MTTR)

MTBF = mean time between failures
MTTR = mean time to recovery
```

Note the second form: you can raise availability by failing less often **or by recovering faster**.
Cutting MTTR (fast failover, quick rollback, good alerting) is often cheaper than making failures rarer.

The "nines":

| Availability | Downtime / year | Downtime / month (30d) |
|---|---|---|
| 99% (two nines) | ~3.65 days | ~7.2 hours |
| 99.9% (three nines) | ~8.77 hours | ~43 minutes |
| 99.99% (four nines) | ~52.6 minutes | ~4.3 minutes |
| 99.999% (five nines) | ~5.3 minutes | ~26 seconds |

Each extra nine is 10× less downtime and usually much more than 10× the cost. At four or five nines,
there's no time for a human to even notice a problem; recovery must be automatic.

Many systems measure availability per **request** rather than per minute: `successful requests / total
requests`. This handles partial outages (a system that fails 5% of requests isn't "up" or "down").

**Composition matters:**

- **In series** (A calls B, both must work): availabilities multiply.
  `0.999 × 0.999 = 0.998`. Every hard dependency you add *lowers* your availability ceiling. A service
  with 10 hard dependencies each at 99.9% can't be better than ~99.0%.
- **In parallel** (redundant copies, any one is enough): unavailabilities multiply.
  Two independent 99.9% replicas: `1 − (0.001 × 0.001) = 99.9999%`.

The word *independent* is doing a lot of work. Two replicas on the same rack, same power supply, or
running the same buggy software version fail together. **Correlated failures** are why the parallel
math is optimistic in practice.

### Reliability

Reliability is about **correctness over time**: the system does what it's supposed to, tolerates user
mistakes, performs well enough, and prevents misuse. A system can be *available but unreliable*: it
answers every request instantly, but some answers are wrong (stale data, corrupted results, double
charges). Availability asks "did you answer?"; reliability asks "was the answer right, consistently?"

### Durability

Durability is about **data, not service**. Once a write is acknowledged, it must survive crashes,
power loss, and disk failures.

- A system can be **unavailable but durable**: the database is down for an hour, but when it returns,
  every committed write is there.
- A system can be **available but not durable**: an in-memory cache that answers fast and loses
  everything on restart.

Durability comes from writing to non-volatile storage before acknowledging (fsync, write-ahead logs),
replicating to multiple machines/zones, backups, and checksums to detect silent corruption. Object
stores often advertise "eleven nines" (99.999999999%) of *durability*, which is a statement about the
chance of losing an object, not about uptime.

### Fault tolerance

First, separate two words:

- A **fault** is one component deviating from its spec (a disk dies, a process crashes, a network link
  drops packets).
- A **failure** is the *system as a whole* failing to provide its service to users.

Fault tolerance (resilience) is designing so that **faults don't become failures**. You can't prevent
all faults, so you decide which ones to tolerate: one machine? one rack? a whole availability zone? a
region? Each level costs more.

Fault categories worth knowing:

- **Hardware faults**: disks, RAM, power, network cards. Mostly random and independent, handled by
  redundancy.
- **Software faults**: bugs triggered by an unusual input, a leap second, a runaway process, a
  dependency slowing down. These are often **correlated** (every node runs the same code), which makes
  them more dangerous than hardware faults.
- **Human faults**: misconfiguration is a leading cause of outages. Mitigations: good defaults, staged
  rollouts, quick rollback, sandboxes.

Some teams deliberately inject faults (chaos engineering, e.g. randomly killing instances) to make sure
the fault-tolerance machinery actually works rather than just existing on a diagram.

---

## 4. Scalability

"Does it scale?" is meaningless on its own. Scalability is the ability to **cope with increased load**,
and you must say *which load* and *what coping looks like*.

### Load parameters

Describe load with numbers that matter for *your* system's architecture:

- requests per second
- read/write ratio
- number of concurrent connections
- data volume (total stored, growth per day)
- size of individual items
- **fan-out** (how much work one request causes)

A classic example is a social feed. Posting is rare (thousands/s); reading timelines is frequent
(hundreds of thousands/s). Two designs:

1. **On read**: when a user opens their feed, query posts from everyone they follow and merge.
   Writes are cheap, reads are expensive.
2. **On write**: when someone posts, insert the post into each follower's precomputed feed.
   Reads are cheap, writes are expensive, and it gets very expensive for accounts with millions of
   followers.

Which design "scales" depends on the load parameter that dominates: here, *the distribution of
followers per user*. Real systems often do a hybrid. The point: you can only reason about scaling once
you've named the load parameters.

### Performance under load

Two ways to ask the question:

1. Increase a load parameter and **keep resources fixed**: how does performance degrade?
2. Increase a load parameter: **how many more resources** do you need to keep performance the same?

"Performance" means throughput for batch systems and response-time percentiles for online systems.

### Scalability vs performance

- A **performance** problem: the system is slow for a single user.
- A **scalability** problem: the system is fast for one user but slow under heavy load.

Ideal (linear) scaling: double the machines, double the throughput. Real systems fall short because of:

- **Contention**: work that must be serialized (a lock, a single leader, a hot row). Amdahl's law: if
  10% of the work is serial, you can never get more than 10× speedup no matter how many machines.
- **Coordination cost**: nodes talking to each other to agree. This can grow faster than the number of
  nodes (pairwise communication is O(n²)), which can make adding nodes *reduce* throughput.

There's no single architecture that scales for everything. A system designed for 100,000 small
requests/s looks very different from one handling 3 large requests/minute, even if both move the same
bytes.

---

## 5. Vertical vs horizontal scaling

| | Vertical (scale up) | Horizontal (scale out) |
|---|---|---|
| What | Bigger machine: more CPU, RAM, faster disk | More machines sharing the load |
| Code changes | Usually none | Often significant: the work must be divisible |
| Ceiling | Hard limit: the biggest machine you can buy | Very high in principle |
| Cost curve | Super-linear: high-end hardware costs disproportionately more | Roughly linear, on commodity hardware |
| Failure | Single machine = single point of failure | Redundancy comes naturally |
| Complexity | Low | High: distribution, coordination, partial failure |

Vertical scaling is underrated. Modern single machines are enormous, and a single well-tuned database
server can handle far more than people expect. It avoids every hard problem in Week 3. The usual path
is: scale up as long as it's comfortable, scale out when you hit the ceiling, need redundancy, or the
cost curve turns against you.

Horizontal scaling is easy for some components and hard for others. That difference is almost entirely
about **state**.

### Stateless vs stateful

- A **stateless** component keeps no data between requests that another instance would need. Any
  instance can handle any request. Example: an app server that reads the user's session from a shared
  store rather than its own memory.
- A **stateful** component holds data that matters across requests: databases, caches, a server
  holding open WebSocket connections, an app server keeping sessions in local memory.

Why it matters:

```
Stateless tier: just add instances

        ┌──▶ app-1 ─┐
LB ─────┼──▶ app-2 ─┼──▶ shared state (DB / cache)
        └──▶ app-3 ─┘

Any request can go to any instance. Lose app-2? Requests go elsewhere, nothing is lost.
```

For a stateful component, adding a node raises immediate questions: *which data lives on the new node?
How does it get there? What happens to requests while data moves? What if the node holding the only
copy dies?* Those questions are replication (Day 13) and partitioning (Day 14), and they're the hard
part of the whole subject.

The standard pattern is to **push state down** to a few specialized stateful systems (databases,
caches, queues, object stores) and keep everything above them stateless. The application tier becomes
trivially scalable and the hard problems are concentrated in components built specifically to solve
them.

---

## 6. The building blocks

Here's a typical request path. Not every system has every piece; the point is to know what job each
one does.

```
                      ┌──────────┐
                      │   DNS    │  name → IP
                      └────▲─────┘
                           │ 1. resolve
┌────────┐ 2. static ┌─────┴────┐
│ Client │──────────▶│   CDN    │──── miss ───▶ object storage / origin
└───┬────┘  assets   └──────────┘
    │ 3. API calls
    ▼
┌───────────────┐
│ Load balancer │  spreads traffic, removes dead servers
└───────┬───────┘
   ┌────┼────┐
   ▼    ▼    ▼
┌─────┐┌─────┐┌─────┐
│ App ││ App ││ App │   stateless business logic
└──┬──┘└──┬──┘└──┬──┘
   │      │      │
   ├──────┼──────┼──────────────┬────────────────┐
   ▼      ▼      ▼              ▼                ▼
┌───────┐ ┌──────────┐   ┌────────────┐   ┌──────────────┐
│ Cache │ │ Database │   │   Queue    │   │Object storage│
└───────┘ └──────────┘   └─────┬──────┘   └──────────────┘
                               ▼
                         ┌───────────┐
                         │  Workers  │  async/background jobs
                         └───────────┘
```

| Block | Its job | Stateful? | Deep dive |
|---|---|---|---|
| **Client** | Browser, mobile app, or another service making requests. Can cache, retry, and hold local state. | Partly | Day 3 |
| **DNS** | Translates a name to an IP address. Also a coarse traffic-steering tool (geographic routing, failover). | Yes (cached records) | Day 2 |
| **CDN** | Network of edge servers close to users that cache content, so requests don't travel to your origin. Cuts latency and origin load. | Cache | Day 7 |
| **Load balancer** | Single entry point that distributes requests across healthy servers. Enables horizontal scaling and hides server failures. | Mostly no | Day 4 |
| **App server** | Runs business logic. Should be stateless so it can be scaled and replaced freely. | No (ideally) | — |
| **Cache** | Keeps hot data in memory to avoid slow or expensive reads. Trades freshness for speed. | Yes (but rebuildable) | Days 5–6 |
| **Database** | System of record. Stores data durably and answers queries, often with transactional guarantees. | Yes | Week 2 |
| **Queue / log** | Accepts work now, processes it later. Decouples producers from consumers, absorbs spikes, enables retries. | Yes | Days 22–23 |
| **Workers** | Consume from the queue and do slow or background work (emails, image resizing, reports). | No (ideally) | Days 22–25 |
| **Object storage** | Stores large immutable blobs (images, videos, backups) cheaply and durably, addressed by key. Not a filesystem, not a database. | Yes | Day 29 |

A useful lens: each block exists to fix a specific weakness of the plain "client → server → database"
setup.

- One server can't handle the load → **load balancer** + more app servers.
- The database is the bottleneck for reads → **cache**, read replicas.
- Users are far away → **CDN**, multi-region.
- Some work is slow and the user shouldn't wait → **queue** + workers.
- Large files bloat the database → **object storage**.
- The database is too big for one machine → partitioning (sharding).

And each block **brings its own new problems**: caches go stale, queues deliver duplicates, replicas
lag, load balancers become single points of failure. Much of the rest of this month is about those
second-order problems.

---

## 7. How it connects

- **Latency and throughput** are what users and capacity plans feel. **Percentiles** are how you
  measure them honestly.
- **Availability, reliability and durability** are what you promise. **Fault tolerance** is how you
  keep the promise.
- **Scalability** is whether you keep the promises as load grows. It depends on load parameters and on
  how much **state** your components hold.
- **Building blocks** are the parts you combine. Each trades something (freshness, simplicity,
  consistency, cost) for something else.

Almost every design choice you'll see this month can be described as: *"this moves dial X at the cost
of dial Y."*

## Common misconceptions

- **"Highly available means my data is safe."** No. Availability is about answering; durability is
  about not losing data. A cache is available and not durable.
- **"Average latency is 50 ms, so we're fine."** The average hides the tail, and the tail is what heavy
  users and fan-out requests hit.
- **"More servers = proportionally more throughput."** Only for the stateless, contention-free parts.
  Shared state and coordination cap the gain.
- **"Two replicas at 99.9% gives 99.9999%."** Only if their failures are independent. Shared racks,
  shared config, and shared bugs make failures correlated.
- **"We'll scale horizontally from day one."** Often premature. Vertical scaling buys a lot of headroom
  with none of the distributed-systems complexity.

---

## Further reading (supplements)

- DDIA, Chapter 1: *Reliable, Scalable, and Maintainable Applications*
- Dean & Barroso, *The Tail at Scale* (Communications of the ACM, 2013): tail latency and hedged requests
- Gil Tene, *How NOT to Measure Latency* (talk): percentiles and coordinated omission
- Google SRE Book, Chapter 3 *Embracing Risk* and Chapter 4 *Service Level Objectives*
