# Day 1: Answers

Read only after attempting `questions.md`.

---

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
If you call B many times per user action, you'll hit that 900 ms tail often.

**6.** `1 − 0.99^50 ≈ 1 − 0.605 ≈ 39.5%`. Nearly 40% of page loads are slow even though each service
is slow only 1% of the time. Lesson: in fan-out architectures, **backend tail latency (p99, p99.9)
becomes the user's typical latency**. Optimizing medians barely helps; reducing the tail (or using
techniques like hedged requests, timeouts with fallbacks, reducing fan-out) does.

**7.** No. Percentiles don't average: the fleet p99 depends on the full combined distribution. If one
server is badly slow and nine are fine, averaging their p99s gives a meaningless middle number. Instead,
collect **histograms** (or mergeable sketches like HDR histogram / t-digest) from each server, merge
them, and compute p99 from the merged distribution.

**8.** Several reasons:
- The server only times its own processing; it doesn't see time spent **queued** before it picked the
  request up (in the accept queue, the load balancer, a thread pool).
- Network time, TLS handshakes, DNS, and client-side processing aren't counted.
- **Coordinated omission** in measurement: the slowest periods produce the fewest samples.
- You may be looking at averages or p50 instead of the tail.

**9.**
a) **Available, not durable:** an in-memory cache with no persistence. Answers instantly, loses all
   data on restart.
b) **Durable, not available:** a database during a planned maintenance window or a crashed primary
   awaiting recovery. Committed data is safe on disk, but no one can read it right now.
c) **Available, not reliable:** a service that responds to every request but sometimes returns stale
   or wrong results, e.g. a replica serving very lagged data, or a bug that double-applies a payment.

**10.** In series, availabilities multiply: `0.999^4 ≈ 0.996` (99.6%), and that's before counting your
own failures. Ways to raise it:
- Make dependencies **soft**: if a non-critical dependency fails, degrade gracefully (show the page
  without recommendations) rather than failing the whole request.
- Add **caching/fallbacks** so you can serve stale data when a dependency is down.
- Make work **asynchronous** via a queue so a dependency being down delays work rather than failing it.
- Make the dependencies themselves redundant.
- Remove dependencies you don't truly need on the critical path.

**11.** The `1 − 0.001²` math assumes the two replicas fail **independently**. In practice they often
share failure causes: same rack or power, same availability zone, same network switch, same software
version with the same bug, same bad config push, same overloaded upstream. Correlated failures can take
both down at once, so real availability is far below six nines.

**12.** Do the math with MTBF = 1000 h, MTTR = 1 h: availability = 1000/1001 ≈ 99.900%.
- Halve frequency (MTBF = 2000 h): 2000/2001 ≈ 99.950%.
- Halve duration (MTTR = 0.5 h): 1000/1000.5 ≈ 99.950%.

They give (nearly) the same gain. But **reducing MTTR is usually easier**: automatic failover, fast
rollbacks, good alerting and runbooks are well-understood engineering, while preventing failures
requires anticipating every possible cause.

**13.** A **fault** is one component misbehaving (a disk dies, a process crashes). A **failure** is the
whole system stopping to serve users. Fault tolerance aims to stop faults becoming failures.

Hardware faults are frequent but mostly **random and independent**: one disk dying doesn't make another
die, so redundancy handles them well. Software faults are **correlated**: every node runs the same
code, so a bug triggered by a particular input, date, or load pattern can hit all nodes at once.
Redundancy doesn't protect you when every replica has the same bug.

**14.** (1) **Which load parameters** you mean: requests/s, read/write ratio, data size, concurrent
users, fan-out, etc. (2) **What acceptable performance** means under that load: throughput targets,
latency percentiles (e.g. p99 < 200 ms). Then the question becomes concrete: "if X doubles, does Y stay
within target, and how many resources does that take?"

**15.** Fan-out on write inserts each new post into every follower's feed. For a user with 10 million
followers, a single post means 10 million writes, which can take a long time and overload the system.
The driving load parameter is **the distribution of followers per user** (fan-out), not just the total
post rate. Typical fix: hybrid — fan out on write for normal accounts, fetch on read for very large
accounts and merge.

**16.** Possible explanations:
- **Coordination overhead**: if nodes must talk to each other (replicate, agree, gossip), communication
  can grow faster than node count, eating the added capacity.
- **Contention** on a shared resource (a lock, a single leader, a hot partition) means extra nodes just
  add more waiters.
- **Rebalancing** in progress: data is being moved to the new node, consuming bandwidth and I/O.
- **Cache dilution**: each node now sees a smaller slice of traffic, so caches are colder.

**17.** App servers are (ideally) **stateless**: any instance can serve any request, so adding one is
just "start it and register with the load balancer." Databases are **stateful**: a new node needs to
know which data it owns, get a copy of that data, stay in sync with writes, and keep everything
consistent during the move. That's replication and partitioning, which are hard.

**18.** With three servers, a user logs in on server A, then the next request goes to server B, which
has no session: the user appears logged out. Fixes:
- **Move the state out**: store sessions in a shared store (a distributed cache or database), making
  app servers stateless. Usually preferred.
- **Put state in the client**: a signed token (e.g. a signed cookie or JWT) the server can verify
  without lookup.
- **Sticky sessions**: have the load balancer always route a user to the same server. Works, but
  uneven load, and the user's session is lost if that server dies. (Covered on Day 4.)

**19.** When the current machine isn't near the top of what's available; when the component is hard to
distribute (a relational database with complex transactions and joins); when the team wants to avoid
distributed-systems complexity; when the workload is fine on one machine and the main concern is cost
of engineering effort. Vertical scaling needs no code changes and avoids partial failure, consistency,
and rebalancing problems. You still need redundancy (a standby) for availability, but that's simpler
than full horizontal scaling.

**20.**
a) **CDN**: serve images from edge locations near users. New problem: cached content can be stale;
   you need a way to update or purge it (cache invalidation, versioned URLs).
b) **Cache** (and/or read replicas): serve repeated reads from memory. New problem: staleness and
   invalidation; cache failure modes (stampedes when hot keys expire).
c) **Queue + workers**: accept the order, enqueue "send email," return immediately; a worker sends it
   later. New problems: messages may be delivered more than once (duplicate emails), failures happen
   out of the user's sight, and you need retries and dead-letter handling.
d) **Load balancer** + more stateless app servers. New problems: the load balancer is a new single
   point of failure, and any state held on app servers (sessions, local caches) now needs to be shared
   or accepted as per-instance.
