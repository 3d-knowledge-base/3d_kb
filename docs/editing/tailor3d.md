---
comments: true
---

# Tailor3D

> *Tailor3D: Customized 3D Assets Editing and Generation with Dual-Side Images*

Tailor3D 的核心直觉是：相比同时编辑四视图甚至更多视图，`正面 + 背面` 这两个视图已经覆盖了大部分对象信息，而且两者重叠少，更适合逐步编辑再重建。

---

## 核心问题

2D 提升回 3D 的路线常见一个矛盾：

- 视图多了，信息更全
- 但视图越多，编辑一致性越难保证

Tailor3D 的答案是：

> 不必一下子管四视图，把 3D 编辑拆成前后双视图编辑，再用专门的双视图 LRM 融合。

---

## 方法框架

### 1. 双阶段图像编辑

- 先编辑 front view
- 再通过 multi-view diffusion 生成 back view
- 然后继续编辑 back view

这样做的好处是：

- 正反面可以独立处理
- 减少同一区域在不同视角中重复编辑带来的冲突

### 2. Dual-sided LRM

- 输入 front / back 两张编辑后图像
- 输出统一的 triplane-NeRF 表示
- 重点是把双视图信息缝合成一个 3D 资产

### 3. LoRA Triplane Transformer + Viewpoint Cross-Attention

- 用 LoRA 低成本地把单视图 LRM 扩展到双视图输入
- 再通过 Viewpoint Cross-Attention 融合 front / back triplane features
- 解决双视图间颜色、几何和亮度不一致的问题

论文里这个比喻很明确：像裁缝一样先处理前后片，再把它们缝合起来。

---

## 它适合什么任务

Tailor3D 不是只做局部改色，而是覆盖几类典型编辑：

- 3D generative fill
- 局部几何补全
- 图案和纹理补全
- style transfer
- front/back 双侧风格融合

因此它更偏向“快速、可迭代定制 3D 资产”的工作流。

---

## 关键实验结论

### 速度

- Dual-sided LRM 推理少于 `5s`
- 整个 step-by-step 编辑流程每步都在秒级
- 在 8 张 A100 上训练约 `6h`

这让它和很多逐样本优化式方法相比更适合快速迭代。

### 双视图融合消融

论文对比了几种 front/back feature 融合方式：

- 直接加和
- 2D 卷积融合
- Viewpoint Cross-Attention

结论是 `Viewpoint Cross-Attention` 最好，说明双视图不是简单拼起来就够，仍需要显式对齐。

### LoRA rank

- 论文比较了 `rank = 2 / 4 / 8`
- 结果显示 `rank = 4` 最合适

这也说明 Tailor3D 的改造量并不大，主要是把已有 LRM 适配到双视图条件。

---

## 为什么它重要

Tailor3D 的贡献不在于提出一个全新的 3D 表示，而在于给 2D 编辑驱动的 3D 定制提供了一条更实用的中间路线：

- 比单视图更完整
- 比四视图更容易控制
- 比端到端 3D 编辑更接近当前成熟的 2D AIGC 工作流

这对实际内容制作流程很有吸引力，因为用户更容易理解和操作 front/back 编辑。

---

## 局限

- 仍然属于 2D 编辑再提升回 3D 的路线，不是直接 mesh-native 或 latent-native 编辑
- 若 front / back 本身生成得不一致，后续 Dual-sided LRM 只能部分修正
- 双视图虽比多视图简单，但侧面细节仍需要网络在融合时补全

---

## 一句话总结

Tailor3D 的主要价值，是把复杂的多视图 3D 编辑问题压缩成一个更实用的 `front/back dual-side editing` 流程，再通过 Dual-sided LRM 把双视图结果快速缝合成统一的 3D 资产。
