# 08 — Real-Time Communication

> Part 2 — Networking & Communication · Prerequisites: topics 02, 03, 06, 07

HTTP is **client-initiated**: the server can only answer requests. But many features need the server to
tell the client something *when it happens*: a new chat message, a price change, a delivery driver's
position, a background job finishing, the next token of generated text. The server knows first; the
client has to find out.

There are four standard ways to do this: **short polling**, **long polling**, **Server-Sent Events**
and **WebSockets**. They differ in latency and overhead, but the bigger difference is architectural.
Three of them hold a connection open for a long time, and that turns stateless web servers (topic 07)
into stateful connection holders. This topic covers the mechanism behind each technique, what it costs a
server to hold millions of open connections, and how a message reaches the right client when clients
are spread across many servers.

By the end of this topic you should be able to:

1. Describe short polling, long polling, SSE and WebSockets at the mechanism level.
2. Explain the "gap" problem in long polling and how cursors and `Last-Event-ID` solve it.
3. Choose between the four techniques for a given feature, and justify the choice.
4. Explain what each open connection costs a server (memory, file descriptors, TLS state, threads) and
   why event-driven servers are needed to hold very many of them.
5. Explain why persistent connections make servers stateful, and what that does to deploys, load
   balancing and failure recovery (reconnection storms).
6. Design fan-out of messages to clients connected to many different servers: connection registry,
   pub/sub backbone, and handling offline clients.

---

## 1. The problem: the server knows first

```
Something happens on the server side:       How does the client find out?
  - user B sends user A a chat message
  - an order's status changes                   Client ──────?────── Server
  - a stock price ticks
  - a report finishes generating
```

Every technique answers two questions:

- **Who initiates?** Does the client keep asking, or does the server send when ready?
- **What is held open?** Nothing between requests, a pending request, or a long-lived connection?

```
Short polling   client asks on a timer          nothing held between asks
Long polling    client asks; server waits        one pending request per client
SSE             one request, endless response    one open HTTP response per client
WebSocket       upgraded, two-way channel        one open socket per client
```

---

## 2. Short polling

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
  requests/second** of mostly empty answers, each with full headers, auth checks, maybe a TLS
  handshake if connections aren't reused (topic 05), and maybe a database query.
- **Upside**: dead simple; works everywhere; **stateless on the server**; no long-lived connections;
  any server can answer any poll; ordinary HTTP caching and load balancing apply.

Shortening the interval improves latency only by multiplying cost; the trade-off is linear and
unforgiving. Short polling is fine when updates are infrequent and a delay of seconds to minutes is
acceptable (checking an export job's status, refreshing a dashboard every minute).

---

## 3. Long polling

The client asks, and the server **doesn't answer until it has something** (or until a timeout, say
30 s, after which it returns empty). As soon as the client gets a response, it immediately asks again:

```
Client                         Server
  │── any updates since 1041? ──▶│
  │                              │  (holds the request open...)
  │                              │  ...event 1042 happens
  │◀──────── 1042, here ─────────│
  │── any updates since 1042? ──▶│  (immediately re-asks)
  │                              │  (holds...)
  │                              │  ...30 s pass, nothing
  │◀──────── nothing ────────────│
  │── any updates since 1042? ──▶│
```

- **Latency**: near-instant for the first event.
- **Cost**: far fewer empty responses (roughly one per client per timeout period when idle, plus one per
  event). But every waiting client ties up an open request on the server, so the server must handle
  many idle connections cheaply (section 7).
- **Upside**: plain HTTP; works through almost any proxy and firewall.

### The gap, and why cursors fix it

Between a response and the next request, **no request is waiting**. Events that occur in that window
have nowhere to go:

```
t=0     server responds with event 1042
t=0-80ms  response in flight; client processes it; new request in flight   ← the gap
t=40ms  event 1043 happens: no pending request for this client!
t=80ms  new request arrives
```

If the server only delivers "new events from now on", event 1043 is lost. The fix is a **cursor**: the
client sends "updates since 1042", and the server keeps a short history of recent events so it can
return everything after that cursor. Each re-request also pays full HTTP header overhead, and a burst of
events turns into a burst of request/response cycles (the server can batch several events into one
response to soften this).

