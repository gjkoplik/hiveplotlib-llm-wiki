---
title: "hiveplotlib — Python Library Source"
type: source
created: 2026-04-06
updated: 2026-04-06
sources: [hiveplotlib-python-repo]
tags: [hiveplotlib, python, software, network-visualization]
---

# hiveplotlib — Python Library Source

**Repository:** `/home/garyk/repos/hiveplotlib` (GitLab-hosted)
**Version:** 0.27.0a1 (unreleased)
**Author:** Gary Koplik (wiki maintainer)

## Summary

[[hiveplotlib]] is the primary Python library for generating and visualizing static [[hive-plot|hive plots]]. It is the **main hub of research and software development** for this wiki. The library supports six visualization backends and provides a comprehensive API for constructing, styling, and rendering hive plots from network data.

## Architecture

### Core Classes
- **`BaseHivePlot`** — Foundation class. Manual axis creation, node placement, edge connection. Full control over construction pipeline.
- **`HivePlot`** — High-level interface. Set partition variable → set sorting variables → build. Handles repeat axes, rotation, axis ordering.
- **`HivePlotMatrix`** — Comparative visualization grid. Construct via `from_partition()`, `from_variable_sweep()`, `from_tags()`, or generic constructor. See [[hive-plot-matrix]].
- **`NodeCollection`** — Pandas-backed node container. Holds node data, partition logic, visualization kwargs.
- **`Edges`** — Tagged edge container. Multiple edge types via tags, per-tag DataFrames, visualization kwargs with hierarchy.
- **`Axis`** — Polar coordinate axis. Start/end distances, angle, node placements, sorting variable.
- **`P2CP`** — [[p2cp|Polar Parallel Coordinates]]. Visualizes tabular (not network) data with the same radial axis approach.

### Visualization Backends
| Backend | Type | Best For |
|---------|------|----------|
| matplotlib | Static | Default, publication figures |
| bokeh | Interactive | Web-based exploration |
| holoviews-bokeh | Interactive | HoloViews composability |
| holoviews-matplotlib | Static | HoloViews composability |
| plotly | Interactive | Web-based, tooltips |
| datashader | Static | Large networks (30K+ edges) |

### Edge Kwarg Hierarchy
Styling precedence (later overrides earlier):
1. `all_edge_kwargs`
2. `repeat_edge_kwargs`
3. `non_repeat_edge_kwargs`
4. `clockwise_edge_kwargs`
5. `counterclockwise_edge_kwargs`

### Data Flow
1. Create `NodeCollection` from DataFrame (or `Node` list)
2. Create `Edges` from edge pairs ± metadata
3. Create `HivePlot`
4. Set partition variable → assign nodes to axes
5. Set sorting variables → position nodes along axes
6. Build axes → connect edges → plot

### Key Utilities
- `polar2cartesian()` / `cartesian2polar()` — Coordinate conversion
- `bezier()` / `bezier_all()` / `bezier_xy_with_nans()` — [[bezier-curves|Bézier curve]] generation with optional numba acceleration
- `networkx_to_nodes_edges()` — NetworkX graph converter

## Scale

| Metric | Count |
|--------|-------|
| Core source modules | 10 |
| Source lines | ~7,700 |
| Visualization backends | 6 |
| Test files | 22 |
| Test functions | 422 |
| Example notebooks | 46 |
| Coverage requirement | 100% |

## Built-in Datasets
- Toy hive plots (minimal, multi-tag, full kwargs)
- Toy P2CPs
- International trade data (Harvard Growth Lab, 2019, 8,112 HS codes)

## Key Design Decisions
- **Minimal base deps:** matplotlib, numpy, pandas only. Optional backends as extras.
- **100% test coverage** strictly enforced; all warnings treated as errors.
- **Numba acceleration** for Bézier curves — parallel edge construction across CPU cores.
- **float32 arrays** for memory efficiency.
- **Type hints throughout** (Union[X, Y] syntax for Python 3.10+ compat).

## See Also

- [[hiveplotlib]] — Entity page (main hub)
- [[hiveplotlib-javascript]] — D3.js companion library
- [[hive-plot]] — Core concept
- [[hive-plot-matrix]] — Comparative visualization
- [[p2cp]] — Polar Parallel Coordinates
