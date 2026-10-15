# 18 — Indexes & Query Planning

> Part 4 — Storing Data · Prerequisites: topics 15, 17

Topic 17 showed how a B-tree finds one key among billions in a couple of disk reads. But a table has many
columns, and queries filter, sort and join on all sorts of them. Which columns get a B-tree? What exactly
does that B-tree store? Why does a query sometimes ignore an index you created specifically for it? And
when you write `SELECT ... FROM orders JOIN customers ...`, how does the database decide which table to
read first and how to match rows up? This topic covers the practical side of indexes — what kinds exist,
what they cost, when they can't help — and the query planner, the component that turns your declarative
SQL into a concrete execution strategy.

By the end of this topic you should be able to:
1. Distinguish clustered (primary) and secondary indexes, and explain what each stores and costs.
2. Explain covering indexes and composite indexes, and apply the leftmost-prefix rule.
3. Recognise queries that can't use an index, and explain the write and storage cost of every index.
4. Describe how a cost-based planner chooses between a sequential scan, an index scan and a bitmap scan.
5. Explain nested loop, hash and merge joins, and when a planner picks each.
6. Read a basic `EXPLAIN` plan and reason about why the planner made its choice.

---

## 1. What an index is (and isn't)

An **index** is an additional data structure, derived from a table, that lets the database find rows
matching some condition without scanning the whole table. It's redundant — you could drop it and rebuild
it from the table — which is exactly why it's a trade-off: it costs space and write work, and pays back
in reads.

Without an index on `email`:

```
SELECT * FROM users WHERE email = 'asha@example.com';
-> read every page of users (millions of rows), check each email     O(n)
```

With a B-tree index on `email`:

```
-> descend the email B-tree (3-4 pages, mostly cached) to the matching entry
-> follow its pointer to the row                                      O(log n)
```

Most indexes in relational databases are B-trees (topic 17), because they support equality, ranges
(`<`, `BETWEEN`), sorting (`ORDER BY`) and prefix matches (`LIKE 'abc%'`). Other kinds exist for other
shapes of query — hash indexes (equality only), GIN/inverted indexes (array elements, JSON keys, full-text;
topic 51), GiST/R-trees (geometry; topic 52), BRIN (block-range summaries for huge, naturally ordered
tables). This topic focuses on B-tree indexes since the principles carry over.

---

## 2. Clustered vs secondary indexes

The key question for any index is: **what's in the leaf pages?**

### 2.1 Clustered index (index-organized table)

In a **clustered index**, the table's rows themselves are stored in the leaf pages of the primary key's
B-tree, in primary-key order. The index *is* the table.

```
InnoDB clustered index on orders(id)

                 [ 1000 | 2000 | 3000 ]
                 /      |      |      \
leaves: [id=1001, customer=7, total=…, status=…] [id=1002, …] ...   <- full rows
```

- **MySQL InnoDB** and **SQL Server** (by default) organise tables this way. A table can have only one
  clustered index, because rows can only be physically sorted one way.
- **Lookup by primary key** is one tree descent and you have the row — no extra hop.
- **Range scans on the primary key** read rows that are physically adjacent: very efficient.
- Inserting in random key order causes page splits throughout the table itself (topic 17), which is why
  InnoDB tables strongly prefer monotonically increasing primary keys.

### 2.2 Heap tables

In a **heap**, rows are stored in no particular order, wherever there's space. Every index, including the
primary key index, is separate from the table, and its leaves contain a **pointer to the row's location**.

```
PostgreSQL: heap + separate indexes

index on orders(id)                         heap file (unordered)
[ ... id=1002 -> (page 87, slot 3) ... ]    page 87: [slot1][slot2][slot3: id=1002 row][...]
```

- **PostgreSQL** works this way; the pointer is a **TID / ctid** — (page number, slot within page).
- All indexes are equal; the "primary key" index is just a unique index.
- Every index lookup needs a second step to fetch the row from the heap, unless the index can answer the
  query on its own (section 3).
- Postgres can periodically re-sort a table by an index with `CLUSTER`, but the order isn't maintained as
  new rows arrive.

### 2.3 Secondary indexes

A **secondary index** is any index other than the clustered one — e.g. `orders(customer_id)`. Its keys
need not be unique (many orders per customer), so each entry is either a list of matching row references
or the key made unique by appending the row reference.

What the leaf points to differs:

| Engine | Secondary index leaf holds | To get the full row |
|---|---|---|
| InnoDB (clustered) | the indexed columns + the **primary key value** | second descent of the clustered (PK) B-tree |
| PostgreSQL (heap) | the indexed columns + the **TID** (page, slot) | direct jump to the heap page |

