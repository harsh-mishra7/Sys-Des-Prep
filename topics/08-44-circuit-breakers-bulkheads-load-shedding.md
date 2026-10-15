# 44 — Circuit Breakers, Bulkheads & Load Shedding

> Part 8 — Reliability & Operations · Prerequisites: topics 02, 26, 43

Topic 43 gave every caller a timeout and a disciplined retry policy. That bounds how long one request
waits, but it doesn't stop a caller from sending thousands of doomed requests per second to a dependency
that is clearly down, it doesn't stop one slow dependency from eating all of a service's threads, and it
doesn't help a server that is simply receiving more work than it can do. This topic covers the
mechanisms that contain failure *inside* a service and at its front door: **circuit breakers** (stop
calling something that's broken), **bulkheads** (don't let one dependency consume resources everyone
needs), **backpressure, load shedding and admission control** (refuse or slow work you can't finish), and
**graceful degradation** (when something must give, give up the least important thing). Together with
topic 43 these are the core "stability patterns" that keep a partial failure partial.

By the end of this topic you should be able to:
1. Explain the circuit breaker state machine (closed, open, half-open) and how it decides to trip.
2. Describe what problems circuit breakers solve and the problems they can cause.
3. Explain bulkheads and how to size per-dependency resource pools.
4. Distinguish backpressure from load shedding, and explain admission control and what signals drive it.
5. Explain why queues must be bounded and why LIFO or CoDel-style queueing helps under overload.
6. Design graceful degradation: fallbacks, feature prioritization, and the pitfalls of fallback paths.

---

## 1. The problem: partial failure spreads

A typical service depends on several others:

```
                       ┌──► recommendations (slow!)
 requests ──► product ─┼──► inventory
             service   ├──► pricing
                       └──► reviews
```

Suppose recommendations becomes slow (each call now takes 5 s until timeout) while everything else is
healthy. Even with a timeout, the product service's shared pool of 200 worker threads fills up with
requests waiting on recommendations. Product pages that could have been rendered fine without
recommendations now fail. A non-essential feature has taken down an essential one.

Three distinct things went wrong, and each pattern in this topic addresses one:

| Problem | Pattern |
|---|---|
| We kept sending requests to something obviously broken, and waiting the full timeout each time | Circuit breaker |
| One dependency's slowness consumed resources shared with all other work | Bulkhead |
| Nobody decided what to do *instead*; the whole page failed | Graceful degradation / fallback |

And a fourth, at the front door: if the *service itself* is overloaded, it must refuse some work early
rather than accept everything and fail everything. That's load shedding and admission control.

---

## 2. Circuit breakers

### 2.1 The idea

Named after the electrical device that cuts power when current is dangerous: when a dependency is
failing, a circuit breaker in the *caller* stops sending it requests for a while and fails them
immediately instead. Popularized by Michael Nygard's *Release It!* and by Netflix's Hystrix library.

Benefits:

- **Fail fast:** callers get an error in microseconds rather than waiting for a timeout, so threads are
  freed and users get a quick (possibly degraded) response.
- **Give the dependency room to recover:** a struggling service stops receiving traffic it can't handle,
  which breaks the retry/overload feedback loop from topic 43.
- **Make the failure explicit:** an open breaker is a clear signal for fallbacks and alerts.

### 2.2 The state machine

```
                   failure rate over threshold
            ┌────────────────────────────────────────────┐
            │                                            ▼
     ┌────────────┐                              ┌────────────┐
     │   CLOSED   │                              │    OPEN    │
     │ calls flow │                              │ calls fail │
     │ normally;  │                              │ immediately│
     │ count      │                              │ (no network│
     │ outcomes   │                              │  call)     │
     └────────────┘                              └────────────┘
            ▲                                            │
            │ trial calls succeed                        │ cool-down timer expires
            │                                            ▼
            │                                    ┌────────────┐
            └────────────────────────────────────│ HALF-OPEN  │
                                                 │ allow a few│
                    trial call fails ──────────► │ trial calls│
                    (back to OPEN, often with    └────────────┘
                     a longer cool-down)
```

- **Closed:** normal. Every call goes through; the breaker records success/failure (and often latency) in
  a sliding window.
- **Open:** the breaker has tripped. Calls are rejected immediately with a "circuit open" error, or routed
  to a fallback. No traffic reaches the dependency. A timer runs.
- **Half-open:** after the cool-down, the breaker lets a limited number of trial requests through. If they
  succeed, it closes; if they fail, it reopens (often with an increased cool-down, i.e. backoff).

### 2.3 Deciding when to trip

Naive breakers trip after "N consecutive failures". Better ones use a **failure rate over a sliding
window with a minimum volume**:

```
 Trip if, in the last 10 seconds:
   requests >= 20              (minimum volume: don't trip on 2 failures out of 3)
   AND failure rate >= 50%
```

Design choices:

- **What counts as a failure:** timeouts, connection errors, 5xx, and often **slow calls** (calls above a
  latency threshold), since a slow dependency is as harmful as a failing one. Client errors (4xx) should
  *not* count: a flood of bad input from one caller shouldn't cut off everyone.
- **Count-based vs time-based windows:** last N calls vs last T seconds. Time-based handles variable
  traffic better.
- **Minimum volume** avoids tripping at low traffic where a few failures are statistically meaningless.
- **Cool-down duration:** long enough for real recovery, short enough not to prolong a blip. Often with
  exponential backoff on repeated reopenings.
- **Half-open trial count:** one probe is fragile (one unlucky failure); a handful is more robust. Some
  implementations ramp traffic gradually rather than snapping fully closed, to avoid a thundering herd
  onto a just-recovered service.

### 2.4 Granularity: what does one breaker protect?

A breaker should cover a unit that fails together:

- **Per dependency** (one breaker for "the reviews service") is the simplest.
- **Per endpoint/operation** is better if one endpoint is broken and others aren't (e.g., `POST /reviews`
  failing while `GET /reviews` works).
