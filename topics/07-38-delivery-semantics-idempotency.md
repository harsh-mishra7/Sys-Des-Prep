# 38 — Delivery semantics & idempotency

> Part 7 — Moving Data Between Systems · Prerequisites: topics 07, 21, 26, 36, 37

Every message system eventually forces the question: "if something fails halfway, does this message get
processed zero times, once, or twice?" Networks drop packets, processes crash between two lines of code,
and acknowledgements get lost (topic 26). Because a sender can't distinguish "the receiver never got it"
from "the receiver got it but the reply was lost", it must choose between **not retrying** (risking loss)
and **retrying** (risking duplicates). That choice defines the **delivery semantics** of a system. This
topic explains at-most-once, at-least-once and exactly-once precisely — including what "exactly-once"
does and doesn't mean — and then the toolkit that makes at-least-once safe in practice: **idempotency**
through natural keys, idempotency keys, dedup tables and upserts. Finally, it covers what to do with
messages that can *never* succeed (poison messages, dead-letter queues) and how to make replay safe.

By the end of this topic you should be able to:
1. Define at-most-once, at-least-once and exactly-once, and say which failure each one tolerates.
2. Explain why exactly-once *delivery* is impossible across an unreliable network, and what "exactly-once processing" (effectively-once) actually achieves.
3. Describe how Kafka's idempotent producer and transactions give exactly-once within Kafka, and where that guarantee ends.
4. Make a consumer idempotent using natural keys, idempotency keys, dedup tables, upserts, or conditional/versioned writes — and pick between them.
5. Explain why the dedup check and the side effect must be atomic, and what happens when they aren't.
6. Design poison-message handling with retry limits and dead-letter queues, and redrive DLQ messages safely.
7. Reason about whether a consumer can safely be replayed from an old position.

---

## 1. The root problem: the ambiguous failure

### 1.1 Two generals, one message

A producer sends a message and waits for an acknowledgement. The ack doesn't come. What happened?

```
Case A: request lost          Case B: processed, ack lost        Case C: slow
P ──x                         P ──────► C (done!)                 P ──────► C (still working…)
                              P ◄──x── ack                        P   ...timeout...
```

From the producer's side all three look identical: a timeout. This is the fundamental uncertainty of
distributed systems (topic 26). The producer has only two moves:

- **Don't retry** → if it was case A, the message is lost.
- **Retry** → if it was case B or C, the message is processed twice.

There is no third move that avoids both, at the transport level. Everything in this topic follows from
accepting that and choosing where to put the compensation.

### 1.2 The same problem on the consumer side

A consumer that reads a message, performs a side effect (DB write, email, API call), and then acks
(topic 36) or commits an offset (topic 37) has the same dilemma inside itself:

```
read m → [side effect] → [ack]
              ▲              ▲
      crash here: no      crash here: effect done,
      effect, no ack      no ack → redelivered → effect again
      → redelivered OK
```

The side effect and the ack live in different systems; no single atomic step covers both — unless we
arrange one (section 3).

---

## 2. The three semantics

### 2.1 At-most-once

"Never retry; accept loss."
- Producer: fire-and-forget (Kafka `acks=0`, UDP).
- Consumer: ack/commit **before** processing.

Each message is processed 0 or 1 times. Appropriate when loss is acceptable and duplicates are worse or
pointless: metrics samples, high-frequency sensor readings where the next value supersedes the last,
real-time game state, log lines in a best-effort pipeline.

### 2.2 At-least-once

"Retry until acknowledged; accept duplicates."
- Producer: retries until it gets a durable ack (`acks=all`, publisher confirms).
- Consumer: ack/commit **after** processing.

Each message is processed 1 or more times. This is the default for anything that matters, because loss
is usually unrecoverable while duplicates can be neutralised. **Most real systems are at-least-once plus
idempotent consumers.**

