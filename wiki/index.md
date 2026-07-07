---
title: Wiki Index
type: index
updated: 2026-07-06
---

# Hive Plot Research Wiki — Index

## Overview

- [[overview]] — High-level synthesis of the wiki's current state of knowledge

## Architecture Decision Records

- [[0001-networkx-integration|ADR 0001 — NetworkX integration]] — v0.28 NetworkX story: unified `graph=` input (not a `from_networkx` classmethod), the `graph_features` package (~43 metrics), `compute_graph_metrics`, `to_networkx` + JSON export, `graph_metric_backend` dispatch, and the `@requires_graph_type` conflict-validation layer. Accepted.
- [[0002-performance-regression-harness|ADR 0002 — Performance regression + equivalence harness]] — the gate for the scaling milestone: pytest relative same-run ratio gates + ASV capture-at-merge history (hybrid), the dtype-aware equivalence wall (sound by construction), two-tier peak-RSS measurement (kernel `getrusage`, `tracemalloc` rejected; tier 2 provisional), pinned tiny/small/medium/large scenarios, canary-armed dormant gates, and the `perf_harness` CI-signal split. Accepted.

## Sources

- [[hiveplotlib-python|hiveplotlib — Python Library Source]] — Comprehensive Python library for hive plots; 6 backends, 50 examples, 100% test coverage; v0.28 streamlines NetworkX integration (1 source)
- [[hiveplotlib-javascript|hiveplotlib-javascript — D3 Visualization Source]] — JavaScript/D3 companion for browser rendering of hiveplotlib JSON exports (1 source)
- [[krzywinski-2012|Krzywinski et al. 2012]] — Foundational paper introducing hive plots (1 source)
- [[perez-2021-hype|Perez et al. 2021 — HyPE]] — Hive Panel Explorer: interactive panel of hive plots for parameter exploration (1 source)
- [[bostock-2012-d3-hive-plots|Bostock 2012 — D3 Hive Plots]] — Influential D3.js implementation with Flare dependency graph (1 source)
- [[nollenburg-2023-computing-hive-plots|Nöllenburg & Wallinger 2023]] — Formal combinatorial framework proving hive plot construction is NP-complete (1 source)
- [[krzywinski-2017-differential|Krzywinski et al. 2017 — Differential Hive Plots]] — Visual diff between two networks (1 source)
- [[ma-2021-subgroup-fairness|Ma, Deng & Mei 2021 — Subgroup Fairness]] — PAC-Bayesian analysis proving GNN accuracy varies by subgroup; training-set distance is key driver (1 source)
- [[kipf-2017-gcn|Kipf & Welling 2017 — GCN]] — Foundational graph convolutional network paper (stub — 0 sources)
- [[subramonian-2024-degree-bias|Subramonian, Kang & Sun 2024 — Degree Bias]] — NeurIPS survey of 38 papers + first rigorous probabilistic bounds for degree bias; filter-type matters (0 sources — web research)
- [[gnnfairviz-2025|GNNFairViz (Ye et al. 2025)]] — Multi-view visual analytics for GNN demographic-attribute fairness; counterfactual bias diagnosis (0 sources — web research)
- [[hsu-2022-gnn-calibration|Hsu et al. 2022 — What Makes GNNs Miscalibrated?]] — Names distance-to-training as a GNN calibration factor; the prior art demoting the prototype's calibration finding (0 sources — web research)
- [[huang-2021-correct-and-smooth|Huang et al. 2021 — Correct & Smooth]] — Propagates edge-correlated errors to improve predictions; establishes "errors cluster on edges" as textbook (0 sources — web research)
- [[jia-benson-2020-residual-correlation|Jia & Benson 2020 — Residual Correlation in GNN Regression]] — Models GNN residual correlation on edges (regression); companion to Correct & Smooth (0 sources — web research)
- [[congalton-1988-error-autocorrelation|Congalton 1988 — Spatial Autocorrelation of Classification Errors]] — Join-count both-wrong statistic on error maps; the sharpest antecedent for the residual screen (0 sources — web research)
- [[ma-2022-homophily-necessity|Ma et al. 2022 — Is Homophily a Necessity?]] — GNNs fail when intra-class neighborhoods are indistinguishable; the mechanism behind the intra-class failure pockets (0 sources — web research)
- [[jin-2022-gnnlens|Jin et al. 2022 — GNNLens]] — Interactive visual analytics for GNN error diagnosis; the nearest prior art to the hive-plot approach (0 sources — web research)
- [[koren-2005-graph-drawing|Koren 2005 — Drawing Graphs by Eigenvectors]] — Spectral graph drawing: Laplacian eigenvectors as Cartesian coordinates; the prior art the spectral hive plot departs from (0 sources — web research)
- [[atkins-1998-spectral-seriation|Atkins et al. 1998 — Spectral Seriation]] — Fiedler-vector ordering provably recovers a 1-D chain order; prior art for the within-axis sort (0 sources — web research)
- [[nedialkova-2014-diffusion-map-md|Nedialkova et al. 2014 — Diffusion Maps for Peptide Folding]] — v1 cuts folded/unfolded while higher eigenvectors resolve within-state substructure; validates the higher-eigenvector sort in chemistry (0 sources — web research)
- [[aron-2020-gnps|Aron et al. 2020 — GNPS Molecular Networking]] — Force-directed MS/MS molecular networks; in-browser view shows one molecular family at a time (the tooling gap) (0 sources — web research)
- [[scalfani-2022-chemical-space-networks|Scalfani et al. 2022 — Chemical Space Networks]] — CSNs default to Fruchterman-Reingold force-directed layout, no spectral component (0 sources — web research)
- [[inoue-2010-diffusion-ppi|Inoue et al. 2010 — Diffusion Spectral Clustering of PPI]] — Radial spectral embedding of PPI networks; closest spatial analog, but a scatter not a hive (0 sources — web research)

