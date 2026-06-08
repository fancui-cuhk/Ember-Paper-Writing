## Meeting notes with Eric

#### General comments

- 主题是什么？东西太多了，没有环绕主题来写。
    - 有两个主题：一个是表格，一个是后面的tail latency vs cost的图，到底是想要突出哪一个？围绕哪个来写的？
- 现有的很多东西，比如architecture diagram，其实不需要放到introduction里面。
    - 铺垫太多了，细节太多了。
- 花了太多精神比较cloud-hosted和cloud-native，特别是cloud-hosted，没必要讲那么多。讲故事来带过就行了。
- 快快进入重点：tail latency问题，分析，前人做了什么，哪里没做好？他们怎么做的？有没有什么漏掉的地方？
- 我们说前人不好只是简单说它们tail latency不好，简单的分析：cache miss就是慢！不要深入分析，把深入分析放到后面！
    - 就坚持这个方向！cache miss是最关键的，其他的放旁边。
- 大概这样的思路：pushdown概念上可以变快，但是！大家不要以为这个这么简单，还是有很多challenges。

#### Detailed comments

- whole index loading is NOT a good naming!
- 可能不需要最开始的table？
- 简单化architecture diagram: current diagram tells one thing -- disaggregation, and that is well-known already.
- be clear: our architecture is based on hard disk drive (HDD).
- 目前的想法：保留系统的布局。走到核心的道理之后：开宗明义要讲diskann。
    - diskann designed for ssd. bad on hdd. so we design sth new.
    - we get information from the upper compute layer, how to use this information?
        - runtime下层看着上层调节？额外的信息，偷看
        - runtime上层通知下层调节？额外的信息，主动给
        - 不需要runtime偷看或主动给，我们直接设计好了，二者可以高效协作。
    - 这里的重点是：不要让别人攻击为什么不卖成单独的hddann？

#### Sell a system or an ann algorithm?
- since our main contribution is hddann, we can name our paper as: "hddann" e.g., "emberann".
- if we stick with the system approach, potential attacks:
    - system as a whole i like it, but nothing special, only contribution is hddann.
- things we need to consider:
    - do we have other optimizations apart from hddann? on system level, beyond hddann?
    - 不要挂羊头卖狗肉！
    - 这个hddann有没有利用整个系统的信息？即，系统给我们送下来的workload有没有给我们的hddann一些信息？
- 如果我们卖系统，big picture上和system斗，ablation和diskann斗。
- or, on a FAST angle: vector search performance on HDD vs. SSD.

#### New storyline

- vector database上云，有两种：cloud-hosted和cloud-native，推崇后者。
- 什么是cloud-native？
- cloud-native SOTA是什么？有什么问题？
- 问题就是high tail latency!
    - 没有必要quote pinecone和milvus的结果，那是他们实验的结果，放在一起不好比较。
- So, why high tail latency?
    - **cold start query == cache miss**!
    - cache miss is definitely slow, everyone knows about that. so RC1 is actually cache miss.
    - cache miss是核心的problem -- root cause.
- But, when solving the cache miss problem, what are the challenges?
    - 现有系统在解这个问题的时候，有哪些approach？各个approach有什么问题？他们有没有漏哪些opportunities？
    - 总之，要回答的问题是：前人做了什么？为啥没做好？
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

- 我想的太多，不要考虑那么多，就顺着一个思路写，稳住思路，不要被问题distract，写完再说。
- make up experimental figures! don't be blocked on anything.
