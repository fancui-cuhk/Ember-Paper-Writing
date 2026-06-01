# Introduction (Draft — Point Form)

> **Status**: Motivation and problem statement only. Solution/Ember design intentionally omitted.

---

## 1. Background: Vector Search as AI Infrastructure

**Standing as a cloud vendor, we observe:**

- LLMs and generative AI have transformed how enterprises interact with unstructured data
- Representation learning enables semantic search via vector similarity
- RAG, multi-modal recommendation, intelligent QA are now production workloads
- **User demand for vector databases has shifted from experimental to production-grade**

**Key observation from production deployments (2024-2026):**

- Users prioritize: hybrid search, metadata filtering performance, operational simplicity, cost predictability
- Latency target: sub-100ms p99 for interactive applications
- Scale range: from <10M vectors (pgvector sufficient) to billion-scale (dedicated systems required)
- Filtering is critical: "searching all vectors is slow; filtering to 10K then searching is fast"

**Research gap we identify:**

- Cloud-native architecture promises cost efficiency and elasticity
- But tail latency under disaggregation remains unaddressed
- This gap blocks universal adoption for latency-sensitive AI applications

---

## 2. What Users Need: Six Core Dimensions

**Note on workload size and recall target:**

> TCO and latency are inherently tied to workload scale and recall requirements. A system may achieve low TCO at 10M vectors with 80% recall, but the same TCO at 1B vectors with 95% recall is a different problem. Unless otherwise noted, our analysis assumes **billion-scale workloads with a target recall of 90%+**.

**Six dimensions:**

- **Low TCO**: Economical at billion-scale; pay for actual usage
- **Low average latency**: At a given recall target, typical queries complete with low latency
- **Low tail latency**: At a given recall target, p99 / p999 latency is bounded and predictable
- **High scalability**: Elastic response to load fluctuations (scale up/down)
- **High availability**: Data durability and fault tolerance
- **Recall tunability**: Users can explicitly select different recall targets to trade off accuracy for latency based on workload requirements

**Tunability check (Pinecone, Milvus, Turbopuffer):**

- **Pinecone** (S1/P1 pods): exposes `ef_search` → users can tune recall vs latency
- **Milvus**: exposes `nprobe` → users can tune recall vs latency
- **Turbopuffer**: SPFresh is self-optimizing, targets 90-95% recall@10 automatically, does NOT expose `nprobe`-like parameters; provides recall endpoint for testing, higher recall available case-by-case → **limited tunability**

---

## 3. Two Architectural Paradigms

### 3.1 Two Approaches

- **Cloud-hosted**: Traditional vector databases (OpenSearch, pgvector, self-hosted Milvus 1.x) rehosted on cloud VMs without fundamental architectural change
- **Cloud-native**: Systems designed from the ground up for cloud infrastructure (Milvus 2.x, Pinecone, Turbopuffer, LindormVector)

### 3.2 Comparison on User Value Dimensions

| Dimension | Cloud-Hosted (EC2/RDS, self-hosted) | Cloud-Native (Pinecone, Milvus 2.x, Turbopuffer) |
|-----------|-------------------------------------|--------------------------------------------------|
| **Low TCO** | ❌ High: VM-hour billing, provision for peak, idle resources wasted | ✅ High: resource pooling, elastic scaling, pay-per-use models |
| **Low average latency** | ✅ Low (warm, memory-resident) | ✅ Low (when cached) |
| **Low tail latency** | ✅ Predictable (single-tenant, always warm) | ❌ Problematic (cold start query can spike to seconds) |
| **High scalability** | ❌ Manual provisioning, slow to scale | ✅ Auto-scaling, stateless compute |
| **High availability** | ⚠️ Self-managed, operational burden on user | ✅ Managed, SLA-backed |
| **Recall tunability** | ✅ Full control (user manages index) | ⚠️ Varies: Pinecone/Milvus expose parameters; Turbopuffer is self-optimizing with limited user control |

**Key insight:**

Cloud-native systems are superior on TCO, scalability, and availability — the dimensions that matter for production AI workloads at scale. However, they introduce a new problem: **tail latency under disaggregation**.

---

## 4. Cloud-Native Architecture

**Objective description (no marketing language):**

```
[Architecture diagram: cloud-native-vector-db-architecture.png]

Components:
- Compute Layer: Stateless query nodes with local cache (memory/SSD)
- Storage Layer: Durable, elastic storage (object storage or distributed filesystem)
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

**This architecture enables:**

- Cost efficiency through resource pooling
- Elastic scaling without data movement
- Separation of concerns (compute vs storage)

**But it also creates the tail latency problem (next section).**

---

## 5. The Tail Latency Problem

**The "but":**

Cloud-native architecture is superior on TCO, scalability, and availability. However, it introduces a critical failure mode that blocks adoption for latency-sensitive applications:

**Tail latency is unacceptable.**

### 5.1 What Causes Tail Latency?

In a disaggregated architecture, when a query arrives and the required index data is not resident in compute memory, the system must perform a **cold start query** — fetching data from the storage layer before query execution can begin.

This happens due to:

- Scale-to-zero (serverless eviction of idle data)
- Cache eviction under memory pressure
- Infrequent access (multi-tenant cold tenants)

### 5.2 Magnitude of the Problem

**Storage layer latency baseline (S3, GCS, Azure Blob):**

- First-byte latency: 100–200ms per GET
- p99 tail: 500ms–1s+ under load
- No latency SLA

**At billion scale:**

| System | Cold Start Query Latency | Source |
|--------|--------------------------|--------|
| Pinecone Serverless | Up to ~20 seconds | Pinecone engineering blog, 2024 |
| Milvus / Zilliz | Multi-second segment loading | Production reports |
| Turbopuffer | ~400–900ms at 1M scale, higher at billion scale | Turbopuffer docs |

For RAG, real-time search, and interactive AI applications, tail latency at this scale is a **blocking problem**.

---

## 6. Root Causes

**Four structural reasons why tail latency is hard to solve in disaggregated vector databases:**

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

**Standing as a cloud vendor:**

- We observe that cloud-native architecture is the inevitable direction for large-scale vector databases (TCO, scalability, availability)
- We also observe that tail latency under disaggregation is a fundamental blocker for latency-sensitive AI workloads
- Existing systems either (a) accept the tail latency problem, or (b) optimize for different storage assumptions (e.g., ESSD instead of S3)

**Our contribution (preview):**

Ember is a cloud-native vector database that addresses the tail latency problem while preserving the TCO, scalability, and availability advantages of cloud-native architecture.

*[Solution section intentionally omitted — to be drafted in subsequent discussion]*

---

## Open Questions

- How to precisely define "workload size + recall target" assumptions in the motivation table?
- Should we include LindormVector in the comparison table (different storage assumption: ESSD vs S3)?
- Data source citations for cold start latency numbers (need authoritative references)

---

**Cross-references:**

- `discussions/2026-06-01-user-value-dims-refinement.md`
- `discussions/2026-06-01-recall-tunability-naming.md`
- `summaries/cloud-hosted-vs-cloud-native.md` (root causes)
- `related-work/lindorm-vector-sigmod2026.md` (storage assumption difference)