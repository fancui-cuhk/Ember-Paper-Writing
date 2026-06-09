# Disk-Based ANN and Distributed ANN: A Survey of the Literature

---

## 1. Why Disk-Based ANN?

The fundamental motivation is **cost and scale**. In-memory ANN algorithms (HNSW, NSG, Vamana) achieve excellent recall and latency, but their memory footprint is prohibitive at billion-scale:

- A 1B × 128-dim float32 dataset requires ~512 GB of raw storage; HNSW's graph structure adds 20–40% overhead
- DRAM costs ~$5–10/GB; NVMe SSD costs ~$0.30–0.50/GB — a **16–33× cost gap**
- A server with 1 TB DRAM is astronomically expensive and impractical for most deployments

The inflection point came around 2019–2021 when web search, recommendation systems, and retrieval-augmented ML began operating at **billion-scale** embedding corpora. Memory was no longer a tunable parameter — it became the dominant cost.

**DiskANN (NeurIPS 2019, Microsoft Research)** was the first practical answer: by keeping only PQ-compressed vectors in DRAM (~10% of dataset size) and storing the full graph and full-precision vectors on NVMe SSD, it demonstrated 95% recall at ~5ms latency on 1B vectors on a single machine — serving **5–10× more points per node** compared to in-memory HNSW at the same recall.

---

## 2. Storage Hierarchy Reference

| Dimension | DRAM | NVMe SSD | HDD |
|---|---|---|---|
| Random read latency | ~100 ns | ~70–100 µs | ~5–10 ms |
| Sequential bandwidth (single thread) | ~50–100 GB/s | ~500 MB/s–1 GB/s | ~150–250 MB/s |
| Sequential bandwidth (saturated, multi-thread) | ~100+ GB/s | ~3–7 GB/s | ~250 MB/s (no gain per spindle) |
| Random IOPS (4 KB) | ~100M+ | ~500K–1M | ~100–200 |
| Cost per GB (enterprise, 2025–2026) | ~$5–10 | ~$0.30–0.50 | ~$0.02–0.03 |
| Cost ratio vs HDD | ~300× | ~16× | 1× |
| Internal parallelism | High | Very high (~47× with concurrent reads) | None (single spindle) |
| Seek penalty | None | None | 5–10 ms mechanical |
| Access granularity | 64 B (cache line) | 4 KB page | 4 KB sector |

**Key insight on SSD bandwidth:** The rated 5–7 GB/s sequential bandwidth of a modern NVMe SSD is *not* achievable with a single thread. A single thread delivers ~500 MB/s–1 GB/s. Saturating all internal NAND channels requires 10–64 concurrent threads or async I/O (io_uring/SPDK), yielding ~3–7 GB/s — a ~47× improvement over single-thread access.

---

## 3. Is All Disk-Based ANN Work on SSD?

**Yes — the entire field assumes NVMe SSD.** Every paper in the literature — DiskANN, SPANN, Starling, PipeANN, OctopusANN, PageANN, B+ANN — explicitly targets NVMe SSD and evaluates exclusively on NVMe hardware. The IISWC 2025 characterization study lists NVMe SSD as a hardware requirement to reproduce results.

**No paper targets HDD.** This reflects a principled judgment: HDD random I/O (~10 ms seek) is incompatible with ANN access patterns. The field moved directly from DRAM to SSD, skipping HDD entirely.

---

## 4. SSD Design Principles Exploited by ANN Papers

Disk-based ANN papers rely on four distinct SSD properties:

### 4.1 Low Random Read Latency (~70–100 µs)
**Philosophy:** Replace DRAM access with SSD access for full-precision vectors. DiskANN's insight: SSD random reads are ~100 µs — fast enough that total query latency stays in the millisecond range if the number of reads is bounded. Keep PQ-compressed vectors in DRAM for navigation; read full-precision vectors from SSD only for final reranking.

