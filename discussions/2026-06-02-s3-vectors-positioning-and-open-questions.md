# Session: Feedback on S3 Vectors Positioning and Open Questions

- **Date**: 2026-06-02
- **Topic Tags**: `s3-vectors`, `oss-vectors`, `positioning`, `cold-start-query`, `open-questions`

## Context

This session records the author's detailed feedback on the agent's previous review of the Introduction (Sections 1–5). The author provided clarifications on two key points and requested that several open questions be added to the draft.

## Key Clarifications from Author

### 1. Positioning Relative to S3 Vectors and OSS Vectors

**Author's Core Argument**:

The author clarified that Ember's approach is fundamentally different from S3 Vectors / OSS Vectors:

- **Ember's approach**: The author intends to **replace or augment the storage layer itself** by building a new distributed storage layer that natively supports vector search. This layer is built on top of cheap media (HDD or low-cost cloud disks) and is optimized specifically for vector search workloads. Its primary purpose is to efficiently handle **cold start queries**. The key selling point is an **architectural idea**: "a storage layer with built-in vector search capability for cold data." Under this architecture, **any index** can be used and optimized. The index format is intended to be open and pluggable.

- **S3 Vectors / OSS Vectors' approach**: These systems add vector search capability **inside the object storage service itself**, primarily targeting performance-insensitive, cost-sensitive batch/analytical workloads. Their indexes are **proprietary and closed**.

**Why Ember does not directly compete with them**:

1. **Technical incompatibility**: Because S3 Vectors and OSS Vectors use proprietary, closed index formats, it is impossible to use them as the underlying storage layer for a disaggregated architecture like Ember's. Their indexes cannot be extracted or composed with an external compute layer.

2. **Different target workloads**: S3 Vectors / OSS Vectors are designed for workloads that can tolerate higher latency. Ember targets interactive/RAG workloads that require predictable tail latency.

3. **Architectural philosophy**: Ember sells a **general architectural pattern** (cheap storage layer with vector search for cold data), not a specific closed system. If major object storage providers in the future support open index formats and optimize them, that would actually be consistent with Ember's philosophy — users would not need to change their storage layer.

**Implication for the paper**:

The author has not yet decided whether to address S3 Vectors / OSS Vectors in the Introduction or defer the discussion to Related Work. The decision will depend on how clearly the Solution section differentiates Ember from these systems.

### 2. Cold Start Query Terminology

The author confirmed that the distinction made in the draft is correct:

- In this paper, "cold start query" refers to the situation where **vector index data** is not ready in compute memory or SSD cache.
- This is different from the classic serverless cold start problem, which focuses on **compute resource provisioning**.

The author agreed that the existing Note can be slightly expanded for clarity.

## Open Questions to Add

The following open questions should be added to the Introduction draft:

1. **S3 Vectors / OSS Vectors positioning**: Should the paper explicitly differentiate Ember from S3 Vectors and OSS Vectors in the Introduction, or should this discussion be deferred to Related Work? The decision depends on whether the Solution section naturally leads readers to compare Ember with these systems.

2. **Low tail latency target (500ms)**: The current draft states that tail latency should be "under 500ms." This number currently lacks a direct citation. One possible justification is that many LLM application clients set timeout thresholds around this range; any cold start latency exceeding client timeout would require special handling on the client side. A reliable source or more rigorous justification is needed.

3. **Cold start query latency numbers (~10s)**: The draft currently uses approximate numbers such as "~10s" for cloud-native cold start latency. These numbers are placeholders. Future versions should either (a) replace them with experimental data, or (b) clearly specify the experimental setting (dataset size, nprobe, storage media, etc.) when citing external sources.

4. **Recall tunability removal**: The author has decided to remove the "Recall tunability" dimension from the user value table. The rationale involves S3 Vectors / OSS Vectors positioning (detailed above). This decision and its justification should be revisited once the Solution section is written.

## Relationship to Previous Records

- Extends: `discussions/2026-06-01-user-value-dims-refinement.md` (removal of Recall tunability)
- Extends: `discussions/2026-06-01-merge-confirmation.md` (classification decisions)
- Related: `related-work/lindorm-vector-sigmod2026.md` (different storage assumption)

## Action Items

- Add the four open questions listed above to the Introduction draft.
- Revisit S3 Vectors / OSS Vectors positioning after the Solution section is drafted.
- Find or justify the 500ms tail latency target.
- Replace placeholder latency numbers with experimental results or properly cited sources.