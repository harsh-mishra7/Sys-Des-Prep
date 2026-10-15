# 45 — Rate Limiting

> Part 8 — Reliability & Operations · Prerequisites: topics 09, 10, 11, 43, 44

Topic 44 showed a server protecting itself by shedding load when it is overloaded, regardless of who
sent it. **Rate limiting** is a related but different idea: it caps how much work each *client* (user,
API key, IP address, tenant) may send per unit of time, whether or not the server is currently
struggling. It's how a public API stops one script from consuming the capacity meant for everyone, how a
login endpoint slows down password guessing, and how a service keeps its bill from a paid upstream
API under control. The concept is simple; the interesting parts are the algorithms (each counts
"requests per time" in a subtly different way, with different burst behavior and memory cost) and the
distributed version, where many servers must agree on one client's count.

By the end of this topic you should be able to:
1. Explain the three reasons to rate limit (protection, fairness, cost) and choose a limiting key.
2. Explain the mechanism of token bucket, leaky bucket, fixed window, sliding window log and sliding
   window counter, and compare their burst behavior, accuracy and memory cost.
3. Explain how to rate limit across many servers: shared counters, atomicity and race conditions, and
   the trade-off between local and global accuracy.
4. Decide between fail-open and fail-closed when the rate limiter itself fails.
5. Describe how limits are communicated to clients (429, headers) and where in the stack to enforce them.

---

## 1. Why rate limit

### 1.1 Three reasons

**Protection.** Without limits, any single client can send unbounded traffic, whether by malice
(scraping, credential stuffing, denial of service) or by accident (a retry loop with no backoff, a
misconfigured cron job running every second instead of every hour). Rate limits put a ceiling on what any
one source can do, protecting capacity for everyone else. Some endpoints need limits for security reasons
specifically: login, password reset, OTP verification and signup are rate limited to make brute-force
guessing impractical.

**Fairness.** Even well-behaved clients compete for a shared resource. A multi-tenant service (topic 48's
noisy-neighbor problem) wants each tenant to get its fair share. Rate limits per tenant, often tiered by
plan ("free: 100 requests/min, pro: 10,000/min"), make the share explicit.

**Cost.** Some operations cost real money: a call to a paid third-party API, an SMS, an expensive ML
inference, egress bandwidth. Limits cap spend per user and protect against runaway bills. Equally, a
service calling a paid or rate-limited upstream rate limits *itself* (client-side limiting) so it stays
within the upstream's quota rather than being throttled or billed.

### 1.2 Rate limiting vs load shedding

| | Rate limiting | Load shedding (topic 44) |
|---|---|---|
| Question asked | "Has *this client* exceeded *its* allowance?" | "Is *the server* over *its* capacity?" |
| Depends on server health? | No, applies even when idle | Yes, kicks in under overload |
| Typical response | 429 Too Many Requests | 503 Service Unavailable |
| Purpose | Fairness, abuse prevention, cost, contracts | Survival |

They complement each other. Rate limits keep any single client from causing overload; load shedding
handles overload anyway (many clients each within limits can still add up to too much).

### 1.3 What to limit by: the key

A rate limit is "N requests per time window *per key*". Choosing the key is a real design decision:

| Key | Good for | Weakness |
|---|---|---|
| User ID / account | Authenticated APIs, fairness between users | Unauthenticated endpoints have no user |
| API key / client ID | Public APIs, partner integrations, billing tiers | One key shared across a customer's whole fleet |
| IP address | Unauthenticated endpoints, login, signup | NAT and corporate proxies put thousands of users behind one IP; attackers rotate IPs cheaply; IPv6 gives each attacker huge ranges (limit by /64 prefix) |
| Tenant / organization | Multi-tenant SaaS fairness | Large tenants need custom limits |
| Endpoint (+ key) | Expensive operations get tighter limits than cheap ones | More limits to manage |
| Global (no key) | Protecting a fragile downstream, a paid upstream quota | Not fair: one client can use it all |

