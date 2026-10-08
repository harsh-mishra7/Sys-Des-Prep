# Day 3: Application Protocols & Communication Styles

Yesterday covered how bytes move: IP delivers packets, TCP turns them into a reliable stream, TLS
secures it. But a byte stream has no meaning of its own. Two programs still have to agree on **what the
bytes say**: where one message ends and the next begins, what a request is asking for, what counts as
success, and what's safe to retry. That agreement is the application protocol.

Today has two halves. The first is **HTTP itself**: why it went through three major versions, and what
each one fixed. The second is about **styles of communication** built on top: REST, RPC/gRPC, GraphQL,
the four ways a server can push data to a client, and the bigger choice between synchronous and
asynchronous communication between services.

By the end of today you should be able to:

1. Explain what limited HTTP/1.1, what HTTP/2 changed, and what problem HTTP/2 couldn't fix.
2. Explain why HTTP/3 runs on QUIC over UDP, and what that buys (and costs).
3. Define safe and idempotent methods and explain why the distinction matters for retries.
4. Read status codes as a contract: who is at fault, and whether a retry could help.
5. Explain how gRPC and Protocol Buffers work, including how schemas evolve without breaking clients.
6. Explain what GraphQL solves and what it trades away.
7. Describe short polling, long polling, SSE and WebSockets at the mechanism level, and choose between
   them.
8. Explain what synchronous calls cost a system (coupling, availability, latency) and what asynchronous
   communication costs in return.

---

## 1. What an application protocol must decide

Every application protocol, from HTTP to a database's wire protocol, answers the same questions:

| Question | Why it's needed | HTTP/1.1's answer |
|---|---|---|
| **Framing**: where does a message end? | TCP is a byte stream with no boundaries (Day 2) | Headers end at a blank line; body length from `Content-Length` or chunked encoding |
| **Semantics**: what is being asked? | The server must know what to do | Method + path + headers |
| **Outcome**: what happened? | The client must know whether to retry, fix, or give up | Status code |
| **Multiplexing**: can several conversations share a connection? | Connections are expensive (Day 2) | Not really (one request at a time) |
| **Metadata**: auth, content type, caching hints | Cross-cutting concerns | Headers |

Most of the history of HTTP is the history of the **multiplexing** row. Most of REST is about the
**semantics** and **outcome** rows.

---

## 2. HTTP/1.1: simple, text-based, one at a time

An HTTP/1.1 request is plain text:

```
GET /users/42 HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer eyJhbGciOi...
                                     ← blank line ends the headers
```

and so is the response:

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 27
Cache-Control: max-age=60

{"id":42,"name":"Ada Lov."}
```

Being text made HTTP easy to debug and easy to implement, which is a big part of why it won. HTTP/1.1
(1997) added two things over HTTP/1.0 that still matter:

- **Persistent connections (keep-alive) by default.** One TCP + TLS connection can carry many requests
  in sequence, avoiding repeated handshakes (Day 2).
- **The `Host` header**, so many websites can share one IP address (virtual hosting).

### The core limitation: one outstanding request per connection

On one HTTP/1.1 connection, the client sends a request and **must wait for the full response** before
the next request can use the connection. If a page needs 80 resources, they queue up.

HTTP/1.1 tried to fix this with **pipelining**: send several requests back-to-back without waiting.
But responses must still come back **in the order the requests were sent**, because there's nothing in
the protocol to say which response belongs to which request. So if the first response is slow (a big
image, a slow database query), every response behind it waits, even if they're ready. That's
**head-of-line blocking at the HTTP layer**. Combined with buggy proxies, it was bad enough that
browsers turned pipelining off.

### The workarounds, and what they cost

Since one connection could only do one thing at a time, everyone opened more connections:

- Browsers open **about 6 connections per origin** in parallel.
- Sites used **domain sharding** (`img1.example.com`, `img2.example.com`, ...) to get 6 connections per
  hostname, multiplied.
- Developers **concatenated** JS/CSS files and combined images into **sprites** to cut the number of
  requests.

All of these are hacks around the protocol. More connections mean more TCP and TLS handshakes, more
slow-start ramps (each connection starts cold), more server memory, and connections competing with
each other for bandwidth instead of cooperating.

### Header overhead

Headers are repeated in full, as text, on every request. Cookies, user agents and auth tokens can add
up to 500 bytes to 2 KB per request, often more than the body of a small API call. On a mobile uplink,
that adds up.

---

## 3. HTTP/2: multiplexing over one connection

HTTP/2 (2015) kept HTTP's **semantics** exactly the same (methods, status codes, headers, URLs) and
replaced the **wire format**. Your application code barely notices; the transport underneath is
completely different.

### Binary framing

Instead of text, every message is split into **frames**, each with a small fixed header:

```
┌──────────────────────────────────────────────────┐
│ Length (24 bits) │ Type (8) │ Flags (8)           │
│ Stream ID (31 bits)                               │
├──────────────────────────────────────────────────┤
│ Payload (HEADERS, DATA, SETTINGS, WINDOW_UPDATE…) │
└──────────────────────────────────────────────────┘
```

Binary framing is faster and less ambiguous to parse than text, but the important field is the
**stream ID**.

### Streams and multiplexing

A **stream** is one request/response exchange. Every frame says which stream it belongs to, so frames
from many streams can be **interleaved** on one connection and reassembled at the other end:

```
HTTP/1.1, one connection:     [req A ──── resp A][req B ─ resp B][req C ── resp C]