Long polling was the standard technique before WebSockets and SSE existed, and it's still a robust
fallback in restrictive networks.

---

## 4. Server-Sent Events (SSE)

The client makes one ordinary HTTP request; the server responds with `Content-Type: text/event-stream`
and **never finishes the response**, writing events into it as they happen:

```
GET /events HTTP/1.1
Accept: text/event-stream

HTTP/1.1 200 OK
Content-Type: text/event-stream

id: 1041
data: {"price": 101.5}

id: 1042
event: trade
data: {"price": 101.7}

: this line is a comment, often sent every 15-30 s as a keep-alive

...  (connection stays open)
```

Each event is a few `field: value` lines (`data`, `id`, `event`, `retry`) followed by a blank line.
Browsers have a built-in `EventSource` API that handles it, including:

- **Automatic reconnection.** If the connection drops, the browser waits (the `retry:` field can set
  how long) and reconnects, sending a `Last-Event-ID: 1042` header so the server can resume from where
  the client left off. The cursor that long polling had to implement by hand is part of the protocol.

Characteristics:

- **One direction only**: server to client. The client sends anything else as normal HTTP requests.
- **Text only** (binary data must be encoded, e.g. base64).
- **It's just HTTP**: works with existing auth (cookies, headers), proxies, load balancers and
  compression. Over HTTP/1.1 each SSE stream uses up one of the browser's ~6 connections per origin
  (topic 06); over HTTP/2 it's just one stream on the shared connection, so the limit doesn't matter.
- **Proxies can interfere**: a proxy that buffers responses (waiting for the whole body before
  forwarding) will hold events back. SSE endpoints must disable response buffering in proxies along the
  path, and send periodic comment lines so idle-connection timeouts don't kill the stream.

SSE fits most "server notifies client" features: live scores, notifications, dashboards, progress
updates, and streaming generated text token by token.

---

## 5. WebSockets

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

The `Sec-WebSocket-Key` / `Accept` exchange (the server hashes the key with a fixed GUID) proves the
server really understands WebSockets and isn't some HTTP server tricked into accepting. After that, both
sides send **frames** with a small header (2 to 14 bytes) carrying text or binary messages, plus control
frames: **ping/pong** for keep-alive and liveness checks, and **close**.

Frames from client to server are **masked** (XORed with a random 4-byte key chosen per frame). This
isn't for confidentiality (TLS does that); it protects **intermediaries**. Without masking, a malicious
page could send payloads crafted to look like HTTP requests, and an old transparent proxy that didn't
understand WebSockets might parse them as HTTP and cache attacker-chosen content under a real URL (cache
poisoning). Random masking makes the bytes on the wire unpredictable to the page's script.

Characteristics:

- **Lowest overhead per message** and true two-way communication.
- **Not HTTP after the upgrade**: no status codes, no caching, no standard semantics, no per-message
  auth headers. You design your own message protocol on top: message types, request IDs to match
  replies to requests, acknowledgements, error formats.
- **No built-in reconnection or resume.** If the connection drops, the client must reconnect and
  re-sync, with your own cursor logic.
- **Middleboxes** sometimes kill idle long-lived connections; ping frames keep them alive and also let
  each side detect a dead peer (a TCP connection whose other end vanished can look open for a long
  time; topic 03).
- **Authentication happens once**, at the upgrade. If a user's token expires or is revoked an hour
  later, the socket is still open; servers need a way to re-check or close connections.

WebSockets fit genuinely **interactive, bidirectional, high-frequency** traffic: chat, multiplayer
games, collaborative editing, trading terminals.

---

## 6. Choosing

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
channel?** If not, SSE is usually simpler than WebSockets and keeps all of HTTP's infrastructure on your
side. If updates are rare and a delay is fine, short polling avoids holding connections at all.

### Between servers: webhooks

A related pattern between systems is the **webhook**: instead of a partner polling your API, they give
you a URL and you `POST` to it when something happens. It's server push between systems, with no
connection held open. Webhooks need retries (the receiver may be down), so receivers must handle
duplicate deliveries idempotently (topic 38), and they should verify a signature on each delivery so
that anyone who learns the URL can't forge events.

