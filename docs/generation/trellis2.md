---
comments: true
---

# TRELLIS 2

> *TRELLIS 2*

TRELLIS 已经通过 SLAT 把 3D latent 变成"有结构、可编辑"的表示；TRELLIS 2 则进一步追问：

> 如果底层几何仍然主要依赖 SDF / isosurface，3D 生成是否仍然停留在一个不够原生的表示层？

它给出的回答是 **O-Voxel**。

---

## 从 SLAT 到 O-Voxel

TRELLIS 的主要贡献是 SLAT：

- 稀疏空间位置负责结构
- 局部 latent 负责细节

这已经很强，但仍然存在几个问题：

- SDF 对开放表面不友好
- 对非流形结构支持不自然
- 几何和材质往往不是原生一体化
- 高分辨率 structured latent 成本仍然偏高

TRELLIS 2 的方向变化在于：

> 不只是让 latent "有结构"，而是要让 latent 更接近 3D 资产结构本身。

---

## O-Voxel 的核心思想

O-Voxel 可以理解为一种更原生的结构化 3D 单元表示。

它和传统 SDF latent 的区别在于：

- 不再把几何等同为一个待抽壳的连续场
- 而是直接在结构化单元中编码更原生的几何/材质信息

这使它更自然地处理：

- **开放表面**
- **非流形结构**
- **带内部结构的对象**
- **原生 PBR 材质**

每个活动体素编码以下信息：

**Shape features** $f_i^{shape}$：

- dual vertex $v \in \mathbb{R}^3$：体素内的 mesh 顶点偏移
- edge intersection flags $\delta \in \{0,1\}^3$：三条轴对齐边是否与表面相交
- splitting weights $\gamma \in \mathbb{R}^1$：用于局部三角化的分裂权重

**Material features** $f_i^{mat}$：

- $c_{\text{basecolor}}$：PBR 基础颜色
- $m_{\text{metallic}}$：金属度
- $r_{\text{roughness}}$：粗糙度
- $\alpha_{\text{opacity}}$：透明度

整体设计基于 Flexible Dual Grid（受 Dual Contouring 启发），是 field-free 的——不需要 SDF 或任何连续场。Mesh↔O-Voxel 之间可以在 CPU 上数秒内完成双向即时转换。

---

## SC-VAE

TRELLIS 2 同时提出了 **SC-VAE**（Sparse Convolutional VAE），目的是进一步提高压缩效率。

### 架构设计

SC-VAE 是一个**纯稀疏卷积**架构（非 Transformer）：

- 采用 **Sparse Residual Autoencoding**：通道-空间信息交换机制
- 残差块使用 **ConvNeXt-style** 设计优化
- 包含 **Early-pruning upsampler**：在上采样早期剪掉不需要的体素
- 16× 空间下采样，一个 $1024^3$ fully-textured asset 可压到约 9.6K latent tokens

### 损失函数

训练分为两个阶段：

**Stage 1（几何 + 材质重建）**：

$$
\mathcal{L}_{s1} = \lambda_v |v̂ - v|^2 + \lambda_\delta \text{BCE}(\hat{\delta}, \delta) + \lambda_\rho \text{BCE}(\hat{\rho}, \rho) + \lambda_{mat} |f̂^{mat} - f^{mat}|_1 + \lambda_{KL} \mathcal{L}_{KL}
$$

- $v̂, v$：dual vertex 预测/GT
- $\hat{\delta}, \delta$：edge intersection flags
- $\hat{\rho}, \rho$：occupancy
- $f̂^{mat}, f^{mat}$：材质特征

**Stage 2（加入渲染监督）**：

$$
\mathcal{L}_{s2} = \mathcal{L}_{s1} + \mathcal{L}_{render}
$$

其中 $\mathcal{L}_{render}$ 包含 mask/depth/normal 的 L1、SSIM 和 LPIPS 损失。

---

## 生成系统

TRELLIS 2 采用三阶段 DiT 生成：

| 阶段 | 功能 | 模型规模 |
|:-----|:-----|:---------|
| Stage 1 | 结构生成（占据/非占据） | 1.3B DiT |
| Stage 2 | 几何细节填充 | 1.3B DiT |
| Stage 3 | 材质生成 | 1.3B DiT |
| **总计** | | **~4B params** |

训练配置：32×H100，batch size 256，AdamW lr 1e-4，训练资产规模约 800K。

推理效率：

- $512^3$ 级别 fully-textured asset 约 3s
- $1024^3$ 约 17s
- $1536^3$ 约 60s

---

## 实验结果

### Reconstruction（Tab 1, Toys4K + Sketchfab）

TRELLIS 2 在 Toys4K 和 Sketchfab 上的 MD、CD、F1、PSNR 等指标均优于 TRELLIS v1、Direct3D-S2、SparseFlex 和 Dora。

### Generation（Tab 2）

| 指标 | TRELLIS 2 |
|:-----|:----------|
| CLIP | 0.894 |
| ULIP-2 | 0.477 |
| Uni3D | 0.436 |
| User study preference | 66.5% / 69.0% |

### Ablation（Tab 3）

Sparse Residual Autoencoding 和 Optimized ResBlock 均对重建质量有明显贡献。

---

## 它在整条文献线里的位置

这条发展线可以概括为：

- `3DShape2VecSet`：先有紧凑 latent set
- `TRELLIS / SLAT`：再让 latent 结构化、可编辑
- **TRELLIS 2 / O-Voxel**：进一步让 latent 更接近 native 3D asset

因此 TRELLIS 2 的意义不在于对 TRELLIS 的渐进式改进，而在于表示路线层面的升级。

---

## 与其他方法的关系

### 相比 TRELLIS v1

- v1 更强调"结构化 latent"
- v2 更强调"native structured latent"

### 相比 LATTICE / VoxSet

- VoxSet 更像 compactness 与 localizability 的折中
- O-Voxel 更像往原生 3D 资产单元推进

### 相比 BPT / FACE

- O-Voxel 仍然是在 latent representation 路线上推进
- BPT / FACE 则在 mesh-native token space 上推进

两者都在追求"更原生"，但方向不同。

---

## 局限

- 尽管 latent 已经更紧凑，但整体系统仍然是大模型路线（~4B params），训练成本并不低
- 形状与材质采用三阶段分别生成，系统复杂度高于单阶段方案
- O-Voxel 更接近 asset representation，但并不等于直接在最终 mesh 编辑空间里工作

---

## 一句话总结

TRELLIS 2 的主要价值，是把 3D latent 表示从"有结构的中间表示"进一步推进到"更原生的 3D 资产表示"，而 O-Voxel + SC-VAE 代表的正是这一方向：不仅要可生成、可压缩，还要更自然地支持开放表面、非流形结构和原生材质。
