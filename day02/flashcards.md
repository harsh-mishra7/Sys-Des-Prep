# Day 2: Flashcards

Cover the answer, say it out loud, then check.

---

**Q:** What does IP promise?
**A:** Best-effort delivery of packets between hosts. Packets may be lost, reordered, duplicated or corrupted.

**Q:** What identifies a TCP connection?
**A:** The 4-tuple: (source IP, source port, destination IP, destination port).

**Q:** Roughly how fast does data travel in fibre?
**A:** About 200 km per millisecond (two-thirds the speed of light).

**Q:** What does TCP promise that IP doesn't?
**A:** Reliable, in-order, duplicate-free byte stream, plus flow control and congestion control.

**Q:** Does TCP preserve message boundaries?
**A:** No. It's a byte stream; applications must add their own framing (length prefix or delimiter).

**Q:** What's the cost of the TCP handshake?
**A:** One RTT before the client can send data.

**Q:** Why are TCP initial sequence numbers random?
**A:** To stop attackers forging packets into a connection, and to avoid confusing packets from an old connection.

**Q:** What's a SYN flood, and the standard defence?
**A:** Flooding a server with SYNs to fill its half-open connection table. Defence: SYN cookies (store no state until the final ACK).

**Q:** What does a dropped packet look like to a TCP application?
**A:** A delay (retransmission after fast retransmit or a timeout of 200 ms or more), not an error. A big source of tail latency.

**Q:** What triggers fast retransmit?
**A:** Three duplicate ACKs, signalling a specific missing segment.

**Q:** What is TIME_WAIT and who enters it?
**A:** The side that closes first keeps the 4-tuple reserved (60 s on Linux) so stray old packets can't corrupt a new connection.

**Q:** What's ephemeral port exhaustion?
**A:** Opening and closing many connections to one destination fills TIME_WAIT and uses up the ~28k client ports. Fix: reuse connections.

**Q:** Flow control vs congestion control?
**A:** Flow control (rwnd) protects the receiver's buffer. Congestion control (cwnd) protects the network. Send ≤ min(rwnd, cwnd).

**Q:** What is slow start?
**A:** A new connection starts with a small cwnd (~10 segments, ~14 KB) and doubles it each RTT until loss or a threshold.

**Q:** What is AIMD?
**A:** Additive increase, multiplicative decrease: grow cwnd linearly, halve it on loss. Lets connections share a link fairly.

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

**Q:** Recursive resolver vs authoritative server?
**A:** Recursive does the lookup for clients and caches. Authoritative holds the definitive records for its zones.

**Q:** DNS resolution path for a cold lookup?
**A:** Stub → recursive resolver → root → TLD → authoritative → answer cached by TTL at each level.

**Q:** The DNS TTL trade-off?
**A:** Long TTL: fast lookups, low load, slow changes. Short TTL: fast changes, more lookups and load.

**Q:** How do you safely change a DNS record with a 1-day TTL?
**A:** Lower the TTL, wait at least one old TTL, change the record, then raise the TTL again.

**Q:** Why is DNS-based failover slow?
**A:** Cached answers until TTL expiry, caches ignoring TTLs, apps caching IPs, and open connections never re-resolving.

**Q:** What's negative caching in DNS?
**A:** "Name doesn't exist" (NXDOMAIN) answers are cached too, for a time set by the zone.

**Q:** Why can't a CNAME be at the zone apex?
**A:** The apex must hold other records (NS, SOA), and a CNAME can't coexist with other records. Providers offer ALIAS/ANAME instead.

**Q:** What is GeoDNS, and its main blind spot?
**A:** Answering with a region-specific IP based on location. It sees the resolver's location, not the user's (EDNS Client Subnet helps).

**Q:** What is anycast?
**A:** The same IP announced via BGP from many sites; routing delivers each user to the nearest one.

**Q:** Who uses anycast?
**A:** DNS root servers, public resolvers, CDNs, DDoS protection.

**Q:** The three things TLS provides?
**A:** Confidentiality, integrity, authentication.

**Q:** How does a client verify a server's certificate?
**A:** Chain to a trusted root CA, name matches, not expired; server proves it holds the private key by signing the handshake.

**Q:** What is forward secrecy?
**A:** Ephemeral Diffie–Hellman keys mean a stolen long-term key can't decrypt past recorded traffic.

**Q:** What is SNI?
**A:** The client names the hostname in its first handshake message, so one IP can serve many sites and certificates.

**Q:** Handshake RTTs: TLS 1.2 vs TLS 1.3?
**A:** TLS 1.2: 2 RTT. TLS 1.3: 1 RTT (client sends its key share up front). Both on top of TCP's 1 RTT.

**Q:** What's the risk of TLS 1.3 0-RTT?
**A:** Early data can be replayed. Only safe for idempotent requests.

**Q:** Session IDs vs session tickets?
**A:** IDs: server stores state (breaks behind a load balancer). Tickets: client stores encrypted state; any server with the key can resume.

**Q:** What does TLS termination at the edge buy you?
**A:** The handshake RTTs happen over the short user-to-edge path, and backends are spared handshake CPU.

**Q:** Cost of a cold HTTPS request vs a reused connection?
**A:** Cold: DNS + TCP (1 RTT) + TLS (1 RTT) + request (1 RTT) + slow start. Reused: 1 RTT, warm cwnd.

**Q:** HTTP keep-alive vs TCP keepalive?
**A:** HTTP keep-alive reuses a connection for many requests. TCP keepalive sends probes on idle connections to detect dead peers.

**Q:** Keep-alive idle timeout rule of thumb?
**A:** Client's idle timeout shorter than the server's, so the client never sends on a connection the server just closed.

**Q:** How do you size a connection pool?
**A:** Little's Law: request rate × time each request holds a connection, plus headroom. Not bigger than the database can use.

**Q:** Why can a bigger pool reduce throughput?
**A:** Past the database's CPU/disk capacity, extra connections only add contention for locks, memory and CPU.

**Q:** The pool multiplication problem and fix?
**A:** Instances × pool size = huge connection counts. Fix: smaller pools or a shared pooler (e.g. PgBouncer).

**Q:** Why set a connection-acquire timeout?
**A:** When a dependency slows down the pool runs dry; the timeout makes requests fail fast instead of piling up.

**Q:** Why do new servers get no traffic from a pooled client?
**A:** Long-lived connections stay where they were opened. Fix: maximum connection lifetime, or per-request (L7) balancing.
