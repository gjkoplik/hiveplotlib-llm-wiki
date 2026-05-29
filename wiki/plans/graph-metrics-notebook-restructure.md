# Plan: Graph-metrics notebook restructure (gallery cleanup + Les Misérables partition-discovery tutorial)

<!--
Hiveplotlib consumer work. Plan tracked in the wiki submodule. CHANGELOG routing
is CHANGELOG.rst (the v0.28.0 WIP entry), not a harness CHANGELOG.
-->

## Goal

After the NetworkX metric-expansion sprint, `examples/computing_graph_metrics.ipynb` mixes `HivePlot` and `HivePlotMatrix` content across three datasets (Karate Club, Les Misérables, the synthetic toy). This plan splits that content cleanly: `computing_graph_metrics.ipynb` becomes a focused `HivePlot`-only **gallery** (Karate for the network demos, the synthetic toy for pure mechanics), and a new **tutorial** on the Les Misérables co-appearance network teaches a "partition discovery" arc (a graph with no natural partition; use graph metrics and community detection to find one; when community detection yields >3 groups, route to `HivePlotMatrix`). When this ships, a user reaching for "how do I compute a metric on a hive plot" lands on a coherent gallery, and a user asking "my graph has no obvious groups, how do I even make a hive plot" lands on a narrative tutorial that ends by handing off to the existing `hive_plots_more_than_three_groups.ipynb` menu of >3-group options. No user-facing API changes; this is a docs-only restructure.

## Prior ADRs / design docs

Populated from research-liaison pre-task findings. No ADRs exist yet (`wiki/wiki/adr/` is absent); the binding decision record is the prior plan.

- **`wiki/wiki/plans/networkx-metric-expansion-and-backend-refactor.md` — the binding prior plan.** Its Workstream E (lines ~958-981) authored the current `computing_graph_metrics.ipynb`. **Workstream E amendment 1** (lines ~1058-1101) is the most load-bearing prior decision: a viz-critic `must-fix` swapped the Karate-Louvain section (4 communities) and the Les-Mis-Louvain section (6 communities) from `HivePlot` to `HivePlotMatrix.from_partition`, because >3 communities break the hive-plot 2-3-axis invariant. The new tutorial's ">3 communities -> HPM" spine is the **same already-accepted argument**, just relocated and given a narrative. Removing "all HPM content" from the gallery means removing exactly those amendment-1 cell pairs (current cells 16-19 and 20-25). The bugfix amendment at line ~1161 fixed the `HivePlotMatrix.from_partition` int-partition `KeyError` so those HPM cells render today.
- **User Resolution item 6** (line ~274): the Karate-vs-Les-Mis dataset split (Karate = familiar / pre-labeled, recovers the historical `club` split; Les Mis = no ground truth, Louvain reveals structure) was an explicit user decision. The tutorial reuses the Les Mis rationale, so it is pre-sanctioned.
- **Rejected alternative** (line ~1074, reaffirmed at ~1068): forcing 2-3 communities via `louvain_communities(resolution=...)` was rejected as dishonest pedagogy. The tutorial keeps the honest "Louvain finds 6 communities" framing and routes to HPM rather than tuning the count down.
- **Known cosmetic deferral** (lines ~1116-1118, 1148-1154): `HivePlotMatrix.from_partition` coerces row/col labels to strings (`hiveplot_matrix.py:1220`); integer community labels render as integer-looking strings. Deferred to v0.28.0 close-out. The tutorial **names communities by their most-central character** rather than using integer labels, which sidesteps this entirely and also avoids the gallery anti-pattern of numbered community labels.
- **Canonical chaining target confirmed:** `examples/hive_plots_more_than_three_groups.ipynb` is the corpus authority that 3 axes is the clean case and >3 groups route to the four options (more axes / two layers / collapse-to-3 / `HivePlotMatrix.from_partition` small multiples). It uses the trade dataset (continents, a *natural* partition) and the datashader backend.
- Supporting context (framing, not constraints): `wiki/wiki/concepts/hive-plot-matrix.md`, `wiki/wiki/analyses/karate-club-walkthrough.md`, `wiki/wiki/analyses/gnn-heterogeneity-hive-plots.md`. Gallery-vs-tutorial conventions live in the harness skills (`hiveplotlib-gallery-notebook`, `hiveplotlib-tutorial-notebook`), not the wiki.

## Patterns this replaces

This is a notebook restructure, so "patterns" here are notebook sections and cross-links, not source symbols. Grepped against the current branch.

**Content removed from `examples/computing_graph_metrics.ipynb`** (the two HPM blocks plus their riders, per prior plan amendment 1):

- **"Using Community Detection as a Partition Variable"** section (current cells 16-19: `community-as-partition-md`, `community-as-partition-code` [`HivePlotMatrix.from_partition` on Karate Louvain, 4 communities], `community-as-partition-xtab-md`, `community-as-partition-xtab-code`). HPM content; leaves the gallery. The community-detection-as-partition technique relocates to the tutorial on Les Mis.
- **"Precompute Once, Reuse Across Plots"** section (current cells 20-25: `precompute-reuse-section-md`, `precompute-reuse-compute-code` [Les Mis load], `precompute-reuse-plot1-md`, `precompute-reuse-plot1-code` [`HivePlotMatrix.from_partition` Les Mis, 6 communities], `precompute-reuse-plot2-md`, `precompute-reuse-plot2-code` [Les Mis harmonic-centrality 3-axis `HivePlot` bin]). The entire section leaves: the "precompute once, reuse" *framing* is dropped (not taught as a lesson; reuse simply runs faster), Les Mis exits the gallery completely, and the 3-axis HC-bin `HivePlot` (a true hive plot) rides along because it is structurally bundled into the Les Mis section. Its mechanic (bin a metric into a <=3-group partition) is the **motivated version** that relocates to the tutorial.

**Decision resolved (amended 2026-05-29) — "Using a Computed Metric as a Partition Variable" section stays on Karate Club** (current cells 12-15: `metric-as-partition-section`, `metric-as-partition-code`, `metric-as-partition-inspect-md`, `metric-as-partition-inspect-code`). This bins Karate `degree` into low/medium/high and builds a fresh `nodes`/`edges` partitioned on the bin. Two options were originally weighed and locked to option (a) (move the demo onto the synthetic toy): **(a)** move the demo onto the synthetic toy (`example_hive_plot()`, no story to wreck; keeps the lower-level `compute_graph_metrics` + `create_partition_variable` path documented in the gallery), or **(b)** retire the section with a pointer to `examples/create_partition_variable.ipynb` and the new tutorial. **The user has reversed that lock: the section keeps binning Karate `degree` into low/medium/high (neither prior option a nor b).**

Why this is coherent and not the "destroys the known story" failure: the "don't wreck Karate's known story" principle targets the *narrative-level* wreck only — community detection (Louvain) shattering the known two-faction graph into many anonymous groups, or making a degree-bin the gallery's *primary* framing. A single, transparently-labeled section demonstrating the discretize-a-metric *mechanic*, while `club` remains Karate's primary partition in every other section, is different in kind: it is the "questionably appropriate but very representative" gallery demo the user explicitly endorses, and it conveys the mechanic well on a familiar graph. The section must be framed as a mechanical demonstration of discretizing a continuous metric into a partition, **not** as a claim that degree-bins are a meaningful grouping of the club network. Karate-partitioned-two-ways (primary `club` partition plus this one degree-bin mechanic section) is a user-endorsed deliberate choice, not an incoherence to flag.

