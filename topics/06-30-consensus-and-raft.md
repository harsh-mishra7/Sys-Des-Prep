# 30 — Consensus & Raft

> Part 6 — Distributed Systems · Prerequisites: topics 23, 26, 27, 28, 29

Topics 26–29 kept running into the same wall. Failover needs everyone to agree on the new leader, or
you get split brain (topic 23). A uniqueness constraint needs everyone to agree which request came first
(topic 27). Linearizability needs a single order of operations that all replicas follow (topic 28). And
all of this has to work while messages are lost, nodes crash, and no one can tell slow from dead (topic
26). The problem underneath all of them is **consensus**: getting several nodes to agree on a value,
such that once decided the decision never changes, even when some nodes fail. This topic explains what
consensus buys you, why it's provably impossible in the purest setting (FLP) and how real systems get
around that, and then goes through **Raft** — the consensus algorithm behind etcd, Consul, CockroachDB,
TiKV, and Kafka's KRaft — in enough depth to trace its behavior through failures. It finishes with the
core idea of Paxos and the arithmetic of why `2f + 1` nodes tolerate `f` failures.

By the end of this topic you should be able to:
1. Define consensus (agreement, validity, termination, integrity) and explain how leader election, total order broadcast and membership reduce to it.
2. State the FLP impossibility result and explain how timeouts, randomization and partial synchrony get around it in practice.
3. Explain Raft's terms, leader election, log replication, commit index and safety rules, and trace them through a leader crash and a partition.
4. Explain why a leader may only directly commit entries from its own term.
5. Describe the core idea of single-decree Paxos (prepare/promise, accept/accepted) and why Raft was designed for understandability.
6. Explain why majorities make `2f + 1` nodes tolerate `f` crash failures, and what that costs.

---

## 1. What consensus is and what it solves

### 1.1 The problem

A set of nodes, each of which may *propose* a value, must *decide* on one value. Formally, a consensus
algorithm must satisfy:

- **Uniform agreement:** no two nodes decide differently.
- **Integrity:** no node decides twice (a decision is final).
- **Validity:** the decided value was proposed by some node (no trivial "always decide 0").
- **Termination:** every node that doesn't crash eventually decides.

Agreement, integrity and validity are **safety** properties; termination is **liveness** (topic 26). The
interesting part is doing this while nodes crash and messages are delayed. If you didn't need fault
tolerance, you could hard-code one "dictator" node that decides — but if it crashes, nobody can ever
decide again. Termination forces the algorithm to make progress *without* waiting for crashed nodes,
which means using a **majority (quorum)** rather than everyone.

### 1.2 Problems that are consensus in disguise

**Leader election.** "Who is the leader for term 7?" is a value the nodes must agree on. Two nodes
believing they're leader = split brain (topic 23). Consensus ensures at most one leader per term.

**Total order broadcast (atomic broadcast).** A protocol for delivering messages to all nodes such that:
(1) *reliable delivery* — if a message is delivered to one node, it's delivered to all; (2) *totally
ordered* — all nodes deliver messages in the same order. This is exactly a **replicated log**: every
replica applies the same sequence of commands, so if each replica is a deterministic state machine,
they all end in the same state. This is **state machine replication**, the foundation of fault-tolerant
databases, lock services and configuration stores.

Total order broadcast ⟺ consensus: repeated consensus decides "what's entry #1, entry #2, ..." of the
log; conversely, given total order broadcast, you can solve consensus by broadcasting proposals and
deciding the first one delivered.

Total order broadcast also gives **linearizable** storage (topic 28): for a uniqueness claim, append
"claim username alice" to the log, wait until it's delivered back to you, and check whether the first
claim for "alice" in the log was yours. Every node sees the same first claim. (Linearizable reads need
an extra step; section 5.)

**Atomic commit.** All participants of a distributed transaction must agree to commit or abort (topic
34). It's a variant of consensus (with the extra rule that any "no" vote forces abort).

**Membership.** "Which nodes are currently in the cluster?" must be agreed, or different nodes compute
different majorities. Consensus systems manage their own membership via the log (section 4.7), and
coordination services built on consensus (topic 31) provide membership for other systems.

**Locks and leases.** "Who holds lock L?" — a value to agree on (topic 31).

### 1.3 Single-leader replication is almost consensus

Topic 23's single-leader replication already gives a total order: the leader orders writes and followers
apply them in order. What's missing is **choosing the leader safely** when the old one fails. Manual
failover delegates that to a human (who is effectively the consensus algorithm). Automatic failover with
a simple timeout risks split brain. Consensus algorithms are essentially single-leader replication plus
a *safe* leader-election and log-recovery procedure.

---

## 2. FLP impossibility

### 2.1 The result

Fischer, Lynch and Paterson (1985) proved: **in an asynchronous system** (no bound on message delay, no
clocks — topic 26), **no deterministic consensus algorithm can guarantee termination if even one node
may crash.**

### 2.2 Why, intuitively

