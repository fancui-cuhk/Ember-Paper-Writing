# Query skew × spatial partitioning — curated survey

**日期：** 2026-06-18  
**主题：** 用户要求对 ADBV、GaussDB-Vector、Pinecone、Milvus、Vexless、GP-ANN、VStream、SABES 做 query skew 深潜；并迭代 web 搜索补全领域文献。

## 产出

- `summaries/query-skew-spatial-partitioning-survey.md`

## 核心结论

| 类型 | 代表 |
|------|------|
| **忽略 / 均匀查询 eval** | SABES, ADBV, CoTra, GaussDB, GP-ANN eval |
| **仅数据量均衡** | SABES++, Vexless constrained k-means, VStream DPT（data drift） |
| **离线 query 频率** | **SABBSR**, SPANN（GP-ANN 引用）, MemANNS, BLISS |
| **在线 access 分裂/合并** | **Quake**, **Ada-IVF** |
| **查询调度不搬数据** | RED-ANNS work-stealing, Harmony load-aware routing |
| **复制热 shard** | GP-ANN, SPIRE（hash 打散 + 正交 replication） |
| **回避 subset routing** | Pinecone（全 slab fan-out + cache）, DistributedANN（随机 KV + 全局图） |
| **空间 DB 先例** | LocationSpark, Tsunami |

## verbatim 来源

- 本地 PDF：`spatial-related/*.pdf`, `distributedann.pdf`, `pinecone.pdf`
- Web：Pinecone slab blog, Milvus ops blogs, Quake/Harmony/SPIRE/Ada-IVF/LocationSpark/Tsunami papers

## 搜索终止条件

多轮 web + repo grep 后，新结果仅为 DynamoDB/Spanner 类 KV hot partition 或已收录论文重复 → 停止迭代。
