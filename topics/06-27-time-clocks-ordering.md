# 27 — Time, Clocks & Ordering

> Part 6 — Distributed Systems · Prerequisites: topics 23, 24, 26

Programs use time constantly: to order events ("which write came last?"), to measure durations
("has this request taken more than 2 s?"), to expire things ("this lease is valid until 12:00:30"), and
to stamp records ("created_at"). On one machine, a single clock serves all of these reasonably well. In a
distributed system, every machine has its own clock, those clocks disagree, and they can jump forwards
and backwards. Using wall-clock timestamps to decide the order of events across machines is one of the
most common ways distributed systems silently lose data. This topic explains what clocks can and can't
tell you, and introduces the *logical* clocks — Lamport clocks, vector clocks, hybrid logical clocks —
that capture ordering correctly without trusting physical time, plus Google's TrueTime, which makes
physical time trustworthy by being honest about its uncertainty.

By the end of this topic you should be able to:
1. Distinguish wall-clock (time-of-day) clocks from monotonic clocks and say which to use for what.
2. Explain how NTP synchronizes clocks, how much skew remains, and the ways clocks go wrong.
3. Show with a timeline how last-write-wins with wall-clock timestamps loses data.
4. Define the happens-before relation and concurrency, and explain why causality is the ordering that matters.
5. Run Lamport clocks and vector clocks by hand, and explain what each can and can't tell you.
6. Explain hybrid logical clocks, and how TrueTime plus commit-wait gives Spanner externally consistent timestamps.

---

## 1. Two kinds of physical clock

Every modern computer has a hardware clock — usually a quartz crystal oscillator — plus OS software that
turns it into time values. Operating systems expose two very different kinds of clock.

### 1.1 Time-of-day (wall) clocks

A **wall clock** returns the current date and time according to some calendar: e.g. `clock_gettime(
CLOCK_REALTIME)` on Linux, `System.currentTimeMillis()` in Java, `Date.now()` in JavaScript. Usually the
number of seconds (or ms) since the Unix epoch, 1970-01-01 00:00:00 UTC.

Wall clocks are meant to agree with the rest of the world, so they are **adjusted**:

- **NTP corrections** can step the clock — move it forward or *backward* abruptly — if it has drifted
  far from the reference.
- **Leap seconds**: UTC occasionally inserts an extra second (23:59:60). Systems handle this badly:
  the 2012 leap second caused livelocks in Linux kernels and outages at several large sites. Many
  providers now "smear" the leap second over hours instead. (Leap seconds are scheduled to be abolished
  by 2035, but the lesson about clock adjustments stands.)
- **Manual changes**, VM migrations, or a misconfigured server.

Consequence: **wall clock time can go backwards**, and the difference between two readings is not a
reliable duration. Also, wall clocks often have coarse resolution on some platforms.

### 1.2 Monotonic clocks

A **monotonic clock** (`CLOCK_MONOTONIC`, Java's `System.nanoTime()`, `performance.now()`) only
promises to move forward. Its absolute value is meaningless — it might be nanoseconds since boot — so
comparing monotonic values *between two machines* is meaningless too. But the difference between two
readings on the same machine is a good measure of elapsed time.

NTP may still *slew* a monotonic clock — speed it up or slow it down by up to ~0.05% — if the local
oscillator runs fast or slow, but it never jumps.

### 1.3 Which to use

| Purpose | Use |
|---|---|
| Measuring a duration (timeout, latency, rate limiting) | Monotonic |
| Deadline for a local wait ("give up after 2 s") | Monotonic |
| Showing a time to a human, logging, `created_at` | Wall clock (accept inaccuracy) |
| Ordering events across machines | **Neither** — use logical clocks (or TrueTime-like bounded clocks) |
| Lease expiry across machines | Wall clock + a safety margin, plus fencing (topic 31) |

A classic bug: computing a timeout as `end = now_wall() + 5s` and looping until `now_wall() > end`. If NTP
steps the clock back by a minute, the wait lasts a minute longer; stepped forward, it ends instantly.

---

## 2. Clock synchronization and its limits

### 2.1 How NTP works

Quartz oscillators drift: their frequency varies with temperature and manufacturing. A typical drift of
~10–200 parts per million (ppm) means **up to ~17 ms per day at 200 ppm — or seconds per day for poor
hardware**. To keep wall clocks close to real time, machines run the **Network Time Protocol (NTP)**
against time servers, which ultimately get time from accurate sources (atomic clocks, GPS receivers).
Servers are arranged in strata: stratum 0 is the reference clock, stratum 1 servers are attached to it,
and so on.

NTP estimates the offset between the local clock and a server:

```
  client                         server
   t0 ──── request ────────────▶ t1  (server receives)
   t3 ◀─── reply ─────────────── t2  (server sends)

  round-trip delay   δ = (t3 − t0) − (t2 − t1)
  clock offset       θ = ((t1 − t0) + (t2 − t3)) / 2
```

The offset formula assumes the outbound and return trips take equal time. If they don't (asymmetric
routes, one direction congested), the estimate is wrong by up to half the asymmetry. **The accuracy of
NTP is fundamentally limited by network delay variance** — the very thing topic 26 said is unbounded.

