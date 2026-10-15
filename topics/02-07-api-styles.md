# 07 — API Styles: REST, RPC, GraphQL

> Part 2 — Networking & Communication · Prerequisites: topics 01, 02, 03, 06

Topics 03 to 06 covered how bytes get from one machine to another: TCP gives a reliable stream, TLS
secures it, and HTTP frames it into requests and responses. HTTP still doesn't say how to shape an API.
Two programs have to agree on what the API *is*: a set of resources to act on, or a set of functions to
call, or a graph to query. They also have to agree on what each operation means, what counts as
success, and above all **what's safe to retry** when the network fails partway through.

This topic covers the three main styles built on top of HTTP: **REST**, **RPC** (and gRPC in
particular), and **GraphQL**. Then it steps back to a larger choice: whether a caller should **wait**
for an answer at all (synchronous communication) or hand the work off and move on (asynchronous
communication).

By the end of this topic you should be able to:

1. Explain REST's core ideas and why a uniform interface lets generic infrastructure work.
2. Define safe and idempotent methods and explain why the distinction matters for retries.
3. Read status codes as a contract: who is at fault, and whether a retry could help.
4. Use idempotency keys, conditional requests (`ETag` / `If-Match`) and cursor pagination, and say what
   each one protects against.
5. Explain how gRPC and Protocol Buffers work, including how schemas evolve without breaking clients.
6. Explain what GraphQL solves for clients and what it trades away on the server.
7. Compare REST, gRPC and GraphQL and say where each fits.
8. Explain what synchronous calls cost a system (coupling, availability, latency, cascading failure) and
   what asynchronous communication costs in return.

---

## 1. What an API style must decide

An API style sits on top of an application protocol (usually HTTP, topic 06) and answers a few
questions every pair of communicating programs must agree on:

| Question | Why it's needed |
|---|---|
| **What is the API made of?** Resources, functions, or a graph? | Determines how operations are named and organized |
| **What does each operation mean?** | The server must know what to do; intermediaries want to know too |
| **What happened?** | The client must know whether to retry, fix its request, or give up |
| **What's safe to repeat?** | Timeouts leave outcomes unknown (topic 03); retries must not cause damage |
| **How is the contract described and evolved?** | Clients and servers are deployed at different times |

The three styles answer these differently:

```
REST     "Here are things (resources). Act on them with a few standard verbs."
           GET /users/42        PUT /users/42        DELETE /orders/7

RPC      "Here are functions. Call them with arguments."
           UserService.GetUser(id=42)     PaymentService.Charge(order, amount)

GraphQL  "Here is a typed graph. Describe the shape you want."
           { user(id: 42) { name orders(last: 5) { total } } }
```

None of these is "best". They make different trade-offs between what **generic infrastructure** can
understand, how **efficient** the wire format is, and how much **freedom the client** has.

---

## 2. REST: designing around resources