Real systems layer several: e.g., per IP for anonymous traffic, per user per endpoint for authenticated
traffic, plus a global cap protecting a downstream. Limits can also be **weighted**: instead of counting
requests, count cost units (an expensive search costs 10 units, a simple read costs 1). GraphQL APIs (topic
07) often limit by computed query cost because one query can do the work of hundreds.

---

## 2. The algorithms

Throughout, assume the limit is "10 requests per minute" unless stated otherwise.

### 2.1 Token bucket

The most widely used algorithm (AWS API Gateway, Stripe, many proxies and libraries).

**Mechanism.** Each key has a bucket that holds up to **B** tokens (the burst capacity). Tokens are added
at a steady rate **r** (the refill rate) up to the cap. Each request takes one token (or *n* for weighted
requests). If a token is available, the request is allowed and the token is removed; if not, it's
rejected (or delayed).

```
            refill r tokens/sec
                    │
                    ▼
            ┌───────────────┐
            │ ● ● ● ● ●     │  capacity B (max tokens)
            └───────┬───────┘
                    │ each request takes 1 token
                    ▼
          token available? ── yes ──► allow
                    │
                    no ──► reject (429)
```

**Implementation: no timer needed.** You don't need a background job adding tokens. Store two values per
key, `tokens` and `last_refill_time`, and compute the refill lazily on each request:

```
 on request(key):
   now     = current_time()
   elapsed = now - last_refill_time
   tokens  = min(B, tokens + elapsed × r)
   last_refill_time = now
   if tokens >= 1:
       tokens -= 1
       allow
   else:
       reject (retry after (1 - tokens) / r seconds)
```

**Behavior.** Two parameters, two meanings:

- **r** is the sustained long-term rate.
- **B** is how much burst is tolerated. A client idle for a while accumulates up to B tokens and can then
  send B requests instantly.

Example: r = 1 token/s, B = 10. A client that's been idle sends 10 requests in one second (all allowed,
bucket empty), then is held to 1 per second. Over any interval of length T, at most B + r × T requests
pass.

**Pros:** allows natural bursts (real clients are bursty: a page load fires 8 API calls at once), O(1)
memory per key (two numbers), simple, and gives an exact "retry after" time. **Cons:** two parameters to
tune; bursts of B can still hit the backend all at once.

### 2.2 Leaky bucket

**Mechanism.** Two variants share the name:

1. **Leaky bucket as a queue (traffic shaper):** requests enter a FIFO queue of size B. The queue
   "leaks" (requests are processed) at a constant rate r. If the queue is full, new requests are
   rejected.

```
 requests (bursty) ──► ┌─────────┐
                       │ ▓▓▓▓    │ queue capacity B
                       │ ▓▓▓▓    │ full → reject
                       └────┬────┘
                            │ drains at constant rate r
                            ▼
                     smooth output to backend
```

2. **Leaky bucket as a meter:** equivalent to a token bucket viewed upside down: a counter rises with each
   request and drains at rate r; requests that would overflow are rejected. This version behaves the same
   as a token bucket.

The queue variant's distinctive property: **output is perfectly smooth**. The backend never sees a burst,
only a constant rate. The cost: requests are **delayed** while queued (adding latency), and a burst fills
the queue so later requests get rejected even if the burst was short. Good for shaping traffic to a
fragile downstream that can only accept a fixed rate (e.g., sending outbound email or calling a partner
API with a strict per-second limit). NGINX's `limit_req` is a leaky bucket; its `burst` parameter is the
queue size and `nodelay` makes it behave more like a token bucket.

| | Token bucket | Leaky bucket (queue) |
|---|---|---|
| Bursts | Allowed up to B, passed through immediately | Absorbed into queue, released smoothly |
| Output rate | Can spike | Constant |
| Added latency | None (allow or reject) | Queueing delay |
| Best for | API rate limits for clients | Shaping traffic toward a fragile consumer |

### 2.3 Fixed window counter

