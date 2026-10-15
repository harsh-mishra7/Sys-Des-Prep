# 42 — Event sourcing & CQRS

> Part 7 — Moving Data Between Systems · Prerequisites: topics 21, 22, 37, 38, 39, 41

A conventional application stores **current state**: the `accounts` row says `balance = 120`. When the
balance changes, the old value is overwritten and gone (unless someone thought to add an audit table).
**Event sourcing** inverts this: the system of record is an append-only log of **events** —
`AccountOpened`, `MoneyDeposited(100)`, `MoneyDeposited(50)`, `MoneyWithdrawn(30)` — and current state
is *derived* by replaying them. It's the stream–table duality of topic 41 applied to an application's
own core data. **CQRS** (Command Query Responsibility Segregation) is the closely related idea of using
*different models* for writing and for reading: a write model that validates commands and records
changes, and one or more read models shaped for queries and kept up to date from the changes. The two
are often used together but are independent. This topic explains how each works, what they make easy
(audit, temporal queries, new views from old data), what they make hard (versioning an immutable
history, eventual consistency, deletion), and when they're worth it.

By the end of this topic you should be able to:
1. Explain how an event-sourced system stores events as the source of truth and rebuilds state from them.
2. Describe the command → validate → event → apply cycle, and how concurrency is controlled with expected versions.
3. Explain snapshots: why, when, and how they interact with the log.
4. Explain event versioning strategies (upcasting, weak schema, new event types, copy-and-transform) and the real costs of an immutable history, including deletion/GDPR.
5. Explain CQRS: separate write and read models, projections, and the eventual consistency between them — and how to cope with it in a UI.
6. Judge when event sourcing and/or CQRS are a good fit and when they're needless complexity.

---

## 1. Events as the source of truth

### 1.1 State-oriented vs event-oriented storage

```
State-oriented (CRUD)                          Event-sourced
accounts                                       event store (stream "account-7")
┌────┬─────────┬────────┐                      v1  AccountOpened      { owner: "ana" }
│ id │ owner   │ balance│                      v2  MoneyDeposited     { amount: 100 }
│ 7  │ ana     │ 120    │  ← only "now"        v3  MoneyDeposited     { amount: 50 }
└────┴─────────┴────────┘                      v4  MoneyWithdrawn     { amount: 30 }
                                               ─────────────────────────────────────
                                               current state = fold(events) → balance 120
```

In the event-sourced version:
- Events are **facts in the past tense** describing what happened in domain terms: `OrderPlaced`,
  `ItemAddedToCart`, `AddressChanged` — not `UPDATE orders SET status=3`.
- Events are **immutable** and **append-only**. You never update or delete an event; corrections are new
  events (`MoneyDepositReversed`).
- **Current state is a derived, disposable view.** It can be rebuilt at any time by replaying the events.

This is how accountants have worked for centuries: a ledger of transactions, with the balance computed
from it; mistakes are fixed by adding a correcting entry, never by erasing.

### 1.2 Event sourcing vs CDC and vs "events for integration"

These get confused, so separate them:

| | What's recorded | Level | Source of truth |
|---|---|---|---|
| CDC (topic 39) | Row-level changes extracted from a mutable DB | Low (columns) | The mutable tables |
| Outbox / integration events (topic 39) | Domain events published *in addition to* state | Domain | The mutable tables |
| Event sourcing | Domain events *instead of* state | Domain | The event log itself |

With event sourcing, the events aren't a side channel; they *are* the database. (You may still publish a
separate, stable set of integration events to other teams rather than exposing internal events — same
contract argument as topic 39.)

### 1.3 Aggregates and streams

Events are grouped into **streams**, usually one per **aggregate** — a cluster of data treated as a unit
for consistency (a term from Domain-Driven Design): one account, one order, one shopping cart. Each
stream has its own sequence of versions `1, 2, 3, …`.

The aggregate is the **consistency boundary**: business rules ("balance may not go negative") are
enforced within one aggregate's stream. Rules spanning aggregates need other tools (sagas, topic 35, or
reservation patterns).

### 1.4 The command → event cycle

Write requests arrive as **commands** — imperative requests that may be rejected: `WithdrawMoney(acct 7,
$200)`.

