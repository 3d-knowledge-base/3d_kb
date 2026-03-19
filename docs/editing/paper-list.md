---
comments: true
---

# Editing Paper List

本页收录 3D Editing 方向的典型文献，按技术路线分组。每篇论文附有简要中文描述；如有详情页则提供链接。

---

## Native 3D Latent Editing（原生 3D 潜空间编辑）

在 TRELLIS 等原生 3D 潜空间骨干上直接操作，避免 2D→3D 提升带来的一致性问题。这是当前最活跃的编辑研究方向。

### VoxHammer (2025.08) → [详情页](voxhammer.md)

无需训练的 3D 编辑方法，基于 TRELLIS 骨干。核心流程是对原始 3D 资产进行 DDIM 反演获取噪声潜变量，再用编辑后的条件重新去噪。技术特点是 KV Cache Replacement 机制：在去噪过程中用原始资产的注意力 KV 缓存替换未编辑区域的对应值，实现精确的局部编辑 + 全局保留。同时提出 Edit3D-Bench 评估基准，包含三维度指标体系（保留度、整体质量、条件对齐）。

### NANO3D (2025.10) → [详情页](nano3d.md)

同样基于 TRELLIS 的无训练编辑方法，但采用不同的技术路线：FlowEdit（而非 DDIM 反演）作为扩散编辑算法，Voxel-Merge 在结构层融合编辑/未编辑体素，Slat-Merge 在潜变量层做精细混合。还提出 Nano3D-Edit-100k 大规模编辑数据集（100K 对），为后续监督式编辑方法提供数据基础。

### 3DEditVerse (2025.10) → [详情页](3deditverse.md)

数据驱动路线的代表作。构建了 116K 编辑对的大规模训练数据集，提出专用编辑模型 3DEditFormer：在 TRELLIS 解码器内部插入 DualAttn（交叉注意力融合源资产信息 + 编辑指令）和 Time-Adaptive Gating（根据去噪时间步动态调节编辑强度）。无需掩码输入，端到端学习编辑位置和编辑内容。

### Steer3D (2025.12) → [详情页](steer3d.md)

前馈式（feed-forward）3D 编辑方法，采用 ControlNet 风格的旁路架构接入 TRELLIS 骨干。训练数据 96K 对，通过双阶段过滤（Dual-LLM 语义正确性检查 + DreamSim 感知一致性过滤）确保数据质量。引入 DPO（Direct Preference Optimization）进一步对齐人类偏好。推理仅需 11.8 秒，是目前最快的原生 3D 编辑方法之一。

### Easy3E (2026.02) → [详情页](easy3e.md)

完全前馈的单视图 3D 编辑流水线。技术特点是 Voxel FlowEdit：将 FlowEdit 算法从 2D 扩展到 3D 体素潜空间，配合轮廓引导（silhouette guidance）和轨迹一致性（trajectory consistency）保持编辑稳定性。纹理精化阶段使用 ERA3D 多视角扩散模型。整个流程无需逐样本优化，全程前馈。

### Native 3D Editing (2025.11) → [详情页](native-3d-editing.md)

直接在 TRELLIS 的 SLAT 潜空间中进行编辑的前馈方法。核心发现是 Token Concatenation 策略（将源资产 token 和噪声 token 拼接输入）优于 Cross-Attention 等替代方案。训练数据通过 Objaverse 部件拆分 + Hunyuan3D 2.1 重建构建。

### VecSet-Edit (2026.02)

在基于集合的 3D 潜表示（VecSet）上进行编辑，与基于体素的方法形成互补。探索了非体素结构化潜空间的编辑可能性。目前无详情页。

---

## 2D-Guided / Lifting-style Editing（2D 引导提升式编辑）

先在 2D 图像空间进行编辑，再提升回 3D。方法成熟但面临多视角一致性挑战。

### PrEditor3D (2024.12) → [详情页](preditor3d.md)

无训练的精确 3D mesh 编辑方法，核心策略是"两路并进，择优合并"。先在 4 个正交视角上使用 MVDream + Prompt-to-Prompt 进行多视图一致的 2D 编辑，再通过 Grounding DINO + SAM 2 精确检测编辑区域。技术特点是 3D 阶段的双路重建：同时重建原始和编辑版本的完整 3D 特征网格（$V_i$ 和 $V_e$），然后在特征空间用精确掩码进行区域替换 + 高斯模糊边界融合。约 74 秒完成一次编辑。

