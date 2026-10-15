# 47 — Deployments & Safe Change

> Part 8 — Reliability & Operations · Prerequisites: topics 09, 16, 21, 46

Most outages are not caused by hardware failing or by traffic spikes. They're caused by **change**: a new
binary, a configuration push, a schema migration, a feature turned on. Industry postmortems repeatedly find
that the majority of incidents follow a change made by the operators themselves. That's good news,
because change is the one thing you control. This topic covers how to change a running system without
taking it down: deployment strategies that limit how many users a bad version can hurt and how fast you can
undo it (rolling, blue-green, canary), feature flags that separate deploying code from releasing behavior,
the expand/contract pattern for changing a database schema while old and new code both run against it, and
why configuration changes deserve the same care as code, since they are among the most common outage
triggers of all.

By the end of this topic you should be able to:
1. Explain why change is the leading cause of outages and the principles of safe change (small, gradual,
   observable, reversible).
2. Describe rolling, blue-green and canary deployments, and compare their cost, speed, and rollback.
3. Explain feature flags: what they decouple, their uses, and their risks.
4. Explain why every deployment creates a period of mixed versions, and what compatibility that requires.
5. Perform a zero-downtime schema change with expand/contract, step by step.
6. Explain why configuration changes cause so many outages and how to make them safe.

---

## 1. Why change is dangerous, and the principles of safe change

A running production system is in a state that has been tested by real traffic. Any change moves it to a
state that has not. Tests and staging catch many problems but never all, because production differs in
data volume, traffic mix, configuration, timing, and dependencies.

So the goal is not "never ship bugs" (impossible) but "**when a change is bad, limit the damage and undo it
quickly**". Four principles:

1. **Small:** smaller changes are easier to review, test, attribute and revert. If ten changes ship
   together and something breaks, which one was it?
2. **Gradual:** expose a change to a small fraction of traffic/servers/regions first, widening only when
   it looks healthy. This bounds the **blast radius**.
3. **Observable:** know quickly whether the change is healthy, by comparing SLIs (topic 46) for the new
   version against the old.
4. **Reversible:** have a fast, tested way back. Rollback should be a routine, low-risk operation, not a
   heroic one. Some changes (schema changes, data migrations, messages already sent) are hard to reverse;
   these need special techniques (section 5).

A useful way to think about impact: **damage ≈ fraction of users exposed × (time to detect + time to
mitigate)**. A bad change exposed to 1% of users for 5 minutes does 1% × 5 = 0.05 "full-outage minutes" of
damage; the same change exposed to 100% for an hour does 60, about 1,200 times more.

---

## 2. Deployment strategies

### 2.1 Recreate (the baseline)

Stop all old instances, start all new ones. Simple, and it guarantees no two versions run at once, but it
causes downtime and everyone gets the new version at the same time. Mostly used for development
environments or when versions genuinely cannot coexist.

### 2.2 Rolling deployment

Replace instances a few at a time. With 10 instances and a batch size of 2:

```
 step 0:  [v1][v1][v1][v1][v1][v1][v1][v1][v1][v1]
 step 1:  [v2][v2][v1][v1][v1][v1][v1][v1][v1][v1]    drain 2, replace, health-check
 step 2:  [v2][v2][v2][v2][v1][v1][v1][v1][v1][v1]
 ...
 step 5:  [v2][v2][v2][v2][v2][v2][v2][v2][v2][v2]
```

For each batch: remove the instances from the load balancer, **drain** in-flight requests (topic 09),
stop them, start the new version, wait for readiness checks to pass, add them back. The orchestrator
(e.g., a Kubernetes Deployment with `maxUnavailable` / `maxSurge`) automates this.

- **Pros:** no extra capacity needed (or only a small surge), no downtime, the default in most platforms.
- **Cons:** for the duration of the rollout, **both versions serve traffic simultaneously** (section 4); a
  given user may hit v1 then v2 then v1. Rollback is another rolling deployment, so it's as slow as the
  rollout. If a batch passes health checks but has a subtle bug (wrong results, slow leak), the rollout
  may continue to 100% before anyone notices, unless it's gated on metrics.
- **Capacity during rollout:** with 2 of 10 instances out of rotation, you're serving with 80% capacity,
  so don't roll during peak if you're near the limit, or use surge (add new before removing old).

