---
comments: true
---

# TripoSG

> *TripoSG: High-Fidelity 3D Shape Synthesis using Large-Scale Rectified Flow Models*

TripoSG 并非单纯的单图到 3D 生成模型，而是一套系统化的 3D 生成方案：高质量数据构建、强几何 VAE、Rectified Flow Transformer、以及进一步的模型 scaling。

---

## 核心问题

TripoSG 要回答的是：

> 为什么 2D 生成已经进入"大模型 + 大数据 + 稳定训练范式"阶段，而 3D 生成还没有完全跟上？

论文认为瓶颈主要有三类：

- **数据问题**：互联网 3D 资产质量参差不齐，朝向、拓扑、纹理和几何一致性都不稳定
- **VAE 问题**：3D latent 压缩质量不够，后续 flow / diffusion 再强也学不到高保真几何
- **scale 问题**：3D 模型过去的训练规模、参数量和工程体系都不够成熟

所以 TripoSG 的目标不是一个单点 module 创新，而是构建一条可以 scale 的 3D shape generation pipeline。

---

## 整体结构

TripoSG 可以概括为三部分：

1. **大规模高质量 3D 数据治理**
2. **强几何 SDF VAE**
3. **Rectified Flow Transformer 生成器**

```text
图像条件
  -> 图像编码器 (CLIP + DINOv2)
  -> Rectified Flow Transformer
  -> 3D latent
  -> SDF VAE Decoder
  -> SDF field
  -> Mesh extraction
```

---

## 1. 数据治理为什么是主角

TripoSG 强调：如果数据质量差，3D 生成系统很难做起来。

它构建了一套数据 pipeline：

- **质量评分**：根据多视角法线图训练质量评估器
- **样本过滤**：清理大底座、多物体、动画异常、姿态异常等资产
- **方向修复**：特别是对角色和动物朝向进行统一
- **条件补全**：为无纹理/弱纹理资产生成可用的 RGB 条件视图
- **场数据生产**：把原始 mesh 处理成适合训练的 watertight / SDF 监督数据

最后得到约 **2M 高质量 3D shape 数据**。

---

## 2. SDF VAE：几何上限的基础设施

TripoSG 选择了 **SDF VAE**，而不是 occupancy VAE。

原因是：

- occupancy 更容易训练，但几何表达更粗
- SDF 对表面和细节更敏感
- 想做高保真 shape generation，latent 本身必须对几何足够忠实

### 编码

- 输入是更密集的 surface points（20,480 点）
- 通过 cross-attention 把点云压成 latent tokens
- 这一点和 `3DShape2VecSet` 的 latent set 思路有明显继承关系

### 解码

- latent -> neural SDF
- 再通过 SDF field 提取 mesh

### 训练监督

不是简单的 SDF reconstruction，而是多项结合：

$$
\mathcal{L}_{vae} = \mathcal{L}_{sdf} + \lambda_{sn}\mathcal{L}_{sn} + \lambda_{eik}\mathcal{L}_{eik} + \lambda_{kl}\mathcal{L}_{kl}
$$

各项具体定义：

- **SDF loss**：$\mathcal{L}_{sdf} = |s - \hat{s}|^2 + \|s - \hat{s}\|_2$
- **Surface normal guidance**：$\mathcal{L}_{sn} = 1 - \langle \frac{\nabla D(x,f)}{\|\nabla D(x,f)\|}, \hat{n} \rangle$
- **Eikonal regularization**：$\mathcal{L}_{eik} = \|\nabla D(x,f) - 1\|_2^2$
- **KL loss**：标准 VAE 隐空间正则化

损失权重：$\lambda_{sn}=10$，$\lambda_{eik}=0.1$，$\lambda_{kl}=0.001$。

其中 normal guidance 显式鼓励 latent 保留局部几何梯度信息，而不仅仅是表面值域。

---

## 3. Rectified Flow Transformer

TripoSG 在生成器上使用 **Rectified Flow**，而不是传统 DDPM。

原因很直接：

