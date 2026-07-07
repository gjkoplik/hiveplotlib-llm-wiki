---
title: Overview
type: overview
created: 2026-04-06
updated: 2026-07-04
sources: [krzywinski-2012, hiveplotlib-python-repo, hiveplotlib-javascript-repo, perez-2021-hype, bostock-2012-d3-hive-plots, nollenburg-2023, krzywinski-2017-differential, ma-2021-subgroup-fairness, kipf-2017-gcn, subramonian-2024-degree-bias, gnnfairviz-2025]
tags: [hive-plot, hiveplotlib, network-visualization]
---

# Hive Plot Research Wiki — Overview

This wiki tracks research, theory, and practice around **[[hive-plot|hive plots]]** and **[[hiveplotlib]]**, the Python library at the center of this work.

## The Big Picture

Hive plots are a network visualization method introduced by [[Martin Krzywinski]] in [[krzywinski-2012|2012]]. The core insight: traditional [[force-directed-layout|force-directed layouts]] produce non-reproducible, non-quantitative "hairballs." Hive plots fix this by assigning nodes to radial axes based on meaningful structural properties ([[node-assignment]]) and drawing edges as [[bezier-curves|Bézier curves]] ([[edge-rendering]]).

The method has been adopted across multiple domains — most strongly in [[applications-bioinformatics|bioinformatics]] (its origin), but also innovatively in [[applications-cybersecurity|cybersecurity]] (where hive plot images serve as ML features for DDoS detection) and [[applications-software-engineering|software engineering]] (code dependency visualization). A running catalog of exploratory applications lives at [[examples-and-applications]]; one recent exploration, [[soccer-passing-hive-plots|soccer passing networks]], sharpens the comparability argument by fixing the layout by pitch structure (so two teams read apples-to-apples, the documented fix for average-position pass-network hairballs) and weighting edges by possession value ([[expected-threat|expected threat]]) rather than raw count.

## The Theory

[[nollenburg-2023-computing-hive-plots|Nöllenburg & Wallinger (2023)]] formalized hive plot construction as a combinatorial optimization problem and proved all three subproblems (vertex partitioning, cyclic axis ordering, vertex ordering) are **NP-complete**. This validates the exploratory approach: use domain knowledge for initial parameters, then explore alternatives via [[hive-plot-matrix|HivePlotMatrix]] panels. [[krzywinski-2017-differential|Differential hive plots]] (2017) extend the method to network comparison via visual diffs.

## The Software

**[[hiveplotlib]]** is the main hub of research and development. It is a comprehensive Python library (by the wiki maintainer) supporting six visualization backends, with 50 example notebooks, 100% test coverage, and features including:
- High-level `HivePlot` API and low-level `BaseHivePlot` for full control
- [[hive-plot-matrix|HivePlotMatrix]] for comparative visualization (released in v0.27)
- [[p2cp|Polar Parallel Coordinates]] for tabular data
- Numba-accelerated [[bezier-curves|Bézier curve]] generation
- [[graph-features|Graph-feature wrappers]] (shipped in v0.28) — 35 node + 8 edge NetworkX metrics computable straight onto axes, with `HivePlot(graph=...)` / `HivePlotMatrix.from_*(graph=...)` ingest and `HivePlot.to_networkx()` export making NetworkX a first-class input *and* output

