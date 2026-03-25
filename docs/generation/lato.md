---
comments: true
---

# LATO

> *LATO: 3D Mesh Flow Matching with Structured TOpology Preserving LAtents* (arXiv 2603.06357, March 2026)

LATO 解决的是 3D 生成中一个核心矛盾：**隐式场方法擅长几何但丢失拓扑，自回归方法保留拓扑但受限于序列长度**。LATO 提出了一种保拓扑的稀疏体素表示 T-Voxels，让 flow matching 扩散模型可以直接生成带有显式连接关系的 3D 网格，同时保持高推理效率。

---

## 核心问题

当前 3D 生成存在一个两难困境：

- **隐式场路线**（TRELLIS、TripoSG、CLAY 等）：依赖 Marching Cubes 提取网格，生成的三角面片极其密集且不规则，不符合美术需求，且训练数据必须是水密（watertight）的
- **自回归路线**（MeshGPT、BPT、MeshAnything V2 等）：直接生成显式 mesh 拓扑，但序列长度随面数二次增长，训练时必须截断序列，导致破面和断裂

LATO 的思路是：

> 设计一种将 mesh 的顶点位置和连接关系都编码进稀疏体素隐空间的表示（T-Voxels），使连续 flow matching 模型能直接生成拓扑信息，同时避免自回归的序列长度瓶颈。

---

## Vertex Displacement Field (VDF)

LATO 的第一步是将显式 mesh 拓扑转化为适合神经网络学习的连续表示。

给定一个 mesh $M = (V, F)$，对于表面上采样的任意点 $p$（位于面 $f$ 上），VDF 定义为：

$$F(p) = \{v - p \mid v \in \{v_{f_0}, v_{f_1}, v_{f_2}\}, p \in f\}$$

即从采样点指向其所在三角面三个顶点的相对位移向量集合。

### 为什么用 VDF 而不是语义标签

一种朴素替代方案是给表面点打标签（vertex / edge / face 三类），但存在两个问题：

1. **类别不平衡**：顶点和边在连续表面上是零测度集，随机采样几乎采不到，监督信号极弱
2. **离散标签不连续**：分类标签在隐空间中产生突变，阻碍扩散模型的梯度流

VDF 提供了**密集且连续**的监督信号：表面上的每一个点都携带指向其所在面三个顶点的位移信息，同时隐含了边的连接关系。

---

## Sparse Voxel VAE 与 T-Voxels

### 编码器

1. 在 mesh 表面均匀采样 $K = 819,200$ 个点
2. 对每个点构建特征向量 $x_k = [p_k, F(p_k), n(p_k)]$（位置 + VDF 位移 + 法线）
3. 空间离散化为稀疏体素网格，每个体素内通过 **PointNet**（共享 MLP + 均值池化）聚合局部特征
4. **Sparse Transformer**（维度 512，8 个头）捕获全局上下文
5. 输出每个活跃体素的均值和方差，通过重参数化采样得到 **T-Voxel 隐变量**

最终 T-Voxel 隐空间为 $128^3$ 分辨率、16 通道。

### 解码器：层次化细分与剪枝

直接回归顶点坐标很困难，因为不同 mesh 的顶点数量差异很大。LATO 采用 hierarchical subdivide-and-prune 策略：

1. **初始剪枝**：Sparse Transformer + 线性分类器预测每个 T-Voxel 的占用概率，丢弃概率 < 0.5 的体素
2. **级联细化**（$L=3$ 级）：每级执行：
    - **体素细分**：每个活跃父体素分裂为 $2^3=8$ 个子体素
    - **特征初始化**：复制父特征 + 相对位置编码 + 3D 卷积
    - **剪枝头**：去除空子体素
    - **Sparse Self-Attention**：聚合局部上下文
    - **Cross-Attention**：子体素作为 query，原始 T-Voxels 作为 key/value，回传全局信息
3. **最终输出**：剩余体素的中心坐标即为预测的顶点集 $\{\hat{v}\}$

### Connection Prediction Head

为恢复完整拓扑，需要预测所有顶点对之间的边连接：

1. 顶点体素特征通过 Cross-Attention 与 T-Voxels 聚合全局上下文
2. 连接两个顶点 $\hat{v}_i$ 和 $\hat{v}_j$ 的特征，经 MLP 预测边存在概率
3. **排列不变性**：对 $(h_i \oplus h_j)$ 和 $(h_j \oplus h_i)$ 两种拼接顺序分别预测，取平均

训练时的采样策略：正样本包含所有真实边对应的顶点对；每个顶点额外采样 $N_n$ 个邻近顶点和 $N_r$ 个随机顶点构成负样本，避免二次复杂度。推理时穷举所有顶点对以确保完整性。

---

## 损失函数

VAE 端到端训练，总损失为：

$$\mathcal{L}_{\text{total}} = \mathcal{L}_{\text{prune}} + \mathcal{L}_{\text{vtx}} + \mathcal{L}_{\text{conn}} + \beta \mathcal{L}_{\text{KL}}$$

| 损失项 | 公式 | 作用 |
|---|---|---|
| $\mathcal{L}_{\text{prune}}$ | 各层级 BCE loss（Eq. 4） | 监督每层体素占用概率 |
| $\mathcal{L}_{\text{vtx}}$ | Asymmetric Loss（Eq. 5），$\gamma_- = 4$, $\gamma_+ = 0$ | 最终层顶点定位，通过非对称权重解决正负样本不平衡 |
| $\mathcal{L}_{\text{conn}}$ | BCE loss（Eq. 6） | 监督边连接预测 |
| $\mathcal{L}_{\text{KL}}$ | KL 散度正则化 | 约束潜空间分布 |

