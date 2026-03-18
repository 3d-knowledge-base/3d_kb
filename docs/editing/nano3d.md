# NANO3D

> *NANO3D: Training-Free 3D Editing Framework (2025.10)*

NANO3D 是基于 TRELLIS 骨干的 training-free 3D 编辑框架。与同样基于 TRELLIS 的 VoxHammer 不同，NANO3D 放弃了 DDIM 反演路线，转而采用 **FlowEdit**（无反演的流编辑算法）作为核心扩散编辑手段，并提出 **Voxel-Merge** 和 **Slat-Merge** 两个显式融合机制，分别在结构层和外观层精确控制编辑区域。此外，NANO3D 还构建了 **Nano3D-Edit-100k** 大规模编辑数据集，为后续监督式 3D 编辑方法提供数据基础。

---

## 核心思想

NANO3D 的设计出发点是：

> 在 TRELLIS 的两阶段生成流程（结构生成 → 潜变量生成）中，分别插入精确的融合操作，使编辑区域被精准替换，非编辑区域完全保留。

关键技术选择是用 **FlowEdit** 替代 DDIM 反演。FlowEdit 构建从源分布到目标分布的**直接编辑路径**，不需要先将源数据反演回噪声空间再重新去噪，因此能更好地保留原始结构信息。

与 VoxHammer 的路线对比：

| 维度 | VoxHammer | NANO3D |
|:-----|:----------|:-------|
| 扩散编辑算法 | DDIM 反演 + 条件去噪 | FlowEdit（无反演） |
| 未编辑区保护 | KV Cache Replacement | Voxel-Merge + Slat-Merge |
| 融合层级 | 注意力特征层 | 体素结构层 + 潜变量层 |
| 掩码来源 | 用户提供 3D 掩码 | 自动计算（XOR + 连通域过滤） |

---

## 方法：两阶段编辑流程

### Stage 1：Voxel-based Structural Edit（体素结构编辑）

在 TRELLIS 的第一阶段（稀疏结构生成）中，对体素空间施加 FlowEdit，得到编辑后的目标体素结构，然后通过 **Voxel-Merge** 将编辑精确移植到原始结构中。

#### Voxel-Merge 流程

```
源体素 V_src ⊕ 目标体素 V_tgt  →  XOR 差异体素集合
           ↓
  连通域分析（connected components）
           ↓
  按体积阈值 τ 过滤小连通域（噪声）
           ↓
  得到 flip mask M
           ↓
  在 M 标记的区域：翻转体素占用状态（transplant）
           ↓
  融合后体素 V_merged
```

**核心思想**：通过 XOR 运算找到源和目标之间的所有差异体素，再通过连通域分析 + 体积阈值过滤掉因扩散过程引入的零散噪声，只保留有意义的编辑区域。最终得到的 flip mask $M$ 精确定义了"哪些体素需要改变"。

**阈值 $\tau$**：控制连通域的最小体积。$\tau = 100$ 为较优值——太小会引入噪声碎片，太大会过滤掉合理的小编辑。

### Stage 2：Latent-based Appearance Edit（潜变量外观编辑）

在 TRELLIS 的第二阶段（SLAT 潜变量生成）中，用编辑后的条件生成新的 SLat 特征，然后通过 **Slat-Merge** 精细混合。

#### Slat-Merge 流程

**复用** Stage 1 产生的 mask $M$：

$$
z_{\text{merged}}^i = \begin{cases} z_{\text{new}}^i & \text{if } p_i \in M \\ z_{\text{src}}^i & \text{if } p_i \notin M \end{cases}
$$

- 在 $M$ 标记的编辑区域：使用新生成的 SLat 特征
- 在 $M$ 之外的未编辑区域：**完全保留**原始 SLat 特征

这一步确保了未编辑区域不仅在几何结构上不变，外观细节也完全一致。

---

## 技术特点

### 1. FlowEdit 替代 DDIM 反演

DDIM 反演的主要问题在于：反演过程本身会引入累积误差，导致重建出的噪声潜变量无法准确还原源数据。FlowEdit 绕过了这个问题：

