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

## 总结

| 组件 | 功能 | 关键技术 |
|:-----|:-----|:---------|
| **Sparse VAE** | 3D 资产 ↔ SLAT 互转 | DINOv2 特征聚合、稀疏 Transformer |
| **Flow Transformer $G_S$** | 生成稀疏结构 | Rectified Flow |
| **Flow Transformer $G_L$** | 在结构上填充细节 | Sparse Rectified Flow |
| **Decoder $\mathcal{D}$** | SLAT → Mesh/3DGS/RF | FlexiCubes、可微分渲染 |

TRELLIS 的 SLAT 表示是一个强大的通用中间语言：既能精确描述已有 3D 资产，也能被生成模型创造，最终通过可复用的解码器转化为多种 3D 格式。这种设计使得 TRELLIS 成为众多 3D 编辑方法的首选骨干架构。