- **Per host/instance** prevents one bad instance from tripping the breaker for a healthy fleet, though
  this overlaps with load balancer outlier ejection (topic 09).

Too coarse, and one bad shard of a sharded store (topic 25) opens the breaker for all shards. Too fine,
and each breaker sees too little traffic to make a decision.

### 2.5 Problems circuit breakers can cause

- **Each caller sees a different picture.** Breakers are local state in each client instance. With 500
  client instances, the "open/closed" decision is made 500 times independently. That's usually fine, even
  desirable, but it means the dependency sees gradual, uneven reductions rather than a clean cutoff.
- **Tripping on the wrong signal.** If a single noisy caller sends invalid requests that cause 500s, the
  breaker may open for everyone sharing that client. Or a breaker that counts slow calls trips during a
  legitimate traffic surge.
- **Turning partial failure into total failure.** If 30% of requests fail because one of three shards is
  down, and the breaker opens at 25%, you've converted a 30% outage into 100%. Fine-grained breakers or
  thresholds above the expected partial-failure rate mitigate this.
- **Recovery flapping:** half-open lets traffic in, the cold dependency (empty caches, fresh JVM) is slow,
  trial calls time out, breaker reopens. Gradual ramp-up helps.
- **Hard to test:** the open state is exercised only during real failures. Teams use fault injection
  (topic 46/47 context: chaos experiments) to verify fallbacks actually work.

### 2.6 Breaker vs retry: how they interact

Retries (topic 43) and breakers work at different time scales. Retries handle individual transient
failures (milliseconds). Breakers handle sustained failures (seconds to minutes). Put the retry *inside*
the breaker's view: if a call fails after its retries, that counts as one failure to the breaker; once the
breaker is open, no retries happen at all because no calls happen. A retry budget and a breaker are
complementary: the budget limits extra load during partial failure, the breaker stops load during total
failure.

