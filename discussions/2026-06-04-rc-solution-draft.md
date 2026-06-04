# Root Causes & Solution — Ember Paper Draft

> **Status**: **Obsolete** (2026-06-04).  
> This draft has been moved to `discussions/` and its content has been incorporated into the Introduction draft. It is retained for historical reference only. Do not edit or reference as authoritative.

---

> **Original Status**: Draft for discussion. Solution section describes intended design; numbers are targets unless marked otherwise.

---

## 6. Root Causes of High Tail Latency in Cloud-Native Vector Databases

Cloud-native vector databases suffer from unacceptable cold-start query latency due to three structural root causes.

**RC1 — Disaggregation places network data transfer on the cold-start query critical path.**

- In cloud-native architectures, compute and storage are physically separated. Index data lives durably on remote object storage.
- A cold-start query — one whose required index is not cached at the compute tier — cannot begin execution until the required index data has been transferred over the network from object storage.
- This network transfer sits directly on the query critical path: every cold-start query waits for it, unconditionally.
- Compare with co-located architectures: even if data is disk-resident, a local disk read costs ~10ms (HDD) — not a 100–200ms network round-trip. Disaggregation replaces local I/O with remote I/O, and that remote I/O is on the critical path.
- Disaggregation is the architectural source of TCO, elasticity, and availability advantages in cloud-native systems — these advantages cannot be retained without accepting this penalty for cold-start queries.

**RC2 — Vector index access patterns are fundamentally mismatched with object storage.**

- Object stores (S3-class) are optimized for high aggregate throughput on large, bulk reads — not for fine-grained, random, query-dependent access.
- Vector index search requires exactly this kind of access: IVF probes a small subset of clusters (nprobe ≪ nlist); HNSW traverses a data-dependent sequence of graph nodes.
- This mismatch forces any cloud-native vector database to choose between two strategies, each with a fundamental limitation:

  - **Strategy A — Whole-Index Loading** (Milvus full-load mode):
    - Load the entire index into compute memory before serving any query.
    - Problem: **Read amplification** — only a small fraction of the index (the probed clusters) is needed per query, but 100% must be transferred over the network. At billion scale, this means transferring tens of gigabytes to use a few megabytes.

  - **Strategy B — On-Demand Cluster Loading** (Turbopuffer):
    - Identify the relevant clusters at query time and fetch only those from object storage.
    - Eliminates read amplification, and cluster fetches can be issued in parallel.
    - Problem: **Index type restriction** — this strategy requires knowing *which* data to fetch before fetching it. IVF-class indexes support this (clusters are identified by centroid search before any data fetch). Graph-based indexes (HNSW) do not: each traversal hop is data-dependent, so the next node to fetch is unknown until the current node has been read. HNSW cannot be served by Strategy B.
    - Turbopuffer explicitly targets ≤3 S3 round-trips per cold-start query, yet cold-start p50 remains ~874ms at 1M-vector scale. At billion scale, maintaining recall requires larger nprobe, which increases the total data fetched per query; S3's aggregate bandwidth becomes the bottleneck, and cold-start latency grows further.

- Neither strategy achieves sub-second cold-start tail latency at billion scale, and neither supports the full spectrum of index types.

**RC3 — Object storage opacity forecloses access-aware optimization.**

- Object stores expose no interface for placement hints, co-location of frequently co-accessed clusters, access-frequency metadata, or prefetch directives.
- Yet access patterns in vector databases are highly structured and skewed:
  - Across tenants: some tenants generate continuous query traffic; many are cold archives rarely queried.
  - Within an index: query load concentrates on popular vector neighborhoods; most clusters are rarely probed.
- A system that remains on top of S3 cannot express: *"these two clusters are always probed together — store them adjacent"* or *"this tenant's index is frequently cold-started — prioritize its placement."*
- This opacity structurally forecloses the class of access-aware optimizations that would otherwise mitigate RC1 and RC2. Any solution that stays within the S3 abstraction is fundamentally limited.
- This motivates Ember's core architectural decision: **replace the storage abstraction itself** with a custom layer that is transparent to the query engine and co-optimizable with it.

---

## 7. Ember: Design Overview

### Core Idea: Cold-Start Query Pushdown

- Ember introduces a **custom distributed storage layer** built on commodity hardware (HDD-class media).
- Unlike S3, this layer is co-designed with the query engine. Storage nodes run ANN search directly on the data they hold.
- Cold-start queries are **pushed down to the storage tier** — executed entirely where the data resides, with no index data transferred to the compute tier.
- This preserves the full disaggregation architecture for hot queries:
  - **Hot queries** (index cached at compute tier): served at the compute tier as in any cloud-native system. TCO, elasticity, and multi-tenancy advantages are fully preserved.
  - **Cold-start queries** (index not in compute cache): pushed down and executed locally at the storage tier. Network is eliminated from the cold-start query critical path. Latency is now bounded by local disk access (~10ms on HDD), not S3 round-trip (~100–200ms).
- The concept of pushing computation to storage nodes is well-established in disaggregated database systems [computation pushdown: Redshift Spectrum, S3 Select, VLDB 2025 disaggregation survey]. **However, applying this idea to vector index search on commodity HDD storage introduces a set of challenges not addressed by prior work.** Ember's primary technical contribution is solving these challenges.

### Why Cold-Start Query Pushdown Is Non-Trivial for Vector Search

**Challenge 1 — HDD Random I/O Bottleneck.**
- HDDs deliver ~100–200 MB/s sequential throughput but only ~100–200 random IOPS (~10ms per random seek).
- A naively laid-out IVF index on HDD — where each cluster's posting list is stored at an arbitrary disk location — requires one random seek per probed cluster.
- At nprobe = 100 and 10ms per seek, disk I/O alone costs ~1 second — already violating the sub-second target before any ANN computation.
- Achieving sub-second cold-start query performance on HDDs requires co-designing index layout and I/O access patterns to maximize sequential access.

