# 01 — Core Vocabulary

> Part 1 — Foundations · Prerequisites: none

Every later topic in this plan (caching, replication, consensus, queues) is a tool for moving one of a
small number of dials: **how fast**, **how much**, **how often it works**, **whether data survives**, and
**how it behaves as load grows**. This topic names those dials precisely, because most bad design
arguments come from two people using the same word for different things. It also introduces the
standard building blocks, so that later topics can refer to them by name.

By the end of this topic you should be able to:

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
- **Service time**: the time the server spends actually *working* on the request, excluding waiting.
- **Throughput**: how many units of work complete per unit time (requests/s, MB/s, messages/s).
- **Bandwidth**: the *capacity* of a channel (the maximum possible throughput). Throughput is what
  you actually achieve.

```
client sends                                              client receives
    │                                                           │
    │── network ──│── queueing (waiting) ──│── service ──│─ network ─│
    │                                                           │
    └──────────────────── response time ────────────────────────┘
```

A highway analogy: latency is how long one car takes to drive from A to B. Throughput is how many cars
arrive at B per hour. A wider highway raises throughput without making any single car faster. A faster
speed limit lowers latency, and (with the same spacing between cars) raises throughput too.

### They are related, but not opposites

The link between them is **Little's Law**, which holds for any stable system (topic 02 goes deeper):

```
L = λ × W

L = average number of requests in the system (concurrency, "in flight")
λ = arrival rate (equals throughput, in steady state)
W = average time each request spends in the system (latency)
```

If a service handles 1,000 req/s and each takes 200 ms, there are on average 200 requests in flight.
If you want more throughput at the same latency, you need more concurrency (more workers, threads,
connections). If concurrency is capped (say, a pool of 50 DB connections), then latency and throughput
are directly traded: throughput can't exceed `50 / W`. Make each request slower and the maximum
throughput drops with it.

### Why improving one can hurt the other

**1. Batching.** Grouping work (writing 100 rows in one disk flush, sending 50 messages in one network
packet) amortizes fixed costs, so throughput rises. But the first item in a batch waits for the batch
to fill, so its latency rises. Kafka producers (`linger.ms`), database group commit, and TCP's Nagle
algorithm all make this trade explicitly.

```
No batching:   [w1]flush [w2]flush [w3]flush [w4]flush      4 flushes, each write waits ~0
Batching:      [w1 w2 w3 w4]────────flush                  1 flush, w1 waited for w2..w4
```

**2. Queueing and utilization.** This is the most important intuition in Part 1. As a resource gets
busier, the time requests spend *waiting in line* grows, and it grows non-linearly. For a simple
single-server queue (the M/M/1 model, explained in topic 02), average time in system is:

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
leader) saturates. Then you are back on the queueing curve above.

**4. Extra work per request for efficiency.** Compression cuts bytes on the wire (more throughput over
a fixed-bandwidth link) but adds CPU time to each request (more latency when the link isn't the
bottleneck).

**Takeaway:** throughput is a *capacity* question ("how much can we push through?"); latency is an
*experience* question ("how long does each request wait?"). You tune for throughput in batch and
background work and for latency on user-facing paths.

---

## 2. Percentiles and tail latency

### Why averages mislead

Latency distributions are not bell curves. They are **right-skewed**: most requests are fast, a few are
very slow (a GC pause, a cache miss, a retransmitted packet, a lock wait, a noisy neighbour on the
same host).

```
count
  │█
  │█
  │██
  │███
  │████▄
  │██████▄▄▁▁ ▁    ▁          ▁                   ▁
  └──────────────────────────────────────────────────── latency
   p50   ↑mean                p99               p99.9
```

Take 100 requests: 98 take 10 ms, 2 take 2,000 ms.

- Mean = (98 × 10 + 2 × 2000) / 100 = **49.8 ms**. Describes no actual request.
- Median (p50) = **10 ms**. What the typical request sees.
- p99 = **2,000 ms**. What 1 in 100 requests sees.

The mean is dragged around by outliers yet still hides how bad they are. Two very different systems can
have the same mean.

