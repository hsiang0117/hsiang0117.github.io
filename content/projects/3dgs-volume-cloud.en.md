---
title: "3DGS-Volume-Cloud"
date: 2026-06-12
tags: ["3DGS", "Volume Rendering", "CUDA", "Relighting", "Research"]
ShowToc: true
summary: "Physically-parameterized 3D Gaussian Splatting for real-time volumetric cloud rendering with arbitrary-sun relighting."
---

## 3DGS-Volume-Cloud

[GitHub](https://github.com/hsiang0117/3DGS-Volume-Cloud)

A research project that replaces ray-marched volumetric clouds in game engines with **physically-parameterized 3D Gaussian Splatting**, targeting real-time rendering with **dynamic relighting** (arbitrary sun direction at inference time).

Built on the [3DGS (Kerbl et al., 2023)](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/) codebase, with the representation, shading, rasterizer, and training pipeline reworked for participating media.

### Why this exists

Volumetric clouds in game engines have long relied on ray-marching: dozens of steps along the view ray per pixel, plus a light march at every sample, with cost climbing as resolution and cloud depth grow. 3D Gaussian Splatting replaces the per-pixel stepping with rasterization, but the cost lands on the representation — a vanilla Gaussian carries SH color plus a heuristic opacity, which describes "what color this looks like from a given direction," not "how thick this medium is, how much it scatters, and in which direction." Fitting a cloud with that bakes in one lighting condition: move the sun and the reconstruction no longer holds.

So the problem here is not "can a cloud be fitted convincingly" but **whether the representation itself can be made physical while keeping rasterization speed** — giving every Gaussian participating-medium quantities like extinction coefficient, scattering albedo and phase eccentricity, so that sun direction becomes an input at inference time rather than a constant baked in during training. That decision propagates all the way down: optical depth has to be integrated analytically instead of alpha-blended heuristically, self-shadowing has to be differentiable so shadow gradients can flow back, and every opacity-based heuristic in densification and pruning has to be replaced by something that still holds under the physical parameterization.

### Demo

<video controls muted loop playsinline style="width: 100%; border-radius: 8px;">
  <source src="/3DGS-Volume-Cloud/video.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

### Two-stage design

- **Stage 1** — train a physically-stable Gaussian point set on a sun-only, black-background dataset.
- **Stage 2** — **freeze** Stage 1's geometry and physical parameters and train only a global environment-lighting network, layering sky-atmosphere shading on top of the sun term for full relighting from arbitrary sun directions.

### Physical Gaussian parameters

Each Gaussian carries medium quantities instead of SH color + heuristic opacity:

| Parameter | Meaning | Activation |
|-----------|---------|------------|
| `σ_t` | peak extinction coefficient (1/m) | softplus, clamp 5 |
| `ω` | scattering albedo (RGB) | sigmoid |
| `g` | Henyey-Greenstein phase eccentricity | 0.8·tanh, forward-scattering |
| `w_n` | 6-octave multi-scattering energy weights | softplus |

The parameterization is a **coherent physical design**, not a patch: `σ_t` is intensive, so densification clones inherit it as-is without halving; opacity is the analytic quantity `1−exp(−τ)` rather than a learnable parameter, so the stock opacity-reset heuristic is replaced by per-point "contribution resurrection."

### Key differences from vanilla 3DGS

- 📐 **Analytic optical-depth rasterization** — the forked CUDA rasterizer accumulates each Gaussian's analytic line integral of optical depth `τ` per pixel, giving physically correct Beer-Lambert extinction (`α = 1 − exp(−τ)`) instead of heuristic alpha blending.
- ☀️ **Light-space self-shadowing (T_light)** — a light-space raster pass records each Gaussian's sun-ward transmittance (energy-weighted over the whole sunlit footprint, not center-sampled), with a **natively differentiable backward** that propagates shadow gradients to all occluders (full geometric gradients through scale/rotation).
- 💡 **Physical shading & relighting** — per-Gaussian radiance combines the HG phase function, six multi-scattering octaves, and self-shadow transmittance; the sun direction is a per-frame input, so trained clouds can be relit from arbitrary directions.
- 🪡 **Needle surgery (structural anisotropy cap)** — measured aniso tails are ~95% thin *disks*, not needles, so the pass **fattens the thin axis ×2** (ratio halves) instead of splitting the long axis, emitting two children offset by ±σ_major/2 along the major axis with `σ_t/3.2` extinction-mass conservation (volume ×2 × overlapping children → exact /4 over-cuts; /3.2 is mass-neutral in practice). It acts as a hard cap without fighting photometric gradients, converging in log2 passes.
- 🌱 **Physics-aware densification & maintenance** — contribution-based pruning (per-Gaussian `Σ(α·T)` CUDA channel), `σ_t` resurrection, adaptive densify threshold; new splits/clones get a 500-iteration `prune_grace` period so they are not pruned immediately, and the maintenance loop only runs during the densify window to prevent net destruction during settle.

### Environment lighting (Stage 2)

`L = T_sun ⊙ [Stage 1 sun term] + ω · Σ_lm E_lm · V_lm`:

- **`T_sun`** — sun atmospheric transmittance, a **3-parameter analytic form** `exp(−m(θ)·τ_RGB)` (fixed-geometry Kasten-Young air mass); low-sun darkening + reddening (τ_B>τ_R) falls out structurally — a purely additive term cannot express "darkening."
- **`E_lm`** — low-order SH of the sky radiance field (small global MLP), additive in-scattering fill.
- **`V_lm`** — per-Gaussian sky-visibility SH transfer vectors, **achromatic and purely geometric**, stored as a code-level buffer rather than a learnable parameter — only the global `T_sun`/`E_lm` are new learnables, so the model cannot regress to vanilla 3DGS.
- The env network and `V_lm` persist as sidecars next to the PLY; the viewer/eval load them automatically.

### Dataset

UE5-rendered volumetric cloud (WDAS cloud VDB): 60 Fibonacci-uniform hemisphere sun directions × rotating cameras = **1458 frames** (train 1306 / test 152) in NeRF-synthetic format with per-frame `sun_direction`, including **4 fully held-out sun directions** (96 frames) as relighting generalization tests. Uniform directional coverage is a precondition for healthy geometric shadow gradients. Acquisition, coordinate conversion, and test splitting are fully scripted (`tools/`).

### Interactive viewer

A viser-based viewer: drag the sun direction live for relighting, inspect diagnostic channels (RGB / T_light / `σ_t` / depth), and optionally composite HDR sky backdrops.

### Tech stack

`Python` · `CUDA` · `C++` · `PyTorch` — custom differentiable rasterizer forked from diff-gaussian-rasterization (analytic-tau / record_front_tau / lightpass-backward passes), three-stage data pipeline (UE capture → coordinate conversion → dataset split).
