---
comments: true
---

# Gaussian to Mesh

从 3D Gaussian Splatting (3DGS) 中提取高质量 Mesh 是一个核心挑战。3DGS 实质上是为快速渲染优化的数百万个无序、离散的 3D 高斯体，并不天然形成连续平滑的几何表面。所有方法都需要引入额外的几何约束或先验。

---

## 四种技术路线

### 路线 1: 几何正则化

在 3DGS 训练中加入损失函数，强制高斯体排列得更像表面（更薄、更平整、分布更均匀），然后从优化后的高斯体提取点云进行重建。

#### SuGaR

- 提出正则化项，鼓励高斯体与表面对齐
- 正则化训练 → 水平集点云采样 → **泊松重建**
- 耗时 ~1h，单张 V100 GPU

#### GSrec

- 混合方法：单目深度/法线先验 + 局部 MLS 隐式场正则化
- 高斯中心点 + 学习到的法线 → **泊松重建**
- 耗时 ~40min，单张 RTX 3090

---

### 路线 2: 显式约束为表面基元

直接将 3D 高斯改造为 2D 面元——去除厚度问题的根源。

#### 2DGS

- 用 **2D 高斯盘**（无厚度椭圆面元）替代 3D 高斯
- 多视角渲染深度图 → **TSDF Fusion** → Marching Cubes
- 耗时 ~11min，RTX 3090

#### Gaussian Surfels

- 将 z 轴 scale **强制设为 0** → "压扁"成 2D 面元
- 深度-法线一致性损失 + 体素切割 → **泊松重建**
- 耗时 **~7min**，RTX 4090

#### PGSR

- 类似平面约束 + **无偏深度渲染**
- 多视角深度图 → **TSDF Fusion**
- 耗时 ~30min，RTX 4090

---

### 路线 3: 引入外部几何先验

利用 3DGS 的渲染能力，结合外部几何模型（MVS、立体匹配、深度估计等）。

#### GausSurf

- 迭代结合 **Patch-match** 与高斯优化
- 纹理丰富区域用 Patch-match，弱纹理区域用法线先验
- 耗时 **~7min**，RTX 3090

#### GS2Mesh

- 为每个视角生成虚拟**立体图像对**
- 送入预训练立体匹配模型（DLNR）→ 深度图 → **TSDF Fusion**
- 3DGS 训练后额外 ~5min

#### DN-Splatter / AGS-Mesh

- 利用 LiDAR / 单目深度法线先验正则化训练
- DN-Splatter → 泊松重建；AGS-Mesh → IsoOctree TSDF
- 耗时 ~40min，RTX 4090

---

### 路线 4: 可微分 Mesh 提取（Mesh-in-the-Loop）

将 Mesh 生成整合到端到端训练中——较新的范式。

#### MILo

- **每次迭代**都从高斯体可微地提取 Mesh
- 可微 Delaunay 三角化 → 可微 Marching Tetrahedra
- Mesh 损失梯度反传回高斯体参数 → 双向绑定
- 耗时 ~50min (DTU) / ~110min (T&T)，RTX 4090

#### GOF

- 导出连续的**高斯透明度场**
- 以透明度场等值面为表面 → **Marching Tetrahedra**
- 耗时 ~24min (T&T)，A100

---

## 系统性对比

| 论文 | 策略分类 | Mesh 提取方法 | 迭代优化 | 耗时 | GPU |
|:-----|:---------|:-------------|:---------|:-----|:----|
| SuGaR | 几何正则化 | 泊松重建 | 可选 | ~1h | V100 |
| GSrec | 正则化 + 隐式场 | 泊松重建 | 否 | ~40min | 3090 |
| 2DGS | 2D 盘 | TSDF Fusion | 否 | ~11min | 3090 |
| Gaussian Surfels | 2D 面元 | 泊松重建 | 否 | **~7min** | 4090 |
| PGSR | 平面约束 | TSDF Fusion | 否 | ~30min | 4090 |
| GausSurf | 外部 MVS | TSDF Fusion | 深度图迭代 | **~7min** | 3090 |
| GS2Mesh | 外部立体匹配 | TSDF Fusion | 否 | ~20min | - |
| DN-Splatter | 外部先验 | 泊松重建 | 否 | ~37min | 4090 |
| AGS-Mesh | 外部先验 | IsoOctree TSDF | 否 | ~40min | 4090 |
| GOF | 透明度场 | Marching Tet | 否 | ~24min | A100 |
| **MILo** | **Loop 内可微** | **可微 MT** | **是（核心）** | ~50min | 4090 |

---

## 趋势与结论

### 重建方法的两个阵营

- **泊松重建**：需要高质量带法线点云，水密性好，对法线精度敏感
- **TSDF Fusion**：依赖多视角深度图，对噪声鲁棒，但易丢失高频细节

### 几何约束的发展

1. **内部约束** (SuGaR) → 2. **显式改变表达** (2DGS, Surfels, PGSR) → 3. **引入外部智慧** (GausSurf, GS2Mesh) → 4. **端到端联合优化** (MILo)

### 效率 vs 质量

- **最高效率**：Gaussian Surfels 和 GausSurf（~7min）
- **最高质量**：MILo 和 GausSurf/GS2Mesh（强几何约束）
- **原始 3DGS 直接提取效果最差**：证明额外几何约束是必要的
