---
title: Wiki Log
type: log
updated: 2026-04-13
---

# Hive Plot Research Wiki — Log

## [2026-04-06] setup | Wiki Initialized
Wiki structure created. Directories: `raw/`, `wiki/sources/`, `wiki/entities/`, `wiki/concepts/`, `wiki/analyses/`. Schema defined in `CLAUDE.md`. Index and log files created. Ready for first source ingest.

## [2026-04-06] ingest | hiveplotlib Python repo
Ingested the hiveplotlib Python repository as a foundational source. Created source summary (`sources/hiveplotlib-python.md`), entity page (`entities/hiveplotlib.md` — designated as main hub), and contributed to 7 concept pages. The library has 6 visualization backends, 46 example notebooks, 422 tests at 100% coverage, and key classes: BaseHivePlot, HivePlot, HivePlotMatrix, NodeCollection, Edges, Axis, P2CP.

## [2026-04-06] ingest | hiveplotlib-javascript repo
Ingested the hiveplotlib-javascript repository. Created source summary (`sources/hiveplotlib-javascript.md`) and entity page (`entities/hiveplotlib-javascript.md`). Single-file D3.js library (~650 lines) that renders hiveplotlib JSON exports as browser SVGs. Maps matplotlib kwargs to SVG/D3 attributes. Published as `@hiveplotlib/d3` on npm.

## [2026-04-06] ingest | Krzywinski et al. 2012
Ingested the foundational hive plot paper. Created source summary (`sources/krzywinski-2012.md`) and entity page (`entities/martin-krzywinski.md`). Paper introduces hive plots as rational alternative to force-directed layouts, with case studies on E. coli regulatory network and cancer gene-disease network. ~257 citations. Created concept pages: hive-plot, node-assignment, edge-rendering, force-directed-layout, hive-plot-matrix, p2cp, bezier-curves.

Pages created: 13. Pages updated: index.md, overview.md, log.md.

## [2026-04-06] ingest | Perez et al. 2021 — HyPE (Hive Panel Explorer)
Ingested via web research. Created source summary (`sources/perez-2021-hype.md`). First interactive formalization of the Hive Panel concept. Python 2.7 + D3.js, demonstrated on forest soil microbiome network (1,880 OTUs, 13,605 edges). Krzywinski is co-author. Tool no longer maintained. Updated hive-plot-matrix.md and martin-krzywinski.md with cross-references.

## [2026-04-06] ingest | Bostock 2012 — D3 Hive Plots
Ingested via web research. Created source summary (`sources/bostock-2012-d3-hive-plots.md`) and entity page (`entities/mike-bostock.md`). Influential D3.js v2 hive plot implementation visualizing Flare toolkit dependencies. Key insight: "force layouts make poor use of the most effective visual channel (position)." Architecturally monolithic vs. hiveplotlib-javascript's rendering-only approach.

## [2026-04-06] ingest | Nöllenburg & Wallinger 2023 — Computing Hive Plots
Ingested via web research. Created source summary (`sources/nollenburg-2023-computing-hive-plots.md`). First formal algorithmic framework for hive plot construction — proves all three subproblems (vertex partitioning, axis ordering, vertex ordering) are NP-complete. Updated hive-plot.md and hive-plot-matrix.md.

## [2026-04-06] ingest | Krzywinski et al. 2017 — Differential Hive Plots
Ingested via web research. Created source summary (`sources/krzywinski-2017-differential.md`) and concept page (`concepts/differential-hive-plot.md`). Visual diff between two networks. Published in Leonardo (MIT Press art/science journal). Not yet implemented in hiveplotlib. Updated martin-krzywinski.md.

## [2026-04-06] ingest | Domain Application Survey
Created three application domain concept pages from web research:
- `concepts/applications-bioinformatics.md` — Strongest domain (HyPE, HiveGraph, pleiotropy)
- `concepts/applications-cybersecurity.md` — Most innovative (DDoS classification via hive plot images as ML features)
- `concepts/applications-software-engineering.md` — Linux kernel, Flare toolkit, code dependencies
Updated hive-plot.md with application domain links.

## [2026-04-06] analysis | Karate Club Walkthrough
Created `analyses/karate-club-walkthrough.md` — step-by-step walkthrough of the karate_club.ipynb example. Covers dataset, partitioning strategy (faction membership), sorting (degree), edge styling (color by intra/inter-faction), and key insights (faction separation visible, leaders prominent, cross-faction connections NOT driven by sociability). Also compared with election_96 and bitcoin_user_ratings examples.

## [2026-04-06] lint | First Health Check
Performed first wiki lint pass. Results below.

**Pages created this session:** 23 content pages + index/overview/log
**Pages updated this session:** hive-plot.md, hive-plot-matrix.md, martin-krzywinski.md

## [2026-04-13] ingest + analysis | GNN Heterogeneity Research Direction
Extended wiki with GNN evaluation research direction. Created concept pages for graph neural networks, GNN evaluation metrics, and structural heterogeneity. Created research proposal analysis page exploring use of HivePlotMatrix to expose classification heterogeneity hidden by aggregate GNN metrics. Added source stubs for Ma et al. 2021 (subgroup fairness) and Kipf & Welling 2017 (foundational GCN). Updated 4 existing pages with cross-references. Pages created: 6. Pages updated: 7 (hive-plot-matrix.md, hiveplotlib.md, hive-plot.md, applications-cybersecurity.md, index.md, overview.md, log.md).

