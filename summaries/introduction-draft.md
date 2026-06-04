# Introduction

> **Status**: Motivation and problem statement only. Solution/Ember design omitted.

---

## 1. Background

> Key message: Vector database faces new challenges in the LLM era.
>
> 1. Vector volume is continuously growing, often billion-scale.
> 2. Workload patterns are heterogeneous: some have frequent queries, some are just cold archives.

- **In the LLM era, vector database on the cloud has become a foundational infrastructure, but also faces new challenges.**
    - Production vector volume is continuously growing, often reaching billion scale.
    - Vector search workload patterns are increasingly heterogeneous:
        - Some workloads involve frequent queries (e.g., real-time recommendation, active RAG sessions);
        - others involve infrequent or archival access (e.g., historical compliance records, long-tail user content).
        - As noted by Milvus, up to 80% of infrastructure resources can be spent on data that is rarely queried (Milvus Tiered Storage blog, Dec 2025).

---

## 2. What users expect?

> Key message: In the LLM era, cloud vector database users have new expectations. Peformance is still important, but users tend to focus on new aspects such as cost and scalability.

- **Low TCO**: Supports billion-scale workloads efficiently.
- **Low average latency**: In most cases, queries should complete with low latency -- ms or tens of ms.
- **Low tail latency**: Applications require sub-second cold start query latency to avoid user-observable stalls in interactive settings (Zilliz FAQ on RAG latency; LLM application latency optimization literature).
- **High scalability**: System can handle growing data volume and bursty query load gracefully.
- **High availability**: Fault tolerance and data durability.

---

## 3. What options do users have?

> Key mesasge: Existing cloud vector database options cannot satisfy user expectations.

- **Cloud-hosted**: Provisioned vector databases/engines hosted on cloud VMs without architectural change (OpenSearch, PostgreSQL+pgvector, Milvus 1.x).

- **Cloud-native**: Systems designed from the ground up for cloud infrastructure (Milvus 2.x, Pinecone, Turbopuffer).

- We put our system Ember here as well.

> All numbers are approximated under billion-scale workloads.

| Dimension | Cloud-Hosted (AWS Opensearch, AWS RDS PG+pgvector) | Cloud-Native (Pinecone, Milvus 2.x, Turbopuffer) | **Ember (our system)** |
|-----------|-------------------------------------|--------------------------------------------------|-------|
| **Low TCO** | ❌ compared to the other two options: 10-100X higher storage cost; 2-10X higher compute cost | ✅ | ✅ |
| **Low average latency** | ✅ ~10ms | ✅ ~15ms | ✅ ~12ms |
| **Low tail latency** | ✅ ~100ms | ❌ seconds to ~20 seconds | ✅ ~500ms |
| **High scalability** | ❌ | ✅ | ✅ |
| **High availability** | ⚠️ | ✅ | ✅ |

**Details:**

**Cloud-Hosted (AWS OpenSearch, AWS RDS PG+pgvector):**

These systems run always-on compute with indexes kept hot in RAM or local SSD. The design guarantees good and stable performance [Uber OpenSearch, Jun 2026], but at high TCO. Scaling is manual and coupled -- compute and storage must scale together. High availability requires explicit multi-AZ provisioning, increasing operational burden.

**Cloud-Native (Pinecone, Milvus 2.x, Turbopuffer):**

These systems decouple compute from storage, using on-demand compute instances, and cheap object storage (S3). Queries with cached indexes run fast. However, queries with uncached indexes must fetch from S3, inflating tail latency to seconds or ~20s. The architecture enables automatic, independent scaling of compute and storage, and provides high availability by default.

**Key insight:**

High TCO and poor scalability make cloud-hosted systems less suitable for the scale and elasticity demands of LLM-era applications. We believe the cloud-native architecture offers a more promising direction, and therefore focus our discussion on it in the following sections.

---

## 4. Cloud-Native Vector Databases

> Key message: Let readers understand what is the cloud-native architecture.

```
Components:
- Compute Layer: Stateless query nodes with SSD cache
- Storage Layer: Durable, elastic storage (object storage)
- Metadata Service: Coordinator (etcd/ZooKeeper) for index metadata, cluster state

Data Flow:
- Index segments stored in storage layer; compute nodes load segments on demand

Query path:
- Hot query (cache hit): Data in compute cache → low latency
- Cold query (cache miss): Compute fetches from storage → high latency

Key properties:
- Compute and storage scale independently
- Compute nodes are stateless and can be added/removed elastically
- Storage guarantees data durability and availability
- Multiple tenants share the compute and storage pools
```

---

## 5. The Tail Latency Problem

> Key message: Elaborate on the tail latency problem in cloud-native databases.

**The "but":**

