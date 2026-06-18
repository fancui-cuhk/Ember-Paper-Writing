# 会话记录：为何 spatial colocation 值得做（尽管 query skew）

**日期:** 2026-06-18  
**主题:** 用户追问 — colocation 明显导致 load imbalance，为何还要做？关键优势是什么？

## 核心结论

Colocation **不是为了 load balance**；是为了让 **分布式 ANN 的搜索结构与数据布局对齐**。ANN（IVF nprobe、LSH multi-probe、graph hop）天然只访问 embedding 空间中的 **邻近分区**；若邻近分区散落在所有节点，每次查询都要付 **跨节点网络/磁盘/merge** 成本。

Papers 赌的是：**subset routing 省下的 mean latency / bandwidth / compute** > **query skew 带来的 tail 风险**。Skew 被当作 **二阶问题**（SABBSR、SPANN bin-pack、work-stealing），或在极端规模被 **放弃 colocation**（DistributedANN → random KV）。

## 四类动机（见 partition survey §4.3 展开）

1. **Multi-probe 共访** — SABES：邻近 bucket 同 query 共访 → 同节点
2. **Subset routing** — ADBV/GaussDB/Vexless/SPANN：少触节点 vs hash fan-out
3. **Graph hop locality** — CoTra ~73.8% local；BatANN 减 cross-server hop
4. **Recall @ 边界** — GaussDB boundary expansion；SPIRE partition density

## 反例

- **DistributedANN** 明确拒绝 semantic cluster routing（hot cluster + subset routing = 运维噩梦）
- **FLANN / block serverless / Milvus hash** — fan-out-all，用并行换 skew 可预测性

## 关系

- `summaries/partition-sharding-vector-search-survey.md` §4.3
- `summaries/query-skew-spatial-partitioning-survey.md`
