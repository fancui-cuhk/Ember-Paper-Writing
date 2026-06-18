# Read Amplification in Disk-Based Vector Search: Survey

## Overview

**Read amplification** here means: the system **reads (or transfers) more bytes than the query logically needs** to produce its approximate nearest-neighbor result. The wasted fraction grows when storage access granularity is coarser than the index’s logical working set.

This note surveys read amplification in **disk-resident** and **disaggregated-storage** vector search: **sources** (§3), **fix approaches** (§4), and **papers** (§5). It complements:

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

## 5. Paper catalog (by mitigation strategy)

Entries include **Category** (graph / IVF / LSH / systems), **RA source** (§3 letter), **Local PDF** when available.

### 5.1 Page-aligned / block-local graph layout

#### DiskANN (NeurIPS 2019)

- **Category:** Graph
- **RA source:** A (sector alignment), B (hop chain), E (rerank)
- **Local PDF:** [`diskann.pdf`](../related-work/pdfs/read-amp-related/diskann.pdf)
- **Technique:** Vamana on SSD with **sector-aligned** nodes; PQ vectors in DRAM for navigation; SSD for full vectors + graph.
- **RA story:** Padding to sector size accepts some internal waste to avoid **partial-page** reads. Still **70–90%** of latency is I/O in follow-on work because **hop count** dominates.

#### PageANN (arXiv 2025)

- **Category:** Graph
- **RA source:** A (explicit target)
- **Local PDF:** [`pageann.pdf`](../related-work/pdfs/read-amp-related/pageann.pdf)
- **Technique:** **Page-node graph** — one logical node maps to **one 4 KB SSD page**; clustered similar vectors; coordinated memory/disk allocation.
- **RA story:** Paper claims elimination of **page-level read amplification** vs DiskANN misalignment; 1.85×–10.83× throughput vs SOTA disk ANN.

#### Starling (SIGMOD 2024)

- **Category:** Graph (within one **segment**)
- **RA source:** A, B
- **Local PDF:** [`starling.pdf`](../related-work/pdfs/read-amp-related/starling.pdf)
- **Technique:** In-memory navigation graph + **reordered disk graph** with **block shuffling** so co-accessed neighbors share blocks; block search strategy.
- **RA story:** Targets Milvus-style **segment** where neighbors span blocks → one hop used to pull whole block including unrelated nodes.

#### OctopusANN (PVLDB 2026)

- **Category:** Graph
- **RA source:** A, B (design-space exploration)
- **Local PDF:** [`octopusann.pdf`](../related-work/pdfs/read-amp-related/octopusann.pdf)
- **Technique:** Joint optimization of **memory layout, disk layout, search algorithm**; page-level I/O model.
- **RA story:** Reports **70–90%** query time in I/O; compositions beat Starling/DiskANN by reducing effective bytes per hop.

#### NaviX (PVLDB 2025)

- **Category:** Graph (graph DBMS disk HNSW)
- **RA source:** A
- **Local PDF:** [`navix.pdf`](../related-work/pdfs/read-amp-related/navix.pdf)
- **Technique:** Disk HNSW with **graph/block partition** aligned to DB pages for filtered queries.
- **RA story:** Page-local layout for hybrid vector+graph workloads.

#### PipeANN (OSDI 2025)

- **Category:** Graph
- **RA source:** B (latency), **F** (speculative over-read)
- **Local PDF:** [`pipeann.pdf`](../related-work/pdfs/read-amp-related/pipeann.pdf)
- **Technique:** Async pipelined best-first search (io_uring); speculative I/O; dynamic beam width.
- **RA story:** Trades **bandwidth RA** for latency — may read pages not on final frontier.

#### Gorgeous (arXiv 2025)

