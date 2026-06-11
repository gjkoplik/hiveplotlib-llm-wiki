# Plan: edge-coverage introspection + gotchas docs page

## Goal

Two deliverables. (1) A queryable `edge_coverage` summary on `HivePlot` reporting what fraction of the underlying edges the current realization actually draws, surfaced as a property (drawn/total counts plus fraction) and as a percent-only clause in `__repr__` (Amendment 1: the repr carries no n/N; the property is the precision channel). Deliberately general: no taxonomy of *why* edges are undrawn (that bookkeeping is fragile once tags, collapsed axes, and repeat axes interact; Gary explicitly chose percent + (n/N)). (2) A short "gotchas" docs page collecting hive plot pitfalls in one place (intra-axis edges, >3-axes adjacency, overplotting, parallel-edge collapse, edge-kwarg-hierarchy overrides, `graph_directed` inference — the latter three per Amendment 4) so other docs link to it instead of re-explaining locally, cross-referencing the coverage property where natural.

Explicitly NOT a warning: `wiki/wiki/plans/graph-metric-conflict-validation.md` rejected a similar warning on the principle that warnings need a real, currently-silent data-fidelity loss with no existing point-of-use signal. This repr line *becomes* that point-of-use signal.

### Out of scope (decided in discussion, recorded here)

- README changes (stays as-is with existing pointers to quick_hive_plots and karate club notebooks).
- A `quick_hive_plot()` convenience function with default partition/sorting — REJECTED: forcing the partition/sorting choice is deliberate interpretability pedagogy; `HivePlotMatrix.from_variable_sweep` is the sanctioned "try several" affordance.
- Any warning for undrawn edges (rejected per principle above).
- Changing `repeat_axes=True` default (breaking, partial fix only).
- `HivePlotMatrix` coverage aggregate — deliberate non-goal, not a deferral (Amendment 2). Edges appear in multiple cells by design, so any matrix-level number is semantically muddled, not just redundant ("it doesn't make sense at the matrix level"). Per-cell coverage is reachable via each populated cell's inherited property.
- P2CP: N/A — P2CPs draw all data; no contradicting prior record. `P2CP` composes a `BaseHivePlot` (`src/hiveplotlib/p2cp.py:54`) rather than inheriting, so placing the property on `BaseHivePlot` does not leak it onto `P2CP`'s public surface.

## Alignment (grill)

Status: aligned (2026-06-10). Planning complete; api-critic planning pass recorded below; all open decisions resolved by Gary (Amendments 1-3). Implementation dispatch **held** until branch `46-more-streamlined-networkx-usage-and-support` merges; this work then goes on a fresh branch off master as a separate MR (see Branch / MR sequencing).

Open decisions: none.

- ~~`HivePlotMatrix` propagation~~ — resolved as a deliberate non-goal, see Out of scope and Amendment 2.
- ~~Degenerate-case return when no edges exist~~ — confirmed by api-critic planning pass (`fraction=nan`, repr parenthetical omitted), pinned in Design notes.

Route any resulting plan change to amend-plan (rule 14).

## Prior ADRs / design docs

No `wiki/wiki/adr/` directory exists yet; no prior ADRs. Relevant plans/concepts (research-liaison pre-task pass):

- `wiki/wiki/plans/scaling-large-networks.md` — WS-1 (not started) changes `relevant_edges` storage from boolean masks to integer index arrays plus a Dask-aware non-materializing variant. Coverage must be representation-agnostic and becomes a **second consumer**; that plan's claim at its line ~65 that `viz/datashader.py:203-204` is "the only known read site" needs updating (it is already stale: `viz/matplotlib.py:460`, `viz/bokeh.py:580`, `viz/holoviews.py:636`, `viz/plotly.py:726`, `hiveplot.py:4555` also read it).
- `wiki/wiki/plans/graph-metric-conflict-validation.md` — warnings-rejection principle cited in Goal; warning conventions if ever needed (plain UserWarning, stacklevel=3, api-critic owns message text).
- `wiki/wiki/plans/hiveplot-unify-axes.md` — HPM mirroring precedent (same name, same shape) if propagation is chosen.
- `wiki/wiki/plans/i-want-to-plan-optimized-hoare.md` — seven-surface kwarg-replication pain; deliberate propagation decisions.
- `wiki/wiki/concepts/edge-rendering.md` — edge tags are separate DataFrames; pipeline reference for numerator/denominator semantics.
- `wiki/wiki/analyses/karate-club-walkthrough.md`, `wiki/wiki/concepts/hive-plot.md` ("Why Three Axes?") — gotchas source material.
- Wiki log 2026-05-31 entry on `relevant_edges` syncing across `Edges.copy` / `reset_edges` — correctness constraint for coverage.

