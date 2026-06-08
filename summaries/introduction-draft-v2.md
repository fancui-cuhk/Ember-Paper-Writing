## Ember: Low Tail Latency Cloud-Native Vector Search

- In the LLM era, cloud vector search systems have become a foundational infrastructure, and face new challenges.
- First-generation cloud vector search systems are **cloud-hosted** -- traditional on-prem systems hosted on cloud VMs.
    - They are designed for pre-LLM workloads and are becoming incompetent in the LLM era.
    - Cost-wise, they are prohibitive, especially under billion-scale workloads.
    - Scalability is poor, due to the use of the monolithic architecture.
- Second-generation systems -- **cloud-native vector search systems**, are designed from the ground up for cloud infrastructure and LLM applications.
    - SOTA systems are Milvus 2.x, Pinecone, Turbopuffer.
    - They are customized for LLM applications: billion-scale, heterogeneous workloads such as RAG and corporate knowledge base.
    - **Auto-scaling compute (w/ SSD cache) + disaggregated object storage (e.g., S3).**
    - This architecture leads to many advantages:
        - Good performance at significantly lower cost.
            - These systems cache vector index in compute RAM and SSD to support ultra-fast queries.
            - Compute is auto-scaling, during workload trough, compute resources can scale down, or scale to zero (serverless).
            - Data is primarily stored on cheap object storage.
              - S3 is 5X cheaper than EBS SSD, 200X cheaper than EC2 RAM.
        - Elastic scaling.
        - High availability.



- **But, cloud-native systems suffer from one fatal flaw: high tail latency.**

| SOTA Systems | Tail latency (sec) FAKE! |
| ------------ | ------------------------ |
| Pinecone     | 10                       |
| Milvus       | 10                       |
| Turbopuffer  | 5                        |



- **Cache miss is the culprit behind the high tail latency problem!**
  - In cloud-native systems, query performance hinges on **whether the required index is cached.**
  - Under **cache miss**, a query must first load the index (might be GBs) from S3.
    - This results in significantly higher query latency.
  - Cloud-native systems term queries with cache miss as **cold start queries.**
- **Existing systems choose to live with cache miss.**
  - They dynamically adjust cache size to make sure cold start queries only account for a minority, so average query performance is guaranteed.



- We present Ember, a new-generation cloud-native vector search system with **low average latency, low cost, high scalability, and also, low tail latency.**
  - We stick with the disaggregated architecture, continue to enjoy its intrinsic good properties: low average latency, low cost, and high scalability.
  - But, we meticulously handle cache miss to **alleviate** the high tail latency problem.
    - *We do not target to completely **solve** the problem and make queries w/ cache miss run as fast as other queries w/ cached index.*



- To this end, Ember employs **cold start compute pushdown**.

  - For a cold start query (i.e., query with cache miss), we push the query execution down to disaggregated storage layer's compute.
  - Consequently, network-based index loading is removed from the critical path of cold start queries.
  - Tail latency can be greatly alleviated.

  - **Compute pushdown is a well-studied technique and has been deployed in various disaggregated storage systems, but we are the first to deploy it in the vector search context.**



- Under the cold start compute pushdown setting, the high tail latency problem gets boiled down to **disk-based vector search in the storage layer.**
- Disk-based vector search is yet another well-studied problem, with SOTA algorithms being DiskANN and SPANN.
- But, our system faces new requirements due to the low cost target:
  - Storage node must have limited compute resources.
  - Storage node must use cheap storage media (e.g., HDD).
    - *EBS SSD is ~14X more expensive than EC2 instance HDD.*

- Unfortunately, SOTA disk-based vector search solutions are designed for powerful compute servers with high-end NVMe SSDs. These solutions break under the "weak CPU + HDD" case.
  - DiskANN is a single-node graph-based solution, its fast graph traversal is built upon SSD's low read latency (~100 us). On SSD, 100 hops (sequential SSD reads) only take ~10ms. This breaks on HDD:
    - One HDD read takes ~10 ms (100X slower than SSD), same 100 hops take ~1s.
    - Distance computation would also be significantly slower due to limited compute resources on one storage node.
  - SPANN is a single-node IVF-based solution, its fast inverted list loading is built upon SSD's high read bandwidth (~7 GB/s). On SSD, loading inverted lists with total size 100 MB only takes ~15 ms. This also breaks on HDD:
    - HDD banwidth is ~200 MB/s (35X lower than SSD), same 100 MB loading takes ~0.5s.
    - Distance computation would also be significantly slower due to limited compute resources on one storage node.



- In response to above problems, we design **EmberANN**, a distributed HDD-based vector search system running in Ember's disaggregated storage layer.

  - **EmberANN is customized for the "weak CPU + HDD" setting. It meticulously parallelizes index loading and query execution to multiple storage nodes, thus utilizing the aggregate compute power and disk bandwidth.**

  - EmberANN runs on a distributed shared-nothing storage layer.

    - There are a set of storage nodes, each attached to a set of HDDs.

  - EmberANN employs an IVF-like index and a scatter-gather query execution framework.

    - An IVF index is divided into fixed-size blocks (e.g., 10 MB) and distributed (with replication) to a set of HDDs.

    - Query execution proceeds as follows:

      - A query coordinator first loads cluster centroids and decides what clusters to probe.

      - Then it divides the cluster probing task onto multiple storage nodes.
        - *If we assign node N to probe cluster x, we must ensure that x resides on at least one HDD attached to node N.*
      - Each storage node individually loads assigned clusters and performs distance calculations.
        - Multi-HDD parallel loading -- high aggregate load bandwidth.

      - Results of all sub-queries are gathered and re-ranked on a query coordinator.

    - *Efficient graph traversal relies on good random read performance, which is not viable for HDDs. Hence, we do not use graph-based solutions.*

  - Under this setting, load balancing among storage nodes is crucial for good performance.

    - EmberANN features **query-aware index block placement and probe request routing**, to (1) adjust index block placement on HDDs, and (2) route cluster probing requests to less-busy storage nodes at runtime.























### Open Questions

- DistributeANN is a distributed disk-based ANN [https://arxiv.org/pdf/2509.06046]. And, other distributed disk-based indexes?

### Deleted

- S3 Vectors problems:
  - S3 Vectors exposes only vector-level Put/Get APIs; it provides no interface for storing or quering a pre-built vector index.
  - S3 Vectors supports only proprietary index, so cold start queries served through it would return inconsistent results with other queries served through the system's own index.
  - S3 Vectors suffers from a low-recall problem; rumors say that it uses aggressgive quantization internally. /cite{https://zilliz.com/blog/will-amazon-s3-vectors-kill-vector-databases-or-save-them}
  - S3 Vectors does not support hybrid search.

- **Cold start queries should be minority in cloud-native systems, given that EmberANN only executes cold start queries, limited compute resources on storage node is not a big problem.**
  - If cold start queries account for majority, we should consider increasing the SSD cache size in the compute layer.
