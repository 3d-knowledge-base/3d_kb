# MeshCoder

> *MeshCoder: Generating Executable Blender Python Scripts from Point Clouds for Parametric 3D Modeling*

MeshCoder（2025.08）提出了一种将点云转换为**可执行 Blender Python 脚本**的方法。虽然它并非直接执行 3D 编辑操作，但其生成的代码是完全可编辑的参数化表示——用户可以修改脚本中的任意参数来改变几何形状、拓扑结构甚至组件组合方式。这使得 MeshCoder 成为**连接 3D 编辑与程序化建模**的桥梁：一旦 3D 资产被"翻译"为代码，所有编辑操作都变成了对代码的修改，天然支持精确控制与可复现性。

---

## 核心挑战

MeshCoder 要解决的关键问题是：

> 如何让模型从 3D 点云生成结构合理、可直接运行的 Blender Python 脚本？

这里面有两个核心障碍：

1. **DSL 表达力不足**：现有方法（如 ShapeAssembly、Shape2Prog）使用自定义 DSL（Domain-Specific Language），语法简单但表达力严重受限，无法覆盖复杂的建模操作。MeshCoder 的解决方案是直接使用 **Blender Python API** 作为目标语言——这是一种工业级、无限表达力的建模接口，能调用 Blender 的全部功能。

2. **缺乏大规模配对数据**：不存在现成的"点云-Blender脚本"配对数据集。MeshCoder 设计了一套完整的 **bootstrapping 数据生成流水线**，从零构建了千万级别的训练数据。

---

## Part Dataset（~1000 万样本）

数据构建的第一阶段是**零件级**数据集。MeshCoder 设计了一套基于 Blender Python API 的程序化生成框架，覆盖五类基本建模操作：

| 操作类型 | 样本数量 | 说明 |
|:---------|:---------|:-----|
| Primitive | 150 万 | 基础几何体（立方体、球体、圆柱等） |
| Translation | 300 万 | 平移变换组合 |
| Bridge Loop | 150 万 | 环桥接操作，连接不同几何面 |
| Boolean | 150 万 | 布尔运算（并集、交集、差集） |
| Array | 240 万 | 阵列修改器，重复排列几何体 |
| **总计** | **~1000 万** | — |

每个样本都是一对"Blender Python 脚本 → 对应 3D 几何体（点云）"。脚本通过程序化模板生成，随机采样参数以覆盖广泛的几何变化。

---

## Part-to-Code 模型

### 架构

Part-to-Code 模型是 MeshCoder 的基础模块，负责将单个零件的点云编码为 Blender Python 脚本：

```text
点云输入
  ↓
Shape Tokenizer
  ├─ Triplane Projection（三平面投影）
  ├─ Patchify（分块）
  ├─ Transformer Encoding
  └─ Cross-Attention with Learnable Queries → Shape Tokens
  ↓
LLM (Llama-3.2-1B + LoRA)
  ↓
Blender Python 脚本
```

**Shape Tokenizer** 的设计是关键：先将点云投影到三个正交平面上，再通过 patchify 切分为块状 token，经过 Transformer 编码后，使用 **cross-attention + 可学习查询向量** 压缩为固定数量的 shape tokens。这些 token 作为 LLM 的前缀条件输入。

**LLM** 采用 Llama-3.2-1B，通过 LoRA 进行参数高效微调。整个模型（Shape Tokenizer + LLM）端到端联合训练，使用标准交叉熵损失。

### 训练

- 硬件：64× A100 GPU
- 时间：约 1 周

---

## Object Dataset（~100 万样本）

有了 Part-to-Code 模型后，MeshCoder 通过 bootstrapping 构建**物体级**数据集：

```text
完整物体（基于 Infinigen Indoor）
  ↓
分解为独立零件
  ↓
对每个零件归一化
  ↓
Part-to-Code 模型推理 → 零件级 Blender 脚本
  ↓
逆变换（恢复原始位置/尺度）
  ↓
组装为完整物体脚本 + 语义标注
```

### 质量过滤

并非所有推理结果都可靠。MeshCoder 使用 **Chamfer Distance（CD）** 作为质量门槛：

