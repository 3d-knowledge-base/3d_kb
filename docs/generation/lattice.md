---
comments: true
---

# LATTICE / VoxSet

> *LATTICE: Democratize High-Fidelity 3D Generation at Scale*

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

因此，VoxSet 的强项恰恰不是"绝对更局部"，而是**更容易被定位和重采样**。

论文把这个想法落到两个很具体的设计上：

- `Voxel Queries`：查询点不再直接采样自表面，而是锚定在 coarse voxel center
- `Query Jitter`：训练时给 query 一个小扰动 $\epsilon \sim U[-\frac{1}{2R}, \frac{1}{2R}]$（R 为最小目标分辨率），缩小训练和测试之间的 query gap

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

在 LATTICE 里，`localizable` 不只是一个口号，它直接进入了 DiT 的条件设计：

- noisy latent token 会接收基于空间位置的 `RoPE`
- 结构信息不是靠后处理补上，而是在 denoising 过程中持续参与

---

## 模型管线

### VoxSet VAE

VAE 采用 cross-attention encoder + symmetric decoder 架构：

- **Encoder**：8 层 self-attention + cross-attention，将 point cloud 压缩为 latent tokens
- **Decoder**：16 层，SDF grid 坐标作为 cross-attention 的 query
- 输入点云 $P \in \mathbb{R}^{N \times 7}$（3D 坐标 + 法线 + 锐边标记）
- 采样策略：均匀采样 + 锐边重要性采样

### 损失函数

VAE 训练沿用 Hunyuan3D-2 的设置：

- **SDF reconstruction loss**：对采样点的 SDF 值做回归
- **KL regularization**：标准 VAE latent 正则化

DiT 采用 **Rectified Flow** (SiT 形式)：

$$
\mathcal{L}_{flow} = \mathbb{E}_{t, x_0, \epsilon} \| v_\theta(x_t, t, c) - (x_0 - \epsilon) \|^2
$$

其中 $x_t = tx_0 + (1-t)\epsilon$，$c$ 为图像条件。训练时 10% 概率将条件替换为零（classifier-free guidance）。

### 两阶段生成

1. **Stage 1 — 结构生成**：用现成的预训练模型（如 Hunyuan3D-2 / Trellis）生成 coarse structure，体素化得到 voxel centers
2. **Stage 2 — 细节几何生成**：VoxSet DiT 在 voxel centers 上生成 latent tokens → VAE decode → Marching Cubes

---

## 训练配置

| 配置 | 参数 |
|:-----|:-----|
| 模型规模 | 0.6B / 1.9B / 4.5B |
| 图像条件 | DINOv2-Giant，输入 1022 分辨率 |
| Token scaling | 从 1024 逐步扩到 6144 |
| 优化器 | DeepSpeed ZeRO |
| Batch size | 最大 2048（适应 GPU 内存） |
| Base LR | 1e-4 → 逐阶段降至 1e-6 |

---

## 实验结果

### 重建能力（Tab 1, LATTICE-Bench）

| Token 数 | CD ↓ ($\times 10^4$) | F1 ↑ ($\times 10^2$) |
|:---------|:-----|:----|
| 4096 | 5.321 | 95.31 |
| 8192 | 2.909 | 98.53 |
| 20480 | 1.893 | 99.59 |

对比：Direct3D-S2 为 CD 4.987 / F1 97.46。VoxSet 在更紧凑的表示下达到了更高重建质量。

### 生成能力（Tab 2, LATTICE-1.9B）

| 指标 | LATTICE-1.9B |
|:-----|:-------------|
| ULIP-I | 0.130 |
| Uni-I | 0.315 |

在保持较低训练成本的同时，生成质量站在第一梯队。

### Test-time Scaling

- 训练最多使用 6144 tokens
- 测试时可扩到 12288、24576 甚至更高
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

---

## 局限

- 第一阶段仍然依赖 coarse structure generation，系统不是单阶段端到端生成
- 表示虽然比纯 VecSet 更可定位，但还不是像 O-Voxel 那样直接面向完整 asset 属性
- 论文中的很多优势依赖 test-time token scaling，推理成本会随 token 数继续上升

---

## 一句话总结

LATTICE / VoxSet 的主要价值，不只是提出了一种半结构化表示，而是把 3D latent 研究里的关键问题重新定义成了 `localizable code`：比起简单地争论 local 或 global，更重要的是 token 是否具备可定位、可重采样、可扩展的空间语义。
