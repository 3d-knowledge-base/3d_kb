---
comments: true
---

# ShapeLLM-Omni

> *ShapeLLM-Omni: A Native Multimodal LLM for 3D Generation and Understanding*

ShapeLLM-Omni 不是传统意义上的单任务编辑方法，而是把 3D 生成、理解、描述和编辑都放进一个统一的 autoregressive 多模态大模型里。它的核心是：把 3D mesh 也离散成 token，使 3D 可以像语言一样参与 next-token prediction。

---

## 核心问题

很多 3D 方法只做单任务：

- 要么 text-to-3D
- 要么 image-to-3D
- 要么 3D understanding
- 要么 3D editing

ShapeLLM-Omni 想做的是：

> 能不能把 3D 作为一种原生模态，直接接进多模态 LLM，让模型在文本、图像、3D 之间自由切换。

---

## 方法框架

### 1. 3D VQVAE

- 先训练一个 3D VQVAE
- 把 mesh 编码成离散 3D token
- 这样 3D 就能像词表 token 一样被 LLM 处理

### 2. 统一 next-token prediction

- 模型继承 Qwen2.5-VL 的图文能力
- 再加入 3D token 词表
- 统一做自回归生成

因此它支持：

- text-to-3D
- image-to-3D
- 3D-to-caption
- 3D editing

### 3. 3D-Alpaca 数据集

论文构建了一个较大的 3D 对话训练集：

- text/image to 3D
- 3D understanding
- 3D editing

总规模约 `2.56M` 样本、`3.46B` tokens。

---

## 编辑相关部分

ShapeLLM-Omni 的 editing 不是单独外挂一个编辑器，而是统一写进语言模型训练中：

- 先定义可执行的编辑 prompt
- 再为资产生成前后编辑图像对
- 用 Trellis 等方法把这些图像对重建成 3D before/after 对
- 最后把它们写成对话格式训练

论文里编辑数据大约有 `62k` 对，最终构成 3D-Alpaca 的一部分。

这意味着它更像“会对话的 3D agent 原型”，而不是最强的单任务编辑器。

---

## 关键实验结论

### 语言能力

引入 3D 能力后，ShapeLLM-Omni 仍保持了接近原 Qwen2.5-VL 的一般语言能力：

- `MMLU = 63.9`
- `PIQA = 78.6`
- `GSM8K = 55.1`

说明加入 3D token 后，模型并没有完全牺牲通用对话能力。

### 3D 生成

- 在 text-to-3D / image-to-3D 上优于多种 baseline
- 但整体仍弱于专门为单任务生成优化的 `TRELLIS`

论文也明确解释了原因：

- 它是一个 all-in-one 模型
- 同时学生成、理解、编辑、对话
- 自然会在单项极限性能上让位于专门系统

### 3D 理解

- 在 3D caption 等理解任务上表现较强
- 仅次于专门为 3D understanding 训练的单任务模型

这验证了它的统一多模态路线是可行的。

---

## 为什么它重要

ShapeLLM-Omni 的价值主要不在单项 SOTA，而在方向上：

- 它证明 3D token 可以真正并入原生多模态 LLM
- 3D 编辑不一定非要做成独立管线，也可以做成对话式能力的一部分

对知识库来说，它更像“3D-native AI”路线的代表，而不是单纯编辑 benchmark 选手。

---

## 局限

- 单任务性能仍不如专门的 3D 生成或编辑模型
- 离散 autoregressive 生成方式在质量上和 flow / diffusion 路线仍有差距
- 编辑能力目前更像基础能力验证，还不是工业级高保真编辑系统

---

## 一句话总结

ShapeLLM-Omni 的主要意义，是把 3D generation、understanding、captioning 和 editing 统一进一个原生多模态 LLM 框架里，让 3D 真正成为可对话、可生成、可操作的离散模态。
