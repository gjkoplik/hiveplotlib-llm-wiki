---
title: Differential Hive Plot
type: concept
created: 2026-04-06
updated: 2026-04-06
sources: [krzywinski-2017-differential]
tags: [hive-plot, differential, network-comparison]
---

# Differential Hive Plot

A **differential hive plot (DHP)** is the visual difference between two [[hive-plot|hive plots]]. It shows nodes and edges that are added, removed, or preserved when comparing two networks (or the same network at two time points).

## How It Works

Given two hive plots with the same axis assignment rules:
- Compute the intersection and difference of nodes and edges
- Visualize using positional similarity in the input hive plots
- Color-code by status: preserved, added, removed

## Use Cases

- **Temporal analysis:** How does a network evolve over time?
- **Condition comparison:** How does a gene regulatory network differ between healthy and diseased states?
- **Version comparison:** How does a software dependency graph change between releases?

## Implementation Status

Not currently implemented in [[hiveplotlib]]. Could be built as a comparison mode computing the diff between two `HivePlot` instances.

## See Also

- [[krzywinski-2017-differential]] — Source paper
- [[hive-plot]] — The base visualization
- [[hive-plot-matrix]] — Another approach to comparison (multiple views vs. diff view)
