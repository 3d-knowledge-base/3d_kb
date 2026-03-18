# 生成模型基础：从 DDPM 到 Flow Matching

视觉生成模型是 3D 生成技术的上游基础。现代 3D 生成方法（TRELLIS、Hunyuan3D、Direct3D 等）的骨干网络几乎全部来自 2D 视觉生成领域的技术演进。本页梳理从 DDPM 到 Flow Matching 的完整发展脉络。

---

## 发展全景

视觉生成技术的演进可划分为六个关键阶段：

| 阶段 | 核心突破 | 代表工作 | 时间 |
|:-----|:---------|:---------|:-----|
| 1. 扩散基础 | 稳定训练替代 GAN | DDPM | 2020 |
| 2. 采样加速 | 确定性采样 + 跳步 | DDIM | 2020 |
| 3. 流模型 | ODE 直线路径 | Flow Matching, Rectified Flow | 2022–2023 |
| 4. 潜空间压缩 | 像素空间 → 潜空间 | LDM / Stable Diffusion | 2022 |
| 5. 架构升级 | U-Net → Transformer | DiT | 2023 |
| 6. 条件控制 | 推理时引导 | CFG | 2022 |

下面逐一展开。

---

## 1. DDPM：扩散模型的起点

> *Denoising Diffusion Probabilistic Models* (Ho et al., 2020)

DDPM 确立了扩散模型的基本范式：**前向加噪 + 反向去噪**。

### 前向过程 (Forward Process)

逐步向数据 $x_0$ 添加高斯噪声：

$$
q(x_t | x_0) = \mathcal{N}(x_t; \sqrt{\bar{\alpha}_t}\, x_0,\; (1 - \bar{\alpha}_t)\, \mathbf{I})
$$

等价的**一步加噪**公式（重参数化）：

$$
x_t = \sqrt{\bar{\alpha}_t}\, x_0 + \sqrt{1 - \bar{\alpha}_t}\, \epsilon, \quad \epsilon \sim \mathcal{N}(0, \mathbf{I})
$$

其中 $\bar{\alpha}_t = \prod_{s=1}^t \alpha_s$ 是累积噪声调度系数。当 $t \to T$，$\bar{\alpha}_T \approx 0$，$x_T \approx \epsilon$（纯噪声）。

!!! note "VP 路径的几何约束"
    前向过程满足 **Variance Preserving (VP)** 约束：$a^2 + b^2 = 1$，其中 $a = \sqrt{\bar{\alpha}_t}$，$b = \sqrt{1-\bar{\alpha}_t}$。这意味着 $x_0 \to x_T$ 的路径是单位圆上的弧线（**弯曲路径**），而非直线。

### 反向过程 (Reverse Process)

训练一个网络 $\epsilon_\theta(x_t, t)$ 预测加入的噪声，使用 MSE 损失：

$$
\mathcal{L}_{\text{DDPM}} = \mathbb{E}_{t, x_0, \epsilon}\left[\|\epsilon - \epsilon_\theta(x_t, t)\|^2\right]
$$

反向采样需要**逐步**进行——每步依赖上一步的输出，且包含**随机项**：

$$
x_{t-1} = \frac{1}{\sqrt{\alpha_t}}\left(x_t - \frac{1-\alpha_t}{\sqrt{1-\bar{\alpha}_t}}\epsilon_\theta(x_t, t)\right) + \sigma_t z, \quad z \sim \mathcal{N}(0, \mathbf{I})
$$

### DDPM 的核心局限

- **采样极慢**：必须走完全部 T 步（通常 T=1000），无法跳步
- **随机性**：每步引入新噪声，同一起点不同采样结果不同
- **不可逆**：无法从生成结果反推回噪声（阻碍编辑）

---

## 2. DDIM：确定性采样与加速

> *Denoising Diffusion Implicit Models* (Song et al., 2020)

DDIM 重新推导了采样过程，发现可以**去掉随机项**，得到确定性采样公式：

$$
x_{t-1} = \sqrt{\bar{\alpha}_{t-1}}\underbrace{\left(\frac{x_t - \sqrt{1-\bar{\alpha}_t}\,\epsilon_\theta(x_t, t)}{\sqrt{\bar{\alpha}_t}}\right)}_{\text{predicted } \hat{x}_0} + \sqrt{1-\bar{\alpha}_{t-1}}\,\epsilon_\theta(x_t, t)
$$