Consequences:

- **InnoDB secondary lookups cost two tree descents** (secondary, then primary). In exchange, when a row
  moves (e.g. a page split in the clustered index), secondary indexes don't need updating, because the PK
  value doesn't change. Also: a wide primary key (say a 36-character UUID string) is copied into **every**
  secondary index entry, bloating all of them.
- **Postgres secondary lookups jump straight to the heap.** But when an update creates a new row version
  at a new location (MVCC, topic 22), every index pointing to the row may need a new entry — unless the
  **HOT** (heap-only tuple) optimisation applies: if no indexed column changed and the new version fits on
  the same page, indexes are left alone.

### 2.4 Random heap access is the hidden cost

For a secondary index that matches many rows, each match may point to a **different page** of the table.
Fetching 10,000 matching rows could mean 10,000 random page reads. This is the single most important
reason the planner sometimes prefers to ignore an index (section 7).

---

## 3. Covering indexes

If an index contains **every column the query needs**, the database can answer from the index alone,
never touching the table. This is a **covering index**, and the plan is an **index-only scan**.

```sql
CREATE INDEX idx_orders_cust_date ON orders (customer_id, placed_at);

SELECT placed_at FROM orders WHERE customer_id = 7 ORDER BY placed_at DESC LIMIT 20;
-- customer_id and placed_at are both in the index -> index-only scan, no heap/clustered lookups
```

To cover extra columns without making them part of the key, some databases let you **include** them in
the leaves only:

```sql
CREATE INDEX idx_orders_cust_date ON orders (customer_id, placed_at) INCLUDE (total, status);  -- Postgres 11+, SQL Server
```

Now `SELECT placed_at, total, status ... WHERE customer_id = 7` is also index-only.

Caveats:
- Bigger indexes cost more memory, disk and write work.
- In InnoDB, every secondary index implicitly contains the primary key, so queries needing only the indexed
  columns plus the PK are covered automatically.
- In Postgres, index-only scans must check the **visibility map** to know whether a page's rows are visible
  to all transactions (MVCC); if the table has many recently modified pages not yet vacuumed, the
  "index-only" scan still visits the heap for those.

---

## 4. Composite indexes and the leftmost-prefix rule

### 4.1 How a composite index is sorted

A **composite** (multi-column) index on `(a, b, c)` is a single B-tree whose keys are tuples, sorted
**lexicographically**: by `a` first, then by `b` within equal `a`, then by `c` within equal `(a, b)`. Like
a phone book sorted by (last name, first name).

```
index on (country, city, signup_date)

(IN, Bengaluru, 2026-01-03)
(IN, Bengaluru, 2026-02-11)
(IN, Bengaluru, 2026-05-20)
(IN, Delhi,     2025-12-30)
(IN, Delhi,     2026-03-02)
(IN, Mumbai,    2026-01-15)
(US, Austin,    2026-02-01)
(US, Boston,    2025-11-11)
```

### 4.2 The leftmost-prefix rule

The index can efficiently find rows only when the condition constrains a **leftmost prefix** of its
columns — just as a phone book helps you find "everyone named Rao" or "Rao, Asha", but not "everyone named
Asha".

| Query condition | Uses the index? | Why |
|---|---|---|
| `country = 'IN'` | Yes | prefix (country) |
| `country = 'IN' AND city = 'Delhi'` | Yes | prefix (country, city) |
| `country = 'IN' AND city = 'Delhi' AND signup_date > '2026-01-01'` | Yes, fully | equality, equality, then range on the next column |
| `city = 'Delhi'` | No (not efficiently) | skips the first column; Delhi rows are scattered across countries |
| `signup_date > '2026-01-01'` | No | skips the first two columns |
| `country = 'IN' AND signup_date > '2026-01-01'` | Partly | narrows to country = IN, then must check every IN row's date |
| `country = 'IN' ORDER BY city` | Yes, for filter **and** sort | within IN, rows are already ordered by city |

Some databases can work around a missing leading column — Oracle and recent MySQL/Postgres versions have
"skip scan" variants that jump between distinct values of the first column — but this only helps when the
leading column has few distinct values.

### 4.3 Range conditions stop the prefix

Once a column has a **range** condition, columns after it can't be used to narrow the search, because
within the range they aren't sorted globally:

```
index (city, signup_date, plan)
WHERE city = 'Delhi' AND signup_date > '2026-01-01' AND plan = 'pro'

-> uses city (=) and signup_date (range) to find a contiguous slice;
   plan is checked row by row inside that slice (still helps if plan is in the index: no heap fetch).
```

