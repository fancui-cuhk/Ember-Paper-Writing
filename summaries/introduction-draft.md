# Introduction (Draft — Point Form)

> **Status**: Motivation and problem statement only. Solution/Ember design intentionally omitted.

---

## 1. Background

- In the LLM era, vector search has become foundational infrastructure for RAG, recommendation, and semantic retrieval.
- Production vector volume routinely reaches billion scale, and continues to grow.
- Not all vectors are accessed equally.
    - Some workloads involve frequent queries (e.g., real-time recommendation, active RAG sessions);
    - others involve infrequent or archival access (e.g., historical compliance records, long-tail user content).
- These requirements pose new challenges for vector database design.

---

## 2. What Users Expect

- **Low TCO**: Supports billion-scale workloads economically and efficiently
- **Low average latency**: On average, queries complete with low latency, ms or tens of ms
- **Low tail latency**: p99 / p999 latency is bounded and predictable -- under 500 ms
- **High scalability**: System can handle growing data volume and bursty query load gracefully
- **High availability**: Data durability guarantees and fault tolerance
- **Recall tunability**: Users can explictily tune index parameters for recall-latency tradeoffs

---

## 3. Two Architectural Paradigms

### 3.1 Two Approaches

- **Cloud-hosted**: Provisioned vector databases/engines (OpenSearch, PostgreSQL+pgvector, Milvus 1.x) rehosted on cloud VMs without fundamental architectural change
  - For good performance, stores vectors and vector indexes on RAM or local SSDs

- **Cloud-native**: Systems designed from the ground up for cloud infrastructure (Milvus 2.x, Pinecone, Turbopuffer)

### 3.2 Comparison on User Value Dimensions

| Dimension | Cloud-Hosted (AWS Opensearch, AWS RDS PG+pgvector) | Cloud-Native (Pinecone, Milvus 2.x, Turbopuffer) | **Ember (our system)** |
|-----------|-------------------------------------|--------------------------------------------------|-------|
| **Low TCO** | ❌ | ✅ | ✅ |
| **Low average latency** | ✅ | ✅ | ✅ |
| **Low tail latency** | ✅ | ❌ | ✅ |
| **High scalability** | ❌ | ✅ | ✅ |
| **High availability** | ⚠️ | ✅ | ✅ |
| **Recall tunability** | ⚠️ | ⚠️ | ✅ |

**Details:**

- **Low TCO**: Cloud-hosted systems keep compute always-on and store indexes on expensive RAM or SSD. Cloud-native systems use on-demand compute scaling and store indexes on cheap object storage (AWS S3).
- **Low average latency**: Cloud-hosted systems keep indexes always hot (RAM/SSD-resident). In cloud-native systems, most queries (except cold start ones) are fast because indexes are cached on compute nodes.
- **Low tail latency**: Cloud-hosted systems are single-tenant. Cloud-native systems suffer from cold start queries (need to prepare compute + load indexes) and noisy neighbors due to multi-tenancy.
- **High scalability**: Cloud-hosted systems require manual provisioning and are hard to scale to billion-scale. Cloud-native systems support auto-scaling.
- **High availability**: Cloud-hosted systems place operational burden on the user. Cloud-native systems are fully-managed by the vendor.
- **Recall tunability**: Both cloud-hosted and cloud-native systems vary — some expose parameters, others optimize internally.

**Key insight:**

In the LLM era, cloud-hosted solutions are becoming obsolete due to its high TCO and limited scalability. Customers with billion-scale vector corpora or bursty query loads suffer a lot on such systems.

Moving forward, we argue that the cloud-native architecture is the future, and we focus our discussions on them in the following.

---

## 4. Cloud-Native Architecture

