---
title: Overview
type: overview
created: 2026-04-06
updated: 2026-05-31
sources: [krzywinski-2012, hiveplotlib-python-repo, hiveplotlib-javascript-repo, perez-2021-hype, bostock-2012-d3-hive-plots, nollenburg-2023, krzywinski-2017-differential, ma-2021-subgroup-fairness, kipf-2017-gcn, subramonian-2024-degree-bias, gnnfairviz-2025]
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

**[[hiveplotlib]]** is the main hub of research and development. It is a comprehensive Python library (by the wiki maintainer) supporting six visualization backends, with 49 example notebooks, 100% test coverage, and features including:
- High-level `HivePlot` API and low-level `BaseHivePlot` for full control
- [[hive-plot-matrix|HivePlotMatrix]] for comparative visualization (released in v0.27)
- [[p2cp|Polar Parallel Coordinates]] for tabular data
- Numba-accelerated [[bezier-curves|Bézier curve]] generation
- [[graph-features|Graph-feature wrappers]] (in-progress v0.28) — 35 node + 8 edge NetworkX metrics computable straight onto axes, with `HivePlot(graph=...)` / `HivePlotMatrix.from_*(graph=...)` ingest and `HivePlot.to_networkx()` export making NetworkX a first-class input *and* output

**[[hiveplotlib-javascript]]** is the browser companion — it renders hiveplotlib's JSON exports as interactive SVGs using D3.js. [[Mike Bostock]]'s earlier [[bostock-2012-d3-hive-plots|D3 implementation (2012)]] was influential but architecturally different (monolithic JS vs. hiveplotlib-javascript's pure rendering layer).

## Current State of Knowledge

| Category | Count |
|----------|-------|
| Sources ingested | 11 |
| Entity pages | 4 |
| Concept pages | 15 |
| Analysis pages | 3 |
| **Total wiki pages** | **33** (+ index, log, overview) |

## Research Directions

### GNN Heterogeneity Diagnosis

Hive plots aren't just a visualization tool — they are potentially a **diagnostic tool for machine learning evaluation**. The [[gnn-heterogeneity-hive-plots|GNN heterogeneity proposal]] argues that [[hive-plot-matrix|HivePlotMatrix]] can expose classification performance variation that standard [[gnn-evaluation|aggregate GNN metrics]] mask — at both the node level and, critically, the **edge level**.

The core idea: train a [[graph-neural-networks|GNN]] on a node classification benchmark (e.g., GCN on Cora), then build a HivePlotMatrix where nodes are partitioned by structural properties (degree, community, local homophily, training-set distance) and edges are colored by correct vs. misclassified. The matrix reveals *which structural decomposition exposes the most [[structural-heterogeneity|performance heterogeneity]]* — answering "your model is 95% accurate overall, but where does it fail?"

A deep reading of [[ma-2021-subgroup-fairness|Ma, Deng & Mei (NeurIPS 2021)]] confirms the theoretical foundation: they prove accuracy disparity exists across structural subgroups and identify training-set distance as the key predictor. But their analysis is purely tabular (bar charts), single-variable-at-a-time, and node-level only. A survey of the citing literature ([[subramonian-2024-degree-bias|Subramonian et al. 2024]] — 38 papers on degree bias) and the visual analytics space ([[gnnfairviz-2025|GNNFairViz]]) confirms that **no existing work** uses network visualization for structural subgroup diagnosis, and **no work at all** examines edge-level heterogeneity in GNN performance. The hive plot approach is novel on multiple axes: multi-dimensional sweep, edge-aware visualization, continuous structural gradients, and visual model cards.

This direction requires no new [[hiveplotlib]] features, only application of existing `HivePlotMatrix.from_variable_sweep()` and the NetworkX integration. The in-progress v0.28 [[graph-features|graph-feature API]] makes the path nearly turnkey: a PyTorch Geometric / NetworkX graph carrying predictions ingests via `from_variable_sweep(graph=...)`, and the structural sweep variables (degree, centrality, community labels) compute in the same call through `node_graph_metrics`, retiring the old manual extract-and-merge step.

## Open Questions and Next Steps

- **Neuroscience gap:** Brain connectome networks are a natural fit for hive plots but no published work exists — this is an unexplored opportunity.
- **Social network analysis:** Surprisingly underexplored given the method's strengths for community detection.
- **Differential hive plots in hiveplotlib:** Not yet implemented — could be a valuable addition for version comparison or temporal analysis.
- **Interactive HivePlotMatrix:** HivePlotMatrix (released in v0.27) supports matplotlib and datashader. Bokeh or Plotly backends would enable browser-based panel exploration, achieving what [[perez-2021-hype|HyPE]] did but within the modern hiveplotlib ecosystem.
- **Automated parameter selection:** Given NP-completeness, could heuristic/approximate optimization (genetic algorithms, simulated annealing) help suggest good axis assignments?
- **igraph graph-feature backend:** v0.28's [[graph-features|metric wrappers]] are deliberately namespaced under `graph_features/networkx/`. The roadmap calls for a sibling `igraph` backend (faster community detection in particular) to slot in alongside, which would broaden the menu of structural sweep variables.

## See Also

- [[index]] — Full catalog of wiki pages
