---
comments: true
---

# Sparc3D

> *Sparc3D: Sparse Representation and Construction for High-Resolution 3D Shapes Modeling*

Sparc3D 关注的问题是：**现有的 3D VAE 在压缩和重建时会损失大量细节，这是因为表示方式和监督信号之间存在模态不匹配。** 它的回答是用一套完全基于稀疏卷积的 3D 原生管线来消除这个 gap。

---

## 核心问题

当前主流的 3D 生成管线几乎都是两阶段：

1. **VAE 压缩**：把 3D shape 编码到 latent space
2. **Latent diffusion**：在 latent space 做扩散生成

问题出在第一步。现有 VAE 的两种常见路线各有缺陷：

**2D 监督路线**（如 TripoSG 的多视角渲染损失）：

- 编码器/解码器可能用 2D backbone
- 3D 几何信息经过 2D 投影再重建，细节丢失不可避免
- 存在模态不匹配：输入是 3D mesh，监督是 2D 渲染

**3D 监督但用 dense grid 路线**：

- 分辨率受限于显存：dense $256^3$ 已经是极限
- 远不够捕捉细节几何

---

## 方法概述

Sparc3D 包含两个核心组件：

### Sparcubes：稀疏可变形 Marching Cubes 表示

Sparcubes 可以看作 FlexiCubes 的稀疏版本：

- 将 mesh 转换为稀疏立方体网格上的 SDF + 变形场
- 只保留表面附近的立方体，而非整个空间的 dense grid
- 分辨率可达 **1024^3**，因为稀疏存储大幅降低显存需求
- 支持开放表面、断开组件、复杂拓扑
- 整个表示是**可微的**，可以通过渲染损失做端到端优化

与 SparseFlex 的思路相似，但 Sparc3D 更强调与下游 VAE 的模态一致性。

### Sparconv-VAE：纯 3D 稀疏卷积 VAE

关键设计决策：**编码器和解码器完全用稀疏 3D 卷积网络构建**。

- **编码器**：接收 Sparcubes 表示，通过稀疏卷积逐步压缩
- **latent space**：也是稀疏 3D 特征图，不是 2D 特征或 unstructured set
- **解码器**：稀疏卷积上采样，直接输出 Sparcubes 表示

这意味着整个管线中没有 2D → 3D 或 3D → 2D 的模态转换。输入是 3D，中间是 3D，输出是 3D。论文称之为 **modality-consistent**（模态一致）。

---

## 为什么"模态一致"很重要

以 TripoSG 为例：

1. 输入 mesh → 多视角渲染 → 2D 编码器 → latent
2. latent → 解码器 → SDF/FlexiCubes → mesh

每次 2D-3D 转换都是有损的。TripoSG 的生成质量已经很好，但如果放大到细节级别，几何精度仍然受到这个管线的制约。

Sparc3D 消除了所有中间的模态转换，直接在 3D 空间做压缩和重建。实验表明，在同等 latent 维度下，Sparconv-VAE 的重建保真度高于用 2D 监督训练的 VAE。

---

## 与 latent diffusion 的集成

Sparconv-VAE 产出的 latent 是结构化的稀疏 3D 特征图。在此之上可以直接用 3D latent diffusion 做生成：

- 扩散模型也可以用稀疏卷积 U-Net
- 整个管线保持 3D 原生，不需要转换

---

## 与其他工作的关系

### 相比 SparseFlex

- SparseFlex 也是稀疏 isosurface 表示，但更偏基础设施层
- Sparc3D 在 SparseFlex 基础上构建了完整的 modality-consistent VAE pipeline
- Sparc3D 更强调 VAE 中的模态匹配问题

### 相比 TRELLIS / SLAT

- TRELLIS 用 structured latent 但 VAE 训练时仍有 2D 渲染监督
- Sparc3D 完全在 3D 空间训练，没有 2D 监督

### 相比 TripoSG / Hunyuan3D

- TripoSG 和 Hunyuan3D 的 VAE 依赖多视角渲染损失
- Sparc3D 用 3D 几何损失直接监督，模态更一致

### 相比 OctFusion

- OctFusion 用八叉树隐表示 + 多尺度扩散
- Sparc3D 用 Sparcubes 稀疏表示 + 稀疏卷积 VAE
- 两者都是 3D 原生路线，但表示方式不同

---

## 优势与局限

### 优势

- 模态一致性：端到端 3D 原生管线，无 2D-3D 模态转换
- 高分辨率：支持 1024^3 稀疏重建
- 支持复杂拓扑（开放表面、断开组件）
- 重建保真度在同类方法中领先

### 局限

- 稀疏卷积实现对工程基础设施要求高（需要 MinkowskiEngine / TorchSparse 等）
- 仍然是 isosurface 路线，最终 mesh 质量受限于 Marching Cubes 变体
- 训练数据需要预处理为 Sparcubes 格式
- 论文尚未在顶会发表

---

## 一句话总结

Sparc3D 通过 Sparcubes 稀疏可变形 Marching Cubes 表示和纯 3D 稀疏卷积 VAE（Sparconv-VAE），构建了一套模态一致的 3D 生成管线，消除了 2D-3D 模态转换带来的细节损失，在 1024^3 分辨率下实现了高保真 3D shape 重建和生成。
