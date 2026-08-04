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

### Why this exists

Individual real-time rendering techniques are all well covered by tutorials online, but those tutorials tend to stand alone: one demo for shadows, another for volumetric clouds, a third for skeletal animation, with parameters hard-coded and a recompile needed to change a value. This project wanted the opposite — those techniques living in one scene, sharing a single lighting setup and a single render pipeline, with every parameter editable at runtime and the result visible immediately.

A sizeable share of the work therefore went into places that produce no pixels directly: splitting rendering into an explicit RenderPass sequence (passes exchange state only through a RenderContext), turning renderable things into components attached to a GameObject, and giving every component type its own Inspector panel. The payoff is that the volumetric cloud's dozen-plus parameters can be dragged and watched live, and several clouds with different parameters can coexist in one scene; the cost is a layer of indirection over a purpose-built demo.

### Screenshots

![Editor overview (docking layout)](https://raw.githubusercontent.com/hsiang0117/TinyOpenGLRenderer/master/data/docked_layout.png)

<img src="https://raw.githubusercontent.com/hsiang0117/TinyOpenGLRenderer/master/data/image.png" alt="Rendering showcase" loading="lazy" style="width: 100%;">

<img src="https://raw.githubusercontent.com/hsiang0117/TinyOpenGLRenderer/master/data/volumeCloud.gif" alt="Volumetric clouds" loading="lazy" style="width: 100%;">

<img src="https://raw.githubusercontent.com/hsiang0117/TinyOpenGLRenderer/master/data/animation.gif" alt="Skeletal animation" loading="lazy" style="width: 100%;">

### Rendering

- 🎨 **Forward HDR pipeline** — RGBA16F render targets, Bloom (ping-pong Gaussian blur), exposure tone mapping
- 💡 **Multiple lights** — directional / point / spot, light data uploaded via SSBO
- 🌑 **Shadows** — 2D shadow map for the directional light (front-face culling against peter-panning), cube map array for omnidirectional point-light shadows
- ✂️ **Real frustum culling** — Gribb-Hartmann 6-plane extraction + AABB tests; off-screen objects (including point lights) are never submitted for drawing
- ☁️ **Volumetric clouds** (Horizon-style raymarch) — Perlin-Worley noise (low-freq FBM shaping + high-freq Worley edge erosion, domain warping, weather-map coverage), dual-lobe HG phase, 4-octave multi-scattering approximation, powder effect, self-shadowing light march; half-resolution rendering with depth-aware bilateral upsampling, empty-space skipping, and world-scale adaptive step counts. **Noise textures are generated in parallel on worker threads** with a **versioned self-describing disk cache** (auto-invalidated on parameter changes). All 15 cloud parameters (type, coverage, noise scale, lighting, wind…) are component-driven and editable live in the Inspector — multiple different clouds per scene
- 🦴 **Skeletal animation** — Assimp import, GPU skinning, bone debug overlay
- 🌅 Skybox and async model loading (PLY + common formats)

### Editor

- 🧩 **Dear ImGui docking** — scene tree / Inspector / render settings / assets / render-to-texture viewport
- 🔧 **Component Inspector** — dedicated panels for Transform, lights, materials, volumetric clouds
- 📌 **Selection gizmo** — XYZ axis arrows at the selected object's origin (rotates with the object, always on top, constant screen size)
- 📜 **In-app log console** — GL debug output and engine logs inside the editor, no system console window
- ➕ **Data-driven Add menu** — one-click creation of lights / meshes / skybox / volumetric clouds

### Architecture

- **ECS-lite** — GameObject + components, per-type cached scene views
- **RenderPass pipeline** — FrameSetup → Shadows → LightUpload → ForwardHdr → Volume/Skeleton → Gizmo → Bloom → Composite, passes share state only through a RenderContext
- **Dependency injection** — Engine as composition root, no singletons (except the logger)
- **RAII GL resources** — move-only wrappers for textures / FBOs / buffers
- **Driver-detail handling** — static meshes bind a dummy texture on the bone-texture unit (8) so sampler completeness passes driver validation even in scenes without animation

### Tech stack

`C++17` · `OpenGL 4.5` — GLFW, glad, glm, Assimp, Dear ImGui (docking), stb_image. Builds with Visual Studio 2022 (x64), all dependencies bundled in-repo. Dependencies are bundled in-repo (`include/` + `lib/`); open the `.sln`, pick Debug/Release x64 and build. On launch it loads a default scene with a skybox, a ground plane and a directional light.
