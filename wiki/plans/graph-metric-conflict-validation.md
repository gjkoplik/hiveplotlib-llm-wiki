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
`@requires_graph_type` decorator on each metric wrapper that (1) attaches a `_hpl_requirement` record to the wrapped
function and (2) generates the enforcement: it wraps the body so a single shared helper `_enforce_graph_type(name,
requirement, graph)` runs first and raises the `ValueError` if the graph violates the requirement. The ~20 hand-written
`if graph.is_directed(): raise` / `if graph.is_multigraph(): raise` blocks are deleted from the wrapper bodies; each body
shrinks to its happy path. So the guards are genuinely replaced by decorator-generated enforcement (the runtime raise
still happens, in one centralized place instead of ~20 inline copies). The requirement data lives on the functions (read
via `getattr(GRAPH_NODE_METRICS[m], "_hpl_requirement", <default>)` across the existing node+edge dicts); there is **no
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
    hand-written guard block; the decorator attaches a `_hpl_requirement` record AND generates the runtime raise via the
    shared `_enforce_graph_type` helper. The decorator-attached record is the single source of truth that both the new
    up-front validator (Workstream B) and the docs metric-table directive (future pass) can read. There is no new
    standalone requirements-dict global. QA's replace-and-sweep should confirm **no** hand-written `if
    graph.is_directed(): raise` / `if graph.is_multigraph(): raise` blocks survive in the wrapper bodies; the
    message-fidelity sweep test in Workstream A proves the centralized raise is behavior- and message-equivalent to the
    deleted guards.

- **Duplication risk, flagged not fixed:** `docs/source/_ext/metric_table_directive.py` already classifies the same
  graph-type axes (`directed_only`, `undirected_only`, `simple_only`, `simple_undirected_only`, `dag_required` at
  `:147-181`) by *empirically probing* each wrapper against four graph variants (`metric_table_directive.py:101-137`).
  After this plan the requirement facts live in two places: the decorator-attached `_hpl_requirement` record (now the
  single source of truth for both the runtime raise and the up-front validator) and this independent, empirical docs
  probe (which discovers requirements by running each wrapper against four graph variants). The docs probe does not
  consume `_hpl_requirement` today and is not in scope to change here. Flag for a future pass: the docs directive could
  read `_hpl_requirement` instead of probing, collapsing the two encodings into one. Out of scope for this plan; noted so
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
  function as `_hpl_requirement`. The record carries `requires_directed: Optional[bool]` (`True` = needs directed,
  `False` = needs undirected, `None` = agnostic), `rejects_multigraph: bool` (derived from `multigraph_ok`), and an
  optional `hint: Optional[str]` (the metric-specific message tail, e.g. the components-family cross-references). Field
  names mirror the guard-clause vocabulary already in the codebase (`is_directed`, `is_multigraph`) and the docstring
  phrasing ("requires a directed graph", "does not support multigraphs"). The tri-state `Optional[bool]` on directedness
  matches the genuine three-way nature of that axis (required / forbidden / agnostic); the boolean `rejects_multigraph`
  matches the asymmetric nature of that axis (some reject, none require). These names are internal (naming-audit-exempt)
  but fixed here so Workstreams A and B share one vocabulary. **No standalone requirements-dict global** (Gary's explicit
  ask): consumers read the record off the function via `getattr(GRAPH_NODE_METRICS[m], "_hpl_requirement", <default>)`.
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

## Notebook review

```
Pending — invoke editorial-critic in post-implementation mode after Workstream E ships.
```

Workstream E is the **only** notebook-touching workstream, and it is hard-gated (see below). The notebook
(`examples/computing_graph_metrics.ipynb`) is a gallery notebook; the addition is a short prose blurb plus a small cell
demonstrating the conflict error and the two-call resolution. No new dataset, no genre drift, no class-scope change (still
a `compute_graph_metrics` / `HivePlot` demonstration), so editorial review is a coherence check, not a sign-off-gated
restructure.

## Workstreams

Dependency order: A (decorator + enforcement) → B (validator + error) and C (inference) can run after A, B and C may run concurrently
but coordinate on the resolved-flag path → D (propagation/docstrings) after B and C → **E (notebook) LAST and GATED**.

### Workstream A: `@requires_graph_type` decorator + centralized enforcement (replaces the hand-written guards)

**Status:** not started
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
  attaches a `GraphTypeRequirement` record as `_hpl_requirement` (`requires_directed: Optional[bool]`,
  `rejects_multigraph: bool` from `not multigraph_ok`, `hint: Optional[str]`), and (2) wraps the body so the shared
  `_enforce_graph_type(name, requirement, graph)` helper runs first and raises the `ValueError` on violation.
- Every guarded wrapper in `GRAPH_NODE_METRICS` and `GRAPH_EDGE_METRICS` is decorated and its hand-written guard block is
  deleted; the body shrinks to its happy path. A metric with no guard either records `requires_directed=None,
  rejects_multigraph=False` (decorated as a no-op) or carries no `_hpl_requirement` (consumers default it the same way
  via `getattr(..., _hpl_requirement, <default>)` — engineer's call, but the default must match an unconstrained metric).
- **No new standalone global.** Requirement data is read off the functions via `getattr(GRAPH_NODE_METRICS[m],
  "_hpl_requirement", <default>)` across the existing node+edge dicts. No `GRAPH_METRIC_REQUIREMENTS` dict is introduced.
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
  raise behavior across the four graph variants" probe, now reading the requirement from `_hpl_requirement` rather than a
  separate dict, and asserting the record predicts the raise outcome for every key.
- No hand-written `if graph.is_directed(): raise` / `if graph.is_multigraph(): raise` block survives in any wrapper body
  (replace-and-sweep grep is clean). 100% coverage, warnings-as-errors hold.

### Workstream B: Up-front conflict validator + decisive error

**Status:** not started
**Files:** `src/hiveplotlib/graph_features/__init__.py` (the validator, called inside `compute_graph_metrics` before any
wrapper runs), plus tests.
**Relationship to Workstream A (do not conflate — amendment A5):** A and B detect different things and produce two
distinct user-facing messages. Workstream A's decorator condenses the **single-metric** guard: one metric vs. the graph
that was actually built → "flip the graph" (or "you cannot build a multigraph for this metric"). Workstream B's up-front
validator handles the **set-conflict** case (e.g. `in_degree` + `triangles` → "these cannot share one graph; split into
two calls"), which the per-metric guard structurally cannot detect because each guard only sees its own metric against
one already-built graph. A does NOT make B redundant, and B does NOT replace A's runtime raise. Both consumers read the
same `_hpl_requirement` record (so the count of consumers is: A's `_enforce_graph_type` guard, B's validator, and the
docs metric-table directive = three consumers, two distinct user-facing messages).
**Done when:**
- A validation pass runs at the top of `compute_graph_metrics` (after `_check_metric_names`, before any metric
  computation), reasoning over the **combined** node+edge requested set by reading each metric's `_hpl_requirement`
  record (via `getattr(GRAPH_NODE_METRICS[m] / GRAPH_EDGE_METRICS[m], "_hpl_requirement", <default>)`) — **not** a
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

**Status:** not started
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

**Status:** not started
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

### Workstream E: Notebook blurb — FINAL, HARD-GATED

**Status:** not started — **DO NOT DISPATCH** until Workstreams A, B, C, and D are all signed off.
**Files:** `examples/computing_graph_metrics.ipynb` (the **only** file). Notebook prose is **Notebook Author's** domain.

> **GATING CONSTRAINT (user was explicit).** Gary is actively editing `examples/computing_graph_metrics.ipynb` right now
> and does not want merge conflicts. This workstream MUST be the last thing dispatched, and ONLY after A, B, C, and D have
> each been signed off. The dispatching session must confirm sign-off of all four before invoking Notebook Author here.
> If any earlier workstream is still open, do not touch the notebook. This is a hard gate, not a soft preference.

**Done when:**
- A short markdown blurb plus a small cell pair is added near the existing graph-type discussion in the notebook,
  showing (a) the conflict error from a `["in_degree", "triangles"]`-style request and (b) the two-call resolution the
  error points to.
- If Workstream C's inference shipped, the blurb mentions that a single unambiguous metric (e.g. `"triangles"`) now
  "just works" without setting `graph_directed`.
- Prose follows the writing-voice rules (no em-dashes, no AI filler, direct and length-disciplined) and the gallery-
  notebook skill conventions.
- `make test-nb` runs the notebook end-to-end clean (the conflict cell must raise in a controlled, expected way, e.g.
  caught in a `try/except` so the notebook executes to completion).
- editorial-critic post-impl review is clean (coherence: right notebook, no dataset/genre drift).

### Workstream F: Guard-message entry-point clarity (single-metric guard names the class-level kwarg)

**Status:** not started — **runs NOW**, after D, before the gated Workstream E (does NOT touch the notebook).
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
hint=None)` decorator on each metric wrapper that (1) attaches a `GraphTypeRequirement` record as `_hpl_requirement`
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
  `getattr(GRAPH_NODE_METRICS[m], "_hpl_requirement", <default>)` across the existing node+edge dicts. Any plan language
  introducing a standalone `GRAPH_METRIC_REQUIREMENTS` global is removed. Workstream B's validator reads `_hpl_requirement`
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
  probe, re-expressed against `_hpl_requirement`.
