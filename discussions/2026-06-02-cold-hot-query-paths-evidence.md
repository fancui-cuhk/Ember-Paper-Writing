# Session: Evidence-Backed Cold/Hot Query Paths

- **Date:** 2026-06-02
- **Topic Tags:** `cold-query`, `hot-query`, `S3-round-trip`, `pinecone`, `milvus`, `turbopuffer`, `evidence`

## User request

Re-verify all claims with sources; produce a single `summaries/` markdown covering **full** cold and hot query execution paths for Pinecone Serverless, Milvus 2.x, and Turbopuffer, with per-step latency only when published (otherwise mark not published). Then commit and push.

## Actions taken

1. Re-fetched Pinecone architecture docs, How Pinecone Works, serverless architecture blog (cold-start up to ~20s at billion scale).
2. Re-fetched Milvus tiered storage docs (2.6.4+) and tiered storage benchmark blog (hot/cold P99, load times, 100M-vector setup).
3. Re-fetched Turbopuffer launch blog (≤3 S3 round-trips, 444ms cold / 10ms warm at 1M×768d), ANN v3 and native-filtering blogs for RTT bounds.
4. Created `summaries/cold-hot-query-paths.md` with step tables, S3 I/O accounting (symbolic where exact counts unpublished), explicit evidence gaps, and reference list.

## Consensus

- **Do not invent** per-step latency; use vendor-published end-to-end numbers only, with benchmark context.
- **Pinecone cold S3 cost** scales with **K** cache-miss slabs; exact GETs per slab **not published**.
- **Milvus** must distinguish **full-load** (cold = load/recovery, 0 S3 per hot query) vs **tiered** (cold = on-demand chunk/index GETs).
- **Turbopuffer** uniquely publishes a **hard cap of 3 S3 round-trips** for cold vector search (design target); filtering path ≤2 RTTs.

## Open questions

- Pinecone: subset slab routing vs full namespace fan-out — official docs ambiguous.
- Milvus: exact chunk GET count per cold query on Storage v2 — needs code-level or experimental measurement.
- Billion-scale cold query numbers for Milvus and Turbopuffer — not in cited primary sources.

## Relation to prior records

- Extends `discussions/2026-05-30-root-cause-analysis.md` with system-specific paths.
- Supports `summaries/introduction-draft.md` §5 cold-start table with cited conditions.
