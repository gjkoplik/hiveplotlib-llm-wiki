---
title: "Krzywinski et al. 2012 — Hive Plots: Rational Approach to Visualizing Networks"
type: source
created: 2026-04-06
updated: 2026-04-06
sources: [krzywinski-2012]
tags: [hive-plot, network-visualization, foundational-paper, bioinformatics]
---

# Krzywinski et al. 2012 — Hive Plots: Rational Approach to Visualizing Networks

**Citation:** Krzywinski, M., Birol, I., Jones, S.J.M. and Marra, M.A., 2012. Hive plots — rational approach to visualizing networks. *Briefings in Bioinformatics*, 13(5), pp.627–644. DOI: 10.1093/bib/bbr069. PMID: 22155641.

**Authors:** [[Martin Krzywinski]] et al., Canada's Michael Smith Genome Sciences Center / BC Cancer Research Center, Vancouver.

## Summary

The foundational paper introducing [[hive-plot|hive plots]] as an alternative to [[force-directed-layout|force-directed layouts]] for network visualization. The core argument: existing layout algorithms lack **reproducibility** and **perceptual uniformity** because they don't use a meaningful node coordinate system. Hive plots fix this by mapping nodes to radially distributed linear axes based on structural properties.

## Key Concepts

### The Problem with Force-Directed Layouts
- Different runs produce different layouts for the same network
- Small network changes cause disproportionately large layout changes
- Dense networks become indecipherable "hairballs"
- Visual positions lack quantitative meaning

### The Hive Plot Solution
Three essential components:
1. **[[node-assignment|Axis assignment rules]]** — Boolean tests determine which axis receives each node (e.g., source/sink/manager classification)
2. **Node coordinate system** — Positions along axes derived from structural parameters (degree, clustering coefficient, betweenness, etc.)
3. **[[edge-rendering|Edge rendering]]** — Connections drawn as [[bezier-curves|Bézier curves]] between nodes on different axes

### Structural Parameters for Node Placement
Node-level: degree, flow, betweenness, closeness, eccentricity, PageRank, eigenvector centrality, clustering coefficient, topological overlap, cut vertex status.
Network-level: modules, assortativity, centralization, density, diameter, radius.

### Properties
Five requirements met: Generality, Flexibility, Transparency, Competence, Speed. Plus two distinguishing properties: **Reproducibility** and **Perceptual Uniformity**.

## Case Studies

### RegulonDB (E. coli Gene Regulatory Network)
- 1,584 genes: 44 regulators, 135 managers, 1,415 workhorses
- Hive plot reveals: managers exceed regulators in connectivity; NSRR is 93% repressive to workhorses; FNR and CRP operate independently

### Cancer Gene-Disease Network
- 258 genes, 3,057 edges from human disease network
- TP53 emerges as central hub; bridge genes (AR, IRF1, VHL) connecting cancer to other diseases clearly revealed

### The Hive Panel (precursor to [[hive-plot-matrix]])
- A 5×5 matrix of hive plots using different parameter combinations — 25 perspectives on one network

## Impact
- ~257 citations (Semantic Scholar, 2026)
- Spawned implementations: HiveR (R), D3.js (Bostock), hiveplot (Python/Eric Ma), [[hiveplotlib]] (Python), jhive (Java)
- Companion website: hiveplot.com
- Krzywinski is also the creator of Circos

## Key Quote
> "If the layout shows a pattern, you can be sure it is due to structure in the underlying data and not on the layout algorithm's interpretation of how the data should be shown."

## See Also

- [[hive-plot]] — Core concept page
- [[hiveplotlib]] — Python implementation
- [[Martin Krzywinski]] — Author
- [[force-directed-layout]] — The approach hive plots replace
- [[node-assignment]] — How nodes map to axes
- [[edge-rendering]] — How edges are drawn
