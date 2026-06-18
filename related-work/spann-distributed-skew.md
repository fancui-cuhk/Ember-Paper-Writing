# SPANN §4.3 — Distributed Search & Query-Skew Placement

## Bibliography

| Field | Value |
|-------|--------|
| **Title** | SPANN: Highly-efficient Billion-scale Approximate Nearest Neighbor Search |
| **Authors** | Qi Chen et al. (Microsoft) |
| **Venue** | NeurIPS 2021 |
| **Relevant section** | **§4.3 Extension to distributed search scenario** |
| **PDF** | [NeurIPS](https://proceedings.neurips.cc/paper_files/paper/2021/file/299dc35e747eb77177d9cea10a802da2-Paper.pdf) |
| **Read date** | 2026-06-18 |

---

## Internal reading note — Ember relevance: **high**

**TL;DR:** After **spatial** micro-partitioning (HBC + closure), SPANN **packs** partitions onto machines with **best-fit bin-packing weighted by query-access history**. This is the clearest prior art for *“keep geometry for subset routing, but rebalance expected QPS across nodes using traces.”*

**Not** the same as posting-length balance (HBC) — that is **data-size / disk I/O** balance on one machine. §4.3 is **query-load** balance across machines.

---

## Mechanism (§4.3)

1. Build many **small spatial partitions** (> machine count) via multi-constraint balanced clustering + closure.
2. Profile **history query access distribution** (paper: ~100k production accesses; train / valid / test split).
3. **Best-fit bin-packing**: pack small partitions into M machine-bins so **data size and query accesses** are both even.
4. Online: **query-aware dynamic pruning** limits which machines each query hits (subset routing preserved).

**Verbatim anchor:**

> "we need to balance not only the data size but also the query access in each machine to avoid the hot spots."

> "use best-fit bin-packing algorithm … according to the history query access distribution."

---

## Comparison — SPANN §4.3 vs Harmony

| Dimension | **SPANN §4.3** | **Harmony** (SIGMOD 2025) |
|-----------|----------------|---------------------------|
| **Problem** | Hot machines under IVF subset routing | Hot partitions under vector/dimension sharding |
| **Workload signal** | **Offline** query-access trace | **Runtime** cost model + query skewness metrics |
| **Placement** | Bin-pack spatial micro-partitions → machines | Adaptive hybrid vector- vs dimension-based granularity |
| **Keeps spatial colocation?** | Yes — packs *already spatial* partitions | Partially — vector-based shard + dimension blocks |
| **Online drift** | Not in paper (GP-ANN: trace can go stale) | Cost model re-evaluates; load-aware routing |

**Ember take:** Same **design pattern** — *decouple “where vectors live in embedding space” from “how much query load each node sees”* — SPANN does it **offline at pack time**; Harmony **online via cost model + routing**. Both are more relevant than fan-out-all (FLANN / block) for a **spatial-colocation + skew** story.

---

## Comparison — SPANN §4.3 vs SABBSR

| | SPANN §4.3 | SABBSR |
|---|------------|--------|
| Unit packed | IVF micro-partitions / posting groups | IVF buckets (centroids) |
| Weight | Query access count per partition | `relevance × size` (probe frequency × points) |
| Setting | Microsoft billion-scale IVF | SABES distributed IVFADC |

---

## Scope caveats

- NeurIPS **main body** is **single-node** hybrid memory–SSD IVF.
- §4.3 is an **extension** (distributed eval, not primary claim).
- Cites **Pyramid** for partial search — see [`pyramid-bigdata2019.md`](pyramid-bigdata2019.md) (Ember: low relevance).

---

## Related repo files

- [`summaries/query-skew-spatial-partitioning-survey.md`](../summaries/query-skew-spatial-partitioning-survey.md) §9
- [`related-work/gp-ann.md`](gp-ann.md) — cites SPANN skew limitation; online replication
- Harmony — survey Part III; no dedicated reading note yet
