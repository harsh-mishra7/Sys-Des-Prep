# 16 — Encoding & Schema Evolution

> Part 4 — Storing Data · Prerequisites: topics 07, 15

Data outlives code. A row written to a database today will be read by code deployed next year; a message
put on a queue by version 41 of a service may be consumed by version 42 — or by version 40, still running
on a machine that hasn't been upgraded yet. Every time data leaves a process' memory (to disk, over the
network, into a queue), it has to be turned into bytes, and every time it comes back, those bytes have to
be turned back into objects by code that may not be the code that wrote them. This topic is about how that
translation works, the formats used to do it, and the discipline that lets the shape of data change
without breaking anyone.

By the end of this topic you should be able to:
1. Explain what encoding (serialization) is and why in-memory representations can't simply be written out.
2. Compare text formats (JSON, XML, CSV) with binary schema-based formats (Protocol Buffers, Thrift, Avro).
3. Define forward and backward compatibility precisely, and explain why rolling deploys need both.
4. State the rules for safely evolving a schema in Protobuf/Thrift and in Avro, and why they differ.
5. Describe what a schema registry does and how compatibility is enforced for message streams.
6. Apply evolution techniques to database schemas (expand/contract migrations) and to APIs (versioning).

---

## 1. Why encoding exists

### 1.1 Two representations of the same data

Inside a running program, data lives in objects, structs, lists and hash maps, connected by **pointers**.
Pointers are memory addresses — meaningful only inside that one process at that one moment. To write data
to a file or send it over a network, you need a **self-contained sequence of bytes**.

```
In memory (process A)                         On the wire / on disk
+----------------------+
| User                 |                       7b 22 69 64 22 3a 31 2c 22 6e 61 6d 65
|  id: 1               |  --- encode --->      22 3a 22 41 73 68 61 22 2c ...
|  name: ptr --------> "Asha"                  (no pointers; just bytes)
|  tags: ptr --------> ["admin","beta"]
+----------------------+                       --- decode ---> objects in process B
```

The translation from in-memory to bytes is called **encoding** (also *serialization* or *marshalling*);
the reverse is **decoding** (*deserialization*, *parsing*, *unmarshalling*).

### 1.2 Why not use the language's built-in serialization?

Java has `java.io.Serializable`, Python has `pickle`, Ruby has `Marshal`. They're convenient but usually a
bad idea for anything that crosses a process boundary for long:

- **Language lock-in.** A Python pickle can't easily be read by a Go service.
- **Security.** Decoding can instantiate arbitrary classes; deserializing untrusted input has led to many
  remote-code-execution vulnerabilities.
- **Versioning is an afterthought.** Change a class and old serialized data may fail to load.
- **Efficiency** is often poor.

So in practice data crossing boundaries is encoded in a **language-neutral** format: textual (JSON, XML,
CSV) or binary with a schema (Protobuf, Thrift, Avro).

### 1.3 Where encoding shows up

```
  service A  --(RPC / REST, topic 07)-->  service B           encoding in the request/response
  service A  --(message queue / Kafka, topics 36-37)-->  C    encoding in the message
  service A  --(writes)--> database --(reads)--> service A'   encoding in stored rows/documents
  database   --(replication log, topic 23)--> replica         encoding in the log
  everything --(dumps, CDC, topic 39)--> warehouse / lake     encoding in files (Parquet, Avro)
```

In each case, the writer and the reader can be running **different versions of the code**. That's the
crux of the topic.

---

## 2. Text formats: JSON, XML, CSV

### 2.1 Why they're popular

- **Human-readable.** You can `curl` an endpoint and read the response.
- **Universal.** Every language has a parser; browsers speak JSON natively.
- **No schema needed to read.** The field names are in the data.

### 2.2 Their problems

- **Numbers are ambiguous.** JSON doesn't distinguish integers from floats, and many parsers (including
  JavaScript's) decode numbers as IEEE-754 doubles. Integers above 2^53 lose precision silently:

  ```
  {"tweet_id": 1234567890123456789}  ->  JS reads 1234567890123456800
  ```

  APIs work around this by sending large IDs as strings. XML and CSV can't distinguish a number from a
  string of digits without an external schema.
