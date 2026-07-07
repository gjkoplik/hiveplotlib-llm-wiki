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
clump = hp.nodes.data["high"] > 25          # boolean mask over node data
sub_hp = hp.subset(nodes=clump)              # induced: keeps edges with BOTH endpoints in the clump
fig, ax = hive_plot_viz(sub_hp)              # a real HivePlot; placements and axis ranges frozen to parent
```

```python
# Example 2: zoom to one node's neighborhood (ego story), or to one axis pair's edges
# Example data: reuses hp from Example 1.

# Call site:
ego = hp.subset(nodes=[17], hops=1)          # node 17, its neighbors, and the connecting edges
a_to_b = hp.subset(edges=("A", "B"))         # edges between axes A and B, plus incident nodes only
```

```python
# Example 3: emphasize in context; register, restyle, inspect, remove named highlights
# Example data: reuses hp from Example 1.

# Call site:
key = hp.add_highlight(nodes=hp.nodes.data["high"] > 28)     # nodes + ALL touching edges (incident default)
core = hp.add_highlight(
    nodes=hp.nodes.data["med"] > 15,
    closure="induced",                                        # narrow: only edges within the group
)
fig, ax = hive_plot_viz(hp)                  # base layer unchanged; overlays drawn on top, one accent color each
hp.update_highlight(key, color="goldenrod")  # restyle by key
hp.highlights                                # inspect registered highlights
hp.remove_highlight(core)
```

```python
# Example 4: condition shorthand + the sweep-to-GIF pattern (per-call cheapness is the point)
# Example data: reuses hp from Example 1; edges carry "low" / "med" / "high" columns.
import matplotlib.pyplot as plt
import narwhals as nw

# Call site:
dense = hp.subset(edges=nw.col("low") > 5)   # filter on a variable not used for partition or sorting

for i, threshold in enumerate([2, 4, 6, 8]):
    frame_hp = hp.subset(edges=nw.col("low") > threshold)
    fig, ax = hive_plot_viz(frame_hp)        # identical geometry every frame; stitch to GIF downstream
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
key = hpm.add_highlight(nodes=node_df["high"] > 28)          # same selection lit in every panel
sub_hpm = hpm.subset(nodes=node_df["group"] == "A")          # broadcast subset, panel geometry frozen
```

```python
# Example 6: dense-edges story at scale; a subset child is a real HivePlot, so datashader just works
# Example data: reuses hp and dense from Example 4.
from hiveplotlib.viz.datashader import datashade_hive_plot_mpl

# Call site:
fig, ax, im_nodes, im_edges = datashade_hive_plot_mpl(instance=dense)   # zero new per-backend code
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

## Notebook review

```
Pending — invoke editorial-critic in post-implementation mode after Workstreams G and H ship.
```

## Viz review

```
Pending — invoke viz-critic in post-implementation mode after Workstreams F, G, and H ship
(overlay palette and layering judged against the viz-quality-bar skill).
```

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
- Evaluates across pandas / polars / cuDF via narwhals dispatch; on lazy Dask edge frames the expression composes into the lazy membership predicate and never materializes.
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

### Workstream F: datashader highlight (self-contained, revertible)

**Status:** not started
**Files:** `src/hiveplotlib/viz/datashader.py`, matching tests
**Done when:**

- Open design item resolved first (spike + maintainer sign-off): light-colormap overlay raster z-ordered on top of the base raster vs highlight rendered as a separate panel; decision recorded here before implementation proceeds.
- Chosen design implemented for `datashade_hive_plot_mpl` (and the edge/node variants as applicable) with datashader markers and tests.
- Subset-on-datashader verified by test in this workstream: a subset child renders through the datashader route with zero new per-backend code, including the Dask-lazy edge route (nothing special needed is a claim; test it).
- ADR 0002 gates: any rasterization-path change clears the equivalence wall and same-run ratio gates.
- Ships as its own revertible commit.

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

**Status:** not started
**Files:** `src/hiveplotlib/hiveplot.py`, matching tests, doc touch-ups
**Done when:**

- Justification (only non-redundant residual): selection by normalized PLACEMENT along an axis. Value ranges are already expressible once WS D ships (`nw.col(...).is_between(...)` unioned with an axis pair); placement-position selection is the genuinely new surface and is I's reason to exist. Disposes the adversary's cold item 6.
- Select by region: an axis pair plus a placement range per axis ("that patch between A and B"), feeding the same subset/highlight machinery; exact parameter shape proposed to api-critic before implementation.
- Ships as its own revertible commit; **explicitly allowed to be cut** if it doesn't pan out (record the cut as a deferred follow-up, not a silent drop).