HTTP/2, one connection:       A1 B1 C1 A2 C2 B2 A3 C3 ...   (frames from streams A, B, C interleaved)
```

This is the big win. A slow response on stream A no longer blocks stream B; B's frames just flow
between A's. Consequences:

- **One connection per origin** is enough. One handshake, one slow-start ramp, and a congestion window
  that grows large because all traffic shares it.
- Domain sharding, concatenation and sprites become unnecessary, and sometimes counterproductive.
- Hundreds of concurrent requests can be in flight (the server advertises a limit,
  `SETTINGS_MAX_CONCURRENT_STREAMS`, often 100 to 250).

HTTP/2 also has **per-stream flow control** (like TCP's receive window, Day 2, but per stream), so one
large download can't use up the receiver's buffers and starve the other streams.

### HPACK header compression

HTTP/2 compresses headers with **HPACK**:

- A **static table** of common headers (`:method: GET`, `:status: 200`, ...) referenced by index.
- A **dynamic table** that both sides build up as the connection is used. The second time a client
  sends the same `Authorization` header, it sends a small index instead of the full value.
- **Huffman coding** for literal strings.

Repeated headers shrink to a few bytes. (HPACK was designed specifically to avoid the compression
side-channel attacks, like CRIME, that hit earlier attempts to gzip headers.)

### Server push: an idea that didn't work out

HTTP/2 let a server send a resource the client hadn't asked for yet ("you requested `index.html`;
you'll need `app.css`, here it is"). In practice servers often pushed things the client already had
cached, wasting bandwidth, and it was hard to get right. Chrome removed support in 2022. The lesson:
the server rarely knows what the client's cache contains. Its replacement is the much simpler
`103 Early Hints` response, which just *tells* the client what to fetch. (Don't confuse HTTP/2 server
push with the general idea of servers pushing events to clients, covered in section 8.)

### The problem HTTP/2 couldn't fix: TCP head-of-line blocking

HTTP/2 removed head-of-line blocking at the *HTTP* layer, but it all still runs over **one TCP
connection**, and TCP delivers bytes strictly in order (Day 2). If one packet is lost:

```
Packets on the wire:  [A1][B1][C1][A2][B2]...
                             ✗ lost

TCP receiver has A1, C1, A2, B2 in its buffer, but cannot hand ANY of them to HTTP/2
until B1 is retransmitted. Streams A and C are stalled by a loss that only affected B.
```

TCP has no idea there are independent streams inside the byte stream. So with HTTP/2:

- On a clean network (inside a data centre), it's a clear win.
- On a lossy network (mobile, congested Wi-Fi), **one lost packet stalls every stream**. With 1 to 2%
  packet loss, HTTP/1.1 with 6 independent connections can actually outperform HTTP/2 on one
  connection, because a loss on one of the 6 only stalls that one.

You can't fix this inside TCP; in-order delivery is TCP's whole contract, and TCP lives in operating
system kernels and middleboxes that take decades to change. Fixing it meant leaving TCP.

### Negotiation

How does a client know a server speaks HTTP/2? During the TLS handshake, using **ALPN**
(Application-Layer Protocol Negotiation): the client lists `h2, http/1.1`, the server picks one. No
extra round trip. (In practice, HTTP/2 is only used over TLS.)

---

## 4. HTTP/3 and QUIC: moving streams into the transport

**QUIC** is a new transport protocol, originally built at Google and standardized in 2021. **HTTP/3** is
HTTP mapped onto QUIC. QUIC runs over **UDP**, and this is exactly the "UDP as a building block" case
from Day 2: UDP provides nothing except ports and a checksum, so QUIC can implement its own
reliability, ordering and congestion control in user space, designed the way HTTP needs.

```
   HTTP/1.1          HTTP/2             HTTP/3
┌───────────┐   ┌───────────┐     ┌───────────┐
│   HTTP    │   │  HTTP/2   │     │  HTTP/3   │
├───────────┤   ├───────────┤     ├───────────┤
│   TLS     │   │   TLS     │     │   QUIC    │  ← streams, reliability,
├───────────┤   ├───────────┤     │ (incl.TLS)│    congestion control,
│   TCP     │   │   TCP     │     ├───────────┤    TLS 1.3 built in
├───────────┤   ├───────────┤     │   UDP     │
│   IP      │   │   IP      │     ├───────────┤
└───────────┘   └───────────┘     │   IP      │
                                  └───────────┘