In the asynchronous model, a crashed node and a very slow node are indistinguishable — there are no
timeouts. Suppose the algorithm is in a state where the outcome is not yet determined ("bivalent": it
could still decide 0 or 1). The proof shows that an adversarial scheduler, which controls message delays,
can always delay one critical message so that the system moves into another bivalent state, and repeat
forever. At each step, the algorithm can't wait for the delayed node (it might have crashed — waiting
forever violates termination) and can't safely proceed without it (it might just be slow, and its
message would have tipped the decision the other way). So there's always an infinite execution in which
no decision is made.

Importantly, FLP doesn't say consensus *usually* fails — just that you can't *guarantee* termination in
every possible execution.

### 2.3 How real systems get around it

FLP's assumptions are: fully asynchronous, deterministic, guaranteed termination. Relax any one:

1. **Partial synchrony + timeouts** (Raft, Paxos, Zab). Assume the system is *eventually* synchronous
   (topic 26). Use timeouts to suspect failures. **Safety never depends on timing**, so wrong suspicions
   can't cause disagreement; they just cost liveness (e.g. an unnecessary election). Once delays are
   bounded long enough, a leader is elected and progress resumes. During bad periods (repeated elections,
   partitions), the system may stall — FLP's infinite execution is possible but unlikely and not
   sustained.
2. **Randomization.** Randomized algorithms (Ben-Or) terminate with probability 1. Raft uses randomized
   election timeouts to make repeated split votes vanishingly unlikely.
3. **Failure detectors.** Chandra and Toueg showed consensus is solvable with an "eventually accurate"
   failure detector (◇S/Ω) — again, an abstraction of timeouts that are eventually right.

So: **safe always, live when the network behaves.** That's the consensus deal.

---

## 3. Quorums: why `2f + 1` nodes tolerate `f` failures

### 3.1 The arithmetic

Consensus decisions require a **majority quorum**: more than half the nodes. With `n = 2f + 1` nodes:

- A majority is `f + 1` nodes.
- If `f` nodes crash, `f + 1` remain — still a majority, so the system can make progress.
- **Any two majorities intersect** in at least one node: `(f + 1) + (f + 1) = 2f + 2 > 2f + 1`.

The intersection property is what makes it *safe*: anything decided by one majority is known to at least
one member of every future majority, so a later decision can always learn about an earlier one. In
particular, two different leaders can't both be elected for the same term (each needs votes from a
majority, and the overlapping node votes only once per term). And during a partition, at most one side
can have a majority, so split brain is impossible.

### 3.2 The table

| Nodes (n) | Majority | Tolerated crash failures (f) |
|---|---|---|
| 1 | 1 | 0 |
| 2 | 2 | 0 |
| 3 | 2 | 1 |
| 4 | 3 | 1 |
| 5 | 3 | 2 |
| 6 | 4 | 2 |
| 7 | 4 | 3 |

Observations:

- **Even numbers don't help.** 4 nodes tolerate 1 failure, like 3, but need a bigger quorum (slower, more
  ways to lose quorum). A 2-node cluster tolerates *zero* failures — worse than 1 node, since either
  failure halts it. A 4-node cluster split 2/2 has no majority on either side.
- **Typical sizes are 3 or 5.** 3 tolerates one failure; 5 tolerates two, letting you do maintenance on
  one node while still surviving another failure. 7+ is rare: every write must reach more nodes, so
  latency and leader load grow, with diminishing benefit.
