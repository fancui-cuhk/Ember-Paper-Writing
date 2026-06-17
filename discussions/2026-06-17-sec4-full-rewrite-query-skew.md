# §4 Full Rewrite — Spatial Locality Survey + Query-Skew Analysis

**Date:** 2026-06-17  
**Topic:** Self-contained §4 rewrite after PDF scan + seminal-venue cross-check  
**Artifact:** `summaries/partition-sharding-vector-search-survey.md` §4

## User request

1. Scan all local PDFs + seminal-venue literature for geometry-driven placement/routing
2. Make §4 self-contained: scope, master table, per-paper detail (redundancy OK)
3. Center research question: **why colocate near clusters if hot spatial query regions cause load imbalance?** (query skew, not partition-size balance)

## Method

- Text-scanned all PDFs under `related-work/pdfs/` for spatial/locality/colocation/sharding keywords
- Manual review of matches + existing survey entries
- Web cross-check for VStream, SPIRE, SABBSR query-frequency, Quake access skew, RED-ANNS work-stealing

## §4 paper set (14 entries)

**Existing (7):** SABES, SABBS/SABBSR, Distributed LSH, ADBV, GaussDB-Vector, CoTra, Vexless

**Added (7):** SPIRE, VStream, Unleashing Graph Partitioning, RED-ANNS, HARMONY, Quake (single-node NUMA — access-skew reference), LindormVector (industry, PDF N/A)

## Key finding — query-load / hot-region handling

| Handles query skew explicitly? | Papers |
|-------------------------------|--------|
| **Yes (strongest)** | **SABBSR** (probe frequency in placement), **Quake** (split/merge on access patterns), **VStream** (dynamic rebalance on stream drift) |
| **Partial** | SABBS (caps), HARMONY (dimension mode when vector shards hot), RED-ANNS (work-stealing), SPIRE (hot-spot mention + elastic QE), Distributed LSH (hot-range replication) |
| **No / uniform queries assumed** | SABES, ADBV, GaussDB, CoTra, Vexless, Unleashing |

**Consensus gap:** Cluster-scale spatial colocation literature optimizes **inter-node traffic** under **i.i.d. queries**; **moving hot regions** (recommender-style) barely addressed except Quake (single node) and SABBSR (offline bucket frequency).

## PDF organization

All §4 downloaded PDFs moved to `related-work/pdfs/sec4/` (11 files). SABES, Distributed LSH, LindormVector remain `NOT_DOWNLOADED`.

## Open research questions (documented in §4.3)

- Query-trace-aware online reshuffle with colocation constraints
- Colocate vs replicate vs elastic-compute when a region heats up
- Temporal hot-spot shift under spatial sharding

## Relation to prior discussions

- Supersedes §4 structure in `2026-06-17-partition-survey-sec4-sabes-table-fix.md` for content (PDF folder move retained)
