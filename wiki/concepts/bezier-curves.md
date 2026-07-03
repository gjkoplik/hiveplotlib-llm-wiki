---
title: Bézier Curves
type: concept
created: 2026-04-06
updated: 2026-07-03
sources: [hiveplotlib-python-repo]
tags: [bezier-curves, edge-rendering, implementation]
---

# Bézier Curves

The mathematical curves used to draw edges in [[hive-plot|hive plots]]. In [[hiveplotlib]], edges are rendered as quadratic Bézier curves (single control point) connecting node positions across axes.

## Implementation in hiveplotlib

Key functions in `hiveplotlib/utils.py`:

- `bezier()` — 1D Bézier curve with single control point
- `bezier_all()` — Multiple 1D Bézier curves
- `bezier_xy_with_nans()` — 2D curves with NaN separators for efficient batch rendering

### Numba Acceleration

For large networks, Bézier curve generation is the computational bottleneck. [[hiveplotlib]] supports numba JIT compilation:

- **Serial mode** — JIT-compiled single-threaded (small workloads)
- **Parallel mode** — JIT-compiled across CPU cores (large workloads, threshold ~4,096 points)
- Mode selected automatically based on workload size

Edge arrays stored as float32 for memory efficiency.

## See Also

- [[edge-rendering]] — How edges use these curves
- [[hiveplotlib]] — Implementation
- [[hive-plot]] — The visualization method
- [[0002-performance-regression-harness|ADR 0002]] — The equivalence wall that compares this kernel's output (curve arrays and their rasterization) across implementation paths