- **Placement matters.** 3 nodes in one datacenter don't survive a datacenter outage. 5 nodes across 3
  zones (2-2-1) survive losing any one zone. Across regions, every commit pays a cross-region round trip
  (topic 29's EC).

### 3.3 Byzantine contrast

Tolerating `f` *Byzantine* nodes needs `3f + 1` (topic 26), because a quorum of `2f + 1` must overlap any
other quorum in at least `f + 1` nodes, so that honest nodes outnumber liars in the overlap. Raft and
Paxos assume crash faults only.

---

## 4. Raft in depth

Raft (Ongaro & Ousterhout, "In Search of an Understandable Consensus Algorithm", 2014) implements a
replicated log. It decomposes consensus into three mostly independent subproblems: **leader election**,
**log replication**, and **safety**, plus membership changes and log compaction.

### 4.1 Roles and terms

Each node is in one of three states:

```
                 times out,                     receives votes
                 starts election                from majority
   ┌──────────┐ ───────────────▶ ┌───────────┐ ───────────────▶ ┌────────┐
   │ Follower │                  │ Candidate │                  │ Leader │
   └──────────┘ ◀─────────────── └───────────┘                  └────────┘
        ▲        discovers current leader or new term  │              │
        │        ◀─────────────────────────────────────┘              │
        └──────────────── discovers server with higher term ──────────┘
```

- **Follower:** passive; responds to the leader and candidates.
- **Candidate:** trying to become leader.
- **Leader:** handles all client requests (followers redirect clients to it), replicates the log.

Time is divided into **terms**, numbered with consecutive integers. Each term starts with an election;
if a candidate wins, it is leader for the rest of the term. A term can end with no leader (split vote);
then a new term starts. **At most one leader per term.**

```
 term:   1          2       3        4
       ├─election─┤normal─┼─elect.─┼(split vote, no leader)┼─election─┤ normal ...
```

Terms act as a **logical clock** (topic 27): every message carries the sender's term.

- If a node sees a higher term than its own, it updates its term and becomes a follower (so a stale
  leader steps down as soon as it hears from anyone in a newer term).
- If a node receives a request with a stale (lower) term, it rejects it.

**Persistent state** on every node, written to disk before responding to RPCs (crash-recovery model,
topic 26): `currentTerm`, `votedFor` (whom it voted for in the current term), and `log[]`.
**Volatile state:** `commitIndex` (highest entry known committed), `lastApplied` (highest applied to the
state machine). Leaders also keep, per follower, `nextIndex` (next entry to send) and `matchIndex`
(highest entry known replicated there).

Raft uses two RPCs: **RequestVote** (candidates collect votes) and **AppendEntries** (leaders replicate
entries; an empty AppendEntries is a **heartbeat**).

### 4.2 Leader election

Leaders send periodic heartbeats (e.g. every 50 ms). A follower that hears nothing for its **election
timeout** (e.g. randomly chosen in 150–300 ms) assumes there's no viable leader and starts an election:

1. Increment `currentTerm`; become candidate.
2. Vote for itself (persist `votedFor`).
3. Send `RequestVote(term, candidateId, lastLogIndex, lastLogTerm)` to all others.
4. Wait for one of:
   - **Votes from a majority** → become leader; immediately send heartbeats to assert authority.
   - **AppendEntries from a leader with term ≥ own** → recognize it; become follower.
   - **Timeout with no winner** (split vote) → increment term, start a new election.

A node grants its vote if: the candidate's term ≥ its own, it hasn't already voted for someone else in
this term, and the **candidate's log is at least as up-to-date** as its own (section 4.5). **Each node
votes at most once per term** (first come, first served), and since a winner needs a majority, at most
one candidate wins per term.

**Randomized timeouts** prevent endless split votes: if all followers timed out simultaneously, they'd
all vote for themselves forever. With random timeouts, usually one node times out first, wins, and
heartbeats suppress others before they time out. This is Raft's practical answer to FLP.

The timing requirement for a stable system:

```
   broadcastTime  ≪  electionTimeout  ≪  MTBF (mean time between failures)
   (~0.5–20 ms)       (~10 ms – 1 s)       (months)
```

Election timeouts too short → spurious elections during GC pauses or network blips (topic 26's
false positives); too long → slow failover.

### 4.3 Log replication

Each log entry contains a **command** for the state machine, the **term** when the leader received it,
and its **index** (position).

```
 index:    1     2     3     4     5     6     7
 leader: [1:x←3][1:y←1][1:y←9][2:x←2][3:x←0][3:y←7][3:x←5]     (term:command)
 f1:     [1:x←3][1:y←1][1:y←9][2:x←2][3:x←0]
 f2:     [1:x←3][1:y←1][1:y←9][2:x←2][3:x←0][3:y←7][3:x←5]
 f3:     [1:x←3][1:y←1]
 f4:     [1:x←3][1:y←1][1:y←9][2:x←2][3:x←0][3:y←7]
                                              ▲
            index 6 is on leader, f2, f4 = 3 of 5 (a majority) → entries 1..6 committed (leader in term 3).
            index 7 is only on leader and f2 → not yet committed.
```

The flow for a client command:

1. Client sends command to the leader.
2. Leader appends it to its own log (persisted).
3. Leader sends `AppendEntries` to all followers in parallel, retrying indefinitely until they all
   eventually store it.
4. Once the entry is stored on a **majority** (leader included), it's **committed**. The leader advances
   `commitIndex`, applies the entry to its state machine, and returns the result to the client.
5. The leader includes `commitIndex` in subsequent AppendEntries (and heartbeats); followers apply
   committed entries in order.

A slow or crashed follower doesn't block commits as long as a majority responds — that's how `f`
failures are tolerated.

### 4.4 The consistency check and log repair

`AppendEntries` carries `prevLogIndex` and `prevLogTerm`: the index and term of the entry immediately
preceding the new ones. A follower **rejects** the request unless its log contains an entry at
`prevLogIndex` with term `prevLogTerm`.

This gives the **Log Matching Property**:

- If two logs have an entry with the same index and term, they store the same command (a leader creates
  at most one entry per index in a term, and never moves entries).
- If two logs have an entry with the same index and term, the logs are **identical in all preceding
  entries** (by induction via the consistency check).

When a follower's log diverges (it missed entries, or has extra uncommitted entries from a deposed
leader), the leader repairs it:

```
 leader (term 8): [1][1][1][4][4][5][5][6][6][6]
 follower:        [1][1][1][4][4][4][4]                 ← has stale term-4 entries at 6,7

 leader sends AppendEntries(prevLogIndex=10, ...) → reject
 leader decrements nextIndex, retries ... → reject
 ... prevLogIndex=5, prevLogTerm=4 → match!
 follower deletes its entries from 6 onward, appends leader's 6..10
```

The leader never deletes or overwrites its own log (**leader append-only**); followers' conflicting
entries are overwritten by the leader's. (Optimization: followers return the conflicting term and its
first index, so the leader can skip back a whole term at a time.)

Overwriting entries sounds dangerous — what if they were committed? Section 4.5 shows they can't be.

### 4.5 Safety: the election restriction

The key safety property is **Leader Completeness**: *if an entry is committed in some term, it is present
in the logs of the leaders of all higher terms.* Without it, a new leader lacking a committed entry would
overwrite it on followers — losing an acknowledged write.

