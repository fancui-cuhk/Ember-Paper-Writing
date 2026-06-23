# Distributed ANNS Taxonomy — Fact-Check Run (2026-06-23)

**Scope:** All 33 in-tree systems + 18 out-of-tree assets + online gap search.  
**Related:** [`summaries/distributed-anns-taxonomy.md`](../summaries/distributed-anns-taxonomy.md)

---

## Executive summary

| Verdict | Count |
|---------|-------|
| **Confirmed correct** | 24 systems (all axes) |
| **Correct org + index; placement wording or nuance** | 5 (ADBV, GaussDB, HAKES, Pyramid, Pinecone) |
| **Should move leaf** | 1 (**SPIRE** — spatial → **random** at node tier) |
| **Correct leaf; index-type footnote needed** | 1 (**Harmony** — not classic IVF; dual offline B_vec + runtime steering) |
| **Correct leaf; sidenote only** | 1 (**CXL-ANNS** — dimension split, not spatial colocation) |
| **Missing from tree (survey-worthy)** | 4 products + 2 research papers |
| **Single-node → include distributed extension** | **SPANN §4.3 only** (already in tree); SPFresh/C-SPANN as **lineage** not separate leaf |

**Action before survey writing:** Fix **SPIRE** placement; add **Turbopuffer**; decide on MemANNS/BLISS; add Harmony/SPANN footnotes on offline vs runtime workload signals.

**Applied (2026-06-23):** All above applied in `summaries/distributed-anns-taxonomy.md` v6.

---

## 1. Per-system verification (in-tree)

### Placement-first → Random placement — **ALL CONFIRMED**

| System | Evidence |
|--------|----------|
| Milvus | Hash PK → segments → local ANN; fan-out-all (`industrial/milvus.md`) |
| Weaviate | Murmur-3 UUID hash → one HNSW/shard |
| Qdrant | Fixed shard count; hash default; tenant shard key = scalar |
| OpenSearch k-NN | Lucene doc routing → segment-local ANN |
| FLANN | Equal disjoint subsets; broadcast to all MPI workers (`flann-distributed-tpami2014.md`) |
| Block-Serverless | Fixed byte blocks; map all partitions (`serverless-block-partitioning-sigmod2025.md`) |
| Auncel | Uniform random worker shards → local IVF |
| Vearch | Document/table partition on PartitionServers |
| Cosmos DB | One DiskANN per physical partition; optional tenant shard key |

### Placement-first → Spatial colocation placement

| System | Verdict | Evidence |
|--------|---------|----------|
| **ADBV** | ✓ org; ✓ placement (subset routing) | k-means **256/512 outer shards** → local IVFPQ; optimizer picks N nearest centroids. **Not** neighbor-centroid colocation on same node (survey §4: “Placement-time colocation: No”). Still **spatial colocation placement** (geometry defines shard boundaries). Survey Category #2 tag is **stale** — functionally **placement-first**. |
| **GaussDB** | ✓ org; ✓ placement | Two-layer k-means IVF + distance-based DN sharding; subset centroid routing. Same Category #2 mismatch as ADBV. |
| **GP-ANN** | ✓ | METIS graph partition → local HNSW + kRt/hRt routing |
| **Vexless** | ✓ | Constrained k-means per function (~1.5 GB) |
| **VStream** | ✓ | LSH + space-filling curve; dynamic partition templates |
| **Pyramid** | ✓ spatial; **weak query-history** | Equal-size graph partitions → sub-HNSW. Optional **sample-query** vertex weights (Algorithm 3) — weaker than SPANN §4.3 bin-pack. Repo: keep **spatial**; footnote optional sample weights (`pyramid-bigdata2019.md`). |
| **Pinecone** | ✓ with nuance | Geometric compaction **within namespace** → slab-local indexes. Cross-tenant = namespace isolation (random w.r.t. vectors). Slab geometry = spatial colocation at storage unit. |

### Index-first → Graph