- **No binary strings.** JSON/XML carry Unicode text, not raw bytes; binary data is typically base64
  encoded, inflating it by ~33%.
- **Schemas are optional and bolted on** (JSON Schema, XML Schema). Many APIs never use them, so the
  "schema" is whatever the code happens to do.
- **CSV is vague.** No types, quoting and escaping rules vary by tool, a comma or newline inside a value
  breaks naive parsers.
- **Verbose.** Every record repeats every field name. Parsing text is CPU-heavy compared with reading
  fixed binary fields.

### 2.3 How big is the overhead?

The same record in different encodings (sizes from the well-known DDIA example):

```json
{ "userName": "Martin", "favoriteNumber": 1337, "interests": ["daydreaming", "hacking"] }
```

| Encoding | Bytes |
|---|---|
| JSON (no whitespace) | 81 |
| MessagePack (binary JSON) | 66 |
| Thrift BinaryProtocol | 59 |
| Thrift CompactProtocol | 34 |
| Protocol Buffers | 33 |
| Avro | 32 |

Binary JSON variants (MessagePack, BSON, CBOR) save a little but still embed field names in every record.
The big win comes from **schema-based** formats, which leave the field names out entirely.

For a public API used by browsers and third parties, JSON's readability usually wins. For high-volume
internal traffic and stored data, the size, speed and — most importantly — the explicit schema of binary
formats tend to win.

---

## 3. Binary schema-based formats

### 3.1 Protocol Buffers and Thrift: field tags

Protocol Buffers (Google) and Thrift (Facebook, now Apache) both require a schema written in an
interface definition language, and a code generator produces classes for each language.

```protobuf
message Person {
  string user_name       = 1;
  int64  favorite_number = 2;
  repeated string interests = 3;
}
```

The key idea: each field has a **field tag** — the number after `=`. The encoded bytes contain the
**tag**, not the name:

```
Protobuf encoding of the record above (conceptually):

  [tag 1, type: length-delimited] [len 6] "Martin"
  [tag 2, type: varint]          1337 as varint (2 bytes)
  [tag 3, type: length-delimited] [len 11] "daydreaming"
  [tag 3, type: length-delimited] [len 7]  "hacking"
```

- Field names never appear in the data, which is why it's compact.
- Integers use **variable-length encoding** (varints): small numbers take 1 byte, larger ones more.
- Each field is self-describing enough (tag + wire type + length) that a parser can **skip a field it
  doesn't recognize**. That one property is the foundation of compatibility.

The tag is the field's true identity. **You can rename a field freely** (names aren't in the data), but
**you can never change a field's tag** — that would make all existing data mean something else.

### 3.2 Avro: no tags, two schemas

Avro (from the Hadoop ecosystem) takes a different approach. Its schema has no tag numbers:

```json
{
  "type": "record", "name": "Person",
  "fields": [
    {"name": "userName",       "type": "string"},
    {"name": "favoriteNumber", "type": ["null", "long"], "default": null},
    {"name": "interests",      "type": {"type": "array", "items": "string"}}
  ]
}
```

The encoded bytes are just the values concatenated in schema order — no tags, no field names, no type
markers:

```
  [len 6] "Martin"  [union branch 1] [1337]  [array: count 2] "daydreaming" "hacking"
```

You **cannot decode Avro data without knowing the exact schema it was written with**. So Avro decoding
always involves two schemas:

- The **writer's schema** — the schema the producer used when encoding.
- The **reader's schema** — the schema the consumer's code expects.

The Avro library **resolves** them side by side, matching fields **by name**:

```
writer's schema                    reader's schema
---------------                    ---------------
userName       (string)  ------>   userName       (string)
favoriteNumber (long)    ------>   favoriteNumber (long)
interests      (array)   --X       (reader doesn't have it -> ignored)
                                   photoURL (string, default null)  <-- not in writer -> filled with default
```

