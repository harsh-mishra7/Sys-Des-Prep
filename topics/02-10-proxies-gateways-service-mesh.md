# 10 — Proxies, API Gateways & Service Mesh

> Part 2 — Networking & Communication · Prerequisites: topics 03, 05, 06, 07, 09

Between a client and the service that finally handles a request, there is usually a chain of programs
that receive the request, inspect or change it, and pass it on. These intermediaries are all
**proxies** of some kind. A load balancer (topic 09) is one. So is the corporate box that filters web
traffic, the CDN edge server (topic 14), the API gateway that checks tokens before anything reaches
your services, and the small proxy that runs next to every service instance in a service mesh.

They exist for one reason: some concerns are **cross-cutting**. TLS, authentication, rate limiting,
retries, timeouts, metrics, routing: every service needs them, and implementing them in each service,
in each language, is repetitive and inconsistent. Putting them in a proxy means doing them once, in one
place, for everything that passes through. The cost is an extra hop, an extra component that can fail,
and the risk of moving too much into the middle.

This topic covers which side a proxy works for (forward vs reverse), what an API gateway does at the
edge of a system, how a service mesh does the same for traffic *between* services, and how anything
finds anything else in a world where instances come and go: service discovery.

By the end of this topic you should be able to:

1. Distinguish a forward proxy from a reverse proxy by whose interests it represents, and list what
   each is used for.
2. Explain what an API gateway does (routing, authentication, rate limiting, aggregation, protocol
   translation) and how a gateway turns into a "god" gateway.
3. Explain the service mesh model: sidecar proxies, the data plane vs the control plane, and how mTLS
   between services works.
4. Weigh what a mesh gives (uniform retries, timeouts, telemetry, security) against what it costs
   (latency, resources, operational complexity).
5. Explain client-side vs server-side service discovery, the role of a service registry, and how
   registrations stay accurate.

---

## 1. What a proxy is

A **proxy** is a program that accepts a connection or request from one side and makes its own
connection or request to the other side, relaying between them.

```
client ──────▶ proxy ──────▶ server
        conn 1        conn 2      (two separate connections; the proxy is in the middle of both)
```

Because it's in the middle, a proxy can **see**, **decide** and **change**: log the request, cache the
response, block it, route it somewhere, add a header, retry it, encrypt or decrypt it. Proxies come in
the same layers as load balancers (topic 09): an L4 proxy relays bytes of TCP connections; an L7 proxy
understands HTTP and acts on individual requests.

The most important question about any proxy is **whose side it is on**.

---

## 2. Forward proxy vs reverse proxy

### Forward proxy: acts for the clients

A **forward proxy** sits in front of a group of **clients** and makes requests to the outside world on
their behalf. Servers see requests coming from the proxy, not the individual clients.

```
   ┌─ client ─┐
   ├─ client ─┼──▶ forward proxy ──▶ internet ──▶ any server (sees the proxy's IP)
   └─ client ─┘
   (office / private network)
```

The clients are configured to use it (browser settings, `HTTPS_PROXY` environment variables) or it is
forced on them transparently by the network. Uses:

