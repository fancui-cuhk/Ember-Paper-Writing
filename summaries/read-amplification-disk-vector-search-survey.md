# Read Amplification in Disk-Based Vector Search: Survey

## Overview

**Read amplification** here means: the system **reads (or transfers) more bytes than the query logically needs** to produce its approximate nearest-neighbor result. The wasted fraction grows when storage access granularity is coarser than the index’s logical working set.

This note surveys read amplification in **disk-resident** and **disaggregated-storage** vector search: **sources** (§3), **fix approaches** (§4), and a **detailed paper catalog** (§5: how each work defines and handles RA). It complements:

- [`disk-based-ann-survey.md`](disk-based-ann-survey.md) — disk ANN algorithms and I/O profiles
- [`partition-sharding-vector-search-survey.md`](partition-sharding-vector-search-survey.md) — where data is placed
- [`cloud-hosted-vs-cloud-native.md`](cloud-hosted-vs-cloud-native.md) — cold query + amplification trade-offs on object storage

**Local PDFs:** §5 papers with downloaded copies live in [`related-work/pdfs/read-amp-related/`](../related-work/pdfs/read-amp-related/) (see [`manifest.tsv`](../related-work/pdfs/manifest.tsv)). Entries mark **Local PDF** path or `NOT_DOWNLOADED` if fetch failed.

---

## 1. Definition and metrics

### 1.1 Working set vs bytes moved

For one query, define:

| Term | Meaning |
|------|---------|
| **Useful bytes** | Data actually consulted to score candidates and return top-k (probed IVF lists, visited graph nodes, reranked vectors) |
| **Bytes read** | What the OS/storage returns (always ≥ useful, often rounded up to page/object boundaries) |
| **Read amplification (RA)** | `bytes_read / useful_bytes` (sometimes reported as **I/O count × page_size / useful_bytes**) |

Amplification can happen at **multiple layers** in one stack:

```
Query needs 3 IVF lists (~300 KB useful)
  → 3 × 4 KB SSD pages each list touches     (page RA)
  → 1 × 512 MB Milvus segment loaded from S3  (segment RA)
  → 1 × 2 GB index shard cached wholesale     (shard RA)
```

Each layer can dominate depending on deployment.

### 1.2 Related terms (do not conflate)

| Term | Distinction |
|------|-------------|
| **Read amplification** | Too many **bytes** per query |
| **Request amplification** | Too many **I/O operations / GETs** (each with fixed latency) — see §7 |
| **I/O count amplification** | Too many **random reads** (graph hops) even if each read is small |
| **Write amplification** | SSD FTL / LSM compaction — largely out of scope unless index updates dominate |

---

## 2. Why in-RAM search is different (mostly)

Your presumption is **directionally correct**: DRAM is **byte-addressable** through the cache hierarchy, so fetching one graph neighbor or one vector does not force a minimum 4 KB transfer.

| Property | DRAM / in-memory ANN | Disk / object storage ANN |
|----------|----------------------|---------------------------|
| Minimum fetch unit | Cache line (~64 B); vector/scalar access | Page/sector (**4 KB** NVMe; **object** on S3) |
| Random access cost | ~100 ns | ~70–100 µs (NVMe); ~100–200 ms first byte (S3 GET) |
| Prefetch | Hardware + software prefetch helps irregular graph walks | Prefetch limited when next hop is **data-dependent** |
| “Load index then query” | Whole graph often resident — **no cold-load RA across network** | Cold path must move index pieces over network/disk |

**Caveats — RAM is not zero amplification:**

1. **Cache-line waste:** Reading one 8-byte pointer may pull a 64 B cache line; negligible vs disk pages.
2. **Whole-index residency:** In-memory HNSW still **loads the full graph into RAM** even though each query visits ~0.01–0.1% of nodes — that is **capacity** waste, not per-query I/O RA.
3. **PQ / compressed navigation:** DiskANN-style systems **deliberately** keep compressed vectors in RAM and read full vectors from disk only for rerank — trading **memory for less disk RA**.

**Bottom line:** Per-query **I/O amplification** is the defining pain of disk/disaggregated vector search; in-RAM search shifts the problem to **memory footprint**, not byte-granularity waste on the critical path.

---

## 3. Taxonomy: sources of read amplification

### A. Storage page granularity (4 KB boundary)

**Mechanism:** SSD and OS read in **4 KB pages**. If a graph node is 200 B but shares a page with unrelated nodes, or if one node spans two pages, one logical fetch wastes sibling bytes or triggers extra pages.

**Index families affected:** Graph (DiskANN, HNSW-on-disk), disk layouts for IVF metadata.

| Mitigation pattern | Papers |
|--------------------|--------|
| Pad node to page / sector size | DiskANN |
| One node = one page | PageANN |
| Co-locate co-accessed neighbors in same page/block | Starling, OctopusANN, NaviX, Gorgeous, VeloANN |

### B. Graph neighbor scatter (structural)

**Mechanism:** Graph traversal follows **data-dependent** edges. Neighbors of a node are rarely contiguous on disk → each hop may read a new page; **sequential dependency** prevents batching until PipeANN-style speculation.

**Typical RA:** 50–200 hops × 4 KB/page; PQ in RAM reduces need to read full vector every hop but **graph topology** still drives page reads.

**Index families:** Vamana/DiskANN, disk HNSW, FreshDiskANN.

### C. IVF posting-list granularity

