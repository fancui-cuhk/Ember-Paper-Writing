# RED-ANNS: An RDMA-Enabled Distributed Framework for Graph-Based ANN Search

## Bibliography

| Field | Value |
|-------|--------|
| **Title** | RED-ANNS: An RDMA-Enabled Distributed Framework for Graph-Based Approximate Nearest Neighbor Search |
| **Authors** | Yue Chen, Kai Zhang, Sipeng Chen, Shihai Xiao, Xiaomin Zou, Ren Ren, Yinan Jing, X. Sean Wang, Li Cao, Mingxiang Wan |
| **Venue** | PVLDB 19(3): 399–412, 2025 |
| **DOI** | [10.14778/3778092.3778101](https://doi.org/10.14778/3778092.3778101) |
| **PDF in repo** | [`pdfs/red-anns.pdf`](pdfs/red-anns.pdf) |
| **Artifact** | [github.com/cheenyuee/RED-ANNS](https://github.com/cheenyuee/RED-ANNS.git) |
| **Read date** | 2026-06-17 |

---

## Internal reading note — problem & setting (author judgment)

**TL;DR:** RED-ANNS is in the **same problem family** and **same distributed graph setting** as DistributedANN, SHINE, d-HNSW, CoTra, BatANN, etc.: **one logical proximity graph** (HNSW/Vamana-class) spread across machines — **not** “partition data → build N independent shard-local graphs → broadcast + merge.”

The paper’s own vocabulary makes this explicit:

| Baseline (what they reject) | RED-ANNS (what they want) |
|----------------------------|---------------------------|
| **Segment / sub-GPS** — dataset split into segments; **HNSW built per segment independently**; query **broadcast** to all nodes; local top-k per segment (map) + global merge (reduce) | **Full-GPS** — **logically full graph** in a **shared memory space** across nodes; search traverses the global graph via **RDMA** |
| MapReduce-style distributed ANNS (Milvus-like segment sharding cited) | Graph-preserving distributed search with remote hops hidden by RDMA + scheduling |

So the core architectural fork is:

> **Many indexes (Category #1 pattern applied to graphs)** vs **one index, externally placed (Category #2)**

RED-ANNS is firmly on the **Category #2** side — same “partition **one** graph instead of building many” thesis as our partition survey §2 row.

---

## What problem they say they solve

1. **Scale beyond single-node DRAM** for in-memory graph ANN (terabyte-scale embeddings).
2. **Reject MapReduce-style graph sharding** because:
   - Smaller / more segments → **indexing efficiency drops** (paper’s Fig. 1: up to ~3× distance-compute overhead with 8 segments/node on DEEP).
   - Each segment gets a **weaker local subgraph** → recall / search quality suffers unless you probe many segments (still wasteful).
3. **Make full-graph search feasible on a cluster** despite cross-node hops — RDMA + placement + scheduling + relaxed BFS, not reverting to independent subgraphs.

Persistent disk ANN (DiskANN) is mentioned only as a latency trade-off (4.2–6.4× slower); the **target deployment is in-memory distributed graph**.

---

## Techniques (how full-GPS is made to work)

1. **Locality-aware data placement** — vectors placed so graph traversal tends to stay near the query’s converging region (paper ties this to ANNS “gradually converging near target”).
2. **Affinity-based query scheduling** — route query to the node that already owns query-proximate vectors.
3. **Dependency-relaxed best-first search** — overlap / hide RDMA latency vs strict hop-by-hop BFS ordering.
4. **Affinity-based work stealing** — when load imbalanced, steal queries but **prioritize locality** (trade-off vs pure spatial affinity).

Index construction uses a **GPS (graph-preserving sharding)** strategy — goal is **not** to cut the graph into disconnected shard indexes.

---

## Relation to sibling papers (same setting, different knobs)

All of these share: **one navigable graph, data/index state split across machines, avoid naive graph cut + merge.**

| Paper | Same problem? | How it differs from RED-ANNS |
|-------|---------------|----------------------------|
| **DistributedANN** | Yes — single DiskANN-scale graph over 1000+ nodes | KV/disk-backed global graph; different storage tier |
| **SHINE** | Yes — HNSW on disaggregated memory | Graph-preserving partition + **joint cross-compute cache** |
| **d-HNSW** | Yes — RDMA disaggregated memory HNSW | Balanced clustering + representative index + pipelined RDMA/compute |
| **CoTra** | Yes — one global Vamana graph, k-means machine partitions | Pull-Push RDMA; coordinator partition per query |
| **BatANN** | Yes — distributed **disk** graph, one logical index | Baton-passing pipeline across nodes |
| **GP-ANN** | Partially — shard-local HNSW but **kRt/hRt routing** to few shards | Accepts shard-local subgraphs + modular routing; not full-GPS over RDMA |
| **Milvus / segment MapReduce** | **No** — explicit **sub-GPS** baseline in RED-ANNS eval | Independent per-segment indexes + all-shard probe |

**Takeaway for Ember writing:** RED-ANNS is not a new partition *primitive* (like SABES centroid colocation in §4). It is another **Category #2 execution stack** paper: preserve graph connectivity, pay cross-node access cost, optimize with **network + placement + query scheduling**.

---

## §4 (spatial colocation) — secondary angle only

Survey currently lists RED-ANNS in §4 because **locality-aware placement** and **affinity scheduling** use embedding-space proximity. That is **real but not the paper’s primary identity**. Primary identity = **full-GPS vs sub-GPS** (one graph vs many).

Query-skew: work-stealing when affinity overloads nodes — **partial** relief, does not reshuffle vector data.

---

## Open questions / follow-ups

- How much does RED-ANNS’s GPS placement resemble CoTra’s balanced k-means vs SHINE’s graph-preserving cut?
- Eval baselines are MapReduce-style and OSS vector DBs — direct comparison to SHINE/d-HNSW/CoTra on same hardware would clarify “same setting, who wins on what axis.”
- Cold / disaggregated-storage path not the focus (in-memory RDMA cluster).

---

## Links in this repo

- Partition survey §2 entry + §4 detail: [`summaries/partition-sharding-vector-search-survey.md`](../summaries/partition-sharding-vector-search-survey.md)
- Distributed graph line overview: [`summaries/distributed_vector_search_related_work.md`](../summaries/distributed_vector_search_related_work.md)
