# 2026-06-18 — SABES PDF + DES/BES naming

## SABES PDF

- 用户手动下载 → `related-work/pdfs/spatial-related/andrade-sabes-sbac-pad-2020.pdf`（自 `sabes.pdf` 重命名）
- 更新：`manifest.tsv`, `spatial-related/README.md`, partition survey §4.4 SABES entry

## DES / BES 是否标准名？

**不是通用工业术语。** 由 Andrade et al. (SBAC-PAD 2020) 在 SABES 论文中 **自行定义** 的 distributed ANN baseline 命名：

| 缩写 | 全称 | 含义 |
|------|------|------|
| **DES** | Data Equal Split | 向量 **round-robin** 均分到 worker |
| **BES** | Bucket Equal Split | **ANN bucket**（IVF/LSH 桶）均分到 worker |
| **SABES** | Spatial-Aware Bucket Equal Split | 在 BES 基础上按 **centroid 空间邻近** 分组 bucket 到 node |
| **SABES++** | （同论文） | 带 load-balance 变体 |

Survey §4.5 使用该词汇表是为了 **对齐 SABES 论文实验对比**；其他文献可能用 “hash sharding / IVF bucket sharding / all-shard probe” 描述类似 idea，但不一定叫 DES/BES。
