# Distributed Vector Search Related Work and Commercial Systems

> **See also:** [partition-sharding-vector-search-survey.md](partition-sharding-vector-search-survey.md) — partition/sharding survey organized by architectural pattern; each entry includes venue, PDF, abstract, understanding, hardware, and partitioning strategy.
>
> **Local PDFs:** [`related-work/pdfs/`](../related-work/pdfs/) · category `distributed-anns-related` in [`categories.md`](../related-work/pdfs/categories.md) (**52** papers as of 2026-06-16).

## Scope

This note organizes representative **distributed vector search** work into two lines: **distributed graph-based** systems and **distributed IVF / partition-based** systems. For each paper, it includes a PDF link, the abstract when available from the source accessed, and a technical summary focused on **hardware**, **target problem**, and **core techniques**.[cite:53][cite:59][cite:191][cite:75][cite:269]

A second part investigates commercial systems and distinguishes among three architectural patterns: **distributed HNSW**, **distributed IVF**, and **partition-then-build-index**. The distinction matters because many production systems are not truly “global distributed indexes”; instead, they shard data first and then build an ANN index independently per shard or segment.[cite:292][cite:297][cite:278][cite:113]

## Distributed graph-based line

### SPIRE — Scalable Distributed Vector Search via Accuracy Preserving Index Construction

