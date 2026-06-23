# Session: PDF Flat Layout + Category Index for Distributed ANNS Lit Review

- **Date:** 2026-06-16
- **Topic Tags:** `pdfs`, `categories`, `distributed-anns-related`, `reorganization`, `literature-review`

## User request

1. Download distributed ANNS papers (include distributable single-node work like SPANN).
2. Flatten `related-work/pdfs/` — one shared folder, no duplicate copies per category.
3. Central **category file** with forward (category → PDFs) and inverted (PDF → categories) index.
4. Categories: `parallel-or-not`, `read-amp-related`, `spatial-related`, **`distributed-anns-related`** (new).
5. Update cross-refs in existing markdown.

## What we did

### PDF consolidation

- Merged all PDFs from `spatial-related/`, `read-amp-related/`, `parallel-or-not/` into flat `related-work/pdfs/` (**70** unique files).
- On duplicate filenames, kept the **larger** file (e.g. `starling.pdf` from read-amp copy).
- Renamed `pase-sigmod2020.pdf` → `pase.pdf`.

### New downloads (distributed lit review)

| File | Source |
|------|--------|
| `flann-pami2014.pdf` | UBC FLANN mirror |
| `pyramid-bigdata2019.pdf` | arXiv 1906.10602 |
| `haghani-distributed-lsh-edbt-2009.pdf` | OpenProceedings (curl `-k`, expired cert) |

### Category index

- **`categories.tsv`** — machine-readable `(category, pdf)` rows.
- **`categories.md`** — human-readable forward + inverted index.
- **`build_categories.py`** — source of truth; re-run after adding PDFs.

| Category | Count |
|----------|-------|
| `distributed-anns-related` | 52 |
| `read-amp-related` | 24 |
| `spatial-related` | 15 |
| `parallel-or-not` | 8 |

**Unassigned:** `serf.pdf` (range-filter ANN; no distributed angle in repo surveys).

Multi-category examples: `spann.pdf` → distributed + read-amp; `lindorm-vector.pdf` → distributed + spatial + read-amp; `milvus.pdf` → distributed + read-amp.

### Cross-ref updates

- Bulk-replaced `spatial-related/`, `read-amp-related/`, `parallel-or-not/` paths in summaries, related-work notes, discussions.
- Updated `partition-sharding-vector-search-survey.md`, `read-amplification-disk-vector-search-survey.md`, `query-skew-spatial-partitioning-survey.md`, `distributed_vector_search_related_work.md`, `related-work/README.md`, `pdfs/README.md`.
- Haghani PDF marked **downloaded** (was NOT_DOWNLOADED).
- Old subfolders retain stub `README.md` → `../categories.md`.

## Still NOT on disk

- C-SPANN / CockroachDB blog
- PASE (ACM 403)
- Milvus prod docs HTML snapshot

## Relation to prior records

- Supersedes per-folder organization from `discussions/2026-06-17-harmony-sec4-removal-pdf-folders.md` and `2026-06-18-spatial-related-shine-dhnsw-batann-sabes-bes.md`.
- Extends partition + distributed surveys for user's **distributed ANNS literature review**.

## Open questions

- Add `serf.pdf` to a category if range-filter + distributed becomes relevant.
- DistVS (NSDI 2026) — no stable open PDF URL found; add when available.
- Optional: dedicated `summaries/distributed-anns-literature-review.md` skeleton keyed off `categories.md`.
