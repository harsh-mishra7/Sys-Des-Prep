# Day 3: Answers

Read only after attempting `questions.md`.

---

## HTTP versions

**1.** Pipelining lets the client send several requests without waiting, but responses must come back
**in the same order** as the requests, because HTTP/1.1 messages carry nothing that says which request
a response belongs to. A slow first response blocks every response behind it, even ones that are
ready: head-of-line blocking at the HTTP layer. Fixing it would need an identifier on every message,
which is a change to the wire format, and that's exactly what HTTP/2's stream IDs are. Buggy proxies
that mishandled pipelined requests made it worse, so browsers just turned it off.

**2.** They worked around HTTP/1.1's **one-request-at-a-time** connections and the browser's limit of
~6 connections per origin. Sharding got more connections; concatenation and sprites reduced the number
of requests. Under HTTP/2: sharding forces **extra connections** (extra DNS lookups, TCP and TLS
handshakes, separate cold congestion windows) and splits traffic that could share one well-warmed
connection. Concatenation hurts caching: changing one small file invalidates the whole bundle, and the
browser can't start using any of it until the large file arrives. With cheap multiplexed requests,
smaller separately-cacheable files are often better.

**3.** A stream is one request/response exchange within an HTTP/2 connection, with its own ID. Every
message is broken into frames, and every frame carries its stream ID, so frames from many streams can
be interleaved on one connection and reassembled by ID at the other end. A slow response doesn't block
others; their frames flow in between. One connection is enough because it can carry hundreds of
concurrent streams, and one connection means one handshake and one congestion window that grows large
because all traffic shares it.

**4.** Most headers repeat almost identically on every request on a connection (cookies, auth tokens,
user agent, accept headers). Compressing each request on its own can only find redundancy *inside*
that request. A **shared dynamic table** lets the second and later requests refer to headers already
sent, by a small index, so repeated headers shrink to a byte or two. HPACK's design (indexed tables
rather than general-purpose compression like gzip) also avoids the compression side-channel attacks
(CRIME) that let attackers infer secrets like cookies from compressed sizes.

**5.** HTTP/2 multiplexes everything over **one TCP connection**. TCP delivers bytes strictly in order,
so when any packet is lost, *every* stream waits for its retransmission, even streams whose data
already arrived. With 2% loss, these stalls happen constantly and hit everything at once. HTTP/1.1 with
six connections spreads the risk: a loss on one connection stalls only that connection's request, and
each connection has its own congestion window, so one loss also halves only one sixth of the total
sending rate.

**6.** The problem (TCP head-of-line blocking) comes from TCP's core contract: one ordered byte stream.
TCP is implemented in operating system kernels and inspected by middleboxes (firewalls, NATs, load
balancers) across the internet. Changes take many years to deploy everywhere, and middleboxes often
drop packets with unfamiliar TCP options (ossification). UDP gives almost nothing but passes through
nearly everything, so QUIC can implement reliability, ordering per stream, and congestion control in
user space, and encrypt its own headers so it can keep evolving.

**7.** With HTTP/2 over TCP, the connection is identified by the 4-tuple. The phone's IP address
changes, so the connection is dead. The client has to notice (often only after a timeout), open a new
TCP connection, do a new TLS handshake, restart from a cold congestion window, and re-request (or
resume, if the server supports range requests). With HTTP/3, QUIC identifies the connection by a
**connection ID**, not the 4-tuple. The client keeps sending from its new address with the same
connection ID; the server validates the new path and the connection continues. **Connection
migration** makes the difference.

**8.** TCP + TLS 1.2: **3** RTTs. TCP + TLS 1.3: **2**. New QUIC: **1** (transport and TLS 1.3
handshakes are combined). Resumed QUIC with 0-RTT: **0** (the request goes in the first flight). 0-RTT
data can be **captured and replayed** by an attacker, because it's sent before the handshake completes
and so isn't protected against replay. Replaying a `GET` is harmless; replaying "transfer $100" is
not. So only idempotent (ideally safe) requests should use 0-RTT.

**9.** Very little gain, possibly a loss. Inside one data centre RTTs are sub-millisecond, so saving a
handshake round trip barely matters (especially with long-lived pooled connections), and packet loss
is rare, so TCP head-of-line blocking rarely triggers. Connection migration doesn't matter for servers
with fixed addresses. Meanwhile QUIC costs **more CPU** per byte (user-space processing, less hardware
offload) and has less mature tooling for internal debugging. HTTP/3's benefits are mostly at the edge,
for users on lossy, high-latency, changing networks.

