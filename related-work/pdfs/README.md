# Local PDF Inventory (Partition / Sharding Survey)

Working copies of papers referenced in [`summaries/partition-sharding-vector-search-survey.md`](../../summaries/partition-sharding-vector-search-survey.md).

## Usage

- Each survey entry lists **Local PDF** when a file exists here.
- **§4 papers** (centroid/bucket spatial proximity) live in [`sec4/`](sec4/) — see [`sec4/README.md`](sec4/README.md).
- If download failed (ACM 403, blog-only source, etc.), the entry says `NOT_DOWNLOADED` and keeps the online **PDF** link.
- HTML snapshots (product docs) are saved as `.pdf` for reference only — not academic PDFs.

## Status (2026-06-16)

| Status | Count | Notes |
|--------|-------|-------|
| Downloaded | ~48 peer-reviewed PDFs | arXiv, PVLDB, USENIX, author mirrors |
| NOT_DOWNLOADED | LindormVector, Distributed LSH (ACM), C-SPANN (blog), Milvus prod docs | Use online links in survey |
| HTML snapshot | Pinecone, Weaviate, Qdrant, OpenSearch | Not paper PDFs |

See [`manifest.tsv`](manifest.tsv) for title → file mapping from the batch download run.

## Naming

`{short-slug}.pdf` derived from paper title (e.g. `spann.pdf`, `analyticdb-v.pdf`).
