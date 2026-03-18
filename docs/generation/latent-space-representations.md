# 3D Latent Space Representations

3D Latent Space Representations 是当前高质量 3D 生成系统主要的基础模块之一。

2D 生成模型已逐步收敛到"更大的模型、更好的数据、更稳定的训练范式"，而 3D 生成至今尚未完全统一，其中一个关键原因在于：

> 3D 数据应当先被压缩成什么样的隐空间，才能使模型既可训练，又能生成高质量结果，并最终稳定输出 mesh？

---

## Why It Matters

一个 3D latent representation 至少同时影响四个方面：

1. **训练是否可扩展**：token 太多，Transformer/DiT/Flow 难以 scale。
2. **几何是否保真**：latent 太弱，细节、锐边、薄结构、拓扑都会丢失。
3. **是否易于控制和编辑**：缺乏显式位置结构，局部控制将难以实现。
4. **最终 mesh 质量**：如果 latent 本身与 mesh 目标脱节，mesh extraction 往往成为瓶颈。

因此 3D latent 并非中间技术细节，而是连接 3D 数据、生成模型、编辑模型和最终 mesh 输出之间的核心桥梁。

---

## A Practical Taxonomy

当前与 mesh generation 关系最紧密的 3D latent / token 表征，大致可分为 6 类：

| 类别 | 代表工作 | 核心思想 | 主要优点 | 主要问题 |
|:-----|:---------|:---------|:---------|:---------|
| **Latent Set / VecSet** | 3DShape2VecSet | 把 3D 形状压成固定长度 latent vectors | 紧凑、Transformer 友好 | 空间位置语义偏弱 |
| **Structured Sparse Latent** | TRELLIS / SLAT | 稀疏空间结构 + 每位置局部 latent | 有显式结构，适合局部编辑 | token 较重 |
| **Native Structured 3D Latent** | TRELLIS 2 / O-Voxel | 更接近原生 3D 资产单元的 latent | 支持开放表面、非流形、PBR | 体系复杂 |
| **Semi-structured Localizable Latent** | LATTICE / VoxSet | 用粗 voxel anchor 增强 set-based latent 的位置语义 | 兼顾紧凑性和可定位性 | 仍非最终 mesh 表示 |
| **Sparse Volumetric High-res Latent** | TripoSG / SparseFlex / Direct3D-S2 | 在 sparse SDF / sparse isosurface / sparse attention 上做高分辨率 scaling | 高保真、工程上可扩展 | 仍需走 field / extraction 路线 |
| **Mesh-native Token Space** | MeshAnything V2 / BPT / FACE | 直接把 mesh 序列本身作为建模空间 | 原生面向 mesh 输出 | 序列建模难度高 |

最后一类严格来说不完全属于"latent space"传统意义上的 VAE latent，但它实际上在回答同一个问题：

> 如果目标是 mesh，模型应当在哪个内部表示空间里生成？

因此在表征研究的完整视角下，这条路线值得关注。

---

## Evolution Line

### 1. VecSet: make 3D latent Transformer-friendly

代表：**3DShape2VecSet**

该工作率先清晰地证明了：

- 3D 形状可以不依赖显式 voxel / grid；
- 可以被编码为固定长度 latent set；
- 再通过 cross-attention 解码成 neural field。

其重要性不仅在于方法本身，更在于建立了一个范式：

> 3D 生成可以像 2D latent diffusion 一样，先压缩到一个紧凑、规则的 latent 空间再建模。

但 VecSet 的局限性在于：虽然紧凑，但 **token 的空间语义不够明确**——模型知道"这些 token 对应这个 shape"，却难以确定"哪个 token 更接近空间里的哪一块"。

### 2. SLAT: make latent structured and editable

代表：**TRELLIS**

TRELLIS 提出 **SLAT (Structured LATent)**：

- 用稀疏体素位置 `{p_i}` 定义结构；
- 用局部 latent `{z_i}` 填充每个位置的细节。

相比 VecSet，SLAT 的关键提升在于：

- **token 具有空间锚点**；
- **结构和细节可以解耦建模**；
- **局部编辑、局部重绘有了天然的操作基础**。

因此 TRELLIS 不仅是一个生成模型，也成为后续多种编辑方法的基础骨干。

### 3. O-Voxel: from structured latent to native 3D latent

代表：**TRELLIS 2**

TRELLIS 已经把 latent 锚定到空间，但底层几何仍大量依赖 SDF / isosurface 逻辑。TRELLIS 2 进一步指出：

- SDF 对开放表面不友好；
- 对非流形结构支持不自然；
- 对几何和材质的统一建模不够原生。

因此它提出 **O-Voxel**，目标是把 latent representation 从"结构化中间表示"推进到"更原生的 3D 资产表示"。

这标志着一个重要的方向变化：

> 研究重点不再仅是让 latent 更适合生成，而是让 latent 本身更像真实的 3D 数据结构。

### 4. VoxSet: localizable code over pure local/global debate

代表：**LATTICE / VoxSet**

LATTICE 提出的一个核心观点是：

