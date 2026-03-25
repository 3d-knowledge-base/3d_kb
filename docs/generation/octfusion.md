---
comments: true
---

# OctFusion

> *OctFusion: Octree-based Diffusion Models for 3D Shape Generation (SGP 2025)*

OctFusion 的核心贡献是提出了一套**八叉树（octree）上的隐空间表示和多尺度统一扩散模型**，在保证输出 mesh 连续、流形的同时，用 2.5 秒在单卡上生成任意分辨率的 3D shape。

---

## 核心问题

3D 扩散生成面临一个效率和质量的两难：

**Dense grid 路线**（如 latent 在 $64^3$ 或 $128^3$ 体素上）：
- 分辨率受限，细节不够
- 显存占用随分辨率立方增长

**Cascaded diffusion 路线**（如先生成粗分辨率再逐步上采样）：
- 需要多个独立的扩散模型
- 模型间的误差会累积
- 训练和推理都更复杂

**Implicit 路线**（如纯 NeRF/SDF + diffusion）：
- 不保证输出的连续性和流形性
- mesh 提取是后处理步骤，质量不可控

OctFusion 想要：高分辨率、快速生成、保证 mesh 质量、一个模型搞定。

---

## 方法概述

### Octree-based latent representation

OctFusion 将 3D 空间用八叉树进行自适应划分：

- 表面附近用高分辨率节点（深层叶子）
- 远离表面的区域用低分辨率节点（浅层叶子）
- 每个叶子节点存储一个局部 latent feature

这种表示的好处：
- **自适应分辨率**：计算集中在表面附近
- **结构化**：八叉树天然具有多尺度层级结构
- **高效**：同样的精度下，节点数远少于 dense grid

### Octree VAE

用一个基于八叉树结构的变分自编码器学习 latent representation：

- 编码器沿八叉树从细到粗聚合信息
- 解码器从粗到细恢复 SDF 值
- 结合 implicit representation 和 explicit 八叉树结构

输出的 SDF 在八叉树叶子节点上采样，可以直接用 Marching Cubes 提取为连续流形 mesh。

### 统一多尺度 U-Net 扩散模型

这是 OctFusion 的关键设计：

- **一个 U-Net** 处理八叉树所有层级，不是每个层级一个模型
- U-Net 的不同分辨率层自然对应八叉树的不同深度
- 权重和计算在不同尺度间共享

好处：
- 避免了 cascaded diffusion 的多模型问题
- 不同尺度的信息可以交互，全局结构和局部细节同时优化
- 训练更简单，只需训一个模型

---

## 性能关键数字

- **生成速度**：2.5 秒（单张 Nvidia 4090）
- **输出质量**：连续、流形 mesh，无需后处理修复
- **条件生成**：支持文本、sketch、类别标签、color field
- **数据集**：ShapeNet 和 Objaverse 上都做了验证

---

## 为什么值得关注

### 1. 统一多尺度设计

cascaded diffusion 在 3D 生成中很常见（如 TRELLIS 在不同尺度用不同模型），但会带来级联误差和训练复杂度。OctFusion 证明了用一个统一 U-Net 可以搞定所有尺度。

### 2. 八叉树表示的实用性

八叉树是图形学中经典的空间数据结构。OctFusion 将它与现代 latent diffusion 结合，是一个很自然但执行良好的方案。

### 3. Mesh 质量保证

由于八叉树 + SDF + MC 的管线，输出 mesh 保证是连续的流形。这对下游应用（物理仿真、渲染管线）很重要。

---

## 与其他工作的关系

### 相比 TRELLIS / SLAT

- TRELLIS 用 structured latent + 不同尺度的独立模型
- OctFusion 用八叉树 + 统一多尺度 U-Net，架构更简洁
- TRELLIS 支持更多输出格式（3DGS + mesh），OctFusion 更专注于 mesh

### 相比 Sparc3D / SparseFlex

- 都是稀疏 3D 表示路线，但结构不同
- Sparc3D 用稀疏卷积在均匀 sparse grid 上
- OctFusion 用八叉树的自适应层级结构

### 相比 Direct3D-S2

- Direct3D-S2 也做 3D latent diffusion
- 但用的是 dense latent grid，不是 octree

### 相比 mesh-native 路线（BPT/FACE/Nautilus）

- OctFusion 仍然是 implicit representation → mesh extraction 的路线
- 不直接生成 mesh 的顶点和面，而是先生成 SDF 再提取

---

## 优势与局限

### 优势

- 统一多尺度 U-Net 避免了 cascaded diffusion 的复杂性
- 八叉树表示兼顾效率和分辨率
- 输出 mesh 保证连续流形
- 生成速度快（2.5s 单卡）
- SGP 2025 接收

### 局限

- 仍然依赖 SDF + Marching Cubes，继承了 isosurface 路线的局限（难以表达非流形结构）
- 八叉树结构相比纯 sparse voxel 对实现有额外要求
- 生成的 mesh 拓扑是由 MC 决定的，面数和面分布不可控
- 纹理生成（color field）是附加功能，不如专门的纹理方法精细

---

## 一句话总结

OctFusion 用八叉树组织 3D latent space 并设计统一的多尺度 U-Net 扩散模型，在避免 cascaded diffusion 复杂性的同时实现了高效、高质量、保证流形的 3D shape 生成。