### 4.2 High Internal Parallelism
**Philosophy:** Issue many concurrent random reads to exploit internal SSD parallelism. A modern NVMe SSD (e.g., Samsung PM1743) achieves 118 MB/s with 1 thread but 5,606 MB/s with 64 threads — a 47× improvement. Both DiskANN and SPANN batch multiple read requests using io_uring or AIO so multiple nodes are fetched in parallel across NAND channels.

### 4.3 4 KB Page Granularity
**Philosophy:** Align index structures to SSD page boundaries to eliminate read amplification. PageANN maps one logical graph node to exactly one physical SSD page — one node fetch = one SSD page read. DiskANN similarly pads each node to sector-aligned sizes.

### 4.4 No Sequential Access Penalty
**Philosophy:** Treat random access as first-class. SSD has no mechanical head, so accessing any address costs the same. Disk-based ANN papers freely use random access graph traversal. The bottleneck is the *count* of I/O operations, not their physical layout.

---

## 5. Server Hardware Assumptions

These papers assume substantial but not extreme servers. They are never explicit about this, but examination reveals consistent patterns:

| Paper | Server RAM | SSD |
|---|---|---|
| DiskANN (NeurIPS 2019) | 64 GB | NVMe |
| SPANN (NeurIPS 2021) | 128 GB | 2.6 TB RAID-0 NVMe |
| OctopusANN (PVLDB 2026) | 128 GB | 4 TB NVMe |
| PageANN (arXiv 2025) | 64 GB | 2 TB NVMe |

- Memory budget is typically **10–30% of the dataset size**
- **SPANN requires ≥30% of dataset size in DRAM** to store its SPTAG centroid graph — a hard floor
- Compute is never the bottleneck (I/O accounts for 70–90% of query latency); all papers use 24–52 core Xeon servers with 16–64 I/O threads
- These are dedicated, well-provisioned servers — not commodity or memory-constrained machines

---

## 6. IVF vs. Graph on SSD

The two dominant index families have fundamentally different I/O profiles on SSD.

### IVF on SSD

| Property | Characteristic |
|---|---|
| Access pattern | Identify nprobe centroids → load nprobe posting lists → score |
| Number of I/O operations | nprobe (typically 10–100) |
| Read size | Large and predictable (whole posting list, 1–100 MB) |
| I/O dependency | **None** — all nprobe reads are independent, fully parallel |
| SSD fit | Excellent — saturates internal parallelism |
| HDD fit | Marginal — nprobe × 10 ms seek latency |

### Graph (HNSW/Vamana) on SSD

| Property | Characteristic |
|---|---|
| Access pattern | Start at entry node → read neighbors → move to best → repeat |
| Number of I/O operations | 50–200+ hops (data-dependent) |
| Read size | Small (~4 KB per hop) |
| I/O dependency | **Strictly sequential** — hop H+1 requires result of hop H |
| SSD fit | Poor — sequential dependency blocks parallelism; I/O = 70–90% of latency |
| HDD fit | Completely broken — 100 hops × 10 ms = ~1 second of pure I/O |

### Why Graph Dominates Despite Poor SSD Fit

1. **Recall per I/O at high recall:** At ≥90–95% recall, graph-based methods are strictly more I/O-efficient. IVF recall improvement is sub-linear in nprobe — doubling nprobe yields diminishing returns. SPANN outperforms DiskANN only under low recall requirements; above 90–95% recall, graph wins.

2. **PQ neutralizes most per-hop I/O cost:** Keeping PQ-compressed vectors in DRAM allows distance approximation without touching SSD. SSD reads are only needed for full-precision reranking, dramatically reducing the per-hop I/O count.

3. **Ecosystem lock-in:** DiskANN is the de facto industrial standard, adopted by Microsoft Azure, Milvus, Huawei GaussDB, and Timescale. Five years of optimization papers (Starling, PipeANN, OctopusANN, PageANN) have built a rich engineering ecosystem around graph-based disk ANN.

