# 2026-06-16 — Partition Survey Taxonomy & Abstract Fix

- **Date:** 2026-06-16
- **Topic:** Reclassify partition/sharding survey; fix paraphrased abstracts; IVF centroid spatial proximity
- **Output:** `summaries/partition-sharding-vector-search-survey.md`

## User request

1. Previous agent wrote **summaries** labeled as "abstract" — user wanted **verbatim paper abstracts**.
2. Old four categories (partition-first, graph-first+shard-local, global graph, adaptive) were confusing.
3. Replace with user's three-layer model plus research on **IVF cluster centroid spatial proximity** for external sharding.

## Consensus — new taxonomy

| # | User intent | Survey section |
|---|-------------|----------------|
| **1** | Partition **data** first → build **independent index** per shard (IVF or HNSW) | §1 |
| **2** | **One logical index** sharded across nodes (DistributedANN, HARMONY, Faiss distributed IVF) | §2 |
| **3** | **Partition-intrinsic** index (IVF, LSH, k-means tree) — partitioning required by algorithm | §3 |
| **4** | Cross-cutting analysis: centroid/bucket **spatial proximity** for **external** placement | §4 |

**Superseded:** prior "adaptive" as a top-level category — Quake/Ada-IVF/SPFresh are **§3** (dynamic IVF maintenance), not a separate architectural layer.

## Abstract fix

- Replaced paraphrased text with **official arXiv abstracts** (API) for all arXiv-linked papers.
- Fixed **SPFresh**, **SPANN**, **ADBV** from paper PDFs.
- Product entries remain **Abstract (from docs)** / **(from wiki)**.
- Remaining PVLDB/SIGMOD/USENIX entries without arXiv mirror flagged in overview for manual publisher verification.

## IVF centroid spatial proximity (§4 findings)

**Yes — several works use this insight**, but mostly for **routing/placement after** IVF/LSH indexing, not as the default production sharding policy:

- **SABES/SABBS** — clearest statement: colocate spatially nearby ANN buckets on same node.
- **ADBV, GaussDB-Vector, HAKES, CoTra, Vexless** — k-means/IVF-assignment or distance-based sharding/routing.
- **Distributed LSH (EDBT 2009)** — locality-preserving bucket→peer mapping.
- **SPANN production** — query-aware machine dispatch + workload bin-packing (spatial + access history).

**Mostly no:** Milvus/Weaviate/Qdrant default **hash** sharding; graph systems (DistributedANN, RED-ANNS) use **graph** placement not IVF Voronoi structure.

**Gap for Ember:** no widely cited system uses pure **centroid k-NN graph partitioning** as primary shard policy.

## Open

- Fetch verbatim abstracts for remaining non-arXiv papers (HARMONY, Milvus SIGMOD, Quake OSDI, etc.).
- SABES original paper PDF/venue (currently via PMC 2024 survey reference).

## Relation

- **Revises:** `discussions/2026-06-16-partition-based-vector-search-survey.md` (taxonomy superseded)
- **Continues:** `discussions/2026-06-16-paper-bibliography-pdf-abstracts.md`
