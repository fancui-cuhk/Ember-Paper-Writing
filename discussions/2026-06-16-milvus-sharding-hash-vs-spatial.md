# Session: Milvus Sharding — Hash Functions vs Spatial Clustering

- **Date:** 2026-06-16
- **Topic Tags:** `milvus`, `sharding`, `hash`, `murmur3`, `crc32`, `partition-key`, `clustering-compaction`, `scatter-gather`, `spatial-partitioning`

## User questions

1. What hash functions does Milvus use for sharding?
2. Why not clustering-based (spatial) partitioning so segments can be pruned, instead of all segments participating in every query?

## Key answers

### Hash functions (from `pkg/util/typeutil/hash.go`, v2.5.16)

| Routing layer | Key | Hash | Route |
|---------------|-----|------|-------|
| **Shard (vchannel)** | Primary key | Int64 → **MurmurHash3-32** (little-endian bytes); VarChar → **CRC32-IEEE** on first 100 chars | `hash % num_shards` |
| **Partition (partition key mode)** | Scalar partition-key field | Same as above (Int64/VarChar) | `hash % partitions_num` (default 16) |

Shard routing binds one PK to one write stream (insert/delete consistency). Partition-key routing is **scalar filter pruning**, not embedding-space routing.

### Three layers — do not conflate

1. **Shard** — hash(PK), write parallelism
2. **Partition** — optional hash(partition_key), tenant/filter scope
3. **Segment** — ~512 MB sealed unit; **not** assigned by vector similarity by default

**Inside each segment:** independent **IVF k-means** or **HNSW** — algorithm-internal pruning only.

### Default query path = fan-out all loaded segments

Proxy → all QueryNodes with relevant segments → local reduce → global merge. Tail = slowest node (WPS blog).

### Why not default spatial cross-node sharding

- **HNSW cannot merge** across segments/nodes → N independent graphs + scatter-gather
- **Dynamic ingest:** global k-means repartition expensive/unstable (SIGMOD 2025 serverless block vs clustering)
- **Recall:** pruning shards/segments loses boundary neighbors unless duplication (SPANN closure, GaussDB boundary expansion)
- **Query skew:** subset routing → hot spatial regions (DistributedANN §4.4 rejects semantic cluster routing for this reason)
- **Product choice:** hash = O(1), predictable write path, segment RAM balancing — trades geometry for ops simplicity

### Emerging: Clustering Compaction (2.6+, opt-in)

- Scalar or **vector** clustering key; k-means analyze + **PartitionStats** for **segment pruning**
- `dataCoord.compaction.clustering.autoEnable: false` by default
- Community reports recall sensitivity (GitHub #36349); not default production path

## Sources

- [Milvus hash.go v2.5.16](https://github.com/milvus-io/milvus/blob/v2.5.16/pkg/util/typeutil/hash.go)
- [Data Processing docs](https://milvus.io/docs/data_processing.md)
- [Use Partition Key](https://milvus.io/docs/use-partition-key.md)
- [Clustering Compaction](https://milvus.io/docs/clustering-compaction.md)
- [Zilliz sharding blog](https://medium.com/@zilliz_learn/sharding-partitioning-and-segments-getting-the-most-from-your-database-d744be168893)
- Repo: `summaries/partition-sharding-vector-search-survey.md`, `summaries/query-skew-spatial-partitioning-survey.md`

## Relation to prior records

- Extends fan-out/straggler discussion (WPS blog, query-skew survey §4 Milvus)
- Clarifies k-means in Milvus is **in-segment IVF**, not cross-node placement (corrects common conflation)

## Open questions

- Production adoption rate of vector clustering compaction + recall tuning guidelines
- Whether Milvus 3.x streaming architecture changes segment-pruning semantics
