# Session: Introduction Structure Feedback and Revisions

- **Date**: 2026-06-01
- **Topic Tags**: `introduction`, `structure`, `classification`, `root-causes`, `core-dimensions`

## User's Feedback Summary

### 1. Background Section
- Current version "does spoilers" — too much forward-looking content
- Should be simple: introduce vector search + LLM-era challenges
- Must include: billion-scale workloads, vector archive (infrequently accessed data)
- Use widely accepted facts only; no fabricated claims
- End with: "this poses new requirements and challenges for cloud vector search" → naturally leads to next section

### 2. TCO Dimension
- "TCO grows sub-linearly with data volume and query load" — is sub-linear scaling realistic?
- Suggestion: "TCO is low, enabling billion-scale workloads"

### 3. Latency Dimensions
- "At a given recall target" makes latency look conditional
- Need cleaner formulation

### 4. Core Dimensions Overall
- Need to find balance between "general" and "clear"
- Should reference seminal research papers and industry blogs
- Not unique to Ember — other systems share similar goals

### 5. Classification: Cloud-Hosted vs Cloud-Native
- Is this widely accepted?
- Mainstream classification is **specialized** vs **general-purpose** (or purpose-built vs extended/integrated)
- Evidence: SingleStore blog, arXiv Survey of Vector Database Management Systems (2310.14021)

### 6. Recall Tunability
- Both cloud-hosted and cloud-native have systems that don't expose parameters
- Should be unified in the table (not two separate columns)

### 7. Key Insight Section
- After the comparison table, should explicitly state: "we focus on cloud-native architecture below"

### 8. Root Causes
- User questions whether the four root causes are comprehensive

### 9. Perspective Framing
- User is a PhD student writing a research paper
- Should say "standing from the perspective of cloud vector database design" — not "we are a cloud vendor"

## Research Findings (Verification)

### Classification Terminology
Confirmed via web search:
- **Specialized / Purpose-built / Native**: Milvus, Pinecone, Weaviate, Qdrant
- **Extended / General-purpose / Integrated**: pgvector (PostgreSQL), MongoDB Atlas, SingleStore-V
- **Vector Libraries**: FAISS, ScaNN (not full databases)

Sources: SingleStore "Ultimate Guide to Vector Database Landscape 2024", arXiv 2310.14021 "Survey of Vector Database Management Systems"

### Vector Archive / Cold Data Workload
Confirmed:
- Tiered storage (hot/warm/cold) is industry practice for vector data
- Cold tier: object storage (S3) for archived documents with negligible query frequency
- Sources: "Beyond Similarity Search: A Unified Data Layer for Production RAG Systems" (arXiv 2605.03275), Mixpeek "Vector Storage Tiering" guide

### Root Causes of Tail Latency
Widely accepted factors:
- Cold start penalty in serverless/disaggregated systems (800ms–3s+)
- Graph traversal (HNSW) causes unbounded tail latency on cold data (sequential I/O)
- Hierarchical structures (SPFresh) bound round-trips to tree height
- Disaggregation exacerbates tail latency when compute must fetch from storage on demand

Sources: Turbopuffer ANN v3 analysis, Pinecone architecture reviews, Actian "Vector Database Benchmarks are Misleading"

## Action Items

1. Rewrite Background section (simple intro + LLM challenges + billion-scale + vector archive + widely accepted facts)
2. Revise Core Dimensions:
   - TCO: "Enables billion-scale workloads economically"
   - Latency: Remove "at a given recall target" or rephrase
   - Reference seminal papers for balance of generality and clarity
3. Change classification: Specialized vs General-purpose
4. Unify Recall Tunability column (both categories have systems without parameter exposure)
5. Add explicit statement after comparison table: focus on cloud-native architecture
6. Re-evaluate root causes for comprehensiveness
7. Change perspective framing: "standing from the perspective of cloud vector database design"

## Next Steps

User to confirm revised introduction-draft.md before further commit.