Cloud-native databases win on TCO, scalability, and availability. BUT, they suffer from a critical failure mode that blocks its adoption for latency-sensitive applications: **Tail latency is unacceptable.**

This is directly due to **cold start queries**:

- When a query arrives, and the required index is not cached in compute memory or SSD, the system must perform a **cold start query** — fetching data from the storage layer before query execution can begin.
- Queries with cached indexes are called **hot queries**.
- *Note: Our use of **cold start query** differs from the serverless literature /cite{pinecone, milvus, turbopuffer}. Here, a query is cold start if the data is not ready, whereas in the serverless context, a query/request is cold if the compute resources are not ready.*

**Real numbers reported at billion scale:**

| System              | Cold Start Query Latency                         |
| ------------------- | ------------------------------------------------ |
| Pinecone Serverless | Up to ~20 seconds                                |
| Milvus / Zilliz     | Multi-second segment loading                     |
| Turbopuffer         | ~400–900ms at 1M scale, no billion scale results |

For RAG, real-time search, and interactive LLM applications, tail latency at this scale is a **blocking problem**.

---

## 6. Root Causes Analysis

> Key message: Analyze the root causes behind the high tail latency problem.

Cloud-native vector databases suffer from unacceptable cold start query latency due to three structural root causes.

**RC1 — Disaggregation places network data transfer on the critical path of cold start queries.**

- A cold start query cannot begin execution until the required index (or a part of it) has been loaded over the network from object storage.

**RC2 — Vector index access patterns are fundamentally mismatched with object storage.**

- Object stores (S3-class) are optimized for high aggregate throughput on large, bulk reads.
- Vector index access patterns are fine-grained, random, and query-dependent.
  - IVF probes a small subset of clusters (nprobe << nlist).
  - HNSW traverses a data-dependent sequence of graph nodes.
- This mismatch forces any cloud-native vector database to choose between two strategies, each with some limitations:

  - **Strategy A — Whole-Index Loading** (Pinecone, Milvus):
    - Load the entire index before serving any query to enjoy aggregate bandwidth.
    - Problem: **Read amplification** — only a small fraction of the index (the probed clusters / a small set of graph nodes) is needed per query, but 100% must be transferred over the network.
      - At billion scale, this means transferring gigabytes to use a few megabytes.
      - Although loading can be parallelized, this is still time-consuming — see cold start results reported by Pinecone and Milvus.
    - *Side note: both Pinecone and Milvus would partition a large dataset into several segments (e.g., 1GB) and build one index for each one. During query execution, they would do a coarse-grained IVF-style selection and then only load indexes of some segments, but these segment indexes are still loaded in their entirety.*

  - **Strategy B — On-Demand Loading** (Turbopuffer):
    - During query time, identify the relevant parts and load only those from object storage.
    - This eliminates read amplification.
      - For IVF, once we identified the clusters to probe, they can be loaded in parallel.
      - Less suitable for HNSW, since this leads to many sequential S3 loads.
        - If there are 20 hops, there would be at most 20 sequential S3 loads, each costing ~100 ms.
    - Problems:
      - **High GET latency** — even if we use IVF-style index, the latency is still dominated by S3 GET latency — averaging 100 ms, with tail latency potentially spiking to 500 ms.

**RC3 — Object storage opacity forecloses workload-aware optimization.**

- Object stores expose no interface for placement hints, co-location of frequently co-accessed clusters, access-frequency metadata, or prefetch directives.
- Yet, access patterns in vector databases are highly skewed:
  - Across tenants: some tenants generate continuous query traffic; many are cold archives rarely queried.
  - Within an index: query load concentrates on popular vector neighborhoods; most clusters are rarely touched.
- A system on top of S3 cannot express: *"this cluster is frequently accessed, replicate it on more servers for higher aggregate bandwidth."*
- This opacity structurally forecloses the class of workload-aware optimizations.

---

## 7. Ember: Design Overview

> Key message: Our core idea is to discard S3, and push the execution of cold start queries down to our customized storage layer - EmberStore. But, this does not solve the problem, there are still challenges, and we have solved them all.

In response to the above high tail latency problem, we propose Ember, a new-generation cloud-native vector database.

### Core Idea: cold start query pushdown

- Ember introduces a **custom block-based distributed storage layer** built on commodity hardware (HDD): **EmberStore**.
- cold start queries are **pushed down to EmberStore** — executed entirely where the data resides, with no network data transfer on the critical path.
  - The index is loaded to the compute layer in the background, to serve subsequent queries.
