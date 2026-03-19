---
comments: true
---

# Metrics

3D 生成与编辑的评估指标汇总。按评估维度组织，标注各指标的使用论文与优劣判断方向。

---

## 1. 条件对齐度 (Condition Alignment)

衡量生成/编辑结果与输入条件（文本、图像、草图）的匹配程度。

| 指标 | 评估内容 | 使用论文 | 方向 |
|:---|:---|:---|:---:|
| **Directional CLIP Score (CLIPdir)** | 文本指令变化方向与 3D 模型渲染图在特征空间中变化方向的一致性 | PrEditor3D | ↑ |
| **CLIP Score / CLIP-T** | 编辑后模型渲染图与目标文本的语义相似度 | VoxHammer, Hunyuan3D 2.0, TRELLIS, MeshPad | ↑ |
| **DINO Image Similarity (DINO-I)** | 图像驱动编辑中，3D 模型渲染图与用户提供的 2D 目标图像的相似度 | VoxHammer | ↑ |
| **ULIP / Uni3D** | 生成 3D 模型与给定条件的跨模态语义和结构相似性（T: 文本, I: 图像子类型） | Hunyuan3D 2.0 | ↑ |
| **GPTEval3D / User Study (匹配度)** | 人类或 GPT-4V 主观判断编辑结果符合指令的程度 | PrEditor3D, VoxHammer, MeshPad | ↑ |

---

## 2. 保留度 (Preservation)

衡量编辑过程中**未编辑区域**的保持程度——3D 编辑的核心难题。

### 2.1 保留度指标总览

| 指标 | 评估内容 | 使用论文 | 方向 |
|:---|:---|:---|:---:|
| **Masked Chamfer Distance (CD)** | 未编辑区域点云在编辑前后的几何形状保持 | VoxHammer, PrEditor3D | ↓ |
| **Masked LPIPS / PSNR / SSIM** | 渲染图上未编辑区域的外观、纹理和结构保留 | VoxHammer | LPIPS: ↓, PSNR/SSIM: ↑ |
| **CLIPdiff-noedit** | 未编辑区域在编辑前后的 CLIP 分数变化（语义内容保持） | PrEditor3D | ↓ |

!!! tip "Masked 指标的意义"
    保留度指标通过掩码仅计算未编辑区域，是衡量编辑方法「泄露」程度的直接度量。Masked CD 衡量几何保持，Masked LPIPS 衡量视觉保持，CLIPdiff-noedit 衡量语义保持。

### 2.2 保留机制分类

各方法在技术上通过不同机制实现未编辑区域的保留，大体分为两类。

#### 显式融合（需要 mask / 区域信息）

| 论文 | 机制 | 技术细节 |
|:---|:---|:---|
| **VoxHammer** | DDIM 反演 + KV Cache Replacement | 对源 mesh 做 DDIM 反演，缓存未编辑区域的 KV 特征；去噪阶段将未编辑区域的 KV 替换回缓存值 |
| **PrEditor3D** | 双路重建 + Copy-Paste 融合 | 同时重建源 mesh 和编辑后 mesh 的 3D 特征网格，根据 2D 编辑 mask 反投影得到的 3D mask 进行区域级 copy-paste |
| **CraftMesh** | Poisson 几何 + 纹理融合 | 几何层面将编辑部件无缝嫁接到源 mesh 上，纹理层面在边界处混合颜色 |
| **Easy3E** | 轨迹级分离 | 编辑区域沿编辑轨迹采样，非编辑区域沿源轨迹采样，通过轮廓引导确定编辑 voxel |

#### 隐式保留（不需要用户输入 mask）