---

## 3. Bulkheads

### 3.1 The idea

A ship's hull is divided into watertight compartments (bulkheads), so a breach floods one compartment
instead of sinking the ship. In software, a **bulkhead** partitions a resource so that exhaustion in one
partition doesn't affect the others.

### 3.2 Per-dependency resource pools

Back to the opening example. Instead of one pool of 200 threads shared by all outbound calls:

```
 WITHOUT bulkheads                         WITH bulkheads
 ┌──────────────────────────────┐          ┌─────────────┐ ┌─────────────┐
 │ shared pool: 200 threads     │          │ recs: 30    │ │ inventory:60│
 │ ████████████████████████████ │          │ ███████████ │ │ ███░░░░░░░░ │
 │ all stuck on recommendations │          │ full → fail │ │ healthy     │
 └──────────────────────────────┘          └─────────────┘ └─────────────┘
 → every request blocked                   ┌─────────────┐ ┌─────────────┐
                                           │ pricing: 60 │ │ reviews: 30 │
                                           │ ██░░░░░░░░░ │ │ █░░░░░░░░░░ │
                                           └─────────────┘ └─────────────┘
                                           → only recommendation calls fail
```

When recommendations is slow, its 30 slots fill up, further recommendation calls are rejected immediately
(fail fast, like an open breaker), and inventory/pricing/reviews keep their own capacity. Product pages
render without recommendations.

The bulkhead can be a thread pool (Hystrix's original approach), a **semaphore** (a counter limiting
concurrent in-flight calls, cheaper than threads, the common choice for async code), or a connection pool
per downstream.

### 3.3 Sizing a bulkhead

Use Little's Law (topic 02): needed concurrency = request rate × latency.

```
 inventory: 400 calls/s × 50 ms p99 = 20 concurrent → size ≈ 20 × headroom (say 2×) = 40
```

When the dependency slows to its timeout (say 1 s), the same 400 calls/s would need 400 slots; the
bulkhead caps it at 40 and rejects the rest. That cap *is* the protection. Too small and you reject
healthy traffic at normal peaks; too large and the protection is weak.

### 3.4 Other bulkhead shapes

The same idea applies at every scale:

- **Per-tenant or per-customer pools/quotas**, so one noisy customer can't consume a shared service (ties
  to rate limiting, topic 45).
- **Separate instance pools by traffic class:** serve interactive user traffic and batch/background jobs
  from separate server fleets or separate queues, so a batch job can't starve users.
- **Separate database connection pools** for critical and non-critical queries.
- **Separate clusters / cells** for groups of customers: cell-based architecture (topic 48) is a
  bulkhead at the infrastructure level.
- **Kubernetes resource limits** (CPU/memory per container) are bulkheads for co-located processes.

The cost of bulkheads is **efficiency**: partitioned capacity can't be borrowed. A pool sitting idle can't
help a busy one. You trade some utilization for isolation, which is almost always worth it for critical
paths.

---

## 4. Backpressure, load shedding and admission control

Circuit breakers and bulkheads protect a *caller* from its dependencies. This section is about a *server*
protecting itself from its callers when offered load exceeds capacity.

### 4.1 Why accepting everything fails everyone

Topic 02's queueing curve: as utilization approaches 100%, wait time grows without bound. If a server
accepts every request into an unbounded queue during overload:

```
 capacity: 1,000 req/s     offered: 1,200 req/s     client timeout: 1 s

 queue grows by 200 req/s
 after 5 s: 1,000 queued → a new request waits ~1 s before starting → it times out
 the server is still processing requests from 1 s ago, whose clients gave up
 → nearly 100% of responses arrive too late; goodput ≈ 0
```

The server is working at full capacity and delivering almost nothing useful. By contrast, if it rejects
the excess 200 req/s immediately, it serves 1,000 req/s successfully: 83% success instead of ~0%.
**Under overload, saying no to some requests is how you say yes to the rest.**