How does the reader know the writer's schema? Depends on context:
- **Large files** (Avro object container files): the writer's schema is stored once in the file header.
- **Database records:** store a schema version number with each record; keep a list of schemas.
- **Network connections:** negotiate the schema at connection setup.
- **Message streams:** prefix each message with a schema ID and look it up in a **schema registry**
  (section 6).

Why go to this trouble? Because Avro schemas have no hand-assigned tags, they're easy to **generate
dynamically** — e.g. from a database table's columns. When a column is added, generate a new schema;
name-based resolution handles the rest. With Protobuf you'd have to assign tags by hand and never reuse
them.

### 3.3 Benefits of schema-based formats

- **Compact**: field names omitted.
- **The schema is documentation** that can't go stale, because decoding depends on it.
- **Compatibility can be checked mechanically** before deploying (section 6).
- **Code generation** gives type-checked access in statically typed languages.
- **Faster** to encode and decode than text parsing.

The costs: you need tooling to read the data (`protoc --decode`, Avro tools), and you must manage schema
files across teams.

---

## 4. Forward and backward compatibility

### 4.1 Definitions

- **Backward compatibility:** *newer code can read data written by older code.*
- **Forward compatibility:** *older code can read data written by newer code.*

```
                writes with           read by
backward:     old schema (v1)  --->  new code (v2)      "new reads old"
forward:      new schema (v2)  --->  old code (v1)      "old reads new"
```

Backward compatibility is the familiar one: you know the old format, so you write the new code to handle
it. Forward compatibility is harder: the old code was written before the new format existed, so the old
code must have been written to **tolerate things it doesn't understand** — typically by ignoring unknown
fields.

### 4.2 Why rolling deploys need both

A **rolling deploy** (topic 47) upgrades a few instances at a time, so for a while old and new versions run
side by side and share databases, caches and queues.

```
time ->      t0           t1                 t2                  t3
instances:   v1 v1 v1     v2 v1 v1           v2 v2 v1            v2 v2 v2
                          ^ mixed versions for the whole window t1..t3
```

During that window:

- A **v2** instance writes a record with a new field; a **v1** instance reads it → needs **forward**
  compatibility.
- A **v1** instance writes a record without the new field; a **v2** instance reads it → needs **backward**
  compatibility.

And the window isn't just minutes:
- A deploy can be **rolled back**, so v1 reads data v2 wrote — forward compatibility again.
- **Mobile apps and client libraries** may never be upgraded; a server may face clients many versions old.
- **Data at rest** is read years later. Messages sit in Kafka for days and get replayed.

### 4.3 The silent-loss trap

There's a subtle forward-compatibility failure: old code that reads a record, modifies it, and writes it
back can **drop fields it didn't know about**.

```
1. v2 writes   { id: 7, name: "Asha", photo_url: "…/a.png" }
2. v1 reads it into a v1 User object (which has no photo_url field) -> field discarded
3. v1 updates name, writes back   { id: 7, name: "Asha R" }
4. photo_url is gone — v2 never deleted it, but it's lost
```

Protobuf's generated code (proto3 since 3.5) **preserves unknown fields** when re-serializing, which
avoids this. With hand-written JSON mapping into typed objects, it's easy to fall into this trap; you need
to explicitly keep unknown fields, or use read-modify-write patterns that only touch the fields you
intend to change (e.g. a partial `UPDATE ... SET name = ...` instead of rewriting the whole record).

---

## 5. Rules for evolving schemas safely

### 5.1 Protobuf / Thrift

| Change | Safe? | Why |
|---|---|---|
| Add a new field with a new tag | Yes | Old code skips the unknown tag (forward); new code sees the field absent and uses the default (backward) |
| Remove a field | Yes, if it was optional, and you **reserve** its tag | Old code that expects it must tolerate absence; the tag must never be reused |
| Rename a field | Yes (on the wire) | Names aren't encoded — but JSON mappings and code using the name break |
| Change a field's tag | **No** | Existing data would be interpreted as a different field |
| Reuse a removed tag for a new field | **No** | Old data with that tag would be decoded into the new field with the wrong meaning |
| Add a `required` field (proto2/Thrift) | **No** | New code rejects old data that lacks it — proto3 removed `required` largely for this reason |
| Change a type | Sometimes | `int32` → `int64` is compatible on the wire, but old code reading a 64-bit value truncates it |