**Mechanism:** After identifying **nprobe** centroids, the system loads **entire posting lists** (often tens of KB–MB each). Useful data may be a subset (especially with PQ + rerank on a fraction of candidates).

**RA drivers:**

- List larger than needed for target recall (unbalanced k-means clusters)
- Loading **full-precision** vectors when PQ scoring would suffice for first stage
- Reading whole list when **query-aware pruning** could skip tail of list (SPANN dynamic nprobe)

**Mitigations:** SPANN HBC (bounded list size 12–48 KB), SPFresh balanced postings, SPANN query-aware list pruning, RAIRS shared-cell layout (reduces redundant distance work, not always bytes read).

### D. Segment / shard / monolithic object (systems layer)

**Mechanism:** Cloud-native DBs store **large sealed units** (Milvus segment ~512 MB, Pinecone slab, whole HNSW shard). Routing may identify the right segment, but **cold load pulls the entire segment** even when only **nprobe** IVF clusters inside are needed.

**This is the Ember cold-path argument:** segmentation **reduces** RA vs one global file but does **not** eliminate it — granularity mismatch moves from “whole index” to “whole segment.”

| Granularity | Typical cold behavior | RA character |
|-------------|----------------------|--------------|
| Whole index on S3 | Load 10–100 GB | Extreme |
| Segment / shard (512 MB–1 GB) | Load one segment | High |
| Per-cluster object (Turbopuffer) | nprobe GETs | Low bytes, high **request** count |
| Per-block / per-list (ideal) | Fetch only probed lists | Low RA if object size ≈ list size |

### E. Full-precision rerank on disk

**Mechanism:** Navigation uses PQ/compressed vectors in RAM; **rerank** reads full vectors from SSD. If rerank set is large or vectors are large, bytes read ≫ strictly necessary for top-k.

**Mitigation:** DiskANN fetches full vector only for candidates in beam; SPANN/IVF-PQ two-stage; HAKES filter/refine split.

### F. Speculative / pipelined I/O (intentional over-read)

**Mechanism:** PipeANN issues **speculative** reads before the search frontier is known → may read pages never used.

**Trade-off:** Lower latency vs higher **bandwidth** RA under concurrency.

---

## 4. Approaches to reduce read amplification (solution catalog)

This section summarizes **fix strategies** at a high level. §5 catalogs individual papers under each strategy. Every approach targets one or more RA sources from §3 (A–F).

### 4.1 Six approach families

| Family | Idea | Primary §3 targets | Trade-off |
|--------|------|-------------------|-----------|
| **Hide** | Keep hot data in RAM so cold I/O is rare | All | Memory cost; cache miss still pays full RA |
| **Layout** | Align physical storage to logical fetch units | A, B, C | Build-time / compaction complexity |
| **Algorithmic** | Read less *useful* data per query (prune, compress, two-stage) | C, E | Recall / accuracy tuning |
| **Systems** | Right-size cloud objects and segments | D | Bytes RA ↔ request amplification |
| **Latency masking** | Overlap or speculate on I/O | B, F | May **increase** bandwidth RA |
| **Bypass** | Avoid disk on the query path | All | Capacity / cost — shifts problem to RAM |

Most production stacks **compose** several families (e.g., PQ in RAM + sector-aligned graph + segment cache).

### 4.2 Catalog of fix approaches

#### (1) Cache / keep hot in memory

**Mechanism:** Centroids, navigation graphs, PQ codes, or whole segments resident in DRAM/SSD cache; disk/object store is the miss path.

**Examples:** DiskANN PQ in RAM; LindormVector LDServer cache; Milvus query-node segment cache; HAKES filter stage in memory.

**Targets:** All sources — amortizes cold RA but does not change layout on miss.

**Limit:** Cold query, scale-out, or memory pressure → RA returns.

---

#### (2) Align storage unit to logical fetch

**Mechanism:** Make the **minimum read size** match what the algorithm needs: one node per page, sector padding, bounded posting lists.

**Examples:** PageANN (1 node = 1 page); DiskANN sector alignment; SPANN/SPFresh bounded lists (12–48 KB); B+ANN semantic blocks on disk pages.

**Targets:** **A** (page waste), **C** (oversized lists).

**Limit:** Padding and fixed caps waste space inside the unit; graph hops still multiply units.

---

#### (3) Co-locate co-accessed data

**Mechanism:** Place neighbors, bucket contents, or probed clusters **physically adjacent** so one read satisfies multiple logical accesses.

**Examples:** Starling block shuffling; OctopusANN joint layout; SK-LSH sorted buckets; Gorgeous neighbor-packed disk blocks; VeloANN affinity-based page placement; B+ANN B+-tree semantic blocks; Turbopuffer per-cluster objects (when nprobe clusters are colocated — see partition survey §4).

**Targets:** **A**, **B**, **C**; also reduces **request count** when batching.

**Limit:** Reordering cost at build/update time; colocation policy must match query routing.

---

#### (4) Prune before or during read

**Mechanism:** Skip lists, tail of lists, or graph branches whose contribution to top-k is below a threshold.

**Examples:** SPANN query-aware dynamic nprobe; incremental LSH probing (I-LSH); graph beam / early termination.

**Targets:** **C** (fewer lists / shorter effective lists), **B** (fewer hops).

**Limit:** Aggressive pruning hurts recall; needs distance models or calibration.

---

#### (5) Compress representations on disk

