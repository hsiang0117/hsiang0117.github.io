---
title: "3DGS-Volume-Cloud"
date: 2026-06-12
tags: ["3DGS", "Volume Rendering", "CUDA", "Relighting", "Research"]
ShowToc: true
summary: "用物理参数化的 3D Gaussian Splatting 实现实时体积云渲染与动态太阳光重打光。"
---

## 3DGS-Volume-Cloud

[GitHub](https://github.com/hsiang0117/3DGS-Volume-Cloud)

一个用**物理参数化 3D Gaussian Splatting** 替代游戏引擎中 ray-marching 体积云的研究项目，目标是实时渲染 + **动态打光**（推理时可任意替换太阳方向）。

基于 [3DGS (Kerbl et al., 2023)](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/) 代码框架，对表示、着色、光栅化器和训练管线做了参与介质（participating medium）方向的重构。

### 演示

<video controls muted loop playsinline style="width: 100%; border-radius: 8px;">
  <source src="/3DGS-Volume-Cloud/video.mp4" type="video/mp4">
  您的浏览器不支持 video 标签。
</video>

### 与原版 3DGS 的核心差异

- ☁️ **物理化的高斯参数** — 每个高斯不再携带 SH 颜色 + 经验 opacity，而是介质物理量：峰值消光系数 `β_peak`、散射反照率 `ρ`（RGB）、Henyey-Greenstein 相函数偏度 `g`，以及可学习的多次散射八度权重。
- 📐 **解析光学厚度光栅化** — fork 的 CUDA 光栅化器逐像素累积高斯沿视线的解析线积分光学厚度 `τ`，以物理正确的 Beer-Lambert 消光（`α = 1 − exp(−τ)`）替代启发式 alpha 混合。
- ☀️ **光源视角自阴影（T_light）** — 光照空间光栅化 pass 记录每个高斯朝向太阳的透射率，配备**原生可微 backward**，把阴影梯度传播给所有前方遮挡者（含经 scale/rotation 的完整几何梯度）。
- 💡 **物理着色与重打光** — 逐高斯辐射结合 HG 相函数、六阶多次散射八度与自阴影透射率；太阳方向是逐帧输入，训练好的云可从任意方向重新打光。
- 🪡 **针手术** — 结构性重写 pass：劈分高各向异性高斯并守恒消光质量，等效各向异性硬上限，不与光度梯度拔河。
- 🌱 **物理化致密化** — 基于贡献度的剪枝（逐高斯 `Σ(α·T)` CUDA 通道）和 `β_peak` 复活机制，替代原版的 opacity 重置启发式。

### 数据集

UE5 渲染的体积云（WDAS cloud VDB）：60 个 Fibonacci 均匀半球太阳方向 × 轮转相机 = 1458 帧，NeRF-synthetic 格式并带逐帧 `sun_direction`，其中 4 个太阳方向整体 held-out 用于重打光泛化测试。

### 技术栈

`Python` · `CUDA` · `C++` · `PyTorch` — 自定义可微光栅化器（fork 自 diff-gaussian-rasterization），基于 viser 的交互式查看器支持实时调节太阳方向。
