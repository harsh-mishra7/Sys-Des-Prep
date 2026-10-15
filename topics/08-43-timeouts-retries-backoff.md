# 43 — Timeouts, Retries & Backoff

> Part 8 — Reliability & Operations · Prerequisites: topics 02, 07, 26, 38

Every distributed system is built out of remote calls, and every remote call can fail in a way that a
local function call cannot: it can simply never come back. Topic 26 showed that over a network you cannot
tell a slow server from a dead one, or a lost request from a lost response. This topic is about the three
small tools every caller uses to live with that uncertainty: the **timeout** (decide when to stop waiting),
the **retry** (try again, because many failures are transient), and **backoff** (wait between attempts so
retrying doesn't make things worse). They look trivial. They are not: misconfigured timeouts and retries
are behind a large share of real outages, because the same mechanism that rescues one request during a
blip can, multiplied across thousands of clients and several layers, keep a system down long after the
original problem has gone away.

By the end of this topic you should be able to:
1. Explain why every remote call needs a timeout, what happens without one, and how to pick a value.
2. Distinguish connect, request, and end-to-end (deadline) timeouts, and explain deadline propagation.
3. Decide which failures are safe and useful to retry, and why idempotency (topic 38) is a precondition.
4. Describe exponential backoff with jitter and explain why jitter matters as much as the backoff.
5. Explain retry budgets and why "retry at one layer only" is a common rule.
6. Compute retry amplification across layers, and explain hedged requests and their cost.
7. Explain metastable failures and retry storms: why a system can stay broken after the trigger is gone.

---

## 1. The timeout: deciding when to stop waiting

### 1.1 What happens without a timeout

Suppose a checkout service calls a payment service, and the payment service's database hangs (a lock is
held, a disk stalls, a GC pause runs long). No error comes back. Nothing comes back.

```
 client ──► checkout (thread 1) ──► payment ──► DB (hung)
            waits...                 waits...
 client ──► checkout (thread 2) ──► payment
            waits...
 ...
 client ──► checkout (thread 200)   ← pool exhausted, checkout stops answering ANYTHING
```

Without a timeout the calling thread (or connection, or coroutine, or file descriptor) is held forever.
Each blocked call pins a resource. Requests keep arriving at the same rate, so the number of stuck
requests grows linearly with time until a pool runs out. Then the checkout service stops serving
*every* request, including ones that never needed the payment service (the product page, the health
check). A failure in one dependency has become a failure of the caller, and the caller's callers are
now stuck waiting on it. This is the basic mechanism of a **cascading failure**.

Little's Law (topic 02) makes this precise: concurrency = arrival rate × time in system. If 100 requests
per second arrive and each one now waits forever, in-flight requests grow by 100 every second with no
upper bound. A timeout puts a ceiling on "time in system", and therefore on the concurrency a slow
dependency can consume: at 100 rps and a 2 s timeout, at most ~200 requests are ever stuck.

**Rule:** every call that crosses a process boundary (network, database, cache, queue, DNS lookup, even
reading from a remote file system) needs an explicit timeout. Library defaults are frequently "infinite"
or several minutes, which in practice is the same thing.

### 1.2 What a timeout does and doesn't tell you

When a timeout fires, you know only that you didn't get an answer in time. You do *not* know:

- whether the request reached the server,
- whether the server processed it (the payment may have gone through),
- whether a response is still on its way.

So a timeout is an *unknown outcome*, not a failure. This is exactly why retries need idempotency
(section 2.3): retrying a timed-out "charge card" call without an idempotency key may charge twice.

A timeout also does not stop the work on the server. The server keeps computing an answer nobody will
read. Section 1.5 (deadline propagation) addresses this.

### 1.3 Kinds of timeouts

| Timeout | Bounds | Typical scale |
|---|---|---|
| **Connect timeout** | TCP handshake (+ TLS) to establish a connection | Small: tens to a few hundred ms within a datacenter. A dead host never completes the handshake, so this catches "host down" fast. |
| **Read / socket timeout** | Time between bytes, or waiting for the first byte of a response | Depends on the operation |
| **Request timeout** | Total time for one attempt of one request | Depends on the operation's latency distribution |
| **End-to-end deadline** | Total time the *original user request* may take, across all hops and retries | Set at the edge, from what the user/product can tolerate |
| **Idle timeout** | How long a pooled or keep-alive connection may sit unused | Seconds to minutes; must be coordinated with load balancers (topic 09) |

A common bug: setting only a socket *read* timeout of 5 s. A server that trickles one byte every 4 s
never trips it, and the call takes forever. Bound the total, not just the gaps.

