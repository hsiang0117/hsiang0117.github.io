---
title: "TinyOpenGLRenderer"
date: 2025-04-01
tags: ["OpenGL", "C++", "渲染器"]
categories: ["项目"]
ShowToc: true
summary: "从零构建的迷你 OpenGL 渲染器，支持 PBR、阴影映射和延迟渲染管线。"
---

## TinyOpenGLRenderer

[GitHub](https://github.com/hsiang0117/TinyOpenGLRenderer)

一个从零构建的迷你 OpenGL 渲染器，用于学习现代图形渲染管线。

### 功能特性

- 🎨 **PBR 着色** — Cook-Torrance BRDF，金属度/粗糙度工作流
- 🌑 **阴影映射** — 方向光 CSM (Cascaded Shadow Maps)
- 📐 **延迟渲染** — G-Buffer + 多光源支持
- ☀️ **IBL** — 基于图像的照明（漫反射 + 镜面反射）
- 🖼️ **HDR + 色调映射** — Reinhard / ACES 色调映射

### 技术栈

- **语言**: C++ 17
- **图形 API**: OpenGL 4.6, GLSL
- **依赖**: GLFW, GLAD, GLM, Assimp, stb_image
- **构建**: CMake

### 架构

```
TinyOpenGLRenderer/
├── src/
│   ├── core/       # 渲染核心、着色器管理
│   ├── renderer/   # PBR、阴影、后处理
│   ├── scene/      # 场景图、模型加载
│   └── utils/      # 相机、输入、资源管理
└── shaders/        # GLSL 着色器
```
