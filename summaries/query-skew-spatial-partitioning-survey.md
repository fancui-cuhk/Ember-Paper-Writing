# Query Skew vs Spatial Partitioning in Distributed Vector Search

**Created:** 2026-06-18  
**Scope:** How systems that use **k-means / centroid / LSH / graph / spatial colocation** defend against **query skew** (hot regions in embedding space → hot nodes), as opposed to **partition-size / vector-count balance** only.

**Reading goal:** After this note, you should know what each major line of work ** says, does not say, or hand-waves** about query-load imbalance.

---

## 0. Taxonomy (how papers “handle” query skew)

| Stance | Meaning | Examples in this note |
|--------|---------|------------------------|
| **A. Ignore / silent** | Spatial placement for latency; no query-rate model | CoTra, GaussDB (prod skew), many evals |
| **B. Assume uniform queries** | Random/i.i.d. query sets in experiments | SABES, ADBV eval, GP-ANN throughput eval |
| **C. Data-size balance only** | Equal vectors/buckets/bytes per node — **not** QPS skew | SABES++, Vexless constrained k-means, IVF balance |
| **D. Query frequency in offline placement** | Profile which centroids/buckets are probed; weight placement | **SABBSR**, **SPANN §4.3**, **Pyramid**, MemANNS, BLISS |
| **E. Runtime query steering** | Route/steal queries without moving data | RED-ANNS work-stealing, Harmony load-aware routing |
| **F. Dynamic repartition on access/drift** | Split/merge/shift boundaries from access counts or stream drift | **Quake**, **Ada-IVF**, VStream DPT, LocationSpark |
| **G. Replication of hot partitions** | Duplicate shards/nodes serving same data | GP-ANN, SPIRE (orthogonal), Distributed LSH hot-range |
| **H. Fan-out-all + cache** | No subset routing; parallel scan + hot caching | Pinecone slabs, block-based serverless |
| **I. Explicitly discuss skew as open problem** | Acknowledge hot clusters under semantic routing | DistributedANN §4.4, GP-ANN related work |

**Key distinction:** **C ≠ query skew defense.** Balancing **indexed vector count** helps straglers when **every node gets similar query counts**; it does **not** help when queries cluster in one region of embedding space and **subset routing** sends traffic to the same few partitions.

---

## Internal reading notes (Ember, 2026-06-18)

| Work | Ember relevance | Takeaway |
|------|-----------------|----------|
| **SPANN §4.3** | **High** | Spatial micro-partitions → **bin-pack to machines using query-access history**. Same *pattern* as **Harmony** (workload-aware node balance) but **offline** at index build. Primary prior art for colocation + skew. → [`related-work/spann-distributed-skew.md`](../related-work/spann-distributed-skew.md) |
| **Harmony** | **High** | **Runtime** cost model + load-aware routing under skew; complements SPANN’s offline pack. |
| **FLANN distributed** | **Low (baseline)** | Same structural choice as **block-based serverless**: **fan-out query to all shards** — skew avoided by giving up subset routing. → [`related-work/flann-distributed-tpami2014.md`](../related-work/flann-distributed-tpami2014.md) |
| **Pyramid** | **Low** | Cited by SPANN; Kafka/replication straggler ops ≠ embedding query-skew placement. Superseded for our purposes by SPANN §4.3. → [`related-work/pyramid-bigdata2019.md`](../related-work/pyramid-bigdata2019.md) |

**Design split for Ember writing:**

- **Fan-out-all lane** (FLANN, block serverless, random partition all-dispatch): no routing hotspots, no geometry win.
- **Subset-routing lane** (SABES, SPANN, GP-ANN, …): needs **second-stage load placement** (SPANN bin-pack, Harmony cost model, SABBSR probe weights, GP-ANN replication).

---

## Part I — Eight requested systems (deep dive)

### 1. AnalyticDB-V (ADBV) — PVLDB 2020

**Spatial placement:** k-means **256 sharding centroids** (separate from IVF nlist); `ClusterBasedPartition(feature)` maps vectors with same cluster to same partition; query optimizer picks **N closest centroids** (not all shards).

**Stance on query skew:** **A + B + C.** Optimizes **vector-count clustering** and **partition pruning**; does **not** model query-frequency hotspots or dynamic reshuffle by QPS.

**Verbatim — why spatial partition (not hash):**

> "To solve this problem, we propose a clustering-based partition approach. For the partitioned column, we apply k-means [17] to calculate centroids according to the number of partitions. … Index building and data manipulation (§3.2) are then conducted on each individual partition. As for partition pruning, ADBV dispatches the query to N partitions, which have …"

**Verbatim — query routing (cardinality, not load):**

> "As for partition pruning, ADBV dispatches the query to N partitions, which have …" *(continues in paper §3.3 on choosing N by centroid distance / optimizer cost)*

**Verbatim — eval assumes shuffled/uniform data layout:**

> "By default, we first shuffle vectors in a dataset, … to build partitions. Otherwise, the partition scheme will be …"

> "is sampled from the original dataset uniformly at random."

**What they do *not* say:** Nothing about **hot query regions** overloading the few partitions whose centroids are nearest popular queries. Production mentions peak QPS but not skew-aware resharding.

**Attitude summary:** *Colocate for prune efficiency; balance clusters at ingest; assume representative/uniform query sampling in eval.*

**Local PDF:** [`analyticdb-v.pdf`](../related-work/pdfs/analyticdb-v.pdf)