### Workstream J: lazy `hops>0` support on Dask edge frames (self-contained, revertible, sequenced late, cuttable)

**Status:** not started
**Files:** `src/hiveplotlib/hiveplot.py` and/or selection helpers, `src/hiveplotlib/frames.py` if the lazy route needs a hook, matching tests
**Done when:**

- Real lazy `hops>0` support attempted: per-hop `is_in` / `unique` / `compute` over the endpoint-ID columns only, honoring the no-full-materialize invariant (the node set is in-RAM by the existing WS A ceiling, so only ID columns are scanned; the `LazyEdgeSubset` frame is never materialized). Replaces the WS A floor's in-memory error for `hops>0` on lazy frames when it lands.
- Save-before-cut rule: on trouble, first diagnose whether it is the same quickly-fixable Dask wart class the scaling sprint already beat (arrow-string, thread-safety, kernel serialization) before degrading back to the WS A `_require_in_memory_edge_subsets` error; only a genuine dead-end cuts this workstream, recorded as a deferred follow-up (not a silent drop).
- Ships as its own revertible commit; if cut, WS A's floor error stays as the shipped behavior for lazy `hops>0`.
- 100% coverage with the `@pytest.mark` markers the Dask route needs; ADR 0002 perf gates apply if any rasterization/curve path is touched (not expected; this is ID-column scanning).
- Disposes the adversary's cold item 4 (real-support half).

### API Critic — post-implementation review (Workstream J)

