---
comments: true
---

# LATTICE / VoxSet

> *LATTICE / VoxSet*

LATTICE 是当前 3D latent representation 文献线里重要的一篇，因为它不只是提出了一个新表示，还把很多争论说清楚了：

> 关键的不是 local vs global，而是 latent code 是否 **localizable**。

---

## 为什么需要 VoxSet

前面的路线各有优点：

- **VecSet**：紧凑、规则、适合 Transformer，但位置语义弱
- **Sparse voxel latent**：位置语义强、易做局部控制，但 token 更重

LATTICE 的判断是：

> 问题不在于到底选 global 还是 local，而在于 token 能不能在生成时被明确定位到某个空间位置。

所以它要做的是一件具体的事：

- 保留 VecSet 的紧凑性
- 同时引入 sparse voxel 的空间锚点

这就是 **VoxSet**。

---

## VoxSet 的核心思想

VoxSet 可以理解为一种**半结构化 latent**：

- token 不是像 dense voxel 那样完全绑定到细网格
- 但也不是像纯 VecSet 那样完全无显式位置锚点

它通过 **coarse voxel centers** 给 token 加锚点，再配合位置编码和训练策略，让模型学会：

> 给定一个位置锚点，生成该位置附近应该出现的几何细节。

因此，VoxSet 的强项恰恰不是“绝对更局部”，而是**更容易被定位和重采样**。

论文把这个想法落到两个很具体的设计上：

- `Voxel Queries`：查询点不再直接采样自表面，而是锚定在 coarse voxel center
- `Query Jitter`：训练时给 query 一个小扰动，缩小训练和测试之间的 query gap

这两步的作用都指向同一个目标：让 token 在 test time 也能被稳定定位。

---

## 为什么 `localizable code` 很重要

这一观点是该论文具有持久参考价值的贡献之一。

很多讨论会说：

- VecSet 太 global
- sparse voxel 更 local
- 所以后者更好

LATTICE 指出这一说法不够准确。

更准确的是：

- VecSet 的问题是 token 不够容易重新定位
- sparse voxel 的优势是 token 有天然 anchor
- 所以决定生成和 scaling 效果的，是 code 是否 **localizable**

这也解释了为什么 VoxSet 会特别强调：

- coarse spatial anchors
- query jitter
- position-aware generation
- test-time scaling

在 LATTICE 里，`localizable` 不只是一个口号，它直接进入了 DiT 的条件设计：

- noisy latent token 会接收基于空间位置的 `RoPE`
- 结构信息不是靠后处理补上，而是在 denoising 过程中持续参与

---

## 它在发展线里的位置

LATTICE 非常适合放在这条线中间理解：

- `3DShape2VecSet`：VecSet，紧凑
- `TRELLIS / SLAT`：structured latent，空间锚点强
- **LATTICE / VoxSet**：在 compactness 与 localizability 之间做折中
- `TRELLIS 2 / O-Voxel`：往更 native 的 3D asset representation 继续推进

所以 VoxSet 的重要性，不只是“提出一个新表示”，而是提供了一个更清晰的比较框架。

---

## 模型管线

LATTICE 采用两阶段生成：

1. 先得到 coarse sparse structure
2. 再在这些 voxel centers 上生成细节几何 latent

这和它的表示假设是对齐的：

- 第一阶段解决 `where`
- 第二阶段解决 `what`

论文自己就把这件事描述为把 3D generation 中混在一起的两个难题拆开。

---

## 实验上有哪些关键信息

### 重建能力

在几何重建实验里，LATTICE 随 token 数提升而稳定变好：

- `4096` tokens: `CD 5.321`, `F1 95.31`
- `8192` tokens: `CD 2.909`, `F1 98.53`
- `20480` tokens: `CD 1.893`, `F1 99.59`

对比里，`Direct3D-S2` 是 `CD 4.987`, `F1 97.46`。这说明 VoxSet 虽然更紧凑，但并没有明显牺牲高保真重建。

### 生成能力

论文报告的 `LATTICE-1.9B` 在几项 image-shape alignment 指标上达到或略高于同代方法：

- `ULIP-I = 0.130`
- `Uni-I = 0.315`

这里最值得注意的不是某一个分数，而是它在保持较低训练成本的同时，生成质量仍然能站在第一梯队。

### Test-time scaling

- 训练最多使用 `6144` tokens
- 测试时可以扩到 `12288`、`24576` 甚至更高
- token 增加后细节继续变丰富，而不是很快饱和

这正是 VoxSet 相比普通 VecSet 最核心的系统优势之一。

---

## 与其他表示的关系

### 相比 VecSet

- VecSet 更紧凑
- VoxSet 更可定位

### 相比 SLAT

- SLAT 更结构化、更适合局部编辑
- VoxSet 更轻，更适合 token scaling 和更紧凑的生成建模

### 相比 O-Voxel

- O-Voxel 更原生、更加 asset-like
- VoxSet 更偏 compact/localizable compromise

所以 VoxSet 很可能长期作为一个折中方案存在，而不是短期过渡。

---

## 为什么值得研究

对后续研究来说，VoxSet 主要的启发是：

- 3D latent 不一定非要高度结构化
- 也不一定非要完全 set-based 无位置
- 更合理的方向可能是：

> 用尽量少的结构，让 latent 获得足够强的位置语义。

这对 generation 和 editing 都有意义，因为很多控制任务其实都建立在 token “知道自己大致在哪” 这个前提之上。

---

## 局限

- 第一阶段仍然依赖 coarse structure generation，系统不是单阶段端到端生成
- 表示虽然比纯 VecSet 更可定位，但还不是像 O-Voxel 那样直接面向完整 asset 属性
- 论文中的很多优势依赖 test-time token scaling，推理成本会随 token 数继续上升

---

## 一句话总结

LATTICE / VoxSet 的主要价值，不只是提出了一种半结构化表示，而是把 3D latent 研究里的关键问题重新定义成了 `localizable code`：比起简单地争论 local 或 global，更重要的是 token 是否具备可定位、可重采样、可扩展的空间语义。
