# Distributed ANNS — Taxonomy (Survey Draft v6)

**Updated:** 2026-06-23 (fact-check corrections applied)  
**Principle:** Distilled vocabulary; **main index type** only; decision tree lists **system names only**; no empty leaves.

---

## 0. Vocabulary

### Organization axis (level 1)

| Term | Meaning |
|------|---------|
| **Placement-first** | **Assign vectors to nodes first** → build a **complete local index** on each node-local split. Each split is an independent ANN index. |
| **Index-first** | **Build one logical index first** → **place index pieces** on nodes. Search traverses one shared index namespace. |

**Two levels of “placement” (disambiguation):**

| Level | Question |
|-------|----------|
| **Organization** | Does node assignment **define** separate local indexes (**placement-first**), or **follow** from one index (**index-first**)? |
| **Node placement strategy** | Given that organization, **how** are splits chosen? (random / spatial colocation / query-history-aware) |

Both branches use node placement strategy; only **placement-first** makes placement **define** the index boundary.

### Node placement strategy (level 2)

| Strategy | Meaning |
|----------|---------|
| **Random placement** | Splits ignore embedding geometry (hash, block, striping, row-key, tenant key, **hash shuffle of spatial index pieces**) |
| **Spatial colocation placement** | Splits use embedding geometry (k-means, graph cut, LSH bucket, tree node) **at the node-assignment layer** |
| **Query-history-aware placement** | Split boundaries or counts refined from **past query statistics** |

**Index types (two only):**

| Type | Includes | “Disguise” rule |
|------|----------|-----------------|
| **Graph** | HNSW, Vamana, DiskANN, PAG, global navigable graph | Disk-resident graph = **Graph** |
| **IVF / LSH** | IVF, inverted file, SPANN tree, LSH buckets, PQ postings | Graph-on-centroids for **cluster selection** → still **IVF / LSH** |

### Sidenotes (not tree branches)

| System / mechanism | Note |
|--------------------|------|
| **Harmony — dimension split** | Can shard by **coordinate subspace** when vector shards skew (partial Euclidean distance + early stop). |
| **Harmony — runtime steering** | Also picks vector vs dimension mode **per query** via cost model (complements offline `B_vec` refinement from past queries). |
| **Harmony — index type** | Distributed **partial-distance scan**, not classic inverted-file or LSH; grouped under IVF/LSH as closest bucket. |
| **CXL-ANNS** | Index-first **global graph**; nodes hold **dimension columns** across CXL expanders — **dimension split**, not spatial colocation. |
| **SPIRE — index vs node** | Hierarchical k-means is **index geometry**; partitions are **hash-shuffled** to storage nodes (anti hot-spot). |
| **HAKES — refine tier** | FilterWorkers use k-means IVF (spatial); RefineWorkers shard by **vector ID** (random). |
| **Pyramid — optional weights** | Sample-query vertex weights on meta-HNSW; weaker than SPANN §4.3 — stays spatial leaf only. |

---

## 1. Placement-first — typical split sizes (real numbers)

| System | Split unit | Documented size / scale |
|--------|------------|-------------------------|
| **Milvus** | Sealed **segment** | **~512 MB** raw field data per segment (default seal target) |
| **Pinecone** | **Slab** (within namespace) | **No fixed MB** — compaction-driven |
| **Weaviate** | **Shard** | **No fixed cap** — one HNSW graph per shard |
| **Qdrant** | **Shard** → **segment** | Shard count fixed at create; segments split/merge dynamically |
| **OpenSearch k-NN** | **Search shard** → Lucene **segment** | Operator-chosen shard size; ANN per Lucene segment |
| **FLANN (MPI)** | Equal **subset** | **⌈N / P⌉ vectors** per worker |
| **Block serverless** | **Fixed byte block** | Equal byte ranges; **P** operator-chosen |
| **Cosmos DB DiskANN** | **Physical partition** | **~10 M vectors** / partition (paper example) |
| **Vexless** | **Function shard** | **~1.5 GB RAM** / Azure Function |
| **Auncel** | **Uniform worker shard** | Equal split across map-reduce workers |
| **ADBV / GP-ANN / GaussDB** | **Centroid shard** | Operator-chosen count (ADBV eval: **512** partitions) |

---

## 2. Decision tree (system names only)

```
DISTRIBUTED ANNS
│
├─ PLACEMENT-FIRST
│   │
│   ├─ Random placement
│   │     Milvus, Weaviate, Qdrant, OpenSearch k-NN, FLANN,
│   │     Block-Serverless, Auncel, Vearch, Cosmos DB
│   │
│   └─ Spatial colocation placement
│         ADBV, GaussDB, GP-ANN, Vexless, VStream, Pyramid, Pinecone
│
└─ INDEX-FIRST
    │
    ├─ Graph
    │   ├─ Random placement
    │   │     DistributedANN
    │   │
    │   └─ Spatial colocation placement
    │         CoTra, RED-ANNS, SHINE, d-HNSW, BatANN, DSANN
    │
    └─ IVF / LSH
        ├─ Random placement
        │     SPIRE
        │
        ├─ Spatial colocation placement
        │     SABES, LindormVector, C-SPANN, HAKES, Haghani, Turbopuffer
        │
        └─ Query-history-aware placement
              SPANN, SABBSR, Harmony
```

