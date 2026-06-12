---
title: "WaterModifier"
date: 2026-04-06
tags: ["Unreal Engine", "Segment Anything", "GIS", "C++", "Python"]
ShowToc: true
summary: "An AI-assisted desktop tool for editing water masks in Cesium quantized-mesh terrain datasets — click a few points on the map, Segment Anything extracts the water, and the edit propagates through every LOD level."
---

## WaterModifier

[GitHub](https://github.com/hsiang0117/WaterModifier)

An AI-assisted desktop tool for editing **water masks in geospatial terrain datasets**. Browse a satellite tile map, click a few foreground / background points, let **Segment Anything** extract the water region, and write the result directly back into Cesium quantized-mesh `.terrain` tiles — automatically synchronized across LOD levels.

Built as two cooperating processes: an **Unreal Engine 5.4** frontend (map browsing, visualization, interaction) and a **Python backend** (SAM inference + terrain file surgery), talking over a TCP socket with a length-prefixed protocol.

<img src="https://raw.githubusercontent.com/hsiang0117/WaterModifier/main/readme/video.gif" alt="WaterModifier demo" loading="lazy" style="width: 100%; border-radius: 8px;">

### Frontend (UE 5.4, C++)

- 🗺️ **TMS tile-map browser** — top-down orthographic camera with drag / scroll-zoom; the tile manager diffs the viewport's tile range each frame and streams in only newly visible tiles
- 🌐 **Live geo-coordinates** — real-time lat/lon readout derived from the tileset's `units-per-pixel` metadata; LOD switching re-anchors the camera to the same geographic position
- 💧 **Water-mask visualization** — parses the quantized-mesh `.terrain` binary format directly in C++ (vertex / index blocks, edge indices, then the extension records — extension type 2 is the water mask) and overlays existing water in blue

### Backend (Python + PyTorch)

- 🤖 **SAM segmentation** — the current viewport is exported as EXR, tone-mapped, and fed to Segment Anything (ViT-B, CUDA); left-clicks are foreground prompts, right-clicks background — iterate points until the mask is right
- ✍️ **Manual mode** — Photoshop-pen-style polygon selection, rasterized with ray-casting point-in-polygon tests
- 📝 **In-place terrain editing** — per-tile mask merge supporting both *overwrite* and *additive* modes, handling uniform 1-byte masks and full 256×256 masks, and appending the water-mask extension to tiles that never had one
- 🧹 **Morphological cleanup** — opening / closing passes remove segmentation noise, followed by smoothing convolutions so edited water edges blend naturally
- 🔁 **LOD synchronization** — edits recursively propagate to higher zoom levels by quadrant-splitting the parent mask and nearest-neighbor upsampling, all the way down to the dataset's max LOD
- 📊 **Live progress** — a timer thread streams the modified-tile count back to the UI every 0.5 s during bulk writes

### Tech stack

`UE 5.4 / C++` · `Python / PyTorch` — segment-anything (ViT-B), NumPy / SciPy / OpenCV, TcpSocketPlugin; operates on TMS tile pyramids and Cesium quantized-mesh-1.0 terrain.