## Patterns this replaces

- `HivePlot.__repr__` edges clause `f"{len(self.edges)} edges, "` at `src/hiveplotlib/hiveplot.py:2543`. Replace with `f"{coverage.total:,} edges ({pct}% drawn), "` (exact spec pinned in Design notes, "Repr display rule"). Existing repr assertions at `tests/hiveplot_test.py:5627`, `:5641`, `:5655`, `:5675-5676` must be updated to the new format.
- Everything else is net new addition.

## Default justifications

No new constructor parameters or defaults. One degenerate-case behavior to pin down (not a default per se): when `self.edges is None` or `len(self.edges) == 0`, the property returns `drawn=0, total=0, fraction=float("nan")` (a 0/0 coverage is undefined, and `0.0` would falsely read as "everything hidden"), and the repr omits the coverage parenthetical. Api-critic planning pass to confirm.

## Naming audit

- New property: **`edge_coverage`** (recommended) on `BaseHivePlot`, inherited by `HivePlot`. Alternatives considered: `coverage` (ambiguous — axis coverage? node coverage?), `drawn_edges` (sounds like it returns the edges themselves), `edge_visibility` (collides with viz-kwarg vocabulary like alpha/zorder). "Coverage" matches user vocabulary from test coverage: fraction of a whole that is exercised. Property, not method: it takes no arguments and is cheap (a dict walk plus mask ORs), matching existing zero-arg properties like `edge_kwarg_hierarchy`.
- New return type: **`EdgeCoverage`** NamedTuple with fields `drawn: int`, `total: int`, `fraction: float`. `fraction` in [0, 1] (not `percent`; matches pandas/numpy vocabulary where user-facing fractions are unit-interval floats). NamedTuple gives named access (`hp.edge_coverage.fraction`), tuple unpacking, and a readable default repr. Export location (Amendment 3, per api-critic recommendation): defined in `src/hiveplotlib/hiveplot.py` next to `BaseHivePlot`, re-exported from the `hiveplotlib` top-level `__init__` (which already re-exports from `hiveplotlib.hiveplot`, `src/hiveplotlib/__init__.py:9-13`), so `from hiveplotlib import EdgeCoverage` works for annotations and `isinstance` checks.
- Docs page title: **"Hive Plot Gotchas"** at `examples/hive_plot_gotchas.ipynb` (recommended). Direct precedent: pandas' long-standing "Gotchas" user-guide page, so the word is established ecosystem vocabulary, not slang. Alternatives: "Common Pitfalls" (fine but no ecosystem anchor), "FAQ" (Gary lukewarm; the page answers "why does my plot look wrong," not arbitrary questions).
- Prose-only terms: "drawn" / "undrawn" edges (used in repr and docs); "realization" only in wiki/plan prose, not user-facing.

## API usage examples

### Proposed (from planner / Orchestrator)

```python
# Example 1: check how many edges the current realization draws
# Example data:
from hiveplotlib.datasets import example_hive_plot

hp = example_hive_plot()  # generates edges between all axes incl. repeat; built without repeat axes

# Call site:
coverage = hp.edge_coverage
print(coverage)                 # EdgeCoverage(drawn=82, total=100, fraction=0.82)
print(coverage.fraction)        # 0.82
print(hp)                       # ... 100 edges (82% drawn), partition='partition_0', ...
```

```python
# Example 2: adding repeat axes recovers intra-axis edges; coverage reflects it
# Example data:
from hiveplotlib.datasets import example_hive_plot

hp_no_repeat = example_hive_plot()
hp_repeat = example_hive_plot(repeat_axes=True)

# Call site:
assert hp_repeat.edge_coverage.fraction > hp_no_repeat.edge_coverage.fraction
```

