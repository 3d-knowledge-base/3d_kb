---
comments: true
---

# Mesh Editing Landscape

Mesh 编辑方法概述。当前方法可按技术路线分为：**基于扩散/生成模型的方法**（主流）、**基于代码的方法**、**基于拓扑/几何处理的方法**，以及更广泛的**前馈式 / 基于优化 / 混合式**分类。

---

## 先看主线：Mesh Editing 方法是怎么发展的

过去两年，mesh editing 不是线性发展的，而是明显分成了三代：

### 第一代：优化式 / 几何处理式

代表：**Poisson-Based Mesh Editing、[Neural Cages](neural-cages.md)、[Text2Mesh](text2mesh.md)、[TEXTure](texture.md)、SKED**

这类方法的共同点是：

- 或者直接在 mesh 顶点/梯度场上做几何处理；
- 或者把 2D 先验当作损失函数，在测试时对单个 3D 对象反复优化。

优点是控制比较直接，理论上能精细修改；缺点也明显：

- 慢（Text2Mesh ~32 min/object，TEXTure ~5 min/object）；
- 每个输入都要重做一遍优化；
- 很难扩展到稳定、高保真、批量化 3D 编辑。

### 第二代：2D 编辑能力提升到 3D

代表：**[MVEdit](mvedit.md)、[Tailor3D](tailor3d.md)、[PrEditor3D](preditor3d.md)、[CraftMesh](craftmesh.md)**

这类方法实质上承认一个现实：

> 2D 图像编辑远比 3D 编辑成熟，因此先把编辑操作放在 2D，再回升到 3D。

它们往往采用：

- 多视图图像编辑；
- 多视图一致性约束；
- 再通过重建模型回到 mesh / triplane / neural field。

优点是直接继承 2D 编辑模型的强大语义能力；问题是：

- 2D → 3D 提升过程会带来不一致；
- 非编辑区域保持不容易；
- 结构控制常常不如原生 3D 表示稳定。

### 第三代：在原生 3D latent 上直接编辑

代表：**TRELLIS、[VoxHammer](voxhammer.md)、[NANO3D](nano3d.md)、[3DEditVerse](3deditverse.md)、[Steer3D](steer3d.md)、[Easy3E](easy3e.md)、[Native 3D Editing](native-3d-editing.md)、[VecSet-Edit](vecset-edit.md)、[AnchorFlow](anchorflow.md)**

这代方法的转折点，是 **TRELLIS/SLAT** 和 **Hunyuan3D VecSet** 这类原生 3D 结构化 latent 的出现。因为一旦 3D 表示本身具备：

- 稀疏空间结构；
- 局部 latent；
- 可局部替换、局部重绘；

那么很多编辑问题就不必再绕回 2D。

因此这一代的核心思想是：

> 尽量把编辑操作直接放在 3D latent / 3D structure 上，而不是仅把 2D 编辑结果"投影"回去。

值得注意的是，这一代内部存在两条并行的 backbone 路线：

- **TRELLIS/SLAT 系**：VoxHammer、NANO3D、3DEditVerse、Steer3D、Easy3E、Native 3D Editing
- **VecSet/Flow 系**：VecSet-Edit（基于 TripoSG LRM）、AnchorFlow（基于 Hunyuan3D 2.1）

---

## 1. 基于扩散/生成模型的 Mesh 编辑方法（主流）

这些方法利用 2D/3D 扩散模型实现 Mesh 的形状与纹理编辑。

### 1.1 基于 TRELLIS/SLAT 骨干