### On mobile: platform push

When a phone app is in the background, the OS won't keep its sockets alive. Delivery goes through the
platform's push service (APNs, FCM), which holds **one** shared connection per device on behalf of all
apps. Your server sends to the push service; the push service wakes the app. Real-time apps typically
combine an open socket while in the foreground with platform push while in the background.

---

## 7. Holding millions of open connections: what it costs

Long polling, SSE and WebSockets all mean **long-lived connections held open on the server**. A chat
or notification service might hold hundreds of thousands per machine. What does each one cost?

### Per-connection resources

| Resource | Why it's used | Rough size |
|---|---|---|
| **File descriptor** | Every socket is an fd; OS limits (`ulimit -n`) default to ~1,024 | Must be raised to millions |
| **Kernel socket buffers** | Send and receive buffers per TCP connection (topic 03) | Kilobytes each when idle; can grow to MBs if traffic flows |
| **TLS session state** | Keys, cipher state, record buffers (topic 05) | Tens of KB in some libraries unless tuned |
| **Application state** | User ID, subscriptions, a per-connection outbound queue | Depends on design; often a few KB |
| **A thread?** | Only in thread-per-connection servers | Typically 0.5–8 MB of stack each, plus scheduling cost |

A rough budget: if each idle connection costs ~20 KB all-in, 1 million connections need ~20 GB of
memory before any messages are sent. Tuned servers get far lower; careless ones far higher.

### Why thread-per-connection doesn't work

A classic server dedicates one thread to each connection, which blocks reading from its socket. For
short requests that's fine. For connections that sit idle for hours, it's ruinous: 100,000 connections
would mean 100,000 threads, gigabytes of stacks, and a scheduler juggling threads that are almost all
asleep.

**Event-driven** servers (the model behind Nginx, Node.js, Netty, Go's runtime network poller) use the
OS's readiness notification (`epoll` on Linux, `kqueue` on BSD/macOS) instead:

```
Thread-per-connection:              Event loop:
  thread 1: read(sock1) ...blocked    one thread: epoll_wait() → "sock 731 and sock 90211 are readable"
  thread 2: read(sock2) ...blocked                → handle those two, go back to waiting
  ... 100,000 sleeping threads        idle sockets cost no thread at all
```

An idle connection then costs only its memory, not a thread. This is the "C10K problem" (ten thousand
concurrent connections, a hard problem around 1999) solved; with tuning, single machines have held
millions of connections (the "C10M" era).

### Other limits that bite

- **Ephemeral ports and the 4-tuple.** A TCP connection is identified by (source IP, source port,
  destination IP, destination port). A **server** accepting on one port can hold connections from
  millions of clients, because the client IPs and ports differ. But a **proxy or load balancer**
  opening connections *to* a backend uses its own IP as the source, so it gets at most ~64,000 source
  ports per (proxy IP, backend IP, backend port). Load balancers fronting millions of WebSockets need
  several source IPs or several backend ports.
- **Heartbeat traffic.** Pinging 1 million connections every 30 s is ~33,000 small messages per second
  of pure overhead, plus timers to track them.
- **Slow consumers.** If the server writes faster than a client reads (a phone on a bad network), the
  per-connection outbound buffer grows. Without a cap, a few thousand slow clients can exhaust memory.
  Servers must bound each connection's queue and then **drop messages, coalesce them** (send only the
  latest price), or **disconnect** the slow client.
- **Capacity is measured in concurrent connections and message rate**, not just requests per second.

### Servers become stateful

This is the deepest cost. A REST server is stateless: any server can handle any request, and you can
kill one at any time. A connection server **holds live state** (the sockets themselves):

- **Deploys must drain.** You can't restart a server without dropping its clients. Deploys must close
  connections gradually, or tell clients to reconnect elsewhere, rather than all at once (connection
  draining; topic 09).
- **Load balancing is uneven.** Load balancers distribute *new* connections. A server that just started
  gets none of the connections that were made hours ago, so after a scale-out or a restart, old servers
  stay loaded and new ones sit idle until connections churn naturally. Servers sometimes deliberately
  close a fraction of connections to force rebalancing.