**Mechanism:** Store PQ / scalar-quantized codes in postings or use asymmetric distance so **useful bytes per candidate** shrink even when whole list is read.

**Examples:** IVF-PQ (Jégou et al.); DiskANN compressed neighbors in RAM; SPANN two-stage scoring.

**Targets:** **C**, **E** — lowers *useful* denominator or makes full list read cheaper.

**Limit:** Extra rerank stage may reintroduce full-vector reads (**E**).

---

#### (6) Two-stage filter → refine

**Mechanism:** Cheap filter (compressed IVF, PQ graph navigation) in fast memory; **full-precision vectors** fetched only for a short candidate list.

**Examples:** HAKES (filter workers + refine workers); DiskANN (PQ navigate, SSD rerank); SPANN (centroid + SPTAG in RAM, disk postings).

**Targets:** **C**, **E** — separates “bytes for navigation” from “bytes for final answer.”

**Limit:** Refine set size and vector dimension dominate tail RA at high recall.

---

#### (7) Choose index family for the storage tier

**Mechanism:** Pick graph vs IVF vs LSH based on **RA shape** on the target media (dependent page hops vs parallel list GETs).

**Examples:** IVF favored on object storage (independent list fetch); graph optimized for local NVMe (PageANN, PipeANN); DSANN aggregation graph for slow DFS.

**Targets:** **B** vs **C** trade-off; not a single knob but a **design fork**.

**Profile:** Graph → `hops × page_size`; IVF → `nprobe × list_size` (see §6).

---

#### (8) Pipeline / speculative I/O (latency, not RA)

**Mechanism:** Issue reads before the frontier is known; overlap I/O with compute (io_uring, async DFS).

**Examples:** PipeANN speculative pages; DSANN async overlap on DFS; LAANN priority I/O–CPU pipeline; Gorgeous async block prefetch.

**Targets:** **B** (latency of sequential hops), **F** (intentional over-read).

**Limit:** **Increases bandwidth RA** under concurrency — a **latency masking** tactic, not an RA reducer.

---

#### (9) Right-size systems fetch units (cloud / disaggregation)

**Mechanism:** Choose segment, slab, shard, or per-list object granularity so cold load ≈ probed working set.

**Examples:** Turbopuffer per-cluster objects (low bytes, high GET count); Milvus ~512 MB segments (low GET count, high bytes); Pinecone slabs (middle ground).

**Targets:** **D** — and tightly coupled to **request amplification** (§7).

**Limit:** No free lunch: smaller objects reduce bytes RA; larger objects reduce request count.

---

#### (10) Batch / amortize cold loads (throughput, not per-query RA)

**Mechanism:** Parallel bulk GETs, whole-segment preload, scan-style I/O — good for **analytics**, poor for **point ANN**.

**Examples:** AnyBlob (OLAP contrast — high RA acceptable); segment seal + bulk upload in Milvus.

**Targets:** **D** at **job** granularity, not single-query latency.

**Limit:** Random ANN queries cannot rely on bulk read without accepting massive RA (see `discussions/2026-06-03-s3-high-bandwidth-vs-vector-search.md`).

---

#### (11) Bypass disk entirely (in-RAM / disaggregated memory)

**Mechanism:** Full graph or index in memory; no per-query storage I/O.

**Examples:** In-memory HNSW, ScaNN, FAISS GPU — outside disk-ANN catalog but the **baseline** Ember contrasts against.

**Targets:** All per-query I/O RA — replaces with **capacity** cost.

---

### 4.3 Mapping: approach → RA source

| §3 source | Most relevant approaches |
|-----------|-------------------------|
| **A** Page granularity | (2) align unit, (3) co-locate |
| **B** Graph scatter | (3) co-locate, (4) prune, (7) index choice, (8) pipeline |
| **C** IVF list size | (2) bounded lists, (4) prune, (5) compress, (6) two-stage |
| **D** Segment / object | (9) right-size units, (1) cache, (10) batch (throughput only) |
| **E** Full-precision rerank | (5) compress, (6) two-stage, (4) smaller refine set |
| **F** Speculative I/O | (8) pipeline — trades RA for latency |

### 4.4 Gaps and composite design (Ember sketch)

**Under-served in literature:**

- Normalized **RA ratio** across families on identical hardware (§8).
- **Segment-internal** fraction touched per query (Milvus-style **D**).
- Joint optimization of **bytes RA + request amplification + tail latency** — papers usually optimize one axis.

**Composite stack** implied by the catalog (not one paper):

1. **(9)** Storage objects sized to posting / page-node (~tens of KB), not 512 MB segments.
2. **(3)** Colocate clusters likely probed together (partition survey §4).
3. **(6)+(5)** Filter in memory, refine from disk only for top candidates.
4. **(1)** Cache hot objects at query nodes — but define success on **cold p99**, not warm mean.

§5 lists papers implementing slices of this stack; §9 states open evaluation questions.

---

## 5. Disk-based paper catalog — RA definition and handling

This section lists **disk-resident** and **disaggregated-storage** vector-search papers in the repo’s read-amp PDF set. For each paper we record:

1. **How RA is defined** (explicit term or proxy metric)
2. **Root cause** targeted (§3 letter)
3. **How RA is handled** (mechanism)
4. **Evaluation metric** for I/O waste

**Entry template:** Category · §3 source · Local PDF · Definition · Problem · Solution · Metrics.

**Quick index**