HTTP is a protocol; **REST** (Representational State Transfer, from Roy Fielding's 2000 dissertation) is
a **style** of using it. Most "REST APIs" in practice follow only part of Fielding's definition, but the
core ideas are what matter:

- **Resources** are the nouns, each named by a URL: `/users/42`, `/users/42/orders`.
- **Representations**: the client never touches the resource itself, only a representation of it
  (JSON, HTML, ...), negotiated with `Accept` / `Content-Type` headers.
- **A uniform interface**: the same small set of methods applies to every resource, with the same
  meaning everywhere. This is the key point. Because `GET` *means* the same thing on every URL, generic
  infrastructure (caches, proxies, retry logic, browsers, crawlers) can act on requests correctly
  without understanding your application.
- **Stateless requests**: each request carries everything needed to process it (credentials, IDs). The
  server keeps no per-client conversation state between requests. This is what lets any server behind a
  load balancer handle any request (topic 01's stateless components; topic 09).
- **Cacheability**: responses say whether and how long they can be cached (topics 11 and 14).

### Methods, and the two properties that matter

| Method | Meaning | Safe | Idempotent |
|---|---|---|---|
| `GET` | Read a representation | Yes | Yes |
| `HEAD` | `GET` without the body | Yes | Yes |
| `OPTIONS` | What can I do here? (used by CORS) | Yes | Yes |
| `PUT` | Replace the resource with this representation (or create it at this URL) | No | Yes |
| `DELETE` | Remove the resource | No | Yes |
| `POST` | Process this: create a child resource, trigger an action | No | No |
| `PATCH` | Apply a partial modification | No | Not guaranteed |

**Safe** means the client is not asking for any change of state: the request is read-only *from the
client's point of view*. A server may still log it or update a counter, but the client isn't
responsible for that. Crawlers, link prefetchers and caches assume safe methods can be called freely.
(This is why "delete" links that are plain `GET`s have historically been wiped out by crawlers following
every link on a page.)

**Idempotent** means making the same request **once or N times has the same effect on the server's
state**. Not the same *response*: the first `DELETE /orders/7` returns `204`, the second returns `404`,
but the order is gone either way, so the state is the same. Every safe method is automatically
idempotent (doing nothing N times is still nothing).

### Why idempotency is the property that matters most

Recall from topic 03: a timeout tells you nothing about whether the server processed your request.

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
  connection failures, but not `POST`. They don't understand your application, but they do understand
  HTTP's method semantics. That's the uniform interface paying off.
- `PUT` is idempotent because it says "the resource should look like *this*", a final state, not a
  change. Setting `balance = 100` twice gives 100. `POST` "add 10" twice gives 20.
- `PATCH` *can* be idempotent (`set email to x`) or not (`append to list`). The protocol can't promise
  either.

A useful way to think about it: **state the desired end state, not the change**, whenever you can.
"Set quantity to 3" survives retries; "add one more" doesn't.

### Making POST safe to retry: idempotency keys

For operations that are inherently non-idempotent, like payments, the common pattern is an
**idempotency key**: the client generates a unique ID per *logical* operation and sends it with every
attempt of that operation:

```
POST /payments
Idempotency-Key: 6f1c2b0e-8d4a-4f7e-9a21-0c9b5e3d7a11
{ "amount": 500, "currency": "USD", ... }
```

The server records each key together with its result. If the same key arrives again, it returns the
stored result instead of processing again:

```
attempt 1:  key K  → not seen → charge card → store (K → "201, payment p_91") → response lost
attempt 2:  key K  → seen     → return stored "201, payment p_91"   (no second charge)
```

Retries become safe. Two details matter: the key must be generated **once per logical operation** (not
per attempt), and the server must handle two attempts with the same key arriving **at the same time**
(for example with a unique constraint on the key, so only one insert wins). Topic 38 covers idempotency
in depth, including where to store keys and for how long.

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
| `202 Accepted` | Received, will be processed **later** | The asynchronous handoff (section 6) |
| `204 No Content` | Success, no body | Common for `DELETE`, `PUT` |
| `301` / `308` | Moved permanently | `308` guarantees the method isn't changed |
| `302` / `307` | Temporary redirect | `307` guarantees the method isn't changed |
| `304 Not Modified` | Your cached copy is still valid | Answer to a conditional `GET` (topic 14) |
| `400 Bad Request` | Malformed request | |
| `401 Unauthorized` | **Not authenticated**: who are you? | Badly named; it means "unauthenticated" |
| `403 Forbidden` | Authenticated, but **not allowed** | |
| `404 Not Found` | No such resource | |
| `409 Conflict` | Conflicts with current state | e.g. duplicate username |
| `412 Precondition Failed` | An `If-Match` / `If-Unmodified-Since` check failed | Optimistic concurrency (below) |
| `422 Unprocessable Content` | Well-formed but semantically invalid | Validation errors |
| `429 Too Many Requests` | Rate limited | Respect `Retry-After` (topic 45) |
| `500 Internal Server Error` | Unhandled server bug | |
| `502 Bad Gateway` | A proxy got an invalid response (or none) from upstream | Upstream crashed or closed the connection |
| `503 Service Unavailable` | Temporarily overloaded or down | Often has `Retry-After`; safe to retry later |
| `504 Gateway Timeout` | A proxy's upstream didn't respond in time | The work **may still have happened** |

Two things often go wrong:

- **Returning `200` with `{"error": ...}` in the body.** Every piece of generic infrastructure (load
  balancer health metrics, retry logic, circuit breakers, caches, monitoring) now thinks the request
  succeeded. Error-rate dashboards look perfect while users fail. A cache might even store the error and
  serve it to others.
- **Retrying `4xx` errors.** A `400` will be a `400` forever. Retrying wastes capacity. (`408 Request
  Timeout` and `429` are the exceptions: they're about timing, not the request's content.)

Also note what `502`, `503` and `504` tell you: they usually come from a **proxy or load balancer**
(topics 09 and 10) reporting on the server behind it. A `504` especially does not mean "nothing
happened"; the backend may have finished the work after the proxy gave up waiting.

### Conditional requests: optimistic concurrency over HTTP

Two clients read the same document, both edit it, both `PUT` it back. The second write silently
overwrites the first: a **lost update** (topic 21). HTTP has a built-in fix:

```
GET /docs/9          →  200 OK, ETag: "v17"
PUT /docs/9
If-Match: "v17"      →  200 OK if the doc is still v17
                     →  412 Precondition Failed if someone else changed it first
```

The `ETag` is a version identifier for the representation. `If-Match` turns the `PUT` into a
**compare-and-set**: "only write if nobody else has written since I read". Here is the race, with and
without it:

```
Without If-Match:                         With If-Match:
  Alice GET  → v17                          Alice GET  → v17
  Bob   GET  → v17                          Bob   GET  → v17
  Alice PUT  → v18 (Alice's edit)           Alice PUT If-Match v17 → ok, now v18
  Bob   PUT  → v19 (Bob's edit;             Bob   PUT If-Match v17 → 412
               Alice's edit is gone)          Bob re-fetches v18, merges, retries
```

This is optimistic concurrency control, which topic 22 covers in detail. The same `ETag` with
`If-None-Match` on a `GET` gives cache revalidation (`304 Not Modified`), covered in topic 14.

### Pagination: offset vs cursor

Collection endpoints must paginate, and the choice has real consequences:

- **Offset** (`?offset=1000&limit=50`): simple, allows jumping to page N. But the database must still
  walk past 1,000 rows to skip them, so deep pages get slower. And if rows are inserted or deleted
  between page requests, items get **skipped or duplicated**: insert a new post at the top and every
  item shifts down one, so the last item of page 1 reappears as the first item of page 2.
- **Cursor / keyset** (`?after=<id of last item>&limit=50`): the server queries
  `WHERE id > :after ORDER BY id LIMIT 50`, which an index (topic 18) serves directly, at the same cost
  no matter how deep you go. It's stable under inserts. It can't jump to an arbitrary page, which is
  rarely needed for APIs. Cursors are often encoded as an opaque token so the server can change what's
  inside them later.

### Versioning a REST API

Clients you don't control (mobile apps, partners) will keep calling an old shape for years. The usual
rules: **add, don't change** (new optional fields are safe; clients should ignore fields they don't
know); never repurpose a field's meaning; and when a breaking change is unavoidable, run a new version
side by side (`/v2/...` or a version header) and retire the old one slowly. The same compatibility idea
appears in protobuf below, and in general in topic 16.

---

## 3. RPC and gRPC: calling functions across the network

REST models the API as **resources you act on**. **RPC** (Remote Procedure Call) models it as
**functions you call**: `getUser(42)`, `chargeCard(order, amount)`. The idea is decades old: make a
remote call look like a local function call. A generated **stub** on the client turns the call into a
network message; a **skeleton** (server stub) on the server turns it back into a call.

```
Client process                                       Server process
  result = users.GetUser(42)                           def GetUser(id): ...
        │                                                   ▲
        ▼                                                   │
  [client stub] serialize args ──── network ────▶ [server stub] deserialize, dispatch
  [client stub] deserialize ◀────── network ───── [server stub] serialize result
```

RPC fits naturally when operations are **actions**, not resource updates (`TransferFunds`,
`RecalculateRoute`), and inside an organization where both sides can share a contract.

### The danger in "it looks like a local call"

A remote call is fundamentally different from a local one, and hiding that is the classic RPC
mistake:

| Local call | Remote call |
|---|---|
| Takes nanoseconds | Takes 0.1 ms to hundreds of ms, highly variable (topic 02) |
| Either returns or throws | Can **time out**, leaving you not knowing whether it ran |
| Never partially fails | The other side can crash halfway through |
| Pass a pointer, no copying | Everything must be serialized and copied |
| Always the same version of the code | The other side may be a different version |

Code that forgets this calls remote functions in loops (100 sequential calls × 2 ms = 200 ms), assumes
they always return, and never sets timeouts. Good RPC frameworks don't hide these differences; they
make them explicit: deadlines, error codes that distinguish "unavailable" from "invalid", and schemas
designed for version skew. gRPC is built around exactly these.

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

The numbers (`= 1`, `= 2`) are the important part. On the wire, protobuf doesn't send field *names*; it
sends each field as `(field number, wire type, value)`:

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
  a string to a new server that thinks field 3 is an int will produce errors, or worse, silently
  misread data.

This property, **old and new code can read each other's data**, is called backward and forward
compatibility. Topic 16 covers it in general (Avro, JSON, schema registries).

### gRPC: RPC on top of HTTP/2

gRPC uses HTTP/2 (topic 06) as its transport. Each call is an HTTP/2 stream; the method is the path
(`/UserService/GetUser`); messages are length-prefixed protobuf in DATA frames; the final status travels
in HTTP/2 trailers. From HTTP/2 it gets multiplexing, flow control and header compression for free. It
supports four call types:

| Type | Shape | Example |
|---|---|---|
| Unary | one request → one response | `GetUser` |
| Server streaming | one request → stream of responses | Subscribe to price updates |
| Client streaming | stream of requests → one response | Upload sensor readings, get a summary |
| Bidirectional streaming | streams both ways, independently | Chat, real-time collaboration |

Streaming is a genuine difference from plain REST: a single gRPC call can stay open and carry many
messages each way, with HTTP/2 flow control applying backpressure if one side reads slowly.

gRPC also builds in things that HTTP+JSON APIs usually bolt on:

- **Deadlines.** A client sets an absolute deadline ("this must finish within 200 ms"). It travels with
  the call (`grpc-timeout` header), and a server that calls further services passes the *remaining*
  time along. If the deadline has passed, every service in the chain can stop working on a request
  nobody is waiting for any more (topic 43).

  ```
  Frontend  deadline = now + 300 ms
     └─▶ Orders     (250 ms left)  ── spends 40 ms ──▶
            └─▶ Inventory  (210 ms left)
                   └─▶ Pricing (150 ms left) ... if 0 left: stop, return DEADLINE_EXCEEDED
  ```
- **Status codes designed for distributed systems**: `UNAVAILABLE` (safe to retry), `DEADLINE_EXCEEDED`
  (outcome unknown), `INVALID_ARGUMENT` (don't retry), `RESOURCE_EXHAUSTED`, `ALREADY_EXISTS`, ...
- **Metadata** (headers) for auth tokens and trace IDs (topic 46).

### gRPC's trade-offs

- **Not human-readable.** You can't `curl` it and read the result; you need tooling (and the `.proto`
  file, or server reflection) to inspect traffic.
- **Browsers can't speak it directly.** Browser JavaScript can't control HTTP/2 framing and trailers
  the way gRPC needs. A translating proxy (gRPC-Web) is required. This is why gRPC is mostly used
  **between services**, with REST or GraphQL facing browsers.
- **Load balancing needs care.** gRPC keeps one long-lived HTTP/2 connection and multiplexes everything
  over it. A **layer 4** load balancer (topic 09) balances *connections*, not requests, so it pins every
  request from a client to one backend for the life of the connection. Adding backends doesn't help
  existing clients. You need **layer 7** (request-aware) load balancing, a service mesh sidecar
  (topic 10), or client-side balancing across connections to several backends.
- **No HTTP caching.** Calls are `POST`s with binary bodies; CDNs and HTTP caches can't help.
- **Tighter coupling.** Both sides compile against shared `.proto` files, which must be distributed and
  versioned.

---

## 4. GraphQL: let the client describe the shape

REST endpoints return fixed shapes chosen by the server. That leads to two opposite problems,
especially for UIs:

- **Over-fetching**: `GET /users/42` returns 40 fields; the screen needs 3.
- **Under-fetching**: showing a user with their last 5 orders and each order's items takes
  `GET /users/42`, then `GET /users/42/orders`, then one `GET` per order. On a mobile network with
  150 ms RTT, several sequential round trips add up to most of a second.

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

One round trip, exactly the requested fields, in exactly that shape. The schema is introspectable, so
tools can autocomplete and validate queries. Front-end teams can change what a screen shows without
waiting for a new backend endpoint. Besides queries, GraphQL has **mutations** (writes) and
**subscriptions** (server-pushed updates, usually over WebSockets; topic 08).

### How it executes

On the server, each field is backed by a **resolver** function that knows how to fetch it. Execution
walks the query tree, calling resolvers level by level:

```
user(id: 42)            → resolver: SELECT * FROM users WHERE id = 42
  ├─ name               → read from the user row
  └─ orders(last: 5)    → resolver: call the orders service
        ├─ total        → read from each order
        └─ items        → resolver per order: call the catalog service
```

Resolvers can call anything: a database, a REST service, a gRPC service. This is what makes GraphQL a
natural **aggregation layer** in front of many backends, and also where its costs come from.

### What GraphQL trades away

**1. HTTP caching.** Queries are usually `POST`ed to a single URL like `/graphql`. To an HTTP cache or
CDN, every request looks the same and isn't cacheable (`POST` isn't). The uniform interface that let
generic infrastructure understand REST requests is gone. Caching moves into the client library (a
normalized cache keyed by object type and ID), or you use **persisted queries**: register queries ahead
of time, then send a hash via `GET`, which makes them cacheable again.

