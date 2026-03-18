# 3D Latent Space Representations

`3D Latent Space Representations` 不是 3D 生成里的一个边角话题，而是当前高质量 3D 生成系统最核心的基础模块之一。

如果说 2D 生成模型已经把问题收敛到“更大的模型、更好的数据、更稳定的训练范式”，那么 3D 生成到今天仍然没有完全统一，最重要的原因之一就是：

> 3D 数据到底应该先被压缩成什么样的隐空间，模型才能既学得动，又生成得好，还能最终稳定地输出 mesh？

这也是为什么这条线值得长期、系统、持续地整理。它不只是“表示方法综述”，而是后续研究设计的出发点。

---

## Why It Matters

一个 3D latent representation，至少同时影响四件事：

1. **训练是否可扩展**：token 太多，Transformer/DiT/Flow 就很难 scale。
2. **几何是否保真**：latent 太弱，细节、锐边、薄结构、拓扑都会丢。
3. **是否易于控制和编辑**：没有显式位置结构，就很难做局部控制。
4. **最终 mesh 质量**：如果 latent 本身与 mesh 目标脱节，最后一步 mesh extraction 往往就是瓶颈。

所以 3D latent 不是中间技术细节，而是连接：

- 3D 数据
- 生成模型
- 编辑模型
- 最终 mesh 输出

之间的真正桥梁。

---

## A Practical Taxonomy

当前和 mesh generation 关系最紧密的 3D latent / token 表征，大致可以分为 6 类：

| 类别 | 代表工作 | 核心思想 | 最强优点 | 主要问题 |
|:-----|:---------|:---------|:---------|:---------|
| **Latent Set / VecSet** | 3DShape2VecSet | 把 3D 形状压成固定长度 latent vectors | 紧凑、Transformer 友好 | 空间位置语义偏弱 |
| **Structured Sparse Latent** | TRELLIS / SLAT | 稀疏空间结构 + 每位置局部 latent | 有显式结构，适合局部编辑 | token 较重 |
| **Native Structured 3D Latent** | TRELLIS 2 / O-Voxel | 更接近原生 3D 资产单元的 latent | 支持开放表面、非流形、PBR | 体系复杂 |
| **Semi-structured Localizable Latent** | LATTICE / VoxSet | 用粗 voxel anchor 增强 set-based latent 的位置语义 | 兼顾紧凑性和可定位性 | 仍非最终 mesh 表示 |
| **Sparse Volumetric High-res Latent** | TripoSG / SparseFlex / Direct3D-S2 | 在 sparse SDF / sparse isosurface / sparse attention 上做高分辨率 scaling | 高保真、工程上可扩展 | 仍要走 field / extraction 路线 |
| **Mesh-native Token Space** | MeshAnything V2 / BPT / FACE | 直接把 mesh 序列本身作为建模空间 | 原生面向 mesh 输出 | 序列建模难度高 |

这里最后一类严格说不完全属于“latent space”传统意义上的 VAE latent，但它实际上在回答同一个问题：

> 如果目标是 mesh，模型到底应该在哪个内部表示空间里生成？

所以在研究梳理里，这条路线不能被排除在外。

---

## Evolution Line

### 1. VecSet: make 3D latent Transformer-friendly

代表：**3DShape2VecSet**

这篇工作的关键意义，是第一次很清楚地证明：

- 3D 形状可以不依赖显式 voxel / grid；
- 可以被编码为固定长度 latent set；
- 再通过 cross-attention 解码成 neural field。

它的重要性不只是方法本身，而是建立了一个范式：

> 3D 生成也可以像 2D latent diffusion 一样，先压缩到一个干净、紧凑、规则的 latent 空间里再建模。

但 VecSet 很快暴露出一个限制：它虽然紧凑，但 token 的空间语义不够明确。也就是说，模型知道“这些 token 对应这个 shape”，却不够知道“哪个 token 更接近空间里的哪一块”。

### 2. SLAT: make latent structured and editable

代表：**TRELLIS**

TRELLIS 提出 **SLAT (Structured LATent)**，本质是：

- 用稀疏体素位置 `{p_i}` 定义结构；
- 用局部 latent `{z_i}` 填充每个位置的细节。

相比 VecSet，SLAT 最重要的提升不是单纯“更局部”，而是：

- **token 有空间锚点了**；
- **结构和细节可以解耦建模**；
- **局部编辑、局部重绘有了天然落脚点**。

这也是为什么 TRELLIS 不只是一个生成模型，更是后来很多编辑方法的基础骨干。

### 3. O-Voxel: from structured latent to native 3D latent

代表：**TRELLIS 2**

TRELLIS 已经把 latent 锚定到了空间，但底层几何仍然大量依赖 SDF / isosurface 逻辑。TRELLIS 2 进一步指出：

