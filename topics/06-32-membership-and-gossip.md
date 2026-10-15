# 32 — Membership, Failure Detection & Gossip

> Part 6 — Distributed Systems · Prerequisites: topics 26, 30, 31

Every cluster has to answer a basic question all the time: *who is in the cluster right now, and which of
them are working?* A load balancer needs it to route traffic (topic 9). A Cassandra coordinator needs it to
pick replicas (topic 24). A consistent-hash ring needs it to know which nodes own which keys (topic 13).
Topic 26 showed that the honest answer is "you can't know for sure": a node that hasn't replied might be
dead, slow, or cut off. This topic is about how real systems answer the question anyway, approximately and
quickly, at scale: **failure detectors** that decide when to suspect a node, and **gossip protocols** that
spread membership information to thousands of nodes without any central coordinator.

By the end of this topic you should be able to:
1. Explain what cluster membership is and why maintaining it is hard.
2. Explain the two properties of a failure detector (completeness and accuracy) and why no detector can be perfect.
3. Describe heartbeat-based detection and why naive all-to-all heartbeating doesn't scale.
4. Explain how gossip spreads information, and why convergence takes O(log N) rounds.
5. Walk through SWIM: direct probe, indirect probe, suspicion, and piggybacked dissemination.
6. Explain the phi-accrual failure detector and why an adaptive suspicion level beats a fixed timeout.
7. Choose between consensus-based membership and gossip-based membership for a given system.

---

## 1. Cluster membership and why it is hard

### 1.1 What membership means

The **membership list** (or "view") is each node's idea of the set of nodes in the cluster and their states:

```
node A's view:
  B   alive    incarnation 4   addr 10.0.0.2
  C   suspect  incarnation 7   addr 10.0.0.3
  D   dead     incarnation 2   addr 10.0.0.4
  E   alive    incarnation 1   addr 10.0.0.5   (joined 30s ago)
```

Membership changes for three reasons:

- **Joins:** a new node is started, or a node recovers.
- **Voluntary leaves:** a node is decommissioned and announces it.
- **Failures:** a node crashes, hangs, or becomes unreachable, and announces nothing.

The first two are easy because the node tells someone. The third is the hard part: failures must be
*detected*, and detection is guesswork.

### 1.2 Why it is hard

1. **Slow vs dead is undecidable (topic 26).** Over an asynchronous network with unbounded delays and
   process pauses, no message pattern lets you distinguish "crashed" from "very slow". Every detector is a
   guess with a timeout somewhere.
2. **Different nodes see different things.** A partition might mean A can reach C but B cannot. So B
   "knows" C is dead and A "knows" C is alive. Without a coordination mechanism, views diverge.
3. **Changes are continuous at scale.** In a 5,000-node cluster, with each node failing on average once
   every couple of years, you expect several failures a day, plus deploys, autoscaling and reboots.
   Membership is never "settled".
4. **Mistakes are costly in both directions.**
   - Declaring a live node dead (**false positive**) triggers rebalancing, re-replication, or a leader
     election. The extra load can overload other nodes and cause more false positives: a cascade.
   - Failing to notice a dead node for a long time (**slow detection**) means requests keep going to it and
     time out, and writes may lack enough replicas.
5. **The detector itself must scale.** If every node pings every other node, that's N² messages.

### 1.3 Two families of solutions

| | Consensus-based membership | Gossip-based membership |
|---|---|---|
| How | Membership is a linearizable record in etcd/ZooKeeper (topic 31) or a Raft log; nodes hold leases/ephemeral nodes | Each node keeps its own view; views are exchanged with random peers and converge |
| Consistency | All nodes agree on one view, totally ordered changes | Eventually consistent: views temporarily differ |
| Scale | Hundreds to low thousands of members per coordination cluster | Tens of thousands of nodes |
| Failure of the mechanism | Coordination cluster down → no membership changes | No central point; degrades gracefully |
| Examples | Kafka (ZooKeeper / KRaft), Kubernetes node leases in etcd, HBase | Cassandra, Riak, Consul/Serf (memberlist), Akka Cluster, ScyllaDB |

