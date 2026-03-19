---
comments: true
---

# VoxHammer

> *VoxHammer: Training-Free 3D Editing via DDIM Inversion and KV Cache Replacement on TRELLIS*

VoxHammer（2025.08）是首批基于 [TRELLIS](../generation/trellis.md) 骨干的 training-free 3D 编辑方法之一。核心思路是将成熟的 2D 扩散反演编辑范式（DDIM Inversion + Controlled Denoising）搬到 TRELLIS 的 3D 结构化潜空间中，并通过 **KV Cache Replacement** 机制解决编辑泄漏问题，实现精确的局部编辑与全局保留。同时提出 **Edit3D-Bench** 评估基准，为 3D 编辑领域建立了系统化的三维评估框架。

---

## 核心思想

VoxHammer 要回答的关键问题是：

> 如何在不训练任何额外模块的前提下，利用预训练 TRELLIS 对已有 3D 资产进行精确局部编辑？

思路很直接：

1. 对原始 3D 资产做 **DDIM 反演**，得到终端噪声 + 每个时间步的中间状态缓存
2. 用编辑后的 2D 图像作为新条件，对终端噪声重新去噪
3. 在去噪过程中，用缓存的原始信息**强制替换**未编辑区域的特征，防止编辑区域的修改"泄漏"到其他位置

这实质上是将 P2P（Prompt-to-Prompt）系列方法的 attention manipulation 思想，从 2D 扩散模型扩展到 TRELLIS 的 3D 稀疏 Flow Transformer。

---

## 整体流程

VoxHammer 的流程严格遵循 TRELLIS 的两阶段架构：

```text
原始 3D 资产
  ↓
TRELLIS Encoder → SLAT 表示 z = {(z_i, p_i)}
  ↓
[Stage 1: 结构 (ST) 阶段]
  DDIM Inversion → 终端噪声 z_T^ST + KV/Latent Cache
  Controlled Denoising (新条件) + KV/Latent Replacement → 编辑后结构 {p_i'}
  ↓
[Stage 2: 稀疏潜变量 (SLAT) 阶段]
  DDIM Inversion → 终端噪声 z_T^SLAT + KV/Latent Cache
  Controlled Denoising (新条件) + KV/Latent Replacement → 编辑后 SLAT
  ↓
TRELLIS Decoder → 编辑后 3D 资产 (Mesh / 3DGS)
```

两个阶段各自独立完成反演→缓存→替换的全套流程。

---

## 关键技术一：DDIM Inversion + RF-Solver

### 为什么需要精确反演

DDIM Inversion 的目标是从干净的潜变量 $z_0$ 出发，反推出对应的终端噪声 $z_T$，使得从 $z_T$ 重新去噪能精确恢复 $z_0$。

但 TRELLIS 使用的是 Rectified Flow，标准 DDIM 反演在离散采样步数有限时会累积数值误差，导致反演不精确 → 重建漂移 → 编辑区域与未编辑区域对不上。

### RF-Solver

VoxHammer 采用 **RF-Solver** 提升反演精度：

- 对 ODE 轨迹使用 **二阶 Taylor 展开**，减少每步离散化的截断误差
- 相比一阶 Euler 方法，在相同步数下重建保真度有明显提升

### CFG 策略

反演阶段的 Classifier-Free Guidance（CFG）策略：

| 时间区间 | CFG 状态 | 原因 |
|:---------|:---------|:-----|
| $t \in [0.0, 0.5)$ | 关闭 | 早期保持可逆性，避免 CFG 引入不可逆偏差 |
| $t \in [0.5, 1.0]$ | 开启 | 后期需要语义清晰度，平衡条件对齐 |

这种分段策略在可逆性与语义清晰度之间取得了平衡。

---

## 关键技术二：KV Cache Replacement

这是 VoxHammer 的主要创新点。

### 问题

在 TRELLIS 的 Sparse Flow Transformer 中，self-attention 是全局的——编辑区域的 token 会通过注意力机制影响到所有其他 token。即使只修改了一个区域的条件，整个模型的注意力分布都会被扰动，导致未编辑区域也发生变化。

### 解决方案

在反演阶段，缓存每个时间步、每一层的 attention KV 值：

$$
\mathcal{C} = \{(K_t^l, V_t^l)\}_{t=1,\ldots,T}^{l=1,\ldots,L}
$$

在去噪阶段，对于未编辑区域的 token：

- 不使用当前去噪过程计算出的 KV
- **直接替换**为缓存中对应时间步、对应层的原始 KV 值

这确保了未编辑区域的 token 在注意力计算中"看到的"与原始资产完全一致，从而阻断了编辑信号的泄漏路径。

### 数学表述

设 $M$ 为 3D mask（1 = 编辑区域，0 = 保留区域），去噪时间步 $t$，第 $l$ 层：

$$
K_t^l = M \odot \hat{K}_t^l + (1 - M) \odot K_{t,\text{cache}}^l
$$

$$
V_t^l = M \odot \hat{V}_t^l + (1 - M) \odot V_{t,\text{cache}}^l
$$

