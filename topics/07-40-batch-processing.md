# 40 — Batch processing

> Part 7 — Moving Data Between Systems · Prerequisites: topics 02, 13, 20, 25, 37

Most of this course has been about *online* systems: a request arrives, and the system must answer within
milliseconds. A different kind of system takes a large, bounded input — yesterday's logs, a full
database export, a month of orders — runs a computation over all of it, and produces an output. Nobody
is waiting on a socket; what matters is **throughput** (how much data per hour) and that the job
finishes correctly. This is **batch processing**. It's how search indexes, recommendation models,
billing reports, analytics tables and machine-learning features are traditionally built. This topic
starts from the Unix philosophy that inspired it, builds up MapReduce and the distributed filesystem it
runs on, explains how joins are done when data doesn't fit on one machine, and ends with modern dataflow
engines like Spark that fixed MapReduce's biggest inefficiencies.

By the end of this topic you should be able to:
1. Explain the Unix pipeline model and which of its properties carry over to distributed batch processing.
2. Walk through a MapReduce job — input splits, map, partition, sort/shuffle, reduce, output — and explain "moving computation to data".
3. Describe how GFS/HDFS store data (blocks, replication, NameNode) and why they suit batch workloads.
4. Explain sort-merge joins, broadcast joins and partitioned hash joins, and when each applies, including skew.
5. Explain how dataflow engines (Spark) improve on MapReduce: DAGs, in-memory intermediate data, narrow vs wide dependencies, and lineage-based recovery.
6. Explain why batch outputs should be immutable and jobs idempotent, and how that makes failures cheap.

---

## 1. Three kinds of systems

Kleppmann's framing (DDIA ch. 10) is a useful map:

| | Services (online) | Batch processing (offline) | Stream processing (near-real-time) |
|---|---|---|---|
| Input | Requests | Bounded dataset of known size | Unbounded stream of events |
| Measured by | Response time, availability | Throughput, job duration | Latency from event to result |
| Runs | Continuously, per request | Periodically (hourly, nightly) | Continuously |
| Example | Checkout API | Nightly sales report | Fraud alert within seconds |

This topic is the middle column; topic 41 is the right-hand one. The key property of batch input is that
it's **bounded**: the job knows when it has seen everything, so it can sort the whole input, compute
exact totals, and then finish.

---

## 2. The Unix philosophy as a data-processing model

### 2.1 A log analysis pipeline

Find the five most requested URLs in a web server log:

```bash
cat access.log |
  awk '{print $7}' |      # extract the URL field
  sort |                  # bring identical URLs together
  uniq -c |               # count each run of identical lines
  sort -rn |              # sort by count, descending
  head -n 5               # top five
```

Each tool does one thing, reading from stdin and writing to stdout. This tiny pipeline already contains
the essential shape of a distributed batch job:

```
extract a key (awk)  →  group identical keys together (sort)  →  aggregate each group (uniq -c)
     "map"                         "shuffle/sort"                     "reduce"
```

### 2.2 Why `sort` instead of a hash map?

You could write a script that keeps a hash map of URL → count in memory. That's faster *if the distinct
URLs fit in RAM*. GNU `sort` instead handles inputs larger than memory: it sorts chunks in memory, spills
sorted runs to disk, and merges them — an external merge sort using sequential I/O (topic 02), the same
strategy as LSM-tree compaction (topic 19). Sorting is the scalable way to bring related records together
when they don't fit in memory. MapReduce inherits this.

### 2.3 The principles that carry over

1. **Uniform interface.** Every program reads and writes the same thing (a stream of lines/bytes), so
   any program can be connected to any other. In distributed batch, the uniform interface is *files in a
   distributed filesystem*.
2. **Separation of logic from wiring.** A program doesn't know where its input comes from or where its
   output goes; the shell wires them. In MapReduce, the framework wires map and reduce tasks together.