- 轨迹更简单（线性插值 $x_t = tx_0 + (1-t)\epsilon$）
- 更适合和大型 Transformer 配合
- 推理路径更直接

### 条件注入

它同时使用两种视觉特征：

- **CLIP 全局语义特征**
- **DINOv2 局部结构特征**

每个 flow block 中，两种特征分别通过独立 cross-attention 注入并合并。

### 架构细节

| 组件 | 参数 |
|:-----|:-----|
| Flow backbone | 2N+1 blocks，N=10，hidden dim 2048，16 heads |
| Base model | ~1.5B params |
| MoE 扩展 | 8 experts，top-2 activated，1 shared expert，应用于 decoder 最后 6 层 |
| 扩展后 | ~4B params |
| VAE encoder | 8 层 |
| VAE decoder | 16 层（更大以增强解码能力） |
| Latent tokens | L∈{512, 2048}，scaled to 4096；C=64 |

### 训练配置

| 阶段 | Token 数 | LR | Steps | 数据量 |
|:-----|:---------|:---|:------|:-------|
| Stage 1 | 512 | 1e-4 | 700K | 2M |
| Stage 2 | 2048 | 5e-5 | 300K | 2M |
| Stage 3 (MoE) | 4096 | 1e-5 | 100K | 1M 高质量子集 |

整体训练约 3 周，使用 160×A100。VAE 单独训练：2.5M steps，12 天，32×A100。

---

## 4. 实验结果

### Flow Model Ablation（Tab 1, 975M model, 512 tokens）

| 条件 | Skip-C | 采样策略 | Normal-FID ↓ |
|:-----|:-------|:---------|:-------------|
| DINOv2 | 无 | R-Flow | 10.69 |
| CLIP+DINOv2 | 无 | R-Flow | 10.61 |
| CLIP+DINOv2 | 有 | DDPM | 9.63 |
| CLIP+DINOv2 | 有 | EDM | 9.50 |
| CLIP+DINOv2 | 有 | R-Flow | 9.47 |

Skip-connection 对生成效果影响最大；R-Flow 在三种采样策略中表现最优。

### Scaling Up Ablation（Tab 2）

| 数据集 | Token 数 | MoE | Normal-FID ↓ |
|:-------|:---------|:----|:-------------|
| Objaverse | 512 | 否 | 9.47 |
| Objaverse | 2048 | 否 | 8.38 |
| Objaverse | 4096 | 否 | 8.12 |
| Objaverse | 4096 | 是 | 7.94 |
| **TripoSG** | **4096** | **是** | **3.36** |

最后一行是最终 TripoSG 模型（最大数据 + 最大模型 + 最高分辨率）。

### 数据质量 vs 数量（Tab 4）

| 数据集 | 数量 | 数据治理 | Normal-FID ↓ |
|:-------|:-----|:---------|:-------------|
| Objaverse | 800K | 无 | 11.61 |
| Objaverse | 180K | 有 | 9.47 |
| TripoSG | 2M | 有 | 5.81 |

数据质量比原始数量更重要；在高质量数据基础上继续扩量有更大收益。

### VAE Ablation（Tab 3）

SDF + normal guidance + eikonal 组合在 Chamfer distance (4.51) 和 Normal Consistency (0.958) 上均优于 occupancy baseline (4.59 / 0.952)。

---

## 5. 优势与局限

### 优势

- 高质量数据治理做得彻底
- SDF VAE 几何保真度强
- Rectified Flow + large Transformer + MoE 体现出明显 scaling 潜力
- 数据质量 > 数据数量的结论对后续工作有指导意义

### 局限

- 仍然依赖 field -> mesh extraction 路线
- 对开放表面、非流形等 native mesh 问题不是最自然的解法
- 工程体系重，复现成本高（160×A100，3 周）

---

## 一句话总结

TripoSG 主要的意义，不是提出一种最全新的 latent 表示，而是把"高质量数据 + 强几何 VAE + 大规模 Rectified Flow Transformer"这条已经在 2D 验证成功的路线，系统化搬到了 3D shape generation。
