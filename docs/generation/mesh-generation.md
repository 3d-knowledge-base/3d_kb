# Mesh Generation Models

3D Mesh 生成模型总览。除了常见的**前馈式 (Feed-forward)** 和**基于优化 (Optimization-based)** 分类，更重要的一条主线其实是：

> 模型到底在什么样的 3D 隐空间 / token 空间里生成？

这决定了模型能否同时做到高保真、可扩展、可编辑，以及最终稳定输出 mesh。

---

## 核心方法对比

### 前馈式模型（一次前向传播生成）

| 模型 | 输入 | 输出表示 | 核心思想 | 发表时间 |
|:-----|:-----|:---------|:---------|:---------|
| **[TRELLIS](trellis.md)** | 文本/图像 | Mesh / 3DGS / RF | SLAT 结构化潜变量，两阶段 Flow Transformer，FlexiCubes 解码 | 2024.12 |
| **[Hunyuan3D 2.0](hunyuan3d.md)** | 单张图/草图 | Mesh (SDF→MC) | ShapeVAE + DiT + Paint 两阶段管线 | 2025.01 |
| **[TripoSG](triposg.md)** | 图像 | Mesh | 大规模高质量数据 + SDF VAE + Rectified Flow Transformer | 2025.02 |
| **Hi3DGen** | - | Mesh | 基于 TRELLIS 预训练参数和 SLAT | 2025.03 |
| **[Direct3D-S2](direct3d-s2.md)** | - | Mesh | Sparse SDF VAE + spatial sparse attention，高分辨率稀疏几何生成 | 2025.05 |
| **GRM** | 文本/图像 | 3DGS | 基于 Transformer 的大型高斯重建模型 | 2024.03 |
| **Shap-E** | 文本 | 隐式函数→Mesh | 直接生成 MLP 权重定义隐式形状 | 2023.05 |
| **Point-E** | 文本 | 点云 | 多步扩散模型生成 3D 点云 | 2022.12 |
| **Edgerunner** | 文本/类别 | Mesh | 将 Mesh 建模为顶点/面序列，自回归生成 | 2024.09 |
| **[BPT](bpt.md)** | 文本/类别 | 高分辨率 Mesh | Blocked and Patchified Tokenization，支撑高面数 mesh AR 训练 | 2024.11 |

---

## 3D 隐空间表征方法：文献主线

这一部分是理解近两年 mesh generation 的关键。

### 1. VecSet：先把 3D 压进 Transformer 友好的 latent set

代表：**3DShape2VecSet**

核心思想是把 3D 形状编码成固定长度的 latent vector set，再通过 cross-attention 从 latent 中读取局部几何信息。它的意义在于：

- 摆脱显式 voxel/grid 的立方复杂度；
- 让 3D latent 更适合 Transformer / diffusion；
- 开启了后续一系列 set-based latent 方案。

但 VecSet 的短板也很明显：**token 虽然紧凑，却缺少显式空间锚点**。

### 2. SLAT：把 latent 锚定到稀疏空间结构上

代表：**TRELLIS**

TRELLIS 的核心不是单纯做一个更强的 image-to-3D，而是提出 **SLAT (Structured LATent)**：

- 稀疏体素位置集合 `{p_i}` 负责结构；
- 每个 active voxel 的局部 latent `{z_i}` 负责细节。

这让 3D latent 同时具备：

- 明确的空间锚点；
- 局部编辑能力；
- 多种输出解码能力（mesh / 3DGS / radiance field）。

代价是 token 数明显比 VecSet 更重。

### 3. O-Voxel：从“结构化 latent”推进到“原生 3D latent”

代表：**[TRELLIS 2](trellis2.md)**

TRELLIS 2 进一步指出：即使有 SLAT，如果底层几何还依赖 SDF/isofsurface，开放表面、非流形结构、原生 PBR 材质仍然不好处理。

因此它提出 **O-Voxel**：

- 不再只依赖场表示；
- 直接在结构化单元中编码几何和材质；
- 借助 SC-VAE 实现更高压缩率。

这一步的本质是把 3D latent 从“带结构的中间表示”推进到“更接近原生 3D 资产的表示”。

### 4. VoxSet：VecSet 与 Sparse Voxel 的折中

代表：**[LATTICE / VoxSet](lattice.md)**

LATTICE 的一个重要的观点是：

> 关键的不是 local vs global，而是 latent code 是否 **localizable**。

所以 VoxSet 的做法是：

- 保留 VecSet 的紧凑性；
- 给 token 加上 coarse voxel anchors；
- 再用位置编码和 query jitter 让模型学会“在给定位置生成对应细节”。

结果就是一种**半结构化 latent**：

- 比纯 sparse voxel 更轻；
- 比纯 VecSet 更易定位；
- 支持 test-time scaling。

### 5. Sparse volumetric 路线：把高分辨率和复杂拓扑做起来

代表：**[TripoSG](triposg.md)、[SparseFlex](sparseflex.md)、[Direct3D-S2](direct3d-s2.md)**

