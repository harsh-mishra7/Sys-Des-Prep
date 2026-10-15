# 53 — Object Storage, Erasure Coding & Merkle Trees

> Part 9 — Specialized Building Blocks · Prerequisites: topics 01, 13, 23, 24, 25, 40 (helpful: 32, 48)

Images, videos, backups, log archives, ML datasets, build artifacts: most of the bytes a modern system
stores are not rows in a database but large opaque blobs. Storing them in a relational database bloats it;
storing them on a single file server makes it a single point of failure with a hard capacity limit.
**Object storage** (Amazon S3, Google Cloud Storage, Azure Blob Storage, and open-source systems like Ceph
and MinIO) is the building block designed for this: effectively unlimited, very durable, cheap per byte,
accessed over HTTP by key.

Making that work at the scale of exabytes raises three questions this topic answers:
- How is an object store structured? (flat namespace, separate metadata and data, chunking,
  content-addressing and deduplication)
- How do you make data survive disk and machine failures without paying 3× for every byte? (erasure
  coding versus replication)
- How do you efficiently check that two copies of a huge dataset are identical, and find the differences
  if not? (Merkle trees)

By the end of this topic you should be able to:
1. Explain the object storage model (buckets, keys, immutable objects, flat namespace) and how it differs from a filesystem.
2. Describe the separation between metadata and data, how large objects are chunked, and how content-addressing enables deduplication.
3. Explain how erasure coding works (from XOR parity to Reed–Solomon) and compare it with replication on durability, storage cost, repair traffic and latency.
4. Explain how a Merkle tree lets two parties find differences in large datasets with little data transfer, and where Merkle trees appear in real systems.

---

## 1. The object storage model

### 1.1 Objects, buckets, keys

An object store holds **objects**: a blob of bytes plus metadata (content type, size, checksum, user
metadata, creation time). Each object lives in a **bucket** and is identified by a **key**, an arbitrary
string. The API is small and HTTP-based:

```
PUT    /bucket/key          upload (replaces the whole object)
GET    /bucket/key          download (optionally a byte range)
DELETE /bucket/key
HEAD   /bucket/key          metadata only
GET    /bucket?prefix=p&delimiter=/    list keys
```

What's notably missing compared with a filesystem:
- **No in-place modification.** You can't append to or overwrite bytes 100–200 of an object. A write
  replaces the whole object. (Objects are effectively **immutable**; a "change" is a new version.)
- **No real directories, renames or atomic moves** (section 1.2).
- **No locking or partial-write semantics.** Concurrent PUTs to the same key: the last one to complete
  wins (last-write-wins, topic 33), unless you use conditional writes (`If-Match` / `If-None-Match`
  with ETags, which S3 and GCS support) to get compare-and-set (topic 22).

These restrictions are the point. Immutability removes the hard parts of distributed storage: no
concurrent partial updates to reconcile, and caches (CDNs, topic 14) can cache objects safely by
key + version.

### 1.2 Flat namespace

A filesystem is a tree: directories contain files and directories, and operations like rename or "list
this directory" operate on tree nodes. An object store's namespace is **flat**: a bucket is one big map
from key strings to objects. A key like `photos/2026/10/cat.jpg` is just a string that happens to contain
slashes.

The "folders" you see in consoles are an illusion built from **prefix listing**: `LIST prefix=photos/2026/
delimiter=/` returns keys starting with that prefix and groups anything past the next `/` into "common
prefixes" that look like sub-directories.

Why flat?
- A flat key → object map partitions trivially by key (range or hash partitioning, topic 25). A
  hierarchical tree needs a directory service where every path lookup walks parent directories, and hot
  directories become bottlenecks.
- No directory invariants to maintain across partitions (a rename of a directory with a million entries
  would be a distributed transaction, topic 34).

Costs:
- **"Renaming a folder" means copying and deleting every object under the prefix**: O(number of objects),
  not atomic. Tools that treat object storage like a filesystem (e.g. Hadoop's older S3 connectors writing
  output to a temp "directory" then renaming it to commit, topic 40) were slow and unsafe for this reason.
- **Key design affects performance.** If the store range-partitions keys by prefix, a burst of writes to
  keys sharing one prefix (e.g. `logs/2026-10-15/...` with everyone writing "today") concentrates on one
  partition until the system splits it. S3 scales request rate per prefix and automatically splits hot
  prefixes, but sudden spikes can still be throttled while it adapts.

### 1.3 Consistency

