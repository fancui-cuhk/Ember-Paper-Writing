- no need to mention LLM in the intoduction.

- cache index **portion**, not cache index.

- we do not **alleviate** the cache miss problem, this means reducing the number of cache miss.

- do not use "must use cheap storage". this is not a requirement, this is what we want.

- the last two paragraphs are bad.
    - change from SSD to HDD:
        - do not use diskann and spann upfront, they are just examples.
        - what happened after this change? the whole landscape changed. then, use diskann and spann for examples.
        - big picture first, say what changed, then use examples.

    - no need to focus on IVF vs. HNSW, focus on SSD vs. HDD.

    - do experiments fast! otherwise we cannot finalize our tone in the introduction. use figures to compare and contrast.

    - "distributed" suddenly shows up. there is a gap. we do not motivate it beforehand.
        - we were compare and contrasting SSD vs. HDD.
        - are we moving from single-node to distributed because of:
            - index cannot fit in single-node?
            - or this is a requirement of running fast queries on HDDs?



this is a new flow:

- challenges of disk-based vector search under compute pushdown:
    - insufficient compute.
    - we want to use cheap HDD.
- old SOTA? diskann and spann
    - after changing from SSD to HDD, xxx has changed!
    - so old SOTA not good enough!
- so we have to design something new in this "weak CPU + HDD in storage layer" setting.
- good sides: abundant storage nodes in the storage layer! (look in a high level of the storage layer)
    - aggregate compute, aggregate disk bandwidth.
- in this storage layer environment, what are the core research questions:
    - traditional problem, but new fixes.
    - load balancing?
    - index partitioning?
    - replication?
- at last, briefly talk about Ember, very brief.

不是讲做了什么，是里面有什么research question，我们怎么解决的？
不要只讲how，讲why，how。

Fri: Intro + detailed experiment plan ready.
