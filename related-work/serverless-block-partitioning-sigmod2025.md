# Building Stateless Serverless Vector DBs via Block-based Data Partitioning

## Bibliography

| Field | Value |
|-------|--------|
| **Title** | Building Stateless Serverless Vector DBs via Block-based Data Partitioning |
| **Authors** | Daniel Barcelona-Pons, Raúl Gracia-Tinedo, Albert Cañadilla-Domingo, Xavier Roca-Canals, Pedro García-López |
| **Venue** | Proc. ACM Manag. Data 3, 6 (SIGMOD), Article 304, December 2025 |
| **DOI** | [10.1145/3769769](https://doi.org/10.1145/3769769) |
| **PDF** | [Author copy](https://danielbcn.com/papers/2025-SIGMOD-Serverless_Vector_DBs_Partitioning.pdf) |
| **PDF in repo** | [`pdfs/building-stateless-serverless-vector-dbs-via-block.pdf`](pdfs/building-stateless-serverless-vector-dbs-via-block.pdf) |
| **Read date** | 2026-06-18 |

---

## Internal reading note — problem & setting

**TL;DR:** First systematic comparison of **clustering-based vs block-based** data partitioning for **stateless FaaS** vector DBs (AWS Lambda + S3). Argues that under **dynamic ingest**, semantic clustering (Vexless-style k-means) is too slow and hard to balance; **fixed-size blocks** win on partitioning cost and straggler avoidance, and can still match Milvus on recall/latency when queries fan out to all partitions.

**Not** a spatial-colocation paper — explicitly **anti-geometry** for partitioning. Listed in partition survey §1 (Category #1, block sharding) and §4.6 excluded table.

---

## Architecture (stateless serverless)

```
Ingest → [Partitioning] → S3 data partitions → [Indexing Lambdas] → S3 indexes
Query batch → Coordinator → Map (all partitions) → Reduce → top-k
```

- **Stateless FaaS:** Lambda cannot keep index in memory between invocations; each query batch re-reads from S3.
- **vs Vexless:** Vexless uses **Azure Durable Functions** (stateful), k-means + centroid filtering + boundary redundancy; static dataset assumption.
- **vs Milvus:** Serverful cluster with Data/Index/Query nodes, gRPC, persistent memory — lower interactive latency, poor elasticity on sparse/bursty workloads.

Prototype: **Lithops** map-reduce on AWS Lambda; **Faiss IVF** (k=512, nprobe=32) per partition.

---

## Two partitioning schemes (Section 4)

| Dimension | Clustering-based (Vexless-style) | Block-based (this paper) |
|-----------|----------------------------------|---------------------------|
| **Partitioning cost** | Full-dataset k-means on VM; O(dataset) view; balanced k-means even costlier | **Byte-range splits**; parallel; no global view — **3.5–5.8× faster** |
| **Load balance** | Unbalanced clusters → **straggler Lambdas**; balanced k-means helps but expensive on continuous ingest | **Equal-sized chunks** → uniform function work |
| **Query** | Filter partitions by **nearest centroids** → fewer functions, better recall per compute — but needs **redundancy** at boundaries | **Query all N partitions** → higher compute, simpler, no centroid routing |
| **Dynamic data** | Re-cluster on growth impractical | Blocks append/stream naturally |

**Key trade-off:** Clustering buys **subset routing** (spatial pruning); blocks buy **ingest/partition simplicity** and accept **all-shard probe** at query time — viable when batch amortizes cold starts and S3 read parallelism is cheap.

---

## Empirical highlights (Sections 6–7)

- Block partitioning: **56–63% lower** partitioning cost vs clustering; query time **±30%** (sometimes faster despite all-partition probe).
- vs Milvus: **9.2–65.6×** faster partitioning; similar recall; **66–99% cost savings** on **sparse** workloads; serverful wins on steady low-latency interactive load.
- Vector **redundancy** (clustering): replicates vectors near partition boundaries (threshold `r` on centroid distances) to recover recall when filtering partitions.

---

## Relation to Ember / partition survey

| | This paper | Vexless / SABES / CoTra |
|---|-----------|-------------------------|
| **Geometry in placement** | **No** — fixed blocks | Yes — k-means / centroids / graph |
| **Query routing** | All partitions | Subset by centroid distance |
| **Primary bottleneck** | FaaS statelessness + ingest churn | Multi-probe / graph hop locality |
| **Load balance target** | **Equal partition bytes** (avoid stragglers) | Bucket count (SABES) or vector count (CoTra) |

**Survey placement:** Category **#1** entry (serverless block sharding); **excluded from §4** (no embedding-space colocation).

---

## Open questions

- At what query QPS does all-partition probe lose to centroid-filtered serverless (even with batching)?
- Hybrid: cheap block ingest + periodic re-cluster for hot partitions?
- Comparison to hash sharding (Milvus default) under same Lambda prototype.