| Paper | Index | Defines RA as… | Primary handle |
|-------|-------|----------------|----------------|
| DiskANN | Graph | Random SSD reads × sector granularity | Sector batching + PQ-in-RAM |
| PageANN | Graph | Page misalignment; wasted bytes per 4 KB read | 1 node = 1 page |
| Starling | Graph | Low **overlap ratio** OR(G) in disk blocks | Block shuffling (BNP/BNF/BNS) |
| OctopusANN | Graph | Page-level OR + I/O time fraction | Joint mem/disk/search design space |
| NaviX | Graph | Page-granularity waste in disk HNSW | Graph/block partition on DB pages |
| PipeANN | Graph | Sequential I/O underutilization; speculative waste | Async pipelined I/O (io_uring) |
| Gorgeous | Graph | Disk I/O count; poor neighbor locality | Neighbor-packed disk blocks |
| LAANN | Graph | **I/O read amplification**; excess I/O ops | Look-ahead + I/O-aware search |
| VeloANN | Graph | **Read amplification** / over-fetch per page | Affinity page placement + buffer pool |
| B+ANN | Graph/disk | Block-level locality vs random hops | k-means++ blocks in B+ tree pages |
| SPANN | IVF | **Disk-access count**; oversized postings | HBC bounded lists + query-aware prune |
| SPFresh | IVF | Unbalanced posting I/O on update | LIRE split/rebalance (SPANN lineage) |
| IVF-PQ | IVF | Bytes per list candidate | PQ codes in postings |
| RAIRS | IVF | **Redundant list I/O** from multi-assignment | SEIL shared-cell layout |
| I-LSH / SK-LSH | LSH | Pages per bucket probe | Incremental probe + sorted buckets |
| HAKES | Hybrid | Full-vector bytes on refine path | Filter (RAM) → refine (disk) |
| LindormVector | Systems/IVF | DFS read per posting on cache miss | LDServer cache + IVFPQ on DFS |
| DSANN | Graph/DFS | Blocking DFS reads; hop amplification | PAG + async overlap |
| Milvus | Systems | Segment-sized cold load vs nprobe need | Segment cache (does not fix unit size) |
| Pinecone / Turbopuffer | Systems | Slab/object granularity vs probed subset | Slabs vs per-cluster objects |
| IISWC 2025 | Measurement | Per-query read bandwidth | DiskANN I/O profiling |

---

### 5.1 Graph indexes on SSD / local disk

#### DiskANN (NeurIPS 2019)

- **Category:** Graph · **§3:** A, B, E · **PDF:** [`diskann.pdf`](../related-work/pdfs/read-amp-related/diskann.pdf)
- **RA definition:** Does not use the phrase “read amplification.” Defines the bottleneck as **too many random SSD reads per query** (hundreds if naïve in-memory graphs are placed on disk) and **sector-granularity waste** — fetching one graph neighborhood may require reading a full **sector-aligned** slot while only part is useful.
- **Root cause:** Graph hop chain (**B**) + node size misaligned with SSD read unit (**A**) + full-precision rerank from SSD (**E**).
- **How handled:**
  - **Vamana** graph with bounded degree to cap hop count.
  - **Sector-aligned layout:** batch up to *W* consecutive neighbors per sector read (amortize random-read latency; accept padding waste if *W* is too large).
  - **Two-tier storage:** PQ-compressed vectors in **DRAM** for navigation distances; **full vectors + graph topology on SSD**, fetched only for candidates in the beam.
- **Metrics:** Random reads/query, latency vs in-memory HNSW; follow-on work reports **70–90%** query time in I/O.

#### PageANN (arXiv 2025)

- **Category:** Graph · **§3:** A (primary) · **PDF:** [`pageann.pdf`](../related-work/pdfs/read-amp-related/pageann.pdf)
- **RA definition:** Explicitly names **misalignment with storage I/O granularity** — SSD must read whole **4 KB pages** while graph nodes are smaller and scattered, so each hop wastes sibling bytes (**page-level read amplification**).
- **Root cause:** Node size ≠ page size; long I/O traversal paths from scattered layout (**A**, **B**).
- **How handled:**
  - **Page-node graph:** cluster similar vectors; map **one logical page-node ↔ one physical SSD page**.
  - Co-designed **memory navigation graph + disk page graph**; representative vectors form inter-page edges.
  - Merging within page packs topology so one page read is fully utilized.
- **Metrics:** Pages read per query; claims **elimination of page-level RA** vs DiskANN; **1.85×–10.83×** throughput vs SOTA disk ANN.

#### Starling (SIGMOD 2024)

- **Category:** Graph (Milvus **segment** setting) · **§3:** A, B · **PDF:** [`starling.pdf`](../related-work/pdfs/read-amp-related/starling.pdf)
- **RA definition:** **Overlap ratio** OR(G) — for vertex *u*, fraction of neighbors in the same disk **block** as *u*; global OR(G) averages over vertices. Low OR ⇒ loading a block pulls mostly **unrelated** vertices (**vertex utilization ratio**).
- **Root cause:** DiskANN-style ID-contiguous layout scatters graph neighbors across blocks (**B**); each block read amplifies bytes (**A**).
- **How handled:**
  - In-memory **navigation graph** + **reordered disk graph**.
  - **Block shuffling** (NP-hard): maximize OR(G) via heuristics **BNP** (padding neighbors into blocks), **BNF** (assign to block with most neighbors), **BNS** (swap low-overlap vertices).
  - **Block-aware search** to exploit colocated neighbors.
