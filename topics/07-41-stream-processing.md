# 41 — Stream processing

> Part 7 — Moving Data Between Systems · Prerequisites: topics 27, 37, 38, 40

Batch processing (topic 40) assumes the input is finished: yesterday's logs, a full export. But most data
never "finishes" — clicks, payments, sensor readings and database changes keep arriving forever. If you
wait for a batch boundary to process them, results are hours stale. **Stream processing** runs
computations continuously over **unbounded** input, producing results seconds (or milliseconds) after the
events happen: fraud alerts, live dashboards, real-time recommendations, keeping derived stores in sync.
Doing that correctly is harder than it sounds. Events arrive late and out of order, so "the count for
10:00–10:05" is never obviously complete. Aggregations need state that must survive crashes. And
"exactly-once" must be achieved without the luxury of rerunning a whole job. This topic covers streams as
data, event time vs processing time, windows, watermarks and late data, stateful processing with
checkpoints (Flink as the example), exactly-once in stream processors, and the Lambda vs Kappa
architectures.

By the end of this topic you should be able to:
1. Explain what makes stream processing different from batch: unbounded input, and the duality of events (streams) and state (tables).
2. Distinguish event time from processing time, and explain why results based on processing time are wrong under delays and replays.
3. Describe tumbling, sliding (hopping) and session windows and when to use each.
4. Explain watermarks, allowed lateness and the trade-off between latency and completeness, including how late events are handled.
5. Explain how stateful operators keep state (state stores) and survive failures via checkpointing (Chandy–Lamport-style barriers in Flink).
6. Explain how exactly-once works inside a stream processor and end-to-end (replayable sources + checkpoints + transactional or idempotent sinks).
7. Compare Lambda and Kappa architectures.

---

## 1. Streams as unbounded data

### 1.1 What a stream is

A **stream** is a sequence of **events** — small, immutable records saying "something happened at time
T": `{user 7 clicked /mug at 10:03:12}`, `{payment p9 for $49 authorised}`, `{row 42 price changed 10 →
12}` (a CDC event, topic 39). Events are produced continuously, usually written to a log such as Kafka
(topic 37), and consumed by stream processors.

Difference from batch, in one sentence: **the input is never complete**. A batch job can sort everything,
then count; a stream job must emit results while more input is still coming, so it must decide *when* a
result is ready and *what to do* if more relevant data shows up later.

### 1.2 What stream processors do

Common jobs, roughly in increasing difficulty:

| Job | Example | Needs state? |
|---|---|---|
| Stateless transform / filter / route | Drop bot traffic; convert currencies; split a topic | No |
| Enrich (stream–table join) | Add user's country to each click | Yes: the table |
| Windowed aggregation | Clicks per product per minute | Yes: partial aggregates |
| Pattern detection (CEP) | 3 failed logins then a success within 1 minute | Yes: per-user partial matches |
| Stream–stream join | Match each ad impression with a click within 30 min | Yes: buffered events from both sides |
| Maintain a materialised view | Current inventory per SKU from change events | Yes: the view |

Engines: **Apache Flink**, **Kafka Streams** (a library inside your app, using Kafka for everything),
**Spark Structured Streaming** (micro-batches or continuous), Apache Beam (an API with runners on
Flink, Dataflow and others), and managed services (Google Cloud Dataflow, Amazon Kinesis Data Analytics
/ Managed Flink).

### 1.3 Events vs state: the stream–table duality

Two ways of looking at the same data:

- A **table** is the *current state*: `balance[acct 7] = 120`.
- A **stream** is the *history of changes*: `+100, +50, -30`.

They're interconvertible:
- **Stream → table:** apply (fold/aggregate) the changes in order. `state = fold(apply, initial, events)`.
- **Table → stream:** capture every change to the table (CDC, topic 39) — the changelog.

```
stream (changelog)            table (state at the end)
acct7 +100                    ┌──────┬─────────┐
acct9 +40         fold ──►    │ acct │ balance │
acct7 +50                     │ 7    │ 120     │
acct7 -30         ◄── CDC     │ 9    │ 40      │
                              └──────┴─────────┘