- **Failures cause reconnection storms.** If a server holding 100,000 connections dies, 100,000
  clients reconnect at once (section 9).

---

## 8. Fan-out to connected clients across many servers

With one server, sending a message to user B is easy: look up B's socket in a local map and write to
it. With 50 connection servers behind a load balancer, user A's message arrives at server 3, but user B
is connected to server 41. Something has to get it there.

```
                 ┌──────────────┐
  user A ───────▶│  server 3    │      who holds B?      ┌──────────────┐
                 └──────────────┘ ─────── ? ───────────▶ │  server 41   │──────▶ user B
                                                         └──────────────┘
```

There are two main approaches, plus a durable store behind both.

### Approach 1: a connection registry and directed routing

Keep a **registry** (a shared store such as Redis, or a coordination service; topic 31) mapping
`user → server` (or `user → [servers]` if a user has several devices). When a client connects, its
server registers it; when it disconnects, it deregisters.

```
1. B connects to server 41      → registry: B → server 41
2. A sends "hi B" to server 3
3. server 3 looks up B          → server 41
4. server 3 forwards to server 41 (direct RPC, or a per-server queue/channel)
5. server 41 writes to B's socket
```

- Precise: each message goes only where it's needed.
- The registry must stay accurate. If server 41 crashes, its entries are stale until they expire, so
  registrations are usually written with a TTL and refreshed by heartbeats (a lease; topic 31).

### Approach 2: pub/sub channels

Use a **pub/sub** backbone (Redis pub/sub, NATS, a message broker; topic 36). Each server subscribes to
the channels its connected clients care about: `user:B`, `room:42`, `ticker:ACME`.

```
1. B connects to server 41      → server 41 subscribes to channel "user:B"
2. A sends "hi B"               → server 3 publishes to "user:B"
3. broker delivers to every subscriber of "user:B" → server 41 → B's socket
```

- No explicit registry: the broker's subscription table *is* the registry.
- Group fan-out is natural: a message to `room:42` reaches every server with a member of that room, and
  each server writes to its local members. The broker sends one copy per **server**, not per client.

### The scale of fan-out

Fan-out is where cost explodes. One message to a chat room of 10 people is 10 socket writes. One score
update to a live event watched by 5 million people is 5 million socket writes, spread across perhaps 500
connection servers:

```
publisher ──1 msg──▶ broker ──500 msgs (one per server)──▶ servers ──5,000,000 writes──▶ clients
```

Techniques for very large fan-out:

- **Tiered fan-out**: the broker sends to a few hundred edge servers, each of which writes to its
  ~10,000 local clients. The tree shape keeps every node's work bounded.
- **Coalescing and sampling**: for rapidly changing values (prices, positions, view counts), send only
  the latest value every N ms instead of every change.
- **Pull for the long tail**: for huge audiences, push a small "something changed" notification and let
  clients fetch the content through cacheable HTTP (topic 14), rather than pushing the full payload.

### Delivery guarantees: what pub/sub doesn't give you

Plain pub/sub (Redis pub/sub, for instance) is **fire-and-forget**: if no subscriber is listening at
that moment, or the client is disconnected, the message is gone. A real system separates two jobs:

```
1. Durable store (database or log, topics 36–37):  every message, with a sequence number per user/room
2. Real-time path (registry or pub/sub):            best-effort fast delivery to whoever is online now
```

When a client reconnects, it sends its cursor ("last message I saw in room 42 was #1187"), and the
server fills the gap from the durable store. If the client is offline entirely, the system may also send
a mobile push notification (section 6). The real-time path can then be lossy without losing data.
Because the client may receive a message both via the live push and via catch-up, client-side
de-duplication by message ID is usually needed.

---

## 9. Failure: reconnection storms and draining

Picture a server holding 200,000 WebSocket connections that crashes (or is restarted carelessly):