## Entities

- [[hiveplotlib]] — **Main hub.** Python library for hive plots, created by Gary Koplik (2 sources)
- [[hiveplotlib-javascript]] — JavaScript/D3 visualization library for hiveplotlib JSON exports (1 source)
- [[Martin Krzywinski]] — Creator of hive plots and Circos (3 sources)
- [[Mike Bostock]] — Creator of D3.js, published influential hive plot implementation (1 source)
- [[statsbomb|StatsBomb]] — Soccer analytics company; its free open event data (via statsbombpy) is behind the soccer-passing exploration (0 sources — maintainer domain knowledge)

## Concepts

- [[hive-plot]] — Rational, quantitative network visualization using radial linear axes (2 sources)
- [[node-assignment]] — How nodes are classified onto axes and positioned along them (2 sources)
- [[spectral-clustering]] — Partitioning a graph via Laplacian eigenvectors; normalizations (rw vs sym), eigenvector bookkeeping, local scaling; the method behind spectral hive plots (3 sources)
- [[edge-rendering]] — How edges are drawn as Bézier curves with layered styling (3 sources)
- [[graph-features]] — NetworkX node/edge metric wrappers (35 node + 8 edge) for partition/sort variables; new in v0.28 (1 source)
- [[force-directed-layout]] — The dominant layout approach that hive plots replace (1 source)
- [[hive-plot-matrix]] — Comparative grid of hive plots for parameter exploration (2 sources)
- [[differential-hive-plot]] — Visual diff between two hive plots for network comparison (1 source)
- [[p2cp]] — Polar Parallel Coordinates for tabular data (1 source)
- [[bezier-curves]] — Bézier curve mathematics and numba-accelerated implementation (1 source)
- [[knowledge-graph]] — Heterogeneous typed graphs (RDF & property graphs): the data model and why whole-graph drawing fails (0 sources — web research)
- [[metapath]] — Sequence of node/relation types; the slice that maps a KG onto two or three hive-plot axes (0 sources — web research)
- [[applications-bioinformatics]] — Strongest adoption domain: gene networks, microbiome, protein interactions (2 sources)
- [[applications-cybersecurity]] — Most innovative domain: ML featurization of hive plot images (0 sources — web research)
- [[applications-software-engineering]] — Code dependency visualization (2 sources)
- [[graph-neural-networks]] — GNN architectures, message passing, benchmarks (2 sources)
- [[gnn-evaluation]] — Standard GNN metrics and the structural heterogeneity gap (2 sources)
- [[gnn-over-smoothing]] — Depth-driven collapse of node embeddings; the flagship broadened direction, rendered as a per-layer hive plot matrix (1 source)
- [[structural-heterogeneity]] — Uneven graph topology and its impact on GNN performance (2 sources)
- [[fixed-layout-comparability]] — Why fixing hive-plot axes by structure makes two networks directly comparable, and the pin-the-axis-ranges corollary (1 source)
- [[expected-threat|Expected Threat (xT)]] — Grid-based soccer possession-value model (Singh 2018); the native fix for "all passes look equal" as value-weighted hive-plot edges (0 sources — maintainer domain knowledge)
- [[flow-motifs|Flow motifs]] — Player-identity fingerprint of 3-pass sequences (Gyarmati et al. 2014); rendered spatially on a fixed hive-plot layout (0 sources — maintainer domain knowledge)

## Analyses

