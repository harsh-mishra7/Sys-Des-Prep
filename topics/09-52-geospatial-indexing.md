# 52 — Geospatial Indexing

> Part 9 — Specialized Building Blocks · Prerequisites: topics 17, 18, 25 (helpful: 51)

"Find restaurants within 2 km of me." "Which drivers are closest to this pickup point?" "Which delivery
zone contains this address?" These are **spatial queries**: the question is about nearness in two
dimensions, not about equality or a one-dimensional range. Ordinary database indexes are one-dimensional:
a B-tree sorts keys along a line (topic 17). The whole art of geospatial indexing is mapping
two-dimensional nearness onto structures that are good at one-dimensional order, or building trees that
partition space directly, and then handling the places where that mapping breaks.

By the end of this topic you should be able to:
1. Explain why a B-tree on `(lat, lng)` (composite or two separate indexes) can't answer proximity queries efficiently.
2. Describe how geohash encodes a location as a string whose prefixes are nested cells, and how a proximity query uses it.
3. Describe quadtrees (fixed and adaptive) and how they handle uneven density.
4. Explain S2 cells: cube projection, Hilbert curve ordering, cell levels and region coverings, and why they're better behaved than geohash.
5. Execute a proximity query correctly, including the boundary problem, neighbor cells, the antimeridian and poles, and the final exact-distance filter.

---

## 1. Why a plain index on (lat, lng) doesn't work

### 1.1 The query

We have points (restaurants) with latitude and longitude, and a query point `Q`. We want all points
within radius `r` of `Q` (a **radius** or **proximity** query), or the `k` closest (**k-nearest
neighbours**, kNN).

The naïve approach computes the distance from `Q` to every point: correct, and O(n) per query. With 10
million points and thousands of queries per second, that's hopeless. We need an index that narrows the
candidates to the neighbourhood of `Q`.

### 1.2 Approximating a circle with a bounding box

First simplification: a circle is awkward, so query the bounding box around it, then filter:

```
lat BETWEEN Q.lat - Δlat AND Q.lat + Δlat
AND lng BETWEEN Q.lng - Δlng AND Q.lng + Δlng
```

(Δlat ≈ r / 111 km per degree; Δlng ≈ r / (111 km · cos(lat)), because meridians converge toward the
poles.) Then compute exact distances for the candidates in the box and keep those within `r`.

### 1.3 Two separate indexes

With a B-tree on `lat` and another on `lng`, the database can use one of them (or combine both with a
bitmap AND, topic 18):

```
                lng range
                  |   |
   ---------------+---+--------------- lat range
   ///////////////|XXX|///////////////      <- the lat index returns this whole
   ---------------+---+---------------         horizontal band (all longitudes)
                  |   |
                  |   |   <- the lng index returns this whole vertical band
                  |   |      (all latitudes)
```

The lat index returns every point in a band that wraps around the entire Earth; the lng index returns
a band running pole to pole. The answer is only the tiny intersection `XXX`. Each band can hold a large
fraction of the dataset, so even with a bitmap AND, the database reads far more index entries than it
returns.

### 1.4 One composite index

A composite B-tree index on `(lat, lng)` sorts by `lat` first, then `lng` (topic 18's leftmost-prefix
rule). A range condition on `lat` means the `lng` part of the index can't narrow the scan: within the
`lat` range, entries are ordered by `lat` and only secondarily by `lng`, so the matching `lng` values are
scattered across the whole band. The index scans the entire latitude band and filters `lng` row by row.
It's the same problem as 1.3 in a different shape.

The root issue: **a B-tree orders keys along one dimension**. Points that are close in 2-D are not close in
the sort order of either coordinate alone. We need a key where 2-D nearness translates to 1-D nearness (a
space-filling curve), or a tree that splits space in two dimensions at once.

---

## 2. Space-filling curves: the core idea

A **space-filling curve** is a path that visits every cell of a grid exactly once. If you number cells in
the order the curve visits them, you get a one-dimensional key for each 2-D cell. Store that key in an
ordinary B-tree (or sorted set, or LSM) and a range of keys corresponds to a region of space.