```python
# Example 3: coverage on a networkx-built hive plot (intra-club edges undrawn with two axes)
# Example data:
import networkx as nx

g = nx.karate_club_graph()

# Call site:
from hiveplotlib import HivePlot

hp = HivePlot(
    graph=g,
    node_graph_metrics="degree",
    partition_variable="club",
    sorting_variables="degree",
)
drawn, total, fraction = hp.edge_coverage  # tuple unpacking; fraction < 1 (intra-club edges hidden)
```

(Examples written against the current branch surface: `HivePlot(graph=...)` is the keyword-only parameter shipped in the in-flight v0.28 work at `src/hiveplotlib/hiveplot.py:2084`.)

### API Critic's take (planning mode)

Reviewed 2026-06-10 against the snippets above; verified `example_hive_plot(repeat_axes=True)` is real (flows through `**hive_plot_kwargs`, `toy_hive_plots.py:370`) and the repr clause being replaced is at `hiveplot.py:2543`. Verdicts on the flagged questions:

- **Property vs method: Agreed.** Zero arguments, cheap, read-only introspection; matches the `edge_kwarg_hierarchy` precedent (`hiveplot.py:2550`), and no-caching means there is no staleness argument for a method.
- **`EdgeCoverage(drawn, total, fraction)`: Agreed**, including field order (reads as n, N, n/N) and `fraction` over `percent` (unit-interval float matches pandas/numpy vocabulary). The NamedTuple earns its place: Example 3's tuple unpacking and Example 1's `.fraction` access are both real usage shapes a bare float or string can't serve, and the default repr is the headline print in Example 1.
- **Missing decision — `EdgeCoverage` export location.** The plan never says where the class lives or whether users can import it. A user annotating `coverage: EdgeCoverage` or catching it in `isinstance` needs an import path; define it in `hiveplot.py` next to `BaseHivePlot` and export from the `hiveplotlib` top level (the `__init__` already re-exports from `hiveplotlib.hiveplot`). Pin before WS-A dispatch.
- **Degenerate no-edges case (`fraction=nan`, repr clause omitted): Agreed.** `nan` is honest where `0.0` lies ("everything hidden") and `1.0` is vacuous; it also does the right thing in the natural user check, since `hp.edge_coverage.fraction < 1` ("are edges hidden?") is `False` on an empty plot. Two docstring requirements: (1) document the NaN explicitly and point users at `total == 0` as the emptiness check (NaN fails `==` against itself, so `fraction != fraction` traps the unwary); (2) keep `drawn=0, total=0` as stated so the counts stay arithmetic-safe.
- **Repr clause: prefer the planner's short form, but pin the discrepancy first.** The Goal sentence ("percent plus drawn/total counts, surfaced both as a property and as a clause in `__repr__`") can be read as the repr also carrying n/N (e.g. `"82% of edges drawn (82/100)"`), while the proposed clause is `f"{total:,} edges ({pct}% drawn), "`. Preferred: the proposed short form. Reason: the clause already carries N, an n/N parenthetical duplicates it ("100 edges (82/100 drawn)"), and the property is the precision channel one access away; the repr's job is the point-of-use nudge, not the full report. If Gary intended n/N literally in the repr, decide now since the repr tests bake the format. The rounding-inward boundary rule (no false 100%/0%) is exactly right either way.
- **Multi-tag counting legibility: Agreed**, because it changes nothing the user already sees: the existing repr's "N edges" is already `len(self.edges)`, i.e. tag-rows, so the coverage clause inherits established semantics rather than coining new ones. Requirement: the property docstring states it in one plain sentence ("an edge appearing under two tags counts once per tag, in both counts"), not only the design notes.

Recurring pattern to carry into WS-A: two pin-before-dispatch items (export location; repr n/N vs percent-only), both cheap now and format-breaking after the repr tests land.

### API Critic — post-implementation review

```
Pending — invoke api-critic in post-implementation mode after Workstream A ships.
```

## Notebook review