- **Policy and filtering**: a company blocks certain sites, scans downloads for malware, logs access.
- **Egress control**: servers in a private network may only reach the internet through a proxy that
  allow-lists destinations (e.g. only the payment provider's API). This limits what a compromised
  server can contact.
- **Caching shared content** for many clients (less common now that most traffic is HTTPS).
- **Anonymity / IP masking**: the server sees the proxy's address.

For HTTPS, a forward proxy normally can't see inside: the client sends `CONNECT example.com:443`, the
proxy opens a TCP tunnel, and TLS runs end to end through it (topic 05). Inspecting the content requires
**TLS interception**: the proxy terminates TLS using a certificate from a corporate CA that has been
installed on every client. That's a deliberate weakening of end-to-end security, used in some corporate
networks.

### Reverse proxy: acts for the servers

A **reverse proxy** sits in front of a group of **servers** and receives requests on their behalf.
Clients think they're talking to the server; they don't know (or care) what's behind it.

```
                               ┌──▶ app server 1
client ──▶ internet ──▶ reverse proxy ──▶ app server 2
                               └──▶ static file server
         (client sees only api.example.com)
```

Uses:

- **Load balancing** across servers (topic 09).
- **TLS termination**: certificates and the CPU cost of handshakes live in one place (topic 05).
- **Caching** responses (topic 11) and serving static files.
- **Compression**, HTTP/2 or HTTP/3 to clients while speaking HTTP/1.1 to backends (topic 06).
- **Buffering slow clients**: the proxy reads the slow client's upload fully, then sends it to the app
  server quickly, so the app server's limited workers aren't tied up waiting on a slow network.
- **Security**: hides backend topology and addresses; a single place for request size limits, header
  sanitization, a web application firewall (WAF), and DDoS absorption.
- **Routing**: by host or path to different backends.

Nginx, HAProxy and Envoy are common reverse proxies. A CDN (topic 14) is a globally distributed reverse
proxy. An API gateway is a reverse proxy specialised for APIs.

### The one-line difference

| | Forward proxy | Reverse proxy |
|---|---|---|
| Acts on behalf of | Clients | Servers |
| Who configures it / knows about it | Clients (or their network) | Server operators; clients usually don't know |
| Hides | Client identities from servers | Server topology from clients |
| Typical uses | Filtering, egress control, anonymity | Load balancing, TLS, caching, routing, protection |

Both are the same kind of machine, a proxy; the difference is **whose interests it serves** and where it
sits.

### What proxies hide, and how to recover it

Because a proxy opens its own connection, the backend sees the **proxy's** IP address as the client.
To recover the real client IP, proxies add headers: `X-Forwarded-For: <client IP>, <proxy1 IP>`,
`X-Forwarded-Proto: https` (the standard form is `Forwarded:`). At L4, the **PROXY protocol** prepends
the original addresses to the TCP stream. Backends must only trust these headers when they come from
their own proxies; otherwise any client can claim any IP by sending the header itself.

---

## 3. API gateways

An **API gateway** is a reverse proxy at the edge of a system of services: the single entry point for
external clients. Instead of the mobile app knowing about 30 internal services, it talks to
`api.example.com`, and the gateway takes care of the rest.

```
                                   ┌──▶ users service
mobile app ──┐                     ├──▶ orders service
web app   ───┼──▶  API gateway  ───┼──▶ catalog service
partners  ───┘    (auth, limits,   └──▶ payments service
                   routing, ...)
```

### What a gateway does

- **Routing**: `/users/*` to the users service, `/orders/*` to orders; also version routing (`/v1` vs
  `/v2`) and gradual traffic shifts.
- **Authentication**: validate the caller's token (a JWT signature, an OAuth access token, an API key;
  topic 54) once at the edge. Downstream services receive a trusted identity (e.g. a header with the
  user ID, set only by the gateway) instead of each re-implementing token validation.
