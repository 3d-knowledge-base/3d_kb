# Mesh Extraction: 从隐式场到三角网格

从隐式场（SDF / 标量场）中提取显式三角网格表面，是 3D 生成 pipeline 的关键一步。本文梳理三种主流方案——经典 Marching Cubes、可微的 DMTet、以及完全可微的 FlexiCubes——的原理与发展关系。

---

## 核心问题

空间被划分为体素网格，每个顶点存储一个标量值（如 SDF 值）。

**问题**：如何从这些离散数值中提取出连续、光滑的三角网格表面（Mesh）？

---

## Marching Cubes (MC)

> *Marching Cubes: A High Resolution 3D Surface Construction Algorithm* (Lorensen & Cline, 1987)

MC 是"元老级"表面提取算法，核心思想是分治法——遍历每个由 8 个相邻体素顶点组成的逻辑立方体。[→ 详情页](marching-cubes.md)

### 算法步骤

**1. 分类 (Classification)**

检查 8 个顶点：值 > 阈值标为"内部"(1)，否则为"外部"(0)。形成 8 位二进制码，共 $2^8 = 256$ 种状态。

**2. 查表 (Lookup Table)**

利用旋转/互补对称性，256 种情况简化为 **15 种基本拓扑**。用索引查预计算表，确定该立方体内的三角形切割方式。

**3. 插值 (Interpolation)**

表面切过的边上，用**线性插值**确定精确切点：

$$
P = P_1 + \frac{Iso - V_1}{V_2 - V_1}(P_2 - P_1)
$$

**4. 法线计算**

通过标量场的梯度估算表面法线，用于光照渲染。

### MC 的局限性

| 问题 | 描述 |
|:-----|:-----|
| **特征丢失** | 顶点只能在网格边上——无法表达锐利边缘/角点，尖角被削成倒角 |
| **Sliver Triangles** | 表面接近顶点时产生极细长三角形，对渲染/模拟不友好 |
| **不可微** | 拓扑变化（查表）是离散的；SDF 微小变化导致符号翻转时网格突变——**无法梯度下降优化** |

!!! warning "AI 时代的最大痛点"
    不可微性使得 MC 无法直接嵌入深度学习的端到端训练管线。

---

## 从 Primal 到 Dual

为解决 MC 的特征丢失问题，图形学界引入了**对偶 (Dual)** 方法：

| 方法 | 顶点位置 | 特点 |
|:-----|:---------|:-----|
| MC (Primal) | 网格的**边**上 | 简单可靠，但无法表达尖角 |
| DC / DMC (Dual) | 网格的**体素内部** | 可以更好地贴合形状 |

**Dual Marching Cubes (DMC)** 是 FlexiCubes 的直接前身——允许在立方体内部放置顶点并连接，但顶点位置通常固定（中心或 QEF 求解），优化中不够稳定。

---

## FlexiCubes

> *Flexible Isosurface Extraction for Gradient-Based Mesh Optimization* (Shen et al., 2023)

FlexiCubes 是对 DMC 的**完全参数化、可微化改造**，专为深度学习梯度优化而设计。[→ 详情页](flexicubes.md)

### 设计思路

> 仅优化 SDF 不够——网格提取过程本身也需要可学习参数。

### 三组可学习参数

**Flexibility 1: 权重 $\alpha \in \mathbb{R}^8$, $\beta \in \mathbb{R}^{12}$ — 控制顶点位置**

DMC 中对偶顶点通常固定在体素中心。FlexiCubes 通过 $\alpha, \beta$ 加权组合，让**对偶顶点在体素内部自由移动**。

- 可精确捕捉锐利特征
- 避开导致劣质三角形的位置
- 完全可微：$\frac{\partial \text{VertexPos}}{\partial \text{Weights}}$ 连续

**Flexibility 2: 网格变形 $\delta$ — 调整底层网格**

允许体素网格顶点本身发生微小位移，让网格"迎合"物体表面。

**Flexibility 3: 分割权重 $\gamma$ — 控制拓扑连接**

DMC 生成四边形需剖分为两个三角形。$\gamma$ 让网络学习最优剖分方式（连接对角线 A-C 还是 B-D），最小化 Loss。

### 工作流程

```
神经网络输出 SDF + (α, β, γ, δ)
      ↓
DMC 规则确定拓扑连接（保证水密性和流形性）
      ↓
α, β 参数计算每个顶点精确位置
      ↓
渲染 → 计算 Loss
      ↓
梯度同时回传到 SDF 和 FlexiCubes 参数
```

### FlexiCubes 解决了什么？

- **高质量网格**：三角形分布均匀，能表现锐利边缘
- **梯度平滑性**：拓扑变化时顶点位置弹性调整，优化不震荡
- **端到端可微**：可直接嵌入微分渲染管线

---

## DMTet (Deep Marching Tetrahedra)

> *Deep Marching Tetrahedra: a Hybrid Representation for High-Resolution 3D Shape Synthesis* (Shen et al., 2021)

DMTet 使用**四面体网格**代替立方体网格，结合**神经网络预测** SDF 值和顶点位移，实现部分可微的网格提取。[→ 详情页](dmtet.md)

### 核心优势

- **四面体无拓扑歧义**：4 个顶点仅 3 种基本情况，比 MC 的 15 种更简洁
- **自适应分辨率**：可在表面附近局部加密四面体，远离表面保持稀疏
- **可变形网格**：神经网络预测顶点位移，网格能主动贴合物体表面

### 应用

Fantasia3D 和 Magic3D 均使用 DMTet 作为几何表征，通过 SDS Loss 优化 SDF 和顶点位移，直接输出高质量 Mesh。

---

## 总结对比

| 特性 | Marching Cubes (1987) | DMTet (2021) | FlexiCubes (2023) |
|:-----|:----------------------|:-------------|:------------------|
| 网格类型 | 立方体 | **四面体** | 立方体（对偶） |
| 核心逻辑 | Primal（顶点在边上） | Primal（边上插值） | Dual（顶点在体素内） |
| 可微性 | 不可微 | **部分可微** | **完全可微** |
| 自适应分辨率 | 不支持 | **支持** | 不支持 |
| 可学习参数 | 无 | SDF + 位移 | SDF + α/β/γ/δ |
| 网格质量 | 一般 | 较好 | **最好** |
| 主要应用 | 医学成像、离线重建 | Fantasia3D, Magic3D | TRELLIS, SparseFlex |

**发展路径**：MC（不可微）→ DMTet（部分可微 + 自适应）→ FlexiCubes（完全可微 + 高质量）

!!! quote "一句话概括"
    **Marching Cubes** 是机械式翻译（死规则）；**DMTet** 是半智能翻译（可变形但不够灵活）；**FlexiCubes** 是全智能翻译官（所有决策都可学习优化）。

---

## 在 3D 生成中的应用

### TRELLIS

SLAT → Mesh 解码路径使用 FlexiCubes：Transformer 输出 SDF 场 + FlexiCubes 参数，经上采样（$64^3 \to 256^3$）后通过 FlexiCubes 提取最终 Mesh，整个过程端到端可微，通过可微分渲染训练。

### SparseFlex

SparseFlex 在 FlexiCubes 基础上引入**稀疏体素结构**——仅在物体表面附近的体素上执行 FlexiCubes 提取，跳过空白区域。这结合了 DMTet 的"自适应"思想与 FlexiCubes 的"完全可微"优势，在保持高质量的同时减少计算量和内存消耗，支持更高分辨率的网格提取。
