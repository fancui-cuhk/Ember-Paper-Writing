# Session: Introduction Framework and Motivation

- **Date**: 2026-05-28
- **Topic Tags**: `introduction`, `motivation`, `cloud-native`, `cold-query`, `architecture`

## User Intent

The user outlined the high-level structure for the Ember paper's Introduction section. Ember is a **serverless vector search service** targeting cloud deployment (positioned against Pinecone, Zilliz Cloud). The core narrative arc:

1. **User Value Dimensions**: Define what users expect from a cloud vector database
2. **System Comparison Table**: Show that no existing system satisfies all dimensions (key motivation)
3. **Cloud-Native Superiority**: Argue that cloud-native is the right direction for the LLM era
4. **Architecture Deep-Dive**: Explain cloud-native architecture clearly (with diagram)
5. **Cold Query Problem**: Present real data showing the critical issue in cloud-native systems
6. **Root Cause Analysis**: Explain WHY cold queries are inevitable in current designs

## Discussion Points and Consensus

### 1. User Value Dimensions (User-Provided)

| Dimension | Description |
|-----------|-------------|
| Low Usage Cost | Support billion-scale workloads economically |
| High Query Performance | Millisecond or tens-of-milliseconds average latency |
| Stable Query Performance / Low Tail Latency | No unacceptable stragglers |
| High Scalability | Respond to load fluctuations (scale up/down) |
| Low Operational Cost | Fully managed, no operational burden on users |
| High Availability | Data durability guarantees |

### 2. Confirmed Supplementary User Value Dimensions

The following two dimensions are **confirmed for inclusion** in the motivation table:

| Dimension | Description |
|-----------|-------------|
| **Recall vs Latency Tunability** | Ability to trade off accuracy for speed based on application needs — different use cases demand different recall/latency tradeoffs |
| **Multi-tenancy Isolation** | Namespaces with performance isolation — critical for SaaS use cases serving multiple customers from shared infrastructure |

### 3. Other Considered Dimensions (Not Included)

The following dimensions were considered but **not included** to maintain focus:

| Dimension | Rationale for Exclusion |
|-----------|------------------------|
| Hybrid Search Support | Core vector search functionality assumed baseline |
| Real-time Ingestion | Secondary feature, not core to tail latency motivation |
| Data Privacy / Compliance | Enterprise requirement but orthogonal to performance/cost thesis |

### 4. System Comparison Data (Real Metrics)

Based on web search, here is the current state of comparative data across **eight dimensions**:

| System | Cost | Mean/p50 Latency | p99 Tail Latency | Scalability | Availability | Operational Cost | Recall/Latency Tunability | Multi-tenancy Isolation |
|--------|------|------------------|------------------|-------------|--------------|------------------|---------------------------|-------------------------|
| **Cloud-Hosted** (EC2/RDS) | High (VM-hour billing) | Low (1-10ms warm) | Low (predictable) | Low (manual provision) | Medium (self-managed) | High (DevOps burden) | **High** (full control) | **Low** (single tenant per instance) |
| **Pinecone Serverless** | Low (10x vs pod) | Low (50-100ms warm p99) | **Extremely High (up to 20s cold)** | High (auto) | High (99.99% SLA) | Low (fully managed) | **Medium** (limited index types) | **High** (native namespace isolation) |
| **Zilliz Cloud** | Low-Medium | Medium (40-60ms) | High (cold segment loading) | High (elastic) | High | Low | **High** (IVF, HNSW, GPU indexes) | **Medium** (collection-level isolation) |
| **Turbopuffer** | Very Low | Very Low (sub-10ms warm p50) | High (300ms-4s cold p99) | High (auto) | High | Low | **Medium** (SPFresh centroid-based) | **Medium** (namespace isolation) |

**Key Data Points Verified**:

**Pinecone Serverless** (Source: Pinecone engineering blog, 2024):
- Cold start: up to ~20 seconds for billion-scale datasets
- Warm query p99: 50-100ms
- Cost: 10x reduction vs pod-based architecture

**Turbopuffer** (Source: Turbopuffer docs, ANN v3 blog):
- Cold query p50: ~874ms for 1M docs, ~57ms median with SPFresh at 100B scale
- Cold query p99: 300ms-4s depending on dataset size
- Warm query p50: sub-10ms, p99 ~35ms
- Architecture: SPFresh centroid-based index specifically designed to minimize S3 round-trips

