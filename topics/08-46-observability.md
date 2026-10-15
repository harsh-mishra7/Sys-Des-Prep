# 46 — Observability

> Part 8 — Reliability & Operations · Prerequisites: topics 01, 02, 26, 43, 44

Every earlier topic described how a system *should* behave. This topic is about how you find out how it
*does* behave. In a single process you can attach a debugger. In a distributed system, a single user
request may touch thirty services, a queue and three databases on a hundred machines, and the thing that
went wrong may be a tail-latency effect (topic 02) affecting 0.5% of requests. **Observability** is the
property of a system that lets you answer questions about its internal state from the outside, using the
data it emits, including questions you didn't anticipate when you built it. In practice that data comes in
three main forms (metrics, logs and traces), and it's tied to decisions through targets: SLIs, SLOs, SLAs
and error budgets. This topic covers what each signal is good and bad at, how distributed tracing actually
works, how to define reliability targets that mean something, and the standard checklists (golden signals,
RED, USE) for knowing what to measure.

By the end of this topic you should be able to:
1. Explain what metrics, logs and traces each answer, how they're stored, and what each costs.
2. Explain why aggregating percentiles is tricky and how histograms address it.
3. Explain how distributed tracing works: trace IDs, spans, parent/child relationships, context propagation,
   and sampling.
4. Define SLI, SLO, SLA and error budget, and explain how error budgets drive decisions.
5. Apply the four golden signals, the RED method and the USE method to choose what to measure.
6. Explain the difference between monitoring and observability, and what makes good alerts.

---

## 1. Monitoring vs observability

**Monitoring** is watching for known failure modes: "alert if error rate > 1%", "alert if disk > 90%". You
decide in advance what to look at. It answers "is something I anticipated going wrong?"

**Observability** is being able to ask *new* questions without shipping new code: "why are checkout
requests from Android users in one region slow since 14:05, but only for carts with more than 20 items?"
Distributed systems fail in novel ways (topic 26: partial failures, gray failures where a component is
degraded but not dead), so you'll regularly face questions nobody predicted.

The two are complementary: monitoring tells you *that* something is wrong (and wakes you up);
observability helps you find *why*. Both rest on the same telemetry.

---

## 2. The three signals

### 2.1 Metrics

A **metric** is a numeric measurement aggregated over time, identified by a name and a set of labels
(also called tags or dimensions):

```
 http_requests_total{service="checkout", method="POST", status="500"}  →  1,234
 http_request_duration_seconds_bucket{service="checkout", le="0.1"}     →  98,120
 db_connections_in_use{pool="orders"}                                   →  17
```

Common types:

| Type | What it is | Example |
|---|---|---|
| **Counter** | Monotonically increasing count; you look at its rate | requests served, errors, bytes sent |
| **Gauge** | A value that goes up and down; you look at its current value | queue depth, memory used, in-flight requests |
| **Histogram** | Counts of observations falling into buckets | request latency, payload size |
| **Summary** | Client-computed quantiles | p99 latency computed in-process |

**What metrics answer well:** "how much / how often / how fast, over time?" Is the error rate rising? What's
the p99 latency trend? How full is the disk? They're the basis for dashboards and alerts.

**Why they're cheap:** a metric is pre-aggregated. Whether a service handles 10 or 10 million requests per
minute, the counter is one number per time interval per label combination. Storage cost depends on the
number of distinct time series, not on traffic.

**The catch: cardinality.** Every unique combination of label values is a separate time series.

```
 labels: service (20) × endpoint (50) × status (10) × region (5)   = 50,000 series   fine
 add user_id (1,000,000)                                           = 50 billion      not fine
```

High-cardinality labels (user IDs, request IDs, full URLs with IDs in them, email addresses) explode
metric storage. Metrics therefore can't answer "what happened to *this* request / *this* user?" They lose
individual detail by design. That's what logs and traces are for.

### 2.2 Percentiles in metrics: why histograms