Another common bug: idle-timeout mismatch. If a load balancer drops idle connections after 60 s and the
client's pool keeps them for 90 s, the client will periodically send a request on a connection that is
already dead and get a reset. Keep the client's idle timeout shorter than every middlebox's.

### 1.4 How to choose a timeout value

There is no universal number; the value comes from the dependency's latency distribution (topic 01's
percentiles) and from what the caller can afford.

1. **Measure the dependency's latency distribution** under normal load. Say p50 = 20 ms, p99 = 120 ms,
   p99.9 = 400 ms.
2. **Pick a percentile you are willing to give up on.** A timeout at roughly the p99.9 means about 0.1%
   of legitimate requests will be cut off. Setting it below p99 means you'll routinely abandon requests
   that would have succeeded, and if you retry them, you add load for nothing.
3. **Add headroom** so normal variance (a GC pause, a deploy) doesn't trip it, but keep it far below
   "forever".
4. **Check it fits the caller's budget.** If the user-facing request has a 1 s deadline and calls three
   services sequentially, each cannot have a 1 s timeout.

Trade-off in both directions:

```
 too SHORT                                   too LONG
 ─────────────────────────────────────────────────────────────────────
 abandons requests that would succeed        slow dependency ties up caller's
 retries add load to a struggling server     threads/connections; cascades
 false "failures" during normal variance     users wait long, then fail anyway
```

Timeouts should also be **revisited** when the dependency changes. A timeout tuned when p99 was 50 ms is
wrong after a schema change moves p99 to 300 ms: suddenly 5% of requests time out and get retried, and
you've built a load amplifier (section 4).

### 1.5 Deadlines and deadline propagation

Per-hop timeouts do not compose. Consider:

```
 user ──► A (timeout 1s) ──► B (timeout 1s) ──► C (timeout 1s)
```

If C takes 0.9 s, B gets the answer, does some work, and returns after 0.95 s; A gets it at 1.05 s. A has
already given up at 1.0 s and returned an error, but B and C did all the work anyway. Worse, if B retries
C after a timeout, B may keep working for 2+ seconds on behalf of a user who left long ago.

The fix is a **deadline**: an absolute point in time (or remaining budget) set once at the edge and passed
along with the request.

```
 edge: deadline = now + 1000ms
 A receives: 1000ms remaining. Spends 50ms. Calls B with "remaining = 950ms - margin"
 B receives: 930ms remaining. Spends 30ms. Calls C with ~890ms.
 C checks: if remaining <= 0 → reject immediately (don't do doomed work)
```

gRPC does this natively: a client deadline is transmitted with the call, and servers can check
`context.Deadline()` and pass the same context downstream. With HTTP you typically carry a header with
the remaining budget. Two benefits:

- **No wasted work.** A server that sees an already-expired deadline refuses the request instead of
  computing an answer nobody will read. Under overload this matters enormously: much of the work an
  overloaded server does is for requests whose clients already timed out.
- **Retries are bounded.** A retry is only attempted if enough budget remains for it to plausibly
  succeed.

Pass *remaining duration* rather than absolute wall-clock time when hosts' clocks may differ (topic 27),
or tolerate small skew by subtracting a margin.

---

## 2. Retries: trying again

### 2.1 Why retry at all

Many failures are **transient**: a packet dropped, a connection reset because a server restarted during a
deploy, a leader election in progress (topic 30), a load balancer sent you to an instance that was just
being drained. Trying again a moment later, usually on a different instance, succeeds. Retrying converts
these blips from user-visible errors into a few extra milliseconds of latency. If each attempt
independently fails with probability 1%, two attempts fail together with probability 0.01%.

The key word is *independently*. Retries help when failures are uncorrelated. When the failure is
"the service is overloaded", the next attempt is not independent: it hits the same overloaded service and
makes it more overloaded. Most of this topic is about that difference.

### 2.2 What to retry and what not to

| Outcome | Retry? | Why |
|---|---|---|
| Connection refused / connect timeout | Usually yes | Request never reached the server; safe even for non-idempotent ops; try another instance |
| Connection reset mid-request, read timeout | Only if idempotent | Outcome unknown; server may have processed it |
| HTTP 503 Service Unavailable, 429 Too Many Requests | Yes, with backoff, honoring `Retry-After` | Server explicitly says "later" |
| HTTP 502/504 from a proxy | Often yes, if idempotent | Upstream blip; outcome may be unknown |
| HTTP 500 | Cautiously | Might be a bug that fails deterministically |
| HTTP 400, 401, 403, 404, 409, 422 | No | The request itself is wrong; retrying gives the same answer |
| gRPC `UNAVAILABLE` | Yes | Defined as transient |
| gRPC `DEADLINE_EXCEEDED` | Only if budget remains (usually no) | |
| gRPC `INVALID_ARGUMENT`, `PERMISSION_DENIED` | No | Deterministic |

