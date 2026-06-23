# Distributed ANNS Taxonomy v3 — Minimal Survey Tree

**Date:** 2026-06-16  
**Topic:** Remove “hybrid”; collapse index types; system-names-only tree; partition sizes; completeness audit  
**Related:** [`summaries/distributed-anns-taxonomy.md`](../summaries/distributed-anns-taxonomy.md)

---

## User decisions

1. No **“hybrid”** — classify by **main index type** (graph-on-centroids → IVF).
2. **Graph = disk-graph**; **IVF ≈ LSH** — two index types only.
3. Decision tree: **system names only**, no caveats in tree body.
4. Index-first: **two-level** branch — index type → **placement**.
5. Completeness: all 51 `distributed-anns-related` assets mapped — 32 in tree, 18 single-node out-of-tree, 1 overlay (NetANNS).

## Partition-first sizes (summary)

Milvus **~512 MB** segments is the best-documented industrial constant. Pinecone slabs and Weaviate/Qdrant shards have **no fixed MB**. Block-serverless uses **equal byte blocks** with operator-chosen **P**. Vexless shards target **~1.5 GB**/function. ADBV uses **512** logical centroid partitions (eval), not a byte cap.

## Completeness gaps

- **Empty leaves:** Graph+query-trace, Graph+dimension, IVF+random — none in corpus.
- **ADBV** moved to **partition-first → spatial** (local IVFPQ per k-means shard), despite repo survey §2 tag — functionally A1.
- **spann.pdf** appears twice: main paper out-of-tree (single-node); **§4.3** in-tree under partition-first query-trace.

## Open

- Pinecone: only under spatial (slab geometry); namespace tenant routing folded into “random” footnote if needed in prose.
- Harmony listed under IVF query-trace **and** dimension split (same system, two placement modes).
