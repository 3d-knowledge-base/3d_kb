---
comments: true
---

# Hi3DGen

> *Hi3DGen: High-fidelity 3D Geometry Generation from Images via Normal Bridging*

Hi3DGen 的核心判断很直接：单张 RGB 图像里包含了语义，但对几何细节并不够友好；如果想把几何保真度继续往上推，应该先显式恢复 normal，再让 3D 生成模型去利用这些 normal。

---

## 核心问题

已有 image-to-3D 方法在几何细节上常见几个瓶颈：

- 训练图像和真实输入之间有 domain gap
- RGB 图像本身存在光照、纹理、阴影歧义
- 直接从 RGB 学 3D，细小结构和锐边容易被抹平

Hi3DGen 的目标可以概括成：

> 用 normal 作为中间桥梁，把 2D 图像里的几何线索先抽出来，再交给 3D 生成骨干。

---

## 方法框架

论文由三个部分组成：

### 1. NiRNE：image-to-normal estimator

- 先把 RGB 图像转换为 normal map
- 用 noise injection 和 dual-stream training 拆开低频结构与高频细节
- 目标是同时获得稳定、可泛化、而且足够锐利的 normal

这里的重点不是“多一个 normal 任务”，而是把 normal 当作 2D 到 3D 之间的几何接口。

### 2. NoRLD：normal-regularized latent diffusion

- 在 TRELLIS 风格的 3D 生成骨干上加入 normal regularization
- normal 不只是额外条件，而是明确约束几何学习方向
- 生成时更倾向于保住局部边缘、折线和表面细节

### 3. DetailVerse 数据构建

- 论文额外构建了高细节 3D 资产数据
- 用合成数据补足人类资产中高频几何细节不足的问题
- 最终清洗后保留约 `700K` 高质量 mesh

所以 Hi3DGen 不是单模块改进，而是把 `normal bridge + 数据补强 + latent diffusion` 组合起来。

---

## 为什么 normal bridging 有用

论文的关键观点是：

- RGB 更擅长表达语义和材质
- normal 更直接表达表面方向和局部形状

因此，若先把图像变成 normal，再让 3D 模型根据 normal 学几何，会比直接从 RGB 生成 mesh 更容易保住精细结构。

这也解释了为什么它特别关注：

- sharp normal estimation
- normal-to-geometry fidelity
- detail-rich training data

---

## 关键实验结论

### Normal 估计

在 LUCES-MV 上，NiRNE 相比其他 normal estimator 有明显优势：

- `NE = 21.837`
- `SNE = 26.628`

对比里：

- `GeoWizard`: `NE 31.381`, `SNE 36.642`
- `StableNormal`: `NE 31.265`, `SNE 37.045`
- `GenPercept`: `NE 28.050`, `SNE 35.289`

这里特别值得看的是 `SNE`，因为它更强调 sharp edge 区域，而 Hi3DGen 正是在这些局部细节上发力。

### 数据集

论文给出的 DetailVerse 统计也说明它确实偏向高细节几何：

- `700K` 合成高质量资产
- 平均 sharp edge 数约 `45,773`
- 中位数约 `14,521`

相比 Objaverse 系列，这个 sharp-edge 密度高很多。

### 3D 生成效果

- 论文用户研究里，Hi3DGen 在几何保真度上优于 Hunyuan3D-2.0、Trellis、Clay、Tripo-2.5、Dora 等方法
- 消融显示，若不用 normal bridge，而是直接 image-to-3D，结果更容易出现假细节和不稳定几何

因此 Hi3DGen 的收益不是只来自更大数据，而是 `normal bridge` 和 `DetailVerse` 共同作用。

---

## 它在文献线里的位置

Hi3DGen 很适合放在这条线上理解：

- `TRELLIS / TripoSG / Hunyuan3D`：把大规模 image-to-3D 做到可用
- `Hi3DGen`：把几何 fidelity 继续往上推，尤其强调细节和锐边

它不像 O-Voxel 或 FACE 那样是在表示层换赛道，而是在现有 LRM / latent diffusion 路线上，把“几何中间信号”这件事做得更明确。

---

## 局限

- normal bridge 增强的是几何细节，但并没有从根本上解决单视图遮挡信息缺失
- 方法依赖高质量 normal estimator，如果前段 normal 出错，后段 3D 结果也会受影响
- 它主要关注 geometry fidelity，对完整 asset 的材质与编辑能力不是论文重点

---

## 一句话总结

Hi3DGen 的主要价值，是把 `RGB -> normal -> geometry` 这条桥接链路明确引入 image-to-3D 流程，用 normal 作为几何中间表示来提升高频细节与锐边保真度。