```
Pending — invoke editorial-critic in post-implementation mode after Workstream B ships.
```

## Design notes (Workstream A semantics — binding)

- **Counting semantics.** Denominator: `len(self.edges)` — total rows across all tag DataFrames. Per `wiki/wiki/concepts/edge-rendering.md`, tags are separate DataFrames; an edge appearing under two tags is two rows and counts once per tag in both numerator and denominator (tags are independent edge sets; collapsing duplicates across tags would require cross-tag row identity that doesn't exist). Document this in the property docstring. Numerator: for each tag, the number of **distinct** edge rows referenced by any `relevant_edges[a0][a1][tag]` cell, summed over tags. Distinct, not sum-of-cell-lengths: the same row can be referenced by multiple axis-pair cells (repeat-axes and bidirectional wiring), and a sum would double-count.
- **Representation-agnostic seam.** Today cells store boolean masks (`edges.py:121-124`, written at `hiveplot.py:898`); scaling WS-1 will move to integer index arrays plus a Dask-aware variant. Implement the per-tag distinct count behind one small private helper (e.g. `Edges._drawn_count(tag)` or a module-level function) that normalizes a cell's stored value: boolean masks → `np.logical_or.reduce(masks).sum()`; index arrays later → `len(reduce(np.union1d, arrays))`. The public property never touches the stored representation directly, so scaling WS-1 swaps one helper body. Update `wiki/wiki/plans/scaling-large-networks.md` WS-1's read-site claim in this workstream (one-line edit listing the helper as a consumer).
- **No caching.** Compute lazily on each property access from `relevant_edges`. This makes correctness across `Edges.copy` (deep-copies `relevant_edges`, `edges.py:256`), `reset_edges` (pops cells, `hiveplot.py:718-779`), and re-`connect_axes` automatic — no state to go stale (per the wiki log 2026-05-31 syncing entry).
- **Repr display rule (exact spec, pinned by Amendment 1 — percent-only, no n/N).** The edges clause at `hiveplot.py:2543` becomes `f"{coverage.total:,} edges ({pct}% drawn), "`, e.g. `"5,100 edges (82% drawn), "`. `pct` is an integer percent: `round(coverage.fraction * 100)`, then rounded inward at the boundaries — never display `100` unless `drawn == total` (clamp to `99`), never display `0` unless `drawn == 0` (clamp to `1`). No n/N in the repr: the clause already carries the total, and the `edge_coverage` property is the precision channel. Note `{total:,}` adds a thousands separator the current clause lacks; repr tests must expect it. Degenerate case (`total == 0`): the parenthetical is omitted, clause is `"0 edges, "`. The repr tests bake this format. `HivePlotMatrix.__repr__` (`hiveplot_matrix.py:2175`) untouched — matrix-level coverage is a deliberate non-goal (Amendment 2).

## Workstreams

### Workstream A: `edge_coverage` property and repr clause

**Status:** not started
**Files:** `src/hiveplotlib/hiveplot.py` (`EdgeCoverage` NamedTuple next to `BaseHivePlot`; property on `BaseHivePlot`; `HivePlot.__repr__` clause at :2528-2548), `src/hiveplotlib/__init__.py` (re-export `EdgeCoverage`), `src/hiveplotlib/edges.py` (private distinct-count helper, if placed there), `tests/hiveplot_test.py` (new coverage tests + repr assertion updates at :5620-5680), `tests/edges_test.py` (helper tests if helper lands on `Edges`), `CHANGELOG.rst`, `wiki/wiki/plans/scaling-large-networks.md` (read-site claim, one line).
**Done when:**

- Api-critic planning-mode pass on the API usage examples is recorded above **before** implementation dispatch.
- `edge_coverage` property exists on `BaseHivePlot` per the binding semantics above; `EdgeCoverage` defined in `hiveplot.py` and importable as `from hiveplotlib import EdgeCoverage`; `HivePlot` repr includes the percent-only coverage clause per the pinned spec; `HivePlotMatrix` and `P2CP` surfaces unchanged.
- Tests cover: multi-tag counting (e.g. `example_multi_tag_hive_plot()`), no-repeat vs repeat-axes coverage delta, distinctness (a row referenced by two cells counts once), degenerate no-edges case, coverage drop after `reset_edges`, preservation through `Edges.copy`, update after re-`connect_axes`, and the repr boundary rule (no false 100%/0%).
- `make test` green (100% coverage, warnings-as-errors), `make format`, `make ty` clean.
- CHANGELOG entry under the in-flight unreleased version (v0.28.0 if still WIP at landing; otherwise the next unreleased section) — new user-visible behavior, so the entry is required regardless of release cycle.
- Api-critic post-implementation review recorded above.

### Workstream B: "Hive Plot Gotchas" docs page

**Status:** not started
**Files:** `examples/hive_plot_gotchas.ipynb` (new), `docs/source/notebooks/index.rst` (add under "Hive Plots"), `docs/source/conf.py` (thumbnail entry, plus `_static/` image), cross-link touches in `examples/adding_repeat_axes.ipynb` and `examples/hive_plots_more_than_three_groups.ipynb` (link only; no prose rewrites).
**Done when:**

- New short tutorial-genre notebook covering at least: (1) intra-axis edges invisible without repeat axes (source: `wiki/wiki/analyses/karate-club-walkthrough.md`), (2) >3 axes means not all axis pairs adjacent, so not all edges visible at once (source: `wiki/wiki/concepts/hive-plot.md` "Why Three Axes?"; link out to `hive_plots_more_than_three_groups`), (3) overplotting — same-color edges drawn on top of each other hide density (link out to `hive_plots_for_large_networks` / datashader material), (4) parallel-edge collapse on networkx multigraph ingestion — edges merge last-write-wins and a warning fires by default (`_warn_if_parallel_edges_collapse`, `src/hiveplotlib/hiveplot.py:258`); entry states what happens, why, what the warning means, and the `graph_multigraph=True` workaround, linking to existing coverage rather than re-teaching (Amendment 4), (5) "I set an edge kwarg and it didn't take effect" — a higher-priority kwarg in the hierarchy (`all_edge_kwargs` → ... → `non_repeat_edge_kwargs`) overrode it and a conflict warning fired; entry names the hierarchy and links to `examples/edge_kwarg_hierarchy.ipynb` as the canonical reference (Amendment 4), (6) `graph_directed` inference — building from nodes/edges with `graph_directed` unset auto-infers directedness from the requested metrics (e.g. `in_degree` implies directed, `triangles` implies undirected), which can surprise users; deliberate overlap with the gated Workstream E blurb in `wiki/wiki/plans/graph-metric-conflict-validation.md`, see Amendment 4. Each of gotchas 4-6 is a few sentences plus links, no demo build-out. Planner note: notebook-author may add gotchas from the viz-quality-bar skill and examples corpus, but keep the page short — it is an index of pitfalls with links, not a re-teaching of each topic.
- Cross-references the WS-A `edge_coverage` property in the intra-axis section ("check coverage, notice undrawn edges, add a repeat axis"), so WS-A must land first or in the same MR.
- Uses built-in datasets already established in the corpus (`example_hive_plot` and/or `nx.karate_club_graph()`); no new datasets.
- Prose follows the writing-voice rules (no em-dashes, no AI filler, length discipline).
- Listed in `index.rst` and `conf.py` thumbnails; `make test-nb` passes for the new notebook; `make docs` builds with no new warnings.
- Editorial-critic notebook review and viz-critic figure pass recorded.

**Notebook-coherence audit:** net-new notebook; tutorial genre; documents `HivePlot` (plus the new property); datasets `example_hive_plot` / karate club, both established in the corpus. No genre drift or dataset additions to sign off.

## Branch / MR sequencing

Current branch `46-more-streamlined-networkx-usage-and-support` carries in-flight v0.28 work. This plan's work belongs on a **fresh branch off master after that MR merges, as a separate MR** (recommended in the originating discussion; nothing here depends on unmerged branch-46 internals except Example 3's `graph=` idiom, which will be on master by then). Both workstreams can ship in one MR, WS-A first.

