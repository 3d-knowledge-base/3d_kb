---
comments: true
---

# Papers

3D 生成、编辑及相关领域文献索引。当前页收录 **79 篇**论文/方法，按研究方向分类，涵盖 mesh 生成中的 3D 隐空间表征主线与 mesh editing 主线的关键文献。

---

## Generation (26)

3D 物体生成方法：从文本/图像到 3D 资产。

---

**TRELLIS** (2412) — 结构化潜变量 (SLAT) + Sparse VAE + 两阶段 Flow Transformer 生成 + FlexiCubes Mesh 解码。当前多个编辑方法的骨干架构。

**TRELLIS 2** (2512) — O-Voxel 表示替代 SDF，原生 PBR 材质，SC-VAE 实现 16× 空间压缩。

![TRELLIS 2 O-Voxel](https://zzk-host-1319605683.cos.ap-beijing.myqcloud.com/20251225203741.png)

**Hunyuan3D 1.0** (2411) — 级联 text→image→multiview→3D pipeline，使用 Hunyuan-DiT。

**Hunyuan3D 2.0** (2501) — 解耦 Shape (DiT) + Texture (Paint) pipeline，高分辨率带纹理 3D 资产生成。

![Hunyuan3D 2.0 架构](https://zzk-host-1319605683.cos.ap-beijing.myqcloud.com/20250918100223.png)

**Hi3DGen** (2503) — 基于 TRELLIS，引入法线图作为几何桥梁 (NiRNE)，单图像到高保真 3D。

**Direct3D-S2** — Sparse SDF VAE + 空间稀疏注意力，实现 1024³ 分辨率 3D 生成。

![Direct3D-S2](https://zzk-host-1319605683.cos.ap-beijing.myqcloud.com/20251130205731.png)

**LATTICE / VoxSet** (2512) — 半结构化 VoxSet，结合稀疏体素结构与 VecSet 紧凑性，可扩展 3D 生成。

**TripoSG** (2502) — 大规模高质量数据 + SDF VAE + Rectified Flow Transformer + MoE 扩展，代表 3D 生成系统化 scaling 路线。

**UltraShape 1.0** (2512) — 粗到细生成框架 + 严格数据治理，高质量 3D 形状生成。

**3DShape2VecSet** (2301) — VecSet 路线奠基工作：固定长度 latent vector set (512×32)，cross-attention 解码隐式场，EDM 扩散生成。→ [详情页](generation/3dshape2vecset.md)

**MeshGPT** (2311) — Mesh-native AR 开创：graph convolution VQ-VAE 学面片码本 + decoder-only transformer 自回归生成三角网格 (CVPR 2024)。→ [详情页](generation/meshgpt.md)

**TEXGen** (2411) — 混合 2D-3D 扩散网络，前馈式 UV 纹理生成。

**TriDiff-4D** (2511) — 基于扩散的 Triplane 重姿态化，36s 内生成 4D 人体 Avatar (H100)。

**SAM 3D** (2511) — 单图像 3D 重建的生成基础模型，同时预测几何、纹理和布局。

**ShapeLLM-Omni** (2506) — 原生多模态 LLM，通过 3D VQVAE tokenization 统一 Mesh 理解、生成与编辑。

![ShapeLLM-Omni](https://zzk-host-1319605683.cos.ap-beijing.myqcloud.com/20251004135739.png)

**Nautilus** (2501) — Locality-aware AE，BFS 邻接遍历 + 共享顶点/边压缩序列，mesh-native AR 生成推到 5000 面 (ICCV 2025)。

**QuadGPT** (2509) — 首个端到端四边形 mesh AR 生成，统一 tri/quad tokenization + tDPO RL 微调 (ICLR 2026)。

**TSSR** (2510) — 离散扩散 mesh-native 生成，拓扑雕刻 + 形状细化两阶段，10,000 面 / 1024^3 分辨率。

**MeshRipple** (2512) — Frontier-aware BFS tokenization + 前沿批量扩展 + sparse-attention global memory，解决 AR mesh 生成的拓扑断裂问题。

**Sparc3D** (2505) — Sparcubes + Sparconv-VAE，纯 3D 稀疏卷积模态一致管线，1024^3 高保真重建。

**OctFusion** (2408) — 八叉树 latent + 统一多尺度 U-Net 扩散，2.5s 生成连续流形 mesh (SGP 2025)。

**VAT** (2412) — Variational Tokenizer，250x 压缩（1MB→3.9KB），多尺度隐式层级，适配 LLM 架构的 3D tokenization。

**PartCrafter** (2506) — 首个 part-aware 3D mesh 生成，compositional latent space + 层级注意力，端到端部件分解。

**Fantasia3D** (2306) — DMTet + SDS + PBR 材质解耦，Text-to-3D 几何与外观分离优化 (ICCV 2023)。→ [详情页](generation/fantasia3d.md)

**Magic3D** (2306) — NeRF coarse + DMTet fine 两阶段 coarse-to-fine Text-to-3D (CVPR 2023)。→ [详情页](generation/magic3d.md)

---

## Editing (22)

3D Mesh 编辑方法：基于扩散/生成模型、代码、拓扑的各种编辑技术。

---

**VoxHammer** (2508) — Training-free，基于 TRELLIS 反演缓存 + 特征注入保留未编辑区，需 3D 掩码。提出 Edit3D-Bench。

![VoxHammer 架构](https://zzk-host-1319605683.cos.ap-beijing.myqcloud.com/20250922221728.png)

**CraftMesh** (2509) — 图像编辑 → Mesh 生成 → 泊松融合 pipeline，高保真 3D Mesh 编辑。

**NANO3D** (2510) — Training-free，基于 TRELLIS + FlowEdit 的编辑 pipeline，创建 Nano3D-Edit-100k 数据集。

![NANO3D pipeline](https://zzk-host-1319605683.cos.ap-beijing.myqcloud.com/20251030114548.png)

**3DEditVerse** (2510) — 116K 训练对的大规模 3D 编辑数据集 + 3DEditFormer 模型，无需 3D 掩码。

![3DEditVerse](https://zzk-host-1319605683.cos.ap-beijing.myqcloud.com/20251019165249.png)

**3DEditFormer** — 条件 Transformer：以源 3D 为条件的 image-to-3D 编辑 (3DEditVerse 的模型部分)。

**PrEditor3D** (2412) — 并行 2D 编辑 + 3D 重建 + 特征空间融合替换，GTR 解码输出编辑后 Mesh。

**Steer3D** (2512) — 类 ControlNet 文本引导，对 TRELLIS Image-to-3D 模型进行单次推理编辑。

**Easy3E** (2602) — 完全前馈的 3D 编辑框架，基于 TRELLIS，使用单张编辑视图。

**MeshPad** (2503) — 草图引导交互式 Mesh 生成/编辑，通过 add/delete 分解操作，基于 MeshAnythingV2。

![MeshPad 方法概览](https://zzk-host-1319605683.cos.ap-beijing.myqcloud.com/20250923193614.png)

**Native 3D Editing** (2511) — 前馈式原生 3D 编辑，直接在 TRELLIS 结构化潜变量上操作，不经过 2D 中间步骤。

**AnchorFlow** (2511) — 无掩码 3D 编辑，在 Hunyuan3D 2.1 上通过锚点对齐的潜流实现。含 Eval3DEdit 数据集。

**CMD** (2505) — Controllable Multiview Diffusion，通过 MVControlNet 实现局部 3D 编辑和渐进生成。

**Instructive3D** (2501) — 文本指令驱动 3D 编辑，在 LRM 的 Triplane 潜空间中使用扩散适配器。

**Masked LRMs** (2412) — 3D 遮挡体 + 条件视图编辑，通过 Masked LRM 补全实现 Mesh 编辑。

**SKED** (2308) — 草图 + 文本引导 NeRF 编辑，结合 SDS 优化。

**ShapeFusion** (2403) — Mesh 顶点空间直接操作的扩散式局部编辑，对选定区域加噪、保持未编辑区域。

**TEXTure** (2302) — 迭代式 3D 纹理生成/编辑，深度条件扩散 + keep/refine/generate 区域分类。

![TEXTure pipeline](https://storage.googleapis.com/static.aiforpro.com/TEXTure_pipeline.png)

**Text2Mesh** (2112) — CLIP 引导测试时优化，修改 Mesh 颜色和局部几何细节。每次输入需重新优化。

**Text2VDM** (2502) — 文本到矢量位移贴图 (VDM) 生成，用于 3D 雕刻笔刷，SDS 引导 Mesh 变形。

![Text2VDM pipeline](https://raw.githubusercontent.com/ht-meng/Text2VDM/main/assets/overview.png)

**GenVDM** (2503) — 图像到 VDM pipeline，通过多视图法线图生成 3D 雕刻笔刷。

![GenVDM pipeline](https://raw.githubusercontent.com/ml-research-study-notes/pics/main/GenVDM_pipeline.png)

**NI-Tex** (2511) — 非等距纹理生成，处理 3D 服装中的拓扑/几何不匹配问题。

**Poisson-Based Mesh Editing** (2005) — 经典几何处理框架：泊松方程操控梯度场，统一变形、合并、平滑。

---

## Mesh Processing (3)

Mesh 重建、补全与结构化处理。

---

**MeshAnything V2** (2408) — Adjacent Mesh Tokenization，利用相邻面共享顶点将序列长度减半，面数 800→1600 (ICCV 2025)。→ [详情页](generation/meshanything-v2.md)

**BPT** (2411) — Blocked and Patchified Tokenization，把 Mesh 序列压缩到约 0.26，使 8k faces 级别训练成为可能。

**FACE** (2603) — one-face-one-token 自回归 mesh 表示，压缩比达到 0.11，属于当前 mesh-native 路线中压缩率较高的一类。

**MeshCoder** (2508) — 点云到可执行 Blender Python 脚本，构建大规模「3D 模型-代码」配对数据集训练 LLM。

![MeshCoder 数据集可视化](https://zzk-host-1319605683.cos.ap-beijing.myqcloud.com/20251118134905.png)

**X-Part** (2509) — 基于扩散的可控 3D 部件分解，使用边界框和语义特征。

---

## Representations & Foundations (12)

3D 表征方法、表面提取、重建模型。

---

**3D Representation Survey** — Voxel, 点云, Mesh, SDF, NeRF, 3DGS, Triplane, DMTet 综述。

![3D 表征概览](https://arxiv.org/html/2410.06475v1/extracted/5912068/3d.png)

**NeRF** — 神经辐射场：MLP 编码连续 5D→4D 函数，实现新视角合成。

**3D Gaussian Splatting (3DGS)** — 显式 3D 高斯椭球表示，实时新视角合成。

**SDF (Signed Distance Function)** — 隐式 3D 形状表示基础。

**Triplane** — 三个正交 2D 特征平面的高效 3D 表示 (EG3D, LRM 等使用)。

**Marching Cubes & FlexiCubes** — 经典 MC 算法与可微分 FlexiCubes Mesh 提取的技术对比。

![Marching Cubes 基本拓扑](https://zzk-host-1319605683.cos.ap-beijing.myqcloud.com/20251201122856.png)

**SparseFlex** (2503) — 稀疏结构化可微 isosurface 表示，支持高分辨率、开放表面、内部结构与 frustum-aware sectional voxel training。

**GTR** — 改进 LRM：Conv 编码器 + PixelShuffle + DiffMC 几何精炼 + 纹理微调。

**SuGaR** (2311) — Surface-Aligned Gaussian Splatting，从 3DGS 高效提取 Mesh。

**GOF** (2404) — Gaussian Opacity Fields，从训练好的 3DGS 点云提取表面。

**AGS-Mesh** (2411) — 自适应高斯 Splatting + 几何先验，手机拍摄室内场景重建。

**GS-2M** (2509) — Gaussian-to-Mesh：TSDF Fusion + Marching Cubes 提取。

---

## Two Core Literature Threads

### Mesh Generation: 3D latent / token representation line

- **3DShape2VecSet** — VecSet：固定长度 latent vector set，开启 set-based 3D latent generation
- **TRELLIS** — SLAT：稀疏空间结构 + 局部 latent，代表 structured latent 路线
- **TRELLIS 2** — O-Voxel：更原生的 structured 3D latent，支持 open surfaces / non-manifold / PBR
- **LATTICE / VoxSet** — semi-structured latent：强调 `localizable code`
- **TripoSG / SparseFlex / Direct3D-S2** — 高分辨率 sparse volumetric scaling 路线
- **OctFusion** — 八叉树 latent + 统一多尺度 U-Net，避免 cascaded diffusion
- **Sparc3D** — 模态一致的纯 3D 稀疏卷积 VAE，消除 2D-3D 转换损失
- **VAT** — 极端压缩的 variational tokenizer (250x)，对接 LLM 架构
- **MeshGPT** — mesh-native AR 开山：graph conv VQ-VAE + decoder-only transformer (CVPR 2024)
- **MeshAnything V2 / BPT / FACE** — mesh-native token 路线，直接把 mesh 本身作为生成对象
- **Nautilus** — locality-aware tokenization，利用面间拓扑邻接压缩序列，5000 面
- **MeshRipple** — frontier-aware BFS + sparse-attention global memory，拓扑连贯生成
- **QuadGPT** — 首个四边形 mesh AR 生成 + tDPO RL 微调
- **TSSR** — 离散扩散替代 AR，拓扑雕刻 + 形状细化，10,000 面
- **PartCrafter** — compositional latent space，端到端部件感知 mesh 生成

### Mesh Editing: method evolution line

- **优化式 / 几何处理式**：Poisson-Based、Neural Cages、Text2Mesh、TEXTure、SKED
- **2D lifting 到 3D**：MVEdit、Tailor3D、CraftMesh、PrEditor3D
- **原生 3D latent editing**：TRELLIS、VoxHammer、NANO3D、3DEditVerse、Steer3D、Easy3E、Native 3D Editing、VecSet-Edit

这条发展线索背后的主要变化是：编辑操作正逐步从“先在 2D 改，再重建回 3D”转向“直接在原生 3D latent 上改”。

---

## Benchmarks & Evaluation (5)

3D 生成评估框架与质量基准。

---

**MATE-3D / HyperScore** (2412) — 多维度质量评估器：4 维绝对评分 (MOS) + 超网络自动化评估器。107K 标注。

![MATE-3D 结果](https://zzk-host-1319605683.cos.ap-beijing.myqcloud.com/20251005113431.png)

**T³Bench** (2310) — 自动化 Text-to-3D 评估：300 提示 × 3 难度级别，区域卷积检测 Janus 问题。

**3DGen-Bench** (2503) — 统一 Text/Image→3D 评估：68K 众包投票 + 56K 专家分数 + 3DGen-Score/Eval 工具。

**Eval3D** (2504) — 基础模型探针的一致性评估：零样本、像素级空间反馈、可解释缺陷定位。

**Hi3DEval** (2508) — 分层级评估：对象→部件→材质三级，30 模型 × 15K 资产，涵盖 PBR 属性评估。

---

## Datasets (5)

3D 数据集与标注资源。

---

**Objaverse-XL** (2307) — 10M+ 3D 对象，来自 GitHub/Thingiverse/Sketchfab/Polycam，含 CLIP 嵌入。

**ABO (Amazon Berkeley Objects)** (2110) — 8K 高质量 3D 模型 + 重建/材质/检索 benchmark (CVPR 2022)。

**ShapeTalk** (2023) — 500K+ 自然语言描述 3D 形状差异，用于语言驱动形状编辑 (CVPR 2023)。

**S2O (Static to Openable)** (2409) — 将静态 3D Mesh 转换为可交互铰接对象，面向 Embodied AI。

**VideoCAD** (2505) — 大规模合成数据集，CAD UI 交互 + VideoCADFormer 模型。

---

## Scene Generation (4)

场景级 3D 生成。

---

**WorldGen** (2511) — 文本到 3D 世界生成：场景规划 → 重建 → 分解 → 增强。

**WorldGrow** (2510) — 通过场景块 Inpainting 实现无限 3D 世界生成，粗到细策略。

**Infinigen** (2306) — 全程序化自然场景 3D 生成器，真实几何，可用于 CV 训练数据。

**Infinigen Indoors** (2406) — Infinigen 扩展到完全程序化室内场景生成。

![Infinigen Indoors](https://zzk-host-1319605683.cos.ap-beijing.myqcloud.com/20251119134804.png)
