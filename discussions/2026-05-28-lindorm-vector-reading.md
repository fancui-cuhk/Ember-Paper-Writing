# Session: LindormVector Paper Reading & related-work Folder

- **Date**: 2026-05-28
- **Topic Tags**: `related-work`, `lindorm-vector`, `IVFPQ`, `disaggregation`, `cold-query`

## User Intent

1. Add a **`related-work/`** folder for recording papers/systems read for the Ember paper.
2. Log today's reading: **LindormVector** (ACM DOI [10.1145/3788853](https://dl.acm.org/doi/10.1145/3788853)).
3. Produce a **summary plus author commentary** on architecture similarity to Ember and whether it addresses our cold-query / tail-latency thesis.

## User-Provided Technical Notes (Preserved)

- **Architecture**: Compute–storage disaggregation; **LDServer** (compute) holds IVFPQ structure; **HNSW on centroids** always on LDServer; **compressed posting lists** primarily on **LindormDFS** (storage), with LDServer memory as cache.
- **Implication**: Same cold-start / tail-latency vulnerability when posting lists are not cached — unless LindormDFS uses very fast, cheap media (SSD + RDMA), in which case cold path may be acceptable.
- **Conclusion**: IVF choice aligns with Ember; paper **does not solve** Ember's target problem (cold query / tail under disaggregation).

## Discussion Points and Consensus

### Repository changes

- Created `related-work/` with `README.md` (naming convention + index).
- Created `related-work/lindorm-vector-sigmod2026.md` (bibliography, summary, cold-path analysis, Ember comparison, author notes, paper-use table).
- Updated `WORKFLOW.md`, `README.md`, `.cursor/rules/paper-writing-workflow.mdc` to include `related-work/` in agent workflow.

### Paper metadata (web-verified)

- **Title**: *LindormVector: A Distributed Vector Engine on a Cloud-Native Multi-Model NoSQL Database*
- **Venue**: SIGMOD 2026 Industry (accepted per [sigmod.org listing](https://2026.sigmod.org/sigmod_industry_papers.shtml))
- **Primary emphasis (public sources)**: Multi-model integration, hybrid scalar/vector/full-text retrieval, CBO/RBO optimizer — not cold-query tail latency.

### Alignment with existing Ember narrative

- **Consistent** with `summaries/cloud-hosted-vs-cloud-native.md`: disaggregated systems cold-fetch index data on cache miss.
- **No conflict** with introduction framework: Lindorm fills a different contribution slot (hybrid multi-model DB) while leaving cold/tail gap for Ember.
- **New nuance for Related Work**: LindormDFS may use **fast storage tiers**; comparisons must state storage/media assumptions to avoid unfair overstatement.

## Open Questions

1. Full PDF: verify cold-latency numbers, posting-list granularity, and experimental storage configuration.
2. Exact LDServer cache policy (eviction, region load, cold tenant) — not in user notes yet.
3. Should `summaries/related-systems.md` be created once 3+ related-work entries exist?

## Relationship to Previous Records

- **Extends**: `2026-05-28-introduction-framework.md` — adds first concrete **related system** entry for comparison table / Related Work.
- **Reinforces**: `summaries/cloud-hosted-vs-cloud-native.md` — Lindorm as another instance of cache + remote storage, not a root-cause solution.

## Action Items

1. User to skim PDF for cold-latency / cache-miss experiments (checklist in related-work note).
2. When writing Related Work, cite Lindorm as **architectural peer, different optimization target**.
3. Optional: add Lindorm row to introduction comparison table after storage assumptions clarified.
