---
comments: true
---

# Native 3D Editing

> *Native 3D Editing (2025.11) — 在 TRELLIS 的 SLAT 隐空间中直接进行前馈式 3D 编辑*

Native 3D Editing 是"原生 3D 编辑"路线的代表工作。它不借助任何 2D 中间编辑结果，而是直接在 TRELLIS 的 SLAT 潜变量空间中完成源模型到目标模型的编辑映射。论文的核心发现是：**Token Concatenation 策略优于 Cross-Attention**，前者让目标 token 能在自注意力中直接关注源 token，从而保持几何一致性；后者将源信息视为外部条件，导致信息碎片化和几何失真。

---

## 核心架构：Token Concatenation

Native 3D Editing 的编辑架构基于 Token Concatenation 策略，将源模型和目标模型的 latent 拼接后送入统一的 Transformer 处理。完整流程如下：

1. **Patchify + 投影 + 位置编码**：将源 SLAT $z_s$ 和加噪目标 SLAT $z_t$ 分别序列化为 token 序列
2. **序列维度拼接**：$h_{\text{comb}} = \text{Concat}(h_t, h_s)$
3. **全自注意力**：目标 token 可以直接 attend 到源 token，信息流完整无遮挡
4. **文本指令**：通过常规的 cross-attention 注入编辑指令
5. **分离与解码**：经过 N 个 Transformer block 后，从拼接序列中分离出目标部分，解码得到编辑后的 SLAT

### 为什么不用 Cross-Attention？

论文同时测试了 Cross-Attention 作为 baseline：将源 SLAT 当作类似文本的外部条件，通过并行的 cross-attention 层注入。结果表明这种方式会导致：

- **信息碎片化**：源信息被压缩到 KV 投影中，target 只能间接访问
- **几何失真**：未编辑区域的几何无法精确保持
- **颜色不一致**：外观信息传递不完整

消融实验明确证实：Token Concatenation 在几何一致性和编辑保真度上优于 Cross-Attention。

---

## 数据构建

高质量的 3D 编辑对（source, target, instruction）是训练的关键。论文针对不同编辑类型设计了专门的数据流水线：

### 删除（Deletion）

- **数据来源**：Objaverse 中具有零件级标注的资产
- **流程**：程序化地移除指定零件 → 用 Gemini 2.5 识别零件名称并生成编辑描述
- **数据量**：Stage 1 共 64K 对，Stage 2 共 45K 对

### 添加与修改（Addition & Modification）

- **数据来源**：3D-Alpaca 2D 编辑数据集
- **流程**：Hunyuan3D 2.1 将 2D 编辑对提升为 3D 对 → **严格人工筛选**
- **筛选标准**：指令一致性（编辑内容是否忠实于指令）、未编辑区域保持（非编辑部分是否完好）、整体质量
- **数据量**：Stage 1 共 47K 对，Stage 2 共 47K 对

### 消融：数据质量的影响

论文发现使用 Hunyuan3D 2.1 生成并经人工筛选的数据，效果优于仅用 TRELLIS 自身编码-解码生成的数据。这说明编辑数据的多样性和保真度对模型性能很重要。

---

## 训练细节

模型基于 TRELLIS 架构，使用预训练权重初始化，分两阶段训练：

| 阶段 | 目标 | 步数 | Batch Size | GPU |
|:-----|:-----|:-----|:-----------|:----|
| **Stage 1** | 稀疏结构（Sparse Structure） | 150K | 12 | 16× A800 |
| **Stage 2** | 局部潜变量（Local Latents） | 80K | 8 | 18× A800 |

- 优化器：AdamW
- 学习率：$1 \times 10^{-4}$

两阶段训练延续了 TRELLIS 的"结构→细节"解耦哲学：先学会在宏观结构层面正确编辑，再学习保持和修改精细外观。

---

## 实验结果

### 主实验（Table 1）

| 方法 | FID ↓ | FVD ↓ | CLIP ↑ |
|:-----|:------|:------|:-------|
| Instant3DiT | 255.5 | 1209.8 | 0.225 |
| VoxHammer | 169.6 | 594.2 | 0.230 |
| **Native 3D Editing** | **91.9** | **286.5** | **0.249** |

三个指标均领先：

- **FID 91.9**：生成质量优于两个基线，说明编辑后的 3D 资产整体保真度高
- **CLIP 0.249**：语义对齐度最好，编辑结果与文本指令匹配更准确
- **FVD 286.5**：这是衡量多视角一致性的重要指标——FVD 衡量多视角渲染序列的时序一致性，领先幅度（降幅超过 50%）证明了原生 3D 编辑在**几何一致性**上的优势

### 为什么 FVD 改进最能说明问题？

FVD（Fréchet Video Distance）评估的是从不同视角渲染的序列质量。如果编辑后的 3D 资产存在几何失真或多视角不一致，FVD 会明显恶化。Native 3D Editing 的 FVD 从 VoxHammer 的 594.2 降到 286.5（降幅超过 50%），直接证明了在原生 3D 隐空间中编辑对几何一致性的优势。

---

## 消融实验

论文的消融实验验证了两个核心设计选择：

### Token Concatenation vs Cross-Attention

Token Concatenation 优于 Cross-Attention。后者将源模型当作外部条件处理，导致目标模型无法充分获取源模型的空间细节，表现为几何失真和颜色不一致。而 Token Concatenation 让源和目标 token 在同一序列中做全自注意力，信息传递无损。

### 数据来源消融

使用 Hunyuan3D 2.1 提升并经人工筛选的编辑对，效果优于仅使用 TRELLIS 编解码器构造的数据。这表明：

- 外部高质量 3D 生成模型可以有效扩展编辑训练数据
- 人工筛选对确保编辑指令一致性和未编辑区域保持是必要的
- 数据质量比数据量更重要

---

## 在编辑路线中的定位

Native 3D Editing 属于 [Mesh Editing Landscape](mesh-editing-landscape.md) 中 **Fully native 3D editing** 子路线的代表。与 tuning-free 方法（VoxHammer, NANO3D）和 2D-guided 方法（CraftMesh, PrEditor3D）不同，它选择了最直接的路径：

> 直接在 3D 隐空间中学习编辑映射，不借助任何 2D 中间步骤。

这条路线的代价是需要大量高质量 3D 编辑对和可观的训练资源，但回报是几何一致性和编辑保真度的提升。从结果来看，这个 trade-off 是值得的。

---

## 一句话总结

Native 3D Editing 证明了一件事：当你拥有足够好的 3D latent 表示（SLAT）、正确的条件注入方式（Token Concatenation 而非 Cross-Attention）、以及高质量的编辑训练数据时，**在原生 3D 空间中直接编辑**可以优于依赖 2D 中间步骤或 training-free tricks 的方法。
