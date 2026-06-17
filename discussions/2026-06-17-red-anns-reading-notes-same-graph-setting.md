# 2026-06-17 — RED-ANNS internal reading notes

## 主题

用户阅读 RED-ANNS 后判断：与 DistributedANN / SHINE / CoTra 等 **同一问题、同一 distributed HNSW（单逻辑图）设定** —— partition one graph，而非 per-segment 独立建图 + MapReduce merge。

## 要点

- 论文自称 **full-GPS** vs baseline **sub-GPS**（segment 独立 HNSW + broadcast + reduce）。
- 归档为 **Category #2**（single logical index, externally sharded）；§4 的 locality-aware placement 是次要角度。
- 阅读笔记：`related-work/red-anns-vldb2026.md`

## 未决

- 与 CoTra/SHINE/d-HNSW 同硬件 direct comparison 未在笔记中展开。
