---
comments: true
---

# QuadGPT

> *QuadGPT: Native Quadrilateral Mesh Generation with Autoregressive Models (ICLR 2026)*

QuadGPT 的意义不在于生成质量的微调提升，而在于它回答了一个此前没人正面回答的问题：**能否用自回归模型直接生成四边形主导的 mesh？**

---

## 核心问题

在 3D 建模实践中，四边形 mesh（quad mesh）比三角 mesh 更受美术师和动画师偏好，原因包括：

- 更好的边循环（edge loop）控制，适合变形和动画
- 更干净的拓扑结构，subdivision 效果更好
- 行业标准工具（Maya、Blender 等）原生支持 quad 工作流

然而，现有 mesh 生成模型（MeshGPT、Nautilus、BPT、FACE 等）清一色生成三角 mesh。要得到 quad mesh，通常的做法是：

1. 先生成三角 mesh
2. 用规则方法（如三角形合并、instant meshes 等）后处理转换为 quad

这个两阶段流程的问题是：转换后的 quad 拓扑质量不可控，经常出现奇异点过多、边循环不连续等问题。

---

## 方法概述

QuadGPT 将 quad mesh 生成建模为**序列预测**问题，有两个关键设计。

### 统一的混合拓扑 tokenization

现实中的"四边形 mesh"往往不是纯 quad，而是混合了少量三角面的 quad-dominant mesh。QuadGPT 设计了一套统一的 tokenization：

- 三角面编码为 3 个顶点坐标
- 四边面编码为 4 个顶点坐标
- 用特殊 token 区分面类型
- 统一进入同一个序列，由 AR 模型处理

这避免了为 tri 和 quad 分别建两套系统的复杂性。

### tDPO：面向拓扑质量的 RL 微调

QuadGPT 的第二个创新是引入了强化学习微调：

- 基于 DPO（Direct Preference Optimization）框架
- 设计了面向**拓扑质量**的奖励函数（topology-aware reward），评估生成 mesh 的边循环质量、奇异点数量等拓扑指标
- 用 RL 微调后的模型在拓扑质量上有明确提升

这是一个值得注意的趋势：将 RL 引入 3D mesh 生成来优化难以用可微损失捕捉的结构性指标。

---

## 为什么值得关注

### 1. 填补了 quad mesh 生成的空白

在 QuadGPT 之前，没有方法能 end-to-end 地生成 quad mesh。所有现有路线都需要三角 mesh 中转。这是一个实际需求很强但研究上被忽视的问题。

### 2. RL 用于 mesh 拓扑优化

tDPO 的思路有一般性价值。mesh 的拓扑质量（边循环连续性、奇异点分布、面形状均匀性等）很难用传统可微损失表达，但可以用后验指标评估。RL 正好适合这类"容易评估、难以微分"的目标。

### 3. 对接实际生产需求

quad mesh 是游戏、影视、工业设计的标准资产格式。一个能直接生成 quad mesh 的模型在应用层面比纯三角 mesh 生成更有价值。

---

## 与其他工作的关系

### 相比 MeshGPT / Nautilus / FACE

- 这些方法只能生成三角 mesh
- QuadGPT 是唯一支持原生 quad 生成的 AR 方法

### 相比 Instant Meshes 等 remeshing 工具

- Instant Meshes 是确定性后处理方法，不是生成模型
- QuadGPT 是学习到的生成分布，能产出多样化的 quad mesh

### 相比 TSSR（离散扩散）

- TSSR 用离散扩散而非 AR，但也只处理三角 mesh
- QuadGPT 在拓扑类型上做了扩展（tri + quad 混合）

---

## 优势与局限

### 优势

- 首个 end-to-end quad mesh 自回归生成方法
- 统一 tri/quad 混合拓扑 tokenization 设计简洁
- RL 微调方案（tDPO）有效提升拓扑质量
- ICLR 2026 接收

### 局限

- 面数规模受限于 AR 序列长度（quad 面需要 4 个顶点，比三角面更长）
- RL 微调增加训练复杂性和成本
- 生成速度仍受 AR 逐 token 解码限制
- 对复杂拓扑（高 genus、非流形）的处理能力尚不清楚

---

## 一句话总结

QuadGPT 首次实现了四边形 mesh 的端到端自回归生成，通过统一的 tri/quad 混合 tokenization 和面向拓扑质量的 RL 微调（tDPO），将 mesh 生成从纯三角形扩展到了实际生产更需要的 quad-dominant 格式。
