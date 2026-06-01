# Session: Root Cause Analysis of Cloud-Native Tail Latency

- **Date**: 2026-05-30
- **Topic Tags**: `root-cause`, `cold-query`, `S3-latency`, `cloud-native`, `architecture`

## User Intent

1. **System Classification**: Consolidate Pinecone Serverless, Zilliz Cloud, and Turbopuffer into a single category — all are **cloud-native/S3-based systems** with similar fundamental characteristics.

2. **Competitor Categorization**: Determine the architectural class of other competitors (Weaviate, Qdrant, pgvectorscale).

3. **Architecture Diagram Style**: Request for "architecture diagram" showing system components, their functions, what they store, and computation responsibilities.

4. **Root Cause Validation**: Verify whether the four root causes previously identified accurately explain why cloud-native systems suffer from high tail latency; search for additional or alternative explanations.

## Discussion Points and Consensus

### 1. System Classification Consolidation

**Confirmed**: The three systems should be grouped under **"Cloud-Native / S3-Based / Serverless Vector Databases"**:

| System | Architecture | Primary Storage | Billing Model |
|--------|--------------|-----------------|---------------|
| **Pinecone Serverless** | Compute-storage disaggregation | S3 (blob storage) | Per query + storage |
| **Zilliz Cloud (Milvus)** | Microservices with tiered storage | S3/MinIO + local cache | Provisioned + storage |
| **Turbopuffer** | Stateless compute over object storage | S3/GCS/Azure Blob | Per query |

**Common Characteristics**:
- Index durably stored on object storage
- Compute nodes load data on-demand
- Elastic scaling / multi-tenant resource pooling
- Pay-per-use (vs. provisioned VM-hour)
- **All suffer from cold query latency due to S3 fetch overhead**

### 2. Competitor Architecture Classification

Based on research, other major competitors fall into the **cloud-hosted (non-S3-based)** category:

| System | Architecture Class | Storage | Deployment Options | Notes |
|--------|-------------------|---------|-------------------|-------|
| **Weaviate** | Cloud-Hosted | Local RAM + SSD (HNSW graph) | Self-hosted, Weaviate Cloud (managed VMs) | Memory-hungry (3-4x raw data size), full graph in RAM |
| **Qdrant** | Cloud-Hosted | Local RAM + SSD (HNSW) | Self-hosted, Qdrant Cloud (managed) | Purpose-built vector engine, Rust-based, no S3 disaggregation |
| **pgvector / pgvectorscale** | Cloud-Hosted | PostgreSQL storage (RDS/local) | Self-hosted on RDS/EC2 | Extension to Postgres, operates within single-node or replicated Postgres architecture |

**Critical Insight**: None of these competitors use S3 as the source-of-truth for vector indexes. They maintain indexes in local memory/storage, avoiding the cold query problem but incurring higher baseline costs and limited elasticity.

**For Paper Citation**: These systems can be cited as examples of "traditional cloud-hosted vector databases" alongside OpenSearch on EC2 and pgvector on RDS, demonstrating that the industry offers no cloud-native solution without cold query penalties.

### 3. Architecture Diagram Reference Examples

Based on web search, here are representative styles for vector database architecture diagrams:

**Example 1: Turbopuffer-Style Layered Architecture**
```
┌─────────────────────────────────────────┐
│          Query Interface (REST/gRPC)     │
├─────────────────────────────────────────┤
│  Compute Layer (Stateless Workers)       │
│  ┌─────┐ ┌─────┐ ┌─────┐               │
│  │Cache│ │Cache│ │Cache│  (Memory/NVMe) │
│  └─────┘ └─────┘ └─────┘               │
├─────────────────────────────────────────┤
│         Object Storage (S3)              │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐  │
│  │Indexes  │ │WAL Logs │ │Metadata │  │
│  └─────────┘ └─────────┘ └─────────┘  │
└─────────────────────────────────────────┘
```

**Example 2: Milvus Microservices Architecture** (from official docs):
- **Proxy Layer**: Request routing, authentication
- **Query Nodes**: Execute vector search (hold hot segments in memory)
- **Index Nodes**: Build vector indexes
- **Data Nodes**: Handle data insertion, log serialization
- **Storage Layer**: S3/MinIO (persist segments), etcd (metadata), Pulsar/Kafka (WAL)

**Recommended Structure for Ember Paper**:
- Show three tiers: **Query Layer** (stateless), **Caching Tier** (ephemeral), **Persistent Storage** (S3)
- Annotate data flow: "cold query" path vs "warm query" path
- Label each component's responsibility clearly

### 4. Root Cause Analysis — Validation and Refinement

#### 4.1 Existing Root Causes (from summaries/cloud-hosted-vs-cloud-native.md)