```protobuf
message Person {
  reserved 4;                 // was 'string email = 4;' — never reuse
  reserved "email";
  string user_name = 1;
  int64  favorite_number = 2;
  repeated string interests = 3;
  string photo_url = 5;       // added later: new tag
}
```

### 5.2 Avro

Because Avro matches by **name** and fills missing fields from **defaults**:

| Change | Backward (new reader, old data) | Forward (old reader, new data) |
|---|---|---|
| Add field **with default** | Yes — default fills the gap | Yes — old reader ignores extra field |
| Add field **without default** | **No** — reader has nothing to fill it with | Yes |
| Remove field **that had a default** | Yes | Yes — old reader uses default |
| Remove field **without default** | Yes | **No** — old reader requires it |
| Rename a field | Only with an **alias** in the reader's schema | Tricky (old reader doesn't know the new name) |
| Change type | Only for promotable types (int → long, float → double) | Depends |

The rule of thumb: **only add or remove fields that have a default value.** In Avro, `null` isn't a
default for any type; to make a field nullable you use a union `["null", "long"]` with default `null`.

### 5.3 The general pattern for any format

Even with JSON and no formal schema, the same principles hold:

1. **Readers ignore fields they don't recognize.** (Never fail on unknown fields in a consumer.)
2. **New fields are optional, with a sensible default** when absent.
3. **Never change the meaning or type of an existing field.** Add a new field instead (`price_cents`
   alongside `price`), migrate readers, then retire the old one.
4. **Never reuse an identifier** (tag, field name, enum value) for a different meaning.
5. **Remove in stages:** stop reading it, stop writing it, then delete it — each step deployed everywhere
   before the next.

### 5.4 Enums: a common trap

Adding a value to an enum looks harmless but isn't forward compatible by default:

```
v2 adds OrderStatus.REFUNDED
v1 receives status = REFUNDED  -> v1 has no such value
   Protobuf proto3: keeps the numeric value, but switch statements hit "default:" or crash
   A strict JSON parser into a Java enum: throws on unknown value
```

Write readers to handle an "unknown" enum value explicitly, and add new enum values to readers before
writers start producing them.

---

## 6. Schema registries

### 6.1 The problem in message streams

In a Kafka topic (topic 37), producers and consumers are deployed independently, often by different teams,
and messages persist for days or forever. Avro needs the writer's schema to decode each message; Protobuf
benefits from knowing what was written. Embedding the full schema in every message would destroy the size
advantage.

### 6.2 What a registry does

A **schema registry** (Confluent Schema Registry, AWS Glue Schema Registry, Apicurio) is a service that:

1. **Stores every version of each schema**, assigning each a unique ID. Schemas are grouped by a
   **subject** (typically per topic, e.g. `orders-value`).
2. **Checks compatibility** when a producer registers a new version, rejecting changes that break the
   configured rule.
3. **Serves schemas by ID** to consumers.

```
Producer                         Registry                       Consumer
   |  register schema v3 -------->  | check vs v2 (BACKWARD)      |
   |  <------------- id = 42 ------ | ok, stored                  |
   |                                                              |
   |  message = [magic byte][schema id 42][Avro payload] ----> Kafka ---->|
   |                                                              | read id 42
   |                                 | <------- GET /schemas/42 --|  (cached after first fetch)
   |                                 | ---- writer's schema ----> |
   |                                                              | decode with writer's schema,
   |                                                              | resolve to reader's schema
```

Each message carries only a few bytes of overhead (a magic byte plus a 4-byte schema ID in Confluent's
wire format).

### 6.3 Compatibility modes

The registry enforces a mode per subject:

| Mode | Guarantee | Typical deploy order |
|---|---|---|
| BACKWARD (default in Confluent) | New schema can read data written with the previous schema | Upgrade **consumers first**, then producers |
| FORWARD | Previous schema can read data written with the new schema | Upgrade **producers first**, then consumers |
| FULL | Both directions | Any order |
| *_TRANSITIVE | Same, but checked against **all** previous versions, not just the last | Needed if consumers may replay old data from the beginning |
| NONE | No checks | Only for experimentation |

The `TRANSITIVE` variants matter because compatibility isn't transitive by default: v3 may be compatible
with v2, and v2 with v1, yet v3 can't read v1 data. A consumer replaying a topic from offset 0 sees all of
them.

### 6.4 Why it's valuable beyond Kafka

The registry turns compatibility from a convention into a **gate**: a breaking change is rejected at build
or deploy time instead of discovered as a consumer crash at 3am. The same idea appears as Protobuf
breaking-change detection in CI (e.g. `buf breaking`) for RPC APIs.

---

## 7. Evolving schemas in databases

### 7.1 Data outlives code

A database is like a message sent to your future self. A single table can contain rows written five years
ago by code that no longer exists. When you deploy new code, you don't rewrite every row (that would be
expensive), so **old and new formats coexist on disk**. Most relational databases handle simple changes
lazily: adding a nullable column with no default doesn't rewrite existing rows; reads fill in `NULL` for
rows that predate the column.

### 7.2 Expand / contract migrations

To change a database schema without downtime while old and new code run simultaneously, split the change
into backward- and forward-compatible steps. Example: rename `users.name` to `users.full_name`.

```
Step 1  EXPAND     ALTER TABLE users ADD COLUMN full_name text;           (nullable; instant)
Step 2  DUAL WRITE deploy code that writes BOTH name and full_name; still reads name
Step 3  BACKFILL   UPDATE users SET full_name = name WHERE full_name IS NULL   (in batches)
Step 4  SWITCH     deploy code that reads full_name (still writes both, in case of rollback)
Step 5  STOP OLD   deploy code that writes only full_name
Step 6  CONTRACT   ALTER TABLE users DROP COLUMN name;
```

At every step, the code currently deployed and the code one version back both work against the current
schema. A direct `ALTER TABLE ... RENAME COLUMN` would instantly break every old instance still running
during the rollout.

Practical concerns:
- **Locks.** Some DDL takes a lock that blocks reads/writes on the table while it runs (e.g. changing a
  column type rewrites the table in Postgres). Tools like `gh-ost` and `pt-online-schema-change` (MySQL)
  or `pg_repack`-style approaches copy the table in the background and swap.
- **Backfills** should run in small batches to avoid long transactions, replication lag (topic 23) and
  lock contention.
- **Indexes** should be created online (`CREATE INDEX CONCURRENTLY` in Postgres).

### 7.3 Schema-on-read stores

In a document store (topic 15), the expand/contract idea still applies, just enforced in code: new code
writes the new shape, readers handle both, an optional background job rewrites old documents, and only
then can readers drop the old-shape handling. Some teams store a `schema_version` field in each document
to make this explicit.

### 7.4 Archival and snapshots

When you dump a database to a warehouse or take a snapshot for backup, the dump is written all at once
with the latest schema — a good moment to encode in a self-describing format such as Avro container files
or Parquet (topic 20), which carry their schema with them.

---

## 8. Evolving APIs

### 8.1 Who controls the upgrade order?

For RPC and REST APIs (topic 07), the usual assumption is: **servers are upgraded first, clients later**
(and some clients — mobile apps, third parties — never). So:

- **Requests** need **backward** compatibility: the new server must accept requests from old clients.
- **Responses** need **forward** compatibility: old clients must cope with responses from the new server
  (extra fields, new enum values).

### 8.2 Techniques

- **Additive changes only** within a version: new optional request fields, new response fields.
- **Tolerant readers** on the client side: ignore unknown fields (the "robustness principle": be
  conservative in what you send, liberal in what you accept — applied carefully, not as an excuse for
  sloppy input validation).
