# Mesh Editing Landscape

Mesh 编辑方法全景。当前方法可按技术路线分为：**基于扩散/生成模型的方法**（主流）、**基于代码的方法**、**基于拓扑/几何处理的方法**，以及更广泛的**前馈式 / 基于优化 / 混合式**分类。

---

## 先看主线：Mesh Editing 方法是怎么演化的

过去两年，mesh editing 不是线性发展的，而是明显分成了三代：

### 第一代：优化式 / 几何处理式

代表：**Poisson-Based Mesh Editing、Neural Cages、Text2Mesh、TEXTure、SKED**

这类方法的共同点是：

- 或者直接在 mesh 顶点/梯度场上做几何处理；
- 或者把 2D 先验当作损失函数，在测试时对单个 3D 对象反复优化。

优点是控制比较直接，理论上能精细修改；缺点也非常明显：

- 慢；
- 每个输入都要重做一遍优化；
- 很难扩展到稳定、高保真、批量化 3D 编辑。

### 第二代：2D 编辑能力提升到 3D

代表：**MVEdit、Tailor3D、PrEditor3D、CraftMesh**

这类方法本质上承认一个现实：

> 2D 图像编辑远比 3D 编辑成熟，因此先把编辑操作放在 2D，再回升到 3D。

它们往往采用：

- 多视图图像编辑；
- 多视图一致性约束；
- 再通过重建模型回到 mesh / triplane / neural field。

优点是直接继承 2D 编辑模型的强大语义能力；问题是：

- 2D -> 3D 提升过程会带来不一致；
- 非编辑区域保持不容易；
- 结构控制常常不如原生 3D 表示稳定。

### 第三代：在原生 3D latent 上直接编辑

代表：**TRELLIS、VoxHammer、NANO3D、3DEditVerse、Steer3D、Easy3E、Native 3D Editing、VecSet-Edit**

这代方法真正的转折点，是 **TRELLIS/SLAT** 这类原生 3D 结构化 latent 的出现。因为一旦 3D 表示本身具备：

- 稀疏空间结构；
- 局部 latent；
- 可局部替换、局部重绘；

那么很多编辑问题就不必再绕回 2D。

因此这一代的核心思想是：

> 尽量把编辑操作直接放在 3D latent / 3D structure 上，而不是仅把 2D 编辑结果“投影”回去。

---

## 1. 基于扩散/生成模型的 Mesh 编辑方法（主流）

这些方法利用 2D/3D 扩散模型实现 Mesh 的形状与纹理编辑。多数以 **TRELLIS** 为骨干架构。

| 方法 | Mesh | 纹理 | 简述 | 图片依赖 |
|:---|:---:|:---:|:---|:---:|
| **TRELLIS** | ✓ | — | 两种编辑模式：宏观结构上按新条件控制生成；边界框局部重生成 | 否 |
| **VoxHammer** | ✓ | ✓ | Training-free，基于 TRELLIS。扩散反演缓存未编辑区域特征，去噪时强制使用缓存特征保留未编辑区。先生成图像后生成 3D，需用户给出 3D 掩码。提出 Edit3D-Bench | 间接 |
| **CraftMesh** | ✓ | ✓ | 先编辑局部图像再提升到 3D，泊松几何融合平滑表面，保留非编辑区 | 间接 |
| **NANO3D** | ✓ | — | 基于 TRELLIS，train-free 的 editing pipeline | 直接 |
| **3DEditVerse** | — | — | 提出 3DEditFormer，基于 TRELLIS + DualAttn。先在 2D 图像编辑，利用 2D 结构特征与原始 3D 微观特征 | 直接 |
| **PrEditor3D** | ✓ | ✓ | 编辑 2D 多视角图像 → 并行重建原始/编辑后 3D 特征 → 特征层面替换修复 → GTR 解码 | 直接 |
| **Steer3D** | — | — | 输入编辑前 2D 视图 + 编辑指令，输出编辑后 Mesh。本质上类似单视图生成，无法保证非编辑区一致性 | 否 |
| **Easy3E** | — | — | （待补充详细信息） | — |

