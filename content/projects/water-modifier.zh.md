---
title: "WaterModifier"
date: 2026-04-06
tags: ["Unreal Engine", "Segment Anything", "GIS", "C++", "Python"]
ShowToc: true
summary: "AI 辅助的地形水域掩码编辑工具——在地图上点几个点，Segment Anything 自动分割水域，修改结果同步到所有 LOD 等级。"
---

## WaterModifier

[GitHub](https://github.com/hsiang0117/WaterModifier)

一个 AI 辅助的桌面工具，用于编辑**地理地形数据集中的水域掩码**。浏览卫星瓦片地图，点几个前景 / 背景点，由 **Segment Anything** 分割出水域，结果直接写回 Cesium quantized-mesh `.terrain` 瓦片——并自动同步到各 LOD 等级。

由两个协作进程构成：**Unreal Engine 5.4** 前端（地图浏览、可视化、交互）+ **Python** 后端（SAM 推理 + 地形文件二进制改写），通过 TCP socket（长度前缀协议）通信。

<img src="https://raw.githubusercontent.com/hsiang0117/WaterModifier/main/readme/video.gif" alt="WaterModifier 演示" loading="lazy" style="width: 100%; border-radius: 8px;">

### 前端（UE 5.4，C++）

- 🗺️ **TMS 瓦片地图浏览器** — 顶视正交相机，拖拽 / 滚轮缩放；瓦片管理器每帧 diff 视口覆盖的瓦片范围，仅增量加载新进入视野的瓦片
- 🌐 **实时地理坐标** — 由瓦片集的 `units-per-pixel` 元数据实时换算经纬度；切换 LOD 时相机重新锚定到相同地理位置
- 💧 **水域可视化** — C++ 直接解析 quantized-mesh `.terrain` 二进制格式（顶点 / 索引块、边索引、扩展记录——类型 2 即水域掩码），将现有水域以蓝色叠加显示

### 后端（Python + PyTorch）

- 🤖 **SAM 分割** — 当前视口导出为 EXR、色调映射后喂给 Segment Anything（ViT-B，CUDA）；左键为前景提示点、右键为背景点，可反复加点迭代直到掩码满意
- ✍️ **手动模式** — Photoshop 钢笔式多边形选区，射线法点内测试栅格化
- 📝 **地形文件原地改写** — 逐瓦片掩码合并，支持*覆盖*与*叠加*两种模式，兼容 1 字节统一掩码和完整 256×256 掩码，并能为从未有过水域掩码的瓦片追加扩展
- 🧹 **形态学清理** — 开 / 闭运算去除分割噪点，再经平滑卷积让水域边缘自然过渡
- 🔁 **LOD 同步** — 修改递归下推至更高缩放等级：父瓦片掩码四象限拆分 + 最近邻上采样，一路同步到数据集最高 LOD
- 📊 **实时进度** — 批量写入期间，定时线程每 0.5 秒向 UI 回传已修改瓦片数

### 技术栈

`UE 5.4 / C++` · `Python / PyTorch` — segment-anything（ViT-B）、NumPy / SciPy / OpenCV、TcpSocketPlugin；作用于 TMS 瓦片金字塔与 Cesium quantized-mesh-1.0 地形数据。
