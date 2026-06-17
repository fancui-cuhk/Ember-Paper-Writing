# LindormVector: A Distributed Vector Engine on a Cloud-Native Multi-Model NoSQL Database

## Bibliography

| Field | Value |
|-------|--------|
| **Title** | LindormVector: A Distributed Vector Engine on a Cloud-Native Multi-Model NoSQL Database |
| **Authors** | Yan Wang, Jian Zhou, Sai Huang, Chao Dou, Hanwen Tian, Zhijie Jiang, Zongning Zhang, Xiaoqi Li, Zhencan Peng, Chunhui Shen, Wei Zhang, Feifei Li, Dong Deng (Alibaba Cloud / affiliations per SIGMOD listing) |
| **Venue** | ACM SIGMOD 2026 — Industry Track (accepted) |
| **DOI** | [10.1145/3788853](https://dl.acm.org/doi/10.1145/3788853) |
| **PDF in repo** | [`pdfs/read-amp-related/lindormvector-sigmod2026.pdf`](pdfs/read-amp-related/lindormvector-sigmod2026.pdf) · also [`pdfs/spatial-related/lindormvector-sigmod2026.pdf`](pdfs/spatial-related/lindormvector-sigmod2026.pdf) |
| **Read date** | 2026-05-28 |

> **Update (2026-06-01)**: Full PDF has been parsed. Key correction: paper's experiments use **ESSD PL1 block storage** (350MB/s, 50K IOPS), not S3/object storage. Cold query / tail latency under cache miss is **not evaluated** — quantized vectors are assumed memory-resident.

---

## Paper Summary

### Stated problem & positioning

LindormVector is the **vector retrieval engine** inside **Lindorm**, Alibaba Cloud’s cloud-native **multi-model NoSQL** database (wide-table, time-series, search, vector, columnar, etc.). The industry paper’s emphasis (from acceptance title and public materials) is not “serverless vector DB at minimum cost,” but:

- **Tight integration** of vector search with scalar and full-text data in one system  
- **Hybrid retrieval** (filters + vectors + text) with a **CBO/RBO optimizer** to pick execution paths (vector-driven, scalar-driven, parallel pipeline)  
- **Production-scale** deployment narrative (VectorDBBench, RAG, recommendation, high QPS)

In other words, the contribution vector is closer to **“vector engine as a first-class citizen inside a multi-model cloud database”** than to **“solve cold-query tail latency under object-storage disaggregation.”**

### Architecture (compute–storage disaggregation)

```
┌─────────────────────────────────────────────────────────────┐
│  Compute: LDServer (per-region / sharded serving nodes)      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Always resident on LDServer:                         │    │
│  │  • IVFPQ index metadata                              │    │
│  │  • Centroid structure + HNSW on centroids (coarse)   │    │
│  └─────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Cached in LDServer memory (when hot):              │    │
│  │  • Compressed posting lists (PQ codes, etc.)       │    │
│  └─────────────────────────────────────────────────────┘    │
│         ▲ miss / cold ── fetch from LindormDFS              │
├─────────┼───────────────────────────────────────────────────┤
│  Storage: LindormDFS (unified distributed storage layer)    │
│  • Primary home of compressed posting lists                 │
│  • Shared-storage, multi-engine cloud-native DFS design     │
└─────────────────────────────────────────────────────────────┘
```

**Query path (IVFPQ, aligned with reader notes + product docs):**

1. **Coarse search**: HNSW (optional, `centroids_use_hnsw`) over **centroids** → select `nprobe` clusters — centroid graph stays on LDServer.  
2. **Fine search**: Load **compressed posting lists** for probed clusters from LindormDFS (or LDServer cache), PQ distance + optional **reorder** with original vectors (`reorder_factor`).  
3. **Hybrid queries**: Optimizer routes combined scalar / full-text / vector plans (paper’s differentiated story vs. “splice two indexes”).

### Index & algorithm choices

| Component | Choice | Role |
|-----------|--------|------|
| Main index | **IVFPQ** | Disk-oriented, compressed posting lists; offline build (`knn.offline.construction=true`) |
| Coarse level | **HNSW on centroids** | Faster centroid probe vs. brute force over `nlist` |
| Quantization | PQ on residuals | ~1:8 compression cited in docs (memory footprint) |
| Alternatives | HNSW, IVFBQ, FLAT | Product supports multiple algorithms; paper/industry focus on integrated stack |

This **IVF + PQ + HNSW-on-centroids** stack is the same **algorithm family** Ember’s narrative already assumes for cloud-native ANN at scale—not a fundamental divergence.

### What the paper optimizes for (inferred)

- **Cost–performance at scale** via disk/DFS-backed IVFPQ and shared storage  
- **Hybrid query quality** (recall under filters, optimizer routing)  
- **Operational integration** inside Lindorm’s multi-model fabric  

What is **not** foregrounded in available materials:

- Explicit **cold-start SLA** or **p99 tail latency** under cache miss  
- **Request amplification** vs. **read amplification** tradeoff on remote storage  
- **Serverless / scale-to-zero** economics (Lindorm is elastic cloud DB, but billing model differs from Pinecone-style per-query object storage)

---

## Cold start & tail latency (reader analysis + extensions)

### Shared vulnerability with Ember’s problem framing

Under the same **compute–storage disaggregation** pattern as Ember/Milvus/Pinecone-class systems:

| Data | Typical location | Cold behavior |
|------|------------------|---------------|
| Centroid + HNSW metadata | LDServer (resident) | Low extra I/O on cold query |
| **Compressed posting lists** | LindormDFS primary; LDServer RAM cache | **Must be fetched on cache miss** |

If posting lists are **not** in LDServer memory, each probed cluster incurs **DFS/network I/O** before IVF probe can finish. That is structurally the same class of problem as:

- Milvus loading segments from object storage  
- Turbopuffer fetching per-cluster S3 objects  

So LindormVector **also faces cold-query latency and tail behavior**, even if the paper does not center that story.

### When the problem “disappears” (important caveat)

Reader note (correct): if LindormDFS is backed by **low-latency media** (e.g., **local NVMe SSD**, or **remote SSD + RDMA**) with ample bandwidth, cold posting-list fetch may drop to **milliseconds**, and tail may look “good enough” in benchmarks.

Implications for Ember’s Related Work:

- Lindorm is **not** necessarily “object-storage-limited” in the same way as S3-first systems; LindormDFS can span performance tiers (Alibaba docs: performance / standard / capacity tiers, hot–cold mixing).  
- **Apples-to-apples comparisons** must state **storage media + network**; otherwise we risk overstating Lindorm’s cold-query pain or understating ours.  
- Ember’s thesis should clarify **target deployment assumption** (e.g., commodity object storage vs. accelerated DFS).

### What Lindorm does *not* appear to solve (vs. Ember goals)

From reader conclusion — **aligned with repo summaries**:

1. **No new answer to read amplification vs. request amplification** at billion scale on **high-latency, opaque object storage**.  
2. **Caching on LDServer** is the main mitigation — same class as “warm compute + cold DFS,” not a redesign of **what** is fetched **how many times** on cold path.  
3. **Paper focus** on multi-model hybrid optimization **does not substitute** for sub-second, predictable **cold p99** under serverless eviction / scale-to-zero.

---

## Comparison with Ember

| Dimension | LindormVector | Ember (intended positioning) |
|-----------|---------------|------------------------------|
| **Architecture paradigm** | Compute–storage separation; LDServer + LindormDFS | Compute–storage separation; stateless compute + **object storage** |
| **Index family** | IVFPQ + HNSW centroids | IVF-class (same broad choice) |
| **Primary contribution** | Multi-model unified engine; hybrid query optimizer | **Cold-query / tail latency** under **object storage** disaggregation |
| **Cold path** | ❌ **Not evaluated** (assumes memory-resident posting lists) | **Explicit problem** + root-cause analysis + targeted design |
| **Storage media** | ESSD / distributed filesystem | **S3 / commodity object storage** |
| **Tail latency focus** | ❌ None | ✅ Core contribution |
| **Related Work role** | **Different storage tier, same index family** — validates IVF direction | Fills the **cold/tail gap** under object storage |

**One-line positioning for the paper:**

> LindormVector demonstrates that industrial cloud-native databases converge on **disaggregated IVFPQ + in-memory centroid structures**, but optimize for **hybrid multi-model queries**, not for **predictable tail latency when posting lists miss compute-side cache on slow storage**.

---

## Author reading notes

> Original judgments preserved below, lightly structured for Related Work drafting.

1. **Similar architecture:** Same compute–storage disaggregation as us; LDServer holds IVFPQ + HNSW on centroids (**always resident**); compressed posting lists live primarily on **LindormDFS**, with LDServer memory as cache.
2. **Same cold-start / tail vulnerability:** If posting lists are not on LDServer, fetch from DFS → high latency—unless LindormDFS is backed by **cheap fast media** (SSD + RDMA), in which case it is less like S3-class systems.
3. **Same IVF family, different problem:** IVF choice aligns with us, but the paper does **not** solve our target problem (cold query / tail under disaggregation); focus is multi-model integration and hybrid retrieval optimization.
4. **Writing strategy:** Cite as **“same architecture, different optimization target”** among industrial systems—avoid implying Alibaba already solved cloud-native vector cold paths; state **storage media assumptions** honestly.

### Verified (2026-06-01, full paper read)

- [x] Does the paper report latency percentiles under **cache miss**? → **No** — assumes quantized vectors always in memory
- [x] **Storage medium** in experiments → **ESSD PL1** (50K IOPS, 350 MB/s), **not S3**
- [x] Posting list **granularity** → per-cluster lists, but **assumed memory-resident**
- [x] **Scale-to-zero / cold multi-tenant** discussion? → **No** — multi-tenant work targets memory footprint
- [x] VectorDBBench conditions → **warm cache + ESSD**, no cold-start test

---

## Use in Ember paper

| Section | Suggested use |
|---------|----------------|
| **Related Work — Industrial cloud-native DBs** | LindormVector as peer: disaggregated IVFPQ, Alibaba production scale |
| **Introduction — Not alone on IVF** | Corroborates IVF+PQ as industry default at billion scale |
| **Motivation — Problem still open** | Even sophisticated industrial systems rely on **cache + DFS** without addressing S3-class cold tail |
| **Do not over-claim** | If Lindorm uses fast DFS tier, narrow comparison to **our target storage model** |

**Cross-references**: `summaries/cloud-hosted-vs-cloud-native.md` (root causes), `discussions/2026-05-28-introduction-framework.md` (motivation table).
