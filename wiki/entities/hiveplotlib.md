---
title: hiveplotlib
type: entity
created: 2026-04-06
updated: 2026-07-06
sources: [hiveplotlib-python-repo, krzywinski-2012]
tags: [hiveplotlib, python, software, network-visualization, main-hub]
---

# hiveplotlib

> **This is the main hub of research and software development for this wiki.**

hiveplotlib is a Python library for generating and visualizing static [[hive-plot|hive plots]]. Created and maintained by Gary Koplik. It is the most comprehensive open-source implementation of Martin Krzywinski's hive plot method.

## Status

- **Current version:** 0.28.0 (released 2026-06-18), focused on streamlined NetworkX integration (see "NetworkX integration" below; the design decisions are distilled in [[0001-networkx-integration|ADR 0001]]).
- **Python support:** 3.10+
- **Repository:** GitLab — `gitlab.com/hiveplotlib` group (migrated to its own group in the v0.28 cycle; visibility not re-verified from the working tree)
- **Install:** `pip install hiveplotlib`
- **License:** see repo

## Core API

### Data Structures
- **`Node`** / **`NodeCollection`** — Individual nodes and pandas-backed collections with partition logic
- **`Edges`** — Tagged edge container supporting multiple edge types
- **`Axis`** — Polar coordinate axis with start/end, angle, node placements

### Main Classes
- **`BaseHivePlot`** — Low-level construction (manual axes, node placement, edge connection)
- **`HivePlot`** — High-level interface (partition → sort → build → plot)
- **`HivePlotMatrix`** — Comparative grid of hive plots, new in v0.27 (see [[hive-plot-matrix]])
- **`P2CP`** — [[p2cp|Polar Parallel Coordinates]] for tabular data

### Visualization Backends
Six backends: matplotlib (default), bokeh, holoviews-bokeh, holoviews-matplotlib, plotly, datashader.

On `master` (post-0.28, unreleased), the `hiveplotlib[holoviews]` extra now requires `holoviews>=1.23` (was `>=1.15`). That release ships the upstream fix for [holoviews #6469](https://github.com/holoviz/holoviews/issues/6469), which let the library drop an internal workaround that had been forcing the single-color case on the holoviews-bokeh back end through `cmap`. Behavior of single-color edges is unchanged; the version floor moves and the hack is gone (GitLab #30, merged 2026-06-26).

### NetworkX integration (v0.28, shipped)

v0.28.0 makes a NetworkX graph a first-class input and output, removing the manual extract-and-merge boilerplate that earlier workflows needed. The combined design rationale (including the declined `from_networkx` classmethod and the deferred future-igraph questions) is recorded in [[0001-networkx-integration|ADR 0001]]:

- **Build from a graph:** `HivePlot(graph=...)` and `HivePlotMatrix.from_partition(graph=...)` / `from_variable_sweep(graph=...)` accept a NetworkX graph directly instead of separate `nodes` / `edges`. `graph_directed` / `graph_multigraph` default off `graph.is_directed()` / `graph.is_multigraph()`, so directed and multigraph inputs flow through without restating the type. (`from_tags` is intentionally excluded: tags are an `Edges`-level concept.)
- **Export to a graph:** `HivePlot.to_networkx()` (and the underlying `converters.nodes_edges_to_networkx()`) round-trip a hive plot back to any of the four NetworkX graph classes; multi-tag `Edges` get a `tag` edge attribute.
- **Graph metrics on construction:** `node_graph_metrics` / `edge_graph_metrics` (plus `*_graph_metric_kwargs` / `*_graph_metric_rename`) compute and attach metric columns *before* partitioning, so they are immediately usable as `partition_variable` / `sorting_variables`. `compute_graph_metrics()` does the same for existing `HivePlot` / `HivePlotMatrix` instances. When you build from `nodes` / `edges` and leave `graph_directed` unset, the internal graph's directedness is inferred from the requested metrics (asking for `triangles` builds an undirected graph; `in_degree` builds a directed one), and a contradictory request raises an up-front `ValueError` instead of silently picking a side. A simple-graph build that collapses same-direction duplicate edges now warns by default (`warn_on_parallel_edge_collapse`). See [[graph-features]] for the full graph-type-handling rules.
- **Backend dispatch for graph metrics:** a `graph_metric_backend` parameter on `compute_graph_metrics()`, `HivePlot`, and all five `HivePlotMatrix` surfaces (including the `from_*` classmethods) routes metric computation through NetworkX's backend dispatch system (nx-parallel tested in CI; nx-cugraph known-good but GPU-only). Unknown or uninstalled backend names raise `InvalidGraphMetricBackendError` up front; metrics a backend does not implement fall back to default NetworkX with an INFO log line (the codebase's first use of stdlib logging). A per-metric reserved `"backend"` key in the metric-kwargs dicts overrides the global value, with explicit `None` as the per-metric opt-out; precedence runs per-metric > per-call > stored construction value, mirroring `graph_directed`. This is orthogonal to the future igraph wrapper-library axis (see [[graph-features]] for the three distinct "backend" senses). New gallery notebook: Graph Metric Backends.
- Removed the long-deprecated `hive_plot_n_axes()` (folded into `HivePlot` back in v0.26).