Two curves matter:

```
Z-order (Morton) curve, 4x4:          Hilbert curve, 4x4:

 0  1 | 4  5                           5  6  9 10
 2  3 | 6  7                           4  7  8 11
 -----+-----                           3  2 13 12
 8  9 |12 13                           0  1 14 15
10 11 |14 15

Interleave the bits of x and y.        Recursive U-shapes; consecutive numbers are
Simple, but has big jumps (7 -> 8      always adjacent cells: no jumps.
goes across the whole grid).
```

Both have the property we want *most of the time*: cells with nearby keys are nearby in space. Neither can
guarantee the reverse: **nearby cells can have very different keys** (for example cells 3 and 9 in the
Z-order grid above are vertically adjacent, and cells 7 and 8 have consecutive keys but sit at opposite
edges of the grid). Every
curve-based scheme has to compensate for this, which is the boundary problem of section 6. The Hilbert
curve has better locality (no long jumps), which is why S2 uses it. Geohash is a Z-order curve.

---

## 3. Geohash

### 3.1 Encoding

Geohash (Gustavo Niemeyer, 2008) repeatedly halves the world, alternating longitude and latitude, and
records one bit per halving:

```
Start: longitude range [-180, 180], latitude range [-90, 90]

bit 1 (lng): is lng in the upper half [0, 180]?        1 = yes, 0 = no -> range halves
bit 2 (lat): is lat in the upper half [0, 90]?         ...
bit 3 (lng): is lng in the upper half of the new range?
... and so on, alternating
```

The bits interleave longitude and latitude (a Z-order curve), and every 5 bits become one base-32
character (alphabet `0123456789bcdefghjkmnpqrstuvwxyz`, which omits `a i l o`). For example, the point
(57.64911, 10.40744) in Denmark encodes as `u4pruydqqvj`.

The key property: **each character subdivides the previous cell into 32 sub-cells**. A geohash is a
hierarchical cell name; a **shorter prefix is a bigger cell containing all longer hashes that start with
it.**

```
u          ~5,000 km cell
u4         ~1,250 km x 625 km
u4p        ~156 km
u4pr       ~39 km x 19.5 km
u4pru      ~4.9 km x 4.9 km
u4pruy     ~1.2 km x 0.61 km
u4pruyd    ~153 m x 153 m
u4pruydq   ~38 m x 19 m
```

| Length | Approx. cell size (at equator) |
|---|---|
| 1 | 5,000 × 5,000 km |
| 4 | 39 × 19.5 km |
| 5 | 4.9 × 4.9 km |
| 6 | 1.2 × 0.61 km |
| 7 | 153 × 153 m |
| 8 | 38 × 19 m |
| 9 | 4.8 × 4.8 m |

### 3.2 Querying with geohash

Store each point's geohash (say at length 9) in an indexed column or a sorted structure. A **prefix
query** becomes a **range scan**: "all points in cell `u4pruy`" = all keys in `[u4pruy, u4pruz)`. Any
B-tree, LSM store, or sorted set can do that efficiently.

For a radius query:
1. Pick a precision whose cell size is comparable to the radius (for 2 km, length 5 at ~4.9 km, or
   length 6 at ~1.2 km with more cells).
2. Compute the geohash of `Q` at that precision, **and its 8 neighbours** (section 6.1 explains why).
3. Range-scan each of the 9 cells.
4. Compute the exact distance to each candidate and discard those beyond `r`.

```
+--------+--------+--------+
|   NW   |   N    |   NE   |
+--------+--------+--------+
|   W    | Q cell |   E    |      9 prefix range scans, then exact distance filter
+--------+--------+--------+
|   SW   |   S    |   SE   |
+--------+--------+--------+
```

Redis's `GEOADD` / `GEOSEARCH` work exactly like this: each member is stored in a **sorted set** (topic 11)
with a 52-bit interleaved geohash as its score; a search computes the covering cells (the center cell and
its neighbours at a suitable precision), does score-range lookups on the sorted set, and filters by exact
distance.

