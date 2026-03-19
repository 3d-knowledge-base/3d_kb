---
comments: true
---

# FlexiCubes

> *Flexible Isosurface Extraction for Gradient-Based Mesh Optimization* (Shen et al., NeurIPS 2023)

FlexiCubes 是对 Dual Marching Cubes (DMC) 的**完全参数化、可微化改造**，专为深度学习梯度优化而设计。它引入三组可学习参数，使网格提取过程本身也能参与端到端的反向传播优化，是当前 3D 生成领域（TRELLIS、SparseFlex 等）广泛使用的 mesh 提取方案。

---

## 设计思路

> 仅优化 SDF 不够——网格提取过程本身也需要可学习参数。

传统 Marching Cubes 是"死规则"：给定 SDF 值，输出固定。即使 SDF 预测完美，MC 的离散查表和边上插值仍会导致信息损失。FlexiCubes 的思路是让提取过程也成为可优化的"软"操作。

---

## 从 Primal 到 Dual：方法论基础

FlexiCubes 基于**对偶 (Dual)** 方法——与 MC 的关键区别在于顶点放置策略：

| 方法类型 | 顶点位置 | 特点 |
|:---------|:---------|:-----|
| **Primal** (MC) | 网格的**边**上 | 简单可靠，但无法表达尖角 |
| **Dual** (DMC/FlexiCubes) | 网格的**体素内部** | 可更好地贴合形状，表达锐利特征 |

Dual Marching Cubes (DMC) 是 FlexiCubes 的直接前身——在每个体素内部放置顶点并连接相邻体素的顶点形成面。但 DMC 的顶点位置通常固定（体素中心或 QEF 求解），不够灵活，优化中不稳定。

FlexiCubes 的贡献是**让 DMC 的所有关键决策都变成可学习参数**。

---

## 三组可学习参数

### Flexibility 1: 权重 $\alpha \in \mathbb{R}^8$, $\beta \in \mathbb{R}^{12}$ — 控制顶点位置

DMC 中对偶顶点通常固定在体素中心。FlexiCubes 通过 $\alpha$（8 个顶点权重）和 $\beta$（12 条边权重）的加权组合，让**对偶顶点在体素内部自由移动**：

- 可精确捕捉锐利特征（尖角、棱线）
- 可避开导致劣质三角形的位置
- 完全可微：$\frac{\partial \text{VertexPos}}{\partial \alpha, \beta}$ 连续

### Flexibility 2: 网格变形 $\delta$ — 调整底层网格

允许体素网格的顶点本身发生**微小位移**，让网格主动"迎合"物体表面。

效果：当表面恰好落在两个网格点之间不利位置时，$\delta$ 可以微调网格使表面穿过更有利的位置，提升提取质量。

### Flexibility 3: 分割权重 $\gamma$ — 控制拓扑连接

DMC 生成的四边形面片需要剖分为两个三角形。一个四边形 ABCD 有两种剖分方式（对角线 A-C 或 B-D）。$\gamma$ 让网络**学习每个四边形的最优剖分方式**：

- 最小化渲染 Loss
- 避免产生退化三角形
- 拓扑连接也成为可优化对象

---

## 工作流程

```text
神经网络输出 SDF + (α, β, γ, δ)
      ↓
DMC 规则确定拓扑连接（保证水密性和流形性）
      ↓
α, β 参数计算每个对偶顶点的精确位置
      ↓
γ 决定四边形的最优三角剖分
      ↓
渲染 → 计算 Loss
      ↓
梯度同时回传到 SDF 和 FlexiCubes 参数 (α, β, γ, δ)
```

关键：**拓扑结构由 SDF 符号决定**（保证网格合法），**几何细节由可学习参数决定**（保证可微优化）。

---

## 子体素精度：SLAT 中的 FlexiCubes

在 TRELLIS 的 SLAT（Structured Latent）框架中，FlexiCubes 展现出强大的**子体素精度**能力：

- **顶点变形 $\delta$** 让锐利边缘突破体素网格限制——表面细节可以精确到比体素尺寸更细的尺度
- **拓扑优化 $\gamma$, $\alpha$, $\beta$** 自适应调整连接方式，在平坦区域使用大三角形，在细节区域使用小三角形
- SLAT 的特征向量如同**"压缩细节包"**——Transformer 在低分辨率预测，FlexiCubes 在高分辨率展开细节

TRELLIS 的具体路径：Transformer 输出 $64^3$ SDF + FlexiCubes 参数，经上采样至 $256^3$ 后通过 FlexiCubes 提取最终 Mesh。

---

## 拓扑局限："拓扑诅咒"

FlexiCubes 虽然强大，但存在固有的**拓扑局限**：

!!! warning "拓扑诅咒 (Topology Curse)"
    FlexiCubes 的拓扑结构由底层网格分辨率决定——**无法学习低多边形 (Low-poly) 拓扑**。输出面片数直接由网格密度控制：

    - $64^3$ 网格 → ~300K 面
    - $256^3$ 网格 → ~1M+ 面
    
    这意味着 FlexiCubes 总是生成高密度 Mesh，需要后续的**重拓扑 (Retopology)** 流程才能用于游戏/动画等对拓扑有要求的场景。

### 两阶段工作流

实践中通常采用：

1. **FlexiCubes 提取**：高密度、高质量几何
2. **Retopology**：通过 Quadriflow/Instant Meshes 等工具简化拓扑，或使用 MeshAnything 等 AI 方法生成 artist-friendly 拓扑

---

## 与 Marching Cubes 对比

| 特性 | Marching Cubes (1987) | FlexiCubes (2023) |
|:-----|:----------------------|:------------------|
| 核心逻辑 | Primal（顶点在边上） | Dual（顶点在体素内） |
| 顶点位置 | 固定（线性插值） | **灵活**（可学习权重 α, β） |
| 底层网格 | 固定 | **可变形**（δ 参数） |
| 三角剖分 | 查表确定 | **可学习**（γ 参数） |
| 锐利特征 | 无法表达（倒角） | **可以表达** |
| 网格质量 | 常有 sliver triangles | **高质量**，可优化 |
| 可微性 | 不可微 | **完全可微** |
| 主要应用 | 医学成像、离线重建 | 生成式 AI、逆向渲染 |

---

## 一句话总结

FlexiCubes 是有智慧的翻译官——通过三组可学习参数（$\alpha/\beta$ 控制顶点位置、$\delta$ 变形网格、$\gamma$ 优化拓扑），将"死板的等值面提取"变成"可微分的几何优化"，是当前 3D 生成 pipeline 的标配组件。
