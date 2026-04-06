---
title: "Krzywinski et al. 2017 — Differential Hive Plots"
type: source
created: 2026-04-06
updated: 2026-04-06
sources: [krzywinski-2017-differential]
tags: [hive-plot, differential, network-comparison, art-science]
---

# Krzywinski et al. 2017 — Differential Hive Plots

**Citation:** Krzywinski, M., Nip, K.M., Birol, I., Marra, M.A., 2017. Differential Hive Plots: Seeing Networks Change. *Leonardo*, 50(5). MIT Press.

## Summary

Introduces **differential hive plots (DHPs)** — the visual difference between two [[hive-plot|hive plots]]. Shows nodes and edges in the intersection or difference of two networks based on positional similarity.

## Key Concept

A DHP enables **temporal or comparative analysis** of network evolution:
- How does a gene regulatory network change under different conditions?
- How does a software dependency graph change between versions?
- What edges/nodes are gained, lost, or preserved?

## Notable

Published in *Leonardo* (MIT Press), an art/science journal — highlighting the crossover appeal of hive plots as both analytical and aesthetic objects. [[Martin Krzywinski]] has consistently straddled the boundary between data visualization as science and as art (see also: Circos).

## Implications for [[hiveplotlib]]

Differential hive plots could be implemented as a comparison mode in [[hiveplotlib]] — computing the diff between two `HivePlot` instances and visualizing additions/removals/changes. This is not currently implemented.

## See Also

- [[hive-plot]] — The base visualization
- [[Martin Krzywinski]] — Author
- [[krzywinski-2012]] — Original paper