### 3.3 Geohash's weaknesses

- **Non-uniform cell shapes and sizes.** Cells are rectangles in lat/lng degrees. A degree of longitude is
  111 km at the equator and shrinks to 0 at the poles, so cells get narrower toward the poles; even-length
  and odd-length geohashes have different aspect ratios.
- **Z-order jumps.** Adjacent cells can have completely different prefixes. The classic example: points
  just either side of the equator or the prime meridian share *no* prefix at all, even though they're
  metres apart. Never assume "common prefix length = closeness".
- **Fixed grid.** Cell size is fixed per length; a dense city and an empty desert get the same cells, so
  the number of points per cell varies enormously.

---

## 4. Quadtrees

### 4.1 Structure

A **quadtree** partitions a 2-D region recursively into four quadrants (NW, NE, SW, SE). It's a tree
in memory (or on disk) rather than a key encoding.

A **point-region (PR) quadtree** with a capacity of `K` points per leaf:
- Start with one leaf covering the whole world.
- Insert points into the leaf whose region contains them.
- When a leaf exceeds `K` points, **split** it into four children and redistribute its points.

```
Sparse rural area: big leaves          Dense city: deep subdivision

+---------------+---------------+
|               |       |   |-+-|
|      .        |   .   |---+-+-|
|               |       | . |:|:|  <- many small leaves where points are dense
|               +-------+---+---+
|               |   .   |  ..   |
+---------------+-------+-------+
|               |               |
|       .       |      .        |
|               |               |
+---------------+---------------+
```

### 4.2 Why it's useful

- **Adapts to density.** Leaves are small where points are dense and large where they're sparse, so each
  leaf holds a bounded number of points. Geohash's fixed grid can't do this.
- **Radius / box query**: descend from the root, visiting only children whose region intersects the
  query circle (or box), collect points from intersecting leaves, filter by exact distance.
- **kNN**: best-first search: keep a priority queue of nodes ordered by minimum possible distance to `Q`;
  pop the closest, expand it; stop when the k-th best point found is closer than the next node in the
  queue. This naturally handles "expand until you have enough results", which fixed-cell schemes need
  extra rounds for.

### 4.3 Costs

- It's a pointer-based tree, natural **in memory** but awkward to store in a database or distribute across
  machines. Systems that use quadtrees for "nearby" services often build the tree in memory on each server
  (from a database snapshot) and rebuild it periodically, rather than updating a shared structure.
- **Moving points** (drivers updating location every few seconds) cause constant removals and inserts and
  possibly splits/merges; a high update rate pushes designs toward simple fixed grids (geohash or S2 cells
  as keys in a hash map or Redis) where an update is "remove from old cell, add to new cell".
- Unbalanced data makes the tree deep in dense areas.

Note: any quadtree cell can be named by its path (e.g. `0-3-1-2`), which is itself a hierarchical key, just
like a geohash with base 4. Map tile systems ("quadkeys" in Bing Maps) do exactly this. Geohash, quadtree
paths and S2 cell IDs are all the same idea: recursive subdivision with a name per cell.

### 4.4 Related trees (for completeness)

- **R-trees** group nearby objects into nested **bounding rectangles** (rather than splitting space at
  fixed points). They index shapes (polygons, lines) as well as points and are what PostGIS (via GiST),
  SQLite's R*Tree module, and Oracle Spatial use.
- **k-d trees** split alternately on x and y at the median. Lucene/Elasticsearch use **BKD trees** (a
  block, disk-friendly k-d tree) for numeric ranges and `geo_point` fields (topic 51).

---

## 5. S2 cells

### 5.1 The idea

Google's **S2 geometry library** is a hierarchical cell system designed to fix geohash's distortions. It
works on the **sphere**, not on a flat lat/lng rectangle:

1. **Project the sphere onto a cube.** Surround the Earth with a cube and project each point outward onto
   one of the cube's **6 faces**. Each face covers a roughly equal area of the Earth.
