# 3D Generation & Editing Knowledge Base

> **作者**: Zhenkui Zhang | **协作者**: [OpenCode](https://opencode.ai) AI Assistant | **最后更新**: 2026-03-25

[![Website](https://img.shields.io/badge/Website-Online-blue)](https://3d-knowledge-base.github.io/3d_kb/)

📖 **在线阅读**: [https://3d-knowledge-base.github.io/3d_kb/](https://3d-knowledge-base.github.io/3d_kb/)

本知识库系统梳理了 3D 视觉生成与编辑领域的核心文献与技术进展，涵盖从基础表征到前沿生成、编辑方法的完整技术栈。

> 📝 本知识库在 **OpenCode** AI 助手辅助下整理完成。

---

## 📚 内容概览

本知识库分为四大模块：

### **Foundations** — 基础表征

- 3D 几何表征：Mesh、SDF、NeRF、3DGS、Triplane
- 网格提取算法：Marching Cubes、DMTet、FlexiCubes
- 分数蒸馏采样（SDS）与生成模型基础（DDPM → DiT）

### **Generation** — 3D 生成

- 3D Latent Space 表征：VecSet、SLAT、O-Voxel
- 前馈式生成与前向网格生成（Direct3D、SPAR3D、LATTE）
- Gaussian-to-Mesh 转换管线
- 场景级 3D 生成方法

### **Editing** — 3D 编辑

- Native 3D 编辑范式与训练无关方法
- Latent-space 编辑（TRELLIS-based）
- 结构编辑与纹理编辑方法
- 编辑领域核心挑战与数据集

### **Evaluation** — 评估指标

- 20+ 评估指标：CLIP Score、CD、IoU、FID、GPTEval3D 等
- 主流 Benchmark 与评测方法论

---

## 🌐 本地运行网站

本仓库使用 [MkDocs](https://www.mkdocs.org/) + [Material](https://squidfunk.github.io/mkdocs-material/) 构建静态网站。

### 安装依赖

```bash
pip install mkdocs-material
```

### 本地预览

```bash
mkdocs serve
```

访问 [http://127.0.0.1:8000](http://127.0.0.1:8000)

### 部署到 GitHub Pages

```bash
mkdocs gh-deploy --force
```

或推送到 `master` 分支，GitHub Actions 会自动部署。

---

## 📂 仓库结构

```
3D_Blog/
├── docs/
│   ├── foundations/    # 基础表征与算法
│   ├── generation/     # 3D 生成方法
│   ├── editing/        # 3D 编辑方法
│   └── evaluation/     # 评估指标与基准
├── mkdocs.yml          # 站点配置
└── README.md           # 本文件
```

---

**License**: MIT | **维护者**: Zhenkui Zhang | **协作**: OpenCode