Topic 01 explained that averages hide tail latency and you need percentiles. Metrics add a subtlety:
**you cannot average percentiles.** If server A's p99 is 100 ms and server B's p99 is 500 ms, the fleet p99
is *not* 300 ms; it depends on how many requests each served and on their full distributions. Likewise you
can't compute an hourly p99 by averaging minutely p99s.

Histograms solve this. Each server counts observations into fixed buckets (e.g., ≤10 ms, ≤50 ms, ≤100 ms,
≤500 ms, ≤1 s, ...). Bucket counts **can** be summed across servers and time ranges, and the percentile is
estimated from the merged buckets:

```
 bucket (≤)    server A   server B   merged
 50ms           900        100        1,000
 100ms          990        500        1,490
 500ms        1,000        990        1,990
 1s           1,000      1,000        2,000

 merged p99 = the 1,980th observation → falls in the ≤500ms bucket
```

The accuracy depends on bucket boundaries; put boundaries near your SLO thresholds (if the target is
"under 300 ms", have a bucket at exactly 300 ms). Newer formats (exponential/native histograms, and
sketches like t-digest and DDSketch) give good accuracy across a wide range with bounded error. Client-side
"summaries" compute exact percentiles per process but cannot be merged, which is why histograms are
generally preferred.

### 2.3 Logs

A **log** is a record of a discrete event, with a timestamp and context:

```
 {"ts":"2026-10-15T10:31:02.118Z","level":"error","service":"checkout",
  "trace_id":"4bf92f3577b34da6","user_id":"u_8812","order_id":"o_5521",
  "msg":"payment authorization failed","provider":"cardco","error":"timeout after 2000ms"}
```

**What logs answer well:** "what exactly happened, in this case?" The specific error message, the
parameters, the sequence of events for one request or one user. They hold the high-cardinality detail that
metrics drop.

**Structured logs** (key-value / JSON) rather than free text make logs queryable: "count errors where
provider=cardco grouped by error", instead of regex-searching sentences. Including the **trace ID** in
every log line connects logs to traces (section 3).

**What they cost:** logs scale with traffic. Each request may produce several lines; at 100,000 requests per
second that's terabytes per day to ship, index and store. Practices to control cost:

- Log levels; keep DEBUG off in production by default.
- Sample high-volume, low-value logs (log 1% of successful requests, 100% of errors).
- Retention tiers: hot searchable storage for days, cheap object storage (topic 53) for longer.
- Don't log in tight loops; aggregate into metrics instead.

Logs also need care for **privacy and security**: no passwords, tokens or full card numbers; personal data
may be subject to regulation and deletion requirements.

**Events vs logs:** a growing practice (sometimes called "wide events" or canonical log lines) is to emit
one rich, structured event per request per service with many fields (duration, status, user tier, cart
size, feature flags, region, build version...). Such events can be sliced along any dimension after the
fact, which is the core of the "observability" capability in section 1, at the cost of storing them.

### 2.4 Traces

A **trace** records the path of one request through a distributed system: every service and operation it
touched, with timing and parent/child relationships.

```
 trace 4bf92f3577b34da6        0ms         100ms        200ms        300ms
 ─────────────────────────────────────────────────────────────────────────
 GET /checkout (gateway)       ├────────────────────────────────────────┤ 310ms
   auth.verify                 ├──┤ 15ms
   cart.get                       ├────┤ 40ms
     redis GET cart:u_8812         ├┤ 2ms
   pricing.quote                       ├──────┤ 60ms
   payment.authorize                          ├─────────────────────┤ 180ms
     cardco HTTP POST                          ├────────────────────┤ 175ms  ← the problem
```

**What traces answer well:** "where did the time go for this request?" and "which services did this request
depend on, and which one failed?" In a fan-out architecture, a latency metric on the gateway tells you the
request was slow; only a trace shows that 175 ms of the 310 ms was a third-party call nested three levels
deep.

**Aggregated over many traces**, they also answer structural questions: what is the dependency graph
(service map)? Which downstream contributes most to p99 for this endpoint?

### 2.5 Comparison