| System | Verdict | Evidence |
|--------|---------|----------|
| **DistributedANN** | ✓ random | Single global DiskANN on **randomly sharded KV** (paper verbatim: “key-value store … randomly sharded”) |
| **CoTra** | ✓ spatial | Balanced k-means + global Vamana; Pull-Push RDMA |
| **RED-ANNS** | ✓ spatial | Logically full graph; locality-aware placement + affinity scheduling |
| **SHINE** | ✓ spatial | Graph-preserving HNSW on disaggregated memory |
| **d-HNSW** | ✓ spatial | Balanced k-means sub-HNSW + meta-HNSW routing |
| **BatANN** | ✓ spatial | Global disk Vamana; graph partition (Gottesburen) + baton-passing |
| **DSANN** | ✓ spatial | Point Aggregation Graph; cluster layer for DFS placement |
| **CXL-ANNS** | ⚠ **index ✓; placement wrong in tree** | Billion-point **global graph** in CXL pool. Placement = **column-wise dimension sharding** across expanders — **not** embedding-neighborhood spatial colocation. → **Sidenote** (dimension split); tree currently says spatial — **fix to sidenote or remove mislabel**. |

### Index-first → IVF/LSH → Spatial colocation

| System | Verdict | Evidence |
|--------|---------|----------|
| **SABES** | ✓ | IVFADC buckets; k-means groups co-probed buckets onto nodes |
| **LindormVector** | ✓ | IVFPQ posting lists aligned to Lindorm KV ranges |
| **C-SPANN** | ✓ | SPANN/SPFresh k-means tree → SQL ranges |
| **HAKES** | ⚠ partial | Index-first two-stage IVF+PQ **filter** (k-means) ✓. RefineWorkers sharded by **vector ID** (random). Survey §4.6 excludes from spatial cluster sharding. **Keep spatial** for filter-tier story; footnote refine tier. |
| **Haghani LSH** | ✓ | ξ-mapping colocates LSH bucket labels on Chord peers |
| **SPIRE** | ❌ **WRONG PLACEMENT** | Index = hierarchical IVF + graph-on-centroids ✓. **Node placement:** paper assigns partition global IDs and **“uniformly shuffle[s] … hash function uniformly distributes partitions across M storage nodes”** to mitigate hot spots (`query-skew` survey §SPIRE verbatim). k-means is **index geometry**, not **node colocation**. → **Move to Random placement** under IVF/LSH (or split: spatial index / random node map — survey uses one leaf → **Random**). |

### Index-first → IVF/LSH → Query-history-aware placement

| System | Verdict | Evidence |
|--------|---------|----------|
| **SPANN** | ✓ | §4.3: spatial micro-partitions → **best-fit bin-packing** using **~100k production query-access history** (NeurIPS 2021 §4.3 verbatim). Main body single-node; distributed extension is **index-first** (one logical IVF namespace, pack posting groups to machines). **Correct in tree.** |
| **SABBSR** | ✓ | Bucket grouping weights **probe frequency × size** (successor to SABES) |
| **Harmony** | ⚠ **leaf OK; index-type weak** | **Query-history-aware:** cost model refines plan using **past queries** / imbalance I(π), adjusts B_vec (user + paper). **Also** runtime vector vs dimension mode + load-aware routing (query-skew stance **E**). Not the same mechanism as SPANN offline bin-pack, but **both** use workload signals — tree leaf defensible with footnote. **Index type:** not inverted-file or LSH bucket structure; **distributed partial-distance** over vector shards + dimension blocks. Closest bucket: **IVF/LSH** only by “cluster-friendly distribution” quote — **needs footnote**. |

---

## 2. Out-of-tree assets — single-node with distributed story

| Paper | Distributed claim in paper/repo? | Include in survey? |
|-------|----------------------------------|-------------------|
| **SPANN §4.3** | Yes — bin-pack + subset machine dispatch | **In tree ✓** |
| **SPANN main (§4.2)** | Single workstation eval only | Out of tree; cite as algorithm baseline |
| **SPFresh** | Single-node incremental IVF partitions | **Lineage** for C-SPANN/Turbopuffer; not a cluster architecture |
| **DiskANN** | k-means for **build parallelism only**; no query routing | Out of tree; **DistributedANN** is the distributed extension (in tree) |
| **FreshDiskANN** | Single-node streaming graph | Out of tree |
| **ScaNN** | No multi-node design; SPIRE cites distributed overhead | Out of tree |
| **Ada-IVF** | Single-node access-driven **local** repartition | Out of tree; conceptual neighbor to SABBSR/Quake |
| **Quake** | Single-node NUMA access-skew split/merge | Out of tree |
| **CrackIVF, RAIRS, VHP, SK-LSH, I-LSH, LEQAT, multi-probe LSH** | Single-node | Out of tree ✓ |
| **Starling, PipeANN, PageANN, OctopusANN** | Single-node disk graph; Starling targets **segment** I/O not cluster | Out of tree ✓ |
| **Curator, PASE** | Single-node | Out of tree ✓ |
| **NetANNS** | Assumes partitioned backend | Overlay only ✓ |

