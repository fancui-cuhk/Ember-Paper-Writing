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

### Core Idea: cold start Query Pushdown

- Ember introduces a **custom distributed storage layer** built on commodity hardware (HDD-class media).
- Unlike S3, this layer is co-designed with the query engine. Storage nodes run ANN search directly on the data they hold.
- cold start queries are **pushed down to the storage tier** — executed entirely where the data resides, with no index data transferred to the compute tier.
- This preserves the full disaggregation architecture for hot queries:
  - **Hot queries** (index cached at compute tier): served at the compute tier as in any cloud-native system. TCO, elasticity, and multi-tenancy advantages are fully preserved.
  - **cold start queries** (index not in compute cache): pushed down and executed locally at the storage tier. Network is eliminated from the cold start query critical path. Latency is now bounded by local disk access (~10ms on HDD), not S3 round-trip (~100–200ms).
- The concept of pushing computation to storage nodes is well-established in disaggregated database systems [computation pushdown: Redshift Spectrum, S3 Select, VLDB 2025 disaggregation survey]. **However, applying this idea to vector index search on commodity HDD storage introduces a set of challenges not addressed by prior work.** Ember's primary technical contribution is solving these challenges.

### Why cold start Query Pushdown Is Non-Trivial for Vector Search

**Challenge 1 — HDD Random I/O Bottleneck.**
- HDDs deliver ~100–200 MB/s sequential throughput but only ~100–200 random IOPS (~10ms per random seek).
- A naively laid-out IVF index on HDD — where each cluster's posting list is stored at an arbitrary disk location — requires one random seek per probed cluster.
- At nprobe = 100 and 10ms per seek, disk I/O alone costs ~1 second — already violating the sub-second target before any ANN computation.
- Achieving sub-second cold start query performance on HDDs requires co-designing index layout and I/O access patterns to maximize sequential access.

**Challenge 2 — Load Imbalance and Hotspot Formation.**
- Query load is highly skewed: some tenants generate continuous traffic; most are cold archives.
- Within a single index, popular vector neighborhoods attract disproportionately more queries.
- A static data layout will concentrate I/O and compute load on a small number of storage nodes, forming hotspots that inflate cold start query tail latency under concurrent load.

**Challenge 3 — Fault Tolerance on Commodity Hardware.**
- HDDs have significantly higher annual failure rates than NVMe SSDs or cloud object storage (which provides 11-nines durability).
- A storage layer built on HDDs must provide replication and failure-recovery mechanisms to ensure data durability without sacrificing query latency.
- Replication decisions interact directly with placement and hotspot mitigation, requiring joint design.

**Challenge 4 — Distributed Index Maintenance.**
- Vector indexes must be updated incrementally as data is inserted or deleted.
- Maintaining a distributed, block-structured IVF index across storage nodes — while preserving the layout properties that enable efficient cold start queries — requires careful coordination.
- Naive strategies (e.g., full index rebuild on write) are too slow for interactive workloads; incremental update strategies must not degrade the sequential-access layout that makes HDD performance acceptable.

**Challenge 5 — Multi-Tenant Interference.**
- Multiple tenants share the storage tier. A burst of cold start queries from one tenant must not significantly degrade cold start query latency for other tenants.
- Resource isolation at the storage tier — for both disk I/O bandwidth and ANN compute — is necessary but adds design complexity.

### Ember's Solutions

**Solution 1 — Access-Aligned Block Index.**
- Ember organizes the IVF index into fixed-size blocks (10–100 MB), where block boundaries are aligned to IVF cluster boundaries: each block contains one or more complete posting lists.
- A cold start query probing k clusters maps directly to fetching the corresponding blocks — no wasted data, no read amplification.
- Each block is stored contiguously on disk, so fetching it is a single large sequential read, exploiting HDD sequential bandwidth rather than suffering its random IOPS limitation.
- This directly addresses Challenge 1.

**Solution 2 — Distributed Scatter-Gather Execution.**
- A cold start query is decomposed into sub-queries, each targeting the storage node(s) holding the relevant blocks.
- A storage-tier coordinator scatters these sub-queries to responsible nodes in parallel; each node executes local ANN search; results are gathered and merged at the coordinator.
- Distributes both I/O load and ANN compute evenly across storage nodes, preventing single-node bottlenecks under high cold start query volume.
- Directly addresses Challenge 2 and Challenge 5.

**Solution 3 — Adaptive Block Placement and Replication.**
- Ember tracks block-level access frequency across tenants and index regions.
- Frequently co-accessed blocks are placed on the same or nearby nodes (reducing cross-node fetches within a single query).
- Hot blocks — from high-QPS tenants or popular vector neighborhoods — are replicated to additional nodes for load distribution.
- Replication across failure domains provides fault tolerance for HDD reliability (Challenge 3), with replication factor tuned to access frequency.
- Directly addresses Challenge 2, Challenge 3.

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