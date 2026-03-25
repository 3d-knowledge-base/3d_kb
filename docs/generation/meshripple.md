---
comments: true
---

# MeshRipple

> *MeshRipple: Structured Autoregressive Generation of Artist-Meshes*

MeshRipple 的核心贡献是为 mesh AR 生成提出了一种更合理的生成顺序：**不是按 z-order 或随机顺序生成面，而是从一个种子面开始、沿表面拓扑像水波一样向外扩展。**

---

## 核心问题

现有 mesh AR 方法面临一个基础性的矛盾：

**序列化顺序与表面拓扑不匹配。**

大多数方法（MeshGPT、MeshAnything）将三角面按 z-order sorting 排列成序列。问题是：

- z-order 是空间坐标排序，不反映面的邻接关系
- 两个在 mesh 上相邻的面，在序列中可能相距很远
- AR 模型用 sliding window 推理时，窗口外的面"看不到"，导致：
  - 孔洞和裂缝
  - 断裂的连通分量
  - 局部合理但全局不连贯

Nautilus 的 locality-aware tokenization 部分缓解了这个问题，但仍然在固定窗口大小的限制内。

---

## 方法概述

MeshRipple 提出三个互相配合的设计：

### 1. Frontier-aware BFS tokenization

核心想法：按照 BFS（广度优先搜索）的顺序遍历 mesh 面，生成一条遵循表面拓扑的序列。

- 从一个种子面开始
- 每次扩展当前"前沿"（frontier）上未访问的邻接面
- 生成顺序天然遵循 mesh 上的邻接关系

这意味着在 AR 生成时，每个新生成的面一定与已生成的面相邻，保证了表面的连通性。

### 2. Expansive prediction strategy

配合 BFS 顺序，MeshRipple 的生成策略不是一次预测一个面，而是一次扩展整个前沿上的所有面：

- 维护一个"活跃前沿"——已生成面的边界边集合
- 每步预测前沿上所有待扩展的面
- 扩展后更新前沿

这种策略保证了表面生长过程的连贯性，避免了随机跳跃。

### 3. Sparse-attention global memory

BFS 顺序解决了局部邻接问题，但长距离依赖（如对称结构、远处的拓扑约束）仍然需要全局信息。MeshRipple 为此引入了：

- 一个全局记忆模块，存储已生成部分的稀疏表示
- 通过稀疏注意力机制，当前生成步可以查询整个已生成 mesh 的全局信息
- 提供了"接近无限"的感受野，同时保持计算可控

---

## 损失函数

MeshRipple 的训练损失包含以下组件：

### AR 生成损失

$$
\mathcal{L}_{AR} = -\sum_{t=1}^{T} \log p_\theta(x_t \mid x_{<t}, \mathbf{m})
$$

标准的 next-token prediction 交叉熵损失，但关键区别在于：

- token 顺序遵循 BFS 拓扑排列而非 z-order
- $\mathbf{m}$ 表示 sparse-attention global memory 提供的上下文

### 拓扑一致性约束

论文还引入了针对 frontier 扩展过程的辅助损失：

- **连通性损失**：惩罚生成的新面与已有前沿面之间不连通的情况
- **边界一致性**：确保新生成面的共享边与前沿已有边的坐标一致

### 条件注入

- 点云/图像条件通过 cross-attention 注入 transformer
- 条件信号不引入额外损失项，通过 teacher-forcing 隐式学习对齐

---

## 架构与训练细节

### 模型架构

- **Backbone**：decoder-only transformer，配合 sparse attention 机制
- **Global memory**：维护已生成面的稀疏特征向量池，每个生成步通过 top-k 稀疏注意力查询
- **Frontier tracking**：在推理时维护活跃前沿边集合，决定下一步扩展的面
- **位置编码**：结合序列位置和 BFS 深度的混合位置编码

### 训练配置

- 训练数据：Objaverse 过滤子集
- 坐标量化：128 级
- BFS tokenization 在数据预处理阶段完成（每个 mesh 只需计算一次）

---

## 实验结果

### 评估设置

- 数据集：Objaverse 子集（多类别混合）
- 对比方法：MeshGPT、MeshAnything V2、Nautilus 等
- 重点评估指标：表面保真度（CD, NC）和**拓扑完整性**（连通分量数、孔洞率）

### 定量结果

论文报告 MeshRipple 在拓扑质量指标上有明确优势：

| 指标 | MeshRipple 表现 |
|---|---|
| 连通分量数 | 接近 ground truth（通常为 1） |
| 孔洞率 | 低于 z-order baseline |
| Chamfer Distance | 与 Nautilus 可比或更优 |
| Normal Consistency | 优于 z-order baseline |

### 关键发现

- BFS tokenization 使得生成的 mesh 几乎不出现孤立连通分量
- sparse-attention global memory 对长距离对称结构（如椅子的四条腿）的建模有帮助
- frontier 扩展策略在生成 1000-3000 面范围的 mesh 时优势最明显

---

## 为什么叫 "Ripple"

名字很好地概括了方法的直觉：

- 想象一块石头丢进水面，产生的涟漪从中心向外扩散
- MeshRipple 从种子面开始，mesh 像涟漪一样一圈一圈向外生长
- 每一圈都与上一圈相连，不会出现孤立的面

---

## 与 Nautilus 的区别

虽然 Nautilus 和 MeshRipple 都利用了 mesh 的拓扑结构，但角度不同：

| | Nautilus | MeshRipple |
|---|---|---|
| 核心策略 | 压缩序列长度（共享顶点/边） | 重新定义生成顺序（BFS 拓扑顺序） |
| 目标 | 生成更多面（5000 面） | 生成更连贯的表面 |
| 全局信息 | 双流条件注入 | sparse-attention global memory |
| 生成方式 | 逐 token 解码 | 前沿批量扩展 |

两者的思路互补，不矛盾。

---

## 与其他工作的关系

### 相比 TSSR（离散扩散）

- TSSR 通过并行去噪解决全局可见性问题
- MeshRipple 保留 AR 框架，但通过拓扑感知的生成顺序和全局记忆来弥补

### 相比 BPT / FACE

- BPT 用 BSP-tree 做空间分割序列化
- FACE 做 face-level codebook
- MeshRipple 更直接地在 mesh 拓扑图上做 BFS

### 相比 FlashMesh

- FlashMesh 关注推理加速（投机解码）
- MeshRipple 关注生成质量（拓扑连贯性）

---

## 优势与局限

### 优势

- BFS 顺序天然保证表面连通性，从根本上减少孔洞和断裂
- frontier-aware 扩展策略使生成过程语义直观
- sparse-attention global memory 在不大幅增加成本的前提下提供全局信息
- 在表面保真度和拓扑完整性上优于同期 baseline

### 局限

- BFS 起始点的选择可能影响生成结果
- 前沿扩展策略增加了实现复杂度
- 面数上限未明确报告是否超过 Nautilus 的 5000
- 仍限于三角 mesh

---

## 一句话总结

MeshRipple 通过 frontier-aware BFS tokenization 将 mesh AR 生成的序列化顺序从空间坐标排序改为拓扑邻接顺序，配合前沿批量扩展和稀疏注意力全局记忆，从根本上解决了 mesh 自回归生成中的表面断裂和拓扑不连贯问题。
