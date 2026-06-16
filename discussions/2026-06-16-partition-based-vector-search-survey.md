# 2026-06-16 Partition & Sharding Vector Search Survey

- **Date:** 2026-06-16
- **Topic:** Partition/sharding in vector search — papers and industry systems
- **Output:** `summaries/partition-sharding-vector-search-survey.md`

## Changes (latest turn)

User requested:
1. **Do not organize by conference** — venue is a field on each entry only
2. **All English**
3. Per paper: **title, PDF link, abstract, understanding (problem + technique)**, plus hardware / partitioning / rationale

Survey restructured into sections by **architectural pattern**:
- A. Partition-first (IVF / LSH / geometric / block)
- B. Graph-first with shard-local indexes
- C. Global / logical graph across machines
- D. Adaptive / dynamic partitioning
- E. Foundational (IVF-PQ)
- F. Industry systems

## Open

- ICDE long tail
- arXiv venue tracking (SPIRE, CoTra, SHINE, Ada-IVF)
- SABES original paper venue mapping
