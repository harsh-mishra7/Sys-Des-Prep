# Day 1: Flashcards

Cover the answer, say it out loud, then check.

---

**Q:** Latency vs throughput, in one line each?
**A:** Latency = time for one unit of work. Throughput = units of work completed per unit time.

**Q:** Bandwidth vs throughput?
**A:** Bandwidth is the channel's maximum capacity; throughput is what you actually achieve.

**Q:** State Little's Law.
**A:** `L = λ × W`: requests in flight = arrival rate × time each spends in the system.

**Q:** Why does batching raise throughput but hurt latency?
**A:** It amortizes fixed costs over many items (more work/sec), but early items wait for the batch to fill.

**Q:** What happens to latency as utilization approaches 100%?
**A:** It grows non-linearly (≈ service time / (1 − utilization)): 2× at 50%, 10× at 90%, 100× at 99%.

**Q:** Why do averages mislead for latency?
**A:** Latency distributions are right-skewed; the mean describes no real request and hides the tail.

**Q:** What is p99?
**A:** The value below which 99% of requests fall; 1 in 100 requests is slower.

**Q:** Tail latency amplification formula for fan-out to N calls?
**A:** P(at least one slow) = 1 − (1 − p)^N. With p = 1%, N = 100 → ~63%.

**Q:** Can you average p99s across servers?
**A:** No. Merge the histograms/sketches, then compute the percentile.

**Q:** What is a hedged request?
**A:** After a short delay, send a duplicate request to another replica and use whichever answers first; cuts tail latency.

**Q:** Coordinated omission?
**A:** A measurement bias where the load generator waits on slow responses, so it under-samples slow periods.

**Q:** Availability vs reliability?
**A:** Availability: is it answering now? Reliability: does it keep answering *correctly* over time?

**Q:** Availability vs durability?
**A:** Availability is about the service responding; durability is about acknowledged data not being lost.

**Q:** Availability in terms of MTBF/MTTR?
**A:** MTBF / (MTBF + MTTR). Improve it by failing less often or recovering faster.

**Q:** Downtime per year at 99.9% / 99.99%?
**A:** ~8.8 hours / ~53 minutes.

**Q:** Availability of components in series vs parallel?
**A:** Series: multiply availabilities (gets worse). Parallel: multiply unavailabilities (gets better, if independent).

**Q:** Fault vs failure?
**A:** Fault: one component deviates from spec. Failure: the whole system stops serving users.

**Q:** Why are software faults more dangerous than hardware faults?
**A:** They're correlated: every node runs the same code, so one bug can hit all replicas at once.

**Q:** What are load parameters?
**A:** Numbers that describe load for your architecture: req/s, read/write ratio, data size, concurrency, fan-out.

**Q:** Performance problem vs scalability problem?
**A:** Performance: slow for one user. Scalability: fast for one user, slow under heavy load.

**Q:** Two things that stop linear scaling?
**A:** Contention (serialized work, Amdahl's law) and coordination cost between nodes.

**Q:** Vertical vs horizontal scaling?
**A:** Vertical: bigger machine, simple, hard ceiling, SPOF. Horizontal: more machines, high ceiling, distributed complexity.

**Q:** Stateless component?
**A:** Holds nothing between requests that another instance would need; any instance can serve any request.

**Q:** Why are stateless tiers easy to scale?
**A:** Add instances behind a load balancer; no data to move, copy, or keep consistent.

**Q:** Standard pattern for state?
**A:** Push state down into a few specialized stateful systems (DB, cache, queue, object store); keep everything above them stateless.

**Q:** Job of a load balancer?
**A:** Single entry point that spreads requests across healthy servers and hides individual server failures.

**Q:** Job of a CDN?
**A:** Cache content at edge locations near users to cut latency and origin load.

**Q:** Job of a queue?
**A:** Decouple producers from consumers: accept work now, process later, absorb spikes, enable retries.

**Q:** Job of object storage?
**A:** Cheap, durable storage for large immutable blobs addressed by key.