**10.** The server advertises HTTP/3 in an `Alt-Svc` response header on an HTTP/2 or HTTP/1.1
response (or in a DNS `HTTPS` record, which can be seen before connecting). The browser remembers it
and tries QUIC on later connections, often racing it against a TCP connection and using whichever
succeeds. If UDP is blocked, the QUIC attempt fails or times out and the browser **falls back to
HTTP/2 over TCP**. This is why every HTTP/3 deployment must still support HTTP/2.

## REST & HTTP semantics

**11.** **Safe**: the request asks for no change to server state (read-only from the client's point of
view). **Idempotent**: sending it once or many times leaves the server in the same state. `PUT` and
`DELETE` are idempotent but not safe. `POST` is neither (and `PATCH` isn't guaranteed to be either).
Every safe method is also idempotent.

**12.** Yes. Idempotency is about the **effect on server state**, not the response. After the first
`DELETE` (which probably succeeded before the timeout), the order is gone. After the retry, the order
is still gone. Same state. The `404` just reports that there's nothing left to delete. Clients retrying
a `DELETE` should treat a `404` as "already done".

**13.** The client doesn't know whether the server processed the request: the request may have been
lost, or the payment may have gone through and only the response was lost. `POST` isn't idempotent, so
retrying could charge twice. The fix is an **idempotency key**: the client generates a unique ID for
this logical payment and sends it on every attempt. The server stores each key with its outcome; when
it sees a key it has already processed, it returns the stored result instead of charging again. (It
also needs to handle two attempts with the same key arriving concurrently, e.g. with a unique
constraint on the key; Day 23.)

**14.** Generic infrastructure can't understand your application, but it does know HTTP's method
semantics. `GET` and `PUT` are defined as idempotent, so repeating them is safe by contract, whatever
the application does. `POST` makes no such promise, so an automatic retry could duplicate a side
effect (an order, a payment). This is the uniform interface paying off: method semantics let
intermediaries make correct decisions without application knowledge.

**15.** (a) **Monitoring and alerting** count it as a success, so error-rate dashboards and SLO alerts
stay green during an outage. (b) **Retry logic** in clients and proxies doesn't retry, because 2xx
means success. (c) **Load balancer passive health checks / outlier detection** won't eject the failing
instance, because it isn't returning 5xx. (d) **Caches** (CDN, reverse proxy, browser) may store the
error response and serve it to others as if it were valid content. Also circuit breakers never open.

**16.** `401` = **not authenticated** ("I don't know who you are"; send credentials). `403` =
authenticated but **not authorized** ("I know who you are, and you can't do this"). `502 Bad Gateway`:
a proxy got an invalid response or none (the upstream crashed, reset the connection, or sent garbage).
`503 Service Unavailable`: the server is temporarily unable to handle the request (overloaded, in
maintenance), often with `Retry-After`. `504 Gateway Timeout`: a proxy gave up waiting for the
upstream to respond.

**17.** Nothing certain. A `504` means a **proxy stopped waiting**; the backend may have failed, may
still be running, or may have completed the transfer after the proxy gave up. The client must not
blindly retry a non-idempotent request. It should retry **with the same idempotency key** (so the
server returns the original outcome if the transfer happened), or query the transfer's status by a
client-generated ID before deciding.

**18.** Each user's `GET` returns the page with an `ETag` (a version, say `"v5"`). Each saves with
`PUT` and `If-Match: "v5"`. The server applies the first `PUT` because the current version is `v5`,
and the page becomes `v6`. The second `PUT` still says `If-Match: "v5"`, which no longer matches, so
the server rejects it with **`412 Precondition Failed`**. That user must re-fetch, merge or redo their
edit, and try again. This is optimistic concurrency control (compare-and-set) built into HTTP, and it
prevents the lost update.

**19.** **Duplicates**: if new posts are inserted at the top between page requests, every item shifts
down; the item that was #50 is now #51 and shows up again at the start of the next page (deletions
cause skips instead). **Slowness**: `OFFSET 10000` makes the database produce and throw away 10,000
rows before returning 50, so cost grows with depth. **Cursor (keyset) pagination** fixes both: the
client passes the sort key of the last item it saw (`?after=<id or timestamp>`), and the server queries
`WHERE key < :after ORDER BY key DESC LIMIT 50`. An index serves this directly at constant cost, and
insertions elsewhere don't shift the window. The trade-off is that you can't jump to page N.

