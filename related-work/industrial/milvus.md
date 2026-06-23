# Milvus — Production Architecture Analysis

**Type:** Industrial system (docs + ops blogs; academic baseline: SIGMOD 2021 [`milvus.pdf`](../pdfs/milvus.pdf), Manu PVLDB 2022)  
**Last updated:** 2026-06-16  
**Survey category:** `distributed-anns-related`, `read-amp-related` (segment I/O)

---

## Sources (primary)

| Source | URL | Used for |
|--------|-----|----------|
| Data Processing (v3.0) | https://milvus.io/docs/data_processing.md | Insert, index build, query, handoff |
| Real-time query deep dive | https://milvus.io/blog/deep-dive-5-real-time-query.md | QueryNode load, local/global reduce |
| Tiered storage overview | https://milvus.io/docs/tiered-storage-overview.md | Cold/hot object-store path |
| WPS troubleshooting (2026) | https://milvus.io/blog/troubleshooting-a-search-slowdown-after-upgrading-milvus-lessons-from-the-wps-team.md | Fan-out straggler, segment skew |
| Query load balancing | https://milvus.io/blog/2022-02-28-how-milvus-balances-query-load-across-nodes.md | Segment RAM balancing |
| Use partition key | https://milvus.io/docs/use-partition-key.md | Scalar hash routing |
| Clustering compaction | https://milvus.io/docs/clustering-compaction.md | Optional vector/scalar segment prune |

Repo: [`summaries/cold-hot-query-paths.md`](../../summaries/cold-hot-query-paths.md), [`summaries/partition-sharding-vector-search-survey.md`](../../summaries/partition-sharding-vector-search-survey.md) §1.

---

## Executive summary

Milvus is **category #1** distributed vector DB: **hash-sharded writes → sealed segments (~512 MB) → independent ANN index per segment → scatter-gather search**. Cross-node placement is **not** embedding-space routing by default; k-means appears **inside** IVF indexes per segment. Query latency is **straggler-bound** (max QueryNode time + proxy merge). Milvus 2.6+ **tiered storage** shifts cost to object storage with **on-demand segment/chunk fetch** on cache miss.

---

## Component architecture

```
Client → Proxy → QueryCoord / DataCoord / Streaming Node(s) / QueryNode(s)
                      ↓
                 Object storage (S3/MinIO): sealed segments, binlogs, index files
                      ↓
                 etcd: metadata (segments, channels, placements)
```

| Component | Role |
|-----------|------|
| **Proxy** | API, validation, timestamp (TSO), **global reduce** on search results |
| **Streaming Node** (v3) | WAL, growing segments, TSO ordering, flush to sealed |
| **DataCoord** | Segment lifecycle, compaction, index build scheduling |
| **QueryCoord** | Segment → QueryNode placement, load balance, handoff |
| **QueryNode** | Load sealed segments + watch DML channels; **local ANN** + **local reduce** |
| **Object storage** | Durable segment field data + serialized index files |

Milvus 3.x consolidates write path on **Streaming Service** (Woodpecker WAL); query path still splits **growing** (streaming) vs **sealed** (historical on QueryNodes).

---

## Distributed storage

### Hierarchy

```
Collection
  └─ Shard (vchannel)     hash(primary key) → write channel
       └─ Partition        optional hash(partition_key) → scalar filter scope
            └─ Segment     size/time seal (~512 MB typical)
                 └─ Binlogs + index files on object storage
```

- **Shard routing:** MurmurHash3-32 (Int64 PK) or CRC32-IEEE (VarChar PK, first 100 chars); `hash % num_shards`.  
- **Partition key:** same hash family; `hash % partitions_num` (default 16). Requires filter in search — **not** ANN subset routing by embedding.  
- **Segment sealing:** Growing segments hold recent WAL data; flush produces **immutable** sealed segments in object storage.

### Persistence layout (typical)

- Field data: binlogs / Storage v2 **chunks** (~16 MB columns in tiered mode).  
- Index: **one whole file per segment per index** (HNSW, IVF, DiskANN, …) — not chunk-split.  
- Tiered storage (2.6.4+): metadata-only `load()`; **chunks** fetched on demand; **indexes** fetched whole per segment on miss.

