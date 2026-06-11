# Plan: graph_metric_backend — networkx backend dispatch for graph metrics

## Goal

Users with large graphs can route hiveplotlib's graph-metric computation through networkx's backend dispatch system (nx-parallel, nx-cugraph, graphblas-algorithms, any `nx._dispatchable` backend) via a single `graph_metric_backend` parameter on `compute_graph_metrics()` and the `HivePlot` / `HivePlotMatrix` construction and method paths. Unsupported metric/backend pairs fall back to default networkx with an INFO log line; unknown or uninstalled backend names raise up front. Zero behavior change when the parameter is left at its default.

## Alignment (grill)

### Maintainer shared-understanding pass (grill), Wave 1 — critic pushback on decision 6, release targeting/ADR, HPM gate, extras + networkx floor (2026-06-10)

- **Confirmed: api-critic must-fix 2 accepted.** Explicit per-metric `"backend"` entries on the degree family (`degree`, `in_degree`, `out_degree`) are caught in the up-front validation pass with an actionable message ("direct structural read; backend dispatch does not apply"), not left to a mid-loop `TypeError`. The silent-skip behavior for the global backend stands as settled. This amends settled decision 6's explicit-override half; route to amend-plan.
- **Confirmed: ships in v0.28.0.** Feeds the combined v0.28 close-out ADR (open question 1 resolved: no standalone record). CHANGELOG folds into the existing v0.28 entry per the maintainer's released-behavior rule.
- **Confirmed: Workstream C is GO.** HivePlotMatrix propagation proceeds ("this should definitely be possible for the HivePlotMatrix as well"). Open question 3 resolved.
- **Confirmed: nx-parallel in `dev` extra only** (open question 2 resolved). **Confirmed: no `[networkx]` extra pin**; clear runtime error naming the version requirement only when dispatch is actually requested on an old networkx (open question 4 resolved). Maintainer notes testing that old-networkx error path may be painful and may end up `# pragma: no cover`; testability approach deferred to Wave 2 discussion.

### Maintainer shared-understanding pass (grill), Wave 2 — stored intent, per-metric None semantics, old-networkx testability (2026-06-10)