3. **Immutable inputs.** Running the pipeline doesn't modify `access.log`; you can rerun with tweaks as
   often as you like. Experimentation is cheap and safe.
4. **Composable, single-purpose stages.** Each stage is simple; complex jobs are chains of stages.
5. **Inspect intermediate output.** Pipe to `less` at any stage to debug.

The big limitation: Unix pipes run on one machine. MapReduce is, roughly, "Unix pipelines distributed
across thousands of machines".

---

## 3. Distributed filesystems: the foundation

### 3.1 Why a special filesystem?

A batch job reading 100 TB can't pull it through one disk or one network link. It needs the data spread
across many machines, read in parallel — ideally from local disks, without crossing the network.

### 3.2 GFS and HDFS

The Google File System (2003) and its open-source counterpart, the **Hadoop Distributed File System
(HDFS)**, share a design:

```
                       ┌──────────────────────────┐
                       │  NameNode (GFS master)   │  metadata only:
                       │  /logs/day1 → blocks     │  file → block IDs,
                       │  b1@{n1,n4,n7} b2@{...}  │  block → DataNodes
                       └──────────┬───────────────┘
                 metadata lookups │
   client ────────────────────────┘
      │  read/write block data directly
      ▼
 ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ...
 │DataNode│ │DataNode│ │DataNode│ │DataNode│ │DataNode│
 │ b1 b5  │ │ b2 b3  │ │ b2 b6  │ │ b1 b3  │ │ b5 b6  │
 └────────┘ └────────┘ └────────┘ └────────┘ └────────┘
```

- Files are split into large **blocks** (64–128 MB, vs 4 KB in a local filesystem). Big blocks keep
  metadata small and favour long sequential reads.
- Each block is **replicated** (default 3 copies) on different machines and racks, so a machine failure
  loses no data. (Modern deployments also use erasure coding for cold data, topic 53.)
- A central **NameNode** holds metadata (namespace, block locations) in memory; it's not on the data
  path. Clients talk to DataNodes directly for bytes. (The NameNode was a single point of failure in
  early HDFS; later versions run an active/standby pair.)
- **Write-once, append-only, read-many.** Files aren't updated in place. That fits batch outputs and
  simplifies replication and consistency enormously.
- Optimised for **throughput over latency**: great at scanning a 1 TB file, poor at millions of tiny
  files or random small reads.

Today, much batch processing reads from **object storage** (S3, GCS — topic 53) instead of HDFS. The
interface is similar (big immutable objects), but storage and compute are separated, so you lose data
locality and gain elasticity — the network in modern datacentres is fast enough that this is usually a
good trade.

### 3.3 Moving computation to data

The crucial idea: shipping a 100 KB program to the machine holding a 128 MB block is far cheaper than
shipping the block to the program. The scheduler asks the NameNode where each block's replicas are and
runs the task **on one of those machines** (or at least in the same rack). This **data locality** made
early clusters, with slow 1 Gb networks, practical.

---

## 4. MapReduce

### 4.1 The programming model

You write two functions:

```
map(key, value)            → list of (k2, v2)        # called once per input record
reduce(k2, list of v2)     → list of (k3, v3)        # called once per distinct k2
```

The framework guarantees that **all values for the same k2 are brought together** to one reduce call.

Word count:

```
map(_, line):
    for word in line.split(): emit(word, 1)

reduce(word, counts):
    emit(word, sum(counts))
```

### 4.2 Execution step by step

