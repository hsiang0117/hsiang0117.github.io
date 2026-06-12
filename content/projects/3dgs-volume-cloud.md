---
title: "3DGS-Volume-Cloud"
date: 2026-06-12
tags: ["3DGS", "Volume Rendering", "CUDA", "Relighting", "Research"]
ShowToc: true
summary: "Physically-parameterized 3D Gaussian Splatting for real-time volumetric cloud rendering."
---

## 3DGS-Volume-Cloud

[GitHub](https://github.com/hsiang0117/3DGS-Volume-Cloud)

A research project that replaces ray-marched volumetric clouds in game engines with **physically-parameterized 3D Gaussian Splatting**, targeting real-time rendering with **dynamic relighting** (arbitrary sun direction at inference time).

Built on the [3DGS (Kerbl et al., 2023)](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/) codebase, with the representation, shading, rasterizer, and training pipeline reworked for participating media.

### Demo

<video controls muted loop playsinline style="width: 100%; border-radius: 8px;">
  <source src="/3DGS-Volume-Cloud/video.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

### Key differences from vanilla 3DGS

- ☁️ **Physical Gaussian parameters** — each Gaussian carries medium quantities instead of SH color + heuristic opacity: peak extinction `β_peak`, scattering albedo `ρ` (RGB), Henyey-Greenstein phase eccentricity `g`, and learnable multi-scattering octave weights.
- 📐 **Analytic optical-depth rasterization** — the forked CUDA rasterizer accumulates the analytic line integral of optical depth `τ` per pixel, giving physically correct Beer-Lambert extinction (`α = 1 − exp(−τ)`) instead of heuristic alpha blending.
- ☀️ **Light-space self-shadowing (T_light)** — a light-space raster pass records each Gaussian's sun-ward transmittance, with a **natively differentiable backward** that propagates shadow gradients to all occluders (full geometric gradients through scale/rotation).
- 💡 **Physical shading & relighting** — per-Gaussian radiance combines the HG phase function, six multi-scattering octaves, and self-shadow transmittance; the sun direction is a per-frame input, so trained clouds can be relit from arbitrary directions.
- 🪡 **Needle surgery** — a structural rewrite pass that splits high-anisotropy Gaussians while conserving extinction mass, acting as a hard anisotropy cap without fighting photometric gradients.
- 🌱 **Physics-aware densification** — contribution-based pruning (per-Gaussian `Σ(α·T)` CUDA channel) and `β_peak` resurrection replace the stock opacity-reset heuristics.

### Dataset

UE5-rendered volumetric cloud (WDAS cloud VDB): 60 Fibonacci-uniform hemisphere sun directions × rotating cameras = 1458 frames in NeRF-synthetic format with per-frame `sun_direction`, including 4 fully held-out sun directions for relighting generalization tests.

### Tech stack

`Python` · `CUDA` · `C++` · `PyTorch` — custom differentiable rasterizer forked from diff-gaussian-rasterization, interactive viser-based viewer with live sun-direction control.
