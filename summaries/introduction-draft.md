# Introduction for "Ember: Low Tail Latency Cloud-Native Vector Search"

---

## 1. Background

> Key message: Vector database faces scale challenges in the LLM era.
>

- In the LLM era, vector database on the cloud has become a foundational infrastructure, but also faces new challenges.
  - Production vector volume is continuously growing, often reaching billion scale.

---

## 2. What Users Expect?

> Key message: In the LLM era, cloud vector database users have new expectations. Peformance is still important, but users tend to focus on new aspects such as cost and scalability.

- **Low TCO**: Supports billion-scale workloads efficiently.
- **Low average latency**: In most cases, queries should complete with low latency -- ms or tens of ms.
- **Low tail latency**: Applications require low tail latency to avoid user-observable stalls in interactive settings /cite{Zilliz FAQ on RAG latency; LLM application latency optimization literature}.
- **High scalability**: System can handle growing data volume and bursty query load efficiently.
- **High availability**: Fault tolerance and data durability.

Other user expectations such as high recall and data consistency are baseline requirements and are satisfied by all systems.

---

## 3. What Options Do Users Have?

> Key mesasge: Existing cloud vector database options cannot satisfy user expectations.

- **Cloud-hosted**: Provisioned vector databases/engines hosted on cloud VMs without architectural change (OpenSearch, PostgreSQL+pgvector, Milvus 1.x).

- **Cloud-native**: Systems designed from the ground up for cloud infrastructure (Milvus 2.x, Pinecone, Turbopuffer).

- We put our system Ember here just for reference.

> Numbers are estimated under billion-scale workloads.
>
> Cost unit: USD per GB-month.

|  | Cloud-Hosted | Cloud-Native Baslines | **Ember (our system)** |
|-----------|-------------------------------------|--------------------------------------------------|-------|
| **Examples** | AWS Opensearch, AWS RDS PG+pgvector | Pinecone, Milvus 2.x, Turbopuffer |  |
| **Low TCO** | ❌ ~10X higher cost for provisioned compute; EC2 RAM: ~5 USD; EBS SSD: ~0.10 USD | ✅ S3: ~0.023 USD | ✅ EC2 instance local HDD: ~0.007 USD |
| **Low average latency** | ✅ ~10ms | ✅ ~15ms | ✅ ~13ms |
| **Low tail latency** | ✅ ~100ms | ❌ seconds to ~20 seconds | ✅ ~200 ms |
| **High scalability** | ❌ | ✅ | ✅ |
| **High availability** | ✅️ | ✅ | ✅ |

**Cloud-Hosted:** These systems run always-on compute with indexes kept hot in RAM or local SSD. The design guarantees good and stable performance, but at high TCO. Scaling is manual and coupled -- compute and storage must scale together. High availability requires explicit multi-AZ provisioning.

**Cloud-Native baselines:** Details elaborated below.

**Key insight:** High TCO and poor scalability make cloud-hosted systems less suitable for the scale and elasticity demands of LLM-era applications. **We believe the cloud-native architecture represents a more promising direction, and therefore focus our discussion on it in the following.**

---

## 4. Cloud-Native Vector Databases

> Key message: Let readers understand what is the cloud-native architecture.

![Cloud-Native Vector Database Architecture](../figures/architecture-diagram.pdf)

- Cloud-native systems systems decouple compute from storage, using on-demand compute instances, and cheap object storage (S3).
- Queries with cached indexes (**hot queries**) run fast.
- Queries with uncached indexes (**cold start queries**) must fetch index from S3, inflating tail latency to seconds or ~20s /cite{pinecone}.
- The architecture enables automatic, independent scaling of compute and storage, and provides high availability by default.
- Some systems employ the serverless mode: some compute nodes are shut down after a while of silence (TTL-based).

---

## 5. The Tail Latency Problem

> Key message: Elaborate on the tail latency problem in cloud-native databases.

Cloud-native databases win on TCO, scalability, and availability. BUT, they suffer from a critical failure mode that blocks its adoption for latency-sensitive applications: **Tail latency is unacceptable.**