### Tiered vs full-load

| Mode | Queryable when | Per-query S3 |
|------|----------------|--------------|
| Full-load | All segment bytes + indexes local | 0 (after load) |
| Tiered | Metadata loaded; bytes lazy | ≥1 GET per missing chunk/index file |

Published benchmark (100M×768d HNSW, 1 QueryNode): tiered first cold search P99 ~**120 ms** vs hot ~**16 ms** ([tiered blog](https://milvus.io/blog/milvus-tiered-storage-80-less-vector-search-cost-with-on-demand-hot%E2%80%93cold-data-loading.md)).

---

## Indexing

- **Build unit:** one **sealed segment** at a time on DataNode.  
- **Algorithms:** IVF (k-means **nlist inside segment**), HNSW, DiskANN, SCANN, etc. via Knowhere.  
- **HNSW constraint:** graphs **cannot merge across segments** → each segment = independent graph; global recall depends on searching **all** loaded segments.  
- **Scalar indexes:** Bloom, inverted, hash on PK/partition fields for filtered query.  
- **Clustering compaction (optional):** offline k-means on vector or scalar **clustering key** → re-seal segments + **PartitionStats** for **segment-level prune** at query time (recall-sensitive; not default).

Index build is CPU/SIMD-heavy; output serialized back to object storage.

---

## Query processing

### Search path (sealed + growing)

1. Client → **Proxy** broadcasts search to **all QueryNodes** holding collection segments (and Streaming Nodes for growing data in v3).  
2. Each **QueryNode** runs ANN on local segments (+ growing segments via DMChannel).  
3. **Local reduce:** dedupe overlap between streaming and sealed.  
4. **Proxy global reduce:** merge top-k across nodes; waits until all shard/channel coverage complete.  
5. Return to client.

**Tail latency:** `max(node latencies) + merge` — not average. WPS production: one QueryNode **5× CPU** of another → **3–5×** cluster latency regression ([WPS blog](https://milvus.io/blog/troubleshooting-a-search-slowdown-after-upgrading-milvus-lessons-from-the-wps-team.md)).

### Load balancing

- QueryCoord balances by **RAM %** on QueryNodes every ~60s.  
- Triggers: overloaded node vs underloaded new nodes; segment move causes **temporary double search** on source+dest.  
- **Not** query-centroid-aware or embedding-hotspot-aware.

### Consistency

- Timestamps (TSO) order inserts/deletes.  
- `guarantee_ts` / `service_ts` on queries for tunable consistency (see deep-dive blog).  
- No cross-segment transactional ANN — eventually consistent segment handoff.

---

## Distributed ANN pattern (survey taxonomy)

| Dimension | Milvus choice |
|-----------|---------------|
| Layer | **#1** — independent index per segment |
| Cross-node routing | **Fan-out all segments** (unless partition-key filter or clustering compaction prune) |
| Spatial sharding | **No** (default hash PK) |
| Subset routing | Partition key (scalar); clustering compaction (opt-in vector) |
| Single-query speedup | **No** — distribution for capacity + aggregate QPS |
| Read amplification | Whole segment / whole index file on cold tiered path |

---

## Production pain points (documented)

- Segment count skew → straggler QueryNodes.  
- More segments → more **merge overhead** at proxy.  
- Tiered cold path: object-store TTFB + whole index file per segment.  
- Shard delegator bottlenecks under uneven placement (GitHub #33293).

---

## Relation to Ember

- **Anti-pattern reference** for scatter-gather tail latency and **segment-granularity RA** on object storage.  
- Contrast with **subset-routing** systems (ADBV, SPANN §4.3) and **single logical graph** (DistributedANN).  
- Tiered storage = industry move toward cost/latency trade-off Ember targets at **finer storage unit** than 512 MB segment.

---

## Bibliography (cite as product + SIGMOD)

- Wang et al., *Milvus: A Purpose-Built Vector Data Management System*, SIGMOD 2021.  
- Yan et al., *Manu: A Cloud Native Vector Database Management System*, PVLDB 2022.  
- Milvus documentation and engineering blogs (URLs above), accessed 2026-06.
