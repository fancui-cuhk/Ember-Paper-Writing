# 会话记录：query skew survey 补 SPANN / Pyramid / FLANN

**日期:** 2026-06-18  
**主题:** 用户指出遗漏三篇与 load balance / query skew 相关的工作

## 要点

1. **SPANN (NeurIPS 2021 §4.3)** — 此前 survey 仅通过 GP-ANN 间接引用。原文 **§4.3 Extension to distributed search** 明确：
   - hot-spot machines 问题
   - 先 spatial micro-partition，再按 **history query access distribution** 做 **best-fit bin-packing**
   - 生产 trace（~100k accesses）offline train / online test
   - 与 posting **length** balance（HBC）是不同维度

2. **Pyramid (Big Data 2019 / arXiv:1906.10602)** — meta-HNSW graph partition：
   - 默认假设 query **均匀**，vertex weight = sample 数据量
   - 可选：hot items → weight = sample query top-k **hit frequency**
   - MIPS 大 norm partition 会 query-hot → 额外 assignment + **Kafka 队列 rebalance + replication**
   - SPANN §4.3 显式 cite Pyramid

3. **FLANN (Muja & Lowe, TPAMI 2014)** — *Scalable Nearest Neighbor Algorithms for High Dimensional Data*：
   - 分布式 = **equal split + broadcast-all queries**
   - **不**做 spatial subset routing → 结构上回避 query skew，代价是无 prune
   - 对比 Aly distributed k-d tree（subset routing → 隐含 skew 风险）

## 共识

- Microsoft 线 **offline query-access placement** 的 primary sources 是 **Pyramid (2019)** 和 **SPANN §4.3 (2021)**，不是 GP-ANN 二手引用。
- SPANN 主文 single-node；§4.3 distributed 是 extension，partition survey 仍应标注 scope。

## 未决

- Pyramid PDF 是否加入 `related-work/pdfs/`（当前仅 arXiv link）
- FLANN PDF 是否加入 repo

## 更新文件

- `summaries/query-skew-spatial-partitioning-survey.md` — 新增 §9–§11，Part III stub 展开

## 关系

- 延续 `discussions/2026-06-18-query-skew-spatial-partitioning-survey.md`
