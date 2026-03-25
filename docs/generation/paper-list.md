---
comments: true
---

# Generation Paper List

本页收录 3D Generation 方向的典型文献，按子方向分组。每篇论文附有简要中文描述；如有详情页则提供链接。

---

## Mesh / Object Generation

### 3DShape2VecSet (2023.01) → [详情页](3dshape2vecset.md)

VecSet 路线的奠基工作。将 3D 形状编码为固定长度的潜向量集合（512 个 32 维向量），潜变量不绑定空间位置，通过 cross-attention 与 3D 空间建立关系。在 VecSet 上训练 EDM 扩散模型实现 3D 形状生成。后续 CLAY、Michelangelo、TripoSG 等方法都继承了这一 set-based latent 范式。SIGGRAPH 2023。

### MeshGPT (2023.11) → [详情页](meshgpt.md)

Mesh-native 自回归生成的开创性工作。用图卷积 VQ-VAE 将三角面片编码为离散码本，再用 decoder-only transformer 自回归预测 token 序列生成三角网格。在 ShapeNet 上 FID 相比 PolyGen 提升约 30 分。序列长度限制了面数，但建立了后续 BPT、MeshAnything V2、FACE、Nautilus 等方法的基本范式。CVPR 2024。

### TRELLIS (2024.12) → [详情页](trellis.md)

提出 SLAT（Structured LATent）表示，将 3D 资产编码为稀疏体素位置 + 局部潜变量的组合。采用 Sparse VAE 编解码 + 两阶段 Rectified Flow Transformer 生成（先结构后细节）。SLAT 可统一解码为 Mesh、3DGS、辐射场等多种格式，成为后续众多编辑方法（VoxHammer、NANO3D、Steer3D 等）的核心骨干。

### TRELLIS 2 (2025.12) → [详情页](trellis2.md)

TRELLIS 的正式续作。引入 O-Voxel 作为更 native 的结构化 3D 资产表示，以及 SC-VAE（Shape-Color VAE）对几何与外观进行显式解耦。相比 SLAT 依赖多视角 DINOv2 特征聚合，O-Voxel 直接从 3D 资产本身编码，减少了信息瓶颈，提升了重建保真度和生成质量。

### Hunyuan3D 2.0 (2025.01) → [详情页](hunyuan3d.md)

腾讯的大规模 3D 生成系统，核心设计是几何-纹理解耦：ShapeVAE 负责几何生成（SDF → Mesh），Paint 模块负责高分辨率纹理合成。采用 DiT 架构和大规模训练数据，在几何精度和纹理质量上均达到工业级水平。支持 Text-to-3D 和 Image-to-3D 两条路径。

### TripoSG (2025.02) → [详情页](triposg.md)

以大规模高质量数据为核心驱动力的 3D 生成方法。采用 SDF VAE 将 3D 形状编码为连续符号距离场潜变量，配合 Rectified Flow Transformer 完成生成。强调数据规模与质量（百万级清洗数据）对生成效果的重要影响，在几何细节和鲁棒性上表现突出。

### SparseFlex (2025.03) → [详情页](sparseflex.md)

聚焦高分辨率、任意拓扑 3D 形状建模的基础表示。将 FlexiCubes 从 dense grid 改造为稀疏结构，仅在表面附近保留活跃体素。技术特点是 frustum-aware sectional voxel training：每次只激活当前视角相关的体素参与训练，使高分辨率可微 isosurface 建模可行。支持开放表面与内部结构。

### Direct3D-S2 (2025.05) → [详情页](direct3d-s2.md)

面向可扩展 3D 生成的方法，采用 Sparse SDF VAE 将 3D 形状编码为稀疏 SDF 潜变量，并在生成阶段使用 Spatial Sparse Attention 降低 Transformer 的计算复杂度。这使得模型能在更高分辨率上生成 3D 资产，同时保持训练和推理效率。

### LATTICE / VoxSet (2025.12) → [详情页](lattice.md)

提出介于纯集合（set）和稀疏体素（sparse voxel）之间的"半结构化"潜变量表示。核心思想是让潜变量既保留一定的空间可定位性（localizable），又不完全绑定到固定网格上，从而在生成灵活性和空间结构信息之间取得平衡。

### MeshAnything V2 (2024.08) → [详情页](meshanything-v2.md)

提出 AMT（Adjacent Mesh Tokenization）方法，利用相邻面片共享顶点的特性将序列长度减半，面数上限从 800 提升到 1600。基于 OPT-350M 架构，以点云为条件进行自回归 mesh 生成。AMT 的邻接遍历思路被后续 Nautilus、EdgeRunner 等方法继承。ICCV 2025。

### BPT (2024.11) → [详情页](bpt.md)

提出 Blocked and Patchified Tokenization 策略，将高面数三角网格分块、patch 化后压缩为更短的 token 序列。这使得自回归 Transformer 能够生成高达数千面的精细 mesh，突破了此前 mesh 生成方法在面数上的瓶颈。

### FACE (2026.03) → [详情页](face.md)

提出 one-face-one-token 的自回归 mesh 生成范式：每个三角面片被编码为单个 token，通过 VQ-VAE 学习离散面片码本。相比逐顶点坐标生成（如 MeshGPT），这种面片级 tokenization 缩短序列长度，使高面数 mesh 的自回归生成变得高效。

