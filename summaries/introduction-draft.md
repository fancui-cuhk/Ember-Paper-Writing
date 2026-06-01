# Introduction (Draft — Motivation Section Only)

> **Status**: Motivation, problem statement, and root causes are complete. Solution/Ember design section **intentionally omitted** — to be added in future discussion.

---

## 1. The Rise of Vector Search in AI Applications

The emergence of large language models (LLMs) and generative AI has fundamentally transformed how enterprises interact with unstructured data. Representation learning technologies embed text, images, audio, and video into high-dimensional vectors, enabling semantic understanding through vector similarity search. This capability has become foundational for modern AI applications including Retrieval-Augmented Generation (RAG), multi-modal recommendation systems, and intelligent question-answering.

Consequently, **vector search has become a critical building block of modern data infrastructure**. Enterprises require efficient, scalable, and cost-effective vector databases to manage billion-scale vector collections at reasonable cost.

---

## 2. What Users Need: A Value-Centric View

Building on the analysis of production workloads and existing systems, we identify six core dimensions that define a cloud-native vector database's value proposition:

| Dimension | Description | Why It Matters |
|-----------|-------------|----------------|
| **Low TCO** | Total cost of ownership must be economical at billion-scale; users pay for actual resources consumed | Enables cost-effective long-term operation of large-scale AI workloads |
| **Low average latency** | At a given recall target, typical queries complete with low latency | Supports interactive applications (chatbots, real-time search) |
| **Low tail latency** | At a given recall target, p99 / p999 latency is bounded and predictable | Critical for SLA-bound production services; stragglers cause timeouts and poor UX |
| **High scalability** | System responds to load fluctuations by scaling resources up or down | Matches the bursty, unpredictable nature of AI workloads |
| **High availability** | Data durability guarantees and fault tolerance | Production systems cannot tolerate data loss or extended outages |
| **Recall tunability** | Users can explicitly select different recall targets to trade off accuracy for latency based on workload requirements | Different workloads demand different recall levels; some prioritize high performance at lower recall, others require high recall |

These dimensions are **logically orthogonal** — Performance dimensions describe latency at a fixed recall target, while Recall tunability describes the ability to choose among multiple recall targets. Cloud-hosted systems excel at latency predictability but fail on cost and elasticity. Cloud-native systems achieve cost efficiency and elasticity but struggle with tail latency, particularly on the **cold path** when data is not resident in compute memory.

---

## 3. Two Architectural Paradigms

### 3.1 Cloud-Hosted Vector Databases

Traditional vector databases (OpenSearch, pgvector on RDS, self-hosted Milvus 1.x) are essentially **rehosted on cloud VMs** without fundamental architectural change. They feature:

- **Colocated compute and storage**: CPU, RAM, and SSD reside on the same VM
- **Always-resident indexes**: Data loaded into RAM or kept on local SSD
- **Single-digit millisecond warm latency** with predictable performance
- **Instance-hour billing**: Pay for provisioned capacity regardless of actual query load
- **Manual provisioning**: Operators must provision for peak load, leading to idle resources during quiet periods

**The fundamental limitation**: At billion scale, keeping all indexes memory-resident requires expensive, large instances. TCO scales linearly with data volume. These systems suit **predictable, sustained workloads** but are a poor fit for heterogeneous or bursty demand.

### 3.2 Cloud-Native Vector Databases

A new generation of systems (Milvus 2.x, Pinecone, Turbopuffer, and industrial systems like LindormVector) are designed from the ground up to exploit cloud infrastructure primitives:

- **Compute-storage disaggregation**: Vector indexes durably stored in distributed storage (DFS, object storage); compute nodes load on demand
- **Resource pooling**: One shared compute fleet serves many tenants, improving utilization
- **Elastic scaling**: Stateless compute scales horizontally; storage scales independently
- **Serverless billing models**: Pay per query or byte accessed (in some systems)

When indexes are cached in compute memory, these systems deliver latency competitive with cloud-hosted alternatives, while achieving **order-of-magnitude better cost efficiency and elasticity**.

**However, a critical failure mode blocks universal adoption: cold query latency.**

---

## 4. The Cold Query Problem

### 4.1 What Is a Cold Query?

In a disaggregated architecture, a **cold query** occurs when the index data required to answer a query is not resident in compute memory. This happens due to:

- **Scale-to-zero**: Serverless systems evict idle data to minimize cost
- **Cache eviction**: Under memory pressure, less-frequently accessed data is evicted
- **Infrequent access**: Multi-tenant systems naturally have cold data for low-activity tenants

When cold, the index must be fetched from the storage layer before query execution can begin.

### 4.2 Magnitude of the Problem

**Object storage latency baseline** (AWS S3, GCS, Azure Blob):
- First-byte latency: **100–200ms per GET request**
- p99 tail latency: **500ms–1s+** under load
- No latency SLA guarantees

**At billion scale**, a vector index reaches tens of gigabytes. Even with parallel fetches, cold loading takes **seconds to tens of seconds**:

| System | Cold Query Latency (Billion Scale) | Source |
|--------|-------------------------------------|--------|
| **Pinecone Serverless** | Up to ~20 seconds | Pinecone engineering blog, 2024 |
| **Milvus / Zilliz** | Multi-second segment loading | Production reports, GitHub issues |
| **Turbopuffer** | ~400–900ms at 1M scale, higher at billion scale | Turbopuffer docs, ANN v3 blog |

For interactive applications — RAG pipelines, real-time semantic search — cold query latency at this scale is **a blocking problem**. The promise of serverless cost efficiency is undermined if the first query after idle time takes tens of seconds.

### 4.3 Structural Root Causes

The cold query problem is **not an implementation bug** but a consequence of fundamental architectural tensions:

**Root Cause 1: Disaggregation penalty for index-structured workloads**
- Graph traversal (HNSW) is **data-dependent**: each hop determines the next node, making prefetching and streaming infeasible
- The entire index or substantial portions must be resident before meaningful execution can begin
- This data transfer is on the critical path of every cold query

**Root Cause 2: Read amplification vs. request amplification tradeoff**
- **Monolithic files** (Milvus): Load entire structure to use small fraction → read amplification
- **Per-cluster objects** (Turbopuffer): nprobe parallel storage requests, each with tail latency variance → request amplification
- Neither strategy achieves sub-second cold queries at billion scale on high-latency storage

**Root Cause 3: Storage layer opacity**
- Object storage exposes no interface for placement hints, co-location of frequently co-accessed data, or read-ahead directives
- The database cannot express access patterns to the storage layer

**Root Cause 4: Heterogeneous access patterns**
- Multi-tenant systems have vastly different query rates across tenants
- Within a single index, query load concentrates on popular vector neighborhoods
- A single static storage layout cannot efficiently serve all patterns

---

## 5. Why Cloud-Native Is Still the Right Direction

Despite the cold query challenge, we argue that **cloud-native architecture is the inevitable direction** for large-scale vector databases:

1. **Workload characteristics**: AI-driven workloads are bursty, multi-tenant, and unpredictable — cloud-native elasticity matches this reality
2. **Economics**: Instance-hour billing forces over-provisioning; pay-per-use models align costs with actual value
3. **Operational burden**: Self-hosted systems impose DevOps costs that most enterprises cannot sustain
4. **Scale**: Billion-scale datasets are increasingly common; disaggregation is the only viable path

The question is not whether to adopt cloud-native architecture, but **how to make it work for latency-sensitive applications**.

---

## 6. Contribution Preview (To Be Completed)

We present **Ember**, a serverless vector database that achieves...

*[Solution section intentionally omitted — to be drafted in subsequent discussion]*

---

## Summary of Motivation Structure

| Section | Content | Cross-references |
|---------|---------|------------------|
| §1 | AI/LLM context establishing vector search importance | — |
| §2 | User value dimensions (7 dimensions, refined) | `discussions/2026-06-01-user-value-dims-refinement.md` |
| §3 | Two architectural paradigms comparison | `summaries/cloud-hosted-vs-cloud-native.md` |
| §4 | Cold query problem: definition, magnitude, root causes | `summaries/cloud-hosted-vs-cloud-native.md` |
| §5 | Argument for cloud-native despite challenges | — |
| §6 | Ember contribution preview | *[Future work]* |

---

**Known open questions**:
- Final name for "performance stability" dimension
- Whether to include LindormVector in the system comparison table (different storage assumption: ESSD vs. S3)
- Data source citations for cold query latency numbers (need authoritative references)