When agreement matters (exactly one leader per shard, a definitive list of who owns a partition), use
consensus. When you need a scalable, approximately right view (which replicas to try, which nodes look
healthy), gossip is the standard choice. Many systems use both: gossip for liveness, consensus for the
authoritative metadata.

---

## 2. Failure detectors

### 2.1 Completeness and accuracy

Chandra and Toueg (1996) formalised failure detectors with two properties:

- **Completeness:** every node that actually crashed is eventually suspected by every correct node.
- **Accuracy:** a correct (live) node is not suspected.

You'd like both, but in an asynchronous system you cannot have perfect accuracy together with completeness:
a slow node is indistinguishable from a dead one, so either you eventually suspect it (possibly wrongly) or
you wait forever. Practical detectors guarantee completeness and trade accuracy against **detection time**:

```
     short timeout                             long timeout
  fast detection  ◄─────────────────────────►  slow detection
  more false positives                          fewer false positives
```

Useful metrics for a real detector:

- **Detection time:** how long from crash to suspicion.
- **Mistake rate:** how often live nodes are wrongly suspected.
- **Message load:** messages per node per second.

### 2.2 Heartbeats

The simplest scheme: each node sends "I'm alive" every interval T (say 1 s). If the monitor hears nothing
for timeout Δ (say 5 s), it suspects the node.

```
A →  ♥     ♥     ♥     ♥     [crash]
     0s    1s    2s    3s                     8s
monitor:                     last heard 3s → suspect at 3+5 = 8s
```

Variations:

- **Push vs pull:** node sends heartbeats (push), or the monitor pings and expects a reply (pull/ping-ack).
- **Centralised:** one monitor (or the coordination service) tracks everyone. Simple, but the monitor is a
  bottleneck and a single point of failure, and it only sees *its* network path.
- **All-to-all:** every node heartbeats every other node. Detection is fast and doesn't depend on one
  monitor, but the load is N(N−1) messages per interval. 1,000 nodes → about 1 million messages/second at T = 1 s.
  It doesn't scale.
- **Ring / neighbours:** each node watches a few neighbours. Cheap, but correlated failures (a rack loses
  power) can take out a node and all its watchers.

The fixed timeout is also a problem. Network latency varies by time of day, load, and GC activity. A
timeout tuned for the quiet case produces false positives during a busy period; one tuned for the worst case
detects real failures slowly. Section 4 (phi-accrual) addresses that.

### 2.3 Why "one node's opinion" is unreliable

