# 会话记录：FLANN / Pyramid / SPANN 内部阅读定位

**日期:** 2026-06-18  
**主题:** 用户对三篇工作的 Ember 相关性裁定

## 用户要点

1. **FLANN distributed** ≈ **block-based serverless (SIGMOD 2025)**：query **fan-out 到所有 shard**；不处理 embedding query skew，靠放弃 subset routing 回避。
2. **Pyramid** — **与 Ember 主线无关**（catalog / SPANN cite chain 即可）。
3. **SPANN §4.3** — **高度相关**：spatial micro-partition 后按 **query history** 做 **bin-packing** 分配到机器；与 **Harmony** 同属 workload-aware node balance（SPANN offline pack vs Harmony runtime cost model）。

## 共识

| 工作 | Ember 定位 |
|------|------------|
| SPANN §4.3 | **Primary prior art** — colocation + query-access-weighted placement |
| Harmony | **Runtime 对照** — 同 pattern，在线 cost model |
| FLANN / block | **Anti-pattern baseline** — fan-out-all |
| Pyramid | **Low** — 可保留 verbatim 条目，不作为 pole |

## 产出

- `related-work/spann-distributed-skew.md`
- `related-work/flann-distributed-tpami2014.md`
- `related-work/pyramid-bigdata2019.md`
- `summaries/query-skew-spatial-partitioning-survey.md` — Internal reading notes (Ember) 表
- `related-work/serverless-block-partitioning-sigmod2025.md` — FLANN 交叉引用

## 关系

- 延续 `discussions/2026-06-18-query-skew-spann-pyramid-flann-additions.md`
