---
title: hiveplotlib-javascript
type: entity
created: 2026-04-06
updated: 2026-04-06
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

## See Also

- [[hiveplotlib]] — Python library (generates the data)
- [[hive-plot]] — Core concept
