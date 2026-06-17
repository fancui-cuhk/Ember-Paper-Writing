# 2026-06-17 — HARMONY out of §4; PDF folder rename + read-amp collection

## 主题

1. 用户指出 HARMONY 不是 k-means 空间分区，而是 **query-aware hybrid**（vector vs dimension 模式 per query）→ 从 partition survey §4 移除。
2. 继续前序任务：`sec4/` → `spatial-related/`；为 read amplification survey 建立 `read-amp-related/` 并下载/链接 PDF。

## 要点

### HARMONY 与 §4 边界

- HARMONY（SIGMOD 2025）确有 k-means-style **vector shards**，但核心贡献是 **per-query cost model** 在 vector 与 dimension 分片间切换，以及 dimension early-stop——属于 **query-time execution strategy / load relief**，不是 SABES 式的 **placement-time centroid colocation**。
- **共识：** 保留在 survey **§2**（single logical index, externally sharded）；从 **§4 master table、§4.3 skew taxonomy、§4.4 详述** 删除；加入 **§4.6 excluded** 表并说明原因。
- PDF 留在 `related-work/pdfs/harmony.pdf`（§2 引用），不在 `spatial-related/`。

### PDF 目录

| 目录 | 用途 | 文件数 |
|------|------|--------|
| `related-work/pdfs/spatial-related/` | partition survey §4 | 10 PDF + README |
| `related-work/pdfs/read-amp-related/` | read-amp survey §5 | 18 PDF + README |
| `related-work/pdfs/` (root) | 全库 + §2 等非 §4 论文 | 含 `harmony.pdf` 等 |

### 下载状态

- **read-amp-related：** DiskANN, PageANN, Starling, OctopusANN, NaviX, PipeANN, SPANN, SPFresh, IVF-PQ, RAIRS, I-LSH, SK-LSH, learned lists, HAKES, DSANN, Milvus, Pinecone (HTML), **AnyBlob** (PVLDB 2023) — 均已就位。
- **仍未下载：** SABES (IEEE), Distributed LSH (OpenProceedings SSL), LindormVector (ACM 403) — 目标路径已在各 README / manifest 标注。

### 文档更新

- `summaries/partition-sharding-vector-search-survey.md` — §4 改为 13 篇；全部 `sec4/` → `spatial-related/`。
- `summaries/read-amplification-disk-vector-search-survey.md` — §5 Local PDF → `read-amp-related/`。
- `related-work/pdfs/manifest.tsv` — spatial + read-amp 路径；HARMONY → root。
- `related-work/pdfs/README.md`, `related-work/README.md` — 两子目录说明。

## 未决项

- LindormVector / SABES / Distributed LSH 需手动或机构访问下载。
- HARMONY 的 skew 缓解（dimension mode）是否单独开「query-aware partitioning」小节 — 未做。

## 与旧记录关系

- 延续 `2026-06-17-sec4-full-rewrite-query-skew.md`；**取代** 其中 HARMONY 作为 §4 论文的结论。
- `sec4/` 文件夹名 ** superseded by** `spatial-related/`（历史 discussion 中 `sec4` 路径仅作记录，以当前 survey 为准）。