```
t=0      server dies; 200,000 sockets drop
t=0-1s   200,000 clients reconnect immediately → land on the remaining servers
         → 200,000 TCP + TLS handshakes (CPU-heavy, topic 05)
         → 200,000 auth checks against the auth service
         → 200,000 catch-up queries against the message store
t=1-5s   remaining servers and the message store overload; some fail
         → their clients reconnect too → wider outage
```

This is a **thundering herd**, and it can turn the loss of one server into a cascading failure (topics
43 and 44). Defences:

- **Exponential backoff with random jitter** on the client: reconnect after a random delay, doubling
  the delay range on each failure, so reconnections spread out over seconds or minutes.
- **Spare capacity** and **admission control** on servers: reject excess handshakes quickly (with a
  "retry later" hint) rather than accepting everything and collapsing.
- **Cheap resync**: resume from a cursor instead of reloading full state.
- **Graceful draining on deploys**: stop accepting new connections on the old server, then close
  existing ones gradually over minutes (or send a "please reconnect" message), so the load moves in a
  trickle instead of a wave.

---

## 10. How it connects

- **TCP** (topic 03) keeps connections alive and is why a vanished peer can look connected; keep-alive
  pings exist because of it.
- **TLS** (topic 05) makes each new connection expensive, which is why reconnection storms are CPU
  storms.
- **HTTP/2** (topic 06) makes SSE cheap in browsers (one stream, not one connection) and is how gRPC
  streaming (topic 07) delivers server push between services.
- **Statelessness** (topics 01, 07) is what persistent connections give up; **load balancers** (topic
  09) distribute new connections, need connection draining, and need L7 support for WebSocket upgrades.
- **Little's Law and queueing** (topic 02) explain slow consumers: bounded queues or unbounded memory.
- **Pub/sub and durable logs** (topics 36–37) are the backbone of cross-server fan-out; **idempotency**
  (topic 38) is needed for webhooks and duplicate deliveries.
- **Backoff with jitter** and **load shedding** (topics 43–44) are what keep reconnection storms from
  becoming outages.
- **Leases and registries** (topic 31) keep the user → server map from going stale.

## Common misconceptions

- **"Real-time means WebSockets."** → For server-to-client updates, SSE is often simpler and keeps
  HTTP's infrastructure. WebSockets earn their cost when the client also sends a lot.
- **"Long polling is just slow polling."** → It's push with HTTP: the server answers the moment an
  event happens. Its cost is holding a pending request per client, not latency.
- **"An idle connection is free."** → It costs a file descriptor, socket buffers, TLS state and
  application memory; with thread-per-connection it also costs a thread. Millions of idle connections
  is a real capacity problem.
- **"Pub/sub guarantees delivery."** → Plain pub/sub delivers only to whoever is subscribed right now.
  Offline clients need a durable store and a cursor to catch up.
- **"WebSocket masking is for security against eavesdroppers."** → It protects intermediary proxies
  from being confused into caching attacker-crafted bytes. Confidentiality comes from TLS.
- **"Adding servers spreads WebSocket load immediately."** → Load balancers only place *new*
  connections; existing ones stay where they are until they churn or are deliberately moved.
- **"If a server dies, its clients just reconnect."** → All at once, unless clients back off with
  jitter. The reconnection storm can take down the remaining servers.

## Further reading (supplements)

- Ilya Grigorik, *High Performance Browser Networking* (free online): chapters "WebSocket" and
  "XMLHttpRequest" (which covers polling, long polling and SSE)
- RFC 6455 (The WebSocket Protocol), sections 1 and 10 on the handshake and masking rationale
- WHATWG HTML Living Standard, "Server-sent events" section; MDN pages on `EventSource` and the
  WebSocket API
- Dan Kegel, "The C10K problem" (classic essay on serving many concurrent connections)
- Phoenix Framework team, "The Road to 2 Million Websocket Connections in Phoenix" (blog post)
- Martin Kleppmann, *Designing Data-Intensive Applications*, chapter 11 (messaging systems,
  "Transmitting Event Streams")

---

## Questions

1. One million clients short-poll every 10 seconds. How many requests per second is that? How would
   long polling change both the request rate and what the server has to hold?

2. In long polling, events can be lost between one response and the next request. Why, and how is it
   prevented? How does SSE handle the same problem?

