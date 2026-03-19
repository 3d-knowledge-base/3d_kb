---
comments: true
---

# ShapeFusion

> *ShapeFusion: A 3D diffusion model for localized shape editing*

ShapeFusion 关注的是顶点空间里的局部形变。它不依赖额外的 3D 潜表示，而是直接在 mesh 顶点坐标上做局部扩散编辑，并通过 mask 训练保证未编辑区域尽量不动。

---

## 核心问题

传统 3DMM / PCA 这类参数模型更擅长全局变化，但不适合精确、局部、可解释的区域编辑：

- latent 空间通常是纠缠的
- 局部调一个区域，别处也会跟着动
- 很难做到“只改这里”

ShapeFusion 的目标是：

> 允许用户指定任意局部区域和控制点，在 3D mesh 上做完全局部的扩散编辑。

---

## 方法框架

### 1. Masked diffusion training

- 训练时只给 mesh 的一部分区域加噪
- mask 由随机 anchor point 的 geodesic 邻域定义
- 未被 mask 的区域保持不变

这样做的直接效果是：模型从训练方式上就学会“局部改、其余保留”。

### 2. 顶点级 denoising

- 去噪器直接在 3D 顶点空间中工作
- 目标是恢复被扰动区域的几何
- 同时保住未被扰动区域的结构

### 3. Mesh-aware denoiser

- 用 vertex-index positional encoding 让每个顶点有稳定身份
- 用分层 mesh convolution 传播局部与远距离信息
- 让编辑区域与未编辑区域边界更平滑

论文这里的重点不是“更强的 latent”，而是让 denoiser 本身真正理解 mesh 拓扑。

---

## 它能做什么

ShapeFusion 支持两类典型操作：

- `localized manipulation`：对选中区域做可控形变
- `region sampling`：在指定局部采样新的部件或表情

用户不需要被限制在固定的人脸部件或少量预定义控制点上，而是可以自己定义区域。

---

## 为什么它重要

ShapeFusion 的意义主要有两点：

- 它把局部编辑问题从 latent disentanglement 转成了 masked diffusion inpainting
- 它说明“局部编辑”可以直接在 mesh 顶点空间里通过扩散过程完成

这和后来的原生 3D latent 编辑路线不同，但它在“局部保留 + 局部变化”这个问题上给出了很干净的形式化方式。

---

## 论文里值得记住的点

- 方法不是依赖固定语义部件，而是支持任意自定义区域
- 通过顶点索引编码和分层 mesh convolution，局部编辑不会轻易破坏整体拓扑结构
- 相比基于优化或 latent disentanglement 的方法，它更直接，也更容易解释编辑范围

---

## 局限

- 论文主要在人脸和人体参数化 mesh 上验证，开放域通用 mesh 场景不是重点
- 依赖固定 mesh 拓扑，泛化到拓扑差异很大的对象并不自然
- 更适合几何变形，不涉及高质量纹理或完整资产编辑流程

---

## 一句话总结

ShapeFusion 的主要价值，是把局部 3D mesh 编辑建模为一个 masked diffusion inpainting 问题，在顶点空间里直接做到“只改局部、其余保留”。