**Challenge 2 — Load Imbalance and Hotspot Formation.**
- Query load is highly skewed: some tenants generate continuous traffic; most are cold archives.
- Within a single index, popular vector neighborhoods attract disproportionately more queries.
- A static data layout will concentrate I/O and compute load on a small number of storage nodes, forming hotspots that inflate cold-start query tail latency under concurrent load.

**Challenge 3 — Fault Tolerance on Commodity Hardware.**
- HDDs have significantly higher annual failure rates than NVMe SSDs or cloud object storage (which provides 11-nines durability).
- A storage layer built on HDDs must provide replication and failure-recovery mechanisms to ensure data durability without sacrificing query latency.
- Replication decisions interact directly with placement and hotspot mitigation, requiring joint design.

**Challenge 4 — Distributed Index Maintenance.**
- Vector indexes must be updated incrementally as data is inserted or deleted.
- Maintaining a distributed, block-structured IVF index across storage nodes — while preserving the layout properties that enable efficient cold-start queries — requires careful coordination.
- Naive strategies (e.g., full index rebuild on write) are too slow for interactive workloads; incremental update strategies must not degrade the sequential-access layout that makes HDD performance acceptable.

**Challenge 5 — Multi-Tenant Interference.**
- Multiple tenants share the storage tier. A burst of cold-start queries from one tenant must not significantly degrade cold-start query latency for other tenants.
- Resource isolation at the storage tier — for both disk I/O bandwidth and ANN compute — is necessary but adds design complexity.

### Ember's Solutions

**Solution 1 — Access-Aligned Block Index.**
- Ember organizes the IVF index into fixed-size blocks (10–100 MB), where block boundaries are aligned to IVF cluster boundaries: each block contains one or more complete posting lists.
- A cold-start query probing k clusters maps directly to fetching the corresponding blocks — no wasted data, no read amplification.
- Each block is stored contiguously on disk, so fetching it is a single large sequential read, exploiting HDD sequential bandwidth rather than suffering its random IOPS limitation.
- This directly addresses Challenge 1.

**Solution 2 — Distributed Scatter-Gather Execution.**
- A cold-start query is decomposed into sub-queries, each targeting the storage node(s) holding the relevant blocks.
- A storage-tier coordinator scatters these sub-queries to responsible nodes in parallel; each node executes local ANN search; results are gathered and merged at the coordinator.
- Distributes both I/O load and ANN compute evenly across storage nodes, preventing single-node bottlenecks under high cold-start query volume.
- Directly addresses Challenge 2 and Challenge 5.

**Solution 3 — Adaptive Block Placement and Replication.**
- Ember tracks block-level access frequency across tenants and index regions.
- Frequently co-accessed blocks are placed on the same or nearby nodes (reducing cross-node fetches within a single query).
- Hot blocks — from high-QPS tenants or popular vector neighborhoods — are replicated to additional nodes for load distribution.
- Replication across failure domains provides fault tolerance for HDD reliability (Challenge 3), with replication factor tuned to access frequency.
- Directly addresses Challenge 2, Challenge 3.

---

## Anticipated Reviewer Challenges

> *Internal note — not for paper inclusion.*

**"Caching or prefetching on S3 can solve the cold-start query problem."**
Caching defers cold-start queries, it does not eliminate them. Prefetching requires knowledge of which tenants will go cold and when — S3 provides no interface to express this. RC3 (opacity) addresses this directly.

**"Turbopuffer already solves this with ≤3 S3 round-trips."**
Turbopuffer's cold-start p50 is ~874ms at 1M-vector scale with an optimized design. At billion scale, nprobe grows with dataset size, increasing total data fetched per query; S3 bandwidth becomes the bottleneck. More fundamentally, Turbopuffer remains bounded by S3's latency floor (~100ms per GET). Ember replaces that floor with local HDD latency (~10ms) through cold-start query pushdown.

**"HDDs are slow and regressive — SSDs are cheap now."**
HDDs offer ~10× lower cost per GB than NVMe SSDs, which directly preserves the TCO advantage. At billion scale, storage cost dominates. Ember demonstrates sub-second cold-start query latency on HDDs with access-aligned layout — NVMe is not required to meet the latency target. SSD deployment is also supported and provides further latency headroom.

**"Computation pushdown is well-known (Redshift Spectrum, S3 Select). What is new?"**
Prior pushdown work targets OLAP operators (filter, project, aggregate) on simple tabular data. Vector ANN search is different: it is iterative and data-dependent, and must run over a carefully laid-out index on commodity hardware. The co-design of IVF index layout, HDD I/O strategy, and distributed execution for this workload is the novel contribution — not the pushdown concept itself.

**"RC1 implies disaggregation is the problem. Why not just use cloud-hosted?"**
Cloud-hosted eliminates cold-start queries but forces indexes onto expensive RAM/SSD and couples compute and storage scaling — returning to the TCO and scalability failures documented in §3. Ember's answer is co-location at the *storage tier* (cheap HDD), not the compute tier. Hot queries still enjoy disaggregation advantages; only cold-start queries are pushed down. This is a strict improvement over both alternatives.

**"Does cold-start query pushdown support HNSW, or only IVF?"**
Ember's access-aligned block index is designed around IVF-class partitioning, which allows pre-identification of the relevant clusters before any data fetch. HNSW's data-dependent traversal is not directly compatible with this design. Supporting HNSW for cold-start queries remains an open challenge; Ember currently targets IVF-class indexes for the cold path.
