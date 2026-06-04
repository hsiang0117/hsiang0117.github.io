---
title: "WaterModifier"
date: 2026-04-06
tags: ["C++", "模拟", "水体"]
categories: ["项目"]
ShowToc: true
summary: "交互式水体模拟与编辑工具，支持实时波形生成和粒子流体模拟。"
---

## WaterModifier

[GitHub](https://github.com/hsiang0117/WaterModifier)

交互式水体模拟与编辑工具，探索实时流体模拟技术。

### 功能特性

- 🌊 **实时水面模拟** — 基于 FFT 的海洋波形
- 💧 **粒子流体** — SPH (Smoothed Particle Hydrodynamics)
- 🖱️ **交互编辑** — 拖拽产生波纹、障碍物放置
- 🎮 **实时预览** — 可调参数即时反馈

### 技术栈

- **语言**: C++
- **图形**: OpenGL
- **UI**: ImGui
- **数学**: Eigen / GLM

### 参考

- Tessendorf 海洋模拟论文
- Müller SPH 论文
