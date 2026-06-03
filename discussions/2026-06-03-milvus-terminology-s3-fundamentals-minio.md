# Session: Milvus Load Metrics, S3 Fundamentals, Slab Parallelism, MinIO

- **Date:** 2026-06-03
- **Topic Tags:** `milvus`, `tiered-storage`, `S3`, `MinIO`, `sequential-read`, `bandwidth`, `cold-query`, `pinecone`, `slab`

## User questions (summary)

1. Confusion about Milvus tiered-storage blog metrics: **data download time**, **index loading time**, and **cold data search time** — should they be added together for a cold user query?
2. Skepticism about S3 bandwidth: back-of-envelope ~200 MB/s from the blog vs marketed “high S3 bandwidth”; what is actually happening?
3. Storage fundamentals: sequential vs random read; 1×1GB vs 2×500MB vs 5×200MB on S3; conditions for S3 peak bandwidth.
4. Interpretation of Pinecone-style cold path: one 1GB slab ≈ one sequential GET ~300 MB/s; many slabs in parallel → linear bandwidth scaling; within-slab 8–16 MiB range GETs → very high bandwidth — is this correct?
5. Record session; add author’s views on **MinIO** use cases.

---

## Discussion要点

### A. Milvus blog terminology (load vs query)

The blog uses **two separate measurement contexts**; they must not be summed as one cold-query critical path.

| Metric | Phase | What it measures |
|--------|-------|------------------|
| **Data download time** | `Collection.load()` | Pulling **field data** (binlogs/chunks) from object storage to QueryNode |
| **Index loading time** | `Collection.load()` (overlapping or following download) | Loading index files into searchable form (mmap / memory / deserialize) |
| **Cold data search time** | After collection is **loaded** | P99 of `search()` when cache misses — **includes on-demand S3 fetch + ANN** in one number |

**Milvus 2.5 full-load:** User cannot query until ~25 min (22 min download + 3 min index load for ~430 GB HNSW benchmark). Afterwards cold search P99 ≈ 15 ms (data already local).

**Milvus 2.6+ tiered:** `load()` ≈ 45 s (28 s + 17 s) with only ~12 GB local initially — **not** downloading full 430 GB. Collection becomes queryable; first cold search P99 ≈ **120 ms** (benchmark: 100M vectors, MinIO, 1 QueryNode 4 vCPU). This **120 ms is not** 28s + 17s + 120ms.

**“~500 ms”** in blog “good-fit” prose is qualitative guidance, not the same row as the 120 ms benchmark table.

**Source:** [Milvus tiered storage blog](https://milvus.io/blog/milvus-tiered-storage-80-less-vector-search-cost-with-on-demand-hot%E2%80%93cold-data-loading.md), [tiered storage overview](https://milvus.io/docs/tiered-storage-overview.md).

### B. Field chunk vs index file (prior thread, retained here)

- **Field chunk:** Column data split into storage chunks; tiered mode can load **per chunk** on demand.
- **Index file:** Per-segment ANN index; must be fetched as a **whole file** (not chunk-split). HNSW/IVF_FLAT embed raw vectors → search may skip separate vector field load; IVF_SQ8/PQ/DISKANN require field data for raw vectors.

### C. S3 bandwidth: why ~200–330 MB/s in blog ≠ “S3 is slow”

**Consensus from AWS docs + VLDB’23 (Durner et al.):**

- Peak **aggregate** throughput needs **many parallel connections**; single connection capped ~**5 Gbps (~625 MB/s)** per AWS re:Post.
- **Per-prefix** soft limit ~**5,500 GET/s**; burst can 503.
- **Small requests:** first-byte latency dominates (~tens of ms); single-request throughput ~**tens of MiB/s** (HDD-like); hundreds of concurrent requests needed to saturate NIC.
- **8–16 MiB** per request often cost-throughput optimal for analytics (VLDB paper).
- Vector/search on object storage: many small/random logical reads → **metadata + TTFB bound**, not wire bandwidth bound (Mach5 blog, aligned with paper).

**Milvus blog caveat:** Benchmark uses **MinIO**, not AWS S3; 1 QueryNode, 4 vCPU, 10 Gbps — effective load throughput reflects **client parallelism + CPU + disk**, not S3 marketing peak.

**Open problem (unchanged):** No public step-by-step cold-query I/O trace for Pinecone/Milvus at billion scale.

### D. User’s slab / parallel-read model — partial agreement

| Claim | Assessment |
|-------|------------|
| 1 GB slab, one GET ≈ sequential transfer ~300 MB/s | **Roughly right** for one large object, one connection; 300 MB/s is **observed average**, not guaranteed. Pinecone slab = **multiple files** (manifest, index, vectors) — structure **not fully published**. |
| N cold slabs in parallel → aggregate bandwidth scales ~linearly with N | **Directionally right** until client NIC, CPU, connection pool, prefix rate limits. **End-to-end query latency** does not scale inversely with N (merge, dependencies, ANN compute). |
| Within slab: 8–16 MiB parallel range GETs → very high bandwidth | **Valid for bulk transfer** per AWS best practice; cold query still limited by **TTFB × #requests**, **parallelism actually used**, and **post-download index work**. Pinecone internal layout **unknown**. |

### E. Sequential vs random read (fundamentals captured for paper)

**Local disk:** Sequential = consecutive offsets in one stream; random = many seeks / file switches.

**S3/MinIO:** Each GET is an independent HTTP operation with TTFB + body transfer. “Sequential” ≈ one large GET or one range stream; “random” ≈ many separate GETs to different objects/offsets. Parallel range GETs on one large object improve **aggregate** throughput beyond single-connection cap.

---

## User views on MinIO (author, 2026-06-03)

Primary deployment motivations (author’s framing):

1. **Local development / testing for S3-targeting software**  
   MinIO exposes an **S3-compatible API**, so applications built for AWS S3 can be tested locally without cloud dependency or cost.

2. **Privacy-sensitive enterprises**  
   Organizations that do not want data in public cloud object storage can run **MinIO on-premises or in a private cluster** to obtain S3-style durability/API while keeping data under their own trust boundary.

**Implication for Ember paper:** When citing Milvus (or other) benchmarks on “object storage,” check whether the backend is **AWS S3 vs self-hosted MinIO** — latency, TTFB, and throttle behavior differ; blog numbers are not automatically transferable to production S3.

---

## Consensus

- Do **not** add Milvus load-phase times to per-query cold search latency for tiered mode.
- “High S3 bandwidth” and “slow cold vector query” are compatible: cold paths are often **request-latency- and parallelism-limited**, not wire-bandwidth-limited.
- Slab parallel GET model is useful for **upper-bound transfer** reasoning; insufficient alone to explain cold-query tail without published Pinecone I/O traces.
- MinIO: treat as **S3 API–compatible** storage layer with distinct deployment motives (dev/test, privacy/on-prem).

## Open questions

- Pinecone: single object vs multi-file per slab; internal parallel range policy.
- Reproduce Milvus-tiered cold-query breakdown on **AWS S3** vs MinIO with controlled concurrency.
- Ember experiments: measure TTFB distribution vs aggregate MB/s for chunk/index GET mixes.

## Relation to prior records

- Continues `discussions/2026-06-02-cold-hot-query-paths-evidence.md` and `summaries/cold-hot-query-paths.md`.
- Clarifies intro-draft “multi-second” Milvus wording vs tiered **120 ms** first-access metric (different phases).
