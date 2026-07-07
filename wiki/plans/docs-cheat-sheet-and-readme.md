# Plan: docs cheat sheet, README leading text, saving-for-publication notebook

## Goal

Three docs deliverables from the 2026-06-10 docs-gaps discussion. (1) A compact, task-oriented **cheat-sheet docs page** (MyST markdown, new toctree section, sphinx-design tabs/dropdowns) answering "how do I do X" lookups without digging through notebooks, with every fenced snippet executed in CI by a new docs-snippet pytest harness (testability == copy-paste-ability). (2) A few sentences of **README leading text** giving arriving skeptics the "when and why hive plots" framing, pointing at the introduction notebook. (3) A new gallery notebook, **saving plots for publication**, consolidating vector export (SVG/PDF), DPI handling, and the datashader DPI/rasterization-parameter coupling currently scattered across datashader notebooks (repeating that scattered info here is the point).

## Alignment (grill)

Status: **aligned / ready-for-execution** (2026-06-10; execution authorized 2026-07-06, record below). Gary closed all three open decisions, each on the recommended option; formal grill skipped in favor of direct decision closure on top of the originating docs-gaps discussion. The branch-46 merge gate that held implementation lifted 2026-07-06 (see Sequencing and Amendment 4).

Decisions closed (Gary, 2026-06-10; details in Plan amendments):

1. Toctree caption **"Quick Reference"**, page title **"Cheat Sheet"** — as recommended.
2. WS-C notebook filename `examples/saving_plots_for_publication.ipynb` — as recommended.
3. README sequencing: this plan's WS-B lands **before** the JOSS plan's Workstream A (see Amendment 3).

