# 26 — Faults & Partial Failure

> Part 6 — Distributed Systems · Prerequisites: topics 01, 03, 23, 24

A program on one computer lives in a mostly deterministic world. If the hardware is healthy, the same
operation gives the same result; if something is badly broken (a kernel panic, a bad memory chip), the
whole machine usually stops. Either it works or it doesn't. The moment a system spans two or more
machines connected by a network, that comfortable model disappears. Some parts work while others don't,
messages vanish or arrive late, and — most unsettling of all — a node often *cannot find out* what
actually happened to a request it sent. This topic is about that uncertainty: where it comes from, why it
can't be engineered away, and how distributed systems are built to make correct decisions in spite of it.
Nearly everything else in Part 6 (clocks, consistency, consensus, locks, transactions) is a response to
the problems laid out here.

By the end of this topic you should be able to:
1. Explain what *partial failure* is and why it makes distributed systems fundamentally different from single-machine programs.
2. List the ways an asynchronous network can misbehave (loss, delay, duplication, reordering) and why the sender can't distinguish them.
3. Explain why timeouts are the only practical failure detector, why "slow" and "dead" are indistinguishable, and how to reason about choosing a timeout.
4. Describe how process pauses (GC, VM suspension, swapping) break assumptions, and why a node can't trust its own sense of time or ownership.
5. Distinguish the crash-stop, crash-recovery and Byzantine fault models, and the timing models (synchronous, partially synchronous, asynchronous).
6. State the Two Generals problem, prove informally why it has no solution, and connect it to idempotency and retries.

---

## 1. Partial failure: the defining property

### 1.1 One machine: "it works or it doesn't"

On a single computer, hardware is designed so that faults surface as *total* failures. A CPU that
computes a wrong answer, memory that silently flips a bit, or a disk write that half-lands are all treated
as catastrophic: ECC memory raises a machine-check exception, the kernel panics, the filesystem journal
(topic 17) is replayed on reboot. The design philosophy is: **prefer crashing to being wrong.** As a
result, a program can assume that if it is still running, the machine underneath it is behaving
correctly. A function call either returns or the whole process is gone.

### 1.2 Many machines: some parts broken, some not

In a distributed system — any set of processes that communicate only by sending messages over a network —
there is no such guarantee. At any moment:

- One node may have crashed while the others keep running.
- A network switch may drop packets between rack A and rack B while both racks talk fine to rack C.
- A node may be alive but stuck in a 10-second garbage-collection pause.
- A disk on one replica may be full, so writes there fail while reads succeed.

This is **partial failure**: some components fail while others continue, and the failures are often
*nondeterministic* — the same operation may succeed one time and fail the next. The deeper problem is not
that things break (things break on one machine too) but that **a node usually cannot tell which parts
are broken.** A remote call that hasn't returned might mean the request was lost, the remote node
crashed, the remote node is slow, or the reply was lost. Each possibility needs a different response.

```
 single machine                         distributed system
 ─────────────                         ──────────────────
 call f(x)                             send request to node B
   │                                     │
   ├─ returns result      ✔              ├─ reply arrives          ✔
   └─ process dies        ✘              ├─ error reply            ✘
      (nobody left to                    └─ ... silence ...        ?
       be confused)                          lost? crashed? slow?
                                             did B do the work?
```

The third outcome — silence — is the source of most distributed-systems complexity.

### 1.3 Why we accept this anyway

Why not just buy one giant, reliable machine? Topic 01 covered the reasons: a single machine has a
ceiling on capacity (vertical scaling), it is a single point of failure, and it lives in one location
(latency for distant users, total loss in a datacenter outage). Supercomputers take the opposite
approach — treat a partial failure as total, checkpoint, and restart the whole job — which works for
batch computations but not for an online service that must keep answering. Internet services choose to
**tolerate faults**: build a reliable system out of unreliable parts, the same way TCP builds a reliable
byte stream over unreliable IP packets (topic 03). The key word is *tolerate*: the system keeps working,
possibly degraded, while some parts are broken.

A useful vocabulary distinction (from topic 01):

| Term | Meaning |
|---|---|
| **Fault** | One component deviating from its spec (a node crashes, a link drops packets) |
| **Failure** | The system as a whole stops providing its service to users |
| **Fault tolerance** | Preventing faults from turning into failures |

Fault tolerance is always relative to an assumed set of faults — the *fault model* (section 5). No system
tolerates everything: if every node is destroyed simultaneously, nothing survives.

---

## 2. Unreliable networks

### 2.1 The asynchronous packet network

Datacenter networks and the internet are **asynchronous packet networks**: a node can send a message
(packet) to another, but the network makes *no guarantee* about when it arrives, or whether it arrives at
all. Concretely, when node A sends a request to node B, any of these can happen:

```
  A ─────request────▶ X               1. Request lost (unplugged cable, dropped by an overloaded switch)
  A ─────request──────────────────▶ B 2. Request queued, delivered much later (congestion, B overloaded)
  A ─────request────▶ B  ✘            3. B crashed (before or after processing!)
  A ─────request────▶ B  (paused)     4. B temporarily unresponsive (GC pause), will respond later
  A ◀──X──reply────── B               5. B processed it, reply lost
  A ◀────────────reply───────────── B 6. B processed it, reply delayed beyond A's patience
```

From A's point of view **all six look identical**: no reply yet. And in cases 3–6, B may or may not have
*performed the operation*. If the request was "charge the customer $50", the difference matters a lot.

### 2.2 The four message pathologies

It helps to name the distinct misbehaviors:

1. **Loss** — a message never arrives. Causes: switch buffer overflow, misconfigured firewall, cable
   damage, a NIC that drops inbound packets but still sends outbound ones (this really happens).
2. **Delay** — a message arrives, but after an unbounded amount of time. Causes: queueing (see below),
   retransmissions, a route change.
3. **Duplication** — a message arrives twice. TCP deduplicates within a single connection, but once an
   application *retries* (new connection, or a retry at a higher layer), the receiver can see the same
   logical request twice. Message brokers (topic 36) redeliver on missing acks for the same reason.
4. **Reordering** — messages arrive in a different order than sent. Within one TCP connection order is
   preserved, but two messages sent over two connections, via two paths, or one of them retried, can
   overtake each other.

TCP (topic 03) hides loss, duplication and reordering *within one connection* — but it does not hide
delay, and when the connection itself breaks (timeout, reset), the application is back to not knowing
which of its sent bytes the other side actually processed. TCP acknowledges *receipt by the kernel*,
not *processing by the application*.

### 2.3 Network partitions

A **network partition** (or "netsplit") is when the network splits the nodes into groups that can
communicate internally but not with each other.

```
     ┌──────── partition ────────┐
     │                           │
   [A] [B]      ✂✂✂✂✂✂        [C] [D] [E]
   group 1   (no traffic)       group 2
```