2. **Subdivide each face as a quadtree.** Level 0 is a whole face; each level splits every cell into 4, down
   to **level 30** (cells about 1 cm² on the ground).
3. **Order cells along a Hilbert curve** on each face (section 2), and encode each cell as a **64-bit
   integer**: 3 bits for the face, then 2 bits per level of the Hilbert position, then a sentinel bit
   marking the level.

```
     cube face 2
         ┌───┐
     ┌───┼───┼───┬───┐
     │ 1 │ 0 │ 4 │ 3 │      the 6 faces unfolded (illustrative); each is a quadtree of
     └───┼───┼───┴───┘      cells ordered along a Hilbert curve
         │ 5 │
         └───┘

cell ID (64 bits):  [face:3][pos at level 1:2][pos at level 2:2]...[1][000...]
                                                                   ^ sentinel: marks the level
```

Properties that fall out of this:
- **Cell IDs are integers in a 1-D order with good locality** (Hilbert: no jumps within a face). Store them
  in any sorted index and range scans correspond to compact regions.
- **Containment is arithmetic.** All descendants of a cell have IDs in a contiguous range
  `[range_min, range_max]` computable from the parent's ID. "Is point P inside cell C?" is an integer
  range check.
- **Cells are roughly equal-area at every level, everywhere.** No polar squashing beyond a bounded factor;
  the area of cells at a given level varies by at most about 2× across the globe (geohash cells vary much
  more in shape and true area).
- **Hierarchy**: the parent of a cell is computed by clearing low bits.

| S2 level | Typical cell edge |
|---|---|
| 0 | ~7,800 km (a cube face) |
| 10 | ~7 km |
| 12 | ~2 km |
| 14 | ~600 m |
| 16 | ~150 m |
| 20 | ~10 m |
| 30 | ~1 cm |

### 5.2 Region coverings

The most useful S2 operation: given a region (a circle, a polygon, a delivery zone), compute a
**covering**: a small set of cells, possibly at **mixed levels**, whose union contains the region.

```
A circle covered by S2 cells (mixed levels: big cells inside, small cells along the edge)

     +-------+---+---+
     |       |   | ..|..
     |       +---+-.-+  .
     |   big   |  .  |   .
     |  cell   | .   |    .
     +---------+.----+-----.
```

You control the trade-off with parameters like `max_cells` (e.g. 8) and min/max level: fewer cells means
fewer range scans but more area outside the region (more false candidates to filter); more cells hug the
shape tightly but cost more scans.

A radius query with S2:
1. Compute a covering of the circle (say up to 8 cells).
2. Convert each cell to its ID range `[range_min, range_max]`.
3. Range-scan the index of point cell IDs (points stored at level 30, or at a fixed level) for each range.
4. Filter candidates by exact spherical distance.

A **point-in-polygon** question ("which zone contains this address?") inverts it: precompute coverings of
all polygons, index polygon cells, then look up the point's cell ancestors. Coverings are what let S2-based
indexes handle arbitrary shapes, not just circles.

S2 is used by Google Maps, MongoDB's `2dsphere` index, CockroachDB's spatial indexing, and Foursquare,
among others. Uber's **H3** is a similar hierarchical system using **hexagons** (each hexagon has 6
equidistant neighbours, which is convenient for smoothing and movement analysis), at the cost of
hexagons not nesting perfectly into parent hexagons.

### 5.3 Comparing the approaches

| | Geohash | Quadtree | S2 |
|---|---|---|---|
| What it is | String key from interleaved bits (Z-order) | Adaptive tree splitting into 4 | 64-bit cell IDs on a cube-projected sphere (Hilbert) |
| Storage | Any sorted index / prefix | In-memory tree | Any sorted index (integer ranges) |
| Density adaptation | No (fixed grid per length) | Yes | Via mixed-level coverings |
| Cell shape/area | Distorted toward poles | Planar unless projected | Near-uniform on the sphere |
| Locality jumps | Z-order: large jumps | n/a (tree) | Hilbert: no jumps within a face; jumps at face edges |
| Shapes / polygons | Awkward | Possible | First-class (coverings) |
| Simplicity | Very simple | Simple in memory | Library needed |

