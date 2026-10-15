# 20 — Column storage, OLTP vs OLAP

> Part 4 — Storing Data · Prerequisites: topics 15, 17, 18 (19 helps)

The storage engines in topics 17–19 were built for one kind of work: many small requests, each reading or
writing a handful of rows by key. "Show order 4512", "add this item to the cart", "decrement stock for SKU 88".
But businesses also ask a completely different kind of question: "what was revenue per region per month over the
last two years?" That query touches hundreds of millions of rows but only three or four columns, and it is
asked by a few analysts, not millions of users. Running it on the same database, laid out the same way, is
slow and can hurt the users. This topic explains why these two workloads diverge so much that they get
different engines, how column-oriented storage makes analytical scans tens to hundreds of times faster, and
how analytical data is organized into warehouses, star schemas and materialized views.

By the end of this topic you should be able to:
1. Distinguish OLTP and OLAP workloads by their access patterns, and explain why each wants a different engine.
2. Explain row-oriented vs column-oriented storage and what each makes cheap and expensive.
3. Explain why columns compress so well (run-length, dictionary, bitmap encoding) and why that speeds queries.
4. Explain vectorized execution and why it fits columnar data.
5. Describe how a data warehouse is fed (ETL/ELT) and structured (fact and dimension tables, star/snowflake).
6. Explain materialized views and data cubes, and their trade-offs.

---

## 1. Two kinds of workload

### 1.1 OLTP: online transaction processing

The term comes from commercial transactions (sales, payments), but now just means "the application's
day-to-day reads and writes". A generic shopping site's database handles:

```
SELECT * FROM orders WHERE order_id = 4512;
UPDATE inventory SET qty = qty - 1 WHERE sku = 'A-88';
INSERT INTO cart_items (cart_id, sku, qty) VALUES (991, 'B-12', 2);
```

