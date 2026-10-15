# 48 — Fault Isolation, Multi-Region & Disaster Recovery

> Part 8 — Reliability & Operations · Prerequisites: topics 23, 24, 25, 29, 44, 47

Topics 43–47 dealt with failures one request, one dependency, or one deployment at a time. This topic
zooms out to the shape of the whole system. Some failures are big: a bad config pushed everywhere, a
"poison" request that crashes every server it touches, a datacenter losing power, a region-wide cloud
outage, a ransomware attack or an operator deleting a production table. The questions become: **how much
of the system can one failure take down** (fault isolation), **how do we keep serving if an entire region
disappears** (multi-region), and **how much data can we lose and how long can we be down** when the worst
happens (disaster recovery). The ideas here, from failure domains and cells to shuffle sharding, RPO and
RTO, are mostly about deliberately limiting how far any failure can spread, and being honest about what
replication does and does not protect against.

By the end of this topic you should be able to:
1. Explain failure domains and blast radius, and identify the failure domains of a typical deployment.
2. Explain cell-based architecture: how it works, what it isolates, and what it costs.
3. Explain shuffle sharding and calculate why it reduces the impact of a poison client.
4. Compare active-passive and active-active multi-region designs, including data replication, failover,
   and consistency trade-offs.
5. Define RPO and RTO, and explain the DR strategy spectrum (backup/restore, pilot light, warm standby,
   active-active).
6. Explain why replication is not a backup, and what makes backups actually restorable.

---

## 1. Failure domains and blast radius

### 1.1 Definitions

A **failure domain** is a set of components that can fail together because they share something: power,
network, hardware, software version, configuration, or a dependency.

**Blast radius** is how much of the system (users, requests, data, features) a single failure affects.

Fault isolation means arranging the system so that each failure domain's blast radius is small, and so that
redundant copies of anything important live in *different* failure domains.

### 1.2 The physical hierarchy

```
 ┌───────────────────────────── Region (e.g., eu-west) ─────────────────────────────┐
 │  ┌──── Availability Zone A ────┐ ┌──── AZ B ────┐ ┌──── AZ C ────┐                │
 │  │ separate building(s), power,│ │              │ │              │                │
 │  │ cooling, network            │ │              │ │              │                │
 │  │  ┌ rack ┐ ┌ rack ┐          │ │  ┌ rack ┐    │ │  ┌ rack ┐    │                │
 │  │  │host  │ │host  │          │ │  │host  │    │ │  │host  │    │                │
 │  │  │ VMs/ │ │      │          │ │  │      │    │ │  │      │    │                │
 │  │  │ pods │ │      │          │ │  │      │    │ │  │      │    │                │
 │  └─────────────────────────────┘ └──────────────┘ └──────────────┘                │
 │  AZs are a few ms apart: synchronous replication across them is practical          │
 └───────────────────────────────────────────────────────────────────────────────────┘
       regions are tens to hundreds of ms apart: synchronous replication is expensive
```

| Domain | Fails together because of | Typical mitigation |
|---|---|---|
| Process / container | Bug, OOM, crash | Multiple instances, restarts |
| Host | Hardware fault, kernel panic, noisy neighbor | Spread replicas across hosts |
| Rack | Top-of-rack switch, power distribution | Rack-aware placement (e.g., Cassandra/HDFS rack awareness) |
| Availability zone | Power, cooling, flooding, network partition | Multi-AZ deployment, synchronous replication across AZs |
| Region | Regional network or control-plane failure, natural disaster | Multi-region (section 4) |
| Provider | Global control-plane or identity outage | Multi-cloud (rare; very costly) |

### 1.3 Logical failure domains

Physical separation isn't enough. Many large outages hit components that were physically redundant but
**logically shared**:

- **Same software version:** a bug triggered by a date (leap second, leap day), a certificate expiry, or a
  specific input crashes every instance of the same binary at once. Redundancy of identical copies doesn't
  help against a deterministic bug.
- **Same configuration:** a global config push (topic 47) hits every AZ and region simultaneously.
- **Shared dependency:** a global auth service, DNS provider, or configuration store that every region uses.
- **Shared control plane:** the system that creates/configures resources (provisioning APIs) may be
  regional or global even when the data plane (serving traffic) is redundant.
- **Same input:** a poison request or a single misbehaving client sent to every server (section 3).

