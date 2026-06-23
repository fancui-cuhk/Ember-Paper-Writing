# Taxonomy v5 — Placement-first rename

**Date:** 2026-06-16  
**Related:** [`summaries/distributed-anns-taxonomy.md`](../summaries/distributed-anns-taxonomy.md)

---

## Decision

Rename **partition-first** → **placement-first**.

## Rationale

- “Partition” collides with IVF k-means partition, Lucene segment, SQL range.
- Aligns with **node placement strategy** as the shared level-2 axis.
- Pairs cleanly with **index-first**: placement defines index boundaries vs index defines what gets placed.

## Disambiguation (survey prose)

- **Level 1 — organization:** placement-first vs index-first  
- **Level 2 — node placement strategy:** random / spatial colocation / query-history-aware  

Both branches use level 2; only placement-first makes level 2 **define** separate local indexes.