**2. Query cost becomes the client's choice.** A client can write a query that's cheap to send and very
expensive to run:

```graphql
{ users(first: 1000) { friends(first: 1000) { friends(first: 1000) { name } } } }
```

That's up to a billion resolutions in one request. Servers need **depth limits**, **complexity / cost
analysis** before execution, timeouts, and rate limits based on cost rather than request count
(topic 45). Allowing only persisted queries (an allow-list) is the strongest defence.

**3. The N+1 problem.** Resolvers run per field. Fetching 50 orders and then the `customer` of each
naively runs 1 query for the orders plus 50 queries for customers. The standard fix is **batching**
(the *DataLoader* pattern): collect all the customer IDs requested within one execution tick and fetch
them in a single `WHERE id IN (...)` query, with a per-request cache so the same customer isn't fetched
twice.

**4. Errors don't use HTTP status codes.** A GraphQL response is usually `200 OK` even if part of it
failed, with an `errors` array alongside partial `data`. Partial success is genuinely useful (the page
can render without the one failed widget), but it means generic monitoring sees a 100% success rate.
You need GraphQL-aware observability.

**5. Authorization per field.** Since any client can reach any part of the graph from any entry point,
authorization must be enforced at the field / object level, not per endpoint.

GraphQL works best as an **aggregation layer** in front of many services, serving varied clients (web,
mobile, partners) whose data needs differ and change often. For simple CRUD, or for service-to-service
calls, its costs usually aren't worth it.