```
 HDFS input (blocks)
 ┌──────┐ ┌──────┐ ┌──────┐
 │split1│ │split2│ │split3│
 └──┬───┘ └──┬───┘ └──┬───┘
    ▼        ▼        ▼
 ┌──────┐ ┌──────┐ ┌──────┐   MAP tasks (one per split, run where the block lives)
 │ map  │ │ map  │ │ map  │   emit (k,v); partition by hash(k) mod R;
 └──┬───┘ └──┬───┘ └──┬───┘   sort each partition in memory, spill sorted runs to LOCAL disk
    │╲      ╱│╲      ╱│
    │ ╲    ╱ │ ╲    ╱ │       SHUFFLE: each reducer fetches its partition from every mapper
    │  ╲  ╱  │  ╲  ╱  │
    ▼   ╲╱   ▼   ╲╱   ▼
 ┌────────┐      ┌────────┐   REDUCE tasks (R of them)
 │reduce 0│      │reduce 1│   merge the sorted runs → each key's values arrive together
 └───┬────┘      └───┬────┘   call reduce(k, values) per key
     ▼               ▼
  part-00000      part-00001   output files in HDFS (replicated)
```

1. **Input splits.** The input is divided into splits (usually one per block). One map task per split.
2. **Map.** Each map task reads its split record by record and calls `map`. Output pairs are
   partitioned by `hash(key) mod R` (R = number of reducers) — the same hash partitioning as topic 25.