## [2026-04-13] query + ingest | Deep Dive on Ma et al. 2021 and GNN Fairness Literature
Full reading of Ma, Deng & Mei 2021 PDF + citation landscape research. Key findings:

**Ma et al. corrections and details:** Authors are Jiaqi Ma, Junwei Deng, Qiaozhu Mei (UMich), not Jing Ma et al. as previously recorded. Paper provides PAC-Bayesian generalization bound (Theorem 3) showing training-set distance (ε_m) is the key predictor of subgroup accuracy. Node centrality (degree, closeness, betweenness, PageRank) shows no clear monotonic trend in their 5-bin bar charts — but this does not rule out non-linear patterns that continuous hive plot axes could capture.

**Citation landscape (2022–2025):** Surveyed citing papers including Subramonian et al. (NeurIPS 2024, 38-paper degree bias survey), GraphPatcher (NeurIPS 2023), DegFairGNN (AAAI 2023), FairACE (2025). All metrics/training-focused, no visualization frameworks. GNNFairViz (IEEE TVCG 2025) is the closest visual analytics competitor but targets demographic-attribute fairness, not structural subgroup diagnosis.

**Novelty confirmed on three axes:** (1) No existing work uses network visualization for structural subgroup performance diagnosis. (2) Edge-level heterogeneity is entirely unexplored in the GNN fairness literature — nobody has examined whether misclassification errors cluster along specific edge types. (3) Multi-dimensional decomposition sweep has no precedent.

**Analysis page reframed:** Added training-set distance and geodesic distance as sweep variables (per Ma et al.). Elevated edge-level heterogeneity as a distinct and novel contribution. Added Related Work section positioning vs. Ma et al., Subramonian et al., and GNNFairViz. Kept degree/centrality as important sweep candidates (non-linear patterns may exist beyond what bar charts capture).

Pages created: 2 (subramonian-2024-degree-bias.md, gnnfairviz-2025.md). Pages rewritten: 1 (ma-2021-subgroup-fairness.md — from stub to full source page). Pages updated: 3 (gnn-heterogeneity-hive-plots.md, index.md, overview.md).

## [2026-04-13] ingest | Subramonian, Kang & Sun 2024 (Degree Bias)
Full web-research ingest. Authors: Arjun Subramonian, Jian Kang, Yizhou Sun (UCLA). Surveys 38 papers on GNN degree bias, catalogs 10+ competing hypotheses, finds many are contradictory (notably H5 vs. H10 on representation variance). Provides first rigorous probabilistic bounds via collision probability, prediction homogeneity, and training dynamics. Key insight: RW and SYM graph filters create degree bias through fundamentally different geometric mechanisms. Heterophilic graphs show weaker degree bias. Visualization: standard scatter/PCA plots only, no network visualization. New sweep variable candidates for HivePlotMatrix: inverse collision probability, prediction homogeneity. Rewrote stub to full source page. Updated concept pages: structural-heterogeneity.md, gnn-evaluation.md, graph-neural-networks.md.

## [2026-04-13] ingest | GNNFairViz (Ye et al. 2025)
Full web-research ingest. Authors: Xinwu Ye et al. (Fudan + Cagliari). Multi-view interactive visual analytics for GNN fairness. Core contribution: bias taxonomy separating model bias from data bias (attribute + structural) via counterfactual what-if comparisons. Key finding: "Overwhelming Effect" — minority node representations dominated by majority neighborhoods through message passing. Uses embedding projections and connectivity summaries, NOT hive plots or rational layouts. Focused on demographic-attribute fairness (gender, race), NOT structural subgroup fairness. Code public on GitHub and PyPI. Confirmed as closest visual analytics competitor but operating in fundamentally different space from HivePlotMatrix proposal. Rewrote stub to full source page. Updated concept pages: gnn-evaluation.md, graph-neural-networks.md.

## [2026-04-13] analysis | Cora Prototype Implementation Plan
Created `analyses/cora-prototype-plan.md` — concrete 5-phase implementation plan for building the GNN heterogeneity HivePlotMatrix prototype on Cora. References actual hiveplotlib API: `HivePlotMatrix.from_variable_sweep()`, `networkx_to_nodes_edges()`, `NodeCollection.create_partition_variable()`, tagged `Edges` for edge-level analysis. Includes: environment setup, GNN training (GCN/GAT/GraphSAGE via PyTorch Geometric), structural property computation (degree, centrality, community, local homophily, training-set distance per Ma et al.), edge-level metadata (error score, cross-group edges, degree ratio), partition sweep, model comparison, and edge-focused visualization. Plan is self-contained for execution in a fresh session.

## [2026-05-17] structure | Added plans/ directory
Added `wiki/plans/` to host working plans alongside ADRs. Motivation: multi-day / multi-week structural plans benefit from version control, and plans + ADRs in the same repo make ADR promotion an intra-repo distillation. Seeded with two in-flight plans: `i-want-to-plan-optimized-hoare.md` and `networkx-metric-expansion-and-backend-refactor.md`. Added `plans/README.md` flagging the directory as scratch work in progress, not curated wiki content. Updated `CLAUDE.md` directory structure.
