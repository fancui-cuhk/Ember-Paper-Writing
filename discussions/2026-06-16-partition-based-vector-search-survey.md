# 2026-06-16 Partition & Sharding Vector Search Survey

- **Date:** 2026-06-16
- **Topic:** Partition/sharding in vector search — papers and industry systems
- **Output:** `summaries/partition-sharding-vector-search-survey.md`

## Changes (latest turn)

User requested:
1. **Do not organize by conference** — venue is a field on each entry only
2. **All English**
3. Per paper: **title, PDF link, abstract, understanding (problem + technique)**, plus hardware / partitioning / rationale

Survey restructured into **three partitioning layers** (see summary overview):

1. Data partition → independent index per shard
2. Single logical index, externally sharded across nodes
3. Partition-intrinsic indexes (IVF / LSH / tree)

Plus **§4** cross-cutting analysis on IVF centroid spatial proximity for external sharding.

**Superseded:** prior four-way split (partition-first / graph-first+shard-local / global graph / adaptive).

See `discussions/2026-06-16-partition-survey-taxonomy-abstract-fix.md` for details.

## Open

- ICDE long tail
- arXiv venue tracking (SPIRE, CoTra, SHINE, Ada-IVF)
- SABES original paper venue mapping
