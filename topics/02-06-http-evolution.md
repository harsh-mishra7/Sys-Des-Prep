# 06 — HTTP: 1.1 → 2 → 3

> Part 2 — Networking & Communication · Prerequisites: topics 03, 04, 05

Topics 03–05 covered how bytes move: IP delivers packets, TCP turns them into a reliable stream, TLS
secures it, DNS finds the server. But a byte stream has no meaning of its own. Two programs still have
to agree on **what the bytes say**: where one message ends and the next begins, what a request is
asking for, what counts as success. HTTP is that agreement for most of the internet, and increasingly
for service-to-service traffic too (gRPC runs on HTTP/2, topic 07).

HTTP's *meaning* (methods, URLs, status codes, headers) has barely changed in 25 years. What changed,
twice, is **how messages are carried over the network**. Each new version exists to fix a specific
performance problem of the one before, and each fix ran into a limit of the layer underneath. Following
that chain is the best way to understand why HTTP/3 ended up on UDP.

By the end of this topic you should be able to:

1. Say what every application protocol must decide, and how HTTP/1.1 decides it.
2. Explain HTTP/1.1's persistent connections, why pipelining failed, and the workarounds the web
   adopted instead.
3. Explain HTTP/2's binary framing, streams, multiplexing and HPACK, and what HTTP/2 could not fix.
4. Explain how QUIC and HTTP/3 remove TCP head-of-line blocking, cut handshake round trips, and survive
   network changes, and what they cost.
5. Choose sensibly between versions for browsers, mobile clients and internal services.

---

## 1. What an application protocol must decide

Every application protocol, from HTTP to a database's wire protocol, answers the same questions:

| Question | Why it's needed | HTTP/1.1's answer |
|---|---|---|
| **Framing**: where does a message end? | TCP is a byte stream with no boundaries (topic 03) | Headers end at a blank line; body length from `Content-Length` or chunked encoding |
| **Semantics**: what is being asked? | The server must know what to do | Method + path + headers |
| **Outcome**: what happened? | The client must know whether to retry, fix, or give up | Status code |
| **Multiplexing**: can several conversations share a connection? | Connections are expensive (topics 03, 05) | Not really (one request at a time) |
| **Metadata**: auth, content type, caching hints | Cross-cutting concerns | Headers |

The history of HTTP versions is almost entirely the history of the **framing** and **multiplexing**
rows. The semantics and outcome rows, which matter for API design, are topic 07's subject.

---

## 2. HTTP/1.1: simple, text-based, one at a time

### What it looks like

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

Being text made HTTP easy to debug (you can type a request into a raw TCP connection) and easy to
implement, which is a big part of why it won.

**Framing** works like this: the receiver reads lines until the blank line, then knows how many body
bytes follow from `Content-Length`. When the sender doesn't know the length in advance (a response
being generated on the fly), it uses **chunked transfer encoding**: a series of chunks each prefixed
with its size, ending with a zero-length chunk. Text framing has a dark side: if two parties (say, a
proxy and a backend) disagree on where a message ends, for example when a request carries both
`Content-Length` and chunked encoding, an attacker can hide a second request inside the first. That's
**request smuggling**, and it's one motivation for the stricter binary framing of HTTP/2.

### Persistent connections

HTTP/1.0 opened a new TCP connection for every request and closed it after the response. With what
topic 03 taught about handshakes, slow start and TIME_WAIT, you can see how wasteful that is: a page
with 50 resources paid for 50 handshakes and 50 cold congestion windows.

HTTP/1.1 (1997) made two changes that still matter:

- **Persistent connections (keep-alive) by default.** One TCP + TLS connection carries many requests
  in sequence, avoiding repeated handshakes and keeping the congestion window warm (topic 03). Either
  side can end it with `Connection: close` or an idle timeout.
- **The `Host` header** is mandatory, so many websites can share one IP address (virtual hosting), the
  HTTP-level counterpart of TLS's SNI (topic 05).

### The core limitation: one outstanding request per connection

On one HTTP/1.1 connection, the client sends a request and **must wait for the full response** before
the next request can use the connection. If a page needs 80 resources, they queue up, each paying at
least one round trip:

```
one connection:  [req 1 ──── resp 1][req 2 ─ resp 2][req 3 ──────── resp 3][req 4 ...
                  ◀── 1 RTT+ ──────▶◀── 1 RTT+ ────▶
```

### Pipelining, and why it failed

