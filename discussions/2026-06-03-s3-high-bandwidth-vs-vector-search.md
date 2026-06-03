# Session: S3 High-Bandwidth Conditions vs Vector Search Workloads

- **Date:** 2026-06-03
- **Topic Tags:** `S3`, `bandwidth`, `AnyBlob`, `vector-search`, `OLAP`, `HNSW`, `IVF`, `cold-query`, `positioning`

## Author’s view: when S3 can deliver ~10 GB/s read bandwidth

The author has internalized the prerequisites for **high aggregate read bandwidth** on object storage (grounded in [AnyBlob / Durner et al., PVLDB 2023](../related-work/anyblob-vldb2023.md)):

| # | Condition | Rationale (author + paper) |
|---|-----------|----------------------------|
| 1 | **Optimal GET granularity ≈ 16 MiB** | Too small → **latency dominates** (per-request TTFB); too large → **bandwidth does not improve proportionally** (diminishing returns past cost-throughput sweet spot; paper cites **8–16 MiB** as optimal for OLAP). |
| 2 | **Enough parallel GETs** — on the order of **hundreds** | AnyBlob paper: ~**200–250** concurrent outstanding requests to saturate **100 Gbit/s** (~10 GB/s class) instances. |
| 3 | **Network is not the bottleneck** | Instance/link must support the target aggregate throughput (e.g. 100 Gbit/s networking). |
| 4 | **CPU keeps up** | Many parallel GETs ⇒ many **HTTPS + TLS/crypto** stacks in flight; download threads compete with query operators for cores (paper’s motivation for AnyBlob / io_uring). |

**If all four hold:** aggregate S3 read bandwidth can reach **~10 GB/s** (order-of-magnitude, 100 Gbit/s class).  
**AnyBlob + Umbra** is the reference design that **exploits this regime** to build a **high-bandwidth OLAP engine** directly on S3.

---

## Author’s view: why this blueprint fits OLAP but clashes with vector search

### 1. Access pattern: random read vs sequential scan

Vector search is **inherently random-access** at the storage abstraction:

- **HNSW:** graph traversal — hop-dependent, **unpredictable** node fetches.
- **IVF:** probe **selected clusters** — subset is query-dependent, not one long sequential scan.

This differs from OLAP **table/column scans** that AnyBlob optimizes (large, predictable, **8–16 MiB** sequential chunks).

**Two bad options for vector on S3 (author framing):**

| Strategy | Bandwidth | Latency / data moved | Outcome |
|----------|-----------|----------------------|---------|
| **A. On-demand (read only what the query needs)** | **Low effective bandwidth** — few GETs, small total bytes | Each GET pays **high per-request latency**; total time **latency-dominated** | Cheap in bytes, **bad for tail latency** on cold path |
| **B. “AnyBlob-style” bulk parallel read (16 MiB × hundreds of GETs)** | **High aggregate bandwidth** | Must read **much data the current query will not use** | Bandwidth high but **load time still high** — read amplification works against query latency |

**Conclusion (author):** AnyBlob’s method is a poor structural fit for vector search unless we accept **read amplification** or accept **latency-bound small reads**.

### 2. Compute: vector search is CPU-heavy

- Vector search spends substantial CPU on **high-dimensional distance computation** (and index navigation).
- OLAP scans emphasize **selection / join / aggregation** with different operator mix.
- If we also run **hundreds of parallel S3 GETs** (HTTPS + crypto), **CPU contention worsens** — network stack + ANN on the same cores.

**Conclusion (author):** High-bandwidth S3 technique **amplifies CPU pressure** on an already compute-heavy workload.

### 3. Economics: high bandwidth implies expensive hardware

Transferring the AnyBlob recipe to vector search implies provisioning:

- **Strong CPUs** (parallel TLS/HTTP + ANN),
- **High network bandwidth** (100 Gbit/s class to matter),

both are **paid resources** — the cost model may erase object-storage **$/TiB** savings for **latency-sensitive, interactive** vector serving.

---

## Consensus (for Ember paper)

- **S3 “can be fast”** is **conditional** (16 MiB-class objects, **hundreds** of parallel GETs, NIC + CPU) — not a property of a single cold index fetch.
- **AnyBlob** = existence proof for **OLAP on S3 at ~10 GB/s class**; **not** a existence proof for **cold vector query at sub-second tail** without amplification.
- Ember’s problem statement can cite this tension explicitly: **random, query-dependent reads** vs **throughput-optimal sequential bulk fetch**.

## Open questions

- Can vector systems use **bounded** bulk prefetch (e.g. cluster-level 16 MiB packs) without full AnyBlob-style over-read?
- Is there a **middle tier** (Ember storage layer) that reshapes random index access into **fewer, larger, co-located** reads without OLAP-level scan amplification?

## Relation to prior records

- Builds on [2026-06-03-milvus-terminology-s3-fundamentals-minio.md](2026-06-03-milvus-terminology-s3-fundamentals-minio.md) (S3 fundamentals, slab parallelism).
- Cites [anyblob-vldb2023.md](../related-work/anyblob-vldb2023.md) for numeric prerequisites.
- Informs `summaries/introduction-draft.md` root causes (read amplification vs latency-dominated small GETs) and Related Work positioning.

## Source (author message, 2026-06-03, paraphrased faithfully)

Author stated the four S3 bandwidth conditions, ~10 GB/s achievable when all hold, AnyBlob as OLAP proof point, and three vector-search objections: (1) random HNSW/IVF access vs sequential OLAP — on-demand = latency-bound low bandwidth vs bulk = read amplification; (2) heavy vector distance CPU + parallel GET CPU load; (3) cost of high CPU + high bandwidth hardware.