### REST vs gRPC vs GraphQL

| | REST (HTTP + JSON) | gRPC | GraphQL |
|---|---|---|---|
| Model | Resources + uniform methods | Functions on services | A typed graph, client-chosen queries |
| Contract | Optional (OpenAPI) | Required (`.proto`) | Required (schema) |
| Payload | JSON text, usually | Protobuf binary | JSON text |
| Transport | HTTP/1.1, 2 or 3 | HTTP/2 | HTTP, usually one `POST` endpoint |
| HTTP caching | Excellent | None | Poor (without persisted queries) |
| Retry semantics | From the method (`GET`/`PUT` idempotent) | From status codes + your own rules | Queries are reads; mutations need care |
| Browser support | Native | Needs a proxy (gRPC-Web) | Native |
| Streaming | Not natively (see SSE, topic 08) | Built in, both directions | Subscriptions (usually over WebSockets) |
| Typical home | Public APIs, simple CRUD | Internal service-to-service | Front-end aggregation layer |

A common real architecture uses all three: gRPC between internal services, a GraphQL or REST gateway
in front of them (topic 10), and REST for public/partner APIs.

---

## 5. Synchronous vs asynchronous communication between services

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

**Latency adds up** along a chain, and fan-out amplifies the tail (topic 02): if a request calls 100
backends in parallel and waits for all of them, and each has a 1% chance of being slow, then
`1 - 0.99^100 ≈ 63%` of requests hit at least one slow backend.

**Failures cascade.** If E slows down, D's threads and connections pile up waiting on E, then C's pile
up waiting on D, all the way back to the user:

```
E slow (2 s instead of 20 ms)
D: requests in flight = rate × latency (Little's Law, topic 02) → 100× more → D's pool full
C: waiting on D → C's pool full
B, A: same → user-facing service unavailable, though only E has a problem
```

One slow dependency can exhaust resources in every service upstream. Timeouts, circuit breakers and
bulkheads (topics 43 and 44) exist mainly to contain this.

### Asynchronous: hand off and move on

Service A puts a message on a **queue or log** (topics 36 and 37) and continues without waiting.
Service B consumes it whenever it can.

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
- **Easy fan-out**: several different consumers can react to the same event (send email, update the
  search index, record analytics) without A knowing about any of them.

What you pay:

- **No immediate answer.** A can't tell the user "done", only "accepted". The result must reach the
  user some other way (polling a status, a push notification, a webhook; topic 08).