HTTP/1.1 tried to fix this with **pipelining**: send several requests back-to-back without waiting.
But responses must still come back **in the order the requests were sent**, because an HTTP/1.1
response carries nothing that says which request it answers. The only way to match them up is order.

```
client sends:     GET /big-report   GET /logo.png   GET /style.css
server prepares:  report: 2 s       logo: 5 ms      css: 5 ms   (logo and css ready almost at once)
server must send: [report ......................][logo][css]
                                                  ↑ ready long ago, stuck behind the report
```

So one slow response blocks every response behind it, even ones that are ready. That's
**head-of-line blocking at the HTTP layer**. Worse, if the connection dies mid-way, the client can't
tell which of the pipelined requests were processed, so it can only safely replay idempotent ones
(topic 07). Combined with buggy proxies that mishandled pipelined requests, it was bad enough that
browsers shipped with pipelining disabled. Fixing it properly needed an identifier on every message,
which meant changing the wire format. That's HTTP/2.

### The workarounds, and what they cost

Since one connection could only do one thing at a time, everyone opened more connections:

- Browsers open **about 6 connections per origin** in parallel.
- Sites used **domain sharding** (`img1.example.com`, `img2.example.com`, ...) to get 6 connections per
  hostname, multiplied.
- Developers **concatenated** JS/CSS files and combined images into **sprites**, and **inlined** small
  resources into the HTML, to cut the number of requests.

All of these are hacks around the protocol. More connections mean more DNS lookups, TCP and TLS
handshakes, more slow-start ramps (each connection starts cold), more server memory, and connections
competing with each other for bandwidth instead of sharing one well-tuned congestion window.
Concatenation hurts caching: change one small file and the whole bundle must be downloaded again.

### Header overhead

Headers are repeated in full, as text, on every request. Cookies, user agents and auth tokens can add
500 bytes to 2 KB per request, often more than the body of a small API call. On a slow mobile uplink,
sending 80 requests' worth of identical headers takes real time, and for small requests it may not fit
in the initial congestion window.

---

## 3. HTTP/2: multiplexing over one connection