- **Metrics:** OR(G), block reads vs hops, I/O time fraction; **43.9×** throughput vs baselines on segment budget.

#### OctopusANN (PVLDB 2026)

- **Category:** Graph · **§3:** A, B · **PDF:** [`octopusann.pdf`](../related-work/pdfs/read-amp-related/octopusann.pdf)
- **RA definition:** Same **page-level overlap ratio** as Starling (Eq. 1 in paper); also reports **I/O time as 70–90%** of query latency. Design-space paper — RA is **bytes wasted per page read × pages touched**.
- **Root cause:** Poor data locality on disk; fragmented optimizations in prior work.
- **How handled:** **Three-axis taxonomy** — memory layout, disk layout, search algorithm — composed systematically (page shuffle, mem graph, beam policies, etc.). Page shuffle = Starling-style reordering to raise OR(G).
- **Metrics:** I/O time %, QPS at matched Recall@10; **4.1–37.9%** over Starling, **87.5–149.5%** over DiskANN.

#### NaviX (PVLDB 2025)

- **Category:** Graph (disk HNSW in graph DBMS) · **§3:** A · **PDF:** [`navix.pdf`](../related-work/pdfs/read-amp-related/navix.pdf)
- **RA definition:** Implicit — **page-granularity** reads in filtered graph+vector queries waste bytes when graph partitions span DB pages.
- **Root cause:** Disk-resident HNSW layout not aligned with storage/page boundaries (**A**).
- **How handled:** **Graph/block partitioning** co-designed with DB page layout so filtered traversals stay page-local.
- **Metrics:** Query latency under hybrid vector+graph filters (paper-specific workloads).

#### PipeANN (OSDI 2025)

- **Category:** Graph · **§3:** B, **F** · **PDF:** [`pipeann.pdf`](../related-work/pdfs/read-amp-related/pipeann.pdf)
- **RA definition:** Does not optimize RA ratio directly. Identifies **serialized I/O+compute** and **underutilized I/O pipeline**; speculative reads may fetch pages **never on the final frontier** (**I/O over-fetching**).
- **Root cause:** Sequential dependency of graph hops (**B**); latency masking can **increase bandwidth RA** (**F**).
- **How handled:** **PipeSearch** — io_uring async pipeline; prefetch next-hop pages while computing current frontier; dynamic beam width; overlap compute with I/O.
- **Metrics:** Latency vs in-memory Vamana (**1.14×–2.02×**); **35%** of DiskANN latency; tracks wasted speculative I/O qualitatively.

#### Gorgeous (arXiv 2025)

- **Category:** Graph · **§3:** A, B · **PDF:** [`gorgeous.pdf`](../related-work/pdfs/read-amp-related/gorgeous.pdf)
- **RA definition:** **Disk I/O count** and poor locality when co-located **vectors** alone fail at high dimension (Starling overlap drops). Notes **disk space amplification** from replicated adjacency in blocks.
- **Root cause:** Vector co-location in 4 KB blocks insufficient when nodes are large (**A**); graph structure still scattered (**B**).
- **How handled:**
  - **Graph-prioritized memory cache** (adjacency lists over full vectors).
  - **Graph-replicated disk blocks** with **neighbor packing** — store neighbors’ **adjacency lists** inside a node’s disk block so later hops avoid extra reads.
  - Async prefetch of packed blocks.
- **Metrics:** Average **disk I/Os per query** (~**39%** reduction vs DiskANN/Starling at matched recall).

#### LAANN (arXiv 2026)

- **Category:** Graph · **§3:** B, F · **PDF:** [`laann.pdf`](../related-work/pdfs/read-amp-related/laann.pdf)
- **RA definition:** Explicitly targets **I/O read amplification** — issuing disk reads whose results are not used because search issues I/O **before** knowing which nodes matter.
- **Root cause:** Decoupled CPU/I/O phases in best-first search (**B**, **F**).
- **How handled:**
  - **Look-ahead search** — phase-aware balance of I/O reduction vs timely issuance.
  - **Priority I/O–CPU pipeline** — use I/O wait time for cheap in-memory work.
  - Lightweight **in-memory centroid graph** over page nodes to guide which pages to read.
- **Metrics:** **I/O operations per query** (**1.59×–6.34×** fewer vs SOTA at Recall@10=0.9); latency decomposition (I/O-only vs overlap vs CPU).

#### VeloANN (arXiv 2026)

- **Category:** Graph · **§3:** A, B · **PDF:** [`veloann.pdf`](../related-work/pdfs/read-amp-related/veloann.pdf)
- **RA definition:** Explicit **read amplification** — “storage stalls” from loading a **whole page** but using one record; **over-fetching** under memory pressure causes page swapping.
- **Root cause:** Poor traversal locality; synchronous I/O (**A**, **B**).
- **How handled:**
  - **Affinity-based page placement** — co-locate related vectors in same page.
  - **Record-level buffer pool** — pin hot neighbor records.
  - Hierarchical compression + coroutine async runtime; beam-aware search prefers cached records.
- **Metrics:** QPS/latency vs PipeANN/DiskANN; storage stall reduction (paper benchmarks).

#### B+ANN (arXiv 2025)

