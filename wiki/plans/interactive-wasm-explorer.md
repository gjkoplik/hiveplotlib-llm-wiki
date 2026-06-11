# Plan: Interactive WASM hive plot explorer

## Goal

A no-install, browser-only app for exploring hive plot construction on the international trade dataset: pick which continents sit on which axes, choose sorting variables (data columns and networkx graph metrics), toggle repeat axes, and switch between static and interactive rendering, with the plot rerendering on every change. Linked from the hiveplotlib docs. Pedagogical payoff: hands-on intuition for the axis/sort/repeat choices that Nollenburg 2023 shows are intractable to optimize automatically, making interactive exploration the validated workflow.

## Alignment (grill)

```
Not yet run — recommended before dispatch for major plans. Run the grill-me skill
or knowingly skip; record each wave below. Route any resulting plan change to
amend-plan (rule 14).
```

## Settled decisions (do not re-litigate)

Settled with the maintainer before this plan was written; recorded here so workstreams don't reopen them.

1. **Framework: Panel.** App authored as a real importable Python module, converted via `panel convert --to pyodide-worker`. marimo (heavier payload, no interactive bokeh), JupyterLite, Shinylive, stlite, and Gradio-lite were surveyed and rejected.
2. **No `nbsite.pyodide` Sphinx directive.** Broken under plain RST since 2023 (panel#5384 / nbsite#288), MyST-only lifecycle, nbsite pins pydata-sphinx-theme<0.17, no non-HoloViz adopters, app code untestable as markdown text (nbsite#242). Instead the converted app is a standalone prerendered page; docs link to it.
3. **Home: separate GitHub repo** (maintainer creates it; proposed name `hiveplotlib-explorer`), own CI, hosted on GitHub Pages, added to hiveplotlib as a submodule for agent context (precedent: `agent-harness/`, `wiki/`, `@hiveplotlib/d3`). Keeps the churning panel/bokeh/pyodide triple and the playwright dependency out of hiveplotlib's release-critical docs builds and 100%-coverage regime; the WASM artifact consumes hiveplotlib from PyPI via micropip regardless.
4. **Version pinning:** the app pins an exact released hiveplotlib version (`hiveplotlib==0.28.x` once 0.28 ships; 0.28 brings the graph-metric sorting variables). Each hiveplotlib release, the maintainer consciously evaluates bumping the pin. App links to stable docs; docs link to the one "latest" explorer.
5. **Dataset: international trade** (shipped in the wheel, 36 KB CSV, 1,158 edges, ~150 countries, continent columns, `export_value` weights, pandas-only load).
6. **Backends: matplotlib + bokeh.** Datashader excluded with cause: numba/llvmlite cannot run in Pyodide (open since 2020), no non-numba fallback; the trade subset is small enough that this costs nothing.
7. **networkx in, scipy out.** Measured Pyodide CDN wire sizes: networkx ~1 MB, scipy +14.9 MB. Consequence: expose only scipy-free metrics; `eigenvector_centrality_numpy` (`src/hiveplotlib/graph_features/networkx/node_metrics.py:466`) and `pagerank` stay off the menu.
8. **Phase one bar: load time, maintainability, testability.** UX polish is a formally deferred phase two (see "Deferred: phase two"). Phase one styling: clean Panel template, hiveplotlib logo, "back to docs" link.

Supporting payload facts: ~29-30 MB compressed total (Pyodide core ~5.6 MB, numpy/pandas/matplotlib ~18 MB, slim panel + bokeh wheels from cdn.holoviz.org ~1 MB each + BokehJS ~0.7 MB, hiveplotlib 0.15 MB, networkx ~1 MB); roughly 5-7 s download at 50 Mbps plus 3-8 s wasm boot; browsers partition HTTP cache by site so first visits always pay full freight. `panel convert` prerenders the UI as a static placeholder while Pyodide boots in a worker; output is plain static files, no special headers needed on Pages.

## Prior ADRs / design docs

No ADRs exist yet. Relevant wiki pages (pre-task sweep complete; nothing on WASM/Pyodide anywhere in the wiki):

- `wiki/wiki/sources/perez-2021-hype.md` — HyPE is abandoned prior art for exactly this kind of interactive hive plot explorer.
- `wiki/wiki/overview.md` (~line 62) — "Interactive HivePlotMatrix" open question; this plan is the single-HivePlot precursor.
- `wiki/wiki/sources/nollenburg-2023-computing-hive-plots.md` — NP-completeness of optimal axis/sort choices; the research justification for interactive exploration.
- `wiki/wiki/entities/hiveplotlib-javascript.md` — existing no-install JS rendering path, rejected here because it cannot recompute sorting/partitioning client-side.
- `wiki/wiki/concepts/hive-plot-matrix.md` — context for the deferred matrix-explorer direction.
- `wiki/wiki/analyses/karate-club-walkthrough.md` — dataset adjudication context behind picking trade over karate club.

## Patterns this replaces

None, this is a net new addition (new repo plus a docs link in hiveplotlib).

## Default justifications

App-level defaults (what a first-time explorer sees before touching anything):

- Axes default `Asia`, `Europe`, `All other continents` (via `axes_order=["Asia", "Europe", None]`): the whole dataset is visible on three axes at first paint; no silently dropped countries, and the collapsed-group choice itself teaches a real library feature.
- Sort default `Total exports` (`export_value` column): a domain quantity a first-time viewer understands with zero graph theory; metric sorts are the discovery, not the entry point.
- `Repeat axes` default on: intra-continent trade is a large share of edges; with repeat axes off those edges are invisible and the first view looks misleadingly sparse. Toggling off then demonstrates what repeat axes buy.
- Renderer default `Static (matplotlib)`: the library default and the cheapest first render after a ~30 MB download; bokeh hover interactivity is the opt-in upgrade.

## Naming audit

- Repo name: `hiveplotlib-explorer` (maintainer confirms at repo creation). Vs. user vocab: ok; "explorer" matches the pedagogical intent and the `@hiveplotlib/d3` ecosystem-repo precedent.
- App title: "Hive Plot Explorer" (the subject is hive plots; "hiveplotlib" appears in the logo and docs link). Vs. user vocab: ok.
- Widget labels: `Axis 1` / `Axis 2` / `Axis 3` continent dropdowns; `Sort nodes by` per axis; `Repeat axes` toggle (hive plot literature term the app is teaching; pair with a one-line help text "draw each group twice to show within-group edges"); `Renderer` with options `Static (matplotlib)` / `Interactive (bokeh)` (users think static vs. interactive, backend names in parentheses for docs cross-reference).
- Metric menu labels: trade-domain vocabulary with the graph term in parentheses ("Trading partners (degree)", "Bridge countries (betweenness)"), not registry keys (`betweenness_centrality`). Final menu in Plan amendment 2.
- Internal module names (`hiveplotlib_explorer.app` etc.) out of scope.

## API usage examples

The user-facing surface is the widget surface plus the hiveplotlib calls each knob state produces. api-critic (planning mode) reviews the widget surface as the API-examples equivalent.

### Proposed (from planner / Orchestrator)

```python
# Example 1: the library call produced by the app's default knob state
# Example data:
import pandas as pd
from hiveplotlib import Edges, HivePlot, NodeCollection
from hiveplotlib.datasets import international_trade_data

data, metadata = international_trade_data(year=2019, hs92_code=8112)
node_data = pd.concat(
    [
        data[["origin_country", "origin_continent"]].rename(
            columns={"origin_country": "country", "origin_continent": "continent"}
        ),
        data[["destination_country", "destination_continent"]].rename(
            columns={"destination_country": "country", "destination_continent": "continent"}
        ),
    ]
).drop_duplicates()
exports = data.groupby("origin_country")["export_value"].sum().rename("export_value")
node_data = node_data.merge(
    exports, how="left", left_on="country", right_index=True
).fillna({"export_value": 0})
nodes = NodeCollection(node_data, unique_id_column="country")
edges = Edges(
    data, from_column_name="origin_country", to_column_name="destination_country"
)

# Call site (default state: Axis 1=Asia, Axis 2=Europe, Axis 3=All other continents,
# Sort nodes by=Total exports, Repeat axes=on, Renderer=Static):
hp = HivePlot(
    nodes=nodes,
    edges=edges,
    partition_variable="continent",
    sorting_variables="export_value",
    backend="matplotlib",
    repeat_axes=True,
    axes_order=["Asia", "Europe", None],
    collapsed_group_axis_name="All Other\nContinents",
)
fig, ax = hp.plot()
```

```python
# Example 2: startup metric precompute (once), then a twiddled knob state
# Example data: nodes / edges from Example 1.
from hiveplotlib.converters import nodes_edges_to_networkx
from hiveplotlib.graph_features import compute_graph_metrics

# App's scipy-free metric menu (label -> GRAPH_NODE_METRICS key); final list
# locked by Plan amendment 2, guarded by WS-B's scipy-absent test.
METRIC_MENU = {
    "Trading partners (degree)": "degree",
    "Export partners (out-degree)": "out_degree",
    "Import partners (in-degree)": "in_degree",
    "Bridge countries (betweenness)": "betweenness_centrality",
}

# Once at app startup: attach every menu metric as node columns so knob changes
# only re-sort, never recompute.
graph = nodes_edges_to_networkx(nodes, edges, directed=True, multigraph=False)
nodes, _ = compute_graph_metrics(
    graph,
    node_metrics=list(METRIC_MENU.values()),
    target_nodes=nodes,
)

# Call site (twiddled state: Axis 1=Africa, sort Africa by Bridge countries (betweenness),
# others by Total exports, Repeat axes=off, Renderer=Interactive):
hp = HivePlot(
    nodes=nodes,
    edges=edges,
    partition_variable="continent",
    sorting_variables={
        "Africa": "betweenness_centrality",
        "Asia": "export_value",
        "All Other\nContinents": "export_value",
    },
    backend="bokeh",
    repeat_axes=False,
    axes_order=["Africa", "Asia", None],
    collapsed_group_axis_name="All Other\nContinents",
)
bokeh_fig = hp.plot()
```

```python
# Example 3: the app module's import surface (what panel convert and the tests consume)
# Example data: none needed; build_app() loads the shipped trade dataset internally.

# Call site (local dev: `panel serve` this module; tests import it directly):
from hiveplotlib_explorer.app import build_app

app = build_app()  # returns a pn.template servable; widgets are plain attributes
app.servable()
```

Feasibility notes (audited against the documented data model): every knob state maps onto `HivePlot.__init__` parameters documented at `src/hiveplotlib/hiveplot.py:1873` (`partition_variable`, dict-form `sorting_variables`, `repeat_axes`, `backend`, `axes_order` with `None` collapse plus `collapsed_group_axis_name`); the precompute uses the documented `compute_graph_metrics` / `nodes_edges_to_networkx` seam. No undocumented conventions needed. One trap: the `hiveplotlib[networkx]` extra pulls scipy (`pyproject.toml:72-75`), so the convert `--requirements` list must name `networkx` directly, never the extra. numba is optional-extra only, so core hiveplotlib runs in Pyodide without it.

### API Critic's take (planning mode)

Reviewed Examples 1-3 and the widget surface, walked as a first-time docs visitor who clicked "Try it live." Library-call verdict: **Agreed with amendments below.** Every knob state maps onto documented surfaces (checked against the `HivePlot` docstring at `src/hiveplotlib/hiveplot.py:1873`, `compute_graph_metrics` at `src/hiveplotlib/graph_features/__init__.py:384`, `nodes_edges_to_networkx` at `src/hiveplotlib/converters.py:50`); data construction is runnable; all seven `METRIC_MENU` keys exist in `GRAPH_NODE_METRICS`.

1. **[must-fix] Define widget behavior for invalid axis-assignment states before WS-B dispatches.** Three free continent dropdowns can reach states the library rejects with `InvalidAxesOrderError`: a continent selected on two axes, or three specific continents chosen out of six with "All other continents" nowhere (`src/hiveplotlib/hiveplot.py:3085` and `:3157`). An uncaught exception in the Pyodide worker reads as a frozen app to this user. Preferred design: swap-on-conflict dropdowns (picking a value a sibling dropdown already holds swaps the sibling to the changer's old value), which makes every reachable state valid by construction, keeps "All other continents" on exactly one axis at all times, and needs no error UI. Whatever WS-B picks, record the mechanism in this plan first.

   **Resolved (2026-06-10):** maintainer accepted as proposed; swap-on-conflict recorded in Plan amendment 1 and Workstream B's done-when.

2. **[must-fix] Seed Louvain in the startup precompute.** `louvain_communities` is stochastic (`src/hiveplotlib/graph_features/networkx/node_metrics.py:658` docstring: pass `seed` for reproducible results); unseeded, the "Louvain community" sort differs across app loads and across playwright smoke-test runs. Preferred amendment to Example 2 (also drops `target_edges`, which is passed but discarded since no edge metrics are requested):

   ```python
   nodes, _ = compute_graph_metrics(
       graph,
       node_metrics=list(METRIC_MENU.values()),
       node_metric_kwargs={"louvain_communities": {"seed": 0}},
       target_nodes=nodes,
   )
   ```

   **Resolved (2026-06-10):** maintainer dropped Louvain from the menu entirely instead of seeding (Plan amendment 2); the `target_edges` drop was applied to Example 2. Amendment 2 also resolves worth-discussing 4 (parallel labels adopted) and moots worth-discussing 6's eigenvector case (metric dropped; the startup-graph test framing is kept for the remaining menu).

3. **[worth-discussing] Repeat-axes default on: agree, justification holds.** Rule 3 cuts toward on: first paint must show the dataset honestly, and hiding intra-continent trade would teach "hive plots drop most of my edges" as lesson one. The toggle teaches in both directions; edges visibly vanishing on toggle-off is at least as legible a reveal as appearing on toggle-on. Cheap reinforcement for WS-B to consider: a "showing X of Y edges" caption so the off state reads as deliberate omission rather than a sparser dataset. Also note somewhere user-visible that the library default is `repeat_axes=False`, so a graduating user isn't surprised.

4. **[worth-discussing] Metric menu labels are internally inconsistent.** "Export partners (out-degree)" and "Import partners (in-degree)" translate to domain vocabulary but their sibling "Degree" doesn't, and a novice can't tell what plain "Degree" adds over the other two. Prefer one parallel shape: "Trading partners (degree)", "Export partners (out-degree)", "Import partners (in-degree)", "Community (Louvain)". Centrality entries can stay literal; they have no clean trade-domain translation and they're the vocabulary the docs teach.

5. **[worth-discussing] Per-axis sort labels should carry the selected continent, not the slot index.** "Sort Asia by" updates the mental model directly; "Sort Axis 1 by" forces a join across two widgets. Bind each sort widget's label to its axis dropdown's current value, including "All other continents" for the collapsed slot (display label without the rendered name's embedded newline).

6. **[worth-discussing] Pin the scipy-absent test to the app's actual startup graph.** Without scipy, `eigenvector_centrality` is pure-Python power iteration and can raise `PowerIterationFailedConvergence` on a real directed graph; "every `METRIC_MENU` entry computes" only locks the guarantee if it runs against the directed trade graph Example 2 builds, not a toy graph. Suggest WS-B's done-when say "computes on the app's startup graph."

7. **[low-confidence] Renderer default: justification accepted, mild lean the other way.** BokehJS is already in the payload (Panel runs on it), 1,158 edges is cheap for bokeh, and hover-to-identify-countries is the most charismatic thing the WASM payload uniquely buys; a Static default means many visitors never see it. Matplotlib-as-default does mirror the library, which is a fair teaching argument. Maintainer's call.

8. **[low-confidence] Verify the default sort doesn't pile nodes at the axis origin.** For `hs92_code=8112` many countries are import-only and get `fillna 0` on `export_value`, which can stack a large share of nodes at the base of each axis on first paint. Check the default render during WS-B (viz-critic's first pass); consider "Total trade" (exports plus imports) if the pile-up is ugly.

**Recurring pattern:** the widget surface is doing invariant-enforcement work the library deliberately leaves to exceptions (axes coverage, sorting-variable dict completeness, metric/graph-type compatibility). The app has no usable exception channel, so every knob interaction must be valid by construction via constrained widget states; treat any reachable library exception as a widget-design bug, not an error-message-wording problem.

### API Critic — post-implementation review

```
Pending — invoke api-critic in post-implementation mode after Workstream B ships
(the widget surface is the user-facing API here, even though it lives in the
explorer repo).
```

## Notebook review

No notebook change.

## External gates

- **G1: GitHub repo exists.** Maintainer creates `hiveplotlib-explorer` (name his call). Blocks Workstreams A and C (deploy halves) and D (submodule add). Workstream B can develop locally before G1.
- **G2: hiveplotlib 0.28 on PyPI.** Blocks Workstream E. The graph-metric knobs exist in the deployed artifact only after the pin lands on 0.28; until then the deployed app may ship with the metric dropdown hidden or the spike-era pin.
- **G3: explorer live on Pages.** Internal gate: Workstream D's docs link goes in only after the URL resolves (linkcheck would otherwise fail).

## Workstreams

### Workstream A: Feasibility spike (end-to-end skeleton)

**Status:** not started (gated on G1)
**Files:** explorer repo: minimal app module, convert invocation, Pages deploy config
**Done when:** a trivial Panel app (trade data loaded, one widget, one matplotlib hive plot) converts via `panel convert --to pyodide-worker --requirements` pinning the latest released hiveplotlib (0.27.x), deploys to the GitHub Pages URL, and a measured cold-cache time-to-interactive plus total payload size are recorded in the Implementation log with an explicit go/no-go call. Target: interactive within ~15 s at 50 Mbps. A no-go halts the plan for maintainer review instead of proceeding.

### Workstream B: App module and unit tests

**Status:** not started (local dev can precede G1)
**Files:** explorer repo: `hiveplotlib_explorer/` package (`app.py` plus support modules), `tests/`, `pyproject.toml`
**Done when:** the full knob surface from "API usage examples" is implemented as an importable module (`build_app()`); axis dropdowns use swap-on-conflict (Plan amendment 1), with a pytest asserting no reachable dropdown state raises `InvalidAxesOrderError`; the metric menu is exactly the four entries in Plan amendment 2; metrics precompute once at startup; pytest covers widget callbacks as plain Python objects (the documented Panel testing pattern); a dedicated CI-reproducible test environment installs hiveplotlib + networkx + panel **without scipy** and asserts every `METRIC_MENU` entry computes on the app's startup graph (the directed trade graph from Example 2), locking the scipy-free guarantee; `panel serve` runs the app locally against the in-repo hiveplotlib checkout or a 0.28 prerelease. api-critic planning-mode review of the widget surface happens before this workstream dispatches; post-impl review after it ships.

### Workstream C: Build, smoke test, deploy, scheduled CI

**Status:** not started (deploy half gated on G1)
**Files:** explorer repo: build script, `tests/ui/`, GitHub Actions workflows
**Done when:** a build script runs `panel convert` with the exact hiveplotlib pin and `--requirements` naming `networkx` directly (never the scipy-pulling extra); a playwright smoke test follows panel's own `test_convert.py` pattern (serve the output dir, wait for the `pn-loading` class to clear with a 90 s timeout, change a widget, assert the plot output updates, flaky max_runs=3); GitHub Actions runs unit + smoke tests on PR, deploys to Pages on main, and a weekly scheduled job rebuilds and smoke-tests (the "confirm it still works regularly" mechanism, budget 1-3 min). All three workflows have green runs.

### Workstream D: hiveplotlib-side integration

**Status:** not started (gated on G1 for the submodule, G3 for the link)
**Files:** hiveplotlib repo: `.gitmodules`, `CLAUDE.md`, `docs/source/README.md`, `CHANGELOG.rst`, possibly `Makefile` (sync target parity with the other submodules)
**Done when:** the explorer repo is registered as a submodule with a short CLAUDE.md note (context for agent workflows, like `agent-harness/` and `wiki/`); the docs landing page (`docs/source/README.md`, pulled into `index.rst`) carries a short "Try it live" link to the explorer with one sentence of framing; CHANGELOG gets a one-line docs entry (released-behavior change: a new user-visible link); `make docs` builds clean and `make linkcheck` passes against the live URL.

### Workstream E: Pin alignment with 0.28

**Status:** not started (gated on G2)
**Files:** explorer repo: requirements pin, build config
**Done when:** the deployed artifact pins `hiveplotlib==0.28.x` from PyPI, the graph-metric sorting knobs are live in the deployed app, and the playwright smoke test passes against the rebuilt artifact. Establishes the per-release ritual: each hiveplotlib release, the maintainer consciously evaluates bumping this pin.

## Deferred: phase two (recorded so it doesn't bleed into phase one)

- Color and edge-kwarg controls (the edge kwarg hierarchy as twiddleable knobs).
- Export capabilities (download PNG / standalone HTML).
- Sphinx-chrome styling mimicry so the app visually reads as part of the docs.
- `--pwa` service-worker caching for repeat-visit load time.
- Edge graph metrics in the knob surface.
- Per-version explorer builds matched to docs versions.
- Interactive HivePlotMatrix explorer (the `wiki/wiki/overview.md` open question; this plan is its precursor).

## Plan amendments

### Amendment 1 (2026-06-10): swap-on-conflict axis dropdowns — In-scope tweak

**Delta:** Workstream B implements the axis dropdowns as swap-on-conflict: selecting a continent a sibling dropdown already holds swaps the sibling to the changer's old value. Every reachable widget state is valid by construction; "All other continents" sits on exactly one axis at all times; no error UI exists.
**Rationale:** resolves planning must-fix 1 (maintainer accepted the critic's preferred design). The Pyodide worker has no usable exception channel, so `InvalidAxesOrderError` must be unreachable, not caught.
**Touched done-whens:** Workstream B (mechanism added).

### Amendment 2 (2026-06-10): metric menu reduced to four entries; Louvain dropped — In-scope tweak

**Delta:** maintainer resolution of planning must-fix 2: drop Louvain entirely (no seeding) and constrain the menu to popular, casual-audience metrics. Final `METRIC_MENU`, with rule-3 justifications:

| Label | Key | Justification |
|---|---|---|
| Trading partners (degree) | `degree` | "Who trades with the most countries" is the first question a non-graph person asks of this data. |
| Export partners (out-degree) | `out_degree` | "Who sells to the most countries," named directly in trade vocabulary. |
| Import partners (in-degree) | `in_degree` | "Who buys from the most countries," the mirror of exports. |
| Bridge countries (betweenness) | `betweenness_centrality` | "Which countries sit on trade routes between others" is the one centrality with a casual story; accessible label per the maintainer's keep condition. |

Dropped: `louvain_communities` (stochastic across loads and test runs; community labels aren't an ordering, so sorting by them has awkward semantics for this audience), `eigenvector_centrality` (without scipy it's pure-Python power iteration with real convergence-failure risk, and "your neighbors' importance" has no casual story), `closeness_centrality` (stable and scipy-free but no crisp trade-domain translation; the menu stays tight). Closeness is the marginal re-add candidate if a fifth entry is ever wanted.

Labels adopt the parallel domain-vocabulary shape from worth-discussing 4, resolving it. Worth-discussing 6's eigenvector convergence concern is moot via the drop; its "test against the app's startup graph" framing is kept for the remaining menu. Example 2 updated in place (menu reduced; `target_edges` dropped from the precompute since no edge metrics are requested, per the critic's note); naming audit updated.
**Rationale:** the app is exploratory and casual, not for the advanced user; Louvain's stochasticity is awkward even seeded, and the seeding need disappears with the drop.
**Touched done-whens:** Workstream B (scipy-absent CI check now ranges over the four-entry menu and computes on the app's startup graph).

### Amendment 3 (2026-06-10): version and enrich the `to_json` payload — Added workstream F (upstream hiveplotlib)

**Delta:** new Workstream F, hiveplotlib-library work (not the explorer repo), covering three maintainer-approved changes to the `to_json` payload:

1. **`schema_version` top-level key** in all three `to_json` payloads: `BaseHivePlot.to_json` (`src/hiveplotlib/hiveplot.py:1769`), `HivePlot.to_json` (`src/hiveplotlib/hiveplot.py:4466`), and `P2CP.to_json` (`src/hiveplotlib/p2cp.py:362`). `HivePlotMatrix` has no `to_json`; no sweep needed there. Proposed value: semver string `"1.0.0"` (the payload's most established consumer lives in the npm ecosystem, where semver is the native versioning vocabulary). Items 2 and 3 are additive and ship in the same release, so the first stamped payload is simply `1.0.0` covering all of it.
2. **Opt-in node metadata.** New `node_data_columns` parameter (default `None`) on `BaseHivePlot.to_json` / `HivePlot.to_json` naming which `NodeCollection.data` columns to include, joined on `unique_id_column` and emitted alongside `unique_id`/`x`/`y` in each axis's `"nodes"` dict, so downstream renderers can drive hover tooltips.
3. **Opt-in edge metadata.** New `edge_data_columns` parameter (default `None`), same shape: named `Edges.data` columns emitted per tag alongside `"ids"`/`"curves"`. This is hover data, distinct from the edge viz kwargs that already pass through.

**Rationale:** the JSON payload is becoming a multi-consumer public API: `@hiveplotlib/d3` v0.2.0 consumes it today, the explorer this plan builds will, and a possible future anywidget-based Jupyter widget (a separate, independently scheduled task; explicitly not scoped into this plan) would too. Versioning before there are three consumers is cheap; after, expensive. Metadata opt-in is what makes downstream hover tooltips possible without each consumer round-tripping back to Python.

**Feasibility audit:** every new read traces to documented data-model surfaces. Node columns: `NodeCollection.data` (pandas-backed, documented) keyed by `unique_id_column`. Edge columns: `Edges.data` per-tag DataFrames, which by documented contract accept arbitrary extra columns beyond `from_column_name`/`to_column_name`; alignment of metadata rows to drawn curves uses the existing `relevant_edges[from_axis][to_axis][tag]` boolean masks that already pair the tag's DataFrame rows with each chunk's `"ids"` array. No undocumented conventions needed.

**Naming audit:** `schema_version` matches the payload's existing snake_case keys (`long_name`, `unique_id`). `node_data_columns` / `edge_data_columns` echo the library's established column vocabulary (`unique_id_column`, `from_column_name`, `partition_variable` taking column names); users think in DataFrame columns, not "metadata." Final names are open to api-critic planning review.

**Default justifications:** both new parameters default `None`, producing a byte-compatible payload modulo the new `schema_version` key: no breakage for `@hiveplotlib/d3` v0.2.0, no payload bloat for users who don't need hover.

**Scope notes:**

- `P2CP.to_json` gets `schema_version` only. Metadata opt-in for P2CP is a **Deferred follow-up**: no known consumer renders P2CP JSON with hover today, and P2CP's payload already carries point ids and tag membership; reopen when a consumer asks.
- The `.. note::` in `HivePlot.to_json`'s docstring (`hiveplot.py:4483-4486`) currently promises node/edge data is *never* included; it must be rewritten to describe the opt-in behavior.
- The anywidget Jupyter widget is referenced as a future payload consumer only; no widget work in this plan.

#### Added workstream F: `to_json` schema version + opt-in node/edge metadata

**Status:** not started (no external gate; independent of Workstreams A-D, but should merge before the 0.28 release so the explorer pin bump in Workstream E and any `@hiveplotlib/d3` adoption see `schema_version` from their first 0.28 payload)
**Files:** `src/hiveplotlib/hiveplot.py` (both `to_json` methods), `src/hiveplotlib/p2cp.py`, `tests/hiveplot_test.py`, `tests/p2cp_test.py`, `CHANGELOG.rst`
**Done when:** all three `to_json` payloads carry a top-level `schema_version` key; `BaseHivePlot.to_json` / `HivePlot.to_json` accept `node_data_columns` and `edge_data_columns` (default `None` leaves the payload unchanged apart from `schema_version`); a column name absent from the corresponding data raises an exception from `src/hiveplotlib/exceptions/`, not a generic Python error; tests cover the default-payload compatibility contract (existing keys unchanged when the new parameters are unset) and the opt-in paths at 100% coverage; docstrings updated, including rewriting the `hiveplot.py:4483` exclusion note; CHANGELOG.rst entries added (`to_json` shipped in v0.27.0, so this changes released behavior; the same-unreleased-version suppression rule does not apply).

##### API usage examples (Workstream F)

```python
# Example data: nodes / edges from this plan's Example 1 (trade dataset).
import json

hp = HivePlot(
    nodes=nodes,
    edges=edges,
    partition_variable="continent",
    sorting_variables="export_value",
    axes_order=["Asia", "Europe", None],
)

# Call site 1: default payload, now stamped.
payload = json.loads(hp.to_json())
payload["schema_version"]  # "1.0.0"

# Call site 2: opt-in hover metadata for a downstream renderer.
payload = json.loads(
    hp.to_json(
        node_data_columns=["continent", "export_value"],
        edge_data_columns=["export_value"],
    )
)
```

##### API Critic's take (planning mode, Workstream F)

```
Pending — invoke api-critic in planning mode on this workstream's API usage
examples before dispatching implementation.
```

##### API Critic — post-implementation review (Workstream F)

```
Pending — invoke api-critic in post-implementation mode after Workstream F ships.
```

## Holdouts

None.

## Implementation log

(empty)
