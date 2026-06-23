# Session: Industrial System Architecture Analyses (Docs-Based)

- **Date:** 2026-06-16
- **Topic Tags:** `industrial`, `milvus`, `pinecone`, `weaviate`, `qdrant`, `opensearch`, `cockroachdb`, `distributed-anns`

## User request

For industrial systems without proper PDFs, read official docs/blogs and write in-depth MD analyses (distributed storage, indexing, query processing). Use these MD files as the category inventory asset instead of HTML snapshot PDFs. Remove temporary scripts when done.

## Deliverables

| File | System |
|------|--------|
| `related-work/industrial/milvus.md` | Milvus 2.x/3.x production (+ SIGMOD PDF retained separately) |
| `related-work/industrial/pinecone.md` | Pinecone Serverless |
| `related-work/industrial/weaviate.md` | Weaviate cluster |
| `related-work/industrial/qdrant.md` | Qdrant distributed |
| `related-work/industrial/opensearch-knn.md` | OpenSearch k-NN |
| `related-work/industrial/cockroachdb-c-spann.md` | CockroachDB C-SPANN (preview) |

## Category index updates

- `categories.tsv`: replaced `pinecone.pdf`, `weaviate.pdf`, `qdrant.pdf`, `opensearch-k-nn.pdf` with `industrial/*.md`
- `distributed-anns-related`: `milvus.pdf` → `industrial/milvus.md` (read-amp keeps `milvus.pdf` SIGMOD)
- Added `industrial/cockroachdb-c-spann.md`
- Regenerated `categories.md` (71 assets)
- Deleted HTML snapshot PDFs from `pdfs/`
- Removed `build_categories.py` (temp script from prior session)

## Cross-ref updates

- `partition-sharding-vector-search-survey.md` §1 production entries
- `read-amplification-disk-vector-search-survey.md` Pinecone entry
- `query-skew-spatial-partitioning-survey.md`, `disk-based-ann-survey.md`

## Consensus

Industrial systems in this repo are **category #1 scatter-gather** except C-SPANN (**range-mapped SPANN tree**). None use default embedding-space cross-node sharding; subset routing is namespace/shard-key/scalar filter tier.

## Relation

- Continues `discussions/2026-06-16-pdf-flat-layout-category-index.md`
- Complements `summaries/cold-hot-query-paths.md`