- 仅保留**所有零件** CD < $5 \times 10^{-3}$ 的物体
- 任何一个零件不达标，整个物体都被丢弃

最终保留的数据覆盖 **41 个物体类别**，约 100 万个高质量样本。

---

## Object-to-Code 模型

Object-to-Code 模型的架构与 Part-to-Code **完全相同**（Shape Tokenizer + Llama-3.2-1B + LoRA），但训练目标是从完整物体的点云直接生成组装好的 Blender 脚本。

核心优势在于**知识迁移**：Object-to-Code 模型使用 Part-to-Code 的预训练权重初始化，而非从随机初始化开始。这使得模型能够快速适应物体级任务：

- 硬件：64× A100 GPU
- 时间：约 **2 天**（对比 Part-to-Code 的 1 周，显著加速）

---

## 实验结果

### 定量比较

MeshCoder 在重建精度上大幅超越现有方法：

| 方法 | CD ($\times 10^{-2}$) ↓ | IoU (%) ↑ |
|:-----|:------------------------|:----------|
| Shape2Prog | 6.00 | 45.03 |
| PLAD | 1.87 | 67.62 |
| **MeshCoder** | **0.06** | **86.75** |

CD 指标上，MeshCoder 比 Shape2Prog 低了**两个数量级**，比 PLAD 低了约 30 倍。IoU 也从 67.62% 提升到 86.75%，说明生成的代码能精确还原原始几何。

### 为什么提升这么大？

1. **Blender Python API 的表达力**远超自定义 DSL，能覆盖更复杂的几何操作
2. **千万级训练数据**提供了充足的学习信号
3. **从零件到物体的分阶段训练**让模型先学会简单操作，再组合为复杂建模

---

## 应用场景

MeshCoder 生成的 Blender 脚本是完全可编辑的参数化表示，这直接打开了多种下游应用：

### 几何编辑

修改脚本中的参数即可改变物体形状。例如，将桌面的宽度参数从 1.0 改为 1.5，重新运行脚本就能得到一张更宽的桌子。所有修改都是精确的、可预测的。

### 拓扑编辑

可以修改网格分辨率相关的参数（如 subdivision level、环切数量等），在不改变整体形状的前提下调整网格的拓扑结构。这在传统基于隐式表示的方法中几乎不可能实现。

### 形状理解

生成的代码本身就是对 3D 形状的**结构化语义描述**。实验表明，将 MeshCoder 输出的代码交给 GPT-4 等大语言模型，可以直接回答关于物体结构的问题（如"这把椅子有几条腿？""桌面是圆的还是方的？"），无需任何额外的 3D 理解模块。

---

## 局限性

- **仅覆盖几何信息**：当前版本只生成几何建模代码，不包含颜色、材质、纹理等外观信息
- **依赖分解质量**：Object Dataset 的构建依赖于 Infinigen 的零件分解，分解不准确会导致数据质量下降
- **计算成本高**：训练 Part-to-Code 模型需要 64× A100 运行一周，并非所有团队都能承担
- **Blender 版本耦合**：生成的脚本与特定 Blender Python API 版本绑定，API 变更可能导致兼容性问题

---

## 在编辑方法谱系中的位置

MeshCoder 代表了一条独特的技术路线——**Code-as-Representation**：

- 与 VoxHammer、NANO3D 等**潜空间编辑**方法不同，MeshCoder 将 3D 资产转化为代码，编辑操作变成对代码的修改
- 与 ShapeAssembly、Shape2Prog 等**程序化建模**方法相比，MeshCoder 使用工业级 Blender Python API 替代受限 DSL，表达力质的飞跃
- 与 MeshAnything 等 **mesh tokenization** 方法互补：MeshAnything 生成网格拓扑，MeshCoder 生成生成网格的程序

这条路线的长期价值在于：代码是最具可解释性、可编辑性和可组合性的 3D 表示形式。

---

## 一句话总结

MeshCoder 通过 bootstrapping 数据流水线和 Shape Tokenizer + LLM 架构，首次实现了从点云到可执行 Blender Python 脚本的高精度转换，将 3D 资产从"不透明的几何数据"变成了"可读、可编辑、可组合的程序化代码"。