### 2.3 Blue-green deployment

Run two complete environments. **Blue** serves all production traffic (v1). Deploy v2 to **green**, which
receives no user traffic, test it there, then switch the router/load balancer to send all traffic to green.

```
                  ┌──────────────────┐
             ┌───►│ BLUE  (v1)       │  ← live
 router ─────┤    └──────────────────┘
             │    ┌──────────────────┐
             └ ─ ►│ GREEN (v2)       │  ← idle, tested
                  └──────────────────┘

 cutover: router switches 100% to green.  rollback: switch back to blue.
```

- **Pros:** the switch is near-instant and so is **rollback** (flip back, blue is still running and warm).
  You can test v2 in a production-identical environment before users see it. No mixed versions *serving
  users* (though see the caveats).
- **Cons:** **double the capacity** during deployment. The cutover is all-at-once: if v2 has a bug that
  only real traffic reveals, 100% of users hit it (until you flip back). Green's caches and JIT are cold
  at cutover, which can cause a latency spike (topic 12). **Shared state** breaks the illusion: both
  environments use the same database, so a schema change still needs the compatibility techniques of
  section 5, and rollback after green has written data in a new format may not be clean. Long-lived
  connections (WebSockets, topic 08) don't move until they reconnect.
- **DNS-based switching** is slow and uneven because of TTLs and caching (topic 04); switching at a load
  balancer is preferred.

### 2.4 Canary deployment

Named after canaries in coal mines: expose the new version to a **small fraction** of traffic first, compare
its health with the old version, and widen gradually.

```
 stage 1:  1% → v2,  99% → v1      watch 15–30 min: errors, latency, saturation, business metrics
 stage 2:  10% → v2                  watch
 stage 3:  50% → v2                  watch
 stage 4:  100% → v2
 at any stage: if v2 looks worse → route 100% back to v1 (auto-rollback)
```

- **Pros:** a bad version hurts a small fraction of users for a short time. Real production traffic
  is the test. Rollback at early stages is cheap.
- **Cons:** slower rollouts; requires traffic splitting (weighted routing in an LB, gateway or mesh, topic
  10) and good metrics; mixed versions run for an extended time; needs enough traffic at 1% to be
  statistically meaningful.