- **Explicit versioning** when a breaking change is unavoidable:
  - In the URL: `/v1/orders`, `/v2/orders`
  - In a header: `Accept: application/vnd.shop.v2+json`
  - As a dated version: some public APIs pin each client to the API version that existed when it first
    integrated, and translate between versions server-side.
- **Deprecation process:** announce, measure who still calls the old version, then remove.
- **gRPC/Protobuf** APIs inherit the Protobuf rules from section 5.1; GraphQL avoids versions by only
  adding fields and marking old ones `@deprecated` (topic 07).

---

## 9. A complete rolling-deploy timeline

Scenario: an orders service adds a `discount_cents` field. Instances share a Postgres table and publish
`OrderPlaced` events to Kafka with an Avro schema.

```
T0  All instances v1. Avro schema v1: {order_id, total_cents}.
T1  Migration: ALTER TABLE orders ADD COLUMN discount_cents int;   (nullable, instant)
    v1 ignores the column; works.
T2  Register Avro schema v2 = v1 + {discount_cents: ["null","int"], default null}.
    Registry (BACKWARD) check: new reader can read v1 data (default fills) -> accepted.
T3  Upgrade consumers to code that reads schema v2 (treats null discount as 0).
    They still receive v1 events -> resolved with default. Backward compat in action.
T4  Rolling deploy of producers to v2:
      - v2 instance writes rows with discount_cents and publishes v2 events
      - a still-running v1 instance reads a row with discount_cents -> ignores it (forward)
      - any not-yet-upgraded consumer reading a v2 event: Avro resolves with its v1 reader
        schema, ignoring the extra field (forward)
T5  A bug is found; producers roll back to v1. Old code meets new rows and new events
    in the topic -> still fine, because everything was forward compatible.
```

If at T2 someone had added `discount_cents` **without a default**, the registry would have rejected it;
without a registry, consumers would have crashed on old messages during a replay.

---

## 10. How it connects

- **Topic 07 (API styles):** REST/JSON, gRPC/Protobuf and GraphQL each have their own evolution story built
  on these principles.
- **Topic 15 (data models):** schema-on-read and schema-on-write are about *where* structure is enforced;
  this topic is about *how it changes over time*.
- **Topic 20 (column storage):** Parquet and ORC are columnar, schema-carrying encodings for analytics.
- **Topic 23 (replication):** replication logs encode changes; leaders and followers may run different
  versions during an upgrade.
- **Topics 36–39 (messaging, Kafka, CDC):** long-lived messages and replays make schema compatibility in
  streams critical; CDC events carry the database schema downstream.
- **Topic 42 (event sourcing):** events are stored forever, so every event schema version must stay
  readable.
- **Topic 47 (deployments):** rolling deploys, canaries and rollbacks are exactly what create the
  mixed-version windows that demand compatibility.

---

## Common misconceptions

- "Backward compatibility means old code can read new data" → It's the reverse. Backward: new code reads
  old data. Forward: old code reads new data.
- "JSON doesn't have a schema, so it can't break" → It has an implicit schema in every reader; renaming or
  retyping a field breaks readers just as surely, only without any tool warning you first.
- "Renaming a Protobuf field is a breaking change" → Not on the wire (the tag is what matters), but it is
  for JSON transcoding and for code that references the name.
- "Deleting a field is always safe" → Only if no reader requires it and its tag/name is never reused.
- "Once all servers are upgraded, compatibility doesn't matter" → Old data at rest, messages in a log,
  rollbacks and un-upgraded clients all keep old formats alive.
- "Binary formats are just about size" → The bigger benefits are an explicit, enforceable schema and
  mechanical compatibility checks.
- "Adding an enum value is always safe" → Old readers may reject or mishandle unknown values.

## Further reading (supplements)

- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 4 ("Encoding and Evolution").
- Martin Kleppmann, "Schema evolution in Avro, Protocol Buffers and Thrift" (blog post, 2012).
- Protocol Buffers documentation: "Updating A Message Type" and the proto3 language guide.
- Apache Avro specification: "Schema Resolution".
- Confluent documentation: "Schema Evolution and Compatibility" for Schema Registry.
- Pramod Sadalage & Scott Ambler, *Refactoring Databases* — expand/contract style migrations.

