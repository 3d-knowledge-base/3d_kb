# 3D Representations

3D 视觉生成与编辑的第一步是选择合适的 3D 表征方法。不同的表征决定了模型的生成方式、渲染效率和编辑灵活性。本文梳理当前主流的五种 3D 表示。

---

## Overview

| 表示 | 类型 | 核心数据 | 渲染方式 | 可编辑性 | 典型应用 |
|:-----|:-----|:---------|:---------|:---------|:---------|
| **Mesh** | 显式 | 顶点 + 面列表 | 光栅化 | 高（直接操作顶点/面） | 游戏、影视、3D 打印 |
| **SDF** | 隐式 | 标量场 (距离值) | Ray Marching | 中（CSG 运算方便） | 医学成像、AI 生成 |
| **NeRF** | 隐式 | MLP 权重 | 体积渲染 (Ray Marching) | 低（网络权重难局部修改） | 新视角合成 |
| **3DGS** | 显式 | 高斯原语集合 | Splatting + Alpha Blending | 高（显式参数可局部操作） | 实时渲染、交互编辑 |
| **Triplane** | 混合 | 3 张正交特征平面 | MLP 解码 + 体积渲染 | 中（2D 特征图可操作） | 生成模型中间表示 |

---

## 1. Mesh（三角网格）

Mesh 是计算机图形学中最基础的 3D 表示——由**顶点 (Vertex)**、**边 (Edge)** 和**面 (Face)** 组成的多面体。

### 核心存储结构

- **顶点列表 (Vertex List)**：有序数组，每个顶点存储 $(x, y, z)$ 坐标
- **面列表 (Face List)**：通过顶点索引定义三角形/四边形面，例如 `f1 = (1, 2, 3)`

这种索引式存储极大节省了空间——每个顶点坐标只存一次，即使被多个面共享。

!!! info "半边数据结构 (Half-Edge)"
    存储更丰富的拓扑信息（边的相邻面、循环关系等），使编辑和遍历更高效，但内存开销更大。

### 主要属性

| 类别 | 属性 | 说明 |
|:-----|:-----|:-----|
| 几何 | 顶点位置、顶点/面法线 | 定义形状与表面朝向，法线决定光照效果 |
| 外观 | UV 坐标、顶点颜色、材质 (PBR) | UV 映射 2D 纹理；材质含颜色/金属度/粗糙度/自发光/透明度 |
| 动画 | 骨骼蒙皮权重、Blend Shapes | 骨骼驱动关节变形；Morph Targets 实现面部表情插值 |

### 常见文件格式

| 格式 | 特点 |
|:-----|:-----|
| **.OBJ** | 文本格式，清晰简单（`v`, `vt`, `vn`, `f`），不支持动画 |
| **.FBX** | Autodesk 二进制格式，支持几乎所有属性（几何、材质、骨骼、动画、相机） |
| **.glTF / .glb** | "3D 的 JPEG"，Web/移动端高效传输，JSON 描述 + 二进制数据，支持 PBR |
| **.STL** | 3D 打印专用，仅存三角面 + 法线，无颜色/纹理 |

### Mesh 编辑方法

| 方法 | 核心思想 | 优点 | 缺点 | 应用场景 |
|:-----|:---------|:-----|:-----|:---------|
| 多边形建模 | 直接操作点、线、面 | 精确、可控 | 曲面建模耗时 | 硬表面模型、游戏资产 |
| 数字雕刻 | 笔刷塑造"数字粘土" | 直观、艺术性强 | 拓扑混乱、面数巨大 | 有机模型、角色设计 |
| 修改器建模 | 非破坏性操作层 | 灵活、易修改 | 依赖软件功能 | 迭代开发、产品可视化 |
| 重新拓扑 | 在高模基础上重建低模 | 优化动画/性能 | 耗时、技术性强 | 动画角色、VR/AR |

---

## 2. SDF（符号距离函数）

**Signed Distance Function (SDF)** 是一种隐式表示：一个函数 $f(p)$ 为空间中任意点 $p = (x, y, z)$ 返回到物体表面的**有符号距离**。

### 核心定义

