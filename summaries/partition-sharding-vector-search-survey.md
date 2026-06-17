# Partition & Sharding in Vector Search: Survey

## Overview

This note catalogs research papers and industry systems where **partitioning or sharding** is central to vector search at scale. Entries are grouped by **where partitioning sits relative to the index**, not by publication venue. Each entry includes: venue, PDF link, **verbatim paper abstract** (from the publisher or arXiv; product docs marked explicitly), our reading notes (problem + technique), hardware, partitioning strategy, and rationale.

### Three partitioning layers (do not conflate)

| Layer | What is partitioned | Why partition | Representative work |
|-------|---------------------|---------------|---------------------|
| **1. Data partition → independent index** | The **dataset** is split into shards/segments/slabs first; each shard gets its **own complete ANN index** (IVF, HNSW, DiskANN, …). | Scale beyond one node, write parallelism, tenant isolation, compaction units — **not** because the index algorithm requires it. | Milvus, Pinecone, Weaviate, Qdrant, VEARCH, Cosmos DB sharded DiskANN |
| **2. Single logical index, externally sharded** | There is **one index** (one IVF inverted file, one graph, …) whose **internal structure is split across machines** for efficiency. | The index is too large for one node; at extreme scale, **N independent indexes** have worse search-cost scaling and (for graphs) recall than one logical index. | DistributedANN, HARMONY, RED-ANNS, BatANN, CoTra |
| **3. Partition-intrinsic index** | **Partitioning is part of the index algorithm** — IVF Voronoi cells, LSH buckets, k-means tree leaves. Required for search even on a single machine. | Algorithm design: prune search space by visiting only nearby partitions (nprobe buckets, tree branches). | IVF-PQ, SPANN/SPFresh (single-node IVF), LSH (SK-LSH, Multi-probe), ScaNN, CrackIVF, Quake |

**Two families of "partition":**

| Family | Categories | OLTP analogy | Purpose |
|--------|------------|--------------|---------|
| **Systems partition** (load distribution) | **#1 + #2** | Hash/range **sharding** in distributed OLTP | Spread **storage and compute** across nodes. Not required by the ANN algorithm. |
| **Index partition** (search pruning) | **#3** | Inverted-list / B-tree **ranges inside an index** | **Algorithm requirement** (IVF nprobe, LSH buckets). Needed even on one machine. |

**#1 vs #2** (both systems partition): **#1** builds **N independent indexes** (Milvus segment HNSW; scatter-gather). **#2** keeps **one logical index** split across nodes (DistributedANN global graph on KV). Same motivation (scale load); different search-cost and recall trade-offs — see DistributedANN entry for when #2 wins at extreme scale.

**#1/#2 vs #3:** Removing cluster sharding in #1/#2 still leaves valid indexes (possibly on one node). Removing coarse partitions in #3 breaks search.

### Are #1/#2 papers all multi-node?

**No.** Evaluations are usually multi-node, but the **same abstractions** appear at single-node scale:

| Single-node use | Why it makes sense | Example |
|-----------------|-------------------|---------|
| **Segment / slab / sealed unit** | Compaction, parallel build, memory budget, selective load from object storage | Milvus ~512 MB segments; Pinecone slabs |
| **Shard key / partition key** | Filter routing, tenant isolation before scatter-gather | Cosmos `vectorIndexShardKey`; Milvus partition key |
| **Scale-out-ready layout** | Same unit becomes a cluster shard later | Weaviate/Milvus architecture |
| **Build-time partition only** | Parallel index construction, not query routing | DiskANN k-means for parallel graph build |

**Local PDFs:** Downloaded copies live in [`related-work/pdfs/`](../related-work/pdfs/) (see [`manifest.tsv`](../related-work/pdfs/manifest.tsv)). **§4 papers** are under [`related-work/pdfs/sec4/`](../related-work/pdfs/sec4/). Entries mark **Local PDF** path or `NOT_DOWNLOADED` if fetch failed.

**Abstract policy:** Abstracts marked *(from docs)* or *(from wiki)* are from product documentation. Paper abstracts from arXiv are verbatim via API. Remaining PVLDB/SIGMOD/USENIX entries may still need manual publisher-page verification.

**Related:** `summaries/distributed_vector_search_related_work.md` · **Discussion:** `discussions/2026-06-16-partition-based-vector-search-survey.md`

---

## 1. Data Partition First → Independent Index per Partition

Production vector DBs most often follow this pattern: hash/range/semantic **data sharding**, then **build or seal** a local IVF or HNSW index inside each shard/segment/slab. Query = scatter to relevant shards + merge top-k.

### Milvus: A Purpose-Built Vector Data Management System