```
1. Load:     read all events of stream "account-7" (v1..v4), fold into state  → balance 120, version 4
2. Decide:   command handler checks rules: 200 > 120 → reject (InsufficientFunds)
             or for $30: valid → produce event MoneyWithdrawn{30}
3. Append:   append MoneyWithdrawn as v5, *only if the stream's current version is still 4*
4. Publish:  event becomes visible to subscribers (projections, other services)
```

Key distinction: a **command** can fail validation; an **event** has already happened and cannot be
rejected by those who read it. The handler's decision logic is a pure function
`decide(state, command) → events`, and state evolution is `apply(state, event) → state` — two small,
very testable functions. "Given these past events, when this command, then expect these new events" is
the natural test format.

### 1.5 Concurrency: optimistic, with expected versions

Two requests withdraw from the same account concurrently:

```
request A: load → v4, balance 120 → decide Withdraw 100 → append v5 expecting version 4 ✔
request B: load → v4, balance 120 → decide Withdraw 100 → append v5 expecting version 4 ✘ conflict
           → reload (v5, balance 20) → re-decide → reject: insufficient funds
```

The event store's append is a **compare-and-set on the stream version** — optimistic concurrency control
(topic 22). Without it, both withdrawals would be appended and the balance would go to −80: a lost
update / write skew (topic 21). Event stores implement this as "append with expected version"
(EventStoreDB), a conditional write (DynamoDB condition expression on `version`), or a unique constraint
on `(stream_id, version)` in a relational table.

### 1.6 Where events are stored

- **Dedicated event stores:** EventStoreDB (KurrentDB), Axon Server — streams, expected-version appends,
  subscriptions.
- **A relational table:** `events(stream_id, version, type, data jsonb, metadata, recorded_at,
  global_position)` with `PRIMARY KEY (stream_id, version)` — simple and transactional; Marten does this
  on Postgres. A global position column lets projections read "all events in order".
- **Kafka (topic 37):** excellent for *distributing* events, but awkward as the primary store for
  aggregates: it has no per-key "append only if the stream's version is N" (optimistic concurrency per
  aggregate), and loading one aggregate's events means scanning a partition. Many systems store events in
  a database and publish them to Kafka (via an outbox/CDC, topic 39).

---

## 2. Rebuilding state from the log

### 2.1 Replay

State is computed by folding events:

```python
def apply(state, event):
    match event:
        case AccountOpened(owner):     return Account(owner=owner, balance=0)
        case MoneyDeposited(amount):   return state.with_balance(state.balance + amount)
        case MoneyWithdrawn(amount):   return state.with_balance(state.balance - amount)

state = reduce(apply, load_events("account-7"), None)
```