**No additional single-node paper in corpus claims a SPANN §4.3-style distributed extension** beyond SPANN itself and engineering products (C-SPANN, Turbopuffer) built on SPFresh/SPANN lineage.

---

## 3. Missing systems (online + repo surveys)

| System | Type | Suggested leaf | In `distributed-anns-related`? |
|--------|------|----------------|-------------------------------|
| **Turbopuffer** | Product | Index-first → IVF/LSH → Spatial colocation | No (in `cold-hot-query-paths`, partition survey product table) |
| **Elasticsearch dense_vector** | Product | Placement-first → Random | No (survey product table only) |
| **pgvector + Citus** | Product | Placement-first → Random | No |
| **LanceDB** | Product | Placement-first → Random (fragments) or Index-first IVF | No |
| **MemANNS** | Research (PIM) | Index-first → IVF/LSH → Query-history-aware | No (query-skew survey only) |
| **BLISS** | Research (learned index) | Index-first → IVF/LSH → Query-history-aware | No |
| **Cloudflare Vectorize** | Product (blog) | Index-first → IVF/LSH → Spatial | No |

**Already in tree, confirmed from web:** DistributedANN, RED-ANNS, BatANN, SPIRE, Harmony — no major 2025–2026 distributed ANNS paper missing beyond above.

---

## 4. Recommended tree corrections (priority order)

1. **SPIRE:** Index-first → IVF/LSH → **Random placement** (hash shuffle to storage nodes).
2. **CXL-ANNS:** Index-first → Graph → keep in tree but **drop “spatial colocation”** — document as **dimension-split sidenote** only (same as Harmony dimension mode).
3. **Harmony:** Keep query-history-aware; add footnote: **runtime routing + offline B_vec refinement**; index type = distributed cluster scan, not classic IVF.
4. **Add Turbopuffer** to Index-first → IVF/LSH → Spatial (product survey section).
5. **ADBV / GaussDB:** Keep placement-first; fix partition survey Category #2 tags when syncing docs.
6. **Pyramid:** Optional footnote for sample-query vertex weights; do **not** move to query-history-aware (superseded by SPANN §4.3 per repo).

---

## 5. Corrected decision tree (proposed)

```
DISTRIBUTED ANNS
├─ PLACEMENT-FIRST
│   ├─ Random placement
│   │     Milvus, Weaviate, Qdrant, OpenSearch k-NN, FLANN,
│   │     Block-Serverless, Auncel, Vearch, Cosmos DB
│   └─ Spatial colocation placement
│         ADBV, GaussDB, GP-ANN, Vexless, VStream, Pyramid, Pinecone
└─ INDEX-FIRST
    ├─ Graph
    │   ├─ Random placement → DistributedANN
    │   └─ Spatial colocation placement
    │         CoTra, RED-ANNS, SHINE, d-HNSW, BatANN, DSANN
    │         (+ CXL-ANNS: dimension-split sidenote, not spatial)
    └─ IVF / LSH
        ├─ Random placement → SPIRE
        ├─ Spatial colocation placement
        │     SABES, LindormVector, C-SPANN, HAKES, Haghani, Turbopuffer*
        └─ Query-history-aware placement
              SPANN, SABBSR, Harmony
        (* Turbopuffer proposed add)
```

---

## 6. Open questions for author

1. **Harmony index type:** Force IVF/LSH vs new “distributed scan” footnote?
2. **MemANNS / BLISS:** Add to tree (download PDFs) or related-work footnote only?
3. **Industrial duplicates:** Elasticsearch = OpenSearch cousin — include or merge?
4. **SPIRE:** Single leaf “Random” vs prose splitting index-geometry (spatial) from node map (random)?
