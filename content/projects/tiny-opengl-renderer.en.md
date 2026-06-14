---
title: "TinyOpenGLRenderer"
date: 2025-04-01
tags: ["OpenGL", "C++", "Real-time Rendering", "Engine"]
ShowToc: true
summary: "A mini real-time renderer / scene editor built on OpenGL 4.5 + C++17 — forward HDR pipeline, shadows, skeletal animation, raymarched volumetric clouds, and a Dear ImGui docking editor."
---

## TinyOpenGLRenderer

[GitHub](https://github.com/hsiang0117/TinyOpenGLRenderer)

A mini real-time renderer and scene editor built on **OpenGL 4.5 + C++17**, made for learning and experimenting with modern rendering techniques. Component-driven ECS-lite architecture, a RenderPass pipeline, and a Dear ImGui (docking) editor UI.

### Screenshots

![Editor overview (docking layout)](https://raw.githubusercontent.com/hsiang0117/TinyOpenGLRenderer/master/data/docked_layout.png)

<img src="https://raw.githubusercontent.com/hsiang0117/TinyOpenGLRenderer/master/data/volumeCloud.gif" alt="Volumetric clouds" loading="lazy" style="width: 100%;">

<img src="https://raw.githubusercontent.com/hsiang0117/TinyOpenGLRenderer/master/data/animation.gif" alt="Skeletal animation" loading="lazy" style="width: 100%;">

### Rendering

- 🎨 **Forward HDR pipeline** — RGBA16F render targets, Bloom (ping-pong Gaussian blur), exposure tone mapping
- 💡 **Multiple lights** — directional / point / spot, light data uploaded via SSBO
- 🌑 **Shadows** — 2D shadow map for the directional light, cube map array for omnidirectional point-light shadows
- ☁️ **Volumetric clouds** (Horizon-style raymarch) — Perlin-Worley noise (low-freq FBM shaping + high-freq Worley edge erosion, domain warping, weather-map coverage), dual-lobe HG phase, 4-octave multi-scattering approximation, powder effect, self-shadowing light march; rendered at half resolution with depth-aware bilateral upsampling and empty-space skipping. All 15 cloud parameters (type, coverage, noise scale, lighting, wind…) are component-driven and editable live in the Inspector
- 🦴 **Skeletal animation** — Assimp import, GPU skinning, bone debug overlay
- 🌅 Skybox and async model loading (PLY + common formats)

### Editor

- 🧩 **Dear ImGui docking** — scene tree / Inspector / render settings / assets / render-to-texture viewport
- 🔧 **Component Inspector** — dedicated panels for Transform, lights, materials, volumetric clouds
- 📌 **Selection gizmo** — XYZ axis arrows at the selected object's origin (rotates with the object, always on top, constant screen size)
- 📜 **In-app log console** — GL debug output and engine logs inside the editor, no system console window

### Architecture

- **ECS-lite** — GameObject + components, per-type cached scene views
- **RenderPass pipeline** — FrameSetup → Shadows → LightUpload → ForwardHdr → Volume/Skeleton → Gizmo → Bloom → Composite, passes share state only through a RenderContext
- **Dependency injection** — Engine as composition root, no singletons (except the logger)
- **RAII GL resources** — move-only wrappers for textures / FBOs / buffers

### Tech stack

`C++17` · `OpenGL 4.5` — GLFW, glad, glm, Assimp, Dear ImGui (docking), stb_image. Builds with Visual Studio 2022 (x64), all dependencies bundled in-repo.
