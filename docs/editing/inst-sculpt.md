---
comments: true
---

# INST-Sculpt

> *INST-Sculpt: Interactive Stroke-based Neural SDF Sculpting*

INST-Sculpt 解决的是一个很具体但很实用的问题：如果形状已经表示成 neural SDF，用户能不能像在 ZBrush 里一样，直接画 stroke 来雕刻，而不用切回 mesh 或其他显式表示。

---

## 核心问题

neural SDF 很适合连续表示和高分辨率建模，但交互编辑并不直接：

- 参数藏在 MLP 权重里
- 用户看不到“哪个局部由哪些参数控制”
- 很难像 mesh 一样直接刷一笔就改一个局部

INST-Sculpt 的目标是：

> 在不离开 neural SDF 表示的前提下，提供接近数字雕刻工具的交互式 stroke 编辑。

---

## 方法框架

### 1. Stroke-based sculpting

- 用户直接在渲染出的 SDF 表面上画一条 stroke
- stroke 可以配合自定义 brush profile
- 还可以沿 stroke 方向做强度调制

### 2. Tubular sampling

- 以 stroke 为中心建立局部 `u-v-n` 坐标系
- 在 stroke 周围的 tubular region 里采样点
- 用这些样本去约束 neural SDF 的局部 fine-tuning

这是论文最重要的实现点，因为它把“画一条线”转成了稳定的 3D 局部监督。

### 3. 实时微调 SDF

- 用局部采样点微调 MLP
- 让零水平集朝目标刷形状移动
- 同时尽量保持周围未编辑区域平滑连续

---

## 关键实验结论

### 编辑速度

论文把 tubular sampling 和 3DNS 的 point-based sampling 做了对比：

- 在多组采样规模下，INST-Sculpt 都更快
- 相对 3DNS 可达到约 `16x` 速度提升

这个加速很重要，因为交互式工具能不能用，首先看延迟是否够低。

### 几何质量

论文用 Chamfer Distance 比较了：

- 直接高分辨率 mesh 编辑
- 粗 mesh 编辑
- 3DNS pointwise editing
- INST-Sculpt

结果显示，INST-Sculpt 在局部编辑区域内的质量明显好于 3DNS，也更接近直接 mesh 编辑的理想目标。

### 表达能力

论文演示了很多不只是“鼓包/凹陷”的 brush 效果：

- 刻字
- 复杂雕纹
- 曲线调制
- 同时雕刻与外凸

这说明它不只是 demo 级局部修改，而是真正朝 sculpting workflow 靠近。

---

## 为什么它重要

INST-Sculpt 的贡献不是新的生成模型，而是把 neural SDF 从“适合表示”推进到“也适合交互编辑”。

它代表的是另一条路线：

- 不追求从文本直接改 3D
- 而是让隐式表示也能支持艺术家式操作

---

## 局限

- 主要是局部交互式几何雕刻，不处理语义级别生成编辑
- 依赖局部微调，虽然快很多，但仍不是完全零成本更新
- 更适合单物体 shape sculpting，不是完整资产级编辑系统

---

## 一句话总结

INST-Sculpt 的主要价值，是把 stroke-based digital sculpting 直接带到 neural SDF 表示里，通过 tubular sampling 实现低延迟、可控、保连续的局部雕刻编辑。
