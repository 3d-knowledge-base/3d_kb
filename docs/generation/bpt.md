# BPT

> *Scaling Mesh Generation via Compressive Tokenization*

BPT 是 mesh-native generation 路线中的关键论文之一。它最核心的贡献，不是单纯“压缩序列更短”，而是通过压缩 mesh tokenization，真正打开了**高面数 mesh 数据训练**的大门。

---

## 核心问题

传统 autoregressive mesh generation 最大的问题是序列太长：

- 一个 face 被拆成多个顶点 token
- 顶点坐标重复出现
- mesh 一复杂，attention 成本迅速失控

这会直接限制训练数据的 face 数，从而进一步限制模型能学到的细节上限。

所以 BPT 的核心判断是：

> 如果 tokenization 不先改好，后面的 mesh generator 再强也很难真正 scale。

---

## BPT = Blocked + Patchified

### 1. Block-wise Indexing

先把空间切成 block：

- 一个顶点先用 `block index` 表示它位于哪个 block
- 再用 `offset index` 表示它在 block 内的位置

这样做的好处是：

- 避免直接使用巨大的全局坐标词表
- 一个顶点最多只需要两个 token
- 相邻点落在同一 block 时还能进一步共享上下文

### 2. Patchified Aggregation

第二步解决的是 mesh 里最大的冗余：共享顶点反复出现在很多 face 中。

它通过聚合相邻 face 为 patch：

- 选 patch center
- 把周围相关 face 一起表示
- 显著减少重复顶点出现次数

结果不仅序列更短，局部性也更好。

---

## 为什么它重要

该论文最关键的实验结论在于：

> 一旦模型能吃下更高面数的训练 mesh，生成质量会明显提升。

也就是说，BPT 的价值不只是 token 压缩本身，而是它让模型能够：

- 使用 8k faces 级别的数据
- 学到更多几何细节
- 减少 incomplete mesh

这是一种非常直接的“表示改进 -> 数据上限提高 -> 生成质量提升”的链式作用。

---

## 在 mesh-native 路线中的位置

可以把这条线简单看成：

- MeshAnything V2：AMT，相邻 face 尽量共享边和顶点
- **BPT**：block-wise + patchified，把序列压到 0.26
- FACE：one-face-one-token，把建模层级直接推到 face level

所以 BPT 更像一个非常关键的中间节点：

- 它还没有彻底放弃 vertex-oriented tokenization
- 但已经显著改变了序列长度和数据可训练范围

---

## 局限

- 仍然属于 mesh sequence compression，而不是语义层级的根本切换
- 相比 FACE，建模单元仍然偏低级
- 主要解决的是“序列太长”，不是 mesh-native 表示的全部问题

---

## 一句话总结

BPT 的核心意义，是通过 `block-wise indexing + patchified aggregation` 把 mesh token 序列压缩到约 `0.26`，从而让 autoregressive mesh generation 能真正利用高面数训练数据，而高面数数据本身正是它提升质量的关键来源。
