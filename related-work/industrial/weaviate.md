# Weaviate — Production Architecture Analysis

**Type:** Industrial system (open-source docs; no ANN systems paper)  
**Last updated:** 2026-06-16  
**Survey category:** `distributed-anns-related`

---

## Sources (primary)

| Source | URL |
|--------|-----|
| Storage concepts | https://docs.weaviate.io/weaviate/concepts/storage |
| Horizontal scaling / cluster | https://docs.weaviate.io/weaviate/concepts/cluster |

Repo: [`summaries/partition-sharding-vector-search-survey.md`](../../summaries/partition-sharding-vector-search-survey.md) §1.

---

## Executive summary

Weaviate is an **open-source** vector database with **collection → shard** layout. Each shard is a **self-contained unit**: LSM object store + inverted index + **one HNSW graph** (not segmented). Cluster scaling uses **UUID hash sharding** (Murmur-3 64-bit) and optional **replication** for QPS/HA. **Sharding increases capacity and import speed; it does not improve query throughput per shard.** Filters run **before** vector search (pre-filtering).

---

## Logical storage units

```
Class (schema) → Index → Shard(s) → { object LSM, inverted LSM, HNSW index }
```

| Store | Algorithm | Notes |
|-------|-----------|-------|
| Object + inverted | **LSM-Tree** (v1.5+) | Memtable → sorted segments; Bloom filters; background merge |
| Vector index | **HNSW** (custom) | **Not segmented** — one large graph per shard; WAL + optional snapshots |

**Design rationale:** LSM segmentation for structured/filter data; **single large HNSW per shard** because HNSW graphs don't merge efficiently across segments.

---

## Distributed storage

### Sharding

- **Shard key:** object **UUID** (v1.8+).  
- **Algorithm:** 64-bit **Murmur-3** hash → shard assignment.  
- **Multi-tenant collections:** one shard **per tenant** (shard count = tenant count).  
- Shards placed on nodes with **most free disk** at class creation.

### Replication

- Configurable **replication factor** (global default or per-collection).  
- Replicas = same data on multiple nodes → **near-linear read QPS** scaling.  
- **Import speed does not scale** with replication.

### Persistence

- Every write → **WAL** before ACK.  
- HNSW: WAL stores **placement decisions**; full graph recoverable by replay.  
- **HNSW snapshots** (v1.31+, default v1.36): point-in-time graph load + delta WAL replay for fast restart.

### Lazy shard loading (v1.36.6+)

- Multi-tenant collections above **1000 shards** or **100 GB** → lazy load at startup.  
- Reduces restart time; first query on cold shard triggers prioritized load.

---

## Indexing

- **HNSW** default; pluggable vector index abstraction (HNSW implementation in production).  
- **HFresh** disk index option reduces pure-RAM pressure for large datasets.  
- Inverted index co-located in shard for **pre-filter** (BM25, boolean, geo, etc.).  
- Vector cache prefill: synchronous by default for eager-loaded collections; async when lazy loading active.

---

## Query processing

### Single-shard query

1. Resolve shard(s) from UUID hash or tenant routing.  
2. **Filter** on inverted/LSM indexes → candidate set.  
3. **HNSW search** on vector index within shard.  
4. Return full objects (not just IDs).

### Multi-shard / cluster query

- Coordinator fans out to **all shards** of the collection (unless tenant/shard pin narrows scope).  
- Merge top-k across shard results.  
- **No embedding-space subset routing** — same fan-out-all family as hash-sharded OLTP.

### Scaling trade-offs (official)

| Increase | Effect |
|----------|--------|
| **Shards** | Larger datasets, faster **imports**; **query QPS unchanged** |
| **Replicas** | Higher **query throughput**, HA; import unchanged |

---

## Cluster operations

- **Raft** consensus for schema changes (v1.25+).  
- **Shard replica movement** (v1.32+): rebalance load across nodes.  
- **Limitation (documented):** dynamic scale-in with data still evolving; shard migration on node removal not fully automatic in older docs.

---

## Distributed ANN pattern

| Dimension | Weaviate |
|-----------|----------|
| Layer | **#1** — HNSW per shard |
| Placement | **Hash(UUID)** — not spatial |
| Graph | **Intact per shard** (not graph-cut distributed) |
| Filtering | **Pre-filter** then ANN (strength vs post-filter) |

---

## Relation to Ember

- Baseline **hash shard + HNSW-per-shard + merge** alongside Milvus/Qdrant.  
- LSM + non-segmented HNSW illustrates **why** disk-ANN papers target single-node graphs — Weaviate keeps one graph per shard for query efficiency.

---

## Bibliography

- Weaviate Documentation, *Storage* and *Horizontal Scaling* (2025–2026).  
- Weaviate open-source: https://github.com/weaviate/weaviate
