# Qdrant — Production Architecture Analysis

**Type:** Industrial system (open-source docs + product articles; no peer-reviewed ANN systems paper)  
**Last updated:** 2026-06-16  
**Survey category:** `distributed-anns-related`

---

## Sources (primary)

| Source | URL |
|--------|-----|
| Collections | https://qdrant.tech/documentation/concepts/collections/ |
| Multitenancy & custom sharding | https://qdrant.tech/articles/multitenancy/ |
| Distributed deployment | https://qdrant.tech/documentation/guides/distributed-deployment/ (conceptual; cluster mode in docs) |

Repo: [`summaries/partition-sharding-vector-search-survey.md`](../../summaries/partition-sharding-vector-search-survey.md) §1.

---

## Executive summary

Qdrant stores **collections** split into **shards** (each with HNSW segments + payload storage). Distributed mode assigns shards to **peer nodes** with **Raft-consensus** cluster metadata. Default sharding uses **collection-defined shard count**; **custom shard keys** pin tenants/regions to shards for **filtered fan-out reduction**. HNSW is built in **segments** when `indexing_threshold` exceeded; optimizer merges/splits segments in background.

---

## Storage model

```
Collection
  └─ Shard (1..N, set at creation)
       └─ Segments (mutable units: plain + HNSW indexed)
            └─ Points (vector + JSON payload + id)
```

- **Points** live in segments; optimizer promotes plain → HNSW segments.  
- **Payload** indexed for filtering (keyword, integer, geo, etc.).  
- Optional **quantization** (scalar/product) with `always_ram` tuning.  
- **On-disk payload** configurable per collection.

---

## Distributed storage & clustering

### Shard placement

- **`shard_number`:** fixed at collection create (hard to change later).  
- **`replication_factor`:** copies across nodes for HA/read scaling.  
- **`write_consistency_factor`:** quorum for writes.  
- Cluster: each shard has a **peer id**; Qdrant routes reads/writes to responsible node.

### Custom shard keys (multitenancy)

From [multitenancy article](https://qdrant.tech/articles/multitenancy/):

- **`shard_key`** on upsert routes point to **specific shard** (tenant, region, etc.).  
- Search with **shard key filter** → **only relevant shards** queried (subset routing at **tenant** granularity, not embedding centroid).  
- Enables **1M+ tenants** with isolated shards vs over-sharding one collection.

### Consistency

- Distributed updates use consensus on cluster state.  
- Segment-level optimizers run per shard independently.

---

## Indexing

- **HNSW** primary ANN (configurable `m`, `ef_construct`, `ef` search).  
- **Sparse vectors** supported (separate index path).  
- Index built when segment size > **`indexing_threshold`** (KB).  
- **Full scan** fallback for tiny collections below threshold.  
- Runtime parameter updates (HNSW, quantization) without full rebuild in recent versions.

---

## Query processing

### Local shard

1. Parse filter on payload indexes.  
2. If HNSW segment exists → ANN with `ef`; else brute-force within segment.  
3. Apply score threshold / limit.

### Distributed query

1. Coordinator determines **target shards** (all, or subset via shard key / custom routing).  
2. Parallel search on peers.  
3. **Merge top-k** at coordinator.

**Without shard key:** behaves as **fan-out to all shards** (same structural pattern as FLANN distributed / Milvus).  
**With shard key:** tenant-scoped queries avoid cluster-wide fan-out.

---

## Distributed ANN pattern

| Dimension | Qdrant |
|-----------|--------|
| Layer | **#1** — HNSW per shard/segment |
| Default placement | Hash / round-robin shard assignment |
| Subset routing | **Custom shard key** (application-defined, not k-means on embeddings) |
| Spatial colocation | **No** default |

---

## Relation to Ember

- Compare **shard key** (scalar/tenant) vs **spatial partition routing** (ADBV, GP-ANN).  
- Open-source transparency on segment/HNSW lifecycle useful for **independent index per shard** cost model.

---

## Bibliography

- Qdrant Documentation, *Collections* and *Distributed Deployment*.  
- Qdrant, *Multitenancy and Custom Sharding* (product article).  
- Qdrant open-source: https://github.com/qdrant/qdrant
