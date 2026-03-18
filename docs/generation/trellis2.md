# TRELLIS 2

> *TRELLIS 2*

TRELLIS 已经通过 SLAT 把 3D latent 变成“有结构、可编辑”的表示；TRELLIS 2 则进一步追问：

> 如果底层几何仍然主要依赖 SDF / isosurface，3D 生成是否仍然停留在一个不够原生的表示层？

它给出的回答是 **O-Voxel**。

---

## 从 SLAT 到 O-Voxel

TRELLIS 的关键贡献是 SLAT：

- 稀疏空间位置负责结构
- 局部 latent 负责细节

这已经很强，但仍然存在几个问题：

- SDF 对开放表面不友好
- 对非流形结构支持不自然
- 几何和材质往往不是原生一体化
- 高分辨率 structured latent 成本仍然偏高

TRELLIS 2 的方向变化在于：

> 不只是让 latent “有结构”，而是要让 latent 更接近真正的 3D 资产结构本身。

---

## O-Voxel 的核心思想

O-Voxel 可以理解为一种更原生的结构化 3D 单元表示。

它和传统 SDF latent 的区别在于：

- 不再把几何完全等同为一个待抽壳的连续场
- 而是直接在结构化单元中编码更原生的几何/材质信息

这使它更自然地处理：

- **开放表面**
- **非流形结构**
- **带内部结构的对象**
- **原生 PBR 材质**

这一步非常关键，因为它实际上把表示研究从“更容易训练的 latent”推进到了“更像真实 3D asset 的 latent”。

---

## SC-VAE

TRELLIS 2 同时提出了 **SC-VAE**，目的是进一步提高压缩效率。

从研究线看，SC-VAE 解决的是一个非常现实的问题：

- structured latent 通常比 set-based latent 更重
- 如果没有更强的压缩方式，表示越原生，成本越高

SC-VAE 的意义就是：

> 在保持 native structured representation 能力的同时，把压缩率继续做上去。

所以 TRELLIS 2 不只是“表示更强”，而是试图同时保住：

- 原生性
- 可扩展性
- 高保真解码能力

---

## 它在整条文献线里的位置

可以把这条演化线简单理解为：

- `3DShape2VecSet`：先有紧凑 latent set
- `TRELLIS / SLAT`：再让 latent 结构化、可编辑
- **TRELLIS 2 / O-Voxel**：进一步让 latent 更接近 native 3D asset

所以 TRELLIS 2 的意义不是“TRELLIS 的小升级”，而是路线升级。

---

## 为什么它重要

TRELLIS 2 对后续研究最重要的启发是：

- 如果最终目标是高质量 3D 资产，而不是只是一个中间几何场
- 那么内部表示就不能只为了训练方便设计
- 它还必须足够接近真实 3D 数据结构

这会直接影响：

- 生成质量
- 编辑可控性
- 材质一致性
- 最终输出 mesh / asset 的工业可用性

---

## 与其他方法的关系

### 相比 TRELLIS v1

- v1 更强调“结构化 latent”
- v2 更强调“native structured latent”

### 相比 LATTICE / VoxSet

- VoxSet 更像 compactness 与 localizability 的折中
- O-Voxel 更像往原生 3D 资产单元推进

### 相比 BPT / FACE

- O-Voxel 仍然是在 latent representation 路线上推进
- BPT / FACE 则在 mesh-native token space 上推进

两者都在追求“更原生”，但方向不同。

---

## 一句话总结

TRELLIS 2 的核心价值，是把 3D latent 表示从“有结构的中间表示”进一步推进到“更原生的 3D 资产表示”，而 O-Voxel + SC-VAE 代表的正是这一方向：不仅要可生成、可压缩，还要更自然地支持开放表面、非流形结构和原生材质。
