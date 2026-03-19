# 3D Generation & Editing Knowledge Base

[3D Generation & Editing Knowledge Base](https://3d-knowledge-base.github.io/3d_kb/)

A structured technical knowledge base focusing on modern 3D visual generation, editing, representations, and evaluation methodologies. 

This repository serves as a systematic documentation of core concepts, algorithmic evolutions, and key literature in the rapidly developing field of AI-driven 3D generation.

## Contents

- **Foundations**: Core 3D geometric representations (Mesh, SDF, NeRF, 3DGS, Triplane), Mesh extraction algorithms (Marching Cubes, DMTet, FlexiCubes), and Score Distillation Sampling (SDS).
- **Generation**: 3D latent space representations (VecSet, SLAT, O-Voxel), feed-forward vs. optimization-based generation, Gaussian-to-Mesh pipelines, and scene generation.
- **Editing**: Native 3D editing paradigms, latent-space editing (TRELLIS-based), training-free vs. optimization methods, and handling structural/textural consistency.
- **Evaluation**: Standard benchmarks, common metrics, and evaluation methodologies for 3D generation quality.

## Setup

This documentation is built with [MkDocs](https://www.mkdocs.org/) and the [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) theme.

```bash
# Install dependencies
pip install mkdocs-material

# Serve locally
mkdocs serve
```

---

*Maintained by Zhenkui Zhang*