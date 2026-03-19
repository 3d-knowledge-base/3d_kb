# Hunyuan3D 2.0

> *Hunyuan3D 2.0: Scaling Diffusion Models for High Resolution Textured 3D Assets Generation*

腾讯开源的大规模 3D 生成系统，提供从单张图片生成高分辨率带纹理 3D 资产的流程。

---

## 动机与挑战

- 传统 3D 资产生产流程复杂、昂贵、门槛高
- 3D 生成领域缺乏类似 Stable Diffusion 的强大开源基础模型
- 目标：打造 3D 领域的"开源基础模型"，降低创作门槛

---

## 整体架构：两阶段解耦

```
条件图像 → [Hunyuan3D-DiT] → 裸网格 (Bare Mesh)
                                    ↓
条件图像 + 裸网格 → [Hunyuan3D-Paint] → 纹理贴图 → 完整 3D 资产
```

**解耦优势**：
- 降低联合优化难度
- Paint 可为任何外部 Mesh 生成纹理（不限于系统自生成的）

!!! warning "解耦局限"
    纹理模型无法修正几何错误。如果 DiT 阶段生成的表面过于光滑（如缺少纽扣凸起），Paint 只能"画"上去而无法创造立体感。

---

## Stage 1: ShapeVAE + DiT — 几何生成

### ShapeVAE：3D 形状 ↔ 潜在编码

**编码器 $\mathcal{E}_s$**：

1. **点云采样**（技术特点）
    - 均匀采样 $P_u$：覆盖整体表面
    - **重要性采样 $P_i$**：在边/角等高曲率区域采样更多点 → 捕捉高频细节
2. **FPS 获取查询点** $Q_u, Q_i \to Q$
3. **交叉注意力**：$Q$ 作为 Query，点云 $P$ 作为 Key/Value → 压缩几何信息
4. **潜在空间映射**：输出均值 $\mathrm{E}(Z_s)$ 和方差 $\mathrm{Var}(Z_s)$

**解码器 $\mathcal{D}_s$**：

1. 输入潜在编码 $Z_s$ + 3D 网格坐标 $Q_g$
2. 通过自注意力 + 交叉注意力，预测每个坐标点的 **SDF 值** $F_{sdf}$
3. **Marching Cubes** 从 SDF 场提取三角网格

**训练**：

$$
\mathcal{L}_r = \mathbb{E}_{x \in \mathbb{R}^3}[\text{MSE}(\mathcal{D}_s(x|Z_s), \text{SDF}(x))] + \gamma \mathcal{L}_{KL}
$$

- 多分辨率训练：动态改变 token 长度，加速收敛 + 提高鲁棒性

### Hunyuan3D-DiT：潜在空间上的生成

**架构**：借鉴 FLUX，双流 + 单流混合 Transformer

| 模块类型 | 处理方式 |
|:---------|:---------|
| 双流模块 | 形状 token 和图像 token 独立处理，仅在注意力计算时交互 |
| 单流模块 | 形状 + 图像 token 拼接后共同通过注意力层 |

**特殊设计**：
- **无位置编码**：token 内容本身编码了几何信息
- **图像编码器**：冻结的 DINOv2-Giant
- **图像预处理**：移除背景、主体居中、白色填充

**训练目标**：Flow Matching

$$
\mathcal{L} = \mathbb{E}_{t, x_0, x_1}[\|u_\theta(x_t, c, t) - u_t\|_2^2]
$$

其中 $x_t = (1-t)x_0 + tx_1$，$u_t = x_1 - x_0$。推理时用 ODE 求解器从噪声积分到潜在编码。

---

## Stage 2: Paint — 纹理生成

### 预处理

**图像去光照 (Delighting)**：

训练一个 Image-to-Image 模型，将带复杂光照的图像转为均匀白光版本。数据通过渲染 3D 数据集获得（随机 HDRI 光照 vs 均匀白光成对）。

**视图选择策略**：

贪心算法，从 4 个正交基础视图开始，迭代选择覆盖最大未覆盖 UV 面积的新视图。

### Hunyuan3D-Paint 架构

**双流图像条件参考网 (Reference-Net)**：

- 参考分支直接接收**无噪声**的参考图像 VAE 特征（$t=0$）
- 基础 SD 2.1 权重**冻结** → 防止风格偏向训练数据风格

**多任务注意力**：

$$
Z_{MVA} = Z_{SA} + \lambda_{ref} \cdot \text{Attention}_{ref} + \lambda_{mv} \cdot \text{Attention}_{mv}
$$

| 注意力分量 | Query 来源 | Key/Value 来源 | 功能 |
|:-----------|:-----------|:---------------|:-----|
| $Z_{SA}$ | 自身 | 自身 | 内容生成 |
| $\text{Attention}_{ref}$ | 当前去噪分支 | 参考网络无噪声特征 | 对齐参考图像 |
| $\text{Attention}_{mv}$ | 当前视图 | 所有其他视图特征 | 多视图一致性 |

**几何条件**：多视图法线贴图 + CCM → VAE 编码 → 与噪声 latent 在通道维度拼接

### 纹理烘焙

1. **视图丢弃训练**：44 个预设视点中每次随机选 6 个 → 推理时可生成密集视图
2. **超分辨率**：ESRGAN 提升每张视图分辨率
3. **反投影烘焙**：密集视图 → UV 贴图；空洞用顶点颜色插值修复

---

## 实验结果

| 评估维度 | 指标 | 结果 |
|:---------|:-----|:-----|
| 形状重建 | V-IoU, S-IoU | ShapeVAE 在各指标上优于其他方法，验证重要性采样有效性 |
| 形状生成 | ULIP-T/I, Uni3D-T/I | DiT 条件对齐度最高 |
| 纹理合成 | FID_CLIP↓, CMMD↓, CLIP-score↑ | Paint 在各指标上表现最好 |
| 端到端 | FID_CLIP↓, CMMD↓, CLIP-score↑ | 超越所有开源和闭源基线 |
| 用户研究 | 整体满意度、质量、图像遵循度 | 所有维度最高偏好 |

---

## 训练总结

| 组件 | 数据来源 | 训练参数 | 训练目标 |
|:-----|:---------|:---------|:---------|
| Image Delighting | 合成成对数据 | 整个模型 | 光照→无光照映射 |
| ShapeVAE | Objaverse 等 | 编码器 + 解码器 | SDF MSE + KL |
| DiT | ShapeVAE 潜在编码 + 图像 | DiT (DINOv2 冻结) | Flow Matching |
| Paint | 自建数据集多视图渲染 | 注意力层 + 输入卷积 + 视图嵌入 (SD冻结) | 扩散去噪 |

---

## 局限性

1. **解耦局限**：纹理无法修正几何错误（单向依赖）
2. **预处理依赖**：去光照模型失效会污染纹理
3. **缺乏失败案例分析**：透明/反射材质、极细结构、复杂拓扑的表现未知

!!! note "输入限制"
    Hunyuan3D 2.0 主要专注于 **Image-to-3D** 路径，Hunyuan3D-DiT 接收单张图像作为条件输入生成裸网格（bare mesh）。虽然论文未明确讨论 Text-to-3D 路径，但系统整体支持文本引导：文本可直接通过 Hunyuan3D-Paint 模块引导纹理生成，也可搭配文生图模型（如 Stable Diffusion）先生成参考图像，再输入 Hunyuan3D-DiT 进行形状生成。