4. **The I/O gap is closing:** PipeANN (OSDI 2025) achieves 1.14×–2.02× the latency of in-memory Vamana through pipelined async I/O — only 14–100% slower than no disk at all.

---

## 7. Key Papers: Graph-Based Disk ANN

### DiskANN (NeurIPS 2019, Microsoft Research)
- **Core idea:** Vamana graph stored on SSD + PQ-compressed vectors in DRAM
- **Memory footprint:** ~10% of dataset size
- **Result:** 95% recall@1 on 1B SIFT vectors at ~5 ms on a single machine
- **Innovation:** Sector-aligned node layout; full-precision vectors only fetched for final reranking

### Starling (SIGMOD 2023)
- Adds SSD-aware graph construction to reduce node degree and improve locality
- Targets SSDs with large sequential read coalescing

### OctopusANN (PVLDB 2026)
- Three orthogonal I/O optimizations over DiskANN
- Achieves 87.5–149.5% higher throughput than DiskANN at 90% recall
- Identifies I/O as 70–90% of query latency; compute is only 9.5% of I/O latency

### PipeANN (OSDI 2025, Tsinghua University)
- **Core insight:** Breaks the strict compute-I/O ordering of best-first search
- Uses io_uring to issue I/O speculatively before the current hop's compute finishes
- Dynamic beam width (DW): small width early ("approach phase"), large width late ("converge phase")
- **Result:** 1.14×–2.02× latency of in-memory Vamana; 35% of DiskANN's latency
- **Trade-off:** Speculative I/O wastes bandwidth; lower throughput than greedy I/O under high concurrency

### PageANN (arXiv 2025)
- Maps each graph node exactly to one SSD page (4 KB), eliminating read amplification
- 1.85×–10.83× higher throughput than state-of-the-art disk-based methods

---

## 8. Key Papers: IVF-Based Disk ANN

### SPANN (NeurIPS 2021, Microsoft Research / Bing)
- **Core idea:** Centroid graph in DRAM (SPTAG); posting lists stored on SSD
- **Three key innovations:**
  1. Hierarchical Balanced Clustering (HBC): constrains posting list size to 12–48 KB for bounded I/O
  2. Closure assignment: boundary vectors duplicated into up to 8 nearby clusters (~20% storage overhead)
  3. Query-aware dynamic pruning: nprobe adapted per query based on distance threshold `(1 + ε₂) × d_min`
- **Result:** 2× faster than DiskANN at 90% recall on SIFT1B, SPACEV1B, DEEP1B with same memory
- **Key claim:** IVF's parallel I/O structure is inherently more SSD-friendly than graph's sequential hops
- **Requires ≥30% dataset size in DRAM** for centroid graph — higher memory floor than DiskANN

---

## 9. Distributed ANN

### 9.1 Motivation
Single-machine disk-based ANN (DiskANN, SPANN) still requires 64–128 GB DRAM. For truly web-scale corpora (hundreds of billions of vectors, e.g., Bing, Alibaba), even this is insufficient. Distributed ANN distributes both the index and compute across many machines.

### 9.2 Approach 1: Vector Sharding (Baseline)
The simplest approach — shard the dataset across N machines, route every query to all shards, merge top-K results. Used by Milvus, Weaviate, Pinecone, and most production vector databases. This is embarrassingly parallel but does **not** reduce per-machine memory requirements — it just spreads the data. Storage cost scales linearly with cluster size.

### 9.3 Approach 2: SPANN Distributed (Microsoft Bing, Production)
SPANN's IVF structure distributes naturally: once centroid IDs are identified, each posting list can be fetched from a different machine in parallel — no cross-machine sequential dependencies. In production on SPACEV1B (32 machines), query-aware pruning reduces dispatched machines per query from 32 to **6.3** (80.3% reduction in inter-machine communication). Graph-based methods have no equivalent — hop chains across machines introduce blocking network round-trips at each hop.

