---
title: Hive Plot Matrix
type: concept
created: 2026-04-06
updated: 2026-05-31
sources: [krzywinski-2012, hiveplotlib-python-repo]
tags: [hive-plot, hive-plot-matrix, comparative-visualization]
---

# Hive Plot Matrix

A grid of [[hive-plot|hive plots]] enabling systematic comparison across different parameters, partitions, or edge types. Originated as the "Hive Panel" in [[krzywinski-2012]] and implemented as `HivePlotMatrix` in [[hiveplotlib]].

## Origin

Krzywinski's 2012 paper included a 5×5 matrix of hive plots showing a single network visualized with 25 different parameter combinations (betweenness, clustering coefficient, closeness, connectivity, branching ratio). This "Hive Panel" concept was formalized as [[perez-2021-hype|HyPE (Hive Panel Explorer)]] in 2021. [[nollenburg-2023-computing-hive-plots|Nöllenburg & Wallinger (2023)]] later proved that the underlying parameter selection problems are NP-complete, validating the panel exploration approach.

## Implementation in hiveplotlib

The `HivePlotMatrix` class (released in v0.27.0) supports four construction modes:

| Mode | Method | Description |
|------|--------|-------------|
| Partition | `from_partition()` | One cell per partition pair (upper-triangular grid) |
| Variable sweep | `from_variable_sweep()` | Sweep over sorting/partition variables |
| Tags | `from_tags()` | One cell per edge tag |
| Generic | constructor | Custom grid of HivePlot instances |

**Backends:** matplotlib and datashader only (not yet supported in interactive backends). As of v0.28, the current backend is inspectable via a public read-only `HivePlotMatrix.backend` property (set it with `set_viz_backend()`).

**NetworkX and graph metrics (v0.28):** `from_partition()` and `from_variable_sweep()` accept a `graph=` NetworkX input directly (mirroring `HivePlot`), and every construction path accepts `node_graph_metrics` / `edge_graph_metrics` so structural variables can be computed and swept in one call (see [[graph-features]]). `from_tags()` keeps no `graph=` shortcut, since edge tags are an `Edges`-level concept. A fix in the same release lets integer-valued partition variables (e.g. `louvain_communities` labels) build cleanly through the off-diagonal collapse path, and `row_labels` / `col_labels` now preserve the partition values' original dtype instead of coercing to strings.

## Use Cases

- Compare how different structural parameters reveal different patterns in the same network
- Compare subgroups within a network (e.g., disease subsystems in [[krzywinski-2012]])
- Explore how partitioning choices affect the visualization

## Research Directions

### GNN Evaluation Diagnostics

A [[hive-plot-matrix|HivePlotMatrix]] is a natural tool for diagnosing [[graph-neural-networks|GNN]] classification performance across graph structure. By sweeping partitioning variables (degree bins, community membership, local homophily) across the matrix and color-coding edges by correct vs. misclassified predictions, a HivePlotMatrix can expose [[structural-heterogeneity|structural performance heterogeneity]] that aggregate metrics mask. The `from_variable_sweep()` construction mode is the natural fit; each row or column represents a different structural decomposition of the same predictions. The v0.28 `graph=` and `node_graph_metrics` parameters make this nearly turnkey: ingest a NetworkX graph carrying predictions and compute the structural sweep variables in the same constructor call (see [[graph-features]]). See [[gnn-heterogeneity-hive-plots]] for the full research proposal.

## See Also

- [[hive-plot]] — The individual visualization
- [[hiveplotlib]] — Implementation
- [[krzywinski-2012]] — Origin of the concept
- [[perez-2021-hype]] — First interactive implementation (HyPE)
- [[nollenburg-2023-computing-hive-plots]] — Proves parameter selection is NP-complete
- [[graph-features]] — Structural metrics the matrix sweeps over
- [[differential-hive-plot]] — An alternative comparison approach (diff vs. panel)
