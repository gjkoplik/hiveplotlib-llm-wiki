# Plan: Same Stats, Different Graphs (matched-statistics network demo)

<!--
Hiveplotlib consumer plan. Formal library support for the Datasaurus-style
"identical graph statistics, different structure" demo prototyped 2026-06-10
(scratch work in data/datasaurus_prototypes/, gitignored; its README.md is the
mechanism reference). Prototype renders and stats reviewed by Gary (grill,
2026-06-11); story standardization pending in hiveplotlib-datasaurus (WS-0).
-->

## Goal

Ship the "Same Stats, Different Graphs" demo as a first-class part of the library: a synthetic dataset generator in `hiveplotlib.datasets` producing five-plus-one networks with pixel-identical degree statistics (same n, m, density, full degree sequence) but instantly distinguishable hive plot signatures, and a tutorial notebook telling the four-beat ladder (stats table foil, standard layouts fail, degree-ordered circle gets halfway, hive plots complete it) plus the hidden-communities twist that motivates partition sweeps. (Ladder set by Amendment 2; the story and figures standardize in a separate story repo before any hiveplotlib code lands, per Amendment 3.) This becomes a centerpiece of the "why choose hive plots" pitch. The variant set (count and membership) is flexible; every shipped variant carries a real-world anchor (Amendment 4). **JOSS dependency:** this work is the "Datasaurus figure" gate (its G2) in `wiki/wiki/plans/joss-submission.md`; the JOSS submission blocks on this plan completing, and that plan now cross-links here (Amendment 4). Prior art: Chen, Soni, Lu, Maciejewski, Kobourov, "Same Stats, Different Graphs" (GD 2018, arXiv:1808.09913), which makes the statistical point with force-directed drawings; we carry it through to hive plots.

## Alignment (grill)

```
Complete (2026-06-11) — three waves below, run as the plan-review mechanism in
place of reading the full plan. Resulting changes bundled as Amendment 4.
```

### Maintainer shared-understanding pass (grill), Wave 1 — JOSS dependency, story repo nature, scope ambition, G2 timing (2026-06-11)

- **JOSS dependency confirmed.** This work is the "Datasaurus figure" gate in the JOSS submission plan (`wiki/wiki/plans/joss-submission.md` per memory). Gary: mention in the JOSS plan as well that this needs to get done first. → Amendment needed: cross-link both plans (→ Amendment 4).
- **Story repo identified and characterized.** Gary already created `hiveplotlib-datasaurus` (adjacent repo, not inside hiveplotlib). It is a **scratch pad**, not a public artifact like hiveplotlib-bioinformatics-examples: figure out what works there, then formally add support in hiveplotlib. Polish bar is "clean and working," not publication-curated repo. → Amendment needed: WS-0 brief names the repo and drops the seeded-repo assumption (→ Amendment 4).
- **Downstream deployment scoped.** The tutorial notebook is the durable artifact, in the same vein as the existing Bitcoin story notebook: "here's how these things work with this data." The figure does not land in the README; at most the README later links to the notebook as an encouraged starting point. README linkage stays out of this plan's workstreams (deferred follow-up note only).
- **G2 timing settled.** Sign the *shape* now (return `Dict[str, nx.Graph]`, baked attributes, private rewiring engine); the internal constants (lognormal params, swaps budget, render kwargs) are decided by WS-0's outcome and transcribed at WS-A. "We're just trying to figure out what's gonna work and then lock it down after." G2's shape component: **confirmed**.

### Maintainer shared-understanding pass (grill), Wave 2 — stripes/anchors, G4, branch, G7, stats verification (2026-06-11)

