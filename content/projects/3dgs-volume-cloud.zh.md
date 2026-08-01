---
title: "3DGS-Volume-Cloud"
date: 2026-06-12
tags: ["3DGS", "Volume Rendering", "CUDA", "Relighting", "Research"]
ShowToc: true
summary: "用物理参数化的 3D Gaussian Splatting 实现实时体积云渲染与任意太阳方向重打光。"
---

## 3DGS-Volume-Cloud

[GitHub](https://github.com/hsiang0117/3DGS-Volume-Cloud)

一个用**物理参数化 3D Gaussian Splatting** 替代游戏引擎中 ray-marching 体积云的研究项目，目标是**实时渲染 + 动态打光**（推理时可任意替换太阳方向）。

基于 [3DGS (Kerbl et al., 2023)](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/) 代码框架，对表示、着色、光栅化器和训练管线做了参与介质（participating medium）方向的重构。

### 演示

<video controls muted loop playsinline style="width: 100%; border-radius: 8px;">
  <source src="/3DGS-Volume-Cloud/video.mp4" type="video/mp4">
  您的浏览器不支持 video 标签。
</video>

### 两阶段设计

- **Stage 1** —— 仅太阳光、黑背景数据集，训出物理参数稳定的高斯点集。
- **Stage 2** —— **冻结** Stage 1 的几何与物理参数，只训一个全局环境光网络，叠加天空大气对云的着色，得到任意太阳方向的全光照 relighting。

### 物理化的高斯参数

每个高斯不再携带 SH 颜色 + 经验 opacity，而是一组参与介质物理量：

| 参数 | 含义 | 激活 |
|------|------|------|
| `σ_t` | 峰值消光系数（1/m） | softplus，clamp 5 |
| `ω` | 散射反照率（RGB） | sigmoid |
| `g` | Henyey-Greenstein 相函数偏度 | 0.8·tanh，前向散射 |
| `w_n` | 6 阶多次散射八度能量权重 | softplus |

参数化是**连贯的物理设计**而非补丁：`σ_t` 是强度量（intensive），致密化 clone 时原样继承、不砍半；opacity 是 `1−exp(−τ)` 的解析量而非可学参数，因此原版的 reset-opacity 启发式被逐点"贡献度复活"（resurrect）取代。

### 与原版 3DGS 的核心差异

- 📐 **解析光学厚度光栅化** — fork 的 CUDA 光栅化器逐像素累积高斯沿视线的解析线积分光学厚度 `τ`，以物理正确的 Beer-Lambert 消光（`α = 1 − exp(−τ)`）替代启发式 alpha 混合。
- ☀️ **光源视角自阴影（T_light）** — 光照空间光栅化 pass 记录每个高斯朝向太阳的透射率（整个向阳 footprint 上的能量加权，而非中心点采样），配备**原生可微 backward**，把阴影梯度传播给所有前方遮挡者（含经 scale/rotation 的完整几何梯度）。
- 💡 **物理着色与重打光** — 逐高斯辐射结合 HG 相函数、六阶多次散射八度与自阴影透射率；太阳方向是逐帧输入，训练好的云可从任意方向重新打光。
- 🪡 **针手术（结构性 aniso 控制）** — 实测高各向异性尾部 ~95% 是"薄盘"而非针，因此**增肥薄轴 ×2**（ratio 减半）而非劈长轴，沿主轴 ±σ_major/2 偏移生成双子，`σ_t/3.2` 守恒消光质量（体积 ×2 × 双子重叠 → 精确 /4 过砍，/3.2 实测质量中性）；等效硬上限，不与光度梯度拔河，每刀 ratio 减半保证 log2 收敛。
- 🌱 **物理化致密化与维护** — 贡献度剪枝（per-Gaussian `Σ(α·T)` CUDA 通道）、`σ_t` 复活机制、自适应 densify 阈值；新分裂/克隆点带 500 迭代 `prune_grace` 宽限期，避免刚生成就被剪掉；维护回路仅在 densify 期运行，防止 settle 期的净销毁。

### 环境光（Stage 2）

`L = T_sun ⊙ [Stage 1 太阳项] + ω · Σ_lm E_lm · V_lm`：

- **`T_sun`** —— 太阳大气透射率，仅 **3 参数解析式** `exp(−m(θ)·τ_RGB)`（Kasten-Young air mass，固定几何）；低太阳变暗 + 变红（τ_B>τ_R）从结构自动落出，纯加性项表达不出"变暗"。
- **`E_lm`** —— 天空辐亮度场的低阶 SH（小全局 MLP），加性内散射填充。
- **`V_lm`** —— 逐高斯天空可见度 SH 传输向量，**无色、纯几何**，代码中以 buffer 而非可学参数存在——新增可学的只有全局 `T_sun`/`E_lm`，逐高斯不加任何色彩自由度，物理上无法退回 vanilla 3DGS。
- 环境网络与 `V_lm` 以 sidecar 存进 PLY 同目录，viewer/eval 自动加载。

### 数据集

UE5 渲染的体积云（WDAS cloud VDB）：60 个 Fibonacci 均匀半球太阳方向 × 轮转相机 = **1458 帧**（train 1306 / test 152），NeRF-synthetic 格式并带逐帧 `sun_direction`，其中 **4 个太阳方向整体 held-out**（96 帧）作为重打光泛化测试。方向均匀覆盖是几何阴影梯度健康工作的前提。采集、坐标系转换、test 切分均脚本化（`tools/`）。

### 交互 Viewer

基于 viser：实时拖动太阳方向做 relighting，可视化通道（RGB / T_light / `σ_t` / depth），可选 HDR 天空背景合成。

### 技术栈

`Python` · `CUDA` · `C++` · `PyTorch` — 自定义可微光栅化器（fork 自 diff-gaussian-rasterization，含 analytic-tau / record_front_tau / lightpass-backward 通道），三阶段采集管线（UE 采集 → 坐标系转换 → 数据集切分）。
