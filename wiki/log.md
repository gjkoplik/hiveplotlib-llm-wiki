---
title: Wiki Log
type: log
updated: 2026-06-12
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

## [2026-05-31] sync | hiveplotlib v0.28 NetworkX streamlining (scheduled weekly update)
Synced the wiki to a week of hiveplotlib development (commits 2026-05-25 → 2026-05-30) building toward v0.28.0 (`0.28.0a0`), themed "streamlined NetworkX usage." Key shipped behavior: NetworkX is now a first-class input *and* output — `HivePlot(graph=...)` and `HivePlotMatrix.from_partition/from_variable_sweep(graph=...)` ingest a graph directly (with `graph_directed`/`graph_multigraph` defaulting off the graph's own type), `HivePlot.to_networkx()` / `converters.nodes_edges_to_networkx()` export back to any of the four graph classes, and a new `graph_features` package wraps 35 node + 8 edge NetworkX metrics indexed by string name, computable straight onto axes via `node_graph_metrics` / `edge_graph_metrics` at construction. The wrappers are namespaced under `graph_features/networkx/` to leave room for a future `igraph` backend (per the roadmap). Also: public `HivePlotMatrix.backend` property; deprecated `hive_plot_n_axes()` removed; several fixes (integer-partition `KeyError`, `Edges.copy`/`reset_edges` `relevant_edges` sync, `from_tags` tag validation). Three new gallery notebooks (Computing Graph Metrics, Exporting to NetworkX, Exporting to JSON) bring the example count to 49.

Pages created: 1 (`concepts/graph-features.md`). Pages updated: 4 (`entities/hiveplotlib.md`, `concepts/hive-plot-matrix.md`, `index.md`, `overview.md`).

Web scan for new external hive plot work (past week): nothing new surfaced. The most recent notable tool remains [[perez-2021-hype|HyPE]] (2020/2021); no 2026 hive plot papers or tools found.

## [2026-06-06] sync + lint | v0.28 graph-metric type handling refinements (scheduled weekly update)
Synced the wiki to a week of hiveplotlib development (commits 2026-05-30 → 2026-05-31 plus in-flight working-tree changes on the `46-more-streamlined-networkx-usage-and-support` branch). No new public API since last week's `0.28.0a0` sweep; this week refines how the graph-metrics feature picks the internal graph's type. Two refinements worth recording:

- **Directedness inferred from the requested metric set.** Building a `HivePlot` / `HivePlotMatrix` from `nodes` / `edges` with `graph_directed` unset now infers directedness from the metrics requested (`triangles` builds an undirected internal graph, `in_degree` a directed one), falling back to `True` when no requested metric cares. A self-contradictory request (one metric needs directed, another undirected) raises the up-front conflict `ValueError` rather than silently choosing. Building from a `graph` keeps that graph's own type and never re-infers. New internal helpers `_infer_graph_directed` / `_as_metric_name_list`; `_graph_directed` is now `Optional[bool]` (`None` = unpinned, re-inferred per call). `compute_graph_metrics()` still validates only and never infers.
- **Parallel-edge-collapse warning (landing on the branch).** A new `warn_on_parallel_edge_collapse` parameter (default `True`) plus a `_warn_if_parallel_edges_collapse` seam emits a `UserWarning` when a simple-graph build merges same-direction duplicate `(from, to)` rows last-write-wins (so a `weight`-using metric sees only one row). Reciprocal `(a, b)` / `(b, a)` merges under an undirected build do not warn, since that merge is the metric's definitional behavior. Expanded `converters.nodes_edges_to_networkx` and `compute_graph_metrics` docstrings cover the weight-loss caveat.

Also this week (not wiki-bearing): test-suite speedups (smaller holoviews / hive-plot test graphs, a `test-fast` no-coverage target, `--no-cov` where possible), centralized graph-type checking, and doc-revision passes.

Pages updated: 2 (`concepts/graph-features.md` — added a "Graph-type handling" section; `entities/hiveplotlib.md` — graph-metrics bullet now notes directedness inference and the collapse warning).

Lint pass: no contradictions or stale claims found. The earlier `graph-features.md` line "graph_directed ... otherwise it defaults to True" was the behavior superseded this week; corrected in the new "Graph-type handling" section (the unconditional `True` default now applies only when no requested metric implies directedness). No new orphan pages; `graph-features` remains well cross-linked. `index.md` graph-features entry still accurate (no new pages, no count changes).

Web scan for new external hive plot work (past week): nothing new surfaced. arXiv and general search return only the known prior art ([[nollenburg-2023-computing-hive-plots|Computing Hive Plots]], the P2CP-revisited paper, [[perez-2021-hype|HyPE]]); no 2026 hive plot papers or tools found.

## [2026-06-11] sync | graph_metric_backend shipped (backend dispatch for graph metrics)
The graph-metric-backend-dispatch plan fully shipped into v0.28.0 (all five workstreams; final QA: 1317 tests, 100% coverage, docs build clean). A `graph_metric_backend` parameter on `compute_graph_metrics()` / `HivePlot` / all five `HivePlotMatrix` surfaces routes metric computation through networkx's backend dispatch system (nx-parallel tested in CI; nx-cugraph known-good, GPU-only). Semantics: strict up-front validation against the runtime registry (`InvalidGraphMetricBackendError`; degree-family per-metric entries rejected with an actionable message), lenient `NotImplementedError` fallback with INFO logging (the codebase's first stdlib logging), per-metric reserved `"backend"` key with explicit-`None` opt-out, and three-level precedence (per-metric > per-call > stored construction intent, mirroring `graph_directed`). New gallery notebook: Graph Metric Backends. Feeds the combined v0.28 close-out ADR; no standalone record.

