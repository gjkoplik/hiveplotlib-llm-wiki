# Plan: docs cheat sheet, README leading text, saving-for-publication notebook

## Goal

Three docs deliverables from the 2026-06-10 docs-gaps discussion. (1) A compact, task-oriented **cheat-sheet docs page** (MyST markdown, new toctree section, sphinx-design tabs/dropdowns) answering "how do I do X" lookups without digging through notebooks, with every fenced snippet executed in CI by a new docs-snippet pytest harness (testability == copy-paste-ability). (2) A few sentences of **README leading text** giving arriving skeptics the "when and why hive plots" framing, pointing at the introduction notebook. (3) A new gallery notebook, **saving plots for publication**, consolidating vector export (SVG/PDF), DPI handling, and the datashader DPI/rasterization-parameter coupling currently scattered across datashader notebooks (repeating that scattered info here is the point).

## Alignment (grill)

Status: **aligned / ready-for-execution** (2026-06-10). Gary closed all three open decisions, each on the recommended option; formal grill skipped in favor of direct decision closure on top of the originating docs-gaps discussion. Implementation remains held on the branch-46 merge gate (see Sequencing).

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
- REJECTED: a subplot-embedding ("hive plot as one panel") notebook — spoon-feeding; the matplotlib backend returns fig/ax first class. At most a one-line cheat-sheet entry ("pass `axes=`").

## Prior ADRs / design docs

No `wiki/wiki/adr/` directory exists yet; no prior ADRs. Relevant plans (research-liaison pre-task pass):

- `wiki/wiki/plans/edge-coverage-and-gotchas-docs.md` — its line 11 records README changes as out-of-scope **for that plan**; this plan supersedes that status quo for the leading-text change only (flag, not contradiction — WS-B adds a one-line cross-reference there). Also: FAQ-page rejection provenance, the gotchas page WS-B cross-links rather than duplicates, and the shared branch-46 merge gate.
- `wiki/wiki/plans/scaling-large-networks.md` — WS7 (perf decision table in `hive_plots_for_large_networks.ipynb`) owns all perf guidance; WS2 will add `stream_chunk_threshold` to the datashader entry points, so WS-C must not pin datashader internals claims that plan is about to rewrite.
- `wiki/wiki/plans/joss-submission.md` — concurrent README work (Citing section, support/governance statement); records that `docs/source/README.md` mirrors the root README and must stay in sync; the leading text is the same statement-of-need argument the JOSS paper makes (prose reuse both directions).
- `wiki/wiki/plans/same-stats-different-graphs.md` line 183 — recurring stale-snippet pattern: check snippets against the *current* `HivePlot` surface (`graph=` keyword from branch 46), not the surface at drafting time. The CI snippet test makes this self-enforcing for the cheat sheet.
- `wiki/wiki/plans/graph-metrics-notebook-restructure.md` — notebook corpus inventory; datashader figure conventions (its WS-C done-when item 8) for WS-C figures.
- Feeder material for WS-B prose: `wiki/wiki/entities/hiveplotlib.md`, `wiki/wiki/concepts/force-directed-layout.md`, `wiki/wiki/sources/bostock-2012-d3-hive-plots.md`, `wiki/wiki/sources/krzywinski-2012.md`.
- Gaps: no prior wiki record of docs-snippet testing conventions, toctree-structure decisions, or sphinx-design usage; no wiki page on the datashader DPI coupling (material lives only in repo notebooks). The snippet-test harness is a net-new docs-testing convention and a likely future ADR candidate.

## Patterns this replaces

- None — all three deliverables are net-new additions. Two sweep obligations that aren't replacements: (a) any WS-B edit to `README.md` must be mirrored in `docs/source/README.md` (the JOSS plan's recorded sync constraint); (b) WS-B adds a one-line supersession note to `wiki/wiki/plans/edge-coverage-and-gotchas-docs.md` under its "Out of scope" README bullet.

## Default justifications

No new library API, parameters, or defaults. Convention-level defaults (binding; see Design notes for specs):

- **Hand-rolled snippet extractor over `mktestdocs`**: the opt-out marker, shared-state ordering, and MyST nested-fence parsing *are* the contract; owning ~40 lines of extractor beats pinning a micro-dependency whose parsing conventions would silently define a CI-critical contract. No new test dependency.
- **Matplotlib `Agg` backend forced in the snippet test**: CI is headless; snippets must not require a display.
- **All five optional-backend markers on the page-runner test** (`networkx`, `bokeh`, `datashader`, `holoviews`, `plotly`): the page's per-backend tabs touch every extra; marking the whole runner keeps backend-less environments green. Test env already installs all extras.
- **Opt-out marker is an HTML comment above the fence** (`<!-- no-run: <reason> -->`): visible in source and to the extractor, invisible in the rendered page; a required reason string keeps opt-outs rare and auditable.

