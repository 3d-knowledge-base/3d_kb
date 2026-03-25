---
comments: true
---

# Nautilus

> *Nautilus: Locality-aware Autoencoder for Scalable Mesh Generation (ICCV 2025)*

Nautilus 的关注点是 mesh-native 自回归生成中最核心的瓶颈之一：**如何把非结构化的三角 mesh 压缩成足够短、又保留拓扑关系的 token 序列**，从而让 AR 模型能生成更大规模（5000 面）的 mesh。

---

## 核心问题

现有 mesh AR 生成方法（如 MeshGPT、MeshAnything）直接将每个三角面序列化为一组顶点坐标 token。这种做法有两个根本问题：

- **序列膨胀**：每个面需要 9 个坐标 token（3 个顶点 x 3 坐标），1000 面就需要 9000 个 token。序列长度直接制约了生成面数上限。
- **拓扑信息丢失**：面的排列顺序是人为选择的（如 z-order sorting），不反映 mesh 上真实的邻接关系。模型需要从坐标层面隐式学习拓扑结构，效率低。

这两个问题共同导致现有方法难以超过 800-1600 面。

---

## 方法概述

Nautilus 的核心思路是利用流形 mesh 的**局部性质**来实现高效压缩。

### Locality-aware tokenization

关键观察：在一个流形三角 mesh 上，相邻面共享边和顶点。如果按照面的拓扑邻接关系来排列序列，那么大量顶点和边可以被相邻面复用，无需重复编码。

具体做法：

- 用 BFS/面邻接遍历确定面的排列顺序，使得序列中相邻 token 对应的面在 mesh 上也是邻居
- 共享顶点和边只编码一次，后续引用通过索引完成
- 这将每个面平均所需的新 token 数从 9 降到约 3

这种压缩不是有损的——它是在利用 mesh 本身的冗余结构。

### Locality-aware Autoencoder

在 tokenization 基础上，Nautilus 训练了一个 autoencoder：

- **编码器**：接收 locality-aware token 序列，输出压缩的 latent codes
- **解码器**：从 latent codes 恢复完整的 mesh token 序列
- 整个 AE 在 token 空间工作，保留了面间的拓扑关系

### Dual-stream Point Conditioner

为了在生成时提供几何引导（通常以点云形式给出条件），Nautilus 设计了一个双流条件模块：

- **全局流**：捕捉整体形状分布
- **局部流**：提取细粒度的局部几何特征

双流特征融合后注入 AR transformer，确保生成的 mesh 在全局一致的同时保持局部结构精度。

---

## 为什么值得关注

### 1. 面数扩展到 5000

在 Nautilus 之前，mesh-native AR 方法的上限大约在 800 面（MeshGPT）到 1600 面（MeshAnything v2）。Nautilus 将这个数字推到 5000，是一个数量级上的进步。

### 2. 压缩思路可迁移

locality-aware tokenization 的核心思想——利用 mesh 拓扑结构做序列压缩——并不绑定于特定的生成模型，后续的 MeshRipple、FlashMesh 等工作都在这个方向上继续发展。

### 3. 质量与可扩展性兼顾

实验表明 Nautilus 在 ShapeNet 和 Objaverse 上都超过了同期方法，在保持拓扑完整性的同时提升了生成的几何精度。

---

## 与其他工作的关系

### 相比 MeshGPT / MeshAnything

- MeshGPT 和 MeshAnything 对每个面独立编码，序列长度随面数线性增长
- Nautilus 利用面间共享结构做压缩，有效降低了序列长度

### 相比 BPT / FACE

- BPT 和 FACE 也做 mesh tokenization，但思路不同：BPT 用 BSP-tree 分割，FACE 做 face-level codebook
- Nautilus 更强调局部邻接关系的利用，走的是拓扑感知压缩路线

### 相比后续 MeshRipple

- MeshRipple 进一步发展了 BFS tokenization 的思想，加入了 frontier-aware 机制和 sparse-attention global memory
- Nautilus 是这条技术线的重要先驱

---

## 优势与局限

### 优势

- 将 mesh-native AR 生成的面数上限从 ~1600 推到 5000
- locality-aware tokenization 是有道理的压缩策略，不是暴力截断
- 双流条件机制平衡了全局一致性和局部精度
- ICCV 2025 接收，方法经过同行评审

### 局限

- 仍然是三角 mesh 生成，不直接处理四边形
- 5000 面对于实际生产管线仍然偏少（美术资产通常需要数万面）
- 自回归生成速度本身就是瓶颈，压缩序列长度缓解但不消除这个问题
- 流形假设限制了对非流形 mesh 的处理能力

---

## 一句话总结

Nautilus 利用流形 mesh 的面间拓扑邻接关系做序列压缩，通过 locality-aware tokenization 和 autoencoder 将 mesh-native 自回归生成的规模从 ~1600 面推到 5000 面，是 mesh AR 生成可扩展性方向上的一个关键进展。
