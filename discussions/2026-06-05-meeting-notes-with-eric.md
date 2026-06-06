## New Introduction Storyline

- Firstly, 




- what is the main topic? 没有环绕主题来写。
    - 有两个主题：一个是表格，一个是后面的tail latency vs cost的图，到底是想要突出哪一个？围绕哪个来写的？
- Sec 1: 主题；Sec 2：给background。architecture diagram其实不需要放到introduction里面

Too many gibberishes!铺垫太多了，细节太多了。
vector database上云，有两种：cloud-hosted和cloud-native，推崇后者，motivate后者。
什么是cloud-native？
cloud-native SOTA是什么？有什么问题？
花了太多精神比较cloud-hosted和cloud-native，特别是cloud-hosted，没必要讲那么多。讲故事来带过就行了。
融合Sec 2和Sec 3

BUT BUT BUT, tail latency!

没有必要quote pinecone和milvus的result，我们做实验验证就行了！

没有必要讲太多的detail，主要是story flow。
cold start query == cache miss!!! the feeling of cache miss. cache miss is definitely slow, so RC1 is kind of cache miss.

whole index loading is NOT a good naming!

cache miss是核心核心核心的problem(RC1)
但是，解这个problem的时候，有什么challenge。

比如，现有系统，在解这个问题的时候，有哪些approach？各个approach有什么问题？他们有没有漏哪些opportunities？

总之，要回答的问题是：前人做了什么？为啥没做好？

可能不需要最开始的table？
简单版architecture diagram

快快进入重点：tail latency问题，分析，前人做了什么，哪里没做好？他们怎么做的？有没有什么漏掉的地方？

our position: in this good cloud-native architecture, we solve the tail latency problem!

but, we better inform them -- not to think too good, we are not magicians, we cannot achieve better tail latency than cloud-hosted ones.

be clear: architecture based on hard disk drive (HDD).

then go to our core solution: pushdown, but nothing smart, do not pretend to be smart!

BUT, under this pushdown setting, there are challenges, missing opportunities (specifically under vector search)!

我们说前人不好只是简单说它们tail latency不好，简单的分析：cache miss就是慢！不要深入分析，把深入分析放到后面！

大概这样的思路：pushdown概念上可以变快，但是！大家不要以为这个这么简单，还是有很多challenges。

就坚持这个方向！RC1是最关键的，其他的放旁边。

在这个setting之下，diskann怎么不行？
diskann/spann on hdd/ssd ???
diskann designed for ssd!
but diskann on hdd ? so slow!!!
so what we do ?
- we optimize based on diskann?
- we design another xxxhddann?
only contribution: xxxhddann!
we can even name our paper as: "xxxhddann!"
e.g., "emberann".

if we remain on this track, potential attacks:
- system as a whole i like it, but nothing special, only contribution is hddann.

do we have other optimizations apart from xxxhddann??? on system level, beyond hddann?

不要挂羊头卖狗肉！卖系统还是只卖hddann?

on a FAST angle: HDD vs. SSD?

这个hddann有没有利用整个系统的信息？即，系统给我们送下来的workload有没有给我们的hddann一些信息？

如果我们卖系统，big picture上和system斗，ablation和diskann斗。

目前的想法：保留上面的布局。走到核心的道理之后：开宗明义要讲diskann。
- diskann designed for ssd. bad on hdd. so we design sth new.
- we get information from the upper compute layer, how to use this information.
    - runtime下层看着上层调节？额外的信息，偷看
    - runtime上层通知下层调节？额外的信息，主动给
    - 不需要runtime偷看或主动给，我们直接设计好了
这里的重点是：不要让别人攻击为什么不卖成单独的hddann？

minor addition: disaggregation --> insufficient compute on the storage side.
- we need to defense against this!!!
- make people understand this setting, why compute is not a big requirement here in the storage layer.

我的问题：
我想太多，想的越多就越不好，想的越多就越错，不要考虑那么多，就顺着这个写，不要有自己的想法。
make up experimental figures! don't be blocked on anything.