### 4.2 Backpressure vs load shedding

Both respond to "too much work", in different ways:

| | Backpressure | Load shedding |
|---|---|---|
| What it does | Signals upstream to **slow down**; the producer waits | **Drops/rejects** excess work |
| Where natural | Pipelines where the producer can pause: streams, queues, TCP | Request/response services with impatient clients |
| Example | TCP flow control (topic 03); a Kafka consumer simply reads slower (topic 37); reactive streams' `request(n)`; a bounded queue that blocks the producer when full | Returning 503/429 immediately when a server is over its limit |
| Effect on work | Delayed, not lost | Lost (client may retry later) |
| Requires | A producer that *can* slow down and has somewhere to buffer | Clients that handle rejection well (backoff, topic 43) |

Backpressure propagates: if a stage blocks its producer, that producer blocks its own producer, all the
way back to the source. Eventually the source is something that cannot wait (a user, a sensor, an external
partner). At that boundary, backpressure turns into load shedding: somewhere, something must be dropped
or rejected. The design question is *where* and *what*.

### 4.3 Bounded queues

Every queue in a system should be bounded. An unbounded queue doesn't prevent overload; it hides it and
converts it into latency (and eventually memory exhaustion). With a bound, overflow becomes an explicit
signal: block the producer (backpressure) or reject (shed).

How long should a queue be? Short. A queue's useful job is absorbing short bursts. A useful bound is in
*time*, not items: "no request should wait longer than X ms in the queue", with X well below the client
timeout.

### 4.4 Smarter queue disciplines

- **Drop expired requests:** with deadline propagation (topic 43), check the deadline when dequeuing and
  discard requests whose client has given up, without doing the work.
- **LIFO under overload:** in FIFO, during a backlog every request waits behind old ones and all of them
  are late. Serving newest-first under overload means recent requests (whose clients are still waiting)
  succeed fast and the old ones (probably already abandoned) time out. Facebook described using
  adaptive LIFO for this reason.
- **CoDel (Controlled Delay):** borrowed from network routers. Track the minimum queueing delay over an
  interval; if even the minimum exceeds a target (e.g., 5–100 ms), the queue is standing, not absorbing a
  burst, so start dropping or shrink the allowed wait. Good at distinguishing bursts (fine) from
  sustained overload (shed).

### 4.5 Admission control: deciding what to let in

**Admission control** is the decision at the front door: admit this request, or reject it now? The
simplest form is a **concurrency limit**: allow at most N in-flight requests; reject the N+1th with 503.

Signals that can drive admission:

- **In-flight count** vs a limit (a semaphore at the server, a bulkhead on the receiving side).
- **Queue time / latency:** if requests are spending more than X ms queued, shed.
- **CPU utilization** or other resource saturation.
- **Adaptive limits:** algorithms like those in Netflix's `concurrency-limits` library borrow TCP
  congestion control ideas (topic 03): increase the allowed concurrency while latency stays near its
  minimum, cut it when latency rises (AIMD: additive increase, multiplicative decrease, or gradient
  methods). The server discovers its own capacity instead of relying on a hand-tuned number that goes
  stale.

Rejection must be **cheap**. If rejecting a request costs nearly as much as serving it (parsing a large
body, TLS handshake, auth lookups), the server can still collapse while "only rejecting". Shed as early as
possible, ideally at the load balancer or before expensive work.

**Client-side throttling** complements server-side shedding: the Google SRE book describes clients
tracking their own ratio of requests to accepts, and locally rejecting a fraction of requests when the
backend is rejecting many, so overloaded backends don't even spend effort on rejections.

### 4.6 Prioritized shedding

Not all requests are equal. When shedding, drop the least valuable first:

```
 priority     example                               shed when
 ─────────────────────────────────────────────────────────────────
 CRITICAL     checkout, payment, login               last
 DEFAULT      product pages, search                  heavy overload
 SHEDDABLE    recommendations, prefetch, analytics   first
 BATCH        reindex jobs, report generation        before anything user-facing
```