A good rule of thumb: retry errors that say "the system is temporarily unable", never errors that say
"your request is wrong". Servers help clients a lot by returning precise status codes (topic 07): an
overloaded server should say 503/429, not 500, so clients know retrying later is meaningful.

### 2.3 Idempotency is the precondition

From topic 38: an operation is **idempotent** if doing it twice has the same effect as doing it once.
`GET /order/42`, `PUT /order/42 {status: shipped}`, and `DELETE /order/42` are idempotent by HTTP
semantics. `POST /charges {amount: 50}` is not: two attempts can mean two charges.

Timeouts make outcomes unknown (section 1.2), so any retry after a timeout risks executing twice. For
non-idempotent operations the standard fix is an **idempotency key**: the client generates a unique ID
per logical operation, sends it with every attempt, and the server records completed keys and returns
the stored result for a repeated key instead of executing again.

```
 attempt 1: POST /charges  Idempotency-Key: 7f3a...  → server charges, response lost
 attempt 2: POST /charges  Idempotency-Key: 7f3a...  → server sees key, returns stored result
```

Without this, the only safe retries are those where you know the request never reached the server
(connect failures).

### 2.4 Where retries happen

Retries hide in many places, often more than people realize:

- Your application code (an explicit loop).
- The HTTP/gRPC client library (many retry by default).
- The service mesh sidecar or API gateway (topic 10), configured separately by another team.
- The database driver (reconnect and re-execute).
- The SDK of a cloud service.
- The **user**, who presses refresh, and the mobile app, which retries on network change.
- Message consumers: an unacknowledged message is redelivered (topics 36, 38), which is a retry.

Each of these is reasonable alone. Stacked, they multiply (section 4).

---

## 3. Backoff: waiting between attempts

### 3.1 Why not retry immediately

Immediate retries are fine for a single dropped packet. They are terrible when the failure is load-related:
the server is struggling, and every failed request comes straight back, doubling (or tripling) the load
at exactly the moment it can least handle it. Even for non-load failures, if the cause takes 2 seconds to
clear (a leader election), five retries in 10 ms all fail and you've given up before it had a chance to
recover.

**Backoff** means waiting before each retry, and waiting longer after each successive failure.

### 3.2 Exponential backoff

The wait after the *n*-th failure is:

```
 wait(n) = min(cap, base × 2^n)

 base = 100ms, cap = 10s:
 attempt:   1      2      3      4      5      6      7      8
 wait:    100ms  200ms  400ms  800ms  1.6s   3.2s   6.4s   10s (capped)
```

The exponential growth means a client quickly backs off to a low request rate when a dependency is
persistently down, while short blips are still retried quickly. The **cap** prevents absurd waits, and
there must also be a **maximum number of attempts** (or a deadline) so the client eventually gives up and
reports failure.

### 3.3 Why jitter matters

Exponential backoff alone has a subtle problem: **synchronization**. Suppose a database restarts and 10,000
clients all get an error at the same instant. With deterministic backoff they all wait exactly 100 ms and
retry at the same instant, then all wait 200 ms and retry at the same instant again.

```
 Without jitter (requests per ms over time):

 load │█               █                       █
      │█               █                       █
      │█               █                       █
      └──────────────────────────────────────────── time
       t=0           t=100ms                 t=300ms
       10k            10k                      10k     ← synchronized spikes

 With full jitter:

 load │▂▃▂▃▂▂▃▂▃▂▃▂▂▃▂▃▂▂▃▂▂▃▂▂▃▂▂▃▂▂▂▃▂▂▃▂▂▂▃▂▂▃▂▂▃
      └──────────────────────────────────────────── time
       same total requests, spread out → server can absorb them
```

The server sees periodic spikes of the full client population, which may be exactly what it cannot
handle, so it fails again and the cycle repeats. This is the **thundering herd** (you've seen its cache
cousin in topic 12).

**Jitter** adds randomness to the wait so clients spread out. The common variants (popularized by an AWS
Architecture Blog post, "Exponential Backoff and Jitter"):

```
 exp       = min(cap, base × 2^n)

 Full jitter:          wait = random(0, exp)
 Equal jitter:         wait = exp/2 + random(0, exp/2)
 Decorrelated jitter:  wait = min(cap, random(base, previous_wait × 3))
```

- **Full jitter** spreads clients the most and generally results in the least total work and fastest
  aggregate completion. Its downside is that some individual waits are very short.
- **Equal jitter** guarantees at least half the backoff, trading a bit of spread for a minimum wait.
- **Decorrelated jitter** grows based on the previous wait rather than the attempt count.

