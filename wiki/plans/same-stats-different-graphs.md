# Plan: Same Stats, Different Graphs (matched-statistics network demo)

<!--
Hiveplotlib consumer plan. Formal library support for the Datasaurus-style
"identical graph statistics, different structure" demo prototyped 2026-06-10
(scratch work in data/datasaurus_prototypes/, gitignored; its README.md is the
mechanism reference). Prototype renders and stats reviewed by Gary (grill,
2026-06-11); story standardization complete in hiveplotlib-datasaurus; sign-off
arrives as Gary's prepared kickoff prompt (G1 semantics in Amendment 5).
-->

## Goal

Ship the "Same Stats, Different Graphs" demo as a first-class part of the library: a synthetic dataset generator in `hiveplotlib.datasets` producing six matched-statistics variants (the settled story ships a four-variant ladder plus the community twist; `core_periphery` left the story 2026-06-12 and ships generator-only, Amendment 5) with pixel-identical degree statistics (same n, m, density, full degree sequence) but instantly distinguishable hive plot signatures, and a tutorial notebook telling the four-beat ladder (stats table foil, standard layouts fail, degree-ordered circle gets halfway, hive plots complete it) plus the hidden-communities twist that motivates partition sweeps. (Ladder set by Amendment 2; the story and figures standardize in a separate story repo before any hiveplotlib code lands, per Amendment 3.) This becomes a centerpiece of the "why choose hive plots" pitch. The variant set (count and membership) is flexible; every shipped variant carries a real-world anchor (Amendment 4; the settled anchor table is folded in under "The mechanism", Amendment 5). **JOSS dependency:** this work is the "Datasaurus figure" gate (its G2) in `wiki/wiki/plans/joss-submission.md`; the JOSS submission blocks on this plan completing, and that plan now cross-links here (Amendment 4). Prior art: Chen, Soni, Lu, Maciejewski, Kobourov, "Same Stats, Different Graphs" (GD 2018, arXiv:1808.09913), which makes the statistical point with force-directed drawings; we carry it through to hive plots.

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

All from the 2026-06-10 prototyping session; resolutions recorded at the 2026-06-11 grill (Amendment 4). Still open: G1 and the branch signal only, and they arrive together as Gary's prepared kickoff prompt (semantics recorded in G1 below; Amendment 5); the anchors amendment has landed (Amendment 5). G6 stays deferred.

