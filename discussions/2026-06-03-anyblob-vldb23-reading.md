# Session: AnyBlob / S3 OLAP Paper Reading Note

- **Date:** 2026-06-03
- **Topic Tags:** `related-work`, `S3`, `OLAP`, `AnyBlob`, `VLDB-2023`

## User request

Summarize the newly added S3 OLAP paper in `related-work/` (`anyblob.pdf`) and store the summary under `related-work/`.

## Actions

- Identified PDF as **Durner, Leis, Neumann — Exploiting Cloud Object Storage for High-Performance Analytics**, PVLDB 2023.
- Created `related-work/anyblob-vldb2023.md` (bibliography, S3 measurements, four findings, AnyBlob design, Umbra integration, Ember positioning).
- Updated `related-work/README.md` index and linked from `summaries/cold-hot-query-paths.md` Appendix C.

## Consensus

- This is the **primary rigorous reference** in-repo for S3 **OLAP** latency/throughput (not vector-specific).
- Supports Ember discussion on **parallel GETs**, **per-request latency**, and **8–16 MiB** read granularity; does **not** imply vector cold query is bandwidth-saturated.

## Relation

- Extends `discussions/2026-06-03-milvus-terminology-s3-fundamentals-minio.md` with citable numbers and findings.
