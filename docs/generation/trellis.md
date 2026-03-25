---
comments: true
---

# TRELLIS

> *TRELLIS: Structured Latent Representations for 3D Generation*

TRELLIS 是当前 3D 生成与编辑领域的核心架构之一。多个编辑方法（VoxHammer, NANO3D, 3DEditVerse, Steer3D）均以 TRELLIS 为骨干。

---

## 核心思想：解耦

TRELLIS 的设计哲学是**解耦**：

- **学习与生成解耦**：Sparse VAE 学习如何理解和表示 3D 世界，Flow Transformer 负责创造新的表示
- **结构与细节解耦**：先确定"骨架"（宏观结构），再填充"血肉"（细节外观）

**SLAT (Structured LATent)** 表示是连接所有模块的核心桥梁。

---

## SLAT 表示

SLAT 是 TRELLIS 的核心中间表示：

$$
z = \{(z_i, p_i)\}_{i=1}^L
$$

- $\{p_i\}$：在 $64 \times 64 \times 64$ 网格中的活动体素位置集合 → 物体的**粗略轮廓**
- $\{z_i\}$：每个活动体素对应的局部潜变量 → 编码**精细几何与外观信息**

SLAT 的通用性在于：同一个表示可以被解码为 Mesh、3DGS、辐射场等多种格式。

---

## Part 1: Sparse VAE — 编解码

Sparse VAE 的任务不是生成 SLAT，而是**学习**如何将已有 3D 资产编码为 SLAT，并再解码回来。

### 编码流程

```
3D 资产
  ↓
Voxelize → 稀疏活动体素位置 {p_i}（结构信息）
  ↓
多视角渲染 → DINOv2 特征提取 → Multiview Average → 体素化特征图 {f_i}（细节信息）
  ↓
Sparse VAE Encoder → 压缩提炼
  ↓
SLAT 表示 z = {(z_i, p_i)}
```

### 解码流程

SLAT → Sparse VAE Decoder → 多种输出头：

| 输出头 | 输出内容 |
|:-------|:---------|
| Mesh 头 | FlexiCubes 参数 + SDF 值 → Mesh |
| 3DGS 头 | 每个体素的多个高斯球参数（位置、颜色、缩放） |
| RF 头 | 局部辐射场参数 |

### 训练

端到端训练，目标是最小化**重建损失**：

1. 原始 3D 资产 → 编码 → SLAT → 解码 → 重建 3D 资产
2. 同一视角渲染原始 vs 重建资产 → 计算 L1 / SSIM / LPIPS 损失
3. 加 KL 散度正则化潜变量空间

---

## Part 2: 两阶段生成

利用训练好的解码器 $\mathcal{D}$，通过两个 Flow Transformer 从无到有创建 SLAT。

### Stage 1: 稀疏结构生成

```
用户条件 (Text/Image) + 随机噪声
        ↓
  Flow Transformer G_S（结构 Rectified Flow）
        ↓
  小解码器 → 稀疏结构 {p_i}
```

**输出**：单色点云骨架——只有宏观形状，没有颜色/纹理。

### Stage 2: 结构化潜变量生成

```
Stage 1 的 {p_i} + 用户条件 + 噪声
          ↓
  Sparse Flow Transformer G_L（潜变量 Rectified Flow）
          ↓
  SLAT 表示 z = {(z_i, p_i)}
```

**输出**：完整的 SLAT——骨架上被赋予了丰富的颜色和细节信息。

### 最终解码

SLAT → 复用 Part 1 训练好的 Sparse VAE Decoder $\mathcal{D}$ → 3DGS / RF / Mesh

---

## SLAT → Mesh 解码说明

Mesh 解码器 $D_M$ 是 TRELLIS 重要的输出路径之一。

### 输出参数

对每个潜变量 $z_i$，解码器输出 64 组参数（对应 $4 \times 4 \times 4$ 高分辨率体素）：

$$
D_M: \{(z_i, p_i)\}_{i=1}^L \to \{\{(w^k, d^k)\}_{k=1}^{64}\}_{i=1}^L
$$