```

Mathematically, state is the integral of the stream over time; the stream is the derivative of the state.
This duality is everywhere: database replication (log → replica tables, topic 23), Kafka compacted
topics as tables (topic 37), Kafka Streams' `KStream`/`KTable`, and event sourcing (topic 42). A stream
processor's **state** (section 5) is a table derived from the streams it has consumed so far.

---

## 2. Event time vs processing time

### 2.1 Two clocks

- **Event time:** when the event actually happened, as recorded by the source (a timestamp in the event).
- **Processing time:** when the stream processor happens to handle it (the machine's clock at that moment).

In a perfect world they'd be nearly equal. In reality the gap (**skew**) varies wildly:

```
event time:       10:00   10:01   10:02   10:03   10:04
                    │       │       │       │       │
                    │ mobile app offline in a tunnel; buffers 3 events
processing time:  10:00   10:01   ...............   10:09 (3 events arrive at once)
```

Causes of delay: mobile clients offline for hours, network retries, a producer batching, a broker
partition lagging, a consumer down for maintenance, and — most dramatically — **replay**: reprocessing a
week of events (topic 37/38) gives processing times all within a couple of hours.

### 2.2 Why processing time gives wrong answers

Suppose we count requests per minute by processing time and the consumer restarts after a 10-minute
outage:

```
Real traffic:  ~1000 req/min, steady

Count by processing time:
10:00 → 1000
10:01–10:10 → 0 (consumer down)
10:11 → 11,000 (backlog drained)   ← looks like a traffic spike that never happened
```

Counting by *event time* gives the true ~1000/min everywhere — just reported later. The general rule:
**if the result describes the real world, window by event time.** Processing time is only appropriate
when you genuinely care about "what the system saw when" (e.g. monitoring the processor's own
throughput) or when approximate, low-latency results are good enough.

### 2.3 Which clock is "event time"?

Even the event timestamp is a clock reading from some device (topic 27). A phone's clock may be wrong by
minutes or set to 2009. A common technique (from DDIA): record three timestamps — the device's time
when the event occurred, the device's time when it was sent, and the server's time when received. The
difference of the last two estimates the device clock's offset, which corrects the first. And for
ordering *within* a key, prefer log order (offsets) over timestamps.

---

## 3. Windows

Aggregating an infinite stream ("total clicks") is either meaningless or a running total. Usually you
want aggregates over **windows** of time.

### 3.1 Tumbling windows

Fixed-size, **non-overlapping**, contiguous. Every event belongs to exactly one window.

```
size 5 min
|  10:00–10:05  |  10:05–10:10  |  10:10–10:15  |
   x  x   x        x x              x   x x x
```

Use for: periodic reports — "orders per 5 minutes", "bytes per hour per customer for billing".

### 3.2 Sliding / hopping windows

Fixed size, but a new window starts every *slide* interval (< size), so windows **overlap** and each
event belongs to several windows.

```
size 10 min, slide 5 min (a "hopping" window)
|  10:00–10:10        |
          |  10:05–10:15        |
                    |  10:10–10:20        |
an event at 10:07 is in windows [10:00,10:10) and [10:05,10:15)
```

Use for: smoothed moving metrics — "error rate over the last 10 minutes, updated every minute",
"average latency over the last 5 minutes".

Terminology varies: Flink calls fixed-slide overlapping windows *sliding windows*; Kafka Streams calls
them *hopping* and reserves "sliding" for windows defined relative to each event (e.g. "all events
within 10 minutes of each other", used for joins). DDIA describes a sliding window as keeping all events
within some interval of each other. The underlying idea — overlapping windows of a fixed length — is the
same.

### 3.3 Session windows

**No fixed size.** A session groups events for a key that are separated by gaps shorter than a
**timeout**; a gap longer than the timeout closes the session.

```
gap timeout = 30 min, one user
events: 10:00 10:05 10:12 ............ (silence 50 min) ......... 11:02 11:10
        └──── session 1 (10:00–10:42) ─┘                         └─ session 2 ─┘
