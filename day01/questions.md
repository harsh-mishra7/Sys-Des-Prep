# Day 1: Questions

Attempt these without looking at the lesson. Write your answers in `notes/` if it helps, then check
`answers.md`.

## Latency & throughput

1. A teammate says "we doubled our throughput, so requests must be faster now." Is that necessarily
   true? Describe a change that raises throughput while making individual requests slower.

2. A service handles 2,000 req/s and has an average response time of 50 ms. Roughly how many requests
   are in flight at any moment? What law did you use, and what does it imply if the service can only
   hold 50 concurrent requests?

3. Your database server sits at 92% CPU. Throughput looks fine, but users complain about slowness.
   Explain what's going on using queueing intuition.

4. Why would a database deliberately delay a commit by a millisecond or two (group commit)? Who wins
   and who loses?

## Percentiles & tail latency

5. Two services both have a mean latency of 40 ms. Service A has p99 = 60 ms; service B has p99 =
   900 ms. Which would you rather depend on, and why does the mean fail to tell them apart?

6. A page load calls 50 backend services in parallel and waits for all of them. Each service has a 1%
   chance of being slow. What fraction of page loads will be slow? What does this tell you about where
   to focus optimization effort?

7. You have 10 servers and each reports its own p99. Can you average them to get the fleet p99? Why
   not, and what should you do instead?

8. Why might your server-side latency metrics look great while users experience slow responses?

## Availability, reliability, durability, fault tolerance

9. Give an example of a system that is:
   a) available but not durable
   b) durable but not (currently) available
   c) available but not reliable

10. Your service has hard dependencies on four other services, each 99.9% available. What's your best
    possible availability? What design changes would raise it?

11. You run two replicas, each 99.9% available. A colleague says "that's six nines." When is that
    claim wrong in practice?

12. Availability ≈ MTBF / (MTBF + MTTR). Your team can either halve how often outages happen or halve
    how long they last. Which gives more availability, and which is usually easier?

13. What's the difference between a fault and a failure? Why are software faults often more dangerous
    than hardware faults, even though hardware faults are more frequent?

## Scalability

14. Someone asks "does our system scale?" What two things do you need to pin down before you can
    answer?

15. In the social feed example, why does the "fan-out on write" approach become a problem for
    accounts with millions of followers? What load parameter is driving that?

16. You add a fourth node to a three-node cluster and total throughput goes *down*. Give two possible
    explanations.

## Scaling & state

17. Why is it easy to add more app servers but hard to add more database servers?

18. An app server stores logged-in user sessions in its own memory. What breaks when you put three of
    these behind a load balancer? Give two ways to fix it.

19. When would you choose vertical scaling over horizontal scaling, even though horizontal scaling has
    a higher ceiling?

## Building blocks

20. For each problem, name the building block you'd reach for first and the new problem it introduces:
    a) Users in another continent see slow image loads.
    b) The database is overwhelmed by repeated reads of the same product pages.
    c) Sending a confirmation email makes the checkout request take 3 seconds.
    d) One app server can't keep up with traffic.