- **$w \in \mathbb{R}^{45}$**：FlexiCubes 参数（灵活性权重 $\alpha, \beta, \gamma, \delta$）
- **$d \in \mathbb{R}^8$**：一个体素 8 个顶点的 SDF 值

### 流程

| 步骤 | 操作 | 分辨率 |
|:-----|:-----|:-------|
| 1 | Transformer 主干网络处理 SLAT tokens | $64^3$ |
| 2 | 两个卷积上采样模块 | $64^3 \to 256^3$ |
| 3 | 参数预测：为每个体素输出 $(w, d)$ | $256^3$ |
| 4 | FlexiCubes 算法提取三角网格 | 最终 Mesh |

### 训练方式

通过**可微分渲染**间接优化：

1. FlexiCubes 提取 Mesh（可微分）
2. 渲染深度图 + 法线图
3. 与 Ground Truth 计算 L1 损失
4. 梯度反向传播回 $D_M$ 参数

---

## 训练与模型规模

| 配置 | 参数 |
|:-----|:-----|
| 训练数据 | ~500K 3D assets |
| 硬件 | 64×A100 (40GB) |
| 训练步数 | 400K steps，batch size 256 |
| 模型规模 | Basic 342M / Large 1.1B / XL 2B |
| SLAT 默认分辨率 | $64^3$ 网格，平均 ~20K active voxels |
| Latent channel | 8 channels |
| 推理 | CFG=3, 50 steps, ~10 秒生成 |

---

## 实验结果

### Reconstruction（Tab 1, Objaverse 测试集）

| 方法 | PSNR ↑ | LPIPS ↓ | CD ↓ | F-score ↑ |
|:-----|:-------|:--------|:-----|:----------|
| LN3Diff | 27.25 | 0.069 | 0.0184 | 0.9955 |
| 3DTopia-XL | 28.41 | 0.052 | 0.0137 | 0.9981 |
| CLAY | 30.99 | 0.034 | 0.0094 | 0.9997 |
| **TRELLIS** | **32.74** | **0.025** | **0.0083** | **0.9999** |

### Generation（Tab 2, Toys4K）

| 方法 | CLIP ↑ | FD_dinov2 ↓ |
|:-----|:-------|:------------|
| LN3Diff | 23.85 | 462.06 |
| Michelangelo | 24.76 | 311.43 |
| CLAY | 25.05 | 287.30 |
| **TRELLIS-L** | **26.60** | **238.60** |
| **TRELLIS-XL** | **26.70** | **237.48** |

User study 显示 TRELLIS 在文本条件下获得 67.1% 偏好率，图像条件下获得 94.5% 偏好率。

### Ablation

- $64^3$ 分辨率比 $32^3$ 有明显提升
- Rectified Flow 优于标准 diffusion
- 模型规模扩大（Basic → Large → XL）带来一致的质量提升

---

## 局限

- SLAT 依赖 SDF 解码路径，对开放表面和非流形结构支持有限
- 默认 $64^3$ 分辨率限制了极精细几何细节的表达
- 两阶段生成的结构阶段如果出错，后续细节阶段无法修正
- 训练需要大规模多视角渲染数据和 DINOv2 特征提取，数据准备成本高

---

## 总结

| 组件 | 功能 | 关键技术 |
|:-----|:-----|:---------|
| **Sparse VAE** | 3D 资产 ↔ SLAT 互转 | DINOv2 特征聚合、稀疏 Transformer |
| **Flow Transformer $G_S$** | 生成稀疏结构 | Rectified Flow |
| **Flow Transformer $G_L$** | 在结构上填充细节 | Sparse Rectified Flow |
| **Decoder $\mathcal{D}$** | SLAT → Mesh/3DGS/RF | FlexiCubes、可微分渲染 |

TRELLIS 的 SLAT 表示是一个通用中间语言：既能精确描述已有 3D 资产，也能被生成模型创造，最终通过可复用的解码器转化为多种 3D 格式。这种设计使得 TRELLIS 成为众多 3D 编辑方法的首选骨干架构。