### 关键突破

| 特性 | DDPM | DDIM |
|:-----|:-----|:-----|
| 采样随机性 | 有（每步加噪声） | **确定性**（同噪声 → 同图像） |
| 采样步数 | 必须 ~1000 步 | **可跳步**（50 步即可） |
| 可逆性 | 不可逆 | **可逆**（ODE 正/反向） |
| 训练 | MSE 噪声预测 | **复用 DDPM 权重**（不需重训） |

### 为什么 DDPM 不能跳步而 DDIM 可以？

DDPM 的每步去噪依赖于"上一步恰好是 $t-1$"的马尔可夫假设，跳步会破坏条件概率的正确性。

DDIM 将过程重新解释为**常微分方程 (ODE)** 的离散化——ODE 的解与步长无关（仅影响精度），因此可以自由选择任意子序列的时间步。

### $\hat{x}_0$ 预测 (Tweedie Estimator)

DDIM 的一个重要副产品是 **$\hat{x}_0$ 预测**——在任意时间步 $t$，可以用当前噪声预测估计原始数据：

$$
\hat{x}_0 = \frac{x_t - \sqrt{1-\bar{\alpha}_t}\,\epsilon_\theta(x_t, t)}{\sqrt{\bar{\alpha}_t}}
$$

这一估计在后续的 SDS、编辑、反演等技术中被反复使用。

---

## 3. 从弯曲路径到直线路径：Flow Matching 与 Rectified Flow

### 路径几何的本质差异

DDPM/DDIM 与 Flow Matching 最根本的区别在于 $x_0 \to x_1$（数据→噪声）的**路径形状**：

| 特性 | VP 扩散 (DDPM/DDIM) | Flow Matching / Rectified Flow |
|:-----|:---------------------|:-------------------------------|
| 路径约束 | $a^2 + b^2 = 1$（圆弧） | $a + b = 1$（直线） |
| 几何形状 | **弯曲**（单位圆弧） | **直线**（线性插值） |
| 采样效率 | 弯曲路径需更多步离散化 | 直线路径天然适合少步采样 |
| 速度场变化 | 方向持续变化 | **方向近似恒定** |
| ODE 求解难度 | 较难（曲率导致误差累积） | 较易（直线→Euler 一步即可近似） |

!!! info "直觉理解"
    想象从 A 点走到 B 点：沿圆弧走需要不断调整方向（每步都要重新计算切线方向），沿直线走只需知道 A→B 的方向然后一直走下去。步数越少，直线的近似误差越小。

### Flow Matching

> *Flow Matching for Generative Modeling* (Lipman et al., 2022)

Flow Matching 将生成建模为 **ODE 求解问题**：学习一个速度场 $v_\theta(x_t, t)$，将噪声分布沿时间流动到数据分布。

**线性插值路径**：

$$
x_t = (1-t)\, x_0 + t\, \epsilon, \quad t \in [0, 1]
$$

**目标速度场**：

$$
v^*(x_t, t) = \epsilon - x_0
$$

**训练目标**（MSE Loss）：

$$
\mathcal{L}_{\text{FM}} = \mathbb{E}_{t, x_0, \epsilon}\left[\|v_\theta(x_t, t) - (\epsilon - x_0)\|^2\right]
$$

关键特性：

- **Simulation-free training**：不需要模拟 ODE 轨迹，直接在随机采样的 $(x_0, \epsilon, t)$ 上训练
- 训练复杂度与扩散模型**完全相同**（一次前向 + 一次 MSE Loss）
- 理论上更优：直线路径使得速度场更平滑，ODE 求解更容易

### Rectified Flow

> *Flow Straight and Fast: Learning to Generate and Transfer Data with Rectified Flow* (Liu et al., 2023)

Rectified Flow 的核心贡献是 **Reflow 过程**——一种迭代式路径矫直方法：

```text
[Round 0] 训练 Flow Matching 模型 v₀
    ↓
用 v₀ 从噪声 ε 生成数据 x̂₀ → 得到配对 (x̂₀, ε)
    ↓
[Round 1] 用新配对 (x̂₀, ε) 重新训练 → v₁（路径更直）
    ↓
用 v₁ 生成新配对 → 重新训练 → v₂（路径更直）
    ↓
... 迭代直到路径几乎是直线
```

