---
title: Hive Plot
type: concept
created: 2026-04-06
updated: 2026-06-12
sources: [krzywinski-2012, hiveplotlib-python-repo]
tags: [hive-plot, network-visualization, core-concept]
---

# Hive Plot

A network visualization method that maps nodes to **radially distributed linear axes** based on meaningful node properties, then draws edges as [[bezier-curves|Bézier curves]] between them. Introduced by [[Martin Krzywinski]] in [[krzywinski-2012|2012]] as a "rational approach to visualizing networks."

## Core Idea

Unlike [[force-directed-layout|force-directed layouts]] where node positions are determined by aesthetic optimization algorithms, hive plots give every node a **meaningful, reproducible position**:

1. **[[node-assignment|Axis assignment]]** — Nodes are classified onto axes by a rule (e.g., source/sink/manager for directed networks, or clustering coefficient ranges for undirected)
2. **Axis positioning** — Each node's position along its axis reflects a structural property (degree, betweenness, etc.)
3. **[[edge-rendering|Edge drawing]]** — Connections rendered as curves between node positions

The name comes from the visual resemblance to a beehive structure.

## Why Three Axes?

Three axes is the typical configuration. With three radially distributed axes, edges between any pair of axes can be drawn without crossing the third axis. More axes require special handling (axis duplication, reordering).

## Repeat Axes

When nodes on the **same axis** need to connect to each other, that axis is duplicated at a nearby angle. In [[hiveplotlib]], this is handled by the `repeat_axes` parameter on `HivePlot`.

## Key Properties

| Property | Description |
|----------|-------------|
| **Reproducibility** | Same data always produces same layout |
| **Perceptual uniformity** | Small data changes → small visual changes |
| **Transparency** | Layout rules are explicit and user-defined |
| **Quantitative** | Node positions have numeric meaning |
| **Comparable** | Different networks can be meaningfully compared |

## Conceptual Relatives

- **Parallel coordinates plots** — Similar axis-based approach, but linear rather than radial
- **[[p2cp|Polar Parallel Coordinates]]** — Radial variant for tabular (non-network) data, implemented in [[hiveplotlib]]
- **Circos** — Another radial visualization by [[Martin Krzywinski]], but for different data types

## Implementations

| Library | Language | Notes | Maintenance status |
|---------|----------|-------|--------------------|
| **[[hiveplotlib]]** | Python | Most comprehensive, 6 backends | Active |
| **[[hiveplotlib-javascript]]** | JavaScript/D3 | Browser rendering of hiveplotlib exports | Active |
| nxviz | Python | Functional API does hive plots (3-group cap) among other layouts; see [[nxviz-comparison]] | Revived June 2026 (dormant 2018–2025) |
| HiveR | R | 2D/3D hive plots | Dormant / unmaintained |
| jhive | Java | Reference implementation by Krzywinski | Dormant / unmaintained |
| D3 hive plot | JavaScript | By [[Mike Bostock]] | Dormant / unmaintained |
| pyveplot | Python | Earlier implementation | Dormant / unmaintained |

## Application Domains

- [[applications-bioinformatics]] — Strongest adoption (gene networks, microbiome, protein interactions)
- [[applications-cybersecurity]] — Most innovative (ML featurization of hive plot images)
- [[applications-software-engineering]] — Code dependency visualization
- [[gnn-heterogeneity-hive-plots|GNN evaluation]] — Proposed: using [[hive-plot-matrix|HivePlotMatrix]] to diagnose [[graph-neural-networks|GNN]] classification heterogeneity across graph structure

## See Also

- [[node-assignment]] — How nodes map to axes
- [[edge-rendering]] — How edges are drawn
- [[hive-plot-matrix]] — Comparative hive plot grids
- [[differential-hive-plot]] — Comparing two networks via visual diff
- [[force-directed-layout]] — What hive plots replace
- [[hiveplotlib]] — Primary implementation
- [[nxviz-comparison]] — Capability comparison with nxviz, the only living alternative
- [[nollenburg-2023-computing-hive-plots]] — Formal algorithmic foundation