Anchor: the section heading is unchanged regardless of dataset, so the anchor link at `setting_partition_variable.ipynb:699` (`...#using-a-computed-metric-as-a-partition-variable`) keeps resolving. Keeping the demo on Karate `degree` also keeps that cross-link's existing prose ("an example using node `degree` as a partition variable") literally accurate, which option (a)'s move to a toy continuous metric would have made stale.

**The `reset_edges` wart** (`reset_edges(axis_id_1="Mr. Hi_repeat", axis_id_2="Officer")` with the bare comment `# only 2 unique partition values / #  so we drop one redundant set of inter-axis edges`) appears pasted in **three** code cells: current cell 7 (`3d936513`, node-metrics init), cell 27 (`2349a8c6`, edge-metrics init), cell 31 (`link-prediction-code`). Replace with: explain the mechanic **once** in a short markdown cell at its first appearance (cell 7's section), and have the later two cells carry a one-line back-reference comment instead of re-explaining. The call itself stays in all three (it is doing real work); only the unexplained-repetition pattern is replaced.

**Cross-link audit (every reference to `computing_graph_metrics.ipynb` in shipped files; auto-generated `docs/source/notebooks/` copies excluded — they regenerate on `make docs`):**

- `docs/source/conf.py:61` — thumbnail mapping `"notebooks/computing_graph_metrics": "_static/computing_graph_metrics.jpg"`. Stays. The **new tutorial needs its own entry added** here (Workstream D).
- `docs/source/gallery_examples/index.rst:19, :37` — `computing_graph_metrics` registered in "The HivePlot Class" gallery section (both `nblinkgallery` and hidden `toctree`). Stays (still a HivePlot gallery). The **new tutorial registers in `docs/source/notebooks/index.rst`** (tutorial index), not here (Workstream D).
- `CHANGELOG.rst:30` — v0.28.0 NetworkX-compat bullet links it as a walkthrough. Stays valid (narrowed gallery still covers `graph=`/metrics on `HivePlot`).
- `CHANGELOG.rst:107-117` — "Three new gallery notebooks" Documentation block. **Needs update** when the tutorial is added (Workstream D / WS-cross-link): the count and framing ("gallery notebooks") no longer fit once a tutorial joins. Add the tutorial here.
- **13 example notebooks cross-link `computing_graph_metrics.ipynb`** as the canonical metrics reference. All remain valid after narrowing (the notebook still owns: the metric tables, requesting metrics, per-metric kwargs, discretizing, column collisions, internal graph type). The audit question per the brief is whether each should *also* point at the new tutorial. Decision (Workstream D): the generic "for more on supported graph metrics, see Computing Graph Metrics" pointers stay as-is (they want the reference, not the narrative). Only notebooks whose own subject is *community detection / partition discovery / >3 groups* gain an added tutorial pointer. Concretely:
  - `examples/hpm_from_partition.ipynb:cell 47` and `examples/hpm_from_variable_sweep.ipynb:cell 50` — both close their "Computing Graph Metrics During Construction" section with the identical "...or discretizing node graph metrics to use as partition variables, see Computing Graph Metrics" sentence. These two are the strongest candidates to **also** point at the new tutorial (the tutorial's payoff *is* an HPM of detected communities). Add a second sentence pointing at the tutorial.
  - `examples/setting_partition_variable.ipynb:699` — links to the `#using-a-computed-metric-as-a-partition-variable` **anchor** and reads "an example using node `degree` as a partition variable." Anchor survives (heading unchanged), and under the amended keep-on-Karate decision the `degree` wording stays literally accurate too. Leave as-is (it wants the gallery's discretization mechanic, not the narrative).
  - `examples/hive_plots_more_than_three_groups.ipynb` — the chaining *target*. Does not currently link `computing_graph_metrics.ipynb`. The new tutorial links *into* it; whether it should link *back* to the tutorial is a Workstream C/D call (a "this technique was motivated in the Les Mis tutorial" back-pointer is reasonable but optional; default to adding it only if it reads naturally).
  - The remaining 11 (`introduction_to_hive_plots`, `quick_hive_plots`, `karate_club`, `networkx_examples`, `creating_hive_plots_from_networkx`, `hive_plot_matrices`, `hive_plots_for_large_networks`, `exporting_hive_plots_to_networkx`, `visualizing_node_metadata`, `visualizing_edge_metadata`, `hpm_from_tags`, `hpm_generic`) — generic metric-reference pointers. Leave as-is; verify each still reads correctly given the narrowed scope (Workstream D done-when).

Survivors that are NOT replace targets (holdouts): see Holdouts.

## Default justifications

No new user-facing API defaults (docs-only). The notebook *dataset* choices are the decisions that need justifying:

- **Gallery keeps two datasets: Karate Club (partitioned by `club`) and the synthetic `example_hive_plot()` toy.** Both are justified and the justification is documented in the gallery prose (Workstream A): Karate Club is *undirected*, so it cannot motivate directed-only metrics (in/out-degree) or the directed-vs-undirected internal-graph-type section; the toy is *directed by default*, so it carries the pure-mechanics sections that need a directed graph or a neutral surface (column-name collisions, the `graph_directed` / `graph_multigraph` internal-graph-type demos). Karate carries the network demos where a real two-faction partition makes the figures legible — including the metric-as-partition section, which (amended 2026-05-29) stays on Karate `degree` as a transparently-labeled demonstration of the discretize-a-metric *mechanic* while `club` remains Karate's primary partition elsewhere. Two datasets, each pulling its weight; no third dataset (Les Mis leaves).
- **Tutorial uses Les Misérables co-appearance network.** Justified as the canonical "no natural partition" network: 77 character nodes, edges = co-appearance in the novel, no ground-truth grouping (contrast Karate's `club`, contrast the trade data's `continent`). This is the dataset that motivates "discovery" because the reader cannot fall back on a pre-labeled grouping. Pre-sanctioned by User Resolution item 6.
- **Tutorial communities named by most-central character, not integer labels.** Justification: numbered community labels are the gallery anti-pattern, but here *teaching the community-detection technique is the point*, so the labels are load-bearing and deserve human-readable names; naming by the most-central member (e.g. by `harmonic_centrality` or `degree` within each community) gives the reader a handle, and as a bonus sidesteps the deferred `from_partition` int->str coercion cosmetic issue. Not knowing the novel is fine; this is a graph-theory journey, not literary analysis (one sentence in the tutorial can say so).

## Naming audit

No new parameters, methods, or classes (docs-only). The user-facing names introduced are **the new tutorial's filename and title**, plus the community-label display names.

**Filename and title** checked against the existing `examples/` tutorial conventions (real-data subtype: noun phrase naming the dataset or concept; snake_case file; `### H3` title; siblings `karate_club.ipynb` / "Zachary's Karate Club", `bitcoin_user_ratings.ipynb`, `comparing_network_subgroups.ipynb` / "Comparing Network Subgroups", `hive_plots_more_than_three_groups.ipynb` / "Hive Plots with More Than 3 Groups", `hive_plots_for_large_networks.ipynb`):

Candidates weighed (the brief's two poles: "finding/choosing a partition with graph metrics" vs. "community detection"):

| Filename | Title | Read |
| --- | --- | --- |
| `finding_a_partition.ipynb` | `### Finding a Partition with Graph Metrics` | Concept-forward; "partition" is the corpus term of art (matches `partition_variable`, `setting_partition_variable.ipynb`, `from_partition`). Verb "finding" carries the discovery arc. |
| `community_detection.ipynb` | `### Community Detection` | Too narrow: community detection is the *back half* of the arc, not the whole story (the front half is "metrics give you values to sort by; you still need axes"). Also reads as a gallery feature-reference title, not a tutorial. |
| `les_miserables.ipynb` | `### Les Misérables Co-Appearances` | Dataset-forward, parallels `karate_club.ipynb`. But the dataset is the vehicle, not the lesson; a reader scanning the tutorial index for "how do I pick a partition" would not recognize this title as the answer. |
| `choosing_a_partition.ipynb` | `### Choosing a Partition` | Close to the winner; "choosing" slightly understates that the data gives you *neither* a partition nor a sorting and you have to *discover* both. |

**Confirmed (amended 2026-05-29): `finding_a_partition.ipynb`, title `### Finding a Partition with Graph Metrics`.** The user confirmed both the filename and title; the prior "flag for user confirmation" is cleared. Reasoning that held up: (1) "partition" is the established corpus vocabulary (every adjacent notebook and the API use it), so a reader looking to solve "what do I partition on?" recognizes the title immediately; (2) "with Graph Metrics" ties it to the gallery it splits from and signals the technique, satisfying the "graph metrics" pole; (3) "Finding" carries the discovery narrative the tutorial is built around, satisfying the framing better than "Choosing"; (4) it does not over-index on "community detection," which is one step of the arc, not the whole thing; (5) snake_case file + `### H3` concept-noun-phrase title matches the conceptual-deep-dive subtype (`comparing_network_subgroups.ipynb`, `hive_plots_more_than_three_groups.ipynb`). Downstream workstreams (C, D) use this slug as fixed, not pending.

**Community-label display names:** the wrapper returns integer labels `0..5` (sorted largest-first per the prior plan's label-order contract). The tutorial maps each to its most-central character's name for display (e.g. `"Valjean's circle"` / `"Valjean"`, etc. — exact names depend on what the render shows in Workstream B). These are prose-only display strings, not API names; checked only for not colliding with `networkx` node ids (they are derived *from* node ids, so any "Valjean" label is the actual node, no collision). Audit OK.

## API usage examples

No API surface change. The "usage examples" here are **notebook-content sketches** for the new tutorial's key cells (the gallery cleanup is subtractive plus prose, so it needs no new runnable sketch beyond the relocated/retained cells already in the notebook). These sketches are runnable Python a notebook-author can build from; they are not final notebook cells (prose, titles, and figure polish are the notebook-author's craft under the tutorial skill).

The tutorial requires the `networkx` and `datashader` extras (the 6-community Les Mis HPM renders poorly in matplotlib; `hive_plots_more_than_three_groups.ipynb` uses datashader for the same reason). Surface up front: `pip install hiveplotlib[networkx,datashader]`.

```python
# Sketch 1: the dataset and the problem (no natural partition, no sorting).
# Lead cell after imports + background. Establishes "the data gives you neither."
import networkx as nx
from hiveplotlib.converters import networkx_to_nodes_edges

G = nx.les_miserables_graph()  # 77 character nodes, co-appearance edges
nodes, edges = networkx_to_nodes_edges(G)
# nodes.data has only the unique_id (character name); no group column, no metric.
nodes.data.head()
```

```python
# Sketch 2: graph metrics give node-level values to sort by, but you still need axes.
# Compute a sorting candidate; show it is a per-node scalar, not a grouping.
from hiveplotlib.graph_features import compute_graph_metrics

nodes, _ = compute_graph_metrics(
    G,
    node_metrics=["harmonic_centrality", "degree"],
    target_nodes=nodes,
    target_edges=edges,
)
nodes.data[["harmonic_centrality", "degree"]].describe()
# harmonic_centrality sorts nodes within an axis, but does not tell us how many
# axes to make or which node goes on which axis. We still need a partition.
```

```python
# Sketch 3a: first discovery path -- discretize a metric into a <=3-group partition.
# This is the relocated, *motivated* "bin a metric into a partition" demo.
hc_partition = nodes.create_partition_variable(
    data_column="harmonic_centrality",
    cutoffs=3,
    labels=["periphery", "supporting", "central"],
)
hp = HivePlot(
    nodes=nodes,
    edges=edges,
    partition_variable=hc_partition,
    sorting_variables="harmonic_centrality",
    axes_order=["periphery", "supporting", "central"],
    repeat_axes=True,
)
# 3-axis hive plot: a defensible, honest partition with an ordinal meaning.
# But the bins are an imposed cut, not a structure the data itself reveals.
```

```python
# Sketch 3b: better discovery path -- detect communities to find data-driven groups.
nodes, _ = compute_graph_metrics(
    G,
    node_metrics=["louvain_communities"],
    node_metric_kwargs={"louvain_communities": {"seed": 0}},
    target_nodes=nodes,
    target_edges=edges,
)
n_communities = nodes.data["louvain_communities"].nunique()
n_communities  # Louvain finds more than 3 communities on Les Mis (expected ~6)
# Honest framing: the data has more than 3 natural groups. We do NOT tune the
# resolution down to force 3; we change the visualization primitive instead.
```

```python
# Sketch 4: >3 communities motivate a HivePlotMatrix.
# Name communities by most-central member for legible labels (and to sidestep the
# deferred int->str label coercion). Exact label derivation is the author's call;
# one workable shape:
import pandas as pd

df = nodes.data
most_central = (
    df.loc[df.groupby("louvain_communities")["harmonic_centrality"].idxmax()]
    .set_index("louvain_communities")["unique_id"]
    .to_dict()
)
df["community"] = df["louvain_communities"].map(
    lambda c: f"{most_central[c]}'s group"
)
# rebuild a NodeCollection/Edges carrying the named 'community' column, then:
hpm = HivePlotMatrix.from_partition(
    nodes=nodes,            # carrying the named 'community' column
    edges=edges,
    partition_variable="community",
    sorting_variables="harmonic_centrality",
    backend="datashader",
    unify_axes=True,
    progress=False,
)
fig, axes, im_nodes, im_edges = hpm.plot()
# 6x6 upper-triangular matrix of pairwise community comparisons.
```

```python
# Sketch 5: hand off to the >3-groups menu. This is prose + a cross-link, not code.
# "A hive plot matrix is one of several ways to handle more than three groups.
#  For the full menu (more axes, two layers, collapsing groups, and the matrix
#  shown here), see the Hive Plots with More Than 3 Groups tutorial." -> link to
#  hive_plots_more_than_three_groups.ipynb. Do NOT re-tread its four options here.
```

### API Critic's take (planning mode)

**N/A — no user-facing API surface change.** This is a docs-only restructure: notebooks move/retire/are-authored, cross-links and docs registration update, no source symbols are added or modified. Per the consumer trip-wire and rule 7, api-critic planning review is not required here, and there is no post-impl api-critic invocation in the dispatch sequence. Confirmed against the brief's expectation. (Sanity note for the dispatching session: if, during Workstream B/C, the notebook-author discovers the named-community workflow needs a helper that does not exist on the shipped surface — i.e. the API usage sketch invents an unauthorized convention — that is a rule-9 surface-back to orchestrator `amend-plan` for a feasibility check, *then* api-critic. The sketches above use only shipped surface: `compute_graph_metrics`, `create_partition_variable`, `from_partition`, plain pandas for the label map.)

### API Critic — post-implementation review

No API surface change. Skipped (no planning take, no post-impl review).

## Workstreams

Four workstreams. Sequencing per the brief: WS-A (gallery cleanup) first — low-risk, fully specified. Then WS-B (tutorial dataset/story **validation gate**) before WS-C (full tutorial draft) — the gate must pass or the tutorial dataset gets reconsidered. WS-D (cross-link + docs-registration sweep) last, after both notebooks exist. WS-A and WS-B are independent and may run concurrently.

**Critic coverage.** Each notebook-restructuring workstream gets two parallel post-impl critics: **editorial-critic** (the notebook as a whole artifact, against the five editorial-bar checks: right notebook / class-scope, dataset coherence, genre fit, section-worth, cross-links) and **viz-critic** (the figures). They are complementary and non-overlapping: editorial-critic owns *structure*, viz-critic owns *figures*. Per the current editorial-critic definition, editorial-critic **reads prior viz-critic reports as input** (so it can tie a figure symptom to its structural cause) and **cites viz-critic findings rather than duplicating them** — it does not re-review figures. Both critics are read-only and propose-only. Editorial-critic's report format is `Status: clean | propose`, with each finding tagged `[must-fix | worth-discussing | low-confidence]` at a specific `<file>:<cell>`; scope-crossing findings (wrong notebook, dataset set, what the notebook teaches) are `must-fix` and route through orchestrator `amend-plan` (rule 14) for user sign-off, not direct edits. Viz-critic's `must-fix` / `should-fix` route the same way. Editorial-critic is especially load-bearing for *this* plan: its "right notebook / class-scope" check (HPM content inlined in a single-class page), "dataset coherence" check (the three-dataset problem), "genre fit" check (gallery-vs-tutorial), and "section-worth" check (the NARROW-scope discipline on WS-C) are the exact failure modes this restructure targets, so it functions as the verification that the split actually landed. The whole plan closes with a qa-engineer release-readiness pass.

### Workstream A: Gallery cleanup — `computing_graph_metrics.ipynb` becomes HivePlot-only

**Status:** not started
**Specialist:** notebook-author (ground in `hiveplotlib-gallery-notebook` skill + `viz-quality-bar`). Post-impl: **editorial-critic** (does the narrowed notebook now read as a single-feature HivePlot **gallery** — class-scope: HivePlot-only, no HPM content; genre fit: still a gallery, not drifted into tutorial voice now that its narrative HPM arc has left; dataset coherence: Karate + toy, justified; section-worth: every section earns its place) **and** viz-critic (retained/edited figures), in parallel.
**Files:** `examples/computing_graph_metrics.ipynb` only. (Do not touch `docs/source/notebooks/` or `docs/source/gallery_examples/` — auto-generated. Cross-link edits to *other* notebooks are Workstream D.)

**Decision (amended 2026-05-29) — the "Using a Computed Metric as a Partition Variable" section STAYS on Karate Club.** This reverses the prior lock to option (a) (move the demo onto the synthetic toy). The section keeps binning Karate `degree` into low/medium/high. Rationale: (1) it conveys the discretize-a-metric mechanic well on a familiar graph and is the "questionably appropriate but very representative" gallery demo the user explicitly endorses; (2) the "don't wreck Karate's known story" principle targets the *narrative-level* wreck only (Louvain shattering the two-faction graph, or a degree-bin becoming the gallery's *primary* framing) — a single transparently-labeled mechanic section, with `club` remaining Karate's primary partition in every other section, is different in kind and not that failure; (3) the section heading is unchanged, so the anchor link from `setting_partition_variable.ipynb:699` (`#using-a-computed-metric-as-a-partition-variable`) keeps resolving, and keeping the demo on `degree` keeps that cross-link's existing "node `degree` as a partition variable" prose literally accurate. **Prose constraint:** frame the section as a mechanical demonstration of discretizing a continuous metric into a partition, NOT as a claim that degree-bins are a meaningful grouping of the club network. Note the Karate mechanic demo and the *motivated* version in the tutorial (Sketch 3a, harmonic centrality on Les Mis) are deliberately different framings: the gallery shows the *mechanic* on a familiar graph, the tutorial shows *why you'd reach for it* on real data where no partition exists. Repetition across gallery/tutorial is acceptable per the brief. **Karate-partitioned-two-ways (primary `club` plus this one degree-bin mechanic section) is a user-endorsed deliberate choice, not a must-fix incoherence** — editorial-critic should not flag it or attempt to revert it.

**Done when:**

1. The two HPM blocks are removed: "Using Community Detection as a Partition Variable" (cells 16-19) and "Precompute Once, Reuse Across Plots" (cells 20-25, including its 3-axis HC-bin rider). No `HivePlotMatrix` usage remains anywhere in the notebook. Les Misérables (`nx.les_miserables_graph()`) no longer appears.
2. The imports cell is pruned to match the narrowed content: `HivePlotMatrix` drops from `from hiveplotlib import ...` if no longer used; `pandas` import (added inline at old cell 19 for the cross-tab) is removed if the cross-tab is gone; `ScalarMappable` / `Normalize` stay iff the Jaccard edge-coloring cell (cell 31) is retained (it is — it is a `HivePlot`, kept). Verify against the final cell set.
3. (Amended 2026-05-29) The "Using a Computed Metric as a Partition Variable" section **stays on Karate Club**: it keeps binning Karate `degree` into low/medium/high and partitioning on the bin. The section heading is preserved (anchor stability for `setting_partition_variable.ipynb:699`). The prose is framed as a mechanical demonstration of discretizing a continuous metric into a partition, NOT as a claim that degree-bins are a meaningful grouping of the club network. `club` remains Karate's primary partition in every other section. (Karate-partitioned-two-ways here is a user-endorsed deliberate choice, not an incoherence — see the decision block above and done-when #11.)
4. The `reset_edges` wart is explained once: a short markdown cell precedes its first appearance (cell 7's node-metrics-init section) explaining *why* the redundant inter-axis edge set is dropped for a 2-partition repeat-axes plot; the two later occurrences (cells 27, 31) carry a one-line back-reference comment (e.g. `# drop the redundant inter-axis edge set (see "..." above)`) rather than the full pasted comment. The `reset_edges` calls themselves remain in all three cells.
5. A two-dataset justification is stated in prose where the toy first appears (or in the lead-in): Karate is undirected (cannot motivate in/out-degree), the toy is directed-by-default and storyless (carries collisions + internal-graph-type mechanics). One or two sentences, gallery voice.
6. The notebook stays NetworkX-only and HivePlot-only. No igraph, no broadening (the prior plan's open question 5 anticipates a separate igraph parity notebook later; out of scope here).
7. Closing pointer(s) updated to gallery convention (prose paragraphs, not `## See Also` bullets). The existing closing pointers to `creating_hive_plots_from_networkx.ipynb` and `exporting_hive_plots_to_networkx.ipynb` stay; **add a pointer to the new tutorial** (`finding_a_partition.ipynb`) framed as "for a narrative walkthrough of using metrics and community detection to find a partition when the data gives you none, see ...". (Coordinate the exact filename with Workstream C; if WS-C runs after WS-A, this pointer is added in Workstream D instead — note it either way so it is not dropped.)
8. `make test-nb` runs `computing_graph_metrics.ipynb` end-to-end and passes; all retained cells execute cleanly (warnings are errors).
9. Voice-rule scan clean on edited markdown (no em-dashes, no AI filler, length matches sibling sections).
10. Edited only at `examples/computing_graph_metrics.ipynb`.
11. editorial-critic post-impl returns `Status: clean`, or `propose` with no `must-fix` finding: confirms the notebook now reads as a single-feature HivePlot **gallery** against the editorial bar — **class-scope** (no `HivePlotMatrix` content survives, primary subject stays on `HivePlot`), **genre fit** (still gallery voice/structure, has not drifted toward tutorial narration now that its HPM arc has left), **dataset coherence** (the Karate + toy set is coherent and justified in prose), and **section-worth** (every section earns its place, nothing a sibling notebook covers better). **Note (amended 2026-05-29): Karate-partitioned-two-ways — `club` as the primary partition plus the one degree-bin mechanic section — is a user-endorsed deliberate choice; editorial-critic must not flag it as a coherence `must-fix` or propose reverting it to the synthetic toy. The check is only that the degree-bin section is framed as a mechanic demonstration (not as a meaningful grouping claim) and that `club` stays the primary framing everywhere else.** viz-critic post-impl returns `good`, or `worth-discussing`-only, on retained figures. Any `must-fix` (editorial or viz) routes through orchestrator `amend-plan` (rule 14) before WS-A closes.

### Workstream B: Tutorial dataset/story validation gate (VALIDATE-THEN-COMMIT)

**Status:** not started
**Specialist:** notebook-author (or whoever renders; ground in `viz-quality-bar` + `hiveplotlib-tutorial-notebook` "Partition design" section). viz-critic reviews the validation render (figure legibility) before WS-C is green-lit. **editorial-critic does NOT run at this gate.** Per its current definition it is post-impl / pre-merge only (its primary input is a finished notebook, which does not exist yet); planning-time arc-vs-data coherence is the orchestrator's job and has already been done in this plan. The gate's "does the discovery story cohere on this data" question previews editorial-critic's dataset-coherence check, but the structured editorial-critic pass belongs on the WS-C draft.
**Files:** none committed to the repo. Scratch renders go to `/tmp/` (mental-model rule 16). This workstream produces a **go/no-go decision and a saved scratch render**, not notebook cells.

**Why gated:** the earlier scratch render (`/tmp/hpm_dataset_explore/lesmis_by_community.png`) has been cleared (machine-local `/tmp`), so the validation must build fresh. The tutorial's whole arc depends on the Les Mis community structure being *legible* and telling a *coherent* story (the 6-community HPM reads, the named-community labels make sense, the discovery narrative lands). If it does not, an alternative partition-less network is chosen here before any drafting cost is sunk. Trade data is explicitly **out** (it is already `hive_plots_more_than_three_groups.ipynb`'s dataset and has a natural `continent` partition, so it cannot motivate discovery).

**Done when:**

1. A fresh render of the Les Mis Louvain-community HPM (datashader backend, `from_partition`, communities named by most-central member) is produced and saved under `/tmp/` (e.g. `/tmp/finding_a_partition_validation/lesmis_communities_hpm.png`). The 3-axis harmonic-centrality-bin `HivePlot` (Sketch 3a) is also rendered for the "discretize" leg.
2. A go/no-go judgment is recorded in the dispatch report (not the repo): does the 6-community structure read legibly as an HPM at notebook render size, do the named-community labels produce a coherent story (each community has a recognizable central character), and does the "metrics give you values -> still need axes -> discretize or detect communities -> >3 groups -> HPM" arc hold on this data? viz-critic concurs or dissents.
3. If **no-go**: the report names the failure (e.g. communities are not legible, labels are confusing, story does not cohere) and proposes an alternative partition-less network (candidates: Florentine families [small, may be too small], a stochastic-block-model synthetic with a *hidden* partition the reader must discover, another `networkx` built-in co-occurrence graph). A no-go routes back to the dispatching session for a re-scope decision before WS-C; it does **not** silently substitute a dataset.
4. The validation confirms the `networkx` + `datashader` extras are what the tutorial needs (matplotlib at 6 communities was the prior viz-critic concern; confirm datashader is the right call here, matching `hive_plots_more_than_three_groups.ipynb`).
5. The community-label derivation approach is validated as runnable on shipped surface (Sketch 4's pattern, or whatever the author lands on): confirm `from_partition` accepts the named string-column partition and the 6x6 matrix builds without error (the int-partition `KeyError` is fixed per the prior bugfix amendment; a *string* partition column is the path the tutorial uses, which also sidesteps the int->str label coercion). **Mechanism note (amended 2026-05-29):** naming communities by most-central character stands. The **default** is Sketch 4's named-column approach — derive a named string partition column, which yields character-named HPM row/column labels for free and sidesteps the int->str coercion. An **alternative** the user raised is relabeling within the `HivePlotMatrix` itself; treat that strictly as a fallback to explore *only if* a clean label-override path exists on the matrix. The known coercion (labels appear to derive from partition values) suggests the named-column path is likely the only robust route; this done-when validates whichever mechanism is chosen.

### Workstream C: Tutorial draft — `finding_a_partition.ipynb`

**Status:** not started (blocked on WS-B go)
**Specialist:** notebook-author (ground in `hiveplotlib-tutorial-notebook` skill + `viz-quality-bar`). Post-impl: **editorial-critic** (genre fit as a tutorial; the arc coheres and the lead-in matches the body; **section-worth under the NARROW-scope discipline** — no survey-style section the cleaned gallery or `hpm_from_variable_sweep.ipynb` already covers better; the >3-groups hand-off aims at the best next step without re-treading it; the two relocated demos sit naturally) **and** viz-critic (the tutorial's figures, datashader HPM especially), in parallel.
**Files:** `examples/finding_a_partition.ipynb` (new). Filename **confirmed** by the user (amended 2026-05-29); use this slug consistently across WS-C and WS-D.

**Scope is NARROW (amended 2026-05-29) — do not drift into a general graph-metrics survey.** This tutorial is the partition-discovery arc only (the spine below). A broader "what you can do with graph metrics as a whole" survey is explicitly OUT of scope and left as a possible *separate future notebook*, not scheduled. Reasons: it would balloon this tutorial, and those other mechanics are already distributed across the cleaned gallery (`computing_graph_metrics.ipynb`: sorting, edge styling, per-metric kwargs, column collisions, internal graph type) and the HPM galleries (the metric sweep in `hpm_from_variable_sweep.ipynb`). Keep the draft disciplined to the spine; resist adding survey-style sections.

**Subtype:** conceptual deep-dive on a real dataset (so: `### H3` title, `#### Background` on Les Mis provenance, `#### References` with the Knuth / Les Mis co-appearance citation, rhetorical question posed early and revisited). Closest siblings to calibrate length and voice: `comparing_network_subgroups.ipynb`, `hive_plots_more_than_three_groups.ipynb`, `karate_club.ipynb`.

**Narrative arc (the spine, do not pad):**

1. **Background + the problem.** Les Mis co-appearance network: 77 characters, edges = co-appearance. Pose the question: a hive plot needs a *partition* (which axis does a node go on) and a *sorting* (where on the axis). This graph gives you *neither*. How do you choose? (One sentence: this is a graph-theory journey, not literary analysis; you do not need to know the novel.)
2. **Metrics give you a sorting, not a partition.** Compute `harmonic_centrality` / `degree`. Show they are per-node scalars: good for sorting *within* an axis, but they do not tell you how many axes or which node on which. You still need a partition. (Sketch 1, 2.)
3. **First path: discretize a metric into a <=3-group partition.** Bin `harmonic_centrality` into periphery/supporting/central, build a 3-axis hive plot. Honest, defensible, ordinal — but an *imposed* cut, not structure the data reveals. (Sketch 3a.) This is the relocated motivated "bin a metric" demo.
4. **Better path: detect communities to find data-driven groups.** Run Louvain. It finds **more than 3** communities (honest framing; do *not* tune resolution to force 3 — cross-reference the rejected-alternative decision). (Sketch 3b.)
5. **>3 communities motivate a `HivePlotMatrix`.** Build the 6x6 HPM with communities named by most-central member, datashader backend. This is the payoff figure. (Sketch 4.)
6. **Hand off.** A hive plot matrix is one of several ways to handle >3 groups; for the full menu, see `hive_plots_more_than_three_groups.ipynb`. Cross-link, do **not** re-tread its four options. Revisit the opening question (we found a partition two ways; community detection found the data-driven one; >3 groups routed us to the matrix).
7. **References.**

**Done when:**

1. `examples/finding_a_partition.ipynb` exists, follows the tutorial skill (H3 title, H4 subsections, `#### Background`, `#### References`, install-extras up front for `networkx,datashader`, prose-to-code balanced).
2. The arc above is realized; the honest "Louvain finds >3 communities -> switch primitive, do not tune resolution" framing is preserved (matches the prior plan's rejected-alternative decision).
3. Communities are displayed by most-central-character names, not integer labels (sidesteps the deferred coercion; avoids the numbered-label anti-pattern).
4. The >3-groups hand-off cross-links `hive_plots_more_than_three_groups.ipynb` and does not duplicate its content. The discretize/detect contrast (ordinal bin vs. categorical community) mirrors the pedagogy the prior viz-critic re-review endorsed (lines ~1139-1146) without copying the gallery's removed cells.
5. The two relocated "orphan" demos are absorbed: community-detection-as-partition (here, on Les Mis, **not** Karate) and the motivated "bin a metric into a partition" demo (here, the harmonic-centrality 3-axis leg).
6. `make test-nb` runs the new notebook end-to-end and passes (warnings are errors). datashader cells render.
7. Voice-rule scan clean (no em-dashes, no AI filler; tutorial length within ~2x its closest sibling per the skill's length discipline).
8. Figure-quality: datashader HPM follows `viz-quality-bar` datashader specifics (accept `cmap_nodes`/`cmap_edges` defaults, `unify_axes=True` for cross-cell comparison, pin rasterization params if cross-plot comparison is implied); the 3-axis hive plot follows hive-plot rules (repeat-axes two-tone if repeat axes used, title `y` tuned, alpha for density).
9. Created only at `examples/finding_a_partition.ipynb`. Docs registration is Workstream D (do not edit `docs/` here).
10. editorial-critic post-impl returns `Status: clean`, or `propose` with no `must-fix` finding: confirms it reads as a motivated **tutorial** against the editorial bar — **genre fit** (motivated tutorial, not a gallery page), arc coheres end to end with the lead-in matching the body, **section-worth** under the NARROW-scope discipline (no survey-style section that the cleaned gallery or `hpm_from_variable_sweep.ipynb` covers better; the draft stays on the partition-discovery spine), the relocated demos sit naturally, and the **cross-link** to `hive_plots_more_than_three_groups.ipynb` aims at the best next step without duplicating its four options. viz-critic post-impl returns `good`, or `worth-discussing`-only, on the figures. Any `must-fix` (editorial or viz) routes through orchestrator `amend-plan` (rule 14) before WS-C closes.

### Workstream D: Cross-link + docs-registration sweep + CHANGELOG

**Status:** not started (blocked on WS-A and WS-C complete)
**Specialist:** docs-engineer (Sphinx toctree / `conf.py` / CHANGELOG) with notebook-author for any notebook *prose* cross-link edits (the two can be one dispatch if the docs-engineer is comfortable editing notebook markdown cells; otherwise notebook-author does the `.ipynb` markdown edits and docs-engineer does `.rst` / `.py` / CHANGELOG). qa-engineer does the final grep audit.
**Files:** `docs/source/notebooks/index.rst`, `docs/source/conf.py`, `CHANGELOG.rst`, plus targeted markdown-cell edits in `examples/hpm_from_partition.ipynb`, `examples/hpm_from_variable_sweep.ipynb` (and optionally `examples/hive_plots_more_than_three_groups.ipynb` for the back-pointer), and the new-tutorial pointer in `examples/computing_graph_metrics.ipynb` if not already added in WS-A. Thumbnail asset under `docs/source/_static/`.

**Done when:**

1. **Tutorial registered in the tutorial index.** The new tutorial is added to `docs/source/notebooks/index.rst` in the **Hive Plots** section (it is a conceptual deep-dive, siblings: `comparing_network_subgroups`, `hive_plots_for_large_networks`, `hive_plots_more_than_three_groups`, `hive_plot_matrices`) — in **both** the `nblinkgallery` block and the hidden `toctree`, same position in each. Not the gallery index (it is a tutorial, not a feature reference). Natural placement: adjacent to `hive_plots_more_than_three_groups` (its hand-off target).
2. **Thumbnail.** A thumbnail entry is added to `nbsphinx_thumbnails` in `docs/source/conf.py` (`"notebooks/finding_a_partition": "_static/finding_a_partition.jpg"` or matching the chosen slug), and the asset is produced under `docs/source/_static/` (per `viz-quality-bar` "Thumbnails" — pick the most representative figure, likely the named-community HPM; strip text; save). If producing the asset is deferred, the entry must still resolve or `make docs` warns.
3. **CHANGELOG.** The `CHANGELOG.rst` Documentation block (lines ~107-117, currently "Three new gallery notebooks") is updated to include the new tutorial and to no longer mis-describe the set as only gallery notebooks (e.g. add a "New tutorial: Finding a Partition with Graph Metrics ..." bullet). Routed to `CHANGELOG.rst` v0.28.0 WIP entry (this is hiveplotlib consumer work). No em-dashes, no AI filler, length discipline.
4. **Targeted tutorial back-pointers added** to the two strongest candidates: `examples/hpm_from_partition.ipynb` (cell 47) and `examples/hpm_from_variable_sweep.ipynb` (cell 50) gain a second sentence pointing at the new tutorial (their existing "Computing Graph Metrics" pointer stays). Optionally `hive_plots_more_than_three_groups.ipynb` gains a "this technique was motivated in the Finding a Partition tutorial" back-pointer if it reads naturally.
5. **New-tutorial pointer present in `computing_graph_metrics.ipynb`** (added here if WS-A did not already add it).
6. **Cross-link integrity verified.** A grep for `computing_graph_metrics` across shipped files confirms: all 13 existing notebook cross-links still read correctly given the narrowed (HivePlot-only) scope; the `setting_partition_variable.ipynb:699` anchor still resolves (heading unchanged by the keep-on-Karate decision) and its "node `degree` as a partition variable" prose stays accurate; `conf.py:61`, `gallery_examples/index.rst:19/37`, `CHANGELOG.rst:30` unchanged and valid. A grep for the new tutorial slug (`finding_a_partition`, confirmed) confirms it appears in `notebooks/index.rst` (both blocks), `conf.py` (thumbnail), `CHANGELOG.rst`, and the back-pointer notebooks.
7. `make docs` builds with the new tutorial in the Hive Plots tutorial gallery, no new warnings (use `make docs`, not `make docs-strict`, and scan the full warning set per the user's recorded preference). `make test-nb` still green for all touched notebooks.
8. No edits to `docs/source/notebooks/` notebook copies or `docs/source/gallery_examples/` (auto-generated).

## Plan amendments

Append-only. Each entry summarizes a delta and tags it Added workstream / In-scope tweak / Deferred follow-up.

### 2026-05-29 — three user-decided amendments (no new workstreams, no API surface)

All three are **In-scope tweaks**: they refine decisions inside existing workstreams (WS-A, WS-B, WS-C) and the upstream plan sections that feed them; no specialist set changes, no done-when is removed, sequencing is unchanged. Triggered by user decisions (rule 14: user asks that modify done-whens / resolve a deferred decision), not by critic findings.

1. **Metric-as-partition demo STAYS on Karate Club — reverses WS-A's locked option (a).** The "Using a Computed Metric as a Partition Variable" section keeps binning Karate `degree` into low/medium/high; it does NOT move to the synthetic toy and is NOT retired (neither prior option a nor b). Rationale recorded for a future editorial-critic: the "don't wreck Karate's known story" principle targets the *narrative-level* wreck only (Louvain shattering the two-faction graph, or a degree-bin becoming the gallery's *primary* framing). A single transparently-labeled section demonstrating the discretize-a-metric *mechanic*, with `club` remaining Karate's primary partition everywhere else, is the "questionably appropriate but very representative" gallery demo the user endorses — different in kind from the wreck. **Prose constraint:** frame the section as a mechanical demonstration of discretizing a continuous metric into a partition, not a claim that degree-bins are a meaningful grouping of the club network. **Karate-partitioned-two-ways is a user-endorsed deliberate choice, not a must-fix incoherence; editorial-critic must not flag or revert it.** The anchor at `setting_partition_variable.ipynb:699` is preserved (heading unchanged) and its "node `degree` as a partition variable" prose stays literally accurate (option (a)'s toy move would have made it stale). *Sections edited:* "Patterns this replaces" (decision block now resolves to keep-on-Karate), "Default justifications" (Karate carries this section; toy still carries column-collision + internal-graph-type mechanics), WS-A decision block, WS-A done-when #3 (rebuilt) and #11 (editorial-critic coverage note added).

2. **Tutorial filename/title confirmed; tutorial scope pinned NARROW.** (a) The user confirmed `finding_a_partition.ipynb` / "Finding a Partition with Graph Metrics"; the naming audit's "flag for user confirmation" is cleared and the slug is fixed for WS-C and WS-D. (b) The tutorial scope is explicitly the partition-discovery arc only (the WS-C spine). A broader "what you can do with graph metrics as a whole" survey is OUT of scope, left as a possible *separate future notebook* (not scheduled) — those mechanics are already distributed across the cleaned gallery and the HPM galleries. *Sections edited:* "Naming audit" (recommendation marked confirmed, flag cleared), WS-C Files note (filename confirmed) and a new "Scope is NARROW / do not drift into a general graph-metrics survey" line.

3. **Community-labeling mechanism — default and fallback recorded (implementation note).** Naming communities by most-central character stands. **Default:** Sketch 4's named-column approach (derive a named string partition column → character-named HPM labels for free, sidesteps int->str coercion). **Fallback:** relabel within the `HivePlotMatrix` itself, to explore *only if* a clean label-override path exists on the matrix (the known coercion suggests labels derive from partition values, so the named-column path is likely the only robust route). WS-B done-when #5 already validates the chosen mechanism. *Sections edited:* WS-B done-when #5 (one-line mechanism note appended).

### 2026-05-29 (later) — re-interrogation against the re-synced harness (editorial-critic def + skills updated)

Full coherence + wiring re-interrogation after a harness editorial-content bump and `.claude/` re-sync. **All In-scope tweaks** (refinements to existing workstreams' editorial-critic wiring and done-whens; no new workstream, no specialist set change, no done-when removed, no API surface, sequencing unchanged). Triggered by the re-sync (rule 14: the harness change is the delta source), not by a critic finding or a user scope ask.

**Verified and confirmed (no change needed):**
- *Working-tree state matches the plan.* Notebook cell structure is exactly as the plan asserts: metric-as-partition section at cells 12-15 on Karate `degree`; the two HPM/Les-Mis blocks at cells 16-19 (Karate Louvain, 4 communities) and 20-25 (Les Mis load + 6-community HPM + 3-axis HC-bin rider); `reset_edges` riders at cells 7, 27, 31; directed/undirected `ValueError` traceback demos at cells 52 and 59. `finding_a_partition.ipynb` does not yet exist (consistent with WS-C "not started"). No rule-9 mismatch.
- *Cross-link claims hold.* `setting_partition_variable.ipynb:699` carries the exact "an example using node `degree` as a partition variable" prose at the `#using-a-computed-metric-as-a-partition-variable` anchor (keep-on-Karate keeps both accurate). The two WS-D back-pointer cells in `hpm_from_partition.ipynb` and `hpm_from_variable_sweep.ipynb` both carry the identical "...or discretizing node graph metrics to use as partition variables, see Computing Graph Metrics" sentence (cited as cells 47/50; actual ids `hpm-fp-metrics-md-3` / `8e648cb9`, prose matches — the numeric indices are approximate but the targets are real). CHANGELOG "Three new gallery notebooks" block at lines 107-117 and the walkthrough bullet at line 30 are present as cited.
- *The three 2026-05-29 user-confirmed decisions are coherently reflected* across "Patterns this replaces", "Default justifications", "Naming audit", and WS-A/WS-B/WS-C. Not re-litigated.
- *Coherence interrogation of both governed notebooks passes the current editorial bar.* WS-A's narrowed `computing_graph_metrics.ipynb` is a coherent single-class HivePlot gallery (class-scope clean once cells 16-25 leave; Karate + toy dataset set justified; the user-endorsed Karate-two-ways is a deliberate choice, not drift). WS-C's `finding_a_partition.ipynb` is a coherent conceptual-deep-dive tutorial (right genre, Les-Mis dataset justified as the "no natural partition" vehicle, spine does not drift into survey territory). The earlier `low-confidence` worry — a HivePlot-class page whose core demo migrates to HPM — is exactly what WS-A removes and WS-C re-homes as a *motivated* tutorial, which is the canonical resolution, not a new flag.

**Amended (the re-sync exposed these gaps):**
- *Stale report-format vocabulary corrected.* The plan described editorial-critic as returning "`clean` or `worth-discussing`-only"; the current def's format is `Status: clean | propose` with findings tagged `[must-fix | worth-discussing | low-confidence]` at `<file>:<cell>`. Rephrased to "`Status: clean`, or `propose` with no `must-fix` finding" in WS-A done-when #11, WS-C done-when #10, and the Critic-coverage block.
- *Division of labor with viz-critic made explicit per the current def.* Critic-coverage block now states editorial-critic **reads prior viz-critic reports as input** and **cites rather than re-reviews** figures, and that scope-crossing findings are `must-fix` routing through `amend-plan` for user sign-off.
- *Two under-specified editorial-bar checks wired into done-whens.* (i) **Genre fit for WS-A**: the gallery must not drift toward tutorial voice now that its narrative HPM arc has left (the inverse of WS-C's "not a gallery page" check). Added to WS-A specialist line + done-when #11. (ii) **Section-worth for WS-C** under the NARROW-scope amendment: editorial-critic checks the draft adds no survey section the cleaned gallery or `hpm_from_variable_sweep.ipynb` covers better. Added to WS-C specialist line + done-when #10.
- *WS-B editorial-critic framing corrected to the def's post-impl-only restriction.* The plan had floated an optional "lightweight editorial-critic look" at the no-notebook gate; the current def restricts editorial-critic to post-impl / pre-merge. Changed to "editorial-critic does NOT run at this gate" (planning-time arc-vs-data coherence is the orchestrator's job, already done here); the structured pass stays on the WS-C draft. The plan did NOT mis-task editorial-critic at planning time anywhere else.

*Sections edited:* "Critic coverage" block; WS-A specialist line + done-when #11; WS-B specialist line; WS-C specialist line + done-when #10.

### 2026-05-29 (WS-A post-impl) — editorial-critic `must-fix`: switch-paragraph promises "toy," but the Per-Metric Keyword Arguments section runs on Karate

**In-scope tweak.** A WS-A post-impl editorial-critic `must-fix` (dataset-coherence, lead-in-vs-body), routed here under rule 14. Single specialist (notebook-author), one file (`examples/computing_graph_metrics.ipynb`), no done-when removed, no specialist set change, no API surface, sequencing unchanged. It refines WS-A's already-shipped done-when #5 (the two-dataset justification prose). It is the only blocker to closing WS-A: done-whens #1-#10 pass and the WS-A viz-critic returned `good`. This is **not** covered by amendment 1 — amendment 1 endorses Karate-partitioned-two-ways (the *partition variable*: `club` plus the one `degree`-bin mechanic section); this finding is a *dataset/lead-in* contradiction (a section the new switch paragraph promises is "on the toy" actually runs on Karate). Distinct axis, distinct fix.

**Independently verified against the post-WS-A notebook (read-only; cell-id-anchored, robust to renumbering):**
- The notebook has exactly **two** top-level `G =` assignments, both `nx.karate_club_graph()`: cell `e5a52839` (early, the gallery's Karate setup) and cell `c0370996` (late, inside `### When Initializing from graph Parameter`). `G` is **never** reassigned between them.
- The switch paragraph (markdown `40821661`, the WS-A-added two-dataset justification under `## Add Node and Edge Graph Metrics to Existing Hive Plot`) reads: "The remaining sections switch from Karate Club to the synthetic `example_hive_plot()` toy ... the cleaner surface for the pure-mechanics sections that follow."
- The `## Per-Metric Keyword Arguments` section (markdown `f73e17d0` + code `22319f8f`) is an H2 *after* that switch paragraph, but its code is `HivePlot(graph=G, partition_variable="club", ...)` — i.e. **Karate** (it sits between cell `e5a52839` and the cell `c0370996` reassignment, and `partition_variable="club"` only resolves on Karate; the toy `example_hive_plot()` never touches `G`). Real lead-in-vs-body contradiction. **Confirmed.**
- The intervening toy sections are correct: `## Add Node and Edge Graph Metrics to Existing Hive Plot` (`add86186`), `## Resolving Column Name Collisions` (`c4f2a4d5`), and `## Controlling the HivePlot Internal Graph Type` / `### When Initializing from nodes and edges Parameters` (`d8ff7467`, `e6ca4494`, `c691a9b5`) all use `example_hive_plot(...)`.

**Related `worth-discussing` (folded into this fix, since option (a) is chosen):** even with the Karate section moved out, the switch paragraph's "the remaining sections / sections that follow" is too absolute, because `### When Initializing from graph Parameter` (code `c0370996` → `663a1ba4`) **deliberately and correctly returns to Karate** — it re-assigns `G = nx.karate_club_graph()` because it needs an *undirected input graph* to demonstrate the graph-type-from-input default. Verified. So the switch sentence must be scoped to the sections it actually covers, with a one-clause "back on Karate here, because we need an undirected input graph" signal where Karate returns.

**Chosen resolution direction — option (a) (relocate), lower-touch:** move the entire `## Per-Metric Keyword Arguments` section (markdown `f73e17d0`, code `22319f8f`, trailing markdown `ae74a57a`) to *before* the switch paragraph (`40821661`), placing it inside the Karate run — natural slot: immediately after the `### Link Prediction Scores as an Edge Metric` section (after `link-prediction-inspect-md`, `23a...`/`link-prediction-inspect-md`) and before `## Add Node and Edge Graph Metrics to Existing Hive Plot`. The section already runs on `graph=G`/`club`, so **no code or prose change inside the section** — it is a 3-cell reorder. This removes the contradiction with the least churn and keeps the toy run (`existing-HP` → `collisions` → `internal-graph-type from nodes/edges`) contiguous. Option (b) (re-cast the section onto the toy) was the higher-touch alternative and is not chosen: it rewrites a working, correct Karate demo to chase the prose rather than fixing the prose, and the per-metric-kwargs demo (`betweenness_centrality` with `k`/`seed` sampling) reads fine on Karate. **Fold in the worth-discussing prose softening in the same edit:** scope the switch sentence to the sections it covers, and add the one-clause "back on Karate here, because we need an undirected input graph" signal at the `### When Initializing from graph Parameter` heading (`4ae9c0a5`) / its first code cell (`c0370996`).

**WS-A done-when(s) touched:** #5 (the two-dataset justification prose is now correct after the lead-in is scoped and the Karate section relocated above it) and, by extension, #11 (re-run editorial-critic to clear the `must-fix`). #8 (`make test-nb` green) and #9 (voice scan) must be re-confirmed after the edit. No done-when is added or removed.

*Sections edited:* "Plan amendments" (this entry). No other plan section changes — WS-A's done-whens stand as written; this records the in-flight fix and routes it.

## Holdouts

The replace-and-sweep / cross-link audit should leave these alone:

- `docs/source/conf.py:61`, `docs/source/gallery_examples/index.rst:19` and `:37`, `CHANGELOG.rst:30` — `computing_graph_metrics.ipynb` stays a HivePlot gallery in "The HivePlot Class"; these references remain valid after narrowing. Not replace targets.
- The 11 generic metric-reference cross-links into `computing_graph_metrics.ipynb` (`introduction_to_hive_plots`, `quick_hive_plots`, `karate_club`, `networkx_examples`, `creating_hive_plots_from_networkx`, `hive_plot_matrices`, `hive_plots_for_large_networks`, `exporting_hive_plots_to_networkx`, `visualizing_node_metadata`, `visualizing_edge_metadata`, `hpm_from_tags`, `hpm_generic`) — kept as-is; they point at the reference content (tables, requesting metrics) that stays in the narrowed gallery. Verified-still-correct in WS-D, not rewritten.
- `examples/computing_graph_metrics.ipynb` cells 7 and 27 retain their `reset_edges(...)` calls (real work); only the *unexplained-repetition* prose pattern is replaced (WS-A done-when #4).
- Rendered-traceback output cells in `computing_graph_metrics.ipynb` (the directed/undirected `ValueError` demos, cells 52 and 59) regenerate on `make test-nb`; not manual edit targets.
- All references inside `wiki/wiki/plans/*.md` and `agent-harness/` are plan/harness history, not shipped artifacts; out of audit scope.

## Implementation log

Append-only. After each workstream completes, one line in the same turn.

- **2026-05-29 — WS-A implementation (notebook-author).** Cleaned `examples/computing_graph_metrics.ipynb` into a HivePlot-only, NetworkX-only gallery: removed the two HPM blocks ("Using Community Detection as a Partition Variable" + "Precompute Once, Reuse Across Plots", 10 cells), so no `HivePlotMatrix`/`from_partition`/Louvain/Les Misérables remains; pruned `HivePlotMatrix` and the inline cross-tab `pandas` from imports (kept `ScalarMappable`/`Normalize` for the retained Jaccard cell); kept "Using a Computed Metric as a Partition Variable" on Karate `degree` (heading preserved for the `setting_partition_variable.ipynb:699` anchor) and reframed it as a discretize-the-metric *mechanic* demo, not a meaningful-grouping claim; added a `reset_edges` explainer markdown cell at its first appearance (links `adding_repeat_axes.ipynb`) and reduced the two later occurrences to one-line back-references (calls retained in all three); stated the two-dataset justification (undirected Karate vs. directed-by-default storyless toy) where the toy first appears; closing pointers stay as gallery prose paragraphs. Notebook executes end-to-end with warnings-as-errors (`pytest -c tests/pytest_examples.ini -k computing_graph_metrics` PASSED; re-executed inplace to refresh outputs); ruff format+check clean; voice scan clean (no em-dashes, no AI filler); only `examples/computing_graph_metrics.ipynb` edited. **Pending:** post-impl editorial-critic + viz-critic (done-when #11). **Deferred:** the `finding_a_partition.ipynb` closing pointer is NOT added (WS-C has not run; notebook does not exist), per done-when #7 — it lands in WS-D so it is not dropped.
