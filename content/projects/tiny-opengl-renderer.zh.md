---
title: "TinyOpenGLRenderer"
date: 2025-04-01
tags: ["OpenGL", "C++", "Real-time Rendering", "Engine"]
ShowToc: true
summary: "基于 OpenGL 4.5 + C++17 的迷你实时渲染器 / 场景编辑器——前向 HDR 管线、阴影、骨骼动画、raymarch 体积云，以及 Dear ImGui docking 编辑器。"
---

## TinyOpenGLRenderer

[GitHub](https://github.com/hsiang0117/TinyOpenGLRenderer)

一个基于 **OpenGL 4.5 + C++17** 的迷你实时渲染器 / 场景编辑器，用于学习与实验现代渲染技术。组件驱动的 ECS-lite 架构、RenderPass 流水线、Dear ImGui（docking）编辑器界面。

### 为什么做这个

现代实时渲染的单项技术在网上都找得到教程，但这些教程往往彼此独立：一个 demo 讲阴影、另一个讲体积云、第三个讲骨骼动画，参数写死在代码里，改一个值就要重编译。这个项目要的是相反的东西——把这些技术收进同一个场景，让它们共享同一套光照与同一条渲染流水线，并且所有参数都能在运行时改、改完立刻看到结果。

因此有相当一部分工程量花在了不直接产生像素的地方：把渲染拆成显式的 RenderPass 序列（各 pass 只通过 RenderContext 交换状态），把可渲染物体做成挂在 GameObject 上的组件，再为每类组件配一个 Inspector 面板。收益是体积云那十几个参数可以边拖边看，同一场景里还能同时存在参数不同的多朵云；代价是相比一个专门写死的 demo，多了一层间接。

### 截图

![编辑器总览（docking 布局）](https://raw.githubusercontent.com/hsiang0117/TinyOpenGLRenderer/master/data/docked_layout.png)

<img src="https://raw.githubusercontent.com/hsiang0117/TinyOpenGLRenderer/master/data/image.png" alt="渲染效果" loading="lazy" style="width: 100%;">

<img src="https://raw.githubusercontent.com/hsiang0117/TinyOpenGLRenderer/master/data/volumeCloud.gif" alt="体积云" loading="lazy" style="width: 100%;">

<img src="https://raw.githubusercontent.com/hsiang0117/TinyOpenGLRenderer/master/data/animation.gif" alt="骨骼动画" loading="lazy" style="width: 100%;">

### 渲染

- 🎨 **前向 HDR 管线** — RGBA16F 渲染目标、Bloom（高斯 ping-pong 模糊）、曝光色调映射
- 💡 **多光源** — 平行光 / 点光 / 聚光，光源数据经 SSBO 上传
- 🌑 **阴影** — 平行光 2D shadow map（front-face culling 防 peter-panning）；点光 cube map array 全向阴影
- ✂️ **真·视锥剔除** — Gribb-Hartmann 六平面提取 + AABB 测试，场景树中不可见的物体（含点光源）不提交绘制
- ☁️ **体积云**（Horizon 风格 raymarch）— Perlin-Worley 噪声（低频 FBM 成形 + 高频 Worley 边缘侵蚀、域扭曲、weather map 控制覆盖率）、双叶 HG 相函数、4 阶多重散射近似、powder 暗边、自阴影 light march；半分辨率渲染 + 深度感知双边上采样、空步跳过、世界尺度自适应步数。**噪声纹理由工作线程后台并行生成**，并带**版本化自描述磁盘缓存**（参数变化自动失效重建）。云类型 / 覆盖率 / 噪声尺度 / 光照 / 风速等 15 项参数全部组件化，Inspector 实时可调，同场景可多朵不同的云
- 🦴 **骨骼动画** — Assimp 导入、GPU 蒙皮、骨骼调试叠加显示
- 🌅 天空盒与异步模型加载（PLY 及常见格式）

### 编辑器

- 🧩 **Dear ImGui docking** — 场景树 / Inspector / 渲染设置 / 资源 / 渲染到纹理的 Viewport
- 🔧 **组件式 Inspector** — Transform、光源、材质、体积云等组件均有对应编辑面板
- 📌 **选中 Gizmo** — 选中物体原点处渲染 XYZ 轴向箭头（随物体旋转、始终最前、恒定屏幕尺寸）
- 📜 **应用内日志控制台** — GL 调试输出 / 引擎日志直接显示在编辑器内，无系统命令行窗口
- ➕ **数据驱动的 Add 菜单** — 一键创建光源 / 网格 / 天空盒 / 体积云

### 架构

- **ECS-lite** — GameObject + 组件，场景视图按类型缓存
- **RenderPass 流水线** — FrameSetup → Shadows → LightUpload → ForwardHdr → Volume/Skeleton → Gizmo → Bloom → Composite，各 pass 仅通过 RenderContext 共享状态
- **依赖注入** — Engine 为组合根，无单例（Logger 除外）
- **GL 资源 RAII** — 纹理 / FBO / Buffer 均为 move-only 包装
- **驱动细节处理** — 静态网格统一绑定骨骼纹理位（单元 8）以保证采样器完整性通过驱动校验，无动画场景也不会踩 GL 验证错误

### 技术栈

`C++17` · `OpenGL 4.5` — GLFW、glad、glm、Assimp、Dear ImGui（docking）、stb_image。Visual Studio 2022（x64）构建，依赖全部捆绑在仓库内。 依赖已捆绑在仓库内（`include/` + `lib/`），打开 `.sln` 选 Debug/Release x64 直接生成；启动后自动加载一个带天空盒、地面与平行光的默认场景。
