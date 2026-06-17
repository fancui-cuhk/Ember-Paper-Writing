# 2026-06-17 — Manual PDF relocation + read-amp literature expansion

## 主题

1. 用户手动下载的 AnyBlob / LindormVector PDF 从 `related-work/` 根目录迁到规范子目录。
2. 在线检索 read amplification 相关遗漏论文，扩充 read-amp survey §5。

## PDF 迁移

| 原路径 | 新路径 |
|--------|--------|
| `related-work/anyblob.pdf` | `related-work/pdfs/read-amp-related/anyblob-vldb2023.pdf` |
| `related-work/lindormvector.pdf` | `read-amp-related/lindormvector-sigmod2026.pdf` + `spatial-related/lindormvector-sigmod2026.pdf` |

根目录副本已删除；`anyblob-vldb2023.md` / `lindorm-vector-sigmod2026.md` 已更新 PDF 路径。Partition survey §2 + §4 LindormVector 改为已下载。

## 新增 read-amp 论文（在线检索，2024–2026 disk I/O / RA 向）

此前 §5 已覆盖 DiskANN–PipeANN–PageANN–Starling–OctopusANN 主线及 SPANN/SPFresh/LSH/systems 层，但 **遗漏** 以下显式 I/O/RA 工作：

| Paper | 为何 relevant | 状态 |
|-------|---------------|------|
| **Gorgeous** (arXiv 2508.15290) | 邻居 adjacency 打包进 4KB block；报告 ~39% 磁盘 I/O 减少 | PDF 已下载 |
| **LAANN** (arXiv 2606.02784) | 显式 I/O-aware look-ahead；1.59×–6.34× 更少 I/O ops | PDF 已下载 |
| **VeloANN** (arXiv 2602.22805) | 命名 read amplification；affinity page placement | PDF 已下载 |
| **B+ANN** (arXiv 2511.15557) | B+ 语义块 + 磁盘页局部性 | PDF 已下载 |
| **IISWC 2025** (AtLarge) | DiskANN SSD I/O 表征 baseline | PDF 已下载 |

## 仍有意未纳入 §5 的邻近工作（供后续）

- **FreshDiskANN / IP-DiskANN** — 更新路径，非 per-query RA 主轴（见 disk-based-ann-survey）
- **Curator, CrackIVF** — 多租户/流式 IVF，RA 次要
- **Turbopuffer** — 产品模式已在 §5.4 文字提及，无 peer-reviewed PDF
- **Azure Cosmos Sharded DiskANN** — 系统分区 + segment，在 partition survey

## 共识

- Read-amp catalog 从 ~18 篇增至 **24 篇 PDF**（含 measurement + 4 篇新 graph layout/I/O 论文）。
- 仍缺统一 **RA ratio** 跨 family benchmark — Ember eval 机会不变。

## 与旧记录关系

- 延续 `2026-06-17-harmony-sec4-removal-pdf-folders.md` 的 PDF 目录整理。