Requests carry a priority (a header, or derived from the endpoint), and admission thresholds differ per
priority: e.g., shed SHEDDABLE at 80% capacity, DEFAULT at 95%, CRITICAL only at hard limits. Retries can
be given lower priority than first attempts, which naturally damps retry storms.

---

## 5. Graceful degradation and fallbacks

### 5.1 Degrade, don't fail

**Graceful degradation** means delivering a reduced but useful experience when part of the system is
unavailable, instead of failing entirely. The product page without recommendations; search results
without personalization; a cached balance labeled "as of 5 minutes ago"; checkout that accepts the order
and processes payment asynchronously.

To do this you must first know which features are **essential** and which are **optional**, for each
user-facing operation. That's a product decision as much as an engineering one.

### 5.2 Fallback options

| Fallback | Example | Caveat |
|---|---|---|
| **Omit** the optional part | Hide the recommendations widget | Simplest and safest |
| **Static/default** value | Show "bestsellers" list instead of personalized recs | Must be reasonable for everyone |
| **Cached/stale** data | Serve last-known price from cache, stale-while-revalidate (topic 11) | Correctness: is stale acceptable here? (A stale price on checkout may not be) |
| **Degraded computation** | Simpler ranking model, fewer search shards, lower image quality | Must be cheaper than the normal path |
| **Defer the work** | Accept order, queue the email/charge for later (topic 36) | Need durable queue and user-visible state |
| **Fail closed with a clear message** | "Payments are temporarily unavailable" | Sometimes the honest answer is the right fallback |

### 5.3 Fallback pitfalls

Fallbacks are code paths that run rarely, which makes them dangerous:

- **Untested fallbacks:** the fallback has a bug that nobody noticed because it only runs during
  outages. Amazon's Builders' Library article "Avoiding fallback in distributed systems" argues for
  preferring designs that make the primary path more reliable over adding rarely-exercised fallbacks.
- **Fallbacks that cause load:** the fallback for "cache unavailable" is "read from the database", which
  sends 100× the normal traffic to the database and takes it down too. A fallback must not be more
  expensive than the primary, or it must be rate-limited itself.
- **Fallbacks that depend on the same thing:** falling back from service A to service B, where B
  depends on the same database that's failing.
- **Silent degradation:** users and operators don't notice the system is degraded; the "temporary"
  fallback becomes permanent. Emit metrics whenever fallbacks are used (topic 46).

A good fallback is cheap, independent of the failed component, exercised regularly (e.g., through fault
injection or by running it in production for a small slice of traffic), and observable.

### 5.4 Putting the patterns together

A single outbound call path, with the patterns layered:

```
 request ──► [admission control / priority shedding]   ← server protects itself
              │
              ▼
         handler: needs recommendations
              │
              ▼
         [bulkhead: recs semaphore (30)] ── full ──────► fallback
              │
              ▼
         [circuit breaker: recs] ── open ──────────────► fallback
              │
              ▼
         [timeout + deadline + budgeted retry]          ← topic 43
              │
              ▼
         network call to recommendations
              │ failure/timeout ─── recorded by breaker ► fallback
              ▼
         success
```

---

## 6. How it connects

- **Topic 02:** queueing near 100% utilization explains why shedding beats queueing; Little's Law sizes
  bulkheads.
- **Topic 03:** TCP flow control is backpressure; TCP congestion control inspired adaptive concurrency
  limits.
- **Topic 09:** load balancer health checks and outlier ejection act like per-instance breakers; the LB is
  the cheapest place to shed.
- **Topic 10:** service meshes and gateways implement breakers, concurrency limits and outlier detection as
  configuration.
- **Topic 11, 12:** stale cache data is a common fallback; a cache-unavailable fallback to the DB is a
  classic stampede.
- **Topic 26:** these patterns are how you keep partial failure partial.
- **Topic 36, 37:** consumers apply natural backpressure by reading slower; bounded queues are the same
  idea in-process.
