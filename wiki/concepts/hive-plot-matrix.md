---
title: Hive Plot Matrix
type: concept
created: 2026-04-06
updated: 2026-04-06
sources: [krzywinski-2012, hiveplotlib-python-repo]
tags: [hive-plot, hive-plot-matrix, comparative-visualization]
---

# Hive Plot Matrix

A grid of [[hive-plot|hive plots]] enabling systematic comparison across different parameters, partitions, or edge types. Originated as the "Hive Panel" in [[krzywinski-2012]] and implemented as `HivePlotMatrix` in [[hiveplotlib]].

## Origin

Krzywinski's 2012 paper included a 5×5 matrix of hive plots showing a single network visualized with 25 different parameter combinations (betweenness, clustering coefficient, closeness, connectivity, branching ratio). This "Hive Panel" concept was formalized as [[perez-2021-hype|HyPE (Hive Panel Explorer)]] in 2021. [[nollenburg-2023-computing-hive-plots|Nöllenburg & Wallinger (2023)]] later proved that the underlying parameter selection problems are NP-complete, validating the panel exploration approach.

## Implementation in hiveplotlib

The `HivePlotMatrix` class (currently unreleased, in development) supports four construction modes:

| Mode | Method | Description |
|------|--------|-------------|
| Partition | `from_partition()` | One cell per partition pair (upper-triangular grid) |
| Variable sweep | `from_variable_sweep()` | Sweep over sorting/partition variables |
| Tags | `from_tags()` | One cell per edge tag |
| Generic | constructor | Custom grid of HivePlot instances |

**Backends:** matplotlib and datashader only (not yet supported in interactive backends).

## Use Cases

- Compare how different structural parameters reveal different patterns in the same network
- Compare subgroups within a network (e.g., disease subsystems in [[krzywinski-2012]])
- Explore how partitioning choices affect the visualization

## See Also

- [[hive-plot]] — The individual visualization
- [[hiveplotlib]] — Implementation
- [[krzywinski-2012]] — Origin of the concept
- [[perez-2021-hype]] — First interactive implementation (HyPE)
- [[nollenburg-2023-computing-hive-plots]] — Proves parameter selection is NP-complete
- [[differential-hive-plot]] — An alternative comparison approach (diff vs. panel)