**Mechanism.** Divide time into fixed windows (e.g., each calendar minute). Keep one counter per key per
window. Increment on each request; reject when the counter exceeds the limit. The window key is derived
from time, e.g. `user:42:2026-10-15T10:31`, and expires after the window.

```
 window:   10:00 ─────────── 10:01 ─────────── 10:02
 counter:  [ 7 ]             [ 10 ] (full)     [ 0 ]
```

**Pros:** trivially simple, O(1) memory, maps directly onto a Redis `INCR` + `EXPIRE`.

**Cons: the boundary burst.** A client can send the full limit at the end of one window and the full limit
again at the start of the next:

```
 limit 10/min
            10:00:00                 10:00:59 │ 10:01:00                 10:01:59
 window 1:  ...................... 10 reqs ──┤
 window 2:                                   ├── 10 reqs ......................
                                   └─────────┘
                                   20 requests within ~2 seconds
```

So the real maximum over any 60-second span is **2× the limit**. Also, all clients' counters reset at the
same instant, so clients that were blocked all retry at the top of the minute, a synchronized spike.

### 2.4 Sliding window log

**Mechanism.** Store the timestamp of every accepted request per key (e.g., in a Redis sorted set). On each
request: delete timestamps older than `now - window`, count what's left, allow if count < limit (and add
the new timestamp).

```
 window = 60s, limit = 3, now = 10:01:30

 log: [10:00:20, 10:00:45, 10:01:10, 10:01:25]
       ── older than 10:00:30, delete ──
 log: [10:00:45, 10:01:10, 10:01:25]  count = 3 → reject
```

**Pros:** exact. Over every possible window of the given length, the limit holds. No boundary burst.

**Cons:** memory is O(limit) per key (one timestamp per request in the window). For a limit of 10,000
requests per hour across a million keys, that's billions of timestamps. Also each request costs more work
(range deletion + count). A subtle choice: whether *rejected* requests are logged. If they are, a client
hammering the endpoint never gets back under the limit; usually only accepted requests are recorded.

### 2.5 Sliding window counter

The practical compromise: nearly the accuracy of the log with the memory of fixed windows.

**Mechanism.** Keep counters for the current fixed window and the previous one. Estimate the count over the
sliding window by assuming the previous window's requests were spread evenly, and weighting by how much of
the previous window still overlaps the sliding window.

```
 limit = 100/min. Now is 10:01:15 (25% into the current window).

 previous window (10:00–10:01): 80 requests
 current window  (10:01–10:02): 30 requests so far

 the sliding window [10:00:15 – 10:01:15] overlaps 75% of the previous window

 estimate = 80 × 0.75 + 30 = 90  → < 100 → allow
```

**Pros:** O(1) memory (two counters), smooths out the boundary burst. **Cons:** an approximation; it
assumes uniform distribution within the previous window, so it can be slightly too lenient or too strict
when traffic was bunched. In practice the error is small (Cloudflare reported its sliding-window approach
being wrong for a tiny fraction of requests), which is why it's common for large-scale limiting.

### 2.6 Comparison

| Algorithm | Memory per key | Burst behavior | Accuracy | Typical use |
|---|---|---|---|---|
| Token bucket | O(1): tokens + timestamp | Allows bursts up to B, then rate r | Exact for its definition | General API limits |
| Leaky bucket (queue) | O(B) queue | Smooths bursts into constant output; adds delay | Exact | Shaping traffic to a downstream |
| Fixed window | O(1): one counter | Up to 2× limit at window boundary | Coarse | Simple quotas, daily/monthly caps |
| Sliding window log | O(limit): timestamps | None beyond limit | Exact | Low-volume, strict limits (login attempts) |
| Sliding window counter | O(1): two counters | Small error near boundaries | Approximate, good | High-scale per-key limits |

A note on long windows: daily or monthly **quotas** ("1 million calls per month on your plan") are
typically fixed windows aligned to billing periods, and that's fine: the boundary burst doesn't matter for
billing. Short-window limits protecting capacity care more about bursts.

