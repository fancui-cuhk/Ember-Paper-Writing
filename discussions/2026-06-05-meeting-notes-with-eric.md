## Meeting notes with Eric

#### General comments

- What is the **central theme**? Too much material; the draft does not orbit one theme.
    - Two competing focal points: the opening table vs. the tail latency vs. cost figure—which one is primary?
- Many elements (e.g., architecture diagram) do **not** belong in the introduction.
    - Too much setup; too much detail upfront.
- Spent too much effort comparing cloud-hosted vs. cloud-native—especially cloud-hosted. **Brief narrative pass** is enough.
- Get to the point quickly: **tail latency**—analysis, what prior work did, what they missed, how they did it, what opportunities remain.
- When criticizing prior work, keep it simple: **tail latency is bad because cache miss is slow**. Save deep analysis for later sections.
    - Stay on this line: **cache miss is the root cause**; everything else is secondary.
- Overall arc: pushdown can help in principle, but it is **not trivial**—there are real challenges.

#### Detailed comments

- whole index loading is NOT a good naming!
- Maybe drop the opening table?
- Simplify architecture diagram: current version only says disaggregation, which is already well known.
- Be clear: our architecture targets **hard disk drive (HDD)**.
- Current plan: keep the system layout. Once the core argument is set, **open with DiskANN**:
    - DiskANN was designed for SSD; it is bad on HDD → we need something new.
    - We receive information from the upper compute layer—how to use it?
        - Runtime: lower layer observes upper layer and adapts? (extra info, “peeking”)
        - Runtime: upper layer notifies lower layer? (extra info, explicit signal)
        - **Preferred:** no peeking or explicit runtime signaling—co-design so layers collaborate efficiently by construction.
    - Key defense: avoid the attack “why not publish standalone HDD-ANN?”

#### Sell a system or an ann algorithm?
- since our main contribution is hddann, we can name our paper as: "hddann" e.g., "emberann".
- if we stick with the system approach, potential attacks:
    - system as a whole i like it, but nothing special, only contribution is hddann.
- things we need to consider:
    - do we have other optimizations apart from hddann? on system level, beyond hddann?
    - Do not bait-and-switch (system paper that is really only an ANN algorithm, or the reverse).
    - Does HDD-ANN use **whole-system information**—i.e., does workload context from the system inform the index?
- If we sell a **system**, compete at the system level; use ablations to compete against DiskANN.
- or, on a FAST angle: vector search performance on HDD vs. SSD.

#### New storyline

- Vector DB on cloud: two models—**cloud-hosted** vs. **cloud-native**; we advocate the latter.
- What is cloud-native?
- What is cloud-native SOTA? What is wrong with it?
- The problem is **high tail latency**.
    - No need to quote Pinecone/Milvus numbers side by side—they are from different experimental setups.
- So, why high tail latency?
    - **cold start query == cache miss**!
    - cache miss is definitely slow, everyone knows about that. so RC1 is actually cache miss.
    - **Cache miss is the root cause.**
- But, when solving cache miss, what are the challenges?
    - What approaches do existing systems take? What is wrong with each? What opportunities did they miss?
    - Core question: **what did prior work do, and why was it not enough?**
    - **actually, previous systems live with cache miss.**
- our position: in this good cloud-native architecture, we solve the tail latency problem!
    - but, we better inform readers: we are not magicians, we cannot achieve better tail latency than cloud-hosted ones.
- go to our core solution: **cold start pushdown**.
    - pushdown eliminates the network cost between compute and storage.
    - we only pushdown cold start queries (the cache missed case), hot queries still enjoy disaggregation.
    - this is nothing smart, we do not pretend to be smart here.
    - we ack: pushdown has been employed in many different settings, such as OLAP.
    - **but none has applied this in the vector search context yet.**
- BUT, under this pushdown setting, there are challenges, missing opportunities (specifically under vector search)!
    - a natural question is: under this local disk setting, why not *diskann or spann*? they are specifically designed for local disks.
        - they are designed for ssd.
        - but we use hdd due to strict cost restriction.
        - diskann/spann on hdd? so slow! due to *xxx reasons*.
- so what do we do?
    - we design something new -- emberann (our contribution).
    - emberann features *xxx techniques*, solves *yyy challenges*.
    - emberann optimizes for the cold start workload in *zzz approaches*.

#### Problems

- why pushdown is an obvious solution to the cache miss problem?
    - Eric said so, but why? we simply state this?
- **after we give the pushdown setting, we talk about challenges first or we directly talk about diskann/spann?**
- after we give the pushdown setting, what are the challenges? and missing opportunities?
- diskann and spann sucks for what reasons?
- and, what does emberann do?
- **how does the cold start workload affect our system design?**
    - this is crucial, it means that we are building emberann as a component in the system, not a standalone ann algorithm.
- disaggregation implies insufficient compute on the storage layer.
    - but now we are pushing query execution down to the storage layer.
    - we need to defense against this!
    - make people understand this setting, why compute is not a big requirement here in the storage layer.
        - put this down somewhere: cold start queries are rare, so no need to have much compute in the storage layer.

#### Personal advice

- Overthinking is the enemy: pick one storyline, stay on it, do not get distracted by side questions—finish the draft first.
- make up experimental figures! don't be blocked on anything.
