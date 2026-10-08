# Day 2: Answers

Read only after attempting `questions.md`.

---

**1.** Packets can be lost (dropped at a full router queue), arrive out of order (different paths),
be duplicated, or be corrupted (detected by checksum and dropped). Also, packets larger than the MTU
must be split. Keeping IP weak keeps the core network simple and fast: routers just forward packets
and hold no per-connection state. Guarantees are added at the endpoints only by the applications that
need them, which let the internet scale and let new protocols be built without changing routers.

**2.** No. Small API calls are bound by **round trips**, not bandwidth. A 2 KB request and response
barely use 1 Gbps; the time is dominated by the 200 ms RTT, multiplied by however many round trips
each call needs (handshakes included). What helps: **reuse connections** (no repeated TCP/TLS
handshakes), **reduce the number of round trips** per operation (batch calls, avoid chatty
sequential calls), or **move the data closer** (cache or replicate it near Mumbai, or move the
dependency).

**3.** Client sends SYN with its initial sequence number (ISN) x. Server replies SYN-ACK with its own
ISN y and acknowledges x+1. Client sends ACK of y+1. Each side learns the other's starting sequence
number (needed to order bytes and detect duplicates) and that the other side is reachable and willing.
The server allocates connection state. Cost: **one full RTT** before the client can send data (the
data can ride along with the final ACK).

**4.** The application sees **no error, just a delay**. The sender doesn't get an ACK, so it
retransmits, either after three duplicate ACKs (fast retransmit, about one RTT) or after a
retransmission timeout (at least 200 ms on Linux, often more). Meanwhile, later bytes wait due to
in-order delivery. A request that normally takes 2 ms takes 200+ ms. These occasional retransmissions
are a classic source of p99/p99.9 latency: rare enough to vanish in the average, common enough to
dominate the tail, especially with fan-out.

**5.** **Flow control** protects the **receiver**: the receiver advertises how much buffer space it
has (rwnd), so a fast sender doesn't overrun a slow reader. **Congestion control** protects the
**network**: the sender keeps its own estimate (cwnd) of how much the path can carry, inferred from
loss or delay, so it doesn't flood router queues. The sender keeps no more than
**`min(rwnd, cwnd)`** bytes unacknowledged in flight.

**6.** Slow start. A new connection starts with a small congestion window (~10 segments, ~14 KB) and
roughly doubles it each RTT: 14, 28, 56, 112, 224, 448 KB... Getting 1 MB through takes about **7
round trips ≈ 700 ms**, while `1 MB / 1 Gbps` is only 8 ms. Bandwidth is irrelevant until the window
grows. Faster: reuse a **warm connection** (cwnd already large), raise the initial window, or shorten
the RTT (serve from closer, e.g. a CDN).

**7.** The **window**. One connection can have at most a window's worth of unacknowledged data in
flight per round trip, so `max throughput ≈ window / RTT`. To fill the pipe you need
`window ≥ bandwidth × RTT` (the **bandwidth-delay product**). With a 64 KB window and 80 ms RTT,
that's 64 KB / 0.08 s ≈ 800 KB/s ≈ 6.5 Mbps, whatever the link speed. Fixes: TCP window scaling and
larger socket buffers, or several parallel connections.

**8.** TCP delivers bytes strictly in order. If segment 3 is lost, segments 4–6 can arrive but the
application can't read them until 3 is retransmitted. For one file, you need all bytes in order
anyway, so nothing is lost by waiting. When many independent requests share one connection (like
HTTP/2 multiplexing), a packet lost from request A blocks requests B and C too, even though their data
already arrived. One loss stalls everything, which is why HTTP/3 moved to QUIC, which orders data per
stream.