$$
f(p) = \begin{cases}
< 0 & \text{点在物体内部} \\
= 0 & \text{点在物体表面（零水平集）} \\
> 0 & \text{点在物体外部}
\end{cases}
$$

所有满足 $f(p) = 0$ 的点构成物体的表面，称为**零水平集 (Zero-level Set)**。

### 离散化：标量场

在计算机中，SDF 被离散化为一个 3D 网格（体素），每个体素存储一个浮点数值。例如一个 $64 \times 64 \times 64$ 的 SDF 网格包含 262,144 个浮点值。

!!! example "Hunyuan3D 中的 Grid Queries"
    论文中的 Grid Queries 就是生成网格点坐标，然后用解码器计算每个点的 SDF 值，形成标量场数据，再通过 Marching Cubes 提取表面。

### CSG 运算

SDF 通过简单的数学运算组合形状（**构造实体几何, CSG**）：

- **并集**：$SDF_{union}(p) = \min(SDF_A(p), SDF_B(p))$
- **交集**：$SDF_{inter}(p) = \max(SDF_A(p), SDF_B(p))$
- **差集**：$SDF_{diff}(p) = \max(SDF_A(p), -SDF_B(p))$

### 优势

- **拓扑灵活性**：轻松表示嵌套、孔洞、互穿结构
- **平滑表面**：本质连续，避免三角面片感
- **高效几何查询**：瞬间判断内/外及距离——碰撞检测利器
- **Ray Marching 渲染**：沿光线安全步进 SDF 值的距离
- **Neural SDF**：用神经网络学习连续 SDF（如 DeepSDF），内存高效、分辨率无限

---

## 3. NeRF（神经辐射场）

**Neural Radiance Field (NeRF)** 用一个 MLP 神经网络隐式表示整个 3D 场景的几何与外观。

### 核心映射

$$
F_\theta: (\mathbf{x}, \mathbf{d}) \to (\mathbf{c}, \sigma)
$$

- 输入：空间坐标 $\mathbf{x} = (x, y, z)$ + 观察方向 $\mathbf{d} = (\theta, \phi)$（5D）
- 输出：颜色 $\mathbf{c} = (R, G, B)$ + 密度 $\sigma$（4D）

密度 $\sigma$ 表示不透明度：0 = 空气，高值 = 不透明表面。

### 训练流程

1. **输入**：多视角静态图像 + 相机位姿（通常由 SfM 预计算）
2. **光线发射**：从相机穿过每个像素向场景发射光线
3. **采样 → 查询**：沿光线取采样点，将每个点的 $(x,y,z,\theta,\phi)$ 送入 MLP，得到 $(R,G,B,\sigma)$
4. **体积渲染**：将光线上所有采样点的颜色和密度按物理规则积分，得到像素颜色
5. **损失计算**：渲染图 vs 真实照片的像素级 L2 Loss → 反向传播更新 MLP 权重

### Per-Scene vs Generalizable

| 类型 | 是否每个场景训练一个网络 | 特点 |
|:-----|:------------------------|:-----|
| **Per-Scene NeRF** | 是 | 高质量，但训练耗时数小时 |
| **Generalizable NeRF** | 否——通过条件输入/少量微调适配新场景 | 快速适配，质量略有折衷 |

### 优势与局限

- **优势**：照片级真实感、视角相关效果（高光/反射）、紧凑存储（MLP 权重仅几 MB）
- **局限**：训练/渲染速度慢（后续 Instant-NGP 等大幅改善）、仅处理静态场景（D-NeRF 扩展至动态）、数据要求高

---

## 4. 3D Gaussian Splatting (3DGS)

**3DGS** 将场景建模为一组**三维高斯椭球**——一种显式、可实时渲染的 3D 表示。

### 高斯原语参数

每个 3D Gaussian 携带：

| 参数 | 符号 | 说明 |
|:-----|:-----|:-----|
| 位置 | $\mu \in \mathbb{R}^3$ | 空间中心点 |
| 形状 | $\Sigma$（等价于 scale + rotation） | 椭球形状与方向 |
| 颜色 | $c$（RGB，可选 SH 系数） | 视角相关外观 |
| 不透明度 | $\alpha$ | 遮挡与贡献强度 |

