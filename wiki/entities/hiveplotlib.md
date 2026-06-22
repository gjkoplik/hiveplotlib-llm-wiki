---
title: hiveplotlib
type: entity
created: 2026-04-06
updated: 2026-06-20
sources: [hiveplotlib-python-repo, krzywinski-2012]
tags: [hiveplotlib, python, software, network-visualization, main-hub]
---

# hiveplotlib

> **This is the main hub of research and software development for this wiki.**

hiveplotlib is a Python library for generating and visualizing static [[hive-plot|hive plots]]. Created and maintained by Gary Koplik. It is the most comprehensive open-source implementation of Martin Krzywinski's hive plot method.

## Status

- **Current version:** 0.28.0 (released 2026-06-18), focused on streamlined NetworkX integration (see "NetworkX integration" below; the design decisions are distilled in [[0001-networkx-integration|ADR 0001]]).
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

### NetworkX integration (v0.28, shipped)

v0.28.0 makes a NetworkX graph a first-class input and output, removing the manual extract-and-merge boilerplate that earlier workflows needed. The combined design rationale (including the declined `from_networkx` classmethod and the deferred future-igraph questions) is recorded in [[0001-networkx-integration|ADR 0001]]:

- **Build from a graph:** `HivePlot(graph=...)` and `HivePlotMatrix.from_partition(graph=...)` / `from_variable_sweep(graph=...)` accept a NetworkX graph directly instead of separate `nodes` / `edges`. `graph_directed` / `graph_multigraph` default off `graph.is_directed()` / `graph.is_multigraph()`, so directed and multigraph inputs flow through without restating the type. (`from_tags` is intentionally excluded: tags are an `Edges`-level concept.)
- **Export to a graph:** `HivePlot.to_networkx()` (and the underlying `converters.nodes_edges_to_networkx()`) round-trip a hive plot back to any of the four NetworkX graph classes; multi-tag `Edges` get a `tag` edge attribute.
- **Graph metrics on construction:** `node_graph_metrics` / `edge_graph_metrics` (plus `*_graph_metric_kwargs` / `*_graph_metric_rename`) compute and attach metric columns *before* partitioning, so they are immediately usable as `partition_variable` / `sorting_variables`. `compute_graph_metrics()` does the same for existing `HivePlot` / `HivePlotMatrix` instances. When you build from `nodes` / `edges` and leave `graph_directed` unset, the internal graph's directedness is inferred from the requested metrics (asking for `triangles` builds an undirected graph; `in_degree` builds a directed one), and a contradictory request raises an up-front `ValueError` instead of silently picking a side. A simple-graph build that collapses same-direction duplicate edges now warns by default (`warn_on_parallel_edge_collapse`). See [[graph-features]] for the full graph-type-handling rules.
- **Backend dispatch for graph metrics:** a `graph_metric_backend` parameter on `compute_graph_metrics()`, `HivePlot`, and all five `HivePlotMatrix` surfaces (including the `from_*` classmethods) routes metric computation through NetworkX's backend dispatch system (nx-parallel tested in CI; nx-cugraph known-good but GPU-only). Unknown or uninstalled backend names raise `InvalidGraphMetricBackendError` up front; metrics a backend does not implement fall back to default NetworkX with an INFO log line (the codebase's first use of stdlib logging). A per-metric reserved `"backend"` key in the metric-kwargs dicts overrides the global value, with explicit `None` as the per-metric opt-out; precedence runs per-metric > per-call > stored construction value, mirroring `graph_directed`. This is orthogonal to the future igraph wrapper-library axis (see [[graph-features]] for the three distinct "backend" senses). New gallery notebook: Graph Metric Backends.
- Removed the long-deprecated `hive_plot_n_axes()` (folded into `HivePlot` back in v0.26).

### Key Features
- [[edge-rendering|Edge kwarg hierarchy]] for layered styling
- [[bezier-curves|Bézier curve]] generation with numba acceleration
- [[graph-features|Graph-feature wrappers]] — 35 node + 8 edge NetworkX metrics, indexed by string name, attachable to a `NodeCollection` / `Edges`
- NetworkX converters both ways: `networkx_to_nodes_edges()` in, `nodes_edges_to_networkx()` / `HivePlot.to_networkx()` out
- 50 example Jupyter notebooks (four new v0.28 gallery pages: Computing Graph Metrics, Graph Metric Backends, Exporting to NetworkX, Exporting to JSON)
- 100% test coverage
- Built-in datasets (toy plots, international trade data)
- Agent-first docs: a hand-curated `llms.txt` (llmstxt.org shape) is served at the docs site root on Read the Docs (via `html_extra_path`), ranking the conceptual entry points first, then the API reference, then the gallery, so an agent pointed at the docs finds the load-bearing pages before the gallery breadth. Docs-infrastructure only, no Python API change. Shipped 2026-06-19 (the `llms-full.txt` full-text concatenation was scoped out, with a revival trigger).

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
- [[0001-networkx-integration|ADR 0001 — NetworkX integration]] — Binding design record for the v0.28 NetworkX story
- [[differential-hive-plot]] — Not yet implemented; potential feature
- [[p2cp]] — Polar Parallel Coordinates
- [[karate-club-walkthrough]] — Step-by-step example walkthrough
