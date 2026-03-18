# SDS (Score Distillation Sampling)

**分数蒸馏采样 (Score Distillation Sampling, SDS)** 是将预训练 2D 扩散模型的知识"蒸馏"到 3D 空间的核心技术，由 Google 的 **DreamFusion** 提出。它使得无需任何 3D 训练数据，仅凭文本描述即可生成 3D 内容。

---

## 为什么需要 SDS？

3D 模型的创建需要大规模带标注的 3D 数据集，而这类数据的获取成本远高于 2D 图像。SDS 的核心思想：

> 既然我们已有能理解文本并生成高质量 2D 图像的强大模型（Stable Diffusion、Imagen 等），能否让它作为"评审"，评判 3D 模型在各角度的渲染图是否符合文本描述？

SDS 正是将这一思想付诸实践——**零样本 (Zero-shot)**，不需要修改 2D 模型，不需要 3D 训练数据。

---

## 工作原理

SDS 是一个反复"渲染→打分→优化"的循环：

### Step 1: 初始化 3D 表示
在三维空间中随机初始化一个可微分的 3D 场景表示（NeRF 或 3DGS），参数记为 $\theta$。

### Step 2: 随机视角渲染
随机选取相机位姿，通过可微分渲染器将当前 3D 模型渲染为 2D 图像 $x = g(\theta)$。

### Step 3: 添加噪声
模拟扩散模型前向过程，向渲染图添加噪声：$x_t = \alpha_t x + \sigma_t \epsilon$。

### Step 4: 扩散模型"打分"
将 $x_t$ 和文本 prompt 送入预训练的 2D 扩散模型。模型预测噪声 $\hat{\epsilon}_\phi(x_t; y, t)$。

**预测噪声与实际噪声的差异** = 模型对"当前渲染图与文本匹配程度"的指导信号。

### Step 5: 计算 SDS Loss & 反向传播

$$
\nabla_\theta \mathcal{L}_{SDS} = \mathbb{E}_{t, \epsilon} \left[ w(t) \left( \hat{\epsilon}_\phi(x_t; y, t) - \epsilon \right) \frac{\partial x}{\partial \theta} \right]
$$

梯度回传更新 3D 参数 $\theta$。

### Step 6: 迭代
重复上述过程数千至数万次，3D 模型逐步被"雕刻"成从任何角度看都符合文本的形状。

---

## 应用场景

### Text-to-3D 生成
最核心的用途——DreamFusion 开创性地实现了文本直接生成 3D 内容，无需 3D 训练数据。

### Text-Guided 3D 编辑
加载已有 3D 模型，根据新 prompt（如"给车加火焰贴纸"）计算 SDS 梯度修改几何/纹理。后续 UDS (Unified Distillation Sampling) 等工作将生成与编辑统一到同一框架。

---

## 局限性

| 问题 | 描述 |
|:-----|:-----|
| **Janus 问题** | 物体在不同方向都出现"正面"（如前后都有脸）——2D 模型缺乏 3D 一致性理解 |
| **过饱和 & 过平滑** | 纹理细节被抹平，颜色过于鲜艳 |
| **多样性有限** | 同一 prompt 倾向生成相似结果（Mode Collapse） |
| **文本对齐不足** | 细节可能与 prompt 不完全一致 |

---

## 改进方案

| 方法 | 核心改进 | 效果 |
|:-----|:---------|:-----|
| **VSD** (Variational Score Distillation) | 更复杂的优化目标（SDS 的泛化形式） | 提升质量和多样性，缓解过饱和 |
| **CSD** (Classifier Score Distillation) | 仅用 CFG 的 guidance 部分 | 简化流程，提升效果 |
| **ESD** (Entropic Score Distillation) | 引入熵最大化，鼓励多视角多样性 | 有效缓解 Janus 问题 |
| 噪声/视角策略优化 | 改进噪声采样策略和视角 prompt 工程 | 提升细节和一致性 |

---

## 总结

SDS 作为桥梁，成功将 2D 生成模型的知识迁移到数据匮乏的 3D 领域。尽管原始 SDS 存在 Janus、过平滑等问题，VSD/CSD/ESD 等后续工作正在持续改善。在 TRELLIS、Hunyuan3D 等现代 3D 生成模型中，SDS 的思想仍是重要的技术基础之一。