| 论文 | 机制 | 技术细节 |
|:---|:---|:---|
| **NANO3D** | Voxel-Merge + Slat-Merge 双层融合 | 通过体素级 XOR 差异自动计算伪 mask，在 Voxel 和 SLAT 特征层分别融合 |
| **3DEditVerse** | DualAttn + Time-Adaptive Gating | 两种 3D 特征并行交叉注意力融合，门控系数随去噪时间步动态调节保留/编辑平衡 |
| **Native 3D Editing** | Token Concatenation | 将源 mesh 的 SLAT token 与噪声 token 拼接，通过全自注意力交互。消融实验证明优于 Cross-Attention |
| **Steer3D** | ControlNet bypass | 在 TRELLIS 主干旁加 ControlNet 分支注入源 mesh 信息，通过 DPO 部分缓解保留性不足 |
| **AnchorFlow** | Latent Anchor Consistency | 全局 latent anchor 对齐源/目标轨迹，anchor-alignment loss 隐式约束非编辑区域 latent 一致性 |

#### 两类机制对比

| 类别 | 代表方法 | 优势 | 劣势 |
|:---|:---|:---|:---|
| 显式融合 | VoxHammer, PrEditor3D, CraftMesh | 保留度高，可量化控制 | 需要精确 3D mask，工作流复杂 |
| 隐式/自动融合 | 3DEditVerse, Native 3D Editing, NANO3D, AnchorFlow | 不需要用户 mask，端到端 | 保留程度依赖内部机制，部分方法可能出现编辑泄露 |

### 2.3 各论文保留度评估方法对比

各论文的保留度评估分为**基于 2D 渲染图**和**基于 3D 点云/网格**两类。

| 论文 | 2D 指标 | 渲染视角数 | 3D 指标 | 采样点数 | 评估类型 |
|:---|:---|:---|:---|:---|:---|
| **VoxHammer** | Masked PSNR / SSIM / LPIPS, FID, FVD, DINO-I, CLIP-T | 未明确 | Masked CD | 未明确 | 显式 mask-based（2D+3D） |
| **3DEditVerse** | PSNR, SSIM, LPIPS, DINO-I | 10 个固定视角 | CD, NC, F1^0.01 | 100,000 点 | 无独立保留度指标，整体对比 GT |
| **Steer3D** | LPIPS | 6 个视角 | CD, F1 (阈值 0.05) | 10,000 点 | 无独立保留度指标，整体对比 GT |
| **PrEditor3D** | CLIPdiff-noedit, GPTEval3D | 未明确 | CD（消融实验） | 未明确 | 语义级（CLIPdiff-noedit） |
| **Native 3D Editing** | FID, FVD, CLIP | 未明确 | 无 | — | 无独立保留度指标 |
| **Easy3E** | CLIP-T, DINO-I, LPIPS, FID | 未明确 | 无 | — | 无独立保留度指标 |
| **NANO3D** | FID, DINO-I | 未明确 | CD | 未明确 | 用户研究（95% 形状保留偏好） |
| **MeshPad** | LPIPS, CLIP, FID | 未明确 | CD | 未明确 | 用户研究 EC（编辑一致性） |

??? info "具体保留度指标详解"

    #### 基于 3D mask 的指标（VoxHammer Edit3D-Bench）

    VoxHammer 提出了 Edit3D-Bench（100 模型 × 3 指令 = 300 编辑任务），每个样本包含人工标注的 3D 编辑区域 mask。

    - **Masked CD**（3D 指标）：仅在未编辑区域的点云上计算 Chamfer Distance
    - **Masked PSNR / SSIM / LPIPS**（2D 指标）：将 3D mask 投影到各渲染视角得到 2D mask，仅在非 mask 区域计算

    | 指标 | 评估维度 | 方向 |
    |:---|:---|:---:|
    | Masked CD | 几何保留 | ↓ |
    | Masked PSNR | 像素级外观保留 | ↑ |
    | Masked SSIM | 结构级外观保留 | ↑ |
    | Masked LPIPS | 感知级外观保留 | ↓ |

    #### 基于语义的指标

    - **CLIPdiff-noedit**（PrEditor3D）：计算未编辑区域在编辑前后的 CLIP 分数变化。值越低表示语义内容保持越好。不需要精确像素级 mask

    #### 用户研究

    - **EC（Edit Consistency）**（MeshPad）：用户对编辑一致性打分（1-5）
    - **Shape Preservation preference**（NANO3D）：A/B 对比，用户选择哪个方法更好地保留了形状

### 2.4 姿态/形变编辑的保留度指标