**9.** **Ephemeral port exhaustion** from TIME_WAIT. Each closed connection leaves its 4-tuple in
TIME_WAIT (60 s on Linux) on the side that closed first. To one destination IP and port, the only
thing that varies is the client's ephemeral port, and Linux has about 28,000 by default.
`28,000 / 60 s ≈ 470` new connections per second is the ceiling; at 1,000/s the client runs out of
ports. The right fix is **connection reuse** (keep-alive / pooling), which also removes a handshake
from every request. Widening the port range or tweaking TIME_WAIT settings only moves the ceiling.

**10.** **Nagle's algorithm interacting with delayed ACKs.** The sender's Nagle logic holds a small
write until outstanding data is acknowledged; the receiver delays its ACK (up to ~40 ms) hoping to
piggyback it on a response. Each waits for the other until the delayed-ACK timer fires. Fix: set
**`TCP_NODELAY`** on the socket to disable Nagle (and write each message in one call rather than
several small writes).

**11.** In a live call, **late data is useless**. Audio from 300 ms ago can't be played now. With TCP,
one lost packet would stall all following audio until it's retransmitted (head-of-line blocking),
causing freezes and growing delay. With UDP the app skips or conceals the missing bit and keeps
playing current audio. The app can also adapt its bitrate itself.

**12.** DNS is usually one small question and one small answer. UDP needs no handshake, so a lookup
is a single round trip instead of two or more, and the server keeps no per-client connection state,
which matters for servers answering huge numbers of clients. If a response is lost, the **client
times out and retries** (possibly to another server). The application provides the reliability it
needs. (DNS falls back to TCP for large responses and zone transfers.)

**13.** QUIC implements its own **reliability, ordering, and congestion control in user space**, on
top of UDP datagrams: sequence numbers, ACKs, retransmissions, all per stream, so a loss in one stream
doesn't block others. Building on UDP was practical: TCP lives in OS kernels and in countless
middleboxes (firewalls, NATs) that inspect and sometimes interfere with transport headers. A brand-new
transport protocol would be blocked by many of them and would take years to roll out across operating
systems. UDP already passes through nearly everything, and QUIC can ship and evolve inside
applications.

**14.** UDP has **no handshake**, so the source address isn't verified. An attacker sends a small
request with the victim's IP forged as the source, and the server sends a much larger response to the
victim. Many servers multiply the attacker's bandwidth. With TCP, the server's SYN-ACK goes to the
forged address, the handshake never completes, and no large response is ever sent.

**15.** A **recursive resolver** does the full lookup on a client's behalf, walking from root to TLD
to authoritative servers, caching what it learns, and returning a final answer. An **authoritative
server** only answers for the zones it owns and gives the definitive record. Your laptop's stub
resolver talks to a **recursive resolver** (ISP, corporate, or a public one like 1.1.1.1).

**16.** **Lower the TTL in advance**: reduce it to something like 60 s, then wait **at least one full
old TTL (a day)** so every cache holding the long TTL has expired. Then make the change, which now
spreads within about a minute. Once stable, raise the TTL again. Keep the old servers running for a
while anyway, because traffic can still reach them: some resolvers ignore or clamp TTLs; some
applications cache DNS results for a long time or until restart; and **open keep-alive or pooled
connections keep using the old IP** without re-resolving.

**17.** DNS failover is **slow and imprecise**. After you remove the IP, clients keep using cached
answers until the TTL expires, longer for caches that ignore TTLs, and existing connections stay on
the dead server. A very low TTL helps but increases lookup latency and load, and still doesn't fix
misbehaving caches. DNS also has no idea about per-request health. Use a **load balancer with health
checks** for fast failover between servers (seconds), and DNS for coarse steering between regions.

**18.** GeoDNS sees the **recursive resolver's** IP address, not the user's. If the user uses a
public resolver whose nearest instance is in Singapore (or a corporate resolver located elsewhere),
the authoritative server thinks the user is in Singapore. The **EDNS Client Subnet** extension lets
the resolver pass part of the user's address so the authoritative server can choose correctly.