---

## 6. Proximity queries and edge cases at cell boundaries

### 6.1 The boundary problem

Every cell-based scheme has the same trap: **a point near the edge of a cell is near points in the
neighbouring cell, which has a different key (often a very different one)**.

```
           cell A        |        cell B
                         |
                     Q • | • P        Q and P are 5 m apart.
                         |
                         |            If you search only Q's cell, P is missed.
```

Searching only the cell containing `Q` misses close points across the edge, even if the cell is much
bigger than the search radius. This is why:
- Geohash searches use the **center cell plus its 8 neighbours**.
- S2 coverings are computed from the **circle**, not from `Q`'s cell, so they include whatever cells the
  circle touches.
- Quadtree searches test intersection of the circle with each node's region, not containment of `Q`.

And never estimate distance by **common prefix length**: two points metres apart across the equator or
prime meridian share no geohash prefix at all, while two points in the same short-prefix cell can be
kilometres apart (opposite corners).

### 6.2 Choosing cell size vs radius

The 9-cell approach only works if the cell is **at least as large as the radius**: otherwise the circle
extends beyond the 3×3 block.

```
cell edge >= r:   the circle around Q fits inside the 3x3 block. Correct.
cell edge <  r:   the circle spills past the neighbours. Missed results.
cell edge >> r:   correct, but 9 huge cells -> many candidates to filter. Slow.
```

So pick the precision whose cell size is just above `r`. Remember that geohash cells shrink in the
east–west direction toward the poles (a degree of longitude covers fewer km), so a precision chosen for the
equator may be too small at 60° latitude; pick precision per query latitude, or use S2 coverings that are
computed for the actual circle.

### 6.3 kNN with cells: expanding rings

"Find the 10 nearest drivers" has no fixed radius. With a cell scheme:
1. Search the center cell + neighbours at a fine precision.
2. If fewer than `k` results (or if the k-th result is farther than the distance to the edge of the
   searched area), **expand**: move one precision level coarser (or add the next ring of cells) and
   search again.
3. Stop when you have `k` candidates **and** the k-th distance is less than the guaranteed covered radius.

The last condition matters: the first `k` points found in the searched cells aren't necessarily the
nearest `k`; a nearer point might sit just outside the searched area. Quadtree best-first search (4.2) and
R-tree kNN handle this naturally.

### 6.4 Always filter by exact distance

Cells approximate; they return a **superset** of candidates (corners of cells extend beyond the circle).
The final step must compute real distance and discard extras. On a sphere, use **haversine** (great-circle
distance) or a more precise geodesic formula for long distances; for short distances an equirectangular
approximation is often fine. Euclidean distance on raw lat/lng degrees is wrong because a degree of
longitude shrinks with latitude.

This is the general **filter-and-refine** pattern: a cheap index narrows candidates (filter) and an exact
computation decides (refine). Bloom filters (topic 50) and inverted-index candidate retrieval followed by
re-ranking (topic 51) are the same shape.

### 6.5 The antimeridian and the poles

- **Antimeridian (±180° longitude)**: a bounding box around a point at longitude 179.9° extends past 180°
  and must wrap to −180°. Naïve `lng BETWEEN a AND b` with `a > b` returns nothing; you must split into two
  ranges. Geohash neighbours must wrap too. (Fiji and parts of Russia and Alaska are real places where this
  bites.)
- **Poles**: near the poles, longitude ranges degenerate (all longitudes meet), so a "box" around a point
  near the North Pole should span all longitudes. Lat/lng-rectangle schemes (geohash, naïve boxes) handle
  this badly. S2's cube projection has no singularity at the poles.
- S2's own seams are at cube face edges: neighbouring cells on different faces have unrelated IDs, which
  coverings handle automatically.

### 6.6 Moving objects and hot cells

For moving objects (drivers, couriers), the index is updated far more often than it's read per object. A
common design keeps a map `cell_id → set of object IDs` (e.g. in Redis or in-memory per server), and on
each location update moves the object between cells only when it crosses a cell boundary. Points of
interest are static and can use heavier structures.

