---
comments: true
---

# FACE

> *FACE: A Face-based Autoregressive Representation for High-Fidelity and Efficient Mesh Generation*

FACE 是 mesh-native generation 路线里有特点的一篇，因为它不是继续在顶点序列上做压缩，而是直接把建模单元改成了 **triangle face**。

---

## 核心主张

FACE 认为，过去 autoregressive mesh generation 的根本问题不是“模型不够强”，而是：

> 生成过程一直工作在错误的语义层级上。

过去的方法普遍把 mesh 展平成很长的顶点坐标序列，这会带来：

- 一个面被拆成很多 token
- 序列极长
- self-attention 成本很高
- 高分辨率 mesh 很难scale

FACE 的回答直接：

> 一个 triangle face 才是 mesh 更自然的基本语义单元，所以应该直接按 face 来生成。

---

## One-face-one-token

FACE 的核心设计可以概括成一句话：

- 一个三角面由 3 个顶点组成
- 展平成一个 9 维向量
- 再把整个 face 映射成一个 latent token
- 自回归过程按 face sequence 而不是 vertex sequence 展开

这样做后：

- 原本一个 face 对应很多 coordinate token
- 现在一个 face 就是一个 token 单元
- 压缩比达到 **0.11**

这不只是“更省”，而是直接改变了模型的建模语义层级。

---

## 模型结构

FACE 不是单纯的 tokenization 技巧，而是一套 **Autoregressive Autoencoder (ARAE)**。

### Shape Encoder

- 输入 point cloud + normals
- 用 `3DShape2VecSet` 风格的 VecSet encoder 压缩成 latent

### Autoregressive Face Decoder

- 先把 face 映射成 token
- 再通过 Transformer 逐 face 生成
- 同时通过 cross-attention 读取 VecSet latent

### CausalMLP Coordinate Decoder

每个生成出来的 face token 最终还要被解码回 9 个离散坐标 token。论文发现：

- 并行解码效果差
- attention-based 也不如预期
- `CausalMLP` 最稳定、效果最好

所以 FACE 实质上有两层自回归：

- **face-level autoregression**
- **within-face coordinate autoregression**

---

## 为什么它比 BPT 更进一步

和 BPT 相比，FACE 不是“压得更狠一点”，而是直接改了问题表述方式。

### BPT 的逻辑

- 继续在 vertex/coordinate sequence 上压缩
- 让序列更短、更局部

### FACE 的逻辑

- 不再默认 vertex coordinate 是基本单元
- 直接把 face 作为 mesh 的基本生成单位

所以 FACE 主要的价值不只是压缩比 0.11，而是：

> 它改变了 autoregressive mesh generation 的基本建模单元。

---

## 质量为什么没有掉

通常较高的压缩会让人担心质量下降，但 FACE 恰恰表现更好。

原因可以概括为三点：

- 序列更短，Transformer 更容易建模长程关系
- face 是更符合 mesh 结构的语义单位
- VecSet encoder 提供了强而稳定的全局 shape latent

所以它是一个典型的“语义层级提升 -> 效率和质量一起涨”的例子。

---

## 在整条线里的位置

如果只看 mesh-native 路线：

- MeshAnything V2：AMT
- BPT：block + patch 压缩
- **FACE**：one-face-one-token

FACE 代表的其实是一种更进一步的原生化趋势：

> 不再把 mesh 看成坐标序列的副产品，而是把 mesh face 本身当作第一性生成对象。

---

## 模型架构细节

| 组件 | 参数 |
|:-----|:-----|
| 总参数量 | ~500M |
| Shape Encoder | 8 层，hidden dim 768 |
| Face Decoder | 24 层，hidden dim 1024（非对称设计） |
| VecSet Latent | 2048 tokens，bottleneck dim 64 |
| 坐标量化 | $[0, 127]$ 离散化 |

非对称 encoder-decoder 设计：encoder 较浅较窄，decoder 较深较宽，因为自回归生成比编码更难。

---

## 损失函数

ARAE 的训练目标是标准的交叉熵损失，直接对 face 内的 9 个离散坐标 token 逐个做分类：

$$
\mathcal{L} = \frac{1}{N}\sum_{i=1}^{N}\sum_{j=1}^{9} \text{CrossEntropy}(\hat{c}_{i,j},\; c_{i,j})
$$

- $N$：face 总数
- $c_{i,j}$：第 $i$ 个 face 的第 $j$ 个坐标 ground truth
- 端到端训练，无需单独的 VAE 预训练阶段

---

## 训练配置

| 配置 | 参数 |
|:-----|:-----|
| ARAE 训练 | 8×A100 (80GB), 100K steps |
| 优化器 | Muon, lr 6e-4 |
| 训练数据 | Objaverse ~130K meshes（<4000 faces） |
| DiT 生成模型 | 350M params, 50K mesh subset |
| DiT 训练 | 32×A100 (80GB), 400K steps, Euler 100 steps |

---

## 实验结果

### Reconstruction（Tab 2）

| 方法 | HD ↓ (Objaverse) | CD ↓ (Objaverse) | HD ↓ (Toys4K) | CD ↓ (Toys4K) |
|:-----|:-----------------|:-----------------|:-------------|:-------------|
| BPT | 0.175 | 0.072 | 0.108 | 0.067 |
| TreeMeshGPT | 0.119 | 0.061 | 0.098 | 0.055 |
| MeshAnything V2 | 0.142 | 0.065 | 0.089 | 0.049 |
| **FACE** | **0.090** | **0.041** | **0.067** | **0.033** |

### 关键设计选择（Ablation）

- Face ordering：ZYX ordering 优于 DFS 和 BFS
- 坐标解码：CausalMLP 优于 Parallel 和 Attention-based 解码
- Encoder query：FPS (farthest point sampling) queries 优于 learnable queries
- 压缩比 0.11，self-attention 理论计算量降低 81×

### Scaling

1.2B large model + 1024 量化级别可进一步提升生成质量。

---

## 局限

- 仍然是自回归序列建模，复杂拓扑情况下难度依旧不低
- face 级离散表示仍有量化上限
- 对极细结构和稀薄部件仍然受输入采样质量影响
- 训练数据限制在 <4000 faces 的 mesh，高面数模型无法直接处理

---

## 一句话总结

FACE 的主要意义，是把 autoregressive mesh generation 的建模单元从 `vertex coordinate` 提升到 `triangle face`，从而用 `one-face-one-token` 重新定义了长序列问题，并同时拿到了更高效率和更强质量。
