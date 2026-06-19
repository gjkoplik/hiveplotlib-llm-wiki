---
title: "ADR 0001: NetworkX integration (graph= input, graph_features, backend dispatch, conflict validation)"
type: adr
created: 2026-06-18
updated: 2026-06-18
status: Accepted
sources: [hiveplotlib-python-repo]
tags: [adr, hiveplotlib, networkx, graph-metrics, api-design]
---

# ADR 0001: NetworkX integration

**Status:** Accepted (shipped in [[hiveplotlib]] v0.28.0, GitLab #46).

## Context

Before v0.28, the NetworkX integration was one helper: `converters.networkx_to_nodes_edges()` turned an `nx.Graph` into a `(NodeCollection, Edges)` tuple. Users could build hive plots *from* a graph but could not export back to one, had to compute structural metrics (degree, centrality, community labels) in NetworkX and hand-merge them onto node data, and had no first-class "I have a graph, give me a hive plot keyed on degree" path. GitLab issue #46 asked for a substantial expansion: bidirectional conversion, a curated graph-metrics system, and a structure that leaves room for non-NetworkX backends later.

The work was large enough to span four working plans (one umbrella plan plus three spun out for size). This ADR is the single distilled record for the whole v0.28 NetworkX story; the four plans are the verbose historical detail.

## Decision

**Unified `graph=` input, not a `from_networkx` classmethod.** A keyword-only `graph=` parameter on `HivePlot.__init__`, `HivePlotMatrix.from_partition`, and `HivePlotMatrix.from_variable_sweep` accepts an `nx.Graph` as an alternative to `(nodes, edges)`. Exactly one of the two shapes is required; a both / neither / partial input raises a `ValueError` naming the recovery path. `graph_directed` / `graph_multigraph` default to `graph.is_directed()` / `graph.is_multigraph()`, so directed and multigraph inputs flow through without the caller restating the type. `HivePlotMatrix.from_tags` is intentionally graph-less: tags are an `Edges`-level concept (already-bucketed edges), with no coherent graph-to-tag-dimension extraction.

The earlier-shipped (uncommitted) `HivePlot.from_networkx` classmethod and the four `HivePlotMatrix.from_networkx*` classmethods were torn out in favor of `graph=`. Folding NetworkX support into the existing constructors avoids a classmethod explosion across the matrix `from_*` family (one `from_networkx_*` per `from_*` constructor) and keeps the two classes symmetric.

**`graph_features` package.** A new `src/hiveplotlib/graph_features/` package wraps NetworkX metrics behind a uniform string-keyed interface. The as-shipped catalog is **~43 metrics: 35 node + 8 edge**, indexed in `GRAPH_NODE_METRICS` / `GRAPH_EDGE_METRICS`, including community-detection (`louvain_communities`, `greedy_modularity_communities`, `label_propagation_communities`, connected-components families) and edge link-prediction (`jaccard_coefficient`, `adamic_adar_index`, `preferential_attachment`, `resource_allocation_index`, `common_neighbor_centrality`). Community / component wrappers project NetworkX's set-of-sets output onto a per-node integer label, label `0` always the largest group, so the labels feed straight into `partition_variable`. Link-prediction wrappers default to scoring *existing* edges (`ebunch=graph.edges()`), reversing the umbrella plan's original exclusion of that family. Every entry, even shape-compatible ones, gets a wrapper, for a uniform wire format that survives NetworkX return-shape changes. The wrappers live under `graph_features/networkx/` to leave room for a sibling backend.

**`compute_graph_metrics` + construction-time metrics.** `graph_features.compute_graph_metrics(graph, node_metrics=..., edge_metrics=...)` runs named metrics and attaches them as columns. On the high-level classes, `node_graph_metrics` / `edge_graph_metrics` (plus `*_graph_metric_kwargs` / `*_graph_metric_rename`) compute and attach metrics *before* partitioning, so they are immediately usable as `partition_variable` / `sorting_variables`; `compute_graph_metrics()` instance methods do the same for built `HivePlot` / `HivePlotMatrix` instances.

**`to_networkx` + JSON export.** `HivePlot.to_networkx()` (over `converters.nodes_edges_to_networkx()`) round-trips a hive plot back to any of the four NetworkX graph classes (`MultiDiGraph` default); multi-tag `Edges` get a `tag` edge attribute. The companion `to_json()` export and gallery notebook ship alongside.

**`graph_metric_backend` dispatch.** A `graph_metric_backend` parameter on `compute_graph_metrics()`, `HivePlot`, and all five `HivePlotMatrix` surfaces routes metric computation through any NetworkX dispatchable backend (`nx._dispatchable`). Semantics: **strict on names** (validated up front against NetworkX's runtime registry of installed backends; unknown / uninstalled names raise `InvalidGraphMetricBackendError`), **lenient on coverage** (a backend that does not implement a metric raises `NotImplementedError`, which triggers a fallback to default NetworkX with an INFO log line — the codebase's first use of stdlib `logging`, since a correct-but-slower result is not a data-fidelity loss and so is not a `warnings.warn`). A per-metric reserved `"backend"` key in the metric-kwargs dicts overrides the global value, with explicit `None` as the per-metric opt-out; precedence runs per-metric `"backend"` > per-call `graph_metric_backend` > stored construction value (mirroring `graph_directed`). The degree family (`degree` / `in_degree` / `out_degree`) is non-participating (direct structural reads, not dispatchable); an explicit per-metric `"backend"` on them is rejected in the up-front pass. nx-parallel is the CI-tested backend (via the `dev` extra and an `nx_parallel` marker); nx-cugraph is known-good but GPU-only; other dispatchable backends should work but are unverified. The engine is *not* inferable from input type (an `nx.Graph` looks identical whichever backend the user wants), which is why this is a parameter, distinct from the two other "backend" senses in the codebase (see [[graph-features]] "Three senses of backend").

**Conflict-validation layer.** A `@requires_graph_type` decorator attaches a graph-type requirement record to each metric wrapper (fields `requires_directed: Optional[bool]` and `allows_multigraph: bool`, default `True`) and centralizes the runtime raise, replacing ~20 hand-written per-wrapper guard blocks. An up-front validator inspects the whole requested node+edge set before any metric runs and raises one decisive `ValueError` when the set is irreconcilable (e.g. `in_degree` needs directed, `triangles` needs undirected), naming both metrics and the split-into-two-calls resolution, rather than the old one-succeeds-the-next-raises whiplash. `graph_directed` is inferred from the requested metric set when the caller leaves it unset and the set is unambiguous (asking for `triangles` builds an undirected internal graph), with explicit values always winning; building from a `graph` input keeps that graph's own type and never re-infers. A simple-graph build that collapses same-direction duplicate `(from, to)` rows warns by default via `warn_on_parallel_edge_collapse` (set `False` to skip the check on large graphs); reciprocal `(a, b)` / `(b, a)` merges under an undirected build do not warn, since that merge is the definitional meaning of the undirected metric.

## Consequences

- **Enables** the GNN-heterogeneity research direction's structural-sweep pattern directly: a graph from PyTorch Geometric flows into `HivePlot(graph=...)` / `HivePlotMatrix.from_*(graph=...)`, and `node_graph_metrics` computes the sweep variables (degree, centrality, community labels) in one step, replacing the old manual `pd.DataFrame(G.degree, ...) + merge` boilerplate. See [[graph-features]], [[gnn-heterogeneity-hive-plots]].
- **Constrains** the metric subsystem to a flat, metric-name-keyed registry and a NetworkX-typed dispatcher signature. The `graph_features/networkx/` subpackage is the structural seam a future igraph backend slots into (roadmap item 6); the seam exists but is not wired.
- **Locks in** three independent "backend" senses (viz backend = parameter; dataframe engine = type dispatch, no parameter; graph-metric engine = parameter): infer from type when the type carries the intent, take a parameter when it cannot. Recorded so future naming work does not collapse them.
- NetworkX stays an **optional** dependency (`[networkx]` extra, which now also pulls `scipy` for convergence-path metrics). The library never imports nx-parallel; that is a `dev`/test-only dependency.
- Backend **support-tier framing** lives in the `graph_metric_backends.ipynb` gallery notebook (descriptive), not in docstrings. Issue links were intentionally removed from all docstrings.

## Alternatives considered

- **`HivePlot.from_networkx` / `HivePlotMatrix.from_networkx*` classmethods.** Shipped first, then declined: a `from_networkx_*` per matrix `from_*` constructor is a classmethod explosion, and `graph=` keeps `HivePlot` and `HivePlotMatrix` symmetric. The discoverability argument for a named classmethod (a reader skimming a notebook sees the NetworkX entry point) is handled by introductory notebook prose instead.
- **`from_tags(graph=...)`.** Built (Workstream N) then reverted (Workstream O): tags are an `Edges`-level concept with no natural graph-to-tag-dimension extraction, and a round-trip had a single-tag corner-case hole. The cleaner end state is a graph-less `from_tags` whose docstring points graph-holding users at `from_partition` / `from_variable_sweep` or manual conversion.
- **Lossless graph round-trip / stashing hive-plot state on the graph.** Rejected per the issue's "deliberately avoids tracking" philosophy; `to_networkx` rebuilds fresh and config is re-specified on import.
- **`nx.config.backend_priority` context for dispatch.** Rejected: its fallback is silent and automatic, defeating strict-on-names, the catchable `NotImplementedError`, and observability.
- **Auto-selecting a faster backend when `graph_metric_backend` is unset.** Rejected as silent install-dependent magic; NetworkX's `NETWORKX_BACKEND_PRIORITY` env var already serves that want explicitly.

## Deferred / Declined

**Deferred (may revisit):**

- **`from_variable_sweep` redundant-arg validator gap.** `from_variable_sweep` silently accepts both `partition_variable` and `partition_variables_list` (and likewise the `sorting_variables` pair): the list branch wins and the singleton is silently ignored in 2D-grid mode rather than raising. A genuine validator gap, kept live; the fix targets `from_variable_sweep`'s validator directly and has no dependency on the torn-out `from_networkx*` surface.
- **The four future-igraph questions**, tied to the live igraph-backend roadmap item (roadmap item 6): (1) gap-metric strategy (lean: `NotImplementedError` naming the gap and NetworkX equivalent, not silent fallback); (2) leidenalg packaging (lean: separate `[igraph-leiden]` extra); (3) GPL license posture; (4) igraph notebook approach (lean: parity notebook). **GPL posture is the gating sub-question:** both igraph and leidenalg are GPL-licensed and hiveplotlib is BSD-3. Optional extras don't relicense the distribution, but the license-boundary call (and a docs note) must land before any igraph code, and #1/#2/#4 are downstream of it.

**Declined (with revival trigger):**

- **`HivePlot.from_networkx` classmethod** — declined in favor of `graph=` (see Decision and Alternatives).
- **`GraphMetricsSpec` dataclass consolidation.** Consolidating the ~8-kwarg graph-metric surface (`node_graph_metrics`, `edge_graph_metrics`, the `*_kwargs` / `*_rename` companions, `graph_metric_backend`, plus the adjacent `graph_directed` / `graph_multigraph` pair) into one typed config object was decided against: the surface is fine in practice, the dominant call is `node_graph_metrics="degree"`, and a spec object turns that one-liner into ceremony. **Revival trigger:** either (a) igraph (or another backend) blows the metric-kwarg block past readable on the matrix `from_*` signatures, or (b) users ask for a reusable typed config. If revived, add `GraphMetricsSpec` and deprecate the redundant params over a two-version window (per the `hive_plot_n_axes` 0.26→0.28 precedent), keeping `node_graph_metrics` / `edge_graph_metrics` as a shorthand alongside the spec so the simple case stays a one-liner.

## References

- Working plans (verbose history): [[i-want-to-plan-optimized-hoare]] (umbrella — bidirectional conversion, `graph_features` package, the `graph=` consolidation, `from_tags` revert), [[networkx-metric-expansion-and-backend-refactor]] (metric catalog expansion + `graph_features/networkx/` subpackage + igraph roadmap), [[graph-metric-backend-dispatch]] (`graph_metric_backend`), [[graph-metric-conflict-validation]] (`@requires_graph_type`, up-front validator, `graph_directed` inference, parallel-edge-collapse warning). Plan paths post-archive: `wiki/wiki/plans/archived/<topic>.md`.
- Wiki: [[hiveplotlib]] (entity hub), [[graph-features]] (the metric subsystem, graph-type handling, backend dispatch, three-senses-of-backend triangle).
- Issue / release: GitLab #46; `CHANGELOG.rst` v0.28.0.
- Roadmap: item 6, "Optional `igraph` backend for `compute_graph_metrics`" (`docs/source/roadmap.rst`).

## See Also

- [[hiveplotlib]]
- [[graph-features]]
- [[gnn-heterogeneity-hive-plots]]