- **Category:** 1
- **Deployment scope:** Multi-node cluster; segments are also compaction units on one query node
- **Local PDF:** [`milvus.pdf`](../related-work/pdfs/milvus.pdf)
- **Venue:** SIGMOD 2021 (+ Manu / Milvus 2.0, PVLDB 2022)
- **PDF:** [Purdue mirror](https://www.cs.purdue.edu/homes/csjgwang/pubs/SIGMOD21_Milvus.pdf) · [Manu (PVLDB 2022)](https://www.vldb.org/pvldb/vol15/p3548-yan.pdf)
- **Abstract:** High-dimensional vector management in AI/data science needs systems beyond existing limits on scale, dynamism, and functionality. Milvus provides SDKs/REST APIs, CPU/GPU optimization, advanced query processing beyond pure similarity search, dynamic updates, and distributed deployment. Manu (Milvus 2.0) adds cloud-native decoupling via WAL/binlog for elasticity and tunable consistency.
- **Understanding**
  - **Problem:** Production vector workloads need distributed ingest, compaction, hybrid filtering, and multiple ANN backends—not a single in-memory index.
  - **Technique:** **Shard → partition → segment** hierarchy: hash sharding for write parallelism; optional partition keys for tenant/time filtering; each sealed segment holds an independent **IVF or HNSW** index (partition-then-build-index).
- **Hardware:** Distributed memory + object storage for sealed segments
- **Partitioning / Sharding:** System: **hash shard → partition (~16 buckets) → segment (~512 MB)**; Index: per-segment **k-means IVF** or **HNSW**
- **Rationale:** Shards scale writes; partitions prune filtered queries; segments are the unit of load/compact—HNSW cannot be merged across machines, so each segment gets its own graph

---

### Vexless: A Serverless Vector Data Management System Using Cloud Functions

- **Category:** 1
- **Deployment scope:** Multi-node (serverless functions)
- **Local PDF:** [`sec4/vexless.pdf`](../related-work/pdfs/sec4/vexless.pdf)
- **§4 centroid-locality:** covered in §4 deep dive
- **Venue:** SIGMOD 2024 (PACMMOD)
- **PDF:** [NSF PAR](https://par.nsf.gov/servlets/purl/10570270)
- **Abstract:** Cloud functions suit bursty/sparse vector workloads but raise sharding, communication, and cold-start challenges. Vexless is the first vector DB optimized for cloud functions, using a resource-aware orchestrator, stateful functions to cut sync overhead, and workload-aware lifetime management. On Azure Functions it cuts cost on bursty/sparse workloads vs. VMs while matching or beating query performance and accuracy.
- **Understanding**
  - **Problem:** Serverless functions have ~1.5 GB memory limits and cold starts; naive hash sharding creates stragglers and excessive cross-function traffic.
  - **Technique:** **Balanced/constrained k-means** shards data so each function owns a locality-aware subset; supports IVF/LSH/HNSW per shard with orchestration for stateful execution.
- **Hardware:** Distributed serverless (Azure Functions, ~1.5 GB/function)
- **Partitioning / Sharding:** **Balanced k-means** shards + local ANN index per shard
- **Rationale:** Semantic clustering beats hash under tight memory; balanced partitions avoid one slow function dominating tail latency

---

### Building Stateless Serverless Vector DBs via Block-based Data Partitioning

- **Category:** 1
- **Deployment scope:** Multi-node (AWS Lambda)
- **Local PDF:** [`building-stateless-serverless-vector-dbs-via-block.pdf`](../related-work/pdfs/building-stateless-serverless-vector-dbs-via-block.pdf)
- **Venue:** SIGMOD 2025 (PACMMOD)
- **PDF:** [Author copy](https://danielbcn.com/papers/2025-SIGMOD-Serverless_Vector_DBs_Partitioning.pdf)
- **Abstract:** Serverless vector DBs on stateless FaaS are promising but partitioning strategy is unclear. This study compares clustering-based vs. block-based partitioning for dynamic datasets on AWS Lambda: block partitioning is up to 5.8× faster to partition, up to 63% cheaper, with similar query latency. A block-based serverless design is competitive with Milvus on indexing, latency, recall, and cost, with large savings on sparse workloads.
- **Understanding**
  - **Problem:** k-means repartitioning is expensive and unstable under continuous ingest in FaaS; clustering quality matters less than ingest latency and cost.
  - **Technique:** **Fixed-size block partitioning** (no semantic clustering) with continuous ingest support; compared against k-means baseline on Lambda.
- **Hardware:** Serverless FaaS + object storage
- **Partitioning / Sharding:** **Block-based** fixed-size chunks (not k-means)
- **Rationale:** Blocks are O(1) to assign, evenly sized, and avoid expensive re-clustering on every update burst

---

### VStream: A Distributed Streaming Vector Search System

- **Category:** 1
- **Deployment scope:** Multi-node (distributed tiered storage)
- **Local PDF:** [`sec4/vstream.pdf`](../related-work/pdfs/sec4/vstream.pdf)
- **§4 centroid-locality:** covered in §4
- **Venue:** PVLDB 2025
- **PDF:** [PVLDB](https://www.vldb.org/pvldb/vol18/p1593-gao.pdf)
- **Abstract:** Batch-oriented vector DBs fit poorly with streaming workloads. VStream provides a dynamic partitioner for shifting vector streams, hierarchical four-level storage with streaming state management, and hot–cold separation using access patterns. It improves query efficiency 251–373×, cuts CPU 2.2–2.5×, and memory 1.5–2.0× vs. existing systems.
- **Understanding**
  - **Problem:** ID/hash sharding destroys spatial locality when vector streams drift; batch indexes cannot keep up with continuous ingest.
  - **Technique:** **LSH + space-filling curves (Z-order/Hilbert)** for partition assignment; dynamic rebalancing across a four-tier memory/local-disk/remote-disk hierarchy.
- **Hardware:** Distributed memory + local disk + remote disk
- **Partitioning / Sharding:** **LSH + space-filling curve** encoding with dynamic rebalancing
- **Rationale:** Curve encoding keeps spatially adjacent vectors in the same partition as streams evolve

---

### Auncel: Fast, Approximate Vector Queries on Very Large Unstructured Datasets

- **Category:** 1
- **Deployment scope:** Multi-node (map-reduce workers)
- **Local PDF:** [`auncel.pdf`](../related-work/pdfs/auncel.pdf)
- **Venue:** NSDI 2023
- **PDF:** [USENIX](https://www.usenix.org/system/files/nsdi23-zhang-zili.pdf)
- **Abstract:** Billion-item vector queries need bounded error/latency, but existing ANNS trades accuracy for latency without guarantees. Auncel builds per-query error–latency profiles from local geometry, samples just enough data per worker, and scales via map-reduce with calibrated local bounds. It is up to 10× faster than SOTA at ≤10% error; DEEP1B queries run in 25ms on four c5.metal instances.
- **Understanding**
  - **Problem:** Approximate search gives no SLO guarantees; distributed IVF amplifies error when merging partial top-k results.
  - **Technique:** Per-query **Error-Latency Profile (ELP)** on local IVF partitions; **map-reduce** with probabilistically calibrated local error bounds before global merge.
- **Hardware:** AWS EC2 multi-worker (map-reduce)
- **Partitioning / Sharding:** **Random uniform shard** → local IVF + local ELP per worker
- **Rationale:** Uniform shards simplify calibration; ELP controls how many local partitions each worker searches under a global bound

---

### NetANNS: A High-Performance Distributed Search Framework Based on In-Network Computing

- **Category:** 1
- **Deployment scope:** Multi-node (assumes partitioned backend)
- **Local PDF:** [`netanns.pdf`](../related-work/pdfs/netanns.pdf)
- **Venue:** IEEE ISPA/BDCloud/SocialCom 2021
- **PDF:** [Conference proceedings](https://www.cloud-conf.net/ispa2021/proc/pdfs/ISPA-BDCloud-SocialCom-SustainCom2021-3mkuIWCJVSdKJpBYM7KEKW/264600a271/264600a271.pdf)
- **Abstract:** Approximate nearest neighbor search (ANNS) frameworks incur preprocessing and data-movement overhead. NetANNS accelerates data preprocessing with programmable switches, integrates multiple ANNS algorithms, and follows MapReduce-style architecture. Experiments show ~2× search efficiency vs. common MapReduce-based distributed ANNS frameworks.
- **Understanding**
  - **Problem:** Distributed ANNS spends too much time on network data movement and preprocessing on CPUs.
  - **Technique:** **In-network computing** on programmable switches for LSH-inspired data/query classification; pluggable ANNS backends atop MapReduce-style **partitioned search**—does not invent a new partition algorithm.
- **Hardware:** Programmable switch + commodity servers
- **Partitioning / Sharding:** Assumes **existing distributed partitioned ANN**; accelerates routing/preprocessing in the network
- **Rationale:** Offloads classification to switches to cut CPU↔network round trips in partitioned search pipelines

---

### Unleashing Graph Partitioning for Large-Scale Nearest Neighbor Search

- **Category:** 1
- **Deployment scope:** Multi-node (shard-local HNSW)
- **Local PDF:** [`sec4/unleashing-graph-partitioning-for-large-scale-near.pdf`](../related-work/pdfs/sec4/unleashing-graph-partitioning-for-large-scale-near.pdf)
- **§4 centroid-locality:** covered in §4
- **Venue:** PVLDB 2025
- **PDF:** [PVLDB](https://www.vldb.org/pvldb/vol18/p1649-gottesbueren.pdf)
- **Abstract:** Large-scale ANNS must partition points into neighborhood-preserving shards and route queries to few shards. This paper designs modular routing (clustering + LSH) usable with any partitioner, enabling balanced graph partitioning without a native routing algorithm. On billion-scale data the pipeline reaches up to 2.14× QPS at 90% recall@10 vs. best competitor.
- **Understanding**
  - **Problem:** Graph indexes cannot run on one machine at billion scale; IVF routing is suboptimal for graph shards.
  - **Technique:** **Balanced graph partitioning** into shards + modular routing (**kRt** hierarchical k-means centers, **hRt** LSH); each shard holds a local **HNSW**.
- **Hardware:** Distributed memory (multi-shard)
- **Partitioning / Sharding:** **Graph partition** + k-means/LSH routing
- **Rationale:** Graph partition preserves neighbor locality; decoupled routing picks few shards per query

---

### VEARCH / Gamma: Real-Time Visual Search on JD E-commerce Platform

- **Category:** 1
- **Deployment scope:** Multi-node (PartitionServers)
- **Local PDF:** [`vearch-gamma.pdf`](../related-work/pdfs/vearch-gamma.pdf)
- **Venue:** Middleware 2018
- **PDF:** [arXiv](https://arxiv.org/pdf/1908.07389.pdf)
- **Abstract:** We present the design and implementation of a visual search system for real time image retrieval on JD.com, the world's third largest and China's largest e-commerce site. We demonstrate that our system can support real time visual search with hundreds of billions of product images at sub-second timescales and handle frequent image updates through distributed hierarchical architecture and efficient indexing methods. We hope that sharing our practice with our real production system will inspire the middleware community's interest and appreciation for building practical large scale systems for emerging applications, such as ecommerce visual search.
- **Understanding**
  - **Problem:** E-commerce visual search needs distributed scale, scalar+vector hybrid queries, and real-time updates.
  - **Technique:** **Master / Router / PartitionServer** architecture; each PS hosts a **document partition** with local **IVFPQ or HNSW** (Gamma engine on Faiss); raft replication.
- **Hardware:** Distributed cluster (PartitionServers + routers)
- **Partitioning / Sharding:** **Document/table partitions** on PartitionServers
- **Rationale:** Horizontal scale via partitions; each partition is an independent ANN index with scalar filtering


---

### Cost-Effective, Low Latency Vector Search with Azure Cosmos DB (Sharded DiskANN)

- **Category:** 1
- **Deployment scope:** Multi-node (Cosmos DB)
- **Local PDF:** [`cost-effective-low-latency-vector-search-with-azur.pdf`](../related-work/pdfs/cost-effective-low-latency-vector-search-with-azur.pdf)
- **Venue:** PVLDB 2025 / arXiv
- **PDF:** [arXiv](https://arxiv.org/pdf/2505.05885.pdf)
- **Abstract:** Vector indexing enables semantic search over diverse corpora and has become an important interface to databases for both users and AI agents. Efficient vector search requires deep optimizations in database systems. This has motivated a new class of specialized vector databases that optimize for vector search quality and cost. Instead, we argue that a scalable, high-performance, and cost-efficient vector search system can be built inside a cloud-native operational database like Azure Cosmos DB while leveraging the benefits of a distributed database such as high availability, durability, and scale. We do this by deeply integrating DiskANN, a state-of-the-art vector indexing library, inside Azure Cosmos DB NoSQL. This system uses a single vector index per partition stored in existing index trees, and kept in sync with underlying data. It supports < 20ms query latency over an index spanning 10 million vectors, has stable recall over updates, and offers approximately 43x and 12x lower query cost compared to Pinecone and Zilliz serverless enterprise products. It also scales out to billions of vectors via automatic partitioning. This convergent design presents a point in favor of integrating vector indices into operational databases in the context of recent debates on specialized vector databases, and offers a template for vector indexing in other databases.
- **Understanding**
  - **Problem:** One large DiskANN per partition is slow and expensive for multi-tenant filtered queries.
  - **Technique:** Default **one DiskANN per physical partition**; optional **`vectorIndexShardKey`** (e.g., tenantID) builds separate smaller DiskANN indexes per key value; filtered queries route to the matching shard index.
- **Hardware:** Cloud-native distributed DB (per replica/partition)
- **Partitioning / Sharding:** **Physical partition** + optional **vectorIndexShardKey** semantic sharding
- **Rationale:** Smaller scoped indexes improve recall, latency, and RU cost for tenant-filtered queries


---

## 2. Single Logical Index, Partitioned Across Nodes

These systems keep **one index abstraction** (one IVF posting namespace, one navigable graph, one k-means tree mapped to KV ranges) but **place parts on different machines**. Partitioning serves scale and I/O parallelism without treating each shard as a fully independent ANN index.

### HARMONY: A Scalable Distributed Vector Database for High-Throughput ANN Search

- **Category:** 2
- **Deployment scope:** Multi-node (4-node eval)
- **Local PDF:** [`sec4/harmony.pdf`](../related-work/pdfs/sec4/harmony.pdf)
- **§4 centroid-locality:** covered in §4
- **Venue:** SIGMOD 2025 (PACMMOD)
- **PDF:** [MIT DSpace](https://dspace.mit.edu/bitstream/handle/1721.1/164256/3749167.pdf)
- **Abstract:** Distributed ANNS suffers load imbalance and high communication from traditional partitioning. HARMONY combines vector-based and dimension-based multi-granularity partitioning to balance compute and cut communication, plus early-stop pruning exploiting monotonicity in dimension-based partitions. On real datasets it reaches 4.63× average throughput on four nodes and 58% gains on skewed workloads vs. leading distributed vector DBs.
- **Understanding**
  - **Problem:** Pure vector sharding causes hot shards and all-to-all candidate merging; pure dimension sharding adds many round trips.
  - **Technique:** **Hybrid partitioning**: k-means-style vector shards plus **dimension-based shards** where partial Euclidean distance accumulates additively; a cost model picks the mode per query; dimension-level early stopping prunes ~97% of candidates by the last machine.
- **Hardware:** Distributed memory cluster
- **Partitioning / Sharding:** **Hybrid vector + dimension** partitioning with dynamic mode selection
- **Rationale:** Vector partitions preserve locality; dimension partitions balance compute; early stop cuts cross-node traffic

---

### LindormVector: A Distributed Vector Engine on a Cloud-Native Multi-Model NoSQL Database

- **Category:** 2
- **Deployment scope:** Multi-node (Lindorm shards/ranges)
- **Local PDF:** NOT_DOWNLOADED (ACM 403)
- **Venue:** SIGMOD 2026 (Industry)
- **PDF:** [ACM](https://dl.acm.org/doi/pdf/10.1145/3788853.3803088)
- **Abstract:** LindormVector embeds distributed IVFPQ-based vector retrieval into Alibaba Lindorm multi-model NoSQL, with compute–storage separation, hybrid scalar/full-text/vector optimization, and production-scale VectorDBBench evaluation.
- **Understanding**
  - **Problem:** Cloud-native multi-model stores need disk-efficient vector search aligned with existing shard/range placement.
  - **Technique:** **k-means IVFPQ** with posting lists placed on Lindorm **shard/range** boundaries; compute-storage disaggregation for filter and refine stages.
- **Hardware:** Distributed memory + SSD/block storage (compute-storage separated)
- **Partitioning / Sharding:** **k-means IVFPQ** aligned with Lindorm **shard/range**
- **Rationale:** IVF+PQ compresses postings for disk; co-locating lists with KV shards avoids a separate routing layer

---

### AnalyticDB-V (ADBV): A Hybrid Analytical Engine Towards Query Fusion for Structured and Unstructured Data

- **Category:** 2
- **Deployment scope:** Multi-node (16-node eval); optional k-means data sharding
- **Local PDF:** [`sec4/analyticdb-v.pdf`](../related-work/pdfs/sec4/analyticdb-v.pdf)
- **§4 centroid-locality:** covered in §4 deep dive
- **Venue:** PVLDB 2020
- **PDF:** [PVLDB](http://www.vldb.org/pvldb/vol13/p3152-wei.pdf)
- **Abstract:** With the explosive growth of unstructured data (such as images, videos, and audios), unstructured data analytics is widespread in a rich vein of real-world applications. Many database systems start to incorporate unstructured data analysis to meet such demands. However, queries over unstructured and structured data are often treated as disjoint tasks in most systems, where hybrid queries (i.e., involving both data types) are not yet fully supported. In this paper, we present a hybrid analytic engine developed at Alibaba, named AnalyticDB-V (ADBV), to fulfill such emerging demands. ADBV offers an interface that enables users to express hybrid queries using SQL semantics by converting unstructured data to high dimensional vectors. ADBV adopts the lambda framework and leverages the merits of approximate nearest neighbor search (ANNS) techniques to support hybrid data analytics. Moreover, a novel ANNS algorithm is proposed to improve the accuracy on large-scale vectors representing massive unstructured data. ANNS is implemented as physical operators in ADBV, and a cost-based optimizer is designed to select the best execution plan with accuracy awareness. Extensive experiments on public and in-house datasets demonstrate the effectiveness of ADBV. ADBV has been deployed on Alibaba Cloud to serve real-world applications.
- **Understanding**
  - **Problem:** Hash/range sharding forces every vector query to fan out to all nodes; no similarity-based pruning at the routing layer.
  - **Technique:** **k-means clustering-based sharding** across nodes so the optimizer routes queries to the nearest N clusters only.
- **Hardware:** Distributed memory cluster (Alibaba Cloud)
- **Partitioning / Sharding:** **k-means cluster-based** cross-node sharding
- **Rationale:** Similarity routing reduces nodes touched per query vs. hash partitioning

---

### GaussDB-Vector: A Large-Scale Persistent Real-Time Vector Database for LLM Applications

- **Category:** 2
- **Deployment scope:** Multi-node (distance-based DN sharding)
- **Local PDF:** [`sec4/gaussdb-vector.pdf`](../related-work/pdfs/sec4/gaussdb-vector.pdf)
- **§4 centroid-locality:** covered in §4 deep dive
- **Venue:** PVLDB 2025
- **PDF:** [PVLDB](https://www.vldb.org/pvldb/vol18/p4951-sun.pdf)
- **Abstract:** Vector DBs address LLM hallucination/inference cost but often trade persistence, scalability, or hybrid filtering for raw speed. GaussDB-Vector is a persistent real-time vector DB with low-latency scalable search, real-time inserts/deletes, HA, distributed search, and hybrid scalar–vector queries via graph-index-oriented storage, buffering, PQ, parallel search, and SIMD/GPU/NPU acceleration. It outperforms baselines by 1–5×.
- **Understanding**
  - **Problem:** LLM serving needs persistent, filterable, distributed vector search—not just a fast in-memory HNSW.
  - **Technique:** Dual paths: **two-layer k-means IVF** with distance-based cross-node sharding for IVF; **PQ-segment subgraphs** plus distance sharding for graph path; page-based persistence and hybrid filter pushdown.
- **Hardware:** Distributed memory + disk (page-based persistence)
- **Partitioning / Sharding:** **Two-layer k-means IVF** + **distance-based k-means sharding**; graph path uses **PQ segment subgraphs**
- **Rationale:** IVF gives sequential I/O and easy maintenance; distance-based sharding enables query routing; graph subgraphs reduce build/I/O cost

---

### HAKES: Scalable Vector Database for Embedding Search Service

- **Category:** 2
- **Deployment scope:** Multi-node (IndexWorker + RefineWorker)
- **Local PDF:** [`hakes.pdf`](../related-work/pdfs/hakes.pdf)
- **Venue:** PVLDB 2025
- **PDF:** [PVLDB](https://www.vldb.org/pvldb/vol18/p3049-ooi.pdf)
- **Abstract:** Graph indexes give low latency/high recall but build slowly, contend under concurrent reads/writes, and scale poorly. HAKES uses a two-stage filter (compressed vectors) + refine index with lightweight ML tuning and early termination, decouples learned-parameter management for writes, and serves it in a disaggregated distributed DB. It beats 12 index baselines and 3 distributed vector DBs, with up to 16× higher throughput.
- **Understanding**
  - **Problem:** Pure graph indexes do not scale under concurrent updates; monolithic indexes cannot disaggregate filter vs. refine resources.
  - **Technique:** **k-means IVF + PQ filter stage** on IndexWorkers; full vectors on RefineWorkers; default sharding by vector ID (disaggregated filter/refine architecture).
- **Hardware:** Distributed memory (IndexWorker / RefineWorker disaggregation)
- **Partitioning / Sharding:** **k-means IVF + PQ** two-stage; RefineWorker sharding by vector ID
- **Rationale:** IVF filter is small and replicable; disaggregates filter vs. refine resources

---

### SPANN: Highly-efficient Billion-scale Approximate Nearest Neighbor Search

- **Category:** 3 only *(partition-intrinsic IVF — not a multi-node systems paper)*
- **Deployment scope:** **Single-node only** in the NeurIPS 2021 paper (workstation + NVMe SSD). Distributed posting dispatch appears in Microsoft production write-ups but is **not evaluated in the paper** — do not cite SPANN as a distributed-systems result.
- **Local PDF:** [`spann.pdf`](../related-work/pdfs/spann.pdf)
- **Venue:** NeurIPS 2021
- **PDF:** [NeurIPS](https://papers.nips.cc/paper_files/paper/2021/file/299dc35e747eb77177d9cea10a802da2-Paper.pdf)
- **Abstract:** Pure in-memory ANNS is expensive at billion scale; hybrid memory–SSD ANNS is needed. SPANN keeps centroids in memory and posting lists on disk, using hierarchical balanced clustering and query-aware posting pruning to cut disk accesses while keeping recall. It is 2× faster than DiskANN at 90% recall with same memory on three billion-scale datasets.
- **Understanding**
  - **Problem:** Disk-resident graph search (DiskANN) has sequential hop chains; billion-scale needs cheaper memory footprint with parallel I/O on **one machine**.
  - **Technique:** **Hierarchical Balanced Clustering (HBC)** with bounded posting sizes, **closure assignment** (boundary duplication), and **query-aware dynamic pruning** of which posting lists to read (IVF-style nprobe at posting granularity).
- **Hardware:** Single-node workstation (~64–128 GB DRAM + NVMe SSD) — all paper benchmarks
- **Partitioning / Sharding:** **HBC k-means clusters** + disk **posting lists**; pruning selects a subset of lists per query
- **Rationale:** IVF postings enable parallel SSD reads without graph hop chains; category **#3** algorithm — systems sharding of postings across machines is a **separate engineering step**, not part of this paper's evaluation

---

### Distributed Similarity Search in High Dimensions using Locality Sensitive Hashing

- **Category:** 2
- **Deployment scope:** Multi-node (peer overlay)
- **Local PDF:** `NOT_DOWNLOADED` → [`sec4/haghani-distributed-lsh-edbt-2009.pdf`](../related-work/pdfs/sec4/haghani-distributed-lsh-edbt-2009.pdf) (openproceedings mirror; use online PDF)
- **§4 centroid-locality:** covered in §4 deep dive
- **Venue:** EDBT 2009 (extends WebDB 2008 work; often cited in distributed LSH lineage)
- **PDF:** [OpenProceedings](https://openproceedings.org/2009/conf/edbt/HaghaniMA09.pdf) · [ACM](https://dl.acm.org/doi/pdf/10.1145/1516360.1516446)
- **Abstract:** We consider distributed KNN/range search in high dimensions using LSH. We map LSH buckets to a structured peer overlay with locality-preserving, load-balanced properties, enabling efficient incremental top-K KNN and range queries with fewer network hops. Real-world evaluations show major gains vs. state-of-the-art distributed similarity search.
- **Understanding**
  - **Problem:** Centralized LSH does not scale; naive bucket-to-peer mapping causes hot spots and excessive hops.
  - **Technique:** **ξ mapping** of LSH bucket labels to a linear peer ID space — nearby labels on adjacent Chord peers; **predicted label distribution** spreads buckets across peers; incremental ring forwarding or multi-probe bucket jumps at query time.
- **Hardware:** **Multi-node P2P** cluster; bucket indexes **in memory** on peers (eval up to 1M global peers, 1k peers per local DHT)
- **Partitioning / Sharding:** **LSH buckets** mapped to peers
- **Rationale:** Colocating nearby buckets reduces peers visited per query vs. random assignment

---

### SPIRE: Scalable Distributed Vector Search via Accuracy Preserving Index Construction

- **Category:** 2
- **Deployment scope:** Multi-node (46 nodes)
- **Local PDF:** [`sec4/spire.pdf`](../related-work/pdfs/sec4/spire.pdf)
- **§4 centroid-locality:** covered in §4
- **Venue:** arXiv (VecDB @ ICML 2025 workshop)
- **PDF:** [arXiv](https://arxiv.org/pdf/2512.17264.pdf)
- **Abstract:** Scaling Approximate Nearest Neighbor Search (ANNS) to billions of vectors requires distributed indexes that balance accuracy, latency, and throughput. Yet existing index designs struggle with this tradeoff. This paper presents SPIRE, a scalable vector index based on two design decisions. First, it identifies a balanced partition granularity that avoids read-cost explosion. Second, it introduces an accuracy-preserving recursive construction that builds a multi-level index with predictable search cost and stable accuracy. In experiments with up to 8 billion vectors across 46 nodes, SPIRE achieves high scalability and up to 9.64X higher throughput than state-of-the-art systems.
- **Understanding**
  - **Problem:** Naive distributed graph sharding loses recall; coarse IVF sharding explodes read cost per query.
  - **Technique:** **Hierarchical k-means** with tuned partition granularity + **accuracy-preserving recursive multi-level index** construction across nodes.
- **Hardware:** 46-node cluster; 8B vectors; distributed SSD + memory
- **Partitioning / Sharding:** **Hierarchical k-means** multi-level distributed index
- **Rationale:** Hierarchy limits probes like IVF while recursive construction preserves accuracy better than naive graph cuts

---

### DISTRIBUTEDANN: Efficient Scaling of a Single DiskANN Graph Across Thousands of Computers

- **Category:** 2
- **Deployment scope:** Multi-node (1000+ machines)
- **Local PDF:** [`distributedann.pdf`](../related-work/pdfs/distributedann.pdf)
- **Venue:** arXiv / VecDB @ ICML 2025
- **PDF:** [arXiv](https://arxiv.org/pdf/2509.06046.pdf)
- **Abstract:** We present DISTRIBUTEDANN, a distributed vector search service that makes it possible to search over a single 50 billion vector graph index spread across over a thousand machines that offers 26ms median query latency and processes over 100,000 queries per second. This is 6x more efficient than existing partitioning and routing strategies that route the vector query to a subset of partitions in a scale out vector search system. DISTRIBUTEDANN is built using two well-understood components: a distributed key-value store and an in-memory ANN index. DISTRIBUTEDANN has replaced conventional scale-out architectures for serving the Bing search engine, and we share our experience from making this transition.
- **Understanding**
  - **Problem:** Partition-and-route (clustered shards + independent indexes) wastes work at **hundreds of billions** of vectors: empirically ~**P·log(|X|/P)** search cost vs. **log |X|** for one graph index, plus recall loss vs. their previous Bing production system.
  - **Technique:** **Spatial/graph-aware partition** of vectors in KV storage + **single logical DiskANN graph** searched globally; in-memory **head index**; near-data scoring on KV hosts.
- **Hardware:** 1000+ nodes; distributed KV + in-memory head index
- **Partitioning / Sharding:** **Graph-aware storage layout** (not N independent shard-local graphs)
- **Rationale:** One navigable graph at 50B+ scale; see **When #2 beats #1** below

**When category #2 (one logical graph) beats category #1 (independent indexes) — Bing-scale characteristics:**

| Factor | Why it pushes toward one global graph (DistributedANN) |
|--------|--------------------------------------------------------|
| **Corpus scale** | **50B–100B+** vectors on one unified retrieval index; fixed small partitions ⇒ **P is huge** ⇒ P·log(N/P) dominates |
| **Single retrieval universe** | Web-scale **one index over all documents**, not per-tenant isolated shards; global kNN quality matters |
| **Graph index family** | DiskANN beam search needs **cross-partition graph connectivity**; naive cut ⇒ recall drop (paper: +7.8 / +4.5 pp recall@5/@200 vs prior production) |
| **Fixed partition size for failover** | Online systems use partitions **smaller than machine capacity** for fast replica bring-up ⇒ many partitions even if one machine could hold more |
| **Throughput SLO** | Prior clustered partition-and-route left **6× headroom** on same footprint when switching to single logical graph |
| **Storage substrate** | KV store as **shared logical disk** + near-data compute — architecture fits **one index layout**, not scatter-gather merge |

**When #1 remains rational:** multi-tenant products (Milvus/Pinecone), **IVF with route-to-subset-of-shards** (not probe-all), smaller corpora, ingest/compaction isolation, HNSW-per-shard where global graph is impractical.

---

### CoTra: Towards Efficient and Scalable Distributed Vector Search with RDMA

- **Category:** 2
- **Deployment scope:** Multi-node (8–16 machines)
- **Local PDF:** [`sec4/cotra.pdf`](../related-work/pdfs/sec4/cotra.pdf)
- **§4 centroid-locality:** covered in §4 deep dive
- **Venue:** arXiv (SIGMOD 2026 listed on some mirrors)
- **PDF:** [arXiv](https://arxiv.org/pdf/2507.06653.pdf)
- **Abstract:** Similarity-based vector search facilitates many important applications such as search and recommendation but is limited by the memory capacity and bandwidth of a single machine due to large datasets and intensive data read. In this paper, we present CoTra, a system that scales up vector search for distributed execution. We observe a tension between computation and communication efficiency, which is the main challenge for good scalability, i.e., handling the local vectors on each machine independently blows up computation as the pruning power of vector index is not fully utilized, while running a global index over all machines introduces rich data dependencies and thus extensive communication. To resolve such tension, we leverage the fact that vector search is approximate in nature and robust to asynchronous execution. In particular, we run collaborative vector search over the machines with algorithm-system co-designs including clustering-based data partitioning to reduce communication, asynchronous execution to avoid communication stall, and task push to reduce network traffic. To make collaborative search efficient, we introduce a suite of system optimizations including task scheduling, communication batching, and storage format. We evaluate CoTra on real datasets and compare with four baselines. The results show that when using 16 machines, the query throughput of CoTra scales to 9.8-13.4x over a single machine and is 2.12-3.58x of the best-performing baseline at 0.95 recall@10.
- **Understanding**
  - **Problem:** Scatter-gather over shard-local graphs wastes work; a single global graph needs remote hops when vectors are randomly sharded.
  - **Technique:** **Balanced k-means** colocates similar vectors on the same machine so **graph traversal** touches mostly local neighbors; **one global proximity graph** spans machines; **Co-Search + Pull-Push** over RDMA when hops cross partitions (no vector replication).
- **Hardware:** **Multi-node RDMA cluster** (8–16 machines); vectors and graph adjacency **in memory**
- **Partitioning / Sharding:** **Balanced k-means machine assignment** + global graph
- **Rationale:** k-means placement concentrates query traffic; global graph avoids per-shard recall loss

---

### RED-ANNS: An RDMA-Enabled Distributed Framework for Graph-Based ANN Search

- **Category:** 2
- **Deployment scope:** Multi-node (RDMA cluster)
- **Local PDF:** [`sec4/red-anns.pdf`](../related-work/pdfs/sec4/red-anns.pdf)
- **§4 centroid-locality:** covered in §4
- **Venue:** PVLDB 2026
- **PDF:** [Author copy](https://kay21s.github.io/RED-ANNS-VLDB2026.pdf)
- **Abstract:** MapReduce-style graph sharding cuts indexing efficiency and adds overhead. RED-ANNS keeps a logically full graph in shared memory and searches via RDMA, using locality-aware placement, affinity scheduling, and dependency-relaxed best-first search to hide remote access cost. It is up to 2.5× faster than MapReduce-style approaches and 5.3× vs. open-source vector DBs.
- **Understanding**
  - **Problem:** Naive HNSW/DiskANN sharding severs cross-partition edges, degrading recall and adding merge overhead.
  - **Technique:** **Locality-aware placement** preserving logical graph connectivity; **RDMA** remote memory access; relaxed best-first search to overlap hops.
- **Hardware:** Distributed memory + **RDMA** (disaggregated memory style)
- **Partitioning / Sharding:** **Avoid naive graph cut**; placement for locality, not independent subgraphs
- **Rationale:** RDMA makes remote hops cheap enough to treat the graph as logically global

---

### SHINE: A Scalable HNSW Index in Disaggregated Memory

- **Category:** 2
- **Deployment scope:** Multi-node (disaggregated memory)
- **Local PDF:** [`shine.pdf`](../related-work/pdfs/shine.pdf)
- **Venue:** arXiv
- **PDF:** [arXiv](https://arxiv.org/pdf/2507.17647.pdf)
- **Abstract:** Approximate nearest neighbor (ANN) search is a fundamental problem in computer science for which in-memory graph-based methods, such as Hierarchical Navigable Small World (HNSW), perform exceptionally well. To scale beyond billions of high-dimensional vectors, the index must be distributed. The disaggregated memory architecture physically separates compute and memory into two distinct hardware units and has become popular in modern data centers. Both units are connected via RDMA networks that allow compute nodes to directly access remote memory and perform all the computations, posing unique challenges for disaggregated indexes.   In this work, we propose a scalable HNSW index for ANN search in disaggregated memory. In contrast to existing distributed approaches, which partition the graph at the cost of accuracy, our method builds a graph-preserving index that reaches the same accuracy as a single-machine HNSW. Continuously fetching high-dimensional vector data from remote memory leads to severe network bandwidth limitations, which we overcome by employing an efficient caching mechanism. Since answering a single query involves processing numerous unique graph nodes, caching alone is not sufficient to achieve high scalability. We logically combine the caches of the compute nodes to increase the overall cache effectiveness and confirm the efficiency and scalability of our method in our evaluation.
- **Understanding**
  - **Problem:** Disaggregated memory forces graph data remote; naive partitioning breaks HNSW navigability.
  - **Technique:** **Logical graph partition** with **cross-compute joint cache**; route query to best partition first; RDMA-backed remote graph storage.
- **Hardware:** Disaggregated memory + RDMA
- **Partitioning / Sharding:** **Graph-preserving logical partitions** + query routing
- **Rationale:** Minimize cuts that would destroy small-world navigation

---

### d-HNSW: A High-performance Vector Search Engine on Disaggregated Memory

- **Category:** 2
- **Deployment scope:** Multi-node (disaggregated memory)
- **Local PDF:** [`d-hnsw.pdf`](../related-work/pdfs/d-hnsw.pdf)
- **Venue:** arXiv
- **PDF:** [arXiv](https://arxiv.org/pdf/2603.13591.pdf)
- **Abstract:** Efficient vector search is essential for powering large-scale AI applications, such as LLMs. Existing solutions are designed for monolithic architectures where compute and memory are tightly coupled. Recently, disaggregated architecture breaks this coupling by separating compution and memory resources into independently scalable pools to improve utilization. However, applying vector database on disaggregated memory system brings unique challenges to system design due to its graph-based index. We present d-HNSW, the first RDMA-based vector search engine optimized for disaggregated memory systems. d-HNSW preserves HNSW's high accuracy while addressing the new system-level challenges introduced by disaggregation: 1) network inefficiency from pointer-chasing traversals, 2) non-contiguous remote memory layout induced by dynamic insertions, 3) redundant data transfers in batch workloads, and 4) resource underutilization due to sequential execution. d-HNSW tackles these challenges through a set of hardware-algorithm co-designed techniques, including 1) balanced clustering with a lightweight representative index to reduce network round-trips and ensure predictable latency, 2) an RDMA-friendly graph layout that preserves data contiguity under dynamic insertions, 3) query-aware data loading to eliminate redundant fetches across batch queries, and 4) a pipelined execution model that overlaps RDMA transfers with computation to hide network latency and improve throughput. Our evaluation results in a public cloud show that d-HNSW achieves up to < 10-2x query latency and > 100x query throughput compared to other baselines, while maintaining a high recall of 94%.
- **Understanding**
  - **Problem:** Remote memory bandwidth is the bottleneck for graph traversal on disaggregated systems.
  - **Technique:** **meta-HNSW** routes to **sub-HNSW per partition** (uniform sampling splits); fetch only the sub-graph needed for each query.
- **Hardware:** Disaggregated memory + RDMA
- **Partitioning / Sharding:** **meta-HNSW + sub-HNSW** hierarchical partition
- **Rationale:** Two-level structure limits remote fetches to one partition’s subgraph per routing step

---

### BatANN: Passing the Baton — High Throughput Distributed Disk-Based Vector Search

- **Category:** 2
- **Deployment scope:** Multi-node (multi-server NVMe)
- **Local PDF:** [`batann.pdf`](../related-work/pdfs/batann.pdf)
- **Venue:** arXiv
- **PDF:** [arXiv](https://arxiv.org/pdf/2512.09331.pdf)
- **Abstract:** Vector search underpins modern information-retrieval systems, including retrieval-augmented generation (RAG) pipelines and search engines over unstructured text and images. As datasets scale to billions of vectors, disk-based vector search has emerged as a practical solution. However, looking to the future, we must anticipate datasets too large for any single server and throughput demands that exceed the limits of locally attached SSDs. We present BatANN, a distributed disk-based approximate nearest neighbor (ANN) system that retains the logarithmic search efficiency of a single global graph while achieving near-linear throughput scaling in the number of servers. Our core innovation is that when accessing a neighborhood which is stored on another machine, we send the full state of the query to the other machine to continue executing there for improved locality. On 1B-point datasets at 0.95 recall using 10 servers, BatANN achieves 3.5-5.59x of the scatter-gather baseline and 1.44-2.09x the throughput of DistributedANN, respectively, while maintaining mean latency below 3 ms. Moreover, we get these results on standard TCP. To our knowledge, BatANN is the first open-source distributed disk-based vector search system to operate over a single global graph.
- **Understanding**
  - **Problem:** Distributed disk graph search with request/response per hop doubles network round trips; scatter-gather over shard-local graphs loses global graph quality.
  - **Technique:** **Neighborhood-aware graph partition** + **baton-passing** (transfer full beam/search state to the owning server); single global Vamana graph.
- **Hardware:** Multi-server NVMe SSD + TCP
- **Partitioning / Sharding:** **Graph partition** minimizing cross-server hops (~20–25% of hops cross servers)
- **Rationale:** Preserves one graph while making cross-server continuation cheap via state handoff

---

### DSANN: Approximate Nearest Neighbor Search of Large Scale Vectors on Distributed Storage

- **Category:** 2
- **Deployment scope:** Multi-node (distributed storage)
- **Local PDF:** [`dsann.pdf`](../related-work/pdfs/dsann.pdf)
- **Venue:** arXiv
- **PDF:** [arXiv](https://arxiv.org/pdf/2510.17326.pdf)
- **Abstract:** Approximate Nearest Neighbor Search (ANNS) in high-dimensional space is an essential operator in many online services, such as information retrieval and recommendation. Indices constructed by the state-of-the-art ANNS algorithms must be stored in single machine's memory or disk for high recall rate and throughput, suffering from substantial storage cost, constraint of limited scale and single point of failure. While distributed storage can provide a cost-effective and robust solution, there is no efficient and effective algorithms for indexing vectors in distributed storage scenarios. In this paper, we present a new graph-cluster hybrid indexing and search system which supports Distributed Storage Approximate Nearest Neighbor Search, called DSANN. DSANN can efficiently index, store, search billion-scale vector database in distributed storage and guarantee the high availability of index service. DSANN employs the concurrent index construction method to significantly reduces the complexity of index building. Then, DSANN applies Point Aggregation Graph to leverage the structural information of graph to aggregate similar vectors, optimizing storage efficiency and improving query throughput via asynchronous I/O in distributed storage. Through extensive experiments, we demonstrate DSANN can efficiently and effectively index, store and search large-scale vector datasets in distributed storage scenarios.
- **Understanding**
  - **Problem:** DiskANN sequential hops and SPANN full posting-list reads are unbearable on distributed file systems (0.1–10 ms I/O).
  - **Technique:** **Cluster layer** for distributed placement + **Point Aggregation Graph (PAG)** inside clusters; sample aggregation points for in-memory graph traversal; async overlap for residual points on DFS.
- **Hardware:** Distributed file system (e.g., Pangu) + commodity cluster
- **Partitioning / Sharding:** **Cluster partition** + graph inside each cluster
- **Rationale:** Hybrid: coarse clusters enable parallel I/O; in-cluster graph handles fine search

---

### CXL-ANNS: Software-Hardware Collaborative Memory Disaggregation for Billion-Scale ANN

- **Category:** 2
- **Deployment scope:** Multi-node (CXL pool)
- **Local PDF:** [`cxl-anns.pdf`](../related-work/pdfs/cxl-anns.pdf)
- **Venue:** USENIX ATC 2023
- **PDF:** [USENIX](https://www.usenix.org/system/files/atc23-jang.pdf)
- **Abstract:** CXL-ANNS disaggregates DRAM via CXL into a memory pool holding billion-point graphs without accuracy loss, but far-memory latency hurts search. It caches hot neighbors locally, prefetches likely next nodes from graph traversal behavior, and parallelizes search across CXL network components with relaxed dependencies. Evaluations show 111.1× QPS and 93.3% lower latency vs. tested large-scale ANNS baselines.
- **Understanding**
  - **Problem:** Billion-point graphs exceed local DRAM; CXL far memory adds latency and bandwidth limits.
  - **Technique:** Graph in CXL pool; **hot neighbor cache** on host; prefetch by traversal pattern; **column-wise vector sharding** across CXL expanders for parallel sub-distance compute.
- **Hardware:** CXL memory pool + host DRAM cache
- **Partitioning / Sharding:** **Column-wise embedding sharding** across expanders; graph node caching (not query routing shards)
- **Rationale:** Dimension sharding uses all expander bandwidth; caching cuts far-memory graph hops

---

### CockroachDB C-SPANN

- **Category:** 2
- **Deployment scope:** Multi-node (KV ranges)
- **Local PDF:** NOT_DOWNLOADED (blog only)
- **Venue:** Product / blog
- **PDF / Source:** [Distributed vector indexing](https://www.cockroachlabs.com/blog/distributed-vector-indexing-cockroachdb/) · [Real-time indexing](https://www.cockroachlabs.com/blog/cspann-real-time-indexing-billions-vectors/)
- **Abstract (from docs):** C-SPANN stores hierarchical k-means tree partitions as **KV ranges** that split/merge/rebalance like table data; SPANN/SPFresh-inspired posting layout on distributed storage.
- **Understanding**
  - **Problem:** Vector indexes must rebalance with SQL data without a central coordinator.
  - **Technique:** **Hierarchical k-means tree** mapped to **CockroachDB ranges** (partitions = contiguous row ranges).
- **Hardware:** Distributed SQL cluster + cloud storage
- **Partitioning / Sharding:** **k-means tree partitions = KV ranges**
- **Rationale:** Same rebalance machinery as relational data—strong fit for partition-first + object/KV storage


---

## 3. Partition-Intrinsic Indexes (IVF / LSH / Tree)

Partitioning here is **inside the index**, not an external systems-layer shard. Even on one machine, search probes only a subset of IVF lists, LSH buckets, or tree leaves. Papers in this section may also appear in §1 or §2 when the same partition structure is reused for distributed placement.

**Single-node graph/disk indexes** (PipeANN, DiskANN, Starling, …) are listed here only when disk **segment/block layout** is partition-like; they do not use coarse vector-space partitioning for search pruning.

### SPFresh: Incremental In-Place Update for Billion-Scale Vector Search

- **Category:** 3
- **Deployment scope:** Single-node (NVMe + DRAM)
- **Local PDF:** [`spfresh.pdf`](../related-work/pdfs/spfresh.pdf)
- **Venue:** SOSP 2023
- **PDF:** [Microsoft Research](https://www.microsoft.com/en-us/research/wp-content/uploads/2023/08/SPFresh_SOSP.pdf)
- **Abstract:** Approximate Nearest Neighbor Search (ANNS) on high dimensional vector data is now widely used in various applications, including information retrieval, question answering, and recommendation. As the amount of vector data grows continuously, it becomes important to support updates to vector index, the enabling technique that allows for efficient and accurate ANNS on vectors. Because of the curse of high dimensionality, it is often costly to identify the right neighbors of a new vector, a necessary process for index update. To amortize update costs, existing systems maintain a secondary index to accumulate updates, which are merged with the main index by globally rebuilding the entire index periodically. However, this approach has high fluctuations of search latency and accuracy, not to mention that it requires substantial resources and is extremely time-consuming to rebuild. We introduce SPFresh, a system that supports in-place vector updates. At the heart of SPFresh is LIRE, a lightweight incremental rebalancing protocol to split vector partitions and reassign vectors in the nearby partitions to adapt to data distribution shifts. LIRE achieves low-overhead vector updates by only reassigning vectors at the boundary between partitions, where in a high-quality vector index the amount of such vectors is deemed small. With LIRE, SPFresh provides superior query latency and accuracy to solutions based on global rebuild, with only 1% of DRAM and less than 10% cores needed at the peak compared to the state-of-the-art, in a billion scale disk-based vector index with a 1% of daily vector update rate.
- **Understanding**
  - **Problem:** Billion-scale IVF/disk indexes cannot afford full rebuilds on every update; boundary-heavy partitions also hurt tail latency.
  - **Technique:** Extends SPANN-style hierarchical balanced k-means partitions with **LIRE**—lightweight split and boundary reassignment only where partitions drift—so postings stay balanced without global reconstruction.
- **Hardware:** Single-node NVMe SSD + DRAM (centroid graph in memory, postings on SSD)
- **Partitioning / Sharding:** Hierarchical **balanced k-means clustering** (SPANN lineage)
- **Rationale:** Cluster partitions bound I/O per probe; k-means groups similar vectors so queries touch few postings; balancing avoids stragglers

---

### PASE: PostgreSQL Ultra-High-Dimensional Approximate Nearest Neighbor Search Extension

- **Category:** 3
- **Deployment scope:** Single-node (PostgreSQL)
- **Local PDF:** [`pase-sigmod2020.pdf`](../related-work/pdfs/pase-sigmod2020.pdf)
- **Venue:** SIGMOD 2020 (Industry)
- **PDF:** [ACM](https://dl.acm.org/doi/pdf/10.1145/3318464.3386131)
- **Abstract:** PASE is a PostgreSQL extension for ultra-high-dimensional ANN, integrating quantization-based and graph-based nearest-neighbor search under one framework so composite SQL vector queries can run on large datasets inside an RDBMS.
- **Understanding**
  - **Problem:** Embedding search must coexist with relational filters and transactions, not live in a separate vector-only engine.
  - **Technique:** Embeds **k-means IVFFlat** (and graph variants) as PostgreSQL index access methods, reusing the DB’s storage and query planner.
- **Hardware:** Single-node memory/disk (PostgreSQL)
- **Partitioning / Sharding:** **k-means IVFFlat**
- **Rationale:** IVF+k-means is the standard Faiss coarse quantizer and maps cleanly onto page-oriented RDBMS storage

---

### RAIRS: Optimizing Redundant Assignment and List Layout for IVF-Based ANN Search

- **Category:** 3
- **Deployment scope:** Single-node IVF list layout
- **Local PDF:** [`rairs.pdf`](../related-work/pdfs/rairs.pdf)
- **Venue:** SIGMOD 2026 (PACMMOD)
- **PDF:** [Author copy](https://www.shimin-chen.com/papers/rairs-pacmmod26.pdf)
- **Abstract:** Redundant IVF assignment reduces missed neighbors but naive second-list selection is poor in Euclidean space, and duplicate vectors across lists cause redundant distance work. RAIRS proposes RAIR (Amplified Inverse Residual) for optimized Euclidean redundant assignment and SEIL, a shared-cell list layout that cuts repeated distance computation. On real datasets RAIRS beats prior redundant-assignment methods and is up to 1.33× faster than IVF-PQ Fast Scan with refinement.
- **Understanding**
  - **Problem:** Redundant IVF (vectors in multiple lists) improves recall but wastes distance computation and list I/O if layout is naive.
  - **Technique:** Optimized **redundant assignment policy (RAIR)** plus **SEIL** inverted-list layout that shares computation across duplicate entries in k-means IVF partitions.
- **Hardware:** General IVF workloads (single- and multi-node)
- **Partitioning / Sharding:** **k-means IVF** with optimized redundant list assignment
- **Rationale:** Partition structure is unchanged; the paper optimizes how vectors are duplicated across IVF lists after k-means clustering

---

### VHP: Approximate Nearest Neighbor Search via Virtual Hypersphere Partitioning

- **Category:** 3
- **Deployment scope:** Single-node (LSH)
- **Local PDF:** [`vhp.pdf`](../related-work/pdfs/vhp.pdf)
- **Venue:** PVLDB 2020
- **PDF:** [PVLDB](http://www.vldb.org/pvldb/vol13/p1443-lu.pdf)
- **Abstract:** LSH-based c-ANN search often explores unbounded, irregular regions, hurting efficiency. VHP imposes a virtual hypersphere around the query and examines only points inside it, emulated by coordinated physical hyperspheres in projection subspaces with principled radius selection. VHP stores LSH projections in B+-trees and expands radii until success probability is met, with guarantees for arbitrarily small c≥1; experiments show up to 2× speedup over SOTA on billion-scale data.
- **Understanding**
  - **Problem:** Standard LSH probing explores irregular, unbounded regions—bad for disk I/O and hard to reason about recall.
  - **Technique:** **Virtual hypersphere partitioning** over LSH projections: bounded spherical search regions with radius expansion until probabilistic guarantees are met; B+-tree layout on disk.
- **Hardware:** Single-node SSD/disk
- **Partitioning / Sharding:** **Virtual hypersphere** over LSH projection buckets
- **Rationale:** Bounded partitions give predictable I/O per probe vs. open-ended LSH bucket chains

---

### SK-LSH: An Efficient Index Structure for Approximate Nearest Neighbor Search

- **Category:** 3
- **Deployment scope:** Single-node (disk LSH)
- **Local PDF:** [`sk-lsh.pdf`](../related-work/pdfs/sk-lsh.pdf)
- **Venue:** PVLDB 2014
- **PDF:** [PVLDB](https://www.vldb.org/pvldb/vol7/p745-liu.pdf)
- **Abstract:** State-of-the-art LSH methods suffer many random disk page accesses when verifying candidates. SK-LSH (SortingKeys-LSH) defines a distance measure and linear order so nearby points are stored locally, letting ANN search touch only a few pages per index file. Experiments on real datasets show superior efficiency and accuracy vs. LSB, C2LSH, and CK-Means.
- **Understanding**
  - **Problem:** LSH bucket verification causes random I/O because hash buckets are not spatially ordered on disk.
  - **Technique:** **Compound hash keys + sorting** so geometrically close LSH buckets land in contiguous pages (SortingKeys-LSH).
- **Hardware:** Single-node disk/SSD
- **Partitioning / Sharding:** **Hash-based LSH buckets** with sorted on-disk layout
- **Rationale:** Physical colocation of nearby buckets turns random reads into sequential scans

---

### Intelligent Probing for Locality Sensitive Hashing: Multi-Probe LSH and Beyond

- **Category:** 3
- **Deployment scope:** Single-node (multi-probe LSH)
- **Local PDF:** [`intelligent-probing-for-locality-sensitive-hashing.pdf`](../related-work/pdfs/intelligent-probing-for-locality-sensitive-hashing.pdf)
- **Venue:** PVLDB 2017 (Vol. 10 retrospective; original Multi-probe LSH: VLDB 2007)
- **PDF:** [PVLDB](https://www.vldb.org/pvldb/vol10/p2021-lv.pdf)
- **Abstract:** LSH needs many hash tables for good quality; multi-probe LSH strategically probes neighboring buckets using query-dependent probabilities of where similar objects fall. This retrospective revisits multi-probe LSH design and recent developments. Intelligent probing can cut hash tables by an order of magnitude while keeping high search quality.
- **Understanding**
  - **Problem:** Many independent LSH tables waste memory and I/O; single-table LSH has poor recall.
  - **Technique:** **Multi-probe** within LSH bucket space—visit neighboring buckets by rank probability instead of maintaining many hash tables.
- **Hardware:** Single-node memory/disk
- **Partitioning / Sharding:** **LSH hash buckets** + intelligent multi-probe routing
- **Rationale:** Partition structure remains LSH buckets; probing strategy reduces tables needed for same recall

---

### LEQAT: Learning-based Query Optimization for Multi-Probe Approximate Nearest Neighbor Search

- **Category:** 3
- **Deployment scope:** Single-node query optimization (not sharding)
- **Local PDF:** [`leqat.pdf`](../related-work/pdfs/leqat.pdf)
- **Venue:** VLDB Journal 2023
- **PDF:** [Springer](https://link.springer.com/article/10.1007/s00778-022-00762-0)
- **Abstract:** Multi-probe ANNS often uses fixed per-query partition/search configs, yielding suboptimal accuracy–efficiency trade-offs. LEQAT formalizes per-query optimization as 0–1 knapsack using estimated kNN distribution over partitions, with an ML model plus efficient optimizers to pick partitions and per-partition search depth. Applied to IVF/HNSW/SSG under clustering-based partitioning, it cuts latency up to 58% and improves throughput up to 3.9×.
- **Understanding**
  - **Problem:** Fixed nprobe or fixed per-partition search depth wastes work on easy queries and under-searches hard ones.
  - **Technique:** Learned model estimates kNN distribution across **k-means IVF partitions**; solves **0–1 knapsack** to pick which partitions and how deep to search per query.
- **Hardware:** General (distributed, disk, GPU settings in paper)
- **Partitioning / Sharding:** Assumes existing **k-means IVF**; optimizes probe allocation
- **Rationale:** Partition geometry is fixed; gains come from query-aware allocation of search budget across partitions

---

### CrackIVF: Cracking Vector Search Indexes

- **Category:** 3
- **Deployment scope:** Single-node
- **Local PDF:** [`crackivf.pdf`](../related-work/pdfs/crackivf.pdf)
- **Venue:** PVLDB 2025
- **PDF:** [PVLDB](https://www.vldb.org/pvldb/vol18/p3951-mageirakos.pdf)
- **Abstract:** Embedding data lakes cannot build full ANNS indexes for every dataset upfront. CrackIVF is an adaptive partition-based index that starts small, answers queries immediately, and incrementally adapts to the workload until it matches conventionally built indexes. It can serve 1M+ queries before some baselines finish building, with 10–1000× faster initialization—ideal for cold or rarely used datasets.
- **Understanding**
  - **Problem:** Choosing nlist upfront for IVF is wrong without knowing future query distribution; full index build blocks time-to-first-query.
  - **Technique:** **Query-driven incremental cracking**: add/refine k-means IVF partitions around query paths (CRACK + REFINE) as queries arrive, with controls on when/where to split.
- **Hardware:** Single-node memory (FAISS-IVF baseline)
- **Partitioning / Sharding:** **Incremental k-means IVF** built from query workload
- **Rationale:** Partitions emerge where queries actually search, avoiding wasted indexing on cold regions

---

### I-LSH: I/O Efficient c-Approximate Nearest Neighbor Search in High-Dimensional Space

- **Category:** 3
- **Deployment scope:** Single-node (external memory)
- **Local PDF:** [`i-lsh.pdf`](../related-work/pdfs/i-lsh.pdf)
- **Venue:** ICDE 2019
- **PDF:** [IEEE](https://doi.org/10.1109/ICDE.2019.00169)
- **Abstract:** I-LSH is an external-memory LSH method for c-ANN that incrementally expands search radius in projected space—finding nearest points per projection instead of exponential bucket expansion—reducing sequential disk I/O while preserving theoretical guarantees.
- **Understanding**
  - **Problem:** External-memory LSH pays too much disk I/O when probing many buckets with exponential radius growth.
  - **Technique:** **Incremental radius expansion** on LSH projections with I/O-efficient bucket layout on SSD/disk.
- **Hardware:** Single-node SSD/disk (external memory)
- **Partitioning / Sharding:** **Hash-based LSH buckets** with incremental probing
- **Rationale:** LSH is inherently partition-based; I-LSH optimizes how many buckets are touched per I/O

---

### I/O Efficient Approximate Nearest Neighbour Search based on Learned Functions

- **Category:** 3
- **Deployment scope:** Single-node (learned disk lists)
- **Local PDF:** [`i-o-efficient-approximate-nearest-neighbour-search.pdf`](../related-work/pdfs/i-o-efficient-approximate-nearest-neighbour-search.pdf)
- **Venue:** ICDE 2020
- **PDF:** [IEEE](https://doi.org/10.1109/ICDE48307.2020.00032)
- **Abstract:** Hash-based ANNS methods rarely optimize external-memory sequential I/O. This paper builds lists of point IDs ordered by learned mapping functions (linear/non-linear) that preserve high-dimensional order, plus an I/O-efficient search framework. On six benchmarks it beats state-of-the-art external-memory ANNS in I/O efficiency and accuracy.
- **Understanding**
  - **Problem:** Trees and LSH on disk cause too many random or redundant page reads in high dimensions.
  - **Technique:** **Learned space partitioning functions** order point IDs into sequential lists; search follows learned order with I/O-aware traversal (not k-means IVF).
- **Hardware:** Single-node disk
- **Partitioning / Sharding:** **Learned function-based** space partitions (ordered ID lists)
- **Rationale:** Learned orderings cluster likely neighbors on the same pages, improving sequential I/O

---

### ScaNN: Accelerating Large-Scale Inference with Anisotropic Vector Quantization

- **Category:** 3
- **Deployment scope:** Single-node
- **Local PDF:** [`scann.pdf`](../related-work/pdfs/scann.pdf)
- **Venue:** ICML 2020
- **PDF:** [PMLR](http://proceedings.mlr.press/v119/guo20h/guo20h.pdf)
- **Abstract:** Quantization scales maximum inner-product search, but standard methods minimize reconstruction error. ScaNN uses anisotropic loss that penalizes errors parallel to each datapoint’s residual more than orthogonal errors, improving MIPS-relevant accuracy. The open-source implementation achieves state-of-the-art on ann-benchmarks.com.
- **Understanding**
  - **Problem:** Standard PQ minimizes reconstruction error, not inner-product ranking accuracy.
  - **Technique:** **k-means tree partitioning** (`num_leaves`) + **anisotropic vector quantization** + asymmetric hashing within leaves.
- **Hardware:** Single-node CPU (Google production)
- **Partitioning / Sharding:** **k-means tree** coarse partitions
- **Rationale:** Tree partitions prune most vectors; anisotropic PQ improves accuracy within each leaf partition

---

### Curator: Efficient Indexing for Multi-Tenant Vector Databases

- **Category:** 3
- **Deployment scope:** Single-node (multi-tenant in-memory)
- **Local PDF:** [`curator.pdf`](../related-work/pdfs/curator.pdf)
- **Venue:** VLDB 2024
- **PDF:** [arXiv](https://arxiv.org/pdf/2401.07119)
- **Abstract:** Vector databases have emerged as key enablers for bridging intelligent applications with unstructured data, providing generic search and management support for embedding vectors extracted from the raw unstructured data. As multiple data users can share the same database infrastructure, multi-tenancy support for vector databases is increasingly desirable. This hinges on an efficient filtered search operation, i.e., only querying the vectors accessible to a particular tenant. Multi-tenancy in vector databases is currently achieved by building either a single, shared index among all tenants, or a per-tenant index. The former optimizes for memory efficiency at the expense of search performance, while the latter does the opposite. Instead, this paper presents Curator, an in-memory vector index design tailored for multi-tenant queries that simultaneously achieves the two conflicting goals, low memory overhead and high performance for queries, vector insertion, and deletion. Curator indexes each tenant's vectors with a tenant-specific clustering tree and encodes these trees compactly as sub-trees of a shared clustering tree. Each tenant's clustering tree adapts dynamically to its unique vector distribution, while maintaining a low per-tenant memory footprint. Our evaluation, based on two widely used data sets, confirms that Curator delivers search performance on par with per-tenant indexing, while maintaining memory consumption at the same level as metadata filtering on a single, shared index.
- **Understanding**
  - **Problem:** Shared IVF forces post-filtering; per-tenant IVF duplicates memory.
  - **Technique:** **Global Clustering Tree (GCT)** with per-tenant **Tenant Clustering Trees (TCT)** as compact subtrees; Bloom filters at internal nodes; **shortlists** for small tenants.
- **Hardware:** In-memory multi-tenant
- **Partitioning / Sharding:** **Hierarchical k-means** shared tree + tenant-specific sub-trees
- **Rationale:** Tenants share coarse partition structure but retain adaptive fine-grained clustering for filtered search

---

### PipeANN: Achieving Low-Latency Graph-Based Vector Search via Aligning Best-First Search with SSD

- **Category:** 3
- **Deployment scope:** Single-node (SSD graph)
- **Local PDF:** [`pipeann.pdf`](../related-work/pdfs/pipeann.pdf)
- **Venue:** OSDI 2025
- **PDF:** [USENIX](https://www.usenix.org/system/files/osdi25-guo.pdf)
- **Abstract:** PipeANN is an on-disk graph ANNS system that aligns best-first search with SSD behavior, avoiding strict compute-I/O ordering across steps. It reaches 1.14×–2.02× the latency of in-memory Vamana and 35.0% of on-disk DiskANN latency on billion-scale datasets without sacrificing accuracy.
- **Understanding**
  - **Problem:** Disk graph search pipelines compute and I/O strictly sequentially, leaving SSD bandwidth idle.
  - **Technique:** **Async pipelined best-first search** with io_uring, speculative I/O, and dynamic beam width—single-segment graph, no distributed sharding.
- **Hardware:** Single-node NVMe SSD + DRAM
- **Partitioning / Sharding:** **No distributed shard**; optimizes single graph segment I/O
- **Rationale:** Relevant as contrast: shows graph can be fast on one machine, motivating why industry still partition-first for scale-out

---

### Starling: An I/O-Efficient Disk-Resident Graph Index Framework for High-Dimensional Vector Similarity Search on Data Segment

- **Category:** 3
- **Deployment scope:** Single-node (one data segment)
- **Local PDF:** [`starling.pdf`](../related-work/pdfs/starling.pdf)
- **Venue:** SIGMOD 2024 (PACMMOD)
- **PDF:** [arXiv](https://arxiv.org/pdf/2401.02116)
- **Abstract:** High-dimensional vector similarity search (HVSS) is gaining prominence as a powerful tool for various data science and AI applications. As vector data scales up, in-memory indexes pose a significant challenge due to the substantial increase in main memory requirements. A potential solution involves leveraging disk-based implementation, which stores and searches vector data on high-performance devices like NVMe SSDs. However, implementing HVSS for data segments proves to be intricate in vector databases where a single machine comprises multiple segments for system scalability. In this context, each segment operates with limited memory and disk space, necessitating a delicate balance between accuracy, efficiency, and space cost. Existing disk-based methods fall short as they do not holistically address all these requirements simultaneously. In this paper, we present Starling, an I/O-efficient disk-resident graph index framework that optimizes data layout and search strategy within the segment. It has two primary components: (1) a data layout incorporating an in-memory navigation graph and a reordered disk-based graph with enhanced locality, reducing the search path length and minimizing disk bandwidth wastage; and (2) a block search strategy designed to minimize costly disk I/O operations during vector query execution. Through extensive experiments, we validate the effectiveness, efficiency, and scalability of Starling. On a data segment with 2GB memory and 10GB disk capacity, Starling can accommodate up to 33 million vectors in 128 dimensions, offering HVSS with over 0.9 average precision and top-10 recall rate, and latency under 1 millisecond. The results showcase Starling's superior performance, exhibiting 43.9$\times$ higher throughput with 98% lower query latency compared to state-of-the-art methods while maintaining the same level of accuracy.
- **Understanding**
  - **Problem:** DiskANN-style graphs on fixed-size **data segments** (Milvus semantics) suffer read amplification when neighbors span disk blocks.
  - **Technique:** **Block shuffling**—reorder graph neighbors so co-accessed nodes share disk blocks; in-memory navigation graph + reordered disk graph.
- **Hardware:** Single-node SSD + memory (segment-sized deployment)
- **Partitioning / Sharding:** **Segment-level** graph; **block-level** layout optimization within segment
- **Rationale:** Vector DBs already partition data into segments; Starling optimizes graph layout within that partition unit

---

### SeRF: Segment Graph for Range-Filtering Approximate Nearest Neighbor Search

- **Category:** 3
- **Deployment scope:** Single-node
- **Local PDF:** [`serf.pdf`](../related-work/pdfs/serf.pdf)
- **Venue:** SIGMOD 2024 (PACMMOD)
- **PDF:** [Author copy](https://miaoqiao.github.io/paper/SIGMOD24_SeRF.pdf)
- **Abstract:** Range-filtering ANNS queries vectors plus ordered attributes, but performance degrades as query range width changes; building one index per range is prohibitive (Ω(n) indexes). SeRF uses a segment graph that losslessly compresses n HNSW indexes for half-bounded ranges with single-index cost, and a 2D segment graph with O(n log n) average size for general ranges. Experiments show large gains over existing methods.
- **Understanding**
  - **Problem:** Filtered ANNS with varying scalar ranges cannot afford one HNSW per range.
  - **Technique:** **Segment graph**—partition by scalar range into segments, each with a local navigable structure; compressed representation shares structure across ranges.
- **Hardware:** Single-node memory
- **Partitioning / Sharding:** **Scalar range segments**; graph inside each segment
- **Rationale:** Query only traverses segments overlapping the filter range, avoiding full-graph scan

---

### NaviX: A Native Vector Index Design for Graph DBMSs With Robust Predicate-Agnostic Search Performance

- **Category:** 3
- **Deployment scope:** Single-node (graph DBMS)
- **Local PDF:** [`navix.pdf`](../related-work/pdfs/navix.pdf)
- **Venue:** PVLDB 2025
- **PDF:** [PVLDB](https://www.vldb.org/pvldb/vol18/p4438-sehgal.pdf)
- **Abstract:** Applications need joint vector + structured/graph queries in one DBMS. NaviX is a disk-based HNSW index inside a GDBMS using prefiltering: evaluate selection subquery first, then kNN on subset S with an adaptive algorithm that uses local selectivity per HNSW node to pick heuristics. Experiments show robustness vs. pre/post-filtering baselines across selectivities and correlations.
- **Understanding**
  - **Problem:** Pre-filter vs. post-filter kNN trade off badly depending on predicate selectivity in graph DB workloads.
  - **Technique:** Disk **HNSW** with **graph/block partition** on disk; adaptive prefiltering using per-node selectivity estimates.
- **Hardware:** Single-node disk + memory (embedded graph DB)
- **Partitioning / Sharding:** **Disk graph/block partition** for page-local I/O
- **Rationale:** Co-design vector index layout with graph DB page structure for filtered hybrid queries

---

### OctopusANN: I/O Optimizations for Graph-Based Disk-Resident ANN (Design Space Exploration)

- **Category:** 3
- **Deployment scope:** Single-node (disk graph)
- **Local PDF:** [`octopusann.pdf`](../related-work/pdfs/octopusann.pdf)
- **Venue:** PVLDB 2026
- **PDF:** [PVLDB](https://www.vldb.org/pvldb/vol19/p1484-li.pdf)
- **Abstract:** SSD graph ANNS is 70–90% I/O-bound. This paper organizes memory layout, disk layout, and search-algorithm optimizations, validates a page-level model, and studies compositions. OctopusANN cuts I/O and beats Starling by 4.1–37.9% and DiskANN by 87.5–149.5% throughput at Recall@10=90%.
- **Understanding**
  - **Problem:** Disk graph ANN latency is dominated by I/O; prior work optimizes one layout dimension at a time.
  - **Technique:** Combined **memory layout + disk page layout + search algorithm** co-design (extends Starling/DiskANN line); **segment/block-level** I/O optimization.
- **Hardware:** Single-node NVMe SSD + memory
- **Partitioning / Sharding:** **Segment/block-level** disk graph layout (not cluster sharding)
- **Rationale:** Aligning graph nodes with SSD pages reduces amplification within each stored segment

---

### PageANN: Scalable Disk-Based ANN with Page-Aligned Graph

- **Category:** 3
- **Deployment scope:** Single-node (disk graph)
- **Local PDF:** [`pageann.pdf`](../related-work/pdfs/pageann.pdf)
- **Venue:** arXiv 2025
- **PDF:** [arXiv](https://arxiv.org/pdf/2509.25487.pdf)
- **Abstract:** Approximate Nearest Neighbor Search (ANNS), as the core of vector databases (VectorDBs), has become widely used in modern AI and ML systems, powering applications from information retrieval to bio-informatics. While graph-based ANNS methods achieve high query efficiency, their scalability is constrained by the available host memory. Recent disk-based ANNS approaches mitigate memory usage by offloading data to Solid-State Drives (SSDs). However, they still suffer from issues such as long I/O traversal path, misalignment with storage I/O granularity, and high in-memory indexing overhead, leading to significant I/O latency and ultimately limiting scalability for large-scale vector search.   In this paper, we propose PageANN, a disk-based approximate nearest neighbor search (ANNS) framework designed for high performance and scalability. PageANN introduces a page-node graph structure that aligns logical graph nodes with physical SSD pages, thereby shortening I/O traversal paths and reducing I/O operations. Specifically, similar vectors are clustered into page nodes, and a co-designed disk data layout leverages this structure with a merging technique to store only representative vectors and topology information, avoiding unnecessary reads. To further improve efficiency, we design a memory management strategy that combines lightweight indexing with coordinated memory-disk data allocation, maximizing host memory utilization while minimizing query latency and storage overhead. Experimental results show that PageANN significantly outperforms state-of-the-art (SOTA) disk-based ANNS methods, achieving 1.85x-10.83x higher throughput and 51.7%-91.9% lower latency across different datasets and memory budgets, while maintaining comparable high recall accuracy.
- **Understanding**
  - **Problem:** DiskANN reads amplify because one graph node spans partial pages or shares pages with unrelated nodes.
  - **Technique:** **One graph node = one SSD page** mapping; clustered layout and coordinated memory/disk allocation.
- **Hardware:** Single-node NVMe SSD
- **Partitioning / Sharding:** **Page-level alignment** (block partition semantics, not distributed shard)
- **Rationale:** Eliminates read amplification at the storage block granularity

---

### DiskANN: Fast Accurate Billion-point Nearest Neighbor Search on a Single Node

- **Category:** 3
- **Deployment scope:** Single-node (k-means for build parallelism only)
- **Local PDF:** [`diskann.pdf`](../related-work/pdfs/diskann.pdf)
- **Venue:** NeurIPS 2019
- **PDF:** [NeurIPS](https://proceedings.neurips.cc/paper/2019/file/09853c7fb1d3f8ee67a61b6bf4a7f8e6-Paper.pdf)
- **Abstract:** In-memory ANNS indices limit dataset size and cost. DiskANN indexes/stores/searches one billion points on a workstation with 64GB RAM and a commodity SSD, meeting high recall, low latency, and high density. On SIFT1B it serves >5000 QPS at <3ms mean latency and 95%+ recall@1, and introduces Vamana, a versatile graph index also strong in-memory.
- **Understanding**
  - **Problem:** Billion-point ANN cannot fit in DRAM; prior disk methods were too slow or inaccurate.
  - **Technique:** **Vamana graph** with sector-aligned SSD layout + PQ in DRAM; for **distributed build**, replica-based **k-means partitions** parallelize construction (query path is single global graph).
- **Hardware:** Single-node NVMe SSD + DRAM (~10% dataset size in memory)
- **Partitioning / Sharding:** **No query-time sharding**; k-means partitions used only for parallel index build
- **Rationale:** Single graph preserves navigability; partitioning appears only as a build-time engineering trick

---

### Quake: Adaptive Indexing for Vector Search

- **Category:** 3
- **Deployment scope:** Single-node (NUMA)
- **Local PDF:** [`sec4/quake.pdf`](../related-work/pdfs/sec4/quake.pdf)
- **§4 centroid-locality:** covered in §4 (single-node access-skew; see §4.3)
- **Venue:** OSDI 2025
- **PDF:** [USENIX](https://www.usenix.org/system/files/osdi25-mohoney.pdf)
- **Abstract:** Existing ANNS indexes struggle under dynamic, skewed workloads. Quake uses multi-level partitioning adapted via a latency cost model and recall-estimation model to set execution parameters, plus NUMA-aware intra-query parallelism. On dynamic workloads it cuts query latency 1.5–38× and update latency 4.5–126× vs. SVS, DiskANN, HNSW, and ScaNN.
- **Understanding**
  - **Problem:** Static k-means IVF partitions become imbalanced under skewed access; fixed nprobe fails as partitions change.
  - **Technique:** **Adaptive hierarchical k-means** with cost-model-driven split/merge; **Adaptive Partition Scanning (APS)** sets nprobe online from recall estimates; NUMA-aware parallel scan.
- **Hardware:** Single-node NUMA multi-core
- **Partitioning / Sharding:** **Dynamic k-means IVF** hierarchy
- **Rationale:** Partitions track workload skew; APS avoids offline nprobe tuning

---

### Ada-IVF: Incremental IVF Index Maintenance for Streaming Vector Search

- **Category:** 3
- **Deployment scope:** Single-node streaming
- **Local PDF:** [`ada-ivf.pdf`](../related-work/pdfs/ada-ivf.pdf)
- **Venue:** arXiv
- **PDF:** [arXiv](https://arxiv.org/pdf/2411.00970.pdf)
- **Abstract:** The prevalence of vector similarity search in modern machine learning applications and the continuously changing nature of data processed by these applications necessitate efficient and effective index maintenance techniques for vector search indexes. Designed primarily for static workloads, existing vector search indexes degrade in search quality and performance as the underlying data is updated unless costly index reconstruction is performed. To address this, we introduce Ada-IVF, an incremental indexing methodology for Inverted File (IVF) indexes. Ada-IVF consists of 1) an adaptive maintenance policy that decides which index partitions are problematic for performance and should be repartitioned and 2) a local re-clustering mechanism that determines how to repartition them. Compared with state-of-the-art dynamic IVF index maintenance strategies, Ada-IVF achieves an average of 2x and up to 5x higher update throughput across a range of benchmark workloads.
- **Understanding**
  - **Problem:** Streaming inserts/deletes drift IVF cluster quality; periodic full rebuild is too expensive.
  - **Technique:** **Adaptive maintenance policy** flags bad partitions + **local re-clustering** only where needed.
- **Hardware:** Single-node / streaming workloads
- **Partitioning / Sharding:** **k-means IVF** with targeted partition rebuild
- **Rationale:** Local fixes cheaper than global re-clustering; complements SPFresh/Quake line

---

### FreshDiskANN: A Fast and Accurate Graph-Based ANN Index for Streaming Similarity Search

- **Category:** 3
- **Deployment scope:** Single-node (k-means for build parallelism only)
- **Local PDF:** [`diskann.pdf`](../related-work/pdfs/diskann.pdf)
- **Venue:** arXiv
- **PDF:** [arXiv](https://arxiv.org/pdf/2105.09613.pdf)
- **Abstract:** Approximate nearest neighbor search (ANNS) is a fundamental building block in information retrieval with graph-based indices being the current state-of-the-art and widely used in the industry. Recent advances in graph-based indices have made it possible to index and search billion-point datasets with high recall and millisecond-level latency on a single commodity machine with an SSD.   However, existing graph algorithms for ANNS support only static indices that cannot reflect real-time changes to the corpus required by many key real-world scenarios (e.g. index of sentences in documents, email, or a news index). To overcome this drawback, the current industry practice for manifesting updates into such indices is to periodically re-build these indices, which can be prohibitively expensive.   In this paper, we present the first graph-based ANNS index that reflects corpus updates into the index in real-time without compromising on search performance. Using update rules for this index, we design FreshDiskANN, a system that can index over a billion points on a workstation with an SSD and limited memory, and support thousands of concurrent real-time inserts, deletes and searches per second each, while retaining $>95\%$ 5-recall@5. This represents a 5-10x reduction in the cost of maintaining freshness in indices when compared to existing methods.
- **Understanding**
  - **Problem:** DiskANN-class graphs assume static data; updates require expensive rebuilds.
  - **Technique:** Streaming update protocol for **single global disk graph** (no partition rebalance)—baseline before SPFresh’s IVF partition approach.
- **Hardware:** Single-node SSD + memory
- **Partitioning / Sharding:** **No distributed sharding**; single graph with incremental updates
- **Rationale:** Shows graph streaming is hard; motivates partition-based rebalance (SPFresh) for billion-scale updates

---

### Product Quantization for Nearest Neighbor Search (IVF-PQ)

- **Category:** 3
- **Deployment scope:** Single-node (foundational IVF-PQ)
- **Local PDF:** [`product-quantization-for-nearest-neighbor-search.pdf`](../related-work/pdfs/product-quantization-for-nearest-neighbor-search.pdf)
- **Venue:** TPAMI 2011 (Jégou et al.)
- **PDF:** [HAL / INRIA](https://inria.hal.science/inria-00514462/document)
- **Abstract:** This paper introduces product quantization for ANNS: decompose space into low-dimensional subspaces, quantize each separately, and represent vectors by short codes. Euclidean distance is estimated from codes; asymmetric distance computation improves precision. Combined with inverted files, it outperforms three SOTA methods and scales to two billion vectors.
- **Understanding**
  - **Problem:** Exact high-dimensional search is prohibitive; full-precision storage does not scale.
  - **Technique:** **k-means coarse quantizer (IVF)** partitions the space; **PQ** compresses vectors within partitions—inverted file lists per cluster.
- **Hardware:** Single-node memory
- **Partitioning / Sharding:** **k-means IVF** + PQ codes per partition
- **Rationale:** Foundational partition primitive reused by SPANN, HAKES, C-SPANN, etc.

---
## 4. Centroid / Bucket / Vector Spatial Locality for Systems Partitioning

**Local PDFs:** [`related-work/pdfs/sec4/`](../related-work/pdfs/sec4/) · [`sec4/README.md`](../related-work/pdfs/sec4/README.md)

### 4.1 What this section is

This section is a **self-contained catalog** of systems that use **geometry in embedding space** — cluster centroids, LSH bucket labels, k-means Voronoi cells, space-filling-curve order, or graph-neighborhood structure — to decide **where vectors/index state live** and/or **which nodes a query touches**.

It answers one systems question:

> *If ANN search naturally visits spatially nearby partitions (IVF nprobe, LSH multi-probe, graph hops to similar vectors), can we colocate those partitions on the same machine / nearby peer / same NUMA node to cut network hops?*

**Included (§4):**

- **Placement-time colocation** — assign buckets/clusters/vectors to nodes using spatial proximity (SABES, SPIRE, CoTra k-means, VStream curve encoding).
- **Query-time centroid routing** — route each query only to partitions whose centroids/representatives are near the query (ADBV, GaussDB-Vector, Vexless, Unleashing kRt/hRt).
- **Both** (most production-flavored designs).

**Excluded (covered elsewhere in this survey):**

| Pattern | Example | Why not §4 |
|---------|---------|------------|
| Hash / ID / block sharding | Milvus, Weaviate, serverless block paper | No embedding geometry |
| Random uniform sharding | Auncel map-reduce workers | Geometry used inside local IVF, not for shard placement |
| Filter/refine stage colocation only | HAKES RefineWorker IVF-assignment | Optimizes disaggregated pipeline, not cluster spatial sharding |
| Query budget optimizers | LEQAT | Tunes nprobe on fixed partitions; no placement |
| Pure graph-cut without routing geometry | Naive MapReduce graph shard | Partition cut ≠ centroid/bucket locality signal |

**Not category #3:** Category #3 uses partitions to **prune search** on one machine. §4 uses geometry to **place or route across machines** (or NUMA domains).

**Survey method:** All PDFs under `related-work/pdfs/` were text-scanned for spatial/locality/colocation/sharding keywords; seminal-venue candidates (SIGMOD, VLDB, OSDI, SOSP, NSDI, EDBT, SBAC-PAD, etc.) were cross-checked. Papers below passed manual abstract+method review for explicit geometry-driven placement or routing.

---

### 4.2 Master table — all §4 papers

| Paper | Venue | Nodes | Storage | What geometry is used | Why colocate / route locally? | Query-time selection | **Query-load / hot-region handling** (not partition-size balance) |
|-------|-------|-------|---------|----------------------|------------------------------|----------------------|---------------------------------------------------------------------|
| **SABES** | SBAC-PAD 2020 | Multi (≤160) | In-memory IVFADC/LSH buckets | **Coarse centroid coordinates** | Co-probed IVF/LSH buckets tend to be spatial neighbors → same node cuts inter-node traffic | Probe **w** nearest bucket centroids → only those nodes | **Not addressed** — round-robin centroid groups; eval assumes uniform query load |
| **SABBS / SABBSR** | Preprint 2024 | Multi (60) | In-memory IVFADC | Same + **bucket size**; SABBSR adds **probe frequency** | Same as SABES | Same as SABES | **SABBSR explicitly:** bucket **relevance = size × query frequency** when grouping centroids; caps per-node load. Closest §4 paper to **hot-bucket** skew |
| **Distributed LSH** | EDBT 2009 | P2P (≤1M peers) | In-memory bucket indexes | **L1 distance between LSH bucket label vectors** | Nearby bucket labels hold similar points → adjacent Chord peers → fewer hops when probing neighbors | Ring neighbor forwarding or multi-probe bucket jumps | **Label-density** balancing across peers; dynamic local-DHT split; cites **hot-range replication** in Chord for access-hot arcs — not query-trace-driven |
| **ADBV** | PVLDB 2020 | Multi (16 eval) | In-memory | **256 k-means sharding centroids** (separate from IVF nlist) | Hash sharding fans out to all nodes; nearest-centroid routing prunes partitions | Optimizer → **N partitions with closest centroids** | **Not addressed** — k-means balances **vector count**; no query-frequency rebalancing; hot queries still concentrate on same centroid-neighborhood partitions |
| **GaussDB-Vector** | PVLDB 2025 | Multi prod | Memory + disk pages | **Two-layer k-means IVF centroids** | Avoid all-DN fan-out; locality in embedding space | Route to DNs for **nearby cluster centroids** + boundary expansion | **Not discussed** — production skew handling unclear from paper |
| **CoTra** | SIGMOD 2026 / arXiv | Multi RDMA (8–16) | In-memory global graph | **Raw vector coordinates** (balanced k-means) | Graph traversal visits similar vectors → colocate to keep **~73.8%** of hops local | Query coordinator = partition with **most vectors this query will touch**; Pull-Push for remote hops | **Not addressed** — equal **vector count** per machine at build time; no query-rate-aware placement; hot query regions overload coordinator partition |
| **Vexless** | SIGMOD 2024 | Serverless functions | ~1.5 GB RAM / function | **Constrained k-means shard centroids** | Semantic shards + activate only nearby centroids vs. all-shard probe | Orchestrator activates shards within **centroid distance threshold** | **Memory cap** per function, not query QPS skew; bursty/sparse eval, not shifting hot regions |
| **SPIRE** | arXiv / VecDB 2025 | Multi (46, 8B vec) | Memory index + SSD vectors | **Hierarchical k-means** cluster centroids | Colocate neighboring clusters; hierarchical routing like IVF | Top-down **nearest centroid** descent level-by-level | **Mentions** mitigating **hot-spots under skewed workloads** via global partition IDs + boundary replication; **no query-trace rebalancer** in paper |
| **VStream** | PVLDB 2025 | Multi tiered | Memory + local + remote disk | **LSH hash → space-filling curve (Z/Hilbert)** | Neighboring vectors same partition; query hits **limited nearby partitions** only | Range filter on 1D curve encoding | **Explicit:** streaming **distribution shift** causes load imbalance → **Dynamic Partitioning Templates** rebalance partition boundaries on workload |
| **Unleashing Graph Partitioning** | PVLDB 2025 | Multi shard HNSW | In-memory | **Graph partition** + **k-means centers (kRt)** or **LSH (hRt)** for routing | Graph METIS cut preserves neighbor locality; modular routing sends query to **few near shards** | kRt: nearest k-means centers; hRt: LSH buckets | **Not addressed** — balanced **graph** partition; routing reduces shards touched but hot query regions still hit same centers |
| **RED-ANNS** | PVLDB 2026 | Multi RDMA | Disaggregated memory graph | **Locality-aware vector placement** preserving GPS graph | Keep graph edges local; affinity scheduling | **Affinity-based** query assignment to node owning query-near vectors | **Explicit:** **work-stealing** when query assignment imbalanced; eval includes **OOD** workloads; trade-off: stealing vs. locality loss |
| **HARMONY** | SIGMOD 2025 | Multi (4 eval) | In-memory | **k-means vector shards** + **dimension-based** shards | Vector shards preserve locality; dimension shards spread compute when vector shards hot | Cost model picks vector vs. dimension mode per query | **Explicit:** targets **skewed workloads** — hybrid mode when vector partitioning causes **hot shards**; 58% gain on skewed eval |
| **Quake** | OSDI 2025 | **Single-node** NUMA | In-memory hierarchical IVF | **Multi-level k-means centroids** | Co-locate partitions on **local NUMA node** to cut remote memory access | Top-down nearest-centroid scan + **APS** adaptive nprobe | **Explicit:** **dynamic skewed access patterns** (popular items get more queries) → **split/merge partitions** by cost model; closest analog to **shifting hot regions** on one machine |
| **LindormVector** | SIGMOD 2026 Industry | Multi | Memory + SSD KV | **k-means IVFPQ** lists aligned to Lindorm **shard/range** | Posting lists co-located with KV shard boundaries | Shard/range routing from Lindorm | **Not discussed** in available materials |

---

### 4.3 Research question — why colocate if hot regions exist?

**The benefit (why papers do it):**

1. **Multi-probe structure:** IVF nprobe, LSH multi-probe, and hierarchical k-means routing visit **sets of nearby partitions/buckets**. Two different situations:
   - **DES / hash all-shard probe:** every node runs local search **in parallel**, then the coordinator **merges** partial top-k. Latency is roughly **max(straggler node) + merge**, not zero — and **cluster work per query scales with P** (all nodes burn CPU/network even if wall-clock is parallel). SABES reports **14.5× vs DES @ 160 nodes** partly because touching all nodes is wasteful at scale.
   - **Subset routing (BES, SABES, ADBV):** only **some** nodes participate. **BES** spreads buckets evenly → probing **w** nearest buckets often means **up to w different nodes**. **SABES** colocates spatially neighboring buckets on the same node → the same **w** probes often hit **1–2 nodes**. Fan-out is still parallel, but you wait for **fewer** remote responses and merge **fewer** partial results. The win is **fewer nodes contacted + less merge/network**, not “sequential latency × w.”
2. **Graph traversal locality:** DiskANN/HNSW hops land on **similar vectors**. Random sharding makes most hops remote (CoTra, RED-ANNS, SPIRE).
3. **Economic argument:** Fewer nodes touched → less network bandwidth, less scatter-gather merge, better cache/NUMA behavior (Quake on single node).

**The caveat (your concern — query skew / hot spatial regions):**

Colocation is a bet that **query-induced load correlates with partition geometry**:

- Queries concentrated in a **popular embedding region** (recommender trends, viral content, seasonal catalog clusters) will hammer the **same centroids/shards** regardless of how evenly **vectors** were split.
- **Shifting hot regions over time** (streaming drift, VStream) can obsolete a static spatial layout.
- Colocation ** amplifies** this: the very nodes that should serve low-latency local probes become **tail-latency stragglers**.

This is **different from partition-size balance** (equal vector count per k-means cluster). Most papers optimize **static data balance** or **storage balance**; fewer optimize **query-rate balance across space**.

**How the literature responds (taxonomy):**

| Strategy | Papers | Mechanism | Does it fix hot query regions? |
|----------|--------|-----------|------------------------------|
| **Ignore / assume uniform queries** | SABES, ADBV, CoTra, GaussDB, Unleashing | Eval on random/i.i.d. query sets | **No** — hot regions not modeled |
| **Cap spatial colocation** | SABBS | Limit buckets/descriptors per node even if centroids want to group | **Partial** — static data cap, not query-rate |
| **Weight by query frequency** | **SABBSR** | Bucket relevance = size × **how often bucket is probed** | **Yes, explicitly** — only §4 paper with probe-frequency in placement objective |
| **Dynamic repartition on drift** | **VStream** | Dynamic Partitioning Templates when stream distribution shifts | **Partial** — data drift, not necessarily query hotspot drift |
| **Adaptive partition split/merge on access** | **Quake** | APS + split/merge when partitions skewed by **access patterns** | **Yes** — but single-node NUMA, not cluster |
| **Hybrid placement modes** | **HARMONY** | Switch to dimension-sharding when vector shards hot | **Partial** — compute rebalance, sacrifices pure spatial locality |
| **Work-stealing / elastic compute** | **RED-ANNS**, SPIRE (elastic QE) | Steal queries or scale stateless query engines | **Partial** — mitigates overload after assignment, does not reshuffle data |
| **Replication at boundaries** | SPIRE, Distributed LSH | Replicate boundary vectors / hot Chord ranges | **Partial** — helps recall + read spread, duplicates storage |
| **Route-only (no colocation of neighbors)** | ADBV, Vexless | Touch fewer nodes via centroid distance; neighbors may still scatter | **Does not solve** hot-node problem — may **reduce** blast radius (fewer nodes) but hot centroid region still hots those nodes |

**Open research questions (barely touched in §4 literature):**

1. **Query-trace-aware placement:** Only SABBSR and Quake incorporate access frequency into layout decisions. No cluster-scale system jointly optimizes **centroid colocation + query QPS caps + online reshuffle**.
2. **Temporal hot spots:** Recommender-style **moving hot regions** — only VStream (data drift) and Quake (access skew) partially address; no §4 paper evaluates **query distribution shift** with spatial colocation held fixed.
3. **Colocation vs. replication trade-off:** When a region becomes hot, should we **split** the partition (lose colocation), **replicate** it (cost storage), or **elastic compute** only (RED-ANNS/SPIRE)?
4. **Defense under colocation:** Why colocate despite skew? Papers argue **mean latency** and **network cost** dominate when probes are wide; they accept tail risk or add **secondary** balancing (SABBSR, HARMONY, work-stealing) rather than abandoning geometry.

---

### 4.4 Paper-by-paper detailed entries

#### SABES — Spatial-Aware Bucket Equal Split (Andrade, Teodoro, Ferreira; SBAC-PAD 2020)

- **PDF:** [DOI](https://doi.org/10.1109/SBAC-PAD49847.2020.00027) · **Local:** `NOT_DOWNLOADED` → [`sec4/andrade-sabes-sbac-pad-2020.pdf`](../related-work/pdfs/sec4/andrade-sabes-sbac-pad-2020.pdf)
- **Hardware:** Multi-node distributed memory; up to **160 nodes**; buckets in **RAM**.
- **Geometry signal:** After IVFADC/LSH indexing, **k-means on coarse centroids**; assign centroid groups to query-processing nodes so spatially close buckets colocate.
- **Why colocate:** BES spreads buckets evenly across nodes but ignores that **multi-probe search visits neighboring buckets** — scattering neighbors → all-node traffic. SABES keeps co-probed buckets on one node (**2.4× vs DES @ 5 nodes**, **14.5× @ 160 nodes**).
- **Query routing:** Same as BES — probe **w** nearest bucket centroids, visit only nodes holding those buckets.
- **Load / hot regions:** Round-robin group assignment; **no query-frequency model**. Descriptor-count skew across nodes possible when bucket sizes differ (motivates SABBS/SABBSR). Eval does not stress hot spatial query regions.

#### SABBS / SABBSR (Pereira, Barreiros Jr., Ferreira, Teodoro; Research Square 2024)

- **PDF:** [Research Square](https://www.researchsquare.com/article/rs-4973077/v1) · **Local:** [`sec4/pereira-sabbs-sabbsr-2024.pdf`](../related-work/pdfs/sec4/pereira-sabbs-sabbsr-2024.pdf)
- **Hardware:** Multi-node; weak scaling to **60 nodes**, **12B × 128-dim** descriptors; in-memory IVFADC.
- **Geometry signal:** Same centroid-group colocation as SABES.
- **Why colocate:** Same inter-node traffic argument as SABES.
- **SABBS:** Caps **buckets and descriptors per node** when grouping — rejects spatial assignments that overload a node (**static data/load cap**).
- **SABBSR:** Adds **bucket relevance = descriptor count × query frequency** (probe frequency) to grouping — explicitly anticipates **hot buckets** that are both large and often probed. **Up to 1.64×** vs prior best at billion scale.
- **Hot-region stance:** **Only §4 paper that names probe frequency in placement.** Still offline/batch partitioning — not online reshuffle as queries drift.

#### Distributed LSH (Haghani, Michel, Aberer; EDBT 2009)

- **PDF:** [OpenProceedings](https://openproceedings.org/2009/conf/edbt/HaghaniMA09.pdf) · **Local:** `NOT_DOWNLOADED` → [`sec4/haghani-distributed-lsh-edbt-2009.pdf`](../related-work/pdfs/sec4/haghani-distributed-lsh-edbt-2009.pdf)
- **Hardware:** P2P overlay; eval to **1M global peers**, **1000 peers/local DHT**; in-memory bucket scans.
- **Geometry signal:** **ξ mapping** (sum or Cauchy LSH) from bucket label vectors to 1D peer IDs — small **L1 label distance** ⇒ similar data ⇒ same/adjacent Chord peers.
- **Why colocate:** Multi-probe LSH jumps buckets; random peer mapping ⇒ **O(log n)** peer hops per jump. Linear ring forwarding visits **physically adjacent peers** for adjacent labels.
- **Load balance vs locality:** Optimizes **expected bucket-label density** across peers (not point counts). `hash(l)` spreads hash tables. Overloaded peers expand local DHT; **hot-range replication** (Pitoura et al.) for access-heavy Chord arcs — **replicates routing responsibility**, not full vector duplication by default.
- **Hot query regions:** **Not modeled explicitly** — balance is statistical over label distribution, not query trace.

#### AnalyticDB-V — ADBV (Wei et al.; PVLDB 2020)

- **PDF:** [PVLDB](http://www.vldb.org/pvldb/vol13/p3152-wei.pdf) · **Local:** [`sec4/analyticdb-v.pdf`](../related-work/pdfs/sec4/analyticdb-v.pdf)
- **Hardware:** Multi-node Alibaba Cloud (**16 nodes** in eval); in-memory hybrid analytics.
- **Geometry signal:** Optional **256 k-means sharding centroids** — each vector assigned to nearest centroid partition (**independent** of internal IVFPQ nlist).
- **Why route (not full colocation):** Hash/range sharding requires all-node fan-out; **centroid-based partition pruning** sends query to **N nearest partitions** (512 partitions → **3 nodes** on Deep1B without recall loss).
- **Placement-time colocation of neighbor centroids:** **No** — neighboring sharding centroids are not grouped onto same node.
- **Hot regions:** **Not addressed.** Popular query neighborhoods hit the same 3 partitions repeatedly → those nodes become hot under skewed query traffic.

#### GaussDB-Vector (Sun et al.; PVLDB 2025)

- **PDF:** [PVLDB](https://www.vldb.org/pvldb/vol18/p4951-sun.pdf) · **Local:** [`sec4/gaussdb-vector.pdf`](../related-work/pdfs/sec4/gaussdb-vector.pdf)
- **Hardware:** Multi-node production; **memory + page-based disk** persistence.
- **Geometry signal:** **Two-layer k-means IVF** centroids for sharding; vectors on DN owning nearest cluster centroid.
- **Why colocate/route:** Query routing to DNs whose centroids are near query; **boundary expansion** among selected centroids for recall at partition borders.
- **Hot regions:** Paper focuses on persistence, hybrid filter, throughput — **no query-skew or hot-spot mitigation** discussed for distance-based sharding.

#### CoTra (SIGMOD 2026 / arXiv 2025)

- **PDF:** [arXiv](https://arxiv.org/pdf/2507.06653.pdf) · **Local:** [`sec4/cotra.pdf`](../related-work/pdfs/sec4/cotra.pdf)
- **Hardware:** **8–16 machine RDMA cluster**; vectors + **holistic proximity graph** in memory.
- **Geometry signal:** **Balanced k-means on raw vectors** — one partition per machine, **no replication**.
- **Why colocate:** Global graph spans machines; random sharding makes most graph hops **remote**. k-means puts similar vectors together so **~73.8%** of accessed vectors (avg over queries) sit on one partition → fewer RDMA hops (**9.8–13.4×** throughput vs single machine @ 16 nodes).
- **Query execution:** Per query, **coordinator partition** = machine that will touch the most vectors (**not** a replica role). **Co-Search** locally; **Pull-Push** RDMA for remote vectors (~25% of accesses still remote).
- **Hot regions:** Build-time **equal vector count** only. Under query skew, **coordinator partitions for hot queries overload** — paper does not rebalance by query rate.

#### Vexless (SIGMOD 2024)

- **PDF:** [NSF PAR](https://par.nsf.gov/servlets/purl/10570270) · **Local:** [`sec4/vexless.pdf`](../related-work/pdfs/sec4/vexless.pdf)
- **Hardware:** **Azure Functions**, ~**1.5 GB** RAM each; per-shard IVF/LSH/HNSW.
- **Geometry signal:** **Constrained k-means** semantic shards; orchestrator activates shards whose **centroids within distance threshold** of query.
- **Why colocate/route:** Serverless cost ∝ functions invoked; semantic routing beats hash all-shard probe under memory limits.
- **Hot regions:** **Memory-balanced** shards, not QPS-balanced. Eval on bursty/sparse workloads — **no shifting hot query region** analysis.

#### SPIRE (arXiv 2025 / VecDB ICML 2025)

- **PDF:** [arXiv](https://arxiv.org/pdf/2512.17264.pdf) · **Local:** [`sec4/spire.pdf`](../related-work/pdfs/sec4/spire.pdf)
- **Hardware:** **46 nodes**, up to **8B vectors**; disaggregated **memory index + SSD vectors**; stateless query engine tier.
- **Geometry signal:** **Hierarchical k-means** — recursive clustering; upper levels route like IVF centroid trees; **spatial locality** reduces cross-node links vs naive graph shard.
- **Why colocate:** Partition-based hierarchy: query descends **nearest centroids** level-by-level; colocating neighboring clusters avoids remote reads during descent.
- **Boundary replication:** Vectors near k-means boundaries **replicated** to neighboring nodes before local re-clustering — recall defense, also spreads boundary query load.
- **Hot spots:** Paper mentions **global partition IDs** to **mitigate hot-spots under skewed workloads** during merge; **elastic** stateless query engines scale independently. **Does not** dynamically move vector data based on query traces.

#### VStream (Gong et al.; PVLDB 2025)

- **PDF:** [PVLDB](https://www.vldb.org/pvldb/vol18/p1593-gao.pdf) · **Local:** [`sec4/vstream.pdf`](../related-work/pdfs/sec4/vstream.pdf)
- **Hardware:** Multi-node; **four-tier** memory / local disk / remote disk; Flink streaming integration.
- **Geometry signal:** **LSH** → low-dim hash space → **space-filling curve (Z-order/Hilbert/Peano)** → 1D partition ID. Neighboring vectors in embedding space → same/nearby partitions.
- **Why colocate:** ID/hash partitioner destroys locality when streams drift; curve encoding preserves neighborhood; query scans **limited partition range** on 1D order.
- **Hot regions / drift:** **Explicit problem statement:** rapid **distribution shift** in streaming vectors causes **load imbalance** under static partitioning. **Dynamic Partitioning Templates (DPT)** continuously **adjust partition boundaries** from per-partition workload. Addresses **data drift**; query hotspot shift is related but not isolated in eval.

#### Unleashing Graph Partitioning (Gottesbüren et al.; PVLDB 2025)

- **PDF:** [PVLDB](https://www.vldb.org/pvldb/vol18/p1649-gottesbueren.pdf) · **Local:** [`sec4/unleashing-graph-partitioning-for-large-scale-near.pdf`](../related-work/pdfs/sec4/unleashing-graph-partitioning-for-large-scale-near.pdf)
- **Hardware:** Multi-node; **shard-local HNSW** per graph partition.
- **Geometry signal:** **Balanced graph partition (METIS-style)** for data placement + modular routers **kRt** (hierarchical **k-means centers**) or **hRt** (**LSH**) to pick **few shards** near query.
- **Why colocate:** Graph cut minimizes **cross-shard edges**; k-means/LSH routing sends query only to shards whose **representatives are near query** — up to **1.72× QPS @ 90% recall@10** vs prior billion-scale methods.
- **Hot regions:** Balanced **vertex count** per shard. **kRt/hRt** reduce shards touched but **hot query clusters** still map to same center-neighborhood shards — **not addressed**.

#### RED-ANNS (PVLDB 2026)

- **PDF:** [Author copy](https://kay21s.github.io/RED-ANNS-VLDB2026.pdf) · **Local:** [`sec4/red-anns.pdf`](../related-work/pdfs/sec4/red-anns.pdf)
- **Hardware:** Multi-node **RDMA** disaggregated memory; logically **full graph** (GPS strategy).
- **Geometry signal:** **Locality-aware placement** — vectors placed to preserve graph connectivity; **affinity scheduling** assigns query to node owning query-proximate vectors.
- **Why colocate:** Avoid MapReduce-style graph cuts; RDMA makes remote hops cheap but locality still wins vs random placement.
- **Hot regions / imbalance:** **Explicit** — affinity can **imbalance query assignment** across nodes. **Work-stealing** module steals queries when load imbalance detected, trading **locality for balance**. Eval includes **in-distribution and OOD** query workloads. Closest cluster-graph analog to query-load rebalancing.

#### HARMONY (SIGMOD 2025)

- **PDF:** [MIT DSpace](https://dspace.mit.edu/bitstream/handle/1721.1/164256/3749167.pdf) · **Local:** [`sec4/harmony.pdf`](../related-work/pdfs/sec4/harmony.pdf)
- **Hardware:** Multi-node (**4 nodes** in eval); in-memory distributed IVFPQ-style pipeline.
- **Geometry signal:** **k-means vector partitioning** preserves embedding locality; **dimension-based partitioning** splits work across coordinate dimensions with additive partial distance.
- **Why hybrid:** Pure **vector** shards → locality but **hot shards** under skew; pure **dimension** shards → balance but many round trips. Cost model **picks mode per query**; dimension early-stop prunes ~97% candidates.
- **Hot regions:** **Explicit skewed workload eval** — **58% throughput gain** on skewed vs leading systems. Uses **dimension mode** as relief valve when vector spatial shards overload — ** sacrifices pure colocation** to spread compute.

#### Quake (OSDI 2025)

- **PDF:** [USENIX](https://www.usenix.org/system/files/osdi25-mohoney.pdf) · **Local:** [`sec4/quake.pdf`](../related-work/pdfs/sec4/quake.pdf)
- **Scope note:** **Single-node NUMA**, not multi-node cluster — included because it is the clearest treatment of **access-pattern skew vs spatial partitions**.
- **Hardware:** Single server, **multi-level k-means IVF** in memory; **NUMA-aware** partition placement and scheduling.
- **Geometry signal:** Partitions assigned to **NUMA nodes** by affinity; search prefers **local partitions** to cut remote memory latency.
- **Why colocate:** Remote NUMA access dominates tail latency when partitions scatter randomly across sockets.
- **Hot regions:** **Central problem** — Wikipedia-like **popular pages get disproportionate queries**; inserts also skew over time. **Adaptive split/merge** of partitions using cost + recall models; **APS** adapts nprobe online. **Directly addresses shifting hot regions** — but on **one machine**, not distributed shard migration.

#### LindormVector (SIGMOD 2026 Industry)

- **PDF:** [ACM](https://dl.acm.org/doi/pdf/10.1145/3788853.3803088) · **Local:** `NOT_DOWNLOADED`
- **Hardware:** Multi-node Lindorm; compute–storage separation; SSD KV.
- **Geometry signal:** **k-means IVFPQ** posting lists stored on **Lindorm shard/range boundaries** — vector index geometry aligned with existing KV partitioning.
- **Why colocate:** Avoid separate routing layer; postings live where KV already shards.
- **Hot regions:** Not discussed in available abstract/industry summary.

---

### 4.5 Baseline vocabulary (SABES lineage)

| Baseline | Split unit | Query behavior |
|----------|-----------|----------------|
| **DES** | Vectors evenly | All nodes probed |
| **BES** | ANN buckets evenly | Nodes with **w** nearest bucket centroids |
| **SABES** | Buckets by **centroid spatial groups** | Same as BES, fewer nodes per query |
| **SABBS / SABBSR** | SABES + caps / **probe-frequency relevance** | Same as BES |

---

### 4.6 Explicitly surveyed but excluded from §4

| Paper | Reason excluded |
|-------|-----------------|
| Auncel (NSDI 2023) | **Random uniform** shard placement; geometry only inside local ELP |
| HAKES (PVLDB 2025) | Optional IVF-list/refine colocation — pipeline optimization, not cluster sharding |
| LEQAT (VLDBJ 2023) | Per-query nprobe knapsack on fixed partitions |
| DistributedANN | Graph/KV **storage layout** for one global graph — not centroid/bucket colocation for probe locality |
| Milvus / Weaviate / Qdrant | Hash sharding |
| Building Stateless Serverless Vector DBs (block partitioning) | Fixed-size blocks, no geometry |

---
## Industry Systems (Category 1 pattern)

Industry entries use official docs/blogs instead of PDFs where no paper exists.

### Pinecone (Serverless)

- **Category:** 1
- **Deployment scope:** Multi-node (compute + object storage)
- **Local PDF:** [`pinecone.pdf`](../related-work/pdfs/pinecone.pdf) (HTML snapshot)
- **Venue:** Product / blog (not a peer-reviewed paper)
- **PDF / Source:** [Serverless Architecture](https://www.pinecone.io/blog/serverless-architecture/) · [Slab Architecture](https://www.pinecone.io/learn/slab-architecture/)
- **Abstract (from docs):** Pinecone serverless separates compute from S3-backed vector storage. Indexes use geometric partitioning (IVF-like centroid hierarchy) split into immutable slabs that compact over time; namespaces provide hard multi-tenant isolation.
- **Understanding**
  - **Problem:** Static IVF cannot handle continuous writes economically; tenants need isolation without duplicate full indexes.
  - **Technique:** **Geometric partitioning** + immutable **slabs** loaded on demand; query executors fetch only relevant slabs from object storage.
- **Hardware:** Compute pool + S3 blobs + optional dedicated read nodes (cache/SSD)
- **Partitioning / Sharding:** **Centroid hierarchy / geometric partitions** + **slabs** + **namespaces**
- **Rationale:** Partitions align with object storage boundaries; slabs enable incremental compaction without full rebuild

---

### Milvus (production architecture)

- **Category:** 1
- **Deployment scope:** Multi-node
- **Local PDF:** docs only (no PDF)
- **Venue:** Product documentation
- **PDF / Source:** [Architecture](https://milvus.io/blog/deep-dive-1-milvus-architecture-overview.md) · [Partition key](https://milvus.io/docs/use-partition-key.md)
- **Abstract (from docs):** Milvus scales via shards (hash on primary key), partitions (optional partition key buckets), and segments (~512 MB sealed units), each holding an independent IVF or HNSW index merged at query time.
- **Understanding**
  - **Problem:** Need write scalability, tenant filtering, and independent index compaction.
  - **Technique:** Classic **partition-then-build-index**: shard → partition → segment → local ANN.
- **Hardware:** Distributed query/streaming nodes + object storage for sealed segments
- **Partitioning / Sharding:** **Hash shard → partition key → segment-local index**
- **Rationale:** Industry reference implementation of three-level sharding

---

### Weaviate

- **Category:** 1
- **Deployment scope:** Multi-node
- **Local PDF:** [`weaviate.pdf`](../related-work/pdfs/weaviate.pdf)
- **Venue:** Product documentation
- **PDF / Source:** [Storage](https://docs.weaviate.io/weaviate/concepts/storage) · [Cluster](https://docs.weaviate.io/weaviate/concepts/cluster)
- **Abstract (from docs):** Each shard is self-contained (objects + inverted index + one HNSW). Sharding uses UUID Murmur3 hash by default; multi-tenancy can use one shard per tenant.
- **Understanding**
  - **Problem:** HNSW cannot be cheaply merged across shards.
  - **Technique:** **One HNSW per shard**; hash-based assignment; scatter-gather top-k merge.
- **Hardware:** Distributed replicas per shard
- **Partitioning / Sharding:** **Hash shard = one full HNSW**
- **Rationale:** Large per-shard graphs beat many tiny graphs for recall and build cost

---

### Qdrant

- **Category:** 1
- **Deployment scope:** Multi-node
- **Local PDF:** [`qdrant.pdf`](../related-work/pdfs/qdrant.pdf)
- **Venue:** Product documentation
- **PDF / Source:** [Multitenancy & custom sharding](https://qdrant.tech/articles/multitenancy/)
- **Abstract (from docs):** Qdrant distributes HNSW indexes across shards; supports custom shard keys for tenant/region pinning and payload-based multitenancy.
- **Understanding**
  - **Problem:** Multi-tenant queries should not fan out to all shards.
  - **Technique:** Default hash sharding + **custom shard keys** so tenant queries hit one shard.
- **Hardware:** Distributed cluster, one HNSW per shard
- **Partitioning / Sharding:** **Hash or custom-key shard → local HNSW**
- **Rationale:** Custom keys eliminate scatter-gather for tenant-scoped queries

---

### OpenSearch k-NN

- **Category:** 1
- **Deployment scope:** Multi-node (search shards)
- **Local PDF:** [`opensearch-k-nn.pdf`](../related-work/pdfs/opensearch-k-nn.pdf)
- **Venue:** Product documentation
- **PDF / Source:** [k-NN methods](https://docs.opensearch.org/latest/mappings/supported-field-types/knn-methods-engines/)
- **Abstract (from docs):** Vectors live in Lucene segments inside standard search shards; each segment may use HNSW or IVF (Faiss, nlist buckets). Coordinator merges top-k across shards.
- **Understanding**
  - **Problem:** Reuse Elasticsearch sharding model for vector search.
  - **Technique:** **Primary/replica search shards**; segment-local HNSW or IVF.
- **Hardware:** Standard OpenSearch cluster
- **Partitioning / Sharding:** **Search shard → Lucene segment → HNSW/IVF**
- **Rationale:** Minimal new distributed logic—vectors follow existing shard model

---

### Turbopuffer · LanceDB · pgvector+Citus · Elasticsearch · AWS OpenSearch Serverless

| System | Source | Partitioning summary |
|--------|--------|---------------------|
| **Turbopuffer** | Product docs | **Centroid/cluster objects** (SPFresh-style); cold queries fetch only relevant clusters from object storage |
| **LanceDB** | Architecture materials | **IVF/IVFPQ over Lance fragments**; columnar fragments align with inverted lists |
| **pgvector + Citus** | Citus docs | **Hash/range SQL sharding**; independent pgvector index per shard |
| **Elasticsearch dense_vector** | Elastic docs | Same as OpenSearch: **shard → segment → HNSW** |
| **AWS OpenSearch Serverless** | AWS docs | Managed **HNSW (Faiss)** shards; partitioning opaque to user |

---

## Implications for Ember / Paper Writing

1. **Most production systems are category #1** (data partition → independent index per shard), not single logical graph (Milvus segments, Weaviate/Qdrant shards, Pinecone slabs).
2. **Category #2** (single logical graph/IVF) avoids scatter-gather recall loss; RED-ANNS, SHINE, DistributedANN, and BatANN preserve graph connectivity across machines.
3. **Category #3 IVF lists align with object storage** (cluster → object/file/range)—consistent with Ember cold-path narrative; **centroid spatial proximity** (§4) is underused for shard placement vs hash sharding.
4. **Pinecone, Turbopuffer, and C-SPANN** emphasize **geometric / k-means partition + selective load**, not full index materialization.

---

## Open Questions

- Long tail of ICDE partition-based papers beyond I-LSH and learned-function partitioning.
- Which arXiv entries (SPIRE, CoTra, SHINE, Ada-IVF) will appear at major venues in 2026.
- Pinecone’s exact ANN algorithm remains partially proprietary.
- **Segment** terminology differs across Milvus (logical), Starling/OctopusANN (disk block)—keep distinct in related work.

---

## Revision History

| Date | Change |
|------|--------|
| 2026-06-16 | Initial survey: partition-first, graph+sharding, industry docs |
| 2026-06-16 | Added other venues and arXiv entries (later merged into this structure) |
| 2026-06-16 | Restructured by architectural pattern (superseded) |
| 2026-06-16 | Reclassified into 3 partitioning layers + §4 centroid proximity |
| 2026-06-16 | **Local PDF inventory** in `related-work/pdfs/`; Category + Deployment scope per entry; §4 expanded; #1+#2 unified as systems partition; replaced paraphrased abstracts with paper text where available |
| 2026-06-16 | Removed Faiss 1T wiki (not a Faiss feature). SPANN → category #3, single-node paper only. DistributedANN: added "When #2 beats #1" Bing-scale table |
| 2026-06-17 | §4: split SABES vs SABBS/SABBSR with correct PDFs; removed hash/block baseline row; replaced vague colocation column with placement/routing table |
| 2026-06-17 | §4 PDFs moved to `related-work/pdfs/sec4/`; in-doc links updated |
| 2026-06-17 | §4 full rewrite: 14 papers, master table, query-skew subsection (§4.3), self-contained per-paper entries; added SPIRE, VStream, Unleashing GP, RED-ANNS, HARMONY, Quake, LindormVector |
