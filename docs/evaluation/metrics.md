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

| 指标 | 评估内容 | 使用论文 | 方向 |
|:---|:---|:---|:---:|
| **Masked Chamfer Distance (CD)** | 未编辑区域点云在编辑前后的几何形状保持 | VoxHammer, PrEditor3D | ↓ |
| **Masked LPIPS / PSNR / SSIM** | 渲染图上未编辑区域的外观、纹理和结构保留 | VoxHammer | LPIPS: ↓, PSNR/SSIM: ↑ |
| **CLIPdiff-noedit** | 未编辑区域在编辑前后的 CLIP 分数变化（语义内容保持） | PrEditor3D | ↓ |

!!! tip "Masked 指标的意义"
    保留度指标通过掩码仅计算未编辑区域，是衡量编辑方法「泄露」程度的直接度量。Masked CD 衡量几何保持，Masked LPIPS 衡量视觉保持，CLIPdiff-noedit 衡量语义保持。

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
