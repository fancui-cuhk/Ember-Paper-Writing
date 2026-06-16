# Partition & Sharding in Vector Search: Survey

## Overview

This note catalogs research papers and industry systems where **partitioning or sharding** is central to vector search at scale. Entries are grouped by **architectural pattern**, not by publication venue. Each entry includes: venue, PDF link, official abstract, our reading notes (problem + technique), hardware, partitioning strategy, and rationale.

| Pattern | Representative work | Core idea |
|---------|---------------------|-----------|
| **Partition-first** | IVF, LSH, SPANN, Pinecone slabs | Partition by similarity/hash/block first; search or build a local index inside each shard |
| **Graph-first + shard-local** | Milvus/Weaviate/Qdrant HNSW-per-shard | One independent graph index per shard; scatter-gather merge |
| **Global / logical graph** | DistributedANN, RED-ANNS, BatANN | Keep one logical graph; use RDMA, graph partition, or baton-passing to avoid naive cuts |
| **Adaptive / dynamic** | Quake, Ada-IVF, SPFresh | Split, merge, or rebalance partitions as data or workload shifts |

**Related:** `summaries/distributed_vector_search_related_work.md` · **Discussion:** `discussions/2026-06-16-partition-based-vector-search-survey.md`

---

## A. Partition-First Indexes (IVF / LSH / Geometric / Block)

### SPFresh: Incremental In-Place Update for Billion-Scale Vector Search