### Performance regression + equivalence harness (shipped 2026-07-03, dev/CI infrastructure)

In place as of 2026-07-03 (GitLab #52; no user-facing API change, not tied to a release): the gate for the upcoming scaling-to-larger-networks milestone. Every scaling workstream must prove "measurably faster (or no worse), output-equivalent to the proven single-shot datashader path, no regression on small graphs" through it before claiming the win. Design decisions distilled in [[0002-performance-regression-harness|ADR 0002]]; key facts:

- **Pytest ratio gates + ASV history hybrid** — per-MR CI pass/fail is pytest asserting relative, same-run ratios (absolute timings never gate, so the setup survives CI-runner migration by construction); ASV owns timing/memory history and the rolling baseline, its native results JSON committed as-is.
- **Equivalence wall** — dtype-aware curve + raster comparison against the proven path, sound by construction (identity must pass, perturbation must fail, comparing-nothing raises).
- **Two-tier peak-RSS measurement** — tier 1 exact kernel high-water mark (child-reported `getrusage(RUSAGE_SELF)` via a two-level subprocess; `tracemalloc` rejected as blind to numba/numpy C allocations); tier 2 psutil-sampled process-tree helper, provisional pending the scaling chain's instrument selection.
- **Capture-at-merge on the designated machine** — ASV history written only by maintainer-local runs (machine tag `ringtail`) at workstream and dependency-bump merges; CI never writes history.

### Key Features
- **`unify_axes` on `HivePlot`** (post-0.28, on branch `51-add-unify-axes-to-hiveplot` / MR !40, unreleased): a `unify_axes` constructor parameter (`HivePlot(..., unify_axes=True)`) plus a post-hoc `HivePlot.unify_axes()` method give every axis the same value range so radial positions are comparable across axes, mirroring the affordance [[hive-plot-matrix|HivePlotMatrix]] already carries. Same `True | {"vmin", "vmax"}` dict / no-arg / one-sided call shapes as HPM (the underlying resolver is now shared code in `_unify_axes.py`); auto-computed `True` unifies within one instance, while the explicit full dict is what lines separate different-data instances up. Extends ADR 0001's HivePlot ↔ HivePlotMatrix symmetry precedent (see [[fixed-layout-comparability]]).
- [[edge-rendering|Edge kwarg hierarchy]] for layered styling
- **`edge_coverage` introspection** (on `master`, post-0.28, unreleased): a read-only `BaseHivePlot.edge_coverage` property returning an `EdgeCoverage(drawn, total, fraction)` NamedTuple (importable from the top level) that reports how many of the underlying edges the current realization actually draws, plus a percent-only `(N% drawn)` clause in `HivePlot.__repr__`. It answers "why does my plot look like it's missing edges?" without a taxonomy of *why* each edge is undrawn: intra-axis edges without repeat axes, non-adjacent axis pairs past three axes, and overplotting all show up as a coverage below 100%. Deliberately not a warning (the repr clause is the point-of-use signal). Computed lazily behind a single representation-agnostic seam so the in-flight scaling membership-storage change is a one-spot update. `HivePlotMatrix` gets no aggregate (edges appear in multiple cells by design; per-cell coverage is reachable through each populated cell).
- [[bezier-curves|Bézier curve]] generation with numba acceleration
- [[graph-features|Graph-feature wrappers]] — 35 node + 8 edge NetworkX metrics, indexed by string name, attachable to a `NodeCollection` / `Edges`
- NetworkX converters both ways: `networkx_to_nodes_edges()` in, `nodes_edges_to_networkx()` / `HivePlot.to_networkx()` out
- 50 example Jupyter notebooks (four new v0.28 gallery pages: Computing Graph Metrics, Graph Metric Backends, Exporting to NetworkX, Exporting to JSON), plus a "Hive Plot Gotchas" tutorial on `master` (post-0.28, unreleased): a short index of hive plot pitfalls with links, one entry point for the intra-axis / >3-axes / overplotting / parallel-edge-collapse / edge-kwarg-hierarchy / `graph_directed`-inference traps, cross-referencing `edge_coverage`
- 100% test coverage
- Built-in datasets (toy plots, international trade data)
- Agent-first docs: a hand-curated `llms.txt` (llmstxt.org shape) is served at the docs site root on Read the Docs (via `html_extra_path`), ranking the conceptual entry points first, then the API reference, then the gallery, so an agent pointed at the docs finds the load-bearing pages before the gallery breadth. Docs-infrastructure only, no Python API change. Shipped 2026-06-19 (the `llms-full.txt` full-text concatenation was scoped out, with a revival trigger).
- **Cheat Sheet quick-reference page** (`docs/source/cheat_sheet.md`, on `master`, post-0.28, unreleased): a compact, task-oriented "how do I do X" lookup page (MyST markdown, its own "Quick Reference" toctree section between the gallery and autodoc), with per-backend sphinx-design tabs for data in/out, node/edge styling, axis layout, and figure export. Every fenced Python snippet runs in CI by construction via a new hand-rolled docs-snippet test harness (`tests/docs_cheat_sheet_test.py`): the page's main stream execs in one shared namespace in page order, and each sphinx-design tab-item execs against a fresh replay of its main-stream prefix plus that tab alone, so sibling-backend state never leaks and copy-paste examples cannot silently rot. The convention is "testability == copy-paste-ability"; the harness proves per-reader-path runnability, not API correctness (the `**kwargs`-permissive plotting surface still forwards a wrong kwarg name green, so that stays an authoring-time signature-check obligation). A required-reason `no-run` marker allows the rare non-runnable line (e.g. an interactive `plt.show()`). Docs-infrastructure only, no Python API change.
- **README "when and why hive plots" framing** (both `README.md` and `docs/source/README.md`, on `master`, post-0.28, unreleased): a few sentences near the top giving arriving skeptics the reproducibility/interpretability argument (a node's position is set by its own properties, so the same data always draws the same picture and two plots compare side by side) against the force-directed "hairball" (arbitrary positions that shift run to run), pointing at the Introduction to Hive Plots notebook. This is the same statement-of-need argument the JOSS paper will make (reusable phrasing).
- **"Saving Hive Plots for Publication" gallery notebook** (`examples/saving_plots_for_publication.ipynb`, on `master`, post-0.28, unreleased): consolidates vector export (SVG/PDF), raster DPI, and the datashader DPI/rasterization coupling (figure `dpi` sets the rasterization bin count, so `pixel_spread_nodes`/`pixel_spread_edges` must move with it, and cross-figure comparisons must pin all of them). Cross-linked with the Cheat Sheet.

## Ecosystem

- **[[hiveplotlib-javascript]]** — D3.js companion for browser rendering of `.to_json()` exports
- **Prior art:** pyveplot (structural reference), HiveR (R), jhive (Java), D3 hive plots (Bostock)

## Development Priorities

- **Scaling to dramatically larger networks (10M+ edges).** The five-workstream scaling super-plan (membership storage → chunked rasterization → fused build → narwhals input boundary → Dask/GPU passthrough) is the next milestone, now unblocked: the performance regression + equivalence harness ([[0002-performance-regression-harness|ADR 0002]]) is in place as the gate every workstream must clear.
- Additional backends and interactivity for HivePlotMatrix (currently matplotlib + datashader only). The new public `HivePlotMatrix.backend` read-only property makes the current backend inspectable without reaching for the private attribute.
- Future graph-feature backends. The v0.28 metric wrappers live under a `graph_features/networkx/` subpackage; the roadmap calls for a sibling `igraph` backend (notably for community detection) to slot in alongside it. See [[graph-features]].
- Exploring GNN evaluation applications ([[gnn-heterogeneity-hive-plots]])

## Research Directions

See [[examples-and-applications]] for the running catalog of worked examples and application explorations (shipped notebooks through throwaway prototypes).

- [[gnn-heterogeneity-hive-plots]] — Using HivePlotMatrix to diagnose GNN classification heterogeneity. The integration pathway is now even more direct: `HivePlot(graph=...)` / `HivePlotMatrix.from_*(graph=...)` ingest a NetworkX graph straight from PyTorch Geometric, and `node_graph_metrics` / `edge_graph_metrics` compute the structural sweep variables (degree, centrality, community labels, etc.) in one step, replacing the old manual `pd.DataFrame(G.degree, ...) + nodes.data.merge(...)` pattern. See [[graph-features]] for the available structural sweep variables.
- [[nn-training-dynamics-p2cp-exploration]] — Exploration (throwaway `hiveplotlib-nn-viz` prototype, not a library feature): a tiny MLP learning MNIST rendered as a [[p2cp|P2CP]] movie over training. Surfaced reusable usage notes (manual plots via `BaseHivePlot`; `datashade_edges_mpl` is edges-only; spread the aggregate then shade) and the finding that an MLP's discriminative structure reads in output space, not its distributed hidden code.
- [[soccer-passing-hive-plots]] — Exploration (throwaway `hiveplotlib-futbol` prototype, not a library feature): soccer passing networks rendered with a fixed layout (pitch thirds as axes, pass endpoints as nodes) so two teams compare apples-to-apples, the documented fix for the average-position hairball. Exercises datashaded [[hive-plot-matrix|HivePlotMatrix]] comparisons and value-weighted edges ([[expected-threat|xT]]). Surfaced the fixed-bounds axis-pinning lesson (pin `axis_kwargs`, do not let the default infer per-data bounds, or cross-team comparability breaks) and the count-vs-column-reduction datashader semantics.
- [[hiveplotlib-bioinformatics-examples]] — Exploration (public `hiveplotlib-bioinformatics-examples` repo, not a library feature): real biological-network hive plots in hiveplotlib's strongest adoption domain ([[applications-bioinformatics]]). A **C. elegans connectome** example (sensory / interneuron / motor axes) and an **E. coli RegulonDB** signed regulatory example, both built to publishable quality. Honest finding: both are credible real-data demonstrations, but neither is a "hero" figure (real networks control for nothing, so comparisons collapse to density); the instant-comparison hero is the engineered [[same-stats-different-graphs|Datasaurus-for-networks]] work, which these corroborate. Exercises `HivePlot(graph=...)` + `node_graph_metrics`, multi-tag `Edges` for semantic edge coloring, the base [[hive-plot-matrix|HivePlotMatrix]] `unify_axes=True` two-panel comparison with shared-metric node positions, `repeat_axes` for intra-partition edges, and an edge-tag emulation of the roadmapped [[differential-hive-plot]] feature.

## See Also

- [[hiveplotlib-python|Source summary]] — Detailed technical documentation of the repo
- [[hiveplotlib-javascript]] — JavaScript companion
- [[hive-plot]] — The visualization method
- [[Martin Krzywinski]] — Invented hive plots
- [[hive-plot-matrix]] — Comparative visualization
- [[graph-features]] — NetworkX node/edge metric wrappers (new in v0.28)
- [[0001-networkx-integration|ADR 0001 — NetworkX integration]] — Binding design record for the v0.28 NetworkX story
- [[0002-performance-regression-harness|ADR 0002 — Performance regression + equivalence harness]] — Binding design record for the scaling milestone's gate
- [[differential-hive-plot]] — Not yet implemented; potential feature
- [[p2cp]] — Polar Parallel Coordinates
- [[karate-club-walkthrough]] — Step-by-step example walkthrough
- [[nn-training-dynamics-p2cp-exploration]] — Exploration: watching a neural network learn via P2CP
- [[soccer-passing-hive-plots]] — Exploration: soccer passing networks as fixed-layout hive plots
- [[hiveplotlib-bioinformatics-examples]] — Exploration: real biological-network hive plots (C. elegans connectome, E. coli GRN)