!!! note "TRELLIS 作为骨干"
    VoxHammer、NANO3D、3DEditVerse、Steer3D 均直接基于 TRELLIS 架构。TRELLIS 的 SLAT 结构化潜变量表示为这些方法提供了统一的编辑基础。详见 [TRELLIS 深度解析](../generation/trellis.md)。

### 这一支方法的内部细分

| 子路线 | 代表方法 | 核心思想 | 优势 | 主要问题 |
|:------|:---------|:---------|:-----|:---------|
| **Tuning-free latent editing** | TRELLIS, VoxHammer, NANO3D | 利用预训练 3D latent / flow 反演直接编辑 | 不必重训大模型，工程门槛低 | 对编辑类型、区域定义较敏感 |
| **2D-guided 3D editing** | CraftMesh, PrEditor3D, 3DEditVerse | 先借力 2D 编辑，再回到 3D latent / reconstruction | 语义编辑能力强 | 2D-3D 一致性和区域保持难 |
| **Control-style native editing** | Steer3D, Easy3E | 在原生 3D backbone 上加入类似 ControlNet / repainting 约束 | 前馈、速度快 | 训练成本高，编辑自由度与保真度仍需平衡 |
| **Fully native 3D editing** | Native 3D Editing, VecSet-Edit | 直接在 3D latent 中改几何/外观，不依赖 2D 中间结果 | 最符合长期方向 | 需要足够强的 3D latent 和高质量编辑数据 |

---

## 2. 基于代码的方法

| 方法 | Mesh | 纹理 | 简述 | 图片依赖 |
|:---|:---:|:---:|:---|:---:|
| **MeshCoder** | ✓ | × | 将点云重建为结构化、可编辑的 Blender Python 脚本。核心是构建大规模「3D 模型-代码」配对数据集，训练 LLM 实现点云到代码的转换 | 否（基于点云） |

---

## 3. 基于拓扑/几何处理的方法

| 方法 | Mesh | 纹理 | 简述 | 图片依赖 |
|:---|:---:|:---:|:---|:---:|
| **Poisson-Based (2005)** | ✓ | — | 经典几何处理框架：通过求解泊松方程操控梯度场来改变 Mesh 几何，统一了变形、合并、平滑等任务。用户交互约束被传播并生成新梯度场，进而重构 Mesh | 不利用 |
| **ShapeFusion** | — | — | 在 Mesh 顶点空间直接操作的扩散式生成方法：对选定区域加噪、在顶点空间编辑，保持未编辑部分不受影响。用于人脸/人体局部变形、表情操控 | — |

---

## 4. 其他 Mesh 编辑/生成相关方法

以下方法虽不完全属于「Mesh 编辑」范畴，但在方法谱系中有重要参考价值：

| 方法 | 年份 | 表示 | 方法类型 | 编辑能力 | 原生 Mesh 友好 | 关键点 |
|:---|:---:|:---|:---|:---|:---:|:---|
| **Neural Cages** | 2021 | mesh + neural cage | 前馈式 | cage-based 交互式几何形变 | **是** | 学习神经网络预测权重，用简单控制笼对复杂网格实时、保细节的交互式形变 |
| **MVEdit** | 2024 | multi-view → mesh | 前馈式 | 多视图条件编辑 | 否 | 将 2D diffusion 编辑能力扩展到 multi-view，结合重建 |
| **INST-Sculpt** | 2025 | neural SDF | 混合式 | 交互式雕刻 | 部分 | 类似 ZBrush 笔触式交互工具，每个笔触触发局部优化更新神经 SDF |
| **Tailor3D** | 2024 | dual-side images → triplane | 前馈式 | 正/反视图编辑→重建 | 否 | 用可编辑的正反两视图作为编辑接口，减少视图冲突 |
| **Sharp-It** | 2024 | multi-view → multi-view | 前馈式 | 几何细节增强/修复 | 否 | 并行增强多视图以得到更高质量重建 |
| **MeshGPT** | 2023 | 自回归 token | 前馈式 | 形状补全 | **是** | 直接生成三角形 token 序列，不经过隐式 + 抽壳 |
| **Text2Mesh** | 2021 | 微调 | 优化式 | 文本→颜色+几何细节 | 是 | CLIP 引导测试时优化，每次输入需重新优化 |
| **TEXTure** | 2023 | 微调 | 优化式 | 纹理+位移贴图 | 是 | 纹理贴图 + 微小几何修改 |
| **SKED** | 2023 | NeRF | 优化式 | Sketch-guided | 否 | 草图 + 文本引导，迭代优化 NeRF |
| **MeshPad** | 2025 | 自回归 | 前馈式 | Sketch-guided 几何编辑 | **是** | 基于 MeshAnythingV2，直接生成三角 Mesh |

