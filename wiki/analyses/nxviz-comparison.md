---
title: nxviz vs hiveplotlib — Capability Comparison
type: analysis
created: 2026-06-12
updated: 2026-06-12
sources: [nxviz-repo-web-2026-06-12]
tags: [hive-plot, hiveplotlib, nxviz, network-visualization, comparison, joss, state-of-the-field]
---

# nxviz vs hiveplotlib — Capability Comparison

Evidence for the JOSS paper's **State of the field** section. nxviz is the only living alternative Python library that draws hive plots, so reviewers will expect a precise, fair comparison. All findings accessed 2026-06-12.

> **Fairness note.** nxviz's author (Eric J. Ma) may read this and the paper. Every delta below is stated to nxviz's actual current behavior, not a stale or strawman version. Where nxviz already does something, that is credited; the hiveplotlib advantage is phrased narrowly (e.g. "user-controllable/styleable repeat axes," not "repeat axes at all").

## What nxviz is

- **Repo:** github.com/ericmjl/nxviz. Author **Eric J. Ma**. License **MIT**.
- **Recently revived.** Largely dormant 2018–2025, then a June-2026 burst:
  - **v0.8.0 (2026-06-02):** added a **plotly backend** via a `PlotBackend` protocol.
  - **v0.9.0 (2026-06-08):** added **chord diagrams**.
- **Current API is functional/declarative**, seaborn-inspired: `nv.hive(G, group_by=, sort_by=, node_color_by=, ...)`. Self-described as "convenient (albeit restrictive)... deliberately restricts customization."
- **Old object-oriented API is deprecated** as of nxviz 0.7 (emits a `DeprecationWarning`); the class `draw()` is a **no-op**; slated for removal at 1.0. So the legacy `HivePlot` class is effectively gone, and any comparison must target the functional API.

## Capability deltas vs [[hiveplotlib]]

| Dimension | nxviz (current) | [[hiveplotlib]] |
|-----------|-----------------|-----------------|
| **Axes** | Hard-caps hive plots at **3 groups** (`ValueError` if >3); axes derived 1:1 from a `group_by` column | 2–3 axes via explicit axis construction; **arbitrary node-to-axis assignment** |
| **Repeat axes** | **Does auto-clone** axes (fixed +30° offset) to route intra-group edges, but **not user-controllable/styleable** | Repeat axes are **first-class and styleable** |
| **Edge styling** | Data-driven encodings only (`edge_color_by` / `alpha_by` / `lw_by` + palette); no precedence hierarchy | all / clockwise / counterclockwise / repeat / non_repeat **kwarg precedence system** |
| **Backends** | matplotlib + plotly (**2**, as of v0.8.0) | **5** (matplotlib, bokeh, holoviews, plotly, datashader) |
| **Large networks** | No datashader/rasterization path | **Datashader** rasterization path |
| **Parallel coords** | Cartesian `parallel` only; **no polar** parallel coordinates | **[[p2cp|Polar Parallel Coordinates]]** |
| **Graph metrics** | Consumes precomputed NetworkX attributes; no metric-computation layer | Ships **[[graph-features|graph-feature helpers]]** (35 node + 8 edge NetworkX metrics) |

**Stale-claim flag:** the older "nxviz is matplotlib-only" line is now wrong. As of v0.8.0 (June 2026) nxviz has matplotlib **and** plotly. The paper must say "2 backends," not "matplotlib-only."

## State-of-the-field framing

The honest positioning: nxviz is back under active development and is a genuine alternative for quick, declarative hive plots when 3 groups and data-driven edge encodings suffice. hiveplotlib differentiates on **control and scale**: arbitrary axis assignment, styleable repeat axes, the edge-kwarg precedence system, five backends including a datashader path for large networks, polar parallel coordinates (no nxviz analogue), and a metric-computation layer. The cleanest single differentiator for a non-specialist reader is **P2CP**: nxviz has no polar parallel coordinates at all.

## See Also

- [[hive-plot]] — competing-implementations table links here
- [[hiveplotlib]] — the library this comparison defends
- [[p2cp]] — the clean differentiator
- [[graph-features]] — the metric-computation layer nxviz lacks
- [[hiveplotlib-research-impact]] — companion JOSS-evidence page
- [[perez-2021-hype]] — the closest *published* comparator (HyPE)