### 2.7 Reject, delay, or degrade

When a request exceeds the limit, options include:

- **Reject** with 429 (most common for APIs).
- **Delay/queue** until allowed (leaky bucket behavior), acceptable for background or internal clients.
- **Degrade:** serve a cheaper response (cached result, lower resolution).
- **Shadow mode:** log "would have been limited" without enforcing; essential when rolling out a new limit
  to see who it would affect before turning it on (topic 47 safe change).

---

## 3. Communicating limits to clients

A good rate limiter tells clients what's happening so well-behaved clients can adapt:

```
 HTTP/1.1 429 Too Many Requests
 Retry-After: 12
 RateLimit-Limit: 100
 RateLimit-Remaining: 0
 RateLimit-Reset: 12
```

- **429 Too Many Requests** means "you specifically are over your limit" (unlike 503, "the server is
  overloaded"; topic 44).
- **Retry-After** tells clients exactly when to retry; topic 43's backoff should honor it. Token buckets
  can compute this exactly.
- **RateLimit-*** headers (an IETF draft standard; many APIs use `X-RateLimit-*` variants) let clients see
  remaining budget and pace themselves before hitting the wall.

Good clients use these to self-throttle. Clients that ignore 429 and retry immediately just burn more
of your capacity on rejections, so rejections must be cheap (topic 44).

---

## 4. Where to enforce

```
 client ──► CDN/edge ──► load balancer / API gateway ──► service ──► downstream
   (a)         (b)               (c)                        (d)          (e)
```

- **(a) Client-side:** a client limits its own outbound rate to respect an upstream's quota. Cooperative,
  not a defense.
- **(b) Edge/CDN (topic 14):** cheapest place to drop abusive traffic (DDoS, scraping by IP), before it
  reaches your infrastructure. Coarse keys (IP, path).
- **(c) Gateway / reverse proxy / mesh (topic 10):** the usual place for API-key and user limits:
  centralized, consistent across services, one place to configure tiers.
- **(d) In the service:** for limits that need business context the gateway doesn't have (e.g., "max 5
  password reset emails per account per hour", cost-weighted limits).
- **(e) Protecting a downstream:** a global limiter in front of a fragile database or third-party API.

Defense in depth is normal: coarse limits at the edge, per-key limits at the gateway, fine-grained
business limits in services.

---

## 5. Distributed rate limiting

### 5.1 The problem

A single server can keep counters in memory. But a real API runs on many servers behind a load balancer
(topic 09), and a client's requests spread across all of them. If each of 10 servers enforces "100/min"
locally, the client actually gets up to 1,000/min. The limit must be enforced on the *combined* count.

### 5.2 Approach 1: a shared counter store

Keep counters in a central fast store, usually Redis, which every server consults.

```
 server 1 ─┐
 server 2 ─┼──► Redis: ratelimit:{key} ──► allow / reject
 server 3 ─┘
```

Fixed window in Redis:

```
 count = INCR ratelimit:user42:202610151031
 if count == 1: EXPIRE ratelimit:user42:202610151031 60
 if count > 100: reject
```

### 5.3 Race conditions

The naive way to implement a check is read-then-write:

```
 server A: GET count → 99     server B: GET count → 99
 server A: 99 < 100, allow    server B: 99 < 100, allow
 server A: SET count 100      server B: SET count 100
 → two requests allowed, counter says 100 (one lost update; limit exceeded)
```

This is the lost-update problem from topic 22, and the fix is the same idea: make the
check-and-update **atomic**.

- **Atomic increment first, check after:** `INCR` returns the new value atomically; compare it to the
  limit. Two concurrent requests get 100 and 101; the second is rejected. (Note the counter may exceed
  the limit, counting rejected requests, which is usually fine.)
- **Lua scripts / server-side functions:** for token bucket or sliding window, which read and write
  several fields, run the whole read-compute-write in a Redis Lua script, which executes atomically.
- **Sorted-set log:** `ZREMRANGEBYSCORE` + `ZCARD` + `ZADD` inside a `MULTI`/script.
- A second subtle race: in the fixed window snippet above, if the server crashes between `INCR` and
  `EXPIRE`, the key never expires and the client is blocked forever. Set the TTL atomically (in the same
  script, or with `SET ... NX EX`).

Also watch out for **clock differences** (topic 27): if each app server computes window boundaries and
token refills from its own clock, skewed servers disagree. Using the store's clock (e.g., Redis `TIME`
inside the script) avoids that.