3. A dashboard needs live metric updates from the server; the user only occasionally changes a filter.
   SSE or WebSockets? Justify your choice.

4. You deploy an SSE endpoint behind a reverse proxy, and clients receive events in big bursts every
   minute or so instead of one at a time. What's the likely cause?

5. Why are WebSocket frames from the client to the server masked?

6. A WebSocket API authenticates users with a token during the upgrade. The token is revoked an hour
   later. What happens, and what should the server do about it?

7. A team's chat server uses one thread per connection and falls over at ~20,000 users even though CPU
   is mostly idle. Explain why, and what architecture fixes it.

8. Estimate the memory needed for 2 million idle WebSocket connections if each costs about 30 KB in
   socket buffers, TLS state and application state. Name two other limits besides memory you would
   need to raise or design around.

9. A load balancer terminates client WebSockets and opens one upstream connection per client to a
   single backend IP and port. It stalls at about 60,000 connections. Why?

10. A server streams price updates to clients. A few hundred clients are on terrible mobile networks.
    Over an hour, server memory climbs steadily. What's happening, and what are the options?

11. A server holding 200,000 WebSocket connections crashes. Describe what happens next and two things
    that keep it from turning into a wider outage.

12. In a chat system with 20 WebSocket servers behind a load balancer, user A (on server 3) sends a
    message to user B (on server 14). What does the system need so the message gets delivered? What if
    B is offline?

13. Compare a connection registry with directed routing against pub/sub channels for cross-server
    fan-out. When does each fit better?

14. A live sports app pushes score updates to 5 million concurrent viewers. Why is naive per-client
    publishing a problem, and how would you structure the fan-out?

15. Ops scale a WebSocket fleet from 10 to 20 servers during a traffic spike. An hour later the original
    10 are still overloaded. Why? What can be done?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** 1,000,000 / 10 = **100,000 requests per second**, almost all returning nothing, each paying for
headers, TLS record processing, auth, and maybe a database check. With long polling, the request rate
drops to roughly the **rate of actual events** plus one re-request per client per timeout period (e.g.
1,000,000 / 30 s ≈ 33,000 req/s at most if nothing ever happens). But the server now **holds up to
1 million open requests at once**, so it must handle huge numbers of idle connections cheaply (an
event-driven server rather than one thread per request).

**2.** After the server sends a response, there's a window (network transit plus the client sending a
new request) in which no request is waiting. Events in that window have no open request to be delivered
on. Prevention: the client sends a **cursor**, the ID of the last event it received, with each request,
and the server keeps recent events so it can return everything after that cursor. SSE builds this in:
each event can carry an `id:`, and on reconnect the browser automatically sends a `Last-Event-ID` header
so the server can resume from there.

**3.** **SSE.** Data flows almost entirely server → client. Occasional filter changes can be ordinary
HTTP requests (or reconnecting the stream with new parameters). SSE gives automatic reconnection with
resume, works with existing HTTP auth, proxies, load balancers and HTTP/2 multiplexing, and needs no
custom message protocol. WebSockets would add complexity (custom framing protocol, manual reconnection,
non-HTTP infrastructure) without a benefit.

**4.** The proxy is **buffering the response**: it collects the body until its buffer fills (or a flush
timer fires) before forwarding, which is a sensible default for normal responses but breaks streaming.
Disable response buffering for the SSE route (most proxies support this per route or via a response
header), and make sure compression isn't buffering either. Also check idle timeouts: send periodic
comment lines so the proxy doesn't cut the connection.

**5.** To protect **intermediaries**, not the endpoints. Without masking, a malicious page could send
WebSocket payloads crafted to look like HTTP requests and responses. Some older transparent proxies that
didn't understand WebSockets would parse those bytes as HTTP and could be tricked into caching
attacker-chosen content for a real URL (cache poisoning). Masking each client frame with a random key
makes the bytes on the wire unpredictable to the page's script, so it can't craft them.

**6.** Nothing, by default: authentication happened once at the upgrade, and the socket stays open, so
the revoked user keeps receiving messages (and sending them) until the connection drops. The server
needs to bound this: track the token's expiry per connection and close (or require re-authentication
over the socket) when it expires; subscribe to revocation events and close affected connections; and
check authorization per message for sensitive actions rather than trusting the connection forever.