Partitions are not exotic. Studies of real datacenters (e.g. Bailis & Kingsbury's "The Network is
Reliable" survey, 2014) catalogued frequent partitions caused by switch failures, misconfiguration,
software upgrades of network gear, and even sharks biting undersea cables. A partition can also be
*partial* or *asymmetric*: A can reach B, B can reach C, but A cannot reach C; or A's packets reach B
but B's replies don't reach A. Asymmetric partitions are particularly nasty because each side sees a
different picture of who is alive.

The consequences of partitions run through the rest of Part 6: split brain in failover (topic 23), the
CAP theorem (topic 29), and the majority quorums of consensus (topic 30).

### 2.4 Why delays are unbounded: queueing

Network delay is mostly *queueing delay* (topic 02). A packet waits:

- in the sender's OS send buffer if TCP's congestion window is full;
- in a switch's output queue if many senders target the same port (*incast*);
- in the receiver's NIC/OS queue if all CPU cores are busy;
- inside a virtualized host while the hypervisor runs another VM.

Plus TCP retransmits after a lost packet (the retransmission timeout is at least ~200 ms on Linux by
default), which shows up as a sudden latency spike. In a multi-tenant environment, a "noisy neighbor"
using the same links can push your latency up arbitrarily. And as topic 02 showed, as utilization
approaches 100%, queueing delay grows without bound.

Contrast this with the old telephone network, which was **synchronous**: a call reserved a fixed slot of
bandwidth (circuit switching) along its whole route, so delay was fixed and bounded. That design wastes
capacity when the line is idle, so data networks chose packet switching to use bandwidth efficiently
for bursty traffic. **Unbounded delay is the price of efficient utilization** — it's a cost/benefit
choice, not a law of nature, but it's the choice every network we use has made.

---

## 3. Timeouts and failure detection

### 3.1 The only tool: waiting

Since a node can't observe another node directly, the only way to decide "B is dead" is to wait for a
response and give up after some time: a **timeout**. Some environments provide extra signals:

- If the process crashed but the OS is alive, connecting to its port gets a TCP RST ("connection
  refused") quickly.
- A script on the node can notice the process died and broadcast that fact (HBase does this).
- A switch management interface can report that a link is down.
- An ack from B's network stack (TCP level) doesn't tell you B's application processed anything.

But these are best-effort. If the whole machine lost power or the network path is broken, there's no one
left to send a "I'm dead" notification. **Ultimately, absence of a response is all you get**, and a
timeout is how you turn absence into a decision.

### 3.2 You can't tell slow from dead

The fundamental limitation: after a timeout fires, **you do not know whether the node is dead, or the
network is slow, or the node is slow.** A timeout is not a fact about the remote node; it's a fact
about *your patience*.

This is not a solvable engineering problem — it follows directly from unbounded delay. If messages can
take arbitrarily long, then for any timeout T there exists a scenario where a perfectly healthy node's
reply takes T+1. So a failure detector built on timeouts is necessarily **unreliable**: it can suspect a
live node (false positive) and can take a while to notice a dead node (detection latency).

### 3.3 Short vs long timeouts: there's no free choice

| | Short timeout | Long timeout |
|---|---|---|
| Detects real failures | Quickly | Slowly — users wait, requests pile up |
| False positives (declaring a live node dead) | Frequent, especially under load spikes | Rare |
| Cost of a false positive | Failover churn, duplicate work, possible double-execution | — |

Declaring a node dead prematurely is not harmless:

- Its work is reassigned. If it was actually alive and mid-operation, the operation may run twice
  (e.g. an email sent twice, a payment processed twice).
- If it was a leader, a new leader is elected while the old one may still think it's in charge — the
  setup for **split brain** (topic 23, topic 31).
- Under heavy load, a slow node gets declared dead, its load moves to other nodes, which then become slow
  and get declared dead too: a **cascading failure**. A timeout that is too aggressive turns a slowdown
  into an outage.

### 3.4 Choosing timeouts in practice

If the network had a guaranteed maximum delay `d` and nodes a guaranteed maximum processing time `r`,
the correct timeout would be `2d + r`: any reply not received by then means failure. Real systems have
neither bound, so instead:

- **Measure.** Look at the real distribution of response times — including p99 and p99.9, not the mean
  (topic 01). A timeout below the tail will fire on healthy requests.
- **Adapt.** Instead of a fixed constant, continuously estimate the round-trip distribution and set the
  timeout from it. TCP's retransmission timer does exactly this (smoothed RTT plus a variance term). The
  **phi-accrual failure detector** (used in Cassandra and Akka; topic 32) outputs a *suspicion level*
  based on how unusual the current silence is given past heartbeat inter-arrival times, and the
  application chooses a threshold.
- **Separate detection from action.** Suspecting a node and taking drastic action (failover) can be
  different steps with different thresholds. Many systems require several missed heartbeats, or
  confirmation from multiple observers (SWIM, topic 32), before declaring death.

### 3.5 Retries: the natural response, and its trap

When a timeout fires, the obvious response is to retry. But remember the six cases in 2.1: if the
original request was processed and only the reply was lost, a retry **executes the operation twice**.
Retries are only safe when the operation is **idempotent** — doing it twice has the same effect as doing
it once (`SET x = 5` is idempotent; `x = x + 1` is not). Section 6 shows why this isn't a fixable flaw
of the network but a theorem, and topic 38 covers how to make operations idempotent (idempotency keys,
dedup tables).

Retries also amplify load: if a service slows down, every client retrying multiplies the traffic at
exactly the worst moment. Exponential backoff with jitter and retry budgets (topic 44) exist to stop
retries from turning a hiccup into a "retry storm".

---

## 4. Process pauses

### 4.1 A node can stop without knowing it

So far we've treated the *network* as the source of uncertainty. But a node can also become unresponsive
for a while **without crashing** and then resume as if nothing happened. Examples:

- **Stop-the-world garbage collection.** A JVM or .NET runtime may halt all application threads to
  collect garbage. Pauses of seconds — occasionally minutes, with large heaps and older collectors —
  have been observed in production.
- **VM suspension / live migration.** The hypervisor can pause a virtual machine, save its memory, and
  resume it later (possibly on another host). From inside the VM, time simply jumps.
- **Laptop lid closed / OS suspend.** The same effect on end-user devices.
- **CPU steal.** On an oversubscribed host the hypervisor gives your vCPU's time to another VM.
- **Swapping / thrashing.** If memory is overcommitted, a simple memory access can trigger a disk read;
  the process crawls.
- **Synchronous disk I/O.** A thread blocked on a slow disk (or a network filesystem like EBS that is
  itself experiencing a hiccup). Even a class-loader touching disk lazily counts.
- **SIGSTOP.** An operator presses Ctrl-Z, or a debugger attaches.

The dangerous part: **the paused thread doesn't notice it was paused.** It was in the middle of a
function, and when it resumes it continues on the next line with all its local variables — including
any conclusions it had drawn about the world — intact but potentially stale.

### 4.2 The classic bug: acting on stale beliefs

Consider a leader that holds a *lease* (a time-limited lock, covered in depth in topic 31):

```
while (true) {
    request = getIncomingRequest();
    // ensure the lease has at least 10 seconds left
    if (lease.expiryTimeMillis - System.currentTimeMillis() < 10000) {
        lease = lease.renew();
    }
    if (lease.isValid()) {
        process(request);   // write to shared storage as "the leader"
    }
}
```

This looks careful. But:

```
t=0s   Node A checks lease: valid until t=30s. OK.
t=1s   A enters a 40-second GC pause  ─────────────────────┐
t=30s  Lease expires. Lock service grants lease to node B. │
t=31s  B starts processing requests as leader.             │
t=41s  A resumes ◀─────────────────────────────────────────┘
       A's code is on the line after isValid() returned true.
       A calls process(request) — writes as "leader".
       Now A and B both write: data corruption.
```

Notice that there was no network fault at all. Also note the comparison against the *local* clock:
if A's clock is out of sync with the lock service's clock (topic 27), the check is wrong even without a
pause. Checking "am I still the leader?" right before acting doesn't help, because the pause can happen
*between* the check and the action — there is no instruction-level gap small enough to be safe.

### 4.3 Lesson: a node cannot trust its own judgement

The general principle (Kleppmann's phrase): **"the truth is defined by the majority."** A node cannot
unilaterally decide it is the leader, that it holds a lock, or that another node is dead, because its own
view may be stale. Decisions must be made by a **quorum** — a majority of nodes voting — so that one
node's paused or partitioned view can't override everyone else. A node that's declared dead by the
quorum *is* dead for the system's purposes, even if it feels perfectly alive, and must step down.

And the shared resource must protect itself: the fix for the lease bug is a **fencing token** — a
number that increases every time the lease is granted, which the storage service checks and rejects if
it's older than one it has already seen (topic 31). The idea: don't rely on the client to know it's
stale; make the server enforce it.

### 4.4 Can we eliminate pauses?

Partially. *Real-time* operating systems give hard guarantees on response time, but at significant cost
(restricted languages, no dynamic allocation, lower throughput) — they're used in avionics, not web
services. Practical mitigations:

- Treat GC pauses like brief planned outages: the runtime warns the application a pause is coming, the
  load balancer stops sending it traffic, then it collects garbage while idle.
- Use collectors with short pauses (ZGC, Shenandoah, Go's GC) or restart processes before full GCs.
- Avoid swap on servers; prefer the OOM killer to thrashing.

None of these make the *guarantee* hold — so correctness must not depend on pauses being short.

---

## 5. Fault models and timing models

To reason about whether an algorithm is correct, we must say precisely which faults it tolerates. This is
the **system model**: an abstraction with two axes, timing and node failure.

### 5.1 Timing models

| Model | Assumption | Realistic? |
|---|---|---|
| **Synchronous** | Bounded network delay, bounded pauses, bounded clock error | No (for our networks) |
| **Partially synchronous** | Behaves synchronously *most of the time*, but occasionally exceeds bounds by an arbitrary amount, for an unknown duration | Yes — the usual model |
| **Asynchronous** | No timing assumptions at all; no clocks, no timeouts | Very restrictive; some algorithms work here, many problems are impossible (FLP, topic 30) |

The partially synchronous model is how real systems behave: the network is usually fast, but
sometimes there's a GC pause or congestion spike. Algorithms like Raft are designed so **safety never
depends on timing** (nothing bad happens even if the bounds are violated) while **liveness does** (the
system makes progress once things calm down). This safety/liveness split is crucial:

- **Safety:** "nothing bad happens" — e.g. two leaders never commit conflicting entries, a uniqueness
  constraint is never violated. If violated, you can point to the moment it broke, and it can't be undone.
- **Liveness:** "something good eventually happens" — e.g. a request eventually gets a response, a
  leader is eventually elected. Often contains the word "eventually".

Well-designed distributed algorithms keep safety under all circumstances the model permits, and give
up only liveness during bad periods.

### 5.2 Node fault models

**Crash-stop (fail-stop).** A node works correctly until it fails, and then it stops forever — it never
sends another message. Simplest to reason about. Example: a node whose disk is destroyed and is never
replaced; a process removed from the cluster permanently.

**Crash-recovery.** A node may crash at any moment, and may later come back, possibly after an unknown
time. Its **stable storage** (disk) survives the crash; its **in-memory state is lost**. This is what
real servers do: they reboot. Algorithms in this model must persist critical state (e.g. Raft persists
`currentTerm`, `votedFor`, and the log *before* replying — topic 30) so that a recovered node doesn't
contradict promises it made before crashing. A node that "forgets" it voted for candidate A in term 5,
then votes for B in term 5 after rebooting, can cause two leaders.

**Byzantine (arbitrary) faults.** A node may do *anything*: send corrupted messages, lie, send
different messages to different peers, collude with other faulty nodes. Named after Lamport's
"Byzantine Generals Problem" (1982). Tolerating `f` Byzantine nodes requires `3f + 1` nodes in total
(versus `2f + 1` for crash faults, topic 30) and much more expensive protocols (PBFT and descendants).

When do you need Byzantine tolerance?

- **Yes:** systems with mutually distrustful participants — blockchains, cross-organization ledgers,
  aerospace control where radiation can flip bits and a wrong answer is lethal.
- **Usually no:** a datacenter system where all nodes run your code and are under your control. If
  there's a bug in your code, all replicas run the same bug, so BFT wouldn't help anyway. Security
  against an attacker who compromised a node is usually handled with authentication, authorization and
  encryption, not BFT.

Even in non-Byzantine systems it pays to guard against *weak* forms of lying: checksums on network
messages and disk blocks (to catch corruption that TCP's weak checksum misses), sanity-checking input
values, and using several NTP servers and discarding outliers (topic 27).

### 5.3 Summary table

```
         timing ►   synchronous    partially synchronous    asynchronous
 faults ▼
 crash-stop             easy               │                  hard / FLP
 crash-recovery                       ◀── most real systems   (topic 30)
 Byzantine               3f+1 nodes, expensive protocols (blockchains, avionics)
```

Most practical datacenter algorithms (Raft, Paxos, ZooKeeper's Zab) assume **partially synchronous,
crash-recovery, non-Byzantine**.

### 5.4 Models are abstractions — reality leaks

A model is a simplification, and real systems violate it in ways that matter:

- "Stable storage survives crashes" — unless the disk dies, the firmware lies about `fsync` (topic 17),
  or the node is replaced with a fresh one with an empty disk. A Raft node that comes back with an empty
  disk under the same identity can violate safety.
- "Crash-recovery" — but a node can also lose *some* data (a corrupted file) and return in a state the
  algorithm never anticipated.
- Quorum algorithms assume nodes remember what they stored — if they don't, the quorum math (topic 24)
  breaks.

Theory tells you what's possible within the model; engineering has to handle what happens when the
model's assumptions fail (which usually means detecting it and stopping, rather than silently proceeding).

---

## 6. The Two Generals problem

### 6.1 The story

Two armies, each led by a general, are camped on opposite hills around a city in the valley. If both
attack at the same time, they win. If only one attacks, it is destroyed. The generals can communicate
only by sending messengers through the valley — and any messenger may be captured (the message lost).

General A sends: "Attack at dawn." Will A attack?

```
  General A                                   General B
     │──── "attack at dawn" ──────▶ (?)            │
     │                                             │
  A doesn't know if B got it. If A attacks and B didn't get it → A dies.
  So A waits for an acknowledgement.
     │                         ◀──── "agreed" ──── │
  Now A knows. But B doesn't know A got the ack.
  If the ack was lost, A won't attack (by A's own rule) → B attacks alone.
  So B needs an ack of the ack...
     │──── "got your agreement" ──▶ (?)            │
  Now A doesn't know if the ack-of-ack arrived...
```

### 6.2 Why it's impossible

Informal proof by contradiction: suppose there is a protocol that guarantees both generals attack
together, using some finite number of messages. Consider the *shortest* successful run, and look at the
**last message** sent in it. That message might be lost. The sender of the last message behaves the
same whether or not it was delivered (it got no further information). So the protocol must work even if
the last message is lost — meaning that message was unnecessary, and there's a shorter successful run.
Contradiction. Therefore **no finite protocol over an unreliable link can guarantee agreement**.

The general lesson: **over an unreliable channel, certainty about the other side's state cannot be
achieved with any number of messages.** Each message adds confidence; none adds certainty.

### 6.3 What it means in practice

You meet the Two Generals problem every day:

- **Online payment.** Your browser sends "charge my card". The connection drops. Did the charge happen?
  The client can't know. If it retries, it might double-charge. If it doesn't, the order might be lost.
- **Database commit over the network.** The client sends `COMMIT`, the connection times out. Was the
  transaction committed? Unknown. (This is also why two-phase commit, topic 34, has a "blocking" window.)
- **Message queues.** A consumer processes a message and sends an ack; the ack is lost; the broker
  redelivers. Did processing happen? Yes, so it's about to happen again.

Since certainty is impossible, real systems change the problem rather than solve it:

1. **Make operations idempotent** so retrying until you get confirmation is safe. The client attaches an
   *idempotency key*; the server records which keys it has processed and returns the stored result on a
   duplicate (topic 38). Now "at-least-once delivery + idempotent processing" ≈ exactly-once *effect*.
2. **Accept probabilistic agreement.** TCP's connection teardown, for example, doesn't guarantee both
   sides agree; it just uses timeouts (`TIME_WAIT`) to make disagreement harmless.
3. **Allow later reconciliation.** Systems that can't confirm in real time reconcile afterwards — e.g.
   payment providers' end-of-day settlement compares records and fixes discrepancies.
4. **Use a durable third party / consensus** to record the decision, so participants can later ask "what
   was decided?" (topic 30, topic 34). This doesn't make the network reliable; it gives a single
   authoritative place to look up the outcome once communication recovers.

### 6.4 Two Generals vs Byzantine Generals

Don't confuse them:

| | Two Generals | Byzantine Generals |
|---|---|---|
| What can go wrong | Messages lost | Some participants lie/act arbitrarily |
| Participants | Honest | Some traitorous |
| Result | Impossible to guarantee agreement over a lossy link | Possible iff fewer than 1/3 of nodes are faulty (with reliable links) |

---

## 7. Putting it together: designing for partial failure

A checklist of habits that follow from this topic:

1. **Every remote call has a timeout.** A call without one can hang forever, holding a thread and a
   connection (and cascading backpressure, topic 44).
2. **For every remote call, ask "what if it succeeded but I never heard?"** Design the retry path
   accordingly — idempotency keys, or reading back state before retrying.
3. **Don't let one node's local view drive a global decision.** Use quorums / consensus (topic 30) for
   leadership and ownership; use fencing tokens (topic 31) to protect resources.
4. **Persist promises before making them** (crash-recovery model). If a node says "yes" to something,
   that yes must survive a reboot.
5. **Make safety independent of timing.** Timeouts may decide *when* to try something (liveness), never
   *whether* something is correct (safety).
6. **Test faults deliberately.** Kill processes, inject latency, partition networks (Jepsen-style
   testing, chaos engineering — topic 48). Bugs in partial-failure handling hide until production
   because the failure modes are rare and timing-dependent.

---

## 8. How it connects

- **Topic 01** introduced faults vs failures and fault tolerance as vocabulary; this topic shows why
  tolerating faults in a distributed setting is fundamentally harder.
- **Topic 02**: queueing explains why network delays are unbounded and grow with utilization.
- **Topic 03**: TCP hides loss/reordering/duplication within a connection but not delay, and not
  application-level uncertainty when a connection breaks.
- **Topic 23**: failover relies on timeouts, which can't distinguish slow from dead — the root cause of
  split brain.
- **Topic 24**: quorums are a way of making decisions without trusting any single node's view.
- **Topic 27** (next): clocks are another thing a node can't trust — wall clocks drift and jump.
- **Topic 29**: CAP is about what a system does during a network partition.
- **Topic 30**: consensus algorithms are designed for exactly this model (partially synchronous,
  crash-recovery) and rely on majorities because a node can't trust itself.
- **Topic 31**: leases, locks and fencing tokens solve the process-pause problem from section 4.
- **Topic 32**: practical failure detectors (heartbeats, SWIM, phi-accrual).
- **Topic 34**: two-phase commit is the Two Generals problem with a coordinator, and blocks for that reason.
- **Topic 38**: idempotency is the practical answer to "did my request get processed?".
- **Topics 44 and 48**: timeouts, retries with backoff, circuit breakers, and chaos testing.

---

## Common misconceptions

- **"The network is reliable inside a datacenter."** → Partitions, packet loss and long delays are
  regularly observed in datacenters; the first of the classic "fallacies of distributed computing" is
  assuming otherwise.
- **"TCP makes the network reliable."** → TCP gives an ordered, deduplicated stream *while the connection
  is alive*. It doesn't bound delay, and when a connection breaks you don't know which data the remote
  application processed.
- **"If a node doesn't respond, it's down."** → It might be slow, paused, overloaded, or its replies
  might be lost. A timeout only says you stopped waiting.
- **"A good enough timeout value removes the problem."** → Any fixed timeout is a trade-off between
  detection speed and false positives; with unbounded delays, no value is correct in all cases.
- **"Retrying a failed request is always safe."** → Only if the operation is idempotent; otherwise a
  retry after a lost reply executes it twice.
- **"If my code checked the lease right before writing, it's safe."** → A pause can occur between the
  check and the write. Safety must be enforced by the resource (fencing), not by the client's belief.
- **"We should build everything Byzantine fault tolerant to be safe."** → BFT costs 3f+1 nodes and
  heavy protocols and doesn't help against bugs shared by all replicas; inside one organization,
  crash-recovery plus checksums and authentication is the normal choice.
- **"The Two Generals problem means distributed agreement is impossible."** → It means *guaranteed*
  agreement over a lossy link in bounded messages is impossible. Real systems use idempotent retries,
  durable logs and consensus to get practical agreement once communication works again.

---

## Further reading (supplements)

- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 8 "The Trouble with Distributed Systems".
- Kleppmann, *Distributed Systems* lecture series (Cambridge), lectures 2.1–2.3 (Two Generals, Byzantine Generals, system models). Free on YouTube.
- Peter Bailis & Kyle Kingsbury, "The Network is Reliable" (ACM Queue, 2014).
- Leslie Lamport, Robert Shostak, Marshall Pease, "The Byzantine Generals Problem" (1982).
- Peter Deutsch et al., "Fallacies of Distributed Computing".
- MIT 6.5840 (Distributed Systems), Lecture 1 introduction.

---

## Questions

1. What is "partial failure", and why does it not exist (in practice) on a single machine?
2. Node A sends a request to node B and receives no reply. List the possible explanations. Which of them
   mean the operation was performed?
3. TCP guarantees reliable, in-order delivery. Why doesn't that solve the problem in question 2?
4. Why are delays in datacenter networks unbounded? What design choice makes them so, and why was that
   choice made?
5. Why is it impossible to build a perfect failure detector in an asynchronous network?
6. Your team sets a 200 ms timeout on calls to an inventory service whose p50 is 20 ms and p99.9 is
   450 ms, and failover triggers after one timeout. What will you observe, especially under load?
7. A leader process checks that its lease is valid, then writes to shared storage. Describe exactly how
   this can produce two concurrent writers even with no network problems.
8. Why is "the truth is defined by the majority" the right principle for leadership decisions?
9. Distinguish crash-stop from crash-recovery. Why does an algorithm in the crash-recovery model need
   to write state to disk before replying to a message?
10. When is Byzantine fault tolerance worth the cost, and why is it usually not used inside a single
    company's datacenter?
11. Explain the difference between safety and liveness properties, and why practical algorithms
    sacrifice liveness rather than safety under bad timing.
12. Sketch the proof that the Two Generals problem has no solution.
13. A mobile app sends "place order" and the connection drops before a response. How should the system
    be designed so the user can safely press "retry"?
14. A Raft node's disk is replaced and it rejoins with an empty disk under its old identity. Which
    system-model assumption is violated, and why does that matter?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Partial failure is when some components of a system fail while others keep working, often
nondeterministically, and the working parts can't reliably tell what failed. On a single machine the
hardware and OS are designed to turn faults into total failures (machine-check exception, kernel panic)
— "crash rather than be wrong" — so a running program can assume the machine is behaving correctly.
There's no observer left in an ambiguous state.

**2.** (a) Request lost in the network; (b) request delayed and still in a queue; (c) B crashed before
processing; (d) B crashed after processing; (e) B paused or overloaded and will process it later;
(f) B processed it and the reply was lost; (g) the reply is delayed. The operation was (or will be)
performed in (d), (e), (f), (g) and possibly (b). A cannot distinguish any of these.

**3.** TCP guarantees order and deduplication within a live connection and retransmits lost packets, but
it doesn't bound delay, and its acknowledgements mean the remote *kernel* received bytes, not that the
application processed them. If the connection times out or resets, the application doesn't know how far
the other side got. A new connection for a retry is a new stream, so duplicates at the application level
are possible.

**4.** Because packets queue — in switch buffers, OS buffers, NICs, hypervisors — and queueing delay
grows without bound as utilization approaches 100% (topic 02). TCP retransmission adds more. Networks
use packet switching rather than circuit switching (which reserves fixed bandwidth and gives bounded
delay) because bursty data traffic would waste reserved capacity; packet switching uses links
efficiently at the cost of variable delay.

**5.** A failure detector must decide based on silence. If delays have no upper bound, then for any
timeout there's a run where a healthy node's reply arrives just after the timeout, so the detector will
sometimes wrongly suspect a live node; and if it waits longer it detects real crashes more slowly. It
can't be both accurate and complete in finite time.

**6.** At least 0.1% of healthy requests (more, since 200 ms is well below p99.9) will time out and
trigger failover. Under load, latency rises, so many more requests exceed 200 ms, causing frequent
spurious failovers. Each failover shifts load to the remaining nodes, which then slow down and get
failed over too — a cascading failure. Also, timed-out requests that were actually processed may be
retried, producing duplicate effects. The timeout should be derived from the tail distribution, failover
should require repeated or multi-observer confirmation, and retries should be idempotent.

**7.** The leader checks the lease (valid until t=30). Immediately after the check, the process is paused
— e.g. a stop-the-world GC of 40 s, or VM suspension. The lease expires at t=30 and the lock service
grants it to another node, which starts writing. When the first process resumes, it continues from the
line after the check and performs its write, believing it is still leader. Now two writers. The fix is
fencing tokens checked by the storage system.

**8.** Any single node's view may be stale or wrong (it may be paused, partitioned, or have a skewed
clock), and it can't detect its own staleness. A majority quorum makes decisions that no minority
(including a lone confused node) can contradict, because any two majorities overlap in at least one
node, which carries knowledge of the decision. So the system's truth is whatever a majority agreed on,
and a node the majority declared dead must step down even if it feels alive.

**9.** Crash-stop: once a node fails it never returns. Crash-recovery: it may come back after a crash,
with disk state intact but memory lost. If a node makes a promise (votes for a candidate, acknowledges
a write) and only keeps it in memory, a crash and restart would let it make a contradictory promise
(vote for a different candidate in the same term), breaking safety. Persisting before replying means
every promise survives restarts.

**10.** When participants don't trust each other or may be adversarial (blockchains, multi-organization
ledgers), or when hardware may produce arbitrary wrong outputs and errors are lethal (aerospace). Inside
one company, all nodes run the same software under one administrator: bugs would be replicated on all
nodes so BFT doesn't help, the 3f+1 node count and protocol overhead are expensive, and security is
handled by authentication/authorization. Lightweight protections (checksums, input validation) cover
the common "weak lying" cases.

**11.** Safety: nothing bad ever happens (e.g. no two leaders commit conflicting data); a violation
occurs at an identifiable point and can't be undone. Liveness: something good eventually happens (e.g.
a request eventually completes). In a partially synchronous system, timing bounds are sometimes
violated; if correctness depended on timing, those periods would corrupt data permanently. Giving up
liveness instead means the system temporarily stalls but recovers once timing normalizes — an outage is
recoverable, corruption often isn't.

**12.** Assume a protocol that guarantees coordinated attack using a finite number of messages. Take the
shortest successful run and its last message. That message could be lost, and its sender can't tell the
difference, so it acts the same either way; the receiver must act the same too (otherwise the outcome
depends on a message whose loss the sender can't detect, so they'd disagree). Hence the last message
is unnecessary and there's a shorter successful run — contradiction. No finite protocol works.

**13.** The client generates an idempotency key (e.g. a UUID) when the user first presses "place order",
and sends it with every attempt. The server stores the key together with the result of processing it,
atomically with the order creation (same database transaction). On a retry with the same key it returns
the stored result instead of creating a second order. The user can retry as many times as needed and
the order is created once.

**14.** The crash-recovery model assumes stable storage survives crashes. A node with an empty disk has
forgotten its `votedFor` and its log entries, including entries it acknowledged as part of a committed
majority. It could vote twice in a term (two leaders) or help a candidate with a stale log win, causing
committed entries to be lost. Safe handling is to treat it as a new member and go through a membership
change, not let it reuse its old identity.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What is partial failure?
**A:** Some components fail while others keep working, nondeterministically, and nodes can't reliably tell which parts failed.

**Q:** Single machine's attitude to faults?
**A:** Prefer crashing to being wrong — faults become total failures.

**Q:** Four ways an asynchronous network misbehaves?
**A:** Loss, delay (unbounded), duplication, reordering.

**Q:** No reply from B — can A know if B did the work?
**A:** No. Lost request, crash, slowness, and lost reply all look identical.

**Q:** What does a TCP ack actually confirm?
**A:** That the remote kernel received the bytes — not that the application processed them.

**Q:** What is a network partition?
**A:** The network splits nodes into groups that can talk internally but not to each other (possibly asymmetrically).

**Q:** Main source of unbounded network delay?
**A:** Queueing (switches, OS buffers, hypervisors) plus retransmissions.

**Q:** Why do packet networks have unbounded delay but phone circuits don't?
**A:** Circuits reserve fixed bandwidth (bounded delay, wasted capacity); packet switching shares links efficiently at the cost of variable delay.

**Q:** What's the only general way to detect a failed node?
**A:** A timeout.

**Q:** What does a timeout tell you?
**A:** That you stopped waiting — not whether the node is dead, slow, or the network is slow.

**Q:** Cost of a timeout that's too short?
**A:** False positives: duplicate work, spurious failover, split brain, cascading failures under load.

**Q:** Cost of a timeout that's too long?
**A:** Slow detection of real failures; users wait and requests pile up.

**Q:** What is a phi-accrual failure detector?
**A:** An adaptive detector that outputs a suspicion level based on how unusual the current silence is versus past heartbeat timing.

**Q:** When is retrying a request safe?
**A:** When the operation is idempotent (or made so with an idempotency key).

**Q:** Examples of process pauses?
**A:** Stop-the-world GC, VM suspension/migration, CPU steal, swapping, blocking disk I/O, SIGSTOP.

**Q:** Why is "check the lease, then write" unsafe?
**A:** The process can pause between the check and the write; the lease expires and someone else gets it.

**Q:** What fixes the paused-lease-holder problem?
**A:** Fencing tokens — monotonically increasing numbers the storage service checks and rejects if stale.

**Q:** "The truth is defined by the majority" — meaning?
**A:** A node can't trust its own view; decisions (leadership, liveness) are made by a quorum.

**Q:** Crash-stop vs crash-recovery?
**A:** Crash-stop: dead forever. Crash-recovery: may come back with disk intact, memory lost.

**Q:** Why must a crash-recovery node persist before replying?
**A:** So it can't break a promise (e.g. vote twice in a term) after restarting.

**Q:** What is a Byzantine fault?
**A:** A node behaving arbitrarily — lying, corrupting or sending inconsistent messages.

**Q:** Nodes needed to tolerate f Byzantine faults vs f crash faults?
**A:** 3f+1 vs 2f+1.

**Q:** Three timing models?
**A:** Synchronous (bounded), partially synchronous (usually bounded, sometimes not), asynchronous (no timing assumptions).

**Q:** Which system model do Raft/Paxos assume?
**A:** Partially synchronous, crash-recovery, non-Byzantine.

**Q:** Safety vs liveness?
**A:** Safety: nothing bad ever happens. Liveness: something good eventually happens.

**Q:** Which do good algorithms sacrifice under bad timing?
**A:** Liveness (temporarily stall), never safety.

**Q:** Two Generals problem in one line?
**A:** Over a lossy channel, no finite number of messages can guarantee both sides agree.

**Q:** Practical answers to the Two Generals problem?
**A:** Idempotent retries, reconciliation, durable records of decisions, accepting probabilistic agreement.

**Q:** Two Generals vs Byzantine Generals?
**A:** Two Generals: honest parties, lossy links. Byzantine: reliable links, some parties lie.