### 5.4 The cost of global accuracy

A shared store makes the limit accurate, but:

- **Latency:** every request now includes a network round trip to Redis (~0.5–1 ms in a datacenter; far
  more cross-region).
- **Load:** the limiter store sees one operation per API request; for a very high-traffic API it becomes a
  hot, critical dependency. It's sharded by key (topic 25/13) to scale, but a single very hot key (one huge
  client) still lands on one shard.
- **Availability:** if the store is down, what happens? (Section 6.)
- **Multi-region:** a single global counter across regions costs cross-region latency on every request
  (topic 48). Usually each region enforces its own share, or counts are synced asynchronously.

### 5.5 Approach 2: local limiting with approximation

Trade accuracy for speed by keeping counters locally and accepting some error:

- **Divide the limit:** with N servers, each enforces limit/N locally. Works if the load balancer spreads
  each client evenly. Fails when N changes (autoscaling) or traffic is uneven: a client hitting only 2 of
  10 servers gets only 20% of its allowance.
- **Local counting with periodic sync:** each server counts locally and, every few hundred milliseconds,
  pushes its deltas to a central store and pulls the global total. Between syncs a client can exceed the
  limit by roughly (rate × sync interval × servers), a bounded overshoot.
- **Local token buckets refilled from a central allocator:** servers lease batches of tokens from the
  central store ("give me 50 tokens for user42") and spend them locally, returning to the store only when
  the batch is used up. Far fewer store round trips; slight unfairness if one server holds tokens another
  needs.
- **Sticky routing:** route each key to the same limiter instance (consistent hashing, topic 13) so its
  counter is local but authoritative.

### 5.6 Local vs global accuracy: choosing

```
 accuracy ◄──────────────────────────────────────────────► speed / availability
 every request checks          periodic sync /          purely local,
 shared store atomically       token leasing            limit/N per server
 (exact, +1 RTT, SPOF risk)    (bounded overshoot)      (no coordination, inaccurate)
```

The right point depends on the purpose:

- **Security limits** (login attempts, OTP guesses): accuracy matters, volume is low → shared store with
  exact algorithm.
- **Fairness / capacity limits for high-volume APIs:** a few percent overshoot is harmless → local with sync.
- **Cost limits against a paid upstream with a hard quota:** must not exceed → central or conservative
  (enforce slightly under the quota).

This is the same consistency-vs-latency trade-off as in topics 28–29, applied to a counter.

---

## 6. Fail-open vs fail-closed

The rate limiter is a dependency. When it's unavailable (Redis down, timeout talking to the limiter
service), each request has to be either allowed (**fail-open**) or rejected (**fail-closed**).

| | Fail-open | Fail-closed |
|---|---|---|
| Behavior when limiter is down | Allow all requests | Reject all requests |
| Risk | Unprotected: abuse or overload gets through | Limiter outage becomes a full outage of the service |
| Choose when | The limit is for fairness/capacity, and the backend has other protection (load shedding) | The limit is a security or cost control where unlimited access is worse than downtime (e.g., OTP verification, a paid SMS endpoint) |

Most API rate limiters **fail open**: a broken limiter shouldn't take down the product, since limiting
is a secondary protection layered on top of load shedding (topic 44). But "fail-open" should still be
safe:

- Use a **short timeout** on the limiter call (a few ms) so a slow limiter doesn't add latency to every
  request (topic 43), plus a circuit breaker so the service stops calling it while it's down.
- Fall back to a **local, approximate limiter** (e.g., an in-memory token bucket at limit/N) rather than
  "no limit at all". This degrades accuracy, not protection.
- **Alert** loudly; a fail-open limiter is an unprotected endpoint.

For security-critical endpoints, fail closed, or fail to a strict local limit, and accept that a limiter
outage blocks those operations.

---

## 7. How it connects

- **Topic 07:** 429 and `Retry-After` semantics; GraphQL query-cost limits.
- **Topic 09, 10:** gateways, proxies and meshes are the usual enforcement points; load balancers spread a
  client across servers, which is what makes limiting distributed.
- **Topic 11, 25, 13:** limiter state lives in a cache-like store (Redis), sharded by key; hot keys are a
  problem.
- **Topic 14:** the edge is the cheapest place to drop abusive traffic.
- **Topic 22:** read-check-write races are lost updates; fix with atomic operations.
- **Topic 27:** clock skew between servers breaks window calculations.
- **Topic 28, 29:** local vs global accuracy is a consistency/latency trade-off.
- **Topic 43:** retry budgets are token buckets; clients should honor `Retry-After`.
- **Topic 44:** rate limiting is per-client admission control; load shedding is the server-wide backstop.
- **Topic 47:** roll out new limits in shadow mode first.
- **Topic 48:** per-tenant limits and noisy neighbors; multi-region limiting.
- **Topic 50:** probabilistic structures (count-min sketch) can approximate counts for huge key spaces.

---

## Common misconceptions

- "Rate limiting protects the server from overload." → Partly. Many clients each under their limit can
  still overload it. You also need load shedding.
- "A 100/min fixed window means at most 100 requests in any minute." → Up to 200 in a short span around a
  window boundary.
- "Token bucket and leaky bucket are the same." → The meter variant is equivalent; the queue variant
  smooths output and adds delay, while a token bucket passes bursts through immediately.
- "Just check the counter, then increment it." → Read-then-write races allow overshoot; the
  check-and-update must be atomic.
- "Per-server limits are fine if I divide by N." → Only if traffic is spread evenly and N is stable;
  autoscaling and sticky traffic break it.
- "Limit by IP is a good universal default." → NAT and proxies put many users behind one IP; attackers
  rotate IPs. Prefer authenticated keys when available.
- "The limiter should fail closed to be safe." → For most limits, failing closed turns a limiter outage
  into a total outage; fail open (to a local approximate limit) unless it's a security/cost control.
- "Sliding window counter is exact." → It's an approximation assuming uniform traffic in the previous
  window; the error is small but nonzero.

## Further reading (supplements)

- Cloudflare blog, "How we built rate limiting capable of scaling to millions of domains" (sliding window
  counter).
- Stripe engineering blog, "Scaling your API with rate limiters" (token buckets, concurrent request
  limiters, load shedders).
- Redis documentation: rate limiting patterns with `INCR` and Lua scripts.
- IETF draft "RateLimit header fields for HTTP" (httpapi working group).
- Google Cloud architecture docs, "Rate-limiting strategies and techniques".
- NGINX documentation for `limit_req` (leaky bucket with burst / nodelay).
- *Site Reliability Engineering* (Google), chapter 21 "Handling Overload" (per-customer limits, client-side
  throttling).

---

## Questions

1. Give an example each of rate limiting for protection, fairness and cost. Which key would you limit by in
   each case?
2. How does rate limiting differ from load shedding? Why do you need both?
3. A token bucket has r = 5 tokens/s and B = 20. A client has been idle for a minute and then sends 50
   requests in one second. How many are allowed, and what happens over the next few seconds?
4. Why can a token bucket be implemented without a background refill timer?
5. When would you choose a leaky bucket (queue) over a token bucket?
6. With a fixed window of 100 requests/minute, what's the most requests a client can make in a 2-second
   span? Explain.
