# 2026-06-16 — Read Amplification in Disk-Based Vector Search

- **Date:** 2026-06-16
- **Topic:** Read amplification survey for disk / disaggregated vector search
- **Output:** `summaries/read-amplification-disk-vector-search-survey.md`

## User request

Investigate **read amplification** in disk-based vector search: read a lot of data, use only part of it. User presumes in-RAM search does not suffer similarly (byte-addressable RAM).

## Consensus

1. **Definition:** RA = bytes (or pages) read ÷ useful bytes for the query; can stack at page, list, segment, and object layers.
2. **RAM:** Per-query I/O RA is minimal; problem shifts to **full-index memory footprint**. Cache-line waste is negligible vs 4 KB pages.
3. **Taxonomy (six sources):** (A) page granularity, (B) graph neighbor scatter, (C) IVF whole-list reads, (D) segment/shard/monolith, (E) full-precision rerank, (F) speculative I/O.
4. **Graph vs IVF:** Graph → many small dependent page reads; IVF → fewer large parallel list reads — different RA shapes.
5. **Cloud:** **Read amplification vs request amplification** trade-off (Milvus segments vs Turbopuffer per-cluster objects).
6. **Ember:** Segmentation reduces but does not eliminate RA; target storage units aligned to probed working set.

## Open

- Normalized RA benchmark across index families
- B+ANN / IISWC 2025 SSD characterization papers
- Segment-internal RA measurements for Milvus

## Follow-up (same day)

- **User request:** High-level catalog of **approaches to reduce read amplification** (not just per-paper entries).
- **Added:** Survey §4 — six families (hide, layout, algorithmic, systems, latency masking, bypass), eleven fix strategies with examples, §3 source mapping, Ember composite sketch. Renumbered former §4–§10 → §5–§11.

## Relation

- Builds on `summaries/disk-based-ann-survey.md`, `summaries/cloud-hosted-vs-cloud-native.md`, partition survey §4 colocation
