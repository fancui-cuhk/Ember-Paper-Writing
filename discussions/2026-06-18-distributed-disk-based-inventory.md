# 会话记录：repo 内 distributed + disk-based 工作清单

**日期:** 2026-06-18  
**主题:** 用户问 repo 相关工作中哪些同时是 distributed 且 disk-based

## 结论摘要

分三档写入 `summaries/disk-based-ann-survey.md` §9.6：

- **Tier A（显式 distributed disk ANN）：** BatANN, DSANN, DistributedANN, SPANN §4.3
- **Tier B（分布式 DB + SSD/DFS/持久页）：** SPIRE, GaussDB-Vector, LindormVector, Cosmos Sharded DiskANN, C-SPANN, VStream
- **Tier C（分布式 + object/block storage 冷路径）：** Milvus, Pinecone, block serverless SIGMOD'25, Turbopuffer, OpenSearch k-NN

**排除：** Harmony, CoTra, RED-ANNS, HAKES, GP-ANN, Vexless, SABES 等 — 分布式但以内存图/IVF 为主。

## 产出

- `summaries/disk-based-ann-survey.md` §9.6 表格