Pages updated: 2 (`entities/hiveplotlib.md` — v0.28 NetworkX-integration bullet + notebook count; `concepts/graph-features.md` — new "Backend dispatch (v0.28)" section including the three-senses-of-backend naming-audit triangle). Plan stays in `plans/` for now (two sibling v0.28 plans still in flight; archiving deferred to the combined ADR promotion).

## [2026-06-12] analysis | JOSS submission — Workstream B evidence filed
Filed Workstream B evidence for the JOSS submission plan (`plans/joss-submission.md`) from completed web research (all accessed 2026-06-12). Created three analysis pages feeding the paper: `analyses/nxviz-comparison.md` (State of the field — nxviz revived June 2026 via v0.8.0 plotly backend + v0.9.0 chord diagrams; deprecated OO `HivePlot`, functional 3-group-capped API; fair per-dimension deltas vs hiveplotlib, including the now-stale "matplotlib-only" claim — nxviz has 2 backends as of June 2026); `analyses/hiveplotlib-research-impact.md` (Research impact — thin-but-real: two 2025 bio papers flagged UNVERIFIED/paywalled for maintainer confirmation, IEEE/TDS mentions, PMC7887807 recorded as a false positive, PyPI ~1,743/month, open gaps noted); `analyses/joss-ai-disclosure-precedents.md` (mandatory AI-disclosure wording — policy verbatim, accepted-paper examples grouped minimal/scoped/detailed, recommendation toward the detailed end with final wording gated on Gary). Updated `concepts/hive-plot.md` (competing-implementations table gains a maintenance-status column; nxviz row added). Author-metadata checklist (G3) presented to Gary in the run report, not filed (needs his answers). Pages created: 3. Pages updated: 3 (`concepts/hive-plot.md`, `index.md`, `log.md`).

## [2026-06-13] sync + lint | weekly verification pass (no new public API)
Scheduled weekly run. No new hiveplotlib commits landed this week (HEAD is still 2026-05-31, branch `46-more-streamlined-networkx-usage-and-support`); the substantive v0.28 work this cycle was the `graph_metric_backend` dispatch feature (already filed 2026-06-11) and the JOSS Workstream B research (already filed 2026-06-12). The tool changes are therefore already captured: the `entities/hiveplotlib.md` hub, `concepts/graph-features.md`, and `index.md` all reflect backend dispatch, directedness inference, and the parallel-edge-collapse warning. This run verified that and ran a lint sweep rather than re-documenting shipped work.

Lint fixes: corrected the example-notebook count from 49 to 50 (the Graph Metric Backends gallery notebook pushed the count up; `examples/*.ipynb` now lists 50) on the two living pages that still carried 49 (`overview.md`, `index.md`). The hub page already read 50.

Lint observations (not fixed, recorded for a future pass): `sources/hiveplotlib-python.md` remains a dated 2026-04-06 snapshot (version `0.27.0a1`, 46 notebooks, 422 test functions); it is a point-in-time source summary, not a living page, so the current numbers live on `entities/hiveplotlib.md`. That same snapshot lists the edge-kwarg hierarchy in a different order than the canonical one in `concepts/edge-rendering.md` and the repo (`all` then `clockwise` then `counterclockwise` then `repeat` then `non_repeat`); worth reconciling when that page is next refreshed. No contradictions, stale claims, or orphan pages found among the living pages; the three new JOSS analyses are registered in `index.md` and cross-linked.

Web scan for new external hive plot work (past week): nothing new surfaced. arXiv and general search return only the known prior art ([[nollenburg-2023-computing-hive-plots|Computing Hive Plots]] (arXiv 2309.02273), the [[p2cp|P2CP-revisited]] paper (arXiv 2109.10193), [[perez-2021-hype|HyPE]], and hiveplotlib's own docs). The June-2026 nxviz revival is already filed in `analyses/nxviz-comparison.md`. No 2026 hive plot papers or new tools found.

Pages updated: 2 (`overview.md`, `index.md` — example count 49 to 50, `updated` dates bumped). No pages created.
