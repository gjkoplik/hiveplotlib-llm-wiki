---
title: hiveplotlib
type: entity
created: 2026-04-06
updated: 2026-05-31
sources: [hiveplotlib-python-repo, krzywinski-2012]
tags: [hiveplotlib, python, software, network-visualization, main-hub]
---

# hiveplotlib

> **This is the main hub of research and software development for this wiki.**

hiveplotlib is a Python library for generating and visualizing static [[hive-plot|hive plots]]. Created and maintained by Gary Koplik. It is the most comprehensive open-source implementation of Martin Krzywinski's hive plot method.

## Status

- **Current version:** 0.27.0 (released 2026-04-10). v0.28.0 in development (`0.28.0a0`), focused on streamlined NetworkX integration (see "NetworkX integration" below).
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

### NetworkX integration (v0.28 in progress)

v0.28.0 makes a NetworkX graph a first-class input and output, removing the manual extract-and-merge boilerplate that earlier workflows needed:

- **Build from a graph:** `HivePlot(graph=...)` and `HivePlotMatrix.from_partition(graph=...)` / `from_variable_sweep(graph=...)` accept a NetworkX graph directly instead of separate `nodes` / `edges`. `graph_directed` / `graph_multigraph` default off `graph.is_directed()` / `graph.is_multigraph()`, so directed and multigraph inputs flow through without restating the type. (`from_tags` is intentionally excluded: tags are an `Edges`-level concept.)
- **Export to a graph:** `HivePlot.to_networkx()` (and the underlying `converters.nodes_edges_to_networkx()`) round-trip a hive plot back to any of the four NetworkX graph classes; multi-tag `Edges` get a `tag` edge attribute.
- **Graph metrics on construction:** `node_graph_metrics` / `edge_graph_metrics` (plus `*_graph_metric_kwargs` / `*_graph_metric_rename`) compute and attach metric columns *before* partitioning, so they are immediately usable as `partition_variable` / `sorting_variables`. `compute_graph_metrics()` does the same for existing `HivePlot` / `HivePlotMatrix` instances. See [[graph-features]].
- Removed the long-deprecated `hive_plot_n_axes()` (folded into `HivePlot` back in v0.26).

### Key Features
- [[edge-rendering|Edge kwarg hierarchy]] for layered styling
- [[bezier-curves|Bézier curve]] generation with numba acceleration
- [[graph-features|Graph-feature wrappers]] — 35 node + 8 edge NetworkX metrics, indexed by string name, attachable to a `NodeCollection` / `Edges`
- NetworkX converters both ways: `networkx_to_nodes_edges()` in, `nodes_edges_to_networkx()` / `HivePlot.to_networkx()` out
- 49 example Jupyter notebooks (three new v0.28 gallery pages: Computing Graph Metrics, Exporting to NetworkX, Exporting to JSON)
- 100% test coverage
- Built-in datasets (toy plots, international trade data)

## Ecosystem

- **[[hiveplotlib-javascript]]** — D3.js companion for browser rendering of `.to_json()` exports
- **Prior art:** pyveplot (structural reference), HiveR (R), jhive (Java), D3 hive plots (Bostock)

## Development Priorities

- Additional backends and interactivity for HivePlotMatrix (currently matplotlib + datashader only). The new public `HivePlotMatrix.backend` read-only property makes the current backend inspectable without reaching for the private attribute.
- Future graph-feature backends. The v0.28 metric wrappers live under a `graph_features/networkx/` subpackage; the roadmap calls for a sibling `igraph` backend (notably for community detection) to slot in alongside it. See [[graph-features]].
- Exploring GNN evaluation applications ([[gnn-heterogeneity-hive-plots]])

## Research Directions

- [[gnn-heterogeneity-hive-plots]] — Using HivePlotMatrix to diagnose GNN classification heterogeneity. The integration pathway is now even more direct: `HivePlot(graph=...)` / `HivePlotMatrix.from_*(graph=...)` ingest a NetworkX graph straight from PyTorch Geometric, and `node_graph_metrics` / `edge_graph_metrics` compute the structural sweep variables (degree, centrality, community labels, etc.) in one step, replacing the old manual `pd.DataFrame(G.degree, ...) + nodes.data.merge(...)` pattern. See [[graph-features]] for the available structural sweep variables.

## See Also

- [[hiveplotlib-python|Source summary]] — Detailed technical documentation of the repo
- [[hiveplotlib-javascript]] — JavaScript companion
- [[hive-plot]] — The visualization method
- [[Martin Krzywinski]] — Invented hive plots
- [[hive-plot-matrix]] — Comparative visualization
- [[graph-features]] — NetworkX node/edge metric wrappers (new in v0.28)
- [[differential-hive-plot]] — Not yet implemented; potential feature
- [[p2cp]] — Polar Parallel Coordinates
- [[karate-club-walkthrough]] — Step-by-step example walkthrough