- **Category:** Graph / B+-tree on disk · **§3:** A, B · **PDF:** [`b-plus-ann.pdf`](../related-work/pdfs/read-amp-related/b-plus-ann.pdf)
- **RA definition:** Implicit — random graph hops cause **block-granularity over-read**; reports **19% fewer cache misses** (CPU hierarchy) vs HNSW; disk path amortizes multiple vectors per block read.
- **Root cause:** Unstructured graph layout on disk (**B**).
- **How handled:** **k-means++ semantic blocks** stored in **B+ tree pages**; hybrid block-level + edge-level traversal; SIMD distance within a block (one I/O → many candidates).
- **Metrics:** Cache misses, QPS/recall vs HNSW/DiskANN-class baselines.

---

### 5.2 IVF / LSH on disk

#### SPANN (NeurIPS 2021)

- **Category:** IVF · **§3:** C · **PDF:** [`spann.pdf`](../related-work/pdfs/read-amp-related/spann.pdf)
- **RA definition:** **Number of disk accesses** and **bytes per posting list read** — large, unbalanced IVF lists force reading data not needed for top-*k*; centroids stay in RAM, **large postings on disk**.
- **Root cause:** Oversized / unbalanced posting lists (**C**); reading whole lists when only head matters.
- **How handled:**
  - **Hierarchical Balanced Clustering (HBC)** — cap posting size (**12–48 KB**).
  - **Closure expansion** for recall at cluster boundaries.
  - **Query-aware dynamic pruning** — skip lists/posting tails by distance threshold (reduces effective nprobe).
  - Centroids + head index (SPTAG) in memory; disk = postings only.
- **Metrics:** Disk-access count, posting bytes read, QPS vs DiskANN at same DRAM budget; **2×** faster at 90% recall on billion-scale sets.

#### SPFresh (SOSP 2023)

- **Category:** IVF (mutable) · **§3:** C · **PDF:** [`spfresh.pdf`](../related-work/pdfs/read-amp-related/spfresh.pdf)
- **RA definition:** Same SPANN framing — **posting length imbalance** ⇒ unpredictable **per-probe I/O** and tail latency on disk.
- **Root cause:** Updates break balanced postings (**C**).
- **How handled:** SPANN-style **small balanced postings** + **LIRE** split/rebalance on ingest; keeps list-size invariants so each disk read stays bounded.
- **Metrics:** Update throughput, query latency tail, recall under churn.

#### IVF-PQ (Jégou et al., TPAMI 2011)

- **Category:** IVF · **§3:** C, E · **PDF:** [`product-quantization-for-nearest-neighbor-search.pdf`](../related-work/pdfs/read-amp-related/product-quantization-for-nearest-neighbor-search.pdf)
- **RA definition:** Implicit — inverted lists store **full vectors** unless compressed; RA = `list_bytes / useful_candidate_bytes`.
- **Root cause:** Full-precision postings (**C**, **E**).
- **How handled:** **PQ codes in postings** + asymmetric distance computation — same list read, fewer bytes per candidate scored.
- **Metrics:** Recall–bits tradeoff; distance computation cost (typically in-RAM in original paper; disk IVF stacks inherit same compression lever).

#### RAIRS (SIGMOD 2026)

- **Category:** IVF · **§3:** C · **PDF:** [`rairs.pdf`](../related-work/pdfs/read-amp-related/rairs.pdf)
- **RA definition:** **Redundant assignment** (vector in multiple lists) causes the **same vector to be read/scored multiple times** across lists — multiplicative I/O and compute waste.
- **Root cause:** Duplicate vectors across IVF lists from redundancy-for-recall (**C**).
- **How handled:** **SEIL** list layout — shared cells across redundant assignments so one physical store serves multiple list memberships; optimized assignment policy.
- **Metrics:** Distance computations avoided, QPS/recall vs naive redundant IVF.

#### I-LSH (ICDE 2019)

- **Category:** LSH / external memory · **§3:** A, C · **PDF:** [`i-lsh.pdf`](../related-work/pdfs/read-amp-related/i-lsh.pdf)
- **RA definition:** **Pages read per query** in external memory — probing many buckets touches many disk pages.
- **Root cause:** LSH multi-probe without early stop (**C**).
- **How handled:** **Incremental probing** — add probes until recall target; stop early to reduce pages read.
- **Metrics:** I/O complexity / pages accessed vs fixed probe count.

#### SK-LSH (PVLDB 2014)

- **Category:** LSH · **§3:** A, C · **PDF:** [`sk-lsh.pdf`](../related-work/pdfs/read-amp-related/sk-lsh.pdf)
- **RA definition:** External-memory **bucket access cost** — random bucket layout ⇒ many seeks/pages.
- **Root cause:** Hash buckets not sequential on disk (**A**).
- **How handled:** **Sorted bucket layout** on disk for sequential reads within a bucket; reduces pages per bucket probe.
- **Metrics:** Disk I/Os vs baseline LSH on disk-resident data.

#### Learned-function lists (ICDE 2020)

- **Category:** LSH/IVF hybrid · **§3:** C · **PDF:** [`i-o-efficient-approximate-nearest-neighbour-search.pdf`](../related-work/pdfs/read-amp-related/i-o-efficient-approximate-nearest-neighbour-search.pdf)
- **RA definition:** **I/O-efficient** ANN — minimize block/page transfers for list probes.
- **Root cause:** Poor on-disk ordering of candidates (**C**).
- **How handled:** **Learned orderings** of posting/bucket contents for sequential I/O during probe.
- **Metrics:** I/O cost vs recall on disk-resident indexes.

---