- 构建 **source → target** 的直接编辑路径
- 不经过"source → noise → target"的迂回
- 原始结构信息保留更完整

### 2. 自动掩码生成

与 VoxHammer 要求用户提供 3D 掩码不同，NANO3D 通过 XOR + 连通域过滤自动识别编辑区域，降低了用户交互成本。

### 3. Nano3D-Edit-100k 数据集

构建了超过 100K 编辑三元组 (source 3D, instruction, target 3D) 的大规模数据集，为监督式 3D 编辑方法提供训练数据。

#### 数据构建流水线

```
Trellis-500K / Objaverse 采样
        ↓
VLM (Qwen-VL-2.5) 生成编辑指令
        ↓
TRELLIS 重建源 3D
        ↓
2D 编辑 (Nano-Banana / Flux-Kontext)
        ↓
NANO3D 框架生成目标 3D
        ↓
Qwen2.5 质量过滤
        ↓
最终编辑三元组
```

数据集构建使用 **32 × A800 GPU**。

---

## 实验结果

### 定量比较

| 方法 | CD ↓ | DINO-I ↑ | FID ↓ |
|:-----|:----:|:--------:|:-----:|
| Tailor3D | — | — | — |
| Vox-E | — | — | — |
| TRELLIS | — | — | — |
| **NANO3D** | **0.013** | **0.950** | **27.85** |

NANO3D 在所有三个指标上均取得较优结果：

- **CD (Chamfer Distance)**：几何保真度最高
- **DINO-I**：语义一致性最好
- **FID**：生成质量较优

### 用户研究

**95% 的用户偏好** NANO3D 的形状保持效果，说明 Voxel-Merge + Slat-Merge 的显式融合策略在人类感知层面也优于基线方法。

### 消融实验

| 配置 | 效果 |
|:-----|:-----|
| 无 Merge | 编辑后几何和外观均受损 |
| 仅 Voxel-Merge | 几何结构修复，但外观细节仍有偏差 |
| 仅 Slat-Merge | 外观改善，但几何不稳定 |
| **Voxel-Merge + Slat-Merge** | **几何和外观同时正确** |

消融结果表明两个 Merge 操作缺一不可：Voxel-Merge 解决结构问题，Slat-Merge 解决外观问题。

### 阈值 $\tau$ 消融

$\tau = 100$ 为较优阈值。低于 100 时噪声碎片影响融合质量，高于 100 时合理的小编辑区域被误过滤。

---

## 优势与局限

### 优势

- **Training-free**：无需额外训练，直接利用预训练 TRELLIS 模型
- **FlowEdit 无反演**：避免 DDIM 反演的累积误差，结构保留更好
- **自动掩码**：无需用户手动标注 3D 掩码，降低交互复杂度
- **双层融合**：结构层 + 外观层的显式融合，非编辑区保持性极强
- **数据集贡献**：100K+ 规模的编辑数据集，推动监督式方法发展

### 局限

- 依赖 TRELLIS 骨干，编辑能力受限于 TRELLIS 的表示能力
- 阈值 $\tau$ 需要手动设定，对不同编辑类型的鲁棒性有待验证
- Training-free 方法的编辑上限仍然受限于预训练模型，复杂编辑场景可能不如训练式方法
- 仅编辑 Mesh 几何，纹理编辑能力未涉及

---

## 总结

NANO3D 在 TRELLIS 骨干上走出了一条与 VoxHammer 不同的 training-free 编辑路线：用 FlowEdit 替代 DDIM 反演避免累积误差，用 Voxel-Merge + Slat-Merge 的双层显式融合替代注意力层面的 KV Cache 替换。其主要价值不仅在于编辑方法本身，还在于 Nano3D-Edit-100k 数据集的构建——这一数据集为后续监督式 3D 编辑方法（如 3DEditVerse、Steer3D 等）提供了可能的训练数据来源，推动了整个 3D 编辑领域从 tuning-free tricks 向 data-driven learning 的转变。
