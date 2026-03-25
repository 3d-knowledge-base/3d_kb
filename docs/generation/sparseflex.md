---
comments: true
---

# SparseFlex

> *SparseFlex: High-Resolution and Arbitrary-Topology 3D Shape Modeling*

SparseFlex 关注的不是“条件生成的语义对齐”，而是一个更底层、更硬的问题：如何在**高分辨率**下稳定建模复杂 3D shape，同时支持**开放表面、内部结构和 arbitrary topology**。

---

## 核心问题

传统 implicit / SDF 路线有两个突出限制：

- **高分辨率训练很重**：分辨率一高，显存和渲染代价迅速上升
- **拓扑不够自由**：开放表面、内部结构、复杂连接关系往往不好处理

SparseFlex 的思路是：

> 不去放弃 isosurface 这条路线，而是把它做成一个高分辨率、稀疏化、可训练的 3D 表示基础设施。

---

## SparseFlex 是什么

SparseFlex 建立在 FlexiCubes 思想上，但把它从 dense grid 变成了一个**稀疏结构化 isosurface 表示**。

每个活跃 voxel 会携带：

- corner SDF values
- deformation vectors
- interpolation / flexibility weights

区别在于：它只保留**表面附近**重要的 voxel，而不是对整个空间都维护 dense grid。

这带来两个直接收益：

- 计算和显存成本下降
- 更适合表达开放表面和复杂边界

---

## 主要创新：frustum-aware sectional voxel training

SparseFlex 值得记住的不是“它是 sparse”，而是它提出了一个有效的训练策略：

### Frustum-aware sectional voxel training

训练时，每次只激活当前视图 camera frustum 相关的那一部分 voxel，而不是整块 3D 结构全部参与渲染与反传。

这意味着：

- 不再为不可见区域白白付出训练成本
- 高分辨率训练变得可行
- 视角甚至可以进入物体内部，从而监督内部结构

这点重要，因为它让 **rendering-supervised 3D shape modeling** 向高分辨率扩展。

---

## VAE / Backbone 结构

SparseFlex 不以多视图图像为主输入，而是更偏几何优先的点云输入路线。

### 编码器

- 先把点云转成 sparse voxel structure
- 聚合局部点特征
- 用 sparse transformer 编码为 latent representation

### 解码器

- 逐步上采样 sparse structure
- 每一级预测 occupancy 并自剪枝 (`self-pruning upsampling`)
- 最终解码出 SparseFlex 表示并做可微 surface extraction

这里的关键在于：上采样不是盲目补全空间，而是伴随 occupancy 筛选进行的，因此对开放表面更友好。

---

## 损失函数

SparseFlex 的训练涉及 VAE 和下游生成模型两部分。

### VAE 训练损失

$$
\mathcal{L}_{VAE} = \mathcal{L}_{recon} + \lambda_{KL}\mathcal{L}_{KL} + \lambda_{render}\mathcal{L}_{render}
$$

- **几何重建损失** $\mathcal{L}_{recon}$：
    - SDF 回归损失：预测的 corner SDF 与 GT SDF 之间的 L1/L2 损失
    - 变形场正则化：deformation vectors 的 L2 范数惩罚，防止过度变形
    - occupancy 分类损失：self-pruning upsampling 中的 binary cross-entropy

- **KL 正则化** $\mathcal{L}_{KL}$：latent space 的标准 KL 散度正则，保证 latent 分布可采样

- **可微渲染损失** $\mathcal{L}_{render}$（frustum-aware 训练的核心）：
    - 对当前视角 frustum 内的活跃 voxel 做可微 surface extraction（基于 FlexiCubes）
    - 渲染后与 GT 多视角图像对比
    - 包含 mask loss（轮廓）和 depth loss（深度图）

### 生成模型训练

在 VAE latent space 上训练扩散模型（如 DiT），使用标准的去噪损失：

$$
\mathcal{L}_{diff} = \mathbb{E}_{t, \epsilon}\left[\| \epsilon - \epsilon_\theta(\mathbf{z}_t, t) \|^2\right]
$$

---

## 架构参数

### Sparse Transformer

- 编码器/解码器均基于 sparse 3D transformer
- 每层包含 sparse 3D self-attention 和 FFN
- 上采样使用 sparse transposed convolution + self-pruning（根据 occupancy 预测裁剪非活跃 voxel）
- 最终输出分辨率可达 $512^3$（部分实验到 $1024^3$），但实际活跃 voxel 数远低于 dense 对应（通常 <1%）

---

## 实验结果

### 主要对比

论文报告了与 FlexiCubes（dense 版本）、DMTet、SDF-based 方法的对比：

| 指标 | SparseFlex 表现 |
|---|---|
| Chamfer Distance | 较 dense FlexiCubes 降低约 82%（论文原文） |
| F-Score | 较 dense FlexiCubes 提升约 88%（论文原文） |
| 训练显存 | 远低于 dense grid 方法（稀疏存储） |
| 拓扑支持 | 支持开放表面和内部结构 |

### 训练效率

- frustum-aware sectional voxel training 使得高分辨率（$512^3$+）训练成为可能
- 相比 dense grid 方法，在同等分辨率下显存降低一个数量级以上

### 重建质量

- 在 Objaverse 数据集上，SparseFlex VAE 的重建保真度优于同分辨率的 dense 方法
- 对开放表面（如碗、杯子内壁）和复杂拓扑（如链条、网格结构）的建模质量明确优于纯 SDF 方法

---

## 它为什么重要

SparseFlex 在表示发展线里的独特价值是：

- 它不是纯 latent set
- 不是纯 structured sparse latent like SLAT
- 也不是 mesh-native token

它更像是：

> 给高分辨率、复杂拓扑 3D shape 建模提供一套能训得起来的 sparse isosurface 基础设施。

这一点对很多后续工作都重要，因为很多系统最终仍然需要高质量表面恢复，而不是只停留在 latent field 里。

---

## 与其他路线的关系

### 相比 TRELLIS / SLAT

- TRELLIS 更偏统一生成框架和结构化 latent
- SparseFlex 更偏高分辨率 shape modeling 基础表示

### 相比 TRELLIS 2 / O-Voxel

- O-Voxel 更强调 native structured 3D asset representation
- SparseFlex 仍然属于 isosurface / field 世界，但把这条路做得更高分辨率、更稀疏、更可训练

### 相比 FACE / BPT

- SparseFlex 不是 mesh-native tokenization
- 它仍然属于从内部表示恢复表面的路线

---

## 优势与局限

### 优势

- 高分辨率训练更可行
- 支持开放表面与内部结构
- 稀疏结构降低成本
- 对 arbitrary topology 更友好

### 局限

- 仍然依赖 isosurface extraction 思维
- 体系偏工程和基础设施，直接 end-to-end 生成叙事不如 TRELLIS / TripoSG 清晰
- 对原生 mesh token 化问题没有直接回答

---

## 一句话总结

SparseFlex 的主要价值，是把可微 isosurface 建模从 dense、高成本、拓扑受限的状态，推进成一种稀疏化、高分辨率、支持开放拓扑与内部结构的可训练 3D 表示基础设施。
