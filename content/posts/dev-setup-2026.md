---
title: "图形学开发环境搭建 (2026)"
date: 2026-05-20
tags: ["工具", "C++", "环境配置"]
categories: ["工具"]
ShowToc: true
summary: "一个适合图形学开发的工具链配置：编译器、调试器、性能分析、着色器编辑。"
---

## 编译器与构建

### MSVC (Visual Studio 2022)
Windows 上图形学开发的首选，调试体验最好：
- **RenderDoc** 插件集成，一键抓帧分析
- **HLSL/GLSL 语法高亮** 通过扩展实现
- PDB 符号调试，查看 GPU 端变量

### CMake
跨平台构建，几乎成为图形项目的标配：
```cmake
cmake_minimum_required(VERSION 3.20)
project(TinyRenderer)

find_package(OpenGL REQUIRED)
find_package(glfw3 REQUIRED)

add_executable(renderer main.cpp)
target_link_libraries(renderer OpenGL::GL glfw)
```

## 图形调试

### RenderDoc
**必装工具**。抓取每一帧的 Draw Call、Shader 输入输出、纹理内容。

### NVIDIA Nsight
NVIDIA GPU 专用，profile 到 shader 指令级别。

## 编辑器

| 工具 | 用途 |
|------|------|
| VS Code + GLSL lint | 轻量着色器编辑 |
| Visual Studio | 主力 C++ 开发 |
| RenderDoc | 图形调试 |
| Blender | 模型/场景编辑 |

## 推荐库

```bash
# vcpkg 管理依赖
vcpkg install glfw3 glm assimp stb
vcpkg install imgui[docking-experimental]
```

### 必备 C++ 库

- **GLFW** — 窗口和上下文管理
- **GLAD** — OpenGL 加载器
- **GLM** — 数学库（仿 GLSL 语法）
- **Assimp** — 模型导入
- **stb_image** — 纹理加载
- **Dear ImGui** — 调试 UI

## 小结

图形学开发的核心工具链并不复杂：一个好的 C++ 编译器 + RenderDoc + 顺手的编辑器就够了。关键是**多写 Shader，多调试 Draw Call**。
