# LATTICE / VoxSet

> *LATTICE / VoxSet*

LATTICE 是当前 3D latent representation 文献线里非常关键的一篇，因为它不只是提出了一个新表示，还把很多争论说清楚了：

> 真正关键的不是 local vs global，而是 latent code 是否 **localizable**。

---

## 为什么需要 VoxSet

前面的路线各有优点：

- **VecSet**：紧凑、规则、适合 Transformer，但位置语义弱
- **Sparse voxel latent**：位置语义强、易做局部控制，但 token 更重

LATTICE 的判断是：

> 问题不在于到底选 global 还是 local，而在于 token 能不能在生成时被明确定位到某个空间位置。

所以它要做的是一件非常具体的事：

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

---

## 为什么 `localizable code` 很重要

这一观点是该论文最具持久参考价值的贡献之一。

很多讨论会说：

- VecSet 太 global
- sparse voxel 更 local
- 所以后者更好

LATTICE 指出这一说法不够准确。

更准确的是：

- VecSet 的问题是 token 不够容易重新定位
- sparse voxel 的优势是 token 有天然 anchor
- 所以真正决定生成和 scaling 效果的，是 code 是否 **localizable**

这也解释了为什么 VoxSet 会特别强调：

- coarse spatial anchors
- query jitter
- position-aware generation
- test-time scaling

---

## 它在演化线里的位置

LATTICE 非常适合放在这条线中间理解：

- `3DShape2VecSet`：VecSet，极致紧凑
- `TRELLIS / SLAT`：structured latent，空间锚点强
- **LATTICE / VoxSet**：在 compactness 与 localizability 之间做折中
- `TRELLIS 2 / O-Voxel`：往更 native 的 3D asset representation 继续推进

所以 VoxSet 的重要性，不只是“提出一个新表示”，而是提供了一个更清晰的比较框架。

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

对后续研究来说，VoxSet 最重要的启发是：

- 3D latent 不一定非要极端结构化
- 也不一定非要完全 set-based 无位置
- 更合理的方向可能是：

> 用尽量少的结构，让 latent 获得足够强的位置语义。

这对 generation 和 editing 都有意义，因为很多控制任务其实都建立在 token “知道自己大致在哪” 这个前提之上。

---

## 一句话总结

LATTICE / VoxSet 的核心价值，不只是提出了一种半结构化表示，而是把 3D latent 研究里的关键问题重新定义成了 `localizable code`：比起简单地争论 local 或 global，更重要的是 token 是否具备可定位、可重采样、可扩展的空间语义。