---

## Questions

1. Why can't a process simply write its in-memory objects to disk and have another process read them?
2. Define backward and forward compatibility. Which one is harder to achieve, and why?
3. During a rolling deploy, describe a concrete situation that needs forward compatibility and one that
   needs backward compatibility.
4. Why does Protobuf let you rename a field freely but never change its tag number?
5. A developer deletes Protobuf field 4 (`email`) and later adds a new field `phone = 4`. What happens
   when new code reads old stored data?
6. Avro data contains no field tags or names. How can a reader decode it, and how does it cope with a
   field the writer had but the reader doesn't?
7. In Avro, why is adding a field without a default backward incompatible but still forward compatible?
8. Explain the "silent loss" problem when old code does read-modify-write on records written by new code.
9. What problem does a schema registry solve for Kafka topics, and what does a message carry to use it?
10. Why do `*_TRANSITIVE` compatibility modes exist? Describe when you'd need them.
11. You need to rename a column in a busy Postgres table with zero downtime during a rolling deploy.
    Lay out the steps and explain why `ALTER TABLE ... RENAME COLUMN` alone isn't safe.
12. A public API starts returning a new enum value `REFUNDED`. An old mobile app crashes. What principle
    was violated, and how should both sides have been designed?
13. JSON can represent the number 1234567890123456789. Why might a client read a different number?
14. For an API, which direction of compatibility do requests need, and which do responses need, assuming
    servers are upgraded before clients?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** In-memory data structures are connected by pointers, which are memory addresses meaningful only
inside that process at that moment. The layout also depends on the language, runtime, and architecture.
To store or transmit data it must be turned into a self-contained byte sequence (encoded) that any reader
can decode.

**2.** Backward: new code reads data written by old code. Forward: old code reads data written by new
code. Forward is harder because the old code was written before the new format existed; it can only cope
if it was written to tolerate (ignore, and ideally preserve) things it doesn't understand.

**3.** Forward: a v2 instance writes a row with a new field and a v1 instance (not yet upgraded, or after
rollback) reads it — v1 must ignore the field. Backward: a v1 instance writes a row without the new field
and a v2 instance reads it — v2 must handle the field's absence with a default.

**4.** Encoded data contains only tags, not names, so the name exists only in the schema and generated
code; renaming doesn't change any bytes. The tag is the field's identity in the data: changing it would
make every existing record's value be interpreted as a different (or unknown) field.

**5.** Old records contain tag 4 with an email string. New code interprets tag 4 as `phone`, so it reads
email addresses as phone numbers — silent data corruption (or a parse error if the wire types differ).
The fix is to `reserve` deleted tags and names so they can never be reused.

**6.** The reader must know the exact writer's schema (from the file header, a version number in the
record, connection negotiation, or a schema ID looked up in a registry). Avro decodes using the writer's
schema and then resolves to the reader's schema by field name; fields present in the writer's schema but
not the reader's are ignored.

**7.** Backward = new reader, old data. The old data lacks the field and the reader has no default to fill
in, so decoding fails. Forward = old reader, new data. The old reader doesn't know the field and simply
ignores it, which is always fine.

