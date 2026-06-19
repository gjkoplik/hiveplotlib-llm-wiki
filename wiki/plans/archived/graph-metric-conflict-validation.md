# Plan: Up-front graph-type conflict validation in `compute_graph_metrics`

## Goal

When a user requests graph metrics whose graph-type requirements are mutually incompatible in a single
`compute_graph_metrics` call (e.g. `node_graph_metrics=["in_degree", "triangles"]`, where `in_degree` needs a directed
graph and `triangles` needs an undirected one), they hit a confusing whiplash today: one metric succeeds, the next raises
"build with `directed=False`", they flip the flag, re-run, and now the *other* metric raises "build with `directed=True`".
Neither error acknowledges the two are irreconcilable in one call, and expensive metrics computed before the raise are
thrown away. This plan adds a single up-front validation pass (over the full node+edge requested set, before any metric
runs) that detects an unsatisfiable directedness or multigraph requirement and raises one decisive, actionable error
naming the conflicting metrics, stating they cannot share one graph, and giving the resolution (split into two calls).
It also fixes the more common single-metric stumble (`node_graph_metrics="triangles"` failing out of the box because the
default is `graph_directed=True`) by inferring the graph type from the requested set when that set is unambiguous. The
user-visible value: a clear, copy-pasteable path forward instead of a dead end, and the common "just give me triangles"
case works without the user having to know about `graph_directed`.

## Prior ADRs / design docs

- **`wiki/wiki/plans/networkx-metric-expansion-and-backend-refactor.md` — the plan whose restriction-raise story this
  extends.** That plan (and its predecessor `i-want-to-plan-optimized-hoare.md`) shipped the per-wrapper graph-type
  restriction raises (the ad-hoc `if graph.is_directed(): raise ValueError(...)` guards this plan reasons over), the
  `compute_graph_metrics` dispatcher, the `_check_metric_names` pre-validation pattern, the `_format_collision_msg`
  user-vocabulary message voice, and the `HivePlot` / `HivePlotMatrix` init-time and post-hoc graph-metric APIs
  (`node_graph_metrics`, `edge_graph_metrics`, `graph_directed`, `graph_multigraph`). This plan adds one validation pass
  and a `@requires_graph_type` decorator that replaces those hand-written guards with centralized enforcement plus a
  function-attached requirement record (amendment A5); it changes no metric coverage.