- **Confirmed: stored-construction-intent mirrors `graph_directed`** (open question 5 resolved). Construction value stored; later `compute_graph_metrics()` calls reuse it unless overridden per call; per-call overrides never mutate the stored value.
- **Confirmed: api-critic must-fix 1 accepted, option (a).** Per-metric `"backend": None` is the documented opt-out: presence-of-key wins, explicit None means default networkx for that metric even under a global `graph_metric_backend`. Requires the sentinel-default pop implementation (a `pop("backend", None) or global` collapse is the bug this guards against); validation skips None values rather than rejecting them. Maintainer initially questioned supporting None at all (instinct: validate strictly against installed backends, scream otherwise); resolved on seeing the single-metric opt-out scenario and that the strict alternative leaves no escape hatch. Global default `graph_metric_backend=None` = default networkx stands; auto-selecting a "faster" backend when unset was considered and rejected (silent install-dependent magic; networkx's `NETWORKX_BACKEND_PRIORITY` already serves that want explicitly).
- **Confirmed: old-networkx error path is tested via monkeypatch** (patch/delete the dispatch-machinery probe on `nx.utils` and assert the raise); `# pragma: no cover` authorized only as fallback if the probe proves genuinely unmockable. Engineer instruction recorded: read the floor version's `_dispatchable` source before writing the except clause; it may need to be a small exception tuple (api-critic low-confidence item 5).

Grill closed 2026-06-10: Wave 2 surfaced no remaining divergence; maintainer aligned. Resulting plan changes routed to orchestrator amend-plan per rule 14.

Top-level design decisions below were settled with the maintainer in discussion before this plan was written; the grill should focus on the open questions listed at the end of "Design decisions", not relitigate the settled ones.

## Prior ADRs / design docs

- `wiki/wiki/plans/networkx-metric-expansion-and-backend-refactor.md` — **critical reconciliation, not a reversal.** Its user resolution item 7 rejected a `backend=` kwarg for the *which-library-wraps-metrics* question (a future igraph integration dispatches on input graph type via isinstance; no parameter needed). `graph_metric_backend` answers a *different* question: which engine networkx dispatches to internally. That engine choice is not inferable from the input type (an `nx.Graph` looks identical whether the user wants nx-parallel or default), so it must be a parameter. `GRAPH_NODE_METRICS` / `GRAPH_EDGE_METRICS` stay metric-keyed, unchanged.
- `wiki/wiki/plans/graph-metric-conflict-validation.md` — shipped. Up-front whole-set validation philosophy (validate everything before any metric runs); decorator requirement records; the per-metric kwargs seam this plan extends; `stacklevel` conventions.
- `wiki/wiki/plans/scaling-large-networks.md` — its naming audit rejected `backend="dask"` for the dataframe layer (narwhals type dispatch, no parameter). See the naming audit's backend-sense triangle.
- `wiki/wiki/plans/edge-coverage-and-gotchas-docs.md` — warnings-rejection principle: fallback to a correct-but-slower computation is not a data-fidelity loss, so it is logging, not `warnings.warn`.
- `wiki/wiki/plans/i-want-to-plan-optimized-hoare.md` — extras-placement precedent; seven-surface kwarg-replication pain motivating the deliberate (own-workstream, maintainer-gated) HivePlotMatrix propagation.
- ADR routing: a standing decision (recorded in two plans) defers ADR promotion to one combined v0.28.0 close-out ADR for the whole networkx story. **RESOLVED 2026-06-10 (grill Wave 1):** this feature feeds the combined v0.28 close-out ADR; no standalone record.
- Coordination: `examples/computing_graph_metrics.ipynb` is touched by the in-flight graph-metrics-notebook-restructure plan; any backend blurb there is a deferred follow-up, not part of this plan (see Workstream E).
- Post-task wiki update: `wiki/wiki/concepts/graph-features.md` needs the third backend concept added (research-liaison, after ship).

## Design decisions (settled with maintainer)

1. **Name:** `graph_metric_backend`, deliberately long to disambiguate from the viz `backend` parameter that coexists in the same `HivePlot.__init__` signature.
2. **Surfaces:** `compute_graph_metrics()` in `src/hiveplotlib/graph_features/__init__.py`, the `HivePlot` construction/method path, and (deliberately, own workstream) `HivePlotMatrix`.
3. **Lenient fallback:** `NotImplementedError` from the dispatch call → fall back to default networkx computation, via try/except around the call. NOT a backend-coverage registry pre-check (registry APIs churn; the exception contract is stabler).
4. **Strict on bad names:** unknown/uninstalled backend names raise at construction/call time, validated up front against networkx's runtime registry of installed backends. No hard-coded allowlist in hiveplotlib.
5. **Precedence:** a per-metric `"backend"` entry in `node_metric_kwargs` / `edge_metric_kwargs` (and the `*_graph_metric_kwargs` HivePlot equivalents) overrides the global `graph_metric_backend`. Specific beats general.
6. **Non-participation (amended 2026-06-10, grill Wave 1 / api-critic must-fix 2):** `degree`, `in_degree`, `out_degree` are direct reads of graph structure (method calls, not `nx._dispatchable` functions), unaffected by `graph_metric_backend`. One documented sentence; a global backend silently skips them (correct result either way). An explicit per-metric `{"degree": {"backend": ...}}` is caught in the up-front validation pass with an actionable error ("direct structural read; backend dispatch does not apply") — NOT left to a mid-loop `TypeError`, which would fire after earlier metrics already computed and contradict the shipped up-front-validation philosophy.
7. **Observability — stdlib logging, net new to the codebase.** Repo convention, stated here once since this introduces logging: module-level `logger = logging.getLogger(__name__)` per module that logs; library code never configures handlers or levels (stdlib last-resort handler only prints WARNING+, so the library is silent by default). INFO line per fallback (metric name, requested backend, "ran on default networkx"). DEBUG line on successful backend dispatch. Not `warnings.warn` (warnings-rejection principle above).
8. **Testing:** nx-parallel is the CI-testable backend (pip-installable, CPU-only), behind a new pytest marker following the existing optional-dep marker pattern. nx-cugraph documented as known-good but GPU-only/untested in CI. 100% coverage and `filterwarnings = error` apply.
9. **Docs framing:** works with any networkx dispatchable backend; tested with nx-parallel in CI; known to work with nx-cugraph (GPU, untestable in our CI); other dispatchable backends should work but are unverified; plumbing-looking breakage → file an issue at https://gitlab.com/geomdata/hiveplotlib/-/issues. Gallery example shows both the `NETWORKX_BACKEND_PRIORITY` env-var route and the parameter, including `logging.basicConfig(level=logging.INFO)` so fallback notices are visible.
10. **Extras decision (RESOLVED 2026-06-10, grill Wave 1):** nx-parallel lands in the `dev` extra only (CI/test dependency; hiveplotlib never imports nx-parallel, and a user-facing extra would imply curation of one specific backend among many).
11. **Per-metric `"backend": None` semantics (added 2026-06-10, grill Wave 2 / api-critic must-fix 1, option a):** presence-of-key wins; an explicit `None` means default networkx for that metric, the documented opt-out under a global `graph_metric_backend`. Requires the sentinel-default pop implementation — a `pop("backend", None) or global` collapse is exactly the bug this guards against. Up-front validation skips `None` values rather than rejecting them.
12. **Three-level precedence chain (api-critic worth-discussing 4, accepted):** per-metric `"backend"` > per-call `graph_metric_backend` > stored construction value. Stated once, in Workstream B's docstring bullet, cross-referenced elsewhere.

**Dispatch mechanism (planner-specified, downstream of the settled semantics).** The `backend=` kwarg must reach the `nx.<func>(...)` call *inside* each wrapper. Wrappers that already take `**kwargs` (e.g. `betweenness_centrality`) need no signature change. The 14 bare-signature wrappers that nevertheless wrap dispatchable nx functions (`core_number`, `degree_centrality`, `in_degree_centrality`, `out_degree_centrality`, `onion_layers`, `reciprocity`, `articulation_points`, `isolates`, `topological_generations`, `label_propagation_communities`, `connected_components`, `strongly_connected_components`, `weakly_connected_components` in node_metrics; `bridges` in edge_metrics) gain an explicit `backend: Optional[str] = None` parameter forwarded to the nx call — their "no networkx algorithm kwargs accepted" contract otherwise stands. `degree` / `in_degree` / `out_degree` stay bare. The central seam in `compute_graph_metrics` resolves the effective backend per metric (sentinel-default pop of `"backend"` from that metric's kwargs dict: key absent → global; key present, even as explicit `None` → the per-metric value, per decision 11), validates all distinct requested backend names up front before any metric runs (skipping `None` values; rejecting degree-family per-metric entries per decision 6), calls the wrapper with `backend=` injected, and on `NotImplementedError` logs INFO and retries without it. Engineer instruction (api-critic low-confidence 5, grill Wave 2): read the floor networkx version's `_dispatchable` source before writing the except clause — it may need to be a small exception tuple. Rejected alternative: wrapping the call in an `nx.config.backend_priority` context — its fallback is silent and automatic, which defeats decisions 3, 4, and 7 (no exception to catch, no strictness, no observability).

**Open questions for the maintainer — ALL RESOLVED 2026-06-10 (grill closed; see "## Alignment (grill)" captures, the authority for these resolutions):**

1. ADR routing — RESOLVED (Wave 1): ships in v0.28.0; feeds the combined v0.28 close-out ADR, no standalone record. CHANGELOG folds into the existing v0.28 entry.
2. nx-parallel extras placement — RESOLVED (Wave 1): `dev` extra only (decision 10). Workstream D ungated.
3. HivePlotMatrix propagation — RESOLVED (Wave 1): GO. Workstream C ungated.
4. networkx version floor — RESOLVED (Wave 1 + Wave 2): no `[networkx]` extra pin; clear runtime error naming the version requirement only when dispatch is actually requested on an old networkx. Error path tested via monkeypatch of the dispatch-machinery probe on `nx.utils`; `# pragma: no cover` authorized only as fallback if the probe proves genuinely unmockable.
5. Stored-construction-intent semantics — RESOLVED (Wave 2): mirror the shipped `graph_directed` pattern exactly (construction value stored; later `compute_graph_metrics()` calls reuse it unless overridden per call; overrides never mutate the stored value).

## Patterns this replaces

- Implicit raw passthrough of a per-metric `"backend"` kwarg: today a user *could* pass `node_metric_kwargs={"pagerank": {"backend": "parallel"}}` and it would reach `nx.pagerank` unvalidated, with no fallback and no logging, via the wrapper `**kwargs` forwarding at `src/hiveplotlib/graph_features/__init__.py:541` (node side) and `src/hiveplotlib/graph_features/__init__.py:597` (edge side). Replace with the governed seam: `"backend"` is popped from per-metric kwargs and routed through the same validation/fallback/logging path as the global parameter.
- The "wrappers without `**kwargs` ... will raise TypeError" docstring enumeration at `src/hiveplotlib/graph_features/__init__.py:444-446` understates the bare-wrapper set and must be rewritten once the 14 dispatchable bare wrappers gain `backend=`; only `degree` / `in_degree` / `out_degree` remain fully bare.

Otherwise net new addition.

## Default justifications

- `graph_metric_backend=None`: default networkx dispatch, byte-identical behavior for every existing user; backends are opt-in acceleration and most users' graphs don't need one.
- New bare-wrapper `backend=None` parameters: same justification, propagated.
- Fallback logging at INFO (not WARNING): a fallback yields a correct result, just slower; the library stays silent by default and users opt into visibility with one `logging.basicConfig` line.

## Naming audit

- New parameter: `graph_metric_backend`. Maintainer's deliberate pick; long on purpose. The codebase now carries three distinct "backend" senses, recorded here so future naming work doesn't collapse them:
  1. **Viz backends** — `backend` parameter on `HivePlot` / viz functions (matplotlib, bokeh, ...). Parameter, user-chosen.
  2. **Dataframe engines** — no parameter; narwhals type dispatch (scaling-large-networks naming audit removed `backend="dask"`). Inferred from input type.
  3. **Graph-metric dispatch engines** — `graph_metric_backend`. Parameter, because the engine is *not* inferable from input type.
  The prefix `graph_metric_` matches the existing `node_graph_metric_*` / `edge_graph_metric_*` family on the same signatures; a user scanning `HivePlot.__init__` sees `backend` (viz) and `graph_metric_backend` (dispatch) and cannot confuse them. OK.
- Per-metric key: `"backend"` inside `node_metric_kwargs` / `edge_metric_kwargs` dicts — exactly networkx's own kwarg name, which is the vocabulary users meet in networkx docs. OK.
- New exception: `InvalidGraphMetricBackendError` in `src/hiveplotlib/exceptions/hive_plot.py` (or a graph-features exceptions module), mirroring the existing `InvalidVizBackendError`. Honors the use-custom-exceptions trip-wire; note the surrounding `graph_features` validation currently raises plain `ValueError` (metric names, type conflicts), so the engineer should confirm one consistent choice with the maintainer if the mirror feels inconsistent in context.
- New pytest marker: `nx_parallel` (hyphen invalid in marker names; underscore form mirrors the package). OK.
- Adjacent non-collisions checked: `n_parallel` (edge-curve CPU parallelism) and `use_numba` already exist on `HivePlot.__init__` (`src/hiveplotlib/hiveplot.py:2111-2112`); `nx_parallel`/nx-parallel prose must not be shortened to "parallel backend parameter" near those. Prose-only terms: "dispatchable backend", "backend dispatch" (networkx's own vocabulary). OK.

## API usage examples

### Proposed (from planner / Orchestrator)

```python
# Example 1: accelerate construction-time node metrics with nx-parallel
# Example data:
import networkx as nx

G = nx.karate_club_graph()

# Call site:
from hiveplotlib import HivePlot

hp = HivePlot(
    graph=G,
    partition_variable="club",
    sorting_variables="betweenness_centrality",
    node_graph_metrics="betweenness_centrality",
    graph_metric_backend="parallel",
)
```

```python
# Example 2: per-metric backend override beats the global parameter
# Example data:
import networkx as nx

G = nx.karate_club_graph()

# Call site:
from hiveplotlib import HivePlot

hp = HivePlot(
    graph=G,
    partition_variable="club",
    sorting_variables="degree",
    node_graph_metrics=["degree", "betweenness_centrality", "pagerank"],
    node_graph_metric_kwargs={"pagerank": {"backend": "graphblas"}},
    graph_metric_backend="parallel",
)
# degree: direct structure read, dispatch does not apply
# betweenness_centrality: nx-parallel (global)
# pagerank: graphblas (per-metric wins)
```

```python
# Example 3: standalone compute_graph_metrics, with fallback visibility
# Example data:
import logging

import networkx as nx

from hiveplotlib.converters import networkx_to_nodes_edges

logging.basicConfig(level=logging.INFO)
G = nx.karate_club_graph()
nodes, edges = networkx_to_nodes_edges(G)

# Call site:
from hiveplotlib.graph_features import compute_graph_metrics

new_nodes, new_edges = compute_graph_metrics(
    G,
    node_metrics=["betweenness_centrality", "core_number"],
    edge_metrics="edge_betweenness_centrality",
    target_nodes=nodes,
    target_edges=edges,
    graph_metric_backend="parallel",
)
# any metric nx-parallel does not implement logs an INFO fallback line and
# computes on default networkx
```

```python
# Example 4: strict failure on a backend that is not installed
# Example data:
import networkx as nx

G = nx.karate_club_graph()

# Call site:
from hiveplotlib import HivePlot

hp = HivePlot(
    graph=G,
    partition_variable="club",
    sorting_variables="degree",
    node_graph_metrics="degree_centrality",
    graph_metric_backend="cugraph",  # raises InvalidGraphMetricBackendError
)                                    # if nx-cugraph is not installed,
                                     # listing the installed backends
```

### API Critic's take (planning mode)

Agreed on the headline shape: the name `graph_metric_backend` (one name across both surfaces beats a shorter `backend` on standalone `compute_graph_metrics`, and the `graph_metric_` prefix lands it in the right tab-complete cluster next to `node_graph_metric_*` / `graph_directed`); the per-metric `{"pagerank": {"backend": ...}}` syntax over a new sugar parameter (reuses the existing kwargs channel and networkx's own kwarg name; a dedicated overrides param would be a seventh replicated surface for marginal gain); INFO-fallback-not-warn; strict up-front validation against the runtime registry. Examples 1-4 are realistic and runnable as written.

Amendments, tagged:

1. **[must-fix] Define per-metric `"backend": None` semantics: presence-of-key wins, explicit `None` means default networkx for that metric.** With a global `graph_metric_backend="parallel"`, the user whose one metric is *slower* under the backend has no opt-out otherwise, and the natural `kwargs.pop("backend", global)` implementation makes an explicit `None` silently collapse into "use global". One sentence in the docstring, one precedence test in Workstream A's done-when.

2. **[must-fix] Disagree with the explicit-override half of settled decision 6 on a ground the settlement didn't weigh: a "natural TypeError" from the bare `degree` wrapper fires mid-loop, after earlier metrics in the list have already computed.** Large graphs are this feature's exact audience, so the wasted computation is maximal, and it contradicts the shipped up-front-validation philosophy (nothing computed and thrown away) this plan itself cites. The seam already special-cases `"backend"` by popping it, so routing a degree-family per-metric `"backend"` entry into the existing up-front validation pass is nearly free and gives a real message ("`degree` is a direct structural read; backend dispatch does not apply") instead of `degree() got an unexpected keyword argument 'backend'`. The silent-skip half (global backend ignores the degree family) stays as settled.

3. **[should-fix] `InvalidGraphMetricBackendError` message quality, two cases:** (a) zero installed backends must not render as "installed backends: []"; special-case to "no dispatchable backends are installed (e.g. `pip install nx-parallel`)". (b) The likely typo is the pip package name, not the registry name (nx-parallel registers as `"parallel"`, nx-cugraph as `"cugraph"`); say so in the message or at minimum the parameter docstring, since Example 4's user typing `"nx-cugraph"` gets an unhelpful "not in installed backends" otherwise.

4. **[worth-discussing] State the full three-level precedence chain once: per-metric `"backend"` > per-call `graph_metric_backend` > stored construction value.** Workstream B's "one place, cross-referenced" docstring bullet should name all three levels in a single sentence; Examples 1-2 only exercise two levels at a time.

5. **[low-confidence] Verify the dispatch exception contract at the networkx version floor before writing the except clause.** Explicit-`backend=` dispatch has raised `NotImplementedError` vs `ImportError` differently across networkx versions, and a backend can implement the function but reject a kwarg combination with its own error; Workstream D's integration test pins current behavior, but Workstream A's `except NotImplementedError` should be written after reading the floor version's `_dispatchable` source (ties into open question 4).

6. **[nit] Examples show only node-side per-metric override;** the gallery notebook (Workstream E) should show or mention that `edge_metric_kwargs` carries `"backend"` identically.

Recurring pattern: the per-metric kwargs dicts are quietly becoming a second API surface — `"backend"` is now a *reserved* key (popped by the seam, not forwarded verbatim like every other key). Document reserved keys explicitly in the `node_metric_kwargs` / `edge_metric_kwargs` docstrings; if a second reserved key ever appears, that is the moment to revisit dedicated parameters.

### API Critic — post-implementation review

**Workstream A (2026-06-10):**

```
Status: propose
API surface reviewed: [compute_graph_metrics(graph_metric_backend=...),
  per-metric "backend" kwargs channel (node_metric_kwargs / edge_metric_kwargs),
  backend= on the 14 dispatchable bare wrappers (node_metrics.py / edge_metrics.py),
  InvalidGraphMetricBackendError, INFO/DEBUG log lines]
Concerns:
  - [should-fix] "How to extend" recipe is stale relative to the dispatch seam —
    at src/hiveplotlib/graph_features/__init__.py:11-18
    Suggested change: add the dispatch decision to the recipe (bare wrapper
    around a dispatchable nx function -> add `backend: Optional[str] = None`
    forwarded via `_backend_kwargs`; `**kwargs` wrapper -> nothing needed;
    direct structural read -> add the name to `_NON_DISPATCHABLE_METRICS`),
    and note that the hand-maintained `_DISPATCHABLE_BARE_NODE_METRICS` list at
    tests/graph_features_test.py:1793 needs the new name, since "automatically
    be tested" currently overpromises on the backend-forwarding axis.
  - [worth-discussing] `:raises:` block omits the `TypeError` the
    `node_metric_kwargs` prose explicitly promises for stray degree-family
    keys — at src/hiveplotlib/graph_features/__init__.py:614-615 (prose) vs
    :648-670 (block)
    Suggested change: add a `:raises TypeError:` line (the :raises: block is
    the load-bearing what-to-catch surface; prose mention is not equivalent).
  - [nit] "No keyword arguments are accepted." now appears twice in the
    `in_degree` / `out_degree` docstrings (lead paragraph + the pre-existing
    "Requires a directed graph" paragraph) — at
    src/hiveplotlib/graph_features/networkx/node_metrics.py:48-56 and :70-78
    Suggested change: drop the duplicate sentence from the older paragraph.
  - [nit] Quoting inconsistency in the degree-family rejection message:
    'backend' in straight quotes while metric names use backticks — at
    src/hiveplotlib/graph_features/__init__.py:362-371
    Suggested change: backtick `backend` for consistency with the rest of the
    message vocabulary.
Test-method-coverage audit: clean (sampled test_compute_graph_metrics_* — all
  call compute_graph_metrics; the parametrized wrapper sweep calls each named
  wrapper; test_bridges_backend_networkx_matches_default calls bridges)
```

Amended-decision verification (all honored): sentinel-default pop via
`_BACKEND_NOT_SET` (`__init__.py:105`, `:325`) with a copy-before-pop
(user kwargs dicts never mutated; regression test at
tests/graph_features_test.py:2268); explicit-`None` opt-out with
presence-of-key-wins (tests at :2017 and :2060); degree-family per-metric
entry rejected up front with the direct-structural-read message, verified to
fire before any metric runs (test at :2148); empty-registry special case never
renders `[]` (test at :1917); registry-vs-pip-name note in both the error
message and the parameter docstring; global backend validated even when every
metric overrides it (`__init__.py:718-720`); old-networkx probe raises only
when dispatch is requested, monkeypatch-tested with no pragma; INFO fallback /
DEBUG dispatch log lines read well cold (metric name + backend name + action,
module-level logger, no handler config). Error-path walkthrough: all four
messages (bad name, degree-family explicit backend, old networkx, empty
registry) name the failing input and a concrete remedy. Explicit
`graph_metric_backend="networkx"` on old networkx raising the >=3.2 error was
checked and is correct (the bare wrappers' nx calls would reject `backend=`
there anyway).

**Workstreams B and C:**

```
Pending — invoke api-critic in post-implementation mode after each of
Workstreams B and C ships (C is a mechanical propagation to a sibling
class and still requires review per mental-model rule 7).
```

## Notebook review

```
Pending — invoke editorial-critic in post-implementation mode after Workstream E
ships the new gallery notebook.
```

Notebook-coherence audit (planner): Workstream E adds a **net-new gallery-genre notebook** (`examples/graph_metric_backends.ipynb`, name negotiable) whose class scope is the graph-metrics feature surface (demonstrated through `HivePlot` and standalone `compute_graph_metrics`), dataset: a generated networkx graph (karate club for mechanics; the notebook-author may add one larger generated graph, e.g. `nx.fast_gnp_random_graph`, if a timing contrast is shown — that is within genre, no real-world dataset added). No existing notebook changes class, genre, or dataset under this plan; the `computing_graph_metrics.ipynb` blurb is explicitly deferred (in-flight restructure plan owns that file).

## Workstreams

### Workstream A: core dispatch seam in graph_features

**Status:** complete (2026-06-10)
**Files:** `src/hiveplotlib/graph_features/__init__.py`, `src/hiveplotlib/graph_features/networkx/node_metrics.py`, `src/hiveplotlib/graph_features/networkx/edge_metrics.py`, `src/hiveplotlib/exceptions/hive_plot.py` (or sibling), `tests/graph_features_test.py` (or the existing mirror), `tests/exceptions_*` mirror as applicable
**Done when:**
- `compute_graph_metrics()` accepts `graph_metric_backend: Optional[str] = None`.
- Up-front validation: all distinct requested backend names (global + any per-metric `"backend"` entries) validated against networkx's runtime installed-backend registry *before any metric runs*, raising `InvalidGraphMetricBackendError` listing installed backends. No hard-coded allowlist. Validation skips explicit per-metric `None` values (decision 11).
- Up-front validation also rejects explicit per-metric `"backend"` entries on `degree` / `in_degree` / `out_degree` with an actionable error ("direct structural read; backend dispatch does not apply") — never a mid-loop `TypeError` (decision 6 as amended). Global-backend silent skip of the degree family unchanged.
- `InvalidGraphMetricBackendError` message quality (api-critic should-fix 3): zero installed backends special-cased to "no dispatchable backends are installed (e.g. `pip install nx-parallel`)", never "installed backends: []"; message or parameter docstring notes that registry names differ from pip package names (nx-parallel registers as `"parallel"`, nx-cugraph as `"cugraph"`).
- Old-networkx handling per open question 4's resolution: runtime error naming the version requirement, raised only when dispatch is requested; tested via monkeypatch of the dispatch-machinery probe on `nx.utils` (`# pragma: no cover` only as authorized fallback if genuinely unmockable). Engineer reads the floor networkx version's `_dispatchable` source before writing the except clause (may need an exception tuple).
- Per-metric `"backend"` popped from `node_metric_kwargs` / `edge_metric_kwargs` via sentinel-default pop (not `pop("backend", None) or global`), overriding the global value; explicit `"backend": None` opts that metric back to default networkx under a global backend. One docstring sentence states the None-opt-out; precedence tests cover both per-metric-wins and explicit-None-opt-out.
- `node_metric_kwargs` / `edge_metric_kwargs` docstrings document `"backend"` as a *reserved* key (popped by the seam, not forwarded verbatim like other keys), per the api-critic recurring-pattern note.
- The 14 dispatchable bare wrappers gain `backend: Optional[str] = None` forwarded to their nx call; `degree` / `in_degree` / `out_degree` untouched and documented with the one-sentence non-participation note.
- try/except `NotImplementedError` fallback around the dispatch call: INFO log (metric, requested backend, fell back to default networkx) then retry without `backend=`; DEBUG log on successful dispatch. Module-level `logging.getLogger(__name__)`, no handler/level config in library code.
- Fallback and bad-name branches covered at 100% *without* nx-parallel installed (monkeypatch/caplog unit tests); the implicit-passthrough docstring at `graph_features/__init__.py:444-446` rewritten.
- `make format`, `make ty`, full test suite green.

### Workstream B: HivePlot threading

**Status:** not started
**Files:** `src/hiveplotlib/hiveplot.py`, `tests/hiveplot_test.py`
**Done when:**
- `HivePlot.__init__` and `HivePlot.compute_graph_metrics()` accept `graph_metric_backend`, threaded through `_apply_graph_metrics` into the Workstream A seam.
- Stored-construction-intent semantics mirror `graph_directed` exactly (open question 5 resolved): stored at construction, per-call override applies to that call only, never mutates the stored value.
- Docstrings cover the parameter, the full three-level precedence chain stated once in a single sentence — per-metric `"backend"` > per-call `graph_metric_backend` > stored construction value (api-critic worth-discussing 4) — the non-participation sentence, and the docs framing of decision 9 (one place, cross-referenced, not repeated verbatim seven times).
- Tests cover construction-path and method-path dispatch and the stored-intent semantics; suite green at 100%.

### Workstream C: HivePlotMatrix propagation

**Status:** not started — UNGATED 2026-06-10 (grill Wave 1: GO)
**Files:** `src/hiveplotlib/hiveplot_matrix.py`, `tests/hiveplot_matrix_test.py`
**Done when:**
- `HivePlotMatrix.__init__`, `compute_graph_metrics()`, `from_partition`, `from_variable_sweep`, `from_tags` accept and thread `graph_metric_backend` through `_apply_graph_metrics` / `_apply_init_graph_metrics`.
- Tests mirror Workstream B's at the matrix surfaces; suite green at 100%.

Rationale for recommending GO despite the seven-surface replication pain: HivePlotMatrix is where metric computation is most expensive (computed once per distinct underlying data group across many cells), so it is exactly where backend acceleration pays; the threading is mechanical and the seam already exists. A NO-GO leaves an asymmetry users will read as a bug. If NO-GO, record as a deferred follow-up instead.

### Workstream D: nx-parallel test infrastructure

**Status:** not started — UNGATED 2026-06-10 (grill Wave 1: `dev` extra only)
**Files:** `pyproject.toml`, `pytest.ini`, `tests/graph_features_test.py` (and/or a dedicated nx-parallel test module mirroring source), `.gitlab` CI config only if the dev-extra route turns out not to reach CI
**Done when:**
- nx-parallel added per the extras decision (recommended: `dev` extra only).
- `nx_parallel` marker registered in `pytest.ini` alongside the existing optional-dep markers; nx-parallel tests carry it (in addition to `networkx` where the module convention requires).
- Marked integration tests: (1) successful dispatch end-to-end through `compute_graph_metrics` with `backend="parallel"` on a metric nx-parallel implements (e.g. `betweenness_centrality`), values equal to the default-networkx result; (2) real fallback on a metric/backend pair that dispatches to nowhere, caplog-asserting the INFO line (if nx-parallel later implements the chosen metric, this test fails visibly and gets re-pointed — acceptable); (3) bad-name raise assertion (unmarked; needs no backend installed).
- `filterwarnings = error` risk handled: if nx-parallel itself emits warnings (joblib/config chatter), add a targeted, commented `ignore:` line to `pytest.ini` following the existing precedent block, never a blanket ignore.
- Full suite green at 100% both with and without nx-parallel installed (coverage of the seam itself must not depend on the marker, per Workstream A).

### Workstream E: docs and gallery notebook

**Status:** not started
**Files:** `examples/graph_metric_backends.ipynb` (new), docs index/toctree files as the docs build requires, `CHANGELOG.md` only per the released-behavior rule
**Done when:**
- New gallery-genre notebook demonstrates: the parameter route, the per-metric override, the `NETWORKX_BACKEND_PRIORITY` env-var route, and the `logging.basicConfig(level=logging.INFO)` one-liner with a visible fallback notice. The notebook shows or mentions that `edge_metric_kwargs` carries `"backend"` identically to the node side (api-critic nit 6). Writing-voice rules apply (no em-dashes, no AI filler).
- Decision 9's support-tier framing appears once in the docs (notebook and/or the `compute_graph_metrics` docstring) with the GitLab issue-tracker link.
- Notebook executes under `make test-nb` (nx-parallel availability in the notebook environment confirmed per the extras decision; if absent, the notebook must demonstrate with what `dev` installs).
- `make docs` builds with no new warnings.
- CHANGELOG: resolved (grill Wave 1) — folds into the existing v0.28 entry; the engineer amends rather than adds. Ships in v0.28.0; feeds the combined v0.28 close-out ADR.
- Deferred (recorded below, not done here): backend blurb in `examples/computing_graph_metrics.ipynb`.

## Plan amendments

### Amendment 1 (2026-06-10): grill close-out — five open questions resolved, two api-critic must-fixes accepted

**Trigger:** grill Waves 1-2 (see "## Alignment (grill)", the authority for all items below) closed with maintainer alignment; api-critic must-fixes 1-2 accepted, should-fix 3, worth-discussing 4, low-confidence 5, and nit 6 all folded in. Routed per rule 14.

All items are **in-scope tweaks** to existing workstreams; no added workstreams, no deferred follow-ups beyond those already recorded below.

1. **Decision 6 amended (must-fix 2):** explicit per-metric `"backend"` on the degree family is rejected in the up-front validation pass with an actionable error, not a mid-loop `TypeError`. Global-backend silent skip unchanged. → decision 6 text, Workstream A done-when.
2. **Decision 11 added (must-fix 1, option a):** per-metric `"backend": None` = documented opt-out to default networkx; presence-of-key wins; sentinel-default pop required; validation skips `None`. One docstring sentence + precedence tests. → decisions list, dispatch-mechanism paragraph, Workstream A done-when. Three-level precedence chain (decision 12, critic item 4) stated once in Workstream B's docstring bullet.
3. **Open question 1:** ships in v0.28.0, feeds the combined v0.28 close-out ADR; CHANGELOG folds into the existing v0.28 entry. → Prior ADRs line, Workstream E done-when.
4. **Open question 2:** nx-parallel in `dev` extra only (decision 10 finalized). Workstream D ungated.
5. **Open question 3:** HivePlotMatrix propagation GO. Workstream C ungated.
6. **Open question 4:** no `[networkx]` extra pin; version-requirement error only on actual dispatch use on old networkx; monkeypatch test of the `nx.utils` dispatch-machinery probe, `# pragma: no cover` only as authorized fallback; engineer reads the floor version's `_dispatchable` source first (critic item 5). → Workstream A done-when, dispatch-mechanism paragraph.
7. **Open question 5:** stored-construction-intent mirrors `graph_directed` exactly. → Workstream B done-when.
8. **Critic should-fix 3 + nit 6 + reserved-key note:** error-message quality (empty-registry special case; registry vs pip names) → Workstream A done-when; edge-side `"backend"` parity shown in the gallery notebook → Workstream E done-when; `"backend"` documented as a reserved key in the metric-kwargs docstrings → Workstream A done-when.

### Deferred follow-up: backend blurb in computing_graph_metrics.ipynb

**Date:** 2026-06-10
**Trigger:** coordination note from research-liaison pre-task pass (file is owned by the in-flight graph-metrics-notebook-restructure plan)
**Target:** after the notebook-restructure plan ships; coordinate as a small addition there or a follow-up tweak here
**Rationale:** two plans editing one notebook concurrently invites merge pain; the new gallery notebook carries the teaching load meanwhile.

### Deferred follow-up: wiki concept page update

**Date:** 2026-06-10
**Trigger:** research-liaison pre-task pass
**Target:** post-task research-liaison update to `wiki/wiki/concepts/graph-features.md` introducing the third backend sense (and the naming-audit triangle)
**Rationale:** wiki curation is post-ship work, not a workstream of this plan.

## Holdouts

- `src/hiveplotlib/hiveplot.py:2087` (`backend` viz parameter) and all viz-module `backend` usages: kept as `backend` — different sense (viz), per the naming-audit triangle. Replace-and-sweep greps for backend-related patterns must not flag these.

## Implementation log

(append-only; one line per completed workstream)

2026-06-10: Workstream A complete. `compute_graph_metrics(graph_metric_backend=...)` added with up-front validation of all distinct backend names (new `InvalidGraphMetricBackendError`; degree-family per-metric `"backend"` rejected with the direct-structural-read message; empty-registry and registry-vs-pip-name message cases), sentinel-default per-metric `"backend"` pop (explicit-None opt-out), `backend: Optional[str] = None` forwarded on the 14 dispatchable bare wrappers via a shared `_backend_kwargs` helper, `NotImplementedError` fallback (verified against the networkx 3.6.1 `_dispatchable` source) with INFO-fallback/DEBUG-dispatch logging via module-level `logging.getLogger(__name__)`, old-networkx probe on `nx.utils.backends` raising only when dispatch is requested (monkeypatch-tested, no pragma needed), stale bare-wrapper docstring enumeration rewritten + reserved-key docs; 15 new tests (plus a parametrized 13-wrapper `backend="networkx"` equality sweep), suite 1300 passed at 100% coverage without nx-parallel, ruff/ty green. CHANGELOG amendment deferred to Workstream E per its done-when.
