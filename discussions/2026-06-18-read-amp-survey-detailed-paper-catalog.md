# 会话记录：Read-amp survey §5 详细论文目录

**日期:** 2026-06-18  
**主题:** 用户要求 disk-based read amplification 论文清单，每篇说明如何 **define** 和 **handle** RA

## 完成

- 重写 `summaries/read-amplification-disk-vector-search-survey.md` **§5**
- 24 篇（graph / IVF / LSH / hybrid / systems / measurement）
- 每篇统一字段：RA definition · root cause (§3) · handling · metrics
- 增加 §5 快速索引表

## 要点

- 仅 **LAANN、VeloANN** 等少数论文显式用 “read amplification”；多数用 proxy（overlap ratio、I/O ops、disk-access count、posting bytes）
- Graph 线：OR(G) + layout shuffle（Starling/OctopusANN）vs page-node（PageANN）vs latency pipeline（PipeANN，可能增 RA）
- IVF 线：bounded postings + prune（SPANN）vs redundant list I/O（RAIRS）
- Systems 线：segment/slab 粒度 vs nprobe（Milvus/Pinecone）— Ember 冷路径论点

## 关系

- 延续 `discussions/2026-06-16-read-amplification-disk-vector-search-survey.md`
- PDFs：`related-work/pdfs/read-amp-related/`