## RPC, gRPC, GraphQL

**20.** Hiding the network encourages code that treats remote calls like local ones: calling them in
loops, assuming they always return, ignoring timeouts and partial failure. Differences: (1) **latency**
is orders of magnitude higher and highly variable; (2) a call can **time out with an unknown outcome**
(it may or may not have run); (3) the remote side can **fail partway** or be unavailable; (4) arguments
must be **serialized and copied** (no pointers or shared memory); (5) the other side may run a
**different version** of the code.

**21.** Field numbers are short (a varint tag combining number and wire type, usually 1 byte), so
messages are much smaller than with repeated field names. More importantly, numbers decouple the wire
format from naming: you can rename fields freely, and old and new code agree on what each field is by
number. **Safe**: adding new fields (old readers skip unknown numbers; new readers use defaults for
missing fields), removing fields (if the number is reserved), renaming fields. **Never**: reuse a field
number for a different field, or change a field's type in an incompatible way.

**22.** Old clients and servers, and old stored data, still use field 4 as a string. A new reader
seeing field 4 with length-delimited wire type where it expects a varint will either error or drop it;
worse, with compatible wire types, data is silently **misinterpreted** (an old value read as a
completely different field). Protobuf prevents this with `reserved 4;` (and `reserved "nickname";`) when
the field is removed: the compiler then refuses any future definition that reuses that number or name.

**23.** A timeout is local: "I'll wait 2 s for this call". A **deadline** is an absolute point in time
for the whole operation that travels with the request. Each service passes along the *remaining* time
to its own downstream calls. Without propagation, a frontend may give up after 1 s while the services
behind it keep working for many more seconds, each with its own fresh timeout, wasting capacity on a
result nobody will use; under load, that wasted work can push the system into overload. With
propagation, every service can stop as soon as the deadline passes, and a service can refuse work that
it can't finish in the remaining time.

**24.** gRPC multiplexes all calls over one (or a few) long-lived HTTP/2 connections. A **layer 4**
load balancer only chooses a backend when a **connection** is opened; after that, every request on that
connection goes to the same backend. The existing clients' connections are already pinned to the old
backends, so the new ones only get traffic from new connections, which rarely open. Fixes: use a
**layer 7** (HTTP/2-aware) load balancer or proxy that balances individual requests/streams;
**client-side load balancing** (the client resolves all backends and spreads calls across connections
to each); or periodically recycle connections (e.g. a maximum connection age) so they rebalance.

**25.** Solves: **over-fetching** (client asks for exactly the fields it needs, saving bandwidth on
mobile); **under-fetching** (one query replaces several sequential REST round trips, which matter a lot
at mobile RTTs); and front-ends can evolve their data needs without new endpoints. Gives up:
(1) **HTTP caching**, since queries are `POST`s to one URL → client-side normalized caches, persisted
queries sent as `GET` with a hash; (2) **predictable server cost**, since clients can send very
expensive queries → depth limits, cost analysis before execution, cost-based rate limiting, allow-listed
persisted queries; (3) **HTTP-level error semantics**, since responses are `200` with an `errors` array
→ GraphQL-aware monitoring; also the N+1 resolver problem (→ DataLoader batching) and per-field
authorization.

**26.** The **N+1 problem**: 1 query for the posts, then one query per post for its author, because
each `author` resolver runs independently. The fix is **batching**, typically the DataLoader pattern:
resolvers request authors by ID through a loader, which collects all IDs requested in the same
execution tick and issues one `SELECT ... WHERE id IN (...)`, then hands each resolver its result
(usually with per-request caching so the same author isn't fetched twice).

## Server push

**27.** 1,000,000 / 10 = **100,000 requests per second**, almost all returning nothing, each paying for
headers, TLS record processing, auth, and maybe a database check. With long polling, the request rate
drops to roughly the **rate of actual events** (plus one re-request per client per timeout period,
e.g. 1,000,000 / 30 s ≈ 33,000 req/s at most if nothing ever happens). But the server now **holds up to
1 million open requests at once**, so it must handle huge numbers of idle connections cheaply (an
event-driven, non-blocking server rather than one thread per request).

**28.** After the server sends a response, there's a window (network transit plus the client sending a
new request) in which no request is waiting. Events that happen in that window have no open request to
be delivered on. Prevention: the client sends a **cursor**, the ID of the last event it received, with
each request, and the server keeps recent events so it can return everything after that cursor. SSE
builds this in: each event can carry an `id:`, and on reconnect the browser automatically sends a
`Last-Event-ID` header so the server can resume from there.