Object stores historically offered weak consistency: Amazon S3 originally gave eventual consistency for
overwrites and listings (a GET right after an overwrite could return the old version; a LIST right after a
PUT might not show the new key). Since December 2020, S3 provides **strong read-after-write consistency**
for PUTs, DELETEs and LISTs; GCS and Azure Blob are also strongly consistent per object. Cross-region
replication remains asynchronous (topic 23, topic 48).

---

## 2. Metadata vs data

### 2.1 Two very different workloads

An object store has two kinds of state:

| | Metadata | Data |
|---|---|---|
| What | Key → {size, checksum, version, ACLs, list of chunk locations} | The bytes, split into chunks |
| Size | Small (hundreds of bytes per object) | Large (KB to TB per object) |
| Operations | Lookups, conditional updates, prefix listings | Sequential write once, read many, delete |
| Needs | Strong consistency, transactions-ish (CAS), sorted for listing | Throughput, capacity, durability, cheap per byte |
| Natural store | A sharded, replicated key-value or database (consensus-backed) | Many storage nodes with lots of disks |

So real systems split them:

```
                        PUT /bucket/videos/a.mp4 (2 GB)
client ──────────────────────────────────────────────►  front end / API layer
                                                            │
                     ┌──────────────────────────────────────┴───────────────────────┐
                     ▼                                                              ▼
           metadata service                                               data / storage nodes
   (sharded KV, sorted by bucket+key,                             (store chunks; know nothing about
    replicated with consensus)                                      keys or buckets, only chunk IDs)
   "videos/a.mp4" -> {size, etag, version,                        node 17: chunk c91, c45, ...
                      chunks: [c91@n17,n4,n33, c92@...]}          node 4:  chunk c91, ...
```

Write path, roughly:
1. The front end splits the incoming stream into chunks and writes each chunk (replicated or erasure-coded,
   section 3) to storage nodes chosen by a placement service, spread across failure domains (disks, racks,
   availability zones; topic 48).
2. Only after all chunks are durable does it write the metadata record pointing at them. **The metadata
   write is the commit point**: until it lands, the object doesn't exist; a crash before it leaves orphan
   chunks, which a background garbage collector later deletes.
3. Overwriting a key writes new chunks and then atomically swaps the metadata pointer; the old chunks are
   garbage-collected (or retained as a previous version if versioning is enabled).

This is the same architecture as GFS/HDFS (topic 40): a metadata master (GFS master / HDFS NameNode)
holding the namespace and chunk locations, and many chunkservers / DataNodes holding fixed-size chunks
(64 MB in GFS, 128 MB HDFS blocks). The difference is that GFS/HDFS keep metadata in **one master's
memory** (which limits the number of files and makes the master a bottleneck), whereas object stores built
for trillions of objects shard the metadata across a distributed database.

### 2.2 Why separate them

- **Independent scaling.** Metadata is IOPS-bound and small; data is bandwidth- and capacity-bound.
- **Data nodes are simple.** They store immutable chunks by ID, verify checksums, and serve reads. No
  knowledge of names, so moving or repairing chunks never touches the namespace logic.
- **Data moves without the client noticing.** Rebalancing a chunk to a new node updates one metadata
  entry.
- **Metadata can use stronger guarantees.** Consensus-replicated metadata gives strongly consistent
  listings and conditional writes, while data uses cheaper coding schemes.

### 2.3 Chunking

Large objects are split into **chunks** (also called blocks, parts, or extents), typically a few MB to
tens of MB.

Why chunk:
- **Parallelism**: chunks of one object live on many nodes, so reads and writes of a 50 GB object use many
  disks and NICs at once.
- **Balanced placement**: a huge object doesn't need one node with 50 GB free.
- **Granular repair**: losing a disk means re-creating the chunks it held, not whole objects.
- **Resumable transfer**: S3's **multipart upload** lets the client upload parts independently (in
  parallel, with retries per part), then a final "complete" call stitches them into one object; until then
  the object isn't visible. Range GETs let clients download in parallel or resume.

Small objects have the opposite problem: billions of tiny objects (thumbnails) make per-object metadata
and per-file overhead dominate. Systems like Facebook's **Haystack** pack many small objects into large
append-only volume files and keep a compact in-memory index of (volume, offset, size), so a read is one
disk seek.

### 2.4 Content-addressing

Normally the key is a name chosen by the user (location-addressing: "the thing stored at this name").
**Content-addressing** names data by a **cryptographic hash of its content**:

```
chunk_id = SHA-256(chunk bytes)     e.g. 9f86d081884c7d65...
```