## Naming audit

- Toctree caption: **"Quick Reference"** (recommended) over "User Guide". The section holds exactly one lookup page; pandas/numpy/matplotlib "User Guide" vocabulary promises narrative chapters that this repo's notebooks already own. Placed between the gallery and autodoc captions in `docs/source/index.rst`.
- Page title: **"Cheat Sheet"**, file `docs/source/cheat_sheet.md`. Direct ecosystem anchor: the matplotlib cheatsheets (the page's stated spirit reference); "quick reference" survives in the caption.
- Test file: `tests/docs_cheat_sheet_test.py`. The `*_test.py` mirror convention maps src files; this is the first docs-page test, named for its subject.
- Opt-out marker token: `no-run` (reads as the action it suppresses; `skip` collides with pytest vocabulary while meaning something different here).
- WS-C notebook: **`examples/saving_plots_for_publication.ipynb`** (recommended). "Publication-quality figures" is established matplotlib-ecosystem vocabulary; "exporting" is taken by the data-out notebooks (`exporting_hive_plots_to_networkx.ipynb`, `exporting_hive_plots_to_json.ipynb`) and would mis-shelve a figure-saving page.
- Prose-only terms: "cheat sheet", "snippet", "publication-quality". No new parameters, methods, or classes.

## API usage examples

The cheat-sheet page is itself an API-usage-examples artifact; the api-critic planning pass should review the representative snippets below (shape, headline idioms, tab/dropdown conventions, opt-out marker), not an exhaustive inventory. All snippets written against the branch-46 constructor surface (`graph=` keyword-only at `src/hiveplotlib/hiveplot.py:2084`; `to_networkx` at `:1839`, `to_json` at `:1769`).

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
# Example data: hp from the page's setup dropdown:
from hiveplotlib.datasets import example_hive_plot

hp = example_hive_plot()

# Call site (matplotlib tab):
hp.update_edge_plotting_keyword_arguments(
    all_edge_kwargs={"color": "darkblue", "linewidth": 0.5, "alpha": 0.3},
)
# Call site (bokeh tab — same task, bokeh vocabulary):
hp.update_edge_plotting_keyword_arguments(
    all_edge_kwargs={"color": "darkblue", "line_width": 0.5, "alpha": 0.3},
)
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

Implementation note for WS-A: Example 2's per-backend kwarg names must be verified against each backend's `rename_edge_kwargs` wiring (`viz/matplotlib.py:439-441` uses `linewidth`/`alpha`/`color`; `viz/bokeh.py:545-547` uses `line_width`/`alpha`/`color`; plotly uses `width`), and `update_edge_plotting_keyword_arguments`'s exact signature confirmed at authoring time — the snippet test catches any drift by construction.

### API Critic's take (planning mode)

```
Pending — invoke api-critic in planning mode on the representative snippets and the
opt-out marker convention before WS-A dispatch.
```

### API Critic — post-implementation review

No library API surface changes in this plan. The post-impl review applies to the cheat-sheet page as a usage-surface artifact: invoke api-critic in post-implementation mode after WS-A ships to review the full page's snippets (headline idioms, per-backend correctness, anything the planning pass's representative sample missed).

```
Pending — invoke api-critic in post-implementation mode after Workstream A ships.
```

## Notebook review

Applies to WS-C only (the cheat sheet is a docs page, not a notebook).

```
Pending — invoke editorial-critic in post-implementation mode after Workstream C ships.
```

## Design notes (binding)

### Snippet-test harness (WS-A)

- **Contract:** `tests/docs_cheat_sheet_test.py` reads `docs/source/cheat_sheet.md`, extracts fenced Python blocks in page order, and `exec`s them in a single shared namespace. A snippet is correct iff it runs given what is visible above it on the page. One runner test for the whole page (shared state crosses sections), carrying all five optional-backend markers.
- **Extractor (hand-rolled, ~40 lines):** a fence opens on a line matching three-or-more backticks followed by `python` and closes on a matching-length backtick run. **MyST gotcha this must handle:** code fences nested inside sphinx-design directives (`{tab-set}`, `{tab-item}`, `{dropdown}`) sit inside *longer* backtick (or colon) fence runs; the extractor matches fence-run lengths rather than assuming bare triple-backticks, and ignores non-Python fences (`bash`, `markdown`, directive bodies).
- **Opt-out:** an HTML comment `<!-- no-run: <reason> -->` on the line(s) immediately above a fence excludes that block. The extractor requires a non-empty reason; the test fails on a bare `no-run`. Used sparingly (genuinely non-runnable entries only, e.g. interactive `fig.show()`).
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
  6. **Figures out** — `savefig` one-liners, the one-line "pass `axes=` to embed in an existing figure" entry, link to the WS-C notebook.
