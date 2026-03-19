---
comments: true
---

# AnchorFlow

> *AnchorFlow: Training-Free 3D Editing via Latent Anchor-Aligned Flows*

AnchorFlow 的核心观点是：训练自由的 3D 编辑之所以常常编辑不够或几何不稳，不只是因为 guidance 弱，而是因为每个 timestep 使用的 latent anchor 不一致，导致编辑方向在扩散过程中不断漂移。

---

## 核心问题

很多 inversion-free 3D 编辑方法隐式依赖每个 timestep 的随机噪声作为 anchor：

- anchor 不稳定，flow 方向会漂
- 漂了以后，语义改动会互相抵消
- 最终要么改得不够，要么几何坏掉

AnchorFlow 要解决的是：

> 在 training-free 的前提下，为 source 和 target trajectory 建立一个更稳定的共享 latent anchor。

---

## 方法框架

### 1. Global latent anchor

- 不再把每个 timestep 的随机噪声当作独立 anchor
- 而是让 source / target trajectory 共享更稳定的全局 latent reference

### 2. Anchor-alignment loss

- 用一个 relaxed 的对齐目标，让两条 trajectory 的单步 inversion 保持接近
- 重点不是完全强绑死，而是在可编辑性和稳定性之间留出空间

### 3. Anchor-aligned update rule

- 在编辑过程中按对齐后的 flow 方向逐步更新
- 保证语义编辑足够强，同时尽量保住几何结构

---

## 它支持哪些编辑

论文里覆盖了几类常见 3D 编辑：

- `action change`
- `object addition`
- `object replacement`
- `style change`

并且方法是 `mask-free` 的，不依赖额外编辑掩码。

---

## 关键实验结论

### Eval3DEdit 基准

AnchorFlow 在 Eval3DEdit 上取得了最好的一组综合结果：

- `Overall CLIP_img = 0.7173`
- `Overall CLIP_txt = 0.4866`

相较 inversion-free editing，平均提升约：

- `CLIP_img +0.0067`
- `CLIP_txt +0.0161`

这说明它在 identity preservation 和 semantic modification 之间取得了更好的平衡。

### 定性结果

- 相比 direct editing，它保留 identity 更好
- 相比 inversion-free editing，它更少出现 under-editing 和局部几何破坏
- 对 rigid 和 non-rigid 编辑都能工作

### 参数分析

论文也明确指出：

- 编辑步数和编辑强度越大，语义变化更强
- 但 identity preservation 会相应下降
- AnchorFlow 提供的是更稳定的可调平衡，而不是完全免调参

---

## 为什么它重要

AnchorFlow 的主要贡献不在表示，而在训练自由编辑算法本身：

- 它把问题从“怎么加更强 guidance”转成“怎么稳定 latent anchor”
- 给 inversion-free / flow-based 编辑提供了更清晰的解释框架

这对后续很多 training-free 3D editing 都有参考价值。

---

## 局限

- 虽然是 mask-free，但仍然属于训练自由编辑，结果依赖底层 flow model 本身能力
- 参数选择仍会影响编辑强度和保留度的平衡
- 论文主要关注编辑算法，不解决底层 3D 生成骨干的表示局限

---

## 一句话总结

AnchorFlow 的主要价值，是通过 `latent anchor consistency` 稳定 training-free 3D 编辑过程，让语义修改更充分，同时减少几何失真和 under-editing。