Consequences:
- **Identity = content.** The same bytes always get the same ID, on any machine, with no coordination
  (compare topic 49's ID schemes).
- **Immutability is built in.** Changing a byte changes the ID; "modifying" content creates a new object.
- **Integrity for free.** Whoever fetches the chunk recomputes the hash and checks it matches the ID. A
  corrupted or tampered chunk is detected, no matter which untrusted node or cache served it.
- **Safe caching anywhere.** A content-addressed blob never changes, so it can be cached forever (the same
  reason versioned URLs work for CDNs, topic 14).
- **Idempotent writes.** Uploading the same chunk twice produces the same ID; the second write is a no-op
  (topic 38).

Where you've seen it: **Git** (every blob, tree and commit is named by its hash), **Docker/OCI images**
(layers addressed by `sha256:` digests), IPFS, backup tools (restic, Borg), Nix store paths.

### 2.5 Deduplication

Content-addressing makes deduplication natural: before storing a chunk, check whether its hash already
exists; if so, just reference it.

```
file v1: [c1][c2][c3][c4]
file v2: [c1][c2][c5][c4]      only c5 is new; v2 costs one chunk of storage
```

How you chunk determines how much you dedupe:
- **Fixed-size chunking** (every 4 MB): simple, but inserting one byte at the start of a file shifts every
  boundary, so every chunk's hash changes and nothing dedupes.
- **Content-defined chunking (CDC)**: slide a rolling hash (e.g. Rabin fingerprint) over the data and cut
  a chunk boundary wherever the hash matches a pattern (e.g. low 20 bits are zero, giving ~1 MB average
  chunks). Boundaries depend on local content, so an insertion only changes the chunk(s) around it; the
  boundaries resynchronize afterwards. Backup systems and rsync-like tools use this.

Costs of deduplication:
- **Reference counting and garbage collection.** A chunk can only be deleted when no object references it.
  That's a distributed reference-counting problem: counts must be updated atomically with metadata, or
  computed by periodic mark-and-sweep. Getting this wrong deletes live data.
- **The dedup index** (hash → location) is itself a large, hot lookup structure; Bloom filters (topic 50)
  are often used to skip lookups for chunks that are definitely new.
- **Hot shared chunks**: one chunk referenced by millions of objects becomes a hot key (topic 12).
- **Privacy side channels**: in cross-user dedup, "this upload was instant, so the chunk already existed"
  reveals that someone else has that exact file. Some services dedupe only within one user's data.
- Losing one deduplicated chunk damages *every* object that references it, so durability of each chunk
  matters even more.

---

## 3. Erasure coding vs replication

### 3.1 The problem

Disks fail. At scale, constantly: with 100,000 disks and an annual failure rate of ~1–2%, a few disks die
every day. Plus whole machines, racks, power domains and, rarely, availability zones. Data must survive
these failures (durability, topic 01). Object stores advertise eleven nines (99.999999999%) of annual
durability for objects.

The obvious approach is **replication**: keep 3 full copies on 3 different failure domains (the GFS/HDFS
default). It tolerates the loss of any 2 copies and is simple. It also costs **3× the storage**: for an
exabyte of data, two extra exabytes of disks, power and space. Erasure coding gets comparable or better
durability for far less overhead.

### 3.2 The simplest erasure code: XOR parity

Take `k` data chunks and add one parity chunk equal to their XOR:

```
D1 = 1011
D2 = 0110
D3 = 1100
P  = D1 ⊕ D2 ⊕ D3 = 0001

Lose D2. Recover it:  D2 = D1 ⊕ D3 ⊕ P = 1011 ⊕ 1100 ⊕ 0001 = 0110  ✓
```

Any single lost chunk can be rebuilt by XOR-ing the others, because XOR is its own inverse. This is
**RAID 5**. Overhead: (k+1)/k, e.g. 4 data + 1 parity = 1.25×. But it only survives one failure.

### 3.3 Reed–Solomon codes

**Reed–Solomon (RS)** generalizes this: from `k` data chunks compute `m` parity chunks such that **any
`k` of the `k + m` chunks are enough to reconstruct the original data**. Written RS(k, m), e.g. RS(6, 3):
6 data chunks, 3 parity chunks, 9 total, any 3 can be lost.

Intuition: treat the `k` data values as the coefficients of a polynomial of degree `k - 1`. Evaluate it at
`k + m` distinct points and store the values. A polynomial of degree `k - 1` is uniquely determined by any
`k` of its points (two points define a line, three define a parabola...), so any `k` surviving chunks let
you recover the polynomial and hence the data. Arithmetic is done in a finite field (GF(2^8)) so that
everything fits in bytes and there's no rounding. Practical codes are **systematic**: the first `k` chunks
are the original data unchanged, so normal reads need no decoding.

```
object (split)          encode                      store on 9 failure domains
[D1][D2][D3][D4][D5][D6] ──► + [P1][P2][P3]  ──►   n1  n2  n3  n4  n5  n6  n7  n8  n9
                                                     D1  D2  D3  D4  D5  D6  P1  P2  P3

lose any 3 nodes (e.g. n2, n5, n8): read any 6 survivors -> decode -> full data
```

### 3.4 The comparison

| | 3× replication | RS(6,3) | RS(10,4) | RS(17,3) |
|---|---|---|---|---|
| Storage overhead | 3.0× | 1.5× | 1.4× | ~1.18× |
| Failures tolerated | 2 | 3 | 4 | 3 |
| Normal read | 1 chunk from any replica | k data chunks (one each from k nodes) | same | same |
| Read when a node is down | Read another replica | Read k of the survivors and **decode** | same | same |
| Repair 1 lost chunk | Copy 1 chunk | Read **k** chunks, compute, write 1 | read 10 | read 17 |
| CPU | None | Encode on write, decode on degraded read | | |

(RS(6,3) is HDFS's standard erasure-coding policy; RS(10,4) was used by Facebook's f4 and HDFS-RAID;
Backblaze's Vaults use 17 data + 3 parity.)

**Durability**: RS(6,3) tolerates 3 simultaneous losses with half the storage of 3× replication, which
tolerates only 2. For the same overhead, erasure coding gives dramatically higher durability; for the same
durability, much lower overhead. That's why it's universal for large, cold data.

**Cost**: at exabyte scale, going from 3× to ~1.4–1.5× halves the disk fleet. That is the entire business
case.

### 3.5 What erasure coding costs

1. **Repair traffic.** With replication, replacing a lost chunk means copying one chunk from a survivor.
   With RS(k, m), rebuilding one lost chunk requires **reading k chunks** across the network (to re-run
   the decoding), then writing one. For RS(10,4), repairing a 256 MB chunk moves 2.5 GB. When a whole node
   with 100 TB dies, recovery traffic is `k` times what replication would need, competing with user
   traffic. Repair speed matters for durability: the longer a stripe stays degraded, the higher the chance
   of further failures before it's repaired.
   - **Locally Repairable Codes (LRC)**, used in Azure Storage, add extra *local* parities over small
     groups (e.g. 12 data chunks in two groups of 6, each with a local parity, plus 2 global parities). A
     single lost chunk (the common case) is repaired from its group of 6 instead of 12, at slightly higher
     overhead (~1.33×).
2. **Degraded reads are slow.** If a data chunk is unavailable, the read must fetch `k` other chunks and
   decode, increasing latency and fan-out (tail latency, topic 02). With replication you just read another
   copy.
3. **Small objects don't fit well.** A 4 KB object split into 6 data chunks of under 1 KB each, spread over
   9 nodes, multiplies per-request overhead. Systems pack small objects into large stripes first, or
   replicate small/hot objects.
4. **Writes and updates are costlier.** Encoding needs CPU and all `k + m` chunks written. Updating part of
   a stripe means recomputing parity: fine for immutable data, painful for mutable data. That's another
   reason object storage (immutable objects) and erasure coding fit together so well.
5. **Wider stripes span more failure domains.** RS(6,3) needs 9 independent failure domains to tolerate 3
   *domain* failures; for AZ-level fault tolerance you need chunks spread across zones, which turns every
   read into cross-zone traffic.

### 3.6 Hot vs cold: tiered approaches

Because of these trade-offs, many systems combine the two:
- **New and hot data** is replicated (fast writes, fast reads, cheap repair).
- **Once data cools** (days or weeks later), it's erasure-coded in the background. Facebook's f4 "warm
  BLOB storage" paper describes exactly this transition from Haystack (replicated) to f4 (RS(10,4) plus a
  geo XOR across regions).
- **Archival tiers** use wide stripes and cheap media, with slow retrieval.

### 3.7 Durability also needs detection

Coding protects against *lost* chunks. You also need to notice **silent corruption** (bit rot, firmware
bugs, misdirected writes): every chunk stores a checksum, verified on every read, and background
**scrubbing** periodically reads and verifies all data so latent errors are found and repaired before a
second failure makes them unrecoverable. Comparing large replicas efficiently is where Merkle trees come in.

---

## 4. Merkle trees and efficient data comparison

### 4.1 The problem

Two replicas each hold a key range with a billion keys. They're supposed to be identical, but some writes
were missed (a node was down, a hint was lost, topic 24). How do you find which keys differ?

- Send all data: terabytes over the network.
- Send a hash of every key/value: still a billion hashes (tens of GB) per comparison.
- Send one hash of everything: tells you *whether* they differ, not *where*.

A **Merkle tree** (hash tree, Ralph Merkle, 1979) gets the best of both: one hash comparison tells you if
anything differs, and if something does, a logarithmic number of further comparisons locates it.

### 4.2 Structure

```
                         root = H(H12 + H34)
                       /                     \
              H12 = H(H1 + H2)          H34 = H(H3 + H4)
              /          \               /          \
         H1=H(B1)    H2=H(B2)       H3=H(B3)     H4=H(B4)
            |           |              |            |
         block 1     block 2        block 3      block 4
        (keys a–f)  (keys g–m)     (keys n–s)   (keys t–z)
```

- **Leaves** are hashes of data blocks (in a key-value store: a hash over all key/version pairs in a small
  key range).
- **Each internal node** is the hash of its children's hashes concatenated.
- **The root** is therefore a fingerprint of all the data: changing any byte anywhere changes its leaf
  hash, which changes its parent, and so on up to the root.

### 4.3 Comparing two trees

Two replicas build trees over the **same key ranges** with the same structure, then:

```
compare(nodeA, nodeB):
    if hash(nodeA) == hash(nodeB): return            # this whole subtree matches; skip it
    if leaf: sync the keys in this leaf's range; return
    for each child pair: compare(childA, childB)
```

Example: only block 3 differs.

```
step 1: compare roots            -> differ
step 2: compare H12 vs H12'      -> equal: skip blocks 1–2 entirely
        compare H34 vs H34'      -> differ
step 3: compare H3 vs H3'        -> differ  -> sync block 3's key range
        compare H4 vs H4'        -> equal
```

Cost: with `n` leaves and `d` differing leaves, you exchange about `O(d · log n)` hashes, instead of `O(n)`.
If the replicas are identical (the common case), it's **one** hash comparison. A tree with a million leaves
has depth 20: locating a single differing leaf takes ~20 levels × 2 hashes, a few KB, to find a difference
among a billion keys.

Only the differing leaf ranges' actual data then needs to be transferred.

### 4.4 Where Merkle trees appear

- **Dynamo-style anti-entropy (topic 24)**: Dynamo, Riak and Cassandra (`nodetool repair`) build Merkle trees
  per key range and compare them between replicas to repair divergence that read repair and hinted handoff
  missed. Dynamo kept a tree per key range (per virtual node, topic 13); a weakness the paper notes is that
  when nodes join or leave, key ranges change and trees must be recomputed.
- **Git**: commits point to tree objects which point to blobs, all content-addressed: a Merkle DAG. Two
  repositories compare commit hashes and only transfer objects the other side lacks; equal tree hashes mean
  identical directories, so `git` skips them.
- **Content-addressed storage and backups**: a file's manifest is a tree of chunk hashes; the root hash
  identifies and verifies the entire file (IPFS, BitTorrent v2 piece hashes, restic snapshots).
- **ZFS and Btrfs**: block pointers contain the checksum of the child block, so the filesystem is a Merkle
  tree and corruption is detected end to end.
- **Certificate Transparency** (topic 05) and blockchains: an append-only log whose Merkle root is
  published, so anyone can verify that an entry is included (**inclusion proof**: the sibling hashes along
  the path to the root, `O(log n)` of them) and that the log was only appended to (**consistency proof**).
- **Databases and sync protocols** like DynamoDB's backups, Cassandra's repair, and many file-sync tools use
  hash trees or the closely related idea of comparing range checksums recursively.

### 4.5 Trade-offs and pitfalls

- **Both sides must partition data identically**: the same key ranges per leaf, the same tree shape.
  Otherwise all hashes differ even when the data is identical.
- **Keeping the tree up to date**: every write changes a leaf hash and its ancestors. Systems either update
  incrementally (cost per write: `O(log n)` hashes) or **rebuild periodically** (Cassandra builds trees
  during a repair by scanning the data, which is expensive I/O, hence repairs scheduled during quiet periods).
- **Leaf granularity**: big leaves mean a small tree but each difference forces transferring a big range
  ("overstreaming" in Cassandra); small leaves mean a bigger tree but precise repair.
- **Many small scattered differences** (e.g. a replica that was down for an hour under random writes) touch
  many leaves, so the comparison approaches a full scan; Merkle trees shine when differences are few.

---

## 5. How it connects

- **Topic 01**: durability vs availability; object storage as one of the core building blocks.
- **Topic 02**: degraded erasure-coded reads fan out to k nodes and raise tail latency.
- **Topic 13 and 25**: partitioning keys and chunks; hot prefixes; per-vnode Merkle trees.
- **Topic 14**: immutable, versioned objects cache safely at the CDN; signed URLs for private objects.
- **Topic 22 and 33**: conditional writes (ETags) as compare-and-set; last-write-wins on unconditional
  overwrites.
- **Topic 23 and 24**: replication versus coding; anti-entropy with Merkle trees in leaderless stores.
- **Topic 38 and 49**: content hashes as deterministic, idempotent IDs.
- **Topic 40**: GFS/HDFS as the ancestors of object storage; the cost of rename-based commits on object
  stores.
- **Topic 48**: spreading chunks across failure domains; cross-region replication; backups.
- **Topic 50**: Bloom filters in dedup indexes.
- **Topic 55**: Merkle trees and anti-entropy in the Dynamo paper.

---

## Common misconceptions

- "Object storage is a filesystem in the cloud." → It's a flat key → immutable-blob map with no directories,
  no partial updates, no atomic renames; "folders" are prefix listings.
- "Renaming a prefix is cheap." → It's a copy and delete of every object under it, not atomic.
- "Erasure coding is strictly better than replication." → It's cheaper and more durable per byte, but
  repairs read k chunks, degraded reads are slower, and small or frequently changing data fits poorly.
- "RS(6,3) can survive any 3 disk failures in the cluster." → It survives the loss of any 3 of the 9 chunks
  *of a given stripe*; that requires the 9 chunks to be on independent failure domains.
- "Content-addressing is just a naming convention." → It provides integrity verification, safe caching,
  idempotent writes and dedup, because the name is a function of the content.
- "Deduplication is free storage savings." → It needs reference counting/GC, a dedup index, and turns one
  lost chunk into many damaged objects.
- "A Merkle tree finds differences in one comparison." → One comparison tells you *whether* replicas differ;
  locating each difference takes O(log n) further comparisons.
- "Replication protects against corruption." → It protects against loss; corruption needs checksums and
  scrubbing, or a corrupt copy may be replicated or served.

---

## Further reading (supplements)

- Ghemawat, Gobioff, Leung, "The Google File System" (SOSP 2003).
- Beaver et al., "Finding a needle in Haystack: Facebook's photo storage" (OSDI 2010) and Muralidhar et al., "f4: Facebook's Warm BLOB Storage System" (OSDI 2014).
- Huang et al., "Erasure Coding in Windows Azure Storage" (USENIX ATC 2012), the LRC paper.
- Backblaze blog, "Backblaze Open-sources Reed-Solomon Erasure Coding Source Code" (2015) — accessible explanation.
- DeCandia et al., "Dynamo: Amazon's Highly Available Key-value Store" (SOSP 2007), section 4.7 (Merkle-tree anti-entropy).
- Ralph Merkle, "A Digital Signature Based on a Conventional Encryption Function" (CRYPTO 1987), and RFC 6962 (Certificate Transparency) for inclusion proofs.
- Amazon S3 documentation on strong consistency (announced December 2020) and multipart upload.

---

## Questions

1. Why is an object store's namespace flat, and what operations become expensive because of it?
2. Walk through the write path of a large object in a store that separates metadata and data. What is the commit point, and what happens if the front end crashes halfway?
3. Why is it useful that data nodes know nothing about keys or buckets?
4. Why does a store full of 2 KB thumbnails need a different design than one full of 2 GB videos?
5. What four properties does content-addressing give you, and why does each follow from "name = hash of content"?
6. Why does fixed-size chunking dedupe poorly when a byte is inserted near the start of a file, and how does content-defined chunking fix it?
7. Give two operational risks introduced by cross-user deduplication.
8. Using XOR parity, show how a lost chunk is recovered. What's the limitation?
9. Compare 3× replication and RS(6,3) on storage overhead, number of failures tolerated, and repair traffic for one lost chunk.
10. A node holding 80 TB of RS(10,4)-coded chunks dies. Why might this be more disruptive to the cluster than with 3× replication, and how do LRCs help?
11. Why do many systems replicate fresh data and erasure-code it later?
12. Two replicas each hold 500 million keys and differ in 3 keys. Describe how Merkle trees find them and roughly how much is exchanged.
13. Why must both replicas use the same key-range-to-leaf mapping? What happened in Dynamo when nodes joined or left?
14. When is a Merkle-tree comparison nearly as expensive as a full comparison?
15. Replicated storage still needs checksums and scrubbing. Why?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** A flat key → object map partitions trivially (range or hash on the key) with no directory tree to maintain across partitions and no hot directory nodes; path resolution is a single lookup. Expensive: renaming or moving a "folder" (copy + delete every object under the prefix, not atomic), listing deep hierarchies (prefix scans), and any filesystem-style commit-by-rename protocol.

**2.** The front end splits the stream into chunks, writes each chunk (replicated or erasure-coded) to storage nodes in different failure domains, and waits until all are durable. Then it writes the metadata record (key → size, checksum, version, chunk list) in the consistent metadata store. That metadata write is the commit point: readers only find the object through it. A crash before it means the object was never created; the client retries, and the already-written chunks are orphans that garbage collection removes. A crash after it means the object exists.

**3.** Chunk placement, repair and rebalancing can happen entirely at the data layer: a chunk can be moved or rebuilt and only its location in metadata changes. Data nodes stay simple (store, checksum, serve by chunk ID), and the namespace logic (listing, ACLs, versioning, consistency) lives in one place built for strong consistency, scaled independently.

**4.** With tiny objects, per-object overhead dominates: metadata entries, per-file filesystem inodes, a disk seek per object, and erasure-coding stripes that would split 2 KB into sub-KB fragments across many nodes. Designs pack many small objects into large append-only volumes with a compact in-memory index (Haystack), and replicate or code at the volume level. Large objects need chunking, parallel multipart transfer and range reads.

**5.** (a) Location-independent identity: the same bytes get the same ID anywhere without coordination. (b) Integrity: the reader recomputes the hash and compares it with the ID, so tampering/corruption is detected. (c) Immutability and safe caching: a given ID can only ever refer to one content, so caches never go stale. (d) Dedup and idempotent writes: storing the same content twice yields the same ID, so the second write is a no-op.

**6.** Fixed boundaries are at fixed offsets; inserting one byte shifts all subsequent data relative to the boundaries, so every chunk after the insertion has different bytes and a different hash. CDC cuts boundaries where a rolling hash of the local content matches a pattern, so boundaries move with the content; after the edited region the same boundaries reappear and the same chunk hashes recur.

**7.** Garbage collection/reference counting bugs can delete a chunk still referenced by other users' objects (data loss across many objects), and one lost or corrupted shared chunk damages every object referencing it. There's also a privacy side channel: instant uploads reveal that someone else already stored identical content. Hot shared chunks can also become hotspots.

**8.** P = D1 ⊕ D2 ⊕ ... ⊕ Dk. If Di is lost, XOR all the surviving data chunks and P: everything else cancels, leaving Di. Limitation: only one lost chunk per stripe can be recovered; a second failure before repair loses data.

**9.** 3×: 3.0× storage, tolerates 2 lost copies, repair copies 1 chunk. RS(6,3): 1.5× storage, tolerates any 3 lost chunks of 9, repair reads 6 chunks (and computes) to rebuild 1. So RS halves storage and adds one more tolerated failure, but multiplies repair network/disk reads by 6.

**10.** Rebuilding each lost chunk requires reading 10 surviving chunks from 10 other nodes: on the order of 800 TB of reads cluster-wide versus 80 TB of copying with replication. That repair traffic competes with user I/O and takes longer, extending the window in which stripes are degraded (and reads must decode). LRCs add local parity over small groups so a single lost chunk is rebuilt from its local group (e.g. 6 chunks instead of 12), cutting repair traffic for the common single-failure case.

**11.** Fresh data is hot (read often, latency-sensitive) and sometimes still being written or deleted; replication gives cheap writes, single-chunk reads, fast degraded reads and cheap repair. Once data cools, its read rate drops and it's immutable, so the storage savings of erasure coding dominate, and coding can be done in bulk in the background.

**12.** Both build trees over the same key ranges (say a million leaves, depth ~20). Compare roots: different. Recurse only into children whose hashes differ; equal subtrees are skipped. Each of the 3 differences costs about 20 levels × 2 hashes, so roughly 3 × 40 hashes ≈ a few KB of hashes. Then only the 3 leaves' key ranges (about 500 keys each at a million leaves) are exchanged and reconciled, rather than 500 million keys.

**13.** Hashes are only comparable if they cover exactly the same set of keys. If one replica's leaf covers a–f and the other's a–g, the hashes differ even with identical data and the whole tree looks different. In Dynamo, trees were per key range; when a node joined or left, ranges were reassigned, so the affected trees had to be recalculated, which is expensive. (Dynamo's later partitioning strategy with fixed equal-size partitions mitigated this.)

**14.** When differences are many and spread across most leaves, e.g. a replica that missed a large volume of random writes, or when the leaf mapping doesn't match. Then almost every subtree differs and you descend into nearly all leaves, approaching the cost of comparing everything (plus the cost of building the trees, which itself requires scanning all data).

**15.** Replication protects against losing a copy, not against a copy silently becoming wrong (bit rot, firmware bugs, misdirected writes). Without checksums a corrupt replica may be served or used as a repair source. Checksums detect corruption on read; scrubbing reads cold data proactively so latent errors are found while healthy copies still exist to repair from.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** What is an object in object storage?
**A:** An immutable blob plus metadata, stored in a bucket under a key, accessed over HTTP.

**Q:** Why can't you edit part of an object?
**A:** Objects are immutable; a write replaces the whole object (a new version).

**Q:** What are "folders" in object storage?
**A:** An illusion from listing keys by prefix with a delimiter; the namespace is flat.

**Q:** Why is renaming a prefix expensive?
**A:** Every object under it must be copied and deleted; it's O(objects) and not atomic.

**Q:** How do you get compare-and-set on object storage?
**A:** Conditional writes with ETags (If-Match / If-None-Match).

**Q:** Metadata vs data in an object store?
**A:** Metadata: small, strongly consistent key → chunk locations; data: large immutable chunks on storage nodes.

**Q:** What is the commit point of an object write?
**A:** The metadata record pointing to the durable chunks.

**Q:** Why chunk large objects?
**A:** Parallel I/O, balanced placement, granular repair, resumable multipart transfers.

**Q:** How are tiny objects stored efficiently?
**A:** Packed into large volume files with a compact in-memory index (Haystack).

**Q:** What is content-addressing?
**A:** Naming data by the cryptographic hash of its content.

**Q:** Benefits of content-addressing?
**A:** Integrity checks, safe caching forever, dedup, idempotent writes, coordination-free IDs.

**Q:** Why does fixed-size chunking dedupe poorly?
**A:** An insertion shifts all later boundaries, changing every subsequent chunk hash.

**Q:** What is content-defined chunking?
**A:** Cutting chunk boundaries where a rolling hash matches a pattern, so boundaries move with content.

**Q:** Main costs of dedup?
**A:** Reference counting/GC, a large dedup index, shared-chunk blast radius, privacy side channels.

**Q:** How does XOR parity recover a chunk?
**A:** XOR all surviving chunks and the parity; tolerates one loss per stripe.

**Q:** What does RS(k, m) guarantee?
**A:** Any k of the k + m chunks reconstruct the data; tolerates m losses.

**Q:** Overhead of RS(6,3) vs 3× replication?
**A:** 1.5× vs 3.0×, and it tolerates 3 losses instead of 2.

**Q:** Repair cost of one chunk under RS(k, m)?
**A:** Read k chunks across the network, decode, write one.

**Q:** What is a degraded read?
**A:** A read with a data chunk missing: fetch k others and decode, raising latency.

**Q:** What do LRCs improve?
**A:** Single-chunk repair reads only a small local group instead of all k chunks.

**Q:** Why replicate hot data and erasure-code cold data?
**A:** Hot data needs fast reads/writes/repair; cold immutable data benefits most from storage savings.

**Q:** What is scrubbing?
**A:** Periodically reading and checksum-verifying all data to find latent corruption.

**Q:** Merkle tree structure?
**A:** Leaves hash data blocks; each parent hashes its children's hashes; the root fingerprints everything.

**Q:** Cost to find d differences among n leaves?
**A:** About O(d · log n) hash comparisons; one comparison if identical.

**Q:** Requirement for comparing two Merkle trees?
**A:** Identical key-range-to-leaf mapping and tree shape.

**Q:** Where does Dynamo/Cassandra use Merkle trees?
**A:** Anti-entropy repair between replicas of a key range.

**Q:** What is a Merkle inclusion proof?
**A:** The sibling hashes on the path to the root, O(log n), proving an entry is in the tree.

**Q:** When do Merkle trees stop helping?
**A:** When differences are spread across most leaves, or leaves are mismatched.
