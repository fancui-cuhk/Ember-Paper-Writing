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

- When a query arrives, and the required index is not resident in compute memory or SSD cache, the system must perform a **cold start query** — fetching data from the storage layer before query execution can begin.
- *Note: Our use of **cold start query** differs from the serverless literature /cite{pinecone, milvus, turbopuffer}. Here, a query is cold-start if the data is not ready, whereas in the serverless context, a query/request is cold if the compute resources are not ready.*

**Real numbers reported at billion scale:**

| System              | Cold Start Query Latency                        |
| ------------------- | ----------------------------------------------- |
| Pinecone Serverless | Up to ~20 seconds                               |
| Milvus / Zilliz     | Multi-second segment loading                    |
| Turbopuffer         | ~400–900ms at 1M scale, higher at billion scale |

For RAG, real-time search, and interactive LLM applications, tail latency at this scale is a **blocking problem**.

---

## 6. Root Causes

> Key message: Analyze the root causes behind high tail latency / slow cold start queries.

#### Root Cause 1: Storage disaggregation penalty

Storage disaggregation is key to many cloud advantages: e.g., elastic scaling, etc. But, it is born with a cost: compute nodes must load data from remote storage frequently.

Disaggregation works well for workloads that can stream or process data in independent chunks, e.g., OLAP table scans. However, vector index search — especially HNSW — is data-dependent: each hop determines the next node to fetch. The entire index, or a substantial portion of it, must reside in memory or otherwise queries will be blocked. This data transfer via network is on the critical path of every cold query – expensive and slow!

#### Root Cause 2: Read amplification vs. Excessive unparallelizable IOs

We either (1) load the index in its entirety before query execution, or (2) load index chunks on demand.

(1) leads to severe read amplification. Only a small portion of the index is actually required by a query.

(2) leads to unparallelizable IOs, especially for HNSW. IVF can alleviate.

Milvus and Pinecone choose (1) – store a full index as one file and load it in its entirety before query.

#### Root Cause 3: Opacity of object storage vs. Heterogeneous and skewed access patterns

On one end, object storage is completely opaque. It exposes no interface for placement hints, co-location of frequently co-accessed clusters, or access-frequency metadata. E.g., it cannot express "these two clusters are always probed together, store them adjacent".

On the other end, access-aware optimization is important in vector DBs: In a multi-tenant system, tenants have vastly different query rates — some query continuously, many are cold archives. Within a single index, query load concentrates on popular vector neighborhoods; access is far from uniform. A single static storage layout cannot efficiently serve all access patterns simultaneously.

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