```
Components:
- Compute Layer: Stateless query nodes with local cache (memory/SSD)
- Storage Layer: Durable, elastic storage (object storage)
- Metadata Service: Coordinator (etcd/ZooKeeper) for index metadata, cluster state
- Data Flow: Index segments stored in storage layer; compute nodes load segments on demand

Query path:
- Hot query (cache hit): Data in compute cache → low latency
- Cold query (cache miss): Compute fetches from storage → high latency

Key properties:
- Compute and storage scale independently
- Compute nodes are stateless and can be added/removed elastically
- Storage provides durability and high availability
- Multiple tenants share the compute and storage pools
```

**But, this architecture leads to the tail latency problem (next section).**

---

## 5. The Tail Latency Problem

**The "but":**

Cloud-native architecture is superior on TCO, scalability, and availability. However, it introduces a critical failure mode that blocks adoption for latency-sensitive applications:

**Tail latency is unacceptable.**

**At billion scale:**

| System              | Cold Start Query Latency                        |
| ------------------- | ----------------------------------------------- |
| Pinecone Serverless | Up to ~20 seconds                               |
| Milvus / Zilliz     | Multi-second segment loading                    |
| Turbopuffer         | ~400–900ms at 1M scale, higher at billion scale |

For RAG, real-time search, and interactive AI applications, tail latency at this scale is a **blocking problem**.

### 5.1 What Causes Such High Tail Latency?

In the disaggregated architecture, when a query arrives, and the required index data is not resident in compute memory or SSD cache, the system must perform a **cold start query** — fetching data from the storage layer before query execution can begin.

This happens due to:

- Scale-to-zero (serverless eviction of idle data)
- Cache eviction under memory pressure
- Infrequent access (multi-tenant cold tenants)

### 5.2 Magnitude of the Problem

**Storage layer latency baseline (S3, GCS, Azure Blob):**

- First-byte latency: 100–200ms per GET
- p99 tail: 500ms–1s+ under load
- No latency SLA

---

## 6. Root Causes

**Four structural reasons why tail latency is hard to solve in cloud-native vector databases:**

1. **Disaggregation penalty for index-structured workloads**
   - Graph traversal (HNSW) is data-dependent; each hop determines the next node
   - Prefetching and streaming are infeasible
   - Substantial portions of the index must be resident before meaningful execution

2. **Read amplification vs. request amplification tradeoff**
   - Monolithic files (Milvus): load entire structure to use small fraction → read amplification
   - Per-cluster objects (Turbopuffer): nprobe parallel storage requests, each with tail latency variance → request amplification
   - Neither achieves sub-second cold start at billion scale on high-latency storage

3. **Storage layer opacity**
   - Object storage exposes no interface for placement hints, co-location, or read-ahead
   - Database cannot express access patterns to storage layer

4. **Heterogeneous access patterns**
   - Multi-tenant systems have vastly different query rates across tenants
   - Within a single index, query load concentrates on popular vector neighborhoods
   - Static storage layout cannot efficiently serve all patterns

---

## 7. Research Gap and Our Contribution

**Standing from the perspective of cloud vector database design:**

- We observe that cloud-native architecture is the inevitable direction for large-scale vector databases (TCO, scalability, availability)
- We also observe that tail latency under disaggregation is a fundamental blocker for latency-sensitive AI workloads
- Existing systems either (a) accept the tail latency problem, or (b) optimize for different storage assumptions (e.g., ESSD instead of S3)

**Our contribution (preview):**

Ember is a cloud-native vector database that addresses the tail latency problem while preserving the TCO, scalability, and availability advantages of cloud-native architecture.

*[Solution section intentionally omitted — to be drafted in subsequent discussion]*

---

## Open Questions

- How to precisely define "workload size + recall target" assumptions in the motivation table?
- Data source citations for cold start latency numbers (need authoritative references)

---

**Cross-references:**

- `discussions/2026-06-01-user-value-dims-refinement.md`
- `discussions/2026-06-01-recall-tunability-naming.md`
- `summaries/cloud-hosted-vs-cloud-native.md` (root causes)
- `related-work/lindorm-vector-sigmod2026.md` (storage assumption difference)