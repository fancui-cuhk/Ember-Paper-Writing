# Distributed ANNS Taxonomy v4

**Date:** 2026-06-16  
**Related:** [`summaries/distributed-anns-taxonomy.md`](../summaries/distributed-anns-taxonomy.md)

---

## Changes

1. **Empty leaves removed** from decision tree (Graph+query-history, Graph+dimension, IVF+random, partition-first+query-history).
2. **Dimension split** → sidenote only (Harmony coordinate subspace); not a tree branch.
3. **SPANN** moved to **index-first → IVF/LSH → query-history-aware** with SABBSR and Harmony; removed from partition-first (prior §4.3 misclassification).
4. **Naming:** second axis = **node placement strategy**; **spatial colocation placement**; **query-history-aware placement**.

## Consensus

SPANN is index-first (one logical inverted index; distributed layout bin-packs **posting lists**, not independent per-shard indexes).

## Audit

51 assets: 33 in tree, 17 single-node out-of-tree, 1 overlay (NetANNS). All classified.