Guideline for column order in a composite index:
1. Columns compared with **equality** first.
2. Then the column used for a **range** or for **ORDER BY**.
3. Then extra columns only to make the index **covering**.

Among equality columns, order by which combinations other queries also need (to share one index) rather
than by "most selective first", which is a common myth — for pure equality lookups on all columns, the
order doesn't change efficiency.

### 4.4 Composite index vs separate single-column indexes

Two indexes `(country)` and `(city)` are not the same as one `(country, city)`. A query on both can at
best use one index and filter, or combine the two with a **bitmap AND** (section 7.5) — useful, but more
work than one tree descent into a composite index that already has exactly the matching rows adjacent.

---

## 5. When an index can't be used

An index on `col` stores the values of `col` in sorted order. Anything that prevents the database from
navigating that order defeats it.

### 5.1 Functions and expressions on the column

```sql
WHERE lower(email) = 'asha@example.com'     -- index on email is sorted by email, not lower(email)
WHERE created_at + interval '1 day' > now() -- arithmetic on the column
WHERE date(created_at) = '2026-10-01'
```

Fixes: rewrite so the bare column is on one side (`created_at >= '2026-10-01' AND created_at <
'2026-10-02'`), or create an **expression index** (`CREATE INDEX ON users (lower(email))`).

### 5.2 Type mismatches and implicit conversion

```sql
-- phone is VARCHAR, indexed
WHERE phone = 9876543210        -- MySQL converts every phone to a number to compare -> full scan
```

### 5.3 Leading wildcards

```sql
WHERE name LIKE 'Ash%'   -- prefix: a range in sorted order -> index usable
WHERE name LIKE '%sha'   -- could be anywhere in the order -> no B-tree help
```

Suffix/substring search needs a different index (trigram GIN index in Postgres, or a search engine,
topic 51).

### 5.4 OR across different columns, negations, and NULL handling

- `WHERE a = 1 OR b = 2` can't be answered by one index on `a`; it needs both indexes combined (bitmap OR)
  or a `UNION`.
- `WHERE status != 'shipped'` matches most of the table; an index rarely helps.
- `IS NULL` is indexable in Postgres and MySQL, but in some databases (Oracle) rows with all-NULL keys are
  not stored in B-tree indexes.

### 5.5 Low selectivity: the index *could* be used, but shouldn't

If a condition matches a large fraction of the table (`WHERE country = 'IN'` when 80% of users are in
India), using the index means a tree descent plus a random page read for each of millions of matches —
slower than reading the table sequentially. The planner, if its statistics are good, will choose a
sequential scan. This isn't the index failing; it's the right decision (section 7).

### 5.6 Leftmost-prefix violations

Covered in section 4.2: a composite index whose leading column isn't constrained can't be navigated.

---

## 6. The write cost of every index

Indexes aren't free. Every index on a table is an extra B-tree that must be kept in sync **on every write**.

### 6.1 What one INSERT costs

```
INSERT INTO orders (...) VALUES (...);   -- table has PK + 5 secondary indexes

write the row (heap page or clustered leaf)             1 page modified
insert into each of 5 secondary index B-trees           5 more leaf pages modified (each a random location)
WAL records for all of the above (topic 17)             6 sets of WAL (+ full-page images after checkpoints)
possible page splits in any of them                     more pages
```

- **UPDATE** of an indexed column = delete the old index entry + insert a new one, for each index on that
  column. (In Postgres, any non-HOT update touches every index.)
- **DELETE** removes entries (or marks them dead for later cleanup).

### 6.2 Other costs

- **Memory:** indexes compete with table data for the buffer pool. Ten indexes on a table may be larger
  than the table.
- **Disk and backups:** more data to store, back up and replicate.
- **Planning time:** more options for the planner to consider.
- **Locking and contention:** hot index pages (e.g. the right edge of an index on a timestamp) can become
  contention points.
- **Unique indexes** also enforce a constraint, requiring a check on every insert.

### 6.3 Practical stance

- Create indexes for **real, frequent queries** — check the slow-query log or `pg_stat_statements`.
- **Drop unused indexes** (`pg_stat_user_indexes.idx_scan = 0` over a long period) and **redundant** ones
  (an index on `(a)` is redundant if `(a, b)` exists, for most purposes).
- For write-heavy tables (event logs, metrics), keep indexes minimal; consider an LSM-based store (topic 19).
- Build indexes on large live tables without blocking writes (`CREATE INDEX CONCURRENTLY` in Postgres;
  online DDL in InnoDB).

