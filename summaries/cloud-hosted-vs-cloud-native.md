## Model 1: Cloud-Hosted Vector Databases
*Examples: OpenSearch on EC2, pg+pgvector on RDS, self-hosted Milvus 1.x on VMs*

- These systems are traditional vector search engines **rehosted on cloud VMs** — the architecture is unchanged from on-premise: a single self-contained VM with colocated CPU, RAM, and SSD.
- Indexes are loaded into RAM or kept on local SSD; queries execute with data always resident, delivering single-digit to tens of milliseconds warm latency.
- The billing unit is the VM instance-hour: you pay regardless of whether queries are arriving.
- This forces operators to **provision for peak load**; during quiet periods the majority of allocated resources sit idle at full cost.
- At billion scale, keeping all indexes memory-resident requires expensive, large instances — TCO scales linearly with data volume.
- These systems suit **predictable, sustained workloads** where peak utilization is well-understood, but are a poor fit for heterogeneous or bursty demand.

---

## Model 2: Cloud-Native Vector Databases
*Examples: Milvus 2.x, Pinecone, Turbopuffer*

- These systems are designed from the ground up to exploit cloud infrastructure primitives: **compute-storage disaggregation**, resource pooling, and elastic scaling.
- Vector indexes are durably stored on **object storage** (e.g., Amazon S3) as the primary persistence layer; compute nodes load indexes on demand to serve queries.
- Resource pooling allows one shared compute fleet to serve many tenants, improving hardware utilization and lowering per-tenant cost.
- Some systems (Pinecone, Turbopuffer) adopt **serverless billing** — pay per query or per byte accessed — further eliminating idle cost; others (Zilliz Cloud dedicated) offer provisioned compute clusters on top of disaggregated storage.
- When indexes are cached in compute memory, query latency is competitive with cloud-hosted systems.
- Low TCO, full manageability, and elastic scaling position these systems as the architectural direction for large-scale AI workloads.
- However, one critical failure mode stands between cloud-native vector databases and universal adoption: **cold query latency**.

---

## The Cold Query Problem within Cloud-Native Vector Databases

- When a tenant's index is not resident in compute memory — due to scale-to-zero, cache eviction under memory pressure, or infrequent access — the index must be fetched from object storage before the query can execute.
- Amazon S3 delivers first-byte latency of **100–200ms per GET request**, with no latency SLA and p99 tail latency that can spike to 500ms–1s+ under load.
- A vector index for a billion-scale collection reaches tens of gigabytes; even with parallel fetches, cold loading takes **seconds to tens of seconds**.
- Pinecone reports cold start latencies of **up to 20 seconds** for billion-scale datasets; independent benchmarks confirm multi-second cold starts even at moderate scale.
- Turbopuffer reduces cold start to **~400–900ms at 1M-vector scale** by storing each IVF cluster as a separate S3 object and fetching only probed clusters in parallel — but this approach faces growing request count and S3 tail latency variance at billion scale, where nprobe is large.
- For interactive applications — RAG pipelines, real-time semantic search — cold query latency at this scale is a **blocking problem**.

---

## Root Causes

*We identify four root causes that make cold query latency hard to solve in disaggregated vector databases:*

### Root Cause 1: Disaggregation penalty for index-structured workloads
- Compute-storage disaggregation works well for workloads that can stream or process data in independent chunks (e.g., OLAP column scans where rows are independent and reads are sequential).
- Vector index traversal — especially graph-based methods like HNSW — is **data-dependent**: each traversal hop determines the next node to fetch, making prefetching and streaming infeasible.
- The entire index, or a substantial portion of it, must be resident in compute memory before meaningful query execution can begin; this data transfer is on the critical path of every cold query.

### Root Cause 2: Fundamental tension between read amplification and request amplification
- Storing the full index as a monolithic file (Milvus) forces loading the entire structure to use a small fraction: for IVF, all nlist cluster posting lists are loaded even though only nprobe clusters (typically nprobe << nlist) are probed; for HNSW, the full graph must be loaded before any traversal begins. This is **read amplification**.
- Decomposing the index into per-cluster objects (Turbopuffer) eliminates read amplification but introduces **request amplification**: nprobe parallel S3 GETs, each carrying 100–200ms baseline latency, subject to S3 tail latency variance, and bounded by per-prefix request rate limits.
- Neither strategy is acceptable for sub-second cold query targets at billion scale: one wastes bytes, the other wastes round trips.

### Root Cause 3: Object storage opacity prevents access-aware optimization
- Object storage exposes no interface for placement hints, co-location of frequently co-accessed clusters, read-ahead directives, or access-frequency metadata.
- The storage layer is entirely opaque to the database: it cannot express "these two clusters are always probed together, store them adjacent" or "this tenant is frequently cold-started, prioritize its placement."
- A purpose-built storage layer can expose and exploit such access information, while object storage structurally cannot.

### Root Cause 4: Heterogeneous and skewed access patterns across tenants and within indexes
- In a multi-tenant system, tenants have vastly different query rates — some query continuously, many are cold archives.
- Within a single index, query load concentrates on popular vector neighborhoods; access is far from uniform.
- A single static storage layout cannot efficiently serve all access patterns simultaneously; adaptive placement and bandwidth allocation are required.
