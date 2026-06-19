# Plan: NetworkX metric expansion plus structural refactor for future igraph backend

## Goal

Close out the NetworkX integration sprint by (1) restructuring `hiveplotlib.graph_features` so the NetworkX-coupled wrappers live in a `networkx/` subpackage (sibling backends can drop in later), (2) adding ~26 high-value NetworkX metric wrappers across two tiers, (3) seeding a roadmap entry for a future igraph backend, and (4) refreshing the example notebook and CHANGELOG. The public surface (`compute_graph_metrics`, `GRAPH_NODE_METRICS`, `GRAPH_EDGE_METRICS`, top-level imports from `hiveplotlib.graph_features`) stays unchanged; the user-visible value is "the metric subsystem now covers community detection, harmonic centrality, eccentricity, link prediction, and a dozen other staples a real graph analyst expects, without bloating the surface."

## Prior ADRs / design docs

- **`wiki/wiki/plans/i-want-to-plan-optimized-hoare.md` — the canonical prior work this plan builds on.** That plan (the first of two plans for GitLab #46) shipped the entire NetworkX consolidation surface: bidirectional conversion (`networkx_to_nodes_edges`, `nodes_edges_to_networkx`, `HivePlot.to_networkx`, `HivePlot.from_networkx`), the `hiveplotlib.graph_features` package with 10 node-metric wrappers plus 2 edge-metric wrappers plus the `compute_graph_metrics` dispatcher, the `HivePlot` / `HivePlotMatrix` init-time and post-hoc graph-metric APIs (`node_graph_metrics`, `edge_graph_metrics`, the `*_kwargs` / `*_rename` companions, `graph_directed` / `graph_multigraph`), three new gallery notebooks (`computing_graph_metrics`, `exporting_hive_plots_to_networkx`, `exporting_hive_plots_to_json`), and the rendered metric tables in the API docs. Workstreams A through O of that plan plus ~12 in-scope tweaks landed and are reflected in the v0.28.0 entry of `CHANGELOG.rst`. The current plan extends that surface (Workstream A restructures the `graph_features` package; Workstreams B and C add ~25 wrappers; Workstream D seeds the future igraph backend in the roadmap; Workstream E refreshes the example notebook). The prior plan stays as the canonical record of the consolidation; this new plan exists because the prior plan was getting too large to fold more into, so the user split execution.
- **One explicit reversal from the prior plan to call out:** the prior plan's Workstream B (i-want-to-plan-optimized-hoare.md:127) deliberately excluded link-prediction functions (`jaccard_coefficient`, `preferential_attachment`, and friends) on the grounds that "they iterate over *non-existing* edges and don't fit the 'augment my existing edges with a column' workflow." This plan's Workstream C2 reverses that decision by adding the link-prediction family via a `_make_link_prediction_wrapper` factory that defaults `ebunch=G.edges()` so the scores attach to existing edges, with a templated docstring warning that NetworkX intends these scores for non-edges by default (per the API critic's item 2 in this plan's planning-mode review). The reversal is deliberate; including the wrappers behind a documented "we apply these to existing edges, NetworkX applies them to non-edges by default" caveat is more useful than the blanket exclusion.
- **No ADRs exist yet in the wiki.** The `wiki/wiki/adr/` directory has not been created; the prior plan above is the binding prior decision record for the consolidation surface this plan extends. ADR promotion for the combined networkx-and-backend-refactor story is **deferred to v0.28.0 close-out** (after both plans ship); the QA Engineer should NOT flag ADR promotion as a worth-discussing concern at this plan's close, because the prior plan's QA pass already established the same deferral. One combined ADR is the target, not two separate ADRs. The structural-refactor decision (Workstream A's `networkx/` subpackage move) and the NetworkX-as-optional dep posture both belong in that future combined ADR.
- Other relevant wiki pages (informational only, no binding decisions):
  - `wiki/wiki/entities/hiveplotlib.md` — current entity status page; lists the NetworkX converter as a key feature, will need a "key APIs" update once this work ships (post-task pass for the Research Liaison, batched with the prior plan's pending updates).
  - `wiki/wiki/sources/hiveplotlib-python.md` — source summary noting "minimal base deps: matplotlib, numpy, pandas only; optional backends as extras." Confirms NetworkX-as-optional is the established convention; this plan keeps that boundary intact.
  - `wiki/wiki/analyses/gnn-heterogeneity-hive-plots.md` and `wiki/wiki/analyses/karate-club-walkthrough.md` — research directions that will benefit from richer metric coverage (community-detection wrappers in particular). Not constraining, but worth knowing the audience.

## Patterns this replaces

The refactor in Workstream A is the only place this plan replaces existing patterns. All Tier 1 and Tier 2 additions are net new wrappers.

Replace-and-sweep targets (grepped against the current branch):

- **Module path `hiveplotlib.graph_features.node_metrics`** referenced at:
  - `src/hiveplotlib/graph_features/__init__.py:5` (docstring), `:13` (extend instructions), `:38` (import line)
  - `src/hiveplotlib/graph_features/node_metrics.py:5` (self-reference docstring; moves with the file)
  - `docs/source/autodoc/hive_plots/graph_features.rst:31` (`automodule:: hiveplotlib.graph_features.node_metrics` directive — required edit, not optional; sphinx silently produces an empty section if the path is wrong)
- **Module path `hiveplotlib.graph_features.edge_metrics`** referenced at:
  - `src/hiveplotlib/graph_features/__init__.py:5` (docstring), `:13` (extend instructions), `:34` (import line)
  - `src/hiveplotlib/graph_features/edge_metrics.py:5` (self-reference docstring; moves with the file)
  - `docs/source/autodoc/hive_plots/graph_features.rst:37` (`automodule:: hiveplotlib.graph_features.edge_metrics` directive — required edit, same reason as above)
- **Files to move:**
  - `src/hiveplotlib/graph_features/node_metrics.py` → `src/hiveplotlib/graph_features/networkx/node_metrics.py`
  - `src/hiveplotlib/graph_features/edge_metrics.py` → `src/hiveplotlib/graph_features/networkx/edge_metrics.py`
- **Notebook traceback strings.** `examples/computing_graph_metrics.ipynb` cells 1370 and 1676 contain rendered traceback output that mentions the old module path (`/home/garyk/repos/hiveplotlib/src/hiveplotlib/graph_features/node_metrics.py`). These are output cells, not source — running `make test-nb` re-executes the notebook and the new path appears naturally in the regenerated traceback. No manual edit needed; flagged here so a post-execution grep doesn't surface them as surviving references.

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

New node-metric keys (Tier 2, 14 wrappers across families):

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
hp = HivePlot(
    graph=G,
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

### Refinement-round addendum (planning mode, second pass)

Re-walked the examples after the orchestrator's refinement pass and after the user resolutions baked in above. Resolutions for items 1-8 hold under the refined plan; nothing in the orchestrator's three changes (Prior ADRs rewrite, the `automodule::` directive promotion, CHANGELOG routing) intersects the user-facing surface in a way that reopens those items. One must-fix surfaced from re-walking Example 2 against the actually-shipped surface, plus one worth-discussing about the implicit `hp` carryover in the headline examples block.

- **[must-fix] Example 2 calls `HivePlot.from_networkx(G, ...)`, which does not exist.** The prior plan's Workstream I removed the `HivePlot.from_networkx` classmethod and replaced it with a keyword-only `graph=` parameter on `HivePlot.__init__`. The shipped surface enforces this with a muscle-memory guard at `src/hiveplotlib/hiveplot.py:1919-1924` that raises `ValueError: \`graph\` is a keyword-only parameter. Rewrite as \`HivePlot(graph=..., ...)\`` when a `nx.Graph` is bound positionally to `nodes`. Example 2 in this plan (at `wiki/wiki/plans/networkx-metric-expansion-and-backend-refactor.md:117`) still uses the old shape:

  ```python
  hp = HivePlot.from_networkx(
      G,
      partition_variable="club",
      sorting_variables="degree",
      node_graph_metrics="degree",
      edge_graph_metrics="jaccard_coefficient",
  )
  ```

  This will hit `AttributeError: type object 'HivePlot' has no attribute 'from_networkx'`, not even the muscle-memory guard (the guard fires on positional `nx.Graph` to `nodes`, not on a missing classmethod). The existing notebook at `examples/computing_graph_metrics.ipynb` consistently uses `HivePlot(graph=G, ...)` (see the `hp_edge` cell at the "Add Edge Metrics When Initializing Hive Plot" section), which is the canonical pattern. The CHANGELOG entry for v0.28.0 also documents `HivePlot` accepts `networkx` graphs "directly via a keyword-only `graph` parameter."

  The original critic's resolution-list item 5 caught the `HivePlot.from_networkx`-vs-two-step mismatch for Example 1 (rewriting it as `networkx_to_nodes_edges` → `compute_graph_metrics` → `HivePlot(nodes=, edges=, ...)`) but Example 2 retained the broken classmethod across the resolution sweep. Same root cause; same fix shape.

  Suggested rewrite:
  ```python
  hp = HivePlot(
      graph=G,
      partition_variable="club",
      sorting_variables="degree",
      node_graph_metrics="degree",
      edge_graph_metrics="jaccard_coefficient",
  )
  hp.update_edge_plotting_keyword_arguments(
      array="jaccard_coefficient", cmap="plasma",
  )
  ```

  This is `must-fix` because it's the highest-leverage example for the C2 link-prediction reintroduction (the whole point of reversing the prior plan's exclusion is "let users color edges by predictor scores"). A reader landing here as their first encounter with link-prediction-on-edges will copy this snippet, hit `AttributeError`, and either dig into the source or assume the feature is broken. Either failure mode burns the plan's framing budget on the very capability it's being added to support. The fix is a one-line snippet edit in the plan; flagging as `must-fix` so the orchestrator's `amend-plan` mode can lock it before code-engineer dispatches.

- **[worth-discussing] Example 2 → 3 → 4 implicitly carry `hp` and `G` from Example 2; if Example 2's `hp` doesn't build, the cascade goes with it.** Examples 3 and 4 both call `hp.compute_graph_metrics(...)`. The implicit-carry is fine as a planning-doc convention (Example 1 sets up `G`; Example 5 stands alone), but with Example 2 broken, a reader can't even smoke-test the post-hoc `compute_graph_metrics` story shown in Examples 3 and 4 against the shipped surface. Once Example 2 is fixed per the must-fix above, this concern dissolves on its own. Worth-discussing rather than must-fix because the dependency reads as intentional planning-doc compression, not a separate bug. Route fix: when amending Example 2, optionally add a one-line comment "(reuses `G`, `hp` from Example 2)" to Examples 3-4 so the carryover is explicit. Low-priority; user's call.

- **[low-confidence] Example 5 redefines `nodes, edges = networkx_to_nodes_edges(G)` but uses metric names that differ from Example 1 (`harmonic_centrality` vs Example 1's `degree`). Probably fine.** Example 5's framing is "the pre-HivePlot pipeline using the lower-level `compute_graph_metrics()`" and demonstrating a different metric makes sense (showing the API isn't single-metric-only). Mentioning here only to flag that a reader speed-scanning Examples 1 and 5 side by side will register the metric difference as intentional, not as a typo. No edit needed.

- **[low-confidence] Example 4's `graph_directed=False` is required for `articulation_points`, but the wrapper-restriction story for this Tier 2 family is unspecified in the plan.** The plan's Workstream C1 done-when #3 mentions "Restriction-raise tests pass for the directed-only and undirected-only wrappers" without listing which Tier 2 wrappers fall into each bucket. The Example 4 comment ("articulation points defined on undirected graphs") implies `articulation_points` enforces undirected, but the plan body doesn't say so explicitly. NetworkX's `articulation_points` is documented as undirected-only; if Workstream C1 mirrors that with a restriction raise, Example 4's `graph_directed=False` is the right call. If Workstream C1 leaves it permissive, the wrapper should probably enforce. Flagging at low confidence because the answer is in NetworkX's docs at implementation time and the implementing engineer should land on the right call by reading those; not a plan-edit blocker. The headline example just needs to match whatever shape Workstream C1 ships.

- **[low-confidence] Workstream E's "Link prediction edge coloring" cell (mentioned in the plan as a notebook addition) implicitly depends on the link-prediction wrappers' default `ebunch=G.edges()` behavior. If a notebook reader changes `G` to a sparse graph, the link-prediction scores degenerate (most pairs are non-edges; the scored set is trivial).** The user resolution item 2 (`_make_link_prediction_wrapper` generates a templated docstring warning the values are link-likelihood scores applied to existing edges) handles the conceptual mismatch on the API side. Worth-discussing analog for the notebook: the Workstream E markdown around the cell should restate the same caveat in user vocabulary ("we're using these as edge-style signals over your existing edges, not predicting new edges"). The plan's Workstream E bullet at line 373 already says "Markdown around the cell should reiterate the docstring's note that the values are link-likelihood scores applied to existing edges," so this is already in the plan — flagging here only for the notebook-author specialist to actually land the prose, not just defer to the docstring text. No plan edit needed.

**Refinement-round verdict:** the orchestrator's three refinement edits (Prior ADRs rewrite, `automodule::` directive promotion, CHANGELOG routing tightening) are sound and don't reopen any of the original critic's user-resolved items. Items 1-8 hold. The Example 2 break is the only true must-fix from this re-walk; everything else is low-priority. Recommend routing the Example 2 fix through orchestrator `amend-plan` before code-engineer dispatch on C2 (since C2's user-facing surface is exactly the link-prediction family Example 2 demos).

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

Pending — invoke api-critic in post-implementation mode after each workstream that lands user-facing API code (Workstreams B, C1, C2 — one invocation per workstream, since each registers a distinct slice of new string keys). Workstream A is a pure file move with no user-facing API change; the api-critic does not need to review it (the import-path seam at `hiveplotlib.graph_features.networkx.*` is an internal advanced-user surface, not a headline new API). Workstream D is docs-only. Workstream E is notebook prose and CHANGELOG; the api-critic may skim the notebook for usage realism but a structured review is not required there.

#### Workstream B (Tier 1 NetworkX additions, 11 node-metric wrappers)

```
Status: propose
API surface reviewed: hiveplotlib.graph_features.degree_centrality,
  in_degree_centrality, out_degree_centrality, harmonic_centrality,
  average_neighbor_degree, onion_layers, square_clustering, constraint,
  effective_size, closeness_vitality, load_centrality
  (all 11 also reachable via hiveplotlib.graph_features.networkx.*)
Concerns:
  - [worth-discussing] `onion_layers` shows "no constraint" in the rendered Node Metric Table
    even though the wrapper requires undirected AND non-multigraph. The classifier in
    `docs/source/_ext/metric_table_directive.py:111-136` recognizes three single-axis
    restrictions (directed_only, undirected_only, simple_only) but the joint-axis case where
    only `simple_undirected` succeeds falls through all three predicates and renders as `""`.
    A user scanning the table to find "which metrics work on my MultiGraph?" will see
    `onion_layers` unflagged, try it, and hit a ValueError that the docstring covers but the
    table denied. The plan's framing flagged this as a potential Workstream E concern; the
    cleanest fix lives in the metric-table directive (one extra predicate plus one sentence
    like "Requires an undirected, non-multigraph graph.") and is mechanically cheap.
    Auto-deferrable: this is the only Tier 1 wrapper that falls into the joint-restriction
    bucket today (the prior 10 didn't surface it; Tier 2's `topological_generations` and
    `articulation_points` may surface other joint cases in Workstream C1). Bundling the
    classifier extension with the C1 sweep is reasonable; flagging here so the C1 dispatcher
    or Workstream E author picks it up, not as a B-blocker — at `docs/source/_ext/metric_table_directive.py:111`.
    Suggested change: add a fourth predicate `simple_undirected_only` (works only on
    `nx.Graph`) returning "Requires an undirected, non-multigraph graph."; bundle with
    Workstream C1 to handle whatever new joint cases the Tier 2 wrappers introduce.
  - [worth-discussing] The new wrappers carry a conceptual paragraph between the "Wraps" line
    and the `:param:` block (e.g. `harmonic_centrality:301-303` explains why it's well-defined
    on disconnected graphs; `constraint:387-389` explains the high/low interpretation;
    `onion_layers:333-334` explains the k-core refinement). The pre-existing 10 wrappers
    (`betweenness_centrality`, `pagerank`, `clustering`, etc.) do not. The new convention is
    strictly better for the niche metrics where a user lands cold (`constraint`,
    `effective_size`, `closeness_vitality`, `onion_layers` especially), but the asymmetry
    means a user browsing the rendered `Node Metrics` autodoc section will see uneven
    detail across rows. User's call: backfill the pre-existing 10 with similar one-paragraph
    blurbs in a separate small docs pass, or accept the asymmetry as "new house style for
    less-familiar metrics." Not a blocker either way — at
    `src/hiveplotlib/graph_features/networkx/node_metrics.py:233-450` (new wrappers) vs `:77-230` (pre-existing).
    Suggested change: if backfilling, target the more-niche pre-existing wrappers
    (`core_number`, `clustering`) first; the centrality trio is likely familiar enough to
    leave alone.
  - [low-confidence] `effective_size` returns `nan` for isolated nodes (NetworkX's
    underlying implementation divides by the node's degree). The wrapper passes through
    cleanly, but a user computing `effective_size` on a sparse graph with isolates will get
    a `nan` column with no warning. The docstring's "Burt's measure of non-redundancy"
    explanation doesn't surface this gotcha. Same issue plausibly applies to `constraint`
    (NetworkX returns `nan` for nodes with no neighbors). Low-confidence because this is
    NetworkX-native behavior — the wrapper is a thin pass-through and adding nan-handling
    in the wrapper would diverge from NetworkX's contract, which is exactly what the "Wraps"
    pattern resists. The right surface is probably a one-sentence "Returns nan for isolated
    nodes" note in the docstring rather than a code change — at
    `src/hiveplotlib/graph_features/networkx/node_metrics.py:398-414` (effective_size) and `:379-395` (constraint).
    Suggested change: add a "Returns ``nan`` for isolated nodes (or nodes whose ego network
    is empty); both originate from NetworkX's underlying implementation." sentence to the
    `effective_size` and `constraint` docstrings.
Test-method-coverage audit: clean. All 5 new restriction-raise tests follow the
  test_<method>_<scenario> contract:
    - test_in_degree_centrality_on_undirected_raises calls in_degree_centrality (line 207)
    - test_out_degree_centrality_on_undirected_raises calls out_degree_centrality (line 218)
    - test_onion_layers_on_directed_raises[DiGraph,MultiDiGraph] calls onion_layers (line 232)
    - test_onion_layers_on_multigraph_raises calls onion_layers (line 248)
  The parametrized `test_node_metric_wrapper_smoke` auto-picks-up all 11 new keys via
  `_node_metric_names()` against `GRAPH_NODE_METRICS`, routing `in_degree_centrality` /
  `out_degree_centrality` through `_directed_fixture()` (test file line 87-92). Other 9 new
  wrappers (no restrictions) flow through `_undirected_fixture()` correctly.
```

**Walkthrough notes (for future reference, not a finding):** walked `harmonic_centrality`,
`onion_layers`, `effective_size`, `constraint`, `closeness_vitality`, and `load_centrality`
on Karate Club as a hypothetical first-time user. The headline wrapper (`harmonic_centrality`)
imports cleanly, returns the expected `dict[node, float]`, and the docstring's "well-defined
on disconnected graphs where closeness isn't" line is exactly the hook a user reaches for
when picking this over closeness. The two directed-only centralities (`in_degree_centrality`,
`out_degree_centrality`) inherit the helpful recovery-message pattern from the prior
`in_degree` / `out_degree` wrappers ("Build the source graph with `directed=True` ..."), so
a user typo'ing the graph type lands on a one-line fix. The backend-explicit import path
`from hiveplotlib.graph_features.networkx import harmonic_centrality` resolves correctly
(confirmed by reading `src/hiveplotlib/graph_features/networkx/__init__.py:11-37`); all 11
new keys are in the subpackage `__all__`.

**Verdict on routing:** no `must-fix` or `should-fix`. The three findings above are all
`worth-discussing` or `low-confidence`. The `onion_layers` table-classifier gap is the
most concrete and likely to bite, but is auto-deferrable to Workstream C1's joint-classifier
sweep (which will need the same extension for any Tier 2 wrappers that introduce new joint
cases). No need to route through orchestrator `amend-plan`; the dispatching session can
continue with Workstreams C1 / C2 / D / E and the user decides on backfill / docstring polish
at v0.28.0 close-out.

#### Workstream C1 (Tier 2 node wrappers and adapter helpers, 14 new wrappers + 2 helpers)

```
Status: propose
API surface reviewed: hiveplotlib.graph_features.eccentricity,
  eigenvector_centrality_numpy, hits_hubs, hits_authorities, reciprocity,
  articulation_points, isolates, topological_generations,
  greedy_modularity_communities, louvain_communities,
  label_propagation_communities, connected_components,
  strongly_connected_components, weakly_connected_components
  (all 14 also reachable via hiveplotlib.graph_features.networkx.*; the two
  private helpers `_partition_to_node_labels` and `_set_to_indicator` are
  internal-only, audited for shape but not for surface)
Concerns:
  - [should-fix] `eigenvector_centrality_numpy` ships with no friendly multigraph
    guard, leaving Workstream C1's done-when #3 ("`eigenvector_centrality_numpy`
    rejects multigraphs") unmet. The sibling `eigenvector_centrality` enforces
    the constraint with an explicit ValueError plus a one-line recovery message
    naming the fix ("Build the source graph with `multigraph=False` (e.g., on
    the HivePlot initialization or `nodes_edges_to_networkx`).") at
    `src/hiveplotlib/graph_features/networkx/node_metrics.py:174-180`.
    `eigenvector_centrality_numpy` (at `:521-537`) has no such guard. NetworkX's
    underlying implementation does NOT raise on a multigraph; it routes through
    `nx.to_scipy_sparse_array` which sums parallel-edge weights into a single
    adjacency entry, returning a silently-different-than-expected result. A user
    calling this on a MultiGraph won't see an error at all (no metric-table
    classifier warning will fire either, since the classifier probes for
    `ValueError` / `NetworkXException` and gets neither). This is the worst
    failure mode for a Workstream B's `onion_layers` table-classifier fix tried
    to close: a user sees no constraint, runs the wrapper, gets confusing values
    rather than a useful error. The sibling-wrapper precedent and done-when #3
    together make this `should-fix`. Cost: ~7 lines mirroring lines 174-180 — at
    `src/hiveplotlib/graph_features/networkx/node_metrics.py:521-537`.
    Suggested change: add an `if graph.is_multigraph(): raise ValueError(...)` guard
    matching the `eigenvector_centrality` template; ship the matching restriction-raise
    test in `tests/graph_features_test.py` (parametrized over `MultiGraph` /
    `MultiDiGraph` like `test_eigenvector_centrality_on_multigraph_raises`); update
    the docstring's "Requires a non-multigraph" sentence to match the sibling.
  - [worth-discussing] `topological_generations` mentions `NetworkXUnfeasible` in
    its prose paragraph (lines 676-678) but the `:raises:` block (line 682) only
    lists `ValueError`. A user reading the Sphinx-rendered docs scrolls to the
    `:raises:` block to learn what they need to catch; the prose mention won't
    register there, and Sphinx won't auto-generate a `:raises:` entry from
    free-form text. Walking a real user: someone with a cyclic directed graph
    calls `topological_generations(my_graph)`, hits a `NetworkXUnfeasible` they
    didn't plan for, and finds no docstring `:raises:` entry to confirm what they
    just saw. The prose paragraph is correct; the load-bearing `:raises:` block
    is incomplete — at
    `src/hiveplotlib/graph_features/networkx/node_metrics.py:680-682`.
    Suggested change: add a second `:raises:` line — `:raises networkx.NetworkXUnfeasible:
    if ``graph`` contains a cycle (raised by the underlying networkx implementation).`
    Pair-symmetry with the existing `:raises ValueError:` line.
  - [low-confidence] `reciprocity` always passes `nodes=graph.nodes` (line 615) so
    the per-node dict path is taken, with no opt-out for the graph-level-scalar
    behavior NetworkX returns when `nodes` is omitted. The docstring spells this
    out ("the wrapper always passes ``nodes=graph.nodes`` so the underlying
    function returns a per-node dict rather than the graph-level scalar"), so a
    user who reads the docstring is not surprised. The constraint is necessary
    because the dispatcher's contract requires `dict[node, value]`, not a scalar.
    Flagged only because a NetworkX-fluent user reaching for `kwargs` on
    `reciprocity` will get a TypeError (the wrapper takes no `**kwargs` at all,
    line 590) when they try to pass `nodes=[specific_subset]`. That's the right
    shape for the dispatcher contract, but the type signature `def reciprocity(graph)`
    (no `**kwargs`) is the visible signal — at
    `src/hiveplotlib/graph_features/networkx/node_metrics.py:590`.
    Suggested change: none required (the docstring carries the load); if the user
    later wants subset-reciprocity, they can call `nx.reciprocity` directly. Leaving
    this for Workstream E's notebook author in case a "compute reciprocity on a
    subset" example is added.
  - [low-confidence] `isolates` works on all four graph types (no guard) but the
    docstring doesn't say so explicitly. A reader scanning the docstrings of the
    structural-position wrappers (`articulation_points`: undirected-only;
    `isolates`: silent on graph-type) might assume `isolates` shares the
    undirected-only constraint by association. NetworkX's `isolates` does indeed
    work on all four graph types (the wrapper is a thin pass-through). The metric
    table will render no constraint sentence, which is correct, but the wrapper
    docstring's silence is asymmetric with the verbose-constraint pattern
    `articulation_points` carries one paragraph above — at
    `src/hiveplotlib/graph_features/networkx/node_metrics.py:647-661`.
    Suggested change: optional one-line "Works on all four graph types (no
    restriction)." appended to the `isolates` docstring to mirror the explicit
    constraint sentences elsewhere; defer to Workstream E or v0.28.0 close-out
    if backfilling all unrestricted wrappers (`degree`, `pagerank`, etc.) gets
    folded in.
Test-method-coverage audit: clean. All 7 new restriction-raise tests follow the
  test_<method>_<scenario> contract; spot-checked:
    - test_reciprocity_on_undirected_raises calls reciprocity (line 319)
    - test_topological_generations_on_undirected_raises calls topological_generations (line 330)
    - test_articulation_points_on_directed_raises[DiGraph,MultiDiGraph] calls
      articulation_points (line 345)
    - test_label_propagation_communities_on_directed_raises[DiGraph,MultiDiGraph]
      calls label_propagation_communities (line 360)
    - test_connected_components_on_directed_raises[DiGraph,MultiDiGraph] calls
      connected_components (line 375)
    - test_strongly_connected_components_on_undirected_raises calls
      strongly_connected_components (line 386)
    - test_weakly_connected_components_on_undirected_raises calls
      weakly_connected_components (line 397)
  All 6 new family shape tests call their named method against
  `_two_component_undirected_fixture()` / `_two_component_directed_fixture()`
  (lines 471-556). The parametrized `test_node_metric_wrapper_smoke` auto-picks-up
  all 14 new keys via `_node_metric_names()` against `GRAPH_NODE_METRICS`, routing
  `topological_generations` through `_dag_fixture()` (line 139) and
  `reciprocity`/`strongly_connected_components`/`weakly_connected_components`
  through `_directed_fixture()` (lines 140-148). The 6 non-restricted Tier 2
  wrappers (`eccentricity`, `eigenvector_centrality_numpy`, `hits_hubs`,
  `hits_authorities`, `isolates`, `greedy_modularity_communities`,
  `louvain_communities`) flow through `_undirected_fixture()` correctly.
  The two private adapters `_partition_to_node_labels` (x2 tests) and
  `_set_to_indicator` (x1 test) have direct unit tests at lines 405-463.
  Gap on the should-fix above: no `test_eigenvector_centrality_numpy_on_multigraph_raises`
  test exists yet (would arrive with the guard fix).
```

**Walkthrough notes (for future reference, not a finding):** walked `eccentricity`,
`hits_hubs`/`hits_authorities`, `reciprocity`, `articulation_points`, `isolates`,
`topological_generations`, `louvain_communities`, `connected_components`,
`strongly_connected_components` as a hypothetical first-time user.

The three new shape families read cleanly. Boolean wrappers
(`articulation_points`, `isolates`) lead with "Return a per-node boolean flag
indicating ..." in the first docstring line, so a user iterating `for k, v in
result.items(): if v: ...` immediately understands the projection. Community /
component wrappers (`louvain_communities`, `connected_components`, etc.) lead
with "Return the <X> community/component label of each node" and every one of
the 7 restates the "label 0 = largest, ties broken by smallest node id, stable
across runs and versions" contract in a dedicated paragraph. A user comparing
two community-detection runs can rely on label 0 meaning the same thing in both
calls; the docstrings make this explicit.

The HITS docstrings (lines 540-587) both spell out the "requesting both runs
the iteration twice (no shared cache)" cost prominently in their own paragraph,
so a user batching `compute_graph_metrics(..., node_metrics=["hits_hubs",
"hits_authorities"])` is not surprised. The User Resolution decision (item 4,
no caching layer) reads cleanly through the surface.

The classifier extension in `docs/source/_ext/metric_table_directive.py:62-161`
correctly classifies `onion_layers` as "Requires an undirected, non-multigraph
graph." (the joint case from Workstream B's worth-discussing finding) and
`topological_generations` as "Requires a directed graph." (via the secondary
DAG probe at lines 100-106 that catches DAG-requiring metrics whose primary
probe on cyclic graphs would fail). The "Requires an undirected, non-multigraph
graph." sentence is a mouthful but unambiguous; nothing better available.

`eigenvector_centrality_numpy` (lines 521-537) correctly cross-references
`eigenvector_centrality` and explains the choice ("more reliable on graphs
where the iterative solver fails to converge"), so a user wondering "why two
eigenvector centralities?" has the answer in the first paragraph. The
multigraph-guard gap (should-fix above) is the one place this wrapper diverges
from its sibling.

The backend-explicit import path `from hiveplotlib.graph_features.networkx
import louvain_communities, connected_components, ...` resolves correctly
(confirmed by reading `src/hiveplotlib/graph_features/networkx/__init__.py:15-51`);
all 14 new keys are in the subpackage `__all__` and the parent
`graph_features/__init__.py` re-export list (lines 40-76).

**Verdict on routing:** one `should-fix` (`eigenvector_centrality_numpy` missing
multigraph guard; done-when #3 explicitly required it). Routes through
orchestrator `amend-plan` before C2 dispatch since the fix lives in the same
file C2 will not touch (C2 is edge-side). The `topological_generations`
`:raises:` block fix is `worth-discussing` and can be picked up either in the
amend-plan fold-in or at Workstream E / v0.28.0 close-out (defers cleanly
either way). The two `low-confidence` items are docstring-polish only; defer to
Workstream E or close-out.

##### Amendment 1 confirmation (post-impl, 2026-05-25)

```
Status: clean
API surface reviewed: hiveplotlib.graph_features.eigenvector_centrality_numpy
  (multigraph guard added), hiveplotlib.graph_features.topological_generations
  (:raises: block extended); narration-count edits across CHANGELOG and plan body
Concerns: none (amendment closes the original should-fix plus the worth-discussing
  raises-block gap cleanly; no new findings introduced)
Test-method-coverage audit: clean. test_eigenvector_centrality_numpy_on_multigraph_raises
  at tests/graph_features_test.py:215-228 calls eigenvector_centrality_numpy in its
  body (line 228), parametrized over MultiGraph / MultiDiGraph in exact parity with
  the sibling test_eigenvector_centrality_on_multigraph_raises at :200-211.
```

**Walkthrough notes (amendment confirmation, not a finding):**

1. **Multigraph guard parity-plus-improvement.** Walked the new
   `eigenvector_centrality_numpy` guard at `src/hiveplotlib/graph_features/networkx/node_metrics.py:546-552`
   against the sibling `eigenvector_centrality` guard at `:174-180`. The
   `ValueError` message body is verbatim identical to the sibling (wrapper name
   swapped, same "Build the source graph with `multigraph=False` (e.g., on the
   HivePlot initialization or `nodes_edges_to_networkx`)." recovery template),
   the `:param graph:` qualifier `(not a multigraph)` matches, and the
   `:raises ValueError:` line matches. The "Requires a non-multigraph" paragraph
   at `:533-539` is the deliberate improvement over the sibling: it explains
   *why* the restriction exists (the parallel-edge collapse in
   `nx.to_scipy_sparse_array`) and explicitly names
   `hiveplotlib.converters.nodes_edges_to_networkx(multigraph=False)` as the
   recovery action. The sibling carries the same restriction so "use
   `eigenvector_centrality` instead" is not a viable fallback; pointing the user
   at the rebuild path is the right call. A first-time user hitting the
   `ValueError` lands on a one-line fix.

2. **`:raises:` block render.** The new
   `:raises networkx.NetworkXUnfeasible: ...` line at
   `src/hiveplotlib/graph_features/networkx/node_metrics.py:701-702` wraps onto
   two lines for the 120-char docstring limit; the four-space continuation
   indent at `:702` is the standard reStructuredText pattern and renders as
   inline continuation (no visible break in the HTML). Trusting QA on the build
   green: a user reading the rendered `:raises:` block now sees both
   `ValueError` (undirected input) and `NetworkXUnfeasible` (cyclic input) as
   separate entries, with the latter as an intersphinx-linked cross-reference to
   NetworkX's docs. The prose paragraph at `:694-696` and the structured block
   at `:700-702` are now consistent; no surprise failure modes for the user.

3. **Count narration consistency.** Spot-checked five sites and they all read
   `14` / `35`: `CHANGELOG.rst:69` ("14 additional NetworkX-backed node-metric
   wrappers"), the plan body's intro ("New node-metric keys (Tier 2, 14 wrappers
   across families)"), the C1 workstream header ("Workstream C1 ... 14 new
   wrappers + 2 helpers"), the C1 done-when count ("10 existing + 11 Tier 1 +
   14 Tier 2 = 35 node metrics"), and the C1 implementation-log entry ("14
   Tier 2 ... 35 keys total: 10 prior + 11 from B + 14 new"). The amendment
   block correctly preserves the prior-drift quote strings (`13 additional`,
   `13 Tier 2 ... 34 keys total`) since those are descriptions of what the
   amendment is fixing. Trusting QA on the comprehensive grep.

4. **CHANGELOG omission judgment (concur, low-confidence).** Concur with the
   code-engineer's call to skip a "Fixed" bullet for the multigraph guard and
   the `:raises:` block. The v0.28.0 entry is WIP and the C1 work has not
   shipped to a release; framing either change as a fix in the user-facing
   changelog would imply a broken prior release that does not exist. The guard
   correction is part of C1's scope (done-when #3 explicitly required it), and
   the `:raises:` block edit is a docstring polish that the existing
   `14 additional NetworkX-backed node-metric wrappers ... eigenvector_centrality_numpy
   ... topological_generations ...` bullet at `CHANGELOG.rst:69-73` already
   covers in aggregate. Low-confidence tag because a stricter "every code
   change gets a bullet" posture could justify mentioning the guard in passing,
   but that's a judgment call and the no-bullet path is defensible.

5. **No new findings introduced by the amendment.** Re-read the full
   `eigenvector_centrality_numpy` docstring (`:521-545`) and the full
   `topological_generations` docstring (`:683-702`) end-to-end after the
   amendment edits. Neither has any awkward sentence fragments, dangling
   references, or sibling-asymmetry regressions introduced by the new content.
   The `eigenvector_centrality_numpy` docstring's new "Requires a non-multigraph"
   paragraph slots between the existing scipy-rationale paragraph and the
   `:param graph:` block; the flow reads naturally. The
   `topological_generations` two-line wrap on `:raises networkx.NetworkXUnfeasible:`
   matches the long-line wrap conventions elsewhere in this file.

**Verdict on routing:** amendment closed cleanly. No `must-fix` or `should-fix`
emerged from this confirmation pass. C2 is clear to dispatch next; no second
amendment loop required.

#### Workstream C2 (Tier 2 edge link-prediction family, 6 new wrappers + factory)

```
Status: propose
API surface reviewed: hiveplotlib.graph_features.bridges,
  jaccard_coefficient, adamic_adar_index, preferential_attachment,
  resource_allocation_index, common_neighbor_centrality
  (all 6 also reachable via hiveplotlib.graph_features.networkx.*; the new
  `_make_link_prediction_wrapper` factory is internal-only, audited for the
  closures it stamps but not for surface)
Concerns:
  - [worth-discussing] `common_neighbor_centrality` accepts `alpha` (per the
    factory's `extra_kwargs="``alpha``"` arg at edge_metrics.py:232), and the
    rendered "Accepts the underlying networkx kwargs: ``ebunch``, ``alpha``."
    line tells the user the kwarg exists, but neither the kwargs line nor the
    long_description names the default value. The long_description does explain
    `alpha`'s role ("weighted by a tunable ``alpha`` parameter") but a user
    reaching for the kwarg to tune the score has to either read NetworkX's
    source or pass `alpha=...` empirically to discover the default. NetworkX's
    `nx.common_neighbor_centrality` defaults to `alpha=0.8`; surfacing that in
    the long_description ("...weighted by a tunable ``alpha`` parameter
    (default ``0.8``)...") gives the user the full picture without forcing
    them out of the docstring — at
    `src/hiveplotlib/graph_features/networkx/edge_metrics.py:226-232`.
    Suggested change: append "(default ``0.8``)" inside the parenthetical in
    the long_description so the rendered docstring reads
    "...weighted by a tunable ``alpha`` parameter (default ``0.8``)..."; no
    code-shape change, one-token edit. Auto-deferrable to Workstream E or
    v0.28.0 close-out as a docstring polish.
  - [worth-discussing] The cross-module helper import at
    `src/hiveplotlib/graph_features/networkx/edge_metrics.py:18`
    (`from hiveplotlib.graph_features.networkx.node_metrics import
    _set_to_indicator`) imports a leading-underscore private helper across
    sibling modules. Both modules currently treat the helper as node-side
    canonical (its docstring at `node_metrics.py:55` names
    `articulation_points` and `isolates` as users, both node-side). Adding
    `bridges` as a third user across the module boundary mildly breaks the
    "private helpers live with their canonical caller" pattern; a neutral home
    (`src/hiveplotlib/graph_features/networkx/_helpers.py` with both
    `_set_to_indicator` and `_partition_to_node_labels`) would be cleaner.
    Taste call, low cost; the existing import works and the `cast` at
    `edge_metrics.py:87-90` documents the cross-shape reuse intent. Not a
    blocker — at `src/hiveplotlib/graph_features/networkx/edge_metrics.py:18`.
    Suggested change: if folded, move `_set_to_indicator` and
    `_partition_to_node_labels` to a new `networkx/_helpers.py` and have both
    `node_metrics.py` and `edge_metrics.py` import from there; also update the
    helper's docstring at `node_metrics.py:55-56` to add `bridges` to the
    "Used by ..." list (the docstring currently names only the node-side
    callers). Defer to v0.28.0 close-out if there's no other helper-reorg
    pressure.
  - [low-confidence] `bridges`'s docstring at
    `src/hiveplotlib/graph_features/networkx/edge_metrics.py:64-66` describes
    bridges as "the edge-side analog of articulation points" — a strong cross-
    reference that helps a user who already understands articulation points
    map the concept onto edges. The reverse cross-reference is missing:
    `articulation_points`'s docstring at
    `src/hiveplotlib/graph_features/networkx/node_metrics.py:644-645` describes
    them as "'bottleneck' nodes in the connectivity sense" but doesn't mention
    `bridges` as the edge-side analog. Now that both wrappers exist, the
    symmetric cross-reference would round out the connectivity-bottleneck story
    for a user scanning either docstring. Low-confidence because this is
    docstring-polish, not API, and the asymmetry is mild — at
    `src/hiveplotlib/graph_features/networkx/node_metrics.py:644-645`.
    Suggested change: append "(see :py:func:`bridges` for the edge-side
    analog)" to the bottleneck sentence in `articulation_points`'s docstring.
    Defer to Workstream E or v0.28.0 close-out.
Test-method-coverage audit: clean. All 6 new wrappers have tests calling
  them by name:
    - test_bridges_on_directed_raises[DiGraph,MultiDiGraph] calls bridges
      (test file line 593)
    - test_bridges_flags_bridge_edges_on_path_graph calls bridges (line 605)
    - test_bridges_no_bridges_on_complete_graph calls bridges (line 621)
    - test_link_prediction_default_ebunch_covers_all_edges[<name>] parametrizes
      over the 5 link-prediction names and resolves each via
      `GRAPH_EDGE_METRICS[name]` then calls `fn(g)` (line 649); the parametrize
      ids are the wrapper names themselves, so the resolved test ID reads
      `test_link_prediction_default_ebunch_covers_all_edges[jaccard_coefficient]`
      etc., satisfying the test-name-contract intent (each link-prediction
      wrapper is exercised by a test whose ID names it)
    - test_link_prediction_on_directed_raises[<name>,<class>] (line 681) and
      test_link_prediction_on_multigraph_raises[<name>] (line 707) parametrize
      similarly
    - test_link_prediction_explicit_ebunch_scores_arbitrary_pairs[<name>]
      (line 734) confirms the explicit-ebunch override path; covers the
      non-default branch through the factory's `kwargs.setdefault("ebunch", ...)`
    - test_link_prediction_wrapper_docstring_carries_non_edge_caveat[<name>]
      (line 760) enforces the templated-docstring contract
    - test_make_link_prediction_wrapper_sets_name_and_qualname (line 766)
      directly exercises the factory entry point and asserts `__name__` /
      `__qualname__` / `__doc__` propagation
  The parametrized `test_edge_metric_wrapper_smoke` auto-picks up all 6 new
  keys via `_edge_metric_names()` against `GRAPH_EDGE_METRICS`. The factory
  closures are exercised through the registry rather than through direct
  `<wrapper_name>(...)` import-and-call in every test, which is the correct
  shape (the registry is the load-bearing user-facing surface; the wrappers
  are also importable as top-level names via the package `__init__`, but
  parametrizing through `GRAPH_EDGE_METRICS[name]` is the canonical pattern
  established by the prior C1 smoke tests).
```

**Walkthrough notes (for future reference, not a finding):** walked all 6 new
wrappers as a hypothetical first-time user.

1. **The link-prediction caveat reads cleanly.** Rendered `help(jaccard_coefficient)`
   lands the user on the imperative one-liner ("Return the Jaccard coefficient
   of each edge's endpoint neighborhoods."), then the "Wraps ..." provenance
   line, then the kwargs line, then the conceptual blurb explaining what
   Jaccard means and how to read it. The caveat paragraph at
   `edge_metrics.py:155-158` arrives AFTER the conceptual blurb, in the right
   position: a user already understands the score before being told the
   wrapper applies it to existing edges rather than NetworkX's non-edge
   default. The phrasing "letting you size or color edges by the predictor"
   frames the rewire positively (here's why we did this), then names the
   NetworkX default ("intends these scores for *non-edges* by default") with
   the parenthetical canonical-use explanation, then names the escape hatch
   ("pass an explicit ``ebunch=...`` to score a different edge set"). A
   NetworkX-fluent user landing here won't second-guess whether to use the
   wrapper; they get the full picture in three sentences. Walked
   `adamic_adar_index` and `preferential_attachment` separately and the same
   structure holds verbatim (factory ensures consistency across all 5).

2. **The factory pattern is invisible at the user surface.** Verified
   `__name__ = nx_func_name` (line 143), `__qualname__ = nx_func_name`
   (line 144), `__module__ = __name__` resolved to
   `hiveplotlib.graph_features.networkx.edge_metrics` (line 145), and the
   templated `__doc__` (line 146). A user typing `help(jaccard_coefficient)`
   sees a function named `jaccard_coefficient` (not `wrapper`); a user
   inspecting `jaccard_coefficient.__module__` gets the edge_metrics module
   (not the factory's introspected module). Stack traces from misuse would
   point at `wrapper` in the traceback frame (the actual `def wrapper(...)`
   line is what executes), but the user-visible name on the function object
   itself is correctly stamped. The signature `def wrapper(graph: nx.Graph,
   **kwargs)` is not opaque `*args, **kwargs` — `graph` and `**kwargs` are
   the right names for the surface. Test
   `test_make_link_prediction_wrapper_sets_name_and_qualname` enforces this
   contract directly.

3. **`bridges` indicator-dict return shape reads correctly.** Walked the
   docstring at lines 56-90 imagining a user iterating
   `for edge, value in bridges(g).items():`. First docstring line: "Return a
   per-edge boolean flag indicating bridges." (the lead is unambiguous —
   value is a bool). Second sentence in the long blurb: "NetworkX returns an
   iterator of bridge edges; this wrapper projects that set onto a
   ``dict[edge_tuple, bool]``" (named the type signature inline). The
   ``True`` / ``False`` semantics are stated parenthetically ("``True`` for
   bridges, ``False`` for everyone else"). The `:return:` line repeats it.
   Three layers of redundancy, each at a different reading depth, exactly
   matches the sibling `articulation_points` pattern. Good.

4. **`bridges` recovery message on directed.** The `ValueError` at lines
   76-82 mirrors the canonical sibling pattern from `articulation_points`
   (node_metrics.py:656-660) verbatim with the wrapper name swapped: "Build
   the source graph with `directed=False` (e.g., on the HivePlot
   initialization or `nodes_edges_to_networkx`)." A user typo'ing the graph
   type lands on a one-line fix. Good.

5. **Edge-tuple key shape.** Multigraphs are rejected by NetworkX's own
   `not_implemented_for` decorator (the factory wrappers) and by the
   wrapper's own `if graph.is_directed(): raise` guard (only undirected
   path runs `nx.bridges`). The only viable graph type that reaches
   `nx.bridges(graph)` is `nx.Graph` (undirected, non-multigraph), whose
   `graph.edges` iterates 2-tuples. The `cast` at lines 87-90 documents the
   intentional shape-widening from the helper's
   `dict[Hashable, bool]` return type to the edge-metric contract's
   `dict[Tuple[Hashable, ...], bool]` and the inline comment explains why
   the cast is safe. Test
   `test_bridges_flags_bridge_edges_on_path_graph` asserts
   `set(result.keys()) == set(g.edges)` on `nx.path_graph(5)`, where
   `g.edges` returns 2-tuples — confirms the cast doesn't mask a 3-tuple
   slip. Good.

6. **Cross-module helper import (worth-discussing flagged above).** The
   import at line 18 reaches across the sibling module boundary for a
   leading-underscore helper. The cast at lines 87-90 carries an explicit
   inline comment naming the reuse rationale, which makes the architectural
   choice legible even though it's slightly off-pattern. Acceptable for now;
   a `_helpers.py` reorg can come later.

7. **Naming consistency confirmed.** `bridges` (plural) matches the set-
   shaped sibling pattern `articulation_points` / `isolates` (both plural,
   both from C1). User Resolution item 1 in the planning round declined the
   `is_*` rename; the C2 implementation preserves that decision and the
   plural-for-set-membership convention is consistent across all three
   indicator-dict wrappers.

8. **The backend-explicit import path works.** Confirmed all 6 new keys are
   re-exported from `src/hiveplotlib/graph_features/networkx/__init__.py`
   (also the parent `__init__.py` at lines 36-45) and listed in
   `edge_metrics.py`'s `__all__` (lines 236-245). A user can
   `from hiveplotlib.graph_features.networkx import jaccard_coefficient` or
   `from hiveplotlib.graph_features import jaccard_coefficient`; both work
   identically. The registry-driven dispatcher path
   (`compute_graph_metrics(..., edge_metrics=["jaccard_coefficient"])`) is
   also live.

**Verdict on routing:** no `must-fix` or `should-fix`. All three findings are
`worth-discussing` or `low-confidence` and are auto-deferrable to Workstream E
(notebook author may surface the `alpha` default in the notebook prose) or to
v0.28.0 close-out (the cross-module helper relocation and the
`articulation_points` ↔ `bridges` cross-reference symmetry both fit naturally
into a docs-pass sweep). No need to route through orchestrator `amend-plan`;
the dispatching session can continue with Workstreams D and E. The C2 surface
ships clean as user-facing API: the link-prediction caveat is well-placed and
phrased, the factory pattern is invisible at the user surface, `bridges`
mirrors the sibling indicator-dict pattern faithfully, and the test-method
coverage is complete (all 6 new wrappers exercised by tests whose
parametrize-id includes the wrapper name).

## Workstreams

Five workstreams in recommended sequence: A → B → C, with D parallelizable against any of A/B/C, and E last so the notebook reflects the settled API. C is large enough that it splits into C1 (node-side wrappers plus adapter helpers) and C2 (edge-side link-prediction family) for clean dispatch.

### Workstream A: Move NetworkX wrappers into a `networkx/` subpackage

**Status:** complete (2026-05-25)
**Files:**

- New: `src/hiveplotlib/graph_features/networkx/__init__.py`, `src/hiveplotlib/graph_features/networkx/node_metrics.py`, `src/hiveplotlib/graph_features/networkx/edge_metrics.py`
- Removed: `src/hiveplotlib/graph_features/node_metrics.py`, `src/hiveplotlib/graph_features/edge_metrics.py`
- Edited: `src/hiveplotlib/graph_features/__init__.py` (imports + the two extend-instructions docstring lines pointing at the new module paths)
- Edited: `docs/source/autodoc/hive_plots/graph_features.rst` (two `automodule::` directives updated from `hiveplotlib.graph_features.{node,edge}_metrics` to `hiveplotlib.graph_features.networkx.{node,edge}_metrics`)
- Possibly edited: `tests/graph_features_test.py` (only if any direct import of the old submodule paths exists — current grep says no)

**Done when:**

1. `git grep "hiveplotlib.graph_features.node_metrics\|hiveplotlib.graph_features.edge_metrics"` returns only the two new file paths and the two updated import lines in `__init__.py`. No survivors anywhere else (especially not in `docs/`, `examples/`, `tests/`, or the rest of `src/`). The rendered-traceback strings inside `examples/computing_graph_metrics.ipynb` are not survivors — they regenerate on `make test-nb` (see "Patterns this replaces" above).
2. `make test` passes with the existing 12 registered metrics still parametrized through the smoke tests. Coverage 100%.
3. `make docs` builds without sphinx warnings about missing references. Confirm the rendered `Node Metrics` and `Edge Metrics` sections in `docs/source/autodoc/hive_plots/graph_features.rst` still show every wrapper's docstring (an empty section is the silent-failure mode of a wrong `automodule::` path).
4. `python -c "from hiveplotlib.graph_features import degree, edge_betweenness_centrality, GRAPH_NODE_METRICS, GRAPH_EDGE_METRICS, compute_graph_metrics; print(GRAPH_NODE_METRICS.keys())"` runs and prints the existing 10 node-metric keys.
5. The new `from hiveplotlib.graph_features.networkx import degree` import path also works (verified by ad-hoc smoke).

### Workstream B: Tier 1 NetworkX additions (11 drop-in wrappers)

**Status:** ✅ complete (closure reconcile 2026-06-18: header was stale; the Implementation log records "Workstream B complete" on 2026-05-25 and all 11 Tier 1 keys are in the shipped `GRAPH_NODE_METRICS`)
**Files:**

- `src/hiveplotlib/graph_features/networkx/node_metrics.py` — add 11 wrappers.
- `src/hiveplotlib/graph_features/__init__.py` — register the 11 keys in `GRAPH_NODE_METRICS` and add them to the `from ... import ...` line plus `__all__`.
- `tests/graph_features_test.py` — restriction-raise tests for any new wrapper that adds a restriction (e.g. `in_degree_centrality` / `out_degree_centrality` should raise on undirected; the parametrized smoke test covers the happy path automatically).
- `CHANGELOG.rst` — bullet listing the Tier 1 additions under the existing **`Added > Graph Metrics`** subsection of the v0.28.0 (WIP) entry. Do NOT create a v0.29.0 entry. The release is held open until both plans for GitLab #46 land; this plan's additions are continuations of the existing v0.28.0 graph-metrics work.

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

**Status:** complete (2026-05-25)
**Files:**

- `src/hiveplotlib/graph_features/networkx/node_metrics.py` — add 14 wrappers plus two private helpers (`_partition_to_node_labels`, `_set_to_indicator`). Optionally add the `@requires_graph_type` decorator and refactor the existing restriction checks if the boilerplate is getting noisy after Workstream B.
- `src/hiveplotlib/graph_features/__init__.py` — register the new keys.
- `tests/graph_features_test.py` — restriction-raise tests where applicable (e.g. `reciprocity` and `topological_generations` require directed graphs; `articulation_points` / `connected_components` require undirected or have specific component-counting semantics; `eigenvector_centrality_numpy` rejects multigraphs; etc.). Add an explicit shape test for the community-detection and connected-components wrappers asserting the values are int labels with at least one distinct group on a non-trivial fixture.
- `CHANGELOG.rst` — bullet for Tier 2 node-side additions, appended to the same existing **`Added > Graph Metrics`** subsection of v0.28.0 (WIP) that Workstream B writes into. Workstream E consolidates the Tier 1 / Tier 2 / refactor bullets at the end.

**Wrappers to add (node-side):**

Single-shape adaptors: `eccentricity`, `eigenvector_centrality_numpy`, `reciprocity`, `topological_generations`.

Tuple-projecting: `hits_hubs`, `hits_authorities` (one underlying NetworkX call, two registry keys).

Set-to-indicator adaptors (boolean): `articulation_points`, `isolates`.

Partition-to-labels adaptors (community family): `greedy_modularity_communities`, `louvain_communities`, `label_propagation_communities`.

Partition-to-labels adaptors (components family): `connected_components`, `strongly_connected_components`, `weakly_connected_components`.

**Done when:**

1. All 14 new keys registered, all 4 helpers live (or 5 if the decorator is folded in).
2. Parametrized smoke test still green across the whole `GRAPH_NODE_METRICS` keyset (all 10 existing + 11 Tier 1 + 14 Tier 2 = 35 node metrics).
3. Restriction-raise tests pass for the directed-only and undirected-only wrappers.
4. Shape tests for the community-detection and connected-components families assert: keys are graph nodes, values are ints, at least 2 distinct labels on a fixture with structure.
5. `_partition_to_node_labels` sorts partitions by size descending, ties broken by smallest node id. Tested directly. Each community-detection and connected-components wrapper docstring documents the label ordering so the user can rely on "label 0 = giant community" semantics.
6. `hits_hubs` and `hits_authorities` wrapper docstrings explicitly note that requesting both metrics in one `compute_graph_metrics` call runs the underlying `nx.hits(G)` iteration twice (no caching layer per User Resolution item 4).
7. `make test`, `make ty`, `make format`, `make docs` all green.
8. CHANGELOG bullet added.
9. api-critic post-impl review filled.

### Workstream C2: Tier 2 edge link-prediction family

**Status:** complete (2026-05-25)
**Files:**

- `src/hiveplotlib/graph_features/networkx/edge_metrics.py` — add 6 wrappers: `bridges` (set-to-indicator) plus the 5 link-prediction wrappers built off the `_make_link_prediction_wrapper(nx_func_name)` factory (`jaccard_coefficient`, `adamic_adar_index`, `preferential_attachment`, `resource_allocation_index`, `common_neighbor_centrality`).
- `src/hiveplotlib/graph_features/__init__.py` — register the 6 keys in `GRAPH_EDGE_METRICS`.
- `tests/graph_features_test.py` — parametrized smoke covers everything; add a restriction-raise test for `bridges` (undirected only per NetworkX), and verify the link-prediction wrappers' return shape on a small fixture (dict keyed by 2-tuples, float values, populated over all edges in the graph).
- `CHANGELOG.rst` — bullet for Tier 2 edge-side additions, appended to the same existing **`Added > Graph Metrics`** subsection of v0.28.0 (WIP). The bullet should call out the link-prediction additions specifically, since they reverse the prior plan's deliberate exclusion of that family (see "Prior ADRs / design docs" above for the reasoning).

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

**Status:** ✅ complete (closure reconcile 2026-06-18: header was stale; the Implementation log records "Workstream D complete" on 2026-05-25, shipped as item 6 "Optional `igraph` backend for `compute_graph_metrics`" in `docs/source/roadmap.rst`)
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

**Status:** ✅ complete, but the notebook narrative was SUPERSEDED (closure reconcile 2026-06-18: header was stale). E shipped (Implementation log entries on 2026-05-25 for the CHANGELOG sweep and the notebook cells, plus amendment 1's `HivePlotMatrix.from_partition` swap). The CHANGELOG sweep stands as-shipped. The notebook cells below, however, were later replaced: see the "Closure reconcile" note at the end of this workstream. The cell-level "Recommended additions" and "Done when" criteria below describe a Louvain / Les-Mis narrative that is NOT in the shipped notebook; they are obsolete, not outstanding.
**Files:**

- `examples/computing_graph_metrics.ipynb` — add small focused cells (per notebook-author skill, prefer additive over rewrite) demonstrating two or three of the highest-value new metrics. Recommended additions:

  - A "**Community detection as a partition variable**" cell pair: compute `louvain_communities` on Karate Club, use the resulting integer-label column as `partition_variable` directly (no discretization needed since labels are already integers / categorical-like). Story: Louvain recovers the historical `club` split on a familiar dataset (compare the computed label column with the ground-truth `club` attribute as a sanity check). This is the headline new capability.
  - A "**Precompute once, reuse across plots**" cell pair using `nx.les_miserables_graph()`. Compute `louvain_communities` and `harmonic_centrality` once with `compute_graph_metrics`, then build two `HivePlot` instances from the same annotated `nodes` / `edges` (one partitioning on community, one sorting by harmonic centrality). Les Mis is the better dataset here than Karate Club because the data is not pre-labeled, so Louvain reveals plot threads rather than just recovering a known partition. This cell also makes the cost story explicit ("community detection is expensive on bigger graphs; compute once, plot many times").
  - A "**Harmonic centrality and average neighbor degree**" cell pair: compute both Tier 1 metrics, show the resulting DataFrame head. Demonstrates a multi-metric init that's lighter than the existing pagerank / betweenness example.
  - A "**Link prediction edge coloring**" cell pair: compute `jaccard_coefficient` and color edges by it. Mirrors the existing `edge_betweenness_centrality` section. Markdown around the cell should reiterate the docstring's note that the values are link-likelihood scores applied to existing edges.

- `examples/computing_graph_metrics.ipynb` — add a short markdown cell near the top mentioning NetworkX's backend dispatch ecosystem (cuGraph, graphblas-algorithms, nx-parallel) as a one-paragraph note: "hiveplotlib computes metrics by calling NetworkX functions; users who install NetworkX backend plugins like `graphblas-algorithms` get those accelerated implementations for free when their version of NetworkX supports backend dispatch." No code change, just a forward-pointer. The prior research findings flagged this as worth one paragraph.

- `CHANGELOG.rst` — consolidate the Tier 1 / Tier 2 entries from Workstreams B / C1 / C2 into a clean section structure inside the existing v0.28.0 (WIP) entry. Do NOT create a v0.29.0 entry. Recommended shape: one "Tier 1 additions" sub-bullet, one "Tier 2 additions" sub-bullet, one "Refactor" line for Workstream A's `graph_features.networkx/` subpackage move. Update the metric counts in the existing **`Added > Graph Metrics`** subsection of v0.28.0 (currently reads "10 node-metric wrappers" and "2 edge-metric wrappers" at `CHANGELOG.rst:54-57`) to the new totals (~35 node-metric wrappers, ~8 edge-metric wrappers). Per writing-voice rules, no em-dashes, no AI filler phrases, length-discipline.

**Done when:**

1. `make test-nb` runs `examples/computing_graph_metrics.ipynb` end-to-end and passes (notebooks run as part of CI / test-all). All new cells execute cleanly.
2. Notebook is only edited at `examples/computing_graph_metrics.ipynb`; `docs/source/notebooks/` and `docs/source/gallery_examples/` are not touched (auto-generated on docs build per CLAUDE.md).
3. New markdown cells follow the writing-voice rules (no em-dashes, no "moreover" / "furthermore" / "in essence", direct voice). Length matches sibling sections.
4. CHANGELOG counts updated correctly and reflect the final shipped state.
5. `make docs` builds the notebook into `docs/source/gallery_examples/` correctly.
6. api-critic skim (optional) confirms the notebook examples match the imagined user task.

**Closure reconcile (2026-06-18): the Workstream E notebook narrative was superseded; the shipped notebook uses a degree-discretization example.**

The Implementation log entries for Workstream E (and amendment 1) describe a notebook narrative built on (1) "Community detection as a partition variable" using `louvain_communities` on Karate Club, (2) a "Precompute Once, Reuse Across Plots" section on `nx.les_miserables_graph()`, and (3) a `HivePlotMatrix.from_partition` swap for the Louvain cells. **That narrative is no longer in the shipped `examples/computing_graph_metrics.ipynb`.** The later graph-metrics notebook-restructure work (tracked in its own plan / MR) replaced it: the shipped notebook now partitions Zachary's Karate Club by node `degree`, discretized into three bins (`"low"` / `"medium"` / `"high"`) via `create_partition_variable`, rather than by Louvain community on Karate Club / Les Mis. Verified against the shipped notebook on branch `46-...`: zero `louvain` mentions, zero `les_miserables` mentions, the degree-discretization story present. The `harmonic_centrality` / `average_neighbor_degree` multi-metric cell and the `jaccard_coefficient` link-prediction-edge-coloring cell survived the restructure.

Consequence for this plan: Workstream E's original cell-level "Recommended additions" and "Done when" criteria (the Louvain-as-partition, Les-Mis-precompute, `HivePlotMatrix`-vs-3-axis-`HivePlot` contrast) are **obsolete, not outstanding work**. The notebook objective (demonstrate the highest-value new metrics with a backend-dispatch forward-pointer) was met; the specific datasets and partition story changed downstream. The ADR should describe the as-shipped notebook (degree-discretization), not the Louvain/Les-Mis narrative this plan's log captured.

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

### 2026-05-25: In-scope tweak — fix Example 2 to use shipped `HivePlot(graph=G, ...)` surface

**Trigger:** api-critic raised a `must-fix` in the refinement-round addendum (see "API Critic's take (planning mode) > Refinement-round addendum (planning mode, second pass)" above). Example 2 at plan line 117 called `HivePlot.from_networkx(G, ...)`, a classmethod that does not exist on the shipped surface. The prior plan's Workstream I (i-want-to-plan-optimized-hoare.md) removed `HivePlot.from_networkx` and replaced it with a keyword-only `graph=` parameter on `HivePlot.__init__`. The shipped guard at `src/hiveplotlib/hiveplot.py:1919-1924` catches positional `nx.Graph` bound to `nodes`, but a call to the missing classmethod fails with `AttributeError: type object 'HivePlot' has no attribute 'from_networkx'` before reaching the guard. The canonical pattern is documented in `examples/computing_graph_metrics.ipynb` and `CHANGELOG.rst:27-33`.

**Edit applied:** Example 2's construction line rewritten from `hp = HivePlot.from_networkx(G, ...)` to `hp = HivePlot(graph=G, ...)`. All other example content preserved (intent: link-prediction edge metric used to color edges, demonstrating the C2 reintroduction). One-line change.

**Cascade resolution:** Examples 3 and 4 implicitly carry `hp` from Example 2 (both call `hp.compute_graph_metrics(...)`). With Example 2 now producing a valid `HivePlot` instance via the shipped surface, the post-hoc `compute_graph_metrics` method calls in Examples 3 and 4 resolve naturally against the existing API (shipped in the prior plan; see plan line 9). No further edits needed for Examples 3-4.

**Other api-critic findings from the refinement round (not acted on, status notes):**

- The `[worth-discussing]` item about Examples 2 → 3 → 4 implicit `hp` carryover is dissolved by this fix (the critic explicitly noted this concern dissolves once Example 2 is corrected). The optional one-line "(reuses `G`, `hp` from Example 2)" comment in Examples 3-4 is the user's call; not applied here.
- The three `[low-confidence]` items (Example 5 metric-name difference vs Example 1; Example 4's `graph_directed=False` matching Workstream C1's restriction story; Workstream E's notebook prose reiterating the link-prediction docstring caveat) are advisory only. The first needs no edit. The second resolves naturally when the C1 engineer reads NetworkX's `articulation_points` docs at implementation time and lands the right restriction. The third is already covered by Workstream E's bullet at plan line 419; flagged for the notebook-author specialist to actually land the prose at execution time.

**Status:** Ready for first workstream dispatch.

### 2026-05-25: In-scope tweak — close C1 done-when #3 (`eigenvector_centrality_numpy` multigraph guard) plus two adjacent low-cost fixes

**Trigger:** api-critic post-implementation review of Workstream C1 (see "API Critic — post-implementation review > Workstream C1" subsection above) raised one `should-fix` (the multigraph guard gap) plus two `worth-discussing` items the dispatching session asked be folded in. The `should-fix` was upgraded to `must-fix` for routing because Workstream C1's done-when criterion #3 explicitly required "`eigenvector_centrality_numpy` rejects multigraphs" and the wrapper shipped without the guard. Without it, a caller passing a MultiGraph hits NetworkX's `nx.to_scipy_sparse_array` path which sums parallel-edge weights into a single adjacency entry, returning silently-different-than-expected values rather than raising. This is the worst failure mode the Workstream B metric-table classifier extension tried to close.

All three items are file-contiguous (same source file, same CHANGELOG entry) and the test additions are mechanical, so bundling into one in-scope tweak is the right shape rather than spreading across C2 dispatch or deferring to Workstream E. C1's status header stays `complete (2026-05-25)`; this amendment ships as its own discrete scope ("C1 ship + amendment 1") rather than reopening C1.

**Scope (three changes, all in `src/hiveplotlib/graph_features/networkx/node_metrics.py` plus tests and CHANGELOG):**

1. **Add the missing `eigenvector_centrality_numpy` multigraph guard.** Mirror the sibling `eigenvector_centrality` pattern at lines 174-180. Insert an `if graph.is_multigraph(): raise ValueError(...)` block before the `return nx.eigenvector_centrality_numpy(graph, **kwargs)` line at `:521-537`, using the canonical recovery-message template ("Build the source graph with `multigraph=False` (e.g., on the HivePlot initialization or `nodes_edges_to_networkx`).") with the wrapper name swapped. Update the docstring at `:521-537` to add the "Requires a non-multigraph" paragraph (mirroring `eigenvector_centrality:164-167`), the `:param graph:` qualifier `(not a multigraph)`, and a `:raises ValueError: if \`\`graph\`\` is a multigraph.` line. The metric-table classifier should pick up the new restriction automatically via its existing probe approach (the classifier widening in C1 covered `NetworkXException`; the new ValueError from the guard fires in the same probe path that classified `eigenvector_centrality`).

2. **Fix `topological_generations` `:raises:` block (worth-discussing #1).** At `src/hiveplotlib/graph_features/networkx/node_metrics.py:684`, the structured `:raises:` block only lists `:raises ValueError:` but the prose paragraph at `:678-680` mentions `NetworkXUnfeasible` is raised by the underlying networkx implementation on cyclic graphs. Sphinx renders the structured block prominently and does not auto-promote prose. Add a second `:raises:` line — `:raises networkx.NetworkXUnfeasible: if \`\`graph\`\` contains a cycle (raised by the underlying networkx implementation).` — matching the existing `:raises ValueError:` line's shape.

3. **Correct count drift in narration (worth-discussing #2).** `CHANGELOG.rst:69` reads "13 additional NetworkX-backed node-metric wrappers"; actual count is 14 (the C1 implementation-log entry enumerates the 14 names and the registry holds 35 total = 10 prior + 11 Tier 1 + 14 Tier 2). Edit `CHANGELOG.rst:69` from `13 additional` to `14 additional`. Also edit the C1 implementation-log entry from `13 Tier 2 ... 34 keys total: 10 prior + 11 from B + 13 new` to `14 Tier 2 ... 35 keys total: 10 prior + 11 from B + 14 new`. Pure narration fix; no code change.

**Tests:**

- Add a new `test_eigenvector_centrality_numpy_on_multigraph_raises` test in `tests/graph_features_test.py`, parametrized over `nx.MultiGraph` and `nx.MultiDiGraph` like the existing `test_eigenvector_centrality_on_multigraph_raises` test (find it via grep — it's the canonical sibling pattern). The new test should assert `ValueError` with a message containing "does not support multigraphs" or similar discriminating substring. The parametrized smoke test already routes `eigenvector_centrality_numpy` through `_undirected_fixture()` (a plain `nx.Graph`), so the smoke test continues to pass unchanged.
- No new test for the `topological_generations` `:raises:` docstring fix (docstring-only change).
- No new test for the CHANGELOG count fix (narration-only change).

**Done when:**

1. `eigenvector_centrality_numpy` raises `ValueError` on `MultiGraph` / `MultiDiGraph` with a message matching the sibling recovery template; the new parametrized restriction-raise test passes.
2. `make test` green at 100% coverage (the new guard branch needs to be covered by the new restriction-raise test).
3. `make ty`, `make format` clean.
4. `make docs` builds with `topological_generations`'s `:raises:` block rendering both ValueError and NetworkXUnfeasible entries. The metric-table classifier renders `eigenvector_centrality_numpy` with "Requires a non-multigraph graph." (or whatever sentence the classifier emits for the multigraph-only restriction — sibling `eigenvector_centrality` carries the same restriction and the classifier should classify both identically).
5. `CHANGELOG.rst:69` reads `14 additional` and the C1 implementation-log entry reads `14 Tier 2 ... 35 keys total`.
6. api-critic post-impl confirmation appended to this plan as a sub-block under "Workstream C1 — amendment 1 follow-up" verifying the three changes landed correctly.

**Items NOT acted on in this amendment (deferred to Workstream E or v0.28.0 close-out):**

- `[low-confidence] reciprocity no-kwargs visible signal`: the api-critic noted that `def reciprocity(graph)` (no `**kwargs`) is the visible signal that subset-reciprocity isn't supported via this wrapper; the docstring already carries the load. No edit required; defer for Workstream E's notebook author in case a "compute reciprocity on a subset" example is added.
- `[low-confidence] isolates docstring silent on graph-type`: `isolates` works on all four graph types but the docstring doesn't say so explicitly, asymmetric with `articulation_points`'s explicit constraint paragraph one above. Defer to Workstream E or v0.28.0 close-out as part of a possible "backfill all unrestricted wrappers (`degree`, `pagerank`, etc.) with an explicit one-line constraint statement" pass. Not a per-wrapper urgency.

**Status:** Ready for dispatch. Recommended sequence: code-engineer first (single dispatch covering all three changes), then qa-engineer (verifies tests + coverage + docs build), then api-critic post-impl confirmation (short — verifying the guard, the `:raises:` block, and the narration count fix all match the sibling-pattern shape). Workstream C2 can dispatch in parallel with this amendment because C2 touches `src/hiveplotlib/graph_features/networkx/edge_metrics.py` and this amendment touches `node_metrics.py`; the only shared file is `CHANGELOG.rst` (C2 appends a new Tier 2 edge-side bullet, this amendment edits the existing line 69 narration count for the node-side bullet), and that conflict is mechanical and resolvable at PR-merge time. If the dispatching session prefers strict serialization, dispatch this amendment first and C2 second.

### 2026-05-25: Workstream E amendment 1 — switch multi-community plots to `HivePlotMatrix.from_partition` (plus bundled polish)

**Workstream:** E (status stays `in_progress` until this amendment ships + viz-critic re-review clean).

**Trigger:** viz-critic post-impl walk of the 4 new hive plots added by Workstream E flagged two `must-fix` items grounded in the viz-quality-bar skill and the corpus's own conventions:

1. **Cell 17 (Karate Club Louvain partition, 4 axes).** Default-resolution Louvain finds 4 communities on Karate Club; the notebook-author authored a 4-axis `HivePlot` to demonstrate "integer labels go directly into `partition_variable` without binning." The hive-plot 2-3 axis rule is load-bearing — a 4-axis plot does not show every pair of axes without overlap, which is the property that makes hive plots a rational layout in the first place. The corpus's own `examples/hive_plots_more_than_three_groups.ipynb` (lines 10-12) opens with "Throughout the `hiveplotlib` documentation, we've routinely partitioned networks into 3 groups. This choice was intentional because **a 3-axis hive plot shows the edges between every pair of axes without any overlap.**" and the same notebook's line 1327 calls Option 1 (just adding axes) "particularly disingenuous." The canonical answer in the corpus for partition-count > 3 is `HivePlotMatrix.from_partition`, which renders the four communities as a 4 x 4 upper-triangular matrix of 2-axis pairwise comparisons plus 4 single-group diagonals.

2. **Cell 23 (Les Misérables Louvain partition, 6 axes).** Same root cause at greater scale; 6 axes on a 77-node graph compounds the issue.

Notebook-author's framing was "axis-count variance is data-motivated, not invented" — they treated the natural community count as a constraint. Viz-critic's stance (which I concur with) is that the natural community count is a signal to switch the visualization primitive to `HivePlotMatrix`, not to compromise the hive-plot invariant. Crucially, the QA pass already concurred with notebook-author that the honest "Louvain finds 4 communities on Karate / 6 on Les Mis, supported by the cross-tab" framing is the right call — forcing 2 communities via `resolution=0.5` would compromise that pedagogy. The fix must preserve the honest community-count framing while switching the primitive.

**Shape chosen:** Option (a) — swap `HivePlot(graph=G, ...)` for `HivePlotMatrix.from_partition(graph=G, ...)` in the two flagged cell pairs and adjust the surrounding markdown to frame the matrix view. The kwarg shape is mechanically clean: confirmed by reading `src/hiveplotlib/hiveplot_matrix.py:913-947`, `HivePlotMatrix.from_partition` accepts the same `graph=`, `partition_variable=`, `sorting_variables=`, `node_graph_metrics=`, `node_graph_metric_kwargs=` parameters the existing cells already use. The pedagogical thrust survives intact: `partition_variable="louvain_communities"` still resolves the integer-label column directly without binning on `HivePlotMatrix` exactly as it does on `HivePlot`, and the cross-tab cell against `club` ground-truth stays unchanged (the partition column is the same column either way; we're just changing the visualization primitive that reads it).

Options (b) and (c) were considered and rejected:

- **(b) Force 2-3 communities via `louvain_communities(..., resolution=0.5)`.** Rejected because it compromises the "Louvain finds finer-grained substructure than the historical 2-way split, but respects the club boundary as a sanity check" framing that notebook-author and QA both already concurred on. The cross-tab pedagogy (4 communities vs. 2 clubs with one crossover) is load-bearing and honest.
- **(c) Replace with a connected-components example or a different graph where Louvain naturally finds 2-3.** Rejected because (1) the Karate Club + Les Misérables pairing is itself pedagogically motivated (Karate as the familiar-pre-labeled dataset, Les Mis as the no-ground-truth dataset where Louvain reveals plot threads — this contrast is the reason both were chosen in the planning round per User Resolution item 6), and (2) replacing the demo entirely is a meaningful pedagogical decision that would benefit from user weigh-in; option (a) preserves the existing pedagogy while resolving the viz issue, so it ships cleaner without surfacing.

**Scope (three changes, all in `examples/computing_graph_metrics.ipynb`):**

1. **Cell pair around id `community-as-partition-md` / `community-as-partition-code`.** Rewrite the construction from `HivePlot(graph=G, partition_variable="louvain_communities", ...)` plus `fig, ax = hp_louvain.plot()` to `HivePlotMatrix.from_partition(graph=G, partition_variable="louvain_communities", sorting_variables="degree", node_graph_metrics=["degree", "louvain_communities"], node_graph_metric_kwargs={"louvain_communities": {"seed": 0}})` plus the matching `hpm_louvain.plot()` call (the `HivePlotMatrix` plot API returns the figure and the 2D `axes` array per the corpus's existing matrix usage; notebook-author should match the canonical `hpm_*.ipynb` plot pattern). Drop the `repeat_axes=True` kwarg (HivePlotMatrix handles diagonal repeats automatically via `include_diagonal=True`, which is the default per `hiveplot_matrix.py:924`). Update the title from `"Karate Club, partitioned by Louvain community"` to a matrix-appropriate framing (`set_title` on the `fig` or via `fig.suptitle` per HPM convention; notebook-author can match the existing HPM tutorial style). Markdown lead-in needs one sentence reframing: replace "Below, we run Louvain on the same graph, use its integer labels as the partition, and cross-tab the result against `club` to see how the detected communities relate to the historical split:" with prose explaining that Louvain finds 4 communities so we use `HivePlotMatrix.from_partition` to lay them out as pairwise comparisons (4 x 4 upper-triangular matrix), preserving the "integer labels go directly into `partition_variable` without binning" pedagogical point. The cross-tab cell (`community-as-partition-xtab-code`) stays unchanged — it reads `hp_louvain.nodes.data` which is the same DataFrame either way (rename to `hpm_louvain.nodes.data` if the variable is renamed); the cross-tab framing ("Louvain finds finer-grained substructure than the historical 2-way split, but respects the club boundary as a sanity check") stays verbatim and lands correctly.

2. **Cell pair around id `precompute-reuse-plot1-md` / `precompute-reuse-plot1-code`.** Same shape as change 1, but for Les Mis: rewrite `HivePlot(nodes=nodes_lesmis, edges=edges_lesmis, partition_variable="louvain_communities", sorting_variables="harmonic_centrality", repeat_axes=True)` to `HivePlotMatrix.from_partition(nodes=nodes_lesmis, edges=edges_lesmis, partition_variable="louvain_communities", sorting_variables="harmonic_centrality")`. Drop `repeat_axes=True`. Update the title and markdown lead-in to match the matrix framing (6 communities → 6 x 6 upper-triangular matrix). The second Les Mis plot in this section (`precompute-reuse-plot2-md` / `precompute-reuse-plot2-code`, the 3-axis HC-bin) stays as `HivePlot` — it's a 3-axis case, the canonical hive plot pattern, and viz-critic flagged it as `good`. The "precompute once, reuse across plots" section lead-in already states "two hive plots from the same `nodes` and `edges`"; that framing survives unchanged, except update to "one as a hive plot matrix (the 6-community Louvain partition), one as a 3-axis hive plot (the harmonic-centrality bin)" or equivalent — notebook-author's call on the exact phrasing. The cost story ("computing the metrics once is faster than recomputing per plot") stays load-bearing and is unaffected by the primitive switch.

3. **Bundled polish from viz-critic's `worth-discussing` items.** Two adjacent low-cost edits, file-contiguous with the above changes, so folded in to avoid a second viz-critic re-walk:
   - **Cell 31 (Karate Jaccard, cividis colorbar) `worth-discussing` on midtone readability.** Viz-critic noted cividis's midtones can wash out at this clim range. Adjust the `clim` from `(0, 0.5)` to a tighter range if the actual Jaccard distribution on Karate Club concentrates below 0.5 (notebook-author should inspect `hp_jaccard.edges.data["jaccard_coefficient"].describe()` and pick a clim that uses the colormap's contrast range; if the distribution is already well-suited to `(0, 0.5)`, leave it). Alternatively, switch to a perceptually-cleaner sequential colormap (`viridis` or `plasma`) — but viz-critic noted the matplotlib appendix in the corpus's viz-quality-bar skill calls `cividis` the path-of-least-resistance sequential default for accessibility, so prefer adjusting the clim over swapping the colormap. Notebook-author's call which lever to pull.
   - **Cell 31 title whitespace `low-confidence` aesthetic.** Viz-critic flagged the `y=0.75` title position as creating visible whitespace above the title. The `y=0.75` value matches the existing `edge_betweenness_centrality` sibling cell exactly (notebook-author flagged this as intentional sibling-matching), so leaving it as-is is defensible. If notebook-author wants to tighten it, adjust both cells in lockstep to maintain visual consistency. Defer if it would break the sibling-match invariant.

**Items NOT acted on (carry-overs to v0.28.0 close-out):**

- **Viz-critic's partial dissent on "matches existing notebook style".** Viz-critic agreed text-styling polish matches; axis-count discipline of the existing notebook (all prior figures are 2-axis or 3-axis) does not. The two `must-fix` items in this amendment resolve the axis-count discipline issue; no separate action needed.
- **No other `worth-discussing` items from viz-critic deferred to close-out.** All non-must-fix items either fold into this amendment (cividis clim, title whitespace) or were already `good` (cells 25 and 31's core layout).

**Done when:**

1. The two flagged cell pairs use `HivePlotMatrix.from_partition` with the existing `partition_variable="louvain_communities"` pedagogical framing preserved.
2. `make test-nb` runs `examples/computing_graph_metrics.ipynb` end-to-end and passes; all matrix plots render correctly (no empty cells unless `include_diagonal=False` is explicitly chosen, which it shouldn't be here).
3. The cross-tab cell (`community-as-partition-xtab-code`) still reads correctly against whatever variable name the Karate Louvain cell binds (`hp_louvain` → `hpm_louvain` is the natural rename); the 4-row crosstab output (4 Louvain communities × 2 club values) is unchanged in semantics.
4. Voice-rule scan clean on edited markdown cells (no em-dashes, no AI filler, length matches sibling sections per CLAUDE.md global writing-voice rules).
5. viz-critic re-review post-impl returns `good` on both cells (or `worth-discussing` only — no must-fix).
6. CHANGELOG no edit required (this is a notebook fix to an unshipped v0.28.0 entry; the existing "examples notebook" line in Workstream E's CHANGELOG sweep already covers the surface).

**Status:** Ready for dispatch. Recommended sequence: notebook-author first (single dispatch covering all three cell-pair changes plus the bundled polish), then qa-engineer (`make test-nb` + voice-rule scan + cross-tab semantics check), then viz-critic post-impl re-review (walks the two now-matrix cells against the viz-quality-bar skill, confirms the must-fix items are closed and no new findings introduced). No user weigh-in required, option (a) preserves the existing pedagogy and the kwarg shape is mechanically clean against the shipped `HivePlotMatrix.from_partition` surface. Workstream E status stays `in_progress` until viz-critic re-review returns clean, then advances to `complete (2026-05-25)`.


##### Viz Critic re-review: Workstream E amendment 1 (2026-05-26)

```
Status: clean (must-fix items closed cleanly, no new findings introduced)
Figures reviewed:
  - examples/computing_graph_metrics.ipynb cell 17 (Karate Louvain HPM, 4x4 upper-triangular)
  - examples/computing_graph_metrics.ipynb cell 23 (Les Misérables Louvain HPM, 6x6 upper-triangular)
  - examples/computing_graph_metrics.ipynb cell 25 (Les Misérables HC-bin 3-axis HivePlot, unchanged sibling)
  - examples/computing_graph_metrics.ipynb cell 31 (Karate Jaccard cividis edge-coloring, polish-tightened)
Polish budget: instructional for all four (caption demonstrates an API surface); HPM
  cells inherit the HPM-mid layout-discipline budget.
Concerns:
  - [worth-discussing] HivePlotMatrix.from_partition coerces row/col labels to strings,
    at src/hiveplotlib/hiveplot_matrix.py:1220. Pre-existing, not introduced by this
    amendment. Visible in cells 17 and 23.
```

**Walkthrough notes (must-fix closure confirmation):**

1. **Hive-plot 2-3 axis rule honored at the cell level.** Both HPMs decompose the multi-community partition into pairwise 2-axis hive plots (off-diagonal cells with the `Other` collapse axis) plus single-axis-with-repeat diagonals. Each cell is a 2-axis or 3-axis hive plot, so the canonical hive-plot invariant (every pair of axes shown without overlap) is preserved per cell. This is the corpus documented answer for partition counts > 3 (`examples/hive_plots_more_than_three_groups.ipynb` lines 10-12). The prior `must-fix` (4-axis Karate, 6-axis Les Mis) is closed cleanly.

2. **4x4 Karate matrix reads comfortably.** Ten populated cells (4 diagonal + 6 upper-triangle off-diagonal) at typical notebook render size. Row and column labels sit cleanly outside the grid; the upper-triangular layout (lower-left blank) is correct and matches the canonical HPM `from_partition` shape. Diagonal cells with `include_diagonal=True` show each community vs. itself via the repeat-axis mechanism; intra-community structure varies meaningfully across the four communities (community 0 is the largest cluster, community 3 is sparse with 4 nodes per the cross-tab), which the diagonal panels convey at a glance.

3. **6x6 Les Mis matrix is dense but legible.** Twenty-one populated cells (6 diagonal + 15 upper-triangle off-diagonal) on a single page. The cells become small at this density, but each remains visually distinguishable: communities 0-3 carry the dense core of the co-appearance network, communities 4-5 are sparser and read clearly as the smaller end. The off-diagonal cells whose communities have no co-appearance edges (the `Other`-collapse pattern with one or two arcs) read cleanly even at small size. Density is borderline but not over the line; pushing past ~8x8 would be the natural threshold for routing to datashader or filtering, but 6x6 sits inside the HPM `unify_axes=True`-without-density-relief regime.

4. **Diagonal repeat-axis cells working as intended.** In both matrices, the diagonal cells show each community intra-group structure via the repeat-axis mechanism (one axis label appears twice on each diagonal panel). The `_repeat` suffix is stripped from display per `hiveplot_matrix.py:1215-1217`, so the visual reads cleanly as "community N on both axes." No two-tone coloring on the diagonals since the off-diagonal cells already carry the inter-vs-intra contrast at the matrix level (different cells = different inter-community pairs).

5. **No `repeat_axes` two-tone gotcha at the matrix level.** The prior expertise gotcha about "repeat axes without two-tone coloring is undifferentiated spaghetti" does not apply here because the matrix structure itself does the inter-vs-intra differentiation (diagonal = intra, off-diagonal = inter), not the edge coloring within a single hive plot. The amendment drop of `repeat_axes=True` (HPM handles diagonal-repeats via `include_diagonal=True`) is the correct shape.

6. **Cell 31 cividis polish reads cleanly.** The tightened `clim=(0, 0.3)` puts the bulk of the Jaccard distribution (per the brief, median ~0.11, Q90 ~0.30) inside the colormap contrast range. Low-Jaccard edges read as dark blue, mid as olive/khaki, high as bright yellow with strong perceptual separation across the actual data range. The `extend="max"` arrow at the right end of the colorbar signals the off-end values (top ~5%, max 0.526) honestly: a reader sees "the saturated yellow edges are above 0.3, not exactly 0.3" without the colorbar lying about a clipped range. The choice to tighten the clim rather than swap the colormap honored the corpus `cividis` convention (matplotlib appendix in the viz-quality-bar skill).

7. **No visual conflict with sibling betweenness cell.** Cell 27 (`plasma`, edge betweenness, `clim=(0, 0.13)`) and cell 31 (`cividis`, Jaccard, `clim=(0, 0.3)`) sit a few cells apart. The two colormaps are visually distinct enough (`plasma` purple-to-yellow vs. `cividis` blue-to-yellow) that a reader does not confuse them, and both colorbars carry their own labels ("Edge Betweenness Centrality" vs. "Jaccard Coefficient") so the metric identity is unambiguous. The corpus convention of one distinct colormap per metric per section is honored.

8. **Title placement clean.** `fig.suptitle("...", y=1.02, size=16)` on the two HPM cells matches the canonical `hpm_*.ipynb` pattern; title whitespace closes correctly above the matrix grid. Cell 31 `y=0.75` for the Jaccard `set_title` preserves the sibling-match invariant with cell 27 identical `y=0.75` for the betweenness cell (per the prior expertise note: title `y` is render-aspect-dependent and the colorbar vertical extent is the cause of the whitespace, not the y value).

**Concur on the precompute-reuse juxtaposition pedagogy.** Concur. The matrix-for-community / 3-axis-for-bin pairing is a stronger pedagogical story than two flat hive plots would have been:

- The matrix view (`HivePlotMatrix.from_partition`, 6 communities to 6x6) frames the Louvain output as "pairwise relationships among communities" and decomposes naturally because the partition values are unordered categorical labels with no "low to high" semantics.
- The 3-axis view (`HivePlot` with `axes_order=["low", "medium", "high"]`) frames the HC-bin output as "ordinal progression along a centrality axis" and works as a 3-axis hive plot because the bins carry a natural ordering and 3 bins is the canonical hive-plot count.
- The contrast between the two visualization primitives ties to the contrast between the two metric types (categorical community labels vs. ordinal centrality bins), which is the pedagogical payload of the `Precompute Once, Reuse Across Plots` section.
- The cost story ("computing the metrics once is faster than recomputing per plot") stays load-bearing across both views.

The notebook-author claim is correct: this is a stronger story than two 3-axis hive plots (which would have flattened the community pedagogy) or two matrices (which would have wasted the natural 3-axis fit of the HC-bin case).

**String row/col labels: ship as-is for v0.28.0; defer the int-preserving fix.** Cosmetic-leaning, not blocking:

- The labels render as integer-looking strings in both Karate (4 communities) and Les Mis (6 communities) HPMs. A reader sees integer-looking strings positioned exactly where they would expect integer community labels to sit. The visual semantics are correct.
- Confusion risk for the canonical Louvain workflow is low: the cross-tab cell immediately below the Karate matrix uses the same column (`hpm_louvain[0, 0].nodes.data["louvain_communities"]`) and that DataFrame view shows the underlying int dtype. A user comparing the rendered matrix labels to the cross-tab will see numerically-matching values either way.
- Where the label dtype would matter is in code that indexes into the HPM by string label vs. int (e.g. positional vs. label-based indexing), but the notebook only uses integer positional indexing (`hpm_louvain[0, 0]`) which works against the underlying `_hive_plots` grid regardless of label dtype. So the user-facing surface in the notebook does not hit the str-vs-int question.
- The fix at `hiveplot_matrix.py:1220` (`group_labels = [str(g) for g in partition_values]`) would be a one-line change to preserve dtype, but it touches the rendered output of every existing `from_partition` call and may interact with downstream matplotlib text rendering (which auto-stringifies anyway). The change is low-risk on its face but warrants a focused dispatch with regression-test coverage rather than folding into this amendment.
- Routing: defer to v0.28.0 close-out as a `worth-discussing` cosmetic polish item. If a separate code-engineer dispatch wants to take it, the change is mechanical (`group_labels = list(partition_values)`) plus a test asserting the dtype round-trips correctly. Not a blocker for shipping.

**Verdict:** must-fix items from the prior viz-critic walk are closed cleanly. No new must-fix or should-fix findings introduced by the amendment. One pre-existing `worth-discussing` finding (string row/col label coercion) surfaced for the first time during this re-walk; the recommendation is to ship v0.28.0 as-is and defer the dtype-preservation polish.

**Routing recommendation:** close Workstream E. Workstream E status advances to `complete (2026-05-26)`. Optional: surface the string-label finding as an item in the v0.28.0 close-out review for the qa-engineer checklist.


### 2026-05-25: New bugfix amendment, `HivePlotMatrix.from_partition` integer-partition collapse path (blocks Workstream E amendment 1)

**Trigger:** notebook-author was dispatched against Workstream E amendment 1 and surfaced back via mental-model rule 9 with `STATUS: BLOCKED`. Attempting `HivePlotMatrix.from_partition(graph=G, partition_variable="louvain_communities", ...)` on both Karate Club (4 communities) and Les Misérables (6 communities) raises `KeyError: np.int64(0)` during cell construction. Notebook-author reproduced the failure mechanically against a synthetic 3-group int partition as well, confirming the failure is independent of dataset size and only requires (a) `from_partition` with (b) more than 2 unique int values in the partition column. The bug also reproduces against any user code calling `HivePlot(..., axes_order=[int_a, int_b, None], partition_variable=<int-column>)` directly, since `from_partition` is one path into the shared `set_axes_order` collapse-replacement code.

**Root cause:** silent `int → str` dtype coercion in the `None`-in-axes collapse-replacement branch of `set_axes_order` at `src/hiveplotlib/hiveplot.py:2848-2855`. With `unique_strings = np.array([0, 1, 2, 3], dtype=int64)` and `replacement_map = {2: "Other", 3: "Other"}`, the list comprehension `np.array([replacement_map.get(s, s) for s in unique_strings])` returns a numpy array of `<U21` (unicode) dtype because numpy promotes the mixed list to a common dtype. Int `0` and `1` are silently coerced to strings `"0"` and `"1"`. Downstream, `groupby` produces keys `["0", "1", "Other"]` (strings) while the `axes_order` list still carries the original integers `[0, 1, "Other"]`. The `groupby_dict[0]` lookup raises `KeyError: np.int64(0)`. The bug surfaces through `HivePlotMatrix.from_partition` because its off-diagonal cell construction at `src/hiveplotlib/hiveplot_matrix.py:1192` always builds a `HivePlot(..., axes_order=[partition_values[i], partition_values[j], None], ...)`, hitting the collapse branch for every off-diagonal cell whenever the partition has more than 2 values, regardless of dtype.

**Why this matters:** the bug is shipped in v0.28.0 as a documented user-facing capability that does not work. The plan's "API usage examples" Example 1 (plan line 89) and the Workstream E notebook headline cell both demonstrate "compute Louvain communities, use the integer labels directly as `partition_variable`," which is the canonical entry point for community-detection-as-partition. With more than 2 communities, `from_partition` is the right primitive (per Workstream E amendment 1's analysis); with the int-partition bug, that primitive is broken. Any user following the documented Karate / Les Mis Louvain workflow hits this. The "API usage examples walked against realistic data" gap (mental-model rule 4) is that the Workstream E amendment 1 planning round confirmed `HivePlotMatrix.from_partition`'s kwarg signature was clean (true) but did not exercise the runtime collapse-replacement path with int-typed `partition_variable` (false claim of "mechanically clean").

**Shape chosen:** Option (b) from the dispatching session's framing, a discrete standalone bugfix amendment that ships before Workstream E amendment 1 re-dispatches. Rationale: the bug lives entirely on the source side (one or two lines in `set_axes_order`) with its own regression test surface (`tests/hiveplot_matrix_test.py` integer-partition coverage gap). Bundling the source fix into Workstream E amendment 1 muddles that amendment's "notebook author swaps primitives" framing and forces the notebook-author specialist to coordinate a source change. Splitting keeps each amendment crisp: this one is a code-engineer bugfix with a regression test; Workstream E amendment 1 stays as-written and re-dispatches against the now-fixed surface.

**Scope (three changes, two files plus CHANGELOG):**

1. **Fix the dtype coercion in `set_axes_order`'s collapse-replacement path.** At `src/hiveplotlib/hiveplot.py:2852-2855`, the line `replacement_strings = np.array([replacement_map.get(s, s) for s in unique_strings])` silently promotes the mixed `int + str` list to numpy's `<U21` unicode dtype. The fix is to force `dtype=object` on the constructor: `replacement_strings = np.array([replacement_map.get(s, s) for s in unique_strings], dtype=object)`. The downstream `replacement_strings[indices]` indexing still works under object dtype (numpy uses Python equality, not vectorized C); the resulting `collapsed_partition_values` is an object-dtype array of mixed int and str, which `pandas` assigns to the new column as-is. The DataFrame column's dtype becomes `object`, which `groupby` handles correctly (keys preserve their original types: ints stay ints, the collapsed-name string stays a string). The downstream `axes_order=[partition_values[i], partition_values[j], None]` lookup then resolves correctly because int `partition_values[i]` matches the int groupby key, and `None` matches the collapsed string label via the existing `None → collapsed_group_axis_name` rewrite at `hiveplot.py:2878-2880`. Code-engineer should verify there are no other call sites in `set_axes_order` or its helpers that assume the new partition column is string-typed; a grep for `_collapsed_axis` (currently 3 hits at `:2796`, `:2858`, `:2881`) catches the relevant surface area.

2. **Add regression tests for integer-typed `partition_variable` on `HivePlotMatrix.from_partition`.** At `tests/hiveplot_matrix_test.py`, add a focused test class or test pair covering:
   - **3-group int partition.** Build a small synthetic graph with `partition_variable="group"` where `group` is an int column with values `[0, 1, 2]`. Call `HivePlotMatrix.from_partition(graph=G, partition_variable="group", sorting_variables=<some_metric>)`. Assert the call returns a `HivePlotMatrix` without raising (this is the headline reproduction case from notebook-author's halt report).
   - **4-group int partition with explicit reproduction-shape framing.** Same as above with 4 int values to cover the off-diagonal cells `(0, 1)`, `(0, 2)`, `(0, 3)`, `(1, 2)`, `(1, 3)`, `(2, 3)` collapse paths. Assert at least one off-diagonal cell's `nodes.data` shows the expected `partition + "_collapsed_axis"` column with the giant int groups preserved and the remaining values relabeled to the collapsed name.
   - **String-partition parity regression (no behavior change expected).** Add a parallel string-typed test if one doesn't already exist in the same test class, parametrized alongside the int case, to lock in that the fix doesn't regress the existing string code path. The existing `tests/hiveplot_matrix_test.py` tests already use `"group"` with string values `"A"/"B"/"C"/"D"` (per notebook-author's halt report); the new test should be parameterized or duplicated to confirm parity.
   - **Direct `HivePlot(..., axes_order=[int_a, int_b, None])` test.** At `tests/hiveplot_test.py`, add a test that exercises the underlying `set_axes_order` collapse path directly with int-typed `partition_variable`, so that future refactors of the matrix layer don't mask a regression in the shared `HivePlot` collapse code. This test should construct a `HivePlot` with `partition_variable` as a 4-value int column, call `set_axes_order(axes=[0, 1, None], ...)`, and assert no `KeyError`.
   - All three test additions should be marked `@pytest.mark.networkx` if they construct via `graph=` and require a NetworkX import; the direct `set_axes_order` test does not need the marker if it builds `nodes` / `edges` without NetworkX.
   - 100% coverage requirement: the new `dtype=object` branch needs to be hit by at least one of these tests. Since the change is a one-arg widening (not a new branch), the existing coverage gates already pass once any int-partition test fires through the line.

3. **Add a `Fixed` entry to v0.28.0 (WIP) in `CHANGELOG.rst`.** Per the dispatching session's recommendation: yes, a `Fixed` entry is warranted because v0.28.0 ships the integer-partition use case as a documented capability (via Workstream E's notebook and the consolidated CHANGELOG bullet about graph-metric integer labels), so users following the documented path would hit the bug. The bug originated when `HivePlotMatrix.from_partition` shipped (prior plan, not this sprint), so this is not a regression from v0.28.0's own changes; framing the entry as a `Fixed` line (not `Changed`) reflects that distinction. Suggested wording: "`HivePlotMatrix.from_partition` raised `KeyError` when `partition_variable` was integer-typed with more than 2 unique values; the off-diagonal cell construction now preserves the original partition dtype through the collapse-replacement path." Place the bullet under a `Fixed` subsection of v0.28.0 (create the subsection if it doesn't already exist; existing v0.28.0 entry has `Added` and `Changed` subsections per Workstream E's consolidation). Voice-rule scan: no em-dashes, no AI filler. The Workstream E amendment 1 work, when it re-dispatches against the fixed surface, does not need its own additional CHANGELOG entry; the existing v0.28.0 notebook line already covers the visible surface there.

**Done when:**

1. `replacement_strings = np.array([...], dtype=object)` lands at `src/hiveplotlib/hiveplot.py:2852-2854` (exact line numbers may shift; the fix is identified by the np.array constructor inside the `None in axes` branch of `set_axes_order`). Code-engineer should confirm no other coercion site lurks downstream (the grep for `_collapsed_axis` covers it).
2. The four new regression tests above pass; existing 907+ tests stay green at 100% coverage.
3. `make test` green; `make ty` green; `ruff format` / `ruff check` clean.
4. The CHANGELOG `Fixed` entry lands in v0.28.0 (WIP) with the suggested wording or equivalent.
5. `python -c "import networkx as nx; from hiveplotlib import HivePlotMatrix; G = nx.karate_club_graph(); ... HivePlotMatrix.from_partition(graph=G, partition_variable=<int-col>, ...)"` runs end-to-end without `KeyError` (the smoke-test reproduction notebook-author confirmed as broken; should now pass).
6. Workstream E amendment 1 re-dispatches against this now-fixed surface; the notebook cells from that amendment's scope land cleanly without source-side workarounds.

**Items NOT acted on in this amendment:**

- **`HivePlot.set_axes_order` collapse-path string-dtype DataFrame column.** Even with `dtype=object`, the `new_partition_variable_name` column ends up as an object-dtype DataFrame column. This is the right shape (preserves the int + collapsed-string mix) but is technically a column-dtype change from the prior implementation's "always-string" behavior. No downstream callers should care (the column is internal-only, name has the `_collapsed_axis` suffix, never user-facing), but if a future user observes the column directly, they get an object dtype rather than a uniform string dtype. Flag for the code-engineer to confirm via grep that no internal consumer relies on the column being uniformly string-typed; if any do (e.g. a backend that calls `.astype(str)` somewhere on a non-collapsed column would still work, but a backend that hard-codes string equality would not), surface as a follow-up.
- **`require_using_all_partition_names=True` int-partition path.** Notebook-author confirmed reproduction via the `None`-in-axes path. The sibling `require_using_all_partition_names=True` path at `hiveplot.py:2883-2893` does not go through the buggy collapse-replacement code (no `replacement_strings` construction), so the int-partition shape works there already. No additional test needed for that path.
- **The bug's discoverability via warning rather than `KeyError`.** A future polish could detect the dtype-coerce condition and raise a clearer error (`InvalidAxesOrderError` with a recovery message naming the dtype mismatch), but the current fix simply makes the path work for int-typed partitions, so no error needs to fire. If the underlying numpy promotion ever surfaces in a different shape (e.g. mixed int + float partition), surface as a follow-up at that time.

**What we learned (for ADR promotion at v0.28.0 close-out):**

This bug exposed a planning-time validation gap in the Workstream E amendment 1 round. The amendment-planning step confirmed `HivePlotMatrix.from_partition`'s kwarg signature was mechanically compatible with the existing call sites (true: `graph=`, `partition_variable=`, `sorting_variables=`, `node_graph_metrics=` all line up), and on that basis the amendment was marked "kwarg shape is mechanically clean." It was not. The signature is clean but the runtime behavior depends on the dtype of `partition_variable`'s values, and the buggy collapse-replacement code path fires for every off-diagonal cell whenever the partition has more than 2 values. Mental-model rule 4 ("Walk every shipped API usage example against realistic data") would have caught this if the planning round had constructed even a tiny `from_partition` call with int-typed values rather than only confirming the kwarg signature. The lesson: when an amendment routes an existing user workflow through a different shipped entry point, the planning round needs to exercise the new entry point against the realistic data shape the amendment targets, not only confirm signature compatibility. Specifically, the feasibility audit's "trace each parameter to a real element in the library's documented data model" step (in the orchestrator's initial-plan workflow step 8) needs to extend, for amendments that re-route data through a different entry point, to "exercise the new entry point against a synthetic minimal instance of the realistic data the amendment targets." This is a cheap step (one `python -c` line) and would have caught this bug at the amendment-planning round rather than the amendment-execution round.

**Status:** Ready for dispatch. Recommended sequence: code-engineer first (single dispatch covering the source fix + the four new regression tests + the CHANGELOG entry; folding the test work into the code-engineer dispatch rather than a separate test-engineer dispatch because the fix and tests are file-contiguous and the test scope is mechanically specified above), then qa-engineer (verifies the fix lands at the named line, the regression tests pass, coverage stays at 100%, and the CHANGELOG entry follows voice rules), then api-critic post-impl review (short, confirms the source change preserves the existing `set_axes_order` contract and the new tests follow the test-name-contract per CLAUDE.md). Once this amendment ships clean, Workstream E amendment 1 re-dispatches per its own existing "Status: Ready for dispatch" routing (notebook-author → qa-engineer → viz-critic). Workstream E status stays `in_progress` across both amendments; advances to `complete` once viz-critic re-review on the now-matrix cells returns clean.

##### API Critic post-implementation review: HivePlotMatrix int-partition bugfix (2026-05-25)

```
Status: clean
API surface reviewed: HivePlot.set_axes_order (collapse-replacement branch of
  the `None in axes` path at src/hiveplotlib/hiveplot.py:2852-2858);
  HivePlotMatrix.from_partition (downstream beneficiary, signature unchanged)
Concerns: none (one-line `dtype=object` widening preserves the documented
  `set_axes_order` contract; closes the KeyError reproduction case without
  introducing new behavior beyond the bug fix; CHANGELOG bullet reads cleanly
  for a user hitting the bug on v0.27.x)
Test-method-coverage audit: clean. All four new tests call the named method in
  their body:
    - test_set_axes_order_collapse_axes_int_partition at
      tests/hiveplot_test.py:3237 calls hp.set_axes_order(axes=[0, 1, None], ...)
    - test_from_partition_int_partition_three_groups at
      tests/hiveplot_matrix_test.py:939 calls HivePlotMatrix.from_partition(...)
    - test_from_partition_int_partition_four_groups_preserves_int_axes at
      tests/hiveplot_matrix_test.py:975 calls HivePlotMatrix.from_partition(...)
    - test_from_partition_string_partition_parity at
      tests/hiveplot_matrix_test.py:1029 calls HivePlotMatrix.from_partition(...)
```

**Walkthrough notes (amendment confirmation, not findings):**

1. **`set_axes_order` contract preserved.** The only consumer of the post-`np.array` `replacement_strings` is the very next line (`replacement_strings[indices]` at `hiveplot.py:2859`), which produces `collapsed_partition_values` assigned directly to the new DataFrame column at `:2865`. A repo-wide grep for `_collapsed_axis` returns three hits (`hiveplot.py:2796, 2799, 2862` plus `node.py:226, 246, 247`); all six operate on the *column name* (string suffix `_collapsed_axis`), not the column *values*. No downstream code does `.astype(str)`, string equality, or any unicode-only operation on the collapsed values, so the dtype widening from `<U21` to `object` is a pure bug fix with no documented-behavior side effect. The new code comment at `:2852-2855` cleanly explains *why* the dtype is forced, which is the right load-bearing context for the next reader.

2. **String-partition users see no regression.** Walked the canonical existing call: `HivePlot(partition_variable="club", ...)` (e.g., `examples/karate_club.ipynb`-style) followed by `set_axes_order(axes=["Mr. Hi", None])`. `unique_strings` is `np.array(["Mr. Hi", "Officer"], dtype="<U7")`, the list comprehension yields `["Mr. Hi", "Other"]`, and `dtype=object` holds them as Python strings. `pandas.DataFrame.groupby` on an object-dtype column with string entries behaves identically to a `<U7` column for equality lookups. `test_from_partition_string_partition_parity` locks this in. No friction.

3. **Float-typed `partition_variable` (unusual but plausible).** With `current_partition_values` as floats (e.g., `[0.0, 1.0, 2.0, 3.0]` after a `compute_graph_metrics` call that produced float scores discretized by the user into a few buckets), `np.unique` returns floats, `np.searchsorted` works on floats, the list comprehension yields `[0.0, 1.0, "Other", "Other"]`, `dtype=object` preserves the mix, and `groupby` keys are `[0.0, 1.0, "Other"]`. The user's `axes_order=[0.0, 1.0, None]` rewrites to `[0.0, 1.0, "Other"]` and lookups resolve cleanly. The only pre-existing wrinkle is NaN-handling (NaN != NaN under groupby), but that's independent of the fix and would have surfaced either way. No new bug introduced.

4. **Mixed-type `partition_variable` (probably nonsense, but check).** A user with a partition column already typed as `object` containing `[0, "a", 2]` is going in as an object-dtype column from the start. `unique_strings` is object-dtype, the list comprehension produces a mixed-type list, and `dtype=object` preserves the mix. The post-fix behavior is strictly *less* surprising than the pre-fix behavior: pre-fix, every element would have been stringified to `["0", "a", "2"]` and the user's `axes_order=[0, "a", None]` lookup would have failed silently on the int; post-fix, the original types are preserved and the user can address each value by its original type. This is a net ergonomic win for the (admittedly weird) mixed-type case, not a new surprise.

5. **CHANGELOG bullet user-facing clarity (worth-discussing only, defer).** The bullet at `CHANGELOG.rst:179-182` reads: "`HivePlotMatrix.from_partition` raised `KeyError` when `partition_variable` was integer-typed with more than 2 unique values. The off-diagonal cell construction's collapse-replacement path now preserves the original partition dtype, so integer community labels (e.g. from `louvain_communities`) flow through cleanly." A user hitting `KeyError: np.int64(0)` on v0.27.x understands (a) the trigger condition (integer `partition_variable` with >2 unique values), (b) what changed (collapse-replacement path preserves dtype), and (c) the canonical real-world case (Louvain integer labels). Good. One low-confidence polish: the bullet only names `HivePlotMatrix.from_partition` as the user-visible failure, but the underlying fix is in `HivePlot.set_axes_order` and would also have surfaced for a user directly calling `HivePlot(partition_variable=<int>, axes_order=[int_a, int_b, None])`. Adding "`HivePlot.set_axes_order(axes=[..., None])` with an integer-typed `partition_variable` raised the same `KeyError`" would close the gap for users hitting the bug via the shared code path. Defer to a CHANGELOG polish pass at v0.28.0 close-out; not worth a bounce.

6. **No new findings from the bugfix itself.** Code comment at `:2852-2855` reads cleanly (names the bug, names the fix, no AI filler, no em-dashes). Test naming follows the existing project pattern (`test_<method>_<scenario>`); the `_preserves_int_axes` suffix on the 4-group test mirrors the `_validates_*` / `_raises` suffix convention used elsewhere in the same test file. Test docstrings spell out the regression-test framing without rationalization (each names the bug being locked, not a substitute). The `pytest.mark.networkx` marker is correctly omitted on the int-partition tests (they construct nodes/edges directly without a NetworkX import); the `make test` matrix should pick them up under the unmarked default run, matching the brief's plan-line 1122 routing.

**Verdict:** amendment closes cleanly. Workstream E amendment 1 (notebook-author re-run for the matrix-primitive swap) is unblocked and ready for dispatch.

### Maintainer grill — deferred-work disposition (2026-06-18)

Closure-pass grill ahead of the combined NetworkX ADR. This plan has no `## Alignment (grill)` section, so the disposition is recorded here. Append-only.

**The four future-igraph open design questions → bucket: MAY HAVE FORGOTTEN ABOUT (live, tied to the igraph-backend roadmap item).**

Open design questions #2-#5 in the "Open design questions" section above (#1 was resolved this sprint by Workstream A's structural seam) are not closed — they are deferred against the live igraph-backend roadmap item shipped as item 6 in `docs/source/roadmap.rst`. They stay on the radar as a coherent roadmap-tied cluster, not as forgotten loose ends. The four:

1. **Gap-metric strategy** — how the future igraph backend handles metrics it does not implement (planner lean: raise `NotImplementedError` naming the gap and the NetworkX equivalent, not silent fallback).
2. **leidenalg packaging** — `[igraph]` vs. separate `[igraph-leiden]` vs. `[igraph-all]` roll-up (planner lean: separate `[igraph-leiden]`, since leidenalg is GPL and some users opt out).
3. **GPL license posture** — the license boundary note needed when an igraph extra ships.
4. **igraph notebook approach** — parity notebook (`computing_graph_metrics_igraph.ipynb`) vs. extending the existing notebook (planner lean: parity notebook).

**GPL posture (#3) is the gating sub-question.** Both `igraph` and `leidenalg` are GPL-licensed; hiveplotlib is BSD-3. Whether an igraph extra is viable at all (and in what packaging shape, which feeds #2) hinges on the license-boundary call: optional extras don't relicense the hiveplotlib distribution, but the boundary needs a deliberate decision and a docs note before any igraph code lands. Resolve #3 first; #1, #2, and #4 are downstream of it.

**Durable home: the combined NetworkX ADR's deferred section.** These four travel together with the roadmap item; the ADR is the canonical forward record, and this plan's "Open design questions" section is the historical breadcrumb.

## Implementation log

Append-only. After each workstream completes, the executing agent writes one line here in the same turn:

- (none yet — no workstreams executed against this plan)
- 2026-05-25: Initial-plan refinement pass (orchestrator). Three changes against the prior draft: (1) "Prior ADRs / design docs" rewritten to cite `wiki/wiki/plans/i-want-to-plan-optimized-hoare.md` as the canonical prior work, flag the link-prediction reversal against the prior plan's line 127 decision, and defer ADR promotion to v0.28.0 close-out (combined ADR for both plans, not two separate ones). (2) `docs/source/autodoc/hive_plots/graph_features.rst` lifted from "possibly edited" to required-edit in Workstream A's files list, with the two `automodule::` directives called out at `:31` and `:37`; done-when adds a check that the rendered Node Metrics / Edge Metrics sections still show every wrapper docstring (the silent-failure mode of a wrong `automodule::` path is an empty section). (3) CHANGELOG routing tightened across Workstreams B / C1 / C2 / E to explicitly say "append to the existing v0.28.0 (WIP) entry; do NOT create v0.29.0," with metric-count update at `CHANGELOG.rst:54-57` called out by line number. No structural changes to workstream decomposition; the prior draft's A / B / C1 / C2 / D / E split, done-when criteria, and post-impl review scaffolding were sound and survive intact.
- 2026-05-25 (entries below carried over from the prior draft, retained for context — these are notebook-author edits to `hpm_*.ipynb` files that were tracked under this plan's banner before this session's refinement; they actually belong to the prior plan's implementation log and should be considered prior work):
- 2026-05-25: Structural cleanup on `examples/hpm_from_variable_sweep.ipynb` (notebook-author). Moved the `## Building from a NetworkX Graph` block and promoted `### Sweeping Over Graph Metrics` to `## Computing Graph Metrics During Construction`, both now sitting between `## Unified Axis Scaling` and `## Styling Directed Edges` to mirror the `hpm_from_partition` template. Rewrote the metrics section intro to lead with the broader "request metrics at construction time" framing (node metrics for sorting / partition, edge metrics for styling) and recast the sweep-multiple-metrics angle as one use case under that umbrella. Split the trailing markdown so the closing pointer to the Computing Graph Metrics page sits in its own cell, matching the template. Re-executed end-to-end cleanly.
- 2026-05-25: Added the data-driven per-cell edge coloring demo to `examples/hpm_generic.ipynb` (notebook-author) now that the `Edges.relevant_edges` cache bug is fixed. After the existing post-hoc `compute_graph_metrics()` attach cell, added a new code cell that iterates `iter_populated_cells()` and applies `update_edge_plotting_keyword_arguments(array="edge_betweenness_centrality", cmap="cividis", clim=(0, 0.06), alpha=1)`, plots, and overlays a shared `ScalarMappable` colorbar across the full 2x2 `axes` array with `shrink=0.7`. Added the gotcha markdown about data-driven vs. directional edge styling, kept the semantics paragraph as a lead-in to the inspect cells (split nodes vs. edges into separate code cells), and added `ScalarMappable` / `Normalize` imports up top. Stuck with the post-hoc `hpm.copy() + compute_graph_metrics()` flow since the section's framing emphasizes the workflow most natural for a generic-constructor user who already holds built `HivePlot`s. Re-executed end-to-end cleanly.
- 2026-05-25: Structural cleanup on `examples/hpm_from_tags.ipynb` (notebook-author). Moved the `## Building Multi-tag \`Edges\` from a NetworkX Graph` block (cells 7-8) and the `## Computing Graph Metrics During Construction` block (cells 30-37) out of their earlier positions and into the paired late-notebook slot between `## Unified Axis Scaling` (including its `### Set a Specific Range` subsection) and `## Styling Directed Edges`, mirroring the `hpm_from_partition` template. Tightened the metrics gotcha cell's forward reference from "(covered later)" to "We discuss directional edge styling in the next section." now that Styling Directed Edges sits directly after. Re-executed end-to-end cleanly.
- 2026-05-25: Workstream A complete. Moved `src/hiveplotlib/graph_features/{node,edge}_metrics.py` into a new `networkx/` subpackage; added `src/hiveplotlib/graph_features/networkx/__init__.py` re-exporting all 12 wrappers so `from hiveplotlib.graph_features.networkx import degree` works as a backend-explicit path; updated the two `automodule::` directives in `docs/source/autodoc/hive_plots/graph_features.rst` and the two extend-instructions docstring lines in `graph_features/__init__.py` to point at the new module paths; top-level `from hiveplotlib.graph_features import degree, ...` reexports unchanged. Tests: 50/50 graph_features tests pass with 100% coverage on the moved files; extended 517-test slice (graph_features + converters + hiveplot + hiveplot_matrix) passes; ruff format clean, ruff check clean, ty clean. Docs: sphinx build succeeded; rendered Node Metrics and Edge Metrics sections show all 10 node-metric and 2 edge-metric wrappers under their new `hiveplotlib.graph_features.networkx.*` paths (autodoc visual-check passed; empty-section silent-failure mode averted). CHANGELOG: no entry added (pure structural move; the submodule path is not a documented user surface and Workstreams B/C/E will write the consolidated v0.28.0 metric-additions entries).
- 2026-05-25: Workstream B complete. Added 11 Tier 1 NetworkX-backed node-metric wrappers to `src/hiveplotlib/graph_features/networkx/node_metrics.py`: `degree_centrality` (wraps `nx.degree_centrality`), `in_degree_centrality` (wraps `nx.in_degree_centrality`, directed-only), `out_degree_centrality` (wraps `nx.out_degree_centrality`, directed-only), `harmonic_centrality` (wraps `nx.harmonic_centrality`), `average_neighbor_degree` (wraps `nx.average_neighbor_degree`), `onion_layers` (wraps `nx.onion_layers`, undirected non-multigraph only), `square_clustering` (wraps `nx.square_clustering`), `constraint` (wraps `nx.constraint`), `effective_size` (wraps `nx.effective_size`), `closeness_vitality` (wraps `nx.closeness_vitality`), `load_centrality` (wraps `nx.load_centrality`). All 11 registered in `GRAPH_NODE_METRICS` (now 21 keys total: 10 prior + 11 new); re-exported from `hiveplotlib.graph_features.networkx.__init__`; restriction-raise checks follow the existing `in_degree`/`out_degree`/`triangles` pattern (directed/multigraph guards raise `ValueError` with a helpful message naming the fix). Tests: parametrized smoke test extended to route the two new directed-only wrappers through `_directed_fixture()`; 5 new restriction-raise tests added (`test_in_degree_centrality_on_undirected_raises`, `test_out_degree_centrality_on_undirected_raises`, `test_onion_layers_on_directed_raises[DiGraph]`, `test_onion_layers_on_directed_raises[MultiDiGraph]`, `test_onion_layers_on_multigraph_raises`), all marked under `pytest.mark.networkx` via the file-level `pytestmark`. All 64 graph_features tests pass (2 unrelated environmental scipy-missing failures pre-date this workstream and don't touch the new wrappers); ruff format / ruff check clean; ty diagnostics on the touched file unchanged from prior state (no new errors or warnings introduced). CHANGELOG: one-line bullet appended inside the existing v0.28.0 (WIP) `Added > Graph Metrics` subsection enumerating the 11 new wrapper keys.
- 2026-05-25: Workstream C1 complete. Added 14 Tier 2 NetworkX-backed node-metric wrappers to `src/hiveplotlib/graph_features/networkx/node_metrics.py`: `eccentricity` (wraps `nx.eccentricity`), `eigenvector_centrality_numpy` (wraps `nx.eigenvector_centrality_numpy`, scipy-backed alternative to `eigenvector_centrality`), `hits_hubs` / `hits_authorities` (each wraps one half of `nx.hits`; per User Resolution item 4 no caching layer between them, docstrings note that requesting both runs the iteration twice), `reciprocity` (wraps `nx.reciprocity` with `nodes=graph.nodes` so the per-node dict path is taken rather than the graph-level scalar; directed-only), `articulation_points` (wraps `nx.articulation_points` via the new `_set_to_indicator` adapter; undirected-only), `isolates` (wraps `nx.isolates` via `_set_to_indicator`; works on all 4 graph types), `topological_generations` (wraps `nx.topological_generations` projecting layers to int generation indices; directed-only), `greedy_modularity_communities` / `louvain_communities` / `label_propagation_communities` (community detection, project NetworkX set-of-sets partitions to int labels via the new `_partition_to_node_labels` adapter; `label_propagation_communities` is undirected-only per NetworkX), `connected_components` (undirected-only), `strongly_connected_components` / `weakly_connected_components` (directed-only). All 14 registered in `GRAPH_NODE_METRICS` (now 35 keys total: 10 prior + 11 from B + 14 new); re-exported from `hiveplotlib.graph_features.networkx.__init__`. Two new private helpers added: `_partition_to_node_labels(partition)` sorts communities by size descending with smallest-node-id tiebreak so label 0 is always the giant community (per User Resolution item 3); `_set_to_indicator(elements, all_keys)` projects a set onto a per-key bool dict. Docstrings on the community-detection and connected-components wrappers explicitly document the "label 0 = giant" ordering contract. Metric-table classifier extension folded into this workstream (per the dispatching session's brief): widened the exception catch in `docs/source/_ext/metric_table_directive.py:_classify_graph_constraint` from `(ValueError, nx.NetworkXNotImplemented)` to `(ValueError, nx.NetworkXException)` (covers `NetworkXUnfeasible` raised by `topological_generations` on cyclic K4); added a secondary DAG probe so DAG-requiring metrics like `topological_generations` correctly classify as "Requires a directed graph." rather than falling through to empty; added the fourth predicate `simple_undirected_only` returning "Requires an undirected, non-multigraph graph." which now fires for `onion_layers` (closes the worth-discussing finding from B's api-critic review). Tests: smoke-routing tuple at `tests/graph_features_test.py:87` extended to route `reciprocity`/`strongly_connected_components`/`weakly_connected_components` through `_directed_fixture()` and `topological_generations` through a new `_dag_fixture()` (a 5-node DAG with two source nodes); 7 new restriction-raise tests added (covering `reciprocity`, `topological_generations`, `articulation_points` x2 parametrized, `label_propagation_communities` x2 parametrized, `connected_components` x2 parametrized, `strongly_connected_components`, `weakly_connected_components`); 6 new family shape tests (greedy_modularity, louvain, label_prop, connected_components, strongly_connected_components, weakly_connected_components) using a new `_two_component_undirected_fixture()` / `_two_component_directed_fixture()` asserting at least 2 distinct int labels and the giant-component-gets-label-0 contract; 3 direct adapter-helper tests (size-descending ordering, smallest-node-id tiebreak, indicator-flag membership). `make test` green at 100% coverage across the whole project (907 tests pass, including the new 99 in graph_features); `make ty` green; `ruff format` / `ruff check` clean across all touched files; `sphinx-build` green with the rendered `Node Metrics` table showing all 35 wrappers including the new entries with correct constraint sentences (onion_layers → "Requires an undirected, non-multigraph graph.", topological_generations → "Requires a directed graph.", reciprocity → "Requires a directed graph.", etc.). CHANGELOG: appended a 14-wrapper Added bullet into the existing v0.28.0 (WIP) `Added > Graph Metrics` subsection, plus a short bullet documenting the metric-table classifier extension.
- 2026-05-25: Workstream C1 amendment 1 shipped (code-engineer). Added the missing `eigenvector_centrality_numpy` multigraph guard mirroring the sibling `eigenvector_centrality` pattern (`ValueError` raise + "Requires a non-multigraph" docstring paragraph + `:param graph:` qualifier + `:raises ValueError:` line); the recovery message names the same `multigraph=False` rebuild path, and the docstring explicitly points the user at `hiveplotlib.converters.nodes_edges_to_networkx(multigraph=False)` because the sibling carries the same restriction and is not a viable fallback. Added `:raises networkx.NetworkXUnfeasible:` line to `topological_generations`'s structured raises block (wrapped to two lines for the 120-char docstring limit; Sphinx renders the intersphinx-linked NetworkX cross-reference correctly). Corrected count drift in narration: `CHANGELOG.rst:69` (13→14), plan body's "New node-metric keys (Tier 2, 13 wrappers across families)" (13→14), Workstream C1 sub-block heading and review prose (13→14 in three spots), Workstream C1 files list (13→14), done-when criteria #1 / #2 (13→14, 34→35 totals), Workstream E CHANGELOG-target count (~34→~35), and the C1 implementation log entry's "14 Tier 2 ... 35 keys total: 10 prior + 11 from B + 14 new" / "all 35 wrappers" / "14-wrapper Added bullet" phrasings; preserved the amendment block's own quoted drift strings since those are descriptions of the prior narration. Tests: added `test_eigenvector_centrality_numpy_on_multigraph_raises` parametrized over `MultiGraph` / `MultiDiGraph`, mirroring the existing `test_eigenvector_centrality_on_multigraph_raises` shape and placed adjacent to it (asserts `ValueError` with `match="multigraph"`); the parametrized smoke test still routes `eigenvector_centrality_numpy` through `_undirected_fixture()` and continues to pass. `pytest tests/graph_features_test.py` green (101 passed, 100% coverage on `src/hiveplotlib/graph_features/networkx/node_metrics.py` including the new guard branch); broader 127-test slice (graph_features + converters) also green; `ruff format` / `ruff check` clean; `ty check` clean; `sphinx-build` green with `-W` (warnings-as-errors), and visual confirmation in the rendered HTML: `topological_generations`'s raises block now shows both `ValueError` and `networkx.NetworkXUnfeasible` entries (the latter as a hyperlinked cross-reference to NetworkX's docs), and the Node Metric Table entry for `eigenvector_centrality_numpy` now reads "Not supported on multigraphs." (the metric-table classifier picked up the new restriction automatically via the `ValueError`-catching probe path, as the amendment predicted). No new CHANGELOG bullet for the guard or `:raises:` fix per the brief; these are corrections to C1's scope rather than independent additions.
- 2026-05-25: Workstream C2 complete. Added 6 Tier 2 NetworkX-backed edge-metric wrappers to `src/hiveplotlib/graph_features/networkx/edge_metrics.py`: `bridges` (wraps `nx.bridges` via `_set_to_indicator` adapter imported from `node_metrics`, undirected-only with a `ValueError` guard matching the canonical recovery-pointer template), and 5 link-prediction wrappers built off a new `_make_link_prediction_wrapper(nx_func_name, *, short_description, long_description, return_phrase, extra_kwargs="")` factory: `jaccard_coefficient`, `adamic_adar_index`, `preferential_attachment`, `resource_allocation_index`, `common_neighbor_centrality` (all wrap their like-named `nx.<name>` function; undirected non-multigraph-only constraint comes through NetworkX's own `NetworkXNotImplemented` decorator). All 6 registered in `GRAPH_EDGE_METRICS` (now 8 keys total: 2 prior + 6 new); re-exported from `hiveplotlib.graph_features.networkx.__init__`. The link-prediction factory defaults `ebunch=graph.edges()` so the score attaches to every existing edge in the graph (reverses the prior plan's deliberate exclusion of the link-prediction family at `wiki/wiki/plans/i-want-to-plan-optimized-hoare.md:127`), and the templated docstring it generates carries the canonical caveat ("By default this wrapper applies the NetworkX link-prediction score to every existing edge in ``graph`` (``ebunch=graph.edges()``); NetworkX intends these scores for *non-edges* by default ... pass an explicit ``ebunch=...`` to score a different edge set.") on every wrapper. The factory sets `__name__` / `__qualname__` / `__module__` / `__doc__` on each closure so Sphinx autodoc, the metric-table classifier, and ad-hoc `help(...)` render them as named entries (verified by reading the generated TOC). Tests: parametrized edge-metric smoke test auto-picks up all 6 new keys via `_edge_metric_names()` against `GRAPH_EDGE_METRICS`; 2 new `bridges` restriction-raise tests (parametrized over `DiGraph` / `MultiDiGraph`) plus 2 shape tests (every edge of `path_graph(5)` is a bridge; no edge of `complete_graph(5)` is a bridge); 5 parametrized link-prediction tests confirming the default `ebunch=graph.edges()` covers every edge in `complete_graph(5)`; 5+5 parametrized link-prediction restriction-raise tests (directed via `DiGraph`/`MultiDiGraph` and undirected multigraph) asserting `NetworkXNotImplemented` with `match=r"directed|multigraph"` (the multigraph decorator fires first for `MultiDiGraph`, so the regex permits both wordings); 5 parametrized tests asserting explicit `ebunch=...` overrides the per-edge default and recovers NetworkX's "score arbitrary pairs" behavior (including non-edge pairs); 5 parametrized docstring-template tests asserting every link-prediction wrapper carries the "non-edges" + "ebunch=graph.edges()" caveat; 1 direct factory test asserting `__name__` / `__qualname__` / `__doc__` propagation. `pytest tests/graph_features_test.py` green (142 passed, 100% coverage on `src/hiveplotlib/graph_features/networkx/edge_metrics.py` including all 6 new wrappers, the `_set_to_indicator` cross-module reuse path, and the factory closure); broader `pytest tests/` green (950 passed at 100% coverage across the whole project); `ruff format` / `ruff check` clean across all touched files (one RUF043 fix: changed `match="directed|multigraph"` to `match=r"directed|multigraph"` for the metacharacters-as-regex pattern); `ty check` clean (one `cast` added inside `bridges` to bridge the `_set_to_indicator` helper's `dict[Hashable, bool]` return type to the edge-metric contract's `dict[Tuple[Hashable, ...], bool]`; one runtime-attribute test reads `getattr(fn, "__name__", None)` instead of `fn.__name__` because ty cannot statically infer the factory's runtime attribute assignment on the returned `Callable`); `make docs` green with `-W` (warnings-as-errors). Visual confirmation in the rendered HTML: the Edge Metric Table now shows all 8 wrappers with correct constraint sentences (`bridges` → "Requires an undirected graph.", the 5 link-prediction wrappers → "Requires an undirected, non-multigraph graph.", the 2 prior centrality wrappers → no constraint sentence), the link-prediction wrappers all render their conceptual paragraphs plus the "non-edges by default" caveat plus the intersphinx-linked `:raises networkx.NetworkXNotImplemented:` block, and the TOC lists all 8 edge wrappers under their proper names (no rogue `wrapper()` entries from the factory closure). CHANGELOG: appended a 6-wrapper Added bullet plus a dedicated bullet for the link-prediction default-behavior shift (calling out the reversal of the prior plan's exclusion) into the existing v0.28.0 (WIP) `Added > Graph Metrics` subsection, positioned after the 14-wrapper C1 node-metric bullet and before the metric-table classifier line.
- 2026-05-25: Workstream D complete (docs-engineer). Added a new item 9 "Optional `igraph` backend for `compute_graph_metrics`" to `docs/source/roadmap.rst` (appended after the existing item 8 "More examples"; not folded as a sub-bullet under item 7 since item 7's prose is NetworkX-scoped and the igraph entry is a future second-backend direction). Entry covers the headline pitch (five community-detection algorithms NetworkX lacks plus speed), names the `graph_features.networkx/` subpackage layout shipped in v0.28.0 as the structural seam, enumerates the seven NetworkX wrappers that would `NotImplementedError` under the future backend (`load_centrality`, `edge_load_centrality`, `closeness_vitality`, `effective_size`, `onion_layers`, `square_clustering`, `topological_generations`), gives the rough scope (~30-35 wrappers, 600-900 LOC including tests), and inline-references the deferred design questions (gap-metric behavior, `leidenalg` GPL packaging, parity-vs-extension notebook call). No live cross-link to the plan: the existing roadmap convention has zero plan/wiki cross-references, so a one-line inline prose pointer ("tracked in the working plan that introduced this roadmap entry") matches the file's discipline. Plan-scaffolding language scrubbed (referred to "the `graph_features.networkx/` subpackage layout" rather than "Workstream A"). Voice-rule scan clean (no em-dashes, no AI filler, length matches sibling items at ~10 lines). `make docs` green with no new warnings; rendered HTML shows the entry as `<ol>` item 9 with all backtick literals resolving correctly. CHANGELOG: no entry added per the dispatching brief (pure roadmap addition; routing for any CHANGELOG bullet under v0.28.0 was the workstream owner's call and the entry was judged not user-visible enough to warrant one; flagged for Workstream E if the consolidation pass wants to surface "future igraph backend tracked in the roadmap" as a one-line "Changed" note).
- 2026-05-25: Workstream E CHANGELOG sweep complete (docs-engineer). Consolidated the `Added > Graph Metrics` subsection of v0.28.0 (WIP) into a curated structure: kept the foundational `HivePlot(*_graph_metrics=...)` constructor bullet as-is; rewrote the master `graph_features` package bullet to lead with the final counts ("35 node-metric and 8 edge-metric functions from `networkx`") and reorganized the wrapper enumeration into nine family sub-bullets (degree / distance / link-analysis / clustering / structural-position / DAG / community-and-components / edge-centralities / edge-link-prediction) covering every key in `GRAPH_NODE_METRICS` and `GRAPH_EDGE_METRICS` (counts verified against the registries at `src/hiveplotlib/graph_features/__init__.py:96-148`: 35 + 8 = 43 entries); added one separate bullet for the backend-explicit `hiveplotlib.graph_features.networkx` import path (Refactor item surfaced as Added because the public path is new and the backend-explicit seam matters to users tracking the future igraph backend); kept the link-prediction `ebunch=graph.edges()` default-behavior bullet as a dedicated standalone item (preserves the user-visible signal that this reverses the prior plan's exclusion); kept the metric-table classifier-extension bullet but tightened it to reference both Node and Edge Metric tables (the fourth constraint sentence fires for both `onion_layers` on the node side and the link-prediction family on the edge side). Dropped from the incremental bullets: the "11 additional Tier 1", "14 additional Tier 2", and "6 additional Tier 2 edge" provenance prefixes; the internal mechanism narration ("via the `_make_link_prediction_wrapper` factory", "via the `_set_to_indicator` adapter"); and the Workstream X cross-references (per plan-scaffolding ban). No v0.29.0 entry created. D CHANGELOG decision: skipped the "roadmap entry for an optional future igraph backend" bullet because the roadmap is itself a forward-pointer document and duplicating that signal in CHANGELOG would be noise; the git log and roadmap section capture provenance for any future trace. First `make docs` ran with one warning (`py:mod reference target not found: hiveplotlib.graph_features.networkx` at line 81, because the parent `networkx/` package isn't autodoc-documented even though its submodules are); fixed by switching the inline xref from `:py:mod:` to a plain code-fence literal, which matches how other prose in the changelog references the path. Final `make docs` green with no warnings; verified the rendered changelog HTML at `public/changelog.html` shows the 35/8 counts, the nine family sub-bullets, the backend-explicit-path bullet (with the literal code fence rendering correctly), and the link-prediction caveat. Voice-rule scan clean (no em-dashes, no AI filler, plan-scaffolding language scrubbed).
- 2026-05-25: Workstream E notebook updates complete (notebook-author). Added 16 cells (8 markdown, 8 code) to `examples/computing_graph_metrics.ipynb` for the v0.28.0 metric additions; only the notebook in `examples/` is touched (auto-generated copies under `docs/source/notebooks/` and `docs/source/gallery_examples/` are not edited). Five additions: (1) one-paragraph backend-dispatch markdown inserted after `## Available Node and Edge Metrics` linking to the NetworkX backends-and-configs docs and naming `graphblas-algorithms`, `nx-parallel`, `nx-cugraph` as plugins that dispatch automatically when installed; (2) `### Requesting Multiple Metrics at Once` subsection demonstrating `node_graph_metrics=["harmonic_centrality", "average_neighbor_degree"]` at init time; (3) `### Using Community Detection as a Partition Variable` subsection on Karate Club showing `partition_variable="louvain_communities"` (integer labels go directly into partition without binning), plus a `pd.crosstab` cell against the `club` ground-truth showing each Louvain community sits within a single club with one crossover node (the markdown explicitly frames this as "Louvain finds finer-grained substructure than the historical 2-way split, but respects the club boundary as a sanity check", since default-`resolution` Louvain finds 4 communities on Karate Club rather than the historical 2; the cross-tab supports this honestly without massaging the kwarg to force a 2-split); (4) new `## Precompute Once, Reuse Across Plots` H2 section on `nx.les_miserables_graph()` (77 nodes, 254 edges, undirected) calling standalone `compute_graph_metrics(G, node_metrics=["louvain_communities", "harmonic_centrality"], ...)` once and building two `HivePlot`s from the same annotated `nodes`/`edges` (one partitioned by Louvain community via integer-label-as-partition, one partitioned by 3-way `create_partition_variable()`-binned harmonic centrality); cost story stated in the section lead-in ("computing the metrics once ... is faster than passing `node_graph_metrics` to each `HivePlot` constructor (which would recompute every time)"); the second plot uses `axes_order=["low", "medium", "high"]` to enforce the natural progression. (5) `### Link Prediction Scores as an Edge Metric` subsection mirroring the existing `edge_betweenness_centrality` cell structure: `edge_graph_metrics="jaccard_coefficient"` at init, `cividis` colorbar with `clim=(0, 0.5)`, same `reset_edges` for the 2-partition + colorbar overlay pattern, plus reiterates the wrapper-docstring caveat in the section lead ("NetworkX intends these scores for *non-edges* (predicting where missing edges might exist), but hiveplotlib applies the score to every existing edge in the graph by default. The result is a per-edge link-likelihood annotation ..."), and the closing markdown points at the `edge_graph_metric_kwargs={"jaccard_coefficient": {"ebunch": ...}}` escape hatch for scoring non-edges. Insertion strategy: anchored each insertion to a stable upstream cell id (`75736a85`, `a62478d7`, `metric-as-partition-inspect-code`, `e11b570c`) and inserted via a one-off script that built each new cell with a stable kebab-case id (`networkx-backend-dispatch-note`, `tier1-multi-metric-*`, `community-as-partition-*`, `precompute-reuse-*`, `link-prediction-*`); no existing cells were edited (45 -> 61 cells, +16 net). H1/H2/H3 outline verified clean (new H3s sit under the right H2 parents; new H2 `## Precompute Once, Reuse Across Plots` sits naturally between the partition section and the edge-metric section). Used the canonical `HivePlot(graph=G, ...)` keyword-only constructor for both Karate Club and Les Mis cells (no usage of the removed `HivePlot.from_networkx` classmethod). Executed end-to-end via WSL-side `.venv/bin/jupyter nbconvert --execute --inplace` and confirmed clean run (61 cells, 4 new figures rendered, 3 new DataFrame heads displayed, no new errors; pre-existing intentional `try/except` traceback cells unchanged); writes 1.7 MB notebook. Voice-rule scan clean on all 9 new markdown cells (no em-dashes, no `delve`/`moreover`/`furthermore`/`underscore`/`in essence`/`it's worth noting`/`let us`/`as we can see`/`in the realm of`, no plan-scaffolding language like "Workstream X" / "Tier 1" / "Tier 2" leaked into prose; the latter are referred to functionally by metric family or capability name). viz-quality-bar scan: instructional polish budget honored (`set_title(..., y=1.05, size=20)` matches the existing instructional sibling pattern; `set_title(..., y=0.75, size=20)` on the Jaccard plot matches the existing edge-betweenness sibling exactly; the four new hive-plot figures use library-default palettes for nodes/edges with `cividis` reserved for the sequential edge-coloring case; the link-prediction plot uses `cividis` (matplotlib-path sequential default per the corpus appendix) to differentiate visually from the betweenness `plasma` plot above it; the 4-axis Karate Louvain plot and 6-axis Les Mis Louvain plot break the 3-axis canonical pattern but the axis counts are motivated by the metric's natural output (Louvain naturally finds 4 communities on Karate Club at default `resolution=1.0` and 6 on Les Mis); the 3-axis Les Mis HC-bin plot is the canonical 3-axis case; all four plots use `repeat_axes=True` consistent with the notebook's existing pattern). No CHANGELOG entry per the dispatching brief (parallel dispatch to docs-engineer handled the CHANGELOG sweep). Insertion script `insert_cells.py` left in the working tree at repo root as a one-off; recommended cleanup before commit.
- 2026-05-25: `HivePlotMatrix.from_partition` integer-partition bugfix amendment complete (code-engineer). Source fix at `src/hiveplotlib/hiveplot.py:2852-2858`: added `dtype=object` to the `replacement_strings = np.array([...])` constructor in the `if None in axes:` collapse-replacement branch of `set_axes_order`, plus a short comment naming the bug (numpy silently promoted the mixed `int + str` list to `<U21`, coercing int partition values to strings and breaking the downstream `axes_order=[int_a, int_b, None]` lookup with `KeyError: np.int64(0)`). Reproduction case from the brief (`HivePlotMatrix.from_partition(graph=karate_club_graph(), partition_variable="louvain_communities", ...)`) now returns a valid 4x4 matrix instead of raising. Four regression tests added: in `tests/hiveplot_matrix_test.py::TestFromPartition`, `test_from_partition_int_partition_three_groups` (3-value int partition, smoke that the call returns a `HivePlotMatrix` with the expected 3x3 shape and int-stringified row/col labels), `test_from_partition_int_partition_four_groups_preserves_int_axes` (4-value int partition, asserts every off-diagonal cell's `axes_order` carries the original int partition values plus the collapsed group name and the `cell.axes` keys match), `test_from_partition_string_partition_parity` (4-value string partition parity, confirms the fix doesn't regress the existing string codepath), plus in `tests/hiveplot_test.py::TestHivePlot`, `test_set_axes_order_collapse_axes_int_partition` (direct `HivePlot(partition_variable=int_col).set_axes_order(axes=[0, 1, None], ...)` test locking the contract at the shared `set_axes_order` collapse-replacement code level, independent of the `HivePlotMatrix` layer). Verified regression behavior by temporarily reverting the source fix: the 3 int-partition tests fail with the documented `KeyError`, the parity test passes (the string codepath always worked); restored the fix and all 4 pass. CHANGELOG: added a `Fixed` bullet to the existing v0.28.0 (WIP) `Fixed` subsection at `CHANGELOG.rst:179-182`, noting the off-diagonal cell construction's collapse-replacement path now preserves the original partition dtype so integer community labels (e.g. from `louvain_communities`) flow through cleanly. Local validation: `pytest tests/hiveplot_test.py tests/hiveplot_matrix_test.py -n 7` green (445 passed); `ruff format` and `ruff check` clean on touched files (one trivial reflow in `tests/hiveplot_test.py` collapsing a multi-line call back onto one line); `ty check` clean. No new files. Did not run full project suite or `make docs` (qa-engineer's scope).
- 2026-05-25: Workstream E amendment 1 re-dispatched and complete (notebook-author) against the now-fixed `HivePlotMatrix.from_partition` int-partition surface. Three changes to `examples/computing_graph_metrics.ipynb`: (1) Karate Louvain cell pair (`community-as-partition-md`/`community-as-partition-code`/`community-as-partition-xtab-code`): swapped `HivePlot(graph=G, partition_variable="louvain_communities", ..., repeat_axes=True)` for `HivePlotMatrix.from_partition(graph=G, partition_variable="louvain_communities", ..., progress=False)`; dropped `repeat_axes=True`; renamed `hp_louvain` to `hpm_louvain`; switched title placement from `ax.set_title(..., y=1.05, size=20)` to `fig.suptitle(..., y=1.02, size=16)` matching the canonical HPM pattern from `hpm_from_partition.ipynb`/`hive_plot_matrices.ipynb`; reframed the lead-in to lead with the pedagogical point (integer labels go directly into `partition_variable` without binning) and explain that 4 communities is the natural trigger for `HivePlotMatrix.from_partition()`; rewrote the cross-tab cell to pull `nodes.data` from `hpm_louvain[0, 0]` (HPM doesn't expose `.nodes` directly; every cell carries the same underlying nodes); cross-tab integrity preserved (4 Louvain communities, community 0 = 1 Mr. Hi + 13 Officer, the single crossover; communities 1/2 entirely Mr. Hi, community 3 entirely Officer). (2) Les Mis Louvain cell pair (`precompute-reuse-plot1-md`/`precompute-reuse-plot1-code`): same swap shape to `HivePlotMatrix.from_partition(nodes=nodes_lesmis, edges=edges_lesmis, ..., progress=False)`; renamed `hp_lesmis_comm` to `hpm_lesmis_comm`; same `fig.suptitle(..., y=1.02, size=16)` title pattern; reframed precompute-reuse section lead-in to say "two views from the same nodes and edges" (hive plot matrix for community structure, 3-axis hive plot for centrality binning); updated the plot 1 markdown to call out 6 communities → 6 x 6 upper-triangular matrix; updated plot 2 markdown to say "second view" and "Three bins fit cleanly on a 3-axis HivePlot." Sibling 3-axis HC-bin plot (`precompute-reuse-plot2-code`) stays as `HivePlot` per the brief, preserving the matrix-vs-3-axis-hive-plot pedagogical contrast. (3) Bundled polish on cell 31 (Karate Jaccard): tightened cividis `clim` from `(0, 0.5)` to `(0, 0.3)` based on actual distribution analysis (`pd.Series(jaccard_values).describe()` showed median 0.11, Q75 0.20, Q90 0.30, max 0.53, so 90% of edges fall in [0, 0.3] and the prior `(0, 0.5)` clim pushed the bulk into cividis's deep-blue zone where contrast collapses); added `extend="max"` to the colorbar so the saturated edges above 0.3 read honestly as "above the displayed range" rather than lying at exactly 0.3; comment in the code names the median/Q90 evidence for the choice. Title `y=0.75` preserved verbatim to maintain the sibling-match invariant with the `edge_betweenness_centrality` cell (viz-critic flagged this as the defensible choice). Imports: added `HivePlotMatrix` to the existing `from hiveplotlib import HivePlot` line; `HivePlot` still in use for the Les Mis HC-bin sibling and all the other non-matrix cells. `make test-nb` green for this notebook (`pytest -c tests/pytest_examples.ini -k computing_graph_metrics` 1 passed in 8.52s under `-W error`). Voice-rule scan clean (no em-dashes, no AI filler, no plan-scaffolding language); cross-tab semantics check confirms 4 communities + 1 crossover preserved. No CHANGELOG edit per the brief.
- 2026-05-26: Carry-over polish: `HivePlotMatrix.from_partition` row/col label dtype preservation (code-engineer). Source fix at `src/hiveplotlib/hiveplot_matrix.py:1220` replaced `[str(g) for g in partition_values]` with `list(partition_values)` so integer partition values flow through to `hpm.row_labels` / `hpm.col_labels` as ints rather than stringified versions. The viz layer already stringifies at the rendering boundary (`src/hiveplotlib/viz/hiveplot_matrix.py:269,287,295,518,528`), so no downstream consumer required strings; this restores the symmetry already in place for `axes_order` / `cell.axes` (preserved by the prior bugfix amendment via `dtype=object`). Tests: updated `tests/hiveplot_matrix_test.py::TestFromPartition::test_from_partition_int_partition_three_groups` to assert `row_labels == [0, 1, 2]` (was `["0", "1", "2"]`) plus per-element `isinstance(label, (int, np.integer))` checks on both label lists; extended `test_from_partition_string_partition_parity` with a matching `isinstance(label, str)` parity assertion on both label lists. Added two `assert hpm.row_labels is not None` / `assert hpm.col_labels is not None` narrowing lines per the code-engineer expertise gotcha (Optional return type needs explicit narrowing before iteration). `pytest tests/hiveplot_matrix_test.py -n 7` green (166 passed); full `make test` green (954 passed at 100% coverage on `hiveplot_matrix.py`); `make format` clean; `make ty` clean; `make docs` clean (zero warnings; the 2 pre-existing autodoc cross-reference warnings in `edge_metrics.py` seen mid-run were independently cleared by the conceptual-paragraph backfill entry below, confirmed by a re-run after that landed). CHANGELOG: extended the existing v0.28.0 (WIP) `Fixed` bullet for the int-partition `KeyError` to also mention `row_labels` / `col_labels` dtype preservation, kept as a single bullet since the two symptoms share the same root cause and recovery story.
- 2026-05-26: Carry-over polish: conceptual-paragraph backfill on the 10 pre-existing node-metric wrappers and the 2 pre-existing edge-metric wrappers (docs-engineer). Restores symmetry with the 25 wrappers added in Workstreams B / C1 / C2 (api-critic's `worth-discussing` from B's post-impl review). Added a 1-3 sentence conceptual paragraph between the "Wraps" line and the `:param graph:` block on each of the 12 wrappers in `src/hiveplotlib/graph_features/networkx/{node,edge}_metrics.py`: `degree` (simplest centrality, first lens for spotting hubs), `in_degree` (inbound edges, sinks/popular targets vs. originators), `out_degree` (outbound edges, sources/broadcasters vs. receivers), `betweenness_centrality` (fraction of shortest paths passing through each node, flags bottleneck/broker nodes), `closeness_centrality` (reciprocal of mean shortest-path distance, plus cross-ref to `harmonic_centrality` for the disconnected-graph variant), `eigenvector_centrality` (scores proportional to neighbors' scores, conceptual ancestor of PageRank), `pagerank` (random-walker stationary distribution, generalizes eigenvector centrality to handle dangling nodes and disconnected components), `clustering` (fraction of neighbor pairs that are themselves connected, local transitivity), `core_number` (largest k for k-core membership, plus cross-ref to `onion_layers` for the finer-grained refinement), `triangles` (closed three-node cycles, simplest local-transitivity measure), `edge_betweenness_centrality` (edge-side analog of node betweenness via cross-module `~hiveplotlib.graph_features.networkx.node_metrics.betweenness_centrality` xref), `edge_load_centrality` (edge-side analog of node load centrality via same cross-module xref shape). Cross-module xrefs use the `~` prefix to render as just the function name; short-form `:py:func:`name`` works within a single module but resolves wrong across the node/edge split (initial naive short-form fired two `py:func reference target not found` warnings, fixed by qualifying the path). Sphinx now resolves all xrefs correctly. Voice-rule scan clean (no em-dashes, no AI filler, no developer-facing meta-commentary; paragraphs match the post-B/C1/C2 wrappers' length discipline of 1-3 sentences). `make format` clean (117 files unchanged, ruff check passed), `make ty` clean, `make docs` green with zero warnings (down from 2 transient warnings during the cross-module-xref fix). No test changes (docstring-only edits hold 100% coverage); no CHANGELOG entry per the dispatching brief (pure docstring polish, not user-visible enough to warrant a CHANGELOG line on its own).
- 2026-05-26: Carry-over polish trio (docs-engineer): items 4, 6, 9 from the v0.28.0 close-out list. (4) `common_neighbor_centrality` docstring now names the NetworkX `alpha=0.8` default; appended one sentence to the wrapper's `long_description` slot in the `_make_link_prediction_wrapper` factory invocation (`src/hiveplotlib/graph_features/networkx/edge_metrics.py:236-241`) pointing the user at `:py:func:`networkx.algorithms.link_prediction.common_neighbor_centrality` defaults to ``alpha=0.8``; pass an explicit ``alpha=`` to override`. Used the qualified path because the short-form `:py:func:`networkx.common_neighbor_centrality`` does not resolve via intersphinx (caught by `make docs` warning on the first pass, fixed before final build). (6) Reverse cross-reference from `articulation_points` to `bridges`: added one sentence ("The node-side analog of `:py:func:`~hiveplotlib.graph_features.networkx.edge_metrics.bridges`.") to the `articulation_points` conceptual paragraph at `src/hiveplotlib/graph_features/networkx/node_metrics.py:681-683`, mirroring the inline-prose shape that `edge_betweenness_centrality` and `edge_load_centrality` use to point at their node-side analogs. (9) Dropped the stale "plan line 741" reference inside the C1 amendment block (`wiki/wiki/plans/networkx-metric-expansion-and-backend-refactor.md:1034`) in favor of just "the C1 implementation-log entry" per the brief's recommendation, since the plan keeps growing and a line number will keep drifting. `make docs` green with zero warnings (initial run flagged one warning on the unqualified `common_neighbor_centrality` xref; corrected to the qualified path and re-ran clean). No test changes (docstring-only); no CHANGELOG entry (pure polish).
- 2026-05-26: Carry-over polish: relocate private helpers to a neutral home (code-engineer). Resolves api-critic's `worth-discussing` cross-module-import blemish from C2's post-impl review. Created `src/hiveplotlib/graph_features/networkx/_helpers.py` (47 lines) housing `_partition_to_node_labels` and `_set_to_indicator` verbatim (signatures and docstrings unchanged); deleted the two definitions from `node_metrics.py` (now imports them from `_helpers.py` alongside removing the no-longer-needed `Iterable` from the `typing` import); retargeted `edge_metrics.py`'s cross-module import from `hiveplotlib.graph_features.networkx.node_metrics` to `hiveplotlib.graph_features.networkx._helpers`, plus a one-word tweak to the adjacent `cast` comment ("reused from the node-metric side" → "shared with the node-metric side") to reflect the new neutral location; updated the three test imports in `tests/graph_features_test.py` (`test_partition_to_node_labels_orders_by_size_descending`, `test_partition_to_node_labels_ties_broken_by_smallest_node_id`, `test_set_to_indicator_flags_membership`) to the new path. Local validation: `make format` clean (118 files unchanged, ruff check passed), `make test` green (954 passed at 100% coverage including `_helpers.py` at 12/12 statements), `make ty` clean, `make docs` green. No CHANGELOG entry (pure internal restructure; private helpers carry leading underscore so no third-party reach-in surface change).
