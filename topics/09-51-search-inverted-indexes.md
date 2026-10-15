# 51 — Search & Inverted Indexes

> Part 9 — Specialized Building Blocks · Prerequisites: topics 17, 18, 19, 23, 25 (helpful: 39)

A B-tree index (topics 17–18) answers "find the rows where `email = 'a@b.com'`" or "where `price` is
between 10 and 20". It cannot efficiently answer "find the products whose description mentions *wireless*
and *noise cancelling*, best matches first". `LIKE '%wireless%'` can't use a B-tree because the B-tree is
sorted by the whole string, not by the words inside it, so the database scans every row. And even if it
found matches, it has no notion of which match is *better*.

Full-text search needs a different index, organized by **words** rather than by rows: the inverted index.
On top of it sit text analysis (deciding what counts as a "word"), relevance scoring (ranking matches),
prefix structures for autocomplete, and a distributed layer that shards and replicates the index. This
topic covers each of those, using Lucene and Elasticsearch as the reference implementation (OpenSearch and
Solr share the same Lucene core).

By the end of this topic you should be able to:
1. Explain what an inverted index is, how it's built, and how boolean and phrase queries execute against it.
2. Describe the text-analysis pipeline (character filters, tokenizer, token filters) and why index-time and query-time analysis must match.
3. Explain TF-IDF and BM25: what each term in the formula rewards or penalises, and why BM25 replaced TF-IDF.
4. Explain how tries (and their compressed forms) support prefix search and autocomplete, and the alternatives.
5. Describe how Elasticsearch stores an index in immutable segments, shards and replicates it, and executes a distributed query, including its consistency and relevance pitfalls.

---

## 1. The inverted index

### 1.1 Forward vs inverted

A normal table is a **forward index**: document → its contents.

```
doc 1: "The quick brown fox"
doc 2: "The lazy brown dog"
doc 3: "A quick dog jumps"
```

An **inverted index** flips it: term → the list of documents containing it.

```
term     postings list (doc IDs, sorted)
------   -------------------------------
a        [3]
brown    [1, 2]
dog      [2, 3]
fox      [1]
jumps    [3]
lazy     [2]
quick    [1, 3]
the      [1, 2]
```

It has two parts:
- The **term dictionary**: the sorted set of all distinct terms (the vocabulary), with a pointer to each
  term's postings. It must support fast exact lookup and, ideally, prefix/range lookup.
- The **postings lists**: for each term, the sorted list of documents containing it, often with extra
  per-document information:
  - **term frequency** (how many times the term occurs in that doc), needed for scoring (section 3);
  - **positions** (word offsets), needed for phrase and proximity queries;
  - **offsets** (character positions), needed for highlighting.

The name "inverted" is just "the index at the back of a book": word → pages.

### 1.2 Building it

Indexing a document:
1. Run the text through **analysis** (section 2) to get a stream of terms with positions.
2. For each term, append `(doc_id, freq, positions)` to that term's postings.

Done naïvely in a mutable structure, this is a write-heavy random-update problem (every new document
touches hundreds of postings lists). Real engines solve it the way LSM-trees do (topic 19): buffer
documents in memory, periodically write out a complete, **immutable** mini-index called a **segment**,
and merge segments in the background. Section 5.1 covers this.

### 1.3 Executing queries

**Single term** (`quick`): look up the term in the dictionary, read its postings: `[1, 3]`.

**AND** (`quick AND dog`): intersect the postings lists. Because both are sorted by doc ID, intersection
is a merge-like walk:

```
quick: [1, 3, 7, 12, 40, 41, 99 ...]
dog:   [2, 3, 12, 50, 99 ...]

pointer walk: advance whichever is smaller; emit when equal  ->  [3, 12, 99, ...]
```

Engines start from the **rarest** term (shortest list) and use **skip lists** / skip pointers stored in
the postings to jump ahead (`advance(target)`) instead of scanning every entry of the long list. The cost
is driven by the shortest list, not the longest.

**OR** (`quick OR dog`): union, typically with a heap over the lists' current positions. Combined with
scoring, engines avoid fully evaluating every doc using algorithms like **WAND / block-max WAND**
(Lucene 8+): if a document cannot possibly beat the current top-k minimum score, skip it.

**NOT** (`quick NOT fox`): walk `quick` and skip docs present in `fox`.

**Phrase** (`"brown fox"`): intersect the doc lists, then check positions: in each candidate doc, is there
a position `p` for `brown` with `fox` at `p + 1`? This is why positions are stored. **Proximity**
(`"brown fox"~2`) allows a small gap.

### 1.4 Compression

Postings lists for common terms are huge (a term in 50% of 1 billion docs has 500 million entries), so
compression matters both for disk and for how much fits in the page cache.

