# Roadmap: Graph Metrics for Deeper Hive-Plot Interpretation (tutorial series + new docs section)

<!--
PROGRAM / ROADMAP plan, NOT a single-PR implementation plan. Hiveplotlib consumer
work; tracked in the wiki submodule. The defining constraint: phased and
NON-BLOCKING. Each tutorial is independently shippable; nothing blocks anything
else; the section goes live small and grows. Detailed, cell-aware workstreams
exist for Phase 1 ONLY. Everything past Phase 1 is a "future tutorials" roadmap:
theme + knob + candidate dataset(s) + risk note + gnn-heterogeneity mapping,
promoted to a real workstream (via orchestrator amend-plan) when its phase comes.
CHANGELOG routing is CHANGELOG.rst (the v0.28.0 WIP entry), not a harness CHANGELOG.
-->

## Goal

Build a tutorials **section** teaching how graph metrics drive deeper hive-plot interpretation on real data. The organizing idea: **a hive plot is three choices** -- a *partition* (which axis a node sits on), a *sorting* (where on the axis it sits), and *edge styling* (how edges are drawn) -- and graph metrics feed all three, while a `HivePlotMatrix` lets you compare those choices. One tutorial per knob, one that ties them together, then a capstone that uses all three to surface structural heterogeneity on plain data.

When this lands, a reader who knows how to *build* a hive plot (the conceptual intros) and how to *compute* metrics (the `computing_graph_metrics.ipynb` gallery) but does not know *which lens reveals what* has a narrative home. The section grows one independently-shippable tutorial at a time; it can go live with two and still read as complete.

This is a **docs-only program**. No user-facing API surface changes anywhere in the series (confirmed by feasibility audit below: every metric and construction mode the tutorials need already ships). api-critic is **N/A series-wide**; see "API surface (none)".

## Prior ADRs / design docs

Populated from research-liaison pre-task findings. No ADRs exist yet (`wiki/wiki/adr/` is absent). The binding decision records are the prior plans and the conceptual analyses.