```

### What QUIC fixes

**1. Independent streams in the transport.** QUIC knows about streams. Each stream is ordered
internally, but streams are independent of each other. A lost packet carrying stream B's data stalls
**only stream B**; A and C carry on. This removes the last head-of-line blocking problem.

**2. Fewer handshake round trips.** TCP + TLS 1.3 needs 1 RTT for TCP, then 1 RTT for TLS, before the
first request. QUIC combines the transport and crypto handshakes:

| Setup | Round trips before the first request |
|---|---|
| TCP + TLS 1.2 | 3 |
| TCP + TLS 1.3 | 2 |
| QUIC, new connection | 1 |
| QUIC, resumed connection (0-RTT) | 0 |

0-RTT has the same replay caveat as TLS 1.3 0-RTT (Day 2): early data can be replayed by an attacker,
so it should only carry requests that are safe to repeat, which in practice means idempotent ones.
(Section 5 explains what that means.)

**3. Connection migration.** A TCP connection is identified by its 4-tuple (Day 2). When your phone
switches from Wi-Fi to cellular, its IP changes, the 4-tuple changes, and every connection dies. Each
must be re-established with new handshakes and a cold congestion window. QUIC identifies connections
by a **connection ID** chosen by the endpoints, so the connection survives the address change.

**4. Encrypted transport headers.** In TCP, sequence numbers, flags and options are visible to every
middlebox on the path, and middleboxes started depending on them. That's why TCP is so hard to
change: any new option might be dropped by some firewall. This is called **ossification**. QUIC
encrypts almost all of its own headers, so middleboxes can't depend on them, and QUIC can keep
evolving.

**5. Evolves in user space.** QUIC ships inside applications and libraries (browsers, server
software), not the OS kernel. New congestion control or loss recovery can roll out with an app update.

HTTP/3 uses **QPACK** instead of HPACK. HPACK assumed in-order delivery of header blocks (which TCP
gave it); QPACK is redesigned so out-of-order streams don't block on each other's header table
updates.

### What QUIC costs

- **CPU.** TCP processing has decades of kernel and network-card optimisation (segmentation offload,
  checksum offload). QUIC does encryption and packet handling per packet in user space, and has
  historically cost noticeably more CPU per byte. This is improving but still real at large scale.
- **UDP gets blocked or throttled.** Some corporate networks and firewalls block UDP other than DNS.
  Clients must be able to fall back to HTTP/2 over TCP.
- **Discovery.** A client can't know in advance that a server supports HTTP/3. Usually the first
  connection uses HTTP/2, and the server advertises HTTP/3 with an `Alt-Svc` response header (or a DNS
  `HTTPS` record). Later connections try QUIC, typically racing it against TCP.
- **Less visibility for operators.** Encrypted headers mean network tools can't easily see loss,
  retransmissions or RTT the way they can for TCP.

### Where each version wins

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Requests per connection at once | 1 | Many (multiplexed) | Many (multiplexed) |
| HOL blocking | HTTP layer and TCP | TCP only | Only within a stream |
| Header compression | None | HPACK | QPACK |
| Handshake (new, TLS 1.3) | 2 RTT | 2 RTT | 1 RTT |
| Survives network change | No | No | Yes (connection IDs) |
| Best fit | Simple clients, debugging | Most traffic, especially inside data centres | Mobile and lossy, high-latency networks |

Inside a data centre, with sub-millisecond RTTs and almost no loss, HTTP/3's advantages mostly
disappear, and HTTP/2 (or gRPC, which uses it) is the common choice for service-to-service traffic.
HTTP/3's gains are largest at the **edge**, between users on unreliable networks and the nearest
server.

---

## 5. REST: designing around resources

HTTP is a protocol; **REST** (Representational State Transfer, Roy Fielding's 2000 dissertation) is a
**style** of using it. Most "REST APIs" in practice follow only part of Fielding's definition, but the
core ideas are what matter:

- **Resources** are the nouns, each named by a URL: `/users/42`, `/users/42/orders`.
- **Representations**: the client never touches the resource itself, only a representation of it
  (JSON, HTML, ...), negotiated with `Accept` / `Content-Type`.
- **A uniform interface**: the same small set of methods applies to every resource, with the same
  meaning everywhere. This is the key point: because `GET` *means* the same thing on every URL, generic
  infrastructure (caches, proxies, retry logic, browsers) can act on requests without understanding
  your application.
- **Stateless requests**: each request carries everything needed to process it (credentials, IDs). The
  server keeps no per-client conversation state between requests. This is what lets any server behind a
  load balancer handle any request (Day 1's stateless components; Day 4).
- **Cacheability**: responses say whether and how long they can be cached (Days 5 and 7).

### Methods, and the two properties that matter

| Method | Meaning | Safe | Idempotent |
|---|---|---|---|
| `GET` | Read a representation | ✅ | ✅ |
| `HEAD` | `GET` without the body | ✅ | ✅ |
| `OPTIONS` | What can I do here? (used by CORS) | ✅ | ✅ |
| `PUT` | Replace the resource with this representation (or create it at this URL) | ❌ | ✅ |
| `DELETE` | Remove the resource | ❌ | ✅ |
| `POST` | Process this: create a child resource, trigger an action | ❌ | ❌ |
| `PATCH` | Apply a partial modification | ❌ | ❌ (not guaranteed) |

**Safe** means the client is not asking for any change of state: the request is read-only *from the
client's point of view*. A server may still log it or update a counter, but the client isn't
responsible for that. Crawlers, prefetchers and caches assume safe methods can be called freely.
(This is why "delete" links that are plain `GET`s have historically been wiped out by crawlers.)

**Idempotent** means making the same request **once or N times has the same effect on the server's
state**. Not the same *response*: the first `DELETE /orders/7` returns `204`, the second returns
`404`, but the order is gone either way, so the state is the same.

### Why idempotency is the property that matters most

Recall from Day 2: a timeout tells you nothing about whether the server processed your request.

```
Client                      Server
  │── POST /payments ────────▶│  processes payment, charges card
  │                           │
  │      ✗ response lost ✗ ◀──│
  │
  timeout... did it happen?