### 渲染流程

1. 将每个 Gaussian 投影到图像平面 → 二维椭圆 splat
2. 按深度排序，计算每个 splat 在像素上的权重
3. 加权混合得到像素颜色：

$$
C(p) = \sum_{i \in \mathcal{I}_p} w_i \, c_i
$$

### 训练 / 优化

1. **初始化**：从 SfM 稀疏点云初始化位置，赋予初始参数
2. **优化**：L1 / MSE / SSIM Loss → 梯度下降更新所有参数
3. **动态操作**：
    - **Splitting**：误差大的区域拆分为更小的 Gaussian
    - **Pruning**：移除低透明度 / 低贡献 Gaussian
    - **Cloning**：复杂区域复制 Gaussian 增强表现力

### 3DGS vs NeRF

| 特性 | NeRF | 3DGS |
|:-----|:-----|:-----|
| 表示方式 | 隐式 MLP | 显式 Gaussian 集合 |
| 渲染方式 | Ray marching + 体积渲染 | 投影 splatting + alpha blending |
| 训练耗时 | 数小时 | 数分钟 |
| 渲染速度 | 慢 | 实时 |
| 编辑友好性 | 难以局部修改 | 显式可控，支持局部操作 |

---

## 5. Triplane

**Triplane** 用三张正交的二维特征平面（XY、XZ、YZ）来编码三维空间特征，是介于全隐式（NeRF）和全显式（Mesh）之间的折中方案。

### 查询流程

给定空间点 $p = (x, y, z)$：

1. **投影**到三张平面：
    - XY 平面：用 $(x, y)$
    - XZ 平面：用 $(x, z)$
    - YZ 平面：用 $(y, z)$
2. 在每张平面上做**双线性插值**，得到 3 个特征向量
3. **融合**（concat / sum / 加权）→ 统一特征向量
4. 送入下游 **MLP Decoder**，输出密度 / 颜色 / SDF 等

### 优势

- 用 2D 特征图处理，**计算和存储高效**
- 可直接利用 **2D 卷积网络**的成熟架构
- 能编码三维结构、密度、颜色等属性

### 典型应用

EG3D、LRM 等生成模型广泛使用 Triplane 作为中间表示。在 TRELLIS 中，Triplane 是 SLAT 解码为 3D 输出的核心路径之一。

---

## 6. Latent Representations for Mesh Generation

如果说前面的 Mesh / SDF / NeRF / 3DGS / Triplane 是**最终输出表示**或**直接可渲染表示**，那么近两年 mesh 生成里更关键的一条线其实是：

> 如何设计一个既适合 Transformer/Flow/Diffusion 建模、又能保留 3D 几何结构的 **latent representation**。

这条线决定了模型能否同时做到：

- token 足够少，训练可扩展；
- 几何足够强，重建和生成不糊；
- 空间语义足够明确，能做局部控制和 test-time scaling；
- 最终能稳定解码回 mesh。

### 演化主线

| 方法 | 代表论文 | 核心表示 | 优势 | 主要问题 |
|:-----|:---------|:---------|:-----|:---------|
| **VecSet** | 3DShape2VecSet | 固定长度 latent vector set | 紧凑、Transformer 友好、适合 latent diffusion | 空间位置是隐式的，不够 localizable |
| **SLAT** | TRELLIS | 稀疏体素位置 + 每个位置一个局部 latent | 显式空间锚点，结构/细节解耦，便于编辑 | token 仍偏多，仍受 field/isosurface 路线约束 |
| **O-Voxel** | TRELLIS 2 | 原生结构化 3D latent | 支持开放表面、非流形、原生 PBR，压缩率更高 | 体系更复杂，训练/解码实现门槛更高 |
| **VoxSet** | LATTICE | coarse voxel anchors + 紧凑 latent set | 兼顾 VecSet 紧凑性与 sparse voxel 可定位性，支持 test-time scaling | 仍然不是最终 mesh-native 表示 |
| **Sparse volumetric latent** | SparseFlex / Direct3D-S2 | 高分辨率 sparse SDF / sparse isosurface latent | 高分辨率、复杂拓扑、工程可扩展 | 仍需在 field / isosurface 世界里做解码 |
| **Mesh-native token** | MeshAnything V2 / BPT / FACE | 直接对 mesh 序列建模 | 原生 mesh 友好，避免“先场后网格” | 序列建模仍受拓扑复杂度影响 |