---

## 5. 方法分类维度

### 按编辑输入方式

- **文本指导**：TRELLIS, Steer3D, Text2Mesh, TEXTure — 灵活但控制精度有限
- **图像引导**：VoxHammer, CraftMesh, PrEditor3D, 3DEditVerse — 利用 2D 编辑结果驱动 3D 编辑
- **草图引导**：SKED, MeshPad — 更直观精确的几何控制
- **交互式**：Neural Cages, INST-Sculpt — 实时形变与雕刻

### 按是否需要训练

- **Training-free**：VoxHammer, NANO3D — 直接利用预训练模型，无需额外训练
- **需要训练**：3DEditVerse (DualAttn), PrEditor3D, MeshPad 等

### 按 2D→3D 路径

- **直接 3D 编辑**：NANO3D, 3DEditVerse, PrEditor3D, MeshPad
- **间接（先 2D 再提升）**：VoxHammer, CraftMesh, Text2Mesh, TEXTure

---

## 6. 当前文献里反复出现的几个关键矛盾

### 1. 编辑强度 vs 未编辑区域保持

这是几乎所有论文都绕不开的矛盾：

- 编辑越强，越容易破坏非编辑区域；
- 约束越强，编辑幅度又会不够。

VoxHammer 通过缓存未编辑区域 latent 来解决；PrEditor3D 用特征替换；3DEditVerse 则希望让模型学会在数据层面理解“哪里该改、哪里不该改”。

### 2. 2D 编辑语义强 vs 3D 结构一致性弱

2D 编辑模型非常强，但一旦进入 3D：

- 多视图容易不一致；
- occluded 区域缺少可靠约束；
- 重建回 mesh 时局部会有伪影。

所以很多 3D 编辑论文实际上都在解决“如何把强大的 2D 编辑先验稳妥搬到 3D”。

### 3. Training-free 灵活 vs Learned editor 上限更高

- **Training-free** 方法部署快、成本低，适合快速验证和工程落地；
- **需要训练** 的 editor 虽然重，但长期上限更高，更容易做复杂指令、区域理解和一致性建模。

这也是为什么 2025 之后开始越来越多工作转向构建 **3D editing datasets**，而不再只做 tuning-free tricks。

### 4. 是否拥有“原生 3D 隐空间”几乎决定了方法上限

这一点和 mesh generation 主线是统一的。

如果 backbone 只是：

- triplane reconstruction model，或
- multi-view lifting pipeline，

那么编辑往往还是“2D 能力外接到 3D”。

而一旦 backbone 拥有像 **SLAT / VoxSet / 原生 3D latent** 这样的结构化表示，编辑才真正可能变成：

> 在 3D 里改 3D，而不是在 2D 里改完再猜 3D。

---

## 7. 一个简化结论

当前 mesh editing 文献可以粗略看成两条路线的竞争：

1. **2D-first**：先用成熟的 2D 编辑模型做语义修改，再想办法恢复到 3D。
2. **3D-native**：直接在原生 3D latent / 3D structure 上做编辑。

前者短期更强，因为 2D 模型成熟；后者长期更重要，因为它更可能同时解决：

- 局部控制；
- 未编辑区域保持；
- 多视图一致性；
- 真实 mesh 输出质量。

从这个角度看，TRELLIS 及其编辑后续工作之所以重要，并不只是因为结果好，而是因为它们第一次让“native 3D editing”成为一条清晰可持续的路线。