Settled by Gary in the originating discussion (recorded so later sessions don't relitigate):

- Cheat sheet is a **docs page**, not a notebook (rejected: too many rendered figures, too big), not doctest, not literalinclude-from-snippet-files (both rejected as too heavy).
- Snippet testing: a pytest that extracts and executes the page's fenced Python blocks in order with shared state; rare explicit opt-out marker allowed; every snippet runs in CI by construction.
- sphinx-design tabs do double duty: per-backend variants AND alternate-example variants when a task needs different data. Setup code (1-2 lines, `hiveplotlib.datasets` helpers) in collapsed dropdowns.
- NO performance-threshold guidance on the page (scaling-large-networks WS7 owns that; standalone perf guide rejected 2026-06-10). Cross-link the gotchas page rather than duplicating.
- README: no pointer to the gotchas page (too niche for the README).
- REJECTED: a subplot-embedding ("hive plot as one panel") notebook — spoon-feeding; the matplotlib backend returns fig/ax first class. At most a one-line cheat-sheet entry ("pass `fig=`/`ax=`"; the 2026-06-10 record said "`axes=`", a parameter that doesn't exist on any backend — corrected at Amendment 8).

**Execution-time record (2026-07-06, Gary's directive):**

- The post-plan grill is knowingly skipped; the 2026-06-10 decision closure above stands as the alignment substance.
- **Auto-dispatch authorized** for this run; the directive is the recorded opt-in utterance per harness convention. Explicit override: the harness default offers auto-dispatch only on plans whose grill failure-mode wave ran (a named `## Failure modes` rubric exists); this plan has no grilled rubric, and that default is overridden here by explicit maintainer directive.
- Halt conditions for the run: any `must-fix`; any `STATUS: BLOCKED`; any workstream-set change (which routes through orchestrator `amend-plan` first). **Refined 2026-07-06 (Gary), superseding the blanket "any must-fix halts" here and in Amendment 6:** a post-impl must-fix that is a *clear defect against the plan's own done-when* auto-resolves inline (routed through `amend-plan` for the disposition record, then dispatched as a fix) without halting; a *judgment-call* must-fix, any `STATUS: BLOCKED`, and any workstream-set change still halt to the maintainer. First applied at Amendment 9 (the two WS-A copy-paste-runnability defects).
- One commit per workstream after the full gate battery (`make format`, `make test`, `make ty`, `make docs`, `make linkcheck` for new pages, plus any touched notebook run), committed by the dispatching session under explicit maintainer authorization; harness agents still never commit.
- Pre-dispatch gates on the amended scope: adversary cold planning challenge and api-critic planning pass (Amendment 6). With the grill skipped, planning-pass items route to orchestrator `amend-plan` for explicit disposition rather than being fought in a grill. Dispositions done: Amendment 7 (adversary, all six items) and Amendment 8 (api-critic, all seven items). Both gates cleared 2026-07-06; WS-A dispatch unblocked.
- WS-C taste churn accepted (Amendment 7, item 4): editorial/viz `worth-discussing` findings with no downstream bearing batch to plan-end qa after WS-C's per-workstream commit exists; follow-up commits are accepted churn per the standing expect-churn-on-doc-notebooks preference. No pre-commit pause exemption for WS-C; the auto-dispatch directive stands unaltered.

## Prior ADRs / design docs

No `wiki/wiki/adr/` directory exists yet; no prior ADRs. Relevant plans (research-liaison pre-task pass):

- `wiki/wiki/plans/edge-coverage-and-gotchas-docs.md` — its line 11 records README changes as out-of-scope **for that plan**; this plan supersedes that status quo for the leading-text change only (flag, not contradiction — supersession note written there by the orchestrator at Amendment 7). Also: FAQ-page rejection provenance, the gotchas page WS-B cross-links rather than duplicates, and the shared branch-46 merge gate.
- `wiki/wiki/plans/scaling-large-networks.md` — WS7 (perf decision table in `hive_plots_for_large_networks.ipynb`) owns all perf guidance; WS2 will add `stream_chunk_threshold` to the datashader entry points, so WS-C must not pin datashader internals claims that plan is about to rewrite.
- `wiki/wiki/plans/joss-submission.md` — concurrent README work (Citing section, support/governance statement); records that `docs/source/README.md` mirrors the root README and must stay in sync; the leading text is the same statement-of-need argument the JOSS paper makes (prose reuse both directions).
- `wiki/wiki/plans/same-stats-different-graphs.md` line 183 — recurring stale-snippet pattern: check snippets against the *current* `HivePlot` surface (`graph=` keyword from branch 46), not the surface at drafting time. The CI snippet test makes run-failure staleness self-enforcing for the cheat sheet; run-green misuse (silently absorbed kwargs) is not caught by execution and stays an authoring-time signature-check obligation (Amendment 7).
- `wiki/wiki/plans/graph-metrics-notebook-restructure.md` — notebook corpus inventory; datashader figure conventions (its WS-C done-when item 8) for WS-C figures.
- Feeder material for WS-B prose: `wiki/wiki/entities/hiveplotlib.md`, `wiki/wiki/concepts/force-directed-layout.md`, `wiki/wiki/sources/bostock-2012-d3-hive-plots.md`, `wiki/wiki/sources/krzywinski-2012.md`.
- Gaps: no prior wiki record of docs-snippet testing conventions, toctree-structure decisions, or sphinx-design usage; no wiki page on the datashader DPI coupling (material lives only in repo notebooks). The snippet-test harness is a net-new docs-testing convention and a likely future ADR candidate.

## Patterns this replaces

- None — all three deliverables are net-new additions. Two sweep obligations that aren't replacements: (a) any WS-B edit to `README.md` must be mirrored in `docs/source/README.md` (the JOSS plan's recorded sync constraint); (b) a one-line supersession note in `wiki/wiki/plans/edge-coverage-and-gotchas-docs.md` under its "Out of scope" README bullet — **done**, written by the orchestrator at Amendment 7 (plan-body writes are orchestrator surface; dropped from WS-B's Files).

## Default justifications

No new library API, parameters, or defaults. Convention-level defaults (binding; see Design notes for specs):

- **Hand-rolled snippet extractor over `mktestdocs`**: the opt-out marker, shared-state ordering, and MyST nested-fence parsing *are* the contract; owning ~40-80 lines of extractor (fence parsing plus the tab-set tracking added at Amendment 7) beats pinning a micro-dependency whose parsing conventions would silently define a CI-critical contract. No new test dependency.
- **Matplotlib `Agg` backend forced in the snippet test**: CI is headless; snippets must not require a display.
- **All five optional-backend markers on the page-runner test** (`networkx`, `bokeh`, `datashader`, `holoviews`, `plotly`): the page's per-backend tabs touch every extra; marking the whole runner keeps backend-less environments green. Test env already installs all extras.
- **Opt-out marker is an HTML comment above the fence** (`<!-- no-run: <reason> -->`): visible in source and to the extractor, invisible in the rendered page; a required reason string keeps opt-outs rare and auditable. Render-invisibility is verified at WS-A's seed-section pass; the fallback shape is MyST's native `% no-run: <reason>` comment (Amendment 8).

## Naming audit

- Toctree caption: **"Quick Reference"** (recommended) over "User Guide". The section holds exactly one lookup page; pandas/numpy/matplotlib "User Guide" vocabulary promises narrative chapters that this repo's notebooks already own. Placed between the gallery and autodoc captions in `docs/source/index.rst`.
- Page title: **"Cheat Sheet"**, file `docs/source/cheat_sheet.md`. Direct ecosystem anchor: the matplotlib cheatsheets (the page's stated spirit reference); "quick reference" survives in the caption.
- Test file: `tests/docs_cheat_sheet_test.py`. The `*_test.py` mirror convention maps src files; this is the first docs-page test, named for its subject.
- Opt-out marker token: `no-run` (reads as the action it suppresses; `skip` collides with pytest vocabulary while meaning something different here).
- WS-C notebook: **`examples/saving_plots_for_publication.ipynb`** (recommended). "Publication-quality figures" is established matplotlib-ecosystem vocabulary; "exporting" is taken by the data-out notebooks (`exporting_hive_plots_to_networkx.ipynb`, `exporting_hive_plots_to_json.ipynb`) and would mis-shelve a figure-saving page.
- Prose-only terms: "cheat sheet", "snippet", "publication-quality". No new parameters, methods, or classes.

## API usage examples

The cheat-sheet page is itself an API-usage-examples artifact; the api-critic planning pass should review the representative snippets below (shape, headline idioms, tab/dropdown conventions, opt-out marker), not an exhaustive inventory. All snippets written against master's surface, re-verified 2026-07-06 against the execution worktree (`graph=` keyword-only at `src/hiveplotlib/hiveplot.py:2157`; `to_networkx` at `:1899`, `to_json` at `:1829`; `update_edge_plotting_keyword_arguments` at `:4309`).

### Proposed (from planner / Orchestrator)

```python
# Example 1: cheat-sheet entry "hive plot from a networkx graph" (headline build idiom)
# Example data:
import networkx as nx

g = nx.karate_club_graph()

# Call site:
from hiveplotlib import HivePlot
from hiveplotlib.viz import hive_plot_viz

hp = HivePlot(
    graph=g,
    node_graph_metrics="degree",
    partition_variable="club",
    sorting_variables="degree",
)
fig, ax = hive_plot_viz(hp)
```

```python
# Example 2: cheat-sheet entry "edge color / width / alpha", per-backend tabs
# (demonstrates the hiveplotlib-x-backend kwarg interaction term: linewidth vs line_width)
# Example data: hp from the page's setup dropdown (replayed per tab; see Design notes):
from hiveplotlib.datasets import example_hive_plot

hp = example_hive_plot()

# Call site (matplotlib tab) — bare kwargs update the default "all_edge_kwargs" setting:
hp.update_edge_plotting_keyword_arguments(color="darkblue", linewidth=0.5, alpha=0.3)
# Call site (bokeh tab — same task, bokeh vocabulary):
hp.update_edge_plotting_keyword_arguments(color="darkblue", line_width=0.5, alpha=0.3)
# Non-default settings go through edge_kwarg_setting=, with a kwarg disjoint from the
# default setting's kwargs (see WS-A authoring conventions), e.g.:
# hp.update_edge_plotting_keyword_arguments(edge_kwarg_setting="repeat_edge_kwargs", linestyle="--")
```

```python
# Example 3: cheat-sheet entries "data out"
# Example data: hp from Example 2's setup dropdown.

# Call site:
g_out = hp.to_networkx()
json_out = hp.to_json()
```

```markdown
<!-- Example 4: the opt-out marker convention (page source, not Python) -->
<!-- no-run: plotly fig.show() opens a browser tab; rendering is interactive-only -->
```

Implementation note for WS-A: Example 2's per-backend kwarg names must be verified against each backend's `rename_edge_kwargs` wiring (`viz/matplotlib.py:439` uses `linewidth`/`alpha`/`color`; `viz/bokeh.py:545` uses `line_width`/`alpha`/`color`; plotly uses `width`), and every snippet's call verified against master's actual signatures (Read the `def` line) at authoring time. The snippet test catches run-failure drift (removed names, changed required arguments) by construction; it does **not** catch run-green misuse — `update_edge_plotting_keyword_arguments(all_edge_kwargs={...})` was exactly that, silently absorbed via `**kwargs` and merged by `|=` (`hiveplot.py:4309-4330`) — so signature verification is a separate authoring-time and api-critic obligation, not something the harness provides (Amendment 7, item 1).

### API Critic's take (planning mode)

Reviewed 2026-07-06 against the execution worktree (`~/repos/hiveplotlib-56-docs-cheat-sheet`, master `dff7ead`). Signature-check performed per Amendment 7 item 1: every entry point's `def` line Read in source, not vocabulary-matched.

**Signature-check results (all clean):**

- Example 1: `HivePlot(graph=..., node_graph_metrics=..., partition_variable=..., sorting_variables=...)` matches `hiveplot.py:2152-2187` — `graph` keyword-only (after the `*` at `:2156`), `partition_variable` / `sorting_variables` required keyword-only, `node_graph_metrics: Optional[Union[str, Sequence[str]]]` so the string `"degree"` is valid; the idiom is verbatim-consistent with `examples/creating_hive_plots_from_networkx.ipynb`'s headline cell (karate club, `club` partition, computed-`degree` sort). `fig, ax = hive_plot_viz(hp)` matches the `Tuple[plt.Figure, plt.Axes]` return at `viz/matplotlib.py:508-526`.
- Example 2: bare kwargs land in `**kwargs` at `hiveplot.py:4309-4315` and merge via `|=` (`:4330`) — the amended snippet now teaches the real convention. matplotlib vocabulary `linewidth`/`alpha`/`color` confirmed at `viz/matplotlib.py:439-441`; bokeh `line_width`/`alpha`/`color` confirmed at `viz/bokeh.py:545-547` and flows to `fig.multi_line(**final_edge_kwargs)`, so `line_width` is the correct user-facing name. `edge_kwarg_setting="repeat_edge_kwargs"` is a valid hierarchy literal (`:2230-2236`). No overlap warning fires from the two amended calls on a fresh `example_hive_plot()` (all five hierarchy dicts start empty; per-tab replay keeps the two vocabularies on separate `hp` objects).
- Example 3: `to_networkx()` (`hiveplot.py:1899`, keyword-only params all defaulted) and `to_json()` (`:1829`, no params) both callable bare; both live on `BaseHivePlot` and are inherited.
- Setup helpers: `example_hive_plot()` (`datasets/toy_hive_plots.py:360`) and `example_multi_tag_hive_plot()` (`:637`) both zero-arg-callable returning `HivePlot`.
- Amendment 5 scope guard: no narwhals/polars/Dask/cuDF idiom in any snippet. Clean.

**Verdict: Agreed on Examples 1-3 as amended and on the per-tab-replay contract** (it kills the cross-backend pollution shape by construction, and no backend-mismatch warning exists on the `HivePlot` viz path — `self.backend` only gates `HivePlotMatrix` viz — so tabs need no `set_viz_backend` boilerplate). Findings below.

- **[must-fix] The "Figures out" embedding entry names a parameter that does not exist: `axes=`.** Design notes inventory item 6 (and the Alignment rejected-bullet it quotes) says 'pass `axes=` to embed in an existing figure'; the real matplotlib signature is `hive_plot_viz(hive_plot, fig=None, ax=None, ...)` (`viz/matplotlib.py:508-511`), and no viz function in any backend has an `axes` parameter. Worse, the trailing `**edge_kwargs` (`:525`) silently absorbs `axes=` and forwards it to `LineCollection` — the exact run-green-misuse shape Amendment 7 item 1 just purged from Example 2, now sitting in the inventory the page will be written from. Suggested change: the entry reads "pass `fig=` and `ax=`" (orchestrator one-word amend to Design notes item 6; the Alignment quote can stand as historical record).
- **[worth-discussing] Keep the non-default-setting example's kwarg disjoint from the default setting's kwargs.** Example 2's commented line reuses `alpha`, which overlaps `all_edge_kwargs`' `alpha` and is the documented conflict-warning trigger; under the runner's warnings-as-errors, an executed version risks failing (or, worse, silently depends on the warning not firing — see the low-confidence gate note below). Suggested change: when WS-A turns that line into a runnable snippet, use a kwarg not present on the default setting (e.g. `linestyle="--"` on `repeat_edge_kwargs`); hierarchy-conflict demonstrations stay in the linked `edge_kwarg_hierarchy.ipynb` per the plan's own link-don't-re-teach bullet. Also: the plan's Example 2 block compresses two tab-items into one fence for review purposes — WS-A must land them as separate tab-items, never one block (one block is the pollution case the replay contract exists to prevent).
- **[worth-discussing] "Figures out" will be the `no-run`-dense section; pre-decide the runnable-variant preference.** `pyproject.toml` carries no kaleido and no selenium, so plotly static export (`fig.write_image`) and bokeh SVG/PNG export (`export_svgs` / `export_png`) are genuinely non-runnable in the test env. Suggested change: headline the runnable exports per backend (matplotlib `fig.savefig("hive_plot.svg")`, plotly `fig.write_html(...)`, bokeh `save(...)`), reserve `no-run` for the system-dependency lines, and adopt a marker-granularity convention: a `no-run` block contains only the non-runnable line(s), with the buildable code in a preceding tested block (Example 4's plotly-tab-wide opt-out would otherwise exempt the entire plotly path from the per-tab guarantee).
- **[worth-discussing] The collapsed setup dropdown is the one reader path the tab contract doesn't model.** The contract guarantees "main-stream prefix + one tab" runs, but the prefix includes dropdown content a skimming copy-paster never expanded, so their first experience is `NameError: name 'hp' is not defined`. Suggested change: dropdown labels state what they define ("Setup: build `hp`"), so the NameError resolves at a glance; cheap authoring convention, no harness change.
- **[low-confidence] Verify the HTML-comment marker actually disappears under this repo's myst-parser config.** `myst_parser` is already in `docs/source/conf.py:38` with no raw-HTML extensions enabled; depending on myst-parser's HTML handling, `<!-- no-run: ... -->` may render as escaped literal text rather than vanishing. Verify during WS-A's seed-section pass; the fallback marker shape is MyST's native comment `% no-run: <reason>` (guaranteed non-rendering, equally greppable). Pick one shape after verification — the extractor should support exactly one.
- **[low-confidence] The conflict-warning gate at `hiveplot.py:4247-4248` reads inverted** (`if warn_on_overlapping_kwargs is not None: warn_on_overlapping_kwargs = self.warn_on_overlapping_kwargs` — a `None` default stays `None`/falsy, so the documented overlap warning may never fire via `update_edge_plotting_keyword_arguments`). Master behavior, out of this plan's scope; noted so the snippet-warning risk above is calibrated (the warning may not fire today, but a page snippet must not depend on a bug) and as a candidate follow-up issue.
- **[low-confidence] Two headline-inventory taste calls for authoring time:** (a) consider a one-line hover entry (bokeh `hive_plot_viz(hp, hover=True)`-style; hover is the reason users pick the interactive backends, and it's a plausible top-five lookup); (b) the page is hive-plot-scoped with P2CP absent — presumably deliberate, but say "hive plots" in the page's one-line intro so a P2CP user knows immediately they're on the wrong page.

**Recurring pattern:** the library's plotting surface is `**kwargs`-permissive at both levels (`update_edge_plotting_keyword_arguments(**kwargs)`, `hive_plot_viz(**edge_kwargs)`), so wrong kwarg names run green all the way to the renderer — this plan has now hit that shape twice (original Example 2, the `axes=` inventory entry). Every kwarg name on the page needs a def-line check at authoring time; the snippet runner structurally cannot backstop this class of error.

### API Critic — post-implementation review

No library API surface changes in this plan. The post-impl review applies to the shipped cheat-sheet page (`docs/source/cheat_sheet.md`) as a usage-surface artifact. Reviewed 2026-07-06 against the execution worktree (`~/repos/hiveplotlib-56-docs-cheat-sheet`, branch 56 off master `dff7ead`). Every kwarg on the full page was signature-checked against the worktree `def`/rename wiring, not vocabulary-matched, because the snippet runner (`tests/docs_cheat_sheet_test.py:12-15,356-361`) explicitly does not validate kwarg correctness: the `**kwargs`-permissive plotting surface forwards a wrong name to the backend and runs green.

```
Status: clean
API surface reviewed: cheat_sheet.md snippets — HivePlot(graph=, node_graph_metrics=, partition_variable=, sorting_variables=), HivePlot(nodes=, edges=, partition_variable=, sorting_variables=), NodeCollection(data=, unique_id_column=), Edges(data=), example_hive_plot(axes_order=, rotation=, repeat_axes=, angle_between_repeat_axes=, collapsed_group_axis_name=), update_node_viz_kwargs(c=/color=, s=/size=, alpha=), update_edge_plotting_keyword_arguments(color=, linewidth=/line_width=, alpha=, edge_kwarg_setting=, zorder=, linestyle=), hive_plot_viz(fig=, ax=), to_networkx(), to_json()
Concerns:
  - none (must-fix, worth-discussing, or low-confidence)
Test-method-coverage audit: clean — this workstream added a docs page, not library methods; no new `test_<method>_*` contract applies. The page-execution test (`docs_cheat_sheet_test.py`) exercises snippet extraction/replay, and every snippet runs in the `test_cheat_sheet_page_runs` reader-path sweep.
```

**Signature-check results (all real, per-section walk):**

- **Data in / networkx.** `HivePlot(graph=g, node_graph_metrics="degree", partition_variable="club", sorting_variables="degree")` — all keyword-only after `hiveplot.py:2156`'s `*`; verified `node_graph_metrics="degree"` resolves to a metric column literally named `"degree"` (`graph_features/__init__.py:748`, `column_name = rename.get(metric_name, metric_name)`), so `sorting_variables="degree"` references a real column end-to-end. `"club"` comes from `karate_club_graph`. `fig, ax = hive_plot_viz(hp)` matches the `Tuple[plt.Figure, plt.Axes]` return (`viz/matplotlib.py:508-526`).
- **Data in / pandas.** `NodeCollection(data=node_df, unique_id_column="id")` (`node.py:132-138`, `"id"` matches the df) and `Edges(data=edge_df)` (`edges.py:99-108`, defaults `from_column_name="from"`/`to_column_name="to"` match the df columns). Page prose ("unique id column; `from` and `to` columns") matches the defaults exactly.
- **Nodes.** matplotlib `update_node_viz_kwargs(c="low", s=40, alpha=0.6)` and bokeh `(color="low", size=8, alpha=0.6)` — kwargs land in `**node_viz_kwargs` (`node.py:279-283`) and flow to `ax.scatter(**final_scatter_kwargs)` (`viz/matplotlib.py:327`, native names `c`/`s`/`alpha`) / `fig.scatter` (`viz/bokeh.py:370-372`, native names `color`/`size`/`alpha`). `"low"` is a real node data column (`toy_hive_plots.py:151`); the string-column-vs-literal branch (`viz/matplotlib.py:313-319`) matches the page's "string names a column, else used as-is" prose.
- **Edges.** matplotlib `(color="darkblue", linewidth=0.5, alpha=0.3)` + `zorder=5` + `linestyle="--"` all land in `**kwargs` (`hiveplot.py:4309-4315`) and flow verbatim to `LineCollection(**final_edge_kwargs)` (`viz/matplotlib.py:499-502`) — every one a native `LineCollection` kwarg. bokeh `(color="darkblue", line_width=0.5, alpha=0.3)` flows to `fig.multi_line(**final_edge_kwargs)` (`viz/bokeh.py:585-591`); `edge_viz_preparation` (`viz/base.py:174-207`) only *fills defaults*, it never renames user keys, so `line_width` reaches `multi_line` as the correct bokeh name (and `color`/`alpha` are accepted bokeh glyph shorthand). `edge_kwarg_setting="repeat_edge_kwargs"` is a valid hierarchy literal (`hiveplot.py:2230-2236`). The two main-stream edge calls (`linestyle` on `repeat_edge_kwargs`, `zorder` on default `all_edge_kwargs`) set disjoint keys on a fresh `hp`, so no overlap-conflict warning fires under warnings-as-errors — the planning-pass worth-discussing on this is resolved as shipped.
- **Axis layout.** `example_hive_plot(axes_order=..., rotation=..., repeat_axes=..., angle_between_repeat_axes=..., collapsed_group_axis_name=...)` forwards through `**hive_plot_kwargs` (`toy_hive_plots.py:371,461`) to the `HivePlot` constructor, where all five are real params (`hiveplot.py:2161-2173`). Axis names `"A"/"B"/"C"` match the helper's default `labels=["A","B","C"]` (`toy_hive_plots.py:365`), so `axes_order=["B","A","C"]` and `["A","B",None]` reference real axes.
- **Data out.** `to_networkx()` (`hiveplot.py:1899`, all params defaulted) and `to_json()` (`:1829`, no params) both callable bare, both inherited from `BaseHivePlot`.
- **Figures out.** `from hiveplotlib.viz import hive_plot_viz` = matplotlib, returns `(fig, ax)` → `fig.savefig(...)` native. plotly `hive_plot_viz` returns `go.Figure` (`viz/plotly.py:819`, single object) → `fig.write_html(...)` native. bokeh `hive_plot_viz` returns `figure` (`viz/bokeh.py:655`, single object) → `output_file`/`save` from `bokeh.plotting`. The embed entry uses `fig=fig, ax=axes[0]` (real params, `viz/matplotlib.py:510-511`), **not** the nonexistent `axes=` the planning-pass flagged as must-fix — that fix landed. `savefig`/`write_html`/`save` are all real writers on their respective figure types.

**Scope guard:** no narwhals/polars/Dask/cuDF idiom in any snippet; the whole page is pandas/networkx/matplotlib-bokeh-plotly. Clean, no must-fix.

**Tab / no-run conventions (copy-paster view):** each backend pair ships as two independent `{tab-item}` fences under one `{tab-set}` (Nodes, Edges, Figures out); the per-tab replay contract (`docs_cheat_sheet_test.py:190-203`) runs each tab against the main-stream prefix only, so no sibling-backend line leaks. `no-run` is used exactly once (the `plt.show()` interactive-window line, with a mandatory reason); no path is broadly exempted from the runner. The planning-pass worry about a plotly-tab-wide opt-out did not materialize — the "Figures out" plotly/bokeh tabs headline runnable exports (`write_html`, `save`) with no `no-run`. Dropdown labels state what they define ("Setup: build `hp`"), so the collapsed-prefix `NameError` risk is signposted as recommended.

**Verdict:** clean. The planning-pass must-fix (`axes=`) and both edge/plotly worth-discussing items are resolved in the shipped page. No new concerns.

## Notebook review

Applies to WS-C only (the cheat sheet is a docs page, not a notebook).

### Editorial-critic — post-implementation

Reviewed `examples/saving_plots_for_publication.ipynb` (branch 56 execution worktree) as a whole artifact against the `hiveplotlib-gallery-notebook` skill. Structure / scope / genre / coherence only; the figures are viz-critic's lane.

```
Status: clean
Notebook reviewed: examples/saving_plots_for_publication.ipynb, genre gallery, class HivePlot
  (viz/export mechanics across the matplotlib and datashader back ends)
Concerns:
  - none (must-fix or worth-discussing)
```

The editorial bar, section by section: gallery genre clean (noun-phrase title, what-not-why lead-in, instructional voice, no rhetorical questions, default→advanced ordering, leans on `example_hive_plot()`; the datashader default→too-fat→rebalanced arc is the skill's endorsed incremental demonstration). Every section earns its place (Vector Formats, Raster DPI, the datashader coupling, and the nested pin-when-comparing subsection all inside figure-saving-mechanics; the pin subsection is a genuine second reason — density-color comparability — not padding). Dataset coherence: one helper throughout, the small matplotlib plot and the 1000/5000 datashader network are two sizings of `example_hive_plot`, the larger-network switch is named in prose, matches datashader.ipynb's data; no new datasets. Stays on `HivePlot`, three-axis default inside the 2-3 axis invariant. Intro matches body. Cross-links resolve and aim well (all three targets exist; the `#Balancing-Pixel-Spread-and-DPI` anchor maps to the real `### Balancing Pixel Spread and DPI` heading; closing cell uses the canonical prose-pointer form). Pre-answered datashader.ipynb overlap confirmed as genuine consolidation, Holdout untouched, no deduplication finding raised.

### Viz-critic — post-implementation

Reviewed the rendered figures of `examples/saving_plots_for_publication.ipynb` (branch 56 execution worktree) against the viz-quality-bar skill.

```
Status: propose
Notebook reviewed: examples/saving_plots_for_publication.ipynb — DPI/pixel-spread figures across
  the matplotlib and datashader back ends
Concerns:
  - two low-confidence (nothing blocking)
```

Core asks verified: the DPI/pixel-spread story is genuinely SHOWN — orange node-bar thickness normalized by image height measures 1.06% (dpi=150) → 3.55% (dpi=50, stale spreads, ~3.3x fatter = the ballooning) → 1.18% (dpi=50, lowered spreads = rebalanced). The pinned side-by-side (cell 16) correctly pins dpi, both pixel spreads, and vmax_nodes/vmax_edges — not a confounded comparison. Datashader cmap defaults accepted. Accessibility passes (copper-orange vs blue holds in grayscale). Polish is instructional/spare, correct for a gallery export notebook.

- **[low-confidence]** The DPI trilogy (cells 9/11/13) renders at mismatched DISPLAY sizes (~3x, inherent to dpi driving raster dimensions). The 11-vs-13 rebalancing pair is airtight (identical size); the 9-vs-11 ballooning comparison is legible on proportional inspection but slightly muddied by the display-size jump. Optional: a one-line aside in cell 10 noting the lower-dpi figure also renders smaller. Do NOT normalize figsize (would change the bin count and undercut the lesson).
- **[low-confidence]** In cell 16, `dpi=150` is listed in `shared_kwargs`, but on the pre-supplied fig/ax path `fig.dpi` (from `plt.subplots`) drives the bin count, not the `dpi` kwarg. They agree here (both 150) so the figure is correct and comparable; a reader copying the pattern and changing only the `shared_kwargs` dpi would see no effect. Teaching-accuracy wrinkle, routes as a prose call.

## Adversary review

Cold-context dissent against the plan and the artifact it ships. This plan predates the adversary sections in the template; added at the 2026-07-06 cold pass.

### Adversary's challenge (planning mode)

```
Status: challenge (6 items)
Plan reviewed: wiki/wiki/plans/docs-cheat-sheet-and-readme.md (cold; grill knowingly skipped
  per Amendment 6, so no post-grill rubric-check will follow — items route to orchestrator
  amend-plan for explicit disposition per the execution-time record)
Angles worked: premise — clean (verified against master worktree dff7ead: README top has no
  when/why framing, just logo + one-liner + badges + Installation, so WS-B's gap is real; WS-A
  and WS-C trace to named workflows from the 2026-06-10 docs-gaps discussion). Approach — clean
  on alternatives (notebook / doctest / literalinclude / mktestdocs / FAQ / subplot-notebook
  rejections all recorded with reasons; unusually thorough); items 1-2 attack the chosen
  harness's contract, not its selection. Size-and-maintenance — no existential item; hand-rolled
  extractor carries the named justification the ecosystem-tools rule requires, sphinx-design is
  docs-extra-only, WS-C's deliberate duplication is pinned by Holdouts.
Items:
  - [must-fix] Example 2 misuses `update_edge_plotting_keyword_arguments` against master's
    actual signature — at API usage examples, Example 2 (and the WS-A implementation note)
    Rubric: no rubric yet — pre-grill
    Push: master (`hiveplot.py:4309`) takes `edge_kwarg_setting: str = "all_edge_kwargs"` plus
    `**kwargs`; there is no `all_edge_kwargs=` keyword. As written, the call merges a kwarg
    literally named `all_edge_kwargs` (dict-valued) into the default setting via `|=`
    (`hiveplot.py:4330`) and raises nothing — the snippet runs green while teaching a wrong
    idiom, and only fails if a later shared-state snippet renders that hp. This falsifies the
    plan's own claim that "the snippet test catches any drift by construction": the harness
    catches run-failures, not run-green misuse. Rewrite Example 2 to the real convention
    (bare kwargs for the default setting, e.g. `hp.update_edge_plotting_keyword_arguments(
    color="darkblue", linewidth=0.5, alpha=0.3)`; `edge_kwarg_setting=` for non-default
    settings), soften the by-construction claim in Design notes, and have the pending
    api-critic planning pass verify every representative snippet against master signatures,
    not just kwarg vocabulary.
  - [must-fix] The binding correctness contract is unsound for tabbed content: sibling
    tab-items are parallel alternatives a reader never sees together, but the extractor runs
    them in flattened sequence on shared state — at Design notes, snippet-test harness
    Rubric: no rubric yet — pre-grill
    Push: two concrete failures follow. (a) Cross-backend kwarg pollution: Example 2's
    matplotlib `linewidth` and bokeh `line_width` merge onto the same `hp` (`|=`,
    `hiveplot.py:4330`), so any later render receives the other backend's vocabulary.
    (b) A tab snippet can lean on a sibling tab's import or state: test green, single-tab
    copy-paste broken — copy-paste-ability, the plan's stated point, is then not what the
    test verifies for tabbed content. Pin a binding tab convention before WS-A dispatch
    (each tab-item self-contained given only non-tab content above it: per-tab variables or
    re-create from the setup dropdown; optionally have the extractor fork/reset namespace per
    tab-set), or explicitly record the weaker flattened-sequence guarantee and drop the
    "correct iff it runs given what is visible above it" phrasing.
  - [worth-discussing] `docs/source/_llms/llms.txt` is absent from every Files list despite
    the standing sync trip-wire — at Workstream A / Workstream C Files
    Rubric: no rubric yet — pre-grill
    Push: the cheat sheet is plausibly the strongest new index candidate in this plan (a
    task-oriented, CI-tested lookup page; the index was rebalanced for agents driving the API
    blind). Under auto-dispatch with the grill skipped, the omission surfaces late as a qa
    taste call. Pre-decide now: add the llms.txt entry to WS-A's Files/done-when
    (docs-engineer-owned), and record an explicit yes/no for WS-C's notebook (feature demo →
    `## Optional`, or no entry).
  - [worth-discussing] Auto-dispatch plus one-commit-per-workstream on a taste-heavy docs plan
    means WS-C's editorial/viz taste findings (worth-discussing, no downstream bearing) batch
    to plan-end after WS-C's commit already exists — at Alignment (grill), execution-time
    record / Amendment 6
    Rubric: no rubric yet — pre-grill
    Push: not relitigating the directive (the override is recorded); flagging its unnamed
    consequence. The harness fit guidance keeps pauses on doc notebooks precisely because
    churn is expected there. Either record explicitly that follow-up commits on WS-C are
    accepted churn, or exempt WS-C from auto-dispatch (one pause before its commit).
  - [low-confidence] Plan-body `file:line` drift extends beyond the three cites Amendment 4
    names — at Feasibility audit
    Rubric: no rubric yet — pre-grill
    Push: verified `example_multi_tag_hive_plot` sits at `toy_hive_plots.py:637` on master,
    not the plan's `:607`. Make the authoring-time re-verify obligation plan-wide (Feasibility
    audit and Design notes cites included), not just WS-A's snippet-surface bullet.
  - [low-confidence] WS-B's Files list has a code workstream editing a different plan's body
    (`edge-coverage-and-gotchas-docs.md` supersession note) — at Workstream B Files
    Rubric: no rubric yet — pre-grill
    Push: plan files are orchestrator-owned surfaces under the write choreography. Have the
    orchestrator write that one-liner at amend time (as Amendment 3 already did for
    `joss-submission.md`) and drop it from WS-B's Files.
```

### Adversary post-impl

```
Status: propose
Artifact reviewed: WS-A (cheat-sheet page + snippet-test harness), branch 56 off master dff7ead,
  attacked blind against the done-whens and binding conventions before reading this plan
Dispositions held: yes — every planning-challenge disposition landed in the shipped artifact
  (see reconcile below). The two must-fixes are a gap the challenge did not name, not a
  reversed disposition.
Concerns:
  - [must-fix] "Data out" cannot be grabbed and run on its own — at
    docs/source/cheat_sheet.md:176-179
    Rubric: no entry (blind pass; no Failure-modes rubric on this plan)
    The block is `g_out = hp.to_networkx()` / `json_out = hp.to_json()` with no import, no
    `hp = ...`, and no "Setup: build hp" dropdown. Every other hp-dependent section defines hp
    inline (Data in, Axis layout) or ships a labeled setup dropdown (Nodes, Edges, Figures out).
    Failure scenario: a reader follows the page's own intro ("grab a section and copy it out"),
    selects the Data out section, pastes it into a fresh REPL/script → NameError: name 'hp' is
    not defined. The harness cannot catch it: test_cheat_sheet_page_runs executes all main-stream
    blocks in one shared namespace, so hp is always defined from Axis layout by the time Data out
    runs. Fix: a `:::{dropdown} Setup: build hp` matching the sibling sections, or an inline
    `hp = example_hive_plot()`.
  - [must-fix] The "Drop a hive plot into an existing matplotlib figure" embed block has the same
    missing-hp defect — at docs/source/cheat_sheet.md:238-245
    Rubric: no entry
    The block imports plt and hive_plot_viz and calls `hive_plot_viz(hp, fig=fig, ax=axes[0])`
    but never defines hp. It is a bare main-stream block after the "Figures out" tab-set, visually
    detached from that section's setup dropdown. Failure scenario: a reader grabs just this block
    → NameError: name 'hp' is not defined. Same harness blind spot (shared main-stream namespace).
    Fix: same as above.
  - [low-confidence] Collapsed setup dropdowns are a softer version of the same trap — at
    docs/source/cheat_sheet.md:75-83, 111-119, 187-195 (Nodes / Edges / Figures out)
    Rubric: no entry
    A skimmer who copies a tab without expanding the dropdown hits NameError: hp. Distinguished
    from the two must-fixes because the dropdown is present and labeled "Setup: build hp", and
    "setup dropdowns collapsed and labeled with what they define" is an explicit WS-A done-when —
    accepted contract, not a defect. Flagged only so the disposition is on record; no change
    needed unless the label should read more imperatively.
  - [low-confidence] llms.txt "backed by a CI-tested example" reads slightly stronger than the
    harness guarantees — at docs/source/_llms/llms.txt:47
    Rubric: no entry
    The harness proves runnability, not API correctness (its own module docstring says so, and
    Amendment 7 item 1 purged exactly this overclaim from the plan). "CI-tested" is defensible
    (the snippet is run in CI), so this is a wording nit, not a correctness overclaim. Optional:
    "CI-run" instead of "CI-tested".

Reconcile (plan-section-as-memory against the planning challenge + Amendments 7-8):
  - Planning must-fix 1 (Example 2 signature misuse) → Amendment 7 item 1 (rewrite to bare kwargs
    + edge_kwarg_setting=). HELD: the shipped Edges section uses bare color/linewidth/alpha and
    edge_kwarg_setting="repeat_edge_kwargs"; no all_edge_kwargs=-as-keyword idiom survives. Every
    backend kwarg on the page checks out against live signatures (matplotlib c/s/linewidth/zorder,
    bokeh color/size/line_width) — the def-line check obligation was honored.
  - Planning must-fix 2 (sibling-tab pollution / dependency) → Amendment 7 item 2 (per-tab-replay
    contract). HELD: _replay_prefix_for_tab + _run implement main-stream-shared / tab-replayed-in-
    fresh-namespace exactly; test_extract_snippets_flags_sibling_tab_variable_as_tab_scoped proves
    bokeh line_width never leaks into the matplotlib replay. The blind pass confirms the challenged
    intra-tab-set failure modes are closed.
  - Planning items 3 (llms.txt) and 6 / Amendment 8 item 1 (axes= not real → fig=/ax=): HELD.
    llms.txt entry present, pinned high, absolute URL; the embed snippet uses the real fig=/ax=
    surface. Scope guard clean (no narwhals/polars/Dask/cuDF), no process references, CHANGELOG in
    the unreleased 0.29.0 section.
  - The gap: Amendment 7 item 2's contract resolves leakage *within* a tab-set. My two blind
    must-fixes are cross-section main-stream dependence (a non-tab section and a non-tab block that
    lean on hp from an earlier section), which the shared main-stream namespace is designed to
    permit. So the dispositions held; a per-section copy-paste breach the challenge never named
    slipped through the harness's one blind spot. Both are downstream-inert (WS-A-local page edits,
    no bearing on WS-B/WS-C) — route to amend-plan.
```

**WS-B (README leading text):**

```
Status: clean
Artifact reviewed: WS-B (README leading text, added lines 5-10 of both README.md and
  docs/source/README.md), branch 56 execution worktree, attacked blind against the done-whens
  and attack angles before reading this plan
Dispositions held: yes — planning item 6 (WS-B Files editing edge-coverage-and-gotchas-docs.md's
  body) landed as intended: the orchestrator wrote that supersession note at Amendment 7 and
  dropped it from WS-B's Files, so the shipped WS-B diff is exactly the two README files and
  nothing else. My other five planning items were WS-A-centric; none bore on WS-B.
Concerns: none rise to must-fix or worth-discussing. Every done-when and every named attack angle
  is clean:
  - No overclaim/false statement. "Each node sits at a position determined by its own properties
    (degree, a partition label, any value you choose)" matches axis.py set_sorting_variable (a
    chosen scalar node value places each node); determinism and the force-directed contrast hold.
  - Voice clean in the added region: no em-dash, no AI filler, no throat-clearing, ~5 sentences
    (not a manifesto). The only em-dash in either file is the pre-existing Krzywinski citation
    (README line 78), an expected survivor outside the changed region.
  - Sync: the top framing region is byte-identical across both copies; the files diverge only
    below (absolute readthedocs URLs vs relative rst/ipynb paths), as the survivors list allows.
  - Scope guard clean: no narwhals/polars/Dask/cuDF/dataframe-input mention, no gotchas-page
    pointer, no process reference. CHANGELOG correctly untouched (no README-prose entry added).
  - Pointer resolves: the Introduction to Hive Plots notebook exists and "linked below" is
    satisfied by the surviving "How to Use and Examples" / "More on Hive Plots" pointers.
Two low-confidence nits recorded for completeness (neither blocking; no change recommended):
  - [low-confidence] "linked below" is a soft pointer, not an inline link — at README.md:10.
    Failure scenario: a reader takes "linked below" as "immediately below", hits the badge block
    and Installation section (~40 lines) before the first Introduction link, and assumes it is
    missing. Satisfied as written because the done-when leans on the surviving pointers.
  - [low-confidence] "the same data always draws the same picture" elides the layout spec — at
    README.md:6-7. Failure scenario: a pedant reads it as absolute and objects that changing the
    sorting variable redraws the picture. The preceding clause ("any value you choose") already
    frames the spec as part of the input, so a fair reader reads "same data + same choices".
```

**WS-C (saving-for-publication gallery notebook):**

```
Status: propose
Artifact reviewed: WS-C (examples/saving_plots_for_publication.ipynb + thumbnail + docs
  registration/cross-links + CHANGELOG/llms.txt entries), branch 56 execution worktree, attacked
  blind against the done-whens and binding scope before reading this plan
Dispositions held: yes — all six of my planning-challenge items were WS-A-centric (disposed at
  Amendment 7); none bore on WS-C, and none reappeared as a WS-C scope breach. No must-fix.
Concerns:
  - [low-confidence] Could not visually confirm the datashader panels SHOW ballooning/rebalancing
    (only that they are real and the code is correct) — CLOSED by viz-critic's same post-impl
    pass (node-bar thickness measured 1.06% @dpi=150 → 3.55% @dpi=50 stale spreads → 1.18% @dpi=50
    lowered spreads; the effect is genuinely shown). Failure scenario it guarded against: titles
    claiming "too fat" / "back in balance" over figures that don't actually move. No longer open.
  - [worth-discussing] Deep anchor datashader.ipynb#Balancing-Pixel-Spread-and-DPI may not resolve
    in built HTML — at the notebook's closing cross-links. Target heading exists (datashader.ipynb
    JSON:1012) and the hyphenated-title convention matches sibling links, so it likely resolves,
    but nbsphinx `###`-heading slugs are a known fragility and a broken intra-doc anchor is not a
    `make docs` warning. BATCHED to the WS-C qa gate for built-HTML verification. Failure scenario:
    the deep link silently 404s while the build stays green. Downstream-inert (notebook-local link).
  - [low-confidence] The pinned side-by-side compares two near-identical networks (seed=1 vs seed=2,
    both 1000/5000 from the same generator), so the "colors mean the same thing" payoff is shown
    mechanically, not as a striking difference — at the "Pin the Parameters When Comparing Figures"
    cell. The comparison is correctly pinned (dpi + both spreads + vmax_nodes/vmax_edges), so this
    is NOT the confounded-comparison finding; it is a pedagogy-vs-punch maintainer judgment call.
    Failure scenario: a reader sees two identical panels and misses why pinning mattered. No change
    required; recorded so the call is on the record.
Angles worked and cleared (not findings): scope-guard held (no stream/chunk/memory/single-shot
  datashader-internals language; the DPI→bin-count→pixel-spread geometry story is library-accurate
  per viz/datashader.py:159-165); master-API-surface only (no narwhals/polars/Dask/cuDF); no
  subplot-embedding section (the fig=/ax= one-liner lives in the cheat sheet per scope); no new
  dataset (example_hive_plot with a valid backend= passthrough); voice clean (no em-dash / AI
  filler in the notebook or the diffed docs prose); registration present in both nblinkgallery and
  toctree under Visualization (no orphan); thumbnail asset exists and renders a valid datashaded
  hive plot; CHANGELOG 3-line entry at correct altitude under Documentation, under the four-line
  cap; llms.txt entry placed under Visualization backends; no process references; the deliberate
  datashader.ipynb overlap respected as an expected survivor.

Reconcile (plan-section-as-memory against the planning challenge + Amendments 7-9):
  - My six planning-challenge items (Example 2 signature misuse, sibling-tab pollution, llms.txt,
    and the axes=→fig=/ax= correction) were all WS-A-centric and disposed at Amendment 7 (five)
    and Amendment 8 (item 1). None named WS-C, and the WS-C artifact introduces none of those
    failure shapes: the fig=/ax= surface the correction landed on is exactly what the notebook's
    comparison cell uses, and the scope guard that Amendment 7 wrote into WS-C's binding scope
    (no datashader-streaming internals) held in the shipped notebook. Dispositions held; the only
    open item routes to the WS-C qa gate (deep anchor), the rest is closed or judgment-call.
```

## Design notes (binding)

### Snippet-test harness (WS-A)

- **Contract (tab-aware; Amendment 7):** `tests/docs_cheat_sheet_test.py` reads `docs/source/cheat_sheet.md` and partitions fenced Python blocks into the **main stream** (every block outside any `{tab-set}`; dropdown content counts as main stream) and per-`{tab-item}` **tab groups**. The main stream `exec`s in page order in one shared namespace. Each tab-item runs in a fresh namespace built by replaying the main-stream blocks above its tab-set, then its own blocks in order; tab blocks never execute in the main namespace. Correctness contract: a main-stream snippet is correct iff it runs given the main-stream content above it; a tab snippet is correct iff it runs given the main-stream content above it plus that tab alone. Replay (not namespace forking) is deliberate: a forked dict still shares mutable objects, so sibling tabs would cross-pollute one `hp`; replay gives each tab its own objects, and keeping tab blocks out of the main namespace means later page content cannot silently depend on any one tab. `:sync:` keys are presentational only — same-sync tabs across tab-sets must not chain state (a cheat-sheet reader jumps straight to one entry). No nested tab-sets (the extractor may hard-error on them). One runner test for the whole page, carrying all five optional-backend markers.
- **What the harness guarantees (and does not):** copy-paste runnability along every reader path (main-stream prefix + one tab). It does **not** catch run-green misuse (kwargs silently absorbed by `**kwargs` — the Example 2 failure shape) or rendered-output correctness; those belong to authoring-time signature checks and the api-critic passes.
- **Extractor (hand-rolled, ~40-80 lines):** a fence opens on a line matching three-or-more backticks followed by `python` and closes on a matching-length backtick run. **MyST gotcha this must handle:** code fences nested inside sphinx-design directives (`{tab-set}`, `{tab-item}`, `{dropdown}`) sit inside *longer* backtick (or colon) fence runs; the extractor matches fence-run lengths rather than assuming bare triple-backticks, and ignores non-Python fences (`bash`, `markdown`, directive bodies). It additionally tracks which `{tab-set}`/`{tab-item}` a Python fence sits inside, driving the tab-aware contract above (incremental on top of the fence-run tracking; the ownership justification in Default justifications is unchanged).
- **Opt-out:** an HTML comment `<!-- no-run: <reason> -->` on the line(s) immediately above a fence excludes that block. The extractor requires a non-empty reason; the test fails on a bare `no-run`. Used sparingly (genuinely non-runnable entries only, e.g. interactive `fig.show()`). **Marker shape verified at the seed-section pass (Amendment 8):** if the HTML comment renders as literal text under this repo's myst-parser config, the marker becomes MyST's native comment `% no-run: <reason>` instead; exactly one shape ships, and the extractor supports only the shape that lands.
- **Headless:** the test forces `matplotlib.use("Agg")` (or `MPLBACKEND=Agg`) before executing any block, and closes figures after the run to keep `-n 7` memory sane.
- **Rationale to record (future ADR candidate):** testability == copy-paste-ability; motivated by the untested-matplotlib-cheat-sheet bug story and the recurring stale-snippet pattern (`same-stats-different-graphs.md:183`).

### Cheat-sheet page (WS-A)

- MyST markdown at `docs/source/cheat_sheet.md`; new toctree block in `docs/source/index.rst` between the gallery and autodoc captions. `sphinx_design` added to `extensions` in `docs/source/conf.py` and `sphinx-design` to the `docs` extra in `pyproject.toml` (docs-only dependency; the snippet test does not import it).
- Tabs: `{tab-set}` with `:sync:` keys for backend tabs (`matplotlib` / `bokeh` / `holoviews` / `plotly` / `datashader` as applicable) so one backend pick switches the whole page; tabs also carry alternate-example variants where a task needs different data. Setup code in collapsed `{dropdown}` blocks, 1-2 lines, leaning on `hiveplotlib.datasets` helpers (`example_hive_plot()`, `example_multi_tag_hive_plot()`) and `nx.karate_club_graph()`.
- Proposed section inventory (authoring may tune; keep compact and lookup-oriented, clean text, no narrative):
  1. **Data in** — from pandas (`NodeCollection` + `Edges`), from networkx (`graph=`, the headline idiom), dataset helpers for trying things.
  2. **Nodes** — color by data column, size, alpha (per-backend tabs).
  3. **Edges** — color by metric/tag, linewidth, alpha, z-ordering; one-line kwarg-hierarchy pointer linking `edge_kwarg_hierarchy.ipynb` (link, don't re-teach).
  4. **Axis layout** — axes order, rotation, repeat axes, angle between repeat axes, collapsing axes.
  5. **Data out** — `to_networkx()`, `to_json()`.
  6. **Figures out** — runnable export one-liners per backend (see WS-A authoring conventions), the one-line "pass `fig=` and `ax=` to embed in an existing figure" entry (real signature: `hive_plot_viz(hive_plot, fig=None, ax=None, ...)`, `viz/matplotlib.py:508-511`; no backend viz function has an `axes` parameter, and `**edge_kwargs` would silently absorb one), link to the WS-C notebook.
- Cross-links out, never duplication: gotchas page (once `edge-coverage-and-gotchas-docs` WS-B ships; if this lands first, link the relevant existing notebooks and add the gotchas link in that plan's sweep), `hive_plots_for_large_networks` for anything perf-flavored. **No perf thresholds on this page.**

### WS-A authoring conventions (api-critic planning pass; Amendment 8)

Binding on page authoring, except the final bullet (recorded discretion):

- **Non-default-setting kwarg examples use a kwarg disjoint from the default setting's kwargs** (e.g. `linestyle="--"` on `repeat_edge_kwargs`, not `alpha` when `all_edge_kwargs` already sets it). Overlap is the documented conflict-warning trigger and the runner executes under warnings-as-errors; hierarchy-conflict demonstrations stay in the linked `edge_kwarg_hierarchy.ipynb` (link, don't re-teach).
- **Two backend vocabularies never share a fence.** Example 2's plan block compresses two tab-items into one fence for review purposes only; on the page they land as separate `{tab-item}`s. One shared fence is exactly the pollution case the replay contract exists to prevent.
- **"Figures out" headlines the runnable exports per backend**: matplotlib `fig.savefig("hive_plot.svg")`, plotly `fig.write_html(...)`, bokeh `save(...)`. Plotly static export (`fig.write_image`) and bokeh `export_svgs`/`export_png` need kaleido/selenium, neither in the test env; those are the `no-run` lines. Marker granularity: a `no-run` block contains only the genuinely non-runnable line(s), with buildable code in a preceding tested block — never a tab-wide opt-out.
- **Setup dropdowns are labeled with what they define** ("Setup: build `hp`"), so a skimming copy-paster who never expanded the dropdown resolves the `NameError` at a glance.
- **Every kwarg name on the page gets a def-line check at authoring time.** The plotting surface is `**kwargs`-permissive at both levels (`update_edge_plotting_keyword_arguments(**kwargs)`, `hive_plot_viz(**edge_kwargs)`), so wrong names run green all the way to the renderer; the snippet runner structurally cannot backstop this class of error (this plan hit the shape twice at planning: the original Example 2, the `axes=` inventory entry).
- **Authoring discretion (recorded, not obligations):** (a) a one-line hover entry for the interactive backends (hover is why users pick them; plausible top-five lookup); (b) a "hive plots" scope statement in the page's one-line intro so a P2CP user knows immediately they're on the wrong page.

### WS-C notebook scope

- Gallery genre (`hiveplotlib-gallery-notebook` skill). Content: vector export (SVG/PDF via `savefig`), raster DPI handling, and the datashader coupling: changing `dpi` changes the rasterization bin count, so `pixel_spread_nodes` / `pixel_spread_edges` must move with it (source material: `examples/datashader.ipynb` cells around its dpi/pixel-spread sections). Consolidation-by-repetition is deliberate; the scattered originals stay (see Holdouts) and a later session must not "deduplicate" either side away.
- **Scaling-WS2 caveat:** do not assert datashader single-shot/streaming internals or memory behavior; `stream_chunk_threshold` is about to land on those entry points. Stick to the DPI/pixel-spread geometry story, which is orthogonal and stable.
- No subplot-embedding section (rejected; the cheat sheet carries the one-line `fig=`/`ax=` embedding entry).

## Workstreams

### Workstream A: cheat-sheet page + snippet-test harness

**Status:** not started
**Files:** `docs/source/cheat_sheet.md` (new), `docs/source/index.rst`, `docs/source/conf.py` (`sphinx_design` extension), `pyproject.toml` (`docs` extra), `tests/docs_cheat_sheet_test.py` (new), `docs/source/_llms/llms.txt` (new cheat-sheet entry), `CHANGELOG.rst`.
**Done when:**

- Api-critic planning pass on the representative snippets and opt-out convention recorded above **before** implementation dispatch; the pass must verify each representative snippet against master's actual signatures (Read the `def` line — kwarg-vocabulary checks alone missed the Example 2 misuse; Amendment 7, item 1). **Done 2026-07-06** — recorded at "API Critic's take (planning mode)", disposed at Amendment 8.
- Page exists per Design notes: section inventory covered, backend tabs `:sync:`ed, setup dropdowns collapsed and labeled with what they define, every snippet copy-paste runnable per the tab-aware contract (main-stream snippets: main-stream content above; tab snippets: main-stream content above plus that tab alone), no perf-threshold content, cross-links per Design notes, and the WS-A authoring conventions (Amendment 8) applied — including the def-line check on every kwarg name on the page.
- Snippet harness per Design notes: extractor handles nested MyST fences and tracks `{tab-set}`/`{tab-item}` boundaries, enforces reasoned `no-run` markers, forces Agg, runs the main stream in order with shared state and each tab-item against a fresh replay of its main-stream prefix; carries all five backend markers; `make test` green (100% coverage, warnings-as-errors).
- Sequencing within the workstream: land the harness plus two or three seed sections first and prove the extractor-vs-sphinx-design-fence interplay before filling the full inventory.
- All snippets written against master's API surface (post-!33, `dff7ead`): `graph=` is the documented headline, no converter-first idioms where it applies, and **no narwhals/polars/Dask/cuDF input-boundary content** (unmerged MR !36; Amendment 5 scope guard). Plan-body `file:line` citations re-verified against the execution worktree 2026-07-06 and corrected in place (Amendment 7); the re-verify obligation is **plan-wide** — every `file:line` cite in this plan (Feasibility audit and Design notes included), re-verified again at authoring time (mental-model rule: check the current constructor, not the remembered one).
- `make docs` builds with no new warnings; the new section renders between gallery and autodoc; `make format`, `make ty` clean.
- `docs/source/_llms/llms.txt` gains a cheat-sheet entry pinned high near the conceptual entry points (docs-engineer-owned; a task-oriented, CI-tested lookup page clears the consequential threshold), absolute `https://hiveplotlib.readthedocs.io/stable/...` URL per convention (Amendment 7, item 3).
- CHANGELOG entry (new user-visible docs page + new testing convention).
- Api-critic post-implementation review of the full page recorded above.

### Workstream B: README leading text

**Status:** not started
**Files:** `README.md`, `docs/source/README.md` (mirror, kept in sync). (The `edge-coverage-and-gotchas-docs.md` supersession note was written by the orchestrator at Amendment 7 — plan bodies are orchestrator surface, not a code-workstream file.)
**Done when:**

- A few sentences of "when and why hive plots" framing for arriving skeptics sit near the top of the README: the reproducibility/interpretability argument vs. force-directed hairballs, pointing at the Introduction to Hive Plots notebook. Feeder material per Prior ADRs; this is the same statement-of-need argument the JOSS paper will make — note any reusable phrasing in the Implementation log for the JOSS plan to pick up.
- Explicitly NO pointer to the gotchas page.
- No narwhals/polars/Dask/cuDF implications in the leading text (Amendment 5 scope guard; that input surface is unmerged MR !36).
- Both README copies identical in the changed region; no collision with the JOSS plan's Workstream A edits (WS-B lands first per the 2026-06-10 sequencing decision; see Plan amendments, Amendment 3).
- Prose passes the writing-voice rules (no em-dashes, no AI filler, length discipline; a few sentences, not a manifesto).
- No CHANGELOG entry (README prose, not released library behavior).

### Workstream C: saving-for-publication gallery notebook

**Status:** not started
**Files:** `examples/saving_plots_for_publication.ipynb` (new), `docs/source/notebooks/index.rst` (or gallery index per genre placement at authoring time), `docs/source/conf.py` (`nbsphinx_thumbnails` entry), thumbnail asset under `docs/source/_static/`, `docs/source/_llms/llms.txt` (conditional `## Optional` entry; see done-when), `CHANGELOG.rst`.
**Done when:**

- Notebook exists per Design notes "WS-C notebook scope": vector export, DPI handling, datashader DPI/pixel-spread coupling consolidated; gallery genre per the `hiveplotlib-gallery-notebook` skill; no subplot-embedding section; no datashader-internals claims WS2 of the scaling plan would invalidate.
- Datashader figures follow the viz-quality-bar datashader specifics (accept `cmap` defaults, pin rasterization params where cross-figure comparison is implied).
- Uses datasets already established in the corpus; no new datasets.
- Registered in the docs index and thumbnails; `make test-nb` passes for the new notebook; `make docs` builds with no new warnings.
- Prose passes the writing-voice rules.
- llms.txt judgment recorded (pre-decided at Amendment 7, item 3 — not a plan-end qa surprise): notebook-author applies the standing consequential-threshold call; expected outcome is a feature-demo entry under `## Optional`, a recorded no-with-reason is acceptable; either way the call lands in the Implementation log.
- CHANGELOG entry (new user-visible notebook).
- Editorial-critic notebook review and viz-critic figure pass recorded.

**Notebook-coherence audit:** net-new notebook; gallery genre; documents `HivePlot` viz/export mechanics across matplotlib and datashader backends; datasets all established in the corpus. Deliberate content overlap with `examples/datashader.ipynb`'s DPI sections is a recorded decision (consolidation is the point), so editorial-critic duplication findings against that overlap are pre-answered; anything else routes normally.

## Feasibility audit

- No net-new library entry points and no new attribute reads of user data; the feasibility surface is the docs/test conventions. Every snippet idiom traces to a verified entry point on master (re-verified 2026-07-06 against the execution worktree; Amendment 7): `HivePlot(graph=...)` keyword-only (`hiveplot.py:2157`), `to_networkx` (`hiveplot.py:1899`), `to_json` (`hiveplot.py:1829`), `update_edge_plotting_keyword_arguments` (`hiveplot.py:4309`, bare-`**kwargs` surface), `hive_plot_viz` per backend (`viz/matplotlib.py:508` and siblings), datasets helpers (`datasets/toy_hive_plots.py:360,637`), datashader `dpi`/`pixel_spread_*` (`examples/datashader.ipynb`, `viz/datashader.py`).
- Harness feasibility risks identified and pinned in Design notes: MyST nested-fence parsing inside sphinx-design directives (the one real extractor hazard), headless matplotlib, backend markers. The seed-sections-first sequencing in WS-A's done-when de-risks the fence interplay before content build-out.
- `sphinx-design` is docs-extra-only; it never enters the import path of `src/` or the snippet test, so no optional-import guard work.

## Sequencing

- **Branch gate — lifted 2026-07-06:** branch `46-more-streamlined-networkx-usage-and-support` merged to master as MR !33 (merge commit `fe53b00`). Execution mechanics (Amendment 4): GitLab issue #56; fresh worktree at `~/repos/hiveplotlib-56-docs-cheat-sheet` on branch `56-docs-cheat-sheet-and-readme` off origin/master (`dff7ead`); draft MR to master with "Closes #56" after first push. Cheat-sheet snippets use the merged `graph=` surface. (The gotchas plan shared this gate.)
- **Within the plan:** WS-A, WS-B, WS-C are independent and can dispatch in any order after the gate; WS-A's internal harness-first sequencing is the only intra-workstream ordering.
- **Across plans:** WS-B and the JOSS plan's Workstream A touch the same two README files; ordering resolved 2026-06-10 — **WS-B lands first**, the JOSS workstream builds on its leading text (Plan amendments, Amendment 3; mirrored note in `joss-submission.md`). The cheat sheet's gotchas cross-link depends on `edge-coverage-and-gotchas-docs` WS-B; whichever lands second adds the link.

## Plan amendments

### Amendment 1 (2026-06-10) — In-scope tweak: naming decision closed

Gary confirmed open decision 1 on the recommended option: toctree caption **"Quick Reference"**, page title **"Cheat Sheet"** (file `docs/source/cheat_sheet.md`). The Naming audit and Design notes stand as written; no plan-body changes needed.

### Amendment 2 (2026-06-10) — In-scope tweak: WS-C filename closed

Gary confirmed open decision 2 on the recommended option: `examples/saving_plots_for_publication.ipynb`. WS-C's Files list stands as written.

### Amendment 3 (2026-06-10) — In-scope tweak: README cross-plan sequencing closed

Gary set the ordering for open decision 3: **WS-B lands before `joss-submission.md` Workstream A.** Rationale: this plan gates only on the branch-46 merge, while the JOSS plan gates on the v0.28 release and the Datasaurus figure, so WS-B will be ready well before; the JOSS README workstream then builds on (and may reuse as statement-of-need prose) the leading text rather than the reverse. Recorded in both plans: Sequencing and WS-B's done-when updated here; a one-line sequencing note added to `joss-submission.md` Workstream A pointing back at this decision.

### Amendment 4 (2026-07-06) — In-scope tweak: branch-46 gate lifted; execution mechanics fixed

Branch `46-more-streamlined-networkx-usage-and-support` merged to master as MR !33 (merge commit `fe53b00`); the Sequencing branch-gate line is updated to lifted. Execution mechanics per Gary's 2026-07-06 directive: GitLab issue #56; fresh worktree at `~/repos/hiveplotlib-56-docs-cheat-sheet` on branch `56-docs-cheat-sheet-and-readme` off origin/master (`dff7ead`); draft MR to master with "Closes #56" after first push. Verified at amendment time: the worktree exists on that branch at `dff7ead`, and `fe53b00` is the !33 merge. Done-whens touched: WS-A's snippet-surface bullet now records the master line-number drift of the plan's pre-merge citations and the re-verify obligation. Dispatch note: WS-A ships its "Figures out" section without the WS-C notebook link if WS-C hasn't landed; whichever lands second adds the link (the pattern this plan already uses for the gotchas-page cross-link).

### Amendment 5 (2026-07-06) — Deferred follow-up, carrying a binding scope guard: master-only API surface

Binding on all three deliverables: every cheat-sheet snippet and all README text runs against master's API surface only; no narwhals/polars/Dask/cuDF input-boundary content anywhere. That surface arrives with unmerged MR !36 (branch 53); documenting it now would pin claims an in-flight branch owns — the same reasoning as WS-C's existing scaling-WS2 caveat, which stands. Deferred follow-up: post-0.29, once MR !36 ships, a cheat-sheet "Data in" entry for the narwhals-supported dataframe inputs (and any README implication) becomes a candidate follow-up; this plan deliberately does not reach ahead. Done-whens touched: WS-A's snippet-surface bullet and a new WS-B bullet carry the guard; WS-C is covered by its existing caveat plus this plan-wide guard.

### Amendment 6 (2026-07-06) — In-scope tweak: execution-mode record (grill skipped, auto-dispatch authorized)

Full record in `## Alignment (grill)` (execution-time record, 2026-07-06). Summary: post-plan grill knowingly skipped per Gary's directive; auto-dispatch authorized by the same directive (the recorded utterance is the opt-in per harness convention), explicitly overriding the harness default that offers auto-dispatch only on plans with a grilled `## Failure modes` rubric; halt on any `must-fix`, any `STATUS: BLOCKED`, or any workstream-set change (routes through orchestrator `amend-plan` first); one commit per workstream after the full gate battery (`make format`, `make test`, `make ty`, `make docs`, `make linkcheck` for new pages, plus any touched notebook run), committed by the dispatching session under explicit maintainer authorization — harness agents still never commit. Next, post-amendment and pre-dispatch: the adversary's cold planning challenge and the api-critic planning pass run against this amended scope; with no grill to fight them in, adversary planning-challenge items route back to `amend-plan` for explicit disposition. Done-whens touched: none.

### Amendment 7 (2026-07-06) — In-scope tweaks: disposition of the adversary's planning challenge (grill skipped; all six items)

With the grill knowingly skipped (Amendment 6), the adversary's six planning-challenge items route here for explicit disposition; this section is what the adversary's post-impl reconcile reads. All six disposed as in-scope tweaks — no workstream-set change. Source claims re-verified at amendment time against the execution worktree (`~/repos/hiveplotlib-56-docs-cheat-sheet`, `dff7ead`).

1. **[must-fix] Example 2 signature misuse — adopted; fixed in place.** Verified: `update_edge_plotting_keyword_arguments(edge_kwarg_setting="all_edge_kwargs", reset_edge_kwarg_setting=False, rebuild_edges=False, **kwargs)` at `hiveplot.py:4309`, `|=` merge at `:4330`; the old snippet runs green while teaching a wrong idiom, exactly as challenged. Example 2 rewritten to bare kwargs (with an `edge_kwarg_setting=` comment for non-default settings). The "snippet test catches any drift by construction" claim softened everywhere it appeared (API-examples implementation note, Prior ADRs same-stats line, new Design-notes guarantee bullet): the harness catches run-failure drift, not run-green misuse. Signature-check obligation (Read the `def` line) added to WS-A's api-critic-planning-pass done-when; the pending planning pass inherits it.
2. **[must-fix] Tab execution semantics — adopted; binding tab-aware contract pinned in Design notes.** Orchestrator-owned implementation-topology call (grill stays at intent level). Contract: main-stream/tab-group partition; main stream runs in one shared namespace; each tab-item runs in a fresh namespace built by replaying its main-stream prefix; tab blocks never enter the main namespace. Replay over namespace forking is deliberate — a forked dict still shares mutable objects, so both Example 2 tabs would mutate one `hp`; replay kills both challenged failure modes (sibling-tab pollution, sibling-tab dependency) by construction. Honest guarantee recorded: copy-paste runnability per reader path, not misuse detection. `:sync:` state-chaining and nested tab-sets banned. Extractor estimate grown to ~40-80 lines (Default justifications updated; ownership justification unchanged). WS-A done-when bullets recast to the tab-aware phrasing. Note: item 1's Example 2 rewrite alone would not have resolved this — the pollution risk is structural.
3. **[worth-discussing] llms.txt — adopted; pre-decided.** WS-A: `docs/source/_llms/llms.txt` added to Files and done-when (docs-engineer-owned entry pinned high near the conceptual entry points; a task-oriented, CI-tested lookup page clears the consequential threshold). WS-C: conditional Files entry plus a done-when bullet — notebook-author applies the standing threshold judgment, expected outcome an `## Optional` feature-demo entry, no-with-reason acceptable, recorded in the Implementation log either way.
4. **[worth-discussing] WS-C taste churn under auto-dispatch — disposed: churn accepted, directive unaltered.** The standing agile-cadence preference expects review churn on doc notebooks; follow-up commits after WS-C's per-workstream commit are that churn, not a defect of the mode. No pre-commit pause exemption for WS-C. Recorded as a bullet in the execution-time record so the consequence is named, as the adversary asked.
5. **[low-confidence] Citation drift — adopted; made plan-wide.** Verified: `example_multi_tag_hive_plot` drifted to `toy_hive_plots.py:637` (plan said `:607`); other spot-checked cites current (`toy_hive_plots.py:360`, `viz/matplotlib.py:439`, `viz/bokeh.py:545`, `viz/matplotlib.py:508`). All plan-body cites corrected in place to master values (API-examples intro, implementation note, Feasibility audit); WS-A's re-verify done-when bullet now states the obligation plan-wide (every `file:line` cite, all sections) and repeats it at authoring time.
6. **[low-confidence] WS-B editing another plan's body — adopted.** Plan bodies are orchestrator surface under the write choreography. The one-line supersession note was written into `wiki/wiki/plans/edge-coverage-and-gotchas-docs.md` under its Out-of-scope README bullet **at this amendment** (orchestrator write, main-checkout wiki — matching what Amendment 3 did for `joss-submission.md`); the file dropped from WS-B's Files (the two README copies remain); "Patterns this replaces" obligation (b) marked done; the Prior-ADRs flag line updated to match.

Done-whens touched: WS-A (api-critic signature check; tab-aware page + harness bullets; plan-wide cite re-verify; new llms.txt bullet), WS-C (new llms.txt bullet). WS-B: Files only. Dispatch recommendation unchanged: api-critic planning pass (now carrying the signature-check obligation) → WS-A.

### Amendment 8 (2026-07-06) — In-scope tweaks: disposition of the api-critic planning pass (all seven items; WS-A unblocked)

The api-critic planning pass (recorded at `### API Critic's take (planning mode)`) verified Examples 1-3 as amended plus the per-tab-replay contract, and returned one must-fix, three worth-discussing, and three low-confidence items. All disposed here as in-scope tweaks — no workstream-set change. The must-fix claim was re-verified at amendment time against the execution worktree (`~/repos/hiveplotlib-56-docs-cheat-sheet`): `hive_plot_viz(hive_plot, fig: Optional[plt.Figure] = None, ax: Optional[plt.Axes] = None, ...)` at `viz/matplotlib.py:508-511`, trailing `**edge_kwargs` at `:525`.

1. **[must-fix] `axes=` is not a real parameter — adopted; fixed in place, plan-wide sweep done.** No viz function in any backend has an `axes` parameter; the matplotlib embedding idiom is `fig=`/`ax=`, and `**edge_kwargs` would silently absorb `axes=` and forward it to `LineCollection` — the same run-green shape Amendment 7 item 1 purged from Example 2. Fixed: Design notes inventory item 6 (now cites the real signature), the WS-C-scope bullet ("`fig=`/`ax=` embedding entry"), and the Alignment settled-bullet (corrected in place with the 2026-06-10 wording noted as historical, not silently rewritten). Remaining `axes=` mentions in the plan sit only in the api-critic's own section, where they correctly describe the bug.
2. **[worth-discussing x3, plus the two recorded taste calls] Authoring guidance — adopted; folded into a new Design notes subsection, "WS-A authoring conventions".** Placement call: Design notes over the WS-A block, because the WS-A page done-when already binds "per Design notes" and these are page-shape rules, not sequencing; the done-when now names the subsection explicitly. Contents: disjoint-kwarg rule for non-default-setting examples (Example 2's commented line updated in place, `alpha=0.8` → `linestyle="--"`); two backend vocabularies never share a fence (Example 2's one-fence compression is plan-review-only); "Figures out" headlines the runnable exports with `no-run` at line granularity (never tab-wide); labeled setup dropdowns; the per-kwarg def-line check (the recurring `**kwargs`-permissive pattern — also a WS-A done-when clause now); the hover-entry and "hive plots"-intro taste calls recorded as authoring discretion, not obligations.
3. **[low-confidence] `no-run` marker render-verification — adopted into the harness contract.** The Design-notes opt-out bullet and the Default-justifications bullet now carry the seed-pass verification: if `<!-- no-run: ... -->` renders as literal text under this repo's myst-parser config, the marker becomes MyST's native `% no-run: <reason>` comment; exactly one shape ships, and the extractor supports only the shape that lands.
4. **[low-confidence] Suspected inverted conflict-warning gate at `hiveplot.py:4247-4248` — out of this plan's scope; routed to the maintainer as a separate follow-up task outside this plan.** No plan obligation; the disjoint-kwarg rule above keeps the page independent of whether the warning fires today.

Done-whens touched: WS-A only (planning-pass gate marked done; page bullet extended with labeled dropdowns, the authoring conventions, and the per-kwarg def-line check). WS-B, WS-C: untouched. Pre-dispatch gates now both cleared (Amendment 7: adversary; here: api-critic): **WS-A is go for dispatch.**

### Amendment 9 (2026-07-06) — In-scope tweaks: disposition of the adversary's WS-A post-impl pass (two must-fixes adopted)

The adversary's post-impl pass on the shipped WS-A page (recorded at `### Adversary post-impl`) held every planning-challenge disposition and found two must-fixes the challenge never named, plus two low-confidence items. All disposed here as **In-scope tweaks to WS-A** — no workstream-set change; WS-B/WS-C untouched; both must-fixes are downstream-inert WS-A-local page edits. Both cited blocks confirmed at amendment time against the execution worktree (`~/repos/hiveplotlib-56-docs-cheat-sheet/docs/source/cheat_sheet.md`).

1. **[must-fix] "Data out" is not copy-paste runnable on its own — adopted; fix + continue (Gary, 2026-07-06).** The block at `cheat_sheet.md:176-179` (`hp.to_networkx()` / `hp.to_json()`) has no import, no local `hp`, and no labeled setup dropdown; a reader who grabs just this section hits `NameError: name 'hp' is not defined`. The harness cannot catch it — `test_cheat_sheet_page_runs` runs all main-stream blocks in one shared namespace, so `hp` is always live from Axis layout by the time Data out executes. This is a defect against WS-A's existing done-when ("every snippet copy-paste runnable per the tab-aware contract"), not a new done-when. Fix: add a labeled `:::{dropdown} Setup: build hp` (or an inline `hp = example_hive_plot()`) matching the sibling sections (Nodes, Edges, Figures out already carry one). Gap named for the record: Data out shipped without the setup dropdown its siblings have.
2. **[must-fix] The `fig=`/`ax=` embed block has the same missing-`hp` defect — adopted; fix + continue.** The bare main-stream block at `cheat_sheet.md:238-245` calls `hive_plot_viz(hp, fig=fig, ax=axes[0])` but never defines `hp`; visually detached from the "Figures out" tab-set's setup dropdown above it, so a reader who grabs just this block hits the same `NameError`. Same harness blind spot (shared main-stream namespace). Fix: same as item 1 (labeled setup dropdown or inline `hp = example_hive_plot()`). Note: this is distinct from Amendment 8 item 1's `axes=` fix, which is already correctly shipped (the block uses the real `fig=`/`ax=` surface); the defect here is the undefined `hp`, not the parameter names.
3. **[low-confidence] Collapsed setup dropdowns as a softer skimmer trap — no change.** At `cheat_sheet.md:75-83, 111-119, 187-195` a reader who copies a tab without expanding its labeled "Setup: build `hp`" dropdown hits the same `NameError`. This is the already-accepted done-when contract ("setup dropdowns collapsed and labeled with what they define" — WS-A done-when; the labeled-dropdown convention in WS-A authoring conventions, Amendment 8). No change; disposition on record.
4. **[low-confidence] llms.txt "CI-tested" wording — authoring discretion, not required.** The `docs/source/_llms/llms.txt:47` "backed by a CI-tested example" reads slightly stronger than the harness guarantees (it proves runnability, not API correctness). A mild nit; "CI-tested" is defensible (the snippet runs in CI). The docs-engineer *may* soften to "CI-run" at its discretion during the item-1/2 fix; not an obligation.

Done-whens touched: none — both must-fixes are defects against WS-A's existing "every snippet copy-paste runnable" done-when, now made whole, not a spec change. Halt-policy refinement (see the execution-time record addendum): these are clear defects against the plan's own done-when, so they auto-resolve inline under the refined policy rather than halting; this dispatch is a docs-engineer fix.

### Amendment 10 (2026-07-06) — Disposition of the WS-C post-impl findings (editorial clean; viz + adversary items routed, no plan change)

WS-C's post-impl critics reported (blocks recorded above: editorial-critic and viz-critic under `## Notebook review`; the adversary's WS-C block under `## Adversary review`). Dispositions; no workstream-set change, no done-when touched.

1. **Editorial-critic — clean, no action.** Whole-artifact structure/scope/genre/coherence pass returned zero must-fix or worth-discussing; the deliberate datashader.ipynb overlap was confirmed as genuine consolidation, no deduplication finding.
2. **Viz-critic — `Status: propose`, both items low-confidence — surfaced to the maintainer, no plan change, no auto-edit.** Core asks verified (the DPI/pixel-spread ballooning→rebalancing effect is genuinely shown; the pinned comparison is not confounded; accessibility holds). Per the run's halt policy, a low-confidence finding surfaces to the maintainer for a prose call and does **not** trigger a plan change or an auto-edit. Both recorded as surfaced-to-maintainer: (a) the DPI trilogy's mismatched display sizes (optional one-line aside in cell 10; do not normalize figsize); (b) the cell-16 `dpi`-in-`shared_kwargs` teaching-accuracy wrinkle (on the pre-supplied fig/ax path `fig.dpi` drives bin count, not the `dpi` kwarg; they agree here so the figure is correct). The maintainer drives prose edits on his own notebooks.
3. **Adversary WS-C post-impl — one item batches to qa, the rest closed/judgment-call.**
   - Finding 1 (could not visually confirm ballooning/rebalancing, low-confidence): **closed** by viz-critic's same post-impl pass (the effect is measured and genuinely shown).
   - Finding 2 (deep anchor `datashader.ipynb#Balancing-Pixel-Spread-and-DPI` may not resolve in built HTML; `worth-discussing`): downstream-inert — WS-C is the last workstream, so under auto-dispatch this batches to the WS-C qa gate for built-HTML verification rather than routing immediately. Recorded as **batched-to-qa**; the WS-C qa gate must additionally verify the deep anchor resolves in built HTML (nbsphinx `###`-heading slugs are a known fragility and a broken intra-doc anchor is not a `make docs` warning).
   - Finding 3 (the pinned side-by-side compares two near-identical seeded networks, so the "colors mean the same thing" payoff is shown mechanically rather than as a striking difference; low-confidence): correctly pinned (not the confounded-comparison shape), so this is a pedagogy-vs-punch maintainer judgment call. No change; on record.

Done-whens touched: none.

## Holdouts

- `examples/datashader.ipynb` (DPI and pixel-spread sections) and the other datashader notebooks' scattered DPI mentions: kept as-is. WS-C consolidates by repetition, deliberately; do not strip the originals or "deduplicate" the new notebook against them.
- `README.md` "More on Hive Plots" section: kept; the WS-B leading text is additive framing at the top, not a rewrite of the existing pointers.

## Implementation log

- 2026-07-06 — WS-A phase 1 (seed): docs-engineer added `sphinx-design` to the `docs` extra (`pyproject.toml`) and `sphinx_design` to `extensions` plus `myst_enable_extensions = ["colon_fence"]` (`docs/source/conf.py`); added the "Quick Reference" toctree between the gallery and autodoc captions (`docs/source/index.rst`, holding `cheat_sheet`); authored the seed page `docs/source/cheat_sheet.md` with intro (hive-plot-scoped, P2CP redirect) and three sections — Data in (`HivePlot(graph=...)` headline), Edges (a `:sync:`ed matplotlib/bokeh `{tab-set}`, two separate tab-items, bare-kwargs form, fed by a labeled collapsed "Setup: build `hp`" dropdown), Data out (`to_networkx()` / `to_json()`). Every kwarg def-line-checked in the worktree source. **`no-run` marker outcome: the HTML-comment shape `<!-- no-run: <reason> -->` renders invisibly** (emitted as a true HTML comment, not escaped literal text); the primary shape holds, no fallback to `% no-run:` needed — the extractor should support the HTML-comment shape. Sphinx-design nesting required colon fences (`:::`) for directive wrappers over backtick escalation: backtick tab-items under a backtick tab-set tripped a `[design.tab]` "parent should be a tab-set" warning, which colon fences (via the newly enabled `colon_fence` extension) resolved. `make docs` builds clean (zero warnings); tabs, `:sync:` wiring, and dropdown all render; Quick Reference sits between gallery and autodoc. Not in phase 1: full inventory, llms.txt entry, CHANGELOG entry, WS-C cross-link (all phase 3 / later).
- 2026-07-06 — WS-A phase 2 (snippet-test harness): test-engineer added `tests/docs_cheat_sheet_test.py` (one new file; ~370 lines incl. the hand-rolled extractor). Extractor is a line scanner over `docs/source/cheat_sheet.md`: it matches backtick-python fences by their captured run length (`` `{3,} `` + `python`, closing on the same length) and tracks a stack of open colon-fence directives (`:::{name}` opens; a bare colon run closes the innermost directive of that run length), so each Python fence is tagged with whether a `{tab-item}` encloses it. Main stream = every runnable block not inside a `{tab-item}` (page-level blocks AND `{dropdown}` setup blocks); it execs in one shared namespace in page order. Each `{tab-item}` execs in a fresh namespace built by replaying only the main-stream blocks above its tab-set plus that tab's own blocks — sibling tabs never share state. On the seed page the extractor finds 6 fences: 3 runnable main-stream (the `graph=` build, the dropdown `example_hive_plot()` setup, `to_networkx`/`to_json`), 1 `no-run`-excluded (`plt.show()`), and 2 tab-items (matplotlib `linewidth`, bokeh `line_width`), each replayed with the imports+dropdown prefix only. `no-run`: HTML-comment shape confirmed; both reasonless forms (`<!-- no-run -->` with no colon and `<!-- no-run: -->` with empty reason) set `bare_no_run_seen`, which the runner asserts False. Headless via `mpl.use("Agg")`, `plt.close("all")` in a `finally`. **5 tests, 1 runner carrying all five backend markers** (`networkx`/`bokeh`/`datashader`/`holoviews`/`plotly`), 4 pure-extractor tests (partition; sibling-tab isolation — asserts the bokeh `line_width` block is absent from the matplotlib tab's replay and from the main stream; reasonless-`no-run` rejection; nested-tab-set hard-error). Falsifiability spot-checked out-of-band: a deliberately broken tab block raises through the runner, so the tab replay is not a silent no-op. Harness proves per-reader-path copy-paste runnability only; it does not catch run-green kwarg misuse (`**kwargs`-permissive surface) — stated honestly in the module docstring, no process refs. Gates: `make test` green (1364 passed, 100% total coverage under `-n 7`, warnings-as-errors); `make format`, `make ty` clean; LF clean, no CRLF warning. Extractor code lives in the test file, outside `--cov=src/hiveplotlib`, so its correctness is locked by the 4 behavior-assertion tests, not coverage.
- 2026-07-06 — WS-A Amendment 9 must-fix resolution: docs-engineer made the two `hp`-undefined blocks copy-paste independent on `docs/source/cheat_sheet.md`. Added a labeled collapsed `:::{dropdown} Setup: build hp` (`:icon: code`, `hp = example_hive_plot()`) matching the Nodes/Edges/Figures-out sibling shape to (1) the "Data out" section above `to_networkx()`/`to_json()` and (2) the `fig=`/`ax=` embed block below the "Figures out" tab-set; every hp-using section now carries its own setup, so a reader expanding the dropdown and copying just that section gets a defined `hp`. Also softened the llms.txt cheat-sheet entry from "CI-tested example" to "CI-run example" (Amendment 9 item 4 discretionary; one-word accuracy fix — the harness proves runnability, not API correctness). Gates: `make test` green (1364 passed, 100% coverage, warnings-as-errors, `-n 7`; the shared-namespace runner replays both new setup blocks as main-stream), `make docs` clean (zero warnings; five setup dropdowns render, up from three), `make format` / `make ty` clean; LF clean, no CRLF, no leaked export files. Files touched: `docs/source/cheat_sheet.md`, `docs/source/_llms/llms.txt` (only).
- 2026-07-06 — WS-A phase 3 (full content pass): docs-engineer filled the remaining section inventory on `docs/source/cheat_sheet.md`. Added three sections and extended two: **Data in** now also covers the pandas path (`NodeCollection` + `Edges`) and the `example_hive_plot()` helper; **Nodes** is a new `:sync:`ed matplotlib/bokeh tab pair styling via `update_node_viz_kwargs` (matplotlib `c`/`s`/`alpha`, bokeh `color`/`size`/`alpha`, color-by-column by passing a column name); **Edges** gained a non-default-setting example (`edge_kwarg_setting="repeat_edge_kwargs"`, disjoint `linestyle="--"`) plus matplotlib `zorder`; **Axis layout** is a new section using `HivePlot` constructor knobs forwarded through the dataset helper (`axes_order`, `rotation`, `repeat_axes`, `angle_between_repeat_axes`, collapse via `axes_order=[..., None]` + `collapsed_group_axis_name`); **Figures out** is a new `:sync:`ed matplotlib/plotly/bokeh tab set of runnable exports (`fig.savefig(".svg")`, `fig.write_html`, bokeh `output_file(...)` + `save(fig)`) plus the `fig=`/`ax=` embedding entry. Every kwarg def-line-checked in worktree source: node kwargs at `node.py:279` / `matplotlib.py:243` / `bokeh.py:323`; edge `zorder` / `linestyle` forwarded through `hiveplot.py:4309` to `matplotlib.py:525`; axis knobs at `hiveplot.py:2161-2173`; `fig`/`ax` at `matplotlib.py:508-511`. Two warnings-as-errors traps caught and fixed: bokeh bare `save(fig, path)` warns (no resources / no title) → switched to `output_file(...)` + `save(fig)`; the "Figures out" writers drop files in cwd → the runner test now `monkeypatch.chdir(tmp_path)` so exports land in pytest's tmp dir, not the repo tree. Two seed-page harness tests were retargeted for the larger page (the sibling-isolation test now keys the matplotlib/bokeh pair by their distinguishing width keyword instead of first-label, since several backend tab pairs now exist; the partition test's docstring de-hardcoded from "the two tab-items"). Added the llms.txt cheat-sheet entry under a new `## Quick reference` section (high, right after Getting started; absolute stable URL) and the CHANGELOG `Added` → Documentation bullet. WS-C cross-link deliberately omitted (WS-C not landed; whichever lands second adds it); gotchas page not hard-linked; perf-flavored content links `edge_kwarg_hierarchy.ipynb` only, no thresholds. Gates: `make test` green (1364 passed, 100% coverage, warnings-as-errors), `make docs` clean (zero warnings; all six sections + all tabs/dropdowns render, Quick Reference between gallery and autodoc), `make format` / `make ty` clean; LF clean, no CRLF, no export files leaked to the tree.
- 2026-07-06 — WS-B (README leading text): docs-engineer added a "when and why hive plots" paragraph (7 lines) directly under the one-line description, above the badges, in both `README.md` and `docs/source/README.md`. The insertion sits in the top region that is byte-identical across both copies (verified: `diff` of lines 1-13 reports identical, so no divergent inline link was introduced — the paragraph names the Introduction to Hive Plots notebook and defers to the existing links below rather than adding a copy-specific one). Argument as shipped: reach for a hive plot when you want a layout you can trust and reproduce; each node's position is set by its own properties, so the same data always draws the same picture and two plots compare side by side; the force-directed "hairball" is the opposite, with arbitrary positions that shift run to run and mean nothing; a pattern in a hive plot comes from the data, not the layout algorithm. Holdout respected: the existing "More on Hive Plots" section is untouched; this is additive framing, not a rewrite. Scope-clean: no narwhals/polars/Dask/cuDF, no gotchas-page pointer, no process references, no CHANGELOG entry (README prose). Gates: `make docs` clean (zero Sphinx warnings; the docs build includes `docs/source/README.md`); no em-dashes in the added prose; LF clean, no CRLF warning. **JOSS statement-of-need reuse:** the whole paragraph is drafted as the same statement-of-need argument the JOSS paper makes and is reusable near-verbatim. Two phrasings worth lifting: "the same data always draws the same picture and two plots can be compared side by side" (reproducibility + comparability in one clause) and "If a hive plot shows a pattern, it comes from the data, not from the layout algorithm" (a plain-English compression of Krzywinski 2012's key quote, safe to reuse as the interpretability claim).
- 2026-07-06 — WS-C (saving-for-publication gallery notebook): notebook-author added `examples/saving_plots_for_publication.ipynb`, a gallery-genre notebook covering vector export (SVG/PDF via `Figure.savefig`), raster DPI, and the datashader coupling where figure `dpi` sets the rasterization bin count so `pixel_spread_nodes`/`pixel_spread_edges` must move with it. Sections: Vector Formats (SVG and PDF), Raster Resolution (DPI), Datashader: DPI Drives the Rasterization (default dpi=150 → dpi=50 with stale spreads ballooning nodes/edges → dpi=50 with lowered spreads back in balance), and a "Pin the Parameters When Comparing Figures" subsection (two seeded networks side by side with `dpi`/`pixel_spread_nodes`/`pixel_spread_edges`/`vmax_nodes`/`vmax_edges` all pinned). Datasets are established corpus helpers only: `example_hive_plot()` for the matplotlib vector/DPI sections and `example_hive_plot(num_nodes=1000, num_edges=5000, backend="datashader")` for the datashader section (the same dataset `datashader.ipynb` uses). Scope-clean per the binding caveats: no datashader single-shot/streaming/memory/`stream_chunk_threshold` claims (only the stable DPI/pixel-spread geometry story); no subplot-embedding section; no new datasets; no narwhals/polars/Dask/cuDF content; no process references. Exports write into a `tempfile.TemporaryDirectory` so nothing lands in the tree (verified clean `git status`). Consolidation-by-repetition against `examples/datashader.ipynb` is deliberate; that Holdout was left untouched. **Registration:** added to `docs/source/gallery_examples/index.rst` under the Visualization section (both `nblinkgallery` and hidden `toctree`, same order — "exporting" is taken by the data-out notebooks per the Naming audit, so figure-saving shelves under Visualization); `nbsphinx_thumbnails` entry in `docs/source/conf.py` pointing at a new `docs/source/_static/saving_plots_for_publication.jpg` (a clean default-dpi datashaded three-axis hive plot, text stripped, distinct from the `datashader` logo thumbnail). **Cross-links, both directions (WS-C lands second):** (a) notebook prose links to the Cheat Sheet page (`../cheat_sheet.md#figures-out`, resolves to `../cheat_sheet.html#figures-out`); (b) added a one-line pointer in `docs/source/cheat_sheet.md`'s "Figures out" section to the notebook (`notebooks/saving_plots_for_publication.ipynb`), matching the existing `[edge kwarg hierarchy notebook](...)` cross-link style. **llms.txt decision: YES, entry added** under `## Optional` → Visualization backends (a feature-demo entry, not a conceptual entry point; it clears the consequential threshold because the datashader DPI/pixel-spread coupling is a real gotcha worth indexing for an agent). Absolute stable URL per convention. **CHANGELOG:** one `Added` → Documentation bullet (three wrapped lines, no sub-bullets, no process refs). Gates: `make test-nb` scoped to the notebook passes (`test_run_to_completion`, 10.4s); notebook re-executed end-to-end via the worktree-resolving `hiveplotlib` kernel, zero error/stderr outputs; `make docs` clean (zero warnings; notebook copied, all six figures rendered, thumbnail in gallery, all cross-links resolved both directions); `make test` green (1364 passed, 100% coverage, warnings-as-errors — the cheat_sheet.md link adds no Python fence so the snippet harness is unaffected); `make format` clean (added `strict=True` to the comparison `zip`). LF clean, no CRLF, no stray export files in the tree (only the intended thumbnail). viz-quality-bar honored: datashader `cmap` defaults accepted, all rasterization params pinned for the cross-figure comparison, the DPI-ballooning contrast verified in the rendered figures.