| | Metrics | Logs | Traces |
|---|---|---|---|
| Unit | Number over time per label set | Discrete event | One request's path (tree of spans) |
| Answers | Is it happening? How much? Trend? | What exactly happened here? | Where did time go? Which hop failed? |
| Cost scales with | Number of time series (cardinality) | Traffic volume | Traffic × sampling rate |
| High-cardinality detail | No | Yes | Yes (as span attributes) |
| Good for alerting | Yes | Limited (log-derived metrics) | Rarely directly |
| Typical tools | Prometheus, Graphite, Datadog metrics | ELK/OpenSearch, Loki, Splunk | Jaeger, Zipkin, Tempo |

A typical debugging workflow moves between them:

```
 alert fires on metric (p99 latency for /checkout)     ← metrics: THAT something's wrong
   → dashboard narrows it: region=eu, since 14:05       ← metrics with labels
   → exemplar / trace search: slow checkout traces      ← traces: WHERE the time goes
   → payment.authorize span slow; open its logs by trace_id  ← logs: WHAT exactly happened
```

**Exemplars** (a sample trace ID attached to a histogram bucket) and putting trace IDs in logs are what let
you jump between signals. **OpenTelemetry** has become the standard vendor-neutral API, SDK and wire format
for producing all three signals, so instrumentation isn't tied to one backend.

---

## 3. Distributed tracing in depth

### 3.1 Spans and traces

- A **span** is one timed operation: "handle HTTP request in cart service", "Redis GET", "SQL query". It
  has a span ID, a name, start and end times, a status, attributes (key-values like `http.status_code`,
  `db.statement`), and optionally events (timestamped logs within the span).
- Each span has a **parent span ID** (except the root). Parent/child relationships form a tree.
- All spans for one request share a **trace ID**. A **trace** is the set of spans with that trace ID,
  reassembled into a tree.

```
 trace_id = T1
 span A (root)      gateway: GET /checkout        parent = none
 ├─ span B          cart: GetCart                 parent = A
 │   └─ span C      redis: GET                    parent = B
 └─ span D          payment: Authorize            parent = A
     └─ span E      http: POST cardco             parent = D
```

### 3.2 Context propagation

The hard part: spans are created by different processes on different machines. How does the payment
service know its span belongs to trace T1 with parent A?

The trace context is **propagated with the request**. When service A calls service B, it adds the trace ID
and its current span ID to the outgoing request; B reads them, creates its own span with
`parent = A's span ID`, and passes its own span ID onward.

```
 gateway                         cart service                    redis client
 ─────────                       ────────────                    ────────────
 start span A (trace T1)
 call cart with header:
   traceparent: 00-T1-A-01  ───► read header: trace T1, parent A
                                 start span B (parent A)
                                 call redis ─────────────────►  span C (parent B)
                                 end span B
 end span A
 each process exports its spans asynchronously to a collector → backend assembles tree by trace_id
```