### Percentiles

- **pN** is the value below which N% of observations fall. p50 = median, then p95, p99, p99.9.
- p50 tells you the typical experience. p99 and p99.9 tell you about the **tail**: the slowest 1% or
  0.1% of requests. *Tail latency* means the latency at these high percentiles.

### Why the tail matters more than it looks

1. **Your heaviest users hit the tail most.** Users with the most data (biggest carts, most followers,
   largest order histories) often make the slowest requests. These are frequently your most valuable
   users.

2. **One user, many requests.** A single page load can trigger dozens of requests. If each has a 1%
   chance of being slow, a user browsing for a few minutes sees the tail constantly.

3. **Fan-out amplifies the tail (tail-latency amplification).** If one user request fans out to N
   backend calls in parallel and must wait for all of them, the user request is as slow as the
   *slowest* call:

   ```
   P(user request hits at least one p99-slow call) = 1 − 0.99^N

   N = 1    → 1%
   N = 10   → ~10%
   N = 100  → ~63%
   ```

   At N = 100, the backend's p99 has become the user's *median*. Topic 02 works through this in more
   detail, along with the techniques (hedged requests, tied requests) used to fight it.

### Practical rules

- **You cannot average percentiles.** The average of each server's p99 is not the fleet's p99. To
  combine, merge the underlying distributions (histograms or sketches such as HDR histograms,
  t-digest), then compute the percentile.
- **Measure from the client's side** where possible. A server measuring only its own processing time
  misses queueing that happens before the request is picked up.
- **Coordinated omission**: if a load generator waits for a slow response before sending the next
  request, it sends fewer requests exactly when the system is slow, so it under-counts the slow
  period. Real users don't politely wait for each other.
- **Service level objectives (SLOs)** are usually stated on percentiles: "p99 < 300 ms over 30 days",
  not "average < 100 ms". Topic 46 covers measuring them.

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
requests`. This handles partial outages (a system that fails 5% of requests isn't simply "up" or
"down").

**Composition matters:**

- **In series** (A calls B, both must work): availabilities multiply.
  `0.999 × 0.999 = 0.998`. Every hard dependency you add *lowers* your availability ceiling. A service
  with 10 hard dependencies each at 99.9% can't be better than ~99.0%.
- **In parallel** (redundant copies, any one is enough): unavailabilities multiply.
  Two independent 99.9% replicas: `1 − (0.001 × 0.001) = 99.9999%`.

```
Series:    client ──▶ [A 99.9%] ──▶ [B 99.9%]         = 99.8%

Parallel:            ┌─▶ [A 99.9%] ─┐
           client ───┤              ├──▶ either works = 99.9999% (if independent)
                     └─▶ [B 99.9%] ─┘
```

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

Durability comes from writing to non-volatile storage before acknowledging (fsync, write-ahead logs,
topic 17), replicating to multiple machines and zones (topics 23–24), backups, and checksums to detect
silent corruption. Object stores often advertise "eleven nines" (99.999999999%) of *durability*, which
is a statement about the chance of losing an object, not about uptime (topic 53).

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
  rollouts, quick rollback, sandboxes (topic 47).

Some teams deliberately inject faults (chaos engineering, e.g. randomly killing instances) to make sure
the fault-tolerance machinery actually works rather than just existing on a diagram. Topic 26 treats
faults and partial failure in depth.

---

## 4. Scalability

"Does it scale?" is meaningless on its own. Scalability is the ability to **cope with increased load**,
and you must say *which load* and *what coping looks like*.

### Load parameters

Describe load with numbers that matter for your system's architecture:

- requests per second
- read/write ratio
- number of concurrent connections
- data volume (total stored, growth per day)
- size of individual items
- **fan-out** (how much work one request causes)
- skew (are a few keys or users much hotter than the rest?)

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
  nodes (pairwise communication is O(n²)), which can make adding nodes *reduce* throughput. (This
  combination of contention and coordination is sometimes modelled as the Universal Scalability Law.)

```
throughput
  │                 ideal (linear)
  │               ╱
  │             ╱       contention only (flattens)
  │           ╱    _.------------------
  │         ╱ _.-'
  │       ╱.-'   _.--.._  contention + coordination (peaks, then falls)
  │     ╱.'  _.-'       ''--.._
  │   ╱.' .-'                  ''--
  │ ╱.'.-'
  └──────────────────────────────────── nodes
