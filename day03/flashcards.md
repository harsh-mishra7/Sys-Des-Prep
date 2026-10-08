# Day 3: Flashcards

Cover the answer, say it out loud, then check.

---

**Q:** What four things must every application protocol decide?
**A:** Framing (where messages end), semantics (what's being asked), outcome (what happened), and multiplexing (whether conversations can share a connection).

**Q:** What's HTTP/1.1's core performance limitation?
**A:** One outstanding request per connection; responses must come back in request order.

**Q:** Why did browsers disable HTTP/1.1 pipelining?
**A:** Responses must return in order, so one slow response blocks the rest (HTTP-layer head-of-line blocking), and many proxies handled it badly.

**Q:** How many connections per origin do browsers typically open over HTTP/1.1?
**A:** About 6.

**Q:** What did HTTP/2 change, and what did it keep?
**A:** Changed the wire format (binary frames, streams, multiplexing, HPACK); kept HTTP semantics (methods, status codes, headers, URLs).

**Q:** What is an HTTP/2 stream?
**A:** One request/response exchange, identified by a stream ID carried on every frame, so streams can be interleaved on one connection.

**Q:** What is HPACK?
**A:** HTTP/2 header compression: static table + dynamic table shared by both sides + Huffman coding. Repeated headers become small indexes.

**Q:** What happened to HTTP/2 server push?
**A:** Rarely helpful (servers pushed what clients already had cached); removed from Chrome in 2022. Replaced by `103 Early Hints`.

**Q:** What head-of-line blocking remains in HTTP/2?
**A:** TCP's: one lost packet stalls every stream on the connection until it's retransmitted.

**Q:** How is HTTP/2 negotiated?
**A:** Via ALPN during the TLS handshake; no extra round trip.

**Q:** What is QUIC?
**A:** A transport protocol over UDP with built-in TLS 1.3, independent streams, its own reliability and congestion control, running in user space.

**Q:** Why is QUIC built on UDP?
**A:** TCP lives in kernels and middleboxes and is effectively impossible to change (ossification); UDP passes through almost everywhere and lets QUIC implement its own guarantees.

**Q:** Round trips before the first request: TCP+TLS 1.2, TCP+TLS 1.3, QUIC, QUIC 0-RTT?
**A:** 3, 2, 1, 0.

**Q:** What is QUIC connection migration?
**A:** Connections are identified by a connection ID, not the IP/port 4-tuple, so they survive network changes (Wi-Fi → cellular).

**Q:** Why does QUIC encrypt its transport headers?
**A:** So middleboxes can't depend on them, which keeps the protocol free to evolve (avoids ossification).

**Q:** What does QUIC cost?
**A:** More CPU per byte, UDP may be blocked (need TCP fallback), discovery via Alt-Svc / DNS HTTPS records, less visibility for network tools.

**Q:** Where does HTTP/3 help most?
**A:** At the edge, on lossy, high-latency or changing networks (mobile). Little benefit inside a data centre.

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

**Q:** What's 202 Accepted for?
**A:** The request was accepted and will be processed later (asynchronous work); usually with a `Location` to check status.

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

**Q:** What does `reserved` do in protobuf?
**A:** Prevents a removed field's number (or name) from ever being reused.

**Q:** What transport does gRPC use?
**A:** HTTP/2: each call is a stream, path is `/Service/Method`, length-prefixed protobuf messages.

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

**Q:** Short polling: main cost?
**A:** Mostly empty responses (N clients / interval requests per second) and latency up to the interval.

**Q:** How does long polling work?
**A:** The server holds the request until there's data or a timeout; the client immediately re-requests, sending a cursor.

**Q:** What is SSE?
**A:** One long HTTP response with `text/event-stream`; the server writes events into it. Server → client only, text, built-in reconnect with `Last-Event-ID`.

**Q:** How does a WebSocket start?
**A:** HTTP/1.1 GET with `Upgrade: websocket` and `Sec-WebSocket-Key`; server answers `101 Switching Protocols`; then the connection carries WebSocket frames.

**Q:** Why are client → server WebSocket frames masked?
**A:** To stop malicious pages crafting bytes that confuse intermediary proxies (cache poisoning).

**Q:** SSE or WebSockets for server → client notifications?
**A:** Usually SSE: simpler, plain HTTP, auto-reconnect. WebSockets when the client also sends at a high rate.

**Q:** What do persistent connections (long polling, SSE, WebSockets) cost?
**A:** Stateful servers, connection draining on deploy, pub/sub fan-out between servers, reconnection storms, slow rebalancing, capacity measured in concurrent connections.

**Q:** What is a webhook?
**A:** Server-to-server push: you POST to a URL the receiver registered. Needs retries, so receivers must be idempotent.

**Q:** What is temporal coupling?
**A:** A synchronous caller can only succeed if the callee is up and responsive right now.

**Q:** Availability of a request through 5 synchronous services at 99.9% each?
**A:** 0.999^5 ≈ 99.5%.

**Q:** How does a slow dependency cascade?
**A:** Callers' threads and connections are held longer, pools fill, callers become slow, and it spreads up the chain.

**Q:** What does async messaging buy?
**A:** Temporal decoupling, load levelling, independent scaling, easy fan-out.

**Q:** What does async messaging cost?
**A:** No immediate result, eventual consistency, duplicates/ordering issues (idempotent consumers), harder observability, a broker to run.

**Q:** When to use synchronous calls?
**A:** When the caller needs the answer to continue (reads, auth, validation, balance checks).

**Q:** Rule of thumb for request paths?
**A:** Keep the synchronous path short; push everything that can happen later onto queues.
