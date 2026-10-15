# 39 — Outbox & change data capture

> Part 7 — Moving Data Between Systems · Prerequisites: topics 17, 21, 23, 34, 36, 37, 38

A typical service owns a database and also needs to tell the rest of the world what changed: publish an
`OrderPlaced` event, invalidate a cache entry, update a search index, feed an analytics warehouse. The
naive approach — write to the database, then publish a message — looks fine in code review and is
subtly broken: the two writes go to two different systems, and any crash or failure between them leaves
the systems disagreeing *permanently*. This is the **dual-write problem**. This topic explains why it
happens, then two families of solutions that both work by turning two writes into one: the
**transactional outbox** pattern (write the event into the same database transaction) and **change data
capture (CDC)** (read the database's own replication log and turn it into a stream of changes). Finally,
it shows how these techniques keep caches, search indexes and other derived data in sync.

By the end of this topic you should be able to:
1. Explain the dual-write problem, including both the crash and the race-condition variants.
2. Explain why 2PC across a database and a broker is rarely used to solve it.
3. Describe the transactional outbox pattern: table design, relay (polling or log-tailing), ordering, cleanup, and its at-least-once guarantee.
4. Explain how CDC works by reading the WAL/binlog (Postgres logical replication, MySQL binlog, Debezium), including snapshots and offsets.
5. Compare outbox vs raw CDC (domain events vs row changes) and when to combine them.
6. Keep caches, search indexes and warehouses in sync with a change stream, and reason about ordering, deletes, lag and rebuilds.

---

## 1. The dual-write problem

### 1.1 The naive code

```python
def place_order(order):
    db.execute("INSERT INTO orders ...")       # write 1: database
    db.commit()
    broker.publish("orders", OrderPlaced(order))  # write 2: message broker
```

Two writes to two systems with no shared transaction. Consider the failure points.

### 1.2 Variant 1: crash or failure between writes

```
t1  INSERT + COMMIT order 991        ✔ in DB
t2  process crashes / broker unreachable / publish times out
t3  OrderPlaced never published      ✘

Result: the order exists, but billing, email and inventory never hear about it.
```

Reverse the order (publish first, then commit) and you get the mirror image:

```
t1  publish OrderPlaced(991)         ✔ consumers start shipping
t2  DB commit fails (constraint violation, deadlock, crash)
Result: consumers act on an order that doesn't exist.
```

And the ambiguous timeout (topic 38) makes it worse: a publish that times out may or may not have
succeeded. Retrying in-process helps with transient errors but not with a crash — the retry state dies
with the process.

### 1.3 Variant 2: the race (even with no failures)

Two concurrent requests update the same product price, each writing the DB and then the search index:

```
request A: DB  price=10 ─────────────────────────────► index price=10
request B:      DB price=12 ─► index price=12
                                                  (A's index write arrives last)

DB says 12 (B committed last)       Index says 10 (A wrote last)   ✘ permanently inconsistent
```

Nothing crashed. Each request did both writes successfully. But the two systems applied the writes in
**different orders**, so they diverge — and since nothing failed, nothing will retry or alert. This is
why "just add retries" doesn't fix dual writes: the problem is the lack of a single ordering, not just
lost writes.

### 1.4 Why not a distributed transaction?

Two-phase commit (topic 34) could in principle make the DB commit and the broker publish atomic (XA
transactions). In practice it's rarely used here:
- Many brokers (Kafka, SQS) don't participate in XA.
- 2PC couples availability: the DB transaction blocks if the broker is slow or down, and an in-doubt
  coordinator failure holds locks.
- It adds latency to every write.

The solutions below avoid distributed transactions by making the **database the only thing written
synchronously**, and deriving everything else from what the database committed.

### 1.5 The principle

> Write to exactly one system of record, atomically. Derive everything else from that system's committed
> history, asynchronously and in commit order.

Both the outbox and CDC are applications of this principle. The log of committed changes provides the
single order that fixes the race; the asynchronous, retryable derivation fixes the crash.

---

## 2. Transactional outbox

### 2.1 The idea

Instead of publishing directly, the service writes the event into an **outbox table in the same database,
in the same transaction** as the business change. A separate **relay** process reads the outbox and
publishes to the broker.

```
             ┌─────────────── one DB transaction ───────────────┐
service ──►  │ INSERT INTO orders (...)                          │
             │ INSERT INTO outbox (id, aggregate_id, type, ...)  │
             └───────────────────────────────────────────────────┘
                                │ (committed together, or not at all)
                                ▼
                        relay process ──► broker topic "orders" ──► consumers
                     (polls or tails the log;
                      marks/deletes sent rows)
```

Because both inserts are in one ACID transaction (topic 21), it's impossible to have the order without
the event or the event without the order. The publish becomes a *retryable, asynchronous* step that
can't be lost: if the relay crashes, the rows are still in the table.

### 2.2 Outbox table design

```sql
CREATE TABLE outbox (
  id             uuid PRIMARY KEY,          -- event ID; consumers dedupe on it (topic 38)
  aggregate_type text NOT NULL,             -- 'order'
  aggregate_id   text NOT NULL,             -- '991' → used as the message key (ordering)
  event_type     text NOT NULL,             -- 'OrderPlaced'
  payload        jsonb NOT NULL,            -- the event body (a stable, versioned contract)
  created_at     timestamptz NOT NULL DEFAULT now(),
  published_at   timestamptz                -- NULL until relayed (polling variant)
);
```

Design notes:
- The **payload is a domain event** written deliberately by application code — a public contract,
  versioned like any API (topic 16), not a dump of internal columns.
- **`aggregate_id` becomes the partition key** in Kafka, so all events for one order stay in order
  (topic 37).
- The **event ID** is generated at creation and travels with the message, so consumers can deduplicate.

### 2.3 Relay option A: polling publisher

```
loop:
  rows = SELECT * FROM outbox WHERE published_at IS NULL
         ORDER BY created_at LIMIT 500 FOR UPDATE SKIP LOCKED;
  for row in rows: broker.publish(key=row.aggregate_id, value=row.payload, id=row.id)
  wait for broker acks
  UPDATE outbox SET published_at = now() WHERE id IN (...);   -- or DELETE
  COMMIT
  sleep(short interval)
```

- Simple; works with any database.
- Adds query load and latency proportional to the poll interval.
- **Ordering pitfall:** ordering by `created_at` or a sequence isn't the same as *commit* order.
  Transaction T1 may get `id=100` and commit after T2 got `id=101`; a poller that read 101 and
  advanced a high-water mark past 100 would skip it. Polling by "unpublished" flag (not by "id > last")
  avoids skipping; per-aggregate ordering is preserved as long as a single aggregate's events aren't
  written concurrently (row locks on the aggregate usually ensure that).
- Multiple relay instances need coordination (`SKIP LOCKED`, or a single leader via a lease, topic 31)
  without breaking per-key order.

### 2.4 Relay option B: log-tailing (outbox + CDC)

Instead of polling the table, tail the database's replication log (section 3) and pick out inserts into
the outbox table. Debezium has a dedicated "outbox event router" for exactly this. Benefits: commit
order, low latency, no polling load. The outbox rows can even be deleted immediately in the same
transaction (insert then delete) — the insert is still in the log, which is all the relay reads.

### 2.5 Guarantees and their consequences

**At-least-once.** The relay publishes, then marks the row. A crash in between republishes the row on
restart. Consumers **must** dedupe by event ID (topic 38). There's no avoiding this — it's the same ack
problem one level down.

**Eventually consistent, with lag.** Consumers see the event shortly after commit — milliseconds to
seconds. Not inside the user's request.

**Ordering** is per aggregate if the relay preserves it and the key is the aggregate ID.

**Cleanup.** Delete or archive published rows (by age or immediately in the log-tailing variant);
otherwise the outbox table grows forever and polling slows.

**Inbox on the other side.** The receiving service often pairs it with an inbox/dedup table (topic 38)
so the full chain is: DB transaction (business + outbox) → relay (at-least-once) → consumer transaction
(inbox + effect). Each hop is atomic locally; the chain is effectively-once end to end.

---

## 3. Change data capture (CDC)

### 3.1 The idea

Every serious database already keeps an ordered, durable log of every committed change — the
write-ahead log (topic 17) or binlog — and uses it for replication (topic 23). **CDC reads that log and
publishes each committed row change as an event.** The application doesn't publish anything; it just
writes to its database normally.

```
app ──► Postgres ──WAL──► logical decoding ──► CDC connector (Debezium) ──► Kafka topic per table
                                                                             "db.public.orders"
                                                                                │
                                         ┌──────────────────┬──────────────────┼──────────────────┐
                                         ▼                  ▼                  ▼                  ▼
                                   search indexer     cache invalidator   warehouse loader   other service
```

### 3.2 Where the changes come from

- **Postgres: logical replication / logical decoding.** The physical WAL records page changes; *logical
  decoding* turns it into row-level change events (insert/update/delete with column values) via an
  output plugin (`pgoutput`, built in). A **replication slot** tracks how far the consumer has read
  (its LSN, log sequence number) and makes Postgres **retain WAL until the slot confirms it** — so a
  stuck CDC connector makes WAL pile up and can fill the disk. Monitor slot lag. Updates/deletes carry
  the old values only per `REPLICA IDENTITY` (by default just the primary key).
- **MySQL: the binlog** in row-based format (`binlog_format=ROW`), with the reader acting like a replica.
  Positions are binlog file + offset or GTIDs.
- **MongoDB: change streams** built on the oplog.
- **DynamoDB Streams**, **Cassandra CDC**, and others expose similar feeds.

**Debezium** is the best-known open-source CDC platform: connectors for Postgres, MySQL, MongoDB, SQL
Server, Oracle and more, usually running on Kafka Connect, emitting a change event per row change into
a Kafka topic (one per table by default).

### 3.3 Anatomy of a change event

```json
{
  "op": "u",                                   // c=create, u=update, d=delete, r=snapshot read
  "before": { "id": 42, "price": 10.00, "name": "Mug" },
  "after":  { "id": 42, "price": 12.00, "name": "Mug" },
  "source": { "db": "shop", "table": "products", "lsn": 2398123344, "txId": 77812, "ts_ms": 1760500000000 },
  "ts_ms": 1760500000105
}
```

- The Kafka **key is the primary key**, so all changes to a row are in one partition, in commit order.
- A delete is typically emitted as a `d` event followed by a **tombstone** (null value) for compaction
  (topic 37), so a compacted CDC topic converges to "current table contents".
- The `lsn`/position can serve as a monotonic **version** for conditional upserts downstream (topic 38).

### 3.4 Initial snapshot + streaming

The log doesn't go back forever (WAL is recycled). To bootstrap, a CDC connector:

1. Notes the current log position.
2. Takes a **consistent snapshot** of existing rows (emitted as `op: r` events).
3. Streams changes from the noted position onward.

Rows changed during the snapshot appear in both, which is fine because consumers apply changes
idempotently (later events win by version). Modern connectors support *incremental snapshots* that
interleave chunks of the snapshot with streaming, so a huge table doesn't block live changes (based on
the watermark-based approach from Netflix's DBLog).

### 3.5 Offsets and delivery

The connector periodically records its log position (in Kafka Connect's offsets topic). After a crash it
resumes from the last recorded position, re-emitting some changes: **at-least-once**, again. Consumers
dedupe or apply idempotently.

### 3.6 Why CDC solves the race

Because events come from the log in **commit order**, the "price=10 vs price=12" race of section 1.3
can't happen: whichever transaction committed last is also last in the log, and every downstream system
applies the changes in that order. The database's own serialisation of transactions becomes the order
for everyone.

---

## 4. Outbox vs CDC: what you're publishing

Both use the "single write + log" principle. The difference is *what* the events represent.

| | Raw CDC (row changes) | Outbox (domain events) |
|---|---|---|
| Event shape | Table rows: before/after column values | Business facts: `OrderPlaced { … }` |
| Who decides content | The schema | Application code |
| Code changes needed | None — works on legacy apps | Every write path must insert outbox rows |
| Coupling | Consumers depend on **internal table schema**; a column rename breaks them | Consumers depend on an explicit, versioned contract |
| Intent | Lost: "status changed from 2 to 5" — why? | Explicit: `OrderCancelled { reason }` |
| Multi-table changes | One event per row; consumers must reassemble | One event describing the whole change |
| Best for | Replicating data into derived stores (search, cache, warehouse) owned by the same team; legacy systems | Integration between services/teams |

A common combination: **outbox written by the app, relayed by CDC** (Debezium outbox router). You get
domain events as a contract plus commit-order, low-latency delivery.

Rule of thumb: inside a team's own boundary, raw CDC to keep derived stores in sync is great. Across
team boundaries, publish deliberate events (outbox), because exposing your table schema makes it a
public API you can never change.

---

## 5. Keeping derived data in sync

### 5.1 Systems of record vs derived data

- **System of record (source of truth):** the authoritative copy — here, the service's database.
- **Derived data:** anything that can be recomputed from the source: caches (topic 11), search indexes
  (topic 51), materialised views, read models (topic 42), warehouse tables (topic 20), recommendations.

The CDC/outbox stream is the pipe from the former to the latter:

```
              ┌──► cache invalidator  ──► Redis      (DEL product:42)
Postgres ─CDC─┼──► search indexer     ──► Elasticsearch (upsert doc 42 if lsn newer)
              ├──► warehouse loader   ──► columnar warehouse (batched, topic 20/40)
              └──► stream processor   ──► aggregates, alerts (topic 41)
```

### 5.2 Caches

Cache invalidation via app code has the dual-write race of section 1.3 (topic 12 covers stale-set
races). With CDC:
- A consumer reads row changes and **deletes** (or updates) the corresponding cache key.
- Because invalidations follow commit order, a stale value can't "win" permanently the way it can with
  racing app writes — though a reader can still repopulate the cache from a lagging replica, so keep TTLs
  as a safety net.
- **Delete rather than set** when possible: deleting is idempotent and order-insensitive; the next read
  fetches fresh data.

### 5.3 Search indexes

- Key documents by the primary key; apply `c`/`u` as upserts and `d` as deletes.
- Use the source position (LSN or row version) as an external version: Elasticsearch supports
  `version_type=external`, rejecting writes older than the stored version — idempotent and
  out-of-order-safe (topic 38).
- Denormalised documents (a product doc including its category name) require **joining** change
  streams — when a category is renamed, every product doc in it must be updated. Often done with a
  stream processor (topic 41) keeping the category table as state.

### 5.4 Warehouses and lakes

CDC feeds analytics stores continuously instead of nightly full dumps. Warehouses prefer large batched
writes, so loaders buffer changes and periodically **merge** (upsert) them into tables, or keep an
append-only change history and compute the current state with a view.

### 5.5 Lag, read-your-writes and rebuilds

- **Lag:** derived stores trail the source. A user who updates their profile and immediately searches may
  see the old version. Mitigations echo topic 23's replication-lag fixes: read critical data from the
  source, or have the UI apply the user's own change locally.
- **Rebuild:** because derived data is derivable, you can delete a broken index and **rebuild** it by
  re-snapshotting the source or replaying a compacted CDC topic from the beginning — the replay safety
  rules from topic 38 apply. Building a new index alongside the old one and then switching reads over
  avoids downtime.
- **Schema changes** in the source table flow through to CDC consumers; use a schema registry and
  compatible changes (topic 16), and expand/contract migrations (topic 47).

### 5.6 Failure modes

| Failure | Effect | Mitigation |
|---|---|---|
| CDC connector down | Derived stores stale; Postgres retains WAL for the slot → disk grows | Alert on slot lag; cap retained WAL (`max_slot_wal_keep_size`) |
| Relay down (polling outbox) | Events delayed, outbox grows | Alert on oldest unpublished row age |
| Duplicate events after restart | Double effects in non-idempotent consumers | Dedupe by event ID / versioned upserts |
| Source schema change | Consumer deserialisation failures (poison, topic 38) | Schema registry, compatibility rules |
| Large transactions | Burst of thousands of events; lag spike | Batch-size limits; chunk backfills |
| Failover to a new primary | Logical replication slot may not exist on the new primary (depending on version/setup) → gap or re-snapshot | Failover slots (Postgres 17+ can sync slots to standbys), or re-snapshot |

---

## 6. Listen-to-yourself and other variations

- **Listen to yourself:** the service doesn't do the follow-up work in the request; it consumes its own
  events (via outbox/CDC) to trigger it. Same principle: one atomic write, everything else derived.
- **Database triggers** writing to an audit/outbox table: same as an outbox, implemented in the DB.
  Works, but logic hidden in triggers is harder to manage.
- **Event sourcing** (topic 42) goes the furthest: the event log *is* the system of record, so there's no
  separate table to keep in sync with it.

---

## 7. How it connects

- **Topic 12 (cache failure modes):** CDC-driven invalidation fixes the stale-write race of app-driven
  invalidation.
- **Topic 16 (schema evolution):** outbox payloads and CDC schemas are contracts.
- **Topic 17 (WAL):** CDC reads the very log the database writes for durability.
- **Topic 21 (transactions):** the outbox relies on one local ACID transaction.
- **Topic 23 (replication):** CDC is logical replication to non-database consumers; same lag issues.
- **Topic 34 (2PC):** the avoided alternative.
- **Topic 35 (sagas):** saga steps publish their events via an outbox so state change and message are atomic.
- **Topic 37 (Kafka):** CDC topics keyed by primary key, compacted, ordered per partition.
- **Topic 38 (delivery semantics):** relays are at-least-once; consumers dedupe and version-check.
- **Topic 41 (stream processing):** joining and aggregating change streams into derived views.
- **Topic 42 (event sourcing/CQRS):** read models kept in sync from an event stream.

---

## Common misconceptions

- **"Write DB, then publish, with retries, is good enough."** → A crash loses the publish, and even
  without failures, concurrent requests can apply writes in different orders to different systems.
- **"Publishing inside the DB transaction (before COMMIT) fixes it."** → The broker isn't part of the
  transaction; the commit can still fail after the message is out.
- **"The outbox gives exactly-once delivery."** → It gives *no loss* (at-least-once). Consumers still
  dedupe.
- **"CDC events are a fine public API."** → They expose your table schema; renaming a column breaks
  other teams. Use domain events for integration.
- **"Ordering by outbox ID equals commit order."** → IDs are assigned before commit; transactions commit
  in a different order.
- **"A CDC connector being down is harmless; it'll catch up."** → On Postgres, the replication slot forces
  WAL retention; a long outage can fill the primary's disk.
- **"Derived data must be backed up like the source."** → It can be rebuilt from the source or the log —
  that's the point (though rebuild time may still justify backups).

## Further reading (supplements)

- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 11 ("Databases and Streams":
  keeping systems in sync, change data capture) and Chapter 12 ("Data Integration").
- Chris Richardson, microservices.io: "Pattern: Transactional outbox" and "Pattern: Transaction log
  tailing".
- Debezium documentation: architecture, Postgres connector, and the "Outbox Event Router".
- Martin Kleppmann, "Turning the database inside-out with Apache Samza" (talk, Strange Loop 2014).
- Andreakis & Papapanagiotou, "DBLog: A Watermark Based Change-Data-Capture Framework" (Netflix, 2020).
- PostgreSQL documentation: "Logical Decoding" and "Replication Slots".

---

## Questions

1. Describe two different failure timelines for "commit to DB, then publish to Kafka", one where the
   event is lost and one where consumers act on something that never happened.
2. Two requests update the same row and each then updates a search index directly. Nothing fails. How can
   the index end up permanently wrong?
3. Why not use two-phase commit between the database and the broker?
4. Explain why the outbox pattern can't lose events. Why does it still produce duplicates?
5. A polling relay selects `WHERE id > last_published_id ORDER BY id`. Describe how it can skip an event.
6. Why should the outbox row's aggregate ID be the Kafka message key?
7. What does a Postgres replication slot do, and what happens to the primary if the CDC connector is down
   for a day?
8. How does a CDC connector bootstrap a table whose WAL history is long gone? What about rows changed
   during the snapshot?
9. Another team asks to consume your raw CDC topic for the `orders` table. What's the risk, and what do
   you offer instead?
10. Explain how using the source LSN as an external version in Elasticsearch handles duplicates and
    reordering.
11. Why is deleting a cache key on change usually better than setting the new value?
12. A product search doc includes the category name. The category is renamed. What must the sync pipeline
    do, and what tool helps?
13. The search index is corrupted. Describe how to rebuild it without downtime.
14. A user edits their profile and immediately sees the old version in search results. Why, and what are
    the options?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Lost: DB commit succeeds, then the process crashes (or the broker is unreachable) before
publishing — the order exists, no event. Phantom: if you publish first and then the DB commit fails
(constraint, deadlock, crash), consumers act on an order that doesn't exist. Also an ambiguous publish
timeout followed by a crash.

**2.** Request A writes DB (10), request B writes DB (12) and index (12), then A's index write (10)
arrives last. DB says 12, index says 10. Each system applied the writes in a different order; no error
occurred, so nothing retries or detects it.

**3.** Many brokers don't support XA; 2PC couples the database's availability and latency to the broker's;
coordinator failures leave in-doubt transactions holding locks. The outbox gets atomicity with a single
local transaction instead.

**4.** The event row is committed in the same ACID transaction as the business change, so it exists iff
the change exists, and stays until relayed. The relay publishes then marks it sent; a crash between
the two republishes it — at-least-once.

**5.** IDs are allocated before commit. T1 gets id 100, T2 gets 101; T2 commits first. The poller sees
101, publishes it, sets last=101. T1 then commits row 100, which is never selected because 100 < 101.
Fix: poll by an unpublished flag, or use log tailing (commit order).

**6.** So all events for one aggregate go to the same partition and are consumed in order (topic 37);
otherwise `OrderShipped` could be processed before `OrderPlaced`.

**7.** It records how far the logical consumer has read (confirmed LSN) and makes Postgres keep all WAL
after that point. If the connector is down for a day, WAL accumulates on the primary's disk and can fill
it (unless `max_slot_wal_keep_size` caps it, in which case the slot is invalidated and the connector
must re-snapshot).

**8.** Record the current log position, take a consistent snapshot of all rows (emitted as read events),
then stream from the recorded position. Rows changed during the snapshot appear twice; consumers apply
idempotently with versions so the later change wins. Incremental snapshots interleave chunks with live
streaming.

**9.** Their code would depend on your internal table schema — renaming/splitting a column breaks them,
so your schema becomes a frozen public API, and row changes lack business intent. Offer domain events
via an outbox (versioned contract), possibly relayed by CDC.

**10.** Each write carries the LSN as its version; Elasticsearch rejects any write whose version isn't
greater than the stored one. A duplicate (same LSN) or an older out-of-order change is rejected, so the
doc always reflects the latest committed change.

**11.** Delete is idempotent and order-independent: duplicates or reordered deletes leave the same
result, and the next read loads the current source value. Setting values can let an older value
overwrite a newer one if events are applied out of order or raced by a reader.

**12.** It must find every product in that category and update their docs — a join between the category
change stream and the products. A stream processor (topic 41) holding the product→category mapping as
state, or a reindex job, does this.

**13.** Create a new index, populate it by re-snapshotting the source (or replaying the compacted CDC
topic from the start) while also applying live changes, wait until it catches up, then switch reads (an
alias) to the new index and drop the old one.

**14.** The search index is derived asynchronously and lags the source. Options: read the user's own
profile from the source DB (read-your-writes), have the client merge its own pending change into results,
or accept and minimise lag.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What is the dual-write problem?
**A:** Writing to two systems without a shared transaction; failures or races leave them inconsistent.

**Q:** Dual-write variant that needs no failure?
**A:** Concurrent requests apply writes to the two systems in different orders.

**Q:** Core principle behind outbox and CDC?
**A:** Write one system of record atomically; derive everything else from its committed log, in order.

**Q:** Why is 2PC rarely used for DB + broker?
**A:** Brokers lack XA support; it couples availability/latency; in-doubt locks.

**Q:** Transactional outbox in one line?
**A:** Insert the event into an outbox table in the same transaction as the change; a relay publishes it.

**Q:** Outbox delivery guarantee?
**A:** At-least-once — no loss, possible duplicates.

**Q:** What should be the message key for outbox events?
**A:** The aggregate ID, for per-aggregate ordering.

**Q:** Polling relay ordering trap?
**A:** IDs are assigned before commit; "id > last" can skip late-committing rows.

**Q:** Two relay options?
**A:** Polling publisher, or log tailing (CDC on the outbox table).

**Q:** What is CDC?
**A:** Reading the database's replication log and emitting each committed row change as an event.

**Q:** Postgres mechanism for CDC?
**A:** Logical decoding (pgoutput) via a replication slot.

**Q:** Risk of a stuck replication slot?
**A:** WAL is retained until confirmed, filling the primary's disk.

**Q:** MySQL source for CDC?
**A:** Row-based binlog.

**Q:** What is Debezium?
**A:** Open-source CDC platform on Kafka Connect emitting per-row change events.

**Q:** Change event contents?
**A:** op, before, after, source position (LSN/txId), timestamp; keyed by primary key.

**Q:** How does CDC bootstrap?
**A:** Note log position, consistent snapshot, then stream from that position.

**Q:** Why does CDC fix the race?
**A:** Events are in commit order, so every consumer applies changes in the same order.

**Q:** Raw CDC vs outbox events?
**A:** CDC = row changes, internal schema, no code changes; outbox = deliberate domain events, a contract.

**Q:** Which to use across team boundaries?
**A:** Outbox/domain events — don't expose internal tables.

**Q:** System of record vs derived data?
**A:** Source of truth vs data recomputable from it (caches, indexes, views).

**Q:** Why prefer cache delete over set?
**A:** Idempotent and order-insensitive.

**Q:** How do you make search updates out-of-order-safe?
**A:** External versioning with the source LSN/row version.

**Q:** How to rebuild a derived index without downtime?
**A:** Build a new one from snapshot/replay plus live changes, then switch an alias.

**Q:** Why can users see stale data after CDC sync?
**A:** Derived stores lag; use read-your-writes from the source when needed.