- SDF 对开放表面不友好；
- 对非流形结构不友好；
- 对几何和材质的统一建模不够原生。

因此它提出 **O-Voxel**，目标是把 latent representation 从“结构化中间表示”推进到“更原生的 3D 资产表示”。

这是一个很重要的方向变化：

> 研究重点不再只是让 latent 更适合生成，而是让 latent 本身更像真实的 3D 数据结构。

### 4. VoxSet: localizable code over pure local/global debate

代表：**LATTICE / VoxSet**

LATTICE 这篇论文最重要的观点之一是：

> 真正关键的不是 local vs global，而是 latent code 是否 **localizable**。

这句话很值得单独记住。

因为它指出：

- VecSet 的问题不只是“太全局”；
- sparse voxel 的优势也不只是“更局部”；
- 关键是 token 能否在生成时被明确放置到某个空间位置上。

VoxSet 的做法就是：

- 保留 set-based latent 的紧凑性；
- 但用 coarse voxel anchors 赋予它位置语义；
- 最终实现更强的 test-time scaling 和更好的细节建模。

### 5. Sparse volumetric scaling: make high resolution actually trainable

代表：**TripoSG / SparseFlex / Direct3D-S2**

这一组工作的重要性在于，它们不只是讨论“表示该怎么设计”，而是在回答：

> 这个表示到底能不能在高分辨率、大模型、大数据条件下真正训起来？

- **TripoSG** 更像系统化 scaling：高质量数据构建 + SDF VAE + Rectified Flow Transformer + MoE。
- **SparseFlex** 更像高分辨率 shape modeling 基础设施：稀疏可微 isosurface + frustum-aware sectional voxel training + arbitrary topology。
- **Direct3D-S2** 更像 sparse SDF VAE / sparse attention 的工程推进。

这条线的价值在于，它让“高分辨率 3D latent generation”从概念走向可训练系统。

### 6. Mesh-native route: generate in mesh space directly

代表：**MeshAnything V2 / BPT / FACE**

这条路线的出发点和前面不一样。前面几类方法大多在问：

> 用什么 latent field / structured latent 才适合生成？

而这条路线问的是：

> 如果最终目标就是高质量 mesh，为什么不直接在 mesh 自身的 token 空间里建模？

- **MeshAnything V2**：AMT，让相邻面尽可能共享边和顶点上下文。
- **BPT**：Blocked and Patchified Tokenization，把 mesh 序列压到 `0.26`。
- **FACE**：one-face-one-token，把建模单元提升到 triangle face，压到 `0.11`。

这条线非常值得长期关注，因为它代表另一种“原生化”方向：不是让 latent 更像 3D，而是直接让生成空间就是 mesh。

---

## What Really Matters

到目前为止，这条文献线里最值得反复咀嚼的，不是“哪篇最好”，而是下面几个判断。

### 1. Compactness is necessary, but not sufficient

只靠 token 少并不够。token 足够少，只能说明训练可能更容易；并不意味着模型就知道细节该放在哪、如何局部控制、如何支持编辑。

### 2. Localizability is the real turning point

从 VecSet 到 SLAT/VoxSet，一个真正的转折点是：

> latent token 开始拥有明确空间锚点。

一旦有了这一点，后续的局部编辑、repainting、mask conditioning、test-time scaling 才开始变得自然。

### 3. Native-ness will likely matter more and more

无论是 O-Voxel 还是 FACE，其实都在做同一件事：

> 把内部表示往“真实 3D 结构”那边推，而不是停留在方便训练的抽象中间层。

这很可能是未来几年一个持续主线。

---

## A Working Comparison Framework

以后继续读这一方向的论文时，可以统一用下面这 6 个问题判断它的位置：

1. **表示单元是什么？** set、voxel、face、patch 还是 field？
2. **token 是否带显式位置语义？**
3. **是更偏 geometry，还是 geometry + appearance 一体化？**
4. **是否支持 open surfaces / non-manifold / interior structures？**
5. **最终输出 mesh 需要多重后处理？**
6. **它更适合 generation、editing，还是 reconstruction backbone？**

如果后面系统整理文献，可以直接按这 6 列做一张持续维护的对比表。

---

## Current Reading Spine

目前最值得作为主线持续追踪的论文，可以先按这个顺序：

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

## Research Implication

对后续研究来说，这条线最重要的启发不是“选哪篇复现”，而是：

- 如果目标是**高质量 image-to-3D mesh generation**，要重点看 latent 是否足够 localizable；
- 如果目标是**native 3D editing**，要重点看 latent 是否天然支持局部替换和重绘；
- 如果目标是**最终产物就是高质量 mesh**，就不能只看 latent field，还必须把 mesh-native token 空间也纳入视野。

也就是说，`3D Latent Space Representations` 本身就应该被视为一个长期维护的研究专题，而不是一篇综述就结束的话题。