对于姿态/形变编辑（如"人物举起手臂"），上述基于 mask 的保留度指标**全部失效**：整个 mesh 都在移动，无法划定"未编辑区域"。但"保留度"仍然有意义——五官、衣服纹理、身体比例应该不变。

合理的姿态变化在微分几何中属于**近似等距变形**（isometric deformation）——表面弯曲但不拉伸。基于此性质，可用内在几何不变量评估保留度。

#### LBO 频谱距离（ShapeDNA）— 全局指标

比较源 mesh 和预测 mesh 的 Laplace-Beltrami 算子特征值序列。特征值按大小排序，天然对齐，不需要顶点对应关系，不需要相同拓扑。

$$d_{\text{spectral}}(M_1, M_2) = \left\| \frac{\lambda^{(1)}}{\text{Area}(M_1)} - \frac{\lambda^{(2)}}{\text{Area}(M_2)} \right\|_2$$

其中 $\lambda^{(i)} = (\lambda_1^{(i)}, \lambda_2^{(i)}, \ldots, \lambda_K^{(i)})$ 是前 K 个 LBO 特征值，除以表面积做尺度归一化。

特点：

- 输出一个标量，衡量全局内在形状是否保持
- 不需要顶点对应，不需要相同拓扑，不需要 mask
- 低特征值（前 10–20 个）反映全局形状，高特征值反映局部细节

#### HKS 特征空间 Chamfer Distance — 分布级指标

Heat Kernel Signature（HKS）是逐顶点的内在几何描述符，对等距变形不变。由于生成式方法输出的 mesh 拓扑通常不同，无法逐顶点比较，因此在 HKS 特征空间（而非 XYZ 坐标空间）计算 Chamfer Distance。

$$\text{HKS-CD}(M_1, M_2) = \frac{1}{|V_1|} \sum_{v \in V_1} \min_{u \in V_2} \|h(v) - h(u)\|^2 + \frac{1}{|V_2|} \sum_{u \in V_2} \min_{v \in V_1} \|h(u) - h(v)\|^2$$

其中 $h(v) \in \mathbb{R}^S$ 是顶点 $v$ 的 HKS 描述子向量（S 个时间尺度）。

特点：

- 不需要顶点对应关系和相同拓扑
- 衡量"局部几何特征的分布是否一致"
- 对称物体的左右对称部位 HKS 相同，Chamfer 会互相匹配

### 2.5 保留度评估的局限性

所有现有保留度指标都有一个核心假设：**能够确定"哪里被编辑了，哪里没被编辑"**。

- Masked CD/PSNR/SSIM/LPIPS 需要显式 3D mask
- CLIPdiff-noedit 需要区分编辑/非编辑区域
- 用户研究依赖人类主观判断

对于姿态/形变编辑，基于 mask 的方法失效。基于内在几何的指标（ShapeDNA, HKS-CD）提供了一种补充方案，但也有局限：

1. **生成式方法不保证等距变形**：TRELLIS 等模型从隐空间解码生成 mesh，即使姿态编辑正确，解码器也可能引入局部拉伸/压缩
2. **更适合方法间比较而非绝对评分**：ShapeDNA 和 HKS-CD 的绝对值难以直接解释，但作为横向对比指标有效
3. **LBO 对 mesh 分辨率敏感**：不同分辨率的 mesh 高阶特征值差异较大，建议在计算前统一重采样或只使用前 20–30 个低阶特征值
4. **非等距形变会被检测到**：如果编辑指令本身就改变内在几何（如"变胖"），高 ShapeDNA 距离不代表保留度差，需要根据编辑类型分别解读

---

## 3. 几何质量 (Geometry Quality)

衡量 3D 形状的整体质量——重建保真度与生成质量。

