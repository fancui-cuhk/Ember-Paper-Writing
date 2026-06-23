# GP-ANN: Unleashing Graph Partitioning for Large-Scale Nearest Neighbor Search

## Bibliography

| Field | Value |
|-------|--------|
| **Short name** | **GP-ANN** |
| **Title** | Unleashing Graph Partitioning for Large-Scale Nearest Neighbor Search |
| **Authors** | Lars Gottesbüren, Tobias Grohser, Sebastian Schlag, Christian Schulz, Henning Meyerhenke |
| **Venue** | PVLDB 18(6): 1649–1662, 2025 |
| **PDF** | [PVLDB](https://www.vldb.org/pvldb/vol18/p1649-gottesbueren.pdf) |
| **PDF in repo** | [`pdfs/gp-ann.pdf`](pdfs/gp-ann.pdf) |
| **Read date** | 2026-06-17 |

---

## Internal reading note — problem & setting

**TL;DR:** GP-ANN is **Category #1** (many shard-local indexes): **balanced graph partition** into shards, each holding a local **HNSW**, plus **modular routing** so a query touches only **few near shards**. It is **not** full-GPS (one logical graph over RDMA) like RED-ANNS.

| Component | What GP-ANN does |
|-----------|------------------|
| **Placement** | **METIS-style balanced graph partition** — minimize cross-shard edges while balancing vertex count per shard |
| **Per-shard index** | Local **HNSW** on each partition's vertices |
| **Routing (kRt)** | Hierarchical **k-means centers** — query routed to shards whose representatives are nearest |
| **Routing (hRt)** | **LSH** buckets — query routed to shards matching hash buckets |

Graph partitioning preserves **neighbor locality** within shards; decoupled routers (kRt/hRt) pick which shards to probe. Reported up to **2.14× QPS @ 90% recall@10** vs prior billion-scale methods on in-memory multi-shard setup.

---

## Relation to RED-ANNS / sub-GPS vs full-GPS

| | GP-ANN | RED-ANNS |
|---|--------|----------|
| **Graph model** | **Shard-local subgraphs** + routing | **One logical full graph** via RDMA |
| **Query path** | Route to k/near shards → local HNSW search | Traverse global graph with remote hops |
| **vs Milvus sub-GPS** | Same broad family (segment-local HNSW) but **routing reduces shards touched** | Explicitly rejects sub-GPS / MapReduce-style cuts |

See [`red-anns-vldb2026.md`](red-anns-vldb2026.md) comparison table.

---

## §4 survey placement

Listed in partition survey **§4** (centroid/spatial locality): graph cut + k-means/LSH routing sends queries to shards whose **representatives are near the query**. **Query skew / hot regions not addressed** — balanced vertex partition; hot query clusters still hit the same center-neighborhood shards.
