# 2026-06-16 — LEANN (MLSys 2026) Reading Note

- **Date:** 2026-06-16
- **Topic tags:** `LEANN`, `storage-efficiency`, `RAG`, `edge`, `out-of-scope`, `interesting-reference`
- **Relation:** Not core to Ember (partition/sharding / tail latency on object storage), but worth recording as a provocative storage–compute trade-off in vector search.

---

## Bibliography

| Field | Value |
|-------|--------|
| **Title** | LEANN: A Low-Storage Overhead Vector Index |
| **Authors** | Yichuan Wang, Zhifei Li, Shu Liu, Yongji Wu, Ziming Mao, Yilong Zhao, Xiao Yan, Zhiying Xu, Yang Zhou, Ion Stoica, Sewon Min, Matei Zaharia, Joseph Gonzalez |
| **Venue** | MLSys 2026 (Oral, Best Paper Session) |
| **PDF** | [OpenReview](https://openreview.net/pdf?id=e8Dp5QkFxP) · [arXiv:2506.08276](https://arxiv.org/pdf/2506.08276) |
| **Code** | [StarTrail-org/LEANN](https://github.com/StarTrail-org/LEANN) |

---

## Abstract (from OpenReview)

Embedding-based vector search underpins many important applications, such as recommendation and retrieval-augmented generation (RAG). It relies on vector indices to enable efficient search. However, these indices require storing high-dimensional embeddings and large index metadata, whose total size can be several times larger than the original data (e.g., text chunks). Such high storage overhead makes it difficult, or even impractical, to deploy vector search on personal devices or large-scale datasets.

To tackle this problem, we propose **LEANN**, a storage-efficient index for vector search that **recomputes embeddings on the fly** instead of storing them, and **compresses state-of-the-art proximity graph indices** while preserving search accuracy. LEANN delivers high-quality vector search while using only a fraction of the storage (e.g., **5% of the original data**) and supporting storage-efficient index construction and updates. On real-world benchmarks, LEANN reduces index size by up to **50×** compared with conventional indices, while maintaining SOTA accuracy and comparable latency for RAG applications.

**TL;DR:** Cuts vector index storage by up to 50× by recomputing embeddings on the fly—without hurting RAG accuracy or latency.

---

## Understanding

### Problem

Conventional vector indexes (HNSW, DiskANN, etc.) store **full-precision embeddings** plus **graph metadata**. For text RAG, index size often **exceeds the raw corpus** (e.g., HNSW index ~188 GB vs. ~76 GB dataset in public benchmarks). This blocks deployment on laptops, mobile/edge devices, and cost-sensitive large-scale storage.

### Technique (from paper + open-source implementation)

1. **Graph-based selective recomputation** — Store a **pruned proximity graph** (HNSW or DiskANN backend) but **not** most embedding vectors. During search, run the embedding model only for nodes on the **active search path**, then rerank with fresh vectors.
2. **High-degree preserving pruning** — Compress the graph by keeping important **hub** nodes/edges; drop redundant connections to shrink metadata.
3. **Compact graph representation** — CSR-style layout; optional `--no-compact` / `--no-recompute` modes trade storage for latency.
4. **Dynamic batching** — Batch on-demand embedding computation (often via a local embedding server) to amortize GPU/CPU cost.
5. **Two-level search** — Graph traversal with PQ or approximate signals first; full recomputation only where needed for final ranking.

Reported outcome: index footprint can drop to **~3–5%** of a traditional index (up to **50× / 97%** reduction in some benchmarks) with **comparable RAG end-to-end quality** because LLM generation dominates latency; query search adds ~2–3× latency vs. stored embeddings but still small vs. total RAG pipeline.

### Why it is interesting (even if out of our field)

- **Orthogonal axis to Ember:** Ember asks how to **search cold data fast on cheap HDD/object storage**; LEANN asks how to **avoid storing embeddings at all** by paying **compute at query time**.
- **Shared tension:** Both papers react to the fact that **vector indexes are huge** relative to raw data—but LEANN removes stored vectors; Ember keeps disaggregated storage and attacks **tail latency / I/O amplification**.
- **RAG latency accounting:** LEANN explicitly argues that **2–3× search slowdown** is acceptable when LLM inference is 10×+ of retrieval—useful framing if we discuss cold-query latency vs. end-to-end pipeline.
- **Partition hook (weak):** DiskANN backend with `recompute=True` mentions **build-time graph partitioning** for smaller indexes—tangential to our partition survey, not the paper’s main story.

---

## Relation to Ember

| Dimension | LEANN | Ember |
|-----------|-------|-------|
| Primary goal | Minimize **index storage** | Minimize **cold-query tail latency** on disaggregated storage |
| Storage assumption | Keep **raw text** + tiny graph; drop stored embeddings | Keep **IVF/graph structures** on object/HDD; optimize **what/when to fetch** |
| Compute assumption | Willing to **re-encode** chunks at query time | **Limited compute** on storage tier; cold queries are rare |
| Index family | Pruned **global graph** + recomputation | Likely **disk/HDD-aware ANN** under compute pushdown |
| Overlap | Both care that ANN metadata dominates cost | Different fix: recompute vs. better cold I/O / partitioning |

**Paper-writing use:** Optional **Related Work / motivation footnote** on “storage cost of embeddings + graph metadata,” with clear disclaimer that LEANN optimizes **personal/edge RAG footprint**, not **cloud tail latency under disaggregation**.

---

## Open questions

- [ ] Read full PDF for exact latency breakdown (graph traverse vs. embed batch vs. rerank).
- [ ] How does recomputation interact with **filtered / multi-tenant** queries?
- [ ] Does `--no-recompute` mode recover a conventional HNSW baseline for apples-to-apples with Ember?

---

## Consensus

- Record as **interesting out-of-scope reference**; do not merge into partition/sharding survey as a core entry unless we add a “storage efficiency / recompute” sidebar later.
