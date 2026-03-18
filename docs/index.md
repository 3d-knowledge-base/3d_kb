# 3D Generation & Editing Knowledge Base

欢迎来到 3D 生成与编辑技术知识库。本站面向实验室组内分享，系统梳理了 3D 视觉生成领域的核心知识。

---

## Site Map

### [Foundations](foundations/index.md) — 基础知识

3D 表征方法（Mesh、SDF、NeRF、3DGS、Triplane）、SDS 分数蒸馏、Marching Cubes 与 FlexiCubes 等核心基础。

### [Generation](generation/index.md) — 3D 生成

TRELLIS、Hunyuan3D 2.0 等代表性模型的深度解析，以及 Mesh 生成、高斯转 Mesh、场景级生成等专题。

### [Editing](editing/index.md) — 3D 编辑

Mesh 编辑方法全景对比、编辑核心挑战分析、编辑数据集汇总。

### [Evaluation](evaluation/index.md) — 评估体系

3D 生成/编辑的评估指标详解与 Benchmark 演进综述。

---

## Quick Reference

| Topic                  | Key Content                                                    |
|:---------------------- |:-------------------------------------------------------------- |
| **3D Representations** | Mesh, SDF, NeRF, 3DGS, Triplane — 各自原理、优劣与适用场景                 |
| **TRELLIS**            | SLAT 表示、Sparse VAE、两阶段 Flow 生成、FlexiCubes 解码                   |
| **Hunyuan3D 2.0**      | ShapeVAE + DiT 几何生成、Paint 纹理生成、FLUX-style 架构                   |
| **Mesh Editing**       | 8+ 主流方法对比（VoxHammer, NANO3D, 3DEditVerse, PrEditor3D...）       |
| **Benchmarks**         | 3D Arena → MATE-3D → T³Bench → 3DGen-Bench → Eval3D → Hi3DEval |
| **Metrics**            | CLIP Score, CD, IoU, F-score, FID, GPTEval3D 等 20+ 指标          |