The details matter less than the principle: **always add jitter**. The same principle applies to anything
periodic that many machines do: cache TTLs (topic 12), cron jobs, heartbeats, client reconnects after a
WebSocket server restarts (topic 08), and polling intervals.

### 3.4 Honor the server's hints

If a server returns `Retry-After: 30` (HTTP) or a gRPC pushback / retry-delay hint, the client should wait at
least that long. The server knows its own state better than the client's generic backoff formula. Servers
under load can use this to spread the recovery deliberately.

---

## 4. Retry amplification across layers

### 4.1 The multiplication

Consider a request path with four layers, each of which tries up to 3 times (1 attempt + 2 retries):

```
 client ──► gateway ──► service A ──► service B ──► database
   ×3          ×3           ×3            ×3
```

If the database is down, how many requests reach it per user action?

```
 client makes 3 attempts
   each reaches the gateway, which makes 3 attempts           → 9
     each reaches A, which makes 3 attempts                   → 27
       each reaches B, which makes 3 attempts to the database → 81
```

**3⁴ = 81 database requests for one user click.** In general, with *k* layers each making *r* total
attempts, the bottom layer sees *rᵏ* times the load. The amplification is worst at the deepest layer,
which is often the shared, hardest-to-scale component (the database). And it happens at exactly the
moment that component is failing. A partially degraded database that could have handled 1× load is
pushed into total failure by 81×.

Latency multiplies too. If each layer waits for its own timeout before retrying, the user waits for the
full nested sequence, long after any useful answer could arrive.

### 4.2 Rules that contain amplification

1. **Retry at one layer only.** Typically either at the edge (closest to the user, who has the full
   context) or at the layer immediately above the failing dependency, not both. Intermediate layers fail
   fast and propagate the error. Some systems mark responses with "don't retry, I already did"
   (e.g., a header or status meaning "overloaded, retries exhausted") so upper layers don't retry again.
2. **Use deadline propagation** (section 1.5) so deeper layers stop retrying when the overall budget is
   gone.
3. **Use retry budgets** (section 4.3) so retries cannot exceed a fraction of normal traffic.
4. **Combine with circuit breakers** (topic 44), which stop calls entirely to a dependency that is clearly
   down.

### 4.3 Retry budgets

A per-request retry limit ("max 3 attempts") bounds the work per request but not the *total* extra load:
if every request fails, every request retries, and load triples. A **retry budget** bounds retries
relative to overall traffic.

```
 Retry budget: retries may be at most 10% of requests (per client, over a sliding window).

 Normal:     1000 req/s, 5 fail → 5 retries   (0.5%, within budget)        → all retried
 Outage:     1000 req/s, 1000 fail → budget allows 100 retries/s           → 900 fail fast
             load on dependency = 1100/s instead of 3000/s
```

Implementation is often a token bucket (topic 45): each successful request deposits a fraction of a
token (e.g., 0.1), each retry withdraws one token. During normal operation the bucket stays full and
every transient failure is retried; during a widespread outage the bucket empties and retries stop, so
the client's load on the dependency is capped at about 1.1× normal. Envoy, Finagle and gRPC's retry
throttling all implement versions of this idea.

Budgets turn a retry policy from "helpful in blips, harmful in outages" into "helpful in blips, nearly
neutral in outages", which is what you actually want.

---

## 5. Hedged requests: retrying for latency, not failure

### 5.1 The tail-latency problem

Topic 02 showed tail latency amplification under fan-out: if a request fans out to 100 servers and each
has a 1% chance of being slow, about 63% of requests hit at least one slow server. Often slowness is not
about the request; it's about the server (a GC pause, a noisy neighbor, a background compaction from
topic 19). Sending the same request to a *different* replica would likely be fast.

### 5.2 How hedging works