- This **to some extent** solves the above three root causes:
  - **RC1**: Network is eliminated from the critical path of cold start queries, since storage (local HDDs) is attached to compute (EmberStore storage nodes).
    - Latency is now bounded by local disk access (~10ms on HDD), not S3 round-trip (~100–200ms).
  - **RC2** and **RC3**: Now the storage layer is completely transient to us, we can design it to suit the access patterns of vector search and we are free to perform workload-aware optimizations.
- Also, this preserves the disaggregation architecture for hot queries:
  - Hot queries are still served at the compute layer as in any cloud-native system.
  - TCO, elasticity advantages are fully preserved.
- The concept of pushing computation to storage nodes is well-established in disaggregated database systems [computation pushdown: Redshift Spectrum, S3 Select, VLDB 2025 disaggregation survey].
- **However, applying this idea to vector index search on commodity HDD storage introduces a set of challenges not addressed by prior work.**

### But, cold start query pushdown is non-trivial for vector search

**Challenge 1 — HDD Random I/O Bottleneck.**
- HDDs are notorious for the poor random I/O performance.
- A naively laid-out IVF index on HDD might require one random seek per probed cluster.

**Challenge 2 — Load Imbalance and Hotspot Formation.**
- Vector search query load is highly skewed, on both the inter-index and the intra-index level.
- A static data layout will concentrate I/O and compute load on a small number of storage nodes.

**Challenge 3 — Fault Tolerance.**
- EmberStore must employ replication to ensure data durability without sacrificing query latency.
- Replication decisions interact directly with placement and hotspot mitigation, requiring joint design.

**Challenge 4 — Workload Shifts**
- Multiple tenants share the storage layer. A burst of cold start queries from one tenant must not significantly degrade cold start query latency for other tenants.

**Challenge 5 — Data Consistency**
- There are two vector engines in Ember, both might handle update queries: one in compute layer, one in storage layer.
- Maintaining data consistency is not trivial in the case.

### Ember's Solutions

**Solution 1 — Access-Aligned Block Index.**
- Ember organizes the IVF index into small fixed-size blocks (10–100 MB), where block boundaries are aligned to IVF cluster boundaries.
  - each block contains one or more clusters.
- A cold start query probing k clusters fetches the corresponding blocks.
  - little or no read amplification.
- Each block (MB scale) is stored contiguously on disk.
  - Fetching a block is a single HDD sequential read, exploiting HDD sequential bandwidth.

**Solution 2 — Distributed Scatter-Gather Execution.**
- EmberStore employs a scatter-gather approach to execution ANNS queries.
  - For a cold start query, a coordinator in EmberStore first selects the clusters to probe (the first half of IVF search).
  - Then, the coordinator divides the query into sub-queries and send them to responsible storage nodes in parallel.
  - Each storage node loads blocks and executes local ANN search in parallel.
  - Results are gathered and re-ranked at the coordinator.
- This distributes both index load and ANN compute *evenly* across storage nodes.
- Besides, big indexes are never transferred via the network, only results are.

**Solution 3 — Adaptive Block Placement and Replication.**
- EmberStore tracks block-level access frequency.
- Each block is replicated to 3 storage nodes by default.
- Hot blocks are replicated to more nodes for load distribution.

**Solution 4 — Workload-Aware Resource Scaling.**
- The workload shifts from time to time. Concretely, at different times, there are different numbers of cold start queries.
- We predict the cold start query load in a time window basis and automatically scale up/down resources in EmberStore.

---

**Cross-references:**

- `discussions/2026-06-01-user-value-dims-refinement.md`
- `discussions/2026-06-01-recall-tunability-naming.md`
- `summaries/cloud-hosted-vs-cloud-native.md` (root causes)
- `related-work/lindorm-vector-sigmod2026.md` (storage assumption difference)

---

## Open Questions

1. **S3 Vectors / OSS Vectors positioning**: Should the paper explicitly differentiate Ember from S3 Vectors and OSS Vectors in the Introduction, or should this discussion be deferred to Related Work? The decision depends on whether the Solution section naturally leads readers to compare Ember with these systems.

2. **Low tail latency target (500ms)**: The current draft states that tail latency should be "under 500ms." This number currently lacks a direct citation. One possible justification is that many LLM application clients set timeout thresholds around this range; any cold start latency exceeding client timeout would require special handling on the client side. A reliable source or more rigorous justification is needed.

3. **Cold start query latency numbers (~10s)**: The draft currently uses approximate numbers such as "~10s" for cloud-native cold start latency. These numbers are placeholders. Future versions should either (a) replace them with experimental data, or (b) clearly specify the experimental setting (dataset size, nprobe, storage media, etc.) when citing external sources.

4. **Recall tunability removal**: The author has decided to remove the "Recall tunability" dimension from the user value table. The rationale involves S3 Vectors / OSS Vectors positioning. This decision and its justification should be revisited once the Solution section is written.