Duplicates come from many sources, not just retries you wrote:
- Producer retries after a lost ack.
- Broker redelivery after a visibility timeout or connection drop (topic 36).
- Kafka consumer rebalances: the new owner restarts from the last committed offset (topic 37).
- Upstream systems that themselves retry (a webhook provider, a payment gateway, a user double-clicking).
- Operators replaying a topic or redriving a DLQ.

### 2.3 Exactly-once

"Each message has its effect exactly once."

Taken literally — the network delivers each message exactly once — this is **impossible** in the
presence of lost messages and crashes, for the reason in section 1.1. What systems actually provide is
**exactly-once processing** (sometimes called **effectively-once**): messages may be *delivered* more
than once, but the *observable effect* on the state is as if each was applied once.

```
exactly-once (effect) = at-least-once (delivery) + deduplication/idempotence (processing)
```

The key questions about any "exactly-once" claim:
1. **Within which boundary?** A system can guarantee exactly-once among components it controls (its own
   log, its own state). Once an effect leaves that boundary — an email, a charge on an external payment
   API, an HTTP call to another service — the guarantee stops unless that external system also
   participates (e.g. it accepts an idempotency key).
2. **Effect on what?** "Exactly-once" in stream processors means the *state and output records* reflect
   each input once; the processing *code* may run several times.

### 2.4 Summary table

| Semantics | Producer | Consumer ack/commit | Failure outcome | Typical use |
|---|---|---|---|---|
| At-most-once | No retries | Before processing | Loss | Metrics, telemetry |
| At-least-once | Retry until acked | After processing | Duplicates | Default for business data |
| Exactly-once (effect) | Retry + dedupe | Atomic with effect, or idempotent effect | Neither (within boundary) | Payments, counters, ledgers, pipelines |

---

## 3. Exactly-once inside a system: how it's built

### 3.1 Kafka's idempotent producer

Problem: a producer's batch reaches the broker, gets written, but the ack is lost; the producer retries
and the broker appends the batch again.

Mechanism: each producer gets a **producer ID (PID)**, and numbers its batches per partition with a
**sequence number**. The broker remembers the last sequence numbers per (PID, partition):
- Sequence = last + 1 → append.
- Sequence ≤ last (already seen) → discard as a duplicate, but ack success.
- Sequence > last + 1 (gap) → error (out of order).

This makes producer retries safe *for a single producer session* writing to Kafka. It also preserves
ordering with several in-flight requests (topic 37).

Limit: if the producer *process* restarts, it gets a new PID (unless it's transactional with a stable
`transactional.id`), so an application that re-sends the same business event after a restart creates a
new record. Idempotent producing dedupes *transport retries*, not *application re-sends*.

### 3.2 Kafka transactions: read-process-write

A stream application reads from topic A, computes, writes to topic B, and commits its offset on A. Three
writes, one crash point between each. Kafka transactions make them atomic:

```
beginTransaction()
   produce(record → topic B, partition 3)
   produce(record → topic C, partition 1)
   sendOffsetsToTransaction({A-0: 1501}, group)   ← offset commit is just a write
commitTransaction()                                 ← to __consumer_offsets
```

- A **transaction coordinator** writes the transaction's state to an internal log and, on commit, writes
  *commit markers* into every partition touched (a form of two-phase commit, topic 34, confined to Kafka).
- Downstream consumers with `isolation.level=read_committed` only see records from committed
  transactions; aborted records are skipped.
