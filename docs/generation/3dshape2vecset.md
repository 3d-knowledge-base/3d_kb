---
comments: true
---

# 3DShape2VecSet

> *3DShape2VecSet: A 3D Shape Representation for Neural Fields and Generative Diffusion Models* (SIGGRAPH 2023)

3DShape2VecSet 是 VecSet 路线的奠基工作，提出了一种将 3D 形状编码为**固定长度潜向量集（latent vector set）**的表示方法，并在此基础上搭建扩散模型进行 3D 形状生成。它首次证明了 set-based latent 可以有效替代体素或点云特征，成为后续 CLAY、Michelangelo、TripoSG 等方法的共同起点。

---

## 核心问题

在 3DShape2VecSet 之前，3D 生成的隐空间设计面临两个困难：

- **固定网格绑定**：基于体素或 triplane 的方法将潜变量绑定到固定的空间位置上，在面对复杂拓扑和不规则几何时难以灵活适应。
- **维度不统一**：点云或 mesh 的顶点数量因形状而异，很难用统一维度的隐空间进行表示和生成。

3DShape2VecSet 的解决思路是：

> 用一组固定数量（M 个）的潜向量组成的集合来表示 3D 形状，这些向量不绑定到任何空间坐标上，而是通过 cross-attention 与 3D 空间建立关系。

---

## 编码器：从 3D 形状到 VecSet

编码器的任务是将一个 3D 形状压缩为 $M$ 个潜向量，论文探索了两种方式：

### Learnable Queries (DETR 式)

- 随机初始化 $M$ 个可学习 query 向量
- 通过 cross-attention 让 query 向量从输入点云中聚合信息
- 类似于 DETR 中 object queries 的机制

### Point Queries (FPS 下采样)

- 用 Farthest Point Sampling (FPS) 从输入点云中选取 $M$ 个锚点
- 以锚点坐标作为 query 的位置编码
- 通过 cross-attention 从完整点云中聚合特征

实验表明 **Point Queries 效果更好**，因为锚点提供了明确的空间参考位置，使 cross-attention 更容易聚焦。

### KL 正则化

编码器输出的每个潜向量维度为 $C=512$，通过线性层压缩到 $C'=32$ 的低维空间，并施加 KL 正则化（权重 0.001）约束潜空间接近标准高斯分布。最终的 VecSet 形状为 $M \times C' = 512 \times 32$。

---

## 解码器：从 VecSet 到隐式场

解码器接收 VecSet 潜向量集和一组 3D 查询点坐标，输出每个查询点的 occupancy 值。

流程如下：

1. **Self-Attention 处理 VecSet**：多层 self-attention blocks 让 $M$ 个潜向量之间交换信息，形成全局一致的形状表示
2. **Cross-Attention 插值**：查询点坐标作为 query，VecSet 作为 key/value，通过 cross-attention 为每个查询点聚合相关特征
3. **FC 层预测 Occupancy**：聚合后的特征经全连接层输出 occupancy 概率

这里的关键设计在于：**cross-attention 替代了传统的空间插值**（如三线性插值）。传统方法需要潜变量有明确的空间位置才能进行插值，而 cross-attention 让查询点自动找到与之相关的潜向量，潜变量不需要绑定到固定网格上。

---

## 损失函数

自编码器的训练损失由两部分组成：

$$\mathcal{L} = \mathcal{L}_{\text{BCE}} + \lambda_{\text{KL}} \cdot \mathcal{L}_{\text{KL}}$$

- **BCE Loss**：对查询点的 occupancy 预测施加二元交叉熵监督
- **KL Loss**：约束编码器输出的潜分布接近标准高斯，$\lambda_{\text{KL}} = 0.001$

---

## 扩散模型：在 VecSet 上生成

在 VecSet 潜空间上训练扩散模型进行生成。采用 EDM (Karras et al., 2022) 框架：

### 去噪网络

- 架构为纯 self-attention transformer
- 输入是加噪后的 VecSet（$M \times C'$）
- 条件信息（如类别、图像、文本）通过 cross-attention 注入
- 输出去噪后的 VecSet

### 采样

- 使用 18 步去噪采样
- 生成的 VecSet 送入解码器，对密集 3D 网格点查询 occupancy，再通过 Marching Cubes 提取 mesh

---

## 实验结果

在 ShapeNet-v2 的 55 个类别上评估：

### 自编码重建质量

| 指标 | 数值 |
|---|---|
| IoU | 0.974 |
| Chamfer-L1 | 0.309 |
| F-Score (τ=0.01) | 0.969 |

### 无条件生成质量（椅子类别）

| 指标 | VecSet | 3DILG | AutoSDF |
|---|---|---|---|
| FID ↓ | **33.0** | 41.2 | 50.4 |
| KID ↓ | **0.020** | 0.027 | 0.031 |

### 关键数字

| 参数 | 值 |
|---|---|
| 潜向量数量 $M$ | 512 |
| 潜向量维度 $C'$ | 32 |
| 编码器维度 $C$ | 512 |
| 去噪步数 | 18 |
| AE 训练 | 8× A100 |
| Diffusion 训练 | 4× A100 |

---

## 与后续工作的关系

3DShape2VecSet 开创了 VecSet 路线，后续工作在此基础上做了不同方向的扩展：

- **CLAY / Michelangelo**：将 VecSet 与更强的条件输入（文本、图像）结合，扩展到更大规模数据
- **TripoSG**：沿 VecSet 思路做 scaling，配合大规模数据和 Rectified Flow Transformer
- **TRELLIS / SLAT**：从纯 set 走向 structured latent，给 VecSet 加入空间结构
- **LATTICE / VoxSet**：在 VecSet 和 structured voxel 之间找到折衷的半结构化表示

VecSet 的核心洞察——**潜变量不需要绑定到固定空间位置，通过 cross-attention 即可建立与 3D 空间的关系**——被后续几乎所有 set-based 3D 生成方法继承。

---

## 局限

- 编码器依赖输入点云的采样质量，对稀疏或不均匀采样的鲁棒性有限
- 解码器的 cross-attention 在查询点数量很多时计算开销较大
- VecSet 中每个潜向量的语义角色不明确，难以进行可控的局部编辑
- 仅在 ShapeNet 上验证，未在大规模多样化数据（如 Objaverse）上测试

---

## 总结

3DShape2VecSet 的核心贡献是证明了用一组不绑定空间位置的潜向量集合来表示 3D 形状是可行且有效的。这一思路绕过了体素网格和 triplane 的空间绑定限制，为后续 VecSet 系列方法奠定了基础。虽然原始方法仅在 ShapeNet 上验证，但其 set-based latent + cross-attention decoding 的范式已经成为 3D 生成领域的基本构建块之一。