| 指标 | 评估内容 | 使用论文 | 方向 |
|:---|:---|:---|:---:|
| **Chamfer Distance (CD)** | 两个点云间的平均距离，评估整体形状相似度 | TRELLIS, VoxHammer, PrEditor3D, MeshPad | ↓ |
| **Edge Chamfer Distance (ECD)** | 尖锐边缘和角点附近采样点云间的 CD，评估细节保持 | MeshAnything | ↓ |
| **V-IoU (Volume IoU)** | 重建模型与原始模型的体积重合度（宏观轮廓准确性） | Hunyuan3D 2.0 | ↑ |
| **S-IoU (Surface IoU)** | 重建模型与原始模型的表面贴合度（精细细节敏感） | Hunyuan3D 2.0 | ↑ |
| **F-score** | 两个形状的表面重合度（精确率+召回率） | TRELLIS, X-Part | ↑ |
| **Normal Consistency (NC)** | 网格表面法线质量，反映平滑度和细节 | MeshAnything | ↑ |

---

## 4. 外观/纹理质量 (Appearance & Texture)

衡量表面外观的视觉质量。

| 指标 | 评估内容 | 使用论文 | 方向 |
|:---|:---|:---|:---:|
| **LPIPS** | 基于深度神经网络的感知相似度（模拟人类视觉系统） | TRELLIS, Hunyuan3D 2.0, MeshPad, VoxHammer | ↓ |
| **LPIPS-N** | 法线图上的感知相似度（评估表面细节） | TRELLIS | ↓ |
| **PSNR** | 逐像素差异，经典图像质量指标 | TRELLIS | ↑ |
| **PSNR-N** | 法线图上的 PSNR（表面细节重建保真度） | TRELLIS | ↑ |
| **CMMD** | CLIP 特征空间中两组图像的分布距离（对细节敏感） | Hunyuan3D 2.0 | ↓ |

---

## 5. 整体质量与真实感 (Overall Quality & Realism)

衡量生成结果的整体质量、真实感和多样性。

| 指标 | 评估内容 | 使用论文 | 方向 |
|:---|:---|:---|:---:|
| **FID (Fréchet Inception Distance)** | 生成数据分布与真实数据分布的相似性（真实感+多样性） | VoxHammer, MeshPad, TRELLIS, Hunyuan3D 2.0 | ↓ |
| **FVD (Fréchet Video Distance)** | 模型旋转渲染序列的时序连贯性 | VoxHammer | ↓ |
| **KD (Kernel Distance)** | 与 FD 类似，衡量分布距离（通常更稳定） | TRELLIS | ↓ |
| **GPTEval3D / User Study (质量)** | 人类或 GPT-4V 对整体美学、真实感、编辑融合自然度的综合打分 | VoxHammer, PrEditor3D, MeshPad | ↑ |

!!! info "FD 变体"
    Fréchet Distance 有多种变体，取决于特征提取器：
    
    - **FID_Incept**: 使用 InceptionV3 特征（经典）
    - **FID_CLIP**: 使用 CLIP 特征
    - **FD_dinov2**: 使用 DINOv2 特征（通常更鲁棒）
    - **FD_point**: 使用 PointNet++ 提取 3D 点云特征（直接评估几何分布）

---

## 6. 效率 (Efficiency)

| 指标 | 评估内容 | 使用论文 | 方向 |
|:---|:---|:---|:---:|
| **Runtime** | 整个编辑/生成流程所需时间 | PrEditor3D, MeshPad | ↓ |
| **Tokens/Second (T/s)** | 每秒生成的 token 数量（自回归方法） | MeshPad | ↑ |

---

## 7. 场景级指标

| 指标 | 评估内容 | 方向 |
|:---|:---|:---:|
| **碰撞率 (Penetration %)** | 场景插入任务中物体是否与场景相交 | ↓ |

---

## 8. 参数量与训练算力

3D 编辑相关模型的参数量和训练开销对比。

### 8.1 总览

