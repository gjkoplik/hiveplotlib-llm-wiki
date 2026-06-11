---
title: Graph Features
type: concept
created: 2026-05-31
updated: 2026-06-11
sources: [hiveplotlib-python-repo]
tags: [hiveplotlib, graph-metrics, networkx, node-assignment]
---

# Graph Features

**Graph features** (a.k.a. graph metrics) are the structural quantities computed from a network's topology: degree, centrality, clustering, community membership, and so on. In a [[hive-plot|hive plot]] they are the natural raw material for [[node-assignment]]: the variable you partition nodes onto axes by, and the variable you sort along each axis by. [[hiveplotlib]] grew a dedicated `graph_features` package in v0.28.0 that wraps these metrics from `networkx` and attaches them as columns to a `NodeCollection` / `Edges`.

## Motivation

Before v0.28, putting a structural metric on a hive plot axis meant computing it in `networkx`, coercing the result into a DataFrame, and merging it onto the node data by hand:

```python
deg = pd.DataFrame(G.degree, columns=["unique_id", "degree"])
nodes.data = nodes.data.merge(deg, on="unique_id")
```

This pattern was repeated, error-prone, and easy to get wrong for edge-level metrics. The `graph_features` package collapses it to a string reference.

## API surface

- `GRAPH_NODE_METRICS` and `GRAPH_EDGE_METRICS` — master dicts indexing every supported wrapper by string name (35 node metrics, 8 edge metrics).
- `compute_graph_metrics(graph, node_metrics=..., edge_metrics=...)` — runs one or more named metrics on a `networkx` graph and attaches the results as columns to a `NodeCollection` / `Edges`. Accepts a single name or a sequence; supports per-metric kwargs and a rename map for column-name collisions.
- On the high-level classes, `node_graph_metrics` / `edge_graph_metrics` (plus `*_graph_metric_kwargs` / `*_graph_metric_rename`) are accepted at `HivePlot` / `HivePlotMatrix` construction. Metrics are computed *before* partitioning runs, so the new columns are immediately referenceable as `partition_variable` / `sorting_variables`. `compute_graph_metrics()` mirrors this for already-built instances.

## Catalog (by family)

**Node metrics (35).**
- Degree / degree centrality: `degree`, `in_degree`, `out_degree`, `degree_centrality`, `in_degree_centrality`, `out_degree_centrality`, `average_neighbor_degree`.
- Distance / shortest-path centrality: `betweenness_centrality`, `closeness_centrality`, `harmonic_centrality`, `load_centrality`, `eccentricity`.
- Link-analysis: `eigenvector_centrality`, `eigenvector_centrality_numpy`, `pagerank`, `hits_hubs`, `hits_authorities`.
- Clustering / cores: `clustering`, `square_clustering`, `core_number`, `triangles`, `onion_layers`.
- Structural position: `constraint`, `effective_size`, `closeness_vitality`, `reciprocity`, `articulation_points`, `isolates`.
- DAG: `topological_generations`.
- Community detection / connected components: `greedy_modularity_communities`, `louvain_communities`, `label_propagation_communities`, `connected_components`, `strongly_connected_components`, `weakly_connected_components`.

**Edge metrics (8).**
- Edge centralities: `edge_betweenness_centrality`, `edge_load_centrality`, `bridges` (per-edge boolean).
- Link-prediction: `jaccard_coefficient`, `adamic_adar_index`, `preferential_attachment`, `resource_allocation_index`, `common_neighbor_centrality`. These default to scoring *existing* edges rather than `networkx`'s usual non-edges default.

## Design notes

- **Stable community labels.** The community-detection and connected-component wrappers project `networkx`'s set-of-sets partition output onto a per-node integer label, with label `0` always assigned to the largest community / component (ties broken by smallest node id). This makes a "color the giant community" workflow deterministic across runs, and the resulting integer labels feed straight into a `partition_variable` (an integer-partition `KeyError` was fixed in the same release so these labels build cleanly).
- **Localized to NetworkX, by design.** The wrappers live under a `graph_features/networkx/` subpackage rather than a flat module. This is deliberate scaffolding for future graph-feature backends: the roadmap calls for an optional `igraph` backend (which ships several fast community-detection algorithms) to slot in alongside the `networkx` one. Note this wrapper-library axis is distinct from the dispatch-engine axis described under "Backend dispatch" below; see "Three senses of backend" for why one gets a parameter and the other will not.
- **scipy dependency.** The `[networkx]` extra now also pulls in `scipy`, which several wrappers depend on for convergence paths (e.g. `eigenvector_centrality`).

## Graph-type handling

The single internal `networkx` graph that backs metric computation has a type (directed vs. undirected, simple vs. multi), and getting that type wrong is the main failure mode. Two v0.28 refinements smooth this over so the common cases work without the caller restating the type.

- **Directedness inferred from the metric set.** When you build a `HivePlot` / `HivePlotMatrix` from `nodes` / `edges` and leave `graph_directed` unset, the internal graph's directedness is inferred from the metrics you request. Asking for an undirected-only metric like `triangles` builds an undirected internal graph without your touching the flag; a directed-only metric like `in_degree` builds a directed one. With no metric that cares about directedness, it falls back to `True`, matching the `(from, to)` semantics of `Edges`. A request that is itself contradictory (one metric needs a directed graph, another an undirected one) is not silently resolved: the up-front conflict validator raises a `ValueError`, and the fix is to split it into two constructions, one per `graph_directed` value. Building from a `graph` input keeps that graph's own type and is never re-inferred, so a mismatched metric raises rather than quietly re-typing the graph you passed. The `compute_graph_metrics()` function itself never infers; it validates the requested set against whatever graph you hand it.
- **Parallel-edge collapse is flagged.** Building a simple (non-multi) internal graph from edge data with same-direction duplicate `(from, to)` rows collapses those rows last-write-wins, so a metric reading an edge `weight` sees only the surviving row. The high-level classes now emit a `UserWarning` when this happens (`warn_on_parallel_edge_collapse=True` by default; set `False` to skip the check on large graphs). Reciprocal `(a, b)` / `(b, a)` rows that merge only because an undirected build treats the pair symmetrically do not warn, since that merge is the definitional meaning of the undirected metric. To keep both edges, pass `graph_multigraph=True`; this only helps for metrics that accept multigraphs, since simple-graph-only metrics like `onion_layers` reject a multigraph, so pre-aggregate the reciprocal weights into a single edge for those.

