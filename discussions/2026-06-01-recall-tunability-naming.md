# Session: Refining "Tunability" Dimension — Recall Tunability

- **Date**: 2026-06-01
- **Topic Tags**: `introduction`, `user-value-dimensions`, `tunability`, `recall`

## Background

In previous discussion (`2026-06-01-user-value-dims-refinement.md`), we simplified "Recall vs latency tunability" to "Tunability". However, the user expressed concern that this dimension is **interrelated with the performance dimensions** (low average latency, low tail latency), and the semantic boundary was not clear.

## The Core Issue

**Performance** and **Tunability** are interdependent:

- **Performance** answers: "At a given recall target, what is the latency?"
- **Tunability** answers: "Can users choose different recall targets?"

Without Tunability, Performance lacks context (which recall level?). Without Performance, Tunability lacks meaning (tradeoff between what and what?).

## User's Final Choice

**Dimension name**: **Recall tunability**

**Rationale**:
- Explicitly emphasizes that the tunable dimension is **recall**
- Distinguishes from other potential tunability axes (e.g., cost, throughput)
- Aligns with the user's intent: different workloads require different recall levels; some prioritize high performance at lower recall, others require high recall

## Revised Definitions

| Dimension | Description |
|-----------|-------------|
| **Low average latency** | At a given recall target, typical queries complete with low latency |
| **Low tail latency** | At a given recall target, p99 / p999 latency is bounded and predictable |
| **Recall tunability** | Users can explicitly select different recall targets to trade off accuracy for latency based on workload requirements |

**Key semantic clarification**:
- Performance dimensions describe latency **at a fixed recall target**
- Recall tunability describes the **ability to choose among multiple recall targets**

These are **logically orthogonal**:
- A system can have excellent Performance at 90% recall but poor Recall tunability (cannot achieve 80% or 95%)
- A system can have wide Recall tunability (70%–95%) but mediocre Performance at each level

## Cross-References

- `discussions/2026-06-01-user-value-dims-refinement.md` — Initial refinement of user value dimensions
- `summaries/introduction-draft.md` — Introduction section draft (to be updated with this decision)

## Action Items

- Update `summaries/introduction-draft.md` with "Recall tunability" dimension
- Update system comparison table accordingly (future discussion)