| 模型 | 总参数量 | 训练硬件 | 训练步数 | 训练数据量 | 推理时间 | 是否需要训练 |
|:---|:---|:---|:---|:---|:---|:---|
| **TRELLIS** | 342M / 1.1B / 2B | 64×A100 (40G) | 400K | ~500K 3D 资产 | ~10s | 完整训练 |
| **TRELLIS 2** | ~4B | 32×H100 (DiT) / 16×H100 (VAE) | 渐进式 | ~800K 3D 资产 | 3s–60s (H100) | 完整训练 |
| **Native 3D Editing** | 未报告 | A800 | 150K+80K | 未报告 | 未报告 | 完整训练 |
| **Steer3D** | 未报告 | 6×A100 | 未报告 | ~96K 编辑对 | 前馈式 | 两阶段训练 |
| **VoxHammer** | 0（training-free） | 1×A100（推理） | N/A | N/A | ~133s | 免训练 |
| **Easy3E** | 仅 Ctrl-Adapter | 未报告 | 未报告 | Objaverse 子集 | ~75s | 极轻量训练 |
| **AnchorFlow** | 0（training-free） | 1×H100（推理） | N/A | N/A | ~26.71s | 免训练 |

### 8.2 TRELLIS 模块参数量

| 网络模块 | 参数量 |
|:---|---:|
| Sparse Structure VAE Encoder (E_S) | 59.3M |
| Sparse Structure VAE Decoder (D_S) | 73.7M |
| SLAT Encoder (E) | 85.8M |
| SLAT Decoder – 3DGS (D_GS) | 85.4M |
| SLAT Decoder – RF (D_RF) | 85.4M |
| SLAT Decoder – Mesh (D_M) | 90.9M |
| Structure Generator G_S (Basic / Large / XL) | 157M / 543M / 975M |
| Latent Generator G_L (Basic / Large / XL) | 185M / 588M / 1073M |

### 8.3 TRELLIS 2 模块参数量

| 模块 | 参数量 | 备注 |
|:---|---:|:---|
| SC-VAE Encoder | 354M | 空间压缩率 16× |
| SC-VAE Decoder | 474M | |
| Sparse Structure Generator | ~1.3B | DiT: width 1536, 30 blocks |
| Geometry Generator | ~1.3B | |
| Material Generator | ~1.3B | 输出完整 PBR |

### 8.4 推理速度对比

#### 3D 编辑方法（VoxHammer 基准）

| 方法 | 推理时间 |
|:---|---:|
| Vox-E | 32 min |
| MVEdit | 242s |
| VoxHammer | 133s |
| Tailor3D | 83s |
| Easy3E | 75s |
| AnchorFlow | 26.71s |
| Instant3DiT | 20s |

#### AnchorFlow 基准（基于 Hunyuan3D 2.1）

| 方法 | 推理时间 |
|:---|---:|
| TextDeformer | 2229.75s |
| MVEdit | 513.55s |
| Editing-by-Inversion | 34.86s |
| Inversion-free Editing (FlowEdit) | 25.77s |
| AnchorFlow | 26.71s |
| Direct Editing (Hunyuan3D 2.1) | 21.01s |

---

## 论文专用指标组合

### VoxHammer (Edit3D-Bench)

三维度评估体系：

| 维度 | 指标 |
|:---|:---|
| 未编辑区保留度 | Masked CD, Masked PSNR, Masked SSIM, Masked LPIPS |
| 整体 3D 质量 | FID, FVD, User Study |
| 条件对齐度 | DINO-I, CLIP-T |

### PrEditor3D

| 指标 | 类型 | 说明 |
|:---|:---|:---|
| GPTEval3D | 自动评估 (质量) | GPT-4V 比较多视图渲染图 |
| CLIPdir | 自动评估 (一致性) | 文本变化方向 vs 图像特征变化方向 |
| CLIPdiff-edit / noedit | 自动评估 (保真度) | 编辑/非编辑区域的 CLIP 分数变化 |
| User Study | 人工评估 | 提示对齐度、视觉质量、形状保持性 |
| CD | 消融研究 | 非编辑区域形状保持 |

### MeshPad

| 指标 | 类型 | 方向 |
|:---|:---|:---:|
| CD | 几何质量 | ↓ |
| FID | 感知质量 | ↓ |
| CLIP Score | 草图匹配度 | ↑ |
| LPIPS | 草图-模型局部匹配度 | ↓ |
| T/s | 效率 | ↑ |
| GQ/GM (User Study) | 生成质量/匹配度 (1-5) | ↑ |
| EQ/EM/EC (User Study) | 编辑质量/匹配度/一致性 (1-5) | ↑ |