### 2.2 How far off are clocks in practice?

- Well-run datacenter NTP: often within a millisecond or a few ms between servers, but with outliers.
- Over the public internet: tens of ms, sometimes ~100 ms or more during congestion.
- With dedicated hardware (GPS receivers per datacenter, Precision Time Protocol (PTP) with hardware
  timestamping, or cloud services like Amazon Time Sync / clock-bound): microseconds.

Ways it goes wrong, all observed in practice:

- The NTP daemon is misconfigured or a firewall blocks NTP traffic — clocks drift for weeks unnoticed.
- The local clock is too far from the server; NTP refuses to sync or steps it abruptly.
- A node syncs against a wrong upstream server, and adopts its wrong time.
- VM pauses (topic 26): from inside the VM, the clock suddenly jumps forward.
- Users' devices (phones, laptops) have arbitrarily wrong clocks — never trust a client timestamp.

The crucial problem: **a skewed clock looks normal.** Nothing crashes. The program reads a plausible
timestamp and keeps going, silently making wrong decisions. If you depend on synchronized clocks, you
must *monitor* the offset between nodes and remove nodes that drift too far.

### 2.3 Clock readings have a confidence interval

Even a well-synced clock is not a point; it's an estimate with an error bar. If the last NTP sync was
against a server with 5 ms round-trip, and drift since then is bounded, the true time is somewhere in
`[now − ε, now + ε]`. Most APIs hide ε and return a single number with microsecond precision —
**precision is not accuracy**. TrueTime (section 7) is notable precisely because it exposes ε.

---

## 3. Why timestamps can't order events across machines

### 3.1 Last-write-wins with wall clocks

Many replicated databases resolve concurrent writes with **last write wins (LWW)**: each write carries a
timestamp, and replicas keep the value with the highest one (Cassandra does this by default; many
multi-leader setups too — topic 24, topic 33). It sounds reasonable. Here's how it loses data:

```
 real time ─────────────────────────────────────────────────────────▶

 Node 1 (clock accurate)           Node 2 (clock 3 ms behind)
   write x=1, ts = 42.004
        │  replicates ─────────────────▶ receives x=1 @42.004, stores it
        │
        │                          client (which READ x=1 from node 2)
        │                          writes x=2 on node 2, ts = 42.003
        │                          (node 2's clock says 42.003!)
        │   ◀─────── replicates x=2 @42.003
   compares 42.003 < 42.004
   → DISCARDS x=2
   final value everywhere: x = 1
```

The second write happened *after* the first — in fact it was *caused* by it (the client read x=1 and
then wrote x=2). Yet it is silently dropped, because node 2's clock was 3 ms slow. No error is reported.
Every replica converges — to the wrong value.

General problems with LWW + physical timestamps:

- **A node with a fast clock "wins" everything** until real time catches up; writes from nodes with
  slower clocks disappear.
- **Causally later writes can lose** to earlier ones, as above.
- **LWW cannot distinguish "sequential" from "concurrent"**: two clients that wrote at nearly the same
  time without seeing each other's writes — truly concurrent — have one write discarded arbitrarily.
  Concurrency should be *detected*, not resolved by coin-flip (topic 33).
