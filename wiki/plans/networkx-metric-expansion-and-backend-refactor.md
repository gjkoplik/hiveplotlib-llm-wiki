# Plan: NetworkX metric expansion plus structural refactor for future igraph backend

## Goal

Close out the NetworkX integration sprint by (1) restructuring `hiveplotlib.graph_features` so the NetworkX-coupled wrappers live in a `networkx/` subpackage (sibling backends can drop in later), (2) adding ~26 high-value NetworkX metric wrappers across two tiers, (3) seeding a roadmap entry for a future igraph backend, and (4) refreshing the example notebook and CHANGELOG. The public surface (`compute_graph_metrics`, `GRAPH_NODE_METRICS`, `GRAPH_EDGE_METRICS`, top-level imports from `hiveplotlib.graph_features`) stays unchanged; the user-visible value is "the metric subsystem now covers community detection, harmonic centrality, eccentricity, link prediction, and a dozen other staples a real graph analyst expects, without bloating the surface."

## Prior ADRs / design docs

- **None — no ADRs exist yet in the wiki.** The `wiki/wiki/adr/` directory has not been created; this is the first plan to surface during a pre-task search. Other relevant wiki pages (informational only, no binding decisions):
  - `wiki/wiki/entities/hiveplotlib.md` — current entity status page; lists the NetworkX converter as a key feature, will need a "key APIs" update once this work ships (post-task pass for the Research Liaison).
  - `wiki/wiki/sources/hiveplotlib-python.md` — source summary noting "minimal base deps: matplotlib, numpy, pandas only; optional backends as extras." Confirms NetworkX-as-optional is the established convention; this plan keeps that boundary intact.
  - `wiki/wiki/analyses/gnn-heterogeneity-hive-plots.md` and `wiki/wiki/analyses/karate-club-walkthrough.md` — research directions that will benefit from richer metric coverage (community-detection wrappers in particular). Not constraining, but worth knowing the audience.

Because the structural-refactor decision (move NetworkX-coupled code into a `networkx/` subpackage to make room for sibling backends) is the kind of architectural call ADRs are designed to preserve, the QA Engineer should flag this plan as **ADR-promotion eligible** once the workstreams ship. Two natural ADR seeds: one on the `graph_features.<backend>/` layout, one on the NetworkX-as-optional dep posture.

## Patterns this replaces

The refactor in Workstream A is the only place this plan replaces existing patterns. All Tier 1 and Tier 2 additions are net new wrappers.

Replace-and-sweep targets (grepped against the current branch):