Raft ensures it with the **election restriction**: a voter denies its vote if its own log is **more
up-to-date** than the candidate's. "More up-to-date" compares the last entries:

- The log whose last entry has the **higher term** is more up-to-date.
- If the last terms are equal, the **longer** log is more up-to-date.

Why it works: a committed entry is stored on a majority. A winning candidate needs votes from a majority.
The two majorities overlap in at least one node, which has the committed entry, and that node refuses to
vote for any candidate whose log is less up-to-date than its own — so the winner's log must contain the
entry (via Log Matching). Hence every new leader already has every committed entry, and data only ever
flows from leader to followers.

### 4.6 Committing entries from previous terms (the subtle rule)

A leader knows an entry from its **current term** is committed once it's on a majority. But Raft
**never commits entries from previous terms by counting replicas**. Here's why (the "Figure 8" scenario
in the Raft paper):

```
 five servers S1..S5; numbers = term of the entry at index 2

 (a) S1 leader (term 2), replicates index 2 [term 2] to S2 only, then crashes.
     S1:[1][2]  S2:[1][2]  S3:[1]  S4:[1]  S5:[1]

 (b) S5 elected leader in term 3 (votes from S3, S4, itself), appends index 2 [term 3],
     crashes before replicating.
     S5:[1][3]

 (c) S1 restarts, elected in term 4 (votes S2, S3), continues replicating its index 2 [term 2]
     to S3. Now [term 2] at index 2 is on S1, S2, S3 — a majority!  Is it committed?

 (d) If S1 crashes now, S5 can be elected in term 5 (its last term 3 > 2, so S2,S3,S4 vote for it)
     and overwrite index 2 on everyone with [term 3].
     → an entry that was on a majority got overwritten.
```

So "on a majority" is not enough for an *old-term* entry. The rule: **a leader only advances
`commitIndex` by counting replicas of an entry from its own current term.** Once a current-term entry is
committed, all preceding entries are committed indirectly (Log Matching). In (c), S1 must first replicate
a term-4 entry to a majority; then S5 (last term 3) can never win an election, and index 2 is safe.

This is why new Raft leaders immediately append a **no-op entry** in their term: committing it commits
everything before it, and lets the leader learn its true commit index quickly.

### 4.7 Summary of Raft's safety properties

| Property | Meaning |
|---|---|
| Election Safety | At most one leader per term |
| Leader Append-Only | A leader never overwrites or deletes entries in its log |
| Log Matching | Same index + term ⇒ same command and identical preceding logs |
| Leader Completeness | A committed entry is in the logs of all future leaders |
| State Machine Safety | If a node applied an entry at index i, no node applies a different entry at i |

### 4.8 Tracing failures

**Leader crash.**

```
 t0   Leader L (term 5) replicating; entry e at index 10 on L, A, B (majority of 5) → committed.
      Entry f at index 11 only on L.
 t1   L crashes.
 t2   A's election timeout fires first. A: term 6, candidate, requests votes.
      C and D (logs end at index 9) vote for A — A's log (last term 5, index 10) is at least as
      up-to-date. A wins with A, C, D (+ B).
 t3   A appends a no-op in term 6, replicates; C, D catch up on index 10 then 11 (no-op).
      Entry f is lost — but it was never committed, so the client never got a success.
      (The client may retry; topic 38's idempotency keys make that safe.)
 t4   L restarts as follower, sees term 6, its index 11 [term 5] conflicts and is overwritten.
```

**Partition with an old leader.**

```
 5 nodes. Leader L in term 3 with node A on one side; B, C, D on the other.

   side 1: L, A          │    side 2: B, C, D
   L keeps thinking it's │    no heartbeats → B times out, term 4, wins with B, C, D.
   leader (term 3).      │    New leader B commits writes (majority 3/5).
   Clients writing to L: │
   entries appended but  │
   never reach majority  │
   → never committed,    │
   clients time out.     │
 ─────────────── heal ────────────────
 L receives a message with term 4 → steps down to follower.
 Its uncommitted term-3 entries are overwritten by B's log.
```

No split brain: L could never commit anything, because it couldn't reach a majority. But note: if L
served **reads** from its local state during the partition, those reads could be stale — section 5.

**Disruptive rejoining node.** A node partitioned alone keeps timing out and incrementing its term (say to
term 50). When it rejoins, its high term forces the healthy leader to step down — needless disruption.
Raft's **PreVote** extension fixes this: a candidate first asks "would you vote for me?" without
incrementing its term; peers that have heard from a leader recently say no.

### 4.9 Cluster membership changes

Changing the set of nodes is dangerous: if nodes switch from old config `C_old` to new config `C_new` at
different moments, there can be an instant where a majority of `C_old` and a majority of `C_new` are
disjoint, electing two leaders.

```
  C_old = {1,2,3}, C_new = {1,2,3,4,5}
  node 1,2 still on C_old: majority = 2 of {1,2,3}  → nodes 1,2 elect leader X
  nodes 3,4,5 on C_new:   majority = 3 of {1..5}   → nodes 3,4,5 elect leader Y
```

Raft's solutions, with configurations stored as special log entries:

- **Single-server changes** (used by most implementations): add or remove one node at a time. Any
  majority of the old config and any majority of the new config (differing by one node) must overlap,
  so no disjoint majorities are possible. Do larger changes as a series of single steps.
- **Joint consensus** (original paper): a transitional config `C_old,new` in which decisions need
  majorities of *both* old and new configurations; then switch to `C_new`.

New nodes typically join as non-voting **learners** until they've caught up, so they don't slow commits
or hurt availability.

### 4.10 Log compaction (snapshots)

The log can't grow forever. Each node periodically writes a **snapshot** of its state machine (as of some
index) to disk and discards log entries up to that index, keeping the last included index and term (for
the consistency check). A follower that has fallen so far behind that the leader has discarded the
entries it needs receives the snapshot via **InstallSnapshot**. This is analogous to adding a new follower
from a backup plus log in topic 23.

### 4.11 Clients and exactly-once commands

If a leader crashes after committing a command but before replying, the client retries with another
leader and the command could be applied twice. Raft's dissertation recommends clients attach a unique
**client ID and sequence number** to each command; the state machine tracks the latest sequence number
per client and ignores duplicates (topic 38).

---

## 5. Reads in a consensus system

Writes go through the log; what about reads?

- **Read from the leader's local state, naively.** Fast but **not linearizable**: a deposed leader (as
  in 4.8's partition) doesn't know it's deposed and serves stale data.
- **Read through the log.** Append the read as a log entry, answer when it's applied. Linearizable but
  costs a disk write and replication per read.
- **ReadIndex.** Leader records its current `commitIndex`, sends a heartbeat round to confirm it's still
  leader (a majority responds), waits until it has applied up to that index, then answers from local
  state. Linearizable; one network round trip, no disk write. Can also be served by followers: a
  follower asks the leader for the read index, waits until it has applied that far, and serves locally.
- **Lease reads.** The leader assumes no other leader can exist for a period after a successful
  heartbeat round (since followers won't start elections before their election timeout). It serves reads
  locally during that lease. No extra round trip, but **relies on bounded clock drift** (topic 27) —
  safety now depends on timing, which purists avoid.
- **Stale/serializable reads.** Any replica answers from its local state — fast, possibly stale.
  etcd offers this as an option.

---

## 6. Paxos: the core idea

### 6.1 Single-decree Paxos

Leslie Lamport's Paxos ("The Part-Time Parliament", written 1989, published 1998; "Paxos Made Simple",
2001) solves consensus on a **single value**. Roles: **proposers** (propose values), **acceptors** (vote;
a majority forms a quorum), **learners** (learn the decision). A node may play all three.

Each proposal has a unique, increasing **proposal number** `n` (e.g. counter + node ID).

**Phase 1 — Prepare / Promise**

1. A proposer picks a number `n` and sends `Prepare(n)` to a majority of acceptors.
2. An acceptor that receives `Prepare(n)` with `n` greater than any number it has already promised
   **promises** not to accept proposals numbered below `n`, and replies with the highest-numbered
   proposal it has already **accepted**, if any (`n_a`, `v_a`).

**Phase 2 — Accept / Accepted**

3. If the proposer gets promises from a majority, it sends `Accept(n, v)`, where **`v` is the value of
   the highest-numbered proposal reported in the promises** — or, if none reported any, its own value.
4. An acceptor accepts `(n, v)` unless it has since promised a higher number. Once a majority of
   acceptors accept the same `(n, v)`, **`v` is chosen**; learners are informed.

```
 Proposer P (n=5)                 Acceptors A1  A2  A3
   ── Prepare(5) ─────────────▶    ✓   ✓   (✓)
   ◀── Promise(5, none) ───────    ✓   ✓
   ── Accept(5, "X") ──────────▶   ✓   ✓
   ◀── Accepted ───────────────    ✓   ✓        majority accepted → "X" chosen
```

### 6.2 Why it's safe

The crucial rule is step 3: a proposer **must adopt** a previously accepted value if any acceptor in its
majority reports one. If a value `v` was chosen (accepted by a majority with number `n`), then any later
proposer with `n' > n` contacts a majority in phase 1, which overlaps the accepting majority, so it
learns about `v` (or a later proposal that also carries `v`, by induction) and must propose `v` again.
So once chosen, a value can't be changed — the same quorum-intersection idea as Raft's election
restriction.

### 6.3 Liveness: dueling proposers