- **`wiki/wiki/plans/graph-metrics-notebook-restructure.md` -- Tutorial 1's binding plan (the partition knob).** This is the load-bearing prior decision for the whole series. It encodes, and this roadmap inherits series-wide:
  - The **corpus conventions** for a real-data tutorial (subtype, `#### Background`, `#### References`, install-extras up front, prose-to-code balance).
  - The **per-tutorial build pattern** (dataset/story validate-then-commit gate -> draft -> editorial-critic + viz-critic post-impl -> docs registration). This roadmap promotes that pattern to a named, reusable section convention (see "The reusable per-tutorial build pattern").
  - The honest **">3 groups -> switch to `HivePlotMatrix`, do NOT tune community resolution down to fake 3"** rule, explicitly reaffirmed there as anti-dishonest-pedagogy (its rejected-alternative decision). Adopted series-wide.
  - The **gallery-vs-tutorial split**: `computing_graph_metrics.ipynb` is the HivePlot mechanics *gallery* ("how do I compute a metric"); the tutorials are the *interpretive* layer ("which lens reveals what"). The series never re-treads gallery mechanics.
  - The **docs-registration mechanics**: register in `docs/source/notebooks/index.rst` (both the `nblinkgallery` block and the hidden `toctree`, same order), add a `nbsphinx_thumbnails` entry in `conf.py`, produce a `_static/` thumbnail, update the `CHANGELOG.rst` v0.28.0 WIP entry.
  - **`finding_a_partition.ipynb`** (Les Misérables, partition-discovery arc) is Tutorial 1 and is **authored by this plan as WS-0** (re-homed from the restructure plan's WS-C/D; see Plan amendments 2026-06-17). T1's full cell-aware spec — dataset/story gate, narrative spine, NARROW-scope discipline, community-labeling default/fallback, naming audit, dataset justification, and API usage sketches — now lives **in this plan** (WS-0, plus the "Patterns this replaces" / "Default justifications" / "Naming audit" / "API usage examples" sections below); it was moved out of `graph-metrics-notebook-restructure.md` on 2026-06-17 so this series plan is self-contained. The restructure plan retains only its gallery-cleanup half (its WS-A) and its history. T1's filename and title are user-confirmed; do not re-audit them.
- **`wiki/wiki/analyses/gnn-heterogeneity-hive-plots.md` -- the conceptual parent (with the GNN/ML coupling stripped out).** Its four knob->interpretation mappings *are* the four spine tutorials, and its heterogeneity thesis *is* the capstone. The full mapping is in "The gnn-heterogeneity mapping" below. **Boundary (settled):** the ML coupling stays in the future-paper lane. The series is PLAIN DATA. See "Series-wide constraints" for the exact strip-out list.
- **Interpretive anchors that exist and back the tutorials' prose** (do not create new wiki pages; see "Concept-page gaps"):
  - `wiki/wiki/analyses/karate-club-walkthrough.md` -- the plain-data interpretive model (Tutorials 1-3). Carries the "bridge-building is independent of within-faction popularity" reading: cross-faction (gray) edges are distributed across all axis positions, not clustered at the high-degree tips. That is a sorting-reveals-structure reading told without any ML.
  - `wiki/wiki/concepts/node-assignment.md` -- partition + sorting (Tutorials 1, 2); the **only** grounding for directed source/manager/sink (Directed expansion). Krzywinski's directed scheme (sources = out-only, managers = both, sinks = in-only) and the undirected clustering-coefficient 3-band scheme both live here.
  - `wiki/wiki/concepts/edge-rendering.md` -- edge styling channels (color/thickness/opacity/linestyle/direction) + the kwarg hierarchy + tags (Tutorial 3). **Note:** it covers styling *mechanics* but no page bridges edge-*metric* -> edge-*color*; Tutorial 3 does that bridging inline.
  - `wiki/wiki/concepts/hive-plot-matrix.md` -- the four HPM modes; the Krzywinski 5x5 "Hive Panel" is itself a metric-sweep template (Tutorial 4).
  - `wiki/wiki/concepts/structural-heterogeneity.md` -- the capstone anchor. Its structural-property -> hive-plot-role table is a **capstone storyboard**, but **drop its GNN-performance / GNN-relevance column** (that is the ML lane).
- **Supporting framing (not constraints):** the tutorial/gallery conventions live in the harness skills (`hiveplotlib-tutorial-notebook`, `viz-quality-bar`), not the wiki.

## API surface (none)

**No user-facing API surface change anywhere in this series. api-critic is N/A series-wide** (per the consumer trip-wire and `mental-model` rule 7: docs-only work needs no api-critic, planning or post-impl). The series is an *application* of existing functionality, which the feasibility audit confirms:

- **Every metric the spine + capstone + expansions reach for already ships** in `src/hiveplotlib/graph_features/networkx/` (verified against `node_metrics.py` and `edge_metrics.py` on this branch):
  - Sorting tutorial (T2): `degree`, `betweenness_centrality`, `pagerank`, `core_number`, `harmonic_centrality`, `average_neighbor_degree` -- all present.
  - Edges tutorial (T3): `edge_betweenness_centrality`, `bridges`, `jaccard_coefficient`, `adamic_adar_index` -- all present. **Pedagogical wrinkle to use, not hide:** the link-prediction wrappers default `ebunch=graph.edges()`, so they score *existing* edges (NetworkX's own default scores non-edges). That reinterpretation is exactly the "expected vs. surprising" reading T3 wants -- a high Jaccard on an existing edge = a structurally expected link; a low score on an existing edge = a surprising one. The wrapper docstrings carry the caveat; T3's prose should surface it in one sentence so a NetworkX-fluent reader is not confused.
  - Directed expansion: `in_degree`, `out_degree`, `topological_generations`, `reciprocity`, `strongly_connected_components`, `weakly_connected_components` -- all present, all with the directed-graph guard.
  - Core/periphery expansion: `core_number`, `onion_layers`, `clustering` -- all present.
  - Community detection (T1, capstone): `louvain_communities`, `greedy_modularity_communities`, `label_propagation_communities` -- all present, with the descending-size label-order contract (label 0 = largest).
- **Every construction mode the series uses already ships:** `HivePlot(graph=..., node_graph_metrics=[...], ...)`, `compute_graph_metrics(...)`, `NodeCollection.create_partition_variable(...)`, `HivePlotMatrix.from_partition(...)`, `HivePlotMatrix.from_variable_sweep(...)`. The four-mode HPM surface is documented in `hive-plot-matrix.md`.
- **Feasibility-audit recovery clause.** If, while drafting any future tutorial, the notebook-author finds the API usage sketch needs a helper that is *not* on the shipped surface (i.e. the sketch invents an unauthorized convention), that is a `mental-model` rule-9 surface-back to orchestrator `amend-plan` for a feasibility check, *then* api-critic if a real surface change is implicated. Default expectation: no surface change is needed; this is a docs program.

### T1 API usage sketches (WS-0; notebook content, not API surface)

These are **notebook-content sketches** for T1's key cells (moved here from `graph-metrics-notebook-restructure.md` 2026-06-17 so this plan holds T1's full spec). They are runnable Python a notebook-author can build from, not final cells (prose, titles, figure polish are the notebook-author's craft under the tutorial skill). All use only shipped surface (`compute_graph_metrics`, `create_partition_variable`, `from_partition`, plain pandas for the label map); no new symbols. The tutorial surfaces `pip install hiveplotlib[networkx,datashader]` up front.

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

## Patterns this replaces

This is a net-new docs section. The audit here is **section-worth / no-duplication** (the brief's explicit ask): map every proposed tutorial's payoff against the already-shipped corpus so the roadmap never authorizes a notebook a sibling already covers better. Grepped/inspected against the current branch.

**Migration (settled boundary -- only one notebook folds in):**

- **`finding_a_partition.ipynb` (Les Misérables) is Tutorial 1, authored by this plan as WS-0** (re-homed from `graph-metrics-notebook-restructure.md`'s WS-C/D; see Plan amendments 2026-06-17). It instantiates the per-tutorial build pattern against the cell-aware spec **this plan now holds in full** (WS-0's three sub-steps: dataset/story gate + draft + tutorial-side docs registration). WS-1 then creates the new section heading and confirms T1 sits in its `index.rst` slot. (The spec was moved out of the restructure plan on 2026-06-17 so this plan is self-contained; the restructure plan now holds only its gallery-cleanup WS-A.)
- **`hive_plots_more_than_three_groups.ipynb` stays standalone and is only linked.** Inspected (cells 0, 19-34): it is *HPM-as-the-fourth-of-four-overflow-options* for a network that *naturally* partitions into >3 groups (trade data, `continent` partition, datashader). Its four options are "more axes / two layers / collapse-to-3 / HPM small multiples." **Tutorial 4 is a different question** ("use an HPM to *compare candidate lenses* and discover which reveals structure"), so they cross-link, they do not duplicate. De-confliction spelled out under Tutorial 4 below.
- **`karate_club.ipynb` stays the general intro.** Not migrated; the section links to it where the karate-club interpretive model is the cleanest illustration (e.g. T2's assortativity reading).

**Section-worth map (each proposed tutorial vs. the shipped notebook that is closest):**

| Proposed tutorial | Closest shipped notebook(s) | Why the tutorial is not a duplicate |
| --- | --- | --- |
| **T1 Finding a Partition** | `computing_graph_metrics.ipynb` (gallery), `setting_partition_variable.ipynb`, `create_partition_variable.ipynb` | Already adjudicated in the T1 plan: the gallery owns the *mechanic* (discretize a metric); T1 owns *why you reach for it* on data with no natural partition, ending in community detection -> HPM. |
| **T2 Sorting with Graph Metrics** (working title) | `setting_sorting_variables.ipynb` (gallery: the `sorting_variables` *parameter* mechanics, on toy data), `karate_club.ipynb` (single sort, single figure) | T2 is the *interpretive* arc: sort by degree vs. betweenness vs. pagerank vs. core-number on one real graph and read *what each surfaces*; the assortativity-as-geometry payoff (sort both axes by degree). The gallery teaches the knob; T2 teaches what turning it does. |
| **T3 Reading the Edges** | `visualizing_edge_metadata.ipynb` (gallery: attach an edge metadata column, H1 genre), `edge_kwarg_hierarchy.ipynb` (the kwarg precedence reference), `computing_graph_metrics.ipynb` ("Link Prediction Scores as an Edge Metric" mechanic) | No shipped page bridges edge-*metric* -> edge-*color interpretation* (confirmed against `edge-rendering.md`, which is styling mechanics only). T3 is that bridge: color/weight by edge-betweenness (bridges/bottlenecks) and by link-prediction (expected vs. surprising). |
| **T4 Comparing Lenses with an HPM** | `hpm_from_variable_sweep.ipynb` (gallery: every `from_variable_sweep` option/kwarg, on abstract data), `hive_plots_more_than_three_groups.ipynb` (HPM as >3-group overflow), `hive_plot_matrices.ipynb` (HPM concept intro) | The sweep gallery is the *feature reference*; T4 is the *interpretive arc* (use a sweep to discover which lens reveals structure on a real graph where you do not yet know). De-conflict with more-than-three-groups: different question (compare-lenses vs. fit->3-groups). |
| **Capstone Finding Surprising Connections** | -- (no shipped notebook does multi-knob structural-heterogeneity on plain data) | Net-new payoff: all three knobs + an HPM together to surface unusual heterogeneity. The plain-data realization of the gnn-heterogeneity thesis. |
| **Core and Periphery** (expansion) | `setting_sorting_variables.ipynb`, T2 | Risk-flagged thinnest-grounded; only promote if it earns a payoff distinct from T2's core-number sort. |
| **Directed: Sources and Sinks** (expansion) | `node-assignment.md` (directed scheme), `hive_plots_more_than_three_groups.ipynb` (the trade dataset) | The only spine/expansion that *needs* a directed graph; payoff (source/manager/sink partition via in/out-degree) is distinct from every undirected tutorial. |

**Holdouts (NOT touched by this program):** every existing notebook except the migration of `finding_a_partition.ipynb` (placed, not edited) and the optional forward-link sentences added to existing notebooks when both link endpoints exist (graceful; see "Cross-links"). No source symbols. `computing_graph_metrics.ipynb` and the HPM galleries keep their mechanics scope; the series sits *beside* them as the interpretive layer.

## Default justifications

No new user-facing API defaults (docs-only). The decisions that need justifying are **the section's identity**, the **best-fit-dataset-per-tutorial strategy**, and **each Phase-1 dataset**.

- **The section is scoped tight: graph metrics -> deeper interpretation -> real data.** Justification: the corpus already has the *mechanics* layer (galleries) and the *general intros* (`karate_club`, the conceptual intros). The gap a user falls into is "I can build a hive plot and I can compute a metric, but I don't know which lens answers my question." That gap is the section's reason to exist, and keeping it tight is what stops every tutorial from drifting into a general graph-metrics survey (the explicit anti-goal).
- **Best-fit-dataset-per-tutorial is the strategy; cross-tutorial dataset consistency is a welcome bonus, not a requirement.** Justification: each knob is best motivated by a graph whose structure *can* show that knob's payoff and *cannot* be read off a label. Forcing one dataset across the series would compromise some tutorials (e.g. an undirected graph cannot motivate the Directed expansion; a graph with an obvious partition cannot motivate T1's discovery arc). Carry the T1 plan's per-dataset discipline: each dataset is justified by what structure it *can* and *cannot* motivate.
- **Phase-1 datasets:**
  - **T1 (WS-0): Les Misérables co-appearance network** (user-sanctioned; the justification now lives here, moved from the restructure plan 2026-06-17). The canonical "no natural partition" network: 77 character nodes, edges = co-appearance in the novel, no ground-truth grouping (contrast Karate's `club`, contrast the trade data's `continent`). This is the dataset that motivates "discovery" because the reader cannot fall back on a pre-labeled grouping. Two T1-specific display/labeling justifications go with it: (i) **communities named by most-central character, not integer labels** — numbered community labels are the gallery anti-pattern, but here *teaching the community-detection technique is the point*, so the labels are load-bearing and deserve human-readable names; naming by the most-central member (e.g. by `harmonic_centrality` or `degree` within each community) gives the reader a handle and as a bonus sidesteps the deferred `from_partition` int->str coercion; (ii) the tutorial is **a graph-theory journey, not literary analysis** — one sentence says the reader does not need to know the novel.
  - **T2 (new, Phase 1): leans Les Misérables, with Karate Club as the 2-faction baseline foil.** Justification (to be confirmed at WS-2's validate-then-commit gate, not locked here): the sorting payoff needs a graph with *visible variation* across multiple metrics (degree, betweenness, pagerank, core-number must each surface a *different* node ordering, or the tutorial has no story). Les Mis has a heavy-tailed degree distribution and clear bridge characters, so betweenness and degree should diverge legibly; Karate is the textbook assortativity illustration (the karate-walkthrough's "bridges are independent of popularity" reading) and is small enough to show the both-axes-sorted-by-degree geometry cleanly. The gate decides which carries the spine and which (if any) is the foil; the dataset is *not* pre-committed. Avoiding Cora/CiteSeer/PubMed/OGB-arxiv is mandatory series-wide (they drag the ML framing in).

## Naming audit

No new parameters, methods, or classes (docs-only). The durable user-facing names this program introduces are **the new docs section name** and **the Phase-1 tutorial filename + title** (`finding_a_partition.ipynb` is user-confirmed and not re-audited). Future-tutorial names are proposed below but only locked when their phase is promoted.

Checked against the established `examples/` + `docs/source/notebooks/index.rst` conventions: tutorials carry `### H3` titles that are noun/verb phrases naming a concept or dataset (`### Comparing Network Subgroups`, `### Hive Plots with More Than 3 Groups`, `### Zachary's Karate Club`); files are snake_case; the index groups them under named `.rst` sections (`Hive Plots`, `Hive Plot Examples`, `Polar Parallel Coordinates Plots`, `P2CP Examples`).

### Section name (durable -- USER-CONFIRMED)

The section is a new heading in `docs/source/notebooks/index.rst` grouping these tutorials.

**Confirmed name: `Using Graph Metrics with Hive Plots`** (user decision, 2026-05-29; see Plan amendment 2026-05-29 #1). This replaces the prior recommendation `Interpreting Hive Plots with Graph Metrics`. Rationale:

- "Interpreting" undersells the section. The section covers using metrics to *construct* hive plots (metrics as sorting variables and partition variables), not only to color or interpret them. "Using" captures both the construct and the read.
- "Using Graph Metrics with Hive Plots" pairs cleanly with the `computing_graph_metrics.ipynb` gallery: *compute* the metrics (gallery = mechanics) then *use* them (this section = application). That compute-vs-use distinction is the gallery-vs-section boundary.

Candidates weighed before the decision:

| Section heading | Read |
| --- | --- |
| **`Using Graph Metrics with Hive Plots`** (USER-CONFIRMED) | Names both the construct and the read (*using* covers sorting/partition variables, not just coloring); pairs with the `computing_graph_metrics.ipynb` gallery on a clean compute-vs-use boundary. Distinguishes cleanly from the `Hive Plots` (concept/mechanics tutorials) and `Hive Plot Examples` (single real-data walkthroughs) sections already present. |
| `Interpreting Hive Plots with Graph Metrics` (prior recommendation, superseded) | Names a payoff (*interpreting*) and the technique, but "interpreting" undersells the construct half of the section (metrics as sorting/partition variables), so the user replaced it. |
| `Graph Metrics` | Too close to the `computing_graph_metrics.ipynb` *gallery* title; reads as a mechanics section, not an application one. Invites the survey drift the series forbids. |
| `Graph Metrics for Hive Plots` | Accurate but ambiguous between "computing" and "using"; the confirmed heading's "Using" verb resolves that. |
| `Deeper Hive Plot Interpretation` | Drops the "graph metrics" hook that ties the section to its conceptual spine and to the gallery it complements. |

Placement in `index.rst`: a new section after `Hive Plots` and before (or after) `Hive Plot Examples`; the exact ordering is a Phase-1 WS-1 detail, but the section comes after the conceptual `Hive Plots` tutorials since it assumes the reader can already build one.

**Section tutorial-title trend.** The section's tutorial titles trend toward the "X with Graph Metrics" parallel (T1 `### Finding a Partition with Graph Metrics`, T2 working title "Sorting with Graph Metrics"); each tutorial's final title is settled per-tutorial at authoring.

### Phase-1 tutorial names

- **T1: `finding_a_partition.ipynb` / `### Finding a Partition with Graph Metrics`** -- **user-confirmed** (the audit now lives here, moved from the restructure plan 2026-06-17; do not re-audit). Reasoning that held up: (1) "partition" is the established corpus vocabulary (`partition_variable`, `setting_partition_variable.ipynb`, `from_partition`), so a reader looking to solve "what do I partition on?" recognizes the title immediately; (2) "with Graph Metrics" ties it to the gallery it splits from and signals the technique; (3) "Finding" carries the discovery narrative the tutorial is built around (better than "Choosing," which understates that the data gives you *neither* a partition nor a sorting); (4) it does not over-index on "community detection," which is one step of the arc, not the whole thing; (5) snake_case file + `### H3` concept-noun-phrase title matches the conceptual-deep-dive subtype (`comparing_network_subgroups.ipynb`, `hive_plots_more_than_three_groups.ipynb`). Rejected candidates: `community_detection.ipynb` (too narrow — the back half of the arc only, and reads as a gallery feature-reference title); `les_miserables.ipynb` (dataset-forward, but the dataset is the vehicle, not the lesson); `choosing_a_partition.ipynb` (understates discovery). **Community-label display names:** the wrapper returns integer labels `0..5` (sorted largest-first per the label-order contract); T1 maps each to its most-central character's name for display (e.g. `"Valjean's circle"`). These are prose-only display strings, not API names; they derive *from* node ids, so any "Valjean" label is the actual node id — no collision. Audit OK.
- **T2: working title `### Sorting with Graph Metrics`; filename PROVISIONAL, finalized at WS-2 authoring** (user guidance, 2026-05-29; see Plan amendment 2026-05-29 #2). Filenames are pragmatic working names; reader-facing titles are what get polished and are freely renamable. The working title "Sorting with Graph Metrics" parallels the likely T1 final title `### Finding a Partition with Graph Metrics` -- both end in "with Graph Metrics" and both sit under the `Using Graph Metrics with Hive Plots` section. **The filename is NOT locked**; it is decided at WS-2 authoring. Open choice:
  - `sorting_with_graph_metrics.ipynb` -- concept-phrase, matches T1 and the tutorial-corpus norm. The dispatching session's mild lean, since the `hpm_*` topic-prefix style is reserved for the gallery families.
  - `graph_metrics_sorting.ipynb` -- topic-prefix; the user's working assumption.
  - (The prior recommendation `sorting_to_reveal_structure.ipynb` is withdrawn.) Checked against corpus conventions: a concept-phrase `### H3` title matches the `Finding a Partition` sibling and the `### Comparing Network Subgroups` shape; snake_case file; does not collide with the `setting_sorting_variables.ipynb` *gallery* (distinct: "setting" = the parameter mechanic, "sorting with graph metrics" = the application). **Flag the filename for confirmation at WS-2 authoring.**

### Future-tutorial names (proposed, locked at promotion only)

Not audited for collisions yet (done when each is promoted via amend-plan): `reading_the_edges.ipynb` / `### Reading the Edges`; `comparing_lenses.ipynb` or `comparing_lenses_with_a_matrix.ipynb` / `### Comparing Lenses with a Hive Plot Matrix`; capstone `finding_surprising_connections.ipynb` / `### Finding Surprising Connections` (or `### Surfacing Structural Heterogeneity`); expansions `core_and_periphery.ipynb`, `directed_sources_and_sinks.ipynb`. These are placeholders to make the roadmap readable, not commitments.

## The gnn-heterogeneity mapping

`gnn-heterogeneity-hive-plots.md` is the conceptual parent. Its four knob->interpretation mappings are the four spine tutorials; its heterogeneity thesis is the capstone. Cite it as the parent in each tutorial's `#### References` or a "this idea generalizes" pointer, but **strip the ML coupling** (see constraints). The mapping, plain-data-translated:

| gnn-heterogeneity mechanism | Spine tutorial | Plain-data translation (the ML strip-out) |
| --- | --- | --- |
| Decompose performance *by a structural property* (axis assignment by degree/community) | **T1 partition** | A real node attribute or a detected community label replaces "predicted label"; the partition is chosen for structure, not to bin model error. |
| *Continuous axis sorting* captures gradients that 5-bin bar charts flatten | **T2 sorting** | Sort by a structural metric to reveal a continuous gradient; the gnn page's "bins flatten non-linear patterns" argument is the conceptual backing. No confidence-as-sort. |
| *Edge color* by a structural/edge property; errors concentrate on specific edge types | **T3 edges** | Color by edge-betweenness (bridges) and link-prediction (expected vs. surprising); a structural edge metric replaces "correct vs. misclassified." |
| *Sweep partition variables across the matrix* to discover the most-informative lens | **T4 HPM** | Sweep candidate sorting/partition lenses to discover which reveals structure; no model-vs-model (GCN/GAT/GraphSAGE) row sweep. |
| The **heterogeneity thesis** (aggregate metrics mask structural performance variation) | **Capstone** | Use all three knobs + an HPM to surface *structural* heterogeneity (unusual connections, uneven topology) on a plain real network; no fairness framing, no training-set-distance variable. |

## Series-wide constraints

These apply to every tutorial; each future workstream inherits them by reference.

- **Tutorial genre throughout** (`hiveplotlib-tutorial-notebook` skill): narrative arc, real data, rhetorical question posed early and revisited, `#### Background` (provenance) + `#### References` (citations), `### H3` title + `#### H4` subsections, prose-to-code balanced. editorial-critic enforces genre fit and section-worth per tutorial.
- **Keep each tutorial NARROW to its knob.** Do not drift into a general graph-metrics survey. That survey content already lives in `computing_graph_metrics.ipynb` (sorting/edge mechanics, per-metric kwargs, collisions, internal graph type) and the HPM galleries (`hpm_from_variable_sweep.ipynb`, `hpm_from_partition.ipynb`). A broad "everything you can do with graph metrics" notebook is explicitly OUT of scope for every tutorial and is not a scheduled future tutorial.
- **The honest ">3 groups -> switch to `HivePlotMatrix`, do NOT tune community resolution down to fake 3" rule, series-wide.** Reaffirmed from the T1 plan as anti-dishonest-pedagogy. Any tutorial that runs community detection and gets >3 groups routes to an HPM rather than tuning the resolution down.
- **PLAIN DATA, NOT GNN/ML.** Cite `gnn-heterogeneity-hive-plots.md` as the conceptual parent, but the ML coupling stays in the future-paper lane. **Leave out, series-wide:** training-set-distance variables, softmax-confidence-as-sorting, correct/misclassified-as-edge-color, the GCN/GAT/GraphSAGE model comparison, and the fairness-literature framing. **Plain-data substitution:** a real node attribute or a community label replaces "predicted label"; a real structural metric replaces "confidence." **Avoid the Cora / CiteSeer / PubMed / OGB-arxiv datasets entirely** (they drag the ML framing in).
- **Only edit notebooks under `examples/`.** The `docs/source/notebooks/` and `docs/source/gallery_examples/` copies auto-generate on `make docs`.
- **Each tutorial registers** in `docs/source/notebooks/index.rst` (both the `nblinkgallery` block and the hidden `toctree`, same order) under the new section, plus a `_static/` thumbnail and a `conf.py` `nbsphinx_thumbnails` entry.
- **Notebooks run end-to-end** (`make test-nb`, warnings-as-errors). datashader/networkx extras surfaced up front where needed.
- **House voice** (no em-dashes, no AI filler, length discipline against the closest sibling). Tutorial skill's voice section is the calibration.

## Concept-page gaps (flag only; do NOT create -- new wiki pages need separate user approval)

The brief's research findings flag these concepts as lacking their own wiki page. The tutorials explain them **inline**; growing wiki pages for them is a **separate, optional, user-approved effort** outside this program's scope.

- centrality (as a family); community detection; assortativity / mixing; core-periphery / k-core; clustering coefficient (best-grounded today: the Krzywinski 3-band partition lives in `node-assignment.md`); link prediction; edge-metrics-as-a-concept; directed-structure-as-a-concept.

The best-grounded existing anchors per tutorial are listed in "Prior ADRs / design docs". A tutorial does not block on a missing wiki page; it carries the explanation in its own prose with a `#### References` citation to the primary literature.

## The reusable per-tutorial build pattern

Pinned ONCE here; every tutorial in the series (T1 already follows it; T2 and all future tutorials instantiate it). This is the same shape as the T1 plan's WS-B/WS-C/critic/WS-D sequence, promoted to a section convention so the roadmap defines it once.

Each tutorial is built in four steps. The first is a hard gate.

1. **Dataset / story validate-then-commit gate (VALIDATE-THEN-COMMIT).** Before any drafting cost is sunk, render the tutorial's *payoff figure(s)* on the candidate dataset to scratch (`/tmp/`, per `mental-model` rule 16 -- never the repo tree). Record a go/no-go in the dispatch report (not the repo): does the structure the tutorial promises actually *show* on this data at notebook render size, and does the narrative arc hold? viz-critic concurs or dissents on the scratch render. **A no-go names the failure and proposes an alternative dataset; it does NOT silently substitute one** -- it routes back to the dispatching session for a re-scope decision. (editorial-critic does NOT run at this gate: per its definition it is post-impl/pre-merge only, its input is a finished notebook. Planning-time arc-vs-data coherence is the orchestrator's job, done in the workstream brief.)
2. **Draft.** notebook-author writes the notebook against the `hiveplotlib-tutorial-notebook` + `viz-quality-bar` skills, NARROW to the tutorial's knob, following the validated dataset/arc. Created only under `examples/`.
3. **Post-impl critics, in parallel: editorial-critic + viz-critic.** Complementary and non-overlapping -- **editorial-critic owns structure** (right notebook / class-scope, dataset coherence, genre fit, section-worth, cross-links), **viz-critic owns figures**. editorial-critic reads prior viz-critic reports as input and cites rather than re-reviews figures. Both are read-only / propose-only; report `Status: clean | propose` with findings tagged `[must-fix | worth-discussing | low-confidence]` at `<file>:<cell>`. **Any scope-crossing or `must-fix` finding routes through orchestrator `amend-plan` (rule 14) for user sign-off; the dispatching session does not edit the plan or the notebook's scope directly.** A finding that would change *what the tutorial teaches*, its *knob scope*, or its *dataset* is by definition scope-crossing -> `must-fix` -> amend-plan, never an in-place tweak.
4. **Docs registration.** Register in `docs/source/notebooks/index.rst` (both blocks, same order) under the new section; add the `conf.py` thumbnail entry + `_static/` asset; update the `CHANGELOG.rst` v0.28.0 (or then-current) WIP entry. Add cross-links *only where both endpoints exist* (graceful; see "Cross-links"). qa-engineer closes with a release-readiness pass (tests, lint, type, `make docs` warning scan, Implementation log + CHANGELOG check).

**Why a gate per tutorial, not once:** dataset risk is per-tutorial (T2's "do the sort metrics diverge legibly" is a different bet than the capstone's "does a plain-data surprising-connections network even exist"). The capstone's gate is the highest-risk in the series (see its entry) and is the reason it builds last.

## Phasing

Phases are independently shippable and non-blocking. The section can go live after Phase 1 and read as complete. **No phase blocks another; no tutorial blocks another.** Sequencing *within* a phase is a recommendation, not a hard dependency, except the calibration checkpoint (a designed gate the user requested).

- **Phase 0 -- section scaffolding (one-time, tiny).** Create the new `index.rst` section (heading + empty/near-empty `nblinkgallery` + `toctree`) and register `finding_a_partition.ipynb` in it once T1 is authored (WS-0). The section can hold a single tutorial. Detailed workstreams: WS-1.
- **Calibration checkpoint (designed gate -- user-requested).** AFTER `finding_a_partition` ships (WS-0) and BEFORE Phase-1 T2 *authoring* (WS-2) begins, this roadmap gets a calibration pass via orchestrator `amend-plan`: fold the learnings from the real, completed T1 notebook (what the build pattern actually cost, which conventions held, which dataset-gate criteria mattered) into the section conventions and the per-tutorial build pattern above. This is an explicit step in the sequencing, not optional. See "Calibration checkpoint" below.
- **Phase 1 -- author T1 + scaffold the section + author T2.** Deliberately light, to avoid an open-ended commitment. Author **T1 `finding_a_partition`** (WS-0: gate, draft, critics, docs), scaffold the section (WS-1), and author **T2 Sorting with Graph Metrics** (working title; WS-2: the gate, draft, critics, docs) -- the most broadly useful next door and the cleanest dataset story. Detailed, cell-aware workstreams: WS-0, WS-1, WS-2. **Phase-1 done-when:** `finding_a_partition` is authored (WS-0) and passes `make test-nb`; the new section exists in `index.rst` and `finding_a_partition` is registered under it (WS-1); the T2 sorting notebook (filename finalized at WS-2 authoring) exists, passes `make test-nb`, is registered with a thumbnail, has cleared editorial-critic + viz-critic post-impl (no open `must-fix`), and `make docs` builds with no new warnings.
- **Phase 2 -- HPM-comparison + Reading the Edges.** Promote **T4 Comparing Lenses with an HPM** and **T3 Reading the Edges** to real workstreams (each via amend-plan, each instantiating the build pattern). Add the forward-links from T1/T2/T3 to T4 now that T4 exists. **Phase-2 done-when (set at promotion):** both notebooks shipped through the full build pattern; the T4<->more-than-three-groups cross-links resolve both directions; no open `must-fix`.
- **Phase 3 -- capstone + expansion candidates.** Promote the **Capstone** (behind its hard validate-then-commit gate; build last) and, if they earn distinct payoffs, the **Core and Periphery** and **Directed: Sources and Sinks** expansions. **Phase-3 done-when (set at promotion):** the capstone ships through the build pattern with its gate passed; any promoted expansion ships likewise; the section reads as a coherent arc end to end.

## Calibration checkpoint (designed gate after T1 ships)

A first-class, user-requested step in the sequencing. Trigger: `finding_a_partition` has shipped (WS-0 closed). Before WS-2 authoring starts, invoke orchestrator `amend-plan` on *this* roadmap to fold T1's real learnings in. Specifically, the calibration pass should:

- Reconcile the **per-tutorial build pattern** above against what T1 actually cost (did the gate catch the right risks? did editorial-critic + viz-critic in parallel work as scoped? did the named-community labeling mechanism land as the default or did the fallback fire?).
- Tighten the **section conventions** (section name confirmed-or-changed by the user; placement in `index.rst` finalized; the cross-link policy validated against how T1's hand-off to `hive_plots_more_than_three_groups.ipynb` actually read).
- Refine **WS-2's dataset-gate criteria** using T1's gate as the worked example (e.g. what "the structure is legible at render size" concretely meant for the Les Mis HPM).
- Record any **convention drift** the T1 build surfaced (e.g. a tutorial-skill idiom that needed bending) so T2 onward inherit the corrected convention.

This is an amend-plan pass that edits this roadmap's "Plan amendments" section and may tune WS-2 in place; it is not a code workstream.

## Cross-links policy (graceful, never blocking)

Cross-links are added **only when both endpoints exist**. A tutorial never blocks on a sibling that has not shipped. Concretely:

- Each tutorial links *forward* to T4 (the comparison lens) at the moment a reader would want small multiples -- but that link is added in the phase where **both** the linking tutorial and T4 exist (Phase 2 adds T1/T2 -> T4 links once T4 ships; T3 ships with its T4 link already valid).
- T4 cross-links `hive_plots_more_than_three_groups.ipynb` (the de-confliction link: "for fitting >3 *known* groups into a hive plot, see ...; this tutorial instead *compares lenses*"). Optionally `hive_plots_more_than_three_groups.ipynb` gains a back-pointer if it reads naturally.
- The capstone links back to T1/T2/T3 (it uses all three knobs) -- valid by Phase 3, when all exist.
- Cross-link the **single best next-step**, not every subordinate reference (tutorial-skill idiom). Format as prose paragraphs, not `## See Also` bullets.

## Workstreams (Phase 1 only)

Detailed, cell-aware workstreams exist for **Phase 1 only**. Everything past Phase 1 is in "Future tutorials (roadmap)". Each Phase-1 workstream instantiates the per-tutorial build pattern.

**Critic coverage (Phase 1).** WS-0 (authoring T1) gets the full build pattern: validate-then-commit gate (viz-critic on the scratch render) -> draft -> **editorial-critic + viz-critic post-impl in parallel** -> docs registration -> qa-engineer. WS-1 (scaffolding/registration) is docs-registration only and gets a qa-engineer verification (no figure, no notebook authored -> no editorial-critic/viz-critic). WS-2 (authoring T2) gets the full build pattern, same shape as WS-0. api-critic is N/A series-wide (docs-only). The phase closes with a qa-engineer release-readiness pass.

### WS-0: Author T1 -- `finding_a_partition.ipynb` (Les Misérables partition-discovery tutorial)

**Status:** not started (soft-gated behind the branch-`46-more-streamlined-networkx-usage-and-support` merge; the validate-then-commit gate sub-step can begin once that branch merges, since the gate renders against the metric surface that branch ships).
**Specialist:** notebook-author (ground in `hiveplotlib-tutorial-notebook` + `viz-quality-bar`). Build pattern: validate-then-commit dataset/story gate (viz-critic on the scratch render) -> draft -> **editorial-critic + viz-critic post-impl in parallel** -> docs registration (docs-engineer for `.rst`/`conf.py`/CHANGELOG if the notebook-author does not do them) -> qa-engineer close.
**Files:** `examples/finding_a_partition.ipynb` (new), then `docs/source/notebooks/index.rst`, `docs/source/conf.py` (thumbnail), `docs/source/_static/finding_a_partition.<ext>`, `CHANGELOG.rst`, and the targeted tutorial back-pointers in `examples/hpm_from_partition.ipynb` / `examples/hpm_from_variable_sweep.ipynb` plus the forward-pointer in `examples/computing_graph_metrics.ipynb`. Do NOT touch `docs/source/gallery_examples/` or `docs/source/notebooks/` notebook copies (auto-generated).

Use the user-confirmed slug `finding_a_partition.ipynb` / `### Finding a Partition with Graph Metrics`; do not re-audit the name or re-justify the Les Mis dataset (both user-confirmed; see "Naming audit" and "Default justifications" below, where T1's full spec now lives). If, while drafting, the API usage sketch needs a helper not on the shipped surface, that is a rule-9 surface-back to orchestrator `amend-plan` for a feasibility check (default expectation: no surface change; this is docs-only).

WS-0 has three sub-steps, instantiating the per-tutorial build pattern: the **dataset/story gate** (validate-then-commit), the **draft**, and the **tutorial-side docs registration**. The full cell-aware spec for each is below (moved here from `graph-metrics-notebook-restructure.md` on 2026-06-17 so this plan is self-contained; that plan no longer holds it).

**Subtype (the draft):** conceptual deep-dive on a real dataset (so: `### H3` title, `#### Background` on Les Mis provenance, `#### References` with the Knuth / Les Mis co-appearance citation, rhetorical question posed early and revisited). Closest siblings to calibrate length and voice: `comparing_network_subgroups.ipynb`, `hive_plots_more_than_three_groups.ipynb`, `karate_club.ipynb`. The tutorial requires the `networkx` and `datashader` extras (the 6-community Les Mis HPM renders poorly in matplotlib; `hive_plots_more_than_three_groups.ipynb` uses datashader for the same reason); surface `pip install hiveplotlib[networkx,datashader]` up front.

**Scope is NARROW — do not drift into a general graph-metrics survey.** This tutorial is the partition-discovery arc only (the spine below). A broader "what you can do with graph metrics as a whole" survey is explicitly OUT of scope and left as a possible *separate future notebook*, not scheduled (those mechanics already live across the cleaned `computing_graph_metrics.ipynb` gallery and the HPM galleries). This is the series-wide NARROW-scope discipline, applied to T1.

**Narrative arc (the spine, do not pad):**

1. **Background + the problem.** Les Mis co-appearance network: 77 characters, edges = co-appearance. Pose the question: a hive plot needs a *partition* (which axis a node goes on) and a *sorting* (where on the axis). This graph gives you *neither*. How do you choose? (One sentence: this is a graph-theory journey, not literary analysis; you do not need to know the novel.)
2. **Metrics give you a sorting, not a partition.** Compute `harmonic_centrality` / `degree`. Show they are per-node scalars: good for sorting *within* an axis, but they do not tell you how many axes or which node on which. You still need a partition. (Sketches 1, 2.)
3. **First path: discretize a metric into a <=3-group partition.** Bin `harmonic_centrality` into periphery/supporting/central, build a 3-axis hive plot. Honest, defensible, ordinal — but an *imposed* cut, not structure the data reveals. (Sketch 3a.) This is the relocated motivated "bin a metric" demo (the gallery shows the *mechanic* on Karate `degree`; T1 shows *why you'd reach for it* on real data where no partition exists).
4. **Better path: detect communities to find data-driven groups.** Run Louvain. It finds **more than 3** communities (honest framing; do *not* tune resolution to force 3 — this is the series-wide anti-dishonest-pedagogy rule). (Sketch 3b.)
5. **>3 communities motivate a `HivePlotMatrix`.** Build the 6x6 HPM with communities named by most-central member, datashader backend. This is the payoff figure. (Sketch 4.)
6. **Hand off.** A hive plot matrix is one of several ways to handle >3 groups; for the full menu, see `hive_plots_more_than_three_groups.ipynb`. Cross-link, do **not** re-tread its four options. Revisit the opening question (we found a partition two ways; community detection found the data-driven one; >3 groups routed us to the matrix).
7. **References.**

**Community-labeling mechanism (default + fallback).** Naming communities by most-central character stands. **Default:** the named-column approach (Sketch 4) — derive a named string partition column, which yields character-named HPM row/column labels for free and sidesteps the deferred int->str label coercion. **Fallback:** relabel within the `HivePlotMatrix` itself, explored *only if* a clean label-override path exists on the matrix (the known coercion suggests labels derive from partition values, so the named-column path is likely the only robust route). The dataset gate validates whichever mechanism is chosen.

**Done when:**

*Sub-step 1 — dataset/story validate-then-commit gate (no repo artifact; renders to `/tmp/` per `mental-model` rule 16):*

1. A fresh render of the Les Mis Louvain-community HPM (datashader backend, `from_partition`, communities named by most-central member) is produced and saved under `/tmp/` (e.g. `/tmp/finding_a_partition_validation/lesmis_communities_hpm.png`). The 3-axis harmonic-centrality-bin `HivePlot` (Sketch 3a) is also rendered for the "discretize" leg.
2. A go/no-go judgment is recorded in the dispatch report (not the repo): does the 6-community structure read legibly as an HPM at notebook render size, do the named-community labels produce a coherent story (each community has a recognizable central character), and does the "metrics give you values -> still need axes -> discretize or detect communities -> >3 groups -> HPM" arc hold on this data? viz-critic concurs or dissents on the scratch render.
3. If **no-go**: the report names the failure (communities not legible, labels confusing, story does not cohere) and proposes an alternative partition-less network (candidates: Florentine families [small, may be too small], a stochastic-block-model synthetic with a *hidden* partition, another `networkx` built-in co-occurrence graph). A no-go routes back to the dispatching session for a re-scope decision before drafting; it does **not** silently substitute a dataset. (Trade data is explicitly **out**: it is `hive_plots_more_than_three_groups.ipynb`'s dataset and has a natural `continent` partition, so it cannot motivate discovery.)
4. The validation confirms the `networkx` + `datashader` extras are what the tutorial needs (matplotlib at 6 communities was the prior viz-critic concern; confirm datashader is the right call, matching `hive_plots_more_than_three_groups.ipynb`).
5. The community-label derivation is validated as runnable on shipped surface: confirm `from_partition` accepts the named string-column partition and the 6x6 matrix builds without error (the int-partition `KeyError` is fixed; a *string* partition column is the tutorial's path, which also sidesteps the int->str label coercion). The default is the named-column approach; the in-matrix relabel fallback is explored only if a clean label-override path exists on the matrix.

*Sub-step 2 — the draft (`examples/finding_a_partition.ipynb`, new):*

6. `examples/finding_a_partition.ipynb` exists, follows the tutorial skill (H3 title `### Finding a Partition with Graph Metrics`, H4 subsections, `#### Background`, `#### References`, install-extras up front for `networkx,datashader`, prose-to-code balanced).
7. The arc above is realized; the honest "Louvain finds >3 communities -> switch primitive, do not tune resolution" framing is preserved.
8. Communities are displayed by most-central-character names, not integer labels (sidesteps the deferred coercion; avoids the numbered-label anti-pattern).
9. The >3-groups hand-off cross-links `hive_plots_more_than_three_groups.ipynb` and does not duplicate its content. The discretize/detect contrast (ordinal bin vs. categorical community) is realized without copying the gallery's removed cells.
10. The two relocated "orphan" demos are absorbed: community-detection-as-partition (here, on Les Mis, **not** Karate) and the motivated "bin a metric into a partition" demo (here, the harmonic-centrality 3-axis leg).
11. `make test-nb` runs the new notebook end-to-end and passes (warnings-as-errors); datashader cells render.
12. Voice-rule scan clean (no em-dashes, no AI filler; tutorial length within ~2x its closest sibling per the skill's length discipline).
13. Figure-quality: datashader HPM follows `viz-quality-bar` datashader specifics (accept `cmap_nodes`/`cmap_edges` defaults, `unify_axes=True` for cross-cell comparison, pin rasterization params if cross-plot comparison is implied); the 3-axis hive plot follows hive-plot rules (repeat-axes two-tone if repeat axes used, title `y` tuned, alpha for density).
14. Created only at `examples/finding_a_partition.ipynb` for the draft sub-step (docs registration is sub-step 3 / WS-1; do not edit `docs/` here).
15. editorial-critic + viz-critic post-impl in parallel both clear (editorial-critic: genre fit as a motivated tutorial, the arc coheres with the lead-in matching the body, **section-worth under the NARROW-scope discipline** — no survey section the cleaned gallery or `hpm_from_variable_sweep.ipynb` covers better, the relocated demos sit naturally, the hand-off aims at the best next step; viz-critic: `good` or `worth-discussing`-only on the figures, datashader HPM especially). Any `must-fix` (editorial or viz) routes through orchestrator `amend-plan` (rule 14) before WS-0 closes.

*Sub-step 3 — tutorial-side docs registration (was the tutorial half of the restructure plan's WS-D; pairs with WS-1's section scaffolding):*

16. The notebook is registered in `docs/source/notebooks/index.rst` under the `Using Graph Metrics with Hive Plots` section (created in WS-1; **both** the `nblinkgallery` block and the hidden `toctree`, same position in each). Not the gallery index (it is a tutorial, not a feature reference).
17. A `nbsphinx_thumbnails` entry is added to `docs/source/conf.py` (`"notebooks/finding_a_partition": "_static/finding_a_partition.jpg"` or matching extension), and the asset is produced under `docs/source/_static/` (per `viz-quality-bar` "Thumbnails" — the most representative figure, likely the named-community HPM; strip text). If producing the asset is deferred, the entry must still resolve or `make docs` warns.
18. The `CHANGELOG.rst` v0.28.0 WIP Documentation block (currently the "three new gallery notebooks" framing) is updated to include the tutorial and to no longer mis-describe the set as only gallery notebooks (e.g. a "New tutorial: Finding a Partition with Graph Metrics ..." bullet). No em-dashes, no AI filler, length discipline.
19. The two targeted tutorial back-pointers are added: `examples/hpm_from_partition.ipynb` (cell 47) and `examples/hpm_from_variable_sweep.ipynb` (cell 50) each gain a second sentence pointing at the tutorial (their existing "Computing Graph Metrics" pointer stays; both currently close their "Computing Graph Metrics During Construction" section with the identical "...or discretizing node graph metrics to use as partition variables, see Computing Graph Metrics" sentence). Optionally `hive_plots_more_than_three_groups.ipynb` gains a "this technique was motivated in the Finding a Partition tutorial" back-pointer if it reads naturally.
20. The gallery **forward-pointer** into `examples/computing_graph_metrics.ipynb` is added: a closing prose paragraph framed as "for a narrative walkthrough of using metrics and community detection to find a partition when the data gives you none, see ...". This is the deferred WS-A done-when #7 pointer in the restructure plan; it lands here (the gallery is narrowed by that plan's WS-A / MR1, and this additive forward-pointer is clean against it once MR1 merges). It does not require editing the gallery's narrowed content, only appending the pointer.
21. A grep confirms the new tutorial slug `finding_a_partition` appears in `notebooks/index.rst` (both blocks), `conf.py` (thumbnail), `CHANGELOG.rst`, and the back-pointer notebooks, and nowhere stale.

*Close:*

22. qa-engineer release-readiness pass: `make test-nb` green for all touched notebooks, `make docs` no new warnings (use `make docs`, not `make docs-strict`; scan the full warning set), lint/type green, Implementation log + CHANGELOG updated.

### WS-1: Scaffold the section + register `finding_a_partition`

**Status:** not started (the section-heading sub-step can run anytime; the registration sub-step pairs with WS-0, which authors the notebook being registered).
**Specialist:** docs-engineer. Post-impl: qa-engineer (grep audit + `make docs` warning scan). No editorial-critic/viz-critic (no notebook authored, no figure produced).
**Files:** `docs/source/notebooks/index.rst` (new section), and `finding_a_partition`'s registration entries (authored here in WS-0; registered here). Do NOT touch `docs/source/gallery_examples/` or `docs/source/notebooks/` notebook copies (auto-generated).

**Done when:**

1. A new section titled `Using Graph Metrics with Hive Plots` (the user-confirmed section name) exists in `docs/source/notebooks/index.rst` with both an `nblinkgallery` block and a hidden `toctree`, placed after the `Hive Plots` tutorials section (it assumes the reader can build a hive plot). The section may contain a single entry.
2. `finding_a_partition` (authored in WS-0) is registered in the new section (both blocks, same order). Its `conf.py` thumbnail entry and `_static/` asset are produced in WS-0; verify they resolve. (WS-0 and WS-1 may be one dispatch; if WS-0 does the registration itself, WS-1 reduces to creating the section heading and confirming T1 sits in it exactly once.)
3. The section heading reads coherently against its siblings (`Hive Plots`, `Hive Plot Examples`); a reader scanning the index can tell this section is the interpretive one.
4. `make docs` builds with the new section, no new warnings (use `make docs`, not `make docs-strict`; scan the full warning set per the user's recorded preference).
5. A grep confirms `finding_a_partition` appears exactly once in each of the new section's two blocks and nowhere stale.
6. Edited only in `docs/source/notebooks/index.rst` (plus whatever T1's track left for placement). No notebook prose edited here.

### WS-2: Author T2 -- the sorting tutorial (working title "Sorting with Graph Metrics"; filename finalized at authoring)

**Status:** not started (authoring soft-gated behind the calibration checkpoint; the validate-then-commit gate sub-step can begin once the dataset candidate is chosen). **The notebook filename is PROVISIONAL** and is finalized at this workstream's authoring step (open choice between `sorting_with_graph_metrics.ipynb` -- concept-phrase, dispatching session's mild lean -- and `graph_metrics_sorting.ipynb` -- topic-prefix, user's working assumption; see the naming audit's T2 entry). The reader-facing `### H3` title trends toward "Sorting with Graph Metrics" (parallels T1's `### Finding a Partition with Graph Metrics`) but is finalized/polished at authoring.
**Specialist:** notebook-author (ground in `hiveplotlib-tutorial-notebook` + `viz-quality-bar`). Build pattern: validate-then-commit gate (viz-critic on scratch render) -> draft -> **editorial-critic + viz-critic post-impl in parallel** -> docs registration (this can fold into a single WS-2 close or split; docs-engineer for `.rst`/`conf.py`/CHANGELOG if the notebook-author does not do them). qa-engineer closes.
**Files:** `examples/<sorting-tutorial>.ipynb` (new; filename finalized at authoring per the naming audit -- `sorting_with_graph_metrics.ipynb` or `graph_metrics_sorting.ipynb`), then `docs/source/notebooks/index.rst`, `docs/source/conf.py`, `docs/source/_static/<slug>.<ext>`, `CHANGELOG.rst`, and any graceful forward-link.

**Knob:** sorting (where on the axis a node sits).
**gnn-heterogeneity mapping:** continuous axis sorting captures gradients that binned bar charts flatten (the T2 row of the mapping table), plain-data-translated (no confidence-as-sort).

**Scope is NARROW to the sorting knob.** Do not survey all metrics, do not re-teach the `sorting_variables` parameter mechanics (`setting_sorting_variables.ipynb` owns that), do not drift into partition or edge-styling (those are T1/T3). The tutorial is "sort one real graph four ways and read what each surfaces, then show the assortativity payoff."

**Narrative arc (the spine; refine at the gate, do not pad):**

1. **Background + the question.** A real graph (Les Mis leaning, per the dataset note; confirmed at the gate). The partition is fixed/given for this tutorial; the question is *where on each axis does a node go, and what does the choice reveal?* Pose: different sortings surface different structure -- which metric answers which question?
2. **Sort by degree.** Hubs at the tips. Read the hub structure.
3. **Sort by betweenness.** Bridge/broker nodes rise; contrast with degree (a high-degree node need not be a bridge -- the karate-walkthrough "bridges independent of popularity" reading is the conceptual model).
4. **Sort by pagerank and/or core-number.** Each surfaces a different ordering; one or two more, not all -- NARROW discipline.
5. **The payoff: assortativity as edge geometry.** Sort *both* axes by degree. Top-to-top edges = hubs talk to hubs (assortative); top-to-bottom = hubs to periphery (disassortative). The sorting choice makes a mixing pattern *visible as geometry*. This is the signature figure.
6. **Reflect + hand off.** Revisit the opening question. Forward-link to T4 (added in the phase where T4 exists) at the "you could compare these sortings side by side" moment. `#### References` (dataset citation + gnn-heterogeneity as conceptual parent).

**Done when:**

1. The sorting tutorial notebook exists under `examples/` (filename finalized at authoring per the naming audit), follows the tutorial skill (H3 title -- trending toward `### Sorting with Graph Metrics`, finalized at authoring; H4 subsections, `#### Background`, `#### References`, install-extras up front for whatever the chosen dataset needs, prose-to-code balanced), NARROW to the sorting knob.
2. The validate-then-commit gate passed before drafting: the chosen dataset's sort-by-{degree, betweenness, pagerank, core-number} each produce a *visibly different* node ordering, and the both-axes-sorted-by-degree assortativity figure *reads* at notebook size. Gate go/no-go recorded in the dispatch report; viz-critic concurred. (A no-go re-scoped the dataset before any drafting.)
3. The assortativity-as-geometry payoff figure is realized and is the tutorial's showcase (full polish per `viz-quality-bar`: two-tone or sequential as appropriate, title `y` tuned, `flexitext` if it earns it).
4. PLAIN-DATA discipline held: no confidence-as-sort, no ML framing; gnn-heterogeneity cited as conceptual parent only. No Cora/CiteSeer/PubMed/OGB-arxiv.
5. Section-worth held: no section the `setting_sorting_variables.ipynb` gallery or `computing_graph_metrics.ipynb` covers better; the tutorial does not re-teach the parameter mechanic.
6. `make test-nb` runs the new notebook end-to-end and passes (warnings-as-errors).
7. Voice-rule scan clean (no em-dashes, no AI filler; length within ~2x the closest sibling -- `comparing_network_subgroups.ipynb` / `karate_club.ipynb`).
8. Created only under `examples/` at the authoring-finalized filename (`sorting_with_graph_metrics.ipynb` or `graph_metrics_sorting.ipynb`). Docs registration is the WS-2 docs sub-step (do not edit `docs/source/notebooks/` copies or `gallery_examples/`).
9. Registered in `docs/source/notebooks/index.rst` under the new section (both blocks, same order); `conf.py` thumbnail entry + `_static/` asset produced; `CHANGELOG.rst` v0.28.0 (or then-current) WIP entry updated. No em-dashes, length discipline on the CHANGELOG bullet.
10. editorial-critic post-impl returns `Status: clean`, or `propose` with no `must-fix`: confirms genre fit (motivated tutorial, not a gallery page), arc coheres with the lead-in matching the body, section-worth under the NARROW-scope discipline (no survey section, no re-teaching the sorting-parameter mechanic), dataset coherence (the chosen graph motivates the sorting payoff), and the cross-link (if T4 exists) aims at the best next step. viz-critic post-impl returns `good`, or `worth-discussing`-only, on the figures (assortativity showcase especially). Any `must-fix` (editorial or viz) routes through orchestrator `amend-plan` (rule 14) before WS-2 closes.
11. qa-engineer release-readiness pass: tests/lint/type green, `make docs` no new warnings, Implementation log + CHANGELOG updated.

## Future tutorials (roadmap)

NOT committed workstreams. Each entry carries theme, knob, the capability it unlocks, candidate dataset(s) with a risk note, the gnn-heterogeneity mapping, and its phase. Promote to a real workstream via orchestrator `amend-plan` when its phase comes, at which point it instantiates the per-tutorial build pattern and earns a full naming audit + dataset gate.

### T3 -- Reading the Edges (Phase 2; the edge knob)

- **Theme.** Color and weight edges by an edge metric to read bridges/bottlenecks (edge-betweenness) and expected-vs-surprising connections (link prediction).
- **Capability unlocked.** Bridges the gap no shipped page covers: edge-*metric* -> edge-*color interpretation* (`edge-rendering.md` is styling mechanics only). Uses the kwarg hierarchy from `edge-rendering.md` and the link-prediction-default-on-existing-edges wrinkle (see "API surface (none)") as the "expected vs. surprising" hook.
- **Candidate datasets + risk.** Les Mis or Karate (low risk; both have legible bridge edges; edge-betweenness on Karate cleanly flags the cross-faction broker edges, matching the karate-walkthrough). **Risk: low.** The link-prediction leg needs a graph where some existing edges are structurally "surprising" (low Jaccard); validate at the gate that the chosen graph has both expected and surprising existing edges so the color scale has a story.
- **gnn-heterogeneity mapping.** Edge-color-by-metric; "errors concentrate on specific edge types" -> a structural edge metric replaces correct/misclassified.
- **Section-worth.** Distinct from `visualizing_edge_metadata.ipynb` (gallery: attach a column) and `edge_kwarg_hierarchy.ipynb` (the precedence reference) and `computing_graph_metrics.ipynb`'s link-prediction *mechanic* section. T3 is the *interpretive* arc.
- **De-confliction note.** Ships with its T4 forward-link valid (T4 lands same phase).
- **Phrasing caveat (captured 2026-05-29, from the WS-A gallery trim).** When T3 makes the inter-vs-intra Jaccard reading, phrase it as a *tendency*, not an absolute. On the Karate render the cross-club (inter-axis) edges run lower on average than the within-club edges, but at least one within-club edge also carries a low Jaccard coefficient, so "low Jaccard = cross-club edge" is false. Before asserting the reading, verify the inter/intra Jaccard split as a data check (group `Edges.data` by inter vs. intra and compare distributions) and have viz-critic confirm the figure communicates it. This is the exact interpretive sentence cut from `computing_graph_metrics.ipynb`'s "Link Prediction Scores as an Edge Metric" section (out-of-scope for the gallery, and inaccurate as phrased); the gallery now keeps only the mechanic.

### T4 -- Comparing Lenses with a Hive Plot Matrix (Phase 2; the meta-tool)

- **Theme.** `from_variable_sweep` over sorting metrics and over partition variables to discover which lens reveals structure. The HPM as a *comparison lens*, the plain-data realization of Krzywinski's 5x5 Hive Panel as a metric-sweep template (`hive-plot-matrix.md`).
- **Capability unlocked.** Ties T1/T2/T3 together: when you do not know which partition or sorting reveals structure, sweep candidates and compare. Each spine tutorial forward-links here at the "you'd want small multiples" moment.
- **Candidate datasets + risk.** Les Mis (sweep sortings: degree/betweenness/pagerank/core-number, reusing T2's metrics) or trade (sweep partition variables). **Risk: low-medium.** The sweep must produce cells that *differ legibly* so the comparison has a point; validate at the gate. Reuse a Phase-1 dataset if it makes the cross-tutorial story cohere (welcome bonus, not required).
- **gnn-heterogeneity mapping.** Sweep-to-discover-most-informative-lens; no GCN/GAT/GraphSAGE model row.
- **De-confliction with `hive_plots_more_than_three_groups.ipynb` (MUST de-conflict; inspected).** That notebook is **HPM-as-the-fourth-of-four-overflow-options** for a network that *naturally* partitions into >3 groups (trade, `continent`, datashader; its options are more-axes / two-layers / collapse-to-3 / HPM small multiples). **T4 is HPM-as-comparison-lens**: the question is "which sorting/partition lens reveals structure," not "how do I fit 6 known groups into a hive plot." Different question. **Cross-link, do not duplicate**: T4 links to more-than-three-groups for "fitting >3 *known* groups"; more-than-three-groups optionally back-links T4 for "comparing *lenses*." Do not re-tread its four options in T4; do not re-tread T4's sweep arc there.

### Capstone -- Finding Surprising Connections / Structural Heterogeneity (Phase 3; the finale)

- **Theme.** All three knobs (partition + sorting + edge styling) plus an HPM together to surface unusual structural heterogeneity on a plain real network. The "why you did all of this" payoff and the plain-data realization of the gnn-heterogeneity thesis. Storyboard: `structural-heterogeneity.md`'s structural-property -> hive-plot-role table, **with the GNN-performance/relevance column dropped**.
- **Capability unlocked.** Demonstrates the whole section's method end to end on one dataset; the section's thesis statement.
- **Candidate datasets + risk -- HIGHEST RISK IN THE SERIES (the confirmed top risk).** No off-the-shelf plain-data "surprising connections / heterogeneity" network is known to exist. Candidates, all **unvalidated**: Bitcoin OTC (directed/signed/temporal; "anomalous trust" angle -- but signed/temporal adds framing complexity, and `bitcoin_user_ratings.ipynb` already uses it for a temporal HPM story, so de-confliction is needed); trade data (surprising cross-continent links -- but it is `hive_plots_more_than_three_groups.ipynb`'s dataset). **Treat as an explicit, hard validate-then-commit gate, built LAST.** The gate must confirm a coherent plain-data "surprising connections" story exists on the chosen data *before* any drafting; a no-go re-scopes the dataset (or, in the worst case, surfaces back that the capstone needs a dataset that does not yet ship, which is a user decision, not a silent substitution).
- **gnn-heterogeneity mapping.** The heterogeneity thesis itself, plain-data-translated: no fairness framing, no training-set-distance, no confidence; a real attribute/community replaces predicted label, a real metric replaces confidence.
- **PLAIN-DATA emphasis.** This is the tutorial most at risk of drifting back toward the GNN framing (it is the thesis tutorial). Hold the line hard: cite the gnn page as the conceptual parent and explicitly tell the story on plain data.

### Core and Periphery (later phase; expansion candidate)

- **Theme.** k-core / `core_number` / `onion_layers` + `clustering` to read a graph's core-periphery structure.
- **Knob.** Sorting (core-number/onion-layer as the axis position) and/or partition (core vs. periphery bands).
- **Candidate datasets + risk.** Unassigned. **Risk: THINNEST-GROUNDED in the series** -- no dedicated wiki page and no associated dataset yet. **Only promote if it earns a payoff distinct from T2's core-number sort** (T2 already sorts by core-number as one of its four metrics); if the only content is "sort by core-number," it folds into T2 rather than becoming its own tutorial. Promotion requires both a dataset gate and a section-worth re-check against T2.
- **gnn-heterogeneity mapping.** Core-periphery is a dimension of structural heterogeneity (`structural-heterogeneity.md` community-density variation), ML-stripped.

### Directed Networks: Sources and Sinks (later phase; expansion candidate)

- **Theme.** in/out-degree on a directed real network; partition into Krzywinski's source / manager / sink scheme (`node-assignment.md` is the grounding -- the *only* page that grounds directed assignment).
- **Knob.** Partition (source/manager/sink by in/out-degree) -- the only spine/expansion that *needs* a directed graph; `in_degree`/`out_degree`/`reciprocity`/`topological_generations` all ship with directed guards.
- **Candidate datasets + risk.** trade data (directed export flows + continents -- but verify it ships *as directed*; the existing `hive_plots_more_than_three_groups.ipynb` uses it with an undirected `continent` partition, so whether the shipped subset carries edge *direction* is a gate question); Bitcoin OTC (directed/signed/temporal); RegulonDB + Flare (textbook directed source/sink cases, but they are *paper figures, not shipped datasets* -- sourcing risk). **Risk: medium** (directed-dataset sourcing + the trade-direction verification).
- **gnn-heterogeneity mapping.** Directed structure is outside the gnn page's node-classification framing; this expansion is grounded by `node-assignment.md` directly, not the gnn page.

## Plan amendments

Append-only. Each entry summarizes a delta and tags it Added workstream / In-scope tweak / Deferred follow-up. **The calibration checkpoint (after T1 ships) is the first expected amendment**; Phase-2 and Phase-3 promotions are subsequent amendments.

### 2026-05-29 -- Section name locked; T2 name guidance (both naming; In-scope tweaks)

User-decided naming updates. Both are pure naming; no scope, workstream, dataset, or notebook-teaching change. Triaged as two **In-scope tweaks** (each is a single-axis name change against surfaces this plan already owns; neither alters what a tutorial teaches, its knob scope, its dataset, or its class scope).

1. **Section name locked: `Using Graph Metrics with Hive Plots`** (replaces the prior recommendation `Interpreting Hive Plots with Graph Metrics`). Now user-confirmed; the section name is no longer a flag-for-confirmation item. Rationale recorded in the Naming audit: "Interpreting" undersells the section, which covers using metrics to *construct* hive plots (metrics as sorting and partition variables), not only to color/interpret them; and "Using Graph Metrics with Hive Plots" pairs cleanly with the `computing_graph_metrics.ipynb` gallery on a compute-vs-use boundary (gallery = compute the metrics = mechanics; this section = use them = application). Swept: Naming audit section-name block (recommended -> user-confirmed, candidate table reordered, prior recommendation marked superseded), WS-1 done-when #1. (Goal prose describes the section idea but never quoted the old heading string, so it needed no edit; its framing already matches the new name.)

2. **T2 naming made provisional, finalized at authoring.** Filenames are pragmatic working names; reader-facing titles get polished and are freely renamable. Recorded working title **"Sorting with Graph Metrics"** (parallels the likely T1 final title "Finding a Partition with Graph Metrics" -- both end in "with Graph Metrics", both sit under the section). The **filename is NOT locked**; it is decided at WS-2 authoring. Open choice recorded: `sorting_with_graph_metrics.ipynb` (concept-phrase, matches T1 and the tutorial-corpus norm; the dispatching session's mild lean, since the `hpm_*` topic-prefix style is reserved for the gallery families) vs. `graph_metrics_sorting.ipynb` (topic-prefix; the user's working assumption). The prior recommendation `sorting_to_reveal_structure.ipynb` is withdrawn. Swept: Naming audit T2 entry, section-worth map row, Phasing Phase-1 bullet, WS-2 heading + Status + Files + done-when #1 + done-when #8.

3. **Section title-trend note added** to the Naming audit: the section's tutorial titles trend toward the "X with Graph Metrics" parallel (T1 "Finding a Partition with Graph Metrics", T2 "Sorting with Graph Metrics"); final titles are settled per-tutorial at authoring.

No dispatch implied by this amendment (naming only). Next dispatch is unchanged: WS-1 (scaffold + migrate, soft-blocked on T1 shipping) and the calibration checkpoint before WS-2 authoring.

### 2026-05-29 -- T3 phrasing caveat captured (In-scope tweak)

Recorded a correctness/phrasing caveat in the T3 "Reading the Edges" roadmap entry (a placeholder refinement; no scope, dataset, workstream, or teaching change). Source: while trimming the `computing_graph_metrics.ipynb` gallery's "Link Prediction Scores as an Edge Metric" section back to mechanics (the interpretive figure-reading belongs to T3), the cut sentence was also found inaccurate. It claimed low-Jaccard edges are the cross-club (inter-axis) ones, but at least one within-club edge carries a low coefficient on the Karate render, so the clean inter=low / intra=high dichotomy is false. T3 must phrase the inter-vs-intra reading as a tendency and verify the split (data check on `Edges.data` plus viz-critic on the render) before asserting it. No dispatch implied.

### 2026-06-17 -- T1 (`finding_a_partition`) authoring re-homed into this plan (Added workstream WS-0)

**Added workstream.** User decision (post-v0.28-review): `finding_a_partition.ipynb` (the Les Misérables partition-discovery tutorial) is **this plan's first authored tutorial**, not a notebook built elsewhere and merely placed here. It was previously tracked as WS-C (draft) + WS-D (tutorial wiring) of `graph-metrics-notebook-restructure.md`, with this roadmap only *migrating it in* at WS-1. That split is reversed: authoring ownership moves here; the restructure plan retains only its gallery-cleanup half (its WS-A). A matching amendment in `graph-metrics-notebook-restructure.md` (2026-06-17) records the scope removal on that side. This is **not** an in-scope tweak: it changes the authored-tutorial workstream set in both plans, so it is tagged as an Added workstream here and a scope removal (Deferred-out) there.

**Delta:**

- **New WS-0 added to "Workstreams (Phase 1 only)": author `finding_a_partition.ipynb`.** It instantiates the per-tutorial build pattern in full (validate-then-commit dataset/story gate with viz-critic on the scratch render -> notebook-author draft -> editorial-critic + viz-critic post-impl in parallel -> docs registration -> qa-engineer close). The complete cell-aware spine, dataset justification (Les Mis = the canonical no-natural-partition network), narrative arc (metrics give a sorting not a partition -> discretize a metric into a <=3-group partition -> detect communities -> >3 communities motivate a `HivePlotMatrix` -> hand off to `hive_plots_more_than_three_groups.ipynb`), NARROW-scope discipline, community-labeling mechanism (named-column default, in-matrix relabel fallback), the user-confirmed filename/title, and all done-whens are **already fully specified in `graph-metrics-notebook-restructure.md`'s WS-B (gate) + WS-C (draft) + the tutorial half of WS-D (registration)**. WS-0 here **adopts those specs by reference** rather than restating them (rule 17). It does not re-audit the filename/title (`finding_a_partition.ipynb` / `### Finding a Partition with Graph Metrics`, user-confirmed) and does not re-justify the Les Mis dataset.
- **Series-wide constraints + the per-tutorial build pattern govern WS-0 as written** (they were authored from the T1 plan in the first place, so WS-0 is their originating instance).
- **Sequencing within Phase 1:** WS-0 (author T1) -> WS-1 (scaffold the section + register T1 in it) -> calibration checkpoint -> WS-2 (author T2). WS-0 supersedes WS-1's "migration" framing: WS-1 no longer places a notebook authored elsewhere; it registers a notebook authored here in WS-0. (WS-1's docs-registration done-whens are otherwise unchanged.) WS-0's own docs registration and WS-1's section scaffolding can be one dispatch or two; if T1's registration lands inside WS-0, WS-1 is reduced to creating the section heading and confirming T1 sits in it.

**Rationale:** the restructure plan's two-MR split (its 2026-05-29 delivery-split amendment) already carved the tutorial (MR2) cleanly off the gallery cleanup (MR1); MR2 was always going to branch off master after the gallery MR merged. Re-homing the authored tutorial onto the series plan that owns the `Using Graph Metrics with Hive Plots` section it lives in is the natural consolidation: the tutorial and its section now have one owner, and the restructure plan narrows to exactly the gallery cleanup it is named for.

**Done-whens touched:**
- *This plan:* Phase-1 done-when gains "`finding_a_partition` authored (WS-0) and registered (WS-1)" ahead of the T2 clause. WS-1's done-whens #2 ("registered ... moved here if T1's own track registered it elsewhere") are rephrased from migration to registration-of-a-locally-authored-notebook (no longer coordinating with another plan's WS-D, since that work is now WS-0/WS-1 here).
- *`graph-metrics-notebook-restructure.md`:* its WS-C and the tutorial half of WS-D are removed from scope (recorded in that plan's 2026-06-17 amendment). WS-B (the dataset gate) moves with the authoring to WS-0 here.

**Body sweep (this plan):** rephrased the "built on its own track / NOT re-authored here / migrates in" language at the Prior-ADRs bullet (line ~32), the Migration bullet under "Patterns this replaces" (line ~61), the Phase-0 bullet, the Phase-1 bullet, the calibration-checkpoint trigger, and WS-1 (Status / Files / done-when #2) to "authored here as WS-0, then registered." The `graph-metrics-notebook-restructure.md` reference at line ~26 stays (it remains the binding source of T1's conventions and full cell-aware spec, which WS-0 adopts by reference); only the "built there, not here" ownership claims flip.

**Not a v0.28 blocker.** T1 ships post-release on its own branch off master after branch `46-more-streamlined-networkx-usage-and-support` merges. The gate is the **branch-46 merge** (the metric surface T1 exercises lives there), not the v0.28 release itself. No release dependency is presumed.

### 2026-06-17 (later) -- T1's full spec moved IN; this plan is now self-contained (supersedes the by-reference choice)

**In-scope tweak** (refines the WS-0 added by the earlier 2026-06-17 amendment; no new workstream, no specialist set change, no done-when removed, sequencing unchanged). User decision, superseding the earlier-same-day by-reference arrangement: make this series plan **self-contained** (it holds T1's full spec, not a pointer into the restructure plan) and the restructure plan **cleanly closeable on its WS-A alone**. Triggered by a user ask (rule 14: a decision changing where a load-bearing spec lives), not a critic finding.

**Delta to this plan:**

- **WS-0's three sub-steps are now spelled out in full here** (dataset/story validate-then-commit gate, draft, tutorial-side docs registration), replacing the prior "Spec adopted by reference" block that pointed at the restructure plan's WS-B/WS-C/WS-D-tutorial-half. The narrative spine, NARROW-scope discipline, honest ">3 communities -> HivePlotMatrix, do not tune resolution", subtype + extras, and community-labeling default/fallback are inlined into WS-0. Done-whens are renumbered 1-22 across the three sub-steps + close (was 1-4, the first two of which deferred to the restructure plan "by reference").
- **The tutorial-specific spec sections moved in** to this plan's body: T1's dataset justification (Les Mis + named-by-character + graph-theory-not-literary-analysis) folded into "Default justifications"; T1's full naming audit (filename/title reasoning + rejected candidates + community-label display names) folded into "Naming audit"; the five **API usage sketches** added as a new "T1 API usage sketches (WS-0)" subsection under "API surface (none)" (notebook content, not surface; still no API change).
- **By-reference language flipped to self-contained** at the Prior-ADRs T1 bullet (line ~32), the Migration bullet (line ~61), and inside WS-0. The restructure plan is still cited as T1's *origin* and the holder of WS-A's gallery cleanup, but is no longer the holder of T1's spec.

**Rationale:** the by-reference arrangement (earlier 2026-06-17 amendment) kept one canonical spec copy to avoid drift, but left the restructure plan un-closeable on WS-A alone (a foreign tutorial spec lingered inside it) and made this plan non-self-contained. Moving (not copying) the spec resolves both: one owner, no drift (the restructure plan deletes what moved), this plan is self-contained, the restructure plan closes on WS-A.

**Done-whens touched:** WS-0's done-whens are restructured from 4 (two by-reference) to 22 self-contained ones across gate/draft/registration/close; no done-when is *removed* in substance (the WS-B and WS-C done-whens are now WS-0 sub-step 1 and sub-step 2 directly). Phase-1 done-when is unaffected (it already reads "`finding_a_partition` authored (WS-0) and registered (WS-1)").

**Cross-plan note:** the matching 2026-06-17 (later) amendment in `graph-metrics-notebook-restructure.md` records the deletion of the moved spec there and confirms that plan is now closeable on WS-A alone.

## Implementation log

Append-only. After each workstream completes, one line in the same turn.

_None yet._
