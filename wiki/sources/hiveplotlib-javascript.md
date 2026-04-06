---
title: "hiveplotlib-javascript — D3 Visualization Source"
type: source
created: 2026-04-06
updated: 2026-04-06
sources: [hiveplotlib-javascript-repo]
tags: [hiveplotlib-javascript, javascript, d3, software, network-visualization]
---

# hiveplotlib-javascript — D3 Visualization Source

**Repository:** `/home/garyk/repos/hiveplotlib-javascript` (GitHub-hosted)
**Package:** `@hiveplotlib/d3` on npm
**Version:** 0.2.0
**Author:** Gary Koplik (wiki maintainer)

## Summary

[[hiveplotlib-javascript]] is the JavaScript/D3.js companion to [[hiveplotlib]]. It renders [[hive-plot|hive plot]] JSON exports from the Python library as interactive SVG visualizations in the browser. The library provides a complete pipeline from [[hiveplotlib]]'s `.to_json()` output to rendered SVG.

## Architecture

Single source file (`hive_plots_d3_viz.js`, ~650 lines, ESM module) with four rendering layers:

1. **Kwarg normalization** — Translates matplotlib-style kwargs to SVG/D3 attributes (color aliases, linestyle → dasharray, scatter size → SVG radius)
2. **SVG setup** — D3 scales, margins, canvas via marcon abstraction
3. **Plot functions** — `plotAxes()`, `plotNodes()`, `plotEdges()`, `plotLabels()`
4. **Entry point** — `visualizeHivePlot()` (async, accepts URL or in-memory object)

### Matplotlib → D3 Mapping
- **30+ colormaps** mapped to D3 interpolators (viridis, plasma, inferno, Blues, Spectral, coolwarm, etc.)
- **Linestyle** → SVG dasharray (solid, dashed, dotted, dashdot)
- **Scatter size** → circle radius: `r = sqrt(s / π)`
- **Color priority:** facecolor > color > c (matching matplotlib conventions)

### Design Decisions
- D3 imported from CDN — browser/Jupyter-ready without build step
- No production dependencies; D3 marked external in minified bundle
- Assumes matplotlib backend kwargs (other backends not supported)
- Async-first for reliable post-render interaction

## Test Suite
- Vitest with jsdom environment (~1,630 lines)
- Covers all kwarg normalization, rendering functions, and integration
- Three fixtures: minimal, multi-tag, full-kwargs

## See Also

- [[hiveplotlib-javascript]] — Entity page
- [[hiveplotlib]] — Python library (generates the JSON input)
- [[hive-plot]] — Core concept