Characteristics:
- Huge number of small queries, each touching a few rows found **by key** via an index (topic 18).
- Reads and writes interleaved; writes are frequent, need low latency, and need transactions (topic 21).
- Queries are mostly known in advance (the application's code paths).
- Each query wants **whole rows** (all columns of one order).
- Latency budget: milliseconds. Working set: the recent, hot rows.
- Users: end customers via the application.

### 1.2 OLAP: online analytical processing

The analytics team asks:

```
SELECT region, date_trunc('month', order_date) AS month, SUM(amount)
FROM   orders
WHERE  order_date >= '2024-01-01'
GROUP  BY region, month;
```

Characteristics:
- Few queries, each scanning **millions to billions of rows**, computing aggregates (sums, counts, averages).
- Touches **few columns** out of possibly hundreds.
- Mostly read; data arrives in bulk loads or streams, rarely updated row by row.
- Queries are ad hoc: an analyst explores, a dashboard groups by a new dimension.
- Latency budget: seconds to minutes is fine. Throughput (rows scanned per second) is what matters.
- Users: analysts, data scientists, BI dashboards, ML pipelines.

### 1.3 Side by side

| | OLTP | OLAP |
|---|---|---|
| Read pattern | Few rows by key | Scan many rows, aggregate |
| Write pattern | Random, low-latency, single rows | Bulk load (batch or stream) |
| Columns used | Most of the row | A few of many |
| Query shapes | Known, repeated | Ad hoc, exploratory |
| Dataset size | GB to TB (current state) | TB to PB (history) |
| Bottleneck | Disk seeks / latency per request | Disk/memory bandwidth, CPU per row |
| Typical engines | Postgres, MySQL, Oracle, SQL Server, DynamoDB | Snowflake, BigQuery, Redshift, ClickHouse, DuckDB, Druid |

### 1.4 Why not run analytics on the OLTP database?

Early on, people did. It breaks down for several reasons:

1. **Interference.** A two-minute analytic scan competes for disk bandwidth, CPU, and buffer pool (topic 17)
   with customer traffic. It evicts the hot OLTP pages from the cache, so customer queries suddenly hit disk.
   Long-running reads can also hold back vacuum in MVCC systems (topic 22), causing bloat.
2. **Wrong layout.** A row store reads entire rows to get three columns (section 2). For a 100-column table,
   ~97% of bytes read are wasted.
3. **Wrong indexes.** OLTP indexes are designed for point lookups. Analysts' ad hoc filters and group-bys
   rarely match them, and adding indexes for every analytic query would slow down writes (topic 18).
4. **Data spread across systems.** Analytics wants to join orders with marketing, support tickets, payments
   provider data, and so on, which live in different databases and services.
5. **History.** OLTP stores current state (the address the customer has now); analytics often wants history
   (what was the address when the order was placed).

The answer is a separate system optimized for analytics: a **data warehouse**. (Read replicas, topic 23, solve
point 1 partly, but not the others.)

---

## 2. Row-oriented vs column-oriented storage

### 2.1 Row-oriented layout

In a row store (Postgres, MySQL), all values of one row are stored together on a page:

```
 Page 1: [1, 2024-03-01, 'EU', 'A-88', 2, 39.98, ...97 more columns...]
         [2, 2024-03-01, 'US', 'B-12', 1, 12.50, ...]
         [3, 2024-03-02, 'EU', 'C-07', 5, 80.00, ...]
 Page 2: ...
```

Great for OLTP: fetching order 2 is one index lookup and one page read that returns everything.

Terrible for `SUM(amount) GROUP BY region`: to read two columns you must read every page of the table, including
all 98 columns you don't need. Bytes read ∝ rows × full row width.

### 2.2 Column-oriented layout

A column store keeps all values of **each column** together, in its own file or segment:

```
 order_id:   1, 2, 3, 4, 5, ...
 order_date: 2024-03-01, 2024-03-01, 2024-03-02, 2024-03-02, ...
 region:     EU, US, EU, EU, US, ...
 sku:        A-88, B-12, C-07, A-88, ...
 qty:        2, 1, 5, 1, ...
 amount:     39.98, 12.50, 80.00, 19.99, ...
```

The crucial invariant: **every column file stores values in the same row order.** The 3rd value in `region` and
the 3rd value in `amount` belong to the same row. To reconstruct a row you take the i-th entry from each column.

Now `SUM(amount) GROUP BY region` reads only the `region` and `amount` files. For a 100-column table that's ~2%
of the data. The effect is huge before any other trick.

### 2.3 What each makes cheap and expensive

| Operation | Row store | Column store |
|---|---|---|
| Fetch one whole row by key | 1 page read | One read per column (expensive for wide rows) |
| Insert one row | Append to one page | Write to every column file (expensive) |
| Update one field | Modify one page | Rewrite a value inside a compressed column segment (expensive) |
| Scan 3 of 100 columns over 1B rows | Reads all 100 columns | Reads 3 columns |
| Compression ratio | Modest (mixed types per page) | High (same type, similar values) |

So column stores are bad at exactly what OLTP does. That's the whole reason two engine families exist.

### 2.4 How column stores handle writes

Writing one row into a dozen compressed, sorted column files is expensive. Column stores borrow the LSM idea
(topic 19):

- New rows go into an in-memory, often row-oriented, write buffer (or a small "delta store").
- Periodically they're converted into a new immutable columnar segment (sometimes called a part, row group, or
  micro-partition) and written sequentially.
