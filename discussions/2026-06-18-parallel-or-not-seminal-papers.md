# 会话记录：Parallel-or-Not 经典文献与报告

**日期:** 2026-06-18  
**主题:** 用户将 SABES/fan-out 讨论泛化为「是否并行化」问题；要求下载顶级经典论文并写 brief report

## 用户核心观点

并行计算好，但**并非总是合适**：workload 不重时 single-node / 小 fan-out 足够；并行越多，network/scheduling overhead 越大。若 workload 极重、并行收益超过 overhead，再大规模并行才合理。

## 完成工作

1. 创建 `related-work/pdfs/`，下载 8 篇经典 PDF（Amdahl, Gustafson×2, Hill–Marty, Gunther USL, Lamport×2, Blumofe–Leiserson）
2. 撰写 `related-work/parallel-or-not-report.md` — 经典定律 + 分布式 ANN 映射 + SABES vs BES 启发式
3. 文件夹 README 记录未下载项（Lamport 1979 SC, Hennessy–Patterson 2019 CACM）

## 共识

- 用户 framing 与 Amdahl / Gustafson / USL / work-stealing 文献一致，可直接用于 paper motivation
- **Subset routing** = 减少无效并行度；**fan-out-all** = 接受 overhead 换 load 可预测性
- SABES 赢 BES 的故事 = **同 ANN 工作量、更少节点 touch**，是 parallel-or-not 在向量检索中的实例

## 未决

- Hennessy–Patterson 2019、Lamport 1979 无稳定 open PDF（已在 README 留链接）
- 交叉点量化（nprobe / bytes vs fan-out）待实验

## 关系

- 延续 `discussions/2026-06-18-why-spatial-colocation-despite-skew.md`
- 新报告：`related-work/parallel-or-not-report.md`