Although the high tail latency problem is well-acknowledged, none existing cloud-native databases aim to solve it.

**Real numbers reported at billion scale:**

| System              | Cold Start Query Latency                       |
| ------------------- | ---------------------------------------------- |
| Pinecone Serverless | Up to ~20 seconds                              |
| Milvus 2.6+         | Multi-second segment loading                   |
| Turbopuffer         | *~900ms at 1M scale, no billion scale results* |

For RAG, real-time search, and interactive LLM applications, tail latency at this scale is a **blocking problem**.

---

## 6. Root Causes Analysis

> Key messages:
>
> 1. High tail latency was due to cold start queries.
> 2. Analyze root causes behind slow cold queries.

This is directly due to **cold start queries**:

- When a query arrives, and the required index is not cached in compute memory or SSD, the system must perform a **cold start query** — fetching data from the storage layer before query execution can begin.
- *Note: Our use of **cold start query** differs from the serverless literature /cite{pinecone, milvus, turbopuffer}. Here, a query is cold start if the data is not cached, whereas in the serverless context, a query/request is cold if the compute resources are not ready.*

#### Cold start queries are slow due to three structural root causes:

**RC1 — Disaggregation places network data transfer on the critical path of cold start queries.**

- A cold start query cannot begin execution until the required index (or a part of it) has been loaded over the network from object storage.

**RC2 — Vector index access patterns are fundamentally mismatched with object storage.**

- Object stores (S3-class) are optimized for high aggregate throughput on large, bulk reads.
- Vector index access patterns are fine-grained, random, and query-dependent.
  - IVF probes a small subset of clusters (nprobe << nlist).
  - HNSW traverses a data-dependent sequence of graph nodes.
- This mismatch forces any cloud-native vector database to choose between two strategies, each with limitations:

  - **Strategy A — Whole-Index Loading** (Pinecone, Milvus):
    - Load the entire index before serving any query to enjoy aggregate bandwidth.
    - Problem: **Read amplification** — only a small fraction of the index (the probed clusters / a small set of graph nodes) is needed per query, but 100% must be transferred over the network.
      - At billion scale, this means transferring gigabytes to use a few megabytes.
    - *Note: Pinecone and Milvus would partition a large dataset into several slabs/segments (e.g., 1GB each) and build one index for each. During query execution, they would do a coarse-grained IVF-style selection and then load indexes of selected slabs/segments, but these indexes are still loaded in their entirety while only a small portion is actually used.*
  - **Strategy B — On-Demand Loading** (Turbopuffer):
    - During query time, load relevant parts of the index from object storage on demand — the OS page loading approach.
    - This eliminates read amplification - only relevant parts are loaded.
      - Suitable for IVF, once we identified the clusters to probe, they can be loaded in parallel.
      - Not suitable for HNSW, since this leads to excessive sequential S3 loads.
        - If there are 100 hops, there would be at most 100 sequential S3 loads, each costing ~100 ms, constituting a more than 10 seconds latency.
    - Problem: **Poor random I/O performance of object storage** — small, random S3 loads cannot enjoy the high aggregate bandwidth of S3; load latency is dominated by network round trips.

**RC3 — Object storage opacity forecloses workload-aware optimization.**

- Object stores expose no interface for placement hints, co-location of frequently co-accessed clusters, or access-frequency metadata.
- Yet, workload patterns in vector databases are highly skewed and can enjoy workload-aware optimizations:
  - Inter-index: some indexes have continuous query traffic; some are cold archives rarely queried.
  - Intra-index: query load concentrates on popular vector neighborhoods; most vectors are rarely touched.
- A system on top of S3 cannot express: *"These two clusters are always probed together, they should be stored contiguously on disk"* or *"this cluster is frequently accessed, replicate it on more servers for higher aggregate bandwidth."*

These root causes do not invalidate the cloud-native architecture — they precisely identify where it falls short. Ember addresses this gap by replacing the storage abstraction while fully preserving disaggregation for hot queries.

---

## 7. Ember: Design Overview

> Key messages:
>
> 1. Clearly position Ember.
> 2. Cold start query pushdown directly addresses RC1 and enables solutions to RC2 and RC3, but introduces new challenges.
> 3. We solve all those challenges.

