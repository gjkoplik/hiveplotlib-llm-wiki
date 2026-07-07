# Plan: anywidget interactive hive plot widget

## Goal

An `anywidget`-based `HivePlotWidget` that renders an interactive hive plot from a built `HivePlot`, in live Jupyter sessions and, critically, **embedded in the static Sphinx docs with no running kernel and no backend server**: all data embedded at build time, all interactivity client-side JS, built on `HivePlot.to_json()` plus the `@hiveplotlib/d3` renderer. The kernel-free docs embed is the hard gate: Workstream 0 is a throwaway feasibility spike and the **only workstream authorized to run now**; every implementation workstream (WS-1+) is gated on the spike's written verdict, a maintainer grill, and an explicit go.

Brief-mode gate: knowingly skipped at the maintainer's explicit instruction (2026-07-06).

## Alignment (grill)

```
Status: deferred by maintainer decision (2026-07-06)
- grill-me brief mode: knowingly skipped (maintainer's explicit instruction).
- Post-plan grill: knowingly skipped for the WS-0 spike only. The maintainer
  grills before any implementation workstream (WS-1+) dispatches; that grill
  (including its failure-mode wave) is a hard gate alongside the WS-0 verdict
  and an explicit go. See "Open questions for the pre-implementation grill".
- The adversary's cold planning-mode challenge still runs on this plan
  (mandatory, rule 18), as does api-critic planning mode; neither is skipped.
Record grill waves below when the pre-WS-1 grill runs.
```

## Failure modes

```
Not yet named — the grill is deferred until after the WS-0 verdict; its
failure-mode wave populates this before any WS-1+ dispatch. Each mode one
line, in the maintainer's words. Consequence meanwhile: no auto-dispatch
(rubric-free plan), which is moot because WS-0 ends in a mandatory HALT.
```

## Prior ADRs / design docs

No binding ADRs: only 0001 (networkx-integration) and 0002 (performance-regression-harness) exist; neither touches this space. The binding-decision-like material lives in the WASM explorer plan (maintainer-settled, never promoted):

- `wiki/wiki/plans/interactive-wasm-explorer.md` — the sibling plan; see "Relationship to the interactive WASM explorer plan" below. Its Settled decision 2 rejects the `nbsite.pyodide` Sphinx directive specifically, **not** docs-embedded interactivity as such; anywidget's JS-only-at-view-time route is a different mechanism and is not foreclosed. Its Settled decision 3 (separate repo, keep churn out of hiveplotlib's release-critical docs build) is precedent for the where-does-it-live decision, though anywidget's dependency profile (pure Python, ipywidgets-based) is far lighter, so this plan proposes in-repo; the grill weighs it.
- Explorer plan **Amendment 3 / Workstream F** — `schema_version` + opt-in `node_data_columns`/`edge_data_columns` on `to_json`; designed, **not shipped** (zero `schema_version` matches in `src/`). Its rationale names this widget explicitly as a future payload consumer. See "Sequencing vs the explorer plan's Workstream F" below.
- `wiki/wiki/entities/hiveplotlib-javascript.md`, `wiki/wiki/sources/hiveplotlib-javascript.md` — **stale (dated 2026-04-06)**: they record `@hiveplotlib/d3` v0.2.0; the local checkout (`/home/garyk/repos/hiveplotlib-javascript`) is at **v0.3.0** with a `<hive-plot>` web component (`./element` export; width/height attributes defaulting 550; a `data` property taking an in-memory parsed payload; `hive-plot-rendered`/`hive-plot-error` events; no shadow DOM; CSS-targetable data attributes), esbuild-minified ESM bundles (`*.min.js`, D3 external via `https://cdn.jsdelivr.net/npm/d3@7/+esm`), and consumption of the current three-key payload including `node_viz_kwargs`. WS-0 re-verifies the npm-published version rather than trusting either source.
- `wiki/wiki/overview.md` (~line 71) — "Interactive HivePlotMatrix" standing open question. **This plan does not address it**: `HivePlotMatrix` has no `to_json`, and matrix-scale payloads are unscoped here. The plan addresses the narrower "live hive plot in the docs" value for single `HivePlot`s.
- `wiki/wiki/sources/perez-2021-hype.md` — HyPE (2021, abandoned): Python precomputes, D3 renders, output a self-contained interactive HTML page with no server. That architecture matches this widget's shape more closely than the explorer's Pyodide shape.
- `wiki/wiki/sources/nollenburg-2023-computing-hive-plots.md` — NP-completeness motivation for interactive exploration, with the caveat that a kernel-free widget only "explores" across pre-baked configurations; recompute-driven exploration remains the explorer's territory.
- `wiki/wiki/entities/hiveplotlib.md` — names hiveplotlib-javascript as the `.to_json()` render path; the v0.28 "Exporting to JSON" gallery page (`examples/exporting_hive_plots_to_json.ipynb`) is the natural docs neighbor for the widget page.
- Wiki gaps: no page on anywidget/ipywidgets/traitlets, nothing on Sphinx+widgets or kernel-free embedding beyond the `nbsite.pyodide` rejection. WS-0's verdict is new ground worth capturing (candidate ADR/analysis material once the work ships).

## Relationship to the interactive WASM explorer plan

**Complement, not supersede.** The wiki's explicit position (explorer plan Amendment 3) carves this widget out as a separate sibling consumer of a shared `to_json` payload, and the two artifacts have disjoint core capabilities: the explorer **recomputes** (re-runs real hiveplotlib in Pyodide: re-sorting, re-partitioning, repeat-axis toggles), while this widget can only **re-render precomputed geometry** client-side; the explorer plan itself rejected hiveplotlib-javascript for exactly that reason.

The honest overlap: both target the docs visitor who wants to touch a live hive plot. This widget, with precomputed payloads embedded at build time, covers a meaningful subset of the explorer's demo value at a tiny fraction of its ~30 MB payload, embedded directly in docs pages rather than one link away. The explorer is entirely unstarted (gate G1 open, grill not run), so once the widget exists the maintainer may deliberately revisit the explorer's priority. That revisit is a maintainer decision recorded here as a post-ship trigger, not a scope claim by this plan.

## Sequencing vs the explorer plan's Workstream F (`to_json` schema work)

Decision, recorded per the dispatch brief:

- **WS-0 proceeds now against today's unversioned three-key payload.** The spike needs no schema stamp and must not block on the other plan.
- **This plan does not absorb WS-F.** It stays homed in the explorer plan, where it is fully specced (API examples, critic placeholders, done-when) and independently dispatchable with no external gates. Absorbing it would duplicate the spec and fork its critic/amendment history.
- **WS-1 soft-depends on WS-F:** recommended sequencing is WS-F ships first (it is small and unblocked), so the widget, which makes `@hiveplotlib/d3` consumption load-bearing inside hiveplotlib's own docs, is born against a stamped payload. If WS-F is still unshipped at WS-1 time, WS-1 may proceed on the unversioned payload without absorbing WS-F, since the widget's Python and JS halves version-lock within one release; both should then ship in the same release. Grill topic.
- **WS-3 (hover metadata) hard-depends on WS-F** items 2-3 (`node_data_columns`/`edge_data_columns`): those are the designed-but-unbuilt mechanism for tooltip metadata, and this plan will not invent a parallel one.

## Patterns this replaces

None — net new addition (new module, new optional extra, new docs notebook). The `HivePlot.to_json` docstring note at `src/hiveplotlib/hiveplot.py:5115-5118` (promises node/edge data is never included) gets rewritten by explorer-plan WS-F, not by this plan.

## Default justifications

- `width=550`, `height=550` (px): matches the `<hive-plot>` element's own defaults in `@hiveplotlib/d3` v0.3.0, keeping the Python and JS surfaces consistent; a square frame suits the radial geometry.
- Payload snapshot at construction (the widget serializes `hive_plot.to_json()` once, at `HivePlotWidget(...)` time): the kernel-free docs embed can only ever be a build-time snapshot, so live sessions get identical semantics instead of a live-sync behavior that silently stops working when the kernel goes away. Re-render by constructing a new widget.
- `hiveplotlib[widget]` optional extra rather than a core dependency: anywidget only earns install weight for notebook users; core library users pay nothing.
- WS-3 only: `node_data_columns=None`, `edge_data_columns=None`, mirroring WS-F's designed `to_json` defaults; no hover metadata means no payload bloat.