- [[examples-and-applications|Examples & Applications Catalog]] — **Hub.** Living catalog of exploratory examples and applications of hiveplotlib (NN-viz, knowledge graphs, GNN heterogeneity, Karate Club)
- [[karate-club-walkthrough]] — Step-by-step hiveplotlib walkthrough using Zachary's Karate Club (1 source)
- [[gnn-heterogeneity-findings]] — **Empirical results + prior-art check.** Ran the Cora/CiteSeer prototype, then adversarially checked every finding: homophily confirmed-but-known, edge contagion refuted, calibration-by-distance is prior art (Hsu 2022), residual screen novel only as a composition. The honest scorecard (10 sources)
- [[gnn-heterogeneity-hive-plots]] — Research proposal (superseded by the findings page; its edge-level novelty claims did not survive) (7 sources)
- [[gnn-research-directions]] — **Broadened menu.** Reopens the GNN scope past node-classification error diagnosis: over-smoothing per-layer matrices, per-epoch training movies, edge-native link prediction, differential architecture/seed diffs, Hetionet KG completion, and mechanism views. Ranked for artifact strength, honest that these are visual wins not novelty wins, with a dispatch decomposition (5 sources)
- [[cora-prototype-plan]] — Implementation plan for the GNN evaluation prototype (shipped; see findings page) (3 sources)
- [[nxviz-comparison]] — Capability comparison: nxviz (revived June 2026) vs hiveplotlib, for JOSS State of the field (1 source — web research)
- [[hiveplotlib-research-impact]] — Citations, downstream uses, and PyPI download stats for the JOSS Research impact statement; thin-but-real (1 source — web research)
- [[joss-ai-disclosure-precedents]] — Accepted-JOSS-paper AI-disclosure examples by posture, to calibrate the paper's mandatory disclosure (1 source — web research)
- [[hive-plots-for-knowledge-graphs]] — Can hive plots render KGs? Reframes them as scoped metapath views; a seven-pattern taxonomy, a Hetionet example, what hiveplotlib supports today, and the ergonomic gaps (3 sources + web research)
- [[nn-training-dynamics-p2cp-exploration]] — Throwaway prototype watching a tiny MLP learn MNIST and Fashion-MNIST as hive-plot/P2CP movies over training; the bounded-panel prior-art position (softmax-P2CP as a Grand-Tour complement, cross-layer co-activation as a narrow three-property intersection, lock-in as validated-inconclusive), reusable hiveplotlib usage notes (HivePlotMatrix, datashade spread-then-shade), and the output-vs-hidden-space finding (1 source)
- [[soccer-passing-hive-plots]] — Throwaway prototype rendering soccer passing networks as fixed-layout hive plots (pitch thirds as axes, pass endpoints as nodes) for cross-team comparability; a nine-figure catalog (flagship comparison, formation lineups, shot build-up, xT-weighted arcs, spatial flow-motifs, turnovers, Barcelona motifs at scale), reusable statsbombpy gotchas, and the fixed-layout-comparability and spatial-flow-motif novelty angles (1 source)
- [[hiveplotlib-bioinformatics-examples]] — Public example repo of real biological-network hive plots (C. elegans connectome, E. coli RegulonDB GRN) in the strongest adoption domain; honest finding that both are credible real-data demos but neither is a "hero" figure (real networks control for nothing, so comparisons collapse to density), with the instant-comparison hero relocated to the engineered Datasaurus-for-networks work. Reconciled 2026-07-04 against a bounded prior-art panel: both examples are re-tellings of published figures (Tabacof 2013 connectome, Cook 2019 dimorphism, Krzywinski 2012 GRN), the GRN role vocabulary is Yu & Gerstein 2006's (not Krzywinski's), the RegulonDB data is under a restrictive EULA (not CC-BY), and two "dead ends" (C. elegans position metadata, the structural dimorphism figure) are actually buildable (7 sources)
- [[same-stats-different-graphs-grounding|Same Stats, Different Graphs: grounding & novelty]] — Literature-mode research run on the matched-degree-sequence demo: network-science foundations confirmed, a novel synthesis versus Chen et al. 2018 (which holds aggregate stats + spring layout, no hive plots), and the honest caveat that the discriminating power is the degree-rank ordering, not the hive glyph; plus the sexual-contact anchor correction (10 sources, web research)
- [[spectral-hive-plots]] — Research direction: a hive plot driven entirely from a spectral decomposition (axes = spectral cuts, within-axis order = a higher eigenvector, edges = the affinity graph). Verdict: novel recombination of known parts, mechanism validated in diffusion-map chemistry; ranked chemistry use cases (10 sources)