- Cross-links out, never duplication: gotchas page (once `edge-coverage-and-gotchas-docs` WS-B ships; if this lands first, link the relevant existing notebooks and add the gotchas link in that plan's sweep), `hive_plots_for_large_networks` for anything perf-flavored. **No perf thresholds on this page.**

### WS-C notebook scope

- Gallery genre (`hiveplotlib-gallery-notebook` skill). Content: vector export (SVG/PDF via `savefig`), raster DPI handling, and the datashader coupling: changing `dpi` changes the rasterization bin count, so `pixel_spread_nodes` / `pixel_spread_edges` must move with it (source material: `examples/datashader.ipynb` cells around its dpi/pixel-spread sections). Consolidation-by-repetition is deliberate; the scattered originals stay (see Holdouts) and a later session must not "deduplicate" either side away.
- **Scaling-WS2 caveat:** do not assert datashader single-shot/streaming internals or memory behavior; `stream_chunk_threshold` is about to land on those entry points. Stick to the DPI/pixel-spread geometry story, which is orthogonal and stable.
- No subplot-embedding section (rejected; the cheat sheet carries the one-line `axes=` entry).

## Workstreams

### Workstream A: cheat-sheet page + snippet-test harness

**Status:** not started
**Files:** `docs/source/cheat_sheet.md` (new), `docs/source/index.rst`, `docs/source/conf.py` (`sphinx_design` extension), `pyproject.toml` (`docs` extra), `tests/docs_cheat_sheet_test.py` (new), `CHANGELOG.rst`.
**Done when:**

- Api-critic planning pass on the representative snippets and opt-out convention recorded above **before** implementation dispatch.
- Page exists per Design notes: section inventory covered, backend tabs `:sync:`ed, setup dropdowns collapsed, every snippet copy-paste runnable given what's visible above it, no perf-threshold content, cross-links per Design notes.
- Snippet harness per Design notes: extractor handles nested MyST fences, enforces reasoned `no-run` markers, forces Agg, runs the page's blocks in order with shared state; carries all five backend markers; `make test` green (100% coverage, warnings-as-errors).
- Sequencing within the workstream: land the harness plus two or three seed sections first and prove the extractor-vs-sphinx-design-fence interplay before filling the full inventory.
- All snippets written against the merged `graph=` surface; no converter-first idioms where `graph=` is the documented headline (mental-model rule: check the current constructor, not the remembered one).
- `make docs` builds with no new warnings; the new section renders between gallery and autodoc; `make format`, `make ty` clean.
- CHANGELOG entry (new user-visible docs page + new testing convention).
- Api-critic post-implementation review of the full page recorded above.

### Workstream B: README leading text

**Status:** not started
**Files:** `README.md`, `docs/source/README.md` (mirror, kept in sync), `wiki/wiki/plans/edge-coverage-and-gotchas-docs.md` (one-line supersession note under its Out of scope README bullet).
**Done when:**

- A few sentences of "when and why hive plots" framing for arriving skeptics sit near the top of the README: the reproducibility/interpretability argument vs. force-directed hairballs, pointing at the Introduction to Hive Plots notebook. Feeder material per Prior ADRs; this is the same statement-of-need argument the JOSS paper will make — note any reusable phrasing in the Implementation log for the JOSS plan to pick up.
- Explicitly NO pointer to the gotchas page.
- Both README copies identical in the changed region; no collision with the JOSS plan's Workstream A edits (WS-B lands first per the 2026-06-10 sequencing decision; see Plan amendments, Amendment 3).
- Prose passes the writing-voice rules (no em-dashes, no AI filler, length discipline; a few sentences, not a manifesto).
- No CHANGELOG entry (README prose, not released library behavior).

### Workstream C: saving-for-publication gallery notebook

**Status:** not started
**Files:** `examples/saving_plots_for_publication.ipynb` (new), `docs/source/notebooks/index.rst` (or gallery index per genre placement at authoring time), `docs/source/conf.py` (`nbsphinx_thumbnails` entry), thumbnail asset under `docs/source/_static/`, `CHANGELOG.rst`.
**Done when:**

- Notebook exists per Design notes "WS-C notebook scope": vector export, DPI handling, datashader DPI/pixel-spread coupling consolidated; gallery genre per the `hiveplotlib-gallery-notebook` skill; no subplot-embedding section; no datashader-internals claims WS2 of the scaling plan would invalidate.
- Datashader figures follow the viz-quality-bar datashader specifics (accept `cmap` defaults, pin rasterization params where cross-figure comparison is implied).
- Uses datasets already established in the corpus; no new datasets.
- Registered in the docs index and thumbnails; `make test-nb` passes for the new notebook; `make docs` builds with no new warnings.
- Prose passes the writing-voice rules.
- CHANGELOG entry (new user-visible notebook).
- Editorial-critic notebook review and viz-critic figure pass recorded.

**Notebook-coherence audit:** net-new notebook; gallery genre; documents `HivePlot` viz/export mechanics across matplotlib and datashader backends; datasets all established in the corpus. Deliberate content overlap with `examples/datashader.ipynb`'s DPI sections is a recorded decision (consolidation is the point), so editorial-critic duplication findings against that overlap are pre-answered; anything else routes normally.

## Feasibility audit

- No net-new library entry points and no new attribute reads of user data; the feasibility surface is the docs/test conventions. Every snippet idiom traces to a verified entry point on the working branch: `HivePlot(graph=...)` keyword-only (`hiveplot.py:2084`), `to_networkx` (`hiveplot.py:1839`), `to_json` (`hiveplot.py:1769`), `hive_plot_viz` per backend (`viz/matplotlib.py:508` and siblings), datasets helpers (`datasets/toy_hive_plots.py:360,607`), datashader `dpi`/`pixel_spread_*` (`examples/datashader.ipynb`, `viz/datashader.py`).
- Harness feasibility risks identified and pinned in Design notes: MyST nested-fence parsing inside sphinx-design directives (the one real extractor hazard), headless matplotlib, backend markers. The seed-sections-first sequencing in WS-A's done-when de-risks the fence interplay before content build-out.
- `sphinx-design` is docs-extra-only; it never enters the import path of `src/` or the snippet test, so no optional-import guard work.

## Sequencing

- **Branch gate:** implementation waits until branch `46-more-streamlined-networkx-usage-and-support` merges; this work goes on a fresh branch off master as a separate MR (cheat-sheet snippets use the `graph=` surface). Same gate as the gotchas plan.
- **Within the plan:** WS-A, WS-B, WS-C are independent and can dispatch in any order after the gate; WS-A's internal harness-first sequencing is the only intra-workstream ordering.
- **Across plans:** WS-B and the JOSS plan's Workstream A touch the same two README files; ordering resolved 2026-06-10 — **WS-B lands first**, the JOSS workstream builds on its leading text (Plan amendments, Amendment 3; mirrored note in `joss-submission.md`). The cheat sheet's gotchas cross-link depends on `edge-coverage-and-gotchas-docs` WS-B; whichever lands second adds the link.

## Plan amendments

### Amendment 1 (2026-06-10) — In-scope tweak: naming decision closed

Gary confirmed open decision 1 on the recommended option: toctree caption **"Quick Reference"**, page title **"Cheat Sheet"** (file `docs/source/cheat_sheet.md`). The Naming audit and Design notes stand as written; no plan-body changes needed.

### Amendment 2 (2026-06-10) — In-scope tweak: WS-C filename closed

Gary confirmed open decision 2 on the recommended option: `examples/saving_plots_for_publication.ipynb`. WS-C's Files list stands as written.

### Amendment 3 (2026-06-10) — In-scope tweak: README cross-plan sequencing closed

Gary set the ordering for open decision 3: **WS-B lands before `joss-submission.md` Workstream A.** Rationale: this plan gates only on the branch-46 merge, while the JOSS plan gates on the v0.28 release and the Datasaurus figure, so WS-B will be ready well before; the JOSS README workstream then builds on (and may reuse as statement-of-need prose) the leading text rather than the reverse. Recorded in both plans: Sequencing and WS-B's done-when updated here; a one-line sequencing note added to `joss-submission.md` Workstream A pointing back at this decision.

## Holdouts

- `examples/datashader.ipynb` (DPI and pixel-spread sections) and the other datashader notebooks' scattered DPI mentions: kept as-is. WS-C consolidates by repetition, deliberately; do not strip the originals or "deduplicate" the new notebook against them.
- `README.md` "More on Hive Plots" section: kept; the WS-B leading text is additive framing at the top, not a rewrite of the existing pointers.

## Implementation log

- (empty)