```

Use for: user sessions on a website (pages per visit, visit duration), bursts of device activity.

Subtlety: sessions can **merge**. A late event at 10:40 might bridge two sessions that had been
considered separate, so the engine must merge their state.

### 3.4 Global windows / triggers

Some engines also offer a global window (everything for a key) combined with **triggers** that decide
when to emit results (every N events, every minute of processing time). Running totals use this.

### 3.5 Windowed joins

- **Stream–stream join:** match events from two streams within a window — e.g. a search event and a
  click on a result within 1 hour. The processor buffers events from both sides for the window
  duration (state!), indexed by the join key.
- **Stream–table join (enrichment):** join each event with the current value in a table — e.g. each
  click with the user's profile. The processor keeps a local copy of the table, fed by a CDC/changelog
  stream (topic 39). Faster and more robust than querying a remote database per event.
- **Table–table join:** maintain a materialised join of two changelogs (e.g. a timeline cache).

Joins introduce a time-dependence problem: when the profile changes, which version should an earlier
click be joined with? Processing in event-time order (or versioning the table) makes it deterministic;
otherwise reprocessing may produce different results — a **slowly changing dimension** problem.

---

## 4. Watermarks and late data

### 4.1 The completeness problem

The window 10:00–10:05 (event time) — when can we emit its count? Events with event time 10:04:59 might
arrive at 10:05:01, or 10:07, or tomorrow. We can never be *certain* we've seen everything.

### 4.2 Watermarks

A **watermark** is a statement flowing through the stream: **"I believe no more events with event time
≤ T will arrive."** When the watermark passes the end of a window, the window is considered complete and
its result is emitted.

```
events (event time) arriving in processing order:
  10:01  10:03  10:02  10:06  10:04  10:08  W(10:05)  10:09  10:03(!)  W(10:10)
                                           ▲                   ▲
                    window [10:00,10:05) fires here    late: event time ≤ current watermark
```

How watermarks are generated:
- **Bounded out-of-orderness (heuristic):** watermark = max event time seen − allowed delay (e.g. 5 s).
  Simple and common (Flink's `forBoundedOutOfOrderness`).
- **Source-provided:** if the source knows its data is ordered (e.g. per partition), it can emit precise
  watermarks.
- With parallel inputs, an operator's watermark is the **minimum** of its inputs' watermarks — so one
  slow or idle partition holds back progress for everyone. Engines provide *idleness* detection to
  exclude idle partitions.

### 4.3 The trade-off: latency vs completeness

- An **aggressive** watermark (small delay) emits results quickly, but more events arrive *late* and are
  missed or require corrections.
- A **conservative** watermark (large delay) is more complete, but every result is delayed by that much.

There's no right answer — it depends on how much the application values speed versus accuracy. A fraud
alert wants speed; a billing total wants completeness.

### 4.4 What to do with late events

Once a window has fired, a late event for it can be:

1. **Dropped** (default in many engines) — and counted in a metric so you know how much is dropped.
2. **Applied with allowed lateness:** keep window state for an extra period (Flink's
   `allowedLateness`) and **emit an updated result** when a late event arrives. Downstream must accept
   *updates/retractions* — e.g. upserting the window's row by key rather than appending.
3. **Sent to a side output** (a separate stream of late events) for separate handling or a later batch
   correction.

Emitting early, speculative results and then refinements is also possible (triggers in Beam/Flink: emit
at watermark, plus early firings every minute, plus late firings). This is the model from Google's
Dataflow paper: separate *what* is computed (the window), *where in event time* (windowing), *when in
processing time* results are emitted (watermarks + triggers), and *how refinements relate* (accumulate,
discard, or retract).

---

## 5. Stateful stream processing

### 5.1 Why state

Anything beyond a stateless filter needs memory across events: window aggregates, join buffers, session
state, dedup sets, CEP partial matches, an enrichment table. For a high-throughput job, this state can be
gigabytes to terabytes.

Keeping it in a remote database (one network round trip per event) would be slow and would make
exactly-once hard. Instead, stream processors keep state **local** to the operator, **partitioned by key**
just like the input stream (topic 25): all events for user 7 go to the same parallel instance, which owns
user 7's state.

### 5.2 State stores

- **In-memory (heap) state:** fastest; limited by RAM; large heaps mean GC pauses.
- **Embedded RocksDB** (an LSM-tree, topic 19) on local disk: state much larger than memory; used by
  Flink's RocksDB state backend and Kafka Streams by default. Supports incremental snapshots because LSM
  SSTables are immutable.
- **Changelog / remote:** Kafka Streams backs each local store with a compacted **changelog topic**
  (topic 37); every state update is also written there, so the store can be rebuilt on another machine
  by replaying the changelog. (Standby replicas keep warm copies to speed failover.)

### 5.3 Checkpointing (Flink)

The core fault-tolerance problem: after a crash, restore the job so that the state and the input
positions are **consistent** — state reflects exactly the events up to the restored positions, no more,
no less.

Flink takes **periodic distributed snapshots** using an asynchronous variant of the **Chandy–Lamport**
algorithm:

```
sources (Kafka partitions)
   │ inject barrier n into each stream, recording the source offsets for checkpoint n
   ▼
 ┌──────┐   events…  │barrier n│  events…
 │ map  │──────────────────────────────►┐
 └──────┘                                ▼
 ┌──────┐                           ┌──────────┐  operator receives barrier n on ALL inputs
 │ map  │──────────────────────────►│ window   │  (aligning: buffers inputs whose barrier
 └──────┘                           │ (state)  │   arrived early) → snapshots its state
                                    └────┬─────┘  asynchronously → forwards barrier n
                                         ▼
                                       sink   → acks checkpoint n
                                         
