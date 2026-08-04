---
title: "WaterModifier"
date: 2026-04-06
tags: ["Unreal Engine", "Segment Anything", "GIS", "C++", "Python"]
ShowToc: true
summary: "An AI-assisted desktop tool for editing water masks in Cesium quantized-mesh terrain datasets — click a few points on the map, Segment Anything extracts the water, and the edit propagates through every LOD level."
---

## WaterModifier

[GitHub](https://github.com/hsiang0117/WaterModifier)

An AI-assisted desktop tool for editing **water masks in geospatial terrain datasets**. Browse a satellite tile map, click a few foreground / background points, let **Segment Anything** extract the water region, and write the result directly back into Cesium quantized-mesh `.terrain` tiles — automatically synchronized across LOD levels. Validated on a real GIS dataset (Yaohu Airport, maxzoom 18).

Built as two cooperating processes: an **Unreal Engine 5.4** frontend (map browsing, visualization, interaction) and a **Python backend** (SAM inference + terrain file surgery), talking over a TCP socket with a length-prefixed protocol.

### Why this exists

In a Cesium quantized-mesh dataset the water mask is not a layer you can open and save on its own — it lives as an extension record at the tail of every individual `.terrain` tile file. Its offset is not even fixed: you have to skip the 88-byte header, then step over the vertex data, the triangle indices (index width switching between 2 and 4 bytes depending on triangle count) and the west / south / east / north edge-index lists, each by its own length prefix, before you can scan the extension records and find out whether type 2 is there at all. The same body of water is also duplicated down the LOD pyramid — edit one level and every level below has four times as many tiles needing the same change.

So a human-scale editing intent — "this reservoir should be water" — lands on the data as byte-level rewrites of thousands of tile files. WaterModifier exists to connect those two ends: a tile pyramid laid out on screen at true geographic coordinates and a few mouse clicks on one side, bulk binary rewriting and level-by-level propagation on the other.

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

### Key engineering points

- **Dual-end binary format consistency** — Cesium quantized-mesh-1.0 is parsed twice, once per end: Python implements the *writer* (manually walking header / vertex blocks / index blocks / edge blocks / extension records, with 2/4-byte index width chosen by triangle count), UE C++ implements the *reader* (visualization) — both ends must agree byte-for-byte on the format
- **Camera ↔ tile coordinate chain** — a C++ function library implements the full pipeline from camera view → tile numbers → geo-coordinates (including tilemapresource.xml parsing and `units-per-pixel` conversion)

### Design decisions and trade-offs

- 🔌 **Commands over the socket, pixels over the filesystem** — the protocol carries only command keywords plus string arguments (click coordinates, LOD, terrain root, bottom-left tile indices, ortho width, tile size), with short replies like `SegmentDone` / `ModifyDone`. The two images are handed over as files in the working directory instead: UE exports the current viewport's render target as EXR for Python to segment, and Python writes a mask PNG that UE reloads as a runtime texture. The rejected alternative was pushing pixel buffers through the socket; the cost is that both processes are pinned to one shared working directory.
- 🧭 **Walking segment by segment rather than seeking to the mask** — the mask offset depends on the vertex count, triangle count and index width, so a fixed offset is not an option and the code steps through using each segment's own length prefix. The cost is that the same traversal is implemented twice — Python writes, UE C++ reads — and both sides' understanding of field offsets and skip rules has to stay in step permanently.
- 📦 **The writer always emits the full grid** — a mask payload is either a 1-byte uniform all-water / all-land flag or a full 256×256 = 65,536-byte grid. A 1-byte payload is promoted outright: rewrite the length field, drop that single byte, append the full grid; a tile with no mask extension at all gets one appended at the end of the file. Keeping the compact 1-byte form would save space but would require two branches at every downstream site; the cost as built is a whole-file read-modify-rewrite rather than an in-place byte patch.
- 📄 **Three filenames as the dataset contract** — the tile-map root must hold `tilemapresource.xml` (supplying the per-LOD units-per-pixel list) and `meta.json` (supplying `maxzoom`), and the terrain root must hold `layer.json`. `maxzoom` comes from a file rather than being inferred from the pyramid on disk: the benefit is predictable behaviour with no directory scan, the cost is that a dataset has to provide that file by convention.
- ⏱️ **Progress reported by a self-re-arming timer** — during a bulk edit a 0.5 s timer writes the completed tile count back to the UI, incrementing once per `.terrain` file written, including recursively synced children. It is simple and stays out of the write loop; the cost is a stream that bypasses the length-prefixed protocol and has to be recognized separately on the receiving end.

### Tech stack

`UE 5.4 / C++` · `Python / PyTorch` — segment-anything (ViT-B), NumPy / SciPy / OpenCV, TcpSocketPlugin; operates on TMS tile pyramids and Cesium quantized-mesh-1.0 terrain. Segmentation uses SAM ViT-B (`segment_anything` pinned to a specific upstream commit, torch 2.4.1 + cu124); the UI offers five ortho-width settings — 256 / 512 / 768 / 1024 / 2048 — with 1024 as the default.