```
Pending — invoke api-critic in post-implementation mode after Workstream J ships (it changes what subset/highlight
do on lazy edge frames, so the user-facing behavior gets the pass).
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

## Holdouts

None. The four TODOs have no legitimate survivors; no other old-pattern text exists.

## Implementation log

2026-07-07: Workstream A complete. Added `HivePlot.subset(nodes=, edges=, hops=0)` that slices already-computed geometry (deepcopy then filter `nodes`/`edges`/per-axis `node_placements`/`hive_plot_edges` ids+curves) into an induced-closure child, freezing axis `vmin`/`vmax` as explicit (`inferred_*=False`); pipeline resolves node seeds + edge-seed endpoints → union → `hops`-expand → induced. Selection vocabulary: node IDs/scalar ID/list, boolean mask (numpy or narwhals-supported Series via `_is_boolean_mask`), narwhals expression, axis pair / list of pairs, single-tag bare edge mask, multi-tag mask/expression dict (absent tag = not selected). Lazy (Dask) edges: `hops=0` ID/axis-pair composes `is_in(surviving_ids)` into the stored predicate (never materializes); mask/expression or `hops>0` on lazy edges → `_raise_lazy_mask_selection_error`; lazy node data raises (RAM ceiling, docstring). Rebuilt `relevant_edges` positionally against filtered frames; added `_slice_curves` (reshape-mask-flatten). New `SubsetBeforeBuildError` (pre-geometry guard, distinct from renderable empty selection). Copy-not-recompute pinned by a spy test on `construct_curves`/`_construct_edge_subset_curves`/`_curves_from_id_pairs`. Deleted the four `data_subset`/`data_highlight` TODOs (node.py, edges.py). Added `TestSubset` (48 tests incl. polars + dask markers, non-RangeIndex positional-vs-label, frozen-geometry array-equality, union/intersection algebra) and a CHANGELOG.rst "Hive Plots" bullet. New subset source lines at 100% coverage; ruff/ty clean; docs render `HivePlot.subset` + `SubsetBeforeBuildError`. `edge_coverage` (from the sibling edge-coverage plan) is not present on branch 53, so its done-when is deferred to when that method lands (child holds its own filtered edges, so it needs no special-casing). Files: src/hiveplotlib/hiveplot.py, node.py, edges.py, exceptions/hive_plot.py, tests/hiveplot_test.py, CHANGELOG.rst.

2026-07-07: Workstream A post-impl gate fixes (api-critic + adversary). Added a shared `_require_full_length_row_mask` guard wired into both `_resolve_node_seed_ids` and `_edge_frame_mask` (reused by WS B/D/E resolvers): a selection expression must evaluate to one boolean per row or it raises a legible `ValueError` (edge path names the tag), closing the node-path raw `IndexError` and the aggregating-edge-expression "wrong subgraph" silent-mis-selection. Added a `hops >= 0` guard (`ValueError`), the missing `:raises: InvalidAxisNameError` docstring line, and docstring clarifications: `nodes=` list/array-vs-tuple-ID + silent-drop-of-unknown-IDs, softened ego wording to "undirected neighborhood; equivalent to `networkx.ego_graph` on an undirected view", and the lazy axis-pair asymmetry. Added 5 tests (node/edge aggregating-expression raises, wrong-length edge mask, multi-tag-dict aggregating-expression names tag, negative hops). Full suite 1636 passed / 14 skipped (cuDF), TOTAL coverage 100%; ruff/ty clean.

2026-07-07: Workstream B complete. Generalized WS A's resolver for closure (one implementation, not two): extracted `_resolve_selection_node_ids(nodes, edges, hops)` (seeds→union→hops, shared by `subset` and highlight) and threaded a `closure: Literal["induced","incident"]` param through `_restrict_to_node_ids` / `_restrict_edges_to_node_ids` / `_restrict_lazy_subset` (induced = both endpoints AND; incident = either endpoint OR); incident keeps far-endpoint node placements (new `_incident_placement_ids`) so overlay curves stay drawable while node data holds only the selected set. `subset` still locks induced. Registry lands on `HivePlot` (not `BaseHivePlot`, so `P2CP` never inherits it): `_highlights` dict init in `__init__`, `add_highlight(nodes=, edges=, hops=0, closure="incident", color=None, key=None, node_kwargs=None, edge_kwargs=None) -> key` (auto integer key via `_find_unique_highlight_key`; collision warns `HighlightKeyCollisionWarning` at `stacklevel=3`; combines nodes+edges into one keyed group), `update_highlight(key, ...)` (full add kwarg set via a `_UNSET` sentinel so omitted keeps / explicit `None` clears), `remove_highlight(key=None)` (bare call clears all — the chosen clear-all affordance), read-only `highlights` property returning a fresh dict of new `Highlight` dataclass records (re-exported from `hiveplotlib/__init__`, mirroring the EdgeCoverage re-export pattern). Palette: new `HIGHLIGHT_ACCENT_PALETTE` (6 Okabe-Ito colorblind-safe hues, assigned first-unused via `_next_highlight_color`, wraps when exhausted; per-highlight `color=` override respected). matplotlib overlay: `_highlight_overlay_plot` builds a throwaway restricted `HivePlot` (clears its registry + base node/edge styling so the accent color wins with no new conflict warnings), `viz/base.iter_highlight_overlays` shared-prep yields `(highlight, overlay)` per backend (BaseHivePlot/P2CP yield nothing), `hive_plot_viz` draws overlay edges (zorder 6) then nodes (zorder 7, via `_draw_overlay_nodes` to skip the empty-axis build warning) strictly above the base; base-without-highlights render is byte-for-byte unchanged (regression test). `node_kwargs`/`edge_kwargs` passthrough SHIPPED (near-free through the viz machinery). Render-time semantics + subset-inherits-no-highlights + copy-survives documented. New `HighlightKeyCollisionWarning` (exceptions/hive_plot.py). Added `TestHighlight` (40 tests: registry lifecycle, palette first-unused/skip/wrap/override, incident-vs-induced via the shared resolver, overlay zorder + byte-for-byte base regression + no-conflict-warning + accent color + kwargs passthrough + multiple/empty/single-node overlays, copy/subset interplay, BaseHivePlot no-registry + P2CP renders, non-RangeIndex ID selection, narwhals expression, dask hops-0-stays-lazy / hops>0-raises). New source 100% covered; ruff/format/ty clean; docs render `add_highlight`/`update_highlight`/`remove_highlight`/`highlights`/`Highlight`/`HIGHLIGHT_ACCENT_PALETTE`/`HighlightKeyCollisionWarning`. Files: src/hiveplotlib/hiveplot.py, viz/base.py, viz/matplotlib.py, exceptions/hive_plot.py, __init__.py, docs/source/autodoc/hive_plots/high_level_hive_plot_api.rst, tests/hiveplot_test.py, CHANGELOG.rst.

2026-07-07: Workstream B post-impl battery fixes (api-critic + viz-critic + adversary, 8 items). Incident overlay now accent-lights only the SELECTED nodes: `_draw_overlay_nodes` filters the scatter to `overlay.nodes.data` ids so far-endpoint placements (kept only so touching edge curves draw) get no dot (was lighting every neighbor). Raised overlay spotlight defaults (`_OVERLAY_EDGE_LINEWIDTH=2.5`, `_OVERLAY_EDGE_ALPHA=0.9`, `_OVERLAY_NODE_SIZE=40`) so a highlight spotlights rather than merely recolors; per-highlight `edge_kwargs=`/`node_kwargs=` still override (extracted `_draw_highlight_overlay` per-overlay helper). `key` is now keyword-only on `add_highlight` (a positional key was silently binding to `edges=`). Added `_validate_highlight_closure` (legible `ValueError` on a bad `closure` like the `"induce"` typo) wired into both `add_highlight` and `update_highlight`. Guarded `update_highlight(hops=None)` (was a raw `TypeError` from `None<0`); scoped the "None clears" docstring to the selection fields (`nodes`/`edges`) only. Docstrings: `highlights` records echo the RAW selection not the resolved set; `add_highlight` dense-overlay caveat + palette-tuned-against-dark-base note. Added 8 `TestHighlight` tests: `test_plot_incident_highlight_does_not_accent_light_neighbor_nodes`, `test_plot_highlight_overlay_uses_raised_spotlight_defaults`, `test_plot_highlight_edge_kwargs_override_raised_defaults`, `test_add_highlight_key_is_keyword_only`, `test_add_highlight_invalid_closure_raises`, `test_update_highlight_invalid_closure_raises`, `test_update_highlight_none_hops_raises`, `test_highlight_non_range_index_mask_selection`. Full suite 1684 passed / 14 skipped (cuDF), TOTAL coverage 100%; ruff format/check + ty clean.

2026-07-07: Workstream C complete. Propagated the highlight overlay to the three vector backends (bokeh, holoviews, plotly) as kwarg translation over the shared `viz/base.py` prep, not a re-implementation. Added `viz/base.iter_overlay_node_placements(overlay)` (single-sourced "selected nodes only" logic: yields per-axis `(x, y)` arrays of the overlay's `nodes.data` ids, excluding the far-endpoint neighbor placements kept so incident edge curves draw); refactored WS B's matplotlib `_draw_overlay_nodes` onto it so all four backends share one filter. Each backend gained `_draw_highlight_overlays` / `_draw_highlight_overlay` / `_draw_overlay_nodes` called at the tail of `hive_plot_viz` (after base nodes), drawing overlay edges through the backend's own `edge_viz` (on the passed fig, `center_plot=False`, `hover=False`) then scattering only the selected nodes directly (bypassing `node_viz` so no neighbor node is accent-lit and no empty-axis build warning fires). Since the overlay `HivePlot` has its base node/edge styling cleared (WS B's `_highlight_overlay_plot`), the accent color wins with no new conflict warnings. Raised spotlight defaults translated per backend (matplotlib's 2.5 linewidth / 0.9 alpha / size-40 baseline): bokeh `line_width=2.5, alpha=0.9, node size=9`; plotly `line_width=2.5, opacity=0.9, node size=14`; holoviews `line_width`/`linewidth`=2.5, `alpha=0.9`, node `size=9` (bokeh sub-backend) / `s=70` (matplotlib sub-backend). holoviews composes edges via `edge_viz` then an accent `hv.Points(group="Highlights")` onto the returned overlay (both sub-backends), with an assert pinning that the overlay always carries axis-pair structure so `edge_viz` never early-returns None. Per-highlight `color=`/`edge_kwargs=`/`node_kwargs=` still override; P2CP paths untouched (guarded by a "still renders" regression per backend). Added 28 tests (7 bokeh, 7 plotly, 14 holoviews = 7 × both sub-backends): draws-overlay, incident-only-lights-selected-nodes, edge_kwargs/node_kwargs override, without-highlights-draws-no-overlay regression, no-conflict-warning, P2CP-unaffected. `viz/base.py` + `viz/bokeh.py` + `viz/holoviews.py` + `viz/plotly.py` at 100% coverage under their backend suites; 224 backend viz tests + 48 matplotlib highlight tests pass; ruff format/check + ty clean. No CHANGELOG entry (WS B's unreleased highlight bullet already covers the cross-backend capability; per house rule, no sibling entry for a refinement of a feature debuting in the same unreleased version). Files: src/hiveplotlib/viz/base.py, viz/bokeh.py, viz/holoviews.py, viz/plotly.py, viz/matplotlib.py, tests/viz_bokeh_test.py, tests/viz_holoviews_test.py, tests/viz_plotly_test.py.