**29.** **SSE.** Data flows almost entirely server → client. Occasional filter changes can be ordinary
HTTP requests (or reconnecting the stream with new parameters). SSE gives automatic reconnection with
resume, works with existing HTTP auth, proxies, load balancers and HTTP/2 multiplexing, and needs no
custom message protocol. WebSockets would add complexity (custom framing protocol, manual reconnection,
non-HTTP infrastructure) without a benefit, since there's no high-rate client → server traffic.

**30.** To protect **intermediaries**, not the endpoints. Without masking, a malicious page could send
WebSocket payloads crafted to look like HTTP requests and responses. Some older transparent proxies that
didn't understand WebSockets would parse those bytes as HTTP and could be tricked into caching
attacker-chosen content for a real URL (cache poisoning). Masking each client frame with a random key
makes the bytes on the wire unpredictable to the page's script, so it can't craft them.

**31.** All 200,000 clients lose their connections at the same moment and immediately try to reconnect,
landing on the remaining servers. This **reconnection storm** brings a huge burst of TCP and TLS
handshakes, authentication and state re-sync (each client fetches missed messages), on top of the extra
200,000 connections. It can overload the remaining servers, which then fail too, making things worse
(a cascading failure). Protections: (1) clients reconnect with **exponential backoff and random jitter**
so reconnections spread out over time; (2) **spare capacity** and load shedding / admission control on
servers so they reject excess handshakes rather than collapse (also: making resync cheap, e.g. resume
from a cursor rather than a full reload).

**32.** (1) A **registry of which server holds which user's connection** (a presence / session store,
e.g. user → server mapping in a shared store), and (2) a **messaging backbone** between servers: server
3 publishes the message, and server 14 receives it (via pub/sub on a per-user or per-server channel,
or a direct server-to-server call) and writes it to B's socket. It also needs a plan for when B is
offline: store the message durably so B gets it on reconnect (with a cursor), and possibly send a push
notification.

## Sync vs async

**33.** 0.999^6 ≈ **0.994**, i.e. 99.4%, roughly 52 hours of downtime a year compared with 8.8 hours for
each service on its own. And that's best case, assuming independent failures and nothing else in the
path. It argues for **shortening the synchronous path**: only call what's essential to give the user an
answer, make other work asynchronous, and add fallbacks / graceful degradation for non-critical
dependencies (e.g. show checkout without recommendations).

**34.** D's calls to E take longer, so each of D's request threads/connections is held longer. By
Little's Law (Day 1), with the same arrival rate, D now has many more requests in flight; its thread
pool and connection pool to E fill up, and new requests to D queue or fail. That makes D slow, so C's
calls to D now hold C's resources longer, and C's pools fill up. The same happens at B, then A. The
user-facing service becomes slow or unavailable, though only E has a problem. Timeouts, circuit
breakers (fail fast when E is unhealthy) and bulkheads (separate resource pools per dependency) stop
this spread (Day 26).

**35.** Benefits: **temporal decoupling** (the consumer can be down; messages wait), **load levelling**
(spikes are absorbed and processed at the consumer's pace), **independent scaling** of consumers, and
easy **fan-out** to multiple consumers without the producer knowing them. Costs: **no immediate
result** for the caller, **eventual consistency** between parts of the system, **duplicate delivery**
and ordering issues (consumers must be idempotent), **harder debugging and observability**, and **a
broker to operate** that must itself be reliable.

**36.** `POST /reports` (with an idempotency key) validates the request, enqueues the job, and returns
**`202 Accepted`** immediately with a `Location: /reports/{id}` header. The client then checks
`GET /reports/{id}`, which returns `{"status": "queued" | "running" | "done" | "failed"}`, and a link to
the result when done. Instead of polling, the server can notify via SSE, WebSockets, push
notification, email, or a webhook. Workers consume jobs from the queue at their own pace.

**37.** (a) **Synchronous**: the purchase can't proceed without the answer, and it must be accurate at
that moment. (b) **Asynchronous**: the order is complete without it; the email service may be slow or
down; a delay of seconds is fine; retries are easy. (c) **Asynchronous**: a search index that's a few
seconds behind is acceptable, the update shouldn't make price changes fail, and indexing can be
batched. (d) **Synchronous**: the price shown and charged depends on it, so the user needs the answer
before confirming.