- **Eventual consistency.** For a while, different parts of the system disagree (the order exists, but
  the email hasn't been sent and the search index doesn't show it).
- **Duplicates and ordering.** Most queues deliver **at least once**, so consumers must be idempotent.
  Ordering is often only guaranteed partially (topics 37 and 38).
- **Harder to debug and observe.** A request's path is spread across queues and time. You need tracing
  that follows messages (topic 46), and you need to watch queue depth and consumer lag.
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
happens asynchronously. The status can be polled, or pushed with SSE, WebSockets or a webhook (topic
08). The `POST` should carry an idempotency key so a retried submission doesn't start two jobs.

### When to use which

Use **synchronous** when the caller genuinely **needs the answer to continue**: reading data to
display, checking authorization, validating input, checking a balance before spending it.

Use **asynchronous** when the work can **happen after the response**, when it's slow, when it fans out
to several consumers, or when the downstream is less reliable than you'd like to depend on directly:
sending emails, generating reports, updating search indexes, processing uploads, recording analytics.

A useful rule of thumb: **keep the synchronous path short.** On the path from a user's request to its
response, call only what's strictly needed, and push everything else onto queues. Each synchronous
dependency you remove is one less thing that can make the user's request fail or slow.

---

## 6. Putting it together: one action, several styles

A user posts a comment in a mobile app on a generic social site:

1. The app sends `POST /posts/9/comments` with an idempotency key, over HTTP/3 (topic 06) to the
   nearest edge.
2. The API gateway (topic 10) authenticates the request, then calls the comments service over **gRPC**
   with a 300 ms deadline (internal, binary, multiplexed over a pooled HTTP/2 connection).
3. The comments service writes the comment to its database and returns **`201 Created`**. That's all
   that happens **synchronously**.
4. It also publishes a `CommentCreated` event to a message log. **Asynchronously**, consumers notify
   the post's author, update the search index, and count the comment for analytics. If the
   notification service is down, comments keep working; notifications catch up later.
5. Other users viewing the post receive the new comment over an open SSE stream or WebSocket (topic
   08), pushed by a server that subscribed to the same event.
6. If the phone lost the `201` response and retried, the **idempotency key** made sure the comment was
   posted only once.

---

## 7. How it connects

- **Idempotency** is the property that makes retries safe. It comes back in retries and backoff
  (topic 43), message delivery (topic 38) and sagas (topic 35).
- **Statelessness** in REST is what lets load balancers (topic 09) send any request to any server;
  persistent connections (topic 08) give some of that up.
- **Cacheability** of `GET` is the foundation of HTTP caching and CDNs (topics 11 and 14). **ETags**
  return in topic 14 for revalidation and topic 22 as optimistic concurrency.
- **gRPC on long-lived HTTP/2 connections** forces **L7 or client-side load balancing** (topic 09) and
  is a main reason service meshes exist (topic 10).
- **Protobuf schema evolution** previews data encoding and compatibility (topic 16).
- **GraphQL as an aggregation layer** is one form of the API gateway pattern (topic 10).
- **Synchronous call chains** explain why timeouts, circuit breakers and bulkheads exist (topics 43 and
  44); **asynchronous** communication is all of Part 7 (topics 36–42).

## Common misconceptions

- **"Idempotent means the response is the same every time."** → It means the *server state* ends up
  the same. A second `DELETE` returns `404`, and that's fine.
- **"POST is for creating, PUT is for updating."** → `PUT` means "make the resource at this URL equal
  to this" (create or replace; idempotent). `POST` means "process this" (not idempotent). The difference
  is about retry semantics, not create vs update.
- **"Return 200 and put the error in the body."** → This hides failures from every piece of generic
  infrastructure: monitoring, retries, load balancer health, circuit breakers, caches.
- **"401 means not allowed."** → `401` means *not authenticated*; `403` means authenticated but *not
  allowed*.
- **"A 504 means the request didn't happen."** → The proxy stopped waiting; the backend may have
  completed the work.
- **"gRPC is just faster REST."** → It's a different model (functions vs resources) with different
  trade-offs: no HTTP caching, no direct browser support, and load-balancing pitfalls.
- **"RPC lets you treat remote calls like local ones."** → That's the trap. Remote calls are slow,
  variable, can time out with unknown outcome, and cross version boundaries.
- **"GraphQL is a database query language."** → It's an API query language. Each field is a resolver
  calling whatever backend you choose, and how efficiently that happens is your problem (N+1).
- **"Asynchronous is always more scalable."** → It moves complexity into eventual consistency,
  duplicate handling and observability. Use it where the work truly doesn't need to finish before the
  response.

## Further reading (supplements)

- Roy Fielding, *Architectural Styles and the Design of Network-based Software Architectures* (2000),
  chapter 5 (REST)
- MDN Web Docs: HTTP request methods and HTTP response status codes references
- RFC 9110 (HTTP Semantics): sections on safe and idempotent methods and conditional requests
- protobuf.dev: the "Language Guide (proto3)" and "Encoding" pages
- grpc.io: "Core concepts" and "Deadlines" guides
- graphql.org: "Learn" section, including the pages on pagination, caching and performance
- Martin Kleppmann, *Designing Data-Intensive Applications*, chapter 4 ("Modes of Dataflow": REST,
  RPC, message passing)

---

## Questions

1. Why does a uniform interface (the same methods meaning the same thing on every URL) matter so much?
   Give two pieces of infrastructure that rely on it.

2. Define "safe" and "idempotent". Give a method that is idempotent but not safe, and one that is
   neither.

