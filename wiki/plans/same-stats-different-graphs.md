# Plan: Same Stats, Different Graphs (matched-statistics network demo)

<!--
Hiveplotlib consumer plan. Formal library support for the Datasaurus-style
"identical graph statistics, different structure" demo prototyped 2026-06-10
(scratch work in data/datasaurus_prototypes/, gitignored; its README.md is the
mechanism reference). Prototype conclusions are strong-but-pending-review:
Gary has NOT yet reviewed the renders. See Gates.
-->

## Goal

Ship the "Same Stats, Different Graphs" demo as a first-class part of the library: a synthetic dataset generator in `hiveplotlib.datasets` producing five-plus-one networks with pixel-identical degree statistics (same n, m, density, full degree sequence) but instantly distinguishable hive plot signatures, and a tutorial notebook telling the three-act story (stats table foil, hairball foil, hive plots name the structures) plus the hidden-communities twist that motivates partition sweeps. This becomes a centerpiece of the "why choose hive plots" pitch. Prior art: Chen, Soni, Lu, Maciejewski, Kobourov, "Same Stats, Different Graphs" (GD 2018, arXiv:1808.09913), which makes the statistical point with force-directed drawings; we carry it through to hive plots.

## Alignment (grill)

```
Not yet run — recommended before dispatch: this plan carries seven open user
decisions (see Gates) and an unreviewed prototype premise. Run the grill-me
skill or knowingly skip; record each wave below. Route any resulting plan
change to amend-plan (rule 14).
```

## Gates (user decisions before dispatch)

All from the 2026-06-10 prototyping session. G1 and G2 block WS-A; G4 blocks WS-B scoping; the rest resolve at the named workstream.

- **G1 — Prototype review (blocks WS-A).** Gary reviews `data/datasaurus_prototypes/prototype_v4_composition.png` (five variants, hairballs vs hive plots) and `prototype_v5_community.png` (community twist 2x2). The visual story is validated by the session but not by Gary. No implementation until go.
- **G2 — API shape (blocks WS-A; api-critic planning mode feeds this).** Recommendation below: one generator returning `Dict[str, nx.Graph]` with hive-plot-ready node attributes baked in; rewiring engine and score functions private. Alternative on the table: `Dict[str, Tuple[NodeCollection, Edges]]` per the datasets library-native convention. Decide after the api-critic planning pass.
- **G3 — Stripes variant (resolve at WS-A or WS-B).** Least real-world structure; on the bubble per the prototype README. Recommendation: generator ships it (cheap, one score function, keeps the "five signatures" figure option open); the notebook decides whether to show it. Cutting it from the generator too is fine if Gary prefers a tighter surface.
- **G4 — Standalone tutorial vs. graph-metrics-series capstone (blocks WS-B).** Recommendation: standalone. The story is a "why hive plots at all" pitch, broader than the metrics series' "which lens reveals what" scope; and the series capstone slot ("Finding Surprising Connections," flagged HIGHEST RISK there for lack of exactly this kind of dataset) can later reuse this generator without this notebook occupying the slot. If Gary picks capstone instead, route through `graph-metrics-tutorial-series.md` via amend-plan; do not duplicate its slot here.
- **G5 — Hairball row in the shipped figure (resolve at WS-B authoring).** Recommendation: keep it. Four indistinguishable spring layouts next to five distinguishable hive plots is the strongest single image; the stats table alone makes the weaker, purely numerical foil.
- **G6 — Animated morph (deferred by default).** Pixel-identical node positions support an edges-only morph between variants. Deferred follow-up; not in any workstream.
- **G7 — Names.** Generator function, notebook filename/title, variant key names. Recommendations in the Naming audit; final call is Gary's, latest at each workstream's dispatch.

## Prior ADRs / design docs

No ADRs exist yet (`wiki/wiki/adr/` absent; `scaling-large-networks.md` reserves first-promotion candidacy). Binding records are prior plans:

- `wiki/wiki/plans/graph-metrics-tutorial-series.md` — its Phase-3 capstone is flagged HIGHEST RISK for lack of a plain-data "surprising structure" dataset; this generator is a direct candidate feed (gate G4). This plan inherits its series conventions where the notebook is concerned: validate-then-commit dataset gate, naming audit, docs-registration mechanics, PLAIN-DATA rule (no GNN/ML framing), gallery-vs-tutorial split.
- `wiki/wiki/plans/graph-metrics-notebook-restructure.md` — in-flight; owns `examples/computing_graph_metrics.ipynb` (currently modified in the working tree) and `finding_a_partition.ipynb`. This plan does not touch either; coordinate docs-index and CHANGELOG edits. Its binding rule ">3 communities → HivePlotMatrix; never tune community resolution down to fake ≤3" is addressed under "Planted-communities honesty" below.
- `wiki/wiki/plans/networkx-metric-expansion-and-backend-refactor.md` — community-label contract (label 0 = largest, descending size) binds *detection outputs*. The generator's planted labels are ground truth, not detection output; emitted as strings `"A"/"B"/"C"` to make that unmistakable (see Naming audit).
- `wiki/wiki/plans/graph-metric-conflict-validation.md` — in-flight on this same branch (46-more-streamlined-networkx-usage-and-support), same v0.28.0 train; sequence CHANGELOG/notebook edits around it. Whether this work lands on this branch or a fresh one off master is Gary's call at dispatch.
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

### Planted-communities honesty

The restructure plan's rule forbids tuning *detection* resolution down to fake ≤3 communities. The generator *plants* exactly 3 communities by construction; that is a synthetic design choice, not a detection result massaged to fit. The notebook must say so explicitly ("planted, not detected"), and if it runs community detection on the variant it must show what detection actually finds rather than asserting 3. The twist figure (degree partition vs. community partition) is the honest motivation for partition sweeps / `HivePlotMatrix`.

## Default justifications

For the recommended generator signature (final shape pending G2 / api-critic):

- `num_nodes=1000`: the validated render scale; large enough that edge-density signatures read as texture at notebook size, small enough to generate in seconds and stay within `make test-nb` budgets.
- `swaps_per_edge=20`: prototype-validated convergence (assortative reaches +0.87, community reaches 93% within-community) without runaway runtime.
- `seed=42`: reproduces the exact renders Gary reviews at G1, so the shipped notebook figures match the validated prototypes.
- `variants=None` → generate all variants: the demo's point is the side-by-side; callers wanting a subset pass an explicit list (also how tests keep runtime down).
- Lognormal parameters (mean 1.6, sigma 0.55, min degree 2) and the 3-community planting are internal constants, not parameters: the generator exists to tell one validated story, and every knob exposed is a knob that can break the matched-statistics invariant or the render recipe. Documented in the docstring instead.

## Naming audit

Checked against networkx vocabulary (the dominant adjacent ecosystem) and the `hiveplotlib.datasets` conventions (synthetic generators use `example_*` prefix, `seed` param).