```

The client faces a choice with no safe answer: retry and maybe charge twice, or don't and maybe never
charge. **If the request is idempotent, the client can simply retry.** That's why:

- Proxies, load balancers and HTTP client libraries will automatically retry idempotent methods on
  connection failures, but not `POST`.
- `PUT` is idempotent because it says "the resource should look like *this*", a final state, not a
  change. Setting `balance = 100` twice gives 100. `POST` "add 10" twice gives 20.
- `PATCH` *can* be idempotent (`set email to x`) or not (`append to list`). The protocol can't promise
  either.

### Making POST safe to retry: idempotency keys

For non-idempotent operations like payments, a common pattern is an **idempotency key**: the client
generates a unique ID per *logical* operation and sends it with every attempt:

```
POST /payments
Idempotency-Key: 6f1c2b0e-8d4a-4f7e-9a21-0c9b5e3d7a11
{ "amount": 500, "currency": "USD", ... }
```

The server records each key with its result. If the same key arrives again, it returns the stored
result instead of processing again. Retries become safe. The details (where to store keys, how long,
what if two attempts arrive at the same time) come back in Day 23.

### Status codes as a contract

Status codes tell the client **who is at fault** and **what to do next**. The first digit carries most
of the meaning:

| Class | Meaning | Should the client retry the same request? |
|---|---|---|
| `2xx` | Success | No need |
| `3xx` | Go somewhere else (or use your cached copy) | Follow the redirect |
| `4xx` | **Client's** problem: the request itself is wrong | **No**, not unchanged; it'll fail again |
| `5xx` | **Server's** problem | **Maybe**, if the method is idempotent, with backoff |

The codes worth knowing precisely:

| Code | Meaning | Note |
|---|---|---|
| `200 OK` | Success, with a body | |
| `201 Created` | A new resource was created | `Location` header points to it |
| `202 Accepted` | Received, will be processed **later** | The asynchronous handoff (section 9) |
| `204 No Content` | Success, no body | Common for `DELETE`, `PUT` |
| `301` / `308` | Moved permanently | `308` guarantees the method isn't changed |
| `302` / `307` | Temporary redirect | `307` guarantees the method isn't changed |
| `304 Not Modified` | Your cached copy is still valid | Answer to a conditional `GET` (Day 7) |
| `400 Bad Request` | Malformed request | |
| `401 Unauthorized` | **Not authenticated**: who are you? | Badly named; it means "unauthenticated" |
| `403 Forbidden` | Authenticated, but **not allowed** | |
| `404 Not Found` | No such resource | |
| `409 Conflict` | Conflicts with current state | e.g. duplicate username |
| `412 Precondition Failed` | An `If-Match` / `If-Unmodified-Since` check failed | Optimistic concurrency (below) |
| `422 Unprocessable Content` | Well-formed but semantically invalid | Validation errors |
| `429 Too Many Requests` | Rate limited | Respect `Retry-After` (Day 27) |
| `500 Internal Server Error` | Unhandled server bug | |
| `502 Bad Gateway` | A proxy got an invalid response (or none) from upstream | Upstream crashed or closed the connection |
| `503 Service Unavailable` | Temporarily overloaded or down | Often has `Retry-After`; safe to retry later |
| `504 Gateway Timeout` | A proxy's upstream didn't respond in time | The work **may still have happened** |

Two things often go wrong:

- **Returning `200` with `{"error": ...}` in the body.** Every piece of generic infrastructure (load
  balancer health metrics, retry logic, caches, monitoring) now thinks the request succeeded. Error
  rates look perfect while users fail. A cache might even store the error.
- **Retrying `4xx` errors.** A `400` will be a `400` forever. Retrying wastes capacity. (`408` and `429`
  are the exceptions: they're about timing, not the request's content.)

Also note what `502`, `503` and `504` tell you: they usually come from a **proxy or load balancer**
reporting on the server behind it. A `504` especially does not mean "nothing happened"; the backend may
have finished the work after the proxy gave up waiting.

### Conditional requests: optimistic concurrency over HTTP

Two clients read the same document, both edit it, both `PUT` it back. The second write silently
overwrites the first: a **lost update** (Day 11). HTTP has a built-in fix:

```
GET /docs/9          →  200 OK, ETag: "v17"
PUT /docs/9
If-Match: "v17"      →  200 OK if the doc is still v17
                     →  412 Precondition Failed if someone else changed it first
```

The `ETag` is a version identifier. `If-Match` turns the `PUT` into a **compare-and-set**: "only write
if nobody else has written since I read". This is optimistic concurrency control, which comes back in
detail on Day 12. The same `ETag` with `If-None-Match` on a `GET` gives cache revalidation (`304`),
covered on Day 7.

### Pagination: offset vs cursor

Collection endpoints must paginate, and the choice has real consequences:

- **Offset** (`?offset=1000&limit=50`): simple, allows jumping to page N. But the database must still
  walk past 1,000 rows to skip them, so deep pages get slower. And if rows are inserted or deleted
  between page requests, items get **skipped or duplicated**.
- **Cursor / keyset** (`?after=<id of last item>&limit=50`): the server queries
  `WHERE id > :after ORDER BY id LIMIT 50`, which an index (Day 9) serves directly, at the same cost no
  matter how deep you go. It's stable under inserts. It can't jump to an arbitrary page, which is
  rarely needed for APIs.

---

## 6. RPC and gRPC: calling functions across the network

REST models the API as **resources you act on**. **RPC** (Remote Procedure Call) models it as
**functions you call**: `getUser(42)`, `chargeCard(order, amount)`. The idea is decades old: make a
remote call look like a local function call. A generated **stub** on the client turns the call into a
network message; a **skeleton** on the server turns it back into a call.

### The danger in "it looks like a local call"

A remote call is fundamentally different from a local one, and hiding that is the classic RPC
mistake:

| Local call | Remote call |
|---|---|
| Takes nanoseconds | Takes 0.1 ms to hundreds of ms, highly variable |
| Either returns or throws | Can **time out**, leaving you not knowing whether it ran |
| Never partially fails | The other side can crash halfway through |
| Pass a pointer, no copying | Everything must be serialized and copied |
| Always the same version of the code | The other side may be a different version |

Good RPC frameworks don't hide these differences; they make them explicit: deadlines, error codes
that distinguish "unavailable" from "invalid", and schemas designed for version skew. gRPC is built
around exactly these.

### Protocol Buffers: a schema and a compact binary format

gRPC's default serialization is **Protocol Buffers** (protobuf). You define messages and services in a
`.proto` file:

```proto
syntax = "proto3";

message User {
  int64  id     = 1;
  string name   = 2;
  string email  = 3;
}

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc WatchUsers (WatchRequest) returns (stream UserEvent);
}
```

A compiler generates client and server code for many languages from this file. The `.proto` is the
**contract** between teams.

The numbers (`= 1`, `= 2`) are the important part. On the wire, protobuf doesn't send field
*names*; it sends each field as `(field number, wire type, value)`:

```
JSON:      {"id":42,"name":"Ada"}          ~22 bytes, field names repeated every message
Protobuf:  08 2A 12 03 41 64 61            7 bytes
           │  │  │  │  └─ "Ada"
           │  │  │  └─ length 3
           │  │  └─ field 2, length-delimited
           │  └─ 42 (varint)
           └─ field 1, varint