- Ties (identical timestamps, more likely with ms resolution) need a tie-breaker, e.g. node ID, which
  is equally arbitrary.

Could you fix it with better clocks? Narrower skew reduces the window, but as long as skew is non-zero
there's a window in which a causally later write gets an earlier timestamp. You'd need skew smaller than
the time between causally related events — which, for a client reading and immediately writing,
could be microseconds.

### 3.2 Other timestamp-ordering hazards

- **Snapshot isolation across nodes.** Using a wall-clock timestamp as a transaction ID ("this
  transaction sees all writes with ts < mine") on multiple nodes means a node with a slow clock can
  create a snapshot that misses already-committed data. (Spanner solves this with TrueTime, section 7.)
- **Log merging.** Interleaving logs from several machines by timestamp can show effects before causes,
  which makes debugging misleading.
- **Ordering of messages.** "Message with the lower timestamp was sent first" is false across senders.

So we need a different notion of order — one based on what *could have influenced* what, not on what
clocks say.

---

## 4. Happens-before and causality

### 4.1 The happens-before relation

Leslie Lamport's 1978 paper "Time, Clocks, and the Ordering of Events in a Distributed System" defined
the relation **happens-before**, written `a → b`. It's the smallest relation such that:

1. If `a` and `b` are events in the **same process** and `a` occurs before `b` in that process's
   execution, then `a → b`.
2. If `a` is the **sending** of a message and `b` is the **receipt** of that message, then `a → b`.
3. **Transitivity:** if `a → b` and `b → c`, then `a → c`.

If neither `a → b` nor `b → a`, the events are **concurrent**, written `a ∥ b`.

```
 P1:  a ────── b ──────────────────── c
                \                    ↑
                 \ m1                │ m2
                  ↘                  │
 P2:  d ─────────── e ────── f ──────┘
                                \
 P3:  g ──────────── h            ↘ (m3) i

 a → b (same process)          b → e (message m1)
 a → e (transitivity)          f → c (message m2)
 a → c, d → c (transitivity)   g ∥ a, g ∥ e, h ∥ b ...  (no path between them)
```

### 4.2 Why "happened before" means "could have influenced"

`a → b` means there's a chain of steps and messages along which information from `a` could have reached
`b`. Concurrency `a ∥ b` means neither *could* have known about the other — not that they happened at
the same instant. Two events a second apart on different machines with no messages between them are
concurrent; two events a microsecond apart in the same process are ordered.

This is **causality**, and it's the ordering that matters for correctness:

- A reply should never be visible before the question it answers.
- A write that was based on reading a value should win over the value it read.
- A "delete user" should not be applied before the "create user" that it depends on.

Physical time order is consistent with causality when clocks are perfect (causes precede effects in
real time), but with imperfect clocks, physical timestamps can *contradict* causality — exactly the
LWW bug. Logical clocks instead derive timestamps from causality itself.

### 4.3 Causal vs total order

- A **causal order** is a *partial* order: some pairs of events are ordered, others (concurrent ones)
  are not.
- A **total order** orders every pair. Any total order that agrees with causality is a valid
  "linearization" of the partial order; there are usually many.

Some problems need only causal order (topic 28: causal consistency). Others need a single total order
everyone agrees on — e.g. "which of two concurrent requests to claim username `alice` wins?" — which
requires consensus (topic 30).

---

## 5. Lamport clocks

### 5.1 The algorithm

A **Lamport clock** is a simple counter per process that produces timestamps consistent with
happens-before.

```
 each process keeps an integer counter C, initially 0

 on a local event:       C = C + 1;  stamp event with C
 on sending message m:   C = C + 1;  attach C to m
 on receiving m with t:  C = max(C, t) + 1;  stamp receive event with C
```

Each timestamp is usually paired with the process ID — `(C, node_id)` — to break ties, giving a total
order: compare counters first, then node IDs.

### 5.2 Worked example

```
 P1:  (1) ──── (2) send ────────────────────────────── (5) recv
                    \                                 ↗
                     \ t=2                      t=4  /
                      ↘                             /
 P2:  (1) ─────────── (3) recv ──── (4) send ──────
 
 P3:  (1) ─── (2) ─── (3)   (no messages)
```

P2's receive: `max(1, 2) + 1 = 3`. P1's receive: `max(2, 4) + 1 = 5`.

### 5.3 What Lamport clocks guarantee — and what they don't

**Guarantee (clock condition):** if `a → b`, then `L(a) < L(b)`.

**Not guaranteed:** the converse. `L(a) < L(b)` does **not** imply `a → b`. In the example, P3's event
with timestamp 2 and P2's event with timestamp 3 are concurrent, yet 2 < 3. Looking at two Lamport
timestamps, you can't tell whether one event caused the other or they were concurrent.

So Lamport timestamps give you a **total order consistent with causality**, which is very useful:

- LWW with Lamport timestamps instead of wall clocks fixes the bug in section 3.1: the client's write
  happened after reading x=1, so (if the client carries the timestamp it read) its write gets a higher
  timestamp and wins. Causally later writes always win.
- Lamport's paper used them to implement a fair distributed mutual exclusion.

But they **can't detect concurrency** — two concurrent writes get ordered arbitrarily and one is
discarded, just like with wall clocks. And the total order is only known *after the fact*: a node
can't know at the moment it assigns a timestamp whether some other node has, or will have, assigned a
lower one. So you can't use Lamport timestamps alone to decide "I'm the first to claim username alice"
right now — you'd have to hear from every other node first. That's why uniqueness constraints need
total order broadcast / consensus (topic 30).

Also note Lamport timestamps have **no relation to physical time**; a quiet node's counter can be far
behind, so they can't answer "what happened around 14:05?".

---

## 6. Vector clocks and version vectors

### 6.1 The algorithm

To detect concurrency, each process tracks not one counter but a **vector** — one counter per process —
recording how many events from each process it "knows about".

```
 with n processes, each process i keeps vector V[1..n], all zeros initially

 on a local event at i:     V[i] = V[i] + 1
 on sending from i:         V[i] = V[i] + 1;  attach a copy of V
 on receiving vector T at i: V[j] = max(V[j], T[j]) for all j;  then V[i] = V[i] + 1
```

Comparison:

- `V ≤ W` iff `V[j] ≤ W[j]` for every j.
- `V < W` (V happened before W) iff `V ≤ W` and `V ≠ W`.
- `V ∥ W` (concurrent) iff neither `V ≤ W` nor `W ≤ V`.

**Guarantee:** `a → b` **if and only if** `V(a) < V(b)`. Vector clocks characterize causality exactly.

### 6.2 Worked example

Three processes A, B, C; vectors written `[A,B,C]`.

```
 A: [1,0,0] ──send──▶                                  
                     \                                  
 B:                   [1,1,0] recv ── [1,2,0] send ──▶ 
                                                     \   
 C: [0,0,1] ────────────────────────────────────────── [1,2,2] recv

 Compare A's [1,0,0] and C's first event [0,0,1]:
   neither ≤ the other → concurrent.  ✔ (no message path between them)
 Compare A's [1,0,0] and C's [1,2,2]:
   [1,0,0] < [1,2,2] → A's event happened before.  ✔ (via B)
```

### 6.3 Version vectors in replicated data

In replicated databases the same idea is applied to **data** rather than processes, under the name
**version vector**: each replica keeps a counter, and each stored value carries a vector saying which
updates from which replica it incorporates. When two versions of a key meet:

- If one vector dominates the other, the dominating version is newer — discard the older one safely.
- If they're concurrent, **both writes must be kept** as *siblings* and merged — by the application
  (the Dynamo shopping-cart union) or by a CRDT (topic 33).

Riak used version vectors (with "dotted version vectors" as a refinement); Amazon's Dynamo paper used
vector clocks for this. This is how a leaderless store (topic 24) can avoid silently losing concurrent
writes, unlike LWW.

### 6.4 Costs

- **Size grows with the number of participants.** Fine for N replicas of a key (3–5 entries). Bad if
  every *client* is an entry — the vector can grow without bound. Systems use server-side vectors,
  prune old entries (risking false "concurrent" verdicts), or use dotted version vectors.
- Comparing and merging vectors costs CPU and metadata bytes on every value.
- Membership changes (adding replicas) complicate vector layout.

### 6.5 Summary: which logical clock for what

| | Lamport clock | Vector clock / version vector |
|---|---|---|
| Size | One integer | One integer per participant |
| `a → b` ⇒ `T(a) < T(b)` | Yes | Yes |
| `T(a) < T(b)` ⇒ `a → b` | **No** | Yes |
| Detects concurrency | No | Yes |
| Gives a total order | Yes (with tie-breaker) | No (partial order) |
| Typical use | Ordering ops in logs, LWW that respects causality, total-order mechanisms | Conflict detection in replicated data, causal consistency |

---

## 7. Bringing physical time back: HLC and TrueTime

Logical clocks are correct but lose any connection to real time. Sometimes you want both: timestamps
that respect causality *and* stay close to wall-clock time, so you can ask for "a snapshot as of
14:05:00" or expire data by age.

### 7.1 Hybrid logical clocks (HLC)

A **hybrid logical clock** (Kulkarni et al., 2014) is a timestamp `(l, c)` where `l` tracks the largest
physical time seen so far and `c` is a small logical counter used when physical time doesn't move
forward enough.

```
 pt = local physical (wall) clock reading

 on local event or send:
     l_old = l
     l = max(l_old, pt)
     c = (l == l_old) ? c + 1 : 0

 on receive message with (l_m, c_m):
     l_old = l
     l = max(l_old, l_m, pt)
     if l == l_old == l_m:  c = max(c, c_m) + 1
     elif l == l_old:       c = c + 1
     elif l == l_m:         c = c_m + 1
     else:                  c = 0
```

Properties:

- If `a → b`, then `HLC(a) < HLC(b)` — like a Lamport clock.
- `l` is always close to physical time — it never runs ahead of the maximum physical clock in the
  system — so HLC timestamps can be used where people expect wall-clock-ish times.
- Fits in 64 bits (e.g. 48 bits of ms + 16 bits counter), so it can replace a timestamp field.

HLC does **not** remove clock skew: if one node's clock runs 200 ms fast, other nodes' HLCs get pulled
200 ms ahead when they hear from it. Systems like CockroachDB and YugabyteDB use HLC and additionally
enforce a **maximum clock offset**: nodes that drift beyond it shut themselves down, and reads
that hit a value with a timestamp in the "uncertainty window" just above their own may need to restart
at a higher timestamp. MongoDB uses a similar cluster-time scheme for causal sessions.

### 7.2 Google Spanner's TrueTime

Spanner (Google's globally distributed SQL database) wanted to assign each transaction a timestamp so
that the timestamp order matches real-time order: if T1 commits before T2 starts (in real time,
anywhere in the world), then `ts(T1) < ts(T2)`. This property is called **external consistency**
(essentially strict serializability, topic 28). It lets Spanner offer lock-free, globally consistent
snapshot reads at any timestamp.

The trick is to stop pretending clock readings are exact. Every datacenter has GPS receivers and atomic
clocks; time daemons sync against them frequently. The **TrueTime API** returns an *interval*:

```
 TT.now()  →  [earliest, latest]
 guarantee: the true absolute time is somewhere within the interval
 typical width 2ε: a few milliseconds (ε ≈ 1–7 ms in the original paper)
```

### 7.3 Commit-wait

When a transaction commits, Spanner:

1. Picks commit timestamp `s = TT.now().latest` (≥ the true current time).
2. **Waits** until `TT.now().earliest > s` — i.e. until it's certain that true time has passed `s`.
   This takes about 2ε.
3. Only then makes the commit visible and acknowledges the client.

```
  TT.now() at commit:     [ earliest ........ latest ]
                                               ↑ s chosen here
  ...wait ~2ε...
  TT.now() later:                                  [ earliest ........ latest ]
                                                     ↑ now earliest > s  → release
```

Why it works: any transaction T2 that starts after T1's commit is acknowledged starts at a true time
after `s`. T2's own timestamp is at least `TT.now().latest` at its commit, which is ≥ true time > s.
So `ts(T2) > ts(T1)` — timestamp order matches real-time order **even though clocks disagree**, because
each node accounts for its uncertainty and *waits it out*.

The cost is latency: every read-write transaction waits ~2ε before completing (overlapped with Paxos
replication, which takes time anyway). That's why Google invested in GPS and atomic clocks — to shrink ε.
The general lesson: **with bounded, known clock uncertainty you can use physical time for ordering, as
long as you wait out the uncertainty.** Without a reliable bound, you can't. (Topic 34 shows how Spanner
combines this with 2PC and Paxos.)

CockroachDB, lacking specialized hardware, takes the HLC + max-offset approach instead and gets a
slightly weaker guarantee (serializable, with protection against most anomalies, but not full external
consistency for unrelated transactions without extra waiting). AWS has since offered bounded-error clock
services (e.g. ClockBound) that expose a TrueTime-like interval.

### 7.4 Summary of approaches

| Approach | Respects causality? | Close to real time? | Detects concurrency? | Needs special hardware? |
|---|---|---|---|---|
| Wall-clock timestamps | **No** | Yes (± skew) | No | No |
| Lamport clock | Yes | No | No | No |
| Vector clock | Yes (exactly) | No | Yes | No |
| HLC | Yes | Yes (± skew) | No | No |
| TrueTime + commit-wait | Yes, and matches real-time order | Yes (± ε, known) | No | GPS/atomic clocks (or a bounded-error clock service) |

---

## 8. How it connects

- **Topic 26**: clocks are another component a node can't fully trust; VM pauses make clocks jump.
- **Topic 21–22**: snapshot isolation needs a notion of "which writes are before my snapshot"; across
  machines that requires logical or bounded timestamps.
- **Topic 23**: replication-lag anomalies are causality violations (seeing an effect before its cause).
- **Topic 24**: leaderless stores need version vectors to detect concurrent writes instead of LWW.
- **Topic 28** (next): causal consistency is built directly on happens-before; linearizability relates
  to real-time order.
- **Topic 30**: total order broadcast / consensus gives the single agreed order that Lamport clocks
  can't give in real time.
- **Topic 31**: leases depend on bounded clock error; fencing tokens are a logical counter.
- **Topic 33**: LWW, siblings and CRDTs — what to do once concurrency is detected.
- **Topic 34**: Spanner's 2PC + Paxos + TrueTime.
- **Topic 45**: distributed tracing propagates causality across services (trace/span IDs).

---

## Common misconceptions

- **"`System.currentTimeMillis()` is fine for measuring how long something took."** → Wall clocks can
  jump; use a monotonic clock for durations.
- **"NTP keeps all our servers in sync."** → To within ms on a good day; drift, misconfiguration,
  network asymmetry and VM pauses produce larger errors, and a wrong clock looks normal.
- **"Last-write-wins keeps the latest write."** → It keeps the write with the highest *timestamp*,
  which with skewed clocks may be an older one, and it silently drops concurrent writes.
- **"Concurrent means at the same time."** → Concurrent means neither event knew about the other; they
  can be seconds apart.
- **"A higher Lamport timestamp means the event caused/came after the other."** → Only the converse
  holds: if a → b then L(a) < L(b). Lower timestamp doesn't imply causality.
- **"Vector clocks give a total order."** → They give a partial order; concurrent events remain
  unordered, which is the point.
- **"HLC eliminates clock skew problems."** → It makes timestamps causality-respecting, but systems
  still need a bounded max offset for snapshot correctness.
- **"Spanner works because Google's clocks are perfect."** → It works because the clock uncertainty is
  *known and bounded*, and Spanner waits it out (commit-wait).

---

## Further reading (supplements)

- DDIA, Chapter 8 "Unreliable Clocks"; Chapter 9 "Ordering Guarantees" (Lamport timestamps, total order broadcast).
- Leslie Lamport, "Time, Clocks, and the Ordering of Events in a Distributed System" (1978).
- Corbett et al., "Spanner: Google's Globally-Distributed Database" (OSDI 2012).
- Kulkarni, Demirbas et al., "Logical Physical Clocks and Consistent Snapshots in Globally Distributed Databases" (2014) — the HLC paper.
- Kleppmann's Cambridge lecture series, lectures 3.1–3.3 (physical time, logical time).
- Kyle Kingsbury's Jepsen analyses of Cassandra (LWW and clock-skew data loss).

---

## Questions

1. Why should you measure a request's duration with a monotonic clock rather than a wall clock? Give a
   concrete failure.
2. Why is NTP's accuracy fundamentally limited by the network? What assumption in its offset
   calculation breaks?
3. Using a timeline, show how LWW with wall-clock timestamps can discard a write that was causally
   after the write that survives.
4. Two events occurred 3 seconds apart on different machines. Are they necessarily ordered by
   happens-before? What would make them ordered?
5. Compute Lamport timestamps: P1 has a local event, then sends m to P2. P2 had 3 local events before
   receiving m, then sends m' to P1. What are the timestamps of each event?
6. If `L(a) = 5` and `L(b) = 9`, what can you conclude? What if `V(a) = [2,1,0]` and `V(b) = [1,3,0]`?
7. Why can't Lamport timestamps alone implement "the first user to register a username wins" at the
   moment of registration?
8. What does a version vector let a leaderless database do that LWW can't? What does the database do
   with two concurrent versions?
9. Why don't vector clocks scale to a system where every client is a participant?
10. What problem do hybrid logical clocks solve that Lamport clocks don't? What problem do they not solve?
11. Explain commit-wait. Why does it guarantee that a transaction starting after T1 commits gets a larger
    timestamp?
12. If TrueTime's ε doubled, what would happen to Spanner's write latency and why?
13. A system merges logs from 20 servers by timestamp to debug an incident. Why might the merged log
    show an effect before its cause, and what would fix it?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Wall clocks can be stepped by NTP, adjusted for leap seconds or changed manually. If NTP steps the
clock back by 2 s during a 50 ms request, the computed duration is negative; stepped forward, it looks
like a 2-second request and may trigger a timeout or an alert. Monotonic clocks never jump, so the
difference of two readings is a valid elapsed time.

**2.** NTP computes offset assuming the request and reply take equal time. With asymmetric or variable
delays — queueing in one direction — the offset estimate is off by up to half the asymmetry. Since
network delay is unbounded and variable (topic 26), the uncertainty can't be reduced below the
variance of the round trip.

**3.** Node 1 (accurate) writes x=1 at ts 100.004 and replicates to node 2. A client reads x=1 from
node 2 and writes x=2 on node 2, whose clock lags by 3 ms and assigns ts 100.003. When replicas compare,
100.003 < 100.004 so x=2 is discarded everywhere, even though it was causally after x=1 (it was based on
reading x=1). No error is raised.

**4.** No. Happens-before is about information flow, not elapsed time. They're ordered only if there's a
chain of process steps and messages from one to the other (e.g. machine A sent a message that machine B
received before its event). Otherwise they're concurrent no matter how far apart.

**5.** P1: local event = 1, send m = 2. P2: local events 1, 2, 3; receive m: max(3, 2) + 1 = 4; send m'
= 5. P1 receives m': max(2, 5) + 1 = 6.

**6.** From Lamport timestamps 5 < 9: only that b did not happen before a; a might have happened before
b, or they might be concurrent. From vectors: [2,1,0] vs [1,3,0] — first component 2 > 1, second 1 < 3
— neither dominates, so they're definitely concurrent.

**7.** Lamport timestamps give a total order, but only after the fact: when a node assigns timestamp
(10, nodeA) to a registration, another node may concurrently have assigned (9, nodeB) to a conflicting
one that it hasn't heard about yet. To be sure it's first, nodeA would have to wait to hear from every
other node — and if one is unreachable, it can't decide. Deciding immediately requires total order
broadcast / consensus.

**8.** It can tell whether one version supersedes another (vector dominates) or whether they were
written concurrently (neither dominates). LWW would silently drop one concurrent write. With version
vectors the database keeps both concurrent versions as siblings and returns them to the reader (or a
CRDT merge function) to merge, then writes back a version whose vector dominates both.

**9.** The vector has one entry per participant. With millions of clients, every value would carry a
huge vector, and new clients add entries forever. Systems keep vectors per server replica, use dotted
version vectors, or prune old entries — pruning risks mistaking an ordered pair for concurrent.

**10.** HLC timestamps respect causality like Lamport clocks but also stay close to physical time, so
they can be used for time-based snapshots, TTLs and human-meaningful queries, in 64 bits. They don't
remove skew: a node with a fast clock drags others' HLCs ahead, and they can't guarantee real-time
ordering of unrelated transactions without a bounded maximum offset and uncertainty handling.

**11.** T1 picks s = TT.now().latest and waits until TT.now().earliest > s before acknowledging, so the
true time at acknowledgement is > s. A transaction T2 that starts after that acknowledgement starts at a
true time > s, and its commit timestamp is TT.now().latest at its commit, which is ≥ the true time then,
hence > s. So ts(T2) > ts(T1), matching real-time order.

**12.** Commit-wait lasts about 2ε, so each read-write transaction's latency increases by roughly the
added 2ε (unless hidden behind replication time). Also, timestamped reads might need to wait longer to be
safe. That's why Spanner invests in GPS/atomic clocks to keep ε small.

**13.** Each server stamps lines with its own wall clock; skew of even a few ms between servers can put a
receiver's "got request" line before the sender's "sent request" line. Fix: propagate causal metadata —
Lamport/HLC timestamps or trace and span IDs (distributed tracing) — and order by that rather than by
local timestamps.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Wall clock vs monotonic clock?
**A:** Wall clock = calendar time, can jump backward/forward. Monotonic = only moves forward, good for durations, meaningless across machines.

**Q:** Which clock for timeouts and latency measurement?
**A:** Monotonic.

**Q:** What does NTP assume when computing offset?
**A:** That outbound and return network delays are equal.

**Q:** Typical quartz drift?
**A:** Tens to ~200 ppm — milliseconds to tens of ms per day, or worse.

**Q:** Why is clock skew especially dangerous?
**A:** A wrong clock looks normal — nothing crashes; decisions are silently wrong.

**Q:** How does LWW with wall clocks lose data?
**A:** A causally later write on a node with a slow clock gets a lower timestamp and is discarded; concurrent writes are dropped arbitrarily.

**Q:** Three rules of happens-before?
**A:** Same-process order; send → receive; transitivity.

**Q:** What does "concurrent" mean in distributed systems?
**A:** Neither event happened before the other — neither could have known about the other.

**Q:** Lamport clock update on receive?
**A:** C = max(C, received) + 1.

**Q:** Lamport clock guarantee?
**A:** If a → b then L(a) < L(b) (not the converse).

**Q:** Can Lamport timestamps detect concurrency?
**A:** No.

**Q:** How do Lamport timestamps become a total order?
**A:** Pair with node ID as tie-breaker: (counter, node_id).

**Q:** Why can't Lamport timestamps enforce uniqueness in real time?
**A:** You only know your timestamp is lowest after hearing from all nodes — need consensus instead.

**Q:** Vector clock guarantee?
**A:** a → b if and only if V(a) < V(b); exact characterization of causality.

**Q:** When are two vectors concurrent?
**A:** When neither is ≤ the other component-wise.

**Q:** What do version vectors enable in replicated stores?
**A:** Detecting concurrent writes and keeping them as siblings instead of dropping one.

**Q:** Main cost of vector clocks?
**A:** Size grows with number of participants.

**Q:** What is a hybrid logical clock?
**A:** (physical-ish time l, logical counter c): respects causality and stays close to wall time.

**Q:** What doesn't HLC fix?
**A:** Clock skew itself — systems still need a max-offset bound.

**Q:** What does TrueTime return?
**A:** An interval [earliest, latest] guaranteed to contain the true time.

**Q:** What is commit-wait?
**A:** Choose ts = latest, then wait until earliest > ts before making the commit visible (~2ε).

**Q:** What does commit-wait guarantee?
**A:** External consistency: if T1 commits before T2 starts, ts(T1) < ts(T2).

**Q:** Why does Google use GPS and atomic clocks for Spanner?
**A:** To make ε small, since every write waits ~2ε.

**Q:** Precision vs accuracy of clocks?
**A:** A timestamp with µs digits can still be ms (or more) wrong.