> 关键的不是 local vs global，而是 latent code 是否 **localizable**。

这一判断的含义是：

- VecSet 的问题不仅是"太全局"；
- sparse voxel 的优势也不仅是"更局部"；
- 关键是 token 能否在生成时被明确放置到某个空间位置上。

VoxSet 的做法是：

- 保留 set-based latent 的紧凑性；
- 用 coarse voxel anchors 赋予位置语义；
- 最终实现更强的 test-time scaling 和更好的细节建模。

### 5. Sparse volumetric scaling: make high resolution actually trainable

代表：**TripoSG / SparseFlex / Direct3D-S2**

这一组工作的重要性在于，它们不只讨论"表示该怎么设计"，而是在回答：

> 这个表示能否在高分辨率、大模型、大数据条件下训练起来？

- **TripoSG** 代表系统化 scaling 路线：高质量数据构建 + SDF VAE + Rectified Flow Transformer + MoE。
- **SparseFlex** 聚焦高分辨率 shape modeling 基础设施：稀疏可微 isosurface + frustum-aware sectional voxel training + arbitrary topology。
- **Direct3D-S2** 推进 sparse SDF VAE / sparse attention 的工程可扩展性。

这条路线使"高分辨率 3D latent generation"从概念走向可训练系统。

### 6. Mesh-native route: generate in mesh space directly

代表：**MeshAnything V2 / BPT / FACE**

与前面几类方法不同，这条路线的出发点是：

> 如果最终目标就是高质量 mesh，为什么不直接在 mesh 自身的 token 空间里建模？

- **MeshAnything V2**：AMT，让相邻面尽可能共享边和顶点上下文。
- **BPT**：Blocked and Patchified Tokenization，把 mesh 序列压到 `0.26`。
- **FACE**：one-face-one-token，把建模单元提升到 triangle face，压到 `0.11`。

这条路线代表另一种"原生化"方向：不是让 latent 更像 3D，而是直接让生成空间就是 mesh。

---

## Key Observations

### 1. Compactness is necessary, but not sufficient

token 数量少仅意味着训练可能更高效，并不意味着模型能确定细节的空间位置、支持局部控制或实现编辑。

### 2. Localizability is the real turning point

从 VecSet 到 SLAT/VoxSet，一个转折点是：

> latent token 开始拥有明确空间锚点。

一旦具备这一属性，后续的局部编辑、repainting、mask conditioning、test-time scaling 才具有自然的操作基础。

### 3. Native-ness will likely matter more and more

无论是 O-Voxel 还是 FACE，其本质都在做同一件事：

> 把内部表示往"真实 3D 结构"方向推进，而不是停留在便于训练的抽象中间层。

这一趋势在未来几年内大概率将持续加强。

### 4. Three-axis positioning

当前 mesh generation 中的 3D latent / token 表征，需要在三个维度之间做取舍：

1. **Compactness**：token 足够少，训练可扩展。
2. **Localizability**：token 能对应明确的空间位置。
3. **Native-ness**：表示尽量接近真实 3D 资产，减少对后处理转换的依赖。

VecSet 更偏 compactness，SLAT 更偏 localizability，O-Voxel 更偏 native-ness，VoxSet 是三者的折中方案，而 BPT / FACE 代表的是 mesh-native 路线的另一端。

---

## A Working Comparison Framework

评估这一方向的论文时，可以统一使用以下 6 个维度进行定位：

1. **表示单元是什么？** set、voxel、face、patch 还是 field？
2. **token 是否带显式位置语义？**
3. **是更偏 geometry，还是 geometry + appearance 一体化？**
4. **是否支持 open surfaces / non-manifold / interior structures？**
5. **最终输出 mesh 需要多少后处理？**
6. **它更适合 generation、editing，还是 reconstruction backbone？**

---

## Core Paper Sequence

理解这一方向的核心论文，建议按以下顺序阅读：

1. **3DShape2VecSet** — VecSet 起点
2. **TRELLIS** — SLAT，structured latent 的关键节点
3. **TRELLIS 2** — O-Voxel，native structured latent
4. **LATTICE / VoxSet** — semi-structured + localizable code
5. **TripoSG** — 大规模 RF + SDF VAE 的系统化 scaling
6. **SparseFlex** — 高分辨率 arbitrary-topology sparse isosurface
7. **Direct3D-S2** — Sparse SDF VAE + spatial sparse attention
8. **MeshAnything V2** — AMT，mesh-native tokenization
9. **BPT** — patch + block 压缩
10. **FACE** — one-face-one-token

---

## Research Implications

从研究视角看，这条文献线提供了以下指导：

- 以**高质量 image-to-3D mesh generation** 为目标时，应重点关注 latent 是否具有足够的 localizability；
- 以 **native 3D editing** 为目标时，应重点关注 latent 是否天然支持局部替换和重绘；
- 以**高质量 mesh 输出**为最终产物时，不能仅关注 latent field，还需将 mesh-native token 空间纳入考量。

3D Latent Space Representations 构成了一个具有独立研究价值的系统性方向，横跨表征设计、生成建模与下游应用多个层面。
