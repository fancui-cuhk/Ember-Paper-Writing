# Industrial System Analyses (Docs-Based)

In-depth architecture notes for **production vector databases** that lack a single peer-reviewed systems paper. These files serve as the **local reference** for the PDF inventory (category `distributed-anns-related`) — same role as `*.pdf` for academic work.

| File | System | Replaces |
|------|--------|----------|
| [milvus.md](milvus.md) | Milvus 2.x/3.x production | HTML snapshot / ops-only view (SIGMOD PDF remains [`../pdfs/milvus.pdf`](../pdfs/milvus.pdf) for read-amp) |
| [pinecone.md](pinecone.md) | Pinecone Serverless | `pinecone.pdf` HTML snapshot |
| [weaviate.md](weaviate.md) | Weaviate cluster | `weaviate.pdf` HTML snapshot |
| [qdrant.md](qdrant.md) | Qdrant distributed | `qdrant.pdf` HTML snapshot |
| [opensearch-knn.md](opensearch-knn.md) | OpenSearch k-NN | `opensearch-k-nn.pdf` HTML snapshot |
| [cockroachdb-c-spann.md](cockroachdb-c-spann.md) | CockroachDB C-SPANN | failed blog PDF download |

**Category index:** [`../pdfs/categories.md`](../pdfs/categories.md)

**Method:** Synthesized from official docs, engineering blogs, and repo surveys (`cold-hot-query-paths`, `partition-sharding-vector-search-survey`). No vendor benchmark numbers invented.