- **Topic 43:** timeouts, retries, deadlines and budgets are the inner layer; breakers stop retry storms;
  shedding is how you exit metastable failure.
- **Topic 45:** rate limiting is admission control per client; 429 vs 503 distinguishes "you're over your
  quota" from "I'm overloaded".
- **Topic 46:** breaker state, rejection counts and fallback usage must be monitored.
- **Topic 48:** cells and shuffle sharding are bulkheads at the infrastructure level.

---

## Common misconceptions

- "A timeout is enough protection." → A timeout bounds each call, but at high traffic even bounded waits
  consume all threads. Bulkheads cap concurrency; breakers stop the calls entirely.
- "Circuit breakers fix the failing service." → They protect the caller and give the dependency a
  chance to recover; they don't fix anything themselves.
- "Open breaker = dependency is down." → It means *this caller's* recent sample crossed a threshold.
  Others may see it differently.
- "Queues absorb overload." → Queues absorb short bursts. Under sustained overload an unbounded queue just
  turns overload into latency and timeouts for everyone.
- "Rejecting requests is a failure; we should serve everyone." → Under overload, rejecting some is the
  only way to serve any at all.
- "Backpressure and load shedding are the same." → Backpressure slows the producer and preserves work;
  shedding drops work. Backpressure eventually must turn into shedding at a source that can't wait.
- "Fallbacks always improve reliability." → Untested or expensive fallbacks can cause outages,
  especially when the fallback hammers another shared component.
- "Bulkheads are free." → They cost utilization: partitioned capacity can't be borrowed.

## Further reading (supplements)

- Michael Nygard, *Release It!* (2nd ed.), chapter "Stability Patterns" (circuit breaker, bulkhead,
  fail fast, shed load, back pressure).
- *Site Reliability Engineering* (Google), chapters 21 "Handling Overload" and 22 "Addressing Cascading
  Failures".
- Amazon Builders' Library: "Using load shedding to avoid overload" and "Avoiding fallback in distributed
  systems".
