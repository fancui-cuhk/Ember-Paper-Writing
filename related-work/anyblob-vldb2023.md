# Exploiting Cloud Object Storage for High-Performance Analytics (AnyBlob)

## Bibliography

| Field | Value |
|-------|--------|
| **Title** | Exploiting Cloud Object Storage for High-Performance Analytics |
| **Authors** | Dominik Durner, Viktor Leis, Thomas Neumann |
| **Venue** | PVLDB 16(11): 2769–2782, 2023 (VLDB 2023) |
| **DOI** | [10.14778/3611479.3611486](https://doi.org/10.14778/3611479.3611486) |
| **PDF in repo** | `related-work/anyblob.pdf` |
| **Artifact** | [github.com/durner/AnyBlob](https://github.com/durner/AnyBlob) |
| **Read date** | 2026-06-03 |

---

## Paper Summary（论文在做什么）

### Problem & motivation

- Cloud **OLAP** engines (Snowflake, Redshift, Databricks, etc.) already use **disaggregated object storage** (S3-class) as source of truth, but most designs still **cache aggressively on local SSD** because remote fetch was historically bandwidth-limited.
- By ~2018+, **100 Gbit/s** instance networking (~12 GB/s) narrowed the gap vs local NVMe, making **direct-on-S3 analytics** more plausible — if the engine can **saturate the network** without wasting CPU.
- Prior work on object storage for databases focused on **OLTP** or caching layers; this paper claims the **first in-depth empirical study** of object-store **latency/throughput/cost** tuned for **general-purpose OLAP retrieval**.

### Three challenges (paper framing)

1. **Achieving instance bandwidth** — high per-request latency ⇒ need **many concurrent outstanding requests**.
2. **Network CPU overhead** — download competes with query execution for cores; vendor SDKs (thread-per-request) oversubscribe CPU.
3. **Multi-cloud** — different vendor APIs; need a unified retrieval layer.

### Approach (blueprint)

1. **Characterize** AWS / Azure / GCP object stores (latency vs object size, throughput, cost, noise).
2. **AnyBlob** — open-source, multi-cloud download manager using **Linux io_uring** (async many requests per thread, fewer context switches than AWS SDK + curl thread-per-request).
3. **Integrate** into **Umbra** via a **retrieval manager + scan operator** that interleaves S3 fetch with columnar scan (not “download everything then query”).

---

## S3 / object-storage performance (core empirical results)

### Architecture note (AWS S3)

- S3 scales by **partitioning prefixes**; high request rates require **spreading keys across prefixes** (same guidance as AWS docs).

### Latency vs request size (Figure 2, cold vs warm)

| Regime | Behavior |
|--------|----------|
| **Small objects** | **First-byte latency dominates** total time; base latency ~**30 ms** (median, from 1 KiB experiments on S3). |
| **Large objects** | Transfer time grows; bandwidth becomes limiting. |
| **Warm access** | 20th consecutive access to same object: lower first-byte and total latency than 1st access. |

**Median decomposition (S3, sufficiently large requests):**

- **Base latency** ≈ **30 ms** (fixed per request).
- **Data latency** ≈ **20 ms/MiB** (slope; 16 MiB total minus base).

Other clouds (anonymized X, Y) show **lower** data latency (12–15 ms/MiB) in their experiments.

### Single-request bandwidth

- Per-object throughput of a **single** request is ~**50 MiB/s** — paper argues this resembles **HDD-class** media behind the API.
- Implication: saturating a **100 Gbit/s** NIC requires on the order of **~100 “HDD-equivalent” parallel streams** (~**200–250 concurrent requests** in their model for 100 Gbit/s instances).

### Throughput at scale (many concurrent GETs, 16 MiB objects)

- With **maximized parallelism** on **100 Gbit/s** instances: median aggregate bandwidth **≥ 75 Gbit/s** on AWS; many runs **75–95 Gbit/s** range.
- **Finding 2:** Object retrieval **can** reach instance network bandwidth — but only with **enough concurrent requests**.

### Cost–throughput sweet spot

- Sweeping request sizes with hundreds of parallel requests (Figure 8):
  - **8–16 MiB** per object: best **cost-throughput** tradeoff for OLAP.
  - Beyond 16 MiB: diminishing returns (e.g. 16→32 MiB doubles bytes but ~1.9× latency; not worth it for cost).

### Variability & “noise”

- Long-running single-object bandwidth over 8 weeks: **25–95 MiB/s** range, median stabilizes ~**55–60 MiB/s** (shared multi-tenant effects).
- **Finding 1 (durability/cost):** Object storage wins on **durability + $/TiB** vs EBS/instance storage for analytics ground truth.

### Official findings (numbered in paper)

| # | Statement |
|---|-----------|
| **F1** | Cloud object storage: best **durability/availability** and favorable **cost** for analytics backends. |
| **F2** | Retrieval can reach **instance network bandwidth** when parallelized. |
| **F3** | Request sizes **8–16 MiB** are **cost-throughput optimal** for OLAP. |
| **F4** | Saturating high-bandwidth networks requires **hundreds of outstanding** requests. |

---

## AnyBlob (system contribution)

```
┌─────────────────────────────────────────────────────────┐
│  Umbra worker threads (scan / operators)                 │
│       ▲  blocks ready                                    │
│       │                                                  │
│  Retrieval manager (schedules fetch, overlaps w/ CPU)   │
│       ▲                                                  │
│  AnyBlob (io_uring, few threads, many async GETs)       │
│       ▲                                                  │
│  S3 / Azure Blob / GCS                                   │
└─────────────────────────────────────────────────────────┘
```

- **Design:** state-machine **message tasks** per connection; **io_uring** for async send/recv; **one thread** can drive many in-flight HTTP requests.
- **vs AWS SDK:** same throughput tier, **lower CPU** — important because download threads compete with query operators on the same cores.
- **Integration:** scan operator requests **columnar block lists** from metadata, then retrieval manager fetches **8–16 MiB-class** objects in parallel while workers process already-fetched batches.

### End-to-end OLAP result (Umbra)

- **Without local caching**, Umbra + AnyBlob reaches **similar performance** to cloud DW systems that **cache on local SSD**, while keeping **better elasticity** (scale compute independently, spot instances, etc.).

---

## 与 Ember / 向量冷查询的关系

### What this paper **does** support for Ember

| Theme | Use in Ember narrative |
|-------|------------------------|
| **Latency vs bandwidth** | Small/random reads are **TTFB-bound**; “parallel slabs” only help if the client actually keeps **hundreds** of requests in flight. |
| **8–16 MiB granularity** | Evidence for **coarse-grained** object reads in disaggregated analytics — aligns with “don’t treat S3 as random block device.” |
| **~30 ms + 20 ms/MiB model** | Back-of-envelope for cold fetch: e.g. 64×8 MiB parallel still has a **tail** dominated by slowest stragglers + CPU. |
| **Single-stream ~50 MiB/s** | Explains why **one** large index GET is not “NVMe speed” without range parallelism. |
| **CPU on network path** | Download + index deserialize + ANN compete for cores — bandwidth not the only bottleneck. |
| **Multi-tenant variance** | S3 latency/throughput **noisy** — tail latency is a first-class concern even for OLAP. |

### What this paper **does not** cover

| Gap | Why it matters for vector DBs |
|-----|------------------------------|
| Workload | **Sequential columnar scan** of large tables — not **graph/IVF index traversal**, not **many tiny metadata reads**. |
| Access pattern | Optimized **throughput**; explicitly notes first-byte matters less when **bandwidth-dominated** — opposite of **latency-sensitive top-k search**. |
| Index structure | No ANN, no slab/segment, no “load index then probe.” |
| Tail at billion scale | No vector cold-start SLA study. |

**Positioning one-liner for Related Work:**  
Durner et al. show that **OLAP on S3 can saturate 100 Gbit/s** with **hundreds of 8–16 MiB parallel GETs** and careful CPU-aware IO (AnyBlob). **Vector databases on S3** face a **harder subproblem**: colder paths often need **many smaller or dependent reads** and **index build**, where **per-request latency** and **client parallelism limits** dominate — the paper’s OLAP blueprint is **necessary context**, not a proof that vector cold query is “solved.”

---

## 与近期讨论的对照

| 我们讨论中的说法 | 论文支持？ |
|------------------|------------|
| S3 高带宽需要大量并行 GET | **Yes** — F2, F4, ~200–250 requests for 100 Gbit/s |
| 单次 GET ~几十–几百 MB/s，不是网卡上限 | **Yes** — ~50 MiB/s single-request, HDD analogy |
| 8–16 MiB range 是 best practice | **Yes** — F3, OLAP cost model |
| Milvus blog ~200–330 MB/s 与 “S3 很快” 不矛盾 | **Consistent** — aggregate with limited concurrency vs paper’s **maximized** experiment |
| 向量冷查询 = 论文的 OLAP scan | **No** — different access pattern and SLO |

---

## 论文写作用途

| Section | Suggested use |
|---------|----------------|
| **Background / S3 primer** | Cite F3 (8–16 MiB), latency model (30 ms + 20 ms/MiB), F4 (hundreds of requests). |
| **Root cause — request amplification** | Contrast OLAP (engine controls parallelism) vs vector systems with **opaque client limits** or **index-dependent serial steps**. |
| **Related Work — disaggregated analytics** | AnyBlob/Umbra as **closest rigorous S3 performance study** for OLAP; distinguish from Pinecone/Milvus/Turbopuffer vector paths. |
| **Ember design** | Motivate **coarse read unit + bounded round-trips + CPU-aware IO** using this paper as **OLAP evidence**, then argue vector needs **stronger** constraints (tail latency). |

---

## Open questions / follow-ups

- [ ] Re-run paper’s latency breakdown on **MinIO** (Milvus blog backend) vs **AWS S3** same region/instance.
- [ ] Map Milvus **index file size** + **chunk size** to paper’s request-size model (how many parallel 8–16 MiB equivalents?).
- [ ] Check whether Ember target storage can keep **prefix sharding** to avoid single-prefix 5.5k GET/s soft limits (AWS doc alignment).

---

## Key citations (BibTeX-friendly)

```bibtex
@article{durner2023anyblob,
  title={Exploiting Cloud Object Storage for High-Performance Analytics},
  author={Durner, Dominik and Leis, Viktor and Neumann, Thomas},
  journal={Proceedings of the VLDB Endowment},
  volume={16},
  number={11},
  pages={2769--2782},
  year={2023},
  doi={10.14778/3611479.3611486}
}
```
