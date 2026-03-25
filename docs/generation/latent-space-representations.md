---
comments: true
---

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

当前与 mesh generation 关系最紧密的 3D latent / token 表征，大致可分为 8 类：

| 类别 | 代表工作 | 核心思想 | 主要优点 | 主要问题 |
|:-----|:---------|:---------|:---------|:---------|
| **Latent Set / VecSet** | 3DShape2VecSet | 把 3D 形状压成固定长度 latent vectors | 紧凑、Transformer 友好 | 空间位置语义偏弱 |
| **Structured Sparse Latent** | TRELLIS / SLAT | 稀疏空间结构 + 每位置局部 latent | 有显式结构，适合局部编辑 | token 较重 |
| **Native Structured 3D Latent** | TRELLIS 2 / O-Voxel | 更接近原生 3D 资产单元的 latent | 支持开放表面、非流形、PBR | 体系复杂 |
| **Semi-structured Localizable Latent** | LATTICE / VoxSet | 用粗 voxel anchor 增强 set-based latent 的位置语义 | 兼顾紧凑性和可定位性 | 仍非最终 mesh 表示 |
| **VecSet-based Scaling** | TripoSG / Hunyuan3D 2.0 | 继承 VecSet latent，用大规模数据 + SDF VAE + RF/DiT 做系统化 scaling | 系统化、可工业部署 | 隐空间仍为 unstructured set |
| **Sparse Volumetric High-res Latent** | SparseFlex / Direct3D-S2 | 在 sparse isosurface / sparse attention 上做高分辨率 scaling | 高保真、工程上可扩展 | 仍需走 field / extraction 路线 |
| **Octree / Modality-consistent Latent** | [OctFusion](octfusion.md) / [Sparc3D](sparc3d.md) | 八叉树自适应或纯 3D 稀疏卷积，消除 2D-3D 模态转换 | 模态一致、自适应分辨率 | 实现复杂，仍依赖 isosurface |
| **Extreme Compression Tokenization** | [VAT](vat.md) | 将 3D shape 压缩至 256 个 token（250x-2000x 压缩） | 极端紧凑，可对接 LLM | 空间局部性丢失 |
| **Mesh-native Token Space** | BPT / FACE / [Nautilus](nautilus.md) / [MeshRipple](meshripple.md) / [QuadGPT](quadgpt.md) / [TSSR](tssr.md) | 直接把 mesh 序列本身作为建模空间 | 原生面向 mesh 输出 | 序列建模难度高 |

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

### 5. VecSet-based scaling: TripoSG 与 Hunyuan3D 2.0

代表：**TripoSG / Hunyuan3D 2.0**

TripoSG 和 Hunyuan3D 2.0 本质上继承了 3DShape2VecSet 的 latent set 路线——VAE 输出为 1D latent vector set（TripoSG: L × C, L ∈ {512, 2048, 4096}, C = 64），而非 sparse 3D voxel grid。它们的重点不在隐空间结构创新，而在大规模系统化 scaling：高质量数据构建、SDF VAE、Rectified Flow / DiT、MoE 等工程手段。SDF field 的查询分辨率可达 512^3，但 latent 本身是 unstructured token set。

### 6. Sparse volumetric scaling: make high resolution actually trainable

代表：**SparseFlex / Direct3D-S2**

与 TripoSG 不同，这一组工作的 latent 具有显式 3D 空间结构。它们不只讨论"表示该怎么设计"，而是在回答：

> 这个表示能否在高分辨率、大模型、大数据条件下训练起来？

- **SparseFlex** 聚焦高分辨率 shape modeling 基础设施：稀疏可微 isosurface + frustum-aware sectional voxel training + arbitrary topology。
- **Direct3D-S2** 推进 sparse SDF VAE / sparse attention 的工程可扩展性。

这条路线使"高分辨率 3D latent generation"从概念走向可训练系统。

### 7. Octree & modality-consistent latent: structural efficiency and pipeline purity

代表：**[OctFusion](octfusion.md)、[Sparc3D](sparc3d.md)**

这一组工作的共同关注点是：**隐空间的结构效率和管线纯度**。

**[OctFusion](octfusion.md)**（SGP 2025）的核心设计是八叉树隐空间：

