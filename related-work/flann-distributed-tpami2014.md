# FLANN Distributed Search (Muja & Lowe, TPAMI 2014)

## Bibliography

| Field | Value |
|-------|--------|
| **Title** | Scalable Nearest Neighbor Algorithms for High Dimensional Data |
| **Authors** | Marius Muja, David G. Lowe |
| **Venue** | IEEE TPAMI, 2014 |
| **PDF** | [UBC FLANN](https://www.cs.ubc.ca/research/flann/uploads/FLANN/flann_pami2014.pdf) |
| **Read date** | 2026-06-18 |

---

## Internal reading note — Ember relevance: **low (anti-pattern reference)**

**TL;DR:** Distributed FLANN is structurally the same as the **SIGMOD 2025 block-based serverless** paper: **equal split + query every shard**. Query skew over embedding hotspots is **not addressed** — it is **avoided** by never doing subset routing.

---

## Mechanism (§5)

- Partition data into **disjoint equal subsets**; independent index per subset.
- **Broadcast** each query to **all** MPI workers; merge partial top-k.

**Verbatim:**

> "During search the query is broadcast to all the indexes and each of them performs the nearest neighbor search within its associated data."

> "The data is distributed equally between the machines … each of them will only have to index and search 1/N of the whole data set"

---

## Comparison — FLANN MPI vs block-based serverless (SIGMOD 2025)

| | **FLANN (2014)** | **Block serverless (2025)** |
|---|------------------|----------------------------|
| **Partition** | Equal disjoint subsets | Equal byte-range blocks |
| **Query path** | All shards / all indexes | All partitions (MapReduce) |
| **Geometry** | k-means tree is **search index**, not placement for routing | Explicitly **anti-geometry** |
| **Skew stance** | Silent — uniform work per query by construction | Stragglers from **unbalanced k-means** motivate blocks |
| **Trade-off** | No prune → no hotspot from routing | No prune → no hotspot from routing |

**Ember take:** Useful as **baseline lineage** (“fan-out-all avoids skew by giving up subset routing”). Not a competitor design for spatial colocation under skew — same structural choice as block partitioning.

**Contrast:** Aly distributed k-d tree (cited in FLANN) routes to **subset** of leaf trees → throughput win but **implicit hotspot risk** if queries cluster — the problem Ember cares about.

---

## Related repo files

- [`related-work/serverless-block-partitioning-sigmod2025.md`](serverless-block-partitioning-sigmod2025.md)
- [`summaries/query-skew-spatial-partitioning-survey.md`](../summaries/query-skew-spatial-partitioning-survey.md) §11