### 5.3 Hybrid memory–disk and distributed file systems

#### HAKES (PVLDB 2025)

- **Category:** IVF two-stage · **§3:** C, E · **PDF:** [`hakes.pdf`](../related-work/pdfs/read-amp-related/hakes.pdf)
- **RA definition:** **Full-precision vector bytes** fetched from slow tier during refine — navigation should not pull entire raw embeddings.
- **Root cause:** Single-stage IVF reads full vectors from disk (**C**, **E**).
- **How handled:** **Filter workers** — compressed IVF in memory; **Refine workers** — full vectors on disk/remote; only short candidate list refined.
- **Metrics:** End-to-end latency, bytes moved to refine tier, recall@k.

#### LindormVector (SIGMOD 2026 Industry)

- **Category:** IVFPQ on DFS · **§3:** D, C · **PDF:** [`lindorm-vector.pdf`](../related-work/pdfs/read-amp-related/lindorm-vector.pdf) · notes: [`related-work/lindorm-vector-sigmod2026.md`](../related-work/lindorm-vector-sigmod2026.md)
- **RA definition:** Each probed cluster ⇒ **DFS read** on cache miss; RA at **posting-list** granularity unless LDServer cache hits.
- **Root cause:** Disaggregated storage + IVFPQ lists not co-located with compute (**D**, **C**).
- **How handled:** Centroids/HNSW in memory; **posting lists on LindormDFS**; **LDServer** compute-side cache — hides RA on warm path, not cold.
- **Metrics:** Hybrid query latency (vector + SQL); cache hit sensitivity (reader analysis: cold path still RA-limited).

#### DSANN (arXiv 2025)

- **Category:** Graph on **distributed file system** · **§3:** B, D · **PDF:** [`dsann.pdf`](../related-work/pdfs/read-amp-related/dsann.pdf)
- **RA definition:** **DFS read latency** (0.1–10 ms vs local NVMe) amplifies cost of each hop/list read; blocking reads of whole graph segments or lists.
- **Root cause:** DiskANN/SPANN designs assume fast local SSD; hop chains on DFS (**B**, **D**).
- **How handled:** **Point Aggregation Graph (PAG)** — sample aggregation points in memory for most traversal; **async overlap** for residual vectors on DFS; reduces blocking round-trips.
- **Metrics:** QPS/latency on Pangu-class DFS vs replicated local DiskANN/SPANN.

---

### 5.4 Systems layer — segment / object granularity

#### Milvus 2.x (SIGMOD 2021)

- **Category:** Systems · **§3:** D · **PDF:** [`milvus.pdf`](../related-work/pdfs/read-amp-related/milvus.pdf)
- **RA definition:** Not named “read amplification” in the paper. Implicit: **sealed segment** (~512 MB) is the load/cache unit from object storage, while a query may need only **nprobe** IVF partitions or a **small graph subset** inside the segment.
- **Root cause:** Coarse **segment object** vs fine probe working set (**D**).
- **How handled:** Segment **cache on QueryNode** (hide RA on warm path); per-segment local IVF/HNSW/DiskANN — does **not** right-size object to probed lists. Tiered storage (2.6.4+) adds **lazy chunk/index fetch** but still segment-scoped.
- **Metrics:** Throughput, latency; segment load time (see `summaries/cold-hot-query-paths.md`).

#### Pinecone serverless · Turbopuffer (product / industry)

- **Category:** Systems · **§3:** D · **PDF:** [`pinecone.pdf`](../related-work/pdfs/read-amp-related/pinecone.pdf) (HTML snapshot)
- **RA definition:** **Slab/object size vs probed IVF clusters** — loading a slab or object pulls bytes beyond the query’s logical working set.
- **Root cause:** LSM **slab** compaction units (Pinecone) or **per-cluster objects** (Turbopuffer) vs nprobe (**D**).
- **How handled:**
  - **Pinecone:** slabs at multiple tiers; IVF inside large slabs; mitigates via **executor cache**, not smaller cold unit.
  - **Turbopuffer:** **one object per cluster** — low **byte RA**, higher **request amplification** (nprobe GETs).
- **Metrics:** Industry latency/recall; bytes vs GET count trade-off (§7).

#### AnyBlob (PVLDB 2023) — contrast (not ANN)

- **Category:** OLAP on object storage · **PDF:** [`anyblob-vldb2023.pdf`](../related-work/pdfs/read-amp-related/anyblob-vldb2023.pdf)
- **RA definition:** **Bulk parallel GET** reads far more data than a point query needs — acceptable for scan, unacceptable for ANN.
- **How handled:** Intentionally high RA for bandwidth; **counterexample** for vector cold path (`discussions/2026-06-03-s3-high-bandwidth-vs-vector-search.md`).

---

### 5.5 Measurement and characterization

#### IISWC 2025 — Storage-Based ANN (AtLarge)

- **Category:** Measurement · **§3:** all · **PDF:** [`iiswc2025-storage-based-ann.pdf`](../related-work/pdfs/read-amp-related/iiswc2025-storage-based-ann.pdf)
- **RA definition:** Empirical **per-query read bandwidth** and I/O overhead for **DiskANN** on modern NVMe — not normalized `bytes_read/useful_bytes`, but profiles how **dataset scale** and **`search_list`** inflate bytes moved.
- **How handled:** N/A (characterization); informs RA benchmarking gap in §8.
- **Metrics:** Read bandwidth vs concurrency, search_list sweep, I/O time share.