---

### 2. GaussDB-Vector — PVLDB 2025

**Spatial placement:** Two-layer k-means IVF; **distance-based sharding** — insert/query route to DNs owning nearest cluster centroids + boundary expansion.

**Stance on query skew:** **A (+ background data redistribution).** **Query routing** uses **cardinality estimation** to pick how many DNs to search — optimizes **recall vs cost**, not **QPS balance**.

**Verbatim — IVF load balance is *data* balance:**

> "GaussDB-Vector uses two-layer clustering for the IVF index in order to balance the load of clusters."

**Verbatim — query routing problem (subset of nodes, not load):**

> "Query Routing. One of the challenges in distributed vector search is determining how many data nodes each search query should be sent to. If too few data nodes are searched, the recall of the ANN search will be very low. Conversely, if too many data nodes are involved, the performance will be suboptimal."

**Verbatim — background redistribution (data drift, not query trace):**

> "GaussDB-Vector supports periodic incremental redistribution. In the background, GaussDB-Vector samples data from each data node and computes centroids for each data node. GaussDB-Vector then relabels each vector, and if the ratio of shifted vectors exceeds a threshold (e.g. 10%), GaussDB-Vector will update the centroids and start redistribution in the background."

**What they do *not* say:** Query-rate-aware placement, replication of hot DN, or work-stealing when many queries hit same centroid neighborhood.

**Attitude summary:** *Spatial routing for recall/latency; rebalance **vectors** when cluster assignment drifts; **silent on query QPS skew**.*

**Local PDF:** [`gaussdb-vector.pdf`](../related-work/pdfs/gaussdb-vector.pdf)

---

### 3. Pinecone (Serverless / slab architecture) — product docs 2025

**Spatial placement:** **Geometric partitioning** at slab level — large slabs use **IVF** ("clusters vectors and scans only the relevant clusters per query"); smaller slabs use PQFS / SimHash.

**Stance on query skew:** **H.** Queries **fan out to all slabs** in namespace (+ memtable); mitigation is **parallelism + caching hot slabs**, not routing queries away from overloaded partitions.

**Verbatim — read path fans out everywhere:**

> "Concurrently, the query is fanned out to all slabs in the namespace."

> "Queries span all slabs, immediately picking up data that has just been written."

**Verbatim — hot data via cache tier (not repartition):**

> "Caching (performance): Frequently accessed ('hot') slabs are cached in memory and on SSD. Less-used slabs are fetched from object storage on demand."

> "Frequently accessed ('hot') slabs are cached in memory and on SSD."

**Verbatim — as usage shifts, scale resources (elasticity, not spatial rebalance):**

> "As usage shifts, resources scale without disruption."

**Trade-off:** Subset routing (IVF *within* a large slab) coexists with **mandatory cross-slab fan-out**. Popular query regions may keep hitting the same slab's IVF clusters — mitigated by **slab-level cache**, not by moving vectors across executors for QPS balance.

**Attitude summary:** *Accept fan-out; hide latency with caching and elastic compute; do not claim query-skew-aware spatial repartition at the architecture level.*