3. **Sort and spill.** Within each partition, the mapper sorts pairs by key, spilling sorted runs to
   **local disk** (not HDFS — it's temporary).
4. **Optional combiner.** A "mini-reduce" on the mapper's output (e.g. pre-summing word counts) shrinks
   the data shipped across the network. Only valid for associative, commutative operations.
5. **Shuffle.** Each reducer pulls its partition from every mapper's local disk. This is the expensive,
   all-to-all network step.
6. **Merge.** The reducer merge-sorts the incoming sorted runs (like `sort`'s merge phase), so all values
   for a key are adjacent.
7. **Reduce.** For each key, call `reduce(key, iterator over values)` — streaming, so a key's values
   needn't fit in memory.
8. **Output.** Results are written to HDFS, replicated, as one file per reducer.

### 4.3 Fault tolerance

At thousands of machines, something fails in nearly every large job. MapReduce handles it simply:
- A failed (or very slow) **task is re-run** on another machine. Map and reduce functions must be
  **deterministic and side-effect-free** (beyond their output) for this to be correct.
- Map outputs on a dead machine's local disk are lost, so completed map tasks on that machine are re-run.
- **Speculative execution:** near the end of a job, slow tasks ("stragglers", often due to a bad disk) are
  duplicated; whichever copy finishes first wins. (Tail latency again — topic 02.)
- Output is written to a temporary location and **atomically renamed** on task completion, so partial
  output from a failed attempt is never visible.

Because inputs are immutable and outputs are only committed on success, **a failed job leaves no
partial state** — you just rerun it. That's the batch world's great simplicity.

### 4.4 Workflows

Real pipelines are chains of jobs — job 1's output directory is job 2's input — orchestrated by a
workflow scheduler (Oozie and Airflow historically; Airflow, Dagster and similar today). Each
intermediate dataset is **materialised** to HDFS: written in full, replicated three times, before the
next job can start. Section 6 explains why that's costly.

### 4.5 Weaknesses of raw MapReduce

- Every job is exactly map → shuffle → reduce; complex logic needs many chained jobs.
- Intermediate results between jobs are written to and replicated in HDFS (heavy I/O).
- A job can't start until the previous job has completely finished.
- Mappers often just re-read what the previous reducer wrote.
- The API is low-level; higher-level languages (Pig, Hive SQL) were built on top to compile into
  MapReduce jobs.

---

## 5. Joins at scale

### 5.1 Why joins are the hard part

Joins in batch processing usually mean: enrich a large table of events with attributes from another
dataset. Example: page-view events (10 TB, `user_id, url, ts`) joined with user profiles (200 GB,
`user_id, country, birth_year`) to count views by country and age band.

In an OLTP database you'd do an index lookup per event (topic 18). In batch, doing 10 billion random
lookups to a remote database would be slow, overload that database, and be non-deterministic (the DB
changes during the job). Instead, take a **snapshot** of the profiles into the distributed filesystem
and join the two datasets as bulk files.

### 5.2 Sort-merge join (reduce-side join)

Both inputs go through mappers that emit the **join key** as the key:

```
page views mapper:  (user_id, ("view", url, ts))
profiles mapper:    (user_id, ("profile", country, birth_year))

shuffle by user_id → each reducer gets, for one user_id:
   [ ("profile", "DE", 1990), ("view", "/a", t1), ("view", "/b", t2), ... ]
```

With a **secondary sort** the profile record arrives *first*, so the reducer reads it, keeps it in a
variable, and streams through the views emitting enriched records — needing only one profile in memory
at a time.

- Works for any sizes, makes no assumptions about input layout.
- Cost: both datasets are shuffled across the network and sorted.
- Bringing all records with the same key to the same place is exactly what the shuffle does — "group by"
  and "join" are both just "bring related data together".

### 5.3 Skew: hot keys

If one key is huge (a celebrity user with 100 million views, a "null" user_id for anonymous traffic), the
single reducer for that key does a disproportionate share of work and the whole job waits for it — the
hot-key problem from topic 25. Fixes:
- **Skewed join / salting:** detect hot keys (by sampling), spread their records across several reducers
  by adding a random suffix, and replicate the matching record from the other side to all of those
  reducers.
- **Two-stage aggregation:** for group-by, aggregate per (key, random salt) first, then combine.
- Spark's **adaptive query execution** can split skewed partitions automatically at runtime.

### 5.4 Broadcast hash join (map-side join)

If one input is **small enough to fit in each worker's memory** (e.g. a 50 MB country-code table, or even
a few GB on big machines):

```
every mapper:
   load small table into an in-memory hash map   (broadcast to all workers)
   stream through its split of the big table:
       look up join key in hash map → emit joined record
```

- **No shuffle and no sort** of the big table — dramatically cheaper.
- Constraint: the small side must fit in memory on every worker.
- Query engines choose this automatically when statistics say one side is small (topic 18's cost-based
  planning, distributed).

### 5.5 Partitioned hash join (bucketed map-side join)

If **both** inputs are large but were **already partitioned the same way** — same key, same hash
function, same number of partitions (e.g. both written by earlier jobs bucketed by `user_id` into 256
files) — then partition *i* of one only needs to be joined with partition *i* of the other:

```
views/part-017   ⨝   profiles/part-017      ← one mapper; profiles part-017 fits in memory
views/part-018   ⨝   profiles/part-018
...
```

Each mapper loads one partition of the smaller side into a hash table and streams the matching
partition of the larger side. No shuffle. Hive calls this a *bucketed map join*. The cost is planning:
the data must have been laid out deliberately in advance. (If both partitions are also sorted by the
key, a **map-side merge join** works without even needing memory for a hash table.)

### 5.6 Choosing

| Join | Requirement | Shuffle? | Best when |
|---|---|---|---|
| Sort-merge (reduce-side) | None | Both sides | Both big, no prior layout |
| Broadcast hash | One side fits in memory | None | Big ⨝ small |
| Partitioned hash | Both sides co-partitioned on the key | None | Both big, laid out in advance |

The same three strategies appear in distributed SQL engines and data warehouses (topic 20) as
"shuffle join", "broadcast join" and "co-located/bucketed join".

---

## 6. Dataflow engines: Spark and friends

### 6.1 What was wrong with materialising everything

A multi-step pipeline in MapReduce writes every intermediate result to HDFS with 3× replication, waits
for each job to finish completely, and then reads it back. Much of that I/O protects data that's only
needed for a few minutes and could be recomputed.

**Dataflow engines** — Apache Spark, Apache Flink (in batch mode), Apache Tez — treat the whole workflow
as a single job: a **directed acyclic graph (DAG)** of operators.

### 6.2 The DAG model

```
read views ─► filter(bots) ─► map(extract) ─┐
                                             ├─► join on user_id ─► groupBy(country, age) ─► count ─► write
read profiles ──────────────────────────────┘
```

- **Operators** are general (map, filter, join, groupBy, aggregate, window…), not just map and reduce.
- The engine sees the *whole* graph, so it can **optimise** it: pipeline consecutive narrow operators
  in one task, pick join strategies, push filters down, avoid unnecessary sorts (sort only when needed,
  hash-based grouping otherwise).
- Spark's SQL/DataFrame layer (Catalyst optimiser) turns declarative queries into such plans —
  the batch world rediscovering the relational query planner (topic 18).

### 6.3 Narrow vs wide dependencies

Spark divides the DAG into **stages** at shuffle boundaries:

- **Narrow dependency:** each output partition depends on a small, fixed set of input partitions
  (map, filter, union, co-partitioned join). These can be pipelined in memory within one task — no
  network.
- **Wide dependency:** each output partition may depend on *all* input partitions (groupBy, reduceByKey,
  non-co-partitioned join). Requires a **shuffle**: data is repartitioned across the network.

```
stage 1 (narrow, pipelined)           stage 2 (after shuffle)
[read → filter → map]  ──shuffle──►  [groupBy → count]  ──► write
```

Shuffles are where the cost is — network, disk, serialisation — so a good plan minimises them.

### 6.4 In-memory intermediate state

Within a job, intermediate data is kept **in memory** (spilling to local disk when needed) instead of
being written to the distributed filesystem with replication. Shuffle outputs go to local disk on the
producing worker, not HDFS. Datasets reused several times (e.g. by iterative machine-learning
algorithms that loop over the same data 50 times) can be **cached in memory** explicitly. This is the
main source of Spark's speedups over MapReduce — often an order of magnitude for iterative workloads.

### 6.5 Lineage-based recovery

If intermediate data isn't replicated, what happens when a machine dies and its in-memory partitions
vanish? Spark's answer is **lineage**: each dataset (RDD — Resilient Distributed Dataset) remembers
*how it was computed* from its parents. A lost partition is simply **recomputed** from its inputs.

```
lost partition p7 of "joined"
  └─ recompute from p7 of "filtered views" and p7 of "profiles"
        └─ which recompute from the input blocks in HDFS (still replicated)
```

- For **narrow** dependencies, recovering one partition requires recomputing just its few parents —
  cheap.
- For **wide** dependencies, a lost partition may need data from *all* parent partitions; recomputing
  could cascade. Spark keeps shuffle output files on local disk partly so it doesn't have to go back
  to the beginning; **checkpointing** an RDD to reliable storage cuts long lineage chains.
- Recomputation must give the same answer, so operators must be **deterministic**. Non-determinism
  (random numbers without a seed, iterating a hash map whose order varies, reading the current time,
  querying external systems) can make recomputed partitions inconsistent with ones already consumed.

Trade-off summary: MapReduce pays to *replicate* intermediate data so it never recomputes; dataflow
engines skip the replication and pay to *recompute* only when something fails. Since failures are rare
relative to the work done, this wins.

### 6.6 Beyond Spark

- **Flink** treats batch as a special case of streaming (bounded streams) — topic 41.
- **Cloud data warehouses / MPP query engines** (BigQuery, Snowflake, Trino, Redshift) run SQL over
  columnar files (topic 20) with the same ideas: partitioned scans, shuffle/broadcast joins, in-memory
  pipelines.
- File formats matter: columnar **Parquet**/**ORC** let jobs read only needed columns and skip row
  groups by min/max statistics; table formats like Apache Iceberg and Delta Lake add atomic commits and
  snapshots on top of object storage.

---

## 7. Outputs and the batch mindset

### 7.1 What batch jobs produce

- **Derived datasets for analytics** (warehouse tables, reports).
- **Search indexes:** Google's original use of MapReduce — build index files in batch, then ship them to
  serving machines.
- **Key-value stores for serving:** e.g. precomputed recommendations per user, built as immutable
  sorted files and bulk-loaded into a serving database.
- **Machine-learning features and models.**

### 7.2 Don't write to the production database from inside the job

A tempting approach: mappers or reducers write each result directly to the serving database via its
client. Problems:
- Thousands of parallel tasks hammer the database (the same overload as topic 36's recovery stampede).
- Tasks are retried and speculatively executed → duplicate or partial writes; the job is no longer
  all-or-nothing.
- Readers see a half-updated database while the job runs.

Better: write a **new, immutable output** (files, or a new database/index version) and **atomically
switch** readers to it when the job succeeds. Rolling back means switching back.

### 7.3 Immutable inputs, idempotent jobs

Because inputs aren't modified and outputs are replaced as a whole:
- A buggy job can be fixed and re-run; the previous output can be restored. Kleppmann calls this
  **human fault tolerance** — recovering from bad code, not just bad hardware.
- Re-running a job on the same input gives the same output: jobs are **idempotent** (topic 38) at the
  dataset level. Partitioning outputs by date (`/sales/date=2026-10-14/`) and *overwriting* a partition
  makes daily jobs safely re-runnable and backfillable.

### 7.4 Limits of batch

The big one is **latency**: results are as stale as the last run, and a day's data can't be processed
until the day is over. Shrinking the batch interval (hourly, every 5 minutes) eventually becomes stream
processing — which processes each event as it arrives over an unbounded input. That's topic 41.

---

## 8. How it connects

- **Topic 02:** sequential I/O, external sorting, stragglers as tail latency.
- **Topic 13/25:** hash partitioning in the shuffle; hot keys as skew.
- **Topic 18:** join strategies and cost-based planning reappear at cluster scale.
- **Topic 19:** external merge sort, like LSM compaction.
- **Topic 20:** columnar formats and warehouses are where batch output often lives.
- **Topic 37:** Kafka topics are a common batch *input* (dumped to object storage) and the bridge to streaming.
- **Topic 38:** deterministic, idempotent jobs; replay = rerun.
- **Topic 39:** CDC streams land in the lake/warehouse for batch processing.
- **Topic 41:** stream processing, the unbounded counterpart; Lambda/Kappa architectures combine both.
- **Topic 53:** object storage and erasure coding underneath modern batch.

---

## Common misconceptions

- **"MapReduce is obsolete, so its ideas are too."** → The model (partition, shuffle, sort, reduce; moving
  computation to data; deterministic re-execution) underlies Spark, warehouses and stream processors.
- **"Batch is just a slow version of streaming."** → It's simpler and stronger for some jobs: bounded
  input allows exact totals, global sorts and trivially re-runnable outputs.
- **"Joins require random lookups."** → At scale they're done by co-locating both sides via shuffle,
  broadcast, or pre-partitioning — all sequential.
- **"Spark is fast because it's in memory, full stop."** → It's fast because it avoids materialising and
  replicating intermediate data, pipelines narrow operators, and optimises the whole DAG — while
  shuffles still hit disk and network.
- **"Lineage means nothing is ever lost."** → Recovery depends on deterministic operators and on the
  original inputs still existing; long lineage chains need checkpoints.
- **"The job can just write results into the live database."** → Retries and speculative execution make
  that non-atomic and it overloads the DB; build immutable outputs and swap.

## Further reading (supplements)

- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 10 ("Batch Processing").
- Dean & Ghemawat, "MapReduce: Simplified Data Processing on Large Clusters" (OSDI 2004).
- Ghemawat, Gobioff & Leung, "The Google File System" (SOSP 2003).
- Shvachko et al., "The Hadoop Distributed File System" (MSST 2010).
- Zaharia et al., "Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster
  Computing" (NSDI 2012).
- Doug McIlroy et al., "UNIX Time-Sharing System: Foreword" (Bell System Technical Journal, 1978) — the
  Unix philosophy in its authors' words.

---

## Questions

1. In the `awk | sort | uniq -c | sort -rn | head` pipeline, which stage corresponds to the MapReduce
   shuffle, and why does `sort` scale where an in-memory hash map may not?
2. Why are HDFS blocks 64–128 MB rather than 4 KB? What workload does that make HDFS bad at?
3. Explain "moving computation to data". Why does it matter less on modern clusters reading from object
   storage?
4. In a MapReduce job, why are map outputs written to local disk rather than HDFS? What happens if that
   machine dies after its map tasks complete?
5. What's a combiner, and why can it be used for `sum` but not for `median`?
6. Why must map and reduce functions be deterministic and free of external side effects?
7. You join 10 TB of orders with a 30 MB product-category table. Which join strategy, and why?
8. Both sides of a join are 5 TB, and neither was partitioned in advance. Describe the join and its cost.
9. A join on `user_id` takes 3 hours; 99% of reducers finish in 10 minutes. What's happening and how do
   you fix it?
10. What's the difference between a narrow and a wide dependency in Spark, and why does it matter for
    both performance and recovery?
11. A Spark job's stage uses `random()` without a seed to sample records. A machine fails mid-job. What
    can go wrong?
12. Why do dataflow engines outperform chained MapReduce jobs for a 10-step pipeline?
13. A colleague's batch job updates the production recommendations table row by row from reducers. List
    the problems and propose a better design.
14. How does writing outputs to a date-partitioned directory and overwriting it make a nightly job safe
    to rerun?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** The first `sort` — it brings all identical URLs together so `uniq -c` (the reducer) can count each
group. `sort` does an external merge sort: sorts chunks in memory, spills sorted runs to disk, merges
them sequentially, so it handles inputs far larger than RAM; a hash map needs all distinct keys in memory.

**2.** Large blocks keep the NameNode's in-memory metadata small and make each read a long sequential
transfer that amortises seeks and task startup. It makes HDFS bad at many small files (metadata
explosion) and small random reads/low-latency access.

**3.** Schedule each task on a machine (or rack) holding a replica of its input block, so the large data
is read from local disk and only the small program moves. With separated storage (S3) there's no local
data, but datacentre networks are now fast enough that reading remotely is acceptable, and separation
gives elasticity.

**4.** Map output is temporary — needed only until reducers fetch it — so replicating it in HDFS would be
wasted I/O. If the machine dies, those outputs are lost, and the framework re-runs the completed map tasks
on another machine.

**5.** A combiner applies reduce-like aggregation to a mapper's local output before the shuffle to cut
network traffic. Valid when the operation is associative and commutative so partial results combine
correctly: partial sums add up. Partial medians can't be combined into the overall median.

**6.** The framework re-runs failed tasks and speculatively runs duplicates; it relies on re-execution
producing the same output and on only the committed output mattering. Non-determinism makes reruns
inconsistent; external side effects get repeated.

**7.** Broadcast hash join: load the 30 MB table into a hash map on every worker and stream the orders
through it. No shuffle or sort of the 10 TB.

**8.** Sort-merge (reduce-side) join: map both inputs to (join key, tagged record), shuffle by key,
merge-sort at reducers, and join each key group (secondary sort to stream). Cost: both 5 TB datasets cross
the network and are sorted.

**9.** Skew: a hot key (e.g. null user_id or a giant account) sends a huge share of records to one
reducer; the job waits for it. Fix: identify hot keys, salt them across several reducers while
replicating the other side's matching records, or handle nulls separately; or use adaptive skew handling.

**10.** Narrow: each output partition depends on a few input partitions (map/filter) — pipelined in memory,
no shuffle, and a lost partition is cheaply recomputed from its parents. Wide: an output partition depends
on all input partitions (groupBy, join) — requires a shuffle, and recomputing a lost partition may need
all parents, so it's costly; shuffle files and checkpoints limit the cascade.

**11.** The lost partition is recomputed with different random choices, so it differs from what
downstream tasks already consumed from the original — e.g. some records counted twice or missing, or two
joined sides inconsistent. Use a deterministic seed (per partition) so recomputation is identical.

**12.** They run the pipeline as one DAG: pipelining narrow steps in memory, writing shuffle data only
to local disk, not materialising and 3×-replicating every intermediate result, not waiting for each job
to finish entirely, and optimising the whole plan (join choices, avoiding unnecessary sorts).

**13.** Thousands of tasks overload the database; task retries and speculative execution cause duplicate
or partial writes; readers see a half-updated table; rollback is hard. Better: write the full output as a
new immutable dataset (files or a new table/index version), validate it, then atomically switch readers;
keep the old version for rollback.

**14.** The job's output for a date is a pure function of that date's (immutable) input, and a rerun
replaces the partition entirely rather than appending — so rerunning or backfilling produces the same
result without duplicates (idempotent at the dataset level), ideally with an atomic swap.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Batch vs online vs stream in one line each?
**A:** Bounded input/throughput; requests/response time; unbounded events/low latency.

**Q:** Unix pipeline stages mapping to MapReduce?
**A:** Extract key = map; sort = shuffle; uniq -c = reduce.

**Q:** Why sort rather than hash for grouping at scale?
**A:** External merge sort handles data larger than memory using sequential I/O.

**Q:** Unix principles carried into batch?
**A:** Uniform interface, logic separate from wiring, immutable inputs, composable stages.

**Q:** HDFS block size and replication?
**A:** 64–128 MB blocks, 3 replicas across machines/racks.

**Q:** What does the NameNode hold?
**A:** Metadata: namespace and block locations — not the data itself.

**Q:** HDFS access pattern?
**A:** Write once, append-only, read many; throughput over latency.

**Q:** Moving computation to data?
**A:** Run tasks where their input blocks live instead of shipping data.

**Q:** MapReduce phases?
**A:** Split → map → partition/sort/spill → shuffle → merge → reduce → output.

**Q:** Where does map output go?
**A:** Local disk (temporary), not HDFS.

**Q:** What's a combiner?
**A:** A local pre-aggregation on mapper output; only for associative/commutative ops.

**Q:** How does MapReduce handle failure?
**A:** Re-run failed tasks; speculative execution for stragglers; atomic output commit.

**Q:** Requirement for safe re-execution?
**A:** Deterministic, side-effect-free functions.

**Q:** Sort-merge join?
**A:** Shuffle both sides by join key; reducers merge each key's records.

**Q:** Broadcast hash join?
**A:** Small side in memory on every worker; stream the big side; no shuffle.

**Q:** Partitioned hash join?
**A:** Both sides pre-partitioned identically; join partition i with partition i; no shuffle.

**Q:** What is skew and a fix?
**A:** A hot key overloads one reducer; salt the key and replicate the other side.

**Q:** What's a dataflow engine?
**A:** Runs a whole workflow as one DAG of operators (Spark, Flink, Tez).

**Q:** Narrow vs wide dependency?
**A:** Narrow: few parent partitions, pipelined. Wide: all parents, needs shuffle.

**Q:** Where are Spark stage boundaries?
**A:** At shuffles (wide dependencies).

**Q:** How does Spark recover lost partitions?
**A:** Lineage: recompute from parent datasets.

**Q:** Replicate vs recompute?
**A:** MapReduce replicates intermediates; Spark recomputes on failure.

**Q:** When to checkpoint an RDD?
**A:** When lineage is long or wide dependencies make recomputation expensive.

**Q:** How should batch jobs publish results?
**A:** Write new immutable output, then atomically switch readers.

**Q:** What is human fault tolerance in batch?
**A:** Fix buggy code and rerun on immutable inputs; restore previous outputs.

**Q:** Main limitation of batch?
**A:** Latency — results are only as fresh as the last run.
