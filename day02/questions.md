# Day 2: Questions

Attempt these without looking at the lesson. Write your answers in `notes/` if it helps, then check
`answers.md`.

## Layers & IP

1. IP is "best-effort." List four things that can go wrong with an IP packet. Why was it a good design
   decision to keep IP this weak?

2. Your service in Mumbai calls a dependency in Virginia (RTT ~200 ms). A teammate proposes upgrading
   the link from 1 Gbps to 10 Gbps to speed up these small API calls. Will it help? What would?

## TCP

3. Walk through the TCP three-way handshake. What does each side learn, and what does it cost before
   the first byte of application data can be sent?

4. TCP is described as "reliable." A request's packet gets dropped once by a congested router. What
   does the application see? How does this connect to tail latency from Day 1?

5. What's the difference between flow control and congestion control? Who is each one protecting, and
   how does the sender combine them?

6. A brand-new TCP connection on a 1 Gbps link with 100 ms RTT is used to download a 1 MB file. Why
   does it take far longer than `1 MB / 1 Gbps`? Roughly how long does it take, and what would make it
   faster?

7. A single TCP connection between two data centres with 80 ms RTT can't get more than a fraction of
   the available bandwidth, even with no packet loss. What limits it? What's the general formula?

8. Explain TCP head-of-line blocking. Why is it harmless for downloading one file but harmful when
   many independent requests share one connection?

9. A service opens a new TCP connection to the same downstream service for every request, at about
   1,000 requests/second, and starts failing with "cannot assign requested address" errors. What's
   happening, and what's the right fix?

10. A request/response protocol shows strange, consistent ~40 ms delays on small messages even inside
    one data centre. What's a likely cause, and what's the usual fix?

## UDP

11. Why does a video-calling app prefer UDP over TCP, even though UDP loses packets?

12. DNS mostly uses UDP. Why is that a good fit? What has to happen when a DNS response is lost?

13. If UDP has no reliability, how can QUIC (built on UDP) deliver web pages reliably? Why build on UDP
    instead of designing a new transport protocol in the OS?

14. Why are UDP services like DNS and NTP used in amplification attacks, while TCP services mostly
    aren't?

## DNS

15. Explain the difference between a recursive resolver and an authoritative name server. Which one
    does your laptop talk to?

16. Your team wants to move a service to new servers by changing its DNS record. The current TTL is
    86,400 seconds (one day). What should you do, and when? Even after you do it right, why might some
    traffic keep going to the old servers?

17. Someone proposes using DNS as the only failover mechanism for a critical service: "if a server
    dies, we just remove its IP from DNS." What's wrong with this plan?

18. A user in Mumbai is being routed by GeoDNS to a Singapore data centre even though a Mumbai data
    centre exists. What might be causing this?

19. What's the difference between GeoDNS and anycast? Which reacts faster to a site going down, and
    why?

## TLS

20. What three properties does TLS provide? Which part of the handshake gives each one?

21. What is forward secrecy, and what makes it possible?

22. Why does TLS 1.3 need fewer round trips than TLS 1.2? What does TLS 1.3 0-RTT add, and why is it
    only safe for some requests?

23. Behind a load balancer with 20 servers, session resumption via session IDs works poorly but
    session tickets work well. Why?

## Connection reuse

24. Add up the round trips for a cold HTTPS request (with TLS 1.3) versus a request on a reused
    connection. Besides latency, what else does reuse save?

25. You have 40 app instances, each with a database pool of 25 connections. The database starts
    struggling. What's the problem, and what are two ways to fix it?

26. A dependency slows from 10 ms to 500 ms per call. Explain what happens to the connection pool and
    to the calling service. What setting limits the damage?

27. After adding three new backend servers behind a load balancer, they receive almost no traffic from
    one client service, which uses a connection pool. Why?