**Source:** [Pinecone slab architecture blog](https://www.pinecone.io/learn/slab-architecture/) · [`industrial/pinecone.md`](../related-work/industrial/pinecone.md)

---

### 4. Milvus — production (SIGMOD 2021 + ops docs)

**Important:** Default sharding is **hash on primary key**, **not** k-means spatial placement. Included because you asked; it illustrates **industry load balancing without geometry**.

**Stance on query skew:** **Segment-level RAM/CPU balancing** on QueryNodes — **not** embedding-hotspot aware.

**Verbatim — balance trigger is memory %, not query centroid heat:**

> "the query coord is programmed to distribute segments evenly to each query node according to the RAM usage of the nodes."

> "the query coord checks the RAM usage (in percentage) of all query nodes every 60 seconds. If either of the following conditions is satisfied, the query coord starts to balance the query load …"

**Verbatim — worst case when segments imbalanced (straggler = slowest node):**

> "The worst case could happen when a few query nodes are exhausted searching on a large amount of data, but newly created query nodes remain idle because no segment is distributed to them"

**Verbatim — distributed search waits for slowest (query skew ⇒ tail latency):**

> "Milvus fans a query out to all QueryNodes and waits for the slowest one to finish. A few overloaded pods were dragging down every single search."

*(WPS team troubleshooting blog, 2025)*

**Verbatim — hash sharding hotspot admission (manual sharding blog):**

> "Data Distribution Imbalance (Hotspots): In multi-tenant use cases, data distribution can range from hundreds to billions of vectors per tenant. This imbalance creates hotspots where certain shards become overloaded while others sit idle."

**Attitude summary:** *Auto-balance **segments** by RAM/score; **does not** colocate by embedding proximity; known production pain when segment count/CPU skew under spatially correlated query load.*

**Sources:** [Milvus load balance blog](https://milvus.io/blog/2022-02-28-how-milvus-balances-query-load-across-nodes.md) · [WPS troubleshooting](https://milvus.io/blog/troubleshooting-a-search-slowdown-after-upgrading-milvus-lessons-from-the-wps-team.md)

---

### 5. Vexless — SIGMOD 2024

**Spatial placement:** **Constrained k-means** semantic shards; orchestrator activates shards within **centroid distance threshold** of query.

**Stance on query skew:** **C + bursty-workload elasticity**, **not query-region skew.** "Load balance" = **equal memory per function**; eval stresses **sparse/bursty QPS**, not **spatially clustered queries overloading one shard**.

**Verbatim — partition imbalance = memory OOM, not QPS:**

> "this imbalance could easily lead to out-of-memory problems. Alternatively, it might require the use of additional serverless cloud functions to handle a dataset, thereby increasing unnecessary deployment costs."

> "To address this imbalanced partition issue, we utilize constrained K-Means [29] … produces the best and well-balanced result."

**Verbatim — balanced k-means may hurt recall at boundaries (trade-off stated):**

> "balanced clustering may force more boundary points between two clusters to achieve a better balance, rather than assigning them to their nearest cluster centroids."

**Verbatim — target workloads (cost under burst, not hot spatial region):**

> "Type 1: Sparse workload. The sparse workload is characterized by a low volume of queries …"

> "Type 2: Bursty workload. The key characteristic of the bursty workload is that it has peak hours and spikes."

> "online users tend to exhibit bursty behavior" *(citing web search traces)*

**Verbatim — lifetime management uses **query arrival rate**, not embedding hotspot:**

> "dynamically adjusts the keep-alive time of each cloud function thread based on the query arrival rate observed in the previous time window."

**Attitude summary:** *Balance **shard memory** for FaaS limits; optimize **cost under burst/sparsity**; **no** "hot query region → repartition centroids" mechanism.*

**Local PDF:** [`vexless.pdf`](../related-work/pdfs/vexless.pdf)

---

### 6. GP-ANN (Unleashing Graph Partitioning) — PVLDB 2025

**Spatial placement:** Graph partition + **kRt/hRt** route query to **few shards** near query.

**Stance on query skew:** **B + G + I.** Throughput eval distributes routing evenly; **explicitly** replicates popular shards; **related work** cites **SPANN §4.3** (query-access bin-packing at index build) and calls skew hard when online distribution shifts.

**Verbatim — query load imbalance as field problem (SPANN as primary prior art):**

> "We believe this is partially due to the difficulty of query load imbalance, as also briefly noted in SPANN [8]. Their approach is to load-balance queries during the partitioning step by incorporating query access frequencies. However, if the online query distribution differs or the training set is not sufficiently representative, this can still incur load imbalance."

**Verbatim — online mitigation via replication + access tracking:**

> "We argue that in a large-scale setup, where replica machines are needed for fault-tolerance and increased throughput, it is worthwhile to selectively replicate heavily loaded machines, which can also be made robust to distribution shift by tracking query access frequencies during the online stage."

**Verbatim — eval setup (even query routing; replicate hot shards):**

> "For routing, queries are distributed evenly. The machine that receives a query forwards it to the hosts that are supposed to be probed."

> "This is larger than the number of shards, as we replicate popular shards to counteract query load imbalance."

> "Replicas of the same shard evenly distribute work amongst themselves."

**Attitude summary:** *Acknowledges skew openly; defers offline placement to **SPANN §4.3**-style query-access weighting; **online** fix = **replicate hot shards** + track access — **does not** rebalance graph cut by query heat in the paper's main design.*

**See also:** **§9 SPANN** (primary source for query-access bin-packing).

**Local PDF:** [`gp-ann.pdf`](../related-work/pdfs/gp-ann.pdf) · notes [`gp-ann.md`](../related-work/gp-ann.md)

---

### 7. VStream — PVLDB 2025

**Spatial placement:** LSH → space-filling curve → contiguous partitions; query scans **limited nearby partitions**.

**Stance on query skew:** **F (partial)** — **Dynamic Partitioning Templates (DPT)** rebalance on **incoming vector counts per partition** (streaming **data drift**), **not** isolated **query** hotspot modeling.

**Verbatim — problem is data stream imbalance under static partition:**

> "these rapid data shifts can significantly cause the load imbalance of existing vector systems … due to their static partitioning strategies."

> "In streaming vector searches, the distribution of data and query streams changes continuously over time."

**Verbatim — DPT adjusts boundaries by partition **vector workload**:**

> "Dynamic Partitioning Templates (DPT) maintain and continuously update the partitions based on the workload in each partition"

> "dynamic partitioning templates which adjust the boundaries of partitions based on the number of vectors in each partition."

**Verbatim — goal stated as real-time load balancing (data-side):**

> "achieving real-time load balancing."

**Attitude summary:** *Colocate neighbors for query efficiency; **rebalance partition boundaries when data stream drifts** — closest §4 analog to **temporal** imbalance, but paper does **not** separate **query-only hotspot shift** with fixed data layout.*

**Local PDF:** [`vstream.pdf`](../related-work/pdfs/vstream.pdf)

---

### 8. SABES — SBAC-PAD 2020

**Spatial placement:** k-means **regions of coarse IVF centroids** on same worker; probe **w** nearest buckets → visit fewer QPs than BES.

**Stance on query skew:** **C (+ cites prior load-balancing literature).** **SABES++** balances **descriptor counts** per QP; **no query-frequency** in placement. Eval does not stress hot spatial query regions.

**Verbatim — why spatial colocation (inter-node traffic, not skew):**

> "SABES extends BES with a bucket distribution in which coarse centroids near in the space are more likely to be assigned to the same QP. The assumption for building SABES is that those data buckets close in the space have a high probability of being accessed by the same query."

**Verbatim — BES/SABES equal **bucket count**, not points or QPS:**

> "BES and SABES assign the same number of buckets to each Worker. However, buckets may have different number of points, creating data imbalance among Workers."

**Verbatim — SABES++ fixes **data** imbalance, not query heat:**

> "SABES++ has been proposed to extend SABES with an additional partitioning step that reduces data imbalance among QPs."

> "The goal is to group centroids near in space, but to avoid imbalance among clusters of centroids (regions)."

**Verbatim — future work acknowledges load balancing gap:**

> "our solution is the first to implement and compare multiple data partition strategies, use locality-aware data partitioning, and perform load balancing (without replication) in ANN search."

**Verbatim — related work on query load (not implemented in SABES eval):**

> "Other works [35,36] have developed load balancing … While these optimizations were not evaluated in ANN, we believe they are worth mentioning because they may inspire new solutions in the domain."

**Attitude summary:** *Assume **uniform query load** in spirit; optimize **co-probed bucket colocation**; **SABES++** = weighted k-means for **point counts**; **query-frequency** left to **SABBSR** (2024 follow-on).*

**Local PDF:** [`sabes.pdf`](../related-work/pdfs/sabes.pdf)

---

### 9. SPANN — NeurIPS 2021 (§4.3 distributed extension)

**Spatial placement:** Hierarchical **multi-constraint balanced clustering (HBC)** for IVF posting lists; centroids in memory, postings on disk; **subset routing** via query-aware dynamic pruning (nearest posting lists only). **§4.3** extends to distributed search: many small spatial partitions → **best-fit bin-packing** into M machines.

**Stance on query skew:** **D + C + I** — **Ember: primary prior art.** Offline **query-access history + best-fit bin-packing** after spatial micro-partitioning. Same *design pattern* as **Harmony** (workload-aware node balance), but **offline at pack time** vs Harmony’s **runtime cost model**. **No online repartition** if distribution shifts (GP-ANN cites this limitation).

**Reading note:** [`related-work/spann-distributed-skew.md`](../related-work/spann-distributed-skew.md)

**Verbatim — posting *size* balance (≠ query QPS, but same “balance” vocabulary):**

> "To address the posting length balance problem, we can leverage the multi-constraint balanced clustering algorithm [30] to partition the vectors evenly into multiple posting lists"

> "we introduce a hierarchical multi-constraint balanced clustering technique (Figure 3) to not only reduce the clustering time complexity … but also balance the length of posting lists."

**Verbatim — hot-spot machines + query-access bin-packing (§4.3):**

> "The only challenge for us is that there may have some hot-spot machines. Therefore, we need to balance not only the data size but also the query access in each machine to avoid the hot spots."

> "To address the hot-spot challenge, we partition the vectors into multiple small partitions (larger than machine number) and then use best-fit bin-packing algorithm [17] to pack the small partitions into large bins (the number of bins equals to the number of machines) according to the history query access distribution. By doing so, we can effectively balance not only the data size but also the queries processed on each machine."

**Verbatim — production query trace for offline placement:**

> "We conduct the experiments below based on the SPACEV1B dataset and use about 100,000 query accesses history from production as the test workload. The workload is evenly split into three sets: train, valid and test. The train set is used in offline distributed index build"

**Verbatim — eval claim (even query + data access across machines):**

> "From the result, we can see that SPANN distributes all the data and query accesses evenly into different machines."

> "random partition solution needs to dispatch the query to all 32 machines for search. Using multi-constraints balanced clustering technique can significantly reduce the number of dispatched machines to 9."

**Verbatim — cites Pyramid for distributed partial search:**

> "The approach Pyramid [15] demonstrates the power of balanced partition and partial search approach in the distributed scenario."

**Attitude summary:** *Does **not** ignore skew in §4.3 — explicitly names hot-spot machines and uses **offline query-access history + bin-packing** after spatial clustering. Still **assumes training trace represents online load**; main paper body is single-node; distributed eval is an extension section, not the primary NeurIPS contribution.*

**Source:** NeurIPS 2021 full paper §4.3 · [`spann.pdf`](../related-work/pdfs/spann.pdf) *(if present)* · [NeurIPS PDF](https://proceedings.neurips.cc/paper_files/paper/2021/file/299dc35e747eb77177d9cea10a802da2-Paper.pdf)

---

### 10. Pyramid — Big Data 2019 / arXiv:1906.10602

**Ember relevance: LOW** — catalog / cite-chain only (SPANN §4.3 references it). Superseded for query-skew placement by **SPANN §4.3**; runtime Kafka/replication is straggler ops, not embedding hotspot repartition. → [`related-work/pyramid-bigdata2019.md`](../related-work/pyramid-bigdata2019.md)

**Spatial placement:** Sample dataset → **k-means centers** → small **meta-HNSW** → **balanced graph partition** (KaHIP) → one **sub-HNSW per partition**; query visits **top-K meta-HNSW partitions** only (subset routing).

**Stance on query skew:** **D + C + E** — **two-tier** defense: (1) **offline** vertex weights from sample-query top-k hit frequency when items are hot; default assumes **uniform query access** and balances by **data count**; (2) **runtime** Kafka queue rebalancing + **replication** for stragglers (slow executor / hot sub-dataset), not embedding-space repartition.

**Verbatim — default uniform-query assumption vs hot-item weighting (Algorithm 3):**

> "Assume that each item in X is equally likely to be accessed by queries, the sub-datasets X¹, X², · · · , Xʷ should have roughly equal size to balance their workloads. Therefore, we set the weight of a vertex in G_m as the number of items it has from X′"

> "There may be scenarios that some items in X are hot (more likely to be accessed by queries) and we are given a set of sample queries. In this case, we can set the weight of a vertex in G_m as the frequency it appears in the top k similarity search results of the queries for load balancing."

> "The design of Algorithm 3 is a joint consideration of efficiency, load balancing and statistical stability."

**Verbatim — MIPS: spatial partition can create query-hot sub-dataset (then fixed by extra assignment):**

> "For query processing, the larger norm partition is very likely to contain the top-K MIPS of most queries for meta-HNSW search, which makes the worker holding the large norm partition much more heavily loaded than the other workers and may become a straggler in the system."

> "The problem that the large norm partition is hot for query processing is also solved as the query is assigned to sub-datasets that are similar to it in direction and no sub-dataset is more likely to attract queries."

**Verbatim — Kafka runtime load balance (straggler / queue skew, not spatial rebalance):**

> "Kafka is used to dispatch queries to the machines and automatically handle load balancing and fault tolerance for the message queues."

> "Stragglers are handled automatically by the message distribution mechanism of Kafka, which periodically re-balances the message queues of the executors. Therefore, the workload of a slow executor is reduced because it receives fewer query processing requests and the requests are offloaded to other executors serving the same sub-HNSW."

**Verbatim — replication for hot/slow workers:**

> "Pyramid relies on replication (which means that the same sub-HNSW is replicated on multiple workers) and Kafka to achieve robustness against straggler and failure."

**Attitude summary:** *One of the earliest explicit **hot-item / sample-query frequency** weightings in distributed HNSW colocation; default eval path assumes **i.i.d. queries** and **equal-size** partitions; **runtime** skew handled by **replication + Kafka**, not by moving vectors when query distribution drifts.*

**Source:** [arXiv:1906.10602](https://arxiv.org/pdf/1906.10602) · IEEE Big Data 2019

---

### 11. FLANN (Muja & Lowe) — *Scalable Nearest Neighbor Algorithms for High Dimensional Data*, IEEE TPAMI 2014

**Ember relevance: LOW (baseline)** — same structural choice as **block-based serverless**: equal split + **broadcast query to all shards**; skew avoided by **no subset routing**. → [`related-work/flann-distributed-tpami2014.md`](../related-work/flann-distributed-tpami2014.md) · [`serverless-block-partitioning-sigmod2025.md`](../related-work/serverless-block-partitioning-sigmod2025.md)

**Spatial placement:** Single-node **priority search k-means tree** (and other indexes). Distributed mode: **disjoint equal-size subsets**, **independent index per subset** — **not** spatial colocation for subset routing in the default design.

**Stance on query skew:** **H + B** — **broadcast query to all machines**; **equal data fractions** (optionally uneven ratios mentioned but not query-driven). **Silent on query hotspots** — throughput scales because every node does equal work per query. Contrasts with Aly et al. distributed k-d tree that routes to **subset of trees** (which would create skew under non-uniform queries).

**Verbatim — equal split + broadcast-all (no query-frequency placement):**

> "The data is distributed equally between the machines, such that for a cluster of N machines each of them will only have to index and search 1/N of the whole data set (although the ratios can be changed to have more data on some machines than others)."

> "The master server broadcasts the query to all of the processes in the cluster and then each process can run the nearest neighbor matching in parallel on its own fraction of the data."

**Verbatim — partition choice: independent subsets, query goes everywhere:**

> "When distributing a large data set for the purpose of nearest neighbor search we chose to partition the data into multiple disjoint subsets and construct independent indexes for each of those subsets. During search the query is broadcast to all the indexes and each of them performs the nearest neighbor search within its associated data."

**Verbatim — acknowledges subset routing alternative (Aly) — higher throughput, skew risk implicit:**

> "In a different approach, Aly et al. [58] introduce a distributed k-d tree implementation where they place a root k-d tree on top of all the other trees (leaf trees) with the role of selecting a subset of trees to be searched and only send the query to those trees. They show the distributed k-d tree has higher throughput compared to using independent trees, due to the fact that only a portion of the trees need to be searched by each query."

**Attitude summary:** *Classic **fan-out-all** distributed NN — **query skew is structurally avoided** (every machine touched every query) at the cost of no pruning. Does **not** discuss hot regions in embedding space; the k-means tree is a **search index**, not a cluster placement strategy for multi-node colocation.*

**Source:** [Muja & Lowe TPAMI 2014 PDF](https://www.cs.ubc.ca/research/flann/uploads/FLANN/flann_pami2014.pdf)

---

## Part II — Direct follow-on to SABES (query frequency — included because it answers the gap)

### SABBS / SABBSR — Research Square 2024 (PMC survey article)

**Stance:** **D — only §4-class paper with probe-frequency in placement objective.**

**Verbatim — problem statement:**

> "SABES groups buckets (clusters in IVFADC) together at the same node without considering either (i) the number of data objects stored in a bucket or (ii) the frequency with which a bucket is consulted when executing a given query workload."

**Verbatim — SABBSR relevance metric:**

> "SABBSR is a more sophisticated strategy that takes into consideration the number of points in a bucket and also how frequently a bucket is queried (bucket/centroid relevance) to measure this impact on processing costs."

> "In our approach, the relevance is measured as the fraction of queries that use a centroid and is calculated in an offline profiling stage in which a separate query workload is employed."

> "the weight of a centroid c as the relevance … (c.relevance()×c.size()"

**Attitude summary:** *Explicitly models **expected load = size × probe frequency**; still **offline** profiling — not online drift.*

**Local PDF:** [`sabbsr.pdf`](../related-work/pdfs/sabbsr.pdf)

---

## Part III — Extended catalog (distributed vector / ANN + query or access skew)

*Found by iterative web + repo search after the eight core systems. Grouped by primary defense mechanism.*

### Explicit access-frequency / split-merge (strongest query-skew story)

#### Quake — OSDI 2025

**Stance:** **F** — cost model on **partition size × access frequency**; split hot/large partitions, merge cold ones. *(Single-node NUMA focus in paper; access-skew maintenance is the clearest query-heat mechanism in vector ANN literature.)*

**Verbatim:**

> "The cost model tracks partition sizes and access frequencies as the workload is processed and determines which partitions are most negatively contributing to overall query latency."

> "Split Partition If a partition (l, j) is too large or **frequently accessed**, we consider splitting it into two partitions"

> "Merge Partition If a partition is **rarely accessed** and below a minimum size threshold, we consider deleting it"

> "partitions become extremely imbalanced due to the **skew in the workload**"

**PDF:** [USENIX OSDI'25](https://www.usenix.org/system/files/osdi25-mohoney.pdf)

---

#### Ada-IVF — arXiv 2024 (same authors as Quake)

**Stance:** **F** — streaming IVF maintenance; repartition partitions with high **imbalance + reconstruction error** (workload-adaptive policy).

**Verbatim:**

> "Partition imbalance and reconstruction error serve as … indicators"

> "workload-adaptive policy that identifies which partitions should be reindexed based on real-time statistics"

> "Violating partitions are split and merged with neighboring partitions"

**PDF:** [arXiv:2411.00970](https://arxiv.org/pdf/2411.00970)

---

### Query-frequency in placement (offline) or replication (online)

#### SPANN — NeurIPS 2021 §4.3

**Stance:** **D + C** — see **§9** above for full quotes. Offline **best-fit bin-packing** of spatial micro-partitions using **history query access distribution**; balances data size and query accesses per machine.

**Verbatim:**

> "we need to balance not only the data size but also the query access in each machine to avoid the hot spots."

> "use best-fit bin-packing algorithm … according to the history query access distribution."

---

#### Pyramid — Big Data 2019

**Stance:** **D + E** — see **§10** above. Default: equal-size partition weights; optional: vertex weight = **top-k hit frequency** on sample queries; runtime Kafka + replication.

**Verbatim:**

> "we can set the weight of a vertex in G_m as the frequency it appears in the top k similarity search results of the queries for load balancing."

---

#### DistributedANN — arXiv / VecDB 2025

**Stance:** **I — argues semantic clustering is hard to load-balance; chooses random KV sharding + global graph instead.**

**Verbatim:**

> "when vectors are partitioned by clustering and only a subset of partitions are used for each search, **load becomes imbalanced** … **The most popular clusters receive the most traffic**"

> "**Semantic partitioning schemes are difficult to load-balance.** Since queries will only access a subset of partitions, efficient serving requires independently scaling each partition with traffic"

> "the underlying key-value store of DistributedANN is **randomly sharded** and so receives a **predictable traffic distribution**."

**Local PDF:** [`distributedann.pdf`](../related-work/pdfs/distributedann.pdf)

---

#### RED-ANNS — PVLDB 2026

**Stance:** **E** — **affinity scheduling** improves locality; **work-stealing** when query assignment imbalanced (does not move vectors).

**Verbatim:**

> "load imbalance caused by query assignment and execution latency, RED-ANNS further incorporates an **affinity-based work stealing**"

> "Stealing Module fetches queries from heavily loaded nodes … work stealing also prioritizes queries with a higher affinity for this node"

**Local PDF:** [`red-anns.pdf`](../related-work/pdfs/red-anns.pdf)

---

#### Harmony — arXiv / SIGMOD 2025 track

**Stance:** **E + hybrid partition** — **load-aware routing** + switch to **dimension-based** partition under skew (58% gain on skewed eval). **Ember:** runtime counterpart to **SPANN §4.3** offline bin-packing — both use workload/query signals to balance nodes while keeping search structure.

**Verbatim:**

> "some nodes becoming overloaded with **'hot' partitions**, while others remain underutilized, particularly under **uneven query workloads**"

> "**load-aware routing** mechanism that minimizes communication overhead"

> "These components analyze resource usage and **query skewness**, leveraging a cost model to select between dimension-based, vector-based, or hybrid partitioning"

> "**58% improvement** over traditional distribution for **skewed loads**"

**PDF:** [arXiv:2506.14707](https://arxiv.org/pdf/2506.14707)

---

#### SPIRE — arXiv 2025

**Stance:** **G (hash placement) + mention of skew** — deliberately **hash** partition IDs to storage nodes to avoid spatial hot spots; cites caching/replication as orthogonal.

**Verbatim:**

> "mitigate **hot-spots under skewed workloads**, we assign each partition a global identifier … **uniformly shuffle** the partitions"

> "The hash function uniformly distributes partitions across the M storage nodes, mitigating hot-spots under skewed workloads"

> "workloads may exhibit skew. We model this by introducing a load-imbalance factor β … Skewness can be mitigated using … **caching** or **replication** … orthogonal to the design of SPIRE."

**Local PDF:** [`spire.pdf`](../related-work/pdfs/spire.pdf)

---

#### MemANNS — arXiv 2024 (PIM / IVFPQ)

**Stance:** **D + online scheduling** — offline placement uses **cluster size × access frequency**; online batch maps clusters to DPUs.

**Verbatim:**

> "The workload of a cluster is mainly contributed by … Let s_i denote the size of cluster i and **f_i denote the frequency with which cluster i is used to process queries**. The workload of a cluster i can be represented as **w_i = s_i * f_i**."

**PDF:** [arXiv:2410.23805](https://arxiv.org/html/2410.23805v1)

---

#### BLISS — KDD 2022 (learning-to-index)

**Stance:** **D** — iterative re-partition balances buckets while learning query→bucket mapping.

**Verbatim:**

> "re-allocating points to buckets each time in a **balanced way**"

> "We prove that BLISS achieves better recall than competing algorithms while **maintaining load balance**."

**PDF:** [KDD'22](https://gaurav16gupta.github.io/papers/BLISS_KDD2022.pdf)

---

### Distributed LSH / P2P (access-hot ranges)

#### Haghani et al. — EDBT 2009 ("LSH at Large")

**Stance:** **G** — label-density balance across peers + **hot-range replication** (Chord arcs).

**Verbatim (from repo survey / paper):**

> "Optimizes **expected bucket-label density** across peers … **hot-range replication** (Pitoura et al.) for access-heavy Chord arcs"

**Local PDF:** [`haghani-distributed-lsh-edbt-2009.pdf`](../related-work/pdfs/haghani-distributed-lsh-edbt-2009.pdf) · [OpenProceedings](https://openproceedings.org/2009/conf/edbt/HaghaniMA09.pdf)

---

### Graph / global-index lines (subset routing + skew)

#### CoTra — SIGMOD 2026 / arXiv

**Stance:** **A + B** — k-means colocation for hop locality; **equal vector count**; **no query-rate rebalance**; hot queries overload coordinator partition.

**Verbatim (from paper structure):**

> "Each query is assigned to a machine for processing" *(coordinator = partition touching most vectors)* — no mention of rebalancing by query popularity.

**Local PDF:** [`cotra.pdf`](../related-work/pdfs/cotra.pdf)

---

#### BatANN, d-HNSW, SHINE

| Paper | Query-skew stance |
|-------|-------------------|
| **BatANN** | **A** — graph/k-means placement for hop locality; no query QPS model |
| **d-HNSW** | **C** — balanced k-means sub-HNSW; recall drops if similar vectors split |
| **SHINE** | **E (partial)** — **adaptive query routing** across compute nodes for cache; graph uncut |

---

### Anti-spatial / fan-out-all (skew handled without centroid routing)

#### FLANN distributed (Muja & Lowe, TPAMI 2014)

**Stance:** **H** — broadcast every query to all MPI workers; equal data split. **Ember:** same lane as block serverless — see [`flann-distributed-tpami2014.md`](../related-work/flann-distributed-tpami2014.md).

**Verbatim:**

> "During search the query is broadcast to all the indexes and each of them performs the nearest neighbor search within its associated data."

---

#### Building Stateless Serverless Vector DBs — SIGMOD 2025

**Stance:** **H** — block partitioning avoids k-means stragglers on ingest; **queries all partitions** (no centroid filter).

**Verbatim:**

> "A block-based data partitioning scheme does **not** consider vector relationships, requiring **all partitions to be queried**"

> "Clustering algorithms like K-means can lead to **unbalanced** dataset partitions … **straggler cloud functions**"

**Note:** [`serverless-block-partitioning-sigmod2025.md`](../related-work/serverless-block-partitioning-sigmod2025.md)

---

### Spatial databases (query skew — pre-embedding, same structural problem)

#### LocationSpark — VLDB 2016 demo / Frontiers 2020

**Stance:** **F** — **query scheduler** detects hotspot partitions when **spatial queries** burst in regions; cost-model repartition.

**Verbatim:**

> "The distribution of the incoming spatial queries … changes dynamically over time, with **bursts in certain spatial regions**. Thus, evenly distributing the input data D to the various workers may result in a **load imbalance** at times."

> "LocationSpark's scheduler identifies the **skewed data partitions** based on a cost model, **repartitions**, and **redistributes** the data accordingly"

> "Query skew is very common in spatial data processing, and is found to **deteriorate the runtime performance**"

**PDF:** [VLDB'16 demo](https://www.vldb.org/pvldb/vol9/p1565-tang.pdf)

---

#### Tsunami — VLDB 2021 (multi-dimensional learned index)

**Stance:** **F (query-skew-aware partition)** — Grid Tree partitions data space using **query workload**, not just data distribution.

**Verbatim:**

> "Instead of relying on caches to reduce query time for frequently accessed keys, Tsunami **automatically partitions data space** using the Grid Tree to **account for query skew**."

**PDF:** [PVLDB 14(1):74](http://vldb.org/pvldb/vol14/p74-ding.pdf)

---

### Industry / ops (hash sharding — not spatial, but hot-shard reality)

| Source | Stance |
|--------|--------|
| **Milvus** blogs/issues | Segment **RAM** balancer; **slowest-node** tail latency; shard delegator bottlenecks |
| **Pinecone** | All-slab fan-out + **hot slab cache** |
| **Weaviate / Qdrant** | Hash/custom-key shards; tenant keys reduce fan-out — **not** embedding-hotspot routing |
| **About Vector Database** primer | Hot shard playbook: replicate, reshard, cache |
| **Production failure modes** (Aakash Sharan 2025) | Metrics: shard load skew ratio, centroid hit-rate skew |

---

### Index **construction** load balance (not query-time, but "overload" wording)

#### SOGAIC — arXiv 2025

**Stance:** Overload-aware **build-time** partition for billion-scale graph construction — **not** query routing skew.

**Verbatim:**

> "overload issues … **load-balancing task scheduling** framework"

**PDF:** [arXiv:2502.20695](https://arxiv.org/html/2502.20695v1)

---

## Part IV — Synthesis

### What most spatial-colocation papers do

1. **Optimize mean latency / network** under **subset routing** (SABES, ADBV, GP-ANN, Vexless, CoTra).
2. **Balance partition size** (vectors, buckets, bytes) — **SABES++, Vexless, SPIRE build, VStream DPT (data counts)**.
3. **Eval with random or shuffled queries** — **implicit uniform query assumption**.
4. **Silence or footnote** query skew — **GaussDB, CoTra, BatANN, ADBV prod**.

### What actually addresses **query** skew (short list)

| Mechanism | Works |
|-----------|--------|
| **Offline probe-frequency / access-weighted placement** | **SPANN §4.3** (bin-pack), **SABBSR**, MemANNS offline, BLISS |
| **Runtime workload-aware balance (Harmony-like)** | **Harmony** (cost model + routing) |
| **Fan-out-all (skew avoided, not routed)** | **FLANN MPI**, block serverless, Pinecone all-slab |
| **Online access-frequency split/merge** | **Quake**, **Ada-IVF** |
| **Query steering without data move** | **RED-ANNS** work-stealing, **Harmony** load-aware routing |
| **Replicate hot shards** | **GP-ANN**, SPIRE (orthogonal), Distributed LSH hot-range |
| **Avoid subset routing** | **Pinecone** (all slabs), **block serverless** (all blocks), **DistributedANN** (global graph + random KV) |
| **Spatial DB lineage** | **LocationSpark**, **Tsunami** (query-aware partition) |

### The "talk-away" patterns

1. **Conflate data drift with query skew** (VStream, GaussDB redistribution).
2. **Conflate memory balance with QPS balance** (Vexless, SABES++).
3. **Conflate elastic scaling with skew fix** (Pinecone, Vexless FaaS billing).
4. **Cite related load-balancing papers** without implementing (SABES §2).
5. **Acknowledge skew then choose different architecture** (DistributedANN → random sharding).
6. **Defer to orthogonal replication/caching** (SPIRE, GP-ANN online).

### Open gap (Ember-relevant)

> Spatial colocation ** amplifies** tail latency under query hotspots because the nodes that should answer fastest (local probes) become **stragglers**. The actionable **subset-routing** poles are **SPANN §4.3** (offline query-access **bin-packing**, Harmony-like) and **Harmony** (runtime cost model + load-aware routing), plus **SABBSR** / **Quake–Ada-IVF** for probe-frequency vs online access split. **Fan-out-all** baselines (**FLANN**, block serverless) sidestep skew by abandoning subset routing.

---

## Part V — Search log (exhaustiveness attempt)

**Seed set (user-requested):** ADBV, GaussDB-Vector, Pinecone, Milvus, Vexless, GP-ANN, VStream, SABES.

**User follow-up additions (2026-06-18):** SPANN §4.3, Pyramid, FLANN (Muja & Lowe TPAMI 2014).

**Expansion iterations (2026-06-18):**

1. Repo `partition-sharding-vector-search-survey.md` §4.3 query-skew table + PDF grep on all ` ` papers.
2. Web: `"query skew" distributed vector search`, `hot partition ANN`, `Milvus hot shard`, `Pinecone hot slab`.
3. Follow citations: **SPANN §4.3** (query-access bin-packing), **Pyramid** (sample-query vertex weights), GP-ANN related work; SABBSR from SABES line; DistributedANN §4.4.
4. Same research group: **Quake**, **Ada-IVF** (Mohoney et al.).
5. Parallel systems: **Harmony**, **RED-ANNS**, **SPIRE**, **MemANNS**, **BLISS**, **LocationSpark**, **Tsunami**, **Distributed LSH**.
6. Industry ops: Milvus GitHub #30978, WPS blog, About Vector Database hot-shards page, production failure modes article.
7. Negative results: **CoTra**, **BatANN**, **d-HNSW**, **SHINE** — grep silent on query QPS.
8. **SOGAIC** — build-time only; included with label.
9. Stopped when new searches returned only duplicates of above or generic KV-store hot-partition docs (DynamoDB/Spanner) without vector-ANN-specific mechanisms.

**Not claimed:** This list is exhaustive over **all arXiv preprints** — but it covers **all papers in this repo's partition/read-amp surveys**, **direct SABES successors**, **Microsoft/Google/Meta distributed ANN lines**, and **spatial DB query-skew literature** that explicitly repartitions on query/access patterns.

---

## Related repo files

- [`partition-sharding-vector-search-survey.md`](partition-sharding-vector-search-survey.md) §4.3–§4.5
- [`discussions/2026-06-17-sec4-full-rewrite-query-skew.md`](../discussions/2026-06-17-sec4-full-rewrite-query-skew.md)
- [`discussions/2026-06-18-spatial-related-shine-dhnsw-batann-sabes-bes.md`](../discussions/2026-06-18-spatial-related-shine-dhnsw-batann-sabes-bes.md)