| 方法 | Mesh | 纹理 | 简述 | 条件输入 | 是否需训练 |
|:---|:---:|:---:|:---|:---|:---:|
| **TRELLIS** | ✓ | — | 宏观结构按新条件控制生成；边界框局部重生成 | 文本/图像 | — |
| **[VoxHammer](voxhammer.md)** | ✓ | ✓ | DDIM 反演缓存未编辑区域 KV 特征，去噪时强制替换。提出 Edit3D-Bench | 编辑图像 + 3D mask | 免训练 |
| **[NANO3D](nano3d.md)** | ✓ | — | FlowEdit + Voxel-Merge/Slat-Merge 双层区域融合，mask-free | 编辑图像 | 免训练 |
| **[3DEditVerse](3deditverse.md)** | ✓ | ✓ | DualAttn + Time-Adaptive Gating，并行交叉注意力融合双路 3D 特征 | 编辑图像 | 需训练 |
| **[Steer3D](steer3d.md)** | ✓ | ✓ | ControlNet bypass + DPO 对齐，前馈式文本驱动 | 文本 | 需训练 |
| **[Easy3E](easy3e.md)** | ✓ | ✓ | Voxel FlowEdit 几何编辑 + ERA3D 多视角纹理精炼 + Ctrl-Adapter | 编辑图像 | 部分训练 |
| **[Native 3D Editing](native-3d-editing.md)** | ✓ | ✓ | Token Concatenation 条件注入，源/目标 token 全自注意力交互 | 文本 | 需训练 |

### 1.2 基于 VecSet/Flow 骨干

| 方法 | Mesh | 纹理 | 简述 | 条件输入 | 是否需训练 |
|:---|:---:|:---:|:---|:---|:---:|
| **[VecSet-Edit](vecset-edit.md)** | ✓ | ✓ | 在 TripoSG VecSet latent 上做 mask-guided token 级编辑：token seeding + attention gating + drift pruning | 编辑图像 + 2D mask | 基于预训练 LRM |
| **[AnchorFlow](anchorflow.md)** | ✓ | ✓ | 基于 Hunyuan3D 2.1，通过 latent anchor consistency 稳定 flow-based 编辑轨迹，mask-free | 文本 | 免训练 |

### 1.3 基于 2D 编辑提升的方法

| 方法 | Mesh | 纹理 | 简述 | 条件输入 | 是否需训练 |
|:---|:---:|:---:|:---|:---|:---:|
| **[CraftMesh](craftmesh.md)** | ✓ | ✓ | 2D 图像编辑 → 3D 重建 → 双阶段 Poisson 融合（几何+纹理），支持 PBR | 编辑图像 | 免训练 |
| **[PrEditor3D](preditor3d.md)** | ✓ | ✓ | 双路并行重建原始/编辑 3D 特征网格，特征层 copy-paste 融合 | 编辑图像 | 免训练 |
| **[Tailor3D](tailor3d.md)** | — | ✓ | 正/反双视图编辑 → Dual-sided LRM + LoRA 融合为 triplane-NeRF | 编辑图像 | 需训练 |
| **[MVEdit](mvedit.md)** | ✓ | ✓ | 2D 扩散去噪步间插入 3D 一致性桥接，多任务支持 | 图像/文本 | 部分训练 |

### 1.4 基于 Triplane/LRM 骨干

| 方法 | Mesh | 纹理 | 简述 | 条件输入 | 是否需训练 |
|:---|:---:|:---:|:---|:---|:---:|
| **[Instructive3D](instructive3d.md)** | — | ✓ | 在冻结 LRM 的 triplane latent 上加文本条件扩散 adapter，不需要配对 3D 编辑数据 | 文本 | 需训练 |
| **[Masked LRMs](masked-lrms.md)** | ✓ | — | 将编辑重构为条件掩码重建：训练 LRM 理解 3D 一致遮挡 mask，支持拓扑变化 | 编辑图像 + mask | 需训练 |

!!! note "TRELLIS 作为骨干"
    VoxHammer、NANO3D、3DEditVerse、Steer3D、Easy3E、Native 3D Editing 均直接基于 TRELLIS 架构。TRELLIS 的 SLAT 结构化潜变量表示为这些方法提供了统一的编辑基础。详见 [TRELLIS 分析](../generation/trellis.md)。

### 子路线对比