Two proposers can keep pre-empting each other: P1 prepares with 5, P2 prepares with 6 (invalidating
P1's accepts), P1 retries with 7, ... forever — FLP in action. The fix is to elect a **distinguished
proposer** (a leader) and use randomized backoff.

### 6.4 Multi-Paxos and its relatives

A replicated log needs consensus on a *sequence* of values. **Multi-Paxos** runs a Paxos instance per
log slot, but with a stable leader that runs phase 1 once for all future slots and then only phase 2
(one round trip) per entry. This is structurally very close to Raft — but the original papers left
leader election, log gaps, membership changes and recovery largely unspecified, so every implementation
(Chubby, Spanner's Paxos groups, etc.) filled the gaps differently. Google's "Paxos Made Live" (2007)
describes how hard that was.

Other members of the family: **Zab** (ZooKeeper Atomic Broadcast), **Viewstamped Replication** (Oki &
Liskov, 1988 — very similar to Raft, predating it), **EPaxos** (leaderless), **Flexible Paxos** (phase-1
and phase-2 quorums need only intersect each other, not both be majorities).

### 6.5 Why Raft was designed for understandability

Raft's authors made understandability an explicit design goal, after finding Paxos hard for students and
engineers to grasp and implement correctly. Choices that follow from that:

| Raft | Paxos (as typically presented) |
|---|---|
| Strong leader: entries flow only leader → followers | Any proposer; symmetric roles |
| Leader election is a separate, well-specified mechanism | Leader is an optimization, underspecified |
| Logs have no holes; Log Matching via consistency check | Independent per-slot instances; holes possible |
| Election restriction ensures leader has all committed entries; no log transfer to leader | New leader must learn/fill values from acceptors |
| Randomized timeouts | Unspecified |
| Membership changes, compaction, client interaction specified | Left to implementers |

In their user study, students scored better on Raft quizzes than Paxos. Raft is not more powerful or
faster than Multi-Paxos — it's a more constrained design that's easier to reason about and implement,
which is why so many open-source systems adopted it.

---

## 7. Costs and limits of consensus

- **Latency:** every write needs a round trip to a majority plus an fsync (topic 17) on each. In one
  datacenter, ~1–5 ms; across regions, 50–150+ ms.
- **Throughput:** all writes go through one leader per group. Systems scale by running many consensus
  groups, one per partition (CockroachDB ranges, TiKV regions, Kafka partitions' metadata) — topic 25.
- **Availability:** needs a majority up and connected; the minority side of a partition stalls (topic 29's
  CP).
- **Sensitive to timeouts:** in networks with variable latency or GC-prone nodes, frequent spurious
  elections degrade performance.
- **Fixed membership by default:** adding/removing nodes is an explicit, careful operation.
- **Doesn't help with Byzantine faults** or with bugs replicated on all nodes.

Because of these costs, most systems use consensus for a **small, critical piece of state** — who is
the leader, the cluster configuration, partition assignments, locks — and use cheaper replication for
the bulk data. That's the role of coordination services like ZooKeeper and etcd (topic 31).

---

## 8. How it connects

- **Topic 23**: Raft = single-leader replication + safe election + log repair; it solves split brain.
- **Topic 24**: quorum intersection reappears, but consensus adds ordering and leadership.
- **Topic 26**: the partially synchronous, crash-recovery model; persisting state before replying;
  safety independent of timing; "the truth is defined by the majority".
- **Topic 27**: terms are a logical clock; lease reads depend on bounded clock drift; total order vs
  Lamport timestamps.
- **Topic 28**: total order broadcast yields linearizable storage; read-index vs lease reads.
- **Topic 29**: consensus systems are CP — majority side continues, minority stalls; EC latency cost.
- **Topic 31** (next): etcd/ZooKeeper expose consensus as a service for locks, leases and membership.
- **Topic 32**: gossip-based membership is the scalable, non-consensus alternative.
- **Topic 34**: Spanner runs 2PC across Paxos groups; atomic commit vs consensus.
- **Topic 37**: Kafka's controller uses Raft (KRaft) for metadata.
- **Topic 38**: client IDs + sequence numbers for exactly-once command application.

---

## Common misconceptions

- **"FLP means consensus is impossible."** → It means no deterministic algorithm can *guarantee*
  termination in a fully asynchronous system with one crash. With timeouts and randomization, consensus
  works in practice: always safe, live when the network behaves.
- **"More nodes = more fault tolerance, always."** → Only odd increments help; 4 nodes tolerate the same
  1 failure as 3 and are slower. More nodes also add write latency.
- **"An entry stored on a majority is committed."** → Only for entries of the leader's current term; old-
  term entries are committed indirectly (Raft Figure 8).
- **"The leader can serve reads locally — it's the leader."** → A deposed, partitioned leader may not
  know it; linearizable reads need ReadIndex, log reads, or (clock-dependent) leases.
- **"Raft is faster/better than Paxos."** → It's essentially equivalent to Multi-Paxos with a stable
  leader; its advantage is that it's fully specified and easier to understand and implement.
- **"Consensus tolerates any failure."** → Crash faults only (with `2f+1`), needs a majority connected,
  and assumes disks don't lose acknowledged state.
- **"Use consensus for all your data."** → Typically used for small critical metadata; bulk data uses
  partitioned consensus groups or cheaper replication.
- **"Clients get exactly-once just because the log is consistent."** → A retried command after a leader
  crash can be applied twice unless the state machine deduplicates by client ID and sequence number.

---

## Further reading (supplements)

- Diego Ongaro & John Ousterhout, "In Search of an Understandable Consensus Algorithm (Extended Version)" (USENIX ATC 2014) — the Raft paper; read Figure 2 closely.
- The Raft website (raft.github.io), including the interactive RaftScope visualization; "The Secret Lives of Data" Raft animation.
- Diego Ongaro's PhD dissertation, "Consensus: Bridging Theory and Practice" (2014) — membership changes, client interaction, PreVote.
- Leslie Lamport, "Paxos Made Simple" (2001).
- Fischer, Lynch, Paterson, "Impossibility of Distributed Consensus with One Faulty Process" (1985).
- Chandra, Griesemer, Redstone, "Paxos Made Live — An Engineering Perspective" (2007).
- DDIA, Chapter 9 "Distributed Transactions and Consensus".
- MIT 6.5840 labs 3A–3D (implement Raft) and lectures on Raft.

---

## Questions

1. State the four properties of consensus. Which are safety and which is liveness?
2. Explain why total order broadcast is equivalent to consensus, and how it gives you a linearizable
   uniqueness check.
3. What does FLP prove, and what three assumptions does it rely on? How does Raft sidestep it?
4. Why does a cluster of 4 nodes tolerate no more failures than a cluster of 3? Why is a 2-node cluster
   worse than a single node for availability?
5. Why does Raft use randomized election timeouts? What happens if timeouts are too short?
6. What is persisted to disk in Raft before responding to RPCs, and what goes wrong if `votedFor` is not?
7. Explain the AppendEntries consistency check and how it establishes the Log Matching Property.
8. State the election restriction and explain, using quorum intersection, why it guarantees Leader
   Completeness.
9. Walk through the "Figure 8" scenario. Why can't a leader commit an entry from a previous term just
   because it's on a majority? What does Raft do instead?
10. A 5-node cluster's leader is partitioned together with one follower. Describe what happens on each
    side, to client writes and reads, and after the partition heals.
11. Why are changes to cluster membership dangerous, and how does adding one server at a time avoid the
    danger?
12. Compare ReadIndex reads with lease-based reads. Which depends on clocks and why?
13. In single-decree Paxos, why must a proposer adopt the highest-numbered previously accepted value it
    hears about in phase 1?
14. What design choices make Raft easier to understand than Paxos?
15. A client sends a command, the leader commits it and crashes before replying, and the client retries
    on the new leader. What can go wrong, and how is it prevented?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Uniform agreement (no two nodes decide differently), integrity (no node decides twice), validity
(the decided value was proposed by someone), termination (every non-crashed node eventually decides).
The first three are safety; termination is liveness.

**2.** Total order broadcast delivers the same messages to all nodes in the same order. Using repeated
consensus you can decide each log position, giving total order broadcast; conversely, with total order
broadcast, each node broadcasts its proposal and everyone decides on the first one delivered — the same
for all. For a uniqueness check: append "claim alice by me" to the log, wait until it's delivered back,
and check whether the first claim on "alice" in the log is yours; all nodes agree on which claim is
first, so exactly one claim succeeds.

**3.** In a fully asynchronous system, no deterministic consensus algorithm can guarantee termination if
even one process may crash. Assumptions: asynchrony (no timing bounds/timeouts), determinism,
guaranteed termination in every execution. Raft assumes partial synchrony, uses timeouts for failure
detection and randomized election timeouts, and guarantees safety always while termination happens
only when the network is well-behaved enough.

**4.** A majority of 4 is 3, so losing 2 leaves no majority — it tolerates 1 failure, the same as 3 nodes
(majority 2), while requiring larger quorums and having a 2/2 split with no majority. With 2 nodes, the
majority is 2, so any single failure halts the system; with 1 node, only that one node's failure halts
it — so 2 nodes double the chance of an outage.

**5.** If followers timed out at the same time, they'd all become candidates, vote for themselves, and
split the vote repeatedly. Randomization makes it likely one node times out first and wins before
others start. Too-short timeouts cause spurious elections during GC pauses or latency spikes,
disrupting the cluster with leader changes and lost throughput.

**6.** `currentTerm`, `votedFor`, and the log entries. If `votedFor` weren't persisted, a node could vote
for candidate A in term 5, crash, restart, and vote for candidate B in term 5 — allowing two candidates
to gather majorities in the same term, i.e. two leaders.

**7.** Each AppendEntries includes the index and term of the entry preceding the new ones. The follower
accepts only if it has an entry at that index with that term. By induction: the initial empty log
matches; each accepted append extends a log that matched the leader's up to prevLogIndex. Combined with
"a leader creates at most one entry per index per term", any two logs with an entry of the same index and
term are identical up to and including that entry.

**8.** A voter rejects a candidate whose log is less up-to-date than its own (compare last entry's term,
then length). A committed entry is on a majority; the winner received votes from a majority; the two
majorities share at least one node, which has the committed entry and would only vote for a candidate
whose log is at least as up-to-date — which implies the candidate's log contains that entry. So every
new leader has all committed entries.

**9.** S1 (term 2) puts an entry at index 2 on S1, S2, crashes. S5 becomes leader in term 3 with a
different index-2 entry, crashes. S1 becomes leader in term 4 and replicates its term-2 entry to S3 — now
on a majority. If S1 crashes, S5 (last log term 3 > 2) can win term 5 with votes from S2, S3, S4 and
overwrite index 2 everywhere. So a majority-replicated old-term entry can still be lost. Raft therefore
only commits by counting replicas of current-term entries; once one is committed, all prior entries are
committed with it. New leaders append a no-op to achieve this quickly.

**10.** The leader's side has 2 of 5 nodes: the leader keeps accepting client writes but can't replicate
them to a majority, so nothing commits; clients time out. If it serves local reads without confirming
leadership, they may be stale. The other side (3 nodes) stops receiving heartbeats, elects a new leader
in a higher term, and commits writes normally. On heal, the old leader sees the higher term, steps down,
and its uncommitted entries are overwritten by the new leader's log.

**11.** If nodes adopt the new configuration at different times, a majority of the old configuration
and a majority of the new could be disjoint, electing two leaders in the same term. When adding or
removing a single server, the old and new configurations differ by one node, so any majority of one
intersects any majority of the other — disjoint majorities are impossible. Larger changes are done as
a series of single-server steps (or with joint consensus).

**12.** ReadIndex: leader records commitIndex, confirms it's still leader by getting heartbeat acks from a
majority, waits until applied, then reads locally — one round trip, no clock assumptions. Lease reads:
after a successful heartbeat round the leader assumes no new leader can be elected for roughly an
election timeout, and serves reads locally with no round trip. It depends on clocks because it assumes
time passes at similar rates on all nodes (bounded drift); a paused or drifting leader could serve reads
after a new leader exists.

**13.** Because a value might already have been chosen (accepted by a majority). The proposer's phase-1
majority overlaps any accepting majority, so at least one acceptor reports that value (or one carried
forward by a later proposal). Adopting the highest-numbered accepted value ensures the proposer
re-proposes the chosen value rather than a different one, so a chosen value is never changed.

**14.** A strong leader with one-way log flow; leader election as a distinct, specified mechanism with
randomized timeouts; logs without holes and a simple consistency check; the election restriction that
guarantees new leaders already have committed entries; fully specified membership changes, compaction and
client interaction; fewer states and a decomposition into separate subproblems.

**15.** The command was committed and applied, but the client didn't get a reply, so the retry appends
it again and it's applied twice (e.g. double increment). Prevention: clients attach a client ID and
monotonically increasing sequence number; the state machine records the last sequence number (and
result) per client and returns the cached result for duplicates instead of reapplying.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Four properties of consensus?
**A:** Uniform agreement, integrity, validity, termination.

**Q:** Which consensus property is liveness?
**A:** Termination.

**Q:** Three problems that reduce to consensus?
**A:** Leader election, total order broadcast (replicated log), membership (also atomic commit, locks).

**Q:** What is state machine replication?
**A:** Deterministic replicas apply the same totally ordered log of commands, so they reach the same state.

**Q:** FLP result?
**A:** In an asynchronous system, no deterministic algorithm guarantees consensus terminates if one node may crash.

**Q:** How do real systems circumvent FLP?
**A:** Partial synchrony with timeouts, randomization; safety never depends on timing, liveness does.

**Q:** Nodes needed to tolerate f crash failures?
**A:** 2f + 1.

**Q:** Why do majorities work?
**A:** Any two majorities overlap in at least one node, which carries previous decisions.

**Q:** Why not 4 nodes?
**A:** Tolerates 1 failure like 3, but larger quorum and possible 2/2 split.

**Q:** Raft's three node states?
**A:** Follower, candidate, leader.

**Q:** What is a Raft term?
**A:** A numbered period starting with an election; at most one leader; acts as a logical clock.

**Q:** What happens when a Raft node sees a higher term?
**A:** It updates its term and becomes a follower.

**Q:** Raft persistent state?
**A:** currentTerm, votedFor, log.

**Q:** How does Raft avoid repeated split votes?
**A:** Randomized election timeouts.

**Q:** When is a Raft entry committed?
**A:** When a current-term entry is stored on a majority (earlier entries then commit with it).

**Q:** AppendEntries consistency check?
**A:** Follower rejects unless it has an entry at prevLogIndex with prevLogTerm.

**Q:** Log Matching Property?
**A:** Same index and term ⇒ same command and identical preceding entries.

**Q:** Election restriction?
**A:** Vote only for candidates whose log is at least as up-to-date (higher last term, or same term and longer).

**Q:** Leader Completeness?
**A:** Committed entries appear in the logs of all future leaders.

**Q:** Why do new Raft leaders append a no-op?
**A:** To commit an entry in their term, which commits all prior entries.

**Q:** What does PreVote fix?
**A:** A rejoining partitioned node with an inflated term disrupting the current leader.

**Q:** Safe way to change Raft membership?
**A:** One server at a time (or joint consensus), with new nodes first as learners.

**Q:** ReadIndex read?
**A:** Record commitIndex, confirm leadership with a majority heartbeat, wait to apply, read locally.

**Q:** Risk of lease reads?
**A:** Depend on bounded clock drift — a paused/drifting leader may serve stale reads.

**Q:** Paxos phases?
**A:** Prepare/Promise, then Accept/Accepted.

**Q:** Key Paxos safety rule?
**A:** Proposer must adopt the highest-numbered accepted value reported in phase 1.

**Q:** Multi-Paxos?
**A:** Stable leader runs phase 1 once, then one-round-trip phase 2 per log slot.

**Q:** Why Raft over Paxos?
**A:** Same power, but designed to be understandable and fully specified (strong leader, no log holes, specified elections/membership).

**Q:** How to get exactly-once commands on top of Raft?
**A:** Client ID + sequence number; state machine dedups.
