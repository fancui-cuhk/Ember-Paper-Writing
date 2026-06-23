# Distributed ANNS Taxonomy v2 — Distilled Terminology

**Date:** 2026-06-16  
**Topic:** User refinements to placement codes, vocabulary, decision tree, LindormVector/SPIRE/C-SPANN  
**Related:** [`summaries/distributed-anns-taxonomy.md`](../summaries/distributed-anns-taxonomy.md), prior [`2026-06-16-distributed-anns-taxonomy-review.md`](2026-06-16-distributed-anns-taxonomy-review.md)

---

## User decisions (consensus)

1. **Merge R1 + R2 → B1 Random:** Hash, block, striping, tenant/partition keys are all **vector-agnostic** — splits do not use embedding geometry.
2. **Eliminate R5; fold into B3 Query-trace-informed:** Harmony’s placement refinement from **past queries** and cost-model `B_vec` tuning is **offline/trace-based**, not a separate “query-time overlay” category. Ignore Harmony online optimization for taxonomy.
3. **Keep B4 Dimension split (old R6):** Harmony dimension shards remain a distinct placement geometry.
4. **Distilled survey vocabulary:** Stop paper-specific jargon in the main tree (shard vs partition vs striping). Use: random, spatial colocation, query-trace-informed, dimension split. **Do not use** “query-time overlay.”
5. **Decision tree:** Under **index-first**, **placement is mandatory** alongside index family — not optional.
6. **Deep dives requested:** LindormVector, SPIRE, C-SPANN partitioning strategies documented in taxonomy §3.

---

## Key clarifications

### LindormVector

- **Index-first** IVFPQ; **spatial colocation** at posting-list ↔ KV range alignment.
- Base Lindorm row sharding may be **random** w.r.t. vectors; vector engine **aligns IVF lists** to existing ranges.

### SPIRE

- **Index-first** hierarchical **spatial colocation** (recursive k-means + graphs on centroids).
- **Subset search** via top-down tree walk; boundary replication for skew — **not** query-trace-informed.

### C-SPANN

- **Index-first** SPANN/SPFresh tree mapped to **SQL ranges** — **spatial colocation**.
- **Subset search** on ranges; SPFresh freshness story.

### Index-first + placement

User correctly noted v1 under-emphasized placement under index-first. Same index family (e.g. global graph) differs sharply between **random striping** (DistributedANN) vs **spatial colocation** (CoTra) vs **query-trace-informed** (Harmony).

---

## Open items

- Whether Pinecone cross-namespace routing deserves a footnote (tenant B1 + in-namespace B2).
- Harmony per-query vector vs dimension **execution** — document as runtime policy under B3/B4 plan, not placement leaf.
- Optional fourth survey dimension: storage tier / fan-out vs subset as explicit table column.

---

## Files updated

- `summaries/distributed-anns-taxonomy.md` — full v2 rewrite