```

Integers use **varints** (small numbers take fewer bytes). The result is typically several times
smaller than JSON and much faster to parse, because there's no text to scan and no field names to
match.

### Schema evolution: why field numbers are forever

Services are never upgraded all at once. During a rollout, old clients talk to new servers and new
clients talk to old servers. Protobuf handles this because fields are identified by number:

- **Adding a field** is safe. Old readers see an unknown field number and skip it (the wire type tells
  them how many bytes to skip). New readers of old messages see the field missing and use the default.
- **Removing a field** is safe *if* you never reuse its number. Mark it `reserved 3;` so nobody does.
- **Renaming a field** is safe on the wire (names aren't sent), though it breaks generated code.
- **Reusing a field number or changing a field's type is not safe.** An old client sending field 3 as
  a string to a new server that thinks field 3 is an int will produce garbage or errors.

This property, **old and new code can read each other's data**, is called backward and forward
compatibility. It comes back on Day 8 and with message schemas in Week 4.

### gRPC: RPC on top of HTTP/2

gRPC uses HTTP/2 as its transport. Each call is an HTTP/2 stream; the method is the path
(`/UserService/GetUser`); messages are length-prefixed protobuf in DATA frames. From HTTP/2 it gets
multiplexing, flow control, and header compression for free. It supports four call types:

| Type | Shape | Example |
|---|---|---|
| Unary | one request → one response | `GetUser` |
| Server streaming | one request → stream of responses | Subscribe to price updates |
| Client streaming | stream of requests → one response | Upload sensor readings, get a summary |
| Bidirectional streaming | streams both ways, independently | Chat, real-time collaboration |

gRPC also builds in things that HTTP+JSON APIs usually bolt on:

- **Deadlines.** A client sets an absolute deadline ("this must finish within 200 ms"). It travels with
  the call (`grpc-timeout` header), and a server that calls further services passes the *remaining*
  time along. If the deadline has passed, every service in the chain can stop working on a request
  nobody is waiting for any more. (More on timeouts on Day 26.)
- **Status codes designed for distributed systems**: `UNAVAILABLE` (safe to retry), `DEADLINE_EXCEEDED`
  (outcome unknown), `INVALID_ARGUMENT` (don't retry), `RESOURCE_EXHAUSTED`, `ALREADY_EXISTS`, ...
- **Metadata** (headers) for auth tokens and trace IDs.

### gRPC's trade-offs

- **Not human-readable.** You can't `curl` it and read the result; you need tooling (and the `.proto`
  file) to inspect traffic.
- **Browsers can't speak it directly.** Browser JavaScript can't control HTTP/2 framing and trailers
  the way gRPC needs. A translating proxy (gRPC-Web) is required. This is why gRPC is mostly used
  **between services**, with REST or GraphQL facing browsers.
- **Load balancing needs care.** gRPC keeps one long-lived HTTP/2 connection and multiplexes everything
  over it. A **layer 4** load balancer (Day 4) balances *connections*, not requests, so it pins every
  request from a client to one backend for the life of the connection. Adding backends doesn't help
  existing clients. You need **layer 7** (request-aware) load balancing, or client-side balancing
  across several connections. Day 2's last question had the same problem with connection pools; HTTP/2
  makes it worse because one connection carries everything.
- **Tighter coupling.** Both sides compile against shared `.proto` files, which must be distributed and
  versioned.

---

## 7. GraphQL: let the client describe the shape

REST endpoints return fixed shapes chosen by the server. That leads to two opposite problems,
especially for UIs:

- **Over-fetching**: `GET /users/42` returns 40 fields; the screen needs 3.
- **Under-fetching**: showing a user with their last 5 orders and each order's items takes
  `GET /users/42`, then `GET /users/42/orders`, then one `GET` per order. On a mobile network with
  150 ms RTT, five sequential round trips is most of a second.

**GraphQL** (Facebook, 2015) exposes a **typed schema** of the whole data graph through **one
endpoint**, and the client sends a query describing exactly the data it wants:

```graphql
query {
  user(id: 42) {
    name
    orders(last: 5) {
      total
      items { name price }
    }
  }
}
```

One round trip, exactly the requested fields, in exactly that shape. The schema is introspectable,
so tools can autocomplete and validate queries. Front-end teams can change what a screen shows without
waiting for a new backend endpoint.

On the server, each field is backed by a **resolver** function that knows how to fetch it.

### What GraphQL trades away

**1. HTTP caching.** Queries are usually `POST`ed to a single URL like `/graphql`. To an HTTP cache or
CDN, every request looks the same and isn't cacheable (`POST` isn't). The uniform interface that let
generic infrastructure understand REST requests is gone. Caching moves into the client library (a
normalized cache keyed by object ID), or you use **persisted queries**: register queries ahead of time,
then send a hash via `GET`, which makes them cacheable again.

**2. Query cost becomes the client's choice.** A client can write a query that's cheap to send and
very expensive to run:

```graphql
{ users(first: 1000) { friends(first: 1000) { friends(first: 1000) { name } } } }
```

That's up to a billion resolutions in one request. Servers need **depth limits**, **complexity / cost
analysis** before execution, timeouts, and rate limits based on cost rather than request count.
Allowing only persisted queries (an allow-list) is the strongest defence.

**3. The N+1 problem.** Resolvers run per field. Fetching 50 orders and then the `customer` of each
naively runs 1 query for the orders plus 50 queries for customers. The standard fix is **batching**
(the *DataLoader* pattern): collect all the customer IDs requested within one execution tick and fetch
them in a single `WHERE id IN (...)` query.

**4. Errors don't use HTTP status codes.** A GraphQL response is usually `200 OK` even if part of it
failed, with an `errors` array alongside partial `data`. Partial success is genuinely useful, but it
means generic monitoring sees a 100% success rate. You need GraphQL-aware observability.

**5. Authorization per field.** Since any client can reach any part of the graph from any entry point,
authorization must be enforced at the field / object level, not per endpoint.

GraphQL works best as an **aggregation layer** in front of many services, serving varied clients
(web, mobile, partners) whose data needs differ and change often. For simple CRUD, or for
service-to-service calls, its costs usually aren't worth it.

### REST vs gRPC vs GraphQL

| | REST (HTTP + JSON) | gRPC | GraphQL |
|---|---|---|---|
| Model | Resources + uniform methods | Functions on services | A typed graph, client-chosen queries |
| Contract | Optional (OpenAPI) | Required (`.proto`) | Required (schema) |
| Payload | JSON text, usually | Protobuf binary | JSON text |
| Transport | HTTP/1.1, 2 or 3 | HTTP/2 | HTTP, usually one `POST` endpoint |
| HTTP caching | Excellent | None | Poor (without persisted queries) |
| Browser support | Native | Needs a proxy (gRPC-Web) | Native |
| Streaming | Not natively (see SSE) | Built in, both directions | Subscriptions (usually over WebSockets) |
| Typical home | Public APIs, simple CRUD | Internal service-to-service | Front-end aggregation layer |

None of these is "best". A common real architecture uses all three: gRPC between internal services,
a GraphQL or REST gateway in front of them, and REST for public/partner APIs.

---

## 8. Getting data from server to client

HTTP is **client-initiated**: the server can only answer requests. But many features need the server
to tell the client something *when it happens*: a new chat message, a price change, a job finishing.
There are four standard ways to do this, in rough order of sophistication.

### Short polling

The client asks repeatedly on a timer:

```
Client                         Server
  │── any updates? ─────────────▶│
  │◀──────────────── no ─────────│
  │      (wait 5 s)
  │── any updates? ─────────────▶│
  │◀──────────────── no ─────────│
  │      (wait 5 s)
  │── any updates? ─────────────▶│
  │◀──────── yes, here ──────────│
