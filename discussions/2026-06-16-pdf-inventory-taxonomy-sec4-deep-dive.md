# 2026-06-16 — PDF Inventory, Taxonomy Tags, §4 Deep Dive

- **Date:** 2026-06-16
- **Topic:** Local PDFs, per-entry classification, #1+#2 vs #3, §4 centroid-locality analysis

## User guidance

Failed PDF downloads are acceptable — proceed with remaining papers.

## Completed

1. **Local PDFs:** ~48 files in `related-work/pdfs/`; survey entries mark **Local PDF** path or `NOT_DOWNLOADED`.
2. **Per-entry fields:** **Category** (1 / 2 / 3 / 2,3 / §4), **Deployment scope** (multi-node vs single-node).
3. **Overview:** Unified **#1 + #2** as **systems partition** (distributed OLTP-style load distribution) vs **#3** index partition.
4. **Multi-node question:** Documented that #1/#2 are not all multi-node — segments/slabs/shard keys apply at single-node scale too.
5. **§4 expanded:** Per-paper analysis — not all IVF; not all colocate adjacent centroids; mechanisms and motivations.

## §4 consensus

| Paper | IVF? | Colocate near centroids? | Primary why |
|-------|------|--------------------------|-------------|
| SABES | IVFADC | Yes | Fewer nodes per query |
| ADBV | Sharding uses k-means; index is VGPQ | Route to near partitions | Avoid all-node fan-out |
| GaussDB-Vector | Yes (IVF path) | Yes (distance sharding) | DN locality + routing |
| HAKES | Yes | Optional (IVF-assignment refine) | Cut refine network traffic |
| SPANN distributed | Yes | Partial (pruning + bin-pack) | Fewer machines + load balance |
| CoTra | No (graph) | Partial (k-means vectors) | Access locality for graph |
| Vexless | Per-shard IVF/LSH/HNSW | Route-only | Serverless cost |
| Distributed LSH | LSH | Yes | Fewer peer hops |
| Faiss distributed | Yes | No | Vertical slice scale-out |
| LEQAT | Yes | N/A (query opt only) | nprobe budget |

## NOT_DOWNLOADED

- LindormVector (ACM 403) — see `related-work/lindorm-vector-sigmod2026.md`
- Distributed LSH EDBT 2009 (ACM 403)
- CockroachDB C-SPANN (blog)
- Milvus production docs

## Relation

- Continues `discussions/2026-06-16-partition-survey-taxonomy-abstract-fix.md`