其中 Asymmetric Loss 的关键在于下调负样本权重（$\gamma_- > \gamma_+$），因为在高分辨率体素中顶点只占极小比例。

---

## Flow Matching 生成

生成阶段采用类似 TRELLIS 的两阶段级联 flow matching：

### Stage 1：结构体素生成

- 从 TRELLIS 预训练的结构模型微调而来
- 从 $32^3$ 隐变量生成 $128^3$ 分辨率的稀疏体素占用
- 微调时加入了开放表面和非流形资产的监督

### Stage 2：拓扑特征生成

- 条件 Flow Matching Transformer
- 以 Stage 1 输出的占用体素为空间锚点
- 合成 16 通道的 T-Voxel 特征
- 以顶点数量 $c_v = \log(N_v)$ 为额外条件
- 损失函数：$\mathcal{L}_{\text{Flow}}(\theta) = \mathbb{E}_{z,t,\epsilon}\left[\|v_\theta(z_t, t, c_v) - (\epsilon - z_0)\|^2\right]$

推理时先采样结构体素，再生成 T-Voxel 特征，最后送入解码器得到顶点和连接关系。

---

## 训练配置

| 参数 | 值 |
|---|---|
| PointNet 特征维度 | 1024 |
| Sparse Transformer 维度 | 512, 8 heads |
| T-Voxel 分辨率 | $128^3$ |
| T-Voxel 通道数 | 16 |
| 解码器 bottleneck 维度 | 512 |
| 采样点数 $K$ | 819,200 |
| 训练数据 | ~400K assets (TRELLIS-500K, Objaverse, Toys4K, ABO) |
| 优化器 | AdamW, cosine schedule $10^{-4} \to 10^{-5}$ |
| VAE 训练 | 8× H100, 4 天, batch=8/GPU |
| Flow Matching 训练 | 7 天, effective batch=128 |

---

## 实验结果

### VAE 重建质量

| 方法 | CD(L2) ↓ | HD ↓ | NC ↑ |
|---|---|---|---|
| MeshGPT | 0.042/0.050 | 0.070/0.096 | 0.824/0.831 |
| PivotMesh | —/0.129 | —/0.263 | —/0.614 |
| MeshCraft | 0.150/0.243 | 0.282/0.299 | 0.531/0.625 |
| **LATO** | **0.042/0.040** | **0.065/0.092** | **0.847/0.834** |

（数值格式为 dense/artistic mesh）

### 几何条件生成质量

| 方法 | CD(L2) ↓ | HD ↓ | NC ↑ |
|---|---|---|---|
| MeshAnything V2 | 0.108/0.066 | 0.227/0.137 | 0.696/0.766 |
| Mesh Silksong | 0.062/0.052 | 0.145/0.101 | 0.784/0.818 |
| FastMesh | 0.064/0.048 | 0.110/0.102 | 0.757/0.813 |
| BPT | 0.059/0.052 | 0.107/0.088 | 0.811/0.824 |
| DeepMesh | 0.051/0.046 | 0.092/0.089 | 0.828/0.827 |
| **LATO** | **0.043/0.044** | **0.084/0.081** | **0.832/0.835** |

### 推理效率

| 方法 | 类型 | 生成 ~5K 面 | 生成 ~15K 面 |
|---|---|---|---|
| DeepMesh | AR | ~416s | ~566s |
| BPT | AR | ~109s | ~252s |
| MeshSilkSong | AR | ~180s | ~300s |
| **LATO** | Flow | **~5.3s** | **~9.9s** |

LATO 的推理时间几乎不随面数增长而变化（flow matching 是并行去噪），而自回归方法的时间与面数成正比。

---

## Ablation Study

### VDF 输入的必要性

| 输入配置 | CD(L2) ↓ | HD ↓ | NC ↑ |
|---|---|---|---|
| w/o $n(p)$（去掉法线） | 0.040 | 0.076 | 0.832 |
| w/o $F(p)$（去掉 VDF） | 0.043 | 0.092 | 0.822 |
| Full model | **0.039** | **0.075** | **0.837** |

去掉 VDF 位移信息后重建质量下降，尤其在拓扑保真度（NC）上。

### 连接预测策略

几何算法（基于距离阈值）在重建时勉强可用，但在生成样本上因分布偏移而崩溃；学习式 Connection Head 在两种情况下都稳定工作。

---

## 局限

- VDF 分辨率受限于底层稀疏体素网格，难以表示极小三角面片或超精细几何细节
- 连接头在推理时需要穷举所有顶点对，计算复杂度随顶点数二次增长
- 当前生成的 mesh 面数和质量不及工业级资产（如 Blender 制作的），仍有差距
- 依赖 TRELLIS 预训练的结构模型作为第一阶段，并非完全独立的生成管线

---

## 与其他方法的定位

LATO 在 3D mesh 生成的方法谱中占据一个独特的位置：

| 路线 | 代表方法 | 拓扑 | 效率 | 几何 |
|---|---|---|---|---|
| 隐式场 + MC | TRELLIS, TripoSG | 密集不规则 | 快 | 好 |
| 自回归 | MeshGPT, BPT, DeepMesh | 可控 | 慢 | 受限 |
| **LATO (T-Voxels)** | **本文** | **可控** | **快** | **好** |

LATO 是第一个在保持 flow matching 效率的同时实现显式拓扑建模的方法，为未来结合两条路线的优势提供了新的范式。

---

## 总结

LATO 通过 Vertex Displacement Field 将 mesh 拓扑信息转化为连续密集信号，再通过 Sparse Voxel VAE 压缩为 T-Voxels 隐空间，使 flow matching 模型能直接生成带有显式顶点位置和连接关系的 mesh。它在几何精度上匹敌隐式场方法，在拓扑质量上匹敌自回归方法，同时推理效率高出一个数量级。