### 关键判断

#### 1. 从"能压缩"到"可定位"

[3DShape2VecSet](../generation/latent-space-representations.md) 首先证明了：3D shape 可以被压缩为一组固定长度 latent vectors，且这组 vectors 适合 Transformer 和 diffusion 建模。

然而后续研究发现，**紧凑**不等于**易于生成**。当 token 缺少明确的空间锚点时，模型在生成阶段更难确定"几何细节应放置在何处"。

[TRELLIS](../generation/trellis.md) 提出的 SLAT 正是对此问题的回应：

- `{p_i}` 提供显式稀疏结构；
- `{z_i}` 负责局部几何/外观细节；
- 表示从"纯 latent set"升级为"带位置的 structured latent"。

#### 2. 从"结构化"到"原生 3D 化"

TRELLIS 将 latent 锚定到空间上，但仍沿用 SDF / isosurface 范式。`TRELLIS 2` 进一步提出 **O-Voxel**：

- 不再仅将 3D 几何视为一个待抽壳的场；
- 而是直接在结构化 3D 单元中编码几何和材质；
- 目标是更自然地处理开放表面、非流形和原生 PBR。

这一步的意义在于：latent 不只是"有结构"，而是开始更接近 **native 3D asset representation**。

#### 3. "localizable code" 比 "local feature" 更重要

[LATTICE / VoxSet](../generation/lattice.md) 提出的关键观点是：

> 真正关键的不是 local vs global，而是 latent code 是否 **localizable**。

具体而言，VecSet 的问题不仅是"太全局"，而是 token 缺少明确位置语义；Sparse voxel 的优势也不仅是"更局部"，而是它具有天然的空间 anchor。

VoxSet 的设计意义在于：

- 保留 set-based latent 的紧凑性；
- 同时引入 voxel anchors，提供位置引导；
- 因而支持更自然的 test-time token scaling。

#### 4. 平行路线：直接把 mesh 本身当作第一性对象

另一条平行路线不再纠结 latent field，而是提出：

> 如果最终目标就是高质量 mesh，为什么不直接建模 mesh 序列本身？

这条路线上有三项代表性工作：

- **MeshAnything V2**：AMT，相邻面尽量共享边，减少重复顶点 token；
- **[BPT](../generation/bpt.md)**：Blocked + Patchified Tokenization，把序列压到约 `0.26`；
- **[FACE](../generation/face.md)**：one-face-one-token，把建模单元直接提升到 triangle face，压缩比达到 `0.11`。

这条路线的意义在于：它不再满足于"先生成一个 field，再转 mesh"，而是将 mesh generation 本身原生化。

### 总结

当前 mesh generation 中的 3D latent / token 表征，本质上在平衡三个核心维度：

1. **Compactness**：token 足够少，训练可扩展。
2. **Localizability**：token 能对应明确的空间位置。
3. **Native-ness**：表示尽量接近真实 3D 资产，减少对后处理转换的依赖。

VecSet 更偏 compactness，SLAT 更偏 localizability，O-Voxel 更偏 native-ness，VoxSet 是三者的折中方案，而 BPT / FACE 代表的是 mesh-native 路线的另一极端。

---

## 表征选择指南

| 场景需求 | 推荐表示 | 理由 |
|:---------|:---------|:-----|
| 游戏 / 影视资产导出 | Mesh | 工业标准，渲染管线原生支持 |
| AI 生成中间表示 | SDF / Triplane | 连续、可微、适合神经网络优化 |
| 新视角合成（高质量） | NeRF | 照片级真实感 |
| 新视角合成（实时） | 3DGS | 实时渲染、显式可编辑 |
| 3D 编辑 pipeline | 3DGS / Mesh | 显式表示便于局部操作 |
| CSG 建模 / 碰撞检测 | SDF | 内外判断与距离查询天然高效 |
