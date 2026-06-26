---
title: Wiki Index
type: index
updated: 2026-06-25
---

# Hive Plot Research Wiki — Index

## Overview

- [[overview]] — High-level synthesis of the wiki's current state of knowledge

## Architecture Decision Records

- [[0001-networkx-integration|ADR 0001 — NetworkX integration]] — v0.28 NetworkX story: unified `graph=` input (not a `from_networkx` classmethod), the `graph_features` package (~43 metrics), `compute_graph_metrics`, `to_networkx` + JSON export, `graph_metric_backend` dispatch, and the `@requires_graph_type` conflict-validation layer. Accepted.

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

## Entities

- [[hiveplotlib]] — **Main hub.** Python library for hive plots, created by Gary Koplik (2 sources)
- [[hiveplotlib-javascript]] — JavaScript/D3 visualization library for hiveplotlib JSON exports (1 source)
- [[Martin Krzywinski]] — Creator of hive plots and Circos (3 sources)
- [[Mike Bostock]] — Creator of D3.js, published influential hive plot implementation (1 source)

## Concepts

- [[hive-plot]] — Rational, quantitative network visualization using radial linear axes (2 sources)
- [[node-assignment]] — How nodes are classified onto axes and positioned along them (2 sources)
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
- [[structural-heterogeneity]] — Uneven graph topology and its impact on GNN performance (2 sources)

## Analyses

- [[examples-and-applications|Examples & Applications Catalog]] — **Hub.** Living catalog of exploratory examples and applications of hiveplotlib (NN-viz, knowledge graphs, GNN heterogeneity, Karate Club)
- [[karate-club-walkthrough]] — Step-by-step hiveplotlib walkthrough using Zachary's Karate Club (1 source)
- [[gnn-heterogeneity-hive-plots]] — Research proposal: HivePlotMatrix for GNN classification diagnostics, node- and edge-level (7 sources)
- [[cora-prototype-plan]] — Implementation plan for Cora GNN evaluation prototype (3 sources)
- [[nxviz-comparison]] — Capability comparison: nxviz (revived June 2026) vs hiveplotlib, for JOSS State of the field (1 source — web research)
- [[hiveplotlib-research-impact]] — Citations, downstream uses, and PyPI download stats for the JOSS Research impact statement; thin-but-real (1 source — web research)
- [[joss-ai-disclosure-precedents]] — Accepted-JOSS-paper AI-disclosure examples by posture, to calibrate the paper's mandatory disclosure (1 source — web research)
- [[hive-plots-for-knowledge-graphs]] — Can hive plots render KGs? Reframes them as scoped metapath views; a seven-pattern taxonomy, a Hetionet example, what hiveplotlib supports today, and the ergonomic gaps (3 sources + web research)
- [[nn-training-dynamics-p2cp-exploration]] — Throwaway prototype watching a tiny MLP learn MNIST as a P2CP movie over training; reusable hiveplotlib usage notes (BaseHivePlot, datashade spread-then-shade), the output-vs-hidden-space finding, and the Grand-Tour-complementary novelty framing (1 source)
