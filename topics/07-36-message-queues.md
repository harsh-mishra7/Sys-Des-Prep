# 36 — Message queues

> Part 7 — Moving Data Between Systems · Prerequisites: topics 02, 07

So far, most communication between components has been a direct request/response call: service A calls
service B and waits for an answer (topic 07). That is simple, but it welds the two services together in
time. If B is slow, A is slow. If B is down, A fails. If A suddenly sends ten times the usual traffic, B
has to absorb all of it right now. A **message queue** breaks that coupling by putting a durable buffer
between the producer of work and the consumer of work. The producer drops a message into the queue and
moves on; the consumer pulls messages out when it is ready. This topic covers why that buffer is so
useful, how classic queue brokers such as RabbitMQ and Amazon SQS actually hand out and track messages
(acks, visibility timeouts, redelivery, competing consumers), and the common routing shapes built on
top: pub/sub, fan-out, routing keys, priority queues and delay queues.

By the end of this topic you should be able to:
1. Explain the three main reasons to go asynchronous — decoupling, buffering, and load smoothing — and what each costs.
2. Walk through the lifecycle of a message in a broker: publish, store, deliver, ack/nack, redeliver, delete.
3. Explain visibility timeouts (SQS) and unacked-message tracking (RabbitMQ), and why both lead to duplicate delivery.
4. Describe the competing-consumers pattern and how prefetch limits affect fairness and throughput.
5. Distinguish point-to-point queues from pub/sub, and describe fan-out, routing-key and topic-pattern routing.
6. Explain how priority and delay queues work and their pitfalls.
7. Recognise when a queue is the wrong tool (you need an answer now, or you need replay — see topic 37).

---

## 1. Why asynchronous messaging

### 1.1 The problem with synchronous chains

Consider a shopping site. When a customer places an order, the order service must:

1. Save the order.
2. Charge the card.
3. Reserve inventory.
4. Send a confirmation email.
5. Notify the analytics pipeline.
6. Tell the recommendation service the user bought something.

If the order service calls all six synchronously, several bad things happen:

```
Client ──► Order svc ──► Payments   (200 ms)
                    ├──► Inventory  (50 ms)
                    ├──► Email      (800 ms, sometimes 30 s when the SMTP provider is slow)
                    ├──► Analytics  (down for maintenance)
                    └──► Recs       (p99 = 2 s)
```