- **Delta encoding**: store differences between consecutive sorted doc IDs, not the IDs themselves:
  `[1000, 1003, 1010, 1011]` → `[1000, 3, 7, 1]`. Gaps are small numbers.
- **Variable-length / bit-packed integers**: small numbers take few bits. Lucene packs blocks of 128 deltas
  using the minimum bit width needed for that block (frame-of-reference).
- **Roaring bitmaps** for very dense sets (e.g. filter caches): chunked bitmaps that switch between arrays,
  bitmaps and runs depending on density.

The term dictionary in Lucene is stored as an **FST** (finite state transducer), a compressed trie-like
automaton that shares both prefixes and suffixes (section 4.3). It usually fits in memory even for
hundreds of millions of terms.

### 1.5 What the inverted index is bad at

- **Updating a document**: there's no "update one posting in place". An update is a delete (mark the old
  doc as deleted in a bitmap) plus an insert of the whole new document into a new segment.
- **Exact numeric ranges and sorting**: Lucene uses different structures for those: BKD trees (a
  block k-d tree) for numeric/geo ranges (topic 52), and **doc values** (a column-oriented per-field store,
  like topic 20's column storage) for sorting and aggregations.
- **Transactions and joins**: a search engine is a derived, eventually consistent view of data whose
  source of truth is usually elsewhere (section 6).

---

## 2. Tokenization and analyzers

### 2.1 Why analysis exists

The inverted index only matches **exact terms**. If the document says "Running" and the user types
"run", the bytes differ and nothing matches. Analysis converts text into normalized terms, so that things
the user would consider "the same word" become the same term.

### 2.2 The pipeline

In Lucene/Elasticsearch an **analyzer** is a pipeline of three stages:

```
raw text
  │
  ▼  character filters   (0 or more) operate on the raw characters
  │     e.g. strip HTML tags, map "&" -> "and", normalize Unicode
  ▼  tokenizer           (exactly 1) splits the character stream into tokens
  │     e.g. standard (Unicode word boundaries), whitespace, n-gram, language-specific
  ▼  token filters       (0 or more) modify, remove or add tokens
        e.g. lowercase, stop words, stemming, synonyms, ASCII folding, edge n-grams
  │
  ▼
terms (+ positions, offsets)  ->  inverted index
```

Example: `"<p>The Runners' café is RUNNING late</p>"` with an English analyzer:

```
char filter (html_strip):   The Runners' café is RUNNING late
tokenizer (standard):       [The] [Runners'] [café] [is] [RUNNING] [late]
lowercase:                  [the] [runners'] [café] [is] [running] [late]
possessive/stop:            [runners] [café] [running] [late]       ("the", "is" removed)
asciifolding:               [runners] [cafe] [running] [late]
stemmer (Porter/snowball):  [runner] [cafe] [run] [late]
```

### 2.3 The main token filters and their trade-offs

| Filter | Does | Gains | Costs |
|---|---|---|---|
| Lowercase | `RUNNING` → `running` | Case-insensitive match | Loses distinctions ("US" vs "us") |
| Stop words | Drops `the`, `is`, `a`... | Smaller index, less noise | Breaks phrases like "to be or not to be", "The Who" |
| Stemming | `running`, `runs` → `run` | Matches word variants (recall) | Over-stemming: `university`/`universe` → `univers`; under-stemming |
| Lemmatization | Dictionary-based base form (`better` → `good`) | More accurate than stemming | Slower, language data needed |
| Synonyms | `tv` ↔ `television` | Recall | Multi-word synonyms are tricky; index-time synonyms need reindexing to change |
| ASCII folding | `café` → `cafe` | Accent-insensitive | Language-specific meaning lost |
| N-grams / edge n-grams | `quick` → `q, qu, qui, quic, quick` | Partial and prefix matching | Index grows a lot |

Tokenization itself is language-dependent. Whitespace splitting fails for Chinese, Japanese and Thai
(no spaces between words), which need dictionary or statistical segmenters or character n-grams.
German compounds (`Donaudampfschiff`) need decompounding.

### 2.4 Index-time and query-time analysis must agree

The query string goes through analysis too. If documents are indexed with stemming (`running` → `run`) but
the query is analyzed without it, the query term `running` looks for a term that doesn't exist in the
index: zero results. Rules:

- By default, use the **same analyzer** at index and query time for a field.
- Deliberate differences are allowed when understood: e.g. edge n-grams at index time (so `qui` is a
  stored term) but a plain analyzer at query time (so the query `qui` isn't itself split into `q`, `qu`).
- **Changing an analyzer requires reindexing**: the terms already in the index were produced by the old
  pipeline. The usual approach is to build a new index alongside the old one and switch an alias to it
  atomically (similar to expand/contract, topic 47).

### 2.5 Precision vs recall

Analysis choices trade **precision** (what fraction of returned results are relevant) against **recall**
(what fraction of relevant documents were returned). Aggressive stemming and synonyms raise recall and
lower precision. A common technique is to index the same text into multiple fields with different
analyzers (e.g. `title` stemmed and `title.exact` unstemmed) and query both, boosting exact matches.

Exact-value fields (IDs, status codes, tags, emails used for filtering) should not be analyzed at all:
Elasticsearch's `keyword` type stores the whole value as one term.

---

## 3. Relevance scoring

Matching tells you *which* documents contain the query terms. Ranking decides the order. The classic
scoring models are built on three intuitions:

1. **Term frequency (TF)**: a document that mentions "fox" 10 times is probably more about foxes than one
   that mentions it once.
2. **Inverse document frequency (IDF)**: matching a rare term is more informative than matching a common
   one. "the" appears everywhere and tells you nothing; "BM25" appears in few documents and tells you a lot.
3. **Length normalization**: a match in a 10-word title means more than a match in a 10,000-word book.

### 3.1 TF-IDF

For a query term `t` and document `d`, in a corpus of `N` documents where `df(t)` documents contain `t`:

```
tf(t, d)  = number of occurrences of t in d       (often sqrt(count) or 1 + log(count))
idf(t)    = log( N / df(t) )
score(d)  = Σ over query terms t of  tf(t, d) · idf(t)     (plus length normalization)
```

Worked example, `N = 1,000,000` documents, query `"quick fox"`:

| Term | df | idf = log10(N/df) |
|---|---|---|
| quick | 100,000 | 1.0 |
| fox | 1,000 | 3.0 |

A document with "fox" once (tf=1) scores 3.0 from fox alone; a document with "quick" three times scores
3.0 from quick. One mention of the rare term is worth three of the common one.

TF-IDF in the vector space model represents documents and queries as vectors of TF-IDF weights and ranks
by cosine similarity, which also normalizes for length. Lucene used a TF-IDF variant as its default until
version 6 (2016).

Problems with plain TF-IDF:
- **TF grows without bound.** A document that repeats "fox" 1,000 times (keyword stuffing) scores enormously.
  Real relevance saturates: the 50th mention of "fox" adds almost nothing over the 10th.
- **Length normalization is crude** and hard to tune.

### 3.2 BM25

BM25 ("Best Matching 25", from the Okapi system, Robertson et al., 1990s) is the default in Lucene,
Elasticsearch, OpenSearch and Solr. For each query term `t`:

```
                         f · (k1 + 1)
score(t, d) = IDF(t) · ─────────────────────────────────
                        f + k1 · (1 - b + b · |d| / avgdl)

f      = frequency of t in d
|d|    = length of d (in terms);  avgdl = average document length in the field
k1     = term-frequency saturation (default 1.2)
b      = length-normalization strength (default 0.75)
IDF(t) = ln( 1 + (N - n + 0.5) / (n + 0.5) )      n = docs containing t
```

What each piece does:

- **Saturation (`k1`)**: as `f` grows, the fraction `f(k1+1) / (f + k1·…)` approaches `k1 + 1`, never
  exceeding it. With `k1 = 1.2` and average length, `f = 1` gives 1.0, `f = 2` gives ~1.38, `f = 10` gives
  ~1.96, `f = 1000` gives ~2.2. Diminishing returns, so stuffing doesn't pay. `k1 = 0` ignores TF entirely;
  large `k1` behaves more like raw TF.

```
contribution
 2.2 |                           ......................   (asymptote k1+1)
     |                 .......···
 1.5 |          ....··
     |     ..··
 1.0 |  .·
     | .
 0.0 +-----------------------------------------------> term frequency f
      1  2     5          10                      50
```

- **Length normalization (`b`)**: the term `(1 - b + b·|d|/avgdl)` scales `k1`. A document twice the
  average length behaves as if each occurrence counts less. `b = 1` normalizes fully; `b = 0` ignores
  length. Short fields like titles often benefit from different settings than long bodies.
- **IDF**: like TF-IDF's, with a "+1" inside the log so it never goes negative for very common terms.

The total score is the sum over query terms, then multiplied by any **boosts** (e.g. a title match counts
2×) and combined with other signals.

### 3.3 Beyond text statistics

BM25 knows nothing about quality, popularity, freshness or the user. Real search ranks with more signals:
- **Function scores**: multiply in recency decay, popularity, ratings, distance (topic 52).
- **Learning to rank**: retrieve the top few hundred candidates with BM25, then re-rank with a trained
  model using many features. Cheap retrieval, expensive re-ranking on a small set.
- **Vector (semantic) search**: embed queries and documents as dense vectors and find nearest neighbours
  (approximate nearest neighbour indexes such as HNSW). Matches meaning rather than exact words; often
  combined with BM25 as **hybrid search**. The inverted index remains the workhorse for exact terms,
  filters and rare words (part numbers, names) where embeddings are weak.

### 3.4 IDF in a distributed index

IDF depends on corpus-wide statistics (`N`, `df`). In a sharded index, each shard computes them **from its
own documents** by default. If a term happens to be concentrated in one shard, its IDF there is lower than
elsewhere, so the same document would score differently depending on which shard it lives on. With many
documents and random routing, shard statistics converge and it doesn't matter; with few documents or
skewed routing, rankings look odd. Elasticsearch's `dfs_query_then_fetch` adds a preliminary round trip to
gather global term statistics first, at extra latency.

---

## 4. Tries and prefix search (autocomplete)

Autocomplete has different requirements from full-text search: the input is a **prefix** (`"wirel"`), the
response must arrive within a keystroke (tens of ms), and you want the **top few** completions by
popularity, not all matches.

### 4.1 The trie

A **trie** (prefix tree) stores strings character by character along paths from the root. All strings
sharing a prefix share the path for that prefix.

```
                 (root)
                /      \
               c        d
               |        |
               a        o
             /   \      |
            r     t*    g*
            |     |
            t*    s*

stored: "cart", "cat", "cats", "dog"     (* marks end of a word)
```

Prefix search for `"ca"`: walk `c → a` (O(length of prefix)), then every word in that subtree is a
completion: `cart, cat, cats`. Lookup time depends on the prefix length, not on how many strings are
stored.

### 4.2 Making it fast: top-k at each node

Collecting the whole subtree is too slow for short prefixes ("a" has millions of completions). The
standard technique: **store the top-k completions (by score/popularity) at each node**.

```
node "ca":  top3 = [ "car insurance" (9120), "cat food" (7004), "cake recipe" (5531) ]
```

Query = walk to the prefix node, return its precomputed list: O(prefix length). The cost moves to
updates: changing a word's popularity means updating the top-k lists on every node along its path. In
practice the trie is rebuilt periodically (e.g. hourly or daily from query logs, via a batch job, topic 40)
rather than updated per keystroke, then shipped to serving nodes; trending terms may come from a separate
fast path fed by stream processing (topic 41, using top-k sketches from topic 50).

### 4.3 Compressing tries

A naïve trie with one node per character is memory-hungry (pointers per node). Variants:
- **Radix tree / Patricia trie**: collapse chains of single-child nodes into one edge labelled with a
  string (`"car"` instead of `c → a → r`).
- **Ternary search tree**: each node has three children (less, equal, greater), saving memory for sparse
  alphabets.
- **FST (finite state transducer)**: a minimal automaton that shares common **suffixes** as well as
  prefixes ("running", "jumping" share "ing"), and can map each key to an output value (e.g. a weight or
  a postings pointer). Lucene uses FSTs for its term dictionary and for its completion suggester; it's
  extremely compact but immutable: built once, then read.

### 4.4 Other ways to do prefix search

| Approach | How | Trade-off |
|---|---|---|
| Trie / FST in memory | Walk prefix, read top-k | Fastest; separate structure to build and ship |
| Edge n-grams in the inverted index | Index `w, wi, wir, wire, wirel...` as terms | Uses the normal search engine; index grows several× |
| Prefix query on the term dictionary | Range scan terms in `[wirel, wirem)` | Works because the dictionary is sorted; can expand to many terms for short prefixes |
| B-tree `LIKE 'wirel%'` | Leftmost-prefix range scan (topic 18) | Fine for small datasets; no relevance or popularity |

Fuzzy autocomplete (tolerating typos: `"wirless"`) uses **Levenshtein automata**: an automaton accepting
all strings within edit distance `n` of the input, intersected with the term FST. Lucene does this
efficiently for edit distance 1–2.

---

## 5. How a search cluster shards and replicates (Elasticsearch)

### 5.1 Inside one shard: Lucene segments

Each Elasticsearch **shard** is a complete Lucene index, which is a set of **immutable segments**. The
write path looks a lot like an LSM-tree (topic 19):

```
index request
    │
    ├──► in-memory indexing buffer  ─── refresh (default every 1s) ──►  new segment (searchable,
    │                                                                    in OS page cache, not fsynced)
    │
    └──► translog (append-only, like a WAL; fsynced per request by default)

flush (periodically): fsync segments to disk ("Lucene commit"), then trim the translog

background: merge small segments into larger ones; purge deleted docs during merge
```

Key consequences:
- **Near-real-time (NRT) search**: a document isn't searchable until the next **refresh** creates a segment
  containing it. Default refresh interval is 1 second. So a write followed immediately by a search may
  not see the write: a read-your-writes violation (topic 23, topic 28). You can force `refresh=wait_for`
  on a write at a cost, but refreshing very often creates many tiny segments and hurts throughput.
- **Durability** comes from the **translog**, the same idea as a WAL (topic 17): an acknowledged write is
  in the translog (fsynced), so after a crash the shard replays it into a fresh buffer.
- **Updates and deletes**: segments are immutable, so a delete sets a bit in the segment's "live docs"
  bitmap, and an update is delete + add. Deleted docs still occupy space and are skipped at query time
  until a **merge** rewrites segments without them. Heavy update workloads mean lots of merging (write
  amplification, topic 19).
- Immutable segments are cache-friendly (no invalidation of OS page cache for a segment's lifetime) and
  need no locking for readers.

### 5.2 Sharding

An Elasticsearch **index** is split into a fixed number of **primary shards**. Each document is routed to
exactly one:

```
shard = hash(_routing) % number_of_primary_shards       (_routing defaults to the document _id)
```

This is hash partitioning with `mod N` (topic 13, topic 25), with exactly the problem that topic 13
describes: changing `N` would remap almost every document. So **the number of primary shards is fixed at
index creation**. To change it you reindex (or use the split/shrink APIs, which only allow multiples or
factors of the original count, precisely so documents' destinations stay predictable). This is the
"fixed number of partitions" rebalancing strategy from topic 25, but with the number chosen once.

Shards (not documents) are the unit that moves between nodes. With 10 primary shards on 2 nodes, adding 3
more nodes just relocates whole shards. The practical guidance is therefore to pick enough shards for
future growth, but not so many that per-shard overhead (each is a full Lucene index with its own memory
and file handles) dominates. **Time-based indices** (one index per day for logs, with rollover and an alias
over them) sidestep the problem: new data goes to new indices with whatever shard count you choose now.

**Custom routing**: setting `_routing` to e.g. a tenant ID puts all of a tenant's documents on one shard,
so queries for that tenant hit one shard instead of all of them (no scatter-gather). The cost is skew:
a huge tenant creates a hot shard (topic 25's celebrity problem).

Note: search indexes are naturally **document-partitioned** (each shard indexes its own documents: a
*local* index in topic 25's terms), not **term-partitioned** (each shard owns a range of terms: a *global*
index). Term partitioning would make single-term queries hit one shard, but multi-term queries and
indexing would touch many shards. Document partitioning makes writes local and every query a
scatter-gather, which parallelizes well and is what practically every search engine does.

### 5.3 Replication

Each primary shard has `number_of_replicas` replica shards (changeable at any time, unlike primaries),
never placed on the same node as their primary (and spread across availability zones with allocation
awareness, topic 48).

The write path is **primary-backup** (single-leader replication per shard, topic 23):

```
client ─► coordinating node ─► node holding primary shard P0
                                 1. validate, index into P0 (buffer + translog)
                                 2. forward the operation to all in-sync replica copies in parallel
                                 3. wait for in-sync replicas to acknowledge
                               ◄─ 4. ack to client
```

- Elasticsearch replicates the **operation** (document replication): each replica re-analyzes and indexes
  the document itself. (OpenSearch also offers **segment replication**, where the primary indexes once and
  replicas copy finished segment files, trading indexing CPU for network bytes and some extra lag.)
- The master node tracks the set of **in-sync copies** per shard (similar in spirit to Kafka's ISR,
  topic 37). A replica that fails to ack is removed from the in-sync set so writes don't block, and must
  recover (copy missing operations or segments) before rejoining.
- Each operation gets a **sequence number** and a **primary term** (an epoch number that increments when a
  new primary is promoted, conceptually like Raft terms, topic 30). These let a recovering replica catch up
  with only the operations it missed and let replicas reject writes from a deposed primary.
- `wait_for_active_shards` controls how many copies must be available before a write is attempted (it's a
  pre-check, not a quorum write).

Replicas serve **reads**, so they scale query throughput as well as providing failover. But because each
copy refreshes independently, two copies may momentarily differ: the same query can return slightly
different results depending on which copy served it (a monotonic-reads issue, topic 23). Using a
`preference` parameter (e.g. a session ID) pins a user to the same copies.

The **cluster state** (index metadata, shard allocations) is managed by an elected **master node**. Since
Elasticsearch 7, master election uses a quorum-based consensus protocol among master-eligible nodes (to
avoid split brain, topic 23 and topic 30), so you run 3 dedicated master-eligible nodes in production.

### 5.4 Distributed query execution: query then fetch

A search request goes to any node, which becomes the **coordinating node**:

```
Query phase (scatter):
  coordinator ──► one copy of every shard (primary or replica)
  each shard: run the query locally, score, return top (from + size) doc IDs and scores
  coordinator: merge the per-shard lists into a global top (from + size)

Fetch phase (gather):
  coordinator ──► only the shards holding the final top `size` docs: "send me these documents"
  coordinator ──► client: the results
```

Consequences:
- **Every query touches every shard** (unless routed). Latency is the slowest shard's latency: tail-latency
  amplification under fan-out (topic 02). More shards = more parallelism but more fan-out.
- **Deep pagination is expensive**: page 1,000 of 10 results asks *every* shard for its top 10,010, and the
  coordinator sorts `shards × 10,010` entries. Elasticsearch caps `from + size` at 10,000 by default and
  offers `search_after` (keyset pagination by the last sort values, like topic 49's ID pagination) and
  point-in-time snapshots instead.
- **Aggregations** (facets, counts) are computed per shard and merged; some, like `terms` top-N, are
  approximate because each shard only returns its local top-N (the same top-k-across-partitions problem
  as Count-Min heavy hitters, topic 50). Cardinality aggregations use HyperLogLog++ (topic 50).

### 5.5 The search engine as derived data

Search engines are usually **not** the source of truth. The canonical data lives in a database; the index
is a **derived view** kept in sync by:
- dual writes from the application (risky: the two can diverge on partial failure, topic 39);
- **change data capture** from the database's log or a transactional outbox into a stream that an indexer
  consumes (topic 39): the robust approach;
- periodic batch reindexing from the database (topic 40).

Because it's derived, you can always rebuild it, which is what makes reindexing for analyzer or mapping
changes acceptable. And because it's asynchronously fed, it's eventually consistent with the source:
searching immediately after writing may not find the write.

---

## 6. How it connects

- **Topic 02**: tail-latency amplification across shard fan-out.
- **Topic 13 and 25**: `hash % N` routing and why primary shard count is fixed; document- vs
  term-partitioned (local vs global) indexes; custom routing hotspots.
- **Topic 17**: the translog is a WAL; durability via fsync.
- **Topic 18**: B-tree indexes vs inverted indexes; leftmost-prefix for `LIKE 'x%'`.
- **Topic 19**: segments + merges are the LSM pattern; deletes are tombstones; write amplification.
- **Topic 20**: doc values are column storage inside the search engine.
- **Topic 23 and 28**: primary-backup replication, in-sync copies, near-real-time visibility breaking
  read-your-writes.
- **Topic 30 and 37**: primary terms and the in-sync set resemble Raft terms and Kafka's ISR.
- **Topic 39–41**: CDC to keep the index in sync; batch-built and stream-updated autocomplete.
- **Topic 50**: top-k and HyperLogLog in aggregations and trending suggestions.
- **Topic 52**: geo queries in Lucene use BKD trees and are combined with text relevance.

---

## Common misconceptions

- "`LIKE '%term%'` with an index is the same as full-text search." → It can't use a B-tree (no leftmost
  prefix), scans everything, and has no notion of word boundaries, stemming or relevance.
- "The search engine is just another database." → It's usually a derived, eventually consistent copy with
  weak transactional guarantees; keep the source of truth elsewhere.
- "A document is searchable as soon as the index request returns." → Only after the next refresh
  (default ~1 s in Elasticsearch). It *is* durable (translog) before it's visible.
- "More term occurrences always means more relevant." → BM25 saturates term frequency on purpose;
  plain TF rewards keyword stuffing.
- "I can change the analyzer on an existing field." → The indexed terms were produced by the old analyzer;
  you must reindex.
- "Updates in Elasticsearch modify the document in place." → They mark the old version deleted and index a
  new one; space is reclaimed at merge time.
- "Adding nodes lets me increase the number of primary shards." → Primary shard count is fixed per index
  (because of `hash % N`); nodes let you spread existing shards and add replicas.
- "Replicas only exist for failover." → They also serve reads, scaling query throughput.

---

## Further reading (supplements)

- Manning, Raghavan, Schütze, *Introduction to Information Retrieval* (free online): chapters 1–2 (inverted index, tokenization), 5 (compression), 6 (TF-IDF and vector space), 11 (BM25).
- Robertson and Zaragoza, "The Probabilistic Relevance Framework: BM25 and Beyond" (2009).
- *Elasticsearch: The Definitive Guide* (older but the conceptual chapters on analysis, relevance and distributed search remain excellent); current Elasticsearch reference docs on "Reading and writing documents" and "Near real-time search".
- Mike McCandless's blog posts on Lucene (FSTs, segment merging visualizations).
- DDIA chapter 3 (full-text search and fuzzy indexes) and chapter 6 (partitioning secondary indexes by document vs by term).

---

## Questions

1. Why can't a B-tree index on a `description` column answer "documents containing the word *wireless*" efficiently, while an inverted index can?
2. Walk through the execution of `"noise cancelling" AND wireless` against an inverted index. Which per-posting data do you need, and why start with the rarest term?
3. Documents were indexed with an English stemming analyzer, but a developer configures the query to use the `keyword` (no-analysis) analyzer. A user searches for "running shoes". What happens?
4. What are the precision/recall effects of adding stop-word removal and synonyms? Give an example where stop-word removal breaks a query.
5. In BM25, what do `k1` and `b` each control? What would happen to rankings with `k1 = 0`? With `b = 0`?
6. Why did BM25 replace classic TF-IDF as the default? Give the specific weakness it fixes.
7. Two identical documents live on different shards and get different scores for the same query. Why, and how can you fix it?
8. How does storing top-k completions at each trie node make autocomplete fast? What does it make harder?
9. Compare edge n-grams in the inverted index with an in-memory FST/trie suggester for autocomplete.
10. A user creates a post and is immediately redirected to a search page that doesn't show it. Explain the mechanism in Elasticsearch, and the options for fixing it.
11. Why is the number of primary shards fixed at index creation in Elasticsearch, but the number of replicas isn't?
12. A multi-tenant app sets `_routing` to the tenant ID. What's gained and what risk appears?
13. Why is fetching page 5,000 of search results expensive in a sharded search cluster, and what's the alternative?
14. Why do search engines partition by document rather than by term?
15. Why should a search index generally not be the source of truth, and how should it be kept in sync with the database?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** A B-tree is sorted by the entire column value, so it can find values with a given prefix (leftmost-prefix rule) but not values containing a word somewhere in the middle; `LIKE '%wireless%'` becomes a full scan. An inverted index is keyed by each individual term, so `wireless` is a dictionary lookup returning the exact postings list of documents containing it.

**2.** Analyze the query into terms `noise`, `cancel` (if stemmed), `wireless`. Look up each term's postings. Intersect doc IDs, starting from the shortest list and using skip pointers to advance the longer lists. For candidate docs, check positions: `cancel` must appear at the position right after `noise` (phrase). Then score with BM25 using term frequencies and field lengths. Starting from the rarest term minimizes the number of candidates and lets the long lists be skipped through rather than scanned.

**3.** The index contains stemmed lowercase terms like `run` and `shoe`. With the keyword analyzer the query is the single term `running shoes` (whole string, original case), which doesn't exist in the index: zero results. Even per-word without stemming, `running` wouldn't match `run`. Index-time and query-time analysis must be compatible.

**4.** Stop-word removal shrinks the index and removes noise, slightly raising precision for most queries, but can wreck queries made of stop words: "to be or not to be", "The Who", "IT" (lowercased to "it"). Synonyms raise recall (a search for "tv" finds "television") but can reduce precision when synonyms are context-dependent ("apple" the fruit vs the company).

**5.** `k1` controls term-frequency saturation (how quickly additional occurrences stop adding score); `b` controls how strongly scores are normalized by document length relative to average. `k1 = 0`: TF is ignored; a document scores the same whether the term appears once or 50 times (pure IDF, binary match). `b = 0`: no length normalization; long documents that mention the term more often by sheer length are favored.

**6.** Classic TF-IDF's term-frequency component grows without bound, so repeating a term (keyword stuffing, or just long repetitive docs) keeps increasing the score. BM25 saturates TF towards `k1 + 1` and has a tunable length normalization, matching how relevance actually behaves.

**7.** IDF (and average length) are computed per shard from local statistics. If the query term's document frequency differs between the shards, its IDF differs, so identical documents score differently. Fixes: use `dfs_query_then_fetch` to collect global term statistics before scoring (extra round trip), use fewer shards for small indices, or rely on large, randomly routed data where shard statistics converge.

**8.** Each node caches its subtree's best k completions, so answering a prefix is just walking to the node (O(prefix length)) and returning the list, without traversing the possibly huge subtree. Harder: updates. A popularity change must update top-k lists on every ancestor node, and memory grows by k entries per node. So tries are usually rebuilt in batch and shipped, rather than updated in place.

**9.** Edge n-grams store every prefix of every word as a term in the regular index: no extra system, full query flexibility (filters, scoring, mid-phrase words), but the index grows several times and queries run through full search machinery. An FST/trie suggester is a compact in-memory structure answering prefix lookups in microseconds with precomputed weights, but it's immutable/rebuilt, supports less flexible matching, and is a separate structure to maintain.

**10.** The write was acknowledged once it was in the in-memory buffer and translog on the primary and replicas, but it only becomes searchable after a refresh writes a new segment (default every 1 s). The redirect searched before the refresh: a read-your-writes violation. Options: `refresh=wait_for` on that write (blocks until the next refresh), read the just-created object directly by ID (GET by ID is real-time from the translog) and merge it into the UI, or accept the small delay. Lowering the global refresh interval hurts throughput by creating many tiny segments.

**11.** Documents are placed by `hash(_routing) % number_of_primary_shards`; changing the count would change almost every document's destination, requiring a full reshuffle. Replicas are just copies of existing shards, so adding or removing them doesn't change where any document belongs: the cluster simply creates or drops copies.

**12.** Gain: all of a tenant's docs live on one shard, so tenant-scoped queries hit one shard rather than fanning out to all, improving latency and throughput. Risk: skew. A very large or very busy tenant makes one shard (and its node) hot and large; shards can't be split per-tenant. Mitigations: dedicated indices for big tenants, or routing partitions (`routing_partition_size`) to spread a tenant over a few shards.

**13.** With `from = 49,990, size = 10`, each shard must produce and return its top 50,000 hits, and the coordinator must merge shards × 50,000 entries to find the 10 it needs, all of which is discarded work. Cost grows linearly with page depth and shard count. Alternative: `search_after` with the last hit's sort values (keyset pagination), plus a point-in-time view for consistent paging.

**14.** With document partitioning each document is indexed entirely on one shard, so a write touches one shard and queries run in parallel on all shards then merge. With term partitioning, indexing one document touches the shards owning each of its hundreds of terms, and multi-term queries must combine postings across shards over the network. Document partitioning keeps writes local and scales query fan-out, so it's the near-universal choice.

**15.** Search engines have weak transactional guarantees, eventual visibility, and their indices are routinely rebuilt (analyzer or mapping changes), so they make a poor system of record. Keep the canonical data in a database and treat the index as a derived view fed by CDC or a transactional outbox (topic 39), plus the ability to fully reindex from the source. Avoid dual writes, which can diverge on partial failure.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What is an inverted index?
**A:** A map from each term to the sorted list of documents (postings) containing it.

**Q:** Two parts of an inverted index?
**A:** The term dictionary and the postings lists.

**Q:** What extra data can postings hold and why?
**A:** Term frequency (scoring), positions (phrase queries), offsets (highlighting).

**Q:** How is an AND query executed?
**A:** Intersect sorted postings, starting from the rarest term, using skip pointers.

**Q:** How are postings compressed?
**A:** Delta-encode sorted doc IDs, then bit-pack/varint the small gaps.

**Q:** Three stages of an analyzer?
**A:** Character filters → one tokenizer → token filters.

**Q:** Stemming trade-off?
**A:** Higher recall (word variants match), lower precision (over-stemming conflates unrelated words).

**Q:** Why must index and query analysis match?
**A:** The query terms must be produced the same way as the indexed terms or they won't match.

**Q:** What does changing an analyzer require?
**A:** Reindexing, usually into a new index then swapping an alias.

**Q:** TF-IDF intuition?
**A:** Score rises with term frequency in the doc and with the term's rarity across the corpus.

**Q:** IDF formula (classic)?
**A:** log(N / df), where df is the number of docs containing the term.

**Q:** BM25 k1?
**A:** Term-frequency saturation; default 1.2; contribution caps at k1 + 1.

**Q:** BM25 b?
**A:** Strength of length normalization relative to average doc length; default 0.75.

**Q:** Why BM25 over TF-IDF?
**A:** TF saturation (resists keyword stuffing) and tunable length normalization.

**Q:** Why do scores differ across shards?
**A:** IDF is computed from per-shard statistics by default; dfs_query_then_fetch uses global stats.

**Q:** How does a trie answer a prefix query?
**A:** Walk the prefix path; all completions are in that node's subtree.

**Q:** How does autocomplete stay fast for short prefixes?
**A:** Precompute and store the top-k completions at each trie node.

**Q:** What is an FST?
**A:** A minimal automaton sharing prefixes and suffixes, mapping keys to outputs; Lucene's term dictionary.

**Q:** What is a Lucene segment?
**A:** An immutable mini inverted index; a shard is a set of segments merged in the background.

**Q:** What is a refresh?
**A:** Turning the in-memory buffer into a searchable segment; default every 1 s (near-real-time search).

**Q:** What is the translog?
**A:** Elasticsearch's write-ahead log for durability between Lucene commits.

**Q:** How are updates handled in Lucene?
**A:** Mark old doc deleted in a bitmap, add the new version; merges reclaim space.

**Q:** How does Elasticsearch route a document?
**A:** hash(_routing) % number_of_primary_shards; _routing defaults to _id.

**Q:** Why is primary shard count fixed?
**A:** Changing N in hash % N would remap nearly every document.

**Q:** Elasticsearch replication model?
**A:** Primary-backup per shard; primary forwards operations to in-sync replicas before acking.

**Q:** What is a primary term?
**A:** An epoch incremented on primary promotion, used to reject operations from a stale primary.

**Q:** Query-then-fetch?
**A:** Query phase gets top IDs+scores from every shard; fetch phase retrieves only the final top docs.

**Q:** Why is deep pagination expensive?
**A:** Every shard returns from+size hits and the coordinator merges them all; use search_after.

**Q:** Document- vs term-partitioned index?
**A:** Search engines partition by document: local writes, scatter-gather queries.

**Q:** How should a search index be kept in sync?
**A:** As derived data via CDC/outbox from the source-of-truth database, with full reindex as fallback.