## Backend dispatch (v0.28)

NetworkX has a backend dispatch system: drop-in acceleration engines (nx-parallel for CPU parallelism, nx-cugraph for GPU, graphblas-algorithms, and any other `nx._dispatchable` backend) that reimplement networkx algorithms behind the same function signatures. v0.28 exposes this through a `graph_metric_backend` parameter on `compute_graph_metrics()`, `HivePlot`, and all five `HivePlotMatrix` surfaces (constructor, `compute_graph_metrics()`, and the three `from_*` classmethods), so large-graph users can route the metric computation that feeds [[node-assignment]] through a faster engine with one argument.

Design points worth remembering:

- **Strict on names, lenient on coverage.** Backend names are validated up front against networkx's runtime registry of installed backends, before any metric runs; unknown or uninstalled names raise `InvalidGraphMetricBackendError` (which warns that registry names differ from pip names: nx-parallel registers as `"parallel"`, nx-cugraph as `"cugraph"`). But a valid backend that simply does not implement a given metric is not an error: the `NotImplementedError` from dispatch triggers a fallback to default networkx, since the result is identical, just slower.
- **First stdlib logging in the codebase.** The fallback emits an INFO log line (metric, requested backend, ran on default networkx) and successful dispatch emits DEBUG; module-level `logging.getLogger(__name__)`, no handler configuration in library code, so the library stays silent by default. Logging rather than `warnings.warn` because a correct-but-slower computation is not a data-fidelity loss.
- **Per-metric reserved key.** `"backend"` inside `node_metric_kwargs` / `edge_metric_kwargs` is a reserved key: the seam pops it (sentinel-default pop, never forwarded verbatim) and an explicit `"backend": None` is the documented per-metric opt-out back to default networkx under a global backend. Precedence is three-level: per-metric `"backend"` > per-call `graph_metric_backend` > stored construction value (the construction value is stashed and reused by later calls, mirroring `graph_directed`; per-call `graph_metric_backend="networkx"` is the documented per-call escape hatch back to the default engine).
- **The degree family does not participate.** `degree` / `in_degree` / `out_degree` are direct structural reads (method calls, not dispatchable functions); a global backend silently skips them, and an explicit per-metric `"backend"` entry on them is rejected in the up-front validation pass with an actionable message rather than a mid-loop `TypeError`.
- **Support tiers:** nx-parallel is tested in CI (via the `dev` extra and an `nx_parallel` pytest marker); nx-cugraph is known-good but GPU-only and untested in CI; other dispatchable backends should work but are unverified.

The Graph Metric Backends gallery notebook demonstrates the parameter route, the per-metric override, fallback visibility via `logging.basicConfig(level=logging.INFO)`, and networkx's own `NETWORKX_BACKEND_PRIORITY` env-var route.

### Three senses of "backend"

Hiveplotlib's design now distinguishes three "backend" concepts, settled across two naming audits. Recording the triangle here so future naming work does not collapse them:

1. **Viz backends** (matplotlib, bokeh, plotly, ...): a `backend` parameter, user-chosen, because a rendering target is a preference nothing can infer. Shipped.
2. **Dataframe engines** (pandas, polars, ... via narwhals): no parameter at all; the engine is dispatched on input type, since a polars frame *is* the declaration of intent (the scaling-large-networks naming audit removed a proposed `backend="dask"` on these grounds). Settled design; the narwhals layer is in-flight work, not yet shipped.
3. **Graph-metric dispatch engines** (nx-parallel, nx-cugraph, ...): a parameter (`graph_metric_backend`), because the engine is *not* inferable from input type; an `nx.Graph` looks identical whether the user wants nx-parallel or default networkx. Shipped in v0.28.

The rule the triangle encodes: infer from type when the type carries the intent; take a parameter when it cannot. The deliberately long name `graph_metric_backend` keeps sense 3 unconfusable with sense 1, which coexists as `backend` in the same `HivePlot.__init__` signature. The future igraph wrapper-library work is yet another axis (which library's algorithms hiveplotlib wraps); per a prior plan resolution it will dispatch on input graph type (sense-2 style, an `igraph.Graph` input declares the intent), not via a parameter.

## Relevance to research

The graph-feature catalog is exactly the menu of **structural sweep variables** the [[gnn-heterogeneity-hive-plots|GNN heterogeneity proposal]] wants to drive a [[hive-plot-matrix|HivePlotMatrix]] with. Degree bins, centrality, community membership, core number, and local clustering are all one string reference away now. It also lowers the cost of the [[karate-club-walkthrough]]-style "partition by community, sort by degree" pattern to a single constructor call.

## See Also

- [[hiveplotlib]] — Implementation
- [[node-assignment]] — How these metrics map nodes onto axes
- [[hive-plot-matrix]] — Sweeping over structural metrics across a grid
- [[gnn-heterogeneity-hive-plots]] — Structural metrics as GNN diagnostic sweep variables
- [[structural-heterogeneity]] — The topological variation these metrics quantify