**19.** **GeoDNS** hands *different IP addresses* to different users based on their (resolver's)
location, at lookup time. **Anycast** gives *everyone the same IP*, announced via BGP from many sites,
and internet routing delivers each packet to the nearest site. **Anycast reacts faster**: a failing
site stops announcing the route, and routers shift traffic to the next-nearest site as routes update,
with no DNS caches involved. GeoDNS failover waits on TTLs and client caching.

**20.** **Confidentiality** (nobody on the path can read data): the key exchange (Diffie–Hellman)
produces shared keys that drive symmetric encryption. **Integrity** (tampering is detected):
authenticated encryption (e.g. AES-GCM) attaches a tag to every record; any modification fails
verification. **Authentication** (you're talking to the real server): the server's certificate, which
chains to a trusted CA, plus the server proving it holds the matching private key by signing the
handshake.

**21.** **Forward secrecy** means that stealing a server's long-term private key later doesn't let an
attacker decrypt traffic they recorded earlier. It comes from using **ephemeral** Diffie–Hellman key
pairs for each session: the session keys come from temporary keys that are thrown away, and the
long-term key is only used to *sign* (authenticate), never to derive the session keys.

**22.** In TLS 1.3 the client **guesses** the key-exchange method and sends its key share in the very
first message (ClientHello), so the server can complete the key agreement in its first reply. That
cuts the handshake to 1 RTT instead of 2. **0-RTT** (on resumption) lets the client send application
data together with its first message, using keys from a previous session. Because that first message
can be captured and **replayed** by an attacker, it's only safe for **idempotent** requests (repeating
them changes nothing), like reading a page, and never for actions like payments.

**23.** With **session IDs**, the session state lives on the server that created it. The next
connection may be routed to one of the other 19 servers, which has no record of it, so a full
handshake is needed (unless servers share a session cache). With **session tickets**, the client
carries the encrypted session state itself. Any server holding the shared ticket encryption key can
decrypt it and resume, wherever the load balancer sends the connection. (It's Day 1's
stateless-vs-stateful theme again.)

**24.** Cold: DNS lookup (up to an RTT or more on a miss) + TCP handshake (1 RTT) + TLS 1.3 (1 RTT) +
request/response (1 RTT) = **3 to 4 RTTs**, plus extra RTTs of slow start if the response is large.
Reused: **1 RTT**, with an already-warm congestion window. Reuse also saves **CPU** (expensive
public-key operations in handshakes), **memory and kernel state** for connections, and **ephemeral
ports / TIME_WAIT entries**. On a database it also saves authentication and per-connection setup (in
Postgres, a whole new process).

**25.** `40 × 25 = 1,000` database connections. Most are idle, but each costs the database memory (a
process per connection in Postgres), and when many are active they compete for CPU, locks and memory
inside the database, so throughput drops. Pools multiply as the stateless tier scales out. Fixes:
**shrink each pool** (size it by Little's Law: query rate × hold time, plus headroom), and/or put a
**shared connection pooler** (like PgBouncer) between the apps and the database, so many client
connections map onto a small number of real database connections.

**26.** By Little's Law, connections in use = call rate × hold time. At 50× the hold time, the pool
needs 50× the connections, so it **runs dry**. New requests **queue waiting for a connection**,
holding their threads and memory; the caller's own latency spikes and it may run out of threads too,
so the slowness spreads upstream. The key setting is a **connection-acquire timeout** (plus request
timeouts on the calls themselves): requests that can't get a connection quickly fail fast instead of
piling up. Circuit breakers (Day 26) build on this.

**27.** The client's pool holds **long-lived connections** that were opened before the new servers
existed. The load balancer only makes a routing decision when a connection is *opened* (for L4
balancing especially), so existing connections stay on the old servers. New servers only get traffic
as new connections open. Fix: set a **maximum connection lifetime** in the pool so connections are
periodically recycled and spread across all servers, or use a load balancer that balances per request
(L7, Day 4).