- **Latency adds up.** The user waits for the slowest required call, and with fan-out the tail latency
  of the whole request is dominated by the worst dependency (topic 02's tail-latency amplification).
- **Availability multiplies.** If each of five dependencies is 99.9% available, the chance all are up is
  roughly 0.999^5 ≈ 99.5%. The order endpoint is now less available than any single piece.
- **Load is passed straight through.** A flash sale that triples order traffic triples traffic on the
  email service at the same instant, even though email could happily be sent a minute later.
- **The caller knows too much.** Every time a new team wants to react to orders, someone edits the order
  service to add another call.

Only some of those steps actually need to happen before we answer the customer. Saving the order and
(perhaps) charging the card do. The email, analytics and recommendations do not — they just need to
happen *eventually and reliably*.

### 1.2 What a queue gives you

Put a queue between the order service and the "after-the-fact" work:

```
Client ──► Order svc ──(save order, charge card)──► respond 200 OK
                │
                └──publish "OrderPlaced"──► [ queue ] ──► Email worker
                                                       ──► Analytics worker
                                                       ──► Recs worker
```

Three distinct benefits fall out:

**Decoupling (in time, in availability, in knowledge).**
- *Time:* producer and consumer don't need to be running at the same moment. The email worker can be
  restarted, deployed, or down for an hour; messages wait.
- *Availability:* a consumer outage no longer fails the producer's request. The blast radius shrinks.
- *Knowledge:* with pub/sub (section 4), the producer doesn't even know who the consumers are. New
  consumers subscribe without touching the producer.

**Buffering.** The queue absorbs a mismatch between the rate at which work arrives and the rate at which
it can be processed. Work is not lost when consumers are busy; it waits.

**Load smoothing (load levelling).** Bursty arrivals become a steady processing rate. Consumers are
provisioned for the *average* load plus some headroom, not the peak. The queue's depth goes up during
the burst and drains afterwards.

```
arrival rate                      processing rate (fixed by #workers)
   ▲                                 ▲
   │      ████                       │
   │     ██████                      │ ───────────────────────
   │  ▄▄████████▄▄                   │
   │▄▄█████████████▄▄▄▄              │
   └──────────────────► time         └──────────────────► time

queue depth
   ▲
   │          ▄▄▄▄
   │       ▄██████▄▄
   │    ▄▄█████████████▄▄
   └──────────────────────────► time   (grows during burst, drains after)
```

This connects directly to topic 02's queueing theory: a server pushed near 100% utilisation sees wait
times explode. A queue in front of workers doesn't remove that maths, but it moves the waiting to a
place designed for it (a durable buffer) instead of to open sockets, thread pools and client timeouts.

### 1.3 What it costs

Asynchrony is not free:

- **No immediate answer.** The producer learns only that the message was *accepted*, not that the work
  succeeded. If the user needs the result ("was my card charged?"), a queue alone doesn't help; you need
  a synchronous call or a later notification path (polling, webhooks, topic 08's push channels).
- **Eventual consistency.** Downstream state lags. The confirmation email arrives 3 seconds or 3 hours
  later depending on backlog.
- **Duplicates and reordering.** As we'll see, practical brokers deliver *at least once* and may reorder
  messages. Consumers must cope (topic 38).
- **Operational surface.** A broker is another stateful, replicated system to run, monitor and capacity-plan.
- **Hidden backlog.** A queue can hide a problem for hours: the system "works" while the backlog grows.
  Queue depth and *age of oldest message* become critical metrics (topic 46).
- **Harder debugging.** A single business action now spans several processes and moments in time;
  you need correlation IDs and tracing.

A good rule: use a queue for work that **can be done later**, **must not be lost**, and **does not
need to return a result to the original caller**.

### 1.4 Queue as a buffer — but bounded

An unbounded queue just converts an overload into a memory or disk exhaustion incident later. Real
systems bound the queue (max length, max bytes, max age) and decide what happens at the limit:

| At the limit | Effect | Typical use |
|---|---|---|
| Reject publishes (backpressure) | Producer sees an error, slows down or fails its own request | Work that must not be silently dropped |
| Drop oldest | Keep freshest data | Telemetry, sensor readings |
| Drop newest | Protect existing backlog | Rare |
| Spill to disk | Larger buffer, slower | RabbitMQ "lazy" / quorum queues |

Topic 44 covers backpressure and load shedding in general; the queue is one of the main places those
decisions are made.

---

## 2. Anatomy of a queue broker

### 2.1 The roles

- **Producer (publisher):** sends messages.
- **Broker:** a server (usually a replicated cluster) that stores messages and hands them out.
- **Queue:** a named buffer inside the broker.
- **Consumer (subscriber, worker):** receives messages and processes them.
- **Message:** a body (bytes — JSON, protobuf, Avro; see topic 16) plus headers/attributes (ID,
  timestamp, content type, correlation ID, routing key, retry count…).

Two widely used examples, which differ in interesting ways:

- **RabbitMQ:** an open-source broker implementing AMQP 0-9-1. Persistent TCP connections; the broker
  *pushes* messages to consumers over a channel, subject to a prefetch limit. Rich routing via
  *exchanges*.
- **Amazon SQS:** a managed queue service. Consumers *pull* via HTTP (`ReceiveMessage`), optionally with
  long polling. Very simple routing (one queue = one stream of work); fan-out via SNS in front.

### 2.2 The lifecycle of a message

The central idea: **a broker does not delete a message when it hands it out; it deletes it when the
consumer says it's done.**

```
  Producer                Broker                         Consumer
     │  publish(m)          │                                │
     │─────────────────────►│ store m (memory / disk,        │
     │                      │  replicated)                   │
     │◄───── publish ack ───│                                │
     │                      │                                │
     │                      │──── deliver m ────────────────►│
     │                      │ mark m "in flight"             │ process m
     │                      │  (unacked / invisible)         │ (DB write, send email…)
     │                      │                                │
     │                      │◄──────────── ack(m) ───────────│
     │                      │ delete m                       │
```

Two acknowledgements are involved, and both matter:

1. **Publisher confirm (producer → broker):** the broker says "I've durably stored this". Without it,
   a producer that fires and forgets can lose messages if the broker crashes before writing to disk.
   RabbitMQ calls this *publisher confirms*; SQS returns success from `SendMessage` only after the
   message is stored redundantly.
2. **Consumer ack (consumer → broker):** the consumer says "I've processed this; you may delete it".

### 2.3 Acks, nacks and the "ack after processing" rule

Where the consumer acks determines what happens on a crash:

**Ack before processing (auto-ack):**

```
receive m → ack → [crash] → work never done → message gone   ⇒ at-most-once
```

**Ack after processing:**

```
receive m → process (DB write succeeds) → [crash before ack] → broker redelivers m
         → process again                                      ⇒ at-least-once (duplicates)
```

Almost every system that cares about not losing work chooses **ack after processing** and makes the
processing idempotent (topic 38). There is no ordering of "do the work" and "ack" that avoids both loss
and duplication, because the ack and the side-effect are two separate actions in two separate systems
— the same fundamental problem as the dual write in topic 39.

A consumer can also **reject / nack** a message:
- *nack with requeue:* "I couldn't do it, give it to someone (maybe me) again". Danger: a message that
  always fails will loop forever (a **poison message**, topic 38).
- *nack without requeue / reject:* drop it, or route it to a **dead-letter queue** if one is configured.

### 2.4 Visibility timeouts (SQS)

SQS has no long-lived connection that can notice a consumer died. Instead it uses a lease:

1. `ReceiveMessage` returns message *m* and a *receipt handle*.
2. *m* becomes **invisible** to other consumers for the **visibility timeout** (default 30 s).
3. If the consumer calls `DeleteMessage(receipt handle)` before the timeout, *m* is gone.
4. If not — consumer crashed, was slow, or forgot — *m* becomes visible again and another consumer
   receives it. Its `ApproximateReceiveCount` increments.

```
t=0     worker A receives m      (m invisible until t=30)
t=12    A finishes, deletes m    ✔ normal case

t=0     worker A receives m      (invisible until t=30)
t=25    A is still busy (slow DB)
t=30    m visible again → worker B receives m
t=34    A finishes, tries DeleteMessage with its (now stale) receipt handle
t=40    B finishes, deletes m
        ⇒ m was processed TWICE, concurrently, by A and B
```

That second timeline is the classic trap: **a visibility timeout shorter than worst-case processing
time causes duplicate, concurrent processing.** Mitigations:

- Set the timeout comfortably above p99.9 processing time.
- For long jobs, **extend** the lease periodically (`ChangeMessageVisibility`) — a heartbeat.
- Still make processing idempotent, because leases can always expire (GC pause, network partition —
  the same lease problem as topic 31).

### 2.5 Unacked tracking (RabbitMQ)

RabbitMQ uses the connection itself as the liveness signal. Delivered-but-unacked messages are tracked
per channel. If the consumer's TCP connection or channel closes, all its unacked messages are
immediately requeued. There is also a consumer *delivery acknowledgement timeout* (default 30 minutes
in recent versions) that closes channels holding a message too long.

The trade-off vs a visibility timeout:
- Fast failure detection when a process dies (connection drops at once)…
- …but a consumer that is *alive but stuck* (deadlocked thread, infinite loop) keeps its messages until
  the timeout. And a network blip that drops the connection after the work was done but before the ack
  arrived causes redelivery — again, at-least-once.

### 2.6 Durability inside the broker

"The broker stored my message" can mean very different things:

- **In memory only:** fast; lost on broker restart.
- **Persistent to local disk:** survives restart, lost if the disk/machine dies.
- **Replicated:** written to several nodes before confirming. RabbitMQ **quorum queues** use Raft
  (topic 30) to replicate each queue across an odd number of nodes; SQS stores messages redundantly
  across multiple availability zones.

As with databases (topic 23), there's a latency cost for each step up in durability: a publisher
confirm on a quorum queue waits for a majority of replicas to fsync.

### 2.7 Push vs pull

| | Push (RabbitMQ consume) | Pull (SQS receive, Kafka poll) |
|---|---|---|
| Latency | Lowest — broker sends as soon as available | Depends on poll interval; long polling helps |
| Flow control | Broker must not overwhelm consumer → **prefetch** limit | Natural: consumer asks only when ready |
| Connection | Long-lived | Can be request-per-batch |
| Empty queue cost | Nothing | Wasted requests unless long polling |

**Long polling** (SQS `WaitTimeSeconds` up to 20 s) is the same idea as HTTP long polling in topic 08:
the receive call waits until a message arrives or the timeout ends, cutting empty responses.

---

## 3. Competing consumers

### 3.1 The pattern

Several identical workers read from **the same queue**. Each message goes to **exactly one** of them
(per delivery attempt). This is how you scale processing horizontally: more workers → more throughput.

```
                      ┌──► worker 1   (gets m1, m4, m7…)
Producer ──► [queue] ─┼──► worker 2   (gets m2, m5, m8…)
                      └──► worker 3   (gets m3, m6, m9…)
```

Properties:
- **Load balancing for free.** Busy workers simply ask for (or are given) fewer messages.
- **Fault tolerance.** If worker 2 dies, its in-flight messages are redelivered to 1 or 3.
- **Elastic scaling.** Autoscale the worker pool on queue depth or oldest-message age.

### 3.2 Prefetch and fairness

With push delivery, a broker that sends every message to the first consumer it sees would overload it.
RabbitMQ's **prefetch count** (`basic.qos`) caps how many unacked messages a consumer may hold.

- `prefetch = 1`: perfectly fair; a worker gets a new message only after acking the last one. But each
  message costs a full round trip, so throughput suffers with fast tasks.
- `prefetch = 100`: high throughput, but a slow worker may sit on 100 messages while others are idle,
  and if it dies, 100 messages are redelivered at once.

Rule of thumb: prefetch ≈ enough to cover the round-trip time at the worker's processing rate
(Little's Law from topic 02: in-flight = throughput × latency).

SQS's equivalent knob is `MaxNumberOfMessages` (1–10) per receive.

### 3.3 Ordering is lost

Even if the queue itself is FIFO, competing consumers destroy ordering in practice:

```
queue: m1 (create user) , m2 (update user email)

worker A gets m1, worker B gets m2
B is faster → "update email" runs before "create user" → update fails or is lost
```

Also, a redelivered message re-enters after messages that were behind it. So a plain queue with
multiple consumers gives you **no ordering guarantee**. Options:

- Design messages to be order-independent (carry full state, use versions — topic 38).
- Use a single consumer (kills scaling).
- Partition by key so all messages for one entity go to one consumer: **SQS FIFO queues** with a
  *message group ID* deliver messages of the same group in order, one at a time, while different groups
  proceed in parallel. RabbitMQ offers *single active consumer* and consistent-hash exchanges.
  This "ordered within a key, parallel across keys" idea is exactly what Kafka partitions provide as
  the core model (topic 37).

### 3.4 Work queue vs task queue libraries

Libraries like Celery, Sidekiq, or BullMQ (on Redis) build a *task queue* on top of a broker: you
enqueue a function name plus arguments, workers execute it, with retries and scheduling. Same
mechanics underneath: competing consumers, acks, redelivery, at-least-once.

---

## 4. Pub/sub, fan-out and routing

### 4.1 Point-to-point vs publish/subscribe

- **Point-to-point (queue):** each message is consumed by one consumer. "Do this job once."
- **Publish/subscribe (topic):** each message is delivered to **every subscriber**. "This happened;
  whoever cares, react."

```
Point-to-point                        Pub/sub (fan-out)
                                                     ┌──► [q-email]     ──► email workers (×3)
P ──► [q] ──► one of {C1,C2,C3}       P ──► topic ───┼──► [q-analytics] ──► analytics worker
                                                     └──► [q-recs]      ──► recs workers (×2)
```

The usual real-world shape is the right-hand one: **pub/sub between services, competing consumers
within a service.** Each subscribing service gets its *own* queue (its own copy of every message), and
within that service many workers compete on that queue. Each service's backlog, retry and failure
are isolated from the others.

Commands vs events is a useful distinction here:
- A **command** ("SendEmail") is addressed to one handler → point-to-point.
- An **event** ("OrderPlaced") states a fact; zero or many parties may react → pub/sub.

### 4.2 RabbitMQ exchanges

In AMQP, producers don't publish to queues directly; they publish to an **exchange** with a **routing
key**. Queues are **bound** to exchanges with binding rules. The exchange type decides how messages
are copied to queues:

| Exchange type | Rule | Example |
|---|---|---|
| **direct** | Routing key must equal the binding key | key `pdf` → queue bound with `pdf` |
| **fanout** | Ignore key; copy to every bound queue | broadcast "OrderPlaced" |
| **topic** | Pattern match on dot-separated words: `*` = one word, `#` = zero or more | `order.*.eu` matches `order.created.eu` |
| **headers** | Match on header values instead of key | `format=pdf AND type=report` |

```
                         binding "order.created.*"
producer ─ key "order.created.eu" ─► [topic exchange] ──────────────────► [q-email]
                                         │ binding "order.#"
                                         ├────────────────────────────────► [q-audit]
                                         │ binding "payment.#"
                                         └──────── (no match) ──────────── [q-payments]
```

### 4.3 SNS + SQS

On AWS, the canonical fan-out is **SNS topic → several SQS queues**. SNS pushes each published message
to every subscription; each subscription is an SQS queue owned by a consuming service. Subscription
*filter policies* provide attribute-based routing (similar to a headers exchange). Without the SQS
queue in between, a subscriber that is down would miss messages; the queue is what gives each
subscriber its durable buffer.

### 4.4 Ephemeral pub/sub

Not every pub/sub is durable. **Redis Pub/Sub** delivers only to subscribers connected *right now*; if
nobody is listening, the message is gone. That is fine for transient signals (cache invalidation
broadcast, typing indicators — see topic 08's fan-out to connected clients) and wrong for business
events.

---

## 5. Priority and delay queues

### 5.1 Priority queues

Sometimes some messages matter more: password-reset emails before marketing emails. Approaches:

1. **Broker-level priority.** RabbitMQ classic queues support `x-max-priority`; messages with higher
   priority are delivered first among the messages *waiting in the queue*.
2. **Separate queues per priority**, with consumers that poll the high queue first, or with more
   workers dedicated to high. SQS has no priority, so this is the standard pattern there.

Pitfalls:
- **Starvation.** If high-priority traffic never stops, low-priority messages never run. Use weighted
  polling (e.g. 4 high : 1 low) or reserve workers for low.
- **Priority only reorders the backlog.** Messages already prefetched to a consumer can't be overtaken.
  With a high prefetch, priority barely works; keep prefetch low on priority queues.
- **Separate queues isolate better.** A flood of low-priority work can't fill the shared buffer and
  block high-priority publishes.

### 5.2 Delay queues and scheduled delivery

"Retry this in 5 minutes", "send a reminder in 24 hours", "cancel the unpaid order after 15 minutes".

Mechanisms:
- **Per-message delay**: SQS `DelaySeconds` (up to 15 minutes) or queue-level delay; the message is
  invisible until the delay passes.
- **TTL + dead-letter trick (RabbitMQ):** publish into a "wait" queue with no consumers and a
  per-queue TTL; when messages expire they are dead-lettered into the real work queue. (The delayed
  message exchange plugin does this more directly.) Note the pitfall: with per-message TTLs, a classic
  queue only expires messages at the *head*, so a long-TTL message can block shorter ones behind it —
  hence one wait queue per delay value (5 s, 30 s, 5 min…) in retry ladders.
- **Sorted-set schedulers**: e.g. a Redis sorted set keyed by due-time, polled by a scheduler that
  moves due items into the work queue.
- **Visibility timeout as retry delay**: on failure, instead of deleting, set the message's visibility
  timeout to the backoff duration so it reappears later.

Delay queues are the building block for **retry with backoff** (topic 43) inside async systems:

```
work queue ──► consumer ──fail──► retry-5s queue ──(TTL)──► work queue
                         ──fail──► retry-1m queue ──(TTL)──► work queue
                         ──fail──► retry-10m queue ─(TTL)──► work queue
                         ──fail (too many)──► dead-letter queue (topic 38)
```

Long delays (days) usually belong in a database table of scheduled jobs, not in a broker, because you
will want to query, cancel and edit them.

---

## 6. Request/reply over a queue

Sometimes you want asynchronous transport but still need a response. The pattern:

1. The requester creates (or reuses) a **reply queue** and generates a **correlation ID**.
2. It publishes the request with headers `reply_to = reply-queue`, `correlation_id = X`.
3. The responder processes and publishes the result to `reply_to` with the same correlation ID.
4. The requester matches replies by correlation ID.

This gives load levelling and retry for the request side, but the requester still waits, so you
reintroduce timeouts and most of the coupling of a synchronous call. Use it sparingly; if you need an
answer now, a plain RPC (topic 07) is usually clearer.

---

## 7. Failure modes and operational concerns

### 7.1 Things that go wrong

| Failure | What you see | Mitigation |
|---|---|---|
| Consumer slower than producer, sustained | Queue depth grows without bound | Scale consumers; backpressure; shed load |
| Visibility timeout < processing time | Same message processed concurrently twice | Longer timeout, heartbeat extension, idempotency |
| Poison message + requeue | One message loops forever, eating a worker | Max receive count → dead-letter queue (topic 38) |
| Producer publishes without confirms | Silent message loss on broker crash | Publisher confirms; durable queues |
| Ack before processing | Loss on consumer crash | Ack after processing |
| Downstream DB slow | All workers blocked; huge backlog; retries pile on | Limit concurrency; circuit breakers (topic 44) |
| Recovery stampede | After an outage, workers drain backlog at max speed and overload the DB | Rate-limit consumers; ramp up gradually |
| Broker memory/disk full | Broker blocks publishers (RabbitMQ flow control) | Bounded queues, alarms on depth |

### 7.2 Key metrics

- **Queue depth** (messages ready + in flight).
- **Age of oldest message** — the best single signal of "are we falling behind?" Depth alone can be
  misleading (10,000 tiny messages may drain in a second).
- **Publish rate vs consume (ack) rate.**
- **Redelivery rate / receive count distribution.**
- **Dead-letter queue depth** — should normally be zero; alert when it isn't.

### 7.3 Messages are not a database

A queue is optimised for messages that are written once, read once and deleted. Brokers like RabbitMQ
perform worst when queues are very long (millions of messages): memory pressure, slow restarts. They
do not support queries ("find all messages for order 123"), and once a message is acked it is **gone** —
you cannot re-read it to rebuild a downstream system or add a new consumer that needs history. That
limitation is what motivates the log-based design in topic 37.

---

## 8. How it connects

- **Topic 02 (queueing):** queues make waiting explicit; utilisation near 100% still means unbounded
  wait — now visible as queue depth.
- **Topic 07 (sync vs async):** this topic is the "async" half of that comparison in depth.
- **Topic 08 (real-time):** pub/sub inside the backend often feeds fan-out to connected clients.
- **Topic 30/31 (consensus, leases):** quorum queues use Raft; visibility timeouts are leases with all
  the usual lease hazards.
- **Topic 35 (sagas):** saga steps are commonly driven by messages over queues.
- **Topic 37 (Kafka):** the log-based alternative — retains messages, supports replay and many
  independent readers, ordered per partition.
- **Topic 38 (delivery semantics):** at-least-once delivery from this topic forces idempotent consumers,
  dead-letter queues, and careful retries.
- **Topic 39 (outbox):** how to publish a message *and* update a database without one of them being lost.
- **Topics 43/44 (retries, backpressure):** retries via delay queues; bounded queues as backpressure.
- **Topic 46 (observability):** queue depth, message age, correlation IDs across async hops.

---

## Common misconceptions

- **"Putting a queue in front makes the system handle more load."** → It handles *bursts* better. If the
  average arrival rate exceeds the processing rate, the backlog grows forever; a queue only delays the
  failure.
- **"The broker deletes a message when it delivers it."** → It deletes on ack. Delivery and completion
  are separate, which is why redelivery and duplicates exist.
- **"A FIFO queue means my messages are processed in order."** → Only with one consumer (or per-group
  ordering like SQS FIFO message groups). Competing consumers and redelivery reorder work.
- **"Exactly-once delivery is a broker setting."** → Brokers give at-most-once or at-least-once.
  Effective exactly-once comes from idempotent processing (topic 38).
- **"Pub/sub means each service shares one queue."** → Each subscribing service needs its own queue
  (its own copy); otherwise services compete for messages and each sees only some of them.
- **"Async is always more scalable."** → It moves work off the request path but adds eventual
  consistency, duplicates, operational load and debugging cost. Use it where "later" is acceptable.
- **"Redis Pub/Sub is a message queue."** → It's fire-and-forget to currently connected subscribers; no
  persistence, no acks.

## Further reading (supplements)

- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 11 ("Stream Processing"), section
  on messaging systems.
- Gregor Hohpe & Bobby Woolf, *Enterprise Integration Patterns* — the canonical catalogue (competing
  consumers, publish-subscribe channel, dead letter channel, correlation identifier, request-reply).
- RabbitMQ documentation: "Consumer Acknowledgements and Publisher Confirms", "Quorum Queues", and the
  AMQP 0-9-1 model explained.
- AWS documentation: "Amazon SQS visibility timeout" and "Amazon SQS FIFO queues".

---

## Questions

1. A team says "we added a queue so the email service can't take down checkout anymore". Which kind of
   decoupling did they achieve, and what new failure must they now monitor for?
2. Why does a broker hand out a message but not delete it until the consumer acks? What would break if
   it deleted on delivery?
3. An SQS consumer takes 20 s on average but occasionally 90 s; the visibility timeout is 30 s. Describe
   precisely what happens on a 90 s message, and two ways to fix it.
4. A RabbitMQ consumer processes a message, commits to the database, and then its network connection
   drops before the ack arrives. What happens next, and why can't the broker avoid it?
5. You set `prefetch = 500` on a queue where each task takes 2 s. What happens to fairness across 10
   workers, and what happens when one worker crashes?
6. Messages "create account" and "set account email" for the same user are published in order onto a
   FIFO queue with 5 competing consumers. Why can the email update fail? Give two designs that fix it.
7. Three services must react to every "OrderPlaced" event, each with several workers. Sketch the queue
   topology and explain why a single shared queue would be wrong.
8. In a RabbitMQ topic exchange, which of these routing keys match the binding `order.*.eu`:
   `order.created.eu`, `order.eu`, `order.paid.us`, `order.refund.partial.eu`? What about `order.#`?
9. Your high-priority queue still processes urgent jobs slowly during a marketing blast, even though
   broker priority is enabled. What's a likely cause?
10. Why is a queue that is allowed to grow without bound dangerous, and what are the options when the
    limit is reached?
11. A new analytics team wants every order event from the past 90 days, plus all future ones. Why is a
    classic queue a poor fit?
12. Which metric better tells you a consumer is falling behind: queue depth or age of oldest message? Why?
13. After a 2-hour outage of the consumer's database, the consumers come back and the database falls over
    again within a minute. Explain the mechanism and the fix.

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Temporal/availability decoupling: checkout publishes and returns even if the email service is
down or slow. The new failure is a silently growing backlog — emails delayed by hours while everything
looks healthy. They must monitor queue depth, age of oldest message, and the dead-letter queue.

**2.** Because delivery is not completion. If the broker deleted on delivery and the consumer crashed
mid-processing, the work would be lost (at-most-once). Keeping the message until ack lets the broker
redeliver after a crash or timeout, giving at-least-once.

**3.** At t=30 the message becomes visible again and a second consumer receives it while the first is
still working; both process it concurrently. The first one's delete at t=90 uses a stale receipt handle
(and may or may not succeed); the second will process and delete it again later. Fixes: raise the
visibility timeout above worst-case time, or have the consumer periodically extend visibility (heartbeat)
while working — and make processing idempotent regardless.

**4.** The broker sees the channel close with the message unacked and requeues it; another consumer
processes it again, duplicating the DB write. The broker can't avoid it because it cannot know whether
the side effect happened — the DB commit and the ack are two separate actions in two systems, and the
crash fell between them.

**5.** The first workers to connect may grab up to 500 messages each, so some workers hold huge
backlogs (500 × 2 s ≈ 17 minutes of work) while others sit idle; fairness is poor and latency of those
messages is high. If one crashes, all 500 unacked messages are redelivered at once (some possibly
already processed → duplicates). Lower prefetch to roughly throughput × round-trip time.

**6.** Different consumers pick up the two messages; the one holding "set email" may finish first,
before the account exists. Fixes: route all messages for a user to the same ordered lane (SQS FIFO
message group ID = user ID, consistent-hash exchange, or Kafka partitioning by key — topic 37); or make
messages order-tolerant (carry versions/full state and let the consumer reject or defer stale/out-of-order
updates).

**7.** One fan-out point (SNS topic or fanout/topic exchange) bound to three queues: q-email, q-analytics,
q-recs. Each service's workers compete on their own queue. With one shared queue, the three services would
compete for messages and each event would reach only one service.

**8.** `order.*.eu` matches only `order.created.eu` (`*` is exactly one word). `order.eu` has too few
words, `order.paid.us` has the wrong last word, `order.refund.partial.eu` has two words in the middle.
`order.#` matches all four keys starting with `order` (`#` = zero or more words), including `order.eu`.

**9.** Priority only reorders messages waiting *in the broker*. If consumers have a large prefetch, they
already hold many low-priority messages locally, and urgent messages queue behind them. Also, shared
workers busy with long low-priority tasks aren't free. Fix: low prefetch, or separate queues with
dedicated workers.

**10.** It hides overload until memory or disk is exhausted, at which point the broker may block all
publishers or crash, and latency for queued items grows unboundedly. At a limit, you can reject
publishes (backpressure to producers), drop oldest, drop newest, or spill to disk — choosing based on
whether data may be lost.

**11.** A classic queue deletes messages once acked, so past events are gone; it doesn't support
multiple independent readers at different positions or replay. A log-based system (topic 37) retains
messages for a period and lets a new consumer group start from the beginning.

**12.** Age of oldest message. Depth depends on message size and processing speed (10k fast messages
may be seconds of work); age directly measures how stale the work is — how far behind consumers are
in time, which is what users experience.

**13.** A huge backlog accumulated. Consumers resume at full concurrency and each message triggers DB
work, so the DB receives a burst far above its normal load and fails again (and retries amplify it).
Fix: cap consumer concurrency or rate, ramp up gradually, use circuit breakers and backoff (topics 43, 44).

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Three main benefits of putting a queue between services?
**A:** Decoupling (time, availability, knowledge), buffering, and load smoothing/levelling.

**Q:** When is a queue the wrong tool?
**A:** When the caller needs the result now, or when consumers need history/replay.

**Q:** When does a broker delete a message?
**A:** When the consumer acks it, not when it delivers it.

**Q:** What are publisher confirms?
**A:** The broker's acknowledgement to the producer that a message is durably stored.

**Q:** Ack before processing gives which semantics?
**A:** At-most-once — a crash after ack loses the work.

**Q:** Ack after processing gives which semantics?
**A:** At-least-once — a crash after the work but before the ack causes redelivery.

**Q:** What is an SQS visibility timeout?
**A:** A lease: after receive, the message is hidden from others for N seconds; if not deleted in time it reappears.

**Q:** Danger of a visibility timeout shorter than processing time?
**A:** Another consumer gets the message while the first is still processing — concurrent duplicates.

**Q:** How do you handle long jobs with SQS?
**A:** Extend visibility periodically (heartbeat with ChangeMessageVisibility).

**Q:** How does RabbitMQ detect a dead consumer?
**A:** Its connection/channel closes; unacked messages are requeued (plus a delivery ack timeout).

**Q:** What is the competing consumers pattern?
**A:** Many workers on one queue; each message goes to one of them — horizontal scaling of processing.

**Q:** What does prefetch control?
**A:** Max unacked messages per consumer — trading throughput against fairness and redelivery burst size.

**Q:** Why do competing consumers break ordering?
**A:** Workers run at different speeds and redelivered messages re-enter later.

**Q:** How does SQS FIFO keep order while scaling?
**A:** Message group ID: in-order within a group, parallel across groups.

**Q:** Point-to-point vs pub/sub?
**A:** P2P: each message to one consumer. Pub/sub: each message to every subscriber.

**Q:** Usual production shape for events?
**A:** Pub/sub between services (one queue per service), competing consumers within each service.

**Q:** RabbitMQ exchange types?
**A:** direct, fanout, topic (`*` one word, `#` zero or more), headers.

**Q:** Command vs event?
**A:** Command: "do X", one handler. Event: "X happened", zero or many reactors.

**Q:** Why put SQS queues behind an SNS topic?
**A:** Each subscriber gets a durable buffer so it doesn't miss messages while down.

**Q:** Redis Pub/Sub durability?
**A:** None — only currently connected subscribers receive messages.

**Q:** Main pitfall of priority queues?
**A:** Starvation of low priority; and prefetched messages can't be overtaken.

**Q:** How is a delay/retry ladder built in RabbitMQ?
**A:** Wait queues with TTL that dead-letter expired messages back into the work queue, one queue per delay.

**Q:** Best metric for "consumers falling behind"?
**A:** Age of the oldest message.

**Q:** What is a poison message?
**A:** A message that always fails processing; with requeue it loops forever — send it to a DLQ.

**Q:** Why can't a classic queue serve a new consumer that needs past data?
**A:** Acked messages are deleted; there's no retained history to replay.