**为什么初始 Flow Matching 的路径不够直？**

虽然 Flow Matching 的训练目标是线性插值，但因为训练数据中 $(x_0, \epsilon)$ 是**随机配对**的，学到的速度场是所有可能路径的**平均**——实际 ODE 轨迹仍有弯曲。

Reflow 通过重新配对（用模型生成的 $\hat{x}_0$ 与对应的 $\epsilon$ 绑定），逐步消除路径交叉，使轨迹趋向直线。

**效果**：经过 1-2 轮 Reflow，模型可以用 **1-2 步**采样生成高质量结果。

!!! note "在 3D 生成中的应用"
    TRELLIS 使用 Rectified Flow 作为其 Flow Transformer 的采样框架。VoxHammer 论文中的 DDIM Inversion 实际上是对 Rectified Flow ODE 的反向求解。

---

## 4. LDM / Stable Diffusion：潜空间革命

> *High-Resolution Image Synthesis with Latent Diffusion Models* (Rombach et al., 2022)

### 核心问题

直接在像素空间做扩散计算量巨大。一张 $512 \times 512 \times 3$ 的图像有 ~786K 维度，每步去噪都需要在这个高维空间上做完整的 U-Net 前向传播。

### 解决方案：两步走

LDM 将扩散过程从像素空间搬到低维潜空间：

```text
[训练阶段 1] 训练 Autoencoder (Encoder + Decoder)
    图像 x ∈ R^{512×512×3} → 潜变量 z ∈ R^{64×64×4}
    压缩比 ~48倍

[训练阶段 2] 在潜空间上训练扩散模型
    前向：z₀ → z_t（加噪）
    反向：z_t → z₀（去噪）
    最后：z₀ → Decoder → 图像
```

### Autoencoder 的演进

LDM 的 Autoencoder 设计继承了一条清晰的演进路线：

| 方法 | 核心贡献 | 路线 |
|:-----|:---------|:-----|
| **AE** | 基础编解码框架 | — |
| **VAE** | 正则化潜空间（KL 散度） | 提升质量 |
| **VQVAE** | 离散码本量化 | 提升效率 |
| **VQGAN** | VQVAE + GAN 判别器 + 感知 Loss | 质量 + 效率 |
| **LDM VAE** | 连续潜空间 + KL 正则化 | 最终方案 |

LDM 最终选择了**连续潜空间 + 轻度 KL 正则化**（而非离散码本），因为连续空间对扩散模型的高斯噪声假设更友好。

### 条件注入机制

LDM / Stable Diffusion 定义了多种将条件信息注入去噪网络的方式：

| 机制 | 注入方式 | 适用条件类型 |
|:-----|:---------|:-------------|
| **Cross-Attention** | 条件编码作为 K, V，潜变量作为 Q | 文本（CLIP 编码）、图像特征 |
| **adaLN (Adaptive LayerNorm)** | 条件调制 LayerNorm 的 scale/shift | 时间步 $t$、类别标签 |
| **Concatenation** | 条件直接拼接到输入通道 | 深度图、mask、参考图像 |
| **额外分支** | ControlNet / ReferenceNet 等 | 边缘图、姿态、参考图像 |

Cross-Attention 是最通用的机制——它允许去噪网络在每层"查询"条件信息，实现灵活的语义对齐。

---

## 5. DiT：Transformer 取代 U-Net

> *Scalable Diffusion Models with Transformers* (Peebles & Xie, 2023)

### 动机

U-Net 是扩散模型的经典骨干，但存在结构瓶颈：

- 编码器-解码器结构 + 跳跃连接 → 工程复杂
- 参数量与层数不成正比 → 扩展性有限
- 无法直接利用大规模 Transformer 训练的基础设施和 Scaling Law

### DiT 架构

DiT 用 **Vision Transformer** 完全替代 U-Net：

```text
潜变量 z ∈ R^{h×w×c}
    ↓
Patchify → 切分为 patch tokens
    ↓
Linear Embedding → token 序列
    ↓
N × Transformer Block (Self-Attention + FFN + adaLN-Zero)
    ↓
Unpatchify → 预测噪声 ε̂
```

### 关键设计：adaLN-Zero

条件信息（时间步 $t$、类别标签）通过 **Adaptive LayerNorm-Zero (adaLN-Zero)** 注入：

