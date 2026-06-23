# OpenSearch k-NN — Production Architecture Analysis

**Type:** Industrial system (Elastic/OpenSearch documentation; Lucene integration)  
**Last updated:** 2026-06-16  
**Survey category:** `distributed-anns-related`

---

## Sources (primary)

| Source | URL |
|--------|-----|
| Approximate k-NN search | https://docs.opensearch.org/latest/vector-search/vector-search-techniques/approximate-knn/ |
| k-NN methods & engines | https://docs.opensearch.org/latest/mappings/supported-field-types/knn-methods-engines/ |

Repo: [`summaries/partition-sharding-vector-search-survey.md`](../../summaries/partition-sharding-vector-search-survey.md) §1.

---

## Executive summary

OpenSearch k-NN extends **Elasticsearch-style sharding** with **native-library ANN indexes** (Faiss, NMSLIB, Lucene HNSW) built **per Lucene segment** per `knn_vector` field. Queries scatter to **shards**; each shard returns top results from **all its segments**; coordinating node merges **`size`** final hits. **Filters apply post-ANN** (cannot push filter into index build path). This is classic **search-engine distribution**, not a purpose-built vector DB routing layer.

---

## Storage & indexing model

### Index structure

```
OpenSearch index (settings: index.knn=true)
  └─ Primary shards (+ optional replicas)
       └─ Lucene segments
            └─ knn_vector field → native ANN index (per segment)
```

- On indexing, OpenSearch builds a **native library index** for each **`knn_vector` field × Lucene segment** pair.  
- Custom **codec** writes vector data for Faiss/NMSLIB/Lucene engines.  
- Indexes loaded into **native memory** at search time (cache / warmup API).  
- **Model-based training** (IVF-PQ etc.): train once → model in system index → segments initialized from model at flush.

### Engines

| Engine | Typical methods |
|--------|-----------------|
| **Faiss** | HNSW, IVF-PQ, … |
| **NMSLIB** | HNSW variants |
| **Lucene** | HNSW (integrated) |

Dimension limit up to **16,000** for supported engines.

---

## Distributed layout

- Standard OpenSearch **primary/replica shards** across data nodes.  
- **No vector-specific sharding key** — documents hashed to shards by routing (default `_id`).  
- **No embedding-space partition routing** at cluster level.  
- Scale-out = **more primary shards** (at index creation) + replicas for read QPS.

---

## Query processing

### Approximate k-NN query

```json
"query": { "knn": { "my_vector": { "vector": [...], "k": 10 } } }
```

1. **Coordinate** query to all relevant shards (routing + replication rules).  
2. Each shard runs ANN on **each segment's** native index.  
3. **Faiss/NMSLIB:** `k` = max docs returned **across all segments of shard**.  
4. **Lucene engine:** `k` per shard (documented difference).  
5. Each shard returns **`size`** candidates to coordinator.  
6. Coordinator merges **`size × num_shards`** candidates → final top **`size`**.

### Filtering

- **Critical limitation:** filters applied **after** ANN search on shard — not index-time partition pruning.  
- Reduces effective recall under strict filters (known OpenSearch/Elastic design).

### Training workflow (IVF etc.)

- Separate **training index** without `index.knn` → Train API → model stored → production index uses `model_id` on `knn_vector` field.

---

## Distributed ANN pattern

| Dimension | OpenSearch k-NN |
|-----------|-----------------|
| Layer | **#1** — segment-local native indexes inside Lucene shards |
| Routing | **Document hash sharding** only |
| Merge | Coordinator top-k over shard partials |
| ANN placement | **Coupled to Lucene segment merge policy** |

Same family as **Elasticsearch dense_vector** and **pgvector + Citus** (relational shard → local index).

---

## Relation to Ember

- Illustrates **ANN bolted onto existing search shard model** — no cross-shard graph, no spatial routing.  
- Post-filter semantics differ from Milvus/Weaviate pre-filter — important for hybrid query related work.

---

## Bibliography

- OpenSearch Project Documentation, *Approximate k-NN search* (2025).  
- OpenSearch Project Documentation, *k-NN methods and engines*.