- **G3 generalized into a real-world-anchor requirement.** Stripes stays (Gary's instinct: bots engaging a narrow popularity band; sharpened in discussion to the broader anchor of *layered* systems — org hierarchies, supply chains, trophic levels). Binding principle: **every shipped variant needs a real-world anchor**, settled during story development and reviewed by Gary as he goes; the variant set (count and membership) is explicitly flexible, committed only to "each tells a good story and shows up interestingly as hive plots versus not hive plots." Anchors must be settled and amended into the plan before implementation.
- **G4 resolved: standalone tutorial**, in the vein of the existing Bitcoin story notebook. Capstone slot in the metrics series remains free to reuse the generator later.
- **Branch resolved:** WS-A/B/C land on a fresh branch off master after branch 46 merges. Gary does the branching himself and signals when it is time to implement on this machine.
- **Stats verified live during the grill** (Gary had only reviewed pictures): all five variants confirmed at 1000 nodes, 3068 edges, density 0.00614, min/mean/median/max degree 2/6.136/5/29, identical sorted degree sequence, and identical per-node degrees (the pixel-identical-positions guarantee). Differing-by-design: assortativity (+0.87 to -0.50), clustering (0.033 to 0.000). The notebook's beat-1 stats table shows the matched side; the writeup owns that non-degree statistics differ, which is the point.
- **G7 resolved: `example_same_stats_graphs`.** Gary floated `example_datasaurus_*`; agreed reasoning against: the name promises a dinosaur the plot doesn't deliver, and `same_stats_graphs` self-describes while citing the prior-art title. "Datasaurus" stays as the informal codename (repo name, prose homage, references). Revisit only if an actual dinosaur-shaped panel ever gets cracked.

### Maintainer shared-understanding pass (grill), Wave 3 — notebook drafted in the story repo (2026-06-11)

- **The tutorial notebook gets drafted in hiveplotlib-datasaurus, not hiveplotlib.** Gary's reasoning: the notebook is the only artifact that exercises the full story end to end, so it is the only place narrative breakage truly surfaces; if the story breaks, the fix is upstream in the data itself, which is the potentially long rabbit hole, and that iteration should happen where commits are cheap and off the release train. When story + notebook are settled, the notebook transitions to hiveplotlib.
- **Port discipline (agreed):** the draft notebook consumes a local prototype module shaped to the agreed API (same function name `example_same_stats_graphs`, same return dict, same baked attributes), so the eventual hiveplotlib port is a near-mechanical import swap and the port itself validates that the shipped API serves the story.
- This restructures WS-0 (gains the draft notebook and the real-world anchors deliverables) and WS-B (becomes a port-and-polish of the settled draft, not a from-scratch authoring). → routed to amend-plan (bundled with Wave 1-2 OPEN items as Amendment 4).

**Grill complete 2026-06-11.** No open divergence. Remaining user-held items: WS-0 figure/story sign-off itself (the work, not a disagreement), real-world anchors to be settled during WS-0 and amended in before WS-A, and the branch signal when it is time to implement in hiveplotlib.

## Gates (user decisions before dispatch)

All from the 2026-06-10 prototyping session; resolutions recorded at the 2026-06-11 grill (Amendment 4). Still open: G1 (it *is* WS-0's sign-off), the anchors amendment, and the branch signal. G6 stays deferred.

- **G1 — Story-repo sign-off (blocks WS-A; reworked by Amendments 3-4). OPEN.** WS-0 standardizes the scientific story, figures, real-world anchors, and draft tutorial notebook in `hiveplotlib-datasaurus`; G1 clears when Gary signs off on the standardized figures and settled draft. No hiveplotlib implementation until go.
- **G2 — API shape. RESOLVED (shape); constants at WS-0.** Confirmed: one generator returning `Dict[str, nx.Graph]` with hive-plot-ready node attributes baked in; rewiring engine and score functions private. The internal constants (lognormal params, swaps budget, render kwargs) are decided by WS-0's outcome and transcribed at WS-A. The `Dict[str, Tuple[NodeCollection, Edges]]` alternative is off the table (Amendment 1's critic justification stands).
- **G3 — Stripes variant. DISSOLVED into the real-world-anchor requirement (Amendment 4).** Every shipped variant needs a real-world anchor settled during WS-0 (stripes' anchor: layered systems); the variant set (count and membership) is flexible. No per-variant gate remains.
- **G4 — Standalone tutorial vs. capstone. RESOLVED: standalone**, in the vein of the Bitcoin story notebook. The metrics-series capstone slot stays free to reuse the generator later.
- **G5 — Composite-figure composition. RESOLVED: decided during WS-0 drafting**, where the figures standardize (the layouts-fail beat is structural per Amendment 2; whether the spring row sits in the shipped composite is settled in the draft notebook). Recommendation carried into WS-0: keep it; spring-next-to-hive-plots is the strongest single image.
- **G6 — Animated morph (deferred by default).** Pixel-identical node positions support an edges-only morph between variants. Deferred follow-up; not in any workstream.
- **G7 — Names. RESOLVED:** function `example_same_stats_graphs` (not `..._networks`); module file `same_stats_graphs.py`; notebook filename/title unchanged (`same_stats_different_graphs.ipynb` / "Same Stats, Different Graphs"). "Datasaurus" is the informal codename only (repo name, prose homage); it never names API surface. Variant keys and baked attributes per the Naming audit.
- **Branch (resolved at the grill):** WS-A/B/C land on a fresh branch off master after branch 46 merges. Gary does the branching himself and signals when it is time to implement.

## Prior ADRs / design docs

No ADRs exist yet (`wiki/wiki/adr/` absent; `scaling-large-networks.md` reserves first-promotion candidacy). Binding records are prior plans:

- `wiki/wiki/plans/graph-metrics-tutorial-series.md` — its Phase-3 capstone is flagged HIGHEST RISK for lack of a plain-data "surprising structure" dataset; this generator is a direct candidate feed (gate G4). This plan inherits its series conventions where the notebook is concerned: validate-then-commit dataset gate, naming audit, docs-registration mechanics, PLAIN-DATA rule (no GNN/ML framing), gallery-vs-tutorial split.
- `wiki/wiki/plans/graph-metrics-notebook-restructure.md` — in-flight; owns `examples/computing_graph_metrics.ipynb` (currently modified in the working tree) and `finding_a_partition.ipynb`. This plan does not touch either; coordinate docs-index and CHANGELOG edits. Its binding rule ">3 communities → HivePlotMatrix; never tune community resolution down to fake ≤3" is addressed under "Planted-communities honesty" below.
- `wiki/wiki/plans/networkx-metric-expansion-and-backend-refactor.md` — community-label contract (label 0 = largest, descending size) binds *detection outputs*. The generator's planted labels are ground truth, not detection output; emitted as strings `"A"/"B"/"C"` to make that unmistakable (see Naming audit).
- `wiki/wiki/plans/graph-metric-conflict-validation.md` — in-flight on this same branch (46-more-streamlined-networkx-usage-and-support), same v0.28.0 train; sequence CHANGELOG/notebook edits around it. Branch resolved at the grill (Amendment 4): WS-A/B/C land on a fresh branch off master after branch 46 merges; Gary branches and signals.
- `wiki/wiki/plans/scaling-large-networks.md` — no constraint at n=1000; matplotlib backend fine, no datashader requirement.
- Wiki anchors to link from the notebook and plan: `analyses/gnn-heterogeneity-hive-plots.md` (conceptual kin; keep ML framing out), `analyses/cora-prototype-plan.md` (HPM partition-sweep recipe), `concepts/structural-heterogeneity.md`, `concepts/node-assignment.md` (degree-rank sorting grounding), `concepts/hive-plot-matrix.md`.

## Patterns this replaces

None — net new addition (new datasets module, new notebook). Coordination notes, not replacements:

- `examples/computing_graph_metrics.ipynb` and `finding_a_partition.ipynb` are owned by the restructure plan; untouched here.
- The root-level `prototype_*.py` / `prototype_*.png` scratch files in the working tree are Gary's to clean (prefer-dump convention); no workstream touches them. Canonical prototypes live in `data/datasaurus_prototypes/` (gitignored).

## The mechanism (reference for all workstreams)

From `data/datasaurus_prototypes/README.md` (the authoritative writeup) and `prototype_degseq_v4.py` / `prototype_degseq_v5_community.py`:

1. One base graph: configuration model with lognormal degrees (n=1000, ~3000 edges, `mean=1.6, sigma=0.55`, min degree 2, simple graph, largest connected component, relabeled to consecutive ints). Lognormal, not Barabási-Albert: BA's degree-tie pileup collapsed the low axis in v1.
2. Degree-preserving double-edge swaps with hill climbing on a per-edge score of endpoint degree ranks r(u), r(v) (deterministic rank tiebreak: degree, then node id). ~20 accepted-or-not swap attempts per edge.
3. Variants: random (`0`), assortative (`-|r(u)-r(v)|`, reaches assortativity +0.87), disassortative (`+|r(u)-r(v)|`, -0.50), core-periphery (`max(r(u),r(v))`), stripes (`-||r(u)-r(v)| - n/2|`), community (`1 if same planted community else 0`; 3 communities by node id mod 3; reaches 93% within-community edges).
4. Invariants by construction: every variant has the identical degree sequence, hence identical n, m, density, and full degree histogram. With axes sorted by degree rank, node positions are pixel-identical across variants; only edges differ.
5. Validated render recipe: degree-tertile partition (`cutoffs=[n/3, 2n/3]`, labels low/med/high), `repeat_axes=True`, sort by degree rank, matplotlib, `all_edge_kwargs={"alpha": 0.08, "lw": 0.4}`.

### Layout counterfactuals (v6/v7 prototypes; Amendment 2)

From `data/datasaurus_prototypes/prototype_degseq_v6_layouts.py` / `prototype_v6_layouts.png` (five variants x four layouts) and `prototype_degseq_v7_ordered_circle.py` / `prototype_v7_ordered_circle.png` (arbitrary-order vs degree-ordered circle), reviewed by Gary 2026-06-11:

- **Random placement and arbitrary-order circular distinguish nothing** (identical noise/disks across all five variants); **spectral is degenerate** (corner clumps). These are strawmen and must not be the tutorial's headline foil.
- **Spring is the strongest fair opponent** and cracks exactly one variant: assortative unrolls into a comet, because +0.87 assortativity makes the graph a quasi-1D degree-band filament and any proximity embedding unrolls it. The tutorial uses spring as the foil and owns this caveat honestly.
- **Degree-ordered circle is the bridge beat**: identical geometry to the arbitrary circle, only the ordering becomes meaningful, and four of five variants separate (assortative hugs the rim with an empty center, disassortative shows center-crossing chords, core-periphery fans from the high-degree arc, stripes bursts a directional cone). Pedagogical point: placement encoding data is the single concession that makes layouts work, and it is the hive plot's founding premise (Krzywinski's "rational layout").
- **The degree-ordered circle stays blind to the planted-community variant**, same as the degree-partitioned hive plot, which hands off to the partition-sweep twist.

### Planted-communities honesty

The restructure plan's rule forbids tuning *detection* resolution down to fake ≤3 communities. The generator *plants* exactly 3 communities by construction; that is a synthetic design choice, not a detection result massaged to fit. The notebook must say so explicitly ("planted, not detected"), and if it runs community detection on the variant it must show what detection actually finds rather than asserting 3. The twist figure (degree partition vs. community partition) is the honest motivation for partition sweeps / `HivePlotMatrix`.

## Default justifications

For the generator signature (shape confirmed at G2; internal constants transcribed from WS-0's writeup at WS-A):

- `num_nodes=1000`: the validated render scale; large enough that edge-density signatures read as texture at notebook size, small enough to generate in seconds and stay within `make test-nb` budgets.
- `swaps_per_edge=20`: prototype-validated convergence (assortative reaches +0.87, community reaches 93% within-community) without runaway runtime.
- `seed=42`: reproduces the exact renders Gary reviews at G1, so the shipped notebook figures match the validated prototypes.
- `variants=None` → generate all variants: the demo's point is the side-by-side; callers wanting a subset pass an explicit list (also how tests keep runtime down).
- Lognormal parameters (mean 1.6, sigma 0.55, min degree 2) and the 3-community planting are internal constants, not parameters: the generator exists to tell one validated story, and every knob exposed is a knob that can break the matched-statistics invariant or the render recipe. Documented in the docstring instead.

## Naming audit

Checked against networkx vocabulary (the dominant adjacent ecosystem) and the `hiveplotlib.datasets` conventions (synthetic generators use `example_*` prefix, `seed` param).

- **Function: `example_same_stats_graphs` (G7 resolved, Amendment 4).** Keeps the `example_*` synthetic-generator convention, echoes the prior-art title ("Same Stats, Different *Graphs*"), and matches the literal return type. Rejected: `example_datasaurus_*` (promises a dinosaur the plot doesn't deliver; "datasaurus" stays an informal codename); `same_stats_different_graphs()` (breaks the `example_*` convention); `example_matched_degree_networks` (loses the citation hook). Module file: `same_stats_graphs.py`.
- **New parameters:** `num_nodes` (matches `example_node_data` et al.; not networkx's `n`, since the datasets module's own convention wins here), `swaps_per_edge`, `variants`, `seed`. All checked against networkx vocab: `seed` matches nx generators; `swaps_per_edge` echoes `nx.double_edge_swap`'s swap framing.
- **Variant keys (dict keys, user-facing):** `"random"`, `"assortative"`, `"disassortative"`, `"core_periphery"`, `"stripes"`, `"community"`. Snake_case for the multiword key (matches nx's `degree_assortativity_coefficient` vocabulary; the prototype's hyphenated `"core-periphery"` is amended → `core_periphery` because every other key is a valid identifier).
- **Baked node attributes:** `degree_rank` (not the prototype's `deg_rank`; no abbreviation in shipped surface), `degree_group` (values `"low"/"med"/"high"`, matching the validated render recipe), `community` (values `"A"/"B"/"C"`; strings, not ints, so they cannot be mistaken for detection-output labels bound by the label-0-largest contract).
- **Notebook (G7 resolved, unchanged):** `examples/same_stats_different_graphs.ipynb`, title `### Same Stats, Different Graphs`. The pun is literal for networks and cites the prior art. No collision in `examples/`.
- **Prose-only terms:** "degree-preserving rewiring," "double-edge swap," "planted communities," "configuration model" — all standard networkx/network-science vocabulary.

## API usage examples

### Proposed (from planner / Orchestrator)

Recommended shape: `example_same_stats_graphs(...) -> Dict[str, nx.Graph]`, every graph carrying `degree_rank`, `degree_group`, and `community` node attributes (identical across variants, since rewiring preserves degrees and the planting is by node id). Headline idiom (per the api-critic planning pass, Amendment 1): `HivePlot(graph=...)` (keyword-only, `src/hiveplotlib/hiveplot.py:2084`), which routes through the converter and carries node attributes into `nodes.data` (attribute flow verified at `src/hiveplotlib/converters.py:37-47`), so the attributes feed `partition_variable` / `sorting_variables` with no undocumented convention. The two-step `networkx_to_nodes_edges()` → `HivePlot(nodes=..., edges=...)` path remains available for users who want the intermediate `NodeCollection` / `Edges` objects, but is not the tutorial shape.

```python
# Example 1: generate the variants and verify the matched-statistics foil
# Example data: (generator is the data source)
from hiveplotlib.datasets import example_same_stats_graphs

graphs = example_same_stats_graphs(seed=42)

# Call site:
for name, g in graphs.items():
    degree_sequence = sorted(d for _, d in g.degree())
    print(name, g.number_of_nodes(), g.number_of_edges(), degree_sequence[-1])
# every line prints the same node count, edge count, and max degree
```

```python
# Example 2: render one variant on the shared degree-rank scaffold
# Example data:
from hiveplotlib import HivePlot
from hiveplotlib.datasets import example_same_stats_graphs
from hiveplotlib.viz import hive_plot_viz

graphs = example_same_stats_graphs(seed=42)

# Call site:
hp = HivePlot(
    graph=graphs["assortative"],
    partition_variable="degree_group",
    sorting_variables="degree_rank",
    axes_order=["low", "med", "high"],
    repeat_axes=True,
    all_edge_kwargs={"alpha": 0.08, "lw": 0.4, "color": "C0"},
)
fig, ax = hive_plot_viz(hp, node_kwargs={"s": 3, "color": "black", "alpha": 0.4})
```

```python
# Example 3: the community twist - same graph, repartitioned
# Example data:
from hiveplotlib import HivePlot
from hiveplotlib.datasets import example_same_stats_graphs
from hiveplotlib.viz import hive_plot_viz

graphs = example_same_stats_graphs(seed=42, variants=["random", "community"])

# Call site: under the degree partition this looks like "random";
# under the planted-community partition the within-community wedges light up
hp = HivePlot(
    graph=graphs["community"],
    partition_variable="community",
    sorting_variables="degree_rank",
    axes_order=["A", "B", "C"],
    repeat_axes=True,
    all_edge_kwargs={"alpha": 0.08, "lw": 0.4, "color": "C0"},
)
fig, ax = hive_plot_viz(hp)
```

Alternative shape weighed and rejected at the api-critic planning pass (G2 resolved nx-first; kept as the planning record): `Dict[str, Tuple[NodeCollection, Edges]]`. Trade-off: the story's acts 1-2 (stats table, spring-layout hairballs) live in networkx, and `HivePlot` accepts the converted pair in one documented call either way; native-first would instead make users round-trip through `nodes_edges_to_networkx()` for the foil acts. Recommendation stands at nx-first; the score functions and the rewiring engine stay private (`_steered_rewire`, module-level `_SCORE_FUNCTIONS`) in either shape.

### API Critic's take (planning mode)

**G2 recommendation: `Dict[str, nx.Graph]`.** Agreed with the planner, with a stronger justification than the plan currently gives: `HivePlot` accepts `graph=` directly (keyword-only, `hiveplot.py:2084`), and that path carries node attributes into `nodes.data` (it routes through the converter). So the "library-native types per datasets convention" alternative buys nothing — the nx return is already one documented call from a `HivePlot`, while native tuples would force a `nodes_edges_to_networkx()` round-trip for acts 1-2 (stats table, spring layouts), which are the story's whole point. The existing convention (generators return what the consuming notebook actually uses: `example_hive_plot` → `HivePlot`, `example_trade_nodes_and_edges` → tuple) supports nx.Graph here, not contradicts it.

**Amend Examples 2 and 3: headline `HivePlot(graph=...)`, drop the converter.** The two-step `networkx_to_nodes_edges()` → `HivePlot(nodes=..., edges=...)` path is the lower-level path leaking into the headline. Preferred form:

```python
hp = HivePlot(
    graph=graphs["assortative"],
    partition_variable="degree_group",
    sorting_variables="degree_rank",
    axes_order=["low", "med", "high"],
    repeat_axes=True,
    all_edge_kwargs={"alpha": 0.08, "lw": 0.4, "color": "C0"},
)
```

One fewer import, one fewer concept, and it showcases the `graph=` parameter this branch just shipped. `graph_directed` / `graph_multigraph` infer correctly from a plain `nx.Graph` (undirected, simple), so no extra kwargs needed. The feasibility audit's converter-path verification stands as verification; it just shouldn't be the tutorial shape.

**Variants parameter shape: agreed** (`Optional[List[str]]`, `None` → all). Two requirements for WS-A: (1) the invalid-name exception message must enumerate the valid variant keys (no other discovery channel exists before generating); (2) the returned dict's insertion order is user-facing — the notebook builds the five-panel figure via `graphs.items()` — so fix and document a canonical order (suggest the narrative order: `random`, `assortative`, `disassortative`, `core_periphery`, `stripes`, `community`) and preserve it under `variants` subsetting.

**Baked attributes: agreed.** `degree_rank` / `degree_group` / `community` with string community labels `"A"/"B"/"C"` is exactly right (the string-vs-int distinction from detection labels is a real safeguard, not pedantry). `"low"/"med"/"high"` values match the existing toy-dataset vocabulary.

**Rewiring engine private: agreed.** The generator tells one validated story; a public engine is a knob that breaks the matched-statistics invariant, and `nx.double_edge_swap` already serves users wanting generic degree-preserving rewiring.

**Seed and signature: agreed** as justified in Default justifications (`seed=42` matches `example_hpm_nodes_and_edges`'s precedent and reproduces the G1 renders).

**Low-confidence, for G7:** `example_same_stats_graphs` would echo the prior-art title ("Same Stats, Different *Graphs*") and the literal return type one notch more literally than `..._networks`. Either is fine; Gary's call.

**Recurring pattern:** plans drafted before `graph=` shipped on this branch default to the converter idiom in usage examples. Check any future plan's snippets against the current `HivePlot` constructor surface, not the surface at drafting time.

### API Critic — post-implementation review

```
Pending — invoke api-critic in post-implementation mode after Workstream A ships.
```

## Feasibility audit

- Net-new entry point `example_same_stats_graphs`: output traces to `nx.Graph`, consumed by the documented `HivePlot(graph=...)` path (keyword-only, `hiveplot.py:2084`), which routes through `networkx_to_nodes_edges()`; node attributes become `NodeCollection.data` columns (verified, `converters.py:37-47`), which are exactly what `partition_variable` / `sorting_variables` / `axes_order` consume. `graph_directed` / `graph_multigraph` infer correctly from a plain `nx.Graph`. No undocumented convention needed. (Headline idiom set by Amendment 1; the converter-path attribute trace stands as the verification.)
- Optional-dependency wiring: `hiveplotlib/datasets/__init__.py` star-imports every module, so the new module **must not** raise at import time (unlike `converters.py`, whose module-level guard is fine because nothing star-imports it into a non-networkx path). The networkx import goes *inside* the generator function, wrapped with the same helpful message naming the `[networkx]` extra (`converters.py:16` is the message template), `# pragma: no cover` on the ImportError branch matching the existing pattern. This keeps `from hiveplotlib.datasets import example_hive_plot` working without networkx.
- Performance: the rewiring loop is pure Python, ~20 x m iterations per variant (~60k at n=1000); seconds per variant, acceptable for the notebook under `make test-nb`. Tests use small `num_nodes` and `swaps_per_edge` (see WS-A done-when).

## Notebook review

Net-new notebook (no existing page's scope drifts), drafted in hiveplotlib-datasaurus during WS-0 and ported at WS-B (Amendment 4); editorial review runs against the ported notebook. Class: `HivePlot` primary; genre: tutorial (narrative arc, `#### Background`, `#### References` citing arXiv:1808.09913); dataset: the new synthetic generator only. Flag for sign-off at authoring: if the community-twist act is built as an actual `HivePlotMatrix` partition sweep rather than v5's 2x2 of `HivePlot`s, the closer leans HPM — acceptable as a closing hand-off, but the page's primary subject must stay `HivePlot`; editorial-critic checks this.

```
Pending — invoke editorial-critic in post-implementation mode after Workstream B ships.
```

## Workstreams

WS-0 blocks WS-A (its sign-off *is* the reworked G1; the anchors must also be amended into this plan, and Gary signals the fresh branch off master post-46-merge). WS-B blocks on WS-A and WS-0's settled draft notebook. WS-C blocks on WS-B.

### Workstream 0: Story development in hiveplotlib-datasaurus (Amendments 3-4)

**Status:** not started — the plan's next action; Gary-driven with session support, no hiveplotlib dispatch
**Files:** none in hiveplotlib. The story repo is **`hiveplotlib-datasaurus`** (adjacent repo on this machine; already exists). It is a scratch pad, not a public curated artifact like hiveplotlib-bioinformatics-examples: figure out what works there, then formally add support in hiveplotlib. Polish bar: "clean and working." Absorbs `data/datasaurus_prototypes/` (v4 composition, v5 community, v6 layouts, v7 ordered circle) so iteration is committable and reviewable off hiveplotlib's release train.
**Boundary (binding):** the story repo owns narrative, figure design, and parameter choices. This plan owns API, tests, tutorial port, and docs registration. The story repo must not accrete API design decisions: generator signature, return shape, variant keys, and baked attribute names belong to this plan's G2 (resolved); any pressure on those surfaces routes back here via amend-plan.
**Done when:**

1. Prototypes migrated and the figures standardized (the four-beat ladder's figure set: stats table, spring foil, degree-ordered circle, hive plots, community twist).
2. **Real-world anchors settled (Amendment 4; absorbs G3).** Every shipped variant has a real-world anchor, settled during story development and reviewed by Gary as he goes. Working set from the grill: assortative = social homophily by popularity; disassortative = sexual contact / predator-prey; core-periphery = interbank lending / internet AS topology / trade; stripes = layered systems (org hierarchies, supply chains, trophic levels, narrow-band bots); community = social/biological modularity; random = the null model. The variant set (count and membership) is flexible. The settled anchors are amended into this plan before WS-A dispatches.
3. **Draft tutorial notebook authored in hiveplotlib-datasaurus (Amendment 4)** as the end-to-end story dress rehearsal: it is the only artifact exercising the full story, so narrative breakage surfaces there, and data-level fixes (the potentially long rabbit hole) stay off the release train. **Port discipline (binding):** the draft consumes a local prototype module shaped to the agreed API (function name `example_same_stats_graphs`, same return dict, same baked attributes) so the WS-B port is a near-mechanical import swap that validates the shipped API. Composite-figure composition (former G5, e.g. spring row in or out) is decided here, where the figures standardize.
4. Parameter choices that the generator will later bake in (lognormal params, swaps budget, render kwargs) are recorded in the story repo's writeup so WS-A transcribes rather than re-derives.
5. Gary signs off on the standardized figures and the settled draft notebook (clears G1).

### Workstream A: Dataset generator + tests

**Status:** not started (gated: WS-0 sign-off / G1, anchors amended into this plan, Gary's branch signal — fresh branch off master after 46 merges. G2 shape resolved; internal constants transcribed from WS-0's writeup)
**Files:** `src/hiveplotlib/datasets/same_stats_graphs.py` (new; name per G7), `src/hiveplotlib/datasets/__init__.py` (star-import line), `tests/datasets_test.py` (new test block) or a mirrored new test module — follow the existing flat `tests/` convention.
**Done when:**

1. Generator exists with the G2-decided signature, docstring at 120-char wrap documenting the mechanism, the matched invariants, the baked node attributes, the internal constants (lognormal params, 3 planted communities), and citing Chen et al. (arXiv:1808.09913).
2. Returned dict uses the canonical narrative insertion order — `random`, `assortative`, `disassortative`, `core_periphery`, `stripes`, `community` — documented in the docstring and preserved under `variants` subsetting (insertion order is user-facing: the notebook builds the multi-panel figure via `graphs.items()`). (Amendment 1)
3. networkx imported inside the function with the helpful `[networkx]`-extra error; `import hiveplotlib.datasets` succeeds without networkx installed.
4. Tests (`@pytest.mark.networkx`) at small scale (e.g. `num_nodes<=100`, `swaps_per_edge<=3`) assert: identical sorted degree sequence across all variants; identical node and edge counts; assortativity sign separation between assortative and disassortative variants at default scale parameters scaled down; community variant's within-community edge fraction exceeds the random variant's; `variants` subset selection works and preserves canonical order; invalid variant name raises a `hiveplotlib` exception (from `src/hiveplotlib/exceptions/`) whose message enumerates the valid variant keys (no other discovery channel exists before generating — Amendment 1); baked attributes present with the documented names and dtypes; determinism under fixed seed. 100% coverage; added test runtime bounded (~seconds, not minutes).
5. Type hints (`Union`, not `|`), ruff format/check clean, `make ty` clean.
6. api-critic post-implementation review filled in this plan; any must-fix routed through amend-plan.

### Workstream B: Tutorial notebook (port-and-polish; reframed by Amendment 4)

**Status:** not started (gated: WS-A shipped, WS-0's draft notebook settled and signed off. G4 resolved standalone; G5/G3 settled in WS-0)
**Files:** `examples/same_stats_different_graphs.ipynb` (new; filename per G7). Nothing under `docs/source/notebooks/` or `gallery_examples/` (auto-generated).
**Scope:** port the settled WS-0 draft from hiveplotlib-datasaurus and polish to hiveplotlib's shipped-notebook bar; not from-scratch authoring. The data import swaps from the local prototype module to `hiveplotlib.datasets` (near-mechanical by WS-0's port discipline; the swap itself validates the shipped API). Narrative or figure-composition problems found here are story problems and route back to WS-0 / amend-plan, not fixed ad hoc in the port.
**Done when:**

1. Four-beat ladder, carried over from the settled draft, (Amendment 2; details in "Layout counterfactuals"): (beat 1) the stats table foil — variants shown identical on n, m, density, degree histogram; (beat 2) standard layouts fail — spring as the foil, not the strawmen (random placement / arbitrary circle / spectral may appear as supporting evidence but not as the headline opponent), with the assortative-comet caveat owned honestly (spring cracks exactly that one variant); (beat 3) the degree-ordered circle gets halfway — identical geometry to the arbitrary circle, only the ordering meaningful, four of five variants separate; placement-encoding-data named as the single concession and as the hive plot's founding premise (Krzywinski "rational layout"); still blind to the community variant; (beat 4) hive plots complete it — explicit labeled partitions on the shared degree-rank scaffold (pixel-identical node positions made explicit), repeat axes for within- vs between-group traffic, swappable partition variables; then the twist — the community variant invisible under the degree partition, popping under the community partition, handing off to partition sweeps / `HivePlotMatrix`.
2. Hive plot construction uses the headline `HivePlot(graph=...)` idiom from "API usage examples" (no `networkx_to_nodes_edges` round-trip unless the prose has a reason to show the intermediate objects); multi-panel figures iterate `graphs.items()` and so inherit the canonical variant order. (Amendment 1)
3. Planted-communities honesty language per "The mechanism" section: planted, not detected; no detection result massaged to 3.
4. `#### Background` + `#### References` (arXiv:1808.09913 and the Datasaurus Dozen as the conceptual ancestor); install-extras (`[networkx]`) surfaced up front; PLAIN-DATA rule held (no GNN/ML framing).
5. House voice (no em-dashes, no AI filler); length disciplined against the closest tutorial sibling.
6. Runs end-to-end under `make test-nb` (warnings-as-errors) in acceptable time at the shipped generator scale.
7. editorial-critic and viz-critic post-impl both run; editorial-critic's Notebook review section filled above; no open must-fix (must-fix routes to amend-plan).

### Workstream C: Docs + CHANGELOG registration

**Status:** not started (gated: WS-B shipped)
**Files:** `docs/source/notebooks/index.rst` (both the `nblinkgallery` block and the hidden `toctree`, same order; section placement: standalone per G4, landing in an existing section, likely `Hive Plot Examples`, unless Gary places it more prominently), `docs/source/conf.py` (`nbsphinx_thumbnails` entry), `docs/source/_static/<thumbnail>`, `CHANGELOG.rst` (one new-feature entry in the v0.28.0 WIP block covering generator + notebook together; no separate entries for same-version refinements), API docs page for the new datasets module if the datasets API reference enumerates modules.
**Done when:**

1. Notebook registered in both `index.rst` blocks, same order; thumbnail entry + `_static/` asset in place.
2. CHANGELOG v0.28.0 WIP entry added, sequenced cleanly against the in-flight conflict-validation and notebook-restructure plans' edits (append, don't reflow).
3. `make docs` builds with no new warnings (regular target, scan the full warning set).
4. qa-engineer release-readiness pass: tests/lint/type green, grep confirms single registration, Implementation log updated.

## Plan amendments

### Amendment 1 (2026-06-10) — In-scope tweaks from api-critic planning pass

Delta source: "API Critic's take (planning mode)" above; pre-dispatch (WS-A/WS-B not started), so all three land as in-scope tweaks to existing briefs — no new workstream, no specialist-span change.

1. **Headline idiom is `HivePlot(graph=...)`.** Examples 2 and 3 rewritten to pass the generator's `nx.Graph` directly (keyword-only `graph=`, `hiveplot.py:2084`; node attributes flow into `nodes.data` via the converter route, so `partition_variable` / `sorting_variables` work unchanged). The two-step `networkx_to_nodes_edges()` path stays documented as the lower-level alternative, and its attribute trace remains the feasibility verification. Plan was drafted before `graph=` shipped on this branch; per the critic, check future plans' snippets against the current constructor surface. WS-B done-when 2 added (notebook uses the headline idiom).
2. **Invalid variant name → error enumerates valid keys.** Folded into WS-A done-when 4; no other discovery channel exists before generating.
3. **Canonical variant order is user-facing.** Returned dict fixes and documents the narrative insertion order `random`, `assortative`, `disassortative`, `core_periphery`, `stripes`, `community`, preserved under `variants` subsetting. New WS-A done-when 2; order-preservation assertion added to done-when 4; WS-B figures iterate `graphs.items()` and inherit it.

G2 note: the critic's pass endorses the `Dict[str, nx.Graph]` recommendation with a stronger justification (native tuples would force a round-trip for the foil acts). G2 remains Gary's call, but planner and critic now agree.

### Amendment 2 (2026-06-11) — In-scope tweak: WS-B narrative ladder gains two counterfactual beats

Delta source: Gary's review of new prototype evidence (`data/datasaurus_prototypes/prototype_degseq_v6_layouts.py` / `prototype_v6_layouts.png`, `prototype_degseq_v7_ordered_circle.py` / `prototype_v7_ordered_circle.png`). This changes what the notebook teaches, which would normally require sign-off; the delta source *is* Gary's directive, so sign-off is in hand. WS-B not started, so it lands as a brief rewrite, no specialist-span change.

1. **Findings encoded** in the new "Layout counterfactuals (v6/v7 prototypes)" subsection under "The mechanism": strawmen distinguish nothing (random placement, arbitrary circle; spectral degenerate); spring is the strongest fair opponent, cracking exactly the assortative variant (comet from the quasi-1D degree-band filament; any proximity embedding unrolls it); degree-ordered circle is the bridge (same geometry, ordering meaningful, four of five separate, blind to community, like the degree-partitioned hive plot).
2. **WS-B done-when 1 rewritten** from the three-act arc to the four-beat ladder: stats table → layouts fail (spring as foil, comet caveat owned honestly) → degree-ordered circle halfway (placement-encoding-data as the single concession; Krzywinski "rational layout") → hive plot completes it, with the community twist as before.
3. **G5 narrowed:** the layouts-fail beat is now structural, so G5 reduces to composite-figure composition (spring row in or out); recommendation unchanged.

### Amendment 3 (2026-06-11) — Added workstream: WS-0 story repo standardization (sequencing change)

Delta source: Gary's decision. Before WS-A lands code in hiveplotlib, the scientific story and figures standardize in a separate scratch repo (the hiveplotlib-bioinformatics-examples pattern), keeping iteration committable and reviewable off hiveplotlib's release train (branch 46 already carries two in-flight plans).

1. **New Workstream 0** added, gating WS-A: migrate the v4-v7 prototypes into the story repo, standardize the four-beat figure set, record the parameter choices WS-A will transcribe. No hiveplotlib files.
2. **G1 reworked:** the render review of two PNGs evolves into sign-off on the story repo's standardized figures; WS-0 done-when 2 *is* the gate.
3. **Migration boundary (binding):** story repo owns narrative, figure design, parameter choices; this plan owns API, tests, tutorial port, docs registration. The story repo must not accrete API design decisions; generator signature / return shape / variant keys / baked attributes stay with G2 (planner-and-critic-agreed, Amendment 1). Pressure on those surfaces routes back via amend-plan.
4. WS-A/B/C briefs otherwise unchanged; they proceed in hiveplotlib as specified once the story stabilizes.

Process note: the grill-me alignment pass is scheduled as the next step after this amendment (Gary's plan-review mechanism in place of a full read); `## Alignment (grill)` placeholder updated to say so.

### Amendment 4 (2026-06-11) — Grill-close bundle: JOSS cross-link, story repo concretized, WS-0 deliverables, WS-B reframe, gate resolutions

Delta source: the completed grill-me pass (Waves 1-3 in `## Alignment (grill)`, authoritative captures of Gary's calls; sign-off in hand for every item).

1. **In-scope tweak — JOSS dependency cross-linked.** This plan is the "Datasaurus figure" gate (G2) in `wiki/wiki/plans/joss-submission.md`. Noted in this plan's Goal; reciprocal edit made in `joss-submission.md` (its G2 and feeder-page entry now point here as the work that must complete first). The one cross-plan edit performed under this amendment.
2. **In-scope tweak — story repo concretized.** The repo exists: `hiveplotlib-datasaurus` (adjacent, on this machine). Scratch pad, not a public curated artifact; polish bar "clean and working." WS-0 brief updated; "name Gary's call" and the seeded-repo assumption dropped.
3. **In-scope tweak (WS-0 brief grows two deliverables; sign-off in hand from the grill).** (a) **Real-world anchors:** every shipped variant needs one (working set in WS-0 done-when 2); settled during story development, reviewed by Gary as he goes, amended into this plan before WS-A dispatches; variant set flexible; G3 dissolves into this requirement. (b) **Draft tutorial notebook** authored in hiveplotlib-datasaurus as the end-to-end dress rehearsal (narrative breakage surfaces there; data-level fixes stay off the release train), under binding port discipline: the draft consumes a local prototype module shaped to the agreed API (name, return dict, baked attributes) so the WS-B port is a near-mechanical import swap validating the shipped API.
4. **In-scope tweak — WS-B reframed** as port-and-polish of the settled draft, not from-scratch authoring; story problems found at port route back to WS-0 / amend-plan. Gating simplified: G4 resolved standalone; G5 (composite-figure composition) decided during WS-0 drafting where the figures standardize.
5. **Gate resolutions recorded:** G2 shape confirmed (`Dict[str, nx.Graph]`, baked attributes, private engine; internal constants transcribed from WS-0); G3 dissolved (item 3a); G4 standalone; G5 at WS-0; G7 resolved — `example_same_stats_graphs` / `same_stats_graphs.py` / notebook filename and title unchanged; "datasaurus" informal codename only. **Branch:** WS-A/B/C on a fresh branch off master after branch 46 merges; Gary branches and signals.
6. **Stale-reference sweep:** `example_same_stats_networks` → `example_same_stats_graphs` everywhere (Naming audit, API usage examples, Feasibility audit, WS-A files); G3/G4/G5/G7 gate text updated to resolutions; branch sentence in "Prior ADRs" updated; Alignment header marked complete.

## Holdouts

- Root-level `prototype_*.py` / `prototype_*.png` (untracked) and `data/datasaurus_prototypes/` (gitignored): scratch record of the prototyping sessions; no workstream edits or removes them in this repo. WS-0 copies them into hiveplotlib-datasaurus (read-only here).
- `examples/computing_graph_metrics.ipynb`, `finding_a_partition.ipynb`: owned by the restructure plan.

## Implementation log

Append-only. After each workstream completes, one line in the same turn.

_None yet._
