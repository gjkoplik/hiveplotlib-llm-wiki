---
title: hiveplotlib
type: entity
created: 2026-04-06
updated: 2026-04-06
sources: [hiveplotlib-python-repo, krzywinski-2012]
tags: [hiveplotlib, python, software, network-visualization, main-hub]
---

# hiveplotlib

> **This is the main hub of research and software development for this wiki.**

hiveplotlib is a Python library for generating and visualizing static [[hive-plot|hive plots]]. Created and maintained by Gary Koplik. It is the most comprehensive open-source implementation of Martin Krzywinski's hive plot method.

## Status

- **Current version:** 0.27.0 (released 2026-04-10)
- **Python support:** 3.10+
- **Repository:** GitLab (private)
- **Install:** `pip install hiveplotlib`
- **License:** see repo

## Core API

### Data Structures
- **`Node`** / **`NodeCollection`** — Individual nodes and pandas-backed collections with partition logic
- **`Edges`** — Tagged edge container supporting multiple edge types
- **`Axis`** — Polar coordinate axis with start/end, angle, node placements

### Main Classes
- **`BaseHivePlot`** — Low-level construction (manual axes, node placement, edge connection)
- **`HivePlot`** — High-level interface (partition → sort → build → plot)
- **`HivePlotMatrix`** — Comparative grid of hive plots, new in v0.27 (see [[hive-plot-matrix]])
- **`P2CP`** — [[p2cp|Polar Parallel Coordinates]] for tabular data

### Visualization Backends
Six backends: matplotlib (default), bokeh, holoviews-bokeh, holoviews-matplotlib, plotly, datashader.

### Key Features
- [[edge-rendering|Edge kwarg hierarchy]] for layered styling
- [[bezier-curves|Bézier curve]] generation with numba acceleration
- NetworkX converter (`networkx_to_nodes_edges()`)
- 46 example Jupyter notebooks
- 100% test coverage (422 tests)
- Built-in datasets (toy plots, international trade data)

## Ecosystem

- **[[hiveplotlib-javascript]]** — D3.js companion for browser rendering of `.to_json()` exports
- **Prior art:** pyveplot (structural reference), HiveR (R), jhive (Java), D3 hive plots (Bostock)

## Development Priorities

- Additional backends and interactivity for HivePlotMatrix (currently matplotlib + datashader only)
- Exploring GNN evaluation applications ([[gnn-heterogeneity-hive-plots]])

## Research Directions

- [[gnn-heterogeneity-hive-plots]] — Using HivePlotMatrix to diagnose GNN classification heterogeneity. The `networkx_to_nodes_edges()` converter provides the integration pathway from PyTorch Geometric / NetworkX graph data with GNN predictions into hiveplotlib's data structures.

## See Also

- [[hiveplotlib-python|Source summary]] — Detailed technical documentation of the repo
- [[hiveplotlib-javascript]] — JavaScript companion
- [[hive-plot]] — The visualization method
- [[Martin Krzywinski]] — Invented hive plots
- [[hive-plot-matrix]] — Comparative visualization
- [[differential-hive-plot]] — Not yet implemented — potential feature
- [[p2cp]] — Polar Parallel Coordinates
- [[karate-club-walkthrough]] — Step-by-step example walkthrough