- **Function: `example_same_stats_networks` (recommended).** Keeps the `example_*` synthetic-generator convention and echoes the prior-art title users may recognize. Alternatives: `same_stats_different_graphs()` (literal citation, but breaks the `example_*` convention that separates synthetic from real datasets); `example_matched_degree_networks` (accurate, loses the citation hook). Final call: G7.
- **New parameters:** `num_nodes` (matches `example_node_data` et al.; not networkx's `n`, since the datasets module's own convention wins here), `swaps_per_edge`, `variants`, `seed`. All checked against networkx vocab: `seed` matches nx generators; `swaps_per_edge` echoes `nx.double_edge_swap`'s swap framing.
- **Variant keys (dict keys, user-facing):** `"random"`, `"assortative"`, `"disassortative"`, `"core_periphery"`, `"stripes"`, `"community"`. Snake_case for the multiword key (matches nx's `degree_assortativity_coefficient` vocabulary; the prototype's hyphenated `"core-periphery"` is amended → `core_periphery` because every other key is a valid identifier).
- **Baked node attributes:** `degree_rank` (not the prototype's `deg_rank`; no abbreviation in shipped surface), `degree_group` (values `"low"/"med"/"high"`, matching the validated render recipe), `community` (values `"A"/"B"/"C"`; strings, not ints, so they cannot be mistaken for detection-output labels bound by the label-0-largest contract).
- **Notebook (provisional, finalized at WS-B per G7):** `examples/same_stats_different_graphs.ipynb`, title `### Same Stats, Different Graphs`. The pun is literal for networks and cites the prior art. No collision in `examples/`.
- **Prose-only terms:** "degree-preserving rewiring," "double-edge swap," "planted communities," "configuration model" — all standard networkx/network-science vocabulary.

## API usage examples

### Proposed (from planner / Orchestrator)

Recommended shape: `example_same_stats_networks(...) -> Dict[str, nx.Graph]`, every graph carrying `degree_rank`, `degree_group`, and `community` node attributes (identical across variants, since rewiring preserves degrees and the planting is by node id). Headline idiom (per the api-critic planning pass, Amendment 1): `HivePlot(graph=...)` (keyword-only, `src/hiveplotlib/hiveplot.py:2084`), which routes through the converter and carries node attributes into `nodes.data` (attribute flow verified at `src/hiveplotlib/converters.py:37-47`), so the attributes feed `partition_variable` / `sorting_variables` with no undocumented convention. The two-step `networkx_to_nodes_edges()` → `HivePlot(nodes=..., edges=...)` path remains available for users who want the intermediate `NodeCollection` / `Edges` objects, but is not the tutorial shape.

```python
# Example 1: generate the variants and verify the matched-statistics foil
# Example data: (generator is the data source)
from hiveplotlib.datasets import example_same_stats_networks

graphs = example_same_stats_networks(seed=42)

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
from hiveplotlib.datasets import example_same_stats_networks
from hiveplotlib.viz import hive_plot_viz

graphs = example_same_stats_networks(seed=42)

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
from hiveplotlib.datasets import example_same_stats_networks
from hiveplotlib.viz import hive_plot_viz

graphs = example_same_stats_networks(seed=42, variants=["random", "community"])

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

Alternative shape for api-critic to weigh (the datasets convention says synthetic generators return library-native types): `Dict[str, Tuple[NodeCollection, Edges]]`. Trade-off: the story's acts 1-2 (stats table, spring-layout hairballs) live in networkx, and `HivePlot` accepts the converted pair in one documented call either way; native-first would instead make users round-trip through `nodes_edges_to_networkx()` for the foil acts. Recommendation stands at nx-first; the score functions and the rewiring engine stay private (`_steered_rewire`, module-level `_SCORE_FUNCTIONS`) in either shape.

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

- Net-new entry point `example_same_stats_networks`: output traces to `nx.Graph`, consumed by the documented `HivePlot(graph=...)` path (keyword-only, `hiveplot.py:2084`), which routes through `networkx_to_nodes_edges()`; node attributes become `NodeCollection.data` columns (verified, `converters.py:37-47`), which are exactly what `partition_variable` / `sorting_variables` / `axes_order` consume. `graph_directed` / `graph_multigraph` infer correctly from a plain `nx.Graph`. No undocumented convention needed. (Headline idiom set by Amendment 1; the converter-path attribute trace stands as the verification.)
- Optional-dependency wiring: `hiveplotlib/datasets/__init__.py` star-imports every module, so the new module **must not** raise at import time (unlike `converters.py`, whose module-level guard is fine because nothing star-imports it into a non-networkx path). The networkx import goes *inside* the generator function, wrapped with the same helpful message naming the `[networkx]` extra (`converters.py:16` is the message template), `# pragma: no cover` on the ImportError branch matching the existing pattern. This keeps `from hiveplotlib.datasets import example_hive_plot` working without networkx.
- Performance: the rewiring loop is pure Python, ~20 x m iterations per variant (~60k at n=1000); seconds per variant, acceptable for the notebook under `make test-nb`. Tests use small `num_nodes` and `swaps_per_edge` (see WS-A done-when).

## Notebook review

Net-new notebook (no existing page's scope drifts). Class: `HivePlot` primary; genre: tutorial (narrative arc, `#### Background`, `#### References` citing arXiv:1808.09913); dataset: the new synthetic generator only. Flag for sign-off at authoring: if the community-twist act is built as an actual `HivePlotMatrix` partition sweep rather than v5's 2x2 of `HivePlot`s, the closer leans HPM — acceptable as a closing hand-off, but the page's primary subject must stay `HivePlot`; editorial-critic checks this.

```
Pending — invoke editorial-critic in post-implementation mode after Workstream B ships.
```

## Workstreams

WS-A blocks on G1 + G2. WS-B blocks on WS-A and G4. WS-C blocks on WS-B.

### Workstream A: Dataset generator + tests

**Status:** not started (gated: G1 prototype review, G2 API shape via api-critic planning pass)
**Files:** `src/hiveplotlib/datasets/same_stats_networks.py` (new; module name final at G7), `src/hiveplotlib/datasets/__init__.py` (star-import line), `tests/datasets_test.py` (new test block) or a mirrored new test module — follow the existing flat `tests/` convention.
**Done when:**

1. Generator exists with the G2-decided signature, docstring at 120-char wrap documenting the mechanism, the matched invariants, the baked node attributes, the internal constants (lognormal params, 3 planted communities), and citing Chen et al. (arXiv:1808.09913).
2. Returned dict uses the canonical narrative insertion order — `random`, `assortative`, `disassortative`, `core_periphery`, `stripes`, `community` — documented in the docstring and preserved under `variants` subsetting (insertion order is user-facing: the notebook builds the multi-panel figure via `graphs.items()`). (Amendment 1)
3. networkx imported inside the function with the helpful `[networkx]`-extra error; `import hiveplotlib.datasets` succeeds without networkx installed.
4. Tests (`@pytest.mark.networkx`) at small scale (e.g. `num_nodes<=100`, `swaps_per_edge<=3`) assert: identical sorted degree sequence across all variants; identical node and edge counts; assortativity sign separation between assortative and disassortative variants at default scale parameters scaled down; community variant's within-community edge fraction exceeds the random variant's; `variants` subset selection works and preserves canonical order; invalid variant name raises a `hiveplotlib` exception (from `src/hiveplotlib/exceptions/`) whose message enumerates the valid variant keys (no other discovery channel exists before generating — Amendment 1); baked attributes present with the documented names and dtypes; determinism under fixed seed. 100% coverage; added test runtime bounded (~seconds, not minutes).
5. Type hints (`Union`, not `|`), ruff format/check clean, `make ty` clean.
6. api-critic post-implementation review filled in this plan; any must-fix routed through amend-plan.

### Workstream B: Tutorial notebook

**Status:** not started (gated: WS-A shipped, G4 standalone-vs-capstone, G5 hairball row, G3 stripes-in-figure)
**Files:** `examples/same_stats_different_graphs.ipynb` (new; filename final at authoring per G7). Nothing under `docs/source/notebooks/` or `gallery_examples/` (auto-generated).
**Done when:**

1. Three-act arc: (act 1) the stats table foil — variants shown identical on n, m, density, degree histogram; (act 2) the hairball foil — spring layouts mostly indistinguishable (kept or cut per G5); (act 3) hive plots on the shared degree-rank scaffold name each structure, with the pixel-identical-node-positions point made explicitly; then the twist — the community variant invisible under the degree partition, popping under the community partition, handing off to partition sweeps / `HivePlotMatrix`.
2. Hive plot construction uses the headline `HivePlot(graph=...)` idiom from "API usage examples" (no `networkx_to_nodes_edges` round-trip unless the prose has a reason to show the intermediate objects); multi-panel figures iterate `graphs.items()` and so inherit the canonical variant order. (Amendment 1)
3. Planted-communities honesty language per "The mechanism" section: planted, not detected; no detection result massaged to 3.
4. `#### Background` + `#### References` (arXiv:1808.09913 and the Datasaurus Dozen as the conceptual ancestor); install-extras (`[networkx]`) surfaced up front; PLAIN-DATA rule held (no GNN/ML framing).
5. House voice (no em-dashes, no AI filler); length disciplined against the closest tutorial sibling.
6. Runs end-to-end under `make test-nb` (warnings-as-errors) in acceptable time at the shipped generator scale.
7. editorial-critic and viz-critic post-impl both run; editorial-critic's Notebook review section filled above; no open must-fix (must-fix routes to amend-plan).

### Workstream C: Docs + CHANGELOG registration

**Status:** not started (gated: WS-B shipped)
**Files:** `docs/source/notebooks/index.rst` (both the `nblinkgallery` block and the hidden `toctree`, same order; section placement depends on G4 — standalone lands in an existing section, likely `Hive Plot Examples`, unless Gary places it more prominently), `docs/source/conf.py` (`nbsphinx_thumbnails` entry), `docs/source/_static/<thumbnail>`, `CHANGELOG.rst` (one new-feature entry in the v0.28.0 WIP block covering generator + notebook together; no separate entries for same-version refinements), API docs page for the new datasets module if the datasets API reference enumerates modules.
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

## Holdouts

- Root-level `prototype_*.py` / `prototype_*.png` (untracked) and `data/datasaurus_prototypes/` (gitignored): scratch record of the prototyping session; no workstream edits or removes them.
- `examples/computing_graph_metrics.ipynb`, `finding_a_partition.ipynb`: owned by the restructure plan.

## Implementation log

Append-only. After each workstream completes, one line in the same turn.

_None yet._
