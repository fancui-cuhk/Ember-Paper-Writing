# When to Parallelize — Classical Laws and Distributed Vector Search

**Date:** 2026-06-18  
**PDFs:** [`pdfs/`](pdfs/) (category `parallel-or-not` in [`pdfs/categories.md`](pdfs/categories.md))  
**Context:** User observation that parallel computing is not always suitable — if per-node work is small, a single node (or small fan-out) avoids network/scheduling tax; if work is super-intensive, parallelism pays off again. This report collects top-tier classical support and maps it to distributed ANN (SABES, SPANN, fan-out-all baselines).

---

## 1. The common wisdom (one sentence)

**Parallelize only when the parallelizable work dominates fixed serial, coordination, and communication costs** — otherwise you pay overhead without proportional gain.

This is not one paper but a stack of results from Amdahl (1967) through Gustafson (1988), Hill–Marty (2008), Gunther’s USL, Lamport’s distributed-systems work, and Blumofe–Leiserson’s work-stealing analysis.

---

## 2. Seminal papers (downloaded)

### 2.1 Gene Amdahl — *Validity of the Single Processor Approach* (1967)

**Local PDF:** `pdfs/amdahl-1967.pdf`

For fraction *s* of inherently serial work and parallel fraction *p* = 1 − *s* on *N* processors:

\[
\text{Speedup} = \frac{1}{s + p/N}
\]

Even a **small serial part** (*s* → 0 still nonzero) caps speedup at **1/s**. The lesson for systems: **do not add processors (or remote nodes) expecting linear gain** if a non-shrinking serial segment remains — routing, merge, RTT, consensus, lock acquisition.

**Distributed ANN mapping:** Coordinator scatter/gather, cross-shard merge, and per-hop network latency are **serial or sub-linearly shrinkable** components. Fan-out to 32 shards when 4 suffice hits Amdahl-like limits unless each shard does enough useful ANN work.

---

### 2.2 John Gustafson — *Reevaluating Amdahl’s Law* (CACM 1988)

**Local PDF:** `pdfs/gustafson-1988.pdf` (+ extended memo `gustafson-scaled-sized-1988.pdf`)

Gustafson (Sandia, 1024-processor hypercube) argues Amdahl’s **fixed-problem-size** model is the wrong question for HPC: users **scale the problem** with machine size. **Scaled speedup**:

\[
\text{Speedup} = s + p \cdot N \quad (\text{with } s + p = 1 \text{ on the parallel machine})
\]

This is much more optimistic than Amdahl when **parallel work grows with *N***.

**Distributed ANN mapping:** A **billion-vector index** on a growing cluster is a Gustafson setting — more machines hold more data and can serve more **independent posting-list / bucket reads in parallel**. Parallelism wins when **per-query parallel work** (bytes scanned, distance comps, disk pages) is large relative to fixed coordination. It does **not** justify fan-out-all on a **fixed small query** where each node does trivial work.

---

### 2.3 Mark Hill & Michael Marty — *Amdahl’s Law in the Multicore Era* (IEEE Computer 2008)

**Local PDF:** `pdfs/hill-marty-2008.pdf`

Extends Amdahl with a **hardware budget model**: a chip has *n* base core equivalents (BCEs); using *r* BCEs for one “fatter” core yields perf(*r*) (Pollack’s rule: sublinear). Designers choose **many small cores vs fewer large cores** under a fixed resource envelope.

**Distributed ANN mapping:** Analogous trade at cluster scale — **fewer nodes with more local index per node** vs **many nodes with thin shards**. Subset routing (SABES, SPANN §4.3) is “use fewer, fatter visits”: colocate co-probed buckets so each touched node does **meaningful** work. Hash fan-out-all is “many thin cores” — good for **throughput under uniform load**, bad when **per-node work ≪ network tax**.

---

### 2.4 Neil Gunther — Universal Scalability Law (2009 book excerpt)

**Local PDF:** `pdfs/gunther-usl-2009.pdf`

\[
C(N) = \frac{N}{1 + \alpha(N-1) + \beta N(N-1)}
\]

Beyond Amdahl’s serial term, **α** captures **contention** (shared bottlenecks) and **β** captures **coherence** (cross-node synchronization). Throughput can **peak and decline** as *N* grows — “retrograde scalability.”

**Distributed ANN mapping:** Coordinator merge queues, NIC saturation, metadata storms, and hot-shard contention are α/β effects. **Query skew onto colocated hot buckets** is a coherence/contention story; **SABBSR**, SPANN bin-packing, and replication exist to push the peak rightward, not to eliminate the USL curve.

---

### 2.5 Leslie Lamport — Logical Time (1978) & Paxos Made Simple (2001)

