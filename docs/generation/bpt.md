---
comments: true
---

# BPT

> *Scaling Mesh Generation via Compressive Tokenization*

BPT 是 mesh-native generation 路线中的关键论文之一。它主要的贡献，不是单纯“压缩序列更短”，而是通过压缩 mesh tokenization，打开了**高面数 mesh 数据训练**的大门。

---

## 核心问题

传统 autoregressive mesh generation 主要的问题是序列太长：

- 一个 face 被拆成多个顶点 token
- 顶点坐标重复出现
- mesh 一复杂，attention 成本迅速失控

这会直接限制训练数据的 face 数，从而进一步限制模型能学到的细节上限。

所以 BPT 的核心判断是：

> 如果 tokenization 不先改好，后面的 mesh generator 再强也很难 scale。

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

第二步解决的是 mesh 里主要的冗余：共享顶点反复出现在很多 face 中。

它通过聚合相邻 face 为 patch：

- 选 patch center
- 把周围相关 face 一起表示
- 减少重复顶点出现次数

结果不仅序列更短，局部性也更好。

### 3. 条件生成骨干

论文并不是只提出一种 tokenization，而是基于 BPT 训练了一个 autoregressive mesh foundation model：

- 条件可以来自图像，也可以来自点云
- Transformer 直接在压缩后的 mesh token 序列上建模
- 目标不是只生成低面数草图，而是生成可直接使用的高面数 mesh

因此，BPT 的论文贡献其实分成两层：

- 表示层：把 mesh sequence 压短
- 系统层：验证压缩后确实能支撑更大规模 mesh generation

---

## 为什么它重要

该论文主要的实验结论在于：

> 一旦模型能吃下更高面数的训练 mesh，生成质量会明显提升。

也就是说，BPT 的价值不只是 token 压缩本身，而是它让模型能够：

- 使用 8k faces 级别的数据
- 学到更多几何细节
- 减少 incomplete mesh

这是一种直接的“表示改进 -> 数据上限提高 -> 生成质量提升”的链式作用。

---

## 实验上到底验证了什么

这篇论文比较有说服力的地方，在于它没有把收益只停留在“压缩率更高”这一层，而是把压缩和生成质量直接联系起来。

### 压缩率

- BPT 相比原始 mesh sequence 长度压缩约 `75%`
- 论文把这个结果表述为 sequence length 降到原来的大约 `0.26`

这件事的直接意义是：模型终于能稳定吃下更多高面数训练数据。

### 点云到 Mesh

论文报告的点云条件生成结果里：

- `MeshAnything`: Hausdorff `0.301`，Chamfer `0.136`
- `MeshAnythingV2`: Hausdorff `0.265`，Chamfer `0.114`
- `BPT`: Hausdorff `0.166`，Chamfer `0.094`

这个提升幅度说明，BPT 不只是让序列更短，而是确实让最终 mesh 更接近高面数参考形状。

### 图像到 Mesh

- 相比 EdgeRunner，BPT 生成结果保留了更多局部结构
- 论文展示的可视化里，复杂边缘和表面细节更完整
- 这和它能利用更高面数训练数据是对应的

### 消融实验

- block size / offset size 不是越极端越好
- 在固定量化分辨率下，论文里 `block size = 8`、`offset size = 16` 的结果最好
- 只靠截断长序列、测试时再用 sliding window attention，并不能替代 BPT，反而更容易产生不完整 mesh

---

## 在 mesh-native 路线中的位置

可以把这条线简单看成：

- MeshAnything V2：AMT，相邻 face 尽量共享边和顶点
- **BPT**：block-wise + patchified，把序列压到 0.26
- FACE：one-face-one-token，把建模层级直接推到 face level

所以 BPT 更像一个关键的中间节点：

- 它还没有彻底放弃 vertex-oriented tokenization
- 但已经改变了序列长度和数据可训练范围

---

## 局限

- 仍然属于 mesh sequence compression，而不是语义层级的根本切换
- 相比 FACE，建模单元仍然偏低级
- 主要解决的是“序列太长”，不是 mesh-native 表示的全部问题
- 论文自己也提到，目前模型参数规模仍然不算大，后续还可以继续做更大规模的 scaling
- Transformer 仍然按序列方式建模，mesh 本身的拓扑归纳偏置没有被充分利用

---

## 一句话总结

BPT 的主要意义，是通过 `block-wise indexing + patchified aggregation` 把 mesh token 序列压缩到约 `0.26`，从而让 autoregressive mesh generation 能利用高面数训练数据，而高面数数据本身正是它提升质量的关键来源。