- **G1 — Story-repo sign-off (blocks WS-A; reworked by Amendments 3-4). OPEN.** WS-0 standardizes the scientific story, figures, real-world anchors, and draft tutorial notebook in `hiveplotlib-datasaurus`; G1 clears when Gary signs off on the standardized figures and settled draft. No hiveplotlib implementation until go. **Kickoff semantics (Amendment 5): Gary sending the prepared WS-A/B/C kickoff prompt, together with his branch signal, *is* the G1 sign-off.** Sending it covers, specifically: (a) the standardized figure set (`figures/fig1_stats_table.png` + `fig1_stats_table_no_community.png`, `fig2_spring.png`, `fig3_ordered_circle.png`, `fig4_hive.png`, `fig5_community_twist.png`); (b) the corrected anchors (`ANCHORS.md` including the 2026-07-04 disassortative correction and Liljeros reference); (c) the caveat-bearing draft narrative (`build_notebook.py` including the Beat-3 ordering-vs-glyph paragraph); (d) the settled 4+1 story shape with `core_periphery` generator-only; (e) the internal constants in `same_stats_graphs.py` as WS-A's transcription source. Precondition: Gary commits the two pending story-repo edits to its `main` first; if the kickoff session finds the port-relevant story-repo working tree differing from `HEAD`, it halts and asks rather than picking a side.
- **G2 — API shape. RESOLVED (shape); constants at WS-0.** Confirmed: one generator returning `Dict[str, nx.Graph]` with hive-plot-ready node attributes baked in; rewiring engine and score functions private. The internal constants (lognormal params, swaps budget, render kwargs) are decided by WS-0's outcome and transcribed at WS-A. The `Dict[str, Tuple[NodeCollection, Edges]]` alternative is off the table (Amendment 1's critic justification stands).
- **G3 — Stripes variant. DISSOLVED into the real-world-anchor requirement (Amendment 4).** Every shipped variant needs a real-world anchor settled during WS-0 (stripes' anchor: layered systems); the variant set (count and membership) is flexible. No per-variant gate remains.
- **G4 — Standalone tutorial vs. capstone. RESOLVED: standalone**, in the vein of the Bitcoin story notebook. The metrics-series capstone slot stays free to reuse the generator later.
- **G5 — Composite-figure composition. RESOLVED: decided during WS-0 drafting**, where the figures standardize (the layouts-fail beat is structural per Amendment 2; whether the spring row sits in the shipped composite is settled in the draft notebook). Recommendation carried into WS-0: keep it; spring-next-to-hive-plots is the strongest single image.
- **G6 — Animated morph (deferred by default).** Pixel-identical node positions support an edges-only morph between variants. Deferred follow-up; not in any workstream.
- **G7 — Names. RESOLVED:** function `example_same_stats_graphs` (not `..._networks`); module file `same_stats_graphs.py`; notebook filename/title unchanged (`same_stats_different_graphs.ipynb` / "Same Stats, Different Graphs"). "Datasaurus" is the informal codename only (repo name, prose homage); it never names API surface. Variant keys and baked attributes per the Naming audit.
- **Branch (resolved at the grill):** WS-A/B/C land on a fresh branch off master after branch 46 merges. Gary does the branching himself and signals when it is time to implement. (Since resolved in reality: branch 46 merged and v0.28 shipped, so the precondition is cleared; the signal rides in the kickoff prompt, Amendment 5.)

## Prior ADRs / design docs

Written when no ADRs existed; ADR 0001 (networkx-integration) and ADR 0002 (performance-regression-harness) have since landed. 0001 binds networkx-facing conventions and is pre-read for WS-A (Amendment 5). Otherwise the binding records are prior plans:

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

From `data/datasaurus_prototypes/README.md` (the original writeup) and `prototype_degseq_v4.py` / `prototype_degseq_v5_community.py`; superseded as WS-A's transcription source by the story repo's `same_stats_graphs.py` itself (Amendment 5), which the description below still matches:

1. One base graph: configuration model with lognormal degrees (n=1000, ~3000 edges, `mean=1.6, sigma=0.55`, min degree 2, simple graph, largest connected component, relabeled to consecutive ints). Lognormal, not Barabási-Albert: BA's degree-tie pileup collapsed the low axis in v1.
2. Degree-preserving double-edge swaps with hill climbing on a per-edge score of endpoint degree ranks r(u), r(v) (deterministic rank tiebreak: degree, then node id). ~20 accepted-or-not swap attempts per edge.
3. Variants: random (`0`), assortative (`-|r(u)-r(v)|`, reaches assortativity +0.87), disassortative (`+|r(u)-r(v)|`, -0.50), core-periphery (`max(r(u),r(v))`), stripes (`-||r(u)-r(v)| - n/2|`), community (`1 if same planted community else 0`; 3 communities by node id mod 3; reaches 93% within-community edges).
4. Invariants by construction: every variant has the identical degree sequence, hence identical n, m, density, and full degree histogram. With axes sorted by degree rank, node positions are pixel-identical across variants; only edges differ.
5. Validated render recipe: degree-tertile partition (`cutoffs=[n/3, 2n/3]`, labels low/med/high), `repeat_axes=True`, sort by degree rank, matplotlib, `all_edge_kwargs={"alpha": 0.08, "lw": 0.4}`.

### Real-world anchors (settled at WS-0; folded in by Amendment 5)

Transcribed from the story repo's `ANCHORS.md` (working tree as of 2026-07-06, including the uncommitted 2026-07-04 correction; verified directly against the file and its diff). This discharges WS-0 done-when 2's fold-in requirement before WS-A dispatches. Panel titles lead with the plain name and put the shorthand in parentheses; prose then uses the shorthand freely once introduced.

| variant | plain name | anchor | status |
|---|---|---|---|
| `random` | no preference | the configuration-model null: "what you'd get by chance," degree-preserving rewiring as the standard null model | in story (high confidence) |
| `assortative` | like connects with like | social homophily by popularity; Newman (2002) found social networks consistently assortative | in story (high) |
| `disassortative` | high connects with low | internet routers, protein interaction, predator-prey; consistently degree-disassortative in Newman (2002). Sexual-contact networks dropped 2026-07-04: scale-free degree distribution (Liljeros 2001) and disassortative by gender (an attribute), not by degree, which is the quantity these variants hold fixed | in story (high) |
| `core_periphery` | dense core, sparse fringe | interbank lending (money-center vs regional banks), international trade, internet AS topology, airline hub-and-spoke | cut from the story 2026-06-12: visually near-identical to disassortative under the degree-tertile partition (both hit -0.50 assortativity), and a core-focused cutoff rescue felt more contrived than the redundancy it fixed. Ships in the generator, absent from the notebook (Amendment 5) |
| `stripes` | layered connections | honest anchor: the bot band (automated accounts engaging a narrow popularity band), where the rank-offset construction is the mechanism; org hierarchies / supply chains / trophic levels are layered by role, not degree rank (metaphor, not model) | in the notebook (medium); likely out of the JOSS headline figure, where the Newman-anchored triad carries the claim |
| `community` | hidden groups | modular systems: social cliques, biological functional modules, topical clusters | in story (high); the twist |

References: Newman, *Assortative Mixing in Networks*, PRL 89, 208701 (2002); Liljeros et al., *The web of human sexual contacts*, Nature 411, 907-908 (2001).

Binding figure-design principle from the same review (for WS-B): the ladder figure holds the question fixed (one partition, one sorting) and varies the network; the twist varies the question. Don't blur the two by giving ladder panels bespoke partitions.

### Layout counterfactuals (v6/v7 prototypes; Amendment 2)

From `data/datasaurus_prototypes/prototype_degseq_v6_layouts.py` / `prototype_v6_layouts.png` (five variants x four layouts) and `prototype_degseq_v7_ordered_circle.py` / `prototype_v7_ordered_circle.png` (arbitrary-order vs degree-ordered circle), reviewed by Gary 2026-06-11:

- **Random placement and arbitrary-order circular distinguish nothing** (identical noise/disks across all five variants); **spectral is degenerate** (corner clumps). These are strawmen and must not be the tutorial's headline foil.
- **Spring is the strongest fair opponent** and cracks exactly one variant: assortative unrolls into a comet, because +0.87 assortativity makes the graph a quasi-1D degree-band filament and any proximity embedding unrolls it. The tutorial uses spring as the foil and owns this caveat honestly.
- **Degree-ordered circle is the bridge beat**: identical geometry to the arbitrary circle, only the ordering becomes meaningful, and four of five variants separate (assortative hugs the rim with an empty center, disassortative shows center-crossing chords, core-periphery fans from the high-degree arc, stripes bursts a directional cone). (Prototype-era counts: with `core_periphery` since cut from the story, the settled notebook's beat 3 separates three of its four ladder variants; Amendment 5.) Pedagogical point: placement encoding data is the single concession that makes layouts work, and it is the hive plot's founding premise (Krzywinski's "rational layout").
- **The degree-ordered circle stays blind to the planted-community variant**, same as the degree-partitioned hive plot, which hands off to the partition-sweep twist.

### Planted-communities honesty

The restructure plan's rule forbids tuning *detection* resolution down to fake ≤3 communities. The generator *plants* exactly 3 communities by construction; that is a synthetic design choice, not a detection result massaged to fit. The notebook must say so explicitly ("planted, not detected"), and if it runs community detection on the variant it must show what detection actually finds rather than asserting 3. The twist figure (degree partition vs. community partition) is the honest motivation for partition sweeps / `HivePlotMatrix`.

## Default justifications

For the generator signature (shape confirmed at G2; internal constants transcribed from the story module `same_stats_graphs.py` at WS-A, Amendment 5):

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

### Adversary — post-impl reviews (Amendment 5)

```
Pending — this plan predates the cold-adversary harness feature (grilled 2026-06-11; no
planning-mode challenge exists to reconcile against). Under the current harness the adversary
post-impl pass is mandatory per shipped workstream: run it on WS-A, WS-B, and WS-C alongside
the other critics (two-message blind-first dispatch; no `## Failure modes` rubric exists in
this plan, so the message-1 extract is the workstream block, its done-whens, and Holdouts).
Findings route through amend-plan. No rubric also means auto-dispatch is not offerable;
per-workstream pauses stand.
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

**Status:** complete except sign-off (Amendment 5, 2026-07-06). Deliverables 1-4 live in the story repo (`main` at `9596090` plus two working-tree edits Gary commits at kickoff: the `ANCHORS.md` 2026-07-04 correction and the `build_notebook.py` Beat-3 caveat); done-when 2's fold-in was performed by Amendment 5. Sign-off (done-when 5 / G1) is Gary's kickoff prompt with branch signal.
**Files:** none in hiveplotlib. The story repo is **`hiveplotlib-datasaurus`** (adjacent repo on this machine; already exists). It is a scratch pad, not a public curated artifact like hiveplotlib-bioinformatics-examples: figure out what works there, then formally add support in hiveplotlib. Polish bar: "clean and working." Absorbs `data/datasaurus_prototypes/` (v4 composition, v5 community, v6 layouts, v7 ordered circle) so iteration is committable and reviewable off hiveplotlib's release train.
**Boundary (binding):** the story repo owns narrative, figure design, and parameter choices. This plan owns API, tests, tutorial port, and docs registration. The story repo must not accrete API design decisions: generator signature, return shape, variant keys, and baked attribute names belong to this plan's G2 (resolved); any pressure on those surfaces routes back here via amend-plan.
**Done when:**

1. Prototypes migrated and the figures standardized (the four-beat ladder's figure set: stats table, spring foil, degree-ordered circle, hive plots, community twist).
2. **Real-world anchors settled (Amendment 4; absorbs G3).** Every shipped variant has a real-world anchor, settled during story development and reviewed by Gary as he goes. Working set from the grill: assortative = social homophily by popularity; disassortative = sexual contact / predator-prey; core-periphery = interbank lending / internet AS topology / trade; stripes = layered systems (org hierarchies, supply chains, trophic levels, narrow-band bots); community = social/biological modularity; random = the null model. The variant set (count and membership) is flexible. The settled anchors are amended into this plan before WS-A dispatches.
3. **Draft tutorial notebook authored in hiveplotlib-datasaurus (Amendment 4)** as the end-to-end story dress rehearsal: it is the only artifact exercising the full story, so narrative breakage surfaces there, and data-level fixes (the potentially long rabbit hole) stay off the release train. **Port discipline (binding):** the draft consumes a local prototype module shaped to the agreed API (function name `example_same_stats_graphs`, same return dict, same baked attributes) so the WS-B port is a near-mechanical import swap that validates the shipped API. Composite-figure composition (former G5, e.g. spring row in or out) is decided here, where the figures standardize.
4. Parameter choices that the generator will later bake in (lognormal params, swaps budget, render kwargs) are recorded in the story repo's writeup so WS-A transcribes rather than re-derives.
5. Gary signs off on the standardized figures and the settled draft notebook (clears G1).

### Workstream A: Dataset generator + tests

**Status:** not started (gated on Gary's kickoff prompt, which is G1 plus the branch signal in one act, Amendment 5; anchors folded in; fresh branch off master, the branch-46 precondition long since cleared. G2 shape resolved; internal constants transcribed from the story module itself)
**Files:** `src/hiveplotlib/datasets/same_stats_graphs.py` (new; name per G7), `src/hiveplotlib/datasets/__init__.py` (star-import line), `tests/datasets_test.py` (new test block) or a mirrored new test module — follow the existing flat `tests/` convention.
**Port source and sanctioned adaptations (Amendment 5):** canonical source is `same_stats_graphs.py` at the story repo's committed `main` (record the commit hash in the Implementation log entry). The module is verified at the G2 shape: `example_same_stats_graphs(num_nodes=1000, swaps_per_edge=20, variants=None, seed=42) -> Dict[str, nx.Graph]`, canonical variant order, baked `degree_rank` / `degree_group` / `community`. Port near-verbatim, preserving exactly the per-variant rng contract (`np.random.default_rng([seed, VARIANT_NAMES.index(name)])`: a variant's graph is identical whether generated alone or with the others, which also makes the six-name canonical list load-bearing even though the story uses five). Apply exactly these adaptations, which are plan requirements the standalone scratch module deliberately lacks, not defects to halt on: (a) `ValueError` becomes the `hiveplotlib` exception per done-when 4, keeping the enumerating message; (b) module-top `import networkx` becomes the function-level import with the `[networkx]`-extra message per done-when 3 and the Feasibility audit; (c) the module docstring's story-repo/plan process references are dropped (self-contained docstring; keep the Chen et al. citation); (d) `VARIANT_NAMES`, `_steered_rewire`, and the score functions stay private surface (`__all__ = ["example_same_stats_graphs"]`).
**Done when:**

1. Generator exists with the G2-decided signature, docstring at 120-char wrap documenting the mechanism, the matched invariants, the baked node attributes, the internal constants (lognormal params, 3 planted communities), and citing Chen et al. (arXiv:1808.09913).
2. Returned dict uses the canonical narrative insertion order — `random`, `assortative`, `disassortative`, `core_periphery`, `stripes`, `community` — documented in the docstring and preserved under `variants` subsetting (insertion order is user-facing: the notebook builds the multi-panel figure via `graphs.items()`). (Amendment 1)
3. networkx imported inside the function with the helpful `[networkx]`-extra error; `import hiveplotlib.datasets` succeeds without networkx installed.
4. Tests (`@pytest.mark.networkx`) at small scale (e.g. `num_nodes<=100`, `swaps_per_edge<=3`) assert: identical sorted degree sequence across all variants; identical node and edge counts; assortativity sign separation between assortative and disassortative variants at default scale parameters scaled down; community variant's within-community edge fraction exceeds the random variant's; `variants` subset selection works, preserves canonical order, and returns graphs identical (same node and edge sets) to the corresponding graphs of a full run (the per-variant rng contract; Amendment 5); invalid variant name raises a `hiveplotlib` exception (from `src/hiveplotlib/exceptions/`) whose message enumerates the valid variant keys (no other discovery channel exists before generating — Amendment 1); baked attributes present with the documented names and dtypes; determinism under fixed seed. 100% coverage; added test runtime bounded (~seconds, not minutes).
5. Type hints (`Union`, not `|`), ruff format/check clean, `make ty` clean.
6. api-critic post-implementation review filled in this plan; any must-fix routed through amend-plan.

### Workstream B: Tutorial notebook (port-and-polish; reframed by Amendment 4)

**Status:** not started (gated: WS-A shipped; the draft is settled and its sign-off arrives with the kickoff prompt, Amendment 5. G4 resolved standalone; G5/G3 settled in WS-0)
**Files:** `examples/same_stats_different_graphs.ipynb` (new; filename per G7). Nothing under `docs/source/notebooks/` or `gallery_examples/` (auto-generated).
**Scope:** port the settled WS-0 draft from hiveplotlib-datasaurus and polish to hiveplotlib's shipped-notebook bar; not from-scratch authoring. The data import swaps from the local prototype module to `hiveplotlib.datasets` (near-mechanical by WS-0's port discipline; the swap itself validates the shipped API). Narrative or figure-composition problems found here are story problems and route back to WS-0 / amend-plan, not fixed ad hoc in the port. **Narrative source of truth (Amendment 5): `build_notebook.py` at the story repo's committed `main`.** The checked-in ipynb is a derived build artifact and was stale by exactly the Beat-3 caveat paragraph at amendment time; port from the builder's cells (or from an ipynb freshly rebuilt from it), never from a stale build. Port-polish strips the draft-status line in the title cell, the "once ported" phrasing in the install note, and every story-repo/process reference: the shipped notebook is self-contained, citing no wiki pages, plans, or story-repo paths.
**Done when:**

1. Four-beat ladder, carried over from the settled draft (Amendment 2; details in "Layout counterfactuals"; counts re-grounded to the settled 4+1 shape by Amendment 5: the ladder is `random`, `assortative`, `disassortative`, `stripes`; `community` is the twist; `core_periphery` appears nowhere in the notebook): (beat 1) the stats table foil — the five story variants shown identical on n, m, density, degree histogram (community sits in the table from the start; the twist re-reads it); (beat 2) standard layouts fail — spring as the foil, not the strawmen (random placement / arbitrary circle / spectral may appear as supporting evidence but not as the headline opponent), with the assortative-comet caveat owned honestly (spring cracks exactly that one variant); (beat 3) the degree-ordered circle gets halfway — identical geometry to the arbitrary circle, only the ordering meaningful, three of the four ladder variants separate (random stays the null); placement-encoding-data named as the single concession and as the hive plot's founding premise (Krzywinski "rational layout"); still blind to the community variant; (beat 4) hive plots complete it — explicit labeled partitions on the shared degree-rank scaffold (pixel-identical node positions made explicit), repeat axes for within- vs between-group traffic, swappable partition variables; then the twist — the community variant invisible under the degree partition, popping under the community partition, handing off to partition sweeps / `HivePlotMatrix`.
2. Hive plot construction uses the headline `HivePlot(graph=...)` idiom from "API usage examples" (no `networkx_to_nodes_edges` round-trip unless the prose has a reason to show the intermediate objects); multi-panel figures iterate `graphs.items()` and so inherit the canonical variant order. (Amendment 1)
3. Planted-communities honesty language per "The mechanism" section: planted, not detected; no detection result massaged to 3.
4. `#### Background` + `#### References` (arXiv:1808.09913 and the Datasaurus Dozen as the conceptual ancestor); install-extras (`[networkx]`) surfaced up front; PLAIN-DATA rule held (no GNN/ML framing).
5. House voice (no em-dashes, no AI filler); length disciplined against the closest tutorial sibling.
6. Runs end-to-end under `make test-nb` (warnings-as-errors) in acceptable time at the shipped generator scale.
7. editorial-critic and viz-critic post-impl both run; editorial-critic's Notebook review section filled above; no open must-fix (must-fix routes to amend-plan).
8. The Beat-3 ordering-vs-glyph paragraph survives the port intact in substance (Amendment 5): the degree-rank ordering, not the hive glyph, does the exposing (a reordered adjacency matrix would separate these structures too); the hive plot's contribution is applying that ordering cleanly and reproducibly, with labeled and repeat axes, at densities where a raw circle or matrix clogs; and the prose owns that the hive plot is handed the degree axis the spring foil never got. The hive-vs-force comparison stays illustrative, never a measured win. This is the settled disposition of the grounding run's uncontrolled-comparison caveat (`wiki/wiki/analyses/same-stats-different-graphs-grounding.md`); the notebook carries the argument in its own prose and cites no wiki pages.

### Workstream C: Docs + CHANGELOG registration

**Status:** not started (gated: WS-B shipped)
**Files:** `docs/source/notebooks/index.rst` (both the `nblinkgallery` block and the hidden `toctree`, same order; section placement: standalone per G4, landing in an existing section, likely `Hive Plot Examples`, unless Gary places it more prominently), `docs/source/conf.py` (`nbsphinx_thumbnails` entry), `docs/source/_static/<thumbnail>`, `CHANGELOG.rst` (one new-feature entry in the current unreleased WIP block, v0.29.0 as of Amendment 5, covering generator + notebook together; no separate entries for same-version refinements), API docs page for the new datasets module if the datasets API reference enumerates modules.
**Done when:**

1. Notebook registered in both `index.rst` blocks, same order; thumbnail entry + `_static/` asset in place.
2. CHANGELOG entry added to the current unreleased WIP block (v0.29.0 as of Amendment 5; the v0.28-era sequencing note against the conflict-validation and notebook-restructure plans is moot, both shipped in v0.28) (append, don't reflow).
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

### Amendment 5 (2026-07-06) — Kickoff readiness: port source pinned, anchors folded in, G1 kickoff semantics, core_periphery disposition

Delta source: Gary's ask (2026-07-06). The WS-A/B/C port will be kicked off by a prepared prompt Gary sends together with his branch signal; tie the kickoff to the story repo's current reality, which has matured well past this plan's WS-0 record. Evidence verified directly in `hiveplotlib-datasaurus` (read-only): committed `main` at `9596090` ("unreviewed pass"), exactly two uncommitted edits (`ANCHORS.md`: the 2026-07-04 disassortative correction + Liljeros reference; `build_notebook.py`: the Beat-3 ordering-vs-glyph caveat), the generator verified at the G2 shape by direct read, and the checked-in ipynb verified stale by exactly the caveat paragraph. All items are in-scope tweaks or fold-ins performed here; no workstream added, no specialist span changed.

1. **In-scope tweak (WS-A + WS-B briefs) — canonical port source is the story repo's committed `main`.** Gary commits the two pending edits when he sends the kickoff prompt; WS-A ports `same_stats_graphs.py`, WS-B ports the narrative from `build_notebook.py` (the ipynb is a derived build artifact, stale at amendment time; never port from a stale build). Rationale: sign-off must bind to an immutable ref (HEAD is currently literally named "unreviewed pass"), and the recorded hash makes the port auditable; working-tree-as-canonical was considered and rejected as unverifiable after the fact. The kickoff session records the story-repo hash in the Implementation log and halts if the port-relevant working tree differs from `HEAD`. WS-A's brief gains the "Port source and sanctioned adaptations" paragraph so the scratch module's deliberate standalone simplifications (generic `ValueError`, module-top networkx import, process references in the docstring) read as sanctioned port adaptations, not brief-vs-source confusion to halt on.
2. **Performed here — real-world anchors folded in** as the new "Real-world anchors" subsection under "The mechanism," transcribed from the corrected working-tree `ANCHORS.md` and verified against the file and its diff (not taken on faith from the task brief). Discharges WS-0 done-when 2 before WS-A dispatches, so the kickoff has no serialized first action. Carries the binding ladder-vs-twist figure-design principle for WS-B.
3. **In-scope tweak (WS-A) — core_periphery ships in the generator, documented and anchored, absent from the notebook.** This is the story repo's own recorded disposition ("the generator keeps the variant available; it just leaves the story," cut 2026-06-12). Rationale for shipping rather than dropping or hiding: (i) the every-shipped-variant-needs-an-anchor rule is satisfied, its anchor row being high-confidence; (ii) dropping it would renumber the per-variant rng streams (`[seed, canonical index]`) for `stripes` and `community`, changing their graphs and invalidating the exact figures G1 signs off, so a drop is not a mechanical port; (iii) in-generator-but-undocumented is incoherent, since the invalid-variant error message enumerates all six keys anyway. Gary gets a one-line veto in the kickoff prompt; silence ships it. WS-A done-when 4 gains the subset-identity assertion (a `variants` subset run returns graphs identical to the corresponding full-run graphs), pinning the rng contract this disposition leans on.
4. **In-scope tweak (Gates) — G1 kickoff semantics recorded.** Sending the prepared kickoff prompt with the branch signal *is* the G1 sign-off; the G1 entry now enumerates what the act covers (standardized figures, corrected anchors, caveat-bearing narrative, 4+1 story shape, internal constants) plus the commit precondition and the dirty-tree halt rule. Sign-off is a defined act, not a vibe.
5. **In-scope tweak (WS-B done-whens) — narrative inputs pinned to survive the port.** Done-when 1's counts re-grounded to the settled 4+1 shape (ladder `random` / `assortative` / `disassortative` / `stripes`; community as the twist; stats table carries all five story variants; beat 3 separates three of four); new done-when 8 requires the Beat-3 ordering-vs-glyph caveat and the illustrative-never-measured framing to survive the port (the settled disposition of the grounding run's uncontrolled-comparison caveat); Scope now names the port-polish strips (draft-status line, "once ported" install phrasing, all process references).
6. **In-scope tweak (current-harness wiring) — adversary post-impl obligation made visible.** The plan predates the cold-adversary feature; an "Adversary — post-impl reviews" pending block now sits with the other critic placeholders (mandatory per shipped workstream; with no `## Failure modes` rubric, the blind-first extract is the workstream block + done-whens + Holdouts, and auto-dispatch is not offerable, so per-workstream pauses stand).
7. **In-scope tweaks (staleness sweep, recorded):** Goal's "five-plus-one networks" re-grounded to six variants / 4+1 story; WS-C's CHANGELOG target moved from the shipped v0.28.0 to the current unreleased WIP block (v0.29.0 today); the branch-46 precondition noted cleared (46 merged, v0.28 shipped); "Prior ADRs" intro updated (ADRs 0001-0002 now exist; 0001 networkx-integration is pre-read for WS-A); "The mechanism" transcription source superseded by the story module; plan-header comment refreshed.
8. **Deferred follow-up — JOSS headline-figure composition.** Which variants the JOSS figure carries (the Newman triad; stripes in or out; the cut core_periphery) stays with `wiki/wiki/plans/joss-submission.md`, decided when that figure is built. This plan ships the notebook's 4+1 and the six-variant generator; nothing here pre-commits the JOSS figure.

Feasibility: no new entry point and no new attribute reads; the port source is verified at the G2 shape by direct read, and the draft's `HivePlot(graph=...)` / `hive_plot_viz(..., fig=..., ax=...)` idiom is empirically validated (the standardized figures were rendered with it against the current library). No fresh grill wave recommended: every item executes decisions already made and recorded (the 2026-06-12 cut, the settled anchors, Gary's own kickoff framing); the one new judgment (item 3) surfaces as a named default-with-veto line in the kickoff prompt Gary reads before sending.

## Holdouts

- Root-level `prototype_*.py` / `prototype_*.png` (untracked) and `data/datasaurus_prototypes/` (gitignored): scratch record of the prototyping sessions; no workstream edits or removes them in this repo. WS-0 copies them into hiveplotlib-datasaurus (read-only here).
- `examples/computing_graph_metrics.ipynb`, `finding_a_partition.ipynb`: owned by the restructure plan.

## Implementation log

Append-only. After each workstream completes, one line in the same turn.

_None yet._