| 子路线 | 代表方法 | 核心思想 | 优势 | 主要问题 |
|:------|:---------|:---------|:-----|:---------|
| **Tuning-free latent editing** | VoxHammer, NANO3D, AnchorFlow | 利用预训练 3D latent / flow 反演直接编辑 | 不必重训大模型，工程门槛低 | 对编辑类型、区域定义较敏感 |
| **2D-guided 3D editing** | CraftMesh, PrEditor3D, 3DEditVerse | 先借力 2D 编辑，再回到 3D latent / reconstruction | 语义编辑能力强 | 2D-3D 一致性和区域保持难 |
| **Control-style native editing** | Steer3D, Easy3E | 在原生 3D backbone 上加入类似 ControlNet / repainting 约束 | 前馈、速度快 | 训练成本高，编辑自由度与保真度仍需平衡 |
| **Fully native 3D editing** | Native 3D Editing, VecSet-Edit | 直接在 3D latent 中改几何/外观，不依赖 2D 中间结果 | 最符合长期方向 | 需要足够强的 3D latent 和高质量编辑数据 |
| **LRM adapter editing** | Instructive3D, Masked LRMs | 在预训练重建模型上加编辑 adapter | 训练成本较低，复用强重建能力 | 受限于底层 LRM 的表示能力 |

---

## 2. 基于代码的方法

| 方法 | Mesh | 纹理 | 简述 | 条件输入 |
|:---|:---:|:---:|:---|:---|
| **[MeshCoder](meshcoder.md)** | ✓ | × | 点云 → Blender Python 脚本。构建大规模「3D 模型–代码」配对数据集，训练 LLM（Llama-3.2-1B）实现参数化建模 | 点云 |

---

## 3. 基于拓扑/几何处理的方法

| 方法 | Mesh | 纹理 | 简述 | 条件输入 |
|:---|:---:|:---:|:---|:---|
| **Poisson-Based (2005)** | ✓ | — | 经典几何处理框架：求解泊松方程操控梯度场，统一了变形、合并、平滑等任务 | 用户交互约束 |
| **[ShapeFusion](shapefusion.md)** | ✓ | — | 在 mesh 顶点空间直接做 masked diffusion：仅对选定局部区域加噪/去噪编辑，保持未编辑顶点不动 | 用户指定区域/控制点 |

---

## 4. 多模态/统一模型方法

| 方法 | Mesh | 纹理 | 简述 | 条件输入 | 是否需训练 |
|:---|:---:|:---:|:---|:---|:---:|
| **[ShapeLLM-Omni](shapellm-omni.md)** | ✓ | ✓ | 统一文本/图像/3D 生成与理解的多模态 LLM，通过 3D VQVAE 将 mesh 离散化为 token | 文本/图像/3D | 需训练 |

!!! warning "编辑能力说明"
    ShapeLLM-Omni 的编辑能力目前处于概念验证阶段，尚未达到工业级质量。其价值更多在于统一架构的探索方向。

---

## 5. 其他 Mesh 编辑/生成相关方法

以下方法虽不完全属于「Mesh 编辑」范畴，但在方法谱系中有参考价值：

| 方法 | 年份 | 表示 | 方法类型 | 编辑能力 | 原生 Mesh 友好 |
|:---|:---:|:---|:---|:---|:---:|
| **[Neural Cages](neural-cages.md)** | 2021 | mesh + neural cage | 前馈式 | cage-based 交互式几何形变 | 是 |
| **[INST-Sculpt](inst-sculpt.md)** | 2025 | neural SDF | 混合式 | 笔触式交互雕刻，局部优化更新神经 SDF | 部分 |
| **Sharp-It** | 2024 | multi-view | 前馈式 | 几何细节增强/修复 | 否 |
| **MeshGPT** | 2023 | 自回归 token | 前馈式 | 形状补全 | 是 |
| **[Text2Mesh](text2mesh.md)** | 2021 | 微调 | 优化式 | 文本→颜色+几何细节 | 是 |
| **[TEXTure](texture.md)** | 2023 | 微调 | 优化式 | 纹理+位移贴图 | 是 |
| **SKED** | 2023 | NeRF | 优化式 | Sketch-guided | 否 |
| **[MeshPad](meshpad.md)** | 2025 | 自回归 | 前馈式 | Sketch-guided 几何编辑 | 是 |

---

## 6. 方法分类维度

### 按编辑输入方式

- **文本指导**：TRELLIS, Steer3D, Native 3D Editing, AnchorFlow, Instructive3D, Text2Mesh, TEXTure — 灵活但控制精度有限
- **图像引导**：VoxHammer, CraftMesh, PrEditor3D, 3DEditVerse, Easy3E, NANO3D, VecSet-Edit, Masked LRMs — 利用 2D 编辑结果驱动 3D 编辑
- **草图引导**：SKED, MeshPad — 更直观精确的几何控制
- **交互式**：Neural Cages, INST-Sculpt, ShapeFusion — 实时形变与雕刻
- **多模态**：ShapeLLM-Omni — 文本/图像/3D 统一输入