So fault isolation has to address both: spread copies across physical domains, *and* avoid letting software,
config, dependencies or clients couple those domains together. Staged rollouts (topic 47) exist precisely so
that a code or config change is never in all failure domains at once.

### 1.4 Static stability

A system is **statically stable** if, when a dependency fails, it keeps working in its current state without
needing to make changes. Example: an application deployed in three AZs with enough capacity in each to
absorb the loss of one AZ *already provisioned*, rather than relying on autoscaling to launch new
instances after the failure. Why it matters: during a zone failure, the control plane (autoscaling,
provisioning APIs) is often impaired too, exactly when you need it. Similarly: cache config locally so
services keep working if the config service is down; keep DNS records pointing at healthy endpoints rather
than depending on a DNS change to fail over.

A related principle: prefer that the **data plane** (serving requests) does not depend on the **control
plane** (making changes) at request time.

---

## 2. Cell-based architecture

### 2.1 The idea

In a conventional design, one big fleet of each service serves all customers. A bug, a poison request, or
an overload anywhere can affect everyone. **Cell-based architecture** splits the whole stack into multiple
independent, complete copies called **cells**, each serving a subset of customers.

```
                       ┌─────────────────────┐
  requests ───────────►│  thin routing layer │  maps customer → cell
                       └──────────┬──────────┘
            ┌─────────────────────┼─────────────────────┐
            ▼                     ▼                     ▼
   ┌────────────────┐   ┌────────────────┐   ┌────────────────┐
   │ CELL 1         │   │ CELL 2         │   │ CELL 3         │
   │ app servers    │   │ app servers    │   │ app servers    │
   │ caches         │   │ caches         │   │ caches         │
   │ queues         │   │ queues         │   │ queues         │
   │ database       │   │ database       │   │ database       │
   │ customers A–H  │   │ customers I–P  │   │ customers Q–Z  │
   └────────────────┘   └────────────────┘   └────────────────┘
     no shared state or dependencies between cells
```

Each cell is a full, self-contained deployment (often spanning multiple AZs internally). Cells don't call
each other. A failure inside a cell (bad deploy, poison workload, database overload, noisy customer) affects
only that cell's customers: **blast radius = 1/N of customers**.

It's a bulkhead (topic 44) at the scale of the whole system, and it's essentially partitioning (topic 25)
applied to the entire stack rather than just the database.

### 2.2 Benefits

- **Bounded blast radius** for bugs, overload and poison inputs.
- **Natural deployment waves** (topic 47): deploy to one cell, watch, then the next. A bad change hurts
  1/N of customers at most, while it bakes.
- **Scale by adding cells** rather than growing one ever-larger fleet; each cell has a known, tested maximum
  size, so you avoid hitting untested scaling limits.
- **Isolation of large customers:** a huge tenant can get its own cell.
- **Testability:** a cell's maximum size can be load-tested; the whole system is "N tested cells".

### 2.3 Costs and hard parts

- **The routing layer** must be extremely simple and reliable, since it's shared by all cells. It should do
  as little as possible: look up the customer's cell (from a mapping, often cached) and forward.
- **Cell mapping:** by customer ID hash, or an explicit mapping table (more flexible, allows moving
  customers).
- **Cross-cell operations:** anything that needs data from multiple customers (global search, analytics,
  "all users" features) doesn't fit. These are built separately, usually asynchronously (topic 39 CDC into a
  global analytics store).
- **Migrating customers between cells** (rebalancing, a customer outgrowing a cell) is a data migration
  with all of topic 47's care.
- **Efficiency:** each cell needs its own headroom; many small fleets have less statistical multiplexing
  than one big one.
- **Operational overhead:** N copies of everything to deploy, monitor and upgrade; needs strong automation.

Cell size is a trade-off: smaller cells give smaller blast radius but more overhead; larger cells are more
efficient but each failure hurts more.

---

## 3. Shuffle sharding

### 3.1 The problem it solves

Suppose a service has 8 workers, and customers are assigned by simple sharding: each customer goes to 2
workers (for redundancy). With 4 disjoint pairs, every customer shares *both* its workers with roughly 1/4
of all other customers.

Now one customer sends a **poison workload**: requests that crash the worker, or that consume all its
resources. The poison customer's requests retry onto its second worker and crash that too. Every customer
assigned to the same pair is completely down: 25% of customers lost.

### 3.2 The mechanism