3. A client sends `DELETE /orders/7`, times out, retries, and gets `404`. Was `DELETE` idempotent?
   Explain.

4. A client sends `POST /payments` and the connection drops before any response arrives. Why can't the
   client just retry? Describe a mechanism that makes the retry safe, including one subtle detail it
   must get right.

5. Why will a reverse proxy or HTTP client automatically retry a failed `GET` or `PUT` but not a
   `POST`?

6. An API returns `200 OK` with `{"error": "database unavailable"}` in the body. List three things in
   the surrounding infrastructure that now behave wrongly.

7. What's the difference between `401` and `403`? Between `502`, `503` and `504`?

8. A client gets a `504 Gateway Timeout` on a request that transfers money. What does it know about
   whether the transfer happened? What should it do?

9. Two users edit the same wiki page at the same time and both save. Describe how `ETag` and `If-Match`
   prevent one edit from silently overwriting the other. Which status code does the loser receive?

10. A feed API uses `?offset=N&limit=50`. Users report seeing the same post twice when scrolling, and
    deep pages are slow. Explain both problems and the alternative design.

11. What's the fundamental danger in RPC's goal of "making a remote call look like a local call"? List
    three ways remote calls differ from local ones.

12. In Protocol Buffers, why are field *numbers* sent on the wire instead of field names? A team removes
    field 4 (`string nickname`) and months later adds a new field 4 (`int64 karma`). What can go wrong,
    and how does protobuf help prevent it?

13. How do gRPC deadlines differ from a simple per-call timeout? Why does propagating them matter in a
    chain of services?

14. A service calls a gRPC backend through a layer 4 load balancer. Ops add 5 new backend instances, but
    the existing ones stay overloaded and the new ones sit idle. Why? What fixes it?

15. What problems does GraphQL solve for a mobile client compared to a set of REST endpoints? Name
    three things it gives up or makes harder, and how each is usually mitigated. A query fetches 100
    posts and each post's author, and the database receives 101 queries: what's that called and what's
    the fix?

16. A checkout request synchronously calls 6 services, each with 99.9% availability. What is the
    best-case availability of checkout? Then: service E in a synchronous chain A → B → C → D → E becomes
    slow (not down). Describe how this spreads to A.

17. A "generate annual report" endpoint takes 2 minutes. Design its API so the client isn't stuck
    waiting on an open request. For each of the following, would you call synchronously or
    asynchronously: (a) check a user's balance before a purchase, (b) send the order confirmation
    email, (c) update the product search index after a price change, (d) validate a coupon code at
    checkout?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Because intermediaries that know nothing about your application can still act correctly. If
`GET` always means "read, no side effects", a CDN or reverse proxy can cache it, a browser can prefetch
it, a crawler can follow it, and a client library or load balancer can retry it after a connection
failure. If `PUT` and `DELETE` always mean idempotent operations, retry logic can repeat them. Without a
uniform interface (as in RPC or GraphQL-over-POST), each of these needs application-specific knowledge,
so generic caches and retries can't help.

**2.** **Safe**: the request asks for no change to server state (read-only from the client's point of
view). **Idempotent**: sending it once or many times leaves the server in the same state. `PUT` and
`DELETE` are idempotent but not safe. `POST` is neither (and `PATCH` isn't guaranteed to be either).
Every safe method is also idempotent.

**3.** Yes. Idempotency is about the **effect on server state**, not the response. After the first
`DELETE` (which probably succeeded before the timeout), the order is gone. After the retry, the order is
still gone. Same state. The `404` just reports that there's nothing left to delete. Clients retrying a
`DELETE` should treat a `404` as "already done".

**4.** The client doesn't know whether the server processed the request: the request may have been
lost, or the payment may have gone through and only the response was lost. `POST` isn't idempotent, so
retrying could charge twice. The fix is an **idempotency key**: the client generates a unique ID for
this logical payment and sends it on every attempt. The server stores each key with its outcome; when it
sees a key it has already processed, it returns the stored result instead of charging again. Subtle
details: the key must be generated once per logical operation, not per attempt (otherwise every retry
looks new); and the server must handle two attempts with the same key arriving concurrently, e.g. a
unique constraint on the key so only one can claim it (topic 38).

**5.** Generic infrastructure can't understand your application, but it does know HTTP's method
semantics. `GET` and `PUT` are defined as idempotent, so repeating them is safe by contract, whatever
the application does. `POST` makes no such promise, so an automatic retry could duplicate a side effect
(an order, a payment). This is the uniform interface paying off.

**6.** (a) **Monitoring and alerting** count it as a success, so error-rate dashboards and SLO alerts
stay green during an outage. (b) **Retry logic** in clients and proxies doesn't retry, because 2xx
means success. (c) **Load balancer passive health checks / outlier detection** won't eject the failing
instance, because it isn't returning 5xx (topic 09). (d) **Caches** (CDN, reverse proxy, browser) may
store the error response and serve it to others as if it were valid content. Circuit breakers also
never open.

**7.** `401` = **not authenticated** ("I don't know who you are"; send credentials). `403` =
authenticated but **not authorized** ("I know who you are, and you can't do this"). `502 Bad Gateway`:
a proxy got an invalid response or none (the upstream crashed, reset the connection, or sent garbage).
`503 Service Unavailable`: the server is temporarily unable to handle the request (overloaded, in
maintenance), often with `Retry-After`. `504 Gateway Timeout`: a proxy gave up waiting for the upstream
to respond.

**8.** Nothing certain. A `504` means a **proxy stopped waiting**; the backend may have failed, may
still be running, or may have completed the transfer after the proxy gave up. The client must not
blindly retry a non-idempotent request. It should retry **with the same idempotency key** (so the server
returns the original outcome if the transfer happened), or query the transfer's status by a
client-generated ID before deciding.

**9.** Each user's `GET` returns the page with an `ETag` (a version, say `"v5"`). Each saves with `PUT`
and `If-Match: "v5"`. The server applies the first `PUT` because the current version is `v5`, and the
page becomes `v6`. The second `PUT` still says `If-Match: "v5"`, which no longer matches, so the server
rejects it with **`412 Precondition Failed`**. That user must re-fetch, merge or redo their edit, and
try again. This is optimistic concurrency control (compare-and-set) built into HTTP, and it prevents the
lost update. (The server's check-and-write must itself be atomic, or two `PUT`s could both pass the
check.)

**10.** **Duplicates**: if new posts are inserted at the top between page requests, every item shifts
down; the item that was #50 is now #51 and shows up again at the start of the next page (deletions cause
skips instead). **Slowness**: `OFFSET 10000` makes the database produce and throw away 10,000 rows
before returning 50, so cost grows with depth. **Cursor (keyset) pagination** fixes both: the client
passes the sort key of the last item it saw (`?after=<id or timestamp>`), and the server queries
`WHERE key < :after ORDER BY key DESC LIMIT 50`. An index serves this directly at constant cost, and
insertions elsewhere don't shift the window. The trade-off is that you can't jump to page N.

