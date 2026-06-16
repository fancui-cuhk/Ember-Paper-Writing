# 2026-06-16 — Paper PDF / Abstract Bibliography Lookup

## Topic

Batch collection of **working PDF URLs** and **official abstracts** (verbatim or close paraphrase) for partition / vector search papers.

## Key points

- Prefer PVLDB direct links, USENIX open access, NeurIPS proceedings, arXiv PDF, author/NSF PAR mirrors.
- ACM DL often returns 403 to scripts, but `dl.acm.org/doi/pdf/...` remains the preferred browser link.
- When official abstract could not be fetched: mark `NOT_FOUND` and attach a secondary paraphrase for reference.

## Venue corrections (relative to survey draft)

- **Distributed LSH (WebDB 2016):** No Haghani et al. WebDB 2016 entry found; canonical paper is **WebDB 2008 / EDBT 2009** — “Distributed similarity search in high dimensions using locality sensitive hashing”.
- **Multi-probe LSH PVLDB 2017:** User likely meant PVLDB **Vol. 10 No. 12 (2017)** retrospective “Intelligent Probing for Locality Sensitive Hashing: Multi-Probe LSH and Beyond” (original Multi-probe LSH at VLDB 2007).
- **SK-LSH PVLDB 2014:** PDF is `vol7/p745-liu.pdf` (not p521-huang).
- **OctopusANN:** PVLDB 2026 full title is *I/O Optimizations for Graph-Based Disk-Resident Approximate Nearest Neighbor Search: A Design Space Exploration*; OctopusANN is the combined system name in the paper.

## Open

- PASE / LindormVector ACM official abstracts need manual DOI page check.
- LEQAT used CoLab/Springer page abstract (Springer page 404; CoLab has full description).

## Relation

- **Input:** User paper list (partition survey coverage set)
- **Output:** JSON-like list from assistant reply in that session