- 将条件编码为 scale ($\gamma$) 和 shift ($\beta$) 参数
- 调制每个 Transformer Block 的 LayerNorm 输出
- "Zero" 指初始化时 scale 接近 0，使初始网络接近恒等映射（稳定训练）

### 为什么 DiT 更好？

| 优势 | 说明 |
|:-----|:-----|
| **Scaling Law** | 参数量↑ → 性能持续提升（与 LLM 类似） |
| **灵活性** | 纯 Transformer，可直接复用 NLP/CV 的训练技巧 |
| **并行性** | 所有 token 并行处理，无需 U-Net 的串行编解码 |
| **统一架构** | 图像/视频/3D 可共用同一骨干 |

!!! note "在 3D 生成中的影响"
    TRELLIS 的 Sparse Flow Transformer 本质上是 DiT 在 3D 稀疏数据上的适配：用 Sparse Attention 处理 SLAT 结构化潜变量中的稀疏 token，条件通过 adaLN 注入。

---

## 6. CFG：推理时的条件引导

> *Classifier-Free Diffusion Guidance* (Ho & Salimans, 2022)

### 核心思想

CFG 是一种**推理时**增强条件对齐的技术，不改变模型架构，只改变采样策略。

### 训练

训练时以概率 $p$（通常 10-20%）**随机丢弃条件**（用空条件 $\varnothing$ 替代），使模型同时学会：

- 有条件生成：$\epsilon_\theta(x_t, t, c)$
- 无条件生成：$\epsilon_\theta(x_t, t, \varnothing)$

### 推理

采样时对两个预测做**线性外推**：

$$
\hat{\epsilon} = \epsilon_\theta(x_t, t, \varnothing) + w \cdot \left(\epsilon_\theta(x_t, t, c) - \epsilon_\theta(x_t, t, \varnothing)\right)
$$

其中 $w$ 是引导强度（guidance scale）：

| $w$ 值 | 效果 |
|:-------|:-----|
| $w = 0$ | 纯无条件生成 |
| $w = 1$ | 标准条件生成（无额外引导） |
| $w = 3\text{–}7$ | 常用范围，质量与多样性平衡 |
| $w > 10$ | 过度引导，过饱和/过平滑 |

### 直觉理解

$\epsilon_c - \epsilon_{uc}$ 可以理解为"条件相比无条件多出来的方向"。$w > 1$ 时，模型沿这个方向**过度移动**（外推），使生成结果更强烈地符合条件，但牺牲多样性。

!!! info "CFG 与 SDS 的关系"
    SDS 的梯度公式 $\nabla_\theta \mathcal{L} \propto (\hat{\epsilon}_\phi - \epsilon)$ 本质上也在使用 CFG 增强后的噪声预测。CFG 的引导强度直接影响 SDS 的优化方向和生成质量。

---

## 技术演进总结

```text
GAN (不稳定)
  ↓  DDPM 替代
DDPM (稳定但慢，1000步)
  ↓  去掉随机项
DDIM (确定性，50步，可逆)
  ↓  弯曲→直线
Flow Matching (直线路径，更高效)
  ↓  迭代矫直
Rectified Flow (1-2步生成)

---（正交维度）---

像素空间扩散 (计算量巨大)
  ↓  压缩到潜空间
LDM / Stable Diffusion (48x 压缩)

U-Net 骨干 (扩展性有限)
  ↓  Transformer 替代
DiT (Scaling Law)

无条件生成
  ↓  推理时引导
CFG (质量↑，多样性↓)
```

---

## 在 3D 生成中的映射

这些 2D 技术直接映射到现代 3D 生成架构：

| 2D 技术 | 3D 对应 | 代表方法 |
|:---------|:--------|:---------|
| LDM 潜空间 | SLAT 结构化潜空间 | TRELLIS |
| DiT | Sparse Flow Transformer | TRELLIS |
| Rectified Flow | Flow Transformer 采样 | TRELLIS |
| CFG | 条件引导（图像/文本） | TRELLIS, Direct3D |
| DDIM Inversion | 3D 编辑反演 | VoxHammer |
| Flow Matching | 3D 潜空间去噪 | Hunyuan3D 2.0 |

理解这些基础技术，是理解现代 3D 生成方法的前提。
