---
comments: true
---

# VAT

> *3D Representation in 512-Byte: Variational Tokenizer is the Key for Autoregressive 3D Generation*

VAT 关注的核心问题是：**如何将 3D shape 压缩到极端紧凑的 token 表示中，使其适合大语言模型风格的自回归生成。** 它实现了 250x 压缩比（1MB mesh → 3.9KB），是目前 3D tokenization 中压缩率最高的方案。

---

## 核心问题

将自回归 transformer 用于 3D 生成面临一个根本挑战：

**3D 数据缺乏天然的序列结构和多尺度层级。**

对比图像 tokenization：

- 图像有规整的 patch 网格，天然的扫描顺序，以及多尺度金字塔
- VQ-VAE/VQGAN 可以将图像压缩为数百个 discrete token
- GPT 风格的 AR 模型可以直接在这些 token 上工作

3D 数据的困难：

- 无序的点云或 mesh 没有天然的扫描顺序
- 3D 特征之间的多尺度关系不像图像那样明确
- 现有方法要么压缩不够（token 太多），要么压缩过度（丢失结构细节）

---

## 方法概述

### In-context Transformer 压缩

VAT 的第一步是将大量无序的 3D 特征压缩为少量有序 token：

- 输入：从 3D shape 提取的一组无序特征（如点云特征）
- 用 in-context transformer 做 cross-attention，将 N 个无序特征压缩到 M 个 learnable query token 中（M << N）
- 关键在于信息损失最小化——通过重建损失确保压缩是 near-lossless 的

### 变分量化：映射到高斯分布

压缩后的 latent token 被映射到一个高斯分布空间：

- 不是直接做 VQ（向量量化），而是 variational approach
- latent space 是连续的高斯分布
- 然后用 residual quantization（残差量化）将连续 latent 离散化

### 多尺度 token 分配

VAT 的一个关键设计：不同尺度的 token 从同一个高斯分布的不同子空间中分配。

- 粗尺度：少量 token，编码全局形状
- 细尺度：更多 token，编码局部细节
- token 数量跨尺度递增

这构建了一个**隐式的层级结构**：

- 粗到细的 token 之间的关系是在同一个连续空间中定义的
- AR 模型可以自然地按从粗到细的顺序生成

### 高分辨率 Triplane 解码

解码时，compact latent tokens 被展开为高分辨率的 triplane representation：

- triplane 可以编码细致的 3D 几何
- 从 triplane 查询 SDF 并用 MC 提取 mesh

---

## 压缩率数字

| 指标 | 数值 |
|---|---|
| 1MB mesh → latent | 3.9KB (250x 压缩) |
| 进一步压缩 | 256 个 int8 token (2000x 压缩) |
| 250x 压缩时 F-score | 96% |
| 2000x 压缩时 F-score | 92% |

这些数字意味着一个完整的 3D shape 可以用 256 个整数表示，与图像 token 数量相当。

---

## 为什么值得关注

### 1. 对接 LLM 架构

如果 3D shape 能用 256 个 token 表示，那它就可以直接放进 GPT 架构的上下文窗口中。这为"multimodal LLM 理解和生成 3D"打开了门。

### 2. 隐式层级结构

多尺度 token 从同一高斯分布分配的设计很优雅——它不需要显式定义层级关系，层级是从分布子空间中自然涌现的。

### 3. 验证了极端压缩的可行性

96% F-score 的 250x 压缩说明 3D shape 的内在维度远低于其原始表示，存在大量可以被利用的冗余。

---

## 与其他工作的关系

### 相比 3DShape2VecSet / LATTICE

- 3DShape2VecSet 用 cross-attention 将 3D 特征编码为 latent set，思路类似
- LATTICE 用 VQ-VAE 做 3D tokenization
- VAT 的压缩比远高于两者

### 相比 Nautilus / BPT（mesh-native tokenization）

- Nautilus/BPT 在 mesh 的顶点和面层面做 token 化
- VAT 在 3D shape 的隐空间层面做 token 化
- 两条路线目标不同：前者保留 mesh 结构，后者最大化压缩

### 相比 TRELLIS 的 SLAT

- SLAT 也是 latent space tokenization，但保留了空间结构（structured latent）
- VAT 更侧重极端压缩，牺牲空间结构换取更少的 token 数

---

## 优势与局限

### 优势

- 极端压缩比（250x-2000x）使 3D 生成与 LLM 架构对接成为可能
- 多尺度隐式层级结构设计优雅
- near-lossless 压缩（96% F-score at 250x）
- 适合 coarse-to-fine 自回归生成

### 局限

- 极端压缩必然在某些细节上有损，92% F-score at 2000x 意味着约 8% 的几何信息丢失
- 解码依赖 triplane + MC，继承了这条管线的分辨率限制
- 压缩为无结构 token 后，空间局部性信息丢失，不利于局部编辑
- 尚未在大规模 Objaverse 全集上验证

---

## 一句话总结

VAT 通过 in-context transformer 压缩、变分量化和多尺度 token 分配，将 3D shape 压缩为 256 个 token（250x-2000x 压缩比），是目前最紧凑的 3D tokenization 方案，为 3D 自回归生成与 LLM 架构的对接提供了关键基础。
