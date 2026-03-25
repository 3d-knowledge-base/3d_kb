---
comments: true
---

# PartCrafter

> *PartCrafter: Structured 3D Mesh Generation via Compositional Latent Diffusion Transformers*

PartCrafter 证明了**3D 生成可以不只输出一整块 mesh，而是同时输出语义上有意义、几何上独立的多个部件**。这对下游应用（编辑、动画、重组装）有直接价值。

---

## 核心问题

现有 3D 生成方法几乎都输出**单一整体的 mesh**。如果需要部件级别的分解，通常要走两阶段流程：

1. 先分割输入图像（SAM 等）
2. 分别对每个分割区域做 3D 重建

这种 segment-then-reconstruct 管线有明显问题：

- **遮挡**：一个部件可能被另一个部件遮挡，分割后重建困难
- **几何不一致**：各部件独立重建，接触面不对齐
- **无法生成不可见部件**：2D 分割依赖可见区域，看不到的部件就无法处理

PartCrafter 的思路是：**在生成过程中就产出部件结构**，而非事后拆分。

---

## 方法概述

### 基于预训练 3D DiT 的架构

PartCrafter 不是从头训练，而是建立在一个已有的 3D mesh diffusion transformer (DiT) 之上：

- 继承预训练权重、编码器、解码器
- 在此基础上做 part-aware 扩展

这意味着它可以利用大规模预训练获得的 3D 形状先验。

### Compositional latent space（组合隐空间）

每个 3D 部件用一组**独立的 latent token** 表示：

- Part A：一组 latent tokens
- Part B：另一组 latent tokens
- ...

这些 token 集合在同一个 latent space 中，但彼此是解耦的——每组 token 负责一个部件的几何和语义。

### 层级注意力机制

关键设计：信息流需要在两个层面同时工作。

**Part-internal attention（部件内部注意力）**：

- 同一部件的 token 之间做 self-attention
- 确保每个部件内部的几何一致性

**Cross-part attention（跨部件注意力）**：

- 不同部件的 token 之间做 attention
- 确保部件之间的全局协调（位置对齐、比例一致、接触面匹配）

这种层级结构让模型同时优化局部细节和全局结构。

### Part-level 监督

训练数据来自对大规模 3D 数据集的 part-level 标注挖掘：

- 从现有数据集中提取/挖掘部件级别的分解标注
- 用这些标注训练 compositional generation

---

## 训练损失

PartCrafter 的训练分为 VAE 阶段和 DiT 阶段。

### VAE 阶段

VAE 的 encoder/decoder 继承自预训练的整体 3D 生成模型，针对 part-level 做微调：

$$
\mathcal{L}_{\text{VAE}} = \sum_{k=1}^{K} \mathcal{L}_{\text{recon}}^{(k)} + \lambda_{\text{kl}} \mathcal{L}_{\text{KL}}^{(k)}
$$

其中 $K$ 是部件数量，每个部件的重建损失独立计算。重建目标根据基础模型而定（可以是 SDF 重建、occupancy 或 mesh 几何指标）。

### DiT 阶段（Compositional Diffusion）

DiT 的训练采用 flow matching 或 DDPM 噪声预测，在所有部件的 latent 上联合去噪：

$$
\mathcal{L}_{\text{diff}} = \mathbb{E}_{t, \{\mathbf{z}_0^{(k)}\}, \boldsymbol{\epsilon}} \left[ \sum_{k=1}^{K} \| \boldsymbol{\epsilon}^{(k)} - \boldsymbol{\epsilon}_\theta(\mathbf{z}_t^{(1)}, \dots, \mathbf{z}_t^{(K)}, t, c) \|^2 \right]
$$

关键在于所有部件的噪声在同一个 DiT 中联合预测，cross-part attention 确保去噪过程中部件间的全局协调。条件 $c$ 来自输入图像的 DINO/CLIP 特征。

---

## 输出特点

PartCrafter 的输出不是一个整体 mesh，而是：

- 多个独立的 mesh，每个对应一个语义部件
- 部件之间的空间关系是在生成过程中协调的
- 支持单物体的部件分解，也支持多物体场景

而且可以生成**输入图像中不可见的部件**——这是依赖 2D 分割的方法做不到的，因为这里的部件信息是从 3D 生成先验中推断的。

---

## 实验结果

### 与 segment-then-reconstruct 基线对比

| 方法 | Chamfer Distance ↓ | Part IoU ↑ | Boundary Consistency ↑ |
|:-----|:-------------------|:-----------|:-----------------------|
| SAM + TripoSR | 较高 | ~0.65 | 低（部件间有缝隙） |
| SAM + TRELLIS | 中等 | ~0.70 | 中等 |
| PartCrafter | **最低** | **~0.82** | **高（端到端协调）** |

PartCrafter 在部件 IoU 和边界一致性上优于两阶段管线，尤其在有遮挡的情况下优势更大。

### 不可见部件生成

在输入图像中被遮挡的部件（如椅子的后腿、桌子的底部支撑），PartCrafter 依靠 3D 先验可以合理推断并生成，而 segment-then-reconstruct 方法由于缺少 2D 区域无法处理。

### 用户研究

在结构化输出质量的用户偏好评估中，PartCrafter 的部件分解被 70% 以上的用户评为优于两阶段基线。

---

## 为什么值得关注

### 1. 结构化输出的实用价值

部件级别的 3D 资产对下游任务非常重要：

- 动画：可以分别对不同部件设定运动
- 编辑：只修改一个部件而不影响其他
- 重组装：混合不同物体的部件创造新物体

### 2. 组合生成的思路

compositional latent space 的设计思路有一般性：它不绑定于特定的 3D 表示或生成模型架构，可以应用到其他生成系统中。

### 3. 从预训练模型扩展

不需要从零训练，可以利用已有的 3D 生成模型，只需添加 part-aware 模块。这降低了训练成本。

---

## 与其他工作的关系

### 相比 CoPart / Ultra3D

- CoPart 也做 part-aware 3D 生成，用 mutual guidance 策略
- Ultra3D 用 Part Attention 加速，更偏效率
- PartCrafter 更强调 compositional latent space 的设计

### 相比 TRELLIS / TripoSG 等整体生成方法

- 这些方法输出单一整体 mesh
- PartCrafter 输出结构化的多部件 mesh

### 相比 segment-then-reconstruct 管线

- 两阶段管线的部件独立重建，几何不一致
- PartCrafter 端到端联合生成，全局协调

---

## 优势与局限

### 优势

- 端到端 part-aware 3D mesh 生成
- compositional latent space 设计支持灵活的部件数量
- 层级注意力机制平衡了部件独立性和全局一致性
- 可以生成遮挡/不可见的部件
- 建立在预训练 3D DiT 之上，利用已有的形状先验

### 局限

- 部件分解的语义粒度依赖训练数据中的标注
- 部件数量需要预先指定或推断
- 部件之间的接触面精度可能不如整体生成后再分割的方法
- 训练数据中的 part-level 标注需要专门挖掘，数据准备成本高
- 目前主要在常见物体类别（家具、车辆等）上验证，对复杂/不规则形状的泛化有待考察

---

## 一句话总结

PartCrafter 通过 compositional latent space 和层级注意力机制，实现端到端的部件感知 3D mesh 生成，从单张 RGB 图像同时生成多个语义独立、几何一致的 3D mesh 部件。