- Martin Fowler, "CircuitBreaker" (martinfowler.com bliki).
- Netflix Tech Blog: Hystrix design notes; "Performance Under Load" (adaptive concurrency limits).
- Kathleen Nichols & Van Jacobson, "Controlling Queue Delay", ACM Queue (2012) (CoDel).
- Ben Maurer, "Fail at Scale", ACM Queue (2015) (adaptive LIFO and CoDel in Facebook's services).

---

## Questions

1. A non-essential dependency becomes slow (not down). Explain how it can take down an essential feature
   of the calling service, even with timeouts in place.
2. Describe the three states of a circuit breaker and what triggers each transition.
3. Why should a breaker use a failure *rate* with a *minimum request volume* rather than "5 consecutive
   failures"?
4. One of three shards of a database is down, so 33% of calls fail. The breaker is configured to open at 25%
   failures, per dependency. What happens, and how would you fix the configuration?
5. Why should 4xx errors usually not count toward tripping a breaker?
6. A service makes 300 calls/s to a dependency with p99 latency 40 ms and a 2 s timeout. Size a bulkhead
   for it and explain what the bulkhead does if the dependency degrades to 2 s per call.
7. What is the efficiency cost of bulkheads, and when is it worth paying?
8. A server's capacity is 1,000 req/s, it receives 1,300 req/s, and it uses an unbounded FIFO queue.
   Clients time out at 1 s. Describe what happens over a minute, and what happens if it sheds instead.
9. Distinguish backpressure from load shedding. Why must backpressure eventually become load shedding
   somewhere?
10. Why can LIFO queueing be better than FIFO during overload?
11. Why must rejection be cheap, and where is the cheapest place to shed?
12. Your cache cluster goes down. The fallback is "read directly from the database". What could go wrong,
    and what would a better fallback look like?
13. How do circuit breakers and retry budgets complement each other?
14. Give an example of prioritized load shedding for a shopping site, and explain why first attempts might
    be prioritized over retries.

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** The slow calls hold the caller's shared threads/connections until they time out. By Little's Law,
concurrency = rate × latency; if latency rises from 50 ms to a 5 s timeout, the same rate needs 100× the
slots. The shared pool fills with requests waiting on the non-essential dependency, so requests needing
only healthy dependencies can't get a thread and fail too. Timeouts bound each call but not the aggregate
concurrency. Bulkheads (separate pools) and breakers (stop calling) contain this.

**2.** Closed: calls pass through and outcomes are recorded; transitions to Open when the failure rate in
the window exceeds the threshold (with minimum volume). Open: calls fail immediately or go to a fallback;
after a cool-down timer it moves to Half-open. Half-open: a few trial calls are allowed; if they succeed it
closes, if they fail it reopens (often with a longer cool-down).

**3.** Consecutive-failure counts are sensitive to traffic levels and noise: at high volume, 5 consecutive
failures may be a tiny fraction of traffic; at low volume, 2 failures of 3 requests is statistically
meaningless but could look like a 66% failure rate. A rate over a sliding window with a minimum volume
trips only when there's enough evidence that a meaningful share of calls are failing.

**4.** The breaker sees 33% > 25% and opens, cutting off *all* calls, including the 67% that would have
succeeded on healthy shards. A partial outage becomes total. Fixes: breakers per shard/endpoint (the
failing shard's breaker opens, others stay closed), or a threshold above expected partial-failure rates,
combined with per-shard health tracking.

**5.** 4xx indicates the request was bad, not that the dependency is unhealthy. If one caller sends many
invalid requests, counting those would open the breaker and cut off valid callers sharing it. The breaker
is supposed to detect dependency health.

**6.** Needed concurrency ≈ 300 × 0.04 = 12; with ~2× headroom, ~25 slots. If the dependency degrades to
2 s, sustaining 300 calls/s would need 600 concurrent slots. The bulkhead caps it at 25: those 25 calls
wait, and everything beyond is rejected instantly (to a fallback). Only ~12 calls/s actually get through
(25 / 2 s), and the rest of the service keeps its threads.

**7.** Partitioned capacity can't be shared: an idle pool for dependency X can't absorb a burst for Y, so
total utilization is lower and you may need more total capacity. It's worth it for isolating critical
paths from non-critical ones, and for dependencies with a history of slowness. Without it, any one
dependency can take everything down.

**8.** The queue grows by 300 req/s. Within a few seconds queueing delay exceeds 1 s, so every request
waits past its client's timeout. The server keeps processing requests whose clients have given up: it's
fully busy with near-zero goodput; clients retry, worsening it; memory eventually runs out. If it sheds
instead, it rejects ~300 req/s immediately and serves 1,000 req/s within timeout: ~77% success, and clients
can back off.

**9.** Backpressure tells the producer to slow down (block, or grant fewer credits), preserving work but
delaying it. Load shedding rejects or drops work. Backpressure propagates upstream stage by stage until it
reaches a source that cannot slow down or buffer indefinitely: a human user, an external client, a
sensor. At that boundary the only remaining options are to buffer (bounded) and then drop/reject, i.e., to
shed.

**10.** In a FIFO backlog, every new request waits behind old ones, so all requests are late and likely
all time out. LIFO serves the newest requests first: their clients are still waiting, so they succeed
quickly; the oldest requests (likely already abandoned) are the ones that time out. Goodput stays high.
Typically used adaptively, only when a backlog exists.

**11.** If rejecting costs nearly as much as serving (TLS, parsing, authentication, DB lookups), then
during overload the server spends its capacity rejecting and still collapses. Shed as early as possible:
at the load balancer or gateway, or on the server before parsing the body; client-side throttling avoids
even the network round trip.

**12.** The database was sized for the cache's miss rate, maybe 5% of reads; suddenly it receives 100%,
a 20× increase, and falls over, turning a cache outage into a full outage (a fallback more expensive than
the primary, depending on a shared component). Better: serve stale data from a local in-process cache,
degrade optional features, rate-limit the fallback path to what the database can handle, and shed
excess requests.

**13.** A retry budget limits extra load during partial failures, when some requests succeed and retries
are useful but must not multiply load. A breaker handles sustained, widespread failure, stopping calls
(and therefore retries) entirely so the dependency can recover. Together: retries are cheap and bounded
during blips; traffic stops during outages.

**14.** Critical: checkout, payment, login; default: product pages, search, cart; sheddable:
recommendations, "customers also viewed", prefetch; batch: reports, reindexing. Under overload, shed batch
and sheddable first. First attempts are prioritized over retries because retries are the amplification
mechanism in retry storms; shedding retries first caps load near 1× and keeps fresh traffic succeeding.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What problem does a circuit breaker solve?
**A:** It stops a caller from sending requests to a clearly failing dependency, failing fast and giving the dependency room to recover.

**Q:** Three circuit breaker states?
**A:** Closed (normal, counting), Open (fail immediately), Half-open (a few trial calls to test recovery).

**Q:** Open → Half-open trigger?
**A:** A cool-down timer expiring.

**Q:** Half-open outcomes?
**A:** Trials succeed → Closed; trials fail → Open again, often with a longer cool-down.

**Q:** Good trip condition?
**A:** Failure rate (including slow calls) above a threshold in a sliding window, with a minimum request volume.

**Q:** Why not count 4xx toward the breaker?
**A:** They reflect bad requests, not dependency health; one bad caller could trip it for all.

**Q:** Risk of a coarse breaker on a sharded dependency?
**A:** One failing shard can open it for all shards, turning partial failure into total.

**Q:** What is a bulkhead?
**A:** A partition of resources (thread pool, semaphore, connection pool) per dependency or class so one can't exhaust shared capacity.

**Q:** How do you size a bulkhead?
**A:** Little's Law: rate × normal latency, plus headroom.

**Q:** Cost of bulkheads?
**A:** Lower utilization; partitioned capacity can't be borrowed.

**Q:** Why shed load rather than queue it?
**A:** Under sustained overload, queues grow until every request times out; shedding the excess lets the rest succeed.

**Q:** Backpressure vs load shedding?
**A:** Backpressure slows the producer (work delayed, kept); shedding rejects work (work lost).

**Q:** Why should every queue be bounded?
**A:** Unbounded queues hide overload as latency and memory growth; a bound makes overload an explicit signal.

**Q:** What is admission control?
**A:** Deciding at the front door whether to accept a request, based on concurrency, queue time, or resource signals.

**Q:** Adaptive concurrency limits?
**A:** Like TCP congestion control: raise the in-flight limit while latency stays low, cut it when latency rises.

**Q:** Why must rejection be cheap?
**A:** If rejecting costs as much as serving, the server collapses anyway. Shed early (LB, before parsing).

**Q:** Why LIFO under overload?
**A:** Newest requests still have waiting clients; serving them first keeps goodput high while abandoned old ones expire.

**Q:** What does CoDel detect?
**A:** A standing queue: when even the minimum queueing delay over an interval exceeds a target, it's sustained overload, not a burst.

**Q:** Prioritized shedding?
**A:** Requests carry priority; shed batch/sheddable first, critical (checkout, login) last; shed retries before first attempts.

**Q:** What is graceful degradation?
**A:** Giving a reduced but useful response (omit, default, stale, deferred) when part of the system fails.

**Q:** Classic dangerous fallback?
**A:** Cache down → read all traffic from the database, which then collapses.

**Q:** Properties of a good fallback?
**A:** Cheap, independent of the failed component, regularly exercised, and observable.

**Q:** 429 vs 503?
**A:** 429: you (this client) exceeded your quota; 503: the server is overloaded or unavailable.