---

## 7. How a query planner chooses

### 7.1 From SQL to a plan

SQL is declarative: it says *what*, not *how*. The database turns it into an executable plan in stages:

```
SQL text
  -> parse          (syntax tree)
  -> analyze/bind   (resolve tables, columns, types)
  -> rewrite        (expand views, simplify)
  -> PLAN/OPTIMIZE  (enumerate alternatives, estimate the cost of each, pick the cheapest)
  -> execute        (a tree of operators pulling rows from their children)
```

Most relational databases use a **cost-based optimizer**: it generates candidate plans — which access path
for each table, which join order, which join algorithm — estimates the cost of each from **statistics**,
and picks the cheapest.

### 7.2 Statistics and selectivity

The planner's key question for each condition is: **how many rows will match?** That's its
**selectivity** (fraction of rows matched). It estimates this from statistics collected by `ANALYZE`
(Postgres; auto-run by autovacuum) or `ANALYZE TABLE` (MySQL):

- Row count and page count of each table.
- Per column: number of distinct values, fraction of NULLs, the **most common values** and their
  frequencies, and a **histogram** of the rest of the distribution.
- Correlation between a column's values and the physical row order (matters for the cost of an index
  scan).

```
users: 10,000,000 rows, 200,000 pages
country: most common values: IN 0.80, US 0.07, GB 0.03, ...
WHERE country = 'GB' -> estimate 300,000 rows
WHERE country = 'IN' -> estimate 8,000,000 rows
```

The cost model weighs sequential page reads, random page reads (Postgres defaults: `seq_page_cost = 1`,
`random_page_cost = 4`; often lowered toward 1.1 on SSDs) and per-row CPU work.

### 7.3 Access path 1: sequential scan

Read every page of the table in order and test each row.

- Cost ≈ number of pages (sequential, cheap per page) + rows × CPU.
- **Wins when** the query needs a large fraction of rows, the table is small, or no index applies.
- Sequential I/O is fast and read-ahead friendly; reading 100% of pages sequentially can beat reading 10%
  of them randomly.

### 7.4 Access path 2: index scan

Descend the index to the first matching entry, walk the leaf entries in order, and for each one fetch the
row from the table.

- Cost ≈ tree descent + matching index entries + **one possibly random table page per matching row**.
- **Wins when** few rows match (high selectivity), or when the output must be **sorted by the index key**
  (an index scan returns rows already ordered, avoiding a sort — and with `LIMIT`, it can stop early).
- Variant: **index-only scan** when the index covers the query (section 3) — no table access at all.

### 7.5 Access path 3: bitmap scan (Postgres)

A middle ground for "moderately many" matching rows:

```
1. Bitmap Index Scan: walk the index, collect the TIDs of all matching rows into an in-memory bitmap,
   one bit per heap page (or per row) — no heap access yet.
2. (Optional) combine bitmaps from several indexes with AND / OR.
3. Bitmap Heap Scan: visit the marked heap pages IN PHYSICAL ORDER, each page once,
   rechecking the condition on its rows.
```

- Turns random heap I/O into (mostly) **ordered** I/O, and reads each page **once** even if it contains
  many matches.
- Can **combine indexes**: `WHERE country = 'GB' AND plan = 'pro'` with separate indexes on each →
  BitmapAnd.
- Loses index order, so it can't satisfy `ORDER BY` without a sort.
- If the bitmap grows too large for `work_mem`, it degrades to page-level ("lossy") bits and rechecks every
  row on those pages.

MySQL's analogous features are "index merge" and Multi-Range Read.

### 7.6 Choosing between them

```
fraction of table matched  ->   0.01%        1%           10%           50%+
typical choice                index scan   bitmap scan  bitmap/seq    seq scan
```

(The exact crossover depends on row width, physical correlation, caching and cost settings — not fixed
numbers.)

A worked example, users table with 10M rows on 200,000 pages, index on `country`:

| Query | Est. rows | Index scan cost (approx.) | Seq scan cost | Choice |
|---|---|---|---|---|
| `country = 'NZ'` | 2,000 | ~2,000 random page reads | 200,000 sequential pages | index (or bitmap) scan |
| `country = 'GB'` | 300,000 | up to ~200,000 random reads (most pages hit anyway) | 200,000 sequential | bitmap scan or seq scan |
| `country = 'IN'` | 8,000,000 | every page, randomly, many times | 200,000 sequential | seq scan |