JobManager: when every operator has acked n, checkpoint n is COMPLETE
            = {source offsets, every operator's state} — a consistent cut
```

1. The coordinator (JobManager) tells sources to inject **checkpoint barrier n** into their streams and
   record their current offsets.
2. Barriers flow with the data. An operator with multiple inputs waits until barrier n has arrived on
   **all** inputs (buffering events from inputs that are "ahead") — this is **barrier alignment**.
3. It then snapshots its state (asynchronously, to durable storage like S3/HDFS — often incrementally)
   and forwards the barrier.
4. When all operators (including sinks) have acknowledged, checkpoint n is complete.

On failure: stop the job, reload every operator's state from the last completed checkpoint, **rewind the
sources** to the recorded offsets, and continue. Events after the checkpoint are reprocessed, but since
state was rolled back to *before* them, each event affects state exactly once.

Notes:
- This needs a **replayable source** (Kafka, Kinesis, files). A source that can't rewind (a raw socket)
  can't give these guarantees.
- Checkpoint interval trades recovery time (how much is replayed) against overhead.
- **Unaligned checkpoints** avoid waiting during backpressure by including in-flight buffered data in the
  snapshot.
- **Savepoints** are user-triggered checkpoints used for upgrades, rescaling, and migrating jobs — stop
  with a savepoint, deploy new code, restore from it (state schema evolution applies, topic 16).
- Spark Structured Streaming uses **micro-batches** instead: each small batch is processed as a
  deterministic batch job with its offsets and state written to a write-ahead log/checkpoint — the batch
  fault-tolerance model (topic 40) applied every second or so.

### 5.4 Rescaling

Since state is partitioned by key, changing parallelism means redistributing state. Flink groups keys
into a fixed number of **key groups** (max parallelism); rescaling reassigns whole key groups to
instances — consistent-hashing-style indirection (topic 13) so state can move without rehashing every key.

---

## 6. Exactly-once in stream processors

### 6.1 Internal exactly-once

From section 5.3: replayable source + consistent checkpoint of offsets and state ⇒ **each event affects
the processor's internal state exactly once**, even though some events are *processed* more than once
after a failure (the "effectively-once" idea from topic 38).

### 6.2 End-to-end exactly-once: the sink problem

The output is the hard part. If the sink writes results to an external system immediately, and the job
then fails and rewinds to the last checkpoint, it re-emits those results — duplicates in the external
system. Two ways out:

**A. Idempotent sinks.** Make writing the same result twice harmless (topic 38): upsert by key (window
key + window start), versioned writes. Replays overwrite with identical values. Simple and very common
(e.g. writing window results to a key-value store or Elasticsearch by deterministic document ID).

**B. Transactional sinks (two-phase commit).** Flink's `TwoPhaseCommitSinkFunction` pattern:

```
between checkpoints:  sink writes output into an open transaction (e.g. Kafka transaction)
on checkpoint barrier: sink "pre-commits" (flush, prepare) as part of its snapshot
when checkpoint n completes everywhere: coordinator notifies → sink COMMITS the transaction
on failure before completion: transaction aborted; replay rewrites the output
```

Downstream readers in `read_committed` mode (topic 37) only see output from completed checkpoints. This is
2PC (topic 34) with the checkpoint as the decision. Cost: **output latency equals the checkpoint
interval**, since results become visible only on commit.

Kafka Streams achieves the same within Kafka using Kafka transactions (`exactly_once_v2`): input offsets,
state changelog writes and output records are committed atomically.

### 6.3 What's still not covered

Side effects that are neither idempotent nor transactional — sending an email or calling a third-party
API from inside the operator — may be repeated after a failure. Same lesson as topic 38: exactly-once
stops at the boundary of systems that participate.

Also, exactly-once requires **determinism** in processing: if an operator's output depends on processing
time, random numbers or external lookups, the replay after a failure may produce *different* output than
the first attempt — and with idempotent sinks, a different value may overwrite the first.

---

## 7. Lambda vs Kappa architecture

### 7.1 Lambda architecture

Proposed when stream processors were seen as fast but approximate/unreliable:

```
                    ┌──────────► batch layer (Hadoop/Spark): recompute everything
                    │            periodically from the master dataset → accurate "batch views"
events ──► master ──┤                                                        │
           dataset  │                                                        ▼
         (immutable)└──────────► speed layer (stream processor): recent      serving layer:
                                 events only → fast, approximate             merge batch view +
                                 "real-time views"                           real-time view at query
```

- **Batch layer:** periodically reprocesses all historical data → correct results, hours stale.
- **Speed layer:** processes only recent events → fresh, possibly approximate; its results are discarded
  once the batch layer catches up.
- **Serving layer:** answers queries by merging the two.

Good ideas in it: keep an immutable master copy of raw events; be able to recompute everything.
Problems:
- **Two codebases** implementing the same logic in two frameworks, which must produce matching results
  and drift apart over time. Frameworks that compile one program to both (Summingbird, later Beam) ease
  but don't remove this.
- Merging results at query time is complex for non-trivial aggregations.
- Operating two pipelines doubles the infrastructure.

### 7.2 Kappa architecture

Proposed by Jay Kreps (2014): **use only the stream processing layer.** Keep the full event history in a
replayable log (Kafka with long retention or tiered storage). To change logic or fix a bug:

```
log (full history) ──► job v1 ──► output table v1   (serving)
       │
       └──────────────► job v2 (new code), starting from offset 0
                         ──► output table v2   (catching up)

when v2 has caught up: switch readers to v2, stop v1, drop table v1
```

- One codebase, one engine.
- "Reprocessing" = running the same streaming job from the start of the log — which only works well with
  an engine that handles event time, watermarks, state and exactly-once correctly (Flink, Kafka Streams,
  Dataflow), so replayed results match what live processing would have produced.

Costs and limits:
- Retaining the full history in the log (storage; tiered storage makes it affordable).
- Reprocessing years of history through a streaming engine can be slower than a tuned batch job.
- Some computations are genuinely easier in batch (global sorts, complex ML training).

### 7.3 Where things stand

The industry trend has been toward **unified engines** that treat batch as bounded streams: Flink runs
both with one API; Beam defines one model with batch and streaming runners; Spark Structured Streaming
uses the same DataFrame API for both. Many organisations end up with a pragmatic mix: streaming for
fresh, incremental results; batch (or a replay job) for backfills and large historical recomputation —
but with shared logic, not two diverging implementations.

---

## 8. Failure modes worth knowing

| Problem | Symptom | Mitigation |
|---|---|---|
| Idle or slow source partition | Watermark stalls; windows never fire | Idleness timeouts; monitor watermark lag |
| Watermark too aggressive | Many late events dropped; counts too low | Increase bound; allowed lateness; side outputs |
| Hot key | One parallel instance overloaded; backpressure up the job | Pre-aggregate/salt keys; two-stage aggregation |
| State growth | Disk/memory exhaustion; slow checkpoints | State TTL; bounded windows; clean up sessions |
| Long checkpoint alignment under backpressure | Checkpoints time out; recovery time grows | Unaligned checkpoints; fix the bottleneck |
| Non-deterministic logic | Replays produce different results | Use event time and data in events; seed randomness |
| Schema change in state | Restore from savepoint fails | State schema evolution; compatible changes (topic 16) |
| Backpressure | Lag grows in source topic | Scale out (up to partitions), optimise slowest operator |

Backpressure in a stream processor is natural flow control (topic 44): a slow operator stops consuming
from its input buffers, which fill and slow down upstream operators, until the source simply reads Kafka
more slowly — lag grows, but nothing crashes or drops.

---

## 9. How it connects

- **Topic 27 (time):** event-time clocks are device clocks with skew; ordering vs timestamps.
- **Topic 34 (2PC):** transactional sinks commit via a two-phase protocol driven by checkpoints.
- **Topic 37 (Kafka):** the replayable, partitioned source and sink; compacted changelogs for state.
- **Topic 38 (delivery semantics):** effectively-once, idempotent sinks, determinism for replay.
- **Topic 39 (CDC):** change streams feed stream–table joins and materialised views.
- **Topic 40 (batch):** bounded vs unbounded; Lambda/Kappa; micro-batches.
- **Topic 42 (event sourcing/CQRS):** read models are stream processors folding events into state.
- **Topic 44 (backpressure):** built into the dataflow.
- **Topic 50 (probabilistic structures):** HyperLogLog / Count-Min sketches for approximate streaming
  aggregates in bounded memory.

---

## Common misconceptions

- **"Stream processing is just batch with small batches."** → Micro-batching is one implementation; the
  essential differences are unbounded input, event-time reasoning, incomplete windows and continuous state.
- **"Use the time the event arrives — it's close enough."** → Delays, outages and especially replays
  make processing time wildly different from event time; real-world metrics must use event time.
- **"A watermark guarantees no late data."** → It's a heuristic estimate; late data can always arrive
  and needs a policy (drop, update, side output).
- **"Exactly-once means each event is processed once."** → Events may be reprocessed after failures;
  the guarantee is that *state and committed output* reflect each event once.
- **"Exactly-once covers whatever the operator does."** → External side effects (emails, API calls) aren't
  rolled back by checkpoints.
- **"Kappa means no batch ever."** → It means one processing model with replay instead of a separate batch
  codebase; bounded reprocessing still happens.
- **"Stateful operators should keep state in a shared database."** → Local, key-partitioned state with
  checkpoints is faster and makes consistency with input offsets possible.

## Further reading (supplements)

- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 11 ("Stream Processing").
- Tyler Akidau, "Streaming 101" and "Streaming 102" (O'Reilly Radar, 2015), and the book *Streaming
  Systems* (Akidau, Chernyak, Lax, 2018).
- Akidau et al., "The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in
  Massive-Scale, Unbounded, Out-of-Order Data Processing" (VLDB 2015).
- Carbone et al., "Lightweight Asynchronous Snapshots for Distributed Dataflows" (2015) — Flink's
  checkpointing; and Chandy & Lamport, "Distributed Snapshots" (1985).
- Jay Kreps, "Questioning the Lambda Architecture" (O'Reilly Radar, 2014).
- Nathan Marz & James Warren, *Big Data* (Manning, 2015) — the Lambda architecture.
- Apache Flink documentation: "Timely Stream Processing", "Stateful Stream Processing", "Checkpointing".

---

## Questions

1. Why can't a stream job simply "wait until all the input has arrived" like a batch job? What does it
   do instead?
2. Explain the stream–table duality with a bank-account example. Where does it appear in Kafka?
3. A consumer computing "orders per minute" by processing time is down for 20 minutes, then catches up.
   What does the chart show, and how would event-time windowing differ?
4. Choose a window type for each: (a) hourly billing totals, (b) "p95 latency over the last 5 minutes,
   refreshed every 30 s", (c) "average number of pages per visit".
5. An event at 10:07 arrives in a hopping window of size 10 min and slide 5 min (windows aligned to
   :00). Which windows include it?
6. A user has two sessions (gap timeout 30 min): one with events from 10:00 to 10:10, another with events
   from 10:50 to 11:05. A late event with event time 10:30 arrives. What should happen?
7. What exactly does a watermark of 10:05 assert? Why might a job's watermark stop advancing entirely?
8. A team sets a 1-second bounded-out-of-orderness watermark for mobile app events. What symptom do they
   see and what are their options?
9. Why do stream processors keep state locally (e.g. RocksDB) rather than in a remote database?
10. Walk through how barrier-based checkpointing gives a consistent snapshot. Why must an operator with
    two inputs align barriers?
11. After a crash, a Flink job restores checkpoint 41 and reprocesses 30 s of events. Why doesn't the
    window count double-count those events? Why might the downstream database still see duplicates?
12. Compare an idempotent sink and a transactional sink. What latency cost does the transactional one pay?
13. An operator calls a third-party enrichment API and uses `now()` to decide if an offer has expired.
    What goes wrong on recovery and replay?
14. What problems of the Lambda architecture does Kappa solve, and what does it require from the log and
    the engine?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** The input is unbounded — it never ends — so "all input" never arrives. The job emits results
incrementally, using windows to scope aggregates and watermarks/triggers to decide when a window's result
is (probably) complete, with a policy for late data.

**2.** The table is the current balance per account; the stream is the sequence of deposits/withdrawals.
Folding the stream yields the table; capturing every table change yields the stream. In Kafka: a compacted
topic is a table's changelog; Kafka Streams' KStream vs KTable; state stores backed by changelog topics.

**3.** Normal counts, then 20 minutes of zero, then one minute with a huge spike as the backlog is
processed — an artefact of the outage. Event-time windows assign each order to the minute it happened,
showing steady counts; the windows for the outage period are just emitted late.

**4.** (a) Tumbling 1-hour windows. (b) Sliding/hopping window, size 5 min, slide 30 s. (c) Session
windows per user with an inactivity timeout, then average pages per session.

**5.** [10:00, 10:10) and [10:05, 10:15).

**6.** 10:30 is within 30 minutes of 10:10 and of 10:50, so it bridges the 40-minute gap: the two sessions
should merge into one session spanning 10:00–11:05 (if allowed lateness permits updating them), and the merged result
emitted as an update/retraction of the earlier ones. If beyond allowed lateness, it's dropped or sent to
a side output.

**7.** That the job believes no further events with event time ≤ 10:05 will arrive, so windows ending at
or before 10:05 can fire. It stalls if any input partition stops producing (idle) or lags, because an
operator's watermark is the minimum across its inputs.

**8.** Mobile events are often delayed by minutes or hours, so many arrive after their windows fired:
counts are too low and lots of late events are dropped. Options: larger bound (more latency), allowed
lateness with updated results, side output of late events for correction, or early/speculative firings
plus late refinements.

**9.** One remote round trip per event would cap throughput and add latency; and local state can be
snapshotted together with input offsets to give consistent recovery and exactly-once. State is
key-partitioned like the input so each instance owns its keys.

**10.** Sources inject barrier n and record offsets; barriers flow in-line with events, splitting the
stream into "before n" and "after n". Each operator snapshots its state when barrier n passes it, so the
snapshot reflects exactly the pre-barrier events. An operator with two inputs must wait for barrier n on
both (buffering the faster one) — otherwise its state would include post-barrier events from one input,
making the snapshot inconsistent with the recorded source offsets.

**11.** State was restored to checkpoint 41, which reflects only events up to the checkpoint's offsets;
the source rewinds to those offsets, so each replayed event is applied to state that didn't include it.
But if the sink wrote results to an external DB during those 30 s, they're written again on replay —
duplicates unless the sink is idempotent or transactional.

**12.** Idempotent: writes are keyed/versioned so re-writes are harmless; results visible immediately;
needs a key-addressable store and deterministic outputs. Transactional: output is written inside a
transaction committed only when the checkpoint completes (2PC); works with append-only sinks like Kafka,
but output becomes visible only at checkpoint completion — latency ≈ checkpoint interval.

**13.** On recovery/replay, the API may return different data and `now()` is later, so replayed events
produce different outputs than the first attempt (e.g. offers wrongly marked expired) — non-deterministic,
breaking exactly-once and making idempotent overwrites change values. The API call may also be repeated.
Fix: use event time; enrich via a stream–table join on a changelog of the reference data.

**14.** Kappa removes the duplicate batch codebase, the drift between the two implementations, and the
query-time merging; reprocessing is just replaying the log with new code into a new output and switching.
It requires the log to retain full history (long retention/tiered storage, or compaction where
appropriate) and an engine with correct event-time semantics, state and exactly-once so replayed results
equal live results.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Defining difference of stream vs batch?
**A:** The input is unbounded — never complete.

**Q:** Stream–table duality?
**A:** Folding a stream gives a table; capturing a table's changes gives a stream.

**Q:** Event time vs processing time?
**A:** When it happened (in the event) vs when the processor handles it.

**Q:** When is processing time OK?
**A:** Monitoring the processor itself, or when approximate low-latency results suffice.

**Q:** Why does replay break processing-time windows?
**A:** Days of events get processing times within hours — spikes and gaps that never happened.

**Q:** Tumbling window?
**A:** Fixed size, non-overlapping; each event in exactly one window.

**Q:** Sliding/hopping window?
**A:** Fixed size, starts every slide interval; overlapping; event in several windows.

**Q:** Session window?
**A:** Per key, closed by an inactivity gap; variable length; can merge.

**Q:** Stream–table join needs?
**A:** A local copy of the table, kept updated by a changelog/CDC stream.

**Q:** What is a watermark?
**A:** An assertion that no more events with event time ≤ T are expected.

**Q:** Watermark with parallel inputs?
**A:** The minimum of the input watermarks — idle/slow partitions hold it back.

**Q:** Watermark trade-off?
**A:** Aggressive = low latency, more late data; conservative = complete, slower.

**Q:** Options for late events?
**A:** Drop, update results within allowed lateness, or side-output them.

**Q:** Why local state?
**A:** Speed, and it can be checkpointed consistently with input offsets.

**Q:** Common local state store?
**A:** RocksDB (LSM-tree) on local disk.

**Q:** How does Kafka Streams make state fault-tolerant?
**A:** Each store is backed by a compacted changelog topic.

**Q:** Flink checkpoint mechanism?
**A:** Barriers injected at sources; operators snapshot state when barriers pass (Chandy–Lamport style).

**Q:** What is barrier alignment?
**A:** Waiting for barrier n on all inputs before snapshotting, buffering faster inputs.

**Q:** Recovery from a checkpoint?
**A:** Restore all state, rewind sources to the checkpoint's offsets, reprocess.

**Q:** Source requirement for exactly-once?
**A:** It must be replayable (e.g. Kafka offsets).

**Q:** What is a savepoint?
**A:** A manual checkpoint for upgrades, rescaling and migration.

**Q:** Two ways to get end-to-end exactly-once at the sink?
**A:** Idempotent sinks, or transactional (2PC) sinks committed on checkpoint completion.

**Q:** Cost of transactional sinks?
**A:** Output visible only at checkpoint completion — latency ≈ checkpoint interval.

**Q:** Lambda architecture?
**A:** Batch layer (accurate) + speed layer (fresh) merged in a serving layer.

**Q:** Lambda's main problem?
**A:** Two codebases implementing the same logic, drifting apart; complex merging.

**Q:** Kappa architecture?
**A:** Only streaming; reprocess by replaying the log into a new output and switching.

**Q:** How does a stream processor handle backpressure?
**A:** Slow operators stop consuming; buffers fill upstream; source reads slower; lag grows.
