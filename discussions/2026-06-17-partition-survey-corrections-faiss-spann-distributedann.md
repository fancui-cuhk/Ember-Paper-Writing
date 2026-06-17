# 2026-06-16 — Partition Survey Corrections (Faiss, SPANN, DistributedANN)

- **Date:** 2026-06-16
- **Topic:** User review of `summaries/partition-sharding-vector-search-survey.md`

## Questions

1. What Bing-scale workload traits force **one sharded global graph** vs partition + independent indexes?
2. Is Faiss "distributed IVF" official? → **No** — wiki case study / bench script, not library API.
3. SPANN paper vs production — paper is **single-node only**; do not blur for research survey.

## Consensus

- **DistributedANN #2** justified when: unified 50B+ corpus, graph index, many small partitions for failover, throughput/recall vs prior partition-and-route (P·log(N/P) vs log N).
- **Removed** Faiss 1T vectors wiki from survey entirely.
- **SPANN:** category **#3 only**; hardware = workstation in paper; production multi-machine dispatch **out of scope** for paper citations.
- Added **"When category #2 beats #1"** table under DistributedANN entry.

## Open

- Whether to add footnote in `distributed_vector_search_related_work.md` that Faiss multi-node is wiki-only (separate file; user asked survey fix only).
