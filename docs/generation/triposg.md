# TripoSG

> *TripoSG: High-Fidelity 3D Shape Synthesis using Large-Scale Rectified Flow Models*

TripoSG 不是单纯“又一个单图到 3D 模型”，而是一套非常系统化的 3D 生成放大方案：高质量数据构建、强几何 VAE、Rectified Flow Transformer、以及进一步的模型 scaling。

---

## 核心问题

TripoSG 要回答的是：

> 为什么 2D 生成已经进入“大模型 + 大数据 + 稳定训练范式”阶段，而 3D 生成还没有完全跟上？

论文认为瓶颈主要有三类：

- **数据问题**：互联网 3D 资产质量参差不齐，朝向、拓扑、纹理和几何一致性都不稳定
- **VAE 问题**：3D latent 压缩质量不够，后续 flow / diffusion 再强也学不到高保真几何
- **scale 问题**：3D 模型过去的训练规模、参数量和工程体系都不够成熟

所以 TripoSG 的真正目标不是一个单点 module 创新，而是构建一条可以真正 scale 的 3D shape generation pipeline。

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

TripoSG 非常强调：如果数据质量差，3D 生成系统很难真正做起来。

它构建了一套完整的数据 pipeline：

- **质量评分**：根据多视角法线图训练质量评估器
- **样本过滤**：清理大底座、多物体、动画异常、姿态异常等资产
- **方向修复**：特别是对角色和动物朝向进行统一
- **条件补全**：为无纹理/弱纹理资产生成可用的 RGB 条件视图
- **场数据生产**：把原始 mesh 处理成适合训练的 watertight / SDF 监督数据

最后得到约 **2M 高质量 3D shape 数据**。

这一步很关键，因为 TripoSG 的很多优势，其实来自“能不能喂给模型一套真正干净的大规模几何训练集”。

---

## 2. SDF VAE：几何上限的基础设施

TripoSG 选择了 **SDF VAE**，而不是 occupancy VAE。

原因是：

- occupancy 更容易训练，但几何表达更粗
- SDF 对表面和细节更敏感
- 想做高保真 shape generation，latent 本身必须对几何足够忠实

### 编码

- 输入是更密集的 surface points
- 通过 cross-attention 把点云压成 latent tokens
- 这一点和 `3DShape2VecSet` 的 latent set 思路有明显继承关系

### 解码

- latent -> neural SDF
- 再通过 SDF field 提取 mesh

### 训练监督

不是简单的 SDF reconstruction，而是多项结合：

- **SDF loss**
- **surface normal guidance**
- **eikonal regularization**
- **KL loss**

其中 normal guidance 很重要，因为它显式鼓励 latent 保留局部几何梯度信息，而不仅仅是表面值域。

---

## 3. Rectified Flow Transformer

TripoSG 在生成器上使用 **Rectified Flow**，而不是传统 DDPM。

原因很直接：

- 轨迹更简单
- 更适合和大型 Transformer 配合
- 推理路径更直接

### 条件注入

它同时使用两种视觉特征：

- **CLIP 全局语义特征**
- **DINOv2 局部结构特征**

这使它既能理解对象类别/语义，又能更好对齐图像局部细节。

### Scaling

TripoSG 的一个重点是 progressive scaling：

- latent token 从低分辨率逐步提升到高分辨率
- 模型从 dense 版本扩到更大的 **MoE** 版本
- 最终参数规模达到约 **4B**

这说明它不只是“结构好”，而是明确在走一条 3D foundation model 的 scaling 路线。

---

## 4. 它在表示演化中的位置

TripoSG 不像 TRELLIS 那样提出全新的结构化 latent，也不像 LATTICE 那样提出一整套 `localizable code` 理论。

它在演化线里的位置更像：

- **承接 `3DShape2VecSet` / latent set + VAE 路线**
- **强调 SDF latent 的几何质量**
- **把大模型 / 大数据 / RF scaling 系统化搬到 3D**

所以它更像是这条线上的**系统化放大器**。

---

## 5. 优势与局限

### 优势

- 高质量数据治理做得非常彻底
- SDF VAE 几何保真度强
- Rectified Flow + large Transformer + MoE 体现出明显 scaling 潜力
- 说明 3D 生成确实可以沿着 foundation-model 路线继续做大

### 局限

- 仍然依赖 field -> mesh extraction 路线
- 对开放表面、非流形等 native mesh 问题不是最自然的解法
- 工程体系重，复现成本高

---

## 一句话总结

TripoSG 最重要的意义，不是提出一种最全新的 latent 表示，而是把“高质量数据 + 强几何 VAE + 大规模 Rectified Flow Transformer”这条已经在 2D 验证成功的路线，真正系统化搬到了 3D shape generation。
