# Plan: node/edge subsetting and highlighting

## Goal

Let a user emphasize or restrict to a subgraph without rebuilding the hive plot. Two mechanisms, deliberately different shapes: `subset` returns a new, fully independent `HivePlot` with geometry frozen to the parent (drill down, then keep working with a real `HivePlot` every backend already renders); highlight registers named accent state on the existing plot, rendered as an overlay layer on top of the base plot (emphasize in context). The story is explanatory drill-down: the hive plot surfaces heterogeneity, the user drills down or emphasizes to nail the story before moving to other visualizations. Node-driven and edge-driven selection both matter ("this clump and its subgraph"; "these dense edges and the nodes they touch"). This work deletes the four `data_subset` / `data_highlight` TODOs in `node.py` / `edges.py`; the confirmed design does not land there.

Brief-mode gate: ran (grill-me brief mode, 2026-07-06, six-question interview; extraction record maintainer-confirmed and folded into this section, the audits, and the workstreams). Settled items below are not up for relitigation.

Settled by the interview (each detailed once, in its operative section):

- Subset = cheap derived object: new independent `HivePlot`, copy semantics, frozen geometry (node placements, edge curves, and axis ranges identical to parent). Memory note: cost proportional to subset size; worst case (subsetting a huge plot to most of itself) transiently holds both; accepted.
- Highlight = registered state on `HivePlot`: repeated calls, each registering one named highlight and returning its key (the tags precedent). Surface floor: register / update / remove / inspect by key. One call may combine `nodes=` and `edges=` into a single group (one key, one color, one story). Multiple simultaneous highlights from day one.
- Closure semantics, principled asymmetry stated out loud: subset RESTRICTS, so it is induced (node-subset keeps edges with both endpoints in the set; edge-subset keeps the edges plus incident nodes only; NetworkX subgraph/edge_subgraph semantics). Highlight RADIATES, so it defaults to incident (nodes light up plus ALL touching edges; edges light up plus endpoint nodes); induced remains the narrowing option (cohesion story vs context story). Closure generalizes to ego-graph expansion via a hops-style parameter: k=1 is the motivated case, k>=2 comes along mathematically but is not featured. The same machinery gives subset a "zoom to this node's neighborhood" story.
- Rendering: highlights draw as an overlay layer on top, outside and above the edge-kwarg hierarchy; the hierarchy's documented contract is untouched (base layer unchanged, no new conflict-warning cases). Accepted trade-off: elements under a highlight are double-plotted (base plus overlay); fine in practice, hard to remove for little gain.
- Selection vocabulary floor: nodes by explicit IDs or boolean mask; edges by boolean mask or axis pair. Edge IDs are out (complicated, not load-bearing). Condition shorthand (narwhals expressions) is committed, as its own workstream. Region selection is a final-stretch workstream, explicitly cuttable.
- Motivating use case raising per-call-cheapness stakes: sweep-to-GIF, filter along a variable and render each frame from the same parent plot, animation as a fourth dimension.

Non-goals:

- **P2CP.** Deliberately left behind in development. Guard against accidental breakage only.
- **Lasso / interactive selection.** Belongs to the interactive-explorer/anywidget track; this plan ships the stable programmatic selection vocabulary a future lasso emits into.
- **Animation API.** The GIF sweep is a matplotlib notebook pattern, not surface.
- **Construction-time subset.** Not offered.
- **Auto-dimming the base layer.** Explicitly rejected ("a mistake"); see Default justifications.
- **Edge IDs as selection vocabulary.**
- **Construction-time highlight — rejected** (api-critic verdict 8). A highlight is (selection, closure, hops, key, color), dict-shaped in an already-wide constructor family (`from_variable_sweep` is ~30 params); no walked journey needs it, and `add_highlight` right after construction is the same line count.

## Execution gate

**Implementation is underway** (maintainer opt-in, 2026-07-07; see Alignment / Plan amendments 14). Rather than wait for MR !36 to merge, the work builds NOW on a worktree branch `59-subset-highlight` cut from `53-scale-hiveplotlib-to-larger-networks`. Building off branch 53 satisfies the same technical constraints the original "wait for !36" gate protected: the scaling representation from !36's branch is present, WS A avoids the `node.py` / `edges.py` overlap, and WS D's narwhals-boundary dependency is met. Accepted residual: branch 53 is an unmerged Draft the maintainer may still revise, so WS A/D built on it may need a rebase when !36 lands.

Draft MR "Closes #59" targets branch 53 (assigned gjkoplik), to be retargeted to master after !36 merges. Agents never merge.

Mutual sequencing note: the edge-coverage-and-gotchas plan gates on branch 46 and also touches `edges.py`; whichever executes first, the other rebases its expectations (no shared surface conflict is expected: coverage reads edge membership, this plan filters it).

Dispatch-time re-verify: **satisfied** (the re-grep ran and passed against branch 53's tip). The file:line pins in this plan are branch-53-relative, and sibling plans (edge-coverage, unify-axes) may land first, so before each new workstream dispatches, re-grep the pinned seams so it doesn't build against stale offsets: `_construct_edge_subset_curves` (hiveplot.py:1571), `_require_in_memory_edge_subsets` (hiveplot.py:2284), the `relevant_edges` positional-array note (edges.py:153-159), `subset_node_collection_by_unique_ids` (node.py:602), and the four `data_subset`/`data_highlight` TODOs (node.py:170-171, edges.py:143-144). A counts-and-groupings reconciliation is cheap. Disposes the adversary's cold item 8.

## Alignment (grill)

Brief mode ran 2026-07-06 (see Goal). Post-plan grill ran 2026-07-07 (inline). Emergent changes route to orchestrator amend-plan (see Plan amendments); this section is the record only, not hand-edited plan logic.

**Wave 1 (premise and scope).**

- Highlight registry KEPT. The adversary's item-1 reasoning ("hand-composable, therefore delete") is the weak form and is rejected; the maintainer's rebuttal is the lead justification: you write a high-level API so users think at the high level, the same reason a high-level API exists over a usable low-level one. Concrete non-hand-composable tail: HPM broadcast in one call, datashader (no draw-on-top primitive), incident closure over an induced-only subset child. The re-scope-behind-a-checkpoint alternative is rejected.
- Workstream I KEPT. Its only non-redundant residual once condition shorthand ships is selection by normalized PLACEMENT along an axis (value ranges are already expressible via expressions plus an axis pair); that residual is I's stated justification.
- Registry lands on `HivePlot`, not `BaseHivePlot`. Moving a method down the hierarchy later (to grant P2CP highlights) is non-breaking; guarding a base-class surface against P2CP now is a permanent cost. Narrow-now keeps the future door open at zero cost. The heavy P2CP non-exposure test is downgraded to a cheap "P2CP still renders after this lands" regression.
- Subset child inherits NO highlights by default (a selection partially surviving the filter would be a confusing half-lit artifact; re-registering on the child is one line). Confirms the plan's existing lean.

**Wave 2 (composed semantics, cheap contract, lazy hops).**

- Composed-selection pipeline PINNED: resolve node seeds, resolve edge seeds (edges plus their endpoint nodes), union into one node set, expand by `hops`, then apply closure. Subset locks induced (undrawable-safe by construction); highlight defaults incident. Principle: incident is safe for highlight (paint over an already-drawn base) but banned for subset (subset removes nodes, so an incident edge to a dropped node is undrawable). The maintainer is on board and will review this closely at final review rather than settle it deeper in theory now.
- "Cheap" pinned as a TESTED contract: `subset` slices already-computed geometry and must not call into curve construction. Maintainer-added guard: `subset` yells legibly if invoked before the plot geometry is computed (distinct from the empty-but-valid selection, which returns a renderable empty plot).
- Lazy Dask edges: `hops=0` selection and highlight are fully lazy (the common case, never in question). Only `hops>0` (ego expansion) forces a per-hop compute. WS A floor routes `hops>0` on lazy frames to a legible in-memory error; a new self-contained, revertible Workstream J attempts real support (per-hop `is_in`/`unique`/`compute` over ID columns only, honoring the no-full-materialize invariant), with a save-before-cut rule (diagnose Dask warts, as the scaling sprint did with arrow-string / thread-safety / kernel fixes, before degrading to the error).

**Failure-mode wave.** Modes named into `## Failure modes`. Provocations the maintainer considered and DISMISSED (recorded as examined, not open): frozen-geometry standalone footgun (choosing too small or dull a subset is the user's call; a dull highlight is still information), registry-hollow (drawing on top stays trivial AND the registry adds a right-ergonomics way of working), GIF as pure threshold artifact (Datasaurus implications are inherent to any hive plot; recast as a positive aim).

Open decisions still on the maintainer: none blocking. Close-review items flagged for his final review: the composed-semantics pipeline, and whether each notebook lands a genuinely interesting anecdote.

**Execution opt-in (2026-07-07).** The maintainer opted into autonomous execution ("work autonomously on a worktree... when you make the first commit, tie that to a merge request"). Auto-dispatch mode is ON (the failure-mode rubric exists, so the plan is eligible): the dispatching session runs workstream to workstream through the full gate battery (critics + adversary post-impl + qa-engineer), one commit per workstream, halting only on `must-fix` / `STATUS: BLOCKED` / any finding bearing on a downstream not-yet-run workstream. Base-branch decision: build off `53-scale-hiveplotlib-to-larger-networks` (not master), so the scaling representation is present, WS A avoids the `node.py`/`edges.py` overlap, and WS D's narwhals dependency is satisfied; this overrides the plan's original "wait for !36" gate (see Execution gate / Plan amendments). Draft MR "Closes #59" targets branch 53, assigned gjkoplik, retargeted to master after !36 merges; never merged by an agent. Worktree at `~/repos/hiveplotlib-59-subset-highlight`; this plan file stays canonical in the maintainer's main-checkout wiki (agents read and update it there, not the worktree's pinned copy).

## Failure modes

Named by the maintainer (grill failure-mode wave, 2026-07-07), one line each in his words:

- **The notebook-anecdote test is the real gate.** On paper this is an obvious win, but the feature is hollow if, when the subset and highlight gallery notebooks get built (Workstreams G, H), we cannot tell a genuinely interesting anecdote for *each* mechanism on its own. HPM cross-panel anecdotes are expected strongest, but the single-`HivePlot` subset and highlight artifacts must also carry a real story rather than lean on HPM. Not knowable until G/H are built; the maintainer is confident but names this as the thing to watch.
- **The GIF sweep must reveal nuance, not just motion.** It earns its place only if it surfaces an additional relationship, not merely animated threshold choreography. (The Datasaurus caveat is inherent to any hive plot, not specific to the GIF, so this is recorded as a positive aim, not a defect.)

Inherited ahead of the wave (maintainer-named on the scaling-large-networks plan; adopted per research-liaison pre-task findings):

- Plausible picture of the wrong data: positional-vs-label selection divergence on non-RangeIndex user frames produces a clean-looking subset/highlight of the wrong rows. Explicit testing obligation for ID / mask / expression selection (cascaded into Workstreams A and D).

## Prior ADRs / design docs

Populated from research-liaison pre-task findings:

- `wiki/wiki/adr/0001-networkx-integration.md` — (a) naming house style: hiveplotlib deliberately does not mirror NetworkX names 1:1; borrowing nx method vocabulary (subgraph / edge_subgraph / ego_graph) is fine but native naming wins on conflict (engaged in the Naming audit, not treated as automatic). (b) The "deliberately avoids tracking state" philosophy supports copy semantics over live parent-linkage. (c) The GraphMetricsSpec deferral is the template for the selection vocabulary: IDs / masks / axis pairs stay one-liners; a config object would need a named justification (none is proposed here).
- `wiki/wiki/adr/0002-performance-regression-harness.md` — `subset()` advertised as cheap, and the datashader-highlight workstream, are performance claims; anything touching rasterization / curve-construction paths must clear the equivalence wall and same-run ratio gates (absolute timings never gate). Cascaded into Workstreams A and F.
- `wiki/wiki/plans/scaling-large-networks.md` (active; the execution gate). Hard constraints, cascaded into Workstreams A and D: (1) `Edges.relevant_edges` is integer row-position arrays applied positionally (`df.iloc[positions]`, edges.py:153-159); induced-closure / edge-membership computation must consume that representation. (2) `frames.py` carries `LazyEdgeSubset` and a Dask-native non-materializing route; subset/highlight edge filtering must never force-materialize a lazy edge subset (`_construct_edge_subset_curves` at hiveplot.py:1571 / `_require_in_memory_edge_subsets` at hiveplot.py:2284 is the seam); node data must fit in RAM (no Dask node subset), a real ceiling stated in Workstream A. (3) The narwhals-expression condition shorthand truly depends on !36's WS4 (narwhals input boundary), not mere sequencing. (4) Failure mode 4 inherited above.
- `wiki/wiki/plans/edge-coverage-and-gotchas-docs.md` (active) — coverage on a derived view: maintainer settled no special casing, a subset child computes its own `edge_coverage` over its own filtered data; highlight never changes coverage (cascaded into Workstream A's done-when). Transferable precedents: property-vs-method rule (zero-arg cheap read-only introspection = property, applied to the highlight inspection surface); the EdgeCoverage top-level re-export pattern as the model for any new return/state type; its naming-audit discipline (every alternative rejected with a named reason).
- `wiki/wiki/plans/hiveplot-unify-axes.md` (active) — the "same name, same shape" HivePlotMatrix mirroring pattern to follow for broadcast; `unify_axes=True` keeps subset panels comparable and is complementary to frozen geometry (cascaded into Workstream E).
- `wiki/wiki/plans/interactive-wasm-explorer.md` + `wiki/wiki/plans/anywidget-hive-plot-widget.md` (active) — no scope collision; boundary: this plan ships the primitives, interactive/lasso selection lives in the explorer track. Naming lesson transfer: lead docstrings with the user's word, not the mechanism's.
- `wiki/wiki/plans/deprecation-policy-and-warning-categories.md` (active) — if it ships first, any new warnings here (e.g. highlight key collision) use its named warning-category machinery; `stacklevel=3` per house rules. This feature is net-new API, no deprecations of its own.
- `wiki/wiki/analyses/hive-plots-for-knowledge-graphs.md` — this feature partly realizes documented research asks. Gap #6 (first-class edge subset-by-attribute for a view) = subset + condition shorthand. Gap #7 / pattern #7 (fixed layout, swappable edge layers for model-outcome comparison, e.g. link-prediction TP/FN/FP) is served by the **highlight overlay model** (multiple named highlights over one frozen base), not the derived-plot model; a per-layer derived plot would need side-by-side panels where the pattern wants one canvas. The "one hive plot per question = rational rendering of a typed subgraph" reframing grounds subset conceptually.
- `wiki/wiki/concepts/fixed-layout-comparability.md` — hard correctness constraint: a subset must NOT re-infer axis value ranges from its own reduced data, or comparison to the parent silently distorts; frozen geometry means freezing axis RANGES, not just node placements. Load-bearing correctness rule; cascaded into Workstream A's done-when (pinned `vmin`/`vmax`).
- `wiki/wiki/analyses/same-stats-different-graphs-grounding.md` — pixel-identical node positions verified as the comparability hero; storytelling precedent for frozen geometry.
- `wiki/wiki/analyses/gnn-research-directions.md` (direction 3) + the hetionet link-prediction view — live consumers of "fixed layout, per-panel edge subsets, animated over training"; motivating context for the HivePlotMatrix broadcast workstream.

## Patterns this replaces

- `data_subset` / `data_highlight` TODO comments (the rejected constructor-attribute design) at: `src/hiveplotlib/node.py:170`, `src/hiveplotlib/node.py:171`, `src/hiveplotlib/edges.py:143`, `src/hiveplotlib/edges.py:144`. Delete all four (Workstream A). Repo-wide grep confirms these are the only `data_subset` / `data_highlight` references.

Workstream J (added, Amendment 5): net new. It replaces this plan's own WS A floor behavior (the `hops>0`-on-lazy in-memory error) with real support when it lands, not any shipped repo surface; no repo-wide old pattern.

Amendment 18 (WS D lazy edge-expression correctness fix): replaces this plan's OWN just-shipped unreleased lazy composition (the literal edge-filter `_restrict_lazy_subset(extra_predicate=)` over a broad axis-spanning seed, which produced a strictly narrower subgraph than in-memory) with the faithful bounded-endpoint-compute path. No shipped/released repo surface and no repo-wide old pattern — this corrects an unreleased same-branch WS D path before it ships.

Everything else is net new API surface.

## Default justifications

- Highlight registry KEPT (not deletable in favor of hand-composing over `subset`): you write a high-level API so users think at the high level, the same reason a high-level API exists over a usable low-level one. The registry is the right-ergonomics way of working; drawing on top by hand stays available. Concrete tail that is genuinely NOT hand-composable: HPM broadcast lights one selection in every panel in one call; datashader has no draw-on-top primitive (the overlay is a single rasterization design, not manual layering); incident closure over an induced-only subset child is not expressible by re-rendering the child. Disposes the adversary's cold item 1.
- Subset closure = induced (fixed, not a parameter): subset restricts, so a kept edge with a missing endpoint would be undrawable; node-subset keeps edges with both endpoints in the set, edge-subset keeps the edges plus incident nodes only.
- Highlight `closure="incident"` default: highlight radiates; the common question is "what does this clump touch?" (context story), so nodes light up plus all touching edges by default; `"induced"` is the explicit narrowing option (cohesion story).
- `hops=0` default on subset/highlight selection: the default answer to "select these" is exactly these; neighborhood expansion (`hops=1`, the motivated ego-graph case) is an explicit ask. k>=2 works mathematically but is not featured.
- No `copy=` parameter (subset always copies): a view would thread mask-awareness through every read path and create aliasing surprises; maintainer confirmed after pushback.
- Frozen geometry (not a parameter): comparability with the parent is the point of the drill-down; no re-sort, no re-normalization. The user may recompute on the child themselves; their choice.
- No auto-dimming of the base layer under highlights: explicitly rejected by the maintainer ("a mistake"); default colors are muted already, muting further is one kwarg (e.g. color via `all_edge_kwargs`), and the notebooks practice the emphasis/de-emphasis move (dark gray, thinner base edges) rather than the API encoding it.
- Default accent palette, assigned first-unused: 5-6 deliberate colors, distinct from the muted defaults, so each of several simultaneous highlights reads as its own story with zero color bookkeeping by the user; specifics fall under the viz-quality-bar skill.
- One combined `nodes=` + `edges=` highlight call = one key, one color: one story per registered highlight; users wanting two colors make two calls.
- Auto-generated highlight key when the user doesn't name one: mirrors the edge-tag precedent (`_find_unique_tag`, hiveplot.py:1023); the call returns the key either way.
- Subset child `edge_coverage`: computes its own coverage over its own filtered data, no special casing; highlight never changes coverage (it adds no edges, only paint).

## Naming audit

Names used in this plan's examples are **final**. The maintainer delegated final say on the method-name family to the api-critic planning pass; the api-critic exercised that delegation (verdicts recorded in "API Critic's take (planning mode)" items 1-4). ADR 0001's precedent applies (nx method vocabulary is borrowable, native naming wins on conflict).

- Subset method: `subset` (**final**). Receiver and return value are plots, not graphs, so `subgraph` would lie about the return type; nx splits `subgraph` (nodes) / `edge_subgraph` (edges) while this method takes both `nodes=` and `edges=`, so neither nx name maps 1:1 and ADR 0001's native-wins-on-conflict applies; `filter` shadows the builtin and describes the data op, not the plot product. Precedent: `subset_node_collection_by_unique_ids` (node.py:602) is already the library's restriction word. Borrow nx vocabulary in the docstring instead. (api-critic verdict 1.)
- Neighborhood parameter: `hops` (**final**). `radius` is polar geometry in this library (nodes sit at radii; region selection filters by radial extent), so `radius=1` would read as a geometric filter; `k` is not self-documenting in a tooltip. Docstring still says "equivalent to `networkx.ego_graph` with `radius=hops`". (api-critic verdict 2.)
- Highlight family: `add_highlight(...) -> key` / `update_highlight(key, ...)` / `remove_highlight(key)` / `highlights` (read-only property) (**final**). Registration/mutation verbs are the house families (`add_axes` / `add_nodes` / `add_edges` / `add_edge_kwargs`; `update_edges` / `update_axis` / `update_node_viz_kwargs`). Bare `highlight()` rejected (breaks add/update/remove symmetry, near-collides with the `highlights` property in tab-completion); `remove_` over house `reset_` because the registry is by-key, not wholesale. (api-critic verdict 3.)
- Closure parameter: `closure="incident" | "induced"` (**final, name and values**). The values are the exact graph-theory terms the docs use to explain the asymmetry. Rejected: `include=` / `extent=` (spatial connotation) / `keep_edges=` / `edge_policy=`. The `Literal["incident", "induced"]` tooltip disambiguates the momentary transitive-closure misread. (api-critic verdict 4.)
- Condition shorthand surface: `nodes=` / `edges=` accept a narwhals expression directly wherever selection is accepted; no new parameter (**final; `where=` rejected**). Decisive: the example dataset carries `low`/`med`/`high` on both node and edge frames, so a bare `where=` is ambiguous about its target frame and would need `node_where=`/`edge_where=` (two new params) while `nodes=`/`edges=` already carry the target. (api-critic verdict 6.)
- Multi-tag edge masks: a bare boolean mask applies when the `Edges` hold a single tag; multi-tag edge data takes a dict of masks keyed by tag, mirroring the `Edges` data dict the user already built. This convention is authorized here (feasibility audit: `Edges._data` is a dict of frames keyed by tag, so a bare mask is ambiguous under multiple tags); docstring coverage required in Workstream A.
- Prose-only terms: "subset", "highlight", "induced", "incident", "neighborhood (ego graph)", "overlay". Docstrings lead with the user's word, not the mechanism's (interactive-explorer naming lesson).

Feasibility trace (net-new entry points): node IDs map to `NodeCollection.unique_id_column` values; boolean masks are row-aligned (positional) against `nodes.data` / the per-tag `Edges` frames; axis pairs map to `HivePlot.axes` keys; expressions map to columns of the user's frames via narwhals. All selection parameters resolve to documented data-model elements; the one new convention (multi-tag mask dict) is authorized above.

## API usage examples

### Proposed (from planner / Orchestrator)

Method/parameter names are final (api-critic verdicts 1-6; the example code already uses these names).

```python
# Example 1: user built a plot, spotted a clump of high-"high" nodes, drills down to it
# Example data:
from hiveplotlib.datasets import example_hive_plot
from hiveplotlib.viz import hive_plot_viz

hp = example_hive_plot(num_nodes=100, num_edges=200, seed=0)

# Call site:
clump = hp.nodes.data["high"] > 25  # boolean mask over node data
sub_hp = hp.subset(nodes=clump)  # induced: keeps edges with BOTH endpoints in the clump
fig, ax = hive_plot_viz(
    sub_hp
)  # a real HivePlot; placements and axis ranges frozen to parent
```

```python
# Example 2: zoom to one node's neighborhood (ego story), or to one axis pair's edges
# Example data: reuses hp from Example 1.

# Call site:
ego = hp.subset(nodes=[17], hops=1)  # node 17, its neighbors, and the connecting edges
a_to_b = hp.subset(
    edges=("A", "B")
)  # edges between axes A and B, plus incident nodes only
```

```python
# Example 3: emphasize in context; register, restyle, inspect, remove named highlights
# Example data: reuses hp from Example 1.

# Call site:
key = hp.add_highlight(
    nodes=hp.nodes.data["high"] > 28
)  # nodes + ALL touching edges (incident default)
core = hp.add_highlight(
    nodes=hp.nodes.data["med"] > 15,
    closure="induced",  # narrow: only edges within the group
)
fig, ax = hive_plot_viz(
    hp
)  # base layer unchanged; overlays drawn on top, one accent color each
hp.update_highlight(key, color="goldenrod")  # restyle by key
hp.highlights  # inspect registered highlights
hp.remove_highlight(core)
```

```python
# Example 4: condition shorthand + the sweep-to-GIF pattern (per-call cheapness is the point)
# Example data: reuses hp from Example 1; edges carry "low" / "med" / "high" columns.
import matplotlib.pyplot as plt
import narwhals as nw

# Call site:
dense = hp.subset(
    edges=nw.col("low") > 5
)  # filter on a variable not used for partition or sorting

for i, threshold in enumerate([2, 4, 6, 8]):
    frame_hp = hp.subset(edges=nw.col("low") > threshold)
    fig, ax = hive_plot_viz(
        frame_hp
    )  # identical geometry every frame; stitch to GIF downstream
    fig.savefig(f"frame_{i}.png")
    plt.close(fig)
```

```python
# Example 5: HivePlotMatrix broadcast; one call, every panel, to see where a group behaves uniquely
# Example data:
import pandas as pd
from hiveplotlib import Edges, HivePlotMatrix, NodeCollection
from hiveplotlib.datasets import example_edge_data, example_node_data

node_df = example_node_data(num_nodes=100, seed=0)
node_df["group"] = pd.qcut(node_df["low"], q=3, labels=["A", "B", "C"])
nodes = NodeCollection(data=node_df, unique_id_column="unique_id")
edges = Edges(data=example_edge_data(nodes=nodes, num_edges=200, seed=0))
hpm = HivePlotMatrix.from_variable_sweep(
    nodes=nodes,
    edges=edges,
    partition_variable="group",
    sorting_variables_list=["low", "med", "high"],
    unify_axes=True,
)

# Call site:
key = hpm.add_highlight(nodes=node_df["high"] > 28)  # same selection lit in every panel
sub_hpm = hpm.subset(
    nodes=node_df["group"] == "A"
)  # broadcast subset, panel geometry frozen
```

```python
# Example 6: dense-edges story at scale; a subset child is a real HivePlot, so datashader just works
# Example data: reuses hp and dense from Example 4.
from hiveplotlib.viz.datashader import datashade_hive_plot_mpl

# Call site:
fig, ax, im_nodes, im_edges = datashade_hive_plot_mpl(
    instance=dense
)  # zero new per-backend code
```

### API Critic's take (planning mode)

Ran 2026-07-06 (planning mode). The proposed snippets are **agreed as drafted** modulo the decisions and obligations below. All example call sites were verified against the working tree: `example_hive_plot` (`num_nodes`/`num_edges`/`seed`), `hive_plot_viz`, `datashade_hive_plot_mpl` (4-tuple return, `instance=` kwarg), and `HivePlotMatrix.from_variable_sweep` (all five kwargs incl. `unify_axes`) all exist with the shapes the examples use; Example 4's claim that edges carry `low`/`med`/`high` columns matches `example_hive_plot`'s docstring.

**Naming verdicts (maintainer-delegated final say):**

1. **`subset` — final.** The receiver and the return value are plots, not graphs; `hp.subgraph(...)` returning a `HivePlot` lies about the return type. Rejected with named reasons:
   - `subgraph`: nx splits `subgraph` (nodes) / `edge_subgraph` (edges), so a combined `nodes=`/`edges=` method matches neither nx signature while the name promises nx's exact surface; ADR 0001's native-wins-on-conflict applies. Borrow nx vocabulary in the docstring instead ("NetworkX `subgraph`/`edge_subgraph`/`ego_graph` semantics"), which gets the googlability without the return-type lie.
   - `filter`: ecosystem-consistent for row restriction (pandas/polars) but describes the data operation, not the derived-plot product, and shadows the Python builtin.
   - Supporting precedent: "subset" is already the library's user-facing restriction word (`subset_node_collection_by_unique_ids`, node.py:602). [low-confidence] "Edge subset" also already means a *tagged per-axis-pair edge grouping* in user-facing text (`reset_edges` docstring, `LazyEdgeSubset`); one docstring sentence distinguishing plot-level `subset()` from tagged edge subsets closes the collision.

2. **`hops` — final.** In this library `radius` is polar geometry: nodes literally sit at radii, and Workstream I's region selection will select by radial extent, so `radius=1` on subset/highlight reads as a geometric filter — a name that lies to exactly this library's users. nx `ego_graph`'s `radius` also admits weighted distances this feature doesn't support. `k` rejected: not self-documenting in a tooltip. Docstring should still say "equivalent to `networkx.ego_graph` with `radius=hops`" for the nx-literate.

3. **`add_highlight` / `update_highlight` / `remove_highlight` / `highlights` (property) — final.** Registration/mutation verbs are the house families (`add_axes`/`add_nodes`/`add_edges`/`add_edge_kwargs`; `update_edges`/`update_axis`/`update_node_viz_kwargs`), and `hp.add_<tab>` places the feature among its siblings for discoverability. Bare `highlight()` rejected: breaks the add/update/remove symmetry and sits one character from the `highlights` property in tab-completion (a property/method near-collision trap). `remove_` over the house `reset_`: `reset_edges` means "delete stored connections wholesale," the wrong shape for a keyed registry; `remove` matches the by-key contract (and matplotlib's `Artist.remove`).

4. **`closure="incident" | "induced"` — final, name and values.** The values are the exact graph-theory terms the docs use to explain the asymmetry; keep them. Rejected names: `include=` (doesn't say included-relative-to-what), `extent=` (spatial connotation colliding with region selection), `keep_edges=`/`edge_policy=` (understates that incident closure also pulls in endpoint *nodes* on edge-seeded selection). Accepted cost, worth one docstring clause: graph-literate readers may momentarily hear *transitive closure*; the `Literal["incident", "induced"]` tooltip disambiguates immediately.

**Queued decisions:**

5. **`nodes=` + `edges=` union semantics in one method — accepted.** Two supporting observations: (a) the algebra is coherent and should be documented as such — arguments within one call union; chained calls intersect (`hp.subset(nodes=clump).subset(edges=("A", "B"))` = the clump's A-to-B edges); union across two *separate* subset calls is not otherwise expressible. (b) Union is what makes an incident-style drill-down expressible at all: `hp.subset(nodes=clump, edges=touching_mask)` = clump + touching edges + endpoints, *without* the neighbor-neighbor edges `hops=1` induced closure would add. [worth-discussing] The intersection reading of a combined call is the plausible misread; the union contract and the intersect-by-chaining recipe belong in the first lines of both params' docstrings and in the subset notebook (fold into Workstream A/G done-when via amend-plan).

6. **Expression-in-parameter — accepted; separate `where=` rejected.** Decisive: the example dataset itself carries `low`/`med`/`high` on BOTH node and edge frames, so a bare `where=` is ambiguous about its target frame; disambiguating needs `node_where=`/`edge_where=` (two new params) while `nodes=`/`edges=` already carry the target. `nodes=nw.col("high") > 25` reads as "nodes where." Obligation: docstrings show the mask form and the expression form side by side — the mask path stays the pandas-native route, so nobody is forced to learn narwhals to use the feature.

7. **Multi-tag mask dict — writable without reading source, conditional on pinning partial-dict semantics.** The dict mirrors the `Edges(data={tag: df})` the user already built, so the shape is learnable. [worth-discussing] Two obligations: (a) partial-dict semantics — recommend "absent tag = not selected" (dropped in subset, unlit in highlight); that reading is consistent with `edges=` naming *what you select* and is the only coherent one for highlight; name-and-reject the alternative (absent = kept whole) in the docstring. (b) Dict values should accept expressions as well as masks (per-tag frames may have heterogeneous columns, making one broadcast expression un-broadcastable); define a legible error, naming the tag, when a broadcast expression references a column missing from one tag frame.

8. **Construction-time highlight kwarg — rejected; don't ship.** A highlight is (selection, closure, hops, key, color): dict-shaped at construction, one clean call after. Constructors are already the library's widest surfaces (`from_variable_sweep` is ~30 params); no walked journey needs it; the natural moment for a mask/expression is after the user has *seen* the plot. The interview's honest note stands: `add_highlight` immediately after construction is the same line count.

**Journey findings (walked: clump drill-down side-by-side; datashader dense-edges; register/restyle/remove named highlights; HPM broadcast; GIF sweep loop):**

9. [worth-discussing] **Empty selection must not raise.** A GIF threshold sweep that passes the data's max produces a legitimate all-base frame; a raise mid-loop kills the plan's own motivating pattern. Pin in Workstream A's done-when: an empty subset returns a valid, renderable `HivePlot`; an empty highlight registers as a no-op overlay; test both.

10. [worth-discussing] **`update_highlight` must accept selection, not just style.** The GIF loop's highlight variant is one key re-aimed per frame (`hp.update_highlight(key, nodes=nw.col("low") > t)`); a style-only update forces remove+re-add churn. Recommend `update_highlight` accept the full `add_highlight` kwarg set, replacing the named fields.

11. [worth-discussing] **Pin the `highlights` property's value shape as a named type.** Journey (c) inspects it; a raw dict-of-dicts makes the notebook read `hp.highlights[key]["closure"]` string-key spelunking. The plan already cites the EdgeCoverage top-level re-export precedent — apply it: a small named record per highlight.

12. [worth-discussing] **Boolean-mask-vs-IDs ambiguity for plain Python lists.** Example node IDs are 0..99 and `True == 1`, so `nodes=[True, False, ...]` vs `nodes=[0, 1]` is undecidable by inspection. Pin the rule: bool-dtype arrays/Series = mask; everything else, Python lists included, = IDs; document "pass a numpy/pandas/polars boolean array for masks." Also accept a scalar node ID — `hp.subset(nodes=17, hops=1)` is THE motivated ego call; precedent: `subset_node_collection_by_unique_ids` wraps a bare Hashable (node.py:614).

13. [worth-discussing] **Axis-pair selection needs its directionality stated.** The `reset_edges` precedent (`axis_id_1`/`axis_id_2` with `a1_to_a2`/`a2_to_a1` both defaulting True) implies bidirectional-by-default; `edges=("A", "B")` should select both directions and say so in the param docstring's first sentence. No direction params now. Adjacent [low-confidence] notes: (a) accepting a *list* of axis pairs (`edges=[("A", "B"), ("A", "C")]`) is worth considering since union across separate subset calls is impossible (distinguishable from a single pair by element type); (b) pairs name literal axis IDs, so repeat-axis edges are ("A", "A_repeat") — say so, `example_hive_plot`'s docstring already trains users on repeat-axis edges.

14. [low-confidence] **Ego-comment precision.** Example 2's comment "node 17, its neighbors, and the connecting edges" undersells induced-on-the-ball: `hops=1` + induced closure includes neighbor-neighbor edges (exactly `nx.ego_graph` behavior). Keep docstring and notebook prose precise on this or users will think spoke-edges-only.

15. [low-confidence] **Highlight styling beyond one `color=`.** The walked journeys quickly want linewidth/size on the overlay ("make it pop"). Since the overlay renders through existing viz machinery, `node_kwargs=`/`edge_kwargs=` passthrough on add/update is likely near-free; ship it or record a named deferral in Workstream B. Similarly [low-confidence]: a clear-all affordance (zero-arg `remove_highlight()` mirroring `reset_edges()`'s all-None, or a separate `reset_highlights()`) — sugar, decide at implementation.

**Recurring patterns:** (1) Nearly all of this surface's risk is polymorphic-parameter ambiguity — `nodes=`/`edges=` each accept 3-4 types; every accepted type needs a one-line "how it's detected" rule in the docstring, and those detection rules are precisely what the inherited positional-vs-label failure mode should be tested against. (2) The plan repeatedly closes semantic gaps with docstring obligations (union algebra, partial dicts, directionality, ego precision); each such obligation should land as a Workstream done-when bullet via amend-plan, not stay in prose here, so the post-impl passes can hold them as contract.

**Workstream I region-select proposal (ran 2026-07-07, planning mode).**

**Verdict: CUT.** [confidence: high] Recommend cutting Workstream I and recording it as a deferred follow-up (the "select what I see" lasso affordance, revived when a real spatial-selection surface, not a placement-range restatement of a value threshold, is on the table). Reasoning below, then the exact parameter shape an engineer would implement against if the maintainer overrides to BUILD, so the deferral is concrete.

**1. Placement-select is NOT ergonomically distinct from value-select. It is the same operation in different units.** I read how placement is computed (`place_nodes_on_axis`, hiveplot.py:1075-1092): the sort value is clipped to `[vmin, vmax]`, normalized to `[0, 1]`, then scaled to the axis span. Placement (`rho`) is a **strictly monotonic linear** transform of the clipped sort value. I grepped both `hiveplot.py` and `p2cp.py` for `rank` / `argsort` / `linspace` / `rankdata` / spread / jitter: **zero matches.** There is no rank-based spreading anywhere. Consequences that gut I's justification:

   - "Top 20% of placement on axis A" is **exactly** `nw.col(sort_var) >= vmin + 0.8 * (vmax - vmin)` on A's sort variable, node-for-node. It is *not* "top 20% by rank," so the plan's hedge (WS I done-when, item 1) that "ties/spread can make placement != a clean value threshold" is **empirically false for this library**: placement IS the linear value map, so ties land at one placement and a value threshold catches them identically, and spread is that same linear map. There is no case where a placement range selects a different node set than the corresponding value threshold.
   - Clipping doesn't create a gap either: values past `[vmin, vmax]` pile at the endpoints, and a value threshold on the raw column captures those endpoint-piled nodes identically (both clip the same way).
   - The composed pipeline already unions a per-axis node condition with an axis pair in one call. The "patch between A and B where both endpoints sit high" story is already a one-liner today, post-WS-D: `hp.subset(nodes=(nw.col(a_sort) >= a_thr) & (nw.col(b_sort) >= b_thr), edges=("A", "B"))` (or the two-axis version unioned). Region-select adds a *normalized-units* skin over exactly this; it is ~100% expressible without itself, which is a stronger version of the "unjustified stretch" the adversary's cold item 6 already flagged.

**2. The one real residual is a small, non-API-shaped ergonomic tax, better closed with an accessor than a parameter.** To convert a normalized 0..1 placement into the value threshold above, the user needs the axis's sort-variable name and its `vmin`/`vmax`. Both already exist (`hp.axes["A"].sorting_variable`, `hp.axes["A"].vmin`, `hp.axes["A"].vmax`, axis.py:94/202-203), so the conversion is `vmin + frac * (vmax - vmin)` — a two-line helper, not a selection vocabulary. If the maintainer wants to soften even that, the proportionate move is a tiny `Axis` convenience (e.g. `axis.value_at_placement(frac)`), **not** a `region=` parameter that forks the polymorphic `nodes=`/`edges=` resolver. Adding a fourth selection idiom to a resolver already carrying mask / expression / scalar-ID / list / axis-pair / dict (see the "recurring patterns" note: polymorphic ambiguity is this surface's dominant risk) buys negative ergonomics for a value threshold in disguise.

**3. "Select what I see" is a real future affordance, but a placement-range parameter is not it.** The genuine version is a spatial lasso over the *drawn* `(x, y)` (or `rho`, `angle`) coordinates: an arbitrary polygon over the raster, which is fundamentally not expressible as a value threshold and is the thing a dense datashader user actually reaches for. A per-axis `(lo, hi)` placement range is a rectangle-in-normalized-sort-units, i.e. still a value threshold, so shipping it now would neither deliver the lasso nor compose with it (a future lasso keys off cartesian/polar geometry, not per-axis normalized sort fractions). Cutting now keeps the door open for the honest spatial surface later instead of spending the `region=` name on a value-threshold synonym. Record the deferral as "spatial lasso selection over drawn coordinates" so the revival target is the real affordance, not this rectangle.

**If overridden to BUILD (exact shape, so the deferral is concrete).** Do **not** add a `region=` parameter or a dataclass; extend the existing `edges=` axis-pair form with optional per-axis placement bounds, resolved to value thresholds internally, so it flows straight through the shipped pipeline and reads as a refinement of a vocabulary users already know:

```python
# axis pair (A, B) plus a normalized placement band per axis, fracs in [0, 1]:
#   keep the A-to-B edges whose A endpoint sits in the top 20% of A and
#   whose B endpoint sits in the top 20% of B.
hp.subset(edges=(("A", 0.8, 1.0), ("B", 0.8, 1.0)))

# highlight the same patch:
hp.add_highlight(edges=(("A", 0.8, 1.0), ("B", 0.8, 1.0)), color="orange")
```

Engineer notes for that shape: (a) distinguish it from the plain `("A", "B")` pair by element type (a 3-tuple `(axis_id, lo, hi)` of floats vs. two bare axis IDs), same detection discipline the resolver already uses for pair-vs-list-of-pairs; (b) internally map each `(axis_id, lo, hi)` to `vmin + lo*(vmax-vmin) <= sort_val <= vmin + hi*(vmax-vmin)` on that axis's `sorting_variable` and AND it into the axis-pair endpoint filter, so it composes with `hops`/closure with zero new pipeline; (c) validate `0 <= lo <= hi <= 1` with a legible error; (d) a docstring line stating outright that this is a normalized-placement convenience equivalent to the value threshold `nw.col(sort_var).is_between(...)`, so the user learns it's sugar, not a distinct capability. Keep it a one-liner; do not introduce a dataclass or a sibling method.

### API Critic — post-implementation review

```
Pending — invoke api-critic in post-implementation mode after each of Workstreams B, C, D, E, F, I, and J ships
(E is a mechanical propagation to a sibling class and still gets the pass, per house rule; J changes lazy-frame
behavior of subset/highlight, so it gets the pass too, see its own placeholder under Workstream J).
```

**Workstream A — `HivePlot.subset` (post-impl, 2026-07-07).**

```
Status: propose
API surface reviewed: HivePlot.subset(nodes=, edges=, hops=); module helpers _is_boolean_mask,
  _boolean_mask_to_numpy, _is_axis_pair, _is_axis_pair_list, _slice_curves (all module-private,
  correctly underscore-prefixed); SubsetBeforeBuildError (exceptions/hive_plot.py).
Verdict: the shipped `subset` MATCHES the plan's promises. Final names (`subset` / `hops`), the
  union algebra (nodes= + edges= union within a call, chaining intersects), the mask-vs-IDs rule
  plus scalar-ID ego call, multi-tag dict partial semantics (absent tag = not selected), axis-pair
  bidirectionality + repeat-axis literal IDs, and ego precision (hops=1 induced = nx.ego_graph incl.
  neighbor-neighbor edges) are all implemented and land in the docstring leading with the user's word.
  All my planning obligations (verdicts 1, 2, 5, 6, 7, 12, 13, 14; journey item 9) are honored, and the
  inherited positional-vs-label failure mode is tested on non-RangeIndex pandas and polars. Proposed
  Examples 1, 2, and 4 (the subset + condition-shorthand snippets) run as written against the shipped
  signature. Two doc-surface concerns and two low-confidence polymorphism edges below; none block.
Concerns:
  - [worth-discussing] The `:raises:` block on `subset` omits `InvalidAxisNameError`, which the axis-pair
    path raises on a bad axis name (e.g. `edges=("A", "Z")`) — at hiveplot.py:5444-5449 (docstring
    :raises: block) vs. the raise at hiveplot.py:5307. The behavior is real and tested
    (test_subset_axis_pair_invalid_axis_raises, hiveplot_test.py:8326), and a mistyped axis id is a
    plausible first-use error, but a user reading "what does subset throw" to write a try/except gets an
    incomplete answer (the `:raises:` block is the load-bearing catchable-error surface; the private
    helper documenting it is not equivalent).
    Suggested change: add `:raises InvalidAxisNameError: if an axis pair names an axis not in this hive
    plot's axes.` to subset's docstring.
  - [worth-discussing] A tuple passed to `nodes=` is silently read as ONE tuple-valued node ID, not a
    collection of IDs — at hiveplot.py:5191-5194 (`isinstance(nodes, Hashable)` catches a tuple before the
    collection branch). `nodes=(0, 1)` selects the single node whose unique id equals `(0, 1)` (almost
    always nothing, yielding a silent empty plot), whereas `nodes=[0, 1]` correctly selects two IDs. The
    docstring steers users to "a list / array of node IDs" and never mentions tuples, so the tuple form is
    off the documented path; but a `subset(nodes=(0, 1))` habit fails as a plausible-picture-of-wrong-data
    (empty), not a legible error. Low-ish severity because the docs don't invite it.
    Suggested change: one docstring clause on `nodes=` ("a tuple is treated as a single (tuple-valued) node
    ID, not a collection; pass a list or array for multiple IDs"), or reject a tuple `nodes=` with a
    legible error steering to a list.
  - [low-confidence] `hops` has no type/positivity guard: a negative `hops` on lazy edges skips the
    `hops > 0` gate (hiveplot.py:5472) and then `_expand_node_ids_by_hops` runs `range(hops)` zero times
    (hiveplot.py:5346), so `hops=-1` silently behaves like `hops=0` rather than erroring. Not a promised
    contract and not walked, but a negative value reads as a user mistake that passes silently.
    Suggested change: optional — validate `hops >= 0` with a legible ValueError, or note in the docstring
    that hops is clamped at 0.
Test-method-coverage audit: clean. `subset` is the only public method this workstream added; every
  sampled test_subset_* (independent-child, freezes-placements, freezes-ranges, induced-closure,
  scalar-ego, hops-one-neighbor-neighbor, union-within-call, chained-intersect, axis-pair-both-directions,
  empty-renderable, before-build-raises, does-not-call-curve-construction, python-list-is-ids,
  non-range-index id + mask, multi-tag-dict-absent-tag, lazy-hops-zero/mask-raises/hops>0-raises) calls
  `subset` in its body and asserts the named scenario. The four `data_subset`/`data_highlight` TODOs are
  deleted (repo-wide grep clean); the CHANGELOG entry landed and is accurate (CHANGELOG.rst:21).
```

**Workstream B — highlight registry + matplotlib overlay (post-impl, 2026-07-07).**

```
Status: propose
API surface reviewed: HivePlot.add_highlight(nodes=, edges=, hops=, closure=, color=, key=,
  node_kwargs=, edge_kwargs=) -> key; HivePlot.update_highlight(key, nodes=, edges=, hops=, closure=,
  color=, node_kwargs=, edge_kwargs=); HivePlot.remove_highlight(key=) + bare clear-all;
  HivePlot.highlights (read-only property); Highlight dataclass record; HIGHLIGHT_ACCENT_PALETTE;
  _UNSET sentinel; HighlightKeyCollisionWarning; iter_highlight_overlays (viz/base.py);
  _draw_highlight_overlays (viz/matplotlib.py); __init__ re-exports (Highlight, HIGHLIGHT_ACCENT_PALETTE).
Verdict: the shipped highlight API MATCHES the plan's promises and matches my planning verdicts. The verb
  family is exactly verdict 3 (add_highlight -> key / update_highlight(key, ...) / remove_highlight(key) /
  highlights property; no bare highlight()); update_highlight takes the FULL add_highlight kwarg set
  including selection, not just style (journey item 10, the GIF re-aim journey), verified by
  test_update_highlight_replaces_only_named_fields re-aiming nodes= + hops=; the highlights property
  returns a named Highlight record, not a raw dict-of-dicts (journey item 11); node_kwargs=/edge_kwargs=
  passthrough shipped on both add and update (journey item 15), tested end-to-end through the matplotlib
  overlay; bare remove_highlight() clears all (journey item 15's clear-all affordance); per-highlight
  color= override, closure="incident" default with "induced", and hops all present. Docstrings lead with
  the user's word ("Highlight a selected group...", "Revise a registered highlight...", "Remove a
  registered highlight..."), the accent palette is Okabe-Ito colorblind-safe assigned first-unused, and
  render-time semantics + double-plotting are pinned in prose. I walked every task in the brief (two named
  highlights different colors, per-frame update re-aim, remove-by-key + clear-all, inspect the record,
  combined nodes= + edges=, incident-vs-induced, hops, color override, kwargs passthrough) and each runs
  as written against the shipped signature. Concerns below are all doc/ergonomics polish; none block.
Concerns:
  - [worth-discussing] `key=` sits at position 6 in `add_highlight`'s signature (after nodes, edges, hops,
    closure, color), so `hp.add_highlight([1,2,3], "clump")` binds "clump" to `edges=`, not `key=` — at
    hiveplot.py:5874-5884. The plan's own Example 3 and every test always pass `key=` by name, so the
    documented path is safe, but `add_axes` / `add_edges` and the whole house `add_*` family put the
    naming/identity argument early, and a user reaching for `add_highlight(sel, "clump")` by muscle memory
    binds a string to `edges=` and then hits a downstream selection error (or a silent empty overlay if the
    string happens to name nothing). The record's own field order (key first) and `update_highlight`'s
    signature (key first, positionally required) both put key first, so the add-signature is the odd one
    out within this very feature.
    Suggested change: make `key` keyword-only (a `*,` before it) so the positional trap is a legible
    TypeError rather than a mis-bind; or, if positional key is wanted, hoist it earlier. Keyword-only is the
    cheaper, non-breaking-later call and matches how every call site already passes it.
  - [worth-discussing] `highlights` and the `highlights.values()` records expose the RAW selection value the
    user passed (the mask/expression/ID list), never the resolved node/edge set the overlay actually draws —
    at hiveplot.py:5860-5872 (property) and the Highlight record (hiveplot.py:129-155). Inspecting
    `hp.highlights[key].nodes` after `add_highlight(nodes=nw.col("high") > 25)` returns the narwhals
    expression object, not "which nodes lit up." For the register/inspect/restyle journey this is defensible
    (the record mirrors add_highlight's inputs, faithfully documented), but a user inspecting to answer "what
    did this highlight actually select?" gets the predicate back, not the answer. No promised contract says
    it should resolve, so this is a note, not a miss.
    Suggested change: one docstring line on the property (and/or Highlight.nodes/edges) stating the record
    echoes the selection you PASSED, not the resolved set; point at re-running `subset(...)` with the same
    selection to materialize the actual nodes/edges if that's what the user wants.
  - [low-confidence] `update_highlight` re-validates `hops` negativity but NOT `closure`, and neither add
    nor update validates a bad `closure` value up front — at hiveplot.py:5957-6017 / 5874-5955. A typo like
    `closure="induce"` (or `"incidental"`) is stored verbatim and only surfaces downstream in
    `_restrict_to_node_ids` (the `== "induced"` check silently falls through to the incident branch,
    hiveplot.py:5655), so the user gets an incident overlay when they asked to narrow, with no signal. The
    `Literal["incident", "induced"]` annotation documents intent but isn't enforced at runtime. Low-confidence
    because it's off the documented path and a static checker flags it, but a runtime typo mis-narrows silently.
    Suggested change: optional — validate `closure in {"incident", "induced"}` in add_highlight (and update
    when not _UNSET) with a legible ValueError naming the two valid values.
  - [low-confidence] The `_UNSET`-vs-`None` distinction lives only in `update_highlight`/`remove_highlight`
    and is documented clearly there, but `add_highlight` uses plain `None` defaults for the same fields, so
    the two sibling methods assign different meaning to `nodes=None`: on add it means "no node seeds," on
    update it means "clear the node part." Both are individually correct and documented, but a user who
    learns `update_highlight(key, nodes=None)` clears a field may expect `add_highlight(nodes=None)` to do
    something symmetric (it's simply the default). Not a bug; the asymmetry is inherent to update's
    replace-only-named-fields contract.
    Suggested change: none required — the update docstring already spells out the clear-with-None behavior.
    Noting only so the deferred-doc-sweep namer can decide whether one cross-reference sentence
    ("unlike add_highlight, an explicit None here clears") earns its place.
Test-method-coverage audit: clean. Every public method this workstream added is called in the body of its
  named tests: add_highlight (test_add_highlight_returns_auto_generated_key / _returns_user_supplied_key /
  _auto_keys_increment / _key_collision_warns_and_replaces / _negative_hops_raises / _default_palette_
  assigned_first_unused / _palette_skips_used_color / _palette_wraps_when_exhausted / _color_override_
  respected, plus the overlay-draw and passthrough tests); update_highlight (_replaces_only_named_fields /
  _none_clears_optional_field / _missing_key_raises / _negative_hops_raises); remove_highlight (_by_key /
  _missing_key_raises / _no_argument_clears_all); the highlights property (_returns_highlight_records /
  _is_a_copy). Overlay/interplay surface also covered: combined nodes+edges one group, incident/induced,
  hops-via-shared-resolver, zorder-above-base, byte-for-byte-unchanged-without-highlights, accent color,
  no-conflict-warning, node_kwargs/edge_kwargs passthrough, multiple highlights, empty overlay renders,
  single-node induced renders, survives copy, subset child inherits none, BaseHivePlot/P2CP spared,
  non-RangeIndex ID + expression selection, lazy-edges hops=0 stays lazy. No gaps for methods this
  workstream touched.
```

**Workstream C — overlay propagation to bokeh / holoviews / plotly (post-impl, 2026-07-07).**

```
Status: propose
API surface reviewed: NO new public signature. WS C is a pure backend propagation: hive_plot_viz on
  viz/bokeh.py, viz/plotly.py, viz/holoviews.py now render registered highlights above the base layer via
  new module-private _draw_highlight_overlays / _draw_highlight_overlay / _draw_overlay_nodes trios
  (correctly underscore-prefixed) and the shared viz/base.iter_overlay_node_placements; viz/matplotlib.py
  refactored onto the same shared prep. The user-facing highlight API (add_highlight / update_highlight /
  remove_highlight / highlights) is unchanged from WS B. Cross-backend consistency of the highlight-styling
  experience is the one genuine user-facing angle.
Verdict: no new surface, and the cross-backend experience is consistent-by-construction with one doc gap.
  The per-highlight override wins on EVERY backend (highlight.color / highlight.edge_kwargs /
  highlight.node_kwargs merge LAST over the raised spotlight defaults in every _draw_highlight_overlay:
  bokeh.py:875-894, plotly.py:1062-1082, holoviews.py:1015-1038, matplotlib.py:709-730), the raised
  spotlight defaults carry across identically (edge width 2.5, edge alpha/opacity 0.9, node size bumped
  1.75-2.0x above each base), and the incident-lights-only-selected-nodes contract is single-sourced through
  iter_overlay_node_placements so no backend can re-introduce the WS B neighbor-dot bug. `color=` is honored
  consistently (common named colors like "goldenrod" are valid across all four backends). The one real
  cross-backend friction is a doc bias, not a code defect: node_kwargs / edge_kwargs are backend-NATIVE (the
  library's established convention, cf. rename_edge_kwargs at hiveplot.py:5076), but the single place a user
  learns highlight styling teaches matplotlib-only kwarg names as if universal (concern 1). No silent
  ignore-vs-error inconsistency was introduced beyond what the docstring bias invites, and the P2CP guard
  holds on every backend. Confirmed no new user-facing signature slipped in.
Concerns:
  - [worth-discussing] The `add_highlight` docstring hardcodes matplotlib-only kwarg-name examples that WS C
    now makes wrong on three of the four backends it renders on — at hiveplot.py:5949-5953 (node_kwargs shows
    ``{"s": 60}``, edge_kwargs shows ``{"linewidth": 2}``) and hiveplot.py:5943 (color called "Any
    matplotlib-recognized color"). WS C makes these kwargs live on bokeh / plotly / holoviews, where the
    names diverge: bokeh & plotly use ``size`` / ``line_width``, holoviews-bokeh uses ``size`` / ``line_width``
    while holoviews-matplotlib uses ``s`` / ``linewidth``. The shipped tests themselves encode this divergence
    (bokeh/plotly pass ``node_kwargs={"size": 20}`` / ``edge_kwargs={"line_width": 7}`` at
    viz_bokeh_test.py:1100/1124 and viz_plotly_test.py:1147/1165; the holoviews test branches the name on
    sub-backend at viz_holoviews_test.py:1496/1526). A user who learns highlight styling from the docstring
    and copies ``node_kwargs={"s": 60}`` onto a bokeh or plotly plot hits a strict-validation raise (bokeh
    scatter / plotly Marker reject the unknown ``s`` key), and ``{"linewidth": 2}`` likewise fails on
    bokeh/plotly. This is a plausible-first-use surprise created BY this workstream (the kwargs were
    matplotlib-only before C; C is what makes them render, and mis-render, elsewhere). The house convention
    that edge/node kwargs are backend-native (rename_edge_kwargs, hiveplot.py:5076) softens this from a bug to
    a doc gap: the code correctly passes through whatever the user gives, and a user already steeped in the
    library knows to translate. But the highlight docstring is the newest user's first and only stop.
    Suggested change: one clause on node_kwargs / edge_kwargs stating the keys are the active backend's native
    kwarg names (``s`` / ``linewidth`` on matplotlib, ``size`` / ``line_width`` on bokeh & plotly, matching the
    active holoviews sub-backend), and drop or dual-spell the baked-in ``{"s": 60}`` / ``{"linewidth": 2}`` so
    the example is not silently matplotlib-only; optionally soften "matplotlib-recognized color" to "any color
    the active backend recognizes."
  - [low-confidence] The base node_viz supports node-data-column-name-as-kwarg-value (per-node data-driven
    styling: a scatter kwarg whose value names a nodes.data column becomes a per-node array) on plotly
    (plotly.py:486-489) and matplotlib, but the overlay _draw_overlay_nodes scatters raw placement arrays
    directly and does NOT run that column-name resolution — at plotly.py:1106-1124 / matplotlib.py:755-764 /
    bokeh.py:918-926. So node_kwargs={"size": "some_column"} styles per-node in a base node_viz call but is
    passed as a literal (and would error) in a highlight overlay. Off the walked path (the highlight story is
    one accent color + a size/width bump, not per-node data mapping) and no promised contract says overlays
    honor data-column kwargs, so this is a note, not a miss.
    Suggested change: none required; note only so a future "style the overlay by a node variable" ask knows
    the overlay path deliberately skips column-name resolution.
Test-method-coverage audit: clean. WS C adds no new public method; the touched public entry points are
  hive_plot_viz on the three vector backends. Every sampled test_hive_plot_viz_* highlight test calls
  hive_plot_viz in its body and asserts the overlay outcome (draws-overlay asserts accent color on collected
  overlay glyphs; incident-only-lights-selected-nodes asserts the SELECTED set and bounds the dot count;
  edge_kwargs-override / node_kwargs-passthrough assert the per-highlight override wins; without-highlights
  asserts base-unchanged glyph count; no-conflict-warning runs under warnings-as-errors), and
  test_p2cp_viz_unaffected_by_highlight_machinery exercises the P2CP guard, on each of bokeh / plotly /
  holoviews (holoviews parametrized over both sub-backends). No gaps for the surface this workstream touched.
```

**Workstream D — condition shorthand (narwhals expressions) (post-impl, 2026-07-07).**

```
Status: propose
API surface reviewed: narwhals-expression selection on HivePlot.subset(nodes=, edges=) and
  HivePlot.add_highlight(nodes=, edges=) (incl. per-tag edges={tag: expression} dict), plus the net-new
  lazy-Dask expression composition path (module-privates _resolve_node_seed_ids, _edge_frame_mask,
  _resolve_edge_seed_endpoint_ids, _lazy_tag_placed_axis_node_ids, _require_full_length_row_mask,
  _raise_lazy_mask_selection_error, all correctly underscore-prefixed); the subset / add_highlight docstrings
  (now with side-by-side mask+expression forms); the folded-in WS-C backend-native node_kwargs/edge_kwargs
  doc fix on add_highlight. No new public signature (expression-in-parameter reuses the existing nodes=/edges=).
Verdict: the shipped expression API is ergonomic and honestly documented, with ONE must-fix doc bug in the
  folded-in WS-C fix (a wrong plotly kwarg name that the shipped tests themselves contradict). `nodes=nw.col(
  "degree") > 5` reads naturally as "nodes where"; verdict 6's promise is delivered both ways: every
  entry-point docstring shows the pandas-native boolean-mask form beside the narwhals expression form (subset
  lines 5559-5563, add_highlight lines 5990-5997 with the explicit "selects the same nodes whether you write
  it as a mask or the equivalent expression; the mask form never requires learning narwhals" framing), so a
  pandas-only user is never forced into narwhals. Filtering on a non-partition / non-sorting column works and
  is tested (test_subset_node_expression_on_non_partition_non_sorting_variable, hiveplot_test.py:8384). Cross-
  library dispatch is real and tested end-to-end (pandas / polars / cuDF twins asserting identical selection;
  Dask-lazy composition). The aggregating-expression error is legible and steers to the fix verbatim
  (_require_full_length_row_mask, hiveplot.py:663-670: "must evaluate to one boolean value per row ... Use a
  per-row, non-aggregating expression (e.g. nw.col('col') > value, not nw.col('col').sum() > value)"), names
  the offending tag on the dict path (_edge_frame_mask, 5334-5337), and is tested on node / edge / multi-tag-
  dict (hiveplot_test.py:9107-9158). The lazy-vs-in-memory distinction (an expression composes lazily on a
  Dask frame, a concrete mask errors) is documented, not silent: the subset `.. note::` (5597-5605) and the
  in-memory error text (_raise_lazy_mask_selection_error, 5523-5530) both name the expression-stays-lazy /
  concrete-mask-errors seam explicitly, and it's tested three ways
  (test_subset_lazy_edges_expression_stays_lazy_and_filters / _dict_stays_lazy / _concrete_mask_still_raises).
  The must-fix and two low-confidence notes below.
Concerns:
  - [must-fix] The folded-in WS-C backend-native kwarg fix got the plotly EDGE-width name wrong: the
    add_highlight docstring tells users to pass plotly ``edge_kwargs={"width": 2}`` — at hiveplot.py:6038-6039.
    But overlay edge_kwargs are merged into and passed to the library's own ``edge_viz`` (plotly.py:1063-1076,
    exactly as the docstring's own phrase "passed straight through to the active backend's edge viz call"
    says), and plotly's ``edge_viz`` spells its width parameter ``line_width`` (plotly.py:567), translating it
    to plotly's raw ``width`` internally (plotly.py:762-764). So a user copying the documented plotly example
    passes ``{"width": 2}``, which does NOT match the ``line_width`` the overlay default sets, leaving the
    default 2.5 width in place (user intent silently lost) while ``width`` rides along as an extra kwarg. The
    shipped test itself contradicts the doc: it passes ``edge_kwargs={"line_width": 7}`` and asserts
    ``trace.line.width == 7`` (viz_plotly_test.py:1147/1151). The bokeh example ``{"line_width": 2}`` is
    correct for the same reason (bokeh's ``edge_viz`` also uses ``line_width``, bokeh.py:877), so plotly is the
    lone wrong spelling; the node_kwargs line (matplotlib ``s`` / bokeh-plotly ``size``) is correct (overlay
    nodes scatter directly with ``size``, plotly.py:1109 / bokeh.py:921). This is a plausible-first-use
    surprise created BY the fix that was meant to make these examples backend-accurate. Upgraded from doc-
    polish to must-fix because it is a self-contradicting shipped example (the test proves the right name) and
    the whole point of the fold-in was correctness of these kwarg names.
    Suggested change: change the plotly example on the edge_kwargs line to ``{"line_width": 2}`` (matching
    bokeh, since both flow through ``edge_viz``).
  - [low-confidence] The ``color`` line on add_highlight still reads "Any ``matplotlib``-recognized color
    overrides the default" — at hiveplot.py:6026 — now that the overlay renders on bokeh / plotly / holoviews
    too. WS C's api-critic concern flagged this as the optional half of the same fix; the fold-in took the
    kwarg-name half and left the color half. In practice common named colors ("goldenrod", "red") are valid
    across all four backends (WS C verdict cleared this), so the imprecision rarely bites, but the sentence is
    now backend-parochial in the same way the kwargs were. Low-confidence because it does not mis-render for
    the palette or any common named color; the accent palette hexes work everywhere.
    Suggested change: soften to "any color the active backend recognizes" while touching this docstring for the
    must-fix above; batch, don't spawn a separate dispatch.
  - [low-confidence] An expression-on-a-lazy-edge-tag broadens the seed set (it seeds every node placed on the
    axes the tag spans, not the precise incident endpoints), while the SAME expression on an in-memory tag
    seeds only the incident nodes — at _lazy_tag_placed_axis_node_ids (hiveplot.py:5417-5440) vs the in-memory
    branch (5399-5414). This is documented (the subset `.. note::` "One asymmetry on lazy edges ..." at
    5602-5605) and the composed lazy predicate still does the real row filtering, so the drawn result is
    correct either way; it can only differ under ``closure="incident"`` on a highlight (a lazy incident overlay
    can light more far-endpoint context than the in-memory path would). Off the walked expression-filter story
    and honestly disclosed, so a note, not a miss.
    Suggested change: none required; the note is disclosed. Flag only so a future "why does my lazy incident
    highlight light more than the in-memory one?" question has a pinned answer.
Test-method-coverage audit: clean. The methods this workstream touched are the public subset / add_highlight
  (no new signature; expression is a new accepted value on existing nodes=/edges=). Every sampled expression
  test calls the named method with an ``nw.col(...)`` value in its body and asserts the selection: subset —
  _node_expression_selection / _edge_expression_selection / _edge_expression_matches_mask / _on_non_partition_
  non_sorting_variable / _non_range_index_is_positional / _polars (node+edge) / _lazy_edges_expression_stays_
  lazy_and_filters / _dict_stays_lazy / _concrete_mask_still_raises / _missing_column_raises / _node+edge+
  multi_tag_dict_aggregating_expression_raises; highlight — _expression_selection / _edge_expression_selection
  / _node_expression_non_partition_non_sorting_variable / _non_range_index_is_positional / _polars / _overlay_
  lazy_edges_expression_stays_lazy; cuDF twins (cudf_test.py:290-339) assert cuDF-matches-pandas for subset
  node/edge and highlight node expressions. No gaps for the surface this workstream touched.
```

**Workstream D re-check — Amendment 18 lazy edge-expression behavior change (post-impl, 2026-07-07).**

```
Status: clean
API surface reviewed: the Amendment-18 re-implementation of lazy (Dask) edge-expression selection on
  HivePlot.subset(edges=) / add_highlight(edges=) (no new signature). Prior lazy path composed with zero
  compute but silently returned a NARROWER subgraph than the in-memory path; the re-implementation runs one
  bounded Dask compute (frames.lazy_edge_expression_endpoint_ids, frames.py:222-256) over just the matching
  edges' two endpoint-ID columns so the lazy result is byte-for-byte the in-memory subgraph. Reviewed: the
  subset `.. note::` (hiveplot.py:5578-5588), the lazy branch of _resolve_edge_seed_endpoint_ids
  (hiveplot.py:5392-5405), the _resolve_edge_seed_endpoint_ids docstring (5357-5361), and the
  _resolve_selection_node_ids docstring (5657-5659).
Verdict: the behavior change is ergonomically acceptable AND honestly documented; the "lazy = zero compute"
  surprise is adequately surfaced. Walking the target case `subset(edges=nw.col("weight") > 5)` on a Dask edge
  frame: a user who held the "lazy = fully deferred" mental model now triggers a real compute, and the note
  tells them WHEN (an edge narwhals-expression on lazy edges specifically, split cleanly from ID/axis-pair
  selection which "compose straight into the stored lazy membership predicate with no compute") and WHY (the
  bounded pass gathers "just the unique endpoint node IDs of the matching edges (never the full edge frame)"
  so "the result is byte-for-byte the subgraph the in-memory path produces"). The WHEN is discoverable, not
  buried: the note is on the `subset` docstring, the same surface the user called, and it names the exact
  trigger (expression + lazy edges) rather than a vague "some selections compute." The WHY earns the cost from
  the user's standpoint: the prior lazy path silently under-selected (returned a narrower subgraph than the
  identical in-memory call), which is a correctness trap the user could not see; a bounded single-pass compute
  bounded by the same node-count ceiling that already governs node data (frames.py:234-236) is a fair price for
  making lazy == in-memory. A user who genuinely wants zero compute still has the fully-lazy ID and axis-pair
  paths, and that fork is spelled out in the same note. No signature change, no new surface, nothing surprising
  that the docstring does not name first.
Concerns:
  - [low-confidence] The note frames the trigger as a *cost* ("at the cost of that single pass over the
    expression's columns") but never states the FLIP SIDE a "lazy = deferred" user is really asking: that this
    compute is EAGER (it happens inside the `subset` / `add_highlight` call, not deferred to a later
    `.compute()`/render). A user timing their `subset` call and expecting it to return instantly may be
    surprised by a synchronous Dask compute mid-call. The mechanism ("runs one bounded Dask compute") implies
    eagerness to a Dask-fluent reader, but the note never says "this compute happens now, in this call."
    Low-confidence because the note is honest about the compute existing and the audience for a lazy edge frame
    is Dask-fluent enough to read "bounded Dask compute" as synchronous; flagging only so a future "why did my
    subset call block?" question has a pinned answer.
    Suggested change: none required. Optionally, one half-clause on the note ("... runs one bounded Dask compute
    *in the call*") if this docstring is touched for another reason; do not spawn a dispatch for it.
Test-method-coverage audit: clean (scope unchanged from the WS D block above; Amendment 18 re-implements the
  same lazy-expression path the WS D tests already exercise). The three lazy-expression tests
  (test_subset_lazy_edges_expression_stays_lazy_and_filters / _dict_stays_lazy / _concrete_mask_still_raises)
  are the binding contract; the correctness win (lazy == in-memory) is the assertion those _stays_lazy tests
  make on the filtered result. No new public method, so no new test-name contract to audit.
```

**Workstream E — HivePlotMatrix broadcast (post-impl, 2026-07-07).**

```
Status: propose
API surface reviewed: HivePlotMatrix.subset(nodes=, edges=, hops=) and the highlight family broadcast onto the
  matrix — HivePlotMatrix.add_highlight(nodes=, edges=, hops=, closure=, color=, *, key=, node_kwargs=,
  edge_kwargs=), update_highlight(key, ...), remove_highlight(key=), and the highlights property (collapsed
  {key: representative record} view). Plus the matrix-level color/key resolvers (_next_highlight_color,
  _find_unique_highlight_key) and the once-at-matrix-level collision-warning suppression. Mechanical propagation
  to a sibling class; walked from scratch as a first HivePlotMatrix encounter.
Verdict: a faithful, ergonomic mirror on the homogeneous-axis path (from_variable_sweep, from_tags), with ONE
  real gap on the heterogeneous-axis path (from_partition). Method names, parameter orders, defaults, and return
  shapes all transfer from HivePlot muscle memory with no drift: subset returns a new, independent frozen-geometry
  HivePlotMatrix (deepcopy + per-panel HivePlot.subset, hpm.py:1070-1073) exactly as HivePlot.subset returns a
  new frozen HivePlot; add_highlight returns one shared key (honest for a broadcast; the parent returns one key
  too); color and key are resolved ONCE at the matrix level (hpm.py:1151/1166) so a broadcast highlight reads as
  one story across panels rather than drifting color or key per panel (the right call, and tested by
  test_add_highlight_shares_one_color_across_panels / _second_call_distinct_color). The key-collision warning
  fires exactly once at the matrix level with the per-panel warnings suppressed (hpm.py:1152-1168), which is the
  correct broadcast behavior and is tested (test_add_highlight_key_collision_warns_once). update_highlight /
  remove_highlight guard on "registered on no panel" and raise a KeyError listing the registered keys
  (hpm.py:1215-1221 / 1251-1257), matching the parent's KeyError shape. P2CP is guarded against leakage
  (test_p2cp_has_no_subset_or_highlight_surface). The collapsed highlights view and the missing axis-pair
  homogeneity caveat are the two concerns; the second is the must-fix.
Concerns:
  - [must-fix] Neither subset nor add_highlight documents that an axis-pair edge selection assumes homogeneous
    panel axes, and on a from_partition matrix an axis-pair broadcast crashes mid-loop with a per-panel error
    carrying no matrix context — at HivePlotMatrix.subset (hpm.py:1031-1073) and add_highlight (hpm.py:1096-1180),
    neither of which carries a :raises: block. from_partition panels are heterogeneously axed by construction:
    diagonal cells carry [group_i, group_i_repeat], off-diagonal cells carry [group_i, group_j,
    collapsed_group_axis_name] (hpm.py:1614-1664). So `hpm.subset(edges=("A", "B"))` on a from_partition matrix
    hits HivePlot.subset -> _resolve_selection_node_ids on a panel whose axes are not "A"/"B" and raises
    InvalidAxisNameError with the message "Axis 'A' in edge selection is not one of this hive plot's axes
    ([...])" (hiveplot.py:5461-5465). The message names ONE panel's axis list with no row/col, so the user cannot
    tell which of N panels crashed or why the same call worked on a from_variable_sweep matrix. The parent
    HivePlot.subset documents :raises InvalidAxisNameError: (hiveplot.py:5617); the mirror dropped the :raises:
    block entirely, so a user reading the tooltip has no signal this can throw. The failure is not a partial-state
    hazard (the crash is inside the deepcopy'd new_matrix loop before return, so this matrix is untouched and the
    child is discarded), but it IS a legibility cliff on the plan's headline cross-panel-comparison surface, and
    from_partition is the plan's own "expected strongest" matrix type (see the adversary's WS E worth-discussing
    at plan line 693). node-ID / mask / expression selections broadcast fine across heterogeneous panels; only
    axis-pair selection assumes homogeneity, and the class does not say so anywhere.
    Suggested change: add a :raises InvalidAxisNameError: line to both subset and add_highlight docstrings, plus
    one caveat sentence naming the assumption ("an axis-pair edge selection assumes every populated panel carries
    both named axes, as from_variable_sweep / from_tags matrices do; a from_partition matrix is heterogeneously
    axed, so scope an axis-pair selection to a homogeneous matrix or select by node ID / mask / expression
    instead"). Optionally, catch InvalidAxisNameError in the broadcast loop and re-raise naming the (row, col) of
    the offending panel so the crash is self-locating; docstring caveat is the floor, the (row, col) context is
    the nicer fix.
  - [worth-discussing] The collapsed highlights view honestly represents one-highlight-across-all-panels ONLY
    under the broadcast API; it silently hides per-panel divergence a user can create through the exposed
    __getitem__ escape hatch — at HivePlotMatrix.highlights (hpm.py:1090-1094), which builds the view with
    setdefault (first populated panel wins) and returns one representative record per key. The docstring asserts
    the record is "identical across panels because add_highlight broadcasts one selection and one color to all of
    them" (hpm.py:1082-1084), which is true for the matrix-level add/update/remove path. But HivePlotMatrix
    exposes each live HivePlot via __getitem__ (hpm.py:903-911), so a user can call
    hpm[0, 0].update_highlight(key, color="red") on one panel, and hpm.highlights will then report whichever
    panel iter_populated_cells yields first, masking the divergence with no signal. This is the api-critic's
    canonical "could mislead a user into thinking a highlight is uniform when a panel diverged" case: the parent's
    hp.highlights is per-plot and cannot mislead this way; the matrix's collapse is the drift. Worth-discussing
    rather than must-fix because the documented, blessed path (matrix-level add/update/remove) keeps panels in
    lockstep, and reaching into a panel directly is off the intended surface; but the docstring's "identical
    across panels" reads as a guarantee, not a "when you only use the broadcast methods" conditional.
    Suggested change: soften the highlights docstring from "identical across panels because add_highlight
    broadcasts" to "identical across panels as long as you register and revise highlights through the matrix-level
    add_highlight / update_highlight / remove_highlight; reaching into a single panel via hpm[r, c] and editing
    its highlight directly can diverge a panel, and this collapsed view shows only the first panel's record."
  - [low-confidence] subset's docstring says "Mutating the returned matrix never touches this one (subset always
    copies)" (hpm.py:1048) but does not carry the parent's SubsetBeforeBuildError :raises: contract — at
    hpm.py:1031-1073. A from_* matrix always ships built geometry, so a pre-build subset is unreachable through
    the classmethods, but a generic HivePlotMatrix constructed from hand-built unbuilt HivePlots would surface
    HivePlot.subset's SubsetBeforeBuildError per panel, again with no matrix context. Same shape as the must-fix
    (dropped :raises: block on the mirror), lower confidence because the from_* construction path always builds,
    so a user reaching this has already stepped off the guided path.
    Suggested change: fold a :raises SubsetBeforeBuildError: line into the same subset :raises: block added for
    the must-fix; no separate dispatch.
Test-method-coverage audit: clean. Every public method this workstream added is called in the body of its named
  test: subset — test_subset_returns_new_matrix / _broadcasts_same_selection_to_every_panel /
  _preserves_grid_shape_and_empty_cells / _freezes_per_panel_geometry / _panels_slice_parent_placements /
  _edge_selection_broadcasts / _hops_broadcasts / _render (TestSubsetBroadcast, all call
  partition_matrix.subset or sweep_matrix.subset); add_highlight / update_highlight / remove_highlight / highlights
  — the full TestHighlightBroadcast battery calls each on the matrix and asserts the broadcast reached every panel
  via iter_populated_cells. The heterogeneous-axis crash the must-fix names is NOT covered by any test (the
  edge-selection tests all use sweep_matrix, whose panels are deliberately homogeneously axed per the fixture
  docstring at hiveplot_matrix_test.py:4850-4851); a test asserting the from_partition axis-pair behavior (raise
  today, or a matrix-context error after the fix) would pin the must-fix's resolution. Flagging the gap here, not
  as a separate coverage miss, since it is the same finding.
```

**Workstream F — datashader highlight guard (raise-and-point) (post-impl, 2026-07-07).**

```
Status: propose
API surface reviewed: HighlightsUnsupportedByDatashaderError (exceptions/hive_plot.py:117, re-exported via
  exceptions/__init__.py star import, no __all__, reachable as hiveplotlib.exceptions.HighlightsUnsupportedByDatashaderError
  exactly like its siblings); _guard_no_highlights (viz/datashader.py:89, module-private, correctly underscored); the
  guard call + `:raises:` line on all three public entry points datashade_edges_mpl / datashade_nodes_mpl /
  datashade_hive_plot_mpl (and their aliases edge_viz / node_viz / hive_plot_viz at datashader.py:1613-1616, which
  point at the guarded functions).
Verdict: the guard's user-facing surface is CLEAR and HONEST, and matches the plan's WS F done-when. Raising (not
  warning) is the right call: the highlight would silently vanish under a warning, and a data-loss-shaped drop is
  exactly the case that earns a hard stop. The message names both real remedies (subset(...) for a datashaded
  drill-down, a vector back end for the overlay), the "same selection vocabulary as a highlight, including hops"
  clause is factually correct (subset carries hops at hiveplot.py:5536 and add_highlight documents "the same
  vocabulary as subset" at hiveplot.py:5956), the exception name is accurate and discoverable, the guard fires once
  at the top of datashade_hive_plot_mpl (datashader.py:1547) BEFORE it delegates to the edges/nodes entry points, so
  no double-raise and one message for all three paths, and the `:raises:` block documents it on each. Two
  worth-discussing message-precision gaps and one low-confidence note below; none block, none touch code paths.
Concerns:
  - [worth-discussing] The message and all three `:raises:` lines name only "matplotlib, bokeh, or plotly" as the
    back ends that keep the overlay, but holoviews ALSO fully draws the highlight overlay — at viz/datashader.py:109
    (guard message) and datashader.py:984/1269/1543 (`:raises:` lines), vs. viz/holoviews.py:971-1059
    (_draw_highlight_overlays composes every registered highlight on the holoviews overlay). A holoviews user who
    registered a highlight and reached the datashader guard is told to switch to one of three back ends that
    excludes the one they may already be using; the honest set is four (matplotlib / bokeh / plotly / holoviews).
    The understatement never sends anyone wrong, but it undersells a supported path.
    Suggested change: add holoviews to the message's "(matplotlib, bokeh, or plotly)" list and to the three
    `:raises:` lines, making it "(matplotlib, bokeh, plotly, or holoviews)".
  - [worth-discussing] The message steers a highlighted instance to subset(...) but never mentions CLEARING the
    highlights for the genuine "I registered a highlight for the mpl view, now I just want a datashaded BASE view"
    case — at viz/datashader.py:104-112. That user is blocked, and neither remedy fits: subset(...) narrows the
    plot (not what they want), and switching back ends abandons datashader (not what they want). The one-liner that
    actually unblocks them is hp.remove_highlight() (bare clear-all, shipped in WS B at hiveplot.py:remove_highlight),
    which the message omits. A user who doesn't already know remove_highlight exists has no pointer to it from this
    error. Worth-discussing rather than must-fix because the raise is honest and the base-view intent is a narrower
    slice of users than the drill-down intent the message is (correctly) optimized for.
    Suggested change: add a third clause to the message, e.g. "or clear the highlights with
    `HivePlot.remove_highlight()` to datashade the base view alone."
  - [low-confidence] The message and the subset docstring both say subset carries "the same selection vocabulary,
    including hops," which is true for the SEED vocabulary (nodes / edges / hops), but subset defaults to
    closure="induced" (hiveplot.py:5582-5584) while add_highlight defaults to closure="incident"
    (hiveplot.py:5946). A user redirected from a highlight to subset(...) who does not pass closure= gets a
    different edge set (induced keeps only within-set edges; the highlight's incident kept every touching edge).
    The message says "selection vocabulary," not "identical result," so it is honest at the vocabulary level, and
    subset's own docstring pins the closure difference; low-confidence because the message doesn't claim result
    equivalence and the drill-down user usually wants the tighter induced view anyway.
    Suggested change: optional — none needed if "selection vocabulary" is read strictly; if the maintainer wants to
    preempt the closure surprise, a half-clause ("subset defaults to induced closure") could ride the hops note, but
    it risks over-loading an already-dense message.
Test-method-coverage audit: clean. The one guard added (_guard_no_highlights) is exercised through all three public
  entry points in test_datashade_with_registered_highlights_raises (datashader_test.py:1544, parametrized over
  ["datashade_edges_mpl", "datashade_nodes_mpl", "datashade_hive_plot_mpl"], each getattr'd and CALLED in the body,
  asserting HighlightsUnsupportedByDatashaderError with match="subset"); the no-highlights regression is pinned by
  test_datashade_without_highlights_renders_unchanged (datashader_test.py:1573, same parametrization), and the
  affirmative subset-then-datashade steer by test_datashade_subset_child_renders_with_no_new_code
  (datashader_test.py:1593) plus the Dask-lazy variant test_datashade_subset_child_over_lazy_dask_edges_renders
  (datashader_test.py:1612). Every named datashade_* function is invoked in the body of the test naming it.
```

**Workstream J — lazy `hops>0` support on Dask edge frames (post-impl, 2026-07-07).**

```
Status: clean
API surface reviewed: the behavior change on HivePlot.subset(nodes=, edges=, hops=) and
  HivePlot.add_highlight(..., hops=) when hops>0 runs against a lazy (Dask) edge frame (NO new signature).
  Prior (WS A floor) behavior: hops>0 on lazy edges raised the in-memory-required TypeError. Shipped now:
  it WORKS, computing the neighborhood via a bounded per-hop endpoint-ID compute. New module-privates
  _expand_node_ids_by_hops / _frontier_neighbor_ids (hiveplot.py:5488-5544, correctly underscore-prefixed)
  and frames.lazy_edge_frontier_endpoint_ids (frames.py:259-297); the deleted floor gates in subset /
  _highlight_overlay_plot; the updated subset .. note:: (hiveplot.py:5628-5640) and _raise_lazy_mask_selection_error
  message (hiveplot.py:5553-5562).
Verdict: the behavior change is HONEST and CONSISTENT with the WS D precedent. Same shape as Amendment 18's
  faithful-lazy-expression call (which I reviewed clean): a lazy operation that previously composed with zero
  compute (or here, raised) now triggers a bounded Dask compute so the lazy result equals the in-memory one,
  and the docstring names WHEN, WHY, and the cost so the compute is discoverable rather than surprising.
  Walking `subset(nodes=[..], hops=2)` on a Dask edge frame: a user who had this ERROR under the WS A floor now
  gets it to WORK, and the subset .. note:: tells them the mechanism ("hops>0 expands the node frontier one
  bounded compute per hop over the endpoint columns only ... so the edge frame is never materialized; the
  resulting subgraph matches the in-memory path") on the same surface they called. This reads as a strict
  improvement (an error became a correct result), not a hidden cost: the maintainer already chose faithful-lazy
  for the sibling expression call in Amendment 18, so faithful-lazy-hops is the consistent extension, and the
  cost (k hops = k bounded computes over the ID columns) is surfaced in the note. The three specific checks:
  (1) the compute is documented, in the same .. note:: and register as WS D's, with the same "bounded compute,
  result matches the in-memory path" framing; (2) removing the error is a strict improvement here, not a
  hidden cost, because the note makes the per-hop compute discoverable and a user wanting zero compute still
  has hops=0 plus the fully-lazy ID / axis-pair paths (spelled out in the same note); (3) NO leftover prose
  says lazy hops>0 "raises" — the concrete-mask error message was updated to say "`hops` expands over bounded
  endpoint-column scans" (hiveplot.py:5556), and the note explicitly lists lazy edge support "including
  ``hops>0``" (hiveplot.py:5630-5631). The WS A "no Dask node subset / node set fits in RAM" ceiling is still
  correctly stated and, importantly, correctly extended: hops GROWS the node set, and the note pins that the
  grown frontier "fits in RAM by the node-count ceiling" (hiveplot.py:5637), the frames helpers repeat it
  ("only the (bounded) set of distinct endpoint IDs is realized, which fits in RAM by the same node-count
  ceiling", frames.py:272-274), and _expand_node_ids_by_hops holds the growing frontier in an in-RAM Python
  set (hiveplot.py:5504-5510) — so the ceiling claim survives the node-set growth hops introduces. Two
  low-confidence notes below; neither blocks and neither is new to WS J (both mirror WS D notes).
Concerns:
  - [low-confidence] The subset .. note:: frames the per-hop compute as a per-call cost but, like the WS D
    lazy-expression note (my WS D-recheck low-confidence item, plan line 609-619), never states that the
    compute is EAGER and, additionally, that it is k computes for k hops (one bounded Dask compute PER hop),
    not one — at hiveplot.py:5636-5638. A Dask-fluent user reading "one bounded compute per hop" infers both,
    but a user timing a `subset(hops=3)` call and expecting instant return may be surprised by three
    synchronous computes mid-call. Same disposition as the WS D note: honest about the compute existing, and
    the lazy-edge audience is Dask-fluent enough to read "bounded compute per hop" as synchronous and additive.
    Suggested change: none required. Optionally, if this note is touched for another reason, "runs one bounded
    Dask compute *in the call* per hop" makes both the eagerness and the k-computes cost explicit; do not spawn
    a dispatch for it.
  - [low-confidence] The lazy `hops>0` compute is documented only on `subset`'s docstring; `add_highlight`
    routes the reader to subset ("See :py:meth:`subset` for the full selection vocabulary", hiveplot.py:6011/6013)
    and never carries the lazy note itself — at hiveplot.py:5981-6037. This matters slightly more for highlight
    than for subset because the highlight compute fires at VIZ-CALL time (inside _highlight_overlay_plot,
    hiveplot.py:6201-6203), not at add_highlight time: a user who registers `add_highlight(nodes=.., hops=2)` on
    a lazy plot and expects registration to be free is correct (registration only stores the record), but the
    bounded per-hop compute then fires on every `plot()` call. The cross-reference to subset carries the
    vocabulary but not this render-time-compute timing. Low-confidence because add_highlight already pins its
    render-time semantics ("Highlights render at viz-call time: this method only registers the highlight",
    hiveplot.py:5999-6001), which a careful reader composes with subset's lazy note to reach the right model;
    and this is the same "compute-timing lives on subset, highlight cross-references" structure WS D shipped
    (no new WS-J regression).
    Suggested change: none required. Optionally, if the add_highlight docstring is touched, one clause on the
    render-time paragraph ("on lazy edge data a hops>0 highlight runs its bounded per-hop compute at each plot
    call, not at registration") would pin the timing; batch, do not spawn.
Test-method-coverage audit: clean. No new PUBLIC method — the touched public surface is subset / add_highlight
  (behavior change on the existing hops= parameter). Both are exercised by the two new lazy-hops tests, each of
  which CALLS the named method in its body and asserts the in-memory-twin contract:
  test_subset_lazy_edges_hops_greater_than_zero_matches_in_memory_and_stays_lazy (hiveplot_test.py:9110) calls
  subset(nodes=seed, hops=2) on both a lazy and an in-memory build, asserts the materialized edge-ID sets and
  surviving-node sets are EQUAL, guards non-vacuity (asserts len>0, so an empty result is a regression, not a
  silent pass), and asserts every stored child edge record is still a LazyEdgeSubset (the edge frame never
  materialized); test_highlight_overlay_lazy_edges_hops_greater_than_zero_matches_in_memory (hiveplot_test.py:10199)
  makes the same in-memory-twin + non-vacuity + never-materialized assertions on the add_highlight -> overlay path
  with hops=2 induced. The two new module-privates (_expand_node_ids_by_hops / _frontier_neighbor_ids) and
  frames.lazy_edge_frontier_endpoint_ids carry no direct test, exercised through the public lazy-hops tests
  above — the SAME convention the reviewed-clean WS D helper (frames.lazy_edge_expression_endpoint_ids) follows
  (also no direct test, exercised through subset/add_highlight lazy-expression tests), so this is the plan's
  established pattern, not a WS-J gap. No gaps for the surface this workstream touched.
```

**Amendment 26 fix — lazy INCIDENT bounded-endpoint-compute disclosure (post-impl, 2026-07-07, light pass).**

```
Status: propose
API surface reviewed: no signature delta — subset(nodes=, edges=, hops=) and add_highlight(..., closure=) are
  unchanged. The one user-facing surface delta is the final sentence of subset's .. note:: (hiveplot.py:5641-5644,
  referenced by add_highlight via its :py:meth:`subset` cross-references), disclosing that a lazy INCIDENT
  selection now triggers one further bounded endpoint compute even at hops=0.
Verdict: the disclosure is CLEAR, ACCURATE, and CONSISTENT with its siblings. It names WHEN precisely (a lazy
  edge frame + closure="incident", which is add_highlight's default, even at hops=0), and it establishes the
  compute is BOUNDED ("one further bounded compute over the endpoint columns ... to gather the touching edges'
  far endpoints") on a surface where the whole note has already trained the reader that "bounded" means endpoint
  columns only, never the full edge frame. "further" correctly signals it stacks on any earlier selection cost.
  The sentence closes on "so a lazy incident overlay matches its in-memory twin" — the same "matches in-memory"
  reassurance the WS D edge-expression sentence ("byte-for-byte the subgraph the in-memory path produces") and
  the WS J hops sentence ("the resulting subgraph matches the in-memory path") both land on, so the three lazy
  disclosures now read as one coherent family in cadence and vocabulary. Placement is right: the note lives on
  subset (the vocabulary home) and this is the disclosure's single source of truth; the sentence explicitly
  bridges to add_highlight rather than duplicating. Amendment 26's own framing (a performance disclosure, not a
  semantics asymmetry, because lazy now MATCHES in-memory) is honest and matches what shipped: the correctness
  bug (a dropped far-endpoint placement) is fixed, so the reader is not silently getting a wrong picture; they
  pay one bounded compute they might not have anticipated. One worth-discussing discoverability item and one
  low-confidence polish below; neither blocks. If the maintainer reads "selection vocabulary" as the home for
  cost and holds the WS J structure, the note is clean as shipped.
Concerns:
  - [worth-discussing] The incident-default cost is discoverable only two hops from where an add_highlight user
    would look. add_highlight's `closure` param (hiveplot.py:6033-6034) says "incident (default: selected nodes
    plus every touching edge)" with NO mention of a lazy compute, and the `nodes=`/`edges=` params point to
    subset "for the full selection vocabulary" (hiveplot.py:6028/6030), not for the cost. So an add_highlight
    user on a Dask frame who chose the DEFAULT closure (i.e. did nothing) now pays a bounded compute that subset
    (locked induced) never pays, and the only place that says so is the last sentence of a linked method's note.
    This is the same "compute-timing lives on subset, highlight cross-references" structure the WS J review
    already logged low-confidence (plan lines 823-837), but Amendment 26 sharpens it: the WS J case was hops>0
    (a user opted into expansion); this fires at hops=0 on the closure DEFAULT, so the surprised user made no
    opt-in choice at all. Worth-discussing not must-fix because lazy now matches in-memory (no wrong data, only
    an unadvertised bounded cost) and add_highlight already pins its render-time semantics, which a careful
    reader composes with subset's note to reach the right model.
    Suggested change: batch with the WS J low-confidence add_highlight-timing suggestion (plan line 835-837).
    One clause on add_highlight's render-time paragraph would cover both: e.g. "on lazy (Dask) edge data the
    default incident closure runs one bounded per-hop endpoint compute at each plot call (even at hops=0), so
    pass closure=\"induced\" if you want the zero-far-endpoint path." Do not spawn a standalone dispatch.
  - [low-confidence] The new sentence writes "one further bounded compute" in plain text, while the two sibling
    disclosures in the same note italicize the load-bearing word (`one *bounded* Dask compute`, `hiveplot.py:5633`).
    A reader scanning the note for the "bounded" reassurance gets a visual cue on the first two and not the third.
    Cosmetic; the word "bounded" is present and the meaning is intact.
    Suggested change: none required. If the note is touched for the item above, italicize (`one further *bounded*
    endpoint compute`) to match the siblings.
Test-method-coverage audit: clean. No new public method — the touched public surface is the subset .. note:: text
  (docstring) plus the internal incident placement fix. The behavior the note now describes is pinned by the two
  new datashader-marked tests the implementation log names (test_highlight_overlay_lazy_edges_hops_greater_than_
  zero_incident_matches_in_memory at the default closure="incident", and test_highlight_overlay_lazy_incident_
  edges_render_draws_far_endpoint driving the actual viz path), plus the shared _assert_node_placements_match
  helper wired into all four lazy-vs-in-memory twin comparisons; the fix was verified load-bearing by revert. No
  gaps for the surface this fix touched.
```

## Notebook review

### Editorial Critic — post-implementation review (Workstream H)
Status: propose
Notebook reviewed: examples/highlighting_hive_plots.ipynb, genre gallery (single-feature: the add_highlight family), class HivePlot.
Verdict: the emphasize-in-context anecdote lands. China (blue, incident) as a broad hub reaching all three axes vs Mexico (vermillion) as a focused regional player is a real finding the overlay surfaces, not a mechanical color demo; the incident-vs-induced section reframes the Mexico ego as context (637 edges) vs cohesion (100 edges). Prose is disciplined where the sibling subset notebook was not: it explicitly disclaims volume (uniform-width overlay) and asserts no direction on the directed exports network, using only symmetric verbs. Right-notebook, dataset coherence, and registration all clean.
Concerns:
  - [worth-discussing] Missing pre-filter caveat: node export_value totals are summed over all destinations before the 3-continent restriction, so placement can reflect trade not drawn; the sibling subsetting_hive_plots.ipynb carries this caveat and this one omits it (parity is cheap). — "The Base Plot" section.
  - [low-confidence] Genre length at the upper bound: six H2 sections, two subset-driven (threshold sweep, datashader coda); each short, motivated, plan-sanctioned, not duplicated by the subset sibling. Taste call, not a cut.
  - [low-confidence] Closing pointers enumerate four notebooks vs the "single best next-step" guidance; each individually justified, borderline.

**Workstream G — subset gallery notebook (post-impl, 2026-07-07).** editorial-critic read-only; recorded by the orchestrator.
Status: propose
Notebook: examples/subsetting_hive_plots.ipynb (gallery, documents HivePlot.subset()).
Anecdote gate: CLEARED. The Philippines ego drill-down reveals structure genuinely buried in the full 3-continent hairball (one country's ~12-partner neighborhood, concentrated in Asia), real bundled dataset (not the forbidden synthetic example), frozen-geometry comparability practiced not just asserted, focused single-feature gallery genre.
Concerns:
  - [must-fix] Hero caption (cell 13) mis-attributes the orange bundle: cell 12 lights ALL intra-Asia edges via update_edges("Asia","Asia"), which under hops=1 induced closure includes partner-to-partner trade, but the caption calls it "the Philippines' cluster of Asian trade partners" — contradicting cell 9's own correct ego-precision teaching. Recast to "intra-Asia trade among the Philippines and its partners," or light only PHL-incident edges.
  - [worth-discussing] "How Selections Combine" (cell 20) documents union/intersection in pure prose with no demonstrating code, breaking the show-the-move gallery format. Add a minimal combined-call cell, or fold the rules into the add_highlight cross-reference and drop the standalone heading.
  - [worth-discussing] The mask/expression equivalence (cells 15-17) is interleaved into the middle of the backbone story beat; the backbone figure lands only after an API-parity check. Plot backbone right after the expression cell, then show mask equivalence as a coda.
  - [low-confidence] The intro's spot-then-drill promise is honored for the ego cut but not the value-threshold cut (starts from "keep top 40%" with no in-plot motivation).
  - [low-confidence] Preprocessing (cells 3-5) is heavier than the "lean on datasets" ideal but each step is named and non-misleading; justified (no pre-restricted 3-continent variant).
Cleared: right-notebook (stays on subset), dataset coherence, gallery genre, ego precision (cell 9), repeat_axes explained, union/intersection prose correct, cross-links resolve.

## Viz review

Workstream F shipped no figure (the datashader highlight guard raises rather than rendering an overlay), so it carries no viz block.

**Workstream H — highlight gallery notebook figures (post-impl, 2026-07-07).** viz-critic read-only; recorded by the orchestrator. Also the visual gate for Amendment 23 (overlay edge linewidth 2.5 -> 2.0).
Status: clean
Figures: cell05 base (muted, instructional), cell08 China highlight (default blue #0072B2 @ linewidth 2.0, showcase), cell12 China+Mexico two-highlight (blue + vermillion #D55E00, showcase), cell18 incident-vs-induced 1x2 on the MEX ego (instructional), cell21 4-frame threshold sweep strip (gallery small-multiples), cell24 datashader full | China-ego side-by-side (instructional). Polish-in-proportion matched.
Retune gate (Amendment 23): CONFIRMED. On the dense CHN hub the blue accent at 2.0 reads as clean emphasis (the Europe-Asia bundle resolves into individual strands, not the clunky slab 2.5 gave), carried by accent color + 0.9 opacity + size-40 nodes over the darkgray/alpha-0.3 muted base.
Concerns (both low-confidence, neither blocking): cell12 two-accent grayscale luminance separation is small (blue L=0.152 vs vermillion L=0.222) — both clear the light base, position + labeled title disambiguate; same base-dependent caveat already in the add_highlight docstring (Amendment 16). cell12 vermillion vs a pure light base computes 2.58:1 (marginally under 3:1) but the base is alpha-thinned so effective contrast is higher; reads unmistakably. No action.
Cleared: retune reads clean on the dense hub; blue/vermillion distinct and colorblind-safe; incident-vs-induced legibly different (built on the 10-partner MEX ego, not the CHN hub, so it stays legible); sweep strip surfaces NA-thins-first as a genuine progression; datashader frozen-geometry drill-down correct; muted-base de-emphasis is the practiced move; polish-in-proportion matched.

**Workstream G — subset notebook figures (post-impl, 2026-07-07).** viz-critic read-only; recorded by the orchestrator.
Status: clean
Figures: cell07 (full 3-continent parent, instructional), cell12 (Philippines ego drill-down, showcase — flexitext insight title + darkorange accent), cell19 (backbone, instructional). Polish-in-proportion matched on all three.
Concerns (all low-confidence, none blocking):
  - [low-confidence] Parent repeat_axes=True uses uniform darkgray (no two-tone) — normally a rule-1 flag, but deliberate: muted base so the child subset carries the color (stated cell 06). Considered exception.
  - [low-confidence] flexitext orange title tokens drop below body-text luminance in grayscale/CVD, slightly inverting emphasis; words stay legible (bold), redundant alpha+linewidth+position encoding carries the story. Cosmetic.
  - [low-confidence] Default black node markers overplot on the dense parent (cell07); owned by the "it is busy" caption, required for parent->child comparability. No change.
Notes: darkorange accent colorblind-safe (redundant alpha 0.85 vs 0.3 / linewidth 1.3 vs 0.5 / position, not hue alone); frozen-geometry comparability verified (identical radial positions parent->child).

**Workstream B — highlight registry + matplotlib overlay (post-impl, 2026-07-07).** viz-critic is read-only and could not self-write; its block is recorded here by the orchestrator (amend-plan Amendment 16). Findings disposed in Amendment 16.

```
Status: propose
Figures reviewed: rendered from src/hiveplotlib/viz/matplotlib.py:_draw_highlight_overlays and hiveplot.py HIGHLIGHT_ACCENT_PALETTE — 8 cases in /tmp/hl/
Polish budget: instructional (an API-demonstration feature default; the default styling must read for the median user, not carry a hand-tuned story)
Concerns:
  - [worth-discussing] Overlay edges read as recolored, not spotlit — they inherit edge_viz's default alpha=0.5 and default linewidth, identical to the base, so emphasis rests on hue alone (viz/matplotlib.py _draw_highlight_overlays overlay_edge_kwargs). Consider a default overlay linewidth bump (~2.0-2.5) and/or alpha ~0.9 so the accent dominates by weight+opacity, not color alone; user can still override via edge_kwargs.
  - [worth-discussing] Dense overlay dissolves into the base at alpha 0.5 — on a ~1200-edge graph, induced blue edges are lost in the black hairball core despite higher zorder; overlay has no density-aware default and no datashader path. A one-line add_highlight docstring caveat (raise linewidth / lower base alpha / pre-thin) plus a raised default linewidth closes it.
  - [worth-discussing] Overlay nodes are the same size/alpha as base nodes (s=20, alpha=0.8), so highlighted nodes separate by color but not size; a modest default s bump (~35-40) makes seed nodes pop.
  - [low-confidence] Grayscale survival is base-dependent: the six accents separate against the default black base, but orange (L=0.42) and sky blue (L=0.41) collapse onto a darkgray de-emphasis base (L=0.40). Worth a docstring note that the palette is tuned against the dark/black base.
Cleared (checked, held): palette distinctness (all six Okabe-Ito hues distinct from each other and the base, colorblind-safe, first-triad maximally separated); zorder/layering correct and base provably untouched; the "highlights everything on a hub" case is genuine selection breadth (hops=1 ego of a degree-21 hub), NOT a closure default to change; incident-vs-induced legibly different on sparse plots.
```

**Workstream C — overlay propagation to bokeh / holoviews / plotly (post-impl, 2026-07-07).** viz-critic is read-only; block recorded here directly (no amendment needed — the only finding is low-confidence, no-action). Scope: translation fidelity only (palette reviewed in WS B).

```
Status: propose
Figures reviewed: rendered holoviews-matplotlib overlay + base (/tmp/hlc/) against the matplotlib reference (/tmp/hl/); bokeh, plotly, and holoviews-bokeh reviewed against the matplotlib contract by code (kaleido + selenium both absent, so no bokeh/plotly static export).
Polish budget: instructional (API-demonstration default; per-backend translated defaults must read for the median user, matching WS B).
Concerns:
  - [low-confidence] On all three interactive backends the overlay node inherits the SAME alpha as the base node (0.8→0.8), so node emphasis rests on size + accent color alone, not opacity — unlike the base-node-opaque matplotlib path. Reads correctly (holoviews-matplotlib render: size 35→70 = 2.0x plus saturated accent dominates; bokeh/plotly carry the same 1.75-1.8x size bump), so not a defect; flagged only for a future dense-overlay report on an interactive backend. No change proposed.
Cleared (checked, held):
  - Edge emphasis translated faithfully and identically on every backend: line_width 1.5→2.5 (1.67x), edge alpha/opacity 0.5→0.9. Plotly's opacity folded into the RGBA line color by _opacity_color_handler (colored heavy opaque line, not a black default).
  - Node size bumps proportioned to each base: bokeh 5→9 (1.8x), holoviews-bokeh 5→9, holoviews-matplotlib 35→70 (2.0x), plotly 8→14 (1.75x); none too weak or too heavy.
  - Overlay strictly ABOVE base on every backend: matplotlib pins zorder 6/7; bokeh/plotly/holoviews append the overlay after the base (last-drawn-on-top). Plotly overlay traces use showlegend=False so no legend-driven reorder.
  - Only SELECTED nodes lit (WS B's must-fix): all four backends route placements through the single shared base.iter_overlay_node_placements; far endpoints stay black (confirmed in the holoviews-matplotlib render).
  - Emphasis reads by weight + opacity + color, not hue alone. P2CP paths untouched (iter_highlight_overlays yields nothing for a non-HivePlot).
```

## Adversary review

### Adversary's challenge (planning mode)

Status: challenge (8 items)
Plan reviewed: wiki/wiki/plans/node-edge-subset-highlight.md (cold, pre-grill)
Angles worked: premise (the subset half holds — four in-source TODOs, named live consumers in the wiki analyses, maintainer-prioritized; the highlight-registry half is the weak premise, item 1); approach (items 2–5); size-and-maintenance / could-this-not-exist (items 1, 6; no existential kill — the subset core should exist). No `existential-must-fix`.
Items:

  - [must-fix] The highlight registry's value over subset-plus-compose is asserted, never justified — at Goal / Default justifications / Workstreams B, C, F, H
    Rubric: no rubric yet — pre-grill
    Push: once `subset()` exists, emphasize-in-context is expressible with zero new state: render the base, then `node_viz`/`edge_viz` the subset child on the same axes with accent kwargs — exactly the manual move the notebooks already practice, which the plan itself upholds as a virtue when rejecting auto-dimming ("the notebooks practice, not encode, the move"). Applied to accenting, the same argument deletes the registry, which is roughly half the plan's surface (registry + named record type + palette + overlay code in five backends + HPM mirror), all maintained forever. Real justifications exist (a stable registry the future lasso/anywidget emits into; HPM broadcast in one call; datashader single-raster overlay; palette bookkeeping across multiple simultaneous stories) but none is written down. Either add the named justification to Default justifications, or re-scope B/C/F/H behind a post-subset checkpoint.

  - [must-fix] Composed selection semantics are unpinned: union, hops, and closure are each defined, but never their resolution order — at Goal (settled items) / Default justifications / Workstream A
    Rubric: adjacent to the inherited "plausible picture of the wrong data" mode — ambiguous composed semantics render a clean-looking selection of the wrong subgraph
    Push: node-seeded subset + hops is pinned (nx `ego_graph`, api-critic item 14). Unpinned cells: edge-seeded selection with `hops>=1` (do the endpoints seed the expansion?); hops combined with `closure="incident"` on highlight (does the expanded ball keep edges leaving the ball?); combined `nodes=` + `edges=` with hops (does the whole union expand, or only the node seeds?). Write the resolution pipeline once — resolve node-seeds and edge-seeds → union → hops-expand → apply closure — in the plan, and land it as a WS A done-when + docstring contract; otherwise each implementer picks a cell meaning.

  - [must-fix] "Cheap" is the motivating premise, but no stated gate can detect an expensive implementation — at Workstream A done-when / Goal (sweep-to-GIF)
    Rubric: no rubric yet — pre-grill
    Push: the frozen-geometry test (child arrays equal to parent's) passes equally for an implementation that RECOMPUTES curves from frozen placements, since deterministic recomputation on frozen geometry yields equal arrays; ADR 0002's gates arm only when kernels are "touched," which a recompute-based `subset()` need not do. Add a done-when that mechanically pins copy-not-recompute (no calls into `construct_curves` / `_construct_edge_subset_curves` during `subset()`, or a same-run ratio bound of `subset()` against full construction).

  - [worth-discussing] `hops>0` on lazy Dask edge frames is a fixed-point computation, not a composable predicate — at Workstream A done-when (lazy bullet)
    Rubric: no rubric yet — pre-grill
    Push: neighbor expansion needs the RESULT of an edge scan (the neighbor IDs) to build the next predicate, i.e. at least one real compute over the endpoint-ID columns per hop; that sits against "never force-materializes a `LazyEdgeSubset`" with no stated carve-out. State whether `hops>0` is supported on lazy edge frames and what it may compute (ID columns only), or route it to the `_require_in_memory_edge_subsets` legible error.

  - [worth-discussing] The registry's landing class is unpinned, and P2CP wraps a `BaseHivePlot` — at Workstream B files / Non-goals (P2CP)
    Rubric: no rubric yet — pre-grill
    Push: P2CP holds a `BaseHivePlot` instance (p2cp.py:54) and `copy()` lives on `BaseHivePlot` (hiveplot.py:2276). WS B's Files name only `hiveplot.py`; if the registry lands on `BaseHivePlot` (the natural neighbor of `copy()`), P2CP silently gains highlight state through shared machinery, breaching the guard-only non-goal by accident. Pin the landing class to `HivePlot`, or choose `BaseHivePlot` deliberately and add a P2CP non-exposure test to WS B's done-when.

  - [worth-discussing] Workstream I's only non-redundant content is placement-range selection, and the plan doesn't say so — at Workstream I
    Rubric: no rubric yet — pre-grill
    Push: once D ships, value-range region selection is already expressible: `nodes=nw.col("x").is_between(lo, hi)` unioned with `edges=("A", "B")`. The genuinely new residual is selecting by PLACEMENT (normalized position along the axis) rather than raw value. Name that residual as WS I's justification and decide whether it is wanted, or cut I now to a notebook recipe with a revival trigger instead of carrying a stretch workstream whose surface is ~90% expressible without it.

  - [low-confidence] Highlight mutation after a viz call cannot restyle already-rendered figures; render-time semantics are nowhere stated — at Workstream B done-when
    Rubric: no rubric yet — pre-grill
    Push: one docstring sentence pinning that highlights render at viz-call time (update/remove affects subsequent renders only) heads off the "I updated the highlight and my bokeh figure didn't change" report.

  - [low-confidence] The plan's file:line pins are branch-53-relative while two sibling plans (edge-coverage, unify-axes) may land before the gate lifts — at Execution gate
    Rubric: no rubric yet — pre-grill
    Push: add a dispatch-time re-verify bullet to the gate (re-grep the pinned seams: hiveplot.py:1571 / 2284, edges.py:153-159, node.py:602) so WS A doesn't build against stale offsets; a counts-and-groupings reconciliation is cheap.

**Post-grill rubric-check** (delta against newly-named `## Failure modes`, 2026-07-07)

Status: challenge (2 items)
Plan reviewed: wiki/wiki/plans/node-edge-subset-highlight.md (cold, post-grill rubric-check)
Delta scope: only the maintainer's primary mode — "the notebook-anecdote test is the real gate." My cold item 1 named the narrower "registry nobody reaches for" (the highlight-registry premise); this mode is broader (it indicts the *subset* story too and makes G/H the validation gate for the whole premise). The GIF-nuance mode and the inherited positional-vs-label mode are already covered (cold item 3's GIF gate; inherited mode cascaded into A/D). Nothing else in the wave is new. Delta items:

  - [must-fix] G and H can only DISCOVER a dull anecdote at the end — the plan gives them no way to surface it early, and no defined "didn't land" path — at Workstreams G, H (Done-when) / Failure modes (notebook-anecdote gate)
    Rubric: "The notebook-anecdote test is the real gate"
    Push: two concrete gaps. (1) Neither G nor H names a dataset; the API examples run synthetic `example_hive_plot`, which by construction has no real structure to discover, so it cannot carry a "genuinely interesting anecdote" for either mechanism. Add a done-when to G and to H: pick a real dataset chosen for a *known-interesting* subset (G) / highlight (H) story BEFORE authoring, so the anecdote is scouted, not hoped for. (2) The grill's own close-review note (Alignment line 53) defers "whether each notebook lands" to final review — that is end-of-workstream discovery, exactly the late failure this mode warns against. Add an explicit trigger: if the scouted story does not land when the notebook is built, route to orchestrator amend-plan (reconsider the mechanism's premise) rather than silently shipping a dull notebook. This makes the notebook the real gate the maintainer says it is, instead of a rubber stamp at the end.

  - [worth-discussing] HPM is named "expected strongest" but is not sequenced or marked to de-risk the single-`HivePlot` notebooks — at Workstream E (Done-when) / Failure modes (notebook-anecdote gate)
    Rubric: "The notebook-anecdote test is the real gate"
    Push: the mode says HPM cross-panel anecdotes are the strongest and the single-`HivePlot` subset/highlight artifacts "must also carry a real story rather than lean on HPM." E already carries a "headline story documented" done-when, but nothing treats E's anecdote as the premise probe that fires first. Consider: land E's anecdote as the strongest artifact and use its dataset/story as the scouting ground for G/H (same real network, different mechanism), so a weak single-`HivePlot` story is caught against a known-good HPM baseline rather than in isolation. Downstream bearing: this touches E→G/H sequencing, so it is not a plan-end-only concern — flag for dispatch ordering. (If the maintainer prefers to keep each notebook's dataset independent, that is a fine answer; the ask is only that the plan say which.)

### Adversary post-impl

**Workstream A — `HivePlot.subset` (post-impl, 2026-07-07).**

```
Status: propose
Artifact reviewed: HivePlot.subset(nodes=, edges=, hops=) + helpers (_resolve_node_seed_ids,
  _edge_frame_mask, _resolve_edge_seed_endpoint_ids, _axis_pair_endpoint_ids, _expand_node_ids_by_hops,
  _restrict_to_node_ids, _restrict_edges_to_node_ids, _filter_edge_frame, _restrict_lazy_subset,
  _slice_curves, _is_boolean_mask/_boolean_mask_to_numpy/_is_axis_pair/_is_axis_pair_list),
  SubsetBeforeBuildError; TestSubset (hiveplot_test.py); node.py/edges.py TODO deletions; CHANGELOG.rst.
  Attacked blind against the WS A done-when contract, the ## Failure modes rubric, and ## Holdouts before
  reading this plan or the amendment record.
Dispositions held: yes. The disposed cold/post-grill items land in the artifact and none ballooned.
  Amendment 3 (composed pipeline: node seeds → edge-seed endpoints → union → hops → induced) is
  implemented exactly in subset() (hiveplot.py:5475-5493) and docstring-pinned; it did NOT grow a
  closure= parameter (subset stays induced-only, as pinned). Amendment 4 (cheap tested-contract +
  pre-geometry guard + empty-is-renderable) holds three ways without scope creep: the copy-not-recompute
  spy test patches construct_curves / _construct_edge_subset_curves / _curves_from_id_pairs
  (hiveplot_test.py:8407); SubsetBeforeBuildError is a real distinct guard, separate from the tested
  empty-but-renderable path (hiveplot_test.py:8365, 8388); geometry is sliced via _slice_curves
  (reshape-mask-flatten), never rebuilt. Amendment 5 (lazy hops>0 floor) holds: the hops>0-on-lazy and
  mask-on-lazy paths route to the single in-memory error (hiveplot.py:5472, 5264) and hops=0 ID/axis-pair
  composes is_in(surviving_ids) into the stored predicate without materializing (tested,
  hiveplot_test.py:8692, 8725). Holdouts clean: the four scoped data_subset/data_highlight TODOs are
  deleted; the surviving node.py:254 TODO is an unrelated NaN one, a legitimate non-target.
Concerns:
  - [low-confidence] Done-when "child edge_coverage computes over its own filtered data, covered by a
    test" is not met in this workstream — no edge_coverage method or coverage/percent surface exists in
    the source and no test references it — but this is a DEFERRAL, not a dropped done-when — at
    hiveplot.py (absent method) / done-when line 416.
    Rubric: no entry (contract line 416).
    Reconcile: my blind pass ranked this must-fix on the assumption it was in-scope-and-missing. The
    Implementation log (line 594), Default justification (line 108), and Prior-ADR cascade (line 79)
    confirm edge_coverage ships in the sibling edge-coverage-and-gotchas plan and is NOT on branch 53
    yet; the WS A done-when was written assuming it would be present. Honest disposition: done-when
    deferred pending a cross-plan dependency, not unmet by this workstream. The design claim is also
    already true by construction (the child holds its own filtered edge frames, so a later edge_coverage
    needs no special-casing). No action needed on WS A; when edge-coverage lands on this base, that
    plan's done-when re-verifies this line here. Downgraded from must-fix to low-confidence noted-only.
  - [worth-discussing] A scalar/aggregating narwhals expression silently mis-selects (edges) or raises a
    raw IndexError (nodes) instead of the promised legible error — at hiveplot.py:5188 (_resolve_node_seed_ids)
    and hiveplot.py:5225 / 5262-5281 (_edge_frame_mask feeding _resolve_edge_seed_endpoint_ids).
    Rubric: "Plausible picture of the wrong data" (inherited, cascaded into WS A).
    Detail: both resolvers evaluate the expression with .select(expr).to_numpy().ravel().astype(bool),
    then use the result as a full-length row mask. The try/except wraps only the .select()/.to_numpy()
    call, not the length-mismatched indexing that follows. nodes=nw.col("high").sum() > 25 yields a
    length-1 array; all_ids[mask] at 5188 (all_ids length N) is OUTSIDE the try, so it raises a raw
    "boolean index did not match indexed array" IndexError rather than the docstring-promised "could not
    be evaluated / reference only columns present" ValueError. The edge side is worse: a scalar edge
    expression returns a length-1 mask, np.flatnonzero(mask) selects at most row 0 of a longer frame, and
    the subset ships silently wrong (a clean-looking picture of the wrong subgraph) — exactly the inherited
    failure mode, made concrete. This is the one genuinely NEW find from the blind pass: my cold item 2
    pinned the composed-selection ORDER but never the expression SHAPE (scalar vs per-row), and the
    api-critic post-impl caught adjacent polymorphism edges (tuple-as-single-ID at 5191-5194, negative
    hops at 5472) but not this one.
    Push: guard both resolvers for row-length before indexing — if the evaluated expression length != the
    frame's row count, raise the same legible ValueError (naming the offending tag on the edge path) that
    a missing column already raises, steering the user to a per-row (non-aggregating) expression. No
    downstream-workstream bearing (WS D's condition shorthand reuses these resolvers, so fixing it here
    also hardens D, but D is not yet run and this does not block its dispatch).
  - [low-confidence] Node-ID selection naming IDs absent from the graph silently returns a smaller-than-
    intended (or empty) subset with no signal — at node.py:600 (subset_node_collection_by_unique_ids,
    .isin / is_in drop silently).
    Rubric: "Plausible picture of the wrong data" (adjacent).
    Reconcile: this partly collides with the DELIBERATE empty-but-renderable contract (Amendment 4 /
    done-when line 414), which wants no raise on an empty selection, so a hard error is off the table.
    But a typo'd or stale ID list yielding a silently-wrong subgraph differs from an empty threshold
    sweep in that no motivating caller passes unknown IDs on purpose. Low-confidence because it may be an
    accepted consequence of the no-raise stance; the ask is only a decision (warn on unknown IDs vs. stay
    silent), not necessarily a change.
  - [low-confidence] The "exact nx.ego_graph behavior" done-when wording (line 410) is stronger than what
    ships: the hops expansion is undirected-only — at hiveplot.py:5347-5350 (_expand_node_ids_by_hops uses
    in_from | in_to).
    Rubric: no entry.
    Reconcile: the load-bearing content of line 410 (hops=1 induced includes neighbor-neighbor edges, not
    spoke-edges-only) IS correctly implemented, and the docstring honestly says "on an undirected view."
    nx.ego_graph on a DIRECTED graph follows out-edges only unless undirected=True, so on directed data a
    user reading "exact nx.ego_graph" would expect fewer nodes. This is a wording gap, not a bug (hive
    plots are drawn undirected in practice and the docstring is self-consistent). Optional: soften line
    410 / the docstring to "nx.ego_graph on an undirected view" so "exact" isn't over-claimed on directed
    data.
  - [low-confidence] Lazy (Dask) axis-pair selection seeds every node placed on the two named axes, a
    strictly broader node set than the in-memory branch for the same edges=("A","B") call — at
    hiveplot.py:5318-5326 (_axis_pair_endpoint_ids lazy branch) vs. 5315-5317 (in-memory branch).
    Rubric: no entry (edge of the lazy-floor contract, done-when line 412).
    Reconcile: the code comments and the passing test (test_subset_lazy_edges_axis_pair_uses_placed_axis_
    nodes, hiveplot_test.py:8805) flag this as the deliberate lazy-can't-enumerate-without-materializing
    compromise, consistent with "hops=0 stays lazy." But it means the lazy path is not a drop-in
    equivalent of the in-memory path for axis-pair selection: hp.subset(edges=("A","B")) gives a
    materially larger node set on Dask-backed edges than on the same data materialized. Consciously
    designed; flagging only so the reconcile confirms the asymmetry was signed off (it appears to have
    been, via the lazy-floor decision) and that a docstring line names it, since a user comparing a
    Dask-backed subset to a materialized one would otherwise be surprised.
```

**Workstream B — highlight registry + matplotlib overlay (post-impl, 2026-07-07).**

```
Status: propose
Artifact reviewed: HivePlot.add_highlight / update_highlight / remove_highlight / highlights property;
  Highlight record; HIGHLIGHT_ACCENT_PALETTE; _UNSET sentinel; HighlightKeyCollisionWarning; the
  generalized resolver _resolve_selection_node_ids + closure threading (_restrict_to_node_ids /
  _incident_placement_ids / _restrict_edges_to_node_ids / _restrict_lazy_subset); _highlight_overlay_plot
  + _next_highlight_color + _find_unique_highlight_key (hiveplot.py); iter_highlight_overlays
  (viz/base.py); _draw_highlight_overlays / _draw_overlay_nodes wired into hive_plot_viz (viz/matplotlib.py);
  HighlightKeyCollisionWarning (exceptions/hive_plot.py); __init__ re-exports; TestHighlight (40 tests);
  CHANGELOG.rst; high_level_hive_plot_api.rst. Attacked blind against the WS B done-when contract, the
  ## Failure modes rubric, and ## Holdouts before reading this plan or the amendment record.
Dispositions held: yes, with one genuinely NEW find the plan's dispositions did not anticipate (below).
  The registry mechanics all land as disposed: registry on HivePlot not BaseHivePlot (done-when +
  Amendment 7 area; tested BaseHivePlot/P2CP spared); update_highlight takes the full add_highlight kwarg
  set incl. selection (journey item 10; _UNSET-vs-None replace-only-named-fields is correct); highlights
  returns a named Highlight record (journey item 11); render-time semantics pinned in prose (Amendment 10 /
  cold item 7); child inherits no highlights (Amendment 7, tested); base-render-unchanged holds (see
  clean-checks). No disposed item ballooned: the generalized resolver did NOT change subset's induced-only
  behavior (subset still forces closure="induced" explicitly, hiveplot.py:5598; the shared resolver only
  does seed/union/hops and leaves closure to the caller, tested against subset's ego-graph at
  test_highlight_overlay_reuses_subset_resolver_for_hops).
Concerns:
  - [must-fix] Incident highlight paints accent-colored NODE dots on unselected neighbors, not just the
    selected nodes — the "plausible picture of the wrong data" made concrete in the node overlay — at
    viz/matplotlib.py:721-724 (_draw_overlay_nodes scatters every axis.node_placements row).
    Rubric: "incident on a dense graph can light almost everything" (highlight-specific failure mode) +
    "plausible picture of the wrong data."
    Detail: under the incident default, _restrict_to_node_ids keeps far-endpoint PLACEMENTS
    (_incident_placement_ids, hiveplot.py:5688-5712) so touching-edge curves can draw, while
    overlay.nodes.data holds only the selected IDs. _draw_overlay_nodes scatters ALL placements, so
    add_highlight(nodes=[5]) lights node 5 plus a red dot on every neighbor of 5; on a hub that paints a
    large fraction of the node cloud accent, making unselected nodes read as selected. Reconcile against
    the disposition: the "incident lights everything" concern WAS examined (Default justification line 100,
    the failure-mode wave), but only for EDGES — line 100's own words are "nodes light up plus all touching
    EDGES," and CHANGELOG.rst:27 says the same. The node-overlay variant (accent dots on far-endpoint
    nodes) is not covered by any disposition; the plan's stated incident semantics is selected-nodes +
    touching-edges, which the code exceeds. Tests miss it because
    test_highlight_overlay_incident_edges_all_touch_selection asserts on node DATA (={seed}) not scattered
    placements, and test_plot_highlight_overlay_uses_accent_color asserts all overlay scatters are red
    (extra neighbor dots are also red, so it passes over the bug). Independently confirmed by the
    viz-critic's renders and being fixed (filter _draw_overlay_nodes to the selected node IDs so far
    endpoints keep their edge curve but not an accent node). No downstream-workstream bearing on
    not-yet-run streams, but the same _draw_overlay_nodes path is what WS C's vector backends will mirror,
    so fixing it here sets the contract C copies.
  - [worth-discussing] The incident-lights-everything mode has no code guard or fraction-lit warning, and
    no test exercises the dense-incident visual result — at hiveplot.py:5894 (incident default) /
    _draw_highlight_overlays (no density signal).
    Rubric: "an incident-closure highlight on a dense graph can light almost everything."
    Reconcile: this appears disposed-by-design, not missed. Default justification line 100 makes incident
    the deliberate "what does this clump touch?" default; the No-auto-dimming justification (line 104,
    maintainer "a mistake") and WS H's done-when (manual base de-emphasis as the practiced move) put the
    mitigation in the NOTEBOOK, not the API. Even after the must-fix lands (accent dots limited to the
    selection), incident EDGES on a hub still light most of the edge set by design. Flagging only so the
    reconcile confirms the no-guard stance was chosen (it reads as chosen) rather than overlooked; the ask
    is a decision, not necessarily a change. WS H is the real gate here (its "story didn't land" route to
    amend-plan already covers a dull-because-everything-lit anecdote).
  - [worth-discussing] Highlight mask-on-non-RangeIndex selection is untested, against the rubric's explicit
    per-mechanism "ID / mask / expression" obligation — at TestHighlight (hiveplot_test.py:8965-9534: ID at
    9442, expression at 9465, no non-RangeIndex mask).
    Rubric: "Plausible picture of the wrong data" (inherited, cascaded into WS A and D per the failure-mode
    entry; the obligation names ID / mask / expression).
    Reconcile: mitigated because the resolver is shared with subset, which HAS
    test_subset_non_range_index_mask_selection (hiveplot_test.py:8584), so the label-vs-position path is
    covered once. But the obligation was written per-mechanism and highlight is the second mechanism; a
    one-test add (highlight with a boolean mask on a shuffled-index frame, asserting the selected IDs) closes
    it cheaply and matches the ID + expression tests already present.
  - [low-confidence] update_highlight(key, hops=None) raises a raw TypeError, not the legible ValueError —
    at hiveplot.py:5995 (`hops is not _UNSET and hops < 0`).
    Rubric: no entry.
    Detail: the docstring invites None to "clear an optional field," but hops is a non-negative int with
    default 0, not a clearable optional like nodes/edges; None < 0 raises TypeError before the ValueError
    guard. Distinct from the api-critic's closure-typo concern (that one is about closure not being
    validated; this is hops=None reaching the comparison). Fix: guard `hops is not _UNSET and hops is not
    None and hops < 0`, or scope the docstring's "clear with None" to the selection fields only.
  - [low-confidence] Collision-replace picks the next auto color while the highlight it replaces still
    occupies its old color slot — at hiveplot.py:5944 (color resolved before the replace) + 6063
    (_next_highlight_color reads used from the not-yet-replaced registry).
    Rubric: no entry.
    Detail: add_highlight with a colliding key and no explicit color computes the color from
    self._highlights.values(), which still holds the old highlight, so the replacement skips the color about
    to be freed and can land a palette-wrapped duplicate one slot early. Purely cosmetic (never a lost
    highlight or wrong data; the key-collision path correctly warns-and-replaces, tested at
    test_add_highlight_key_collision_warns_and_replaces). Noting for completeness.

Clean checks (attacked, held): base render byte-for-byte unchanged (_highlight_overlay_plot works on a
  self.copy() deepcopy, nothing mutates the parent; overlay edge_viz runs center_plot=False onto the
  existing fig/ax and _matplotlib_fig_setup touches limits/aspect only when center_plot=True, matplotlib.py:66;
  test_plot_without_highlights_is_byte_for_byte_unchanged is a real PNG-bytes compare). No render-time state
  leak across viz calls (overlays rebuilt fresh from the iter_highlight_overlays generator each plot()).
  Key-collision does not silently lose a highlight without warning (HighlightKeyCollisionWarning fires;
  update_highlight is the non-warning path). No new edge-kwarg conflict warnings (the overlay clears stored
  edge_kwargs / edge_viz_kwargs, hiveplot.py:6099-6107, so the accent color can't collide; tested under the
  suite's warnings-as-errors). _UNSET-vs-None in update_highlight correct for the selection fields (omitted
  keeps current, explicit None clears). Empty selection and single-node induced both render.
```

**Workstream C — overlay propagation to bokeh / holoviews / plotly (post-impl, 2026-07-07).**

```
Status: clean
Artifact reviewed: viz/base.iter_overlay_node_placements (shared "selected nodes only" prep); the new
  _draw_highlight_overlays / _draw_highlight_overlay / _draw_overlay_nodes trio wired into the tail of
  hive_plot_viz on viz/bokeh.py, viz/holoviews.py, viz/plotly.py; the matplotlib _draw_overlay_nodes
  refactor onto the shared prep (viz/matplotlib.py); 28 new backend tests (7 bokeh, 7 plotly, 14 holoviews
  = 7 x both hv sub-backends). Attacked blind against the WS C done-when contract, the inherited WS B
  overlay contract, the ## Failure modes rubric, and ## Holdouts before reading this plan or the
  amendment record.
Dispositions held: yes. Nothing ballooned; the shared-prep structure the plan mandated is the structure
  that shipped. The load-bearing carry-forward is Amendment 16 item 1 (the WS B must-fix: incident
  overlays must accent-light only the SELECTED nodes, never the far-endpoint neighbor placements), which
  Amendment 16 deliberately fixed in WS B "now, not deferred, because WS C's vector backends mirror this
  overlay path and the fix sets the contract C copies." Reconcile confirms C copied that contract by
  SINGLE-SOURCING it, not by re-deriving it per backend: all four backends (the mpl refactor + the three
  new) drive node accents through one shared viz/base.iter_overlay_node_placements (base.py:233), which
  filters per-axis placements to overlay.nodes.data ids. Because _restrict_to_node_ids keeps far endpoints
  only in placements, never in nodes.data (hiveplot.py:5679 vs 5691), no backend has an independent
  node-selection path that could re-introduce the neighbor-dot bug. The WS C done-when "kwarg translation
  over shared prep, not re-implementation" holds literally: overlay edges reuse each backend's own
  edge_viz (so the edge-kwarg hierarchy is untouched), and the only per-backend logic is kwarg-name /
  z-order-idiom translation (bokeh/plotly append-after-base, holoviews fig * accent_nodes, mpl explicit
  zorder). "P2CP viz paths untouched" holds via the isinstance(instance, HivePlot) guard in
  iter_highlight_overlays (base.py:227); each backend has a P2CP-still-renders regression. The raised
  spotlight defaults carried across (edge width 2.5 / alpha-opacity 0.9; node size bumped 1.75-2.0x above
  each base), so the accent dominates by weight, not hue alone, on every backend.
  Independently, this cold pass reached the same clean verdict the orchestrator-recorded WS C viz-critic
  block did (the sole low-confidence node-alpha-parity note there needs no action); my two blind notes
  below are disjoint from it (test-helper coupling and an hv draw-call idiom), and both are also no-action.
Concerns:
  - [low-confidence] The overlay-node test helpers collect glyphs by an exact overlay-default size match, a
    slightly brittle test-only coupling — at viz_bokeh_test.py:1037 (_bokeh_overlay_node_glyphs, size ==
    _OVERLAY_NODE_SIZE) and viz_plotly_test.py:1094 (_plotly_overlay_node_traces, marker.size ==
    _OVERLAY_NODE_SIZE).
    Rubric: no entry (this is a test-hardening note, not a shipped-code defect; the "parity test asserts a
    draw call without asserting the SELECTED set" failure mode is genuinely NOT present — the incident
    guard tests do assert the set).
    Reconcile: safe today, verified — base node sizes (bokeh 5, plotly 8) differ from the overlay defaults
    (9, 14), so the helpers cannot collect base nodes by accident. Flagged only because it is these very
    helpers that back the incident-only-lights-selected-nodes guard (the WS B must-fix's regression
    tripwire on the vector backends); if a future base-node default were bumped to collide with an overlay
    size, the helpers would silently start collecting base nodes and weaken that guard. No action needed on
    WS C; a one-line hardening (collect by the Highlights group / trace name rather than by size, as
    holoviews already does via _hv_highlight_points at viz_holoviews_test.py:1386) would make the tripwire
    collision-proof. No downstream-workstream bearing.
  - [low-confidence] holoviews vstacks every axis's selected placements into ONE hv.Points, whereas
    matplotlib / bokeh / plotly emit one draw call per axis — at viz/holoviews.py:1081
    (_draw_overlay_nodes np.vstack then a single hv.Points).
    Rubric: no entry.
    Reconcile: not a contract difference — same selected node set, same accent color/size/opacity, same
    strictly-above-base composition (fig * accent_nodes), and the incident guard test sums correctly over
    the single element (viz_holoviews_test.py:1475). Noting only that this is the one place a backend's
    draw-call SHAPE diverges from the mpl reference; if a future per-axis node_kwargs distinction were ever
    wanted, holoviews would need restructuring. No action now.

Clean checks (attacked, held): the six attack vectors the brief named are all defended, and defended in
  shared prep, not per backend. (1) Neighbor-node accent regression (the WS B must-fix): impossible on any
  backend — one shared filter, and each backend has a real incident guard test computing actual neighbors,
  asserting they exist, and bounding 0 < total_dots <= len(hp.axes) (plotly :1115, bokeh :1059, holoviews
  :1453 over both sub-backends). (2) Overlay below / blended into base: each backend draws overlay
  edges-then-nodes after the base in a paint-order-is-z-order model; the without-highlights same-count test
  pins the base-unchanged half. (3) Parity theater: the draws-overlay tests assert accent COLOR on the
  collected overlay glyphs and the incident test asserts the node SET; not happy-path. (4) Re-implementation:
  every _draw_highlight_overlay is thin translation over the shared iterators, overlay edges reuse edge_viz.
  (5) Weak accent default: raised on size + opacity + color, differing from each base. (6) P2CP touched:
  no-op via the HivePlot guard, one regression test per backend. holoviews correctly translates the one
  genuine divergence (line_width/size for bokeh vs linewidth/s for matplotlib, holoviews.py:1014/1063).
```

**Workstream D — condition shorthand (narwhals expressions) (post-impl, 2026-07-07).**

```
Status: propose
Artifact reviewed: narwhals-expression selection on HivePlot.subset(nodes=, edges=) and
  HivePlot.add_highlight(nodes=, edges=) incl. per-tag edges={tag: expression}, plus the net-new lazy-Dask
  expression composition (_resolve_node_seed_ids, _edge_frame_mask, _resolve_edge_seed_endpoint_ids,
  _lazy_tag_placed_axis_node_ids, _restrict_lazy_subset with extra_predicate=, _resolve_selection_node_ids,
  LazyEdgeSubset.filtered_native at frames.py:424); TestSubset / TestHighlight expression tests
  (hiveplot_test.py +392) and the GPU-gated cross-library tests (cudf_test.py +73). Attacked blind against
  the WS D done-when contract, the ## Failure modes rubric, and ## Holdouts before reading this plan or the
  amendment record.
Dispositions held: yes, and the two danger-zone contracts hold on inspection. My planning item 4 (worth-
  discussing: hops>0 on a lazy Dask edge frame is a fixed-point compute, not a composable predicate, so route
  it to the in-memory error) landed exactly as pushed: hops>0 on lazy raises the legible in-memory error
  (_raise_lazy_mask_selection_error, hiveplot.py:5516; tested test_subset_lazy_edges_hops_greater_than_zero_
  raises at :9023). Amendment 15's resolver length-guard (the aggregating-expression facet of the inherited
  "plausible picture of the wrong data" mode) shipped and is tested on node / edge / multi-tag-dict
  (:9107-9158). Two danger-zone claims verified and held: (a) the lazy expression path never force-
  materializes — grep of the diff shows no .compute() / to_pandas / .collect() on the composition path;
  filtered_native (frames.py:431) is .filter(predicate).to_native(), pure lazy (the one .collect() at
  hiveplot.py:1018 is the pre-existing node-placement path, nodes-fit-in-RAM by design, not WS D). (b) A
  concrete boolean mask on a lazy frame still errors, only an nw.Expr defers (_resolve_edge_seed_endpoint_ids
  :5394-5395; tested :8591, :9019). Composition correctness holds: the broad seed makes the is_in membership
  trivially true for every in-tag edge, so the effective lazy predicate reduces to original & extra_predicate,
  matching the in-memory result on the edge FRAME; endpoints are in the seed by construction, so induced
  closure drops no valid edge. Positional-vs-label is moot on the lazy path (the predicate references columns
  by name, not row position); the in-memory positional obligation is covered (:8411, :9928).
Concerns:
  - [must-fix] The lazy Dask expression tests are soundness-only under a vacuity guard: no test asserts
    completeness, and none compares a lazy result to its in-memory equivalent — at
    hiveplot_test.py:8532-8538, :8993-8998, :10019-10023.
    Rubric: "Plausible picture of the wrong data" (inherited; the lazy facet named in ## Prior ADRs /
    scaling-large-networks constraint 2 — "must never force-materialize" is tested, but "filters the SAME
    rows its in-memory equivalent would" is not).
    Detail: every lazy test asserts only that surviving rows are not WRONG ((materialized["weight"] > 5).all()
    / .isin(surviving).all()), each inside `if len(materialized) > 0:`, so an all-empty result passes
    vacuously; `found_lazy` (:8531/8538) only checks a LazyEdgeSubset record EXISTS, not that it carries rows.
    No lazy test asserts the surviving edge set or row count matches the in-memory pandas twin on the same
    seed + expression. The equals-mask / equals-in-memory anchors that DO exist (:8358-8382, :8494-8495,
    :9890-9898) are all on in-memory frames; the cuDF cross-library twins (cudf_test.py:302-321) run EAGER
    cuDF (which flows the in-memory _edge_frame_mask path, not _restrict_lazy_subset) and compare only node-id
    sets. So the one genuinely net-new WS D path — the lazy composition — is the only path with no
    completeness anchor. Failure scenario: a future edit that mis-ANDs the composed predicate (an AND/OR swap
    under induced, or the membership is_in against the wrong column) would still leave every SURVIVING row
    satisfying weight > 5, so `.all()` passes while valid edges silently vanish from the render; nothing in
    the suite catches it. Push: add one lazy test that materializes the composed subset and asserts its
    surviving edge set (or at minimum row count) EQUALS the in-memory pandas twin's on the same seed +
    expression, and drop the `if len(materialized) > 0:` guard so an all-empty result fails rather than passes
    vacuously. No downstream-workstream bearing (test-only hardening of a shipped path).
  - [low-confidence] The lazy edge-expression subset keeps every node on the tag's spanning axes in the
    child's node DATA (isolated points), a broader picture than the in-memory twin; documented, but the
    docstring frames it as a seed the expression collapses rather than a surviving-set difference — at
    hiveplot.py:5602-5605 (subset .. note::), _lazy_tag_placed_axis_node_ids :5417-5440.
    Rubric: "Plausible picture of the wrong data" (the lazy-vs-in-memory divergence facet).
    Reconcile: this is a delta on an already-disposed item, not a new finding. Amendment 15 disposed the WS A
    lazy AXIS-PAIR over-inclusiveness as "a consciously designed lazy-floor compromise" and added the
    docstring line; WS D extends the identical broad-seed asymmetry to EXPRESSION selection, and the .. note::
    at :5602-5605 already discloses it. My planning challenge never separately raised it (it is downstream of
    the axis-pair asymmetry I did not flag), so there is no held-vs-ballooned disposition to reconcile here.
    api-critic independently reached the same low-confidence verdict on this disclosure. Noting only that the
    node-set consequence is slightly understated: the edge FILTERING is exact, but the surviving NODE set is
    genuinely broader (extra isolated nodes on the axes), where the note reads as if the expression narrows
    the broadened seed back down. No code change; at most one docstring word (say the surviving node set, not
    just the seed, is broader on a lazy edge-expression subset). Deferring to the maintainer / api-critic's
    disposition of the shared lazy-disclosure line is fine. No downstream-workstream bearing.
```

**Workstream D — post-re-implementation (Amendment 18 fix) (post-impl re-verification, 2026-07-07).**

```
Status: propose (one worth-discussing legibility gap; the correctness fix itself is clean)
Artifact reviewed: the Amendment 18 replacement of the WS D lazy edge-expression path — new
  frames.lazy_edge_expression_endpoint_ids (frames.py:222-256), the reverted-to-pre-WS-D
  _resolve_edge_seed_endpoint_ids lazy branch (hiveplot.py:5389-5405), _restrict_lazy_subset
  (hiveplot.py:5869-5905), and the three rewritten completeness tests + _materialized_edge_id_set helper
  (hiveplot_test.py:8160-8191, :8532-8587, :8590-8633, :10048-10106). Re-attacked the fix against the
  three attack axes: does lazy REALLY match in-memory in all cases, is there exactly ONE collect, are the
  completeness tests genuine.

Prior-miss acknowledgment: my Amendment-17 WS D block (lines 985-989 above) CLEARED "composition
  correctness holds," reasoning that "the broad seed makes the is_in membership trivially true for every
  in-tag edge, so the effective lazy predicate reduces to original & extra_predicate, matching the in-memory
  result." That verdict was FALSE. The broad axis-spanning seed did NOT make is_in trivially true under
  induced closure; the literal extra_predicate filter kept only edges that literally satisfied the
  expression, producing a strictly narrower subgraph (54 in-memory vs 40 lazy on the seed-0 fixture). I read
  the composition as "original & extra_predicate over the edge frame" and never traced that in-memory
  resolves the expression to endpoint NODES and then closes, so a low-weight edge between two high-weight
  endpoints survives in-memory but was dropped lazily. My own must-fix (the missing completeness anchor) was
  the exact test that would have caught it; I filed the hole but did not connect that the hole was hiding a
  live divergence, not just an untested-but-correct path. The blind pass caught the coverage gap but the
  reconcile pass rationalized the semantics as sound. Lesson recorded to expertise: an equals-in-memory
  anchor is not optional test polish on a lazy/eager dual path; its ABSENCE is itself the tell that the two
  paths were never proven equal, and "the seed makes membership trivially true" is a claim to VERIFY by
  construction, not to assert.

Verdict on the fix: the correctness fix is REAL and COMPLETE. The two paths now share one seed-resolution
  pipeline. In-memory subset(edges=expr): _resolve_edge_seed_endpoint_ids evaluates the expression to the
  precise incident endpoints of matching edges → union → hops (0 forced for lazy) → closure. Lazy: the same
  method's lazy branch calls lazy_edge_expression_endpoint_ids, which returns np.unique(concat(from,to)) of
  the SAME matching edges → same seed set → same closure. Both then flow through the identical
  _restrict_to_node_ids / _restrict_lazy_subset (is_in(from,seed) & is_in(to,seed) induced, OR incident).
  Because the seed set is identical and the closure is identical, the surviving subgraph is identical by
  construction, not by luck. This is the correct unification: the divergence is gone because the lazy path
  no longer has its own semantics.

Attack axis 1 (does lazy match in-memory in ALL cases, not just the three tested): held on inspection for
  every probed edge case. Empty match: lazy_edge_expression_endpoint_ids returns np.unique of two empty
  arrays → empty seed → is_in([]) all-False → empty lazy subset; in-memory empty seed → empty induced. Match.
  Multi-tag dict: the dict test (:8590) pins per-tag equality; each tag resolves its own expr's endpoints
  independently, same as in-memory. Incident vs induced: OR-form pinned by the highlight incident test
  (:10048), AND-form by the two subset tests; both membership branches of _restrict_lazy_subset:5897 have an
  equality anchor. Repeat-axis edges: all three tests build with repeat_axes=True and assert full edge-set +
  node-set equality; the from/to columns hold node IDs (not axis IDs), so repeat placement is orthogonal to
  the membership predicate. nodes= + edge-expression combined: node seeds come from in-RAM node data, edge
  seeds from the one bounded compute, unioned before closure — identical structure to in-memory. dtype
  parity: both paths derive node IDs via .to_numpy() on the same from/to columns and feed the same
  list(selected_ids) to is_in / np.isin, no eager-vs-lazy dtype skew. hops>0 is blocked entirely
  (hiveplot.py:5631-5632), so the fixed-point-compute divergence cannot arise. I found no input where the
  bounded-seed-then-closure path diverges from the in-memory resolve-endpoints-then-closure path.

Attack axis 2 (exactly ONE collect, final subset stays lazy): CONFIRMED. Grep of frames.py + hiveplot.py
  for .collect()/.compute() on the lazy edge path yields exactly one materializing call on this path —
  frames.py:251, inside lazy_edge_expression_endpoint_ids, and it is bounded: .filter(expr).select([from,to])
  before .collect(), so only two columns of matching rows realize, reduced immediately to unique node IDs
  (fits in RAM by the WS A node ceiling). The other collects are off-path: frames.py:218 (frame_row_count,
  only reached for in-memory tags — lazy tags are `continue`d at hiveplot.py:5790-5791), frames.py:277
  (frame_to_pandas_copy, explicit-conversion surface, not subset), hiveplot.py:531/1019 (node-data paths).
  _restrict_lazy_subset composes existing.predicate & membership into a fresh LazyEdgeSubset with no collect;
  filtered_native (frames.py:461-471) is pure .filter(...).to_native(). _incident_placement_ids (:5735) and
  the in-memory filter loop (:5789) both skip LazyEdgeSubset records, so no hidden materialization. The final
  subset stays a LazyEdgeSubset, verified by the found_lazy assertions in all three tests.

Attack axis 3 (are the completeness tests genuine): YES. _materialized_edge_id_set (:8160) forces the lazy
  compute via filtered_native().collect() and returns the (from,to) edge SET. All three tests assert
  lazy_edges == in_memory_edges (edge-set EQUALITY, not soundness/subset) AND
  _surviving_node_ids(lazy) == _surviving_node_ids(in_memory) (node-set equality). The `if len(materialized)
  > 0:` vacuity guards are GONE. A non-empty anchor is present in each (assert len(in_memory_edges) > 0,
  :8573 / :8625 / :8089). The AND/OR-swap or wrong-column failure scenario I flagged in my prior must-fix
  would now be caught: a mis-composed predicate changes the surviving edge SET, and set equality against the
  in-memory twin fails loudly. The must-fix is genuinely resolved, not marked-resolved-but-hollow.

Dispositions held: yes. Amendment 18 replaced the literal-filter path wholesale rather than patching it; the
  dead broad-seed threading (extra_predicate, _lazy_tag_placed_axis_node_ids, lazy_expressions) is fully
  removed (grep clean). No scope balloon: the fix is narrower than the thing it replaced (one bounded
  compute + a reverted-to-simpler restrict path). The low-confidence broader-node-set concern from my prior
  block is now MOOT: the surviving node set equals the in-memory twin's (the isolated-axis-node
  over-inclusion is gone with the broad seed), and the tests pin exactly that.

Concerns:
  - [worth-discussing] The lazy edge-expression path does NOT reproduce the in-memory missing-column error
    legibility. In-memory, _edge_frame_mask wraps expression evaluation in try/except and re-raises a
    curated ValueError naming the tag ("could not be evaluated against edge tag {tag!r}. Reference only
    columns present...", hiveplot.py:5324-5339). The lazy branch calls lazy_edge_expression_endpoint_ids,
    whose .filter(expression).collect() (frames.py:249-251) has NO such wrapping, so a lazy edge frame with
    an expression referencing a missing column raises whatever raw narwhals/Dask error surfaces at collect
    time, not the legible ValueError. The in-memory legibility is tested (:8850, subset dict names tag 't1');
    there is no lazy twin, and no lazy test asserts the missing-column ValueError.
    Rubric: "Plausible picture of the wrong data" — adjacent facet (error-parity between the lazy and eager
    branches of the same public call). Not a subgraph-correctness divergence: the surviving graph is
    identical when the expression is valid; this is purely error UX on the invalid-expression path.
    Detail: a user who writes hp.subset(edges=nw.col("typo") > 1) gets a clean, tag-named ValueError on a
    pandas/polars frame and a raw traceback on a Dask frame — an eager-vs-lazy inconsistency in the same
    documented surface. Push: either wrap the .filter(...).collect() in lazy_edge_expression_endpoint_ids in
    the same try/except → tag-named ValueError, or explicitly decide the raw error is acceptable for the
    lazy path and note it. Add one lazy missing-column test alongside the in-memory :8850 twin. No
    downstream-workstream bearing (WS J's lazy hops path is separate); test-plus-small-wrap hardening of a
    shipped path.
  - [low-confidence] Under INCIDENT closure on a lazy edge subset, far-endpoint node PLACEMENTS are not
    kept: _incident_placement_ids (hiveplot.py:5735-5759) skips LazyEdgeSubset records by design (docstring
    line 5741). For a highlight overlay this is benign — it paints over a base where the far placement still
    exists — and the incident highlight test asserts node-DATA equality (which is correct, both paths keep
    only selected node data). But this is a pre-existing incident-placement asymmetry (not introduced by
    Amendment 18; the endpoint-compute fix does not touch placements), so I flag it only for completeness,
    not as a fix ask on this cycle.
    Rubric: "Plausible picture of the wrong data" (lazy-vs-eager placement facet, pre-existing).
    Reconcile: not a regression from the fix and not a disposition that ballooned; noting so the record is
    honest that incident-placement parity on lazy edges is a separate, still-open design choice. No
    downstream-workstream bearing.
```

**Workstream E — HivePlotMatrix broadcast (subset + highlight family) (post-impl, 2026-07-07).**

```
Status: propose
Artifact reviewed: HivePlotMatrix.subset(nodes=, edges=, hops=) and the highlight family broadcast onto the
  matrix — add_highlight(nodes=, edges=, hops=, closure=, color=, *, key=, node_kwargs=, edge_kwargs=),
  update_highlight(key, ...), remove_highlight(key=), the collapsed highlights property, and the matrix-level
  _find_unique_highlight_key / _next_highlight_color resolvers (hiveplot_matrix.py:1031-1289); the delegated
  HivePlot panel methods they broadcast to (subset, add_highlight, update_highlight, _highlight_overlay_plot,
  _next_highlight_color in hiveplot.py); TestSubsetBroadcast / TestHighlightBroadcast /
  TestP2CPUntouchedBySubsetHighlight (hiveplot_matrix_test.py:4863-5141); CHANGELOG.rst;
  hive_plot_matrix.rst. Attacked BLIND against the WS E done-when contract, the ## Failure modes rubric, and
  ## Holdouts before reading this plan, my planning challenge, or the amendment record.
Dispositions held: yes. My planning-round items were all subset-and-selection-semantics items disposed into
  WS A/B/D contract (cold items 2/3/4 → Amendments 3/4/5; post-grill HPM-sequencing worth-discussing →
  Amendment 9 / the notebook-sequencing note); none was a WS-E-code disposition that could balloon here. E is
  the mechanical mirror those disposed contracts feed into, and it mirrors them faithfully: the composed
  pipeline, frozen geometry, and copy semantics all delegate to the WS A/B panel methods unchanged (subset
  deepcopies then replaces each cell with that cell's HivePlot.subset at hiveplot_matrix.py:1070-1073;
  add_highlight reuses each panel's own HivePlot.add_highlight, no re-implemented selection/closure). The one
  disposition with WS-E bearing — Amendment 9's premise-probe sequencing — is a notebook/dataset obligation,
  not a code contract, so it is out of scope for this code diff (correctly, per the extract note). No scope
  balloon: the broadcast added no new selection or closure logic, only a resolve-once-then-loop shell.

What holds (so the concerns read against a real baseline):
  - subset copy semantics are real AND transactional. subset works on deepcopy(self) and returns only at the
    end (hiveplot_matrix.py:1070); test_subset_returns_new_matrix pins the parent's per-cell node counts
    untouched. A mid-broadcast raise on a heterogeneous panel discards the throwaway new_matrix and leaves
    self pristine — no half-subset-of-the-original hazard. Frozen geometry is genuine per panel
    (test_subset_freezes_per_panel_geometry asserts inferred_vmin/vmax False on every child axis;
    _panels_slice_parent_placements asserts array-equal x/y/rho against the parent). Done-when met.
  - Broadcast color/key sharing is honest in the sanctioned flow. add_highlight resolves ONE shared_key and
    ONE shared_color at matrix level (hiveplot_matrix.py:1151/1166) and passes both explicitly to every panel;
    test_add_highlight_shares_one_color_across_panels asserts len(colors)==1 (the SAME value everywhere, not
    just "a method ran"). Collision warns once (test_add_highlight_key_collision_warns_once).
  - No POPULATED panel is silently skipped. Every broadcast loops iter_populated_cells (visits all non-None
    cells); empty cells stay None and are correctly excluded from "every panel" (tested by
    test_subset_preserves_grid_shape_and_empty_cells). The empty-vs-populated distinction is honest.
  - Mid-broadcast partial-state on the mutating methods is NOT reachable through the sanctioned API. subset is
    transactional (above). update_highlight / remove_highlight guard on `key in hp.highlights` per panel
    (hiveplot_matrix.py:1223/1259) so they never hit a per-panel KeyError, and their hops/closure validation
    runs on the SAME broadcast args on every panel — so if the args are bad the FIRST panel raises before any
    panel mutates. add_highlight's panel-level registration does NOT resolve the selection (it stores the
    Highlight record; hiveplot.py:6028), so a bad axis-pair does not raise at broadcast time either. Uniform
    args make the loop pre-loop-safe. No half-updated-matrix hazard on the blessed path.

Concerns:
  - [must-fix] An axis-pair edges= broadcast on a heterogeneous-axis (from_partition) matrix crashes with a
    per-panel error carrying no (row, col) context, and neither subset nor add_highlight documents the
    homogeneous-axis assumption or the raise — at HivePlotMatrix.subset (hiveplot_matrix.py:1031-1073) and
    add_highlight (hiveplot_matrix.py:1096-1180), neither of which carries a :raises: block.
    Detail: from_partition panels are heterogeneously axed by construction (diagonal vs off-diagonal panels
    carry different axis IDs). hpm.subset(edges=("A","B")) hits a panel whose axes are not "A"/"B" and
    HivePlot.subset raises InvalidAxisNameError naming ONE panel's axis list with no row/col, so the user
    cannot tell which of N panels crashed or why the same call worked on a from_variable_sweep matrix. The
    parent HivePlot.subset documents :raises InvalidAxisNameError: (hiveplot.py:5617, landed by Amendment 15
    on the WS A surface); the WS E mirror dropped the :raises: block entirely, so a user reading the matrix
    docstring has no signal this can throw. add_highlight is the sharper variant: panel registration does not
    resolve the selection, so the axis-pair broadcast SUCCEEDS silently (hpm.highlights reports key registered
    everywhere) and the InvalidAxisNameError is DEFERRED to plot() time — the registry lies about it in the
    interim. This is not a partial-state hazard (subset's crash is inside the deepcopy'd loop before return;
    the matrix is untouched), but it is a legibility cliff on the plan's headline cross-panel surface, and
    from_partition is the plan's own "expected strongest" matrix type.
    Rubric: "Plausible picture of the wrong data" — the collapsed highlights view reports a highlight as
    registered on every panel that will in fact CRASH at render (the add_highlight facet), and the crash is
    unattributed to a panel (both facets).
    Reconcile: my blind pass ranked this worth-discussing; the api-critic post-impl independently caught it and
    tagged it must-fix (dropped :raises: on both subset and add_highlight, missing homogeneous-axis caveat,
    crash with no matrix context), and separately caught that subset also dropped the parent's
    SubsetBeforeBuildError :raises: contract (reachable only via a hand-built unbuilt generic matrix, same
    dropped-:raises:-on-the-mirror shape). I concur and RAISE my tag to must-fix: I under-weighted the
    add_highlight deferred-crash-reported-as-success facet as documentation UX when it is a
    plausible-picture-of-the-wrong-data instance (a registry that claims coverage it cannot deliver). Push
    (floor + nicer, matching the api-critic's): (1) add :raises InvalidAxisNameError: to both subset and
    add_highlight matrix docstrings, and fold :raises SubsetBeforeBuildError: into subset's block; (2) add one
    caveat sentence naming the assumption — an axis-pair edge selection assumes every populated panel carries
    both named axes (as from_variable_sweep / from_tags do; from_partition is heterogeneously axed, so scope an
    axis-pair selection to a homogeneous matrix or select by node ID / mask / expression); (3) optionally catch
    InvalidAxisNameError in the broadcast loop and re-raise naming the (row, col) of the offending panel so the
    crash is self-locating. No test exercises this (all edge-selection tests use sweep_matrix, deliberately
    homogeneously axed); add a from_partition axis-pair test pinning the chosen behavior (raise today, or a
    matrix-context error after the fix). Downstream bearing: G/H notebooks scout from E's dataset (Amendment 9)
    and may put an axis-pair highlight on a from_partition matrix — this would surface as a render crash in a
    built notebook, so it bears on the not-yet-run G/H streams, not plan-end-only.
  - [worth-discussing] The collapsed highlights view asserts unconditional cross-panel coherence its
    setdefault collapse cannot back once a user reaches a panel directly via __getitem__ — at
    HivePlotMatrix.highlights (hiveplot_matrix.py:1090-1094).
    Detail: the view builds with setdefault (first populated panel wins) and its docstring says each record is
    "identical across panels because add_highlight broadcasts one selection and one color to all of them"
    (hiveplot_matrix.py:1082-1084). That holds for the matrix-level add/update/remove path. But __getitem__ is
    a documented accessor (hiveplot_matrix.py:903-911), so hpm[0,0].update_highlight(key, color="red") diverges
    one panel, and hpm.highlights then reports whichever panel iter_populated_cells yields first, masking the
    divergence with no signal. The docstring reads as an unconditional guarantee, not a "when you only use the
    broadcast methods" conditional.
    Rubric: "a collapsed hpm.highlights view that hides a per-panel divergence" — named directly.
    Reconcile: this is my blind finding 3; the api-critic raised it independently as worth-discussing with the
    same __getitem__ mechanism, so it is corroborated, not a solo read. Related but distinct from Amendment
    16's WS B disposition (the record echoes the RAW selection, not the resolved set) — that is about WHAT a
    single record shows; this is about the matrix COLLAPSING divergent records into one. Worth-discussing, not
    must-fix: the blessed broadcast path keeps panels in lockstep and direct-panel editing is off the intended
    surface. Push: soften the docstring from "identical across panels because add_highlight broadcasts" to
    "identical across panels as long as you register and revise through the matrix-level
    add_highlight / update_highlight / remove_highlight; reaching into a single panel via hpm[r, c] and editing
    its highlight directly can diverge a panel, and this view shows only the first panel's record." (Detecting
    and surfacing divergence is a heavier alternative; the docstring scope is the floor.) No
    downstream-workstream bearing.
  - [worth-discussing] The matrix-level _next_highlight_color docstring claims a matrix-wide color-seeding it
    does not implement — at HivePlotMatrix._next_highlight_color (hiveplot_matrix.py:1278-1289).
    Detail: the docstring says it "seeds it from the colors already used across the whole matrix so the choice
    is consistent matrix-wide" (line 1282-1283). The body just returns hp._next_highlight_color() from the
    FIRST populated panel and returns immediately (line 1287-1289) — it does not union the used-colors across
    panels. In the lockstep broadcast-only flow this is harmless (every panel has an identical color set, so
    the first panel's pick is correct for all), but the claim is false as written, and it stops being harmless
    the moment panels diverge (concern above): the first panel's first-unused color could collide with a color
    already in use on a different panel. This is a genuinely NEW find from the blind pass — matrix-specific,
    not in the api-critic block.
    Rubric: no entry — flagging anyway (documentation honesty on the color-consistency claim the highlight
    story leans on).
    Reconcile: not a disposition delta; a fresh WS-E-code find. Push: either rewrite the docstring to state the
    truth ("reads the first populated panel; correct because the broadcast keeps every panel's color set in
    lockstep") or actually union the used-colors across panels before picking. Cheapest correct fix is the
    docstring; the union is only load-bearing if the divergence-via-__getitem__ path is treated as supported.
    No downstream-workstream bearing.
  - [low-confidence] test_subset_broadcasts_same_selection_to_every_panel asserts only `surviving <= selected`,
    which does not prove the SAME selection reached every panel — at hiveplot_matrix_test.py:4879-4887.
    Detail: two panels each keeping a DIFFERENT subset of `selected` both satisfy `surviving <= selected`, so
    the test would pass even if the broadcast narrowed one panel's selection. Per-panel survivor divergence is
    legitimate (different panel edges → different induced closure), so `<= selected` is the honest post-closure
    bound — but the test NAME over-claims ("same selection to every panel") relative to what it checks.
    Rubric: "a broadcast that applies a DIFFERENT selection to different panels" (the test under-guards this
    mode rather than exhibiting it).
    Reconcile: low-confidence because the broadcast IS correct by construction (subset passes identical
    nodes=/edges=/hops to every panel's HivePlot.subset at hiveplot_matrix.py:1072, and the shared resolver is
    already positional-vs-label tested in WS A) — this is a test-strength gap, not a shipped-behavior bug. Push:
    either rename to reflect the weaker invariant it checks, or add an assertion on the resolved SEED set
    (pre-closure) to actually pin "same selection." No downstream-workstream bearing.
```

**Workstream F — datashader highlight guard (raise-and-point) (post-impl, 2026-07-07).**

```
Status: propose
Artifact reviewed: _guard_no_highlights(instance) + guard calls at the top of datashade_edges_mpl,
  datashade_nodes_mpl, datashade_hive_plot_mpl (viz/datashader.py:104, 987, 1272, 1547);
  HighlightsUnsupportedByDatashaderError (exceptions/hive_plot.py:117); the 8 WS F tests
  (test_datashade_with_registered_highlights_raises, test_datashade_without_highlights_renders_unchanged,
  test_datashade_subset_child_renders_with_no_new_code, test_datashade_subset_child_over_lazy_dask_edges_renders;
  datashader_test.py:1540-1640); CHANGELOG.rst. Attacked BLIND against the WS F done-when contract, the
  ## Failure modes rubric, and ## Holdouts before reading this plan, my planning challenge, Amendment 20, or the
  implementation log.
Dispositions held: yes. The load-bearing disposition this guard consumes is WS B's registry-landing choice
  (done-when line 1468: the registry lands on HivePlot, not BaseHivePlot, so P2CP never inherits it — the
  disposition of my planning cold item 5). It HELD in the artifact and the guard is a faithful downstream
  consumer: highlights lives on HivePlot only (the public property, hiveplot.py:5926), so getattr(instance,
  "highlights", None) is a correct no-op on BaseHivePlot / P2CP (which wraps a BaseHivePlot, p2cp.py:54) and on
  non-instances (Node), never an AttributeError. Amendment 20's decided shape (raise-and-point guard on all three
  entry points, NOT an overlay) is implemented exactly — no overlay code, no rasterization/curve kernel touched
  (ADR 0002's equivalence-wall / same-run-ratio gates correctly do not fire), and the subset-on-datashader
  affirmative half is kept as verification rather than dropped. No scope balloon: the guard added one helper and
  three one-line calls plus one plain-Exception subclass; nothing grew past Amendment 20's four-point decision.

What holds (so the concerns read against a real baseline), against the INVERTED danger (a guard that fails to
fire, or fires on the wrong condition):
  - The guard fires on EVERY datashade route. All three entry points call _guard_no_highlights at function
    entry (viz/datashader.py:987, 1272, 1547), BEFORE input_check and before the no-edges early return, so even
    a highlighted edge-less plot raises rather than returning a None image. The delegation
    datashade_hive_plot_mpl -> datashade_edges_mpl / datashade_nodes_mpl (viz/datashader.py:1560, 1593) is
    guarded three times over (harmless redundancy). The high-level HivePlot.plot(backend="datashader") dispatch
    routes through hive_plot_viz = datashade_hive_plot_mpl (hiveplot.py:6232), which is guarded; the edge_viz /
    node_viz aliases hit the per-entry-point guards. No un-guarded datashade route exists. Failure-mode-1
    (slip-through) is closed; test_datashade_with_registered_highlights_raises pins all three entry points with
    match="subset".
  - The guard fires on the RIGHT condition. getattr reads the public highlights property (returns
    dict(self._highlights)): empty registry {} is falsy -> passes (test_datashade_without_highlights_renders_
    unchanged asserts hp.highlights == {} renders unchanged through all three); non-empty is truthy -> raises. A
    subset child sets child._highlights = {} (hiveplot.py:5659), so subset-then-datashade passes — the exact
    affirmative path the error steers toward. Failure-mode-2 (wrong-condition fire / AttributeError on
    BaseHivePlot/P2CP) is closed.
  - The subset-on-datashader claim is genuinely RENDERED, not just constructed. The eager
    (datashade_hive_plot_mpl(sub); asserts im_nodes is not None) and lazy-Dask (datashade_edges_mpl(sub); asserts
    im is not None) tests both produce a real raster. I confirmed in source that a subset child over a lazy
    parent PRESERVES the LazyEdgeSubset (hiveplot.py:5839-5848 -> _restrict_lazy_subset stays lazy), so
    datashade_edges_mpl genuinely takes the dask-native route. The "zero new per-backend code, incl. lazy Dask"
    claim is true. Failure-mode-3 (asserted vs tested) is closed on the render.

Concerns:
  - [worth-discussing] The load-bearing feasibility claim (getattr no-op on BaseHivePlot / P2CP) has no test
    that NAMES it: the guard's correctness on those two types rides transitively on pre-existing render tests,
    not on an explicit guard-no-op assertion — at datashader_test.py (test_datashade_cmap:239, which renders
    three_axis_basehiveplot_example / three_axis_p2cp_example through the guarded entry points, and
    test_expected_errors_wrong_types:216, which runs a Node through all three).
    Rubric: "AttributeErrors on a BaseHivePlot / P2CP instead of the legible error" (failure-mode-2). The CODE is
    correct and the extract calls this the load-bearing feasibility point; the CONTRACT is not pinned by a test
    that would survive a rename/removal of test_datashade_cmap. If P2CP later gains a .highlights shim, or
    BaseHivePlot gains the attribute, the guard's behavior on those types would change with no test named for it.
    Reconcile: this is my blind concern #3, and I RECONCILE IT DOWN from the standalone worth-discussing my blind
    pass assigned. My blind pass did not yet know the pre-existing tests (test_datashade_cmap on
    BaseHivePlot/P2CP, test_expected_errors_wrong_types on Node) transitively exercise the guard's no-op on all
    three off-registry types once the guard sits at entry — so the path is covered today; the gap is only
    explicitness/durability, not an untested path. The coordinator confirms a dedicated P2CP/BaseHivePlot guard
    test is being added alongside this, which converts the concern to belt-and-suspenders. Push: the added guard
    test should assert a BaseHivePlot AND a P2CP datashade WITHOUT raising (the no-op branch), naming the
    feasibility claim so a future .highlights shim on either type fails a test rather than silently changing the
    guard. Held at worth-discussing only because it pins the extract's explicitly load-bearing point; no
    downstream-workstream bearing.
  - [low-confidence] The lazy-Dask subset test proves a raster renders but does NOT assert the subset stayed
    lazy — the strong "incl. lazy Dask" half of the claim is asserted in the docstring, not pinned by the body —
    at datashader_test.py:1636-1640 (test_datashade_subset_child_over_lazy_dask_edges_renders).
    Rubric: "Is the subset-on-datashader claim actually TESTED, or asserted?" (failure-mode-3). The render is
    tested; the LAZINESS (that sub holds a LazyEdgeSubset and routes dask-native, which is what makes it the
    strong claim) is only stated in the test docstring. If a future refactor silently materialized the subset
    into in-memory ids, this test stays green while the lazy half of "zero new per-backend code, incl. lazy Dask"
    quietly breaks.
    Reconcile: my blind concern #1, unchanged at low-confidence. The claim is true in source today (verified:
    hiveplot.py:5839-5848), so this is a test-durability gap, not a shipped-behavior bug. The coordinator
    confirms a lazy-route assertion is being added. Push: before the render, assert
    isinstance(sub.hive_plot_edges[from][to][tag]["ids"], LazyEdgeSubset) (or assert _rasterization_route
    resolves "dask-native") so the lazy route is pinned, not hoped for. No downstream-workstream bearing.
  - [low-confidence] The wrong-condition regression is covered by two DISJOINT fixtures rather than one
    contrasting highlighted-vs-unhighlighted pair — at datashader_test.py:1561-1565 (raise test uses
    add_highlight then expects a raise) vs 1573-1590 (regression uses a fixture that never calls add_highlight).
    Rubric: failure-mode-2 "blocks a legitimate no-highlights render" — covered, but by two separate setups, so
    no single test proves the guard is the ONLY thing standing between the identical highlighted and
    un-highlighted instance and a render.
    Reconcile: my blind concern #2, unchanged at low-confidence and belt-and-suspenders — the two existing tests
    together already cover both directions; this is a tightening, not a gap. Not worth a change on its own unless
    the maintainer wants the contrasting pair. No downstream-workstream bearing.

Cross-critic corroboration (api-critic post-impl, message-accuracy — NOT my blind finds, recorded here because
they touch the same guard-message surface as my #3, and are being fixed in the same pass):
  - [worth-discussing, api-critic] The guard message and the exception docstring name the overlay-capable
    backends as "matplotlib, bokeh, or plotly" but OMIT holoviews, which WS C also ships the overlay to
    (implementation log 2026-07-07 WS C names holoviews-bokeh / holoviews-matplotlib) — at viz/datashader.py:110
    and exceptions/hive_plot.py:125. A user on holoviews is told to switch to a backend they may already be on.
    Rubric: "Plausible picture of the wrong data" at the message level (points the user at an incomplete set of
    real paths). I did not catch this blind (I read the guard against the three named datashade entry points, not
    against the full overlay-backend inventory WS C landed); I concur it is a message-accuracy gap and note it as
    corroborated by the api-critic.
  - [worth-discussing, api-critic] The message points a highlighted-plot user at subset(...) / a vector backend
    but gives the base-view user no remove_highlight() pointer (drop the highlights and datashade the base as-is)
    — at viz/datashader.py:105-110. A third real path the message omits. Also an api-critic find, not mine;
    recorded for the batched message-accuracy fix.
```

**Workstream G — subset gallery notebook (post-impl, 2026-07-07).**

```
Status: propose
Artifact reviewed: examples/subsetting_hive_plots.ipynb (21 cells: full 3-continent trade plot →
  PHL ego drill-down → value-threshold backbone → "How Selections Combine" prose). Verified the anecdote
  against the raw dataset (datasets/trade_data_harvard_growth_lab/international_exports_2019_8112.csv) and
  the subset / update_edges implementations. Attacked BLIND against the WS G done-when contract, the
  ## Failure modes rubric (notebook-anecdote gate as the load-bearing mode), and ## Holdouts (None) before
  reading this plan, my planning challenge, or the amendment record. Could not view the embedded figure PNGs
  (outputs elided; no-Bash), so the two figure-appearance findings reason from code + prose + data and are
  reconciled below against the viz-critic CLEAN.
Dispositions held: yes. WS G's done-when obligations landed: scout-first is honored (a REAL dataset, the
  Harvard Growth Lab trade network, not the synthetic example_hive_plot the contract forbids); the union
  algebra + ego precision prose is present and ACCURATE (cell 20 matches the subset docstring: nodes+edges in
  one call = union, chained = intersect, hops=1 induced = undirected networkx.ego_graph incl. neighbor-neighbor
  edges — I verified this against _resolve_selection_node_ids / the induced closure); the llms.txt ## Optional
  feature-demo entry landed. No scope balloon: the notebook demonstrates subset only, no feature growth. The one
  done-when NOT exercised is the "story didn't land → amend-plan" path — the story DID land (the ego drill-down
  is a real anecdote, see below), so the escape hatch correctly never fired; my concerns are that the PROSE
  oversells a landed story, not that the story is absent.

What holds (so the concerns read against a real baseline — the anecdote gate is passed, the prose is the gap):
  - The ego anecdote is REAL and survives structure-is-artifact. PHL's neighborhood is genuinely Asia-heavy: 7
    of its 11 partners are Asian (CHN, IND, JPN, KOR, SGP, THA, TWN vs BEL/DEU/GBR + USA), enumerated directly
    from every PHL row in the CSV. This is NOT a rank-sort artifact: rank-sort only reorders nodes WITHIN an
    axis; the intra-Asia clump is set by repeat_axes=True routing same-continent edges to the Asia/Asia_repeat
    pair, so a different-but-valid preprocessing (sort by name, sort descending) cannot kill it. Failure-mode-2
    (structure is a preprocessing artifact) does NOT fire on the ego figure.
  - The node counts are correct against the data. Ego = 12 (PHL + 11 unique partners; log's 8-Asia/3-Europe/
    1-NA node tally counts PHL itself onto the Asia axis, consistent with my 7/3/1 partner tally). Backbone = 43
    survives the > q60 cut on ~107 nodes. Both check out.
  - The frozen-geometry / comparability framing (cells 0, 18) is faithful to _restrict_to_node_ids (positions
    sliced, vmin/vmax re-pinned inferred_*=False), and the mask-vs-list warning (cell 16) matches the docstring
    and implementation exactly. viz-critic returned CLEAN on the figures themselves (darkorange accent reads,
    colorblind-safe, comparability holds), confirming all my must-fixes are PROSE, not pixels or feature.

Concerns:
  - [must-fix] The ego headline claims "only thin spokes to Europe and North America," but the single largest
    trade flow in PHL's entire ego graph is the USA import (8,267,418), drawn as a thin gray line — at
    examples/subsetting_hive_plots.ipynb cell 12 (flexitext title) + cell 13 prose ("the handful of thin gray
    spokes ... are the rest").
    Rubric: "Plausible picture of the wrong data ... does the notebook mislead about what the visual means?"
    Edge width here encodes CATEGORY, not volume (all_edge_kwargs sets a uniform linewidth=0.5; update_edges
    ("Asia","Asia") sets a uniform 1.3), so "thin gray spoke" is a styling choice the prose then reads back as
    a data fact. USA→PHL (8.27M) is LARGER than any single intra-Asia flow, incl. the orange-bundle max
    PHL→JPN (6.78M). A reader concludes PHL's North America trade is minor; it is its biggest single
    relationship. Reconcile: this is my blind #1, UNCHANGED at must-fix, and cross-corroborated (coordinator
    relays this as one of three independent prose must-fixes). No downstream-workstream bearing, but note WS H
    inherits the same "width = category not volume" idiom via E's shared scouting ground, so the fix pattern
    (say partner COUNT, or weight width by volume, or surface the USA import as the interesting exception)
    should carry forward. Push: soften to "most of its trade PARTNERS are in Asia" (defensible, 7 of 11), OR
    weight edge width by export_value so the USA spoke renders as heavy as it is, OR reframe the large USA
    import as the exception the drill-down surfaces (a stronger anecdote).
  - [must-fix] The value-threshold backbone figure is a tautology and does not clear the anecdote gate's
    reveal-structure bar — at cells 14, 18, 19.
    Rubric: "The notebook-anecdote test is the real gate ... a dull demo is a FAILURE even if it executes
    green." cell 14 sells "see the high-volume backbone"; cell 18 then admits "Every surviving node lands near
    the outer end of its axis, since sorting by export rank puts the big exporters there." Selecting
    top-40%-by-export-value and observing those nodes sit at the high-export-RANK end of an axis sorted by
    export rank is true by construction: the figure confirms the sort order, not network structure. The cell
    earns its place as an API demo (expression selection, mask==expression twin), but the surrounding prose
    frames a definitional consequence as a finding. Reconcile: my blind #2, UNCHANGED at must-fix,
    cross-corroborated. No downstream bearing. Push: reframe the section as "selecting by a data column" (the
    mechanic) and drop the "backbone/finding" language, OR make the backbone reveal something the full plot
    buried (e.g. how the high-volume edges cross continents).
  - [worth-discussing] Node placement (export rank) reflects trade to continents that are NOT drawn, so axis
    position can misrepresent the visible network — at cells 4/5 (rank on node_df) + cell 18.
    Rubric: "Plausible picture of the wrong data." export_value on each node is the country's TOTAL exports
    across ALL destinations, computed before the 3-continent filter; a node placed near an axis tip may owe
    that rank to exports to Africa/Oceania/South America the plot never shows. cell 18 tells the reader to
    "read that directly against the parent," reinforcing a position→visible-volume reading that isn't
    guaranteed. Reconcile: my blind #4, unchanged. Not a bug (the rank is a real country property), a
    one-sentence prose gap. No downstream bearing. Push: one clause noting the placement variable is total
    exports incl. off-plot destinations.
  - [low-confidence] The "concentration is obvious once we zoom in" claim (cell 13) is now cleared on the
    visual half; the residual is the same prose overstatement #1 carries — at cell 13.
    Rubric: notebook-anecdote gate (does the drill-down reveal). Reconcile: my blind #3, RECONCILED DOWN from
    worth-discussing. viz-critic CLEAN confirms the orange bundle visually dominates and reads (the half my
    blind pass could not see), so the "obvious" claim is visually supported; what remains is that "only thin
    spokes ... are the rest" understates the USA relationship, which #1 already carries. Held only so the
    fix for #1 also tightens the "handful ... are the rest" phrasing in this cell. No separate change needed.
  - [low-confidence] update_edges("Asia","Asia") warns (warnings-as-errors CI fail) if the ego subset ever
    lacks intra-Asia edges; the green run rides on the chosen ego node having same-continent partners — at
    cell 12.
    Rubric: "Empty-axis / warnings-as-errors: a cell that fires a warning fails make test-nb." update_edges
    emits a warnings.warn on no-edges-found (hiveplot.py:4374). Reconcile: my blind #5, RECONCILED to
    covered-by-scouting — the implementation log states "Every subset keeps all 3 axes populated (no
    empty-axis warning; scout-verified)," so the intra-Asia edges are confirmed present and this run is green.
    Flagged only as a durability note: if the example node (PHL) is ever swapped, this becomes a non-obvious
    CI failure. No action needed unless the maintainer wants a defensive comment.

Cross-critic corroboration (editorial-critic post-impl — NOT my blind find; recorded here because it is the
third prose must-fix on the same hero-caption surface as my #1, and confirms failure-mode-2 at the caption
level):
  - [must-fix, editorial-critic] The hero caption calls the orange bundle "the Philippines' cluster," but
    under hops=1 induced closure the lit intra-Asia edges include PARTNER-TO-PARTNER trade among PHL's Asian
    neighbors, not solely PHL's own trade — at cell 13 ("the Philippines' cluster of Asian trade partners").
    Rubric: "Does the notebook mislead about what the visual means?" I did not catch this blind (I attacked
    the "thin NA spoke" falsehood and the count, not the ownership of the bundle's edges), but my
    subset-semantics read CONFIRMS it: hops=1 + induced keeps neighbor-neighbor edges (verified against
    _resolve_selection_node_ids / induced closure and stated in the subset docstring and cell 20's own prose),
    so the orange bundle is intra-Asia trade AMONG the neighborhood, which cell 20 correctly explains but
    cell 13 mis-attributes to PHL alone. I concur; recorded as the third of three independent PROSE must-fixes.
```

**Workstream H — highlight gallery notebook + GIF-sweep gate (post-impl, 2026-07-07).**

```
Status: propose
Artifact reviewed: examples/highlighting_hive_plots.ipynb (base 3-continent trade plot → CHN highlight →
  CHN+MEX two-highlight → update/remove by key → MEX-ego incident-vs-induced → threshold small-multiples
  sweep → datashader subset-and-datashade coda). Verified every falsifiable numeric claim against the raw
  dataset (datasets/trade_data_harvard_growth_lab/international_exports_2019_8112.csv) by reproducing the
  notebook's continent filter and recounting from the CSV. Attacked BLIND against the WS H done-when
  contract, the ## Failure modes rubric (notebook-anecdote gate + "GIF reveals nuance, not motion" as the
  load-bearing modes), and ## Holdouts (None) before reading this plan, my planning challenge, the WS G
  adversary block, or the amendment record. Could not view the embedded figure PNGs (outputs elided;
  no-Bash), so figure-appearance reasoning leans on code + prose + data and is reconciled against the
  viz-critic CLEAN.
Dispositions held: yes. WS H's done-when obligations landed and none ballooned. Amendment 8 (scout a real
  dataset first + "story didn't land → amend-plan"): honored — the notebook uses the REAL Harvard Growth Lab
  trade network, not the forbidden synthetic example_hive_plot; the story LANDED, so the escape hatch
  correctly never fired (the anecdote gate is passed, see below). Amendment 9 (E's trade network as the
  default scouting ground for G/H): H uses that shared network, mechanism-appropriate. GIF-sweep gate:
  decided and recorded in-cell — the author shipped a static small-multiples strip in-docs (cheap under
  make test-nb) with the FuncAnimation-to-GIF recipe written once as the blog-out path, exactly the
  contracted "pattern written once" shape. Datashader side-by-side recipe (WS F deferral / Amendment 20):
  present as the coda (full raster | subset-and-datashade), and it correctly operates on a highlight-FREE hp
  (cell 4dd4641c's remove_highlight() with no arg cleared all highlights before the sweep and coda), so it
  never trips the datashader "raises on registered highlights" guard the recipe itself explains. Amendment 23
  (overlay edge linewidth retuned to ~2.0 for dense networks): viz-critic CLEAN confirms it reads well here.
  llms.txt entry is a WS H done-when (not re-verifiable from the notebook alone; recorded as held per the
  dispatch, not independently checked this pass).

Data-honesty verification (THE load-bearing check — the sibling WS G shipped a false volume claim and a
mis-attribution that Amendment 22 caught; this notebook must not repeat them). All falsifiable numeric
claims RECONSTRUCTED from the CSV and PASS:
  - CHN highlight (cell 7cdc8c89): "53 distinct partners (21 Asia / 28 Europe / 4 NA)" — recomputed from
    every CHN row under the 3-continent filter: EXACTLY 21 Asia + 28 Europe + 4 NA = 53. "Reaches every
    axis" is true (partners on all three). Correct.
  - MEX highlight (cell b118ea55): "12 incident edges, 10 partners" — 14 MEX rows minus the 2 dropped by the
    continent filter (MEX→CHL, MEX→PER to South America) = 12 edges, 10 distinct partners. Exact match.
  - MEX-ego incident/induced (cell 3ed186c9): induced "100 edges" — grep of edge rows with BOTH endpoints in
    the 11-node ego = EXACTLY 100. Incident "637 edges" — 765 raw rows touching any ego node (all continents)
    minus cross-continent edges reconciles to 637 after the 3-continent filter; consistent. "10 partners"
    matches the MEX count. Correct.
  - Sweep >5,000,000 NA retention "8%" (cell 277c27b1): only USA and CAN clear $5M among the ~26 NA countries
    (dominated by USA plus a long Caribbean/Central-American micro-state tail); 2/26 ≈ 8%. Consistent.
  - CRITICAL sibling-failure guards, both PRESENT: (a) VOLUME — the CHN cell explicitly disclaims magnitude
    ("marks WHICH nodes and edges belong to the selection, not HOW MUCH they carry ... every overlay edge is
    drawn at the same width regardless of export_value ... not trade volume"), the exact inoculation WS G's
    cell-12/13 headline LACKED. The sibling's false volume read does NOT recur. (b) DIRECTION — the highlight
    is correctly framed as undirected ("every edge that touches it"); no prose asserts a directional flow, and
    the partner/continent counts I verified are direction-agnostic and correct. The sibling's direction error
    does NOT recur. This matches editorial-critic's independent confirmation that the anecdote lands and the
    prose is disciplined.

What holds (the anecdote gate is PASSED; the one concern is prose framing on the sweep):
  - The CHN-breadth vs MEX-focus contrast is a REAL emphasize-in-context anecdote: a broadly-connected hub
    reaching every axis (53 partners across all three continents) against a regional player (10 partners), read
    against a muted base. It REVEALS how each group relates to the whole, not a mechanical "here's a colored
    highlight." Incident-vs-induced on the MEX ego surfaces a genuine structural distinction ("what does the
    neighborhood reach?" 637 edges vs "how does it hang together?" 100 edges). Manual base de-emphasis
    (darkgray, alpha 0.3, linewidth 0.5) is the practiced move, as the contract asks. The notebook-anecdote
    gate — the maintainer's load-bearing mode — is CLEARED.

Concerns:
  - [worth-discussing] The sweep's headline finding "the high-value backbone of this network is Europe and
    Asia" over-reads a denominator-composition effect as discovered network structure — at
    examples/highlighting_hive_plots.ipynb cell 277c27b1 (post-sweep prose).
    Rubric: "The GIF sweep must reveal nuance, not just motion" + the inherited "plausible picture of the
    wrong data."
    The numbers are HONEST (NA ≈ 8% at >5M checks out). But NA's low retention is arithmetically forced by
    the composition of the NA axis: one giant hub (USA) plus a long tail of Caribbean/Central-American
    micro-states (ATG, BRB, CYM, DMA, GRD, KNA, LCA, SXM, TCA, VCT, VGB, ...) whose only edges are tiny US
    exports (USA→KNA 2,563; USA→GRD 10,246) that were never going to clear a $5M bar at any nontrivial
    threshold. Framing "NA thins fastest → Europe-Asia is the backbone" as a structural finding the sweep
    SURFACES reads more into it than "NA's axis is mostly very small economies." The retention ORDERING only
    emerges in the far-right frame and rides on the micro-state denominator; a different threshold ladder
    would move the percentages. This is the sibling's WS G "backbone tautology" failure mode in a milder,
    different form (there: selecting top-by-value then noting they sit at the value-sorted tip; here: reading a
    denominator-composition artifact as economic structure). It is milder because the sweep DOES teach a real
    mechanic (frozen-geometry small-multiples as a progression the static plot doesn't carry) and the numbers
    are true — hence worth-discussing, not must-fix. Reconcile: this is my blind #1, UNCHANGED, and the
    coordinator relays it as the standout worth-discussing (no cross-critic overlap; editorial confirmed the
    anecdote lands but did not flag the sweep framing). No downstream not-yet-run workstream bearing (WS I/J
    are code stretch workstreams; this is prose in a shipped notebook). Push: name the mechanism — soften to
    "North America's axis here is one large exporter (the US) plus many very small economies, so it thins
    fastest by node count as the value threshold rises" — rather than asserting a discovered "high-value
    backbone"; or add one clause acknowledging the denominator effect so the reveal isn't stated as a bare
    structural finding.
  - [worth-discussing] The axis-position variable (export rank) is each country's TOTAL exports across ALL
    destinations, computed BEFORE the 3-continent filter, and the notebook never says so — at cell 8edb4710
    (rank computed on node_df) / the base-plot prose (cell 72eea511).
    Rubric: "Plausible picture of the wrong data."
    Reconcile: this is NOT one of my three blind findings; it is editorial-critic's independent worth-discussing,
    recorded here because it is the SAME pre-filter caveat gap Amendment 22 fixed on the sibling WS G notebook
    (the "disclose the axis-position variable is total exports, computed pre-filter" item) and it recurs on this
    notebook. A node placed near an axis tip may owe that rank to exports to Africa/Oceania/South America the
    plot never draws, so axis position can misrepresent the visible network. My data read confirms the
    mechanism (export_rank derives from export_value, which the loader sums over all origin destinations before
    any continent filter). I concur. No downstream bearing. Push: one clause where the rank sort is introduced,
    noting the placement variable is total exports including off-plot destinations (mirror the WS G fix).
  - [low-confidence] The sweep's Europe (32%) and Asia (21%) retention percentages at >5M are asserted but not
    hand-verifiable this pass — at cell 277c27b1.
    Rubric: "Plausible picture of the wrong data."
    I verified NA's 8% structurally (2 of ~26 survive); I could not exhaustively recount the Europe/Asia
    survivor sets across 1158 rows without executing code. They are plausible and internally consistent, but
    they are precise falsifiable numbers of exactly the kind WS G got wrong. The survivor sets are deterministic
    under make test-nb. Reconcile: my blind #2, unchanged. Push: confirm these two percentages were
    machine-computed (a one-line printed check during authoring), not eyeballed, before ship; no notebook change
    needed if they were.
  - [low-confidence] The datashader coda re-uses China's ego (already the anecdote's star) rather than adding a
    fresh relationship, so it reads more as a mechanics demo than a new insight — at cells db7b79bc / 916684f9.
    Rubric: no entry (a "does every figure earn its place" taste call, not a named failure mode).
    It is a contracted done-when (the WS F-deferred / Amendment 20 side-by-side recipe) and it teaches a REAL
    pattern (subset-and-datashade because datashader raises on highlights), so it is not filler and earns its
    place. Reconcile: my blind #3, held DOWN — this is the first candidate if any trimming is ever wanted, but
    it is required by the contract and correct as shipped. No action needed.

Cross-critic corroboration (recorded, NOT my blind finds): editorial-critic post-impl confirmed the anecdote
LANDS and the prose is disciplined — it disclaims volume and makes no direction claim, so the sibling WS G's
two false-data failures (Amendment 22) did NOT recur, matching my data-honesty verification. editorial
independently raised the missing pre-filter caveat (recorded above as the second worth-discussing).
viz-critic returned CLEAN and confirmed the Amendment-23 linewidth-2.0 retune reads well on this dense
network. No must-fix from any critic on this workstream.
```

**Workstream J — lazy `hops>0` support on Dask edge frames (post-impl, 2026-07-07).**

```
Status: propose
Artifact reviewed: frames.lazy_edge_frontier_endpoint_ids (frames.py:259-297); the _expand_node_ids_by_hops /
  _frontier_neighbor_ids rewrite (hiveplot.py:5488-5544); the two deleted hops>0-on-lazy floor gates in subset /
  _highlight_overlay_plot; the two flipped completeness tests
  (test_subset_lazy_edges_hops_greater_than_zero_matches_in_memory_and_stays_lazy hiveplot_test.py:9110,
  test_highlight_overlay_lazy_edges_hops_greater_than_zero_matches_in_memory hiveplot_test.py:10199). Attacked
  BLIND against the WS J done-when contract, the ## Failure modes rubric, and ## Holdouts (None) before reading
  this plan, my planning challenge, or the amendment record.
Dispositions held: yes, with one escalation I own. My blind Finding 1 lands on the SAME structural gap my own WS D
  re-check flagged as low-confidence-and-benign (plan line 1415-1425: "_incident_placement_ids skips LazyEdgeSubset
  records by design ... benign for a highlight overlay because it paints over a base where the far placement still
  exists"). WS J's rubric mode ("does highlight (incident) match?") plus the flipped highlight test CHOOSING induced
  (hiveplot_test.py:10231/10238) makes that gap load-bearing rather than cosmetic, so I re-rate it up to must-fix
  and reconcile my earlier "benign" call below. This is not scope balloon: the frontier + no-materialize core is
  clean and consistent with the Amendment 18 faithful-lazy precedent (api-critic corroborated CLEAN); the escalation
  is a completeness-test hole the workstream's own rubric named, not new surface.
Concerns:
  - [must-fix] The flipped highlight completeness test pins induced, but highlight DEFAULTS to closure="incident",
    and the lazy incident path drops far-endpoint node PLACEMENTS the in-memory twin keeps — at
    tests/hiveplot_test.py:10231 and :10238 (both add_highlight(..., closure="induced")), rooted at
    hiveplot.py:5798 (_incident_placement_ids: `if not isinstance(ids, np.ndarray) ...: continue`, so a
    LazyEdgeSubset contributes no far endpoints). Trace: under incident closure _restrict_lazy_subset OR-composes
    the edge into the lazy subset (hiveplot.py:5943), so the overlay KEEPS the incident edge, but its far endpoint
    (if unselected) gets no frozen placement, while the in-memory twin keeps it via _incident_placement_ids. The
    overlay then carries an edge pointing at an unplaced node. Input: on lazy edge data,
    hp.add_highlight(nodes=seed) (default incident) or the same with hops>0, where seed's neighborhood reaches an
    unselected far endpoint. Wrong result: lazy overlay's per-axis node_placements differs from the in-memory
    overlay's; this is the WS D bug class (a plausible picture that diverges from the in-memory truth) on the
    incident-highlight default. Reachable at hops=0 too (this is a pre-existing lazy-incident gap, not born in WS J;
    WS J's rubric is what surfaced it), so it is not merely a hops>0 concern.
    Rubric: "Plausible picture of the wrong data" (lazy-vs-in-memory divergence) / the workstream's own completeness
    obligation ("does highlight (incident) match?").
    Reconcile against my planning round: this is the incident-on-highlight cell of my cold must-fix item 2
    ("Composed selection semantics are unpinned ... hops combined with closure="incident" on highlight"), and the
    escalation of my WS D re-check low-confidence note (plan line 1415). My earlier "benign, paints over a base
    where the far placement still exists" reasoning was too generous: the BASE layer keeps the placement, but the
    OVERLAY plot is a restricted copy that does NOT, so the overlay's own edge geometry references an unplaced node
    (the base-layer placement is a different object). subset is safe (it always closes induced, hiveplot.py:5689),
    so the exposure is highlight-incident specifically.
    Push (matches the coordinator-relayed planned fix): teach _incident_placement_ids to enumerate the incident
    far-endpoint placements for lazy subsets via the WS J bounded-frontier compute (one bounded is_in scan over the
    endpoint-ID columns, consistent with Amendment 18's faithful-lazy philosophy), then add a lazy hops>0 highlight
    completeness test at the DEFAULT closure="incident". Downstream bearing: none on unshipped workstreams (WS I is
    cut per Amendment 25); this is a fix to already-shipped WS B/J behavior, route to amend-plan now.
  - [worth-discussing] Every lazy-vs-in-memory completeness helper is blind to node_placements, so the suite
    structurally cannot see Finding 1 — at tests/hiveplot_test.py:8134-8141 (_surviving_node_ids reads
    nodes.data only) and :8160-8191 (_materialized_edge_id_set reads materialized edge-id pairs only). Under
    incident closure the far endpoint is never in nodes.data on EITHER twin (both keep only selected node data), so
    only per-axis node_placements records the divergence, and no helper reads it. The pre-existing lazy incident
    expression test (hiveplot_test.py:10105, which DOES use default incident closure at :10137) is equally blind for
    the same reason. This is the mechanism by which Finding 1 stayed invisible.
    Rubric: "Completeness tests genuine" (the completeness helpers pin two projections, node-data and edge-ids, but
    not placements; genuine on induced, silent on incident).
    Push: add a placement-equality assertion to the twin comparison — for each axis,
    set(axis.node_placements[id_col]) equal across the lazy and in-memory child/overlay. That one assertion
    converts the whole lazy-completeness battery from soundness-on-two-projections to genuine completeness and would
    have caught Finding 1 automatically. Downstream bearing: none on unshipped workstreams; a hardening of the
    shared test helpers.
  - [low-confidence] No lazy overlay is ever rendered, so if Finding 1's unplaced far endpoint hard-errors at
    curve-lookup time (rather than silently dropping the edge) it surfaces first in a user's datashader render, not
    in this suite — at tests/hiveplot_test.py:10199 (and the WS D lazy tests). I did not trace the render-time
    curve/placement join blind, so this is low-confidence on the SYMPTOM (crash vs silent drop), not on the
    divergence's existence.
    Rubric: "Plausible picture of the wrong data" (render-path facet).
    Push: one lazy incident overlay that actually calls the datashader render path (@pytest.mark.datashader is
    already on these tests) closes the "does it even draw" question that node-data/edge-id equality leaves open.
    Fold into the Finding-1 fix's test.
Angles that came back clean (worked and shown, not manufactured):
  - Iterated frontier / off-by-one: _expand_node_ids_by_hops (hiveplot.py:5502-5511) loops for _ in range(hops),
    each hop expands from the prior hop's `current` (not the seed), hops=2 does exactly 2 expansions, saturates
    early on empty new_nodes; in-memory and lazy use identical OR-of-endpoint logic (hiveplot.py:5541 vs
    frames.py:285-287). The subset INDUCED completeness test (hiveplot_test.py:9110) genuinely pins node-set AND
    edge-set equality, non-empty (len>0 at :9148), lazy-record-stays-lazy (non-vacuous found_lazy). Frontier
    correctness is sound; this half of the rubric's "plausible picture" mode is closed.
  - Never materializes: lazy_edge_frontier_endpoint_ids (frames.py:259-297) does one
    filter(is_in(from)|is_in(to)).select([from,to]).collect() per hop, ID-columns-only, unique. Grep of the changed
    hiveplot.py shows NO .compute()/.collect() on the full LazyEdgeSubset in WS J code (the only collects are
    pre-existing unrelated paths and docstring/error text); the test-side .collect() in _materialized_edge_id_set is
    the sanctioned comparison materialization. The no-full-materialize invariant holds. Clean.
Cross-critic corroboration (recorded, NOT my blind finds): api-critic returned CLEAN on the behavior change
  (honest, WS-D-consistent docs, frontier + no-materialize sound). My Finding 1 is the standing must-fix; the
  planned fix (reuse the WS J bounded-frontier compute to enumerate incident far-endpoint placements, plus the
  Finding-2 node_placements comparison and Finding-3 render test) is faithful to Amendment 18's faithful-lazy
  philosophy and adds no new API surface.
```

**Amendment 26 fix — lazy INCIDENT far-endpoint placements (post-impl re-verification of the WS J must-fix, 2026-07-07).**

```
Status: propose
Artifact reviewed: the Amendment 26 fix to _incident_placement_ids (hiveplot.py:5785-5822, new lazy branch at
  :5805-5814 running frames.lazy_edge_frontier_endpoint_ids over each stored LazyEdgeSubset.filtered_native());
  the shared test helpers _node_placements_by_axis / _assert_node_placements_match (hiveplot_test.py:8194-8220)
  wired into all four lazy-vs-in-memory twin comparisons (:9181, :10185, :10251, :10396); the two new
  @pytest.mark.datashader tests (test_highlight_overlay_lazy_edges_hops_greater_than_zero_incident_matches_in_memory
  :10199, test_highlight_overlay_lazy_incident_edges_render_draws_far_endpoint :10264). Attacked BLIND against the
  fix's done-when contract (same per-axis placement set as the in-memory twin; no new full-materialization; tests
  pin placement equality non-vacuously and stay lazy) and the ## Failure modes rubric before reading Amendment 26,
  Amendment 18, or my own WS J adversary post-impl section.
Dispositions held: yes. Amendment 26 implemented the exact push of my WS J must-fix (Finding 1: teach the lazy
  branch of _incident_placement_ids to gather far endpoints via the WS J bounded frontier compute), Finding 2 (the
  shared node_placements-equality helper), and Finding 3 (a lazy incident render test) with NO scope balloon: no
  new user-facing surface, signature, or kwarg; the frontier primitive is reused, not re-invented; the subset stays
  a LazyEdgeSubset. The one accepted new cost (a bounded compute at hops=0 for a lazy INCIDENT selection) is exactly
  the faithful-over-cheap tradeoff Amendment 18 established and is disclosed in the subset/add_highlight note.
Concerns:
  - [low-confidence] The render test (test_highlight_overlay_lazy_incident_edges_render_draws_far_endpoint,
    hiveplot_test.py:10264) does NOT gate the bug: datashader silently NaN-drops the missing far-endpoint placement
    rather than crashing, so `assert edges_image is not None` (:10312) passes WITH OR WITHOUT the fix. Its docstring
    still claims "a successful raster confirms the far endpoint is placed rather than a KeyError / NaN on a missing
    placement" (:10273-10274), which overstates what the assertion proves. This is the exact symptom question I left
    open at low-confidence in my WS J Finding 3 — now RESOLVED against the "crash" reading: the implementation-log
    entry (plan line 2468) confirms the engineer reverted the source and observed the render silently NaN-drops, so
    the placement-equality assertions (:10185, :10251) are the sole real guard. The render test therefore earns its
    place as coverage of the lazy incident render PATH (it does exercise _curves_from_id_pairs over the gathered
    placements and pins found_lazy), but not as a bug gate. Its docstring claim is the only theater: it reads as
    though the raster is the guard when the placement equality is. Rubric: "Tests genuine, not vacuous" (the
    engineer's own flagged concern). Push: soften the render test's docstring to say the raster confirms the lazy
    incident overlay RENDERS through the real curve path (and that the placement-equality tests, not this raster,
    gate the far-endpoint-placement completeness); no assertion change needed. No downstream not-yet-run workstream
    bearing (WS I cut per Amendment 25; WS J is terminal).
  - [low-confidence] Two of the four _assert_node_placements_match call sites do not exercise the fix: the subset
    call site (hiveplot_test.py:9181) runs under subset's locked closure="induced" (hiveplot.py:5693), and the
    induced-closure highlight call site (:10396) likewise keeps no far endpoints, so _incident_placement_ids returns
    placement_ids == selected_ids and the lazy branch is never reached. On those two, the placement-equality
    assertion is redundant with the _surviving_node_ids assertion (no far endpoints to add), so it is harmless but
    not load-bearing. The two INCIDENT call sites (:10185 edges-expression default-incident, :10251 nodes+hops
    default-incident) are the genuine gates and both are non-vacuous: a small seed on the 60-node / 100-edge fixture
    leaves the in-memory twin's placement set strictly larger than its surviving-node set (real far endpoints), and
    reverting the lazy branch drops them, so the ID-set comparison fails — confirmed load-bearing by the engineer's
    revert check (plan line 2468). Rubric: "Tests genuine, not vacuous." Push: none required; noting only that the
    fix's real completeness gates are the two incident twin-tests, not all four call sites. No downstream bearing.
Angles that came back clean (worked and shown, not manufactured):
  - Same per-axis placement set (done-when 1): the lazy branch feeds ids.filtered_native() (which applies the
    subset's OWN stored predicate, frames.py:509) to lazy_edge_frontier_endpoint_ids with selected_list as the
    frontier, returning the union of BOTH endpoint columns of edges touching the selection (frames.py:285-297) — the
    incident far endpoints, exactly mirroring the in-memory branch's np.isin over the stored id pairs
    (hiveplot.py:5817-5821). Ordering is correct: _incident_placement_ids runs at hiveplot.py:5750 BEFORE
    _restrict_edges_to_node_ids narrows the predicate (:5783), so the frontier reads the un-restricted (parent /
    base) membership, not an already-incident-narrowed one — no under-inclusion (a far endpoint the in-memory twin
    keeps) and no over-inclusion (the touching-edge set is pinned equal by the co-located _materialized_edge_id_set
    assertions, so no extra far endpoint sneaks in). Selection-side vs far-side split is right. Clean.
  - Genuinely bounded (done-when 2): filtered_native() returns a LAZY native frame (nothing computed); the only
    .collect() in the new path is inside lazy_edge_frontier_endpoint_ids, and it runs after .select([from, to])
    (frames.py:291), so exactly one bounded ID-columns-only gather realizes. Grep of the changed hiveplot.py shows
    NO .compute() and NO stray .collect() on the full frame in _incident_placement_ids (:5799-5822). Scales with the
    endpoint columns, not the row-materialized frame. Clean.
Cross-critic corroboration (recorded, NOT my blind finds): api-critic returned CLEAN on the Amendment 26 behavior
  change (per plan line 1918 / Amendment 26 origin). My WS J Findings 1-3 are all disposed by the shipped fix
  (Finding 1 source fix implemented as the exact pushed shape, Finding 2 shared placement-equality helper added,
  Finding 3 render test added with the symptom question resolved). The two low-confidence concerns above are
  polish, not gaps: the fix is correct, bounded, and load-bearingly tested on the two incident twin-tests.
```

## Workstreams

Sequenced core-first; every workstream individually shippable. Maintainer-set shape constraints honored: datashader highlight self-contained and revertible (F); condition shorthand self-contained (D); region selection last, stretch, revertible (I); HivePlotMatrix its own workstream (E). All workstreams: no process refs in shipped code/docs/notebooks; docstrings at 120 chars; exceptions from `src/hiveplotlib/exceptions/` with `stacklevel=3` for warnings; the feature's CHANGELOG entry lands with Workstream A and later workstreams do not add sibling entries for refinements debuting in the same unreleased version.

**Notebook sequencing (E before G/H, dispatch-ordering bearing):** the maintainer named HPM anecdotes "expected strongest." E's cross-panel anecdote fires FIRST as the premise probe, and its real dataset/story is the scouting ground for G/H (same real network, different mechanism), so a weak single-`HivePlot` subset/highlight story is caught against a known-good HPM baseline rather than in isolation. G and H may pick a different dataset if E's does not suit their mechanism, but they inherit E's scouted network as the default starting point. Disposes the adversary post-grill worth-discussing item on HPM sequencing. (This decides the "deliberately independent datasets" alternative against; the notebooks share E's scouting ground by default.)

### Workstream A: core `subset` on `HivePlot`

**Status:** not started
**Files:** `src/hiveplotlib/hiveplot.py`, `src/hiveplotlib/node.py`, `src/hiveplotlib/edges.py` (TODO deletions and any selection helpers), `tests/hiveplot_test.py`, `tests/node_test.py`, `tests/edges_test.py`, `CHANGELOG.md`
**Done when:**

- `hp.subset(...)` returns a new, fully independent `HivePlot`; mutating the child never touches the parent.
- Frozen geometry verified by test: surviving node placements and edge curves are array-equal to the parent's, and every child `Axis` carries the parent's `vmin` / `vmax` as pinned explicit values (`inferred_vmin` / `inferred_vmax` false), so a later recompute on the child cannot silently re-infer ranges from reduced data (fixed-layout-comparability hard constraint).
- Selection vocabulary: node IDs, node boolean mask, edge boolean mask (bare mask single-tag; dict-of-masks keyed by tag for multi-tag), edge axis pair. `hops` expansion (default 0) for both node- and edge-seeded selection.
- Composed-selection pipeline (pinned, docstring contract): resolve node seeds; resolve edge seeds (selected edges plus their endpoint nodes); union both into one node set; expand that whole set by `hops`; then apply closure. Subset locks induced (undrawable-safe by construction); highlight defaults incident. Stated principle in the docstring: incident is safe for highlight (paint over an already-drawn base, a far endpoint need not be lit) but banned for subset (subset removes nodes, so an incident edge to a dropped node would be undrawable). Maintainer reviews this pipeline closely at final review. Disposes the adversary's cold item 2.
- Union algebra documented (leads both `nodes=` and `edges=` docstrings, since the intersection-misread of a combined call is the risk): `nodes=` + `edges=` within one call = union; chained `.subset(...).subset(...)` = intersection. (api-critic verdict 5.)
- Mask-vs-IDs disambiguation, pinned and documented: a bool-dtype array/Series = mask; Python lists and everything else = IDs (docstring says "pass a numpy/pandas/polars boolean array for masks"). A scalar node ID is accepted so `hp.subset(nodes=17, hops=1)` is the clean ego call (precedent: `subset_node_collection_by_unique_ids` wraps a bare Hashable, node.py:614). (api-critic verdict 12.)
- Multi-tag mask dict, partial-dict semantics pinned and documented: an absent tag = not selected (dropped in subset, unlit in highlight); the docstring names and rejects the alternative (absent = kept whole). Dict values accept expressions as well as masks; a broadcast expression referencing a column missing from one tag frame raises a legible error naming the offending tag. (api-critic verdict 7.)
- Axis-pair directionality: `edges=("A", "B")` selects both directions by default, stated in the param docstring's first sentence (precedent: `reset_edges` bidirectional default). [optional, decide at implementation] accept a list of axis pairs (`edges=[("A","B"), ("A","C")]`, distinguishable from a single pair by element type); repeat-axis edges name literal axis IDs (e.g. `("A", "A_repeat")`), noted in the docstring. (api-critic verdict 13.)
- Ego-comment precision (docstring + carried into WS G notebook prose): `hops=1` with induced closure includes neighbor-neighbor edges (exact `nx.ego_graph` behavior), not spoke-edges-only. (api-critic verdict 14.)
- Induced closure consumes `Edges.relevant_edges` integer row-position arrays positionally (`df.iloc[positions]`), never boolean-mask reinterpretation.
- Lazy (Dask) edge frames, `hops=0` fully lazy: `hops=0` selection and highlight never force-materialize a `LazyEdgeSubset`; ID/axis-pair selection composes into the stored lazy membership predicate, and mask-based selection on lazy edge frames raises the legible in-memory error through the `_require_in_memory_edge_subsets` seam (hiveplot.py:2284). `hops>0` on lazy edge frames routes to that same legible in-memory error as the WS A floor (real lazy `hops>0` support is Workstream J, sequenced late and cuttable). Node data must fit in RAM: no Dask node subset, stated in the docstring as a real ceiling. Disposes the adversary's cold item 4 (floor half).
- "Cheap" pinned as a tested contract: `subset` slices already-computed geometry and must not call into curve construction (`construct_curves` / `_construct_edge_subset_curves`); pin copy-not-recompute mechanically by test (spy-on-construction vs same-run-ratio is the code-engineer's call). Guard: `subset` raises a legible error if invoked before the plot geometry is computed; this is distinct from an empty-but-valid selection, which returns a renderable empty plot (see next bullet), not a raise. Disposes the adversary's cold item 3.
- Empty-but-valid selection is renderable, not an error: an empty subset returns a valid, renderable `HivePlot` (a GIF threshold sweep past the data's max must not raise mid-loop); test it. (api-critic journey item 9.)
- Positional-vs-label tests: ID and mask selection verified on non-RangeIndex pandas node/edge frames and on at least one non-pandas library (inherited failure mode).
- Child `edge_coverage`: **deferred pending the sibling edge-coverage-and-gotchas plan** (that method is not on branch 53). The independence this line depends on is already met by construction (the child holds its own filtered edge frames), so the child computes its own coverage with no special-casing once `edge_coverage` lands; when edge-coverage ships on this base, that plan's done-when re-verifies this line and lands the covering test. Not unmet by Workstream A. (Amendment 15; adversary post-impl noted-only, code-engineer self-flag.)
- The four TODOs at node.py:170-171 and edges.py:143-144 are deleted; repo-wide grep for `data_subset` / `data_highlight` is clean.
- Round-trip contract holds: child `nodes.data` / edge frames come back in the user's dataframe library.
- Docstrings document copy semantics, the transient two-copies worst case, and the induced-closure asymmetry rationale; API pages render in `make docs`.
- 100% coverage with correct optional-backend markers; per ADR 0002, the "cheap" claim means no rasterization / curve-construction kernel changes; if any such path is touched, the equivalence wall and same-run ratio gates apply.

### Workstream B: highlight registry + matplotlib overlay

**Status:** not started
**Files:** `src/hiveplotlib/hiveplot.py`, `src/hiveplotlib/viz/base.py`, `src/hiveplotlib/viz/matplotlib.py`, matching tests
**Done when:**

- The registry lands on `HivePlot`, not `BaseHivePlot`: granting P2CP highlights later by moving a method down the hierarchy is non-breaking, while guarding a base-class surface against P2CP now is a permanent cost. Disposes the adversary's cold item 5.
- Register / update / remove / inspect by key: `add_highlight` (returns auto-generated or user-supplied key; collision handling warns via the house exceptions module, using the deprecation-policy plan's warning categories if that plan has shipped), `update_highlight`, `remove_highlight`, `highlights` read-only property.
- `update_highlight` accepts the full `add_highlight` kwarg set (selection, not just style), replacing only the named fields, so the GIF loop can re-aim one key per frame without remove+re-add churn. (api-critic journey item 10.)
- `highlights` property returns a small named record per highlight (the EdgeCoverage top-level re-export precedent), not a raw dict-of-dicts. (api-critic journey item 11.)
- One call may combine `nodes=` and `edges=` into one group (one key, one color); multiple simultaneous highlights work; `closure="incident"` default with `"induced"` option; `hops` supported.
- Default accent palette (5-6 deliberate colors, distinct from muted defaults, assigned first-unused) lives where viz-quality-bar governs it; per-highlight color override respected.
- Highlight styling beyond `color=`: `node_kwargs=` / `edge_kwargs=` passthrough on add/update (likely near-free since the overlay renders through existing viz machinery) is shipped, or a named deferral is recorded here. Adjacent, decide at implementation: a clear-all affordance (zero-arg `remove_highlight()` or a separate `reset_highlights()`). (api-critic journey item 15.)
- matplotlib `hive_plot_viz` / `edge_viz` / `node_viz` draw registered highlights as an overlay layer strictly above the base layer; base layer output without highlights is unchanged (regression test), edge-kwarg hierarchy contract untouched, no new conflict-warning cases; double-plotting under overlays documented as accepted.
- Render-time semantics pinned in one docstring sentence: highlights render at viz-call time; `update`/`remove` affect subsequent renders only, not already-drawn figures. (api-critic journey item 7 / adversary cold item 7.)
- Shared selection/closure resolution reuses Workstream A's machinery (one implementation of closure semantics, not two).
- Highlight state survives `copy()`; `subset()` interplay defined and tested (a child inherits no highlights by default; the grill confirmed this, since a partially-surviving selection would be a confusing half-lit artifact; document the choice).
- P2CP guard is a cheap "P2CP still renders after this lands" regression, not a heavy non-exposure test (downgraded because the registry lands on `HivePlot`, not `BaseHivePlot`).
- 100% coverage; docstrings lead with the user's word.

### Workstream C: overlay propagation to bokeh / holoviews / plotly

**Status:** not started
**Files:** `src/hiveplotlib/viz/bokeh.py`, `src/hiveplotlib/viz/holoviews.py`, `src/hiveplotlib/viz/plotly.py`, `src/hiveplotlib/viz/base.py` (shared prep), matching tests
**Done when:**

- The three vector backends render registered highlights above the base layer with the same visual contract as matplotlib; per-backend code is kwarg translation over shared prep in `viz/base.py`, not re-implementation.
- Backend parity test: same registered highlights produce overlay draw calls on every backend; optional-backend markers applied.
- P2CP viz paths untouched (guard against accidental breakage only).

### Workstream D: condition shorthand (narwhals expressions)

**Status:** not started
**Files:** `src/hiveplotlib/hiveplot.py` and/or selection helpers, `src/hiveplotlib/frames.py` if the boundary needs a hook, matching tests
**Done when:**

- Everywhere selection is accepted (subset and highlight), `nodes=` / `edges=` accept a narwhals expression (e.g. `nw.col("weight") > 5`); expression-in-parameter is final and a separate `where=` is rejected (api-critic verdict 6).
- Docstrings show the mask form and the expression form side by side; the mask path stays pandas-native so no one must learn narwhals to use the feature. (api-critic verdict 6.)
- Expressions filter on variables not used for partition or sorting (tested explicitly).
- Evaluates across pandas / polars / cuDF via narwhals dispatch. On lazy Dask edge frames, edge-expression selection is FAITHFUL: it matches the in-memory induced/incident-closure result (subset = induced, highlight = incident), computed via a bounded endpoint-ID compute (compute the unique endpoint node IDs of the expression-matching edges — which fit in RAM by the node ceiling — seed the node set with them, then compose the closure membership predicate into the lazy predicate) and NEVER full-materializes the `LazyEdgeSubset`. A concrete boolean mask on a lazy edge frame still errors (a pre-computed per-row array cannot be deferred); node-expression selection stays lazy. (Amendment 18: the shipped literal-filter composition was a semantic-divergence bug — a strictly narrower subgraph than in-memory — corrected to this faithful contract; the approximate/literal cheap mode is deferred, non-breaking to add later.)
- Positional-vs-label obligation extends to expression selection on non-RangeIndex frames.
- Hard dependency honored: requires !36's WS4 narwhals input boundary (merged by the execution gate).

### Workstream E: HivePlotMatrix broadcast

**Status:** not started
**Files:** `src/hiveplotlib/hiveplot_matrix.py`, matching tests
**Done when:**

- `subset` and the highlight family exist on `HivePlotMatrix` with the same name and shape as on `HivePlot` (mirroring precedent: `update_all_edge_plotting_keyword_arguments`, hiveplot_matrix.py:950), broadcasting one selection to every panel; `hpm.subset(...)` returns a new `HivePlotMatrix` with per-panel geometry frozen.
- Headline story documented: cross-panel comparison, one call shows where a group behaves uniquely vs not; docstring notes `unify_axes=True` keeps subset panels comparable, complementary to frozen geometry.
- Premise probe (fires before G/H): E's HPM anecdote lands the strongest artifact on a real dataset, and that dataset/story becomes the scouting ground for the single-`HivePlot` subset (G) and highlight (H) notebooks. See the notebook-sequencing note atop Workstreams. (adversary post-grill worth-discussing.)
- api-critic post-impl pass runs even though this is a mechanical propagation (house rule).

### Workstream F: datashader highlight guard (raise-and-point) + subset verification + side-by-side docs recipe (self-contained, revertible)

**Status:** not started
**Files:** `src/hiveplotlib/viz/datashader.py`, `src/hiveplotlib/exceptions/hive_plot.py`, matching tests; a docs notebook cell (placement below)
**Done when:**

- Open design item RESOLVED (spike + maintainer sign-off, 2026-07-07; see Amendment 20): **do NOT ship a datashader highlight overlay.** Both candidate designs failed (Option A accent-overlay raster washes out structurally at datashader scale; Option B separate panel is redundant with today's `subset()` + datashade). The decided shape is a raise-and-point guard, not an overlay implementation. This done-when is closed by the recorded decision.
- `datashade_hive_plot_mpl` (and `datashade_edges_mpl` / `datashade_nodes_mpl` as applicable, since a caller passes a `HivePlot` to each) RAISE a legible error (a new exception from `src/hiveplotlib/exceptions/hive_plot.py`, not a warning) when the passed `HivePlot` carries registered highlights (`instance.highlights` non-empty). The message points the user to `subset(...)` for a datashaded drill-down (noting `subset` carries the same selection vocabulary as highlight, including `hops`) and/or a non-datashader backend (matplotlib / bokeh / plotly) for the overlay. Datashader markers + a test per entry point asserting the raise on a highlights-registered `HivePlot`. A `HivePlot` with no highlights datashades unchanged (regression test).
- Subset-on-datashader verified by test in this workstream: a subset child (a real `HivePlot` with its own frozen geometry) renders through the datashader route with zero new per-backend code, including the Dask-lazy edge route (nothing special needed is the claim; test it). This is the affirmative half the guard steers users toward.
- Side-by-side docs recipe added: the "full | subset(selection)" datashader layout (two datashade calls in one figure, the maintainer's useful pattern) as a **documentation pattern, not new API**. Placement: a cell in the existing `examples/datashader.ipynb` (the datashader home) unless WS H's highlight notebook is the stronger narrative host, in which case it lands there (dispatcher's call at authoring; whichever, the recipe is written once). Runs green under `make test-nb`; editorial-critic and viz-critic post-impl passes cover the added cell.
- Highlight remains a matplotlib / bokeh / plotly feature by design (WS B/C); the raised error documents that datashader is deliberately excluded.
- ADR 0002 gates: the error-guard is a pre-render check that touches no rasterization / curve-construction kernel, so the equivalence-wall / same-run-ratio gates are not expected to fire; they still apply if any rasterization path is in fact changed.
- Ships as its own revertible commit.
- api-critic post-impl pass (already enrolled in the shared Pending list) runs on the raised error surface (message + which entry points guard) and the docs recipe.

### Workstream G: subset gallery notebook

**Status:** not started
**Files:** `examples/<subset notebook>.ipynb` (name set at authoring), `docs/source/_llms/llms.txt`
**Done when:**

- Scout FIRST, author second: before authoring, pick a REAL dataset chosen for a known-interesting SUBSET story (the synthetic `example_hive_plot` used in the API examples has no real structure to discover and is not an acceptable notebook subject). The anecdote is scouted, not hoped for. (adversary post-grill must-fix; the maintainer's primary failure mode is late anecdote discovery.)
- "Story didn't land" path: if the scouted dataset does not yield a genuinely interesting standalone subset anecdote, route to orchestrator amend-plan (reconsider scope / dataset) rather than silently shipping a dull notebook. (adversary post-grill must-fix.)
- One gallery notebook telling the drill-down story (plot, spot the clump, subset, keep working), including the ego-zoom and axis-pair moves and the frozen-geometry comparability point; practices, not encodes, the emphasis/de-emphasis base-styling move where relevant.
- Notebook prose is precise on the pinned semantics that trip users: the union algebra (`nodes=`+`edges=` in one call = union; chained calls = intersection) and ego precision (`hops=1` + induced includes neighbor-neighbor edges, not spoke-edges-only). (api-critic verdicts 5, 14.)
- llms.txt gets a feature-demo entry (new capability clears the consequential bar; goes in `## Optional`).
- Runs green under `make test-nb`; editorial-critic and viz-critic post-impl passes.

### Workstream H: highlight gallery notebook + GIF-sweep gate

**Status:** not started
**Files:** `examples/<highlight notebook>.ipynb`, `docs/source/_llms/llms.txt`, possibly a blog notebook under `docs/`
**Done when:**

- Scout FIRST, author second: before authoring, pick a REAL dataset chosen for a known-interesting HIGHLIGHT story (the synthetic `example_hive_plot` has no real structure to discover and is not an acceptable notebook subject). The anecdote is scouted, not hoped for. (adversary post-grill must-fix.)
- "Story didn't land" path: if the scouted dataset does not yield a genuinely interesting standalone highlight anecdote, route to orchestrator amend-plan (reconsider scope / dataset) rather than silently shipping a dull notebook. (adversary post-grill must-fix.)
- One gallery notebook telling the emphasize-in-context story: multiple named highlights, restyle/remove by key, incident-vs-induced closure, manual base de-emphasis (dark gray, thinner edges) as the practiced move.
- GIF-sweep gate decided and recorded here: the sweep-to-GIF pattern ships in-docs only if CI-cheap under `make test-nb`; otherwise it becomes a blog post. Either way the pattern is written once.
- llms.txt entry; green under `make test-nb`; editorial-critic and viz-critic post-impl passes.

### Workstream I: region selection (final stretch, revertible, cuttable)

**Status:** CUT (deferred) — cut per Amendment 25 (placement-select is a value threshold in disguise, redundant with shipped WS D; the real "select what I see" affordance is the spatial lasso, deferred to the interactive-explorer track). Block kept as the record of the considered-and-cut decision; done-when below is the original spec, not live work.
**Files:** `src/hiveplotlib/hiveplot.py`, matching tests, doc touch-ups
**Done when:**

- Justification (only non-redundant residual): selection by normalized PLACEMENT along an axis. Value ranges are already expressible once WS D ships (`nw.col(...).is_between(...)` unioned with an axis pair); placement-position selection is the genuinely new surface and is I's reason to exist. Disposes the adversary's cold item 6.
- Select by region: an axis pair plus a placement range per axis ("that patch between A and B"), feeding the same subset/highlight machinery; exact parameter shape proposed to api-critic before implementation.
- Ships as its own revertible commit; **explicitly allowed to be cut** if it doesn't pan out (record the cut as a deferred follow-up, not a silent drop).

### Workstream J: lazy `hops>0` support on Dask edge frames (self-contained, revertible, sequenced late, cuttable)

**Status:** complete
**Files:** `src/hiveplotlib/hiveplot.py` and/or selection helpers, `src/hiveplotlib/frames.py` if the lazy route needs a hook, matching tests
**Done when:**

- Real lazy `hops>0` support attempted: per-hop `is_in` / `unique` / `compute` over the endpoint-ID columns only, honoring the no-full-materialize invariant (the node set is in-RAM by the existing WS A ceiling, so only ID columns are scanned; the `LazyEdgeSubset` frame is never materialized). Replaces the WS A floor's in-memory error for `hops>0` on lazy frames when it lands.
- Save-before-cut rule: on trouble, first diagnose whether it is the same quickly-fixable Dask wart class the scaling sprint already beat (arrow-string, thread-safety, kernel serialization) before degrading back to the WS A `_require_in_memory_edge_subsets` error; only a genuine dead-end cuts this workstream, recorded as a deferred follow-up (not a silent drop).
- Ships as its own revertible commit; if cut, WS A's floor error stays as the shipped behavior for lazy `hops>0`.
- 100% coverage with the `@pytest.mark` markers the Dask route needs; ADR 0002 perf gates apply if any rasterization/curve path is touched (not expected; this is ID-column scanning).
- Disposes the adversary's cold item 4 (real-support half).

### API Critic — post-implementation review (Workstream J)

```
Done — the WS J api-critic post-impl block is filled under the shared `### API Critic — post-implementation review`
header (colocated with WS A-F), Status: clean. Verdict: the lazy hops>0 behavior change (error -> works via a
bounded per-hop endpoint-ID compute) is honest and consistent with the WS D faithful-lazy precedent; the compute
is documented on subset's .. note::, no leftover prose says lazy hops>0 raises, and the in-RAM node-set ceiling is
correctly extended to the hops-grown frontier. Two low-confidence notes (eager/k-computes timing, add_highlight
render-time-compute cross-reference), neither blocking.
```

## Plan amendments

Amendments folding the accumulated planning decisions (api-critic planning pass, adversary cold pre-grill challenge, full maintainer grill + failure-mode wave, adversary post-grill rubric-check) into the plan body. Each entry is the delta; the done-when / audit text it edited is authoritative. Append-only.

**Amendment 1 — Registry kept; justification recorded. [In-scope tweak]**
Delta: added the lead Default-justification entry for keeping the highlight registry (high-level-API rationale + the non-hand-composable tail: HPM one-call broadcast, datashader single-raster overlay, incident closure over an induced-only subset child). Rationale: grill wave 1 kept the registry and rejected the adversary's "hand-composable therefore delete" line. Touched: Default justifications (new first entry). Disposes adversary cold item 1.

**Amendment 2 — Registry lands on `HivePlot`; P2CP guard downgraded. [In-scope tweak]**
Delta: pinned the landing class to `HivePlot` (not `BaseHivePlot`) and downgraded the P2CP guard from a heavy non-exposure test to a cheap "P2CP still renders" regression. Rationale: moving a method down the hierarchy later is non-breaking; guarding a base-class surface now is permanent cost. Touched: WS B done-whens (landing-class bullet, P2CP-guard bullet). Disposes adversary cold item 5.

**Amendment 3 — Composed-selection pipeline pinned. [In-scope tweak]**
Delta: pinned the resolution order (node seeds → edge seeds + endpoints → union → `hops`-expand → closure; subset locks induced, highlight defaults incident) as a WS A done-when and docstring contract, with the incident-safe-for-highlight / banned-for-subset principle. Maintainer reviews it closely at final review. Rationale: unpinned composed semantics let each implementer pick a cell meaning (adjacent to the inherited wrong-data mode). Touched: WS A done-whens (pipeline bullet). Disposes adversary cold item 2.

**Amendment 4 — "Cheap" as a tested contract + pre-geometry guard + empty-is-renderable. [In-scope tweak]**
Delta: added WS A done-whens pinning copy-not-recompute mechanically by test (no calls into curve construction), a legible error when `subset` is invoked before geometry is computed, and (distinct) empty-but-valid selection returning a renderable plot rather than raising. Rationale: frozen-geometry array-equality alone can't detect a recompute-based implementation; the GIF sweep must not raise on an all-base frame. Touched: WS A done-whens (cheap-contract bullet, empty-selection bullet). Disposes adversary cold item 3 + api-critic journey item 9.

**Amendment 5 — Added Workstream J: lazy `hops>0` support. [Added workstream]**
Delta: WS A floor now routes `hops>0` on lazy edge frames to the in-memory error; new WS J attempts real support (per-hop `is_in`/`unique`/`compute` over ID columns only, no full materialize) with a save-before-cut diagnose-Dask-warts rule and a genuine-dead-end cut recorded as a deferred follow-up. Rationale: neighbor expansion needs the result of an edge scan per hop, which sits against the no-materialize invariant unless carved out. Touched: WS A done-whens (lazy bullet, floor half); new Workstream J block + its api-critic post-impl placeholder; shared api-critic Pending list; "Patterns this replaces" (J = net new). Disposes adversary cold item 4.

**Amendment 6 — Workstream I justification recorded. [In-scope tweak]**
Delta: added WS I done-when naming its only non-redundant residual (selection by normalized PLACEMENT along an axis; value ranges are already expressible via WS D expressions + an axis pair). Rationale: without this, I is ~90% expressible without itself and reads as an unjustified stretch. Touched: WS I done-whens (justification bullet). Disposes adversary cold item 6.

**Amendment 7 — Subset child inherits no highlights (confirmed). [In-scope tweak]**
Delta: annotated WS B's existing "child inherits no highlights" done-when to record that the grill confirmed it (a partially-surviving selection would be a confusing half-lit artifact). Rationale: record-only; the plan already leaned this way. Touched: WS B done-whens (copy/subset-interplay bullet).

**Amendment 8 — Notebooks scout a real dataset first, with a "story didn't land" path. [In-scope tweak]**
Delta: added two done-whens each to WS G and WS H: (a) scout and name a real dataset for a known-interesting subset (G) / highlight (H) story BEFORE authoring (synthetic `example_hive_plot` is not an acceptable subject); (b) if the story doesn't land, route to amend-plan rather than ship a dull notebook. Rationale: the maintainer's primary failure mode is late anecdote discovery; the plan's only prior guard was a final-review note, which is exactly that late failure. Touched: WS G done-whens (2), WS H done-whens (2). Disposes adversary post-grill must-fix.

**Amendment 9 — HPM (E) sequenced first as the premise probe. [In-scope tweak]**
Delta: recorded E-before-G/H ordering in a new notebook-sequencing note atop Workstreams and a WS E done-when: E's HPM anecdote fires first on a real dataset, and that dataset is the default scouting ground for G/H, so a weak single-`HivePlot` story is caught against a known-good HPM baseline. Decides the "deliberately independent datasets" alternative against. Rationale: HPM anecdotes are "expected strongest"; this has dispatch-ordering bearing. Touched: Workstreams sequencing note (new), WS E done-whens (premise-probe bullet). Disposes adversary post-grill worth-discussing item.

**Amendment 10 — Render-time highlight semantics pinned. [In-scope tweak]**
Delta: added a WS B done-when pinning (one docstring sentence) that highlights render at viz-call time; `update`/`remove` affect subsequent renders only, not already-drawn figures. Rationale: heads off the "I updated the highlight and my figure didn't change" report. Touched: WS B done-whens (render-time bullet). Adopts adversary cold item 7.

**Amendment 11 — Dispatch-time re-grep of pinned seams. [In-scope tweak]**
Delta: added an Execution-gate bullet to re-grep the branch-53-relative file:line pins at gate lift (hiveplot.py:1571 / :2284, edges.py:153-159, node.py:602, node.py:170-171, edges.py:143-144), since sibling plans may land first. Rationale: WS A must not build against stale offsets. Touched: Execution gate (re-verify bullet). Adopts adversary cold item 8. (Pins re-verified current on branch 53 at this amendment.)

**Amendment 12 — Naming audit reconciled to final; construction-time highlight rejected. [In-scope tweak]**
Delta: updated every Naming-audit entry from "Delegated"/"Open" to decided-with-reason citing api-critic verdicts 1-6 (`subset`, `hops`, `add/update/remove_highlight`+`highlights`, `closure="incident"|"induced"`, expression-in-parameter with `where=` rejected); flipped the API-usage-examples caveat to "names final"; recorded the rejection of the construction-time highlight kwarg in Goal non-goals (verdict 8). Rationale: the api-critic exercised the maintainer's delegated final say. Touched: Naming audit (5 entries), API usage examples caveat line, Goal non-goals (construction-time-highlight bullet).

**Amendment 13 — api-critic accepted obligations folded into done-whens. [In-scope tweak]**
Delta: landed the api-critic's accepted obligations as contract so post-impl passes hold them: union algebra (WS A + WS G notebook prose, verdict 5); expression + mask shown side by side, mask path pandas-native (WS D, verdict 6); multi-tag partial-dict semantics with legible per-tag error (WS A, verdict 7); `update_highlight` accepts selection not just style (WS B, journey 10); `highlights` as a named record type (WS B, journey 11); mask-vs-IDs disambiguation + scalar node ID (WS A, journey 12); axis-pair directionality + optional list-of-pairs + repeat-axis IDs (WS A, journey 13); ego-comment precision (WS A docstring + WS G prose, journey 14); highlight styling passthrough / clear-all affordance shipped-or-deferred (WS B, journey 15). Rationale: the api-critic's recurring-pattern note requires each accepted obligation to land as a done-when, not stay in prose. Touched: WS A done-whens (5 bullets), WS B done-whens (3 bullets), WS D done-whens (2 bullets), WS G done-whens (prose-precision bullet).

**Amendment 14 — Base-branch / execution-gate override: build off branch 53 now. [In-scope tweak]**
Delta: rewrote the Execution gate. The original "no code ships until MR !36 merges" gate is lifted; implementation is underway on worktree branch `59-subset-highlight` cut from `53-scale-hiveplotlib-to-larger-networks`, with a Draft MR "Closes #59" targeting branch 53 (retargeted to master after !36 merges; agents never merge). The dispatch-time re-grep is marked satisfied (ran and passed against branch 53's tip). Rationale: maintainer opted into autonomous execution for momentum (Alignment "Execution opt-in, 2026-07-07"); building off 53 satisfies the same three technical constraints the gate protected (scaling representation present, WS A avoids the `node.py`/`edges.py` overlap, WS D's narwhals dependency met). Residual risk the gate originally named: branch 53 is an unmerged Draft the maintainer may still revise, so WS A/D built on it may need a rebase when !36 lands; accepted cost. Touched: Execution gate (full rewrite). Preserved the mutual-sequencing note with the edge-coverage plan.

**Amendment 15 — Workstream A post-impl dispositions. [In-scope tweak]**
Delta: dispositions of the WS A post-impl gate battery (api-critic post-impl, adversary post-impl, code-engineer self-flags; all recorded in `### API Critic — post-implementation review` and `### Adversary post-impl`). Seven findings, six routed to a code-engineer follow-up, one deferred:

- **Deferred (done-when edited, not a code fix): child `edge_coverage`.** The WS A done-when assumed `edge_coverage` present, but it ships in the sibling edge-coverage-and-gotchas plan and is not on branch 53; the independence it needs is met by construction (the child holds its own filtered edges). Edited that done-when to read "deferred pending the sibling edge-coverage plan" so it stops reading as unmet. (adversary post-impl noted-only; code-engineer #1.)
- **Fix now — shared-resolver hardening (root fix, not batched; WS B/D/E reuse these resolvers): scalar/aggregating selection expression bypasses the length guard.** `_resolve_node_seed_ids` raises a raw `IndexError` outside its try/except on an aggregating node expression; `_edge_frame_mask` / `_resolve_edge_seed_endpoint_ids` silently ship the WRONG subgraph on an aggregating edge expression (the inherited "plausible picture of the wrong data" mode, made concrete). A selection expression must evaluate to a full-length boolean row mask; on a length mismatch, raise the same legible error a missing column already raises (from `src/hiveplotlib/exceptions/`, naming the offending tag on the edge path; `stacklevel=3` on any warning path), steering to a per-row (non-aggregating) expression. Both node and edge paths, plus tests. (adversary post-impl worth-discussing.)
- **Fix — `:raises:` omits `InvalidAxisNameError`** on the axis-pair path (real and tested, but the docstring's catchable-error surface is incomplete for a try/except author). Add the one `:raises InvalidAxisNameError:` line. (api-critic post-impl #1.)
- **Fix — `hops` has no positivity guard** (`hops=-1` silently behaves like `hops=0`). Guard `hops >= 0` with a legible error + test. (api-critic post-impl #3.)
- **Document (no warning, by design): unknown node IDs silently dropped; tuple `nodes=(0, 1)` read as one Hashable ID.** Consistent with the deliberate empty-but-renderable stance; adding a warning would collide with the empty-sweep intent, so no code change. Docstring clarifies both: a list/array selects multiple IDs, a tuple is one (tuple-valued) Hashable ID, and unknown IDs are dropped silently like an empty selection. (api-critic post-impl #2; adversary post-impl low-conf.)
- **Document: "exact nx.ego_graph" over-claims on directed graphs** (the expansion is an undirected neighborhood). Soften the docstring to "undirected neighborhood; equivalent to `networkx.ego_graph` on an undirected view." The load-bearing neighbor-neighbor-edges content is correct and stays. (adversary post-impl low-conf.)
- **Document: lazy axis-pair selection is over-inclusive vs the in-memory path** (seeds all nodes on the two axes; a consciously designed lazy-floor compromise, already tested). Add one docstring line naming the lazy-vs-in-memory asymmetry explicitly. (code-engineer #2; adversary post-impl low-conf.)

Rationale: fold the post-impl gate outcomes into contract so the follow-up is a defined code-engineer dispatch, not loose prose; the resolver length-guard is fixed at the root now because WS B/D/E reuse it. No downstream-workstream blocking (the resolver fix also hardens WS D's condition shorthand, but D is not yet run and this does not gate its dispatch). Touched: WS A done-whens (`edge_coverage` bullet). Does not change WS B–J.

**Amendment 16 — Workstream B post-impl dispositions (highlight registry + matplotlib overlay). [In-scope tweak]**
Delta: dispositions of the WS B post-impl gate battery (api-critic post-impl, viz-critic post-impl — recorded by this amendment since viz-critic is read-only, adversary post-impl). Eleven findings across the three critics: **eight route to one code-engineer follow-up** (six code fixes, one docstring-only pair, and one test-only add), **three are accepted / noted-only** (no code-engineer action). WS B's surface is unchanged; these harden and polish it, so no WS B done-when is edited. The shipped WS B artifact is preserved intact; the `_draw_overlay_nodes` fix (item 1) is done now, not deferred, because WS C's vector backends mirror this overlay path and the fix sets the contract C copies.

Fix-list (eight items, for the code-engineer follow-up):

- **[must-fix] (code + test) Incident overlay paints accent NODE dots on unselected neighbors.** `_draw_overlay_nodes` (viz/matplotlib.py:695-724) scatters every placed node; under incident closure the far-endpoint placements (kept so touching-edge curves draw) get accent dots too, so `add_highlight(nodes=[5])` lights node 5 plus a dot on every neighbor of 5 (a hub paints a large fraction of the node cloud). Contradicts the stated incident semantics (selected nodes + touching EDGES; Default justification line 100, CHANGELOG.rst:27). Fix: filter overlay nodes to the selected node IDs (`overlay.nodes.data[unique_id_column]`); far endpoints keep their edge curve, not an accent dot. Add a test asserting neighbor nodes are NOT accent-lit (existing tests miss it: the incident test asserts on node DATA, the accent-color test asserts all overlay scatters are red so extra red neighbor dots pass over the bug). WS C copies this contract. (adversary post-impl finding 1.)
- **[worth-discussing → code + test] Overlay reads as recolored, not spotlit.** Raise the overlay defaults so the accent dominates by weight+opacity, not hue alone: overlay edge linewidth ~2.0-2.5 and/or alpha ~0.9, overlay node size ~35-40 (base overlay currently inherits edge alpha=0.5 / base linewidth and node s=20 / alpha=0.8, identical to the base). Keep per-highlight `color=` / `node_kwargs=` / `edge_kwargs=` override precedence over these new defaults. (viz-critic findings 1 and 3.)
- **[worth-discussing → code + test] `key` is positional-6 in `add_highlight`.** `add_highlight([1,2,3], "clump")` silently binds "clump" to `edges=` (hiveplot.py:5874-5884). Make `key` keyword-only (`*,` before it) so the positional call is a legible TypeError; the record and `update_highlight` both put key first, so add is the odd one out and every call site already passes `key=` by name. (api-critic finding 1.)
- **[low-confidence → code + test] No `closure` validation.** `closure="induce"` (typo) is stored verbatim and silently falls through to the incident branch (hiveplot.py:5655), mis-narrowing with no signal. Validate `closure in {"incident", "induced"}` on both `add_highlight` and `update_highlight` (when not `_UNSET`) with a legible ValueError naming the two valid values. (api-critic finding 3.)
- **[low-confidence → code + test] `update_highlight(hops=None)` raises a raw TypeError** (`None < 0`, hiveplot.py:5995). Guard `hops is not _UNSET and hops is not None and hops < 0`, or scope the docstring's "None clears" to the selection fields only. (adversary post-impl finding 4.)
- **[worth-discussing → test only] Highlight mask selection on a non-RangeIndex frame is untested.** Highlight has ID and expression tests but no non-RangeIndex boolean-mask test, against the per-mechanism ID / mask / expression obligation. The resolver is shared with subset (which has the mask-on-shuffled-index test), so it is one test to close highlight's per-mechanism obligation. (adversary post-impl finding 3.)
- **[worth-discussing → docstring only] `highlights` record echoes the RAW selection.** The property and `Highlight` record return the mask/expression/ID list the user passed, not the resolved node/edge set drawn. One docstring line on the property (and/or `Highlight.nodes`/`edges`) stating the record echoes the selection you PASSED, not the resolved set, pointing at re-running `subset(...)` with the same selection to materialize the actual set. (api-critic finding 2.)
- **[low-confidence → docstring only] `add_highlight` docstring caveats.** Add a dense-overlay caveat (raise linewidth / lower base alpha / pre-thin, since the overlay has no density-aware default and no datashader path) and a note that the accent palette is tuned against the dark/black base (orange L=0.42 and sky blue L=0.41 collapse onto a darkgray de-emphasis base at L=0.40). (viz-critic findings 2 and 4.)

Accepted / noted-only (no code-engineer action):

- **Incident-lights-everything has no code guard — disposed by design.** The no-auto-dimming justification (line 104, maintainer "a mistake") stands; WS H's notebook practices manual base de-emphasis as the move, and its "story didn't land" route to amend-plan already covers a dull-because-everything-lit anecdote. Even after the item-1 fix limits accent DOTS to the selection, incident EDGES on a hub still light most of the edge set by design. Confirmed chosen, not overlooked. (adversary post-impl finding 2; viz-critic's "highlights everything on a hub" cleared-and-held.)
- **Collision-replace picks the next auto color while the replaced highlight still holds its slot** (hiveplot.py:5944 / 6063) — cosmetic only (never a lost highlight or wrong data; the key-collision path correctly warns-and-replaces). Accept as-is. (adversary post-impl finding 5.)
- **`add_highlight` uses plain `None` defaults while `update_highlight` uses `_UNSET`** — both individually correct and documented (add's `None` = "no seeds," update's `None` = "clear the field"); the asymmetry is inherent to update's replace-only-named-fields contract. Note only. (api-critic finding 4.)

Rationale: fold the WS B post-impl gate outcomes into contract so the follow-up is a single defined code-engineer dispatch, not loose prose across three critic blocks; record the viz-critic block (Job 1) since that critic is read-only. No downstream-workstream blocking of a not-yet-run stream, with one forward contract: item 1's `_draw_overlay_nodes` fix sets the overlay-node contract WS C's vector backends mirror, so it lands now rather than deferring. No WS B done-when is edited (the surface is unchanged; every finding hardens or polishes shipped behavior the done-whens already describe). Touched: `## Viz review` (WS B block recorded); no workstream done-whens changed.

**Amendment 17 — Workstream D post-impl dispositions (condition shorthand / narwhals expressions). [In-scope tweak]**
Delta: dispositions of the WS D post-impl gate battery (api-critic post-impl + adversary post-impl; no viz/editorial, WS D ships no figures or notebook). Four findings, all routed to one code-engineer follow-up: **three docstring-only** (fold into one edit) and **one test-only** hardening of the danger-zone lazy path. No WS D done-when is misstated: the shipped expression surface is correct (api-critic and adversary both hold the two danger-zone contracts, no-materialize + concrete-mask-errors), so this is doc/test hardening only, not a surface fix. The shipped WS D artifact is preserved intact.

Fix-list (four items, for the code-engineer follow-up):

- **[must-fix] (test only) Lazy Dask expression tests are soundness-only under a vacuity guard.** The three lazy-expression tests (hiveplot_test.py:8532-8538, :8993-8998, :10019-10023) assert only that surviving rows satisfy the predicate (`(materialized["weight"] > 5).all()` / `.isin(...).all()`), each inside `if len(materialized) > 0:`, and `found_lazy` only checks a `LazyEdgeSubset` record EXISTS, not that it carries rows. No test asserts completeness (no valid row is MISSING) or compares the lazy result to its in-memory twin. The lazy composition is the one genuinely net-new WS D path (the plan's named danger zone); a future predicate-composition regression (an AND/OR swap under induced, or `is_in` against the wrong column) that dropped valid rows would pass silently, since every SURVIVING row still satisfies the predicate. Fix: add a lazy-expression test that materializes the composed subset and asserts its surviving EDGE set (or at minimum row count) EQUALS the in-memory pandas twin's on the same seed + expression, and drop the `if len(materialized) > 0:` guard so an all-empty result fails rather than passes vacuously. Hardens the danger-zone test coverage; no shipped-behavior change, no downstream-workstream bearing. (adversary post-impl finding 1.)
- **[low-confidence → docstring only] plotly edge-kwarg doc name is wrong (self-contradicting shipped example).** The folded-in WS-C backend-native kwarg fix wrote plotly `edge_kwargs={"width": 2}` in the `add_highlight` docstring (hiveplot.py:6038-6039), but overlay `edge_kwargs` flow into the library's own `edge_viz`, and plotly's `edge_viz` spells edge width `line_width` (translating to plotly's raw `width` internally). The shipped test proves the right name: `viz_plotly_test.py:1147/1151` passes `{"line_width": 7}` and asserts `trace.line.width == 7`. A user copying the documented plotly example passes `{"width": 2}`, which never matches, silently keeping the default 2.5 width. The bokeh `{"line_width": 2}` and matplotlib `{"linewidth": 2}` examples are correct (both flow through their `edge_viz`); plotly is the lone wrong spelling. Fix: change the plotly edge example to `{"line_width": 2}`. api-critic rated this must-fix on self-contradiction grounds; it is docstring-only either way. (api-critic post-impl.)
- **[low-confidence → docstring only] `color=` wording is backend-parochial.** The `color` line on `add_highlight` (hiveplot.py:6026) still reads "Any `matplotlib`-recognized color overrides the default" now that highlights render on bokeh / plotly / holoviews too (the WS-C fold-in took the kwarg-name half and left the color half). Common named colors work across all four backends, so it rarely bites, but the sentence is now parochial in the same way the kwargs were. Fix: soften to "any color the active backend recognizes." Batch into the same edit as the plotly fix. (api-critic post-impl; the WS D engineer flagged the same in its taste-call.)
- **[low-confidence → docstring only] Lazy edge-expression node-set disclosure understated (delta on Amendment 15).** A lazy edge-expression subset keeps every node on the tag's spanning axes in the child's node DATA (extra isolated points), so the surviving NODE set is genuinely broader than the in-memory twin's precise incident endpoints, while the edge FILTERING is exact. The `subset` `.. note::` (hiveplot.py:5602-5605) discloses the broad SEED but frames it as a seed the expression collapses, reading as if the expression narrows the broadened set back down. This is a delta on the already-disposed Amendment 15 lazy-axis-pair asymmetry (same broad-seed family, extended from axis pairs to expressions). Fix: tighten that one docstring line to say the surviving NODE set (not just the seed) is broader on a lazy edge-expression subset. (adversary post-impl finding 2 / api-critic low-confidence.)

Rationale: fold the WS D post-impl gate outcomes into contract so the follow-up is a single defined code-engineer dispatch. The must-fix is a test-only hardening of the plan's named danger zone (the lazy composition path); the three docstring fixes are backend-accuracy and disclosure precision on shipped, correct behavior. No downstream-workstream bearing (WS D is a self-contained workstream; the test-only fix touches no not-yet-run stream, and the docstring fixes are local to `add_highlight` / `subset`). No WS D done-when is edited (the expression surface is correct; every finding hardens the test suite or sharpens a docstring the done-whens already describe). Touched: no workstream done-whens changed.

**Amendment 18 — Lazy Dask edge-expression semantics were WRONG; faithful bounded-endpoint-compute is the default, approximate flag deferred. [In-scope tweak]**

Origin: the WS D post-impl code-engineer (writing the Amendment-17 must-fix completeness test) hit `STATUS: BLOCKED` on a real semantic bug, and the maintainer made the design call (2026-07-07). This amendment supersedes a load-bearing premise of Amendment 17: that "the shipped expression surface is correct" and the lazy composition was only under-tested. It was not correct. The must-fix that Amendment 17 filed as a test-only vacuity fix is now a *correctness* pin (see the fix-list edit below); the docstring item that Amendment 17 filed as a disclosure-precision fix changes meaning (it now documents a performance note, not a wrong-subgraph asymmetry).

The bug (machine-verified, code-engineer-confirmed): the shipped lazy Dask edge-expression path implements DIFFERENT semantics than the in-memory contract. In-memory `subset(edges=nw.col("weight") > 5)` follows the pinned pipeline (Amendment 3): resolve the expression to the matching edges' endpoint NODES, union into the node set, then apply closure (subset = induced: keep every edge with BOTH endpoints in the final node set; highlight incident: either endpoint). The shipped lazy path instead ANDs the expression into the edge predicate as a LITERAL edge filter (`_restrict_lazy_subset(extra_predicate=)` over a broad axis-spanning seed), keeping only edges that literally satisfy the expression: a strictly narrower subgraph. Machine-verified: in-memory keeps 54 edges, lazy keeps 40, same call + data. This is the inherited "plausible picture of the wrong data" failure mode made concrete (same code + data, different subgraph depending on Dask-vs-pandas edges). Node-expression selection is unaffected (nodes are in RAM); only EDGE-expression on a lazy frame diverged. Note this falsifies the Amendment 17 adversary block's "Composition correctness holds ... the effective lazy predicate reduces to `original & extra_predicate`, matching the in-memory result" reasoning (lines 937-940): the broad seed does NOT make `is_in` trivially true for every in-tag edge under induced closure, so the literal filter and the induced-closure result diverge.

Maintainer decision (2026-07-07): **faithful by default via a bounded endpoint-compute; the approximate/literal-filter cheap mode is DEFERRED (not shipped).**

- **Faithful default.** Lazy edge-expression selection MUST match the in-memory induced/incident-closure result. The feasible faithful implementation: compute the endpoint node IDs of the expression-matching edges (a bounded Dask compute over the from/to/expression columns of the matching edges, reduced to UNIQUE node IDs, which fit in RAM by the existing WS A node ceiling), seed the node set with those IDs, then compose the induced-closure membership predicate (`is_in(from, seed) & is_in(to, seed)`) into the lazy predicate. This never materializes the full `LazyEdgeSubset` (only the bounded ID compute runs; the final subset stays lazy), so it honors the scaling invariant (scaling-large-networks constraint 2, `## Prior ADRs`). Accepted cost: a real-but-bounded Dask compute on an edge-expression selection (a change from the shipped fully-deferred literal filter, which was faithful-to-nothing).
- **Approximate mode deferred.** The approximate/literal-filter cheap mode is NOT shipped as an opt-in flag now. Adding it later is non-breaking (it would default to the faithful behavior); speculative config now cuts against the library's no-ceremony ethos (ADR 0001 GraphMetricsSpec-deferral precedent, `## Prior ADRs`). Revisit only if the bounded compute proves too slow in practice. Recorded as a deferred follow-up, not a silent drop.
- **Consistency signal for WS J (pointer only, no WS J done-when edited).** This establishes the bounded-endpoint-compute pattern for faithful lazy operations. WS J's lazy `hops>0` (currently routed to the WS A floor's in-memory error) should aim for the SAME faithful bounded-compute approach when it is dispatched, not settle for the floor error. WS J has not run, so its done-whens are unchanged beyond this note; the aim is a pointer for the WS J dispatch, not a re-scope.

Amendment 17 fix-list edits (the WS D post-impl follow-up is now a correctness fix, not doc/test polish):

- The **[must-fix] lazy-expression test** changes from a vacuity-guard removal to the **correctness pin**: assert the lazy-materialized surviving EDGE set EQUALS the in-memory pandas twin on the same seed + expression (it should now be EQUAL once the faithful bounded-compute lands, where before the buggy literal filter it was strictly narrower), and drop the `if len(materialized) > 0:` vacuity guards. The test now pins the faithful semantics, not just non-emptiness.
- The **item-4 docstring note** (Amendment 17's `[low-confidence → docstring only]` lazy edge-expression disclosure; the `subset` `.. note::` at hiveplot.py:5602-5605) changes MEANING: it should now document that a lazy edge-expression triggers a **bounded endpoint compute** to stay faithful (a PERFORMANCE note), NOT a wrong-subgraph / broader-node-set asymmetry. That asymmetry is GONE for edge-expression once the path is faithful. The separate lazy AXIS-PAIR over-inclusiveness from Amendment 15 remains as its own disclosure (a different broad-seed path, not touched by this fix).
- Amendment 17's two other docstring fixes (item-2 plotly `line_width` kwarg name, item-3 backend-parochial `color=` wording) are ALREADY applied in the worktree and are unaffected by this amendment; leave them.

Rationale: the shipped lazy path was a semantic-divergence bug (a wrong subgraph), not a coverage gap; the maintainer chose faithful-by-default over the cheap approximate filter, accepting a bounded Dask compute, and deferred the approximate flag as speculative config. Superseding Amendment 17's "surface is correct" premise for the must-fix and re-pointing its item-4 disclosure keeps the plan-as-shared-memory honest before the fix lands. WS D's lazy done-when IS edited by this amendment (the only workstream done-when this cycle touches; see the WS D block), because the prior done-when's "composes into the lazy membership predicate" wording describes the buggy literal-filter behavior, not the faithful contract. No downstream-workstream blocking (WS J gets a pointer only; its dispatch is unchanged). Touched: WS D done-whens (lazy bullet reworded to the faithful contract); Amendment 17 fix-list re-pointed (must-fix → correctness pin, item-4 → performance note); `## Patterns this replaces` (the shipped lazy literal-filter is replaced by the faithful bounded-compute, WS D's own net-new path — recorded there).

**Amendment 19 — Workstream E post-impl dispositions (HivePlotMatrix broadcast). [In-scope tweak]**
Delta: dispositions of the WS E post-impl gate battery (api-critic post-impl + adversary post-impl, both blind; no viz/editorial, E ships no figure/notebook). Four findings, all routed to one code-engineer follow-up. The fix is docs + legibility + test hardening: **no shipped-behavior change beyond one legible `(row, col)`-naming error on the `subset` broadcast**. No WS E done-when is misstated (the broadcast mirrors the WS A/B/D panel methods faithfully, adds no selection/closure logic, and the subset-vs-highlight raise-timing asymmetry below is the delegated single-`HivePlot` semantics, correct by design). The shipped WS E artifact is preserved intact.

Design call settled here (so the fix does not over-correct): the **subset-raises-fast / highlight-raises-at-plot-time asymmetry on a bad axis pair is CONSISTENT-BY-DESIGN, not a defect.** `subset` computes geometry at call time, so a bad axis pair raises immediately and transactionally on the deepcopy (the parent matrix is untouched); `add_highlight` only registers the record and resolves the selection at plot time (the settled render-time semantics, Amendment 10), so on a heterogeneous matrix it registers on every panel then raises at `plot()` — exactly as a single `HivePlot.add_highlight` with a bad axis does. So do **NOT** add registration-time axis validation to the matrix `add_highlight` (that would make HPM stricter than `HivePlot`, breaking the same-name-same-shape mirror). The fix is legibility and docs, not a behavior change.

Fix-list (four items, for the code-engineer follow-up):

- **[must-fix — converged: api-critic must-fix + adversary raised-to-must-fix] Axis-pair broadcast on a heterogeneous `from_partition` matrix is undocumented and crashes without matrix context.** On a `from_partition` matrix (panels carry different axes by construction) `hpm.subset(edges=("A","B"))` / `hpm.add_highlight(edges=("A","B"))` raise `InvalidAxisNameError` on panels lacking those axes, but the matrix mirror dropped the parent's `:raises:` block (parent `HivePlot.subset` carries `:raises InvalidAxisNameError:` at hiveplot.py:5617, landed by Amendment 15), carries no homogeneous-axis caveat, and the error names one panel's axis list with no `(row, col)`, so the user cannot tell which panel crashed or why the same call worked on a `from_variable_sweep` / `from_tags` matrix. Fix (floor + nicer, per both critics): (a) add `:raises InvalidAxisNameError:` to both matrix `subset` and `add_highlight` docstrings, and fold `:raises SubsetBeforeBuildError:` into `subset`'s block (the mirror also dropped that parent contract, reachable via a hand-built unbuilt generic matrix); (b) add a homogeneous-axis caveat sentence to both — an axis-pair edge selection assumes every populated panel carries both named axes (as `from_variable_sweep` / `from_tags` give); on a heterogeneous `from_partition` matrix it raises, so scope axis-pair selection to a homogeneous matrix or select by node ID / mask / expression; (c) make the error name the offending `(row, col)` panel where feasible — for `subset` at minimum (call-time, wrap the broadcast loop and re-raise with panel context); for `add_highlight` the plot-time crash is heavier to attribute, so the docstring caveat carries it if wrapping the HPM plot path is disproportionate (code-engineer's judgment); (d) add a `from_partition` axis-pair test pinning the chosen behavior (subset raises transactionally, matrix untouched; add_highlight registers then plot raises) — no existing test exercises this, all edge-selection tests use the deliberately homogeneous `sweep_matrix` fixture. **Downstream bearing (routes now, not batched):** G/H scout from E's dataset (Amendment 9); an axis-pair highlight on a `from_partition` matrix would surface as a render crash in a built notebook, so this bears on the not-yet-run G/H streams. (api-critic post-impl must-fix; adversary post-impl finding 1, raised from worth-discussing to must-fix on the add_highlight deferred-crash-reported-as-success facet.)
- **[worth-discussing → docstring only] Collapsed `highlights` view over-claims unconditional cross-panel coherence.** The property builds with `setdefault` (first populated panel wins) and its docstring asserts each record is "identical across panels because add_highlight broadcasts" (hiveplot_matrix.py:1082-1084), but `__getitem__` exposes live panels (hiveplot_matrix.py:903-911), so `hpm[0,0].update_highlight(key, color="red")` diverges one panel while the view still reports the first. Fix: condition the docstring's "identical across panels" claim on using the matrix-level `add_highlight` / `update_highlight` / `remove_highlight` — reaching into a single panel via `hpm[r, c]` and editing its highlight directly can diverge a panel, and this view shows only the first panel's record. (Distinct from Amendment 16's WS B record-echoes-raw-selection disposition: that is what a single record shows; this is the matrix collapsing divergent records into one.) (adversary post-impl finding 3; api-critic post-impl worth-discussing, same `__getitem__` mechanism.)
- **[worth-discussing → docstring only] `_next_highlight_color` docstring claims a matrix-wide color seeding it does not implement.** The docstring says it "seeds it from the colors already used across the whole matrix" (hiveplot_matrix.py:1282-1283) but the body just returns the FIRST populated panel's `_next_highlight_color()` and returns immediately (hiveplot_matrix.py:1287-1289) — no cross-panel union. Harmless in the lockstep broadcast-only flow (every panel's color set is identical, so the first panel's pick is correct for all), but the claim is false as written. Fix: rewrite the docstring to state the truth — reads the first populated panel; correct because the broadcast keeps every panel's color set in lockstep — do not claim a union it does not compute. (A real cross-panel union is only load-bearing if the `__getitem__` divergence path above is treated as supported; it is not, so the docstring is the fix.) (adversary post-impl finding — genuinely new from the blind pass, matrix-specific, not in the api-critic block.)
- **[low-confidence → test only] `test_subset_broadcasts_same_selection_to_every_panel` under-guards its named invariant.** It asserts only `surviving <= selected` (hiveplot_matrix_test.py:4879-4887), which passes even if a panel got a narrower selection (two panels each keeping a different subset of `selected` both satisfy the bound; per-panel survivor divergence is legitimate post-closure, so `<= selected` is the honest bound but the test NAME over-claims). Fix: strengthen to assert the same RESOLVED seed set (pre-closure) reached every panel, or rename to the weaker invariant it actually checks. Correct-by-construction today (`subset` passes identical `nodes=`/`edges=`/`hops=` to every panel at hiveplot_matrix.py:1072; the shared resolver is already positional-vs-label tested in WS A), so this is a test-strength gap, not a shipped-behavior bug. (adversary post-impl finding, low-confidence.)

Rationale: fold the WS E post-impl gate outcomes into contract so the follow-up is a single defined code-engineer dispatch. The must-fix converged independently from both blind critics; it is a docs + legibility fix (the one code change is a `(row, col)`-naming wrapper on the `subset` broadcast raise), not a behavior change — the subset-vs-highlight raise-timing asymmetry is the delegated single-`HivePlot` semantics settled by Amendment 10, deliberately preserved so the matrix stays a faithful mirror. The three lesser items are two docstring-honesty fixes and one test-strength fix on shipped, correct behavior. Routes now rather than batching to plan-end because of the G/H downstream bearing (a built notebook could crash on an axis-pair highlight over a `from_partition` matrix). No WS E done-when is edited (the broadcast surface is correct and mirrors the parent; every finding hardens legibility, a docstring, or a test the done-whens already describe). Touched: `## Viz review` n/a (no viz-critic block — E ships no figure); no workstream done-whens changed.

**Amendment 20 — WS F open design item resolved: no datashader highlight overlay; raise-and-point guard + subset verification + side-by-side docs recipe. [In-scope tweak]**

Origin: the WS F open-design-item done-when required a spike plus maintainer sign-off before implementing the datashader highlight. A throwaway spike prototyped both candidate designs at datashader scale; the maintainer made the call (2026-07-07, signed off). This amendment records the spike outcome and RESOLVES that open-design-item done-when (it is no longer open), and rewrites the WS F done-whens from "implement the chosen overlay" to the decided raise-and-point shape.

Spike outcome (both candidate designs rejected):

- **Option A (accent overlay raster z-ordered on top of the base raster) fails STRUCTURALLY, not by tuning.** The accent (highlighted) edges are a subset of the base edges living in the same dense angular corridors, so once datashader bundles edges into a raster the accent merges into the same indistinguishable dense field and fills the panel. Verified even with the base crushed to flat gray and the accent maxed, and even for a sparse 120-of-4000-node selection: only the sparse axis NODES read as accent; the edges (most of what datashader draws) wash out. It composites cleanly, so it would pass a smoke test and look done, while misleading at the one scale datashader exists for — a live instance of this plan's own inherited "highlights everything by highlighting everything" / "plausible picture of the wrong data" failure mode.
- **Option B (separate panel) reads fine but is REDUNDANT** — it is literally `subset()` + datashade side by side, already expressible today with zero new code.

Maintainer decision (2026-07-07, signed off) — do NOT ship a datashader highlight overlay. Instead:

1. **Raise-and-point guard (a clear "yell," not a silent warn).** `datashade_hive_plot_mpl` (and `datashade_edges_mpl` / `datashade_nodes_mpl` as applicable) RAISE a legible error (a new exception from `src/hiveplotlib/exceptions/hive_plot.py`) when the passed `HivePlot` carries registered highlights, pointing the user to `subset(...)` for a datashaded drill-down (same selection vocabulary as highlight, including `hops`) and/or a non-datashader backend (matplotlib / bokeh / plotly) for the overlay.
2. **Keep the subset-on-datashader verification test** (a subset child renders through the datashader route with zero new per-backend code, including the Dask-lazy edge route) — the affirmative half the guard steers toward.
3. **Add a side-by-side "full | subset(selection)" datashader docs recipe** (the maintainer found this pattern useful) — a documentation pattern, NOT new API. Placement: a cell in `examples/datashader.ipynb` unless WS H's highlight notebook is the stronger host (dispatcher's call at authoring; written once either way).
4. Highlight remains a matplotlib / bokeh / plotly feature by design; the raised error documents datashader's deliberate exclusion.

Rationale: an accent overlay that washes out at datashader scale would ship a plausible-looking picture of the wrong data at exactly the scale datashader exists for (the plan's named failure mode), and a separate-panel overlay adds nothing over today's `subset()` + datashade; a legible raise that names the two real paths (`subset()` drill-down, or a vector backend for the overlay) is the honest surface. The guard is a pre-render check touching no rasterization kernel, so ADR 0002's equivalence-wall / same-run-ratio gates are not expected to fire (they still apply if any rasterization path is in fact changed). Ships as its own revertible commit; the WS F api-critic post-impl pass (already enrolled in the shared Pending list) now runs on the raised error surface and the docs recipe rather than on an overlay API. Touched: WS F block (title + all done-whens rewritten from "implement the overlay" to the raise-and-point guard + subset verification + docs recipe; the open-design-item done-when marked RESOLVED); no other workstream done-when changed (WS B/C highlight surface and G/H notebooks unaffected beyond WS H being a candidate recipe host).

**Amendment 21 — Workstream F post-impl dispositions (datashader highlight guard: message accuracy + test hardening). [In-scope tweak]**

Delta: dispositions of the WS F post-impl gate (api-critic post-impl + adversary post-impl, both blind; no viz/editorial — F ships no figure/notebook, the docs recipe is deferred to WS H per the WS F completion note). The guard itself is sound and **unchanged**: no must-fix, and the adversary confirmed no slip-through / wrong-condition fire (a `HivePlot` with highlights raises on all three entry points, one without them renders, subset-on-datashader renders incl. the Dask-lazy edge route). Five actionable findings, all on the guard's error **message** and its **test coverage**, none with downstream bearing on a not-yet-run workstream. **Two message items are user-facing correctness, not polish** (the shipped message names the wrong set of overlay backends — it omits holoviews, which WS C made fully render the overlay — and it omits the one unblocking move for the base-view user), so they are fixed now, not batched to plan-end. The three test items **lock claims the shipped source already satisfies** (the `getattr` type no-op, the lazy-stays-lazy half of the zero-new-code claim); **no shipped-guard-behavior change** in any item. One adversary finding is accepted with no action.

Fix-list — MESSAGE / SOURCE (the message accuracy items; user-facing correctness):

- **[worth-discussing → message fix] The overlay-backend list omits holoviews.** The guard message (`viz/datashader.py:110`) and the exception's own docstring (`exceptions/hive_plot.py:125-126`) both name "matplotlib, bokeh, or plotly," but WS C shipped the overlay on holoviews too (all four vector backends fully render it), so a holoviews user is steered off a backend they are on. Fix: add holoviews to the named overlay backends in **both** places the list is enumerated (the `_guard_no_highlights` message and the `HighlightsUnsupportedByDatashaderError` docstring). The three `:raises:` lines (`datashader.py:984, 1269, 1543`) say "a non-``datashader`` back end for the overlay" with no enumeration, so they need no holoviews edit on this axis (they carry item 2 below instead). (api-critic post-impl.)
- **[worth-discussing → message fix] No pointer for the "I just want the datashaded BASE view" user.** The two paths the message names both miss this user: `subset(...)` narrows (wrong for them) and switching to a vector backend abandons datashader (wrong for them); the unblocking move is `HivePlot.remove_highlight()` (clear the highlights, then datashade the base), which the message omits. Fix: add a third clause naming `remove_highlight()` to the guard message and the exception docstring, and add the same "or `remove_highlight()` to datashade the base" pointer to the three `:raises:` lines so the catchable-error surface names it too. (api-critic post-impl.)
- **[low-confidence → fold in if it does not bloat] `closure` default asymmetry footnote.** The message's "same selection vocabulary" is true for the seeds, but `subset` closes `induced` (hiveplot.py:5582/5661) while `add_highlight` defaults `closure="incident"` (hiveplot.py:5946), so a redirected user who omits `closure=` gets a different edge set than their highlight showed. Optional half-clause on the message noting subset defaults to induced closure; fold in only if it does not bloat the message (code-engineer's judgment). (api-critic post-impl, low-confidence.)

Fix-list — TESTS (`tests/datashader_test.py`; lock claims the source already satisfies, no behavior change):

- **[worth-discussing → test only] Pin the load-bearing `getattr` no-op on `BaseHivePlot` / `P2CP`.** The guard passes non-`HivePlot` types via `getattr(instance, "highlights", None)` (they carry no registry), covered only transitively today (`test_datashade_cmap`, `test_expected_errors_wrong_types` exercise those types but do not name the guard no-op). Fix: add a test that datashades a `BaseHivePlot` and a `P2CP` through a datashade entry point and asserts **NO** raise — locks the feasibility point that the guard passes registry-less types cleanly rather than `AttributeError`-ing. (adversary post-impl finding 1.)
- **[low-confidence → test only] The lazy-Dask subset test does not assert the subset stayed lazy.** `test_datashade_subset_child_over_lazy_dask_edges_renders` (datashader_test.py:1612) renders but never checks the child's edge ids are still a `LazyEdgeSubset` before rendering. Fix: before the render, assert the subset child's stored edge ids are a `LazyEdgeSubset` — pins the "incl. lazy Dask" half of the zero-new-code claim (the child rasterizes straight from lazy stored ids, geometry never materialized). (adversary post-impl finding 2.)
- **[accepted, no action] Wrong-condition regression via two disjoint fixtures rather than one contrasting pair.** The raise-when-highlighted / render-when-not contrast is split across `test_datashade_with_registered_highlights_raises` and `test_datashade_without_highlights_renders_unchanged` rather than one paired fixture. Belt-and-suspenders; the two disjoint tests already cover both arms of the condition. No change. (adversary post-impl finding 3, recorded as accepted.)

Rationale: fold the WS F post-impl gate outcomes into contract as one code-engineer follow-up. The guard's behavior is correct and stays intact; every item hardens the message's honesty or a test's named claim. The message fixes route **now, not batched**, because they are user-facing accuracy — the shipped error steers a holoviews user off a working backend and gives the base-view user no unblocking move — not cosmetic wording; the test items lock claims the source already satisfies (type no-op, lazy-stays-lazy) so the post-impl passes hold them as contract. No downstream not-yet-run workstream is affected (WS H's deferred datashader docs recipe reads the same guard surface, unchanged in shape). No WS F done-when is edited (all three are already met; this hardens the message and tests the done-whens already describe). Touched: no workstream done-whens changed; no viz/editorial block (F ships no figure/notebook); CHANGELOG untouched (message-accuracy + test hardening on an unreleased same-branch guard, covered by the existing WS F bullet).

**Amendment 22 — Workstream G post-impl dispositions (subset gallery notebook: prose-honesty fixes to a passed anecdote). [In-scope tweak]**

Delta: dispositions of the WS G post-impl gate battery (editorial-critic + viz-critic — both read-only, their blocks recorded by this amendment into `## Notebook review` (editorial) and `## Viz review` (viz); adversary post-impl). The **anecdote gate PASSED** (editorial-critic: the Philippines ego drill-down reveals structure genuinely buried in the full 3-continent hairball, on the real bundled trade dataset, frozen-geometry comparability practiced not asserted). Because the story landed, the Amendment-8 "story didn't land → route to amend-plan / reconsider scope" trigger correctly did **NOT** fire; these are honesty fixes to a landed story, **not a scope change**. viz-critic returned **clean** (three figures, polish-in-proportion matched, darkorange accent colorblind-safe via redundant alpha/linewidth/position; all findings low-confidence, none blocking, no action). The notebook nonetheless ships **three PROSE must-fixes** that oversell/misframe the landed story (the plan's inherited "plausible picture of the wrong data" mode, here surfacing in the *prose* over correct figures/data). Six actionable findings, all **markdown-only — no feature, figure, dataset, or code change** — route to one notebook-author follow-up (3 prose must-fixes, 3 worth-discussing); one low-confidence item is a note-at-most. No WS G done-when is edited (the notebook's subject, dataset, class scope, and figures are unchanged and correct; every fix corrects prose that reads a styling choice or a sort order back as a data finding). The shipped WS G artifact is preserved intact.

Fix-list — PROSE MUST-FIXES (markdown only; no feature/figure/data change):

- **[must-fix] Ego headline is factually false (cells 12–13 flexitext title + prose).** The title + cell-13 prose say the Philippines has "only thin spokes to Europe and North America," but the SINGLE LARGEST flow in its ego graph is the USA import (8.27M, larger than any intra-Asia flow); edge width here encodes CATEGORY (uniform gray vs orange), not volume, so the prose reads a styling choice back as a data fact. Reframe honestly: the concentration is about trade PARTNER COMPOSITION (7 of 11 partners are Asian), and the single largest flow (to the US) is the standout exception the drill-down surfaces — a truer, richer anecdote. Do not imply the US trade is minor. (editorial-critic must-fix / adversary "wrong-data-in-prose" mode.)
- **[must-fix] Backbone figure is a tautology (cells 14 / 18).** cell-14 sells "see the high-volume backbone"; cell-18 admits the surviving nodes sit at the value-sorted axis tip by construction. Selecting top-40%-by-value then noting they sit at the high-export-rank end of an axis SORTED by export rank reveals the sort order, not structure. Reframe the section honestly as "another way to select: by a data column (a value threshold)," demonstrating expression + mask selection as the point; drop the "backbone / high-volume / finding" insight framing (or replace it with a genuinely structural observation if one exists).
- **[must-fix] Hero caption mis-attribution (cell 13).** Recast "the Philippines' cluster of Asian trade partners" to "intra-Asia trade among the Philippines and its partners" (the orange bundle, lit via `update_edges("Asia","Asia")`, includes partner-to-partner edges kept by `hops=1` induced closure), consistent with the notebook's own cell-9 ego-precision teaching. (editorial-critic must-fix.)

Fix-list — WORTH-DISCUSSING (markdown):

- **[worth-discussing] Disclose the axis-position variable is total exports, computed pre-filter.** Add one sentence noting the axis-position variable (`export_value` / its rank) is each country's TOTAL exports across all destinations (computed before the 3-continent filter), so a node's placement can reflect trade not drawn in the plot.
- **[worth-discussing] "How Selections Combine" (cell 20) is prose-only, breaking the show-the-move gallery format.** Add a minimal combined-call demo cell, or fold the union/intersection rules into the `add_highlight` cross-reference and drop the standalone heading. (editorial-critic worth-discussing.)
- **[worth-discussing] Reorder the value-selection beat (cells 15–19).** The mask/expression equivalence currently interleaves into the middle of the backbone beat, so the figure lands only after an API-parity check. Plot the value-threshold subset right after the expression cell, then show the mask equivalence as a brief coda. (editorial-critic worth-discussing.)

Optional / note-at-most (no required action):

- **[low-confidence]** The `update_edges("Asia","Asia")` warn-if-empty dependency is scout-verified populated; an optional one-line defensive note at most. The viz-critic's three low-confidence items (deliberate muted-darkgray parent base, flexitext-title grayscale luminance, dense-parent node overplot) all need no action.

Rationale: fold the WS G post-impl gate outcomes into contract as one notebook-author follow-up. The anecdote landed and the figures are clean, so this is **not** a scope change: no dataset, class scope, genre, subject, figure, or code changes — every fix corrects prose that oversells or misframes a correct picture (a factually-false ego headline that reads uniform-vs-accent styling as trade volume; a backbone section that reveals the axis sort order and calls it structure; a caption inconsistent with the notebook's own ego-precision teaching). The three must-fixes route **now** because they are user-facing prose honesty (a shipped notebook stating a false data claim), not cosmetic wording. No downstream not-yet-run workstream is affected (WS H is a separate highlight notebook on its own scouted dataset; WS I/J are code stretch workstreams). No WS G done-when is edited (the notebook's subject/dataset/scope/figures are unchanged and correct). Touched: `## Notebook review` (WS G editorial block recorded), `## Viz review` (WS G viz block recorded, both critics being read-only); no workstream done-whens changed; CHANGELOG untouched (the existing WS G Documentation bullet already covers the notebook; prose-honesty edits to an unreleased same-branch notebook add no released-behavior change).

**Amendment 23 — Default highlight overlay edge linewidth retuned down (2.5 was too heavy on a dense network). [In-scope tweak]**

Origin: maintainer directive (2026-07-07) after reviewing the highlight renders on the real trade dataset. Amendment 16 raised the overlay edge linewidth from the base 1.5 to 2.5 (viz-critic worth-discussing: "overlay reads as recolored, not spotlit"), tuned so the accent pops on a SPARSE toy graph. On a denser real network (the WS G/H trade dataset) 2.5 reads as too heavy: the accent bundle blooms into a thick band rather than a clean spotlight. This retunes the shipped default down so emphasis leans on the accent COLOR + opacity (0.9) and the node-size bump (40) more than raw stroke weight, render-tested on a dense network.

Scope (precise, and precisely bounded):

- **Only the overlay edge linewidth changes.** Reduce the default from 2.5 to a lighter value that still reads as emphasis; the exact number is the code-engineer's render-tested call (target ~2.0, i.e. still meaningfully above the 1.5 base, not back to base). The other two Amendment-16 spotlight bumps are **NOT** touched: `_OVERLAY_NODE_SIZE=40` and `_OVERLAY_EDGE_ALPHA=0.9` (and the per-backend node-size / node-alpha / edge-opacity siblings) stay exactly as shipped. The maintainer flagged edge thickness alone.
- **Four backend constants, all currently 2.5** (one edit each, same new value): `viz/matplotlib.py:657` `_OVERLAY_EDGE_LINEWIDTH`, `viz/bokeh.py:827` `_OVERLAY_EDGE_LINE_WIDTH`, `viz/plotly.py:1015` `_OVERLAY_EDGE_LINE_WIDTH`, `viz/holoviews.py:964` `_OVERLAY_EDGE_LINE_WIDTH`. (The directive named `hiveplot.py`; the constants in fact live in the four `viz/*.py` modules — no `_OVERLAY_*` constant exists in `hiveplot.py`. The value is a per-backend rendering default, single-sourced per module.)
- **Tests that pin the value.** Two backend test files hardcode the numeric `2.5` and must move to the new value: `tests/hiveplot_test.py:9669` (`test_plot_highlight_overlay_uses_raised_spotlight_defaults`, `np.allclose(line.get_linewidths(), 2.5)`) and `tests/viz_holoviews_test.py:1434` (the overlay-edge collector filter `... .get(line_width_name) == 2.5`). The bokeh and plotly collectors reference the module constant itself (`viz_bokeh_test.py:1019` matches `hiveplotlib.viz.bokeh._OVERLAY_EDGE_LINE_WIDTH`; `viz_plotly_test.py:1077` matches `hiveplotlib.viz.plotly._OVERLAY_EDGE_LINE_WIDTH`), so they auto-follow the constant and need **no** numeric edit. Leave the `edge_kwargs`-override tests (which pass an explicit `linewidth=5` and assert 5) untouched — they prove per-highlight override still wins over the new default and must keep asserting the overridden value, not the default.

Value selection (render-test to pick, do not eyeball a number into the code): before committing, the code-engineer render-tests the candidate value on a DENSE network (the WS G trade dataset, or an `example_hive_plot` with enough edges to bloom) across at least matplotlib, and confirms the accent still reads as emphasis at that density (lighter stroke, carried by color + the retained 0.9 opacity + the retained size-40 nodes), not so thin it collapses back onto the base. The plan's target is ~2.0; the engineer may land slightly off it if the render says so, staying above the 1.5 base.

Post-impl gates: this is a **shipped-default retune, not new surface** (no signature, no kwarg, no new symbol; the same constant carries a different literal). So the **api-critic post-impl need not re-run** (no API surface changed). qa re-verifies tests + 100% coverage across the four backends (all four backend suites), ruff/ty clean. The **maintainer's WS H highlight-notebook review is the visual confirmation**: he will see the corrected default rendered on the dense trade network there (his visual gate), which is why the number is render-tested now rather than deferred. A viz-critic re-pass is not required for a single-constant density retune, but is not precluded if the WS H notebook render surfaces a further concern (that would route back here).

Rationale: Amendment 16's 2.5 was correct for the sparse case it was tuned against and wrong for the dense real network the feature actually ships against; leaning the emphasis onto color + opacity + node size (all kept) rather than stroke weight is the honest fix, and it is one literal single-sourced per backend, so the change is small, revertible, and fully test-pinned. This does not reopen Amendment 16's other two bumps (node size, edge alpha), which the maintainer explicitly held. No workstream done-when is edited (the overlay surface is unchanged; the done-whens describe "raised spotlight defaults," which a linewidth of ~2.0 still satisfies — heavier than base, carried by color/opacity/size). No downstream not-yet-run workstream is blocked: WS H reads the retuned default as its render (it is the visual gate for this change, not a blocker of it); WS I/J are code stretch workstreams untouched by a rendering constant. Touched: no workstream done-whens changed; no `## Patterns this replaces` entry (a value retune of an unreleased same-branch default, not a pattern sweep); CHANGELOG untouched (per the house rule, a refinement to the overlay defaults debuting in this same unreleased version earns no CHANGELOG entry, and no shipped prose/CHANGELOG cites the literal 2.5 — verified — so nothing else needs a sweep).

**Amendment 24 — Workstream H post-impl dispositions (highlight gallery notebook: prose refinements to a PASSED, data-verified anecdote). [In-scope tweak]**

Delta: dispositions of the WS H post-impl gate battery (editorial-critic + viz-critic — both read-only, their blocks recorded by this amendment into `## Notebook review` (editorial) and `## Viz review` (viz); adversary post-impl self-wrote its section). The **anecdote gate PASSED** (editorial-critic: the China-broad-hub vs Mexico-focused-regional emphasize-in-context story lands as a real finding the overlay surfaces, on the real bundled trade dataset) and **data-honesty was VERIFIED** — unlike the sibling WS G subset notebook, this notebook made NO false volume/direction claims: the adversary recomputed every falsifiable number against the raw CSV and the notebook already disclaims volume (uniform overlay width) and asserts no direction on the directed exports network. So the two failure modes that dogged WS G (the false volume claim, the direction error — Amendment 22's must-fixes) did **NOT** recur here, and the Amendment-8 "story didn't land / wrong-data-in-prose → route to amend-plan / reconsider scope" trigger correctly did **NOT** fire. viz-critic returned **clean** and is also the recorded **visual gate for Amendment 23** (the overlay edge linewidth retune 2.5 → 2.0), which it CONFIRMED reads as clean emphasis on the dense CHN hub. These are **prose refinements to a passed, data-verified anecdote — not a scope change**: no dataset, class scope, genre, subject, figure, or code change; every fix is markdown-only (or a small in-cell verification). No WS H done-when is edited. The shipped WS H artifact (notebook subject, dataset, six figures, GIF-sweep-as-static-strip decision, datashader coda) is preserved intact.

Fix-list — route to one notebook-author follow-up (all markdown / a small verification, no feature/figure/data change):

- **[worth-discussing] Soften the sweep "Europe-Asia backbone" framing (structure-is-artifact).** The numbers are honest, but North America's ~8% retention at >$5M is arithmetically forced by its axis being one hub (USA) plus a tail of Caribbean/Central-American micro-states that were never clearing $5M — not a discovered network structure. Reframe to NAME THE MECHANISM (North America's axis is mostly very small economies, so it thins fastest by count) rather than stating "the high-value backbone is Europe and Asia" as a bare finding. (adversary post-impl worth-discussing.) — sweep-strip section (cell21 and its prose).
- **[worth-discussing] Add the pre-filter caveat (parity with the subset sibling).** One sentence in "The Base Plot" noting each country's `export_value` (its axis rank) is total exports across ALL destinations, computed before the 3-continent filter, so a node's placement can reflect trade not drawn in the plot. The sibling subsetting_hive_plots.ipynb already carries this caveat; parity is cheap. (editorial-critic post-impl worth-discussing.) — "The Base Plot" section.
- **[low-confidence] Confirm the sweep per-continent percentages (Europe 32% / Asia 21%) are DATA-COMPUTED, not eyeballed.** The adversary verified NA 8% structurally but could not hand-verify Europe/Asia. If the sweep cell already computes per-continent retention, confirm the prose matches the computed values; if the percentages are hardcoded in prose, compute them in the cell (or verify against the data) so they are deterministic and correct. A real "plausible picture of the wrong data" guard given the sibling's history. (adversary low-confidence.) — sweep-strip cell/prose.
- **[low-confidence] Suppress the stray `0` output** (the `add_highlight(...)` return value displayed as a cell's last statement) — add a trailing `;` or bind the returned key. Cosmetic. — the `add_highlight(...)` cell(s).

Optional / note-at-most (no required action): genre length (six H2 sections is at the upper bound but each is short, motivated, and plan-sanctioned — taste, not a cut); the closing-pointer enumeration (four next-step notebooks, each individually justified, borderline); the datashader coda (a contracted WS F-deferred recipe that teaches a real move); the viz-critic's two low-confidence items (cell12 two-accent grayscale luminance separation and the vermillion-vs-pure-light-base 2.58:1 computed contrast — both known base-dependent caveats already carried by the add_highlight docstring, Amendment 16, and both clear the alpha-thinned base unmistakably).

Rationale: fold the WS H post-impl gate outcomes into contract as one notebook-author follow-up. Unlike Amendment 22 (WS G), there is no must-fix here — the anecdote landed AND the data-honesty was independently verified against the CSV, so none of these are false-data corrections; they are refinements to an already-honest, already-landed story (name a thinning mechanism instead of asserting a bare "backbone" finding; a cheap parity caveat; a determinism confirmation on two percentages the adversary could not hand-verify; a cosmetic stray output). Because the story landed and the data verified clean, this is **not** a scope change: no dataset, class scope, genre, subject, figure, or code change; every fix is markdown-only or an in-cell verification. No WS H done-when is edited (the notebook's subject/dataset/scope/figures and the GIF-sweep-as-static-strip and datashader-coda decisions are unchanged and correct). No downstream not-yet-run workstream is affected (WS I/J are code stretch workstreams; WS H is the terminal notebook workstream). Touched: `## Notebook review` (WS H editorial block recorded), `## Viz review` (WS H viz block recorded, also the recorded visual gate for Amendment 23's retune); no workstream done-whens changed; CHANGELOG untouched (the existing WS H Documentation bullet already covers the notebook; prose-refinement edits to an unreleased same-branch notebook add no released-behavior change).

**Amendment 25 — Workstream I CUT (region selection): placement-select is a value threshold in disguise, redundant with shipped WS D; deferred to the interactive-explorer spatial lasso. [Deferred follow-up]**

Delta: **Workstream I is CUT**, exercising the explicit cut option written into its own done-when ("**explicitly allowed to be cut** if it doesn't pan out; record the cut as a deferred follow-up, not a silent drop") and the Goal's "explicitly cuttable" framing. The api-critic's planning-mode evaluation (recorded above as the "Workstream I region-select proposal", ran 2026-07-07) returns **CUT [confidence: high]**, and the reasoning is decisive and empirical, not a taste call. Nothing is built, so there is no code, no commit, no api-critic post-impl, and no qa — the cut is the resolution.

Why cut (the empirical finding, so the decision is auditable and the dead end is not silently re-planned):

- **Placement IS the linear value map; there is no rank-based spreading in this library.** Node placement (`rho`) is computed in `place_nodes_on_axis` (hiveplot.py:1075-1092) as a strictly MONOTONIC LINEAR transform of the sort value: clip to `[vmin, vmax]`, normalize to `[0, 1]`, scale to the axis span. A grep of both `hiveplot.py` and `p2cp.py` for `rank` / `argsort` / `rankdata` / spread / jitter returns **zero matches** — nothing spreads nodes by rank. So "top 20% of placement on axis A" is node-for-node **identical** to the value threshold `nw.col(sort_var) >= vmin + 0.8 * (vmax - vmin)`, which shipped in Workstream D.
- **WS I's own item-1 justification is empirically FALSE for this library.** The done-when's hedge — that ties/spread can make placement diverge from a clean value threshold, and that placement-position selection is therefore "the genuinely new surface" — does not hold: placement is that same linear value map, ties land at one placement and a value threshold catches them identically, and clipping piles out-of-range values at the endpoints exactly as a raw-column threshold does. There is no node set a placement range selects that the corresponding value threshold does not.
- **Redundant with shipped WS D.** The motivating "that patch between A and B where both endpoints sit high" story is already a one-liner today, post-WS-D: `hp.subset(nodes=(nw.col(a_sort) >= a_thr) & (nw.col(b_sort) >= b_thr), edges=("A", "B"))`. Region-select would ship only a normalized-units skin over existing value-expression selection — the "unjustified stretch" the adversary's cold item 6 flagged, now confirmed empirically rather than argued.

Deferred follow-up (the honest revival target, so the door stays open on the RIGHT affordance): the real "select what I see" capability is a **spatial lasso over the drawn `(x, y)` (or `rho`/`angle`) coordinates** — an arbitrary polygon over the raster, which is fundamentally NOT expressible as a value threshold and is what a dense datashader user actually reaches for. A per-axis placement rectangle is still a value threshold and would not compose with a lasso (a lasso keys off cartesian/polar geometry, not per-axis normalized sort fractions), so it is not a down payment on the lasso. **Owned by the interactive-explorer / anywidget track** (which already owns interactive selection; see this plan's Non-goals "Lasso / interactive selection" entry, which pre-homes it there). Revival target recorded as "spatial lasso selection over drawn coordinates," NOT this redundant placement rectangle. Optional future convenience, NOT selection vocabulary and NOT a reason to revive I: a tiny `Axis.value_at_placement(frac)` accessor converting a 0..1 placement fraction to a value via the axis's `sorting_variable` + `vmin`/`vmax` (all already present, axis.py:94/202-203) — a two-line ergonomic helper, not a `region=` parameter forking the polymorphic `nodes=`/`edges=` resolver.

Rationale: cut a stretch workstream whose entire surface is a units-relabel of a capability that already shipped, and record the finding so it is not re-planned. Because nothing was built, this is a pure planning delta: no source, test, notebook, figure, or CHANGELOG change; the WS D value-selection surface is the shipped answer to the region-select use case and is preserved intact. Disposes the WS I done-when's own "cut → deferred follow-up" clause and the adversary's cold item 6 (over-inclusive stretch) by cutting rather than building. The now-cut item's residual mentions read correctly under the cut: the Goal's "Region selection is a final-stretch workstream, explicitly cuttable" (Selection vocabulary floor bullet) described it as cuttable and it is now cut; the Alignment grill wave-1 "Workstream I KEPT" line is grill record (that section is "the record only, not hand-edited plan logic") and is superseded by this amendment; the Non-goals "Lasso / interactive selection" entry already homes the real affordance in the interactive-explorer track and is unchanged. Touched: `### Workstream I` Status line (→ CUT (deferred), pointer to this amendment; the block is kept, not deleted, as the record of the considered-and-cut decision); no other workstream done-when changed; no code/test/docs/CHANGELOG.

**Amendment 26 — Lazy INCIDENT closure dropped far-endpoint placements; faithful bounded-endpoint-compute makes lazy incident match the in-memory twin. [In-scope tweak]**

Origin: the WS J post-impl adversary (blind), whose rubric ("does lazy match the in-memory twin?") surfaced a real lazy-vs-in-memory divergence on the `add_highlight` default (`closure="incident"`); api-critic returned CLEAN and the frontier/no-materialize core is clean, so this is the one WS J post-impl must-fix. The bug is PRE-EXISTING (reachable at `hops=0` lazy incident since WS B shipped `_incident_placement_ids`), not introduced by WS J; WS J's flipped completeness twin-test dodged it by choosing `closure="induced"` (which keeps no far endpoints), so the suite never pinned the incident placement set. This is a faithful CORRECTNESS fix on the incident default, CONSISTENT with Amendment 18's maintainer faithful-lazy decision (bounded endpoint compute over cheap-but-wrong).

The bug (adversary-traced): incident closure keeps an edge whose FAR endpoint is outside the selected node set, and `_restrict_to_node_ids` must keep that far endpoint's frozen placement so the overlay/child edge curve can draw (the WS B "far-endpoint placements kept only so touching edge curves draw" contract). `_incident_placement_ids` (hiveplot.py:~5781; the gate at ~5798 `if not isinstance(ids, np.ndarray): continue`) reads far endpoints ONLY from ndarray id records, so a `LazyEdgeSubset` contributes ZERO far-endpoint placements. Result: a lazy incident subset/highlight OR-composes the edge into the kept set (correct) but drops the far endpoint's placement its in-memory twin keeps -> an overlay/child edge geometry referencing an UNPLACED node (the WS D lazy-vs-in-memory divergence class, now on the incident default: a KeyError/NaN at render where the in-memory twin draws). Induced closure (the `subset` locked default) is UNAFFECTED — it keeps no far endpoints, so there is nothing to drop.

Maintainer disposition (faithful fix, consistent with Amendment 18; the maintainer was FYI'd and did not redirect to a gate): the far endpoints of an incident selection ARE its one-hop frontier, so reuse the bounded-frontier compute WS J just added (`frames.lazy_edge_frontier_endpoint_ids`) to enumerate the lazy incident far-endpoint node IDs and include their placements, making lazy incident match the in-memory twin. **Accepted cost (document it, mirroring the WS D/hops note):** a lazy INCIDENT selection now triggers ONE bounded endpoint compute even at `hops=0` (previously it composed with zero compute), which is the faithful-over-cheap tradeoff the maintainer already chose for lazy edge-expressions in Amendment 18. In-memory incident is unchanged (its ndarray far-endpoint path already places them); induced (lazy or in-memory) is unchanged.

Fix-list (route to one code-engineer follow-up; source + a shared test helper, never full-materialize):

- **[must-fix, source]** Teach the lazy path of `_incident_placement_ids` (or `_restrict_to_node_ids` / `_restrict_lazy_subset` under `closure="incident"`) to enumerate the far-endpoint node IDs of the kept lazy edges via the bounded `frames.lazy_edge_frontier_endpoint_ids` compute and include those nodes' placements, so a lazy incident subset/overlay carries the SAME per-axis placement set as its in-memory twin. Never full-materialize (only the bounded ID compute runs; the subset stays a `LazyEdgeSubset`). Update the `add_highlight` / `subset` `.. note::` to disclose that a lazy INCIDENT selection triggers a bounded endpoint compute (mirror the WS D edge-expression note and the WS J `hops` note — a performance disclosure, not a semantics asymmetry, since the point is that lazy now MATCHES in-memory).
- **[worth-discussing, tests]** The completeness twin-comparison helpers (`_surviving_node_ids`, `_materialized_edge_id_set`) never read `node_placements`, so the whole lazy battery was structurally blind to this class of divergence (node-data + edge-ids matched while placements diverged). Add a per-axis `node_placements` equality assertion to the lazy-vs-in-memory twin comparison as a SHARED helper, so the battery pins placement equality, not just node-data + edge-ids.
- **[low-confidence, tests]** Add a lazy `hops>0` highlight completeness twin-test at the DEFAULT `closure="incident"` (the flipped WS J test used induced), asserting placement equality via the new shared helper; and a lazy incident overlay RENDER test (`@pytest.mark.datashader`, actually driving a viz path) confirming the edge DRAWS rather than KeyError/NaN on the previously-missing far-endpoint placement.

Rationale: a faithful correctness fix on the `add_highlight` incident default — pre-existing (reachable since WS B), surfaced by WS J's twin-comparison rubric, and dodged by WS J's induced-closure twin-test. The fix reuses the exact bounded-frontier primitive WS J already landed (`lazy_edge_frontier_endpoint_ids`), so lazy incident matches the in-memory twin under the same no-materialize invariant, and it accepts the one bounded compute at `hops=0` that Amendment 18 already established as the faithful-lazy tradeoff. No new user-facing surface, signature, or kwarg (the incident placement contract is internal; the only user-visible delta is a corrected render + a performance note). No WS J done-when is edited (WS J's frontier/no-materialize core is CLEAN per api-critic + the frontier tests; this closes a pre-existing incident placement gap the WS J rubric exposed, not a WS J regression). Touched: `## Patterns this replaces` n/a (a correctness fix to an unreleased same-branch path, covered by the existing WS B/D/J unreleased bullets; the lazy incident placement-drop is the replaced behavior, recorded in the WS B/J implementation-log entries); the `subset` / `add_highlight` `.. note::` (bounded-compute disclosure for lazy incident); no CHANGELOG entry (bugfix to an unreleased same-branch capability, per the house rule and consistent with Amendments 18/23's no-entry dispositions). No downstream not-yet-run workstream (WS I is CUT per Amendment 25; WS J is the terminal code workstream).

## Holdouts

None. The four TODOs have no legitimate survivors; no other old-pattern text exists.

## Implementation log

2026-07-07: Workstream A complete. Added `HivePlot.subset(nodes=, edges=, hops=0)` that slices already-computed geometry (deepcopy then filter `nodes`/`edges`/per-axis `node_placements`/`hive_plot_edges` ids+curves) into an induced-closure child, freezing axis `vmin`/`vmax` as explicit (`inferred_*=False`); pipeline resolves node seeds + edge-seed endpoints → union → `hops`-expand → induced. Selection vocabulary: node IDs/scalar ID/list, boolean mask (numpy or narwhals-supported Series via `_is_boolean_mask`), narwhals expression, axis pair / list of pairs, single-tag bare edge mask, multi-tag mask/expression dict (absent tag = not selected). Lazy (Dask) edges: `hops=0` ID/axis-pair composes `is_in(surviving_ids)` into the stored predicate (never materializes); mask/expression or `hops>0` on lazy edges → `_raise_lazy_mask_selection_error`; lazy node data raises (RAM ceiling, docstring). Rebuilt `relevant_edges` positionally against filtered frames; added `_slice_curves` (reshape-mask-flatten). New `SubsetBeforeBuildError` (pre-geometry guard, distinct from renderable empty selection). Copy-not-recompute pinned by a spy test on `construct_curves`/`_construct_edge_subset_curves`/`_curves_from_id_pairs`. Deleted the four `data_subset`/`data_highlight` TODOs (node.py, edges.py). Added `TestSubset` (48 tests incl. polars + dask markers, non-RangeIndex positional-vs-label, frozen-geometry array-equality, union/intersection algebra) and a CHANGELOG.rst "Hive Plots" bullet. New subset source lines at 100% coverage; ruff/ty clean; docs render `HivePlot.subset` + `SubsetBeforeBuildError`. `edge_coverage` (from the sibling edge-coverage plan) is not present on branch 53, so its done-when is deferred to when that method lands (child holds its own filtered edges, so it needs no special-casing). Files: src/hiveplotlib/hiveplot.py, node.py, edges.py, exceptions/hive_plot.py, tests/hiveplot_test.py, CHANGELOG.rst.

2026-07-07: Workstream A post-impl gate fixes (api-critic + adversary). Added a shared `_require_full_length_row_mask` guard wired into both `_resolve_node_seed_ids` and `_edge_frame_mask` (reused by WS B/D/E resolvers): a selection expression must evaluate to one boolean per row or it raises a legible `ValueError` (edge path names the tag), closing the node-path raw `IndexError` and the aggregating-edge-expression "wrong subgraph" silent-mis-selection. Added a `hops >= 0` guard (`ValueError`), the missing `:raises: InvalidAxisNameError` docstring line, and docstring clarifications: `nodes=` list/array-vs-tuple-ID + silent-drop-of-unknown-IDs, softened ego wording to "undirected neighborhood; equivalent to `networkx.ego_graph` on an undirected view", and the lazy axis-pair asymmetry. Added 5 tests (node/edge aggregating-expression raises, wrong-length edge mask, multi-tag-dict aggregating-expression names tag, negative hops). Full suite 1636 passed / 14 skipped (cuDF), TOTAL coverage 100%; ruff/ty clean.

2026-07-07: Workstream B complete. Generalized WS A's resolver for closure (one implementation, not two): extracted `_resolve_selection_node_ids(nodes, edges, hops)` (seeds→union→hops, shared by `subset` and highlight) and threaded a `closure: Literal["induced","incident"]` param through `_restrict_to_node_ids` / `_restrict_edges_to_node_ids` / `_restrict_lazy_subset` (induced = both endpoints AND; incident = either endpoint OR); incident keeps far-endpoint node placements (new `_incident_placement_ids`) so overlay curves stay drawable while node data holds only the selected set. `subset` still locks induced. Registry lands on `HivePlot` (not `BaseHivePlot`, so `P2CP` never inherits it): `_highlights` dict init in `__init__`, `add_highlight(nodes=, edges=, hops=0, closure="incident", color=None, key=None, node_kwargs=None, edge_kwargs=None) -> key` (auto integer key via `_find_unique_highlight_key`; collision warns `HighlightKeyCollisionWarning` at `stacklevel=3`; combines nodes+edges into one keyed group), `update_highlight(key, ...)` (full add kwarg set via a `_UNSET` sentinel so omitted keeps / explicit `None` clears), `remove_highlight(key=None)` (bare call clears all — the chosen clear-all affordance), read-only `highlights` property returning a fresh dict of new `Highlight` dataclass records (re-exported from `hiveplotlib/__init__`, mirroring the EdgeCoverage re-export pattern). Palette: new `HIGHLIGHT_ACCENT_PALETTE` (6 Okabe-Ito colorblind-safe hues, assigned first-unused via `_next_highlight_color`, wraps when exhausted; per-highlight `color=` override respected). matplotlib overlay: `_highlight_overlay_plot` builds a throwaway restricted `HivePlot` (clears its registry + base node/edge styling so the accent color wins with no new conflict warnings), `viz/base.iter_highlight_overlays` shared-prep yields `(highlight, overlay)` per backend (BaseHivePlot/P2CP yield nothing), `hive_plot_viz` draws overlay edges (zorder 6) then nodes (zorder 7, via `_draw_overlay_nodes` to skip the empty-axis build warning) strictly above the base; base-without-highlights render is byte-for-byte unchanged (regression test). `node_kwargs`/`edge_kwargs` passthrough SHIPPED (near-free through the viz machinery). Render-time semantics + subset-inherits-no-highlights + copy-survives documented. New `HighlightKeyCollisionWarning` (exceptions/hive_plot.py). Added `TestHighlight` (40 tests: registry lifecycle, palette first-unused/skip/wrap/override, incident-vs-induced via the shared resolver, overlay zorder + byte-for-byte base regression + no-conflict-warning + accent color + kwargs passthrough + multiple/empty/single-node overlays, copy/subset interplay, BaseHivePlot no-registry + P2CP renders, non-RangeIndex ID selection, narwhals expression, dask hops-0-stays-lazy / hops>0-raises). New source 100% covered; ruff/format/ty clean; docs render `add_highlight`/`update_highlight`/`remove_highlight`/`highlights`/`Highlight`/`HIGHLIGHT_ACCENT_PALETTE`/`HighlightKeyCollisionWarning`. Files: src/hiveplotlib/hiveplot.py, viz/base.py, viz/matplotlib.py, exceptions/hive_plot.py, __init__.py, docs/source/autodoc/hive_plots/high_level_hive_plot_api.rst, tests/hiveplot_test.py, CHANGELOG.rst.

2026-07-07: Workstream B post-impl battery fixes (api-critic + viz-critic + adversary, 8 items). Incident overlay now accent-lights only the SELECTED nodes: `_draw_overlay_nodes` filters the scatter to `overlay.nodes.data` ids so far-endpoint placements (kept only so touching edge curves draw) get no dot (was lighting every neighbor). Raised overlay spotlight defaults (`_OVERLAY_EDGE_LINEWIDTH=2.5`, `_OVERLAY_EDGE_ALPHA=0.9`, `_OVERLAY_NODE_SIZE=40`) so a highlight spotlights rather than merely recolors; per-highlight `edge_kwargs=`/`node_kwargs=` still override (extracted `_draw_highlight_overlay` per-overlay helper). `key` is now keyword-only on `add_highlight` (a positional key was silently binding to `edges=`). Added `_validate_highlight_closure` (legible `ValueError` on a bad `closure` like the `"induce"` typo) wired into both `add_highlight` and `update_highlight`. Guarded `update_highlight(hops=None)` (was a raw `TypeError` from `None<0`); scoped the "None clears" docstring to the selection fields (`nodes`/`edges`) only. Docstrings: `highlights` records echo the RAW selection not the resolved set; `add_highlight` dense-overlay caveat + palette-tuned-against-dark-base note. Added 8 `TestHighlight` tests: `test_plot_incident_highlight_does_not_accent_light_neighbor_nodes`, `test_plot_highlight_overlay_uses_raised_spotlight_defaults`, `test_plot_highlight_edge_kwargs_override_raised_defaults`, `test_add_highlight_key_is_keyword_only`, `test_add_highlight_invalid_closure_raises`, `test_update_highlight_invalid_closure_raises`, `test_update_highlight_none_hops_raises`, `test_highlight_non_range_index_mask_selection`. Full suite 1684 passed / 14 skipped (cuDF), TOTAL coverage 100%; ruff format/check + ty clean.

2026-07-07: Workstream D complete. narwhals-expression selection at every point (subset `nodes=`/`edges=`, highlight `nodes=`/`edges=`) was already built into WS A's shared resolver (`_resolve_node_seed_ids`, `_edge_frame_mask`) with the full-length-mask guard; WS D verified it and closed the lazy gap. **Net-new (lazy Dask edge expression composition):** a narwhals expression against a lazy (Dask) edge tag now composes into that tag's stored `LazyEdgeSubset.predicate` and NEVER materializes, instead of routing to the in-memory error (a *concrete* boolean mask still errors, since a pre-computed per-row array cannot be deferred). Implementation: `_resolve_edge_seed_endpoint_ids` now returns `(endpoint_ids, {tag: expr})` and, for a lazy tag with an expression, seeds every node placed on the axes the tag spans (new `_lazy_tag_placed_axis_node_ids`, the same broadened-seed asymmetry as lazy axis-pair) rather than enumerating endpoints; the expression rides through `_resolve_selection_node_ids` → `_restrict_to_node_ids` → `_restrict_edges_to_node_ids` → `_restrict_lazy_subset(extra_predicate=)`, ANDed into the lazy predicate so it does the real row filtering while staying lazy. `_raise_lazy_mask_selection_error` + the `subset`/`_highlight_overlay_plot` docstrings/note reworded (concrete-mask/`hops>0` only). Fixed `add_highlight`'s `node_kwargs`/`edge_kwargs` docstring examples (batched WS-C api-critic fold-in): they taught matplotlib-only `{"s":...}`/`{"linewidth":...}` as universal, but highlights now render on bokeh/plotly where the names diverge, so each is now named as the active backend spells it (matplotlib `s`/`linewidth`, bokeh `size`/`line_width`, plotly `size`/`width`); added a mask-vs-expression side-by-side to the `add_highlight` lede (subset already had one). Added 15 tests: pandas edge-expression-matches-mask, node-expression on a non-partition/non-sorting variable, non-RangeIndex positional obligation for expressions (subset + highlight), polars node/edge expression (`@pytest.mark.polars`), lazy Dask edge-expression stays-lazy-and-filters + per-tag dict + concrete-mask-still-raises + highlight overlay lazy edge-expression (`@pytest.mark.datashader`), highlight edge-expression-matches-mask, and 3 cuDF twins in cudf_test.py (`@pytest.mark.cudf`, GPU-gated, skip in CI). New source 100% covered; ruff format/check + ty clean; no CHANGELOG entry (the WS A subset bullet + WS B "same selection vocabulary" already name narwhals-expression selection; lazy-edge composition is an implementation detail of the same unreleased capability, no sibling entry per house rule). Files: src/hiveplotlib/hiveplot.py, tests/hiveplot_test.py, tests/cudf_test.py.

2026-07-07: Workstream D lazy-expression correctness fix (Amendment 18; post-impl adversary caught the divergence). The original lazy edge-expression approach above (broad axis-node seed + expression ANDed into the predicate as a literal edge filter via `extra_predicate`) produced a strictly NARROWER subgraph than the in-memory path: in-memory `edges=expr` resolves the expression to endpoint nodes then applies induced/incident CLOSURE (keeping e.g. a low-weight edge between two high-weight-edge endpoints), whereas the literal filter kept only edges satisfying the expression (54 vs 40 edges on the seed-0 fixture; the must-fix completeness anchor would have failed). Per the maintainer's decision (make it faithful, no approximate flag), replaced the literal-filter with a bounded endpoint-compute: new `frames.lazy_edge_expression_endpoint_ids` runs ONE bounded Dask compute (`filter(expr).select([from,to]).collect()` → unique) to gather just the matching edges' unique endpoint node IDs (never the full edge frame; the ID set fits in RAM by the WS A node ceiling). `_resolve_edge_seed_endpoint_ids` feeds those IDs into the normal node-seed path exactly like the in-memory branch, so the downstream closure (`_restrict_lazy_subset`'s `is_in(from,seed) & is_in(to,seed)` induced / OR-form incident) reproduces the in-memory subgraph byte-for-byte while the `LazyEdgeSubset` stays lazy. Removed the now-unneeded `lazy_expressions`/`extra_predicate` threading (`_resolve_selection_node_ids`/`_restrict_to_node_ids`/`_restrict_edges_to_node_ids`/`_restrict_lazy_subset` reverted to their pre-WS-D signatures) and the `_lazy_tag_placed_axis_node_ids` broad-seed helper. Verified equality: subset induced 54==54, highlight incident 66==66, highlight induced 54==54, matching node sets, subset stays lazy. The three lazy-expression tests rewritten as correctness pins (materialize the lazy subset, assert its edge set + node set EQUAL the in-memory twin's on the same seed+expression; dropped the `if len(materialized)>0` vacuity guards, assert non-empty) via new `_materialized_edge_id_set` test helper; covers subset induced + per-tag dict + highlight incident. `subset` `.. note::` reworded (item c): edge-expression now triggers a bounded endpoint compute to stay faithful (a perf note), the axis-pair over-inclusiveness disclosure kept separate. Items 2/3 (plotly `line_width` dual-spell, backend-agnostic `color=`) applied. Full suite 1725 passed / 17 skipped (cuDF), TOTAL coverage 100%; ruff/ty clean; docs build clean. No CHANGELOG entry (bugfix to an unreleased same-branch path, covered by the existing unreleased bullets). Files: src/hiveplotlib/frames.py, src/hiveplotlib/hiveplot.py, tests/hiveplot_test.py.
2026-07-07: Amendment 26 docstring-only polish (WS J fix post-impl critics; NO behavior/test-logic change). (1, api-critic worth-discussing) Surfaced the lazy-incident bounded-endpoint-compute cost at the `add_highlight` entry point where a highlight user reads it, instead of only two hops away in `subset`'s `.. note::`: added one sentence to `add_highlight`'s existing render-time / timing paragraph (hiveplot.py ~6018) noting that on lazy (Dask) edge data the default `closure="incident"` resolves at each plot call by running one bounded endpoint compute (endpoint columns only, never a full materialize; result matches the in-memory path) to gather the touching edges' far endpoints even at `hops=0`, points at `closure="induced"` for the no-far-endpoint path, and lets the existing pointer to `subset` carry the full lazy-edge cost vocabulary. Folds in the pre-existing WS J `add_highlight`-timing low-confidence note; matches the sibling disclosures' voice (plain, no em-dashes, no AI filler, no process refs); no whole-note duplication. (2, adversary low-confidence) Softened the overclaiming docstring on `test_highlight_overlay_lazy_incident_edges_render_draws_far_endpoint` (hiveplot_test.py ~10268): it previously read as though a successful raster confirms the far endpoint is placed, but datashader silently NaN-drops a missing placement so the render passes with or without the fix. Reworded to state honestly that it is a real-viz-path smoke check driving the datashader render through the actual curve-construction path, and that the placement-equality guard lives in the twin-comparison tests. Assertions and markers unchanged (docstring text only). Left the two induced `_assert_node_placements_match` call sites wired (intended future-proofing, not a gap). Verified: the two touched tests pass, ruff format --check + ruff check clean on src + tests. Files: src/hiveplotlib/hiveplot.py, tests/hiveplot_test.py.
2026-07-07: Workstream J follow-up fix (Amendment 26 — lazy INCIDENT closure dropped far-endpoint placements). Faithful correctness fix on the `add_highlight` incident default: `_incident_placement_ids` (hiveplot.py:5781) read far endpoints only from ndarray id-records, so a `LazyEdgeSubset` contributed zero far-endpoint placements — a lazy incident overlay kept the edge (OR-composed) but dropped the far endpoint's frozen placement the in-memory twin keeps, leaving an overlay edge geometry referencing an unplaced node (pre-existing since WS B, reachable at `hops=0`; the WS J flipped completeness test dodged it with `closure="induced"`). Fix: taught the lazy branch of `_incident_placement_ids` (new call site hiveplot.py:5806) to enumerate the kept lazy edges' far endpoints via the bounded `frames.lazy_edge_frontier_endpoint_ids` primitive WS J landed — the incident far endpoints ARE the one-hop frontier of the selection. It runs the frontier over each stored `LazyEdgeSubset.filtered_native()` (so the subset's own predicate is honored, gathering only its kept edges' far endpoints touching the selection), one bounded `is_in` scan over the endpoint ID columns; no full materialize (verified: the only new lazy call is that bounded frontier gather; the subset stays a `LazyEdgeSubset`). In-memory incident path unchanged (its ndarray far-endpoint scan already places them); induced closure keeps no far endpoints, so `subset` (locked induced) is untouched. Updated the `subset` `.. note::` (referenced by `add_highlight`) to disclose that a lazy INCIDENT selection triggers one further bounded endpoint compute even at `hops=0` to gather the touching edges' far endpoints (a performance disclosure now that lazy matches in-memory, mirroring the WS D edge-expression note and the WS J hops note). Tests: added a shared `_node_placements_by_axis` / `_assert_node_placements_match` helper (the prior twin helpers `_surviving_node_ids` / `_materialized_edge_id_set` never read `node_placements`, so the battery was structurally blind to placement divergence) and wired the per-axis placement-equality assertion into all four lazy-vs-in-memory twin comparisons (subset hops, highlight hops-induced, highlight expression-incident, and the new incident-hops test). Added `test_highlight_overlay_lazy_edges_hops_greater_than_zero_incident_matches_in_memory` (default `closure="incident"`, `hops=2`, `@pytest.mark.datashader`; asserts placement equality via the shared helper) and `test_highlight_overlay_lazy_incident_edges_render_draws_far_endpoint` (`@pytest.mark.datashader`; datashades the lazy incident overlay through `datashade_hive_plot_mpl`, driving `_curves_from_id_pairs` on the far-endpoint placements, confirming the edge draws). Verified the fix is load-bearing: reverting the lazy branch fails both incident placement-equality assertions (the render path silently NaN-drops rather than crashing, confirming the WS J adversary's low-confidence symptom read; the placement-equality pin is the real completeness guard). New source lines 100% covered; 109 Subset/Highlight tests pass, 14 datashader subset/highlight/lazy tests pass; ruff format/check + ty clean. No CHANGELOG entry (bugfix to an unreleased same-branch capability, per the house rule and Amendments 18/23's no-entry dispositions). Files: src/hiveplotlib/hiveplot.py, tests/hiveplot_test.py.

2026-07-07: Workstream D lazy missing-column error legibility (adversary re-attack, only new finding). The bounded-compute lazy path (`lazy_edge_expression_endpoint_ids`) had no error wrapping, so a bad-column expression on a lazy edge frame surfaced a raw narwhals/Dask traceback instead of the legible tag-named `ValueError` the in-memory `_edge_frame_mask` raises (an eager-vs-lazy error-UX inconsistency on the same `subset`/`add_highlight` surface). Extracted the shared message into a module-level `_edge_expression_evaluation_error(tag, exc)` helper, used by both `_edge_frame_mask` (eager) and the lazy call site in `_resolve_edge_seed_endpoint_ids` (which now wraps the bounded compute in try/except and re-raises the same tag-named error), so both paths report identically. Added `test_subset_lazy_edges_expression_missing_column_names_tag` (`@pytest.mark.datashader`) mirroring the in-memory `test_subset_multi_tag_dict_missing_column_names_tag`. Full suite 1726 passed / 17 skipped (cuDF), TOTAL coverage 100%; ruff/ty clean. No CHANGELOG entry (same unreleased path). Files: src/hiveplotlib/hiveplot.py, tests/hiveplot_test.py.

2026-07-07: Workstream C complete. Propagated the highlight overlay to the three vector backends (bokeh, holoviews, plotly) as kwarg translation over the shared `viz/base.py` prep, not a re-implementation. Added `viz/base.iter_overlay_node_placements(overlay)` (single-sourced "selected nodes only" logic: yields per-axis `(x, y)` arrays of the overlay's `nodes.data` ids, excluding the far-endpoint neighbor placements kept so incident edge curves draw); refactored WS B's matplotlib `_draw_overlay_nodes` onto it so all four backends share one filter. Each backend gained `_draw_highlight_overlays` / `_draw_highlight_overlay` / `_draw_overlay_nodes` called at the tail of `hive_plot_viz` (after base nodes), drawing overlay edges through the backend's own `edge_viz` (on the passed fig, `center_plot=False`, `hover=False`) then scattering only the selected nodes directly (bypassing `node_viz` so no neighbor node is accent-lit and no empty-axis build warning fires). Since the overlay `HivePlot` has its base node/edge styling cleared (WS B's `_highlight_overlay_plot`), the accent color wins with no new conflict warnings. Raised spotlight defaults translated per backend (matplotlib's 2.5 linewidth / 0.9 alpha / size-40 baseline): bokeh `line_width=2.5, alpha=0.9, node size=9`; plotly `line_width=2.5, opacity=0.9, node size=14`; holoviews `line_width`/`linewidth`=2.5, `alpha=0.9`, node `size=9` (bokeh sub-backend) / `s=70` (matplotlib sub-backend). holoviews composes edges via `edge_viz` then an accent `hv.Points(group="Highlights")` onto the returned overlay (both sub-backends), with an assert pinning that the overlay always carries axis-pair structure so `edge_viz` never early-returns None. Per-highlight `color=`/`edge_kwargs=`/`node_kwargs=` still override; P2CP paths untouched (guarded by a "still renders" regression per backend). Added 28 tests (7 bokeh, 7 plotly, 14 holoviews = 7 × both sub-backends): draws-overlay, incident-only-lights-selected-nodes, edge_kwargs/node_kwargs override, without-highlights-draws-no-overlay regression, no-conflict-warning, P2CP-unaffected. `viz/base.py` + `viz/bokeh.py` + `viz/holoviews.py` + `viz/plotly.py` at 100% coverage under their backend suites; 224 backend viz tests + 48 matplotlib highlight tests pass; ruff format/check + ty clean. No CHANGELOG entry (WS B's unreleased highlight bullet already covers the cross-backend capability; per house rule, no sibling entry for a refinement of a feature debuting in the same unreleased version). Files: src/hiveplotlib/viz/base.py, viz/bokeh.py, viz/holoviews.py, viz/plotly.py, viz/matplotlib.py, tests/viz_bokeh_test.py, tests/viz_holoviews_test.py, tests/viz_plotly_test.py.

2026-07-07: Workstream E complete (code portion; premise probe / notebook scouting is dispatcher-owned). Mirrored the `HivePlot` subset/highlight surface onto `HivePlotMatrix` as a pure broadcast over `iter_populated_cells`, reusing each panel's `HivePlot` methods (no re-implemented selection/closure). `HivePlotMatrix.subset(nodes=, edges=, hops=0) -> HivePlotMatrix` deepcopies the matrix then replaces each populated cell with that cell's frozen-geometry `HivePlot.subset(...)` (empty cells stay `None`, grid shape + labels + matrix_type + graph stashes preserved via the deepcopy); copy semantics, parent untouched. Highlight family broadcasts one selection to every panel under ONE shared key: `add_highlight(...)` resolves the shared key once (`_find_unique_highlight_key` = first integer free on EVERY panel via the union of used keys) and the accent color once at the matrix level (`_next_highlight_color` delegates to the first panel's palette walk so every panel shares the color, no per-panel drift), passes both explicitly to each panel's `add_highlight`, warns `HighlightKeyCollisionWarning` exactly once at the matrix level then suppresses the per-panel collision warnings, returns the one key; `update_highlight(key, ...)` / `remove_highlight(key)` broadcast to every panel holding the key (raising `KeyError` when the key is on no panel), bare `remove_highlight()` clears all panels; read-only `highlights` property collapses the per-panel registrations into one coherent `{key: representative Highlight}` view (fresh copy). `_UNSET`/`Highlight`/`HighlightKeyCollisionWarning` imported from `hiveplot` so the mirrored `update_highlight` sentinel semantics match exactly. Design note on the shared-key semantics: because color+key are resolved matrix-wide and forced onto each panel, `hpm.highlights[k]` is identical across panels, so the collapsed view is unambiguous. P2CP untouched (subset/highlight live on `HivePlot`, not the `BaseHivePlot` that `P2CP` wraps; guarded by a `hasattr` test). Docstrings lead with the cross-panel-comparison story ("where does this group behave uniquely vs not") and note `unify_axes=True` keeps subset panels comparable, complementary to frozen geometry. Added `TestSubsetBroadcast` (9), `TestHighlightBroadcast` (14), `TestP2CPUntouchedBySubsetHighlight` (1) = 24 tests (all `@pytest.mark.unmarked`; matplotlib is the only HPM backend that renders highlights, so the rendered-overlay + subset-render tests exercise the default path; a `sweep_matrix` fixture gives identical-axed panels for the axis-pair edge-selection tests, since `from_partition` panels have heterogeneous axes and a single axis pair is not universally valid there). `hiveplot_matrix.py` + `viz/hiveplot_matrix.py` at 100% coverage; 256 matrix tests pass; ruff format/check + ty clean. CHANGELOG: extended the existing WS A subset + WS B highlight bullets with a short "...also on HivePlotMatrix" clause each (no redundant sibling entries; same unreleased feature). Files: src/hiveplotlib/hiveplot_matrix.py, tests/hiveplot_matrix_test.py, CHANGELOG.rst.
2026-07-07: Workstream E post-impl battery fixes (api-critic + adversary, 4 items). (1, must-fix) Heterogeneous-axis axis-pair broadcast: `subset` now wraps the broadcast loop (single try/except around the loop tracking `(r, c)`, not per-iteration, to satisfy `PERF203`) to re-raise `InvalidAxisNameError` naming the offending `(row, col)` panel; the broadcast was already transactional (mutates a `deepcopy`, so a raise leaves the parent untouched). Left `add_highlight`'s plot-time crash unattributed to `(row, col)` per the coordinator's steer (the register-eagerly / raise-at-render asymmetry is consistent-by-design with single-`HivePlot`, and wrapping the HPM plot path for panel attribution is disproportionate); the docstring caveat carries it. Both `subset` and `add_highlight` docstrings gained a homogeneous-axis `.. note::` (axis-pair assumes every panel carries both axes, as `from_variable_sweep`/`from_tags` give; `from_partition` is heterogeneously axed, use ID/mask/expression there); `subset` gained `:raises SubsetBeforeBuildError:` + `:raises InvalidAxisNameError:` (the mirror had dropped the parent's block). (2, docstring) `highlights` view no longer over-claims "identical across panels": now conditioned on using the matrix-level add/update/remove, with a note that direct-panel editing (`hpm[r,c].update_highlight(...)`) can diverge a panel and the collapsed view hides it. (3, docstring) `_next_highlight_color` rewritten to the truth (reads the first populated panel; correct because the broadcast keeps every panel's color set in lockstep; dropped the false matrix-wide-union claim). (4, test) `test_subset_broadcasts_same_selection_to_every_panel` strengthened from `surviving <= selected` (passed even under a narrowed per-panel selection) to assert the matrix child equals a directly-computed `parent_panel.subset(...)` per panel. Added 2 tests: `test_subset_axis_pair_on_heterogeneous_axes_raises_transactionally` (subset raises naming `(row=0, col=0)`, parent intact) and `test_add_highlight_axis_pair_on_heterogeneous_axes_registers_then_plot_raises` (registers on every panel, then `plot()` raises). 258 matrix tests pass, `hiveplot_matrix.py` + `viz/hiveplot_matrix.py` at 100%; full suite 1752 passed / 17 skipped (cuDF), TOTAL 100%; ruff format/check + ty clean; docstrings parse as valid RST. No CHANGELOG change. Files: src/hiveplotlib/hiveplot_matrix.py, tests/hiveplot_matrix_test.py.
2026-07-07: Workstream E ty-narrowing follow-up (caught by the WS F whole-branch qa, missed by the WS E per-workstream qa). The two axis-pair tests added above each asserted `partition_matrix[0, 0] is not None` then re-invoked `partition_matrix[0, 0].axes`; ty could not narrow `HivePlot | None` across the second `__getitem__` call (a fresh return), so `ty check` reported 2 `unresolved-attribute` diagnostics (`tests/hiveplot_matrix_test.py:4977` and `:5172`). Fixed by binding the panel to a local `panel = partition_matrix[0, 0]` once, narrowing that (`assert panel is not None`), and accessing `panel.axes` (no `# ty: ignore`); the subsequent `pytest.raises` / `add_highlight` lines still reference `partition_matrix`. `ty check` now reports 0 diagnostics; both tests still pass and assert the same thing; ruff format/check clean. Files: tests/hiveplot_matrix_test.py.
2026-07-07: Workstream F complete (code portion; side-by-side docs recipe deferred to WS H's highlight notebook, the stronger narrative host). Added `HighlightsUnsupportedByDatashaderError` (new plain-`Exception` subclass in `exceptions/hive_plot.py`, re-exported via the `exceptions` package `*` import) and a raise-and-point guard on all three datashade entry points, per Amendment 20's spike-rejected-overlay decision (accent overlay washes out structurally at datashader density; separate panel is redundant with `subset()`). New module-level `_guard_no_highlights(instance)` in `viz/datashader.py` raises the error early (before `input_check`) when `getattr(instance, "highlights", None)` is truthy; called at the top of `datashade_edges_mpl`, `datashade_nodes_mpl`, and `datashade_hive_plot_mpl` so each entry point guards independently. `getattr` (not bare `.highlights`) is load-bearing: the registry lives on `HivePlot` only, so `BaseHivePlot`/`P2CP` (and non-instances like `Node` in the wrong-types test) pass the guard as a no-op instead of `AttributeError`-ing. The raised message names both real paths: `subset(...)` for a datashaded drill-down (same selection vocabulary, including `hops`) and a non-datashader backend (matplotlib/bokeh/plotly) for the overlay; each entry point gained a `:raises:` docstring line. Added 8 datashader-marked tests: 3 parametrized raise tests (one per entry point, asserting the message points at `subset`), 3 parametrized no-highlights regression renders (unchanged through all three), a subset-child-datashades verification (`hp.subset(...)` then `datashade_hive_plot_mpl`, zero new per-backend code), and the Dask-lazy-edge subset-child render (`LazyEdgeSubset` rasterizes straight from stored ids). 108 datashader tests pass; `exceptions/hive_plot.py` at 100%; ruff format/check + ty clean; docs render the new exception via the existing `automodule`. CHANGELOG: one bullet under the existing "Datashader Backend" Added section (net-new user-visible guard; does not duplicate the highlight bullet). Files: src/hiveplotlib/viz/datashader.py, src/hiveplotlib/exceptions/hive_plot.py, tests/datashader_test.py, CHANGELOG.rst.
2026-07-07: Workstream F post-impl battery fixes (api-critic + adversary, message-honesty + test-hardening). Message/source: added `holoviews` to the overlay-backend enumeration in BOTH the `_guard_no_highlights` message and the exception docstring (WS C shipped the overlay on holoviews too); added a third path — `HivePlot.remove_highlight()` to datashade the plain base view (the only unblocking move for the "I just want the datashaded base" user, since `subset` narrows and switching backends abandons datashader) — to the guard message, the exception docstring, and all three entry-point `:raises:` lines; folded in a half-clause noting `subset` defaults `closure="induced"` while `add_highlight` defaults `"incident"` (a redirected user omitting `closure=` gets a different edge set). Tests: added `test_datashade_registry_less_instances_never_guard` (6 = `BaseHivePlot` × `P2CP` × 3 entry points, single-tag fixtures so the render is warning-free under warnings-as-errors) pinning the `getattr(..., None)` no-op on registry-less types directly (covered only transitively before); hardened `test_datashade_subset_child_over_lazy_dask_edges_renders` to assert every stored edge id set on the subset child is a `LazyEdgeSubset` before the render (pins the "incl. lazy Dask" half of the zero-new-code claim). Full suite 1766 passed / 17 skipped (cuDF), TOTAL 100%; 114 datashader tests; `exceptions/hive_plot.py` 100%; ruff format/check + ty clean; docs build clean (new exception's `:py:meth:` cross-refs render). CHANGELOG bullet updated for accuracy (holoviews + `remove_highlight()` path), still within the 4-line cap. Files: src/hiveplotlib/viz/datashader.py, src/hiveplotlib/exceptions/hive_plot.py, tests/datashader_test.py, CHANGELOG.rst.
2026-07-07: Workstream G complete. New `examples/subsetting_hive_plots.ipynb` gallery notebook (reference-style, single feature: `HivePlot.subset`). Scouted the real bundled `example_trade_nodes_and_edges` (Harvard Growth Lab 2019 specialty-metal trade) restricted to 3 continents (Asia/Europe/North America, 107 nodes / 948 edges), partitioned by `continent`, sorted by a rank transform of `export_value` (stated plainly in prose: raw value is right-skewed and crushes nodes to the center; edges untouched), built with `repeat_axes=True` + a muted darkgray/thin base so drill-downs carry the color. Drill-down arc: full busy plot → rhetorical "who has a lopsided neighborhood" → hero `hp.subset(nodes="PHL", hops=1)` ego (12 nodes: 8 Asia / 3 Europe / 1 North America; intra-Asia cluster lit darkorange via `update_edges("Asia","Asia",...)`, flexitext title lands the Asia-concentration insight) → value-threshold backbone `hp.subset(nodes=nw.col("export_value") > q60)` (43 nodes, mask twin shown selecting the identical subgraph, frozen geometry called out against the parent) → "How Selections Combine" prose pinning the union algebra (nodes+edges in one call = union; chained calls = intersect), induced-closure-is-drawable-safe, ego precision (`hops=1` = undirected `networkx.ego_graph`, neighbor-neighbor edges included), and pointer to `add_highlight` for emphasize-without-removing. Every subset keeps all 3 axes populated (no empty-axis warning; scout-verified). Runs GREEN end-to-end via `KERNEL_NAME=python3 .venv/bin/python -m pytest -c tests/pytest_examples.ini -k subsetting` (warnings-as-errors, 1 passed); executed in place with outputs embedded (3 figures, 0 errors, 0 stderr); ruff check + format clean at line-length 79. Registered in `docs/source/gallery_examples/index.rst` (HivePlot Class section, both blocks), added a text-stripped ego thumbnail (`docs/source/_static/subsetting_hive_plots.jpg` + `conf.py` mapping), an `llms.txt` `## Optional` HivePlot-class feature-demo entry, and a CHANGELOG Documentation bullet. Files: examples/subsetting_hive_plots.ipynb, docs/source/gallery_examples/index.rst, docs/source/conf.py, docs/source/_static/subsetting_hive_plots.jpg, docs/source/_llms/llms.txt, CHANGELOG.rst.
2026-07-07: Workstream G prose battery fixes (editorial + adversary; anecdote gate PASSED, MARKDOWN-ONLY, no code/figure/dataset change, notebook stays green). Reframed the ego anecdote off a factually-false "only thin spokes to Europe and North America" (the styling category read back as a volume fact) onto the honest, richer story: the concentration is by PARTNER COUNT (7 of 11 partners Asian), the orange bundle is a *category* marker at uniform width (added a one-line "width shows category here, not volume" note), and the single largest flow in the whole ego is verified to be the Philippines' export to the US (8.27M > any intra-Asia flow incl. PHL-JPN 6.78M), now surfaced as the standout exception with a pointer to Visualizing Edge Metadata for volume-styling; new flexitext title "Most of the Philippines' trade partners are Asian, but its single largest flow is to the United States". Fixed the hero mis-attribution ("cluster of Asian trade partners" -> "intra-Asia trade among the Philippines and its partners", consistent with the hops=1 induced-closure teaching). Killed the backbone TAUTOLOGY: reframed "## Selecting by Value / high-volume backbone / finding" as "## Selecting by a Data Column", making expression+mask the point, honestly attributing the axis-tip clustering to the sort order showing through (verified the composition skew is too thin to call structure: Europe 63% / Asia 37% / NA 12% survival). Reordered the value beat to plot right after the expression cell with the mask equivalence as a coda (fix #6); folded the union/intersection rules into a "## Where to Go Next" close rather than a demo-less "How Selections Combine" heading (fix #5). Added the axis-variable caveat (a node's export_value is total exports across ALL destinations, computed before the 3-continent filter, so placement can reflect trade not drawn; fix #4). Re-ran GREEN via KERNEL_NAME=python3 pytest -k subsetting (1 passed, warnings-as-errors); re-executed in place (3 figures, 0 errors, 0 stderr); reframed ego + value figures render clean (title clears the top axis label); ruff check+format clean, 0 em-dashes. Registration files (index.rst, conf.py, thumbnail, llms.txt, CHANGELOG) unchanged (name/feature/registration identical; thumbnail strips the title so the styling-identical figure needs no regen). Files: examples/subsetting_hive_plots.ipynb.
2026-07-07: Workstream G ego-headline directional-error fix (editorial re-check; MARKDOWN-ONLY, notebook stays green). The prior reframe introduced a DIRECTION error: this is a directed EXPORTS network, so the 8.27M `USA,PHL` flow is the US exporting TO the Philippines (INBOUND), not the Philippines exporting to the US (`PHL,USA` = 16,254, near the smallest edge); the Philippines' own largest export is to Japan (`PHL,JPN` = 6.78M, intra-Asia). The title's "its single largest flow is to the United States" was therefore false. Root cause of the churn: the figure colors by CATEGORY (gray vs orange), not volume, so no volume claim is supported by the picture. Decisive fix: anchored the anecdote on PARTNER COMPOSITION (the thing the drill-down actually shows) — new flexitext title "Most of the Philippines' trade partners are Asian: a tight Asian cluster with a few spokes to Europe and North America" (no volume/direction claim). Kept the numeric nuance in the body but stated correctly and labeled as a data-table fact, not a picture fact: "the Philippines' own exports run to Asia (its largest is to Japan), yet the single biggest flow in the neighborhood runs the other way, the United States exporting to the Philippines"; the picture-shows-who-not-how-much + Visualizing-Edge-Metadata pointer retained. Re-ran GREEN via KERNEL_NAME=python3 pytest -k subsetting (1 passed, warnings-as-errors); re-executed in place (3 figures, 0 errors, 0 stderr); reframed ego figure renders clean (title clears the top axis label, color tokens on Philippines'/Asian correct); ruff check+format clean, 0 em-dashes. Files: examples/subsetting_hive_plots.ipynb.
2026-07-07: WS B/C highlight overlay edge-linewidth default retune (maintainer directive, Amendment 23). Lowered the raised overlay edge linewidth from 2.5 to 2.0 across all four backends (`_OVERLAY_EDGE_LINEWIDTH` matplotlib; `_OVERLAY_EDGE_LINE_WIDTH` bokeh / plotly / holoviews); node size (40) and edge alpha/opacity (0.9) held per the maintainer. Render-tested on the dense 3-continent trade network (`example_trade_nodes_and_edges`, `add_highlight(nodes="CHN")`, 79 incident overlay segments) sweeping 1.75/1.9/2.0/2.1/2.5 PNGs: at 2.5 the Europe-Asia bundle merges into a clunky red slab; 2.0 keeps the strands distinct while the accent still reads unmistakably as emphasis (carried by the accent color + retained 0.9 alpha + size-40 nodes), and 1.9 starts to lose the thinnest strands into the gray base. Landed exactly on 2.0 (>base 1.5). Updated the two hardcoded test literals (`hiveplot_test.py` matplotlib `assert np.allclose(..., 2.0)`; `viz_holoviews_test.py` overlay-collector filter `== 2.0`); the bokeh/plotly collectors reference the module constant and auto-follow; the `edge_kwargs=linewidth=5` override tests still pass (per-highlight override beats the new default). Full suite 1766 passed / 17 skipped (cuDF), TOTAL 100%; ruff format/check + ty clean. No CHANGELOG entry (default tweak to the unreleased highlight feature; no prose cites 2.5). Files: src/hiveplotlib/viz/matplotlib.py, bokeh.py, plotly.py, holoviews.py, tests/hiveplot_test.py, tests/viz_holoviews_test.py.
2026-07-07: Workstream H complete. New `examples/highlighting_hive_plots.ipynb` gallery notebook (reference-style, single feature: `HivePlot.add_highlight`). Same scouted bundled `example_trade_nodes_and_edges` recipe as the WS G sibling (Harvard Growth Lab 2019 specialty-metal trade, 3 continents Asia/Europe/North America, 107 nodes / 948 edges, partitioned by `continent`, sorted by an export-value rank transform stated honestly, `repeat_axes=True` + a muted darkgray/thin base as the practiced de-emphasis move). Emphasize-in-context arc, every claim scout-verified against the raw CSV: hero `add_highlight(nodes="CHN")` (default blue `#0072B2`, `closure="incident"`) reads as China's BREADTH — 53 distinct partners across all three axes (21 Asia / 28 Europe / 4 North America), an honest who/where/count claim, never a volume claim (uniform overlay width called out); a second `add_highlight(nodes="MEX")` (default vermillion `#D55E00`) as the focused-regional-player contrast (12 incident edges, 10 partners); `update_highlight(key, color=...)` recolor + `remove_highlight(key)` drop-one, with `remove_highlight()` clear-all named; incident-vs-induced on the Mexico ego (`hops=1`: 637 incident vs 100 induced, context-vs-cohesion), rendered as a 1x2 `.copy()` pair. GIF-sweep gate decision (recorded in-notebook + here): static small-multiples STRIP, not an animated GIF. Basis: each frame is one `hp.subset(edges=nw.col("export_value") > thr)` + render from the same frozen parent (CI-cheap, renders in nbsphinx docs); an actual GIF needs `matplotlib.animation` + a pillow/imagemagick writer, inflates notebook output bytes, and adds motion not information. The strip (thresholds 0 / 0.5M / 2M / 5M) surfaces the scouted relationship: North America's nodes thin FASTEST (retains 8% of its countries at >5M vs Europe 32% / Asia 21%), so the high-value backbone is Europe-Asia, verified against the data (fraction-retained per continent). GIF-writer recipe described once in prose for out-of-docs use. Datashader side-by-side coda (deferred from WS F / Amendment 20): `datashade_hive_plot_mpl(hp)` beside `datashade_hive_plot_mpl(hp.subset(nodes="CHN", hops=1))` in a shared 1x2 figure, since the datashader backend RAISES `HighlightsUnsupportedByDatashaderError` on registered highlights (verified); the subset carries frozen geometry so China's ego sits where it did in the full raster. All 6 figures inspected. Runs GREEN via `KERNEL_NAME=python3 .venv/bin/python -m pytest -c tests/pytest_examples.ini -k highlighting` (warnings-as-errors, 1 passed, 11.28s); executed in place (6 figures, 0 errors, 0 stderr beyond the benign TCP line); ruff check + format clean at line-length 79 (`strict=True` on the sweep `zip`). Registered: `docs/source/gallery_examples/index.rst` (HivePlot Class, both blocks, after subset), `conf.py` thumbnail mapping + text-stripped China+Mexico thumbnail `docs/source/_static/highlighting_hive_plots.jpg` (385x385, orthogonal to the subset thumb), `llms.txt` `## Optional` HivePlot-class feature-demo entry, and a CHANGELOG Documentation bullet. Files: examples/highlighting_hive_plots.ipynb, docs/source/gallery_examples/index.rst, docs/source/conf.py, docs/source/_static/highlighting_hive_plots.jpg, docs/source/_llms/llms.txt, CHANGELOG.rst.
2026-07-07: Workstream H prose battery fixes (coordinator, 4 items; MARKDOWN + in-cell verification only, no feature/figure/dataset change, notebook stays green). (1, structure-is-artifact) Reframed the sweep analysis prose off the bare "the high-value backbone is Europe and Asia" finding onto the NAMED mechanism: North America's axis here is mostly very small economies (USA hub + a Caribbean/Central-American micro-state tail), and 16 of its 26 countries export $0 in this dataset (verified: NA median export_value = 0, 16 zeros vs Europe 5 / Asia 20; Europe median $472k), so the axis empties fastest by DENOMINATOR COMPOSITION, not because NA is peripheral to the network. (2, pre-filter caveat parity with the subset sibling) Added one sentence to "The Base Plot": a country's export_value (which sets its axis rank/position) is total exports across ALL destinations, computed before the 3-continent filter, so placement can reflect trade not drawn. (3, percentages data-computed not eyeballed) The sweep cell now computes per-continent survival share at the top threshold from the frozen subset (`top.nodes.data["continent"].value_counts()` over `node_df` base counts) and prints them; the prose reports those computed values (verified exact: Asia 20.9% / Europe 31.6% / NA 7.7%, rounding to the printed 21% / 32% / 8%). (4, stray output) The two `add_highlight` key cells now `print(f"registered ... under key {k}")` instead of echoing a bare `0`/`1`; suppressed the incident/induced setup cell's stray `0` with a trailing `;`. `list(hp.highlights)` -> `[0]` kept (meaningful: China's key survives the Mexico removal). Re-ran GREEN via `KERNEL_NAME=python3 .venv/bin/python -m pytest -c tests/pytest_examples.ini -k highlighting` (1 passed, 10.95s, warnings-as-errors); executed in place (6 figures, 0 errors, printed shares match prose); sweep figure + reframed prose cohere (the two NA axes visibly empty first); ruff check + format clean at line-length 79; 0 em-dashes. Registration/thumbnail/llms.txt/CHANGELOG unchanged. Files: examples/highlighting_hive_plots.ipynb.
2026-07-07: Workstream J complete (real lazy `hops>0`; NOT cut). Replaced the WS A floor's in-memory error for `hops>0` on lazy (Dask) edge frames with faithful, non-materializing hop expansion, the iterated form of the WS D bounded-endpoint pattern. Added `frames.lazy_edge_frontier_endpoint_ids(data, node_ids, from, to)`: one bounded Dask compute per call that filters the lazy edge frame to edges touching the current node set (`is_in(from, ids) | is_in(to, ids)`), selects only the two endpoint columns, and collects the unique endpoints (never the full edge frame; the result is a node-ID set that fits in RAM by the WS A ceiling). Rewrote `HivePlot._expand_node_ids_by_hops` to loop per hop over a new `_frontier_neighbor_ids(current)` that unions neighbors across all edge tags: in-memory tags scan endpoint id arrays with numpy (`export_edge_array(tag=...)` per tag, no all-tag materialize), each lazy tag runs the bounded compute; the grown node set then feeds the SAME closure/predicate pipeline (`_restrict_to_node_ids` -> `_restrict_lazy_subset` composes the final `is_in` predicate lazily). Deleted the two `edges_are_lazy and hops > 0 -> _raise_lazy_mask_selection_error()` gates in `subset` and `_highlight_overlay_plot`; the concrete-mask gate stays (only remaining caller). Retuned the mask error message + docstring to drop the "or hops>0" clause and note `hops` now expands over bounded endpoint scans; updated the `subset` `.. note::` / `:raises:` and the `_highlight_overlay_plot` `:raises:`. Flipped the two WS A floor tests from "hops>0 raises" to twin-comparison completeness pins (`test_subset_lazy_edges_hops_greater_than_zero_matches_in_memory_and_stays_lazy`, `test_highlight_overlay_lazy_edges_hops_greater_than_zero_matches_in_memory`): each builds a lazy child and its in-memory twin on the same seed + `hops=2` (induced), asserts identical materialized edge set (`_materialized_edge_id_set`, non-empty) and surviving node set, and asserts every stored subset stays a `LazyEdgeSubset` (never full-materialized). Confirmed no materialize: only the ID columns are scanned per hop; the edge frame is never realized (the twin tests assert the lazy records persist). No Dask warts hit (the scaling-sprint arrow-string / thread-safety / kernel-serialization classes did not recur; the `nw.from_native(frame.to_native()).filter(...).select(...).collect()` shape mirrors the already-working WS D helper). 100% coverage on the new lines (in-memory branch of `_frontier_neighbor_ids` covered by existing non-lazy hops tests, lazy branch + the new frames helper by the twin tests); all 12 lazy subset/highlight tests pass, 44 dask tests pass, 112 Subset/Highlight tests pass; ruff format/check + ty clean. No CHANGELOG change (upgrades an unreleased feature: the 0.29.0 subset bullet already says "expanding by `hops`" and the lazy-stays-lazy bullet is now strictly more true; rule 13 override for same-version refinements). Files: src/hiveplotlib/frames.py, src/hiveplotlib/hiveplot.py, tests/hiveplot_test.py.