```

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

Vertical scaling is underrated. Modern single machines are enormous (hundreds of cores, terabytes of
RAM, millions of SSD IOPS), and a single well-tuned Postgres server can handle far more than people
expect. It avoids every hard problem in Parts 5 and 6. The usual path is: scale up as long as it's
comfortable, scale out when you hit the ceiling, need redundancy, or the cost curve turns against you.

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

```
State stuck on an instance: what goes wrong

  request 1 (login) ──▶ LB ──▶ app-1   [session for alice stored in app-1's memory]
  request 2 (cart)  ──▶ LB ──▶ app-2   "who is alice?" → user appears logged out
```

For a stateful component, adding a node raises immediate questions: *which data lives on the new node?
How does it get there? What happens to requests while data moves? What if the node holding the only
copy dies?* Those questions are replication (topics 23–24) and partitioning (topic 25), and they're the
hard part of the whole subject.

The standard pattern is to **push state down** to a few specialized stateful systems (databases,
caches, queues, object stores) and keep everything above them stateless. The application tier becomes
trivially scalable and the hard problems are concentrated in components built specifically to solve
them. Where state must stay near the app (sessions), the options are a shared store, a signed token
held by the client, or sticky sessions at the load balancer (topic 09), which work but fight against
statelessness.

---

## 6. The building blocks

Here's a typical request path for a generic shopping site. Not every system has every piece; the point
is to know what job each one does.

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
| **Client** | Browser, mobile app, or another service making requests. Can cache, retry, and hold local state. | Partly | Topics 03–08 |
| **DNS** | Translates a name to an IP address. Also a coarse traffic-steering tool (geographic routing, failover). | Yes (cached records) | Topic 04 |
| **CDN** | Network of edge servers close to users that cache content, so requests don't travel to your origin. Cuts latency and origin load. | Cache | Topic 14 |
| **Load balancer** | Single entry point that distributes requests across healthy servers. Enables horizontal scaling and hides server failures. | Mostly no | Topic 09 |
| **App server** | Runs business logic. Should be stateless so it can be scaled and replaced freely. | No (ideally) | — |
| **Cache** | Keeps hot data in memory (e.g. Redis) to avoid slow or expensive reads. Trades freshness for speed. | Yes (but rebuildable) | Topics 11–12 |
| **Database** | System of record (e.g. Postgres, Cassandra, DynamoDB). Stores data durably and answers queries, often with transactional guarantees. | Yes | Topics 15–25 |
| **Queue / log** | Accepts work now, processes it later (e.g. RabbitMQ, Kafka). Decouples producers from consumers, absorbs spikes, enables retries. | Yes | Topics 36–38 |
| **Workers** | Consume from the queue and do slow or background work (emails, image resizing, reports). | No (ideally) | Topics 36–41 |
| **Object storage** | Stores large immutable blobs (images, videos, backups) cheaply and durably, addressed by key (e.g. S3). Not a filesystem, not a database. | Yes | Topic 53 |

A useful lens: each block exists to fix a specific weakness of the plain "client → server → database"
setup.

- One server can't handle the load → **load balancer** + more app servers.
- The database is the bottleneck for reads → **cache**, read replicas.
- Users are far away → **CDN**, multi-region.
- Some work is slow and the user shouldn't wait → **queue** + workers.
- Large files bloat the database → **object storage**.
- The database is too big for one machine → partitioning (sharding).

And each block **brings its own new problems**: caches go stale, queues deliver duplicates, replicas
lag, load balancers become single points of failure. Much of the rest of this plan is about those
second-order problems.

---

## 7. How it connects

- **Latency and throughput** are what users and capacity plans feel. **Percentiles** are how you
  measure them honestly. Topic 02 explains *where* the time goes (hardware latencies, queueing) and
  why the tail grows under fan-out.
- **Availability, reliability and durability** are what you promise. **Fault tolerance** is how you
  keep the promise: replication (topics 23–24), consensus (topic 30), and the resilience patterns of
  Part 8 (topics 43–48).
- **Scalability** is whether you keep the promises as load grows. It depends on load parameters and on
  how much **state** your components hold. Stateless tiers scale behind a load balancer (topic 09);
  stateful ones need partitioning (topic 25) and replication.
- **Building blocks** are the parts you combine. Each trades something (freshness, simplicity,
  consistency, cost) for something else.

Almost every design choice you'll see in this plan can be described as: *"this moves dial X at the
cost of dial Y."*

## Common misconceptions

- **"Highly available means my data is safe."** No. Availability is about answering; durability is
  about not losing data. A cache is available and not durable.
- **"Average latency is 50 ms, so we're fine."** The average hides the tail, and the tail is what heavy
  users and fan-out requests hit.
- **"We doubled throughput, so requests got faster."** Throughput and latency are different measures;
  batching or higher utilization can raise throughput while making each request slower.
- **"85% CPU means 15% headroom."** Queueing delay grows non-linearly; at 85% utilization, requests
  may already spend most of their time waiting.
- **"More servers = proportionally more throughput."** Only for the stateless, contention-free parts.
  Shared state and coordination cap the gain.
- **"Two replicas at 99.9% gives 99.9999%."** Only if their failures are independent. Shared racks,
  shared config, and shared bugs make failures correlated.
- **"We'll scale horizontally from day one."** Often premature. Vertical scaling buys a lot of headroom
  with none of the distributed-systems complexity.

## Further reading (supplements)

- Martin Kleppmann, *Designing Data-Intensive Applications* (DDIA), Chapter 1: *Reliable, Scalable,
  and Maintainable Applications*
- Dean & Barroso, *The Tail at Scale* (Communications of the ACM, 2013): tail latency and hedged requests
- Gil Tene, *How NOT to Measure Latency* (talk): percentiles and coordinated omission
- Google SRE Book, Chapter 3 *Embracing Risk* and Chapter 4 *Service Level Objectives*

---

## Questions

1. A teammate says "we doubled our throughput, so requests must be faster now." Is that necessarily
   true? Describe a change that raises throughput while making individual requests slower.

2. A service handles 2,000 req/s and has an average response time of 50 ms. Roughly how many requests
   are in flight at any moment? What does it imply if the service can only hold 50 concurrent requests?

3. Your database server sits at 92% CPU. Throughput looks fine, but users complain about slowness.
   Explain what's going on using queueing intuition.

4. Why would a database deliberately delay a commit by a millisecond or two (group commit)? Who wins
   and who loses?

5. Two services both have a mean latency of 40 ms. Service A has p99 = 60 ms; service B has p99 =
   900 ms. Which would you rather depend on, and why does the mean fail to tell them apart?

6. You have 10 servers and each reports its own p99. Can you average them to get the fleet p99? Why
   not, and what should you do instead?

7. Why might your server-side latency metrics look great while users experience slow responses?

8. Give an example of a system that is: (a) available but not durable; (b) durable but not (currently)
   available; (c) available but not reliable.

9. Your service has hard dependencies on four other services, each 99.9% available. What's your best
   possible availability? What design changes would raise it?

10. You run two replicas, each 99.9% available. A colleague says "that's six nines." When is that
    claim wrong in practice?

11. Availability ≈ MTBF / (MTBF + MTTR). Your team can either halve how often outages happen or halve
    how long they last. Which gives more availability, and which is usually easier?

12. What's the difference between a fault and a failure? Why are software faults often more dangerous
    than hardware faults, even though hardware faults are more frequent?

13. Someone asks "does our system scale?" What do you need to pin down before you can answer? In the
    social feed example, what load parameter makes "fan-out on write" a problem?

14. You add a fourth node to a three-node cluster and total throughput goes *down*. Give two possible
    explanations.

15. An app server stores logged-in user sessions in its own memory. What breaks when you put three of
    these behind a load balancer? Give two ways to fix it. More generally, why is it easy to add app
    servers but hard to add database servers?

16. When would you choose vertical scaling over horizontal scaling, even though horizontal scaling has
    a higher ceiling?

17. For each problem, name the building block you'd reach for first and the new problem it introduces:
    (a) users on another continent see slow image loads; (b) the database is overwhelmed by repeated
    reads of the same product pages; (c) sending a confirmation email makes checkout take 3 seconds;
    (d) one app server can't keep up with traffic.

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Not necessarily. Throughput (work per second) and latency (time per request) are different
measures. **Batching** raises throughput while slowing individual requests: if you wait to collect 100
writes before one disk flush, you do far more writes per second, but each write waits for the batch to
fill. Similarly, running a server at higher utilization pushes more requests through but makes each
one queue longer.

**2.** Little's Law: `L = λ × W = 2,000 req/s × 0.05 s = 100` requests in flight. If the service can
only hold 50 at once, it can't sustain 2,000 req/s at 50 ms: max throughput is `50 / 0.05 = 1,000
req/s`. The rest will queue (raising latency) or be rejected. You need either more concurrency or lower
per-request time.

**3.** At high utilization, queueing delay explodes non-linearly. In a simple model, time in system ≈
`service time / (1 − utilization)`: at 92% that's ~12.5× the bare service time. Throughput is high
because the server is almost never idle, but requests spend most of their time waiting in line. Any
small burst pushes it closer to 100% and latency spikes further. Fix: add capacity or reduce load to
get headroom.

**4.** An fsync to disk is expensive and has a fixed cost regardless of how much data it covers. By
waiting briefly to collect commits from many transactions and flushing them together, the database
does one fsync instead of many: **throughput wins**. Each individual transaction waits a little
longer: **latency loses** (slightly). Under heavy write load this is a great trade; under light load
the delay is mostly wasted.

**5.** Service A. Its worst 1% of requests are only slightly slower than typical; Service B's are
over 20× slower. The mean collapses the whole distribution into one number and can't see shape: a few
huge outliers and many fast requests can average out to the same value as uniformly moderate requests.
If you call B many times per user action (or fan out to it), you'll hit that 900 ms tail often.

**6.** No. Percentiles don't average: the fleet p99 depends on the full combined distribution. If one
server is badly slow and nine are fine, averaging their p99s gives a meaningless middle number. Instead,
collect **histograms** (or mergeable sketches like HDR histogram / t-digest) from each server, merge
them, and compute p99 from the merged distribution.

**7.** Several reasons:
- The server only times its own processing; it doesn't see time spent **queued** before it picked the
  request up (in the accept queue, the load balancer, a thread pool).
- Network time, TLS handshakes, DNS, and client-side processing aren't counted.
- **Coordinated omission** in measurement: the slowest periods produce the fewest samples.
- You may be looking at averages or p50 instead of the tail.

**8.**
(a) **Available, not durable:** an in-memory cache with no persistence. Answers instantly, loses all
data on restart.
(b) **Durable, not available:** a database during a maintenance window, or a crashed primary awaiting
recovery. Committed data is safe on disk, but no one can read it right now.
(c) **Available, not reliable:** a service that responds to every request but sometimes returns stale
or wrong results, e.g. a replica serving very lagged data, or a bug that double-applies a payment.

**9.** In series, availabilities multiply: `0.999^4 ≈ 0.996` (99.6%), and that's before counting your
own failures. Ways to raise it:
- Make dependencies **soft**: if a non-critical dependency fails, degrade gracefully (show the page
  without recommendations) rather than failing the whole request.
- Add **caching/fallbacks** so you can serve stale data when a dependency is down.
- Make work **asynchronous** via a queue so a dependency being down delays work rather than failing it.
- Make the dependencies themselves redundant.
- Remove dependencies you don't truly need on the critical path.

**10.** The `1 − 0.001²` math assumes the two replicas fail **independently**. In practice they often
share failure causes: same rack or power, same availability zone, same network switch, same software
version with the same bug, same bad config push, same overloaded upstream. Correlated failures can take
both down at once, so real availability is far below six nines.

**11.** With MTBF = 1000 h, MTTR = 1 h: availability = 1000/1001 ≈ 99.900%.
- Halve frequency (MTBF = 2000 h): 2000/2001 ≈ 99.950%.
- Halve duration (MTTR = 0.5 h): 1000/1000.5 ≈ 99.950%.

They give (nearly) the same gain. But **reducing MTTR is usually easier**: automatic failover, fast
rollbacks, good alerting and runbooks are well-understood engineering, while preventing failures
requires anticipating every possible cause.

**12.** A **fault** is one component misbehaving (a disk dies, a process crashes). A **failure** is the
whole system stopping to serve users. Fault tolerance aims to stop faults becoming failures.
Hardware faults are frequent but mostly **random and independent**: one disk dying doesn't make another
die, so redundancy handles them well. Software faults are **correlated**: every node runs the same
code, so a bug triggered by a particular input, date, or load pattern can hit all nodes at once.
Redundancy doesn't protect you when every replica has the same bug.

**13.** Two things: (1) **which load parameters** you mean (requests/s, read/write ratio, data size,
concurrent users, fan-out, skew); (2) **what acceptable performance** means under that load
(throughput targets, latency percentiles such as p99 < 200 ms). Then the question becomes concrete:
"if X doubles, does Y stay within target, and how many resources does that take?" In the feed
example, fan-out on write inserts each post into every follower's feed, so one post by an account with
10 million followers means 10 million writes. The driving parameter is **the distribution of followers
per user**, not just the total post rate. Typical fix: a hybrid (fan out on write for normal accounts,
fetch on read for huge ones and merge).

**14.** Possible explanations:
- **Coordination overhead**: if nodes must talk to each other (replicate, agree, gossip), communication
  can grow faster than node count, eating the added capacity.
- **Contention** on a shared resource (a lock, a single leader, a hot partition) means extra nodes just
  add more waiters.
- **Rebalancing** in progress: data is being moved to the new node, consuming bandwidth and I/O.
- **Cache dilution**: each node now sees a smaller slice of traffic, so local caches are colder.

**15.** A user logs in on server A, then the next request goes to server B, which has no session: the
user appears logged out. Fixes:
- **Move the state out**: store sessions in a shared store (a distributed cache or database), making
  app servers stateless. Usually preferred.
- **Put state in the client**: a signed token (signed cookie or JWT) the server can verify without a
  lookup.
- **Sticky sessions**: have the load balancer always route a user to the same server. Works, but load
  becomes uneven and the session is lost if that server dies (topic 09).

Generally: stateless app servers can be added by "start it and register with the load balancer."
Database servers are stateful: a new node must know which data it owns, get a copy, stay in sync with
writes, and keep everything consistent during the move. That's replication and partitioning.

**16.** When the current machine isn't near the top of what's available; when the component is hard to
distribute (a relational database with complex transactions and joins); when the team wants to avoid
distributed-systems complexity; when the workload fits on one machine and engineering effort is the
main cost. Vertical scaling needs no code changes and avoids partial failure, consistency, and
rebalancing problems. You still need redundancy (a standby) for availability, but that's simpler than
full horizontal scaling.

**17.**
(a) **CDN**: serve images from edge locations near users. New problem: cached content can be stale;
you need a way to update or purge it (cache invalidation, versioned URLs).
(b) **Cache** (and/or read replicas): serve repeated reads from memory. New problem: staleness and
invalidation; cache failure modes such as stampedes when hot keys expire (topic 12).
(c) **Queue + workers**: accept the order, enqueue "send email," return immediately; a worker sends it
later. New problems: messages may be delivered more than once (duplicate emails), failures happen out
of the user's sight, and you need retries and dead-letter handling.
(d) **Load balancer** + more stateless app servers. New problems: the load balancer is a new single
point of failure, and any state held on app servers (sessions, local caches) now needs to be shared or
accepted as per-instance.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Latency vs throughput, in one line each?
**A:** Latency = time for one unit of work. Throughput = units of work completed per unit time.

**Q:** Response time vs service time?
**A:** Response time is what the client sees end to end (network + queueing + service); service time is only the time spent actually working.

**Q:** Bandwidth vs throughput?
**A:** Bandwidth is the channel's maximum capacity; throughput is what you actually achieve.

**Q:** State Little's Law.
**A:** `L = λ × W`: requests in flight = arrival rate × time each spends in the system.

**Q:** Why does batching raise throughput but hurt latency?
**A:** It amortizes fixed costs over many items (more work/sec), but early items wait for the batch to fill.

**Q:** What happens to latency as utilization approaches 100%?
**A:** It grows non-linearly (≈ service time / (1 − utilization)): 2× at 50%, 10× at 90%, 100× at 99%.

**Q:** Why do averages mislead for latency?
**A:** Latency distributions are right-skewed; the mean describes no real request and hides the tail.

**Q:** What is p99?
**A:** The value below which 99% of requests fall; 1 in 100 requests is slower.

**Q:** Tail latency amplification formula for fan-out to N calls?
**A:** P(at least one slow) = 1 − (1 − p)^N. With p = 1%, N = 100 → ~63%.

**Q:** Can you average p99s across servers?
**A:** No. Merge the histograms/sketches, then compute the percentile.

**Q:** Coordinated omission?
**A:** A measurement bias where the load generator waits on slow responses, so it under-samples slow periods.

**Q:** Availability vs reliability?
**A:** Availability: is it answering now? Reliability: does it keep answering *correctly* over time?

**Q:** Availability vs durability?
**A:** Availability is about the service responding; durability is about acknowledged data not being lost.

**Q:** Availability in terms of MTBF/MTTR?
**A:** MTBF / (MTBF + MTTR). Improve it by failing less often or recovering faster.

**Q:** Downtime per year at 99.9% / 99.99%?
**A:** ~8.8 hours / ~53 minutes.

**Q:** Availability of components in series vs parallel?
**A:** Series: multiply availabilities (gets worse). Parallel: multiply unavailabilities (gets better, if independent).

**Q:** Fault vs failure?
**A:** Fault: one component deviates from spec. Failure: the whole system stops serving users.

**Q:** Why are software faults more dangerous than hardware faults?
**A:** They're correlated: every node runs the same code, so one bug can hit all replicas at once.

**Q:** What are load parameters?
**A:** Numbers that describe load for your architecture: req/s, read/write ratio, data size, concurrency, fan-out, skew.

**Q:** Performance problem vs scalability problem?
**A:** Performance: slow for one user. Scalability: fast for one user, slow under heavy load.

**Q:** Two things that stop linear scaling?
**A:** Contention (serialized work, Amdahl's law) and coordination cost between nodes.

**Q:** Vertical vs horizontal scaling?
**A:** Vertical: bigger machine, simple, hard ceiling, SPOF. Horizontal: more machines, high ceiling, distributed complexity.

**Q:** Stateless component?
**A:** Holds nothing between requests that another instance would need; any instance can serve any request.

**Q:** Why are stateless tiers easy to scale?
**A:** Add instances behind a load balancer; no data to move, copy, or keep consistent.

**Q:** Standard pattern for state?
**A:** Push state down into a few specialized stateful systems (DB, cache, queue, object store); keep everything above them stateless.

**Q:** Job of a load balancer?
**A:** Single entry point that spreads requests across healthy servers and hides individual server failures.

**Q:** Job of a CDN?
**A:** Cache content at edge locations near users to cut latency and origin load.

**Q:** Job of a queue?
**A:** Decouple producers from consumers: accept work now, process later, absorb spikes, enable retries.

**Q:** Job of object storage?
**A:** Cheap, durable storage for large immutable blobs addressed by key.
