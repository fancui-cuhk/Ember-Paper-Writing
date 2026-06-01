# LindormVector: A Distributed Vector Engine on a Cloud-Native Multi-Model NoSQL Database

## Bibliography

| Field | Value |
|-------|--------|
| **Title** | LindormVector: A Distributed Vector Engine on a Cloud-Native Multi-Model NoSQL Database |
| **Authors** | Yan Wang, Jian Zhou, Sai Huang, Chao Dou, Hanwen Tian, Zhijie Jiang, Zongning Zhang, Xiaoqi Li, Zhencan Peng, Chunhui Shen, Wei Zhang, Feifei Li, Dong Deng (Alibaba Cloud / affiliations per SIGMOD listing) |
| **Venue** | ACM SIGMOD 2026 — Industry Track (accepted) |
| **DOI** | [10.1145/3788853](https://dl.acm.org/doi/10.1145/3788853) |
| **Read date** | 2026-05-28 |

> **Update (2026-06-01)**: Full PDF has been parsed. Key correction: paper's experiments use **ESSD PL1 block storage** (350MB/s, 50K IOPS), not S3/object storage. Cold query / tail latency under cache miss is **not evaluated** — quantized vectors are assumed memory-resident.

---

## Paper Summary（论文在做什么）

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

## 作者阅读笔记（你的思考）

> 以下保留你的原始判断，并略作结构化，便于日后改写 Related Work。

1. **架构相似**：与我们一样采用 compute–storage disaggregation；上层 LDServer 存 IVFPQ，centroid 上建 HNSW 加速，这部分**常驻 LDServer**；压缩后的 posting list **主要在 LindormDFS**，LDServer 内存作 cache。  
2. **同样存在冷启动与 tail 问题**：posting list 不在 LDServer 上则要拉 DFS → 高延迟；除非 LindormDFS 背后是**很便宜的高速介质**（SSD + RDMA），否则与 S3 类系统同族。  
3. **IVF 路线一致，问题不同**：IVF 选型与我们相同，但**没有解决我们要解决的问题**（cold query / tail under disaggregation），而是把精力放在多模一体化、混合检索优化上。  
4. **写作策略**：Related Work 中应作为 **“同架构、不同优化目标”** 的工业界代表，避免审稿人认为“阿里云已经做了 cloud-native vector”；同时诚实写出 **存储介质假设** 可能削弱其冷路径问题的程度。

### 已验证（2026-06-01 读全文后更新）

- [x] 论文是否报告 **cache miss** 下的延迟分位数？ → **❌ 否**，假设 quantized vectors always in memory  
- [x] LindormDFS 在论文实验中的 **具体介质** → **ESSD PL1** (50K IOPS, 350MB/s)，**非 S3**  
- [x] Posting list 的 **粒度** → per-cluster posting lists，但 **assumed memory-resident**  
- [x] 与 **scale-to-zero / 多租户冷 tenant** 相关的讨论？ → **❌ 无**，multi-tenant 优化针对 memory footprint  
- [x] VectorDBBench 成绩条件 → **warm cache + ESSD**，未测试 cold start

---

## Use in Ember paper

| Section | Suggested use |
|---------|----------------|
| **Related Work — Industrial cloud-native DBs** | LindormVector as peer: disaggregated IVFPQ, Alibaba production scale |
| **Introduction — Not alone on IVF** | Corroborates IVF+PQ as industry default at billion scale |
| **Motivation — Problem still open** | Even sophisticated industrial systems rely on **cache + DFS** without addressing S3-class cold tail |
| **Do not over-claim** | If Lindorm uses fast DFS tier, narrow comparison to **our target storage model** |

**Cross-references**: `summaries/cloud-hosted-vs-cloud-native.md` (root causes), `discussions/2026-05-28-introduction-framework.md` (motivation table).
