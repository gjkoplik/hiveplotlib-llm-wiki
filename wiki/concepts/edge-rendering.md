---
title: Edge Rendering
type: concept
created: 2026-04-06
updated: 2026-04-06
sources: [krzywinski-2012, hiveplotlib-python-repo, hiveplotlib-javascript-repo]
tags: [hive-plot, edge-rendering, bezier-curves]
---

# Edge Rendering

How connections between nodes are drawn in [[hive-plot|hive plots]]. Edges appear as [[bezier-curves|Bézier curves]] connecting nodes across different axes.

## Visual Encoding

Edges can encode information through:
- **Color** — Edge type, weight, or categorical attribute
- **Thickness** (linewidth) — Edge weight or importance
- **Opacity** (alpha) — Density or confidence
- **Linestyle** — Dashed, dotted, solid for categorical distinctions
- **Direction** — Clockwise vs. counterclockwise orientation indicates flow in directed networks

## Edge Kwarg Hierarchy (hiveplotlib)

In [[hiveplotlib]], edge styling follows a precedence hierarchy (later overrides earlier):

1. `all_edge_kwargs` — Applied to every edge
2. `repeat_edge_kwargs` — Edges between an axis and its repeat
3. `non_repeat_edge_kwargs` — Edges between distinct axes
4. `clockwise_edge_kwargs` — Edges flowing clockwise
5. `counterclockwise_edge_kwargs` — Edges flowing counterclockwise

This enables patterns like: "all edges gray, but clockwise edges blue and counterclockwise edges red."

## Edge Tags

[[hiveplotlib]] supports **multiple edge types** via tags. Each tag represents a different category of relationship (e.g., "activating" vs. "repressing" in gene regulatory networks). Tags are stored as separate DataFrames in the `Edges` class.

## Rendering Pipeline

In [[hiveplotlib]]:
1. `add_edge_ids()` — Extract edge pairs between axes
2. `add_edge_curves_between_axes()` — Generate Bézier control points
3. `construct_curves()` — Discretize curves (with optional numba acceleration)
4. Backend renders SVG paths / matplotlib paths / etc.

In [[hiveplotlib-javascript]]:
- Pre-computed curves arrive in JSON
- `plotEdges()` renders them as SVG paths with D3

## See Also

- [[bezier-curves]] — The curve mathematics
- [[hive-plot]] — The visualization method
- [[node-assignment]] — Nodes must be placed before edges connect them