7. Why is the sliding window log rarely used for high-volume per-user limits?
8. Previous window: 50 requests, current window: 40 requests so far, we are 40% into the current window,
   limit 80. Does the sliding window counter allow the next request?
9. Two app servers check a Redis counter with GET, compare to the limit, then SET the incremented value.
   What can go wrong and how do you fix it?
10. Your API runs on 20 servers behind a load balancer and autoscales between 10 and 40. Why is "each server
    enforces limit / 20" a poor design?
11. What is the trade-off between checking a shared Redis counter on every request vs local counting with
    periodic sync? When would you pick each?
12. The Redis cluster backing your rate limiter goes down. What should happen for (a) a public read API, (b)
    an OTP verification endpoint?
13. Why should you deploy a new rate limit in shadow mode first?
14. Why is IP address a weak rate limiting key, and what are its legitimate uses?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Protection: limit login attempts to slow credential stuffing, keyed by IP and by target account.
Fairness: per-tenant API limits in a multi-tenant SaaS so one tenant can't starve others, keyed by
tenant/API key. Cost: cap SMS sends per user per hour, keyed by user ID (and globally, to cap total spend).

**2.** Rate limiting asks whether a specific client exceeded its allowance, regardless of server state;
load shedding asks whether the server is over capacity, regardless of who sent the work. Many clients each
within their limits can still overload the server (needs shedding), and one client can unfairly consume
capacity while the server is healthy (needs rate limiting).

**3.** After a minute idle the bucket is full at 20 tokens (capped). In the one second, 20 are allowed
immediately, plus about 5 more refilled during that second: ~25 allowed, ~25 rejected. Afterwards, the
client is held to 5 requests/second; the bucket refills to 20 only if it stays idle for 4 seconds.

**4.** Refill depends only on elapsed time. Storing `tokens` and `last_refill_time`, each request computes
`tokens = min(B, tokens + (now - last) × r)` before checking. The result is identical to continuous
refilling, with O(1) work per request and no timers for millions of idle keys.

**5.** When the goal is to shape traffic into a smooth constant rate for a downstream that can't handle
bursts (a partner API with a strict per-second limit, an email sender, a fragile legacy system), and when
added queueing delay is acceptable. Token buckets pass bursts straight through.

**6.** Up to 200: 100 in the last second of one window and 100 in the first second of the next. Fixed
windows reset counters at boundaries, so the limit only holds per calendar window, not per arbitrary
60-second span.

**7.** It stores one timestamp per request in the window per key: memory O(limit) per key. For high limits
and millions of keys that's huge, and each check does range deletions and counts. It's best for low limits
where exactness matters (e.g., 5 login attempts per 15 minutes).

**8.** Overlap with previous window = 60%. Estimate = 50 × 0.6 + 40 = 70 < 80, so the request is allowed.

**9.** Both servers can read the same value (e.g., 99), both decide to allow, both write 100: two requests
allowed at the limit and one increment lost, the lost update anomaly. Fix by making check-and-update
atomic: `INCR` and compare the returned value, or run the whole algorithm in a Lua script. Also set the
TTL atomically so a crash between INCR and EXPIRE can't leave a never-expiring key.

**10.** The divisor is wrong whenever the server count changes: at 40 servers clients get double their
limit; at 10 they get half. It also assumes each client's requests spread evenly across servers; with
connection reuse (topic 03) or sticky routing a client may hit only a few servers and be limited far below
its allowance. You need a shared count or an allocation that adapts.

**11.** A shared check is exact but adds a round trip per request, makes the store a hot, critical
dependency, and is hard across regions. Local counting with periodic sync is fast and resilient but allows
bounded overshoot between syncs. Pick shared/exact for low-volume security or hard-cost limits; pick
local-with-sync for high-volume fairness/capacity limits where a few percent overshoot is fine.

