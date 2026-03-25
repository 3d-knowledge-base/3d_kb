---
comments: true
---

# MeshGPT

> *MeshGPT: Generating Triangle Meshes with Decoder-Only Transformers* (CVPR 2024)

MeshGPT 是 mesh-native 自回归生成的开创性工作，首次证明了可以用 decoder-only transformer 直接自回归地生成三角网格的顶点和面片序列，而不需要经过隐式场 + Marching Cubes 的间接路径。

---

## 核心问题

在 MeshGPT 之前，3D mesh 生成主要走两条路：

- **隐式场路线**（SDF / Occupancy → Marching Cubes）：生成的 mesh 是密集且不规则的三角面片，拓扑质量差，不适合下游使用
- **早期序列化尝试**（PolyGen）：直接逐坐标预测顶点，但缺乏对局部几何和拓扑结构的建模能力

MeshGPT 的思路是：

> 先用 VQ-VAE 为 mesh 面片学习一个离散码本（vocabulary），然后用 decoder-only transformer 自回归预测码本索引序列，生成完整的三角网格。

---

## 方法概览

MeshGPT 采用两阶段训练：

### 第一阶段：学习 Mesh Vocabulary (VQ-VAE)

目标是将三角面片编码为离散 token：

1. **图卷积编码器**：将 mesh 的局部几何和拓扑信息编码为每个面片的连续特征向量。编码器使用 graph convolution 操作，能同时捕捉面片的几何属性（顶点坐标、法线、面积）和拓扑关系（相邻面片的连接方式）
2. **向量量化**：将连续特征量化到一个离散码本中，每个面片映射为一个 token index
3. **解码器**：从量化后的 token 恢复面片的顶点坐标

VQ-VAE 的关键在于图卷积编码器能够编码**局部拓扑信息**，使得码本中的每个 token 不仅代表几何形状，还隐含了面片之间的连接关系。

### 第二阶段：自回归生成 (Decoder-Only Transformer)

在学好码本后，mesh 生成变成一个序列预测问题：

1. 将训练集中的每个 mesh 通过 VQ-VAE 编码为 token 序列
2. 训练 decoder-only transformer 预测 next token index
3. 生成时从起始 token 开始，逐步预测后续 token，直到输出终止符
4. 将 token 序列通过 VQ-VAE 解码器恢复为三角面片坐标

---

## 序列化策略

MeshGPT 对 mesh 的序列化遵循以下规则：

- 每个三角面片由 3 个顶点的 $(x, y, z)$ 坐标表示，共 9 个数值
- 面片按 z-y-x 排序（先按最低 z 坐标排序）
- 顶点坐标被量化到有限精度
- 整个 mesh 被展平为一个一维 token 序列

这种直接序列化的方式简单但序列很长，这也是后续 BPT、MeshAnything V2 等方法重点改进的方向。

---

## 实验结果

在 ShapeNet 上的评估：

| 指标 | MeshGPT | PolyGen | AtlasNet |
|---|---|---|---|
| FID ↓ | **30.6** | 60.8 | 55.2 |
| Shape Coverage ↑ | **75.0%** | 66.1% | 68.3% |

MeshGPT 相比 PolyGen 在 FID 上提升约 30 分，shape coverage 提升约 9%。

### 生成特点

- 生成的 mesh 具有清晰的拓扑结构，面片排布接近手工建模风格
- 面片数量可控（通过序列长度限制）
- 支持类别条件生成

---

## 与后续工作的关系

MeshGPT 开创了 mesh-native AR 生成范式，后续改进主要在序列压缩和扩展性上：

| 方法 | 改进方向 |
|---|---|
| **MeshAnything V2** | Adjacent Mesh Tokenization，序列长度减半 |
| **BPT** | Blocked and Patchified Tokenization，面数上限从数百提升到 8K |
| **FACE** | One-face-one-token，压缩比达 0.11 |
| **Nautilus** | BFS 邻接遍历 + 共享顶点/边编码，5000 面 |
| **MeshRipple** | Frontier-aware BFS tokenization，拓扑连贯性 |
| **EdgeRunner** | 图遍历序列化 |
| **DeepMesh** | RL 微调提升拓扑质量 |
| **QuadGPT** | 扩展到四边形 mesh |

这些方法的共同出发点都是 MeshGPT 确立的 "VQ-VAE 学码本 + AR transformer 生成序列" 的基本范式。

---

## 局限

- **序列长度瓶颈**：直接序列化导致序列很长，限制了生成 mesh 的面数（实际只能生成数百面的简单形状）
- **仅在 ShapeNet 上验证**：ShapeNet 的形状复杂度和多样性有限
- **无条件/类别条件生成为主**：未探索图像或文本条件下的生成
- **拓扑质量依赖码本**：VQ-VAE 的码本容量和图卷积的感受野限制了能建模的拓扑复杂度

---

## 总结

MeshGPT 的核心贡献是确立了 mesh-native 自回归生成的基本范式：graph convolution VQ-VAE 学习面片码本 + decoder-only transformer 自回归生成 token 序列。虽然原始方法受限于序列长度，只能生成简单形状，但它证明了直接生成 mesh 拓扑是可行的，催生了一整个 mesh-native 生成方向的后续研究。
