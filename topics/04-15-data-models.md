# 15 — Data Models

> Part 4 — Storing Data · Prerequisites: topics 01, 11

Before you pick a database, you pick a *shape* for your data. That choice is more fundamental than the
product name on the box: it decides which questions are cheap to ask, which are expensive, what the
application code has to do by hand, and how painful it is to change your mind later. A data model is the
abstraction a database offers you — tables and rows, JSON documents, keys and opaque values, wide rows
with sorted columns, or vertices and edges. Every model makes some relationships easy to express and
others awkward. This topic is about recognizing the shape of your data and your queries, and matching
them to a model, rather than arguing about "SQL vs NoSQL" as if it were a matter of taste.

By the end of this topic you should be able to:
1. Describe the relational, document, wide-column, key-value and graph models, and the shape of data each fits best.
2. Explain normalization and denormalization, and reason about when to join versus when to embed.
3. Distinguish schema-on-write from schema-on-read, and say what each costs you.
4. Explain why "SQL vs NoSQL" is the wrong question, and instead design starting from access patterns.
5. Recognize the one-to-many, many-to-one and many-to-many relationships in a domain and predict which models handle each well.

---

## 1. What a data model actually is

A data model is a contract between you and the storage layer about **what can be stored** and **what can
be asked**. Applications are built in layers of models:

```
  Your domain:     customers, orders, products, reviews
        |
        v   (you decide how to represent these)
  Data model:      tables / documents / key-values / graph
        |
        v   (the database decides how to represent these)
  Storage engine:  B-tree pages, LSM SSTables, column files  (topics 17, 19, 20)
        |
        v
  Bytes on disk / SSD
```

Each layer hides the complexity of the one below. When you choose a data model you are choosing:

- **What a "unit" of data is.** A row? A whole document? A value behind a key?
- **What relationships look like.** Foreign keys? Nested objects? Explicit edges?
- **What the query language can express.** Arbitrary joins and aggregates? Lookups by key only?
  Traversals of arbitrary depth?
- **Where the work goes.** Into the database (joins, constraints) or into your application code
  (stitching data together, enforcing consistency by hand).

### 1.1 The three relationship shapes

Almost every modelling decision comes down to how a model handles three shapes of relationship. Take a
generic shopping site:

```
one-to-many:   a customer  ──<  their addresses
               an order    ──<  its line items

many-to-one:   many orders ──>  one customer
               many products ──> one category
               many users  ──>  one city ("Bengaluru")

many-to-many:  products  >──<  orders        (an order has many products; a product is in many orders)
               users     >──<  users         (follows / friendships)
```