- 空间用八叉树自适应划分，表面附近用高分辨率节点，远处用低分辨率节点
- 在此之上设计了统一的多尺度 U-Net 扩散模型，权重和计算在不同 octree 层级间共享
- 避免了 cascaded diffusion（多模型级联）的复杂性
- 输出保证连续流形 mesh，单卡 2.5 秒生成

OctFusion 代表的趋势是：**latent 的空间结构不需要是均匀的**。八叉树让模型把算力集中在表面附近，是一种更合理的计算资源分配方式。

**[Sparc3D](sparc3d.md)** 的关注点更本质——**模态一致性（modality consistency）**：

- 指出现有 VAE 普遍用 2D 渲染损失做监督，导致 3D → 2D → 3D 的信息路径存在模态不匹配
- 提出 Sparcubes（稀疏可变形 MC 表示）+ Sparconv-VAE（纯 3D 稀疏卷积 VAE）
- 整个管线中没有 2D → 3D 或 3D → 2D 的模态转换：输入是 3D，中间是 3D，输出是 3D
- 在 1024^3 分辨率下实现了高保真重建

Sparc3D 系统性地指出的这个问题——VAE 训练时的模态不匹配——是此前被忽视但实际影响很大的因素。预计后续更多方法会采用纯 3D 原生训练管线。

### 8. Extreme compression: make 3D fit into LLM context

代表：**[VAT](vat.md)**

VAT 走了一条与上述所有方法不同的路：**极端压缩**。

- 用 in-context transformer 将大量无序 3D 特征压缩到少量 learnable query token 中
- 映射到高斯分布后做残差量化
- 不同尺度的 token 从同一高斯分布的不同子空间分配，构建隐式层级

关键数字：**250x 压缩比**（1MB mesh → 3.9KB，96% F-score），进一步到 256 个 int8 token 实现 2000x 压缩。

VAT 的意义在于：如果 3D shape 能用 256 个 token 表示，它就可以直接进入 GPT 类模型的 context window。这为 multimodal LLM 统一理解和生成 3D 打开了可能性。

代价是：极端压缩后空间局部性信息基本丢失，不利于局部编辑。

### 9. Mesh-native route: generate in mesh space directly

代表：**MeshAnything V2 / [BPT](bpt.md) / [FACE](face.md) / [Nautilus](nautilus.md) / [MeshRipple](meshripple.md) / [QuadGPT](quadgpt.md) / [TSSR](tssr.md)**

这条路线的出发点是：

> 如果最终目标就是高质量 mesh，为什么不直接在 mesh 自身的 token 空间里建模？

2024-2026 年间，这条路线经历了快速演进：

**序列压缩阶段**（2024）：

- **MeshAnything V2**：AMT，让相邻面尽可能共享边和顶点上下文。
- **BPT**：Blocked and Patchified Tokenization，把 mesh 序列压到 `0.26`。
- **FACE**：one-face-one-token，把建模单元提升到 triangle face，压到 `0.11`。

**拓扑感知序列化阶段**（2025）：

- **[Nautilus](nautilus.md)**（ICCV 2025）：利用流形 mesh 面间的拓扑邻接关系做序列压缩——BFS 遍历保证相邻面在序列中也相邻，共享顶点/边只编码一次。面数上限推到 5000。
- **[MeshRipple](meshripple.md)**：frontier-aware BFS tokenization + 前沿批量扩展 + sparse-attention global memory。生成过程像涟漪一样从种子面向外扩展，天然保证表面连通性。

**拓扑类型和范式扩展阶段**（2025-2026）：

- **[QuadGPT](quadgpt.md)**（ICLR 2026）：首个端到端四边形 mesh 自回归生成。统一的 tri/quad 混合 tokenization + tDPO（面向拓扑质量的 RL 微调）。
- **[TSSR](tssr.md)**：首个用离散扩散（而非 AR）做 mesh-native 生成的方法。拓扑雕刻 + 形状细化两阶段，10,000 面，1024^3 分辨率。全局并行推理对拓扑一致性天然更友好。

这条路线的进展速度很快，面数上限从 2024 年的 ~800（MeshGPT）增长到 2025 年的 10,000（TSSR），18 个月内增长了一个数量级。

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

