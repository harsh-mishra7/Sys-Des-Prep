# 35 — Sagas

> Part 6 — Distributed Systems · Prerequisites: topics 21, 22, 34

Topic 34 showed that two-phase commit gives atomicity across nodes, at the price of holding locks across
network round trips, needing every participant to be available, and requiring every participant to speak the
same protocol. For a business process that spans several independent services (orders, payments,
inventory, shipping), each with its own database, and which may take seconds, hours or days (waiting for a
payment provider, a warehouse, a human approval), those prices are unaffordable. You can't hold a row lock on
inventory for two days, and a third-party payment API doesn't do 2PC. The **saga** pattern is the
alternative: break the business transaction into a sequence of local transactions, each committed
immediately, and if a later step fails, undo the earlier ones with **compensating** transactions. This topic
covers how sagas work, the two ways of coordinating them, the TCC variant, and the price you pay: sagas give
up isolation, so you have to design around the anomalies that result.

By the end of this topic you should be able to:
1. Explain why long-lived business transactions can't use distributed locks or 2PC, and what a saga is.
2. Design compensating actions for a saga, and explain what makes a compensation valid (and what can't be compensated).
3. Compare choreography and orchestration, including their failure handling and observability.
4. Explain TCC (try-confirm-cancel) and how it differs from a plain saga.
5. Name the isolation anomalies sagas suffer from (lost updates, dirty reads, fuzzy reads) and apply countermeasures such as semantic locks, commutative updates, pessimistic ordering, re-reading values, and versioning.
6. Explain the reliability machinery a saga needs: durable state, idempotent steps, retries, and the outbox.

---

## 1. Long-lived business transactions

### 1.1 The problem

Consider placing an order on a shopping site where each capability is a separate service with its own database:

```
  Order service      Inventory service     Payment service       Shipping service
  (orders DB)        (stock DB)            (talks to card PSP)   (talks to carrier)
     create order  →   reserve stock    →    charge card       →   schedule shipment
```

We want "all or nothing": if the card is declined, the stock must not stay reserved and the order must not
look confirmed. On a single database (topic 21) this would be one ACID transaction. Here:

- **2PC is impractical** (topic 34): the payment provider and carrier are external and don't support it; locks
  would be held across slow external calls; any one service's outage blocks all; the coordinator's failure
  strands in-doubt transactions.
- **Some steps take a long time.** A payment may need 3-D Secure confirmation by the user; a fraud review may
  take an hour; a shipment is scheduled days later. Holding database locks for that long would serialise
  everything behind them.
- **Distributed locks don't help** (topic 31): they'd be long-held, fragile under failure, and still not
  give atomic rollback.

### 1.2 The saga idea

Garcia-Molina and Salem (1987) introduced sagas for long-lived transactions in a single database; the idea
transfers directly to microservices:

- A **saga** is a sequence of local transactions T1, T2, ..., Tn. Each Ti commits immediately in its own
  service, releasing its locks.
- For each Ti (except possibly the last) there is a **compensating transaction** Ci that semantically undoes Ti.
- If Tk fails, the saga runs Ck−1, ..., C1 in reverse order.

```
success:   T1 → T2 → T3 → T4                                  (all committed)
failure:   T1 → T2 → T3 ✗ → C2 → C1                           (T3 failed; undo T2, T1)
```

Guarantee: either all of T1..Tn eventually complete, or the completed ones are compensated. This is
**atomicity in an eventual, semantic sense**. What is lost compared to an ACID transaction:

- **Isolation (the I in ACID).** Intermediate states (order created, stock reserved, card not charged yet) are
  committed and visible to other transactions. Section 5 covers the consequences.
- **Instant rollback.** Undo is a new forward transaction that runs later, may fail, and must be retried.

Sagas are often summarised as ACD: atomic (eventually), consistent (per service, and eventually overall),
durable, but not isolated.

---

## 2. Compensating actions

### 2.1 What compensation is (and isn't)

A compensating transaction is **not a rollback**. Ti has committed, other transactions may have seen and used
its result, and side effects (an email, a charge) may have happened in the real world. The compensation is a
*new business action* that restores a semantically acceptable state.

| Step Ti | Compensation Ci |
|---|---|
| Create order (status PENDING) | Mark order CANCELLED (not delete: keep the audit trail) |
| Reserve 2 units of stock | Release the 2-unit reservation |
| Charge card €50 | Refund €50 (a separate transaction with the provider; the charge stays on the statement) |
| Send "order confirmed" email | Send "sorry, your order was cancelled" email |
| Book a hotel room | Cancel the booking (maybe with a cancellation fee) |

Note the asymmetry: you can't "unsend" an email or make a refund invisible. Compensation restores the
*business* invariant, not the bit-for-bit previous state.

### 2.2 Requirements on compensations

- **Must eventually succeed.** If a compensation can fail permanently, the saga gets stuck half-done. So
  compensations are retried until they succeed, and must be designed to always be possible (e.g. releasing
  a reservation always works; refunding is always allowed). If it truly can't succeed, escalate to a human
  (alerting, a manual-intervention queue).
- **Idempotent.** Retries and redelivered messages mean a compensation may run more than once. Releasing the
  same reservation twice must not release someone else's stock. Use the saga ID / reservation ID as an
  idempotency key (topic 38).
- **Commutative with the original step, where possible.** Messages can be reordered: the compensation might
  arrive before the step it compensates (e.g. a timeout fired, the saga decided to cancel, but the original
  "reserve" request was still in flight). The service should record "cancelled for saga 42" so that a late
  "reserve for saga 42" is rejected or immediately undone. This is sometimes called handling the **"dangling"**
  or out-of-order compensation.

### 2.3 Pivot transactions and step ordering

Steps fall into three kinds (Richardson's terminology):

- **Compensatable transactions:** can be undone (reserve stock, create pending order).
- **Pivot transaction:** the go/no-go point. If it succeeds, the saga will run to completion; it is neither
  compensated nor retried-as-optional. Often the step that's hardest to undo, like charging the card or
  submitting to an external system.
- **Retriable transactions:** come after the pivot and are guaranteed to eventually succeed with retries
  (mark order CONFIRMED, notify shipping, send confirmation email).

```
 compensatable        pivot        retriable
 T1 create order  →  T3 charge  →  T4 confirm order → T5 schedule shipping → T6 send email
 T2 reserve stock
```

Design principle: **put the steps that can fail for business reasons, or are hard to undo, as late as possible
before the pivot, and put irreversible side effects after it.** Don't send the confirmation email at step 1;
send it once the outcome is certain. Do cheap validation first (fail early, little to compensate).

### 2.4 Forward recovery vs backward recovery

- **Backward recovery:** on failure, compensate completed steps (the default described above).
- **Forward recovery:** after the pivot, a failure is handled by retrying until success rather than
  compensating. Shipping is temporarily down → keep retrying; don't refund a paid order because of a transient
  outage.

---

## 3. Choreography vs orchestration

There are two ways to coordinate who does what next.

### 3.1 Choreography

Each service listens for events and decides on its own what to do, publishing new events. No central component.

```
 Order svc ──OrderCreated──►  Inventory svc ──StockReserved──►  Payment svc
     ▲                              ▲                                │
     │                              │                         PaymentFailed
     │                              └──────(release stock)◄──────────┤
     └────────────────────(cancel order)◄────────────────────────────┘
                 (or on PaymentSucceeded → Order svc marks CONFIRMED)
```

Pros:

- Simple for short sagas; no extra service to build or run.
- Loose coupling: services only know events, not each other's APIs.
- No central point of failure.

Cons:

- **The workflow is implicit.** To understand "how is an order placed?", you must read every service's
  event handlers. Hard to reason about and to change.
- **Cyclic dependencies:** order service listens to payment events, payment listens to inventory events,
  inventory listens to order events.
- **Hard to answer "what state is saga 42 in?"** The state is spread across services; you need tracing
  (topic 46) to reconstruct it.
- **Compensation logic is scattered**, and adding a step means changing several services.
- **Risk of event spaghetti** as sagas grow beyond 3–4 steps.

### 3.2 Orchestration

A central **orchestrator** (a saga execution coordinator) tells each participant what to do, using commands
and replies, and keeps the saga's state as an explicit state machine.

```
                        ┌──────────────────────────────┐
                        │  Order saga orchestrator     │  state persisted:
                        │  saga 42: STOCK_RESERVED     │  {step, data, history}
                        └──┬───────────┬───────────┬───┘
         ReserveStock cmd  │           │ Charge    │ ReleaseStock (on failure)
                           ▼           ▼           ▼
                    Inventory svc   Payment svc   Inventory svc
                           │ reply      │ reply
                           └─────►──────┴──►── orchestrator advances state machine
```

The orchestrator's state machine for the order example:

```
 PENDING ─reserve ok─► STOCK_RESERVED ─charge ok─► PAID ─ship scheduled─► CONFIRMED
    │                       │                       
 reserve fail          charge fail                  
    ▼                       ▼                       
 CANCELLED ◄──release ok── RELEASING_STOCK
```

Pros:

- **Explicit, centralised workflow:** one place shows the whole saga and its compensation paths.
- **Easy to see state** of each saga instance; easy to add timeouts ("if payment hasn't replied in 15 min,
  cancel").
- Participants are simpler: they expose commands (reserve, release) and don't need to know about the saga.
- No cyclic dependencies: the orchestrator depends on participants, not the other way round.

Cons:

- One more component to run; its state must be durable and highly available.
- Risk of a "smart orchestrator, anaemic services" design, where business logic piles up centrally.
- Coupling to the orchestrator.

Workflow engines such as Temporal, Cadence, AWS Step Functions, Camunda and Netflix Conductor are general
orchestrators: they persist each workflow's state and history, handle retries and timeouts, and let you write
the saga as (durable) code.

### 3.3 Choosing

| | Choreography | Orchestration |
|---|---|---|
| Where the flow lives | Spread across event handlers | In one state machine |
| Good for | Short sagas (2–4 steps), simple flows | Longer or branching sagas, timeouts, human steps |
| Observability | Needs distributed tracing to reconstruct | Query the orchestrator |
| Coupling | Event-level, but cycles appear | Participants decoupled from each other, coupled to orchestrator |
| Failure point | None central | Orchestrator (made durable/HA) |

Many systems mix: orchestration within a domain, choreography (domain events) between domains.

### 3.4 Reliability machinery (needed either way)

Sagas run over unreliable networks (topic 26), so every interaction can be lost, duplicated or delayed.

- **Durable saga state.** The orchestrator persists the current step *before* sending the next command, so a
  crash resumes rather than restarts or forgets the saga.
- **Atomic "update state + send message".** A service must commit its local transaction and publish its event
  (or the orchestrator must save state and send its command) atomically. Otherwise a crash between the two
  leaves the saga stuck or diverged. The standard solution is the **transactional outbox** (topic 39): write
  the message to an outbox table in the same local transaction; a relay publishes it.
- **At-least-once delivery + idempotent handlers** (topic 38): messages will be redelivered; every step and
  compensation dedups by saga ID and step.
- **Timeouts:** a participant may never reply. The orchestrator must decide (after a timeout) to retry, or to
  compensate, knowing the original command might still be applied later (section 2.2).
- **Monitoring of stuck sagas** and a manual-repair path; in practice there will always be some.

---

## 4. TCC: try-confirm-cancel

### 4.1 The pattern

TCC (popularised by Pat Helland and others, and used widely in payments and booking systems) splits each
participant's work into three operations:

- **Try:** check business rules and **reserve** resources, without making the effect final. E.g. place a hold
  of €50 on the card (authorisation), mark 2 units of stock as reserved, tentatively hold a seat.
- **Confirm:** turn the reservation into the real effect (capture the €50, decrement stock, issue the ticket).
  Must succeed if Try succeeded (only retried, never fails for business reasons).
- **Cancel:** release the reservation (void the hold, unreserve stock).

The coordinator calls Try on all participants; if all succeed, Confirm on all; otherwise Cancel on those that
tried.

```
 coordinator ─Try─► Payment (auth hold €50)   ✓
             ─Try─► Inventory (reserve 2)     ✓
             ─Try─► Seats (hold 14A)          ✗ (taken)
             ─Cancel─► Payment (void hold)
             ─Cancel─► Inventory (unreserve)
```

### 4.2 TCC vs plain saga vs 2PC

- **Like 2PC** in shape: a tentative phase then a decision phase. But Try is an *application-level*
  reservation that is visible and committed in each service, not a database lock held open, and reservations
  usually **expire** (a card authorisation lapses after some days; a seat hold after 10 minutes), which
  avoids 2PC's indefinite blocking.
- **Better isolation than a plain saga:** other transactions see "2 reserved" rather than "2 sold", so they
  can treat the resources as unavailable without the effect being final; no real-world effect happens until
  Confirm. Compensation (Cancel) is simpler because nothing irreversible happened yet.
- **Cost:** every participant must implement three operations and a reservation model, and needs to handle
  out-of-order calls (Cancel before Try arrives, Confirm after reservation expiry). Third-party services must
  natively support reservations (card networks do: auth/capture/void).

Pitfalls specific to TCC:

- **Empty cancel:** Cancel arrives for a Try that never executed (it was lost). Cancel must succeed as a no-op.
- **Hanging try:** Try arrives *after* Cancel (delayed in the network). The participant must remember the
  cancel and reject the late Try, or reserved resources leak.
- **Idempotence** of all three operations.

---

## 5. Isolation anomalies in sagas and countermeasures

Because each step commits immediately, sagas run without isolation between them. Concurrent sagas (and
ordinary transactions) can see and interfere with each other's intermediate states. The anomalies are the
distributed equivalents of those in topic 21:

### 5.1 The anomalies

**Lost updates.** One saga overwrites a change made by another without reading it.

```
saga A (create order 7): T1 creates order 7 PENDING
saga B (cancel order 7):  cancels order 7 → CANCELLED
saga A: T4 marks order 7 CONFIRMED          ← B's cancellation lost; customer charged for a cancelled order
```

**Dirty reads.** A saga or a query reads data written by another saga's step that is later compensated.

```
saga A: reserves the last unit of item X (stock = 0)
saga B: sees stock = 0, rejects a customer's order ("sold out")
saga A: payment fails → compensates → stock = 1
→ B turned away a customer for stock that was never actually sold
```

Worse version: a credit-limit check reads a balance lowered by a step that is later compensated, or a report
counts a sale that gets refunded.

**Fuzzy / non-repeatable reads.** Two steps of the same saga read the same data and get different values
because another saga changed it in between, so the saga makes decisions based on inconsistent views.

### 5.2 Countermeasures

These come from the literature on sagas (Garcia-Molina & Salem; Lars Frank's "semantic ACID" work) as
catalogued in Richardson's *Microservices Patterns*.

**Semantic lock.** A compensatable step sets an application-level flag on the records it touches,
indicating "this record is part of an in-progress saga and may change". E.g. the order's status is
`APPROVAL_PENDING` or `CANCEL_PENDING`, not just PENDING/CONFIRMED. Other sagas and queries that see the flag can:

- fail ("order is being processed, try again"), or
- wait/retry until the flag clears, or
- proceed but treat the value as provisional (show "pending" in the UI).

The flag is cleared by the final retriable step (CONFIRMED) or by the compensation (CANCELLED). This prevents
the lost-update example above: saga B sees `APPROVAL_PENDING` and refuses or waits. Unlike a database lock it
survives across services and long durations; the cost is that the application must handle the "locked" state,
including deadlocks between sagas and stuck flags (use timeouts).

**Commutative updates.** Design operations so the order doesn't matter: `debit(30)` and `credit(30)` instead of
`set balance = 70`. A compensation (`credit(30)`) then commutes with other sagas' updates, so undoing one saga
doesn't overwrite another's changes. This removes lost updates for numeric data. (It's the same idea as CRDT
counters in topic 33.)

**Pessimistic view (reordering steps).** Reorder the saga to minimise the business risk of dirty reads. E.g.
for "cancel order" which releases stock and refunds credit: do the step whose dirty read would be harmful
later, or move the risky update into a retriable step (after the pivot) where it can't be compensated, so
nobody ever reads a value that will be rolled back.

**Reread value.** Before overwriting a record, re-read it and verify it hasn't changed since the saga first
read it; if it has, abort and possibly restart the saga. This is optimistic concurrency control (topic 22)
across saga steps, typically using a version number column.

**Version file.** Record the operations applied to a record so they can be reordered correctly when they
arrive out of order. E.g. the payment service gets "cancel authorisation" before "authorise" for the same
saga; it logs the cancel, and when the authorise arrives, it knows to skip it. This is the general form of the
"hanging try" fix.

**By value (risk-based choice).** Choose the mechanism per request based on its business risk: low-value
orders use a saga; high-value or high-risk ones go through a path with stronger guarantees (e.g. a distributed
transaction within one database, or extra manual checks).

### 5.3 Putting it together: the order saga, hardened

```
T1  Order:      create order status=APPROVAL_PENDING           (semantic lock)
T2  Inventory:  reserve stock (reservation row, saga_id)        (TCC-style: reserved, not sold)
T3  Payment:    authorise card (hold) — PIVOT-ish               (TCC try; idempotency key = saga_id)
T4  Payment:    capture                                         (retriable)
T5  Inventory:  convert reservation to sale                     (retriable)
T6  Order:      status=CONFIRMED (clears lock); send email      (retriable)

Compensations: C3 void authorisation, C2 release reservation, C1 status=CANCELLED (clears lock)
All steps: idempotent by (saga_id, step); state changes + outgoing messages via outbox.
```

Concurrent "cancel order" sagas see APPROVAL_PENDING and wait or fail. Stock queries can show "available"
vs "reserved" separately, so a dirty read of a reservation doesn't make the product look sold out if the UI
chooses not to count reservations. Balance-like counters use increments, not sets.

---

## 6. How it connects

- **Topic 21 (transactions):** sagas trade the I of ACID for availability and autonomy; the anomalies mirror
  lost updates, dirty reads and non-repeatable reads.
- **Topic 22 (concurrency control):** semantic locks are application-level pessimistic locks; reread-value is
  optimistic concurrency control.
- **Topic 31 (locks):** why long-held distributed locks are not the answer.
- **Topic 33 (CRDTs):** commutative updates make compensations safe to reorder, like CRDT counters.
- **Topic 34 (2PC):** the alternative with full atomicity and isolation; TCC resembles 2PC at the application level.
- **Topic 36–38 (queues, Kafka, delivery semantics):** saga messages are delivered at least once; every
  step must be idempotent.
- **Topic 39 (outbox & CDC):** atomically commit a step and publish its event.
- **Topic 42 (event sourcing):** choreographed sagas often react to domain events; event-sourced services
  naturally record saga history.
- **Topic 43 (timeouts & retries):** orchestrators need timeouts to decide between retrying and compensating.
- **Topic 46 (observability):** tracing is essential to follow choreographed sagas.

---

## Common misconceptions

- "A compensating transaction is a rollback." → It's a new forward action that semantically undoes the step;
  the original stays committed and may already have been seen or had external effects.
- "Sagas are ACID across services." → They provide eventual atomicity, not isolation; intermediate states are
  visible to others.
- "Compensation can't fail." → It can fail transiently, so it must be idempotent and retried; if it fails
  permanently, the saga needs human intervention.
- "Choreography is always better because it's decoupled." → For longer flows it hides the workflow, creates
  cyclic dependencies and makes it hard to see saga state. Orchestration is often clearer.
- "TCC is just 2PC." → The shape is similar, but Try creates committed, visible, application-level reservations
  that usually expire, rather than holding database locks while blocking.
- "If every step is idempotent, the saga is correct." → Idempotency handles duplicates; you also need ordering
  handling (late tries, early cancels) and isolation countermeasures.
- "Sagas remove the need for transactions." → Each step is still a local ACID transaction, and the step
  plus its outgoing message must be atomic (outbox).

---

## Further reading (supplements)

- Garcia-Molina & Salem, "Sagas" (SIGMOD 1987).
- Chris Richardson, *Microservices Patterns* (Manning, 2018), Ch. 4 ("Managing transactions with sagas"),
  including the countermeasures catalogue; also microservices.io pages on the saga pattern.
- Pat Helland, "Life beyond Distributed Transactions: an Apostate's Opinion" (CIDR 2007).
- Caitie McCaffrey, "Distributed Sagas: A Protocol for Coordinating Microservices" (talk, 2015–2017).
- Lars Frank & Torben Zahle, "Semantic ACID properties in multidatabases using remote procedure calls and update
  propagations" (1998), source of several countermeasures.
- Kleppmann, *Designing Data-Intensive Applications*, Ch. 9 (limits of distributed transactions) and Ch. 12
  ("Enforcing constraints", "Timeliness and integrity").
- Temporal or AWS Step Functions documentation on the saga pattern.

---

## Questions

1. Why is 2PC unsuitable for an order flow that spans an internal inventory service, an external payment
   provider and a shipping partner?
2. Define a saga and explain what guarantee it gives in place of ACID atomicity.
3. Why is a compensating transaction not the same as a rollback? Give an example where the difference is visible.
4. What properties must a compensating action have, and why?
5. What is a pivot transaction, and how should steps be ordered around it? Where should you send the
   "order confirmed" email, and why?
6. A command to reserve stock times out. The orchestrator decides to cancel and sends "release reservation".
   Then the original reserve request arrives. What goes wrong, and how do you prevent it?
7. Compare choreography and orchestration. For a 7-step saga with timeouts and a manual approval step, which
   would you choose and why?
8. A service updates its database and then publishes an event, but crashes between the two. What happens to the
   saga, and what pattern fixes it?
9. Explain TCC. How does it give better isolation than a plain saga?
10. What are the "empty cancel" and "hanging try" problems in TCC?
11. Give an example of a dirty read caused by a saga, and a countermeasure.
12. Saga A is creating order 7 while saga B cancels it. Show how a lost update happens and how a semantic lock prevents it.
13. Why do commutative updates help with compensation? Give an example.
14. What does the "reread value" countermeasure correspond to in single-database concurrency control?
15. An engineer argues: "Our saga steps are idempotent and we use the outbox, so we have ACID." What's missing?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** The external parties don't participate in 2PC; locks would be held for the duration of slow external
calls (possibly minutes or days for payment authentication or shipping); every service must be up for any order
to commit; and a coordinator failure would leave in-doubt transactions holding locks. The flow needs each step
to commit independently.

**2.** A sequence of local transactions, each committed in its own service, with a compensating transaction for
each step. Guarantee: either all steps eventually complete, or every completed step is eventually compensated.
Atomicity is eventual and semantic; isolation is not provided.

**3.** A rollback erases uncommitted changes as if they never happened. A compensation is a new committed action
after the original was committed and possibly observed. Example: a card charge compensated by a refund; the
customer sees both the charge and the refund on their statement. Or an email sent then "cancellation" email.

**4.** It must eventually succeed (retried until it does, designed to always be possible), otherwise the saga is
stuck half-done; it must be idempotent, because retries and redelivery will run it more than once; and it should
handle reordering with the original step (compensation arriving first), because messages can be delayed.

**5.** The pivot is the go/no-go step: once it succeeds the saga runs to completion (with retries) and is never
compensated. Compensatable steps go before it, ordered so likely-to-fail checks happen early; irreversible side
effects and retriable steps come after. The confirmation email goes after the pivot, because before it the
order might still be cancelled and you can't unsend an email.

**6.** The release finds no reservation and does nothing (or fails), then the late reserve succeeds, leaving stock
reserved forever for a cancelled saga (leaked resources). Prevent it by recording the cancellation keyed by
saga ID (a version file / tombstone); when the late reserve arrives, the service sees the saga was cancelled and
rejects it. And make release a no-op success if nothing was reserved (empty cancel).

**7.** Choreography: services react to each other's events; decentralised and loosely coupled, but the flow is
implicit, cycles appear, and state is hard to see. Orchestration: a central state machine sends commands; explicit
flow, easy timeouts and status, but one more durable component. For 7 steps with timeouts and a human step,
orchestration (e.g. a workflow engine), since timeouts, waiting and visibility are central to the flow.

**8.** The local change is committed but the event is never published, so the next step never runs (or the
orchestrator never learns), and the saga hangs in an intermediate state. The transactional outbox: write the
event to an outbox table in the same local transaction and have a relay (or CDC) publish it, with at-least-once
delivery and idempotent consumers.

**9.** Each participant offers Try (check and reserve resources tentatively), Confirm (make the reservation real;
must succeed) and Cancel (release). The coordinator tries all, then confirms all or cancels the tried ones.
Isolation is better because other transactions see resources as "reserved", not consumed; no irreversible
effect happens until Confirm, and Cancel only releases a hold.

**10.** Empty cancel: Cancel arrives for a Try that never ran (lost); the participant must treat it as a successful
no-op. Hanging try: Try arrives after Cancel (delayed); the participant must remember the cancel and refuse the
Try, or the reservation leaks.

**11.** Saga A reserves the last unit; saga B sees stock 0 and rejects a customer; A's payment fails and it releases
the unit. B acted on data that was compensated. Countermeasures: semantic lock / separate "reserved" from "sold"
so B treats reserved stock as provisional; or reorder steps (pessimistic view) so the stock decrement happens
after the payment pivot.

**12.** A creates order 7 (PENDING). B cancels it (CANCELLED). A's last step sets CONFIRMED, overwriting B's
cancellation, and the customer is charged for a cancelled order. With a semantic lock, A creates the order as
APPROVAL_PENDING; B sees the flag and fails or waits; once A finishes (CONFIRMED) B can run its cancel saga
properly (with refund), or if A compensates (CANCELLED) B has nothing to do.

**13.** Commutative operations give the same result in any order, so a compensation doesn't overwrite changes made
by other sagas in between. Example: saga A debits 30 (balance 100→70), saga B debits 20 (→50), A compensates by
credit(30) (→80). With "set balance = 100" as the compensation, B's debit would be lost.

**14.** Optimistic concurrency control: read a version, and before writing check it's unchanged (compare-and-set);
abort/restart if not. It prevents lost updates.

**15.** Isolation. Idempotency and the outbox ensure each step happens exactly once in effect and messages
aren't lost, which gives eventual atomicity and durability. But intermediate states are still visible to
other sagas and queries, so dirty reads, lost updates and fuzzy reads can occur unless countermeasures are designed in.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What is a saga?
**A:** A sequence of local transactions, each with a compensating transaction to undo it if a later step fails.

**Q:** Why not 2PC for long-lived business transactions?
**A:** Locks held for long durations, external services don't support it, needs all participants up, blocking.

**Q:** Which ACID property do sagas give up?
**A:** Isolation.

**Q:** Compensation vs rollback?
**A:** Compensation is a new forward action that semantically undoes a committed step; rollback discards uncommitted changes.

**Q:** Three requirements for compensations?
**A:** Eventually succeed (retry), idempotent, tolerate reordering with the original step.

**Q:** What is the pivot transaction?
**A:** The go/no-go step; after it the saga only moves forward with retries.

**Q:** Where do irreversible side effects (emails) belong?
**A:** After the pivot.

**Q:** Backward vs forward recovery?
**A:** Backward: compensate completed steps. Forward: retry remaining steps until success.

**Q:** Choreography?
**A:** Services react to each other's events; no central coordinator.

**Q:** Orchestration?
**A:** A central orchestrator sends commands and tracks saga state as a state machine.

**Q:** Main drawback of choreography?
**A:** Workflow is implicit and scattered; cyclic dependencies; hard to see saga state.

**Q:** Main drawback of orchestration?
**A:** Another durable, highly available component; risk of logic centralising there.

**Q:** Name two workflow engines used as orchestrators.
**A:** Temporal (or Cadence), AWS Step Functions (also Camunda, Conductor).

**Q:** How do you atomically commit a step and publish its event?
**A:** Transactional outbox.

**Q:** TCC stands for?
**A:** Try (reserve), Confirm (make final), Cancel (release).

**Q:** Why does TCC isolate better than a plain saga?
**A:** Resources are reserved, not consumed; no final effect until Confirm.

**Q:** Empty cancel?
**A:** Cancel for a Try that never ran; must succeed as a no-op.

**Q:** Hanging try?
**A:** Try arriving after Cancel; must be rejected.

**Q:** Three saga isolation anomalies?
**A:** Lost updates, dirty reads, fuzzy (non-repeatable) reads.

**Q:** Semantic lock?
**A:** An application-level status flag (e.g. APPROVAL_PENDING) marking a record as part of an in-progress saga.

**Q:** Commutative updates countermeasure?
**A:** Use order-independent operations (debit/credit) so compensations don't overwrite others' changes.

**Q:** Pessimistic view countermeasure?
**A:** Reorder steps so risky updates happen in retriable steps, reducing dirty-read risk.

**Q:** Reread value countermeasure?
**A:** Re-check a record's version before updating; abort if changed (optimistic locking).

**Q:** Version file countermeasure?
**A:** Record operations on a record so out-of-order arrivals can be handled correctly.

**Q:** By-value countermeasure?
**A:** Choose saga vs stronger mechanism per request based on business risk.