- **Coarse authorization**: "only partner keys may call `/partner/*`". Fine-grained decisions ("can
  user 42 edit order 7?") belong in the service that owns the data.
- **Rate limiting and quotas** per API key, user or IP (topic 45).
- **Protocol translation**: REST/JSON or GraphQL outside, gRPC inside (topic 07); gRPC-Web for
  browsers; WebSocket termination (topic 08).
- **Aggregation**: one client request fans out to several services and the gateway combines the
  results, saving the client round trips (especially valuable on mobile; the same motivation as GraphQL
  in topic 07).
- **Cross-cutting edge policy**: TLS termination, CORS headers, request size limits, request IDs and
  tracing headers (topic 46), access logging, caching of public responses.

### Backend for frontend (BFF)

Different clients need different things: the mobile app wants small, aggregated payloads; the web app
wants richer pages; partners want a stable, versioned public API. Rather than one gateway trying to
serve all of them, the **BFF** pattern gives each client type its own thin gateway, owned by the team
that builds that client:

```
mobile app ──▶ mobile BFF ──┐
web app    ──▶ web BFF    ──┼──▶ internal services
partners   ──▶ public API ──┘
```

### The risk: the "god" gateway

Because every request passes through it, the gateway is a tempting place to put logic. Over time:

```
Year 1: routing, auth, rate limits
Year 2: + aggregation for the home screen
Year 3: + "if user is premium, call pricing v2", + order validation rules,
          + transforming responses for each app version, + feature flags ...
Year 4: every feature change needs a gateway change; one team owns it; deploys are scary
```

This is the **god gateway**: business logic has leaked into shared infrastructure. Problems:

- **Coupling and bottleneck**: every team's change goes through one codebase and one team's review
  and release cycle. The gateway becomes the thing microservices were supposed to remove.
- **Blast radius**: a bad gateway deploy or config change breaks *everything* at once (topic 48).
  Config changes are a leading cause of outages (topic 47).
- **Unclear ownership**: rules about orders live partly in the orders service and partly in the
  gateway; nobody has the full picture.
- **Performance**: heavy aggregation and transformation make the gateway CPU-bound and add latency to
  every request.

The guideline: **keep the gateway about edge concerns** (who is calling, are they allowed in, how much,
where does it go) and keep **business rules in services**. If aggregation is needed, put it in a BFF or
GraphQL layer owned by the client team, not in the shared platform gateway.

A gateway also needs the same high availability as a load balancer (topic 09): many instances, behind
an L4 tier, with careful, staged config rollouts.

---

## 4. Service mesh

An API gateway handles **north-south** traffic (clients entering the system). In a system of many
services, most traffic is **east-west**: service to service. That traffic needs the same things: TLS,
retries, timeouts, load balancing, metrics, access control.

### The problem a mesh solves

Without a mesh, each service implements these in its own code, typically with a shared library:

```
orders service (Java)   ── uses RPC library v3.2: retries, timeouts, metrics, TLS
payments service (Go)   ── uses a different library: different retry defaults
catalog service (Python)── forgot to set timeouts entirely
```

Problems: every language needs its own library; upgrading a library means redeploying every service;
behaviour differs between services (one retries 3 times, another 5, multiplying load; topic 43);
security policies are inconsistent. The idea of a **service mesh** is to move this networking behaviour
**out of the application** into a proxy that runs alongside it.

### Sidecars and the data plane

In the classic design, every service instance gets its own **sidecar proxy** (commonly Envoy) running
next to it, in the same pod or on the same host. All of the instance's inbound and outbound network
traffic is transparently redirected through its sidecar (usually by iptables rules), so the
application just calls `http://payments/charge` as if nothing were there.

```
┌────── pod A ───────┐                          ┌────── pod B ───────┐
│ orders app         │                          │ payments app       │
│   │ plain HTTP     │                          │   ▲ plain HTTP     │
│   ▼ (localhost)    │      mTLS over network   │   │ (localhost)    │
│ sidecar proxy ─────┼─────────────────────────▶│ sidecar proxy      │
└────────────────────┘                          └────────────────────┘
```

On each call, the outbound sidecar does service discovery, load balancing (per request, e.g. power of
two choices; topic 09), timeouts, retries with budgets, circuit breaking (topic 44), mTLS, and records
metrics and trace spans (topic 46). The inbound sidecar terminates mTLS, checks whether the caller is
allowed, and records its own metrics.

The set of all these proxies, which actually carry the traffic, is the **data plane**.

### The control plane

Hundreds or thousands of sidecars need configuration: which instances exist for each service, what
retry policy applies, which services may talk to which, what certificates to use. The **control plane**
(for example Istio's istiod, or Linkerd's control plane) is a separate set of services that:

- Watches the platform (e.g. Kubernetes) for services and instances, and pushes **endpoint lists** to
  every sidecar (service discovery; section 5).
- Translates high-level policy ("retry payments calls twice on `UNAVAILABLE`", "send 5% of traffic to
  catalog v2", "only orders may call payments") into proxy configuration and pushes it out.
- Acts as a **certificate authority**, issuing and rotating short-lived certificates for each workload.

```
                ┌─────────────── control plane ───────────────┐
                │  service registry · policy · certificate CA │
                └───────┬─────────────────┬─────────────┬─────┘
                  config, endpoints, certs (pushed)     │
                        ▼                 ▼             ▼
data plane:      [sidecar]──[sidecar]──[sidecar]──[sidecar]   ← carry every request
                    app A      app B      app C      app D
```

Crucially, the control plane is **not on the request path**. If it goes down, the sidecars keep working
with their last-known configuration; you just can't change config, and certificates eventually expire if
it stays down too long. This separation of a fast, simple, distributed data plane from a slower,
centralized control plane appears throughout networking (routers, SDN) and in many distributed systems.

### mTLS between services

Ordinary TLS (topic 05) authenticates the **server** to the client. **Mutual TLS (mTLS)** also
authenticates the **client** to the server: both sides present certificates and verify each other's.

In a mesh:

1. Each workload gets an **identity**, tied to what it is rather than where it is: for example
   `spiffe://example.com/ns/shop/sa/orders` (the SPIFFE standard for workload identities).
2. The control plane's CA issues each sidecar a **short-lived certificate** (hours to a day) carrying
   that identity, and rotates it automatically.
3. When orders calls payments, the two sidecars run a TLS handshake in which **both** present
   certificates. Each verifies the other's certificate against the mesh CA.
4. The payments sidecar now knows, cryptographically, that the caller is `orders`, and can enforce
   policy: "only `orders` and `refunds` may call payments".

What this gives:

- **Encryption in transit** inside the network, with zero application changes.
- **Strong service identity**, independent of IP addresses (which change constantly as pods come and
  go, so IP-based firewall rules are fragile).
- **A foundation for zero-trust networking**: being "inside the network" grants nothing; every call is
  authenticated and authorized.

Short-lived certificates matter: revoking certificates is notoriously unreliable, so instead certificates
expire quickly and are continuously reissued. A stolen one is useful only briefly.

### What a mesh costs

- **Latency**: every call passes through two extra proxies (outbound and inbound sidecar), typically
  adding around a millisecond or less per hop at p50, more at the tail. In a deep call chain, this adds
  up.
- **Resources**: a proxy per instance. With thousands of pods, sidecars' CPU and memory become a
  significant share of the cluster.
- **Operational complexity**: a control plane to run and upgrade, a new class of failures (a bad config
  push that breaks all traffic; certificate expiry), and debugging that now spans app, sidecar and
  control plane.
- **Leaky abstraction**: the mesh can retry, but only the application knows whether an operation is
  idempotent (topic 07). Mesh retries on non-idempotent calls can duplicate side effects. The app still
  needs deadlines, idempotency keys and sensible fallbacks.

Newer designs reduce the cost by moving away from one sidecar per pod: **per-node proxies** shared by all
pods on a host (Istio's "ambient" mode splits a per-node L4 layer for mTLS from optional L7 proxies), or
**proxyless** meshes where gRPC libraries take configuration directly from the control plane.

A mesh makes sense when there are **many services in several languages** and consistent security and
observability matter. For a handful of services, a good shared library or a few explicit proxies is
often simpler.

### Gateway vs mesh

| | API gateway | Service mesh |
|---|---|---|
| Traffic | North-south (external clients in) | East-west (service to service) |
| Location | At the edge, a shared tier | Next to every instance (sidecar) or every node |
| Focus | Client-facing APIs: auth of end users, quotas, API products, protocol translation | Service identity (mTLS), retries/timeouts, per-request balancing, telemetry |
| Typical identity | End users, API keys | Workloads (service identities) |

They complement each other; many meshes include an **ingress gateway** that is the edge entry point
into the mesh.

---

## 5. Service discovery

In a static world, you'd put server addresses in a config file. In a modern platform, instances are
created and destroyed constantly (autoscaling, deploys, failures), and each gets a new IP. **Service
discovery** answers: "where are the healthy instances of service X right now?"

### The service registry

At the heart of discovery is a **registry**: a database of `service → [instance addresses]`.
Examples: Consul, etcd or ZooKeeper (topic 31), Eureka, the Kubernetes API server's Endpoints objects.

How entries get in and stay correct:

- **Self-registration**: the instance registers itself on startup and deregisters on shutdown.
- **Third-party registration**: the platform (e.g. Kubernetes, or a registrar watching a cluster
  manager) registers instances as it starts them.
- **Staying accurate**: a crashed instance can't deregister itself. So registrations are kept alive by
  **heartbeats** or **health checks**, and entries expire if they stop (a **lease** with a TTL; topic
  31). Until expiry, a dead instance may still be handed out, so clients must also tolerate failures
  (retries, passive health checks).

The registry is critical infrastructure: if it's unavailable, nothing can find anything. Registries are
usually replicated (consensus-based ones like etcd and Consul; topic 30), and clients **cache** the last
known instance list so a registry outage doesn't immediately break traffic.

### Client-side discovery

The **client** queries the registry (or subscribes to changes), gets the list of instances, and picks
one itself using a load-balancing algorithm.

```
                 1. "where is payments?"
orders ─────────────────────────────────▶ registry
   │   ◀──── [10.0.1.7, 10.0.3.2, 10.0.5.9]
   │
   │ 2. pick one (e.g. power of two choices), call it directly
   └─────────────────────────────────────▶ 10.0.3.2 (payments)
```

- No extra network hop; no central load balancer to scale.
- Clients can do smart, per-request balancing (good for gRPC).
- Every client needs registry integration and balancing logic: a library per language, kept up to date.
  The service mesh sidecar is essentially **client-side discovery moved into a proxy**, which removes
  the per-language library problem.

### Server-side discovery

The client sends the request to a **well-known address** (a load balancer or router); that component
queries the registry and forwards to an instance.

```
orders ──▶ payments.internal (load balancer / router) ──▶ registry lookup ──▶ 10.0.3.2
```

- Clients are simple: they only know one name.
- Discovery and balancing logic live in one place.
- An extra hop, and the load balancer must itself be highly available and scaled (topic 09).

Kubernetes Services are a well-known example: a stable virtual IP and DNS name per service; the node's
networking layer (kube-proxy) forwards connections to a current pod. Note this is connection-level (L4),
so it has the gRPC pinning problem from topics 07 and 09.

### DNS as service discovery

DNS (topic 04) is the oldest discovery mechanism: resolve `payments.internal` and get instance IPs, or
use SRV records to get host and port. It's universal, since every client already speaks DNS. But
caching and TTLs make it slow to reflect changes; some clients cache resolved addresses for the
process's lifetime; and DNS carries no health or load information. Discovery systems like Consul expose
a DNS interface for compatibility, typically with very short TTLs.

| | Client-side | Server-side | DNS-based |
|---|---|---|---|
| Who picks the instance | The client (library or sidecar) | A load balancer / router | The client, from DNS answers |
| Extra hop | No | Yes | No |
| Client complexity | High (unless sidecar) | Low | Low |
| Per-request balancing | Yes | If the balancer is L7 | No (connection-level at best) |
| Freshness | Real-time (watch/subscribe) | Real-time at the balancer | Limited by TTLs and caching |

---

## 6. How it connects

- **TCP and connection reuse** (topic 03): every proxy is two connections; proxies pool upstream
  connections and must manage ports and idle timeouts.
- **DNS** (topic 04) is the simplest form of service discovery and the outermost routing layer.
- **TLS termination** (topic 05) happens at reverse proxies and gateways; **mTLS** extends TLS to
  authenticate clients too.
- **API styles** (topic 07): gateways translate between REST/GraphQL and gRPC; GraphQL and BFFs are
  forms of aggregation; idempotency decides which retries a proxy or mesh may safely do.
- **Real-time connections** (topic 08) often terminate at the gateway or edge proxy.
- **Load balancing** (topic 09) is what most proxies do; client-side balancing depends on service
  discovery.
- **Caching and CDNs** (topics 11 and 14) live in reverse proxies at the edge.
- **Coordination services, leases** (topic 31) and **consensus** (topic 30) underpin service registries.
- **Retries, timeouts, circuit breakers** (topics 43–44) are configured in meshes; **rate limiting**
  (topic 45) in gateways; **tracing** (topic 46) relies on proxies propagating trace headers.
- **Config changes and blast radius** (topics 47–48): a shared gateway or mesh control plane is a place
  where one bad change breaks everything.
- **Authentication and authorization** (topic 54): gateways validate end-user tokens; meshes provide
  service identity.

## Common misconceptions

- **"A reverse proxy is just a load balancer."** → Load balancing is one job; reverse proxies also
  terminate TLS, cache, compress, buffer slow clients, route and protect.
- **"Forward and reverse proxies are different technologies."** → They're the same kind of program;
  the difference is whose side it's on and who configures it.
- **"The API gateway is the right place for business logic because every request passes through it."**
  → That's exactly why it isn't: it couples every team to one component and gives every bug maximum
  blast radius. Keep it to edge concerns.
- **"The control plane handles requests."** → It only configures the data plane. Sidecars keep serving
  with their last config if the control plane is down.
- **"mTLS is just encryption."** → The point is mutual *authentication*: each service gets a verifiable
  identity, enabling service-to-service authorization.
- **"With a service mesh, the app doesn't need to think about failures."** → The mesh can't know which
  operations are safe to retry, what fallback makes sense, or what deadline the user needs.
- **"Kubernetes Services balance gRPC requests."** → A ClusterIP Service balances connections (L4);
  gRPC's long-lived connections still pin to one pod.
- **"Trust `X-Forwarded-For`."** → Only if your own proxy set it; clients can send any value.

## Further reading (supplements)

- Envoy proxy documentation: "What is Envoy" and the architecture overview (listeners, clusters, xDS)
- Istio documentation: "Architecture" and "Security" concepts (mTLS, identities)
- William Morgan, "What's a service mesh? And why do I need one?" (Buoyant / Linkerd blog)
- Chris Richardson, microservices.io: "API Gateway / Backends for Frontends", "Client-side discovery",
  "Server-side discovery" and "Service registry" pattern pages
- Sam Newman, *Building Microservices* (2nd ed.), chapters on communication and security
- SPIFFE project (spiffe.io): "SPIFFE concepts" overview of workload identity

---

## Questions

1. Explain the difference between a forward proxy and a reverse proxy in terms of whose interests each
   represents. Give two uses of each.

2. A forward proxy at a company allows employees to browse HTTPS sites. How does it handle the TLS
   traffic, and what would it take for it to inspect the content?

3. Why does a reverse proxy in front of application servers help with slow clients uploading large
   request bodies?

4. A backend logs client IPs, and every request appears to come from 10.0.0.5. Why? How is the real
   client IP recovered, and what's the security pitfall?

5. List five responsibilities of an API gateway. Which kind of authorization belongs in the gateway, and
   which in the service?

6. Describe how an API gateway becomes a "god" gateway and the specific harms that causes. What's the
   guideline that prevents it?

7. What is the backend-for-frontend pattern, and what problem does it solve compared with one shared
   gateway?

8. A company has 80 services in Java, Go, Python and Node. Each has its own retry and timeout behaviour,
   and TLS between services is patchy. Explain how a service mesh addresses this, naming the data plane
   and control plane roles.

9. The mesh control plane goes down for 20 minutes. What keeps working, what stops working, and what
   would happen if it stayed down for several days?

10. Walk through how mTLS lets the payments service enforce "only the orders service may call me". Why
    are mesh certificates short-lived?

11. A mesh is configured to retry all failed calls 3 times. What can go wrong, and what does this say
    about what a mesh can and can't take over from the application?

12. What does a service mesh cost? When would you not adopt one?

13. Compare client-side and server-side service discovery. Where does a mesh sidecar fit?

14. An instance crashes without deregistering from the service registry. What happens, and what
    mechanisms limit the damage?

15. Why is plain DNS a weak service discovery mechanism for rapidly changing instances?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** A **forward proxy** acts for **clients**: it sits in front of a group of clients and makes
outbound requests on their behalf, hiding them from servers. Uses: content filtering/policy in an
office, egress control for servers in a private network (allow-listed destinations), anonymity, shared
caching. A **reverse proxy** acts for **servers**: it sits in front of them and receives requests on
their behalf, hiding their topology from clients. Uses: load balancing, TLS termination, caching,
compression, routing by path/host, protection (WAF, size limits, DDoS absorption).

**2.** The browser sends `CONNECT site:443` to the proxy, which opens a TCP connection to the site and
then blindly relays bytes in both directions. TLS runs end to end between browser and site, so the proxy
sees only the destination host (and traffic volume), not the content. To inspect content, the proxy must
do **TLS interception**: terminate the browser's TLS with a certificate for the site generated on the
fly, signed by a corporate CA that has been installed as trusted on every employee device, then open its
own TLS connection to the real site.

**3.** Application servers often have a limited pool of workers. If a client on a slow network takes
30 seconds to upload a body, a worker connected directly to it is tied up for 30 seconds doing nothing.
A reverse proxy **buffers** the request: it receives the whole body from the slow client (cheaply, with
an event-driven design), then forwards it to the app server over a fast local network in milliseconds.
The same applies to slow clients reading large responses.

**4.** 10.0.0.5 is the proxy (or load balancer): it opened its own connection to the backend, so the
backend's TCP peer is the proxy. The real IP is passed in a header (`X-Forwarded-For`, or the standard
`Forwarded`) at L7, or with the PROXY protocol at L4. The pitfall: clients can send `X-Forwarded-For`
themselves. The backend must only trust values added by its own proxies (e.g. take the address added by
the last trusted hop), otherwise anyone can spoof their IP to evade rate limits or IP allow-lists.

**5.** Routing to services; authenticating callers (validating tokens/API keys) once at the edge; rate
limiting and quotas; protocol translation (REST/GraphQL outside, gRPC inside; gRPC-Web); request
aggregation; TLS termination, CORS, request IDs, logging. **Coarse** authorization belongs in the
gateway ("is this caller allowed to use this API at all?", "partner keys only on `/partner/*`").
**Fine-grained**, data-dependent authorization ("may user 42 refund order 7?") belongs in the service
that owns the data and its rules.

**6.** Because every request passes through it, teams keep adding things: aggregation for specific
screens, per-app-version response transformations, pricing rules, validation, feature flags. Harms:
every feature change needs a gateway change, so the gateway team becomes a **bottleneck** and teams are
**coupled**; a bad deploy or config change has **maximum blast radius**; business rules are split between
services and gateway, so **ownership is unclear**; heavy logic makes it slow and expensive. Guideline:
the gateway handles **edge concerns only** (identity, admission, quotas, routing); business logic stays
in services, and client-specific aggregation goes in BFFs owned by client teams.

**7.** BFF gives each client type (mobile, web, partner API) its own thin gateway layer, owned by the
team building that client. One shared gateway serving all clients accumulates conflicting
client-specific logic and couples all client teams to one component. With BFFs, each client gets exactly
the aggregation and payload shapes it needs, and its team can change them independently, while the
shared platform gateway (if any) stays small.

**8.** A mesh moves networking behaviour out of the apps into a **sidecar proxy** beside each instance;
all traffic goes through it transparently. The sidecars (**data plane**) do discovery, per-request load
balancing, timeouts, retries with budgets, circuit breaking, mTLS and telemetry, identically for every
language, with no library to maintain per language. The **control plane** watches the platform for
instances, turns central policy into proxy config, pushes endpoints and config to every sidecar, and acts
as CA issuing and rotating certificates. Retry/timeout policy and mTLS become uniform and centrally
managed; changing them doesn't require redeploying 80 services.

**9.** The data plane keeps working: sidecars continue routing, balancing, retrying and doing mTLS with
their **last-known** configuration and endpoints, because the control plane is not on the request path.
What stops: config and policy changes, and updates to endpoint lists, so new instances (from deploys or
autoscaling) won't receive traffic and removed ones may still be targeted until passive health checks
eject them. Over several days, workload **certificates would expire** without rotation, and mTLS
connections would start failing, causing an outage.

**10.** The control plane's CA issues each workload a certificate carrying its identity (e.g. a SPIFFE
ID for the orders service account). When orders calls payments, the sidecars do a TLS handshake where
**both** present certificates and verify them against the mesh CA. The payments sidecar thus learns,
cryptographically, that the caller is `orders` and checks it against an authorization policy allowing
only `orders`; anything else is rejected before reaching the app. Certificates are short-lived because
revocation is unreliable: rather than revoking, certificates expire within hours and are reissued
automatically, so a stolen key or certificate is only useful briefly.

**11.** (a) Non-idempotent calls (charge a card, create an order) may be **duplicated** when a request
succeeded but the response was lost or timed out. (b) Retries at several layers multiply: if the app, the
sidecar and the next service each retry 3 times, a failing deep dependency can see 27× the load, a retry
storm (topic 43). (c) Retrying after the user's deadline has passed wastes work. The mesh can apply
generic mechanics, but only the application knows which operations are idempotent, what deadline the
user has, and what fallback is acceptable; it still needs idempotency keys, deadlines and sensible
retry policy per route.

**12.** Costs: extra latency (two proxy hops per call), CPU and memory for a proxy per instance,
operational complexity (control plane upgrades, config push failures, certificate expiry, harder
debugging across app/sidecar/control plane), and a learning curve. You might not adopt one with a small
number of services, a single language where a shared library already provides retries, timeouts and
metrics, or a team without the capacity to operate it; also for very latency-sensitive paths where the
extra hops matter.

**13.** **Client-side**: the client queries/watches the registry and picks an instance itself; no extra
hop, no central bottleneck, smart per-request balancing; but each client needs discovery and balancing
logic per language. **Server-side**: the client calls a stable address (load balancer/router) that looks
up instances and forwards; simple clients, centralized logic; but an extra hop and a component to scale
and keep available. A mesh sidecar is **client-side discovery moved into a local proxy**: the app sees a
simple address like server-side discovery, but the decision is made locally beside the client, with no
central hop.

**14.** The registry still lists the dead instance, so clients (or load balancers) keep sending some
requests to it, which fail with connection errors or timeouts. Limiting mechanisms: registrations are
**leases** with a TTL kept alive by heartbeats or registry health checks, so the entry expires within
seconds; clients use **passive health checks / outlier detection** to stop using an instance that fails;
and **retries** of idempotent requests on another instance hide the failures from users.

**15.** Resolvers and clients cache answers for the TTL, and some clients (or language runtimes) cache
resolved addresses much longer, even for the process lifetime, so new instances aren't used and dead ones
are still called. DNS carries no health or load information, and returns whole lists that clients often
use naively (first record). It also only gives connection-level balancing. Very short TTLs help but
increase DNS load and don't fix clients that ignore them.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What is a proxy?
**A:** A program that accepts a connection/request from one side and makes its own to the other, relaying and able to inspect, change or route.

**Q:** Forward vs reverse proxy?
**A:** Forward acts for clients (hides them from servers); reverse acts for servers (hides them from clients).

**Q:** Uses of a forward proxy?
**A:** Content filtering, egress control/allow-listing, anonymity, shared caching.

**Q:** Uses of a reverse proxy?
**A:** Load balancing, TLS termination, caching, compression, buffering slow clients, routing, protection (WAF, limits).

**Q:** How does a forward proxy handle HTTPS normally?
**A:** `CONNECT` tunnel: relays encrypted bytes; can't see content unless it intercepts TLS with a client-trusted corporate CA.

**Q:** How does a backend learn the real client IP behind a proxy?
**A:** `X-Forwarded-For` / `Forwarded` headers (L7) or PROXY protocol (L4); trust them only from your own proxies.

**Q:** What is an API gateway?
**A:** A reverse proxy at the system's edge: single entry point doing routing, auth, rate limits, translation, aggregation.

**Q:** Gateway authorization vs service authorization?
**A:** Gateway: coarse (may this caller use this API?). Service: fine-grained, data-dependent rules.

**Q:** What is a god gateway?
**A:** A gateway that has accumulated business logic: a coupling bottleneck with maximum blast radius and unclear ownership.

**Q:** Rule for what goes in a gateway?
**A:** Edge concerns only (identity, admission, quotas, routing); business logic stays in services.

**Q:** What is a BFF?
**A:** Backend for frontend: a thin gateway per client type, owned by that client's team.

**Q:** North-south vs east-west traffic?
**A:** North-south: clients into the system (gateway). East-west: service to service (mesh).

**Q:** What problem does a service mesh solve?
**A:** Inconsistent, per-language networking logic (retries, timeouts, TLS, metrics) in every service.

**Q:** What is a sidecar proxy?
**A:** A proxy running beside each service instance, transparently handling all its inbound and outbound traffic.

**Q:** Data plane vs control plane?
**A:** Data plane: the proxies carrying requests. Control plane: configures them (endpoints, policy, certificates); not on the request path.

**Q:** What happens if the mesh control plane is down?
**A:** Traffic continues with last-known config; no config/endpoint updates; certificates eventually expire.

**Q:** What is mTLS?
**A:** TLS where both client and server present and verify certificates, so each side authenticates the other.

**Q:** What does mTLS give a mesh?
**A:** Encryption in transit plus verifiable service identity, enabling service-to-service authorization (zero trust).

**Q:** Why are mesh certificates short-lived?
**A:** Revocation is unreliable; quick expiry plus automatic rotation limits the value of a stolen certificate.

**Q:** What does a mesh cost?
**A:** Extra latency per hop, proxy CPU/memory per instance, control plane operations, new failure modes.

**Q:** What can't a mesh decide for the app?
**A:** Which operations are idempotent (safe to retry), the user's deadline, and the right fallback.

**Q:** What is a service registry?
**A:** A database of service → healthy instance addresses (Consul, etcd, ZooKeeper, Eureka, Kubernetes endpoints).

**Q:** How do registry entries stay accurate?
**A:** Heartbeats/health checks with TTL leases; entries expire if the instance stops renewing.

**Q:** Client-side discovery?
**A:** The client looks up instances and picks one itself: no extra hop, smart balancing, but logic in every client.

**Q:** Server-side discovery?
**A:** The client calls a stable address; a load balancer/router looks up instances and forwards: simple clients, extra hop.

**Q:** Where does a mesh sidecar fit in discovery?
**A:** Client-side discovery moved into a local proxy: simple app, local per-request decisions.

**Q:** Why is DNS weak for service discovery?
**A:** Caching/TTLs delay changes, some clients cache forever, no health or load info, connection-level balancing only.
