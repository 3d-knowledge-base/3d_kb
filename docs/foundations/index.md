# Foundations

本节介绍 3D 生成与编辑所需的核心基础知识。

---

## Topics

### [3D Representations](3d-representations.md)
3D 表征方法概述：Mesh、SDF、NeRF、3D Gaussian Splatting、Triplane。每种表示的原理、数据结构、优劣与适用场景。

### [SDS (Score Distillation Sampling)](sds.md)
分数蒸馏采样——用预训练的 2D 扩散模型指导 3D 生成的技术原理。包括 DreamFusion 的设计思路，以及 VSD、CSD、ESD 等后续改进。

### [Mesh Extraction](mesh-extraction.md)
从隐式场（SDF / 标量场）提取显式三角网格的算法：Marching Cubes、DMTet、FlexiCubes 三种方案的原理与发展。

### [Generative Models](generative-models.md)
视觉生成模型基础：从 DDPM 到 DDIM、Flow Matching、Rectified Flow、LDM/Stable Diffusion、DiT、CFG 的技术发展线索——3D 生成的上游基础。