```

- **Latency**: an event waits up to one full interval (on average half) before the client sees it.
- **Cost**: most requests return nothing. 1 million clients polling every 5 s is **200,000
  requests/second** of mostly empty answers, each with full headers, auth checks and maybe a database
  query.
- **Upside**: dead simple; works everywhere; stateless on the server; no long-lived connections.

Shortening the interval improves latency only by multiplying cost. Short polling is fine when updates
are infrequent and a delay of seconds to minutes is acceptable (checking an export job's status).

### Long polling

The client asks, and the server **doesn't answer until it has something** (or until a timeout, say
30 s, after which it returns empty). As soon as the client gets a response, it immediately asks again:

```
Client                         Server
  │── any updates? ─────────────▶│
  │                              │  (holds the request open...)
  │                              │  ...event happens
  │◀──────── yes, here ──────────│
  │── any updates since X? ─────▶│  (immediately re-asks)
  │                              │  (holds...)
```

- **Latency**: near-instant for the first event.
- **Cost**: far fewer empty responses; but every waiting client ties up an open request on the server,
  so the server must handle many idle connections cheaply (event-driven servers, not a thread per
  request).
- **The gap**: between a response and the next request, events can occur. The client must send a
  **cursor** ("updates since event 1042") so the server can catch it up, otherwise those events are
  lost. Each re-request also pays full HTTP header overhead.
- **Upside**: plain HTTP; works through almost any proxy and firewall.

Long polling was the standard technique before WebSockets and SSE existed, and it's still a robust
fallback.

### Server-Sent Events (SSE)

The client makes one ordinary HTTP request; the server responds with `Content-Type:
text/event-stream` and **never finishes the response**, writing events into it as they happen:

```
GET /events HTTP/1.1
Accept: text/event-stream

HTTP/1.1 200 OK
Content-Type: text/event-stream

id: 1041
data: {"price": 101.5}

id: 1042
data: {"price": 101.7}

...  (connection stays open)
```

Each event is a few `field: value` lines followed by a blank line. Browsers have a built-in
`EventSource` API that handles it, including:

- **Automatic reconnection.** If the connection drops, the browser reconnects and sends a
  `Last-Event-ID: 1042` header, so the server can resume from where the client left off. The cursor
  that long polling had to implement by hand is part of the protocol.

Characteristics:

- **One direction only**: server to client. The client sends anything else as normal HTTP requests.
- **Text only** (binary data must be encoded, e.g. base64).
- **It's just HTTP**: works with existing auth, proxies, load balancers and HTTP/2. Over HTTP/1.1 each
  SSE stream uses up one of the browser's ~6 connections per origin; over HTTP/2 it's just one stream
  on the shared connection, so the limit doesn't matter.

SSE fits most "server notifies client" features: live scores, notifications, dashboards, progress
updates, and streaming generated text token by token.

### WebSockets

A WebSocket is a **full-duplex, message-oriented** channel: either side can send a message at any time.
It starts as an HTTP request and then **leaves HTTP behind**:

```
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