**Canary analysis.** The crucial part is judging "worse". Compare the canary against a **baseline** running
the old version at the same time, under the same traffic mix, rather than against yesterday (time-of-day
effects) or against the whole old fleet (which includes long-running instances with warm caches; some
teams start a fresh baseline to compare like with like). Compare error rates, latency percentiles, CPU and
memory, and business metrics (checkout conversion). Automated canary analysis tools (e.g., Netflix and
Google's Kayenta) run statistical tests and gate promotion.

**Canary by what?** Random percentage of requests; specific instances; internal users first ("dogfooding");
one region or one cell (topic 48) at a time. For stateful user experiences, assign by user (stable hashing)
so one user sees a consistent version.

**Staged rollouts across regions/cells.** At large scale, a canary is followed by "waves": one cell, then one
availability zone, then one region, then the rest, with bake time between waves. This ensures no change ever
reaches everything at once, which bounds blast radius even for bugs that take hours to surface.

### 2.5 Comparison

| | Rolling | Blue-green | Canary |
|---|---|---|---|
| Extra capacity | None/small surge | 2× during deploy | Small |
| Users exposed to a bad version | Grows with rollout, possibly 100% before noticed | 100% at cutover | Small fraction, short time |
| Rollback speed | Slow (re-roll) | Instant (flip) | Fast (re-route) |
| Mixed versions serving | Yes, during rollout | Briefly / not at all | Yes, for the whole rollout |
| Needs | Health checks, draining | Router switch, two envs | Traffic splitting, metrics, analysis |

These combine: a common setup is canary first, then a rolling deployment gated on canary metrics; or
blue-green per cell with cells upgraded in waves.

### 2.6 Rollback vs roll-forward

Rolling back (to the previous known-good version) is generally the fastest and safest mitigation:
**mitigate first, debug later**. Roll-forward (shipping a fix) is tempting but means making another change
under pressure. Rollback only works if the old version is still compatible with the current state: if the
new version has migrated data or written a new format the old one can't read, rollback can make things worse.
That's why compatibility (sections 4–5) is a prerequisite for safe deployment.

---

## 3. Feature flags

### 3.1 Deploy vs release

A **feature flag** (feature toggle) is a runtime switch that decides whether a code path runs:

```
 if flags.enabled("new_checkout_flow", user):
     return new_checkout(cart)
 else:
     return old_checkout(cart)
```

This separates two things that deployments otherwise bundle:

- **Deploy:** getting new code onto servers. A technical event.
- **Release:** turning on new behavior for users. A product event.

Code can be deployed "dark" (present but off), then released gradually, to specific users, and turned off
instantly without a redeploy. A flag flip takes effect in seconds; a rollback deploy takes minutes.

### 3.2 Uses

| Kind | Purpose | Lifetime |
|---|---|---|
| **Release toggle** | Hide unfinished or unreleased features; allows trunk-based development (merge to main continuously, keep incomplete work off) | Days to weeks, then removed |
| **Progressive rollout** | Enable for 1% → 10% → 100% of users, canary-style, at the feature level | Weeks |
| **Experiment** | A/B test variants, measure impact | Length of experiment |
| **Ops toggle / kill switch** | Disable an expensive or risky feature under load (graceful degradation, topic 44) | Long-lived |
| **Permission / entitlement** | Enable features for certain plans or customers | Long-lived |

Targeting rules let a flag be on for internal users, a percentage of users (stable hashing on user ID so each
user stays in one group), one region, or particular tenants.

### 3.3 Risks of flags

- **Flags are configuration changes.** Flipping a flag on for 100% is a global, instant change, exactly the
  kind section 6 warns about. Roll out flags gradually too.
- **Combinatorial complexity:** n boolean flags give 2ⁿ possible code paths; most combinations are never
  tested. A flag interacting with another flag causes surprises.
- **Flag debt:** stale flags (the feature shipped a year ago, the flag still exists) clutter code and can
  be flipped accidentally. Removing a flag is part of finishing a feature. The well-known Knight Capital
  incident (2012) involved repurposing an old flag that reactivated dead code on a server that hadn't
  received the new deployment: hundreds of millions of dollars lost in under an hour.
- **The flag service is a dependency.** If it's unreachable, what does each flag default to? Clients
  typically cache flag values locally and use safe defaults.
- **Flags hide behavior from code review:** what runs in production depends on runtime state, not just the
  code.

---

## 4. Mixed versions: the compatibility requirement

Every strategy except "recreate" means that for some time, **old and new versions run at the same time**:

- During a rolling or canary rollout, v1 and v2 servers both handle requests.
- Clients (mobile apps especially) run old versions for months.
- After a rollback, v1 reads data that v2 wrote.
- Services deploy independently; service A's v2 talks to service B's v1.

So every change must be compatible in both directions across the deployment window:

```
 v1 server ──writes──► data / messages / cache ──read by──► v2 server     (forward: v2 reads v1's output)
 v2 server ──writes──► data / messages / cache ──read by──► v1 server     (backward: v1 reads v2's output)
```

Topic 16 covered this for encodings: **backward compatibility** (new code reads old data) and **forward
compatibility** (old code reads new data, ignoring unknown fields). The same applies to:

- **APIs:** add fields, don't remove or rename them; new required parameters break old clients (topic 07).
- **Message formats** on queues: a v2 producer's messages may be consumed by a v1 consumer (topic 36, 16).
- **Cache entries:** v2 writes a cache entry in a new shape; v1 reads it and crashes. Version the cache key
  or keep the format compatible (topic 11).
- **Database schemas:** the subject of the next section.

The general rule: **never make a change that requires two components to switch at the same instant.**
Split it into steps, each compatible with the previous one.

---

## 5. Zero-downtime schema changes: expand/contract

### 5.1 Why schema changes are hard

The database is shared by all versions of the application at once, and schema changes can't be rolled out
gradually per instance: once a column is dropped, it's dropped for everyone. If you rename a column
`name → full_name` in one step:

```
 t0  ALTER TABLE users RENAME COLUMN name TO full_name;
 t1  v1 servers (still running during rollout): SELECT name FROM users → ERROR
 t2  v2 servers: fine
 → every request on v1 servers fails until the rollout completes
 → rollback to v1 is impossible without renaming back
```

Additionally, some DDL operations lock the table (topic 22) or rewrite it entirely (on large tables, minutes
to hours), blocking reads/writes. Online schema change tools (e.g., `gh-ost`, `pt-online-schema-change` for
MySQL) and database-specific features (Postgres can add a nullable column without a rewrite; `CREATE INDEX
CONCURRENTLY` avoids blocking writes) are needed for large tables.

### 5.2 The expand/contract pattern (parallel change)

Break the change into steps where every step is compatible with the code running before and after it:

1. **Expand:** add the new structure alongside the old one, without removing anything.
2. **Migrate:** make code write to both, backfill existing data, switch reads to the new structure.
3. **Contract:** once nothing uses the old structure, remove it.

### 5.3 Worked example: rename `users.name` to `users.full_name`

```
 Step 1 (schema, expand)   ALTER TABLE users ADD COLUMN full_name TEXT NULL;
                           Old code ignores the new column. Safe. Nullable, so no table rewrite.

 Step 2 (code deploy A)    Write BOTH name and full_name on every insert/update.
                           Still READ from name.
                           Mixed v1/vA: v1 writes only name; vA writes both. Reads use name → consistent.

 Step 3 (data backfill)    UPDATE users SET full_name = name WHERE full_name IS NULL
                           in small batches (avoid long locks, replication lag, topic 23).
                           Now every row has full_name.

 Step 4 (code deploy B)    READ from full_name. Keep writing both.
                           Mixed vA/vB: both write both columns → either can read either. Safe.
                           Rollback to vA is safe (it reads name, still maintained).

 Step 5 (code deploy C)    Stop writing name. Only full_name.
                           Rollback to vB still safe (vB reads full_name).
                           (Rollback to vA would NOT be: name is now stale. Don't go back that far.)

 Step 6 (schema, contract) Once no running code (including old batch jobs, reports, other services)
                           uses name: ALTER TABLE users DROP COLUMN name;
```

Each step is individually deployable and individually reversible to the step before. It takes days instead
of minutes, which is the price of no downtime and safe rollback.

### 5.4 The same pattern elsewhere

- **Adding a NOT NULL column:** add as nullable → deploy code that always writes it → backfill → add the
  NOT NULL constraint (in Postgres, add it as `NOT VALID` then `VALIDATE` to avoid a long lock).
- **Splitting a table** or moving data to a new store: dual-write (or use CDC, topic 39, to avoid dual-write
  inconsistency), backfill, verify, switch reads, stop writes to old, delete old.
- **Changing an API:** add the new endpoint/field → migrate clients → monitor usage of the old one → remove
  it.
- **Changing a message schema:** consumers learn to accept the new format first, producers switch after.

The key ordering rule: **readers before writers** when adding (make everyone able to read the new thing
before anyone writes it), and **writers before readers** when removing (stop writing before you stop being
able to read). With a schema registry (topic 16), compatibility checks can enforce this automatically.

### 5.5 Data migrations at scale

Backfills over large tables need care: run in batches with throttling (so replication lag and lock
contention stay bounded), make them idempotent and resumable (topic 38), verify counts/checksums before
switching reads, and run them as tracked jobs rather than one-off scripts.

---

## 6. Configuration changes: a leading cause of outages

### 6.1 Why config is so dangerous

Configuration here means anything that changes behavior without a code deploy: feature flags, timeouts,
rate limits, routing rules, firewall/ACL rules, DNS records, certificates, load balancer settings,
autoscaling settings, BGP announcements.

Many of the most visible outages of the past decade have been configuration changes: bad BGP or DNS
configuration making whole companies unreachable, a regular expression in a pushed WAF rule consuming all
CPU on edge servers worldwide, an invalid config file pushed to every node of a global service at once.
Postmortem studies consistently rank configuration changes alongside code deploys as the top triggers.

Why config is more dangerous than code, despite seeming smaller:

- **Global and instant:** config systems are designed to push changes everywhere within seconds, which is
  great for fixing things and terrible for breaking things. A code deploy rolls gradually; a config push often
  doesn't.
- **Less testing:** code goes through review, CI, staging. Config is often "just a value" edited in a UI or a
  YAML file without tests.
- **Less validation:** a syntactically valid value can still be semantically disastrous (timeout `0`, rate
  limit `10` instead of `10000`, a regex with catastrophic backtracking, a route sending all traffic to one
  region).
- **Hidden coupling:** a value consumed by many services at once; changing it affects all of them.
- **Hard to attribute:** if dashboards show code deploys but not config changes, the cause of an incident
  is non-obvious.

### 6.2 Treating config like code

- **Version control and review:** config in a repository, changed via pull requests with review, diffs and
  history ("config as code").
- **Validation:** schema validation, range checks, semantic checks (does this route point at a live
  target? does this regex compile within a complexity bound?), and tests that load the config and run
  representative requests.
- **Staged rollout:** push config with the same canary/wave process as code: one cell, one region, then
  everywhere, with health checks between stages. A global instant push should be reserved for emergencies.
- **Fast rollback:** keep previous versions and make reverting one action.
- **Observability:** record config changes as events on dashboards, so an incident can be correlated with
  "what changed at 14:05?"
- **Safe consumption:** services should validate config on load and **keep running with the last known-good
  config** if the new one is invalid, rather than crashing. A component that crashes on bad config, when
  config is pushed globally, crashes globally.
- **Static stability:** systems should keep working with their current config if the config service is
  unavailable (topic 48).

### 6.3 Change management as a practice

Pulling it together, organizations that change safely tend to have:

- Every change (code, config, schema, flag, infrastructure) going through the same pipeline: reviewed,
  validated, staged, observable, reversible.
- Automatic rollback when canary metrics degrade.
- Deployment freezes when error budgets are exhausted (topic 46), and caution around peak periods.
- A change log that incident responders check first: "what changed?" is the first question in almost
  every incident.

---

## 7. How it connects

- **Topic 04:** DNS-based cutovers are slowed by TTLs; DNS changes are config changes with global blast radius.
- **Topic 07, 16:** backward/forward compatibility of APIs and encodings is what makes mixed versions safe.
- **Topic 09, 10:** load balancers drain connections and split traffic; meshes and gateways do weighted
  routing for canaries.
- **Topic 11, 12:** cache formats must be compatible across versions; blue-green cutover causes cold caches.
- **Topic 21, 22:** DDL can take locks; long backfill transactions cause contention.
- **Topic 23:** large backfills create replication lag.
- **Topic 38, 39:** backfills must be idempotent; CDC avoids dual-write inconsistencies when moving data.
- **Topic 44:** feature flags as kill switches implement graceful degradation.
- **Topic 45:** new rate limits rolled out in shadow mode.
- **Topic 46:** SLIs drive canary analysis; error budgets gate releases; change events on dashboards.
- **Topic 48:** cells and regions are the units of staged rollouts; static stability under config-service
  failure.

---

## Common misconceptions

- "Most outages are caused by hardware failure or traffic." → Most follow a change the operators made:
  code, config, schema, flags.
- "Blue-green means no mixed versions and trivial rollback." → The database and caches are shared; schema
  and data written by the new version can make rollback unsafe.
- "Rolling deploys are safe because health checks pass." → Health checks catch crashes, not wrong results
  or slow degradation. Gate on SLI comparisons.
- "A canary is fine if its error rate looks OK compared with last week." → Compare against a concurrent
  baseline under the same traffic.
- "Renaming a column is a one-line migration." → With mixed versions running, it's a multi-step
  expand/contract over several deploys.
- "Feature flags make releases risk-free." → Flag flips are global config changes; they need gradual
  rollout too, and stale flags are a hazard.
- "Config changes are small and low-risk." → They are often global, instant and less tested than code,
  which makes them one of the top outage causes.
- "We'll roll forward with a quick fix." → Under pressure, rolling back to known-good is usually faster and
  safer; mitigate first, debug later.

## Further reading (supplements)

- *Site Reliability Engineering* (Google), chapter 8 "Release Engineering"; *The Site Reliability
  Workbook*, chapter 16 "Canarying Releases".
- Amazon Builders' Library: "Automating safe, hands-off deployments" and "Ensuring rollback safety during
  deployments".
- Martin Fowler, "BlueGreenDeployment", "ParallelChange" (expand/contract) and Pete Hodgson, "Feature
  Toggles (aka Feature Flags)" on martinfowler.com.
- Jez Humble & David Farley, *Continuous Delivery* (2010).
- GitHub engineering blog on `gh-ost` (online schema migrations for MySQL).
- Netflix/Google, "Automated Canary Analysis with Kayenta" (Netflix Tech Blog / Google Cloud blog).
- Public postmortems, e.g., Cloudflare's July 2019 WAF regex outage write-up, and the SEC report on Knight
  Capital (2012).

---

## Questions

1. Why is "small, gradual, observable, reversible" a better goal than "no bugs reach production"?
2. In a rolling deployment of 20 instances, a new version passes health checks but returns wrong prices
   0.5% of the time. What happens, and what would have caught it?
3. Blue-green promises instant rollback. Describe a scenario where flipping back to blue doesn't restore
   correct behavior.
4. Why should a canary be compared with a concurrent baseline rather than yesterday's metrics or the whole
   old fleet?
5. What's the difference between deploying and releasing, and how do feature flags separate them?
6. Give two risks of feature flags and how to mitigate each.
7. Why does every non-recreate deployment require backward and forward compatibility? Give three places
   where compatibility matters besides the database.
8. A developer wants to rename `orders.amount` to `orders.total_cents` in a single migration deployed with
   the new code. What breaks? Write out the expand/contract steps.
9. In the expand/contract sequence, after you stop writing the old column, which rollbacks are still safe?
10. Why does the order "readers before writers" matter when introducing a new message format on a queue?
11. Why are configuration changes such a common outage cause, even though they're "just values"?
12. A service crashes on startup if its config file fails to parse. The config system pushes changes to all
    hosts within 30 seconds. Describe the failure and two fixes.
13. An incident starts at 14:05. What's the first question to ask, and what tooling makes it fast to answer?
14. When would you choose roll-forward over rollback?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Bugs that only appear under real production data, traffic and timing will always slip through
testing. You can't drive the probability to zero, but you can control the impact: small changes are easier
to attribute and revert, gradual exposure limits how many users are affected, observability shortens time
to detect, and reversibility shortens time to mitigate. Impact ≈ exposure × duration, and those you control.

**2.** Health checks only verify the process is up and responding, so the rollout proceeds batch by batch,
possibly reaching all 20 instances before anyone notices wrong prices; then rollback is another slow
rolling deploy. Gating each step on SLIs and business metrics (price-mismatch errors, order failures,
conversion), compared against old instances, or a canary stage with automated analysis, would have caught
it at a small fraction of traffic.

**3.** Green ran a migration that changed the schema or wrote data in a new format (e.g., stored amounts in
cents instead of dollars, or a new enum value). Blue's code can't read or misinterprets that data, so after
the flip blue fails or computes wrong results. Similarly, green may have written cache entries or queue
messages in a format blue can't parse. Shared state breaks the isolation of the two environments.

**4.** Yesterday differs by time-of-day, day-of-week, traffic mix and other deploys; the whole old fleet
differs in uptime (warm caches, JIT, long-running memory growth). A baseline of old-version instances
started at the same time and receiving the same traffic slice isolates the effect of the code change itself.

**5.** Deploying puts new code on servers; releasing exposes new behavior to users. With a flag, code is
deployed in an "off" state, and release happens later by flipping the flag, gradually, to targeted users,
and reversibly in seconds without a redeploy.

**6.** (a) A flag flip is a global instant change: roll flags out gradually (percentages, cells) with
monitoring, like code. (b) Flag debt and combinatorial paths: remove flags once features are fully launched,
track flag ownership and age, limit long-lived flags. (Also: flag service unavailability → cache values
locally with safe defaults.)

**7.** During a rollout or canary, old and new versions run simultaneously and share data and traffic; after
a rollback, old code reads what new code wrote. So new code must read old outputs (backward) and old code must
tolerate new outputs (forward). Beyond the database: API requests/responses between services and with
old mobile clients, messages on queues, cache entries, serialized files/events, and RPC schemas.

**8.** Old instances still running during the rollout query `amount` and fail; rollback is impossible
without renaming back. Steps: (1) add nullable `total_cents`; (2) deploy code writing both, reading
`amount`; (3) backfill `total_cents = amount * 100` (or whatever conversion) in batches; (4) deploy code
reading `total_cents`, still writing both; (5) deploy code writing only `total_cents`; (6) once nothing reads
`amount` (check other services, reports, jobs), drop it.

**9.** Rolling back to the version that reads the new column and writes both (step 4's code) is safe, since
it reads the column that is still maintained. Rolling back further, to a version that reads the old column,
is not safe: the old column is no longer updated, so it would read stale data.

**10.** During the transition, producers and consumers are at different versions. If a producer writes the
new format before all consumers can read it, old consumers fail or mis-process messages (possibly poison
messages that block a partition). Updating readers first ensures any message in the new format can be read
by every consumer that might receive it.

**11.** Config systems are built to propagate changes globally within seconds (no gradual rollout), config
gets less review and testing than code, values can be syntactically valid but semantically disastrous, one
value may be consumed by many services, and config changes are often invisible on dashboards so incidents
are slow to attribute.

**12.** A malformed file is pushed everywhere within 30 seconds; every host that restarts (or reloads)
crashes, so the whole fleet can go down at once: a global outage from one edit. Fixes: validate config
before pushing (schema and semantic checks in CI); stage config rollouts by cell/region with health
checks; make services keep running on the last known-good config if a new one fails to parse.

**13.** "What changed?" Code deploys, config pushes, flag flips, schema migrations, infrastructure changes,
and dependency changes around 14:05. Tooling: a central change log/event stream annotated on dashboards,
deployment and flag audit trails, and config in version control.

**14.** When rollback isn't safe or possible: the new version has already made irreversible changes (data
migrated, new-format data written that the old version can't read, messages sent), or the old version has
a known severe problem (e.g., a security fix). Also when the fix is trivial, well understood and the
deployment pipeline is fast. Otherwise, roll back first.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Leading trigger of outages?
**A:** Changes made by operators: code deploys, config pushes, flag flips, schema changes.

**Q:** Four principles of safe change?
**A:** Small, gradual, observable, reversible.

**Q:** Rolling deployment?
**A:** Replace instances in batches (drain, replace, health-check), with no extra capacity but mixed versions.

**Q:** Rolling deployment weakness?
**A:** Health checks miss subtle bugs, so it can reach 100% before detection; rollback is slow.

**Q:** Blue-green deployment?
**A:** Two full environments; switch the router from blue (old) to green (new); flip back to roll back.

**Q:** Blue-green costs?
**A:** 2× capacity, all-at-once exposure, cold caches, and shared DB state that can make rollback unsafe.

**Q:** Canary deployment?
**A:** Send a small fraction of traffic to the new version, compare against a baseline, widen gradually.

**Q:** What should a canary be compared with?
**A:** A concurrent baseline running the old version under the same traffic.

**Q:** Staged rollout waves?
**A:** One cell → one zone → one region → everywhere, with bake time between.

**Q:** Deploy vs release?
**A:** Deploy puts code on servers; release turns behavior on for users. Flags separate them.

**Q:** Kill switch?
**A:** An ops flag that disables a feature quickly, e.g., under load.

**Q:** Flag risks?
**A:** Global instant flips, combinatorial paths, stale flag debt, flag-service dependency.

**Q:** Why must versions be compatible both ways?
**A:** Old and new run at the same time and read each other's data, messages and caches; rollback means old reads new.

**Q:** Expand/contract steps?
**A:** Add new structure → dual-write → backfill → read new → stop writing old → drop old.

**Q:** Readers or writers first when adding a format?
**A:** Readers first; when removing, writers stop first.

**Q:** Why backfill in batches?
**A:** Avoid long locks, replication lag and load spikes; make it resumable and idempotent.

**Q:** Adding a NOT NULL column safely?
**A:** Add nullable, write it always, backfill, then add the constraint (validated without a long lock).

**Q:** Why are config changes dangerous?
**A:** Global, instant, less tested, semantically unvalidated, hard to attribute.

**Q:** Treating config like code means?
**A:** Version control, review, validation, staged rollout, fast rollback, change events on dashboards.

**Q:** Last known-good config?
**A:** Services keep running on the previous valid config if a new one fails validation instead of crashing.

**Q:** Mitigate or debug first?
**A:** Mitigate first (usually roll back), then debug.

**Q:** When is rollback unsafe?
**A:** When the new version wrote data or formats the old version can't handle.

**Q:** First question in an incident?
**A:** What changed?
