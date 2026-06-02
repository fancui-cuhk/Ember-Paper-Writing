# Cold vs. Hot Query Paths: Pinecone, Milvus 2.x, Turbopuffer

**Purpose:** Evidence-backed execution paths for vector **search** queries, with object-storage I/O and latency notes.  
**Last updated:** 2026-06-02  
**Scope:** Public documentation and vendor blogs only. Where the vendor does not publish a number, this doc says **not published** — no invented breakdowns.

---

## Terminology (Ember paper)

| Term | Meaning here |
|------|----------------|
| **Cold query** | Required index/slab/segment data is **not** resident in the serving node’s memory or local SSD cache; the read path must pull from object storage (S3/MinIO/GCS). |
| **Hot query** | Required data is already cached locally; search runs without object-storage reads on the critical path. |
| **Cold start query** (intro draft) | Same as cold query above — **not** serverless “compute cold start” (Lambda/pod spin-up), unless noted separately. |

---

## 1. Pinecone Serverless

**Architecture sources:** [Pinecone docs — Architecture](https://docs.pinecone.io/guides/get-started/database-architecture), [How Pinecone Works](https://www.pinecone.io/how-pinecone-works/), [Serverless architecture blog](https://www.pinecone.io/blog/serverless-architecture/).

**Storage model:** Namespace → immutable **slabs** on object storage (S3). Each slab is a **set of files** (vectors, ANN index, metadata bitmaps, manifest). Writes use WAL on S3 → memtable → L0/L1/L2 compaction.

**Important official tension (do not merge):**

- [How Pinecone Works](https://www.pinecone.io/how-pinecone-works/) states slabs stay warm on executor SSD and compaction **prewarms** new L1 slabs to reduce cold starts.
- Same page + [docs](https://docs.pinecone.io/guides/get-started/database-architecture) also say executors **fetch from object storage** when a slab is not cached (first access or after eviction).
- [Serverless architecture blog](https://www.pinecone.io/blog/serverless-architecture/): cold-start latency **“a couple of seconds for most datasets”** and **“up to around 20 seconds”** for **billion-scale** queries (maximum latency for first-time / cold paths).

---

### 1.1 Hot query path (slabs cached on executor SSD)

| Step | Component | Action | Object storage (S3) | Parallelism | Latency |
|------|-----------|--------|---------------------|-------------|---------|
| 1 | API Gateway | Auth, route to regional data plane | None | — | **Not published** (per-step) |
| 2 | Query router | Validate limits; determine target slabs + memtable; scatter-gather | None (routing metadata assumed in control plane / in-memory) | Fan-out to executors in parallel | **Not published** (per-step) |
| 3 | Query executors | Search assigned **cached** slabs (algorithm per slab manifest: Ananas / PQFS / IVF) | **0 GET** on critical path | Executors parallel; slabs per executor parallel | **Not published** (vendor cites warm namespaces much faster than pods; no p50/p99 table in cited pages) |
| 4 | Memtable | Search recent writes not yet in slabs | None | Parallel with slab search | **Not published** |
| 5 | Query router | Merge top-k, dedupe, return | None | — | **Not published** |

**End-to-end hot latency:** **Not published** as an official step breakdown. Third-party benchmarks (e.g. pod vs serverless comparisons) exist but are **not** Pinecone primary sources.

---

### 1.2 Cold query path (one or more slabs not on executor cache)

| Step | Component | Action | Object storage (S3) | Parallelism | Latency |
|------|-----------|--------|---------------------|-------------|---------|
| 1–2 | API Gateway + router | Same as hot | None for routing (typical) | Same scatter-gather | **Not published** (per-step) |
| 3a | Query executors | For each **cache-miss** slab: download slab from S3, cache on local SSD, then search | **≥ 1 S3 read operation per cold slab** (slab = multiple files; **exact GET count per slab not published**) | **Across executors:** parallel. **Across cold slabs on different executors:** parallel. **Within one slab’s file set:** **not published** (likely parallel GETs, but unverified) | **Not published** per-step; **end-to-end** cold-start: see row below |
| 3b | Query executors | Slabs already cached | 0 | Parallel with 3a | Included in search phase |
| 4 | Memtable | Same as hot | None | Parallel | **Not published** |
| 5 | Router merge | Same as hot | None | — | **Not published** |

**S3 round-trip count (cold search, order-of-magnitude):**

- Let **K** = number of slabs assigned to this query that are **not** in local cache.
- **Documented lower bound:** **K** slab fetches from object storage (each fetch may be one or more S3 API calls; **not published**).
- **Upper bound (unfiltered namespace fan-out):** Official slab blog describes queries fanned to **all slabs in a namespace**; at steady state Pinecone cites **~9–20 slabs** in an example namespace ([How Pinecone Works — compaction](https://www.pinecone.io/how-pinecone-works/)); at scale, **“dozens to hundreds”** of slabs per namespace ([How Pinecone Works — scaling](https://www.pinecone.io/how-pinecone-works/)).
- **Router slab selection:** Docs say router **“identifies which slabs contain relevant data”** ([Architecture docs](https://docs.pinecone.io/guides/get-started/database-architecture)). **IVF-style subset routing vs. full fan-out is not specified** in the same doc set; treat **K** as workload-dependent.

**End-to-end cold latency (published totals only):**

| Setting | Metric | Value | Source |
|---------|--------|-------|--------|
| Most datasets | Cold-start (max) | ~few seconds | [Serverless architecture blog](https://www.pinecone.io/blog/serverless-architecture/) |
| Billion-scale | Cold-start (max) | **up to ~20 s** | Same |
| Per-step (gateway / router / S3 TTFB / ANN) | — | **Not published** | — |

**Community / support anecdotes (secondary, not spec):** Pinecone community threads report high p99 when indexes are rarely queried (cache eviction); numbers vary by client region and payload — use only as qualitative support for cache-miss behavior.

---

### 1.3 Pinecone: what is *not* in public docs

- Exact **S3 API call count** per slab (manifest + vector + index files).
- Per-step latency breakdown for cold vs hot queries.
- Whether unfiltered search always touches **all** slabs vs partition-pruned subset (official text supports both “identifies relevant slabs” and slab-architecture “fan out to all slabs” depending on page).

---

## 2. Milvus 2.x

**Primary sources:** [Tiered Storage overview](https://milvus.io/docs/tiered-storage-overview.md) (Milvus **2.6.4+**), [Tiered Storage blog / benchmarks](https://milvus.io/blog/milvus-tiered-storage-80-less-vector-search-cost-with-on-demand-hot%E2%80%93cold-data-loading.md), Milvus architecture (proxy → query coord → query node).

Milvus 2.x persists sealed segments in object storage (S3/MinIO). Paths (typical layout):

- `insert_log/{collectionID}/{partitionID}/{segmentID}/{fieldID}/{logID}`
- `stats_log/...`
- Index files referenced in segment metadata (per index build)

**Two modes matter for “cold”:**

| Mode | When collection becomes queryable | Cold behavior |
|------|-----------------------------------|---------------|
| **Full-load** (2.5 and default before tiered) | After **all** segment field data + indexes are local | Queries usually **no S3**; “cold” = **load/ recovery** time, not per-query |
| **Tiered Storage** (2.6.4+) | After **metadata-only** lazy load (+ optional warmup) | Per-query **on-demand** chunk/index fetch on cache miss |

---

### 2.1 Hot query path — Full-load mode

| Step | Component | Action | Object storage | Parallelism | Latency |
|------|-----------|--------|----------------|-------------|---------|
| 1 | Proxy | Receives search RPC | None | — | **Not published** (per-step) |
| 2 | Query Coordinator | Plan: target segments / replicas | None | — | **Not published** |
| 3 | QueryNode | ANN search on segments already in memory / mmap | **0** (steady state) | Multi-segment parallel across nodes | Blog benchmark **hot P99 ≈ 15 ms** (see §2.5 setup) |
| 4 | Proxy | Aggregate results | None | — | Included above |

**S3 round trips per hot query:** **0** (assuming load completed and no eviction).

---

### 2.2 “Cold” in Full-load — Collection load / node recovery (not a single query)

This is the dominant “multi-second” cold path in Milvus deployments **without** tiered storage.

| Phase | Action | Object storage | Parallelism | Latency (published) |
|-------|--------|----------------|-------------|-------------------|
| `Collection.load()` | Download **all** fields + indexes for **all** segments | **Many GETs** (per field binlog file, stats log, index file per segment; exact count = Σ segments × (fields + indexes)) | Milvus loads segments in parallel across cluster; within segment, multi-field parallel **not fully documented** | Blog: **~25 min** total load for **100M × 768d** HNSW dataset ([tiered blog setup](https://milvus.io/blog/milvus-tiered-storage-80-less-vector-search-cost-with-on-demand-hot%E2%80%93cold-data-loading.md)) |
| Breakdown | Data download 22 min; index loading 3 min | — | — | Same table in blog |
| After load | Queries hot | 0 per query | — | Hot P99 **15 ms** (same benchmark) |

**Per-query S3 round trips during load:** N/A (bulk prefetch). **Per-query after load:** 0.

---

### 2.3 Hot query path — Tiered Storage (2.6.4+)

| Step | Component | Action | Object storage | Parallelism | Latency |
|------|-----------|--------|----------------|-------------|---------|
| 1–2 | Proxy + QueryCoord | Same as full-load | None | — | **Not published** per-step |
| 3 | QueryNode | Search using **cached** field chunks + cached index files | **0** | Segment-level parallelism | **Hot P99 ≈ 16 ms** (vs 15 ms full-load) in blog table |
| 4 | Proxy | Return | None | — | — |

**S3 round trips:** **0**.

---

### 2.4 Cold query path — Tiered Storage (first access after cache miss)

Official mechanism ([tiered docs](https://milvus.io/docs/tiered-storage-overview.md)):

- **Fields:** partial load at **chunk** granularity (only chunks needed for the query).
- **Indexes:** loaded **per segment**, **whole file** (cannot split across chunks).

| Step | Component | Action | Object storage | Parallelism | Latency |
|------|-----------|--------|----------------|-------------|---------|
| 1–2 | Proxy + QueryCoord | Plan search | None | — | **Not published** |
| 3 | QueryNode | Cache miss on required chunks / index files | **≥ 1 GET per missing chunk** + **≥ 1 GET per missing index file** (per segment touched) | Partial load uses on-demand fetch; **thread pool width / max parallel GETs not published** in overview doc | **Cold first-access P99 ≈ 120 ms** (blog table); **cached-after-first-access P99 ≈ 18 ms** |
| 4 | QueryNode | Run ANN on loaded data | 0 additional if fully cached | — | **Not published** (split between I/O vs compute) |
| 5 | Proxy | Return | None | — | — |

**S3 round-trip count (cold tiered search, symbolic):**

For each segment **s** touched by the query:

- **C_s** = number of **field chunks** fetched (depends on filter, projection, chunking — Storage v2 uses multi-chunk columns; **exact chunk count per query not in tiered overview**).
- **I_s** = number of **index files** fetched (typically 1 per vector index per segment, if missing).

**Total ≥ Σ_s (C_s + I_s)** parallel GETs (lower bound; dedupe not specified).

Blog “good-fit” text mentions cold data **first access ~500 ms** in some cost-sensitive scenarios — **different from the 120 ms benchmark row**; treat as qualitative guidance, not the same experiment.

**Warm tiered (data previously fetched, still cached):** blog **P99 ≈ 28 ms** (“warm data searches”).

---

### 2.5 Milvus benchmark context (for any latency number above)

From [tiered blog](https://milvus.io/blog/milvus-tiered-storage-80-less-vector-search-cost-with-on-demand-hot%E2%80%93cold-data-loading.md):

- 100M vectors, 768d float32, HNSW (~430 GB index), 10 scalar fields
- 1 QueryNode: 4 vCPU, 32 GB RAM, 1 TB NVMe; MinIO backend; 10 Gbps net
- Access: 80% queries → recent 20% of data; 15% → 30–90d; 5% → older
- “Search” = vector search API, not scalar `query()`

**Not published:** per-step latency inside QueryNode (S3 TTFB vs deserialize vs HNSW).

---

### 2.6 Milvus: what is *not* in public docs

- Exact **GET count** per cold tiered query (chunk map size is segment-specific).
- Parallelism cap for partial load (e.g. vs S3 prefix limits).
- Cold query latency at **billion** scale (intro draft placeholder; no official billion cold-query table found).

---

## 3. Turbopuffer

**Primary sources:** [Launch blog](https://turbopuffer.com/blog/turbopuffer), [ANN v3 blog](https://turbopuffer.com/blog/ann-v3), [Native filtering blog](https://turbopuffer.com/blog/native-filtering), [Performance docs](https://turbopuffer.com/docs/performance).

**Architecture:** Stateless query tier over **object storage (source of truth)** + **memory/SSD cache**. Index: **SPFresh** / hierarchical centroid tree (ANN v3 adds quantization + sharding). Namespace = prefix on object storage.

---

### 3.1 Hot query path (namespace cached on memory/SSD)

| Step | Component | Action | Object storage | Parallelism | Latency |
|------|-----------|--------|----------------|-------------|---------|
| 1 | API | Receive query | None | — | **Not published** per-step |
| 2 | Query node | Traverse SPFresh tree from **cache** (centroids in DRAM/SSD; data vectors per ANN v3 layout) | **0** for hot path | Intra-node parallel search; ANN v3 uses scatter-gather on NVMe for rerank | **1M vectors, 768d, 3 GB namespace: warm p90 ≈ 10 ms** ([launch blog](https://turbopuffer.com/blog/turbopuffer)) |
| 3 | API | Return top-k | None | — | — |

**S3 round trips:** **0**.

**ANN v3 at 100B scale (different deployment assumption):** targets **p99 ≤ 200 ms** with tree sized to sit on **SSD** and centroids in DRAM — this is **not** the same as “every byte cold from S3” ([ANN v3 blog](https://turbopuffer.com/blog/ann-v3)).

---

### 3.2 Cold query path (cache empty / eviction / node replacement)

| Step | Component | Action | Object storage | Parallelism | Latency |
|------|-----------|--------|----------------|-------------|---------|
| 1 | API | Route to node (prefer cache-locality; any node can serve) | None | — | **Not published** |
| 2 | Query planner + storage engine | Bounded **round-trips** to S3: fetch tree nodes / clusters needed for ANN | **≤ 3 round-trips** for **vector search** (design target, [launch blog](https://turbopuffer.com/blog/turbopuffer)) | **Within** each round-trip: planner balances **bytes per trip** vs **trip count**; p90 object-storage latency **~250 ms for <1 MB** (same blog) | **1M × 768d: cold p90 ≈ 444 ms** (launch blog bar chart) |
| 3 | ANN + optional rerank | Search + fetch full-precision vectors for rerank (ANN v3: small fraction from NVMe/SSD) | Extra reads may be **inside** the 3-trip budget or local SSD — **not split in public doc** | Parallel NVMe scatter-gather for rerank (ANN v3) | **Not published** per-step |
| 4 | API | Return | None | — | — |

**S3 round-trip count (cold vector search):**

- **Design maximum: 3** round-trips ([launch blog](https://turbopuffer.com/blog/turbopuffer)).
- **ANN v3 cold namespace:** round-trips bounded by **tree height** (not sequential graph hops) ([ANN v3 blog](https://turbopuffer.com/blog/ann-v3)).

**Cold query with native metadata filter:** **≤ 2** round-trips in fully cold cache case ([native filtering blog](https://turbopuffer.com/blog/native-filtering)) — **different query type** than pure vector ANN.

**Other published cold numbers:**

| Scenario | Metric | Value | Source |
|----------|--------|-------|--------|
| Node dies, reload namespace | Qualitative | **~500 ms** cold query | [Launch blog](https://turbopuffer.com/blog/turbopuffer) |
| 1M docs BM25 | cold p90 | **285 ms** | Launch blog |
| Billion scale | End-to-end cold from S3 only | **Not published** as a single official table (intro draft “higher at billion” is qualitative; ANN v3 assumes SSD residency at scale) |

**Per-step breakdown (S3 RTT #1 / #2 / #3 / ANN compute):** **Not published**.

---

### 3.3 Turbopuffer: what is *not* in public docs

- Exact bytes fetched per round-trip for a given `top_k` / probe setting.
- Cold latency table at **100B vectors** with **no** local SSD cache (ANN v3 post assumes SSD for data plane).

---

## 4. Side-by-side comparison

| Dimension | Pinecone Serverless | Milvus 2.x (tiered) | Milvus 2.x (full-load) | Turbopuffer |
|-----------|-------------------|---------------------|------------------------|-------------|
| **Hot path S3 reads** | 0 | 0 | 0 | 0 |
| **Cold path S3 model** | Per **slab** fetch on cache miss; **K** fetches | Per **chunk + whole index file** | Bulk at **load**; 0 per query after | **≤ 3** RTTs (vector ANN design target) |
| **Cold parallelism** | Executor / slab parallel (**details N/P**) | Partial load (**cap N/P**) | Segment parallel load | Multi-trip planner; trips sequential, bytes parallel inside trip (**partially described**) |
| **Published cold E2E** | Up to **~20 s** (billion); few s typical | **120 ms** P99 first-access (100M bench) | **25 min** load; then 15 ms hot | **444 ms** p90 (1M, 768d, cold) |
| **Published hot E2E** | **N/P** (step breakdown) | **16 ms** P99 hot (same bench) | **15 ms** P99 | **10 ms** p90 warm (1M) |

**Legend:** N/P = not published.

---

## 5. Evidence gaps (for Ember experiments)

1. **Pinecone:** Measure **K** cold slabs and S3 GET count per query under controlled namespace size and filter/no-filter.
2. **Milvus tiered:** Measure **Σ(C_s + I_s)** for cold queries on Storage v2 collections.
3. **All three:** Microbenchmark **TTFB vs throughput** per parallel GET batch (validate “parallel slabs ≠ one big read” without inventing numbers).
4. **Billion-scale cold query:** Only Pinecone publishes a **~20 s** order-of-magnitude cap; Milvus/Turbopuffer need Ember-measured conditions.

---

## 6. References

1. Pinecone — Database architecture: https://docs.pinecone.io/guides/get-started/database-architecture  
2. Pinecone — How Pinecone Works: https://www.pinecone.io/how-pinecone-works/  
3. Pinecone — Serverless architecture (cold-start latency quote): https://www.pinecone.io/blog/serverless-architecture/  
4. Milvus — Tiered Storage overview (2.6.4+): https://milvus.io/docs/tiered-storage-overview.md  
5. Milvus — Tiered Storage blog & benchmarks: https://milvus.io/blog/milvus-tiered-storage-80-less-vector-search-cost-with-on-demand-hot%E2%80%93cold-data-loading.md  
6. Turbopuffer — Launch blog (3 RTTs, 444 ms / 10 ms bars): https://turbopuffer.com/blog/turbopuffer  
7. Turbopuffer — ANN v3 (tree height bounds RTTs, 200 ms p99 at 100B with SSD): https://turbopuffer.com/blog/ann-v3  
8. Turbopuffer — Native filtering (≤2 RTTs cold): https://turbopuffer.com/blog/native-filtering  

---

## Source discussions

- `discussions/2026-05-30-root-cause-analysis.md`
- `discussions/2026-06-02-s3-vectors-positioning-and-open-questions.md`
- Session 2026-06-02: user request for evidence-backed cold/hot paths and S3 round-trip accounting.
