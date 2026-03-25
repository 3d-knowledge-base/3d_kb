---
comments: true
---

# DMTet (Deep Marching Tetrahedra)

> *Deep Marching Tetrahedra: a Hybrid Representation for High-Resolution 3D Shape Synthesis* (Shen et al., NeurIPS 2021)

DMTet 是一种混合 3D 表征方法，将**可变形四面体网格**与**神经网络预测**相结合，实现高分辨率、可微分的网格提取。它是 FlexiCubes 的重要前身，在 Fantasia3D、Magic3D 等 Text-to-3D 方法中被广泛采用。

---

## 核心思想

传统 Marching Cubes 使用规则的立方体网格，DMTet 的主要设计特点是：

1. 使用**四面体 (Tetrahedra)** 网格代替立方体网格
2. 用**神经网络**预测四面体顶点上的 SDF 值和位移
3. 支持**自适应分辨率**——在几何细节丰富处加密，平坦处稀疏

---

## 四面体 vs 立方体

| 特性 | 立方体网格 (MC) | 四面体网格 (DMTet) |
|:-----|:----------------|:-------------------|
| 基本单元 | 8 顶点立方体 | 4 顶点四面体 |
| 拓扑歧义 | 存在（256 → 15 种，部分有歧义） | **无歧义**（仅 3 种基本情况） |
| 分辨率 | 均匀网格 | **可自适应**（局部加密） |
| 变形灵活性 | 网格固定 | 顶点可位移 |
| 实现复杂度 | 简单 | 中等 |

四面体的主要优势是**拓扑无歧义**：一个四面体只有 4 个顶点，$2^4 = 16$ 种内外组合，归约后仅剩 3 种基本情况（无面片、一个三角形、两个三角形），不存在 MC 中多种等价切割方式的歧义问题。

---

## 架构与流程

### 输入

初始化一个覆盖目标区域的**变形四面体网格** $\mathcal{T} = \{V, T\}$：

- $V$：顶点集合，每个顶点有 3D 坐标
- $T$：四面体连接关系

### 神经网络预测

对每个四面体顶点 $v_i$，神经网络预测两个量：

$$
\text{MLP}(v_i) \rightarrow (s_i, \Delta v_i)
$$

- $s_i \in \mathbb{R}$：该顶点的 **SDF 值**（正/负决定内外）
- $\Delta v_i \in \mathbb{R}^3$：该顶点的**位移向量**（允许网格变形）

### Marching Tetrahedra 提取

基于预测的 SDF 值，在每个四面体内执行类似 MC 的提取：

1. 根据 4 个顶点的 SDF 符号确定拓扑（仅 3 种情况）
2. 在符号变化的边上插值确定表面切点
3. 连接切点形成三角形面片

### 可微性

整个过程可微：

- SDF 预测 → 可微（神经网络）
- 顶点位移 → 可微（加法操作）
- 边上插值 → 可微（线性插值关于 SDF 值连续）

!!! note "可微性的关键"
    与 MC 不同，DMTet 的可微性主要来自：(1) 四面体内拓扑更简单，歧义更少；(2) 顶点位移提供了额外的连续自由度。但严格来说，当 SDF 符号翻转导致拓扑变化时，梯度仍然不完全连续——这正是后续 FlexiCubes 进一步解决的问题。

---

## 训练损失

DMTet 的训练目标根据具体应用场景有所不同。以 shape reconstruction（给定目标 3D shape 重建其 mesh）为例：

### 几何重建损失

$$
\mathcal{L}_{\text{recon}} = \mathcal{L}_{\text{surf}} + \lambda_{\text{def}} \mathcal{L}_{\text{def}} + \lambda_{\text{reg}} \mathcal{L}_{\text{reg}}
$$

**表面损失** $\mathcal{L}_{\text{surf}}$：提取 mesh 表面与 GT 表面之间的 Chamfer Distance 或 point-to-surface 距离。

$$
\mathcal{L}_{\text{surf}} = \text{CD}(\mathcal{S}_{\text{pred}}, \mathcal{S}_{\text{gt}})
$$

**变形正则化** $\mathcal{L}_{\text{def}}$：约束顶点位移不要过大，防止四面体退化（翻转）：

$$
\mathcal{L}_{\text{def}} = \frac{1}{|V|} \sum_{i} \| \Delta v_i \|^2
$$

**拉普拉斯平滑正则** $\mathcal{L}_{\text{reg}}$：鼓励提取 mesh 表面的平滑性，减少噪声：

$$
\mathcal{L}_{\text{reg}} = \frac{1}{|V_s|} \sum_{v \in V_s} \| v - \frac{1}{|\mathcal{N}(v)|}\sum_{u \in \mathcal{N}(v)} u \|^2
$$

其中 $V_s$ 是提取的表面顶点，$\mathcal{N}(v)$ 是顶点 $v$ 的邻居。

### 在生成任务中的损失

当 DMTet 用于 Text-to-3D（如 Fantasia3D）时，损失变为 SDS loss：

$$
\nabla_\theta \mathcal{L}_{\text{SDS}} = \mathbb{E}_{t, \boldsymbol{\epsilon}} \left[ w(t)(\boldsymbol{\epsilon}_\phi(\mathbf{x}_t; y, t) - \boldsymbol{\epsilon}) \frac{\partial \mathbf{x}}{\partial \theta} \right]
$$

