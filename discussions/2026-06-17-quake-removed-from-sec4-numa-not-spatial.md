# 2026-06-17 — Quake removed from §4 (NUMA ≠ spatial colocation)

## 主题

用户阅读 Quake 后指出：论文未讨论 **embedding 空间邻近 cluster 的 colocation**；其 locality 是 **IVF partition → NUMA socket 的 affinity**，属于硬件内存局部性，与 §4（SABES/ADBV 式 centroid/bucket colocation）不同。

## 共识

**同意用户判断。**

Quake 机制（PDF 核对）：
- **Adaptive hierarchical k-means** + split/merge 应对 **access skew**
- **APS** 动态 nprobe
- **NUMA data placement：** `assigns index partitions to specific NUMA nodes` + **affinity-based scheduling** + 同 socket 内 work stealing
- Top-down **nearest-centroid** 扫描是 **IVF 搜索路由**，不是把空间上相邻的 cluster 刻意放到同一机器

因此 Quake 属于 **§3 Category 3**（partition-intrinsic + dynamic maintenance + NUMA 执行优化），**不属于 §4**。

## 变更

- 从 §4 master table、§4.4 详述删除 Quake
- §4.6 excluded 表增加 Quake 及原因
- §4.3 query-skew 仍 **引用 Quake 为 §3 类比**（access skew / split-merge），不再列为 §4 论文
- `quake.pdf`：`spatial-related/` → `related-work/pdfs/quake.pdf`
- §3 Quake 条目：去掉 `§4 centroid-locality` 标记；Local PDF 更新

## §4 论文数

13 → **12**

## 与旧记录关系

- 修正 `2026-06-17-sec4-full-rewrite-query-skew.md` 中将 Quake 列为 §4 论文的结论
- 与 HARMONY 移出 §4 同理，但 Quake 保留在 §4.3 作为 **query-skew 相关 work**（§3 侧）