### 4. Modality consistency is an underappreciated factor

Sparc3D 指出的模态不匹配问题——VAE 用 2D 渲染损失训练但目标是 3D 几何——是一个被低估但影响实际的因素。当 VAE 的训练监督与目标模态一致时（3D 输入、3D latent、3D 监督），重建保真度有可观提升。这个观察可能改变后续 VAE 设计的默认选择。

### 5. Structural adaptivity beats uniform grids

OctFusion 的八叉树和 SparseFlex 的稀疏体素都指向同一个方向：3D 空间中的信息分布是高度不均匀的（集中在表面附近），隐空间的结构应当反映这种不均匀性。均匀 dense grid 在高分辨率下既不经济也不必要。

### 6. Mesh-native tokenization is evolving fast

从坐标排序到拓扑感知 BFS，从三角 mesh 到四边形 mesh，从自回归到离散扩散——mesh-native 路线在 2024-2026 年间经历了三代快速迭代。面数上限的数量级增长（800 → 10,000）和拓扑类型的扩展（tri → quad）表明这条路线正在走向实用。

### 7. Three-axis positioning

当前 mesh generation 中的 3D latent / token 表征，需要在三个维度之间做取舍：

1. **Compactness**：token 足够少，训练可扩展。
2. **Localizability**：token 能对应明确的空间位置。
3. **Native-ness**：表示尽量接近真实 3D 资产，减少对后处理转换的依赖。

VecSet 更偏 compactness，SLAT 更偏 localizability，O-Voxel 更偏 native-ness，VoxSet 是三者的折中方案，OctFusion/Sparc3D 强调结构效率和模态纯度，VAT 把 compactness 推到极端，而 BPT / FACE / Nautilus / TSSR 代表的是 mesh-native 路线的另一端。

---

## A Working Comparison Framework

评估这一方向的论文时，可以统一使用以下 7 个维度进行定位：

1. **表示单元是什么？** set、voxel、octree node、face、patch 还是 field？
2. **token 是否带显式位置语义？**
3. **是更偏 geometry，还是 geometry + appearance 一体化？**
4. **是否支持 open surfaces / non-manifold / interior structures？**
5. **最终输出 mesh 需要多少后处理？**
6. **训练管线中是否存在模态转换（2D ↔ 3D）？**
7. **它更适合 generation、editing，还是 reconstruction backbone？**

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
8. **[OctFusion](octfusion.md)** — 八叉树 latent + 统一多尺度 U-Net
9. **[Sparc3D](sparc3d.md)** — 模态一致的纯 3D 稀疏卷积 VAE
10. **[VAT](vat.md)** — 极端压缩 variational tokenizer
11. **MeshAnything V2** — AMT，mesh-native tokenization
12. **BPT** — patch + block 压缩
13. **FACE** — one-face-one-token
14. **[Nautilus](nautilus.md)** — locality-aware BFS，5000 面
15. **[MeshRipple](meshripple.md)** — frontier-aware BFS + global memory
16. **[QuadGPT](quadgpt.md)** — 首个 quad mesh AR + tDPO
17. **[TSSR](tssr.md)** — 离散扩散 mesh-native，10,000 面

---

## Research Implications

从研究视角看，这条文献线提供了以下指导：

- 以**高质量 image-to-3D mesh generation** 为目标时，应重点关注 latent 是否具有足够的 localizability，以及训练管线是否存在模态不匹配。
- 以 **native 3D editing** 为目标时，应重点关注 latent 是否天然支持局部替换和重绘。
- 以**高质量 mesh 输出**为最终产物时，不能仅关注 latent field，还需将 mesh-native token 空间纳入考量——尤其是拓扑感知序列化（Nautilus、MeshRipple）和非 AR 范式（TSSR）。
- 以**与 LLM 统一**为长期目标时，极端压缩路线（VAT）的进展值得持续跟踪。
- 以**实际生产资产**为目标时，quad mesh 生成（QuadGPT）和 part-aware 生成（[PartCrafter](partcrafter.md)）是值得关注的方向。

3D Latent Space Representations 构成了一个具有独立研究价值的系统性方向，横跨表征设计、生成建模与下游应用多个层面。