- **TripoSG**：更像系统化 scaling 路线，强调高质量数据、SDF VAE、Rectified Flow Transformer、MoE 扩展。
- **SparseFlex**：强调高分辨率 sparse isosurface representation，支持 open surfaces、interior structures，以及 frustum-aware sectional voxel training。
- **Direct3D-S2**：强调 Sparse SDF VAE + spatial sparse attention，把高分辨率 sparse volumetric generation 做得更可扩展。

这一组工作很重要，因为它们证明：3D 生成并不只缺“好表示”，还缺一套能训练到高分辨率、大数据规模的系统工程。

### 6. Mesh-native 路线：直接把 mesh 当成第一性对象

代表：**MeshAnything V2、[BPT](bpt.md)、[FACE](face.md)**

这条路线不再问“什么 latent field 最好”，而是在问：

> 如果最终目标就是生成 mesh，那 mesh 序列应该怎么表示？

- **MeshAnything V2**：AMT，相邻面尽量共享边，一个新面尽可能只引入一个新顶点。
- **BPT**：Blocked and Patchified Tokenization，把序列长度压到约 `0.26`，从而支撑高面数训练。
- **FACE**：one-face-one-token，直接把 triangle face 当作基本生成单元，把压缩比推进到 `0.11`。

相比 latent voxel 路线，这条路线的好处是更原生 mesh-friendly；缺点是仍然要直接面对复杂拓扑的序列建模难题。

### 总结

| 路线 | 代表工作 | 主要优点 | 主要短板 |
|:-----|:---------|:---------|:---------|
| VecSet | 3DShape2VecSet | 紧凑、易训练 | 位置语义弱 |
| Structured latent | TRELLIS / SLAT | 有结构、有局部性、便于编辑 | token 较重 |
| Native structured latent | TRELLIS 2 / O-Voxel | 原生几何 + 材质，支持开放拓扑 | 体系复杂 |
| Semi-structured latent | LATTICE / VoxSet | 紧凑与可定位性折中 | 仍非最终 mesh 表示 |
| Sparse volumetric scaling | TripoSG / SparseFlex / Direct3D-S2 | 高分辨率与工程可扩展 | 仍需 field / surface 解码 |
| Mesh-native token | MeshAnything V2 / BPT / FACE | 直接面向 mesh 输出 | 自回归序列建模难度高 |

### 基于优化的模型（迭代优化 3D 表示）

| 模型 | 输入 | 输出表示 | 核心思想 | 发表时间 |
|:-----|:-----|:---------|:---------|:---------|
| **DreamFusion** | 文本 | NeRF | 开创 SDS 概念，2D 扩散模型指导 NeRF 优化 | 2022.09 |
| **Magic3D** | 文本 | Mesh | 两阶段（粗 NeRF → 精 Mesh），高分辨率纹理 | 2022.11 |
| **MVDream** | 文本 | NeRF→Mesh | 多视角扩散模型 + SDS 优化 | 2023.08 |
| **LucidDreamer** | 文本 | NeRF | ISM (Interval Score Matching) 改进 SDS | 2023.11 |
| **Latent-NeRF** | 文本 | NeRF | 优化潜向量→解码 NeRF | 2022.11 |
| **SJC** | 文本 | NeRF | Score Jacobian Chaining 替代 SDS | 2022.12 |

---

## 重建模型（多视图 → 3D）

| 模型 | 输入 | 输出 | 核心特点 |
|:-----|:-----|:-----|:---------|
| **GTR** | 多视图图像 | Mesh | LRM 基础上优化，DiffMC 全分辨率几何监督 + 秒级纹理精炼 |
| **Neuralangelo** | 多视图视频/图像 | Mesh (Neural SDF) | 哈希网格上的 Neural SDF，极擅长大场景高频细节 |

---

## 闭源模型

| 模型 | 备注 |
|:-----|:-----|
| Tripo-v2.5 | 闭源 |
| Meshy-4 | 闭源 |
| ~~Rodin-v1.5~~ | 闭源 |

---

## 趋势观察

1. **前馈式正在取代优化式**：速度从数小时降至数秒/分钟
2. **“隐空间表征”正在成为核心竞争点**：从 VecSet 到 SLAT、O-Voxel、VoxSet，竞争焦点已不是单纯 backbone 大小
3. **TRELLIS 逐步确立为主流骨干架构**：Hi3DGen、多个编辑方法都直接基于 TRELLIS 预训练
4. **SDF / isosurface 仍是主流，但面临更 native 的 3D 表示挑战**：TRELLIS 2、SparseFlex、FACE 都在往更原生方向推进
5. **Flow / Rectified Flow 正在替代传统 diffusion 训练**：TRELLIS、TripoSG 等都采用了这一方向
6. **mesh-native 路线重新获得关注**：MeshAnything V2、BPT、FACE 表明直接生成 mesh 仍是一条具有竞争力的路线
