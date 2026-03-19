---
comments: true
---

# Datasets

3D 编辑与生成相关的数据集汇总。分为**编辑专用数据集**和**通用 3D 数据集**两部分。

---

## 1. 编辑专用数据集

以下数据集专为 3D 编辑任务构建，包含编辑前/后配对或编辑指令标注。

| 数据集                   | 来源论文                  | 说明                              |
|:--------------------- |:--------------------- |:------------------------------- |
| **3D-Alpaca**         | ShapeLLM-Omni         | 面向 Mesh 理解、生成与编辑的多任务数据集         |
| **3DEditVerse**       | 3DEditVerse           | 大规模 3D 编辑数据集，配合 3DEditFormer 使用 |
| **Nano3D-Edit**       | NANO3D                | NANO3D 编辑 pipeline 的配套数据集       |
| **Native3D-FullAttn** | NATIVE 3D EDITING     | 原生 3D 编辑数据集                     |
| **ShapeTalk**         | ShapeTalk (CVPR 2023) | 文本指令编辑点云，在隐式空间操作，预测隐向量的更新向量     |

!!! note "Edit3D-Bench"
    VoxHammer 提出的小型 benchmark，规模较小，主要用于评估 3D 局部编辑效果。

---

## 2. 通用 3D 物体数据集

广泛用于 3D 生成模型训练与评估的大规模数据集。

| 数据集                         | 简介                               | 规模                   | 格式                 | 许可                |
|:--------------------------- |:-------------------------------- |:-------------------- |:------------------ |:----------------- |
| **Objaverse-XL**            | AllenAI 超大 3D 对象库，用于大规模训练        | 10M+ 去重对象            | glTF/GLB, USD, OBJ | 元数据 ODC-By；对象许可各异 |
| **Objaverse-LVIS**          | Objaverse 精选子集，LVIS 类别对齐         | ~44K 对象              | .glb               | 混合许可              |
| **ABO**                     | Amazon Berkeley Objects，电商高质量 3D | ~7.9K 高质量网格          | PBR 材质 + 图片 + 文本   | CC BY-NC 4.0      |
| **3D-FUTURE**               | 家具 CAD + 高分辨纹理                   | 9,992 家具模型           | CAD mesh + 纹理      | 天池申请              |
| **ShapeNet / ShapeNetCore** | 经典大规模 CAD 语义数据库                  | >3M 索引 / ~51K (Core) | OBJ                | 官网申请              |
| **ModelNet**                | 普林斯顿 CAD 模型库                     | 127,915 模型，662 类     | OFF                | —                 |
| **ABC**                     | 一百万 CAD 模型，含精确参数化描述              | 1M 模型                | STEP/STL/OBJ       | —                 |
| **Google Scanned Objects**  | Google 扫描的家居用品                   | ~1,030 物品            | SDF/OBJ            | 开源                |
| **P³-SAM**                  | 大规模 3D 部件数据集                     | ~2.3M 带分割标注对象        | —                  | —                 |
| **Toys4K**                  | 儿童常见物体 3D 集合                     | 4,179 实例，105 类       | mesh/点云            | 研究许可              |

---

## 3. 室内场景数据集

| 数据集              | 简介                       | 规模               | 格式           |
|:---------------- |:------------------------ |:---------------- |:------------ |
| **ScanNet**      | RGB-D 室内扫描，带语义分割         | 1,513 场景         | PLY          |
| **Matterport3D** | 大规模室内 RGB-D              | 90 建筑级场景         | PLY/OBJ      |
| **HM3D**         | Habitat 1000 室内重建场景      | 1,000 场景         | PLY          |
| **Gibson**       | 室内空间重建库                  | 572 模型           | PLY/OBJ      |
| **Replica**      | 18 个高质量室内场景重建            | 18 场景            | PLY + HDR 纹理 |
| **HSSD**         | Habitat Synthetic Scenes | 211 场景，18,656 对象 | Habitat 格式   |

---

## 4. 人体与动物数据集

| 数据集       | 简介        | 规模                   | 格式      |
|:--------- |:--------- |:-------------------- |:------- |
| **FAUST** | 真实人体高精度扫描 | 300 扫描（10 人 × 30 姿势） | PLY     |
| **TOSCA** | 人体和动物变形形状 | 80 网格                | OBJ/PLY |

---

## 5. 手绘/草图 Benchmark

| 数据集                     | 说明                          | 使用方     |
|:----------------------- |:--------------------------- |:------- |
| **IKEA**                | 188 个家具模型集合，用于评估模型鲁棒性       | MeshPad |
| **Hand-drawn Sketches** | 50 个手动创建和编辑的草图，用于评估真实手绘输入效果 | MeshPad |

---

## 数据集选择建议

!!! tip "常见使用模式"
    - **训练大模型**：Objaverse-XL（规模最大）
    - **评估生成质量**：Objaverse-LVIS（类别标注清晰）、GSO（扫描精度高）
    - **Mesh 编辑研究**：3DEditVerse、Nano3D-Edit、3D-Alpaca
    - **草图编辑**：ShapeNetCore + IKEA + Hand-drawn Sketches (MeshPad)
    - **场景级任务**：ScanNet、Matterport3D