- **Relationship to Workstream B clarified** (new paragraph in B): the decorator condenses the SINGLE-METRIC guard ("flip
  the graph"); B's up-front validator still handles the SET-CONFLICT case ("split into two calls"), which the per-metric
  guard structurally cannot detect. A does not make B redundant. Three consumers of `_hpl_requirement` (A's guard, B's
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

```
(remaining Plan amendments slots below, append-only as future emergent work surfaces.)
```

## Holdouts

- **None on the per-wrapper guards (changed by amendment A5).** Under the original registry-only framing the hand-written
  guard blocks were a Holdout. Gary's fuller-decorator decision removes that holdout: the guards are genuinely **replaced**
  by the `@requires_graph_type` decorator's centralized `_enforce_graph_type` raise (the single-metric runtime safety net
  for direct-wrapper callers and resolved-flag mismatches is preserved, just relocated from ~20 inline blocks into one
  shared helper). QA's replace-and-sweep should confirm no hand-written guard block survives; Workstream A's
  message-fidelity sweep test is the proof the relocated raise is behavior- and message-equivalent.
- **Docs metric-table directive's empirical probe is a Holdout (re-justified).** `docs/source/_ext/metric_table_directive.py`
  still discovers requirements by probing each wrapper against four graph variants rather than reading `_hpl_requirement`.
  Kept as-is this plan (collapsing it into a reader of `_hpl_requirement` is the flagged future pass under "Patterns this
  replaces"). It is independent and empirical, so it does not break when the decorator lands; not in scope to change here.

## Implementation log

(append-only; one line per completed workstream)

2026-05-29: Workstream A complete. Added `GraphTypeRequirement` dataclass, `_enforce_graph_type` helper, and `@requires_graph_type(*, directed=None, multigraph_ok=True, hint=None)` decorator to `graph_features/networkx/_helpers.py` (decorator attaches a frozen `_hpl_requirement` record via `functools.wraps` and runs centralized enforcement before each wrapper body; `_NO_REQUIREMENT` is the unconstrained default for `getattr` reads). Deleted all ~20 hand-written `if graph.is_directed()/is_multigraph(): raise` guard blocks from `node_metrics.py` (16 wrappers) and `edge_metrics.py` (`bridges` + the 5 link-prediction factory products), applying the decorator instead; components-family cross-references preserved via `hint=`, `onion_layers` decorated `directed=False, multigraph_ok=False` with directed-first enforcement order. Verified raise/message equivalence vs. the pre-refactor guards across all four graph variants for every wrapper (0 mismatches), no surviving inline guard (grep clean), `make format`/`ty` clean, existing 160 graph-metric tests pass. No new global introduced.
2026-05-29: Workstream B complete (source only; tests are test-engineer's). Added `_requirement_for(name)` (reads `_hpl_requirement` off the wrapper across both master dicts, defaulting `_NO_REQUIREMENT`) and `_check_graph_type_conflicts(*, node_metrics, edge_metrics, graph, from_hive_plot)` to `graph_features/__init__.py`. The validator runs at the top of `compute_graph_metrics` after the per-side target+name checks but before any wrapper call, reasoning over the COMBINED node+edge set. Directedness standoff (a `requires_directed is True` metric and a `requires_directed is False` metric together) raises one `ValueError` naming both sides in backticks, stating they cannot share one internal graph (noting `graph.is_directed()`), with resolution text branching on entry point. Multigraph rejection (`graph.is_multigraph()` and a `rejects_multigraph` metric) raises a parallel one-sided error. Added private keyword-only `_from_hive_plot: bool = False` to `compute_graph_metrics` selecting the HivePlot vs. dispatcher-direct message (HivePlot wiring is Workstream C's job). Restructured the dispatcher body so name validation moved up front and the two redundant inner `_check_metric_names` calls were removed; added `assert target_nodes/target_edges is not None` to keep `ty` happy after the up-front target checks. Added a `:raises ValueError:` docstring line for the conflict. `make format`/`ty` clean; existing `tests/graph_features_test.py -m networkx -n 7` 536 passed, 8 skipped.
2026-05-29: Workstream A tests added to `tests/graph_features_test.py` (mirrors the flat `graph_features_test.py` convention; the `_helpers` adapter tests already live there). New section covers: (1) the per-wrapper raise/verbatim-message sweep parametrized over every `GRAPH_NODE_METRICS` + `GRAPH_EDGE_METRICS` key x the four directed/multigraph variants, with expected raise/no-raise and exact message text derived from each wrapper's own `_hpl_requirement` (the natural form), plus a `_graph_variant`/`_expected_message` helper pair; (2) explicit hint assertions: `connected_components` directed-graphs tail, `strongly`/`weakly_connected_components` undirected tail, and `onion_layers` two-axis (directed/multigraph/both) confirming directed-first order; (3) `topological_generations` isolated (needs a DAG; complete-graph directed variant is cyclic and would raise unrelated `NetworkXUnfeasible`, so the sweep skips it and a dedicated test exercises the undirected-raise + DAG-happy-path cleanly); (4) registry-agreement: every metric resolves a `GraphTypeRequirement` (default `requires_directed=None, rejects_multigraph=False` for unconstrained), and `_hpl_requirement` predicts actual raise outcome per variant; (5) decorator/helper unit tests for `requires_graph_type` and `_enforce_graph_type` directly (tri-state directed True/False/None, `multigraph_ok` toggle, `hint` append, `functools.wraps` name/docstring preservation, directed-before-multigraph order, `_hpl_requirement` field values). ~409 new parametrized cases. `pytest -m networkx -n 7`: 536 passed, 8 skipped (the `topological_generations` rows in the two sweeps, by design); `_helpers.py` at 100% coverage (42/42 statements, 0 missing); warnings-as-errors clean; `ruff format`/`ruff check`/`ty` clean. No source touched.
2026-05-29: Workstream B tests added to `tests/graph_features_test.py` (flat layout; new "Up-front graph-type conflict validation" section after the Workstream A decorator tests). 12 new tests + a `_multigraph_fixture` helper: pure-node directedness conflict (via `compute_graph_metrics` and one direct `_check_graph_type_conflicts` call), cross-boundary conflict (`in_degree` node + `bridges` edge proving node+edge share one graph), one-sided multigraph conflict (dispatcher `multigraph=False` branch and HivePlot `graph_multigraph=False` branch), no-conflict pass-through (`triangles`+`clustering`+`bridges` attaches columns), the load-bearing A1/A2 message-branch pair (dispatcher message asserted to NOT contain `graph_directed` and to instruct `to_undirected`/"OWN graph"/"each call needs a graph of the right type"; HivePlot message asserted to contain `graph_directed=True`/`graph_directed=False` and NOT `to_undirected`), the no-wasted-computation spy (monkeypatched `GRAPH_NODE_METRICS` with `functools.wraps` spies preserving `_hpl_requirement`, asserting neither wrapper ran before the raise), and singular/plural agreement (single-metric-per-side "requires", multi-metric-per-side "require", multi-metric multigraph "do not support"/"these metrics"). Pure-edge directedness conflict (scenario 2) is structurally impossible: every guarded edge metric in `GRAPH_EDGE_METRICS` declares `requires_directed=False` (no directed-required edge metric exists), so that case is covered via the cross-boundary scenario and documented in a section comment. `pytest tests/graph_features_test.py -m networkx -n 7`: 548 passed, 8 skipped (pre-existing `topological_generations` sweep skips); `src/hiveplotlib/graph_features/__init__.py` at 100% coverage (132/132 statements, 0 missing) with both message branches, the multigraph branch, singular/plural, and the pass-through all covered; warnings-as-errors clean; `ruff format`/`ruff check`/`ty` clean. No source touched.
2026-05-29: Workstream C tests added (test-engineer). `tests/hiveplot_test.py` (`TestHivePlotNetworkx`): 12 new tests covering the nodes/edges inference path (the only `graph_directed_explicit=False` entry point, so the only one that exercises `_infer_graph_directed`, including the previously-uncovered `concrete.pop()` line) — headline undirected-only `triangles` infers and computes (stash unchanged, asserting inference is per-call), all-directed `["in_degree","out_degree"]` infers True, agnostic-only `degree` keeps default, ambiguous `["in_degree","triangles"]` defers to the WS-B conflict `ValueError` (message asserted to reference `graph_directed`, name both metrics, say "cannot share one internal graph"), explicit `graph_directed` True/False win over inference, `graph_multigraph` never inferred (explicit `True` + `clustering` still conflicts; undirected inference leaves multigraph default), and the `compute_graph_metrics` method path (omitted flag defers to the `_graph_directed_explicit=False` stash and infers; per-call `graph_directed` True/False is explicit). `tests/hiveplot_matrix_test.py` (`TestHivePlotMatrixNetworkx`): 18 new tests (incl. parametrized over `from_partition`/`from_variable_sweep`) mirroring the infer-capable classmethod paths (undirected-only, all-directed, agnostic-only, ambiguous-defers-to-validator, explicit True/False, no multigraph inference) plus the documented KNOWN ASYMMETRY that `HivePlotMatrix.__init__` and `from_tags` (concrete `graph_directed=True` default, no sentinel) stash `_graph_directed_explicit=True` and never infer (so `triangles` raises there absent an explicit `graph_directed=False`), and the matrix method-path inference. ~30 new tests / 40 parametrized cases. Full `pytest tests/ -n 7`: 1387 passed, 8 skipped (pre-existing `topological_generations`); `hiveplot.py` and `hiveplot_matrix.py` at 100% coverage (the C inference lines, gating, explicitness capture, and `_from_hive_plot=True` wiring all covered); warnings-as-errors, `ruff format`/`ruff check`/`ty` clean. No source touched.
2026-05-29: Workstream D complete (docs-engineer). The conflict-raise and inference docstring prose had already been authored inline by the B/C source engineers across all five required sites, so this pass verified accuracy against the shipped behavior rather than re-authoring: dispatcher `compute_graph_metrics` (`graph_features/__init__.py`) carries the two new `:raises ValueError:` entries (directedness standoff + multigraph rejection) plus the amendment-A3 "does not infer graph type; inference is a HivePlot/HivePlotMatrix construction-time convenience" sentence, and does NOT list the private `_from_hive_plot` in the public param block; `HivePlot.__init__` / `HivePlot.compute_graph_metrics` (`hiveplot.py`) document the up-front conflict error, the two-call resolution, and the inference-with-explicit-wins behavior on `graph_directed` + the failure-mode notes on `node_graph_metrics`/`graph_multigraph`; `HivePlotMatrix` mirrors all five methods (`from_partition`, `from_variable_sweep`, `compute_graph_metrics`, `__init__`, `from_tags`) and documents the asymmetry honestly (`__init__`/`from_tags` take a concrete `graph_directed=True`, never infer, and steer users to the `from_*` classmethods or an explicit `graph_directed=False`; the inferring paths are `from_partition`/`from_variable_sweep`). Docs-engineer fixes applied: three docstring lines were 121 chars (trimmed to <=120 at `graph_features/__init__.py:421`, `hiveplot_matrix.py:1128`, `:1484`), and a Sphinx `py:meth reference target not found: __init__` warning in `HivePlotMatrix.compute_graph_metrics` was resolved by demoting the bare `:py:meth:`__init__`` cross-ref to plain ``` ``__init__`` ``` (the file's existing convention; the qualified `from_*` cross-refs resolve fine). `make docs` builds clean with zero warnings; `ruff format`/`ruff check` clean. No new public API surface, so no `docs/source/autodoc/` rst changes and no notebook index entries; no external links added, so no linkcheck. **Duplication risk (flagged, not fixed):** `docs/source/_ext/metric_table_directive.py` independently classifies graph-type requirements by empirically probing each wrapper against four graph variants. It does not read the `_hpl_requirement` record this plan made the single source of truth; it is a separate encoding, out of scope here, flagged so a future pass can consolidate (already noted in "Patterns this replaces" and "Holdouts").
2026-05-29: Workstream C complete (source only; tests are test-engineer's). Added module-level `_infer_graph_directed(node_graph_metrics, edge_graph_metrics, *, default)` + `_as_metric_name_list` helpers to `hiveplot.py` (reads each requested node+edge metric's `requires_directed` via `graph_features._requirement_for`, discards `None`/agnostic; returns the single concrete value when exactly one remains, else `default` so a zero-concrete agnostic set keeps the default and a two-concrete contradictory set defers to Workstream B's validator). Threaded a new `graph_directed_explicit: bool = True` kwarg through `HivePlot._apply_graph_metrics` and `HivePlotMatrix._apply_graph_metrics` / `_apply_init_graph_metrics`; inference runs inside `_apply_graph_metrics` right before `nodes_edges_to_networkx(..., directed=graph_directed)`, only when not explicit. Explicitness captured at each build site BEFORE `_resolve_graph_or_nodes_edges` collapses the `None` sentinel: `HivePlot.__init__`, `HivePlotMatrix.from_partition`, `from_variable_sweep` compute `graph_directed is not None or graph is not None` and stash it as `_graph_directed_explicit`; the `compute_graph_metrics` methods on both classes treat a per-call `graph_directed` as explicit else defer to the stash. `HivePlotMatrix.__init__` / `from_tags` take a concrete `graph_directed: bool = True` (no sentinel, no `graph=`), so they stash `_graph_directed_explicit=True` (always pinned, no inference) — a deliberate, documented asymmetry. Also wired `_from_hive_plot=True` into the `compute_graph_metrics(...)` call inside both `_apply_graph_metrics` methods so the conflict error uses the HivePlot-flavored (`graph_directed`) message. Multigraph inference intentionally not added (asymmetric axis; default `False` always safe). `make format`/`ty` clean; 611 networkx tests pass (8 pre-existing `topological_generations` skips); smoke-confirmed `node_graph_metrics="triangles"` infers undirected on both classes and explicit `graph_directed=True` + triangles raises the `graph_directed`-referencing conflict.
2026-05-29: Workstream F complete (source only; the three sweep reconstructor helpers are the test-engineer's job). Re-opened the WS-A verbatim-locked text (per A6, option C) in the three `_enforce_graph_type` message templates (`graph_features/networkx/_helpers.py:58-80`): each resolution sentence now names BOTH the class-level kwarg and the low-level one. directed-required: "Build the source graph as directed: pass `graph_directed=True` on `HivePlot` / `HivePlotMatrix` initialization, or `directed=True` on `nodes_edges_to_networkx`."; undirected-required: same shape with `graph_directed=False` / `directed=False`; multigraph-rejected: "...as a non-multigraph: pass `graph_multigraph=False` ... or `multigraph=False` on `nodes_edges_to_networkx`." Raise behavior unchanged (same metrics raise on the same directed×multigraph variants, directed-before-multigraph order for `onion_layers` preserved, `hint=` tails still appended by `_enforce_graph_type`). Verified no per-metric wrapper docstring quotes these base strings (grep shows the strings live only in `_helpers.py`). `make format`/`ty` clean; `pytest tests/graph_features_test.py -m networkx -n 7` shows 62 failures, ALL `AssertionError` on the expected-string assertions (the WS-A sweep, the components/onion hint tests, and the decorator/helper unit tests) — zero non-AssertionError failures, raise/no-raise outcomes intact — pending the test-engineer's update of `_expected_directed_required_msg` / `_expected_undirected_required_msg` / `_expected_multigraph_rejected_msg` (`graph_features_test.py:1199-1220`). Added an "Other Changes" CHANGELOG note.
2026-05-29: Workstream F tests updated (test-engineer). Re-aligned the three WS-A message-fidelity reconstructor helpers in `tests/graph_features_test.py` (`_expected_directed_required_msg` / `_expected_undirected_required_msg` / `_expected_multigraph_rejected_msg`, `:1199-1222`) to the clarified post-A6 wording: each now reconstructs the new "pass `graph_directed=...` on `HivePlot` / `HivePlotMatrix` initialization, or `directed=...` on `nodes_edges_to_networkx`" form (and `graph_multigraph=False` / `multigraph=False` for the multigraph-rejected case), matching the shipped `_helpers.py:58-81` source character-for-character (confirmed against source, including the ` / ` spacing and trailing period before any hint tail). No other change needed: every exact-match assertion in the sweep (`test_requires_graph_type_enforcement_matches_requirement`), the `topological_generations` isolated test, the components-family / `onion_layers` hint tests, and the decorator/helper unit tests all route through these three helpers (the standalone `match=` / `in message` checks assert only unchanged fragments, so no inline base-string copies existed to chase). Assertions kept exact-match (no weakening to substring/regex); the components-family hint tails are still asserted appended after the new base text. `pytest tests/graph_features_test.py -m networkx -n 7`: 548 passed, 8 skipped (pre-existing `topological_generations` sweep skips); `_helpers.py` at 100% coverage (42/42 statements, 0 missing); warnings-as-errors clean; `ruff format`/`ruff check` clean. No source touched.
