---
title: "Bostock 2012 — D3 Hive Plots"
type: source
created: 2026-04-06
updated: 2026-04-06
sources: [bostock-2012-d3-hive-plots]
tags: [hive-plot, d3, javascript, software-engineering, visualization]
---

# Bostock 2012 — D3 Hive Plots

**Author:** Mike Bostock (creator of D3.js)
**Published:** March 18, 2012
**URL:** bost.ocks.org/mike/hive/

## Summary

An influential writeup and D3.js implementation of [[hive-plot|hive plots]], applying them to **software dependency visualization**. Bostock visualized the Flare visualization toolkit's class dependency graph, demonstrating hive plots on a software engineering use case.

## Key Arguments

Bostock's critique of force-directed layouts aligns with [[krzywinski-2012]] but is stated more concisely:

> "Many methods of graph drawing, such as force layouts, do not assign intrinsically-meaningful positions to nodes: the position is only approximate, in the hope that related nodes appear nearby. While intuitive, these methods arguably make poor use of the most effective visual channel (that is, *position*), and in the worst case produce an indecipherable hairball."

On hive plots:
> "Hive plots define a linear layout for nodes, grouping nodes by type and arranging them along radial axes based on some property of data. The explicit position encoding has the potential to better reveal the network structure while communicating additional information."

## Implementation Details

- **D3 version:** v2 (pre-D3 v7)
- **3 axes at 120° intervals** with a minor angular offset to split the "both" category
- **4 logical axes from 3 visual:**
  - Source (top) — classes with only outgoing dependencies
  - Target (bottom-left) — classes with only incoming dependencies
  - Target-source and source-target (bottom-right, duplicated) — classes with both
- **Custom `link()` shape generator** — cubic Bézier curves with control points at 1/3 and 2/3 angular distance
- **Nodes sorted by package** (not degree), grouping related classes to reveal macro structure
- **Interactivity:** mouseover highlights connected links

## Comparison with [[hiveplotlib-javascript]]

| Aspect | Bostock 2012 | hiveplotlib-javascript |
|--------|-------------|----------------------|
| Computation | All in JS (parses graph, assigns axes, computes positions) | JS only renders; Python does all computation |
| Data format | Raw graph JSON | Pre-computed JSON with coordinates + curves |
| Curve generation | Custom JS Bézier generator | Pre-computed in Python |
| Scope | Bespoke single visualization | General-purpose renderer |
| Styling | Hardcoded CSS | Maps matplotlib kwargs to SVG |

The key architectural difference: Bostock's is **monolithic** (JS does everything), while [[hiveplotlib-javascript]] is a **pure rendering layer** trusting [[hiveplotlib]] for all analytical work.

## Companion Examples

- **Hive Plot (Links)** — gist.github.com/mbostock/2066415
- **Hive Plot (Areas)** — gist.github.com/mbostock/2066421 — aggregate relationships using filled areas

## Insights Worth Noting

- **Axis duplication trick:** Nodes with both incoming and outgoing dependencies "are duplicated to reveal dependencies within this category." Without this, intra-group edges would be invisible. This is the same concept as [[hiveplotlib]]'s `repeat_axes`.
- **Flexibility emphasis:** "You can use any number of methods to group and position nodes along each axis, customizing the layout to suit your needs."
- Bostock also links to **matrix diagrams** and **hierarchical edge bundling** as complementary approaches.

## See Also

- [[hive-plot]] — The visualization method
- [[hiveplotlib-javascript]] — Modern D3 renderer for hiveplotlib
- [[krzywinski-2012]] — The paper Bostock references
- [[applications-software-engineering]] — Hive plots for code dependencies