**12.** (a) Fail open, ideally to a local approximate limiter, with a short timeout and circuit breaker on
the limiter call, and alerting; server-side load shedding remains as a backstop. (b) Fail closed (or to a
strict local limit): unlimited OTP guesses would make brute force feasible, which is worse than briefly
blocking verification.

**13.** You can't easily predict which real clients a new limit will affect (a big customer with a
legitimate batch job, a mobile app retrying). Shadow mode logs "would have been limited" decisions
without enforcing, so you can measure impact, adjust limits or contact clients, then enforce safely.
It's an instance of the safe-change practices in topic 47.

**14.** Many legitimate users can share one IP (NAT, corporate proxies, mobile carrier gateways), so a
per-IP limit punishes them collectively; attackers can rotate through many IPs (botnets, cloud IPs, IPv6
ranges). It's still useful for unauthenticated endpoints (signup, login, anonymous browsing) and at the
edge for DDoS mitigation, usually combined with other keys (account, device fingerprint) and limiting IPv6
by prefix.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Three reasons to rate limit?
**A:** Protection (abuse, accidents), fairness between clients/tenants, and cost control.

**Q:** Rate limiting vs load shedding?
**A:** Rate limiting caps each client regardless of server health (429); load shedding rejects work when the server is over capacity (503).

**Q:** Token bucket parameters?
**A:** Refill rate r (sustained rate) and capacity B (max burst).

**Q:** Max requests through a token bucket in time T?
**A:** B + r × T.

**Q:** How is a token bucket refilled without timers?
**A:** Lazily: store tokens and last timestamp; on each request add elapsed × r, capped at B.

**Q:** Leaky bucket (queue) property?
**A:** Smooths bursts into a constant output rate, at the cost of queueing delay.

**Q:** Fixed window weakness?
**A:** Boundary burst: up to 2× the limit around a window edge; synchronized resets.

**Q:** Sliding window log mechanism?
**A:** Store each accepted request's timestamp; drop those older than the window; allow if count < limit.

**Q:** Sliding window log cost?
**A:** O(limit) memory per key.

**Q:** Sliding window counter formula?
**A:** previous_count × (fraction of previous window still overlapping) + current_count.

**Q:** Sliding window counter trade-off?
**A:** O(1) memory, near-exact; assumes uniform traffic in the previous window.

**Q:** Which algorithm for 5 login attempts per 15 minutes?
**A:** Sliding window log (exact, low volume).

**Q:** HTTP status and headers for rate limiting?
**A:** 429 Too Many Requests, Retry-After, RateLimit-Limit/Remaining/Reset.

**Q:** Why is rate limiting across servers hard?
**A:** A client's requests spread over many servers, so limits must apply to the combined count.

**Q:** Race in a shared counter?
**A:** Read-then-write lets concurrent requests both pass; use atomic INCR or a Lua script.

**Q:** Hidden INCR + EXPIRE bug?
**A:** Crash between them leaves a key with no TTL, blocking the client forever; set TTL atomically.

**Q:** Cost of a shared counter store?
**A:** Extra round trip per request, hot critical dependency, cross-region latency.

**Q:** Local limiting with periodic sync?
**A:** Count locally, push/pull global totals every interval; bounded overshoot, low latency.

**Q:** Why is "limit / N per server" fragile?
**A:** N changes with autoscaling and traffic isn't evenly spread.

**Q:** Fail-open vs fail-closed?
**A:** Open: allow all if limiter is down (availability first). Closed: reject all (protection first).

**Q:** Typical choice for API limits?
**A:** Fail open, to a local approximate limiter, with short timeout, circuit breaker and alerts.

**Q:** When fail closed?
**A:** Security or hard-cost controls (OTP, login, paid SMS).

**Q:** Why is IP a weak key?
**A:** NAT/proxies share IPs among many users; attackers rotate IPs.

**Q:** What is shadow mode?
**A:** Logging would-be rejections without enforcing, to measure impact before turning a limit on.

**Q:** Weighted rate limiting?
**A:** Requests consume cost units proportional to their expense instead of 1 each.