其中 $\hat{K}, \hat{V}$ 为当前去噪计算的值，$K_{\text{cache}}, V_{\text{cache}}$ 为反演阶段缓存的值。

---

## 关键技术三：Latent Replacement + 软掩码

除了 KV 替换外，VoxHammer 还在潜变量层面进行替换：

- 在每个时间步，将未编辑区域的潜变量直接用缓存的原始值替换
- 在编辑区域与保留区域的**边界处**，使用 **Gaussian Blur** 生成软掩码，平滑过渡

$$
z_t = M_{\text{soft}} \odot \hat{z}_t + (1 - M_{\text{soft}}) \odot z_{t,\text{cache}}
$$

其中 $M_{\text{soft}}$ 是经过高斯模糊的 3D mask。这避免了硬边界导致的接缝伪影。

---

## Edit3D-Bench 评估基准

VoxHammer 不只是一个编辑方法，还提出了一套系统化的 3D 编辑评估框架。

### 数据构成

| 来源 | 数量 | 说明 |
|:-----|:-----|:-----|
| GSO (Google Scanned Objects) | 50 | 真实扫描物体 |
| PartObjaverse-Tiny | 50 | 合成物体，带部件分割 |
| **总计** | **100 模型** | 每模型 3 种编辑 = **300 编辑任务** |

每个编辑任务附带：3D mask、对应 2D mask、FLUX.1 Fill 生成的编辑后参考图像。

### 三维评估体系

| 维度 | 指标 | 衡量目标 |
|:-----|:-----|:---------|
| **保留度** | Masked CD, PSNR, SSIM, LPIPS | 未编辑区域是否完好保持 |
| **整体质量** | FID, FVD, User Study | 编辑后资产的整体视觉质量 |
| **条件对齐** | DINO-I, CLIP-T | 编辑结果是否匹配参考图像 / 文本 |

这套三维度指标体系弥补了以往 3D 编辑工作评估不统一的问题。

---

## 实验结果

### 定量比较

VoxHammer 与五个基线方法在 Edit3D-Bench 上的比较：

| 方法 | 保留度 | 整体质量 | 条件对齐 | 备注 |
|:-----|:-------|:---------|:---------|:-----|
| Vox-E | 弱 | 弱 | 弱 | 优化式，NeRF 表示 |
| MVEdit | 中 | 中 | 中 | 2D lifting |
| Tailor3D | 中 | 中 | 中 | 双视图 |
| Instant3DiT | 中 | 中 | 中 | — |
| TRELLIS Inpainting | 中 | 较强 | 较强 | 原生 inpainting 模式 |
| **VoxHammer** | **强** | **强** | **强** | **全指标最优** |

### 用户研究

| 对比 | VoxHammer 偏好率 |
|:-----|:----------------|
| vs 排名第二的 2D-lifting 基线 | **70.3%** |
| vs TRELLIS Inpainting | **81.2%** |

### 消融实验

| 配置 | 效果 |
|:-----|:-----|
| 移除 KV Cache Replacement | 保留度**明显下降**，编辑泄漏明显 |
| 随机噪声初始化（不做反演） | 空间漂移，编辑区域位置不准确 |
| 移除 Latent Replacement | 边界处出现伪影 |
| 完整 VoxHammer | 最优 |

---

## 优势与局限

### 优势

- **Training-free**：直接复用预训练 TRELLIS，无需额外训练数据或微调
- **保留度强**：KV Cache + Latent Replacement 双重机制确保未编辑区域高保真
- **系统化评估**：Edit3D-Bench 为后续工作提供统一基准
- **工程友好**：单张 A100 GPU，25 步采样即可完成

### 局限

- **需要用户提供 3D mask**：不像 3DEditVerse / Steer3D 等方法可以自动定位编辑区域
- **间接依赖 2D 编辑**：需要先通过 FLUX.1 Fill 等工具在 2D 生成编辑后的参考图像
- **反演精度上限**：尽管使用了 RF-Solver，离散化误差仍不可完全消除，编辑幅度越大，累积偏差越明显
- **受限于 TRELLIS 能力边界**：TRELLIS 本身的几何/纹理表达能力决定了 VoxHammer 的上限

---

## 在编辑方法谱系中的位置

VoxHammer 属于 **Tuning-free Latent Editing** 路线：

- 与 **NANO3D** 同属 training-free 阵营，但技术路线不同（VoxHammer 用 DDIM Inversion，NANO3D 用 FlowEdit）
- 与 **3DEditVerse / Steer3D** 的区别在于不需要训练，但灵活性和上限可能受限
- 其 KV Cache Replacement 思想与 2D 领域的 P2P / MasaCtrl 一脉相承，是将 2D attention manipulation 技术成功迁移到 3D 的代表

---

## 一句话总结

VoxHammer 的主要贡献是证明了"DDIM 反演 + KV Cache 替换"这套 2D 扩散编辑范式可以直接迁移到 TRELLIS 的 3D 结构化潜空间中，在不训练任何额外参数的前提下实现高保真的局部 3D 编辑。