- **PDF / paper**: [arXiv:2512.17264](https://arxiv.org/abs/2512.17264) [cite:191]
- **Abstract**: “Scaling Approximate Nearest Neighbor Search (ANNS) to billions of vectors requires distributed indexes that balance accuracy, latency, and throughput. Yet existing index designs struggle with this tradeoff. This paper presents SPIRE, a scalable vector index based on two design decisions. First, it identifies a balanced partition granularity that avoids read-cost explosion. Second, it introduces an accuracy-preserving recursive construction that builds a multi-level index with predictable search cost and stable accuracy. In experiments with up to 8 billion vectors across 46 nodes, SPIRE achieves high scalability and up to 9.64X higher throughput than state-of-the-art systems.” [cite:191]
- **Hardware**: commodity CPU cluster; the paper summary explicitly reports experiments on up to **8 billion vectors across 46 nodes** rather than specialized accelerators.[cite:191]
- **Problem targeted**: distributed ANN systems often lose recall or explode in read cost when partition granularity is chosen poorly; SPIRE targets the **partition granularity** problem and the instability of accuracy under scale-out.[cite:191]
- **Techniques**:
  - balanced partition granularity to avoid excessive read cost;[cite:191]
  - **accuracy-preserving recursive construction** to form a **multi-level distributed index**;[cite:191]
  - predictable search cost and stable accuracy under scale-out.[cite:191]
- **Interpretation**: architecturally, SPIRE is closer to a **hierarchical partitioned distributed index** than to a pure HNSW-over-shards design. It belongs in the graph-based line only loosely; the source available emphasizes multi-level distributed indexing rather than a plain IVF or plain HNSW abstraction.[cite:191]

### RED-ANNS — An RDMA-Enabled Distributed Framework for Graph-Based ANN Search

- **Paper link**: [PVLDB / DOI page](https://dl.acm.org/doi/10.14778/3778092.3778101) [cite:112]
- **Abstract availability**: full abstract was not fetched in this session, but the indexed source identifies it as an **RDMA-enabled distributed framework for graph-based ANN search**.[cite:112]
- **Hardware**: **RDMA-capable cluster / disaggregated memory style networking** is central to the design, because the contribution is explicitly RDMA-enabled distributed graph search.[cite:112]
- **Problem targeted**: graph ANN quality degrades when the graph is naively partitioned; RED-ANNS targets the loss caused by **cross-partition graph cuts** in distributed settings.[cite:112]
- **Techniques**:
  - uses **RDMA** to expose a more global view of graph connectivity;[cite:112]
  - aims to maintain a logically fuller graph across machines rather than treating each shard as an isolated local graph.[cite:112]
- **Interpretation**: RED-ANNS represents the “distributed graph without severe partition damage” line, where fast remote memory access is used to preserve graph traversal quality.[cite:112]

### CoTra — Towards Efficient and Scalable Distributed Vector Search with RDMA

- **PDF / paper**: [arXiv:2507.06653](https://arxiv.org/abs/2507.06653) [cite:131]
- **Abstract availability**: the search result states that CoTra addresses the computation–communication tension in distributed vector search and improves throughput at fixed recall.[cite:131]
- **Hardware**: **RDMA cluster**, with optimizations explicitly designed for high-speed network communication between machines.[cite:131]
- **Problem targeted**: distributed search suffers from a computation–communication tradeoff; graph or shard routing may require too much cross-machine coordination.[cite:131]
- **Techniques**:
  - clustering-based partitioning;[cite:131]
  - asynchronous execution;[cite:131]
  - task push / communication-aware execution.[cite:131]
- **Interpretation**: CoTra is a system paper about distributed execution, not a new single-node ANN primitive. Its main contribution is to make distributed search scale while keeping communication from dominating throughput.[cite:131]

### SHINE — A Scalable HNSW Index in Disaggregated Memory

- **PDF / paper**: [arXiv:2507.17647](https://arxiv.org/abs/2507.17647) [cite:70]
- **Abstract**: the source says SHINE studies HNSW in **disaggregated memory**, where compute and memory are separated and connected by RDMA, and seeks to avoid accuracy loss from graph partitioning.[cite:70]
- **Hardware**: **disaggregated memory architecture** with **RDMA networks** connecting compute and memory nodes.[cite:70]
- **Problem targeted**: standard graph partitioning breaks HNSW’s navigability when the index must scale beyond a single node.[cite:70]
- **Techniques**:
  - graph-preserving HNSW design for disaggregated memory;[cite:70]
  - combined logical caching across compute nodes;[cite:70]
  - avoids partitioning the graph into weakly connected subgraphs.[cite:70]
- **Interpretation**: SHINE is a strong example of **distributed HNSW** in the strict sense, because the goal is to keep HNSW logically intact despite remote memory placement.[cite:70]

### d-HNSW — RDMA-Based Vector Search Engine for Disaggregated Memory

- **Paper**: [arXiv:2603.13591](https://papers.cool/arxiv/2603.13591) [cite:132]
- **Abstract availability**: indexed description says it is the first RDMA-based vector search engine for disaggregated memory and uses balanced clustering, representative indexes, RDMA-friendly graph layout, and pipelined RDMA+compute.[cite:132][cite:133]
- **Hardware**: **RDMA-based disaggregated memory** environment.[cite:132][cite:133]
- **Problem targeted**: HNSW-like graph search on disaggregated memory suffers from remote-memory latency and layout inefficiency.[cite:132][cite:133]
- **Techniques**:
  - balanced clustering;[cite:132]
  - representative index;[cite:132]
  - RDMA-friendly graph layout;[cite:132]
  - pipelined overlap of RDMA and compute.[cite:132]
- **Interpretation**: this is another “global graph over remote memory” design, with hardware/network co-design as the main contribution.[cite:132][cite:133]

### DistributedANN — Efficient Scaling of a Single DiskANN Graph Across Thousands of Computers

- **PDF / paper**: [arXiv:2509.06046](https://arxiv.org/abs/2509.06046) [cite:85]
- **Abstract summary**: DistributedANN searches over a **single 50B vector graph** spread across more than a thousand machines, using a distributed key-value store plus an in-memory ANN index, and reports 26 ms median latency and 100,000+ QPS.[cite:85]
- **Hardware**: large distributed cluster with **1000+ machines**; built on a distributed KV store and in-memory ANN layer.[cite:85]
- **Problem targeted**: conventional partition-and-route architectures for large vector search waste work and degrade efficiency; the paper asks how to scale a **single DiskANN graph** rather than many isolated shard-local graphs.[cite:85]
- **Techniques**:
  - treat distributed storage as a large shared SSD-like substrate;[cite:85]
  - distribute a single logical DiskANN graph;[cite:85]
  - replace partitioned routing with global graph search.[cite:85]
- **Interpretation**: this is a state-of-the-art industrial-style distributed graph index and one of the clearest examples of a true **global distributed graph** rather than shard-local indexes.[cite:85]

## Distributed IVF / partition-based line

### Auncel — Fast, Approximate Vector Queries on Very Large Unstructured Datasets

- **PDF / paper**: [USENIX NSDI 2023 PDF](https://www.usenix.org/system/files/nsdi23-zhang-zili.pdf) [cite:53]
- **Abstract**: “This paper presents Auncel, a vector query engine for large unstructured datasets that provides bounded query errors and bounded query latencies. The core idea of Auncel is to exploit local geometric properties of individual query vectors to build a precise error-latency profile (ELP) for each query. This profile enables Auncel to sample the right amount of data to process a given query while satisfying its error or latency requirements. Auncel is a distributed solution that can scale out with multiple workers. … Auncel only takes 25 ms to process a vector query on the DEEP1B dataset that contains one billion items with four c5.metal EC2 instances.” [cite:53]
- **Hardware**: commodity CPU-only cloud deployment on **four AWS c5.metal EC2 instances** in the billion-scale setting; the paper also discusses 128 workers mapped onto this setup.[cite:53]
- **Problem targeted**: existing IVF-based systems do not provide **bounded error** or **bounded latency**, and they use query-agnostic or black-box heuristics that choose overly large sampling sizes / probe counts.[cite:53]
- **Techniques**:
  - IVF as the base ANN structure;[cite:53]
  - **query-aware error-latency profile (ELP)** using high-dimensional geometry;[cite:53]
  - early termination for bounded error;[cite:53]
  - runtime latency-bound handling;[cite:53]
  - **distributed map-reduce execution** with probabilistic calibration so local error bounds do not amplify into excessive global error.[cite:53]
- **Interpretation**: Auncel is best viewed as a **distributed IVF execution and probe-control paper**. Its intellectual core is not new clustering; it is **per-query control of how much IVF to search**, plus safe distributed aggregation.[cite:53]

### LEQAT — Learning-based Query Optimization for Multi-Probe Approximate Nearest Neighbor Search

- **PDF / paper**: [VLDB Journal page / DOI summary](https://colab.ws/articles/10.1007%2Fs00778-022-00762-0) [cite:59]
- **Abstract**: the source states that multi-probe ANN methods often use fixed configurations, and LEQAT uses a machine-learning model to estimate **kNN distributions** and determine the partitions to probe and the number of searching neighbors in each partition, reducing latency by up to 58% and improving throughput by up to 3.9×.[cite:59]
- **Hardware**: commodity hardware; the available source emphasizes applicability across **disk-based, GPU-based, and distributed** scenarios rather than a specialized hardware platform.[cite:59]
- **Problem targeted**: fixed multi-probe configurations are suboptimal, especially under clustering-based partitioning. LEQAT asks how to pick **which partitions to probe** and **how much to search within each** on a per-query basis.[cite:59]
- **Techniques**:
  - formulate per-query optimization as a **0–1 knapsack** problem;[cite:59]
  - learn the query’s **kNN distribution** over partitions;[cite:59]
  - apply to IVF, HNSW, and SSG under clustering-based partitioning.[cite:59]
- **Interpretation**: LEQAT is a canonical **distributed IVF / partition-based optimizer**. Conceptually it solves a generalization of the **nprobe selection** problem.[cite:59]

### SPANN — Highly-Efficient Billion-Scale Approximate Nearest Neighbor Search

- **PDF / paper**: [arXiv:2111.08566](https://arxiv.org/abs/2111.08566) [cite:44]
- **Abstract availability**: not fetched in full in this session, but prior indexed material identifies SPANN as centroids in memory with posting lists on SSD and a foundational disk-based IVF-like system.[cite:44][cite:126]
- **Hardware**: **memory + SSD** architecture; centroids are memory-resident while posting lists live on SSD.[cite:44][cite:126]
- **Problem targeted**: billion-scale search with high recall under limited DRAM; in-memory graph indexes are too memory-expensive, while naive disk ANN has poor efficiency.[cite:44][cite:126]
- **Techniques**:
  - hierarchical balanced clustering;[cite:44][cite:126]
  - in-memory centroid / head index;[cite:44][cite:126]
  - posting lists stored on SSD;[cite:44][cite:126]
  - query-aware pruning.[cite:44][cite:126]
- **Interpretation**: SPANN is the most influential **IVF-on-disk** line. Although originally not framed as a distributed paper, its cluster/posting-list structure extends naturally to distributed storage and inspired later distributed variants.[cite:44][cite:126]

### SPFresh — Incremental In-Place Update for Billion-Scale Vector Search

- **PDF / paper**: [SOSP 2023 PDF](https://www.microsoft.com/en-us/research/wp-content/uploads/2023/08/SPFresh_SOSP.pdf) [cite:73]
- **Abstract availability**: full text not fetched here, but the indexed source identifies SPFresh as an update-oriented extension of SPANN.[cite:73][cite:130]
- **Hardware**: inherits SPANN’s **memory + SSD / posting-list** architecture.[cite:73][cite:130]
- **Problem targeted**: SPANN-style indexes are efficient for static data but hard to update incrementally; SPFresh targets **in-place updates without full rebuild**.[cite:73][cite:130]
- **Techniques**:
  - LIRE protocol for in-place update;[cite:73][cite:130]
  - split oversized clusters;[cite:73][cite:130]
  - merge undersized clusters.[cite:73][cite:130]
- **Interpretation**: SPFresh matters for distributed IVF-like systems because cluster maintenance and posting-list mutation are core operational challenges once a production system must support fresh writes.[cite:73][cite:130]

### C-SPANN / CockroachDB distributed vector indexing

- **Paper / blog**: [Cockroach Labs blog](https://www.cockroachlabs.com/blog/distributed-vector-indexing-cockroachdb/) [cite:32]
- **Abstract availability**: not a paper abstract; the source explains how CockroachDB adapts SPANN to a distributed SQL engine.[cite:32][cite:35]
- **Hardware**: commodity distributed SQL cluster; storage and query execution are inherited from CockroachDB’s distributed architecture.[cite:32][cite:35]
- **Problem targeted**: adapt SPANN-style vector indexing to a database that already shards and replicates relational data.[cite:32][cite:35]
- **Techniques**:
  - SPANN / SPFresh-inspired distributed vector indexing;[cite:32][cite:35]
  - cluster/posting-list design integrated with distributed KV ranges.[cite:32][cite:35]
- **Interpretation**: a practical example of using **partition-first distributed DB infrastructure** and layering an IVF-like vector index on top.[cite:32][cite:35]

### DSANN — Approximate Nearest Neighbor Search of Large Scale Vectors on Distributed Storage

- **PDF / paper**: [arXiv:2510.17326 PDF](https://arxiv.org/pdf/2510.17326) [cite:269]
- **Abstract**: “In this paper, we present a new graph-cluster hybrid indexing and search system which supports Distributed Storage Approximate Nearest Neighbor Search, called DSANN. DSANN can efficiently index, store, search billion-scale vector database in distributed storage and guarantee the high availability of index service. DSANN employs the concurrent index construction method to significantly reduces the complexity of index building. Then, DSANN applies Point Aggregation Graph to leverage the structural information of graph to aggregate similar vectors, optimizing storage efficiency and improving query throughput via asynchronous I/O in distributed storage.” [cite:269]
- **Hardware**: **distributed storage** environment; the abstract emphasizes storage efficiency, asynchronous I/O, and high availability rather than GPU/RDMA hardware.[cite:269]
- **Problem targeted**: no efficient and effective ANN indexing algorithm for **distributed storage scenarios** with large vector sets, where single-machine in-memory or local-disk indexes are too costly or fragile.[cite:269]
- **Techniques**:
  - **cluster layer on top** for distributed placement;[cite:269]
  - **Point Aggregation Graph inside clusters**;[cite:269]
  - concurrent index construction;[cite:269]
  - asynchronous I/O for distributed storage.[cite:269]
- **Interpretation**: DSANN is a **cluster-on-top, graph-inside** hybrid. It sits between distributed IVF and distributed graph search.[cite:269]

### NetANNS — A High-Performance Distributed Search Framework Based on In-Network Computing

- **Paper**: [IEEE page](https://ieeexplore.ieee.org/document/9644813/) [cite:75]
- **Abstract availability**: full abstract was not fetched here, but the indexed source identifies it as an **in-network computing** framework for distributed ANN search.[cite:75]
- **Hardware**: **programmable switch / in-network computing + commodity servers**.[cite:75]
- **Problem targeted**: distributed ANNS frameworks incur preprocessing and data-movement overhead; NetANNS targets these **system-level** inefficiencies.[cite:75]
- **Techniques**:
  - in-network computing for preprocessing/routing;[cite:75]
  - pluggable support for multiple ANN methods;[cite:75]
  - framework-level acceleration rather than a new ANN algorithm.[cite:75]
- **Interpretation**: NetANNS is best classified as a **system optimization layer** rather than a new IVF or graph index. It is relevant to the partition-based line because it assumes distributed partitioned search and optimizes its execution path.[cite:75]

## Commercial systems: distributed indexing taxonomy

Commercial systems use three recurring patterns.

1. **Distributed HNSW**: each shard contains an HNSW-like graph, and the cluster distributes/shards those graphs across nodes.[cite:292][cite:297]
2. **Distributed IVF**: the system uses IVF or IVFPQ as the ANN primitive, distributing lists / shards / trained indexes across nodes.[cite:278][cite:116][cite:109]
3. **Partition-then-build-index**: the system first partitions data into shards or segments, then builds an ANN index independently inside each shard/segment, and merges top-k results globally.[cite:279][cite:282][cite:292][cite:297]

These are not mutually exclusive. In practice, many systems support both HNSW and IVF, but their **distribution mechanism** is often still partition-first.[cite:116][cite:119][cite:113]

### Milvus

| Aspect | Findings |
|---|---|
| Distribution model | Milvus stores data in **segments**; each segment has an **independent index**, and Query Nodes load sealed segments from object storage as needed and perform segment-level search.[cite:113][cite:279] |
| IVF vs HNSW | Milvus supports IVF-family and HNSW-family indexes, but at the distributed layer the pattern is segment-first: search each segment, get local top-k, then merge.[cite:279][cite:282][cite:288] |
| Classification | Primarily **partition-then-build-index**; can host both **distributed IVF** and **distributed HNSW** only in the loose sense that each segment has one of these indexes.[cite:279][cite:282][cite:113] |
| Notes | This is not a single global HNSW or single global IVF across the cluster; it is many segment-local indexes searched and merged.[cite:279][cite:282] |

### Qdrant

| Aspect | Findings |
|---|---|
| Distribution model | Qdrant uses **shards** distributed across nodes with replication and Raft-based metadata management; segments are the physical units inside shards.[cite:280][cite:287] |
| IVF vs HNSW | Public architecture materials emphasize **HNSW** as the main ANN index, with one vector index per shard/segment and distributed query processing across shards.[cite:280][cite:283][cite:287] |
| Classification | Mostly **distributed HNSW** implemented as **partition-then-build-index**.[cite:280][cite:287] |
| Notes | Qdrant is not described as a single global cross-node HNSW; it is shard-local HNSW plus distributed routing and merge.[cite:280][cite:287] |

### Weaviate

| Aspect | Findings |
|---|---|
| Distribution model | A class/index is composed of one or more **shards**; each shard is self-contained and contains object store, inverted index, and vector index, and can be placed on different nodes.[cite:292][cite:297] |
| IVF vs HNSW | Weaviate’s main vector index is a custom **HNSW** implementation; the docs explicitly say each shard has its own vector index and that HNSW avoids segmentation inside the shard for efficiency.[cite:292][cite:295] |
| Classification | **Distributed HNSW** and **partition-then-build-index**.[cite:292][cite:297] |
| Notes | This is a clean example of a shard-local self-contained HNSW design rather than a cross-shard global graph.[cite:292][cite:297] |

### OpenSearch / Amazon OpenSearch Service

| Aspect | Findings |
|---|---|
| Distribution model | OpenSearch distributes data using standard **search shards** and allows vector fields to choose **HNSW** or **IVF** methods.[cite:119][cite:116] |
| IVF vs HNSW | Both are supported in OpenSearch generally; AWS guidance even recommends different shard counts for HNSW and IVFPQ at billion scale.[cite:116][cite:109] |
| Classification | Supports both **distributed HNSW** and **distributed IVF**, but operationally still follows **partition-then-build-index** on shards.[cite:119][cite:109][cite:116] |
| Notes | OpenSearch Serverless vector collections support only **HNSW**, not IVF.[cite:294][cite:301] |

### Pinecone

| Aspect | Findings |
|---|---|
| Distribution model | Pinecone partitions vectors across **pods** and executes queries in parallel across those pods; storage and compute are separated in the serverless design, with blob storage as source of truth.[cite:291][cite:293] |
| IVF vs HNSW | Public materials describe proprietary ANN structures and a geometrically partitioned index builder; secondary descriptions characterize per-pod ANN as graph-like and similar to HNSW, but Pinecone does not publicly document the exact algorithm in enough detail to label it definitively.[cite:296][cite:291] |
| Classification | Clearly **partition-then-build-index**; exact classification as distributed HNSW vs IVF is **not fully public**.[cite:296][cite:293] |
| Notes | The serverless architecture highlights a geometrically partitioned index builder, suggesting partition-aware indexing rather than a single monolithic global structure.[cite:296] |

### LanceDB

| Aspect | Findings |
|---|---|
| Distribution model | OSS LanceDB is embedded and storage-centric; enterprise materials describe a query fleet, plan execution fleet, and indexer fleet over object storage, with distributed query plans and NVMe-backed cache.[cite:204][cite:213] |
| IVF vs HNSW | Public materials emphasize **IVF / IVFPQ** integrated with the Lance format.[cite:202][cite:215][cite:204] |
| Classification | Best understood as **distributed IVF** over object storage, though its OSS edition is not a full clustered DB product.[cite:204][cite:202][cite:213] |
| Notes | LanceDB is distinctive because the storage substrate is a **columnar object-store-backed format**, not because it invents a radically different ANN primitive.[cite:204][cite:213] |

## Practical distinctions

The most important architectural distinction is whether the system maintains a **single logical global ANN structure** across machines or whether it **partitions first and builds local indexes**. Most commercial systems today fall into the second category, even when they advertise distributed HNSW or distributed IVF.[cite:279][cite:282][cite:292][cite:297]

- **True global distributed graph** examples are rare and mostly research/industrial papers such as DistributedANN, SHINE, RED-ANNS, and d-HNSW.[cite:85][cite:70][cite:112][cite:132]
- **Distributed IVF / list-based designs** are more natural for commodity clusters and storage disaggregation because clusters/lists map cleanly to shards, files, or object-store fragments.[cite:53][cite:278][cite:204]
- **Partition-then-build-index** dominates products because it composes well with replication, tenant isolation, background compaction, and standard distributed database operations.[cite:113][cite:292][cite:297][cite:293]

For system design discussions, “distributed HNSW” should therefore be used carefully. In many products, it means **HNSW-per-shard plus distributed top-k merge**, not a globally navigable HNSW spanning machines.[cite:292][cite:297][cite:280][cite:287]