### 按是否需要训练

- **Training-free**：VoxHammer, NANO3D, AnchorFlow, CraftMesh, PrEditor3D — 直接利用预训练模型
- **部分训练**：Easy3E（仅 Ctrl-Adapter）, MVEdit（StableSSDNeRF 微调）
- **需要训练**：3DEditVerse, Steer3D, Native 3D Editing, Instructive3D, Masked LRMs, ShapeLLM-Omni, MeshPad 等

### 按 3D 骨干表示

- **TRELLIS/SLAT**：VoxHammer, NANO3D, 3DEditVerse, Steer3D, Easy3E, Native 3D Editing
- **VecSet/Flow**：VecSet-Edit（TripoSG）, AnchorFlow（Hunyuan3D 2.1）
- **Triplane/LRM**：Instructive3D, Masked LRMs, Tailor3D, MVEdit
- **直接 mesh 操作**：ShapeFusion, Neural Cages, MeshPad, MeshCoder, Text2Mesh, TEXTure

---

## 7. 当前文献里反复出现的几个关键矛盾

### 1. 编辑强度 vs 未编辑区域保持

几乎所有论文都绕不开的矛盾：

- 编辑越强，越容易破坏非编辑区域；
- 约束越强，编辑幅度又会不够。

不同方法的解决策略：VoxHammer 通过缓存未编辑区域 latent；PrEditor3D 用特征替换；3DEditVerse 让模型通过门控学会平衡；AnchorFlow 通过稳定 latent anchor 减少编辑漂移。详见 [保留度评估](../evaluation/metrics.md)。

### 2. 2D 编辑语义强 vs 3D 结构一致性弱

2D 编辑模型能力强，但一旦进入 3D：

- 多视图容易不一致；
- 遮挡区域缺少可靠约束；
- 重建回 mesh 时局部会有伪影。

所以很多 3D 编辑论文实际上都在解决"如何把强大的 2D 编辑先验稳妥搬到 3D"。

### 3. Training-free 灵活 vs Learned editor 上限更高

- **Training-free** 方法部署快、成本低，适合快速验证；
- **需要训练** 的 editor 虽然重，但长期上限更高，更容易做复杂指令、区域理解和一致性建模。

这也是为什么 2025 之后越来越多工作转向构建 **3D editing datasets**（3DEditVerse 116K 对、NANO3D-Edit 100K、Steer3D 96K 对），而不再只做 tuning-free tricks。

### 4. 是否拥有"原生 3D 隐空间"几乎决定了方法上限

如果 backbone 只是 triplane reconstruction model 或 multi-view lifting pipeline，那么编辑往往还是"2D 能力外接到 3D"。

而一旦 backbone 拥有像 **SLAT / VecSet / 原生 3D latent** 这样的结构化表示，编辑才可能变成：

> 在 3D 里改 3D，而不是在 2D 里改完再猜 3D。

当前两条主要的 backbone 路线（TRELLIS/SLAT 和 Hunyuan3D/VecSet）各有侧重，但都在推动这个方向。

---

## 8. 小结

当前 mesh editing 文献可以粗略看成两条路线的竞争：

1. **2D-first**：先用成熟的 2D 编辑模型做语义修改，再想办法恢复到 3D。
2. **3D-native**：直接在原生 3D latent / 3D structure 上做编辑。

前者短期更强，因为 2D 模型成熟；后者长期更有潜力，因为它更可能同时解决：

- 局部控制；
- 未编辑区域保持；
- 多视图一致性；
- 真实 mesh 输出质量。

从这个角度看，TRELLIS 及其编辑后续工作的价值，在于它们率先让"native 3D editing"成为一条清晰可持续的路线。而 VecSet-Edit 和 AnchorFlow 则表明这条路线不局限于单一骨干——基于 TripoSG 和 Hunyuan3D 的原生 3D latent 同样可以支持高质量编辑。
