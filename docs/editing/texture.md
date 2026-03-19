---
comments: true
---

# TEXTure

> *TEXTure: Text-Guided Texturing of 3D Shapes*

TEXTure 聚焦的不是改几何，而是给已有 3D mesh 生成高质量纹理。它的关键贡献是：直接利用 depth-conditioned diffusion 逐视角给 mesh 上色，同时通过 trimap 机制减少不同视角之间的纹理接缝。

---

## 核心问题

给 3D 对象生成纹理比 2D 图像生成难，难点主要有两类：

- 不同视角生成结果容易不一致
- 重新投射回 3D 表面后容易出现 seams

TEXTure 要解决的是：

> 怎样直接借助强大的 2D diffusion，把单视图高质量纹理稳定地扩展成整物体一致纹理。

---

## 方法框架

### 1. 逐视角渲染 + diffusion 绘制

- 先从某个视角渲染 mesh 的深度图
- 用 depth-to-image diffusion 根据文本生成该视角纹理
- 再把该视图结果投射回 mesh 顶点或 atlas

### 2. 动态 trimap

每一步把当前渲染图分成三类区域：

- `keep`：已经画好且不应再改的区域
- `refine`：已经出现过，但当前视角更好，值得重绘的区域
- `generate`：第一次出现，需要新生成的区域

这就是 TEXTure 最关键的设计，因为它直接决定了多视角之间能否保持连续。

### 3. 改造 diffusion sampling

- 对 `keep` 区域尽量冻结
- 对 `generate` 区域同时结合 depth-guided 与 mask-guided diffusion
- 对 `refine` 区域在保留已有纹理的基础上重新细化

这样做的结果，是把“每个视角独立画一张图”改成“在已有 3D 纹理状态上逐步补全”。

---

## 能做哪些任务

TEXTure 不只支持 text-guided texturing，还扩展到了：

- texture transfer
- 纹理编辑
- scribble-based texture editing
- 从少量图像提取语义纹理再迁移

因此它本质上是一个通用 3D 纹理编辑框架，而不只是文本贴图生成器。

---

## 关键实验结论

### 用户研究

论文用户研究里，TEXTure 相比 Text2Mesh 和 Latent-Paint 有更好的整体质量和文本一致性：

- `Overall Quality = 3.93`
- `Text Fidelity = 4.44`
- 推理时间约 `5 min`

对比里：

- `Text2Mesh`: 质量 `2.57`，文本一致性 `3.62`，约 `32 min`
- `Latent-Paint`: 质量 `2.95`，文本一致性 `4.01`，约 `46 min`

这说明 TEXTure 在质量、文本对齐和速度上都更均衡。

### 视觉效果

- 相比纯 CLIP 引导方法，TEXTure 的全局一致性明显更强
- 在人物、动物、木雕、照片风格等文本提示上都能生成较自然纹理
- 对非定向或复杂拓扑模型也能工作，比如 Klein bottle

---

## 为什么它重要

TEXTure 的重要性在于，它把 2D diffusion 真正转成了一个可用的 3D 纹理工作流：

- 不是只在单视图上看起来对
- 而是逐步把局部正确的 2D 纹理积累成整体一致的 3D 纹理

在后续很多 3D 编辑工作里，这类“先渲染、再 diffusion、再投回 3D”的思路都能看到它的影响。

---

## 局限

- 主要聚焦纹理，不处理几何结构变化
- 仍然依赖逐视角迭代和投射，复杂场景下速度不算快
- 一致性虽然明显优于逐视角独立生成，但仍受视角规划和投射误差影响

---

## 一句话总结

TEXTure 的主要贡献，是用 `depth-conditioned diffusion + dynamic trimap` 把单视图强纹理生成能力稳定扩展到整物体表面，从而显著缓解 3D 纹理生成中的多视角接缝问题。
