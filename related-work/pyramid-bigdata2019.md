# Pyramid — Distributed Similarity Search (Big Data 2019)

## Bibliography

| Field | Value |
|-------|--------|
| **Title** | Pyramid: A General Framework for Distributed Similarity Search on Large-scale Datasets |
| **Authors** | Shiyuan Deng, Xiao Yan, Kelvin Kai Wing Ng, Chenyu Jiang, James Cheng |
| **Venue** | IEEE Big Data 2019 / [arXiv:1906.10602](https://arxiv.org/pdf/1906.10602) |
| **Read date** | 2026-06-18 |

---

## Internal reading note — Ember relevance: **low**

**TL;DR:** Catalog entry for completeness. SPANN §4.3 cites Pyramid for “balanced partition + partial search,” but for **query-skew placement** the actionable line is **SPANN §4.3** (bin-packing on access history), not Pyramid.

**Why low relevance for Ember:**

1. **Default path** assumes uniform queries and equal-size graph partition weights — same “balance bytes, hope QPS follows” as many §4 papers.
2. Optional **sample-query top-k frequency** weights on meta-HNSW vertices are a weaker, earlier version of the SPANN/SABBSR idea — no production trace, no bin-packing over many micro-partitions.
3. **Runtime** skew handling is **Kafka queue rebalance + replication** (straggler / slow executor), not embedding-space **re-partition on query heat**.
4. **MIPS large-norm** hot-partition fix is a domain-specific assignment hack, not a general query-skew framework.

**Keep in survey** as historical cite chain (SPANN → Pyramid). **Do not** treat as a primary pole alongside SPANN / Harmony / SABBSR.

---

## What it does say (for the record)

- Algorithm 3: vertex weight = item count **or** top-k hit frequency on sample queries.
- Kafka: “automatically handle load balancing … for the message queues.”
- Replication of sub-HNSWs for stragglers.

See [`summaries/query-skew-spatial-partitioning-survey.md`](../summaries/query-skew-spatial-partitioning-survey.md) §10 for verbatim quotes.