### 7.7 Reading EXPLAIN

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE country = 'NZ';
```
```
Bitmap Heap Scan on users  (cost=40.1..6012.3 rows=2050 width=96) (actual time=0.9..7.8 rows=1987 loops=1)
  Recheck Cond: (country = 'NZ')
  Heap Blocks: exact=1940
  ->  Bitmap Index Scan on users_country_idx  (cost=0.0..39.6 rows=2050 width=0) (actual rows=1987)
        Index Cond: (country = 'NZ')
Planning Time: 0.2 ms
Execution Time: 8.4 ms
```

- Read the tree from the innermost node outward; children feed parents.
- `cost=start..total` is in arbitrary planner units; `rows=` is the **estimate**.
- With `ANALYZE`, `actual ... rows=` shows reality. **A large gap between estimated and actual rows is the
  number-one cause of bad plans.**

### 7.8 Why planners get it wrong

- **Stale statistics:** a table grew 100× since the last `ANALYZE`, or a bulk load just happened.
- **Correlated columns:** the planner assumes conditions are independent. `city = 'Delhi' AND country =
  'IN'` is estimated as P(Delhi) × P(IN), underestimating badly because one implies the other. Extended
  statistics (`CREATE STATISTICS` in Postgres) can capture this.
- **Skew and parameterised queries:** a cached generic plan for `WHERE customer_id = $1` may be good for a
  typical customer and terrible for one customer with millions of orders.
- **Estimates compound through joins:** a 10× error at the bottom of a five-join plan can become 10,000×
  at the top, leading to a nested loop that should have been a hash join.

Fixes, roughly in order: refresh statistics, increase the statistics sample for a skewed column, add
extended statistics, rewrite the query, add the right index; use hints (where supported) as a last resort.

---

## 8. Join algorithms

A join matches rows from two inputs on a condition. There are three fundamental algorithms. Call the
inputs **outer** (left) and **inner** (right).

### 8.1 Nested loop join

```
for each row r in outer:
    for each row s in inner where s.key = r.key:     # ideally via an index on inner.key
        emit (r, s)
```

- **Without an index on the inner side:** O(N × M) — every outer row scans the whole inner input. Only
  acceptable when both are tiny.
- **With an index on the inner join key** ("index nested loop"): O(N × log M). Excellent when the **outer
  side is small** — e.g. 20 orders for one customer, each looking up its customer row by primary key.
- **Supports any join condition**, including inequalities and arbitrary expressions (`a.x < b.y`).
- **Produces rows immediately** — no build phase — so it pairs well with `LIMIT`.
- Danger: if the planner underestimates the outer side (expects 10 rows, gets 1,000,000), you get a
  million index lookups.

### 8.2 Hash join

```
BUILD:  scan the smaller input; insert each row into an in-memory hash table keyed by join key
PROBE:  scan the larger input; for each row, look up its key in the hash table, emit matches
```

```
customers (small, 50k rows)  --build-->  hash table { customer_id -> row }
orders (large, 50M rows)     --probe-->  for each order: lookup customer_id -> emit joined row
```

- O(N + M): each input read once. The workhorse for **large, unsorted inputs with an equality condition**.
- **Equality joins only** (`a.id = b.id`): hashing destroys order, so it can't do ranges.
- Needs memory for the build side. If it doesn't fit (`work_mem` in Postgres), the join partitions both
  inputs into batches on disk by hash ("grace hash join") and processes one batch at a time — correct but
  slower.
- Has a **startup cost** (the build) before the first row is produced.
- MySQL got hash joins only in 8.0.18 (2019); before that, it relied on nested loops.

### 8.3 Merge join (sort-merge join)

```
sort outer by key, sort inner by key (or read both from indexes already in key order)
walk both with two pointers:
    if left.key < right.key: advance left
    if left.key > right.key: advance right
    if equal: emit all matching combinations, advance