Cells are also natural **partition keys** (topic 25): shard by a coarse cell ID so that a city lives on a
few nodes. The risk is the familiar hotspot: one dense downtown cell becomes a hot partition. Adaptive
schemes (split hot cells into children, or use finer levels in dense areas, like a quadtree does) mitigate
it.

---

## 7. How it connects

- **Topic 17 and 18**: why a one-dimensional B-tree and the leftmost-prefix rule fail for 2-D ranges; how
  cell IDs turn spatial queries back into B-tree range scans.
- **Topic 11**: Redis sorted sets as the storage behind `GEOSEARCH`.
- **Topic 19**: cell-ID keys work equally well in LSM stores (Bigtable/Cassandra row keys).
- **Topic 25**: geographic partitioning by cell; hot cells as hot partitions.
- **Topic 50**: filter-and-refine, as with Bloom filters.
- **Topic 51**: Lucene's BKD trees for geo points; combining distance with text relevance in scoring.
- **Topic 49**: like time-ordered IDs, cell IDs trade locality for possible hotspots on range-partitioned
  stores.

---

## Common misconceptions

- "A composite index on (lat, lng) makes radius queries fast." → The range on `lat` prevents the `lng`
  part from narrowing anything; it scans a whole latitude band.
- "Points with a long common geohash prefix are close, and close points share a long prefix." → The first
  half is true (same small cell); the second is false at cell boundaries, especially across the equator or
  prime meridian.
- "Search the cell containing the query point." → You must include neighbouring cells (or a covering of
  the circle), or you'll miss nearby points across the boundary.
- "Geohash cells are squares of equal size." → They're lat/lng rectangles that narrow toward the poles and
  alternate aspect ratio by length.
- "Euclidean distance on lat/lng is fine." → A degree of longitude shrinks with latitude; use haversine or
  a projected approximation.
- "The cell index gives the answer." → It gives candidates; an exact distance filter is always required.
- "The first k results found are the k nearest." → Only if the searched area is guaranteed to contain the
  k-th nearest; otherwise expand.

---

## Further reading (supplements)

- S2 Geometry documentation (s2geometry.io), especially "S2 Cells" and "S2 Region Coverer".
- Gustavo Niemeyer's original geohash description; the Wikipedia article on Geohash (precision table, examples).
- Samet, *Foundations of Multidimensional and Metric Data Structures* (quadtrees, k-d trees, R-trees).
- Guttman, "R-Trees: A Dynamic Index Structure for Spatial Searching" (1984).
- Redis documentation on `GEOADD` / `GEOSEARCH` (geohash-in-sorted-set implementation).
- Uber Engineering, "H3: Uber's Hexagonal Hierarchical Spatial Index" (2018).

---

## Questions

1. Explain precisely why a composite B-tree index on `(lat, lng)` doesn't make "points within 1 km" fast.
2. How does geohash turn a 2-D point into a 1-D key, and why does a shared prefix mean "same cell"?
3. Why does a geohash radius search query 9 cells instead of 1? Draw a case where searching 1 cell gives a wrong answer.
4. You pick geohash length 7 (~150 m cells) for a 2 km radius search with 9 cells. What goes wrong? What about length 3 (~156 km)?
5. Two cafés are 10 m apart but their geohashes share no prefix. How is that possible?
6. How does a quadtree adapt to uneven density, and why does that matter for query cost?
7. Why might a ride-hailing system prefer a fixed cell grid over a quadtree for tracking drivers?
8. What problems with geohash does S2 solve, and how (cube projection, Hilbert curve)?
9. What is an S2 region covering, and what trade-off does `max_cells` control?
10. Why is "all descendants of a cell form a contiguous ID range" such a useful property?
11. Describe a correct kNN search ("10 nearest drivers") using fixed cells. What's the subtle stopping condition?
12. A bounding-box query for a point at longitude 179.95° returns nothing. Why?
13. Why is the exact-distance filter always needed, and why is Euclidean distance on degrees wrong?
14. You shard a geo index by coarse cell ID. What benefit do you get, and what failure mode should you expect?