- A stable `transactional.id` lets a restarted producer **fence** its previous incarnation (via an
  epoch, like topic 31's fencing tokens): the zombie's late writes are rejected.

Result: for a pipeline that reads from Kafka and writes only to Kafka, each input affects outputs and
offsets exactly once. Kafka Streams enables this with `processing.guarantee=exactly_once_v2`; Flink
achieves similar end-to-end guarantees with checkpoints plus two-phase-commit sinks (topic 41).

Where it ends: if your processing step also writes to Postgres or calls an API, that write is outside the
transaction. A crash after the API call but before `commitTransaction()` causes the call to repeat.

### 3.3 Storing the offset with the result

The general trick that gives exactly-once against an external store: **write the consumer position in
the same transaction as the side effect.**

```
BEGIN;
  UPDATE accounts SET balance = balance + 50 WHERE id = 7;
  UPDATE consumer_offsets SET offset = 1501 WHERE group='billing' AND partition=3;
COMMIT;
```

On restart, the consumer reads its offset from the database (not from Kafka) and seeks there. Either both
the effect and the progress are committed, or neither. This needs the side effect to land in a
transactional store you control, and is the same "make the two writes one write" idea as the outbox
(topic 39).

---

## 4. Idempotency: making duplicates harmless

### 4.1 Definition

An operation is **idempotent** if applying it once or many times leaves the system in the same state:
`f(f(x)) = f(x)`. HTTP's PUT and DELETE are defined as idempotent; POST is not (topic 07).

Some operations are naturally idempotent, others aren't:

| Operation | Idempotent? |
|---|---|
| `SET status = 'shipped' WHERE order_id = 9` | Yes |
| `DELETE FROM carts WHERE id = 4` | Yes (second time deletes nothing) |
| `balance = balance + 50` | No |
| `INSERT INTO orders (...)` with generated ID | No — creates two orders |
| Send email "your order shipped" | No — two emails |
| `counter.increment()` | No |
| "Set profile to version 12" (full state) | Yes |

The goal of this section is to turn the "No" rows into "Yes" in practice. Note one subtlety:
idempotence is about the state *after a sequence including other operations*. `SET status='shipped'` is
idempotent alone, but a delayed duplicate arriving after `SET status='delivered'` would roll the state
back. That's why versioning (4.6) is often needed alongside idempotence.

### 4.2 Natural keys

If the message carries something that uniquely identifies the business fact, use it as a unique key in
the target store.

- Payment events keyed by the payment provider's `charge_id`.
- `order_line (order_id, line_number)` as primary key.
- A daily rollup keyed by `(account_id, date)`.

```sql
CREATE TABLE payments (
  charge_id  text PRIMARY KEY,   -- natural key from the provider
  order_id   bigint NOT NULL,
  amount     numeric NOT NULL
);
INSERT INTO payments VALUES ('ch_8x2', 991, 49.00)
ON CONFLICT (charge_id) DO NOTHING;
```

A duplicate hits the unique constraint and does nothing. The database's unique index does the
deduplication atomically (topic 22's concurrency control handles two concurrent duplicates: one wins,
one conflicts).

### 4.3 Idempotency keys

When there's no natural key — "create a payment of $49" — the **client generates a unique key per
logical operation** (a UUID) and sends it with every retry of that operation. The server remembers the
key and the result:

```
POST /payments
Idempotency-Key: 6f1c-...-a9
{ "amount": 4900, "currency": "usd" }
```

Server logic:

```
1. Look up key.
   - Not found → insert key with state "in progress" (unique constraint!), then do the work,
     then store the response (status code + body) with the key.
   - Found, completed → return the stored response; do NOT redo the work.
   - Found, in progress → another request with this key is running → return 409/retry-later.
   - Found, but request body differs → reject (client bug: same key, different operation).
2. Expire keys after a window (e.g. 24 h) longer than any client retry window.
```

Payment APIs (Stripe's `Idempotency-Key` header is the well-known example) use exactly this. Key points:
- The **client** must generate the key *once per intent* and reuse it across retries. Generating a new
  key per attempt defeats the purpose.
- The key must be stored **atomically with the effect**, or at least with an "in-progress" lock that
  survives crashes (section 4.5).
- Returning the *same response* matters: a client that timed out and retries should get the original
  result (e.g. the created payment ID), not an error.

For messages, the producer assigns a unique **message ID / event ID** at creation (not at send), and it
plays the role of the idempotency key for consumers.

### 4.4 Dedup tables (inbox pattern)

A consumer keeps a table of message IDs it has processed — sometimes called an **inbox** (the receiving
counterpart of topic 39's outbox):

```sql
CREATE TABLE processed_messages (
  consumer   text,
  message_id uuid,
  processed_at timestamptz DEFAULT now(),
  PRIMARY KEY (consumer, message_id)
);
```

```
BEGIN;
  INSERT INTO processed_messages (consumer, message_id) VALUES ('billing', $id);
     -- unique violation ⇒ duplicate ⇒ ROLLBACK and ack the message
  UPDATE accounts SET balance = balance + 50 WHERE id = 7;   -- the real effect
COMMIT;
ack message
```

Because the dedup insert and the effect are in **one database transaction**, there's no window where
one happened without the other. If we crash before COMMIT, neither is recorded and the redelivery
processes normally. If we crash after COMMIT but before the ack, the redelivery hits the unique key and
is skipped.

Costs and choices:
- **Retention:** the table grows; prune entries older than the maximum redelivery/replay window. If you
  replay older than that window, dedup no longer protects you.
- **Where it lives:** must be in the same transactional store as the effect. A dedup set in Redis
  guarding a write in Postgres is *not* atomic.
- **Per-key sequence instead of a set:** if messages for a key arrive in order (Kafka partition), you can
  store just the highest processed offset or sequence per key and skip anything ≤ it — O(keys) storage
  instead of O(messages).

### 4.5 Why "check then act" outside a transaction fails

A tempting but broken version:

```
if not redis.exists(msg.id):       # check
    charge_card(msg)               # act
    redis.set(msg.id, 1)           # record
```

Failure timelines:
- **Crash between act and record:** card charged; ID not recorded; redelivery charges again.
- **Two consumers, same message** (visibility timeout expired, topic 36): both check, both see "not
  exists", both charge. Classic race (topic 22's lost-update/write-skew family).

Fixes: make the record and the act atomic (same transaction), or *claim first* with an atomic operation
(`SET NX`, `INSERT` with unique constraint) before acting **and** make the act itself idempotent at the
external system (pass the message ID as the idempotency key to the payment provider). For external side
effects you can't put in your transaction, the only real fix is pushing idempotency into the external
system.

### 4.6 Upserts and conditional (versioned) writes

**Upsert** — "insert, or update if it exists" — makes "set state to X" idempotent:

```sql
INSERT INTO product_view (product_id, name, price, version)
VALUES (42, 'Mug', 12.00, 17)
ON CONFLICT (product_id) DO UPDATE
  SET name = EXCLUDED.name, price = EXCLUDED.price, version = EXCLUDED.version
  WHERE product_view.version < EXCLUDED.version;
```

The `WHERE version <` clause is the important bit: it makes the write **conditional on being newer**, so
a duplicate or a *late, out-of-order* old message can't overwrite newer state. This is optimistic
concurrency control (topic 22) applied to message consumption. Sources of versions: a row version from
the source DB, a Kafka offset (monotonic per partition), an LSN from CDC (topic 39).

**Design messages for idempotence.** "Set balance to 150 (version 9)" is easier to apply idempotently
than "add 50". Carrying full state or absolute values ("state transfer") instead of deltas often removes
the need for a dedup table altogether. When you do need deltas (financial ledgers), give each delta a
unique ID and insert it as a row in an append-only ledger with a unique constraint; the balance is a sum.

### 4.7 Choosing a technique

| Situation | Technique |
|---|---|
| Message carries a unique business ID | Natural key + unique constraint / `ON CONFLICT DO NOTHING` |
| Client-initiated create with no natural ID | Idempotency key, stored with the response |
| Arbitrary effect in your own DB | Dedup (inbox) table in the same transaction |
| Messages are "current state of entity X" | Upsert with version check |
| Ordered stream per key | Store last processed sequence/offset per key |
| External side effect (email, payment API) | Pass a stable ID as the external system's idempotency key; or claim-then-act with accepted rare duplicates |

Emails and push notifications are the hard case: most providers have no idempotency keys, so you
usually accept "very rarely twice" and minimise the window (claim in DB, send, mark sent).

---

## 5. Poison messages and dead-letter queues

### 5.1 Transient vs permanent failures

A consumer failure is either:
- **Transient:** the DB timed out, a dependency returned 503, a lock conflict. Retrying later will
  probably work.
- **Permanent:** the message is malformed, references an entity that will never exist, violates a
  business rule, or triggers a bug in the consumer. Retrying will **never** work.

A message that fails permanently is a **poison message**. With naive "nack and requeue" (topic 36), it
loops forever: it consumes a worker continuously, floods logs, and in a Kafka partition it blocks every
record behind it (head-of-line blocking, topic 37).

### 5.2 Retry policy

- **Classify errors.** Validation/deserialisation errors → don't retry; go straight to the DLQ.
  Timeouts/5xx → retry.
- **Bound retries.** SQS: a *redrive policy* with `maxReceiveCount` (e.g. 5) moves the message to a DLQ
  after N receives. RabbitMQ: dead-letter exchange on reject, plus a retry counter header. Kafka: no
  built-in; consumer code publishes the record to a retry topic or DLQ topic and commits.
- **Back off between retries** (topic 43): delay queues / retry topics with increasing delays (5 s, 1 min,
  10 min), never hot loops.

```
           ┌──────────── transient failure, attempts < N ─────────────┐
           │                                                           ▼
main ──► consumer ── success ──► done                       retry topic / delay queue
           │                                                  (backoff)  │
           │ permanent failure, or attempts ≥ N                         │
           ▼                                                             │
     dead-letter queue  ◄───────────────────────────────────────────────┘ (back to main)
```

### 5.3 Dead-letter queues

A **DLQ** is a queue/topic holding messages that couldn't be processed, so the main flow can continue and
humans (or tooling) can investigate. Good DLQ practice:

- **Keep context:** original message, headers, original topic/partition/offset, error message and stack
  trace, attempt count, timestamps.
- **Alert on it.** A DLQ nobody watches is data loss with extra steps.
- **Redrive tooling:** after fixing the bug or the data, move messages back to the main queue (SQS has a
  DLQ redrive feature). Redriven messages are, by definition, duplicates of earlier attempts and arrive
  *late and out of order* — consumers must be idempotent and version-aware.
- **Ordering caveat:** in an ordered stream, sending record #100 for key K to the DLQ and continuing
  with #101 for K means #101 is applied before #100. If per-key order matters, options are: stop the
  partition (block) until fixed, or park all subsequent records for that key too (track "keys currently
  in DLQ" and divert their later records), or design consumers to tolerate it via versions.

### 5.4 Block vs skip

In a log consumer, a poison record forces a choice:
- **Block** (stop and alert): preserves order and correctness; halts the whole partition.
- **Skip to DLQ**: keeps the system flowing; sacrifices order for that key and needs follow-up.

Financial ledgers often prefer blocking; analytics pipelines prefer skipping.

---

## 6. Replay safety

### 6.1 Why replay happens

Log-based systems (topic 37) invite replay: rewind a consumer group to fix a bug, rebuild a derived store,
bootstrap a new service, redrive a DLQ, or recover after restoring a backup. Replay is a mass, deliberate
duplicate delivery.

### 6.2 What makes a consumer replay-safe

1. **Idempotent writes** (section 4): reprocessing an event leaves state as it was.
2. **Deterministic processing:** the output depends only on the event (and state derived from earlier
   events), not on "now", random numbers, or current values in other services. If a handler does
   `if now() > order.deadline`, replaying a year later gives a different answer. Use the event's
   timestamp (event time, topic 41) instead.
3. **No external side effects on replay** — or side effects guarded by idempotency keys. Replaying a
   year of `OrderPlaced` must not re-send a year of confirmation emails or re-charge cards. Common
   approach: separate the "update derived state" consumers (replay freely) from "trigger side effects"
   consumers (never replay, or check a sent-log first).
4. **Versioned / conditional writes** so that replaying old events doesn't overwrite newer state that
   came from elsewhere.
5. **Schema compatibility:** the current consumer code must still parse old events (topic 16).
6. **Dedup window covers the replay range** — if dedup records expire after 7 days and you replay 30,
   dedup won't help; rely on natural keys and versions instead.

### 6.3 A worked example

A "loyalty points" consumer reads `OrderPlaced` and does `points = points + order.total / 10`.
A bug miscounted points for a week; you rewind 7 days.

- Naive handler: every order in the week is counted **again** — users get double points.
- Better: store each award as a row `points_ledger(order_id PRIMARY KEY, user_id, points)` and compute
  the balance as a sum. Replay with corrected code would hit conflicts on `order_id`… and *not* fix the
  wrong values. So for a correction, use `ON CONFLICT (order_id) DO UPDATE SET points = EXCLUDED.points`
  — the replay overwrites each award with the corrected value: idempotent **and** corrective.

The lesson: design derived state so that "reprocess event E" means "recompute E's contribution", not
"add E's contribution again".

---

## 7. How it connects

- **Topic 07 (REST):** idempotent HTTP methods; idempotency keys on POST.
- **Topic 21/22 (transactions, concurrency):** atomic dedup-plus-effect, unique constraints, optimistic
  version checks.
- **Topic 26 (partial failure):** the ambiguous timeout is the root of all of this.
- **Topic 31 (fencing):** producer epochs fence zombie producers.
- **Topic 34 (2PC):** Kafka transactions are an internal two-phase commit.
- **Topic 35 (sagas):** every saga step and compensation must be idempotent because it will be retried.
- **Topic 36/37:** ack/commit placement determines the base semantics.
- **Topic 39 (outbox/CDC):** outbox relays are at-least-once; consumers dedupe by event ID.
- **Topic 41 (stream processing):** exactly-once state via checkpoints; replay-safe determinism.
- **Topic 43 (retries):** backoff and retry budgets for transient failures.

---

## Common misconceptions

- **"Our broker supports exactly-once, so consumers don't need to be idempotent."** → Broker-level
  exactly-once covers the broker's boundary. Your DB writes and API calls are outside it.
- **"Exactly-once means the code runs once."** → It means the *effect* is applied once; code may run
  several times.
- **"At-least-once only produces duplicates when we retry."** → Redelivery, rebalances, upstream retries,
  replays and DLQ redrives all produce duplicates.
- **"Checking a Redis set before processing makes it idempotent."** → Not unless the check/record is
  atomic with the effect; crashes and races leave gaps.
- **"Generate a new idempotency key on each retry."** → The key must identify the *intent* and be reused
  across all retries of it.
- **"An upsert is enough for out-of-order messages."** → A plain upsert lets an old duplicate overwrite
  newer state; add a version condition.
- **"The DLQ solves poison messages."** → It moves them aside; without alerts, investigation and safe
  redrive, it's just a slower way to lose data.

## Further reading (supplements)

- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 11 ("Fault tolerance" in stream
  processing; idempotence) and Chapter 12 ("The end-to-end argument for databases", "Enforcing
  constraints").
- Saltzer, Reed & Clark, "End-to-End Arguments in System Design" (1984) — why dedup must ultimately live
  at the endpoints.
- Confluent blog, "Exactly-Once Semantics Are Possible: Here's How Kafka Does It" (Neha Narkhede, 2017),
  and KIP-98 (idempotent producer and transactions).
- Stripe engineering blog, "Designing robust and predictable APIs with idempotency" (Brandur Leach, 2017).
- AWS documentation, "Amazon SQS dead-letter queues".

---

## Questions

1. A producer sends a message and times out waiting for the ack. List the three things that might have
   happened and explain why no transport protocol can tell them apart.
2. Why do almost all business systems choose at-least-once rather than at-most-once?
3. A vendor says their queue offers "exactly-once delivery". What two questions would you ask to
   understand what they really guarantee?
4. Explain how Kafka's idempotent producer dedupes retries. Why doesn't it help if the application
   restarts and re-sends the same business event?
5. A Kafka Streams app with exactly-once enabled also sends an SMS for each record. Can a customer get
   two SMSes? Why?
6. A consumer stores processed message IDs in Redis and the side effect in Postgres. Describe two
   timelines in which duplicates still occur.
7. Rewrite `UPDATE accounts SET balance = balance + 50 WHERE id = 7` processing so redelivery is harmless.
   Give two different approaches.
8. A client times out on `POST /payments` and retries with a new idempotency key. What happens, and
   whose bug is it?
9. Messages carry a product's full current state. A plain upsert is used. Explain how a redelivered old
   message causes a bug and fix it.
10. A Kafka consumer hits a record that fails deserialisation. What happens with "retry forever", and what
    are the options?
11. Why is it dangerous to redrive a DLQ into an order-sensitive consumer without thinking?
12. You rewind a consumer group 10 days. The consumer's dedup table keeps IDs for 3 days. What protects
    you, if anything?
13. A handler computes "is this order late?" using `now()`. Why is that a problem for replay, and what's
    the fix?
14. Your consumer both updates a read model and sends a "welcome" email. You need to replay it to rebuild
    the read model. How should you structure things?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** The request was lost; the request was processed but the ack was lost; or the receiver is still
processing (slow). The sender's only signal is the absence of a reply within a deadline, which is
identical in all three cases — this is inherent to asynchronous networks with crashes (topic 26).

**2.** Loss is usually unrecoverable and invisible, while duplicates can be neutralised by idempotent
processing. At-least-once + idempotence yields correct effects; at-most-once silently drops work.

**3.** "Within what boundary — does it cover my consumer's database writes and external calls?" and
"Exactly-once delivery or exactly-once effect — what happens if my consumer crashes after doing the work
but before acking?" (Any honest answer will involve idempotence or transactions on your side.)

**4.** Each producer session has a PID and per-partition sequence numbers; the broker tracks the last
sequence and discards a batch it has already appended (acking it as success). A restart creates a new
PID (or the app re-sends with a new sequence), so the broker sees a new, valid record; it can't know
it's the same business event. That needs an application-level event ID.

**5.** Yes. Exactly-once covers Kafka state and output topics. If the task crashes after sending the SMS
but before the transaction commits, the record is reprocessed and the SMS sent again. Fix: make the SMS
call idempotent (provider idempotency key / sent-log claimed transactionally) or move it to a separate
consumer that dedupes.

**6.** (a) Postgres commit succeeds, crash before writing the ID to Redis → redelivery finds no ID →
effect applied again. (b) Two consumers receive the same message concurrently (visibility timeout
expired), both check Redis, both find nothing, both apply. Also (c) Redis loses the key on failover.

**7.** (a) Dedup/inbox table: insert message ID into `processed_messages` and do the update in the same
Postgres transaction; a unique violation means skip. (b) Ledger: insert a row `(message_id PK, account_id,
+50)` with `ON CONFLICT DO NOTHING` and compute balance as a sum (or maintain it in the same transaction
only when the insert succeeded). Alternatively, (c) use absolute state with a version check.

**8.** The server sees a new key, treats it as a new payment, and charges again. It's the client's bug:
the key must be generated once per logical operation and reused on every retry.

**9.** Suppose v17 (price 12) arrives, then a redelivered v16 (price 10). A plain upsert writes v16 over
v17, rolling the price back. Fix: conditional upsert `... DO UPDATE ... WHERE stored.version <
EXCLUDED.version`.

**10.** It never succeeds, the consumer never commits past it, and the whole partition stalls with
growing lag (head-of-line blocking) while logs flood. Options: classify as permanent and send straight to
a DLQ topic (with metadata) and continue; or block and alert if ordering/correctness requires.

**11.** Redriven messages are duplicates and are out of order relative to later messages for the same
keys. A consumer that applies them blindly may overwrite newer state with older or double-apply. It must
be idempotent and version-aware, or redrive must be done per key in order.

**12.** The dedup table only covers the last 3 days, so the first 7 days of the replay aren't protected by
it. What protects you is natural-key uniqueness, conditional/versioned upserts, and absolute-state
writes. If the consumer does `balance += x`, nothing protects you.

**13.** `now()` during replay is days later, so every order looks late — the output differs from the
original run; processing isn't deterministic. Use the event's own timestamp (event time) and data in the
event or derived state.

**14.** Split them: one consumer group maintains the read model (idempotent upserts, replayable), another
consumer group sends emails (never rewound, or checks a "sent" table keyed by user ID before sending).
Replay only the read-model group.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Why can't a sender distinguish loss from a lost ack?
**A:** Both look like a timeout; the network gives no other signal.

**Q:** At-most-once in one line?
**A:** No retries / ack before processing; may lose, never duplicates.

**Q:** At-least-once in one line?
**A:** Retry until acked / ack after processing; never loses, may duplicate.

**Q:** What does exactly-once really mean in practice?
**A:** Exactly-once effect: at-least-once delivery plus deduplication/idempotence, within a boundary.

**Q:** Where does a broker's exactly-once guarantee stop?
**A:** At its boundary — external DB writes, emails and API calls aren't covered.

**Q:** How does Kafka's idempotent producer work?
**A:** Producer ID + per-partition sequence numbers; broker drops already-seen batches.

**Q:** What do Kafka transactions make atomic?
**A:** Writes to multiple partitions plus the consumer offset commit.

**Q:** What does read_committed do?
**A:** Consumers skip records from aborted transactions.

**Q:** General trick for exactly-once into an external DB?
**A:** Store the consumer offset in the same transaction as the effect.

**Q:** Define idempotent.
**A:** Applying it once or many times gives the same state.

**Q:** Natural key dedup?
**A:** Use a unique business ID as a unique key; duplicates conflict and do nothing.

**Q:** What is an idempotency key?
**A:** A client-generated ID per logical operation, reused on retries; server stores key + response.

**Q:** Biggest idempotency-key mistake?
**A:** Generating a new key per attempt instead of per intent.

**Q:** What is an inbox/dedup table?
**A:** A table of processed message IDs, inserted in the same transaction as the effect.

**Q:** Why is Redis-check-then-Postgres-write unsafe?
**A:** Not atomic: crashes between steps and concurrent consumers both slip through.

**Q:** Why add a version condition to an upsert?
**A:** So old/duplicate messages can't overwrite newer state.

**Q:** Delta vs state messages for idempotence?
**A:** Absolute state ("set to X, v9") is naturally idempotent; deltas ("add 50") need unique IDs.

**Q:** What is a poison message?
**A:** A message that fails permanently; retrying never succeeds.

**Q:** What is a DLQ?
**A:** A queue/topic for messages that couldn't be processed, kept for inspection and redrive.

**Q:** SQS mechanism to move poison messages?
**A:** Redrive policy with maxReceiveCount → DLQ.

**Q:** What to store with a DLQ message?
**A:** Original payload/headers, source position, error, attempt count, timestamps.

**Q:** Block vs skip on a poison record in a partition?
**A:** Block keeps order but stalls; skip keeps flowing but breaks order for that key.

**Q:** Two requirements for replay safety?
**A:** Idempotent writes and deterministic processing (no now(), randomness, live lookups).

**Q:** How to replay without re-sending emails?
**A:** Separate side-effect consumers from state-building consumers; replay only the latter.
