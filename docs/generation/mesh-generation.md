---
comments: true
---

# Mesh Generation Models

3D Mesh 生成模型总览。除了常见的**前馈式 (Feed-forward)** 和**基于优化 (Optimization-based)** 分类，更重要的一条主线其实是：

> 模型到底在什么样的 3D 隐空间 / token 空间里生成？

这决定了模型能否同时做到高保真、可扩展、可编辑，以及最终稳定输出 mesh。

关于隐空间本身的系统性梳理，请参见 **[3D Latent Space Representations](latent-space-representations.md)**。本页聚焦**生成模型**本身的对比和方法路线。

---

## 核心方法对比

### 前馈式模型（一次前向传播生成）

| 模型 | 输入 | 输出表示 | 核心思想 | 发表时间 |
|:-----|:-----|:---------|:---------|:---------|
| **[TRELLIS](trellis.md)** | 文本/图像 | Mesh / 3DGS / RF | SLAT 结构化潜变量，两阶段 Flow Transformer，FlexiCubes 解码 | 2024.12 |
| **[TRELLIS 2](trellis2.md)** | 文本/图像 | Mesh (O-Voxel) | 原生 3D 结构 latent，SC-VAE + Flow DiT，开放拓扑 + PBR | 2025.06 |
| **[Hunyuan3D 2.0](hunyuan3d.md)** | 单张图/草图 | Mesh (SDF→MC) | ShapeVAE + DiT + Paint 两阶段管线 | 2025.01 |
| **[TripoSG](triposg.md)** | 图像 | Mesh | 大规模高质量数据 + SDF VAE + Rectified Flow Transformer | 2025.02 |
| **[Hi3DGen](hi3dgen.md)** | 图像 | Mesh | 基于 TRELLIS 预训练参数和 SLAT | 2025.03 |
| **[Direct3D-S2](direct3d-s2.md)** | 图像 | Mesh | Sparse SDF VAE + spatial sparse attention，高分辨率稀疏几何生成 | 2025.05 |
| **[OctFusion](octfusion.md)** | 文本/图像 | Mesh | 八叉树 latent + 统一多尺度 U-Net 扩散，单卡 2.5 秒 | 2025 (SGP) |
| **[Sparc3D](sparc3d.md)** | 图像 | Mesh | 模态一致 3D VAE (Sparcubes + Sparconv-VAE)，1024³ | 2025.06 |
| **[LATO](lato.md)** | 文本/图像 | Mesh | Latent Tree Optimization + 3D-aware generation | 2025.02 |
| **GRM** | 文本/图像 | 3DGS | 基于 Transformer 的大型高斯重建模型 | 2024.03 |
| **Shap-E** | 文本 | 隐式函数→Mesh | 直接生成 MLP 权重定义隐式形状 | 2023.05 |
| **Point-E** | 文本 | 点云 | 多步扩散模型生成 3D 点云 | 2022.12 |

### Mesh-native 前馈式模型（直接生成 mesh 序列 / 结构）

| 模型 | 输入 | 建模范式 | 核心思想 | 面数上限 | 发表时间 |
|:-----|:-----|:---------|:---------|:---------|:---------|
| **Edgerunner** | 文本/类别 | AR | 将 Mesh 建模为顶点/面序列，自回归生成 | ~800 | 2024.09 |
| **[BPT](bpt.md)** | 文本/类别 | AR | Blocked and Patchified Tokenization，压缩比 0.26 | ~3000 | 2024.11 |
| **[FACE](face.md)** | 文本/类别 | AR | one-face-one-token，压缩比 0.11 | ~4000 | 2024.12 |
| **[Nautilus](nautilus.md)** | 条件特征 | AR | locality-aware AE + BFS 拓扑序列化 | 5000 | 2025 (ICCV) |
| **[MeshRipple](meshripple.md)** | 条件特征 | AR | frontier-aware BFS + sparse-attention global memory | ~4000 | 2025.06 |
| **[QuadGPT](quadgpt.md)** | 文本/图像 | AR + RL | 首个 quad mesh AR + tDPO 拓扑 RL 微调 | ~3000 | 2026 (ICLR) |
| **[TSSR](tssr.md)** | 条件特征 | 离散扩散 | 拓扑雕刻 + 形状细化两阶段，首个非 AR mesh-native | 10,000 | 2025.06 |
| **[PartCrafter](partcrafter.md)** | 文本/图像 | AR (part-aware) | part-level mesh assembly，组件级可控生成 | ~2000/part | 2025.06 |
| **[VAT](vat.md)** | 条件特征 | AR (token) | 250x 压缩到 256 tokens，可对接 LLM | N/A (token) | 2025.05 |

