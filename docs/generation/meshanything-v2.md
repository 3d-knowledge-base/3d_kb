---
comments: true
---

# MeshAnything V2

> *MeshAnything V2: Artist-Created Mesh Generation with Adjacent Mesh Tokenization* (ICCV 2025)

MeshAnything V2 提出了 **Adjacent Mesh Tokenization (AMT)**，通过利用相邻面片共享顶点的特性将 mesh 序列化长度减半，使自回归生成的面数上限从 MeshAnything V1 的 800 提升到 1600。

---

## 核心问题

Mesh-native 自回归生成面临一个根本性的效率问题：

- MeshGPT 式的序列化中，每个三角面片需要 9 个坐标值（3 个顶点 × 3 维坐标）
- 但相邻面片之间共享 2 个顶点，这些共享顶点被重复编码，造成序列冗余
- 序列越长，transformer 的计算量越大，能生成的面数越少

MeshAnything V2 的核心洞察是：

> 如果当前面片和前一个面片相邻（共享一条边），那么只需要编码 1 个新顶点的坐标就能完整描述这个面片，序列长度可以减半。

---

## Adjacent Mesh Tokenization (AMT)

AMT 是本文的核心贡献，其思路是将 mesh 的序列化从"每面 3 个顶点"变为"每面尽可能只用 1 个新顶点"。

### 算法流程

1. **顶点和面排序**：按 z-y-x 顺序排列所有顶点，再按面的最小顶点索引排序面
2. **BFS 式遍历**：从第一个面开始，优先选择与当前面共享一条边的相邻面作为下一个面
3. **相邻面编码**：如果下一个面与当前面共享 2 个顶点，则只编码 1 个新顶点坐标（3 个值），而非完整的 9 个值
4. **非相邻面处理**：当没有可用的相邻面时，插入一个 `&` token 标记序列重启，然后用完整的 9 个值编码新面片

### Vertices Swap (`$` token)

AMT 还引入了一个 `$` token 机制：

- 在相邻面的遍历过程中，共享边的两个顶点存在两种排列顺序
- `$` token 允许交换共享边的顶点顺序，从而探索更多的相邻面，减少序列重启次数
- 这一机制使得遍历覆盖率更高，序列更紧凑

### 压缩效果

| 指标 | MeshAnything V1 | MeshAnything V2 (AMT) |
|---|---|---|
| 每面平均 token 数 | 9 | ~4.5 |
| S Ratio (序列压缩比) | 1.000 | 0.497 |
| 面数上限 | 800 | 1600 |

---

## 模型架构

### 点云编码器

- 输入：条件点云（来自粗糙 mesh 或其他 3D 重建方法的表面采样）
- 编码为条件 token 序列，注入 transformer 的 cross-attention

### 自回归 Transformer

- 基于 **OPT-350M** 架构（decoder-only transformer，3.5 亿参数）
- 输入：AMT 序列化后的 mesh token
- 输出：逐 token 预测下一个坐标值或特殊 token（`&`、`$`、EOS）
- 训练：标准的 next-token prediction + cross-entropy loss

---

## 训练细节

| 参数 | 值 |
|---|---|
| 模型 | OPT-350M |
| 训练数据 | Objaverse 230K pairs + ObjaverseXL |
| 硬件 | 32× A800 GPU |
| 训练时间 | 4 天 |
| 面数上限 | 1600 |
| 坐标量化精度 | 128 级 |

---

## 实验结果

### 重建质量（Artist Mesh）

| 方法 | CD ↓ | ECD ↓ | NC ↑ |
|---|---|---|---|
| MeshAnything V1 | 0.895 | 4.131 | 0.924 |
| **MeshAnything V2** | **0.874** | **3.876** | **0.933** |

### 关键对比

| 指标 | V1 | V2 |
|---|---|---|
| CD (Chamfer Distance) | 0.895 | 0.874 |
| NC (Normal Consistency) | 0.924 | 0.933 |
| S Ratio (序列长度比) | 1.000 | 0.497 |
| 平均顶点数 | 457 | 553 |
| 平均面数 | 809 | 1002 |

V2 在序列长度缩短约一半的情况下，生成了更多面数的 mesh，且几何精度和法线一致性均有提升。

### Ablation: AMT 的贡献

| 设置 | CD ↓ | NC ↑ | S Ratio |
|---|---|---|---|
| Baseline (无 AMT) | 0.895 | 0.924 | 1.000 |
| AMT (无 `$` token) | 0.886 | 0.929 | 0.533 |
| AMT + `$` token (完整) | **0.874** | **0.933** | **0.497** |

`$` token 进一步将序列长度从 0.533 压缩到 0.497，同时提升了生成质量。

---

## 与其他方法的关系

MeshAnything V2 在 mesh-native 序列化效率的发展线上处于关键位置：

- **MeshGPT**：原始逐坐标序列化，面数上限低
- **MeshAnything V1**：引入点云条件，但序列化效率未改进
- **MeshAnything V2 (AMT)**：利用邻接关系压缩序列，面数翻倍
- **BPT**：通过分块 patch 化进一步压缩，推到 8K 面
- **Nautilus**：用 BFS 邻接 + 共享顶点/边编码，5000 面
- **FACE**：one-face-one-token，压缩比达 0.11

AMT 的邻接遍历思路也被后续的 Nautilus、EdgeRunner、Mesh Silksong 等方法继承和扩展。

---

## 局限

- 面数上限仍为 1600，对于高精度几何细节仍然不够
- 非相邻面的序列重启（`&` token）仍然带来额外开销
- 依赖输入点云的质量作为条件
- 基于 OPT-350M 的参数规模，在复杂形状上的泛化能力有限
- AMT 的遍历顺序对生成结果有影响，但最优遍历策略没有理论保证

---

## 总结

MeshAnything V2 的核心贡献是 Adjacent Mesh Tokenization：通过利用三角面片之间的邻接共享关系，将 mesh 自回归生成的序列长度减半。这一看似简单的改进在实践中将面数上限从 800 提升到 1600，并成为后续多种改进型序列化方法的基础。