## Answers

<details>
<summary>Show answers (attempt the questions first)</summary>

**1.** The index is sorted by `lat`, then by `lng` within equal `lat`. The query has a range on `lat`, so the index can only narrow to the latitude band; within that band entries are ordered by `lat` and the `lng` values of matching points are scattered throughout. The engine scans every entry in the band (which wraps around the world) and filters `lng` per row. The leftmost-prefix rule: after a range on the first column, later columns can't be used for seeking.

**2.** It repeatedly halves the longitude and latitude ranges alternately, recording one bit per halving (interleaving the bits: a Z-order curve), then encodes 5 bits per base-32 character. Each character refines the previous cell into 32 sub-cells, so all points inside a cell share the cell's geohash as a prefix. A prefix query is therefore a range scan on a sorted index.

**3.** Because a point near the edge of its cell can be within the radius of points in the adjacent cell. Example: Q sits 5 m from the eastern edge of cell A; P sits 5 m inside cell B to the east. Searching only cell A misses P although it's 10 m away. Searching the 3×3 block covers the circle as long as the cell is at least as large as the radius.

**4.** Length 7: the 3×3 block spans only ~450 m, so the 2 km circle extends far beyond it: many true results are missed (incorrect). Length 3: correct, but the 9 cells cover ~470 km square, returning a huge number of candidates for the exact-distance filter (slow). Choose the precision whose cell size is just above the radius (length 5, ~4.9 km, or length 6 with more cells at high latitude care).

**5.** Geohash is a Z-order curve: at the top level the world is split at the equator and the prime meridian, so points on opposite sides of those lines differ in the very first bits. Similar large jumps happen at other cell boundaries. Closeness doesn't imply a common prefix.

**6.** It splits a leaf into four only when it exceeds a capacity K, so dense areas get many small leaves and sparse areas few large ones, keeping points per leaf bounded. A query visits only nodes intersecting the search region, and each leaf costs at most K distance checks, so query cost stays predictable in both cities and deserts (a fixed grid would have overloaded cells in cities and empty cells elsewhere).

**7.** Drivers update positions every few seconds. With a fixed grid, an update is a constant-time "if the cell changed, remove from old set, add to new set" on a hash map or Redis set keyed by cell ID, easy to shard and replicate. A quadtree needs pointer updates, splits and merges under heavy write rates and is hard to share across servers. Fixed grids with neighbor queries are simpler and good enough when cells are tuned.

**8.** Geohash works on a flat lat/lng rectangle (cells distort toward the poles, singularities at poles, wrap at the antimeridian) and uses a Z-order curve (big locality jumps). S2 projects the sphere onto a cube's six faces so cells are roughly equal-area everywhere with no pole singularity, and orders cells along a Hilbert curve, which has no jumps within a face, giving better range-scan locality. It also gives integer cell IDs with arithmetic containment and mixed-level coverings.

**9.** A covering is a small set of S2 cells (possibly at different levels) whose union contains a region such as a circle or polygon. Each cell becomes one ID range to scan. `max_cells` trades the number of range scans (fewer cells = fewer scans) against tightness (fewer cells = larger cells = more area outside the region = more candidates to discard).

**10.** It turns "is this point inside this cell?" and "find all points in this cell" into a single integer range check / range scan on any sorted index, at any level, without storing the hierarchy explicitly. Coverings at mixed levels therefore become a handful of B-tree range scans.

**11.** Search the cell containing Q plus neighbours at a fine level; if you have fewer than k candidates, expand (coarser level or next ring) and repeat. Stopping condition: you have at least k candidates **and** the distance to the k-th nearest candidate is no greater than the minimum distance from Q to the boundary of the searched area. Otherwise a nearer point could lie just outside the searched cells.

**12.** The box spans longitudes 179.9 to 180.0 and then continues at -180.0 to -179.9. A naive `lng BETWEEN 179.9 AND -179.9` has min > max and matches nothing. The query must split into two ranges ([179.9, 180] and [-180, -179.9]), and neighbour computation must wrap.