**[[hiveplotlib-javascript]]** is the browser companion — it renders hiveplotlib's JSON exports as interactive SVGs using D3.js. [[Mike Bostock]]'s earlier [[bostock-2012-d3-hive-plots|D3 implementation (2012)]] was influential but architecturally different (monolithic JS vs. hiveplotlib-javascript's pure rendering layer).

## Current State of Knowledge

| Category | Count |
|----------|-------|
| Source pages | 27 |
| Entity pages | 5 |
| Concept pages | 21 |
| Analysis pages | 14 |
| **Total wiki pages** | **67** (+ index, log, overview) |

## Research Directions

### GNN Heterogeneity Diagnosis

Hive plots were proposed as a **diagnostic tool for machine learning evaluation**: the [[gnn-heterogeneity-hive-plots|GNN heterogeneity proposal]] argued that a [[hive-plot-matrix|HivePlotMatrix]] could expose classification performance variation that aggregate [[gnn-evaluation|GNN metrics]] mask. The prototype ran (GCN/GAT/GraphSAGE on Cora and CiteSeer), and an adversarial literature check followed. The honest outcome, filed in [[gnn-heterogeneity-findings]], is mostly negative and worth remembering:

- The strongest simple predictor (local homophily) is a **confirmation** of known results ([[ma-2022-homophily-necessity|Ma 2022]] and others), not a discovery.
- The proposal's headline novelty, **edge-level "error contagion,"** was **refuted**: the apparent both-wrong-edge enrichment (2.22x) collapses to ~1.16x once you condition on community and homophily, and the premise (errors correlate on edges) is textbook ([[huang-2021-correct-and-smooth|Correct & Smooth]], [[jia-benson-2020-residual-correlation|Jia & Benson]]). The earlier "no work examines edge-level heterogeneity" claim was wrong.
- The most interesting effect, **calibration structured by distance-to-training**, turned out to be prior art ([[hsu-2022-gnn-calibration|Hsu et al. 2022]] names it as a calibration factor on the same model and datasets); only a narrow error-conditional refinement survives.
- The one durable contribution is a **residual screen** (fit a covariate baseline for per-node error, form an edge-level independence null, flag joint failures the baseline cannot explain), and even that is novel only as a *composition* of prior parts ([[congalton-1988-error-autocorrelation|Congalton 1988]] join-count, slice discovery, [[jin-2022-gnnlens|GNNLens]]). Its intra-class failure pockets still need a label-noise audit before they can be called model blind spots.

The takeaway for the wiki: the statistics did the finding; the hive plot contributed positional layout, and [[jin-2022-gnnlens|GNNLens]] already occupies the visual-GNN-error-diagnosis space. The technical path is still turnkey (the v0.28 [[graph-features|graph-feature API]] ingests a PyTorch Geometric / NetworkX graph via `from_variable_sweep(graph=...)`), but the research value here is the method and the negative results, not a headline diagnosis.

The response to that dead end is [[gnn-research-directions]]: a deliberately broadened menu that stops fighting the medium. It changes the levers the first pass held fixed (the task, what gets plotted, the datasets, the encoding grammar) and is ranked for **artifact strength** rather than novelty, which is the honest bar for a visualization library. The flagship is [[gnn-over-smoothing]] rendered as a per-layer [[hive-plot-matrix|HivePlotMatrix]] (watch class separation collapse with depth); other directions include per-epoch training movies (the GNN analog of the [[nn-training-dynamics-p2cp-exploration|nn-viz]] work, less contrived because there is a real graph to watch), edge-native link prediction (the edge *is* the prediction, so the medium finally fits), [[differential-hive-plot|differential]] architecture/seed diffs (data already in hand), and Hetionet KG completion (joining the [[hive-plots-for-knowledge-graphs|KG thread]]).

### Spectral Hive Plots

A second direction treats the hive plot as a **readout of a spectral decomposition**. From one spectral clustering you get all three hive-plot inputs at once: the k-way partition (the first cuts) assigns the axes, a *higher* eigenvector the partition does not use sorts nodes within each axis, and the affinity graph supplies the edges. The idea came from a chemistry-minded collaborator; a prototype lives in the `hiveplotlib-spectral` repo, written up in [[spectral-hive-plots]].

A five-angle literature pass settled the novelty question: it is a **novel recombination of established parts**, not new in any single ingredient and not already assembled anywhere. Conventional hive plots never assign axes spectrally ([[krzywinski-2012]], [[perez-2021-hype|HyPE]], [[nollenburg-2023-computing-hive-plots|Nöllenburg & Wallinger]]); [[koren-2005-graph-drawing|spectral graph drawing]] uses eigenvectors as Cartesian coordinates, not radial cluster axes; eigenvector ordering is classic [[atkins-1998-spectral-seriation|spectral seriation]]. Crucially the one non-obvious move, reading within-cluster structure from a higher eigenvector, is already standard in [[nedialkova-2014-diffusion-map-md|diffusion-map molecular dynamics]], which de-risks the idea and locates the novelty in the recombination. The strongest use cases are in chemistry: MD conformational analysis (eigenvectors already carry kinetic meaning) and force-directed molecular-network tools ([[aron-2020-gnps|GNPS]], chemical space networks) that today show only one cluster family at a time.

## Open Questions and Next Steps

- **Neuroscience gap:** Brain connectome networks are a natural fit for hive plots but no published work exists — this is an unexplored opportunity.
- **Social network analysis:** Surprisingly underexplored given the method's strengths for community detection.
- **Differential hive plots in hiveplotlib:** Not yet implemented — could be a valuable addition for version comparison or temporal analysis.
- **Interactive HivePlotMatrix:** HivePlotMatrix (released in v0.27) supports matplotlib and datashader. Bokeh or Plotly backends would enable browser-based panel exploration, achieving what [[perez-2021-hype|HyPE]] did but within the modern hiveplotlib ecosystem.
- **Automated parameter selection:** Given NP-completeness, could heuristic/approximate optimization (genetic algorithms, simulated annealing) help suggest good axis assignments?
- **igraph graph-feature backend:** v0.28's [[graph-features|metric wrappers]] are deliberately namespaced under `graph_features/networkx/`. The roadmap calls for a sibling `igraph` backend (faster community detection in particular) to slot in alongside, which would broaden the menu of structural sweep variables.

## See Also

- [[index]] — Full catalog of wiki pages