**7.** Each connection costs a **thread**, even when idle. 20,000 threads mean gigabytes of stack
memory and a scheduler managing thousands of mostly sleeping threads; memory or thread limits run out
long before CPU does. The fix is an **event-driven** (non-blocking) server: a small number of threads
use `epoll`/`kqueue` to learn which sockets are ready and handle only those, so an idle connection costs
only its memory, not a thread.

**8.** 2,000,000 × 30 KB ≈ **60 GB**, so either a big machine or several servers (and tuning buffers and
TLS state can cut it a lot). Other limits: **file descriptors** (per-process and system-wide limits,
default ~1,024 per process); **heartbeat load** (pinging 2 million connections every 30 s is ~67,000
messages per second); **ephemeral ports** if a proxy connects upstream per client; kernel connection
tracking tables in firewalls/NAT; and per-connection outbound queue limits for slow consumers.

**9.** Each upstream connection from the load balancer to the backend is identified by (LB IP, source
port, backend IP, backend port). With one LB IP and one backend address, only the **source port**
varies, and there are roughly 64,000 (in practice, the configured ephemeral range, often ~28,000 to
~60,000). Fixes: give the load balancer several source IPs, have the backend listen on several ports, or
add more backend IPs; or multiplex many clients over fewer upstream connections.

**10.** Those clients **read slower than the server writes**, so their per-connection outbound buffers
keep growing: an unbounded queue (topic 02's queueing at a bottleneck). Options: **cap each connection's
queue**; when it's full, **coalesce** (keep only the latest price per symbol, which is all the client
needs), **drop** older messages, or **disconnect** the slow client so it reconnects and resyncs. Watch
per-connection buffer size as a metric.

**11.** All 200,000 clients lose their connections at the same moment and immediately try to reconnect,
landing on the remaining servers. This **reconnection storm** brings a burst of TCP and TLS handshakes,
authentication and state re-sync (each client fetches missed messages), on top of the extra 200,000
connections. It can overload the remaining servers and shared dependencies, which then fail too (a
cascading failure). Protections: (1) clients reconnect with **exponential backoff and random jitter**
so reconnections spread out; (2) **spare capacity** and **admission control / load shedding** so servers
reject excess handshakes quickly rather than collapse; also cheap resync from a cursor.

**12.** (1) A way to find **which server holds B's connection**: a registry (user → server, with TTLs
refreshed by heartbeats) or pub/sub subscriptions (server 14 subscribed to `user:B` when B connected);
(2) a **messaging path between servers**: server 3 forwards or publishes, and server 14 writes it to B's
socket. If B is offline, the message must already be in a **durable store** with a sequence number, so
B catches up on reconnect by sending its cursor; the system may also send a mobile push notification.
The real-time path can be best-effort because the durable store is the source of truth.

**13.** **Registry + directed routing**: each message goes exactly to the server(s) holding the
recipient; efficient for one-to-one messaging; but the registry must be kept accurate (stale entries
after crashes, so leases/TTLs). **Pub/sub channels**: no separate registry, the broker's subscriptions
play that role; naturally handles group fan-out (rooms, topics, tickers), sending one copy per
interested server; but the broker becomes a central component carrying all traffic, and plain pub/sub
is fire-and-forget. Directed routing fits mostly one-to-one traffic; pub/sub fits rooms, topics and
broadcast-style traffic.

**14.** Publishing one message per client means 5 million publishes per update through the broker, all
at once: the broker and network become the bottleneck, and each update takes a long time to reach the
last viewer. Instead, publish **once** to a channel; each of the (say) 500 connection servers subscribes
to it, receives one copy, and writes to its ~10,000 local clients (**tiered fan-out**). Further: coalesce
rapid updates (send the latest score every few hundred ms), keep payloads tiny, and for large content
push a "changed" signal and let clients fetch the content via a CDN-cacheable URL.

