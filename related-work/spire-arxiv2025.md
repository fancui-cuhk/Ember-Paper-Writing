# SPIRE: Scalable Distributed Vector Search via Accuracy Preserving Index Construction

## Bibliography

| Field | Value |
|-------|--------|
| **Title** | Scalable Distributed Vector Search via Accuracy Preserving Index Construction |
| **Authors** | Yuming Xu, Qianxi Zhang, Qi Chen, Baotong Lu, Menghao Li, Philip Adams, Mingqin Li, Zengzhong Li, Jing Liu, Cheng Li, Fan Yang |
| **Venue** | arXiv:2512.17264 (2025); VecDB @ ICML 2025 workshop |
| **PDF in repo** | [`pdfs/spatial-related/spire.pdf`](pdfs/spatial-related/spire.pdf) |
| **Read date** | 2026-06-17 |

---

## Internal reading note — index family (author judgment)

**TL;DR:** SPIRE is a **hybrid HNSW + IVF** design. **In essence it is IVF / partition-based** (vectors live in k-means leaf partitions on SSD; query cost is dominated by **which partitions you read**). **On top**, each non-leaf level builds a **proximity graph index (HNSW-class) over partition centroids** so the query **navigates the hierarchy** to the right IVF clusters before probing leaves.

Do **not** read SPIRE as “distributed global HNSW” (RED-ANNS / SHINE line). It explicitly **rejects** sharding one dense HNSW graph across nodes (>80% cross-node hops in their Table 1). It also **rejects** naive coarse IVF where fidelity loss forces probing many partitions (Figure 1: query mis-routed to wrong centroid → extra partition reads kill throughput).

---

## Architecture (how the hybrid decomposes)

```
                    ┌─────────────────────────────┐
  Query ──────────► │ Top level: graph on centroids│  ← HNSW / proximity graph
                    │ (in memory, replicated)      │     (navigate to region)
                    └──────────────┬──────────────┘
                                   │ cross-node only between levels
                    ┌──────────────▼──────────────┐
                    │ Mid levels: same pattern   │  ← recursive: graph on centroids
                    │ (accuracy-preserving build)  │     of lower-level partitions
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
  Leaf L0 ─────────►│ IVF-style partitions       │  ← vectors + local index on SSD
                    │ (balanced granularity G)   │     partition probes / reads
                    └─────────────────────────────┘
```

**Search path:** top-down graph traversal on centroids level-by-level → narrow to relevant leaf partition(s) → read vectors from SSD.

**Storage ops:** only **top-level** graph kept in memory (stateless compute tier, replicable); lower levels + vectors on SSD.

---

## What problem they optimize (paper framing)

Two failure modes of distributed ANNS at billion scale:

| Approach | Latency | Throughput / reads |
|----------|---------|-------------------|
| **Sharded global graph (HNSW across nodes)** | Bad — cross-node graph hops dominate | Fewer partition reads, but remote traversal expensive |
| **Partition routing only (IVF / DSPANN / ADBV-style)** | Better — route to few centroids | Bad — **fidelity loss** at boundaries → must probe many partitions → read amplification |

SPIRE’s two design decisions:

1. **Balanced partition granularity** — pick k-means partition density just before the inflection where Recall@k requires “excessive” extra partition probes (metric: **partition density**).
2. **Accuracy-preserving recursive construction** — build multi-level hierarchy; each level is a “single-level vector index” over **centroids of the level below**, with construction tuned so **end-to-end** accuracy is preserved (not optimizing each level in isolation).

Core trade-off they name: **query latency (cross-node comm)** vs **throughput (vector reads / IO)** at fixed recall.

---

## Relation to other systems

| System | Same as SPIRE? | Difference |
|--------|----------------|------------|
| **SPANN / DSPANN** | Yes — partition + hierarchical routing | SPIRE adds **recursive graph-over-centroids** levels + **balanced granularity** + accuracy-preserving multi-level build |
| **ADBV / Pinecone / SPTAG** | Yes — centroid hierarchy for routing | SPIRE foregrounds **read-cost vs cross-node** trade-off; production-scale 8B / 46 nodes eval |
| **RED-ANNS / CoTra / SHINE** | **No** — those preserve **one navigable graph over vectors** | SPIRE **does not** maintain a single global HNSW over all vectors |
| **Milvus segment sharding** | Distant cousin — segment = coarse partition | SPIRE tunes **leaf partition size** explicitly for read cost |

**Survey placement note:** SPIRE fits **§4** (centroid hierarchy routing, colocate neighboring clusters) **and** read-amp survey (partition read cost). Primary identity = **hierarchical IVF with graph navigation layers**, not graph-first distributed ANNS.

---

## Query-skew / hot spots (secondary)

- **Global partition IDs** + **boundary vector replication** mentioned for skewed workloads.
- **Elastic stateless query engines** scale compute separately.
- **No** query-trace-driven repartition in paper.

---

## Open questions / follow-ups

- At leaf L0, is local search brute force, small graph, or PQ scan? (Paper: partitions on disk; local index details in §implementation.)
- Compare directly to **C-SPANN** / **GaussDB two-layer k-means IVF** — same hybrid family, different storage tier.
- Ember angle: SPIRE optimizes **partition read count × partition size** — aligns with read-amplification survey §3.C / §3.D tension.

---

## Links in this repo

- Partition survey §2 + §4: [`summaries/partition-sharding-vector-search-survey.md`](../summaries/partition-sharding-vector-search-survey.md)
- Distributed line (prior “loosely graph” note): [`summaries/distributed_vector_search_related_work.md`](../summaries/distributed_vector_search_related_work.md)
- Read amplification (partition reads): [`summaries/read-amplification-disk-vector-search-survey.md`](../summaries/read-amplification-disk-vector-search-survey.md)
