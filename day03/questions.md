# Day 3: Questions

Attempt these without looking at the lesson. Write your answers in `notes/` if it helps, then check
`answers.md`.

## HTTP versions

1. HTTP/1.1 supports pipelining, yet browsers disabled it. What problem did pipelining leave unsolved,
   and why couldn't the protocol fix it?

2. Before HTTP/2, sites used domain sharding, file concatenation and image sprites. What were these
   working around? Why can they hurt performance under HTTP/2?

3. Explain what a "stream" is in HTTP/2 and how multiplexing works on one connection. Why is one
   connection per origin enough?

4. Why does HTTP/2 compress headers with a dynamic table that both sides maintain, rather than
   compressing each request's headers independently?

5. On a mobile network with 2% packet loss, a page loads *slower* over HTTP/2 than over HTTP/1.1.
   Explain how this can happen.

6. Why was HTTP/3 built on UDP instead of fixing TCP?

7. A user on a train walks from Wi-Fi coverage onto cellular in the middle of a download. What happens
   with HTTP/2, and what happens with HTTP/3? What QUIC feature makes the difference?

8. Count the round trips before the first request byte reaches the server for: TCP + TLS 1.2,
   TCP + TLS 1.3, new QUIC, resumed QUIC with 0-RTT. Why should 0-RTT be limited to idempotent requests?

9. Your team runs internal services in one data centre over HTTP/2. Someone proposes switching
   everything to HTTP/3 for performance. What would you expect, and why?

10. How does a browser discover that a server supports HTTP/3, and what happens if UDP is blocked on
    the user's network?

## REST & HTTP semantics

11. Define "safe" and "idempotent". Give a method that is idempotent but not safe, and one that is
    neither.

12. A client sends `DELETE /orders/7`, times out, retries, and gets `404`. Was `DELETE` idempotent?
    Explain.

13. A client sends `POST /payments` and the connection drops before any response arrives. Why can't the
    client just retry? Describe a mechanism that makes the retry safe.

14. Why will a reverse proxy or HTTP client automatically retry a failed `GET` or `PUT` but not a
    `POST`?

15. An API returns `200 OK` with `{"error": "database unavailable"}` in the body. List three things in
    the surrounding infrastructure that now behave wrongly.

16. What's the difference between `401` and `403`? Between `502`, `503` and `504`?

17. A client gets a `504 Gateway Timeout` on a request that transfers money. What does it know about
    whether the transfer happened? What should it do?

18. Two users edit the same wiki page at the same time and both save. Describe how `ETag` and `If-Match`
    prevent one edit from silently overwriting the other. Which status code does the loser receive?

19. A feed API uses `?offset=N&limit=50`. Users report seeing the same post twice when scrolling, and
    deep pages are slow. Explain both problems and the alternative design.

## RPC, gRPC, GraphQL

20. What's the fundamental danger in RPC's goal of "making a remote call look like a local call"? List
    three ways remote calls differ from local ones.

21. In Protocol Buffers, why are field *numbers* sent on the wire instead of field names? What can you
    safely change in a schema, and what must you never do?

22. A team removes field 4 (`string nickname`) from a message, and months later adds a new field 4
    (`int64 karma`). What can go wrong, and how does protobuf help prevent it?

23. How do gRPC deadlines differ from a simple per-call timeout? Why does propagating them matter in a
    chain of services?

24. A service calls a gRPC backend through a layer 4 load balancer. Ops add 5 new backend instances,
    but the existing ones stay overloaded and the new ones sit idle. Why? What fixes it?

25. What problems does GraphQL solve for a mobile client compared to a set of REST endpoints? Name
    three things it gives up or makes harder, and how each is usually mitigated.

26. A GraphQL query fetches 100 posts and each post's author. The database receives 101 queries. What
    is this problem called and what's the standard fix?

## Server push

27. One million clients short-poll every 10 seconds. How many requests per second is that? How would
    long polling change both the request rate and what the server has to hold?

28. In long polling, events can be lost between one response and the next request. Why, and how is it
    prevented? How does SSE handle the same problem?

29. A dashboard needs live metric updates from the server; the user only occasionally changes a
    filter. SSE or WebSockets? Justify your choice.

30. Why are WebSocket frames from the client to the server masked?

31. A server holding 200,000 WebSocket connections crashes. Describe what happens next and two things
    that keep it from turning into a wider outage.

32. In a chat system with 20 WebSocket servers behind a load balancer, user A (on server 3) sends a
    message to user B (on server 14). What does the system need so the message gets delivered?

## Sync vs async

33. A checkout request synchronously calls 6 services, each with 99.9% availability. What is the
    best-case availability of checkout? What does this argue for?

34. Service E in a synchronous chain A → B → C → D → E becomes slow (not down). Describe how this
    spreads to A.

35. Give three benefits and three costs of putting a queue between two services instead of calling
    synchronously.

36. A "generate annual report" endpoint takes 2 minutes. Design its API so the client isn't stuck
    waiting on an open request. Which status code starts the interaction?

37. For each of the following, would you call synchronously or asynchronously, and why? (a) check that
    a user has enough balance before a purchase, (b) send the order confirmation email, (c) update the
    product search index after a price change, (d) validate a coupon code at checkout.