**15.** Load balancers only place **new** connections. The connections that existed before the
scale-out stay on the original 10 servers until clients disconnect, which for WebSockets can take hours.
New servers receive only the trickle of new connections. Fixes: have overloaded servers **gradually
close** a fraction of their connections (with a reconnect hint and client-side jitter) so they
redistribute; cap connections per server so new ones go elsewhere; or set a maximum connection
lifetime so connections churn and rebalance naturally.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Why is server push a problem for plain HTTP?
**A:** HTTP is client-initiated: the server can only answer requests.

**Q:** Short polling: main cost?
**A:** Mostly empty responses (N clients / interval requests per second) and latency up to the interval.

**Q:** How does long polling work?
**A:** The server holds the request until there's data or a timeout; the client immediately re-requests, sending a cursor.

**Q:** What is the long-polling "gap"?
**A:** Between a response and the next request no request is pending; events then are lost unless the client sends a cursor and the server keeps recent history.

**Q:** What is SSE?
**A:** One long HTTP response with `text/event-stream`; the server writes events into it. Server → client only, text, built-in reconnect.

**Q:** How does SSE resume after a disconnect?
**A:** The browser reconnects automatically and sends `Last-Event-ID`; the server replays from there.

**Q:** Why can a proxy break SSE?
**A:** Response buffering holds events back; idle timeouts cut quiet streams. Disable buffering, send comment keep-alives.

**Q:** How does a WebSocket start?
**A:** HTTP/1.1 GET with `Upgrade: websocket` and `Sec-WebSocket-Key`; server answers `101 Switching Protocols`; then the connection carries WebSocket frames.

**Q:** Why are client → server WebSocket frames masked?
**A:** To stop malicious pages crafting bytes that confuse intermediary proxies (cache poisoning).

**Q:** What do you give up after a WebSocket upgrade?
**A:** HTTP semantics: status codes, caching, per-request auth, built-in reconnection. You design your own message protocol.

**Q:** SSE or WebSockets for server → client notifications?
**A:** Usually SSE: simpler, plain HTTP, auto-reconnect. WebSockets when the client also sends at a high rate.

**Q:** What is a webhook?
**A:** Server-to-server push: you POST to a URL the receiver registered. Needs retries, so receivers must be idempotent; sign deliveries.

**Q:** What does an idle open connection cost a server?
**A:** A file descriptor, kernel socket buffers, TLS state, application state, and a thread if the server is thread-per-connection.

**Q:** Why do connection servers use event loops?
**A:** `epoll`/`kqueue` let a few threads serve many sockets; idle connections cost memory but no thread.

**Q:** What is the C10K problem?
**A:** Serving 10,000+ concurrent connections on one machine; solved by event-driven, non-blocking I/O.

**Q:** Why can a proxy run out of ports with many upstream connections?
**A:** Connections to one backend IP:port differ only by source port; ~64K per source IP.

**Q:** How do you handle slow consumers?
**A:** Bound each connection's outbound queue; coalesce, drop, or disconnect when it fills.

**Q:** Why do persistent connections make servers stateful?
**A:** Each server holds specific live sockets; it can't be killed or replaced without dropping those clients.

**Q:** Why doesn't scaling out rebalance WebSocket load?
**A:** Load balancers only place new connections; old ones stay until they churn or are deliberately closed.

**Q:** What is a reconnection storm?
**A:** Many clients reconnecting at once after a server failure, overwhelming the rest with handshakes, auth and resync.

**Q:** Defences against reconnection storms?
**A:** Client backoff with jitter, spare capacity, admission control, cheap cursor-based resync, gradual draining on deploys.

**Q:** Two ways to route a message to a client on another server?
**A:** A user → server registry with directed forwarding, or pub/sub channels each server subscribes to for its clients.

**Q:** Why does plain pub/sub need a durable store beside it?
**A:** Pub/sub only reaches current subscribers; offline or reconnecting clients catch up from the store via a cursor.

**Q:** How do you fan out to millions of viewers?
**A:** Publish once; each server gets one copy and writes to its local clients (tiered fan-out); coalesce updates; pull large content via CDN.

**Q:** How do mobile apps get pushes in the background?
**A:** Through the platform push service (APNs/FCM), which keeps one connection per device for all apps.