**Shuffle sharding** assigns each customer a *random combination* of workers (a "shuffle shard") rather than
one of a few fixed groups. With 8 workers and shards of 2, there are C(8,2) = 28 possible combinations.

```
 workers:      W1  W2  W3  W4  W5  W6  W7  W8

 customer A:   ●           ●                        {W1, W4}
 customer B:       ●               ●                {W2, W6}
 customer C:   ●                       ●            {W1, W7}
 customer D:               ●   ●                    {W4, W5}
 ...

 A is poison → W1 and W4 crash
 C still has W7, D still has W5 → degraded but up (as long as clients retry on the other worker)
 only a customer assigned exactly {W1, W4} is fully down: 1/28 of customers
```

The probability that another customer shares **all** of the poison customer's workers drops dramatically.

### 3.3 The numbers

With *n* workers and shard size *k*, the number of possible shards is C(n, k). Some examples:

| Workers n | Shard size k | Combinations C(n,k) | Fraction of customers fully sharing a given shard |
|---|---|---|---|
| 8 | 2 | 28 | ~3.6% |
| 8 | 4 | 70 | ~1.4% |
| 100 | 5 | 75,287,520 | ~0.0000013% |

Amazon's Route 53 example (described in the Builders' Library) used 2,048 virtual name servers and shuffle
shards of 4, giving hundreds of billions of combinations: effectively every customer has a unique set, so a
DDoS against one customer's domain affects others only partially and only on overlapping servers.

Compare with simple sharding at the same redundancy: blast radius = k/n of customers (whole groups); with
shuffle sharding, the blast radius of *complete* failure is ~1/C(n,k), and partial overlap is tolerable
if clients retry across the shard's members (topic 43) so a customer with one healthy worker remaining still
gets served.

### 3.4 Requirements and limits

- Clients (or the router) must **retry/fail over** across the workers in their shard; otherwise partial
  overlap still causes failures.
- It isolates *per-customer* problems (a poison client, a DDoS on one tenant, one noisy neighbor). It does
  not help against problems that affect all workers (a bad deploy to all, overall overload).
- Assignment must be stable (persisted or deterministic hash-based) so customers don't move constantly.
- Combined with cells: shuffle-shard customers across cells, or within a cell's workers.

---

## 4. Multi-region

### 4.1 Why go multi-region

