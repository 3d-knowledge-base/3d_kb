---
comments: true
---

# Direct3D-S2

> *Direct3D-S2: Gigascale 3D Generation Made Easy with Spatial Sparse Attention*

Direct3D-S2 属于 sparse volumetric latent 路线中的代表工作之一。它关注的核心不是"提出一个最全新的表示哲学"，而是把 **Sparse SDF VAE + spatial sparse attention** 这条高分辨率 3D 生成路线推进到更可扩展的阶段。

---

## 核心问题

高分辨率 3D 生成常见的痛点是：

- 3D token 一多，注意力成本爆炸
- dense voxel/SDF 路线很难 scale
- 即使结构基本正确，局部边缘、锐角、薄结构也容易糊或碎

Direct3D-S2 的目标可以概括成：

> 在保留 sparse SDF 几何表达能力的同时，解决高分辨率训练与建模成本问题。

---

## 方法直觉

它的核心由两部分组成：

### 1. Sparse SDF VAE (SS-VAE)

- 用 sparse SDF 表示几何
- 输入、latent、输出都保持 sparse volumetric format（统一稀疏表示）
- 不再像一些早期方法那样在 point cloud / latent vec / dense volume 之间来回切换
- 端到端 SDF 重建框架，训练更稳，高分辨率重建链条更直接

训练配置：

- 多分辨率训练 $\{256^3, 384^3, 512^3\}$，1 天 on 8×A100，batch 4/GPU，lr=1e-4
- Fine-tune $1024^3$，1 天，lr=1e-5，batch 1/GPU
- **总计 2 天 on 8×A100**（远少于同类方法通常需要的 32+ GPU）

### 2. Spatial Sparse Attention (SSA)

SSA 是本文的核心创新，由三个模块组成：

- **Sparse 3D Compression** ($m_{cmp}=4$)：将稀疏 token 按空间分块压缩，获取全局 attention scores
- **Spatial Blockwise Selection** ($m_{slc}=8$)：基于 attention scores 选择最相关的空间块
- **Sparse 3D Window** ($m_{win}=8$)：局部窗口内做精细 attention

三步协同使模型既能捕获全局上下文，又不会因 token 数量增长导致计算爆炸。

SSA 通过定制 Triton GPU kernel 实现，在 128K tokens 时：

- Forward：**3.9×** 加速（vs FlashAttention-2）
- Backward：**9.6×** 加速

---

## SS-DiT 架构

| 组件 | 参数 |
|:-----|:-----|
| DiT layers | 24 层 |
| Hidden dim | 1024 |
| Attention | GQA，group=2，每组 16 heads，head dim 32 |
| 条件编码 | DINOv2-Large，输入 518×518 |
| 训练策略 | Progressive $256^3 \to 1024^3$ |

### 训练配置（Tab 1）

| 分辨率 | 平均 Token 数 | LR | Batch | 时间 |
|:-------|:-------------|:---|:------|:-----|
| $256^3$ | ~2,058 | 1e-4 | 8×8 | 2 天 |
| $384^3$ | ~5,510 | 1e-4 | 8×8 | 2 天 |
| $512^3$ | ~10,655 | 5e-5 | 8×8 | 2 天 |
| $1024^3$ | ~45,904 | 2e-5 | 2×8 | 1 天 |

**总计 7 天 on 8×A100**（另有 7 天训练 structure prediction DiT）。

---

## 实验结果

### Image-to-3D 定量对比（Tab 2）

| 方法 | ULIP-2 ↑ | Uni3D ↑ | OpenShape ↑ |
|:-----|:---------|:--------|:------------|
| Trellis | 0.2825 | 0.3755 | 0.1732 |
| Hunyuan3D 2.0 | 0.2535 | 0.3738 | 0.1699 |
| TripoSG | 0.2626 | 0.3870 | 0.1728 |
| Hi3DGen | 0.2725 | 0.3723 | 0.1689 |
| **Direct3D-S2** | **0.3111** | **0.3931** | **0.1752** |

三项 image-shape alignment 指标均优于所有对比方法。

### User Study

40 名参与者对 75 个 mesh 进行评分（1-5 分），Direct3D-S2 在 image consistency 和 overall quality 两个维度上均优于其他方法。

### SSA Ablation

- 仅用 window：局部细节好但表面不规则（缺少全局上下文）
- 加 compression：变化不大，主要为 selection 提供 attention scores
- 加 selection：模型聚焦于全局重要区域，mesh 质量明显提升
- Full attention baseline：因被迫 packing tokens 导致高频表面伪影

---

## 优势与局限

### 优势

- 延续 sparse SDF 的几何表达能力
- SSA 使高分辨率 3D DiT 训练实际可行
- 仅 8×A100 即可训练 $1024^3$ 生成模型
- 统一 sparse volumetric format 提升训练稳定性

### 局限

- 仍然依赖 field -> mesh 的恢复链条
- 对开放表面 / 原生 mesh 结构的表达不如 O-Voxel / FACE 路线自然
- SSA forward 加速比 (3.9×) 小于 backward (9.6×)，主要受 top-k sorting 开销限制
- 依赖定制 Triton kernel，工程实现门槛不低

---

## 一句话总结

Direct3D-S2 的意义，在于把 Sparse SDF VAE + Spatial Sparse Attention 这条路线向高分辨率、可扩展的 3D 生成系统推进了一步，用仅 8 GPU 实现了 $1024^3$ gigascale 3D 生成，是 sparse volumetric latent 路线中的关键工程推进节点。