- **No ADRs exist yet** (`wiki/wiki/adr/` is not created). ADR promotion for the whole networkx story is **deferred to a
  single combined v0.28.0 close-out ADR** (per the prior plan's recorded deferral). This work feeds the *same* combined
  ADR. **QA must NOT flag ADR promotion at this plan's close**, and no per-plan ADR is spawned.

### Standalone plan vs. fold-in decision

**Decision: standalone plan (this file), not a new workstream inside
`networkx-metric-expansion-and-backend-refactor.md`.**

Rationale. Research liaison's read is correct that this is a natural extension of that plan's restriction-raise story.
But (1) execution of the networkx work was already split once for size (the prior plan spun out of
`i-want-to-plan-optimized-hoare.md` for exactly that reason), and the metric-expansion plan is already long with its own
in-flight workstream chatter and API-critic threads; (2) this work has a distinct, self-contained user-facing deliverable
(one new validation behavior) with its own API-critic planning pass, its own test surface, and a hard notebook-gating
constraint that wants its own prominent home; (3) it does not touch metric coverage, the package restructure, or the
igraph roadmap that the prior plan owns. Both plans still promote into the *same* combined v0.28.0 ADR, so nothing is lost
by keeping the planning docs separate. A dedicated, short plan keeps the conflict-validation story legible.

## Patterns this replaces

**Amended (see amendment A5): the hand-written guard blocks are now REPLACED, not held out.** This plan introduces a
`@requires_graph_type` decorator on each metric wrapper that (1) attaches a `_hpl_graph_type_requirement` record to the wrapped
function and (2) generates the enforcement: it wraps the body so a single shared helper `_enforce_graph_type(name,
requirement, graph)` runs first and raises the `ValueError` if the graph violates the requirement. The ~20 hand-written
`if graph.is_directed(): raise` / `if graph.is_multigraph(): raise` blocks are deleted from the wrapper bodies; each body
shrinks to its happy path. So the guards are genuinely replaced by decorator-generated enforcement (the runtime raise
still happens, in one centralized place instead of ~20 inline copies). The requirement data lives on the functions (read
via `getattr(GRAPH_NODE_METRICS[m], "_hpl_graph_type_requirement", <default>)` across the existing node+edge dicts); there is **no
new standalone `GRAPH_METRIC_REQUIREMENTS` global** (Gary's explicit ask).

- **Ad-hoc per-wrapper graph-type guard blocks**, currently hand-written `if ...: raise ValueError(...)` clauses (deleted
  by Workstream A and regenerated by the decorator's `_enforce_graph_type`):
  - Directed-required guards (`if not graph.is_directed(): raise`): `node_metrics.py:55` (`in_degree`), `:81`
    (`out_degree`), `:304` (`in_degree_centrality`), `:330` (`out_degree_centrality`), `:624` (`reciprocity`), `:703`
    (`topological_generations`), `:857` (`strongly_connected_components`), `:894` (`weakly_connected_components`).
  - Undirected-required guards (`if graph.is_directed(): raise`): `node_metrics.py:264` (`triangles`), `:394`
    (`onion_layers`, also multigraph-reject at `:401`), `:654` (`articulation_points`), `:788`
    (`label_propagation_communities`), `:822` (`connected_components`); `edge_metrics.py:84` (`bridges`), `:140`
    (link-prediction factory directed-reject).
  - Multigraph-reject guards (`if graph.is_multigraph(): raise`): `node_metrics.py:151` (`eigenvector_centrality`), `:203`
    (`clustering`), `:232` (`core_number`), `:401` (`onion_layers`), `:548` (`eigenvector_centrality_numpy`);
    `edge_metrics.py:147` (link-prediction factory multigraph-reject, covering `jaccard_coefficient`,
    `adamic_adar_index`, `preferential_attachment`, `resource_allocation_index`, `common_neighbor_centrality`).
  - **Replace with:** the `@requires_graph_type` decorator (Workstream A). Each wrapper gains the decorator and loses its
    hand-written guard block; the decorator attaches a `_hpl_graph_type_requirement` record AND generates the runtime raise via the
    shared `_enforce_graph_type` helper. The decorator-attached record is the single source of truth that both the new
    up-front validator (Workstream B) and the docs metric-table directive (future pass) can read. There is no new
    standalone requirements-dict global. QA's replace-and-sweep should confirm **no** hand-written `if
    graph.is_directed(): raise` / `if graph.is_multigraph(): raise` blocks survive in the wrapper bodies; the
    message-fidelity sweep test in Workstream A proves the centralized raise is behavior- and message-equivalent to the
    deleted guards.

- **Duplication risk, flagged not fixed:** `docs/source/_ext/metric_table_directive.py` already classifies the same
  graph-type axes (`directed_only`, `undirected_only`, `simple_only`, `simple_undirected_only`, `dag_required` at
  `:147-181`) by *empirically probing* each wrapper against four graph variants (`metric_table_directive.py:101-137`).
  After this plan the requirement facts live in two places: the decorator-attached `_hpl_graph_type_requirement` record (now the
  single source of truth for both the runtime raise and the up-front validator) and this independent, empirical docs
  probe (which discovers requirements by running each wrapper against four graph variants). The docs probe does not
  consume `_hpl_graph_type_requirement` today and is not in scope to change here. Flag for a future pass: the docs directive could
  read `_hpl_graph_type_requirement` instead of probing, collapsing the two encodings into one. Out of scope for this plan; noted so
  the duplication is on the record.

## Default justifications

- **`graph_directed` inference when the requested set is unambiguous (Workstream C).** When every requested metric that
  declares a directedness requirement agrees (all directed-required, or all undirected-required, or none care), and the
  caller did **not** pass an explicit `graph_directed`, the dispatcher infers the satisfying value rather than defaulting
  blindly to `True`. Justification grounded in workflow: a user typing `node_graph_metrics="triangles"` is asking for a
  triangle count; they should not have to know that the internal graph defaults to directed and that triangles needs it
  undirected. Inference removes the single most common stumble. **The inference is opt-out: any explicit `graph_directed`
  value the user passes always wins** (and then the conflict validator explains why, if it conflicts with a declared
  requirement). When the requested set is itself contradictory (a directed-required and an undirected-required metric
  together), inference cannot resolve it and the Workstream B validator raises instead. The multigraph axis needs no
  inference default: no metric *requires* a multigraph, so the existing `graph_multigraph=False` default already satisfies
  every metric's requirement; a user who explicitly asks for `graph_multigraph=True` plus a multigraph-rejecting metric
  gets the conflict error.

No other new user-facing defaults. The new error is a raise, not a default. The registry is internal.

## Naming audit

User-facing names are minimal here (the headline change is error *behavior*, not new parameters). Check against the
NetworkX-adjacent and the library's own established vocabulary:

- **New parameters:** none. The validator and registry are internal; the error surfaces through the existing
  `compute_graph_metrics` / `HivePlot` / `HivePlotMatrix` signatures unchanged.
- **New methods/classes:** none user-facing. Internal helpers only (names are naming-audit-exempt but listed for the
  implementing engineer in Workstream A).
- **Internal decorator + record (engineer's call on exact shape, illustrative):** a `@requires_graph_type(*,
  directed=None, multigraph_ok=True, hint=None)` decorator that attaches a `GraphTypeRequirement` record to the wrapped
  function as `_hpl_graph_type_requirement`. The record carries `requires_directed: Optional[bool]` (`True` = needs directed,
  `False` = needs undirected, `None` = agnostic), `rejects_multigraph: bool` (derived from `multigraph_ok`), and an
  optional `hint: Optional[str]` (the metric-specific message tail, e.g. the components-family cross-references). Field
  names mirror the guard-clause vocabulary already in the codebase (`is_directed`, `is_multigraph`) and the docstring
  phrasing ("requires a directed graph", "does not support multigraphs"). The tri-state `Optional[bool]` on directedness
  matches the genuine three-way nature of that axis (required / forbidden / agnostic); the boolean `rejects_multigraph`
  matches the asymmetric nature of that axis (some reject, none require). These names are internal (naming-audit-exempt)
  but fixed here so Workstreams A and B share one vocabulary. **No standalone requirements-dict global** (Gary's explicit
  ask): consumers read the record off the function via `getattr(GRAPH_NODE_METRICS[m], "_hpl_graph_type_requirement", <default>)`.
- **Error-message vocabulary (the real user-facing surface):** the new conflict message adopts the established voice from
  `_format_collision_msg` (`graph_features/__init__.py:180-201`) and the per-wrapper guards: name the metric(s) in
  backticks, name the requirement in plain language, and reference *both* the function kwarg (`graph_directed`) and the
  HivePlot-level kwarg the way the collision message references `rename_kwarg` + `hive_plot_kwarg`. The resolution
  sentence uses the same "Build the source graph with `directed=...`" phrasing the guards already use, extended to "these
  two cannot share one graph; split into two calls."

No prose-only terms introduced beyond the error text itself.

## API usage examples

This work modifies user-facing API behavior (a new validation raise plus graph-type inference), so an api-critic
planning-mode pass is required. Snippets below are the runnable cases a user will hit.

### Proposed (from planner / Orchestrator)

```python
# Example 1: the conflict case — directed-required and undirected-required node metrics in one call.
# Today: whiplash (one succeeds, the other raises "flip the flag", repeat). After this plan: one decisive error up front.
# Example data:
import networkx as nx
from hiveplotlib.converters import networkx_to_nodes_edges
from hiveplotlib.graph_features import compute_graph_metrics

G = nx.karate_club_graph()
nodes, edges = networkx_to_nodes_edges(G)

# Call site:
# raises ValueError up front, before any metric is computed. Because this is a direct
# `compute_graph_metrics` call (no `graph_directed` parameter exists here), the message
# does NOT mention `graph_directed`; it tells you to build a graph of each satisfying type:
#   `in_degree` requires a directed graph; `triangles` requires an undirected graph.
#   These cannot share one `nx.Graph` in a single call. Compute them in two separate calls,
#   building a directed graph for `in_degree` and an undirected graph for `triangles`
#   (each call returns fresh nodes/edges you can chain) ...
nodes, _ = compute_graph_metrics(
    G,
    node_metrics=["in_degree", "triangles"],
    target_nodes=nodes,
    target_edges=edges,
)
```

```python
# Example 2: the resolution the error points to. The key point (amendment A2): each call needs
# its OWN graph of the satisfying type, not just a different metric list against the same graph.
# `in_degree` needs a genuinely directed graph; reusing the undirected karate graph from Example 1
# would make `in_degree` meaningless (in-degree equals out-degree on an undirected graph), so we
# build a directed graph for the first call and an undirected view for the second. The chaining
# handles the nodes/edges; the user supplies the right graph object per call.
# Example data:
import networkx as nx
from hiveplotlib.converters import networkx_to_nodes_edges
from hiveplotlib.graph_features import compute_graph_metrics

G_directed = nx.DiGraph(nx.karate_club_graph())
nodes, edges = networkx_to_nodes_edges(G_directed)

# Call site:
# first call: directed metric on the directed graph
nodes, edges = compute_graph_metrics(
    G_directed,
    node_metrics="in_degree",
    target_nodes=nodes,
    target_edges=edges,
)
# second call: undirected metric on the undirected view, chaining the augmented nodes/edges
G_undirected = G_directed.to_undirected()
nodes, edges = compute_graph_metrics(
    G_undirected,
    node_metrics="triangles",
    target_nodes=nodes,
    target_edges=edges,
)
# nodes.data now has both 'in_degree' and 'triangles' columns
```

```python
# Example 3: the common single-metric stumble, fixed by inference.
# Today: HivePlot(graph=G, node_graph_metrics="triangles") fails because graph_directed defaults to True.
# After this plan: the dispatcher infers graph_directed=False because every requested metric agrees on undirected.
# Example data:
import networkx as nx
from hiveplotlib import HivePlot

G = nx.karate_club_graph()

# Call site:
hp = HivePlot(
    graph=G,
    partition_variable="club",
    sorting_variables="triangles",
    node_graph_metrics="triangles",
    # no graph_directed passed: inferred as False since `triangles` is undirected-only
)
fig, ax = hp.plot()
```

```python
# Example 4: explicit graph_directed always wins over inference; if it conflicts, the validator explains why.
# Example data:
import networkx as nx
from hiveplotlib import HivePlot

G = nx.karate_club_graph()

# Call site:
# raises ValueError: `triangles` requires an undirected graph, but graph_directed=True was passed explicitly.
#   Pass graph_directed=False (or omit it to let it be inferred) ...
hp = HivePlot(
    graph=G,
    partition_variable="club",
    sorting_variables="degree",
    node_graph_metrics="triangles",
    graph_directed=True,  # explicit, conflicts with triangles' undirected requirement
)
```

```python
# Example 5: cross-boundary conflict (node metric vs. edge metric sharing one graph).
# `in_degree` (node, directed-required) and `bridges` (edge, undirected-required) cannot share one graph.
# Example data:
import networkx as nx
from hiveplotlib.converters import networkx_to_nodes_edges
from hiveplotlib.graph_features import compute_graph_metrics

G = nx.karate_club_graph()
nodes, edges = networkx_to_nodes_edges(G)

# Call site:
# raises ValueError up front naming both the node metric and the edge metric and their opposing
# requirements. Direct-dispatcher path, so the resolution tells the user to build a directed graph
# for `in_degree` and an undirected graph for `bridges` and run two calls (no `graph_directed`).
nodes, edges = compute_graph_metrics(
    G,
    node_metrics="in_degree",
    edge_metrics="bridges",
    target_nodes=nodes,
    target_edges=edges,
)
```

### API Critic's take (planning mode)

Walked the surface as a user hitting the conflict, recovering, and exercising inference. Verified signatures against
`graph_features/__init__.py:275` (`compute_graph_metrics`) and the resolution-site framing in Workstream C. The error
behavior is the right call and the inference default is well-justified. Three friction points, two of them must-fix
because they put a wrong instruction in the user's hands.

**1. (must-fix) The error in Examples 1 and 5 must not tell `compute_graph_metrics` callers to "reference `graph_directed`."**
`compute_graph_metrics(graph, *, node_metrics=..., ...)` receives an *already-built* `nx.Graph` and has **no
`graph_directed` parameter** (verified at `graph_features/__init__.py:275-287`). Inference lives one level up, in
`_apply_graph_metrics` (Workstream C correctly establishes this). So a user calling the dispatcher directly (Examples 1,
2, 5) who is told to "pass `graph_directed=False`" will go looking for a kwarg that does not exist on the function they
called. Workstream B's done-when says the message references "both `graph_directed` and the HivePlot-level
`graph_directed` kwarg" as if the dispatcher had its own flag; it doesn't. Preferred: the conflict message must branch on
entry point. For the **direct `compute_graph_metrics`** path the resolution is "these two metrics cannot share one
`nx.Graph`; compute them in two calls, building a directed graph for `in_degree` and an undirected graph for `triangles`
(e.g. via `to_undirected()`)." For the **HivePlot/HivePlotMatrix** path the resolution references `graph_directed` (the
kwarg that genuinely exists there). One message that names a nonexistent function kwarg fails the first-error walk for
the dispatcher's own callers. (Mirrors the expertise entry "the recovery message the user hits determines whether the
transition feels supported or hostile.")

**2. (must-fix) Example 2's resolution is heavier than the error sentence promises — the prose undersells the rebuild.**
The error says "split into two calls (each returns fresh nodes/edges you can chain)." But Example 2 shows the user must
*also* build two different graphs (`G_directed` and `G_directed.to_undirected()`) and, in Example 1, the starting graph
is the undirected `nx.karate_club_graph()`, which has **no** directed view to give `in_degree` meaningful values. "Two
calls" frames the fix as re-invoking with a different metric list; the real work is constructing the right graph object
per call. The error sentence must say so explicitly: the chaining handles the *nodes/edges*, but the user supplies a
directed graph for one call and an undirected one for the other. Otherwise a user copy-pastes "two calls," keeps the same
graph, and `in_degree` on an undirected karate graph silently returns degree-equals-out (or raises again), reopening the
whiplash this plan exists to close. Recommend Workstream B's done-when add: "the resolution sentence names that each call
needs its own graph of the satisfying type, not just a different metric list."

**3. (worth-discussing) The inference/conflict asymmetry across entry points should be stated in one place.**
At the HivePlot level a user gets *both* inference (Example 3, the common case "just works") and the conflict raise
(Example 4). At the bare `compute_graph_metrics` level a user gets *only* the conflict raise — no inference is possible
because the graph is pre-built, so `node_metrics="triangles"` on a directed graph still hits the per-wrapper guard, not a
friendly inference. This is defensible (Workstream C is explicit that inference is a HivePlot-level feature), but a user
who learns the "triangles just works" behavior from the HivePlot path and then drops down to `compute_graph_metrics`
will be surprised it doesn't. Recommend the Workstream D dispatcher docstring state plainly: "`compute_graph_metrics`
does not infer graph type; it validates the requested set against the graph you pass. Graph-type inference is a
`HivePlot` / `HivePlotMatrix` construction-time convenience." One sentence closes the cliff.

**Agreed on the rest.** The tri-state `requires_directed` / boolean `rejects_multigraph` registry shape matches the
genuine asymmetry of the two axes. Explicit-`graph_directed`-always-wins (Example 4) is the right precedence and the
error that explains the explicit conflict is the correct follow-through. The multigraph axis needing no inference default
(no metric *requires* a multigraph) is sound. Cross-boundary node-vs-edge conflict detection (Example 5) is correct given
node and edge metrics share the one internal graph — good that the plan caught it rather than validating the two sets
independently.

**Recurring pattern for the post-impl pass:** the load-bearing surface here is the *error string*, not a signature. Both
must-fix items are about the message telling the user the right next move. Post-impl should read the actual raised
message text (not just assert a `ValueError` fires) for each of the five examples, and confirm the message branches
correctly between the dispatcher-direct and HivePlot entry points.

### API Critic — post-implementation review

```
Status: propose
API surface reviewed: compute_graph_metrics (new _check_graph_type_conflicts validator + branched
  error message), HivePlot.__init__ / HivePlot.compute_graph_metrics (graph_directed inference),
  HivePlotMatrix.from_partition / from_variable_sweep (inference) / __init__ / from_tags (no-infer
  asymmetry) / compute_graph_metrics, _infer_graph_directed.
Concerns:
  - [worth-discussing] The single-metric guard a HivePlotMatrix(..., node_graph_metrics="triangles")
    user hits names `directed=False`, a kwarg that does not exist on the class they called — at
    src/hiveplotlib/graph_features/networkx/_helpers.py:65-71.
    Suggested change: see ruling on task 4 below; either the guard names the HivePlot-level kwarg
    `graph_directed=False`, or accept-as-shipped with the docstring steer as mitigation.
  - [low-confidence] `_check_graph_type_conflicts` directedness-standoff head references
    `directed={graph.is_directed()}` for BOTH entry-point branches, including the dispatcher-direct
    one — at src/hiveplotlib/graph_features/__init__.py:248. `is_directed` is a graph property a
    dispatcher caller can reason about, so this is fine; flagged only because the planning must-fix
    was "dispatcher message must not name a nonexistent FUNCTION kwarg," and `directed=` here reads
    as a graph property, not the `graph_directed` param. Verified it is NOT the banned `graph_directed`.
Test-method-coverage audit: clean — sampled hiveplot_matrix_test.py:3719/3738/3760/3788 (the
  asymmetry-wall tests call HivePlotMatrix.__init__ / from_tags with node_graph_metrics="triangles"
  in-body), hiveplot_matrix_test.py:3537/3640 (from_* inference paths), and the WS-B message-branch
  pair in graph_features_test.py (per WS-B/C logs). Methods this workstream touched are exercised by
  same-named tests.
```

**Walk of the four real tasks (the load-bearing output here is the error-string experience, per the
planning recurring-pattern note).**

**Task 1 — dispatcher-direct `["in_degree", "triangles"]` recovery: PASS.** The validator at
`graph_features/__init__.py:257-265` (the `else` branch, `from_hive_plot=False`) raises: "... Split this
into two calls, each against its OWN graph of the satisfying type: build a directed graph for `in_degree`
and an undirected graph (such as `graph.to_undirected()`) for `triangles`, then chain the returned
nodes/edges. A different metric list against the same graph is not enough; each call needs a graph of the
right type." This closes the exact planning failure mode (the must-fix #2 risk that a user copy-pastes
"two calls" and reuses one graph). The message names a separate graph per side, names `to_undirected()`
as the concrete mechanism, and explicitly says "a different metric list against the same graph is not
enough." It does NOT mention `graph_directed` (must-fix #1 honored). Actionable as shipped.

**Task 2 — `HivePlot(...)` recovery via the `graph_directed` message: PASS.** The `from_hive_plot=True`
branch (`__init__.py:251-256`) raises: "... Split this into two calls: set `graph_directed=True` for
`in_degree` and `graph_directed=False` for `triangles`, then chain the returned nodes/edges." `graph_directed`
is a real kwarg on `HivePlot.__init__` and `HivePlot.compute_graph_metrics`, so the two-call recovery is
correct and the user can act on it directly. Both planning must-fixes are satisfied in their respective
branches.

**Task 3 — inference happy path and surprise surface: PASS / coherent.** `_infer_graph_directed`
(`hiveplot.py:187-226`) collects concrete `requires_directed` values, drops agnostics, and returns the
single concrete value when exactly one remains, else `default`. So `node_graph_metrics="triangles"` infers
undirected and "just works" (Example 3). Explicit-wins is gated cleanly: inference only runs `if not
graph_directed_explicit` (`hiveplot.py:2249`), and explicitness is captured as `graph_directed is not None
or graph is not None` before the `None` sentinel collapses (`hiveplot.py:2000`). The ambiguous-set-defers
behavior is coherent: a contradictory set yields two concrete values, `len(concrete) != 1`, so inference
returns `default` unchanged and the WS-B validator raises the decisive standoff error on the resolved set
rather than inference silently picking a side. That hand-off (inference declines, validator explains) reads
correctly from the user's seat. No surprise.

**Task 4 — HivePlotMatrix `__init__`/`from_tags` no-infer asymmetry: ACCEPT-AS-SHIPPED, with one
worth-discussing follow-up.** Ruling: the documented split (the `from_*` classmethods infer, `__init__`/
`from_tags` take a concrete `graph_directed=True` and never infer) is acceptable as shipped, NOT a must-fix.
Rationale: giving `__init__`/`from_tags` inference parity requires widening their signature to
`Optional[bool]=None`, a user-facing default change explicitly out of this plan's scope, and the asymmetry is
grounded in a real structural difference (`__init__`/`from_tags` take no `graph=` and a concrete bool, so
there is no sentinel to detect "user didn't pin it"). The docstrings at `hiveplot_matrix.py:267-275` and
`:1900-1908` document the asymmetry honestly and steer toward the `from_*` classmethods or an explicit
`graph_directed=False`. That steer is sufficient mitigation for the docstring-reading user.

The one wrinkle is the runtime wall a user hits WITHOUT reading the docstring. A
`HivePlotMatrix(..., node_graph_metrics="triangles")` call does not reach the WS-B set-conflict validator
(it is a single-metric request, not a standoff); it falls through to the per-metric `_enforce_graph_type`
guard, which raises (`_helpers.py:65-71`): "`triangles` does not support directed graphs (nx.DiGraph or
nx.MultiDiGraph). Build the source graph with `directed=False` (e.g., on the HivePlot initialization or
`nodes_edges_to_networkx`)." The kwarg this user actually needs on the class they called is
`graph_directed=False`, but the guard names `directed=False` and points at "the HivePlot initialization or
`nodes_edges_to_networkx`." A HivePlotMatrix `__init__` user reading only the traceback could go looking for
`directed=`, find no such kwarg, and stall before reaching the docstring's `graph_directed=False` steer. This
is a confusing-wall risk, but I rate it **worth-discussing, not must-fix**, because (a) the guard message is
the SHARED single-metric guard whose text WS-A locked as verbatim-equivalent to the pre-refactor message
(changing it for the HPM case alone would fork that shared string and is outside this plan's message-fidelity
contract), (b) the same guard already says "e.g., on the HivePlot initialization," which a reader can map to
"the construction-time `graph_directed` flag" with one inferential step, and (c) the docstring steer exists
for the user who reads it. If Gary wants to close the wall fully, the cleanest follow-up (separate plan,
since it touches the WS-A-locked shared guard text) is to have the guard append a HivePlot-level hint
("on a HivePlot / HivePlotMatrix, pass `graph_directed=False`") rather than only `directed=False`.

**Agreed on the rest.** Both planning must-fixes landed correctly and are branch-verified in source. The
A3 dispatcher-docstring inference-asymmetry sentence shipped (`__init__.py:418-422`), closing the "triangles
just works on HivePlot but not on the bare dispatcher" cliff. The multigraph one-sided error (`__init__.py:268-286`)
correctly branches its rebuild hint on entry point too (`graph_multigraph=False` for HivePlot vs.
`multigraph=False when building it` for the dispatcher), mirroring the directedness branch — good that the
multigraph axis got the same branch treatment rather than a shared string. Singular/plural agreement in the
message (`requires`/`require`, `does`/`do`, `this metric`/`these metrics`) is handled. No new public
parameters, so the signature surface is unchanged and the headline change is purely the error behavior, as
planned.

**Workstream F addendum (post-impl, 2026-05-29) — closes the Task-4 wrinkle above. Verdict: AGREED-AS-SHIPPED.**

```
Status: clean
API surface reviewed (F): the three _enforce_graph_type single-metric guard message templates
  (src/hiveplotlib/graph_features/networkx/_helpers.py:58-81) — directed-required, undirected-required,
  multigraph-rejected. No signature change (A6 option B rejected, confirmed); resolution-text only.
Concerns: none must-fix.
Test-method-coverage audit (F): not re-run; the three strings are reconstructed and exact-match
  asserted by _expected_*_msg helpers in graph_features_test.py:1199-1222 (per the F test-engineer log).
```

Focused-question answer: **yes, resolved.** The wrinkle I raised under Task 4 was that a
`HivePlotMatrix(..., node_graph_metrics="triangles")` user (who can't benefit from the `from_*` inference)
hit a guard naming only `directed=False`, a kwarg absent from the constructor they called, and could stall
before reaching the docstring `graph_directed=False` steer. The shipped undirected-required string
(`_helpers.py:66-73`) now reads: "Build the source graph as undirected: pass `graph_directed=False` on
`HivePlot` / `HivePlotMatrix` initialization, or `directed=False` on `nodes_edges_to_networkx`." That
`graph_directed=False` is a real public kwarg on **both** entry points (verified, not assumed):
`HivePlotMatrix.__init__` declares `graph_directed: bool = True` and `graph_multigraph: bool = False`
(`hiveplot_matrix.py:321-322`); `HivePlot.__init__` declares `graph_directed: Optional[bool] = None` and
`graph_multigraph: Optional[bool] = None` (`hiveplot.py:1980-1981`). So the stalled HPM `__init__`/`from_tags`
user now sees, in the traceback itself, the exact kwarg name they need on the class they actually called. The
directed-required and multigraph-rejected branches mirror the fix with `graph_directed=True` / `directed=True`
and `graph_multigraph=False` / `multigraph=False` respectively, both real kwargs on both constructors.

Dispatcher-direct reader check (the one residual risk worth a sentence): a `compute_graph_metrics(...)` caller
who built their own graph and called the dispatcher directly has *neither* `graph_directed` nor `graph_multigraph`
(those are construction-time class kwargs; the dispatcher takes a built `graph`). The single sentence names both
knobs at their respective homes ("... pass `graph_directed=False` on `HivePlot` / `HivePlotMatrix` initialization,
**or** `directed=False` on `nodes_edges_to_networkx`"). The `or` cleanly partitions the two audiences: the
dispatcher-direct caller reads past the class-level clause to the `nodes_edges_to_networkx` clause, which is the
helper that actually built their graph. This reads clearly rather than confusingly — the class-level kwarg is
explicitly scoped to "initialization," so a dispatcher caller is not misled into hunting for a `graph_directed`
arg on `compute_graph_metrics`. (Note: a dispatcher caller who hand-rolled a `nx.DiGraph` directly, not via
`nodes_edges_to_networkx`, gets neither named knob and must reason from the leading "Build the source graph as
undirected" clause — same one-inferential-step situation the message already had, unchanged by F and not worsened.)

A6 scope held: option B (widening the HPM `bool` defaults to `Optional[bool]`) was correctly NOT taken — the
signatures are unchanged, so the deliberate no-infer asymmetry stands and only the resolution text moved. This is
the right shape for the fix: it closes the runtime-wall confusion without a user-facing default change. Nothing to
propose.

### Workstream G / A8 — complexity read on the `graph_directed` / `graph_multigraph` pair (post-impl, 2026-05-31)

```
Status: propose
API surface reviewed: HivePlot.__init__ (graph_directed/graph_multigraph/graph_source_attribute_name),
  HivePlot.compute_graph_metrics, HivePlotMatrix.__init__, HivePlotMatrix.from_partition /
  from_variable_sweep / from_tags, HivePlotMatrix.compute_graph_metrics. (A8 itself is a private
  internal refactor: _graph_directed: Optional[bool] replacing the (_graph_directed,
  _graph_directed_explicit) pair. PUBLIC signatures unchanged by A8; this review is the
  user-complexity audit Gary requested across the whole cluster, not a sign-off on the refactor.)
Concerns:
  - [should-fix] The `graph_directed` default TYPE silently differs across the HivePlotMatrix surface:
    Optional[bool]=None (infers) on from_partition/from_variable_sweep, but bool=True (never infers)
    on __init__/from_tags. A user reading the four matrix constructors side by side sees the same
    param name behave differently with no name signal — at hiveplot_matrix.py:320, :977, :1328, :1761.
    Suggested change: leave the behavior (it is structurally grounded), but make the docstring on
    __init__/from_tags open with one verbatim-shared sentence ("This constructor never infers
    directedness; see from_partition for the inferring path") so the divergence is discoverable from
    the tooltip, not only from a careful cross-read.
  - [worth-discussing] graph_directed and graph_multigraph read as a matched pair (same prefix, same
    "internal graph type" framing, adjacent in every signature) but None means two different things:
    infer-per-call for directed, follow-construction for multigraph. The asymmetry is honest and
    documented, but the visual pairing primes a user to expect symmetry — at hiveplot.py:1895-1916,
    hiveplot_matrix.py:267-278. Suggested change: doc/framing only; lead the graph_multigraph
    docstring with "Unlike graph_directed, this is never inferred" (the HPM __init__ docstring at
    :276 already does this; HivePlot.__init__ at :1908 buries it mid-paragraph — hoist it to the
    first clause for parity).
Test-method-coverage audit: clean (scope: A8 touched only the private stash representation). The
  public compute_graph_metrics / __init__ / from_* methods are unchanged in signature and were
  covered under WS-B/C/F (see the audits above); A8's stash collapse is exercised through the same
  per-call override and pinning tests since the observable behavior is identical.
```

**Primary verdict (Gary's actual question): the complexity is correct-but-heavy, and the weight is
real, but it is mostly ESSENTIAL complexity sitting behind one avoidable PUBLIC tax. I would not call
the feature over-built. Two doc/framing fixes (above) buy most of the relief; no signature change is
warranted.** Detail per the four axes Gary named:

**Axis 1 — the behavior matrix per construction path.** Walked as a user. The cases are: (a)
nodes/edges unpinned → infer per call, (b) nodes/edges pinned → fixed, (c) graph-built → pinned to
input type, (d) per-call override on `compute_graph_metrics`, plus the metric-requirement interaction
and the up-front conflict error. That is genuinely a lot of cases on paper, but the user does NOT meet
them all at once: the natural reading is a single rule, "the internal graph's directedness is whatever
your input most strongly implies, and you can always override it." A user starting from `graph=` never
thinks about (a)/(b); a user starting from nodes/edges with one metric never thinks about (c). The
matrix is learnable because the paths are disjoint in practice. The inference ("magic") is a NET HELP,
not a hidden case: it removes the single sharpest historical stumble (the old "triangles defaults to
directed and raises" trap that cell `e6661ae3` calls out as a behavior change). The cost of inference
is exactly one new sentence a user must hold ("on nodes/edges I infer; on graph I don't"), and the
notebook teaches it in two clearly-separated subsections. Verdict: learnable, inference earns its keep.
LEAVE-AS-IS on the behavior; the only debt is making the one inference rule MORE prominent (axis 2).

**Axis 2 — graph_directed vs graph_multigraph asymmetry.** This is where the avoidable weight sits.
The two params look like a matched set and behave differently, and `None` is overloaded across them
(infer vs follow-construction). The behavior is honest (multigraph has no inference channel; directed
does), so the asymmetry is a FEATURE in the sense that collapsing it would lie. But it is a discoverability
TAX: nothing in the names or signature ordering signals the divergence, so a user has to read both full
docstring paragraphs to learn it. The HPM `__init__` docstring already leads with "Like graph_directed,
it is never inferred" / "This constructor takes a concrete bool default and never infers" — that is the
right move. `HivePlot.__init__` does NOT lead with it (the "never inferred" clause is buried at
:1910 mid-paragraph). The cheap, high-leverage fix is purely doc framing: hoist the "unlike
graph_directed, never inferred" contrast into the FIRST clause of every graph_multigraph docstring, and
add the inference-vs-no-inference one-liner to the head of the cluster. should-fix on the framing,
LEAVE-AS-IS on the semantics.

**Axis 3 — HivePlot vs HivePlotMatrix duplication, and the matrix from_* spread.** This is the second
real source of load, and it is partly avoidable. The cluster appears on `HivePlot.__init__`,
`HivePlot.compute_graph_metrics`, `HivePlotMatrix.__init__`, three matrix `from_*` constructors, and
`HivePlotMatrix.compute_graph_metrics` — seven surfaces. The genuinely confusing wrinkle is INSIDE the
matrix: `from_partition` / `from_variable_sweep` take `graph_directed: Optional[bool]=None` (they infer,
because they accept nodes/edges or a `graph` and route through `_resolve_graph_or_nodes_edges`), while
`__init__` / `from_tags` take `graph_directed: bool=True` (they never infer). Same param name, four
constructors on ONE class, two different default types and two different behaviors, with no name signal.
A user who learns "the matrix infers like HivePlot does" from `from_partition` and then reaches for
`from_tags` is silently wrong. This is the sharpest single ergonomic edge in the whole cluster. It is
structurally grounded (`__init__` takes pre-built `HivePlot`s and `from_tags` takes a concrete bool, so
there is no `None` sentinel to mean "user didn't pin it"), so I am NOT recommending a signature change
(that was A6 option B, deliberately rejected, and widening to `Optional[bool]=None` is a user-facing
default change out of scope). But it should be DISCOVERABLE from the tooltip: the should-fix above asks
for a verbatim-shared lead sentence on `__init__`/`from_tags` naming the no-infer behavior and pointing
at the inferring constructors. As for "which directedness applies where" (the matrix's own `_graph_directed`
stash vs the child HivePlots'): a user does not see this — the matrix builds its cells and stashes one
directedness; there is no observable split. Not a user-facing concern. LEAVE-AS-IS on structure,
should-fix on the cross-constructor docstring discoverability.

**Axis 4 — graph_source_attribute_name riding along.** Low impact. It is a third member of a cluster
that already carries two confusable params, so it adds visual length to every signature and docstring,
but it is correctly defaulted (`None` → `"_hiveplotlib_source"`), only matters when
`graph_multigraph=True`, and every docstring scopes it to that case with "override only if it collides."
A first-time user correctly ignores it. It marginally worsens the "this cluster is big" first impression
but adds no decision the user has to make. LEAVE-AS-IS; if the cluster ever gets a doc-level grouping
header, this param belongs last and flagged as advanced.

**Top simplifications, ranked by leverage (all doc/framing; no code/signature change):**
1. (should-fix) Make the matrix `__init__`/`from_tags` no-infer behavior discoverable from the tooltip
   with a shared lead sentence pointing at the inferring `from_*` constructors. Closes the sharpest edge
   (axis 3) without touching the deliberately-rejected signature widening.
2. (should-fix) Hoist "unlike graph_directed, never inferred" to the first clause of every
   graph_multigraph docstring (HivePlot.__init__ is the laggard at :1908). Closes the asymmetry-surprise
   (axis 2) with a one-clause move that HPM __init__ already models.
3. (nice-to-have) Add a one-paragraph "Controlling the internal graph type" framing note at the head of
   the cluster (the notebook section `b69c9a50` already exists and is good; a docstring-level pointer to
   it from `HivePlot.__init__` would let the API-reference reader find the single mental model without
   reading the notebook).

None of these is must-fix: the behavior is correct, the notebook teaches it well, and the error
messages (post-WS-F) name the right kwarg on the right class. The complexity is justified by the
underlying reality (networkx genuinely makes directedness/multigraph load-bearing per metric); the only
unjustified part is that two of the cheapest signals (a shared lead sentence, a hoisted clause) are not
yet in place, leaving a careful-cross-read tax that a tooltip-skimming user pays as silent wrongness.
Route the two should-fix items to a docs-scope follow-up; they are pure prose moves with no rendering
risk and no behavior change.

## Notebook review

### Editorial-critic RE-CLEAR (post-impl, closure reconcile 2026-06-18)

**Verdict: ready-to-ship. No must-fix.** An editorial-critic post-impl pass on the current `examples/computing_graph_metrics.ipynb` (which carries both Workstream E's structural revision of the `## Controlling the HivePlot Internal Graph Type` section and Workstream L's `## Parallel Edge` collapse-warning demo) returned ready-to-ship. The notebook is the right home, the datasets are coherent (Karate Club for `graph=` cases, the existing toy for nodes/edges cases), there is no genre drift (still a gallery `compute_graph_metrics` / `HivePlot` demonstration), and no class-scope drift (still a `HivePlot` page; `HivePlotMatrix` is referenced in prose only).

The pass's only finding (notebook length / the internal-graph-type material split across two sub-sections) is **permanently declined** per the maintainer's single-notebook decision: the graph-metrics story stays in one notebook rather than being split into a separate internal-graph-type notebook. This is a standing design call, not an open item.

**This closes the editorial gate for both Workstream E and Workstream L.** The original placeholder ("Pending — invoke editorial-critic in post-implementation mode after Workstream E ships") is satisfied; both workstreams' editorial-critic done-whens (the WS-E A7 gate and WS-L done-when #11) are met by this verdict.

Workstream E is the **only** notebook-touching workstream. The gate is now **lifted** (amendment A7) and the scope is a
**structural revision**, not a blurb. The notebook (`examples/computing_graph_metrics.ipynb`) is a gallery notebook; its
existing internal-graph-type section is being restructured to teach the new graph-type handling precisely (defaults,
friendly inference, and a set of break-case cells that error in controlled `try/except` blocks). **No new dataset**
(Karate Club for `graph=` cases, the existing `example_hive_plot` toy for nodes/edges cases, both already present),
**no genre drift** (still a gallery `compute_graph_metrics` / `HivePlot` demonstration; the break-cases are mechanic
demos, not a tutorial pivot), and **no class-scope change** (still a `HivePlot` page; `HivePlotMatrix` is referenced in
prose only, not made the subject). Because the revision restructures an existing section and adds error-demo cells but
introduces no new dataset and no genre/class drift, editorial-critic's pass is a structure-and-coherence check, not a
sign-off-gated dataset/genre change. Gary explicitly asked for editorial-critic involvement on this revision.

## Workstreams

Dependency order: A (decorator + enforcement) → B (validator + error) and C (inference) can run after A, B and C may run concurrently
but coordinate on the resolved-flag path → D (propagation/docstrings) after B and C → F (guard-message clarity) after D →
**E (notebook structural revision) LAST**. (A-D and F have shipped; E's original hard gate is LIFTED per amendment A7.)

### Workstream A: `@requires_graph_type` decorator + centralized enforcement (replaces the hand-written guards)

**Status:** ✅ complete (closure reconcile 2026-06-18: header was stale; Implementation log "Workstream A complete" 2026-05-29, plus the A-tests entry). The `@requires_graph_type` decorator and `_enforce_graph_type` helper are in the shipped `graph_features/networkx/_helpers.py`, and all ~20 wrappers are decorated.
**Posture (amended A5):** Gary locked the FULLER decorator approach. This workstream does NOT add a parallel registry on
top of the guards; it **replaces** the hand-written guards with a decorator that both declares the requirement and
generates the raise. Net fewer lines, but a **wider sweep**: all ~20 wrapper bodies change (each gains a decorator and
loses its guard block), where the prior registry-only framing touched none of the bodies. Reflect this in scope/risk.
**Files:** `src/hiveplotlib/graph_features/networkx/_helpers.py` (or a small new sibling, engineer's call) for the
`@requires_graph_type` decorator, the `GraphTypeRequirement` record, and the shared `_enforce_graph_type(name,
requirement, graph)` helper; `src/hiveplotlib/graph_features/networkx/node_metrics.py` and
`.../edge_metrics.py` (decorate every guarded wrapper, delete the guard blocks); plus tests under `tests/`.
**First step (confirm before designing):** confirm the *actual current form* of the guards on this branch. Verified
2026-05-29: hand-written `if not graph.is_directed(): raise ValueError(...)` / `if graph.is_multigraph(): raise
ValueError(...)` clauses (e.g. `node_metrics.py:55`, `:394`, `:401`), **NOT** a `@requires_graph_type` decorator. If the
engineer instead finds the decorator already present, halt under rule 9 and surface the mismatch.
**Done when:**
- A `@requires_graph_type(*, directed=None, multigraph_ok=True, hint=None)` decorator exists. Applied to a wrapper it (1)
  attaches a `GraphTypeRequirement` record as `_hpl_graph_type_requirement` (`requires_directed: Optional[bool]`,
  `rejects_multigraph: bool` from `not multigraph_ok`, `hint: Optional[str]`), and (2) wraps the body so the shared
  `_enforce_graph_type(name, requirement, graph)` helper runs first and raises the `ValueError` on violation.
- Every guarded wrapper in `GRAPH_NODE_METRICS` and `GRAPH_EDGE_METRICS` is decorated and its hand-written guard block is
  deleted; the body shrinks to its happy path. A metric with no guard either records `requires_directed=None,
  rejects_multigraph=False` (decorated as a no-op) or carries no `_hpl_graph_type_requirement` (consumers default it the same way
  via `getattr(..., _hpl_graph_type_requirement, <default>)` — engineer's call, but the default must match an unconstrained metric).
- **No new standalone global.** Requirement data is read off the functions via `getattr(GRAPH_NODE_METRICS[m],
  "_hpl_graph_type_requirement", <default>)` across the existing node+edge dicts. No `GRAPH_METRIC_REQUIREMENTS` dict is introduced.
- **Message fidelity (carve-outs preserved):** `_enforce_graph_type` reproduces the current message text exactly:
  - Standard directed-required / undirected-required / multigraph-rejected messages keep their current "Build the source
    graph with `directed=True`/`directed=False`/`multigraph=False` (e.g., on the HivePlot initialization or
    `nodes_edges_to_networkx`)." phrasing.
  - **Components-family hints** survive via `hint=`: `connected_components` (undirected-required) ends with "For directed
    graphs, see `strongly_connected_components` and `weakly_connected_components`."; `strongly_connected_components` and
    `weakly_connected_components` (both directed-required) end with "For undirected graphs, see `connected_components`."
    These tails are NOT flattened away; they are passed as the decorator's `hint=` and appended by `_enforce_graph_type`.
  - **`onion_layers` two-axis:** `@requires_graph_type(directed=False, multigraph_ok=False)`. `_enforce_graph_type`
    checks both axes in a **defined order matching the current source: directed first (`node_metrics.py:394`), then
    multigraph (`:401`)**, so both raise paths stay reachable and message-equivalent.
- **Sweep test (the safety net):** for EVERY wrapper, assert the decorator-generated enforcement raises on exactly the
  same graph variants (across the four directed×multigraph combinations) that the old hand-written guard did, AND that
  the raised message text matches the pre-refactor message verbatim, including the components-family hints and
  `onion_layers`' two-axis behavior. This proves the condensation changed no raise behavior. (Capture the pre-refactor
  expected strings as test fixtures so the assertion is exact.)
- **Registry-agreement probe (re-expressed):** keep the existing "recorded requirement agrees with the wrapper's actual
  raise behavior across the four graph variants" probe, now reading the requirement from `_hpl_graph_type_requirement` rather than a
  separate dict, and asserting the record predicts the raise outcome for every key.
- No hand-written `if graph.is_directed(): raise` / `if graph.is_multigraph(): raise` block survives in any wrapper body
  (replace-and-sweep grep is clean). 100% coverage, warnings-as-errors hold.

### Workstream B: Up-front conflict validator + decisive error

**Status:** ✅ complete (closure reconcile 2026-06-18: header was stale; Implementation log "Workstream B complete" 2026-05-29). The `_check_graph_type_conflicts` up-front validator with entry-point-branched messages is in the shipped `graph_features/__init__.py`.
**Files:** `src/hiveplotlib/graph_features/__init__.py` (the validator, called inside `compute_graph_metrics` before any
wrapper runs), plus tests.
**Relationship to Workstream A (do not conflate — amendment A5):** A and B detect different things and produce two
distinct user-facing messages. Workstream A's decorator condenses the **single-metric** guard: one metric vs. the graph
that was actually built → "flip the graph" (or "you cannot build a multigraph for this metric"). Workstream B's up-front
validator handles the **set-conflict** case (e.g. `in_degree` + `triangles` → "these cannot share one graph; split into
two calls"), which the per-metric guard structurally cannot detect because each guard only sees its own metric against
one already-built graph. A does NOT make B redundant, and B does NOT replace A's runtime raise. Both consumers read the
same `_hpl_graph_type_requirement` record (so the count of consumers is: A's `_enforce_graph_type` guard, B's validator, and the
docs metric-table directive = three consumers, two distinct user-facing messages).
**Done when:**
- A validation pass runs at the top of `compute_graph_metrics` (after `_check_metric_names`, before any metric
  computation), reasoning over the **combined** node+edge requested set by reading each metric's `_hpl_graph_type_requirement`
  record (via `getattr(GRAPH_NODE_METRICS[m] / GRAPH_EDGE_METRICS[m], "_hpl_graph_type_requirement", <default>)`) — **not** a
  standalone requirements global (consistent with "no new global", amendment A5).
- Directedness conflict (the set contains both a directed-required and an undirected-required metric) raises one
  `ValueError` that (1) names the conflicting metrics in backticks and their opposing requirements, (2) states they
  cannot share one internal graph in a single call, (3) gives the resolution in the `_format_collision_msg` style,
  **with the resolution text branching on entry point** (see amendment A1):
  - **Direct `compute_graph_metrics` callers** (the graph is already built; there is no `graph_directed` parameter on
    this function — verified at `graph_features/__init__.py:275-287`): the resolution must NOT mention `graph_directed`.
    It must tell the user to split into two calls and **build a separate graph of the satisfying type for each call**
    (e.g. a directed graph for `in_degree`, an undirected graph such as `graph.to_undirected()` for `triangles`),
    chaining the returned nodes/edges. The resolution must say each call needs its **own** graph of the satisfying
    type, not merely a different metric list against the same graph (amendment A2).
  - **HivePlot / HivePlotMatrix callers** (the conflict surfaces through the construction-time / `compute_graph_metrics`
    method path where `graph_directed` genuinely exists): the resolution references the `graph_directed` kwarg, since at
    this level the user controls directedness through that flag.
- The branch is selected by an entry-point signal the validator receives (engineer's call on the mechanism: e.g. a
  parameter passed by the HivePlot/HivePlotMatrix call site vs. the bare-dispatcher default). The dispatcher-direct
  message is the default; the HivePlot-level message is opt-in from the construction path.
- The validator reasons over **declared** requirements independent of the resolved flag value; it may *also* note the
  resolved value to explain why neither half worked (e.g. "the internal graph resolved to directed=True").
- Cross-boundary conflicts (node metric vs. edge metric, since they share one graph) are detected (Example 5).
- A multigraph conflict (an explicit `graph_multigraph=True` plus a multigraph-rejecting metric in the set) raises a
  parallel decisive error naming the metric and pointing at `graph_multigraph=False`.
- No wasted computation: the raise happens before any wrapper is called (assert via a test that a metric whose wrapper
  would be expensive/observable is never invoked when the set is unsatisfiable).
- New tests cover: pure-node conflict, pure-edge conflict, cross-boundary conflict, multigraph conflict, and the
  no-conflict pass-through (a satisfiable set computes normally). Each `test_<scenario>` exercises the validator path.
- A test reads the **actual raised message text** (not just that a `ValueError` fires) and asserts the dispatcher-direct
  message does NOT contain `graph_directed` and DOES instruct building a separate graph per call (amendments A1, A2);
  the HivePlot-level message (exercised once the construction path reaches the validator, coordinate with Workstream
  C/D) DOES reference `graph_directed`.
- 100% coverage, warnings-as-errors hold.

### Workstream C: Graph-type inference for unambiguous requested sets

**Status:** ✅ complete (closure reconcile 2026-06-18: header was stale; Implementation log "Workstream C complete" 2026-05-29). The `_infer_graph_directed` helper and its threading through `_apply_graph_metrics` on both classes are shipped. Note: the `_graph_directed` / `_graph_directed_explicit` stash pair this workstream introduced was later collapsed into a single `_graph_directed: Optional[bool]` by Workstream G.
**Files:** `src/hiveplotlib/graph_features/__init__.py` and/or `src/hiveplotlib/hiveplot.py` /
`src/hiveplotlib/hiveplot_matrix.py` at the point where `graph_directed` is resolved before the internal graph is built.
That site is the `_apply_graph_metrics` helper on each class, which passes the resolved value through as `directed=`
(`HivePlot._apply_graph_metrics` is `hiveplot.py:2140`, building with `directed=graph_directed` at `hiveplot.py:2169`;
`HivePlotMatrix._apply_graph_metrics` is `hiveplot_matrix.py:476`, building at `hiveplot_matrix.py:508`). **The earlier
`:508` anchors were stale; confirm the exact lines against current source before editing** (amendment A4). Plus tests.
**Decision to lock with the engineer / api-critic before implementing:** where inference lives. The internal graph is
built one level *up* from `compute_graph_metrics` (in `_apply_graph_metrics`, which calls `nodes_edges_to_networkx` with
the resolved `directed=` before handing the built graph to `compute_graph_metrics`). `compute_graph_metrics` receives an
already-built `nx.Graph` and cannot retroactively change its directedness. So inference must happen at the graph-build
site (the `HivePlot` / `HivePlotMatrix` `_apply_graph_metrics` path, and any direct `nodes_edges_to_networkx` +
`compute_graph_metrics` user pipeline is the user's own responsibility). **This means inference is a HivePlot/HivePlotMatrix-
level feature, not a `compute_graph_metrics`-level one** — confirm this framing with api-critic, since it shapes which
docstrings change in Workstream D.
**Done when:**
- When `graph_directed` is **not** explicitly passed and every requested metric that declares a directedness requirement
  agrees, the inferred value is used to build the internal graph (Example 3).
- An explicit `graph_directed` always wins (Example 4); when it conflicts with a declared requirement, the Workstream B
  validator's error fires (the validator must run on the resolved set so the explicit-conflict case is caught with a
  message that names the explicit flag).
- When the requested set is itself contradictory, inference does not silently pick a side; the Workstream B validator
  raises.
- The stored `_graph_directed` attribute semantics are preserved: inference applies per-call to the *built graph*, it
  does not mutate the stored construction-time intent (consistent with the existing "explicit overrides apply to this
  call only" contract).
- New tests: inference picks undirected for an all-undirected set, picks directed for an all-directed set, leaves an
  explicit value untouched, and defers to the validator on a contradictory set.
- 100% coverage, warnings-as-errors hold.

### Workstream D: Propagation + docstrings (HivePlot, HivePlotMatrix, dispatcher)

**Status:** ✅ complete (closure reconcile 2026-06-18: header was stale; Implementation log "Workstream D complete" 2026-05-29). The conflict-raise + inference docstrings ship across the dispatcher, `HivePlot`, and the `HivePlotMatrix` mirror; `make docs` builds clean.
**Files:** `src/hiveplotlib/graph_features/__init__.py` (dispatcher docstring `:339-347` raises block),
`src/hiveplotlib/hiveplot.py` (the `compute_graph_metrics` method and `__init__` `graph_directed` / `node_graph_metrics`
param docstrings), `src/hiveplotlib/hiveplot_matrix.py` (mirror: `compute_graph_metrics` `:619-664` and the two
classmethod/init docstrings). Docstring writes are **Docs Engineer's** domain; the behavior code lands in B/C.
**Verify (do not assume):** that the new error surfaces naturally through `HivePlot.compute_graph_metrics`,
`HivePlot.__init__`, and the `HivePlotMatrix` mirror (all route through the lowest level per the grep:
`hiveplot.py:514`, `hiveplot_matrix.py:514`). `HivePlotMatrix` **does** mirror these methods (confirmed:
`hiveplot_matrix.py:606` `compute_graph_metrics`, plus init/classmethod stashing of `_graph_directed` at `:1236`,
`:1656`, `:1950`), so it needs the same docstring treatment. api-critic post-impl covers the mechanical sibling
propagation.
**Done when:**
- The dispatcher docstring's `:raises ValueError:` block documents the new up-front conflict raise (separate from the
  existing name-validation and collision raises). Current raises block is `graph_features/__init__.py:339-346`; confirm
  against source before editing.
- The dispatcher docstring states plainly that `compute_graph_metrics` does **not** infer graph type — it validates the
  requested set against the already-built graph the caller passes — and that graph-type **inference is a `HivePlot` /
  `HivePlotMatrix` construction-time convenience** (amendment A3). This closes the inference-asymmetry cliff: a user who
  learns "triangles just works" from the HivePlot path and drops down to the bare dispatcher should not be surprised it
  doesn't infer.
- `HivePlot.__init__`, `HivePlot.compute_graph_metrics`, `HivePlotMatrix.__init__` (both classmethods), and
  `HivePlotMatrix.compute_graph_metrics` docstrings document the conflict-raise behavior and (if Workstream C lands the
  inference at this level) the `graph_directed` inference, referencing the HivePlot-level kwarg names.
- Decision recorded on per-wrapper docstrings: the per-wrapper guard docstrings already document their single-axis
  requirement; they do **not** need to mention the cross-metric conflict (that is a dispatcher-level concern). State this
  explicitly so Docs Engineer does not over-edit.
- `make docs` builds clean (scan all warnings, not first-warning). 100% coverage holds (docstring-only changes do not
  affect coverage, but the build must pass).

### Workstream E: Notebook structural revision — FINAL (gate LIFTED; see amendment A7)

**Status:** ✅ complete (closure reconcile 2026-06-18: header was stale). Gate lifted 2026-05-30; the structural revision shipped (Implementation log "Workstream E complete" 2026-05-30). The editorial-critic gate is now CLOSED with a ready-to-ship verdict — see the "## Notebook review" section and the editorial RE-CLEAR recorded there. **Re-scoped from a "blurb" to a structural revision by amendment A7; the A7 done-when, the
section-by-section outline, and the verified break-case API usage examples below supersede the original blurb done-when.**
**Files:** `examples/computing_graph_metrics.ipynb` (the **only** file; edit ONLY this `examples/` copy, never the
auto-generated `docs/source/notebooks/` copy). Notebook prose is **Notebook Author's** domain.

> **GATE LIFTED (amendment A7, 2026-05-30).** The original hard gate ("Gary is actively editing the notebook; do not
> touch it until A-D sign off, no merge conflicts") is **released.** Gary has reviewed the settled changes to the
> `HivePlot` class and the graph-metric handling and is on board. Workstreams A-D and F have all shipped (see the
> Implementation log). E now runs as the closing workstream, re-scoped per A7.

**Done when (superseded by amendment A7).** See **A7** for the operative done-when, the section-by-section outline of the
revised notebook coverage, the verified break-case API usage examples, the minimal-new-viz constraint, the CHANGELOG
routing, and the dispatch recommendation. The original blurb-scoped done-when (a short markdown blurb plus a small
cell pair) no longer holds; A7 replaces it.

### Workstream F: Guard-message entry-point clarity (single-metric guard names the class-level kwarg)

**Status:** ✅ complete (closure reconcile 2026-06-18: header was stale; Implementation log "Workstream F complete" 2026-05-29 plus the F-tests entry). The three `_enforce_graph_type` message templates name both the class-level and low-level kwargs in the shipped source.
**Origin:** amendment A6 (api-critic post-impl `worth-discussing`, Gary's option-C ruling). See A6 for the full rationale and the rejected options A and B.
**Files:** `src/hiveplotlib/graph_features/networkx/_helpers.py` (the three `_enforce_graph_type` message templates), and `tests/graph_features_test.py` (the three reconstructor helpers the WS-A sweep asserts against).
**Done when:**
- The three message templates in `_enforce_graph_type` (`_helpers.py`: directed-required `:58-64`, undirected-required `:65-71`, multigraph-rejected `:72-78`) each name BOTH the class-level kwarg and the low-level `nodes_edges_to_networkx` kwarg in their resolution sentence:
  - directed-required: names `graph_directed=True` (on `HivePlot` / `HivePlotMatrix` initialization) AND `directed=True` (on `nodes_edges_to_networkx`).
  - undirected-required: names `graph_directed=False` AND `directed=False`.
  - multigraph-rejected: names `graph_multigraph=False` AND `multigraph=False`.
  Exact wording is the engineer's call, keeping the existing "point the user to the two places" style. The `hint=` tails (components-family cross-references) are appended after and stay unchanged.
- Raise BEHAVIOR is unchanged: the same metrics raise on the same directed x multigraph variants; only the resolution-text wording changes. `onion_layers`' two-axis directed-first order is preserved.
- The WS-A message-fidelity sweep's expected-string helpers in `tests/graph_features_test.py` (`_expected_directed_required_msg` / `_expected_undirected_required_msg` / `_expected_multigraph_rejected_msg`, `:1199-1220`) are updated to the new wording, so `test_requires_graph_type_enforcement_matches_requirement` (`:1280+`) and `test_requires_graph_type_topological_generations_isolated` and the explicit-hint / `onion_layers` assertions all pass against the clarified text. This is the deliberate re-opening of the WS-A verbatim contract (per A6) — the test now proves "matches the intended post-clarification text," not "identical to the pre-refactor guard text."
- No other consumer asserts or duplicates these three base strings: the WS-B validator messages (`graph_features/__init__.py:254/261`) are separate and out of scope; the per-wrapper docstrings narrate the raise abstractly and are not edited unless one literally quotes a changed template (verify).
- `make format` / `make ty` clean; `pytest tests/graph_features_test.py -m networkx -n 7` green; 100% coverage on `_helpers.py` holds; warnings-as-errors hold.
- After F's code + test land, QA re-verifies the message-fidelity sweep is green against the NEW strings and reads the changed assertions as the intended new contract (A6), not a regression.

## Plan amendments

Reconciliation pass after the api-critic planning-mode review (recorded under "API Critic's take") raised two
`must-fix` items and one `worth-discussing` item that contradicted Workstream B's done-when, plus a stale-anchor note.
All four are in-scope tweaks to existing workstreams (no new workstream, no notebook scope change, no deferral); they
sharpen done-when criteria and usage examples the engineer will implement against.

### A1 (In-scope tweak) — Error message branches on entry point (must-fix)

`compute_graph_metrics(graph, *, node_metrics=..., ...)` receives an already-built `nx.Graph` and has **no
`graph_directed` parameter** (verified at `graph_features/__init__.py:275-287`). The original Workstream B done-when
required the conflict message to reference "both `graph_directed` and the HivePlot-level `graph_directed` kwarg" as if
the dispatcher had its own flag. It doesn't. **Fix:** Workstream B done-when now requires the resolution text to branch:
the direct-dispatcher message must NOT mention `graph_directed` (it tells the user to build two graphs of the satisfying
type and run two calls); only the HivePlot / HivePlotMatrix path references `graph_directed`. Usage Examples 1 and 5
(direct-dispatcher path) updated to drop the `graph_directed` framing; Examples 3 and 4 (HivePlot path) keep it. Added a
Workstream B test requirement to read the actual raised message text and assert the branch.

### A2 (In-scope tweak) — Resolution says each call needs its OWN graph of the satisfying type (must-fix)

The error's "split into two calls (each returns fresh nodes/edges you can chain)" undersells the work: the user must
also build a directed graph for one call and an undirected graph for the other. Reusing one graph (e.g. running
`in_degree` on the undirected karate graph) reopens the whiplash this plan exists to close. **Fix:** Workstream B
done-when now requires the resolution sentence to state each call needs its own graph of the satisfying type, not merely
a different metric list against the same graph. Usage Example 2's framing comment made explicit; Example 1's error
comment names building a directed graph for `in_degree` and an undirected one for `triangles`.

### A3 (In-scope tweak) — Dispatcher docstring states inference is a HivePlot/HivePlotMatrix convenience (worth-discussing)

Inference asymmetry: HivePlot / HivePlotMatrix get inference plus the conflict raise; bare `compute_graph_metrics` gets
only the raise (its graph is pre-built, so no inference is possible). A user who learns "triangles just works" from the
HivePlot path and drops to the bare dispatcher would be surprised. **Fix:** Workstream D done-when now requires the
dispatcher docstring to state plainly that `compute_graph_metrics` does not infer graph type and that inference is a
`HivePlot` / `HivePlotMatrix` construction-time convenience.

### A4 (In-scope tweak) — Stale line anchors in Workstreams C and D

The api-critic noted Workstream C's `hiveplot.py:508` / `hiveplot_matrix.py:508` anchors are stale against current
source though the conceptual framing is right. **Fix:** Workstream C now points at the real resolved-flag/graph-build
site, the `_apply_graph_metrics` helper on each class (`HivePlot._apply_graph_metrics` at `hiveplot.py:2140`, building
with `directed=graph_directed` at `hiveplot.py:2169`; `HivePlotMatrix._apply_graph_metrics` at `hiveplot_matrix.py:476`,
building at `hiveplot_matrix.py:508`), with an explicit "confirm against current source before editing" note. Workstream
D's dispatcher raises-block anchor updated to `graph_features/__init__.py:339-346` with the same confirm note. All
anchors are guidance, not authoritative; the engineer confirms before editing.

### A5 (In-scope tweak — Workstream A reposture) — Fuller `@requires_graph_type` decorator REPLACES the guards (Gary, 2026-05-29)

Gary reviewed the design and locked a decision that flips Workstream A's posture. The original framing added a per-metric
requirements **registry as a NEW source of truth on top of** the hand-written guards, leaving the ~20 guard blocks in
place as a Holdout. Gary wants the FULLER version: a `@requires_graph_type(*, directed=None, multigraph_ok=True,
hint=None)` decorator on each metric wrapper that (1) attaches a `GraphTypeRequirement` record as `_hpl_graph_type_requirement`
(tri-state `requires_directed`, `rejects_multigraph` bool, optional `hint`), AND (2) **generates** the raise: it wraps
the body so a single shared `_enforce_graph_type(name, requirement, graph)` helper runs first and raises the
`ValueError` on violation. The hand-written `if ...: raise` blocks are **deleted** from the ~20 wrapper bodies; each body
shrinks to its happy path.

This is an in-scope tweak (no new workstream, no notebook scope change, no deferral; the notebook gate in Workstream E is
untouched and stays hard-gated last). It is a sizeable reposture of Workstream A and is recorded as such. Changes folded
into the plan in place:

- **Posture flip — guards REPLACED, not held out.** "Patterns this replaces" now lists the guard blocks as deleted-and-
  regenerated; the prior Holdout on the guards is removed and re-justified (the only remaining holdout is the docs probe).
  Updated: "Patterns this replaces" intro and replace-with bullet, the duplication-risk note (now two encodings, not
  three), the "Holdouts" section, the Prior-ADRs summary line, and the naming-audit "internal record" bullet.
- **No new global** (Gary's explicit ask). The requirement data lives on the functions, read via
  `getattr(GRAPH_NODE_METRICS[m], "_hpl_graph_type_requirement", <default>)` across the existing node+edge dicts. Any plan language
  introducing a standalone `GRAPH_METRIC_REQUIREMENTS` global is removed. Workstream B's validator reads `_hpl_graph_type_requirement`
  too.
- **Message-fidelity carve-outs preserved** (Workstream A done-when): `_enforce_graph_type` reproduces the current message
  text verbatim. Standard directed/undirected/multigraph messages keep "Build the source graph with
  `directed=...`/`multigraph=False` ..." phrasing. The components-family cross-references (`connected_components` ↔
  `strongly_connected_components` / `weakly_connected_components`) survive via the decorator's `hint=` (NOT flattened).
  `onion_layers` rejects both directed and multigraph under `@requires_graph_type(directed=False, multigraph_ok=False)`,
  and `_enforce_graph_type` checks both axes in the current source order (directed first at `node_metrics.py:394`, then
  multigraph at `:401`) so both raise paths stay reachable and message-equivalent.
- **Sweep test added** (Workstream A done-when): for EVERY wrapper, assert decorator-generated enforcement raises on
  exactly the same graph variants the old guard did and that the raised message text matches the pre-refactor message
  (including the components-family hints and `onion_layers`' two-axis behavior). Keep the existing registry-agreement
  probe, re-expressed against `_hpl_graph_type_requirement`.
- **Relationship to Workstream B clarified** (new paragraph in B): the decorator condenses the SINGLE-METRIC guard ("flip
  the graph"); B's up-front validator still handles the SET-CONFLICT case ("split into two calls"), which the per-metric
  guard structurally cannot detect. A does not make B redundant. Three consumers of `_hpl_graph_type_requirement` (A's guard, B's
  validator, docs directive), two distinct user-facing messages.
- **Blast-radius note** (Workstream A scope/risk): A now touches all ~20 wrappers (each gains a decorator, loses its guard
  body), where before it touched none of the bodies. Net fewer lines, but a wider sweep.

**User-facing error CONTRACT unchanged.** Same messages, same raises (same metrics raise on the same graph variants, same
text including hints and `onion_layers`' two raise paths); only the internal mechanism moves from ~20 inline blocks to one
decorator-driven helper. No new entry point, no new attribute reads on user input data, no signature change. Therefore
**no fresh api-critic planning pass is required for A5** (the existing planning-mode "API Critic's take" still stands; the
existing post-impl plan still applies to Workstreams B/C/D where the user-facing message branching lives).

### A6 (Added workstream — Workstream F) — Single-metric guard names the class-level kwarg, not just the low-level one (Gary, 2026-05-29, option C)

**Source.** api-critic post-impl `worth-discussing` finding (recorded above under "API Critic — post-implementation review", Task 4 wrinkle): the single-metric `_enforce_graph_type` guard a `HivePlotMatrix(..., node_graph_metrics="triangles")` user hits names only `directed=False` (the low-level `nodes_edges_to_networkx` kwarg) and steers to "the HivePlot initialization or `nodes_edges_to_networkx`". A `HivePlotMatrix.__init__` / `from_tags` user who cannot benefit from inference (the documented A5/Task-4 no-infer asymmetry) reads the traceback and goes looking for a `directed=` kwarg that does not exist on the constructor they called; the kwarg they actually need is the class-level `graph_directed=False`.

**Gary's decision: option C, executed NOW (not deferred), to close this plan out end-to-end.** Roads not taken, recorded so future readers know:

- **Option A — thread entry-point context into the deep guard (mirroring B's `_from_hive_plot`): REJECTED** as too invasive / chaotic for the payoff. The single-metric guard runs deep inside each wrapper via the decorator; plumbing an entry-point signal down to `_enforce_graph_type` to fork the message per caller is disproportionate to the gain.
- **Option B — widen `HivePlotMatrix.__init__` / `from_tags` `graph_directed: bool = True` to `Optional[bool] = None` for inference parity: HARD NO.** Those concrete `True` defaults were deliberately chosen to make sense alongside how the other constructor inputs work (no `graph=`, no sentinel); changing them is unwanted. This is a deliberate, documented asymmetry (see Task 4 ruling), not a gap to close by signature change.

**Chosen fix (option C): clarify the three message templates so each names BOTH knobs at their respective places.** Update the three message branches in `_enforce_graph_type` (`src/hiveplotlib/graph_features/networkx/_helpers.py`: directed-required at `:58-64`, undirected-required at `:65-71`, multigraph-rejected at `:72-78`). Each currently says, e.g., "Build the source graph with `directed=False` (e.g., on the HivePlot initialization or `nodes_edges_to_networkx`)." The clarified form names the class-level kwarg explicitly alongside the low-level one, e.g. "Build the source graph as undirected: pass `graph_directed=False` on `HivePlot` / `HivePlotMatrix` initialization, or `directed=False` on `nodes_edges_to_networkx`." Likewise the directed-required branch names `graph_directed=True` / `directed=True`, and the multigraph-rejected branch names `graph_multigraph=False` / `multigraph=False`. Exact wording is the engineer's call; the requirement is that each of the three names BOTH the class-level kwarg (`graph_directed` / `graph_multigraph`) and the low-level `nodes_edges_to_networkx` kwarg (`directed` / `multigraph`), keeping the existing "point the user to the two places" style the messages already use. The components-family `hint=` tails are appended after by `_enforce_graph_type` and are unaffected.

**Critical coupling — this amendment DELIBERATELY re-opens the WS-A verbatim contract.** These three strings are the verbatim-locked messages that Workstream A's message-fidelity sweep test asserts character-for-character, reconstructed in `tests/graph_features_test.py` as `_expected_directed_required_msg` / `_expected_undirected_required_msg` / `_expected_multigraph_rejected_msg` (`:1199-1220`), consumed by `_expected_message` (`:1223-1237`) and the per-wrapper sweep `test_requires_graph_type_enforcement_matches_requirement` (`:1280+`). The test's PURPOSE shifts slightly: it no longer proves "identical to the pre-refactor guard text"; it now proves "matches the intended post-clarification text." That is a deliberate change of the expected strings, **not a regression** — QA must read the changed assertions as the intended new contract, not a backslide on A5's fidelity guarantee. The A5 "User-facing error CONTRACT unchanged" line above is now superseded for these three single-metric messages by this amendment (raise behavior / which metrics raise on which variants is unchanged; only the resolution-text wording changes to name the class-level kwarg).

**Consumer audit (confirmed, so the engineer/test-engineer scope is bounded):**
- The three template strings are emitted only by `_enforce_graph_type` (`_helpers.py:58-78`) and asserted only by the three reconstructor helpers in `tests/graph_features_test.py:1199-1220`. No other consumer asserts or duplicates these exact base strings.
- **NOT in scope:** the Workstream B conflict-validator messages (`graph_features/__init__.py`, e.g. `:254`, `:261`) are separately constructed strings (they already branch on entry point and already name `graph_directed` / `to_undirected()`); this amendment does not touch them.
- The per-metric wrapper docstrings describe the raise abstractly (e.g. `node_metrics.py:481` narrates `multigraph=False` in prose); they do not reproduce the locked template strings and should not need edits. Engineer/test-engineer verify this holds and leave them alone unless a docstring literally quotes a changed template.

**Scope boundary.** Workstream F is source + test only. It does NOT touch the notebook; Workstream E stays hard-gated last for Gary's review, untouched by this amendment. F slots to run NOW, before E.

### A7 (Workstream E re-scope) — Gate LIFTED; blurb → structural revision of `computing_graph_metrics.ipynb` (Gary, 2026-05-30)

**Source.** A user ask. Gary has reviewed and is on board with the settled changes to the `HivePlot` class and the
graph-metric handling (decorator requirements registry, up-front conflict validator, `graph_directed` inference,
clarified guard messages, all shipped: see the Implementation log for A-D, F). **The Workstream E gate is lifted.**

**Triage: in-place re-scope of Workstream E (NOT a new workstream, NOT a deferral).** The notebook-documentation
workstream already exists (E) and already owns this one file; what changes is its done-when. It stays a single-file
(`examples/computing_graph_metrics.ipynb`), single-specialist-owned (Notebook Author) effort. It is **not** an in-scope
tweak: re-scoping from a blurb to a structural revision substantially changes what the notebook teaches and how its
internal-graph-type section is organized, and it pulls editorial-critic into the post-impl review. But the change is
fully contained in the *already-planned* workstream, so the right shape is to supersede E's done-when in place, not to
mint a sibling workstream E2. **No genre/class/dataset drift** (gallery notebook, `HivePlot` subject, existing datasets),
so it is not a sign-off-gated restructure; it is a structure-and-coherence revision.

**Why a blurb is the wrong container (re-review of the settled tree).** Re-running the settled tree surfaced a hard
correctness problem a blurb cannot address: **the notebook's existing "wrong graph type triggers a ValueError" demo is
now broken by the inference feature.** Cell `e6ca4494` builds `hp_internal_graph = example_hive_plot(num_edges=500)` (an
**unpinned** nodes/edges build, so `_graph_directed_explicit=False`) and then shows
`hp_internal_graph.compute_graph_metrics(node_graph_metrics="triangles")` raising a `ValueError`. Verified against the
settled tree: that call **no longer raises**, because the method-path defers to the unpinned construction stash and
inference builds an undirected graph (triangles computes; the stale traceback in the committed cell output is wrong).
The cell's entire teaching premise is now false on that exact code path. A blurb bolted onto the end leaves a
self-contradicting notebook (one section says "this raises," inference says it doesn't). The internal-graph-type section
must be **restructured**, not appended to, so the revision is correctly sized as a structural revision.

**A7 refinement (Gary, 2026-05-30, after reviewing the A7 structure below).** Two refinements to the A7 outline, applied
in place to the teaching points, the section outline, and the break-case examples (the outline now covers both init paths
in parallel; the conflict examples are reframed around interpretability, not a wording contrast):

1. **Tell the whole graph-type story TWICE, once per init path, in parallel.** Run the inference behavior, the
   single-metric break, AND the conflict cases under *both* the `HivePlot(nodes=, edges=, ...)` world (default
   `graph_directed=True`) and the `HivePlot(graph=, ...)` world (mirrors the input graph's type). Their defaults diverge
   subtly, and the divergence is the whole reason to tell it twice. Redundancy across the two narratives is **explicitly
   acceptable**; do not collapse them to avoid repetition. (Re-verified per-path behavior against the settled tree on
   2026-05-30, see the revised teaching points #2-#5 and the re-grounded examples below: the genuinely divergent surface
   is *inference* and the *single-metric break trigger*; the *conflict* error text coincides across the two HivePlot
   init paths and is shown in both worlds for completeness, not because it differs.)
2. **Conflict errors: interpretable, not a wording contrast.** Do NOT structure the conflict section around contrasting
   the differently-worded `HivePlot` vs. direct-`compute_graph_metrics` messages. Showing the standoff error in one or
   both contexts is fine; the bar is simply that the raised error reads as actionable. The direct-dispatcher form is
   optional and light, not framed as a message comparison. **Keep** the cross-boundary node+edge conflict (`in_degree` +
   `bridges`) as a distinct failure mode worth showing (validator reasoning over the combined node+edge set); confirmed
   correct, exact message verified below.

**Key teaching points the revision must land (all verified against the settled working tree, 2026-05-30; per-path
re-verification 2026-05-30 for the A7 refinement).**

**The two init paths whose defaults diverge (the spine of the A7-refinement parallel narrative).** Tell each of #2-#4
below in both worlds:

- **`HivePlot(nodes=, edges=, ...)`** — defaults `graph_directed=True`, `graph_multigraph=False` (matching the
  `(from, to)` `Edges` semantics). Built without an explicit `graph_directed`, the build is **unpinned** and eligible
  for metric-driven inference (`graph_directed_explicit=False`; verified at `hiveplot.py:2002`).
- **`HivePlot(graph=, ...)`** — `graph_directed` / `graph_multigraph` **mirror the input graph's type**
  (`graph.is_directed()` / `graph.is_multigraph()`; verified at `hiveplot.py:143-149`). Passing `graph=` makes the build
  **pinned** (`graph_directed_explicit=True`, because `graph is not None`), so **inference never runs** on a `graph=`
  build. The graph's own type is the pin.

1. **Defaults, restated as the divergence above.** From `nodes`/`edges`: `graph_directed=True`, `graph_multigraph=False`,
   override with the explicit kwargs. From `graph=`: defaults match the input graph's type, override with the explicit
   kwargs. The two defaults are the reason the rest of the story is told twice.
2. **Friendly inference — present in the nodes/edges world, ABSENT in the `graph=` world.** This is the headline new
   behavior and a documented behavior change, and it is the first place the two worlds diverge.
   - **nodes/edges:** built without an explicit `graph_directed` and given an unambiguous requested set (a single
     directedness-needing metric, or a non-conflicting set), the class **infers** the satisfying directedness. So
     `HivePlot(nodes=..., edges=..., node_graph_metrics="triangles")` (no `graph_directed`) now "just works" (builds
     undirected) where it previously raised. **Documented behavior change: a request that used to raise may now succeed.**
     (Verified: NO RAISE, `triangles` computes.)
   - **`graph=`:** no inference. `triangles` "just works" only when the *input* graph is already undirected (the common
     case, e.g. `nx.karate_club_graph()` → default `graph_directed=False`). Pass a **directed** `graph=` (e.g.
     `nx.DiGraph(nx.karate_club_graph())`) and the same single-metric `triangles` request **raises** the single-metric
     guard, with no explicit flag involved (the input graph's directedness is the pin). (Verified: undirected input →
     NO RAISE; directed input → RAISE.)
   - Inference is opt-out everywhere: an explicit `graph_directed` always wins, and pinning it (or building from a
     `graph=`) blocks inference. Multigraph is never inferred in either world (no metric *requires* a multigraph;
     default `False` is always safe), so an explicit `graph_multigraph=True` (nodes/edges) or a multigraph `graph=`
     input, plus a multigraph-rejecting metric, is the only multigraph break path.
3. **Construction infers (nodes/edges only); a *pinned* plot's method does not (both worlds).** An **unpinned** nodes/edges
   construction (and a later method call that leaves `graph_directed` unset) infers; a **pinned** construction (explicit
   `graph_directed`, or *any* `graph=` build) does not, so its method call with no per-call flag reuses the pinned value
   and an opposite-need metric raises the single-metric guard. (Verified: unpinned `example_hive_plot(...)` then method
   `triangles` → infers, no raise; init with `graph_directed=True`, or a directed `graph=` input, then `triangles` →
   raises.)
4. **The break-cases (precise ways to break it), shown in BOTH worlds.** Two distinct error families:
   - **Single-metric guard** (`_enforce_graph_type`): one metric vs. a graph of the wrong type. *This is where the two
     worlds genuinely diverge in how the break is triggered, even though the raised message is byte-identical.*
     **nodes/edges:** inference defuses the naive `triangles` request, so to demo the break you must **pin**
     `graph_directed=True` (the corrected replacement for the now-broken cell `e6ca4494`). **`graph=`:** pass a
     **directed** input graph and request `triangles` — the break is intrinsic to the input, no flag needed. Both raise
     the identical post-F message naming `graph_directed=False` / `directed=False`. (Both verified.)
   - **Set-conflict validator** (`_check_graph_type_conflicts`): a requested *set* that cannot share one internal graph,
     caught up front before any metric runs. A directedness standoff (`in_degree` + `triangles`), including across the
     node/edge boundary (`in_degree` node + `bridges` edge), plus the multigraph rejection (`clustering` +
     multigraph). **Shown in both worlds for completeness (redundancy is intentional per the A7 refinement).** Verified
     finding that shapes the outline: across the two *HivePlot* init paths the conflict message is **identical** except
     for the `directed=<resolved>` token echoed in the head (e.g. `directed=True` from a directed pin, `directed=False`
     from an undirected one); the resolution sentence (`graph_directed=True for in_degree and graph_directed=False for
     triangles`) does not differ by init path. The genuine message branch is HivePlot-vs-direct-`compute_graph_metrics`
     (the dispatcher form drops `graph_directed` and says "build two graphs"), and per the A7 refinement that direct
     form is **optional and light**, not framed as a wording contrast.

#### Section-by-section outline of the revised notebook coverage

The revision is confined to the final section, **"Controlling the `HivePlot` Internal Graph Type"** (cells `b69c9a50`
onward) plus its two subsections. Everything before that section (the node/edge metric intro, the partition-variable
section, the per-metric-kwargs section, the collision section, cells `799ec9e4` through `69db8e70`) **stays as-is** and
is out of scope for this revision (Notebook Author should re-run it to refresh outputs but not restructure it).

**Sections that STAY unchanged (structure):** `## Computing Graph Metrics` intro; `## Available Node and Edge Metrics`;
`## Add Node Metrics When Initializing Hive Plot` (incl. multi-metric); `## Using a Computed Metric as a Partition
Variable`; `## Add Edge Metrics When Initializing Hive Plot` (incl. link-prediction); `## Per-Metric Keyword Arguments`
(incl. the existing `TypeError` demo, which is unaffected); `## Add Node and Edge Graph Metrics to Existing Hive Plot`;
`## Resolving Column Name Collisions`.

**Section that gets RESTRUCTURED — `## Controlling the HivePlot Internal Graph Type`:**

- **Intro (cell `b69c9a50`):** keep the framing (the class builds an internal graph; `graph_directed` / `graph_multigraph`
  control its type). **Add** one sentence introducing that, when built from `nodes`/`edges` without an explicit
  `graph_directed`, the class now *infers* directedness from the requested metrics when the request is unambiguous, and
  that there are a few precise ways to break this (which the section then walks through). Sets up the section as a
  "here is how it works, and here is how it breaks" walk.

**Parallel-narrative shape (A7 refinement).** The two init-path subsections below are the two halves of the parallel
story. Each runs the same three beats for its own world — (a) inference behavior, (b) the single-metric break, (c) a
set-conflict demo — so a reader who only builds one way still sees the whole story. The cross-cutting `### When Requested
Metrics Disagree` subsection then consolidates the conflict-validator teaching (kept once as a concept home), showing the
standoff in both worlds since the conflict cases are where the redundancy is most verbatim and the user has explicitly
accepted it. Notebook Author may either inline the per-path conflict demo in each subsection AND keep the consolidated
subsection, or keep the conflict demos only in the consolidated subsection with a one-line pointer from each init-path
subsection; either satisfies "told twice." The non-negotiable parallelism is (a) and (b): inference and the single-metric
break MUST each appear in both init-path subsections, because that is where the two worlds actually behave differently.

- **Subsection `### When Initializing from nodes and edges Parameters` (cells `a4e98996` onward) — restructure:**
  - Keep the defaults explanation (`graph_directed=True`, `graph_multigraph=False`) and the bullet list of which metrics
    need which type. **Update** the bullets so they read against the inference behavior (e.g. note `triangles` needs
    undirected and that inference now supplies it out of the box from a nodes/edges build).
  - **(a) Inference "just works" — NEW cell pair.** A fresh `HivePlot(nodes=..., edges=..., node_graph_metrics="triangles")`
    (or the `example_hive_plot` toy with `node_graph_metrics`/`sorting_variables="triangles"`) with **no** `graph_directed`,
    showing triangles computed and no error (Example E-1). Prose: this is the inference path; the class saw an
    undirected-only metric and built an undirected graph. Reuses the existing toy dataset; **no new viz needed** (a
    `.nodes.data.head()` table suffices). Then a short paragraph calling out the **behavior change**: the same request
    used to raise because the internal graph defaulted to directed; inference removes that stumble for unambiguous
    requests.
  - **(b) Single-metric break — REPLACE the broken `ValueError` demo (cells `bf9c494f` / `e6ca4494` / `75c6b213` /
    `040dc431`).** The current "running triangles on the default directed graph raises" demo is **false** post-inference
    (verified: it now infers and computes). In the nodes/edges world the break is only reachable by **pinning**
    directedness to block inference: build (or method-call) with `graph_directed=True`, then request `triangles` → the
    single-metric guard raises (Example E-2; exact message below). This is the surviving, correct version of the old
    demo. Prose must say plainly that inference is what changed: an unpinned request infers and succeeds, so to see the
    guard you pin the wrong type. **Delete** the stale `_graph_directed` / `nodes.data` inspection cells (`75c6b213`,
    `040dc431`) that exist only to narrate the now-wrong raise.
  - **(c) Set-conflict in this world.** Either inline a directedness-standoff demo built from nodes/edges with an
    explicit `graph_directed` (Example E-3a), or point to the consolidated `### When Requested Metrics Disagree`
    subsection. If inlined, keep it short (one `try/except` cell + a sentence); the consolidated subsection carries the
    fuller treatment.
  - **Keep** the override-on-init cell (`e32465a8` / `c691a9b5`): building `example_hive_plot(graph_directed=False)` and
    computing `triangles` still works and shows the explicit-override path. Re-run to refresh output.

- **Subsection `### When Initializing from graph Parameter` (cells `4ae9c0a5` onward) — extend to the full three beats:**
  - **(a) Inference is ABSENT here — make the contrast explicit.** Keep the "undirected `graph=` input →
    `graph_directed=False` default → triangles works" demo (`663a1ba4`). **Add a sentence** that this is *not* inference:
    a `graph=` input pins directedness to the input graph's type, so `triangles` works here only because Karate Club is
    already undirected. This is the construction-infers-vs-pinned teaching point #3 and the per-path divergence from the
    nodes/edges world.
  - **(b) Single-metric break in this world — intrinsic to the input graph.** The cleanest demo of the divergence is a
    **directed** input graph: `HivePlot(graph=nx.DiGraph(nx.karate_club_graph()), ...)` then request `triangles` → the
    single-metric guard raises with **no explicit flag** (Example E-2g; same exact message as E-2). This is the parallel
    of E-2: in the nodes/edges world you must pin to break it; in the `graph=` world a directed input breaks it on its
    own. The notebook's existing explicit-`graph_directed=True`-override cells (`fe49d861` / `e9c19a2f` / `60cbcdfb`) are
    **verified still correct** and may be kept as a second flavor of the same break (overriding an undirected input to
    directed), Notebook Author's call; if both are kept, frame the `nx.DiGraph` one as "the input is directed" and the
    override one as "we forced an undirected input to directed." Re-run either to refresh the traceback (the message text
    changed under Workstream F to name `graph_directed=False` alongside `directed=False`).
  - **(c) Set-conflict in this world.** Either inline a directedness-standoff demo from a `graph=` input (Example E-3g;
    a directed `nx.DiGraph` input requesting `["in_degree", "triangles"]` raises the standoff), or point to the
    consolidated subsection. Same brevity guidance as the nodes/edges world.

- **`### When Requested Metrics Disagree` subsection (set-conflict, the consolidated concept home) — reframed per the A7
  refinement (interpretable, not a wording contrast):**
  This is the heart of the re-scope. The single-metric guard demos above cover "one metric, wrong graph." This subsection
  covers "a *set* that cannot share one graph," which the validator catches up front. **Frame it as: the error is
  decisive and actionable** (it names both metrics, says they cannot share one internal graph, and gives a concrete
  two-call fix). Do **not** frame it as a contrast between the HivePlot and dispatcher wordings.
  - **NEW cell — directedness standoff via HivePlot, shown in both worlds (redundancy intended).** Request
    `node_graph_metrics=["in_degree", "triangles"]` on a HivePlot built from nodes/edges with an explicit
    `graph_directed` (Example E-3a) AND on a HivePlot built from a directed `graph=` input (Example E-3g), each caught in
    `try/except` → `traceback.print_exc()`. Prose: `in_degree` needs directed, `triangles` needs undirected; no single
    graph satisfies both, so the class refuses up front instead of computing one and failing on the other. Note for the
    reader (or just in the plan, Notebook Author's call) that the two messages coincide except for the resolved
    `directed=` value echoed in the head; this is the redundancy the parallel narrative accepts, not a wording delta to
    dwell on.
  - **NEW cell — cross-boundary node + edge conflict (KEEP — distinct failure mode).** `node_graph_metrics="in_degree"`
    + `edge_graph_metrics="bridges"` on a HivePlot, caught in `try/except` (Example E-6). Prose: this is the same kind of
    standoff, but it spans the node/edge boundary — `in_degree` (node, directed) and `bridges` (edge, undirected) both
    run against the *one* internal graph, and the validator reasons over the combined node+edge set to catch it. This
    demonstrates the validator's combined-set reasoning and is worth its own cell (Gary confirmed; exact message below).
  - **NEW prose + optional cell — the two-call resolution.** Explain the fix the error points to: split into two calls,
    one per `graph_directed` value, chaining the augmented nodes. **No viz needed** (tables/tracebacks).
  - **NEW cell — multigraph rejection.** `node_graph_metrics=["degree", "clustering"]` with `graph_multigraph=True`
    (nodes/edges) or a multigraph `graph=` input, caught in `try/except` → the multigraph-rejection error pointing at
    `graph_multigraph=False` (Example E-4). Prose: some metrics (`clustering`, `eigenvector_centrality`, `core_number`)
    reject multigraphs; requesting one against a multigraph internal graph fails fast.
  - **OPTIONAL cell — direct `compute_graph_metrics` form (light, NOT a contrast).** A single
    `compute_graph_metrics(G, node_metrics=["in_degree", "triangles"], ...)` call caught in `try/except` (Example E-5),
    showing the dispatcher-direct message (no `graph_directed`; "build two graphs, run two calls"). Per the A7
    refinement this is **optional and light** — include it only as "the same conflict surfaces when you call the
    dispatcher directly, with a resolution phrased for a pre-built graph," NOT as a side-by-side wording comparison with
    the HivePlot message. Notebook Author's call whether to include at all.

- **Closing cross-references (cell `3d907517`):** keep the existing pointers to the NetworkX-from/to pages.

**Net new figures: zero.** Every new cell is a `.nodes.data` / `.edges.data` table or a `try/except` +
`traceback.print_exc()` error demo, matching the section's existing style. The minimal-new-viz constraint is satisfied by
construction. (If Notebook Author judges one small plot helps the inference "just works" cell, that is a single figure
and pulls viz-critic into the post-impl review per the dispatch recommendation; the default expectation is no new viz.)

#### Break-case API usage examples (verified runnable, exact messages captured 2026-05-30)

All error demos must run end-to-end under `make test-nb` (warnings-as-errors), so each wraps the failing call in
`try/except` + `traceback.print_exc()`, exactly like the notebook's existing internal-graph-type error cells
(`e6ca4494`, `e9c19a2f`, `706ec82b`). These are authorized, verified examples; Notebook Author must not invent an
unauthorized data-construction convention. Datasets: Karate Club (`nx.karate_club_graph()`) for `graph=` / direct-
dispatcher cases; `example_hive_plot(...)` for nodes/edges cases. Both are already imported in the notebook.

**Example E-1 — friendly inference "just works" (the headline; no error).** Built from `nodes`/`edges`, no
`graph_directed`, single undirected-only metric. (Verified: triangles computes, no raise.)

```python
import networkx as nx
from hiveplotlib import HivePlot
from hiveplotlib.converters import networkx_to_nodes_edges

G = nx.karate_club_graph()
nodes, edges = networkx_to_nodes_edges(G)

# no graph_directed passed; the class infers undirected because `triangles` is undirected-only
hp = HivePlot(
    nodes=nodes,
    edges=edges,
    partition_variable="club",
    sorting_variables="triangles",
    node_graph_metrics="triangles",
)
hp.nodes.data.head()  # triangles column present, no error
```

**Example E-2 — single-metric guard, NODES/EDGES world: pin `graph_directed=True` + `triangles` (the corrected break
demo).** In the nodes/edges world inference defuses the naive `triangles` request (E-1), so the break is reachable only
by **pinning** directedness, which blocks inference. This is the correct replacement for the now-broken cell `e6ca4494`.
(Verified raise.)

```python
import traceback

import networkx as nx
from hiveplotlib import HivePlot
from hiveplotlib.converters import networkx_to_nodes_edges

G = nx.karate_club_graph()
nodes, edges = networkx_to_nodes_edges(G)

# explicit graph_directed=True pins the internal graph to directed (blocks inference)
try:
    HivePlot(
        nodes=nodes,
        edges=edges,
        partition_variable="club",
        sorting_variables="club",
        node_graph_metrics="triangles",
        graph_directed=True,  # pin directed; triangles needs undirected, so this raises
    )
except ValueError:
    traceback.print_exc()
```

Exact message (verified):

```
ValueError: `triangles` does not support directed graphs (nx.DiGraph or nx.MultiDiGraph). Build the source graph as undirected: pass `graph_directed=False` on `HivePlot` / `HivePlotMatrix` initialization, or `directed=False` on `nodes_edges_to_networkx`.
```

(`example_hive_plot(...)` is the equally valid nodes/edges dataset for this cell; the toy `HivePlot` already exposes
`.nodes` / `.edges`, so the method-call shape `hp = example_hive_plot(num_edges=500, graph_directed=True)` then
`hp.compute_graph_metrics(node_graph_metrics="triangles")` raises the identical message. Notebook Author's call which
dataset; both verified.)

**Example E-2g — single-metric guard, `graph=` world: a DIRECTED input graph + `triangles` (the parallel break).** No
pinning needed: a `graph=` build mirrors the input graph's type and never infers, so a directed input *is* the pin. This
is the `graph=`-world parallel of E-2, and it raises the byte-identical message. (Verified raise.)

```python
import traceback

import networkx as nx
from hiveplotlib import HivePlot

# a directed input graph; HivePlot pins graph_directed=True from it and does not infer
G_directed = nx.DiGraph(nx.karate_club_graph())

hp_directed = HivePlot(
    graph=G_directed,
    partition_variable="club",
    sorting_variables="degree",
    node_graph_metrics="degree",
)

# triangles needs undirected; the directed internal graph raises the single-metric guard
try:
    hp_directed.compute_graph_metrics(node_graph_metrics="triangles")
except ValueError:
    traceback.print_exc()
```

Exact message (verified, identical to E-2):

```
ValueError: `triangles` does not support directed graphs (nx.DiGraph or nx.MultiDiGraph). Build the source graph as undirected: pass `graph_directed=False` on `HivePlot` / `HivePlotMatrix` initialization, or `directed=False` on `nodes_edges_to_networkx`.
```

(The notebook's existing cells `fe49d861` / `e9c19a2f` / `60cbcdfb` show the *other* `graph=`-world flavor: an
**undirected** input overridden to `graph_directed=True`, then `triangles` raises the same message. Both are verified
correct; keep one or both per the section outline.)

**Example E-3a — set-conflict (directedness standoff), NODES/EDGES world.** `in_degree` (directed-only) and `triangles`
(undirected-only) in one request, built from nodes/edges with an explicit `graph_directed` so the validator runs on the
resolved set. (Verified raise.)

```python
import traceback

import networkx as nx
from hiveplotlib import HivePlot
from hiveplotlib.converters import networkx_to_nodes_edges

G = nx.karate_club_graph()
nodes, edges = networkx_to_nodes_edges(G)

# in_degree needs directed, triangles needs undirected: no single internal graph satisfies both
try:
    HivePlot(
        nodes=nodes,
        edges=edges,
        partition_variable="club",
        sorting_variables="club",
        node_graph_metrics=["in_degree", "triangles"],
        graph_directed=True,  # explicit so the validator runs on the resolved set
    )
except ValueError:
    traceback.print_exc()
```

Exact message (verified, HivePlot branch references `graph_directed`; head shows `directed=True` from the pin):

```
ValueError: Conflicting graph-type requirements: `in_degree` requires a directed graph, but `triangles` requires an undirected graph. These cannot share one internal graph in a single call (the internal graph resolved to `directed=True`, which satisfies at most one side). Split this into two calls: set `graph_directed=True` for `in_degree` and `graph_directed=False` for `triangles`, then chain the returned nodes/edges.
```

**Example E-3g — set-conflict (directedness standoff), `graph=` world.** Same standoff from a directed `graph=` input
(no explicit flag; the input pins directed). (Verified raise.)

```python
import traceback

import networkx as nx
from hiveplotlib import HivePlot

G_directed = nx.DiGraph(nx.karate_club_graph())

# same standoff, reached through a directed graph= input rather than an explicit flag
try:
    HivePlot(
        graph=G_directed,
        partition_variable="club",
        sorting_variables="degree",
        node_graph_metrics=["in_degree", "triangles"],
    )
except ValueError:
    traceback.print_exc()
```

Exact message (verified, identical to E-3a — the resolution sentence does not differ by init path; the head's
`directed=True` here is the resolved value from the directed input):

```
ValueError: Conflicting graph-type requirements: `in_degree` requires a directed graph, but `triangles` requires an undirected graph. These cannot share one internal graph in a single call (the internal graph resolved to `directed=True`, which satisfies at most one side). Split this into two calls: set `graph_directed=True` for `in_degree` and `graph_directed=False` for `triangles`, then chain the returned nodes/edges.
```

(An **undirected** `graph=` input requesting the same set raises the identical resolution with `directed=False` in the
head instead; this is the only token that moves across HivePlot init paths, confirming the conflict text is path-invariant
apart from the echoed resolved value. Verified.)

**Example E-4 — set-conflict (multigraph rejection), shown in both worlds.** `clustering` rejects multigraphs. In the
nodes/edges world `graph_multigraph=True` builds a multigraph internal graph; in the `graph=` world a `MultiGraph` input
pins it. Both raise the identical message. (Both verified.)

```python
import traceback

import networkx as nx
from hiveplotlib import HivePlot
from hiveplotlib.converters import networkx_to_nodes_edges

# nodes/edges world: graph_multigraph=True builds a multigraph internal graph
G = nx.karate_club_graph()
nodes, edges = networkx_to_nodes_edges(G)

try:
    HivePlot(
        nodes=nodes,
        edges=edges,
        partition_variable="club",
        sorting_variables="club",
        node_graph_metrics=["degree", "clustering"],
        graph_multigraph=True,
    )
except ValueError:
    traceback.print_exc()

# graph= world: a MultiGraph input pins the internal graph as a multigraph (identical raise)
try:
    HivePlot(
        graph=nx.MultiGraph(nx.karate_club_graph()),
        partition_variable="club",
        sorting_variables="degree",
        node_graph_metrics=["degree", "clustering"],
    )
except ValueError:
    traceback.print_exc()
```

Exact message (verified, identical for both worlds; references `graph_multigraph=False`):

```
ValueError: Conflicting graph-type requirements: `clustering` does not support multigraphs, but the internal graph is a multigraph. Rebuild the source graph as `graph_multigraph=False` to compute this metric.
```

**Example E-6 — cross-boundary conflict (node metric + edge metric, KEEP as a distinct failure mode).** `in_degree`
(node, directed-only) and `bridges` (edge, undirected-only) share the *one* internal graph, so the validator catches the
standoff across the node/edge boundary by reasoning over the combined set. Gary confirmed this is correct and worth its
own cell. (Verified raise.)

```python
import traceback

import networkx as nx
from hiveplotlib import HivePlot
from hiveplotlib.converters import networkx_to_nodes_edges

G = nx.karate_club_graph()
nodes, edges = networkx_to_nodes_edges(G)

# in_degree (node) needs directed, bridges (edge) needs undirected: one internal graph, two needs
try:
    HivePlot(
        nodes=nodes,
        edges=edges,
        partition_variable="club",
        sorting_variables="club",
        node_graph_metrics="in_degree",
        edge_graph_metrics="bridges",
        graph_directed=True,  # explicit so the validator runs on the resolved set
    )
except ValueError:
    traceback.print_exc()
```

Exact message (verified, names the node metric and the edge metric):

```
ValueError: Conflicting graph-type requirements: `in_degree` requires a directed graph, but `bridges` requires an undirected graph. These cannot share one internal graph in a single call (the internal graph resolved to `directed=True`, which satisfies at most one side). Split this into two calls: set `graph_directed=True` for `in_degree` and `graph_directed=False` for `bridges`, then chain the returned nodes/edges.
```

(The same cross-boundary standoff reached through a directed `graph=` input, or through the dispatcher directly, is
verified and raises the corresponding branch message; the nodes/edges form above is the canonical demo.)

**Example E-5 (OPTIONAL, light — NOT a wording contrast) — direct `compute_graph_metrics` caller.** Per the A7
refinement, this is included only to show the same conflict surfaces when you call the dispatcher directly, with a
resolution phrased for a pre-built graph (no `graph_directed`; build two graphs, run two calls). Do **not** frame it as a
side-by-side comparison with the HivePlot message. Notebook Author's call whether to include at all. (Verified raise.)

```python
import traceback

import networkx as nx
from hiveplotlib.converters import networkx_to_nodes_edges
from hiveplotlib.graph_features import compute_graph_metrics

G = nx.karate_club_graph()
nodes, edges = networkx_to_nodes_edges(G)

# direct caller: there is no `graph_directed` parameter here; the message says build two graphs
try:
    compute_graph_metrics(
        G,
        node_metrics=["in_degree", "triangles"],
        target_nodes=nodes,
        target_edges=edges,
    )
except ValueError:
    traceback.print_exc()
```

Exact message (verified, dispatcher-direct branch — no `graph_directed`):

```
ValueError: Conflicting graph-type requirements: `in_degree` requires a directed graph, but `triangles` requires an undirected graph. These cannot share one internal graph in a single call (the internal graph resolved to `directed=False`, which satisfies at most one side). Split this into two calls, each against its OWN graph of the satisfying type: build a directed graph for `in_degree` and an undirected graph (such as `graph.to_undirected()`) for `triangles`, then chain the returned nodes/edges. A different metric list against the same graph is not enough; each call needs a graph of the right type.
```

#### A7 operative done-when (supersedes the original Workstream E blurb done-when)

- The `## Controlling the HivePlot Internal Graph Type` section is **restructured** per the outline above and tells the
  graph-type story for **both init paths in parallel** (A7 refinement): inference and the single-metric break each appear
  in **both** the nodes/edges subsection and the `graph=` subsection (this is the non-negotiable parallelism), with the
  set-conflict validator taught in a consolidated `### When Requested Metrics Disagree` subsection (and optionally inlined
  per path). Specifically: inference "just works" is shown for nodes/edges (Example E-1) and its absence is made explicit
  for `graph=`; the now-broken `ValueError` demo (cell `e6ca4494` and its `_graph_directed`/`nodes.data` narration cells)
  is **replaced** with the nodes/edges single-metric break that pins to block inference (Example E-2) AND the `graph=`
  single-metric break from a directed input (Example E-2g); the conflict subsection covers the directedness standoff in
  both worlds (Examples E-3a, E-3g), the cross-boundary node+edge conflict (Example E-6, KEPT), and the multigraph
  rejection in both worlds (Example E-4), with the direct-`compute_graph_metrics` form (Example E-5) optional and light.
- The **documented behavior change** is called out in prose: a request that previously raised (an unpinned nodes/edges
  build requesting an unambiguous metric like `triangles` without `graph_directed`) now succeeds via inference.
- The **construction-infers-vs-pinned** asymmetry (teaching point #3) is stated, and the per-path divergence is explicit:
  a pinned plot (explicit `graph_directed`, or *any* `graph=` input) does not infer, so in the nodes/edges world the
  single-metric break needs an explicit pin while in the `graph=` world a directed input breaks it intrinsically.
- The conflict section reads as **interpretable/actionable** (A7 refinement #2), not as a HivePlot-vs-dispatcher wording
  contrast; the cross-boundary node+edge conflict is retained as a distinct demo of the validator's combined-set
  reasoning.
- Every error demo uses `try/except` + `traceback.print_exc()` and the notebook **runs end-to-end clean under
  `make test-nb`** (warnings-as-errors). No new warning.
- **Minimal new viz:** the default is **zero new figures** (tables and tracebacks only). If Notebook Author adds one
  small figure for the inference cell, that is the only figure and triggers viz-critic in the post-impl review.
- Prose follows the writing-voice rules (no em-dashes, no AI filler, direct, length-disciplined) and the **gallery**-
  notebook skill conventions. Length discipline: the new subsection should not dwarf its sibling subsections.
- Edit **only** `examples/computing_graph_metrics.ipynb`; never the auto-generated `docs/source/notebooks/` copy.
- **CHANGELOG:** no entry. Per Gary's recorded preference (no CHANGELOG churn for refinements to a not-yet-released
  feature), and this is documentation of an already-unreleased feature debuting in the same version; the notebook
  revision adds no released-behavior change of its own. (Confirm the feature is unreleased before finalizing; the
  Implementation log treats A-F as unreleased v0.28.0 work, so no CHANGELOG entry is expected.)
- editorial-critic post-impl review is clean (structure and coherence: right notebook, no dataset/genre/class drift, the
  restructured section reads coherently, section-worth holds).

#### Cross-plan coordination (folded in so the record is coherent)

- **`examples/computing_graph_metrics.ipynb` already carries a separate cleanup pass.** The `HivePlot`-only WS-A cleanup
  from `graph-metrics-notebook-restructure.md` (done in a separate fork) plus a small Jaccard-section trim (the
  link-prediction "inspect" cell cut to a pure mechanic pointer; an interpretive inter-vs-intra reading removed, a
  phrasing caveat filed under that other plan's tutorial-series T3) are **already in the working-tree notebook**. This A7
  structural revision builds **on top of** the current working-tree state, not the pre-cleanup state. Notebook Author
  reads the current notebook (not git history) as the baseline.
- **Cross-plan overlap flagged:** `graph-metrics-notebook-restructure.md` WS-A *cleaned* this notebook (HivePlot-only
  scoping); this conflict-validation Workstream E *documents the new feature* in it. Two separate efforts touching one
  file. They do not collide (the cleanup reorganized the metric sections; this revision restructures the internal-graph-type
  section to teach the new behavior), but both plans should note the shared file so a future reader does not double-count
  or re-litigate the cleanup.
- **Re-execution refreshes tracebacks and may flip raise/no-raise on some cells.** After the source change, re-running
  the notebook refreshes the existing internal-graph-type error tracebacks (Workstream F changed the message text to name
  `graph_directed=` alongside `directed=`) and **flips cell `e6ca4494` from raising to not-raising** (inference). Notebook
  Author must re-run the full notebook and reconcile every graph-type cell against actual behavior, not against the
  committed (now partly stale) outputs.

#### A7 api-critic determination

**api-critic is N/A for Workstream E.** This workstream is **docs-only documentation of an already-shipped API surface**.
The conflict validator, the inference, the branched messages, and the clarified guard text all shipped in Workstreams
B/C/D/F and were already reviewed by api-critic in post-impl mode (see "API Critic — post-implementation review" and the
Workstream F addendum above; both verdicts AGREED/clean). Workstream E adds no new entry point, no new parameter, no new
attribute read, and no signature change; it documents existing behavior in a notebook. The error-string surface api-critic
flagged as load-bearing was already walked and signed off. No fresh api-critic pass is required for E.

#### A7 dispatch recommendation

Single specialist drafts, then critics review, then QA verifies. Order:

1. **notebook-author** — structural draft of the revised `## Controlling the HivePlot Internal Graph Type` section per
   the outline and the verified Examples E-1, E-2/E-2g, E-3a/E-3g, E-4, E-5, E-6 (both init paths told in parallel per the
   A7 refinement). Re-run the full notebook; reconcile all graph-type cells against actual behavior (esp. the `e6ca4494`
   flip and the F-refreshed tracebacks).
2. **editorial-critic** (post-impl) — Gary explicitly asked to involve it. It owns structure / scope / genre / coherence /
   section-worth: confirm the restructured section is the right shape, the new `### When Requested Metrics Disagree`
   subsection earns its place and does not dwarf its siblings, no dataset/genre/class drift, and the notebook still reads
   as a coherent gallery `HivePlot` page.
3. **viz-critic** (post-impl) **only if** Notebook Author added a new figure. The default expectation is zero new figures
   (tables + tracebacks), in which case **skip viz-critic**. If one inference figure was added, run viz-critic on it.
4. **qa-engineer** — `make test-nb` (notebook runs end-to-end clean, warnings-as-errors), writing-voice scan (no
   em-dashes, no AI filler, length discipline), confirm only the `examples/` notebook changed, and confirm no CHANGELOG
   entry was added (per the A7 done-when and Gary's recorded preference).

**api-critic: NOT in the sequence** (see the A7 api-critic determination — docs-only documentation of an already-shipped
surface).

### A9 (Added workstreams H, I, J, K, L) — Warn when the graph-metrics path silently collapses parallel edges (Gary, 2026-05-31)

**Source.** A user ask following a forget-vs-invent / safe-vs-risky-conversion discussion. A new, self-contained
feature: a runtime **warning** fired from the graph-metrics path when a metric request silently collapses parallel
`(from, to)` edges. Triaged as **added workstreams (H source warning, I HivePlotMatrix mirror-or-confirm, J docs, K
tests, L notebook demo)**, not in-scope tweaks: the feature spans code-engineer, test-engineer, docs-engineer,
notebook-author, plus api-critic (new user-facing warning surface) and editorial-critic (notebook), which is the
workstream shape. It lands in this plan because this plan owns the `graph_multigraph` default and the
`_apply_graph_metrics` seam the warning fires from.

**The behavior, grounded in current source (confirmed, not re-derived).** `nodes_edges_to_networkx`
(`converters.py:50`) builds the internal graph; its own docstring note (`converters.py:84-86`) states that with
`multigraph=False` and duplicate `(from, to)` pairs, networkx's `Graph` / `DiGraph` silently merge duplicates
(last-write-wins), dropping the non-final parallel edges' attributes. The graph-metrics path resolves
`graph_multigraph=False` by default for the nodes/edges case (`HivePlot._apply_graph_metrics` resolves
`graph_multigraph` to a concrete bool, then builds at `hiveplot.py:2270`; the matrix mirror builds in its static
`_apply_graph_metrics` at `hiveplot_matrix.py:~530`), so the collapse happens *implicitly* (the user never chose it). A
direct `nodes_edges_to_networkx` caller defaults to `multigraph=True` and only collapses on an explicit
`multigraph=False`. Inference (`_infer_graph_directed`, `hiveplot.py`) only touches directedness; multigraph is never
inferred, and the `graph=` input path is pinned to `graph.is_multigraph()`, never inferred (so the
`__init__`/`from_tags` vs `from_*` inference asymmetry that bit `graph_directed` does NOT bite here).

**THE WARNING TO BUILD (only one of the two candidates).** Fired from the **graph-metrics path** (NOT the converter)
when ALL of: metrics are requested, the resolved `graph_multigraph` is `False`, AND the edge data actually contains
duplicate `(from, to)` rows (within the relevant tag/tags) that networkx will merge. The message names how many parallel
edges are being collapsed and points at `graph_multigraph=True` (the class-level kwarg) to keep them distinct. It uses
the project warning conventions: a plain `warnings.warn(...)` matching the existing precedent (`p2cp.py:621`,
`viz/datashader.py:145`) with `stacklevel=3` per CLAUDE.md so it points at the user's call. A custom `Warning` subclass
is the engineer's call; note `src/hiveplotlib/exceptions/` currently holds only `Error` classes (no `Warning`
precedent), so a plain `UserWarning` is acceptable and likely cleaner unless api-critic wants a named class for
filterability.

**Placement: metrics path, not `nodes_edges_to_networkx`.** A converter warning would punish deliberate-collapse callers
(the converter default is `multigraph=True`; collapse there is an explicit `multigraph=False` choice). The metrics path
is the only place the collapse is a silent *default*. The seam is the `_apply_graph_metrics` body on each class, right
after `graph_multigraph` is resolved to a concrete bool and before/at the `nodes_edges_to_networkx(...,
multigraph=graph_multigraph)` build (HivePlot `hiveplot.py:2263-2278`; matrix `hiveplot_matrix.py:~531-545`). Engineer
picks the precise sub-seam (inline check vs. a small `_detect_collapsing_parallel_edges(nodes_or_edges, graph_multigraph)`
helper) and justifies it; a shared helper called from both class seams is preferred over duplicated inline logic.

**Explicitness: flag, do not over-engineer (api-critic to weigh in).** Ideally the warning is loudest when
`graph_multigraph` *defaulted* to `False` rather than being user-set, mirroring the directedness `_graph_directed`
pinned/unpinned machinery. But the directedness machinery exists because directedness is *inferred*; multigraph is
never inferred, so a parallel `_graph_multigraph_explicit` field would be new state for one warning. **The plan's
recommendation: do NOT add a `_graph_multigraph_explicit` field; warn whenever the resolved `graph_multigraph` is
`False` AND genuine duplicates exist, regardless of whether the user set it.** Rationale: a user who explicitly passed
`graph_multigraph=False` and still has duplicates is still silently losing those edges' attributes; the warning is
correct and actionable for them too (it points them at `graph_multigraph=True`). This avoids minting parallel-to-the-
directedness state for a single warning and keeps detection a pure function of the resolved value + the data. **api-critic
decides** in the H planning pass whether suppressing the warning on an explicit `graph_multigraph=False` is worth the
state; the default expectation is the simpler "warn whenever real duplicates collapse."

**Detection: cheap and precise.** Duplicate `(from, to)` detection on the edge DataFrame(s): for each tag's edge frame,
test whether `(from, to)` pairs are non-unique (e.g. `df.duplicated(subset=[from_col, to_col]).any()`), and count the
collapsed surplus (`len(df) - num_unique_pairs`, summed across tags) for the message. Fire ONLY on genuine duplicates
(no false positive on a simple graph with no repeats). For multi-tag `Edges`, the duplicate test is per the same
`(from, to)` semantics the converter merges on; the engineer confirms the exact cross-tag-vs-per-tag grouping against
how `nodes_edges_to_networkx` adds edges (it iterates per tag at `converters.py:147-160` but adds all into one graph, so
duplicates that collapse are `(from, to)` pairs repeated *across the combined edge set*, not only within one tag —
confirm and detect on the combined set).

**REJECTED warning — record, do NOT build (road not taken).** A **directedness warning** ("we are trusting your
`from`/`to` ordering as edge direction") was considered and **rejected.** Rationale, captured so it is not relitigated:
(1) it fires on the common default path (directed-by-default for nodes/edges), making it noise on the dominant case;
(2) it keys off unknowable user intent (whether the `from`/`to` ordering was *meant* as direction); (3) the new
two-world `graph_directed` docstrings (the 2026-05-30 clarity pass) already explain the directedness behavior at point
of use; and (4) the genuinely-undirected `graph=` input path already pins to `graph.is_directed()` and the conflict
validator already *errors* rather than fabricating, so there is no silent-fidelity-loss gap for directedness analogous
to the multigraph collapse. The multigraph collapse warning is worth building precisely because it covers a *real,
currently-silent, data-fidelity loss* with no existing point-of-use signal; the directedness warning covers no such gap.

**Plan-flow notes (bake in).**
- **api-critic planning pass on Workstream H before code lands** (new user-facing warning surface: message text,
  `graph_multigraph=True` steer, the explicitness decision above), and **post-impl after H** (and after I if the matrix
  mirror changes any user-facing message). The load-bearing surface is the *warning string* and the explicitness call.
- **No CHANGELOG entry** (Gary's standing rule, see MEMORY): this is behavior debuting within the unreleased v0.28.0
  graph-metrics feature, not a change to released behavior. Tell engineers explicitly.
- **ADR still deferred** to the combined v0.28.0 close-out; do NOT flag ADR promotion at this feature's close.
- **The notebook gate is intact**: the new notebook section (Workstream L) joins the already-shipped Workstream E
  internal-graph-type material in `examples/computing_graph_metrics.ipynb` and is sequenced LAST.

**CORRECTED warnings-as-errors surface (the scoping the brief sharpened).**
- **Notebooks will NOT fail on the new warning.** `pytest.ini` sets `filterwarnings=error` but with `testpaths=./tests`
  AND `--ignore=./tests/examples_test.py`; notebooks run via `nbconvert`'s `ExecutePreprocessor` (`make test-nb` =
  `jupyter nbconvert --execute`), which fails only on a cell *raising* (`CellExecutionError`), not on warnings.
  Warnings just print. Precedent: `examples/edge_kwarg_hierarchy.ipynb` deliberately displays conflict warnings and
  passes. So the notebook (Workstream L) is a teaching opportunity, not a landmine.
- **The real must-handle surface is `tests/`.** Any existing test that computes graph metrics on potentially-duplicate
  edge data under default `graph_multigraph=False` will trip `filterwarnings=error` and fail unless it wraps the call in
  `pytest.warns(...)` or uses duplicate-free data. **Workstream K opens with an explicit `tests/` SWEEP** (see its
  done-when): inventory those sites and decide per site (expect-the-warning via `pytest.warns` vs. clean data). This is
  a `tests/` discipline step, NOT a notebook-corpus risk.

**Feasibility audit (no new entry point; no new user-data attribute reads).** The warning fires from the existing
`_apply_graph_metrics` seam on both classes; no new public method or parameter is added (the existing class-level
`graph_multigraph` kwarg is the escape hatch the message points at). Detection reads the *existing* `from`/`to` columns
of the already-constructed `Edges` (`edges.from_column_name` / `edges.to_column_name`, `edges._data` per tag), which are
documented members of the data model, not new attributes. No data-shape contract changes. Feasibility passes.

**Sequencing.** **K's `tests/` sweep is an early precondition** (run the sweep before/alongside H so the warning lands
into a test suite that already expects it). Then: **H (source warning, HivePlot)** → **I (HivePlotMatrix
mirror-or-confirm)** → **J (docs: `graph_multigraph` docstrings on both classes)** → **K (tests, the bulk after the
sweep)** → **L (notebook demo, LAST, alongside the shipped Workstream E material)**.

#### API Critic's take — A9 / parallel-edge-collapse warning (planning mode, 2026-05-31)

Walked it as a user computing metrics on duplicate-edge data and hitting the warning. Verified `graph_multigraph` is a
real kwarg on every relevant entry point (HivePlot `__init__` `hiveplot.py:1997` and `compute_graph_metrics` `:2307`;
HPM `__init__` `hiveplot_matrix.py:333`, `from_partition` `:990`, `from_variable_sweep` `:1342`, `from_tags` `:1776`,
`compute_graph_metrics` `:650`). Compared the message voice against the existing kwarg-conflict warning
(`p2cp.py:621`). **The feature, the placement, and the explicitness recommendation are all the right calls. Two
must-fixes on the message text (the two facts that make it actionable), one should-fix on the steer's entry-point
fidelity, the rest agreed.**

**1. (must-fix) The message must name the attribute loss, not just the multiplicity.** The plan's framing in places
reduces to "N parallel edges are being collapsed," which a user can read as a benign count-only effect ("fine, I only
wanted one edge between those nodes anyway"). The load-bearing, non-obvious fact is that networkx merges last-write-wins
and **drops the non-surviving duplicates' edge attributes** (weights, types, any per-edge metric inputs). That is silent
*data* loss, not just multiplicity collapse, and it is the entire reason this warning exists over staying quiet. The
message must say so explicitly. Proposed text (fires from the class seam, so the user is always on a HivePlot /
HivePlotMatrix call): "`N` duplicate `(from, to)` edge(s) will be merged into single edges for metric computation
because `graph_multigraph=False`. networkx keeps only the last duplicate's edge attributes (weights, types, etc.); the
others are dropped before the metric runs. Pass `graph_multigraph=True` to keep parallel edges distinct." This names the
count, names *what* is lost (attributes, last-write-wins), and names the real escape hatch.

**2. (must-fix) The steer must name `graph_multigraph=True` (the class-level kwarg), not the low-level
`multigraph=True`.** A9's detection seam fires from `_apply_graph_metrics` on the class, so the user is *always* on a
`HivePlot` / `HivePlotMatrix` call here (the bare `nodes_edges_to_networkx` caller who passes `multigraph=False` made an
explicit choice and is correctly NOT warned, per the placement decision). So unlike the WS-F directedness guard, there
is no dispatcher-direct branch to worry about: every user who hits THIS warning has `graph_multigraph` available on the
exact call they made. The message names `graph_multigraph=True` and stops there. It should NOT also offer
`multigraph=True` on `nodes_edges_to_networkx` as a co-equal remedy: that kwarg lives on a function this user did not
call, and offering it reopens the precise "go hunt for a kwarg that isn't on your entry point" trap WS-F spent a whole
workstream closing. (The brief asks me to "note the low-level `multigraph=True` exists too." It does, and the J
docstrings are the right home for that cross-reference, but the runtime warning string is not — keep the warning's one
remedy pinned to the kwarg on the user's actual call.)

**3. (should-fix) The HPM mirror (Workstream I) must name `graph_multigraph=True`, not a bare `multigraph=True`.**
Unlike `graph_directed` (whose `Optional[bool]=None`-infers vs. `bool=True`-never asymmetry across the four HPM
constructors caused the WS-G discoverability tax), `graph_multigraph` is `bool=False` on `__init__`/`from_tags` and
`Optional[bool]=None` on the `from_*` classmethods, but **it is a real kwarg with the same name on all of them**, and the
warning fires uniformly (no inference, so no path-dependent asymmetry in WHEN it fires). So the `graph_multigraph=True`
steer is correct verbatim on every HPM entry point — no per-entry-point branching of the string is needed (cleaner than
the WS-F directedness guard had it). The should-fix is only that Workstream I's post-impl confirm the mirrored matrix
warning names `graph_multigraph=True` so the HPM `__init__` user, who can't lean on `from_*` inference, sees the kwarg
that exists on the class they called. Same trap class as WS-F; verify it does not recur in the mirror.

**On the key explicitness decision: AGREED — do NOT add `_graph_multigraph_explicit`, warn even on explicit
`graph_multigraph=False`.** From the user's seat this is the right call, not nagging, for three reasons. (a) The warning
reports a *fact about the data* (these specific edges' attributes are being dropped right now), not a second-guess of a
*preference*. A user who set `graph_multigraph=False` chose "collapse parallel edges"; they did not necessarily know
THIS dataset has duplicates whose attributes vanish. Surfacing the concrete loss is still news. (b) The escape hatch for
a user who genuinely wants silent collapse is already first-class and zero-new-state: Python's `warnings` filters
(`warnings.filterwarnings("ignore", ...)`, or `with warnings.catch_warnings():`). That is the idiomatic way to suppress
a known-and-accepted warning, and it is strictly better than minting a `_graph_multigraph_explicit` field that would
only ever feed this one suppression. (c) The asymmetry argument the plan makes is correct and decisive: the directedness
`_explicit` machinery exists because directedness is *inferred* (the pinned/unpinned distinction is load-bearing for
inference itself); multigraph is never inferred, so there is no existing state to piggyback on and the new field would
buy exactly one warning-suppression that `warnings` filters already provide for free. Minting state parallel to the
directedness machinery for a single warning is the over-engineering A9 rightly resists. Recommend the J docstring note
that the warning can be silenced with a standard `warnings` filter if the collapse is intended, so the suppression path
is discoverable without the field.

**On discoverability / consistency: AGREED.** Firing uniformly across all construction paths (because multigraph is
never inferred, there is no `from_*`-vs-`__init__` asymmetry in WHEN it fires) reads as consistent — genuinely the
simpler, cleaner cousin of the `graph_directed` situation, and the plan is right that the asymmetry that bit directedness
does not bite here. The `graph_multigraph=True` steer names a kwarg that exists on every entry point a user can hit this
warning from (verified above), so the WS-F failure mode (a remedy naming a kwarg absent from the called entry point)
does not recur — *provided* must-fix #2 and should-fix #3 hold and the string stays pinned to `graph_multigraph=True`
rather than drifting to the low-level `multigraph=True`.

**On the named-`Warning`-subclass-vs-`UserWarning` question (engineer's call per A9): low-confidence lean toward plain
`UserWarning`.** The project has no `Warning` subclass precedent (`exceptions/` holds only `Error` classes), and the
existing kwarg-conflict warnings (`p2cp.py:621`, `viz/datashader.py:145`) are plain `warnings.warn`. A named subclass
buys per-category filterability (a user could ignore *just* the collapse warning while keeping others), a mild plus
given recommendation (b) leans on `warnings` filters as the suppression path — filtering by category is cleaner than by
message regex. But it is a new public symbol to name, document, and export, and the filter-by-message path works without
it. Not worth blocking on; if the engineer wants the cleaner filter story, a single
`ParallelEdgeCollapseWarning(UserWarning)` is reasonable, otherwise plain `UserWarning` matches precedent. Flagging only
so the post-impl pass records which was chosen and (if a subclass) that it is exported and documented.

#### API Critic — post-implementation review (A9) / parallel-edge-collapse warning (post-impl mode, 2026-05-31)

Read the actual shipped helper (`hiveplot.py:232-273`), both call sites (`hiveplot.py:2314`, `hiveplot_matrix.py:539`),
and traced the call-stack depth from every public entry point to `warnings.warn`. Walked it as a user computing metrics
on duplicate-edge data.

```
Status: propose
API surface reviewed: _warn_if_parallel_edges_collapse (shipped message string + stacklevel),
  HivePlot.__init__ / HivePlot.compute_graph_metrics (warning paths),
  HivePlotMatrix.__init__ / .compute_graph_metrics / .from_partition / .from_variable_sweep / .from_tags (warning paths)
Concerns:
  - [must-fix] stacklevel=3 points at an internal frame on the HPM __init__ and HPM compute_graph_metrics-method
    paths (the seam depth differs from every other path) — at hiveplot.py:272 (the stacklevel arg) / hiveplot_matrix.py:613
    Suggested change: make the warning frame target adapt to the call depth (e.g. pass an explicit `stacklevel` through
    `_warn_if_parallel_edges_collapse` so the two HPM-init-routed paths use stacklevel=4 while the four direct-seam paths
    stay at 3), or hoist the warning out of `_apply_graph_metrics` to a single fixed-depth seam. See trace below.
  - [agreed] message text honors must-fix #1, must-fix #2, should-fix #3 verbatim (details below).
Test-method-coverage audit: out of scope this pass (no public method renamed/added; helper is private). The named
  warning behavior should have a `test_*parallel*collapse*` test on both classes — flagged to qa for the name-contract
  audit, not re-run here.
```

**must-fix #1 (name the attribute loss) — SATISFIED, verbatim.** The shipped string (`hiveplot.py:268-271`) is the exact
proposed text: "...networkx keeps only the last duplicate's edge attributes (weights, types, etc.); the others are
dropped before the metric runs." Names the count, names *what* is lost (attributes, last-write-wins), names the escape
hatch. Singular/plural (`edge`/`edges`) resolved at runtime via `edge_word` (`:266`). This is the load-bearing fact and
it shipped intact. Agreed.

**must-fix #2 + should-fix #3 (steer to class-level `graph_multigraph=True`, never low-level `multigraph=True`; same on
HPM) — SATISFIED.** The string's one remedy is "Pass `graph_multigraph=True` to keep parallel edges distinct"
(`:270-271`). It never mentions `multigraph=True` on `nodes_edges_to_networkx`. The helper is shared, so the HPM path
emits the identical string. `graph_multigraph` is confirmed a real kwarg on every entry point that can reach the
warning: HivePlot `__init__` and `compute_graph_metrics` (both route through `_apply_graph_metrics` at `hiveplot.py:2119`
/ `:2419`), and HPM `__init__`, `compute_graph_metrics`, `from_partition`, `from_variable_sweep`, `from_tags` (all route
through the static `_apply_graph_metrics` at `hiveplot_matrix.py:497`). The WS-F failure mode (remedy naming a kwarg
absent from the called entry point) does not recur. Agreed.

**Stacklevel-depth trace (the must-fix). The seam depth is NOT uniform across the six entry points.** `warnings.warn`
sits in `_warn_if_parallel_edges_collapse` (frame 1). With `stacklevel=3`, the warning is attributed to frame 3 (two
levels above the `warn` call). Tracing each path:

- HivePlot `__init__` → `_apply_graph_metrics` (`:2119`) → helper. Frames: 1 helper, 2 `_apply_graph_metrics`, 3 =
  `__init__` body at `hiveplot.py:2119`. **stacklevel=3 lands on the user's `HivePlot(...)` call. Correct.**
- HivePlot `compute_graph_metrics` method → `_apply_graph_metrics` (`:2419`) → helper. Frame 3 = the method body at
  `:2419`, whose caller is the user's `hp.compute_graph_metrics(...)`. **Correct.**
- HPM `from_partition` / `from_variable_sweep` / `from_tags` → static `_apply_graph_metrics` (`:1179` / `:1534` /
  `:1953`) → helper. Frame 3 = the classmethod body, called directly by the user. **Correct (all three).**
- HPM `__init__` → `_apply_init_graph_metrics` (`:428`) → static `_apply_graph_metrics` (`:613`) → helper. Frame 3 =
  `_apply_init_graph_metrics` body at `hiveplot_matrix.py:613` — **an internal frame, not the user's `HivePlotMatrix(...)`
  call, which is frame 4.** Wrong by one.
- HPM `compute_graph_metrics` method → `_apply_init_graph_metrics` (`:725`) → static `_apply_graph_metrics` (`:613`) →
  helper. Same extra seam: frame 3 = `_apply_init_graph_metrics` at `:613`. **Wrong by one.**

The asymmetry is real and exactly the "seam depth differs" risk the brief asked me to trace: the HPM `from_*`
classmethods reach the static helper in one hop, but the HPM `__init__` and HPM `compute_graph_metrics`-method paths go
through the extra `_apply_init_graph_metrics` dedup-loop frame, so they need stacklevel=4 to point at the user. As
shipped, a user who writes `HivePlotMatrix(nodes, edges, node_graph_metrics="degree")` on duplicate-edge data sees the
warning attributed to `hiveplot_matrix.py:613` (a `self._apply_graph_metrics(...)` line inside the library), not their
own call. The message body is still correct and actionable; only the source location is misleading, which is precisely
what `stacklevel` exists to get right. The cheapest honest fix is an explicit `stacklevel` argument on the helper:
the four direct-seam callers pass 3, the two init-routed callers pass 4. (This was masked in the planning take, which
assumed a single uniform "class seam"; the HPM init path's `_apply_init_graph_metrics` dedup layer is the extra frame
that breaks the assumption.)

**Fires-once-per-computation (the noisy-warning check) — reads correctly, with one nuance.** On the HivePlot side the
helper is hit exactly once per `compute_graph_metrics` / construction. On the HPM side the warning lives inside the
per-dedup-group loop (`hiveplot_matrix.py:610-625`), so it fires once per *distinct underlying `(nodes, edges)` data
pair*, not once per cell. For the common matrix (every cell backed by one shared source, the typical
`from_partition` / `from_variable_sweep` usage) that is exactly once — no N-cell spam, the user's real fear. A matrix
genuinely backed by K distinct edge tables warns up to K times, but each warning concerns a different edge set with
different dropped attributes, so that is information, not noise. The "once per metric computation (not per cell)" claim
holds in spirit; the precise statement is "once per distinct source-data group." No concern.

**On the `UserWarning` vs. named-subclass question (flagged low-confidence in planning):** shipped as plain
`UserWarning`, matching project precedent (`p2cp.py`, `viz/datashader.py`). Agreed; no new public symbol to name/export
is the right call for a single warning, and `warnings` filters cover the suppression story.

### A10 (Added workstreams M, N, O) — Refine the parallel-edge-collapse warning: same-direction-only semantics, hybrid detection, and an opt-out kwarg (Gary, 2026-06-01)

**Source.** A Gary-approved design refinement of the just-shipped warning (Workstreams H/I/J/K + the 2026-05-31
stacklevel and build-diff perf log entries). Three COUPLED changes fold in as one coherent amendment ahead of his next
task (graph scaling support), where the opt-out matters. Triaged as **added workstreams (M code, N docs, O tests)**, not
in-scope tweaks: change 3 adds a NEW user-facing boolean kwarg threaded through ~7 metric entry points, which by itself
forces a naming audit, an api-critic planning pass on the new surface, and a co-touch across code-engineer +
docs-engineer + test-engineer (the workstream shape). The three changes share one helper rewrite and one test-reversal,
so they ship together rather than as three micro-amendments.

**Grounded in current shipped source (confirmed, not re-derived).** The helper is
`_warn_if_parallel_edges_collapse(edges, graph, graph_multigraph, stacklevel=4)` at `hiveplot.py:232-280`. After the
2026-05-31 perf pass it counts collapses as a build-diff: `num_collapsing = sum(len(df) for df in edges._data.values()) -
graph.number_of_edges()` (`hiveplot.py:268-269`). It is called from both `_apply_graph_metrics` seams right after the
`nodes_edges_to_networkx(...)` build (`hiveplot.py:2314`, `hiveplot_matrix.py:537-ish`, passing the built `graph` and
per-path `stacklevel` 4/5). The shipped tests live in `TestHivePlotParallelEdgeCollapseWarning` /
`TestHivePlotMatrixParallelEdgeCollapseWarning` (log entry 2062), plus the 14 inference/directedness tests the 2026-05-31
build-diff pass wrapped in `pytest.warns(...)` (log entry 2064 enumerates all 14).

#### Change 1 — same-direction-only semantics (correctness)

The build-diff over-counts. In an UNDIRECTED simple graph, reciprocal rows `(a, b)` and `(b, a)` merge into one edge,
and the build-diff (`input_rows - graph.number_of_edges()`) counts that merge as a collapse. Gary decided we should NOT
warn on reciprocals, only on genuine same-direction duplicate rows (`(a, b)` listed 2+ times).

**Rationale to record (this is a deliberate semantics narrowing, not a bug-bug).** Reciprocal-merge only happens when
you go undirected, and choosing or accepting undirected is itself the statement "I treat this pair symmetrically," so it
is the definitional meaning of the undirected metric the user asked for, not a hidden surprise. It belongs to the
DIRECTEDNESS axis, which this plan deliberately chose NOT to warn on (the REJECTED directedness warning recorded in A9).
Same-direction duplicates are the genuine MULTIGRAPH data loss: multiplicity the user may not realize is being dropped,
distorting `degree` / `out_degree` / etc. So: warn on same-direction duplicates, NOT reciprocals. (This reverses the
"intended correctness improvement" claim in the 2026-05-31 perf log entry, which counted reciprocal merges as genuine
collapse. That claim is superseded here.)

#### Change 2 — hybrid detection (keep the fast path)

Compute the same-direction-duplicate count as `rows - (number of distinct ORDERED (from, to) pairs)`. Get the
distinct-ordered-pairs count cheaply by branching on the BUILT graph's directedness:

- **Directed build (`graph.is_directed()`):** `graph.number_of_edges()` already equals the count of distinct ordered
  pairs (a `DiGraph` keeps `(a, b)` and `(b, a)` distinct), so the existing build-diff is EXACT and free, reuses the
  build, no scan. **Keep it unchanged on this branch.**
- **Undirected build:** the merged graph cannot distinguish reciprocals, so `graph.number_of_edges()` undercounts the
  distinct ordered pairs. Count distinct ordered `(from, to)` pairs with a lean vectorized pass over the edge columns.
  NOT a `pd.concat` full copy; minimize the copy, handle the common single-tag case directly, and remember duplicates
  must be counted ACROSS ALL TAGS COMBINED (the converter merges every tag into one graph, `converters.py:147-160`). This
  branch is the only one that scans.

Net: directed builds (the nodes/edges default) pay ~O(V) and reuse the build; only undirected builds pay an O(E)
vectorized scan. Both branches yield same-direction-only semantics. The directed/undirected of the BUILT graph is the
right signal because that is exactly what determines whether `number_of_edges()` already equals distinct-ordered-pairs.

#### Change 3 — opt-out kwarg (for scaling)

Add a boolean opt-out so a user computing metrics on a very large graph can skip the collapse detection ENTIRELY (not
just silence the message; a `warnings` filter mutes output but still runs the detection, which is the wrong tradeoff at
scale). When the flag is `False`, short-circuit `_warn_if_parallel_edges_collapse` immediately: no scan, no build-diff,
no count, on BOTH branches.

- **Default `True`** (warn by default; the safety net for the normal user). The scaling power user opts out explicitly.
  Justification grounded in workflow: the normal user computing metrics on a modest graph wants the silent-data-loss
  safety net; only the scaling power user, who has accepted the collapse and is paying per-call cost on a huge graph,
  needs to turn detection off, and they do so explicitly. This mirrors the existing `warn_on_no_edges=True` /
  `warn_on_overlapping_kwargs=True` defaults (warn by default, opt out explicitly).
- **NAMING (audit, see below).** Working name `warn_on_parallel_edge_collapse`.
- **THREADING.** Expose it on the ~7 metric entry points that already carry `graph_directed` / `graph_multigraph`:
  `HivePlot.__init__`, `HivePlot.compute_graph_metrics`; `HivePlotMatrix.__init__`, `from_partition`,
  `from_variable_sweep`, `from_tags`, `compute_graph_metrics`. **Open for api-critic:** whether it needs the
  construction-time-stash + per-call-override pattern that `graph_directed` / `graph_multigraph` use, or whether a
  simpler per-call / per-construction bool suffices. It only matters at metric-computation time and is never inferred, so
  the simpler shape is plausible; flag for api-critic to rule. (Note `warn_on_overlapping_kwargs` already stashes as
  `self.warn_on_overlapping_kwargs` at `hiveplot.py:2115` and is a plain construction-time bool with no per-call
  override; that is the lighter precedent the simpler shape would follow.)

#### Naming audit (REQUIRED — new user-facing kwarg)

Check the working name `warn_on_parallel_edge_collapse` against the library's own established vocabulary (the dominant
adjacent surface here is hiveplotlib's existing `warn_on_*` booleans, not NetworkX, since this is a hiveplotlib-level
control with no NetworkX analog).

- **Precedent (confirmed in source):** `warn_on_no_edges: bool = True` (`hiveplot.py:1344`, `:1615`; also `p2cp.py:279`,
  `toy_hive_plots.py:103`) and `warn_on_overlapping_kwargs: bool = True` (`hiveplot.py:2042`, stashed `:2115`). Both are
  `warn_on_<noun-phrase>` booleans defaulting `True`. The noun phrase is the THING warned about (`no_edges`,
  `overlapping_kwargs`).
- **Assessment of `warn_on_parallel_edge_collapse`:** fits the family. The noun phrase `parallel_edge_collapse` names the
  thing warned about (the collapse of parallel edges), consistent with `overlapping_kwargs` naming the overlap. Singular
  `edge` matches `warn_on_no_edges`? No — that one is plural. But `parallel_edge_collapse` reads as a compound noun
  (the "parallel-edge collapse" event), so singular `edge` as the adjective-noun modifier is correct English and reads
  cleaner than `parallel_edges_collapse`. **Recommendation: `warn_on_parallel_edge_collapse`** (Gary's working name) is
  the right choice; it is consistent with the `warn_on_*` siblings and reads naturally. api-critic confirms in the
  planning pass (final call on `edge` vs `edges`, and on threading shape).

#### API usage examples

```python
# Example 1 (default): warn on a genuine same-direction duplicate. The opt-out kwarg defaults True.
import networkx as nx
from hiveplotlib import HivePlot
from hiveplotlib.node import NodeCollection
from hiveplotlib.edges import Edges

nodes = NodeCollection(data={"unique_id": [0, 1, 2], "x": [0.0, 1.0, 2.0]}, unique_id_column="unique_id")
# (0, 1) listed twice in the SAME direction -> genuine multigraph collapse, attributes of the
# non-final duplicate are dropped before the metric runs.
edges = Edges(edges=[[0, 1], [0, 1], [1, 2]], from_column_name="from", to_column_name="to")

# Call site (default warn_on_parallel_edge_collapse=True): a UserWarning fires naming 1 collapsing edge.
hp = HivePlot(
    nodes=nodes,
    edges=edges,
    partition_variable="x",
    sorting_variables="degree",
    node_graph_metrics="degree",
    # warn_on_parallel_edge_collapse=True  # the default; warns on the genuine duplicate
)
```

```python
# Example 2 (NEW semantics): a reciprocal pair on an UNDIRECTED build does NOT warn.
# (a, b) and (b, a) merge into one undirected edge; that is the definitional meaning of the
# undirected metric, not a hidden surprise, so no warning fires (change 1).
import networkx as nx
from hiveplotlib import HivePlot
from hiveplotlib.node import NodeCollection
from hiveplotlib.edges import Edges

nodes = NodeCollection(data={"unique_id": [0, 1], "x": [0.0, 1.0]}, unique_id_column="unique_id")
edges = Edges(edges=[[0, 1], [1, 0]], from_column_name="from", to_column_name="to")

# Call site: `triangles` is undirected-only, so the internal graph builds undirected and merges
# (0, 1) / (1, 0) into one edge. NO warning fires (reciprocal, not a same-direction duplicate).
hp = HivePlot(
    nodes=nodes,
    edges=edges,
    partition_variable="x",
    sorting_variables="x",
    node_graph_metrics="triangles",
    # no graph_directed: inferred False (undirected); reciprocal merge is silent by design
)
```

```python
# Example 3 (opt-out for scaling): skip detection ENTIRELY on a large graph.
# warn_on_parallel_edge_collapse=False short-circuits the helper immediately: no scan, no build-diff,
# no count. This is the scaling escape hatch (a `warnings` filter would still run the detection).
import networkx as nx
from hiveplotlib import HivePlot

G = nx.gnm_random_graph(100_000, 500_000)  # large; the user has accepted any collapse

hp = HivePlot(
    graph=G,
    partition_variable="some_attr",
    sorting_variables="degree",
    node_graph_metrics="degree",
    warn_on_parallel_edge_collapse=False,  # skip collapse detection; pay no per-call cost
)
```

```python
# Example 4 (HivePlotMatrix mirror): the kwarg threads through the matrix entry points identically.
import networkx as nx
from hiveplotlib import HivePlotMatrix

G = nx.karate_club_graph()
hpm = HivePlotMatrix.from_partition(
    graph=G,
    partition_variable="club",
    node_graph_metrics="degree",
    warn_on_parallel_edge_collapse=False,  # available on from_partition / from_variable_sweep / from_tags / __init__ / compute_graph_metrics
)
```

#### API Critic's take (planning mode) — A10 (Workstream M, the `warn_on_parallel_edge_collapse` opt-out kwarg)

Reviewed the new surface against the shipped source (the `graph_directed`/`graph_multigraph` stash-and-override pattern
at `hiveplot.py:2125-2126` / `:2426-2429`, and the plain-bool `warn_on_overlapping_kwargs` stash at `:2115`). Ruling
the four open questions:

**1. Name — `agreed` on `warn_on_parallel_edge_collapse`, `True` default.**
The naming audit's conclusion holds. It fits the `warn_on_<noun-phrase-of-the-thing>` family (`warn_on_no_edges`,
`warn_on_overlapping_kwargs`), and the noun phrase names what is warned about. On `edge` vs `edges`: singular is correct
here and the audit's reasoning is right, but for a sharper reason than "compound noun reads cleaner." The two siblings
split on a real grammatical rule, not on taste: `no_edges` is plural because it is a bare noun (the absence of edges, a
count), whereas `parallel_edge_collapse` is an attributive-noun modifier stack (`parallel-edge collapse`, the *event*),
and English attributive nouns go singular (`car park`, not `cars park`). So singular `edge` is not in tension with plural
`no_edges`; both follow the same rule applied to different grammatical roles. Keep `warn_on_parallel_edge_collapse`.
Reject `parallel_edge_merge` (the codebase's shipped vocabulary is "collapse" everywhere: the helper
`_warn_if_parallel_edges_collapse`, the message, this plan; introducing "merge" forks the term) and
`collapsed_parallel_edges` (drops the `warn_on_` prefix that signals "this is a warning toggle, not a behavior toggle"
to a tab-completing user, which is exactly the prefix that defends against the footgun in point 3). `True` default is
correct and already grounded in the matching `warn_on_*` precedent; the silent-data-loss safety net is the common case,
the scaling opt-out is the explicit-power-user case.

**2. Threading shape — `should-fix`: use the PLAIN construction-time bool, NOT the stash-and-override pattern.**
This is the decisive call and I disagree with treating it as symmetric to `graph_directed`/`graph_multigraph`. The
stash-and-override pattern exists for those two for a specific reason that does NOT apply here: their `None` is a
load-bearing *sentinel* that means "I have not pinned this, infer it / follow construction," and the per-call override
lets a user re-decide directedness on a later `compute_graph_metrics` call without rebuilding (the `triangles`-then-
`in_degree` re-inference contract recorded under A8/`hiveplot.py:2426`). `warn_on_parallel_edge_collapse` has no
inference channel and no third state: it is a plain bool with two meanings (warn / don't), exactly the shape of
`warn_on_overlapping_kwargs`, which is correctly a plain construction-time bool with no per-call channel. Walking the two
users the plan names:
  - *Opt out at construction* (the scaling user, Example 3/4): plain stash `self.warn_on_parallel_edge_collapse = ...`
    serves this directly and is the dominant case. This user builds one large graph and never wants the scan.
  - *Opt out on a single later `compute_graph_metrics` call*: this is a real but thin case. The honest question is
    whether it earns a per-call kwarg on all 7 entry points. I rule **no** for the construction-time default, **yes** for
    a per-call override ONLY on the two `compute_graph_metrics` methods — i.e. the hybrid below.

**Recommended shape (the hybrid, narrower than full stash-and-override):**
  - Add `warn_on_parallel_edge_collapse: bool = True` (plain bool, NOT `Optional`) to the 5 *construction* entry points
    (`HivePlot.__init__`, `HivePlotMatrix.__init__`, `from_partition`, `from_variable_sweep`, `from_tags`). Stash it as
    `self.warn_on_parallel_edge_collapse` exactly like `warn_on_overlapping_kwargs`.
  - On the 2 `compute_graph_metrics` methods, add `warn_on_parallel_edge_collapse: Optional[bool] = None` and resolve
    `if warn_on_parallel_edge_collapse is None: warn_on_parallel_edge_collapse = getattr(self,
    "warn_on_parallel_edge_collapse", True)` (override wins, does not mutate the stash), mirroring the existing
    `graph_directed`/`graph_multigraph` resolution two lines above it. This gives the thin per-call-opt-out case a home
    without coining a `None` sentinel on the constructors where it would be meaningless (a constructor `None` for a
    pure-warning bool has no "infer" reading to justify it).
This is consistent with BOTH precedents simultaneously: plain bool on constructors (matches `warn_on_overlapping_kwargs`,
the right sibling since this is also a pure warning toggle), `Optional` per-call override on the metric methods (matches
`graph_directed`/`graph_multigraph`, the right sibling since that is where per-call re-decision lives). It avoids the
dishonest `Optional[bool] = None` on 5 constructors where the `None` would carry no inference semantics and would invite
a reader to ask "infer from what?"

**3. Footgun — `should-fix` docstring requirement: the name controls the WARNING, not the COLLAPSE.**
The genuine risk: a scaling user reads `warn_on_parallel_edge_collapse=False` and concludes their parallel edges will be
*kept* (no collapse). They will not be; the collapse still happens (last-write-wins, attributes of duplicates dropped),
only the warning and its O(E) detection are skipped. The `warn_on_` prefix is the first-line defense (it signals a
warning toggle, which is why I rejected `collapsed_parallel_edges` in point 1), but the prefix alone does not say "the
collapse still occurs." Require the param docstring on every entry point to lead with the disambiguation, e.g.: *"Whether
to detect and warn about same-direction duplicate edges collapsing during metric computation (default ``True``). Setting
``False`` skips the detection entirely (a perf escape hatch for large graphs); it does **not** change whether edges
collapse — duplicates are still merged last-write-wins. To preserve parallel edges, use ``graph_multigraph=True``
instead."* That last sentence is load-bearing: it routes the user who actually wanted to keep their edges to the kwarg
that does it. This belongs in Workstream N's docstring brief verbatim; flag it there.

**4. Semantics framing — `agreed`. "Same-direction duplicates warn, reciprocals don't" is coherent, and Examples 1-3
demonstrate it correctly.**
From the user's seat the line is defensible: choosing/accepting an undirected build IS the statement "I treat this pair
symmetrically," so a reciprocal merge is the definitional meaning of the metric requested, not silent loss (A9's
rejected-directedness-warning reasoning, applied consistently). Same-direction duplicates are genuine multiplicity loss
that distorts `degree`. Example 1 (same-direction `[[0,1],[0,1],[1,2]]` fires), Example 2 (reciprocal `[[0,1],[1,0]]`
on an undirected `triangles` build does not fire) demonstrate the narrowing correctly. One **worth-discussing** note on
the examples, not a blocker: Example 1 builds an undirected graph by inference (`degree` does not pin directedness, so
`_infer_graph_directed` falls back to `default=True` → directed), so it actually exercises the *directed* hybrid branch,
not the undirected one. That is fine and the comment is accurate, but the prose comment "genuine multigraph collapse"
reads as if it were testing the undirected scan path. If the planner wants the examples to visibly cover BOTH hybrid
branches (the thing Change 2 is about), add a one-line note to Example 1 that this is the directed-branch case and that
the undirected-branch same-direction case is covered in the tests (it is, per Workstream O's "DIRECTED build AND
UNDIRECTED build" bullet). Cosmetic; the semantics are right.

**Recurring pattern (for the post-impl pass and the expertise file):** when a new boolean is *visually* adjacent to a
`None`-sentinel pair on a shared signature (`graph_directed`/`graph_multigraph`), the gravitational pull is to give it
the same `Optional[bool] = None` shape for symmetry. Resist when the new bool has no third state — `None` symmetry there
is dishonest (no "infer" reading) and taxes the reader. Pick the sibling whose *semantics* match (pure warning toggle →
`warn_on_overlapping_kwargs`), not the sibling that happens to sit on the same line.

#### Plan-flow notes (bake in)

- **api-critic planning pass on Workstream M BEFORE code lands** (new user-facing kwarg surface: name, default,
  threading consistency with the sibling `graph_*` kwargs and the `warn_on_*` precedent, whether it needs the stash
  pattern). **Post-impl pass after M ships** (and after any sibling-propagation review the dispatching session triggers).
  The load-bearing surface is the new kwarg + the narrowed semantics.
- **No CHANGELOG entry** (Gary's standing rule, see MEMORY): refining behavior debuting within the unreleased v0.28.0
  graph-metrics feature. Tell engineers explicitly.
- **ADR still deferred** to the combined v0.28.0 close-out; do NOT flag ADR promotion at this feature's close.
- **The notebook gate is intact / the notebooks are NOT in this amendment's scope.** Workstreams E and L stay GATED
  behind Gary's review; this amendment does not disturb them. The new semantics (no longer warning on reciprocals) does
  NOT contradict Workstream L's planned demo (which builds a SAME-DIRECTION duplicate and is unaffected); no L re-scope
  is needed. If the dispatching session finds L's demo data is reciprocal-based, route back here, but per A9/L the demo
  is a same-direction `(from, to)` duplicate, so it still fires.

#### Critical test-reversal (explicit step in Workstream O)

The 2026-05-31 build-diff perf pass (log entry 2064) WRAPPED 14 existing inference/directedness tests in
`pytest.warns(...)` BECAUSE the build-diff made undirected-reciprocal builds fire the warning. With Change 1
(same-direction-only), those reciprocal cases NO LONGER fire, so those 14 `pytest.warns` wrappers MUST be REMOVED
(reverted to plain calls / the prior `nullcontext` form), or those tests will FAIL asserting a warning that no longer
fires. This is an explicit, load-bearing step in Workstream O. The 14 sites (from log entry 2064):
- `tests/hiveplot_test.py`: `test_hiveplot_init_infers_undirected_for_undirected_only_metric`,
  `..._init_explicit_directed_false_honored`, `..._init_graph_explicit_directed_wins_over_inference`,
  `..._compute_graph_metrics_method_omitted_flag_infers` (first call only),
  `..._method_per_call_false_honored`, `..._method_per_call_false_honored_on_graph_build`.
- `tests/hiveplot_matrix_test.py`: `test_hpm_classmethod_inference_on_nodes_edges_path` (the conditional
  `pytest.warns`/`nullcontext`), `..._classmethod_explicit_directed_false_honored`,
  `..._init_explicit_false_needed_for_triangles_under_asymmetry`,
  `..._from_tags_explicit_false_needed_for_triangles_under_asymmetry`,
  `..._compute_graph_metrics_method_omitted_flag_infers` (first call only),
  `..._method_per_call_false_honored`, `..._method_per_call_false_honored_on_graph_build`.

The engineer confirms each site's fixture is genuinely reciprocal-only (not also same-direction-duplicated) before
reverting its wrapper; if any site mixes reciprocals AND same-direction duplicates, the warning still fires there and the
wrapper stays — halt and surface rather than blanket-reverting all 14.

NEW dedicated tests to ADD (Workstream O):
- Same-direction duplicates DO fire, on a DIRECTED build AND on an UNDIRECTED build (proving the hybrid branches both
  detect genuine duplicates).
- Reciprocals do NOT fire on an undirected build (the change-1 narrowing).
- The opt-out flag `warn_on_parallel_edge_collapse=False` skips detection: assert no warning AND, ideally, that the
  detection path is not taken (e.g. monkeypatch/spy that the build-diff/scan body is not entered, or that
  `graph.number_of_edges()` is not consulted) so the test proves the short-circuit, not just silence.
- The hybrid directed-vs-undirected branches both produce CORRECT counts (a directed build with a same-direction
  duplicate counts it; an undirected build with both a same-direction duplicate AND a reciprocal counts ONLY the
  same-direction one).

#### API Critic — post-implementation review (A10) — `warn_on_parallel_edge_collapse` + narrowed collapse semantics (2026-06-01)

Read the shipped source for all 7 entry points, the rewritten `_warn_if_parallel_edges_collapse` + `_num_distinct_ordered_edge_pairs` helpers, and the dedicated tests. Walked it against my own A10 planning ruling (name, hybrid threading shape, footgun-doc requirement, semantics framing). The implementation honored the ruling on all four axes.

**Status: clean.**
**API surface reviewed:** `warn_on_parallel_edge_collapse` on `HivePlot.__init__`, `HivePlot.compute_graph_metrics`, `HivePlotMatrix.__init__`, `HivePlotMatrix.from_partition`, `HivePlotMatrix.from_variable_sweep`, `HivePlotMatrix.from_tags`, `HivePlotMatrix.compute_graph_metrics`; the narrowed (same-direction-only) collapse-warning semantics and message.

**1. Threading matches the ruling — confirmed.** The 5 constructors carry plain `warn_on_parallel_edge_collapse: bool = True` (HivePlot `:2108`, HPM `:350` / `:527` / `:607` / `:1864`) and each stashes verbatim as `self.warn_on_parallel_edge_collapse = warn_on_parallel_edge_collapse` (HivePlot `:2183`; HPM `__init__` `:440` and classmethod `instance.` stashes `:1374` / `:1826` / `:2150`), exactly mirroring `warn_on_overlapping_kwargs`. The 2 `compute_graph_metrics` methods carry `Optional[bool] = None` (HivePlot `:2428`, HPM `:687`) and resolve `if warn_on_parallel_edge_collapse is None: warn_on_parallel_edge_collapse = getattr(self, "warn_on_parallel_edge_collapse", True)` (HivePlot `:2506-2509`, HPM `:773-776`) into a local, never writing back to `self`. The per-call-False-then-revert contract is directly proven by `test_..._method_per_call_false_leaves_stash_untouched` (`hiveplot_test.py:7345-7362`): a per-call `False` runs silent, `hp.warn_on_parallel_edge_collapse is True` still holds, and the next omitted call fires. No `None` sentinel leaked onto a constructor; the dishonest-`None` failure mode I flagged in planning did not ship.

**2. Footgun closed — confirmed on all 7.** Every `:param warn_on_parallel_edge_collapse:` leads with what the flag controls (detect-and-warn on same-direction duplicates during metric computation), states default `True`, and carries the load-bearing disambiguation verbatim: it does **not** change whether edges collapse (duplicates still merge last-write-wins), and "To preserve parallel edges instead, use ``graph_multigraph=True``." Verified at HivePlot `:2037-2040` (init) and `:2483-2488` (method, which also carries the no-stash-mutation per-call sentence); HPM `:293-296` (init), `:1190-1193` (from_partition), `:1560-1563` (from_variable_sweep), `:2015-2018` (from_tags), `:750-755` (method). The "I want to keep my edges" user is routed to `graph_multigraph=True` on every surface. Nothing misleading; the helper docstring (`:301-302`) restates the same caveat. No concern.

**3. Semantics coherent end to end — confirmed.** Helper short-circuits on `if not warn_on_parallel_edge_collapse: return` as the literal first line (`:305-306`), before the `if graph_multigraph: return` (`:308-309`) and any count/scan; the directed branch reuses `graph.number_of_edges()` and the undirected branch calls the lean `_num_distinct_ordered_edge_pairs` scan (`:312-315`), both yielding `rows - distinct_ordered_pairs`, so reciprocals no longer count. The message (`:321-327`) says "{N} duplicate `(from, to)` edge(s) will be merged into single edges ... networkx keeps only the last duplicate's edge attributes ... Pass `graph_multigraph=True` to keep parallel edges distinct." This does NOT over-claim relative to the narrower trigger: every counted edge is now a genuine same-direction `(from, to)` duplicate, so "duplicate `(from, to)`", "merged ... last-write-wins", and the `graph_multigraph=True` remedy all hold exactly. The `graph_multigraph` notes were correctly tightened to "same-direction duplicate `(from, to)` rows" and now explicitly exclude reciprocals (verified HivePlot init `:2022-2026`; per Workstream N all 7 carry the matching sentence). No place reads as if reciprocals still warn.

**4. Signature consistency — confirmed, no trip hazard.** On both `compute_graph_metrics` and all constructors, `warn_on_parallel_edge_collapse` sits immediately after the `graph_directed` / `graph_multigraph` / `graph_source_attribute_name` cluster (e.g. HivePlot init `:2105-2108`, method `:2425-2428`), reading naturally as the last graph-build control before `use_numba`. The name slots into the existing `warn_on_*` family (`warn_on_no_edges`, `warn_on_overlapping_kwargs`); the `bool`-on-constructors / `Optional`-on-methods split is the SAME split the adjacent `graph_*` kwargs already use, so a user who has internalized that pattern reads no new rule. No ordering, naming, or default inconsistency.

**Test-method-coverage audit:** clean for the methods this workstream touched. `warn_on_parallel_edge_collapse` is exercised at construction and via `compute_graph_metrics` on both classes; the short-circuit-precedes-detection proof uses a `monkeypatch` spy on `_num_distinct_ordered_edge_pairs` (a raising stub) so the test proves the scan is never entered, not just silence (`hiveplot_test.py:7255` and Workstream O log); same-direction-fires / reciprocal-silent / mixed-count-excludes-reciprocal cover both hybrid branches; the stash-untouched revert is tested directly. Named `test_*` bodies call the named surface in each case sampled.

```
(remaining Plan amendments slots below, append-only as future emergent work surfaces.)
```

### Closure-reconcile note for the ADR (2026-06-18): the shipped decorator field is `allows_multigraph`

For the combined NetworkX ADR writer: this plan's body describes the `@requires_graph_type` decorator with a `multigraph_ok=True` kwarg (e.g. Workstream A's done-when: `@requires_graph_type(*, directed=None, multigraph_ok=True, hint=None)`) and a `GraphTypeRequirement` record field `rejects_multigraph: bool` (`= not multigraph_ok`). **That naming is stale against shipped reality.** The shipped decorator kwarg and the record field are both named **`allows_multigraph`** (default `True`), not `multigraph_ok` / `rejects_multigraph`. Verified in source: `@requires_graph_type(allows_multigraph=False)` at `src/hiveplotlib/graph_features/networkx/node_metrics.py:123` (and siblings), and `not _requirement_for(m).allows_multigraph` at `src/hiveplotlib/graph_features/__init__.py:298`. The companion field `requires_directed` is unchanged.

Behavior is equivalent: the design collapsed the planned `multigraph_ok` kwarg + derived `rejects_multigraph` field into a single `allows_multigraph` field carried straight through (`rejects_multigraph == not allows_multigraph`), so every raise/no-raise outcome the plan body describes holds. Only the spelling of the kwarg/field changed. The ADR should use `allows_multigraph`.

## Holdouts

- **None on the per-wrapper guards (changed by amendment A5).** Under the original registry-only framing the hand-written
  guard blocks were a Holdout. Gary's fuller-decorator decision removes that holdout: the guards are genuinely **replaced**
  by the `@requires_graph_type` decorator's centralized `_enforce_graph_type` raise (the single-metric runtime safety net
  for direct-wrapper callers and resolved-flag mismatches is preserved, just relocated from ~20 inline blocks into one
  shared helper). QA's replace-and-sweep should confirm no hand-written guard block survives; Workstream A's
  message-fidelity sweep test is the proof the relocated raise is behavior- and message-equivalent.
- **Docs metric-table directive's empirical probe is a Holdout (re-justified).** `docs/source/_ext/metric_table_directive.py`
  still discovers requirements by probing each wrapper against four graph variants rather than reading `_hpl_graph_type_requirement`.
  Kept as-is this plan (collapsing it into a reader of `_hpl_graph_type_requirement` is the flagged future pass under "Patterns this
  replaces"). It is independent and empirical, so it does not break when the decorator lands; not in scope to change here.

### A8 (Added workstream — Workstream G) — Collapse `(_graph_directed, _graph_directed_explicit)` into a single `_graph_directed: Optional[bool]` (Gary, 2026-05-31)

**Source.** A user ask. This plan introduced the `(_graph_directed, _graph_directed_explicit)` stash pair on `HivePlot`
(and mirrored it on `HivePlotMatrix`) under Workstream C to gate metric-driven inference. With A-F and E all shipped,
Gary wants the internal state cleaned up: collapse the two fields into one `Optional[bool]`. Triaged as an **added
workstream (G)**, not an in-scope tweak: the change spans code-engineer (two source files, six matrix sites plus the
HivePlot init / `_apply_graph_metrics` / `compute_graph_metrics` plumbing), test-engineer (many `_graph_directed_explicit`
assertions across both test files encode behavioral contracts and must be re-expressed, not deleted), and qa-engineer
(full-suite + coverage verification). Multiple specialists co-touch a contract surface, which is the workstream shape.
It lands here because this is the plan that introduced the pair.

**Diagnosis (validated against current source 2026-05-31).** Today `_graph_directed_explicit=False` means "unpinned,
infer per call," and in that state the companion `_graph_directed` (stashed `True` on the nodes/edges-no-override path)
is dead state: never load-bearing, only read as the ultimate fallback inside `_infer_graph_directed(..., default=...)`
at `hiveplot.py:2354-2357`. Meanwhile `_graph_multigraph` sits beside it with identical `_graph_*` surface naming but
no `_explicit` companion (multigraph is never inferred). Two sibling attrs, same surface naming, different semantics.

**Proposed shape (locked with Gary).**
- `_graph_directed: Optional[bool]`. `None` = "unpinned, infer per call"; a concrete bool = "pinned." The "default True"
  stops being object state and becomes a function-local fallback: `_apply_graph_metrics` calls
  `_infer_graph_directed(..., default=True)` when the resolved value is `None`. `_infer_graph_directed`'s own signature
  stays `default: bool` (no change to that helper).
- `_graph_directed_explicit` retired entirely.
- **`_graph_multigraph` STAYS `bool` — do NOT touch its type.** It has no inference channel; making it `Optional` would
  be symmetry for symmetry's sake and would hide a real semantic difference. Gary has explicitly opted out of forcing
  symmetry here. The honest type asymmetry (`_graph_directed: Optional[bool]` vs. `_graph_multigraph: bool`) tracks the
  real difference: only directedness has an inference channel.

**Hard constraint — per-call inference must keep working on EVERY call, not just the first.** A build-with-`triangles`
followed by `compute_graph_metrics(node_graph_metrics="in_degree")` must re-infer cleanly to directed on the second call.
So inference must NOT write back to the stash; `None` persists. This rules out any "freeze first inference into the
stash" design. The current control flow already honors this (inference is per-call inside `_apply_graph_metrics`); the
collapse preserves it for free as long as `compute_graph_metrics` passes the per-call-or-stash `Optional[bool]` straight
through rather than freezing it. The existing no-write-back tests (`hiveplot_test.py:6788-6807`,
`hiveplot_matrix_test.py:3867-3896`) are the regression net and must be re-expressed, not dropped.

**Two decisions recorded (Gary-endorsed).**
1. **Defensive `getattr` fallback flips `True` → `None`.** The defensive reads `getattr(self, "_graph_directed", True)`
   at `hiveplot.py:2355` and `hiveplot_matrix.py:708` exist for objects predating the attribute. Under the new semantics
   the natural fallback is `None` (= infer), so they become `getattr(self, "_graph_directed", None)`. This is a behavior
   shift only for old pickled instances lacking the attr (an acceptable no-notice change for a private attr).
2. **No-notice privacy.** `_graph_directed` is underscore-private; the only "promise" it is a bool is the private
   class-level annotation at `hiveplot_matrix.py:293`. No public method returns or documents it. Changing `bool` →
   `Optional[bool]` is no-notice. Replace-and-sweep (below) confirms no public/doc/example surface promises the type.

**Feasibility audit (no new entry point, no new attribute reads).** This is a pure internal-state refactor: one private
field replaces two, the inference fallback moves from object state to a function-local `default=True`. No new
user-facing entry point, no new reads of user-input-data attributes, no signature change to any public method (the
`HivePlot.compute_graph_metrics` / `HivePlotMatrix.compute_graph_metrics` `graph_directed: Optional[bool] = None`
signatures are unchanged; the private `_apply_graph_metrics` params change but they are internal). Feasibility audit
passes trivially; no data-model mapping is touched.

**Replace-and-sweep (confirmed 2026-05-31).** `_graph_directed_explicit` appears in: `src/hiveplotlib/hiveplot.py`,
`src/hiveplotlib/hiveplot_matrix.py`, `tests/hiveplot_test.py`, `tests/hiveplot_matrix_test.py` (and this plan). **No
hand-authored `docs/` or `examples/` reference** to either private attr exists; the lone `docs/` grep hit
(`docs/source/notebooks/computing_graph_metrics.ipynb:1774`, a code-output line `hp_internal_graph._graph_directed`) is
auto-generated from the `examples/` copy and is not a hand-authored promise that the attr is a bool. Sweep clean: the
collapse touches source + tests only.

**Files and the exact sites to change (anchors are guidance; confirm against current source before editing — the brief's
line numbers were already slightly off from current source).**

*`src/hiveplotlib/hiveplot.py`:*
- `__init__` (~2002, 2057-2058): drop the `graph_directed_explicit = graph_directed is not None or graph is not None`
  derivation; stash `_graph_directed` as `Optional[bool]` (`None` on the nodes/edges-no-override path), drop the
  `self._graph_directed_explicit = ...` line.
- `_resolve_graph_or_nodes_edges` (~140-152, the `else` branch at ~150-151): return `None` on the
  nodes/edges-no-override branch instead of defaulting to `True`. The `graph is not None` branch keeps returning
  `graph.is_directed()`. Update the helper's return-type annotation (~90), the `:param graph_directed:` docstring
  (~105-106), and the `:return:` description (~112) to reflect `Optional[bool]` for the `graph_directed` slot.
- `_apply_graph_metrics` (~2229-2257): collapse the `graph_directed: bool` + `graph_directed_explicit: bool = True`
  params into a single `graph_directed: Optional[bool]`; `None` means "consult `_infer_graph_directed`." The branch
  becomes `if graph_directed is None: graph_directed = _infer_graph_directed(..., default=True)`, removing the
  `if not graph_directed_explicit:` gate. Update the docstring (~2239-2242) to describe the `None`/pinned dichotomy.
- `compute_graph_metrics` (~2350-2373): drop the `graph_directed_explicit` computation; resolve
  `graph_directed = graph_directed if graph_directed is not None else getattr(self, "_graph_directed", None)` (both may
  be `None` → infer downstream); pass the single `graph_directed` through (no `graph_directed_explicit=` kwarg). The
  defensive `getattr` default flips to `None` (decision 1).
- Param docstring (~2321-2330): the current prose leans on the "stored default `True`" framing; rewrite to describe the
  unpinned (`None`) / pinned dichotomy honestly. (Coordinate with the 2026-05-30 docstring-clarity voice already in the
  plan; keep that voice.)

*`src/hiveplotlib/hiveplot_matrix.py` (FULL MIRROR — the matrix keeps its OWN stash, not delegation):*
- class-level annotation `_graph_directed: bool` (~293) → `_graph_directed: Optional[bool]`.
- `__init__` (~410): drop the hardcoded `self._graph_directed_explicit = True`. Here directedness comes from children
  and is genuinely pinned, so stash a concrete bool (NOT `None`); just remove the `_explicit` line.
- convenience constructor #1 (derive ~1142, stash ~1299): drops the
  `graph_directed_explicit = graph_directed is not None or graph is not None` derivation; on the unpinned nodes/edges
  branch stash `None`, drop the `instance._graph_directed_explicit = ...` line.
- convenience constructor #2 (derive ~1501, stash ~1740): same pattern.
- convenience constructor #3 (~2047): drop the hardcoded `instance._graph_directed_explicit = True`; stash a concrete
  bool, drop the `_explicit` line.
- matrix `_apply_graph_metrics` (~498-520) and `compute_graph_metrics` (~704-725): mirror the HivePlot collapse — single
  `graph_directed: Optional[bool]` param, `None`-triggered `_infer_graph_directed(..., default=True)`, the defensive
  `getattr(self, "_graph_directed", None)` default (decision 1), no `graph_directed_explicit` plumbing.
- The two genuinely-pinned paths (`__init__` and ctor #3) MUST stash a concrete bool, never `None`.

*`tests/hiveplot_test.py` and `tests/hiveplot_matrix_test.py`:* the `_graph_directed_explicit` assertions across both
files (e.g. `hiveplot_test.py:6653`, `:6742-6743`, `:6788-6807`, `:6843-6898`; `hiveplot_matrix_test.py:3543-3544`,
`:3599-3619`, `:3715-3716`, `:3763-3795`, `:3867-3978`, `:3992-4049`) encode the pinned/unpinned and no-write-back
contracts. Re-express each as a `_graph_directed is None` (unpinned) / `_graph_directed is <bool>` (pinned) check rather
than deleting it. The no-write-back invariant tests (asserting the first call's inference did not mutate the stash) keep
their teaching value: assert `_graph_directed is None` still holds after an inferring call on the unpinned path. Test
name = test body contract holds (these tests already exercise the collapsed surface); per CLAUDE.md, if any named test
cannot be re-expressed against the collapsed attribute as-is, halt under rule 9 rather than substitute.

**Done when:**
- `HivePlot._graph_directed` and `HivePlotMatrix._graph_directed` are `Optional[bool]`: `None` on the unpinned
  nodes/edges-no-override path, a concrete bool on every pinned path (explicit `graph_directed`, built from a `graph`,
  `HivePlotMatrix.__init__`, and the genuinely-pinned matrix ctor). `_graph_directed_explicit` is gone from both source
  files (grep clean across `src/`).
- `_graph_multigraph` is UNCHANGED (`bool`, no `_explicit` companion, no `Optional` widening).
- `_apply_graph_metrics` on both classes takes a single `graph_directed: Optional[bool]` and, when it is `None`, derives
  the value via `_infer_graph_directed(..., default=True)`; `_infer_graph_directed`'s signature is unchanged.
- Inference is still per-call and never writes back to the stash: a build-with-`triangles` then
  `compute_graph_metrics(node_graph_metrics="in_degree")` re-infers to directed on the second call (the no-write-back
  tests prove `_graph_directed is None` persists on the unpinned path).
- The defensive `getattr(self, "_graph_directed", ...)` reads default to `None` (decision 1), not `True`.
- The `_resolve_graph_or_nodes_edges` return-type annotation, `graph_directed` `:param:`, and `:return:` docstring
  reflect `Optional[bool]`; the `HivePlot.compute_graph_metrics` / matrix-method `graph_directed` param docstrings
  describe the unpinned(`None`)/pinned dichotomy in the established 2026-05-30 voice.
- Replace-and-sweep clean: no `_graph_directed_explicit` survives in `src/` or `tests/`; no hand-authored `docs/` or
  `examples/` reference exists (confirmed); the auto-generated notebook hit is not edited.
- Tests re-expressed (not deleted): every `_graph_directed_explicit` assertion in both test files becomes an equivalent
  `_graph_directed is None` / `is <bool>` check preserving its original contract.
- `make format` / `make ty` clean; full `pytest tests/ -n 7` green; 100% coverage on both source files holds;
  warnings-as-errors hold.

**No api-critic pass required.** This is a private-internal refactor: no new user-facing API surface, no public-signature
change (the public `compute_graph_metrics` `graph_directed: Optional[bool] = None` signatures are untouched; only private
`_graph_*` state and private `_apply_graph_metrics` params change). The user-facing inference/conflict behavior is
preserved byte-for-byte (same metrics infer, same conflicts raise the same messages). So no api-critic planning OR
post-impl pass is needed for G. (If, against expectation, the engineer finds a public signature or documented behavior
must change to land the collapse, halt under rule 9 and route back to `amend-plan`.)

**No notebook change required.** `examples/computing_graph_metrics.ipynb` references only the PUBLIC `graph_directed`
surface, never the private attrs (verified), so behavior is preserved and outputs re-execute identically; no
notebook-author dispatch and no editorial-critic pass are needed for G. **One OPTIONAL taste-call, NOT a required
change:** cell ~1588 says metric computation "defaults to `graph_directed=True`." That is still accurate as observable
behavior. If the source docstring rewrite reframes around unpinned/pinned, Gary is open to a small matching prose tweak
here. Flag it as an optional notebook-author touch-up only; the notebook is not yet checked in and Gary expects it may
change slightly. Do NOT gate G on it.

**No CHANGELOG entry.** Internal refactor of private state introduced by this same unreleased plan; per the
released-behavior-only CHANGELOG rule, no entry.

### Added workstream G: Collapse the `_graph_directed` stash pair

**Status:** ✅ complete (closure reconcile 2026-06-18: the "test re-expression + qa pending" tail was stale; Implementation log "Workstream G complete" 2026-05-31). The `_graph_directed_explicit` field is fully retired — zero hits across both `src/` and `tests/` — so the test re-expression shipped alongside the source collapse.
**Files:** `src/hiveplotlib/hiveplot.py`, `src/hiveplotlib/hiveplot_matrix.py`, `tests/hiveplot_test.py`,
`tests/hiveplot_matrix_test.py`. (Operative scope, sites, and done-when are in amendment A8 above.)

### Added workstream H: Parallel-edge-collapse warning on the HivePlot graph-metrics path

**Status:** ✅ complete (closure reconcile 2026-06-18: header was stale; Implementation log "Workstream H complete" 2026-05-31). The shared `_warn_if_parallel_edges_collapse` helper is wired into `HivePlot._apply_graph_metrics` in the shipped source. (Note: the detection semantics were later narrowed to same-direction-only by Workstream M.)
**Origin:** amendment A9 (user ask). The operative rationale, placement justification, explicitness decision, rejected
directedness warning, and warnings-as-errors scoping are in A9 above.
**Files:** `src/hiveplotlib/hiveplot.py` (`_apply_graph_metrics`, `hiveplot.py:2231-2295`), plus a small detection helper
(engineer's call on module: `hiveplot.py` private helper, or a sibling in `converters.py` / `graph_features/` if it is
genuinely shared with the matrix seam — prefer ONE shared helper both class seams call). Tests are Workstream K's.
**api-critic:** planning-mode pass REQUIRED before code lands (new user-facing warning surface). Post-impl pass REQUIRED
after H ships.
**Done when:**
- A warning fires from the graph-metrics path (the `_apply_graph_metrics` seam, after `graph_multigraph` is resolved to
  a concrete bool at `hiveplot.py:2263-2268`, before/at the `nodes_edges_to_networkx(..., multigraph=graph_multigraph)`
  build at `:2270`) when ALL of: metrics are requested, resolved `graph_multigraph is False`, AND the combined edge set
  contains duplicate `(from, to)` rows that networkx will merge.
- Detection is cheap and precise (e.g. `df.duplicated(subset=[from_col, to_col]).any()` over the combined edge set per
  the converter's add-order, `converters.py:147-160`), reading the existing `edges.from_column_name` /
  `edges.to_column_name` / `edges._data`. It fires ONLY on genuine duplicates (a simple, duplicate-free graph does not
  warn) and counts the collapsed surplus for the message.
- The message names how many parallel edges are being collapsed and points at `graph_multigraph=True` (the class-level
  kwarg) to keep them distinct. Exact wording is the engineer's call, finalized against api-critic's planning pass.
- Emitted via `warnings.warn(..., stacklevel=3)` (per CLAUDE.md), matching the existing `warnings.warn` precedent
  (`p2cp.py:621`). Custom `Warning` subclass is the engineer's + api-critic's call (no `Warning` precedent in
  `src/hiveplotlib/exceptions/` today; plain `UserWarning` is acceptable).
- **Explicitness decision (per A9, api-critic to confirm):** the default is to warn whenever resolved
  `graph_multigraph is False` and real duplicates exist, regardless of whether the user set the flag. Do NOT add a
  `_graph_multigraph_explicit` field unless api-critic's planning pass affirmatively asks for suppression-on-explicit.
- The warning does NOT fire when `graph_multigraph=True` (no collapse), when no metrics are requested (no graph built),
  or when there are no duplicates.
- 100% coverage, warnings-as-errors hold (the new tests in Workstream K wrap the firing path in `pytest.warns`).

### Added workstream I: HivePlotMatrix mirror-or-confirm

**Status:** ✅ complete (closure reconcile 2026-06-18: header was stale; Implementation log "Workstream I complete" 2026-05-31). Resolved as automatic-via-shared-helper: `HivePlotMatrix` calls the same `_warn_if_parallel_edges_collapse` helper from its `_apply_graph_metrics` seam in the shipped source.
**Origin:** amendment A9.
**Files:** `src/hiveplotlib/hiveplot_matrix.py` (the static `_apply_graph_metrics`, `hiveplot_matrix.py:~497-545`, which
mirrors HivePlot's seam and builds via `nodes_edges_to_networkx(..., multigraph=graph_multigraph)` at `:~531-545`).
Tests are Workstream K's.
**Done when:**
- Decide explicitly and record in the Implementation log whether the warning is **automatic** (because Workstream H's
  detection lives in a shared helper both `_apply_graph_metrics` bodies call) or needs its **own seam** (because the
  matrix body is a separate static helper). The matrix `_apply_graph_metrics` is a distinct method body, so the warning
  must be reachable from it; the cleanest shape is a shared helper called from both. Confirm and implement.
- Confirm (per A9) that multigraph is never inferred, so the `__init__`/`from_tags` (concrete `graph_multigraph=False`
  default) vs. `from_*` inference asymmetry that affected `graph_directed` does NOT apply here: every matrix construction
  path resolves `graph_multigraph` the same way, so the warning fires uniformly. Record this confirmation.
- The matrix warning, if its text is user-facing and differs from HivePlot's, is covered by api-critic's post-impl pass
  (mechanical sibling propagation per CLAUDE.md); if it is the identical shared-helper message, note that for api-critic.
- 100% coverage, warnings-as-errors hold.

### Added workstream J: Docstrings for the parallel-edge-collapse warning

**Status:** ✅ complete (closure reconcile 2026-06-18: header was stale; Implementation log "Workstream J complete" 2026-05-31). The `graph_multigraph` collapse-warning note shipped on all seven entry points. (Note: Workstream N later tightened that note to same-direction-only semantics.)
**Origin:** amendment A9. Docstring writes are Docs Engineer's domain; the behavior lands in H/I.
**Files:** `src/hiveplotlib/hiveplot.py` and `src/hiveplotlib/hiveplot_matrix.py` (the `graph_multigraph` param
docstrings on `__init__` and the `compute_graph_metrics` methods, plus the `from_*`/`from_tags` matrix constructors that
carry the param).
**Done when:**
- The `graph_multigraph` docstrings (HivePlot + HivePlotMatrix) mention that the graph-metrics path **warns** when
  `graph_multigraph=False` collapses duplicate `(from, to)` edges, and name `graph_multigraph=True` as the escape hatch
  to keep parallel edges distinct. This rides on the existing two-world / "never inferred" framing the 2026-05-30 and
  Workstream-G doc passes landed; extend that prose, do not restructure it.
- The per-wrapper metric docstrings are NOT edited (the warning is a path-level concern, not a per-metric one); state
  this so Docs Engineer does not over-edit.
- `make docs` builds clean (scan all warnings, not first-warning). Coverage unaffected by docstring-only changes.

### Added workstream K: Tests for the warning + the `tests/` warnings-as-errors sweep

**Status:** ✅ complete (closure reconcile 2026-06-18: header was stale; Implementation log "Workstream K complete" 2026-05-31). The two dedicated `TestHivePlot*ParallelEdgeCollapseWarning` test classes shipped (26 tests). (Note: Workstream O added the same-direction-semantics tests on top after the M narrowing.)
**Origin:** amendment A9. The `tests/` sweep precondition is the load-bearing early step (see A9's corrected
warnings-as-errors scoping).
**Files:** `tests/hiveplot_test.py`, `tests/hiveplot_matrix_test.py` (mirroring the source files), plus any existing
graph-metric test sites surfaced by the sweep.
**Done when:**
- **Sweep step (run FIRST, before/alongside H landing):** inventory every existing `tests/` site that computes graph
  metrics on potentially-duplicate edge data under default `graph_multigraph=False`. For each, decide and apply:
  wrap in `pytest.warns(...)` (expect the warning) or switch to duplicate-free data. Record the inventory + per-site
  decision in the Implementation log. (Under `filterwarnings=error` an un-wrapped firing path fails CI; the sweep is the
  discipline that keeps the suite green when H lands.)
- New tests assert, via `pytest.warns(...)` (mandatory under `filterwarnings=error`): the warning **fires** on real
  duplicate `(from, to)` data under default `graph_multigraph=False` with metrics requested, on **both** HivePlot and
  HivePlotMatrix.
- New tests assert the warning does **NOT** fire (use `warnings.catch_warnings()` with `simplefilter("error")`, or rely
  on the suite-wide `filterwarnings=error` so an un-warned call simply passes) when: no duplicates exist,
  `graph_multigraph=True` is set, or no metrics are requested.
- Each `test_<scenario>` exercises the warning path through the named entry point (test-name = test-body contract per
  CLAUDE.md).
- 100% coverage on the new source (H/I), warnings-as-errors hold.

### Added workstream L: Notebook demo of the parallel-edge-collapse warning (LAST, joins shipped Workstream E)

**Status:** ✅ complete (closure reconcile 2026-06-18: header was stale). The shipped `examples/computing_graph_metrics.ipynb` carries a "Parallel Edge" section demonstrating the collapse warning (the full warning text and `warn_on_parallel_edge_collapse` are present). The editorial-critic gate is CLOSED with a ready-to-ship verdict — see the editorial RE-CLEAR recorded in the "## Notebook review" section.
**Origin:** amendment A9. Notebook prose is Notebook Author's domain.
**Files:** `examples/computing_graph_metrics.ipynb` (the ONLY file; edit only this `examples/` copy, never the
auto-generated `docs/source/notebooks/` copy).
**Done when:**
- A notebook section/cell **deliberately demonstrates** the warning printing in-cell, modeled on the existing precedent
  in `examples/edge_kwarg_hierarchy.ipynb` (which intentionally shows override/conflict warnings printing). Build edge
  data with a duplicate `(from, to)` pair, request a metric under default `graph_multigraph=False`, and show the warning
  text printed, with prose explaining the silent collapse and pointing at `graph_multigraph=True`.
- The cell does NOT need a `try/except` (a warning prints, it does not raise; `make test-nb` fails only on a raising
  cell). Confirm the notebook runs end-to-end clean under `make test-nb`.
- The demo coheres with the already-shipped Workstream E `## Controlling the HivePlot Internal Graph Type` material
  (same gallery notebook, same `HivePlot` subject, existing datasets; no new dataset, no genre/class drift). Place it
  near the `graph_multigraph` material, not early.
- **No CHANGELOG entry** (refinement to the unreleased feature, per A9 and Gary's standing rule).
- Prose follows the writing-voice rules (no em-dashes, no AI filler, length discipline) and the gallery-notebook skill.
- editorial-critic post-impl review is clean (structure/coherence: right notebook, no dataset/genre/class drift, the new
  cell earns its place and does not dwarf its siblings). viz-critic only if a figure is added (default: no new figure,
  the demo is a printed warning + a table).

### Added workstream M: Same-direction-only semantics + hybrid detection + opt-out kwarg (code)

**Status:** complete (2026-06-01)
**Origin:** amendment A10 (Gary-approved design refinement). The semantics rationale, the hybrid-branch design, the
opt-out justification, the naming audit, the API usage examples, and the api-critic gate are in A10 above.
**Files:** `src/hiveplotlib/hiveplot.py` (`_warn_if_parallel_edges_collapse` helper rewrite at `:232-280`; thread the new
kwarg through `HivePlot.__init__` and `HivePlot.compute_graph_metrics`, and through `_apply_graph_metrics` to the helper
call at `:2314`), `src/hiveplotlib/hiveplot_matrix.py` (thread the kwarg through `__init__`, `from_partition`,
`from_variable_sweep`, `from_tags`, `compute_graph_metrics`, and through the static `_apply_graph_metrics` /
`_apply_init_graph_metrics` to the helper call at `:537-ish`). Tests are Workstream O's.
**api-critic:** planning-mode pass REQUIRED before code lands (new user-facing kwarg surface + narrowed semantics);
post-impl pass REQUIRED after M ships.
**First step (confirm before designing):** confirm the helper's CURRENT shipped form is the build-diff at
`hiveplot.py:268-269` (`num_input_edges - graph.number_of_edges()`), signature
`_warn_if_parallel_edges_collapse(edges, graph, graph_multigraph, stacklevel=4)`. If the engineer finds a different
form (e.g. the older `pd.concat` scan, or an already-present opt-out kwarg), halt under rule 9 and surface the mismatch.
**Done when:**
- **Same-direction-only semantics (change 1):** the count is `rows - (distinct ORDERED (from, to) pairs)`. Reciprocal
  `(a, b)` / `(b, a)` rows that merge on an undirected build are NOT counted; same-direction `(a, b)` repeats ARE.
- **Hybrid detection (change 2):**
  - Directed build (`graph.is_directed()`): keep the existing build-diff (`num_input_edges - graph.number_of_edges()`);
    it already equals distinct-ordered-pairs, exact and free, no scan.
  - Undirected build: count distinct ordered `(from, to)` pairs with a lean vectorized pass over the edge columns across
    ALL tags combined (NOT a `pd.concat` full copy; minimize the copy; handle the common single-tag case directly). This
    is the only branch that scans.
  - Both branches yield identical same-direction-only counts on the same data.
- **Opt-out kwarg (change 3):** a new boolean `warn_on_parallel_edge_collapse` (name confirmed by api-critic per the A10
  naming audit), default `True`, threaded through the 7 entry points listed in Files. When `False`, the helper
  short-circuits IMMEDIATELY (before the multigraph check, the build-diff, and the scan), on BOTH branches. The threading
  shape (construction-time stash + per-call override vs. plain construction-time bool) is api-critic's call in the
  planning pass; implement the shape api-critic rules.
- The shipped warning MESSAGE is unchanged in wording (it already names the count, the attribute loss, and the
  `graph_multigraph=True` steer); only the COUNT it reports changes (same-direction-only). Confirm the message still
  reads correctly with the narrowed count.
- `stacklevel` behavior (4 direct seams / 5 HPM-init-routed) is preserved; the kwarg threading does not change call
  depth. The fires-once-per-source-group behavior is preserved.
- `make format` / `make ty` clean; 100% coverage on the rewritten helper's branches (multigraph early return, opt-out
  short-circuit, directed-branch build-diff, undirected-branch scan, no-collapse return, fire path); warnings-as-errors
  hold.
- **No CHANGELOG entry** (refinement to the unreleased v0.28.0 feature).

#### API Critic — post-implementation review (Workstream M)

**Done (closure reconcile 2026-06-18: this placeholder was stale).** Workstream M's surface is covered by the **"API Critic — post-implementation review (A10) — `warn_on_parallel_edge_collapse` + narrowed collapse semantics (2026-06-01)"** block earlier in this amendment (under A10), which returned **Status: clean**. That review walked all seven entry points carrying `warn_on_parallel_edge_collapse`, the rewritten `_warn_if_parallel_edges_collapse` + `_num_distinct_ordered_edge_pairs` helpers, and the narrowed same-direction-only semantics — i.e. exactly the surface Workstream M shipped. No separate post-impl pass is outstanding; this placeholder redirects to that A10 review.

### Added workstream N: Docstrings for the opt-out kwarg + narrowed semantics

**Status:** ✅ complete (closure reconcile 2026-06-18: header was stale; Implementation log "Workstream N complete" 2026-06-01. Outside the orchestrator's literal "A-K" tick instruction but ticked as the corollary that makes M/N/O consistent — M and O already read complete — and flagged in the orchestrator's report). The `warn_on_parallel_edge_collapse` docstrings and the same-direction-only `graph_multigraph` note shipped across all seven entry points.
**Origin:** amendment A10. Docstring writes are Docs Engineer's domain; the behavior lands in M.
**Files:** `src/hiveplotlib/hiveplot.py` and `src/hiveplotlib/hiveplot_matrix.py` (the new
`warn_on_parallel_edge_collapse` param docstring on all 7 entry points, plus an update to the EXISTING
`graph_multigraph` / parallel-edge-collapse-warning docstrings that Workstream J landed to reflect same-direction-only
semantics and the opt-out).
**Done when:**
- Every entry point carrying the new kwarg documents `warn_on_parallel_edge_collapse` (default `True`, what it controls,
  and that `False` skips detection entirely for scaling, contrasted with a `warnings` filter which still runs detection).
- The existing `graph_multigraph` docstring note (from Workstream J) is updated to say the warning fires on
  SAME-DIRECTION duplicate `(from, to)` rows (genuine multigraph collapse), and explicitly that reciprocal `(a, b)` /
  `(b, a)` rows merging on an undirected build do NOT warn (that is the definitional meaning of the undirected metric).
  This rides the existing two-world / never-inferred framing; extend it, do not restructure.
- The per-wrapper metric docstrings are NOT edited (path-level concern, per the J convention).
- `make docs` builds clean (scan all warnings, not first-warning). Coverage unaffected by docstring-only changes.

### Added workstream O: Test reversal + new semantics/hybrid/opt-out tests

**Status:** complete (new dedicated tests added; the 14-site reversal was done by Workstream M)
**Origin:** amendment A10. The 14-site `pytest.warns` REVERSAL is the load-bearing early step (see A10's
critical-test-reversal section for the enumerated sites).
**Files:** `tests/hiveplot_test.py`, `tests/hiveplot_matrix_test.py` (the existing
`TestHivePlotParallelEdgeCollapseWarning` / `TestHivePlotMatrixParallelEdgeCollapseWarning` classes from log entry 2062,
plus the 14 inference/directedness tests log entry 2064 wrapped).
**Done when:**
- **Reversal step (run alongside M landing):** revert the 14 `pytest.warns(...)` wrappers added by the 2026-05-31
  build-diff pass (enumerated in A10) to plain calls / `nullcontext`, since those reciprocal-only builds no longer fire.
  Per site, the engineer FIRST confirms the fixture is genuinely reciprocal-only (not also same-direction-duplicated);
  if a site mixes reciprocals AND same-direction duplicates, the warning still fires there and the wrapper stays — halt
  and surface rather than blanket-reverting.
- **New tests (each `test_<scenario>` exercises the named path; test-name = test-body contract):**
  - Same-direction duplicates DO fire on a DIRECTED build AND on an UNDIRECTED build (both hybrid branches detect genuine
    duplicates), on both HivePlot and HivePlotMatrix.
  - Reciprocals do NOT fire on an undirected build (the change-1 narrowing).
  - `warn_on_parallel_edge_collapse=False` skips detection: assert no warning fires AND, ideally, that the detection path
    is not entered (spy/monkeypatch proving the short-circuit, not just silence).
  - The hybrid branches produce CORRECT counts: a directed build counts a same-direction duplicate; an undirected build
    with both a same-direction duplicate AND a reciprocal counts ONLY the same-direction one.
- The existing fire/no-fire tests (singular/plural wording, no-metrics, `graph_multigraph=True`, stacklevel/user-frame
  attribution) still pass against the rewritten helper.
- `pytest tests/hiveplot_test.py tests/hiveplot_matrix_test.py -m networkx -n 7` green; 100% coverage on the rewritten
  helper holds; warnings-as-errors hold.
- **No CHANGELOG entry** (tests for the unreleased v0.28.0 feature).

## Implementation log

(append-only; one line per completed workstream)

2026-05-29: Workstream A complete. Added `GraphTypeRequirement` dataclass, `_enforce_graph_type` helper, and `@requires_graph_type(*, directed=None, multigraph_ok=True, hint=None)` decorator to `graph_features/networkx/_helpers.py` (decorator attaches a frozen `_hpl_graph_type_requirement` record via `functools.wraps` and runs centralized enforcement before each wrapper body; `_NO_REQUIREMENT` is the unconstrained default for `getattr` reads). Deleted all ~20 hand-written `if graph.is_directed()/is_multigraph(): raise` guard blocks from `node_metrics.py` (16 wrappers) and `edge_metrics.py` (`bridges` + the 5 link-prediction factory products), applying the decorator instead; components-family cross-references preserved via `hint=`, `onion_layers` decorated `directed=False, multigraph_ok=False` with directed-first enforcement order. Verified raise/message equivalence vs. the pre-refactor guards across all four graph variants for every wrapper (0 mismatches), no surviving inline guard (grep clean), `make format`/`ty` clean, existing 160 graph-metric tests pass. No new global introduced.
2026-05-29: Workstream B complete (source only; tests are test-engineer's). Added `_requirement_for(name)` (reads `_hpl_graph_type_requirement` off the wrapper across both master dicts, defaulting `_NO_REQUIREMENT`) and `_check_graph_type_conflicts(*, node_metrics, edge_metrics, graph, from_hive_plot)` to `graph_features/__init__.py`. The validator runs at the top of `compute_graph_metrics` after the per-side target+name checks but before any wrapper call, reasoning over the COMBINED node+edge set. Directedness standoff (a `requires_directed is True` metric and a `requires_directed is False` metric together) raises one `ValueError` naming both sides in backticks, stating they cannot share one internal graph (noting `graph.is_directed()`), with resolution text branching on entry point. Multigraph rejection (`graph.is_multigraph()` and a `rejects_multigraph` metric) raises a parallel one-sided error. Added private keyword-only `_from_hive_plot: bool = False` to `compute_graph_metrics` selecting the HivePlot vs. dispatcher-direct message (HivePlot wiring is Workstream C's job). Restructured the dispatcher body so name validation moved up front and the two redundant inner `_check_metric_names` calls were removed; added `assert target_nodes/target_edges is not None` to keep `ty` happy after the up-front target checks. Added a `:raises ValueError:` docstring line for the conflict. `make format`/`ty` clean; existing `tests/graph_features_test.py -m networkx -n 7` 536 passed, 8 skipped.
2026-05-29: Workstream A tests added to `tests/graph_features_test.py` (mirrors the flat `graph_features_test.py` convention; the `_helpers` adapter tests already live there). New section covers: (1) the per-wrapper raise/verbatim-message sweep parametrized over every `GRAPH_NODE_METRICS` + `GRAPH_EDGE_METRICS` key x the four directed/multigraph variants, with expected raise/no-raise and exact message text derived from each wrapper's own `_hpl_graph_type_requirement` (the natural form), plus a `_graph_variant`/`_expected_message` helper pair; (2) explicit hint assertions: `connected_components` directed-graphs tail, `strongly`/`weakly_connected_components` undirected tail, and `onion_layers` two-axis (directed/multigraph/both) confirming directed-first order; (3) `topological_generations` isolated (needs a DAG; complete-graph directed variant is cyclic and would raise unrelated `NetworkXUnfeasible`, so the sweep skips it and a dedicated test exercises the undirected-raise + DAG-happy-path cleanly); (4) registry-agreement: every metric resolves a `GraphTypeRequirement` (default `requires_directed=None, rejects_multigraph=False` for unconstrained), and `_hpl_graph_type_requirement` predicts actual raise outcome per variant; (5) decorator/helper unit tests for `requires_graph_type` and `_enforce_graph_type` directly (tri-state directed True/False/None, `multigraph_ok` toggle, `hint` append, `functools.wraps` name/docstring preservation, directed-before-multigraph order, `_hpl_graph_type_requirement` field values). ~409 new parametrized cases. `pytest -m networkx -n 7`: 536 passed, 8 skipped (the `topological_generations` rows in the two sweeps, by design); `_helpers.py` at 100% coverage (42/42 statements, 0 missing); warnings-as-errors clean; `ruff format`/`ruff check`/`ty` clean. No source touched.
2026-05-29: Workstream B tests added to `tests/graph_features_test.py` (flat layout; new "Up-front graph-type conflict validation" section after the Workstream A decorator tests). 12 new tests + a `_multigraph_fixture` helper: pure-node directedness conflict (via `compute_graph_metrics` and one direct `_check_graph_type_conflicts` call), cross-boundary conflict (`in_degree` node + `bridges` edge proving node+edge share one graph), one-sided multigraph conflict (dispatcher `multigraph=False` branch and HivePlot `graph_multigraph=False` branch), no-conflict pass-through (`triangles`+`clustering`+`bridges` attaches columns), the load-bearing A1/A2 message-branch pair (dispatcher message asserted to NOT contain `graph_directed` and to instruct `to_undirected`/"OWN graph"/"each call needs a graph of the right type"; HivePlot message asserted to contain `graph_directed=True`/`graph_directed=False` and NOT `to_undirected`), the no-wasted-computation spy (monkeypatched `GRAPH_NODE_METRICS` with `functools.wraps` spies preserving `_hpl_graph_type_requirement`, asserting neither wrapper ran before the raise), and singular/plural agreement (single-metric-per-side "requires", multi-metric-per-side "require", multi-metric multigraph "do not support"/"these metrics"). Pure-edge directedness conflict (scenario 2) is structurally impossible: every guarded edge metric in `GRAPH_EDGE_METRICS` declares `requires_directed=False` (no directed-required edge metric exists), so that case is covered via the cross-boundary scenario and documented in a section comment. `pytest tests/graph_features_test.py -m networkx -n 7`: 548 passed, 8 skipped (pre-existing `topological_generations` sweep skips); `src/hiveplotlib/graph_features/__init__.py` at 100% coverage (132/132 statements, 0 missing) with both message branches, the multigraph branch, singular/plural, and the pass-through all covered; warnings-as-errors clean; `ruff format`/`ruff check`/`ty` clean. No source touched.
2026-05-29: Workstream C tests added (test-engineer). `tests/hiveplot_test.py` (`TestHivePlotNetworkx`): 12 new tests covering the nodes/edges inference path (the only `graph_directed_explicit=False` entry point, so the only one that exercises `_infer_graph_directed`, including the previously-uncovered `concrete.pop()` line) — headline undirected-only `triangles` infers and computes (stash unchanged, asserting inference is per-call), all-directed `["in_degree","out_degree"]` infers True, agnostic-only `degree` keeps default, ambiguous `["in_degree","triangles"]` defers to the WS-B conflict `ValueError` (message asserted to reference `graph_directed`, name both metrics, say "cannot share one internal graph"), explicit `graph_directed` True/False win over inference, `graph_multigraph` never inferred (explicit `True` + `clustering` still conflicts; undirected inference leaves multigraph default), and the `compute_graph_metrics` method path (omitted flag defers to the `_graph_directed_explicit=False` stash and infers; per-call `graph_directed` True/False is explicit). `tests/hiveplot_matrix_test.py` (`TestHivePlotMatrixNetworkx`): 18 new tests (incl. parametrized over `from_partition`/`from_variable_sweep`) mirroring the infer-capable classmethod paths (undirected-only, all-directed, agnostic-only, ambiguous-defers-to-validator, explicit True/False, no multigraph inference) plus the documented KNOWN ASYMMETRY that `HivePlotMatrix.__init__` and `from_tags` (concrete `graph_directed=True` default, no sentinel) stash `_graph_directed_explicit=True` and never infer (so `triangles` raises there absent an explicit `graph_directed=False`), and the matrix method-path inference. ~30 new tests / 40 parametrized cases. Full `pytest tests/ -n 7`: 1387 passed, 8 skipped (pre-existing `topological_generations`); `hiveplot.py` and `hiveplot_matrix.py` at 100% coverage (the C inference lines, gating, explicitness capture, and `_from_hive_plot=True` wiring all covered); warnings-as-errors, `ruff format`/`ruff check`/`ty` clean. No source touched.
2026-05-29: Workstream D complete (docs-engineer). The conflict-raise and inference docstring prose had already been authored inline by the B/C source engineers across all five required sites, so this pass verified accuracy against the shipped behavior rather than re-authoring: dispatcher `compute_graph_metrics` (`graph_features/__init__.py`) carries the two new `:raises ValueError:` entries (directedness standoff + multigraph rejection) plus the amendment-A3 "does not infer graph type; inference is a HivePlot/HivePlotMatrix construction-time convenience" sentence, and does NOT list the private `_from_hive_plot` in the public param block; `HivePlot.__init__` / `HivePlot.compute_graph_metrics` (`hiveplot.py`) document the up-front conflict error, the two-call resolution, and the inference-with-explicit-wins behavior on `graph_directed` + the failure-mode notes on `node_graph_metrics`/`graph_multigraph`; `HivePlotMatrix` mirrors all five methods (`from_partition`, `from_variable_sweep`, `compute_graph_metrics`, `__init__`, `from_tags`) and documents the asymmetry honestly (`__init__`/`from_tags` take a concrete `graph_directed=True`, never infer, and steer users to the `from_*` classmethods or an explicit `graph_directed=False`; the inferring paths are `from_partition`/`from_variable_sweep`). Docs-engineer fixes applied: three docstring lines were 121 chars (trimmed to <=120 at `graph_features/__init__.py:421`, `hiveplot_matrix.py:1128`, `:1484`), and a Sphinx `py:meth reference target not found: __init__` warning in `HivePlotMatrix.compute_graph_metrics` was resolved by demoting the bare `:py:meth:`__init__`` cross-ref to plain ``` ``__init__`` ``` (the file's existing convention; the qualified `from_*` cross-refs resolve fine). `make docs` builds clean with zero warnings; `ruff format`/`ruff check` clean. No new public API surface, so no `docs/source/autodoc/` rst changes and no notebook index entries; no external links added, so no linkcheck. **Duplication risk (flagged, not fixed):** `docs/source/_ext/metric_table_directive.py` independently classifies graph-type requirements by empirically probing each wrapper against four graph variants. It does not read the `_hpl_graph_type_requirement` record this plan made the single source of truth; it is a separate encoding, out of scope here, flagged so a future pass can consolidate (already noted in "Patterns this replaces" and "Holdouts").
2026-05-29: Workstream C complete (source only; tests are test-engineer's). Added module-level `_infer_graph_directed(node_graph_metrics, edge_graph_metrics, *, default)` + `_as_metric_name_list` helpers to `hiveplot.py` (reads each requested node+edge metric's `requires_directed` via `graph_features._requirement_for`, discards `None`/agnostic; returns the single concrete value when exactly one remains, else `default` so a zero-concrete agnostic set keeps the default and a two-concrete contradictory set defers to Workstream B's validator). Threaded a new `graph_directed_explicit: bool = True` kwarg through `HivePlot._apply_graph_metrics` and `HivePlotMatrix._apply_graph_metrics` / `_apply_init_graph_metrics`; inference runs inside `_apply_graph_metrics` right before `nodes_edges_to_networkx(..., directed=graph_directed)`, only when not explicit. Explicitness captured at each build site BEFORE `_resolve_graph_or_nodes_edges` collapses the `None` sentinel: `HivePlot.__init__`, `HivePlotMatrix.from_partition`, `from_variable_sweep` compute `graph_directed is not None or graph is not None` and stash it as `_graph_directed_explicit`; the `compute_graph_metrics` methods on both classes treat a per-call `graph_directed` as explicit else defer to the stash. `HivePlotMatrix.__init__` / `from_tags` take a concrete `graph_directed: bool = True` (no sentinel, no `graph=`), so they stash `_graph_directed_explicit=True` (always pinned, no inference) — a deliberate, documented asymmetry. Also wired `_from_hive_plot=True` into the `compute_graph_metrics(...)` call inside both `_apply_graph_metrics` methods so the conflict error uses the HivePlot-flavored (`graph_directed`) message. Multigraph inference intentionally not added (asymmetric axis; default `False` always safe). `make format`/`ty` clean; 611 networkx tests pass (8 pre-existing `topological_generations` skips); smoke-confirmed `node_graph_metrics="triangles"` infers undirected on both classes and explicit `graph_directed=True` + triangles raises the `graph_directed`-referencing conflict.
2026-05-29: Workstream F complete (source only; the three sweep reconstructor helpers are the test-engineer's job). Re-opened the WS-A verbatim-locked text (per A6, option C) in the three `_enforce_graph_type` message templates (`graph_features/networkx/_helpers.py:58-80`): each resolution sentence now names BOTH the class-level kwarg and the low-level one. directed-required: "Build the source graph as directed: pass `graph_directed=True` on `HivePlot` / `HivePlotMatrix` initialization, or `directed=True` on `nodes_edges_to_networkx`."; undirected-required: same shape with `graph_directed=False` / `directed=False`; multigraph-rejected: "...as a non-multigraph: pass `graph_multigraph=False` ... or `multigraph=False` on `nodes_edges_to_networkx`." Raise behavior unchanged (same metrics raise on the same directed×multigraph variants, directed-before-multigraph order for `onion_layers` preserved, `hint=` tails still appended by `_enforce_graph_type`). Verified no per-metric wrapper docstring quotes these base strings (grep shows the strings live only in `_helpers.py`). `make format`/`ty` clean; `pytest tests/graph_features_test.py -m networkx -n 7` shows 62 failures, ALL `AssertionError` on the expected-string assertions (the WS-A sweep, the components/onion hint tests, and the decorator/helper unit tests) — zero non-AssertionError failures, raise/no-raise outcomes intact — pending the test-engineer's update of `_expected_directed_required_msg` / `_expected_undirected_required_msg` / `_expected_multigraph_rejected_msg` (`graph_features_test.py:1199-1220`). Added an "Other Changes" CHANGELOG note.
2026-05-30: Docstring clarity pass (docs-engineer, Gary-approved framing; no behavior/signature/CHANGELOG change). Restructured the `graph_directed` / `graph_multigraph` param docstrings on the construction-time entry points so resolution behavior splits by HOW the object was built rather than blending the cases into one dense paragraph, using inline `**From ``graph``:**` bold labels in flowing prose (no nested list blocks, to keep the RST warning-free). `HivePlot.__init__` + `HivePlotMatrix.from_partition` / `from_variable_sweep` (all have a `graph` param and a `None` sentinel): two-world framing — **From `nodes`/`edges`** infers from the requested set (default `True` absent a deciding metric), **From `graph`** takes `graph.is_directed()` / `graph.is_multigraph()` and does NOT re-infer (opposite-need metric raises the conflict `ValueError`, steer to explicit flag or `graph.to_undirected()`). `HivePlotMatrix.__init__` / `from_tags` (concrete `graph_directed: bool = True`, no sentinel, no `graph` path) documented honestly as never-inferring: directedness is whatever you pass, an undirected-only metric like `triangles` needs an explicit `graph_directed=False` here, steer to the inferring `from_*` classmethods otherwise. The two `compute_graph_metrics` METHODS (HivePlot + HivePlotMatrix) reframed construction-state-shaped (pinned vs. unpinned) to match the `graph_directed_explicit = graph_directed is not None or self._graph_directed_explicit` resolution: explicit-this-call wins and does not mutate the stash; left `None`, a PINNED construction (explicit flag, or built from a `graph`; for HPM also `__init__`/`from_tags` which are always pinned) reuses the stored value with no re-inference (opposite-need metric raises), while an UNPINNED `nodes`/`edges` construction infers when unambiguous (else stored default `True`). This corrected the prior HivePlot-method text that understated the `graph=` case as un-pinned. `graph_multigraph` everywhere kept its accuracy (never inferred; `None` uses construction-time stored value; nodes/edges default `False` noted to differ from `to_networkx`), just aligned to the same voice/structure. Confirmed signatures + resolution logic against source before writing (hiveplot.py:2000, 2353-2360; hiveplot_matrix.py:321-322, 410, 707-713, 986-987, 1338-1339, 1772-1773). `make docs` builds clean with ZERO warnings (bold-inline-label RST renders fine). No code logic, signatures, defaults, or CHANGELOG touched; docstrings only.
2026-05-29: Workstream F tests updated (test-engineer). Re-aligned the three WS-A message-fidelity reconstructor helpers in `tests/graph_features_test.py` (`_expected_directed_required_msg` / `_expected_undirected_required_msg` / `_expected_multigraph_rejected_msg`, `:1199-1222`) to the clarified post-A6 wording: each now reconstructs the new "pass `graph_directed=...` on `HivePlot` / `HivePlotMatrix` initialization, or `directed=...` on `nodes_edges_to_networkx`" form (and `graph_multigraph=False` / `multigraph=False` for the multigraph-rejected case), matching the shipped `_helpers.py:58-81` source character-for-character (confirmed against source, including the ` / ` spacing and trailing period before any hint tail). No other change needed: every exact-match assertion in the sweep (`test_requires_graph_type_enforcement_matches_requirement`), the `topological_generations` isolated test, the components-family / `onion_layers` hint tests, and the decorator/helper unit tests all route through these three helpers (the standalone `match=` / `in message` checks assert only unchanged fragments, so no inline base-string copies existed to chase). Assertions kept exact-match (no weakening to substring/regex); the components-family hint tails are still asserted appended after the new base text. `pytest tests/graph_features_test.py -m networkx -n 7`: 548 passed, 8 skipped (pre-existing `topological_generations` sweep skips); `_helpers.py` at 100% coverage (42/42 statements, 0 missing); warnings-as-errors clean; `ruff format`/`ruff check` clean. No source touched.
2026-05-31: Workstream G complete (source only; the `_graph_directed_explicit` test re-expression is test-engineer's). Collapsed the `(_graph_directed: bool, _graph_directed_explicit: bool)` stash pair into a single `_graph_directed: Optional[bool]` on both classes (`None` = unpinned, infer per call; concrete bool = pinned), retiring `_graph_directed_explicit` entirely; `_graph_multigraph` left untouched (`bool`, no inference channel). `hiveplot.py`: `_resolve_graph_or_nodes_edges` now returns `None` (not `True`) on the nodes/edges-no-override branch (return-type annotation `bool`→`Optional[bool]`, `:param graph_directed:`/`:return:` docstrings updated); `__init__` dropped the `graph_directed_explicit` derivation and the `_explicit` stash (the resolved value flows straight to the stash, `None` when unpinned), dropped the `graph_directed_explicit=` kwarg to `_apply_graph_metrics`; `_apply_graph_metrics` param collapsed to `graph_directed: Optional[bool]` with the gate now `if graph_directed is None: ... default=True` (was `if not graph_directed_explicit: ... default=graph_directed`); `compute_graph_metrics` dropped the `graph_directed_explicit` computation, flipped the defensive `getattr` default `True`→`None`, stopped passing `graph_directed_explicit`; the method's `graph_directed` param docstring reframed from "stored default True" to the unpinned(`None`)/pinned dichotomy in the 2026-05-30 voice. `hiveplot_matrix.py` (full mirror, all six sites + the third pinned ctor): class annotation `_graph_directed: bool`→`Optional[bool]`; `__init__` and `from_tags` dropped the hardcoded `_graph_directed_explicit = True` (both stash a concrete bool, genuinely pinned); `from_partition`/`from_variable_sweep` dropped the `graph_directed_explicit` derivation, the `_explicit` stash, and the `_apply_graph_metrics` kwarg (unpinned nodes/edges branch now stashes `None`); the static `_apply_graph_metrics` and `_apply_init_graph_metrics` params collapsed to `Optional[bool]` with the same `if graph_directed is None: ... default=True` gate; the method `compute_graph_metrics` flipped the defensive `getattr` default to `None` and the param docstring to the unpinned/pinned framing. `_infer_graph_directed`'s `default: bool` signature unchanged. Grep `_graph_directed_explicit` across `src/` is clean (0 hits). `ruff format`/`ruff check`/`ty check src/` all clean. Smoke-confirmed HivePlot per-call re-inference and no-write-back: unpinned build stashes `None`, stays `None` across two inferring `compute_graph_metrics` calls (including a directed-input `in_degree` re-infer), graph-built pins to `False`. No CHANGELOG (internal refactor of unreleased private state). One optional taste-call surfaced to Gary (notebook cell ~1588 "defaults to graph_directed=True"); not gated on G.
2026-05-31: Workstream G tests re-expressed (test-engineer). Translated every `_graph_directed_explicit` reference in `tests/hiveplot_test.py` and `tests/hiveplot_matrix_test.py` (both inside the `pytest.mark.networkx`-marked `TestHivePlotNetworkx` / `TestHivePlotMatrixNetworkx` classes) onto the single `_graph_directed: Optional[bool]` encoding; grep `_graph_directed_explicit` across `tests/` is now clean (0 hits). Mapping applied: unpinned `_explicit is False` → `_graph_directed is None`; pinned `_explicit is True` → `_graph_directed is <concrete bool>` (`is False` for an explicit `graph_directed=False` pin, `is True` for a directed-input/from-graph pin or `__init__`/`from_tags` concrete default). The dead-state triangles init case (`hiveplot_test.py` `test_hiveplot_init_infers_undirected_for_undirected_only_metric`, formerly asserting stashed `_graph_directed=True` alongside `_explicit=False`) now asserts `_graph_directed is None` post-build and that inference still produces the `triangles` column. No test renames needed: the only `explicit` test-name fragments (`test_hpm_classmethod_explicit_directed_true/false_*`) name the user-passed `graph_directed` flag, not the retired attribute, and stay accurate; no test was named for `_graph_directed_explicit`. **No-write-back invariant strengthened** in both `..._compute_graph_metrics_method_omitted_flag_infers` tests (HivePlot + the `from_partition`/`from_variable_sweep`-parametrized HPM mirror): final assertions now lock `assert hp._graph_directed is None` (resp. `hpm`) BEFORE the first call, AFTER the `triangles` (undirected) inference, AND AFTER the `["in_degree","out_degree"]` (directed) re-inference — proving per-call inference never writes a concrete directedness back to the unpinned stash and the second call re-infers cleanly. Three docstrings referencing the old pair (the two `_inference_digraph` fixtures + the per-call/from-graph non-mutation tests) reworded to the pinned/unpinned framing; no bug-archaeology or process provenance added. `pytest tests/hiveplot_test.py tests/hiveplot_matrix_test.py -n 7 -m networkx --no-cov`: 116 passed, 0 failures. No source/notebook touched; coverage gate left to qa-engineer.
2026-05-31: Workstream G complexity-review follow-up (docs-engineer; the two should-fix + one nice-to-have from the A8 complexity read, docstring prose only, no behavior/signature/rendering/CHANGELOG/notebook change). **Change 1 (hoist "never inferred" to the first clause of every `graph_multigraph` docstring):** rewrote all six public `graph_multigraph` param descriptions to open with "unlike `graph_directed`, this is never inferred from the requested metrics. It controls whether each row ..." so the asymmetry is visible in a tooltip/first glance — `HivePlot.__init__` (`hiveplot.py:1918`, the laggard), `HivePlot.compute_graph_metrics` (`hiveplot.py:~2340`), `HivePlotMatrix.__init__` (`hiveplot_matrix.py:275`), `HivePlotMatrix.compute_graph_metrics` (`hiveplot_matrix.py:~688`), `from_partition` + `from_variable_sweep` (`hiveplot_matrix.py:1100`, `:1454`, identical text), `from_tags` (`hiveplot_matrix.py:1893`). **Change 2 (shared lead sentence on the two non-inferring matrix constructors):** confirmed constructor identity by signature default (`__init__` `:320` and `from_tags` `:1761` take `graph_directed: bool = True`; `from_partition` `:977` / `from_variable_sweep` `:1328` take `Optional[bool] = None`), then prepended a verbatim-identical lead sentence to the `graph_directed` param on `HivePlotMatrix.__init__` (`hiveplot_matrix.py:267`) and `from_tags` (`hiveplot_matrix.py:1886`): "this constructor fixes directedness to whatever you pass and never infers it from the requested metrics; use :py:meth:`from_partition` or :py:meth:`from_variable_sweep` if you want directedness inferred from the metric set." **Change 3 (cluster-level framing note):** added a `.. note:: **Controlling the internal graph type.**` block to the `HivePlot.__init__` lead section (`hiveplot.py:~1804`, immediately after the existing internal-graph-rebuild note — the most-read API-reference home for the cluster, per the review's recommendation #3) tying `graph_directed` + `graph_multigraph` + `graph_source_attribute_name` together as the internal-graph-type knobs, stating the one inference rule, and pointing at the "Controlling the `HivePlot` Internal Graph Type" section of `computing_graph_metrics` (notebook NOT edited, only referenced). All edited lines wrap <=120; voice scanned for em-dashes (none; the only em-dash in either file is a pre-existing code comment, exempt). Could not run `make docs` (this shell is Windows-side Git Bash with no `make`/`uv` on PATH and the Linux `.venv` binaries error on UNC exec; `make docs` also rebuilds every example notebook end-to-end, unneeded for prose-only changes). Validated RST instead with Windows-side docutils 0.22.4: the `.. note::` directive and every edited `:param:` field-list block parse clean — the only docutils complaints are `:class:`/`:py:class:`/`:py:meth:` "unknown role" notices, which Sphinx registers at build time and which already pepper these same docstrings. No autodoc rst or notebook index changes (no new public surface); no external links added (no linkcheck).
2026-05-31: Workstream G editorial-critic doc-sufficiency follow-up (notebook-author; two additive markdown-only edits to `examples/computing_graph_metrics.ipynb`, no code cells, no re-execution). (1) In the section-intro cell `b69c9a50` ("Controlling the `HivePlot` Internal Graph Type"), added the governing "one rule" sentence right after the directedness-inference sentence: directedness CAN be inferred from the requested metrics when unambiguous, but `graph_multigraph` is NEVER inferred (it is only ever what you set explicitly, else the construction default `False` from `nodes`/`edges` or the input graph's own type from `graph=`). (2) In the closing cell `3d907517`, added a single-line `HivePlotMatrix` cross-reference noting it carries the same `graph_directed`/`graph_multigraph` controls so the section's mental model transfers, kept light as a pointer (not HPM-constructor documentation). Only those two markdown cells changed; both additions em-dash-free and in Gary's voice. No CHANGELOG (prose refinement to the unreleased feature).
2026-05-31: Workstream G cluster-note propagation (docs-engineer; docstring prose only, no behavior/signature/rendering/CHANGELOG/notebook change). Two fixes building on Gary's hand-rephrased `.. note:: **Controlling the internal graph type.**` block (`hiveplot.py:1804`, now the source of truth). **Fix 1 (HivePlot reference):** replaced the hand-wavy "'Controlling the `HivePlot` Internal Graph Type' section of the `computing_graph_metrics` notebook" tail with a real Sphinx redirect `:doc:`Computing Graph Metrics </notebooks/computing_graph_metrics>`` (page's actual H1 title + built doc path, leading-slash absolute, mirroring the established `toy_hive_plots.py:251` pattern); dropped the section-name mention (URL-fragment noise to a docs reader). Rest of Gary's note kept verbatim. **Fix 2 (HPM propagation):** verified the four `HivePlotMatrix` constructor signatures by default before writing — `__init__` (`:321`) and `from_tags` (`:1764`) take `graph_directed: bool = True` (never infer), `from_partition` (`:978`) and `from_variable_sweep` (`:1330`) take `graph_directed: Optional[bool] = None` (infer when unset) — confirming the inference split. Added the sibling note to the `HivePlotMatrix` CLASS docstring (right after the `graph_source_attribute_name` param at `hiveplot_matrix.py:282`, the primary class entry where the full param block lives since `__init__`'s own docstring is a stub), with the directedness sentence ADAPTED for accuracy: directedness is inferred from the requested metrics only on `from_partition`/`from_variable_sweep` when `graph_directed` is left unset, while `__init__`/`from_tags` keep whatever directedness you pass and never infer, a `graph` input keeps its own type, and `graph_multigraph` is never inferred on any path. Same `:doc:`Computing Graph Metrics`` redirect kept. Bolded lead + overall shape/voice match Gary's HivePlot version so the two read as siblings. All edited lines wrap <=120; scanned both edits for em-dashes (none). `make` unavailable in this Windows-side Git Bash shell (no `uv`/`.venv` on PATH over the UNC mount), so validated RST with Windows-side docutils 0.22.4 instead: both `.. note::` blocks (with `:doc:`/`:py:meth:` roles registered as generic) parse with ZERO warnings. No autodoc rst or notebook index changes (no new public surface); no external links added (no linkcheck). No CHANGELOG (docs refinement to unreleased feature).
2026-05-30: Workstream E complete (notebook-author). Structurally revised the `## Controlling the HivePlot Internal Graph Type` section of `examples/computing_graph_metrics.ipynb` (the only file; the auto-generated `docs/source/notebooks/` copy untouched) per A7. Everything before that section is unchanged and re-ran for fresh outputs. The section now tells the graph-type story for both init paths in parallel. **`### When Initializing from nodes and edges`:** (a) inference "just works" — the toy `example_hive_plot(num_edges=500)` computes `triangles` with no `graph_directed` (E-1), followed by an explicit behavior-change callout; (b) the single-metric break **replaces the now-broken cell `e6ca4494`** — pin `graph_directed=True` to block inference, then `triangles` raises (E-2); the stale `_graph_directed` / `nodes.data` inspection cells (`75c6b213`, `040dc431`) are **removed**; the override-on-init cell (`c691a9b5`) is kept and re-run; (c) a one-line pointer to the consolidated conflict subsection. **`### When Initializing from graph`:** (a) the undirected-input demo (`663a1ba4`) kept, with new prose making the inference-is-absent contrast explicit; (b) a **new** single-metric break from a directed `nx.DiGraph(nx.karate_club_graph())` input that raises with no flag (E-2g), plus the existing undirected-overridden-to-directed flavor (`fe49d861`/`60cbcdfb`) kept and re-run; (c) a one-line pointer. **New `### When Requested Metrics Disagree` subsection:** the directedness standoff `["in_degree", "triangles"]` via HivePlot (E-3a, built from fresh `networkx_to_nodes_edges(G)` for self-containment), the cross-boundary `in_degree` + `bridges` node/edge conflict KEPT as its own cell (E-6), a two-call resolution cell producing both columns (chaining; sorts by `unique_id` since the raw Karate `club` column is non-numeric and would break axis sorting), and the multigraph rejection `["degree", "clustering"]` + `graph_multigraph=True` (E-4). The optional direct-`compute_graph_metrics` form (E-5) was deliberately **omitted** to keep the subsection from dwarfing its siblings (taste call surfaced in the report). All six error demos use `try/except` + `traceback.print_exc()`; net new figures: zero. Re-executed the full notebook against the project venv (WSL, kernel `hiveplotlib`); all 31 code cells ran, 0 error outputs, every captured traceback matches the A7-verified message text byte-for-byte (re-verified all eight examples against the settled tree before writing). Passes the real CI gate: `pytest -c tests/pytest_examples.ini -k computing_graph_metrics` (warnings-as-errors) → 1 passed, no warnings. Voice clean (0 em-dashes, 0 AI-filler across the whole notebook); `ruff format --check` clean and no E501 on authored cells under `examples/ruff.toml` (one pre-existing I001 import-sort nit in the untouched imports cell, invisible to the root/CI ruff config, left for Gary). No CHANGELOG entry (refinement to the unreleased feature, per A7 done-when). editorial-critic post-impl pending.

2026-05-31: Polish to the Workstream E two-call resolution cell (`metrics-disagree-resolution-code`) in `examples/computing_graph_metrics.ipynb` (the only file touched). Changed the first call's `sorting_variables="unique_id"` to `sorting_variables="in_degree"`. That call already computes `in_degree`, so sorting by it is on-theme for a graph-metrics notebook and mirrors the "compute a metric and use it as the sorting variable in one call" pattern the degree section uses; it supersedes the original `unique_id` choice noted in the 2026-05-30 entry (that note's reasoning still holds for why the raw Karate `club` string column can't sort, but `in_degree` is a valid numeric sort and the better fit here). The cell's output is `nodes.data.head()`, whose row order does not depend on the axis sort, so the visible table is byte-for-byte unchanged. Re-ran the full notebook against the project venv (WSL, kernel `hiveplotlib`) and saved it in place; passes the real CI gate `pytest -c tests/pytest_examples.ini -k computing_graph_metrics` with `-W error` (warnings-as-errors) → 1 passed, 0 warnings. No CHANGELOG entry (refinement to the unreleased feature).

2026-05-31: Workstream H complete (source + existing-test sweep; dedicated fire/no-fire tests are Workstream K's). Added a shared module-level helper `_warn_if_parallel_edges_collapse(edges, graph_multigraph)` to `hiveplot.py` (sibling to `_infer_graph_directed`, so Workstream I can import it into the matrix seam exactly as `hiveplot_matrix.py` already imports `_infer_graph_directed`/`_resolve_graph_or_nodes_edges`). Wired it into `HivePlot._apply_graph_metrics` at the seam right after `graph_directed` resolves and before the `nodes_edges_to_networkx(...)` build (`hiveplot.py:267` call site). **Detection:** early-returns when `graph_multigraph` is truthy (no collapse); otherwise concatenates the `[from_col, to_col]` columns across ALL tags of `edges._data` (mirroring the converter, which adds every tag's edges into one graph, so duplicates are counted over the combined set, not per tag), and counts the collapsing surplus as `combined.duplicated(subset=[from_col, to_col]).sum()`; returns silently when zero. Fires only on genuine duplicates. **Warning class:** plain `UserWarning` (no `Warning`-subclass precedent in `src/hiveplotlib/exceptions/`, which holds only `Error` classes; matches the existing `warnings.warn` precedent at `p2cp.py:621` / `viz/datashader.py:145`, per api-critic's planning lean; suppression is via standard `warnings` filters). **Exact final message** (singular/plural agreement on edge(s)): `"{N} duplicate \`(from, to)\` edge(s) will be merged into single edges for metric computation because \`graph_multigraph=False\`. networkx keeps only the last duplicate's edge attributes (weights, types, etc.); the others are dropped before the metric runs. Pass \`graph_multigraph=True\` to keep parallel edges distinct."` — names the count, names the attribute loss (last-write-wins, weights/types dropped) per must-fix #1, and steers to the class-level `graph_multigraph=True` and ONLY that (not the low-level `multigraph=True`) per must-fix #2. `stacklevel=3` per CLAUDE.md so it points at the user's call site. **Explicitness:** stateless per the settled decision — no `_graph_multigraph_explicit` field; warns whenever resolved `graph_multigraph is False` and real duplicates exist, regardless of whether the user set the flag. **Existing-test sweep:** exactly one HivePlot-path test genuinely collapses real duplicates under `graph_multigraph=False`: `tests/hiveplot_test.py::TestHivePlotNetworkx::test_hiveplot_init_graph_infers_graph_multigraph` (the `hp_simple` branch, which deliberately collapses the repeat `(0,1)` edge to drop node 1's degree from 3 to 2). Wrapped that one construction in `with pytest.warns(UserWarning, match="will be merged into single edges")` (the collapse is genuine and on-purpose for the test). All other multigraph-with-duplicates HivePlot tests (`...defaults_to_stored_graph_multigraph`, `...graph_source_attribute_name`) use `graph_multigraph=True` so no warning fires; the dispatcher (`compute_graph_metrics`) and `HivePlotMatrix` seams do not call the helper (HPM is Workstream I), so no test there is affected by this change. Verified zero warning-caused failures across `hiveplot_test.py` / `hiveplot_matrix_test.py` / `graph_features_test.py` / `converters_test.py`; the 38 remaining failures in this Windows-side env are all pre-existing missing-optional-dependency failures (`scipy`/`datashader` not installed), unrelated to this change. `ruff format`/`ruff check` clean on both touched files; `ty check` clean on the new helper lines (the 12 ty diagnostics on `hiveplot.py` are pre-existing optional-backend module-resolution noise and an unrelated pre-existing unused-ignore at `:4359`, none in the added code). No CHANGELOG (behavior debuting within the unreleased v0.28.0 graph-metrics feature). Flag for Workstream K: dedicated tests should assert all three helper branches (multigraph-true early return, no-duplicate return, fire path) plus the singular/plural edge wording.
2026-05-31: Workstream I complete (source + existing-test sweep; dedicated HPM fire/no-fire tests are Workstream K's). **Automatic-via-shared-helper, NOT a new seam.** Wired `HivePlotMatrix` to call the SAME shared `_warn_if_parallel_edges_collapse` helper Workstream H added to `hiveplot.py` (no new helper, no new message, no new state): imported it alongside the existing `_infer_graph_directed` / `_resolve_graph_or_nodes_edges` from `hiveplotlib.hiveplot` (`hiveplot_matrix.py:34-39`), then called it at the matrix metrics seam in the static `HivePlotMatrix._apply_graph_metrics` right after `graph_directed` resolves to a concrete bool and before the `nodes_edges_to_networkx(..., multigraph=graph_multigraph)` build (`hiveplot_matrix.py:537`), exactly mirroring HivePlot's `hiveplot.py:2314` placement. **Fires once per metric computation, not per cell:** `_apply_graph_metrics` is the static helper the `from_*` classmethods (`:1179/:1534/:1953`) call once on the shared underlying nodes/edges before constructing cells, and `_apply_init_graph_metrics` (`:610-625`) calls it once per distinct underlying nodes/edges group (deduplicated by `id()`), so the warning is not emitted N times per matrix. So because H's detection lives in a shared helper both `_apply_graph_metrics` bodies call, the HPM warning is automatic and carries H's identical message naming `graph_multigraph=True` (satisfies api-critic should-fix #3). **Asymmetry confirmation (per A9):** the helper depends ONLY on the resolved concrete `graph_multigraph` and the edge data, neither of which involves inference; multigraph is never inferred and `graph=` pins to `graph.is_multigraph()`, so the `__init__`/`from_tags` (concrete `graph_multigraph=False`) vs. `from_*` inference asymmetry that complicated `graph_directed` does NOT apply: every HPM construction path resolves `graph_multigraph` the same way and the warning fires uniformly. **Existing-test sweep:** zero HPM sites needed fixing. The only HPM tests that build duplicate-edge (`repeat (0,1)`) data and compute metrics (`hiveplot_matrix_test.py:~3222`, `:~3291`) all pass `graph_multigraph=True`, so the helper early-returns and no warning fires; the multigraph-inference tests build from `MultiGraph`/`DiGraph` inputs (resolved `graph_multigraph=True`) and likewise do not warn. Confirmed by green runs (no `filterwarnings=error` trip). `pytest tests/hiveplot_matrix_test.py -m networkx -n 7 --no-cov`: 66 passed; `tests/hiveplot_test.py tests/graph_features_test.py -m networkx -n 7 --no-cov`: 415 passed (WSL `.venv`, all extras). `ruff format`/`ruff check`/`ty check` clean on `hiveplot_matrix.py`. No CHANGELOG (refinement to the unreleased graph-metrics feature).
2026-05-31: Workstream H/I post-impl must-fix (api-critic A9): corrected the parallel-edge-collapse warning's `stacklevel` so it points at the user's call on all six reachable entry points. Took **approach (1)** (thread an explicit stacklevel; surgical, no seam hoist, fires-once behavior untouched): added `stacklevel: int = 4` to `_warn_if_parallel_edges_collapse` (`hiveplot.py`) and `_warn_stacklevel: int = 4` to the static `HivePlotMatrix._apply_graph_metrics`, with the `_apply_init_graph_metrics` dedup-loop call site (`hiveplot_matrix.py:~613`) passing `_warn_stacklevel=5`. **Empirically the prior hardcoded `stacklevel=3` was off-by-one on EVERY path, not only the two HPM-init-routed ones the critic flagged** (the critic's relative diagnosis — init-routed paths need +1 over direct paths — was correct, but the absolute base was one too low). Confirmed per-path depths against current source by recording the warning's attributed `(filename, lineno)`: HivePlot `__init__` / `compute_graph_metrics` need **4** (helper→`_apply_graph_metrics`→user); HPM `from_partition`/`from_variable_sweep`/`from_tags` need **4** (helper→static→classmethod→user); HPM `__init__` / `compute_graph_metrics` need **5** (helper→static→`_apply_init_graph_metrics`→`__init__`/`compute`→user). Verified with an ad-hoc script (since removed) asserting the recorded warning's `(filename, lineno)` equals the user's own call line on all four spot-checked paths (the two previously-broken HPM init-routed paths now land on the user's `HivePlotMatrix(...)` / `hpm.compute_graph_metrics(...)` line instead of the internal `hiveplot_matrix.py:428` frame; the HivePlot-init and HPM-`from_partition` regression guards now also land on the user's line, where `stacklevel=3` had landed on the internal `self._apply_graph_metrics(...)` line); fires-once-per-distinct-source-group reconfirmed (two cells sharing one source → exactly one warning). No message-text change, no new warning class, no new state, no CHANGELOG. `make format`/`make ty` clean; `pytest tests/hiveplot_matrix_test.py tests/graph_features_test.py -n 7` 559 passed, `tests/hiveplot_test.py -k "graph or metric or directed or multigraph or collapse or parallel"` 49 passed (WSL `.venv`, all extras).
2026-05-31: Workstream K complete (test-engineer; dedicated parallel-edge-collapse warning tests, no source touched). Added two new `@pytest.mark.networkx` test classes, one per class seam: `TestHivePlotParallelEdgeCollapseWarning` in `tests/hiveplot_test.py` (10 tests) and `TestHivePlotMatrixParallelEdgeCollapseWarning` in `tests/hiveplot_matrix_test.py` (16 tests = 12 methods, two parametrized x2 over `from_partition`/`from_variable_sweep`). Each class builds its own duplicate-edge fixtures (`_edges_with_duplicates(num_duplicates)` repeats the `(0,1)` pair N times; `_edges_no_duplicates` for the silent case). **HivePlot coverage:** fires at construction (`HivePlot.__init__`) and via the `compute_graph_metrics` method; silent on (a) no duplicates, (b) `graph_multigraph=True`, (c) no metrics requested (each silent case under `warnings.catch_warnings()` + `simplefilter("error")` so an unexpected warning fails); singular `1 duplicate ... edge` vs. plural `2 duplicate ... edges` wording. **HivePlotMatrix coverage:** fires on `from_partition`/`from_variable_sweep` (parametrized) + `from_tags` (separate, no `progress`/`graph=` kwarg) + `__init__` + the `compute_graph_metrics` method; silent under `graph_multigraph=True` and with no metrics; singular/plural wording; **fires-once-per-source-group** test builds a 3-cell dict matrix from ONE shared `HivePlot` cell and asserts exactly one collapse warning via `warnings.catch_warnings(record=True)` (proving no per-cell spam). **Stacklevel / user-frame attribution (the api-critic must-fix regression guard):** five `record=True` tests capture `call_lineno = inspect.currentframe().f_lineno + 1` immediately before the triggering call and assert the recorded `warning.filename == __file__` AND `warning.lineno == call_lineno`, covering one direct seam (`HivePlot.__init__`, `HivePlotMatrix.from_partition`) plus both init-routed paths (`HivePlotMatrix.__init__`, HPM `compute_graph_metrics` method) and the HivePlot method path; these lock the now-correct `stacklevel=4` (direct) / `5` (init-routed) and would catch any off-by-one regression. All expected-warning tests use `pytest.warns(UserWarning, match=...)`; all should-be-silent tests confirm silence. `pytest tests/hiveplot_test.py::TestHivePlotParallelEdgeCollapseWarning tests/hiveplot_matrix_test.py::TestHivePlotMatrixParallelEdgeCollapseWarning -m networkx -n 7`: **26 passed**; the parent networkx classes still green (`TestHivePlotNetworkx` + `TestHivePlotMatrixNetworkx`: 116 passed). Helper coverage: all of `_warn_if_parallel_edges_collapse` (`hiveplot.py:232-279`) executes (multigraph-true early return, no-duplicate return, fire path, singular vs. plural `edge`/`edges`) — none of those lines appears in `--cov=src/hiveplotlib/hiveplot --cov-report=term-missing` Missing output. `ruff format`/`ruff check` clean on both touched files. No source touched; no CHANGELOG (tests for behavior debuting within the unreleased v0.28.0 graph-metrics feature).
2026-05-31: Workstream J complete (docs-engineer; docstring prose only, no behavior/signature/rendering/CHANGELOG/notebook change). Appended a short, consistent note about the new parallel-edge-collapse `UserWarning` to all seven `graph_multigraph` param docstrings, riding the existing two-world / "never inferred" framing (extended, not restructured). The note lands on the collapsing (`False`) case in each: when `graph_multigraph=False` and the edge data has duplicate `(from, to)` rows, a `:class:`UserWarning`` notes networkx merges those duplicates last-write-wins (dropped duplicates' edge attributes lost) before metrics run; pass `graph_multigraph=True` to keep parallel edges distinct, or silence the warning with a standard Python `warnings` filter if the collapse is intended (the equivalent low-level kwarg being `multigraph=True` on `:py:func:`hiveplotlib.converters.nodes_edges_to_networkx``). Per the api-critic planning take, the two facts the runtime warning string deliberately omits (the low-level `multigraph=True` kwarg and the `warnings`-filter suppression path) live in the docstrings, not the runtime string, kept to one brief clause each. Sites touched: `HivePlot.__init__` `graph_multigraph` (`hiveplot.py:1972`, **From `nodes`/`edges`** `False` branch) and `HivePlot.compute_graph_metrics` method (`hiveplot.py:2403`, resolved-`False` branch); `HivePlotMatrix.__init__` (`hiveplot_matrix.py:279`), `from_partition` + `from_variable_sweep` (identical two-world text, `:1127` / `:1482`), `from_tags` (`:1923`), and `compute_graph_metrics` method (`:709`). Confirmed each location's current framing before editing (two-world `**From``graph``:**` constructors vs. concrete-`False`-default `__init__`/`from_tags` vs. resolved-value methods); the per-wrapper metric docstrings were NOT edited (the warning is a path-level concern, not per-metric, per the done-when). All edited lines wrap <=120; scanned for em-dashes (none). `make docs` (regular target, WSL): build succeeded with ZERO new warnings. No autodoc rst or notebook index changes (no new public surface); no external links added (no linkcheck). No CHANGELOG (refinement to the unreleased v0.28.0 graph-metrics feature).

2026-06-01: Workstream N follow-on doc sweep CORRECTION (docs-engineer; docstring prose only, no behavior/signature/rendering/CHANGELOG/notebook change). Fixes a misleading escape-hatch in the prior follow-on sweep entry above. The canonical clause presented `pass ``graph_multigraph=True`` or pre-aggregate the weights before building if you need both kept` as a universal fix, but several undirected metrics REJECT multigraphs (registry: `_hpl_graph_type_requirement.allows_multigraph=False`), so `graph_multigraph=True` is unavailable to them and the only recourse is pre-aggregation. The old wording erased that dead-end. **Registry verification:** read `requires_graph_type` call sites in `graph_features/networkx/node_metrics.py` / `edge_metrics.py`. The metric that is BOTH undirected-required AND multigraph-rejecting (`directed=False, allows_multigraph=False`) is `onion_layers` (node_metrics) plus the link-prediction edge wrappers (edge_metrics `:146`). Used `onion_layers` as the named example. (Note vs. memory candidates: `clustering` and `core_number` reject multigraphs but are directedness-AGNOSTIC `directed=None`, not strictly undirected-required; `triangles` is `directed=False` but ALLOWS multigraphs. So `onion_layers` is the unambiguous BOTH-axes example, used in place of the brief's `clustering` placeholder.) **Corrected canonical clause (the 7 entry points, identical):** "That reciprocal merge keeps one row's attributes last-write-wins, harmless for unweighted or structural metrics, but a metric given an edge ``weight`` reflects only one of two differing reciprocal weights. ``graph_multigraph=True`` keeps both edges only for metrics that accept multigraphs; metrics that require a simple graph (e.g. ``onion_layers``) reject the multigraph, so for those pre-aggregate the reciprocal weights into a single edge before building." Sites: `hiveplot.py` `HivePlot.__init__` + `compute_graph_metrics`; `hiveplot_matrix.py` `__init__`, `from_partition`, `from_variable_sweep`, `from_tags`, `compute_graph_metrics`. **Dispatcher note (`graph_features/__init__.py` `compute_graph_metrics`):** applied the same metric-dependent correction adapted to its pre-built-graph framing (supplying a multigraph keeps both edges only for accepting metrics; reject-multigraph metrics need pre-aggregation). **Converter note (`converters.py` `nodes_edges_to_networkx`):** this one is METRIC-AGNOSTIC (the converter builds graphs, does not run the metric), so did NOT add the metric-dependent caveat; instead removed the "a metric given an edge ``weight``..." metric-consumption framing that implied a downstream metric can always use the multigraph, rewording to a build-level statement: reciprocals merge last-write-wins under `multigraph=False`, carrying only the surviving row's attributes; build with `multigraph=True` to retain both as parallel edges, or pre-aggregate beforehand. All edited lines wrap <=120; scanned for em-dashes (none). `make docs` (regular target, WSL): build succeeded with ZERO new warnings. No autodoc rst / notebook index / external-link changes. No CHANGELOG.
2026-06-01: Workstream O complete (test-engineer; new dedicated tests for the A10 refined behavior. The 14-site `pytest.warns` reversal was already done by Workstream M, not redone here. No source touched). Added two new `@pytest.mark.networkx` test classes per source file. **`tests/hiveplot_test.py`:** `TestHivePlotParallelEdgeCollapseSameDirectionSemantics` (13 tests) + `TestNumDistinctOrderedEdgePairs` (2 helper tests). **`tests/hiveplot_matrix_test.py`:** `TestHivePlotMatrixParallelEdgeCollapseSameDirectionSemantics` (12 tests). Fixtures: `_edges_same_direction_duplicate` (one genuine `(0,1)` repeat, no reciprocal), `_edges_reciprocal_only` (`(0,1)`/`(1,0)`, no same-direction repeat), `_edges_two_duplicates_and_reciprocal` (two `(0,1)` repeats + a `(2,3)`/`(3,2)` reciprocal). **Coverage of the rewritten `_warn_if_parallel_edges_collapse` branches:** (1) same-direction duplicate FIRES on a DIRECTED build (`degree`, inferred directed → `graph.number_of_edges()` branch) and on an UNDIRECTED build (`triangles` inferred undirected on HivePlot, `graph_directed=False` on HPM since `__init__` never infers → `_num_distinct_ordered_edge_pairs` scan branch), both classes; (2) reciprocal-only undirected build is SILENT (the change-1 narrowing), both classes, asserted under `warnings.simplefilter("error")`; (3) opt-out: `warn_on_parallel_edge_collapse=False` is silent on both branches, AND the short-circuit-precedes-detection proof is via `monkeypatch.setattr(hiveplotlib.hiveplot, "_num_distinct_ordered_edge_pairs", <raises AssertionError>)` on the undirected path (confirming the helper is never called with the flag `False`; a raise would fail the test), plus a directed-branch silence assertion; (4) per-call override semantics on `compute_graph_metrics`: per-call `False` suppresses; omitted (`None`) falls back to the construction stash (build default `True` → omitted call fires; build `False` → omitted call silent); a per-call `False` leaves `self.warn_on_parallel_edge_collapse is True` and a later omitted call (with `node_graph_metric_rename` to dodge the degree-column collision) still fires, proving no stash mutation; (5) hybrid count correctness: with 2 same-direction dups + a reciprocal, both directed and undirected builds report exactly `2 duplicate ... edges` (reciprocal excluded). **Helper unit tests:** `_num_distinct_ordered_edge_pairs` single-tag returns 3 distinct over `[(0,1),(1,2),(0,1),(2,3)]`, multi-tag returns 3 counting a cross-tag `(0,1)` duplicate once (covers the `pd.concat` multi-tag path). One in-flight state mismatch caught and corrected before reporting: the first append anchored on a 4-line tail block that recurs across the two stacklevel-attribution tests, inserting the new classes mid-`TestHivePlotParallelEdgeCollapseWarning` and orphaning its `compute_graph_metrics` attribution test under the helper class (it failed with `AttributeError: no attribute '_nodes'`); re-anchored on the file's true end and restored the orphaned test to its original class (this is the expertise "append on the file's true tail, not a recurring body line" gotcha). `pytest tests/hiveplot_test.py tests/hiveplot_matrix_test.py -m networkx -n 7`: 168 passed (52 in the four collapse-warning classes, the original `TestHivePlot*ParallelEdgeCollapseWarning` classes included and green). 100% coverage on both helpers held: a `--cov=src/hiveplotlib/hiveplot --cov-report=term-missing` run shows no line in 232-327 (the two helpers) in the Missing output (every branch covered: opt-out return, multigraph return, directed branch, undirected branch, fire, no-fire; single-tag + multi-tag helper paths). `ruff format`/`ruff check` clean (EM101 fixed by assigning the AssertionError message to a variable; several over-120 docstrings reworded). No CHANGELOG (tests for the unreleased v0.28.0 feature).
2026-05-31: Workstream H/I perf fix (code-engineer; Gary flagged the duplicate scan as needlessly O(E) on every metrics computation). Replaced the per-call O(E) `pd.concat`+`.duplicated()` scan inside `_warn_if_parallel_edges_collapse` with a near-free post-build edge-count diff: networkx already de-duplicates when building a simple graph, so the collapse count is now `sum(len(df) for df in edges._data.values()) - graph.number_of_edges()` (input rows is O(num_tags); `number_of_edges()` is ~O(V), reusing the build that already happens). **New signature:** `_warn_if_parallel_edges_collapse(edges, graph, graph_multigraph, stacklevel=4)` (added the built `graph` param; `graph: "nx.Graph"` resolves via the existing `TYPE_CHECKING` import). **Branches:** `if graph_multigraph: return` (a multigraph preserves parallels, so nothing is counted) -> `if num_collapsing <= 0: return` -> singular/plural `edge`/`edges` -> warn. **Call-site move:** in BOTH `_apply_graph_metrics` seams (`hiveplot.py` HivePlot, `hiveplot_matrix.py` HPM) the warning call moved from BEFORE the `nodes_edges_to_networkx(...)` build to immediately AFTER it, passing the built `graph`. Message text, count semantics, fires-once behavior, warning class, and `stacklevel` (4 direct seams / 5 HPM-init-routed) all unchanged; the move is a few lines down within the same function, so call depth is identical (verified: all five K stacklevel/user-frame-attribution tests still pass). **Intended correctness improvement (undirected reciprocal collapse):** the old ordered-tuple `[from, to]` scan treated `(a, b)` and `(b, a)` as distinct, missing the genuine silent collapse when an UNDIRECTED simple graph merges reciprocal rows into one edge; the build-diff counts these correctly. Updated 14 tests whose expected fire-state/count changed ONLY for this reason (all build undirected via inferred-undirected `triangles` or explicit `graph_directed=False` from reciprocal-containing `_inference_*` fixtures): wrapped the now-correctly-firing undirected build in `pytest.warns(UserWarning, match="will be merged")`. In `tests/hiveplot_test.py`: `test_hiveplot_init_infers_undirected_for_undirected_only_metric`, `..._init_explicit_directed_false_honored`, `..._init_graph_explicit_directed_wins_over_inference` (78-collapse symmetric-DiGraph -> undirected karate rebuild), `..._compute_graph_metrics_method_omitted_flag_infers` (first call only), `..._method_per_call_false_honored`, `..._method_per_call_false_honored_on_graph_build`. In `tests/hiveplot_matrix_test.py`: `test_hpm_classmethod_inference_on_nodes_edges_path` (conditional `pytest.warns`/`nullcontext` on the `triangles` param only, added `contextlib` import), `..._classmethod_explicit_directed_false_honored`, `..._init_explicit_false_needed_for_triangles_under_asymmetry`, `..._from_tags_explicit_false_needed_for_triangles_under_asymmetry` (2 collapses: official + social reciprocals), `..._compute_graph_metrics_method_omitted_flag_infers` (first call only), `..._method_per_call_false_honored`, `..._method_per_call_false_honored_on_graph_build`. No test changed for any other reason. Rationale: no opt-out kwarg needed since detection is now ~free (Gary's original always-on-cost concern is resolved). `pytest tests/hiveplot_test.py tests/hiveplot_matrix_test.py tests/graph_features_test.py tests/converters_test.py -m networkx -n 7`: 533 passed; full `tests/ -n 7` confirms `hiveplot_matrix.py` at 100% and the helper's four branches covered (none in term-missing output). `make format`/`make ty` clean. No CHANGELOG (refinement to the unreleased v0.28.0 graph-metrics feature).
2026-06-01: Workstream M complete (code-engineer; A10 three coupled changes + the 14-site test reversal). Confirmed the shipped helper was the expected build-diff form before designing. **Change 1+2 (same-direction-only via hybrid):** `_warn_if_parallel_edges_collapse` now counts `num_input_edges - num_distinct_ordered_pairs`. On a DIRECTED build (`graph.is_directed()`) it keeps the existing free build-diff (`graph.number_of_edges()` already equals distinct ordered pairs on a DiGraph, no scan). On an UNDIRECTED build it calls a new module-level helper `_num_distinct_ordered_edge_pairs(edges)` that returns `len(pairs.drop_duplicates())` over the `[from_col, to_col]` columns: the common single-tag case reads that one tag's two columns directly (no concat), multi-tag stacks only the two id columns per tag via `pd.concat(..., ignore_index=True)` (dropped the deprecated `copy=False` kwarg, which tripped a Pandas4Warning under warnings-as-errors) so duplicates are counted across all tags combined. Reciprocal `(a,b)`/`(b,a)` rows merging on an undirected build are no longer counted; same-direction `(a,b)` repeats still are. **Change 3 (opt-out kwarg):** added `warn_on_parallel_edge_collapse` threaded through all 7 entry points per the api-critic hybrid ruling: **plain `bool = True`** on the 5 constructors (`HivePlot.__init__`, `HivePlotMatrix.__init__`/`from_partition`/`from_variable_sweep`/`from_tags`), stashed as `self.warn_on_parallel_edge_collapse` exactly like `warn_on_overlapping_kwargs` (HPM also gets a class-level `warn_on_parallel_edge_collapse: bool` annotation); **`Optional[bool] = None`** per-call override on the 2 `compute_graph_metrics` methods, resolved `if warn_on_parallel_edge_collapse is None: warn_on_parallel_edge_collapse = getattr(self, "warn_on_parallel_edge_collapse", True)` (per-call value applies to that call only, no stash mutation). Threaded into `_apply_graph_metrics` / `_apply_init_graph_metrics` and into the helper, whose FIRST line is now `if not warn_on_parallel_edge_collapse: return` (short-circuits before the multigraph check and any count/scan, both branches). Added a minimal accurate `:param:` line on each of the 7 entry points making clear the flag controls the WARNING/detection only (edges still merge last-write-wins; pointing to `graph_multigraph=True` to preserve them); the surrounding `graph_multigraph` docstrings left for Workstream N. Message wording unchanged; only the reported count narrows. **Test reversal (14 sites, all confirmed reciprocal-ONLY before reverting, none mixed):** reverted the `pytest.warns("will be merged")` wrappers added by the 2026-05-31 build-diff pass to plain calls. 6 in `tests/hiveplot_test.py` (the `_inference_nodes_edges` fixture `(2,0)`/`(0,2)` reciprocal, and the symmetric `nx.DiGraph(karate)` whose every edge is a reciprocal pair); 8 in `tests/hiveplot_matrix_test.py` (the `_inference_nodes_edges` matrix fixture reciprocal, the multi-tag `from_tags` fixture whose official `(2,0)`/`(0,2)` and social `(6,7)`/`(7,6)` are both reciprocal-only, and the `omitted_flag_infers` first-call); converted the one conditional `pytest.warns`/`nullcontext` block to a plain call and removed the now-unused `contextlib` import. `make format`/`make ty` clean; `pytest tests/hiveplot_test.py tests/hiveplot_matrix_test.py tests/graph_features_test.py -n 7`: 877 passed. No CHANGELOG (refinement to the unreleased v0.28.0 graph-metrics feature).
2026-06-01: Readability refactor of the parallel-edge-collapse caveat (docs-engineer; docstring prose only, no behavior/signature/rendering/CHANGELOG/notebook change). Per Gary's feedback that the collapse/weighted-reciprocal caveat was too verbose shoved inside the `graph_multigraph` param, moved the full collapse-WARNING + weighted-reciprocal material OUT of the `graph_multigraph` param on all 7 entry points and into ONE consolidated method-level `.. note:: **Parallel-edge collapse.**` in each method body, leaving the param lean. **Kept in each `graph_multigraph` param:** the core two-world definition (nodes/edges default `False`; `graph=` uses `graph.is_multigraph()`) plus the short pointer "See the note on parallel-edge collapse in this method's description." (plain English, no `:ref:`/`:doc:` anchor, so no cross-reference target to resolve). **Moved to the body note (identical text at all 7 so they cannot drift):** when `graph_multigraph=False` (the default for nodes/edges) and the edge data has same-direction duplicate `(from, to)` rows, those collapse last-write-wins and a `:class:`UserWarning`` fires (skippable via `warn_on_parallel_edge_collapse=False`); reciprocal `(a, b)`/`(b, a)` rows merging under an undirected build do NOT warn (definitional meaning of the undirected metric); either collapse keeps one row's attributes, so a metric given an edge `weight` reflects only one of two differing reciprocal weights; `graph_multigraph=True` retains both edges only for multigraph-accepting metrics, while simple-graph-requiring metrics (e.g. `onion_layers`, registry-verified multigraph-rejecting) need pre-aggregation. **Placement** follows the existing `graph_directed`/`graph_multigraph`-control-the-internal-graph note convention: alongside that note in the two `compute_graph_metrics` methods and after the "Controlling the internal graph type" note in `HivePlot.__init__`/`HivePlotMatrix.__init__`; in the three `from_*` classmethods it follows the "internal graph is rebuilt" note. Sites: `hiveplot.py` `HivePlot.__init__` + `compute_graph_metrics`; `hiveplot_matrix.py` `__init__`, `from_partition`, `from_variable_sweep`, `from_tags`, `compute_graph_metrics`. **Converter (`converters.py` `nodes_edges_to_networkx`) and dispatcher (`graph_features/__init__.py` `compute_graph_metrics`) confirmed already carrying their caveat as body `.. note::` admonitions (not in params) and LEFT AS-IS.** The `warn_on_parallel_edge_collapse` param descriptions kept their footgun-leading wording (controls the warning/detection only, not whether edges collapse; for keeping edges use `graph_multigraph=True`) unchanged. All edited lines wrap <=120; scanned for em-dashes (none). `make docs` (regular target, WSL): build succeeded with ZERO new warnings (the plain-English note reference introduced no broken cross-reference). No autodoc rst / notebook index / external-link changes. No CHANGELOG.
2026-06-01: Workstream N complete (docs-engineer; docstring prose only, no behavior/signature/rendering/CHANGELOG/notebook change). **Task 1 (`warn_on_parallel_edge_collapse` on all 7 entry points):** M had already authored the api-critic verbatim text on each entry point, so this pass verified it satisfies the FOOTGUN ruling rather than re-authoring. Each `:param warn_on_parallel_edge_collapse:` leads with what the flag controls (detect-and-warn about same-direction duplicate `(from, to)` edges collapsing during metric computation), states the default `True`, says `False` skips the detection entirely as a performance escape hatch, and carries the load-bearing disambiguation that it does **not** change whether edges collapse (duplicates still merge last-write-wins), routing a user who wants to KEEP parallel edges to `graph_multigraph=True`. The 2 `compute_graph_metrics` methods additionally carry the per-call-override semantics (left `None` uses the value stored at construction, default `True`; an explicit value applies to this call only and does not mutate the stored setting), consistent with how `graph_directed`/`graph_multigraph` describe theirs. No edit needed for Task 1 (text already correct and matches the verbatim). **Task 2 (tighten the `graph_multigraph` warning note for same-direction-only semantics):** the J-pass note on all 7 `graph_multigraph` params previously triggered on "duplicate `(from, to)` rows"; updated each to fire on "same-direction duplicate `(from, to)` rows" and appended one sentence per site: "The warning fires only on same-direction duplicates; reciprocal `(a, b)` / `(b, a)` rows that merge only because an undirected build treats the pair symmetrically do **not** warn (that merge is the definitional meaning of the undirected metric)." Extended the existing two-world / never-inferred framing in place (no restructure); the per-wrapper metric docstrings were NOT touched (path-level concern, per the J convention). Sites: `hiveplot.py` `HivePlot.__init__` (`:2022`) + `compute_graph_metrics` (`:2469`); `hiveplot_matrix.py` `__init__` (`:280`), `compute_graph_metrics` (`:734`), `from_partition` + `from_variable_sweep` (`:1173`/`:1542`), `from_tags` (`:1999`). All edited lines re-wrapped <=120 (fixed two that overflowed after the insertion); scanned for em-dashes (none). `make docs` (regular target, WSL): build succeeded with ZERO new warnings. `make format` clean. No autodoc rst or notebook index changes (no new public surface); no external links added (no linkcheck). No CHANGELOG (refinement to the unreleased v0.28.0 graph-metrics feature).
2026-06-01: Workstream N follow-on doc sweep (docs-engineer; docstring prose only, no behavior/signature/rendering/CHANGELOG/notebook change). Documents the silent-reciprocal-merge weighted-loss limitation consistently everywhere it is relevant, extending the reciprocal note N added. **Canonical wording (the 7 graph-metric entry points, identical clause inserted right after "...definitional meaning of the undirected metric)." so they cannot drift):** "That reciprocal merge keeps one row's attributes last-write-wins, which is fine for unweighted or structural metrics, but a metric given an edge ``weight`` then reflects only one of two differing reciprocal weights; pass ``graph_multigraph=True`` or pre-aggregate the weights before building if you need both kept." Sites 2 (all 7): `hiveplot.py` `HivePlot.__init__` (`graph_multigraph` param, the `**From ``nodes``/``edges``:`` `False` branch) + `HivePlot.compute_graph_metrics` method (resolved-`False` branch); `hiveplot_matrix.py` `HivePlotMatrix.__init__`, `from_partition` + `from_variable_sweep` (identical two-world text, edited together via replace_all), `from_tags`, and `compute_graph_metrics` method. **Site 1 (converter-level reframing, no "warning" mention since the converter has none):** extended the existing `.. note::` in `nodes_edges_to_networkx` (`converters.py`) to also state that an undirected :py:class:`networkx.Graph` merges reciprocal `(a, b)` / `(b, a)` rows last-write-wins, that this is fine for unweighted/structural metrics, and that a metric given an edge ``weight`` reflects only the surviving edge's weight; routes to ``multigraph=True`` (the converter-level kwarg) or pre-aggregation. **Site 3 (dispatcher-level reframing, framed for the direct-API user):** added a new `.. note::` to `compute_graph_metrics` (`graph_features/__init__.py`) after the same-network-consistency paragraph: if the supplied `graph` merged parallel or reciprocal edges (e.g. a simple `Graph`/`DiGraph`), a metric given an edge ``weight`` reflects only the surviving edge's weight; pass a multigraph or pre-aggregate. **Site 4 assessed and SKIPPED (both deliberate, per the brief's judgment call):** (a) `BaseHivePlot.to_networkx` — defaults to `multigraph=True` (collapse is opt-in there) and bills itself a "thin wrapper" around `nodes_edges_to_networkx`, whose docstring (site 1) now carries the full note; duplicating it here for a path whose default never triggers the merge would be bloat. (b) `_warn_if_parallel_edges_collapse` — internal underscore-prefixed helper, no autodoc surface; its docstring already explains the reciprocal-vs-same-direction mechanics for a maintainer, and the user-action caveat (pass `graph_multigraph=True`/pre-aggregate) is developer-doc noise there. All edited lines wrap <=120; scanned for em-dashes (none). `make docs` (regular target, WSL): build succeeded with ZERO new warnings (no broken `:class:`/`:py:` refs introduced). No autodoc rst or notebook index changes (no new public surface); no external links added (no linkcheck). No CHANGELOG (refinement to the unreleased v0.28.0 graph-metrics feature).