## Plan amendments

### Amendment 1 (2026-06-10) — In-scope tweak: repr clause format pinned, percent-only

Gary chose the api-critic's recommended short form: `f"{coverage.total:,} edges ({pct}% drawn), "` (e.g. `"5,100 edges (82% drawn), "`), no n/N in the repr — the clause already carries the total and the `edge_coverage` property is the precision channel. Exact spec (integer percent, inward-rounded boundaries, thousands separator, degenerate `"0 edges, "` form) pinned in Design notes "Repr display rule"; Goal and "Patterns this replaces" updated to match. Resolves the api-critic planning pass's repr-discrepancy pin-before-dispatch item.

### Amendment 2 (2026-06-10) — In-scope tweak: `HivePlotMatrix` coverage recorded as deliberate non-goal

Not a deferral awaiting demand. Gary's rationale is conceptual, not merely "per-cell access already works": a matrix-level coverage aggregate doesn't make sense ("it doesn't make sense at the matrix level") — edges appear in multiple cells by design, so any matrix-level number is semantically muddled, not just redundant. Recorded under Out of scope; Alignment open decision closed; Design notes HPM sentence updated. `HivePlotMatrix.__repr__` stays untouched and the WS-A repr test surface is final.

### Amendment 3 (2026-06-10) — In-scope tweak: `EdgeCoverage` export location pinned