其中 $\mathbf{x}$ 是 DMTet 提取 mesh 的渲染图像，$\theta$ 包含 MLP 预测的 SDF 值和顶点位移。DMTet 的可微性使得 SDS 梯度可以回传到四面体网格参数。

---

## 自适应分辨率

DMTet 的另一优势是支持**非均匀网格密度**：

```text
初始粗四面体网格
      ↓
训练若干轮 → 预测 SDF
      ↓
识别表面附近的四面体（SDF 值接近 0）
      ↓
局部细分（Subdivision）→ 在表面附近加密
      ↓
继续训练 → 更精细的几何
```

这意味着：

- **表面附近**：高密度四面体，捕捉细节
- **远离表面**：粗四面体，节省计算
- 支持比均匀网格**更高的有效分辨率**，同等内存下细节更丰富

---

## 实验结果

### Shape Reconstruction

在 ShapeNet 上的 3D shape reconstruction 任务中，DMTet 与其他方法的对比：

| 方法 | Chamfer Distance ↓ | F-Score (%) ↑ | 分辨率 |
|:-----|:-------------------|:--------------|:------|
| Marching Cubes (dense) | 0.87 | 85.3 | $128^3$ |
| Neural MC | 0.72 | 89.1 | $128^3$ |
| DMTet (uniform) | 0.58 | 92.7 | $128^3$ 等效 |
| DMTet (adaptive) | **0.43** | **95.2** | $256^3$ 等效 |

自适应分辨率版本在同等计算预算下达到了更高有效分辨率，F-Score 提升明显。

### Mesh 质量对比

| 指标 | MC $128^3$ | DMTet (adaptive) |
|:-----|:-----------|:-----------------|
| 面片数 | ~300K | ~180K（自适应更少） |
| 自交（self-intersection）比例 | 低 | 低 |
| 非流形边数 | 0 | 0 |
| 拓扑歧义 | 有 | 无 |

### 下游任务：GET3D

在 GET3D (NVIDIA, NeurIPS 2022) 中，DMTet 作为可微 mesh 提取器，使得 GAN 可以端到端生成 mesh：

| 类别 | FID ↓ (DMTet) | FID ↓ (without DMTet) |
|:-----|:-------------|:---------------------|
| Car | 29.3 | 45.6 |
| Chair | 36.1 | 52.8 |
| Motorbike | 32.7 | 48.4 |

DMTet 的可微性对 GAN 训练的稳定性和生成质量有明显帮助。

---

## 在 3D 生成中的应用

### Fantasia3D (2023)

DMTet 在 SDS 驱动的 Text-to-3D 中的典型应用：

- 用 DMTet 作为几何表征，通过 SDS Loss 优化 SDF 值和顶点位移
- 几何与外观解耦：先优化几何（DMTet），再优化纹理（PBR 材质）
- 相比 NeRF 表征，DMTet 直接输出高质量 Mesh

### Magic3D (2023)

NVIDIA 的两阶段 Text-to-3D 方法，第二阶段使用 DMTet：

- **第一阶段**：低分辨率 NeRF + SDS 粗优化
- **第二阶段**：将 NeRF 转换为 DMTet 表征，高分辨率精细优化
- 利用 DMTet 的自适应分辨率在细节处加密

### GET3D (2022)

NVIDIA 的 3D 生成模型，直接在 DMTet 上构建生成器：

- GAN 框架：生成器输出 DMTet 参数（SDF + 位移 + 纹理）
- 判别器在 2D 渲染上判别
- 利用 DMTet 的可微性实现端到端对抗训练

---

## 与 MC、FlexiCubes 的定位对比

| 特性 | MC | DMTet | FlexiCubes |
|:-----|:---|:------|:-----------|
| 网格类型 | 立方体 | **四面体** | 立方体（对偶） |
| 可微性 | 不可微 | **部分可微** | **完全可微** |
| 自适应分辨率 | 不支持 | **支持** | 不支持（均匀网格） |
| 可学习参数 | 无 | SDF + 位移 | SDF + α/β/γ/δ |
| 拓扑歧义 | 有 | **无** | 无 |
| 网格质量 | 一般 | 较好 | **最好** |
| 梯度平滑性 | 无 | 一般 | **最好** |
| 时间线 | 1987 | 2021 | 2023 |

**发展路径**：MC（不可微）→ DMTet（部分可微 + 自适应）→ FlexiCubes（完全可微 + 高质量）

---

## 局限性

| 问题 | 描述 |
|:-----|:-----|
| **梯度不完全连续** | 拓扑变化处仍有梯度不连续问题，虽然比 MC 好但不如 FlexiCubes |
| **四面体网格构建复杂** | 相比规则立方体网格，四面体网格的初始化和细分操作更复杂 |
| **已被 FlexiCubes 部分替代** | 在纯可微优化场景中，FlexiCubes 通常是更好的选择 |
| **自适应分辨率需额外设计** | 细分策略的设计和实现增加了工程复杂度 |

---

## 一句话总结

DMTet 是连接"不可微 MC"与"完全可微 FlexiCubes"之间的关键桥梁——通过四面体网格和神经网络预测实现了部分可微和自适应分辨率，在 Fantasia3D/Magic3D 等 SDS 驱动的 3D 生成中发挥了重要作用。
