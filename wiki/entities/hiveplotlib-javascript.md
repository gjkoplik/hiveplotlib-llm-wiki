---
title: hiveplotlib-javascript
type: entity
created: 2026-04-06
updated: 2026-10-03
sources: [hiveplotlib-javascript-repo]
tags: [hiveplotlib-javascript, javascript, d3, software]
---

# hiveplotlib-javascript

JavaScript/D3.js visualization library for rendering [[hive-plot|hive plot]] JSON exports from [[hiveplotlib]]. Published as `@hiveplotlib/d3` on npm.

## Status

- **Current version:** 0.2.0
- **Node support:** 20+
- **Repository:** GitHub
- **Install:** `npm install @hiveplotlib/d3` or via CDN

## What It Does

Takes the JSON output of `hiveplotlib.HivePlot.to_json()` and renders it as an interactive SVG in the browser. Handles all matplotlib-style kwargs (color, alpha, linewidth, linestyle, colormaps, scatter sizes) by mapping them to SVG/D3 equivalents.

## Architecture

Single ESM source file (~650 lines). Four layers: kwarg normalization → SVG setup → plot functions (axes, nodes, edges, labels) → async entry point. D3 v7 imported from CDN.

## Relationship to hiveplotlib

This is the **front-end complement** to [[hiveplotlib]]'s Python computation. The Python library handles all network analysis, node placement, and [[bezier-curves|Bézier curve]] generation. The JS library handles rendering. The bridge is a JSON format containing axis definitions, pre-computed node coordinates, pre-computed edge curves, and visualization kwargs.

## Related experiments

- **`hiveplotlib-trade-demo`** (private GitHub repo, first and only commit 2026-09-30). A throwaway single-page demo of 2019 minor-metals trade (HS 8112) testing whether click-to-highlight and animated axis re-sorting make a good showcase. It does not use this library or `to_json()`: a bespoke export script writes each country's axis position under four sorting variables, and the page redraws each edge as an SVG quadratic Bézier from the same closed-form control point [[hiveplotlib]] uses (midpoint angle between the two axes, mean radius of the endpoints), checked against hiveplotlib's sampled curves at export time. Animation interpolates radii and redraws curves per frame, which pre-computed curve JSON cannot do. Open question, the maintainer's: fold the interaction into this library, rebuild the demo on it, or keep them separate, and whether `to_json()` should replace the bespoke export. The demo came out of a 2026-09-30 first read of Datoviz (GPU plotting), where the maintainer ruled out any integration and kept datashader as the at-scale answer; plain browser SVG was the recommendation for an interactive demo at this size.

## See Also

- [[hiveplotlib]] — Python library (generates the data)
- [[hive-plot]] — Core concept
