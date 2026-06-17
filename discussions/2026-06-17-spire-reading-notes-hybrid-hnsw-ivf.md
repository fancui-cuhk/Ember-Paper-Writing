# 2026-06-17 — SPIRE internal reading notes (hybrid HNSW + IVF)

## 主题

用户阅读 SPIRE 后判断：本质是 **IVF/partition-based**，上层用 **HNSW（proximity graph）在 centroids 上导航** 以找到正确的 IVF partition/cluster。

## 共识

**同意。** 论文明确对比 (1) sharded global HNSW（cross-node hops 主导 latency）与 (2) 纯 partition routing（fidelity loss → 多 partition probe → read/throughput 损失）。SPIRE = recursive k-means 层次 + **每层在 centroid 上建 proximity graph**，叶子 L0 才是 vector partition（SSD）。

**不是** RED-ANNS/CoTra 式「单逻辑全局图」。

## 归档

- 阅读笔记：`related-work/spire-arxiv2025.md`

## 未决

- L0 局部搜索具体实现细节待补读。
