# SHINE/d-HNSW/BatANN → spatial-related; serverless note; SABES vs BES

**日期：** 2026-06-18

## PDF 迁移

`shine.pdf`, `d-hnsw.pdf`, `batann.pdf` → `related-work/pdfs/spatial-related/`（与 CoTra、RED-ANNS 同目录）

已更新：manifest、spatial-related/README、partition survey §2 条目 + §4 master table + §4.4 深读条目。

## 阅读笔记

- 新增 `related-work/serverless-block-partitioning-sigmod2025.md`（SIGMOD 2025 block vs clustering serverless）

## SABES vs BES（论文 Section 5 共识）

- **DES**：向量 round-robin → 每 QP 有完整 IVF 结构但数据分散 → 查询 **broadcast 全部 QP**
- **BES**：IVF **bucket（centroid）** round-robin 均分到 QP → 查询只访问 **w 个最近 centroid 所在 QP**，最多 min(w, n) 个节点
- **BES 问题**：w 个空间上相邻的 bucket 常被 round-robin 打到 **不同 QP** → 仍要联系多个节点
- **SABES**：对 coarse centroids 做 k-means 成 **region**，整 region 给同一 QP → w 个 probe 常落在 **1–2 个 QP**
- **负载**：SABES 只均衡 **bucket 数**，不均衡 **点数** → **SABES++** 用 weighted k-means 同时考虑空间邻近与每 region 向量数；**SABES+r** 用 region 间复制进一步减 fan-out
- **未做**：query-rate / hot-region 感知的动态重分区（eval 假设均匀查询）

## 与旧记录

- 延续 prior 对话对 BatANN/d-HNSW/SHINE 是否 explicit spatial partition 的判断
- SABES DES/BES 命名见 `2026-06-18-sabes-manual-pdf-des-bes-naming.md`