| # | Root Cause | Validation Status | Notes |
|---|-----------|-------------------|-------|
| 1 | Disaggregation penalty for index-structured workloads | ✅ **Confirmed** | HNSW graph traversal requires data-dependent hops; cannot prefetch/stream from S3 |
| 2 | Read amplification vs request amplification tradeoff | ✅ **Confirmed** | Monolithic files = load unused data; decomposed objects = nprobe parallel GETs |
| 3 | Object storage opacity | ✅ **Confirmed** | S3 provides no placement hints, co-location controls, or read-ahead directives |
| 4 | Heterogeneous access patterns | ✅ **Confirmed** | Static storage layout cannot serve all tenants/regions efficiently |

#### 4.2 Supporting Evidence (Not Separate Root Causes)

User feedback: S3 latency characteristics and HNSW-Storage mismatch are already covered by existing root causes. We clarify these as **supporting details** rather than independent causes:

**S3 Latency Characteristics** (supports Root Cause 3):
- S3 GET baseline: **100–200ms**, p99 tail: **500ms–1s+**
- Throughput limit: **5,500 GET/sec per prefix**
- Scaling is gradual — HTTP 503 "Slow Down" during ramp-up
- **Implication**: Even "fast" S3 fetches are 1000x slower than memory access

**HNSW-Storage Mismatch** (manifestation of Root Cause 1):
- Graph traversal requires memory residency for efficient hop-chasing
- Each edge = potential S3 round-trip = 100-200ms
- **Turbopuffer's solution**: SPFresh tree bounds round-trips to logarithmic height

#### 4.3 Additional Consideration: Metadata Discovery Overhead (Deferred)

**S3 LIST Operation Overhead** — identified from turbopuffer engineering blog, but requires clarification:

> **Question**: When loading a cold index from S3, how does the system know which files to fetch?

**The Problem**:
- S3 is **object storage**, not a filesystem — no native directory listing
- To discover which index segments exist, systems call `ListObjectsV2` API
- Each LIST returns max 1,000 keys; large buckets need **paginated sequential calls**
- Reported latency: **~200ms per page** (network round-trip)

**However**: Production systems (Pinecone, Turbopuffer) typically avoid LIST at query time by:
- Maintaining a `metadata.json` or index manifest in a **known location**
- This file contains the list of all segment files for a given index
- Only need to **GET this single metadata file** first, then fetch segments

**Verdict**: LIST overhead is a **design/implementation challenge**, not a fundamental root cause. Well-designed systems avoid it. **Not included in core root cause list**.

#### 4.3 Refined Root Cause Summary

We retain **four validated root causes** (original list confirmed accurate):

```
┌─────────────────────────────────────────────────────────────────────────┐
│           WHY CLOUD-NATIVE VECTOR DATABASES HAVE HIGH TAIL LATENCY      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. Disaggregation Penalty for Index-Structured Workloads               │
│     → Graph traversal (HNSW) is data-dependent; each hop determines     │
│       next node. Cannot prefetch or stream from S3.                     │
│     → Supporting evidence: HNSW requires memory residency; each edge    │
│       traversal = potential S3 round-trip (~100-200ms)                │
│                                                                         │
│  2. Read Amplification vs Request Amplification Tradeoff               │
│     → Monolithic index files: load 100% to use 1% (nprobe/nlist)      │
│     → Decomposed per-cluster objects: nprobe parallel S3 GETs           │
│       (each 100-200ms, subject to tail variance and rate limits)        │
│     → Neither achieves sub-second cold queries at billion scale         │
│                                                                         │
│  3. Object Storage Opacity                                              │
│     → S3 provides no placement hints, co-location controls, or          │
│       read-ahead directives for frequently co-accessed data             │
│     → Supporting evidence: S3 baseline 100-200ms, p99 500ms-1s+,       │
│       5,500 GET/sec/prefix limit — fundamentally not a low-latency    │
│       system; optimized for throughput and durability                  │
│                                                                         │
│  4. Heterogeneous and Skewed Access Patterns                            │
│     → Multi-tenant workloads: some tenants hot, most cold (archives)    │
│     → Within single index: access concentrates on popular vector        │
│       neighborhoods (skewed distribution)                               │
│     → Static storage layout cannot serve all patterns efficiently       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Open Questions / To Be Verified

1. **Architecture Diagram**: Confirm preferred visual style (layered boxes showing: Query Interface → Compute Layer → Caching Tier → Persistent Storage, with data flow annotations for cold vs warm paths)
2. **HNSW Alternative Feasibility**: Should we discuss whether graph-based indexes can be adapted for S3, or if tree-based (SPFresh) is the only viable path?
3. **S3 Express One Zone**: Should we mention S3 Express (single-digit ms latency) as a potential mitigation, or focus strictly on commodity S3?

## Relationship to Previous Records

- **Extends**: `2026-05-28-introduction-framework.md` — Refines root cause analysis and competitor classification
- **References**: `summaries/cloud-hosted-vs-cloud-native.md` — Original 4 root causes validated; S3 LIST overhead considered but excluded as implementation detail

## Action Items

1. **User to confirm**: Architecture diagram style (see Section 3 for examples)
2. **User to confirm**: 4 root causes are comprehensive for paper — no additional fundamental causes identified in research
3. **Next discussion**: Design section — Ember's approach to mitigating the 4 root causes
