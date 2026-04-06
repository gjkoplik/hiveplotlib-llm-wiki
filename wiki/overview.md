---
title: Overview
type: overview
created: 2026-04-06
updated: 2026-04-06
sources: [krzywinski-2012, hiveplotlib-python-repo, hiveplotlib-javascript-repo, perez-2021-hype, bostock-2012-d3-hive-plots, nollenburg-2023, krzywinski-2017-differential]
tags: [hive-plot, hiveplotlib, network-visualization]
---

# Hive Plot Research Wiki — Overview

This wiki tracks research, theory, and practice around **[[hive-plot|hive plots]]** and **[[hiveplotlib]]**, the Python library at the center of this work.

## The Big Picture

Hive plots are a network visualization method introduced by [[Martin Krzywinski]] in [[krzywinski-2012|2012]]. The core insight: traditional [[force-directed-layout|force-directed layouts]] produce non-reproducible, non-quantitative "hairballs." Hive plots fix this by assigning nodes to radial axes based on meaningful structural properties ([[node-assignment]]) and drawing edges as [[bezier-curves|Bézier curves]] ([[edge-rendering]]).

The method has been adopted across multiple domains — most strongly in [[applications-bioinformatics|bioinformatics]] (its origin), but also innovatively in [[applications-cybersecurity|cybersecurity]] (where hive plot images serve as ML features for DDoS detection) and [[applications-software-engineering|software engineering]] (code dependency visualization).

## The Theory

[[nollenburg-2023-computing-hive-plots|Nöllenburg & Wallinger (2023)]] formalized hive plot construction as a combinatorial optimization problem and proved all three subproblems (vertex partitioning, cyclic axis ordering, vertex ordering) are **NP-complete**. This validates the exploratory approach: use domain knowledge for initial parameters, then explore alternatives via [[hive-plot-matrix|HivePlotMatrix]] panels. [[krzywinski-2017-differential|Differential hive plots]] (2017) extend the method to network comparison via visual diffs.

## The Software

**[[hiveplotlib]]** is the main hub of research and development. It is a comprehensive Python library (by the wiki maintainer) supporting six visualization backends, with 46 example notebooks, 100% test coverage, and features including:
- High-level `HivePlot` API and low-level `BaseHivePlot` for full control
- [[hive-plot-matrix|HivePlotMatrix]] for comparative visualization (in development)
- [[p2cp|Polar Parallel Coordinates]] for tabular data
- Numba-accelerated [[bezier-curves|Bézier curve]] generation

**[[hiveplotlib-javascript]]** is the browser companion — it renders hiveplotlib's JSON exports as interactive SVGs using D3.js. [[Mike Bostock]]'s earlier [[bostock-2012-d3-hive-plots|D3 implementation (2012)]] was influential but architecturally different (monolithic JS vs. hiveplotlib-javascript's pure rendering layer).

## Current State of Knowledge

| Category | Count |
|----------|-------|
| Sources ingested | 7 |
| Entity pages | 4 |
| Concept pages | 11 |
| Analysis pages | 1 |
| **Total wiki pages** | **23** (+ index, log, overview) |

## Open Questions and Next Steps

- **Neuroscience gap:** Brain connectome networks are a natural fit for hive plots but no published work exists — this is an unexplored opportunity.
- **Social network analysis:** Surprisingly underexplored given the method's strengths for community detection.
- **Differential hive plots in hiveplotlib:** Not yet implemented — could be a valuable addition for version comparison or temporal analysis.
- **Interactive HivePlotMatrix:** Currently matplotlib/datashader only. Bokeh or Plotly backends would enable browser-based panel exploration, achieving what [[perez-2021-hype|HyPE]] did but within the modern hiveplotlib ecosystem.
- **Automated parameter selection:** Given NP-completeness, could heuristic/approximate optimization (genetic algorithms, simulated annealing) help suggest good axis assignments?

## See Also

- [[index]] — Full catalog of wiki pages
