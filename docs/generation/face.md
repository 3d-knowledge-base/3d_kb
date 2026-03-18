# FACE

> *FACE: A Face-based Autoregressive Representation for High-Fidelity and Efficient Mesh Generation*

FACE 是 mesh-native generation 路线里非常有标志性的一篇，因为它不是继续在顶点序列上做压缩，而是直接把建模单元改成了 **triangle face**。

---

## 核心主张

FACE 认为，过去 autoregressive mesh generation 的根本问题不是“模型不够强”，而是：

> 生成过程一直工作在错误的语义层级上。

过去的方法普遍把 mesh 展平成很长的顶点坐标序列，这会带来：

- 一个面被拆成很多 token
- 序列极长
- self-attention 成本很高
- 高分辨率 mesh 很难真正 scale

FACE 的回答非常直接：

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

所以 FACE 本质上有两层自回归：

- **face-level autoregression**
- **within-face coordinate autoregression**

---

## 为什么它比 BPT 更激进

和 BPT 相比，FACE 不是“压得更狠一点”，而是直接改了问题表述方式。

### BPT 的逻辑

- 继续在 vertex/coordinate sequence 上压缩
- 让序列更短、更局部

### FACE 的逻辑

- 不再默认 vertex coordinate 是基本单元
- 直接把 face 作为 mesh 的基本生成单位

所以 FACE 最核心的价值不只是压缩比 0.11，而是：

> 它从根本上改变了 autoregressive mesh generation 的基本建模单元。

---

## 质量为什么没有掉

通常极端压缩会让人担心质量下降，但 FACE 恰恰表现更好。

原因可以概括为三点：

- 序列更短，Transformer 更容易建模长程关系
- face 是更符合 mesh 结构的语义单位
- VecSet encoder 提供了强而稳定的全局 shape latent

所以它是一个很典型的“语义层级提升 -> 效率和质量一起涨”的例子。

---

## 在整条线里的位置

如果只看 mesh-native 路线：

- MeshAnything V2：AMT
- BPT：block + patch 压缩
- **FACE**：one-face-one-token

FACE 代表的其实是一种更激进的原生化趋势：

> 不再把 mesh 看成坐标序列的副产品，而是把 mesh face 本身当作第一性生成对象。

---

## 局限

- 仍然是自回归序列建模，复杂拓扑情况下难度依旧不低
- face 级离散表示仍有量化上限
- 对极细结构和稀薄部件仍然受输入采样质量影响

---

## 一句话总结

FACE 的核心意义，是把 autoregressive mesh generation 的建模单元从 `vertex coordinate` 提升到 `triangle face`，从而用 `one-face-one-token` 从根本上重新定义了长序列问题，并同时拿到了更高效率和更强质量。
