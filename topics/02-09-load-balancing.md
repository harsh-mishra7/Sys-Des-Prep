# 09 — Load Balancing

> Part 2 — Networking & Communication · Prerequisites: topics 01, 02, 03, 04, 05, 06, 07

Horizontal scaling (topic 01) means running many copies of a service instead of one big machine. That
immediately raises a question: when a request arrives, **which copy handles it?** Clients shouldn't
have to know how many servers exist, which ones are healthy, or which are busy. A **load balancer** sits
in front of the pool, gives clients one stable address, and spreads requests across the servers behind
it, steering around the ones that are dead or overloaded.

That sounds simple, but most of the interesting questions are in the details. At what layer does the
balancer look at traffic (TCP connections or HTTP requests)? How does it pick a server, and why does
"pick the least busy" turn out to be subtle? How does it know a server is unhealthy, and how does it
remove one without cutting off users mid-request? What happens to sessions that live in one server's
memory? And since every request now goes through the load balancer, how do you keep *it* from being the
single point of failure?

By the end of this topic you should be able to:

1. Explain why load balancers exist: distribution, health, a stable address, and decoupling clients
   from the server pool.
2. Explain the difference between L4 (transport) and L7 (application) load balancing, including what
   each can see and what it costs.
3. Describe round robin, weighted, least connections, least response time, hashing and power of two
   choices, and when each works or fails.
4. Explain active and passive health checks, connection draining and slow start.
5. Explain sticky sessions and why they fight statelessness.
6. Explain how the load balancer itself is made highly available: active-passive pairs with floating
   IPs, active-active, DNS-level balancing and anycast.

---

## 1. Why load balancers exist

Without a load balancer, clients would need a list of servers and logic to choose between them, and
every client would need updating whenever a server was added, removed, or died. A load balancer takes
that job:

```
                    ┌─────────────────────┐
                    │                     │──▶ app server 1
 clients ──────────▶│    load balancer    │──▶ app server 2   (healthy)
 (one address:      │ (VIP 203.0.113.10)  │──▶ app server 3
  api.example.com)  │                     │ ✗  app server 4   (failing health checks: skipped)
                    └─────────────────────┘
```

Its jobs:

- **Distribution**: spread work so no server is overloaded while others idle. Since latency rises
  sharply as utilization approaches 100% (topic 02), even a modest imbalance can make one server's p99
  much worse than the rest.
- **Health**: detect failed or failing servers and stop sending them traffic. This turns individual
  server failures into reduced capacity rather than user-visible errors (availability, topic 01).
- **A stable address**: clients talk to one name or IP (often a **virtual IP**, VIP). Servers can be
  added, removed, replaced and deployed behind it without clients noticing.
- **Operations**: rolling deploys, draining, and gradual traffic shifts all happen at the balancer.
- **Often more**: TLS termination (topic 05), HTTP/2 to HTTP/1.1 translation, compression, request
  routing, rate limiting. These overlap with reverse proxies and gateways (topic 10).

The precondition for all of this is that **any server can handle any request**. That's why stateless
application servers (topics 01 and 07) matter so much, and why section 6 (sticky sessions) is a
compromise.

---

## 2. L4 vs L7 load balancing