**Local PDFs:** `lamport-clocks-1978.pdf`, `lamport-paxos-simple-2001.pdf`  
**Turing Award:** 2013 (distributed and concurrent systems).

- **Logical clocks:** In distributed systems there is **no global clock**; establishing order requires **extra messages and metadata** — pure overhead for applications that do not need it.
- **Paxos:** **Consensus is expensive**; Lamport’s later simplification is pedagogical, but the point stands: **coordination has a lower bound on message rounds and persistence**.

**Distributed ANN mapping:** Every cross-node query path pays **ordering, consistency, and failure-handling** costs that a single-node ANN avoids. This supports **“touch fewer nodes”** as a latency strategy, not only a load-balance strategy.

*(Lamport’s 1979 sequential-consistency note is cited in the literature but not in our PDF set — see README online link.)*

---

### 2.6 Blumofe & Leiserson — *Work Stealing* (JACM 1999)

**Local PDF:** `pdfs/blumofe-leiserson-work-stealing-1999.pdf`  
**Leiserson:** Turing Award 2019 (parallel and distributed computing).

Provably efficient scheduling for strict multithreaded DAGs:

- **Time:** \(T_1/P + O(T_\infty)\) — near-linear when **total work** \(T_1\) is large vs critical path \(T_\infty\).
- **Communication:** \(O(P T_\infty (1 + n_d) S_{\max})\) — **more processors ⇒ more steal traffic**.

Work-stealing is **the best practical scheduler**, yet it still says: **parallelism without sufficient per-task work increases scheduling/communication overhead**.

**Distributed ANN mapping:** Fan-out-all (FLANN distributed, block serverless, hash sharding) resembles **eager parallel launch**; subset routing resembles **steal only when local work is exhausted** — probe locally colocated buckets first, expand fan-out only if recall demands it.

---

## 3. Cited but not locally archived

| Paper | Why it matters |
|-------|----------------|
| **Hennessy & Patterson**, *A New Golden Age for Computer Architecture*, CACM 2019 (Turing 2017) | Names **Amdahl’s Law** as a continuing constraint post-Moore; argues for **domain-specific** architectures when general parallel scaling stalls — analogous to **specialized subset routing** vs generic fan-out. [ACM link](https://cacm.acm.org/research/a-new-golden-age-for-computer-architecture/) |
| **Lamport 1979**, sequential consistency | Defines the **memory-coherence tax** multiprocessors pay; same spirit as cross-node cache/coherence in distributed indexes. |

---

## 4. Synthesis for Ember / distributed vector DB

| Regime | Favor | Classical anchor |
|--------|-------|------------------|
| Small index, cheap query, tight tail-SLO | **1 node or minimal fan-out** | Amdahl serial fraction; USL α at low *N* |
| Large index, heavy nprobe / disk IO, recall-bound | **Parallel across nodes** if per-shard work ≫ RTT + merge | Gustafson scaled work; Blumofe \(T_1/P\) term |
| Same buckets probed, different node placement | **Colocate co-probed buckets** (SABES) vs round-robin (BES) | Hill–Marty “fewer, fatter” visits; Lamport coordination cost |
| Predictable load > minimum latency | **Fan-out-all + cache** (FLANN, Pinecone slabs) | Accept USL contention; buy skew predictability |

**SABES vs BES (from repo reading):** Same *w* buckets probed, but SABES touches **~4 nodes vs ~12** (32-node cluster) — not less ANN work, **less parallel overhead**. That is exactly the “don’t parallelize across 12 nodes when 4 suffice” principle, grounded in decades of parallel computing theory.

**Decision heuristic (paper-backed):**

\[
\text{Parallelize across } N \text{ nodes iff } \sum_i W_i \;\gg\; C_{\text{fixed}} + C_{\text{net}}(N) + C_{\text{merge}}(N)
\]

where \(W_i\) is useful ANN work on node *i*, and \(C_{\text{net}}\) grows with fan-out (USL β, Blumofe communication term).

---

## 5. Related repo entries

- [`summaries/partition-sharding-vector-search-survey.md`](../summaries/partition-sharding-vector-search-survey.md) §4 — spatial colocation, SABES
- [`summaries/query-skew-spatial-partitioning-survey.md`](../summaries/query-skew-spatial-partitioning-survey.md) — fan-out-all vs subset routing
- [`discussions/2026-06-18-why-spatial-colocation-despite-skew.md`](../discussions/2026-06-18-why-spatial-colocation-despite-skew.md)

---

## 6. Open questions

- Quantitative **crossover point**: at what nprobe / bytes-read does fan-out to *N* shards beat colocated subset on real cloud networks?
- Does **Gustafson scaling** (index grows with cluster) change Ember’s tail-latency story vs fixed-size benchmarks?
- Can USL parameters (α, β) be **measured** on Milvus/SPANN-style deployments for a motivation figure?
