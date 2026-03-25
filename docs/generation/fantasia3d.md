---
comments: true
---

# Fantasia3D

> *Fantasia3D: Disentangling Geometry and Appearance for High-quality Text-to-3D Content Creation* (Chen et al., ICCV 2023)

Fantasia3D 是较早的高质量 Text-to-3D 方法之一，核心思路是**将几何和外观解耦优化**：用 [DMTet](../foundations/dmtet.md) 作为几何表征通过 SDS loss 优化 mesh，再用 PBR 材质模型优化外观。

---

## 核心思想

DreamFusion 开创了 SDS（Score Distillation Sampling）驱动的 Text-to-3D 范式，但使用 NeRF 表征，输出不是可直接使用的 mesh。Fantasia3D 的改进：

1. **几何表征**：用 DMTet 代替 NeRF，直接优化可微 mesh
2. **解耦优化**：先优化几何（normal map 引导），再优化外观（PBR 材质）
3. **PBR 材质**：输出 albedo、roughness、metallic 等物理材质参数，可用于标准渲染管线

---

## 方法流程

### 第一阶段：几何优化

- 初始化 DMTet 四面体网格
- 神经网络预测每个顶点的 SDF 值和位移
- 提取 mesh，渲染 normal map
- 用 SDS loss（基于 Stable Diffusion）优化几何，条件为 text prompt
- 额外的法线一致性正则化，避免表面噪声

$$
\mathcal{L}_{\text{geo}} = \mathcal{L}_{\text{SDS}}^{\text{normal}} + \lambda_{\text{reg}} \mathcal{L}_{\text{smooth}}
$$

### 第二阶段：外观优化

- 固定几何（DMTet 参数冻结）
- 优化表面上的 PBR 材质参数（albedo, roughness, metallic）
- 用可微渲染（nvdiffrast）生成 RGB 图像
- 再用 SDS loss 优化外观

$$
\mathcal{L}_{\text{app}} = \mathcal{L}_{\text{SDS}}^{\text{RGB}}
$$

---

## 实验结果

Fantasia3D 在当时（2023 年中）的 Text-to-3D 方法中属于较高质量：

| 方法 | 输出格式 | Mesh 质量 | 纹理质量 | CLIP Score ↑ |
|:-----|:---------|:---------|:---------|:------------|
| DreamFusion | NeRF | 无直接 mesh | 中等 | ~28 |
| Magic3D | Mesh (DMTet) | 较好 | 较好 | ~30 |
| Fantasia3D | Mesh (DMTet) + PBR | **好** | **好（PBR）** | ~29 |

Fantasia3D 的优势在于 PBR 材质输出，可以在不同光照下重新渲染，而其他方法的纹理通常 baked-in。

---

## 与其他工作的关系

- **DreamFusion**：Fantasia3D 的直接改进，用 DMTet 替换 NeRF
- **Magic3D**：类似时期的工作，也用 DMTet 但采用两阶段 coarse-to-fine 策略
- **后续工作**（如 ProlificDreamer、DreamCraft3D）：进一步改进了 SDS loss 和多视角一致性

---

## 局限性

- SDS 优化速度慢（单个物体需数十分钟到数小时）
- Janus problem（多面人脸）：SDS 的固有问题，Fantasia3D 未完全解决
- 几何和外观的解耦不完美：某些情况下几何阶段的法线引导不够准确
- 已被前馈式 3D 生成方法（如 TRELLIS、TripoSG）在速度和质量上超越

---

## 一句话总结

Fantasia3D 通过 DMTet + SDS + PBR 材质解耦的方式，实现了文本到高质量可渲染 3D mesh 的生成，是 SDS 驱动 Text-to-3D 路线的代表性工作。
