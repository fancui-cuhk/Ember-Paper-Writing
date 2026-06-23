# Session: Distributed ANNS Taxonomy Review

- **Date:** 2026-06-16
- **Topic:** User three-dimensional taxonomy (partition vs index first; placement; IVF vs HNSW) — validate, complete, polish

## User draft (summary)

Dims: (1) index-first vs partition-first, (2) clustering vs random vs query-aware placement, (3) IVF vs HNSW. Decision tree with ADBV, Milvus/block, DistributedANN, CoTra, SABES, SPANN/HARMONY.

## Consensus

- **Dim 1 is correct** and should anchor the survey (maps to repo #1 / #2).
- **Dim 2 needs refinement:** split **query-aware** into **R4 workload-aware placement** (SPANN §4.3) vs **R5 query-time routing** (HARMONY, LEQAT, ADBV optimizer). Add **scalar/tenant**, **graph partition**, **LSH**, **dimension shard**.
- **Dim 3 too narrow:** add disk graph, LSH, hybrid (SPIRE, HAKES); for partition-first, index type is **orthogonal**.
- **Misplacements:** HARMONY ≠ IVF query-aware partition; SPANN main paper ≠ index-then-partition; ADBV is partition-first + subset routing (also listed in survey §2 — technique is per-partition index).

## Missing systems called out

RED-ANNS, SHINE, d-HNSW, BatANN, GP-ANN, GaussDB, Vexless, VStream, LindormVector, SPIRE, C-SPANN, Haghani LSH, Pinecone, industrial MDs, FLANN, HAKES, pyramid, replication axis.

## Deliverable

- [`summaries/distributed-anns-taxonomy.md`](../summaries/distributed-anns-taxonomy.md) — polished tree, full asset mapping, per-category motivation/pros/cons.

## Open

- Whether to add storage-tier and update-model as explicit survey dimensions (recommended as §2.1 footnote table).