- In response to the above high tail latency problem, we propose Ember, a new-generation cloud-native vector database that features **low TCO, high scalability, low average latency and low tail latency**. 
- Ember is cloud-native in the sense that compute remains fully stateless, elastic, and disaggregated.

<img src="../figures/map.pdf" alt="Positioning Ember in the cloud vector database design space" style="zoom:200%;" />

### Core Idea: cold start query pushdown

- Ember introduces a **custom block-based distributed storage layer** built on commodity hardware (HDD): **EmberStore**.
  - EmberStore is the storage layer — it replaces S3 but serves the same architectural role (durable, elastic storage), with the addition of compute co-location for cold-start query pushdown.

- Cold start queries are **pushed down to EmberStore** — executed entirely where the data resides, with no network data transfer on the critical path.
  - The index is loaded to the compute layer in the background, to serve subsequent queries.
- This directly addresses RC1 and sets the stage for solutions to RC2 and RC3:
  - **RC1**: Network is eliminated from the critical path of cold start queries, since storage is physically attached to compute (both on EmberStore storage nodes) for cold start queries.
  - **RC2** and **RC3**: Now the storage layer is completely transparent to us, we can design it to suit the access patterns of vector search and we are free to perform workload-aware optimizations.
- Also, this preserves the disaggregation architecture for hot queries:
  - Hot queries are still served in the disaggregated architecture as in any cloud-native system.
  - TCO, elasticity advantages are fully preserved.
  - *Note: Cold start queries should only account for the minority of the system. /cite{}*

- The concept of pushing computation to the storage layer is well-established in disaggregated database systems /cite{computation pushdown: Redshift Spectrum, S3 Select, VLDB 2025 disaggregation survey}.
- **However, applying this idea to vector search on commodity storage introduces a set of challenges not addressed by prior work.**

### Cold-Start Query Pushdown on Commodity Hardware: Challenges

**Challenge 1 — HDD Random I/O Bottleneck.**

- HDDs are notorious for the poor random I/O performance.
- A poorly laid-out IVF index on HDD would require one random seek per probed cluster.
- At nprobe = 100 and ~10ms per random seek, disk I/O alone costs ~1s — already violating the sub-second target before any ANN computation.

**Challenge 2 — Hotspots and Workload Imbalance.**

- Vector search query load is highly skewed, on both the inter-index and the intra-index level (static spatial imbalance: hot blocks concentrated on a few nodes).
- A static data layout will concentrate I/O and compute load on a small number of storage nodes.

**Challenge 3 — Fault Tolerance.**
- EmberStore must employ replication to ensure data durability without sacrificing query latency.
- Replication decisions interact directly with placement and hotspot mitigation, requiring joint design.

**Challenge 4 — Workload Shifts**
- Multiple tenants share the storage layer. A burst of cold start queries from one tenant must not significantly degrade cold start query latency for other tenants (dynamic temporal imbalance: burst of cold-start queries arriving simultaneously).

**Challenge 5 — Data Consistency**
- There are two vector engines in Ember, both might handle update queries: one in compute layer, one in storage layer.
- Cold-start query pushdown introduces a dual-write problem — updates may be applied to the compute-layer index and the EmberStore index at different times, creating a window of inconsistency. Reads during this window may return stale results.

### Ember's Solutions to Above Challenges

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

- EmberStore tracks block-level access frequency。
- Each block is replicated to 3 storage nodes by default.
- Hot blocks are replicated to more nodes for load distribution.
- Replication serves dual purposes — load distribution for hot blocks and fault tolerance for HDD reliability; replicas are placed across failure domains to ensure both.

**Solution 4 — Workload-Aware Resource Scaling.**
- The workload shifts from time to time. Concretely, at different times, there are different numbers of cold start queries.
- We use reactive auto-scaling triggered by observed queue depth (not predictive) and automatically scale up/down resources in EmberStore. This helps steady-state throughput, not burst spikes; scaling latency is acknowledged as a limitation for sudden cold-start bursts.

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