- **Module path `hiveplotlib.graph_features.node_metrics`** referenced at:
  - `src/hiveplotlib/graph_features/__init__.py:5` (docstring), `:13` (extend instructions), `:38` (import line)
  - `src/hiveplotlib/graph_features/node_metrics.py:5` (self-reference docstring)
  - `docs/source/autodoc/hive_plots/graph_features.rst` (autodoc include, needs verification — Workstream A's "done when" includes a build of `make docs` to catch any broken references)
- **Module path `hiveplotlib.graph_features.edge_metrics`** referenced at:
  - `src/hiveplotlib/graph_features/__init__.py:5` (docstring), `:13` (extend instructions), `:34` (import line)
  - `src/hiveplotlib/graph_features/edge_metrics.py:5` (self-reference docstring)
  - `docs/source/autodoc/hive_plots/graph_features.rst` (autodoc include, same check as above)
- **Files to move:**
  - `src/hiveplotlib/graph_features/node_metrics.py` → `src/hiveplotlib/graph_features/networkx/node_metrics.py`
  - `src/hiveplotlib/graph_features/edge_metrics.py` → `src/hiveplotlib/graph_features/networkx/edge_metrics.py`

Replacement strategy:

- Inside `src/hiveplotlib/graph_features/__init__.py`, change the two `from hiveplotlib.graph_features.{node,edge}_metrics import ...` lines to `from hiveplotlib.graph_features.networkx.{node,edge}_metrics import ...`. The top-level reexports stay identical (`from hiveplotlib.graph_features import degree, ...` keeps working).
- Add a new `src/hiveplotlib/graph_features/networkx/__init__.py` exporting the wrappers so an advanced user can `from hiveplotlib.graph_features.networkx import degree` if they want a backend-explicit import path. Not advertised in the changelog (internal seam, not a headline).
- Update the two extend-instructions docstrings in `graph_features/__init__.py` to point at the new path (one-line edit each).
- Sweep `tests/` and `examples/` for any direct imports of `hiveplotlib.graph_features.node_metrics` / `edge_metrics` — currently zero such direct imports exist (everything reaches through `hiveplotlib.graph_features`), but the QA pass post-execution should confirm.

If any documentation file or notebook references `hiveplotlib.graph_features.node_metrics` directly, decide on the case as it surfaces: if it's a path the user is supposed to know, fix it; if it's incidental, fix it. There is no holdout.

## Default justifications

No new user-facing defaults are introduced by this plan; every new wrapper inherits the same shape as existing wrappers (string key in the registry, optional `**kwargs` forwarded to the underlying NetworkX function, restriction errors when the graph type doesn't match). The dispatcher signature stays the same. Justifications retained from the existing design:

- `node_metrics=None`, `edge_metrics=None`: a user calling `compute_graph_metrics` who hasn't specified metrics is exploring the API surface; defaulting to `None` (no work, no surprise columns) matches that workflow. (Existing default; unchanged.)
- `source_attribute_name="_hiveplotlib_source"`: only relevant for multigraph edge metrics, and the value matches what `nodes_edges_to_networkx()` writes by default, so the round-trip works without the user setting anything. (Existing default; unchanged.)

The optional `@requires_graph_type(directed=..., multigraph=...)` decorator helper (Workstream C optional sub-item) does not introduce a user-facing default; it's an internal DRY tool. Defaults inside link-prediction wrappers (`ebunch=G.edges()` when omitted) match the user workflow "I want this metric over every edge in my graph" — that's the canonical NetworkX entry point and consistent with how edge-betweenness already operates.

## Naming audit

Internal package paths (`graph_features/networkx/...`) are out of scope per template. The user-facing names introduced by this plan are **the new metric string keys**. Every key mirrors the matching NetworkX function name verbatim. This is the right choice because NetworkX is the dominant adjacent ecosystem for the metric vocabulary and users will reach for the NetworkX name when in doubt.

New node-metric keys (Tier 1, 11):

- `degree_centrality`, `in_degree_centrality`, `out_degree_centrality`, `harmonic_centrality`, `average_neighbor_degree`, `onion_layers`, `square_clustering`, `constraint`, `effective_size`, `closeness_vitality`, `load_centrality`.
- Vs. user vocab: **all match `networkx.<name>` verbatim, OK.** The `_centrality`-suffixed names sit alongside the existing `betweenness_centrality` / `closeness_centrality` / `eigenvector_centrality` family; consistency is preserved.

New node-metric keys (Tier 2, 13 wrappers across families):

- `eccentricity`, `eigenvector_centrality_numpy`, `hits_hubs`, `hits_authorities`, `reciprocity`, `articulation_points`, `isolates`, `topological_generations`, `greedy_modularity_communities`, `louvain_communities`, `label_propagation_communities`, `connected_components`, `strongly_connected_components`, `weakly_connected_components`.
- Vs. user vocab: **mostly match `networkx.<name>` verbatim, with two flagged amendments below.**
- `hits_hubs` / `hits_authorities`: NetworkX exposes one `nx.hits(G)` function returning a 2-tuple `(hubs, authorities)`. Splitting into two registry keys (each returning one dict) lets the dispatcher's "name → dict" contract hold without forcing the user to think about tuples. The `hits_` prefix preserves the NetworkX root word; the `_hubs` / `_authorities` suffixes match the names NetworkX uses internally for the two output dicts. OK.
- `articulation_points` / `isolates` / `connected_components` family: in NetworkX these return *sets* or *generators of sets*, not per-node dicts. Wrapping them to `dict[node, bool]` (membership flag) or `dict[node, int]` (component label) is necessary to fit the registry shape. The string keys stay verbatim. OK.
- `topological_generations`: NetworkX returns a generator of node sets (one per DAG layer). The wrapper assigns each node its layer index as an int. Name stays verbatim. OK.

New edge-metric keys (Tier 2, 6):

- `bridges`, `jaccard_coefficient`, `adamic_adar_index`, `preferential_attachment`, `resource_allocation_index`, `common_neighbor_centrality`.
- Vs. user vocab: **all match `networkx.<name>` verbatim, OK.**

Internal helpers (not user-facing, naming-audit-exempt but called out for the implementing engineer):

- `_partition_to_node_labels(partition)` — set-of-sets to `{node: int_label}` adapter.
- `_set_to_indicator(elements, all_keys)` — set of elements to `{key: bool}` adapter, used by `articulation_points`, `bridges`, `isolates`.
- `_make_link_prediction_wrapper(nx_func_name)` — factory returning a wrapper that defaults `ebunch=G.edges()` and materializes the generator to `dict[(u, v), value]`.
- `@requires_graph_type(directed=..., multigraph=...)` — optional DRY decorator. **Not mandatory.** The Code Engineer implementing Workstream B should decide based on how the boilerplate count actually looks once Tier 1 lands; if it's already noisy, fold in this decorator. The decorator name reads as a guard clause, consistent with how the existing wrappers phrase their restrictions in error messages.

No prose-only terms are introduced.

## API usage examples

### Proposed (from planner / Orchestrator)

```python
# Example 1: community-detection on Karate Club, used as the partition variable.
# Two-step pattern: compute_graph_metrics writes the column first, then HivePlot can resolve it.
import networkx as nx
from hiveplotlib import HivePlot
from hiveplotlib.converters import networkx_to_nodes_edges
from hiveplotlib.graph_features import compute_graph_metrics

G = nx.karate_club_graph()
nodes, edges = networkx_to_nodes_edges(G)
nodes, _ = compute_graph_metrics(
    G,
    node_metrics=["louvain_communities", "degree"],
    node_metric_kwargs={"louvain_communities": {"seed": 0}},
    target_nodes=nodes,
    target_edges=edges,
)
hp = HivePlot(
    nodes=nodes,
    edges=edges,
    partition_variable="louvain_communities",
    sorting_variables="degree",
)
fig, ax = hp.plot()
```

```python
# Example 2: link-prediction edge metric, used to color edges
hp = HivePlot.from_networkx(
    G,
    partition_variable="club",
    sorting_variables="degree",
    node_graph_metrics="degree",
    edge_graph_metrics="jaccard_coefficient",
)
hp.update_edge_plotting_keyword_arguments(
    array="jaccard_coefficient", cmap="plasma",
)
```

```python
# Example 3: post-hoc compute of multiple Tier 1 metrics on an existing HivePlot
hp.compute_graph_metrics(
    node_graph_metrics=[
        "harmonic_centrality",
        "average_neighbor_degree",
        "square_clustering",
    ],
)
hp.nodes.data.head()
```

```python
# Example 4: structural-position node metric (articulation_points as a bool)
hp.compute_graph_metrics(
    node_graph_metrics="articulation_points",
    graph_directed=False,  # articulation points defined on undirected graphs
)
hp.nodes.data[["unique_id", "articulation_points"]].head()
# articulation_points is a bool column ready to use for node styling
```

```python
# Example 5: pre-HivePlot pipeline using the lower-level compute_graph_metrics()
from hiveplotlib.converters import networkx_to_nodes_edges
from hiveplotlib.graph_features import compute_graph_metrics

nodes, edges = networkx_to_nodes_edges(G)
nodes, _ = compute_graph_metrics(
    G,
    node_metrics=["louvain_communities", "harmonic_centrality"],
    node_metric_kwargs={"louvain_communities": {"seed": 0}},
    target_nodes=nodes,
    target_edges=edges,
)
# nodes.data now has 'louvain_communities' (int label) and 'harmonic_centrality' (float)
```

### API Critic's take (planning mode)

Walked Examples 1-5 as a user, plus the seven framings in the prompt. The plan is mostly right but I have one strong rename push, one strong docstring push, and a handful of taste calls. The verbatim-NetworkX-name principle in the naming audit is the correct default, but it shouldn't override the dispatcher's own contract when the two conflict.

The friction list:

- **[high] Boolean-typed columns should use `is_*` names, not the NetworkX plural-set names. Rename `articulation_points` → `is_articulation_point`, `isolates` → `is_isolate`, and the edge-side `bridges` → `is_bridge`.** The naming audit defends the verbatim names by appealing to "NetworkX is the dominant adjacent ecosystem." That logic holds when the wrapper preserves the NetworkX *contract* (a centrality wrapper returns a float per node, just like NetworkX does). It breaks here, because the dispatcher's contract is one column per registry key and the column value is per-node. NetworkX's `nx.articulation_points(G)` returns an iterator of nodes; our wrapper returns `dict[node, bool]`. The column name `articulation_points` reads as "a list of articulation points lives in this cell," which is exactly wrong. `is_articulation_point` reads correctly as a boolean attribute. Same logic for `isolates` (the column tells you whether *this* node is an isolate, not "the isolates") and `bridges` (whether *this* edge is a bridge). Example 4 in the plan literally has to add a comment, `# articulation_points is a bool column ready to use for node styling`, to explain the column's shape; that comment is the smell. With `is_articulation_point` it would not need the comment. Cost: 3 string-key renames in Workstream C1/C2 plus a one-line note in the naming audit explaining that the "match NetworkX" rule yields to the dispatcher's per-element semantics when NetworkX returns a set/iterator and we project to per-element. Worth it.

- **[high] The link-prediction docstrings must warn that the values are link-likelihood scores being applied to existing edges, not non-edges.** NetworkX's `nx.jaccard_coefficient(G)` is documented as a non-edge link predictor; users who know NetworkX will read `edge_graph_metrics="jaccard_coefficient"` and reasonably wonder why we're computing a non-edge metric on edges. The hive-plot reinterpretation (use the score as an edge-style signal over existing edges) is fine and useful, but it needs to be stated. Concretely, the `_make_link_prediction_wrapper` factory should generate a docstring that says something like: "By default this wrapper applies NetworkX's link-prediction score to every existing edge in `graph` (`ebunch=G.edges()`), letting you size or color edges by the predictor. Pass `ebunch=...` explicitly to score a different edge set." Without that, a careful NetworkX user will assume we either misuse the function or that the column means something it doesn't. Cost: factory generates a templated docstring; no code-shape change.

- **[medium] Keep `_centrality`-suffix names verbatim, but rename the community/component families to singular per-node forms: `louvain_communities` → `louvain_community`, `greedy_modularity_communities` → `greedy_modularity_community`, `label_propagation_communities` → `label_propagation_community`, `connected_components` → `connected_component`, `strongly_connected_components` → `strongly_connected_component`, `weakly_connected_components` → `weakly_connected_component`.** Same logic as the boolean rename, weaker confidence because the convention is muddier. NetworkX's `nx.louvain_communities(G)` returns `list[set[node]]` ("the communities, plural"); our column contains *this node's* community label. Singular reads correctly, plural reads as "the cell holds the list of all communities." The user vocab argument cuts both ways here: a NetworkX-fluent user will type `louvain_community` when they want one-label-per-node and `louvain_communities` when they want the partition-of-the-graph object; we expose the former, so we should name it that way. The mixed precedent in the existing registry (`clustering` singular, `triangles` plural-but-it-means-a-count, `core_number` singular) doesn't bind us. Recommend renaming. If the team decides to keep plural for ecosystem-match, document the per-node semantics inline at the registry entry. Medium not high because there is a genuine taste call here that reasonable people will disagree on.

- **[medium] Commit to a documented label assignment order for the community/component wrappers and state it in their docstrings.** Walk 5 in the prompt is right: when `connected_components` returns 50 groups, the int labels are otherwise meaningless. NetworkX's `connected_components(G)` doesn't guarantee an order across versions. The adapter helper `_partition_to_node_labels(partition)` should sort the partition before assigning labels, and "sorted by" should be documented. Recommend: sort by component size descending, ties broken by smallest node id. This makes label 0 the giant component, label 1 the second-largest, and so on, which matches how a user will reach for the column ("color the big component differently"). Without this, a re-run of the same code can produce label-flipped output, which breaks the precompute-and-reuse story below. Cost: ~5 lines in the helper plus one sentence per wrapper docstring. Cheap.

- **[medium] The HITS split into `hits_hubs` / `hits_authorities` is fine, but cache the underlying `nx.hits(G)` call to avoid recomputing when both are requested.** The two-key shape is the right user-facing surface; tuple-of-columns returns would force a registry-shape change that isn't worth it for one metric pair. But `nx.hits(G)` is a single iterative computation that produces both dicts at once. The naive implementation (each wrapper calls `nx.hits(G)` independently) doubles the work whenever the user asks for both. The dispatcher already iterates the requested metrics in order; the wrappers could memoize on the graph identity. Concretely: a tiny module-private cache keyed by `id(graph)` and cleared at the end of `compute_graph_metrics`, or a small "pair-aware" path where the two HITS wrappers share state. The cleanest version: the implementer can make `hits_hubs` and `hits_authorities` thin closures over a `_hits_cached(graph)` helper that uses `functools.lru_cache` (with `id(graph)` as the key) and is invalidated per-`compute_graph_metrics` call. Without this, "I want hubs and authorities" pays double. Worth-discussing because it's an internal-only concern; doesn't affect the surface.

- **[medium] Example 1 is incoherent as written and should be fixed in the plan.** The snippet passes `partition_variable="louvain_communities"` but uses `HivePlot.from_networkx`, which computes graph metrics during init, not before. The `partition_variable` lookup happens against the node DataFrame before the metric column is attached, so the partition string won't resolve. The notebook flow already shows the right pattern in the "Using a Computed Metric as a Partition Variable" cell: convert via `networkx_to_nodes_edges`, then `compute_graph_metrics`, then build the `HivePlot`. Example 1 in the plan should be rewritten in that two-step shape, otherwise the highest-value new capability (community as partition) demos as broken. Replacement:

  ```python
  import networkx as nx
  from hiveplotlib import HivePlot
  from hiveplotlib.converters import networkx_to_nodes_edges
  from hiveplotlib.graph_features import compute_graph_metrics

  G = nx.karate_club_graph()
  nodes, edges = networkx_to_nodes_edges(G)
  nodes, _ = compute_graph_metrics(
      G,
      node_metrics=["louvain_community", "degree"],
      node_metric_kwargs={"louvain_community": {"seed": 0}},
      target_nodes=nodes,
      target_edges=edges,
  )
  hp = HivePlot(
      nodes=nodes,
      edges=edges,
      partition_variable="louvain_community",
      sorting_variables="degree",
  )
  fig, ax = hp.plot()
  ```

  This is also worth pinning in the notebook (Workstream E) as the canonical pattern; integer community labels are categorical-like and skip the `create_partition_variable` discretization step that continuous metrics need.

- **[medium] Workstream E should include an explicit "precompute once, reuse across plots" cell.** Walk 6 in the prompt: community detection is expensive (Louvain especially on bigger graphs), and the existing notebook quietly assumes per-plot recomputation. With the Tier 2 additions, recomputing is genuinely wasteful. One small notebook cell pair showing `nodes, edges = compute_graph_metrics(G, node_metrics=[...], target_nodes=nodes, target_edges=edges)` followed by *two* `HivePlot(nodes=nodes, edges=edges, ...)` calls (e.g. one partitioning on community, one sorting on harmonic centrality) is enough. This is a one-paragraph addition in Workstream E's "small focused cells" list; flagging here so the notebook author picks it up.

- **[low] Forward-looking igraph dispatch: the right shape is `isinstance` dispatch on the `graph` argument, no `backend=` kwarg.** When igraph eventually lands, `compute_graph_metrics(my_ig_graph, node_metrics=["louvain_community"])` should Just Work via `isinstance(graph, ig.Graph)`. A `backend="igraph"` kwarg would force users to specify something the type system already knows. The plan's "structure for isinstance dispatch but don't wire it" recommendation in open question #1 is right; option (a) in that discussion (widen the `graph` parameter to `Union[nx.Graph, ig.Graph]` plus a small `_dispatch_backend(graph)` helper) is the ergonomic path. Workstream A's move into `graph_features/networkx/` leaves room for a sibling `graph_features/igraph/` and a flat top-level registry that resolves `name + graph_type → wrapper`. The one thing to confirm before shipping Workstream A: the `GRAPH_NODE_METRICS` / `GRAPH_EDGE_METRICS` dicts should stay metric-keyed (not backend-keyed); the backend split lives one level down inside each wrapper or in a sibling registry the dispatcher consults via type. Low because this is forward-looking and the plan is explicit that no wiring happens this sprint.

**Recurring theme.** The naming audit applied "match NetworkX verbatim" as a near-absolute rule. That rule is correct when the wrapper preserves NetworkX's contract (centrality returns float per node, both here and there). It is wrong when our adapter changes the shape (NetworkX returns a *set* of articulation points; we return a *bool per node*). Two strong recommendations and one medium recommendation above flow from that single mis-application. Suggest amending the naming audit's "OK" lines to add a clause: "when the wrapper preserves the per-element shape; rename to a per-element form when we project a set/partition to a per-element column."

**Verdict:** ship-with-minor-tweaks. The structural moves (Workstream A), the metric coverage choices, and the adapter-helper design are all sound. The string-key renames for boolean projections and the link-prediction docstring template are the only items I'd block on; the community/component singulars and the label-order commitment are strong recommendations; the HITS caching and the Example 1 fix are easy wins; the precompute-reuse notebook cell is a low-cost notebook ask.

### User resolutions (planning mode)

The user walked the critic's friction list and recorded the following decisions:

- **[item 1, declined]** Keep `articulation_points` / `isolates` / `bridges` rather than `is_articulation_point` / `is_isolate` / `is_bridge`. Reasoning: at the call site, `node_metrics=["articulation_points"]` reads naturally as the metric's name, parallel to `node_metrics=["pagerank"]`. The per-element-bool nature is documented in each wrapper's docstring; this preserves the "match NetworkX name" convention for searchability.
- **[item 2, accepted]** `_make_link_prediction_wrapper` generates a templated docstring on each registered wrapper warning the values are link-likelihood scores applied to existing edges (NetworkX intends these for non-edges by default). Reflected in Workstream C2.
- **[item 3, accepted]** `_partition_to_node_labels` commits to sorting partitions by size descending, ties broken by smallest node id. Each community/component wrapper docstring documents this ordering. Reflected in Workstream C1.
- **[item 4, declined — Option A]** Skip the HITS caching layer. The two `hits_*` wrappers each call `nx.hits(G)` independently; their docstrings note that requesting both runs the iteration twice. Reasoning: HITS converges fast on the graph sizes hive plots typically visualize, and the cross-call graph rebuild in `HivePlot` is a much larger source of duplicate work that we're not optimizing either. Simpler code wins. Reflected in Workstream C1.
- **[item 5, accepted]** Example 1 rewritten in the two-step `networkx_to_nodes_edges` → `compute_graph_metrics` → `HivePlot` pattern. Plural metric name (`louvain_communities`) preserved per item-8. Reflected in "API usage examples" above.
- **[item 6, accepted with dataset swap]** Workstream E uses Karate Club for the headline "Community detection as a partition variable" cell (story: Louvain recovers the historical `club` split on a familiar dataset). Adds `nx.les_miserables_graph()` for the "precompute once, reuse across plots" cell (story: Louvain reveals plot threads in character co-appearance, where communities are not pre-labeled). Reflected in Workstream E.
- **[item 7, confirmed]** Forward-looking igraph dispatch confirmed as `isinstance`-based on the `graph` argument, no `backend=` kwarg. Aligned with open question #1's planner recommendation; no plan edit needed.
- **[item 8, declined]** Keep plural community/component family names (`louvain_communities`, `connected_components`, etc.). Reasoning: as sibling backends (igraph) come online, strict "match NetworkX verbatim" loses force; consistency across the registry matters more. Plural reads correctly at the call site (parallel to "I want the louvain_communities metric"). The per-node int-label nature is documented per wrapper.

The critic's two flagged blockers: item 2 accepted; item 1 declined on call-site readability grounds. The critic's recurring theme ("match NetworkX yields to dispatcher's per-element semantics") is noted but not adopted globally; the user's read is that call-site naming matters more than column-shape signal, and shape is documented per-wrapper.

### API Critic — post-implementation review

Pending — invoke api-critic in post-implementation mode after each workstream that lands user-facing API code (Workstreams B and C). Workstream A is a pure file move with no user-facing API change; the api-critic does not need to review it. Workstream D is docs-only. Workstream E is notebook prose and CHANGELOG; the api-critic may skim the notebook for usage realism but a structured review is not required there.

## Workstreams

Five workstreams in recommended sequence: A → B → C, with D parallelizable against any of A/B/C, and E last so the notebook reflects the settled API. C is large enough that it splits into C1 (node-side wrappers plus adapter helpers) and C2 (edge-side link-prediction family) for clean dispatch.

### Workstream A: Move NetworkX wrappers into a `networkx/` subpackage

**Status:** not started
**Files:**

- New: `src/hiveplotlib/graph_features/networkx/__init__.py`, `src/hiveplotlib/graph_features/networkx/node_metrics.py`, `src/hiveplotlib/graph_features/networkx/edge_metrics.py`
- Removed: `src/hiveplotlib/graph_features/node_metrics.py`, `src/hiveplotlib/graph_features/edge_metrics.py`
- Edited: `src/hiveplotlib/graph_features/__init__.py` (imports + docstrings)
- Possibly edited: `docs/source/autodoc/hive_plots/graph_features.rst` (if it references the old path)
- Possibly edited: `tests/graph_features_test.py` (only if any direct import of the old submodule paths exists — current grep says no)

**Done when:**

1. `git grep "hiveplotlib.graph_features.node_metrics\|hiveplotlib.graph_features.edge_metrics"` returns only the two new file paths and the two updated import lines in `__init__.py`. No survivors anywhere else (especially not in `docs/`, `examples/`, `tests/`, or the rest of `src/`).
2. `make test` passes with the existing 12 registered metrics still parametrized through the smoke tests. Coverage 100%.
3. `make docs` builds without sphinx warnings about missing references.
4. `python -c "from hiveplotlib.graph_features import degree, edge_betweenness_centrality, GRAPH_NODE_METRICS, GRAPH_EDGE_METRICS, compute_graph_metrics; print(GRAPH_NODE_METRICS.keys())"` runs and prints the existing 10 node-metric keys.
5. The new `from hiveplotlib.graph_features.networkx import degree` import path also works (verified by ad-hoc smoke).

### Workstream B: Tier 1 NetworkX additions (11 drop-in wrappers)

**Status:** not started
**Files:**

- `src/hiveplotlib/graph_features/networkx/node_metrics.py` — add 11 wrappers.
- `src/hiveplotlib/graph_features/__init__.py` — register the 11 keys in `GRAPH_NODE_METRICS` and add them to the `from ... import ...` line plus `__all__`.
- `tests/graph_features_test.py` — restriction-raise tests for any new wrapper that adds a restriction (e.g. `in_degree_centrality` / `out_degree_centrality` should raise on undirected; the parametrized smoke test covers the happy path automatically).
- `CHANGELOG.rst` — bullet listing the Tier 1 additions under the "Added > Graph metrics" subsection of the v0.28.0 entry.

**Wrappers to add (all node-side):**

`degree_centrality`, `in_degree_centrality`, `out_degree_centrality`, `harmonic_centrality`, `average_neighbor_degree`, `onion_layers`, `square_clustering`, `constraint`, `effective_size`, `closeness_vitality`, `load_centrality`.

**Done when:**

1. All 11 new wrappers registered in `GRAPH_NODE_METRICS`.
2. Parametrized smoke test (`test_node_metric_wrapper_smoke`) passes for every new key — uses `_undirected_fixture()` by default, with `_directed_fixture()` for the in/out variants. Verify the smoke test's keyset-coverage assertion still holds (`set(result.keys()) <= set(g.nodes)`).
3. Restriction-raise tests added and passing for `in_degree_centrality`, `out_degree_centrality` (require directed), and any other Tier 1 wrapper that enforces a graph-type restriction (per the NetworkX docs at implementation time).
4. `make test` green at 100% coverage. No new warnings (CI runs with `filterwarnings = error`).
5. `make ty` green; `make format` produces no diff.
6. CHANGELOG bullet added.
7. api-critic post-impl review filled in this plan's "API Critic — post-implementation review" subsection.

### Workstream C1: Tier 2 node wrappers and adapter helpers

**Status:** not started
**Files:**

- `src/hiveplotlib/graph_features/networkx/node_metrics.py` — add 13 wrappers plus two private helpers (`_partition_to_node_labels`, `_set_to_indicator`). Optionally add the `@requires_graph_type` decorator and refactor the existing restriction checks if the boilerplate is getting noisy after Workstream B.
- `src/hiveplotlib/graph_features/__init__.py` — register the new keys.
- `tests/graph_features_test.py` — restriction-raise tests where applicable (e.g. `reciprocity` and `topological_generations` require directed graphs; `articulation_points` / `connected_components` require undirected or have specific component-counting semantics; `eigenvector_centrality_numpy` rejects multigraphs; etc.). Add an explicit shape test for the community-detection and connected-components wrappers asserting the values are int labels with at least one distinct group on a non-trivial fixture.
- `CHANGELOG.rst` — bullet for Tier 2 node-side additions.

**Wrappers to add (node-side):**

Single-shape adaptors: `eccentricity`, `eigenvector_centrality_numpy`, `reciprocity`, `topological_generations`.

Tuple-projecting: `hits_hubs`, `hits_authorities` (one underlying NetworkX call, two registry keys).

Set-to-indicator adaptors (boolean): `articulation_points`, `isolates`.

Partition-to-labels adaptors (community family): `greedy_modularity_communities`, `louvain_communities`, `label_propagation_communities`.

Partition-to-labels adaptors (components family): `connected_components`, `strongly_connected_components`, `weakly_connected_components`.

**Done when:**

1. All 13 new keys registered, all 4 helpers live (or 5 if the decorator is folded in).
2. Parametrized smoke test still green across the whole `GRAPH_NODE_METRICS` keyset (all 10 existing + 11 Tier 1 + 13 Tier 2 = 34 node metrics).
3. Restriction-raise tests pass for the directed-only and undirected-only wrappers.
4. Shape tests for the community-detection and connected-components families assert: keys are graph nodes, values are ints, at least 2 distinct labels on a fixture with structure.
5. `_partition_to_node_labels` sorts partitions by size descending, ties broken by smallest node id. Tested directly. Each community-detection and connected-components wrapper docstring documents the label ordering so the user can rely on "label 0 = giant community" semantics.
6. `hits_hubs` and `hits_authorities` wrapper docstrings explicitly note that requesting both metrics in one `compute_graph_metrics` call runs the underlying `nx.hits(G)` iteration twice (no caching layer per User Resolution item 4).
7. `make test`, `make ty`, `make format`, `make docs` all green.
8. CHANGELOG bullet added.
9. api-critic post-impl review filled.

### Workstream C2: Tier 2 edge link-prediction family

**Status:** not started
**Files:**

- `src/hiveplotlib/graph_features/networkx/edge_metrics.py` — add 6 wrappers: `bridges` (set-to-indicator) plus the 5 link-prediction wrappers built off the `_make_link_prediction_wrapper(nx_func_name)` factory (`jaccard_coefficient`, `adamic_adar_index`, `preferential_attachment`, `resource_allocation_index`, `common_neighbor_centrality`).
- `src/hiveplotlib/graph_features/__init__.py` — register the 6 keys in `GRAPH_EDGE_METRICS`.
- `tests/graph_features_test.py` — parametrized smoke covers everything; add a restriction-raise test for `bridges` (undirected only per NetworkX), and verify the link-prediction wrappers' return shape on a small fixture (dict keyed by 2-tuples, float values, populated over all edges in the graph).
- `CHANGELOG.rst` — bullet for Tier 2 edge-side additions.

**Done when:**

1. All 6 new keys registered. Factory helper is in place and unit-tested as the entry point for each of the 5 link-prediction wrappers (the wrappers themselves become 2-line registrations).
2. Parametrized smoke test still green across `GRAPH_EDGE_METRICS` (2 existing + 6 new = 8 edge metrics).
3. Link-prediction wrappers default `ebunch=G.edges()`. Test confirms the default produces a value for every edge in a small fixture's edge list.
4. `_make_link_prediction_wrapper` generates a templated docstring on each registered wrapper. The template warns that the values are link-likelihood scores being applied to existing edges (NetworkX's `jaccard_coefficient` and friends are intended for non-edges by default); the user can pass an explicit `ebunch=...` to score a different edge set. Verified by inspecting `help(jaccard_coefficient)` on the registered wrapper in an ad-hoc smoke.
5. `bridges` restriction-raise test passes on a directed input.
6. `make test`, `make ty`, `make format`, `make docs` all green.
7. CHANGELOG bullet added.
8. api-critic post-impl review filled.

### Workstream D: Roadmap entry for future igraph backend

**Status:** not started
**Files:**

- `docs/source/roadmap.rst` — add a new numbered item (or sub-bullet under existing item 7, "More networkx compatibility") covering the future igraph backend pitch.

**Content shape:** 5-15 line entry, matching the prose style of existing roadmap items. Cover:

- The headline pitch: igraph unlocks five community-detection algorithms NetworkX lacks (Leiden, Infomap, walktrap, spinglass, fast-greedy modularity) plus speed.
- The structural prerequisite is already done in this sprint (Workstream A).
- Known coverage gaps where the future igraph backend would raise `NotImplementedError`: `load_centrality`, `edge_load_centrality`, `closeness_vitality`, `effective_size`, `onion_layers`, `square_clustering`, `topological_generations`.
- Estimated future scope: ~30-35 wrappers, 600-900 LOC including tests.
- The open design questions deferred to that future sprint (see "Open design questions" section below).

**Done when:**

1. New roadmap entry renders correctly in `make docs`.
2. Entry length matches the prose discipline of existing roadmap items (no overlong text).
3. Cross-link from the entry back to this plan's deferred-questions list (or a one-line "see related deferred questions in the sprint plan that introduced this entry").

Workstream D is parallel-eligible against A/B/C: a docs writer can draft the prose against the current state of the roadmap file at any point. Recommend dispatching it once Workstream A is complete so the "structural prerequisite is done" framing is accurate.

### Workstream E: Notebook updates and final CHANGELOG sweep

**Status:** not started
**Files:**

- `examples/computing_graph_metrics.ipynb` — add small focused cells (per notebook-author skill, prefer additive over rewrite) demonstrating two or three of the highest-value new metrics. Recommended additions:

  - A "**Community detection as a partition variable**" cell pair: compute `louvain_communities` on Karate Club, use the resulting integer-label column as `partition_variable` directly (no discretization needed since labels are already integers / categorical-like). Story: Louvain recovers the historical `club` split on a familiar dataset (compare the computed label column with the ground-truth `club` attribute as a sanity check). This is the headline new capability.
  - A "**Precompute once, reuse across plots**" cell pair using `nx.les_miserables_graph()`. Compute `louvain_communities` and `harmonic_centrality` once with `compute_graph_metrics`, then build two `HivePlot` instances from the same annotated `nodes` / `edges` (one partitioning on community, one sorting by harmonic centrality). Les Mis is the better dataset here than Karate Club because the data is not pre-labeled, so Louvain reveals plot threads rather than just recovering a known partition. This cell also makes the cost story explicit ("community detection is expensive on bigger graphs; compute once, plot many times").
  - A "**Harmonic centrality and average neighbor degree**" cell pair: compute both Tier 1 metrics, show the resulting DataFrame head. Demonstrates a multi-metric init that's lighter than the existing pagerank / betweenness example.
  - A "**Link prediction edge coloring**" cell pair: compute `jaccard_coefficient` and color edges by it. Mirrors the existing `edge_betweenness_centrality` section. Markdown around the cell should reiterate the docstring's note that the values are link-likelihood scores applied to existing edges.

- `examples/computing_graph_metrics.ipynb` — add a short markdown cell near the top mentioning NetworkX's backend dispatch ecosystem (cuGraph, graphblas-algorithms, nx-parallel) as a one-paragraph note: "hiveplotlib computes metrics by calling NetworkX functions; users who install NetworkX backend plugins like `graphblas-algorithms` get those accelerated implementations for free when their version of NetworkX supports backend dispatch." No code change, just a forward-pointer. The prior research findings flagged this as worth one paragraph.

- `CHANGELOG.rst` — consolidate the Tier 1 / Tier 2 entries from Workstreams B / C1 / C2 into a clean section structure (probably one "Tier 1 additions" sub-bullet, one "Tier 2 additions" sub-bullet, one "Refactor" line for Workstream A). Update the metric counts in the existing "Graph metrics" section of the v0.28.0 entry: change "10 node-metric wrappers" to the new total, "2 edge-metric wrappers" to the new total. Per writing-voice rules, no em-dashes, no AI filler phrases, length-discipline.

**Done when:**

1. `make test-nb` runs `examples/computing_graph_metrics.ipynb` end-to-end and passes (notebooks run as part of CI / test-all). All new cells execute cleanly.
2. Notebook is only edited at `examples/computing_graph_metrics.ipynb`; `docs/source/notebooks/` and `docs/source/gallery_examples/` are not touched (auto-generated on docs build per CLAUDE.md).
3. New markdown cells follow the writing-voice rules (no em-dashes, no "moreover" / "furthermore" / "in essence", direct voice). Length matches sibling sections.
4. CHANGELOG counts updated correctly and reflect the final shipped state.
5. `make docs` builds the notebook into `docs/source/gallery_examples/` correctly.
6. api-critic skim (optional) confirms the notebook examples match the imagined user task.

## Holdouts

None — the replace-and-sweep audit in Workstream A targets only two module-path strings, and there's no teaching context that needs to keep the old path alive. If a holdout surfaces during execution (e.g. an inline comment in an unrelated file that mentions `node_metrics.py` for historical reasons), the engineer should add it here with a one-line reason.

## Open design questions (deferred to future igraph sprint, except #1)

The user asked these be surfaced as discussion items. They are NOT blockers for this sprint, except for #1 which informs Workstream A's structure.

1. **Graph type acceptance in `compute_graph_metrics`** (affects Workstream A's structure). Should the dispatcher eventually accept either `nx.Graph` or `ig.Graph` via `isinstance` dispatch, or always take NetworkX and convert internally?
   - **Planner recommendation for this sprint:** structure the move in Workstream A to *leave room for* isinstance dispatch, but don't wire it. Concretely: the registry keys (`GRAPH_NODE_METRICS`, `GRAPH_EDGE_METRICS`) stay flat and metric-name-keyed; the dispatcher stays NetworkX-typed in its signature for now. When the igraph backend ships, the engineer can either (a) widen the `graph` parameter to `Union[nx.Graph, ig.Graph]` and add a small `_dispatch_backend(graph)` helper inside `compute_graph_metrics`, or (b) split the registries into `GRAPH_NODE_METRICS_NX` / `GRAPH_NODE_METRICS_IG` and have a thin top-level resolver. Option (a) reads more ergonomic. The Workstream A move makes neither option harder.
   - **Action this sprint:** none beyond what Workstream A already does. The structural seam exists once the `networkx/` subpackage lands.
2. **Gap-metric strategy for future igraph backend.** Defer.
   - Three candidate behaviors: raise `NotImplementedError`, silently fall back to NetworkX, or allow per-metric override. Planner recommendation: `NotImplementedError` with a message naming the gap and pointing to the NetworkX equivalent. Silent fallback violates the "predictable backend" expectation.
3. **leidenalg packaging for future igraph.** Defer.
   - Bundle into `[igraph]`, separate `[igraph-leiden]`, or `[igraph-all]` roll-up. Planner recommendation: separate `[igraph-leiden]` extra; leidenalg is GPL and some users may want to opt out specifically.
4. **GPL license posture for future igraph.** Defer.
   - Optional extras don't contaminate the hiveplotlib distribution license, but a brief README/docs note explaining the license boundary will be needed. ADR-worthy when it lands.
5. **Notebook strategy when igraph eventually ships.** Defer.
   - Parity notebook (`examples/computing_graph_metrics_igraph.ipynb`) or extend the existing one. Planner recommendation: parity notebook, since the backend differences are pedagogical enough to deserve their own walkthrough.

These should be linked from the Workstream D roadmap entry as "deferred design questions tracked in the sprint plan that introduced this roadmap item."

## Plan amendments

None yet. The Orchestrator will populate this section in amend-plan mode if
emergent work surfaces (rule 14 trigger).

## Implementation log

Append-only. After each workstream completes, the executing agent writes one line here in the same turn:

- (none yet)