### 9.4 Approach 3: DSANN (Ant Group / ECNU, arXiv 2025)
Targets **distributed file system (DFS) storage** (Alibaba's Pangu) where I/O latency is 0.1–10 ms — far worse than local SSD.

**Problem:** Both DiskANN and SPANN break on DFS:
- DiskANN's sequential hop chain means each hop incurs a network round-trip (~1–10 ms), making queries take seconds
- SPANN must load entire posting lists before fine-grained search; DFS latency dominates

**Solution — Point Aggregation Graph (PAG):**
- Sample 10–25% of vectors as "aggregation points" → build an in-memory proximity graph
- Store remaining 75–90% as "residual points" in DFS
- Graph traversal on aggregation points happens asynchronously in memory; DFS I/O for residuals is overlapped
- Construction complexity: O(n log pn) — parallelizable across commodity machines, strictly better than DiskANN's O(n log n)

**Result:** Efficient billion-scale ANN on distributed object storage; index built and served across commodity machines without large per-machine DRAM.

### 9.5 Approach 4: HARMONY (Renmin University / MIT / Tsinghua, 2025)
**Radical approach:** shard by *dimensions*, not by *vectors*.

**Key insight:** Euclidean distance decomposes additively:

$$D^2(\mathbf{p}, \mathbf{q}) = \sum_{k=1}^{M} D_k^2(\mathbf{p}, \mathbf{q})$$

Each machine computes a partial distance over its assigned dimensions. Once the running partial sum exceeds the current threshold, the candidate is pruned and remaining machines are skipped — **dimension-level early stopping**. By machine 3 of 4, over 80% of candidates are already pruned; 97.4% pruning by machine 4.

**Trade-off:** Multiple network round-trips per query (each partial distance must be returned to coordinator) vs. vector sharding's single dispatch. HARMONY uses a cost model to dynamically choose between dimension-based and vector-based partitioning based on workload skew.

**Graph incompatibility:** HARMONY notes explicitly that graph-based indexes are poorly suited to distribution — query paths generate edges across machines, resulting in high network latency per hop. IVF and cluster-based approaches are inherently more distribution-friendly.

---

## 10. Graph vs. IVF: Distribution Summary

| Dimension | Graph (HNSW/Vamana) | IVF/Cluster |
|---|---|---|
| Single-machine SSD fit | Poor (sequential hops) | Good (parallel posting list reads) |
| Single-machine recall efficiency | High at ≥95% recall | High at ≤90% recall |
| Distribution model | Very difficult — hop chains require sequential cross-machine I/O | Natural — posting lists fetched from different machines in parallel |
| DRAM floor | ~10% of dataset (DiskANN) | ~30% of dataset (SPANN) |
| Engineering ecosystem | Dominant — DiskANN, Starling, PipeANN, OctopusANN, PageANN | Thinner — SPANN, DSANN |
| Production adoption | Microsoft Azure, Milvus, Huawei GaussDB, Timescale | Microsoft Bing (SPANN), Alibaba (DSANN) |

---

## 11. Timeline of Key Developments

| Year | Paper | Venue | Contribution |
|---|---|---|---|
| 2019 | DiskANN | NeurIPS | First practical single-machine SSD-based graph ANN |
| 2021 | SPANN | NeurIPS | IVF-based SSD ANN; 2× faster than DiskANN at 90% recall; production Bing deployment |
| 2023 | Starling | SIGMOD | SSD-aware graph construction for better locality |
| 2025 | PipeANN | OSDI | Async pipelined search; 35% of DiskANN latency; 1.14–2.02× in-memory |
| 2025 | DSANN | arXiv | Graph-cluster hybrid for distributed file system storage |
| 2025 | HARMONY | VLDB | Dimension-based distribution with early stopping; 97.4% candidate pruning |
| 2025 | PageANN | arXiv | Node-to-page alignment; 1.85–10.83× higher throughput |
| 2026 | OctopusANN | PVLDB | Three I/O optimizations; 87.5–149.5% higher throughput than DiskANN |
