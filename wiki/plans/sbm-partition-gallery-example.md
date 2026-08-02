# Plan: SBM Partition Structure Gallery Example

<!--
Working plan (wiki submodule). Concise per mental-model rule 17 (plans shape).
Single gallery notebook: keep proportionate. Defers to the
hiveplotlib-gallery-notebook skill for house style.
-->

## Goal

Ship one **gallery example notebook** that demonstrates partition / community structure on a **Stochastic Block Model (SBM)** with known, controllable ground truth. It is the clean, universally-legible companion to the `finding_a_partition.ipynb` tutorial (T1, Les Misérables partition *discovery*, currently on MR !39): T1 is the messy real narrative with no ground truth; this example is the opposite pole. The maintainer builds the block structure with `networkx.stochastic_block_model`, so a reader with zero domain knowledge can watch the planted truth appear on the axes, watch a detector (`louvain_communities`) recover it (nodes colored by *detected* community over *planted*-block axes, so recovery reads as color-purity per axis, per the A1 encoding), and watch both break as inter-block edge density rises.

The notebook's **spine is the density sweep** (A5): a single static hive plot alone cannot show that the partition *generated* the edges rather than being assigned post hoc, so the reason-to-exist is the controllable-structure sequence, dialing `p_out` and watching structure change. The clean figure is the sweep's **low-`p_out` anchor**, not a standalone hero. The payoff no real dataset can give: ground truth plus controllability plus a visible detectability threshold. This is an *application* of already-shipped functionality; there is no API surface change.

Brief-mode gate: knowingly skipped (single gallery notebook off a settled maintainer conversation, 2026-07-07; the design points below are pinned, so plan-shaping choices are not underdetermined).

## Alignment (grill)

Grill knowingly skipped per maintainer autonomy directive 2026-07-07: the maintainer delegated this notebook autonomously and reviews at the end. The adversary still runs cold in planning mode (mandatory, rule 18) and post-impl. Per the orchestrator playbook, with the grill skipped the adversary's planning challenge does not get fought in a grill, so it routed to an amend-plan pass for explicit disposition before dispatch. **That disposition is complete** (Amendment 1, 2026-07-07): all six items adopted, folded into the sections below. The plan is cleared for dispatch to the Workstream A design gate.

```
Status: grill knowingly skipped (maintainer autonomy directive, 2026-07-07); adversary planning challenge disposed via amend-plan (Amendment 1, 2026-07-07)
Open decisions: none (design points pinned; adversary challenge A1-A6 all adopted)
```

## Failure modes

Not warranted — grill knowingly skipped (see Alignment). The adversary's cold planning read supplies the adversarial rubric this plan is held to; its six items are disposed in Amendment 1 (all adopted). Its post-impl pass attacks the shipped notebook against the (now amended) done-whens and the "Section-worth / no-duplication" bar below. Implementers may still append a mode here if they hit one mid-build.

## Prior ADRs / design docs

- `wiki/wiki/adr/0001-networkx-integration.md` — the graph-metric integration this notebook stands on (community-detection metric strings, `graph`/`nodes`+`edges` init paths). Relevant as the shipped substrate; no decision reopened.

Prior wiki *thinking* to point the reader at (do not create or edit these pages here):

- `wiki/wiki/analyses/gnn-heterogeneity-hive-plots.md` and `wiki/wiki/analyses/spectral-hive-plots.md` — SBM-like planted structure with controllable inter-cluster overlap and color-by-recovery. Both are research-proposal / prototype territory (GNN model diagnosis; spectral decomposition). **The richer SBM idea (plant *surprising* cross-block edges, color nodes/edges by detector-vs-truth recovery) is reserved for the future capstone of the tutorial series, not this gallery example.** This example is the simpler clean-reference cousin: planted blocks, a plain detector recovery, a density sweep, nothing bespoke. De-conflict, do not merge.

## Patterns this replaces

None, this is a net new addition. It does not restructure or supersede any existing notebook; `computing_graph_metrics.ipynb`, `setting_partition_variable.ipynb`, `create_partition_variable.ipynb`, and `hpm_from_partition.ipynb` all stay as-is. The only edit to an existing artifact is a one-line forward cross-link added to `finding_a_partition.ipynb` (T1), gated on sequencing (see Workstream A done-whens).

## Section-worth / no-duplication audit

The load-bearing audit. Verdict: **earns its place.** The distinct payoff is **ground truth + controllability + watch-it-break**, which no shipped notebook owns. Inspected against the near neighbors:

- `computing_graph_metrics.ipynb` (Karate Club + `example_hive_plot`): teaches the community-detection *mechanic* (how to request `node_graph_metrics`, discretize a metric into a partition, color edges by an edge metric). It never uses a generative block model, has no ground-truth partition to recover, runs no detector-vs-truth comparison, and shows no detectability sweep. Its partition-by-metric example bins a *continuous* node metric (`degree`) into `low/medium/high`; there is no planted community the method could recover or fail to recover.
- `setting_partition_variable.ipynb` / `create_partition_variable.ipynb`: partition *mechanics* on the `example_*` datasets. Those datasets do carry an A/B/C partition, but it is a **numeric-cutoff** partition (bin the continuous `low` column), not a network-generative block structure. Crucially, in those notebooks the partition does not generate the edges; here the partition **is** what generated the edges, which is the entire point and the source of the recover-then-break narrative.
- `hpm_from_partition.ipynb`: HivePlotMatrix construction from a partition; a different class and a different question (matrix layout), not single-hive-plot community legibility.

The distinctness the notebook must make explicit in prose (so it does not read as "another A/B/C partition demo"): the partition is the *generator* of the edge structure, the ground truth is *known by construction*, and the reader watches a detector recover it and then fail as inter-block density rises. **Crucially (A5): that distinctness is only *visible* in the sweep, not in any single frame.** A reader looking at one static hive plot cannot tell "these blocks generated these edges" from "these nodes were assigned to A/B/C and happen to cluster" (exactly what `setting_partition_variable` / `create_partition_variable` already show on `example_*`). The reason-to-exist becomes visible only when structure changes as `p_out` is dialed. So the sweep **is** the spine; the clean figure is its low-`p_out` anchor. If a draft leans on "how to call `louvain_communities`" as its spine, or lets the clean anchor frame stand alone as the headline with the sweep as a footnote, it has collapsed into `computing_graph_metrics` / partition-notebook territory and the editorial-critic should reject it.

Nearest-neighbor collision to actively avoid: the `example_*` A/B/C look. Use the SBM's own integer `block` labels (rename to legible axis names) and a visibly different dataset shape (block sizes, a clean-vs-degrading pair) so the figure does not read as a re-skin of the existing partition notebooks.

## Default justifications

No new library defaults (no API change). Notebook-level parameter choices, justified so the design/parameter gate has a target and can be adjusted:

