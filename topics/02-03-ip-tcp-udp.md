# 03 — IP, TCP & UDP

> Part 2 — Networking & Communication · Prerequisites: topics 01, 02

Every box on a system diagram talks to the others over a network, and the network sets hard limits on
what any design can do. The speed of light fixes the minimum round trip between two cities. TCP decides
how fast a new connection is allowed to send, and turns every lost packet into a stall. Opening a
connection costs round trips, CPU and kernel state before a single useful byte moves.

You don't need to be a network engineer to do system design. You need to understand **what each layer
promises, what it costs, and how it fails**, because those costs show up everywhere: in tail latency,
in why connection pools exist, in why databases fall over when an app tier scales out, and in why
HTTP/3 abandoned TCP (topic 06). This topic covers the transport layer. DNS (topic 04), TLS (topic 05)
and HTTP (topic 06) build directly on it.

By the end of this topic you should be able to:

1. Say what IP, TCP, UDP and application protocols each promise, and what they don't.
2. Walk through the TCP handshake, how TCP makes delivery reliable, and how a connection closes.
3. Explain the difference between flow control and congestion control, and what slow start costs.
4. Explain TCP head-of-line blocking and why it matters.
5. Say when UDP's lack of guarantees is exactly what you want.
6. Explain why opening connections is expensive and how keep-alive and connection pooling avoid it,
   including how pools fail.

---

## 1. The layers that matter

Networks are built in layers. Each layer uses the one below it and offers a slightly better service to
the one above. The textbook OSI model has seven layers; in practice four (plus TLS) matter for system
design.

```
┌─────────────────────────────────────────────────────────────┐
│ Application   HTTP, gRPC, DNS, SMTP, database wire protocols│  what the bytes mean
├─────────────────────────────────────────────────────────────┤
│ (Security)    TLS                                           │  encrypt + authenticate (topic 05)
├─────────────────────────────────────────────────────────────┤
│ Transport     TCP, UDP (and QUIC, built on UDP)             │  process-to-process delivery
├─────────────────────────────────────────────────────────────┤
│ Network       IP (v4, v6)                                   │  host-to-host delivery, routing
├─────────────────────────────────────────────────────────────┤
│ Link          Ethernet, Wi-Fi                               │  one hop, one physical link
└─────────────────────────────────────────────────────────────┘
```

When you send data, each layer wraps the data from the layer above in its own header
(**encapsulation**):

```
[ Ethernet hdr [ IP hdr [ TCP hdr [ TLS record [ HTTP request ] ] ] ] ]
```

