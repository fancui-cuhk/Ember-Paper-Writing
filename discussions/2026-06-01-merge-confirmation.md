# Session: Post-Merge Confirmations and Next Steps

- **Date**: 2026-06-01
- **Topic Tags**: `meta`, `classification`, `root-causes`, `lindorm`

## Confirmations from User

### 1. System Classification Terminology

**Decision**: Retain **Cloud-hosted vs Cloud-native** classification.

- Do NOT change to "Specialized vs General-purpose" (per arXiv 2310.14021)
- Rationale: User's original terminology better captures the architectural distinction relevant to the cold query/tail latency problem
- Cloud-hosted: Provisioned VMs with local storage
- Cloud-native: Compute-storage disaggregation with object storage (S3)

### 2. Root Causes HNSW-Centric Wording

**Status**: Deferred for future discussion.

- Current Introduction Draft uses HNSW-specific language ("graph traversal")
- User acknowledges need to generalize to cover both HNSW and IVF indexes
- User will revisit this later
- Previous analysis (discussions/2026-05-30-root-cause-analysis.md) identified that IVF faces similar challenges via "random access pattern" and "request amplification"

### 3. LindormVector Related Work

**Status**: User has manually updated.

- LindormVector uses ESSD (not S3) as storage layer
- Different architectural assumption from Ember's cloud-native/S3-based design
- User has already incorporated appropriate comparison in related-work/

## Current State of Repository

### Synchronized Content
- `summaries/introduction-draft.md`: Complete point-form draft with 6 dimensions, comparison table, 4 root causes, architecture description
- `discussions/`: All session records from May 28, May 30, June 1
- `related-work/lindorm-vector-sigmod2026.md`: Analysis of LindormVector (ESSD-based approach)

### Deferred Items (for future sessions)
1. Generalize root cause wording beyond HNSW
2. Finalize architecture diagram style
3. Solution section (Ember design) — intentionally omitted from Introduction draft

## Next Steps

Awaiting user direction for next discussion topic. Options:
- Design section: Ember's approach to mitigating the 4 root causes
- Related work expansion: Comparison with LindormVector, other competitors
- Root cause refinement: Generalizing index-traversal formulation
- Experimental design: Evaluation methodology for tail latency claims
