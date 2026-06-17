# Partition Survey §4 Revision — SABES Papers & Placement Table

**Date:** 2026-06-17  
**Topic:** Fix §4 (Centroid / Bucket Spatial Proximity), clarify SABES/SABBS/SABBSR, repair PDFs  
**Related:** `summaries/partition-sharding-vector-search-survey.md` §4 · prior `discussions/2026-06-17-partition-survey-corrections-faiss-spann-distributedann.md`

## User requests (session 1)

1. Explain SABES / SABBS / SABBSR — corrupted `sabes-pmc-survey.pdf`
2. Remove distributed Faiss from survey context
3. Remove **Hash / block (baseline)** row — not centroid-spatial
4. Replace vague **"Colocate adjacent centroids?"** column

## User requests (session 2)

1. Delete **all** distributed Faiss occurrences repo-wide
2. Remove **HAKES** from §4 (filter/refine colocation ≠ centroid spatial sharding)
3. Clarify CoTra **"primary partition"** — not replication; per-query coordinator on hottest k-means shard
4. Rewrite CoTra core insight: k-means colocates similar vectors to cut **inter-node graph hops**
5. Add **hardware** (single/multi-node, memory/SSD) and **load balance** for all §4 systems
6. Explain Distributed LSH overlay: locality vs **bucket-label / peer storage load**
7. Remove **LEQAT** from §4
8. Delete **Summary for Ember** table
9. Move SABES/SABBS full paper entries **above** comparison table (dedupe bottom)

## Changes made

### §4 structure

- Full **SABES** and **SABBS/SABBSR** bibliographic entries immediately after baseline naming table.
- Comparison table adds **Hardware** column; deep dives add hardware + load balance per system.
- Removed: HAKES, LEQAT, Ember summary table, duplicate bottom entries.

### CoTra wording

- **No vector replication.** "Hottest partition" = query coordinator for that query only.
- Core insight: balanced k-means keeps graph neighbors local → fewer RDMA hops during traversal.

### Distributed LSH

- Load balance = **predicted bucket-label density** across peers + `hash(l)` spread + dynamic local DHT split/join + optional Chord hot-range replication — **not** balancing bucket point counts directly.

### Distributed Faiss scrub

- Removed from `distributed_vector_search_related_work.md`, `related-work/pdfs/README.md`, manifest.
- Updated stale discussion taxonomy rows.

## Open items

- SABES 2020 local PDF (IEEE paywall).
- Distributed LSH PDF via openproceedings (automated download failed SSL).

## 2026-06-17 (sec4 PDF folder)

- Moved §4 paper PDFs to `related-work/pdfs/sec4/`; updated survey + manifest links.
- Deleted corrupted HTML masquerading as `andrade-sabes-sbac-pad-2020.pdf`.