A **hedged request** (described in Dean & Barroso's "The Tail at Scale") sends the request to one replica,
and if no response arrives within some threshold (say the p95 latency), sends a second copy to another
replica. Whichever answers first wins; the other is cancelled or ignored.

```
 t=0     send to replica 1
 t=p95   no answer yet → send to replica 2
 t=...   replica 2 answers → use it, cancel replica 1
```

Because the hedge fires only for the slowest ~5% of requests, extra load is about 5%, but the p99 and
p99.9 can drop dramatically: the slow tail is now the *minimum* of two mostly independent latencies.

Variants:

- **Tied requests:** send to two replicas immediately, each tagged with the other's identity; when one
  starts executing it tells the other to cancel. Lower latency than waiting for p95, small duplicate work.
- **Backup requests only for reads,** since reads are naturally idempotent.

### 5.3 When hedging hurts

- **Non-idempotent operations:** a hedge is a deliberate duplicate. Only hedge reads or idempotent writes.
- **When the slowness is load-related:** if everything is slow because the whole cluster is overloaded,
  hedging adds load where there is none to spare. Hedging must be budgeted too (e.g., hedge at most 5% of
  requests) and should switch off when the dependency is broadly unhealthy.
- **Correlated slowness:** if both replicas share the slow disk or the same overloaded shard, the second
  copy doesn't help.

Hedging and retrying are the same mechanism (send again) with different triggers: retries react to
failures, hedges react to slowness. Both need budgets.

---

## 6. Metastable failures and retry storms

### 6.1 A failure that outlives its cause

Most people assume that when the trigger of an outage goes away (the bad deploy is rolled back, the
network blip ends), the system recovers. A **metastable failure** is one where it doesn't: the system has
moved into a bad state that sustains itself. The term comes from the 2021 HotOS paper "Metastable
Failures in Distributed Systems" (Bronson et al.).

A system has (at least) two stable states at the same load:

```
                      trigger (spike, blip, cache flush)
   ┌───────────────┐  ─────────────────────────────────►  ┌──────────────────────┐
   │ STABLE        │                                      │ METASTABLE (broken)  │
   │ efficient,    │  ◄─────────────────────────────────  │ sustaining feedback  │
   │ low latency   │   only by removing load drastically  │ loop keeps it down   │
   └───────────────┘                                      └──────────────────────┘
```

The bad state is held in place by a **sustaining effect**, usually some form of work amplification that
increases when the system is slow.

### 6.2 The retry storm, step by step

A database normally runs at 70% utilization, serving 7,000 requests/s of capacity 10,000/s. Clients use a
1 s timeout and 3 attempts, with no budget and no jitter.

```
 t0  normal: 7,000 req/s, latency 20 ms, utilization 70%

 t1  trigger: a 10-second network blip makes 50% of requests time out.
     those requests retry → offered load ≈ 7,000 + 3,500 retries + ... ≈ 12,000/s

 t2  blip ends. But the DB is now receiving 12,000/s > 10,000/s capacity.
     queues grow (topic 02: wait time explodes near 100% utilization)
     latency rises above 1 s → MORE requests time out → MORE retries

 t3  offered load ≈ 7,000 × 3 = 21,000/s. Nearly every request times out.
     The DB is 100% busy... doing work whose clients already gave up (no deadline check).
     Goodput (useful completed work) ≈ 0.

 t4  the trigger is long gone, but the system stays down. Rolling back the bad change does nothing.
```

Throughput is high, **goodput** is near zero. This is the signature of a metastable failure: the server
is busy doing work that doesn't help anyone.

```
 goodput
   ▲
   │        ┌──── stable region
   │      ╱  ╲
   │    ╱      ╲      ← past this point, more load → less useful work
   │  ╱          ╲___
   │╱                ╲________________  (congestion collapse: busy, ~0 goodput)
   └──────────────────────────────────────► offered load
```

### 6.3 Other sustaining effects

Retries are the most common, but the pattern recurs:

- **Cache loss:** a cache is flushed or a cold cache is deployed (topic 12). Every request goes to the
  database, which slows, so requests time out before they can repopulate the cache, so the cache stays
  cold.
- **Queue backlog:** a queue builds up during a blip; consumers now process old messages whose producers
  have already timed out and resent them, so the backlog never shrinks (topic 36).
- **GC / memory pressure:** more in-flight requests → more memory → longer GC pauses → slower → more
  in-flight requests.
- **Connection storms:** timeouts close connections; reconnecting costs TLS handshakes (topic 05),
  which costs CPU, which makes the server slower, which causes more timeouts.
- **Failover loops:** an overloaded node fails health checks (topic 09), its load moves to the others,
  which then fail health checks too.

### 6.4 Getting out and staying out

Recovering usually requires **removing load drastically**, well below the level the system handled
before: blocking traffic at the edge, shedding load (topic 44), letting caches warm, then ramping up
slowly. Prevention is cheaper:

| Defense | What it breaks |
|---|---|
| Retry budgets (4.3) | Caps retry amplification at ~1.1× |
| Retry at one layer | Removes multiplicative amplification |
| Jittered exponential backoff (3.3) | Spreads and slows retries |
| Deadline propagation (1.5) | Server drops work whose client has gone → goodput stays high |
| Load shedding / admission control (topic 44) | Keeps the server in its efficient region |
| Circuit breakers (topic 44) | Stops callers hammering a dead dependency |
| LIFO queueing / bounded queues (topic 44) | Serves fresh requests that can still succeed |
| Request coalescing (topic 12) | Prevents cold-cache stampedes |
| Test with overload | Find the tipping point before production does |

The deeper lesson: **systems are usually tuned for efficiency in the normal state, and the same
efficiency mechanisms (retries, caches) become the sustaining effects in the bad state.**

---

## 7. A reasonable default policy

Not a rule, but a sane starting point for a service calling another service:

- Explicit connect timeout (short) and request timeout (around the dependency's p99.9 plus headroom),
  both well within the caller's own deadline.
- Propagate a deadline; refuse work whose deadline has expired.
- Retry only transient errors, only idempotent operations (or with idempotency keys).
- At most 2–3 total attempts, exponential backoff with full jitter, capped, honoring `Retry-After`.
- A retry budget of ~10% per client.
- Retry at one layer; other layers fail fast.
- Hedging only for idempotent reads with tail-latency problems, and with a budget.
- Metrics on retries and timeouts (topic 46): a rising retry rate is an early warning.

---

## 8. How it connects

- **Topic 01, 02:** percentiles drive timeout choice; queueing theory explains why load near capacity
  turns retries into collapse; Little's Law explains why missing timeouts exhaust pools; tail
  amplification under fan-out motivates hedging.
- **Topic 07:** idempotent HTTP methods and precise status codes tell clients what is safe to retry.
- **Topic 09, 10:** load balancers and meshes do their own retries and health checks; they're one of the
  hidden retry layers. Idle-timeout mismatch with LBs is a classic bug.
- **Topic 12:** thundering herd and jittered TTLs are the cache form of the same synchronization problem.
- **Topic 26:** the root cause: you cannot distinguish slow from dead, so timeouts produce unknown outcomes.
- **Topic 27:** pass remaining durations rather than absolute times across hosts with skewed clocks.
- **Topic 38:** idempotency and idempotency keys make retries safe; redelivery in queues is a retry.
- **Topic 44:** circuit breakers, bulkheads and load shedding are the next layer of defense.
- **Topic 45:** token buckets implement retry budgets; 429 responses are a retry signal.
- **Topic 46:** you need retry rates, timeout rates and goodput metrics to see storms forming.

---

## Common misconceptions

- "Retries make a system more reliable." → They make it more reliable against *independent, transient*
  failures. Against overload they make it less reliable, possibly permanently (metastable failure).
- "A timeout means the request failed." → It means the outcome is unknown. The server may have
  completed it.
- "The library has a sensible default timeout." → Many defaults are infinite or minutes long. Set
  timeouts explicitly.
- "Exponential backoff is enough." → Without jitter, clients that failed together retry together.
- "Max 3 attempts bounds the extra load." → Per request, yes. In aggregate, 3 attempts can triple load
  during an outage. Only a retry budget bounds total extra load.
- "Each layer should retry to be safe." → Layered retries multiply: r^k. Retry at one layer.
- "If we fix the trigger, the system will recover." → Not in a metastable failure; you must remove load
  to break the feedback loop.
- "Hedged requests double the load." → Hedging after the p95 adds roughly 5% load; but it must still be
  budgeted, because during overload everything is past p95.
- "A timeout stops work on the server." → Only the client stops waiting. Without deadline propagation the
  server keeps working.

## Further reading (supplements)

- Marc Brooker, "Exponential Backoff And Jitter", AWS Architecture Blog (2015).
- Amazon Builders' Library: "Timeouts, retries, and backoff with jitter" (Marc Brooker).
- Jeffrey Dean & Luiz André Barroso, "The Tail at Scale", Communications of the ACM (2013).
- Nathan Bronson et al., "Metastable Failures in Distributed Systems", HotOS 2021; and Huang et al.,
  "Metastable Failures in the Wild", OSDI 2022.
- *Site Reliability Engineering* (Google), chapters 21 "Handling Overload" and 22 "Addressing Cascading
  Failures".
- Michael Nygard, *Release It!*, chapters on stability patterns (timeouts, integration points).
- gRPC documentation: "Deadlines" and "Retry" guides.

---

## Questions

1. A service calls a dependency with no timeout. The dependency starts hanging (not erroring). Walk
   through what happens to the calling service over the next minute, using Little's Law.
2. Your call to a payment service times out. What do you actually know about the payment? What must be
   true before you can safely retry?
3. A dependency has p50 = 10 ms, p99 = 80 ms, p99.9 = 300 ms. A teammate sets the timeout to 50 ms "to keep
   things fast" and adds 3 retries. What happens?
4. Why is a connect timeout usually much shorter than a request timeout?
5. A request passes through A → B → C, each with its own 2 s timeout. The user-facing SLO is 2 s. What goes
   wrong, and how does deadline propagation fix it?
6. Why does exponential backoff without jitter still produce load spikes? Give a scenario.
7. Four layers each make up to 3 attempts. The bottom database is failing. How many database calls does one
   user action produce, and why is this especially dangerous?
8. Explain how a retry budget implemented with a token bucket behaves during normal operation vs during a
   full outage of the dependency.
9. Which of these should be retried: 503 with `Retry-After: 5`, 400, connection refused on a POST, read
   timeout on a POST without idempotency key, gRPC UNAVAILABLE on a GET?
10. When do hedged requests help, and when do they make things worse?
11. A bad deploy causes 5 minutes of errors. It's rolled back, yet the service stays at ~100% CPU with
    nearly all requests timing out for the next hour. What is likely happening, and why didn't the
    rollback fix it?
12. In a metastable failure, why is "throughput" a misleading metric? What should you look at instead?
13. Why does deadline propagation improve goodput during overload?
14. A mobile app retries every failed request immediately up to 5 times, and the API gateway also retries
    upstream calls 3 times. Which change would you make first and why?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Each request to the hung dependency holds a thread/connection indefinitely. At arrival rate λ, the
number of stuck requests grows by λ per second without bound (Little's Law: concurrency = λ × time, and
time is unbounded). At 100 rps, a 200-thread pool is exhausted in ~2 s. After that the caller cannot serve
*any* request, including those that don't touch the dependency: health checks fail, its own callers time
out or pile up. The failure cascades upward. A timeout would have capped stuck requests at λ × timeout.

**2.** You know only that no response arrived in time. The request may never have arrived, may have been
processed and the response lost, or may still be processing. Before retrying you need the operation to be
idempotent, typically via an idempotency key that the payment service uses to deduplicate attempts and
return the original result. Otherwise a retry may double-charge.

**3.** 50 ms is below p99, so roughly 2–5% of requests that would have succeeded are cut off. Each is
retried, adding load; retries also hit the same latency distribution, so some fail again. Load on the
dependency rises a few percent in normal times, and if the dependency slows even slightly (p50 drifts up),
a large share of requests exceed 50 ms and get retried up to 3 more times: load can multiply toward 4×,
which slows it further. It's a recipe for a retry storm. A timeout near p99.9 plus headroom is safer.

**4.** Establishing a connection is a small fixed amount of work (handshake round trips) that doesn't
depend on the request; within a datacenter it takes around a millisecond. If it doesn't complete quickly,
the host is likely down or unreachable, and failing fast lets you try another instance. Also, a failed
connect is always safe to retry because the request was never sent. A request timeout must allow for the
actual work, which varies by operation.

**5.** A doesn't know how long B and C will take; if C takes 1.9 s, B returns at ~2 s, and A has already
timed out and returned an error to the user. B and C did wasted work. If B retries C, B may run for 4 s on a
request nobody is waiting for. With deadline propagation, the edge sets "deadline = now + 2 s"; each hop
passes the remaining budget; C sees how much time is left and refuses work that can't finish; B won't retry
if insufficient budget remains. No wasted work, and the user's limit is respected end to end.

**6.** Clients that failed at the same moment compute identical waits and retry at the same moment. E.g., a
database restarts and 10,000 connected clients all fail at t=0; they all retry at 100 ms, 300 ms, 700 ms...
producing synchronized spikes of 10,000 requests that the just-recovered database can't absorb, so it fails
again. Jitter randomizes waits and spreads the same retries over time.

**7.** 3⁴ = 81. It's dangerous because amplification is greatest at the deepest layer, which is typically
the shared, hardest-to-scale component, and it's applied exactly when that component is already failing.
A database that might have handled normal load while degraded gets 81× and collapses fully. Latency also
compounds as each layer waits on nested timeouts.

**8.** Each successful request adds a fraction of a token (e.g., 0.1); each retry costs one token. Normally
almost all requests succeed, the bucket stays full, and every occasional failure is retried. In a full
outage no requests succeed, so no tokens are added; the bucket drains after a few retries and further
failures are returned immediately without retry. Load on the dependency stays near 1× (plus ~10%) instead
of 3×, giving it room to recover.

**9.** 503 with Retry-After: yes, after at least 5 s (with jitter). 400: no, the request is invalid.
Connection refused on POST: yes, the request never reached the server so it can't have been processed.
Read timeout on POST without idempotency key: no (not safely), outcome unknown and could double-apply.
gRPC UNAVAILABLE on GET: yes, it's transient by definition and GET is idempotent.

**10.** They help when tail latency is caused by per-server, uncorrelated slowness (GC pauses, background
compactions, noisy neighbors), the operation is idempotent (usually reads), and there's spare capacity: a
hedge after p95 costs ~5% extra load but cuts p99 sharply. They hurt when slowness is caused by overall
overload (every request exceeds p95, so you double load), when the operation isn't idempotent, or when
replicas share the cause of slowness. Hedging should be budgeted and disabled when the dependency is
broadly unhealthy.

**11.** A metastable failure, most likely a retry storm (or cold-cache feedback loop). During the 5 minutes
of errors, retries pushed offered load above capacity; latency rose past client timeouts; timed-out
requests were retried, keeping load above capacity. The server spends its CPU on requests whose clients
have already given up. The rollback removed the trigger, but the sustaining loop (retries ↔ latency) is
independent of the trigger. Recovery needs a drastic load reduction (shed load, block at the edge, disable
retries), then a gradual ramp.

**12.** Because the server can be 100% busy completing requests whose clients already timed out: high
throughput, near-zero useful output. Look at goodput (successful responses delivered to a waiting client
within deadline), success rate, and the ratio of retries to first attempts.

**13.** Under overload, queues build and many requests sit until their client has given up. Without
deadlines the server still executes them, spending capacity on wasted work and delaying requests that
could still succeed. With deadlines, expired requests are discarded cheaply before executing, so capacity
goes to requests whose clients are still waiting: useful output stays high instead of collapsing.

**14.** Fix the app's immediate retries first: add exponential backoff with jitter, reduce attempts, and add
a budget. It sits at the top, so it multiplies everything below it (5 × 3 = 15× per action), and immediate,
unjittered retries from millions of devices synchronize into storms. Then decide on a single retry layer
(likely the gateway, closer to the dependency) and make the other fail fast.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Why does every remote call need a timeout?
**A:** Without one, a hung dependency pins caller resources forever; stuck requests grow without bound until pools exhaust and the failure cascades upward.

**Q:** What does Little's Law say about timeouts?
**A:** Concurrency = arrival rate × time. A timeout caps time, so it caps how many requests a slow dependency can hold.

**Q:** What does a timeout tell you about the outcome?
**A:** Nothing definite: the request may or may not have been processed. It's an unknown outcome.

**Q:** Connect timeout vs request timeout?
**A:** Connect bounds establishing the connection (short, catches dead hosts, always safe to retry); request bounds one full attempt.

**Q:** How do you choose a timeout value?
**A:** From the dependency's latency distribution (around p99.9 plus headroom), checked against the caller's own deadline.

**Q:** What is deadline propagation?
**A:** Set one end-to-end deadline at the edge and pass the remaining budget to every hop, so downstream work and retries stop when the user has given up.

**Q:** Precondition for retrying after a timeout?
**A:** The operation must be idempotent, e.g., via an idempotency key the server deduplicates on.

**Q:** Which errors should be retried?
**A:** Transient ones: connect failures, 503/429 (honoring Retry-After), gRPC UNAVAILABLE. Not 4xx request errors.

**Q:** Exponential backoff formula?
**A:** wait = min(cap, base × 2^attempt), plus a maximum attempt count or deadline.

**Q:** Why add jitter?
**A:** Clients that failed together otherwise retry together, producing synchronized spikes (thundering herd).

**Q:** Full jitter?
**A:** wait = random(0, min(cap, base × 2^n)).

**Q:** Retry amplification with k layers, r attempts each?
**A:** r^k requests at the bottom layer, e.g., 3⁴ = 81.

**Q:** Common rule to prevent amplification?
**A:** Retry at only one layer; others fail fast and propagate the error.

**Q:** What is a retry budget?
**A:** A cap on retries as a fraction of total requests (e.g., 10%), often a token bucket refilled by successes.

**Q:** Why isn't "max 3 attempts" enough?
**A:** It bounds per-request work, but in an outage every request retries, so total load can triple.

**Q:** What is a hedged request?
**A:** After a delay (e.g., p95) with no response, send a duplicate to another replica and use whichever answers first.

**Q:** Cost and benefit of hedging after p95?
**A:** About 5% extra load for a large drop in p99/p99.9.

**Q:** When must you not hedge?
**A:** Non-idempotent operations, or when slowness is due to overall overload.

**Q:** What is a metastable failure?
**A:** A bad state sustained by a feedback loop (e.g., retries) that persists after the trigger is removed.

**Q:** Throughput vs goodput?
**A:** Throughput is work done; goodput is useful work delivered to clients still waiting. In collapse, throughput is high and goodput near zero.

**Q:** Name three sustaining effects for metastable failures.
**A:** Retry storms, cold caches that can't refill, queue backlogs of stale work (also GC pressure, reconnect storms).

**Q:** How do you recover from a metastable failure?
**A:** Remove load drastically (shed, block at edge, disable retries), let the system stabilize, then ramp up gradually.

**Q:** Classic idle-timeout bug?
**A:** Client keeps pooled connections longer than the load balancer's idle timeout, so it sends on dead connections.

**Q:** What should `Retry-After` do to a client's backoff?
**A:** The client should wait at least that long; the server's hint overrides the generic formula.