**8.** If old code decodes a record into an object type that lacks the new field, the field is discarded.
When it writes the object back, the field is gone, even though no one intended to delete it. Prevent it
by preserving unknown fields (as Protobuf's generated code does) or by writing only the fields you
changed (partial updates).

**9.** Producers and consumers are independent and messages are long-lived; consumers need the writer's
schema to decode (essential for Avro), but embedding it in each message wastes space, and nothing stops a
producer from shipping a breaking change. The registry stores versioned schemas, checks compatibility at
registration, and serves schemas by ID. Each message carries a small prefix with the schema ID.

**10.** Compatibility checked only against the previous version isn't transitive: v3 may be compatible
with v2 and v2 with v1, yet v3 can't read v1 data. If a consumer can replay a topic from the beginning
(or data at rest spans many versions), every version must be compatible with all earlier ones.

**11.** Expand/contract: add the new column (nullable); deploy code that writes both columns; backfill
old rows in batches; deploy code that reads the new column (still writing both, for rollback); deploy
code that writes only the new column; finally drop the old column. A direct rename instantly breaks every
instance still running old code (which queries the old name) during the rollout, and makes rollback
impossible.

**12.** Forward compatibility: old readers must tolerate values they don't know. The client should have
mapped unknown enum values to an "unknown" case instead of crashing; the server should have treated adding
enum values as potentially breaking (e.g. gated by client version or announced in advance), knowing old
clients would remain in the field.

**13.** Many JSON parsers, including JavaScript's, decode all numbers as 64-bit IEEE floats, which can
represent integers exactly only up to 2^53. Larger integers are rounded silently. Large IDs should be sent
as strings.

**14.** Requests: backward compatibility (the new server must accept requests from older clients).
Responses: forward compatibility (older clients must handle responses produced by the newer server,
e.g. ignore new fields).

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What is encoding (serialization)?
**A:** Translating in-memory data structures into a self-contained byte sequence; decoding is the reverse.

**Q:** Why avoid language-specific serialization (pickle, Java Serializable)?
**A:** Language lock-in, security risks from instantiating arbitrary classes, poor versioning and efficiency.

**Q:** Two weaknesses of JSON beyond size?
**A:** Ambiguous numbers (large integers lose precision) and no binary strings (base64 needed).

**Q:** Why are Protobuf/Thrift/Avro more compact than JSON?
**A:** Field names are not in the data; the schema supplies them.

**Q:** In Protobuf, what identifies a field in the encoded data?
**A:** Its field tag number.

**Q:** Can you rename a Protobuf field? Change its tag?
**A:** Rename: yes (on the wire). Change tag: never.

**Q:** Why must removed Protobuf tags be reserved?
**A:** Reusing a tag makes old data decode into a new field with the wrong meaning.

**Q:** Why is adding a required field incompatible?
**A:** New code would reject old data lacking it (proto3 dropped `required` for this reason).

**Q:** What two schemas does Avro decoding use?
**A:** The writer's schema (used to encode) and the reader's schema (expected by the code), resolved by field name.

**Q:** Avro's main evolution rule?
**A:** Only add or remove fields that have a default value.

**Q:** Why are Avro schemas good for dynamically generated schemas?
**A:** No hand-assigned tags; fields are matched by name.

**Q:** Backward compatibility?
**A:** New code can read data written by old code.

**Q:** Forward compatibility?
**A:** Old code can read data written by new code.

**Q:** Why do rolling deploys need both directions?
**A:** Old and new versions run simultaneously and read each other's data (and rollbacks put old code back on new data).

**Q:** What is the silent-loss problem?
**A:** Old code reads a record, drops unknown fields, and writes it back, deleting data it never knew about.

**Q:** What does a schema registry do?
**A:** Stores versioned schemas with IDs, enforces compatibility on registration, serves schemas to consumers.

**Q:** What does a message carry when using a registry?
**A:** A small header with the schema ID (plus the encoded payload).

**Q:** BACKWARD registry mode: who upgrades first?
**A:** Consumers first, then producers.

**Q:** Why TRANSITIVE compatibility modes?
**A:** Pairwise compatibility isn't transitive; replaying old data needs compatibility with all previous versions.

**Q:** Expand/contract migration in one line?
**A:** Add new, dual-write, backfill, switch reads, stop old writes, drop old — each step compatible with the previous code.

**Q:** Why is a direct column rename unsafe during a rolling deploy?
**A:** Old instances still query the old name and break instantly; rollback becomes impossible.

**Q:** For APIs where servers upgrade first: requests need ___, responses need ___?
**A:** Requests: backward compatibility. Responses: forward compatibility.

**Q:** Why can adding an enum value break clients?
**A:** Old readers may reject or crash on values they don't know.

**Q:** General rule for readers in any format?
**A:** Ignore (and preferably preserve) unknown fields; treat new fields as optional with defaults.