**11.** Hiding the network encourages code that treats remote calls like local ones: calling them in
loops, assuming they always return, ignoring timeouts and partial failure. Differences: (1) **latency**
is orders of magnitude higher and highly variable; (2) a call can **time out with an unknown outcome**
(it may or may not have run); (3) the remote side can **fail partway** or be unavailable; (4) arguments
must be **serialized and copied** (no pointers or shared memory); (5) the other side may run a
**different version** of the code.

**12.** Numbers are short (a varint tag combining number and wire type, usually 1 byte), so messages
are much smaller than with repeated field names. More importantly, numbers decouple the wire format from
naming: you can rename fields freely, and old and new code agree on what each field is by number. In
the scenario, old clients and servers, and old stored data, still use field 4 as a string. A new reader
seeing field 4 with length-delimited wire type where it expects a varint will error or drop it; with
compatible wire types, data is silently **misinterpreted**. Protobuf prevents this with `reserved 4;`
(and `reserved "nickname";`) when the field is removed: the compiler then refuses any future
definition that reuses that number or name.

**13.** A timeout is local: "I'll wait 2 s for this call". A **deadline** is an absolute point in time
for the whole operation that travels with the request. Each service passes along the *remaining* time
to its own downstream calls. Without propagation, a frontend may give up after 1 s while the services
behind it keep working for many more seconds, each with its own fresh timeout, wasting capacity on a
result nobody will use; under load, that wasted work can push the system into overload. With
propagation, every service can stop as soon as the deadline passes, and a service can refuse work it
can't finish in the remaining time.

**14.** gRPC multiplexes all calls over one (or a few) long-lived HTTP/2 connections. A **layer 4**
load balancer only chooses a backend when a **connection** is opened; after that, every request on that
connection goes to the same backend. The existing clients' connections are already pinned to the old
backends, so the new ones only get traffic from new connections, which rarely open. Fixes: a **layer 7**
(HTTP/2-aware) load balancer or sidecar proxy that balances individual requests/streams; **client-side
load balancing** (the client discovers all backends and spreads calls across connections to each); or
periodically recycling connections (a maximum connection age) so they rebalance.

**15.** Solves: **over-fetching** (client asks for exactly the fields it needs, saving bandwidth on
mobile); **under-fetching** (one query replaces several sequential REST round trips, which matter a lot
at mobile RTTs); and front-ends can evolve their data needs without new endpoints. Gives up:
(1) **HTTP caching**, since queries are `POST`s to one URL → client-side normalized caches, persisted
queries sent as `GET` with a hash; (2) **predictable server cost**, since clients can send very
expensive queries → depth limits, cost analysis before execution, cost-based rate limiting,
allow-listed persisted queries; (3) **HTTP-level error semantics**, since responses are `200` with an
`errors` array → GraphQL-aware monitoring; also per-field authorization. The 101 queries are the **N+1
problem**: one query for the list, then one per post for its author, because each `author` resolver
runs independently. The fix is **batching** with the DataLoader pattern: collect all author IDs
requested in the same execution tick and issue one `SELECT ... WHERE id IN (...)`.

**16.** 0.999^6 ≈ **0.994**, i.e. 99.4%, roughly 52 hours of downtime a year compared with 8.8 hours
for each service alone, and that's best case, assuming independent failures. It argues for shortening
the synchronous path and adding fallbacks for non-critical dependencies. For the slow E: D's calls to E
take longer, so each of D's threads/connections is held longer. By Little's Law, with the same arrival
rate, D now has many more requests in flight; its thread pool and its connection pool to E fill up, and
new requests to D queue or fail. That makes D slow, so C's calls to D hold C's resources longer, and C's
pools fill up. The same happens at B, then A. The user-facing service fails though only E has a problem.
Timeouts, circuit breakers and bulkheads stop this spread (topics 43 and 44).