- **One-to-many where the "many" belongs to the "one"** (an order's line items) forms a *tree*. Trees
  nest naturally: they fit a document.
- **Many-to-one** (lots of things referring to the same shared entity) is where nesting breaks down. If
  you copy "Bengaluru" into every user, renaming it means updating every copy. You want to store it once
  and *refer* to it — which needs some form of join.
- **Many-to-many** is a web, not a tree. Neither side "owns" the other. This is where relational joins
  and graph models shine and where pure document models struggle.

Keep these three in your head; every model below is basically a different answer to "how do I represent
and query them?"

---

## 2. The relational model

### 2.1 Shape

Data lives in **relations** (tables): unordered sets of **tuples** (rows), each with the same named,
typed **attributes** (columns). Relationships are expressed by storing the key of another row (a
**foreign key**) and resolving it at query time with a **join**.

```
customers                       orders                              order_items
+----+--------+----------+      +-----+-------------+--------+      +-----+------------+-----+
| id | name   | city_id  |      | id  | customer_id | placed |      | oid | product_id | qty |
+----+--------+----------+      +-----+-------------+--------+      +-----+------------+-----+
| 1  | Asha   | 10       |      | 501 | 1           | 09-01  |      | 501 | 77         | 2   |
| 2  | Ravi   | 10       |      | 502 | 2           | 09-02  |      | 501 | 78         | 1   |
+----+--------+----------+      +-----+-------------+--------+      | 502 | 77         | 1   |
                                                                    +-----+------------+-----+
cities: (10, "Bengaluru")       products: (77, "Kettle"), (78, "Mug")
```

### 2.2 Why it has dominated for 50 years

- **Declarative queries (SQL).** You say *what* you want; the query planner (topic 18) decides *how*.
  When you add an index, existing queries get faster without being rewritten.
- **Joins are first-class.** Many-to-one and many-to-many relationships are natural. Any table can be
  joined to any other on any column, including in ways nobody anticipated when the schema was designed.
- **Ad-hoc queries.** Because the data isn't pre-shaped for one access path, new questions ("revenue per
  city last month") can be answered without restructuring the data.
- **Integrity constraints.** Foreign keys, unique constraints, `NOT NULL`, `CHECK` — the database
  refuses bad data instead of trusting every code path to be careful.
- **Transactions** across many rows and tables (topic 21).

### 2.3 Where it hurts

- **Object-relational impedance mismatch.** Application code works with nested objects (an `Order` with
  a list of `LineItem`s, each with a `Product`). Tables are flat. An ORM translates, but loading one
  "object" can mean several queries or a multi-way join, and the classic **N+1 query problem** (load 1
  order list, then 1 query per order for its items) comes from this gap.
- **Rigid schema for heterogeneous data.** If every product category has different attributes (a TV has
  screen size, a shirt has fabric), you end up with sparse tables, an entity-attribute-value table, or a
  JSON column bolted on.
- **Horizontal scaling of joins.** Joins are cheap when both tables are on one machine. Once data is
  partitioned across machines (topic 25), a join between partitions means network traffic, and
  cross-partition transactions get expensive (topic 34). Relational databases scale out, but you must
  design for it.

---

## 3. The document model

### 3.1 Shape

A **document** is a self-contained, nested structure — usually JSON or a binary JSON-like format (BSON in
MongoDB). Documents are grouped into collections and looked up by an ID, and most document databases
also support secondary indexes on fields inside the document.

```json
{
  "_id": 501,
  "customer": { "id": 1, "name": "Asha" },
  "placed": "2026-09-01",
  "items": [
    { "product_id": 77, "name": "Kettle", "qty": 2, "price": 1499 },
    { "product_id": 78, "name": "Mug",    "qty": 1, "price": 299 }
  ],
  "shipping": { "line1": "12 MG Road", "city": "Bengaluru" }
}
```

### 3.2 What it is good at

- **Locality.** The whole order is stored together, so loading it is one lookup and (often) one
  contiguous read from disk. In the relational version that same read might touch three tables and many
  pages scattered across disk.
- **Tree-shaped data.** One-to-many relationships where the children belong to the parent — line items,
  a résumé's job history, a blog post's paragraphs — map directly onto nesting.
- **Matches application objects.** What you store looks like what your code manipulates, so there is less
  translation.
- **Flexible structure.** Different documents in the same collection can have different fields
  (see schema-on-read, section 7).

### 3.3 Where it hurts

- **Many-to-one and many-to-many.** If `customer.name` is copied into each order and the customer changes
  their name, every order document has to be found and updated — or you accept stale copies. If you store
  only `customer_id` instead, you need a join, and many document databases either don't support joins or
  support only limited forms (MongoDB's `$lookup`), so the application ends up issuing a second query and
  stitching results together itself. The join didn't go away; it moved into your code.
- **Locality only helps if you need the whole document.** If you usually read one small field of a large
  document, you still pay to load the whole thing. Updates that grow a document may force the database
  to rewrite it entirely.
- **Deeply nested access.** You can't directly refer to "the second line item of order 501" the way you
  can refer to a row by key; you refer to the document and a path within it.

### 3.4 Convergence

The line has blurred. PostgreSQL and MySQL have rich JSON column types with indexing (`jsonb` with GIN
indexes in Postgres). Document databases have added joins, multi-document transactions and schema
validation. The real question is no longer "which product category" but "for this piece of data, do I
want it embedded or referenced?"

---

## 4. The key-value model

### 4.1 Shape

The simplest possible model: a giant dictionary. `put(key, value)`, `get(key)`, `delete(key)`. The value
is usually **opaque** to the database — a blob of bytes it does not look inside.

```
"session:9f2c"    -> {user_id: 1, expires: ...}   (bytes; the DB doesn't care)
"cart:1"          -> [77, 77, 78]
"rate:ip:1.2.3.4" -> 17
```

Examples: Redis (which adds typed values like lists, sets, sorted sets), Memcached, DynamoDB at its
simplest, RocksDB as an embedded engine, etcd.

### 4.2 Why it's useful

- **Predictable performance.** A lookup by key is O(1) in a hash table or O(log n) in a sorted structure,
  regardless of how much other data exists. There is no query planner to surprise you.
- **Trivial to partition.** Hash the key, pick a node (topics 13, 25). No query ever needs two partitions,
  because no query mentions more than one key.
- **Ideal for caches, sessions, feature flags, counters, rate limits** — anything accessed by a
  known identifier.

### 4.3 The cost

You can only ask "give me the value for this key." "All carts containing product 77" is impossible
without scanning everything — unless you maintain a second key-value mapping (`product:77:carts`) yourself,
which is a hand-built secondary index that your application must keep in sync. Key-value stores push
almost all modelling work into **key design**.

Sorted key-value stores soften this: if keys are kept in order, you can do **range scans** (`all keys
starting with "order:1:"`), which lets you cluster related items under a common prefix. This idea grows
into the wide-column model.

---

## 5. The wide-column model

### 5.1 Shape

"Wide-column" (Cassandra, HBase, Bigtable, ScyllaDB; DynamoDB is closely related) is best understood as
a **two-level sorted key-value map**:

```
partition key  ->  ( clustering key -> columns )   sorted by clustering key

"customer:1" -> {
                  ("2026-09-01", order 501) -> {total: 3297, status: "shipped"}
                  ("2026-09-14", order 517) -> {total: 899,  status: "placed"}
                  ("2026-10-02", order 533) -> {total: 120,  status: "placed"}
                }
"customer:2" -> {
                  ("2026-09-02", order 502) -> {total: 1499, status: "delivered"}
                }
```

- The **partition key** decides which node stores the data (it is hashed, topic 13). All rows with the
  same partition key live together on the same replicas.
- The **clustering key** orders rows *within* a partition, on disk. So "the 20 most recent orders of
  customer 1" is a single sequential read from one node.
- Different rows can have different columns; missing columns take no space. The name "wide" comes from
  the fact that a partition can hold a very large number of rows/columns.

Note: "wide-column" is not "column-oriented". Column-oriented storage (topic 20) is about laying out each
column contiguously for analytics; wide-column stores are actually row-ish under the hood.

### 5.2 Query-first design

In a wide-column store you **design the tables from the queries**, not from the entities:

```
Query A: "orders for a customer, newest first"     -> table orders_by_customer
            partition key: customer_id, clustering: placed DESC

Query B: "orders containing a product, by date"    -> table orders_by_product
            partition key: product_id,  clustering: placed DESC
```

The same order is written to both tables. That's deliberate denormalization: each query becomes a
single-partition read. The price is write amplification and the burden of keeping the copies consistent
(often with no multi-table transactions).

### 5.3 Strengths and costs

| Strength | Cost |
|---|---|
| Massive write throughput (usually built on LSM-trees, topic 19) | Queries outside the designed access patterns are slow or impossible |
| Linear horizontal scaling by adding nodes | No joins; limited or no cross-partition transactions |
| Efficient range scans within a partition | "Hot partitions" if one key is very popular (topic 25) |
| Flexible, sparse columns | A new query often means a new table plus a backfill |

---

## 6. The graph model

### 6.1 Shape

When many-to-many relationships are the *main point* of the data — social networks, road networks,
fraud rings, recommendation, knowledge graphs, access-control hierarchies — a graph is the natural model.

- **Vertices (nodes)** — entities, with properties.
- **Edges** — relationships, with a label and properties of their own, connecting two vertices.

```
   (Asha:Person) --FOLLOWS--> (Ravi:Person) --LIVES_IN--> (Bengaluru:City)
        |                          |                            |
     BOUGHT                     REVIEWED {stars: 5}          WITHIN
        v                          v                            v
   (Kettle:Product) <----------------                     (Karnataka:State)
```

The **property graph** model (Neo4j, with the Cypher query language; Apache TinkerPop/Gremlin) and the
**triple store** model (RDF, SPARQL: every fact is `subject predicate object`) are the two main families.

### 6.2 Why not just use relational tables?

You *can* store a graph in two tables (`vertices`, `edges`). The difficulty is in querying:

> "Find everyone who lives somewhere within Karnataka."

People may be linked `LIVES_IN` a city, which is `WITHIN` a district, which is `WITHIN` a state — a
variable number of hops. In SQL this needs a recursive common table expression (`WITH RECURSIVE`), which
works but is clumsy. In Cypher it's one line:

```
MATCH (p:Person)-[:LIVES_IN]->()-[:WITHIN*0..]->(:State {name: "Karnataka"}) RETURN p
```

Graph databases also store adjacency so that following an edge from a vertex is a direct pointer hop
("index-free adjacency" in Neo4j's terms), instead of an index lookup per hop. For deep traversals this
matters.

### 6.3 Costs

- Hard to partition: a good graph partitioning that keeps most edges local is a genuinely hard problem,
  and traversals that cross machines are slow.
- Less mature tooling and fewer operators know them well.
- Overkill if your relationships are mostly one or two hops — a relational join does that fine.

---

## 7. Schema-on-write vs schema-on-read

"Schemaless" is a misleading word. Data always has *some* structure, because the code reading it assumes
fields exist and have types. The real question is **where the schema is enforced**.

### 7.1 Schema-on-write

The database knows the schema and **rejects writes that don't conform**. Relational databases work this
way.

```
INSERT INTO orders (id, customer_id, placed) VALUES (503, 'abc', 'yesterday');
ERROR: invalid input syntax for type integer: "abc"
```

- Like static typing in a programming language: errors are caught early, at the boundary.
- Every reader can rely on the structure.
- Changing the schema means a **migration** (`ALTER TABLE`), which on some databases and for some
  changes can lock a large table or rewrite it. Modern Postgres can add a nullable column, or one with a
  constant default, almost instantly; other changes (changing a column type) still rewrite the table.

### 7.2 Schema-on-read

The database stores whatever you give it; the **structure is interpreted when the data is read**.
Document databases and data lakes (JSON or Parquet files in object storage) work this way.

- Like dynamic typing: flexible, but errors surface late — in the reader, possibly months later.
- Changing the format is cheap at write time: new code just starts writing the new shape. But now the
  collection contains **both shapes**, and every reader must handle both:

```js
// Old docs: { name: "Asha Rao" }   New docs: { first_name: "Asha", last_name: "Rao" }
function firstName(user) {
  if (user.first_name !== undefined) return user.first_name;
  return user.name.split(" ")[0];          // handle the old shape forever, or backfill
}
```

### 7.3 When each makes sense

| Situation | Better fit |
|---|---|
| Many apps/teams read the same data; correctness matters | Schema-on-write |
| Items are genuinely heterogeneous (product attributes by category) | Schema-on-read (or a JSON column) |
| Structure is decided by an external party and may change | Schema-on-read |
| You need the database to enforce invariants | Schema-on-write |
| Exploratory analytics over raw dumps | Schema-on-read |

Either way, you still need to manage how the schema *changes over time* — that is the subject of topic
16.

---

## 8. Normalization vs denormalization

### 8.1 Normalization

**Normalization** means storing each fact exactly once and referring to it by ID everywhere else. In the
relational version of the shop, the city name "Bengaluru" lives in one row of `cities`; customers store
`city_id`.

Why it matters:

- **One place to update.** Rename the city — one row changes. Nothing can drift out of sync.
- **No update anomalies.** With duplication you get bugs like "the order page shows the new product name,
  the invoice shows the old one."
- **Smaller storage** for repeated values.
- **Consistent meaning.** An ID has no meaning to humans, so it never needs to change even when the
  human-facing value does.

The normal forms (1NF, 2NF, 3NF, BCNF) are formal rules for removing different kinds of redundancy. For
practical purposes the rule of thumb is: *every non-key column should describe the key, the whole key,
and nothing but the key.*

### 8.2 Denormalization

**Denormalization** means intentionally duplicating data so that a read can be answered from one place
without a join.

```
Normalized read:       orders JOIN customers JOIN order_items JOIN products   (4 tables)
Denormalized read:     one order document with customer name and product names embedded
```

Why you'd do it:

- **Read speed.** Joins cost CPU and, if the data is on different machines, network round trips.
- **Partitioning.** If everything a query needs is in one partition, the query scales horizontally.
- **Historical truth.** Sometimes the copy is *correct* rather than a compromise: an order should record
  the price and product name **at the time of purchase**, not whatever they are today. That's not
  denormalization so much as recording a different fact.

The cost:

- **Writes do more work.** Changing a duplicated value means updating every copy.
- **Consistency becomes your job.** If the update to copy #3 fails, copies disagree. Without multi-record
  transactions you need patterns like outbox/CDC (topic 39) or background repair to keep them in sync.
- **Storage grows.**

### 8.3 A worked example of drift

```
t0  product 77 name = "Kettle"
    - copied into 40,000 order docs and the "trending" cache
t1  marketing renames product 77 -> "Electric Kettle 1.5L"
t2  update job starts rewriting the 40,000 order docs
t3  job crashes at doc 18,200
t4  search results show "Electric Kettle 1.5L", half of order history shows "Kettle"
```

For order history this may be fine — or even desired. For something like a user's display name shown next
to their comments, it's a visible bug. The decision "should this be a copy or a reference?" is really
"**is this value a snapshot or a live pointer?**"

### 8.4 Joins vs embedding: a decision guide

Embed (document-style) when:
- The child data belongs to the parent and has no meaning on its own (line items of an order).
- You almost always read parent and children together.
- The number of children is bounded (an order has dozens of items, not millions).
- The children are not shared with other parents.

Reference (join-style) when:
- The data is shared by many parents (product, customer, city).
- It changes independently and all readers must see the latest version.
- The "many" side is unbounded (all comments ever on a popular post — embedding would grow the document
  without limit).
- You need to query the child on its own ("all reviews with 1 star").

Many real designs are hybrids: reference the canonical record and embed a small, read-heavy snapshot of
it (e.g. store `customer_id` *and* `customer_name` in the order, accepting the name may be stale).

---

## 9. Why "SQL vs NoSQL" is the wrong framing

### 9.1 The term is muddled

"NoSQL" lumps together key-value stores, document stores, wide-column stores and graph databases, which
have almost nothing in common with each other. Meanwhile:

- Many "NoSQL" databases now have SQL-like query languages (Cassandra's CQL, DynamoDB's PartiQL) and
  transactions.
- Relational databases store and index JSON documents.
- "NewSQL" / distributed SQL systems (Spanner, CockroachDB, YugabyteDB) give you SQL and relational
  semantics while partitioning across many machines.

The historical reasons people moved to NoSQL — needing to scale writes beyond one machine, wanting
flexible schemas, wanting specialized query models — are real, but they are *properties*, not a
category. Ask about the properties.

### 9.2 Think in access patterns

Instead, start from the questions your application asks of its data:

1. **List the queries and their frequencies.**
   "Load an order by ID" (very frequent), "list a customer's recent orders" (frequent), "revenue by city
   per month" (daily batch), "find users two hops away" (rare).
2. **For each, note the key it is accessed by, the result size, and latency needs.**
3. **List the writes and their consistency needs.** "Placing an order must atomically decrement stock" is
   a strong constraint.
4. **Identify the relationship shapes** (section 1.1). Mostly trees? Lots of many-to-one references?
   A dense web?
5. **Estimate growth and whether one machine can hold it.** (Topic 25 covers what changes when it can't.)

Then choose:

| If the dominant pattern is... | A good fit is often... |
|---|---|
| Varied, evolving queries; relationships everywhere; strong integrity needs | Relational |
| Self-contained aggregates read and written as a unit; heterogeneous structure | Document (or relational + JSON columns) |
| Lookup by a known key; extreme throughput; simple values | Key-value |
| Very high write volume; queries known up front; range reads within a key | Wide-column |
| Multi-hop traversals over a web of relationships | Graph |
| Large scans and aggregates over history | Column store / warehouse (topic 20) |
| Full-text relevance search | Search index (topic 51) |

### 9.3 Polyglot persistence

Real systems often use several models side by side — the relational database as the **system of
record**, a Redis cache for hot keys, a search index for text queries, a warehouse for analytics — with
data flowing between them via change data capture or streams (topics 39, 41). The trade-off is that you
now have several copies of the truth to keep in sync, and several systems to operate. Default to fewer
systems until an access pattern genuinely demands another.

### 9.4 A worked example: product reviews

Access patterns:
- (A) Show a product page with the 20 newest reviews — extremely frequent.
- (B) Show all reviews by a user on their profile — occasional.
- (C) Average rating per product — shown on every product card.
- (D) Moderation: find reviews containing banned words — rare, batch.

Reasoning:
- (A) wants reviews **grouped by product, sorted by time**. In relational terms, an index on
  `(product_id, created_at DESC)` (topic 18). In wide-column terms, partition by `product_id`, cluster by
  `created_at`.
- (B) is a different grouping; relational handles it with a second index; wide-column needs a second
  table.
- (C) is an aggregate recomputed on every page view unless you **denormalize** it into the product row
  (`rating_sum`, `rating_count`) and update on each new review.
- (D) is a scan or a job for a search index.

A single relational database with two indexes and one denormalized counter handles all four comfortably
until the review volume outgrows a single machine. That's an access-pattern argument, not a brand argument.

---

## 10. How it connects

- **Topic 11 (caching):** caches are key-value stores; denormalized, precomputed values are what caches
  hold.
- **Topic 16 (encoding & schema evolution):** schema-on-read and schema-on-write both still need a plan for
  how data changes shape over time.
- **Topics 17, 19, 20 (storage engines):** the data model sits on top of a storage engine; B-trees,
  LSM-trees and column files explain *why* certain access patterns are cheap.
- **Topic 18 (indexes & query planning):** how relational databases make the access patterns you identify
  here fast, and how joins are actually executed.
- **Topic 21 (transactions):** normalization relies on multi-row transactions; denormalization without them
  needs other consistency tools.
- **Topic 25 (partitioning):** the partition key is a data-model decision; joins and denormalization
  choices are driven by what lives on which machine.
- **Topics 39, 42 (CDC, event sourcing & CQRS):** systematic ways to maintain denormalized read models
  from a normalized source of truth.
- **Topic 51 (search):** an inverted index is another data model, specialized for text.

---

## Common misconceptions

- "NoSQL databases are schemaless" → The schema exists in the code that reads the data; it's just not
  enforced by the database (schema-on-read). You still have to manage changes to it.
- "Relational databases don't scale" → They scale vertically very far, read-scale with replicas, and can
  be partitioned; distributed SQL systems exist. What gets expensive is cross-partition joins and
  transactions, which is true of any model.
- "Denormalization is always a performance hack" → Sometimes the copy is the correct data (the price at
  purchase time). The question is whether a value should be a snapshot or a live reference.
- "Document databases eliminate joins" → They eliminate joins only for data that is naturally a tree.
  For many-to-one and many-to-many relationships, the join moves into application code.
- "Wide-column means column-oriented" → Wide-column stores (Cassandra, HBase) are partitioned sorted
  key-value stores; column-oriented storage (topic 20) is an on-disk layout for analytics.
- "Pick the model first, then the queries will work out" → Pick the queries first. The model should
  follow from access patterns.
- "Graph databases are for social networks only" → Any domain where variable-depth traversals over
  relationships are the main query (fraud detection, dependency graphs, permissions) fits.

## Further reading (supplements)

- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 2 ("Data Models and Query Languages").
- E. F. Codd, "A Relational Model of Data for Large Shared Data Banks" (1970) — the paper that started it.
- Chang et al., "Bigtable: A Distributed Storage System for Structured Data" (OSDI 2006).
- Alex DeBrie, *The DynamoDB Book* / Rick Houlihan's re:Invent talks on DynamoDB single-table design —
  access-pattern-first modelling taken to its extreme.
- PostgreSQL documentation, "JSON Types" (`jsonb`) — the relational/document convergence in practice.

---

## Questions

1. Why does the document model handle one-to-many relationships well but struggle with many-to-one?
2. A shopping site stores the customer's full name inside every order document. The customer changes
   their name. Walk through what happens, and argue whether this is a bug or correct behaviour.
3. Why is "schemaless" a misleading description of a document database? What is the more accurate term,
   and what cost does it shift and to where?
4. You're adding a required `currency` field to orders. Describe how this plays out under schema-on-write
   versus schema-on-read.
5. In a wide-column store, why do you typically create one table per query? What new problems does that
   introduce?
6. Why is a key-value store easy to partition but hard to query flexibly?
7. When would you prefer a graph database over relational tables with a `edges` table? When would that be
   overkill?
8. Explain why "SQL vs NoSQL" is the wrong framing. What should you ask instead?
9. A collection of blog posts embeds all comments inside each post document. A post goes viral and gets
   200,000 comments. What goes wrong?
10. Give an example where storing a duplicated value is not denormalization in the "performance hack"
    sense, but simply recording the correct fact.
11. What is the object-relational impedance mismatch, and how does the N+1 query problem arise from it?
12. A team stores both `orders_by_customer` and `orders_by_product` tables in Cassandra. A write to the
    second table fails after the first succeeded. What does the user see, and how might you prevent or
    repair this?
13. Why does the relational model handle ad-hoc, unanticipated queries better than a wide-column or
    key-value model?
14. You must store product attributes that differ wildly by category (TVs, shirts, books). Describe two
    ways to model this in a relational database and one in a document database, with trade-offs.

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** One-to-many where the children belong to the parent forms a tree, which nests directly inside a
document and can be read in one lookup with good locality. Many-to-one means many records refer to the
same shared entity. Nesting it means copying it into every referrer, so updates must touch all copies and
copies can drift. Referencing it by ID instead requires a join, which many document databases support
poorly, so the join moves into application code.

**2.** The new name is written to the customer record, but the 300 order documents still contain the old
name. Order history now shows the old name unless you run an update across all of them (which may
partially fail). Whether that's a bug depends on meaning: if the order should record "name at time of
purchase" (like a shipping label), the old name is correct. If the UI is supposed to show the customer's
current name, it's a bug — and you should store a reference (customer_id) and look the name up, or
accept a background job to refresh copies.

**3.** The data still has structure; the code reading it assumes fields and types. The accurate term is
schema-on-read: the structure is interpreted when data is read, not enforced on write. It shifts the
cost of validation and of handling old formats from the database (at write time) to every reader
(forever, or until data is backfilled).

**4.** Schema-on-write: you run a migration — add the column, backfill existing rows (or give it a default),
then add `NOT NULL`. The database guarantees every row has a currency. Schema-on-read: new code starts
writing `currency`; old documents lack it. Every reader must handle the missing case (e.g. default to
"INR"), or you run a backfill job over all documents. Until the backfill completes, both shapes coexist.

**5.** Wide-column stores answer efficiently only queries that read one partition, in clustering-key
order. Each distinct grouping/sort needs its own partition and clustering key, hence its own table. The
cost: each logical write becomes multiple physical writes (write amplification), the copies can diverge
if one write fails (often without multi-table transactions), and any new query pattern needs a new table
plus a backfill.

**6.** Every operation names exactly one key, so you can hash the key to pick a node and no request ever
needs two machines. But the database doesn't understand the value, so it can't search inside it or find
items by anything other than the key; any other access path must be built by the application as extra
key mappings that it keeps in sync.

**7.** Prefer a graph database when the main queries are traversals of variable or deep length
("friends of friends of friends", "all resources this user can reach through nested groups", "rings of
accounts sharing devices") — recursive SQL works but is awkward and each hop is an index lookup. It's
overkill when relationships are mostly one or two hops, which joins do well, and you'd be taking on a
less familiar system that's harder to partition.

**8.** "NoSQL" groups unrelated models, and the boundary has blurred: relational databases store JSON,
NoSQL databases offer SQL-like languages and transactions, and distributed SQL exists. Instead ask:
what are the queries and their frequencies, by what keys is data accessed, what are the relationship
shapes, what consistency do writes need, and how big does it get? Then pick the model that makes the
dominant access patterns cheap.

**9.** The document grows without bound: every new comment rewrites or grows a large document, loading
the post now loads 200,000 comments, the document may exceed size limits (MongoDB caps at 16 MB),
concurrent comment writes contend on one document, and you can't easily page through comments. Unbounded
"many" sides should be referenced (comments in their own collection, indexed by post_id and time).

**10.** An order line item storing the product's name and price at the time of purchase. If the product
later changes price, the order must still reflect what the customer paid. The copy is a snapshot of a
historical fact, not a cache of the current value.

**11.** Applications use nested objects; relational tables are flat, so an ORM must translate between
them. If code loads a list of 50 orders and then lazily accesses `order.items` on each, the ORM issues 1
query for the orders plus 50 queries for items: N+1 round trips instead of one join or one batched
`IN (...)` query.

**12.** The order appears in the customer's order list but not when browsing orders containing that
product (or vice versa). To prevent: use a logged/atomic batch if the store supports it (Cassandra's
logged batches guarantee eventual application of all statements), write to one source and derive the
other via CDC/stream processing (topic 39), or make writes idempotent and retry until both succeed. To
repair: a periodic reconciliation job comparing the tables.

**13.** Relational data isn't pre-arranged for one access path; any table can be joined to any other on
any column, and the query planner figures out how to execute it, using whatever indexes exist. Wide-column
and key-value models physically arrange data around specific keys, so a question that doesn't use those
keys means a full scan or building a new table/index.

**14.** Relational option 1: a table per category (`tvs`, `shirts`) sharing a `products` base table —
strong typing, but many tables and awkward cross-category queries. Relational option 2: a JSON/`jsonb`
`attributes` column on `products`, optionally with a GIN index — flexible, still in one table, weaker
validation. (The entity-attribute-value table is a third option, generally painful to query.) Document
option: each product is a document with whatever fields it needs — natural fit, but validation and
handling of differing shapes live in the application (or in optional schema validation rules).

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What is a data model?
**A:** The abstraction a database exposes for what can be stored and what can be asked (tables, documents, key-values, wide rows, graphs).

**Q:** Which relationship shape fits naturally into a document?
**A:** One-to-many where the children belong to the parent (a tree).

**Q:** Why do many-to-one relationships push you toward joins?
**A:** The shared entity should be stored once and referenced; copying it everywhere means updates must touch every copy.

**Q:** Core idea of the relational model?
**A:** Tables of rows with fixed columns; relationships via foreign keys resolved by joins; declarative queries.

**Q:** Main advantage of the document model?
**A:** Locality — a self-contained aggregate is read and written in one operation, matching application objects.

**Q:** What happens to joins in a document database that doesn't support them?
**A:** They move into application code as extra queries and manual stitching.

**Q:** What is the value in a key-value store, from the database's point of view?
**A:** Usually an opaque blob; the database only understands the key.

**Q:** Why are key-value stores easy to partition?
**A:** Every operation touches exactly one key, so hashing the key picks one node.

**Q:** Wide-column model in one line?
**A:** A two-level sorted map: partition key → (clustering key → columns).

**Q:** What does the partition key decide? The clustering key?
**A:** Partition key: which node(s) store the data. Clustering key: sort order of rows within the partition.

**Q:** Why one table per query in wide-column stores?
**A:** Efficient queries read one partition in clustering order; each grouping/sort needs its own layout.

**Q:** When is a graph database the right model?
**A:** When variable-depth traversals over many-to-many relationships are the main queries.

**Q:** Schema-on-write vs schema-on-read?
**A:** On-write: DB enforces structure and rejects bad writes. On-read: DB stores anything; readers interpret structure.

**Q:** Programming-language analogy for the two schema approaches?
**A:** Schema-on-write ≈ static typing; schema-on-read ≈ dynamic typing.

**Q:** Cost of schema-on-read when a format changes?
**A:** Old and new shapes coexist; every reader must handle both until data is backfilled.

**Q:** What is normalization?
**A:** Storing each fact once and referring to it by ID everywhere else.

**Q:** What is denormalization?
**A:** Deliberately duplicating data so reads avoid joins, at the cost of more write work and consistency risk.

**Q:** The key question when deciding copy vs reference?
**A:** Is the value a snapshot (copy is correct) or a live pointer (must reflect the current value)?

**Q:** When should you not embed children?
**A:** When they're shared, change independently, are queried on their own, or the count is unbounded.

**Q:** Why is "SQL vs NoSQL" the wrong framing?
**A:** NoSQL groups unrelated models and the lines have blurred; the real question is which access patterns must be cheap.

**Q:** First step of access-pattern-driven design?
**A:** List the queries and writes, their frequency, keys, result sizes, and consistency needs.

**Q:** What is polyglot persistence and its main cost?
**A:** Using several data stores for different access patterns; cost is keeping copies in sync and operating more systems.

**Q:** What is the object-relational impedance mismatch?
**A:** The gap between nested application objects and flat relational tables, bridged by ORMs (and causing N+1 queries).

**Q:** Wide-column vs column-oriented?
**A:** Wide-column = partitioned sorted key-value (Cassandra); column-oriented = per-column on-disk layout for analytics.
