# Day 2: Networking Foundations

Every box on yesterday's diagram talks to the others over a network, and the network sets hard limits
on what any design can do. The speed of light fixes the minimum round trip between two cities. TCP
decides how fast a new connection is allowed to send. DNS decides which server a client reaches at all,
and caches that decision for minutes or hours. TLS adds round trips before the first byte of real data
moves.

You don't need to be a network engineer for system design. You need to understand **what each layer
promises, what it costs, and how it fails**, because those costs show up everywhere: in tail latency,
in failover time, in why connection pools exist, and in why HTTP/3 was built (Day 3).

By the end of today you should be able to:

1. Say what IP, TCP, UDP and application protocols each promise, and what they don't.
2. Walk through the TCP handshake, how TCP makes delivery reliable, and how a connection closes.
3. Explain the difference between flow control and congestion control, and what slow start costs.
4. Explain TCP head-of-line blocking and why it matters.
5. Say when UDP's lack of guarantees is exactly what you want.
6. Trace a DNS lookup from the client to the authoritative server, and explain how caching and TTLs
   shape failover.
7. Explain GeoDNS and anycast, and how they differ.
8. Say what a TLS handshake establishes, what it costs, and how resumption reduces that cost.
9. Explain why opening connections is expensive and how keep-alive and pooling avoid it.

---

## 1. The layers that matter

Networks are built in layers. Each layer uses the one below it and offers a slightly better service to
the one above. The textbook OSI model has seven layers; in practice four matter for system design.

```
┌─────────────────────────────────────────────────────────────┐
│ Application   HTTP, gRPC, DNS, SMTP, database wire protocols│  what the bytes mean
├─────────────────────────────────────────────────────────────┤
│ (Security)    TLS                                           │  encrypt + authenticate
├─────────────────────────────────────────────────────────────┤
│ Transport     TCP, UDP (and QUIC, built on UDP)             │  process-to-process delivery
├─────────────────────────────────────────────────────────────┤
│ Network       IP (v4, v6)                                   │  host-to-host delivery, routing
├─────────────────────────────────────────────────────────────┤
│ Link          Ethernet, Wi-Fi                               │  one hop, one physical link
└─────────────────────────────────────────────────────────────┘
```

When you send data, each layer wraps the layer above's data in its own header (**encapsulation**):

```
[ Ethernet hdr [ IP hdr [ TCP hdr [ TLS record [ HTTP request ] ] ] ] ]
```

Routers along the path only look at the IP header. Only the two endpoints look at TCP and above.

### IP: best-effort delivery between hosts

IP gets a **packet** from one host address to another, hop by hop through routers. That's all. It
promises almost nothing:

- Packets can be **lost** (a router's queue was full, so it dropped them).
- Packets can arrive **out of order** (they took different paths).
- Packets can be **duplicated**.
- Packets can be **corrupted** (detected by checksums and dropped).
- There's a maximum packet size (the **MTU**, typically 1,500 bytes on Ethernet). Larger data must be
  split.

This is a deliberate design choice. Keeping the core network simple and "dumb" made the internet
scalable; any guarantees are added at the endpoints. Remember this pattern: **an unreliable layer
underneath, with reliability built on top at the edges.** You'll see it again with queues, retries and
idempotency later in the month.

### Ports and the 4-tuple

An IP address identifies a host. A **port** (16 bits, 0 to 65,535) identifies a process on that host.
A TCP connection is identified by its **4-tuple**:

```
(source IP, source port, destination IP, destination port)
```

The server listens on a well-known port (443 for HTTPS). Each client connection uses a temporary
**ephemeral port** chosen by the client's OS. This matters later: the number of ephemeral ports is
finite, and that limits how many connections one client can open to one server.

### The speed of light is the floor

Data in fibre travels at roughly two-thirds the speed of light, about **200 km per millisecond**. Real
paths aren't straight, and every router adds a bit. Rough round-trip times (RTT):

| Path | Typical RTT |
|---|---|
| Same data centre | 0.1 to 0.5 ms |
| Same region, different data centres | 1 to 2 ms |
| Across a continent | 30 to 80 ms |
| Across an ocean | 70 to 150 ms |
| Opposite side of the world | 200 to 300 ms |

No amount of engineering removes this. What you *can* control is **how many round trips** a request
needs. Much of today's lesson is really about counting round trips.

---

## 2. TCP: a reliable byte stream on top of an unreliable network

TCP turns IP's lossy, unordered packets into what looks to the application like a **reliable, ordered
stream of bytes** between two processes. It promises:

- **Reliable delivery**: every byte arrives, or the connection reports an error.
- **In-order delivery**: bytes arrive in the order they were sent.
- **No duplicates**.
- **Flow control**: the sender won't overwhelm the receiver.
- **Congestion control**: the sender won't (deliberately) overwhelm the network.

Note what's missing: TCP has **no message boundaries**. If you write 100 bytes then 200 bytes, the
other side might read 300 bytes at once, or 50 then 250. Application protocols must add their own
framing (a length prefix, or a delimiter like HTTP's blank line).

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
that's 150 ms spent before the request has even started.

Why random ISNs? If they were predictable, an attacker could forge packets that look like part of an
existing connection. They also stop stray packets from an old connection being mistaken for the new one.

**SYN floods**: the server keeps state for every half-open connection (after SYN, before the final
ACK). An attacker sending millions of SYNs from fake addresses fills that table. The standard defence,
**SYN cookies**, encodes the needed state into the server's sequence number so the server stores
nothing until the final ACK arrives.

### How TCP makes delivery reliable

- **Sequence numbers**: every byte has a number. The receiver uses them to reorder segments and
  discard duplicates.
- **Acknowledgements (ACKs)**: the receiver tells the sender "I have everything up to byte N"
  (cumulative ACK).
- **Retransmission timeout (RTO)**: if no ACK arrives in time, resend. The timeout is computed from a
  running estimate of the RTT and its variance. After repeated timeouts it **backs off exponentially**,
  so a dead link doesn't get hammered.
- **Fast retransmit**: if the receiver keeps getting segments *after* a gap, it keeps ACKing the same
  number. Three duplicate ACKs tell the sender "something specific went missing" and it resends
  immediately without waiting for the timer.
- **Checksums** catch corruption.

The important consequence for system design: **a lost packet doesn't cause an error, it causes a
delay.** The RTO starts around 1 second on many systems for the initial SYN and is at least 200 ms on
Linux for established connections. A single dropped packet can turn a 2 ms request into a 200+ ms one.
This is one of the main sources of the tail latency you met on Day 1.

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

The side that closes first enters **TIME_WAIT** and keeps the 4-tuple reserved for a while. This
exists so that delayed packets from the old connection can't be mistaken for a new connection with the
same 4-tuple, and so the final ACK can be resent if it was lost.

Why you care: a client that opens and closes many short connections to the **same server** piles up
TIME_WAIT entries, each holding an ephemeral port. Linux's default ephemeral range is about 28,000
ports. With a 60-second TIME_WAIT, that caps a client at roughly `28,000 / 60 ≈ 470` new connections
per second to one destination before it runs out of ports. This is **ephemeral port exhaustion**, and
the real fix is to stop opening so many connections (section 7).

### Flow control: don't overwhelm the receiver

The receiver has a finite buffer. If the application reads slowly, the buffer fills. In every ACK the
receiver advertises a **receive window (rwnd)**: "I have room for this many more bytes." The sender
never has more than rwnd unacknowledged bytes in flight. If rwnd drops to zero, the sender stops and
periodically probes until space opens up.

Flow control protects **the receiving process**. It's a conversation between the two endpoints only.

### Congestion control: don't overwhelm the network

The receiver might have plenty of room, but some router in the middle might not. If every sender blasts
at full speed, router queues overflow, packets drop, everyone retransmits, and throughput collapses (the
internet actually suffered **congestion collapse** in 1986, which is why this mechanism exists).

The sender keeps a second limit, the **congestion window (cwnd)**, which is its own estimate of how much
the network can take. Nobody tells it this number; it infers it from loss (and, in newer algorithms,
from delay).

```
bytes in flight ≤ min(rwnd, cwnd)
                       │      │
     receiver's limit ─┘      └─ network's limit (sender's guess)
```

**Slow start.** A new connection doesn't know the network's capacity, so it starts small (typically
**10 segments, about 14 KB**) and roughly **doubles cwnd every round trip** while ACKs keep coming
back:

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
lets many connections share a link fairly.

Modern algorithms refine this. **CUBIC** (the Linux default) grows faster after recovering from loss.
**BBR** (from Google) models the bottleneck bandwidth and RTT directly instead of waiting for loss,
which works better on links with deep buffers or random loss (like mobile networks).

**Why slow start matters for system design:**

- A **new connection is slow at first**, regardless of bandwidth. Transferring 1 MB on a fresh
  connection at 100 ms RTT takes several round trips just to ramp up: 14 KB, 28 KB, 56 KB, 112 KB,
  224 KB, 448 KB... about 7 round trips, so **~700 ms**, even on a gigabit link.
- A **reused, warm connection** already has a large cwnd and sends at full speed immediately. This is
  a big reason connection reuse matters (section 7).
- On many systems, cwnd also resets after a connection sits idle for a while, so even keep-alive
  connections can lose their warm-up.

### Bandwidth-delay product

To keep a pipe full you need enough data in flight to cover a full round trip:

```
bytes in flight needed = bandwidth × RTT

Example: 1 Gbps × 100 ms = 100 Mbit = 12.5 MB in flight
```

If either window (rwnd or cwnd) is smaller than this, you can't use the full bandwidth. Max throughput
on one connection is roughly `window / RTT`. The original TCP window field allowed only 64 KB, which
at 100 ms RTT caps one connection at about **5 Mbps** no matter how fat the link. The **window scaling**
option fixed this. The general lesson: **on long, fast links, a single TCP connection is limited by
RTT**, not bandwidth. That's why large transfers often use several parallel connections.

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
download). It's harmful when you send **several independent things over one connection**: a lost
packet belonging to resource A stalls resources B and C as well, even though their bytes already
arrived. HTTP/2 multiplexes many requests over a single TCP connection and gets hit by exactly this on
lossy networks. Solving it required leaving TCP behind (HTTP/3 over QUIC, Day 3).

### Nagle's algorithm and delayed ACKs

Two old optimizations to avoid sending lots of tiny packets:

- **Nagle's algorithm** (sender): if there's unacknowledged data in flight, hold small writes and
  bundle them until the ACK arrives.
- **Delayed ACK** (receiver): wait briefly (up to ~40 ms on Linux) before ACKing, hoping to piggyback
  the ACK on a response.

Each is reasonable alone. Together they can deadlock briefly: the sender waits for an ACK before
sending, the receiver waits for more data before ACKing, and you get mysterious ~40 ms stalls on
request/response protocols. This is why latency-sensitive software sets **`TCP_NODELAY`** (disables
Nagle). It's Day 1's batching trade-off again: Nagle buys throughput with latency.

---

## 3. UDP: when unreliability is a feature

UDP is barely more than raw IP with ports added. Its header is 8 bytes. It offers:

- **No connection**: no handshake. The first packet can carry data.
- **No reliability**: lost packets are just gone.
- **No ordering**: packets arrive in whatever order they arrive.
- **No congestion or flow control**: it sends as fast as you tell it to.
- **Message boundaries are kept**: one send is one datagram on the other side (unlike TCP's stream).

That sounds strictly worse. It isn't, because TCP's guarantees have costs, and some applications don't
want to pay them:

| Use case | Why UDP fits |
|---|---|
| **DNS** | One small question, one small answer. A TCP handshake would triple the round trips. If the reply is lost, just ask again. |
| **Voice and video calls** | A late packet is useless: audio from 300 ms ago can't be played now. TCP would stall everything waiting for a retransmit (HOL blocking). Better to skip the lost bit and keep going. |
| **Online games** | Only the latest position matters. A retransmitted old position is worse than nothing. |
| **Metrics (e.g. StatsD)** | Fire-and-forget. Losing one sample of a counter is fine; blocking the app to send it isn't. |
| **QUIC / HTTP/3** | Builds its *own* reliability, ordering and congestion control on top of UDP, but per stream, avoiding TCP's HOL blocking (Day 3). |

The key insight: **UDP lets the application decide which guarantees it needs.** TCP's guarantees are
all-or-nothing and built into the OS kernel, which makes them hard to change. Building on UDP lets you
pick exactly what you need and evolve it in application code. That's precisely why QUIC is built on
UDP rather than being a new kernel protocol.

What you give up, and must handle yourself if you need it: retransmission, ordering, duplicate
detection, fragmenting large messages, and congestion control. An application blasting UDP with no
congestion control harms everyone else on the link.

A security note: because UDP has no handshake, the source address can be **spoofed**. Attackers send a
small request to a UDP service (DNS, NTP, memcached) with the victim's address as the source, and the
service sends a much larger reply to the victim. This is an **amplification attack**. TCP's handshake
prevents it, because the spoofed address never completes the handshake.

---

## 4. DNS: names to addresses

Humans and config files use names (`api.example.com`); IP needs addresses (`93.184.215.14`). DNS is the
globally distributed, heavily cached database that maps between them. It's also, quietly, one of the
most important **traffic-steering** tools you have.

### The hierarchy

DNS is a tree, and responsibility is **delegated** down it:

```
                         . (root)
           ┌─────────────┼──────────────┐
          com           org             uk         ← TLD servers
     ┌─────┴─────┐
  example      google                              ← each domain's authoritative servers
     │
  api.example.com → 93.184.215.14                  ← the actual record
```

- **Root servers** know where the TLD servers are. There are 13 root server *names* (a to m), but
  each is served by many machines around the world via anycast (section 5).
- **TLD servers** (`.com`, `.org`, `.uk`) know which name servers are authoritative for each domain.
- **Authoritative servers** for a domain hold the actual records. When you configure DNS at a
  provider, you're editing records on its authoritative servers.

### Recursive vs authoritative

Two very different roles:

- An **authoritative server** answers only for the zones it owns, and it gives the definitive answer.
  It doesn't go looking for anything else.
- A **recursive resolver** (your ISP's resolver, or public ones like `1.1.1.1` and `8.8.8.8`, or your
  company's internal resolver) does the legwork on the client's behalf: it walks the tree, caches
  everything it learns, and returns the final answer.

Your laptop runs a **stub resolver**: it just asks the recursive resolver and waits.

### The resolution path

A cold lookup of `api.example.com`, with nothing cached anywhere:

```
┌────────┐ 1. api.example.com?  ┌───────────────┐ 2. api.example.com?  ┌──────────┐
│ Client │─────────────────────▶│   Recursive   │─────────────────────▶│   Root   │
│ (stub) │                      │   resolver    │◀─ "ask .com servers" ─└──────────┘
└────────┘                      │               │
     ▲                          │               │ 3. api.example.com?  ┌──────────┐
     │                          │               │─────────────────────▶│ .com TLD │
     │                          │               │◀ "ask ns1.example.." ─└──────────┘
     │                          │               │
     │                          │               │ 4. api.example.com?  ┌───────────────┐
     │                          │               │─────────────────────▶│ Authoritative │
     │                          │               │◀── 93.184.215.14 ────│ (example.com) │
     │ 5. 93.184.215.14         │               │     TTL 300          └───────────────┘
     └──────────────────────────│ caches each   │
                                │ answer by TTL │
                                └───────────────┘
```

The client asks one question and gets one answer (a **recursive query**). The resolver asks a series of
questions and follows referrals (**iterative queries**). In practice the root and TLD answers are almost
always already cached, so a typical miss costs one trip from the resolver to the authoritative server.

### Caching and TTLs

Every record carries a **TTL** (time to live, in seconds). Anyone who caches the answer may reuse it
until the TTL expires. Caches exist at many levels: the browser, the OS, the recursive resolver, and
sometimes the application runtime itself.

Caching is what makes DNS scale to the whole internet. It also creates the central trade-off:

| | Long TTL (hours) | Short TTL (30–60 s) |
|---|---|---|
| Lookup latency | Mostly cache hits, fast | More misses, slower |
| Load on authoritative servers | Low | High |
| Resilience if your DNS provider is down | Cached answers keep working for a while | Clients fail sooner |
| How fast a change takes effect | Slow: old answers linger up to the TTL | Fast |
| Usefulness for failover | Poor | Better |

Things that make DNS changes slower than the TTL suggests:

- Some resolvers and clients **ignore or clamp TTLs** (enforcing a minimum, or caching longer).
- Some **applications cache resolved addresses forever** (certain language runtimes and connection
  pools resolve once at startup and never again).
- **Open connections don't re-resolve at all.** A client with a keep-alive connection to the old IP
  keeps using it until the connection closes.
- **Negative caching**: "this name doesn't exist" (NXDOMAIN) is cached too, for a time set by the
  zone. If you query a name before you create it, the "doesn't exist" answer can stick around.

**Lesson: DNS-based failover is coarse and slow.** It's great for steering traffic between regions over
minutes. It's not a substitute for a load balancer that removes a dead server in seconds (Day 4). A
common practice is to **lower the TTL well in advance** of a planned migration (at least one old-TTL
period before), make the change, then raise it again.

### Record types you'll meet

| Type | Maps | Notes |
|---|---|---|
| **A** | name → IPv4 address | |
| **AAAA** | name → IPv6 address | |
| **CNAME** | name → another name | "Look up that name instead." Adds another lookup. Not allowed at the zone apex (`example.com` itself), which is why providers offer non-standard **ALIAS/ANAME** records. |
| **NS** | zone → its authoritative name servers | How delegation works. |
| **MX** | domain → mail servers | |
| **TXT** | name → arbitrary text | Domain ownership proofs, email security (SPF, DKIM). |
| **SRV** | service → host + port | Used by some service-discovery systems. |

A name can return **several A records**. Clients usually try them in some order, which gives crude
load spreading (**round-robin DNS**). It's crude because DNS doesn't know whether a server is healthy
or busy, and caches hand the same order to many clients.

### DNS as a traffic-steering tool

Because the authoritative server chooses what to answer, it can give **different answers to different
clients**:

- **GeoDNS**: answer based on where the query seems to come from. European users get the European
  data centre's IP, Asian users get the Asian one.
- **Latency-based routing**: answer with the region that has the lowest measured latency from the
  client's network.
- **Weighted routing**: send 90% of answers to one target and 10% to another (useful for gradual
  migrations).
- **Health-checked failover**: stop handing out an IP whose health check fails.

A catch with GeoDNS: the authoritative server sees the **resolver's** address, not the client's. If a
user in Mumbai uses a public resolver whose nearest instance is in Singapore, they look like they're in
Singapore. The **EDNS Client Subnet** extension lets resolvers pass along part of the client's address
to fix this, at some cost to privacy and cache efficiency.

---

## 5. Anycast

GeoDNS gives different *addresses* to different users. **Anycast** gives everyone the **same address**,
and the network itself delivers each user to the nearest location.

How it works: many sites around the world announce the same IP prefix to the internet's routing system
(**BGP**). Each router picks what it considers the shortest path to that prefix, so packets from Paris
naturally land at the Paris site and packets from Tokyo at the Tokyo site.

```
           Same IP 192.0.2.1 announced from every site

   User in Paris ──▶ router ──▶ ┌──────────────┐
                                │ Site: Paris  │
                                └──────────────┘
   User in Tokyo ──▶ router ──▶ ┌──────────────┐
                                │ Site: Tokyo  │
                                └──────────────┘
   User in NYC   ──▶ router ──▶ ┌──────────────┐
                                │ Site: NYC    │
                                └──────────────┘
```

| | GeoDNS | Anycast |
|---|---|---|
| Who decides the location | Your DNS server, at lookup time | Internet routing (BGP), per packet |
| Address users see | Different per region | One address everywhere |
| Reaction to a site failing | Limited by DNS TTLs and client caching | Site stops announcing the route; traffic shifts in seconds to minutes as routes update |
| Precision | Based on the resolver's location (may be off) | "Nearest" in routing terms, which usually but not always means geographically near |
| Typical users | Multi-region apps steering to regional data centres | DNS root and public resolvers, CDNs, DDoS protection |

Anycast is also a strong **DDoS defence**: attack traffic is absorbed by whichever site is closest to
each attacker, instead of all landing on one place.

The subtle part: routes can change mid-conversation, so packets from one TCP connection could suddenly
start arriving at a *different* site that has no state for that connection. That's why anycast was
first used for short, stateless UDP exchanges like DNS. In practice routes are stable enough that CDNs
run TCP over anycast successfully, but it's a real constraint to be aware of.

---

## 6. TLS: securing the connection

TCP delivers bytes reliably but in plain text, and anyone on the path can read or alter them. **TLS**
(Transport Layer Security; its predecessor was SSL) sits between TCP and the application and provides
three things:

1. **Confidentiality**: an observer on the path can't read the data.
2. **Integrity**: tampering is detected; modified data is rejected.
3. **Authentication**: the client knows it's talking to the real `api.example.com`, not an impostor.
   (Optionally the server can verify the client too: **mutual TLS**, common between internal services.)

### What the handshake establishes

Before encrypted data can flow, the two sides must agree on:

- the **TLS version and cipher suite** (which algorithms to use);
- the **server's identity**, proven by its certificate;
- a set of **shared secret keys**, which nobody watching the exchange can work out.

**Authentication** uses certificates. The server presents a certificate binding its name to a public
key, signed by a **certificate authority (CA)**, often through a chain of intermediate certificates.
The client checks that the chain leads to a root CA it already trusts (built into the OS or browser),
that the name matches, and that the certificate hasn't expired. The server then proves it holds the
matching private key by signing part of the handshake.

**Key exchange** uses (Elliptic Curve) **Diffie–Hellman**: each side generates a temporary key pair and
sends the public half; combining their own private half with the other's public half gives both sides
the same secret, which an eavesdropper can't compute. Because these key pairs are temporary
(**ephemeral**) and thrown away, stealing the server's long-term private key later doesn't let an
attacker decrypt past recorded traffic. That property is **forward secrecy**.

From then on, data is protected with fast **symmetric encryption** (such as AES-GCM or ChaCha20) using
the agreed keys. Expensive public-key maths is only used in the handshake.

**SNI** (Server Name Indication): the client says which hostname it wants *in the first handshake
message*, so one IP address can host many sites with different certificates. This is how load
balancers and CDNs serve thousands of domains from shared IPs.

### The cost: round trips and CPU

```
TCP + TLS 1.2 (fresh)                       TCP + TLS 1.3 (fresh)

Client                    Server            Client                    Server
  │── SYN ───────────────▶│                   │── SYN ───────────────▶│
  │◀──────────── SYN-ACK ─│  1 RTT            │◀──────────── SYN-ACK ─│  1 RTT
  │── ACK, ClientHello ──▶│                   │── ACK, ClientHello ──▶│
  │◀── ServerHello, cert ─│  2 RTT            │   (+ key share)       │
  │── key exchange, Fin ─▶│                   │◀── ServerHello, cert, │  2 RTT
  │◀─────────────── Fin ──│  3 RTT            │    Fin (+ key share) ─│
  │── HTTP request ──────▶│                   │── Fin, HTTP request ─▶│
  │◀───── HTTP response ──│  4 RTT            │◀───── HTTP response ──│  3 RTT
```

- **TLS 1.2** needs **2 round trips** of handshake on top of TCP's one.
- **TLS 1.3** cut that to **1 round trip**: the client guesses the key-exchange method and sends its
  key share in the very first message.

At 100 ms RTT, a fresh HTTPS request over TLS 1.2 spends 300 ms on setup before the request is even
sent. TLS 1.3 saves 100 ms of that.

There's CPU cost too. The public-key operations in the handshake (certificate signature, key exchange)
are far more expensive than encrypting bulk data, which modern CPUs do in hardware. A server handling
many *new* connections spends noticeable CPU on handshakes; a server with long-lived connections
barely notices TLS.

### Session resumption

If a client has talked to a server recently, it can skip most of the handshake:

- **Session IDs** (TLS 1.2): the server remembers the session's keys under an ID; the client presents
  the ID next time. Downside: the server must store state, and a different server behind a load
  balancer won't have it.
- **Session tickets** (TLS 1.2) / **pre-shared keys** (TLS 1.3): the server encrypts the session state
  with a key only servers know and hands it to the client to store. Any server sharing that ticket key
  can resume, which works well behind a load balancer.

Resumption saves the certificate exchange and the expensive public-key work.

**0-RTT** (TLS 1.3 "early data"): on resumption, the client can send application data *in its very
first message*, before the handshake finishes. The catch is serious: an attacker who captures that
first message can **replay** it, and the server may not be able to tell. So 0-RTT is only safe for
**idempotent** requests (a GET that just reads data), never for "transfer $100". You'll meet
idempotency properly on Days 3 and 23.

### Where TLS ends

TLS is often **terminated** at the load balancer or CDN edge: the client's TLS connection ends there,
and traffic continues to backend servers either in plain text inside a trusted network or over a
separate TLS connection. Terminating close to the user means the expensive handshake round trips are
short ones. Re-encrypting to the backend (or using mutual TLS between services) is increasingly the
default, because "the internal network is trusted" has repeatedly proven untrue.

---

## 7. Connection reuse: keep-alive and pooling

Add up what a **brand-new HTTPS connection** costs a client 100 ms away from the server:

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
warmed up. And the server and client both save CPU (handshakes, crypto, kernel bookkeeping), memory
(per-connection buffers and state), and ephemeral ports / TIME_WAIT entries.

### Keep-alive

**HTTP keep-alive** (persistent connections): instead of closing the TCP connection after each
response, leave it open and send the next request over it. It's the default in HTTP/1.1 and later.
Both sides close it after an idle timeout.

Pitfalls:

- **Idle timeout mismatches.** If the server (or a load balancer or NAT in between) closes an idle
  connection while the client still thinks it's open, the client's next request fails with a reset.
  Rule of thumb: the **client's idle timeout should be shorter than the server's**, so the client
  always gives up first.
- Don't confuse **HTTP keep-alive** with **TCP keepalive**: the latter is an OS feature that sends
  small probe packets on idle connections to detect a dead peer (and keep NAT and firewall mappings
  alive).

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
connections. Opening one involves TCP, often TLS, then authentication and session setup. Some
databases dedicate significant memory per connection; PostgreSQL, for instance, starts a whole
operating-system **process** for each one. A database that can comfortably run a few hundred
connections can fall over at a few thousand.

**Sizing a pool** uses Little's Law from Day 1:

```
connections needed ≈ request rate to the dependency × time each request holds a connection

500 queries/s × 10 ms each = 5 connections busy on average
```

Then add headroom for bursts. Bigger isn't better: the database has a fixed number of CPU cores and
disks. Past that, extra connections just queue *inside* the database, competing for locks and memory,
and throughput often goes **down**. A small pool with a short queue in front of it usually beats a huge
pool.

**The multiplication problem.** Pools are per process. 50 app instances × 20 connections each = 1,000
database connections, even if most are idle. As you scale the stateless tier out, you can overwhelm
the stateful tier behind it. The fix is a **shared pooler** (a proxy such as PgBouncer) that many app
instances connect to, which in turn holds a small number of real connections to the database.

Other pooling pitfalls:

- **Stale connections**: the server, a firewall, or a NAT silently dropped an idle connection. Pools
  validate connections on borrow, or retire them after a maximum lifetime.
- **Pool exhaustion**: if a dependency slows down, each request holds its connection longer, so the
  pool runs dry and requests queue for a connection. Always set a **timeout for acquiring** a
  connection, so a slow dependency causes fast errors instead of a pile-up (Day 26).
- **Load imbalance**: long-lived connections stick to whichever backend they first reached. When you
  add a new server behind a load balancer, existing pools don't move to it. Setting a maximum
  connection lifetime makes connections slowly redistribute.
- **DNS changes are invisible** to already-open pooled connections (section 4).

---

## 8. Putting it together: what happens when you open a URL

`https://api.example.com/orders/42`, from a cold start:

```
 1. DNS        Browser cache? OS cache? → recursive resolver → (root → .com) → authoritative
               → 203.0.113.10, TTL 60.    Answer may be GeoDNS-chosen or an anycast address.
 2. TCP        SYN → SYN-ACK → ACK                         1 RTT
 3. TLS 1.3    ClientHello (SNI, key share) → ServerHello,
               cert, Finished → client verifies the cert   1 RTT
 4. HTTP       GET /orders/42 → response                   1 RTT (+ slow start if large)
 5. Reuse      Connection stays open (keep-alive); the next request costs 1 RTT
```

Every technique in this lesson either removes one of these round trips or makes it shorter:

| Technique | What it saves |
|---|---|
| DNS caching | The DNS lookup |
| Keep-alive / pooling | TCP + TLS handshakes and slow start, on every request after the first |
| TLS 1.3 | One handshake RTT |
| Session resumption / 0-RTT | Certificate work, and with 0-RTT one more RTT |
| CDN edge / anycast / GeoDNS | Makes every RTT shorter by moving the server closer |
| TLS termination at the edge | Handshake RTTs happen over the short user-to-edge path |
| QUIC / HTTP/3 (Day 3) | Combines transport and TLS handshakes; removes TCP HOL blocking |

---

## 9. How it connects

- **IP** is unreliable by design; **TCP** builds reliability at the endpoints. Loss becomes *delay*
  (retransmission), which feeds directly into **tail latency** (Day 1).
- **Congestion control and slow start** make new connections slow, which is why **connection reuse**
  is such a big deal.
- **Head-of-line blocking** in TCP is the reason HTTP/3 moved to QUIC over UDP (Day 3).
- **DNS caching and TTLs** set the speed limit for DNS-based failover; load balancers (Day 4) handle
  the fast, fine-grained part.
- **Anycast and GeoDNS** are the foundations CDNs build on (Day 7).
- **Connection pools** are a concurrency limit, sized with Little's Law, and their exhaustion is a
  classic failure mode under slow dependencies (Day 26).
- The pattern "unreliable layer below, guarantees built on top" recurs with message delivery and
  idempotency (Days 22–23).

## Common misconceptions

- **"TCP guarantees delivery."** It guarantees delivery *or an error*. If the network is broken, data
  doesn't arrive; you get a timeout or a reset. And an ACK only means the peer's *kernel* received the
  bytes, not that the application processed them.
- **"More bandwidth makes everything faster."** For small requests, latency is dominated by round
  trips, and RTT is bound by distance. A fatter pipe doesn't shorten the handshake.
- **"UDP is just unreliable TCP."** It's a minimal building block that lets applications choose their
  own guarantees. QUIC is built on it precisely for that reason.
- **"Changing DNS takes effect in TTL seconds."** Only for well-behaved caches. Clamped TTLs,
  application-level caching and open connections can keep traffic going to the old address for much
  longer.
- **"Flow control and congestion control are the same thing."** Flow control protects the receiver
  (rwnd); congestion control protects the network (cwnd). The sender obeys the smaller of the two.
- **"TLS is too slow to use internally."** Bulk encryption is cheap on modern hardware; the cost is
  mostly in handshakes, which connection reuse amortizes.
- **"A bigger connection pool means more throughput."** Past the database's real capacity, extra
  connections add contention and reduce throughput.

---

## Further reading (supplements)

- Ilya Grigorik, *High Performance Browser Networking* (free online): chapters on TCP, UDP and TLS are
  the best practical treatment of today's material
- Julia Evans's zines and blog posts on DNS and networking: short, clear mental models
- Cloudflare Learning Center: articles on DNS, anycast and TLS handshakes
- Hussein Nasser (YouTube): TCP, TLS and connection pooling deep dives
- RFC 9293 (TCP) and RFC 8446 (TLS 1.3), if you want the authoritative source for a detail