Because `apply` must reproduce the same state every time, it must be **deterministic** and free of side
effects: no reading the clock (use the event's timestamp), no calling other services, no sending emails
while replaying. Side effects belong in reactions to *newly* appended events, guarded by idempotency
(topic 38).

### 2.2 What replay buys you

- **Complete audit trail.** Every change, why (event type), when, and by whom (event metadata). Not a
  bolted-on audit table that can drift from reality — the log *is* the data.
- **Temporal queries.** "What was this account's state on 1 March?" — replay events up to that date.
  "What did the customer see when they complained?"
- **Debugging.** Reproduce a bug by replaying the exact sequence of events locally.
- **New views of old data.** A new feature needs "how often do users remove items from carts before
  checking out?" In a CRUD system that information was overwritten. With events (`ItemRemovedFromCart`),
  you build a new projection and replay all history into it.
- **Fixing derived state.** A bug in a projection? Fix the code, rebuild from events (the replay-safety
  rules of topic 38 apply).

### 2.3 The cost of replay

Loading an aggregate by replaying all of its events is fine for 50 events and slow for 50,000 (a
long-lived account, a busy inventory item). And rebuilding a projection over the *entire* event store may
take hours. Snapshots address the first; careful projection design and parallelism address the second.

---

## 3. Snapshots

### 3.1 The mechanism

A **snapshot** is a saved copy of an aggregate's state as of some version:

```
events:     v1 v2 v3 ... v998 v999 v1000 v1001 v1002 v1003
snapshot:                              ▲
                          { balance: 5320, ... } @ v1000

load = read snapshot @1000, then apply only v1001..v1003
```

- Taken periodically: every N events (e.g. 100), or on a schedule, often asynchronously in the background.
- Stored separately from the events (a snapshots table/stream). The **events remain the source of truth**;
  a snapshot is a cache and can be deleted and regenerated at any time.

### 3.2 Pitfalls

- **Snapshot schema coupling.** A snapshot is a serialised *state object*. When the aggregate's code
  changes shape, old snapshots may not deserialise. The usual rule: version snapshots with the code, and
  when incompatible, **discard them** and rebuild from events. Never let snapshots become the only copy.
- **Don't snapshot prematurely.** If aggregates rarely exceed a few hundred events, replay is cheap. Long
  streams can also indicate a modelling problem — an aggregate that lives forever (e.g. "the warehouse")
  might be better split into time-bounded streams ("closing the books": an `AccountingPeriodClosed`
  event carries the opening balance into the next period's stream).
- Projections (read models, section 5) are a kind of snapshot too: they're persisted derived state that
  can be rebuilt.

---

## 4. Event versioning and the cost of an immutable history

### 4.1 The problem

Events are stored forever, but the code that reads them evolves. In two years, `OrderPlaced` v1 events
(with a single `address` string) sit in the store alongside v3 events (with structured `shipping_address`
and `billing_address`). Every replay — of aggregates and of projections — must still understand all of
them. Unlike a CRUD schema migration, you can't (in principle) rewrite history.

This is schema evolution (topic 16), but with **unbounded backward compatibility**: the reader must handle
every version ever written.

### 4.2 Strategies

1. **Additive, tolerant changes (weak schema).** Add optional fields with defaults; never remove or
   repurpose fields; readers ignore unknown fields. With JSON or Avro/Protobuf following compatibility
   rules (topic 16), many changes need nothing more.
2. **Upcasting.** On read, transform old event versions into the current shape before they reach
   handlers:

   ```
   stored: OrderPlaced_v1 { address: "1 Main St, Springfield" }
       └─ upcaster v1→v2: { shipping_address: parse(address), billing_address: same }
   handler sees only OrderPlaced_v2
   ```

   Upcasters chain (v1→v2→v3), keep handler code clean, and leave stored data untouched. The cost is a
   growing pile of upcasters that must be maintained and tested forever.
3. **New event type.** If the meaning changes, introduce a new event (`OrderPlacedWithSplitAddresses`) and
   handle both. Clear, but handlers accumulate cases.
4. **Copy-and-transform migration.** As a last resort, migrate the store: read all events, transform them,
   write a *new* event store, switch over. This rewrites history (breaks the "immutable" ideal) and is a
   big operation, but sometimes beats carrying many upcasters forever. In-place rewriting of events is
   generally avoided because other consumers may already have read the originals.
5. **Version in metadata.** Always store an explicit schema version (or type name with version) with each
   event, so readers can dispatch correctly.

**Design events for longevity:** name them after business facts, include what consumers need, avoid
leaking internal class structure (serialising a language object directly ties the store to the code).

### 4.3 Other costs of immutability

- **Deleting data (GDPR "right to erasure").** An append-only log containing personal data conflicts with
  a legal obligation to delete it. Approaches:
  - **Crypto-shredding:** encrypt each person's personal data in events with a per-person key stored
    elsewhere; to "forget" them, delete the key. The events remain but the PII is unreadable.
  - **Keep PII out of events:** store it in a separate, mutable store referenced by ID.
  - **Truncate or rewrite** the affected streams (if the store supports it), accepting the loss of
    immutability for those streams.
- **Mistakes are permanent.** A buggy handler that emitted wrong events can't just be "fixed in the DB";
  you append compensating events and make projections handle them.
- **Storage grows forever.** Usually cheap (events are small; tiered storage, topic 37), but it's
  unbounded and backups/replays grow with it.
- **Queries on current state are indirect.** "Find all accounts with balance > 1000" can't be answered
  from the event store efficiently — you need a projection. That leads to CQRS.
- **Learning curve and tooling.** Fewer developers know the patterns; ORMs, admin tools and ad-hoc SQL
  "just look at the table" debugging don't apply.

---

## 5. CQRS: separate write and read models

### 5.1 The idea

**Command Query Responsibility Segregation:** use one model to handle **commands** (writes) and a
different model, or several, to handle **queries** (reads).

Why? In many systems the write side and read side want very different things:

| Write side | Read side |
|---|---|
| Enforce invariants on one aggregate at a time | Show data from many aggregates at once |
| Normalised, consistent, transactional | Denormalised, precomputed, fast |
| Low volume relative to reads | Often 10–1000× more traffic |
| Shaped by business rules | Shaped by screens, APIs, search |

A single model serving both ends up compromised: complex joins for every page view, or denormalised
tables that make writes painful.

```
                commands                                         queries
user ──► command handler ──► write model ──► changes ──► projector ──► read model(s) ◄── user
         (validate, decide)  (event store                (consumes      ├ SQL view for order history
                              or normalised DB)           events)       ├ Elasticsearch for search
                                                                         ├ Redis for dashboard counters
                                                                         └ graph DB for recommendations
```

### 5.2 CQRS without event sourcing, and vice versa

- **CQRS without ES:** writes go to a normal normalised database; changes are propagated (via outbox or
  CDC, topic 39) into read-optimised stores. A search index fed by CDC is a lightweight form of CQRS.
  Even within one database, materialised views are a mild form.
- **ES without CQRS:** possible for simple cases where queries only ever look up a single aggregate by ID
  (load and replay). As soon as you need a list or search across aggregates, you need projections — so in
  practice ES nearly always comes with CQRS.
- CQRS is a **spectrum**, not a binary: from "separate query classes over the same tables" to "separate
  services with separate databases".

### 5.3 Projections

A **projection** (projector, read-model builder) subscribes to the event stream and maintains a read
model. It's a stream processor (topic 41) folding events into a table:

```
on OrderPlaced(o):      INSERT INTO order_summary (id, customer, total, status) VALUES (..., 'placed')
on OrderShipped(o):     UPDATE order_summary SET status='shipped' WHERE id = o.id
on OrderCancelled(o):   UPDATE order_summary SET status='cancelled' WHERE id = o.id
store checkpoint: last processed global position = 88,213
```

Projection design rules, all from earlier topics:
- **Track position** (the global position/offset processed), ideally in the same transaction as the read
  model update — exactly-once into the read store (topic 38).
- **Idempotent and version-aware** updates, because delivery is at-least-once.
- **Per-aggregate ordering** must be preserved (partition by aggregate ID, topic 37).
- **Rebuildable:** to change a read model's shape, create a new one, replay all events from position 0
  into it, then switch queries over (blue/green for read models). Make rebuilds fast enough (batching,
  parallelism by partition) that you're not afraid to do them.
- **No side effects during replay** (topic 38 section 6): projections that send emails must not resend on
  rebuild — separate "reactors" from "projectors".

### 5.4 Eventual consistency between the models

The read model is updated *after* the write commits, asynchronously. So there's a window during which a
query doesn't reflect a command that already succeeded:

```
t0  POST /orders → command accepted, OrderPlaced appended (v1)       → 201 Created
t1  browser redirects to GET /orders  (reads order_summary)
t1  projector hasn't processed OrderPlaced yet → order missing from list  ✘ "where's my order?"
t2  projector catches up → order appears
```

This is read-after-write inconsistency, the same problem as reading from a lagging replica (topic 23),
and the same family of fixes applies:

- **Return what the client needs from the command.** The command response includes the new order's
  data/ID, and the UI renders from that (optimistic UI).
- **Read-your-writes via version tokens.** The command returns the position/version it produced; the
  query says "give me a view at least as fresh as position 88,214", and the read side waits briefly (or
  falls back to the write model) until the projection has caught up.
- **Read from the write model** for the immediate confirmation page (load the aggregate by ID — always
  consistent), and from read models elsewhere.
- **Synchronous projection** for a critical read model: update it in the same transaction as the event
  append (possible when both live in one database). Gives strong consistency at the cost of coupling and
  write latency.
- **UX design:** "Your order is being processed" — make asynchrony visible rather than surprising.

Also note what CQRS **doesn't** allow: making decisions on the write side by querying a read model.
"Reject if a user with this email exists" checked against a lagging read model can let duplicates
through (two concurrent sign-ups both see "no such email"). Uniqueness across aggregates needs a
consistent mechanism: a unique constraint in a reservation table, a dedicated aggregate per email, or
accepting and compensating later (topic 35). This is one of the trickiest parts of applying CQRS.

### 5.5 Failure modes

| Failure | Effect | Mitigation |
|---|---|---|
| Projector crashes / is slow | Read model stale; lag grows | Monitor projection lag (position behind head) |
| Projection bug | Wrong read data | Fix, rebuild from events |
| Poison event in projector | Projection stuck at one position | DLQ vs block decision (topic 38) |
| Event schema change | Projector fails to deserialise old events | Upcasters, tolerant readers |
| Concurrent writers to one aggregate | Version conflicts → retries | Expected-version appends; smaller aggregates |
| Write-side validation via read model | Invariants violated under concurrency | Validate only within an aggregate or via consistent reservations |

---

## 6. When is it worth it?

### 6.1 Good fits

- **Domains where history *is* the business:** accounting and ledgers, banking, payments, insurance
  claims, logistics tracking, healthcare records — where auditability is mandatory and "how did we get
  here?" matters.
- **Complex domains with rich behaviour** where capturing intent (`OrderCancelledByCustomer` vs
  `OrderCancelledFraudSuspected`) has real value.
- **Many different read requirements** over the same writes (CQRS): search, analytics, dashboards, mobile
  APIs.
- **Collaborative/conflict-heavy domains** where knowing exactly which operations occurred aids merging.
- **Systems that need temporal queries or reproducible debugging.**

### 6.2 Poor fits

- **Simple CRUD** (admin panels, settings, content management) where events would just be `FieldUpdated`
  — all the costs, none of the benefits.
- **Teams new to the patterns** under time pressure; the learning curve is steep and mistakes in event
  design are permanent.
- **Applying it everywhere.** Event sourcing is usually best applied to a few core aggregates or bounded
  contexts, not an entire company's system.
- **When strong, immediate cross-entity consistency on reads is essential** and you can't design around
  eventual consistency.

### 6.3 A middle path

Many systems take the cheaper parts: a conventional database as the source of truth, an **outbox** of
domain events (topic 39) for integration and audit, and **CQRS read models** fed by those events or by CDC
for search and dashboards. You get decoupled read models and an event history for the parts that need it,
without making every write go through an event store.

---

## 7. How it connects

- **Topic 16 (schema evolution):** events must be readable forever; upcasting is schema evolution on read.
- **Topic 17/23 (WAL, replication):** a database is internally event-sourced — the WAL is the log, tables
  are the projection. Event sourcing lifts that pattern into the application.
- **Topic 21/22 (transactions, concurrency):** expected-version appends are optimistic concurrency control.
- **Topic 23 (replication lag):** read-your-writes problems reappear between write and read models.
- **Topic 35 (sagas):** cross-aggregate processes react to events and issue commands.
- **Topic 37 (Kafka):** distribution of events; compacted topics for read-model state.
- **Topic 38 (delivery semantics):** projections are idempotent, at-least-once consumers; replay safety.
- **Topic 39 (outbox/CDC):** how to publish events from the event store reliably; CQRS without ES via CDC.
- **Topic 41 (stream processing):** projections are stream processors; stream–table duality.
- **Topic 51 (search):** a search index is a typical CQRS read model.

---

## Common misconceptions

- **"Event sourcing means using Kafka as the database."** → Kafka is great for distributing events but
  lacks per-aggregate optimistic concurrency and efficient per-aggregate loading; most event-sourced
  systems store events in an event store or database and publish to Kafka.
- **"CQRS requires event sourcing (or vice versa)."** → They're independent. CQRS works with a normal DB
  plus CDC; ES needs some read models for anything beyond lookup by ID.
- **"Event sourcing gives you an audit log for free, so it's always worth it."** → It also brings
  versioning forever, eventual consistency, deletion difficulty and a learning curve. For simple CRUD an
  audit table is cheaper.
- **"Events are like CDC row changes."** → ES events are domain facts capturing intent, not column diffs.
- **"Snapshots replace the event log."** → Snapshots are a disposable cache; events remain the truth.
- **"Immutable means you can never fix anything."** → You fix forward with compensating events; history
  shows both the mistake and the correction (and rewriting is possible as a rare, deliberate migration).
- **"The read model can validate commands."** → It lags; invariants must be enforced in the write model.

## Further reading (supplements)

- Martin Fowler, "Event Sourcing" (2005) and "CQRS" (2011), martinfowler.com.
- Greg Young, "CQRS Documents" (2010), and *Versioning in an Event Sourced System* (Leanpub, 2017).
- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 11 ("Event Sourcing", "State,
  Streams, and Immutability") and Chapter 12 ("Unbundling Databases").
- Microsoft Azure Architecture Center: "Event Sourcing pattern" and "CQRS pattern".
- Eric Evans, *Domain-Driven Design* (2003) — aggregates and consistency boundaries.
- Martin Kleppmann, "Turning the database inside-out with Apache Samza" (talk, 2014).

---

## Questions

1. In an event-sourced bank account, where does the "current balance" live, and what happens if that
   copy is lost?
2. What is the difference between a command and an event? Why can a command be rejected but an event
   can't be?
3. Two concurrent withdrawals of $100 each hit an account with $120. Walk through how the event store
   prevents an overdraft.
4. Why must `apply(state, event)` be deterministic and side-effect free? Give an example of a bug if it
   isn't.
5. A new product manager asks how many users removed an item from their cart before checking out over the
   last year. Contrast how a CRUD system and an event-sourced system can answer.
6. An aggregate has 200,000 events and loads slowly. What are two remedies, and what's the main pitfall of
   the first?
7. `CustomerRegistered` v1 has `name`; v2 splits it into `first_name`/`last_name`. Describe two ways to
   handle stored v1 events.
8. A user invokes their right to be forgotten, and their email appears in thousands of events. What are
   the options?
9. Why would an event-sourced system almost always need CQRS?
10. After placing an order, a user is redirected to their order list and the order isn't there. Explain
    the cause and three possible fixes.
11. Sign-up checks a read model to ensure the email is unused. Under load, duplicate accounts appear. Why,
    and how should uniqueness be enforced?
12. A projection has a bug that miscomputed order totals for months. How do you fix the read model, and
    what properties must the projection have for this to be safe?
13. Is "CDC from Postgres into Elasticsearch for search" a form of CQRS? Is it event sourcing?
14. Give one domain where event sourcing is a strong fit and one where it's overkill, and explain why.

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** It's derived: computed by folding the account's events (possibly cached in a snapshot or read
model). If a derived copy is lost, rebuild it by replaying the events; only losing events loses data.

**2.** A command is a request to do something ("Withdraw $200"); it's validated against current state and
may be rejected. An event is a record that something *has happened* ("MoneyWithdrawn $30"); it's already a
fact in the log, so consumers can't reject it — only react, possibly with compensating actions.

**3.** Both load the stream at version 4 (balance 120) and decide to emit MoneyWithdrawn(100). Each appends
"expecting version 4". The first succeeds (v5); the second's expected version no longer matches, so the
append fails. It reloads (balance 20), re-runs the decision, and rejects with insufficient funds.

**4.** Because state is rebuilt by replaying events many times (loading aggregates, rebuilding
projections); each replay must produce identical state and must not repeat external effects. E.g. if
`apply` checks `now()` to decide whether a coupon has expired, replay months later yields different state;
if it sends an email, every replay re-sends it.

**5.** CRUD: the removals were overwritten (the cart row only shows current contents), so it can't be
answered for the past — start logging and wait a year. Event-sourced: `ItemRemovedFromCart` and
`CheckoutCompleted` events exist; write a new projection and replay history to compute it.

**6.** Snapshots (load from the latest snapshot plus subsequent events), or remodelling the aggregate into
time-bounded streams (e.g. closing periods with a carry-forward event). Snapshot pitfall: snapshots are
serialised state that can break when code changes; they must be versioned and disposable, rebuildable
from events.

**7.** Upcasting: on read, transform v1 into v2 (split the name heuristically) before handlers see it;
stored data unchanged. Or handle both versions in handlers (or tolerate both via a new event type). As a
last resort, copy-and-transform the whole store into a new one.

**8.** Crypto-shredding (encrypt PII with a per-user key, delete the key); keep PII outside events in a
mutable store referenced by ID (delete there); or truncate/rewrite the affected streams, sacrificing
immutability for them. Projections and downstream copies must also be purged.

**9.** The event store can efficiently load one aggregate by ID, but any query across aggregates (lists,
search, filters, totals) would require replaying everything. So you build read models (projections) from
the events — which is CQRS.

**10.** The read model (order list) is updated asynchronously by a projector and hadn't processed the new
event yet — eventual consistency between write and read models. Fixes: render the confirmation from the
command's response; use a version token and wait for the projection to reach it (read-your-writes); read
the just-created order from the write model; or update that projection synchronously in the same
transaction.

**11.** The read model lags and the check-then-act isn't atomic: two concurrent sign-ups both see "no such
email" and both succeed. Enforce uniqueness in the write side with a consistent mechanism: a unique
constraint/reservation table checked atomically, an aggregate keyed by email, or detect-and-compensate.

**12.** Fix the projection code, build a new read model by replaying all events from position 0, and switch
queries over when it's caught up. Requirements: projection is deterministic, idempotent/version-aware,
side-effect free (no emails on replay), and all event versions are still readable (upcasters).

**13.** It's a form of CQRS (separate, read-optimised model kept in sync from the write side's changes). It
isn't event sourcing: the source of truth is Postgres's mutable tables, and CDC events are row diffs, not
domain events that constitute the record.

**14.** Strong fit: a payments ledger — history and auditability are the business, corrections are
naturally compensating entries, many read views. Overkill: a user-preferences settings page — simple
CRUD, events would be "FieldUpdated", and the costs (versioning, eventual consistency) buy nothing.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Event sourcing in one line?
**A:** Store the sequence of domain events as the source of truth; derive state by replaying them.

**Q:** How are mistakes fixed in an event-sourced system?
**A:** By appending compensating/correcting events, not editing old ones.

**Q:** ES vs CDC?
**A:** ES events are domain facts and the source of truth; CDC events are row diffs from mutable tables.

**Q:** What is an aggregate?
**A:** A consistency boundary (one account, one order) with its own event stream and versions.

**Q:** Command vs event?
**A:** Command: a request that may be rejected. Event: a past fact that can't be.

**Q:** The two core functions of an event-sourced aggregate?
**A:** decide(state, command) → events; apply(state, event) → state.

**Q:** How is concurrency controlled on append?
**A:** Append with expected version — optimistic concurrency per stream.

**Q:** Why must apply be deterministic?
**A:** State is rebuilt by replay many times; it must give the same result with no side effects.

**Q:** Benefits of keeping all events?
**A:** Audit trail, temporal queries, debugging by replay, new views from old data.

**Q:** What is a snapshot?
**A:** Cached aggregate state at a version; load snapshot + later events.

**Q:** Main snapshot pitfall?
**A:** Schema coupling to code; must be versioned and disposable.

**Q:** What is upcasting?
**A:** Transforming old event versions into the current shape on read.

**Q:** Event versioning strategies?
**A:** Additive/tolerant changes, upcasting, new event types, copy-and-transform migration.

**Q:** GDPR in an immutable log?
**A:** Crypto-shredding, PII outside events, or rewriting affected streams.

**Q:** Why is Kafka awkward as the primary event store?
**A:** No per-aggregate expected-version append; loading one aggregate needs scanning.

**Q:** CQRS in one line?
**A:** Separate models for commands (writes) and queries (reads).

**Q:** Are CQRS and ES dependent?
**A:** No — independent, though ES almost always needs CQRS read models.

**Q:** What is a projection?
**A:** A consumer that folds events into a read model and tracks its position.

**Q:** How do you change a read model's shape?
**A:** Build a new one by replaying all events, then switch queries.

**Q:** Why can a user not see their just-placed order?
**A:** Read models update asynchronously — eventual consistency.

**Q:** Fixes for read-after-write in CQRS?
**A:** Use command response, version tokens, read write model, or synchronous projection.

**Q:** Why not validate commands against read models?
**A:** They lag; concurrent commands can both pass the check.

**Q:** Good fit for event sourcing?
**A:** Ledgers, payments, audit-heavy or behaviour-rich domains.

**Q:** Poor fit for event sourcing?
**A:** Simple CRUD, or applying it to everything.

**Q:** A database's internal analogue of event sourcing?
**A:** The WAL is the event log; tables are the projection.