**13.** Cells only approximate the circle: they return a superset (cell corners lie outside the radius). Exact distance removes those false candidates (filter-and-refine). Euclidean distance on degrees treats a degree of longitude as equal to a degree of latitude, but a degree of longitude is 111 km × cos(latitude); at 60° it's half as long. Use haversine (or a local projection).

**14.** Benefit: geographically local queries hit one or a few shards (no scatter-gather), and a region's data is co-located. Failure mode: skew. Dense cities produce hot shards with most of the data and traffic, while ocean and desert cells sit idle (topic 25 hotspot). Mitigate by splitting hot cells into finer cells across shards or using adaptive partitioning.

</details>

## Flashcards

Cover the answer, say it out loud, then check.

**Q:** Why can't a B-tree on (lat, lng) answer radius queries efficiently?
**A:** It's one-dimensional; the range on lat stops lng from narrowing the scan (scans a whole latitude band).

**Q:** What is a space-filling curve used for?
**A:** Mapping 2-D cells to a 1-D order so a sorted index can range-scan regions.

**Q:** Z-order vs Hilbert?
**A:** Z-order interleaves bits (simple, big jumps); Hilbert has no jumps between consecutive cells (better locality).

**Q:** How is a geohash built?
**A:** Alternate halving longitude and latitude, one bit each, interleaved; 5 bits per base-32 char.

**Q:** What does a geohash prefix represent?
**A:** A larger cell containing every point whose geohash starts with it.

**Q:** Approx. cell size for geohash length 5, 6, 7?
**A:** ~4.9 km, ~1.2 × 0.6 km, ~153 m.

**Q:** Why search 9 geohash cells?
**A:** Points near the cell edge can be close to points in neighbouring cells.

**Q:** Rule for choosing geohash precision?
**A:** Cell size just at or above the search radius.

**Q:** Does close imply a shared geohash prefix?
**A:** No; points across the equator or prime meridian can share none.

**Q:** How does Redis implement GEOSEARCH?
**A:** 52-bit geohash scores in a sorted set; range lookups for covering cells, then distance filter.

**Q:** PR quadtree split rule?
**A:** Split a leaf into four quadrants when it exceeds K points.

**Q:** Main advantage of a quadtree?
**A:** Adapts to density: small leaves in dense areas, large in sparse.

**Q:** Main drawback of quadtrees?
**A:** Pointer-based, in-memory, awkward to distribute and to update at high rates.

**Q:** What are R-trees?
**A:** Trees of nested bounding rectangles that index shapes; used by PostGIS via GiST.

**Q:** How does S2 map the Earth?
**A:** Project the sphere onto 6 cube faces, quadtree each face, order cells along a Hilbert curve.

**Q:** S2 cell ID?
**A:** A 64-bit integer: 3 face bits, 2 bits per level, then a sentinel bit; 30 levels to ~1 cm.

**Q:** Why are S2 descendant ranges useful?
**A:** All descendants form a contiguous ID range, so containment is a range check.

**Q:** What is an S2 covering?
**A:** A set of mixed-level cells whose union covers a region; each is one range scan.

**Q:** What does max_cells trade?
**A:** Fewer range scans vs a looser fit (more false candidates).

**Q:** What is H3?
**A:** Uber's hierarchical hexagonal grid; uniform neighbour distances, imperfect nesting.

**Q:** Filter-and-refine?
**A:** Index returns a superset of candidates; exact distance computation decides.

**Q:** Which distance formula for lat/lng?
**A:** Haversine (great-circle) or a geodesic; not Euclidean on degrees.

**Q:** Antimeridian pitfall?
**A:** Boxes crossing ±180° must be split into two longitude ranges.

**Q:** kNN stopping condition with cells?
**A:** k candidates found and k-th distance ≤ guaranteed covered radius; otherwise expand.

**Q:** Hazard of partitioning by geographic cell?
**A:** Dense cities become hot partitions.
