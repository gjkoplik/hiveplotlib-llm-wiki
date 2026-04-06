---
title: "Perez et al. 2021 — Hive Panel Explorer (HyPE)"
type: source
created: 2026-04-06
updated: 2026-04-06
sources: [perez-2021-hype]
tags: [hive-plot-matrix, interactive, bioinformatics, tool]
---

# Perez et al. 2021 — Hive Panel Explorer (HyPE)

**Citation:** Perez, S.E.I., Hahn, A.S., Krzywinski, M., Hallam, S.J., 2021. Hive Panel Explorer: an interactive network visualization tool. *Bioinformatics*, 37(3), pp.436–437. DOI: 10.1093/bioinformatics/btaa683. PMCID: PMC8058766.

**Authors:** Perez, Hahn, [[Martin Krzywinski]], Hallam — BC Genome Sciences Centre / UBC.

## Summary

HyPE is the first published software tool formalizing [[Martin Krzywinski]]'s "Hive Panel" concept from [[krzywinski-2012]] into an interactive application. It generates a matrix of interactive [[hive-plot|hive plots]] in the browser, allowing users to explore multiple axis assignment and node ordering combinations simultaneously.

## Key Contribution

The central problem HyPE addresses: hive plots require choosing axis assignment rules and sorting metrics **a priori**. This is a design bottleneck — which parameters best reveal the structure? HyPE sidesteps this by showing many parameter combinations at once in an interactive panel, letting the user discover which views are informative.

## Technical Architecture

- **Backend:** Python 2.7 + NetworkX
- **Frontend:** D3.js (using Bostock's hive plot plugin) + JavaScript + CSS
- **Output:** Self-contained HTML page with interactive SVG graphics
- **Input:** Node file (CSV/TSV) + edge file (CSV/TSV), or GraphML
- **Repository:** github.com/hallamlab/HivePanelExplorer (GNU license, not actively maintained)

## Case Study: Forest Soil Microbiome

- 26 soil samples near Williams Lake, BC — natural vs. harvested plots
- 1,880 OTU nodes, 13,605 edges (6,967 positive, 6,638 negative correlations)
- Revealed: microbial community modules partition by harvesting treatment and soil depth
- Module A (7 cm depth): 55% indicator OTUs for harvested surface horizons
- Module B (13 cm depth): 16% indicators for natural surface horizons

## Relationship to [[hive-plot-matrix|HivePlotMatrix]]

HyPE was the first formalization; [[hiveplotlib]]'s `HivePlotMatrix` is a more modern, Pythonic re-implementation:

| Aspect | HyPE | HivePlotMatrix |
|--------|------|----------------|
| Language | Python 2.7 + D3.js | Python 3.10+ |
| Interactivity | Yes (browser) | Static (matplotlib/datashader) |
| Construction modes | User-defined panels | 4 modes (partition, sweep, tags, generic) |
| Maintenance | Abandoned | Active development |
| Integration | Standalone tool | Part of [[hiveplotlib]] ecosystem |

## See Also

- [[hive-plot-matrix]] — The concept
- [[krzywinski-2012]] — Origin of the Hive Panel idea
- [[hiveplotlib]] — Modern implementation
- [[Martin Krzywinski]] — Co-author
