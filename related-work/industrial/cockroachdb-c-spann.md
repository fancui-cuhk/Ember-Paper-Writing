# CockroachDB C-SPANN — Production Architecture Analysis

**Type:** Industrial system (product blog + distributed SQL integration; preview as of v25.2)  
**Last updated:** 2026-06-16  
**Survey category:** `distributed-anns-related`

---

## Sources (primary)

| Source | URL |
|--------|-----|
| Introducing Distributed Vector Indexing | https://www.cockroachlabs.com/blog/distributed-vector-indexing-cockroachdb/ |
| pgvector background (24.2) | CockroachDB release notes / docs |

Repo: [`summaries/partition-sharding-vector-search-survey.md`](../../summaries/partition-sharding-vector-search-survey.md) — C-SPANN entry §2.

---

## Executive summary

CockroachDB adds **C-SPANN** (Cockroach-SPANN): **SPANN/SPFresh-style** disk-friendly ANN adapted to **distributed SQL ranges**. Vectors live in transactional tables (pgvector-compatible since 24.2); C-SPANN provides **secondary ANN index** with **incremental freshness**, **range-based partitioning** of the index tree, and **no central query coordinator**. Goal: combine **OLTP + vector ANN** without a separate vector DB or brute-force O(N) scan.

---

## Prior state (24.2)

- **pgvector** types and distance functions in distributed SQL.  
- Search = **full table / index scan** — O(N) — unusable at millions+ vectors.

---

## C-SPANN design goals (vendor-stated)

| Goal | Mechanism direction |
|------|---------------------|
| **Accuracy** | Tunable recall@k; target ~99% at ~1M scale with reasonable resources |
| **Freshness** | Incremental updates like SPFresh — inserts/deletes visible without full rebuild; **no single-node coordinator** |
| **Scalability** | Disk-oriented; map index to **Cockroach ranges**; splits/merges work with normal range lifecycle |
| **Latency** | Low tens of ms acceptable for distributed disk SQL |
| **Resource** | **RaBitQ** compression — small index footprint (important for **serverless** SQL pods that cannot rebuild multi-GB RAM structures on cold start) |
| **Build** | Background index build without starving foreground OLTP |

---

## Distributed storage mapping

- **Hierarchical k-means tree** (SPANN lineage) mapped to **CockroachDB ranges** (contiguous row key spans).  
- **Partitions = ranges** — automatic distribution, replication, and rebalancing via existing SQL machinery.  
- Design avoids **unsplittable hot ranges** and **query hotspots** from naive vector clustering (vendor claim).  
- **Disk-based** posting/list storage at scale (billions of vectors target).

Core data structure (blog): **k-means tree** powering partition boundaries — spatial structure at **index** layer, stored in **SQL range** layer.

---

## Indexing

- Adapted from **Microsoft SPANN + SPFresh** (LIRE-style updates, split/merge clusters).  
- **RaBitQ** for compact representations → faster build and search, less RAM duplication on SQL nodes.  
- Index maintenance integrated with **Cockroach changefeeds / background jobs** mental model (details in preview docs).

---

## Query processing

- ANN query issued as **SQL** with vector index hint / operator (preview API).  
- **Predictable small number of serial network round-trips** across ranges (vendor design goal).  
- **No requirement** to route all queries through one vector coordinator node.  
- Combines **filters, joins, aggregations** with vector search in one query plan.

---

## Distributed ANN pattern

| Dimension | C-SPANN |
|-----------|---------|
| Layer | **#2-ish** — one logical SPANN-style index partitioned by **ranges** |
| Routing | **Tree / centroid navigation** → subset of ranges (not fan-out-all) |
| OLTP | **Same store** as transactional data |
| Freshness | **First-class** (SPFresh lineage) vs static SPANN paper |

Closest academic relatives: **SPANN**, **SPFresh**, **Cosmos DB sharded DiskANN** — but mapped to **CRDB ranges** instead of Cosmos physical partitions.

---

## Preview caveats

- Public preview (25.2); not all stated goals fully shipped at announcement.  
- No independent benchmark paper in repo PDF set — cite **blog + product docs** only.  
- Recall/latency numbers in blog are **targets**, not reproduced here as measured facts.

---

## Relation to Ember

- Example of **IVF/posting-on-disk + range sharding** inside operational DB — aligns with Ember’s “integrate with storage layer” narrative vs standalone Milvus scatter-gather.  
- **Freshness + range splits** contrast with sealed-segment Milvus compaction latency.

---

## Bibliography

- Bressler, *Introducing Distributed Vector Indexing to CockroachDB*, Cockroach Labs Blog, June 2025.  
- Microsoft Research: *SPANN* (NeurIPS 2021), *SPFresh* (SOSP 2023) — algorithmic lineage.