```

```
left (sorted):  1  3  3  5  8
right (sorted): 3  4  5  5  9
matches:        (3,3) (3,3) (5,5) (5,5)
```

- O(N + M) after sorting; sorting costs O(N log N) unless the inputs are **already sorted** (from a B-tree
  index scan or an earlier sort).
- **Shines when both inputs are large and already ordered by the join key** — e.g. joining two tables on
  their primary keys, or when the result must be sorted by that key anyway.
- Handles equality and can be adapted to some range joins; streams with low memory use.

### 8.4 Comparison

| | Nested loop | Hash join | Merge join |
|---|---|---|---|
| Best when | Outer small, index on inner key | Large unsorted inputs, equality | Both inputs large and already sorted |
| Join conditions | Any | Equality only | Equality (some ranges) |
| Memory | Low | Build side in memory (spills if not) | Low (sorts may spill) |
| First row latency | Immediate | After build | After sorts (immediate if presorted) |
| Output order | Outer's order | Probe side's order (roughly) | Sorted by join key |
| Failure mode | Explodes if outer is underestimated | Spills to disk if build side underestimated | Expensive sorts if inputs unsorted |

### 8.5 Join order

For a query joining n tables, the planner must also choose the **order** of joins — the number of
possible orders grows factorially. Postgres searches exhaustively up to a limit (`join_collapse_limit`,
default 8) and switches to a genetic algorithm (GEQO) beyond `geqo_threshold` (default 12 tables). The
general principle: **join early the pairs that shrink the intermediate result the most**, so later joins
handle fewer rows.

### 8.6 A worked example

```sql
SELECT c.name, o.id, o.total
FROM customers c JOIN orders o ON o.customer_id = c.id
WHERE c.email = 'asha@example.com';
```

Likely plan:

```
Nested Loop
  -> Index Scan using customers_email_key on customers c   (Index Cond: email = '...')   -> 1 row
  -> Index Scan using orders_customer_id_idx on orders o   (Index Cond: customer_id = c.id) -> ~40 rows
```

One customer row (unique index), then an index lookup for their orders: nested loop is ideal.

Now drop the `WHERE` and compute revenue per customer across all 50M orders:

```
HashAggregate
  -> Hash Join (o.customer_id = c.id)
       -> Seq Scan on orders o           (probe side, 50M rows)
       -> Hash
            -> Seq Scan on customers c   (build side, 1M rows)