**Milvus/Zilliz** (Source: Milvus GitHub issues, production blogs):
- Cold segment loading causes multi-second spikes (3s+ reported in production)
- QueryNode memory must be 2x index size to avoid swapping latency
- Milvus 2.6 tiered storage reduces costs by 50% but cold query problem remains

### 5. Cloud-Native Architecture Explanation (Draft)

The user requests a clear architectural diagram and explanation covering:

```
┌─────────────────────────────────────────────────────────────┐
│                    Compute Layer (Stateless)                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ Query Node 1│  │ Query Node 2│  │    ... (Elastic)    │  │
│  │ (Cache)     │  │ (Cache)     │  │                     │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│         ▲                                                    │
│         │ Load index on demand                               │
├─────────┼────────────────────────────────────────────────────┤
│         │                                                    │
│  ┌──────┴──────────────────────────────────────────────────┐ │
│  │              Object Storage (S3/GCS/Azure)               │ │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────────┐ │ │
│  │  │ Index   │ │ Index   │ │ Index   │ │   Metadata      │ │
│  │  │Segment 1│ │Segment 2│ │Segment N│ │   (etcd)        │ │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────────────┘ │ │
│  └──────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Key Concepts to Explain**:
1. **Compute-Storage Disaggregation**: Index durably stored on object storage; compute nodes load on demand
2. **Hot Query**: Data cached in compute memory/SSD → sub-10ms latency possible
3. **Cold Query**: Data not resident → must fetch from object storage (S3 GET: 100-200ms baseline, p99 up to 500ms-1s+)
4. **Serverless**: Stateless compute with scale-to-zero; pay per query/byte accessed
5. **Scale Up/Down**: Elastic compute pool serves multiple tenants

### 6. Cold Query Problem Magnitude

| System | Cold Query Latency (Billion Scale) |
|--------|-------------------------------------|
| Pinecone Serverless | Up to 20 seconds |
| Milvus (Zilliz) | Multi-second segment loading |
| Turbopuffer | ~400ms-900ms at 1M scale, higher at billion scale |

**Critical Insight**: Even "optimized" cloud-native systems face fundamental tension:
- **Read Amplification** (store full index as monolithic file): Load entire structure to use small fraction
- **Request Amplification** (decompose into per-cluster objects): nprobe parallel S3 GETs with tail latency variance

### 7. Root Causes (from summaries/cloud-hosted-vs-cloud-native.md)

1. **Disaggregation Penalty for Index-Structured Workloads**: Graph traversal (HNSW) is data-dependent; cannot prefetch/stream
2. **Read Amplification vs Request Amplification Tradeoff**: Neither monolithic files nor fine-grained decomposition achieves sub-second cold queries at scale
3. **Object Storage Opacity**: No placement hints, co-location controls, or read-ahead directives
4. **Heterogeneous Access Patterns**: Static storage layout cannot serve all tenants/index regions efficiently

## Open Questions / To Be Verified

1. ~~Should supplementary dimensions be included in the motivation table?~~ **RESOLVED: Recall/Latency Tunability and Multi-tenancy Isolation confirmed**
2. **Do we have experimental data for cloud-hosted systems (e.g., pgvector on RDS p99 latency)?** — Current data from web sources may need verification
3. **What is the exact architectural diagram style for the paper?** — User mentioned "a figure" but specific style (layered, flowchart, etc.) not specified
4. **Should we mention specific competitors beyond Pinecone/Milvus/Turbopuffer?** (e.g., Weaviate, Qdrant, pgvectorscale)

## Relationship to Previous Records

- **Extends**: `2026-05-28-workflow-setup.md` — This is the first substantive technical discussion following workflow establishment
- **References**: `summaries/cloud-hosted-vs-cloud-native.md` — Contains root cause analysis that should be cited in the "WHY" section

## Action Items

1. **User to confirm**: Which supplementary dimensions (if any) to include
2. **User to provide**: Experimental p99 latency data for cloud-hosted baseline (or confirm we rely on published benchmarks)
3. **User to review**: System comparison table for accuracy before inclusion in paper
4. **Next discussion**: Design section — How Ember solves the cold query problem (architecture preview)