── from here on, the TCP connection carries WebSocket frames, not HTTP ──
```

The `Sec-WebSocket-Key` / `Accept` exchange proves the server really understands WebSockets (and
isn't some HTTP server tricked into accepting). After that, both sides send **frames** with a small
header (2 to 14 bytes) carrying text or binary messages, plus control frames: **ping/pong** for
keep-alive and **close**. Frames from client to server are **masked** (XORed with a random key) so
that attacker-controlled bytes can't look like valid HTTP to a confused proxy along the way, which
could otherwise be tricked into caching them.

Characteristics:

- **Lowest overhead per message** and true two-way communication.
- **Not HTTP after the upgrade**: no status codes, no caching, no standard semantics. You design your
  own message protocol on top (message types, acknowledgements, request IDs).
- **No built-in reconnection or resume.** If the connection drops, the client must reconnect and
  re-sync, with your own cursor logic.
- **Proxies and middleboxes** sometimes kill idle long-lived connections; ping frames keep them alive.

WebSockets fit genuinely **interactive, bidirectional, high-frequency** traffic: chat, multiplayer
games, collaborative editing, trading terminals.

### What every persistent-connection approach costs

Long polling, SSE and WebSockets all mean **long-lived connections held open on the server**. This
changes the shape of the system:

- **Servers become stateful.** Each server holds a set of live connections. A server can't be removed
  without dropping its clients. Deploys must **drain** connections gradually (Day 4), otherwise every
  client reconnects at once.
- **Fan-out needs a messaging layer.** User A's message arrives at server 3, but user B is connected to
  server 7. Servers need a **pub/sub** backbone (e.g. Redis pub/sub or a message broker, Week 4) plus a
  way to know which server holds which user.
- **Reconnection storms.** If a server holding 100,000 connections dies, 100,000 clients reconnect at
  once, possibly overwhelming the others. Clients must reconnect with **exponential backoff and
  jitter** (Day 26).
- **Load balancing is uneven.** Load balancers distribute *new* connections. A server that just started
  gets no share of connections that were made hours ago. Long-lived connections rebalance slowly.
- **Capacity is measured in concurrent connections**, not requests per second. Each idle connection
  still costs memory (kernel socket buffers, TLS state, application state), typically a few KB to tens
  of KB.

### Choosing

| | Short polling | Long polling | SSE | WebSockets |
|---|---|---|---|---|
| Direction | Client pulls | Server → client (simulated) | Server → client | Both ways |
| Latency | Up to the interval | Near real-time | Real-time | Real-time |
| Overhead per update | Full request, mostly empty | Full request per event | Small | Smallest |
| Plain HTTP? | Yes | Yes | Yes | Only for the handshake |
| Reconnect / resume | N/A | Manual cursor | Built in (`Last-Event-ID`) | Manual |
| Server holds connections | No | Yes | Yes | Yes |
| Good for | Rare updates, simplicity | Fallback, restrictive networks | Notifications, feeds, streaming output | Chat, games, collaboration |

A good default question: **does the client need to send a high rate of messages back over the same
channel?** If not, SSE is usually simpler than WebSockets and keeps all of HTTP's infrastructure on
your side.

A related pattern between servers is the **webhook**: instead of a partner polling your API, they give
you a URL and you `POST` to it when something happens. It's server push between systems. Webhooks need
retries (the receiver may be down), so receivers must handle duplicate deliveries idempotently.

---

## 9. Synchronous vs asynchronous communication between services

So far, everything has been about *how* two programs talk. The deeper design choice is **whether the
caller waits**.

### Synchronous: request and wait

Service A calls service B and **blocks until B answers** (REST, gRPC unary). It's the natural model:
easy to write, easy to reason about, and the caller gets an immediate answer, including errors.

But it creates **temporal coupling**: A can only succeed if B is up and responsive **right now**. In a
chain of synchronous calls, that coupling compounds.

**Availability multiplies.** If a request needs A → B → C → D → E, all synchronous, the request
succeeds only if all five are up:

```
Each service 99.9% available  →  0.999^5 ≈ 99.5% for the request
                                  (8.8 hours/year of downtime → ~44 hours/year)
```

**Latency adds up** along a chain, and fan-out amplifies the tail (Day 1): if a request calls 100
backends in parallel and waits for all of them, and each has a 1% chance of being slow, then
`1 - 0.99^100 ≈ 63%` of requests hit at least one slow backend.

**Failures cascade.** If E slows down, D's threads and connections pile up waiting on E, then C's pile
up waiting on D, all the way back to the user. One slow dependency can exhaust resources in every
service upstream (Day 2's pool exhaustion, generalized). Timeouts, circuit breakers and bulkheads
(Day 26) exist mainly to contain this.

### Asynchronous: hand off and move on

Service A puts a message on a **queue or log** (Week 4) and continues without waiting. Service B
consumes it whenever it can.

```
Synchronous:     A ──request──▶ B        (A waits; B must be up now)
                 A ◀──response── B

Asynchronous:    A ──msg──▶ [ queue ] ──msg──▶ B   (A moves on; B processes when it can)
```

What you gain:

- **Temporal decoupling**: B can be down for a deploy or crash; messages wait in the queue. A keeps
  accepting work.
- **Load levelling**: a spike of 10,000 messages is absorbed by the queue and processed at B's pace,
  instead of overwhelming B.
- **Independent scaling**: add more B consumers to drain a backlog.
- **Easy fan-out**: several different consumers can react to the same event (send email, update
  search index, record analytics) without A knowing about any of them.

What you pay:

- **No immediate answer.** A can't tell the user "done", only "accepted". The result must reach the
  user some other way (polling a status, a push notification, a webhook).
- **Eventual consistency.** For a while, different parts of the system disagree (the order exists, but
  the email hasn't been sent and the search index doesn't show it).
- **Duplicates and ordering.** Most queues deliver **at least once**, so consumers must be idempotent.
  Ordering is often only guaranteed partially (Days 22–23).
- **Harder to debug and observe.** A request's path is spread across queues and time. You need
  tracing that follows messages (Day 28), and you need to watch queue depth and consumer lag.
- **A new component to run**: the broker itself must be highly available and durable.

### The bridge: 202 Accepted

A common pattern combines both styles at the API boundary:

```
Client                     API                       Queue / Workers
  │── POST /reports ───────▶│
  │                         │── enqueue job ─────────────▶│
  │◀── 202 Accepted ────────│
  │    Location: /reports/77
  │                                                       │ ...works...
  │── GET /reports/77 ─────▶│   → {"status": "running"}
  │── GET /reports/77 ─────▶│   → {"status": "done", "url": ...}