### Hi3DGen (2025.03) → [详情页](hi3dgen.md)

基于 TRELLIS 风格骨干的高保真几何生成方法，在几何细节精度上进一步提升。

### Nautilus (2025.01) → [详情页](nautilus.md)

利用流形 mesh 面间拓扑邻接关系做序列压缩的 locality-aware autoencoder。通过 BFS 邻接遍历和共享顶点/边编码将序列长度大幅缩短，配合双流点云条件模块，将 mesh-native AR 生成规模从 ~1600 面推到 5000 面。ICCV 2025。

### QuadGPT (2025.09) → [详情页](quadgpt.md)

首个端到端四边形 mesh 自回归生成方法。设计统一的 tri/quad 混合拓扑 tokenization，并引入面向拓扑质量的 RL 微调方法 tDPO（基于 DPO），在 quad mesh 生成的几何精度和拓扑质量上建立新基准。ICLR 2026。

### TSSR (2025.10) → [详情页](tssr.md)

用离散扩散模型（而非自回归）做 mesh-native 生成。将生成分为拓扑雕刻（Topology Sculptor）和形状细化（Shape Refiner）两阶段，通过并行去噪实现全局拓扑推理。支持 10,000 面、1024^3 分辨率。

### MeshRipple (2025.12) → [详情页](meshripple.md)

通过 frontier-aware BFS tokenization 将 mesh AR 生成的序列化顺序从空间坐标排序改为拓扑邻接顺序。配合前沿批量扩展和稀疏注意力全局记忆，从根本上解决了 mesh 自回归生成中的表面断裂和拓扑不连贯问题。

### Sparc3D (2025.05) → [详情页](sparc3d.md)

提出 Sparcubes（稀疏可变形 Marching Cubes 表示）和 Sparconv-VAE（纯 3D 稀疏卷积 VAE），构建模态一致的 3D 生成管线。消除 2D-3D 模态转换带来的细节损失，在 1024^3 分辨率下实现高保真重建和生成。

### OctFusion (2024.08) → [详情页](octfusion.md)

八叉树上的隐空间表示和统一多尺度 U-Net 扩散模型。避免 cascaded diffusion 的多模型复杂性，在单张 4090 GPU 上 2.5 秒生成任意分辨率的连续流形 mesh。SGP 2025。

### VAT (2024.12) → [详情页](vat.md)

Variational Tokenizer，通过 in-context transformer 压缩和变分残差量化，实现 250x 压缩比（1MB mesh → 3.9KB, 96% F-score）。多尺度 token 从同一高斯分布的不同子空间分配，构建隐式层级结构，适合 coarse-to-fine 自回归生成。

### PartCrafter (2025.06) → [详情页](partcrafter.md)

首个端到端 part-aware 3D mesh 生成模型。通过 compositional latent space 和层级注意力机制，从单张 RGB 图像同时生成多个语义独立、几何一致的 3D mesh 部件。建立在预训练 3D DiT 之上。

### Fantasia3D (2023.06) → [详情页](fantasia3d.md)

DMTet + SDS + PBR 材质解耦的 Text-to-3D 方法。将几何和外观解耦优化：先用 DMTet 作为几何表征通过 SDS loss 优化 mesh，再优化 PBR 材质参数。输出可在标准渲染管线中重新光照。ICCV 2023。

### Magic3D (2023.06) → [详情页](magic3d.md)

NVIDIA 的两阶段 coarse-to-fine Text-to-3D 方法。第一阶段用 Instant-NGP (NeRF) 快速粗优化，第二阶段转换为 DMTet 进行高分辨率精细优化。生成速度比 DreamFusion 快 2 倍，分辨率更高。CVPR 2023。

### Edgerunner (2024.09)

面向 artist-style mesh 的自回归生成方法，技术特点在于压缩 mesh 序列的设计，使得生成的 mesh 更接近手工建模的拓扑质量。

---

## Scene / World Generation

### WorldGen (2025.11) → [详情页](scene-generation.md)

采用"规划 → 重建 → 增强"三阶段流水线生成完整 3D 世界。先用 LLM 规划场景布局，再利用 3D 重建技术构建初始场景，最后通过增强模块提升视觉质量和一致性。

### WorldGrow (2025.10) → [详情页](scene-generation.md)

通过 scene-block inpainting 策略实现无限 3D 世界扩展：每次在已有场景边界处生成新的场景块并无缝拼接，从而实现理论上无限延伸的 3D 世界生成。

### MIDI (2024.12) → [详情页](scene-generation.md)

专注多实例 3D 生成与组合装配。先独立生成场景中的各个 3D 物体，再通过组合策略将它们合理放置到统一场景中，实现从文本描述到多物体 3D 场景的端到端生成。

### Infinigen (2023.06) → [详情页](scene-generation.md)

大规模程序化自然世界生成系统，通过精心设计的程序化规则生成地形、植被、岩石、水体等自然元素，能产出极高质量和多样性的 3D 自然场景数据。

### Infinigen Indoors (2024.06) → [详情页](scene-generation.md)

Infinigen 的室内扩展版本，将程序化生成范式应用于室内场景，自动生成家具、装饰、房间布局等，为室内 3D 场景理解和生成提供大规模训练数据。