- **Survive a region failure:** regional outages of cloud providers do happen (network, power, control plane).
- **Latency:** serve users from a nearby region (topic 02's cross-continent round trips are 100+ ms).
- **Data residency:** regulations that require data to stay in a jurisdiction.

The cost is significant: more infrastructure, cross-region data transfer, and above all the hard problem of
data that lives in more than one place, with distances that make synchronous coordination slow (topic 29:
PACELC, the latency side of the trade-off even when there's no partition).

### 4.2 Active-passive (primary / standby)

One region (**active**/primary) serves all traffic and holds the authoritative data. Another region
(**passive**/standby) receives replicated data and stands ready to take over.

```
       users
         │ (DNS / global LB points at primary)
         ▼
 ┌─────────────────┐   async replication    ┌─────────────────┐
 │ REGION A        │ ─────────────────────► │ REGION B        │
 │ active          │   (lag: seconds)       │ passive         │
 │ app + DB leader │                        │ DB replica      │
 └─────────────────┘                        │ app: scaled down│
                                            │ or off          │
                                            └─────────────────┘
 failover: promote B's replica to leader, scale up B's apps, redirect traffic to B
```

**Replication** is usually asynchronous (topic 23): synchronous replication across regions adds a
cross-region round trip (tens to hundreds of ms) to every write. Consequence: at failover, the writes that
hadn't replicated yet are **lost** (or stuck in the failed region until it returns). That loss window is
the RPO (section 5).

**Failover steps** and their pitfalls:

1. **Detect** the region is down. Not trivial (topic 26/32): is region A down, or is the monitoring path
   between regions broken? Automatic failover on a false detection can be worse than the outage.
2. **Decide** to fail over: often a human decision for region-level events, precisely because of the risk
   above and the data loss involved.
3. **Promote** the standby database. Must **fence** the old primary (topic 31) so that if it comes back it
   doesn't keep accepting writes: otherwise **split brain**, two primaries with diverging data.
4. **Scale up** standby compute (if it was scaled down), warm caches.
5. **Redirect traffic:** DNS change (slow, TTLs, topic 04) or global load balancer / anycast (faster, topic 09).
6. **Fail back** later: re-sync the original region (including reconciling writes that were stranded there)
   and switch back, which is a second risky operation.

**Pros:** simpler data model (one writer, no conflicts), cheaper if the standby is scaled down.
**Cons:** standby capacity is idle (or under-provisioned and must scale during the emergency); failover is
rarely exercised and often fails when needed; possible data loss; RTO of minutes to hours.

A common rule: **if you don't regularly test failover, you don't have failover.** Teams run game days or
periodically serve from the standby region for real.

### 4.3 Active-active

All regions serve live traffic. Users are routed to a nearby region (GeoDNS, anycast, topic 04/14).

```
          users (EU)                     users (US)
              │                              │
              ▼                              ▼
     ┌─────────────────┐  replication  ┌─────────────────┐
     │ REGION EU       │ ◄───────────► │ REGION US       │
     │ serves traffic  │               │ serves traffic  │
     └─────────────────┘               └─────────────────┘
 region failure: route its users to the other region (which must have spare capacity)
```

**Pros:** no idle capacity; failover is just routing away from a region (and that path is exercised
constantly); lower latency for users near each region; RTO can be seconds to minutes.

**Cons:** the data problem. Where do writes go? Options:

| Approach | How | Trade-off |
|---|---|---|
| **Partition by home region** | Each user/record has a home region that owns its writes (e.g., EU users' data mastered in EU). Other regions forward writes or read replicas. | No conflicts; but a region failure makes its users' data unavailable for writes (or requires failing over ownership, i.e., active-passive per partition). Cross-region operations are complex. |
| **Multi-leader replication** | Every region accepts writes, replicates asynchronously to others (topic 24) | Conflicts when the same record is written in two regions concurrently; needs conflict resolution: last-writer-wins (loses data), CRDTs (topic 33), app-level merges |
| **Globally consistent database** | A consensus-based database spanning regions (topic 30; e.g., Spanner, CockroachDB) | Writes pay cross-region consensus latency; strong consistency |
| **Leaderless with quorums** | Dynamo-style (topic 24), e.g., Cassandra with per-datacenter consistency levels | Tunable; `LOCAL_QUORUM` keeps latency low but cross-region reads may be stale |

This is CAP/PACELC (topic 29) made concrete: during a partition between regions, either regions keep
accepting writes and diverge (availability; resolve later) or some refuse writes (consistency). Even without
partitions, consistency costs cross-region latency.

**Capacity:** with N regions in active-active, each must have headroom to absorb a failed region's load.
With 2 regions, each must be able to handle 100% of traffic, so each runs at ≤50% normally; with 3 regions,
each must handle 150% of its normal share, so ≤67%. More regions = less wasted headroom, more complexity.

**Hidden coupling:** "active-active" systems often still have a single-region dependency (a global auth
database, a primary-only write path for one table, a control plane in one region). Region failure then takes
everything down anyway. Fault isolation (section 1.3) must be audited per dependency.

### 4.4 Comparison

| | Active-passive | Active-active |
|---|---|---|
| Traffic | One region | All regions |
| Writes | One writer region | Partitioned, multi-leader, or global consensus |
| Conflicts | None | Possible (unless partitioned or consensus) |
| Idle capacity | Standby (can be scaled down) | Headroom in every region |
| Failover | Promote + redirect; risky, rarely tested | Route away; exercised constantly |
| Typical RTO | Minutes to hours | Seconds to minutes |
| Data loss on failover | Async lag (RPO > 0) | Depends on replication; async → same issue |
| Complexity | Lower | Much higher (data model, conflicts) |

Many organizations use a mix: active-active stateless tiers and read paths, with active-passive (or
home-region partitioning) for the strongly consistent writes.

---

## 5. Disaster recovery: RPO and RTO

### 5.1 Definitions

```
                 last good          disaster              service
                 recovery point     strikes               restored
 time ─────────────────●──────────────✖─────────────────────●──────►
                       │◄─── RPO ────►│◄──────── RTO ───────►│
                       data written here is lost   downtime
```

- **RPO (Recovery Point Objective):** the maximum acceptable **data loss**, measured in time. "RPO = 5
  minutes" means after recovery, we may have lost at most the last 5 minutes of writes. Determined by how
  often/continuously data is copied elsewhere.
- **RTO (Recovery Time Objective):** the maximum acceptable **downtime**: time from the disaster to service
  restored. Determined by how fast you can detect, decide, restore/fail over and redirect.

Both are business decisions: a payments ledger may need RPO ≈ 0 and RTO of minutes; an internal analytics
dashboard may accept RPO = 24 h and RTO = 1 day. Lower values cost more, roughly exponentially.

### 5.2 The DR strategy spectrum

| Strategy | What runs in the recovery region | RPO | RTO | Cost |
|---|---|---|---|---|
| **Backup & restore** | Nothing; backups stored there (or in object storage) | Hours (backup frequency), or minutes with continuous log archiving | Hours to days (provision infra, restore data) | Lowest |
| **Pilot light** | Core data replicated continuously (DB replica); compute off or minimal | Seconds to minutes (replication lag) | Tens of minutes to hours (start/scale compute) | Low |
| **Warm standby** | A scaled-down but fully functional copy of the system, running | Seconds to minutes | Minutes (scale up, redirect) | Medium |
| **Active-active (multi-site)** | Full production in multiple regions serving traffic | ~0 with synchronous/consensus replication, else seconds | Seconds to minutes (route away) | Highest |

Moving down the table lowers RPO and RTO and raises cost and complexity. Choose per system (and per data
type within a system) according to what the business can tolerate.

### 5.3 Backups vs replication

This is the most important and most misunderstood distinction in DR.

**Replication** (topics 23–24) keeps live copies in sync. It protects against **losing a machine, zone or
region**: if the primary's disk dies, the replica has the data.

But replication faithfully copies **everything**, including mistakes:

```
 10:00:00  operator runs:  DELETE FROM orders;    (forgot the WHERE clause)
 10:00:00.05  replica 1 applies DELETE    ✓ "replicated successfully"
 10:00:00.08  replica 2 applies DELETE    ✓
 10:00:01     cross-region replica applies DELETE ✓
 → every copy is now empty. Replication worked perfectly.
```

Likewise for a bug that corrupts data, ransomware that encrypts it, a bad migration that drops a column, or a
compromised account that deletes data. **Replication is not a backup.**

**Backups** are **point-in-time copies** kept separately, so you can go back to a state *before* the problem:

| Protects against | Replication | Backups |
|---|---|---|
| Disk/host/AZ/region failure | Yes, quickly (low RPO/RTO) | Yes, slowly |
| Accidental deletion / bad migration | No (replicated instantly) | Yes |
| Application bug corrupting data | No | Yes (restore to before the bug) |
| Ransomware / malicious deletion | No | Yes, if backups are isolated and immutable |
| Gives point-in-time recovery | No (only "now") | Yes |

You need **both**: replication for availability, backups for recoverability.

Variations:

- **Delayed replica:** a replica that deliberately applies changes an hour behind (MySQL supports this). Gives
  a window to stop it before a destructive change applies. Helpful, but not a substitute for backups.
- **Point-in-time recovery (PITR):** a full base backup plus the archived write-ahead log (topic 17), so you can
  restore to any moment, e.g., 09:59:59, just before the bad DELETE. Postgres WAL archiving and most managed
  databases offer this. RPO is then limited by how often WAL segments are shipped.
- **Snapshots:** fast, storage-level copies; good, but if they live in the same account/system they may be
  deleted along with the data.

### 5.4 What makes backups real

A backup you've never restored is a hope, not a backup. Common ways backups fail when needed:

- Backup job silently failing for months (no monitoring on backup success).
- Backups that don't include everything (a new database or bucket added after the backup policy was written,
  encryption keys not backed up, configuration missing).
- Restore takes far longer than the RTO (restoring tens of TB from cold storage takes hours or days).
- Backups stored in the same account/region and deleted or encrypted by the same incident (ransomware,
  compromised credentials, region loss).
- Inconsistent backups (snapshotting several systems at different moments: orders restored to 09:00, payments
  to 08:00).

Practices:

- **Test restores regularly**, automatically, and measure the time against RTO.
- **3-2-1 rule:** at least 3 copies, on 2 different media/storage types, 1 off-site (another region/account).
- **Immutable / write-once backups** and separate credentials/accounts, so an attacker or a mistake in
  production can't delete them.
- **Monitor** backup jobs and alert on failures or unexpected size changes.
- **Document and rehearse** the DR runbook, including the human decision points (who decides to fail over?).

### 5.5 DR is a process, not a design

Many DR plans fail in practice because they're never exercised: DNS TTLs are longer than assumed, the
standby lacks a secret, capacity in the recovery region isn't available, someone with the right permission
is unreachable. Regular **game days** and **chaos experiments** (deliberately failing a zone, killing a
database primary, blocking a dependency) turn the plan into a tested capability, and they also test the
patterns of topics 43–44 under real conditions.

---

## 6. How it connects

- **Topic 04, 09, 14:** DNS, global load balancing and anycast route traffic between regions; TTLs slow
  DNS-based failover.
- **Topic 17:** WAL archiving enables point-in-time recovery.
- **Topic 23:** async replication lag determines RPO in active-passive; failover and promotion pitfalls.
- **Topic 24, 33:** multi-leader and leaderless replication for active-active; conflict resolution, CRDTs.
- **Topic 25:** partitioning by home region; cells are partitioning of the whole stack.
- **Topic 26, 32:** detecting region failure is hard; false positives cause harmful failovers.
- **Topic 29, 30:** CAP/PACELC and cross-region consensus costs drive multi-region data choices.
- **Topic 31:** fencing prevents split brain when promoting a standby.
- **Topic 39:** CDC feeds cross-cell analytics and region replication pipelines.
- **Topic 43, 44:** retries across a shuffle shard; cells and shuffle sharding are bulkheads at
  infrastructure scale.
- **Topic 46:** RPO/RTO are DR's SLOs; per-cell metrics show blast radius.
- **Topic 47:** deploy in waves per cell/AZ/region; config pushes are a logical shared failure domain.
- **Topic 53:** object storage and erasure coding underlie durable, off-site backup storage.

---

## Common misconceptions

- "We're in three AZs, so we're isolated from failures." → Physical redundancy doesn't protect against
  shared software, config, dependencies or poison inputs that hit every AZ at once.
- "Autoscaling will replace lost capacity." → During a zone failure the control plane may be impaired;
  static stability means having the capacity already.
- "Active-active is just active-passive with both turned on." → The hard part is data: where writes go,
  conflicts, and consistency vs latency across regions.
- "Our standby region means we can fail over." → Untested failover usually fails; it must be exercised.
- "Replication means we have backups." → Replication copies deletions and corruption instantly; only
  point-in-time backups let you go back.
- "We take backups, so we can recover." → Only if restores are tested, complete, isolated from the
  incident, and fast enough for the RTO.
- "RPO = 0 is just a setting." → It requires synchronous replication or consensus across failure domains,
  which costs write latency, and still doesn't cover logical corruption.
- "Shuffle sharding prevents all outages." → It limits the impact of per-customer problems; it does
  nothing against fleet-wide bugs or overall overload.
- "Smaller cells are always better." → They reduce blast radius but add overhead and reduce efficiency.

## Further reading (supplements)

- Amazon Builders' Library: "Workload isolation using shuffle-sharding" (Colm MacCárthaigh), "Static
  stability using Availability Zones", and "Minimizing correlated failures in distributed systems".
- AWS Well-Architected Framework: Reliability Pillar, and the whitepaper "Disaster Recovery of Workloads on
  AWS" (backup/restore, pilot light, warm standby, multi-site).
- AWS whitepaper / re:Invent talks on cell-based architecture ("Reducing the Scope of Impact with Cell-Based
  Architecture").
- Martin Kleppmann, *Designing Data-Intensive Applications*, chapter 5 (replication, multi-leader,
  multi-datacenter operation).
- *Site Reliability Engineering* (Google), chapter 26 "Data Integrity: What You Read Is What You Wrote".
- GitLab's public postmortem of its January 2017 database incident (accidental deletion; several backup
  mechanisms found not working).

---

## Questions

1. What is a failure domain? List the failure domains a request to a typical three-AZ web service passes
   through, including at least two logical (non-physical) ones.
2. A service runs in three AZs with plenty of redundancy, yet a single event takes it fully down. Give three
   plausible causes.
3. What does static stability mean, and why does it matter specifically during an AZ failure?
4. Describe cell-based architecture. What is the blast radius of a bad deploy if you deploy to cells one at a
   time, and what can't cells help with easily?
5. With 8 workers and simple sharding into 4 pairs, a poison customer takes down its pair. What fraction of
   customers are fully down? With shuffle sharding (shard size 2), what fraction?
6. Why does shuffle sharding require clients to retry across the workers in their shard?
7. Walk through an active-passive regional failover. Where can data be lost, and what is split brain?
8. Why do many organizations keep region failover as a human decision rather than fully automatic?
9. In active-active, what are three ways to handle writes, and what does each trade off?
10. Two regions run active-active. What is the maximum safe utilization of each, and why? What about three
    regions?
11. Define RPO and RTO. A system uses async replication to another region with typical lag of 2 s and a
    failover runbook that takes 40 minutes. What are its realistic RPO and RTO?
12. An engineer runs `DELETE FROM customers` without a WHERE clause on the primary. The database has two
    synchronous replicas and a cross-region replica. What protects you, and what doesn't?
13. Your nightly backups have been "succeeding" for a year. Name four ways they could still fail you in a
    disaster.
14. Order the DR strategies from cheapest to most expensive and explain what you gain at each step.

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** A failure domain is a set of components that can fail together because they share something. Physical:
the process, host, rack, AZ, region (and the provider). Logical: the software version deployed everywhere, the
configuration pushed to all instances, shared dependencies (auth service, DNS provider, config store), a shared
control plane, and the same client input reaching every server.

**2.** A bad code or config change pushed to all AZs at once; a deterministic bug triggered by a date, a
certificate expiry or a specific input that crashes every instance; a shared global dependency failing (DNS,
auth, a single-region database); a poison request retried across all servers; overload that hits every AZ.
Physical redundancy doesn't protect against these.

**3.** Keeping working during a dependency failure without having to make changes. During an AZ failure the
provisioning/autoscaling control plane may itself be impaired or overwhelmed by everyone trying to launch
capacity, so a design that relies on launching replacements may fail exactly then. Pre-provisioning enough
capacity in the remaining AZs (and caching config locally) means nothing needs to change to survive.

**4.** The whole stack is split into N independent, complete copies (cells), each serving a subset of
customers via a thin router; cells share no state. Deploying cell by cell limits a bad deploy to 1/N of
customers while it bakes in the first cell. Cells don't easily support cross-customer operations (global
search, analytics, features spanning customers in different cells), and the router itself is shared; also
moving customers between cells is a migration.

**5.** Simple sharding: the poison customer's pair is shared by 1/4 of customers, so 25% are fully down.
Shuffle sharding: C(8,2) = 28 combinations, so only customers with exactly the same two workers, ~1/28 ≈ 3.6%,
are fully down; others overlap on at most one worker and stay up if they fail over to the other.

**6.** Most other customers share one of the poison customer's workers. If they send requests to the dead
worker and don't retry on their other worker, they'll see failures too, and the isolation benefit is lost.
Retrying across the shard turns partial overlap into mere degradation instead of an outage.

**7.** Detect region A failure; decide to fail over; promote region B's replica, fencing A's old primary so it
can't accept writes; scale up B's compute and warm caches; redirect traffic via DNS/global LB; later re-sync and
fail back. Data loss: writes acknowledged in A but not yet replicated to B (async lag) are lost or stranded.
Split brain: if A comes back (or was never actually down, only unreachable) and still accepts writes while B is
also primary, two primaries diverge and their data must be reconciled.

**8.** Detecting region failure is ambiguous: a monitoring path failure looks like a region failure (topic 26).
Failing over accepts data loss, risks split brain, and is a heavy, rarely-practiced operation; a false-positive
automatic failover can cause a bigger outage than the original blip. A human can weigh the evidence and the
data-loss cost.

**9.** (a) Partition by home region: each record's writes go to its owner region: no conflicts, but writes for
a failed region's records are unavailable until ownership fails over. (b) Multi-leader: any region accepts
writes, replicating async: low latency and available, but concurrent writes conflict and need resolution (LWW
loses data, CRDTs constrain data types). (c) Global consensus database: strongly consistent, but every write
pays cross-region round-trip latency.

**10.** With two regions, each must absorb the other's full load if one fails, so each must handle 100% of
total traffic: ≤50% utilization normally. With three regions, losing one puts its third onto the other two,
so each must handle 1.5× its normal share: ≤67% utilization. More regions waste less headroom.

**11.** RPO: the data at risk is whatever hadn't replicated, so typically ~2 s, but lag can spike during the
very incident that triggers failover, so the realistic RPO is "seconds, possibly minutes". RTO: detection +
decision + 40-minute runbook, so likely an hour or more, unless the runbook is automated and practiced.

**12.** Not the replicas: the synchronous replicas and the cross-region replica apply the DELETE within
milliseconds to seconds; replication protects against hardware/region loss, not mistakes. What protects you:
point-in-time recovery (base backup + archived WAL) to just before the DELETE, a delayed replica if one exists
(stop it before it applies the change), or the last full backup (losing everything since).

**13.** Restores have never been tested and fail (corrupt or incompatible files); the backup doesn't include
everything (new databases, buckets, encryption keys, config); restore time is far longer than the RTO; backups
were in the same account/region and were deleted or encrypted in the same incident; backups of related systems
were taken at inconsistent times; the job "succeeded" but backed up an empty or wrong target.

**14.** Backup & restore (cheapest: only storage; RPO hours, RTO hours to days) → pilot light (continuous data
replication keeps RPO to seconds/minutes; compute must still start, RTO tens of minutes to hours) → warm standby
(scaled-down running system; RTO minutes) → active-active (full capacity in multiple regions, failover by
routing; RTO seconds to minutes, lowest RPO, highest cost and data complexity). Each step buys lower data loss
and downtime with more always-on infrastructure and complexity.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Failure domain?
**A:** A set of components that can fail together because they share something (power, network, software, config, dependency).

**Q:** Blast radius?
**A:** How much of the system or user base a single failure affects.

**Q:** Logical failure domains?
**A:** Shared software version, config, dependencies, control plane, or input that couple physically separate components.

**Q:** Static stability?
**A:** Continuing to work during a failure without making changes, e.g., capacity pre-provisioned to survive losing an AZ.

**Q:** Data plane vs control plane?
**A:** Data plane serves requests; control plane makes changes. Serving should not depend on the control plane.

**Q:** Cell-based architecture?
**A:** The whole stack split into independent complete copies, each serving a subset of customers via a thin router.

**Q:** Cells' blast radius?
**A:** About 1/N of customers per cell failure; deploys roll cell by cell.

**Q:** Hard parts of cells?
**A:** The shared router, cross-cell operations, moving customers, and efficiency/operational overhead.

**Q:** Shuffle sharding?
**A:** Assign each customer a random combination of k workers out of n so few customers share their exact set.

**Q:** Shuffle sharding blast radius?
**A:** About 1/C(n,k) of customers fully affected, versus k/n with simple grouping.

**Q:** Requirement for shuffle sharding to work?
**A:** Clients retry/fail over across the workers in their shard.

**Q:** Active-passive?
**A:** One region serves and writes; a standby receives replicated data and takes over on failover.

**Q:** Active-active?
**A:** All regions serve live traffic; failover is routing away from a region.

**Q:** Hardest part of active-active?
**A:** Writes: partition by home region, multi-leader with conflicts, or cross-region consensus latency.

**Q:** Split brain?
**A:** Two primaries accepting writes after a failover without fencing, causing divergence.

**Q:** Active-active headroom for 2 regions?
**A:** Each must handle 100% of traffic, so ≤50% utilization normally.

**Q:** RPO?
**A:** Maximum acceptable data loss, measured in time.

**Q:** RTO?
**A:** Maximum acceptable downtime from disaster to restoration.

**Q:** DR strategies, cheapest to costliest?
**A:** Backup & restore, pilot light, warm standby, active-active.

**Q:** Pilot light?
**A:** Data continuously replicated to the recovery region; compute off or minimal until needed.

**Q:** Why is replication not a backup?
**A:** It instantly copies deletions and corruption to every replica; it has no point-in-time history.

**Q:** Point-in-time recovery?
**A:** Base backup plus archived WAL, allowing restore to any moment (e.g., just before a bad DELETE).

**Q:** 3-2-1 rule?
**A:** 3 copies, 2 different media/storage types, 1 off-site.

**Q:** Why immutable, separate-account backups?
**A:** So ransomware, compromised credentials or mistakes in production can't delete them too.

**Q:** An untested backup is...?
**A:** A hope, not a backup: test restores and measure them against RTO.

**Q:** Why is region failover often a human decision?
**A:** Detection is ambiguous, failover loses data and risks split brain; false positives can be worse than the outage.