**Survey gap:** No paper in this catalog reports a unified **RA ratio** across graph and IVF on identical hardware; OctopusANN/LAANN/Gorgeous post-date IISWC 2025.

---

## 6. Graph vs IVF: amplification profile

| Dimension | Graph on disk | IVF on disk |
|-----------|---------------|-------------|
| **Primary RA mechanism** | Many **small page reads** (hops) | Fewer **large list reads** (nprobe) |
| **Dependency** | Sequential hops block parallelism | Lists **independent** → parallel I/O |
| **Typical RA shape** | `hops × page_size` vs `visited_nodes × node_size` | `nprobe × list_size` vs `candidates × code_size` |
| **Best mitigations** | PageANN, Starling, PipeANN, PQ-in-RAM | SPANN bounded lists, pruning, PQ |
| **Cloud cold path** | Hard — must load graph structure before walk | Easier — fetch **named lists** if storage is list-addressable |

**OctopusANN / disk-based survey consensus:** Graph is **I/O-bound (70–90%)** on SSD; IVF can **saturate SSD parallelism** when lists are independent.

---

## 7. Read amplification vs request amplification (object storage)

On **S3-class** storage, the bottleneck is often **per-GET latency**, not bandwidth.

| Strategy | Bytes moved (RA) | Requests (RA) | Cold latency |
|----------|------------------|---------------|--------------|
| Monolithic index file | Very high | Low (1 GET) | Seconds–minutes |
| Large segment (Milvus) | High | Low–medium | Seconds |
| Per-cluster object (Turbopuffer) | Low | **nprobe GETs** | 100–200 ms × effective parallelism |
| Ideal: sized posting blocks | Low | nprobe (or fewer if batched) | Needs storage API support |

**Neither extreme** achieves sub-second cold p99 at billion scale without **caching**, **batching**, or **placement-aware** storage (Ember thesis in `summaries/cloud-hosted-vs-cloud-native.md`).

---

## 8. Measurement: how papers quantify amplification

| Paper / line | Metric used |
|--------------|-------------|
| PageANN | Pages read per query; node–page alignment |
| OctopusANN | I/O time fraction; page-level model |
| PipeANN | I/O vs compute timeline; wasted speculative reads |
| LAANN | I/O operation count per query |
| Gorgeous | Average disk I/Os per query |
| SPANN | Posting bytes read; machines dispatched (distributed) |
| DiskANN | Bytes in DRAM vs SSD; hop count |
| Starling | Block reads vs hops |
| IISWC 2025 | Per-query read bandwidth vs concurrency / search_list |

**Gap:** Few papers report a single normalized **RA ratio** (`bytes_read / useful_bytes`) across graph and IVF on the **same hardware** — opportunity for Ember evaluation.

---

## 9. Implications for Ember

1. **RAM vs disk framing:** Position in-RAM ANN as avoiding **per-query I/O RA**; disk/disaggregated systems fight RA at **page, list, segment, and object** layers.

2. **Segmentation ≠ solved:** Milvus/Pinecone coarse routing ** lowers** RA but loading a 512 MB segment to use 3 IVF lists is still **orders-of-magnitude** RA — address head-on in related work.

3. **Design target:** Storage units sized to **probed working set** (posting list / page-node / co-located cluster group), with **parallel fetch** without monolithic segment load.

4. **Colocation insight** (from partition survey §4): Clusters probed together should be **physically adjacent** in storage — reduces both RA and request count when batching reads.

5. **Graph on object storage:** Highest RA + request pain — graph cold path is a strong differentiator for **IVF/partition-first** Ember narrative.

---

## 10. Open questions

- Unified **RA benchmark** suite: same datasets, report bytes/read ops/useful bytes for DiskANN vs SPANN vs PageANN vs Milvus segment load.
- **Segment-internal** RA: what fraction of a sealed Milvus segment is touched per typical nprobe?
- **Rerank stage** contribution to RA at high recall — when does full-vector fetch dominate?
- **PipeANN speculative waste** quantified under multi-tenant load.
- **B+ANN** and IISWC 2025 SSD characterization — **added** (§5.5); VeloANN, Gorgeous, LAANN added to graph RA catalog.
- Interaction of **read amplification** with **tail latency** (p99) vs mean — under-studied in disk ANN papers.

---

## 11. Source discussions

- `discussions/2026-05-30-root-cause-analysis.md`
- `discussions/2026-06-03-s3-high-bandwidth-vs-vector-search.md`
- `discussions/2026-06-04-rc-solution-draft.md`
- `discussions/2026-06-04-reviewer-attacks.md`
- `related-work/lindorm-vector-sigmod2026.md`

## Revision history

| Date | Change |
|------|--------|
| 2026-06-16 | Initial survey: definition, taxonomy, paper catalog, RA vs request amplification |
| 2026-06-16 | Added §4 solution catalog: six approach families, eleven fix strategies, RA-source mapping |
| 2026-06-17 | §5 PDFs collected in `related-work/pdfs/read-amp-related/`; Local PDF links updated; AnyBlob downloaded |
| 2026-06-17 | Relocated manual AnyBlob/LindormVector PDFs; added Gorgeous, LAANN, VeloANN, B+ANN, IISWC 2025 to §5 catalog |
| 2026-06-18 | §5 rewritten: per-paper **RA definition**, root cause, handling mechanism, and evaluation metrics (24 papers) |