- 2-3 blocks: keeps the sweep's clean anchor frame a single clean hive plot within the 2-3 axis rule (>3 blocks is an HPM, gestured at but not the demonstration).
- Seeded SBM generation (`seed=` on `stochastic_block_model`, and `seed=` on `louvain_communities`): per-notebook determinism, matching the series convention and keeping `make test-nb` reproducible.
- A small node count (order tens per block): universal legibility and matplotlib-sized rendering; large enough to read block bundles, small enough that individual edges stay visible.
- Inter-block probability sweep as **at most three static frames** (A4 cap: three is the plan's own snippet, made the ceiling, not an illustration; not an animation, not a study): shows the detectability threshold as within-block bundles blur into cross-block ones, kept gallery-sized. Each frame must differ *legibly* from its neighbor at the gallery's render size; a frame that doesn't earn its vertical space is dropped, and three legibly-distinct frames that can't be found means the sweep is dropped rather than padded.

Concrete starting parameters are the design/parameter gate's job to lock (Workstream A); the above are the constraints it optimizes within.

## Naming audit

Check user-facing names against gallery-sibling conventions (`examples/` filenames + the H3-title rows in `docs/source/notebooks/index.rst`).

- Notebook H1 title (gallery is H1 per the gallery skill; the "H3" the brief mentions is the tutorial convention, so gallery here is H1): **`# Partition Structure with a Stochastic Block Model`**. Weighed against siblings — sibling gallery titles are noun phrases naming the feature (`Setting a Partition Variable`, `Computing Graph Metrics`, `Creating Hive Plots from NetworkX`). "Partition Structure with a Stochastic Block Model" is a noun phrase, names the demonstrated thing (partition structure), and names the mechanism (SBM) the way `Creating Hive Plots from NetworkX` names its mechanism. Preferred over `Stochastic Block Model` alone (too bare, reads as a data-source page not a hive-plot page) and over `Partition Structure with Known Ground Truth` (accurate but hides the recognizable SBM keyword a searching reader types).
- Filename: **`partition_structure_sbm.ipynb`**. Snake_case, matches sibling shape (`setting_partition_variable.ipynb`, `computing_graph_metrics.ipynb`). Preferred over `stochastic_block_model.ipynb` (reads as a data-source/backend page, mirroring the `bokeh.ipynb` backend-file pattern, which miscategorizes it) and over `sbm_partition.ipynb` (abbreviation-first is unlike every sibling). `partition_structure_sbm` leads with the hive-plot concept and tails with the mechanism, matching how the title reads.
- Prose-only terms: *stochastic block model* / *SBM*, *ground truth*, *planted partition* / *planted blocks*, *inter-block* / *within-block*, *detectability threshold*, *community detection*. Standard network-science vocabulary; italicize on first appearance per the gallery skill.

Recommendation locked pending the design gate only insofar as the figure content may nudge the title; the filename is final. The dispatching session should treat title + filename as the naming decision of record unless the editorial-critic proposes better.

## API usage examples

No user-facing API change; all snippets are runnable on **shipped surface only**. The gallery skill prefers `HivePlot(graph=...)` over the two-step converter; the one place the two-step converter is warranted here is when a detected-label column must be inspected as a DataFrame column before plotting, which mirrors `computing_graph_metrics`'s "Using a Computed Metric as a Partition Variable" precedent. Every idiom shown below is exercised by a shipped notebook, so each snippet has a runnability witness:

- `graph=` init + `node_graph_metrics` + `node_graph_metric_kwargs`: `computing_graph_metrics.ipynb`.
- `repeat_axes=True` at 3 blocks draws each inter-block pair exactly once (no de-duplication needed; see the design-gate correction in Amendment 2). The 2-group near-neighbor `computing_graph_metrics.ipynb:145` calls `reset_edges` because a *2-block* ring draws the single inter-block set twice; that doubling does not occur at 3 blocks (the ring geometry places 0↔1 as `0_repeat→1`, 1↔2 as `1_repeat→2`, 0↔2 as `0→2_repeat`, one curve per pair). Axis-id spelling for reference (base axes are ints `0,1,2`; repeats are strings `"0_repeat"` etc.), kept in case a reader adapts the notebook to 2 blocks, where the de-dup *would* be needed.
- coloring nodes by a data column via `node_kwargs={"c": "<column>", ...}`: `visualizing_node_metadata.ipynb:288`. That witness colors by a continuous column with a `cmap`; here the same idiom colors by the **categorical** `louvain_communities` label column (integer community labels map through the cmap to distinct per-community colors), which is the A1 encoding. **The categorical color needs a pinned normalization** (`vmin=0, vmax=9`); see the A1-completeness correction in Amendment 2.

**A1 correction applied below:** the design colors nodes by the *detected* community (`louvain_communities`) while the axes stay the *planted* `block` (ground truth). There is no `detected == truth` per-node flag (that would need a label-alignment step this notebook must not carry). Recovery reads as **color-purity per axis**: good recovery => each planted-block axis is near-monochromatic; breakdown => colors scramble across axes. **The color must pin `vmin=0, vmax=9`** in `node_kwargs` (Amendment 2): `node_viz` draws a separate `ax.scatter()` per axis and matplotlib normalizes `c` per call, so a single-community axis with no pinned normalization self-normalizes to one colormap point and every axis renders the same color, making the whole encoding invisible.

**A2 correction (superseded — see Amendment 2):** the plan previously called for `reset_edges` de-duplication on every snippet. The design/parameter gate rendered and verified the locked 3-block SBM: with `repeat_axes=True` the total edge-curves drawn equals `graph.number_of_edges()` exactly, i.e. **there is no doubling at 3 blocks and no de-dup is needed**. Shipping a `reset_edges` de-dup loop at 3 blocks would delete a third of the inter-block signal (or `KeyError`). The snippets below draw the inter-block pairs directly; the A2 *intent* (draw the cross-block signal once, cleanly) is satisfied for free.

```python
# Example 1: build an SBM with a planted 3-block partition, plot partitioned by the true block
# Example data:
import networkx as nx
from hiveplotlib import HivePlot

sizes = [20, 20, 20]
p_in, p_out = 0.5, 0.02
probs = [
    [p_in, p_out, p_out],
    [p_out, p_in, p_out],
    [p_out, p_out, p_in],
]
g = nx.stochastic_block_model(
    sizes, probs, seed=0
)  # plants an integer `block` node attribute

# Call site:
# the planted `block` attribute rides onto the node data as a column and is used directly as the partition.
# At 3 blocks the repeat-axis ring draws each inter-block pair exactly ONCE (0<->1 as 0_repeat->1,
# 1<->2 as 1_repeat->2, 0<->2 as 0->2_repeat); no reset_edges de-dup is needed and none is correct
# (the doubling reset_edges corrects is a 2-block phenomenon; see Amendment 2).
hp = HivePlot(
    graph=g,
    partition_variable="block",
    sorting_variables="degree",
    node_graph_metrics="degree",
    repeat_axes=True,
)
fig, ax = hp.plot()
```

```python
# Example 2: run the detector on the SAME graph; axes stay the planted truth, node COLOR is the detected community
# Example data:
# reuse `g` from Example 1 (the SBM with the planted `block` attribute)

# Call site:
# louvain_communities is a shipped node-metric string returning an integer community label per node;
# seed makes it deterministic. NO detected-vs-truth flag: axes are TRUTH, node color is DETECTED.
# Two-tone edges (within-block vs cross-block) are set at CONSTRUCTION via the edge-kwarg hierarchy,
# NOT passed to .plot(). repeat_*/non_repeat_* keys distinguish intra-block (repeat-axis) curves from
# inter-block ones.
hp_detected = HivePlot(
    graph=g,
    partition_variable="block",  # axes are the ground-truth blocks
    sorting_variables="degree",
    node_graph_metrics=["degree", "louvain_communities"],
    node_graph_metric_kwargs={"louvain_communities": {"seed": 0}},
    repeat_axes=True,
    repeat_edge_kwargs={
        "color": "lightgray"
    },  # within-block bundles, set at construction
    non_repeat_edge_kwargs={
        "color": "firebrick"
    },  # cross-block bundles, set at construction
)
# color each node by its DETECTED community (a categorical column); good recovery => each planted-block
# axis is near-monochromatic, breakdown => colors scramble across axes. No label alignment needed.
# vmin/vmax MUST be pinned: node_viz scatters each axis separately and matplotlib normalizes `c` per
# scatter call, so a constant-community axis with no pinned range self-normalizes to one colormap point
# and every axis renders the same color. tab10 has 10 slots => vmin=0, vmax=9. Do NOT pass `zorder`
# in node_kwargs (viz hardcodes zorder=5; passing it raises TypeError).
node_kwargs = {
    "c": "louvain_communities",
    "cmap": "tab10",
    "vmin": 0,
    "vmax": 9,
    "s": 50,
    "edgecolor": "black",
}
fig, ax = hp_detected.plot(node_kwargs=node_kwargs)
```

```python
# Example 3: the detectability sweep — up to THREE static frames at rising inter-block density (A4 cap)
# Example data:
sizes = [20, 20, 20]

# Call site:
# Keep seed=0: the threshold frame is phase-transition-sensitive and the seed-0 appearance
# (partial scramble on 2 of 3 axes at the middle p_out) is what the gate validated. Each frame is
# its own full-size figure, NOT a subplots grid.
for p_out in (
    0.01,
    0.05,
    0.12,
):  # within-block bundles blur into cross-block ones as this rises
    probs = [
        [0.5, p_out, p_out],
        [p_out, 0.5, p_out],
        [p_out, p_out, 0.5],
    ]
    g_sweep = nx.stochastic_block_model(sizes, probs, seed=0)
    hp_sweep = HivePlot(
        graph=g_sweep,
        partition_variable="block",
        sorting_variables="degree",
        node_graph_metrics=["degree", "louvain_communities"],
        node_graph_metric_kwargs={"louvain_communities": {"seed": 0}},
        repeat_axes=True,
    )
    # no reset_edges de-dup at 3 blocks (each inter-block pair already draws once)
    fig, ax = hp_sweep.plot(
        node_kwargs={"c": "louvain_communities", "cmap": "tab10", "vmin": 0, "vmax": 9}
    )
```

### API Critic's take (planning mode)

N/A — no user-facing API surface. This section is retained only to record the runnability witnesses: `computing_graph_metrics.ipynb` for the `graph=` init and the `node_graph_metrics` + `node_graph_metric_kwargs` idioms; the two-tone edge styling via `repeat_edge_kwargs` / `non_repeat_edge_kwargs` set at `HivePlot(...)` construction (the edge-kwarg hierarchy in CLAUDE.md); `visualizing_node_metadata.ipynb:288` for coloring nodes by a data column (`node_kwargs={"c": "<column>", ...}`), the mechanism the A1 encoding reuses on the categorical `louvain_communities` column (with a pinned `vmin`/`vmax`, per Amendment 2). No `reset_edges` de-duplication is used: at the locked 3-block SBM each inter-block pair draws once (Amendment 2 supersedes the earlier A2 de-dup call). api-critic is not invoked on this plan (no API change), per the brief and CLAUDE.md's post-impl-critic guidance.

### API Critic — post-implementation review

No API surface change — api-critic is N/A for this plan (application of shipped functionality). The gate battery on Workstream A is editorial-critic + viz-critic + adversary + qa-engineer.

## Notebook review

Pending — invoke editorial-critic in post-implementation mode after Workstream A ships. It owns gallery-genre fit and the section-worth bar above (right notebook, one-dataset discipline, class scope stays on the single hive plot, no drift into HPM as the hero).

## Viz review

Pending — invoke viz-critic in post-implementation mode after Workstream A ships. Figures (the sweep is the spine, per A5): the clean single hive plot as the sweep's low-`p_out` anchor (partitioned by true `block`, nodes colored by detected `louvain_communities` per A1), and the up-to-three rising-density sweep frames (A4 cap). Polish budget: instructional (gallery reference figures), against the viz-quality-bar skill.

## Adversary review

### Adversary's challenge (planning mode)

```
Status: challenge (6 items)
Plan reviewed: wiki/wiki/plans/sbm-partition-gallery-example.md (cold, no grill — sole planning-mode pass)
Items:
  - [must-fix] The recovered-vs-truth agreement flag needs a label-alignment step the plan never names — at API usage examples (Example 2) / done-whens
    Rubric: no rubric yet — grill skipped
    Push: See A1 below. Either add the alignment step to the design (and prove it on shipped surface), or cut the detector-recovery beat to a "detector finds the same blocks" statement with no per-node truth-match coloring. Decide before dispatch; this is the beat most likely to halt the implementer under rule 9.
  - [must-fix] Every code snippet sets repeat_axes=True but omits reset_edges; on 2-3 blocks each inter-block bundle renders twice — at API usage examples (all three examples)
    Rubric: no rubric yet — grill skipped
    Push: See A2. The near-neighbor (computing_graph_metrics) calls reset_edges precisely to drop this redundant set on 2 groups; on 3 blocks there are 3 redundant pairs. Either the notebook drops them (and the design gate confirms the de-duplicated figure still reads) or the plan states, with a rendered witness, that the doubled bundles are acceptable. As written the API-usage witnesses model an idiom the near-neighbor explicitly corrects.
  - [worth-discussing] The "watch it break" payoff is asserted, never shown to be achievable at gallery size before committing to build — at Goal / Default justifications
    Rubric: no rubric yet — grill skipped
    Push: See A3. The whole section-worth verdict rests on the density sweep reading as a legible break on a hive plot at ~20 nodes/block. That is a design-gate output, but the plan commits to the deliverable before the gate proves it. Add an explicit gate exit: if no (block-count, p_in, p_out, node-count) combination makes the break legible on a static matplotlib hive plot, that is a rule-9 surface-back to the maintainer, not a silent fallback to more frames / bigger figures / a different backend.
  - [worth-discussing] "Handful of static frames" has no number and no legibility gate; this is where a gallery page balloons — at Default justifications / done-whens
    Rubric: no rubric yet — grill skipped
    Push: See A4. Pin an upper bound (three frames is the plan's own snippet; make that the cap, not an example) and require each frame to differ legibly from its neighbor at thumbnail-adjacent render size, or drop a frame. "A few frames of a sweep" with no cap is the classic gallery-length regression.
  - [worth-discussing] Section-worth verdict leans on "the partition generated the edges," which is a prose claim the figure cannot show — at Section-worth / no-duplication audit
    Rubric: no rubric yet — grill skipped
    Push: See A5. A reader looking at the hero figure cannot see that the blocks generated the edges vs. were assigned post hoc (the example_* A/B/C partition also puts communities on axes). The distinctness is real but lives in the sweep (structure you can dial), not the hero. Make the sweep the spine and the hero its p_out=low anchor; do not let the hero stand alone as the payoff, or it is a re-skin of the partition notebooks exactly as the audit fears.
  - [low-confidence] The deferred T1 forward-link is a prose-recorded skip with no arming trigger — at Sequencing / done-whens
    Rubric: no rubric yet — grill skipped
    Push: See A6. "Defer and record in the Implementation log" re-arms nothing; when !39 merges, no one re-checks the log. Name the trigger (a checklist bullet on the !39 merge, or a grep target the maintainer runs at convergence) or accept the forward-link may never land and say so.
```

**Existential verdict: NOT existential.** The could-this-not-exist angle (A5/A7 below) lands as "the framing must shift, and the payoff must be proven at the gate," not "this should not be built." The ground-truth + controllability + dial-the-break idea is genuinely distinct from every shipped sibling *if and only if* the sweep is the spine and the break is shown to read. I am not tagging `existential-must-fix`. See A7 for the honest statement of where the "one sentence in an existing notebook" alternative would win, so the maintainer-proxy can weigh it.

#### Detail (the three mandated angles)

**A1 — Premise (must-fix): the detector-recovery beat hides a label-alignment problem.** `louvain_communities` returns communities with arbitrary integer labels; there is no reason detected-label `0` corresponds to planted `block` `0`. A "per-node agreement flag" (`detected == truth`) is therefore ill-defined until the detected labels are aligned to the truth labels (a best-permutation / assignment step). The plan's Example 2 comment ("label 0 is the largest recovered community by the wrapper's stable ordering") notices the labels are ordered but does not solve the alignment; a stable ordering of detected communities does not make detected-`0` equal truth-`0`. This is the single most likely place the implementer halts under rule 9 ("if a helper seems needed, halt, do not invent"), because a correct agreement flag *does* need logic the notebook doesn't have on shipped surface. Two honest resolutions: (a) design the alignment explicitly and prove it runs on shipped surface (it is a few lines of pandas/`scipy.optimize.linear_sum_assignment` or a greedy contingency-table argmax, but it must be *specified* here, not discovered mid-build); or (b) cut per-node truth-match coloring entirely and make the detector beat qualitative ("Louvain recovers three communities that line up with the planted blocks," shown by coloring nodes by detected label and letting the reader see the axes already are the blocks). Option (b) is smaller and I lean toward it on the subtract bias, but the choice is the maintainer's; either way it must be settled before dispatch, not deferred to the gate.

**A2 — Approach (must-fix): the repeat_axes idiom is modeled wrong against its own near-neighbor.** The shipped `computing_graph_metrics.ipynb` partitions Karate Club by `club` (2 groups) with `repeat_axes=True` and then calls `hp.reset_edges(axis_id_1="Mr. Hi_repeat", axis_id_2="Officer")` with an inline note that on two groups the inter-group edges are otherwise drawn twice, once on each side. The plan's three snippets all set `repeat_axes=True` and none drop the redundant set. On 3 blocks the redundancy is threefold (each of the three inter-block pairs draws twice). This matters beyond snippet hygiene: the "cross-block bundles multiplying" signal that carries the whole "watch it break" narrative is exactly the inter-block edge set, so drawing it doubled (and mirrored) is precisely the visual the sweep is trying to make legible. The design gate must decide the repeat-axis + reset_edges topology up front and confirm the *de-duplicated* figure still reads, and the API-usage witnesses in this plan must be corrected so they don't ship an idiom the near-neighbor exists to correct. (Note: with 3 blocks and repeat axes, the reset_edges call is not one line; it is one per redundant pair. Confirm the ergonomics at the gate.)

**A3 — Premise (worth-discussing): "watch it break" on a hive plot is asserted, not evidenced.** The payoff claim is that as `p_out` rises, within-block bundles blur into cross-block ones and a reader *sees* the detectability threshold on the hive plot. On a hive plot specifically, within-block edges are repeat-axis (intra-group) curves and cross-block edges are inter-axis curves; whether the transition reads as a legible "break" at ~20 nodes/block in static matplotlib is an empirical open question the plan defers entirely to the design gate. That deferral is defensible *if* the gate is allowed to fail loudly. Right now the plan commits to shipping the deliverable and treats the gate as a parameter-tuning step, not a go/no-go. Add an explicit gate exit condition: if no combination makes the break legible on a static hive plot, surface back under rule 9 rather than reach for more frames, a bigger figure, or datashader (all three are already named out-of-scope, which is good; the missing piece is that "the break doesn't read" is itself a surfaceable outcome, not a tuning failure to paper over).

**A4 — Approach (worth-discussing): the sweep frame count is unbounded.** "A handful of static frames" and "a few static frames" appear with no cap. The plan's own Example 3 uses three (`0.01, 0.05, 0.12`). Gallery length discipline (skill: "a gallery page much longer than its closest sibling is too long") is the relevant bar, and an open-ended frame sequence is the standard way a gallery page outgrows it. Make three the cap, not an illustration, and add a per-frame legibility requirement: each frame must differ visibly from its neighbor at the render size the gallery actually shows, or it is not earning its vertical space. Three well-chosen frames (clean / threshold / blurred) is almost certainly the right budget; the plan should say so rather than leave "a few" open.

**A5 — Size/could-this-not-exist (worth-discussing): the hero figure alone is the re-skin the audit fears.** The section-worth audit's distinctness rests on "the partition *is* what generated the edges." That is true by construction, but it is invisible in a single static hive plot: a reader cannot distinguish "these blocks generated these edges" from "these nodes were assigned to A/B/C and happen to cluster," which is exactly what `setting_partition_variable.ipynb` / `create_partition_variable.ipynb` already show on the `example_*` A/B/C partition. The distinctness only becomes *visible* when the reader watches structure change as they dial `p_out` — i.e., in the sweep, not the hero. Disposition: make the sweep the spine of the notebook and demote the hero to the sweep's `p_out=low` anchor frame, so the page's reason-to-exist is the thing only this page can show (controllable structure), not the thing three pages already show (communities on axes). If instead the hero stands alone as the headline and the sweep is a footnote, the editorial-critic's own stated rejection trigger ("reads as another A/B/C partition demo") fires.

**A6 — Size/maintenance (low-confidence): the deferred forward-link has no arming trigger.** The T1 forward-link (T1 → this example) is deferred with "record it in the Implementation log so it is picked up when the branches converge." A skip recorded as prose re-arms nothing: when !39 merges, nothing makes anyone re-open this plan's Implementation log. Either name a concrete trigger (add it to the !39 merge checklist, or leave the maintainer a grep target like `grep -L partition_structure_sbm examples/finding_a_partition.ipynb` to run at convergence) or accept that the forward-link may simply never land and state that as the expected outcome. Low-confidence because the back-link (this → T1) carries the cross-reference either way and a missing forward-link is a soft loss, not a broken page.

**A7 — the honest "one sentence instead" comparison (for the maintainer-proxy).** Where the subtract alternative *would* win: if the notebook shipped only the hero figure (planted blocks on axes) plus the detector beat, it would be almost pure overlap with `computing_graph_metrics` + the partition notebooks, and a single sentence added to `computing_graph_metrics` ("for a partition that *generates* the edges with known ground truth, see ...") plus one figure would cover it more cheaply. What tips it to earning a full page is the *sweep* — a controllable-structure demonstration no shipped notebook has, and one that genuinely can't be a sentence because its whole content is the sequence of frames. So the page earns its place *conditioned on the sweep being the spine and reading legibly* (A3, A5). If the design gate finds the sweep doesn't read (A3's exit), the honest move is to fall back to the one-sentence-plus-figure addition, not to ship a hero-only page that duplicates three existing ones. That conditional is why this is not `existential-must-fix`: the page has a real reason to exist, but the reason is load-bearing on a gate that hasn't run.

Because the grill is knowingly skipped, this challenge routes to an amend-plan pass for explicit disposition (adopt / rule out, with the delta each adoption lands) rather than being fought in a grill.

### Adversary post-impl

```
Status: propose
Artifact reviewed: Workstream A — examples/partition_structure_sbm.ipynb (+ index.rst / conf.py / CHANGELOG / _static thumbnail); diff vs master dff7ead
Dispositions held: yes — A1-A6 all held in the shipped artifact, no scope balloon; both hypotheses that fixing A1/A2 might introduce a new bug are DISPROVEN by the baked figures (see reconciliation)
Concerns:
  - [low-confidence] Threshold-frame caption "the top-left axis has taken on the color of the block to its right" is a loose spatial claim that is hard to trace against the baked figure — at examples/partition_structure_sbm.ipynb (markdown after the threshold plot, "recovery starts to slip" section)
    Rubric: no entry
  - [low-confidence] Axes ship as raw integer labels 0/1/2, not renamed to legible names as the Section-worth audit softly asked — at examples/partition_structure_sbm.ipynb (already surfaced as an in-flight taste call in the Implementation log; flagging only to record I saw it)
    Rubric: no entry
```

**Blind pass (against done-whens + series constraints, before re-reading the planning challenge).** Every hardened done-when is satisfied in the shipped diff:
- **A1 encoding shipped correctly.** `partition_variable="block"` (axes = planted truth), `node_kwargs` colors by detected `louvain_communities` with `vmin=0, vmax=9` pinned; NO per-node `detected == truth` flag anywhere. The baked `draft_clean.png` renders three *distinct* monochrome-per-axis colors (orange / blue / green), i.e. the pinned normalization is doing its job and the per-scatter self-normalization collapse that Amendment 2 exists to prevent did **not** survive. Recovery-as-color-purity is legible.
- **No `reset_edges` de-dup loop** (A2 correction). Grep-clean; two-tone edges set at `HivePlot(...)` construction via `repeat_edge_kwargs` / `non_repeat_edge_kwargs`, not passed to `.plot()`. No `zorder` in `node_kwargs`.
- **3-frame cap** (A4): exactly three full-size figures (clean / threshold / broken), each its own figure, not a subplots grid; each legibly distinct at render size.
- **Sweep is the spine** (A5): the "Watching recovery break down" section frames the clean plot as the sweep's low-`p_out` anchor ("The single clean plot above could be any partitioned network... That only becomes visible when we change the structure"), not a standalone hero.
- matplotlib-only (no datashader); registered in **both** `index.rst` blocks adjacent to `computing_graph_metrics`, same order; `conf.py` thumbnail entry present; CHANGELOG bullet under `Version 0.29.0 (unreleased)` → `Documentation` in user terms; no gnn citation; no process refs (rule 15 clean).
- **Section-worth holds cold, now seeing the notebook.** It does **not** re-teach the detection mechanic — it delegates it ("For more on requesting node metrics like this one, see the Computing Graph Metrics page") and spends its own words on the ground-truth + controllability + watch-it-break payoff no sibling owns. The maintenance cost earns the gain.

**Reconciliation (my planning items A1-A6 vs their dispositions in Amendments 1 & 2, checked in the artifact).** All six dispositions held:
- **A1 held.** The subtract (drop truth-match flag; axes=truth, color=detected) shipped, and Amendment 2's completion (pinned `vmin`/`vmax`) shipped and *works* in the baked clean frame.
- **A2 held.** The Amendment-2 correction (no `reset_edges` at 3 blocks) shipped. Reconciliation check the prompt asked for: **dropping `reset_edges` left no real doubling** — the baked clean frame shows each inter-block coral fan drawn once, not a mirrored double-set, corroborating the gate's `total curves == number_of_edges()` claim against the visible artifact.
- **A3 held.** The legibility exit resolved GO with evidence (the break is visible in `draft_broken.png`); exit was not triggered.
- **A4 held.** Three frames, capped, legibly distinct.
- **A5 held.** Sweep-as-spine shipped; no static hero crept back (the disposition I most feared could regress in drafting did not).
- **A6 held as designed.** T1 absent from this tree → forward-link correctly did not land; SBM→T1 back-link shipped; post-merge trigger tracked by the dispatching session.

**Did fixing A1/A2 introduce a NEW problem? No — both hypotheses DISPROVEN by the baked artifact:**
- The `vmax=9` bound assumes ≤10 communities. The broken frame (`p_out=0.40`) finds **5** communities (title: "Louvain now finds 5 scrambled groups"), well under 10; no color clamp or tab10 wrap. (Direction is intuitive: at `p_out=0.40` vs `p_in=0.5` the graph is near-Erdős-Rényi-dense, and Louvain resolves dense graphs into *few* large modules, not many.)
- Dropping `reset_edges` on 3 blocks leaves no doubling (confirmed visually above).

**Verdict: propose (two low-confidence prose/taste nits, both already in or adjacent to the maintainer's in-flight batch review). No must-fix, no worth-discussing.** Dispositions held cleanly, no scope balloon, the artifact matches the contract. Nothing here needs to block the workstream; the two nits route as taste calls, not corrections.

## Workstreams

Single workstream. Build pattern is the gallery flavor: a design/parameter gate precedes the draft so the block-count and connectivity parameters are chosen to read cleanest before prose is written.

### Workstream A: SBM partition-structure gallery notebook

**Status:** not started
**Files:**
- `examples/partition_structure_sbm.ipynb` (new)
- `docs/source/gallery_examples/index.rst` (register in "The HivePlot Class" section — both the `nblinkgallery` block and the hidden `toctree`, same order; place adjacent to `computing_graph_metrics`)
- `docs/source/conf.py` (add an `nbsphinx_thumbnails` entry: `"notebooks/partition_structure_sbm": "_static/partition_structure_sbm.jpg"`)
- `docs/source/_static/partition_structure_sbm.jpg` (new thumbnail)
- `CHANGELOG.rst` (one bullet under `Version 0.29.0 (unreleased)` → `Documentation`)
- `examples/finding_a_partition.ipynb` (T1) — one-line forward cross-link, **only if T1 is present in the working tree at build time** (see sequencing note); otherwise it becomes the A6 named post-merge follow-up (armed when both !39 and this MR merge; tracked by the dispatching session as a task chip / MR note, not an Implementation-log line).

**Build sequence (record, do not pre-assign agents):**
1. Design/parameter gate: **this gate has run; its outputs are locked below** (see Amendment 2 for the empirical corrections). **3 blocks was chosen deliberately over 2** (richer 3-axis break story; the notebook is not a `reset_edges` demo). The gate rendered and verified on the locked 3-block SBM: with `repeat_axes=True` the total edge-curves drawn equals `graph.number_of_edges()` exactly, so **no `reset_edges` de-dup is needed or correct at 3 blocks** (the ring draws each inter-block pair once; the doubling `reset_edges` corrects is a 2-block phenomenon; a de-dup loop here would delete a third of the inter-block signal or `KeyError`). The **A1 encoding reads only with `vmin=0, vmax=9` pinned** in `node_kwargs`: nodes colored by *detected* `louvain_communities` over *planted*-block axes render monochrome without the pinned normalization (per-scatter self-normalization), near-monochromatic-per-axis with it. Other locked draft constraints the gate verified: no `zorder` in `node_kwargs` (viz hardcodes `zorder=5`; passing it raises `TypeError`); two-tone edges set at `HivePlot(...)` construction via `repeat_edge_kwargs` / `non_repeat_edge_kwargs`, not passed to `.plot()`; each frame rendered as its own full-size figure, not a subplots grid; seed 0 kept (the threshold frame is phase-transition-sensitive and the seed-0 appearance, a partial scramble on 2 of 3 axes, is what was validated). **A3 exit was honored at this gate:** a `(3 blocks, p_in=0.5, p_out sweep, ~20 nodes/block)` combination makes the break legible on a static matplotlib hive plot at gallery render size; had none been found, that would have been a rule-9 surface-back, not a fallback to more frames / a bigger figure / datashader. viz-critic concurs on the chosen frames before the draft. Hand the maintainer runnable code + rendered images each round (working-style: co-design wants runnable artifacts, not narrated conclusions).
2. Draft the notebook (gallery skill): H1 title, lead-in on what it covers (assume the reader knows what a hive plot is), imports, body sections default→advanced with **the density sweep as the spine (A5)**: the clean figure enters as the sweep's low-`p_out` anchor, detector-recovery coloring, then the rising-density frames, not a standalone hero followed by a decorative sweep. Closing cross-link paragraph(s). Seed all generation.
3. Register in `index.rst` (both blocks), add the `conf.py` thumbnail entry, generate the `_static` thumbnail, add the CHANGELOG bullet.

**Done when:**
- `examples/partition_structure_sbm.ipynb` exists and runs end to end under the worktree's own kernel (`make test-nb` or an explicit `python3`-kernel run; do not rely on the user-level `hiveplotlib` kernel in a secondary worktree).
- The notebook builds its graph with `nx.stochastic_block_model(..., seed=0)`, partitions every hive plot by the **planted** `block` label (axes = ground truth), and runs `louvain_communities` (seeded) with **nodes colored by the detected community** and a **pinned normalization** (`node_kwargs={"c": "louvain_communities", "cmap": "tab10", "vmin": 0, "vmax": 9, ...}`), so recovery reads as **color-purity per axis**. **The pinned `vmin`/`vmax` is required (Amendment 2):** without it `node_viz`'s per-axis `ax.scatter()` self-normalizes each single-community axis to one colormap point and every axis renders the same color, making the recovery-as-color-purity encoding invisible. **No `detected == truth` per-node flag** (A1: that would need a label-alignment step the notebook must not carry; good recovery = near-monochromatic axes, breakdown = scrambled colors). No `zorder` in `node_kwargs` (viz hardcodes `zorder=5`; passing it raises `TypeError`). All data construction is on shipped surface; no helper is invented (if one seems needed, halt per rule 9, do not invent).
- **No `reset_edges` de-duplication at 3 blocks (Amendment 2, supersedes the earlier A2 de-dup done-when):** the gate verified that with `repeat_axes=True` the locked 3-block ring draws each inter-block pair exactly once (total curves == `graph.number_of_edges()`), so the notebook ships **no** `reset_edges` de-dup loop; adding one would delete a third of the inter-block signal or `KeyError`. The doubling the near-neighbor `computing_graph_metrics` corrects is a **2-block** phenomenon; the A2 intent (draw the cross-block signal once) is satisfied for free at 3 blocks. Two-tone edge styling (within-block vs cross-block) is set at `HivePlot(...)` construction via `repeat_edge_kwargs` / `non_repeat_edge_kwargs`, not passed to `.plot()`.
- The notebook includes **at most three static frames** (A4 cap) at rising inter-block density showing within-block bundles blurring into cross-block ones (the detectability threshold made visual). Each frame differs legibly from its neighbor at gallery render size, or it is dropped; if three legibly-distinct frames can't be found, the sweep is dropped rather than padded. Not an animation; kept gallery-sized.
- **The design gate's A3 legibility exit was honored:** the notebook ships only because a `(block-count, p_in, p_out, node-count)` combination was found that makes the break legible on a static matplotlib hive plot at gallery render size. If the gate had found none, that is a rule-9 surface-back (recorded as such), never a silent fallback to more frames, a bigger figure, or datashader.
- The clean anchor figure is a **single clean hive plot on 2-3 axes** (block = axis), entering as the **low-`p_out` anchor of the sweep**, not a standalone hero (A5). More-than-3-blocks is at most gestured at as "this becomes a HivePlotMatrix," never the demonstration (2-3 axis rule; class scope stays on the single hive plot).
- Prose makes the distinct payoff explicit (ground truth + controllability + watch-it-break; the partition *generated* the edges), and **the density sweep is the spine (A5)**, so the page does not read as a rehash of the partition-mechanics notebooks. It does not lean on "how to call `louvain_communities`" as its spine, nor let the clean anchor frame stand alone as the headline.
- Registered in `docs/source/gallery_examples/index.rst` under "The HivePlot Class" (both `nblinkgallery` and hidden `toctree`, same order), with a matching `conf.py` thumbnail entry and a `_static/partition_structure_sbm.jpg` thumbnail.
- One CHANGELOG bullet under `Version 0.29.0 (unreleased)` → `Documentation`, in user terms.
- House voice throughout (no em-dashes, no AI filler); no process references (no wiki-plan / WS-label / grill / date mentions in the shipped notebook or CHANGELOG) per rule 15.
- llms.txt: judged and **skipped** unless the editorial/docs pass argues otherwise. Per the trip-wire, a gallery variation on already-indexed partition/community material is a routine addition, not a new conceptual entry point; adding an entry would bloat the hand-curated index. Record the judgment either way.
- The T1 forward cross-link (T1 → this example) is applied only if T1 is in-tree at build time; otherwise it is a **named post-merge follow-up (A6), armed when BOTH MR !39 (T1) and this example's MR have merged to master**. Until both merge it does not land, and the dispatching session tracks it (a task chip / MR-description note), *not* an orphaned Implementation-log line that nothing re-reads. The notebook itself still points back at `finding_a_partition.ipynb` as the real-data narrative now (a graceful forward-reference — the link text ships even though T1's file lands via !39).

**Out of scope (do not let the notebook accrete these):**
- The richer "plant surprising cross-block edges, color by detector-vs-truth recovery across the whole graph" narrative — reserved for the tutorial-series capstone (see Prior ADRs / design docs).
- Any datashader frame (matplotlib only; the synthetic graph is small and the sweep frames stay legible in matplotlib). If a sweep frame genuinely needs rasterization to stay readable, that is a rule-9 surface-back, not a silent backend switch.
- HivePlotMatrix as the hero (>3 blocks is a gesture, not the demonstration).
- Any new dataset helper in `hiveplotlib.datasets` (the SBM is built inline in the notebook via `nx`, matching gallery convention of minimal custom data; a shipped `example_sbm` helper would be a library API change, out of scope for this doc-only plan).

## Plan amendments

### Amendment 1 (2026-07-07) — dispose the adversary planning challenge (grill skipped)

*In-scope tweaks.* All six adversary planning items (A1-A6) adopted; the maintainer-proxy (dispatching session, under an autonomy delegation) decided each. No `existential-must-fix`; the core concept (SBM gallery with **ground truth + controllability + watch-it-break**) is unchanged, so none is an added workstream or a deferred follow-up. A1 and A5 are the load-bearing design changes. Feasibility of the two design changes was re-checked on shipped surface before adoption (witnesses cited below).

- **A1 (must-fix) — adopted, the subtract.** Cut the per-node `detected == planted-block` truth-match flag (ill-defined: `louvain_communities` returns arbitrary label integers, so a correct flag needs a permutation/contingency alignment step the gallery must not carry). Replaced with an alignment-free encoding: **axes = planted `block` (truth); node color = detected `louvain_communities`.** Recovery reads as color-purity per axis (near-monochromatic = good, scrambled = breakdown). *Delta:* rewrote Goal, Section-worth audit, API-usage Example 2 (+3), and the detector done-when; dropped all `detected == truth` comment/logic. *Feasibility:* coloring nodes by a data column via `node_kwargs={"c": "<column>", ...}` is shipped, witness `visualizing_node_metadata.ipynb:288`; the categorical `louvain_communities` label column maps through a `cmap` to distinct per-community colors. **(Completed by Amendment 2:** the encoding is only *visible* with a pinned `vmin=0, vmax=9` in `node_kwargs`; without it the per-axis scatters self-normalize to one colormap point and every axis is the same color.)

- **A2 (must-fix) — adopted, then SUPERSEDED by Amendment 2.** As disposed here the plan called for `reset_edges` de-duplication on every snippet, on the premise (from the 2-block near-neighbor) that `repeat_axes=True` draws the inter-block set twice. **The design gate rendered the locked 3-block SBM and found no such doubling: total curves == `graph.number_of_edges()`, so no de-dup is needed or correct at 3 blocks** (a de-dup loop would delete a third of the inter-block signal or `KeyError`). The doubling is a **2-block** phenomenon; the A2 *intent* (draw the cross-block signal once) holds for free at 3 blocks. See Amendment 2 for the corrected snippets, gate step, and done-when. *Original feasibility note (retained for the 2-block case only):* `reset_edges` per inter-block pair is the 2-group near-neighbor idiom, witness `computing_graph_metrics.ipynb:145`, signature `hiveplot.py:898`; repeat-axis names `"<block>_repeat"` / `"<block>"`.

- **A3 (worth-discussing) — adopted.** Added an explicit design-gate **legibility EXIT**: if no `(block-count, p_in, p_out, node-count)` combination makes the break legible on a static matplotlib hive plot at gallery render size, that is a rule-9 surface-back to the dispatching session, never a silent fallback to more frames / a bigger figure / datashader. *Delta:* written into the design-gate build step and a done-when.

- **A4 (worth-discussing) — adopted.** Capped the connectivity sweep at **three static frames** (the plan's own snippet count, now the ceiling), each required to differ legibly from its neighbor; if three legibly-distinct frames can't be found, drop the sweep rather than pad it. *Delta:* updated the Default-justifications sweep bullet, the API-usage Example 3 comment, and the sweep done-when.

- **A5 (worth-discussing) — adopted, framing correction.** Made the **density sweep the notebook's spine**; demoted the clean figure to the sweep's **low-`p_out` anchor**. A single static SBM hive plot alone is the "another A/B/C partition demo" re-skin (that the partition generated the edges is invisible in one frame); the distinct payoff is realized *by* the sweep. *Delta:* rewrote Goal, the Section-worth distinctness paragraph, the draft build step, the anchor-figure and prose done-whens.

- **A6 (low-confidence) — adopted lightly.** The deferred T1 → SBM forward-link gets a named trigger: a **post-merge follow-up armed when BOTH MR !39 (T1) and this example's MR have merged to master**, tracked by the dispatching session (task chip / MR-description note), not an orphaned Implementation-log line. Until both merge it does not land; that is the stated expected outcome. The SBM → T1 back-link still ships now as a graceful forward-reference. *Delta:* updated the T1 Files bullet, the T1 done-when, and the Sequencing note.

Post-impl adversary reads this disposition alongside its blind pass on the shipped notebook.

### Amendment 2 (2026-07-07) — design-gate empirical corrections

*In-scope tweaks.* Two empirical corrections the SBM design/parameter gate surfaced by rendering and verifying the locked 3-block SBM; disposed by the dispatching session as maintainer-proxy. Both make the design *more* correct and neither changes the core concept or the GO (**still**: SBM gallery with ground truth + controllability + watch-it-break; the density sweep is the spine). Correction 1 reverses part of Amendment 1's A2 disposition on new empirical evidence; Correction 2 closes a plan-completeness gap in the A1 encoding.

- **Correction 1 — A2 was factually wrong at 3 blocks; NO `reset_edges` de-dup is needed or correct.** The gate rendered and verified: on the locked **3-block** SBM with `repeat_axes=True`, total edge-curves drawn == `graph.number_of_edges()` exactly, i.e. **no doubling**. The 3-block ring draws each inter-block pair once (0↔1 as `0_repeat→1`, 1↔2 as `1_repeat→2`, 0↔2 as `0→2_repeat`). The doubling `computing_graph_metrics` corrects is a **2-block** phenomenon (reproduced: 2 blocks draw the inter-block set twice). So the A2 disposition's *intent* (draw the cross-block signal once, cleanly) is satisfied for free at 3 blocks, and the notebook must **NOT** ship a `reset_edges` de-dup loop (at 3 blocks it would delete a third of the inter-block signal or `KeyError`). **3 blocks was chosen deliberately over 2** (richer 3-axis break story; the notebook is not a `reset_edges` demo). *Delta:* rewrote the A2-correction preamble in `## API usage examples`; dropped the `reset_edges` loops from all three examples; corrected the API-Critic's-take witness recording; restated the design-gate step and the de-dup done-when as "no de-dup at 3 blocks"; marked Amendment 1's A2 entry SUPERSEDED. The axis-id spelling note (base axes ints `0,1,2`; repeats strings `"0_repeat"` etc.) is kept for reference in case a reader adapts to 2 blocks, where the de-dup *would* apply.

- **Correction 2 — A1 requires a pinned `vmin`/`vmax`; without it the encoding renders monochrome.** `node_kwargs={"c": "louvain_communities", "cmap": "tab10"}` alone renders every node the same color: `node_viz` draws a separate `ax.scatter()` per axis, each planted-block axis holds a single detected-community value, and matplotlib normalizes `c` per scatter call, so each constant-`c` axis self-normalizes to the same colormap point. Fix (gate-verified to produce distinct per-axis colors): pin **`vmin=0, vmax=9`** (tab10 has 10 discrete slots) in `node_kwargs`. Without it the entire recovery-as-color-purity encoding is invisible. This is a plan-completeness gap, not a design change. *Delta:* added `"vmin": 0, "vmax": 9` to the A1 encoding snippets (Examples 2 and 3) in `## API usage examples`; added the pinned-normalization requirement to the detector done-when and the design-gate step; annotated Amendment 1's A1 entry as completed here.

- **Other locked draft constraints the gate verified (recorded here so the plan carries them, not only the dispatch brief):** no `zorder` in `node_kwargs` (viz hardcodes `zorder=5`; passing it raises `TypeError`); two-tone edge kwargs (within-block vs cross-block) set at `HivePlot(...)` construction via `repeat_edge_kwargs` / `non_repeat_edge_kwargs`, **not** passed to `.plot()`; each sweep frame rendered as its own full-size figure, not a subplots grid; seed 0 kept (the threshold frame is phase-transition-sensitive and the seed-0 appearance, a partial scramble on 2 of 3 axes, is what was validated). *Delta:* folded into Example 2 (edge kwargs + no-zorder), Example 3 (own-figure + seed-0), the design-gate step, and the detector done-when.

Core concept and GO unchanged. Post-impl adversary reads this correction alongside Amendment 1 and its blind pass on the shipped notebook.

## Holdouts

None.

## Implementation log

Append-only. After the workstream completes, one line in the same turn.

- Workstream A: `examples/partition_structure_sbm.ipynb` authored + executed clean (outputs baked, kernelspec `hiveplotlib`); both post-impl critics clear (no must-fix; two taste calls, axis-label form and grayscale palette, surfaced to the maintainer for batch review); registered in the gallery index ("The HivePlot Class", adjacent to `computing_graph_metrics`) with the `conf.py` thumbnail entry + a 2400x2400 clean-frame `_static` thumbnail + a `Version 0.29.0` Documentation CHANGELOG bullet; `llms.txt` entry and the reciprocal T1 link deliberately skipped (routine gallery addition; T1 absent from this tree); adversary + qa close pending.
- QA close (Workstream A): `Status: fail` on one blocker only. `make test` 1359 passed / 100% coverage, `ty` clean, `make format` no-op, notebook executes clean under `-W error` (python3 kernel), `uv audit` clean, all grep audits clean (registration exact, no stale hits, `llms.txt` correctly untouched, no rule-15 scaffolding, no em-dashes / AI filler, no `reset_edges` / `zorder`), CHANGELOG entry within cap, kernelspec `hiveplotlib` correct. Blocker: the graceful T1 forward-link (`[Finding a Partition](finding_a_partition.ipynb)`, closing cell) produces the sole docs-build warning (`nbsphinx.localfile` file-not-found, T1 absent pre-!39), which violates the zero-warnings bar. New gallery page + 3 figures + thumbnail all build and resolve. Recommend the maintainer drop the forward-link text now and let the A6 post-merge follow-up add it with T1's reciprocal link once both MRs merge; that clears the warning and matches house cross-link convention (all siblings use local `.ipynb` links, none absolute URLs).
- QA blocker fix (Workstream A): qa found the SBM→`finding_a_partition` forward-link broke the docs build (T1 absent from this tree; nbsphinx local-file file-not-found warning under warnings-as-errors); markdown-only edit removed the link and reworded the closing paragraph to keep the synthetic-with-ground-truth vs. real-data-discovery contrast without naming/linking the absent notebook (no re-execution; code-cell hash unchanged, 3 baked figures + kernelspec `hiveplotlib` intact), both reciprocal cross-links deferred to the post-merge follow-up; `make docs` now zero-warning.

---

## Extras

`hiveplotlib[networkx]`. The notebook needs `networkx.stochastic_block_model` (graph construction), `networkx_to_nodes_edges` / the `graph=` init path, and the `louvain_communities` node-metric string, all of which live behind the `networkx` extra. Surface the install command up front in the lead-in (`pip install hiveplotlib[networkx]`), matching `computing_graph_metrics.ipynb`. matplotlib is the render backend (default); no datashader.

## Sequencing (note for the dispatching session, not plan content to decide)

- This ships as a **standalone deliverable** on its **own branch + MR off master**, with its own issue — separate from MR !39's tutorial batch. It is not part of the tutorial-series Phase 1. The dispatching session sets that up; this plan does not.
- **`finding_a_partition.ipynb` (T1) is not in this working tree** (verified: 0 cells found under `examples/finding_a_partition.ipynb`; it is absent from `conf.py`'s thumbnail list too). T1 lives on MR !39. Consequences:
  - The **back-link** (this notebook → T1) is written now as a graceful forward-reference; both endpoints resolve once !39 merges.
  - The **forward-link** (T1 → this example) edits T1's file, which this branch does not contain. Apply it only if T1 has landed in the working tree by build time; otherwise it is the **A6 named post-merge follow-up, armed when BOTH MR !39 (T1) and this example's MR have merged to master**. The dispatching session tracks it as a task chip / MR-description coordination note (not an Implementation-log line that nothing re-reads); until both merge it does not land, and that is the stated expected outcome, not a blocker on shipping this notebook.