If A can't reach B, maybe B is dead, or maybe the A–B link is broken, or A itself is overloaded and dropping
packets. Declaring B dead purely on A's view causes false positives from *A's* problems. Good detectors ask
others for a second opinion (SWIM's indirect probes) or let the suspected node refute the accusation
(SWIM's suspicion mechanism).

---

## 3. Gossip protocols

### 3.1 The idea

**Gossip** (also called **epidemic protocols**) spreads information the way rumours or infections spread.
Periodically (every round, e.g. once per second), each node picks one or a few random peers and exchanges
information with them. Nobody coordinates; information just diffuses.

```
round 0:  A knows the update                              informed: 1
round 1:  A → C                                           informed: 2
round 2:  A → F, C → B                                    informed: 4
round 3:  A → D, C → G, F → H, B → E                      informed: 8
...                                                       roughly doubles
```

Three exchange styles:

- **Push:** the informed node sends the update to its random peer. Effective early when few nodes know.
- **Pull:** each node asks a random peer "what do you have that I don't?". Effective late when most know
  (the stragglers find it quickly).
- **Push-pull:** both in the same exchange. Fastest overall, and what most systems do.

### 3.2 Convergence time

With push gossip and fanout 1, the number of informed nodes roughly doubles each round at first, so reaching
about half the cluster takes ~log₂N rounds. The final stragglers take another O(log N) rounds (and pull
makes this phase faster). The standard result: **all N nodes learn an update in O(log N) rounds with high
probability.**

| Cluster size N | log₂ N | rough rounds to full spread |
|---|---|---|
| 100 | ~7 | ~10 |
| 1,000 | ~10 | ~15 |
| 10,000 | ~13 | ~20 |

So with 1-second rounds, a 10,000-node cluster converges in tens of seconds. Each node sends only a constant
number of messages per round regardless of N, so per-node load is O(1) and total load O(N), versus O(N²) for
all-to-all heartbeating.

Why it is robust: there is no special node and no fixed path. If some messages are lost or some nodes are
down, the update still arrives via other random paths. Redundancy is built in.

Trade-offs:

- **Probabilistic, eventual.** No hard bound on when a specific node learns something; views are
  temporarily inconsistent.
- **Redundant traffic.** Late in the spread, most exchanges carry news the receiver already has.
- **Bounded message size.** You can't send the whole cluster state every round in a big cluster; systems
  send digests (versions) first and only the differences.

### 3.3 Anti-entropy and rumour mongering

Two classic modes (Demers et al., 1987):

- **Anti-entropy:** periodically compare *entire* state with a random peer and reconcile all differences.
  Guarantees eventual convergence, more expensive. Cassandra and Dynamo's replica repair uses Merkle
  trees (topic 53) to make the comparison cheap.
- **Rumour mongering:** spread only *recent* updates ("hot rumours"); a node stops spreading a rumour after
  it has been sent a certain number of times. Cheaper, but a small chance some node misses it, so it is
  usually backed by occasional anti-entropy.

### 3.4 Gossip-based membership with heartbeat counters

A simple, widely used design (similar to Cassandra's):

- Each node keeps, for every known node, a **heartbeat counter** and the local time it last saw that counter
  increase.
- Each node increments its own counter every round.
- Each round it gossips its table (or a digest of it) to a random peer. On receipt, for each entry, keep the
  higher counter and reset "last increased" to now if it went up.
- If a node's counter hasn't increased for T_fail, mark it suspect/dead. After a further T_cleanup, remove it
  (the wait prevents a dead node's old entry from being re-introduced by another node's stale gossip).

```
A's table                    B's table              A after merging B's
node  counter  last-up       node  counter          node  counter  last-up
A     105      now           A     101              A     105      now
B     88       now-1s        B     90               B     90       now      ← increased, refresh
C     40       now-12s       C     40               C     40       now-12s  ← stuck → suspect
```

Note that "C is suspect" is a *local* decision, based on its counter not growing. No node ever directly
declares another dead to the cluster; each concludes it independently from gossiped evidence. Cassandra
combines this with phi-accrual (section 4.2) to judge "hasn't increased for long enough".

Cassandra also gossips other state along with heartbeats: each node's token ranges (topic 13), schema
version, load, and status (joining, normal, leaving). The heartbeat state carries a **generation** (set at
startup) and **version**, so a restarted node is recognised as newer than its old incarnation.

---

## 4. SWIM and phi-accrual

### 4.1 SWIM

**SWIM** (Scalable Weakly-consistent Infection-style process group Membership, Das, Gupta & Motivala,
2002) separates the two jobs: *detecting* failures and *disseminating* membership changes. It is the basis
of HashiCorp's memberlist (used in Consul, Serf, Nomad) and influenced many others.

**Failure detection (each protocol period, e.g. 1 s), run by every node M:**

1. Pick one member J (round-robin through a shuffled list, so every node is probed within bounded time).
2. Send `ping` to J. If an `ack` arrives within a short timeout, J is fine.
3. Otherwise, pick k other members (e.g. k = 3) and send them `ping-req(J)`. Each pings J on M's behalf and
   forwards any ack to M.
4. If no ack arrives (direct or indirect) by the end of the period, M marks J **suspect**.

```
        M ──ping──► J      (no ack: maybe J dead, maybe M–J link broken)
        │
        ├─ping-req(J)─► X ──ping──► J ──ack──► X ──ack──► M   ✓ J alive
        ├─ping-req(J)─► Y ──ping──► J
        └─ping-req(J)─► Z ──ping──► J
```

The indirect probe is the key idea: a single congested link between M and J no longer causes J to be
declared failed, because other nodes reach J through different paths. Load per node is constant: one ping per
period plus occasional ping-req traffic, independent of N.

**Suspicion sub-protocol.** Rather than going straight to "dead", J is first marked **suspect**, and that is
gossiped. If J hears it is suspected, it refutes by gossiping an `alive` message with a higher
**incarnation number** (a counter only J itself may increment). If no refutation arrives within the
suspicion timeout, J is declared **dead** (SWIM calls it "confirm"). Precedence rules:

```
alive(J, inc=i)   overrides  suspect(J, inc<i)
suspect(J, inc=i) overrides  alive(J, inc≤i)
dead(J)           overrides everything
```

The incarnation number prevents stale "suspect" rumours from killing a node that has already proven it's alive.

**Dissemination by piggybacking.** Membership updates (join, suspect, alive, dead) are not sent as separate
broadcasts. They ride on the ping, ping-req and ack messages the detector is already sending. Each update is
piggybacked a limited number of times (proportional to log N), which is enough for it to reach everyone with
high probability. Gossip comes "for free" from the failure detector's traffic.

**Lifeguard.** HashiCorp found that SWIM nodes which are themselves slow (CPU-starved, GC-paused) send
false suspicions, because *they* can't process acks in time. Their Lifeguard extensions make a node less
aggressive when it sees signs of its own slowness ("local health awareness") and shorten suspicion timeouts
as independent confirmations arrive. This is a practical example of section 2.3: be suspicious of your own
judgement.

### 4.2 Phi-accrual failure detector

Hayashibara et al. (2004) proposed: instead of a binary "alive / dead after Δ", output a continuous
**suspicion level φ (phi)** that grows the longer a heartbeat is overdue, *relative to the history of
inter-arrival times*.

How it works:

1. Keep a sliding window of recent heartbeat inter-arrival times from a node (say the last 1,000).
2. Estimate their distribution (mean μ and standard deviation σ; the original assumes a normal distribution,
   Akka and Cassandra use simpler approximations such as an exponential distribution).
3. Given the time t since the last heartbeat, compute the probability P that a heartbeat would arrive *even
   later than t* if the node were alive.
4. φ = −log₁₀(P).

Interpretation: φ = 1 means about a 10% chance we're wrong to suspect, φ = 2 means 1%, φ = 3 means 0.1%,
φ = 8 means 1 in 100 million.

```
heartbeats usually every ~1s (σ small)
time since last heartbeat:  1s     2s     3s     4s
phi:                        0.3    2      6      12     → crosses threshold 8 → suspect

heartbeats on a jittery link, every ~1s but σ large
time since last heartbeat:  1s     2s     3s     4s
phi:                        0.3    0.8    1.6    2.8    → still below threshold
```

The application chooses a **threshold** (Cassandra's `phi_convict_threshold` defaults to 8; Akka's default is
also around 8). Benefits:

- **Adapts automatically.** On a jittery network, heartbeat gaps are more variable, so φ rises more slowly and
  false positives fall. On a steady network, detection is fast.
- **Separates measurement from policy.** Different consumers can act at different levels: e.g. stop sending
  new requests at φ = 3, trigger rebalancing only at φ = 10.

Limits: it still relies on a timeout in effect. A long GC pause on the monitored node will still raise φ
past the threshold, as it should, since the node really is unresponsive. And a sudden shift in network
conditions takes time to be reflected in the window.

### 4.3 Comparing approaches

| | Fixed-timeout heartbeat | Phi-accrual | SWIM |
|---|---|---|---|
| Output | Binary | Continuous φ, threshold chosen by user | Alive / suspect / dead |
| Adapts to network | No | Yes, via history | Partly (indirect probes, Lifeguard) |
| Message load per node | Depends on topology (all-to-all: O(N)) | Same as underlying heartbeats | O(1) |
| Robust to a single bad link | No | No | Yes (ping-req) |
| Used in | Simple systems, coordination-service sessions | Cassandra, Akka | Consul, Serf, Nomad (memberlist) |

---

## 5. Putting it together: what systems do with membership

### 5.1 How the view is used

- **Routing decisions** (which replica to send a read to) can tolerate a slightly stale view: if a node is
  wrongly thought dead, you just use another replica; if wrongly thought alive, the request times out and you
  retry elsewhere (topic 43).
- **Ownership decisions** (who is the primary for this shard, who owns this token range) cannot tolerate two
  nodes believing different things. These go through consensus (topic 30, 31) even if liveness comes from gossip.

That's why Cassandra uses gossip for liveness but, historically, needed care around schema and topology
changes (and newer versions moved cluster metadata to a consensus-backed log), and why Kafka moved its
controller metadata into a Raft quorum (KRaft) rather than gossip.

### 5.2 Avoiding flapping and cascades

- **Hysteresis / suspicion periods:** don't immediately evict; require a period of suspicion, as in SWIM.
- **Don't re-replicate instantly:** Cassandra marks a node down but doesn't move its data until an operator
  removes it; HDFS waits ~10 minutes before re-replicating a dead DataNode's blocks. A brief network blip
  then costs nothing.
- **Rate-limit the reaction:** cap how many nodes can be evicted per minute; if many nodes appear to fail at
  once, the problem is more likely the network or the observer.

### 5.3 Join and seed nodes

A brand-new node knows nobody. Gossip systems configure a few **seed nodes** whose addresses are known in
advance (Cassandra's `seeds` list). The new node contacts a seed, gets its membership table, and then
participates in normal gossip. Seeds have no other special role; they only bootstrap. To prevent a
partitioned cluster from never re-merging, nodes also periodically gossip with a seed.

---

## 6. How it connects

- **Topic 26 (faults):** "can't tell slow from dead" is why failure detectors are probabilistic.
- **Topic 9 (load balancing):** health checks are a centralised heartbeat failure detector.
- **Topic 13 (consistent hashing):** the ring is built from the membership view; gossip spreads token ownership.
- **Topic 24 (leaderless replication):** Cassandra/Dynamo choose replicas, sloppy quorums and hinted handoff
  based on gossip's view of liveness; anti-entropy is gossip for data.
- **Topic 30 (consensus):** consensus can maintain membership authoritatively; Raft itself has a heartbeat-based
  failure detector (election timeout).
- **Topic 31 (coordination):** ephemeral nodes/leases implement consensus-based membership.
- **Topic 33 (CRDTs):** gossip is a natural transport for state-based CRDTs; membership tables merge like CRDTs.
- **Topic 43–44 (timeouts, circuit breakers):** per-client failure detection at the request level.
- **Topic 53 (Merkle trees):** make anti-entropy comparisons cheap.

---

## Common misconceptions

- "A good enough failure detector can tell exactly when a node is dead." → In an asynchronous system, no
  detector can be both complete and perfectly accurate. Every detector trades detection speed for mistakes.
- "Gossip is unreliable because messages are random." → The randomness is what makes it robust: no single
  path or node is critical, and convergence in O(log N) rounds holds with high probability.
- "Gossip is slow." → It's O(log N) rounds; for tens of thousands of nodes that's tens of seconds with 1 s
  rounds, while each node does constant work.
- "If node A can't reach B, B is down." → It may be A's link or A itself. SWIM's indirect probes and Lifeguard
  exist because of this.
- "Phi-accrual has no timeout." → It replaces a *fixed* timeout with an adaptive one derived from observed
  heartbeat history; the threshold still determines when you act.
- "Gossip membership means all nodes agree on who is alive." → Views converge eventually but may differ at
  any instant. Decisions that need agreement must use consensus.
- "Marking a node dead should immediately trigger data re-replication." → That turns every transient blip
  into a heavy, possibly cascading, data movement. Systems wait.

---

## Further reading (supplements)

- Kleppmann, *Designing Data-Intensive Applications*, Ch. 8 (unreliable networks, timeouts, "The truth is
  defined by the majority").
- Das, Gupta & Motivala, "SWIM: Scalable Weakly-consistent Infection-style Process Group Membership Protocol" (DSN 2002).
- Hayashibara et al., "The φ Accrual Failure Detector" (SRDS 2004).
- Demers et al., "Epidemic Algorithms for Replicated Database Maintenance" (PODC 1987).
- Chandra & Toueg, "Unreliable Failure Detectors for Reliable Distributed Systems" (JACM 1996).
- DeCandia et al., "Dynamo: Amazon's Highly Available Key-value Store" (SOSP 2007), membership and failure detection sections.
- HashiCorp, "Lifeguard: Local Health Awareness for More Accurate Failure Detection" (paper and blog post), and the memberlist docs.
- Cassandra documentation: gossip and failure detection.

---

## Questions

1. Why is it fundamentally impossible to build a failure detector that is both complete and always accurate?
2. A cluster has 2,000 nodes. Why is all-to-all heartbeating a bad idea, and what do you use instead?
3. Explain why gossip spreads an update to all N nodes in roughly O(log N) rounds.
4. What are the trade-offs between push, pull and push-pull gossip? Which phase of spreading is each good at?
5. What is the difference between anti-entropy and rumour mongering?
6. In heartbeat-counter gossip, why does a node wait an extra cleanup period before removing a dead node's entry?
7. Walk through SWIM's probe of node J when the direct link M→J is congested but J is healthy.
8. What is an incarnation number in SWIM, and what problem does it solve?
9. How does SWIM disseminate membership changes without separate broadcast messages?
10. A phi-accrual detector monitors two nodes, one on a stable link and one on a jittery link. Both stop
    heartbeating at the same moment. Which is suspected first, and why is that desirable?
11. Your Cassandra cluster starts marking healthy nodes as down during a busy period every evening. What could
    be happening, and what would you look at?
12. Why do many systems use gossip for liveness but consensus for ownership decisions such as "who is the primary"?
13. Why do systems like HDFS wait minutes before re-replicating data from a node marked dead?
14. What are seed nodes in a gossip system for, and are they a single point of failure?
15. A monitoring node becomes CPU-starved and starts suspecting many peers at once. How do SWIM/Lifeguard-style
    designs limit the damage?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** In an asynchronous system messages and processes can be delayed arbitrarily, so a crashed node and a
very slow node produce identical observations (silence). To be complete you must eventually suspect a silent
node; that will sometimes be a slow live node, violating accuracy. You can only trade detection time
against mistake rate.

**2.** All-to-all needs N(N−1) ≈ 4 million messages per interval; with 1 s heartbeats that's millions of
messages per second, and every node handles ~2,000 messages per second just for liveness. Use a scheme with
O(1) per-node load: SWIM (one probe per period plus indirect probes) or gossip-based heartbeat counters.

**3.** Initially each informed node informs about one new node per round, so the informed count roughly doubles:
reaching ~N/2 takes ~log₂N rounds. Finishing off the last uninformed nodes takes another O(log N) rounds (pull
helps since each uninformed node asks a random peer, and most peers know). Total O(log N) w.h.p.

**4.** Push: informed nodes send; great early when few know, but inefficient late because most pushes go to
already-informed nodes. Pull: everyone asks; inefficient early (most peers don't know) but excellent late
because uninformed nodes find the update quickly. Push-pull does both in one exchange and converges fastest.

**5.** Anti-entropy compares and reconciles the full state with a random peer, guaranteeing eventual
convergence at higher cost. Rumour mongering spreads only recent updates and stops after a while; cheaper, but
a node may miss an update, so it's often backed by periodic anti-entropy.

**6.** Other nodes may still hold the dead node's old entry. If you delete it immediately, a later gossip
message containing it looks like news and reintroduces the node as if alive. Keeping a "dead" record (a
tombstone) long enough for all views to learn the death prevents that.

**7.** M pings J, but the ack doesn't arrive in time. M sends ping-req(J) to k random members. They ping J via
other paths; J acks them; they forward the ack to M. M receives an ack before the period ends and keeps J
alive. Without indirect probing, M would have suspected J because of a single bad link.

**8.** A counter only the node itself increments. When J learns it's suspected (at incarnation i), it gossips
alive(J, i+1), which overrides suspect(J, i). It prevents stale suspicion messages from killing a node that has
proven it's alive, and orders alive/suspect claims about the same node.

**9.** Updates are piggybacked on the detector's existing ping, ping-req and ack messages. Each update is
attached to a bounded number of outgoing messages (~λ log N), which is enough for infection-style spread to
reach every node with high probability, at no extra message cost.

**10.** The stable-link node is suspected first. Its history has small variance, so a gap of a few seconds is
very unlikely for a live node and φ rises fast. The jittery node's history includes long gaps, so φ rises
slowly. That's desirable: on the jittery link a long gap is less evidence of failure, so the detector avoids
false positives while still detecting fast on stable links.

**11.** Likely GC pauses or overloaded nodes delaying heartbeat sending/processing, or network congestion
increasing latency variance, pushing φ past `phi_convict_threshold`. Look at GC logs and pause times, CPU and
thread-pool saturation, network latency/drops, and gossip-stage backlog. Fix the cause; raising the threshold
only trades for slower detection.

**12.** Gossip views are only eventually consistent; two nodes can disagree. For routing that's harmless (retry
another replica). For ownership, disagreement means two primaries (split brain) or nobody. Ownership needs one
agreed, ordered decision, which needs consensus.

**13.** Most "dead" nodes are actually briefly unreachable (reboot, network blip, long GC). Re-replicating
terabytes immediately would cost huge network and disk I/O, could overload the remaining nodes and cause more
apparent failures (a cascade), and becomes wasted work when the node comes back.

**14.** Seeds give new nodes a known contact point to bootstrap their membership list, and help re-merge
partitioned gossip groups. They are not a single point of failure: there are several, they're only needed to
join, and after joining, nodes gossip with anyone.

**15.** Indirect probes mean a suspicion isn't based on the slow node's view alone; the suspect also gets a
chance to refute with a higher incarnation number before being declared dead. Lifeguard additionally makes a
node that sees signs of its own slowness (missed acks, late responses to itself) lengthen its timeouts and
be less eager to suspect others.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What is cluster membership?
**A:** Each node's view of which nodes are in the cluster and their state (alive, suspect, dead).

**Q:** Why is membership hard?
**A:** Can't tell slow from dead; nodes see different things; changes are continuous; mistakes cascade; detection must scale.

**Q:** Failure detector completeness vs accuracy?
**A:** Completeness: crashed nodes are eventually suspected. Accuracy: live nodes aren't suspected.

**Q:** Core trade-off of any failure detector?
**A:** Detection speed vs false-positive rate.

**Q:** Why doesn't all-to-all heartbeating scale?
**A:** O(N²) messages per interval.

**Q:** Gossip in one sentence?
**A:** Each round, each node exchanges state with a few random peers; information spreads epidemically.

**Q:** Gossip convergence time?
**A:** O(log N) rounds with high probability.

**Q:** Per-node message load of gossip?
**A:** Constant per round, independent of N.

**Q:** Push vs pull gossip strengths?
**A:** Push: fast early. Pull: fast late. Push-pull: both.

**Q:** Anti-entropy?
**A:** Periodically reconcile full state with a random peer (often using Merkle trees).

**Q:** Rumour mongering?
**A:** Spread only recent updates for a limited number of rounds; cheaper, may miss nodes.

**Q:** Heartbeat-counter gossip failure rule?
**A:** If a node's counter hasn't increased for T_fail, suspect it; remove after a further cleanup period.

**Q:** SWIM's two components?
**A:** Failure detection (ping, ping-req) and dissemination (piggybacked gossip).

**Q:** What does SWIM's ping-req do?
**A:** Asks k other nodes to probe the target, so one bad link doesn't cause a false suspicion.

**Q:** SWIM suspicion states?
**A:** Alive → suspect → dead (confirmed), with refutation possible while suspect.

**Q:** SWIM incarnation number?
**A:** Counter incremented only by the node itself to refute suspicion; higher incarnation alive overrides older suspect.

**Q:** What is Lifeguard?
**A:** HashiCorp's SWIM extensions: a node aware of its own slowness becomes less eager to suspect others.

**Q:** Phi-accrual output?
**A:** A continuous suspicion level φ = −log₁₀(P(heartbeat would be this late if alive)).

**Q:** φ = 3 means?
**A:** About a 0.1% chance a live node would be this late.

**Q:** Cassandra's default phi threshold?
**A:** 8.

**Q:** Why is phi-accrual adaptive?
**A:** It learns the heartbeat inter-arrival distribution; jittery links raise φ more slowly.

**Q:** Gossip vs consensus for membership?
**A:** Gossip: scalable, eventually consistent liveness. Consensus: agreed authoritative view for ownership decisions.

**Q:** Seed nodes?
**A:** Known addresses new nodes contact to bootstrap membership; not otherwise special.

**Q:** Why delay re-replication after a node is marked dead?
**A:** Most failures are transient; immediate data movement is expensive and can cascade.