The "layer" refers to how deep into the traffic the balancer looks (topic 03's layers).

### Layer 4: transport-level

An **L4** load balancer works with TCP (or UDP) connections. It sees IP addresses and ports, and nothing
of the application data. It picks a backend **when a connection is opened**, and from then on every
byte of that connection goes to that backend.

```
client ── TCP SYN to VIP:443 ──▶ L4 LB: choose backend 2 for this 4-tuple
                                 ──▶ all packets of this connection go to backend 2
```

There are two styles:

- **Pass-through / NAT**: the balancer rewrites destination addresses on packets and forwards them;
  the TCP connection is really between the client and the backend. Some designs (**direct server
  return**) let the backend reply straight to the client, bypassing the balancer on the way back, which
  helps because responses are usually much larger than requests.
- **Proxying (terminating)**: the balancer accepts the client's TCP connection and opens a separate one
  to the backend, copying bytes between them.

Strengths: very fast (it can be done in kernels, with eBPF, or in hardware), protocol-agnostic (works
for databases, MQTT, anything over TCP/UDP), can handle millions of connections, and with pass-through
it doesn't need TLS keys because it never decrypts anything.

Weaknesses: it can only balance **connections**, not requests. With long-lived connections carrying
many requests (HTTP keep-alive, HTTP/2, gRPC; topics 06–07), one connection can carry a huge share of
traffic, and it's pinned to one backend for its lifetime. It can't route by URL, header or cookie, and it
can't retry a failed HTTP request because it doesn't know where requests begin and end.

### Layer 7: application-level

An **L7** load balancer terminates the client's connection (including TLS), **parses** the
application protocol (usually HTTP), and makes a decision **per request**:

```
client ══ TLS + HTTP/2 ══▶ L7 LB ── parses each request ──┬─▶ GET /api/users/42 → users pool, backend 5
                                                           ├─▶ GET /static/app.js → static pool
                                                           └─▶ POST /checkout     → checkout pool, backend 2
```

What it can do because it understands requests:

- **Content-based routing**: by path, host, header, cookie, method.
- **Per-request balancing** over long-lived connections, including splitting one client's HTTP/2
  streams across many backends. This is what gRPC needs (topic 07).
- **Retries** of failed idempotent requests on another backend; per-request timeouts.
- **Observe** status codes and latencies per backend, feeding passive health checks (section 4).
- **Modify** traffic: add headers (`X-Forwarded-For` so the backend knows the real client IP), rewrite
  paths, compress, cache.

Costs: more CPU per request (TLS termination and parsing), more latency (small, but present), it must
hold TLS certificates and keys, and it only works for protocols it understands.

| | L4 | L7 |
|---|---|---|
| Sees | IPs, ports, TCP/UDP | Full requests: paths, headers, cookies, bodies |
| Balances | Connections | Individual requests |
| TLS | Usually passes through | Usually terminates |
| Routing by content | No | Yes |
| Retries failed requests | No | Yes (idempotent ones) |
| Cost per byte | Very low | Higher (parsing, crypto) |
| Typical use | Front door for huge traffic, non-HTTP protocols, in front of L7 tier | HTTP/gRPC APIs, microservice routing |

Large systems often **layer** them: an L4 tier (fast, simple, scales out) spreads connections across an
L7 tier (smart, per-request), which spreads requests across application servers.

```
Internet ──▶ L4 balancers (anycast / ECMP) ──▶ L7 proxies (TLS, routing) ──▶ service pools
```

---

## 3. Balancing algorithms

The algorithm decides which healthy backend gets the next connection (L4) or request (L7). There are
two broad families: **static** ones that ignore current load, and **dynamic** ones that react to it.

### Round robin

Hand requests to backends in turn: 1, 2, 3, 1, 2, 3, ...

- Simple, stateless, perfectly even **in count**.
- Assumes every request costs the same and every server is equally fast. If one request in a hundred
  is a 5-second report and the rest take 5 ms, round robin happily sends the next request to the server
  stuck on the report.

### Weighted round robin

Give each backend a weight proportional to its capacity: a server with weight 2 gets twice as many
requests as one with weight 1. Useful for mixed hardware, or for canary deploys (send 1% of traffic to a
new version by weighting it 1:99; topic 47). Good implementations interleave rather than sending runs
(`A A B A A B` rather than `A A A A B B`).

### Least connections (and least outstanding requests)

Send the new request to the backend with the **fewest active connections** (L4) or **fewest in-flight
requests** (L7). This reacts to reality: a server stuck on slow requests accumulates in-flight work and
stops receiving more. By Little's Law (topic 02), in-flight count = arrival rate × latency, so a server
that has become slow automatically shows a higher count. Weighted variants divide by capacity.

Watch-outs: a server that **fails fast** (returning errors in 1 ms) has very few in-flight requests,
so least-connections sends it *more* traffic. This "black hole" effect is why passive health checks
(section 4) must accompany it.

### Least response time

Prefer the backend with the lowest recent latency (often combined with fewest in-flight requests),
measured with a moving average. It favours fast servers, but it's sensitive to noise, and it shares the
fail-fast problem: an instance returning errors quickly looks very fast.

### Hashing

Pick the backend from a hash of something in the request: client IP, a user ID, a session cookie, the
URL. The same key always goes to the same backend (while the pool is unchanged).

- Useful when backends hold **per-key state or caches**: routing all requests for user 42 to the same
  server makes its local cache effective. Also the basis of IP-based stickiness.
- Load is only as even as the keys: one huge customer or a hot URL overloads its server.
- With naive `hash(key) % N`, adding or removing one backend remaps almost every key, wiping out every
  cache at once. **Consistent hashing** (topic 13) fixes this, moving only about 1/N of keys.

### Power of two choices

Pick **two backends at random**, then send the request to the **less loaded** of the two (by in-flight
requests, say).

Why not just pick the globally least-loaded backend? Because in real systems, especially with **many
load balancers** each making decisions independently, load information is **stale**. If 50 balancers all
see that server 7 has the fewest connections, they all send their next requests to server 7 at the same
moment, and it gets flooded (**herd behaviour**). Then they all avoid it. Load oscillates.

```
"Pick the least loaded" with stale info, many balancers:
   all balancers → server 7 (looked emptiest) → server 7 overloaded → all avoid it → ...

Power of two choices:
   balancer A compares 3 vs 11 → 3     balancer B compares 7 vs 19 → 7
   balancer C compares 2 vs 7  → 2     (choices spread out; no herd)
```

The randomness spreads decisions out, while comparing two still avoids the worst choices. There's a
striking mathematical result behind it: when placing n items into n bins, picking one random bin gives a
maximum load around log n / log log n, while picking the better of **two** random bins drops it to
around log log n. Going from one choice to two is a huge improvement; going from two to three adds
little. It's cheap (no global state), robust to staleness, and widely used in modern proxies.

### Summary

| Algorithm | Uses current load? | Strength | Weakness |
|---|---|---|---|
| Round robin | No | Simple, even counts | Ignores request cost and server speed |
| Weighted RR | No | Handles uneven capacity, canaries | Weights are static guesses |
| Least connections / requests | Yes | Adapts to slow requests and slow servers | Fail-fast servers attract traffic; herding with many LBs |
| Least response time | Yes | Favours fast servers | Noisy; fast errors look good |
| Hashing | No | Affinity for caches/state | Uneven with hot keys; remaps on change unless consistent |
| Power of two choices | Yes (sampled) | Near-optimal, robust to stale info, no herding | Slightly random; needs load signal |

---

## 4. Health checks, draining and slow start

### Active health checks

The balancer **probes** each backend on a schedule: open a TCP connection, or send `GET /healthz` every
few seconds, and expect a `200` within a timeout. A backend is marked **unhealthy** after N consecutive
failures (say 3) and **healthy** again after M consecutive successes (say 2). The thresholds stop a
single blip from flapping a server in and out.

```
probe: ok ok ✗ ok ✗ ✗ ✗ → unhealthy (3 in a row) → stop routing ... ok ok → healthy → route again
```

What should the health endpoint check? There's a real tension:

- **Too shallow** ("process is up, return 200"): a server whose database connection pool is broken
  still passes, and keeps receiving traffic it can only fail.
- **Too deep** ("check the database, the cache and three downstream services"): when the shared
  database has a brief hiccup, **every** server fails its health check at once, the balancer removes the
  whole pool, and a partial degradation becomes a total outage.

A common compromise: the health check verifies the server itself can work (process alive, not
deadlocked, local resources OK), and does **not** fail because of shared dependencies that all servers
depend on equally. Many balancers also **fail open**: if every backend looks unhealthy, send traffic to
all of them anyway, since that's more likely a health-check problem than a total failure.

Kubernetes splits this into **liveness** (restart me if this fails) and **readiness** (don't send me
traffic right now) checks, and the distinction is the same idea.

### Passive health checks (outlier detection)

Instead of (or as well as) probing, the balancer **watches real traffic**: if a backend returns five
`5xx` errors in a row, or its connection attempts time out, or its error rate is far above its peers',
**eject** it for a while (say 30 s, growing on repeat offences), then let it back in.

- Catches problems probes miss: a server that answers `/healthz` fine but fails real requests.
- Reacts within a few requests rather than a few probe intervals.
- Needs real traffic to fail first, so some requests are lost; and it relies on servers returning
  honest status codes (topic 07's "200 with error in the body" defeats it).
- Usually capped: never eject more than, say, 50% of the pool, so that a widespread problem doesn't
  empty it.

Most production setups use both.

### Connection draining (deregistration delay)

When you remove a server (deploy, scale-in), cutting it off instantly would kill every request it's
handling. **Draining** removes it gracefully:

```
1. Mark server 3 as "draining": the LB sends it no NEW requests/connections
2. In-flight requests finish normally (up to a drain timeout, e.g. 30–300 s)
3. Long-lived connections are closed politely: HTTP/1.1 "Connection: close" on the next response,
   HTTP/2 GOAWAY frame, WebSocket close / reconnect hint (topic 08)
4. After the timeout, anything still open is cut; the server shuts down
```

The application must cooperate: on a shutdown signal it should stop accepting work, fail its readiness
check, finish what's in flight, then exit. A common bug is a server that exits immediately on the
signal, before the balancer has even noticed it's draining.

### Slow start (warm-up)

A newly added server is often **cold**: empty local caches, JIT-compiled code not yet optimized,
connection pools not yet filled. If least-connections sees it has zero connections and sends it a
full share (or more) of traffic instantly, its latency spikes and it may fail health checks.

**Slow start** ramps a new backend's weight from a small fraction to full over a configured window
(say 60 s), so it warms up under gradually increasing load.

---

## 5. Where state lives, and why it matters

Balancing works best when servers are **interchangeable**. The more a server holds per-client state in
its own memory (a session, a shopping cart, an in-progress upload, an open WebSocket), the more the
balancer is constrained: requests for that client must go back to *that* server.

The standard design is to **move state out** of the application servers into a shared store (a
database, Redis, or the client itself via signed tokens; topic 54). Then any server can handle any
request, failures lose nothing, and the balancer has full freedom.

---

## 6. Sticky sessions

**Sticky sessions** (session affinity) make the balancer send all requests from one client to the same
backend. Mechanisms:

- **Cookie-based** (L7): the balancer sets a cookie like `lb=server3` on the first response; later
  requests carrying it go to server 3.
- **IP hash** (L4 or L7): hash the client IP. Breaks when many users share one IP (corporate NAT, mobile
  carrier NAT puts thousands behind one address, all landing on one server) and when a phone's IP
  changes.
- **Hash on an application key** (user ID, session ID), ideally with consistent hashing.

Why people use them: the app keeps sessions in local memory, or benefits from a warm per-user cache, or
holds a multi-step workflow (upload chunks) locally.

Why they fight statelessness:

- **Uneven load.** Stickiness overrides the algorithm. A server that happens to hold heavy users stays
  hot; a newly added server gets only new sessions, so scale-out helps slowly (the same problem as
  long-lived connections in topic 08).
- **Failures lose state.** If server 3 dies, every session pinned to it is gone: users are logged out,
  carts emptied. Stickiness hides the statefulness until the worst moment.
- **Draining takes longer.** A server can't be removed until its sessions expire or are moved.
- **Hotspots are sticky too.** A misbehaving client hammering one server can't be spread out.

Sticky sessions are best treated as a **performance optimization** (for cache warmth), never as a
correctness requirement: the system should still work, just a little slower, if a request lands on a
different server. If losing affinity would lose data, the state belongs in a shared store.

---

## 7. Making the load balancer itself highly available

Every request goes through the load balancer, so it must not be a single point of failure. Several
techniques, often combined:

### Active-passive pair with a floating IP

Two balancer machines; one (active) owns the **virtual IP** that clients connect to; the other
(passive) watches it with heartbeats. If the active stops responding, the passive **takes over the
IP**. On a LAN this is done with a protocol like **VRRP** (implemented by tools such as keepalived): the
new owner sends a **gratuitous ARP** announcing "this IP is now at my MAC address", so switches redirect
traffic to it. In clouds, an API call moves an elastic/floating IP to the standby.

```
             VIP 203.0.113.10
                  │
        ┌─────────┴─────────┐
   ┌────┴────┐   heartbeat  ┌─────────┐
   │ LB-A    │◀────────────▶│ LB-B    │
   │ ACTIVE  │              │ PASSIVE │
   └─────────┘              └─────────┘
   LB-A dies → LB-B claims the VIP (gratuitous ARP / cloud API) → traffic resumes
```

- Simple, failover in a second or a few seconds.
- Half the hardware sits idle; capacity is limited to one machine.
- Existing connections on the failed balancer are lost unless connection state is synchronized.
- **Split brain** risk: if the heartbeat link fails but both are alive, both may claim the VIP. This is
  the same problem that appears in database failover (topic 23); fencing and careful timeouts are the
  answer.

### Active-active

Several balancers all handle traffic at once. Ways to spread traffic across *them*:

- **ECMP** (equal-cost multi-path) routing: the upstream router sees several equal routes to the VIP
  (one per balancer) and hashes each flow's 4-tuple to pick one. All balancers advertise the same IP
  (via BGP); if one dies, it stops advertising and the router drops that path. Flows hashed to the dead
  balancer move; consistent hashing in the balancers (topic 13) helps the survivors pick the *same*
  backend for those flows.
- **DNS** returning several balancer IPs.

Active-active uses all capacity and scales out, at the cost of more complexity.

### DNS-level balancing

DNS can return **multiple A records** (round robin across them) or answer differently based on health
and the client's location (**GeoDNS**; topic 04). This balances across **balancers, data centres, or
regions**, not individual servers:

- Coarse and slow: resolvers and clients cache answers for the TTL (topic 04), and some ignore TTLs, so
  failover takes minutes for a portion of users.
- Clients choose, and many just take the first record.
- No knowledge of load or individual requests.

So DNS is the **outermost** layer (which region or site), with real load balancers inside each site.

### Anycast

The same IP address is announced via BGP from many locations; the internet's routing sends each client
to the "nearest" one (topic 04). If a site goes down, it withdraws its announcement and traffic flows to
the next nearest. Used by CDNs and DNS providers at the global edge; often combined with ECMP inside each
site.

### The full stack

```
client
  │  DNS / GeoDNS / anycast: which region or site?
  ▼
site edge: routers with ECMP across L4 balancers (active-active)
  │
  ▼
L7 proxies (TLS termination, routing, retries)     ← active-active pool behind the L4 tier
  │
  ▼
service backends (health-checked, drained, slow-started)
```

Each layer is redundant, and each balances across the redundant layer below it.

### Client-side load balancing

An alternative for internal traffic: skip the central balancer. Each client gets the list of backend
instances from **service discovery** (topic 10) and picks one itself (often with power of two choices).
No extra hop, no central bottleneck, per-request balancing for gRPC. The cost: every client needs the
logic (a library per language, or a sidecar proxy; topic 10), and health knowledge is spread across many
clients.

---

## 8. How it connects

- **Stateless services** (topics 01 and 07) are what let a balancer send any request anywhere; sticky
  sessions and persistent connections (topic 08) erode that.
- **Queueing and Little's Law** (topic 02) explain why imbalance hurts tail latency and why in-flight
  counts reflect slowness.
- **TCP and connection reuse** (topic 03) explain why L4 balancers balance connections, not requests;
  **HTTP/2 and gRPC** (topics 06–07) make that a real problem.
- **TLS termination** (topic 05) usually happens at the L7 balancer.
- **DNS, GeoDNS and anycast** (topic 04) form the outermost balancing layer.
- **Reverse proxies, API gateways and service meshes** (topic 10) are L7 balancers with more features;
  **service discovery** (topic 10) feeds the backend list.
- **Consistent hashing** (topic 13) makes hash-based balancing stable under change.
- **Failover and split brain** return for databases (topic 23) and leader election (topics 30–31).
- **Retries** at the balancer must respect idempotency (topic 07) and retry budgets (topic 43);
  **load shedding** (topic 44) complements balancing when the whole pool is overloaded.
- **Canary and blue-green deploys** (topic 47) are implemented by weighting traffic at the balancer.

## Common misconceptions

- **"Round robin spreads load evenly."** → It spreads *request counts* evenly. If requests vary in
  cost or servers vary in speed, the load is uneven.
- **"Always send to the least-loaded server."** → With many balancers and stale information, that
  causes herding. Power of two choices is usually better.
- **"An L4 load balancer balances requests."** → It balances connections. With HTTP/2 or gRPC, one
  connection can carry all of a client's requests to one backend.
- **"Health checks should verify every dependency."** → Deep checks on shared dependencies can mark the
  whole pool unhealthy at once and turn a partial failure into a total outage.
- **"Sticky sessions make stateful apps scale."** → They make stateful apps *work* behind a balancer,
  at the cost of uneven load and lost sessions on failure. Moving state out is the real fix.
- **"DNS round robin is a load balancer."** → It's coarse distribution with caching delays and no
  health or load awareness. Useful between sites, not as the only layer.
- **"Two load balancers means no single point of failure."** → Only if failover actually works:
  heartbeats, IP takeover, split-brain protection and enough capacity on the survivor.
- **"Removing a server from the pool is instant and harmless."** → Without draining, its in-flight
  requests fail. Without slow start, a new server is flooded while cold.

## Further reading (supplements)

- Michael Mitzenmacher, "The Power of Two Choices in Randomized Load Balancing" (PhD thesis, 1996) and
  the survey "The Power of Two Random Choices: A Survey of Techniques and Results"
- Daniel E. Eisenbud et al., "Maglev: A Fast and Reliable Software Network Load Balancer" (Google,
  NSDI 2016)
- Google, *Site Reliability Engineering* book (free online), chapters 19 "Load Balancing at the
  Frontend" and 20 "Load Balancing in the Datacenter"
- Envoy proxy documentation: "Load balancing" and "Outlier detection" architecture overviews
- Sam Rose, "Load Balancing" (interactive visual essay, samwho.dev)

---

## Questions

1. A team runs one powerful server and asks why they'd want three smaller ones behind a load balancer
   instead. Give three reasons, and the precondition that makes it work.

2. What can an L7 load balancer do that an L4 one can't? Why might you still put an L4 tier in front of
   an L7 tier?

3. A service has two kinds of requests: 98% take 5 ms and 2% take 3 s. It runs behind a round-robin
   balancer and has terrible p99 latency even at modest load. Explain why, and which algorithm would
   help.

4. Least connections is used, and one backend starts failing every request instantly with a `500`. What
   happens to traffic, and what stops it?

5. Why does "send each request to the least-loaded backend" behave badly when 40 independent load
   balancers share the same backend pool? How does power of two choices avoid the problem?

6. A cache tier is balanced with `hash(URL) % N`. Ops add one server to go from 10 to 11. What happens to
   the cache hit rate, and what technique avoids it?

7. A health check endpoint queries the shared database. The database has a 20-second hiccup. What
   happens, and how should the health check be designed instead?

8. Compare active and passive health checks. Why do most production systems use both?

9. During every deploy, users see a burst of `502` errors. The deploy replaces servers one at a time.
   What's probably missing, and what's the correct shutdown sequence?

10. After a server is added to the pool, its latency spikes and it fails health checks within a minute,
    then recovers once removed. What's happening and what feature addresses it?

11. An app stores sessions in local memory and uses cookie-based sticky sessions. List three problems
    this causes, and what the long-term fix is.

12. A balancer uses IP-hash stickiness. One server is badly overloaded while others are idle. What's a
    likely cause?

13. Describe how an active-passive load balancer pair with a floating IP fails over. What can go wrong?

14. Why is DNS-based balancing unsuitable as the only layer for fast failover? Where does it fit?

15. Internal services talk over gRPC. Compare putting a central L7 load balancer between them with
    client-side load balancing.

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** (a) **Availability**: one server failing (or being deployed) leaves the others serving, instead
of a full outage; (b) **scalability**: add servers as load grows instead of hitting the ceiling of the
biggest machine; (c) **operations**: rolling deploys, draining and canaries without downtime; also
running servers at moderate utilization keeps queueing delay low. The precondition is that **servers are
interchangeable**: stateless, with sessions and data in shared stores, so any server can handle any
request.

**2.** L7 parses requests, so it can route by path/host/header/cookie, balance **individual requests**
(including HTTP/2 streams on one connection), retry failed idempotent requests on another backend,
apply per-request timeouts, observe status codes for outlier detection, and modify headers. An L4 tier
in front is still useful because it's very cheap per byte and scales to huge connection counts; it
spreads connections across many L7 proxies, which do the expensive TLS and parsing work, and it lets the
L7 tier itself be scaled out and replaced without changing the public address.

**3.** Round robin ignores request cost. A server that gets one of the 3-second requests keeps receiving
its share of new requests while busy, so they queue behind the slow one (head-of-line delay), and the
servers' load is uneven at any moment. Those queued requests make p99 terrible. **Least outstanding
requests** (or power of two choices on in-flight count) helps: a server busy with slow requests has more
in flight, so new requests go elsewhere. (Separating slow requests into their own pool also helps.)

**4.** A server failing instantly has almost **no in-flight requests**, so least connections sees it as
the least loaded and sends it **more** traffic: a black hole that eats a disproportionate share of
requests. **Passive health checks / outlier detection** stop it: after several consecutive `5xx`s, the
balancer ejects it. Active checks might catch it too, but only if the health endpoint fails as well, and
only after a few probe intervals.

**5.** Each balancer's view of load is **stale** and they decide independently. When one backend looks
least loaded, all 40 balancers send their next requests to it at the same moment, flooding it; then they
all see it as busy and avoid it, so load oscillates (herding). With power of two choices, each balancer
picks two random backends and takes the less loaded of the pair. Different balancers sample different
pairs, so decisions spread out, while still avoiding badly overloaded backends. Theory shows two random
choices already cut the maximum load dramatically compared with one.

**6.** With `% N`, changing N from 10 to 11 changes the target for about 10/11 of all URLs (a key stays
put only if `hash % 10 == hash % 11`). Nearly every request now goes to a server that doesn't have it
cached; the hit rate collapses and the origin gets a flood of misses. **Consistent hashing** (topic 13)
moves only about 1/11 of keys (those the new server takes over).

**7.** Every server's health check fails at the same time, because they all depend on the same
database. The balancer marks the **entire pool** unhealthy and stops routing, so the site goes fully
down, even for requests that didn't need the database (cached pages, static content), and recovery takes
extra time as checks pass again. Better: the health check verifies only the server's own ability to work
(process alive, event loop responsive, local resources), not shared dependencies; the app degrades
gracefully when the database is down; and the balancer fails open if all backends look unhealthy.

**8.** Active checks probe on a schedule: they detect dead servers even with no traffic and can take a
server out before it receives requests, but they test only the health endpoint, and react only after
several intervals. Passive checks watch real traffic: they catch failures the health endpoint doesn't
show and react within a few requests, but need real requests to fail first and need honest status codes.
Together they cover each other's blind spots.

**9.** **Connection draining** (and a graceful shutdown in the app). Servers are being killed while the
balancer still routes to them or while they have requests in flight, so the balancer sees resets and
returns `502`. Correct sequence: (1) mark the server as draining/deregister it so it gets no new
requests (the app can also fail its readiness check); (2) wait for in-flight requests to finish, closing
keep-alive connections politely (`Connection: close`, HTTP/2 `GOAWAY`); (3) after the drain timeout,
stop the process. The app must not exit immediately on the shutdown signal.

**10.** The new server is **cold**: empty caches, unwarmed JIT, empty connection pools. With least
connections it looks idle and gets a full (or larger) share of traffic at once; requests are slow, which
piles up in-flight work and fails health checks. **Slow start** ramps its weight up gradually over a
warm-up window so it warms under increasing load. (Pre-warming caches before registering also helps.)

**11.** (a) **Lost sessions on failure**: when a server dies or is replaced, all its users are logged
out or lose carts. (b) **Uneven load**: heavy users stay pinned; new servers only get new sessions, so
scale-out helps slowly. (c) **Slow drains/deploys**: a server can't be removed until its sessions end.
Also, hotspots can't be spread. Long-term fix: move session state to a **shared store** (Redis, a
database) or into a signed client-side token, making servers stateless; keep stickiness, if at all, only
as a cache-warmth optimization.

**12.** Many users share **one public IP** (a corporate NAT or a mobile carrier's carrier-grade NAT), so
they all hash to the same server. Or one client (a partner integration, a bot) is very heavy. IP hashing
can't split traffic from one IP across servers. Hash on something finer-grained (a session or user ID),
or avoid stickiness.

**13.** The active balancer owns the virtual IP; the passive one exchanges heartbeats with it (e.g.
VRRP). When heartbeats stop, the passive claims the VIP: on a LAN by sending gratuitous ARP so switches
send the IP's traffic to its MAC, in a cloud by calling an API to reassign the floating IP. Traffic
resumes within seconds. What can go wrong: **split brain** if the heartbeat link fails while both are
alive (both claim the VIP); connections on the failed balancer are lost unless state was synchronized;
the passive may not actually have enough capacity or correct config (it's rarely exercised); flapping if
timeouts are too tight.

**14.** Resolvers and clients cache DNS answers for the TTL, and some ignore low TTLs, so changes take
minutes or longer to reach everyone; clients often just use the first record and don't retry others
quickly; DNS knows nothing about load or individual requests. It fits as the **outermost** layer:
choosing a region or site (GeoDNS, latency-based answers), and spreading across several balancer IPs,
with real health-checked load balancers inside each site.

**15.** **Central L7 balancer**: clients stay simple (one address), policy is configured in one place,
and it balances per request over HTTP/2. But it's an extra network hop, a shared capacity and failure
domain to scale and keep highly available. **Client-side balancing**: clients get the instance list from
service discovery and pick per request (e.g. power of two choices), so no extra hop and no central
bottleneck; but every client needs the logic and discovery integration (a library per language, or a
sidecar proxy as in a service mesh, topic 10), and health information is spread across clients.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What are a load balancer's core jobs?
**A:** Distribute load, route around unhealthy servers, give clients one stable address, enable deploys/draining.

**Q:** What precondition makes load balancing work best?
**A:** Interchangeable (stateless) servers: any server can handle any request.

**Q:** What does an L4 load balancer see and balance?
**A:** IPs and ports; it balances connections, choosing a backend when a connection opens.

**Q:** What does an L7 load balancer add?
**A:** Parses requests: content routing, per-request balancing, retries, timeouts, status-code observation, header changes.

**Q:** Why is L4 balancing a problem for gRPC?
**A:** One long-lived HTTP/2 connection carries all requests, so they pin to one backend.

**Q:** What is direct server return?
**A:** An L4 setup where backends reply straight to the client, bypassing the balancer on the (larger) response path.

**Q:** Round robin's weakness?
**A:** Equal counts, not equal load: ignores request cost and server speed.

**Q:** What is weighted round robin used for?
**A:** Uneven server capacity and canary traffic splits.

**Q:** Why does least connections adapt to slow servers?
**A:** In-flight = rate × latency (Little's Law); a slow server accumulates in-flight work and gets fewer new requests.

**Q:** What is the "black hole" problem?
**A:** A server failing fast has few in-flight requests, so load-aware algorithms send it more traffic. Fixed by outlier detection.

**Q:** What is power of two choices?
**A:** Pick two random backends, send to the less loaded. Avoids herding on stale info; max load drops from ~log n/log log n to ~log log n.

**Q:** Why does "least loaded" herd with many balancers?
**A:** They share stale information and all pick the same "emptiest" server at once.

**Q:** Hash-based balancing: benefit and risk?
**A:** Affinity (warm caches, per-key state); uneven with hot keys, and `% N` remaps almost everything when N changes (use consistent hashing).

**Q:** Active vs passive health checks?
**A:** Active: periodic probes of a health endpoint. Passive: watch real traffic for errors/timeouts and eject outliers.

**Q:** Danger of deep health checks?
**A:** A shared dependency hiccup fails every server's check at once, emptying the pool.

**Q:** What is failing open in health checking?
**A:** If all backends look unhealthy, route to all anyway; it's likelier a check problem than total failure.

**Q:** What is connection draining?
**A:** Stop new traffic to a server, let in-flight requests finish (closing keep-alives politely), then remove it.

**Q:** What is slow start?
**A:** Ramping a new backend's share of traffic up gradually so it warms caches and JIT before full load.

**Q:** Ways to implement sticky sessions?
**A:** LB-set cookie, client IP hash, hash on a session/user ID.

**Q:** Why do sticky sessions fight statelessness?
**A:** Uneven load, slow rebalancing, lost sessions when a server dies, slower drains.

**Q:** Right role for stickiness?
**A:** A performance optimization (cache warmth), never a correctness requirement.

**Q:** How does an active-passive LB pair fail over?
**A:** Heartbeats (VRRP); on failure the standby claims the virtual IP via gratuitous ARP or a cloud API call.

**Q:** Main risk of active-passive failover?
**A:** Split brain: both claim the VIP when heartbeats fail but both are alive.

**Q:** How are active-active load balancers fronted?
**A:** ECMP routing (routers hash flows across balancers announcing the same IP via BGP), or multiple DNS records.

**Q:** Limits of DNS-level balancing?
**A:** Caching and TTLs delay failover; no load or request awareness; clients often take the first record. Use it to pick a site/region.

**Q:** What is anycast?
**A:** One IP announced from many locations; routing sends clients to the nearest; withdraw to fail over.

**Q:** What is client-side load balancing?
**A:** Clients get the instance list from service discovery and pick a backend themselves; no central hop, but logic in every client.