## Naming audit

Adjacent ecosystem vocabulary: Jupyter widgets / anywidget / ipywidgets.

- New class `HivePlotWidget`: the ecosystem uses both bare nouns (`ipyleaflet.Map`) and `*Widget` suffixes (anywidget's own docs); the suffix wins here because the bare noun collides with `HivePlot` itself. Ok.
- New module `hiveplotlib.widget` (`from hiveplotlib.widget import HivePlotWidget`): deliberately **not** under `hiveplotlib.viz`, whose backend modules implement the shared function interface (`axes_viz`/`node_viz`/`edge_viz`/`hive_plot_viz`) that a widget class does not fit. Flag for api-critic.
- New extra name `widget`: proposed over `jupyter` ("Jupyter widget" is the ecosystem term, and anywidget also runs in VS Code, Colab, and marimo, so `jupyter` overclaims). api-critic / grill call.
- New parameters `width` / `height` (int px): match the `<hive-plot>` element attribute names. Ok.
- New notebook filename `interactive_hive_plot_widget.ipynb`: parallel to existing `exporting_hive_plots_to_json.ipynb` style. Ok.
- Prose-only terms: "widget", "kernel-free", "client-side interactivity", "embedded at build time".
- Internal names (ESM file name, traitlet names) out of scope.

## API usage examples

The eventual user-facing widget surface (WS-1+, gated). api-critic reviews in planning mode next.

### Proposed (from planner / Orchestrator)

```python
# Example 1: render an interactive hive plot of the karate club network in a live notebook
# Example data:
import networkx as nx

from hiveplotlib import HivePlot

g = nx.karate_club_graph()
hp = HivePlot(
    graph=g,
    partition_variable="club",
    sorting_variables="degree",
    node_graph_metrics="degree",
    repeat_axes=True,
)

# Call site (last expression of a cell; displays as the cell output):
from hiveplotlib.widget import HivePlotWidget

HivePlotWidget(hp)
```

```python
# Example 2: a larger render for a presentation notebook
# Example data: hp from Example 1.

# Call site:
HivePlotWidget(hp, width=800, height=800)
```

```python
# Example 3 (gated on explorer-plan Workstream F shipping): hover tooltips from node metadata
# Example data: hp from Example 1.

# Call site:
HivePlotWidget(hp, node_data_columns=["club", "degree"])
```

The docs-page usage is Example 1 verbatim inside `examples/interactive_hive_plot_widget.ipynb`; the kernel-free embed is a property of the docs build (WS-0's subject), not extra API surface.

Feasibility audit: `hive_plot` traces to the documented `HivePlot` class; the widget internally calls the documented `to_json()` (`src/hiveplotlib/hiveplot.py:5098`). `graph=`, `partition_variable`, `sorting_variables`, `node_graph_metrics`, `repeat_axes` in the example data are all documented `HivePlot.__init__` parameters on the working branch (`src/hiveplotlib/hiveplot.py:2636-2671`). `width`/`height` trace to the `<hive-plot>` element's documented attributes. `node_data_columns`/`edge_data_columns` trace to explorer-plan WS-F's designed parameters, which are **not yet in src** — that is exactly the recorded WS-3 hard dependency, not an unauthorized convention.

### API Critic's take (planning mode)

Snippets independently verified against `HivePlot.__init__` on this branch (`src/hiveplotlib/hiveplot.py:2636-2671`): all five example-data kwargs exist, `graph` is keyword-only and Example 1 passes it by keyword (dodging the positional-graph guard at `hiveplot.py:2677`), and the construction matches `examples/karate_club.ipynb`'s "Let's build the hive plot" cell minus styling kwargs. Examples 1-3 as written: **Agreed.** `width`/`height` as int pixels: **Agreed** (matches the `<hive-plot>` element and the px precedent already inside hiveplotlib's bokeh/plotly backends).

The two flagged calls:

- **Module home: `hiveplotlib.widget`. Agreed.** Every `viz` module is a backend implementing the shared function interface, so a class-shaped module would be the one sibling breaking the pattern; the top-level home also makes extra, module, and class one noun (`pip install hiveplotlib[widget]` then `from hiveplotlib.widget import HivePlotWidget`), the exact string the guarded-import error teaches.
- **Extra name: `[widget]` over `[jupyter]`. Agreed.** Same one-noun coherence, and anywidget runs in VS Code / Colab / marimo, so `jupyter` overclaims.

Concerns:

- [must-fix] No constructor input guard in WS-1's surface. `P2CP.to_json` exists (`src/hiveplotlib/p2cp.py:362`) and emits a two-key payload the renderer can't draw, so `HivePlotWidget(my_p2cp)` fails client-side in the browser console, where the widget has no Python exception channel (blank widget, no traceback); `HivePlotWidget(my_hpm)` dies on a bare `AttributeError`. Suggested change: WS-1 validates the input is a `BaseHivePlot` (covers the manual workflow, which also has `to_json`, `src/hiveplotlib/hiveplot.py:2309`) and raises a recovery-oriented error naming what's supported; add to WS-1's done-when.
- [worth-discussing] Construction inherits `to_json`'s `TypeError` on lazy (Dask-backed) edge subsets (`src/hiveplotlib/hiveplot.py:2284-2304`). Snapshot-at-construction surfaces it at the right moment (a Python error in a live kernel, not a blank widget), but the widget's `:raises:` block must list it; a prose mention is not equivalent for the user asking "what does this throw".
- [worth-discussing] No payload-scale guidance. The snapshot embeds `num_steps_per_edge` (default 100) discretized points per edge, so payload size and SVG element count scale linearly with edge count, and a user coming off the large-network work will hand it a browser-killer. Suggested change: one docstring sentence scoping the widget to small/medium networks (datashader stays the big-network route); whether a size warning is also wanted is a grill call.
- [worth-discussing] Snapshot default: agreed as justified, on the condition the class docstring's first paragraph states it ("state is frozen at construction; re-render by constructing a new widget"), because the stale path (`w = HivePlotWidget(hp)`, mutate `hp`, re-display `w`) fails silently. A `refresh()` (grill question 5) is nice-to-have, not a WS-1 requirement.
- [worth-discussing] WS-3 names: agreed on mirroring WS-F's `node_data_columns` / `edge_data_columns` (pass-through transparency beats coining `hover_*` synonyms), on the condition each param's docstring leads with the user's word: "shown as hover tooltips". The mechanism name never says hover, and hover is what the user scanning the signature is looking for.
- [low-confidence] WS-2's done-when doesn't name the extras install banner (`hiveplotlib[widget]`, plus `[networkx]` for the example data) that `examples/karate_club.ipynb` sets the precedent for; the notebook skills likely cover it, named here so it isn't dropped.

Recurring pattern: after construction the widget has no Python exception channel (rendering is client-side JS; failures land in the browser console as a blank or frozen widget), so every reachable failure must either surface at construction time as a legible Python error or be designed out. The first two concerns are both instances; WS-1 should treat "the constructor raises early and legibly" as the design rule, not a per-case patch.

### API Critic — post-implementation review

```
Pending — invoke api-critic in post-implementation mode after Workstream 1 ships.
(Not applicable to Workstream 0: the spike is throwaway and ships no API.)
```

## Notebook review

```
Pending — invoke editorial-critic in post-implementation mode after Workstream 2
ships examples/interactive_hive_plot_widget.ipynb. No notebook in WS-0/WS-1.
```

## Viz review

```
Pending — invoke viz-critic in post-implementation mode after Workstream 2 ships
(the widget's rendered output in the docs page). WS-0's spike renders are
throwaway evidence artifacts, not reviewed figures.
```

## Adversary review

### Adversary's challenge (planning mode)

```
Status: challenge (7 items)
Plan reviewed: wiki/wiki/plans/anywidget-hive-plot-widget.md (cold; grill knowingly
deferred until post-WS-0, so this challenge is the only dissent on record before
the spike runs)
Items:
  - [existential-must-fix] The plan's load-bearing noun "interactive" is supplied by
    nothing in scope, and the docs already ship kernel-free interactive hive plots
    — at Goal / WS-1 done-when / open question 2
    Rubric: no rubric yet — pre-grill
    Evidence: @hiveplotlib/d3 v0.3.0 registers zero pointer handlers (no `.on(`,
    no click/hover/zoom anywhere in hive_plots_d3_viz.js or hive_plot_element.js;
    the element's only events are the hive-plot-rendered / hive-plot-error
    lifecycle dispatches at hive_plot_element.js:121,151). It is a static-SVG
    renderer with CSS-targetable attributes. Meanwhile examples/hover_info.ipynb
    is an entire existing docs page whose committed bokeh outputs render live,
    hover-enabled hive plots with no kernel today. WS-1 ships render-only; WS-3
    (tooltips) is double-gated on unshipped WS-F plus renderer-side work this
    plan explicitly scopes out to the JS repo.
    Push: as scoped, the shipped docs artifact is a static SVG delivered through
    a new module, a new extra, anywidget, and a multi-link view-time CDN chain:
    strictly less interactive than the existing bokeh docs page, and no more
    capable than a build-time SVG image (or a raw <hive-plot> embed, which needs
    no anywidget at all for the docs-embed half; anywidget is only earned by the
    live-notebook goal). Before any WS-1 dispatch the maintainer must answer:
    what does this widget do in the docs that hover_info's bokeh figure does not,
    and which in-scope work supplies it? If the answer requires renderer-side or
    widget-side interaction JS, scope it explicitly (amend WS-1 or add a
    workstream); if no answer survives, WS-1+ should not exist and the plan
    reduces to WS-0's knowledge value. Grill question 2 currently frames this as
    MVP polish; it is the plan's existence question.
  - [must-fix] Route (a) cannot transfer to the publishing pipeline, and the spike
    as designed will report that it can — at WS-0 "The pipeline being mirrored" /
    verdict grammar / WS-2 done-when
    Rubric: no rubric yet — pre-grill
    Evidence: the published docs are built by Read the Docs, not
    build_sphinx_docs.sh: .readthedocs.yaml does its own notebook copy in
    pre_build, pip-installs extras, registers no jupyter kernel, and sets
    fail_on_warning: true; docs/source/conf.py:129 pins nbsphinx_kernel_name =
    "hiveplotlib". An output-free notebook forces build-time execution on a
    kernel that does not exist there (NoSuchKernel, build failure). The spike's
    step 1 registers its own venv kernel, so route (a) passes in /tmp while
    being unshippable on RTD without RTD-config work that appears nowhere in
    the plan.
    Push: change the protocol BEFORE the spike runs. Name RTD as the pipeline
    the verdict must transfer to; demote route (a) to diagnostic-only (or add
    an explicit per-route RTD-transferability line to the verdict block); make
    WS-2's "whichever the verdict endorsed" read "route (b), unless the plan
    gains an RTD-config amendment".
  - [must-fix] The spike builds without the real pipeline's gates, so a PASS
    overstates transfer — at WS-0 protocol steps 2 and 5
    Rubric: no rubric yet — pre-grill
    Evidence: RTD builds with fail_on_warning: true and local builds use -W -n
    (build_sphinx_docs.sh); the spike's minimal project builds bare, and its
    two-extension conf mirrors "the settings that matter" by judgment while the
    real conf.py also loads sphinx_copybutton, autodoc, intersphinx, and an
    in-repo custom directive. One new nbsphinx/myst warning from the widget
    page fails the real build after a green spike.
    Push: build the spike with -W -n (one-line change to step 5), and either
    mirror the real conf.py settings wholesale minus repo-coupled bits or
    record the divergence list in the verdict so the grill sees what was not
    mirrored.
  - [must-fix] The verdict grammar never collects the evidence two grill questions
    are promised — at WS-0 step 4 / verdict block / open questions 1-2
    Rubric: no rubric yet — pre-grill
    Evidence: the plan says "WS-0's renderer check informs what exists today"
    (question 2), but step 4 and the verdict block require only a rendered SVG,
    element counts, npm version, and CDN URLs. Nothing records what interaction
    the rendered hive plot actually supports, and nothing records whether each
    CDN URL is version-pinned or floating (question 1's facts).
    Push: add to step 4 a playwright hover/click pass over rendered .node/.edge
    elements recording events observed and any DOM response, landing as an
    explicit "interaction inventory" line in the verdict block; extend the CDN
    inventory to pinned-vs-floating per URL. This is also the evidence the
    existential item above gets decided on; without it the grill decides on
    vibes.
  - [worth-discussing] Explorer settled decision 3 cuts against the in-repo home
    harder than the plan admits — at "Relationship to the interactive WASM
    explorer plan" / open question 1
    Rubric: no rubric yet — pre-grill
    Push: decision 3's recorded rationale is keeping churning JS out of the
    release-critical docs build, not install weight, and this plan couples the
    published docs' view-time rendering to a v0.x npm package plus the
    ipywidgets embed chain, with linkcheck blind to JS and no tripwire that
    notices a silently broken widget on the published site. Frame vendoring the
    esbuild-minified @hiveplotlib/d3 bundle (question 1) as the default pending
    a named reason against, and name who or what notices view-time breakage
    either way.
  - [worth-discussing] Route (b) has a silent-degradation trap the plan never
    names — at WS-2 done-when
    Rubric: no rubric yet — pre-grill
    Push: saved widget state lives in notebook metadata and is dropped by any
    future re-save without store-widget-state; the docs then build green with a
    hole where the widget was, in a repo whose notebooks are re-executed and
    re-committed routinely. WS-2 needs a build-artifact tripwire (assert the
    built HTML contains the widget-state script; a grep is legitimate there as
    a regression check even though it is not spike-grade evidence) and a stated
    notebook re-execution workflow for the widget page.
  - [low-confidence] The spike could be smaller — at WS-0 protocol
    Rubric: no rubric yet — pre-grill
    Push: the decisive core is one route-(b) notebook plus the renderer notebook,
    built under the real gate flags; the counter-widget pair and route (a) are
    diagnostic decomposition. Keep them if failure isolation is wanted; trim if
    spike budget matters. The route-(a) item above already removes it from the
    verdict's critical path.

Verified and holding (not manufactured into objections): pyproject.toml:62
(ipywidgets in docs extra), the conf.py claims (nbsphinx at 36-43, kernel at
129, npmjs linkcheck-ignore at 191), the .axis/.node/.edge pass-criterion
selectors match the real renderer classes (hive_plots_d3_viz.js:329/409/469),
example_hive_plot exists (datasets/toy_hive_plots.py:360), and the WS-F
characterization matches the explorer plan. Three-angle summary: premise is
the weak leg (existential item); approach and sizing are conditionally sound
once the interaction question is answered; the complement-not-supersede
framing vs the explorer is honest as recorded.
```

### Adversary post-impl

```
Status: propose
Artifact reviewed: WS-0 kernel-free docs-embed feasibility spike (verdict block
  + evidence in /tmp/anywidget-spike/; code throwaway, verdict durable)
Dispositions held: yes. Planning items 2/3/4 were mandated pre-spike and all
  three show up honored in the shipped artifact: RTD named as the pipeline the
  verdict transfers to and route (a) demoted to diagnostic-only (item 2); the
  spike built under `-W -n` with a recorded conf-divergence list (item 3); the
  interaction inventory and per-URL pinned-vs-floating CDN status both present
  in the verdict (item 4). Item 6's WS-2 tripwire and item 5's vendoring framing
  are gated future-workstream adoptions, not in scope for WS-0, and did not leak
  into the spike. No scope ballooned past its disposition.

Reconcile with the planning challenge (existential item 1): the spike CONFIRMS
  it, empirically. My cold planning read predicted @hiveplotlib/d3 v0.3.0 is a
  static-SVG renderer with zero pointer handlers (read from the JS source: no
  `.on(`, only render/error lifecycle events). The spike's interaction inventory
  turned that code-read prediction into a MEASURED FACT on the built page:
  results.json shows .axis/.node/.edge all `dom_changed_on_hover: false` and
  `dom_changed_on_click: false`, byte-identical outerHTML lengths across hover
  and click (143/143/143, 162/162/162, 1752/1752/1752), `has_title_child:
  false` on all three, and pointer events intercepted by the parent SVG /
  theme chrome (nothing listening). This STRENGTHENS the existential concern
  going into the HALT: the render-only widget ships strictly less interactivity
  than examples/hover_info.ipynb's existing kernel-free bokeh hover page, and
  that is now confirmed on the artifact, not inferred. The spike did exactly
  what item 4 was mandated to make it do — collect the evidence the existence
  question is decided on — and the evidence lands against WS-1+ existing as
  scoped. Item 1 remains the maintainer's to resolve at the reconsider
  checkpoint; the spike hands that checkpoint a hard measurement, not vibes.

Verdict holds up: the PASS grade is substantially honest and evidence-traced.
  Route (b) core (i)-(iii) is substantiated by results.json (widget present,
  "embedded build-time value: 68", clicks 0->2) + screenshots + no_kernel_proof
  .json (zero websockets, zero kernel ws, no localhost port but 8000). The exact
  .axis=3/.node=100/.edge=76 counts trace to real payloads, not magic constants
  (expected_counts.json `example` = 3/100/76, computed by make_payloads.py from
  example_hive_plot().to_json()); the counter's 68 is the karate node count with
  repeat_axes=True (expected_counts.json `karate` nodes=68, counter_value.txt=68).
  The interaction-inventory finding is correctly scoped as an existence question
  for the maintainer, not a WS-0 failure: PASS asked the renderer to render with
  expected counts, which it did (renderer.png shows a real three-axis hive plot).
  The PASS-vs-pass-with-caveats grade is defensible under the plan's own criteria
  (PASS-WITH-CAVEATS is reserved for a renderer/CDN failure or material wrinkle;
  the renderer passed and the CDN wrinkle self-recovered). All required verdict
  fields present and filled; conf.py corroborates the divergence list.

Concerns:
  - [must-fix] Caveat 2's "cold run 404'd on primary CDN, fell back to jsdelivr"
    is contradicted by the very evidence file written to characterize it —
    at verdict caveat 2 / network_404.json
    Rubric: no entry (grill deferred; failure-mode rubric not yet populated)
    network_404.json shows `"failed": []` and BOTH anywidget-related requests
    returning 200 (@jupyter-widgets/html-manager 200, anywidget@~0.11.* 200),
    while results.json console (all three pages) logs a `404` error line
    immediately followed by the jsdelivr fallback log. capture_404.py is the
    tool purpose-built to record the failing URL and it captured zero failures.
    So the URL that 404'd is nowhere in the evidence; the verdict reads
    causation from an ordering coincidence (404 line then fallback line) it
    cannot pin. Does NOT fail the spike — route (b) core passes independent of
    this — but the caveat's evidence is internally inconsistent. Reconcile:
    either re-run capture_404.py cold to capture the 404 URL, or soften the
    caveat to "a 404 error and a fallback log co-occur in console; the failing
    URL was not captured." Routing: this bites the durable WS-0 verdict itself,
    not a gated downstream workstream, so it is the one finding the grill does
    not automatically cover — recommend a small pre-HALT verdict-text
    correction (or an explicit maintainer ack that the caveat is soft) rather
    than riding into the grill.
  - [worth-discussing] "Route (b) transfers to RTD" is a reasoned architectural
    inference, not something the spike reproduced — at verdict "RTD
    transferability" line
    Rubric: no entry (grill deferred)
    The spike built locally with `nbsphinx_kernel_name = "spike"` set (conf.py:23)
    — the SAME named-kernel mechanism route (a) uses — and served via
    python -m http.server. Route (a) and route (b) came back byte-identical in
    results.json (both value 68, both clicks 0->2, identical console/URLs), which
    is consistent with "route (b) rendered from stored outputs" but EQUALLY
    consistent with "both executed at build time on the spike kernel," because
    the local build has that kernel. The one condition that would prove route (b)
    is kernel-independent at BUILD time (build with no kernel registered, or
    unreachable) was never run; no nbconvert invocation is recorded in the spike
    sources either. The kernel-freeness that IS proven is view-time (browser
    opened no kernel websocket), which is real and sufficient for the serving
    model — but the "transfers to RTD" claim rests on the ipywidgets stored-output
    embed model, not on a reproduced no-kernel build. Honest scope: route (b) is
    architecturally the RTD-shippable route; the spike proved the view-time embed
    chain is kernel-free and did not itself reproduce the no-kernel build.
    Downstream bearing: YES — WS-1/WS-2 build the real widget page on exactly this
    premise; if the no-kernel-build assumption is wrong it fails at RTD build
    time, not spike time. Routing: WS-1/WS-2 are hard-gated on the grill and the
    grill's failure-mode wave is the right place to name "no-kernel-build not
    reproduced" as a mode WS-2's done-when must close (e.g. a first-real-build
    RTD smoke). Rides into the grill; no pre-HALT amendment needed. Recommend the
    verdict's "route (b) transfers" wording be read as inference at the HALT so
    the maintainer weighs it as such.
  - [low-confidence] "kernel-free" top-line could be misread as covering the
    build; it is proven for serving/viewing only — at verdict headline /
    Build gates line
    Rubric: no entry (grill deferred)
    no_kernel_proof.json tests view-time connections (websockets, off-port
    localhost) for counter_route_b and renderer only; the build itself used the
    spike kernel (verdict Build gates: "route (a) executed on the spike kernel
    during the build"). The body text disambiguates, so this is a wording nit,
    not a false claim. Rides into the grill with item 2; no action needed
    pre-HALT.
```

## Workstreams

WS-0 is authorized now. **WS-1, WS-2, and WS-3 are planned in full but explicitly gated**: none may dispatch before (a) the WS-0 verdict is written into this plan, (b) the maintainer's grill runs (including the failure-mode wave), and (c) the maintainer gives an explicit go. After WS-0 the dispatching session creates a GitLab issue pointing at this plan (assigned to the maintainer per standing preference) and HALTs. No library code beyond the spike, no branch, no MR until the go.

### Workstream 0: kernel-free docs-embed feasibility spike

**Status:** not started
**Files:** none in any repo. All spike artifacts live in WSL `/tmp/anywidget-spike/` (or the session scratchpad); throwaway by design. The only repo write this workstream makes is its verdict into this plan file (the "WS-0 verdict" block below plus an Implementation log line). No edits to `pyproject.toml` or any dependency file, no branch, no `src/` or `examples/` changes, no commits.

**Question:** does a minimal anywidget, with its state embedded at docs build time, render and stay interactive in built Sphinx HTML served by a plain static file server, with no Jupyter kernel and no backend, through the same pipeline hiveplotlib's docs actually use?

**The pipeline the verdict must transfer to (named): Read the Docs.** The published docs are built by RTD per `.readthedocs.yaml`, not by `build_sphinx_docs.sh`: RTD's own `pre_build` job copies `examples/*.ipynb` into `docs/source/notebooks/`, pip-installs the package with extras, registers **no** jupyter kernel, and builds with `fail_on_warning: true` (local builds run `python -m sphinx -n`, plus `-W` in the strict form). Both pipelines run **nbsphinx** (`docs/source/conf.py:36-43`) with the kernel pinned to `hiveplotlib` (`conf.py:129`) and `nbsphinx_execute` left at its default `auto`: notebooks committed **with** outputs render from stored outputs (the pinned kernel is never used); notebooks **without** outputs execute at build time on that kernel — which does not exist on RTD, so an output-free notebook is a `NoSuchKernel` build failure there. Consequence for the two candidate routes: **(b) committed with outputs plus saved widget state is the only RTD-shippable route today**; (a) committed output-free with build-time execution (where nbclient's default `store_widget_state=True` captures widget state) is **diagnostic-only in this spike** — unshippable without an RTD-config amendment (kernel registration) that appears nowhere in this plan. The expected embed chain: notebook widget state lands as an `application/vnd.jupyter.widget-state+json` script in the built page, nbsphinx/ipywidgets load the client-side widget manager from a CDN, and anywidget's front-end reads the widget's ESM out of the embedded state (anywidget carries `_esm` in the model state; its front-end package is on npm for exactly this embed path). The `docs` extra already carries `ipywidgets` for nbsphinx widget metadata (`pyproject.toml:62`). WS-0 confirms this chain end to end rather than trusting it.

**Spike protocol:**

1. Isolated venv at `/tmp/anywidget-spike/venv` (via `uv venv` / `uv pip`), installing: hiveplotlib from the local checkout path (read-only build; no `-e`), `networkx` (directly, never the scipy-pulling `[networkx]` extra), `anywidget`, `ipywidgets`, `sphinx`, `nbsphinx`, `myst-parser`, `pydata-sphinx-theme`, `ipykernel`, `jupyter-client`, `nbconvert`, `playwright`. Register a jupyter kernel from this venv and point the spike's `nbsphinx_kernel_name` at it (mirroring the named-kernel mechanism).
2. Minimal Sphinx project in `/tmp/anywidget-spike/` mirroring the real settings that matter: `extensions = ["nbsphinx", "myst_parser"]`, `html_theme = "pydata_sphinx_theme"`, `nbsphinx_kernel_name` set, `nbsphinx_execute` left at default. Record in the verdict every real-`conf.py` setting deliberately not mirrored (sphinx_copybutton, autodoc, intersphinx, the in-repo custom directive), so the grill sees the divergence list rather than trusting "the settings that matter" by judgment.
3. Two notebooks carrying the same minimal anywidget (a counter-style widget whose ESM renders a button and whose Python state embeds a value computed at build time from a real hiveplotlib object, e.g. the node count of a `HivePlot.to_json()` payload): one committed output-free (route (a), nbsphinx executes at build time; **diagnostic-only** per the RTD note above), one pre-executed via `jupyter nbconvert --to notebook --execute` so outputs plus widget state are stored (route (b), the RTD-shippable route).
4. A third notebook for the renderer/CDN check: a widget whose ESM imports `@hiveplotlib/d3` from CDN (jsdelivr) and hands an embedded, real, current `to_json()` payload (e.g. from `hiveplotlib.datasets.example_hive_plot()` or the karate club construction in Example 1) to the `<hive-plot>` element or `visualizeHivePlot()`. This simultaneously re-verifies the npm-published package version/exports against the local v0.3.0 checkout and confirms the current three-key payload (`axes`/`edges`/`node_viz_kwargs`) renders. This rendered hive plot is the subject of step 6's interaction inventory.
5. Build the Sphinx project **under the real gates**: `python -m sphinx -W -n` (RTD sets `fail_on_warning: true`; the strict local build passes `-W`, and `-n` is always on locally). A spike built bare could go green while hiding a warning that fails the real build. Serve the built HTML with `python -m http.server` only; ensure no Jupyter process is running.
6. Headless-browser evidence via playwright (chromium installed into the user cache; leave it, per the dump-don't-clean preference): for each page, assert the widget's DOM node exists, assert the build-time-embedded value is displayed, drive an interaction (click) and assert the DOM changes in response, capture the browser console, and save screenshots to `/tmp/anywidget-spike/evidence/`. On the renderer page, additionally run a hover/click pass over the rendered `.axis`/`.node`/`.edge` elements, recording events observed and any DOM response — the **interaction inventory** (the evidence the adversary's existential item and grill questions 1-2 get decided on). Record every URL the served pages fetch at view time, with pinned-vs-floating status per URL.

**Pass/fail criteria (evidence-based; a grep of built HTML is not evidence):**

- **PASS:** on **route (b)** (the RTD-shippable route), the minimal widget in the statically served built HTML (i) renders, (ii) displays the value embedded at build time, and (iii) responds to a click with a DOM change, all verified by the playwright run with screenshots saved, no kernel or non-static server involved; AND the renderer check (protocol step 4) renders an SVG hive plot with the expected `.axis`/`.node`/`.edge` element counts from the embedded payload. Route (a)'s result is recorded as diagnostic either way; it cannot supply a PASS.
- **PASS-WITH-CAVEATS:** route (b)'s minimal-widget core (i)-(iii) passes, but the renderer/CDN check fails or CDN-at-view-time shows a material wrinkle. Each caveat named explicitly with its console/error evidence; caveats become WS-1/WS-2 design constraints.
- **FAIL:** route (b)'s core fails: the minimal widget does not render kernel-free, the displayed state was not embedded at build time, or interactivity requires a kernel. A route-(a)-only success is still a FAIL for shipping purposes (record its diagnostic result; the only recovery is an RTD-config amendment, a maintainer decision at the HALT). A FAIL halts the plan for maintainer review; do not attempt architecture substitutions (rule 9: no silent substitution).

The executing agent writes an explicit verdict (pass / fail / pass-with-caveats) into the "WS-0 verdict" block below, filling every field: per-route results with RTD transferability, the build-gate outcome and conf divergence list, the interaction inventory, the verified npm `@hiveplotlib/d3` version, the per-URL pinned-vs-floating CDN inventory, and pointers to the evidence artifacts.

**Environment notes (bake-in, do not rediscover):** agents run Windows-side. In-repo and in-WSL commands go through `wsl.exe -e bash -lc "cd <dir> && <cmd>"`; never lead the `-lc` string with `/` (MSYS path-mangling), so call uv as `~/miniconda3/bin/uv` after a leading `cd`. Never heredoc file content through the bridge; author every spike file (notebooks, `conf.py`, `index.rst`, ESM strings, the playwright script) with the Write/Edit tools over the UNC path (`\\wsl.localhost\Ubuntu\tmp\anywidget-spike\...`). Reserve the bridge for running commands.

**Done when:** the spike ran end to end per the protocol; the verdict block below is filled with pass/fail/pass-with-caveats plus the required inventory; the Implementation log has its line; `/tmp/` holds the evidence; the working tree shows **no** changes beyond this plan file (`git status` checked, including the CRLF-warning sweep).

**WS-0 verdict:**

```
Verdict: PASS (route (b) core + renderer element-count check both succeeded,
  verified kernel-free by playwright with screenshots). Two view-time caveats
  recorded below become WS-1/WS-2 design constraints; neither breaks route
  (b)'s core (i)-(iii) nor the renderer check, so neither downgrades the
  verdict to pass-with-caveats (that grade is reserved for a renderer/CDN
  *failure* or a *material* wrinkle; both caveats here self-recovered or were
  already-known scope facts). Env: hiveplotlib 0.29.0a0 (non-editable install
  from local checkout), anywidget 0.11.0, ipywidgets 8.1.8, nbsphinx 0.9.8,
  nbconvert 7.17.1, nbclient 0.11.0, sphinx 9.1.0, pydata-sphinx-theme 0.19.0,
  playwright chromium 149. Static server: python -m http.server 8000 only.

Route (b) saved widget state (RTD-shippable route; PASS hinges here): YES.
  counter_route_b.ipynb pre-executed via `jupyter nbconvert --to notebook
  --execute` (nbclient store_widget_state default), committed with outputs +
  notebook-level `widgets` metadata carrying the anywidget `_esm`. In the
  statically served built HTML the widget (i) renders (`.spike-counter` DOM
  node present), (ii) displays the build-time-embedded value ("embedded
  build-time value: 68", the karate-club node count computed at build time
  from a real HivePlot.to_json()), and (iii) responds to a click with a DOM
  change ("clicks: 0" -> "clicks: 2"). Zero websockets opened, zero kernel
  connections, zero requests to any localhost port but 8000 (no_kernel_proof
  .json). Screenshot: counter_route_b.png.

Route (a) build-time execution (diagnostic-only): YES (diagnostic; cannot
  supply a PASS). counter_route_a.ipynb committed output-free; nbsphinx
  executed it at build time on the registered `spike` kernel (kernel warning
  in the build log confirms execution), and nbsphinx stored the resulting
  widget state, so the served page rendered identically (value 68, click 0->2,
  kernel-free at view time). This only works because the spike registered a
  kernel named to match `nbsphinx_kernel_name`; on RTD no such kernel exists,
  so route (a) is unshippable there (see RTD transferability). Screenshot:
  counter_route_a.png.

RTD transferability: route (b) transfers; route (a) does not. Route (b) ships
  its widget state inside committed notebook outputs + metadata, which RTD
  renders from stored outputs with no kernel and no execution — consistent
  with .readthedocs.yaml (no registered jupyter kernel) and fail_on_warning:
  true (the spike built clean under -W). Route (a) requires build-time
  execution on the conf.py:129-pinned `hiveplotlib` kernel, which does not
  exist on RTD (NoSuchKernel -> build failure); shipping it would need an
  RTD-config amendment (kernel registration) that appears nowhere in this
  plan. Confirmed the client-side embed chain fetches only CDN + static assets
  at view time (no backend), so static hosting (RTD's model) is sufficient.

Build gates: sphinx -W -n CLEAN (0 warnings, 0 errors; 4 sources built, route
  (a) executed on the spike kernel during the build). Real-conf.py settings
  deliberately NOT mirrored (divergence list): (1) `metric_table_directive`
  in-repo custom directive; (2) `sphinx_copybutton`; (3) `sphinx.ext.autodoc`;
  (4) `sphinx.ext.intersphinx` and everything hanging off it (intersphinx_
  mapping, intersphinx_timeout, nitpick_ignore, the `fix_reference` missing-
  reference hook, tls_verify); (5) `nbsphinx_execute_arguments` (InlineBackend
  figure formats/dpi); (6) `linkcheck_ignore` (incl. the npmjs 403 ignore).
  None of these touch the notebook-widget embed path; the widget-state script,
  ipywidgets embed manager, and view-time CDN chain are all independent of
  them. Note: a NEW nbsphinx/myst warning introduced by the real widget
  notebook (WS-2) would still fail the real -W build; the spike proves the
  mechanism warning-free, not the eventual WS-2 page.

Interaction inventory (rendered .axis/.node/.edge on renderer.html): STATIC
  SVG, NO POINTER HANDLERS. Over `.axis` (path), `.node` (circle), `.edge`
  (path): outerHTML length is byte-identical before hover, after hover, and
  after click on all three (no attribute/child mutation); no `<title>` child
  on any (no native SVG tooltip); hover/click on `.axis`/`.node`/`.edge`
  produced no DOM response. (Playwright reported the elements as pointer-
  occluded by the parent SVG / the pydata theme's back-to-top button, i.e.
  even dispatching a real pointer event required working around the theme
  chrome — there is simply nothing listening.) This confirms the adversary's
  existential-item evidence directly: @hiveplotlib/d3 v0.3.0 is a static-SVG
  renderer with CSS-targetable data attributes and only render/error lifecycle
  events; the widget as scoped (render-only) ships a static SVG, strictly less
  interactive than the existing bokeh hover_info.ipynb docs page. This is the
  evidence grill questions 1-2 and the existence question decide on. (Not a
  verdict downgrade: the PASS criteria ask the renderer to *render with the
  expected element counts*, which it did; interactivity of that render is the
  separate existence question the maintainer settles at the HALT.)

@hiveplotlib/d3 npm version verified: 0.3.0 (matches the local checkout). npm
  `exports`: {".": "./hive_plots_d3_viz.js", "./element":
  "./hive_plot_element.js"}; `main`: hive_plots_d3_viz.js. Renderer used the
  `./element` export's `<hive-plot>` web component, setting its `data` property
  to the embedded example_hive_plot() payload (current three-key schema:
  axes / edges / node_viz_kwargs). Rendered SVG element counts matched the
  payload EXACTLY: .axis=3, .node=100, .edge=76 (expected 3/100/76 from the
  payload's axis count, summed node placements, and summed non-empty edge
  `ids`). Screenshot: renderer.png (three axes A/B/C, node circles, curved
  edges).

CDN loaded at view time (per URL, from request interception on renderer.html;
  full list in evidence/results.json):
  - https://cdnjs.cloudflare.com/ajax/libs/require.js/2.3.4/require.min.js
    — PINNED (2.3.4). [nbsphinx widget bootstrap]
  - https://cdn.jsdelivr.net/npm/@jupyter-widgets/html-manager@^1.0.1/dist/
    embed-amd.js — FLOATING (caret range ^1.0.1). [ipywidgets embed manager]
  - https://cdn.jsdelivr.net/npm/anywidget@~0.11.*/dist/index.js — FLOATING
    (tilde-star ~0.11.*). [anywidget front-end; see caveat 2]
  - https://cdn.jsdelivr.net/npm/@hiveplotlib/d3@0.3.0/hive_plot_element.js
    and .../hive_plots_d3_viz.js — PINNED (0.3.0, from the ESM import). [the
    renderer; pinned because the spike ESM hard-codes @0.3.0]
  - https://cdn.jsdelivr.net/npm/d3@7/+esm — FLOATING (major-only @7); it
    transitively pulls ~40 d3-* submodules, each PINNED to an exact x.y.z.
  - https://cdn.jsdelivr.net/npm/mathjax@4/tex-mml-chtml.js (+ mathjax@4 sre
    workers/mathmaps) — FLOATING (major-only @4). [pydata theme mathjax]
  Summary: the ipywidgets embed manager, anywidget front-end, D3 top-level,
  and mathjax are FLOATING; require.js, @hiveplotlib/d3, and the d3-* leaf
  modules are PINNED. Vendoring (open question 1) would pin @hiveplotlib/d3
  but D3 stays external/floating even in the esbuild bundle.

Evidence: /tmp/anywidget-spike/evidence/
  - counter_route_a.png, counter_route_b.png, renderer.png (screenshots)
  - results.json (per-page: DOM node presence, embedded value, click DOM
    change, renderer element counts + interaction inventory, view-time URLs,
    console, page errors)
  - no_kernel_proof.json (zero websockets, zero kernel connections, zero
    off-static-port localhost requests, both widget pages)
  - network_404.json (anywidget/embed-manager request status)
  Spike sources under /tmp/anywidget-spike/ (venv, conf.py, index.rst, the
  three notebooks, the two ESM files, make_payloads.py, make_notebooks.py,
  playwright_check.py); throwaway.

Caveats (become WS-1/WS-2 design constraints, do not downgrade the PASS):
  1. Render-only means static SVG. The shipped docs artifact of a render-only
     widget is a non-interactive SVG (interaction inventory above). This is the
     existence question (open question 2 / adversary item 1), surfaced at the
     HALT, not a spike failure.
  2. anywidget front-end CDN is floating, with console evidence of a fallback
     path. On one cold playwright run the console showed a 404-error line
     ("Failed to load resource: ... 404") immediately followed by an anywidget
     fallback log line ("Falling back to https://cdn.jsdelivr.net/npm/ for
     anywidget@~0.11.*"); the widget rendered regardless. The purpose-built
     failed-request capture (network_404.json) recorded zero failed requests
     with both anywidget requests at 200, so the failing URL was never
     captured: the 404-then-fallback ordering in the console is the only
     evidence, and the "primary CDN 404'd, so anywidget fell back" mechanism is
     inferred from that ordering, not proven (the 404 line names no URL, and it
     could originate elsewhere on the page). What is established: the render
     never blocked, and this remains a floating, multi-CDN, version-range
     dependency resolved at view time, relevant to open question 1 (vendoring
     / who notices a silently broken widget).
  3. View-time CDN dependence is inherent (see the CDN inventory): the embed
     chain fetches require.js, the ipywidgets embed manager, the anywidget
     front-end, @hiveplotlib/d3, ~40 d3-* modules, and mathjax at view time;
     linkcheck is blind to all of it (npmjs already linkcheck-ignored). Feeds
     open question 1's vendoring/breakage-noticer decision.
  4. Two Jupyter servers were running elsewhere on the host during the spike
     (unrelated nbclassic sessions on other ports). They were provably
     uninvolved: no_kernel_proof.json shows the browser opened zero websockets
     and contacted no localhost port but 8000. Kernel-freeness holds; noted for
     honesty since the protocol says "verify no Jupyter process is running" —
     the meaningful guarantee (the served pages have no comm channel to any
     kernel) is proven directly rather than by host-wide process absence.
```

### Post-WS-0 gate (procedure, not a workstream)

Verdict written → dispatching session creates a GitLab issue pointing at this plan (assigned to the maintainer) → **HALT** for the maintainer. WS-1+ additionally require the grill (with failure-mode wave) and an explicit go. Auto-dispatch is not offered on this plan until the rubric exists.

### Workstream 1: `HivePlotWidget` core (GATED)

**Status:** not started (gated: WS-0 verdict + grill + explicit go; soft-dependency on explorer-plan WS-F, see the sequencing section)
**Files:** `src/hiveplotlib/widget.py` (new module: `HivePlotWidget` + its ESM/CSS assets, inline or as package data per the WS-0-informed CDN/bundling decision), `pyproject.toml` (`widget = ["anywidget"]` extra; add `hiveplotlib[widget]` to the `dev` and `docs` extras; new pytest marker `widget` alongside the optional-backend markers), `tests/widget_test.py`, `CHANGELOG.rst`, `CLAUDE.md` (architecture pointer list).
**Done when:** `HivePlotWidget(hp)` renders an interactive hive plot in a live Jupyter session via the WS-0-verified mechanism (ESM wrapping `@hiveplotlib/d3`, payload snapshot from `to_json()` at construction); the constructor validates its input is a `BaseHivePlot` (covers `HivePlot` and the manual workflow) and raises a recovery-oriented error naming what's supported — a `P2CP` or `HivePlotMatrix` must fail at construction in Python, never client-side as a blank widget — with tests covering the guard (api-critic's design rule: after construction there is no Python exception channel, so every reachable failure surfaces at construction or is designed out); the class docstring's first paragraph states the snapshot-freeze ("state is frozen at construction; re-render by constructing a new widget") and one payload-scale sentence scoping the widget to small/medium networks (datashader stays the big-network route); the `:raises:` block lists the `TypeError` inherited from `to_json()` on lazy Dask-backed edge subsets; `width`/`height` behave per the defaults section; the anywidget import is guarded with the helpful-extra error naming `hiveplotlib[widget]`, and `hiveplotlib/__init__.py` is verified not to auto-import the new module (the datasets star-import gotcha); tests mirror source as `widget_test.py` with the `widget` marker at 100% coverage, honoring the test-name = test-body contract; `make format` / `make ty` clean; CHANGELOG entry (new released behavior); no process refs in shipped code (no wiki-plan/WS-label/grill citations); api-critic + adversary post-impl invoked.

### Workstream 2: docs page with the kernel-free embed (GATED)

**Status:** not started (gated on WS-1; embed route: (b), unless the plan gains an RTD-config amendment)
**Files:** `examples/interactive_hive_plot_widget.ipynb` (new), `docs/source/conf.py` (`nbsphinx_thumbnails` entry), `docs/source/_static/` thumbnail, docs index rST, `docs/source/_llms/llms.txt`.
**Done when:** the new notebook (gallery genre, `HivePlot` class scope, karate club dataset, matching `examples/karate_club.ipynb`'s dataset precedent) demonstrates the widget with Example 1's construction and lands in the built docs as a kernel-free interactive page via **route (b)** — committed with outputs plus saved widget state — unless the plan gains an RTD-config amendment endorsing build-time execution; the notebook opens with the extras install banner (`hiveplotlib[widget]`, plus `[networkx]` for the example data) per `examples/karate_club.ipynb`'s precedent; `make docs` builds clean and the published page's widget works from static hosting (verified the WS-0 way: served statically, browser-checked); the docs-build verification asserts the built widget page's HTML contains the `application/vnd.jupyter.widget-state+json` script (a grep is legitimate here as a regression check, though not spike-grade evidence) — guarding the silent-degradation trap where a routine notebook re-save without widget state drops the widget and the docs build green with a hole; the re-execution workflow for this notebook is recorded in the Implementation log (routine re-saves must go through an execution path that stores widget state, e.g. `jupyter nbconvert --execute` with nbclient's `store_widget_state` default), and whether the built-HTML assertion lands as a durable CI/test tripwire versus a qa verification step is a grill call; `make test-nb` passes; llms.txt gains an entry (new capability, consequential: proposed under `## Optional` as a feature demo; notebook-author's judgment call to pin higher); no separate CHANGELOG entry (the widget debuts in the same unreleased version; WS-1's entry covers it); editorial-critic, viz-critic, and adversary post-impl invoked. Notebook-coherence flags for sign-off: none beyond the new page itself (existing dataset, established genre, single-class scope).

### Workstream 3: hover metadata passthrough (GATED)

**Status:** not started (gated on WS-1 **and** explorer-plan WS-F items 2-3 shipping)
**Files:** `src/hiveplotlib/widget.py`, `tests/widget_test.py`, `CHANGELOG.rst` (only if the widget shipped in a prior release by then; otherwise the same-unreleased-version suppression applies), possibly the WS-2 notebook (a hover section).
**Done when:** `HivePlotWidget(hp, node_data_columns=[...], edge_data_columns=[...])` passes through to WS-F's `to_json` parameters and the rendered widget shows tooltips from the embedded metadata; each passthrough parameter's docstring leads with the user's word ("shown as hover tooltips") before the mechanism; defaults `None` keep the payload unchanged; tests cover the passthrough at 100% coverage. **External dependency flag:** if tooltip rendering needs renderer-side work, that lands in the hiveplotlib-javascript repo (a different consumer, outside this plan); route the coordination through the maintainer rather than scoping JS work here.

## Open questions for the pre-implementation grill

Embedded for the maintainer's post-WS-0 grill; none block WS-0.

1. **CDN-at-view-time policy for shipped docs.** The mechanism inherently loads CDN JS at view time (ipywidgets embed manager, D3 v7, possibly `@hiveplotlib/d3` itself). WS-0 delivers the exact per-URL pinned-vs-floating inventory. Accept for RTD-hosted docs (linkcheck already ignores npmjs, `conf.py:191`), or vendor the esbuild-minified `@hiveplotlib/d3` bundle into the widget's assets (D3 stays external even in that bundle)? The adversary's framing (item 5) stands for this grill: vendoring is the **default posture pending a named reason against** — explorer settled decision 3's recorded rationale is keeping churning JS out of the release-critical docs build, not install weight — and either way the maintainer must name who or what notices a silently broken widget on the published site (linkcheck is blind to JS).
2. **The existence question (supersedes the earlier "MVP interaction set" framing — this is not MVP polish).** The adversary's existential item: `@hiveplotlib/d3` v0.3.0 registers zero pointer handlers (static SVG with CSS-targetable attributes; its only events are render/error lifecycle dispatches), and `examples/hover_info.ipynb` already ships kernel-free hover-enabled hive plots in the docs today via committed BokehJS outputs. As scoped, WS-1's docs artifact would be *less* interactive than that existing page. Before any WS-1 dispatch the maintainer must answer: what does this widget do in the docs that the bokeh page does not, and which in-scope work supplies it? If the answer needs renderer-side or widget-side interaction JS, amend WS-1 or add a workstream; if no answer survives, WS-1+ do not exist and the plan reduces to WS-0's knowledge value. WS-0's interaction inventory (amended verdict grammar) is the evidence this gets decided on; the dispatching session surfaces it as the reconsider checkpoint at the post-WS-0 HALT.
3. **WS-F timing.** Dispatch explorer-plan WS-F before WS-1 (recommended), or accept an unversioned first widget release with both halves version-locked in one release?
4. **Extra name** `[widget]` vs `[jupyter]`; **module home** `hiveplotlib.widget` vs somewhere under `viz`. api-critic's planning take feeds this.
5. **Snapshot semantics.** Is construct-time payload freezing acceptable long-term, or is a documented `refresh()`/re-sync wanted for live sessions?
6. **Explorer priority revisit.** Once the widget ships, does the WASM explorer's demo-value case still justify its ~30 MB payload and six workstreams, or does its priority drop? (Post-ship trigger; not this plan's call.)

## Plan amendments

Cycle 1 (2026-07-06): both planning-mode critic sections landed; the maintainer ordered WS-0 to run **before** the grill and mandated the adversary's items 2-4 as pre-spike fixes (without them the spike verdicts on an untrustworthy protocol). The grill still interrogates the full challenge afterward; this cycle adopts only what must land pre-spike plus cheap done-when adoptions on gated workstreams the grill can relitigate.

### In-scope tweak: WS-0's verdict must transfer to Read the Docs; route (a) demoted to diagnostic-only

**Date:** 2026-07-06
**Trigger:** adversary planning item 2 (must-fix; maintainer-mandated pre-spike)
**Workstream affected:** WS-0 (pipeline note, protocol step 3, pass/fail criteria, verdict grammar) + WS-2 (status line, route language)
**Change:** "the pipeline is `build_sphinx_docs.sh`" → the published docs build on RTD (`.readthedocs.yaml`: own `pre_build` notebook copy, no registered jupyter kernel, `fail_on_warning: true`; re-verified on this branch), where an output-free notebook hits `NoSuchKernel` against `conf.py:129`'s pinned kernel. Route (b) is the only RTD-shippable route today; route (a) is diagnostic-only and cannot supply a PASS (a route-(a)-only success is a FAIL for shipping, its diagnostic result recorded — the recovery is an RTD-config amendment, a maintainer call at the HALT); the verdict gains an RTD-transferability line. WS-2's "whichever the verdict endorsed" → "route (b), unless the plan gains an RTD-config amendment".

### In-scope tweak: spike builds under the real gates (`-W -n`), conf divergences recorded

**Date:** 2026-07-06
**Trigger:** adversary planning item 3 (must-fix; maintainer-mandated pre-spike)
**Workstream affected:** WS-0 (protocol steps 2 and 5, verdict grammar)
**Change:** bare `sphinx` build → `python -m sphinx -W -n`, mirroring RTD's `fail_on_warning: true` and the strict local form (verified: `build_sphinx_docs.sh` always passes `-n`, `-W` in its strict branch). Step 2 now records every real-`conf.py` setting deliberately not mirrored (copybutton, autodoc, intersphinx, custom directive) into the verdict's divergence list.

### In-scope tweak: verdict grammar gains the interaction inventory and per-URL pinned-vs-floating CDN status

**Date:** 2026-07-06
**Trigger:** adversary planning item 4 (must-fix; maintainer-mandated pre-spike)
**Workstream affected:** WS-0 (protocol steps 4 and 6, verdict block)
**Change:** step 6 adds a playwright hover/click pass over the renderer page's `.axis`/`.node`/`.edge` elements (events observed + DOM response) landing as an explicit `Interaction inventory:` verdict line — the evidence the existential item and grill questions 1-2 get decided on; the CDN inventory becomes per-URL pinned | floating.

### In-scope tweak: WS-2 gains a built-HTML widget-state tripwire and a stated re-execution workflow

**Date:** 2026-07-06
**Trigger:** adversary planning item 6 (worth-discussing), adopted now rather than left for the grill because the RTD amendment above makes route (b) the sole shippable route, so its silent-degradation trap (notebook re-save without widget state → docs build green with a hole) is now load-bearing
**Workstream affected:** WS-2 (done-when; gated, so the grill can relitigate)
**Change:** done-when adds: assert the built widget page's HTML contains the `application/vnd.jupyter.widget-state+json` script (grep legitimate as a regression check); record the notebook's re-execution workflow in the Implementation log; CI-vs-qa home for the assertion is a named grill call.

### In-scope tweak: WS-1 constructor input guard (api-critic must-fix) + docstring adoptions (api-critic worth-discussings)

**Date:** 2026-07-06
**Trigger:** api-critic planning mode (one must-fix, four worth-discussings); folding into gated done-whens is cheap and the grill can relitigate
**Workstream affected:** WS-1, WS-2, WS-3 (done-whens; all gated)
**Change:** WS-1 validates input is a `BaseHivePlot` and raises a recovery-oriented error (a `P2CP`/`HivePlotMatrix` fails in Python at construction, never as a blank widget), tests cover the guard, and api-critic's design rule (every reachable failure surfaces at construction or is designed out) is named in the done-when; WS-1's docstring carries the snapshot-freeze lede, a payload-scale scoping sentence, and a `:raises:` entry for `to_json()`'s Dask `TypeError`; WS-3's passthrough params lead with "shown as hover tooltips"; WS-2's notebook opens with the extras install banner. Whether a runtime payload-size *warning* is also wanted stays a grill call, as api-critic framed it.

### Dispositions of the remaining planning findings (2026-07-06)

Recorded so the adversary's post-impl reconcile has dispositions to read. The planning challenge as a whole is **not** disposed here; the maintainer fights it in the post-WS-0 grill.

- **Adversary item 1 (existential-must-fix): stands, undisposed — not the orchestrator's to resolve.** Per the tiered-adversary rule, the dispatching session surfaces it to the maintainer at the post-WS-0 HALT as the reconsider checkpoint. This amendment only (a) reframed open question 2 as the existence question (superseding its "MVP polish" framing) and (b) ensured WS-0's amended verdict grammar collects the interaction-inventory evidence the decision needs.
- **Adversary item 5 (vendoring as default posture + breakage-noticer): deferred to the grill.** Open question 1 now carries its framing verbatim-in-substance. WS-1/WS-2 are hard-gated on that grill, so adopting a vendoring default now would pre-empt a maintainer policy call with zero pre-spike consequence; WS-0's per-URL pinned-vs-floating inventory feeds the decision either way.
- **Adversary item 7 (low-confidence, trim the spike): ruled out — the three-notebook spike stands.** Route (a)'s marginal cost is one notebook against the fixed venv/Sphinx/playwright setup; its diagnostic value is failure isolation (if route (b) fails, route (a) says whether the embed chain or the state-save path broke) plus the evidence for any RTD-config amendment the maintainer might order at the HALT. Route (a) is already off the verdict's critical path per the RTD amendment, so keeping it cannot inflate a PASS.

## Holdouts

- `src/hiveplotlib/hiveplot.py:5115-5118`: the `to_json` docstring exclusion note stays as-is under this plan; its rewrite belongs to explorer-plan WS-F.

## Implementation log

2026-07-06: Workstream 0 complete. Ran the kernel-free docs-embed spike end to
end in throwaway /tmp/anywidget-spike/ (isolated uv venv; non-editable
hiveplotlib install + anywidget/ipywidgets/nbsphinx/myst/pydata-theme/
playwright). Built two counter notebooks (route (a) output-free, route (b)
nbconvert-executed with stored widget state) + a renderer notebook embedding a
real example_hive_plot() to_json() payload rendered via @hiveplotlib/d3 v0.3.0's
<hive-plot> element. Sphinx built clean under -W -n; served statically via
python -m http.server; playwright verified (screenshots) that route (b) renders
kernel-free, displays the build-time-embedded value (karate node count 68),
responds to a click with a DOM change, and that the renderer emits an SVG hive
plot with exact expected element counts (.axis=3/.node=100/.edge=76); zero
websockets / zero kernel connections captured. Verdict: PASS (route (b) is the
sole RTD-shippable route; route (a) diagnostic-only). Interaction inventory
recorded static-SVG-no-handlers (the existence-question evidence); per-URL CDN
pinned/floating inventory and conf-divergence list filled. Cleaned up a build/
dir the non-editable install left in the repo tree (my side-effect); working
tree restored to baseline. Verdict block above filled.