```

Most rows are needed: sequential scans plus a hash join beat millions of index lookups. If the index on
`orders.customer_id` didn't exist, the first query would also degrade: a nested loop would scan all of
orders, so the planner would likely choose a hash join with a sequential scan of orders.

---

## 9. How it connects

- **Topic 15 (data models):** access patterns identified there become the indexes here; a key-value store
  is essentially "only the primary index".
- **Topic 17 (B-trees):** the structure every index here is built from; page splits and WAL cost explain
  why every index slows writes.
- **Topic 19 (LSM-trees):** secondary indexes on LSM stores (Cassandra, RocksDB-based systems) work
  differently and can be costlier.
- **Topic 20 (column storage):** analytic engines mostly skip B-tree indexes in favour of scanning
  compressed columns with min/max metadata.
- **Topic 22 (concurrency control):** MVCC is why Postgres index-only scans need the visibility map and why
  updates create new index entries.
- **Topic 25 (partitioning):** secondary indexes in a partitioned database are local or global, with very
  different costs.
- **Topic 40 (batch processing):** distributed joins (broadcast hash join, sort-merge join) are these same
  algorithms spread across machines.
- **Topics 51, 52 (search, geospatial):** specialised index structures for text and location.

---

## Common misconceptions

- "Adding an index can only help" → Every index slows writes, uses memory and disk, and may still be
  ignored for low-selectivity queries.
- "If an index exists, the database will use it" → The planner uses it only if the query can navigate it
  (leftmost prefix, no function on the column) and if it's estimated cheaper than a scan.
- "A sequential scan in the plan means something is wrong" → For queries touching a large fraction of a
  table, it's the fastest option.
- "Indexes on `a` and `b` are as good as an index on `(a, b)`" → A composite index finds the matching rows
  directly; separate indexes need a bitmap AND or filtering.
- "Put the most selective column first in a composite index" → Put equality columns first, then the range
  or sort column; the order of equality-only columns matters for reuse, not efficiency.
- "Hash joins are always faster than nested loops" → Nested loops with an index win when the outer side is
  small, and they return the first rows immediately.
- "The planner is smart, so plans are always right" → Plans are only as good as statistics and estimates;
  stale stats and correlated columns cause most bad plans.

## Further reading (supplements)

- Markus Winand, *SQL Performance Explained* and the free site use-the-index-luke.com — the best practical
  guide to indexes and the leftmost-prefix rule.
- PostgreSQL documentation: "Indexes" chapter, "Using EXPLAIN", and "How the Planner Uses Statistics".
- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 3 (indexes, clustered vs secondary).
- Leis et al., "How Good Are Query Optimizers, Really?" (VLDB 2015) — cardinality estimation errors in practice.
- CMU 15-445/645 Database Systems lectures (Andy Pavlo) on query execution, join algorithms and optimisation.
- MySQL documentation: "Optimization and Indexes" and "Clustered and Secondary Indexes" (InnoDB).

---

## Questions

1. What is stored in the leaf pages of a clustered index versus a secondary index in InnoDB? What does a
   secondary index lookup cost as a result?
2. Why does Postgres not have clustered indexes in the InnoDB sense, and what does its secondary index leaf
   point to? What's the downside when rows are updated?
3. Why is a wide primary key (e.g. a 36-byte UUID string) particularly costly in InnoDB?
4. Explain what makes a query an index-only scan. Give an example and how `INCLUDE` helps.
5. You have an index on `(country, city, signup_date)`. For each, say whether it can use the index and how:
   (a) `city = 'Delhi'` (b) `country = 'IN' AND signup_date > '2026-01-01'`
   (c) `country = 'IN' AND city = 'Delhi' ORDER BY signup_date`.
6. Why does a range condition on one column of a composite index stop the following columns from narrowing
   the search?
7. List four reasons a query might not use an index on `email` even though one exists.
8. A table has 12 indexes and inserts are slow. Explain where the time goes.
9. Why might the planner choose a sequential scan for `WHERE country = 'IN'` even though there's an index on
   `country`? Is that a mistake?
10. What problem does a bitmap scan solve compared with a plain index scan? What does it give up?
11. The planner estimated 10 rows for a filter but 2 million rows matched, and the query took minutes.
    What plan likely went wrong, and how would you investigate and fix it?
12. When does a nested loop join beat a hash join? When does it catastrophically lose?
13. Why can't a hash join be used for `JOIN ... ON a.start_time < b.end_time`?
14. When is a merge join the natural choice?
15. Why do planners make bad estimates for `WHERE city = 'Delhi' AND country = 'IN'`?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** Clustered index leaves hold the complete rows, sorted by primary key. Secondary index leaves hold
the indexed column values plus the primary key value. A secondary lookup therefore descends the secondary
B-tree to find the PK, then descends the clustered B-tree to fetch the row — two tree traversals (unless
the secondary index covers the query).

**2.** Postgres stores rows in an unordered heap, and every index (including the PK index) is separate.
Index leaves hold a TID (page number, slot) pointing directly into the heap. Because MVCC writes a new row
version, often at a new location, an update may require new entries in every index; the HOT optimisation
avoids this only if no indexed column changed and the new version fits on the same page.

**3.** Every secondary index entry contains the primary key, so a wide PK bloats every secondary index,
using more memory and disk. Random UUIDs also insert at random positions in the clustered index (the
table itself), causing page splits, fragmentation and poor cache locality.

**4.** When every column the query reads (in SELECT, WHERE, ORDER BY) is present in the index, the database
can answer from the index alone without visiting the table. E.g. index `(customer_id, placed_at)` and
`SELECT placed_at FROM orders WHERE customer_id = 7`. `INCLUDE (total)` stores extra columns in the leaves
(not as part of the sort key), so queries also selecting `total` remain index-only.

**5.** (a) No — skips the leading column; Delhi rows are scattered under each country (maybe a skip scan
if few countries). (b) Partly — seeks to country = IN, then must check every IN entry's signup_date because
dates aren't sorted across cities. (c) Yes, fully — equality on country and city finds a contiguous slice
already sorted by signup_date, so no sort is needed.

**6.** Within a range of the earlier column, the later column's values are sorted only within each
distinct value of the earlier column, not across the whole range. So there's no single contiguous region
of the index for the combined condition; the later column can only be filtered entry by entry.

**7.** A function or expression is applied to the column (`lower(email)`); implicit type conversion; a
leading wildcard (`LIKE '%@example.com'`); an OR with an unindexed condition; the planner estimates that so
many rows match a sequential scan is cheaper; stale statistics mislead it. (Any four.)

**8.** Each insert writes the row plus an entry into each of 12 B-trees, each likely on a different leaf
page: 13 page modifications, possibly page splits, WAL for all of them (and full-page images after
checkpoints), more dirty pages for the buffer pool to flush, and unique checks for unique indexes. The
indexes also compete for buffer pool memory, increasing cache misses.

**9.** If 80% of rows match, an index scan would do a random table page read for almost every row, and
read most pages anyway, many times over. A sequential scan reads each page once in order. It's the correct
choice, assuming the statistics are accurate.

**10.** A plain index scan may read heap pages randomly and repeatedly (one fetch per matching row). A
bitmap scan first collects all matching row locations, then reads each needed heap page once in physical
order, and can combine multiple indexes with AND/OR. It gives up the index's ordering (needs a separate
sort for ORDER BY) and has to recheck conditions, especially if the bitmap becomes lossy.

**11.** Likely a nested loop join (or index scan) chosen because of the low estimate, executed 2 million
times. Investigate with `EXPLAIN ANALYZE`, comparing estimated vs actual rows at each node to find where
the estimate diverges. Fix by running `ANALYZE`, raising the statistics target for skewed columns, adding
extended statistics for correlated columns, or rewriting the query; adding a better index may also help.

**12.** It wins when the outer input is small and there's an index on the inner join key: a handful of
cheap lookups, no build phase, first rows immediately (good with LIMIT). It loses catastrophically when the
outer side is large (especially when underestimated) or there's no index on the inner side — then it does
millions of lookups or N × M comparisons.

**13.** A hash table only finds rows whose key is exactly equal to the probe key; hashing destroys
ordering, so there's no way to find all keys less than a value. Inequality joins need nested loops (or a
merge-style algorithm on sorted inputs).

**14.** When both inputs are large and already sorted on the join key — e.g. read via B-tree indexes on
that key, or sorted for an earlier step — and/or the output needs to be sorted by the key. Then the join is
a single linear pass with little memory.

**15.** By default, planners assume conditions on different columns are independent and multiply their
selectivities. But Delhi implies India, so the true selectivity is just P(Delhi), much larger than P(Delhi)
× P(IN). The underestimate can lead to bad join choices. Extended (multi-column) statistics fix this.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What is an index?
**A:** A redundant, derived structure that finds rows matching a condition without a full scan; faster reads, slower writes.

**Q:** Clustered index?
**A:** The table rows stored in the leaves of the primary-key B-tree, in key order.

**Q:** How many clustered indexes can a table have, and why?
**A:** One — rows can be physically ordered only one way.

**Q:** What does an InnoDB secondary index leaf contain?
**A:** The indexed columns plus the primary key value.

**Q:** What does a Postgres index leaf point to?
**A:** A TID: (heap page number, slot) of the row version.

**Q:** Cost of an InnoDB secondary index lookup?
**A:** Two B-tree descents: secondary index, then the clustered index.

**Q:** What is HOT in Postgres?
**A:** Heap-only tuple update: if no indexed column changes and it fits on the same page, indexes aren't updated.

**Q:** Covering index?
**A:** An index containing every column a query needs, enabling an index-only scan.

**Q:** What does `INCLUDE` do in an index?
**A:** Stores extra columns in the leaves without making them part of the sort key, to make the index covering.

**Q:** How is a composite index `(a, b, c)` sorted?
**A:** Lexicographically: by a, then b within a, then c within (a, b).

**Q:** The leftmost-prefix rule?
**A:** A composite index can narrow the search only on a leftmost prefix of its columns.

**Q:** Recommended column order in a composite index?
**A:** Equality columns, then the range/sort column, then covering-only columns.

**Q:** Why does `WHERE lower(email) = ...` skip an index on email?
**A:** The index is sorted by email, not lower(email); use an expression index or rewrite.

**Q:** Why can't `LIKE '%abc'` use a B-tree?
**A:** Matches can be anywhere in the sort order; only prefixes form a range.

**Q:** Write cost of an index?
**A:** Every insert/update/delete must modify it too: extra page writes, WAL, splits, memory.

**Q:** What statistics does a planner use?
**A:** Row/page counts, distinct values, null fraction, most common values, histograms, physical correlation.

**Q:** When does a sequential scan win?
**A:** When a large fraction of rows match, the table is small, or no usable index exists.

**Q:** When does an index scan win?
**A:** Few rows match, or output must be in index order (especially with LIMIT).

**Q:** How does a bitmap scan work?
**A:** Collect matching row locations from index(es) into a bitmap, then read heap pages once in physical order.

**Q:** Number-one cause of bad plans?
**A:** Wrong row-count estimates (stale stats, correlated columns, skew).

**Q:** Nested loop join is best when?
**A:** Small outer input and an index on the inner join key.

**Q:** Hash join in two phases?
**A:** Build a hash table on the smaller input, probe it with each row of the larger input.

**Q:** Hash join limitation?
**A:** Equality conditions only; needs memory for the build side (spills to disk otherwise).

**Q:** Merge join is best when?
**A:** Both inputs are large and already sorted by the join key.

**Q:** Which join algorithm supports arbitrary conditions like `<`?
**A:** Nested loop.

**Q:** How does Postgres handle many-table join ordering?
**A:** Exhaustive search up to a limit; genetic optimizer (GEQO) beyond ~12 tables.