Routers along the path only look at the IP header. Only the two endpoints look at TCP and above. (In
practice, many "middleboxes" such as firewalls, NAT devices and load balancers also peek at TCP
headers. That habit turns out to matter a lot: it's part of why TCP is so hard to change, topic 06.)

When people say "L4" and "L7" (as in "an L4 load balancer", topic 09) they mean the transport layer
(TCP/UDP: sees addresses and ports) and the application layer (HTTP: sees paths, headers, cookies).

### IP: best-effort delivery between hosts

IP gets a **packet** from one host address to another, hop by hop through routers. That's all. It
promises almost nothing:

- Packets can be **lost** (a router's queue was full, so it dropped them).
- Packets can arrive **out of order** (they took different paths).
- Packets can be **duplicated**.
- Packets can be **corrupted** (detected by checksums and dropped).
- There's a maximum packet size, the **MTU** (typically 1,500 bytes on Ethernet). Larger data must be
  split into several packets.

This is a deliberate design choice. Keeping the core network simple and "dumb" (routers forward packets
and keep no per-connection state) made the internet scalable; any guarantees are added at the
endpoints by whoever needs them. Remember this pattern: **an unreliable layer underneath, with
reliability built on top at the edges.** It recurs with message queues, retries and idempotency
(topics 36–38).

### Ports and the 4-tuple

An IP address identifies a host (more precisely, a network interface). A **port** (16 bits, 0 to
65,535) identifies a process on that host. A TCP connection is identified by its **4-tuple**:

```
(source IP, source port, destination IP, destination port)
```

The server listens on a well-known port (443 for HTTPS, 5432 for Postgres). Each client connection
uses a temporary **ephemeral port** chosen by the client's OS. Two consequences come up later:

- The number of ephemeral ports is finite (about 28,000 by default on Linux), which limits how many
  connections one client can have to one server address and port (section 2, TIME_WAIT).
- When a client's IP address changes (a phone moving from Wi-Fi to cellular), the 4-tuple changes, and
  every TCP connection it had is dead. QUIC fixes this with connection IDs (topic 06).

### The speed of light is the floor

Data in fibre travels at roughly two-thirds the speed of light, about **200 km per millisecond**. Real
paths aren't straight, and every router adds a little. Rough round-trip times (RTT):

| Path | Typical RTT |
|---|---|
| Same data centre | 0.1 to 0.5 ms |
| Same region, different data centres | 1 to 2 ms |
| Across a continent | 30 to 80 ms |
| Across an ocean | 70 to 150 ms |
| Opposite side of the world | 200 to 300 ms |

No amount of engineering removes this (topic 02 puts these numbers next to memory and disk). What you
*can* control is **how many round trips** an operation needs. Much of this topic, and of topics 04–06,
is really about counting round trips.

---

## 2. TCP: a reliable byte stream on top of an unreliable network

TCP turns IP's lossy, unordered packets into what looks to the application like a **reliable, ordered
stream of bytes** between two processes. It promises:

- **Reliable delivery**: every byte arrives, or the connection reports an error.
- **In-order delivery**: bytes arrive in the order they were sent.
- **No duplicates**.
- **Flow control**: the sender won't overwhelm the receiver.
- **Congestion control**: the sender won't (deliberately) overwhelm the network.

Note what's missing: TCP has **no message boundaries**. If you write 100 bytes then 200 bytes, the other
side might read 300 bytes at once, or 50 then 250. Application protocols must add their own **framing**
(a length prefix, or a delimiter like HTTP/1.1's blank line after the headers). Every "my JSON message
arrived in two halves" bug comes from forgetting this.

### The three-way handshake

Before any data flows, the two sides set up state: each picks a random **initial sequence number
(ISN)** and tells the other.

```
Client                                        Server
  │                                             │  (listening on :443)
  │ ── SYN, seq=x ────────────────────────────▶ │
  │                                             │  allocates state for the half-open connection
  │ ◀──────────────── SYN-ACK, seq=y, ack=x+1 ──│
  │                                             │
  │ ── ACK, ack=y+1 ──────────────────────────▶ │
  │    (data can ride along with this ACK)      │
  │                                             │
  │══════════ connection established ══════════│
```

**Cost: one full round trip before the client can send its first byte of data.** At a 150 ms RTT,
that's 150 ms spent before the request has even started. (An extension called **TCP Fast Open** lets a
returning client put data in the SYN, but it's rarely usable across the internet because middleboxes
interfere with it; QUIC achieves the same goal more reliably, topic 06.)

Why random ISNs? If they were predictable, an attacker could forge packets that look like part of an
existing connection. Randomness also stops stray packets from an old connection being mistaken for the
new one.

**SYN floods**: the server keeps state for every half-open connection (after SYN, before the final
ACK). An attacker sending millions of SYNs from fake source addresses fills that table and legitimate
clients can't connect. The standard defence, **SYN cookies**, encodes the needed state into the
server's own sequence number, so the server stores nothing until a final ACK proves the client is real.

### How TCP makes delivery reliable

- **Sequence numbers**: every byte has a number. The receiver uses them to reorder segments and discard
  duplicates.
- **Acknowledgements (ACKs)**: the receiver tells the sender "I have everything up to byte N"
  (**cumulative ACK**). Modern TCP also uses **selective ACKs (SACK)**: "I also have bytes 5000–8000",
  so the sender resends only the gap.
- **Retransmission timeout (RTO)**: if no ACK arrives in time, resend. The timeout is computed from a
  running estimate of the RTT and its variance. After repeated timeouts it **backs off exponentially**,
  so a dead link doesn't get hammered.
- **Fast retransmit**: if the receiver keeps getting segments *after* a gap, it keeps ACKing the same
  number. Three duplicate ACKs tell the sender "something specific went missing", and it resends
  immediately without waiting for the timer.
- **Checksums** catch corruption.

The important consequence for system design: **a lost packet doesn't cause an error, it causes a
delay.** The initial RTO for a SYN is about 1 second on many systems, and the minimum RTO for an
established connection is 200 ms on Linux. A single dropped packet can turn a 2 ms request into a
200+ ms one; a dropped SYN can turn it into a 1+ second one. Rare enough to vanish in the average,
common enough to dominate p99: this is one of the main sources of tail latency (topics 01, 02),
and fan-out (one request calling 50 backends) makes it much worse.

And when the network really is broken, "reliable" means **you get an error eventually**: a timeout or
a connection reset. TCP cannot deliver over a dead path, and how long it takes to *tell* you depends on
timeouts you configure (topic 43).

### What an ACK does and doesn't mean

An ACK means the peer's **kernel** has the bytes in its receive buffer. It does not mean the peer
application read them, processed them, or committed anything to disk. If the server process crashes
right after the ACK, the data is gone and the client may never know whether its request was acted on.
That is why "did my request succeed?" is an application-level question, answered with responses,
idempotency and retries (topics 07, 38), not by TCP.

### Closing a connection, and TIME_WAIT

Either side can close. Each direction is shut down separately with a FIN that the other side ACKs:

```
Active closer                                Passive closer
  │ ── FIN ──────────────────────────────────▶ │
  │ ◀──────────────────────────────────── ACK ─│
  │ ◀──────────────────────────────────── FIN ─│
  │ ── ACK ──────────────────────────────────▶ │
  │                                             │
  TIME_WAIT (2 × MSL, 60 s on Linux)            closed
```

The side that closes first enters **TIME_WAIT** and keeps the 4-tuple reserved for a while (twice the
"maximum segment lifetime"). This exists so that delayed packets from the old connection can't be
mistaken for a new connection with the same 4-tuple, and so the final ACK can be resent if it was lost.

A connection can also end abruptly with a **RST** (reset): "I have no such connection" or "I'm aborting
this". The application sees "connection reset by peer". Common causes are a crashed process, a server
or proxy that already closed an idle connection, or a firewall that dropped its state.

Why TIME_WAIT matters to you: a client that opens and closes many short connections to the **same
server** piles up TIME_WAIT entries, each holding an ephemeral port. Linux's default ephemeral range is
about 28,000 ports. With a 60-second TIME_WAIT, that caps a client at roughly
`28,000 / 60 ≈ 470` new connections per second to one destination before it runs out of ports and
fails with errors like "cannot assign requested address". This is **ephemeral port exhaustion**. Kernel
tweaks move the ceiling; the real fix is to stop opening so many connections (section 4).

### Flow control: don't overwhelm the receiver

The receiver has a finite buffer. If the application reads slowly, the buffer fills. In every ACK the
receiver advertises a **receive window (rwnd)**: "I have room for this many more bytes." The sender
never has more than rwnd unacknowledged bytes in flight. If rwnd drops to zero, the sender stops and
periodically probes until space opens up.

Flow control protects **the receiving process**. It's a conversation between the two endpoints only.
It's also a natural form of **backpressure**: a slow consumer automatically slows the producer, all
the way back up the chain if each hop only reads as fast as it can forward. (Topic 36 meets the same
idea in queues.)

### Congestion control: don't overwhelm the network

The receiver might have plenty of room, but some router in the middle might not. If every sender blasts
at full speed, router queues overflow, packets drop, everyone retransmits, and useful throughput
collapses. The internet actually suffered **congestion collapse** in 1986, with throughput on some links
dropping by a factor of a thousand; congestion control was added in response.

The sender keeps a second limit, the **congestion window (cwnd)**, which is its own estimate of how
much the network can take. Nobody tells it this number; it infers it from loss (and, in newer
algorithms, from delay).

```
bytes in flight ≤ min(rwnd, cwnd)
                       │      │
     receiver's limit ─┘      └─ network's limit (sender's guess)
```

**Slow start.** A new connection doesn't know the network's capacity, so it starts small (typically
**10 segments, about 14 KB**) and roughly **doubles cwnd every round trip** while ACKs keep coming back:

```
cwnd (segments)
  │                                         loss!
  │                                    ╱╲
  │                                 ╱     ╲___ halve, then grow linearly
  │                              ╱              ╱‾‾╱
  │                          ╱              ╱‾‾    (AIMD sawtooth)
  │                     ╱
  │              __/
  │  ____----‾‾
  └──────────────────────────────────────────────── time (RTTs)
    slow start          congestion avoidance
    (exponential)       (additive increase, multiplicative decrease)
```

**Congestion avoidance (AIMD).** After a threshold, growth becomes linear (+1 segment per RTT). On
loss, cwnd is cut (roughly halved). This "additive increase, multiplicative decrease" sawtooth is what
lets many connections share a link roughly fairly.

Modern algorithms refine this. **CUBIC** (the Linux default) grows faster after recovering from loss.
**BBR** (from Google) models the bottleneck bandwidth and RTT directly instead of waiting for loss,
which works better on links with deep buffers or random loss (like mobile networks).

**Why slow start matters for system design:**

- A **new connection is slow at first**, regardless of bandwidth. Transferring 1 MB on a fresh
  connection at 100 ms RTT takes several round trips just to ramp up: 14 KB, 28 KB, 56 KB, 112 KB,
  224 KB, 448 KB... about 7 round trips, so **~700 ms**, even on a gigabit link where the raw transfer
  would take 8 ms.
- A **reused, warm connection** already has a large cwnd and sends at full speed immediately. This is a
  big reason connection reuse matters (section 4).
- On many systems cwnd also resets after a connection sits idle for a while, so even keep-alive
  connections can lose their warm-up.

### Bandwidth-delay product

To keep a pipe full you need enough data in flight to cover a full round trip:

```
bytes in flight needed = bandwidth × RTT

Example: 1 Gbps × 100 ms = 100 Mbit = 12.5 MB in flight
```

If either window (rwnd or cwnd) is smaller than this, you can't use the full bandwidth. Maximum
throughput on one connection is roughly `window / RTT`. The original TCP window field allowed only
64 KB, which at 100 ms RTT caps one connection at about **5 Mbps** no matter how fat the link. The
**window scaling** option fixed the field size, but socket buffer sizes still need to be large enough.
The general lesson: **on long, fast links, a single TCP connection is limited by RTT**, not bandwidth.
That's why large cross-region transfers often use several parallel connections.

### Head-of-line blocking

TCP promises in-order delivery. So if segment 3 is lost, segments 4, 5 and 6 can arrive and sit in the
receiver's buffer, but the application **can't read any of them** until segment 3 is retransmitted:

```
sent:      [1] [2] [3] [4] [5] [6]
arrived:   [1] [2]  ✗  [4] [5] [6]
app reads: [1] [2]  ...waiting...waiting...  (one RTO or fast retransmit later)
           [3] [4] [5] [6]
```

This is **head-of-line (HOL) blocking**. It's harmless when the stream really is one sequence (a file
download needs all bytes in order anyway). It's harmful when you send **several independent things over
one connection**: a lost packet belonging to resource A stalls resources B and C as well, even though
their bytes already arrived. HTTP/2 multiplexes many requests over a single TCP connection and gets hit
by exactly this on lossy networks. Solving it required leaving TCP behind (HTTP/3 over QUIC, topic 06).

### Nagle's algorithm and delayed ACKs

Two old optimizations to avoid sending lots of tiny packets:

- **Nagle's algorithm** (sender): if there's unacknowledged data in flight, hold small writes and bundle
  them until the ACK arrives.
- **Delayed ACK** (receiver): wait briefly (up to ~40 ms on Linux) before ACKing, hoping to piggyback
  the ACK on a response.

Each is reasonable alone. Together they can stall: the sender waits for an ACK before sending the rest
of a message, the receiver waits for more data before ACKing, and you get mysterious ~40 ms delays on
request/response protocols. This is why latency-sensitive software (database drivers, RPC frameworks,
Redis clients) sets **`TCP_NODELAY`**, which disables Nagle. It's topic 01's batching trade-off again:
Nagle buys throughput with latency.

---

## 3. UDP: when unreliability is a feature

UDP is barely more than raw IP with ports added. Its header is 8 bytes. It offers:

- **No connection**: no handshake. The first packet can carry data.
- **No reliability**: lost packets are just gone.
- **No ordering**: packets arrive in whatever order they arrive.
- **No congestion or flow control**: it sends as fast as you tell it to.
- **Message boundaries are kept**: one send is one datagram on the other side (unlike TCP's stream).

That sounds strictly worse than TCP. It isn't, because TCP's guarantees have costs, and some
applications don't want to pay them:

| Use case | Why UDP fits |
|---|---|
| **DNS** | One small question, one small answer. A TCP handshake would double the round trips, and the server would need per-client connection state. If the reply is lost, just ask again (topic 04). |
| **Voice and video calls** | A late packet is useless: audio from 300 ms ago can't be played now. TCP would stall everything waiting for a retransmit (HOL blocking). Better to skip or conceal the lost bit and keep going. |
| **Online games** | Only the latest position matters. A retransmitted old position is worse than nothing. |
| **Metrics (e.g. StatsD)** | Fire-and-forget. Losing one sample of a counter is fine; blocking the app to send it isn't. |
| **QUIC / HTTP/3** | Builds its *own* reliability, ordering and congestion control on top of UDP, but per stream, avoiding TCP's HOL blocking (topic 06). |

The key insight: **UDP lets the application decide which guarantees it needs.** TCP's guarantees are
all-or-nothing and built into the OS kernel, which makes them hard to change. Building on UDP lets you
pick exactly what you need and evolve it in application code. That's precisely why QUIC is built on UDP
rather than being a new kernel protocol.

What you give up, and must handle yourself if you need it: retransmission, ordering, duplicate
detection, splitting large messages to fit the MTU, and congestion control. An application blasting UDP
with no congestion control harms everyone else on the link, and some networks throttle UDP for exactly
that reason.

A security note: because UDP has no handshake, the source address can be **spoofed**. Attackers send a
small request to a UDP service (DNS, NTP, memcached) with the victim's address as the source, and the
service sends a much larger reply to the victim. Thousands of such servers multiply the attacker's
bandwidth. This is an **amplification attack**. TCP's handshake prevents it: the SYN-ACK goes to the
spoofed address, the handshake never completes, and no large response is ever sent. (QUIC limits how
much a server may send to an unverified address for the same reason.)

---

## 4. Connection reuse: keep-alive and pooling

### Why opening a connection is expensive

Add up what a **brand-new HTTPS connection** costs a client 100 ms away from the server (DNS is
topic 04, TLS is topic 05):

```
DNS lookup (cache miss)         0 to 100+ ms
TCP handshake                   1 RTT   = 100 ms
TLS 1.3 handshake               1 RTT   = 100 ms
Request + response              1 RTT   = 100 ms   ← the part you actually wanted
Slow start                      more RTTs if the response is large
───────────────────────────────────────────────
≈ 300 to 400 ms, of which only ~100 ms is useful work
```

On a reused connection the same request costs **one round trip**, and the congestion window is already
warm. Both sides also save:

- **CPU**: TLS handshakes involve expensive public-key operations; the kernel does work per connection.
- **Memory**: per-connection socket buffers and state in the kernel and the application.
- **Ports**: no ephemeral port consumed and no TIME_WAIT entry left behind.

Inside a data centre the RTT is tiny, but the CPU and memory costs remain, and for databases the
per-connection setup is far heavier (below).

### Keep-alive

**HTTP keep-alive** (persistent connections): instead of closing the TCP connection after each
response, leave it open and send the next request over it. It's the default in HTTP/1.1 and later
(topic 06). Both sides close it after an idle timeout.

Pitfalls:

- **Idle timeout mismatches.** If the server (or a load balancer or NAT device in between) closes an
  idle connection while the client still thinks it's open, the client's next request fails with a
  reset. Rule of thumb: the **client's idle timeout should be shorter than the server's**, so the
  client always gives up first. A request that hits this race is also a reason clients retry safe
  requests on a fresh connection.
- Don't confuse **HTTP keep-alive** with **TCP keepalive**: the latter is an OS feature that sends small
  probe packets on idle connections to detect a dead peer (and keep NAT and firewall mappings alive).
  Without it, a connection to a host that vanished (power loss, no FIN or RST sent) can look open
  forever.

### Connection pooling

A **connection pool** keeps a set of open connections to a dependency and lends them out:

```
App threads                Pool (max 20)                  Database
 req ─┐                  ┌─────────────────┐
 req ─┼── borrow ───────▶│ ■ ■ ■ ■ ■ □ □ □ │═══ 20 long-lived connections ═══▶ DB
 req ─┘◀── return ───────│ ■ busy  □ idle  │
                         └─────────────────┘
          when all are busy: wait (up to a timeout) or fail fast
```

Pooling matters most for **database connections**, which are much more expensive than HTTP
connections. Opening one involves TCP, often TLS, then authentication and session setup. Some databases
dedicate significant memory per connection; PostgreSQL, for instance, starts a whole operating-system
**process** for each one. A database that comfortably runs a few hundred connections can fall over at a
few thousand.

**Sizing a pool** uses Little's Law (topic 02):

```
connections needed ≈ request rate to the dependency × time each request holds a connection

500 queries/s × 10 ms each = 5 connections busy on average
```

Then add headroom for bursts. Bigger isn't better: the database has a fixed number of CPU cores and
disks. Past that, extra connections just queue *inside* the database, competing for locks, memory and
CPU, and throughput often goes **down** (the queueing behaviour from topic 02). A small pool with a
short queue in front of it usually beats a huge pool.

**The multiplication problem.** Pools are per process. 50 app instances × 20 connections each = 1,000
database connections, even if most are idle. As you scale the stateless tier out, you can overwhelm the
stateful tier behind it. The fix is a **shared pooler** (a proxy such as PgBouncer) that many app
instances connect to, which in turn holds a small number of real connections to the database.

Other pooling failure modes:

- **Stale connections**: the server, a firewall or a NAT silently dropped an idle connection. Pools
  validate connections on borrow, or retire them after a maximum lifetime.
- **Pool exhaustion**: if a dependency slows down, each request holds its connection longer. By
  Little's Law, 50× the hold time needs 50× the connections, so the pool runs dry and requests queue
  for a connection, holding threads and memory while they wait. The slowness spreads upstream. Always
  set a **timeout for acquiring** a connection, so a slow dependency causes fast errors instead of a
  pile-up. Circuit breakers and bulkheads (topic 44) build on this.
- **Load imbalance**: long-lived connections stick to whichever backend they first reached. When you add
  a new server behind a load balancer that balances connections (L4, topic 09), existing pools don't
  move to it. Setting a maximum connection lifetime makes connections slowly redistribute.
- **DNS changes are invisible** to already-open pooled connections: they keep talking to the old IP
  until they're closed (topic 04).

### Putting it together

Every technique in topics 03–06 either removes a round trip or makes it shorter:

| Technique | What it saves | Topic |
|---|---|---|
| Keep-alive / pooling | TCP + TLS handshakes and slow start, on every request after the first | 03 |
| DNS caching | The DNS lookup | 04 |
| TLS 1.3 | One handshake RTT | 05 |
| Session resumption / 0-RTT | Certificate work, and with 0-RTT one more RTT | 05 |
| HTTP/2 multiplexing | Extra connections (and their handshakes) per origin | 06 |
| QUIC / HTTP/3 | Combines transport and TLS handshakes; removes TCP HOL blocking | 06 |
| CDN edge / anycast / GeoDNS | Makes every RTT shorter by moving the server closer | 04, 14 |

---

## 5. How it connects

- **Topic 01–02**: TCP turns loss into *delay* (retransmission), which feeds directly into tail latency
  and is amplified by fan-out. Pool sizing is Little's Law; oversized pools hit the queueing cliff.
- **Topic 04 (DNS)** runs mostly over UDP for exactly the reasons in section 3, and open pooled
  connections are why DNS changes take effect slowly.
- **Topic 05 (TLS)** adds handshake round trips and CPU on top of TCP's, which makes reuse even more
  valuable.
- **Topic 06 (HTTP)**: HTTP/1.1's keep-alive, HTTP/2's multiplexing over one connection, and HTTP/3's
  move to QUIC over UDP are all responses to the costs and the head-of-line blocking described here.
- **Topic 09 (load balancing)**: L4 vs L7 balancing is about which of these layers the balancer reads;
  long-lived pooled connections defeat connection-level balancing.
- **Topics 36–38**: "unreliable layer below, guarantees built on top at the edges" returns with message
  delivery, and the "did my request happen?" problem returns as idempotency.
- **Topics 43–44**: timeouts on connect and acquire, and circuit breakers, are how you stop TCP's
  patient retransmissions and exhausted pools from turning into cascading failure.

## Common misconceptions

- **"TCP guarantees delivery."** → It guarantees delivery *or an error*. If the path is broken, data
  doesn't arrive; you get a timeout or a reset, possibly much later. And an ACK only means the peer's
  *kernel* received the bytes, not that the application processed them.
- **"More bandwidth makes everything faster."** → For small requests, latency is dominated by round
  trips, and RTT is bound by distance. A fatter pipe doesn't shorten the handshake, and a fresh
  connection can't use the bandwidth until slow start ramps up.
- **"UDP is just unreliable TCP."** → It's a minimal building block that lets applications choose
  their own guarantees. QUIC is built on it precisely for that reason.
- **"Flow control and congestion control are the same thing."** → Flow control protects the receiver
  (rwnd); congestion control protects the network (cwnd). The sender obeys the smaller of the two.
- **"A packet loss shows up as an error I can log."** → It shows up as latency: a retransmission
  after a duplicate-ACK signal or a timeout of 200 ms or more.
- **"Messages I write to a TCP socket arrive as the same messages."** → TCP is a byte stream; you must
  frame messages yourself.
- **"A bigger connection pool means more throughput."** → Past the database's real capacity, extra
  connections add contention and reduce throughput.
- **"TIME_WAIT is a bug to tune away."** → It protects new connections from old packets. Running out
  of ports because of it means you're opening too many connections; reuse them.

## Further reading (supplements)

- Ilya Grigorik, *High Performance Browser Networking* (free online): chapters 1–3 on latency, TCP and
  UDP are the best practical treatment of this material
- W. Richard Stevens, *TCP/IP Illustrated, Volume 1*: the classic deep reference
- Van Jacobson, "Congestion Avoidance and Control" (1988): the paper written after the congestion
  collapse
- Cardwell et al., "BBR: Congestion-Based Congestion Control" (ACM Queue, 2016)
- The HikariCP wiki page "About Pool Sizing": why small database pools outperform large ones
- RFC 9293 (TCP), RFC 768 (UDP), if you want the authoritative source for a detail

---

## Questions

1. IP is "best-effort." List four things that can go wrong with an IP packet. Why was it a good design
   decision to keep IP this weak?

2. A service in Mumbai calls a dependency in Virginia (RTT ~200 ms). A teammate proposes upgrading the
   link from 1 Gbps to 10 Gbps to speed up these small API calls. Will it help? What would?

3. Walk through the TCP three-way handshake. What does each side learn, and what does it cost before
   the first byte of application data can be sent?

4. TCP is described as "reliable." A request's packet gets dropped once by a congested router. What
   does the application see? How does this connect to tail latency?

5. A client sends a request, the server's TCP stack ACKs it, and then the server process crashes. Did
   the request "succeed"? What does this tell you about what TCP can and can't promise?

6. What's the difference between flow control and congestion control? Who is each one protecting, and
   how does the sender combine them?

7. A brand-new TCP connection on a 1 Gbps link with 100 ms RTT is used to download a 1 MB file. Why
   does it take far longer than `1 MB / 1 Gbps`? Roughly how long does it take, and what would make it
   faster?

8. A single TCP connection between two data centres with 80 ms RTT can't get more than a fraction of
   the available bandwidth, even with no packet loss. What limits it? What's the general formula?

9. Explain TCP head-of-line blocking. Why is it harmless for downloading one file but harmful when many
   independent requests share one connection?

10. A service opens a new TCP connection to the same downstream service for every request, at about
    1,000 requests/second, and starts failing with "cannot assign requested address" errors. What's
    happening, and what's the right fix?

11. A request/response protocol shows strange, consistent ~40 ms delays on small messages even inside
    one data centre. What's a likely cause, and what's the usual fix?

12. Why does a video-calling app prefer UDP over TCP, even though UDP loses packets?

13. If UDP has no reliability, how can QUIC (built on UDP) deliver web pages reliably? Why build on UDP
    instead of designing a new transport protocol in the OS?

14. Why are UDP services like DNS and NTP used in amplification attacks, while TCP services mostly
    aren't?

15. You have 40 app instances, each with a database pool of 25 connections. The database starts
    struggling. What's the problem, and what are two ways to fix it?

16. A dependency slows from 10 ms to 500 ms per call. Explain what happens to the connection pool and
    to the calling service. What setting limits the damage?

17. After adding three new backend servers behind a load balancer, they receive almost no traffic from
    one client service, which uses a connection pool. Why?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Packets can be lost (dropped at a full router queue), arrive out of order (different paths), be
duplicated, or be corrupted (detected by checksum and dropped). Also, packets larger than the MTU must
be split. Keeping IP weak keeps the core network simple and fast: routers just forward packets and hold
no per-connection state. Guarantees are added at the endpoints, only by the applications that need
them, which let the internet scale and let new protocols be built without changing routers.

**2.** No. Small API calls are bound by **round trips**, not bandwidth. A 2 KB request and response
barely use 1 Gbps; the time is dominated by the 200 ms RTT, multiplied by however many round trips each
call needs (handshakes included). What helps: **reuse connections** (no repeated TCP/TLS handshakes),
**reduce the number of round trips** per operation (batch calls, avoid chatty sequential calls), or
**move the data closer** (cache or replicate it near Mumbai, or move the dependency).

**3.** Client sends SYN with its initial sequence number (ISN) x. Server replies SYN-ACK with its own
ISN y and acknowledges x+1. Client sends ACK of y+1. Each side learns the other's starting sequence
number (needed to order bytes and detect duplicates) and that the other side is reachable and willing.
The server allocates connection state. Cost: **one full RTT** before the client can send data (the data
can ride along with the final ACK).

**4.** The application sees **no error, just a delay**. The sender doesn't get an ACK, so it
retransmits, either after three duplicate ACKs (fast retransmit, about one extra RTT) or after a
retransmission timeout (at least 200 ms on Linux, often more; about 1 s for a lost SYN). Meanwhile,
later bytes wait because of in-order delivery. A request that normally takes 2 ms takes 200+ ms. These
occasional retransmissions are a classic source of p99/p99.9 latency: rare enough to vanish in the
average, common enough to dominate the tail, especially with fan-out.

**5.** Not necessarily. The ACK only means the server's **kernel** received the bytes into its buffer.
The application may never have read them, or may have crashed halfway through processing. The client
may see a reset or a timeout and can't tell whether the work happened. TCP promises byte delivery
between kernels, not that an operation was carried out. Knowing the outcome needs an application-level
response, and recovering safely needs retries plus idempotency (topics 07, 38).

**6.** **Flow control** protects the **receiver**: the receiver advertises how much buffer space it has
(rwnd), so a fast sender doesn't overrun a slow reader. **Congestion control** protects the
**network**: the sender keeps its own estimate (cwnd) of how much the path can carry, inferred from loss
or delay, so it doesn't flood router queues. The sender keeps no more than **`min(rwnd, cwnd)`** bytes
unacknowledged in flight.

**7.** Slow start. A new connection starts with a small congestion window (~10 segments, ~14 KB) and
roughly doubles it each RTT: 14, 28, 56, 112, 224, 448 KB... Getting 1 MB through takes about **7 round
trips ≈ 700 ms**, while `1 MB / 1 Gbps` is only 8 ms. Bandwidth is irrelevant until the window grows.
Faster: reuse a **warm connection** (cwnd already large), raise the initial window, or shorten the RTT
(serve from closer, e.g. a CDN).

**8.** The **window**. One connection can have at most a window's worth of unacknowledged data in
flight per round trip, so `max throughput ≈ window / RTT`. To fill the pipe you need
`window ≥ bandwidth × RTT` (the **bandwidth-delay product**). With a 64 KB window and 80 ms RTT, that's
64 KB / 0.08 s ≈ 800 KB/s ≈ 6.5 Mbps, whatever the link speed. Fixes: TCP window scaling and larger
socket buffers, or several parallel connections.

**9.** TCP delivers bytes strictly in order. If segment 3 is lost, segments 4–6 can arrive but the
application can't read them until 3 is retransmitted. For one file you need all bytes in order anyway,
so nothing is lost by waiting. When many independent requests share one connection (like HTTP/2
multiplexing), a packet lost from request A blocks requests B and C too, even though their data already
arrived. One loss stalls everything, which is why HTTP/3 moved to QUIC, which orders data per stream.

**10.** **Ephemeral port exhaustion** from TIME_WAIT. Each closed connection leaves its 4-tuple in
TIME_WAIT (60 s on Linux) on the side that closed first. To one destination IP and port, the only thing
that varies is the client's ephemeral port, and Linux has about 28,000 by default.
`28,000 / 60 s ≈ 470` new connections per second is the ceiling; at 1,000/s the client runs out of
ports. The right fix is **connection reuse** (keep-alive / pooling), which also removes a handshake from
every request. Widening the port range or tweaking TIME_WAIT settings only moves the ceiling.

**11.** **Nagle's algorithm interacting with delayed ACKs.** The sender's Nagle logic holds a small
write until outstanding data is acknowledged; the receiver delays its ACK (up to ~40 ms) hoping to
piggyback it on a response. Each waits for the other until the delayed-ACK timer fires. Fix: set
**`TCP_NODELAY`** on the socket to disable Nagle (and write each message in one call rather than several
small writes).

**12.** In a live call, **late data is useless**. Audio from 300 ms ago can't be played now. With TCP,
one lost packet would stall all following audio until it's retransmitted (head-of-line blocking),
causing freezes and growing delay. With UDP the app skips or conceals the missing bit and keeps playing
current audio. The app can also adapt its bitrate itself.

**13.** QUIC implements its own **reliability, ordering and congestion control in user space**, on top
of UDP datagrams: packet numbers, ACKs, retransmissions, with ordering per stream, so a loss in one
stream doesn't block others. Building on UDP was practical: TCP lives in OS kernels and in countless
middleboxes (firewalls, NATs) that inspect and sometimes interfere with transport headers. A brand-new
transport protocol would be blocked by many of them and would take years to roll out across operating
systems. UDP already passes through nearly everything, and QUIC can ship and evolve inside
applications.

**14.** UDP has **no handshake**, so the source address isn't verified. An attacker sends a small
request with the victim's IP forged as the source, and the server sends a much larger response to the
victim. Many servers multiply the attacker's bandwidth. With TCP, the server's SYN-ACK goes to the
forged address, the handshake never completes, and no large response is ever sent.

**15.** `40 × 25 = 1,000` database connections. Most are idle, but each costs the database memory (a
whole process per connection in Postgres), and when many are active they compete for CPU, locks and
memory inside the database, so throughput drops. Pools multiply as the stateless tier scales out.
Fixes: **shrink each pool** (size it by Little's Law: query rate × hold time, plus headroom), and/or put
a **shared connection pooler** (like PgBouncer) between the apps and the database, so many client
connections map onto a small number of real database connections.

**16.** By Little's Law, connections in use = call rate × hold time. At 50× the hold time, the pool
needs 50× the connections, so it **runs dry**. New requests **queue waiting for a connection**, holding
their threads and memory; the caller's own latency spikes and it may run out of threads too, so the
slowness spreads upstream to its callers. The key setting is a **connection-acquire timeout** (plus
request timeouts on the calls themselves): requests that can't get a connection quickly fail fast
instead of piling up. Circuit breakers (topic 44) build on this.

**17.** The client's pool holds **long-lived connections** that were opened before the new servers
existed. A connection-level (L4) load balancer only makes a routing decision when a connection is
*opened*, so existing connections stay on the old servers. New servers only get traffic as new
connections open. Fix: set a **maximum connection lifetime** in the pool so connections are
periodically recycled and spread across all servers, or use a load balancer that balances per request
(L7, topic 09).

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What does IP promise?
**A:** Best-effort delivery of packets between hosts. Packets may be lost, reordered, duplicated or corrupted.

**Q:** What identifies a TCP connection?
**A:** The 4-tuple: (source IP, source port, destination IP, destination port).

**Q:** Roughly how fast does data travel in fibre?
**A:** About 200 km per millisecond (two-thirds the speed of light).

**Q:** What does TCP promise that IP doesn't?
**A:** A reliable, in-order, duplicate-free byte stream, plus flow control and congestion control.

**Q:** Does TCP preserve message boundaries?
**A:** No. It's a byte stream; applications must add their own framing (length prefix or delimiter).

**Q:** What's the cost of the TCP handshake?
**A:** One RTT before the client can send data.

**Q:** Why are TCP initial sequence numbers random?
**A:** To stop attackers forging packets into a connection, and to avoid confusing packets from an old connection.

**Q:** What's a SYN flood, and the standard defence?
**A:** Flooding a server with SYNs to fill its half-open connection table. Defence: SYN cookies (store no state until the final ACK).

**Q:** What does a dropped packet look like to a TCP application?
**A:** A delay (fast retransmit, or a timeout of 200 ms or more), not an error. A big source of tail latency.

**Q:** What triggers fast retransmit?
**A:** Three duplicate ACKs, signalling a specific missing segment.

**Q:** What does a TCP ACK actually prove?
**A:** That the peer's kernel has the bytes. Not that the application read or processed them.

**Q:** What is TIME_WAIT and who enters it?
**A:** The side that closes first keeps the 4-tuple reserved (60 s on Linux) so stray old packets can't corrupt a new connection.

**Q:** What's ephemeral port exhaustion?
**A:** Opening and closing many connections to one destination fills TIME_WAIT and uses up the ~28k client ports. Fix: reuse connections.

**Q:** Flow control vs congestion control?
**A:** Flow control (rwnd) protects the receiver's buffer. Congestion control (cwnd) protects the network. Send ≤ min(rwnd, cwnd).

**Q:** What is slow start?
**A:** A new connection starts with a small cwnd (~10 segments, ~14 KB) and doubles it each RTT until loss or a threshold.

**Q:** What is AIMD?
**A:** Additive increase, multiplicative decrease: grow cwnd linearly, cut it (roughly halve) on loss. Lets connections share a link fairly.

**Q:** CUBIC vs BBR, in one line?
**A:** CUBIC (Linux default) is loss-based; BBR models bottleneck bandwidth and RTT directly instead of waiting for loss.

**Q:** Bandwidth-delay product?
**A:** Bandwidth × RTT: the bytes in flight needed to keep a pipe full. Max throughput per connection ≈ window / RTT.

**Q:** What is TCP head-of-line blocking?
**A:** A lost segment stops the app reading later bytes that already arrived, until the gap is filled. Hurts when independent streams share one connection.

**Q:** Nagle + delayed ACK problem, and the fix?
**A:** Sender waits for an ACK, receiver delays its ACK: ~40 ms stalls. Fix: `TCP_NODELAY`.

**Q:** What does UDP provide?
**A:** Ports, a checksum, and message boundaries. No connection, reliability, ordering, or congestion control.

**Q:** Name four good uses of UDP.
**A:** DNS, voice/video calls, online games, fire-and-forget metrics (and QUIC/HTTP/3).

**Q:** Why is QUIC built on UDP?
**A:** UDP passes through existing networks; QUIC adds its own per-stream reliability in user space, avoiding kernel and middlebox limitations.

**Q:** What is a UDP amplification attack?
**A:** A small spoofed request makes a server send a much larger reply to the victim. Possible because UDP has no handshake.

**Q:** Cost of a cold HTTPS request vs a reused connection?
**A:** Cold: DNS + TCP (1 RTT) + TLS (1 RTT) + request (1 RTT) + slow start. Reused: 1 RTT, warm cwnd.

**Q:** HTTP keep-alive vs TCP keepalive?
**A:** HTTP keep-alive reuses a connection for many requests. TCP keepalive sends probes on idle connections to detect dead peers.

**Q:** Keep-alive idle timeout rule of thumb?
**A:** Client's idle timeout shorter than the server's, so the client never sends on a connection the server just closed.

**Q:** How do you size a connection pool?
**A:** Little's Law: request rate × time each request holds a connection, plus headroom. Not bigger than the database can use.

**Q:** The pool multiplication problem and fix?
**A:** Instances × pool size = huge connection counts. Fix: smaller pools or a shared pooler (e.g. PgBouncer).

**Q:** Why set a connection-acquire timeout?
**A:** When a dependency slows down the pool runs dry; the timeout makes requests fail fast instead of piling up.

**Q:** Why do new servers get no traffic from a pooled client?
**A:** Long-lived connections stay where they were opened. Fix: maximum connection lifetime, or per-request (L7) balancing.