HTTP/2 (2015, derived from Google's SPDY experiment) kept HTTP's **semantics** exactly the same
(methods, status codes, headers, URLs) and replaced the **wire format**. Application code barely
notices; the transport underneath is completely different.

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

Binary framing is faster and unambiguous to parse (every frame states its length; there's no
"where does this end?" guesswork to smuggle requests through). But the important field is the
**stream ID**.

### Streams and multiplexing

A **stream** is one request/response exchange, with its own ID. Every frame says which stream it
belongs to, so frames from many streams can be **interleaved** on one connection and reassembled at the
other end:

```
HTTP/1.1, one connection:     [req A ──── resp A][req B ─ resp B][req C ── resp C]

HTTP/2, one connection:       A1 B1 C1 A2 C2 B2 A3 C3 ...   (frames from streams A, B, C interleaved)
```

This is the big win. A slow response on stream A no longer blocks stream B; B's frames simply flow
between A's, and responses can complete in any order. Consequences:

- **One connection per origin** is enough. One handshake, one slow-start ramp, and a congestion window
  that grows large because all traffic shares it.
- Domain sharding, concatenation and sprites become unnecessary, and sometimes counterproductive
  (sharding defeats the single warm connection; bundles hurt caching when small requests are cheap).
- Hundreds of concurrent requests can be in flight. The server advertises a limit,
  `SETTINGS_MAX_CONCURRENT_STREAMS`, often 100 to 250.

HTTP/2 also has **per-stream flow control** (like TCP's receive window from topic 03, but per stream),
so one large download can't fill the receiver's buffers and starve the other streams. Clients could
also signal **priorities** between streams (load the CSS before the images), though the original
scheme was complex and rarely implemented well, and has since been replaced by a simpler one.

Two operational consequences of multiplexing:

- **Cheap streams can be abused.** Because a client can open and cancel streams very quickly, servers
  must limit how much work a single connection can trigger. The 2023 "Rapid Reset" attack opened and
  immediately cancelled streams in a loop, making servers start work for requests nobody waited for.
- **One connection carries everything**, so load balancing per connection (L4, topic 09) balances
  badly: all of a client's requests go to one backend. HTTP/2-aware (L7) balancers spread individual
  streams instead. This shows up most with gRPC (topic 07).

### HPACK header compression

HTTP/2 compresses headers with **HPACK**:

- A **static table** of 61 common header entries (`:method: GET`, `:status: 200`, ...) referenced by
  index.
- A **dynamic table** that both sides build up as the connection is used. The second time a client
  sends the same `Authorization` header, it sends a small index instead of the full value.
- **Huffman coding** for literal strings.

Why a table shared across the whole connection, rather than compressing each request on its own? Most
headers repeat almost identically on every request on a connection (cookies, tokens, user agent). Each
request on its own has little internal redundancy; the redundancy is *between* requests. Repeated
headers shrink to a byte or two.

HPACK was designed specifically to avoid the compression side-channel attacks (like **CRIME**) that hit
an earlier approach of gzip-compressing headers: with general-purpose compression, an attacker who can
inject text into requests and observe their compressed size can guess a secret cookie byte by byte.

Note the hidden assumption: both sides must process header blocks **in the same order** to keep their
dynamic tables in sync. TCP's in-order delivery guarantees that. Hold on to that thought for HTTP/3.

### Server push: an idea that didn't work out

HTTP/2 let a server send a resource the client hadn't asked for yet ("you requested `index.html`;
you'll need `app.css`, here it is"). In practice servers often pushed things the client already had
cached, wasting bandwidth, and it was hard to get right. Chrome removed support in 2022. The lesson:
the server rarely knows what the client's cache contains. Its replacement is the much simpler
`103 Early Hints` response, which just *tells* the client what it'll need, and lets the client decide.
(Don't confuse HTTP/2 server push with the general idea of servers pushing events to clients, which is
topic 08.)

### Negotiation

How does a client know a server speaks HTTP/2? During the TLS handshake, using **ALPN** (topic 05): the
client lists `h2, http/1.1`, the server picks one. No extra round trip. Browsers only use HTTP/2 over
TLS; a cleartext variant (`h2c`) exists and is sometimes used between internal services.

### The problem HTTP/2 couldn't fix: TCP head-of-line blocking

HTTP/2 removed head-of-line blocking at the *HTTP* layer, but everything still runs over **one TCP
connection**, and TCP delivers bytes strictly in order (topic 03). If one packet is lost:

```
Packets on the wire:  [A1][B1][C1][A2][B2]...
                             ✗ lost

TCP receiver has A1, C1, A2, B2 in its buffer, but cannot hand ANY of them to HTTP/2
until B1 is retransmitted. Streams A and C are stalled by a loss that only affected B.
```

TCP has no idea there are independent streams inside the byte stream. So with HTTP/2:

- On a clean network (inside a data centre), it's a clear win.
- On a lossy network (mobile, congested Wi-Fi), **one lost packet stalls every stream**, and the single
  connection's congestion window is cut on every loss, slowing everything. With 1 to 2% packet loss,
  HTTP/1.1 with 6 independent connections can actually outperform HTTP/2 on one connection, because a
  loss on one of the 6 stalls only that one and halves only one sixth of the total sending rate.

You can't fix this inside TCP: in-order delivery of one byte stream is TCP's whole contract, and TCP
lives in operating system kernels and in middleboxes across the internet that take decades to change.
Fixing it meant leaving TCP.

---

## 4. HTTP/3 and QUIC: moving streams into the transport

**QUIC** is a transport protocol, originally built at Google and standardised by the IETF in 2021.
**HTTP/3** (2022) is HTTP mapped onto QUIC. QUIC runs over **UDP**, and this is exactly the "UDP as a
building block" case from topic 03: UDP provides nothing except ports and a checksum, so QUIC can
implement its own reliability, ordering and congestion control in user space, designed the way HTTP
needs.

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

### Why not just fix TCP?

Because TCP is effectively frozen. It's implemented in every operating system kernel, so a change takes
years to reach most devices. And middleboxes (firewalls, NATs, load balancers) inspect TCP headers and
often drop or mangle packets carrying options they don't recognise. Extensions like TCP Fast Open
(topic 03) struggled for exactly this reason. This is called **ossification**: a protocol so widely
inspected it can no longer evolve. UDP, by contrast, passes through almost every network, and whatever
runs on top of it is invisible to middleboxes if it's encrypted.

### What QUIC fixes

**1. Independent streams in the transport.** QUIC knows about streams. Each stream is ordered
internally, but streams are independent of each other. A lost packet carrying stream B's data stalls
**only stream B**; A and C carry on.

```
Packets:  [A1][B1][C1][A2][B2]
               ✗ lost
QUIC delivers A1, A2 to stream A and C1 to stream C immediately.
Only stream B waits for B1's retransmission.
```

This removes the last head-of-line blocking problem between requests (blocking *within* one stream
remains, which is correct: a single response still needs its bytes in order).

**2. Fewer handshake round trips.** TCP + TLS 1.3 needs 1 RTT for TCP, then 1 RTT for TLS, before the
first request. QUIC carries the TLS 1.3 handshake inside its own connection setup, so transport and
crypto are negotiated together:

| Setup | Round trips before the first request |
|---|---|
| TCP + TLS 1.2 | 3 |
| TCP + TLS 1.3 | 2 |
| QUIC, new connection | 1 |
| QUIC, resumed connection (0-RTT) | 0 |

0-RTT has the same replay caveat as TLS 1.3 0-RTT (topic 05): early data can be replayed by an
attacker, so it should only carry requests that are safe to repeat, which in practice means idempotent
ones (topic 07).

**3. Connection migration.** A TCP connection is identified by its 4-tuple (topic 03). When a phone
switches from Wi-Fi to cellular, its IP changes, the 4-tuple changes, and every connection dies. Each
must be noticed as dead (often only after a timeout), re-established with new handshakes and a cold
congestion window, and in-flight requests retried. QUIC identifies connections by **connection IDs**
chosen by the endpoints, so the client keeps sending from its new address with the same connection
ID, the server validates the new path, and the connection continues.

**4. Encrypted transport headers.** In TCP, sequence numbers, flags and options are visible to every
middlebox on the path, and middleboxes started depending on them; that's how TCP ossified. QUIC
encrypts almost all of its own headers (only a few bits and the connection ID are visible), so
middleboxes can't depend on them, and QUIC can keep evolving.

**5. Better loss recovery.** In TCP, a retransmitted segment has the same sequence number as the
original, so when an ACK arrives the sender can't tell which copy it acknowledges, which muddies RTT
measurement. QUIC gives every packet a new, strictly increasing packet number, even retransmissions, so
loss detection and RTT estimation are unambiguous.

**6. Evolves in user space.** QUIC ships inside applications and libraries (browsers, server software),
not the OS kernel. New congestion control or loss recovery can roll out with an app update.

**QPACK instead of HPACK.** Recall HPACK's hidden assumption: header blocks are processed in order.
QUIC streams arrive independently, so if header compression updates travelled on request streams, a
lost packet on one stream could block header decoding on others, reintroducing head-of-line blocking.
**QPACK** sends dynamic-table updates on dedicated streams and lets each request say which table state
it depends on, so out-of-order streams don't block one another (or block only when they genuinely
depend on an update that hasn't arrived).

### What QUIC costs

- **CPU.** TCP processing has decades of kernel and network-card optimisation (segmentation offload,
  checksum offload). QUIC does encryption and packet handling per packet in user space, and has
  historically cost noticeably more CPU per byte. This is improving but is still real at large scale.
- **UDP gets blocked or throttled.** Some corporate networks and firewalls block UDP other than DNS.
  Clients must be able to fall back to HTTP/2 over TCP, so every HTTP/3 deployment still runs HTTP/2.
- **Discovery.** A client can't know in advance that a server supports HTTP/3. Usually the first
  connection uses HTTP/2, and the server advertises HTTP/3 with an `Alt-Svc` response header (or a DNS
  `HTTPS` record, topic 04, which can be seen before connecting). Later connections try QUIC, typically
  racing it against TCP and using whichever succeeds first.
- **Less visibility for operators.** Encrypted headers mean network tools can't easily see loss,
  retransmissions or RTT the way they can for TCP. Debugging moves to endpoint logs (e.g. qlog).
- **Load balancing and connection IDs.** Because connections can change address, a load balancer can't
  route by 4-tuple; QUIC-aware balancers route by connection ID, with servers encoding routing
  information into the IDs they issue.

### Where each version wins

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP | TCP | QUIC over UDP |
| Requests per connection at once | 1 | Many (multiplexed) | Many (multiplexed) |
| HOL blocking | HTTP layer and TCP | TCP only | Only within a stream |
| Framing | Text | Binary frames | Binary frames (QUIC streams) |
| Header compression | None | HPACK | QPACK |
| Handshake RTTs (new, TLS 1.3) | 2 | 2 | 1 |
| Survives network change | No | No | Yes (connection IDs) |
| Best fit | Simple clients, debugging, legacy | Most traffic, especially inside data centres | Mobile and lossy, high-latency networks |

Inside a data centre, with sub-millisecond RTTs, almost no loss, fixed addresses and long-lived pooled
connections, HTTP/3's advantages mostly disappear, and HTTP/2 (or gRPC, which uses it) is the common
choice for service-to-service traffic. HTTP/3's gains are largest at the **edge**, between users on
unreliable networks and the nearest server, which is exactly where CDNs (topic 14) have deployed it.

### What each version fixed: the chain

```
HTTP/1.0  one request per connection        → handshake + slow start on every request
   │ fix: persistent connections
HTTP/1.1  one request at a time per conn    → HTTP-layer HOL blocking; 6 conns, sharding, bundling
   │ fix: stream IDs + multiplexing (binary framing)
HTTP/2    many streams over one TCP conn    → TCP-layer HOL blocking; 4-tuple-bound connections
   │ fix: move streams into the transport, on UDP
HTTP/3    independent streams in QUIC       → costs CPU, needs UDP, needs TCP fallback
```

Each step fixed the bottleneck of its predecessor and exposed the next one down the stack. It's a good
general lesson: optimising one layer often just reveals the limit of the layer beneath it.

---

## 5. How it connects

- **Topic 03**: persistent connections and multiplexing exist because of handshake, slow-start and
  TIME_WAIT costs; TCP head-of-line blocking is why HTTP/3 left TCP; QUIC is the canonical example of
  building custom guarantees on UDP.
- **Topic 04**: DNS `HTTPS` records advertise HTTP/3 support; connection reuse means DNS changes don't
  reach open HTTP/2 connections.
- **Topic 05**: ALPN negotiates HTTP/2 inside the TLS handshake; QUIC embeds TLS 1.3; 0-RTT's replay
  risk carries over.
- **Topic 07**: the semantics HTTP kept constant across versions (methods, safety, idempotency, status
  codes) are the substance of REST; gRPC is built on HTTP/2 streams.
- **Topic 08**: real-time techniques (long polling, SSE, WebSockets) are built on these versions and
  inherit their connection behaviour.
- **Topics 09, 10**: L7 balancers and proxies must understand HTTP/2 streams to balance them;
  multiplexed long-lived connections defeat connection-level (L4) balancing.
- **Topic 14**: CDNs terminate HTTP/3 and TLS at the edge, close to users, and talk HTTP/1.1 or HTTP/2
  to the origin.

## Common misconceptions

- **"HTTP/2 changed how HTTP works."** → It changed the wire format only. Methods, status codes,
  headers and URLs are identical; applications mostly don't notice.
- **"HTTP/2 removed head-of-line blocking."** → It removed it at the HTTP layer. TCP's head-of-line
  blocking remains, and on lossy networks it can make HTTP/2 slower than HTTP/1.1.
- **"HTTP/1.1 can't do several requests at once."** → It can, with several connections (browsers use
  about 6), or in theory with pipelining. Each connection does one at a time.
- **"HTTP/3 is always faster."** → It wins on lossy, high-latency, changing networks. Inside a data
  centre it gains little and costs more CPU.
- **"QUIC is unreliable because it's on UDP."** → QUIC implements its own reliability, ordering and
  congestion control; UDP is just the envelope that gets through middleboxes.
- **"Domain sharding and bundling are best practices."** → They were workarounds for HTTP/1.1. Under
  HTTP/2 and HTTP/3 they often hurt.
- **"HTTP/2 server push is how servers send real-time events."** → Server push was for pre-sending
  resources and has been removed from browsers. Real-time push is topic 08.
- **"Upgrading to HTTP/3 means dropping HTTP/2."** → HTTP/3 needs HTTP/2 (or 1.1) as a fallback for
  networks that block UDP and for discovery.

## Further reading (supplements)

- Ilya Grigorik, *High Performance Browser Networking*: chapters on HTTP/1.x and HTTP/2
- Daniel Stenberg, *HTTP/3 explained* (free online): a short, clear walk through QUIC and HTTP/3
- Robin Marx's articles on HTTP/3 and QUIC (Smashing Magazine series "HTTP/3: From A to Z") and his
  talks on HTTP/2 prioritisation
- Langley et al., "The QUIC Transport Protocol: Design and Internet-Scale Deployment" (SIGCOMM 2017)
- Cloudflare blog: posts on HTTP/3, QUIC and the HTTP/2 Rapid Reset attack
- RFC 9112 (HTTP/1.1), RFC 9113 (HTTP/2), RFC 7541 (HPACK), RFC 9000 (QUIC), RFC 9114 (HTTP/3),
  RFC 9204 (QPACK)

---

## Questions

1. HTTP/1.1 supports pipelining, yet browsers disabled it. What problem did pipelining leave unsolved,
   and why couldn't the protocol fix it without changing the wire format?

2. Why did HTTP/1.0's one-connection-per-request model hurt so much, in terms of what topic 03 taught
   about TCP?

3. Before HTTP/2, sites used domain sharding, file concatenation and image sprites. What were these
   working around? Why can they hurt performance under HTTP/2?

4. Explain what a "stream" is in HTTP/2 and how multiplexing works on one connection. Why is one
   connection per origin enough?

5. Why does HTTP/2 compress headers with a dynamic table that both sides maintain, rather than
   compressing each request's headers independently? Why not just gzip them?

6. On a mobile network with 2% packet loss, a page loads *slower* over HTTP/2 than over HTTP/1.1.
   Explain how this can happen.

7. Why was HTTP/3 built on UDP instead of fixing TCP?

8. A user on a train walks from Wi-Fi coverage onto cellular in the middle of a download. What happens
   with HTTP/2, and what happens with HTTP/3? What QUIC feature makes the difference?

9. Count the round trips before the first request byte reaches the server for: TCP + TLS 1.2,
   TCP + TLS 1.3, new QUIC, resumed QUIC with 0-RTT. Why should 0-RTT be limited to idempotent requests?

10. Why did HTTP/3 need QPACK instead of reusing HPACK?

11. Your team runs internal services in one data centre over HTTP/2. Someone proposes switching
    everything to HTTP/3 for performance. What would you expect, and why?

12. How does a browser discover that a server supports HTTP/3, and what happens if UDP is blocked on
    the user's network?

13. A service calls a backend cluster over HTTP/2 through a layer 4 load balancer. One backend is at
    90% CPU while the others are idle. Why?

14. Why did HTTP/2 server push fail in practice, and what replaced it?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Pipelining lets the client send several requests without waiting, but responses must come back
**in the same order** as the requests, because HTTP/1.1 messages carry nothing that says which request
a response belongs to. A slow first response blocks every response behind it, even ones that are
ready: head-of-line blocking at the HTTP layer. Fixing it needs an identifier on every message, which is
a change to the wire format, and that's exactly what HTTP/2's stream IDs are. Buggy proxies that
mishandled pipelined requests, and the ambiguity about which requests were processed when a connection
died, made it worse, so browsers just turned it off.

**2.** Every request paid a full TCP handshake (1 RTT, plus TLS round trips for HTTPS), started from a
cold congestion window in slow start so even modest responses took several round trips, consumed
server CPU and memory for connection setup, and left a TIME_WAIT entry behind, risking ephemeral port
exhaustion at high request rates. Persistent connections pay those costs once and amortise them.

**3.** They worked around HTTP/1.1's **one-request-at-a-time** connections and the browser's limit of
~6 connections per origin. Sharding got more connections; concatenation and sprites reduced the number
of requests. Under HTTP/2: sharding forces **extra connections** (extra DNS lookups, TCP and TLS
handshakes, separate cold congestion windows) and splits traffic that could share one well-warmed
connection. Concatenation hurts caching: changing one small file invalidates the whole bundle, and the
browser can't use any of it until the large file arrives. With cheap multiplexed requests, smaller
separately cacheable files are often better.

**4.** A stream is one request/response exchange within an HTTP/2 connection, with its own ID. Every
message is broken into frames, and every frame carries its stream ID, so frames from many streams can
be interleaved on one connection and reassembled by ID at the other end. A slow response doesn't block
others; their frames flow in between. One connection is enough because it can carry hundreds of
concurrent streams, and one connection means one handshake and one congestion window that grows large
because all traffic shares it.

**5.** Most headers repeat almost identically on every request on a connection (cookies, auth tokens,
user agent, accept headers). Compressing each request on its own can only find redundancy *inside* that
request. A **shared dynamic table** lets the second and later requests refer to headers already sent,
by a small index, so repeated headers shrink to a byte or two. General-purpose compression like gzip
was avoided because of side-channel attacks (CRIME): if an attacker can inject text into requests and
observe compressed sizes, they can guess secrets like cookies byte by byte. HPACK's indexed-table design
resists that.

**6.** HTTP/2 multiplexes everything over **one TCP connection**. TCP delivers bytes strictly in order,
so when any packet is lost, *every* stream waits for its retransmission, even streams whose data already
arrived. With 2% loss, these stalls happen constantly and hit everything at once, and every loss cuts
the single connection's congestion window. HTTP/1.1 with six connections spreads the risk: a loss on one
connection stalls only that connection's request, and each connection has its own congestion window,
so one loss halves only one sixth of the total sending rate.

**7.** The problem (TCP head-of-line blocking) comes from TCP's core contract: one ordered byte stream.
TCP is implemented in operating system kernels and inspected by middleboxes (firewalls, NATs, load
balancers) across the internet. Changes take many years to deploy everywhere, and middleboxes often drop
packets with unfamiliar TCP options (ossification). UDP gives almost nothing but passes through nearly
everything, so QUIC can implement reliability, per-stream ordering and congestion control in user space,
and encrypt its own headers so it can keep evolving.

**8.** With HTTP/2 over TCP, the connection is identified by the 4-tuple. The phone's IP address
changes, so the connection is dead. The client has to notice (often only after a timeout), open a new
TCP connection, do a new TLS handshake, restart from a cold congestion window, and re-request (or
resume, if the server supports range requests). With HTTP/3, QUIC identifies the connection by a
**connection ID**, not the 4-tuple. The client keeps sending from its new address with the same
connection ID; the server validates the new path and the connection continues. **Connection migration**
makes the difference.

**9.** TCP + TLS 1.2: **3** RTTs. TCP + TLS 1.3: **2**. New QUIC: **1** (transport and TLS 1.3
handshakes are combined). Resumed QUIC with 0-RTT: **0** (the request goes in the first flight). 0-RTT
data can be **captured and replayed** by an attacker, because it's sent before the server has
contributed any fresh randomness, so it isn't protected against replay. Replaying a `GET` is harmless;
replaying "transfer $100" is not. So only idempotent (ideally safe) requests should use 0-RTT.

**10.** HPACK assumes both sides process header blocks in the **same order**, so their dynamic tables
stay in sync. TCP guaranteed that order for HTTP/2. QUIC streams are delivered independently, so if
table updates rode on request streams, a lost packet on one stream would block header decoding on other
streams, bringing back the head-of-line blocking QUIC was built to remove. QPACK sends table updates on
dedicated streams and lets each request declare which table state it needs, so independent streams
don't block one another.

**11.** Very little gain, possibly a loss. Inside one data centre RTTs are sub-millisecond, so saving a
handshake round trip barely matters (especially with long-lived pooled connections), and packet loss is
rare, so TCP head-of-line blocking rarely triggers. Connection migration doesn't matter for servers with
fixed addresses. Meanwhile QUIC costs **more CPU** per byte (user-space processing, less hardware
offload) and has less mature tooling for internal debugging. HTTP/3's benefits are mostly at the edge,
for users on lossy, high-latency, changing networks.

**12.** The server advertises HTTP/3 in an `Alt-Svc` response header on an HTTP/2 or HTTP/1.1 response
(or in a DNS `HTTPS` record, which can be seen before connecting). The browser remembers it and tries
QUIC on later connections, often racing it against a TCP connection and using whichever succeeds. If UDP
is blocked, the QUIC attempt fails or times out and the browser **falls back to HTTP/2 over TCP**. This
is why every HTTP/3 deployment must still support HTTP/2.

**13.** HTTP/2 sends all of a client's requests as streams over **one long-lived connection**. A layer 4
load balancer only chooses a backend when a connection opens, so every request from that client goes to
whichever backend got the connection. A few busy clients pinned to one backend overload it while others
sit idle. Fix: an HTTP/2-aware (L7) load balancer or proxy that balances individual streams, client-side
load balancing across several connections, or a maximum connection age that forces periodic rebalancing
(topic 09).

**14.** The server rarely knows what the client already has cached, so it often pushed resources the
client didn't need, wasting bandwidth and competing with the resources the client actually requested.
It was also complex to configure well and the measured gains were small. Chrome removed it in 2022. The
replacement is `103 Early Hints`: an interim response that tells the client which resources it will
probably need, leaving the decision (and the cache check) to the client.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What four things must every application protocol decide?
**A:** Framing (where messages end), semantics (what's being asked), outcome (what happened), and multiplexing (whether conversations can share a connection).

**Q:** How does HTTP/1.1 frame a message?
**A:** Headers end at a blank line; body length from `Content-Length` or chunked transfer encoding.

**Q:** What two HTTP/1.1 additions still matter?
**A:** Persistent (keep-alive) connections by default, and the mandatory `Host` header for virtual hosting.

**Q:** What's HTTP/1.1's core performance limitation?
**A:** One outstanding request per connection; responses must come back in request order.

**Q:** Why did browsers disable HTTP/1.1 pipelining?
**A:** Responses must return in order, so one slow response blocks the rest (HTTP-layer head-of-line blocking), and many proxies handled it badly.

**Q:** How many connections per origin do browsers typically open over HTTP/1.1?
**A:** About 6.

**Q:** Name three HTTP/1.1-era workarounds that HTTP/2 made unnecessary.
**A:** Domain sharding, file concatenation, image sprites (and inlining).

**Q:** What is request smuggling?
**A:** A proxy and backend disagree on where a text-framed HTTP/1.1 message ends, letting an attacker hide a second request inside the first.

**Q:** What did HTTP/2 change, and what did it keep?
**A:** Changed the wire format (binary frames, streams, multiplexing, HPACK); kept HTTP semantics (methods, status codes, headers, URLs).

**Q:** What is an HTTP/2 stream?
**A:** One request/response exchange, identified by a stream ID carried on every frame, so streams can be interleaved on one connection.

**Q:** What does SETTINGS_MAX_CONCURRENT_STREAMS do?
**A:** Lets the server cap how many streams a client may have open at once on a connection (often 100–250).

**Q:** What is HPACK?
**A:** HTTP/2 header compression: static table + dynamic table shared by both sides + Huffman coding. Repeated headers become small indexes.

**Q:** Why not gzip HTTP headers?
**A:** Compression side channels (CRIME) let attackers infer secrets like cookies from compressed sizes.

**Q:** What happened to HTTP/2 server push?
**A:** Rarely helpful (servers pushed what clients already had cached); removed from Chrome in 2022. Replaced by `103 Early Hints`.

**Q:** How is HTTP/2 negotiated?
**A:** Via ALPN during the TLS handshake; no extra round trip.

**Q:** What head-of-line blocking remains in HTTP/2?
**A:** TCP's: one lost packet stalls every stream on the connection until it's retransmitted.

**Q:** Why do L4 load balancers balance HTTP/2 badly?
**A:** All of a client's requests share one long-lived connection, and L4 balances connections, not requests.

**Q:** What is QUIC?
**A:** A transport protocol over UDP with built-in TLS 1.3, independent streams, its own reliability and congestion control, running in user space.

**Q:** What is protocol ossification?
**A:** Middleboxes depend on visible protocol details (like TCP headers), so the protocol can no longer change.

**Q:** Why is QUIC built on UDP?
**A:** TCP lives in kernels and middleboxes and is effectively impossible to change; UDP passes through almost everywhere and lets QUIC implement its own guarantees.

**Q:** Round trips before the first request: TCP+TLS 1.2, TCP+TLS 1.3, QUIC, QUIC 0-RTT?
**A:** 3, 2, 1, 0.

**Q:** What is QUIC connection migration?
**A:** Connections are identified by a connection ID, not the IP/port 4-tuple, so they survive network changes (Wi-Fi → cellular).

**Q:** Why does QUIC encrypt its transport headers?
**A:** So middleboxes can't depend on them, which keeps the protocol free to evolve (avoids ossification).

**Q:** Why does QUIC never reuse packet numbers?
**A:** So an ACK unambiguously identifies which transmission arrived, making loss detection and RTT estimates accurate.

**Q:** Why QPACK instead of HPACK?
**A:** HPACK needs in-order header processing; QPACK avoids cross-stream blocking on independently delivered QUIC streams.

**Q:** What does QUIC cost?
**A:** More CPU per byte, UDP may be blocked (need TCP fallback), discovery via Alt-Svc / DNS HTTPS records, less visibility for network tools.

**Q:** How does a client discover HTTP/3?
**A:** `Alt-Svc` header on an earlier HTTP/2 response or a DNS `HTTPS` record; then it races QUIC against TCP.

**Q:** Where does HTTP/3 help most?
**A:** At the edge, on lossy, high-latency or changing networks (mobile). Little benefit inside a data centre.

**Q:** The one-line story of HTTP versions?
**A:** 1.1 reused connections; 2 multiplexed streams over one TCP connection; 3 moved streams into QUIC to escape TCP head-of-line blocking.