---

## 3. Completeness audit

### 3.1 In the tree — corpus assets (32)

| Asset | Leaf |
|-------|------|
| industrial/milvus.md | Placement-first → Random |
| industrial/weaviate.md | Placement-first → Random |
| industrial/qdrant.md | Placement-first → Random |
| industrial/opensearch-knn.md | Placement-first → Random |
| industrial/pinecone.md | Placement-first → Spatial colocation |
| industrial/cockroachdb-c-spann.md | Index-first → IVF/LSH → Spatial colocation |
| flann-pami2014.pdf | Placement-first → Random |
| building-stateless-serverless-vector-dbs-via-block.pdf | Placement-first → Random |
| auncel.pdf | Placement-first → Random |
| vearch-gamma.pdf | Placement-first → Random |
| cost-effective-low-latency-vector-search-with-azur.pdf | Placement-first → Random |
| analyticdb-v.pdf | Placement-first → Spatial colocation |
| gaussdb-vector.pdf | Placement-first → Spatial colocation |
| gp-ann.pdf | Placement-first → Spatial colocation |
| vexless.pdf | Placement-first → Spatial colocation |
| vstream.pdf | Placement-first → Spatial colocation |
| pyramid-bigdata2019.pdf | Placement-first → Spatial colocation |
| spann.pdf | Index-first → IVF/LSH → Query-history-aware |
| distributedann.pdf | Index-first → Graph → Random |
| cotra.pdf | Index-first → Graph → Spatial colocation |
| red-anns.pdf | Index-first → Graph → Spatial colocation |
| shine.pdf | Index-first → Graph → Spatial colocation |
| d-hnsw.pdf | Index-first → Graph → Spatial colocation |
| batann.pdf | Index-first → Graph → Spatial colocation |
| dsann.pdf | Index-first → Graph → Spatial colocation |
| sabes.pdf | Index-first → IVF/LSH → Spatial colocation |
| sabbsr.pdf | Index-first → IVF/LSH → Query-history-aware |
| haghani-distributed-lsh-edbt-2009.pdf | Index-first → IVF/LSH → Spatial colocation |
| lindorm-vector.pdf | Index-first → IVF/LSH → Spatial colocation |
| hakes.pdf | Index-first → IVF/LSH → Spatial colocation |
| spire.pdf | Index-first → IVF/LSH → **Random** |
| harmony.pdf | Index-first → IVF/LSH → Query-history-aware |

### 3.2 In the tree — products not in `distributed-anns-related` (1)

| System | Leaf | Source |
|--------|------|--------|
| **Turbopuffer** | Index-first → IVF/LSH → Spatial colocation | SPFresh-style cluster objects on object storage (`cold-hot-query-paths.md`, partition survey product table) |

### 3.3 Sidenote only — in corpus, no tree leaf (1)

| Asset | Why sidenote |
|-------|------------|
| cxl-anns.pdf | Index-first global graph + **dimension-column** placement across CXL — no random/spatial/query-history leaf fits |

### 3.4 Out of tree — single-node algorithm (17)

ada-ivf, crackivf, quake, rairs, vhp, scann, diskann, freshdiskann, spfresh, product-quantization, leqat, intelligent-probing, i-lsh, sk-lsh, i-o-efficient, curator, pase — single-node or algorithm baseline; **DiskANN** → see **DistributedANN** for distributed extension; **SPFresh** → lineage for C-SPANN / Turbopuffer.

### 3.5 Out of tree — infrastructure overlay (1)

netanns.pdf — in-network accelerator; assumes existing partitioned backend.

### 3.6 Candidates to add later (not in tree)

MemANNS, BLISS (query-history-aware IVF research); Elasticsearch, pgvector+Citus, LanceDB (products, same models as in-tree cousins).

---

## 4. Main-index reclassification (no “hybrid”)

| System | Main type | Node placement strategy |
|--------|-----------|-------------------------|
| SPANN | **IVF / LSH** | Query-history-aware (§4.3 bin-pack on query-access history) |
| SPIRE | **IVF / LSH** | **Random** (hash shuffle of k-means partitions to nodes) |
| SABES, LindormVector, C-SPANN, HAKES, Haghani, Turbopuffer | **IVF / LSH** | Spatial colocation |
| SABBSR, Harmony | **IVF / LSH** | Query-history-aware |
| DistributedANN | **Graph** | Random |
| CoTra, RED-ANNS, SHINE, d-HNSW, BatANN, DSANN | **Graph** | Spatial colocation |
| GP-ANN, Pyramid | **Graph** | Spatial colocation (placement-first) |
| CXL-ANNS | **Graph** | Dimension split (sidenote) |

---

## Related

- [`partition-sharding-vector-search-survey.md`](partition-sharding-vector-search-survey.md)  
- [`discussions/2026-06-23-distributed-anns-taxonomy-fact-check.md`](../discussions/2026-06-23-distributed-anns-taxonomy-fact-check.md)
