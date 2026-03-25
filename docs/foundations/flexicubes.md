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

## 损失函数与训练

### Shape Reconstruction 任务

在 shape reconstruction 评估中，随机采样 1000 个点，比较 ground truth mesh 和提取 mesh 的 SDF 值差异进行优化。

### 等边三角形正则化

为控制输出网格质量，FlexiCubes 可选用边长正则化：

$$
R_{edge} = \frac{1}{|T|} \sum_{t \in T} \sum_{e_i \in t} (|e_i| - \bar{e})^2, \quad \bar{e} = \frac{1}{|E|}\sum_{e_i \in E} |e_i|
$$

训练策略：先做 1000 步纯重建优化，再加 300 步重建 + 正则化（正则化权重从 0 线性增到 100）。

---

## 实验结果

### Shape Reconstruction 消融（Tab 3, $64^3$ 网格）

| 配置 | IN>5° ↓ | CD ($10^{-5}$) ↓ | F1 ↑ | ECD ($10^{-2}$) ↓ |
|:-----|:--------|:-----------------|:-----|:-----------------|
| DMC centroid | 53.02 | 5.85 | 0.65 | 2.60 |
| + flex vertex | 40.88 | 5.34 | 0.68 | 0.99 |
| + grid deform | 39.46 | 5.01 | 0.69 | 0.98 |
| **+ flex quad split = FlexiCubes** | **34.87** | **4.87** | **0.70** | **0.71** |

每一组参数都带来了明确的质量提升。

### 网格质量对比（Tab 4, $64^3$ + 正则化）

| 方法 | IN>5° ↓ | CD ($10^{-5}$) ↓ | Aspect>4 ↓ | Radius>4 ↓ | Angle<10° ↓ |
|:-----|:--------|:-----------------|:-----------|:-----------|:------------|
| MC | 52.37 | 6.33 | 11.71% | 11.71% | 11.84% |
| DMTet(80) | 48.66 | 5.17 | 17.31% | 16.68% | 17.83% |
| **FlexiCubes** | **34.87** | **4.87** | **2.93%** | **4.49%** | **2.04%** |

FlexiCubes 的退化三角形比例远低于 MC 和 DMTet。

### Inverse Rendering: nvdiffrec（Tab 5, NeRF Synthetic）

FlexiCubes 与 DMTet 在 PSNR 上相当（约 30-33 dB），但在 Chamfer Distance 上 FlexiCubes 多数场景更优（如 Chair: 0.45 vs 4.51 $\times 10^{-2}$，Ship: 10.5 vs 55.8 $\times 10^{-2}$）。

### 3D 生成: GET3D（Tab 6, ShapeNet FID）

| 方法 | Motorbike ↓ | Chair ↓ | Car ↓ |
|:-----|:-----------|:--------|:------|
| DMTet | 48.90 | 22.41 | 10.60 |
| **FlexiCubes** | **44.87** | **17.51** | **9.55** |

仅修改 GET3D 最后一层以输出 FlexiCubes 参数（21 weights/cube），即可获得 FID 改善。

### 计算开销（Tab 7）

| 方法 | Forward (ms) | Backward (ms) | Memory (MB) | 分辨率 |
|:-----|:------------|:-------------|:------------|:-------|
| MC | 2.28 | 0.43 | 12.05 | $64^3$ |
| DMTet | 2.33 | 1.38 | 22.44 | $64^3$ |
| **FlexiCubes** | **8.93** | **7.32** | **116.56** | $64^3$ |

FlexiCubes 的灵活性以额外计算和内存为代价（约 4-5× forward、5× backward、5× memory vs DMTet），但在应用级 pipeline（如 nvdiffrec）中总开销增幅可控（307 vs 315 ms/iter）。

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

FlexiCubes 通过三组可学习参数（$\alpha/\beta$ 控制顶点位置、$\delta$ 变形网格、$\gamma$ 优化拓扑），将"死板的等值面提取"变成"可微分的几何优化"，是当前 3D 生成 pipeline 的标配组件。
