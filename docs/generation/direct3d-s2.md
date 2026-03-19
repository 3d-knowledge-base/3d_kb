---
comments: true
---

# Direct3D-S2

> *Direct3D-S2*

Direct3D-S2 属于 sparse volumetric latent 路线中的代表工作之一。它关注的核心不是“提出一个最全新的表示哲学”，而是把 **Sparse SDF VAE + spatial sparse attention** 这条高分辨率 3D 生成路线推进到更可扩展的阶段。

---

## 核心问题

高分辨率 3D 生成常见的痛点是：

- 3D token 一多，注意力成本爆炸
- dense voxel/SDF 路线很难 scale
- 即使结构基本正确，局部边缘、锐角、薄结构也容易糊或碎

Direct3D-S2 的目标可以概括成：

> 在保留 sparse SDF 几何表达能力的同时，解决高分辨率训练与建模成本问题。

---

## 方法直觉

它的核心由两部分组成：

### 1. Sparse SDF VAE

- 用 sparse SDF 表示几何
- 把 shape 压缩到 latent 空间
- 保持对表面细节和局部几何的表达能力

### 2. Spatial Sparse Attention

- 不是对所有 token 做全局 dense attention
- 而是在空间上只对相关的稀疏邻域做交互
- 通过压缩、选择、窗口三步，把注意力集中到真正重要的稀疏区域

这件事的意义很直接：

- 3D attention 成本可控了
- 模型可以处理更高分辨率的结构
- 同时尽量保留局部几何依赖

### 3. 统一的 sparse volumetric VAE

Direct3D-S2 还有一个容易被忽略但很重要的点：

- 输入、latent、输出都保持 sparse volumetric format
- 不再像一些早期方法那样在 point cloud / latent vec / dense volume 之间来回切换

这带来两个实际收益：

- 训练更稳
- 高分辨率重建链条更直接

---

## 它在表示发展里的位置

Direct3D-S2 适合放在这条线上理解：

- `3DShape2VecSet`：让 3D latent 先变得紧凑可建模
- `TRELLIS / SLAT`：让 latent 带上结构锚点
- `TripoSG`：把 SDF VAE + RF 大模型路线系统化
- **Direct3D-S2**：把 sparse SDF + sparse attention 路线往高分辨率继续推

所以它的价值更多体现在**工程扩展性**，而不是表示哲学上的根本转向。

---

## 为什么值得关注

Direct3D-S2 的概念影响力虽不如 TRELLIS 或 O-Voxel 那样突出，但其贡献在于明确了一个重要的工程判断：

> 如果想继续保留 SDF / field 这条路线，就必须在 sparse computation 和 attention 结构上做文章，否则高分辨率成本会迅速失控。

这意味着它对后续很多高分辨率 3D 系统都有参考价值，尤其是在：

- 高精度几何
- 大模型 3D generation
- sparse latent backbone

这几个方向上。

---

## 论文里最值得记住的实验点

### 计算效率

论文给出的 SSA 核心结论很明确：

- 在 `128k` tokens 时，forward 约 `3.9x` 加速
- backward 约 `9.6x` 加速

这说明它不是“换一个 attention 写法”，而是真的把高 token 数的 3D DiT 训练成本压下来了。

### 分辨率扩展

- 论文把训练分辨率一路推到 `1024^3`
- 只用 `8` 张 GPU 训练，而很多 volumetric 路线在更低分辨率上都需要更多卡

所以它最直接的贡献，就是把 gigascale 3D generation 从“能做 demo”推进到“训练成本可接受”。

### 图像到 3D 指标

在文中对比里，Direct3D-S2 在三项 image-shape alignment 指标上都优于 Trellis、Hunyuan3D 2.0、TripoSG、Hi3DGen：

- `ULIP-2 = 0.3111`
- `Uni3D = 0.3931`
- `OpenShape = 0.1752`

这些结果对应的不是单纯“表面更平滑”，而是生成 mesh 和输入图像之间的语义与几何对齐更强。

---

## 优势与局限

### 优势

- 延续 sparse SDF 的几何表达能力
- 更适合高分辨率场景
- sparse attention 让训练/推理更可扩展

### 局限

- 仍然依赖 field -> mesh 的恢复链条
- 对开放表面 / 原生 mesh 结构的表达不如 O-Voxel / FACE 路线自然
- 即使整体结构好，边缘和棱角仍可能偏粗糙，局部还有碎片问题
- 论文的主要突破点在高分辨率训练与注意力效率，表示层本身仍属于 sparse field 世界
- 即使只用 8 张 GPU，训练和实现门槛仍然不低，尤其依赖定制 kernel 和稀疏系统实现

---

## 一句话总结

Direct3D-S2 的意义，在于把 `Sparse SDF VAE + spatial sparse attention` 这条路线向高分辨率、可扩展的 3D 生成系统推进了一步，它更像是 sparse volumetric latent 路线中的关键工程推进节点。