### CraftMesh (2025.09) → [详情页](craftmesh.md)

纯推理时流水线，不需要任何训练。先对参考图像进行 2D 编辑，再通过 mesh 生成模型重建编辑后的 3D 形状。技术特点是双重泊松融合：Poisson Geometric Fusion 在几何层面将编辑区域与原始 mesh 无缝拼接，Poisson Texture Harmonization 在纹理层面消除接缝，实现编辑区域与保留区域的自然过渡。

### MVEdit (2024.03)

多视角图像编辑后提升回 3D 的方法。在多个视角上同时进行 2D 编辑，再通过多视角一致性约束融合回 3D 表示。目前无详情页。

### Tailor3D (2024.07)

采用双视图编辑策略减少正面/背面不一致问题，是 2D 提升路线中处理多视角一致性的一种实用方案。目前无详情页。

### Text2Mesh (2021.12)

早期 CLIP 引导的优化式 mesh 编辑方法。通过 CLIP 损失驱动 mesh 顶点位移和颜色优化，使 mesh 外观匹配文本描述。开创了文本引导 3D 编辑的研究方向，但优化速度慢且编辑精度有限。目前无详情页。

### TEXTure (2023.02)

基于深度引导的纹理编辑与重绘方法。利用深度信息控制 2D 扩散模型在指定视角生成纹理，再投射回 3D 表面。主要聚焦纹理层面的编辑，几何形状保持不变。目前无详情页。

### SKED (2023.08)

草图/文本引导的优化式 3D 编辑方法，结合 SDS（Score Distillation Sampling）损失和用户草图约束进行 NeRF 优化。允许更精确的空间控制，但优化过程耗时较长。目前无详情页。

---

## Geometry / Interactive / Other（几何编辑 / 交互式编辑 / 其他）

### MeshPad (2025.03) → [详情页](meshpad.md)

基于草图的交互式 mesh 编辑方法，建立在 MeshAnythingV2 自回归 mesh 生成框架上。定义了 Addition（添加新几何元素）和 Deletion（删除已有部分）两种原子操作，用户通过绘制草图指定编辑意图，模型自回归生成编辑后的 mesh。是 mesh-native 编辑路线的代表。

### MeshCoder (2025.08) → [详情页](meshcoder.md)

将 3D 编辑转化为代码生成问题：输入点云描述，输出可执行的 Blender Python 脚本。构建了约 100 万对的 paired 数据集（3D 模型 ↔ Blender 脚本）。虽非直接编辑方法，但产出的是完全可编辑的参数化表示，连接了 3D 编辑与程序化建模。

### ShapeFusion (2024.03)

在顶点空间进行扩散编辑的方法，通过局部保留约束限制编辑范围。直接操作 mesh 顶点坐标，不依赖中间潜表示。目前无详情页。

### Poisson-Based Mesh Editing (2005)

经典梯度场 mesh 编辑算法，至今仍是许多方法的基础组件（如 CraftMesh 的泊松融合）。核心思想是通过求解泊松方程保持梯度场约束，实现形变后的表面平滑性。目前无详情页。

### Neural Cages (2019)

基于控制笼（cage）的神经网络可控形变方法。用户操作低多边形笼体，神经网络学习将笼体形变映射为高精度 mesh 形变。目前无详情页。

### INST-Sculpt (2025.02)

在神经 SDF 表示上进行交互式雕刻的方法，允许用户像使用数字雕刻工具一样直接操作隐式表面。目前无详情页。

### AnchorFlow (2025.11)

基于锚点流的 3D 编辑方法，通过定义锚点上的变形流场驱动 mesh 形变。目前无详情页。

### Instructive3D (2025.01)

指令驱动的 3D 编辑方法，通过自然语言指令控制 3D 资产的修改。目前无详情页。

### Masked LRMs (2024.12)

通过掩码策略在大规模重建模型（LRM）上实现局部编辑：遮住需要编辑的区域，让 LRM 根据新条件重建该区域，从而实现编辑效果。目前无详情页。

### ShapeLLM-Omni

将大语言模型与 3D 形状理解和操作结合的多模态方法，支持通过对话进行 3D 资产的理解和编辑。目前无详情页。
