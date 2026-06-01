# Session: Refining User Value Dimensions for Introduction

- **Date**: 2026-06-01
- **Topic Tags**: `introduction`, `user-value-dimensions`, `motivation`, `paper-structure`

## Context

Following the LindormVector paper reading and summary update, we return to the Introduction section to refine the **User Value Dimensions** table that forms the foundation of the motivation narrative.

## User's Proposed Changes

### 1. Cost Dimensions: Merge

| Original | New |
|----------|-----|
| Low usage cost | **Low TCO** (Total Cost of Ownership) |
| Low operational cost | (merged into Low TCO) |

**Rationale**: Both are aspects of cost; users care about the bottom line, not the distinction between infrastructure and operational costs.

### 2. Performance Dimensions: Clarify

| Original | Options Discussed |
|----------|-------------------|
| High query performance | **Low average latency** |
| Stable query performance / Low tail latency | **Low tail latency** |

**User's suggestion**: Be explicit — "low average latency" and "low tail latency" are clearer than vague "high performance" and "stable performance."

**Agent's note**: "Low average latency" and "low tail latency" are more precise. Alternative could be "predictable performance" but that is vague. Explicit latency metrics are better for technical paper.

### 3. Tunability: Simplify

| Original | New |
|----------|-----|
| Recall vs latency tunability | **Tunability** |

**Rationale**: Shorter, still clear in context.

### 4. Multi-tenancy Isolation: Rethink

**User's concern**: "Multi-tenancy isolation" exposes implementation details to users. Cloud's value is that users don't need to know there are other tenants.

**Reframe the question**: What user-visible guarantees does multi-tenancy isolation provide?

| Aspect | User-Facing Guarantee |
|--------|----------------------|
| Security / Privacy | **Data isolation** — my data is not visible to others |
| Performance | **Performance stability** — other users' load doesn't affect my latency |

**Options**:
1. Keep "Multi-tenancy isolation" but frame as **performance stability under shared infrastructure**
2. Rename to **Performance isolation** — emphasizes the guarantee, not the mechanism
3. Merge into **Low tail latency** if the primary concern is tail spikes from noisy neighbors

**Open**: User to decide final framing.

## Revised User Value Dimensions (Draft)

| Dimension | Description | Priority |
|-----------|-------------|----------|
| **Low TCO** | Economical at billion-scale; pay for actual usage | Must-have |
| **Low average latency** | Typical query response in milliseconds | Must-have |
| **Low tail latency** | p99 / p999 latency bounded and predictable | Must-have |
| **High scalability** | Handle load fluctuations (scale up/down) | Must-have |
| **High availability** | Data durability, fault tolerance | Must-have |
| **Tunability** | Trade off accuracy for speed per query | Should-have |
| **[TBD: Performance isolation]** | Stable performance despite shared infrastructure | Should-have / Nice-to-have |

## System Comparison Table: Implications

The motivation table (existing systems vs. dimensions) needs update:

| System | Low TCO | Low Avg Latency | Low Tail Latency | High Scalability | High Availability | Tunability | [Perf Isolation] |
|--------|---------|-----------------|------------------|------------------|-------------------|------------|------------------|
| **Cloud-Hosted** (EC2/RDS) | ❌ High VM cost | ✅ Low (warm) | ✅ Predictable | ❌ Manual | ⚠️ Self-managed | ✅ Full control | ✅ (single tenant) |
| **Pinecone Serverless** | ✅ Low | ✅ Low (warm) | ❌ Up to 20s cold | ✅ Auto | ✅ 99.99% SLA | ⚠️ Limited | ✅ Namespace |
| **Turbopuffer** | ✅ Very low | ✅ Very low (warm) | ⚠️ 300ms-4s cold | ✅ Auto | ✅ Managed | ⚠️ SPFresh only | ⚠️ Basic |
| **Zilliz/Milvus** | ⚠️ Medium | ✅ Medium | ❌ Cold segment load | ✅ Elastic | ✅ Managed | ✅ Multiple | ⚠️ Collection |
| **[Ember target]** | ✅ | ✅ | ✅ (goal) | ✅ | ✅ | ✅ | ✅ |

## Next Steps

1. Confirm final name for "performance isolation" dimension (or drop)
2. Draft Introduction.md focusing on:
   - Problem statement (cloud-native direction is right, but cold query blocks adoption)
   - User value dimensions table
   - System comparison showing gap
   - Architecture explanation (disaggregation)
   - Cold query problem magnitude
   - Root causes (from `summaries/cloud-hosted-vs-cloud-native.md`)

**Explicitly out of scope for this discussion**: Solution / Design / Ember's approach (to be added later).

## Date Note

This discussion uses the correct date **2026-06-01**. Previous discussions were incorrectly dated 2026-05-28 — will maintain as-is for historical accuracy but new discussions use correct dates.