The W3C **Trace Context** standard defines the `traceparent` header: version, trace ID (16 bytes), parent
span ID (8 bytes), and flags (including whether this trace is sampled). gRPC carries it in metadata; message
queues (topic 36/37) carry it in message headers so a trace can span asynchronous hops (a producer's span
links to the consumer's span).

Propagation must also work **inside** a process: when a request handler spawns async work, calls a thread
pool, or awaits a future, the current context must follow. Tracing libraries keep context in thread-local or
async-local storage and wrap executors; this is where tracing commonly breaks (a span is "orphaned" because
the context was lost across a thread hop).

If any service in the chain doesn't propagate the header (an old service, a proxy that strips headers), the
trace breaks into disconnected pieces. Tracing is only as good as its weakest hop, which is why service
meshes and standardized libraries (OpenTelemetry auto-instrumentation) matter.

**Baggage** is related: key-value pairs propagated with the context to all downstream services (e.g.,
`tenant=acme`), usable for attributes or routing. It adds bytes to every request, so keep it small.

### 3.3 Sampling

Recording every span of every request is expensive at high traffic. Sampling decides which traces to keep.

- **Head-based sampling:** decide at the root (e.g., keep 1% of traces) and propagate the decision in the
  trace flags so every service records or skips consistently. Cheap and simple, but blind: the decision is
  made before you know whether the request will be slow or fail, so 99% of interesting traces are lost.
- **Tail-based sampling:** every service records spans and sends them to a collector, which buffers each
  trace until it's complete and then decides: keep all errors, all traces slower than p99, and 1% of the
  rest. Captures the interesting traces, but requires buffering all spans briefly and routing all of a
  trace's spans to the same collector instance, which costs more infrastructure.
- **Adaptive/rate-limited sampling:** keep N traces per second per endpoint, so rare endpoints are
  represented and hot endpoints don't dominate.

A consequence: traces are best used for "show me examples of slow requests" and structural analysis, while
**metrics remain the source of truth for rates and percentiles**, since sampled traces don't give exact
counts.

### 3.4 What tracing costs and where it struggles

- Overhead per span (small, but not zero) and network/storage for exporting spans.
- Instrumentation effort and propagation gaps (above).
- Clock skew (topic 27): spans from different machines are timed by different clocks, so a child can appear
  to start before its parent. Tracing UIs adjust for this, but cross-machine timings in a trace are
  approximate; durations measured within one machine are reliable.
- Fan-out-heavy requests (a search touching 500 shards) produce enormous traces.

---

## 4. SLIs, SLOs, SLAs and error budgets

Metrics tell you numbers. Reliability targets tell you which numbers matter and how good is good enough.

### 4.1 Definitions

- **SLI (Service Level Indicator):** a measurement of some aspect of service quality, usually expressed as a
  ratio of good events to total events.
  - Availability SLI: `successful requests / valid requests`
  - Latency SLI: `requests served faster than 300 ms / valid requests`
  - Freshness SLI (for pipelines): `records updated within 10 min / total records`
- **SLO (Service Level Objective):** a target for an SLI over a time window. "99.9% of checkout requests
  succeed over a rolling 28 days." "99% of search requests complete in under 300 ms."
- **SLA (Service Level Agreement):** a *contract* with customers that specifies consequences (refunds,
  credits) if a level isn't met. SLAs are business/legal agreements; they are normally looser than internal
  SLOs so that you notice and react before you owe anyone money.

```
 SLI: what we measure        "fraction of requests < 300ms"     = 99.62% this month
 SLO: what we aim for        "≥ 99.5% of requests < 300ms"       ✓ met
 SLA: what we promise        "≥ 99.0%, or 10% credit"            ✓ far from breach
```

### 4.2 Choosing good SLIs

- **Measure what users experience.** Server CPU is not an SLI; "did the user's request succeed quickly" is.
  The closer to the user you measure (load balancer logs, client-side telemetry), the more accurately the SLI
  reflects reality, though it may include things outside your control.
- **Express as good/total ratios.** This makes them comparable and composable into budgets.
- **Define "valid" and "good" carefully:** do 4xx errors count as failures? (Usually not; the client sent a
  bad request.) Do health checks count? (No.) Is a request that succeeded after a retry a success? (From the
  user's view, yes.)
- **Use percentile-based latency SLIs**, not averages (topic 01).

### 4.3 Choosing SLO targets: the nines

| SLO | Allowed bad over 30 days | Per day |
|---|---|---|
| 99% | 7.2 hours | 14.4 min |
| 99.9% | 43.2 minutes | 1.44 min |
| 99.95% | 21.6 minutes | 43 s |
| 99.99% | 4.3 minutes | 8.6 s |
| 99.999% | 26 seconds | 0.86 s |

Each extra nine costs dramatically more (redundancy, multi-region, slower release processes) and users
often can't tell the difference: if a user's own phone network is 99% reliable, a 99.999% service is
indistinguishable from 99.99%. Also, a service cannot be more available than the dependencies it calls
serially without redundancy: if it needs three dependencies each at 99.9%, its ceiling is roughly
0.999³ ≈ 99.7%. **100% is the wrong target**: it's unachievable and forbids all change.

### 4.4 Error budgets

The **error budget** is the complement of the SLO: with a 99.9% SLO over 28 days, 0.1% of requests (about
40 minutes' worth of full outage) may fail. That's a budget to *spend*.

```
 SLO 99.9% over 28 days, 10,000,000 requests expected
 error budget = 10,000 failed requests

 day 1–10:  3,000 failures (a bad deploy)     → 30% of budget spent
 day 11–20: 1,000 failures                     → 40% spent
 day 21:    major incident, 7,000 failures     → 110% → budget EXHAUSTED
```

Why it's powerful: it turns reliability from an argument into a policy.

- **Budget remaining:** the team can ship features, run experiments, do risky migrations. Reliability is
  "good enough"; spending effort to exceed the SLO is wasted.
- **Budget exhausted:** a pre-agreed policy kicks in: freeze non-critical releases, prioritize
  reliability work, require extra review, until the budget recovers.

It aligns incentives: developers want velocity, operators want stability; both agree on the number. It also
gives a principled answer to "should we do this risky deploy on Friday?": how much budget is left?

### 4.5 Alerting on SLOs: burn rate

The naive alert "error rate > 0.1%" fires constantly on tiny blips and doesn't distinguish "will exhaust
the budget in 2 hours" from "will exhaust it in 3 weeks". Better: alert on **burn rate**, how fast the
budget is being consumed relative to the steady rate that would exactly exhaust it at the end of the window.

```
 burn rate 1   → budget runs out exactly at end of window (fine)
 burn rate 14.4 over 1 hour → 2% of a 30-day budget gone in one hour → page someone now
 burn rate 6 over 6 hours   → 5% gone → page
 burn rate 1 over 3 days    → ticket, look at it this week
```

The Google SRE Workbook recommends **multi-window, multi-burn-rate** alerts: page if a high burn rate is
seen over both a long window (e.g., 1 h, to be significant) and a short window (e.g., 5 min, to confirm
it's still happening and reset quickly after recovery).

### 4.6 What makes a good alert

- **Alert on symptoms, not causes.** Page on "users see errors / slowness" (SLO burn), not on "CPU at 90%"
  (which might be fine) or "one replica down" (which redundancy handles). Cause-based signals go on
  dashboards and in tickets.
- **Every page must be actionable and urgent.** If the right response is "wait and see", it's not a page.
- **Avoid alert fatigue:** noisy alerts train people to ignore them, so the real one gets ignored too.
- **Link alerts to runbooks and dashboards.**

---

## 5. What to measure: golden signals, RED, USE

With unlimited possible metrics, these checklists tell you which matter.

### 5.1 The four golden signals (Google SRE book)

For any user-facing service:

| Signal | Meaning | Notes |
|---|---|---|
| **Latency** | Time to serve requests | Separate successful from failed: a fast error (failing in 1 ms) can drag averages down and hide a problem. Use percentiles. |
| **Traffic** | Demand on the system | Requests/sec, sessions, transactions/sec, bytes/sec |
| **Errors** | Rate of failed requests | Explicit (5xx), implicit (200 with wrong content), policy (slower than SLO counts as failure) |
| **Saturation** | How "full" the service is | The most constrained resource: CPU, memory, I/O, thread pools, queue depth. Often a leading indicator: latency rises as saturation approaches 100% (topic 02). |

### 5.2 RED method (for request-driven services)

Tom Wilkie's simplification for each microservice/endpoint:

- **R**ate: requests per second.
- **E**rrors: failed requests per second.
- **D**uration: distribution of request latency (histograms → percentiles).

RED is essentially the golden signals minus saturation, applied uniformly to every service so all dashboards
look alike. It describes the *user's* (caller's) view of a service.

### 5.3 USE method (for resources)

Brendan Gregg's method for every *resource* (CPU, memory, disk, network interface, but also thread pools,
connection pools, locks, queues):

- **U**tilization: fraction of time the resource is busy (or fraction of capacity used).
- **S**aturation: extra work queued because the resource is full (run-queue length, swap activity, pool
  wait count, queue depth).
- **E**rrors: error events on the resource (disk errors, dropped packets, failed allocations).

USE is for finding *why* a service is slow (which resource is the bottleneck); RED is for knowing *whether*
it is slow. Saturation matters especially: a resource at 100% utilization with no queue is fine; one at 100%
with a growing queue is the bottleneck.

```
 RED: is the service healthy from the outside?    → "checkout p99 latency doubled"
 USE: which resource inside is the problem?        → "DB connection pool: 100% used, 40 waiting"
```

### 5.4 Signals from earlier topics

The reliability mechanisms of topics 43–45 need their own metrics, because they hide problems by design:

- Retry rate and retry-budget exhaustion (topic 43): a rising retry rate is an early warning of a storm.
- Timeouts per dependency; deadline-exceeded counts.
- Circuit breaker state changes; bulkhead rejections; load-shedding counts by priority (topic 44).
- Fallback usage (topic 44): silent degradation is only visible through metrics.
- Rate limiter rejections per key (topic 45).
- Queue depth and consumer lag (topics 36, 37).
- Cache hit ratio (topic 11), replication lag (topic 23).

---

## 6. How it connects

- **Topic 01, 02:** percentiles and tail latency are what latency metrics must capture; saturation near
  100% predicts latency blow-up.
- **Topic 10:** service meshes and gateways emit RED metrics and propagate trace headers without app changes.
- **Topic 26:** partial and gray failures are why you need observability rather than just up/down checks.
- **Topic 27:** clock skew distorts cross-machine timings in traces and log ordering.
- **Topic 36, 37:** trace context in message headers spans async hops; consumer lag is a key saturation metric.
- **Topic 43, 44, 45:** retries, breakers, shedding and limits must be instrumented, or they hide trouble.
- **Topic 47:** canary analysis compares SLIs between old and new versions; error budgets gate releases.
- **Topic 48:** RPO/RTO are the disaster-recovery analogues of SLOs; per-cell metrics reveal blast radius.
- **Topic 50:** sketches (t-digest, DDSketch, HyperLogLog) power mergeable percentile and cardinality metrics.

---

## Common misconceptions

- "Observability = having metrics, logs and traces." → Having the three signals is the means; the goal is
  being able to answer new questions. Three disconnected data silos with no shared IDs give little of that.
- "Average latency is a good health metric." → Averages hide the tail; use percentiles from histograms.
- "You can average p99s across servers." → Percentiles don't average; merge histograms instead.
- "Put user_id as a metric label so we can debug per user." → That explodes cardinality; per-user detail
  belongs in logs/traces.
- "Trace 100% of requests." → Usually too expensive; sample, preferably tail-based to keep errors and slow
  traces.
- "Higher SLOs are always better." → Each nine costs far more and slows change; target what users need.
- "SLA and SLO are the same." → An SLO is an internal target; an SLA is a contract with penalties, set looser.
- "Alert on high CPU." → Page on user-visible symptoms (SLO burn); resource metrics are for diagnosis.
- "An exhausted error budget means the team failed." → It's a signal to shift priorities to reliability,
  by policy, not a blame mechanism.

## Further reading (supplements)

- *Site Reliability Engineering* (Google), chapters 4 "Service Level Objectives" and 6 "Monitoring
  Distributed Systems" (golden signals).
- *The Site Reliability Workbook* (Google), chapters "Implementing SLOs" and "Alerting on SLOs" (burn rates).
- Benjamin Sigelman et al., "Dapper, a Large-Scale Distributed Systems Tracing Infrastructure", Google
  technical report (2010).
- Brendan Gregg, "The USE Method" (brendangregg.com).
- Tom Wilkie, "The RED Method: key metrics for microservices architecture" (Grafana/Weaveworks blog and talk).
- Charity Majors, Liz Fong-Jones & George Miranda, *Observability Engineering* (O'Reilly, 2022).
- W3C Trace Context recommendation; OpenTelemetry documentation (concepts: signals, context propagation,
  sampling).

---

## Questions

1. For each of these questions, which signal (metrics, logs, traces) would you reach for first, and why? (a) Is
   the checkout error rate higher than yesterday? (b) Why did order o_5521 fail? (c) Which downstream call
   makes /search slow at p99?
2. Why can't you add `user_id` as a label on a request-count metric? Where should that information go?
3. Server A reports p99 = 80 ms on 1,000 requests and server B reports p99 = 400 ms on 100 requests. What is
   the fleet p99? What would you need to compute it properly?
4. Walk through how a trace is assembled when a request flows gateway → cart → Redis, including what header
   carries what.
5. A trace shows the gateway span and the payment span, but nothing in between and they're not connected.
   What likely happened?
6. Compare head-based and tail-based sampling. Which would you choose if you mainly want to debug rare
   errors?
7. Define SLI, SLO and SLA for a search service. Why is the SLA looser than the SLO?
8. A service depends serially on three services, each at 99.9% availability. What availability can it
   promise at most, and what could raise that ceiling?
9. Your SLO is 99.9% over 30 days and you've used 90% of the error budget by day 12. What does a typical
   error-budget policy say, and why is that better than an ad-hoc debate?
10. Why is "page when the error rate exceeds 0.1%" a poor alert for a 99.9% SLO? What's better?
11. Explain the difference between RED and USE, and give an example where USE finds what RED cannot.
12. Why should latency for failed requests be tracked separately from successful ones?
13. Your service uses retries, circuit breakers and fallbacks. Why might its SLO dashboards look healthy while
    the system is close to collapse, and what metrics would reveal it?
14. Why is "saturation" often called a leading indicator?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** (a) Metrics: an error-rate time series compared over time, cheap and exact. (b) Logs (and that
request's trace): you need the specific event's details, found by order ID or trace ID. (c) Traces: they
show per-hop timing within requests; filter to slow /search traces and see which spans dominate. Metrics
alone show /search is slow, not where inside.

**2.** Every label value combination is a separate time series; millions of users × other labels would
create billions of series and overwhelm the metrics store. Per-user detail belongs in structured logs or
trace attributes, which handle high cardinality because they're stored per event (and sampled/indexed).

**3.** You can't compute it from the two p99s; percentiles don't average. With 1,100 requests, the p99 is the
1,089th fastest; it depends on the full distributions. If each server exported a histogram with the same
buckets, you'd sum bucket counts and read the merged p99 from the combined distribution.

**4.** The gateway starts a root span A with a new trace ID T1. When calling cart, it adds a `traceparent`
header containing T1, A's span ID, and the sampled flag. Cart reads it, starts span B with parent A and the
same trace ID, and when calling Redis, its client library creates span C with parent B. Each process exports
its finished spans asynchronously to a collector; the backend groups spans by trace ID and builds the tree
from parent IDs.

**5.** Context propagation broke somewhere between them: a service in the middle isn't instrumented, a proxy
strips the header, or the context was lost across a thread/async hop inside a service (e.g., work handed to an
executor without carrying the context). The payment service then started a new trace instead of continuing
T1.

**6.** Head-based decides at the root before the outcome is known (cheap, consistent, but keeps only a random
fraction, so most rare errors are discarded). Tail-based buffers complete traces and decides afterward (keep
all errors and slow traces), at the cost of collector infrastructure. For rare errors, tail-based.

**7.** SLI: fraction of search requests that return successfully in under 400 ms. SLO: ≥99.5% over a rolling 28
days. SLA: customers get a credit if fewer than 99% of searches meet that over a calendar month. The SLA is
looser so the team reacts (SLO breached, budget exhausted) well before contractual penalties apply.

**8.** About 0.999³ ≈ 99.7%, since a failure of any of them fails the request. To raise it: make calls
non-serial or optional (graceful degradation, topic 44), add caching/fallbacks so a dependency failure
doesn't fail the request, or make the dependencies more available (redundancy).

**9.** A typical policy freezes non-essential feature releases and redirects engineering effort to
reliability (fixing the causes of the burn, improving rollback/testing) until the budget recovers. It's
better than ad-hoc debate because it was agreed in advance by product and engineering, it's tied to a
user-centered number, and it removes the emotional "who's right" argument during an incident.

**10.** It ignores duration: a 2-minute blip at 0.2% errors pages someone even though it costs almost no
budget, while a slow sustained 0.15% burn eventually exhausts the budget with the same alert noise. Better:
burn-rate alerts over multiple windows, paging on fast burn (e.g., 14.4× over 1 h confirmed by 5 min) and
ticketing on slow burn.

**11.** RED (rate, errors, duration) describes a service from its callers' view: is it healthy? USE
(utilization, saturation, errors) describes each resource: which one is the bottleneck? Example: RED shows
p99 latency doubled on checkout; USE on resources shows the DB connection pool at 100% utilization with 40
waiters (saturation) while CPU is at 30%, pointing at pool size or slow queries rather than compute.

**12.** Failures are often very fast (immediate rejections, circuit-open errors, validation failures).
Mixing them into a latency distribution drags percentiles down, making things look faster precisely when
they are failing. Conversely, slow timeouts can distort the success distribution. Track latency per
outcome.

**13.** These mechanisms mask failures: retries turn errors into successes (with extra load), fallbacks serve
degraded responses that count as "success", and breakers fail fast. The SLI may look fine while the system
burns retry budgets and runs on fallbacks. Reveal it with retry rate, retry-budget exhaustion, breaker state
changes, fallback-served counts, shed counts, and dependency error rates measured before retries.

**14.** Latency and errors rise only once saturation is already high: queueing theory (topic 02) says wait
time grows sharply as utilization nears 100%. Watching queue depths, pool wait counts and utilization trends
shows the system approaching the cliff before users feel it.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Monitoring vs observability?
**A:** Monitoring checks known failure modes; observability lets you ask new, unanticipated questions from telemetry.

**Q:** What do metrics answer?
**A:** How much, how often, how fast, over time: trends and alerts.

**Q:** What do logs answer?
**A:** What exactly happened in a specific event.

**Q:** What do traces answer?
**A:** Where time went and which hop failed for a request across services.

**Q:** Metric cost driver?
**A:** Cardinality: the number of distinct label combinations (time series).

**Q:** Log cost driver?
**A:** Traffic volume.

**Q:** Counter vs gauge?
**A:** Counter only increases (look at its rate); gauge goes up and down (look at its value).

**Q:** Why histograms instead of reported percentiles?
**A:** Percentiles can't be averaged; histogram buckets can be summed across servers and time.

**Q:** Structured logging?
**A:** Key-value/JSON log events that can be queried and aggregated, including trace IDs.

**Q:** What is a span?
**A:** One timed operation with trace ID, span ID, parent span ID, timing, status and attributes.

**Q:** What is a trace?
**A:** All spans sharing a trace ID, assembled into a tree by parent IDs.

**Q:** Context propagation?
**A:** Passing trace ID and parent span ID with each outgoing call (e.g., W3C `traceparent` header) and across threads.

**Q:** Head vs tail sampling?
**A:** Head: decide at the root, before outcome. Tail: decide after the trace completes, keeping errors/slow ones.

**Q:** SLI?
**A:** A measured ratio of good events to valid events (e.g., requests under 300 ms / all requests).

**Q:** SLO?
**A:** A target for an SLI over a window (e.g., 99.9% over 28 days).

**Q:** SLA?
**A:** A contract with consequences if a level isn't met; looser than the SLO.

**Q:** Error budget?
**A:** 1 − SLO: the allowed amount of failure in the window, spent on change and risk.

**Q:** What happens when the error budget is exhausted?
**A:** Pre-agreed policy: freeze risky releases, prioritize reliability until it recovers.

**Q:** 99.9% monthly downtime allowance?
**A:** About 43 minutes.

**Q:** Burn rate?
**A:** How fast the error budget is consumed relative to the rate that would use exactly all of it over the window.

**Q:** Four golden signals?
**A:** Latency, traffic, errors, saturation.

**Q:** RED?
**A:** Rate, Errors, Duration: per request-driven service.

**Q:** USE?
**A:** Utilization, Saturation, Errors: per resource.

**Q:** Alert on symptoms or causes?
**A:** Symptoms (user-visible SLO burn) for pages; causes for dashboards and diagnosis.

**Q:** Availability ceiling with three serial 99.9% dependencies?
**A:** About 99.7%.

**Q:** What is an exemplar?
**A:** A sample trace ID attached to a metric data point, linking metrics to traces.