**17.** `POST /reports` (with an idempotency key) validates the request, enqueues the job, and returns
**`202 Accepted`** immediately with `Location: /reports/{id}`. The client then checks
`GET /reports/{id}`, which returns `{"status": "queued" | "running" | "done" | "failed"}` and a link to
the result when done; instead of polling, the server can notify via SSE, WebSockets, push notification
or a webhook. Workers consume jobs at their own pace. (a) **Synchronous**: the purchase can't proceed
without an accurate answer now. (b) **Asynchronous**: the order is complete without it; the email
service may be slow or down; retries are easy. (c) **Asynchronous**: a search index a few seconds
behind is fine, and price changes shouldn't fail because indexing does. (d) **Synchronous**: the price
shown and charged depends on it.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What are REST's core ideas?
**A:** Resources named by URLs, representations, a uniform interface of methods, stateless requests, cacheable responses.

**Q:** Why does the uniform interface matter?
**A:** Because methods mean the same thing everywhere, generic infrastructure (caches, proxies, retry logic) can act correctly without understanding the application.

**Q:** What does "safe" mean for an HTTP method?
**A:** The client isn't requesting any state change (read-only from the client's view). GET, HEAD, OPTIONS.

**Q:** What does "idempotent" mean?
**A:** Doing it once or N times leaves the server in the same state. The response may differ.

**Q:** Which methods are idempotent?
**A:** GET, HEAD, OPTIONS, PUT, DELETE. Not POST. PATCH not guaranteed.

**Q:** Why is idempotency so important?
**A:** A timeout leaves the outcome unknown; idempotent requests can simply be retried safely.

**Q:** How do you make a POST safe to retry?
**A:** An idempotency key: a client-generated unique ID per logical operation; the server stores the result per key and returns it on repeats.

**Q:** PUT vs POST, in one line?
**A:** PUT = "make the resource at this URL equal to this" (idempotent). POST = "process this" (not idempotent).

**Q:** 4xx vs 5xx?
**A:** 4xx: the client's request is wrong; don't retry unchanged. 5xx: the server's problem; maybe retry (idempotent methods, with backoff).

**Q:** 401 vs 403?
**A:** 401: not authenticated. 403: authenticated but not allowed.

**Q:** 502 vs 503 vs 504?
**A:** 502: proxy got a bad/no response from upstream. 503: temporarily unavailable/overloaded. 504: proxy timed out waiting for upstream.

**Q:** Does a 504 mean the request didn't happen?
**A:** No. The proxy stopped waiting; the backend may have completed the work.

**Q:** Why is "200 with an error in the body" harmful?
**A:** Monitoring, retries, health checks, circuit breakers and caches all treat it as success.

**Q:** How do ETag + If-Match prevent lost updates?
**A:** The write only succeeds if the resource's version still matches the one the client read; otherwise 412 Precondition Failed. A compare-and-set.

**Q:** Offset vs cursor pagination?
**A:** Offset: simple, jump to any page, but slow when deep and skips/duplicates under inserts. Cursor: `WHERE key > last`, constant cost, stable, no page jumping.

**Q:** What's the classic mistake of RPC?
**A:** Pretending a remote call is like a local one, hiding latency, partial failure, unknown outcomes and version skew.

**Q:** What does protobuf send on the wire instead of field names?
**A:** Field number + wire type (as a varint tag), then the value. Integers as varints.

**Q:** Safe protobuf schema changes?
**A:** Add fields, remove fields (reserving the number), rename fields. Never reuse a field number or change a field's type incompatibly.

**Q:** What transport does gRPC use?
**A:** HTTP/2: each call is a stream, path is `/Service/Method`, length-prefixed protobuf messages, status in trailers.

**Q:** gRPC's four call types?
**A:** Unary, server streaming, client streaming, bidirectional streaming.

**Q:** Deadline vs timeout?
**A:** A deadline is an absolute time for the whole operation, propagated downstream with the remaining budget; a timeout is a local per-call wait.

**Q:** Why does gRPC need L7 or client-side load balancing?
**A:** It sends everything over long-lived HTTP/2 connections; L4 balancers balance connections, so all requests pin to one backend.

**Q:** Why can't browsers call gRPC directly?
**A:** Browser JS can't control HTTP/2 framing and trailers as gRPC needs; it requires a gRPC-Web proxy.

**Q:** What does GraphQL solve?
**A:** Over-fetching and under-fetching: the client requests exactly the fields and nested data it needs, in one round trip.

**Q:** What does GraphQL give up?
**A:** HTTP caching, predictable server cost, HTTP status-code error semantics; adds N+1 risk and per-field authorization.

**Q:** What is the N+1 problem and its fix?
**A:** One query for a list, then one per item for related data. Fix: batching (DataLoader): collect IDs, fetch with one `IN (...)` query.

**Q:** How do you protect a GraphQL server from expensive queries?
**A:** Depth limits, query cost analysis, cost-based rate limits, timeouts, persisted (allow-listed) queries.

**Q:** What is temporal coupling?
**A:** A synchronous caller can only succeed if the callee is up and responsive right now.

**Q:** Availability of a request through 5 synchronous services at 99.9% each?
**A:** 0.999^5 ≈ 99.5%.

**Q:** How does a slow dependency cascade?
**A:** Callers' threads and connections are held longer, pools fill, callers become slow, and it spreads up the chain.

**Q:** What does async messaging buy, and cost?
**A:** Buys: temporal decoupling, load levelling, independent scaling, fan-out. Costs: no immediate result, eventual consistency, duplicates/ordering, harder observability, a broker to run.

**Q:** What's 202 Accepted for?
**A:** The request was accepted and will be processed later; usually with a `Location` to check status.

**Q:** Rule of thumb for request paths?
**A:** Keep the synchronous path short; push everything that can happen later onto queues.