```

The client gets a fast synchronous answer ("accepted, here's where to check"), and the slow work
happens asynchronously. The status can be polled, or pushed with SSE, WebSockets or a webhook.

### When to use which

Use **synchronous** when the caller genuinely **needs the answer to continue**: reading data to display,
checking authorization, validating input, checking a balance before spending it.

Use **asynchronous** when the work can **happen after the response**, when it's slow, when it fans out
to several consumers, or when the downstream is less reliable than you'd like to depend on directly:
sending emails, generating reports, updating search indexes, processing uploads, recording analytics.

A useful rule of thumb: **keep the synchronous path short.** On the path from a user's request to its
response, call only what's strictly needed, and push everything else onto queues. Each synchronous
dependency you remove is one less thing that can make the user's request fail or slow.

---

## 10. Putting it together: one action, several styles

A user posts a comment in a mobile app:

1. The app sends `POST /posts/9/comments` with an idempotency key, over **HTTP/3** to the nearest edge
   (fast handshake; survives the phone switching networks mid-request).
2. The API gateway authenticates the request, then calls the comments service over **gRPC** with a
   300 ms deadline (internal, binary, multiplexed over a pooled HTTP/2 connection).
3. The comments service writes the comment to its database and returns **`201 Created`**. That's all
   that happens **synchronously**.
4. It also publishes a `CommentCreated` event to a **message log**. **Asynchronously**, consumers send
   a notification to the post's author, update the search index, and count the comment for analytics.
   If the notification service is down, comments keep working; notifications catch up later.
5. Other users viewing the post receive the new comment over an open **SSE** stream (or WebSocket),
   pushed by a server that subscribed to the same event.
6. If the phone lost the `201` response and retried, the **idempotency key** made sure the comment
   was posted only once.

---

## 11. How it connects

- HTTP/2's **single-connection multiplexing** depends on Day 2's lesson that connections are expensive;
  HTTP/3 exists because of Day 2's **TCP head-of-line blocking**, and runs on **UDP** for the reasons
  Day 2 gave.
- **Idempotency** is the property that makes retries safe. It comes back in resilience (Day 26), message
  delivery (Day 23) and distributed transactions (Day 21).
- **Statelessness** in REST is what lets load balancers (Day 4) send any request to any server;
  persistent connections (WebSockets, SSE) give some of that up.
- **Cacheability** of `GET` is the foundation of HTTP caching and CDNs (Days 5 and 7); **ETags** return
  on Day 7 for revalidation and Day 12 as optimistic concurrency.
- **gRPC on long-lived HTTP/2 connections** forces **L7 load balancing** (Day 4).
- **Protobuf schema evolution** previews data encoding and compatibility (Day 8, Week 4).
- **Synchronous call chains** explain why timeouts, circuit breakers and bulkheads exist (Day 26);
  **asynchronous** communication is all of Week 4.

## Common misconceptions

- **"HTTP/2 removed head-of-line blocking."** It removed it at the HTTP layer. TCP's in-order delivery
  still stalls every stream on a single lost packet. HTTP/3 removes that.
- **"HTTP/3 is always faster."** Its gains are on lossy, high-latency networks and in connection setup.
  Inside a data centre the difference is small, and QUIC costs more CPU.
- **"Idempotent means the response is the same every time."** It means the *server state* ends up the
  same. A second `DELETE` returns `404`, and that's fine.
- **"POST is for creating, PUT is for updating."** `PUT` means "make the resource at this URL equal to
  this" (create or replace; idempotent). `POST` means "process this" (not idempotent). The difference is
  about retry semantics, not create vs update.
- **"Return 200 and put the error in the body."** This hides failures from every piece of generic
  infrastructure: monitoring, retries, load balancer health, caches.
- **"401 means not allowed."** `401` means *not authenticated*; `403` means authenticated but *not
  allowed*.
- **"A 504 means the request didn't happen."** The proxy stopped waiting; the backend may have
  completed the work.
- **"gRPC is just faster REST."** It's a different model (functions vs resources) with different
  trade-offs: no HTTP caching, no direct browser support, and load-balancing pitfalls.
- **"GraphQL is a database query language."** It's an API query language. Each field is a resolver
  calling whatever backend you choose, and how efficiently that happens is your problem (N+1).
- **"Real-time means WebSockets."** For server-to-client updates, SSE is often simpler and keeps HTTP's
  infrastructure. WebSockets earn their cost when the client also sends a lot.
- **"Asynchronous is always more scalable."** It moves complexity into eventual consistency, duplicate
  handling and observability. Use it where the work truly doesn't need to finish before the response.

---

## Further reading (supplements)

- Ilya Grigorik, *High Performance Browser Networking* (free online): chapters on HTTP/1.1, HTTP/2,
  WebSockets and SSE
- Daniel Stenberg, *HTTP/3 explained* (free online): short, clear explanation of QUIC
- MDN Web Docs: HTTP methods, status codes, `EventSource` and WebSockets references
- protobuf.dev: the "Language Guide" and "Encoding" pages
- graphql.org: "Learn" section, including the pages on pagination, caching and performance
- RFC 9110 (HTTP semantics), RFC 9113 (HTTP/2), RFC 9000 (QUIC), RFC 9114 (HTTP/3), RFC 6455
  (WebSockets), if you want the authoritative source for a detail
