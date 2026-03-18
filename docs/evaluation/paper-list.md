# Evaluation Paper List

本页收录 3D Evaluation 方向的代表性文献与评估框架，按类别分组。每条附有简要中文描述；如有详情页则提供链接。

---

## Benchmark / Evaluator Papers

### 3D Arena (2024) → [详情页](benchmarks.md)

基于众包偏好投票的 3D 生成评估平台。采用匿名成对比较 + ELO 评分系统，累计超过 12 万次社区投票。评估粒度为单一综合质量（"哪个更好"），适合大规模模型快速排名，但无法区分几何、纹理、对齐等维度的差异。

### T3Bench (2023) → [详情页](benchmarks.md)

面向 Text-to-3D 的自动化多视角评估基准。设计了分层提示集（单对象 → 带环境单对象 → 多对象），核心创新是**区域卷积（Regional Convolution）**检测机制，专门捕捉传统 CLIP 平均分无法发现的 Janus（多面人）问题。同时通过多视图字幕 + GPT-4 判断语义对齐性。

### MATE-3D / HyperScore (2024) → [详情页](benchmarks.md)

从"哪个更好"到"好到什么程度"的范式转变。采用多维度绝对评分（MOS 0-10），涵盖语义对齐、几何质量、纹理质量和综合质量四个独立维度。提出 HyperScore 超网络评估器：根据评估维度条件动态生成预测头权重，单模型即可输出多维度专门分数。标注规模达 107,520 个独立评分。

### 3DGen-Bench (2025) → [详情页](benchmarks.md)

首次统一 Text-to-3D 和 Image-to-3D 两大任务的综合评估基准。规模为 1,020 提示 × 19 模型 = 11,220 个 3D 资产，混合众包投票（68K）和专家多维度标注（56K）。提供互补工具套件：3DGen-Score（基于 CLIP 的快速排序工具）和 3DGen-Eval（基于 MLLM 的可解释诊断工具）。

### Eval3D (2025) → [详情页](benchmarks.md)

革命性零样本评估范式：不依赖人类偏好标注训练，而是利用基础模型（DINOv2、Depth Anything、Zero-1-to-3 等）作为客观"探针"。核心逻辑是高质量 3D 资产的不同属性应内在自洽。评估五大一致性维度（几何、语义、结构、文本对齐、美学），能提供像素级空间反馈，实现可解释的缺陷定位。

### Hi3DEval (2025) → [详情页](benchmarks.md)

评估粒度推向极致的分层级诊断框架。评估从对象级（整体几何/纹理/对齐）→ 部件级（语义部件质量）→ 材质级（PBR 物理材质属性），首次将反照率、金属度等材质维度纳入 3D 生成评估。规模覆盖 30 个模型、15,300 资产，配合 M²AP 多智能体自动化标注流程。

---

## Metric Families

### 几何质量指标 → [详情页](metrics.md)

包括 Chamfer Distance (CD)、Hausdorff Distance (HD)、F-score、Volume/Surface IoU 等。衡量生成或编辑结果与参考形状的几何保真度，是 3D 评估最基础的一类指标。CD 测量平均点云距离，F-score 衡量表面重合度，IoU 分体积和表面两种变体分别反映宏观轮廓和精细细节。

### 外观质量指标 → [详情页](metrics.md)

包括 LPIPS、SSIM、PSNR、FID、CMMD 等。通过多视角渲染图评估纹理和视觉质量。LPIPS 基于深度网络模拟人类感知，FID 衡量生成分布与真实分布的距离，CMMD 在 CLIP 特征空间中计算分布距离，对纹理细节更敏感。

### 条件对齐指标 → [详情页](metrics.md)

包括 CLIP Score、Directional CLIP Score、DINO-I、ULIP/Uni3D 等。衡量生成/编辑结果与输入条件（文本、图像）的匹配程度。Directional CLIP Score 特别适合编辑任务：它衡量的是文本变化方向与渲染图变化方向在特征空间中的一致性，而非简单的绝对匹配。

### 编辑保留度指标 → [详情页](metrics.md)

包括 Masked CD、Masked LPIPS/PSNR/SSIM、CLIPdiff-noedit 等。通过掩码仅计算未编辑区域的变化，是衡量编辑方法"泄露"程度的直接度量。3D 编辑的核心难题之一就是在编辑目标区域的同时保持其余部分不变——这类指标直接量化这一能力。