- **Venue:** SOSP 2023
- **PDF:** [Microsoft Research](https://www.microsoft.com/en-us/research/wp-content/uploads/2023/08/SPFresh_SOSP.pdf)
- **Abstract:** ANNS on high-dimensional vectors is widely used, but supporting index updates is hard: secondary-index plus periodic global rebuild causes latency/accuracy swings and high rebuild cost. SPFresh supports in-place updates via LIRE, a lightweight incremental rebalancing protocol that splits partitions and reassigns boundary vectors to adapt to distribution shift. On a billion-scale disk index with 1% daily updates, it beats global-rebuild baselines using ~1% peak DRAM and <10% peak cores.
- **Understanding**
  - **Problem:** Billion-scale IVF/disk indexes cannot afford full rebuilds on every update; boundary-heavy partitions also hurt tail latency.
  - **Technique:** Extends SPANN-style hierarchical balanced k-means partitions with **LIRE**—lightweight split and boundary reassignment only where partitions drift—so postings stay balanced without global reconstruction.
- **Hardware:** Single-node NVMe SSD + DRAM (centroid graph in memory, postings on SSD)
- **Partitioning / Sharding:** Hierarchical **balanced k-means clustering** (SPANN lineage)
- **Rationale:** Cluster partitions bound I/O per probe; k-means groups similar vectors so queries touch few postings; balancing avoids stragglers

---

### PASE: PostgreSQL Ultra-High-Dimensional Approximate Nearest Neighbor Search Extension

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

### Milvus: A Purpose-Built Vector Data Management System

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

### HARMONY: A Scalable Distributed Vector Database for High-Throughput ANN Search

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

### Building Stateless Serverless Vector DBs via Block-based Data Partitioning

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

### LindormVector: A Distributed Vector Engine on a Cloud-Native Multi-Model NoSQL Database

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

### RAIRS: Optimizing Redundant Assignment and List Layout for IVF-Based ANN Search

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

### AnalyticDB-V (ADBV): A Hybrid Analytical Engine Towards Query Fusion for Structured and Unstructured Data

- **Venue:** PVLDB 2020
- **PDF:** [PVLDB](http://www.vldb.org/pvldb/vol13/p3152-wei.pdf)
- **Abstract:** Unstructured analytics grows rapidly but hybrid structured+unstructured queries remain disjoint in most systems. AnalyticDB-V (ADBV) lets users express hybrid queries in SQL by embedding unstructured data as vectors, uses lambda architecture plus ANNS (including VGPQ), and implements ANNS as physical operators with accuracy-aware cost-based optimization. Experiments on public and in-house data show strong performance; ADBV is deployed on Alibaba Cloud.
- **Understanding**
  - **Problem:** Hash/range sharding forces every vector query to fan out to all nodes; no similarity-based pruning at the routing layer.
  - **Technique:** **k-means clustering-based sharding** across nodes so the optimizer routes queries to the nearest N clusters only.
- **Hardware:** Distributed memory cluster (Alibaba Cloud)
- **Partitioning / Sharding:** **k-means cluster-based** cross-node sharding
- **Rationale:** Similarity routing reduces nodes touched per query vs. hash partitioning

---

### VHP: Approximate Nearest Neighbor Search via Virtual Hypersphere Partitioning

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

### GaussDB-Vector: A Large-Scale Persistent Real-Time Vector Database for LLM Applications

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

- **Venue:** PVLDB 2025
- **PDF:** [PVLDB](https://www.vldb.org/pvldb/vol18/p3049-ooi.pdf)
- **Abstract:** Graph indexes give low latency/high recall but build slowly, contend under concurrent reads/writes, and scale poorly. HAKES uses a two-stage filter (compressed vectors) + refine index with lightweight ML tuning and early termination, decouples learned-parameter management for writes, and serves it in a disaggregated distributed DB. It beats 12 index baselines and 3 distributed vector DBs, with up to 16× higher throughput.
- **Understanding**
  - **Problem:** Pure graph indexes do not scale under concurrent updates; monolithic indexes cannot disaggregate filter vs. refine resources.
  - **Technique:** **k-means IVF + PQ filter stage** on IndexWorkers; full vectors on RefineWorkers; sharding by vector ID or **by IVF assignment** to co-locate refine with probed lists.
- **Hardware:** Distributed memory (IndexWorker / RefineWorker disaggregation)
- **Partitioning / Sharding:** **k-means IVF + PQ** two-stage; optional **IVF-assignment sharding** for refine stage
- **Rationale:** IVF filter is small and replicable; co-locating refine with IVF lists cuts network traffic

---

### VStream: A Distributed Streaming Vector Search System

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

### I-LSH: I/O Efficient c-Approximate Nearest Neighbor Search in High-Dimensional Space

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

### SPANN: Highly-efficient Billion-scale Approximate Nearest Neighbor Search

- **Venue:** NeurIPS 2021
- **PDF:** [NeurIPS](https://papers.nips.cc/paper_files/paper/2021/file/299dc35e747eb77177d9cea10a802da2-Paper.pdf)
- **Abstract:** Pure in-memory ANNS is expensive at billion scale; hybrid memory–SSD ANNS is needed. SPANN keeps centroids in memory and posting lists on disk, using hierarchical balanced clustering and query-aware posting pruning to cut disk accesses while keeping recall. It is 2× faster than DiskANN at 90% recall with same memory on three billion-scale datasets.
- **Understanding**
  - **Problem:** Disk-resident graph search (DiskANN) has sequential hop chains; billion-scale needs cheaper memory footprint with parallel I/O.
  - **Technique:** **Hierarchical Balanced Clustering (HBC)** with bounded posting sizes, closure assignment (boundary duplication), query-aware dynamic nprobe; **M-way machine partition** of posting lists in Bing production (32→6.3 machines avg. per query).
- **Hardware:** Single-node or 32-machine Bing cluster
- **Partitioning / Sharding:** **HBC k-means clusters**; posting lists **sharded across machines**
- **Rationale:** IVF postings fetch in parallel with no cross-machine hop dependencies—natural fit for distributed search

---

### ScaNN: Accelerating Large-Scale Inference with Anisotropic Vector Quantization

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

### Auncel: Fast, Approximate Vector Queries on Very Large Unstructured Datasets

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

### Curator: Efficient Indexing for Multi-Tenant Vector Databases

- **Venue:** VLDB 2024
- **PDF:** [arXiv](https://arxiv.org/pdf/2401.07119)
- **Abstract:** Multi-tenant vector DBs either share one index (slow filtered search) or build per-tenant indexes (high memory). Curator indexes each tenant with a tenant-specific clustering tree encoded compactly as subtrees of a shared tree, using Bloom filters and shortlists for fast tenant-filtered search with memory near shared-index metadata filtering. Search matches per-tenant performance on YFCC100M and arXiv.
- **Understanding**
  - **Problem:** Shared IVF forces post-filtering; per-tenant IVF duplicates memory.
  - **Technique:** **Global Clustering Tree (GCT)** with per-tenant **Tenant Clustering Trees (TCT)** as compact subtrees; Bloom filters at internal nodes; **shortlists** for small tenants.
- **Hardware:** In-memory multi-tenant
- **Partitioning / Sharding:** **Hierarchical k-means** shared tree + tenant-specific sub-trees
- **Rationale:** Tenants share coarse partition structure but retain adaptive fine-grained clustering for filtered search

---

### Cost-Effective, Low Latency Vector Search with Azure Cosmos DB (Sharded DiskANN)

- **Venue:** PVLDB 2025 / arXiv
- **PDF:** [arXiv](https://arxiv.org/pdf/2505.05885.pdf)
- **Abstract:** Vector indexing is critical for semantic search and AI agents; specialized vector DBs optimize quality/cost but general databases can too. We argue scalable, high-performance, cost-efficient vector search belongs in a general-purpose cloud-native DB, and present Cosmos DB’s integrated vector indexing with sharded DiskANN and production evaluation.
- **Understanding**
  - **Problem:** One large DiskANN per partition is slow and expensive for multi-tenant filtered queries.
  - **Technique:** Default **one DiskANN per physical partition**; optional **`vectorIndexShardKey`** (e.g., tenantID) builds separate smaller DiskANN indexes per key value; filtered queries route to the matching shard index.
- **Hardware:** Cloud-native distributed DB (per replica/partition)
- **Partitioning / Sharding:** **Physical partition** + optional **vectorIndexShardKey** semantic sharding
- **Rationale:** Smaller scoped indexes improve recall, latency, and RU cost for tenant-filtered queries

---

### Distributed Similarity Search in High Dimensions using Locality Sensitive Hashing

- **Venue:** EDBT 2009 (extends WebDB 2008 work; often cited in distributed LSH lineage)
- **PDF:** [ACM](https://dl.acm.org/doi/pdf/10.1145/1516360.1516446)
- **Abstract:** We consider distributed KNN/range search in high dimensions using LSH. We map LSH buckets to a structured peer overlay with locality-preserving, load-balanced properties, enabling efficient incremental top-K KNN and range queries with fewer network hops. Real-world evaluations show major gains vs. state-of-the-art distributed similarity search.
- **Understanding**
  - **Problem:** Centralized LSH does not scale; naive bucket-to-peer mapping causes hot spots and excessive hops.
  - **Technique:** **LSH bucket → peer overlay mapping** that is locality-preserving (nearby buckets on neighboring peers) and load-balanced.
- **Hardware:** Distributed peer cluster
- **Partitioning / Sharding:** **LSH buckets** mapped to peers
- **Rationale:** Colocating nearby buckets reduces peers visited per query vs. random assignment

---

### SABES / BES / DES: Spatial-Aware Distributed Bucket Partitioning (CBMR lineage)

- **Venue:** Multimedia / CBMR literature (SABES extends BES; see e.g. PMC 2024 survey citing original SABES work)
- **PDF:** [PMC survey (references SABES)](https://pmc.ncbi.nlm.nih.gov/articles/PMC11469379/)
- **Abstract:** Distributed content-based multimedia retrieval compares **Data Equal Split (DES)**—even vector split across nodes requiring all-node probes—**Bucket Equal Split (BES)**—ANN algorithm buckets assigned to nodes—and **Spatial-Aware Bucket Equal Split (SABES)**, which colocates spatially nearby buckets on the same node so queries touch fewer nodes than BES.
- **Understanding**
  - **Problem:** DES probes every node; BES ignores spatial correlation between buckets on different nodes.
  - **Technique:** After ANN indexing produces buckets (e.g., LSH/IVF buckets), **SABES** assigns buckets to nodes to maximize spatial locality while balancing load.
- **Hardware:** Distributed memory cluster
- **Partitioning / Sharding:** **ANN buckets → nodes** (BES); **spatially aware** bucket grouping (SABES)
- **Rationale:** Queries probe nearby buckets; colocating those buckets on one node cuts inter-node traffic

---

### VEARCH / Gamma: Real-Time Visual Search on JD E-commerce Platform

- **Venue:** Middleware 2018
- **PDF:** [arXiv](https://arxiv.org/pdf/1908.07389.pdf)
- **Abstract:** We present JD’s real-time visual search system supporting hundreds of billions of product images at sub-second latency with frequent updates via distributed hierarchical architecture and efficient indexing. Gamma (Vearch engine) supports real-time indexing of vectors plus scalars using Faiss-based ANN backends.
- **Understanding**
  - **Problem:** E-commerce visual search needs distributed scale, scalar+vector hybrid queries, and real-time updates.
  - **Technique:** **Master / Router / PartitionServer** architecture; each PS hosts a **document partition** with local **IVFPQ or HNSW** (Gamma engine on Faiss); raft replication.
- **Hardware:** Distributed cluster (PartitionServers + routers)
- **Partitioning / Sharding:** **Document/table partitions** on PartitionServers
- **Rationale:** Horizontal scale via partitions; each partition is an independent ANN index with scalar filtering

---

### NetANNS: A High-Performance Distributed Search Framework Based on In-Network Computing

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

### SPIRE: Scalable Distributed Vector Search via Accuracy Preserving Index Construction

- **Venue:** arXiv (VecDB @ ICML 2025 workshop)
- **PDF:** [arXiv](https://arxiv.org/pdf/2512.17264.pdf)
- **Abstract:** Scaling ANNS to billions of vectors requires distributed indexes that balance accuracy, latency, and throughput. SPIRE identifies balanced partition granularity to avoid read-cost explosion and introduces accuracy-preserving recursive construction for a multi-level index with predictable search cost. On up to 8B vectors across 46 nodes, SPIRE achieves up to 9.64× higher throughput than state-of-the-art systems.
- **Understanding**
  - **Problem:** Naive distributed graph sharding loses recall; coarse IVF sharding explodes read cost per query.
  - **Technique:** **Hierarchical k-means** with tuned partition granularity + **accuracy-preserving recursive multi-level index** construction across nodes.
- **Hardware:** 46-node cluster; 8B vectors; distributed SSD + memory
- **Partitioning / Sharding:** **Hierarchical k-means** multi-level distributed index
- **Rationale:** Hierarchy limits probes like IVF while recursive construction preserves accuracy better than naive graph cuts

---

## B. Graph-First with Shard-Local Indexes

### PipeANN: Achieving Low-Latency Graph-Based Vector Search via Aligning Best-First Search with SSD

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

- **Venue:** SIGMOD 2024 (PACMMOD)
- **PDF:** [arXiv](https://arxiv.org/pdf/2401.02116)
- **Abstract:** Disk-resident HVSS on capacity-limited segments must balance accuracy, efficiency, and space. Starling combines an in-memory navigation graph with a locality-enhanced reordered disk graph and a block search strategy to minimize I/O. On 2GB RAM / 10GB disk it holds 33M×128D vectors with >0.9 precision, <1ms latency, and 43.9× throughput vs. SOTA at same accuracy.
- **Understanding**
  - **Problem:** DiskANN-style graphs on fixed-size **data segments** (Milvus semantics) suffer read amplification when neighbors span disk blocks.
  - **Technique:** **Block shuffling**—reorder graph neighbors so co-accessed nodes share disk blocks; in-memory navigation graph + reordered disk graph.
- **Hardware:** Single-node SSD + memory (segment-sized deployment)
- **Partitioning / Sharding:** **Segment-level** graph; **block-level** layout optimization within segment
- **Rationale:** Vector DBs already partition data into segments; Starling optimizes graph layout within that partition unit

---

### SeRF: Segment Graph for Range-Filtering Approximate Nearest Neighbor Search

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

### Unleashing Graph Partitioning for Large-Scale Nearest Neighbor Search

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

### NaviX: A Native Vector Index Design for Graph DBMSs With Robust Predicate-Agnostic Search Performance

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

- **Venue:** arXiv 2025
- **PDF:** [arXiv](https://arxiv.org/pdf/2509.25487.pdf)
- **Abstract:** Disk graph ANNS suffers long I/O paths, page misalignment, and heavy in-memory indexing overhead. PageANN aligns logical page-nodes with SSD pages, clusters similar vectors, co-designs disk layout with merging, and coordinates memory–disk allocation. It achieves 1.85×–10.83× throughput and 51.7%–91.9% lower latency vs. SOTA disk ANNS at comparable recall.
- **Understanding**
  - **Problem:** DiskANN reads amplify because one graph node spans partial pages or shares pages with unrelated nodes.
  - **Technique:** **One graph node = one SSD page** mapping; clustered layout and coordinated memory/disk allocation.
- **Hardware:** Single-node NVMe SSD
- **Partitioning / Sharding:** **Page-level alignment** (block partition semantics, not distributed shard)
- **Rationale:** Eliminates read amplification at the storage block granularity

---

## C. Global / Logical Graph Across Machines

### DiskANN: Fast Accurate Billion-point Nearest Neighbor Search on a Single Node

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

### DISTRIBUTEDANN: Efficient Scaling of a Single DiskANN Graph Across Thousands of Computers

- **Venue:** arXiv / VecDB @ ICML 2025
- **PDF:** [arXiv](https://arxiv.org/pdf/2509.06046.pdf)
- **Abstract:** DISTRIBUTEDANN searches a single 50B-vector DiskANN graph spread over 1000+ machines with 26ms median latency and 100K+ QPS. It is ~6× more efficient than partition-and-route strategies that send each query to a subset of shards.
- **Understanding**
  - **Problem:** Partition-and-route (IVF or hash shard + local graph) wastes work and loses recall vs. a true global graph at Bing scale.
  - **Technique:** **Spatial/graph-aware partition** of vectors in KV storage + **single logical DiskANN graph** searched globally; in-memory head index on hot nodes.
- **Hardware:** 1000+ nodes; distributed KV + in-memory head index
- **Partitioning / Sharding:** **Graph-aware storage partition** (not independent shard-local indexes)
- **Rationale:** Keeps one navigable graph while spreading vectors across machines

---

### CoTra: Towards Efficient and Scalable Distributed Vector Search with RDMA

- **Venue:** arXiv (SIGMOD 2026 listed on some mirrors)
- **PDF:** [arXiv](https://arxiv.org/pdf/2507.06653.pdf)
- **Abstract:** Large-scale vector search hits single-machine memory limits; distributed execution faces a computation–communication tension. CoTra scales vector search on RDMA clusters using balanced k-means partitioning plus a global proximity graph (DiskANN-style build partition) so queries concentrate on fewer machines while preserving graph quality.
- **Understanding**
  - **Problem:** Scatter-gather over many shard-local graphs adds communication; naive graph cuts hurt quality.
  - **Technique:** **Balanced k-means** assigns vectors to machines + **holistic proximity graph** spanning partitions; RDMA for remote access; query focuses on near partitions.
- **Hardware:** RDMA cluster
- **Partitioning / Sharding:** **Balanced k-means machine assignment** + global graph
- **Rationale:** k-means placement concentrates query traffic; global graph avoids per-shard recall loss

---

### RED-ANNS: An RDMA-Enabled Distributed Framework for Graph-Based ANN Search

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

- **Venue:** arXiv
- **PDF:** [arXiv](https://arxiv.org/pdf/2507.17647.pdf)
- **Abstract:** HNSW excels in memory but must be distributed beyond billions of vectors. SHINE places a logical HNSW in disaggregated memory with graph-aware partitioning, cross-compute joint caching, and query routing to the best partition, preserving graph integrity vs. naive cuts.
- **Understanding**
  - **Problem:** Disaggregated memory forces graph data remote; naive partitioning breaks HNSW navigability.
  - **Technique:** **Logical graph partition** with **cross-compute joint cache**; route query to best partition first; RDMA-backed remote graph storage.
- **Hardware:** Disaggregated memory + RDMA
- **Partitioning / Sharding:** **Graph-preserving logical partitions** + query routing
- **Rationale:** Minimize cuts that would destroy small-world navigation

---

### d-HNSW: A High-performance Vector Search Engine on Disaggregated Memory

- **Venue:** arXiv
- **PDF:** [arXiv](https://arxiv.org/pdf/2603.13591.pdf)
- **Abstract:** Vector search engines assume coupled compute/memory, but disaggregated architectures separate pools for utilization. d-HNSW targets high-performance ANNS on disaggregated memory using meta-HNSW + per-partition sub-HNSW so queries fetch only relevant sub-graphs and reduce remote bandwidth.
- **Understanding**
  - **Problem:** Remote memory bandwidth is the bottleneck for graph traversal on disaggregated systems.
  - **Technique:** **meta-HNSW** routes to **sub-HNSW per partition** (uniform sampling splits); fetch only the sub-graph needed for each query.
- **Hardware:** Disaggregated memory + RDMA
- **Partitioning / Sharding:** **meta-HNSW + sub-HNSW** hierarchical partition
- **Rationale:** Two-level structure limits remote fetches to one partition’s subgraph per routing step

---

### BatANN: Passing the Baton — High Throughput Distributed Disk-Based Vector Search

- **Venue:** arXiv
- **PDF:** [arXiv](https://arxiv.org/pdf/2512.09331.pdf)
- **Abstract:** Billion-scale disk vector search must scale beyond one server and high throughput demands. BatANN keeps a single global graph via neighborhood-aware partitioning and baton-passing of full query state across NVMe servers, avoiding scatter-gather overheads and improving throughput 2.5–6.5× vs. partitioned baselines.
- **Understanding**
  - **Problem:** Distributed disk graph search with request/response per hop doubles network round trips; scatter-gather over shard-local graphs loses global graph quality.
  - **Technique:** **Neighborhood-aware graph partition** + **baton-passing** (transfer full beam/search state to the owning server); single global Vamana graph.
- **Hardware:** Multi-server NVMe SSD + TCP
- **Partitioning / Sharding:** **Graph partition** minimizing cross-server hops (~20–25% of hops cross servers)
- **Rationale:** Preserves one graph while making cross-server continuation cheap via state handoff

---

### DSANN: Approximate Nearest Neighbor Search of Large Scale Vectors on Distributed Storage

- **Venue:** arXiv
- **PDF:** [arXiv](https://arxiv.org/pdf/2510.17326.pdf)
- **Abstract:** State-of-the-art ANNS indices on single-machine memory/disk limit scale and storage cost. DSANN stores indices on distributed storage with a cluster layer for placement and in-cluster Point Aggregation Graph to overlap I/O, addressing limitations of DiskANN hops and SPANN whole-list reads on DFS.
- **Understanding**
  - **Problem:** DiskANN sequential hops and SPANN full posting-list reads are unbearable on distributed file systems (0.1–10 ms I/O).
  - **Technique:** **Cluster layer** for distributed placement + **Point Aggregation Graph (PAG)** inside clusters; sample aggregation points for in-memory graph traversal; async overlap for residual points on DFS.
- **Hardware:** Distributed file system (e.g., Pangu) + commodity cluster
- **Partitioning / Sharding:** **Cluster partition** + graph inside each cluster
- **Rationale:** Hybrid: coarse clusters enable parallel I/O; in-cluster graph handles fine search

---

### CXL-ANNS: Software-Hardware Collaborative Memory Disaggregation for Billion-Scale ANN

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

## D. Adaptive, Dynamic, and Maintenance-Oriented Partitioning

### Quake: Adaptive Indexing for Vector Search

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

- **Venue:** arXiv
- **PDF:** [arXiv](https://arxiv.org/pdf/2411.00970.pdf)
- **Abstract:** Streaming updates degrade static IVF indexes. Ada-IVF adaptively maintains IVF by detecting problematic partitions and performing local re-clustering instead of full rebuilds, improving update throughput 2–5× vs. full rebuild baselines while preserving search quality.
- **Understanding**
  - **Problem:** Streaming inserts/deletes drift IVF cluster quality; periodic full rebuild is too expensive.
  - **Technique:** **Adaptive maintenance policy** flags bad partitions + **local re-clustering** only where needed.
- **Hardware:** Single-node / streaming workloads
- **Partitioning / Sharding:** **k-means IVF** with targeted partition rebuild
- **Rationale:** Local fixes cheaper than global re-clustering; complements SPFresh/Quake line

---

### FreshDiskANN: A Fast and Accurate Graph-Based ANN Index for Streaming Similarity Search

- **Venue:** arXiv
- **PDF:** [arXiv](https://arxiv.org/pdf/2105.09613.pdf)
- **Abstract:** Graph indices achieve high recall on billion-point SSD datasets but lack efficient streaming updates. FreshDiskANN extends graph-based ANNS to support fast insert/delete while maintaining search accuracy and millisecond-level latency on disk-resident indexes.
- **Understanding**
  - **Problem:** DiskANN-class graphs assume static data; updates require expensive rebuilds.
  - **Technique:** Streaming update protocol for **single global disk graph** (no partition rebalance)—baseline before SPFresh’s IVF partition approach.
- **Hardware:** Single-node SSD + memory
- **Partitioning / Sharding:** **No distributed sharding**; single graph with incremental updates
- **Rationale:** Shows graph streaming is hard; motivates partition-based rebalance (SPFresh) for billion-scale updates

---

## E. Foundational

### Product Quantization for Nearest Neighbor Search (IVF-PQ)

- **Venue:** TPAMI 2011 (Jégou et al.)
- **PDF:** [HAL / INRIA](https://inria.hal.science/inria-00514462/document)
- **Abstract:** This paper introduces product quantization for ANNS: decompose space into low-dimensional subspaces, quantize each separately, and represent vectors by short codes. Euclidean distance is estimated from codes; asymmetric distance computation improves precision. Combined with inverted files, it outperforms three SOTA methods and scales to two billion vectors.
- **Understanding**
  - **Problem:** Exact high-dimensional search is prohibitive; full-precision storage does not scale.
  - **Technique:** **k-means coarse quantizer (IVF)** partitions the space; **PQ** compresses vectors within partitions—inverted file lists per cluster.
- **Hardware:** Single-node memory
- **Partitioning / Sharding:** **k-means IVF** + PQ codes per partition
- **Rationale:** Foundational partition primitive reused by SPANN, HAKES, C-SPANN, Faiss sharding, etc.

---

## F. Industry Systems & Product Documentation

Industry entries use official docs/blogs instead of PDFs where no paper exists.

### Pinecone (Serverless)

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

### Faiss (distributed IVF)

- **Venue:** Open-source library / wiki
- **PDF / Source:** [Indexing 1T vectors](https://github.com/facebookresearch/faiss/wiki/Indexing-1T-vectors) · [distributed_ondisk](https://github.com/facebookresearch/faiss/blob/main/benchs/distributed_ondisk/README.md)
- **Abstract (from wiki):** Faiss shards billion-scale datasets by vertical slices, trains global IVF centroids, builds per-shard inverted lists on disk, and assigns nprobe list reads to worker nodes for merged search.
- **Understanding**
  - **Problem:** IVF index exceeds single-machine RAM/disk.
  - **Technique:** **Vertical data slice per machine** + **shared k-means centroids** + on-disk inverted lists + client-side nprobe dispatch.
- **Hardware:** Multi-node CPU + OnDiskInvertedLists
- **Partitioning / Sharding:** **Vertical slice sharding** + **IVF lists** per shard
- **Rationale:** De facto industrial pattern for distributed IVF

---

### OpenSearch k-NN

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

### CockroachDB C-SPANN

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

1. **Most production systems are partition-then-build-index**, not global graph (Milvus segments, Weaviate/Qdrant shards, Pinecone slabs).
2. **Graph sharding’s core pain** is cross-partition edge cuts and scatter-gather tail latency; RED-ANNS, SHINE, DistributedANN, and BatANN represent the “preserve logical graph” research line.
3. **IVF / partition-first aligns with object storage** (cluster → object/file/range)—consistent with Ember cold-path narrative.
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
| 2026-06-16 | **Restructured by architectural pattern** (not venue); all English; added PDF, abstract, and understanding per paper |
