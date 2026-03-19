---
title: 3D Generation & Editing Knowledge Base
description: 系统梳理 3D 视觉生成与编辑领域的表征、生成、编辑与评估方向核心文献
hide:
  - navigation
comments: true
---

# 3D Generation & Editing Knowledge Base

<p class="byline">by <strong>Zhenkui Zhang</strong> · 2025.03</p>

---

<div class="grid cards" markdown>

-   :material-cube-outline:{ .lg .middle } **Foundations**

    ---

    3D 几何表征（Mesh、SDF、NeRF、3DGS、Triplane）、网格提取算法（Marching Cubes、DMTet、FlexiCubes）、SDS 分数蒸馏，以及从 DDPM 到 DiT 的生成模型基础。

    [:octicons-arrow-right-24: 进入](./foundations/index.md){ .md-button }

-   :material-creation:{ .lg .middle } **Generation**

    ---

    3D Latent Space 表征设计（VecSet → SLAT → O-Voxel），前馈式与 mesh-native 生成模型，Gaussian-to-Mesh 管线，以及场景级 3D 生成。

    [:octicons-arrow-right-24: 进入](./generation/index.md){ .md-button }

-   :material-pencil-ruler:{ .lg .middle } **Editing**

    ---

    Native 3D 编辑方法对比（VoxHammer、NANO3D、Easy3E、Steer3D 等），编辑主要挑战分析，数据集与评估。

    [:octicons-arrow-right-24: 进入](./editing/index.md){ .md-button }

-   :material-chart-box-outline:{ .lg .middle } **Evaluation**

    ---

    3D 生成与编辑的评估指标（CLIP Score、CD、IoU、FID、GPTEval3D 等 20+ 指标）与 Benchmark 综述。

    [:octicons-arrow-right-24: 进入](./evaluation/index.md){ .md-button }

</div>

---

<div class="grid cards" markdown>

-   :material-book-open-variant: **Quick Reference**

    ---

    [:octicons-arrow-right-24: 3D Representations — 五种表示的原理与适用场景](./foundations/3d-representations.md)

    [:octicons-arrow-right-24: Latent Space Representations — VecSet 到 mesh-native token](./generation/latent-space-representations.md)

    [:octicons-arrow-right-24: Mesh Editing Landscape — 三代编辑方法对比](./editing/mesh-editing-landscape.md)

-   :material-file-document-multiple: **Paper Notes**

    ---

    [:octicons-arrow-right-24: TRELLIS / TRELLIS 2 — SLAT 与 O-Voxel](./generation/trellis.md)

    [:octicons-arrow-right-24: Hunyuan3D — ShapeVAE + DiT + Paint](./generation/hunyuan3d.md)

    [:octicons-arrow-right-24: BPT / FACE — Mesh-native tokenization](./generation/bpt.md)

-   :material-list-box: **Paper Lists**

    ---

    [:octicons-arrow-right-24: Generation Paper List](./generation/paper-list.md)

    [:octicons-arrow-right-24: Editing Paper List](./editing/paper-list.md)

    [:octicons-arrow-right-24: All Papers](./papers.md)

</div>