- **Category:** Graph
- **RA source:** A, B (explicit I/O reduction target)
- **Local PDF:** [`gorgeous.pdf`](../related-work/pdfs/read-amp-related/gorgeous.pdf)
- **Technique:** **Graph-prioritized memory cache** (adjacency lists over full vectors); **graph-replicated disk blocks** with **neighbor packing** (prefetch neighbors' adjacency lists in same 4 KB block); async prefetch.
- **RA story:** Reports **~39% fewer disk I/Os** at matched recall vs DiskANN/Starling; addresses Starling's diminishing block overlap at high dimension by packing **graph structure** instead of only co-locating vectors.

#### LAANN (arXiv 2026)

- **Category:** Graph
- **RA source:** B, F (I/O count reduction + pipeline)
- **Local PDF:** [`laann.pdf`](../related-work/pdfs/read-amp-related/laann.pdf)
- **Technique:** **Look-ahead search** (phase-aware I/O vs compute trade-off); **priority I/O–CPU pipeline**; lightweight in-memory centroid graph over page nodes; page-granular disk graph.
- **RA story:** Explicitly targets **I/O read amplification** and hop count — **1.59×–6.34× fewer I/O operations** vs SOTA disk ANN at Recall@10=0.9; complements PipeANN's latency masking with **I/O-aware search policy**.

#### VeloANN (arXiv 2026)

- **Category:** Graph
- **RA source:** A, B (read amplification / over-fetch)
- **Local PDF:** [`veloann.pdf`](../related-work/pdfs/read-amp-related/veloann.pdf)
- **Technique:** **Affinity-based page placement**; hierarchical compression; **record-level buffer pool** (hot neighbor records pinned); coroutine async runtime; beam-aware search prioritizing cached records.
- **RA story:** Names **read amplification** and storage stalls from poor traversal locality; co-locates related vectors in same page to cut **over-fetching** and page swapping under memory pressure.

#### B+ANN (arXiv 2025)

- **Category:** Graph / B+-tree hybrid
- **RA source:** A, B (block locality)
- **Local PDF:** [`b-plus-ann.pdf`](../related-work/pdfs/read-amp-related/b-plus-ann.pdf)
- **Technique:** k-means++ **semantic blocks** stored in **B+ tree** pages on disk; hybrid block- and edge-level in-memory traversal; batched SIMD distance within blocks.
- **RA story:** Improves **spatial/temporal locality** vs random graph hops — **19% fewer cache misses** vs HNSW; block reads amortize multiple vectors per I/O vs fine-grained DiskANN hops (different index family, same RA axis).

---

### 5.2 Bounded / selective IVF list reads

#### SPANN (NeurIPS 2021)

- **Category:** IVF on disk
- **RA source:** C
- **Local PDF:** [`spann.pdf`](../related-work/pdfs/read-amp-related/spann.pdf)
- **Technique:** Hierarchical balanced clustering → **bounded posting size** (12–48 KB); centroids + SPTAG in RAM; **query-aware dynamic pruning** of lists.
- **RA story:** Limits **maximum bytes per list read**; reduces **nprobe effective** via distance threshold — directly attacks IVF list RA.

#### SPFresh (SOSP 2023)

- **Category:** IVF on disk (updates)
- **RA source:** C
- **Local PDF:** [`spfresh.pdf`](../related-work/pdfs/read-amp-related/spfresh.pdf)
- **Technique:** Balanced small postings (SPANN lineage); LIRE split/rebalance.
- **RA story:** Balanced postings bound **per-probe I/O** and tail latency — RA control via list size invariants.

#### SPANN / IVF-PQ (Jégou et al., foundational)

- **Category:** IVF
- **RA source:** C, E
- **Local PDF:** [`product-quantization-for-nearest-neighbor-search.pdf`](../related-work/pdfs/read-amp-related/product-quantization-for-nearest-neighbor-search.pdf) (verify — small HAL export)
- **Technique:** Inverted file + PQ codes in lists; asymmetric distance computation.
- **RA story:** PQ shrinks **useful-bytes-per-candidate** inside each list read.

#### RAIRS (SIGMOD 2026)

- **Category:** IVF
- **RA source:** C (redundant assignment → duplicate work/bytes)
- **Local PDF:** [`rairs.pdf`](../related-work/pdfs/read-amp-related/rairs.pdf)
- **Technique:** SEIL layout shares cells across redundant IVF assignments.
- **RA story:** Cuts **redundant distance computation and list I/O** from duplicate vectors across lists.

#### I-LSH (ICDE 2019) · SK-LSH (PVLDB 2014) · Learned-function lists (ICDE 2020)

- **Category:** LSH / external memory
- **RA source:** A, C (bucket/page layout)
- **Local PDF:** [`i-lsh.pdf`](../related-work/pdfs/read-amp-related/i-lsh.pdf), [`sk-lsh.pdf`](../related-work/pdfs/read-amp-related/sk-lsh.pdf), [`i-o-efficient-approximate-nearest-neighbour-search.pdf`](../related-work/pdfs/read-amp-related/i-o-efficient-approximate-nearest-neighbour-search.pdf)
- **Technique:** Incremental probing; **sorted bucket layout** on disk; learned orderings for sequential I/O.
- **RA story:** Reduce **pages touched per bucket** and improve sequentiality — RA via better locality on disk.

---

### 5.3 Hybrid memory–disk / two-stage (reduce disk useful-set)

#### HAKES (PVLDB 2025)

- **Category:** IVF two-stage (filter + refine)
- **RA source:** C, E
- **Local PDF:** [`hakes.pdf`](../related-work/pdfs/read-amp-related/hakes.pdf)
- **Technique:** Compressed IVF filter in memory; full vectors on RefineWorkers; optional IVF-assignment sharding.
- **RA story:** Minimizes **full-vector bytes read**; filter stage avoids loading raw embeddings for entire lists.

#### LindormVector (SIGMOD 2026 Industry)

- **Category:** Systems (IVFPQ on DFS)
- **RA source:** D, C
- **Local PDF:** [`lindorm-vector.pdf`](../related-work/pdfs/read-amp-related/lindorm-vector.pdf) — see [`related-work/lindorm-vector-sigmod2026.md`](../related-work/lindorm-vector-sigmod2026.md)
- **Technique:** Centroids/HNSW resident; **posting lists on LindormDFS** with LDServer cache.
- **RA story:** On cache miss, each probed cluster = DFS read; **does not solve** object-level RA vs request trade-off on cold path (reader analysis).

#### DSANN (arXiv 2025)

- **Category:** Graph + cluster on **distributed file system**
- **RA source:** B (hop chain on slow DFS), D
- **Local PDF:** [`dsann.pdf`](../related-work/pdfs/read-amp-related/dsann.pdf)
- **Technique:** Point Aggregation Graph — sample aggregation points in memory; async overlap DFS reads for residuals.
- **RA story:** DiskANN/SPANN RA **worse on DFS** (0.1–10 ms I/O); PAG limits **sequential blocking reads** of whole lists/hops.

---

### 5.4 Systems-layer / cloud disaggregation (segment & object RA)

#### Milvus 2.x / segments

- **Category:** Systems (#1 partition → local index)
- **RA source:** D
- **Local PDF:** [`milvus.pdf`](../related-work/pdfs/read-amp-related/milvus.pdf) (SIGMOD paper); prod docs not archived
- **Technique:** Sealed **segments** (~512 MB) with local IVF/HNSW; object storage backing.
- **RA story:** Cold query may load **whole segment**; IVF inside may need only **nprobe ≪ nlist** clusters → segment-level RA persists (reviewer attack note in `discussions/2026-06-04-reviewer-attacks.md`).

#### Pinecone serverless · Turbopuffer (product pattern)

- **Category:** Systems
- **RA source:** D ↔ **request amplification** trade-off
- **Local PDF:** [`pinecone.pdf`](../related-work/pdfs/read-amp-related/pinecone.pdf) (HTML snapshot)
- **Technique:** Pinecone: geometric partitions + **slabs**; Turbopuffer: **per-cluster S3 objects**.
- **RA story:** Turbopuffer **low bytes RA**, **high nprobe request count**; Pinecone slab middle ground.

#### AnyBlob (PVLDB 2023) — OLAP on object storage (contrast)

- **Category:** Not vector — **counterexample**
- **Local PDF:** [`anyblob-vldb2023.pdf`](../related-work/pdfs/read-amp-related/anyblob-vldb2023.pdf) · notes in [`related-work/anyblob-vldb2023.md`](../related-work/anyblob-vldb2023.md)
- **RA story:** Bulk parallel GETs achieve bandwidth but **read much unused data** — poor fit for **random IVF/graph** unless accepting RA (`discussions/2026-06-03-s3-high-bandwidth-vs-vector-search.md`).

---

### 5.5 I/O measurement & characterization

#### IISWC 2025 — Storage-Based Approximate Nearest Neighbor (AtLarge)

- **Category:** Measurement (DiskANN on NVMe)
- **RA source:** All (empirical I/O profiling)
- **Local PDF:** [`iiswc2025-storage-based-ann.pdf`](../related-work/pdfs/read-amp-related/iiswc2025-storage-based-ann.pdf)
- **Technique:** Reproducible characterization of **DiskANN** on modern SSDs — per-query bandwidth vs concurrency, dataset scale, `search_list` sweep.
- **RA story:** Quantifies how **larger datasets and higher search_list** increase **per-query read bandwidth** and I/O overhead; useful baseline for normalized RA benchmarking (§8 gap).

**Survey gap note:** Papers above (PageANN, OctopusANN, LAANN, Gorgeous) post-date or extend the IISWC study; no unified cross-family RA ratio table exists yet.

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