Per the api-critic recommendation, accepted by Gary: define `EdgeCoverage` in `src/hiveplotlib/hiveplot.py` next to `BaseHivePlot`; re-export from the `hiveplotlib` top-level `__init__` (which already re-exports from `hiveplotlib.hiveplot`). `from hiveplotlib import EdgeCoverage` is the supported import path for annotations and `isinstance` checks. Naming audit, WS-A Files, and WS-A done-when updated. Resolves the api-critic planning pass's export-location pin-before-dispatch item.

### Amendment 4 (2026-06-10) — In-scope tweak: WS-B gotchas page expanded from three to six entries

Gary approved (2026-06-10 docs-gaps discussion) adding three gotchas to WS-B's coverage list, keeping the page's character (an index of pitfalls with links, not a re-teaching of each topic; each new entry is a few sentences plus links):

4. **Parallel-edge collapse on graph ingestion.** Networkx multigraph ingestion collapses parallel edges last-write-wins; a warning fires by default (behavior from `wiki/wiki/plans/graph-metric-conflict-validation.md`, implemented at `src/hiveplotlib/hiveplot.py:258` `_warn_if_parallel_edges_collapse`). Entry covers what happens, why, what the warning means, and the `graph_multigraph=True` workaround.
5. **Edge-kwarg-hierarchy overrides.** A higher-priority kwarg in the hierarchy overrode the one the user set, with a conflict warning. Entry names the hierarchy and links to `examples/edge_kwarg_hierarchy.ipynb` (canonical reference per CLAUDE.md).
6. **`graph_directed` inference.** With `graph_directed` unset, directedness is auto-inferred from requested metrics (`in_degree` implies directed, `triangles` implies undirected). **Overlap-is-deliberate decision:** Gary explicitly decided this earns its own gotcha entry EVEN IF the gated Workstream E notebook blurb in `graph-metric-conflict-validation.md` (not yet reviewed) also covers it. A later session must not "deduplicate" this entry away against that blurb.

Provenance: these emerged from comparing the gotchas page scope against a proposed conceptual-FAQ docs page; the FAQ page was **rejected** and its surviving entries fold into this gotchas page instead. Gotchas 4 and 6 document v0.28 in-flight branch-46 behavior, consistent with the existing Branch / MR sequencing (work starts after branch 46 merges); both behaviors verified present on the branch (feasibility: prose-plus-links only, no new entry points, no new datasets). Goal sentence and WS-B done-when updated to match; notebook-coherence audit unchanged (same genre, same datasets, no demo build-out for the new entries).

## Holdouts

- `src/hiveplotlib/datasets/toy_hive_plots.py:381-385` (`example_hive_plot` docstring note about fewer edges visualized without repeat axes): kept as-is — it documents the dataset generator's behavior, not a re-explanation the gotchas page replaces. Optionally add a cross-link in WS-B, not required.

## Implementation log

- (empty)
