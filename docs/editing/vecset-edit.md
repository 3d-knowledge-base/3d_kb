---
comments: true
---

# VecSet-Edit

> *VecSet-Edit: Unleashing Pre-trained LRM for Mesh Editing from Single Image*

VecSet-Edit 的重要性在于，它把编辑从 voxel-based latent 推到了 VecSet latent。论文的核心观察是：即使 VecSet token 是无序集合，它们仍然对应相对稳定的局部表面区域，因此可以做局部编辑。

---

## 核心问题

已有原生 3D 编辑方法里，很多依赖 voxel latent：

- 好处是空间位置明确
- 代价是分辨率受限，而且常常要额外 3D mask

VecSet-Edit 要回答的是：

> 如果 backbone 不是 voxel，而是更紧凑的 VecSet latent，能不能也做到局部、可控、且高保真的 mesh editing？

---

## 方法框架

论文建立在 TripoSG 这类 VecSet LRM 上，核心模块包括：

### 1. Mask-guided Token Seeding

- 从 2D 编辑 mask 出发，先选出一批粗粒度 editable tokens
- 不需要额外 3D mask

### 2. Attention-aligned Token Gating

- 进一步保留和编辑区域相关性最高的 token
- 缩小编辑范围，减少 spillover 到未编辑区域

### 3. Drift-aware Token Pruning

- VecSet token 在 denoising 过程中不是固定在栅格上的，会漂移
- 若漂移失控，编辑区域和保留区域会互相干扰
- 因此论文显式删除那些与参考几何不一致的漂移 token

### 4. Detail-preserving Texture Baking

- 最后重新烘焙纹理
- 只在必要区域更新，尽量保留原始 mesh 的高频外观细节

---

## 为什么它重要

VecSet-Edit 的意义不只是“又做了一个 3D 编辑方法”，而是验证了：

- 局部编辑不一定非要依赖 voxel grid
- unordered token set 只要有稳定 locality，也可以支持 region-aware editing

这对后续基于 VecSet / set latent 的 3D 编辑很关键，因为它打开了另一条路线。

---

## 关键实验结论

### Edit3D-Bench 上的结果

在未编辑区域保留上，VecSet-Edit 比对比方法更强：

- `CD = 0.011`
- `PSNR = 29.63`
- `SSIM = 0.97`
- `LPIPS = 0.04`
- `DINO-I = 0.92`

对比里：

- `MVEdit`: `CD 0.188`
- `Instant3DiT`: `CD 0.124`
- `Trellis`: `CD 0.014`
- `VoxHammer`: `CD 0.018`

也就是说，在“未编辑区域尽量不动”这件事上，VecSet-Edit 做得比 voxel-based SOTA 还要更好。

### 效率

- 论文报告整体编辑时间约 `200s`
- 比 `Trellis / VoxHammer` 这种约 `600s` 的设置更快

这说明 VecSet backbone 不只提高了 fidelity，也带来了更好的效率。

### 消融实验

消融结果非常清楚：

- 只有 RePaint：`CD 0.024`
- 加 `Token Seeding` 后显著改善
- 再加 `Token Gating`、`Token Pruning` 后 DINO-I 和几何保留继续提升
- 去掉 `Detail-preserving Texture Baking` 后，外观保真明显下降

因此四个模块各自负责不同问题：定位、收缩编辑区域、控制漂移、保住纹理细节。

---

## 与 VoxHammer 的关系

两者都在做原生 3D latent editing，但差别很明显：

- `VoxHammer`：依赖 voxel latent，位置清楚，但分辨率与 3D mask 成本更高
- `VecSet-Edit`：依赖 VecSet latent，更紧凑，但必须自己解决 token-locality 和 drift 问题

所以 VecSet-Edit 更像是在证明：`set-based latent 不是不能编辑，而是需要不同的定位机制。`

---

## 局限

- 虽然不需要 3D mask，但仍需要 2D mask 和编辑图像条件
- token-locality 是经验上稳定，而不是像 voxel grid 那样天然显式
- 整体仍建立在预训练 VecSet LRM 的表示能力上，编辑上限受 backbone 约束

---

## 一句话总结

VecSet-Edit 的主要贡献，是证明了高保真 VecSet latent 也能支持局部 mesh editing：通过 `token seeding + token gating + drift pruning`，它把无序 token 集合转化成可定位、可控的编辑空间。