- Background processes merge small segments into larger ones (ClickHouse's MergeTree is named for this).
- Queries read both the buffer and the segments and combine them.
- Updates and deletes are often implemented as "mark deleted + write new version", with the actual cleanup at
  merge time; or by rewriting whole micro-partitions (Snowflake).

That's why many analytic systems are happiest with bulk loads and append-only data, and treat frequent
single-row updates as an anti-pattern.

### 2.5 Sort order

Since rows are stored in an order shared by all columns, you can choose that order. Sorting by, say,
`(order_date, region)`:
- A filter `WHERE order_date BETWEEN ...` reads one contiguous slice of each column instead of the whole thing.
- The first sort column becomes long runs of identical values, which compress extremely well (2.6).
- Segments carry **min/max metadata** ("zone maps"): this segment's `order_date` ranges from X to Y. A query can
  skip whole segments whose range can't match. This is data skipping without a traditional index.

Only one sort order per copy of the data is possible (like a clustered index, topic 18). Some systems (Vertica,
from the C-Store research) store several copies sorted differently; you need replicas anyway, so each can be
sorted to help a different query pattern.

### 2.6 Hybrid formats

Pure column-per-file storage is one end. Most real formats are **row groups of columns** (PAX layout): split the
table into chunks of, say, 1 million rows, and within each chunk store each column contiguously. Apache Parquet
and ORC work this way. You get the column benefits for scans while keeping a row's columns physically near each
other, and each row group carries per-column statistics for skipping.

---

## 3. Compression

### 3.1 Why columns compress so well

A column holds values of one type, often from a small domain, often repetitive: `region` has maybe 10 distinct
values across a billion rows; `status` has 5; `order_date` repeats each day's date thousands of times in a
sorted table. Row pages mix integers, strings, timestamps and floats, which compresses poorly. Columnar data
commonly compresses 5–10x, sometimes far more.

### 3.2 Encodings

**Dictionary encoding.** Replace each distinct value with a small integer code:

```
 region values:  EU, US, EU, EU, APAC, US, ...
 dictionary:     0=APAC, 1=EU, 2=US
 encoded:        1, 2, 1, 1, 0, 2, ...         (2 bits each instead of a string)
```

Filters can be evaluated on codes: `WHERE region = 'EU'` becomes "code == 1", an integer comparison.

**Run-length encoding (RLE).** Store (value, count) runs instead of repeated values. Extremely effective on
sorted columns:

```
 sorted region:  APAC×12,000,000  EU×41,000,000  US×47,000,000
 stored as:      (APAC, 12M), (EU, 41M), (US, 47M)    → three pairs for 100M rows
```

**Bitmap encoding.** For a low-cardinality column, keep one bitmap per distinct value, with bit i set if row i has
that value:

```
 region:   EU  US  EU  EU  APAC US
 EU   :     1   0   1   1   0    0
 US   :     0   1   0   0   0    1
 APAC :     0   0   0   0   1    0
```

Then `WHERE region IN ('EU','US') AND status = 'shipped'` becomes bitwise OR and AND over bitmaps, which CPUs do
64+ bits at a time. Sparse bitmaps are themselves run-length compressed (e.g. Roaring bitmaps).

**Delta encoding / frame of reference.** For sorted or slowly-changing numbers (timestamps, IDs), store the
difference from the previous value or a base: `1700000000, 1700000003, 1700000007` → base `1700000000` and
`0, 3, 7`, which fit in a few bits each (bit-packing).

General-purpose compression (LZ4, Zstd) is often layered on top.

### 3.3 Why compression makes queries faster, not just smaller

Analytic queries are usually bound by how fast bytes move from disk (or network, or memory) into the CPU.
If data is 8x smaller, scanning it reads 8x fewer bytes. Decompression costs CPU, but lightweight encodings
like dictionary, RLE and bit-packing are so cheap (and often the query operates on encoded data directly) that
it's a large net win. Smaller data also means more of it fits in memory caches.

---

## 4. Vectorized execution

### 4.1 The problem with row-at-a-time execution

Classic databases execute queries with the **Volcano / iterator model**: each operator (scan, filter, aggregate)
has a `next()` method that returns one row. For a billion rows, that's billions of virtual function calls per
operator, each processing one value, with branches the CPU can't predict and data scattered across memory. The
overhead per row dwarfs the actual work (adding a number).

### 4.2 Processing a batch at a time

Vectorized engines (pioneered by MonetDB/X100, later VectorWise; now in DuckDB, ClickHouse, Snowflake, Velox,
Databricks Photon) pass **batches of values from one column** (e.g. 1,024 or 4,096 values) between operators:

```
 Row-at-a-time:   for each row: f = filter(row); if f: agg.add(row.amount)       (call per row)

 Vectorized:      batch_region  = next 1024 region codes          (one array)
                  mask          = (batch_region == 1)             (one tight loop → bit mask)
                  batch_amount  = next 1024 amounts
                  sum          += masked_sum(batch_amount, mask)  (one tight loop)
```

Why this is fast:
- **Per-call overhead amortized** over 1,000+ values.
- **Tight loops over contiguous arrays** of one type: CPU caches and prefetchers love sequential memory.
- **SIMD** (single instruction, multiple data): one CPU instruction compares or adds 4, 8, or 16 values at once.
  Compilers can auto-vectorize simple loops over arrays.
- **Fewer branch mispredictions**: filters produce bitmasks or selection vectors instead of branching per row.

Columnar storage feeds this naturally: a column segment is already an array of one type. That's why columnar
storage and vectorized execution almost always come together.

An alternative to vectorization is **query compilation** (generating machine code specialized to the query,
as in HyPer/Umbra or Spark's whole-stage codegen). Both aim to remove interpretation overhead; some engines mix
them.

### 4.3 Putting it together: why analytic scans are 10–100x faster

For `SUM(amount) GROUP BY region` over a wide table, compared with a row store:
- Read 2 columns instead of 100 (≈50x fewer bytes).
- Those columns compress 5–10x more.
- Zone maps skip segments outside the date range.
- Vectorized, SIMD execution processes values far faster per CPU cycle.
- Analytic engines also parallelize scans across many cores and machines (MPP, massively parallel processing),
  and often separate storage (object storage, topic 53) from compute so you can scale compute for big queries.

The multiplied effect is why a query taking 20 minutes on Postgres can take seconds on a column store.

---

## 5. Data warehouses

### 5.1 What a warehouse is

A **data warehouse** is a separate database that holds a read-optimized copy of data from many operational
systems, with history, for analysis. The OLTP systems keep running the business; the warehouse answers
questions about it.

```
  Orders DB (Postgres) ──┐
  Payments service ──────┤
  CRM (SaaS) ────────────┼──▶  Extract ─▶ Transform ─▶ Load  ──▶  DATA WAREHOUSE  ──▶ BI dashboards
  Clickstream (Kafka) ───┤                                       (column store)       analysts, ML
  Support tickets ───────┘
```

### 5.2 ETL vs ELT

- **ETL (Extract, Transform, Load):** pull data from sources, clean and reshape it in a separate processing
  layer, then load the result into the warehouse. Traditional approach when warehouse compute was expensive.
- **ELT (Extract, Load, Transform):** load raw data into the warehouse first, then transform it there with SQL
  (tools like dbt). Popular now because cloud warehouses have cheap, elastic compute and keeping raw data lets
  you re-transform later.
- Extraction can be periodic batch dumps (nightly, topic 40) or continuous **change data capture** from the OLTP
  database's replication log (topic 39), feeding streams (topic 41) for near-real-time analytics.

Freshness is the main trade-off: a nightly batch warehouse is up to a day stale.

### 5.3 Data lakes and lakehouses (briefly)

A **data lake** stores raw files (often Parquet) in cheap object storage (topic 53) with no enforced schema
(schema-on-read, topic 15). Query engines (Spark, Trino, Athena) read the files directly. A **lakehouse** adds
table formats (Apache Iceberg, Delta Lake, Hudi) on top of those files, providing transactions, schema
evolution (topic 16), and snapshot reads, so the lake can act like a warehouse. The storage underneath is still
columnar files; the ideas in this topic apply unchanged.

---

## 6. Star and snowflake schemas

OLTP schemas are normalized (topic 15) to avoid update anomalies. Warehouses are organized around analysis,
usually as a **star schema** (also called dimensional modeling).

### 6.1 Fact tables

A **fact table** records events: one row per thing that happened (a sale, a page view, a payment). It holds:
- **Measures**: numbers you aggregate (`quantity`, `amount`, `discount`).
- **Foreign keys** to dimension tables.

Fact tables are huge (billions of rows), narrow-ish in practice but sometimes with 100+ columns, and append-only.

### 6.2 Dimension tables

**Dimension tables** describe the who, what, where, when, how of each event: `dim_customer`, `dim_product`,
`dim_store`, `dim_date`. They're much smaller and wide with descriptive attributes you filter and group by
(`product.category`, `customer.country`, `date.is_holiday`, `date.fiscal_quarter`).

```
                         ┌──────────────┐
                         │  dim_date    │
                         │ date_key PK  │
                         │ day, month,  │
                         │ quarter,     │
                         │ is_holiday   │
                         └──────┬───────┘
   ┌──────────────┐     ┌───────┴────────────┐     ┌──────────────┐
   │ dim_customer │     │    fact_sales      │     │ dim_product  │
   │ customer_key ├─────┤ date_key      FK   ├─────┤ product_key  │
   │ name, country│     │ customer_key  FK   │     │ name,        │
   │ segment      │     │ product_key   FK   │     │ category,    │
   └──────────────┘     │ store_key     FK   │     │ brand        │
                        │ quantity, amount   │     └──────────────┘
                        └───────┬────────────┘
                         ┌──────┴───────┐
                         │  dim_store   │
                         │ store_key PK │
                         │ city, region │
                         └──────────────┘
```

The picture looks like a star: the fact table at the center, dimensions radiating out. A typical query joins
the fact table with a few dimensions, filters on dimension attributes, and aggregates measures:

```sql
SELECT p.category, d.quarter, SUM(f.amount)
FROM   fact_sales f
JOIN   dim_product p ON f.product_key = p.product_key
JOIN   dim_date    d ON f.date_key    = d.date_key
WHERE  d.year = 2025 AND p.brand = 'Acme'
GROUP  BY p.category, d.quarter;
```

Engines optimize this pattern: filter the small dimensions first, build hash tables from them, and probe them
while scanning the fact table (a hash join, topic 18), often pushing the resulting key filters down into the
fact scan to skip segments.

### 6.3 Why a separate date dimension?

Because analysts filter and group by calendar concepts the raw timestamp doesn't carry: fiscal quarter, week of
year, holiday flags, day of week. Precomputing them once in a small table makes every query simpler.

### 6.4 Snowflake schema

A **snowflake schema** normalizes dimensions further: `dim_product` references `dim_brand`, which references
`dim_manufacturer`. Less redundancy, more joins. Star schemas are usually preferred for simplicity and query
speed; the redundancy in dimensions is cheap because they're small.

### 6.5 History: slowly changing dimensions

If a customer moves from Berlin to Paris, should last year's sales count under Berlin or Paris? Warehouses
handle this with **slowly changing dimension** techniques. The most common, "type 2", adds a new dimension row
with a new surrogate key and validity dates, so old facts keep pointing at the Berlin version and new facts at
the Paris version. This is something OLTP schemas, which keep only the current state, can't answer.

### 6.6 Wide denormalized tables

With columnar storage, a very wide table with dimension attributes copied into every fact row ("one big table")
is also common: unused columns cost nothing to scan, and compression absorbs the repetition. Joins are avoided
at the cost of harder updates to dimension attributes.

---

## 7. Materialized views and data cubes

### 7.1 Views vs materialized views

A normal (virtual) **view** is a saved query: every time you query it, the database runs the underlying query.
A **materialized view** actually stores the result on disk, like a cached table:

```sql
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT region, date_trunc('month', order_date) AS month, SUM(amount) AS revenue
FROM   orders
GROUP  BY region, month;
```

Querying `monthly_revenue` reads a few thousand precomputed rows instead of scanning a billion.

### 7.2 The cost: keeping it fresh

When the base data changes, the materialized view is stale. Options:
- **Full refresh:** recompute everything (Postgres's `REFRESH MATERIALIZED VIEW`). Simple, expensive, stale
  between refreshes.
- **Incremental maintenance:** apply only the changes (a new order adds its amount to one group's sum). Some
  warehouses do this automatically for supported query shapes; streaming systems (topic 41) are built around
  it. Aggregates like `SUM` and `COUNT` are easy to update incrementally; `MEDIAN` or `COUNT DISTINCT` are hard.
- Every write to the base table now has extra work, so materialized views shift cost from reads to writes,
  much like an index (topic 18), or a cache (topic 11) with its invalidation problem.

### 7.3 Data cubes (OLAP cubes)

A **data cube** is a materialized grid of aggregates over several dimensions:

```
                     product category →
                 ┌──────────┬──────────┬──────────┬────────┐
       date  ↓   │ Books    │ Toys     │ Garden   │ TOTAL  │
                 ├──────────┼──────────┼──────────┼────────┤
     2025-01     │  12,400  │   8,100  │   3,300  │ 23,800 │
     2025-02     │  11,900  │   9,700  │   4,100  │ 25,700 │
     TOTAL       │  24,300  │  17,800  │   7,400  │ 49,500 │
                 └──────────┴──────────┴──────────┴────────┘
     (with a third dimension, e.g. region, the grid becomes a cube; more → hypercube)
```

Each cell is precomputed, so "total sales of Toys in February" is a lookup, and rolling up (summing a row or
column) or drilling down is instant. The cost: you can only answer questions along the predefined dimensions;
any filter on an attribute that isn't a cube dimension needs the raw data. And the number of cells grows
multiplicatively with dimensions. Modern columnar engines are fast enough that cubes are less central than they
were, but the idea lives on in rollup tables and pre-aggregations (Druid, Pinot, ClickHouse's aggregating
merge trees).

---

## 8. Blurring the line: HTAP

Some systems try to serve both workloads (HTAP, hybrid transactional/analytical processing): SQL Server and
Oracle have in-memory columnar copies of row tables; TiDB keeps row-store replicas (TiKV) plus columnar
replicas (TiFlash) fed by replication; SingleStore combines both formats. The underlying insight is the same as
in this topic: the two workloads want different physical layouts, so HTAP systems typically keep **two layouts
of the same data** and route each query to the appropriate one. The difficulty is keeping them consistent and
isolating the workloads' resource use.

---

## 9. How it connects

- **Topic 15 (data models):** normalization for OLTP vs denormalized star schemas and wide tables for OLAP;
  schema-on-read in data lakes.
- **Topic 16 (encoding):** Parquet/ORC are columnar file encodings; schema evolution applies to warehouse tables.
- **Topics 17 & 19 (B-trees, LSM):** row engines for OLTP; column stores borrow LSM's buffer-then-merge write path.
- **Topic 18 (indexes & query planning):** zone maps and sort keys replace many secondary indexes; hash joins
  dominate star-schema queries; materialized views trade write cost for read speed like indexes.
- **Topic 21–22 (transactions, concurrency):** long analytic queries on an OLTP database hold old snapshots and
  delay vacuum.
- **Topic 23 (replication):** read replicas offload some analytics, and replication logs feed CDC.
- **Topics 39–41 (CDC, batch, stream):** how data gets from OLTP into the warehouse, and how materialized views
  are maintained incrementally.
- **Topic 53 (object storage):** where lakehouse and cloud warehouse data physically lives.

---

## Common misconceptions

- "Column stores are just faster databases." → They're faster for scans over few columns and much slower for
  single-row lookups, inserts and updates. Using one as an OLTP database is a mistake.
- "Column-oriented means the same as wide-column (Cassandra, Bigtable)." → Wide-column stores group data by row
  key and column families; within a partition data is stored row by row. They are OLTP-style systems. Column
  stores store each column's values contiguously for analytics.
- "Compression trades speed for space." → In analytics, compression usually makes queries faster because they
  are bound by I/O and memory bandwidth, and lightweight encodings can be processed without full decompression.
- "Analytics on a read replica is the same as a warehouse." → A replica isolates load but keeps the row layout,
  OLTP indexes, current-state-only data, and a single source.
- "Materialized views are free speed." → They cost storage, write-time maintenance, and staleness, exactly like
  caches and indexes.
- "Star schemas are obsolete now that storage is cheap." → Dimensional modeling is about making data
  understandable and queryable; it's still the dominant warehouse design, even if wide tables are also common.

## Further reading (supplements)

- *Designing Data-Intensive Applications*, Ch. 3 ("Transaction Processing or Analytics?", "Column-Oriented
  Storage"). (The 2nd edition expands this material.)
- Stonebraker et al., "C-Store: A Column-oriented DBMS" (VLDB 2005).
- Boncz, Zukowski, Nes, "MonetDB/X100: Hyper-Pipelining Query Execution" (CIDR 2005): vectorized execution.
- Abadi, Boncz, Harizopoulos, "The Design and Implementation of Modern Column-Oriented Database Systems"
  (Foundations and Trends in Databases, 2013).
- Kimball & Ross, *The Data Warehouse Toolkit*: dimensional modeling and star schemas.
- Dageville et al., "The Snowflake Elastic Data Warehouse" (SIGMOD 2016).
- Apache Parquet documentation: file format and encodings.

---

## Questions

1. Why does a query reading 3 of 100 columns run so much faster on a column store than on a row store?
2. Why are single-row inserts and point lookups slow on a column store? How do real column stores mitigate
   the insert cost?
3. In a column store, how does the engine know which values from different column files belong to the same row?
4. Why does sorting the table by `(date, region)` improve both compression and query speed? Why can you only
   pick one sort order per copy?
5. Explain dictionary, run-length, and bitmap encoding, and describe a column where each would work well.
6. Why can compression make a scan *faster*, even though decompression costs CPU?
7. What is the overhead of row-at-a-time (Volcano) execution, and how does vectorized execution remove it?
8. An analyst runs a 10-minute aggregation directly on the production Postgres primary at noon. List the ways
   this can hurt customers.
9. What's the difference between a fact table and a dimension table? Give an example of each for a shopping
   site.
10. A customer moves country. How should a warehouse represent this so that last year's sales stay attributed
    to the old country?
11. A materialized view of daily revenue is refreshed nightly. What are the costs and limitations? What changes
    if you need it up to the minute?
12. What's the difference between ETL and ELT, and why has ELT become popular?
13. Why is Cassandra (a "wide-column" store) not a column store in this topic's sense?
14. Would you build a cube for "revenue by category by month by region"? What questions could it not answer?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** The row store must read every page of the table, which contains all 100 columns for every row; the column
store reads only the three column files, about 3% of the data. On top of that, those columns compress better,
segments can be skipped via min/max metadata, and execution can be vectorized over the column arrays.

**2.** A point lookup needs the i-th value from each column file, which is a separate read per column, possibly
requiring decompressing a whole segment for one value. An insert must append a value to every column file and
disrupts compression and sort order. Mitigation: buffer new rows in memory (or a row-oriented delta store),
flush them in batches as new immutable columnar segments, and merge segments in the background, the same idea
as an LSM-tree. Updates are typically delete-mark plus insert.

**3.** Position. All column files store values in the same row order, so the i-th entry in each column belongs to
row i. Reconstructing a row means reading the i-th entry from each column.

**4.** Sorted columns have long runs of equal values (especially the first sort key), which run-length encoding
collapses dramatically. Filters on the sort key read one contiguous slice, and min/max zone maps let the engine
skip whole segments. Only one order per copy is possible because the row order is shared by all columns;
reordering one column means reordering all of them. Systems that want more keep multiple copies (replicas) with
different sort orders.

**5.** Dictionary: replace each distinct value with a small integer code; good for strings with moderate
cardinality, e.g. `country` or `product_category`. RLE: store (value, run length); good for sorted or slowly
changing columns, e.g. `order_date` in a date-sorted table. Bitmap: one bit vector per distinct value; good for
very low cardinality columns used in filters, e.g. `status` (5 values) or `is_returned`, since filters become
bitwise AND/OR.

**6.** Analytic scans are typically bound by bytes moved (disk, network, memory bandwidth) rather than CPU. If data
is 8x smaller, 8x fewer bytes are moved. Lightweight decodings are very cheap per value, and many operations
(filter on dictionary code, sum over RLE runs as value×count) run directly on the encoded data. More data also
fits in caches.

**7.** Each operator's `next()` call returns one row, so a billion rows mean billions of virtual calls, per-row
branching, and poor cache behavior; overhead dominates the useful work. Vectorized execution passes batches of
~1,000 values of one column between operators; each operator runs a tight loop over a contiguous array, which
amortizes call overhead, uses CPU caches well, enables SIMD, and replaces branches with bitmasks.

**8.** It competes for CPU and disk I/O; its large scan evicts hot pages from the buffer pool so customer queries
start hitting disk; it may take locks or hold an old snapshot that prevents vacuum from cleaning dead rows
(bloat, topic 22); on a busy primary it can raise latency for every request and in the worst case trigger
timeouts and retries (topic 43). It also may produce inconsistent results if it runs at a weak isolation level
while data is changing (topic 21).

**9.** A fact table records events with measures and foreign keys, e.g. `fact_sales` (one row per line item sold:
quantity, amount, discount, plus date_key, product_key, customer_key). A dimension table describes an entity
that facts refer to, e.g. `dim_product` (name, brand, category, price tier) or `dim_date` (day, month,
fiscal quarter, holiday flag).

**10.** Use a slowly changing dimension (type 2): instead of overwriting the customer row, close the old row (set
its valid_to date) and insert a new row with a new surrogate key and the new country. Facts from last year point
to the old surrogate key and stay attributed to the old country; new facts point to the new one.

**11.** Costs: storage, a refresh job that rescans the base data nightly (expensive and growing), and data up to a
day stale. Limitations: only answers the query it was defined for. For up-to-the-minute freshness you need
incremental maintenance: process each change (e.g. via CDC and stream processing, topics 39 and 41) and update
the affected aggregates, which works well for SUM/COUNT but is hard for MEDIAN or COUNT DISTINCT, and adds work
to the write path.

**12.** ETL transforms data in a separate processing step before loading; ELT loads raw data into the warehouse and
transforms with SQL inside it. ELT is popular because cloud warehouses have elastic, cheap compute, keeping raw
data allows re-transforming when business logic changes, and SQL-based transformation tooling (e.g. dbt) is
easy to version and test.

**13.** Cassandra stores data partitioned by a row key, and within a partition rows (with their columns) are
stored together, sorted by clustering key. Access is by key and the layout favors reading whole rows within a
partition, which is OLTP-style. A column store stores all values of one column for many rows contiguously to
scan them for analytics. The naming overlap is unfortunate.

**14.** If those are the dimensions analysts constantly slice by, a cube (or rollup table) gives instant answers
and drill-downs. It could not answer anything involving an attribute that isn't a cube dimension (e.g. revenue
by customer segment, by brand, or by payment method), anything needing finer granularity (per day, per
product), or non-additive measures not precomputed (distinct customers). Those need the underlying fact table.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** OLTP access pattern in one line?
**A:** Many small reads and writes of a few rows by key, low latency, transactional.

**Q:** OLAP access pattern in one line?
**A:** Few big queries scanning many rows and few columns, computing aggregates.

**Q:** What is the bottleneck for OLTP vs OLAP?
**A:** OLTP: latency per request (seeks, locks). OLAP: bandwidth (bytes scanned per second) and CPU per row.

**Q:** Row store layout?
**A:** All values of a row stored together on a page.

**Q:** Column store layout?
**A:** All values of each column stored contiguously; rows aligned by position across column files.

**Q:** What's cheap and expensive in a column store?
**A:** Cheap: scanning a few columns over many rows. Expensive: point lookups, single-row inserts and updates.

**Q:** How do column stores ingest writes efficiently?
**A:** Buffer rows in memory, write immutable columnar segments in batches, merge them in the background (LSM-style).

**Q:** What are zone maps?
**A:** Per-segment min/max values per column, letting queries skip segments that can't match.

**Q:** Dictionary encoding?
**A:** Replace distinct values with small integer codes; filters compare codes.

**Q:** Run-length encoding?
**A:** Store (value, count) runs; great for sorted columns.

**Q:** Bitmap encoding?
**A:** One bit vector per distinct value; filters become bitwise AND/OR.

**Q:** Why does compression speed up scans?
**A:** Scans are I/O/bandwidth bound; fewer bytes moved outweighs cheap decoding.

**Q:** What is the Volcano model's problem for analytics?
**A:** One function call per row per operator; overhead dominates the actual computation.

**Q:** What is vectorized execution?
**A:** Operators process batches (~1,000 values) of a column in tight loops, using caches and SIMD.

**Q:** What is PAX / row groups (Parquet, ORC)?
**A:** Split the table into row chunks; within each chunk store columns contiguously, with per-column stats.

**Q:** What is a data warehouse?
**A:** A separate, read-optimized (usually columnar) database holding historical data from many sources for analysis.

**Q:** ETL vs ELT?
**A:** ETL transforms before loading; ELT loads raw data and transforms inside the warehouse.

**Q:** Fact table?
**A:** One row per event, with numeric measures and foreign keys to dimensions. Huge, append-only.

**Q:** Dimension table?
**A:** Descriptive attributes (who/what/where/when) to filter and group by. Small and wide.

**Q:** Star vs snowflake schema?
**A:** Star: fact table with denormalized dimensions. Snowflake: dimensions further normalized into sub-dimensions.

**Q:** What is a slowly changing dimension (type 2)?
**A:** A dimension change adds a new row with validity dates instead of overwriting, preserving history.

**Q:** Materialized view?
**A:** A query result stored on disk and refreshed when base data changes.

**Q:** What does a materialized view cost?
**A:** Storage, maintenance work on writes or refreshes, and staleness.

**Q:** What is a data cube?
**A:** A precomputed grid of aggregates across several dimensions, enabling instant roll-up and drill-down.

**Q:** Limitation of a cube?
**A:** Can only answer questions along its predefined dimensions and granularity.

**Q:** Is Cassandra a column store?
**A:** No; it's a wide-column (row-key partitioned) OLTP store, not columnar analytics storage.

**Q:** What is HTAP?
**A:** Systems serving both OLTP and OLAP, typically by keeping row and columnar copies of the same data.