---

## 3D 隐空间表征方法：文献主线

这一部分是理解近两年 mesh generation 的关键。详细分析请参见 **[3D Latent Space Representations](latent-space-representations.md)**，此处仅做概要对比。

### 路线总结

| 路线 | 代表工作 | 主要优点 | 主要短板 |
|:-----|:---------|:---------|:---------|
| VecSet | 3DShape2VecSet | 紧凑、易训练 | 位置语义弱 |
| Structured latent | TRELLIS / SLAT | 有结构、有局部性、便于编辑 | token 较重 |
| Native structured latent | TRELLIS 2 / O-Voxel | 原生几何 + 材质，支持开放拓扑 | 体系复杂 |
| Semi-structured latent | LATTICE / VoxSet | 紧凑与可定位性折中 | 仍非最终 mesh 表示 |
| VecSet-based scaling | TripoSG / Hunyuan3D 2.0 | 系统化、可工业部署 | 隐空间仍为 unstructured set |
| Sparse volumetric scaling | SparseFlex / Direct3D-S2 | 高分辨率与工程可扩展 | 仍需 field / surface 解码 |
| Octree / modality-consistent | OctFusion / Sparc3D | 自适应分辨率、模态纯度 | 实现复杂，仍依赖 isosurface |
| Extreme compression | VAT | 极端紧凑 (256 tokens)，可对接 LLM | 空间局部性丢失 |
| Mesh-native token | BPT / FACE / Nautilus / TSSR / QuadGPT | 直接面向 mesh 输出 | 自回归序列建模难度高 |

### 关键文献线索

1. **VecSet** → **SLAT** → **O-Voxel**：从紧凑 set 到有结构、再到原生 3D latent，逐步增加 native-ness。
2. **TripoSG / Hunyuan3D 2.0**：继承 VecSet latent 路线，通过大规模数据 + SDF VAE + RF/DiT 做系统化 scaling（注意 TripoSG 的 latent 是 VecSet 而非 sparse volumetric）。
3. **SparseFlex / Direct3D-S2**：在 sparse volumetric 上做工程 scaling，latent 具有显式 3D 空间结构。
3. **OctFusion / Sparc3D**：关注结构效率（八叉树自适应）和管线纯度（纯 3D 训练消除模态转换）。
4. **BPT → FACE → Nautilus → MeshRipple → TSSR**：mesh-native tokenization 的快速演进，压缩比 0.26 → 0.11 → 拓扑感知，面数 ~800 → 10,000。
5. **QuadGPT / PartCrafter**：从 tri mesh 扩展到 quad mesh、从 whole shape 扩展到 part-aware，mesh-native 路线正在走向生产级需求。
6. **VAT**：极端压缩 (250x-2000x)，让 3D 进入 LLM context window。

---

## 基于优化的模型（迭代优化 3D 表示）

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

1. **前馈式正在取代优化式**：速度从数小时降至数秒/分钟，且质量已可匹配甚至超越优化式方法。
2. **隐空间表征成为核心竞争点**：从 VecSet 到 SLAT、O-Voxel、VoxSet、Octree latent，竞争焦点已不是单纯 backbone 大小，而是 latent 的结构化程度和原生性。
3. **TRELLIS 系列确立为主流骨干架构之一**：Hi3DGen、多个编辑方法都直接基于 TRELLIS 预训练。TRELLIS 2 进一步将 latent 从 SDF-based 推向 native 3D 资产。
4. **模态一致性正在被重视**：Sparc3D 指出 2D 渲染监督训练 3D VAE 存在模态不匹配，纯 3D 原生训练管线（3D 输入 → 3D latent → 3D 监督）可能成为后续默认选择。
5. **Mesh-native 路线快速演进**：面数上限从 ~800 (2024 MeshGPT) 增长到 10,000 (2025 TSSR)，同时扩展到 quad mesh (QuadGPT)、part-aware (PartCrafter)、离散扩散 (TSSR) 等新方向。
6. **Flow / Rectified Flow 正在替代传统 diffusion 训练**：TRELLIS、TripoSG、TRELLIS 2 等主流方法均采用 Rectified Flow。
7. **极端压缩和 LLM 对接成为新方向**：VAT 用 256 个 token 表示一个 3D shape (250x 压缩)，使 3D 可以直接进入 LLM context window，打开 multimodal 统一建模的可能性。
8. **从 whole shape 到 part-aware**：PartCrafter 代表的 part-level mesh assembly 方向，更贴近实际 3D 建模工作流——组件级可控、可复用、可局部替换。
