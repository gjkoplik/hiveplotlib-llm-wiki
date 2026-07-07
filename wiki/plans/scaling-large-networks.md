# Plan: scaling hiveplotlib to larger networks

## Goal

Open doors for hiveplotlib users to render dramatically larger graphs (10M+ edges, OGB-arxiv-scale, and beyond) without hitting peak-memory cliffs, and let users bring their own dataframe library (polars, modin, Dask, cuDF) without the library forcing pandas at the input boundary. No observed RAM pain today; this is forward-looking infrastructure to unblock use cases the library cannot serve.

### Two distinct meanings of "out-of-core" (read this before sequencing)

"Out-of-core" means two different things here, and narwhals is required for only one of them. Getting this split right is what makes the dependency chain below correct.

1. **Internal chunking of an in-RAM-but-huge edge frame.** hiveplotlib holds a single large pandas/arrow edge frame and streams it itself, never materializing all curves or all rasters at once. This is the minimum-viable out-of-core. It needs **no narwhals**. It is delivered by the membership-storage redesign (Workstream 1), the reduction-aware chunked rasterization (Workstream 2), and the fused build path (Workstream 3, the real resident-memory unlock).
2. **Bring-your-own foreign frame engine** (Dask, polars, cuDF). The user's data already lives in a non-pandas frame, possibly off-RAM (Dask) or on-GPU (cuDF). This needs narwhals as the dispatch layer. It is delivered by narwhals at the input boundary (Workstream 4) and Dask + cuDF/GPU passthrough (Workstream 5).

The load-bearing consequence: **narwhals is the on-ramp for the bring-your-own-engine and GPU stories, not a prerequisite for in-RAM-but-huge scaling.** Workstreams 1-3 can ship and unblock 10M+ edge plots without any narwhals dependency landing.

### Transient vs. resident memory

Two memory costs are in play, and they need separate fixes:

- **Transient memory** is the concat copy built right before rasterization (`datashader.py:224-231`, `np.concatenate` + `pd.DataFrame`). Chunked rasterization (Workstream 2) removes it.
- **Resident memory** is the per-curve geometry persisted on the object by `construct_curves` (`hive_plot_edges[...][tag]["curves"]`, roughly 808 MB per 1M edges at `num_steps=100` float32, ~8 GB at 10M, impossible at 1B). Chunked rasterization does **not** touch this; the curves still get built and held. The fused build path (Workstream 3) is what removes the resident cost by sampling each chunk's curves, rasterizing them into the running aggregate, and discarding them.

### Sequencing and scope guards

Every workstream here is its own MR with its own validation, and **every workstream is gated by the separate performance-regression harness** (`wiki/wiki/plans/performance-regression-harness.md`, sequenced first). Each workstream's done-when includes harness-based speed + no-regression validation, plus the var/std-equivalence gate wherever rasterization changes. Gating semantics are ratio-based per the harness plan's 2026-06-11 amendments (pytest owns the equivalence gate and relative same-run CI ratio assertions; ASV owns rolling baseline / history / blog-figure data).

Out of this super-plan entirely: the Bézier "Bernstein hoist" (loop-invariant weight table; a small bandwidth-bound win) ships as a standalone reviewed PR, and numba autotuning is deferred (already mined). Do not add either here.

The autonomous-research while-loop is also deferred: these workstreams land first, manually / agent-assisted with careful human review, and only once the surface is clean and the harness trusted does an autonomous loop get wired up. Do not wire a loop prematurely.

## Alignment (grill)

Status: aligned (3 waves, 2026-06-05). Open decision resolved post-grill: `use_dask` → **Path A** (drop the parameter; detect Dask via narwhals and handle it uniformly like every other engine; partition-sizing and shuffle-cost guidance lives in the Dask notebook). Routing to orchestrator `amend-plan` alongside the speedup-blog-notebook scope add.

This plan predates the dedicated `## Alignment (grill)` section; the full wave captures live under "Plan amendments" below (`Maintainer shared-understanding pass (grill), Wave 1/2/3`). Future plans capture directly here. Summary:

- **Wave 1 — premise / scope / core-dep.** Forward-looking investment, no external demand signal; motivations are unblocking the maintainer's GNN work at larger scale and the field's lack of good very-large-network viz. Ship all five workstreams. narwhals core-dep endorsed (load-bearing: pure-Python, `none-any` wheel, no compiled extensions).
- **Wave 2 — algebra / memory / threshold.** `use_dask` and `stream_chunk_threshold` are orthogonal axes (input engine vs. internal geometry streaming). var/std are single-shot-or-delegate (recovered to streaming in WS5). Resident-vs-transient measured cumulatively (WS2+WS3, never WS2 alone). Shared streaming threshold surfaced explicitly at review.
- **Wave 3 — harness / ADR / `use_dask`.** Anti-reward-hack guard (equivalence gate + identity/perturbation meta-check) sufficient for supervised work. Full harness wanted; success bar is feasibility-at-scale first. Speedup blog notebook added (→ amend-plan). Autonomous loop deferred. Single umbrella ADR on close. `use_dask` deepened: auto-safety unachievable, residual risk narrow, leaning lenient — resolved above to Path A.

### Maintainer shared-understanding pass (grill), Wave 4 — adversary-challenge disposition (2026-07-03)

Pre-dispatch alignment session for the WS1+ execution window (maintainer at keyboard, then low-supervision with remote phone review). Dispositions on the adversary's cold challenge (see `## Adversary review`):

- Items 1 (Path A never cascaded into normative sections) + 2 (WS1's stale consumer map): confirmed as plan-text drift, no design change; routed to orchestrator `amend-plan` before WS1 dispatch.
- Item 3 (unattended executability): the green-lit auto window is WS1 → WS4 in full (doc artifacts drafted, churn-expected), then WS5's CPU-exercisable work with GPU-gated pieces parked; hard halt before WS5 close / WS6 / WS7. Maintainer utterance recording the auto-dispatch opt-in: "I'm happy with you moving as far and as quick as you can."
- Item 4 (silently-skipping memory gates): accepted and strengthened — memory gates always run (no RAM-unavailable skip on the canonical 32 GB host); when RAM is contended they serialize after the parallel suite finishes; a skipped memory gate in the auto window is a halt-back, not a proceed.
- Items 5 / 6 / 7: dispositions accepted as proposed — WS1's value restated as structural rather than "strictly better in RAM always" (plus the OGB-arxiv attribution fix); the `[dask]`-extra question reconciled against the fact that `dask[dataframe]` already ships in the `datashader` extra; WS4's pin-time checklist gains "re-derive the `<2` bound, not just verify it." All fold into the same `amend-plan` dispatch.
- Execution-window mechanics recorded: maintainer pre-authorizes one commit per workstream to branch `53-scale-hiveplotlib-to-larger-networks` when that workstream's full gate battery is green (an explicit standing exception to agents-never-commit, scoped to this execution window only); doc artifacts are draft-and-continue; progress surfaced via dashboard artifact, per-workstream markdown review packets, and push notifications.
- **Addendum (2026-07-03, later same session):** with the GPU verification env built at `~/venvs/hiveplotlib-gpu` and the cupy mempool metric proven end-to-end on the RTX 5070, the maintainer extended the auto window to include **WS5 in full** (GPU-gated tests and VRAM measurement run unattended on the canonical host; notebooks draft-and-continue). This supersedes the item-3 window bullet above. Hard halts remaining: WS6 (release-slug + publication prose), WS7 (parks with WS6 so its numbers are final), chain close (tier-2 pruning decision + umbrella ADR promotion), plus the standing halts (any must-fix, any `STATUS: BLOCKED`, any finding bearing on downstream not-yet-run workstreams).

### Maintainer shared-understanding pass (grill), Wave 5 — failure-mode elicitation (2026-07-03)

The failure-mode wave ran. Four named modes captured in `### Failure modes` below (modes 1-3 maintainer-named; mode 4 proposed by the dispatching session from the adversary's item-2 mechanism and adopted by the maintainer). The adversary's conditional post-grill rubric-check runs against these before WS1 dispatch.

### Failure modes

What would make this work a failure even with every gate green (maintainer-named, 2026-07-03). Living rubric: implementers append weeds-level modes they hit; the adversary attacks shipped workstreams against this list.

1. **Renders-but-unreadable.** The 10M-edge render completes but at the validated raster resolution the structure is mush; a real user needs a readable picture at scale (as good as rasterization can give — a 10M-edge hive plot is a blob, but it must be the right blob at usable resolution), not a figure object that merely returned.
2. **Edge-only scaling.** Scenarios escalate edges while node counts stay token; if node-side cost stalls relative to edge-side, real large networks (big on both axes) still can't render. Node scaling needs coverage proportional to edge scaling.
3. **Unrepresentative parallelism.** A local Dask/multiprocessing speedup that is an artifact of the local cluster setup and would not hold on real distributed hardware (a Coiled-scale cluster); published numbers must be representative of larger hardware, or framed explicitly as local-only.
4. **Plausible picture of the wrong data.** A selection/indexing bug (e.g. positional-vs-label divergence on non-RangeIndex user frames) that clean fixtures never exercise: the plot renders confidently, gates stay green because the equivalence baseline shares the wrong selection, and the picture lies.

## Prior ADRs / design docs

*Section refreshed 2026-07-03; originally written when no ADRs existed and ADR promotion had never been exercised.* Two ADRs now exist:

- `wiki/wiki/adr/0001-networkx-integration.md` — the v0.28 NetworkX story (`graph=` input, `graph_features`, backend dispatch, conflict validation). Background context only for this plan.
- `wiki/wiki/adr/0002-performance-regression-harness.md` — **directly load-bearing**: the shipped performance-regression + equivalence gate every workstream here must clear. Binding decisions this plan inherits: pytest relative same-run ratio gates + ASV capture-at-merge history (never absolute timings in CI), the dtype-aware equivalence wall, two-tier peak-RSS measurement (tier 2 provisional — the chain-close pruning checkpoint below), pinned tiny/small/medium/large scenarios, the canary-armed dormant gates awaiting WS2, and the `perf_harness` CI lane. Its working plan (still cited by raw path throughout this document) is archived at `wiki/wiki/plans/archived/performance-regression-harness.md`.

Other relevant prior thinking lives in design docs and source pages rather than ADRs:

- `wiki/wiki/sources/hiveplotlib-python.md` lines 79-84 — documents the "minimal base deps: matplotlib, numpy, pandas only" stance. The Workstream 2 decision on whether narwhals joins core deps is a deliberate deviation from this and needs explicit justification in the eventual ADR.
- `wiki/wiki/analyses/gnn-heterogeneity-hive-plots.md` line 113 — names OGB-arxiv (169k nodes, 40 classes) as the "large-scale stress test (datashader backend)" target. Workstreams 2+3 are the unblock (1.16M edges ≈ 1 GB of curves is feasible today; removing the transient and resident memory costs is what changes scale); Workstream 1 is the structural precursor. *(Attribution corrected 2026-07-03; previously claimed "Workstream 1 is the unblock.")*
- `wiki/wiki/analyses/cora-prototype-plan.md` lines 411-434 — "Phase 5: Iterate and Scale" depends on the datashader path holding up at OGB-arxiv scale. Direct dependency on Workstreams 2+3 (same 2026-07-03 attribution correction).
- `wiki/wiki/entities/hiveplotlib.md` lines 53-56 — "Development Priorities" does not currently list scaling. Entity page updates in the post-task research-liaison pass.
- `wiki/wiki/concepts/edge-rendering.md` lines 39-46 — "Rendering Pipeline" documents the four-stage pipeline. Workstream 1 changes the shape of step 4 for the datashader backend; the concept page gains a note on chunked aggregation but does not change overall shape.
- `wiki/wiki/concepts/applications-cybersecurity.md` lines 19-23 — DDoS-classification ML-featurization use case (Rivas 2019, Guarino 2020, Bragg 2025) is where billion-edge plots could plausibly arise.
- `wiki/wiki/sources/perez-2021-hype.md` — HyPE at 1,880 OTUs / 13,605 edges as state-of-the-art comparator. Useful framing for "several orders of magnitude beyond prior tools" when writing the eventual ADR(s).
- `wiki/wiki/concepts/bezier-curves.md` — documents the Bézier kernel that this plan repeatedly names as the dominant compute and resident-memory cost. Records the numba serial/parallel switch (threshold ~4,096 points) and float32 edge-array storage. Reference it for any kernel-touching context, especially the fused-build resident-memory discussion (Workstream 3) and the per-partition curve materialization ceiling (Workstream 5).
- `wiki/wiki/sources/nollenburg-2023-computing-hive-plots.md` — hive plot construction is NP-complete; useful framing for "why is this intrinsically expensive at scale" in the eventual ADR.
- CHANGELOG line 238-239 documents the already-shipped float32 work in `src/hiveplotlib/hiveplot.py:1018-1055`. This plan builds on that base.

ADR promotion strategy at task close: one umbrella ADR with numbered decisions, since the workstreams form a single architectural story (membership storage → reduction algebra → fused build → input boundary → BYO-engine/GPU). Decided as one ADR rather than several because the decisions are coupled (the membership-storage structure is shared by the in-RAM sparse rewrite and the Dask non-materializing equivalent; narwhals at the boundary is what makes Dask passthrough cheap and cuDF/GPU a marginal addition; the fused build is what makes the in-RAM out-of-core story real). The numbered decisions the ADR should cover: (1) sparse integer-index membership storage (in-RAM, with the Dask non-materializing equivalent), (2) the reduction algebra (additive vs. mean-accumulate vs. delegate-or-single-shot for var/std), (3) the fused build-and-rasterize path and the resident-vs-transient memory distinction, (4) narwhals-as-on-ramp (input boundary, core-dep deviation), and (5) cuDF/GPU passthrough riding on narwhals + datashader's CUDA path. *(2026-07-03 update: ADR promotion has since been exercised — ADRs 0001 and 0002 exist — so this plan's umbrella ADR follows the established procedure at close, numbered next in sequence.)*

**Chain-close checkpoint — tier-2 pruning decision (2026-07-03; tracked here alongside ADR promotion so it cannot be skipped at close):** before this super-plan closes, an explicit recorded removal-or-retention decision on the harness's provisional tier-2 process-tree peak-RSS helper (`measure_peak_rss_tree`, `runners/performance/profiling_utils.py`). If the profiling practice actually used across the chain did not employ tier-2 and its presence can no longer be justified, it MUST be removed before close: the `measure_peak_rss_tree` function, `_sample_tree_rss`, its two tests, the tree workload, and psutil from the `testing` extra if nothing else needs it. Retention likewise requires an explicit recorded justification (the retained unique property is external-observer/ungameability), not a silent lingering. Canonical rationale record: `wiki/wiki/plans/performance-regression-harness.md`, 2026-07-03 amendment. *Pre-staged input (2026-07-05, WS5): WS5's profiling practice did NOT use tier-2 — tier-1 `measure_peak_rss` was selected for the Dask route (threaded scheduler verified in-env) and the GPU regime measures VRAM via RMM statistics, outside tier-2's domain; see the 2026-07-05 amendment, item 3.*

## Patterns this replaces

The replace-and-sweep audit cuts across all five workstreams. Counts come from a `grep -E '\.loc\[|\.isin\(|\.iloc\[|to_numpy\(\)|to_dict\(|\.apply\(|\.groupby\(|\.merge\(|\.values\b|pd\.concat|pd\.DataFrame|pd\.cut|pd\.qcut|pd\.Series|astype\('` sweep of `src/hiveplotlib/`: 206 occurrences across 19 files. Not all are in scope; the audit below names the in-scope ones by workstream.

### Workstream 1 — membership storage redesign (sparse integer indices)

- **`BaseHivePlot.__store_edge_ids` membership storage** (def at `src/hiveplotlib/hiveplot.py:893`, store site at `:958`; `self.edges.relevant_edges[from_axis_id][to_axis_id][tag] = indices_to_store`). Today stores a full-length boolean mask per `(axis_pair, tag)`. Edges partition across axis-pairs (each edge belongs to exactly one ordered pair), so the bool storage is O(num_pairs × num_edges) mostly-False bytes; integer indices are O(num_edges) total across all pairs. The byte-level win is dtype- and axis-count-dependent (a 3-axis plot stores ~6 one-byte bool masks ≈ 6 bytes/edge; int64 indices ≈ 8 bytes/edge — worse, not better; int32 wins) and is MBs against the curves' GBs either way. **WS1's value is structural:** one storage decision whose Dask non-materializing equivalent (Workstream 5) extends it rather than redesigns it, and a positional-take consumer shape WS2/WS3 stream over. *(Claim restated 2026-07-03; supersedes "strictly better in-RAM whenever multiple axis-pairs exist (always)" here and in the 2026-05-29 Added-workstream amendment.)*
- **Read sites of `relevant_edges`** (grep-verified 2026-07-03; the earlier "only known read site is `datashader.py:203-204`" claim was false): `src/hiveplotlib/viz/matplotlib.py:460-463` (`.loc[relevant_edges, val]`), `src/hiveplotlib/viz/bokeh.py:580-581` (`_data[tag][relevant_edges]`), `src/hiveplotlib/viz/plotly.py:726-729` and `:774-776` (`.loc[relevant_edges, val]` and `_data[tag][relevant_edges].iloc[i, :]`), `src/hiveplotlib/viz/holoviews.py:636-637` (`.loc[relevant_edges, :]`), `src/hiveplotlib/viz/datashader.py:203-204` (`_data[tag][mask]`), and `src/hiveplotlib/hiveplot.py:4650-4655` in `to_json` (def at `:4561`; `.loc[relevant_edges, val]`). All are boolean-mask reads today, and **all move in lockstep** to positional takes (`.iloc[idx]` / `take`): with integer indices, `.loc[int_array]` is label-based (wrong rows or KeyError on any non-RangeIndex user frame, while RangeIndex-built tests stay green) and bare `df[int_array]` becomes column selection. Value-type-agnostic sites are verified-not-broken, not rewritten: `Edges.copy()`'s deepcopy (`src/hiveplotlib/edges.py:244-256`; docstring wording checked), the matrix-cell shared reference (`src/hiveplotlib/hiveplot_matrix.py:693`), and the reset/pop bookkeeping (`src/hiveplotlib/hiveplot.py:782-835`).

### Workstream 2 — reduction-aware chunked rasterization (datashader backend)

- **List-of-arrays accumulation followed by `np.concatenate` + `pd.DataFrame` construction**, found at `src/hiveplotlib/viz/datashader.py:181-234` (the transient concat copy). Replace with per-`(g1, g2, tag)`-chunk `cvs.line()` calls and a **reduction-aware** aggregation (additive for count/sum; elementwise OR — or max — for `any`, since summing boolean `any` rasters produces counts; mean accumulates sum + count and divides at end; max/min are cheaply combinable via elementwise max/min, supported in the streamed path or left in the fallback bucket per a stated implementer decision; var/std and exotic reductions do not sum per-chunk rasters, see Workstream 2 done-when). Drop the chunk's curves array after each `cvs.line()` call.
- **Existing density-correction division** at `src/hiveplotlib/viz/datashader.py:242-247` already implements the divide-at-end pattern that the mean path reuses (accumulate sum + count, divide once). Reduction-aware streaming generalizes this existing logic rather than inventing it.
- **`pd.concat([hive_plot.axes[axis_id].node_placements for axis_id in hive_plot.axes])`** at `src/hiveplotlib/viz/datashader.py:386-388`. Replace with per-axis `cvs.points()` aggregation; concatenate rasters rather than DataFrames (node points are count-based, so the additive path applies directly).
- **Same-shape accumulation patterns in `datashade_hive_plot_mpl`** at `src/hiveplotlib/viz/datashader.py:445-631` inherit the streaming refactor through its two helpers (`datashade_edges_mpl`, `datashade_nodes_mpl`); the wrapper itself does not duplicate the accumulation logic.
- **HivePlotMatrix datashader path** at `src/hiveplotlib/viz/hiveplot_matrix.py:40-59, 196-218, 438-464`. The matrix calls `datashade_hive_plot_mpl` per cell. Each cell internally rasterizes its own chunks now; per-cell streaming benefits flow up automatically. No matrix-level loop change needed, but the matrix-level peak depends on cell peak, so a memory test at the matrix level is also wanted.

### Workstream 3 — fused build + internal streaming

Mostly net new: a fused build-and-rasterize path that lives alongside (does not replace) the existing two-stage `construct_curves` → rasterize flow.

- **`construct_curves`** at `src/hiveplotlib/hiveplot.py:1381+` persists every curve on the object (`hive_plot_edges[...][tag]["curves"]`). The fused path is a new code route that, per chunk, samples the curves for that chunk only, hands them to the Workstream 2 per-chunk aggregator, and discards them, never persisting all curves. This is the resident-memory unlock. It does not delete `construct_curves`; the two-stage path remains the default for the common in-RAM case and the equivalence baseline.
- **Per-chunk metadata carry.** The fused path must retain each chunk's metadata column so the metadata-coloring trick survives (it does, as long as per-chunk metadata is held alongside the per-chunk curves). No source site is replaced here; this is a constraint on the new path.

### Workstream 4 — narwhals at input boundary

In-scope pandas-specific operations (those touching user-provided dataframes or doing internal frame work that polars/modin/Dask could carry):

- **`NodeCollection.__init__`** at `src/hiveplotlib/node.py:132-161`. Includes `data.copy()`, `data.columns`, `self.data.index.to_numpy()`, `self.data[col].duplicated(...)`. Replace with narwhals-wrapped operations; `NodeCollection.data` property returns whatever the user passed (or pandas when constructed from numpy).
- **`NodeCollection.check_unique_ids`** at `src/hiveplotlib/node.py:163-173`. Currently `np.unique(self.data[col].values)`. Replace with narwhals `is_duplicated` / row count.
- **`NodeCollection.create_partition_variable`** at `src/hiveplotlib/node.py:188-277`. Heavy lift: `pd.cut` and `pd.qcut` semantics need to be preserved through narwhals. Narwhals coverage of `cut` / `qcut` is partial; this is the most uncertain piece of Workstream 2. The `(-inf, c0]` left-open semantics with `[-inf, *cutoffs, inf]` will likely need an adapter that falls back to pandas inside the narwhals wrapper if the underlying frame lacks `cut`. Document the fallback path in the docstring.
- **`Edges._validate_edge_data`** at `src/hiveplotlib/edges.py:126-187`. The validation reads `.columns`, normalizes np arrays via `pd.DataFrame`, and stores the result. Replace `.columns` access and column-existence checks with narwhals; the numpy-input path can stay pandas internally (we're creating a default frame to hold the array; no user dataframe involved).
- **`Edges.add_edges`** at `src/hiveplotlib/edges.py:211-238`. The `pd.concat` is the issue. Replace with narwhals `concat`.
- **`Edges.export_edge_array`** at `src/hiveplotlib/edges.py:253-281`. Uses `.loc[:, [...]].to_numpy()` and `np.vstack`. Stay pandas-internal where convenient; the output is numpy regardless. (Narwhals `select(...).to_numpy()` is the equivalent; treat as a refactor for consistency rather than a load-bearing change.)
- **`HivePlot.add_edge_ids`** at `src/hiveplotlib/hiveplot.py:962+`. Uses `node_placements.loc[..., unique_id_column].to_numpy()` plus `np.isin`. Replace with narwhals `is_in` if the underlying frame supports it; fall back to `to_numpy() + np.isin` otherwise.
- **The datashader metadata-extraction path** at `src/hiveplotlib/viz/datashader.py:201-222`. The `hive_plot.edges._data[tag][mask].drop(columns=[...])` and column-presence check pass through narwhals.
- **The datashader node-metadata path** at `src/hiveplotlib/viz/datashader.py:393-407`. `node_placements.columns`, `node_placements.loc[:, columns]`, and the `astype(np.float32)` calls pass through narwhals.

Out-of-scope pandas-specific operations (kept as pandas for documented reasons, listed in Holdouts):

- **`nx.to_pandas_edgelist`** in `src/hiveplotlib/converters.py:41`. Networkx output is pandas; converting through narwhals adds no value because the next thing we do with it is hand it to `Edges`. Holdout.
- **Legacy `dataframe_to_node_list`** at `src/hiveplotlib/node.py:430-450`. Independently flagged for deprecation; do not touch in this plan. Holdout (separate workstream/plan).
- **`Node.add_data`** at `src/hiveplotlib/node.py:58-73`. Dict-based, no dataframe ops. Out of scope by definition.
- **All `viz/{bokeh,plotly,holoviews,matplotlib}.py` paths** that consume pandas frames mid-pipeline. Those backends are downstream of `add_edge_ids` and `construct_curves`; once those produce numpy curves arrays, the viz backends do not see user dataframes. Out of scope.
- **`hiveplot_matrix.py` partition / variable-sweep construction**. Construction-time dataframe ops that read `NodeCollection.data` columns; narwhals coverage on the read side will fall out for free once `NodeCollection` is narwhals-backed at the input boundary. List in audit but do not separately tweak.

### Workstream 5 — Dask + cuDF/GPU passthrough

Mostly net new / verify-rather-than-rewrite. The point of Workstream 5 is "Dask and cuDF DataFrames flow through Workstreams 1-4 with no path-specific code in user-facing methods." If Workstream 4 routes everything through narwhals correctly, the membership storage from Workstream 1 already chose its structure, and the fused build from Workstream 3 streams per-chunk, the foreign-engine path is mostly free. Specific spots to verify and replace:

- **`HivePlot.add_edge_ids` `np.isin`** at `src/hiveplotlib/hiveplot.py:962+`. Narwhals `is_in` should compile to a Dask-supported / cuDF-supported `isin` on those backends. Verify rather than re-implement.
- **Dask non-materializing membership storage.** Workstream 1 chose the in-RAM sparse integer-index structure; Workstream 5 supplies its Dask equivalent at the same `__store_edge_ids` site (`src/hiveplotlib/hiveplot.py:893`/`:958`). For Dask input, materializing a per-edge bool array (or even a full integer-index array) into RAM defeats the out-of-core story (a 1B-edge graph forces a 1B-element array per `(axis_pair, tag)`). The Dask path keeps the membership as a Dask Series, a per-tag boolean column on the edge frame kept as Dask, or an equivalent non-materializing structure. The pandas/polars case keeps the Workstream 1 in-RAM integer-index storage. Structural choice is an implementation decision; the constraint is no per-edge materialization into RAM for Dask.
- **Fused build chunk iteration** (post-Workstream 3). The fused path's per-chunk loop reads pandas-shaped data and produces per-chunk numpy curves. Verify the iteration accepts Dask `.itertuples()` / partition-wise `.compute()` semantics (and the cuDF equivalent) or document the materialization point per partition.
- **`datashade_edges_mpl` per-chunk loop** (post-Workstream 2). `cvs.line()` accepts Dask and cuDF DataFrames natively; verify the chunk DataFrame produced by streaming is wrappable in a small Dask/cuDF DataFrame when the input was Dask/cuDF, and falls back to pandas when the input was pandas.
- **cuDF/GPU rasterization path.** datashader's CUDA path runs `cvs.line` over cuDF / dask-cudf and produces cupy aggregates. Once Dask passthrough works, cuDF is a small marginal addition (narwhals dispatches to cuDF; datashader handles the GPU rasterization). The aim, per the maintainer framing, is to support fat-GPU users "without going deep on cuDF" — narwhals does the dispatch. Add a smoke test verifying datashader + narwhals version support for the cuDF path against the pinned versions.

## Default justifications

Workstream-by-workstream new defaults.

- **Workstream 1 (membership storage)** introduces no user-facing parameters (internal storage-structure change). `relevant_edges` is an internal attribute.
- **Workstream 2 (reduction-aware rasterization)** introduces one new user-facing parameter on the datashader entry points (see "Workstream 2 default: streaming threshold" below). The single-shot path stays the default.
- **Workstream 3 (fused build)** introduces no new public parameter at planning time; it routes through the Workstream 2 streaming opt-in. If the fused path needs its own opt-in distinct from the rasterization threshold, that surfaces to api-critic at implementation and routes back here via `amend-plan`.
- **Workstream 4 (narwhals)** introduces no new parameters at the public surface; it changes what `data` parameters accept (`pd.DataFrame` → any narwhals-supported frame), and the return semantics of `NodeCollection.data` / `Edges.data` properties are preserved (the user gets back what they passed in).
- **Workstream 5 (Dask + cuDF/GPU)** introduces **no** new user-facing parameters: Dask and cuDF both flow through narwhals dispatch with no opt-in (Path A, 2026-06-05; cascaded into this section 2026-07-03).

### Workstream 2 default: streaming-vs-single-shot threshold

Decision: the datashader entry points gain a parameter (name audited below as `stream_chunk_threshold`) defaulting to `None`, meaning "auto-decide by edge-set size." The **single-shot path stays the default for the common case** (plenty of datashader use is far below any memory wall), and the streamed path is the opt-in / escape hatch for scale.

- Default `None` → auto threshold by edge count: below the auto threshold, single-shot; at or above it, stream. The user can force either way (a numeric threshold, or an explicit force-on / force-off; exact type resolved in the naming audit below at implementation).
- **var/std and other non-additive reductions are always single-shot-or-delegate regardless of the threshold**, because per-chunk raster summation is mathematically wrong for moment-based reductions (the existing gallery `examples/datashading_statistical_summaries_of_metadata.ipynb` uses `ds.var` twice and `ds.mean` once; naive per-chunk summation would silently regress those figures). The streamed path applies only to reductions with a cheap, exact combine: additive (count, sum), elementwise OR (any), sum-plus-count (mean), and optionally elementwise max/min (max, min) per the implementer's stated classification.
- The single-shot path doubles as the equivalence / regression baseline: every streamed-path result must match its single-shot counterpart within tolerance (exact for count/sum/any; division-tolerance for mean).

Rationale for default-single-shot: the memory win only matters above a memory wall most users never hit, and the single-shot path is the proven, already-shipped behavior. Defaulting to stream would slow the median small-graph case and risk subtle aggregation drift on every plot for a benefit only large-graph users see. Loud opt-in (or size-triggered auto) keeps the common case untouched and the scale case reachable.

### Workstream 5 default: none (Path A)

`use_dask` was dropped entirely on 2026-06-05 (Path A; see the "Removed parameter (Workstream 5)" amendment). Inference-from-frame-type is the shipped behavior: narwhals detects Dask and dispatches, uniform with every other engine. The disclosure the dropped gate used to carry lives in the Dask gallery notebook, Example 4's inline footgun comments, and the failure-point reraise (see WS5 done-when). This subsection previously specified the `use_dask: Optional[bool] = None` raise-on-`None` default; that text was cascaded out 2026-07-03 (historical record: the 2026-05-17 amendment and the api-critic planning-mode take, both preserved verbatim).

Worth flagging: `bezier_xy_with_nans`'s `dtype=np.float64` default (`src/hiveplotlib/utils.py:256`). Library code already passes `dtype=np.float32` explicitly at every call site (`hiveplot.py:1024, 1037, 1046, 1055`). Flipping the kernel default is cosmetic and unrelated to scaling work; called out here per the brief but explicitly not in any workstream.

### Workstream B default: narwhals as a core dependency (not an extra)

Decision: narwhals is added as a **core dependency** in `pyproject.toml`, pinned `>=1.x,<2` (specific minor pin chosen at implementation time within that bound range; the major bound itself is re-derived at WS4 start per pin-time checklist item (c), 2026-07-03 — the bound was fixed in 2026-05 and narwhals releases fast).

Rationale:

- Narwhals is pure-Python with no compiled extensions, no platform-specific binaries, and no threading layer. The risk profile that justified keeping numba optional (compiled wheels, segfaults across platforms, parallelism complexity) does not apply.
- Optional narwhals would require conditional code paths that CI either tests twice (matrix doubling) or doesn't test (silent rot). Core narwhals is paradoxically the lower-CI-complexity choice.
- The wiki's documented "minimal base deps: matplotlib, numpy, pandas only" stance (`wiki/wiki/sources/hiveplotlib-python.md:79-84`) is a deliberate deviation. The post-task research-liaison pass should update that source page to acknowledge narwhals as a pure-Python dispatcher in the core deps.
- A brief design note in `CLAUDE.md` documents the abstraction layer so future maintainers don't relitigate the decision.

### Workstream B default: narwhals usage pattern is pass-through, not opinionated-internal

Decision: when a user passes a polars frame, internal ops dispatch to polars; when they pass pandas, ops dispatch to pandas; when they pass Dask, ops dispatch to Dask. The library does **not** convert to a canonical internal frame type (does not always-convert-to-polars-internally regardless of input).

Rationale:

- The library's source-dataframe operations (`np.isin` axis-pair classification, boolean mask + drop, `pd.cut` / `pd.qcut`, `pd.concat`, `np.unique`) sum to roughly 10-50 ms on a 1M-edge graph in pandas vs 2-10 ms in polars. Total HivePlot construction time is 1-3 seconds, dominated by Bezier curve construction. Polars-internal would save roughly 2% of total wall time; not enough to justify the architectural complexity.
- "Always-polars internally" breaks the Dask and cuDF/GPU stories: converting Dask to polars defeats the out-of-core purpose; converting cuDF to polars pulls data off the GPU. Either you accept that breakage (excludes Workstream 5 entirely) or you special-case those backends, which is the complexity cost without the bottleneck win.
- Pass-through respects the user's choice of where their data lives (RAM vs out-of-core vs GPU) when the library doesn't gain enough from overriding that choice to justify the move.

Confirmation against the API usage examples: Example 2 ("polars in, polars out") and Example 3 ("pandas in, pandas out") already reflect pass-through behavior accurately; no wording change needed.

## Naming audit

### Workstream 1 (membership storage)

No new user-facing names. `relevant_edges` is an existing internal attribute; only its stored value type changes (boolean mask → integer index array). No public parameter, method, or class added.

### Workstream 2 (reduction-aware rasterization)

One new user-facing parameter on the datashader entry points (`datashade_edges_mpl`, `datashade_nodes_mpl`, `datashade_hive_plot_mpl`). Proposed name: **`stream_chunk_threshold`**.

- The dominant adjacent vocabulary is datashader / dask, where "chunk" and "partition" name a unit of streamed work and "stream" names the incremental-aggregation idea. "Threshold" reads as "the size above which behavior switches," which is exactly the semantics (auto-switch to streaming at or above N edges). A user scanning the datashader signature recognizes all three tokens.
- Type and default: `Optional[int] = None`. `None` → auto-decide by edge-set size (library picks the switch point). An explicit `int` forces the switch point (set very high to force single-shot, very low to force streaming). Resolved at implementation whether a sentinel / bool is also wanted for unconditional force-on/force-off; if a richer type than `Optional[int]` is chosen, it routes back here via `amend-plan` for a fresh audit.
- Alternatives considered and rejected: `chunked=True` (a bare bool hides the size-trigger nuance and invites "why is my small plot streaming?"); `use_streaming` (a bare opt-in name that loses the threshold semantics, and streaming-vs-not is genuinely a size question, not a yes/no preference); `max_edges_in_memory` (over-promises a hard memory cap the parameter does not enforce). `stream_chunk_threshold` keeps the size-trigger meaning legible.
- Constraint the name must not obscure: var/std are **always** single-shot-or-delegate regardless of this parameter. The docstring states that the threshold governs additive / sum-plus-count reductions only and that moment-based reductions ignore it.

### Workstream 3 (fused build)

No new user-facing name at planning time. The fused path is an internal code route selected by the Workstream 2 `stream_chunk_threshold` alone; foreign frames (Dask, cuDF) ride the same single shared streaming policy (interaction §1), with no engine-specific selector (Path A; cascaded 2026-07-03). If implementation finds the fused path needs its own distinct opt-in, the new name routes back here via `amend-plan` for an audit before landing.

### Workstream 4 (narwhals)

No new parameter names. The `data` parameter to `NodeCollection.__init__` and `Edges.__init__` keeps its name; what changes is the accepted type annotation (was `pd.DataFrame`, becomes `IntoDataFrame` from narwhals). The docstring grows a note: "Accepts any narwhals-supported frame (pandas, polars, modin, pyarrow, Dask, cuDF). Returns whatever you passed in the same library (round-trip contract); the library does not coerce frame types behind your back."

`IntoDataFrame` is narwhals's own name for this type; using it preserves user vocabulary (anyone already using narwhals or polars recognizes it; anyone using pandas-only doesn't need to know it exists because the docstring tells them "pandas works"). No friction expected.

Round-trip contract docstring sketch on both `NodeCollection.data` and `Edges.data` properties (per planning-mode api-critic Q2, must-fix):

> Returns the frame in the same library you passed in. Constructing with `pd.DataFrame` returns `pd.DataFrame`; constructing with `pl.DataFrame` returns `pl.DataFrame`; constructing with a `np.ndarray` (Edges only) returns `pd.DataFrame` (the library's default for raw-array input, since no original frame existed). The library does not coerce frame types behind your back.

The ndarray-input-becomes-pandas asymmetry is the one named exception and is documented in the docstring (not a bug, a documented carve-out).

### Workstream 5 (Dask + cuDF/GPU)

No new user-facing names (Path A, 2026-06-05; cascaded into this section 2026-07-03). Dask takes **no** opt-in parameter, matching cuDF: both ride on narwhals dispatch, and neither warrants a gate. The earlier `use_dask` audit (the parameter name, its `Optional[bool]` type, and the `None`/`True`/`False` × Dask/non-Dask branch semantics) is superseded; the `backend="dask"` and `lazy=True` rejections are moot now that no parameter is added at all. Historical record: the 2026-05-17 amendment and the api-critic planning-mode take (preserved verbatim above); the api-critic planning-mode re-review endorses the zero-parameter surface.

### Internal-only

Internal narwhals-related helper names (`_to_narwhals(df)`, `_from_narwhals(nw_df)`) are internal module/package names and out of scope for this audit per template guidance.

## API usage examples

### Proposed (from planner / Orchestrator)

```python
# Example 1: Workstream 2 (reduction-aware chunked rasterization).
# User has a 10M-edge graph, wants a datashaded count/sum/mean plot.
# Default (stream_chunk_threshold=None): the library auto-decides. Below the
# auto threshold it runs the proven single-shot path (no behavior change for
# small plots); at/above it, it streams per-(axis_pair, tag) chunk, dropping
# each chunk's curves after rasterizing. Peak transient memory drops from
# O(total_edges * num_steps) to O(largest_chunk * num_steps). The common case
# is untouched; large-graph users get the streamed path for free.
import networkx as nx
from hiveplotlib import HivePlot, networkx_to_nodes_edges
from hiveplotlib.viz.datashader import datashade_hive_plot_mpl

g = nx.karate_club_graph()  # in practice: ogbn-arxiv at 169k nodes, 1.16M edges
nodes, edges = networkx_to_nodes_edges(g)
hp = HivePlot(nodes=nodes, edges=edges)
# ... partition + construct_curves ...

# default: auto-decide (single-shot for small, stream for large)
fig, ax, im_nodes, im_edges = datashade_hive_plot_mpl(hp)

# force streaming regardless of size (escape hatch for scale):
fig, ax, im_nodes, im_edges = datashade_hive_plot_mpl(hp, stream_chunk_threshold=0)

# Note: with a var/std reduction, the streamed path does NOT apply; the library
# stays single-shot (or delegates partial-aggregate combination to datashader
# over a Dask/cuDF frame) because per-chunk raster summation is wrong for
# moment-based reductions. `stream_chunk_threshold` is ignored for var/std.


# Example 2: Workstream 4 (narwhals). User has a polars DataFrame; library accepts it.
import polars as pl
from hiveplotlib import Edges, NodeCollection

node_df_pl = pl.read_parquet("nodes.parquet")
edge_df_pl = pl.read_parquet("edges.parquet")

nodes = NodeCollection(data=node_df_pl, unique_id_column="id")
edges = Edges(data=edge_df_pl, from_column_name="src", to_column_name="dst")

# nodes.data is polars (round-trip preserved).
type(nodes.data)  # polars.DataFrame
type(edges.data)  # polars.DataFrame


# Example 3: Workstream 4 (mixed pandas + polars). User wants pandas in, polars out
# is not a goal; what is a goal is "polars in, polars out, no surprise pandas
# materialization mid-pipeline."
import pandas as pd
import polars as pl
from hiveplotlib import Edges, NodeCollection

# pandas-typed user keeps pandas behavior, no migration needed.
nodes_pd = NodeCollection(data=pd.read_csv("nodes.csv"), unique_id_column="id")
type(nodes_pd.data)  # pandas.DataFrame


# Example 4: Workstream 5. User has a Dask DataFrame; plain passthrough, no
# opt-in parameter (Path A): narwhals detects Dask and dispatches, uniform
# with every other engine. Two Dask-only footguns to know at the call site:
# - per-partition curve materialization: ~largest_partition_rows * num_steps
#   * 4 bytes must fit in RAM; repartition before passing to hiveplotlib.
# - nodes are the in-memory side: axis placement materializes each axis's
#   node subset into pandas, so node data must fit in RAM; there is no Dask
#   sort/shuffle inside hiveplotlib. (Corrected 2026-07-05: supersedes this
#   comment's earlier shuffle/sort-upstream warning, which WS5 resolved as
#   no-shuffle; see the WS5 post-impl disposition amendment.)
# Full guidance: examples/creating_hive_plots_from_dask.ipynb.
import dask.dataframe as dd
from hiveplotlib import Edges, NodeCollection

edge_ddf = dd.read_parquet("huge_edges/*.parquet")  # 100M edges
node_ddf = dd.read_parquet("huge_nodes/*.parquet")

nodes = NodeCollection(data=node_ddf, unique_id_column="id")
edges = Edges(data=edge_ddf, from_column_name="src", to_column_name="dst")

hp = HivePlot(nodes=nodes, edges=edges)
# ... rest of the pipeline runs Dask-aware ...


# Example 5: Workstream 5 (cuDF / GPU). Fat-GPU user, no extra ceremony beyond
# passing a cuDF frame. narwhals dispatches to cuDF; datashader's CUDA path
# rasterizes on the GPU (cvs.line over cuDF -> cupy aggregates). No `use_cudf`
# parameter: cuDF carries no partition-by-partition materialization risk, so it
# needs no opt-in gate. The single requirement is a supported datashader+narwhals
# version (verified by a smoke test, see Workstream 5 done-when).
import cudf
from hiveplotlib import Edges, NodeCollection

node_gdf = cudf.read_parquet("nodes.parquet")
edge_gdf = cudf.read_parquet("edges.parquet")

nodes = NodeCollection(data=node_gdf, unique_id_column="id")
edges = Edges(data=edge_gdf, from_column_name="src", to_column_name="dst")

hp = HivePlot(nodes=nodes, edges=edges)
# nodes.data / edges.data round-trip to cuDF (data stays on the GPU).
type(nodes.data)  # cudf.DataFrame
```

### API Critic's take (planning mode)

Reviewed Examples 1-4 as a user typing the class names and hitting tab against polars and Dask frames. Three orchestrator questions answered, then per-example friction, then the cross-cutting pattern.

**Q1 — "Infer from frame type" vs. opt-in for Dask (Workstream C):** [must-fix]

Recommend opt-in via `use_dask: Optional[bool] = None` with the default `None` meaning "raise if a Dask frame is passed without an explicit opt-in." Reasoning:

- The two failure modes are asymmetric. A user with a small Dask frame for dev silently paying Dask overhead is a wasted-cycles annoyance (fixable by `df.compute()`). A user with a large Dask frame who didn't realize it was Dask, hitting `cvs.line()` on partitions that materialize one-by-one into pandas inside narwhals... can OOM on a single partition or, worse, succeed in 100x the wallclock with chunked I/O thrash and no warning. The OOM/wallclock mode is the silent one; surfacing it explicitly is the safer default.
- The plan's own "Naming audit" already notes `use_dask=True` with a pandas frame should raise `TypeError`, which means the parameter is already in the design space. The choice is whether the *default* is "infer" or "explicit." Explicit is cheaper to walk back later (loosen `None` to mean "infer" in a future minor) than the reverse (silent inference users came to depend on, now you'd ship a deprecation).
- Workstream C's done-when says "no path-specific code in user-facing methods" — that goal is still met with an opt-in: the parameter lives at `NodeCollection.__init__` and `Edges.__init__`, the rest of the pipeline is unchanged. Opt-in is a one-line user cost (`use_dask=True`) for a much louder safety net.
- The plan's Naming audit rejects `backend=...` and `lazy=True` for good reasons; `use_dask` is the right name. Reject the "don't add a parameter" recommendation in favor of the fallback (auto-detect via `None`) the plan already names.

Concrete recommendation: `use_dask: Optional[bool] = None`, with `None` meaning "if a Dask frame is detected, raise `TypeError` instructing the user to pass `use_dask=True` explicitly." This is the loud-failure default; users who want auto-detect can pass `use_dask=True` once they've understood the cost.

**Q2 — `.data` round-trip guarantee, documented or not:** [must-fix, document it]

Document it. Once `NodeCollection(data=polars_df).data` returns polars (per the plan), users will write `nc.data.filter(pl.col("x") > 0)` and ship it. If a future refactor switches `.data` to "always pandas because internal storage simplified," every downstream polars user breaks silently. The right time to commit to a contract is when the surface is born; the right place is the docstring on `.data` and `.__init__`.

Suggested docstring sketch for both `NodeCollection.data` and `Edges.data`:

> Returns the frame in the same library you passed in. Constructing with `pd.DataFrame` returns `pd.DataFrame`; constructing with `pl.DataFrame` returns `pl.DataFrame`; constructing with a `np.ndarray` (Edges only) returns `pd.DataFrame` (the library's default for raw-array input). The library does not coerce frame types behind your back.

The np.ndarray-input-becomes-pandas asymmetry is real and worth naming in the docstring rather than hoping nobody notices. It is a documented exception, not a bug.

**Q3 — Cross-example pattern:** [worth-discussing]

Yes, two patterns:

- *Frame-library introspection.* Across Examples 2-4 the user immediately wants to confirm "what did the library actually store?" — Example 2 does `type(nodes.data)`, Example 3 does the same. A `NodeCollection.frame_library` property returning a string like `"pandas"` / `"polars"` / `"dask"` is more durable than `type(...)` and cheaper to document. Not blocking; a quality-of-life add that should be considered alongside Workstream B.
- *Convenience converter.* A `to_pandas()` method on `NodeCollection` and `Edges` would help users who built a `HivePlot` from polars but want to drop into pandas for a one-off `.describe()` or for an integration with a pandas-only downstream tool. Cheap to add (`nw.from_native(self.data).to_pandas()`), and signals "you can always escape to pandas if you need to" which is a comforting safety property for the narwhals adoption. Strictly optional; flagging here so the code-engineer can add it in passing rather than as a follow-up.

Recommend adding `frame_library` as a property in Workstream B; defer `to_pandas()` to a worth-discussing follow-up unless code-engineer finds it cheap.

**Per-example walkthrough:**

- *Example 1 (Workstream A):* Agreed. The "no code change for the user" punchline is the right framing; this example is purely a memory-characteristics note in the docstring. The note in `datashade_hive_plot_mpl`'s docstring should name the per-chunk shape concretely (`O(largest_chunk * num_steps)` is the right magnitude).

- *Example 2 (Workstream B, polars in/out):* Mostly agreed. One friction: `unique_id_column="id"` and `from_column_name="src"`/`to_column_name="dst"` are required kwargs the user is expected to know. That is unchanged from today's pandas surface and out of scope here, but the docstring should make clear that polars column-name semantics are identical (no `.` vs `_` translation, no case folding). Suggested addition to the planner's example: a single line `# polars column names round-trip; no transformation` before the constructors. [nit]

- *Example 3 (mixed pandas + polars):* Agreed. The example correctly shows pandas users see no change. The Q2 docstring fix above covers the "is it always pandas or sometimes polars" confusion this example might raise.

- *Example 4 (Workstream C, Dask):* Per Q1 above, this example must change. Suggested rewrite:

  ```python
  # Workstream C. User has a Dask DataFrame, opts in explicitly.
  import dask.dataframe as dd
  from hiveplotlib import Edges, NodeCollection

  edge_ddf = dd.read_parquet("huge_edges/*.parquet")  # 100M edges
  node_ddf = dd.read_parquet("huge_nodes/*.parquet")

  # Explicit opt-in. Without `use_dask=True`, the constructor raises
  # TypeError to surface "this will materialize partitions one-by-one;
  # are you sure?" Default is loud-failure rather than silent inference.
  nodes = NodeCollection(data=node_ddf, unique_id_column="id", use_dask=True)
  edges = Edges(data=edge_ddf, from_column_name="src", to_column_name="dst",
                use_dask=True)

  hp = HivePlot(nodes=nodes, edges=edges)
  # ... rest of the pipeline runs Dask-aware ...
  ```

  The example should also include a "what error do I see if I forget?" snippet:

  ```python
  # Forgetting `use_dask=True` raises early:
  # >>> NodeCollection(data=node_ddf, unique_id_column="id")
  # TypeError: Detected a Dask DataFrame but `use_dask` is not set.
  #     Pass `use_dask=True` to opt in to the Dask passthrough. Note that
  #     hiveplotlib will materialize partitions one-by-one inside the
  #     rasterization loop; ensure each partition fits in memory.
  ```

  This is the loud failure mode the orchestrator's Q1 was asking about. If `use_dask=True` is passed but Dask is not installed, the error should name the missing extra (`pip install hiveplotlib[dask]` or whatever the chosen install marker is).

**Other concerns:**

- *Partially set-up Dask (`dask.delayed` vs `dask.DataFrame`):* The plan does not name what happens if a user passes a `dask.delayed` or a Dask `Bag` or a `dask.array`. Recommend: narwhals's `from_native` will raise on non-DataFrame Dask objects; surface that error with a clarifying message ("hiveplotlib accepts `dask.dataframe.DataFrame`; got `dask.delayed.Delayed`. Materialize first with `.compute()` or wrap in `dd.from_delayed(...)`."). [should-fix in Workstream C]

- *`create_partition_variable` and `cut`/`qcut` fallback:* Plan correctly flags this as the most uncertain piece. From an API perspective: if the fallback materializes a polars frame through pandas internally, the *returned* partition column should still land in the user's original frame type. A polars user calling `create_partition_variable` expects the resulting `nc.data` to remain polars; if the internal fallback silently makes it pandas, that breaks the Q2 round-trip contract. Add to Workstream B done-when: "if `create_partition_variable` falls back to pandas for `cut`/`qcut`, the result is converted back to the user's original frame type before being assigned to `self.data`." [must-fix in Workstream B]

- *Variable-naming consistency at the boundary:* Today `NodeCollection(data=...)` and `Edges(data=...)` both use `data`. Good. The narwhals docstring note should use the same word ("frame") consistently across both; the plan says "any narwhals-supported frame" in one place and "Returns whatever you passed" in another — both fine, just make sure both classes' docstrings say it the same way. [nit]

- *The `.data` getter on `Edges` already does dict-vs-singleton dispatch.* Worth verifying the round-trip contract holds across the multi-tag dict case too: a user passing `data={"tag_a": polars_df_a, "tag_b": polars_df_b}` should get back a dict-of-polars, not a dict-of-pandas. Add a multi-tag polars test case to Workstream B's test matrix. [should-fix, test coverage]

**Recurring pattern summary:**

The strongest cross-cutting concern is documenting the round-trip contract (Q2). Without it, the API silently grows a "polars in, pandas out under some conditions" mode that users will discover only when they hit it. The `use_dask` opt-in (Q1) is the second-strongest; opt-in is asymmetrically cheaper to relax later than to tighten. The `frame_library` property (Q3) is a nice-to-have. The `create_partition_variable` round-trip preservation (under Other concerns) is must-fix and ties Q2 to a concrete done-when bullet.

### API Critic — planning-mode re-review (post-`use_dask` removal)

**Date:** 2026-06-05
**Scope:** Re-review of the amended Workstream 5 surface after the maintainer's post-grill Path A call dropped `use_dask` entirely (see "Removed parameter (Workstream 5)" amendment and `## Alignment (grill)` Wave 3). This is additive to the historical planning-mode take above (which reflects the now-superseded opt-in decision via the 2026-05-17 Q1 must-fix); that take is preserved as-is. The grill explicitly asked: should there be ANY parameter on this surface? I walked the zero-parameter surface as a Dask user and a cuDF user, and re-read Examples 4 and 5.

**Bottom line: the zero-parameter surface is the right call, and I endorse it.** Reversing my own 2026-05-17 Q1 is correct on the evidence the grill surfaced. My original asymmetry argument rested on the silent-OOM mode being the dangerous one; Wave 2 and Wave 3 dismantled that premise on two independent grounds I had not weighted: (a) the `stream_chunk_threshold` axis (orthogonal, engine-independent) already serves "chunk regardless of input type," so `use_dask` was never the lever for the internal-streaming concern I implicitly conflated it with; and (b) WS3's per-partition geometry streaming tames the `num_steps × 4 bytes` multiplier *within* a partition, narrowing the residual risk to a user whose raw partition rows don't fit in RAM, which is pathological partitioning the Dask user already owns. Once OOM-safety falls away, my Q1 stood only on reversibility, which the maintainer explicitly down-weighted as ordinary API risk. Surface consistency then dominates: pandas / polars / pyarrow / cuDF all "just work" with no gate; a mandatory `use_dask=True` (whose only valid value is `True`) singled Dask out for ceremony that bought nothing the notebook disclosure doesn't already carry. Detect-and-passthrough reads as the clean, uniform surface, not a footgun. A user typing `Edges(data=...)` against a Dask frame gets the same mental model as against every other engine, which is exactly the property the round-trip contract (Q2) is selling. **Agreed on Path A.**

That said, the removal leaves four specification gaps the cascade did not fully close. None reverses Path A; all are about making the now-silent surface honest at its real failure points.

**Concern 1 — the missing-Dask `ImportError` trigger condition is under-specified, and "Dask frame passed but Dask not installed" is close to a contradiction in terms. [worth-discussing]**

WS5 done-when keeps: "passing a Dask frame when Dask is not installed raises `ImportError` naming the missing extra." Walk it literally: to *have* a `dask.dataframe.DataFrame` object in hand to pass, the user already imported Dask, so Dask is installed in that process. The genuine trigger is not "user holds a Dask frame in an env without Dask." It is one of: (a) hiveplotlib gates its *narwhals-Dask dispatch code path* behind an optional `hiveplotlib[dask]` extra that pulls a pinned Dask (so the user's ambient Dask works for construction but hiveplotlib refuses to dispatch without the extra installed), or (b) the rasterization path needs `dask-cudf` / a datashader-Dask integration the bare `dask.dataframe` install doesn't provide. The error's *trigger condition* (the thing that must be true for it to fire) is unstated, and it determines whether the message even makes sense. Today's missing-extra convention (`converters.py:15-17`, `viz/datashader.py:23-28`: a top-of-module `try/import/except ImportError` naming `pip install hiveplotlib[<extra>]`) fires at *import* of the optional module, not at frame-detection time, which doesn't map cleanly onto "I detected your frame is Dask." Suggested change: WS5 done-when should specify *what* import or capability is actually absent when this `ImportError` fires (the `hiveplotlib[dask]` extra's contents, or the datashader-Dask integration), and pin the trigger to that, not to "a Dask frame was passed." If the answer is "narwhals raises its own error when it can't dispatch to an uninstalled backend," then say so and let narwhals own the message rather than promising a hiveplotlib-authored extra-naming `ImportError` that may never have a path to fire.

**Concern 2 — nothing now specifies what the user sees when a Dask computation blows up at the rasterization loop. [worth-discussing]**

This is the real cost of dropping the construction-time acknowledgment, and it is the orchestrator's explicit question. Under the old gate, a user got a loud signal at `Edges(...)` construction that materialization happens per-partition. Now the first signal a user gets that their partition sizing is wrong is whatever the kernel/datashader/Dask stack throws *deep in the rasterization loop*, potentially after a long shuffle for `place_nodes_on_axis`, far from the `Edges(...)` call site and with a traceback rooted in numpy/datashader internals rather than anything hiveplotlib-shaped. The notebook partition-sizing guidance (WS5 done-when) is the right *preventive* disclosure, but a user who didn't read the notebook gets an un-attributable MemoryError with no breadcrumb back to "repartition before passing to hiveplotlib." Suggested change: add a WS5 done-when bullet that the per-partition curve materialization point catch-and-reraise (or at minimum documents) the partition-size failure with a message pointing back to the notebook's repartition guidance, so the blow-up site is self-describing rather than a bare OOM. This is the honesty obligation the dropped gate used to discharge at construction time; it should not vanish entirely, only move to where the failure actually happens.

**Concern 3 — Example 4 should show the partition-sizing pointer inline, not just gesture at the notebook. [worth-discussing]**

The amendment says rewrite Example 4 to plain passthrough "mirroring Example 5's cuDF passthrough shape; keep a one-line pointer to the notebook's partition-sizing / shuffle guidance." Good direction. But cuDF and Dask are *not* symmetric on the one axis that matters here: cuDF carries no partition-materialization or shuffle footgun, Dask carries both. Making Example 4 a near-verbatim twin of Example 5 (the explicit framing in Example 5 today is "no extra ceremony beyond passing a cuDF frame") risks erasing the one thing a Dask user most needs to see at the call site. The deleted Example 4b at least made the materialization model visible. Suggested change: Example 4's inline comment should name the two concrete footguns (per-partition curve materialization ceiling; `place_nodes_on_axis` shuffle cost) in one line each, not just a "see the notebook" pointer, so the example carries the disclosure the gate used to. The cuDF symmetry is a trap; the surfaces are uniform in *shape* (no parameter) but not in *what the user must know*.

**Concern 4 — `_validate_edge_data`'s current `isinstance(val, (pd.DataFrame, np.ndarray))` assertion is where detection moves; the plan should name it so the Dask path doesn't trip the existing guard. [low-confidence]**

At `edges.py:164` today, validation asserts each tag's value `isinstance(val, (pd.DataFrame, np.ndarray))` and raises otherwise. A Dask (or polars, or cuDF) frame fails that `isinstance` check as written, so the WS4/WS5 narwhals refactor must replace this guard with a narwhals-aware "is this a supported frame?" check, and that exact site is where Dask detection and any missing-backend error naturally live. WS4 already lists `_validate_edge_data` in its replace-list, so this is mostly a note: confirm the narwhals predicate that replaces the `isinstance` guard is the single detection point for *all* engines (so Dask isn't special-cased back in), and that its failure message for a genuinely-unsupported object (a `dask.delayed`, a `dask.array`, a bare list) is the one the parked "clearer error for partially set-up Dask objects" amendment punted to the notebook. With `use_dask` gone, that parked item's "narwhals `from_native` raises anyway" rationale is now the *entire* safety net for malformed input; worth a one-line WS5/WS4 confirmation that the narwhals raise is reached (not pre-empted by a stale `isinstance` assert) and is legible.

**Recurring pattern (this re-review):** the through-line in concerns 1-3 is that dropping the gate was right but the gate carried *disclosure*, and disclosure has to land somewhere. The plan moved it to the notebook (correct primary home) but left three secondary surfaces thin: the at-failure-site message (concern 2), the at-call-site example comment (concern 3), and the missing-backend error's actual trigger (concern 1). None is a parameter; all are "make the silent surface self-describing at the point a confused user actually is." Endorse zero parameters; tighten the disclosure that the parameter used to carry.

### API Critic — post-implementation review

(The "API Critic's take (planning mode)" subsection above is the historical record of the planning-mode review and uses the original A/B/C workstream letters; it is preserved verbatim. The placeholders below track post-implementation invocations against the restructured 5-workstream chain.)

```
No post-impl critic invocation needed for Workstream 1 (membership storage):
internal storage-structure change to `relevant_edges`, no public surface change.

Workstream 2 (reduction-aware rasterization): review filled below, 2026-07-03.

Workstream 3 (fused build): review filled below, 2026-07-04. No
fused-path-specific opt-in name was introduced (route selection stayed on
`stream_chunk_threshold`), so the naming-audit re-invocation trigger never
fired.

Workstream 4 (narwhals): review filled below, 2026-07-04 (covers the base
workstream plus the boundary-gap follow-up: styling gathers, add_nodes,
frame_library / to_pandas fold-ins, [polars] extra, partition-label carve-out).

Workstream 5 (Dask + cuDF/GPU): review filled below, 2026-07-05. Zero-parameter
Path A surface held, so the review covers acceptance/error behavior, the
dask-native rasterization route, DaskComputationError, and the GPU
verification workflow (notebooks are the separate pending dispatch).
```

#### Workstream 2 review (2026-07-03): `stream_chunk_threshold` on the three datashader entry points

```
Status: propose
API surface reviewed: [datashade_edges_mpl, datashade_nodes_mpl,
  datashade_hive_plot_mpl, the hive_plot_viz / node_viz / edge_viz aliases,
  HivePlotMatrix reachability via viz/hiveplot_matrix.py **kwargs forwarding,
  module-level policy (_use_streamed_rasterization,
  _DEFAULT_STREAM_CHUNK_THRESHOLD, _STREAMED_REDUCTION_COMBINE_MODES),
  streamed missing-metadata-column error paths]
Concerns:
  - [worth-discussing] Explicitly forced streaming is silently ignored for
    non-streamable reductions: stream_chunk_threshold=0 with ds.var("col") on a
    10M-edge plot runs single-shot (full transient memory cost) with no runtime
    signal; the docstring carve-out is the only disclosure — at
    src/hiveplotlib/viz/datashader.py:94-95.
    Suggested change: warn only when an explicit non-None threshold would have
    streamed but the reduction has no combine mode (default None stays silent, so
    no new warning on any existing call). The plan's Default justifications
    specified docstring-level disclosure and that shipped as planned; this is a
    proposed targeted upgrade for the explicit-force case, not a disagreement
    with the justified default.
  - [worth-discussing] datashade_hive_plot_mpl's :raises ValueError: names only
    the reduction_edges missing-edge-column condition; the wrapper equally raises
    for reduction_nodes referencing a missing node metadata column (both paths,
    via datashade_nodes_mpl) — at src/hiveplotlib/viz/datashader.py:991-992.
    Suggested change: extend the :raises: line to also cover reduction_nodes /
    node placements (one line; the :raises: block is the load-bearing
    what-do-I-catch surface, and WS2 rewrote this docstring block).
  - [low-confidence] The single-shot fallback set is an exact-type allowlist, but
    the docstrings frame it by mathematical property ("moment-based and other
    non-combinable"); ds.count_cat("col") is combinable in principle yet silently
    single-shot — at src/hiveplotlib/viz/datashader.py:480-481 (also :727-728,
    :986-988).
    Suggested change: phrase the carve-out as "any reduction other than the six
    named (e.g. ds.var, ds.std, ds.count_cat) always rasterizes single-shot."
  - [low-confidence] stream_chunk_threshold=False force-streams (bool is an int;
    False == 0), the opposite of a toggle-reader's intent; damage is perf-only
    since streamed output matches single-shot — at
    src/hiveplotlib/viz/datashader.py:99-104.
    Suggested change: optional bool-rejecting TypeError in
    _use_streamed_rasterization; acceptable to leave as-is (the name does not
    invite bool, per the naming audit's rejection of chunked=True).
  - [low-confidence] The literal "1,000,000" is hardcoded in three docstrings,
    the CHANGELOG entry, and the module policy comment while the value lives in
    _DEFAULT_STREAM_CHUNK_THRESHOLD; retuning the constant (plausible after WS3's
    cumulative measurements) strands five prose sites — at
    src/hiveplotlib/viz/datashader.py:48.
    Suggested change: one comment line on the constant naming the prose sites to
    sweep when the value changes.
Test-method-coverage audit: clean — sampled the ten new tests
  (tests/datashader_test.py:542, :579, :611, :634, :659, :696, :724, :762, :779,
  :802); every test_<entry point>_* body calls its named entry point, including
  the two policy tests calling _use_streamed_rasterization directly.
```

User walkthrough record (per dispatch brief):

- **(a) Small-graph user, passes nothing:** the no-change promise is visible ("Default ``None`` auto-decides by edge count, streaming at or above 1,000,000 edges") and true; below the threshold the else-branch is the pre-WS2 single-shot code, and zero drawn items short-circuits to single-shot. Clean.
- **(b) 2M-edge user, auto-streamed:** nothing in the output or its docs surprises; the docstring states streamed output matches single-shot and names the combine algebra. There is no observable signal which path ran; acceptable given output equivalence (a debug-level log line would aid support forensics someday, but it does not warrant surface).
- **(c) Forcing each way:** discoverable from the docstring alone; all three docstrings state "pass a small value (e.g. ``0``) to force streaming or a value above your edge count to force single-shot."
- **(d) var/std user:** the carve-out is the closing sentence of the parameter docstring on all three entry points, factual and non-alarming; concerns 1 and 3 above are the residue. The streamed missing-metadata-column errors match the single-shot message verbatim on both edge and node paths (verified against tests at :779 and :802).
- **(e) Bogus values:** negative int force-streams (consistent with at-or-above semantics); a float works via comparison; ``True`` streams (matching likely intent); ``False`` is concern 4; a string raises a bare TypeError from the ``>=`` inside the private policy helper, with the kwarg visible in the user's own traceback frame. No friendly validation was planned; none is required.

Decided non-concern (the 1e-15 question from the dispatch brief): the ~1e-15 relative streamed-vs-single-shot difference for sum/mean (float addition order across chunks; count/any/max/min bit-exact) stays out of the docstring. The docstring parenthetical already discloses the mechanism (per-chunk accumulation), the difference is invisible in image space, and the default-suite equivalence tests pin the contract mechanically (exact assertion for count/any/max/min, rtol=1e-12 for sum/mean, via `_STREAMED_REDUCTION_EXACTNESS` in `tests/datashader_test.py:491-498`). "Matches single-shot output" is honest at the level a raster consumer operates; advertising ulp-level noise would manufacture concern with no actionable consequence. The durable record for anyone bisecting bit-level raster differences is that test map plus this note.

Also walked and cleared: the last-in-signature append (after `fig_kwargs`, before `**kwargs`) on all three functions cannot rebind any previously-valid positional call; the matrix path reaches the parameter through `viz/hiveplot_matrix.py`'s `**kwargs` forwarding into `datashade_hive_plot_mpl` (lines 54-59); and the single shared (non-`_nodes`/`_edges`-suffixed) parameter on `datashade_hive_plot_mpl` is the right call despite breaking that wrapper's suffix grammar — it is one shared memory policy applied against each rasterization's own item count (plan interaction §1), the forwarding is disclosed in the docstring, and per-surface control remains available by calling the two underlying functions directly.

#### Workstream 3 review (2026-07-04): fused build + ids-only rendering semantics

```
Status: propose
API surface reviewed: [BaseHivePlot.construct_curves (new resident-memory note),
  BaseHivePlot.add_edge_curves_between_axes (delegation refactor),
  BaseHivePlot._construct_edge_subset_curves (new build-without-persist core),
  viz/datashader._chunk_edge_curves (per-chunk seam),
  datashade_edges_mpl / datashade_hive_plot_mpl ids-only + mixed-state
  semantics (incl. hive_plot_viz / edge_viz aliases),
  CHANGELOG 0.29 datashader entries]
Concerns:
  - [worth-discussing] The headline wrapper's stream_chunk_threshold doc carries
    the memory half of the ids-only story (built on the fly, never persisted)
    but drops the time half (curves rebuilt on EVERY render; construct_curves()
    is the cache-it escape hatch), which lives only behind a cross-ref to
    datashade_edges_mpl — at src/hiveplotlib/viz/datashader.py:1077-1079.
    Suggested change: append the one clause "rebuilt on every render; run
    ``construct_curves()`` once to cache curves for repeated renders" to the
    wrapper's parameter doc (the wrapper and its hive_plot_viz alias are the
    scale user's most likely entry point).
  - [worth-discussing] Ids-only is now a documented, supported terminal state
    for datashader users, but every vector backend still renders it as a
    silently edge-less figure: the per-record guard skips records lacking
    "curves" and input_check only warns when hive_plot_edges is empty
    entirely — at src/hiveplotlib/viz/matplotlib.py:491 (same guard pattern in
    bokeh/holoviews/plotly).
    Suggested change: warn when a record holds "ids" but no "curves" ("edges
    specified but no curves constructed; run construct_curves()"). Pre-existing
    behavior whose traffic WS3 raises (datashader-prototype-then-matplotlib-
    final-figure handoff); consumer-code change, route via amend-plan. This is
    interaction §5's user-facing residue.
  - [worth-discussing] A mixed-state render silently fills ids-only chunks at
    DEFAULT geometry alongside persisted non-default geometry; pre-WS3 that
    state raised KeyError: 'curves', which forced the user to pick the fill
    parameters — at src/hiveplotlib/viz/datashader.py:246-250.
    Suggested change: one clause in the stream_chunk_threshold docs ("subsets
    with persisted curves render as stored, including non-default geometry;
    the rest build at defaults"); a warning keyed on the mixed condition alone
    would reach exactly the affected user, but the doc clause may suffice given
    the narrow population (non-default add_edge_curves_between_axes on a subset
    of pairs, then construct_curves() skipped).
  - [worth-discussing] The below-threshold ids-only SINGLE-SHOT cell (build
    transiently, never raise) is endorsed semantics (verdict below) but
    unrecorded scope: the done-when neither authorizes nor describes it, while
    the shipped CHANGELOG and docstrings already promise it to users — at
    src/hiveplotlib/viz/datashader.py:655-668 and the WS3 done-when.
    Suggested change: one-line amend-plan recording the design completion;
    no code change.
  - [worth-discussing] Pre-existing bug walked on the touched method: the
    both-directions ambiguous-tag ValueError prints the axis_id_2 -> axis_id_1
    direction twice and never shows the axis_id_1 -> axis_id_2 tag list — at
    src/hiveplotlib/hiveplot.py:1235-1236 (same copy-paste at :1747 in
    add_edge_kwargs).
    Suggested change: first line becomes axis_id_1 -> axis_id_2 over
    hive_plot_edges[axis_id_1][axis_id_2].keys(). Verified pre-existing against
    the pre-WS3 ASV build copy, so not a WS3 regression; two-token fix at two
    sites.
Test-method-coverage audit: clean — sampled the nine new datashader tests
  (tests/datashader_test.py:1081, :1126, :1165, :1199, :1220, :1254, :1281,
  :1317, :1358) and the two hiveplot tests (tests/hiveplot_test.py:2376,
  :2417); every test_<entry point>_* body calls its named entry point
  (datashade_edges_mpl, datashade_hive_plot_mpl, _chunk_edge_curves,
  _construct_edge_subset_curves). Sampled perf-lane names
  (tests/performance_regression_test.py:319, :337, :364) call measure_peak_rss
  / time_call / assert_raster_equivalent as named.
```

**Single-shot ids-only cell verdict (the dispatch brief's explicit question): endorse, record via amendment.** "Datashader entry points render from stored ids regardless of construction state; persistence only happens when the user runs `construct_curves()`" is a cleaner contract than "fused works only above the threshold." The alternative would make the API's most basic behavior scale-dependent in the most confusing direction (KeyError below 1M edges, working render above), and would turn any future retuning of `_DEFAULT_STREAM_CHUNK_THRESHOLD` into a silent behavior break for small ids-only callers. The cell keeps the two axes orthogonal (the threshold picks the memory strategy; construction state gates nothing), is pinned bit-exact by `test_datashade_edges_mpl_ids_only_single_shot_matches_two_stage`, and is already promised to users in the shipped CHANGELOG entry. Right disposition: record-and-keep (concern 4), not revert.

User walkthrough record (per dispatch brief):

- **(a) Classic user (construct_curves + any backend):** clean, verified no-change. `add_edge_curves_between_axes` delegates to `_construct_edge_subset_curves` with the control-point anchoring passed explicitly (`control_angle_axis_ids`), so persisted output stays bit-identical; the missing-ids KeyError text is verbatim-preserved via `_missing_edge_ids_message`; a later streamed render reuses persisted arrays by identity (test at tests/datashader_test.py:1220). Two-stage docstrings unchanged in substance.
- **(b) Scale user (skip construct_curves, datashade):** just works, fused, nothing persisted. Discoverable at the three places this user actually looks: the `stream_chunk_threshold` param docs, the new `construct_curves` resident-memory note pointing back at the datashader module, and the CHANGELOG. The second-render-costs-the-same story plus its escape hatch is documented, but only on `datashade_edges_mpl` (concern 1: hoist one clause into the wrapper).
- **(c) Custom-geometry user (num_steps=50 at scale):** no silent num_steps confusion is reachable because the entry points take no geometry parameters; the only route to custom geometry is the two-stage one, named in the same sentence that discloses the default-parameter fused build. Clean, with the mixed-state caveat (concern 3) as the one silent edge.
- **(d) Mixed-state user:** per-chunk independence is exact and tested (persisted chunk reused by identity, others left unpersisted; tests at :1254, :1281). The one surprise is the silent default-geometry fill where pre-WS3 behavior was a loud KeyError (concern 3).
- **(e) Vector-backend user in ids-only state:** behavior unchanged from pre-WS3, but it is a silent empty-edges figure, not a legible error (concern 2). The docs disclose the constraint in three places; the failure site itself says nothing.

Also walked and cleared: `datashade_nodes_mpl` correctly omits the fused story (nodes have no persistence stage); chunk collection gates on stored ids so empty-id records contribute no chunks (test at :1387); the CHANGELOG entry is reader-facing and names both the capability and the "all other backends still require construct_curves()" boundary; and the naming audit's "no new user-facing name" held, so the done-when's amend-plan naming trigger never fired.

**Addendum (2026-07-04, WS3 touch-up review): `unconstructed_curves_check` warning, disposition item 2 execution.**

```
Status: propose (execution faithful to the adopted contract; 1 module-wide item)
API surface reviewed: [viz.input_checks.unconstructed_curves_check, edge_viz
  warning behavior in viz/{matplotlib,bokeh,plotly,holoviews}]
Concerns:
  - [worth-discussing] Through the headline wrappers (hive_plot_viz ->
    edge_viz), stacklevel=3 attributes the warning to the wrapper's internal
    edge_viz call (e.g. src/hiveplotlib/viz/matplotlib.py:625,
    src/hiveplotlib/viz/bokeh.py:756), not the user's render line; the
    datashader-prototype-then-vector-final-figure user this warning targets
    most often renders via hive_plot_viz — at
    src/hiveplotlib/viz/input_checks.py:161.
    Suggested change: module-wide follow-up, not a touch-up defect: all nine
    pre-existing input_check warnings share the same one-frame-short
    attribution through the wrappers, and the disposition pinned "stacklevel=3
    matching the existing input_checks.py sites" (which the execution
    matched); if picked up, thread an optional stacklevel parameter through
    both check helpers and bump it from the wrappers.
Test-method-coverage audit: clean —
  test_edge_viz_warns_once_when_stored_edge_ids_lack_curves (all four backend
  test files; matplotlib copy parametrized over fully-ids-only and mixed),
  test_edge_viz_no_warn_when_constructed_curves_cover_stored_edge_ids
  (tests/viz_matplotlib_test.py:187), and
  test_datashade_edges_mpl_ids_only_render_raises_no_warning
  (tests/datashader_test.py:1221) each call their named entry point in the
  body.
```

Walkthrough record for the five checks in the review brief:

- **(a) Message: clean.** Problem, then fix-this-backend (`construct_curves()`), then the datashader alternative is the order the confused vector-backend caller needs, and matches the disposition's contract. Bare `construct_curves()` (where the module's older messages class-qualify, e.g. `HivePlot.build_hive_plot()`) is right here, not a miss: the method lives on `BaseHivePlot` and is inherited by `HivePlot`, both reachable in this state via `add_edge_ids`, so one unqualified name serves both without an isinstance branch. No jargon concerns ("edge subset" and "stored edge ids" match `connect_axes` / `construct_curves` docstring vocabulary).
- **(b) stacklevel: split verdict.** Correct for direct `edge_viz(hp)` calls including the `hiveplotlib.viz.edge_viz` path (a plain re-export in `viz/__init__.py`, no extra frame): three frames up from the warn site is the user's line. One frame short through `hive_plot_viz` (the worth-discussing item above). `p2cp_viz` is exempt in practice: `P2CP.build_edges` routes through `connect_axes`, which always constructs curves, so no public P2CP path reaches the ids-only state (which also means the HivePlot-flavored remedy text never confronts a P2CP user).
- **(c) Once-per-render: reasonable.** Early return caps at one warning per `edge_viz` call regardless of how many subsets are unconstructed; exactly-one asserted in the matplotlib test including the mixed state. Repeated renders in one session dedupe under Python's default warning filter, so no nag spiral.
- **(d) Datashader exemption: confirmed structural, no firing path.** The helper is imported only by the four vector backends; `viz/datashader.py` imports `input_check` alone, and `datashade_hive_plot_mpl` composes `datashade_edges_mpl` + matplotlib `axes_viz` (axes branch, never edges) + `datashade_nodes_mpl`, none of which reach the helper. Pinned executably by the no-warn test on both routes (threshold 0 and default) under `simplefilter("error")`.
- **(e) Voice: consistent.** Impersonal lead matches the module's nodes-branch messages ("At least one of your axes has..."), plain `UserWarning`, backtick-quoted remedies, `stacklevel=3`. The helper's docstring accurately documents the skip semantics, the once-per-call cap, and the datashader exemption.

#### Workstream 4 review (2026-07-04): narwhals input boundary + boundary-gap follow-up

```
Status: propose
API surface reviewed: [NodeCollection.__init__ / .data / .frame_library /
  .to_pandas() / check_unique_ids / create_partition_variable / copy,
  Edges.__init__ / .data / .frame_library / .to_pandas() / add_edges /
  export_edge_array / _validate_edge_data error surface,
  BaseHivePlot.add_nodes / add_edge_ids / to_json,
  hiveplotlib.frames module (as_eager_frame TypeError text, helper gathers),
  styled-render kwarg gathers across matplotlib/bokeh/plotly/holoviews,
  pyproject.toml [polars] extra, CHANGELOG 0.29 Input Data entry,
  CLAUDE.md design note]
Concerns:
  - [must-fix] create_partition_variable's docstring documents the labels=None
    string-bin mechanism on non-pandas engines but not its plot-content
    consequence: string bins sort lexicographically at the partition seam, so
    at multi-digit bounds (e.g. cutoffs [5, 20]) a polars-built plot orders
    axes "(-inf, 5.0]", "(20.0, inf]", "(5.0, 20.0]" while the pandas twin
    orders them numerically — a silent cross-engine axis-order divergence the
    user can ship without noticing, and the one divergence the bit-exact
    equivalence gate deliberately doesn't cover (its twins all pass explicit
    labels) — at src/hiveplotlib/node.py:299-306 (the note), sort site
    src/hiveplotlib/hiveplot.py:440.
    Suggested change: one sentence appended to the docstring note ("because
    these string labels sort lexicographically, downstream axis order can
    differ from the pandas path at multi-digit bounds; pass explicit `labels`
    to control axis order"), and carry the same line into the pending polars
    gallery notebook's brief. Escalated over worth-discussing because the
    2026-07-04 disposition item 3(c) records this as "documented in the
    docstring" while only the mechanism half landed; the fix is one sentence
    with no rendering risk, on a surface debuting this release.
  - [worth-discussing] The shared as_eager_frame TypeError says "lists,
    arrays, delayed objects, and lazy frames are not supported here", but on
    the Edges entry point (n, 2) numpy arrays ARE supported (handled upstream
    of the detection point) — a user passing a list of [from, to] pairs is
    steered away from the np.array(...) fix the class docstring advertises —
    at src/hiveplotlib/frames.py:44-51.
    Suggested change: drop "arrays" from the shared message, or branch the
    accepted-inputs clause on context (append "or a (n, 2) numpy array" when
    the context is Edges data).
  - [worth-discussing] HivePlotMatrix consumes polars-built collections
    through direct reads on user frames (nodes.data[pv].unique() at
    src/hiveplotlib/hiveplot_matrix.py:1325, :1761, :1800, :2187; .to_numpy()
    gathers at :172-175) that happen to duck-type on polars, but the class
    appears in neither WS4's Files, tests, nor Holdouts: no test pins
    matrix-on-polars and no plan record says it is in or out of the contract.
    Users reading "NodeCollection and Edges accept narwhals frames" will
    reasonably feed a polars-built collection to from_partition.
    Suggested change: one polars-marked HivePlotMatrix.from_partition
    smoke/equality test (likely green as-is), or a named Holdout with a
    revival trigger; mechanical-propagation review rule — the sibling class
    surface needs its own record.
  - [worth-discussing] The [polars] extra reads as a support gate but is a
    test-dependency vehicle: no src code imports polars and polars support
    needs only core narwhals, unlike every sibling extra (bokeh, datashader,
    networkx) which gates an importable hiveplotlib module — at
    pyproject.toml:96-98.
    Suggested change: one comment line on the extra (matching the file's
    comment conventions) saying polars input support itself requires only
    core narwhals and the extra exists as the test/dev install vehicle. The
    CHANGELOG correctly does not advertise the extra; keep it that way.
  - [worth-discussing] CLAUDE.md's coverage trip-wire still enumerates the
    optional-backend markers as networkx/bokeh/datashader/holoviews/plotly;
    WS4 registered @pytest.mark.polars (53 marked tests) but the list wasn't
    extended, so future agents reading the trip-wire miss the marker — at
    CLAUDE.md:19 (CLAUDE.md is in WS4's Files list).
    Suggested change: add `.polars` to the enumerated marker list.
  - [low-confidence] Edges.frame_library's mixed-tag ValueError names only the
    library set, not which tag holds which library, so a many-tag user
    re-derives the mapping by hand — at src/hiveplotlib/edges.py:246-252.
    Suggested change: include the per-tag mapping in the message (e.g.
    "{'pandas': ['tag_a'], 'polars': ['tag_b']}"). The raise itself is the
    right call (verdict below).
  - [low-confidence] P2CP remains pandas-only behind a legible assert
    ("`data` must be pandas DataFrame") but is absent from the plan's Holdouts
    list, leaving the third data-ingesting class unrecorded; user-facing
    claims elsewhere are properly scoped to NodeCollection/Edges — at
    src/hiveplotlib/p2cp.py:83.
    Suggested change: add P2CP to Holdouts with a revival trigger (route
    through frames.py helpers if polars-P2CP demand appears); plan-record fix
    only, no code change this release.
  - [low-confidence] hiveplotlib.frames shipped public-shaped (no underscore,
    public function names, user-facing docstrings) while the naming audit
    scoped only underscore-named helpers (_to_narwhals) out of audit; the
    module now sits between internal and public (cross-referenced only from a
    private method's docstring, no autodoc page) — at
    src/hiveplotlib/frames.py:1.
    Suggested change: either underscore the module or accept it as public and
    give it an autodoc entry in a docs pass; flagged low-confidence per the
    post-impl rename rule.
Test-method-coverage audit: gaps: [test_add_edge_ids_polars_matches_pandas
  (tests/hiveplot_test.py:7866) never calls add_edge_ids in its body; the
  method runs only inside HivePlot.__init__'s internal connect_axes path via
  _build_twin_hive_plot, and the test asserts its stored outputs. A one-line
  direct hive_plot.add_edge_ids(...) call (or a rename to
  test_polars_relevant_edges_...) closes it.] Everything else sampled clean:
  node_test.py:242/:255/:333/:366 (frame_library, to_pandas,
  create_partition_variable), edges_test.py:336/:355/:401/:413/:429
  (add_edges, frame_library, to_pandas), hiveplot_test.py:7962/:7996
  (add_nodes) each call their named entry point in the body; the
  tests/polars_test.py battery names surfaces (styled renders, rasters,
  to_json) it exercises directly.
```

**Mixed-tag `frame_library` verdict (the dispatch brief's explicit question): the ValueError is the right call, endorse.** A sometimes-`str`-sometimes-`dict` return would make the property's type unstable, tax the common single-library case with a type check, and break the mirror with `NodeCollection.frame_library` (always `str`) that the symmetric-fold-in amendment explicitly asked for. Mixed-library tags are constructible (nothing forbids `Edges(data={"a": pandas_df, "b": polars_df})`, and each tag round-trips and renders independently), so the property must answer *something*; "no single name applies, here are the libraries" is the honest answer, the message is legible, and `:raises ValueError:` is documented. The per-tag-mapping message polish above is the only residue.

**Equivalence-gap resolution verdict (frontloaded; the workstream's motivating gap): yes-with-caveats.** The 2026-07-03 amendment demanded that polars input draw the same picture as pandas, not merely run; the shipped gate over-delivers (bit-exact, no tolerance: placements, `relevant_edges`, ids, curves, edge/node rasters across single-shot/streamed/fused) and the boundary-gap follow-up extended bit-exactness to the styled-render surface the base workstream half-shipped (per-collection matplotlib values, bokeh CDS contents, plotly serialized figures incl. hover customdata, holoviews vdims, `to_json` string equality). The caveat is the one divergence that remains by design: `labels=None` partition bins stringify on non-pandas engines and can reorder axes versus the pandas twin, and the twins all sidestep it with explicit labels while the docstring names only the mechanism (the must-fix above). Close the sentence-sized doc gap and the closure is complete.

User walkthrough record (per dispatch brief):

- **(a) Polars user end-to-end: clean.** Construct (input cloned, mutation-isolated), partition (frame stays polars post-call, pinned by test), style-by-column across all five backends plus `to_json` (bit-exact twins), `.data` round-trips polars, repr/len/copy all dispatch cleanly. The eighth-site hovertemplate fix means plotly hover works rather than IndexErroring. The one divergence a polars user can hit is the `labels=None` axis-order carve-out (must-fix above).
- **(b) Pandas user: verified literally unchanged, and not narwhals-spammed.** Every touched site has an `isinstance(pd.DataFrame)` fast-path running the verbatim old idiom (including duplicate-column and non-string-column-name frames narwhals itself rejects). Docstrings add one to two acceptance sentences per class with pandas listed first; "eager"/"narwhals" jargon is confined to those sentences; the `data` property docstrings lead with plain round-trip language. Proportionate.
- **(c) Error paths: legible, one wording lie.** Bare list / lazy frame / dask.delayed all land in `as_eager_frame`'s TypeError naming the received type and accepted inputs (single detection point, as the planning re-review's concern 4 asked). Mixed-library `add_nodes`/`add_edges` raise legible TypeErrors naming both libraries with `:raises:` documented. The one flaw is the shared message denying arrays on the Edges surface (worth-discussing above).
- **(d) [polars] extra: test-dep vehicle wearing a support-gate costume.** Judged above (worth-discussing). Harmless in effect — installing polars is what a polars-curious user needs anyway — but the extra's existence implies gating that doesn't exist; one pyproject comment keeps future maintainers from "simplifying" support behind it.
- **(e) create_partition_variable labels=None on polars: documentation not loud enough where the user hits it.** The mechanism (string bins) is documented; the consequence (axis-order divergence) is not, and the user hits the consequence in the rendered plot, not in the docstring they didn't reread. The must-fix above is the sentence-sized fix; the pending polars gallery notebook is the second, louder home.
- **(f) frame_library / to_pandas() vs planning intent: matches, execution exceeded the sketch.** Symmetric on both classes per the amendment; `Edges.to_pandas()` mirrors the `data` property's single-vs-dict shape; the mixed-tag ValueError is a semantics the planning take never specified, resolved correctly (verdict above). The implementation split "public `to_pandas()` uses the engine's own fidelity-preserving converter (pyarrow caveat documented in a note)" from "internal geometry seam stays dependency-free via column-wise numpy" — better than the planning sketch's single `nw.from_native(...).to_pandas()` one-liner, and the caveat is documented on both methods.

Also walked and cleared: `add_nodes` preserves new-nodes-first concat order on both paths (pinned against the pandas twin); ndarray-to-pandas exception wording aligns verbatim-in-substance across both `data` properties per the done-when, with `NodeCollection`'s cross-reference to the `Edges` exception preempting the "does it apply here?" question; `unique_id_column=None` row-number fallback for index-less libraries is documented in the param doc and tested including the name-collision path; `NodeCollection.data`'s new property+setter preserves the old plain-attribute semantics (setter kept for `graph_features` writes, no new validation, no regression); `frames.py`'s narwhals-rejected-pandas-degenerates rationale (duplicate columns, non-string column names) is recorded in the module docstring where a future refactorer will look.

#### Workstream 4 notebook reviews: editorial-critic + viz-critic (2026-07-04)

*Transcribed by the dispatching session: both critics are read-only agents and report findings rather than writing plan sections; this record closes the qa completeness gate. Their full reports live in the session transcript; every finding below was implemented in the notebook revision pass (same day) unless noted.*

- **Editorial-critic (`examples/creating_hive_plots_from_polars.ipynb`): propose.** One must-fix: the `labels=None` cross-engine axis-order divergence was missing from the page the disposition designated as its second home — one-sentence fix specified and landed in the "Create Partition Variable" cell. Two worth-discussing (pandas wrongly lumped into the no-own-page clause; a colon promising a `to_pandas()` demo the code cell didn't deliver) and two low-confidence wording items (exactly-two-columns phrasing; extras lead-in overclaiming) — all landed in the revision. Clean on genre fit, dataset coherence (the `pl.from_pandas` stand-in judged legitimate as written), section-worth, voice, cross-links, index placement, and the llms.txt entry (right call, right wording, length earned).
- **Viz-critic (both figures, instructional polish tier): propose.** One worth-discussing: missing thumbnail wiring (asset + `nbsphinx_thumbnails` entry) — landed in the revision (magma styled figure exported as the asset). Clean on everything else, with two notable verifications: the default render is byte-identical to the pandas sibling's (sha256 match — a live cross-engine visual-regression proof), and the prose's self-validating-color claim was confirmed in the rendered pixels (each axis brightens monotonically outward, tiling the magma ramp).
- **api-critic touch-up addendum: knowingly not run.** The 10-item critic-fixes touch-up implemented the critics' own specifications (disposition items 1-10); a re-read would review the reviewers' own prescriptions, so the dispatching session recorded this decision instead of dispatching a circular pass. qa verified the fixes match the disposition items.
- **Post-revision critic re-look: knowingly not run**, same reasoning — the revision implemented the editorial/viz critics' own findings verbatim; qa verified execution (notebook re-executed clean, docs build zero-warnings, thumbnail renders).

#### Workstream 5 review (2026-07-05): Dask + cuDF/GPU passthrough (zero-parameter surface)

```
Status: propose
API surface reviewed: [NodeCollection lazy paths (__init__ / __repr__ / __len__ /
  check_unique_ids / copy / frame_library / to_pandas / create_partition_variable
  rejection), Edges lazy paths (__init__ / _validate_edge_data / __repr__ /
  __len__ / add_edges / export_edge_array / to_pandas / frame_library),
  hiveplotlib.frames (as_frame, as_eager_frame lazy rejection, is_lazy_frame,
  frame_row_count, LazyEdgeSubset), BaseHivePlot lazy guards (add_edge_ids /
  _add_edge_ids_lazy / construct_curves / _construct_edge_subset_curves /
  to_json / _require_in_memory_edge_subsets), viz/datashader dask-native route
  (_rasterization_route, _dask_native_edge_raster, entry-point docstrings) +
  cuDF collection-frame arm (_collection_frame_backend / _to_host_array),
  exceptions.DaskComputationError, Makefile test-gpu, CONTRIBUTING.md "GPU
  Verification", pytest.ini markers + filterwarnings, CHANGELOG entry]
Concerns:
  - [must-fix] CONTRIBUTING's GPU-venv setup is not runnable as written: the
    snippet installs only cupy/cudf/dask-cudf plus `-e "<repo>[datashader]"`,
    but `make test-gpu` invokes pytest with pytest.ini's addopts, which need
    pytest, pytest-cov (--no-cov), pytest-html (--html/--self-contained-html),
    pytest-timeout (--timeout), and pytest-xdist (-n 0) — exactly the
    `[testing]` extra; a contributor following the three lines verbatim gets
    "No module named pytest" (the Implementation log itself records the real
    venv gained the "pytest stack" by hand) — at CONTRIBUTING.md:126.
    Suggested change: `-e "<path/to/repository>[datashader,testing]"`.
  - [must-fix] The lazy-Dask full-materialization consequence is undocumented
    on the public conversion surfaces while the Implementation log claims
    "export_edge_array/to_pandas are documented deliberate materializations":
    the notes live only on private helpers (frames.frame_to_pandas_copy,
    Edges._tag_edge_array); the public NodeCollection.to_pandas /
    Edges.to_pandas / Edges.export_edge_array docstrings say nothing, so the
    out-of-core user's one accidental-OOM footgun on this surface is the one
    without a docstring sentence (record-claims-documented + consequence-
    missing) — at src/hiveplotlib/node.py:302-304, src/hiveplotlib/edges.py:281-283
    and :391-396.
    Suggested change: one sentence per docstring ("A lazy (Dask) frame is
    computed into memory here; a deliberate materialization for explicit
    conversion / export requests").
  - [worth-discussing] The construct_curves lazy TypeError's remedy is
    under-scoped: "materialize the edge data first (e.g. `.compute()` ...) to
    construct persisted curves" omits the "and rebuild the hive plot" step its
    to_json twin message correctly carries; computing the frame alone changes
    nothing about the stored lazy state — at src/hiveplotlib/hiveplot.py:1621-1630.
    Suggested change: align the remedy clause with
    _require_in_memory_edge_subsets' wording ("materialize ... and rebuild the
    hive plot, or rasterize with ...").
  - [worth-discussing] The polars-LazyFrame rejection names the type and the
    accepted inputs but offers no collect-first remedy ("lazy frames from
    other libraries are not supported here" ends the message), while the
    sibling Dask-on-eager-op message models the remedy shape (`.compute()`) —
    at src/hiveplotlib/frames.py:44-57.
    Suggested change: append "collect them to eager frames first (e.g.
    `.collect()` on a `polars.LazyFrame`)".
  - [worth-discussing] The vector-backend ids-only warning leads with a
    remedy that is a guaranteed dead end on a lazy hive plot: "Run
    `construct_curves()`" raises the lazy TypeError, so the Dask user recovers
    in two hops (warning -> TypeError -> datashader) instead of one — at
    src/hiveplotlib/viz/input_checks.py:160-167.
    Suggested change: when the unconstructed subsets are lazy, lead with the
    datashader remedy (and name `.compute()` + rebuild) instead of
    construct_curves(); consumer-code change, route via amend-plan if picked up.
  - [worth-discussing] _validate_edge_data's docstring — the exact method the
    WS5 done-when names as the single detection point — still describes the
    pre-WS5 contract: "a lazy frame" sits in the rejected-examples list,
    as_eager_frame is named as the detection point (the code now calls
    as_frame), and the :raises: line implies lazy Dask input raises; the Edges
    :param data: line also omits the lazy-Dask acceptance the class intro
    carries — at src/hiveplotlib/edges.py:170-179 and :62.
    Suggested change: reword to name as_frame and lazy-Dask acceptance
    ("lazy frames from other libraries" in the reject list).
  - [worth-discussing] Plan-record drift the notebook author could inherit:
    API usage Example 4's second inline footgun comment still teaches the
    superseded shuffle story ("place_nodes_on_axis sorts, which on Dask is a
    shuffle ... consider sorting upstream"), which WS5 resolved as no-shuffle
    (notebook fact 2: no Dask sort exists inside hiveplotlib; nodes are the
    in-memory side) — at the plan's API usage examples, Example 4.
    Suggested change: one-line amend-plan cascade replacing the shuffle
    comment with the real constraint (node data must fit in RAM).
  - [low-confidence] DaskComputationError's message quotes a doc title
    ("Hive Plots from Dask") for a page that does not exist yet; the shipped
    sibling's H1 convention is "Creating Hive Plots from Polars", so the
    quoted string will be a substring of the likely title rather than an exact
    match — at src/hiveplotlib/viz/datashader.py:792.
    Suggested change: pin the quoted title in the notebook-author brief (or
    verify the substring match at notebook qa).
  - [low-confidence] Public docstrings now cross-reference
    hiveplotlib.frames.LazyEdgeSubset (the add_edge_ids lazy note) while
    hiveplotlib.frames has no autodoc page (nitpicky is off, so no build
    warning; the reference renders unlinked and the class is undiscoverable
    from the docs site) — stakes raised on WS4's open frames-module
    public-shape item — at src/hiveplotlib/hiveplot.py:1200-1202.
    Suggested change: resolve the WS4 item (autodoc page or underscore the
    module) in the docs pass.
Test-method-coverage audit: clean — sampled tests/dask_test.py (as_frame :175,
  as_eager_frame :188, is_lazy_frame :197, frame_row_count :207,
  create_partition_variable :330, add_edges :317, add_edge_ids :491/:561,
  datashade_edges_mpl :572/:607/:723, construct_curves :620, to_json :629,
  datashade_nodes_mpl :651) and tests/cudf_test.py (:120, :136, :180, :214);
  every test_<entry point>_* body calls its named entry point, and the two
  descriptively named tests (test_vector_backend_warns_on_dask_ids_only_state,
  test_polars_lazy_frames_still_rejected) call edge_viz / Edges directly.
```

**Resolution verdict (frontloaded; this workstream's disclosure shape was set by the planning re-review's four post-Path-A concerns): yes-with-caveats.** Concern 1 (ImportError trigger condition) closed as respecified by the 2026-07-03 no-extra amendment: the module-level datashader-extra guard is the only ImportError, and no frame-detection-time ImportError was promised or shipped. Concern 2 (failure-site message) closed: `DaskComputationError` carries the per-partition byte formula, the `.repartition()` remedy, the docs pointer, and the chained original, pinned by the poisoned-partition test. Concern 4 (single detection point, legible malformed-Dask errors) closed: `as_frame` is the single point, and `dask.delayed` / `dask.array` / bare-list messages are pinned by parametrized tests; the residue is the stale `_validate_edge_data` docstring (worth-discussing above). Concern 3 (Example 4 inline footguns) is the one half-closure: the comments exist, but the shuffle comment now teaches a footgun the implementation dissolved (worth-discussing above). Close the two record items and the closure is complete.

User walkthrough record (per dispatch brief):

- **(a) Dask user, `dd.read_parquet` -> construction -> datashade: zero ceremony, and it genuinely just works.** `NodeCollection` requires an explicit `unique_id_column` with a legible TypeError explaining why (row numbers cannot be assigned lazily); the uniqueness check is one lazy count pass with the pre-existing `check_uniqueness=False` escape hatch; `Edges` validates by column metadata with no compute; reprs are compute-free and say so; `len()` documents its count computation. `HivePlot` construction stores `LazyEdgeSubset` at both sites and skips curves; `datashade_hive_plot_mpl(hp)` renders with no parameter, every reduction (var/std included) delegated to datashader's Dask pipeline, partition-count invariant, and both entry-point docstrings disclose that `stream_chunk_threshold` is never consulted for lazy input. Every eager-only operation degrades legibly: the shared `as_eager_frame` message names what lazy input DOES support before giving the `.compute()` remedy (the right order for a user who thinks Dask support is broken), `to_json` correctly says "and rebuild the hive plot", and `construct_curves` is the one message with the under-scoped remedy (concern above). `frame_library` answers `"dask"` without computing; `to_pandas()` materializes silently (must-fix above). The `DaskComputationError` guidance holds up for the blown-partition user: the byte formula sizes partitions concretely, `.repartition()` is the named action, and the chained cause preserves the real error; the broad `except Exception` means a non-memory partition failure also wears repartition guidance, acceptable because the message hedges ("can exhaust memory") and always chains the actual error.
- **(b) cuDF user: flows exactly like polars until the reduction set bites.** Round trip stays cuDF (`frame_library == "cudf"`), count/any/max/min reproduce the pandas twin's raster through the CUDA path on both routes. sum/mean/var/std raise upstream (NVVM kernel compilation) with no hiveplotlib breadcrumb in the traceback; the smoke battery pins raise-not-mis-render, so the failure is loud, not wrong. The plan's chosen disclosure home is the pending cuDF notebook + docs page (2026-07-05 amendment rider, notebook-facts item 4); that rider is now the sole load-bearing disclosure for this footgun, so the notebook dispatch must treat it as non-optional. No library-side doc change requested: the limitation is stack-version-dependent and a docstring pin would go stale at the first upstream fix.
- **(c) polars-LazyFrame user: rejected legibly, one clause short.** The single detection point raises a TypeError naming the received type and the accepted inputs, and the test pins the message; the missing piece is the one-token `.collect()` remedy (worth-discussing above).
- **(d) Contributor: `make test-gpu` is solid; CONTRIBUTING's setup is one extra short of runnable.** The target's missing-venv error is legible and names both the CONTRIBUTING section and the `GPU_VENV` override; serial `-n 0`, `--no-cov`, and separate report paths are all justified in comments. CONTRIBUTING names the pandas<3 constraint honestly, the install order is correct (cudf pins pandas 2.x first; the editable install leaves it satisfied), and the "fully skipped run on a GPU machine means the environment is broken" warning is exactly the honesty the never-in-CI lane needs. The pytest-stack omission is the must-fix above.

Also walked and cleared: no `use_dask` anywhere in src or tests (grep-verified; Path A held completely); mixed lazy + eager tags are rejected legibly at `add_edge_ids` with actionable guidance (pass `tag=`, avoid mixing) rather than silently materialized; an unplaced-axis lazy subset stays an empty lazy subset and rasterizes to nothing without triggering a row count; `HivePlot.copy()` / `Edges.copy()` carry lazy membership through deepcopy by sharing immutable task graphs (`LazyEdgeSubset.__deepcopy__`), with raster equality pinned; the dask-native route's missing-metadata-column ValueError fires before any computation and matches the in-memory routes' message verbatim; the NaN-sentinel partition guard is invisible at the API surface (behavior-free, documented at the builder with the upstream-quirk rationale); the pytest.ini `NumbaPerformanceWarning` ignore is message-scoped to the CUDA grid message and cannot mask CPU-side numba warnings; the CHANGELOG entry is reader-facing and names the zero-parameter surface, the lazy end-to-end story, and the install story in four lines; `DaskComputationError` follows the exceptions-module conventions and its docstring carries the mechanism and the always-chained-cause contract.

#### Workstream 5 notebook reviews: editorial-critic + viz-critic (2026-07-05)

*Transcribed by the dispatching session (read-only critics; same convention as the WS4 transcription block above). All findings were implemented in the same-day notebook micro-pass.*

- **Editorial-critic (both pages): propose.** Two worth-discussing: a Dask-page colon-promise mismatch (fixed by rewording, mirroring the polars-page precedent) and the `DaskComputationError` doc-title alignment (landed as touch-up item 11: message now quotes "Creating Hive Plots from Dask" exactly). One low-confidence adopted: a captured `ds.max("weight")` GPU render added so the supported reduction set is shown working, not just the failure demo. Clean on: VRAM embargo (zero hits), partition-sizing framed as guidance with the threaded-scheduler/local-only disclosure (Failure mode 3), the visible no-shuffle section, both cuDF normative disclosures framed as current-stack facts, the honest all-markdown-with-captured-outputs design (endorsed as a fair extension of the graph_metric_backends precedent), index order, both llms.txt entries, dataset coherence, voice, length.
- **Viz-critic (two in-page figures + two thumbnails): propose.** One worth-discussing: both new thumbnails carried axis labels against the text-free rule (fixed: re-exported with `show_axes_labels=False`). One low-confidence adopted: cuDF attachment re-exported at dpi 150 for sibling parity. Clean on: 3-axis rule, partition legibility, datashader-default palette discipline, the Purples thumbnail orthogonalization (sanctioned non-data-semantic), five-thumbnail gallery distinguishability including grayscale survival, attachment honesty (real GPU render matching shown code, alt text present).
- **Post-micro-pass critic re-look: knowingly not run** — the micro-pass implemented the critics' own findings verbatim; qa verifies execution.

#### Whole-surface 0.29 release-freeze review (2026-07-06): three-persona walk (pandas-only, polars, Dask 10M+)

```
Status: propose
API surface reviewed: [the full 0.29 CHANGELOG surface as one system:
  NodeCollection / Edges narwhals boundary (__init__, data, frame_library,
  to_pandas, create_partition_variable, add_edges, add_nodes, copy, len, repr,
  export_edge_array), hiveplotlib.frames error messages, LazyEdgeSubset,
  stream_chunk_threshold on all three datashader entry points + aliases +
  HivePlot.plot() and HivePlotMatrix forwarding, the dask-native route,
  DaskComputationError, lazy guards (construct_curves, to_json, add_edge_ids),
  unconstructed_curves_check, relevant_edges value change, CHANGELOG 0.29,
  the three new gallery notebooks, llms.txt]
Concerns: 0 must-fix, 4 worth-discussing, 4 low-confidence (below)
Test-method-coverage audit: n/a this pass (no new methods; per-WS audits stand,
  one WS4 gap was closed by disposition item 7(c))
```

**Freeze verdict up front: no must-fix, and no rename I would stand behind.** The names that freeze at 0.29 (`stream_chunk_threshold`, `frame_library`, `to_pandas`, `DaskComputationError`, `LazyEdgeSubset`, the `hiveplotlib.frames` module) all survive the whole-surface walk; `to_pandas` in particular matches polars / pyarrow / cuDF spelling exactly, and `frame_library` matches the docstrings' own "same library you passed in" vocabulary. The one naming question worth a conscious decision before freeze is low-confidence item 5. Every per-workstream must-fix I traced (CONTRIBUTING `[datashader,testing]`, the three public materialization sentences, the construct_curves "and rebuild" remedy, the `.collect()` clause, the lazy-aware ids-only warning, the exact notebook title in `DaskComputationError`) verifiably landed in the tree.

**Worth-discussing:**

1. **Public `:raises:` blocks omit the Dask route's exceptions.** `datashade_edges_mpl` and `datashade_hive_plot_mpl` list only `ValueError` (src/hiveplotlib/viz/datashader.py:952-954 and :1502-1504), while the Dask route raises `DaskComputationError` and a mixed lazy/in-memory `TypeError`; the full raise inventory lives only on the private `_dask_native_edge_raster` (:744-749). The 10M-edge Dask user asking "what do I catch" reads the public docstring. Suggested change: add `:raises DaskComputationError:` and `:raises TypeError:` lines to both public entry points. Cross-WS seam (WS2 wrote the docstrings, WS5 added the raises), so no per-WS pass owned it.
2. **`NodeCollection`'s class intro contradicts its own param doc on lazy acceptance.** The intro says "``data`` accepts any eager narwhals-supported dataframe" (src/hiveplotlib/node.py:96-98) while `:param data:` (node.py:120-123) accepts lazy Dask; sibling `Edges` carries lazy acceptance in both places (edges.py:33-37). A Dask user reading the tooltip intro concludes node frames must be computed first, the exact ceremony Path A removed. Suggested change: one "or a lazy ``dask.dataframe.DataFrame``" clause in the intro sentence.
3. **`HivePlotMatrix`'s engine-coverage record stops at polars.** `from_partition` reads `nodes.data[partition_variable].unique()` then `sorted(...)` directly (src/hiveplotlib/hiveplot_matrix.py:1325-1326). The WS4 disposition (2026-07-04, item 6) gave the sibling class a polars record per the mechanical-propagation rule; Dask/cuDF collections remain untested and unrecorded (cuDF Series refuses host iteration; Dask `.unique()` is lazy), and neither Holdouts nor the CHANGELOG says whether the matrix is in or out of the Dask/cuDF contract. No promise is broken (the CHANGELOG scopes the claim to NodeCollection/Edges), so this is a record gap, not a bug. Suggested change: one smoke test or a Holdouts bullet with a revival trigger, same shape as the polars disposition.
4. **Three factual doc slips on touched tooltip surfaces, all one-word fixes.** `Edges.__len__`'s summary says "number of nodes in the ``Edges``" for an edge count (src/hiveplotlib/edges.py:120, directly above the new lazy sentence); the `Edges` intro says "The ``Edge`` class ingests" (edges.py:31, adjacent to the WS4-rewritten acceptance sentence); `NodeCollection`'s intro has "correponds" (node.py:95). These are the first sentences a 0.29 reader meets on the debuting boundary.

**Low-confidence:**

5. **`DaskComputationError` names more than it covers.** Four other Dask-compute sites raise raw Dask errors (`frame_row_count` behind `len()`, `check_unique_ids`, the `to_pandas` family, the axis-placement compute at src/hiveplotlib/hiveplot.py:819); only the rasterization loop raises this class (viz/datashader.py:841-852). A user wrapping their pipeline in `except DaskComputationError` catches only the render failure. `DaskRasterizationError` would scope it honestly, and 0.29 is the last free rename; keeping the broad name is defensible if future wrapping of the other sites is plausible. Low-confidence per the post-impl rename rule; flagged because the freeze makes it now-or-deprecation-cycle.
6. **The `dask.delayed` / `dask.array` rejection names no object-specific remedy.** `_unsupported_type_message`'s remedy clause covers only lazy frames ("collect other libraries' lazy frames to eager frames first (e.g. `.collect()` on a `polars.LazyFrame`)", src/hiveplotlib/frames.py:44-59); a `dask.delayed` holder gets "not supported here" with no `dd.from_delayed(...)` / compute pointer, which the parked planning-era item's sketch carried. The shipped message meets the "legible" bar (names the received type and the accepted `dask.dataframe.DataFrame`), so parking held; this is residual polish, one clause.
7. **`NodeCollection.copy()` on lazy data silently triggers a count computation** (the constructor re-runs `check_unique_ids`, src/hiveplotlib/node.py:278-284 into :208), while sibling surfaces document their compute behavior (`__repr__` compute-free and says so, `__len__` "triggers a count computation"). Narrow: `HivePlot.copy()` deepcopies and bypasses it. One docstring sentence, or skip the re-check on copy.
8. **`Edges.copy()`'s "never share storage" phrasing doesn't cover the lazy case.** The docstring (src/hiveplotlib/edges.py:357-359) promises deep-copied `relevant_edges` storage, but `LazyEdgeSubset.__deepcopy__` deliberately shares the immutable task graph (frames.py:408-421). Sharing is safe; the absolute phrasing is the only issue. One clause.

**Clean checks (walked and judged fine, so this section evidences coverage):**

- **Persona A (pandas-only, upgrades untouched code):** every boundary site keeps an `isinstance(pd.DataFrame)` fast path running the verbatim old idiom (node.py:172-173, edges.py:212-213, frames.py throughout); junk-input errors improve (0.28's bare `AttributeError` on a list becomes a `TypeError` naming `pandas.DataFrame` first); narwhals vocabulary is confined to the acceptance sentences; the one new warning (`unconstructed_curves_check`) fires only in the state that previously rendered a silently edge-less figure, with a CHANGELOG Changed entry; the `relevant_edges` value change is disclosed with positional-application guidance.
- **Persona B (polars):** construction clones (mutation-isolated on both paths); `create_partition_variable` reattaches the partition column in-library with the lexicographic-order consequence documented at node.py:339-341 and taught in the notebook; `add_edges` / `add_nodes` concat in-library with legible mixed-library TypeErrors; dict-of-polars round-trips per tag; a `polars.LazyFrame` is rejected with the `.collect()` remedy; `frame_library` / `to_pandas` are demoed in the polars notebook, not just documented.
- **Persona C (Dask 10M+):** zero-parameter acceptance is honest end to end; `unique_id_column` required for lazy nodes with a reasoned TypeError (node.py:186-190); every eager-only operation degrades with a two-part message stating what lazy input DOES support before the `.compute()` + rebuild remedy (`as_eager_frame` frames.py:117-124, `construct_curves` hiveplot.py:1621-1631 with the "and rebuild" clause landed, `to_json` via `_require_in_memory_edge_subsets` hiveplot.py:2299-2306); the nodes-are-the-in-memory-side footgun is disclosed at the `NodeCollection` param doc itself; the partition ceiling is self-describing at the failure point (byte formula, `.repartition()`, exact "Creating Hive Plots from Dask" title, chained cause) plus the notebook's 32 GB / threaded-scheduler-framed guideline; all three `stream_chunk_threshold` docstrings disclose the var/std carve-out, and the two edge-rendering ones disclose the lazy-input non-consultation; the ids-only warning is lazy-aware and leads with the datashader remedy; the three mixing-related TypeErrors share "avoid mixing" vocabulary.
- **Docs discoverability:** all three new notebooks registered and carried in llms.txt (lines 102-104) with accurate one-line scoping; the cuDF page names the pandas<3 pin, the count/any/max/min GPU set, and the `to_pandas()` CPU escape hatch; the Dask install story is `pip install hiveplotlib[datashader]`.
- **Settled items engaged, not re-raised:** the WS2 explicit-force-silently-ignored, bool-threshold, and hardcoded-1,000,000 items; the `[polars]` extra framing (pyproject comment landed at pyproject.toml:96); the frames-module autodoc question (batched, rides the docs pass); `Edges.frame_library`'s per-tag message polish (batched); the pyarrow advertised-not-tested residual (recorded rationale stands). Nothing in the whole-surface walk produced new grounds against any of these dispositions.

*Section retrofitted 2026-07-03: this plan predates the adversary's addition to the harness, so the challenge below is a cold planning-mode pass run after the 2026-06-05 grill but before the failure-mode wave (rubric-free), and before Workstream 1 dispatch.*

### Adversary's challenge (planning mode)

Status: challenge (7 items)
Plan reviewed: wiki/wiki/plans/scaling-large-networks.md (cold — retrofit; post-grill, pre-failure-mode-wave, no rubric)
Items:
  - [must-fix] The Path A `use_dask` removal was never applied to the plan's normative sections; the live workstream text still specifies the rescinded parameter — at Workstream 5 done-when, Naming audit (WS5, WS3), Default justifications, API usage examples, Workstream 3 done-when.
    Rubric: no rubric yet — pre-failure-mode-wave
    Push: The 2026-06-05 "Removed parameter (Path A)" amendment lists cascading edits (drop the `use_dask` bullets, rewrite Example 4, delete 4b, update Naming audit and Default justifications) that were never performed: lines ~129, 141-144, 188, 204-209, 276-299, 460, 536, 603-607, 618 still carry the dropped parameter as live spec, with no superseded markers — unlike the 2026-06-11 and 2026-07-02/03 amendments, which WERE cascaded into the done-whens. Under auto-dispatch, an implementer reading the WS5 block (or WS3's "for foreign frames `use_dask`" route-selection clause — which bites early in the chain) executes a spec the maintainer explicitly killed, or halts on the contradiction mid-window. Route to orchestrator amend-plan for a reconciliation pass **before WS1 dispatches**, and while there: the fused-path selection rule for foreign frames is unspecified once `use_dask` is gone (WS3 delegated it to the dropped parameter) — state it (e.g., Dask input always takes the fused streamed route).
  - [must-fix] WS1's "sole consumer" premise is false in today's tree, and the storage change silently shifts indexing semantics at the unlisted consumers — at Workstream 1 (Files, done-when), Patterns this replaces (WS1).
    Rubric: no rubric yet — pre-failure-mode-wave
    Push: `relevant_edges` is read today at `viz/matplotlib.py:460-463` (`.loc[mask, val]`), `viz/bokeh.py:580-581` (`_data[tag][mask]`), `viz/plotly.py:726-729, 775`, `viz/holoviews.py:636-637`, `hiveplot.py:4650-4655` (`to_json`), deep-copied in `Edges.copy()` (`edges.py:244-256`), and shared across matrix cells (`hiveplot_matrix.py:693`) — not only `datashader.py:203-204`. Bool mask → integer indices changes meaning at those sites: `.loc[int_array]` is label-based (right answer only on a default RangeIndex; wrong rows or KeyError on user frames with any other index, and tests built on RangeIndex frames pass green), `df[int_array]` becomes column selection. Also every `hiveplot.py` line ref in the plan is stale (~100-300 lines: `__store_edge_ids` def at 893 / store at 958, `add_edge_ids` at 962, `construct_curves` at 1381). Amend WS1's Files/done-when to sweep and move **all** read sites in lockstep (explicit positional take everywhere), pin the index dtype, and write the WS1 dispatch brief grep-anchored, never plan-line-ref-anchored.
  - [must-fix] The "run WS1→WS7 with minimal supervision" intent is not executable as stated; the chain's back half has gates an unattended run cannot discharge — at Sequencing and scope guards / Workstreams 4-7.
    Rubric: no rubric yet — pre-failure-mode-wave
    Push: WS5 carries a hard local-GPU gate ("no GPU claim without a measured cupy mempool peak — not stubbed, not assumed") that requires installing the cupy/RAPIDS stack on the host and running `make test-gpu` at the machine; WS6 needs a release-version slug decision and is published prose; WS4's and WS5's gallery notebooks and WS7's table are taste-heavy (the harness's own auto-dispatch fit guidance says doc notebooks keep the pauses, and the maintainer's cadence expectation is churn on doc notebooks). Scope the auto-dispatch green-light to WS1→WS3 plus WS4's code surface, with named parking points at the WS4 notebook, WS5, WS6, and WS7 for when the maintainer is back; and note auto-dispatch is only offerable once the failure-mode wave lands the rubric.
  - [worth-discussing] The load-bearing memory measurements can silently skip in the unattended window — at Workstream 2 / Workstream 3 done-when (memory-bound tests "skipped by default if RAM unavailable").
    Rubric: no rubric yet — pre-failure-mode-wave
    Push: WS2/WS3's central claims (transient and resident peak-RSS bounds at 10M edges) live in tests that skip when RAM is short. Under auto-dispatch nobody is watching the skip, so the done-when is satisfiable with the one measurement that justifies the workstream never having run — the same dormant-gate hole the 2026-07-02 arming amendment closed for the streamed-vs-single-shot gates. Require the between-workstream report to state ran-vs-skipped for each memory-bound gate, with a skip treated as halt-back (or at minimum a loud batched flag), not a green pass.
  - [worth-discussing] WS1's memory claim is overstated: "strictly better in-RAM whenever multiple axis-pairs exist (always)" is arithmetically false in the most common case — at Patterns this replaces (WS1), Workstream 1 done-when, Prior ADRs (OGB-arxiv bullet).
    Rubric: no rubric yet — pre-failure-mode-wave
    Push: A 3-axis plot stores ~6 ordered `(pair, tag)` bool masks ≈ 6 bytes/edge; int64 indices ≈ 8 bytes/edge — worse, not strictly better. The win needs int32 (4 bytes) or many pairs, and either way it is MBs against the curves' GBs. The real WS1 value is structural (one storage decision extended by WS5's non-materializing equivalent; a positional-take consumer shape WS2 streams over). Pin the index dtype, restate the claim at O() level plus the structural justification, and fix the Prior-ADRs bullet claiming "Workstream 1 is the unblock" for OGB-arxiv (1.16M edges ≈ 1 GB of curves is feasible today; WS2+WS3 are the unblock) — otherwise the harness-gated "records the in-RAM memory reduction" bullet is judged against an expectation the change cannot meet.
  - [worth-discussing] The planned `hiveplotlib[dask]` extra ignores that `dask[dataframe]` already ships inside the `datashader` extra — at Plan amendments ("maintainer dispositions on the three open A1 disclosure items"), Workstream 5 done-when (ImportError bullet).
    Rubric: no rubric yet — pre-failure-mode-wave
    Push: `pyproject.toml:34` already pulls `dask[dataframe]` with the datashader extra (datashader issue 1319), and the plan itself declares Dask-without-datashader nonsensical — so the target user has Dask installed by the time they can rasterize, the new extra is a strict subset of one they already need, and its extra-naming `ImportError` may again have no path to fire (the exact trap api-critic Concern 1 named, now with the packaging fact on the ground). At WS5 dispatch, re-derive the Concern-1 disposition against the actual pyproject; likely resolution is "the datashader extra IS the Dask story" with no new extra added.
  - [low-confidence] The narwhals `>=1.x,<2` core-dep bound was fixed in 2026-05 and the pin-time checklist never re-derives it — at Default justifications (narwhals core dep), Workstream 4 done-when (pin-time checklist).
    Rubric: no rubric yet — pre-failure-mode-wave
    Push: The WS4 pin-time checklist verifies wheel purity and `make ty`, but not that 1.x is still narwhals's live major at pin time; narwhals releases fast, and pinning a core dep to a dying major creates resolver conflicts for users on fresh stacks. Add "confirm the pinned range tracks the current major (or record why staying behind is right)" to the checklist.

Angles worked, for the record: the **premise** (forward-looking, no external demand; GNN-scale motivation) was grilled 2026-06-05 and the motivating satellite work is still live — no re-litigation, the premise stands. The **approach** (5-code + 2-docs chain, harness-gated, narwhals-as-on-ramp not prerequisite) is sound; the harness having shipped (ADR 0002) collapses no workstream, and the dormant-gate/canary interface the plan cites verifies against the tree (`tests/performance_regression_test.py` skip-marked gates, `tests/performance_regression_canary_test.py`, `runners/performance/profiling_utils.py` tier-1/tier-2 helpers, `benchmarks/benchmarks.py`). **Could-this-not-exist:** the chain is past that gate legitimately; no `existential-must-fix`. The items above are drift and execution-safety, not premise kills.

**Post-grill rubric-check (2026-07-03; planning mode, second invocation — delta against the four Wave 5 modes only, not a second full challenge)**

Status: challenge (1 item)
Plan reviewed: wiki/wiki/plans/scaling-large-networks.md (cold, post-grill rubric-check)

Delta per newly named mode (riders verified present in the normative workstream text, not only claimed in the 2026-07-03 failure-mode-riders amendment — the verify-the-cascade check item 1 taught this plan):

- **Mode 1 (renders-but-unreadable): covered.** WS2 done-when (streamed render validated readable at the shipped/documented raster resolution; rendered image in the between-workstream review packet, auditable in the auto window) + WS6 published-figures rider. WS3/WS5 rubric-only is a recorded decision the post-impl attack enforces; acceptable since both render through the same pipeline WS2's image-check exercises.
- **Mode 2 (edge-only scaling): covered.** WS2's transient-RSS and WS3's resident-memory 10M-edge scenario definitions both require node count scaled in proportion to edge count, no token node counts. Beyond-scenario benchmarks rubric-only by recorded decision.
- **Mode 3 (unrepresentative parallelism): covered.** WS5 done-when (published Dask numbers state scheduler + hardware, framed local-only unless verified representative) + WS6 rider; WS7's pre-existing provenance bullet (machine spec, version, measurement method in adjacent prose) verified as claimed, no edit needed.
- **Mode 4 (plausible picture of the wrong data): covered at WS1, gap at WS4** (item below). WS1 carries the positional-indexing rule at every read site plus non-RangeIndex tests asserting identical output vs. a RangeIndex baseline (an independent baseline, which defeats the shared-baseline trap the mode names); those are permanent default-suite tests, so WS4's narwhals re-expression of the gather (interactions §4) stays guarded on the pandas side.

Items (appended):
  - [must-fix] Mode 4 has no output-equivalence gate on the WS4 polars branch: WS4's done-when requires polars input to "work end-to-end" and round-trip its frame *type*, but never that a polars-built plot draws the same picture as the identical data through pandas — while WS5's Dask branch carries exactly that bullet ("produces an equivalent rasterization to the pandas path") — at Workstream 4 done-when.
    Rubric: Failure mode 4 (plausible picture of the wrong data)
    Push: An engine-specific selection/ordering divergence in the narwhals re-expression (polars has no index; `is_in` / filter / metadata-extraction semantics can diverge silently) passes every stated WS4 gate: the round-trip tests check frame type, the polars test matrix checks the calls run, the harness gate checks pandas-path speed only, and coverage checks lines, not values. The plot renders confidently and lies, inside the green-lit auto window with no maintainer read. Route to orchestrator amend-plan before WS4 dispatch (WS1-WS3 are unaffected): mirror WS5's Dask bullet — the same synthetic nodes/edges through polars and pandas produce identical edge selections and node placements and an equivalent rasterization, including at least one non-trivial-selection case (`add_edge_ids` filtering + a metadata-column reduction).

### Adversary post-impl

**Workstream 1 — membership storage redesign (2026-07-03)**

```
Status: propose
Artifact reviewed: Workstream 1 (bool masks → int32 positional indices); uncommitted working tree on branch 53-scale-hiveplotlib-to-larger-networks
Dispositions held: yes — item 2's consumer-map amendment held in full (all six read sites on positional takes, independently re-grepped: no `.loc[int_array]` or bare-bracket selection over the indices anywhere in src/; verify-not-break sites value-type-agnostic); item 5's restatement held (int32 decision logged with the honest at-or-below-mask expectation, harness gate framed as no-speed-regression + recorded delta); no scope ballooning (diff confined to the WS1 Files list; the retained boolean-mask parameter at `__store_edge_ids` is the done-when's mandated WS5 centralization, documented at the store site, not creep)
Concerns:
  - [worth-discussing] `.astype(np.int32)` carries no overflow guard: past 2**31 - 1 edge rows, `np.flatnonzero`'s int64 positions silently wrap negative, and `.iloc` accepts negative positions — wrong rows selected from the *end* of the frame, no exception, every gate green — at src/hiveplotlib/hiveplot.py:952
    Rubric: Failure mode 4 (plausible picture of the wrong data) — the silent-wrong-selection shape exactly, on the plan's own scaling axis
    Push: the logged premise ("beyond any in-RAM pandas edge frame this path stores") is enforced by nothing; add a one-line loud guard at the store site (raise, or promote to int64, when the mask length exceeds np.iinfo(np.int32).max). Downstream bearing: WS5 extends this exact store site with the Dask counterpart — settle guard-vs-promote before the int32 assumption is baked into the non-materializing structure.
  - [low-confidence] the store-site comment justifies O(num_edges) total storage with "edges partition across (from, to, tag) triples"; repeat axes and bidirectional connect_axes place one edge in multiple triples, so the bound holds via bounded multiplicity, not partition — at src/hiveplotlib/hiveplot.py:960-965
    Rubric: no entry — comment accuracy; flagging because this comment is the WS5 design record the done-when mandates, so it should be exact
  - [low-confidence] the shuffled fixture's power to catch label-based selection is verified only as a one-time claim in the Implementation log; nothing executable pins the fixture's sensitivity, so a future edit (non-unique weights, identity permutation) would silently defang all six non-RangeIndex tests at once — at tests/conftest.py:71-74
    Rubric: Failure mode 4 (the guard tests must stay sharp to keep the mode closed)
    Push: cheap in-fixture asserts (weights unique; shuffled index differs from RangeIndex)
  - [low-confidence] the WS1 block still reads "Status: not started" while the Implementation log records completion; in an auto-dispatch window the plan is the shared memory, and stale status fields are the same drift class as challenge item 1 (uncascaded plan text) — at plan Workstream 1 block, Status line
    Rubric: no entry — plan bookkeeping, orchestrator-owned
```

Reconciliation notes (blind findings vs. planning-round history):

- The post-grill rubric-check credited WS1's non-RangeIndex tests as "an independent baseline, which defeats the shared-baseline trap." Strictly, the RangeIndex twin runs the same shipped code, so those six tests prove *index-invariance*, not selection correctness. What actually defeats mode 4's shared-baseline trap is the storage test's independent `np.isin`-based recomputation plus its `ids == edges[positions]` assertion — and that shipped, so the chain is closed. Carry the distinction into WS4: its mirrored polars output-equivalence gate (rubric-check must-fix, already routed) gets true independence from the cross-engine comparison against the pandas-validated path, and should keep that as its anchor.
- The harness-gated done-when bullet (no-speed-regression sweep + recorded membership-storage delta) is honestly named as remaining in the Implementation log; the Wave 4 commit mechanics (commit only on a green full gate battery) are the mechanical arming trigger, so this is not a dormant-gate finding.

**Workstream 2 — reduction-aware chunked rasterization (2026-07-03)**

```
Status: propose
Artifact reviewed: Workstream 2 (stream_chunk_threshold + streamed combine paths on the three
  datashader entry points; armed perf gates; 10M stress gate; canary deletion); uncommitted
  working tree on branch 53-scale-hiveplotlib-to-larger-networks
Dispositions held: yes — the 2026-07-02 arming amendment held in full: (a) the three dormant
  gates unskipped and wired via explicit per-path threshold pins (FORCE_STREAMED_THRESHOLD /
  FORCE_SINGLE_SHOT_THRESHOLD, so retuning the auto threshold cannot change what a gate
  measures), (b) canary git-rm'd with zero surviving references (README scrubbed; the
  constants-consistency test never bound it and still binds seed + the four ASV scales; the
  stress constants' deliberate exclusion is documented on both sides), (c) the 0.5 RSS constant
  KEPT — a pre-registered bound passed at 0.466, not a constant shaped to pass — with the
  WS2-alone vs WS2+WS3-cumulative split (~1.6 GB resident curves in both children, WS3's job)
  and an explicit no-quiet-retuning revisit trigger recorded at the constant; the trigger is
  mechanically armed (a false-fail surfaces as a gate failure, not a prose condition nobody
  re-checks). The memory-gates-always-run amendment held (stress skip keyed to TOTAL RAM only,
  never available RAM; serial lane 35 passed / 0 skipped). The per-mode unit-test amendment
  held (streamed path, single-shot selection, auto/forced threshold policy, and var/std
  carve-out each carry a dedicated default-suite @pytest.mark.datashader test). max/min
  classification recorded as a stated decision in the Implementation log per the done-when.
  Planning item 4's strengthened disposition (no silent memory-gate skips) held in the
  artifact. No scope ballooning: the matrix path is reached through existing kwargs plumbing
  (verified, tests/hiveplot_matrix_test.py:2283), hiveplot_matrix.py untouched;
  "tests/viz/datashader_test.py (or equivalent path)" landed at tests/datashader_test.py.
Concerns:
  - [must-fix, CLOSED 2026-07-03] Streamed ds.mean divides by the all-rows count, not the column's non-NaN count:
    single-shot datashader mean excludes NaN-valued rows from its denominator, but the streamed
    path's count_agg (plain ds.count()) counts every drawn row, so any NaN in the mean column
    yields a silently different raster from the single-shot baseline that all three docstrings
    and the CHANGELOG promise to match — on both edge and node streamed paths, whenever
    streaming engages (auto at >= 1M items, or forced) — at
    src/hiveplotlib/viz/datashader.py:312-315 (edges; nodes at :380-383, divide at :198-202)
    Rubric: Failure mode 4 (plausible picture of the wrong data) — every equivalence fixture
    ("low", "weight") is NaN-free, so all gates stay green while the picture lies
    Push: for combine_mode "mean", accumulate the denominator as ds.count(reduction.column) (a
    separate aggregate from the plain-count density correction, which is correctly shared with
    single-shot), and add a NaN-holding-metadata case to the streamed-vs-single-shot
    parametrized tests so the fixture class the mode names is actually exercised.
    Closure (2026-07-03, adversary delta-read on the fix diff; skeptical re-verification, not
    log-trusting): CLOSED. (a) Two-denominator wiring confirmed against single-shot on both
    paths: datashader's own mean composes sum(col) / count(col) (non-NaN count) inside the
    aggregation, i.e. pre-spread, and the streamed mean divide consumes the new per-chunk
    ds.count(<mean's column>) accumulation (add-combined; exact integer addition) at
    src/hiveplotlib/viz/datashader.py:203-209 before the spread at :210, while the retained
    all-rows count_agg feeds only the post-spread density divide at :214-219; single-shot's
    counterparts are the in-aggregation mean finalize plus the spread all-rows ds.count()
    divide at :650-654 (edges) and :859-863 (nodes). Ordering and denominator each map to
    their exact single-shot twin on both streamed paths (:325-332 edges, :405-412 nodes).
    (b) The NaN tests bind, not just exist: fixture mutations are live (node_placements is a
    plain attribute, edges._data mutated in place, both read fresh at render); at 20% NaN the
    old all-rows denominator diverges by the per-pixel NaN share, ~12 orders of magnitude past
    rtol=1e-12, so no regression hides in tolerance. Edge side analytically guaranteed
    (NaN-edge curves share hundreds of pixels with valid curves); node side requires a
    seeded same-pixel NaN/valid collision and is confirmed by the implementer's out-of-tree
    50-67% divergence measurement. (c) No division artifacts: combined sum is non-NaN iff the
    non-NaN count >= 1, so the only zero-denominator case is NaN/0 -> NaN, matching
    single-shot's counts>0 gate; warning-free under filterwarnings=error (suite green, 1412).
    (d) No leak: mean_count_agg is gated on combine_mode == "mean" at both accumulation sites
    and in _finalize_streamed_raster; count/sum/any/max/min and the var/std carve-out are
    untouched, pinned by the new sum/count NaN cases and the bit-exact count assertion.
    Residual (low-confidence, does not reopen): nothing executable pins the node fixture's
    mixed-pixel sensitivity — a future seed or num_nodes edit could silently defang the
    node-side mean NaN test (same class as the WS1 fixture-sensitivity item that earned an
    in-fixture assert); cheap hardening would assert in the fixture that some aggregation
    pixel mixes NaN and valid "low" nodes. The edge-side test independently guards the shared
    finalize logic either way.
  - [worth-discussing] The shipped equivalence gate is quietly looser than the plan's contract
    language for sum: the done-when demands "Exact match for count/sum/any" (reaffirmed by the
    2026-06-11 algebra amendment: "'exact match for count/sum/any' stands"), but chunk-order
    float addition makes exact sum unattainable and the tests correctly assert
    rtol=1e-12/atol=1e-14 — the code is right, the plan language is wrong, and the deviation
    shipped documented in tests/log but never routed to amend-plan — at
    tests/datashader_test.py:491-498, :536 vs the WS2 var/std-equivalence done-when bullet
    Rubric: no entry — contract-language drift (the same no-quiet-loosening principle the
    2026-07-02 arming amendment encodes)
    Push: amend-plan to restate the exactness set (count/any/max/min exact; sum/mean bounded
    float-associativity tolerance, bound stated). Downstream bearing: WS3's fused path inherits
    these same gates and would be judged against the same impossible "exact" wording — route
    now, not to plan-end batch.
  - [worth-discussing] The matrix-level memory test the WS2 pattern block calls "also wanted"
    was deferred unilaterally in the Implementation log ("no matrix-level RSS gate... roughly
    double the stress-lane cost"); the rationale is plausible and the forwarding equality test
    covers the wiring, but a plan-text want disposed only in the log is invisible to the
    plan-as-shared-memory — at "Patterns this replaces" (WS2, HivePlotMatrix bullet) vs the
    Implementation log WS2 entry
    Rubric: no entry — deferred-work bookkeeping (classify, never silently drop)
    Push: one-line amend-plan disposition: defer to WS3's resident-memory work (where
    matrix-level peak — the sum of per-cell resident geometry — actually bites) or reject with
    the recorded rationale. Downstream bearing: WS3.
  - [low-confidence] The stress-scenario comments overclaim their proportionality lineage: "the
    same 20:1 edge:node ratio as every pinned scenario above" is false (tiny ~2.3:1, small
    2.5:1, medium 10:1; only large is 20:1) — the rider's substance holds (matches the large
    scenario's shape, no token node counts; the 9.28 GB build-only child proves node-side cost
    is exercised) but the gate rationale misstates the facts it cites — at
    tests/_performance_regression_workloads.py:40 (same slip at
    tests/performance_regression_test.py:248-249)
    Rubric: Failure mode 2 (the rider held; its recorded justification should be exact)
  - [low-confidence] Pre-existing bug spotted during the attack, believed not WS2's diff:
    `if dpi not in fig_kwargs:` tests the integer dpi VALUE against the dict's keys, so it is
    always true and a caller's fig_kwargs={"dpi": ...} is silently clobbered by the dpi
    parameter (contrast the correct "figsize" string check one line up) — at
    src/hiveplotlib/viz/datashader.py:501 and :746
    Rubric: no entry — out-of-scope observation; trivial fix ('"dpi" not in fig_kwargs')
    wherever routed.
```

Attack notes (blind step, recorded for the audit trail):

- Angles that came back clean: the **10M denominator switch** is defensible and honestly recorded — the build-only denominator isolates the render transient (both children share the ~8-9 GB resident geometry) and keeps the gate always-runnable on the 32 GB host; measured 1.107 vs ~1.9 predicted for a full-copy transient, so the plan's transient-at-chunk-scale-not-total-scale claim is still what is proven. Caveat worth remembering, not a finding: the 1.35 bound tolerates up to ~3x chunk-scale transient, and when WS3 shrinks the resident share the same constant becomes much stricter — renegotiate it there, explicitly. **`any` OR-combine**: no addition path survives (the combine table routes `any` to logical_or; the only additive count aggregate is the separate density/mean denominator, which is a count reduction, and the ds.any("col") density-correction quirk mirrors single-shot exactly, preserving equivalence). **var/std carve-out**: provably unreachable by the threshold — the exact-type combine-mode lookup gates `_use_streamed_rasterization` before any threshold comparison, so var/std/first/last/count_cat and every unlisted reduction (subclasses included) route single-shot; covered by the policy unit test plus the gallery-mirroring var (nodes+edges via wrapper) and std byte-equality tests. **Silent 1M default flip**: docstring + CHANGELOG obligations met on all three entry points; below the threshold the else-branch is the byte-identical pre-WS2 code — but the flip's safety at/above the threshold IS the equivalence contract, which is what elevates the mean/NaN concern to must-fix.
- **Mode 1 verified independently, not taken from the log:** opened `/tmp/ws2_review/streamed_render_10m_edges_500k_nodes_default_raster.png` — six labeled axes (A/B/C + repeats), per-pair density lobes, and log-density gradients all legible; the right blob at the shipped raster resolution. The dispatching session should still land the image in the between-workstream review packet before /tmp is cleaned.
- Reconciliation notes: the api-critic's post-impl "decided non-concern" on advertising ~1e-15 noise in the user docstring is distinct from concern 2 here, which is about the plan's gate language, not user docs — no re-litigation. Implementation-log claims spot-checked rather than trusted: the matrix-level equality test exists and asserts per-cell raster equality under forced streaming; the canary grep, constants-consistency binding, and armed-gate skip logic were verified directly in the tree.

**Workstream 3 — fused build + internal streaming (2026-07-04)**

```
Status: propose
Artifact reviewed: Workstream 3 (fused build-and-rasterize: _construct_edge_subset_curves
  extraction + control-angle passthrough; _chunk_edge_curves get-or-build seam; fused perf
  gates incl. FUSED_PEAK_RSS_MAX_FRACTION_LARGE=0.30; TimeRenderMatplotlib canary); uncommitted
  working tree on branch 53-scale-hiveplotlib-to-larger-networks
Dispositions held: yes, with one flagged extension (concern 1) — the exactness-set correction
  (disposition 2) cascaded into the fused gates exactly as restated (bit-exact count/any/max/min
  via _STREAMED_REDUCTION_EXACTNESS; rtol=1e-12/atol=1e-14 sum/mean at
  tests/datashader_test.py:538), so WS3 was judged against attainable language. The matrix
  memory assertion (disposition 3) correctly took the reject branch of its iff: the shipped
  stress workloads build a BaseHivePlot, so a matrix-level assertion would require the second
  stress-scale workload the bound forbids; no matrix gate shipped, consistent — the
  done-when's "rejection stated in the Implementation log" line is the one piece outside this
  pass's read set, for qa to confirm. The dpi bug (disposition 4) remains untouched at its two
  sites, out of scope held. The WS2 mean-NaN fix (disposition 1) is inherited intact by the
  fused route and extended: the fused NaN-metadata test exercises the two-denominator wiring
  through transiently built chunks. Path A held: _use_streamed_rasterization stays the single
  route selector; no engine or per-path selector appeared. Memory-gate policy held: fused
  stress gates skip only on total RAM, never available; serial lane 39 passed / 0 skipped.
  WS2's dormant caveat (1.35 stress bound becomes stricter "when WS3 shrinks the resident
  share") correctly did not fire: WS3 added separate fused constants and left the streamed
  gate's curve-resident children untouched, so 1.35's character is unchanged. Scope: beyond
  concern 1, the diff stays inside the WS3 Files list (gates landing in tests/ rather than
  runners/performance/ follows the WS2 layout precedent).
Concerns:
  - [worth-discussing] The ids-only rendering extension shipped past the plan's literal text on
    the implementer's flag alone, and it narrowly breaks the done-when's "does not change the
    default two-stage behavior": chunk collection now gates on stored ids, so a
    partially-constructed instance (ids stored both directions, curves deliberately built for
    one) renders its unconstructed subsets with default-parameter geometry on the datashader
    path, where WS2 excluded them — silently diverging from the vector backends on the same
    instance, at any threshold, no opt-in — at src/hiveplotlib/viz/datashader.py:628-639
    (collection) and :655-659 (single-shot transient build) vs the WS3 done-when's
    no-default-change bullet. Verdict on the code itself: the extension is right and the
    single-shot half is forced — restricting transient builds to the streamed route would make
    picture CONTENT flip with stream_chunk_threshold, a worse violation of the same bullet
    (threshold as semantic selector). The ONE-policy decomposition held: route selection purely
    _use_streamed_rasterization, curves-presence as orthogonal per-chunk state, symmetric on
    both routes; no hidden second selector. Behavior is tested (mixed-persistence, ids-only
    single-shot cell) and documented (both entry-point docstrings, construct_curves note,
    CHANGELOG).
    Rubric: no entry — plan-contract deviation shipped without routing (the exact class
    disposition 2's process note retroactively routed for WS2)
    Push: retroactive amend-plan disposition, WS2-precedent shape: supersede the
    no-default-change sentence with the ids-as-source-of-truth semantics, and record the
    vector-backend divergence for partially-constructed instances as accepted (or scope it).
    Downstream bearing — route now, not plan-end batch: WS5's Dask/cuDF path rides this same
    policy and inherits ids-only semantics; WS6/WS7 prose will describe render behavior.
  - [low-confidence] Refactor bit-identity vs the PRE-refactor tree is argued structurally, not
    pinned executably: every new equivalence test compares twins that both route through the
    shared _construct_edge_subset_curves (tests/hiveplot_test.py:2376 compares the helper to
    construct_curves, which now delegates to it; the fused raster gates compare the helper to
    itself), so the battery cannot detect a divergence introduced by the extraction — the
    shared-baseline trap the WS1 reconciliation note named, and no golden fixture exists
    in-tree (assert_raster_equivalent is same-run two-instance). What holds it up: the
    control-angle passthrough at src/hiveplotlib/hiveplot.py:1297-1301 is the conservative
    choice (direct calls keep historical (axis_id_1, axis_id_2) anchoring for BOTH directions;
    construct_curves keeps per-direction (from, to) since it delegates with a2_to_a1=False per
    stored direction; the fused default matches construct_curves — confirmed bit-exact across
    all nine chunks, bidirectional and repeat-axis pairs included, by the count gates), plus
    pre-existing relative-geometry tests. Residual, not a defect found.
    Rubric: Failure mode 4-adjacent (shared-baseline equivalence topology)
    Push: optional hardening only — a tiny-scenario pinned-value smoke (small golden array)
    would pin the refactor class for good; otherwise accept the structural argument.
  - [low-confidence] The fused stress gate's 4.5 bound has an unnamed false-fail drift vector:
    its denominator (ids-only assembly child) excludes the datashader/matplotlib import stack
    the numerator carries, so dependency-upgrade RSS growth erodes the ~0.77 GB headroom
    (4.5 x 0.975 GB vs 3.628 GB measured) with no hiveplotlib change; the comment names the
    import delta but carries no revisit trigger, unlike the 0.5 constant's explicit
    no-quiet-retuning language — at tests/performance_regression_test.py:125-136. The bound
    itself BINDS today: headroom is below one chunk (~0.9 GB), so even a single-retained-chunk
    residency regression fails gate (a), and full materialization fails both braced bounds by
    >2x — no regression hides between them.
    Rubric: no entry — gate-comment completeness (the recorded justification should carry its
    own revisit condition)
    Push: one sentence at the constant: on false-fail, re-measure the import delta explicitly
    before touching 4.5.
  - [low-confidence] Stale WS2 docstring contradicts the new collection rule:
    _streamed_edge_raster's chunk_axis_pairs param still reads "pairs holding at least one
    curve for tag" while collection now gates on stored ids (the call-site comment at :628-630
    says so correctly) — at src/hiveplotlib/viz/datashader.py:322.
    Rubric: no entry — doc drift inside the shipped diff's own seam; one-line fix.
```

Attack notes (blind step, recorded for the audit trail):

- Angles that came back clean: **the 0.30 fused constant is honest, not shaped** — measured
  0.208 with ~44% relative headroom, the named failure mode (full residency) lands near ~0.5
  and fails clearly, and both children pin their render path via explicit thresholds so auto
  retunes cannot shift what the gate measures. **Keeping the 0.5 streamed constant un-retuned
  is correct by construction, not just argued**: WS3 added separate fused gates and never
  touched the 0.5 gate's children, which both still construct curves at build time, so its
  measured ratio (0.466) does not move with WS3; the constants file cross-references the fused
  analog as "deliberately tighter", making the relationship legible in-artifact.
  **Metadata carry cannot skew**: the repeat factor is computed from the row count of the
  actual array feeding the canvas (persisted or transient), metadata ordering is the same
  ids/relevant_edges path the persisted route always used, and the fused NaN-metadata test
  binds the inherited two-denominator mean wiring. **The discard contract is structural, not
  just tested**: the render path contains zero write sites to hive_plot_edges (helpers return
  arrays; single-shot accumulates into locals), so an error mid-render cannot leak a partial
  write; tests additionally pin object identity of persisted arrays through a streamed render
  and nothing-persisted through the full-plot wrapper. **No silent custom-geometry path**: the
  datashader entry points accept no construction parameters end to end, so transient builds
  can only ever use defaults, and the rebuilt-every-render / run-construct_curves-for-custom-
  geometry caveat is present at both entry-point docstring sites plus the construct_curves
  note. **Render canary matches its amendment**: TimeRenderMatplotlib, matplotlib only,
  tiny+small, with the one-canary-for-the-family rationale recorded in the class docstring;
  no bokeh/plotly/holoviews benchmarks added. **Dedicated default-suite unit tests** for route
  selection, the get-or-build seam, and discard behavior run under @pytest.mark.datashader in
  the parallel coverage lane, reusing the workload module's scenario recipe (one definition,
  no drift).
- Tiny-floor fused timing gate does untimed warmup of each path before sampling and takes
  best-of-3 per side, so neither side inherits first-call cost (the same-run timing-ratio
  gotcha checked and clear).
- Process caveat for the dispatching session: this pass could not run git status (no shell);
  given the 2026-07-04 CRLF gotcha on datashader.py, sweep git status for CRLF warnings before
  commit.

**Workstream 4 — narwhals at the input boundary (2026-07-04)**

```
Status: propose
Artifact reviewed: Workstream 4 (frames.py boundary module + NodeCollection/Edges/HivePlot
  narwhals dispatch + boundary-gap follow-up: eight styling-gather sites + add_nodes;
  narwhals>=2.0.0,<3.0.0 core dep); uncommitted working tree on branch
  53-scale-hiveplotlib-to-larger-networks
Dispositions held: yes — challenge item 7's bound re-derivation (1836 amendment) fired at Gate
  Zero and caught the dead 1.x major (the checklist earned its keep, recorded honestly, not
  retrofitted); the rubric-check must-fix's equivalence gate (1864 amendment) over-delivered
  (bit-exact, no tolerance: placements incl. dtypes, relevant_edges membership, stored ids,
  curves, edge/node rasters across single-shot/streamed/fused, plus the follow-up's styled-render
  battery and to_json string equality); the boundary-gap disposition items 1-2 held (eight sites
  converted; the eighth surfaced-then-converted with a necessity justification — the mandated
  hover twin test cannot pass without it — not quiet ballooning; add_nodes mirrors add_edges
  verbatim incl. the legible mixed-library TypeError). Interaction §4's WS1 rework landed as
  positional gathers with the pandas fast-path preserving the WS1 idiom per site. No scope
  ballooning beyond the recorded eighth site; frames.py itself is the architecture the
  amendment's fix implies (Files-list absence is bookkeeping, concern 7).
Concerns:
  - [must-fix] The advertised engine set is broader than the shipped surface: CHANGELOG and
    docstrings claim "any eager narwhals-supported dataframe (e.g. polars, pyarrow)", but seven
    user-frame `.columns` attribute reads were knowingly left un-routed (log: "work on polars
    as-is") — node-styling membership checks at viz/matplotlib.py:316, viz/plotly.py:478,
    node-hover column lists at viz/plotly.py:451, viz/bokeh.py:412, viz/holoviews.py:460, and
    edge-hover column lists at viz/bokeh.py:614, viz/holoviews.py:743. `pyarrow.Table.columns`
    is a list of ChunkedArrays, not names: column-mapped node styling silently fails to resolve
    and hover tooltips render garbage on an engine the CHANGELOG names — confident wrong output
    with every gate green, because the gates are polars-only. The corrected Holdouts bullet
    still claims the backends touch "internally produced frames only", so qa's eight-site sweep
    passes while these user-frame reads remain.
    Rubric: Failure mode 4 (engine-flavored: gates green, picture/annotations lie)
    Push: either finish the sweep mechanically (route the seven reads through hashable_columns —
    already imported in three of the four backends) or narrow the claim (drop pyarrow from the
    advertised examples; record pandas+polars as the verified set, other engines best-effort)
    and align the Holdouts bullet. Downstream bearing: WS5's cuDF passthrough flows through
    these same sites (cudf `.columns` duck-types, but the decision of claim-vs-sweep should
    land before WS5 inherits it).
  - [must-fix] Three datashader sites route pandas through narwhals with no isinstance
    fast-path, regressing the degenerate-pandas contract the fast-path exists to preserve:
    viz/datashader.py:286 (as_eager_frame on the user edge frame) and :434/:903 (as_eager_frame
    on node_placements — which the plan documents as ALWAYS pandas, so the wrap guards against
    a library that cannot occur there). The implementer's own record states narwhals raises
    DuplicateError on duplicate-column pandas frames and only pandas honors non-string column
    names, and verified such frames "still accepted" — at the boundary. But placements carry
    every user node column, so a pandas hive plot with a duplicate or non-string node column
    name now fails at every datashade_nodes_mpl call (:903 is the unconditional single-shot
    path), and a duplicate/non-string-column edge frame fails metadata reductions at :286,
    where 0.28 pandas ops worked. Contradicts the done-when's "pandas ... identically to
    today", the CHANGELOG's "pandas input behaves exactly as before", and the log's
    '"pandas path unchanged" holds literally'. No test covers the class (all fixtures use
    string columns).
    Rubric: no entry — done-when breach (the pandas no-regression bullet)
    Push: fast-path the three sites (placements are guaranteed pandas — drop the wrap or
    isinstance-branch it; :286's job is exactly column_values_at_positions, which already
    carries the fast-path), plus one degenerate-pandas datashader test (dup-column or
    int-named metadata column) to pin the contract where it now actually bites.
  - [worth-discussing] The one recorded cross-engine divergence is exactly the one thing the
    bit-exact gate never exercises: every twin fixture passes explicit labels
    (labels=["A","B","C"] in tests/polars_test.py and tests/hiveplot_test.py), so the
    labels=None stringified-Interval axis-ordering divergence (lexicographic sorted() at
    hiveplot.py:440 vs pandas groupby's numeric Interval order; diverges at multi-digit bounds)
    ships documented-but-never-executed. Endorsing the api-critic's doc-sentence must-fix, one
    extension: add a single polars-marked twin at multi-digit default-label bounds pinning the
    documented divergence itself (pandas order numeric, polars order lexicographic-string), so
    a future narwhals/polars change cannot silently morph the documented behavior into a new
    undocumented one. Axis order is plot content, not cosmetics: it changes adjacencies and
    which edge chunks exist.
    Rubric: Failure mode 4 (the gate-shaping residue of the disposed 3(c) record)
  - [worth-discussing] set_axes_order's None-collapse breaks on polars for non-string partition
    values: the pandas branch is deliberately engineered to preserve int partition keys
    (dtype=object comment at hiveplot.py:3410-3416), but the non-pandas branch hands the mixed
    int+collapsed-name-string object array to nw.new_series (via with_appended_column at
    :3428), where polars strict inference raises an opaque mid-method error. The only polars
    collapse test uses string labels (tests/hiveplot_test.py:7919).
    Rubric: no entry — flagging anyway (loud failure, but an engineered pandas behavior with no
    polars mirror and an illegible error on a supported surface)
    Push: legible error or stringify-on-collapse for non-pandas engines (consistent with the
    partition-label stringification precedent) + an int-valued polars collapse test.
  - [low-confidence] The measured 2.0.0 floor covers the 43 pre-follow-up polars tests; the
    follow-up added 10 more (53 total, incl. the styled-render battery over the three new
    frames.py helpers) that never ran against narwhals==2.0.0. The helpers' narwhals surface
    (from_native/getitem/to_numpy/columns/new_series) is a subset of what the 43 exercised, so
    the floor is probably still honest — one `-m polars` re-run against 2.0.0 at qa (or a
    recorded subset rationale) closes the gap.
    Rubric: no entry — bound-honesty hygiene
  - [low-confidence] The uid-from-row-numbers fallback is documented and tested, but its one
    silent-misalignment residue isn't named anywhere user-facing: a pandas frame whose index is
    a permutation of 0..n-1 (e.g. post-.sample) gets index-value uids; pl.from_pandas drops the
    index, so the same ids name different rows on the polars twin — valid-looking ids, wrong
    nodes, no error. One pitfall line in the pending polars gallery notebook ("pass
    unique_id_column explicitly when moving frames between engines") closes it.
    Rubric: Failure mode 4 (user-triggered, library-documentable)
  - [low-confidence] Plan bookkeeping drift, same class as WS1's stale Status line: frames.py,
    tests/frames_test.py, and tests/polars_test.py are absent from WS4's Files list though they
    are the workstream's architecture; and the Files/done-when text still carries the dead
    `>=1.x,<2` literal the 1836 amendment's checklist item (c) superseded (the Status line
    records the re-derived bound, the normative bullets don't).
    Rubric: no entry — plan bookkeeping, orchestrator-owned
```

Attack notes (blind step, recorded for the audit trail):

- **Two-code-path drift (the fast-path's standing risk) came back pinned, not just asserted**:
  at the boundary and all eight follow-up sites the polars twin battery compares the pandas
  branch (the verbatim pre-refactor idiom, per site) against the narwhals branch — true
  dual-path topology, not the shared-helper masking the comparison-topology check warns about.
  Every frames.py helper has both branches exercised via ENGINE_PARAMS parametrization, with a
  pandas positional-vs-label divergence case (tests/frames_test.py:219) pinning the semantics
  that matter. The three datashader sites are the exception — both twins share the narwhals
  path there — but pre-refactor pandas output at those sites is pinned by the WS2/WS3
  golden/equivalence batteries, which ran green (default suite 1518, serial lane 39/0/0), so
  fidelity is anchored by an independent in-tree artifact. (Their remaining problem is concern
  2, degenerate pandas, not string-column drift.)
- **"Bit-exact" is honest**: assert_array_equal / assert_frame_equal (dtypes included) /
  serialized-figure and to_json string equality, no hidden tolerance, fused and streamed routes
  included; done-when asked for harness tolerance, shipped stronger.
- **Hot loops clean**: no narwhals or frames imports in utils.py, axis.py, or the numba
  modules; every helper call site is per-plot/per-axis/per-chunk (the plotly per-edge .iloc is
  the pre-existing shape on an internal pandas gather). Render canary 191/212 ms vs 210/299 and
  211/203 dev refs — in band; dev-mode numbers are sanity checks, the formal per-boundary ASV
  capture at merge remains the durable record.
- **Ninth-site sweep**: no label-based gathers (.loc / set_index / np.where-on-columns) remain
  on user frames anywhere in viz/ or to_json; what remains is the `.columns` attribute family
  (concern 1). The eighth site's conversion is behavior-identical on pandas
  (hashable_columns preserves order; .index() first-match == np.where(...)[0][0]).
- Deduped against the api-critic's WS4 items (P2CP holdout, [polars]-extra framing, CLAUDE.md
  marker list, frames.py public shape, test-name contract on test_add_edge_ids_polars): not
  re-raised here; concern 3 extends rather than repeats its labels=None must-fix.
- Process caveat repeated from WS3: no shell in this pass; sweep git status for CRLF warnings
  before commit (2026-07-04 datashader.py precedent).

**Workstream 5 — Dask + cuDF/GPU passthrough (2026-07-05)**

```
Status: propose
Artifact reviewed: Workstream 5 (frames.py as_frame/LazyEdgeSubset; __store_edge_ids lazy
  branch + _add_edge_ids_lazy; datashader _rasterization_route / _dask_native_edge_raster /
  NaN-sentinel partition builder / _curves_from_id_pairs extraction / _to_host_array + cuDF
  collection frames; DaskComputationError; tests/dask_test.py (38) + tests/cudf_test.py (14);
  perf-lane Dask scenario; Makefile test-gpu; CONTRIBUTING GPU section; pytest.ini; CHANGELOG);
  uncommitted working tree on branch 53-scale-hiveplotlib-to-larger-networks, judged against
  the WS5 block as amended through 2026-07-05 (RMM instrument language)
Dispositions held: yes — Path A held to zero tokens (no `use_dask` anywhere in src/tests/
  examples/docs); the no-[dask]-extra disposition held (pyproject's only dask dep rides the
  datashader extra, the module guard names `hiveplotlib[datashader]`, and no hiveplotlib-
  authored frame-detection ImportError exists — as_frame raises TypeError with the legible
  accepted-inputs message, dask.delayed/dask.array/polars-LazyFrame rejections all pinned);
  the 2026-07-05 measured-deviations disposition held in the artifact (RMM statistics is the
  shipped instrument with the cupy-pool-reads-zero rationale recorded in the test module
  docstring; the float-reduction limitation is pinned raise-or-match; tier-1 Dask instrument
  selection is recorded verbatim at BOTH consuming sites with the deciding fact verified
  in-env; the pytest.ini ignore is message-scoped exactly as accepted); WS3's ids-only
  semantics inherited cleanly (lazy subsets never persist curves; construct_curves/to_json/
  vector-warn guards all fire legibly and are tested); WS4's .columns sweep and pandas
  fast-path restorations held at every WS5-inherited site (metadata reads via
  hashable_columns/column_values_at_positions). Scope: no ballooning; tests landed at
  tests/{dask,cudf}_test.py rather than the Files list's tests/integration/ paths (WS2 layout
  precedent), and LazyEdgeSubset landed in frames.py, extending the WS4-flagged Files-list
  bookkeeping — record, not creep.
Concerns:
  - [worth-discussing] The discharged VRAM gate's streamed figure is suspect: 16,315,420 bytes
    is 2.0007x the single-shot 8,155,004, the exact shape of a shared-statistics-epoch
    measurement (RMM peak_bytes is monotone and the in-tree test's own pattern runs both
    routes sequentially in one process with no reset between them, tests/cudf_test.py:202-206)
    — the parsimonious read is phase-1 retention/ordering contamination, the face-value read
    is "streaming DOUBLES VRAM", which contradicts the streamed route's bounded-by-largest-
    chunk story; nothing in-tree measures per-route isolated peaks (the test asserts only
    peak > 0 and total growth), so the amendment's "per-process VRAM peak (exact)" is exact
    for the instrument but unisolated per route — at tests/cudf_test.py:180-210 and the
    2026-07-05 amendment's discharge record.
    Rubric: Failure mode 3 (a memory number that would not hold as framed; it becomes a
    published number the moment the cuDF notebook or WS6 quotes it)
    Push: re-measure each route in a fresh process (or a pushed/popped statistics scope per
    route), correct or annotate the recorded pair, and gate the numbers' flow into the pending
    cuDF notebook / WS6 on that re-measurement. Downstream bearing — route now, not plan-end
    batch: cuDF notebook, WS6 blog, chain-close instrument records all consume these numbers.
  - [low-confidence] The NaN sentinel promotes the metadata column to float64 on the Dask
    route only (np.concatenate with np.array([np.nan])); with an INTEGER metadata column,
    datashader's int-sum (zero-fill) vs float-sum (NaN-fill) untouched-pixel semantics can
    diverge from the pandas twin, and every dask fixture uses a float weight, so the class
    never runs — the one clean-fixture residue in an otherwise strong 8-reduction battery —
    at src/hiveplotlib/viz/datashader.py:684-687.
    Rubric: Failure mode 4 (the clean-fixtures-never-exercise shape)
    Push: one int-weight parametrization on the dask equivalence twins; pass or a documented
    fill-divergence note, either is fine.
  - [low-confidence] Nothing pins the sentinel workaround's test sensitivity: no recorded
    mutation check that DELETING the sentinel fails a named test. The empty-partition-with-
    metadata test and the 1-vs-4 partition-count invariance test are the plausible nets (via
    dd.concat they do create empty-between-populated topology), but whether they reproduce
    the upstream mis-stitch is unverified — same guard-sharpness class as the WS1 fixture-
    sensitivity item — at tests/dask_test.py:452-488, :723-730 vs the sentinel at
    src/hiveplotlib/viz/datashader.py:641-690.
    Rubric: Failure mode 4 (guard tests must stay sharp to keep the mode closed)
    Push: one-time local mutation run (drop the sentinel, record which test fails, name it in
    the sentinel comment or the log). Forward-compat is clean either way: a NaN vertex draws
    nothing, so an upstream fix cannot make the sentinel harmful.
  - [low-confidence] Mixed same-tag lazy + in-memory subsets crash illegibly: two
    add_edge_ids calls from DIFFERENT Edges instances (one lazy, one eager) under the same
    tag on different axis pairs pass the per-call resolver, then _dask_native_edge_raster
    hits the eager pair's ndarray ids with .from_column_name/.filtered_native →
    AttributeError, where the within-one-Edges case gets the legible mixing TypeError — at
    src/hiveplotlib/viz/datashader.py:733-735, :753-754.
    Rubric: no entry — flagging anyway (exotic usage, loud but illegible)
    Push: isinstance guard in the chunk loop raising the existing avoid-mixing message.
  - [low-confidence] The cuDF moment-reduction smoke's `except Exception: return` passes on
    ANY raise, including a hiveplotlib-side crash in those four reductions (bounded: count/
    any/max/min assert strictly with no try/except) — pin the raise's origin (match NVVM /
    kernel-compilation in the message) so "raise" keeps meaning "upstream unsupported" — at
    tests/cudf_test.py:243-249.
    Rubric: no entry — test-discrimination hygiene.
```

Attack notes (blind step, recorded for the audit trail):

- Angles that came back clean: **lazy-route threshold silence is documented and pinned, not
  silent** — all three entry-point docstrings state lazy input never consults
  `stream_chunk_threshold`, and a dedicated test asserts identical rasters at 0 / None / 10^18
  (the WS2 explicit-force-silence concern answered properly on the new surface; the P2CP
  surface cannot reach it, pandas-only holdout). **Delegation equivalence is honest**: the
  twins' edge SELECTION is genuinely independent (np.isin masks + positional storage vs
  narwhals is_in predicates), the membership twin test compares stored ids to lazily-filtered
  content exactly, tolerances are recorded and justified (bit-exact count/any/max/min;
  rtol=1e-9 for sum/mean/var/std with the cross-partition-addition-order rationale in the
  docstring — recorded slack, not quiet slack), and cross-partition combining is really
  exercised (10k edges / 2 partitions with heavy pixel overlap, plus 1-vs-4 partition
  invariance). Shared-geometry residue (_curves_from_id_pairs feeds both twins) is the WS3
  low-confidence refactor item, already on record; not re-raised. **The non-materializing
  claim is proven, not asserted**: the poisoned-partition test builds a full HivePlot over a
  frame that raises on ANY materialization — construction succeeding is a structural proof
  that add_edge_ids/store composes predicates only; to_json (both classes), construct_curves,
  and the vector-backend warn path are guarded and tested; the only lazy computations in warm
  paths are aggregation-only (uniqueness count, partition values, __len__) and each is
  documented at its site. **The sentinel is aggregation-inert by construction** (NaN
  coordinates draw no segment, so the sentinel's metadata NaN can never enter any reduction,
  mean/var included), and the bit-exact count/max/min twins double as the leak net; the
  DaskComputationError reraise chains the original and its byte arithmetic is correct.
  **_to_host_array's fake-cupy test is not coverage theater**: it pins the module-name
  detection and .get() dispatch (the only CPU-decidable semantics), while the real
  device-to-host transfer is exercised end-to-end by the GPU lane's strict-match reductions —
  the pairing, plus the CONTRIBUTING cadence, is the honest split for a never-in-CI path.
  **Scheduler/hardware disclosure (mode 3) holds at every in-tree recorded-number site**: the
  perf workload and perf test both name the threaded scheduler with the verified deciding
  fact; CONTRIBUTING and the Makefile name the dedicated GPU env; no performance numbers ship
  in CHANGELOG or docstrings. The CHANGELOG entry is Path-A-clean (uniform dispatch, no
  parameter, dask-ships-with-datashader).
- Process caveat repeated from WS3/WS4: no shell in this pass; datashader.py was touched
  again — sweep git status for CRLF warnings before commit (2026-07-04 precedent).

*(Chain close: adversary verdicts on WS6/WS7 and the tier-2 pruning record append here.)*

## Workstreams

Workstreams are sequenced as a dependency chain. Each is its own MR with its own validation; each is gated by the separate performance-regression harness plan (`wiki/wiki/plans/performance-regression-harness.md`), which is sequenced first. The original A/B/C workstreams map forward as: A → Workstream 2, B → Workstream 4, C → Workstream 5; Workstreams 1 (membership storage) and 3 (fused build) are new.

### Workstream 1: Membership storage redesign (sparse integer indices)

**Status:** shipped — commit `608575e` on branch `53-scale-hiveplotlib-to-larger-networks` (2026-07-03); full gate battery green (adversary must-fix-free, qa pass)
**Depends on:** performance-regression harness (gating only). No in-plan workstream dependency; this ships first in the chain.
**Files:**
- `src/hiveplotlib/hiveplot.py:893` (def) / `:958` (store site) (`BaseHivePlot.__store_edge_ids`; change `relevant_edges[a1][a2][tag]` storage from boolean mask to integer indices of selected edges)
- **All six read sites, moved in lockstep to positional takes** (grep-verified 2026-07-03; re-grep `relevant_edges` in `src/` at dispatch rather than trusting these line refs): `src/hiveplotlib/viz/matplotlib.py:460-463`, `src/hiveplotlib/viz/bokeh.py:580-581`, `src/hiveplotlib/viz/plotly.py:726-729` + `:774-776`, `src/hiveplotlib/viz/holoviews.py:636-637`, `src/hiveplotlib/viz/datashader.py:203-204`, `src/hiveplotlib/hiveplot.py:4650-4655` (`to_json`)
- Seventh consumer, added on master after WS-1 shipped (added by `edge-coverage-and-gotchas-docs.md` WS-A, 2026-07-06): `Edges._drawn_count()` (`src/hiveplotlib/edges.py`) normalizes `relevant_edges` cell values for `BaseHivePlot.edge_coverage`; at the !36 lockstep reconcile, swap its boolean-mask branch to the index-array branch (`len(reduce(np.union1d, arrays))`). It counts without indexing a DataFrame, so WS-1's positional-take rule does not apply to it. Its two direct `test_drawn_count_*` tests (`tests/edges_test.py`) plant boolean masks into cells and swap to planted index arrays with the helper body; a miss there is loud, not silent (`np.union1d` over planted bools fails both).
- Verify-not-break (value-type-agnostic): `src/hiveplotlib/edges.py:244-256` (`Edges.copy()` deepcopy + docstring wording), `src/hiveplotlib/hiveplot_matrix.py:693` (shared matrix-cell reference), `src/hiveplotlib/hiveplot.py:782-835` (reset/pop bookkeeping)
- `tests/hiveplot_test.py`, `tests/viz/datashader_test.py`, and the sibling backend test files as needed (assert membership round-trips correctly; assert storage is integer-index-shaped, not full-length-bool-shaped; non-RangeIndex coverage per done-when)
- `CHANGELOG.rst` (entry for the storage change)

**Done when:**
- `relevant_edges[a1][a2][tag]` stores integer indices of the selected edges, not a full-length boolean mask. Total membership storage across all axis-pairs is O(num_edges), not O(num_pairs × num_edges).
- **Every** `relevant_edges` read site moves in lockstep (the six verified sites in Files: matplotlib, bokeh, plotly ×2, holoviews, datashader, `to_json`), producing byte-identical output across **all** backends — not just datashader — and identical `to_json` output vs. pre-change behavior on the existing test suites.
- **Positional-indexing rule at every read site (2026-07-03):** the integer indices are applied positionally (`.iloc[idx]` / `.take(idx)`), never label-based and never bare bracket selection — `.loc[int_array]` selects by label (wrong rows or KeyError on non-RangeIndex frames) and `df[int_array]` is column selection. No read site may keep `.loc` / `df[...]` selection over the new integer indices.
- **Non-RangeIndex test coverage (Failure mode 4):** the test matrix includes edge frames with a non-default index (at minimum a shuffled/offset integer index; a string index too if cheap), asserting identical selections/output vs. the same data on a RangeIndex, so label/positional divergence cannot pass green.
- All existing tests pass with no regressions (the change is internal; no public entry point moves — but note `relevant_edges` is a non-underscored attribute, so its stored value type changing is the one user-visible edge, carried by the CHANGELOG entry).
- A test asserts the stored value is integer indices (length == number of selected edges) rather than a full-length boolean array, on a multi-axis-pair graph. The index dtype is an explicit implementer decision recorded in the Implementation log (int32 vs int64; the O() claim is dtype-independent, byte-level comparisons are not).
- The in-RAM storage structure is designed to admit the Dask non-materializing equivalent that Workstream 5 supplies (the integer-index choice here is the in-RAM half of one storage decision spanning both workstreams; document the intended Dask counterpart in a code comment or the Implementation log so Workstream 5 does not redesign it).
- Harness-gated validation: the performance-regression harness confirms no speed regression on the standard sweep and records the in-RAM membership-storage delta on a multi-axis-pair graph. Expectation set honestly (2026-07-03): the delta is MBs against the curves' GBs and can go either direction depending on dtype and axis count; WS1's value is structural, so the gate is no-speed-regression plus a recorded delta, not a headline memory reduction.
- CHANGELOG entry added.

**Interactions:** see "Cross-workstream performance interactions" §4 (WS4 must re-express the integer-index gather through narwhals; do not bake pandas `.iloc`-specific assumptions into it) and §3 (the integer-index structure is extended, not redesigned, by WS5's Dask non-materializing storage).

### Workstream 2: Reduction-aware chunked rasterization (datashader backend)

**Status:** shipped — commit `413b743` on branch `53-scale-hiveplotlib-to-larger-networks` (2026-07-03); dormant gates armed + canary deleted; adversary must-fix (streamed-mean NaN denominator) fixed and CLOSED same day; api-critic + adversary post-impl filled; qa pass
**Depends on:** Workstream 1 (consumes the integer-index membership at `datashader.py:203-204`); performance-regression harness (gating).
**Files:**
- `src/hiveplotlib/viz/datashader.py` (primary; refactor `datashade_edges_mpl`, `datashade_nodes_mpl`, verify `datashade_hive_plot_mpl` benefits flow through; add `stream_chunk_threshold` parameter)
- `tests/viz/datashader_test.py` (or equivalent path; add memory-bound tests gated by available RAM, and var/std-equivalence tests)
- `runners/performance/...` (extend the datashader sweep to validate at 10M+ and add peak-memory measurement; coordinate with the harness plan)
- `CHANGELOG.rst` (entry for the refactor and the new parameter)
- `docs/source/...` autodoc for the three datashader functions (signature gains `stream_chunk_threshold`; docstring grows a note on memory characteristics and the var/std carve-out)
- `wiki/wiki/concepts/edge-rendering.md` lines 39-46 ("Rendering Pipeline" gains note on chunked aggregation; post-task research-liaison pass)

**Done when:**
- The three datashader entry points accept `stream_chunk_threshold: Optional[int] = None`. Default `None` auto-decides by edge count; an explicit value forces the switch point. The single-shot path remains the default for the common (small) case.
- Reduction-aware streaming with an explicit per-reduction combine algebra over `(g1, g2, tag)` chunks: count/sum combine additively; **`any` combines via elementwise OR (or max), never addition** (summing boolean `any` rasters produces counts); mean accumulates per-chunk sum + count and divides once at the end (reusing the existing density-correction divide pattern at `datashader.py:242-247`); the chunk's curves array is dropped before the next chunk.
- max/min are cheaply combinable via elementwise max/min. The implementer either supports them in the streamed path or leaves them in the single-shot-or-delegate fallback bucket; either way the classification is a stated decision recorded in the Implementation log, not an accident of the "exotic" catch-all.
- Order of operations on the streamed path: combine **raw per-chunk aggregates** first, then apply `tf.spread` and the density-correction divide exactly once at the end, matching the single-shot code at `datashader.py:236-247` (which spreads and divides after aggregation). Per-chunk spreading before combination would drift from the single-shot baseline and show up as near-tolerance equivalence failures.
- **var/std and any other non-additive / exotic reduction never sum per-chunk rasters.** They either (i) delegate partial-aggregate combination to datashader by feeding it a Dask/cuDF frame (verify the pinned datashader version combines moment-based partial aggregates across partitions; flag the version dependency) or (ii) fall back to the single-shot path. `stream_chunk_threshold` is ignored for these reductions.
- Same streaming shape for `datashade_nodes_mpl` over axes (node points are count-based; additive applies).
- **var/std-equivalence gate:** the streamed-or-delegated path produces output matching the single-shot baseline within tolerance for every supported reduction, verified explicitly against `examples/datashading_statistical_summaries_of_metadata.ipynb`'s `ds.var` (×2) and `ds.mean` (×1) usage. Exact match for count/sum/any; division-tolerance for mean; var/std verified against the single-shot or delegated result.
- All existing datashader tests pass with no regressions (visual or numeric).
- A new memory-bound test confirms peak transient RSS for a synthetic 10M-edge graph on the streamed path stays under O(largest_chunk × num_steps × 4 bytes) rather than O(total_edges × num_steps × 4 bytes). The synthetic scenario scales node count in proportion to edge count, stated in the scenario definition — no token node counts (Failure mode 2). **Memory-gate policy (2026-07-03):** on the canonical host (ringtail, 32 GB) this gate always runs — the RAM-availability skip guard exists for foreign environments only; when RAM is contended the gate serializes after the parallel suite finishes rather than skipping; in the auto-dispatch window a skipped memory gate is a halt-back, not a pass.
- The at-scale streamed render is validated as **readable at the raster resolution we ship/document** — the right blob with visible structure at usable resolution, not merely a returned figure object (Failure mode 1). The between-workstream review packet includes the rendered image so the check is auditable in the auto window.
- **Dedicated default-suite unit tests per rendering mode (2026-07-03 testing directive):** every new rendering mode/branch WS2 introduces (the streamed path, the single-shot selection branch, the `stream_chunk_threshold` auto/forced threshold policy, and the var/std ignore-the-threshold carve-out) carries its own unit tests in `tests/` under `@pytest.mark.datashader`, running in the parallel coverage run and exercising the mode directly as a unit, not merely reached indirectly. Redundancy with the harness's equivalence gates is acceptable and expected: a per-mode unit-test failure localizes which mode broke, where a gate failure only says two paths disagree. Harness-lane tests (`performance` / `perf_harness` markers) contribute zero coverage after the 2026-07-03 marker split, so the 100% gate on `src/hiveplotlib` can only be satisfied by these default-suite tests; a new rendering-mode line without one fails CI.
- Harness-gated validation: the performance-regression harness confirms no speed regression on the single-shot (default) path, records the streamed-path memory reduction at 10M+ edges, and the run passes the var/std-equivalence gate. **This bullet is NOT satisfiable with the harness's streamed-vs-single-shot gates still dormant (2026-07-02 amendment).** Concretely, WS2 completion requires: (a) **unskipping and wiring the three named dormant gates in `tests/performance_regression_test.py`** — the tiny-floor timing band, the large-scenario streamed-vs-single-shot peak-RSS fraction, and the streamed-vs-single-shot raster equivalence via per-side render kwargs; (b) **deleting the canary test that guards the arming** (the `stream_chunk_threshold` signature canary in the default suite, which fails the moment the parameter appears in the datashader entry point's signature); (c) **making the 0.5 RSS-fraction (`STREAMED_PEAK_RSS_MAX_FRACTION_LARGE`) unskip decision explicitly against the WS2-alone vs WS2+WS3-cumulative distinction** (interactions §2) rather than quietly retuning the constant.
- CHANGELOG entry added.
- Docstring note explaining chunked aggregation, the `stream_chunk_threshold` semantics, and the var/std carve-out added to the three datashader entry points.
- API critic post-impl review section filled in this plan.
- **Boundary-gap follow-up (2026-07-04 amendment): column-mapped styling parity.** `edge_viz_kwargs` / `node_viz_kwargs` naming metadata columns, plus `to_json`, work on polars-built plots with output matching the pandas twin. The seven pandas-only gather sites (`viz/matplotlib.py:464`, `viz/bokeh.py:582`, `viz/holoviews.py:638`, `viz/plotly.py:730` + `:780` hover customdata, `hiveplot.py:4842` `to_json` kwarg resolution, `:4865` `to_json` `set_index`) are re-expressed positionally through the narwhals boundary (the `_edge_chunk_metadata_values` precedent; internal materialization for plotting consumption is fine, user frames never mutated or coerced). Polars-marked tests per backend + `to_json`, mirroring the WS1 non-RangeIndex equivalence battery's assertions (per-collection styling values / CDS contents / serialized figure incl. customdata / vdim values / `to_json` string equality).
- **Boundary-gap follow-up (2026-07-04 amendment): `add_nodes` through the boundary.** `BaseHivePlot.add_nodes` (`hiveplot.py:621`) concat mirrors `Edges.add_edges` (pandas fast-path preserved; narwhals vertical concat; legible mixed-library `TypeError`). Polars-marked tests: incremental add keeps `nodes.data` polars; mixed-library case raises the legible error.

**Interactions:** see "Cross-workstream performance interactions" §1 (the streaming-vs-single-shot threshold is ONE shared policy consumed by both WS2 and WS3, not set per-workstream; per-chunk `cvs.line` is a small-scale speed regression, so the harness must assert no-regression at small scale), §2 (WS2's memory win is only partly realized until WS3; measure cumulative peak-RSS, not per-workstream deltas, to avoid undervaluing it), and §3 (WS5 later lets the var/std reductions WS2 punts to single-shot stream over a Dask/cuDF frame).

### Workstream 3: Fused build + internal streaming (resident-memory unlock)

**Status:** shipped — commit `7c0c610` on branch `53-scale-hiveplotlib-to-larger-networks` (2026-07-04); fused route lands the resident-memory unlock (large: 5.06 GB two-stage → 1.05 GB fused); post-impl critics + disposition touch-up complete; qa pass (default suite 1444 at 100%, perf lane 39/0)
**Depends on:** Workstream 2 (consumes the per-chunk reduction-aware aggregator); performance-regression harness (gating).
**Files:**
- `src/hiveplotlib/hiveplot.py:1381+` (`construct_curves`; add a fused build-and-rasterize code route alongside the existing two-stage path, not replacing it)
- `src/hiveplotlib/viz/datashader.py` (the fused path feeds per-chunk curves directly into the Workstream 2 per-chunk aggregator)
- `tests/hiveplot_test.py`, `tests/viz/datashader_test.py` (fused-vs-two-stage equivalence tests; resident-memory test)
- `runners/performance/...` (resident-memory measurement on the fused path; coordinate with the harness plan)
- `benchmarks/benchmarks.py` (add the matplotlib-backend render canary `time_` benchmark; 2026-07-03 amendment)
- `CHANGELOG.rst` (entry)
- `docs/source/...` (note the fused path and its resident-memory characteristics)
- `wiki/wiki/concepts/bezier-curves.md` (post-task research-liaison pass; note the fused build avoids persisting all curves)
- `src/hiveplotlib/viz/input_checks.py` + `src/hiveplotlib/viz/{matplotlib,bokeh,plotly,holoviews}.py` and their tests (2026-07-04 touch-up: ids-only warning, item 2 of the disposition amendment)

**Done when:**
- A fused build-and-rasterize path samples each `(g1, g2, tag)` chunk's curves, hands them to the Workstream 2 per-chunk aggregator, and discards them, never persisting all curves on the object (`hive_plot_edges[...][tag]["curves"]` is not materialized for all chunks at once on this path).
- **Non-persistence is datashader-path-only.** The vector backends (matplotlib, bokeh, plotly, holoviews) consume the materialized curve geometry to draw lines, so they require the persisted `hive_plot_edges[...]["curves"]` arrays. The fused path's non-persistence applies only to the datashader rasterization path; the staged `construct_curves` + persist must remain for the vector backends. Non-persistence is never generalized globally. (See "Cross-workstream performance interactions" §5.)
- **Vector-backend ids-only warning (2026-07-04 disposition amendment, item 2):** when a vector-backend edge render (matplotlib, bokeh, plotly, holoviews) encounters at least one edge record holding non-empty `"ids"` but no `"curves"`, exactly one warning fires per render call, naming `construct_curves()` as the fix and datashader as the backend that renders from ids. The warning must never fire on the datashader path (the shared `input_check(objects_to_plot="edges")` is called by `viz/datashader.py:568` too; gate the warning to the vector backends). Plain `warnings.warn(..., stacklevel=3)` per the existing `input_checks.py` convention. `pytest.warns` tests per backend plus no-warn cases (datashader path; fully-persisted render); existing suite swept for newly-raising renders under `filterwarnings = error`. CHANGELOG Changed entry (released 0.28 behavior was a silently edge-less figure).
- The two-stage `construct_curves` → rasterize path remains the default and the equivalence baseline; the fused path is selected via the streaming opt-in (`stream_chunk_threshold`) alone — foreign-frame input (Dask, cuDF) rides the same single shared streaming policy (interaction §1), with no engine-specific selector (Path A; cascaded 2026-07-03). The two-stage path's persisted output is unchanged bit-for-bit; additionally, the datashader entry points render from stored ids regardless of construction state, on both the streamed and single-shot routes (transient default-geometry build, never persisted); vector backends are unchanged. (Sentence superseded by the 2026-07-04 disposition amendment, item 1; cascaded same pass.)
- Per-chunk metadata is retained alongside per-chunk curves so the metadata-coloring trick survives; a test confirms a metadata-colored fused-path plot matches the two-stage plot within tolerance.
- A resident-memory test confirms the fused path's peak resident curve storage stays O(largest_chunk × num_steps × 4 bytes), not O(total_edges × num_steps × 4 bytes), on a synthetic 10M-edge graph whose node count scales in proportion to edge count (Failure mode 2). Same memory-gate policy as WS2 (2026-07-03): always runs on the canonical host (ringtail, 32 GB; serialize after the parallel suite when RAM is contended, never skip); in the auto-dispatch window a skipped memory gate is a halt-back, not a pass.
- **Matrix-level peak-memory assertion (2026-07-03 disposition of WS2's deferred want):** a `HivePlotMatrix`-level peak-resident assertion rides this workstream's resident-memory work iff it reuses the existing stress-scale workload/measurement (no second stress-scale workload); if it would require one, reject with WS2's recorded rationale (identical per-cell streamed code path; the matrix forwarding equality test covers the wiring) and state the rejection in the Implementation log.
- Fused-vs-two-stage equivalence: the fused path produces output matching the two-stage path across count/sum/any/mean reductions (var/std stay single-shot-or-delegate per Workstream 2), judged against the WS2 exactness set (2026-07-03 disposition amendment): bit-exact for count/any/max/min; tight float tolerance (rtol=1e-12; addition-order provenance) for sum/mean.
- **Dedicated default-suite unit tests for the fused path (2026-07-03 testing directive):** the fused build-and-rasterize route carries its own unit tests in `tests/` under `@pytest.mark.datashader`, running in the parallel coverage run and exercising the fused path directly as a unit (route selection, per-chunk handoff to the WS2 aggregator, discard behavior), not merely reached through the fused-vs-two-stage harness gate. Same rationale and structural enforcement as the parallel WS2 bullet: unit-test failures localize which mode broke, and harness-lane tests contribute zero coverage, so the 100% gate can only be satisfied here.
- Harness-gated validation: the performance-regression harness confirms no speed regression on the default two-stage path, records the fused-path resident-memory reduction at 10M+ edges, and passes the equivalence gate. WS3 inherits the same three-gate obligation on the fused path (tiny-floor timing band, large-scenario streamed-vs-single-shot peak-RSS fraction, raster equivalence) that WS2 arms; the gates run against the fused route too (2026-07-02 amendment).
- **Vector-backend render canary lands here (2026-07-03 amendment):** a `time_` ASV benchmark rendering through the **matplotlib backend** (the library's default viz entry point) at the small scale (optionally tiny as well; implementer's call, both run in milliseconds) exists in `benchmarks/benchmarks.py`, is captured in the ASV suite, and shows no regression vs. the pre-WS3 baseline. All four vector backends (matplotlib, bokeh, holoviews, plotly) consume the same persisted-curves contract (float32 arrays, NaN-separated blocks) via the shared two-stage path, so one matplotlib benchmark canaries the whole family. Explicitly do NOT add bokeh/plotly/holoviews benchmarks (env bloat for optional extras, no planned changes to those renderers). Once in the suite it runs at every capture-at-merge thereafter; see the 2026-07-03 render-canary amendment.
- If implementation finds the fused path needs a user-facing opt-in distinct from `stream_chunk_threshold`, that new name routes back to orchestrator `amend-plan` for a naming audit before landing.
- CHANGELOG entry added.
- Docstring note on the fused path's resident-memory characteristics added.
- API critic post-impl review section filled in this plan.

**Interactions:** see "Cross-workstream performance interactions" §1 (the fused path always chunks; its streaming-vs-single-shot threshold is the SAME shared policy as WS2's, not a second per-workstream threshold; harness asserts no-regression at small scale), §2 (this workstream is what finally removes the resident curve cost WS2 leaves behind; cumulative peak-RSS along the chain is the right measure), and §5 (the non-persistence done-when bullet above).

### Workstream 4: Narwhals at the input boundary

**Status:** shipped — commit `3bf0e94` on branch `53-scale-hiveplotlib-to-larger-networks` (2026-07-04); bound re-derived to `>=2.0.0,<3.0.0` (floor verified 56/56 at 2.0.0); polars output-equivalence bit-exact across all backends incl. styling; polars gallery notebook + thumbnail shipped; all critic records filled, both adversary must-fixes + api-critic must-fix addressed; qa must-fixes (test contract, plan records) closed; final suite 1533 at 100%, perf lane 39/0, docs zero-warnings
**Depends on:** performance-regression harness (gating). Independent of Workstreams 1-3 in code; can proceed in parallel, but the chain orders it after the in-RAM scaling work so the BYO-engine on-ramp lands on a stable streaming surface.
**Files:**
- `pyproject.toml` (narwhals as core dependency; the planned `>=1.x,<2` was dead at pin time — re-derived to `>=2.0.0,<3.0.0` at Gate Zero, 2026-07-04)
- `CLAUDE.md` (brief design note on the narwhals abstraction layer)
- `src/hiveplotlib/frames.py` (boundary-helper module — the workstream's architecture; added to this list 2026-07-04, post-impl disposition bookkeeping fix)
- `src/hiveplotlib/node.py` (`NodeCollection.__init__`, `check_unique_ids`, `create_partition_variable`)
- `src/hiveplotlib/edges.py` (`__init__`, `_validate_edge_data`, `add_edges`, `export_edge_array`)
- `src/hiveplotlib/hiveplot.py:962+` (`add_edge_ids` `np.isin`-style filter)
- `src/hiveplotlib/viz/datashader.py:203-204, 393-407` (metadata extraction and node-placements `pd.concat`; narwhals refactor parallel to Workstream 2; note the metadata read now indexes by integer index per Workstream 1)
- `src/hiveplotlib/viz/matplotlib.py:464`, `src/hiveplotlib/viz/bokeh.py:582`, `src/hiveplotlib/viz/holoviews.py:638`, `src/hiveplotlib/viz/plotly.py:730,780`, `src/hiveplotlib/hiveplot.py:4842,4865` (column-mapped kwarg gathers + `to_json`; boundary-gap follow-up, 2026-07-04 amendment)
- `src/hiveplotlib/hiveplot.py:621` (`BaseHivePlot.add_nodes` concat; boundary-gap follow-up, 2026-07-04 amendment)
- `tests/node_test.py`, `tests/edges_test.py`, `tests/hiveplot_test.py` (extend with polars-backend cases for input-boundary methods); `tests/frames_test.py` (boundary-helper units) and `tests/polars_test.py` (the polars twin battery) — the latter two added to this list 2026-07-04, post-impl disposition bookkeeping fix
- `CHANGELOG.rst` (entry naming the polars/narwhals support and the deps decision)
- `docs/source/...` autodoc updates for `NodeCollection`, `Edges` (note narwhals-supported frame acceptance, "returns whatever you passed")
- `wiki/wiki/entities/hiveplotlib.md` (post-task research-liaison pass; "Development Priorities" gains scaling)
- `wiki/wiki/sources/hiveplotlib-python.md` lines 79-84 (post-task; "minimal base deps" note expanded to acknowledge narwhals as a pure-Python dispatcher)

**Done when:**
- Narwhals is declared as a **core dependency** in `pyproject.toml`, pinned `>=2.0.0,<3.0.0` (the planned `>=1.x,<2` was superseded at Gate Zero by checklist item (c)'s re-derivation, 2026-07-04; floor measured, not assumed). A brief design note is added to `CLAUDE.md` explaining the abstraction layer (so future maintainers don't relitigate the core-dep decision).
- Pin-time checklist, run at WS4 start: (a) the pinned narwhals version still publishes a `py3-none-any` wheel and has not grown a default compiled fast-path (the grill Wave 1 load-bearing fact); (b) `make ty` passes against narwhals's typing at the pinned version, including the `IntoDataFrame` annotation on the input boundary; (c) **re-derive the version bound, don't just verify it (2026-07-03):** confirm the planned `>=1.x,<2` range still tracks narwhals's current live major at pin time (the bound was fixed in 2026-05 and narwhals releases fast; pinning a core dep to a dying major creates resolver conflicts on fresh stacks) — move the bound if a new major is live, or record why staying behind is right. A type-check failure in a core dep blocks the whole boundary refactor, so all three checks run before any code moves.
- `NodeCollection(data=polars_df, ...)` and `Edges(data=polars_df, ...)` work end-to-end; `.data` round-trips to polars.
- **Polars output-equivalence gate (Failure mode 4; 2026-07-03):** a hive plot built from polars input produces output equivalent to the pandas-input path on the same data: identical edge selections (`relevant_edges` membership), identical node placements, and an equivalent rasterization within the harness tolerance (mirroring WS5's Dask equivalence bullet), including at least one non-trivial case exercising `add_edge_ids` filtering plus a metadata-column reduction. Frame-type round-trip and running end-to-end do not count as equivalence.
- `NodeCollection(data=pandas_df, ...)` and `Edges(data=pandas_df, ...)` continue to work identically to today; no regression.
- Docstrings on `NodeCollection.data` and `Edges.data` properties explicitly document the round-trip contract: the property returns the frame in the same library the user passed in, the one named exception is numpy-ndarray input to `Edges` (which becomes pandas because no original frame existed), and the library does not coerce frame types. Wording aligns across both classes.
- Round-trip contract is enforced by tests: for each accepted frame library, construct, then assert `type(nc.data)` / `type(edges.data)` matches the input library; assert the numpy-ndarray-to-pandas exception on `Edges`; assert the contract holds across `Edges`'s dict-vs-singleton dispatch (multi-tag dict-of-polars round-trips to dict-of-polars, not dict-of-pandas).
- `create_partition_variable` works with polars input; if `cut` / `qcut` semantics force a pandas fallback inside the narwhals wrapper, the resulting partition column is converted back to the user's original frame type before assignment to `self.data` (a polars `NodeCollection` stays polars after the call). A test verifies the post-call `nc.data` is the same frame type as the pre-call frame.
- `add_edge_ids` works with polars-backed `NodeCollection`.
- `datashade_edges_mpl` and `datashade_nodes_mpl` accept hive plots built from polars input (the metadata extraction path goes through narwhals).
- Test matrix gains polars cases for at least: `NodeCollection.__init__`, `check_unique_ids`, `create_partition_variable`, `Edges.__init__`, `Edges.add_edges`, `HivePlot.add_edge_ids`, `datashade_edges_mpl` metadata-column reductions. Coverage stays 100%.
- CI shape: single `pytest` invocation. A new `@pytest.mark.polars` marker is added following the existing optional-backend marker pattern (`bokeh`, `datashader`, `holoviews`, `plotly`, `networkx`). Polars-parametrized cases use per-case marks via `pytest.param("polars", marks=pytest.mark.polars)`. Marker isolation works correctly: a polars-marked case does not run under `pytest -m datashader`.
- All warnings-as-errors still pass (narwhals does not introduce a deprecation warning at the version pinned).
- Harness-gated validation: the performance-regression harness confirms narwhals pass-through introduces no speed regression on the pandas path (the bottleneck-math claim that the boundary ops are a ~2% slice holds in measurement, not just on paper). "No regression" includes the matplotlib render canary (landed at WS3) staying within its historical band, guarding against a change on the shared curves exit path taxing every vector backend while datashader-specific gates stay green (2026-07-03 amendment).
- CHANGELOG entry.
- Docstring updates on `NodeCollection.__init__`, `Edges.__init__`, the `data` properties.
- API critic post-impl review section filled in this plan.
- Post-impl disposition touch-up (2026-07-04) lands: the seven-item worklist in "Disposition (Workstream 4 post-impl critics)" — seven-site `.columns` routing through `hashable_columns`, datashader pandas fast-paths + degenerate-pandas regression test, labels=None consequence sentence + multi-digit-bounds twin test, `set_axes_order` stringify-on-collapse + int-valued polars test, context-aware `as_eager_frame` wording, `HivePlotMatrix` polars `from_partition` test, direct `add_edge_ids` call, CLAUDE.md marker-list + pyproject-comment folds.

**Interactions:** see "Cross-workstream performance interactions" §4 (this workstream re-expresses WS1's integer-index gather through narwhals; after it lands, measure the pandas path before/after to confirm the abstraction tax is ~0, since a small constant tax could mask a WS1 micro-win).

### Workstream 5: Dask + cuDF/GPU passthrough

**Status:** implemented (uncommitted) — branch `53-scale-hiveplotlib-to-larger-networks`, 2026-07-05; code + tests + local GPU verification complete (default suite 1584 at 100% coverage, perf lane 41/0, `make test-gpu` 14/14). Post-impl critics complete 2026-07-05 (api-critic + adversary), disposed in the 2026-07-05 post-impl disposition amendment. VRAM hard gate: **discharged, per-route figures verified in isolation** (2026-07-05 re-measure) — single-shot 8,155,022 bytes (8.16 MB), streamed 16,315,438 bytes (16.32 MB), each in its own isolated process (fresh per-route RMM scope reproduces the original 16,315,420, so the shared-epoch suspicion is disproven: the streamed 2.0007x is real and raster-buffer-dominated at smoke scale, two ~8.16 MB aggregate buffers alive at combine time vs single-shot's one). These are the publishable per-route numbers. Touch-up worklist + GPU re-measurement complete (2026-07-05 disposition; see Implementation log). Both gallery notebooks + micro-pass shipped; notebook critic records transcribed; qa PASS 2026-07-05 (final suite 1589 at 100%, perf lane 41/0, GPU lane 14/14, docs zero-warnings). **Shipped — commit `a6e1a8d` on branch `53-scale-hiveplotlib-to-larger-networks` (2026-07-05). The auto-dispatch window is complete; the chain halts here per the Wave 4 addendum (WS6, WS7, and chain close are the maintainer's).**
**Depends on:** Workstreams 1, 3, and 4 (the membership-storage structure whose Dask non-materializing equivalent lands here; the fused build's per-chunk streaming; and the narwhals dispatch layer). Performance-regression harness (gating).
**Files:**
- `src/hiveplotlib/hiveplot.py:962+, 1381+` (verify `add_edge_ids` `is_in` and the fused build's chunk iteration work on Dask- and cuDF-backed input)
- `src/hiveplotlib/hiveplot.py:893`/`:958` (`BaseHivePlot.__store_edge_ids`; supply the Dask non-materializing equivalent of the Workstream 1 integer-index storage, so Dask input does not materialize a per-edge array into RAM)
- `src/hiveplotlib/viz/datashader.py` (verify the per-chunk DataFrame fed to `cvs.line()` can be Dask- or cuDF-typed when the input was Dask/cuDF; verify the datashader CUDA path for cuDF)
- `tests/dask_test.py` (new; in-memory partitioned synthetic Dask DataFrame; small enough to live in CI; `@pytest.mark.dask` marker) *(path corrected 2026-07-05: landed flat per the WS2 test-layout precedent, not at the originally planned `tests/integration/dask_passthrough_test.py`)*
- `tests/cudf_test.py` (new; cuDF/GPU smoke test verifying datashader + narwhals version support; gated by the `@pytest.mark.cudf` marker and skipped where no GPU is available) *(path corrected 2026-07-05: landed flat, not at the originally planned `tests/integration/cudf_smoke_test.py`)*
- `Makefile` (new `test-gpu` target running the GPU-gated tests, e.g. `pytest -m cudf`)
- `CONTRIBUTING.md` (local GPU verification cadence for the never-in-CI cuDF path; migrates to a releases-specific doc when one exists)
- `CHANGELOG.rst` (entry)
- `examples/creating_hive_plots_from_dask.ipynb` + `examples/creating_hive_plots_from_cudf.ipynb` and their `docs/source/gallery_examples/index.rst` registrations (per the 2026-06-05 two-notebook amendment, cascaded here 2026-07-03; the Dask page owns partition-size guidance for the per-partition Bezier memory ceiling and the sort-cost / shuffle disposition for `place_nodes_on_axis`; the cuDF page owns the GPU story)

**Done when:**
- A Dask DataFrame flows through narwhals dispatch with **no opt-in parameter**, uniformly with pandas / polars / pyarrow / cuDF (Path A, 2026-06-05; cascaded into this section 2026-07-03). No `use_dask` anywhere: not on the surface, not in tests, not in docs or notebook prose.
- With Dask installed alongside the datashader extra, construction succeeds and a small Dask DataFrame end-to-end test (in-memory, ~2 partitions, synthetic 10k edges) produces an equivalent rasterization to the pandas path.
- **Missing-integration error (reframed per the two 2026-06-05 disclosure amendments + the 2026-07-03 no-new-extra decision):** no `hiveplotlib[dask]` extra is added. The only integration hiveplotlib's Dask path needs beyond the user's ambient Dask is the **datashader extra**, which already ships `dask[dataframe]` (`pyproject.toml:34`); the existing top-of-module guard at `src/hiveplotlib/viz/datashader.py:23-28` (`pip install hiveplotlib[datashader]`) is the `ImportError` that fires, at import of the rasterization module — hiveplotlib authors no frame-detection-time `ImportError`, which the api-critic's Concern 1 showed has no path to fire. For a frame narwhals cannot dispatch, narwhals's own error surfaces, not a hiveplotlib wrapper.
- The narwhals engine-detection predicate replacing the `isinstance(val, (pd.DataFrame, np.ndarray))` guard at `src/hiveplotlib/edges.py:164` (`_validate_edge_data`) is the **single detection point** for all engines, and its failure message for a malformed Dask object (`dask.delayed`, `dask.array`, a bare list) is legible (2026-06-05 dispositions amendment, cascaded 2026-07-03; resolves the parked clearer-error deferred item).
- **Failure-point disclosure:** when a Dask computation blows up in the per-partition materialization / rasterization loop, hiveplotlib reraises with repartition / partition-size guidance pointing at the Dask notebook **and** chains the original exception (`raise ... from err`) (2026-06-05 dispositions amendment, cascaded 2026-07-03).
- CI shape: single `pytest` invocation. A new `@pytest.mark.dask` marker is added following the existing optional-backend marker pattern (`bokeh`, `datashader`, `holoviews`, `plotly`, `networkx`, `polars`). Dask-gated tests carry the marker so they only run under `pytest -m dask` (or the umbrella `make test`). Marker isolation works correctly: a Dask-marked case does not run under `pytest -m datashader`. A `@pytest.mark.cudf` marker is registered the same way for the GPU-gated tests; it is consumed by the `make test-gpu` target below, never by CI.
- **Measurement methodology is defined before the WS5 numbers are trusted.** The harness's single-process `measure_peak_rss` (`wiki/wiki/plans/performance-regression-harness.md`, Workstream B) fully captures WS1–3's in-RAM, numba-threaded work (threads share one address space, so all parallel allocation lands in one process's `ru_maxrss`) but does **not** reach either of WS5's two memory regimes. The two regimes are covered as follows:
  - **Multi-process Dask path — explicit instrument SELECTION at WS5 (2026-07-03 amendment; replaces the earlier "inherited from the harness MR (tier 2), WS5 does not build its own" framing):** WS5 selects and records the memory instrument among three candidates. Deciding fact: Dask's default local scheduler for dataframe work is **threaded** (one process), in which case tier-1 single-process exact RSS (`measure_peak_rss`) suffices and neither tier-2 nor Dask-native diagnostics are needed. Candidates: (a) **tier-1** if WS5 lands on the threaded scheduler; (b) **Dask-native diagnostics** (MemorySampler / per-worker metrics) if on the distributed scheduler — preferred where applicable per the maintainer's ecosystem-standard-tooling principle (2026-07-03 amendment); (c) **tier-2** (the harness's `measure_peak_rss_tree`, now provisional) only where an external whole-tree observer is needed: the multiprocessing scheduler (which has no Dask diagnostics), or external verification of framework self-reporting (which matters most for the future autonomous-loop use, where the metric must resist gaming). A small Dask-input scenario on the performance runner must state which instrument it reports.
  - **cuDF/GPU path (WS5 owns this, hard gate):** GPU work allocates **VRAM, not RSS**, so peak RSS misses it entirely. The canonical metric is **per-process VRAM peak via RMM allocation statistics (`rmm.statistics`; exact)** *(instrument amended 2026-07-05: the originally pinned cupy mempool measured exactly 0 bytes for this workload, since datashader's CUDA aggregates allocate via numba and cuDF via RMM; see the 2026-07-05 amendment)* — `nvidia-smi` reports whole-GPU memory, which is shared and noisy (the canonical host already sits ~5 GB used by unrelated apps) and cannot isolate the workload's peak, so it is a fallback / sanity-check only, never the published figure. This regime is **testable locally and is NOT hardware-blocked**: the canonical host has a usable GPU (NVIDIA RTX 5070, 12 GB). It lands at WS5 for a dependency-and-consumer reason (no cuDF / datashader-CUDA path to measure until WS5; the cupy/RAPIDS stack that both enables the synthetic test and is the metric source arrives with WS5), not because the hardware is missing. As of 2026-06-26 the env had no working GPU-allocation library (`numba.cuda.is_available()` False; cupy and pynvml absent). **No WS5 GPU scaling claim is accepted without a measured per-process VRAM peak on the local GPU via RMM allocation statistics — not stubbed, not assumed.** *(Gate DISCHARGED 2026-07-05: 8.16 MB single-shot / 16.32 MB streamed on the 2k-edge smoke, RTX 5070; instrument wording amended per the 2026-07-05 amendment. Correction in flight 2026-07-05: the per-route split is suspect — both routes were measured in one shared monotone RMM-statistics epoch — so re-measure per route in isolation before publishing anywhere; see the WS5 post-impl disposition amendment, item 1. Re-measured per route in isolation 2026-07-05: single-shot 8,155,022 bytes (8.16 MB) and streamed 16,315,438 bytes (16.32 MB), each in its own process; a fresh per-route RMM scope inside one process reproduces the original streamed figure (16,315,420 bytes), so the pair stands as measured — the 2.0007x is a real property of the streamed route at smoke scale (two raster-sized aggregate buffers, ~8.16 MB each for the ~1428x1428 count grid, alive at combine time, vs single-shot's one), not a shared-epoch artifact. The verified per-route pair is the publishable figure.)* **Environment update (2026-07-03, supersedes the 2026-06-26 no-GPU-library status):** cudf-cu12 26.6.0 requires pandas<3 — incompatible in one env with the library's pandas-3 dev/CI environment (installing it into the main venv downgraded pandas 3.0.3 → 2.3.3; that venv is being rebuilt — never install the cuDF stack into the main venv). A dedicated GPU verification venv exists at `~/venvs/hiveplotlib-gpu` (cupy-cuda12x 14.1.1, cudf-cu12 + dask-cudf-cu12 26.6.0, dask/distributed 2026.1.1, pandas 2.3.3, editable hiveplotlib with the datashader extra). The cupy mempool peak metric was verified end-to-end on the RTX 5070 (2026-07-03), so this gate is environmentally de-risked. *(2026-07-05: that probe validated the mempool mechanism itself; the real workload's allocations bypass the cupy pool entirely, hence the instrument amendment above.)*
  Proportionality: WS5 is the furthest-out workstream, gated behind WS1–4, so this is a "design the measurement before you trust the numbers" requirement, not a demand to build a full multi-process / GPU benchmarking rig now. Canonical records: the performance-regression-harness plan's measurement-scope-limitation note on Workstream B, its tier-2 process-tree helper note, and its refined GPU-VRAM note (all 2026-06-26).
- The `relevant_edges` storage at `BaseHivePlot.__store_edge_ids` (`src/hiveplotlib/hiveplot.py:893`/`:958`) supplies the Dask non-materializing equivalent of the Workstream 1 integer-index storage: when input is Dask, the per-`(axis_pair, tag)` membership is **not** materialized as a per-edge array into RAM (which would silently defeat the out-of-core story for a 1B-edge graph). Implementation chooses between (a) storing the membership as a Dask Series, (b) adding a per-tag boolean column to the edge frame (kept as Dask), or (c) an equivalent structure that avoids materializing a per-edge array into RAM. The pandas/polars case keeps the Workstream 1 in-RAM integer-index storage. The choice between the three structural options is an implementation decision; the requirement is the no-materialization guarantee for Dask input. (This is the second half of the single storage decision designed in Workstream 1; do not redesign the in-RAM structure here.)
- cuDF/GPU passthrough: a cuDF frame flows through narwhals dispatch and the datashader CUDA path (`cvs.line` over cuDF → cupy aggregates) end-to-end with no `use_cudf` parameter. A smoke test (`tests/integration/cudf_smoke_test.py`, GPU-gated, skipped where no GPU) verifies datashader + narwhals version support for the cuDF path against the pinned versions. The Dask/cuDF interplay for the var/std delegation path (Workstream 2 option (i)) is verified here against the pinned datashader version. Any new CUDA-path rendering-mode branch in `src/hiveplotlib` likewise carries dedicated default-suite unit tests for its CPU-exercisable logic (2026-07-03 testing directive; only genuinely GPU-only execution stays behind the GPU-gated smoke test).
- Local verification cadence for the never-in-CI cuDF path: the GPU gate means CI can never run these tests (permanent silent-rot risk, by the same logic that justified core narwhals). A `make test-gpu` Makefile target runs the GPU-gated tests (e.g. `pytest -m cudf`) and **must target the dedicated GPU env at `~/venvs/hiveplotlib-gpu` (or an equivalent env carrying the cuDF stack), never the main pandas-3 venv** (cudf-cu12 requires pandas<3; 2026-07-03). `CONTRIBUTING.md` documents the cadence: run `make test-gpu` locally before any release that bumps datashader or narwhals, or that touches the input boundary or the datashader backend. No releases-specific doc exists yet (verified 2026-06-11); when one does, this guidance migrates there and CONTRIBUTING.md is the home until then.
- The Dask sort cost is addressed in the Dask example notebook (or equivalent doc). `HivePlot.place_nodes_on_axis` sorts nodes by a variable, which in Dask requires a shuffle (the most expensive Dask operation). Implementer chooses between (a) accept the cost (the Dask sort happens, document expected wall-time impact) or (b) document a "sort upstream before passing to hiveplotlib" pattern in the notebook. Either disposition is acceptable; the requirement is that the question is answered visibly to users, not left as a silent footgun. *(Resolved 2026-07-05, WS5 implementation: no-shuffle — no Dask sort exists inside hiveplotlib; each axis's node subset materializes into the internal pandas placements table, so the notebook's visible answer is "nodes are the in-memory side; node data must fit in RAM," not a sort-upstream pattern. See the WS5 post-impl disposition amendment and Implementation log notebook fact 2.)*
- The Dask example notebook (or the Dask passthrough docs) documents partition-size guidance. The Bezier kernel materializes one Dask partition's curves into numpy at a time; the per-partition memory ceiling is roughly `largest_partition_size * num_steps * 4 bytes` (float32). A user with few-but-huge partitions can still exceed RAM, defeating the streaming win. Documentation names a concrete repartition guideline (e.g., "for very large edge sets, repartition to ensure no partition exceeds ~10M edges before passing to hiveplotlib"; the exact recommended ceiling is implementer's call based on benchmarks).
- Documentation lands as the two gallery notebooks per the 2026-06-05 two-notebook amendment (cascaded 2026-07-03): `examples/creating_hive_plots_from_dask.ipynb` (partition sizing, shuffle disposition, failure-point guidance, and the install story `pip install hiveplotlib[datashader]`) and `examples/creating_hive_plots_from_cudf.ipynb`, which **names the cuDF pandas-version constraint (cudf-cu12 requires pandas<3, one env cannot hold both)** rather than implying `pip install` into a pandas-3 env works, and **names the GPU reduction-support set at the pinned stack (count/any/max/min; sum/mean/var/std raise upstream — NVVM kernel-compilation failure; see the 2026-07-05 amendment)**. No `use_dask` in any prose (Path A).
- Published Dask performance numbers (notebook, blog, docs) state the scheduler (threaded / multiprocessing / distributed) and the hardware, and are framed as local-machine results unless verified representative of real distributed hardware (Failure mode 3; 2026-07-03).
- Harness-gated validation: the performance-regression harness confirms no speed regression on the non-Dask paths and records the Dask out-of-core memory behavior on the synthetic partitioned scenario; var/std-equivalence holds for the delegated path. "No regression" includes the matplotlib render canary (landed at WS3) staying within its historical band, covering exactly the case where a Dask-space cleverness earns its speedup by taxing the shared curves exit path every vector backend consumes (2026-07-03 amendment). *(Equivalence note 2026-07-05: cross-kernel "bit-exact"/exact raster comparisons here and in sibling done-whens are raster-level equivalence, contingent on pixel quantization absorbing 1-ulp serial-vs-parallel kernel codegen diffs; see the 2026-07-05 CI-follow-up disposition amendment.)*
- CHANGELOG entry naming the Dask + cuDF passthrough (no new parameter; uniform narwhals dispatch, per Path A).
- API critic post-impl review section filled.

**Interactions:** see "Cross-workstream performance interactions" §3 (this workstream retroactively improves WS2: the var/std reductions WS2 punts to single-shot can stream once datashader gets a Dask/cuDF frame and combines moment aggregates across partitions; the non-materializing storage here extends WS1's integer-index structure rather than redesigning it).

**Dependency:** Workstream 5 requires Workstreams 1, 3, and 4 to land first. Workstream 1 fixes the membership-storage structure whose Dask non-materializing counterpart lands here; Workstream 3's fused build produces the per-chunk streaming the Dask/cuDF path rides on; Workstream 4 routes input dataframes through narwhals so Dask and cuDF flow through validation/filtering without bespoke code paths.

## Cross-workstream performance interactions

These workstreams ship as separate MRs, but their performance characteristics are coupled. The per-MR harness gate catches a workstream that regresses on its own; what it cannot catch is compound behavior, where a workstream's full value is masked until a later workstream lands, or where two individually-neutral changes interact badly. The interactions below are recorded so that when the maintainer turns each workstream into its own issue, the relevant cross-effect surfaces and informs how that workstream is evaluated.

### 1. WS2/WS3 chunking overhead at small scale (the one genuine regression risk)

Per-chunk `cvs.line` is slightly slower than one single-shot call (per-call overhead, less vectorization), so streaming is a speed *regression* below the scale where memory forces it. WS2 introduces the streamed path; WS3's fused build always chunks (it never materializes). The risk: if WS2 and WS3 each set the streaming-vs-single-shot threshold independently, a small-graph regression ships.

Mitigation: the streaming-vs-single-shot threshold must be ONE shared policy decision in one place, consumed by both WS2 and WS3, not set per-workstream. The harness must assert no-regression at small scale after each merge. (Cross-ref WS2, WS3.)

### 2. WS2's memory win is only partly realized without WS3 (masked benefit, not a regression)

WS2 alone removes only the transient datashader concat copy; the resident materialized curves (~808 MB per 1M edges at `num_steps=100` float32) remain until WS3 lands. Benchmarking WS2 in isolation shows a modest memory delta and risks undervaluing it.

Mitigation: measure cumulative peak-RSS along the chain, not just per-workstream deltas. (Cross-ref WS2, WS3.)

### 3. WS5 retroactively improves WS2 (positive interaction)

The var/std reductions that WS2 must punt to single-shot can stream once WS5 feeds datashader a Dask/cuDF frame (datashader combines moment aggregates across partitions). WS5's non-materializing storage also extends WS1's integer-index structure. These compound favorably. (Cross-ref WS2, WS5; storage link WS1 ↔ WS5.)

### 4. WS4 will re-touch WS1's code (rework, not regression)

WS1's integer-index gather, if written with pandas `.iloc`, has no polars equivalent, so narwhals (WS4) must re-express it.

Mitigation: WS1 should not bake pandas-specific indexing assumptions into the gather. After WS4, measure the pandas path before and after to confirm the narwhals abstraction tax is ~0 (the plan estimates ~2% total / near-zero per-op; verify it, since a small constant tax could mask a WS1 micro-win). (Cross-ref WS1, WS4.)

### 5. WS3's "never persist" is datashader-path-only (a backend-architecture constraint)

The vector backends (matplotlib, bokeh, plotly, holoviews) consume the materialized curve geometry to draw lines, so they require the persisted `hive_plot_edges[...]["curves"]` arrays. WS3's non-persistence can apply only to the datashader rasterization path; the staged `construct_curves` + persist must remain for the vector backends. WS3 must never generalize non-persistence globally. (Cross-ref WS3; stated in WS3's done-when.)

### What escapes per-MR gating

The per-MR harness no-regression gate catches any single workstream that regresses on its own. What escapes per-MR gating is compound behavior: masked benefits (interaction 2), and the rare case of two individually-neutral changes interacting badly (interaction 1). That is why the harness needs a cumulative / rolling-baseline benchmark across the chain, not just per-MR deltas. This requirement is being recorded in the parallel `wiki/wiki/plans/performance-regression-harness.md`.

## Plan amendments

### In-scope tweak: Workstream C ships `use_dask: Optional[bool] = None` with default = raise-on-Dask

> **SUPERSEDED 2026-06-05** by "Removed parameter (Workstream 5): `use_dask` dropped entirely (Path A)" below. The maintainer's post-grill final call dropped the `use_dask` parameter; Dask is now detected via narwhals and handled uniformly like every other engine. The entry is preserved as historical record of the planning-mode decision; do not implement the raise-on-default behavior described here.

**Date:** 2026-05-17
**Trigger:** Planning-mode api-critic Q1 (must-fix). Quoted in the "API Critic's take (planning mode)" subsection: "Recommend opt-in via `use_dask: Optional[bool] = None` with the default `None` meaning 'raise if a Dask frame is passed without an explicit opt-in.'"
**Workstream affected:** Workstream C (Optional Dask passthrough), with cascading edits to Default justifications, Naming audit (Workstream 3 subsection), and API usage examples (Example 4).
**Change:**
- Default justifications: the recommended path flips from "no new parameter, infer from frame type" to "`use_dask: Optional[bool] = None` with `None` raising on Dask input." The "no new parameter" path is now the documented alternative considered. Rationale: failure modes are asymmetric (silent OOM/thrash on partition-by-partition materialization is the dangerous mode; opt-in surfaces it loudly at construction), and opt-in is asymmetrically cheaper to relax later than to tighten.
- Naming audit (Workstream 3): `use_dask` is now in scope, not a fallback. `Optional[bool]` with `None` default. Branch semantics for `None` / `True` / `False` × Dask / non-Dask are spelled out.
- Workstream C "Done when": new bullets requiring the parameter, the raise-on-default behavior with pinned error message text, all four `(use_dask × frame-type)` branches tested, the `ImportError` path when Dask is not installed, and the documentation example showing the explicit opt-in.
- API usage examples: Example 4 rewritten to show `use_dask=True` invocation; new Example 4b shows the `TypeError` users see when they forget the opt-in.

### In-scope tweak: `.data` round-trip is documented contract, enforced by tests

**Date:** 2026-05-17
**Trigger:** Planning-mode api-critic Q2 (must-fix). Quoted: "Once `NodeCollection(data=polars_df).data` returns polars (per the plan), users will write `nc.data.filter(pl.col('x') > 0)` and ship it. If a future refactor switches `.data` to 'always pandas because internal storage simplified,' every downstream polars user breaks silently. The right time to commit to a contract is when the surface is born; the right place is the docstring on `.data` and `.__init__`."
**Workstream affected:** Workstream B (Narwhals at the input boundary), with a cascading edit to the Naming audit (Workstream 2 subsection).
**Change:**
- Naming audit (Workstream 2): the round-trip contract docstring sketch is now part of the planned docstring text on `NodeCollection.data` and `Edges.data`, including the one named exception (numpy-ndarray input to `Edges` becomes pandas because no original frame existed). Wording aligns across both classes.
- Workstream B "Done when": new bullet requiring the docstring text on both properties to document the round-trip contract; new bullet requiring tests that enforce the contract for each accepted frame library (pandas in → pandas out, polars in → polars out), assert the documented numpy-to-pandas exception, and assert the contract holds across `Edges`'s dict-vs-singleton dispatch (multi-tag dict-of-polars round-trips to dict-of-polars).

### In-scope tweak: `create_partition_variable` pandas fallback restores frame type before assignment

**Date:** 2026-05-17
**Trigger:** Planning-mode api-critic "Other concerns" (must-fix). Quoted: "If `create_partition_variable` falls back to pandas for `cut`/`qcut`, the result is converted back to the user's original frame type before being assigned to `self.data`."
**Workstream affected:** Workstream B (Narwhals at the input boundary).
**Change:**
- Workstream B "Done when": new bullet requiring that when `create_partition_variable` falls back to pandas for `cut`/`qcut` semantics, the resulting partition column is converted back to the user's original frame type before being assigned to `self.data`. A test verifies that the post-call `nc.data` is the same frame type as the pre-call frame (polars in → polars out across a `create_partition_variable` call). Closes the silent-conversion hole that would otherwise undercut the round-trip contract above.

### Deferred follow-up: `frame_library` property on both `NodeCollection` and `Edges`

**Date:** 2026-05-17 (initial); updated 2026-05-17 (maintainer follow-up: symmetric mirroring to `Edges`)
**Trigger:** Planning-mode api-critic Q3 (worth-discussing). Quoted: "A `NodeCollection.frame_library` property returning a string like `'pandas'` / `'polars'` / `'dask'` is more durable than `type(...)` and cheaper to document. Not blocking; a quality-of-life add that should be considered alongside Workstream B." Subsequent maintainer note: the round-trip contract applies symmetrically to both `NodeCollection` and `Edges`, so the property should mirror to both classes; implementing only on `NodeCollection` would leave an asymmetric ergonomic gap.
**Target:** Maintainer call. If cheap to add during Workstream B execution, the code-engineer implements a `frame_library` property on **both** `NodeCollection` **and** `Edges`; otherwise defer to a follow-up plan. Not promoted to a done-when bullet because the round-trip contract above already gives users the durable behavior the property names; the property is documentation-as-API for ergonomic convenience rather than a load-bearing piece of the surface.
**Rationale:** Quality-of-life on top of the round-trip contract; not part of the load-bearing surface. Symmetric on both classes because the round-trip contract is symmetric. Surfaced here so the code-engineer sees it before starting Workstream B and can fold it in opportunistically across both classes in one pass (or leave it for a future plan with explicit done-when bullets).

### Deferred follow-up: `to_pandas()` convenience method on `NodeCollection` and `Edges`

**Date:** 2026-05-17
**Trigger:** Planning-mode api-critic Q3 (worth-discussing). Quoted: "A `to_pandas()` method on `NodeCollection` and `Edges` would help users who built a `HivePlot` from polars but want to drop into pandas for a one-off `.describe()` or for an integration with a pandas-only downstream tool. Cheap to add (`nw.from_native(self.data).to_pandas()`), and signals 'you can always escape to pandas if you need to' which is a comforting safety property for the narwhals adoption."
**Target:** Maintainer call. If cheap during Workstream B, code-engineer adds it; otherwise defer. Same disposition as `frame_library` above.
**Rationale:** Adds a one-line escape hatch, doesn't change any load-bearing surface, but does grow the public method count by one per class. Worth a maintainer eyeball before promotion to done-when, since taste on "library should not grow methods that wrap existing one-liners" varies.

### Deferred follow-up: multi-tag polars dict test coverage explicitly enumerated in Workstream B

**Date:** 2026-05-17
**Trigger:** Planning-mode api-critic "Other concerns" (should-fix, test coverage). Quoted: "The `.data` getter on `Edges` already does dict-vs-singleton dispatch. Worth verifying the round-trip contract holds across the multi-tag dict case too: a user passing `data={'tag_a': polars_df_a, 'tag_b': polars_df_b}` should get back a dict-of-polars, not a dict-of-pandas. Add a multi-tag polars test case to Workstream B's test matrix."
**Target:** Already partially covered by the round-trip contract enforcement bullet in the in-scope tweak above (which explicitly names the dict-vs-singleton dispatch case). Surfaced here as a follow-up flag so the test-engineer treats the multi-tag polars dict case as a named scenario, not a hidden assumption.
**Rationale:** The done-when bullet now names this case, so it is in-scope rather than deferred; this entry exists so the trigger and provenance are visible alongside the other amendments.

### Deferred follow-up: clearer error for partially set-up Dask objects (`dask.delayed`, `dask.array`)

**Date:** 2026-05-17 (initial); updated 2026-05-17 (maintainer disposition: parked indefinitely)
**Trigger:** Planning-mode api-critic "Other concerns" (should-fix). Quoted: "The plan does not name what happens if a user passes a `dask.delayed` or a Dask `Bag` or a `dask.array`. Recommend: narwhals's `from_native` will raise on non-DataFrame Dask objects; surface that error with a clarifying message ('hiveplotlib accepts `dask.dataframe.DataFrame`; got `dask.delayed.Delayed`. Materialize first with `.compute()` or wrap in `dd.from_delayed(...)`.')."
**Target:** **Parked indefinitely (post-Workstream C reconsideration; likely notebook documentation rather than code change).** Revisit after Workstream C ships. Maintainer disposition: the underlying narwhals raise already prevents silent misuse, and any user-facing improvement is likely better documented in the eventual Dask example notebook than added as a custom error path in code.
**Rationale:** The `TypeError` raise from narwhals's `from_native` already prevents silent misuse, so safety is in place. Adding a custom error-message wrapper inside code grows the surface for negligible ergonomic gain; the same guidance ("hiveplotlib accepts `dask.dataframe.DataFrame`; materialize first with `.compute()` or wrap with `dd.from_delayed(...)`") sits more naturally in the Dask example notebook's prose, where it can be expanded with worked examples. Revisit only if post-Workstream C usage shows users hitting this path frequently and the notebook is insufficient.

### Design clarification: `frame_library` property mirrors to both `NodeCollection` and `Edges`

**Date:** 2026-05-17
**Trigger:** Maintainer follow-up on the existing "Deferred follow-up: `frame_library` property" entry above. Paraphrased from the conversation: "The deferred entry currently reads as if `frame_library` applies only to `NodeCollection`; it should mirror to `Edges` as well, since the round-trip contract applies symmetrically to both classes."
**Workstream affected:** Workstream B (Narwhals at the input boundary). Edits the existing deferred follow-up entry above to make the symmetry explicit.
**Change:**
- The deferred follow-up entry's title and body now read "a `frame_library` property on both `NodeCollection` and `Edges`," with a note on the symmetric round-trip contract motivating the symmetric API.
- No done-when change; the entry remains a deferred follow-up rather than a load-bearing requirement. The clarification ensures that if the code-engineer picks it up opportunistically during Workstream B, they implement both at once rather than leaving an asymmetric ergonomic gap.

### Deferred follow-up update: `dask.delayed` / `dask.array` clearer-error item parked indefinitely

**Date:** 2026-05-17
**Trigger:** Maintainer disposition on the existing "Deferred follow-up: clearer error for partially set-up Dask objects" entry above. Paraphrased from the conversation: "Slide the dask.delayed / dask.array clearer-error item from 'maintainer call, cheap fold-in' to 'parked indefinitely; revisit after Workstream C ships.' Rationale: the underlying narwhals raise already prevents silent misuse, and any user-facing improvement is likely better documented in the eventual Dask example notebook than added as a custom error path in code."
**Workstream affected:** Workstream C (Optional Dask passthrough). Edits the existing deferred follow-up entry above to update its status.
**Change:**
- The deferred follow-up entry's target status now reads "parked indefinitely (post-Workstream C reconsideration; likely notebook documentation rather than code change)" rather than the prior "maintainer call, cheap fold-in" framing.
- Rationale spelled out in the entry: safety is already in place via the narwhals `from_native` raise; the value-add is message polish, which sits more naturally in the Dask example notebook's prose than in a code-level error wrapper. Revisit only if post-Workstream C usage shows users hitting this path frequently and the notebook is insufficient.

### In-scope tweak: narwhals is a core dependency, pinned `>=1.x,<2`

**Date:** 2026-05-17
**Trigger:** Maintainer follow-up resolution of the open question Workstream B left for the api-critic + maintainer review ("Narwhals as core dep vs. extra"). Maintainer ask, paraphrased from the follow-up conversation: "Narwhals as a core dependency, pinned `>=1.x,<2`; the optional-extra path is rejected."
**Workstream affected:** Workstream B (Narwhals at the input boundary), with cascading edits to Default justifications and Workstream B Files.
**Change:**
- Default justifications: new "narwhals as a core dependency (not an extra)" section captures the rationale (narwhals is pure-Python with no compiled extensions, no platform-specific binaries, no threading layer; the risk profile that justifies keeping numba optional does not apply; optional narwhals would force CI matrix doubling or silent rot; and the wiki's documented "minimal base deps" stance is a deliberate deviation that the post-task research-liaison pass should reflect into `wiki/wiki/sources/hiveplotlib-python.md:79-84`).
- Workstream B Files: `pyproject.toml` entry rewritten as "add narwhals as core dependency, pinned `>=1.x,<2`"; new `CLAUDE.md` entry for the brief design note on the abstraction layer (kept so future maintainers do not relitigate the decision).
- Workstream B "Done when": the bullet that previously deferred the decision now reads "Narwhals is declared as a **core dependency** in `pyproject.toml`, pinned `>=1.x,<2`. A brief design note is added to `CLAUDE.md` explaining the abstraction layer."

### Design clarification: narwhals usage pattern is pass-through, not opinionated-internal

**Date:** 2026-05-17
**Trigger:** Maintainer follow-up clarification. Paraphrased from the conversation: "Make the narwhals usage pattern explicit: pass-through, not opinionated-internal. When a user passes polars, dispatch to polars; when they pass pandas, dispatch to pandas; when they pass Dask, dispatch to Dask. We do not convert to a canonical internal frame type."
**Workstream affected:** Workstream B (Narwhals at the input boundary), with a cascading edit to Default justifications. No example change required.
**Change:**
- Default justifications: new "narwhals usage pattern is pass-through, not opinionated-internal" section captures the rationale. Bottleneck math: source-dataframe operations sum to ~10-50 ms on a 1M-edge graph in pandas vs ~2-10 ms in polars; total HivePlot construction is 1-3 seconds dominated by Bezier curves; polars-internal saves roughly 2% of total wall time, not enough to justify the architectural complexity. "Always-polars internally" also breaks the Dask and cuDF/GPU stories (converting Dask to polars defeats out-of-core; converting cuDF to polars pulls data off the GPU); special-casing those backends is the complexity cost without the bottleneck win. Pass-through respects where the user's data lives.
- API usage examples: Examples 2 ("polars in, polars out") and 3 ("pandas in, pandas out") already reflect pass-through behavior accurately; confirmation note added inside the new Default justifications subsection. No rewording needed.

### Design clarification (Workstream C): `relevant_edges` storage must be backend-aware

**Date:** 2026-05-17
**Trigger:** Maintainer follow-up design note. Paraphrased from the conversation: "The `relevant_edges` storage at `__store_edge_ids` (`dict[a1][a2][tag] = numpy bool ndarray`) needs to be backend-aware. For Dask, materializing a 1B-bool numpy array per (axis_pair, tag) silently defeats the out-of-core story; the Workstream C implementation must avoid that materialization."
**Workstream affected:** Workstream C (Optional Dask passthrough), with cascading edits to Patterns this replaces (Workstream 3 subsection) and Workstream C Files.
**Change:**
- Patterns this replaces (Workstream 3): `BaseHivePlot.__store_edge_ids` at `src/hiveplotlib/hiveplot.py:580-647` added to the replace-list. Today writes `edges.relevant_edges[a1][a2][tag] = indices_to_store` as a numpy bool ndarray; Workstream C replaces this with a backend-aware structure when input is Dask.
- Workstream C Files: `src/hiveplotlib/hiveplot.py:580-647` (`__store_edge_ids`) added.
- Workstream C "Done when": new bullet requiring the storage to not materialize a per-edge bool ndarray into RAM when input is Dask. Implementation chooses between (a) Dask Series storage, (b) a per-tag boolean column on the edge frame (kept as Dask), or (c) an equivalent structure that avoids the materialization. Pandas case keeps the existing numpy ndarray storage.
- The structural choice between the three options is a Workstream C implementation decision; the requirement is the no-materialization guarantee.

### Design clarification (Workstream C): Dask sort cost (`place_nodes_on_axis`) must be addressed in docs

**Date:** 2026-05-17
**Trigger:** Maintainer follow-up design note. Paraphrased from the conversation: "`HivePlot.place_nodes_on_axis` sorts nodes on an axis by a variable. In Dask, sort requires a shuffle, which is the most expensive Dask operation. Surface this as a Workstream C design note with two possible dispositions for the implementer to choose between."
**Workstream affected:** Workstream C (Optional Dask passthrough), with cascading edit to Workstream C Files (notebook scope).
**Change:**
- Workstream C "Done when": new bullet requiring the Dask example notebook (or equivalent doc) to address the sort-cost question. Implementer chooses between (a) accept the cost (the Dask sort happens, document expected wall-time impact) or (b) document a "sort upstream before passing to hiveplotlib" pattern.
- Either disposition is acceptable; the requirement is that the question is answered visibly to users, not left as a silent footgun.

### Design clarification (Workstream C / docs note): per-partition Bezier memory ceiling and partition sizing

**Date:** 2026-05-17
**Trigger:** Maintainer follow-up design note. Paraphrased from the conversation: "The Bezier kernel materializes one Dask partition's curves into numpy at a time. The memory ceiling for the curves array is `largest_partition_size * num_steps * 4 bytes` (float32). If a user has a Dask frame with few-but-huge partitions, the per-partition curves array can still exceed RAM, defeating the streaming win."
**Workstream affected:** Workstream C (Optional Dask passthrough), with cascading edit to Workstream C Files (notebook scope) and Workstream C "Done when".
**Change:**
- Workstream C "Done when": new bullet requiring the Dask example notebook (or the Dask passthrough docs) to document partition-size guidance. Names a concrete repartition guideline (e.g., "for very large edge sets, repartition to ensure no partition exceeds ~10M edges before passing to hiveplotlib"; the exact recommended ceiling is implementer's call based on benchmarks).
- Together with the sort-cost disposition above, the Dask example notebook now owns two concrete user-facing guidance pieces (sort and partition sizing) so the "Dask passthrough is opt-in and not silent footguns" story is visible to readers.

### In-scope tweak (bookkeeping): CI shape is single pytest invocation with new polars and dask markers

**Date:** 2026-05-17
**Trigger:** Maintainer follow-up bookkeeping. Paraphrased: "Confirm the CI shape decision is recorded somewhere in the plan: single `pytest` invocation, with new `@pytest.mark.polars` and `@pytest.mark.dask` markers following the existing optional-backend marker pattern. Polars-parametrized cases use per-case marks via `pytest.param('polars', marks=pytest.mark.polars)`. Marker isolation works correctly (a polars-marked case does not run under `pytest -m datashader`)."
**Workstream affected:** Workstream B (polars marker), Workstream C (dask marker).
**Change:**
- Workstream B "Done when": new bullet recording the CI shape decision (single `pytest` invocation, new `@pytest.mark.polars` marker following the existing optional-backend pattern: `bokeh`, `datashader`, `holoviews`, `plotly`, `networkx`). Polars-parametrized cases use `pytest.param("polars", marks=pytest.mark.polars)`, marker isolation verified.
- Workstream C "Done when": parallel bullet for `@pytest.mark.dask`.

### Restructure: 5-workstream dependency chain replaces the A/B/C split

**Date:** 2026-05-29
**Trigger:** Maintainer discussion 2026-05-29. The plan is kept as ONE super-plan (maintainer-approved) but its workstreams are reordered and expanded into a clean dependency chain, with implementation strictly separate per workstream (each its own MR, each gated by the separate performance-regression harness plan being created in parallel at `wiki/wiki/plans/performance-regression-harness.md`). All entries below carry this same trigger.
**Workstreams affected:** all. The new chain is: 1 Membership storage redesign → 2 Reduction-aware chunked rasterization (was A) → 3 Fused build + internal streaming (new) → 4 Narwhals at input boundary (was B) → 5 Dask + cuDF/GPU passthrough (was C). The original A/B/C done-when content is preserved and forwarded under the new numbering; the changes below are additive or corrective.
**Change:**
- Goal reframed around the **two distinct meanings of "out-of-core"**: internal chunking of an in-RAM-but-huge frame (Workstreams 1-3, no narwhals) vs. bring-your-own foreign frame engine (Workstreams 4-5, needs narwhals). The load-bearing correction: narwhals is the on-ramp for BYO-engine + GPU, **not** a prerequisite for in-RAM-but-huge scaling. Added the transient-vs-resident memory distinction and the scope guards (Bézier Bernstein hoist and numba autotuning are out; autonomous loop deferred).
- ADR strategy expanded from three numbered decisions to five (sparse membership storage, reduction algebra, fused build, narwhals-as-on-ramp, cuDF/GPU).
- Prior ADRs / design docs: added `wiki/wiki/concepts/bezier-curves.md` (the previously-missing citation for the kernel that dominates compute and resident memory; documents the numba ~4,096-point serial/parallel switch and float32 storage) and `wiki/wiki/sources/nollenburg-2023-computing-hive-plots.md` (NP-completeness framing).
- Every workstream's done-when gains a harness-gated validation bullet (speed + no-regression on the default path; memory reduction recorded on the scaled path; var/std-equivalence wherever rasterization changes).

### Added workstream: Workstream 1 — Membership storage redesign (sparse integer indices)

**Date:** 2026-05-29
**Trigger:** Maintainer discussion 2026-05-29. Folds the maintainer's "#4 sparse" idea together with the existing Workstream C Dask-storage requirement — they are ONE data-structure decision, designed once.
**Workstream affected:** new Workstream 1; pulls the `__store_edge_ids` storage decision out of the old Workstream C.
**Change:**
- New Workstream 1 replaces the full-length boolean mask at `BaseHivePlot.__store_edge_ids` (`src/hiveplotlib/hiveplot.py:797`) with integer indices of the selected edges. Edges partition across axis-pairs (each edge belongs to exactly one ordered pair), so the bool storage is O(num_pairs × num_edges) mostly-False bytes; integer indices are O(num_edges) total, strictly better in-RAM whenever multiple axis-pairs exist (always). This is the in-RAM half of the single storage decision; the Dask non-materializing equivalent stays in Workstream 5 and is explicitly designed to extend this structure, not redesign it.
- Downstream consumer correction: the metadata indexing at `src/hiveplotlib/viz/datashader.py:203-204` (`_data[tag][mask]`, boolean-mask indexing) becomes `.iloc[idx]` / positional take. Added to Patterns this replaces (Workstream 1) and to the Workstream 1 Files / done-when.
- Replace-and-sweep audit reorganized: the `__store_edge_ids` site moved from the old Workstream 3 subsection into the new Workstream 1 subsection; the Dask non-materializing equivalent remains under Workstream 5.

### In-scope correction (Workstream 2): reduction-aware aggregation replaces "additive raster aggregation"

**Date:** 2026-05-29
**Trigger:** Maintainer discussion 2026-05-29. The old Workstream A said "additive raster aggregation," which is **wrong** for `ds.var` / `ds.std`. The gallery `examples/datashading_statistical_summaries_of_metadata.ipynb` uses `ds.var` twice and `ds.mean` once; naive per-chunk raster summation would silently regress those figures.
**Workstream affected:** Workstream 2 (was Workstream A).
**Change:**
- Aggregation is now **reduction-aware**: count/sum/any stream additively; mean accumulates per-chunk sum + count and divides once (reusing the existing density-correction divide at `datashader.py:242-247`); var/std and any non-additive/exotic reduction never sum per-chunk rasters and instead either delegate partial-aggregate combination to datashader over a Dask/cuDF frame (verify against the pinned datashader version) or fall back to single-shot.
- The single-shot path stays the **default** for the common case and doubles as the equivalence/regression baseline. New done-when bullet: a **var/std-equivalence gate** verified against the gallery notebook's `ds.var`/`ds.mean` usage.
- Memory-bound test updated to float32 (`× 4 bytes`, matching the kernel's actual dtype) rather than the old `× 8 bytes`.

### Added workstream: Workstream 2 parameter — `stream_chunk_threshold` (with naming audit)

**Date:** 2026-05-29
**Trigger:** Maintainer discussion 2026-05-29. The reduction-aware rasterization needs a user-facing control: single-shot default for the common case, streamed path as the opt-in/escape hatch for scale.
**Workstream affected:** Workstream 2 (was Workstream A, previously a pure internal refactor with no public surface).
**Change:**
- New parameter `stream_chunk_threshold: Optional[int] = None` on the three datashader entry points. `None` → auto-decide by edge-set size; an explicit `int` forces the switch point; var/std always single-shot-or-delegate regardless. Naming audit added (Workstream 2 subsection): rejected `chunked=True`, `use_streaming`, `max_edges_in_memory`; `stream_chunk_threshold` keeps the size-trigger meaning legible against datashader/dask vocabulary.
- Workstream 2 is now an API-surface-changing workstream: its post-impl api-critic placeholder is added (previously "no post-impl critic needed for Workstream 1 / the streaming refactor").
- API usage Example 1 rewritten to show the default auto-decide, the force-stream escape hatch, and the var/std carve-out note.
- Default justifications gains a "Workstream 2 default: streaming-vs-single-shot threshold" subsection.

### Added workstream: Workstream 3 — Fused build + internal streaming (resident-memory unlock)

**Date:** 2026-05-29
**Trigger:** Maintainer discussion 2026-05-29. The old Workstream A (chunked rasterization, viz-side) only removes the **transient** concat copy (`datashader.py:224-231`). It does NOT remove the **resident** cost: `construct_curves` materializes and persists every curve on the object (`hive_plot_edges[...]["curves"]`, ~808 MB per 1M edges at num_steps=100 float32, ~8 GB at 10M, impossible at 1B). The real in-RAM out-of-core unlock needs a fused build-and-rasterize path.
**Workstream affected:** new Workstream 3; depends on Workstream 2's per-chunk aggregator.
**Change:**
- New Workstream 3: a fused path samples each chunk's curves, rasterizes into the running aggregate, and discards them, never persisting all curves. Constraint: the fused path retains each chunk's metadata column so the metadata-coloring trick survives. The two-stage `construct_curves` path remains the default and the equivalence baseline; the fused path is selected via the streaming opt-in.
- Needs no narwhals (this is the in-RAM/internal out-of-core case, decision (a) case 1 from the Goal). Resident-vs-transient memory distinction added to the Goal.
- If the fused path needs its own opt-in distinct from `stream_chunk_threshold`, that routes back to `amend-plan` for a naming audit before landing.

### In-scope expansion (Workstream 5): cuDF/GPU passthrough rides on narwhals + datashader CUDA

**Date:** 2026-05-29
**Trigger:** Maintainer discussion 2026-05-29. cuDF/GPU support is a small marginal addition once Dask passthrough works: narwhals dispatches to cuDF, datashader's CUDA path (`cvs.line` over cuDF / dask-cudf → cupy aggregates) does the GPU rasterization. Maintainer framing: support fat-GPU users "without going deep on cuDF" — narwhals does the dispatch.
**Workstream affected:** Workstream 5 (was Workstream C, "Optional Dask passthrough"), retitled "Dask + cuDF/GPU passthrough".
**Change:**
- No `use_cudf` parameter: cuDF rides on narwhals dispatch and carries no partition-by-partition materialization risk, so it needs no opt-in gate (audited in the Workstream 5 naming subsection). The `use_dask` gate is specific to Dask's materialization risk.
- New done-when bullet plus a new test file (`tests/integration/cudf_smoke_test.py`, GPU-gated) verifying datashader + narwhals version support for the cuDF path against the pinned versions. The var/std delegation path (Workstream 2 option (i)) is verified here against the pinned datashader version.
- API usage Example 5 (cuDF/GPU) added. Dependency updated: Workstream 5 now requires Workstreams 1, 3, and 4 (membership-storage structure, fused build, narwhals dispatch).

### Bookkeeping: stale source-line references corrected to the working tree

**Date:** 2026-05-29
**Trigger:** Maintainer discussion 2026-05-29 / restructure pass. The original plan cited `__store_edge_ids` at `hiveplot.py:580-647` and `add_edge_ids` at `:707-749`; the working tree has `__store_edge_ids` membership storage at `hiveplot.py:797` and `add_edge_ids` at `:801+`. The datashader metadata read is at `:203-204` (boolean-mask indexing) with the density-correction divide at `:242-247`.
**Workstreams affected:** Workstreams 1, 2, 4, 5 (Files, Patterns this replaces).
**Change:** Replaced the stale `580-647` and `707-749` references with `:797` and `:801+` respectively, and pinned the datashader metadata read to `:203-204` and the density-correction divide to `:242-247`, throughout Patterns this replaces, Default justifications, and the affected workstream Files lists.

### Design clarification: cross-workstream performance interactions recorded as a standalone section

**Date:** 2026-05-29
**Trigger:** Maintainer discussion on cross-workstream regression risk. The five workstreams ship as separate MRs, but their performance characteristics are coupled in ways the per-MR harness gate cannot catch (masked benefits and two-neutral-changes-interacting-badly are compound effects, not per-MR deltas). The maintainer wants each interaction surfaced so that when a workstream becomes its own issue, the relevant cross-effect informs how it is evaluated.
**Workstreams affected:** all (analysis is cross-cutting; no done-when content changes except the one WS3 bullet noted below).
**Change:**
- New top-level section "Cross-workstream performance interactions" (placed before Plan amendments) captures five interactions: (1) WS2/WS3 chunking overhead at small scale, the one genuine regression risk, mitigated by a single shared streaming-vs-single-shot threshold consumed by both plus a per-MR small-scale no-regression assertion; (2) WS2's memory win is only partly realized without WS3, a masked benefit, mitigated by measuring cumulative peak-RSS along the chain; (3) WS5 retroactively improves WS2 (var/std can stream over Dask/cuDF) and its storage extends WS1's, a positive interaction; (4) WS4 re-touches WS1's gather (rework, not regression), mitigated by not baking pandas `.iloc` assumptions into WS1 and verifying the ~0 narwhals tax after WS4; (5) WS3's non-persistence is datashader-path-only, a backend-architecture constraint (vector backends need the persisted curves). A closing note explains why the harness needs a cumulative / rolling-baseline benchmark, recorded in the parallel `wiki/wiki/plans/performance-regression-harness.md`.
- A one-line "Interactions" cross-reference added under each affected workstream's done-when (WS1 → §3, §4; WS2 → §1, §2, §3; WS3 → §1, §2, §5; WS4 → §4; WS5 → §3).
- WS3 gains one new done-when bullet stating the datashader-path-only non-persistence constraint (interaction §5), since the constraint is load-bearing for that workstream rather than purely advisory.

### Maintainer shared-understanding pass (grill), Wave 1 — premise, scope, core-dep

**Date:** 2026-06-05
**Trigger:** Maintainer-led Socratic pass over the plan's load-bearing premises, to make the implicit reasoning explicit and surface any divergence between the plan-as-written and the maintainer's actual intent. No done-when changes from this wave except the open item flagged below.

- **Premise (why now).** There is no explicit external demand signal (no user complaint, no dated deadline). This is intentional forward-looking investment, accepted as such. Two motivations: (1) unblock the maintainer's own GNN-heterophily work so it can try genuinely larger networks instead of being scale-constrained, and (2) the field lacks good visualizations of very large networks, and this is an opportunity to build them. Adjacent in-progress repos near hiveplotlib carry the initial GNN work. Consequence for evaluation: speed/latency wins are secondary; the load-bearing payoff is "can render at all at a scale the library currently cannot," so success is measured against unblocking scale, not shaving wall-time on small graphs.
- **Scope ambition.** Intent is to ship all five workstreams as one aggressive forward-looking pass, accepting more up-front investment to avoid coming back later. Timing is flexible (depends how long it takes), but 4-5 (BYO-engine, Dask, GPU) are committed, not "only if demand materializes." The single-umbrella-ADR strategy stands.
- **narwhals as a core dependency (cold defense).** Endorsed. Named regret conditions: (1) deep long-term API instability — acceptable and cheaply mitigated via `>=,<` version bounds and eventual removal if narwhals is ever absorbed/obsoleted; judged infrequent enough not to matter (the maintainer pins almost nothing in `pyproject.toml` by preference and is fine with a narrow bound here). (2) The disqualifying condition would be narwhals shipping compiled / platform-specific wheels (Rust/C binaries) the way numba-class deps do — that would reintroduce the per-platform wheel-install fragility the "matplotlib/numpy/pandas just work everywhere" stance protects against. **Load-bearing fact the endorsement rests on:** narwhals is pure-Python, `py3-none-any` wheel, zero required dependencies. Pin-time check for the implementing workstream (WS4): confirm the pinned narwhals version still publishes a `none-any` wheel and has not grown a default compiled fast-path.
- **OPEN — `use_dask` default (WS5), maintainer instinct diverges from the shipped api-critic decision.** The plan currently ships loud-failure (`TypeError` on Dask input without `use_dask=True`, 2026-05-17 amendment, api-critic Q1). Maintainer's first-pass instinct is the opposite: if a user passes a Dask frame they presumably want Dask, so just use it; a warning would be an acceptable middle, a hard `TypeError` feels heavier than warranted. Grill counter-point raised: the strongest argument for the strict default is **reversibility, not OOM-safety** (strict→lenient is a non-breaking loosening; lenient→strict is a breaking deprecation), which also makes warn-instead-of-error the weakest of the three options (preserves neither optionality nor clean UX). **Decision pending maintainer's final call; WS5 not changed until then.** If the call flips to lenient, this cascades to WS5 done-when (drop the raise-on-default and the pinned-message test), the Naming audit (WS5), Default justifications (`use_dask`), and API usage Examples 4/4b.

### Maintainer shared-understanding pass (grill), Wave 2 — reduction algebra, memory model, threshold

**Date:** 2026-06-05
**Trigger:** Continuation of the grill, dropping to the correctness/memory claims the chain rests on. No done-when changes; this wave confirms maintainer buy-in and records two clarifications and one process expectation.

- **`use_dask` vs `stream_chunk_threshold` are orthogonal axes (clarification that resolves a conflation).** Two independent decisions were being read as one: (axis 1) which engine the user's input frame lives in (narwhals' job; `use_dask` is the Dask-specific acknowledgment of the lazy partition-materialization footgun the in-RAM/on-GPU engines don't carry) vs. (axis 2) whether hiveplotlib streams its own curve geometry internally (`stream_chunk_threshold`, engine-independent, applies to pandas just as much as Dask). They compose; neither implies the other. Consequence for the open WS5 item above: the maintainer's "I want backend chunking regardless of input type" concern is served entirely by `stream_chunk_threshold` and is **not** an independent argument for the `use_dask` gate. The case for the gate remains reversibility alone (strict→lenient is non-breaking; lenient→strict needs a deprecation).
- **var/std reduction algebra — confirmed, with the "recovered later" nuance.** Maintainer confirms the core point (non-composable / moment-based reductions cannot sum per-chunk rasters and must run in one place). Recorded nuance so it is not misremembered as "var/std can never stream": the design is single-shot **or delegate**. datashader itself combines moment-based partial aggregates across partitions correctly (parallel-variance), so once WS5 feeds it a Dask/cuDF frame the var/std case streams after all. This is exactly the "WS5 retroactively improves WS2" positive interaction (§3); the maintainer need not hold the algorithm, only the "punt now, recovered in WS5" shape.
- **Resident-vs-transient memory — maintainer explicitly goes on trust + the harness, which is endorsed as the correct stance.** Maintainer does not pre-believe specific magnitudes and is relying on the performance-regression harness to demonstrate the win (and on "strictly better, or near-free weakly-better" as the bar). The one failure mode of that stance, recorded as a guard: WS2 measured **in isolation** looks underwhelming (it removes only the transient concat copy; the resident curve cost remains until WS3's fused build). The plan already defends this via interaction §2 (cumulative rolling-baseline measurement, not per-workstream deltas). Operating rule for the maintainer: **judge WS2 and WS3 cumulatively, never WS2 alone.**
- **Process expectation (maintainer ask): surface the shared-threshold check in review, and be explicit about perf-checking cadence.** At WS2 and WS3 code/perf review time, the dispatching session is to surface explicitly that both consume ONE streaming-vs-single-shot threshold policy (interaction §1), and to state in each pass whether performance is being validated then or deliberately deferred. Keeps the no-silent-small-graph-regression guarantee from living only in plan text; makes the perf-validation cadence a visible conversation rather than an assumption.

### Maintainer shared-understanding pass (grill), Wave 3 — harness keystone, scope, ADR, and deepened `use_dask`

**Date:** 2026-06-05
**Trigger:** Final grill wave: the harness anti-reward-hack guard, harness proportionality, deferral/ADR confirms, and a deepened `use_dask` analysis after the maintainer pushed for a real mechanism rather than the reversibility argument.

- **`use_dask` deepened — auto-safety is not achievable, but the residual risk is narrower than the plan's framing; decision now leans lenient+warn (still OPEN, maintainer's final call).** Why an auto-threshold (the maintainer's proposed default) cannot work for Dask the way it does for `stream_chunk_threshold`: the streaming threshold can be auto because hiveplotlib *owns* the in-RAM data and knows the edge count exactly; for Dask, hiveplotlib does *not* own execution, and partition sizes can't be read without triggering a compute, so "auto" would mean an expensive probe-compute or a wrong guess that OOMs anyway. The memory ceiling is a property of the user's partition structure, which the library cannot safely auto-fix without overriding the whole reason for using Dask. Honest counter-weight (narrows the risk): once WS3 streams a materialized partition's *geometry*, the large `num_steps × 4 bytes` multiplier is tamed within a partition; the only un-handleable case is a partition whose *raw* edge rows don't fit in RAM (pathological user partitioning), plus the unavoidable shuffle cost of `place_nodes_on_axis` (a time footgun, documented regardless). Strongest argument for the maintainer's lenient lean: **surface consistency** — pandas/polars/pyarrow/cuDF all "just work" with no opt-in, so a mandatory `use_dask=True` singles out Dask for a residual risk the Dask user already owns. Three live options: (1) strict `TypeError`, (2) lenient just-use-it, (3) lenient + one-time construction warning naming the partition-sizing and shuffle footguns. Recommendation moved to **(3)**: with the maintainer having explicitly down-weighted reversibility (accepting reversal cost as a normal API risk), the warning is no longer dominated; it keeps the surface consistent and still discharges the one honesty obligation. Implementation wrinkle for (3): `filterwarnings = error` means hiveplotlib's own Dask tests must expect/filter the warning; users unaffected. **Pending maintainer's final call.** If (2) or (3): cascade to WS5 done-when (drop the raise-on-default + pinned-message test; for (3) add the warning + its test), Naming audit (WS5), Default justifications (`use_dask`), API usage Examples 4/4b.
- **Harness anti-reward-hack guard — confirmed sufficient for supervised work.** The concrete reward-hack the gate defends against: an agent told "faster / less RAM" finding that the cheapest win is to not do the work (skip edges, empty raster, persist nothing). The **equivalence gate** is the defense (claimed speedup must still match the single-shot path's rasterized output within tolerance; deleting edges fails it); the identity-pass + perturbation-fail pair is the meta-check proving the checker is not a rubber stamp. Maintainer and agent converged: sufficient for the supervised, human-reviewed work actually planned. Stronger adversarial checks (fuzzed inputs, non-degeneracy assertions, multiple perturbations) are wanted only *before* unattended autonomous operation, which is deferred — so no gap for this plan.
- **Harness proportionality — all of it wanted; success bar is feasibility-first.** Maintainer wants the full harness (peak RSS, speed, no-regression, unit suite green, deps sane). Success is measured **feasibility-at-scale first, speed second**: "at least feasible to run larger even if it can't run faster" is a win; strictly-better or near-free weakly-better is the hope. Reinforces the Wave 1 premise (the payoff is rendering at a scale the library currently cannot, not shaving small-graph wall-time).
- **NEW SCOPE — speedup blog-post notebook in `docs/` (route to orchestrator `amend-plan`).** Maintainer wants a blog-style notebook documenting the speedups, mirroring the existing Numba-speedup blog notebook under `docs/` (the blog notebooks under `docs/` are the sanctioned exception to the examples-only rule). Two consequences: (a) it justifies polishing clean speedup viz (the viz now feeds a published artifact, not only the gate); it need not show every marginal speedup, though doing so is a bonus. (b) it is a genuine emergent scope addition (a new docs deliverable), so per the harness rule it should route through orchestrator `amend-plan` to be slotted properly (its own small docs workstream, or a done-when on the final workstream) rather than absorbed silently. Recorded here; not designed in this grill.
- **Deferral + ADR confirms.** (a) The autonomous-research while-loop stays deferred: the near-term work is large structural changes where gains are coarse rather than loop-mineable; the maintainer will spot-run a loop occasionally but not wire it as standing automation soon. (b) ADR promotion on close is a **single umbrella ADR** with the five numbered decisions, unless a mid-stream milestone is release-worthy enough to pause and cut early. Parallels the graph-metrics ADR being promoted across its several plans/branches.

### Removed parameter (Workstream 5): `use_dask` dropped entirely (Path A)

**Date:** 2026-06-05
**Trigger:** Maintainer final call after the grill (`## Alignment (grill)`, Wave 3 left `use_dask` OPEN leaning option (3) lenient+warn; the maintainer resolved to **Path A** post-grill). Supersedes the 2026-05-17 "Workstream C ships `use_dask: Optional[bool] = None` with default = raise-on-Dask" amendment (marked superseded above) and overrides Wave 3's option-(3) lean.
**Workstream affected:** Workstream 5, with cascading edits to Naming audit (WS5), Default justifications, and API usage Examples 4 / 4b.
**Decision:** Drop `use_dask` entirely. Detect Dask via narwhals and handle it uniformly like every other engine (pandas / polars / pyarrow / cuDF). No opt-in gate, no acknowledgment flag, no warning.
**Rationale (from the grill):** A flag whose only valid value is `True` is ceremony. Auto-safety is unachievable for Dask anyway (partition sizes aren't knowable without triggering a compute), but WS3's per-partition geometry streaming narrows the residual risk to pathological partition sizing, which the Dask user already owns. Singling out Dask with a mandatory flag is inconsistent with every other engine "just working" (surface-consistency, the strongest lean from Wave 3). The partition-sizing and shuffle-cost guidance already lives in the WS5 Dask-notebook done-whens and is the honest disclosure mechanism instead of a gate.
**Change:**
- **WS5 "Done when":** drop the bullet requiring `use_dask: Optional[bool] = None`; drop the raise-on-`None` `TypeError` bullet and its pinned-message test; drop the four `(use_dask × frame-type)` branch-test bullet. **Keep**, reframed without `use_dask`: passing a Dask frame when Dask is not installed raises `ImportError` naming the missing extra (`pip install hiveplotlib[dask]`). New shape: a Dask frame flows through narwhals dispatch with no opt-in; the small in-memory ~2-partition synthetic end-to-end equivalence test, the `@pytest.mark.dask` marker bullet, the non-materializing `__store_edge_ids` storage bullet, the sort-cost and partition-sizing notebook bullets, and the harness-gated validation all stand unchanged.
- **Naming audit (WS5):** remove `use_dask` (and its `Optional[bool]` / branch-semantics paragraph). Record that Dask takes **no** opt-in parameter, matching cuDF: both ride on narwhals dispatch; neither warrants a gate. The `backend=` and `lazy=` rejections become moot (no parameter is added at all) and can be noted as such.
- **Default justifications:** remove the "Workstream 5 default: `use_dask`" subsection (both the recommended-path paragraph and the "alternative considered, rejected: infer from frame type" paragraph; inference-from-frame-type is now the shipped behavior, not a rejected alternative). Update the "Workstream 5 (Dask + cuDF/GPU)" line under the per-workstream defaults list to read that WS5 introduces **no** new user-facing parameter (Dask and cuDF both flow through narwhals dispatch with no opt-in).
- **API usage examples:** rewrite Example 4 to plain Dask passthrough (`NodeCollection(data=node_ddf, ...)` / `Edges(data=edge_ddf, ...)` with no `use_dask`), mirroring Example 5's cuDF passthrough shape; keep a one-line pointer to the notebook's partition-sizing / shuffle guidance. Delete Example 4b (the forgot-the-flag `TypeError` demo) entirely.
- **Documentation:** the WS5 Dask-notebook done-whens already own partition-sizing and shuffle-cost guidance; they are the disclosure mechanism that replaces the dropped gate. No new docs scope from this removal beyond ensuring the notebook prose does not reference `use_dask`.
**api-critic re-review:** the grill asked api-critic to pressure-test "should there be ANY parameter here." See the dispatch recommendation in the amend-plan report; a short planning-mode api-critic pass on the simplified (zero-parameter) WS5 surface is recommended before WS5 lands.

### Added workstream (Workstream 6): speedup blog-post notebook in `docs/source/blog/`

**Date:** 2026-06-05
**Trigger:** Grill Wave 3 NEW SCOPE bullet, routed to `amend-plan`: "Maintainer wants a blog-style notebook documenting the speedups, mirroring the existing Numba-speedup blog notebook under `docs/`." Genuine emergent docs deliverable, not absorbed silently.
**Triage:** Added workstream. Shaped as its own small docs workstream (Workstream 6) rather than a done-when on WS5, because it depends on the *measured* speedup / feasibility work from Workstreams 2, 3, and 5 all existing (it documents their results), so it sequences last; folding it into WS5's done-when would block WS5 closure on a cross-workstream narrative artifact. It is a `docs/`-blog deliverable (the sanctioned exception to the examples-only rule), so it does not touch `examples/`.
**Pattern confirmed:** the existing Numba-speedup blog notebook lives at `docs/source/blog/v0.27.0_speedups.ipynb` (sibling blog notebooks: `docs/source/blog/v0.26.0_release.ipynb`, `docs/source/blog/migrating_to_new_hiveplot_class.ipynb`). The new notebook mirrors this location and genre.

**Status:** not started
**Depends on:** Workstreams 2, 3, and 5 (it documents their measured speedups / feasibility-at-scale results). Sequences last in the chain. Performance-regression harness (gating, as the source of the numbers it reports). Figures trace to the per-workstream gate records in this plan, the post-merge `make benchmark-capture` before/after against the pre-chain baseline, and the real-data ogbn-products runs (2026-07-05 log entries): a feasibility-first before/after story, not per-workstream marginal (2026-07-06 execution-practice update; the optional per-boundary ASV series is bonus material, not a dependency).
**Files:**
- `docs/source/blog/<version>_scaling_speedups.ipynb` (new; exact filename/version slug chosen at authoring time to match the release it ships with, following the `v0.27.0_speedups.ipynb` convention)
- the blog toctree / index entry wherever the existing blog notebooks are registered (mirror how `v0.27.0_speedups.ipynb` is wired into the docs build)
- `CHANGELOG.rst` (entry for the new blog post if the existing blog-notebook precedent changelogs blog posts; match precedent)

**Done when:**
- A blog-style notebook exists under `docs/source/blog/`, mirroring the structure and genre of `docs/source/blog/v0.27.0_speedups.ipynb` (the existing Numba-speedup blog post), and is wired into the docs build the same way (toctree / index entry).
- It documents the scaling work's payoff. Per the grill, the success bar is **feasibility-at-scale first, speed second**: the notebook leads with "can now render at a scale the library previously could not," and reports speedups where they exist. It need not show every marginal speedup (doing so is a bonus, not a requirement).
- The speedup / feasibility figures it presents are clean, published-quality viz (the viz now feeds a published artifact, not only the perf gate; this justifies the viz polish called for in the viz-quality-bar skill).
- Failure-mode riders (2026-07-03): any at-scale render shown is readable at the published raster resolution (Failure mode 1), and any Dask/parallelism numbers state scheduler + hardware and are framed local-only unless verified representative of distributed hardware (Failure mode 3).
- The notebook executes end-to-end under `make docs` (and `make test-nb` if blog notebooks are in that sweep; confirm against the existing blog notebooks' coverage) with no warnings (warnings-as-errors).
- Prose follows the writing-voice rules (no em-dashes, no AI filler, direct voice, length discipline).
- The numbers it reports are sourced from the performance-regression harness runs (not ad-hoc one-off timings), so the published claims trace to the gated benchmark.
- CHANGELOG entry if the existing blog-notebook precedent changelogs blog posts.

**Interactions:** depends on the *results* of Workstreams 2, 3, and 5; it is a reporting artifact, so it carries no new source-code or API surface and no cross-workstream performance coupling of its own. It does carry a viz-critic post-impl review (it produces published figures) and an editorial-critic post-impl review (it adds a notebook); no api-critic review (no API surface).

### In-scope tweak (Workstream 5): re-land the disclosure the dropped `use_dask` gate used to carry

**Date:** 2026-06-05
**Trigger:** Planning-mode api-critic re-review of the now-zero-parameter WS5 surface (recorded above as "API Critic — planning-mode re-review (post-`use_dask` removal)", Concerns 1-4). The critic endorsed Path A but found the removed gate carried disclosure that has to re-land at WS5's real failure points. All four concerns are worth-discussing; folded in here.
**Workstream affected:** Workstream 5 (Dask + cuDF/GPU passthrough). Done-when edits only; no new entry point, no new attribute read (no feasibility audit needed). Builds on "Removed parameter (Workstream 5): `use_dask` dropped entirely (Path A)" above.
**Change:**
- **Fix the missing-Dask `ImportError` trigger (Concern 1).** The WS5 done-when bullet that today reads "passing a Dask frame when Dask is not installed raises `ImportError`" is near-contradictory (you cannot hold a `dask.dataframe.DataFrame` without Dask installed). Reframe it to name what is *actually* absent: the additional integration hiveplotlib's Dask path needs beyond the user's ambient Dask, i.e. a `hiveplotlib[dask]` extra (and/or the datashader-Dask version support the rasterization path requires). Follow the existing missing-extra convention (`src/hiveplotlib/converters.py:15-17`, `src/hiveplotlib/viz/datashader.py:23-28`): a top-of-module `try/import/except ImportError` naming `pip install hiveplotlib[dask]`. **That convention fires at module import, not at frame-detection time**, so the WS5 done-when must specify the real trigger: either (a) hiveplotlib gates its Dask-dispatch / datashader-Dask integration behind the extra and the `ImportError` fires at import of that gated module, or (b) narwhals raises its own "cannot dispatch to uninstalled backend" error and hiveplotlib lets narwhals own the message rather than promising a hiveplotlib-authored extra-naming `ImportError` with no path to fire. The done-when names which of (a)/(b) holds and pins the trigger to it, not to "a Dask frame was passed."
- **Add a done-when for failure-point disclosure (Concern 2).** With no construction-time gate, nothing now tells a user what they see when a Dask computation blows up deep in the rasterization loop (post-shuffle, a numpy/datashader traceback far from the `Edges(...)` call). New WS5 done-when: the per-partition materialization / rasterization failure point catches-and-reraises (or at minimum documents) the partition-size failure with a message pointing back to the notebook's repartition / partition-size guidance, so the blow-up site is self-describing rather than a bare un-attributable MemoryError. This is the replacement for the acknowledgment the gate used to provide: disclosure at the failure point instead of a gate at construction.
- **Example 4 vs Example 5 distinctness (Concern 3).** Examples 4 (Dask) and 5 (cuDF) are uniform in *shape* (no parameter) but not in *what the user must know*: Dask alone carries the per-partition curve-materialization ceiling **and** the `place_nodes_on_axis` shuffle cost; cuDF carries neither. Example 4's inline comment must name **both** Dask footguns in one line each, not merely point at the notebook. The cuDF symmetry is a trap; do not make Example 4 a near-verbatim twin of Example 5. (Supersedes, for Example 4's inline comment, the "keep a one-line pointer to the notebook's partition-sizing / shuffle guidance" wording in the Path A amendment above: keep the notebook pointer, but the inline comment also names the two footguns concretely.)
- **Record Concern 4 as an implementation-time note (low-confidence).** At WS4/WS5 implementation, confirm the narwhals engine-detection predicate that replaces the `isinstance(val, (pd.DataFrame, np.ndarray))` guard at `src/hiveplotlib/edges.py:164` (in `_validate_edge_data`) is the **single** detection point for all engines (so Dask is not special-cased back in), and that its failure message for a genuinely-unsupported object (`dask.delayed`, `dask.array`, a bare list) is legible. With `use_dask` gone, the parked "clearer error for partially set-up Dask objects" amendment relies entirely on this narwhals raise being reached (not pre-empted by a stale `isinstance` assert); this is a one-line confirmation, not a new done-when.

### In-scope tweak (Workstream 5): maintainer dispositions on the three open A1 disclosure items

**Date:** 2026-06-05
**Trigger:** Maintainer dispositions on the three open items from "In-scope tweak (Workstream 5): re-land the disclosure the dropped `use_dask` gate used to carry" above (Concerns 1, 2, 4). Sharpens that entry's done-when edits; does not rewrite it. No new entry point, no new attribute read beyond the already-planned `_validate_edge_data` detection point (feasibility audit covered there); none needed.
**Workstream affected:** Workstream 5 (Dask + cuDF/GPU passthrough). Done-when sharpening plus one packaging add (`[dask]` extra).
**Change:**
- **`[dask]` extra confirmed, paired with datashader (resolves Concern 1's open (a)/(b) choice toward (a)).** Add a `hiveplotlib[dask]` install extra in `pyproject.toml`; this is the real target of the reframed missing-extra `ImportError`. Design intent to honor: the Dask path is meaningful only with the datashader backend (a 10M-edge hive plot in a vector backend is nonsensical), so Dask-without-datashader must not be presented as a sensible configuration. Implementer's call on the exact dependency shape, either `[dask]` depends on datashader, or the docs pair them explicitly; the constraint is that the two travel together. Done-when: the gated Dask-dispatch / datashader-Dask module raises the extra-naming `ImportError` (`pip install hiveplotlib[dask]`) at import per the existing convention (`src/hiveplotlib/viz/datashader.py:23-28`).
- **Failure-point reraise must chain the original exception (sharpens Concern 2).** The failure-point disclosure done-when is tightened from "catches-and-reraises (or at minimum documents)" to a hard requirement: when a Dask computation blows up in the materialization / rasterization loop, hiveplotlib reraises with its repartition / partition-size guidance **and** chains the underlying exception (`raise ... from err`), so the user sees both the "here's what to run" guidance and the real traceback. Pass on the actual failure; do not swallow it.
- **Promote Concern 4 from low-confidence verify-note to a built done-when (resolves the parked Dask-object item).** The narwhals engine-detection predicate **is built** as the single detection point for all engines, replacing the `isinstance(val, (pd.DataFrame, np.ndarray))` guard at `src/hiveplotlib/edges.py:164` in `_validate_edge_data`. Its failure message must cover the malformed-Dask-object case (`dask.delayed` / `dask.array` passed instead of a `dask.dataframe.DataFrame`). This is now a load-bearing done-when, not a one-line confirmation.
- **Deferred item resolved-by this done-when.** The above gives the long-parked "Deferred follow-up: clearer error for partially set-up Dask objects (`dask.delayed`, `dask.array`)" entry (and its "parked indefinitely" update) a concrete home, the single detection point's failure message, rather than notebook prose. Mark that deferred item **resolved-by** this WS5 done-when.

### In-scope tweak (Workstream 6): blog notebook speaks in reader terms, no plan scaffolding

**Date:** 2026-06-05
**Trigger:** Maintainer reader-facing framing constraint for the shipped blog artifact, reinforcing the existing harness rule "plan-internal scaffolding banned from shipped artifacts" for this specific deliverable.
**Workstream affected:** Workstream 6 (speedup blog-post notebook). One new done-when; no source or API surface (no feasibility audit needed).
**Change:**
- New WS6 done-when: the blog notebook speaks **entirely in reader terms** (what got faster or newly feasible and why it matters to someone plotting large networks) and must **NOT** use workstream names, plan-internal labels, phase numbers, or any plan scaffolding. The bar: "10M-edge plots that used to be impossible now render," not "Workstream 3's fused build removed the resident curves."

### Added workstream (Workstream 4): "Hive Plots from polars" gallery notebook

**Date:** 2026-06-05
**Trigger:** Maintainer decision on per-engine documentation: each in-memory narwhals engine does **not** get its own notebook; polars gets one reference page (it lands with the narwhals boundary), and the other in-memory engines are covered by a one-liner in it.
**Triage:** Added workstream (a new documentation deliverable on WS4), not an in-scope tweak: it is a new gallery notebook with its own done-when, editorial-critic, and viz-critic gates, slotting into the existing docs gallery section "Hive Plots from Different Data Sources" alongside `creating_hive_plots_from_pandas.ipynb` and `creating_hive_plots_from_networkx.ipynb`. Homed in WS4 because polars support lands with the narwhals input boundary. No new entry point, no new attribute read (the constructors already accept polars after WS4) — no feasibility audit needed. This is distinct from the Workstream 6 speedup blog (which remains the cross-cutting performance narrative); this is a single-data-source reference page.
**Notebook-coherence audit:** genre gallery; class — the `NodeCollection` / `Edges` / `HivePlot` construction surface (matches its data-source siblings, which are about the input boundary, not one class); dataset — leans on `hiveplotlib.datasets.example_*` per the gallery skill, single source; section "Hive Plots from Different Data Sources." Single-data-source reference page, no class drift, no second dataset. Clean.
**Change (WS4 Files):**
- `examples/creating_hive_plots_from_polars.ipynb` (new gallery notebook; follows the `hiveplotlib-gallery-notebook` skill).
- `docs/source/gallery_examples/index.rst` — register in the "Hive Plots from Different Data Sources" section, in both the `nblinkgallery` block and the hidden `toctree`, same order (matches the existing two siblings around lines 127-142).
**Change (WS4 "Done when"):**
- A "Hive Plots from polars" gallery notebook exists at `examples/creating_hive_plots_from_polars.ipynb`, registered in both blocks of the gallery index's "Hive Plots from Different Data Sources" section. It is a short, focused reference demonstration of constructing a hive plot from a polars frame, showing the "polars in, polars out" round-trip; it leans on `hiveplotlib.datasets.example_*` rather than building data from scratch and follows the gallery skill (H1 noun-phrase title, instructional voice, writing-voice rules).
- The notebook carries a single line noting that the other in-memory narwhals engines (pandas, pyarrow, modin) behave identically (the frame returns in the same library), which is why they do **not** each get their own notebook (per the gallery skill's no-page-for-a-one-off rule).
- The notebook executes end-to-end (`make test-nb` / local kernel) with no warnings (warnings-as-errors).

### In-scope tweak (Workstream 5): refine the Dask + cuDF doc deliverable into two parallel gallery notebooks

**Date:** 2026-06-05
**Trigger:** Maintainer decision on per-engine documentation, paired with the polars-notebook decision above: the WS5 doc deliverable splits into two named gallery reference pages, one per engine with a distinct capability story (Dask out-of-core; cuDF/GPU).
**Triage:** In-scope tweak refining WS5's existing documentation done-when (the prior "one new example or notebook section demonstrating Dask passthrough ... plus a cuDF/GPU note" / `examples/scaling_with_dask.ipynb`) into two named gallery notebooks. WS5 already owns the partition-sizing, shuffle-cost, and failure-point disclosure done-whens; this tweak relocates the *home* of that teaching from a terse inline plan example into a dedicated Dask gallery page and adds a parallel cuDF page. No new entry point, no new attribute read — no feasibility audit needed. Both pages slot into "Hive Plots from Different Data Sources." Distinct from the Workstream 6 speedup blog (cross-cutting performance narrative); these are single-data-source reference pages.
**Notebook-coherence audit (both pages):** genre gallery; class — the construction / input-boundary surface (matches the data-source siblings); dataset — each leans on `hiveplotlib.datasets.example_*` (single source per page, with a small partitioned synthetic only where Dask requires it to show partitions); section "Hive Plots from Different Data Sources." Each is a single-data-source reference page, no class drift. Clean.
**Scope guard (maintainer decision, recorded):** stop at three engine notebooks — polars (above), Dask, cuDF. Do **not** add dedicated notebooks for pyarrow or modin (no distinct capability story; pyarrow behaves like polars, modin overlaps Dask; both covered by the one-liner in the polars notebook, per the gallery skill's no-page-for-a-one-off rule). The lazy / SQL narwhals backends (DuckDB, PySpark, Ibis) are **out of scope** for this plan (their lazy / query-engine materialization semantics are not something hiveplotlib addresses here); recorded below as a speculative future follow-up, not part of this plan.
**Change (WS5 Files):** replace the single `docs/source/...` / `examples/scaling_with_dask.ipynb` notebook line with two gallery notebooks plus their index registrations:
- `examples/creating_hive_plots_from_dask.ipynb` (new gallery notebook; the home for the Dask nuances).
- `examples/creating_hive_plots_from_cudf.ipynb` (new gallery notebook; the home for the cuDF/GPU nuances).
- `docs/source/gallery_examples/index.rst` — register both in the "Hive Plots from Different Data Sources" section, in both the `nblinkgallery` block and the hidden `toctree`, same order.
**Change (WS5 "Done when"):** the existing WS5 Dask-notebook done-whens (sort-cost / `place_nodes_on_axis` shuffle disposition; per-partition Bezier memory ceiling and partition-size guidance; the failure-point reraise-with-guidance disclosure from the "re-land the disclosure" amendment) now land specifically in `examples/creating_hive_plots_from_dask.ipynb`. The plan's inline API-usage Example 4 stays minimal and correct (per the Path A amendment, naming the two Dask footguns in one line each); the teaching lives in the Dask notebook. New / refined bullets:
- A "Hive Plots from Dask" gallery notebook exists at `examples/creating_hive_plots_from_dask.ipynb`, registered in both index blocks. It is the home for the Dask nuances: what the partition parameters mean for scaling on your own machine versus a cluster tool like Coiled; the per-partition curve-materialization ceiling; the `place_nodes_on_axis` shuffle cost; and the failure-point guidance (what to run when a Dask computation blows up). Follows the gallery skill; executes end-to-end with no warnings.
- A "Hive Plots from cuDF" gallery notebook exists at `examples/creating_hive_plots_from_cudf.ipynb`, registered in both index blocks. It is the home for the GPU-efficiency nuances of accelerating hive plot construction on a cuDF/GPU frame. Follows the gallery skill; executes end-to-end with no warnings (GPU-gated where it must actually run on a GPU; otherwise the prose / non-executing cells follow the existing GPU-doc precedent).
- The scope guard above holds: no pyarrow or modin notebook; lazy/SQL backends out of scope.

### Deferred follow-up: lazy / SQL narwhals backends (DuckDB, PySpark, Ibis)

**Date:** 2026-06-05
**Trigger:** Scope guard recorded with the WS5 two-notebook refinement above.
**Target:** Speculative future follow-up, **not part of this plan**. The lazy / query-engine narwhals backends (DuckDB, PySpark, Ibis) carry materialization semantics (lazy evaluation, query-engine execution) that hiveplotlib does not address in this scaling plan. Revisit only if there is a concrete demand signal and a clear story for how their materialization model maps onto hiveplotlib's per-partition geometry streaming. Recorded so the boundary of "three engine notebooks (polars, Dask, cuDF)" is explicit and the omission is a decision, not an oversight.

### Added workstream (Workstream 7): performance decision table in the large-networks tutorial

**Date:** 2026-06-10
**Trigger:** Maintainer-approved docs deliverable from a docs-gaps discussion (2026-06-10). A standalone "performance guide" docs page was considered and **rejected**; the agreed shape is a decision table added to the existing tutorial `examples/hive_plots_for_large_networks.ipynb`. Deliberately deferred to the end of the scaling program because Workstreams 2, 3, and 5 are about to rewrite the performance story; numbers published earlier would go stale immediately.
**Triage:** Added workstream (new documentation deliverable with its own done-when and editorial-critic gate), not an in-scope tweak and not a WS6 done-when: it edits a tutorial notebook in `examples/` while WS6 ships a blog notebook in `docs/source/blog/`; different artifact, different genre, different review surface. Folding it into WS6 would block the blog's closure on a separate notebook edit. Sequences last, parallel to WS6 (no dependency between 6 and 7). No new entry point, no new attribute read — no feasibility audit needed.
**Notebook-coherence audit:** `examples/hive_plots_for_large_networks.ipynb` — genre tutorial; class `HivePlot` plus the viz backends (the page's existing subject is "how to visualize a large hive plot, matplotlib's limits, pivot to datashader"); dataset Wikipedia squirrel pages (Rozemberczki, Allen, and Sarkar 2021), single source. A backend decision table is squarely on this page's existing subject; no genre drift, no class drift, no added dataset (the table reports measured numbers as static markdown, it does not load new data). Clean; no sign-off flag.

**Status:** not started
**Depends on:** Workstreams 2, 3, and 5 (the table's thresholds describe the post-streaming, post-fused-build, post-passthrough performance reality); the performance-regression harness, specifically ASV `time_` / `peakmem_` benchmarks (which own timing history/statistics per the harness's 2026-06-11 ASV amendment) plus the harness's `measure_peak_rss` / `measure_peak_rss_tree` helpers, as the source of every published number.
**Files:**
- `examples/hive_plots_for_large_networks.ipynb` (add a backend decision-table section; the only notebook edited — `docs/source/notebooks/` is auto-generated)
- `runners/performance/...` (the sweep scenarios that produce the table's timings; coordinate with the harness plan rather than inventing ad-hoc timing code)
- `CHANGELOG.rst` (entry if precedent changelogs notebook updates; match precedent)

**Done when:**
- The tutorial gains a decision table giving rough graph-size thresholds (node/edge counts) for backend choice: where matplotlib becomes slow, where the interactive backends (bokeh/plotly) choke the browser, and where datashader becomes the right answer.
- The table includes a `num_steps` row: reducing the Bézier interpolation points per curve is a first-order lever at large scale (curve memory and build time scale linearly in `num_steps`, and at datashader scale fewer points are visually indistinguishable after rasterization). Documented user guidance only — `num_steps` reduction is not equivalence-preserving (it changes output geometry), so it stays out of the Workstream 1-3 streaming paths.
- Every timing in the table is an honest measured number produced via the harness's WS-B measurement primitives (median-of-N, fixed seeds) on named sweep scenarios — no guessed or remembered numbers.
- The table is static markdown with its provenance stated in adjacent prose (machine spec, hiveplotlib version, measurement method); the benchmarks are **not** executed live inside the notebook, so `make test-nb` execution time is unaffected.
- The section speaks in reader terms (which backend to reach for at what scale and why); no workstream names, plan labels, or other plan scaffolding (same bar as the WS6 reader-terms done-when).
- Prose follows the writing-voice rules (no em-dashes, no AI filler, direct voice, length discipline) and the table's length stays proportional to the page (a table plus a short paragraph, not a new essay).
- The notebook executes end-to-end (`make test-nb`) with no warnings (warnings-as-errors).
- Editorial-critic post-impl review filled (notebook restructure). No api-critic review (no API surface); no viz-critic review unless implementation adds a figure alongside the table (a figure add routes back through `amend-plan` only if it changes the section's scope; otherwise just invoke viz-critic).
- CHANGELOG entry if precedent applies.

**Interactions:** a reporting artifact like WS6; no source-code or API surface, no performance coupling of its own. Distinct from WS6: the blog notebook is the cross-cutting release narrative ("what got faster"), this table is durable user guidance ("which backend at which size") living where users already look for large-network help. The two should cross-link rather than duplicate numbers.

### In-scope correction (Workstream 2): `any` combines by elementwise OR, max/min classified explicitly, spread/divide once at end

**Date:** 2026-06-11
**Trigger:** Maintainer review session 2026-06-11 (must-fix). The plan said "count / sum / any aggregate additively per chunk" in WS2's done-when and in Patterns this replaces. `ds.any` is **not** additive: summing boolean `any` rasters produces counts.
**Workstream affected:** Workstream 2, with cascading edits to Patterns this replaces (Workstream 2) and Default justifications (streaming-threshold subsection). No new entry point or attribute read; no feasibility audit needed.
**Change:**
- Correct combine for `any` is elementwise OR (or max). WS2 done-when, Patterns this replaces, and the Default justifications algebra list all corrected. The var/std-equivalence gate wording ("exact match for count/sum/any") stands: OR-combine remains exact.
- max/min classified explicitly: cheaply combinable via elementwise max/min. The implementer may support them in the streamed path or leave them in the single-shot-or-delegate fallback bucket, but the classification must be a stated decision recorded in the Implementation log, not an accident of the "exotic" catch-all. New WS2 done-when bullet.
- Order of operations pinned: the streamed path combines **raw per-chunk aggregates**, then applies `tf.spread` and the density-correction divide exactly once at the end (the current single-shot code at `src/hiveplotlib/viz/datashader.py:236-247` spreads and divides after aggregation). Per-chunk spreading before combination would drift from the single-shot baseline and show up as near-tolerance equivalence failures. New WS2 done-when bullet.
- The 2026-05-29 "reduction-aware aggregation" amendment and grill Wave 2 text repeating "count/sum/any stream additively" stand as historical record (append-only); this entry supersedes them on the `any` algebra.

### In-scope tweak (Workstream 5): `make test-gpu` local verification cadence for the GPU-gated cuDF path

**Date:** 2026-06-11
**Trigger:** Maintainer review session 2026-06-11. The cuDF path is GPU-gated and never runs in CI — a permanent silent-rot risk, by the same logic the plan uses to justify core narwhals. No new entry point or attribute read; no feasibility audit needed.
**Workstream affected:** Workstream 5 (Files and done-when).
**Change:**
- New `make test-gpu` Makefile target running the GPU-gated tests (e.g. `pytest -m cudf`). The implied `@pytest.mark.cudf` marker is registered alongside the existing optional-backend markers and named in the WS5 CI-shape bullet next to the `dask` marker.
- `CONTRIBUTING.md` documents the cadence: run `make test-gpu` locally before any release that bumps datashader or narwhals, or that touches the input boundary or the datashader backend. Verified 2026-06-11: no releases-specific doc exists yet; the maintainer plans one eventually, at which point this guidance migrates there. CONTRIBUTING.md is the home for now.
- `Makefile` and `CONTRIBUTING.md` added to WS5 Files; cadence bullet added to WS5 done-when.

### In-scope tweak (Workstream 7): backend decision table gains a `num_steps` row

**Date:** 2026-06-11
**Trigger:** Maintainer review session 2026-06-11. `num_steps` (Bézier interpolation points per curve) is a first-order lever at large scale that the decision table should surface. No new entry point or attribute read; no feasibility audit needed.
**Workstream affected:** Workstream 7 (done-when).
**Change:** New done-when bullet: the table includes a `num_steps` row stating that curve memory and build time scale linearly in `num_steps`, and at datashader scale fewer points are visually indistinguishable after rasterization. It is not equivalence-preserving (it changes output geometry), so it stays out of the Workstream 1-3 streaming paths and lands as documented user guidance only.

### In-scope tweak (Workstream 4): pin-time checklist gains a `make ty` check

**Date:** 2026-06-11
**Trigger:** Maintainer review session 2026-06-11. A type-check failure in a core dep blocks the whole boundary refactor, so it must be caught at pin time, not mid-refactor. No new entry point or attribute read; no feasibility audit needed.
**Workstream affected:** Workstream 4 (done-when).
**Change:** The WS4 pin-time checklist (previously living only in the grill Wave 1 amendment prose as the `py3-none-any` wheel check) is promoted to an explicit WS4 done-when bullet run at WS4 start, with two parts: (a) the existing check that the pinned narwhals version still publishes a `py3-none-any` wheel with no default compiled fast-path, and (b) new: `make ty` passes against narwhals's typing at the pinned version, including the `IntoDataFrame` annotation on the input boundary.

### Bookkeeping: harness gating semantics pointer (cross-plan, 2026-06-11)

**Date:** 2026-06-11
**Trigger:** The sibling plan `wiki/wiki/plans/performance-regression-harness.md` was amended in parallel (2026-06-11) to an ASV hybrid: pytest keeps the equivalence gate and relative same-run CI ratio assertions; ASV owns rolling baseline / history / blog-figure data.
**Workstreams affected:** none substantively; all harness-gated validation bullets inherit the new semantics from the harness plan.
**Change:** One-line pointer added to "Sequencing and scope guards" noting the ratio-based gating semantics. The harness plan file itself was not edited from here.

### In-scope tweak (Workstream 5): name the measurement methodology for the multi-process Dask and cuDF/GPU memory regimes

**Date:** 2026-06-26
**Trigger:** Maintainer discussion (2026-06-26). The harness's `measure_peak_rss` is single-process by design, which fully covers WS1–3 (numba parallelism is thread-based: shared address space, captured by one process's peak RSS) but does **not** cover WS5's two new memory regimes. Recorded so nobody publishes a WS5 memory figure produced by a measurement that silently misses most of the allocation. No new entry point, no new attribute read; no feasibility audit needed.
**Workstream affected:** Workstream 5 (done-when only).
**Change:**
- The vague WS5 measurement line ("Performance runner gains a small Dask-input scenario (or it is left as a worth-discussing follow-up by the api-critic)") is **upgraded to a named measurement-methodology requirement**: before any WS5 memory number is trusted, WS5 must define how peak memory is measured for (a) the **multi-process Dask path** — process-tree RSS aggregation across worker processes, because a multiprocessing / distributed scheduler spreads allocation across processes the parent's `ru_maxrss` does not see — and (b) the **cuDF/GPU path** — a distinct GPU-VRAM metric (cupy mempool stats / `nvidia-smi`), because GPU work allocates VRAM, not RSS, and peak RSS misses it entirely.
- A small Dask-input scenario on the performance runner remains a reasonable first step, but it must state which metric it reports (process-tree aggregate, not single-process RSS).
- Cross-references the canonical record: the measurement-scope-limitation note on Workstream B in `wiki/wiki/plans/performance-regression-harness.md` (added in parallel, 2026-06-26), which holds the full rationale for why `measure_peak_rss` covers WS1–3 but not WS5.
- **Proportionality (recorded):** WS5 is the furthest-out workstream and gated behind WS1–4. This is a "design the measurement before you trust the numbers" flag, not a demand for a full multi-process / GPU benchmarking rig now. Consistent with the harness plan's standing "Out of scope: heavy GPU/cuDF/Dask benchmarking deps" boundary; the extension is WS5's responsibility, not WS-B's.

### In-scope tweak (Workstream 5): GPU-VRAM metric pinned to per-process cupy mempool peak; tier-2 multi-process RSS inherited from the harness MR

**Date:** 2026-06-26
**Trigger:** Maintainer discussion (2026-06-26) following a local GPU capability probe, keeping WS5 aligned with the harness plan's refined GPU-VRAM note (`wiki/wiki/plans/performance-regression-harness.md`, "2026-06-26 — GPU-VRAM note refined with local capability probe; stays WS5"). Sharpens the GPU bullet of the 2026-06-26 measurement-methodology amendment above; does not move scope. No new entry point, no new attribute read; no feasibility audit needed.
**Workstream affected:** Workstream 5 (done-when only).
**Change:**
- **Canonical GPU metric pinned to per-process cupy mempool peak (exact), not `nvidia-smi`.** The earlier "cupy mempool stats / `nvidia-smi`" either/or framing is replaced: `nvidia-smi` reports whole-GPU memory, which is shared and noisy (the canonical host already sits ~5 GB used by unrelated apps), so it cannot isolate the workload's peak. cupy's mempool stats are per-process and exact. `nvidia-smi` is downgraded to a fallback / sanity-check role only. The WS5 GPU-VRAM done-when bullet (and the GPU sub-bullet of the 2026-06-26 measurement-methodology amendment) are updated to say per-process cupy mempool peak.
- **Testable locally, not hardware-blocked.** The canonical host has a usable GPU (NVIDIA RTX 5070, 12 GB). The GPU-VRAM metric lands at WS5 for a dependency-and-consumer reason, not a missing-hardware reason: there is no cuDF / datashader-CUDA code path to measure until WS5, and the cupy/RAPIDS stack that both enables the synthetic test and is the metric source arrives with WS5. As of 2026-06-26 the env has no working GPU-allocation library (`numba.cuda.is_available()` is False; cupy and pynvml absent), so the synthetic test and the metric install together at WS5.
- **Hard done-when gate.** No WS5 GPU scaling claim is accepted without a measured per-process VRAM peak on the local GPU via cupy mempool stats. Not stubbed, not assumed.
- **Tier-2 multi-process RSS now inherited, not owned by WS5.** The harness's tier-2 multi-process measurement (process-tree sampled peak-RSS, `measure_peak_rss_tree` / `tree=True`) has already shipped in the harness MR (it is buildable and testable without Dask, validated against a synthetic stdlib `multiprocessing` workload). WS5 inherits it for the Dask multi-process case and **only owns the GPU-VRAM piece**. This narrows the 2026-06-26 measurement-methodology amendment above, whose multi-process Dask sub-bullet described a metric WS5 would have had to build; that work now lands upstream. Cross-references the harness plan's Workstream B measurement note (`wiki/wiki/plans/performance-regression-harness.md`, "2026-06-26 — Tier-2 process-tree peak-RSS helper moves INTO Workstream B").

### Bookkeeping: WS7 harness-dependency pointer corrected (`time_median_of_n` dropped upstream)

**Date:** 2026-06-27
**Trigger:** Maintainer-approved documentation-reconciliation pass on the harness plan (2026-06-27; see that plan's 2026-06-27 amendment in `wiki/wiki/plans/performance-regression-harness.md`). WS7's "Depends on" line still cited the harness's "WS-B measurement primitives (`time_median_of_n`, median-of-N default N=5, fixed seeds, plus the peak-RSS helper)"; `time_median_of_n` was dropped by the harness's 2026-06-11 ASV amendment (ASV `time_` benchmarks own timing history/statistics), so the pointer described a helper that never shipped.
**Change:** WS7's dependency wording corrected to: ASV `time_` / `peakmem_` benchmarks plus the harness's `measure_peak_rss` / `measure_peak_rss_tree` helpers as the source of every published number. No scope change, no new entry point or attribute read (no feasibility audit needed); stale-pointer correction only.

### In-scope tweak (Workstreams 2 + 3): "harness-gated validation" requires arming the dormant gates — unskip, canary deletion, explicit RSS-constant decision

**Date:** 2026-07-02
**Trigger:** Adversary post-impl review of the shipped performance-regression harness (2026-07-02; full record in `wiki/wiki/plans/performance-regression-harness.md`, "Adversary post-impl review (2026-07-02)" and its 2026-07-02 amendment). Must-fix finding #2 there: WS2's "Harness-gated validation" done-when was satisfiable with the harness's three streamed-vs-single-shot gates still dormant (skip-marked), because nothing mechanical forced them to arm when `stream_chunk_threshold` lands. The harness MR adds a canary test in the always-running default suite that fails the moment the parameter appears in the datashader entry point's signature; this amendment makes the arming an explicit WS2 obligation.
**Workstreams affected:** Workstream 2 (done-when edited), Workstream 3 (harness-gated bullet edited).
**Change:**
- **WS2's harness-gated done-when now names the enforcement explicitly:** (a) unskip and wire the three dormant gates in `tests/performance_regression_test.py` — tiny-floor timing band; large-scenario streamed-vs-single-shot peak-RSS fraction; streamed-vs-single-shot raster equivalence via per-side render kwargs; (b) delete the canary test that guards the arming (the `stream_chunk_threshold` signature canary in the default suite); (c) make the 0.5 RSS-fraction (`STREAMED_PEAK_RSS_MAX_FRACTION_LARGE`) unskip decision explicitly against the WS2-alone vs WS2+WS3-cumulative distinction (interactions §2) rather than quiet retuning. Note the dormant tiny-floor timing gate WS2 inherits was also de-biased in the harness MR (unconditional warmup of both paths + best-of-N; the cold-vs-warm first-call import/JIT bias was ~66x), so the 1.25x band is meaningful at unskip time.
- **WS3's harness-gated bullet inherits the same three-gate obligation on the fused path:** the gates armed at WS2 run against the fused route too, not only WS2's chunked rasterization.
- No new entry point, no new attribute read (no feasibility audit needed); done-when enforcement tightening only. The canary test's exact name is the code-engineer's call in the in-flight harness MR; this plan refers to it descriptively.

### In-scope tweak (Workstream 5 + chain close): Dask memory instrument becomes an explicit selection; harness tier-2 demoted to provisional with a pruning checkpoint

**Date:** 2026-07-03
**Trigger:** Maintainer review of the harness's tier-2 process-tree peak-RSS helper (2026-07-03; decision is keep-but-demote; canonical rationale record in `wiki/wiki/plans/performance-regression-harness.md`, "2026-07-03 — Tier-2 process-tree helper demoted to provisional"). The prior framing — WS5 "inherits tier 2 for the Dask multi-process case rather than building its own" (2026-06-26 amendments above) — presumed tier-2 is the chosen instrument; it is a candidate. No new entry point, no new attribute read; no feasibility audit needed.
**Workstream affected:** Workstream 5 (measurement-methodology done-when edited) plus a new chain-close obligation recorded alongside ADR promotion under "Prior ADRs / design docs".
**Change:**
- **WS5's multi-process Dask measurement sub-bullet becomes an explicit instrument SELECTION** among three candidates, with the deciding fact recorded: Dask's default local scheduler for dataframe work is **threaded** (one process), in which case tier-1 single-process exact RSS suffices and neither tier-2 nor Dask-native diagnostics are needed. Candidates: (a) tier-1 if WS5 lands on the threaded scheduler; (b) Dask-native diagnostics (MemorySampler / per-worker metrics) if on the distributed scheduler — preferred where applicable; (c) tier-2 only where an external whole-tree observer is needed (the multiprocessing scheduler, which has no Dask diagnostics; or external verification of framework self-reporting, which matters most for the future autonomous-loop use where the metric must resist gaming). Supersedes the "inherited from the harness MR, not owned by WS5" wording in the two 2026-06-26 entries above (left intact per append-only).
- **Maintainer rationale (durable principle, recorded in substance):** low-level bespoke code whose correctness is contingent on the implementing session's understanding rather than the maintainer's fluency is a maintenance liability; ecosystem-standard instruments with community guarantees and uniformity with how the rest of the world profiles (e.g. Dask's own profiler for Dask code) are preferred even at some capability cost. Tier-2's one retained unique justification is the external-observer/ungameability property.
- **Hard pruning trigger (chain-close checkpoint, the compromise):** if, as the scaling workstreams actually execute, the profiling practice in use does not employ tier-2 and its presence can no longer be justified, it MUST be removed (the `measure_peak_rss_tree` function, `_sample_tree_rss`, its two tests, the tree workload, and psutil from the `testing` extra if nothing else needs it) BEFORE this super-plan closes. Removal or retention is an explicit recorded decision at chain close, tracked in the chain-close checkpoint added under "Prior ADRs / design docs" alongside ADR promotion — not a silent lingering.

### In-scope tweak (Workstream 3, validation riders on Workstreams 4 + 5): vector-backend render canary benchmark (matplotlib)

**Date:** 2026-07-03
**Trigger:** Maintainer discussion (2026-07-03). **The risk:** the harness's render benchmarks currently cover only the datashader backend. All four vector backends (matplotlib, bokeh, holoviews, plotly) consume the same persisted-curves contract (float32 arrays, NaN-separated blocks) via the shared two-stage path. A later workstream doing clever things in Dask space (WS4/WS5) could earn its speedup via a change that, e.g., adds a persistence/materialization step on the shared exit path, taxing every non-datashader build while all datashader-specific gates stay green. Functional tests catch breakage and the equivalence gate catches numeric drift, but nothing measures whether vector-backend consumption of the shared contract got slower.
**Workstreams affected:** Workstream 3 (Files + done-when; the benchmark lands with WS3's dispatch, the first workstream that touches curve persistence, where the risk first materializes), Workstreams 4 and 5 (harness-gated validation bullets; once in the suite the canary runs at every capture-at-merge thereafter, so canary-within-historical-band becomes part of "no regression").
**Change:**
- One `time_` ASV benchmark rendering through the **matplotlib backend** (the library's default viz entry point) at the small scale (optionally tiny as well; implementer's call, both are milliseconds) is added to `benchmarks/benchmarks.py` as part of WS3. Matplotlib is a core dependency, so this adds zero ASV environment weight; since all vector backends consume the identical curves structure, one benchmark canaries the whole family.
- **Explicitly NOT added:** bokeh / plotly / holoviews benchmarks. Env bloat for optional extras, no planned changes to those renderers, and the maintainer's ecosystem-minimalism principle.
- WS3's done-when gains: benchmark exists, captured in the ASV suite, no regression vs. the pre-WS3 baseline. WS4's and WS5's harness-gated validation bullets gain one line each: the matplotlib render canary staying within its historical band is part of "no regression," covering exactly the Dask-cleverness-taxes-the-shared-path scenario.
- **Proportionality, on the record:** the maintainer called this "probably overkill in practice" but wants the negative result recorded. The value is being able to say no backend regressed, not just the measured ones.
- Dev/benchmark tooling only; no new library entry point, no new attribute read (no feasibility audit needed). A one-line cross-reference added in parallel to `wiki/wiki/plans/performance-regression-harness.md` (Workstream C's ASV history bullet) noting the planned WS3 suite addition.

### In-scope tweak (Workstreams 2 + 3, rider on Workstream 5): every new rendering mode carries dedicated default-suite unit tests

**Date:** 2026-07-03
**Trigger:** Maintainer testing directive (2026-07-03). Every new datashader rendering mode the scaling workstreams introduce (WS2's chunked/streamed path and its threshold branches, WS3's fused build path, and by extension any WS5 CUDA-path branches) must carry dedicated unit tests in the library's default test suite (`tests/`, `@pytest.mark.datashader`, running in the parallel coverage run), exercising each mode directly as a unit, not merely reached indirectly through the harness's gates. No new entry point, no new attribute read; no feasibility audit needed.
**Workstreams affected:** Workstream 2 (done-when bullet added), Workstream 3 (done-when bullet added), Workstream 5 (one-line rider appended to the cuDF/GPU passthrough done-when bullet).
**Rationale (recorded in substance):**

1. **Debuggability.** A per-mode unit-test failure localizes which render mode broke; a performance-harness gate failure says only that two paths disagree, which is much harder to debug. Redundancy between unit tests and the harness's equivalence gates is acceptable and expected: they serve different purposes (localized correctness vs. cross-path agreement).
2. **Structural enforcement, now in place.** As of the harness plan's 2026-07-03 `perf_harness` marker split (`wiki/wiki/plans/performance-regression-harness.md`, "2026-07-03 — Harness-validation tests get a dedicated `perf_harness` marker, out of the shipped-code CI signal"), harness-lane tests (`performance` / `perf_harness` markers) are deselected from the default coverage run and the serial performance stage runs `--no-cov`, so they contribute zero coverage. The 100% gate on `src/hiveplotlib` can therefore only be satisfied by default-suite unit tests; any new rendering-mode line without one fails CI. The maintainer explicitly accepts temporary coverage failures during workstream development as pressure on the implementing/qa agents to write the proper unit tests rather than leaning on harness-lane coverage.

**Change:**
- WS2 done-when gains a dedicated-unit-tests bullet naming its modes: the streamed path, the single-shot selection branch, the `stream_chunk_threshold` auto/forced threshold policy, and the var/std carve-out.
- WS3 done-when gains the parallel bullet for the fused build path (route selection, per-chunk handoff, discard behavior).
- WS5's cuDF/GPU passthrough bullet gains a one-line rider: new CUDA-path rendering-mode branches carry dedicated default-suite unit tests for their CPU-exercisable logic; only genuinely GPU-only execution stays behind the GPU-gated smoke test.
- **Dispatch briefs for WS2 and WS3 (and WS5's CUDA-path work) must carry this requirement explicitly**; the dispatching session names it in the code-engineer/qa-engineer briefs rather than relying on the agents finding it in the done-when.

### In-scope tweak (Workstream 5): no `[dask]` extra — the datashader extra is the Dask install story

**Date:** 2026-07-03
**Trigger:** Adversary challenge item 6 (worth-discussing), disposed in grill Wave 4: the planned `hiveplotlib[dask]` extra ignored that `dask[dataframe]` already ships inside the `datashader` extra (`pyproject.toml:34`, verified 2026-07-03).
**Workstream affected:** Workstream 5 (done-when: missing-integration error bullet). Supersedes the `[dask]`-extra add in the 2026-06-05 "maintainer dispositions on the three open A1 disclosure items" amendment.
**Decision:** No new extra. The datashader extra IS the Dask install story: the plan itself declares Dask-without-datashader nonsensical, so the target user already installs `hiveplotlib[datashader]` — which carries `dask[dataframe]` — before they can rasterize. A `[dask]` extra would be a strict subset/alias of one the user already needs: packaging ceremony with no configuration it enables, and its extra-naming `ImportError` would have no path to fire (the exact trap api-critic Concern 1 named, now confirmed by the packaging fact). The honest error surface is the existing top-of-module datashader-extra guard (`src/hiveplotlib/viz/datashader.py:23-28`), which fires at import of the rasterization module; narwhals owns the error for a frame it cannot dispatch. This also honors the "Dask and datashader travel together" design intent by packaging fact rather than a new extra, and keeps the maintainer's no-ceremony principle (the same logic that dropped `use_dask`).
**Change:** WS5's `ImportError` done-when rewritten in place (see the reconciliation entry below); the Dask notebook's install story is `pip install hiveplotlib[datashader]`.

### Reconciliation: Path A `use_dask` cascade executed into the normative sections

**Date:** 2026-07-03
**Trigger:** Adversary challenge item 1 (must-fix), disposed in grill Wave 4 as plan-text drift, no design change: the 2026-06-05 "Removed parameter (Workstream 5): `use_dask` dropped entirely (Path A)" amendment listed cascading edits that were never performed; the normative sections still mandated the rescinded parameter. Under auto-dispatch an implementer would execute a spec the maintainer explicitly killed.
**Workstreams affected:** Workstreams 3 and 5, Naming audit (WS2 parenthetical, WS3, WS5), Default justifications, API usage examples, api-critic post-impl placeholder, WS5 Files. All edits performed in place this pass:
- **WS5 done-when walked bullet-by-bullet against Path A + the two 2026-06-05 disclosure amendments:** dropped the `use_dask` parameter bullet, the raise-on-`None` `TypeError` + pinned-message-test bullet, and the four-branch test bullet; kept the ~2-partition end-to-end equivalence test reframed without the flag; rewrote the missing-extra `ImportError` bullet per the no-new-extra decision above; folded in the disclosure amendments' single-detection-point (`edges.py:164`) and failure-point-reraise (`raise ... from err`) done-whens, which had also never been cascaded; rewrote the doc bullet to the two gallery notebooks (2026-06-05 two-notebook amendment, also uncascaded) and the CHANGELOG bullet (no new parameter). WS5 Files' single-notebook line replaced by the two notebooks + index registrations.
- **WS3:** the fused-route selection clause ("`stream_chunk_threshold`, and for foreign frames `use_dask`") respecified in the done-when and the WS3 Naming audit: foreign-frame input (Dask, cuDF) rides the same single shared streaming policy (interaction §1); no engine-specific selector exists.
- **Naming audit (WS5)** rewritten to the zero-parameter surface; **Default justifications** `use_dask` subsection replaced with a Path A stub and the per-workstream defaults line corrected; **Example 4** rewritten to plain passthrough naming the two Dask footguns inline one line each (per api-critic Concern 3: the cuDF symmetry is a trap; the surfaces are uniform in shape, not in what the user must know); **Example 4b deleted**; the WS5 api-critic post-impl placeholder reworded (behavior change, no new parameter).
- Residual-mention sweep run over the plan: remaining `use_dask` mentions live only in historical sections (grill waves, superseded amendments, preserved api-critic takes, adversary challenge), which stay verbatim per append-only.
- Feasibility audit: no new entry points, no new attribute reads anywhere in this pass — reconciliation and done-when tightening only.

### In-scope tweak (Workstream 1): consumer map corrected to all six verified read sites; positional-indexing rule; non-RangeIndex tests

**Date:** 2026-07-03
**Trigger:** Adversary challenge item 2 (must-fix), disposed in grill Wave 4: WS1's "sole consumer" premise was false, and bool→integer storage silently shifts indexing semantics at the unlisted consumers. All sites grep-verified this pass (`rg relevant_edges src/`), not taken from the adversary's report.
**Workstream affected:** Workstream 1 (Files + done-when, rewritten in place), Patterns this replaces (WS1).
**Change:**
- Verified read sites (all move in lockstep): `viz/matplotlib.py:460-463`, `viz/bokeh.py:580-581`, `viz/plotly.py:726-729` + `:774-776`, `viz/holoviews.py:636-637`, `viz/datashader.py:203-204`, `hiveplot.py:4650-4655` (`to_json`, def `:4561`). Verify-not-break sites (value-type-agnostic): `edges.py:244-256` (`Edges.copy()`), `hiveplot_matrix.py:693` (shared reference), `hiveplot.py:782-835` (reset/pop).
- New done-when bullets: byte-identical output across ALL backends (not just datashader); positional-only application of the indices (`.iloc`/`take`; `.loc[int_array]` is label-based, `df[int_array]` is column selection); non-RangeIndex edge-frame test coverage so label/positional divergence cannot pass green (Failure mode 4); index dtype an explicit logged decision.
- Stale line refs corrected plan-wide in normative sections against the working tree (verified this pass): `__store_edge_ids` def `:893` / store `:958` (was `:797`), `add_edge_ids` `:962+` (was `:801+` / `:707-749`), `construct_curves` `:1381+` (was `:1068+`). The WS1 dispatch brief must be grep-anchored, never plan-line-ref-anchored.

### In-scope correction (Workstream 1 + Prior ADRs): WS1 value restated as structural; OGB-arxiv unblock re-attributed

**Date:** 2026-07-03
**Trigger:** Adversary challenge item 5 (worth-discussing), disposed in grill Wave 4: "strictly better in-RAM whenever multiple axis-pairs exist (always)" is arithmetically false in the common case (3 axes: int64 indices ≈ 8 bytes/edge vs ~6 one-byte bool masks), and the Prior-ADRs bullet crediting WS1 with the OGB-arxiv unblock judged WS1 against an expectation it cannot meet.
**Sections edited in place:** Patterns this replaces (WS1) — claim restated at O() level plus the structural justification (one storage decision extended by WS5; a positional-take consumer shape WS2/WS3 stream over; MBs against the curves' GBs either way), superseding the same claim in the 2026-05-29 Added-workstream amendment (left verbatim per append-only). WS1's harness-gated done-when reframed to "records the membership-storage delta" with the honest expectation stated. Prior ADRs: OGB-arxiv unblock re-attributed to WS2+WS3 (1.16M edges ≈ 1 GB of curves is feasible today) on both the gnn-heterogeneity and cora-prototype bullets; WS1 is the structural precursor.

### In-scope tweak (Workstream 4): pin-time checklist gains bound re-derivation

**Date:** 2026-07-03
**Trigger:** Adversary challenge item 7 (low-confidence), disposed in grill Wave 4: the `>=1.x,<2` narwhals bound was fixed in 2026-05 and the checklist only verified wheel purity + `make ty`, never that 1.x is still the live major at pin time.
**Change (performed):** WS4's pin-time checklist gains item (c): re-derive the version bound at WS4 start — confirm the range tracks narwhals's current live major, move it if not, or record why staying behind is right. Matching notes added at the Default-justifications pin decision.

### In-scope tweak (Workstreams 2 + 3): memory gates always run on the canonical host

**Date:** 2026-07-03
**Trigger:** Grill Wave 4 disposition of adversary item 4 (accepted and strengthened): memory-bound tests described as "skipped by default if RAM unavailable" could silently skip in the unattended window, satisfying the done-when with the one measurement that justifies the workstream never having run.
**Change (performed):** WS2's transient-RSS and WS3's resident-memory done-when bullets rewritten: on the canonical host (ringtail, 32 GB) the memory gates always run — the skip guard exists for foreign environments only; when RAM is contended they serialize after the parallel suite finishes; in the auto-dispatch window a skipped memory gate is a halt-back, not a pass.

### In-scope tweak (Workstream 5): dedicated GPU verification env; cuDF pandas<3 constraint

**Date:** 2026-07-03
**Trigger:** New environment fact (2026-07-03): cudf-cu12 26.6.0 requires pandas<3, incompatible in one env with the library's pandas-3 dev/CI environment (installing it into the main venv downgraded pandas 3.0.3 → 2.3.3; the venv is being rebuilt). A dedicated GPU verification venv now exists at `~/venvs/hiveplotlib-gpu` (cupy-cuda12x 14.1.1, cudf-cu12 + dask-cudf-cu12 26.6.0, dask/distributed 2026.1.1, pandas 2.3.3, editable hiveplotlib with the datashader extra); the cupy mempool peak metric was verified end-to-end on the RTX 5070 (2026-07-03).
**Change (performed):** WS5's GPU measurement bullet gains the environment update (gate environmentally de-risked; never install the cuDF stack into the main venv); `make test-gpu` must target the dedicated GPU env (or equivalent), never the main pandas-3 venv; the cuDF gallery notebook must name the pandas-version constraint rather than implying `pip install` into a pandas-3 env works.

### In-scope tweak (Workstreams 2, 3, 5, 6): failure-mode riders (modes 1-3)

**Date:** 2026-07-03
**Trigger:** Grill Wave 5's `### Failure modes` rubric, bound into done-whens where a checkable rider is cheap; left rubric-only where a done-when would over-constrain.
**Change (performed):**
- **Mode 1 (renders-but-unreadable):** WS2 gains a done-when — the at-scale streamed render is validated readable at the shipped/documented raster resolution, with the rendered image in the between-workstream review packet (auditable in the auto window). WS6 gains a one-line rider on published figures. WS3/WS5 renders are rubric-only (the adversary's post-impl attack owns them; a per-workstream image requirement there would duplicate WS2's check on the same pipeline).
- **Mode 2 (edge-only scaling):** WS2's and WS3's 10M-edge synthetic scenarios must scale node count in proportion to edge count, stated in the scenario definition. Benchmark additions beyond these scenarios are rubric-only (the Failure modes list already binds the adversary; a blanket done-when on "any benchmark" would over-constrain harness coordination).
- **Mode 3 (unrepresentative parallelism):** WS5 gains a done-when — published Dask numbers state scheduler + hardware and are framed local-only unless verified representative; WS6's rider covers the blog. WS7 already requires machine-spec provenance on every number (no edit needed).
- Mode 4 landed as WS1's non-RangeIndex test requirement (see the WS1 consumer-map entry above).

### In-scope tweak (Workstream 4): polars output-equivalence gate

**Date:** 2026-07-03
**Trigger:** Adversary post-grill rubric-check must-fix (Failure mode 4): WS4's done-when required polars input to work end-to-end and round-trip its frame *type*, never that a polars-built plot draws the same picture as the identical data through pandas, while WS5's Dask branch already carries exactly that equivalence bullet. An engine-specific selection divergence in the narwhals re-expression (wrong rows selected on the polars path) would pass every stated WS4 gate and render a confident wrong picture.
**Change (performed):** WS4 "Done when" gains the mirrored equivalence gate: polars input produces identical edge selections (`relevant_edges` membership), identical node placements, and an equivalent rasterization within the harness tolerance vs. the pandas path on the same data, including at least one non-trivial case exercising `add_edge_ids` filtering plus a metadata-column reduction. WS4 is not yet started, so this lands in its done-when before dispatch; no shipped surface reopened.

### In-scope tweak (Workstream 1): scale-adaptive index dtype at `__store_edge_ids` (int32-overflow disposition)

**Date:** 2026-07-03
**Trigger:** Adversary WS1 post-impl `worth-discussing`: `.astype(np.int32)` at the store site has no overflow guard — past 2^31 - 1 rows, int64 positions wrap negative and `.iloc` silently selects wrong rows from the end of the frame (Failure mode 4 shape); guard-vs-promote needed settling before WS5 bakes the int32 assumption into the Dask counterpart.
**Change (in flight, code-engineer):** Maintainer chose a middle ground — scale-adaptive index dtype at `__store_edge_ids`: int32 while the row count fits `np.iinfo(np.int32).max`, int64 beyond. No raise, no unconditional promotion; dtype selection factored into a testable helper. WS5 bearing: the store-site comment (WS5's design record per WS1's done-when) now documents the adaptive policy; WS5's Dask non-materializing structure is unaffected (it never materializes index arrays). Supersedes the flat int32 dtype decision in the 2026-07-03 Implementation log entry; the implementing engineer records the update there on completion.

### Execution practice (all workstreams + merge): per-workstream marginal performance attribution

**Date:** 2026-07-03
**Trigger:** Maintainer ask (this session, post-Wave-4): per-workstream *marginal* performance attribution must stay recoverable, not just cumulative end-of-chain numbers.
**Change (recorded):**
- One commit per workstream on branch `53-scale-hiveplotlib-to-larger-networks` (the Wave 4 mechanic) makes every workstream boundary a named benchmarkable point in history.
- Interim: each boundary review packet carries dev-mode before/after numbers (`make benchmark-dev`) so the margin is visible per workstream during the sprint — throwaway sanity numbers, not the durable record.
- Durable record: after merge, the ASV suite runs per boundary commit on the canonical host (ringtail) across the whole series. ASV builds the package from each commit while reading `benchmarks/` from the working tree, so one consistent benchmark-definition set spans the series; benchmarks calling not-yet-existing API at an older commit fail for that commit — expected and fine. This per-commit series is the marginal-vs-cumulative data source WS6's blog figures consume (one-line pointer added to WS6's dependency framing).
- **Hard constraint on the maintainer's merge: branch 53 must NOT be squash-merged.** Squashing collapses the boundary commits and destroys the attribution series; merge preserving the per-workstream commits.

*Superseded in part (2026-07-06): the per-boundary ASV series is now optional and no-squash a maintainer preference; see the execution-practice update below. The boundary-commit mechanic and dev-mode interim numbers stand as executed history.*

### Disposition (Workstream 2 post-impl critics): streamed-mean NaN must-fix; exactness-set correction; matrix memory test to WS3; dpi bug out of scope

**Date:** 2026-07-03
**Trigger:** Adversary WS2 post-impl (1 must-fix, 2 worth-discussing with downstream bearing on WS3, 2 low-confidence); api-critic WS2 post-impl carried no items needing amendment action. Routed per rule 14; one entry, four dispositions.

1. **In-scope tweak (WS2), must-fix confirmed, fix in flight:** streamed `ds.mean` divides by the all-rows count where the single-shot baseline excludes NaN-valued rows from its denominator, so NaN-holding metadata yields a silently different raster whenever streaming engages (auto or forced), on both edge and node streamed paths. A correctness bug against WS2's own equivalence done-when; no design fork, the single-shot baseline defines correct semantics. Fix (code-engineer, concurrent with this pass): accumulate the mean denominator as per-chunk `ds.count(reduction.column)` (a separate aggregate from the plain-count density correction, which is correctly shared with single-shot); add NaN-metadata cases to the streamed-vs-single-shot parametrized tests on both streamed paths; add NaN-consistency checks on count/sum. WS2 stays commit-pending until this lands plus qa.
2. **In-scope correction (supersedes two normative sentences; downstream bearing WS3):** "Exact match for count/sum/any" (WS2 done-when, var/std-equivalence bullet; Default justifications, "exact for count/sum/any; division-tolerance for mean") is unattainable for sum and mean: float addition order across chunk partial sums; the shipped gates correctly assert rtol=1e-12/atol=1e-14. Normative exactness set, effective from this entry: **bit-exact for count/any/max/min; tight float tolerance (rtol=1e-12; provenance: addition order across chunk partial sums) for sum/mean.** WS3's fused-path gates inherit this set; a pointer was added to WS3's equivalence bullet this pass so WS3 dispatches against attainable language. Cascade into the two superseded sentences batches to the next plan-edit window (kept out of this pass to minimize the concurrent-edit surface). Process note: the deviation shipped documented in tests and the Implementation log without routing through amend-plan; this entry is the retroactive routing (same no-quiet-loosening principle as the 2026-07-02 arming amendment).
3. **In-scope tweak (WS3 done-when): matrix-level memory assertion folded into WS3 with a cheapness bound.** The WS2 pattern-block "also wanted" matrix-level memory test, deferred in the Implementation log only (roughly doubles stress-lane cost while exercising the identical per-cell code path; forwarding equality test covers the wiring), is now disposed at plan level: it rides WS3's resident-memory work (where matrix-level peak, the sum of per-cell resident geometry, actually changes character) **iff** it reuses WS3's existing stress-scale workload/measurement; if it would require a second stress-scale workload, reject with WS2's recorded rationale, rejection stated in the Implementation log. Bullet added to WS3's done-when.
4. **Out-of-scope note (seen, not adopted):** pre-existing `if dpi not in fig_kwargs` bug (`src/hiveplotlib/viz/datashader.py:501`, `:746`; the integer dpi value is tested against the dict's keys, always true, silently clobbering user-supplied `fig_kwargs={"dpi": ...}`). Scaling-unrelated; flagged to the maintainer as a separate task outside this plan.

No amendment action: the api-critic's worth-discussing/low-confidence items (explicit-force warning, `:raises:` line, carve-out phrasing, bool threshold, hardcoded "1,000,000" prose sites) and the adversary's remaining low-confidence item (stress-comment 20:1 proportionality misstatement) stay recorded in their critic sections and batch to plan-end qa per the auto-dispatch rule (no downstream bearing on not-yet-run workstreams).

### Disposition (Workstream 3 post-impl critics): ids-only route matrix authorized; vector-backend warning + ValueError fix + hardening adopted; doc folds

**Date:** 2026-07-04
**Trigger:** Adversary WS3 post-impl (1 worth-discussing with downstream bearing, 3 low-confidence); api-critic WS3 post-impl (5 worth-discussing). Both converge on item 1. Routed per rule 14; one entry, six dispositions. All are in-scope tweaks to WS3 (no added workstream; no new "Patterns this replaces" entries owed). Items 2-6 form the WS3 touch-up worklist; WS3 stays commit-pending until the touch-up lands plus qa.

1. **Design completion authorized (adversary WD + api-critic concern 4, both endorsing): ids-as-source-of-truth rendering is the recorded WS3 surface.** The shipped route matrix: datashader entry points render from stored ids regardless of construction state; all four `{single-shot, streamed} × {persisted, ids-only}` cells render; `stream_chunk_threshold` picks the memory strategy only; persistence happens only via `construct_curves()`. This exceeds the plan's literal "streamed fused" text, and the single-shot half is forced: restricting transient builds to the streamed route would make picture content flip with the threshold value (threshold as semantic selector), a worse violation of the same done-when. ONE-policy verified held (`_use_streamed_rasterization` is the sole route selector; curves-presence is orthogonal per-chunk state). **Supersedes** the final sentence of WS3's done-when third bullet ("It does not delete or change the default two-stage behavior."); replacement, cascaded into the bullet this pass (no concurrent editor; the touch-up dispatches after this pass): "The two-stage path's persisted output is unchanged bit-for-bit; additionally, the datashader entry points render from stored ids regardless of construction state, on both the streamed and single-shot routes (transient default-geometry build, never persisted); vector backends are unchanged." **Accepted behavioral corner, recorded:** a partially-constructed instance (ids stored both directions, curves deliberately built for one) now renders its unconstructed subsets at default geometry on the datashader path, where pre-WS3 raised `KeyError: 'curves'`; vector backends are untouched. **WS5 inherits these semantics** (the Dask/cuDF path rides the same policy); WS6/WS7 prose describes render behavior in these terms. This is the retroactive routing the WS2-exactness process precedent requires (no quiet contract deviation, even endorsed ones).
2. **In-scope tweak (WS3 touch-up), adopted: vector-backend ids-only warning (api-critic concern 2).** Adopted now, not deferred: WS3 is the release that makes ids-only a documented terminal state and raises the handoff traffic (datashader prototype, then vector-backend final figure); WS4's input boundary has no relation to the vector-backend guard sites, so deferral would orphan it. Contract: when a vector-backend edge render (matplotlib, bokeh, plotly, holoviews) encounters at least one edge record holding non-empty `"ids"` but no `"curves"` (covers fully ids-only and mixed states), exactly one warning fires per render call, naming `construct_curves()` as the fix and datashader as the backend that renders from ids. **Hard constraint: the warning must never fire on the datashader path**, and the shared `input_check(objects_to_plot="edges")` is called by `viz/datashader.py:568` too, so an unconditional edges-branch warning is wrong; gate it to the vector backends (mechanism is implementer's choice, e.g. an opt-in flag on `input_check` or a sibling helper the four call). Style: plain `warnings.warn(..., stacklevel=3)` matching the existing `input_checks.py` sites (no custom class; the exceptions-module convention governs exceptions, and this file's warnings are builtin). Tests: `pytest.warns` per backend plus a no-warn case on datashader and on fully-persisted renders; sweep the existing suite for vector renders in ids-holding-no-curves state that would now raise under `filterwarnings = error`. CHANGELOG **Changed** entry (released 0.28 behavior was a silently edge-less figure). Done-when bullet and Files line added to WS3's block this pass.
3. **In-scope tweak (WS3 touch-up), doc folds (api-critic concerns 1 + 3; adversary low-confidence stale-docstring item).** (a) Hoist the time half of the ids-only story into `datashade_hive_plot_mpl`'s `stream_chunk_threshold` doc at `src/hiveplotlib/viz/datashader.py:1077-1079` ("curves are rebuilt on every render; run `construct_curves()` once to cache them for repeated renders"); the wrapper and its `hive_plot_viz` alias share the docstring, and it is the scale user's likeliest entry point. (b) Mixed-state clause ("subsets with persisted curves render as stored, including non-default geometry; the rest build at defaults") in both entry-point `stream_chunk_threshold` docs where the ids-only sentence already lives, mirrored as one line on the `_chunk_edge_curves` seam docstring at `:246-250`. Doc-only: the mixed-condition warning option is rejected (narrow population; the api-critic itself judged the clause likely sufficient; item 2's warning covers the vector-backend half of the silent-state problem). (c) Fix `_streamed_edge_raster`'s stale `chunk_axis_pairs` param doc at `:322`: "holding at least one curve for ``tag``" becomes "holding at least one stored edge id for ``tag``".
4. **In-scope tweak (WS3 touch-up), adopted with its own CHANGELOG Fixed entry: pre-existing ambiguous-tag ValueError bug (api-critic concern 5).** The both-directions ValueError prints the `axis_id_2 -> axis_id_1` line twice and never shows the `axis_id_1 -> axis_id_2` tag list, at `src/hiveplotlib/hiveplot.py:1235-1236` (`add_edge_curves_between_axes`) and the same copy-paste at `:1747` (`add_edge_kwargs`). Fix: the first f-string line becomes `{axis_id_1} -> {axis_id_2}` over `hive_plot_edges[axis_id_1][axis_id_2].keys()`; add a test pinning that both directions appear in the message at both sites (grep confirms no existing test locks the current string, so this is a new assertion, no sweep). Adopted into the touch-up rather than a separate task: released-behavior fix (verified pre-existing against the pre-WS3 ASV build) in a WS3-touched method, the touched-file ratchet analog, and a two-token fix at two sites does not earn standalone-task overhead. Because it fixes released 0.28 behavior, it takes a CHANGELOG **Fixed** entry (0.29 currently has no Fixed subsection; add one), unlike the same-version refinements in items 1-3.
5. **In-scope tweak (WS3 touch-up): 4.5 stress-bound revisit trigger (adversary low-confidence).** Extend the `FUSED_STRESS_MAX_PEAK_OVER_IDS_ONLY_BUILD_RATIO` comment (`tests/performance_regression_test.py:125-136`) with one sentence naming the false-fail drift vector (the datashader/matplotlib import stack sits in the numerator child only, so dependency-upgrade RSS growth erodes the ~0.77 GB headroom with zero hiveplotlib change) and the revisit trigger: on a false-fail, re-measure the import-stack delta explicitly and record the new measurement before touching 4.5; never quietly retune. Mirrors the no-quiet-retuning protocol the 0.5 constant already carries.
6. **In-scope tweak (WS3 touch-up), adopted conditionally: pinned golden-value smoke on `_construct_edge_subset_curves` (adversary low-confidence).** One tiny fixed-seed test comparing the helper's output against a small hardcoded literal array (few edges, small `num_steps`): the only baseline in the battery that does not itself route through the shared helper, closing the shared-baseline equivalence topology executably. Exact float32 equality if stable, else `np.testing.assert_allclose` at tight rtol (extraction regressions such as wrong anchoring or direction are gross, not epsilon-scale). Condition per the adversary's own framing: if this cannot land as one small test (e.g. cross-platform value instability forces machinery), skip it and record one Implementation-log line; the structural argument (conservative control-angle passthrough plus nine-chunk bit-exact count gates) already stands.

No amendment action: the adversary's "dispositions held" confirmations and the api-critic's clean walkthrough records need none; the matrix-rejection statement the adversary left for qa is present in the 2026-07-04 Implementation log entry ("Matrix-level stress assertion: REJECTED per the cheapness bound") for qa to confirm; the CRLF process caveat is the dispatching session's standing pre-commit sweep (personal-gotchas entry), not plan content.

### Disposition (Workstream 4 boundary gaps): Holdout premise corrected, styling gathers + `add_nodes` adopted as follow-up; api-critic brief records

**Date:** 2026-07-04
**Trigger:** WS4 code-engineer surfaced three plan holes in its Implementation log entry (2026-07-04) rather than sweeping past the Holdouts section (correctly). Routed per rule 14; one entry, three dispositions. Items 1-2 are in-scope tweaks to WS4 (single-surface fixes riding the shipped boundary; no added workstream) and together form the **WS4 boundary-gap follow-up worklist**; WS4 stays commit-pending until it lands plus critics + qa.

1. **In-scope tweak (WS4 follow-up), adopted: column-mapped styling gathers re-expressed through the narwhals boundary; Holdouts rationale corrected.** The vector-backend Holdout's premise ("these backends do not see user dataframes; they consume numpy curves arrays") is false for column-mapped styling: when `edge_viz_kwargs` / `node_viz_kwargs` name metadata columns, kwarg resolution gathers directly from user edge/node frames at seven pandas-only sites, grep-verified against the working branch this pass (the replace-and-sweep set for qa): `viz/matplotlib.py:464`, `viz/bokeh.py:582`, `viz/holoviews.py:638`, `viz/plotly.py:730` + `:780` (hover customdata), `hiveplot.py:4842` (`to_json` kwarg resolution), `:4865` (`to_json`'s `nodes.data.set_index`). **Eighth site (2026-07-04, surfaced-then-converted during the follow-up itself):** `viz/plotly.py`'s hovertemplate column-position lookup (old `:645-652`, `np.where(df.columns == from_column_name)` — breaks on polars, whose `.columns` is a list; converted to a `hashable_columns` index lookup, behavior-identical on pandas; Implementation log has the record). Converted in the same dispatch rather than routed back because the mandated hover twin test cannot pass without it; qa audits the replace-and-sweep set as **eight** sites. Consequence: default rendering works on polars, styled rendering breaks — a half-shipped surface against WS4's own round-trip done-when and API Example 2's "no surprise pandas materialization" promise. Downstream bearing forces routing now, not batching to plan-end qa: the polars gallery notebook (WS4 deliverable, unbuilt) may legitimately demonstrate column styling, and WS5's Dask/cuDF input flows through the same sites. Fix: re-express the gathers positionally through the boundary — the `_edge_chunk_metadata_values` precedent (single column `to_numpy()` indexed by the stored integer positions) covers the column gathers; for the whole-frame gathers (bokeh CDS, holoviews, plotly customdata) the implementer picks narwhals row-gather or column-wise numpy materialization into backend-native structures (internal materialization for plotting consumption is fine; user frames are never mutated or coerced). Tests mirror the WS1 non-RangeIndex equivalence battery, polars twin vs pandas twin: matplotlib per-collection styling values, bokeh ColumnDataSource contents, plotly serialized figure incl. hover customdata, holoviews vdim values, `to_json` string equality; all polars-marked. **Rejected alternative:** documenting "column styling requires pandas" as a limitation — contradicts the round-trip contract and ships a silently-broken styling surface on the release that debuts polars support. Holdouts bullet corrected this pass; done-when bullets + Files lines added to WS4's block this pass.
2. **In-scope tweak (WS4 follow-up), adopted: `BaseHivePlot.add_nodes` concat through the boundary.** `hiveplot.py:621` still runs `pd.concat([nodes.data, self.nodes.data])`, so polars-on-polars incremental node adds fail with a raw pandas `TypeError` — an input-boundary surface with an illegible error, exactly WS4's remit. Folded into the follow-up rather than deferred to plan-end: `Edges.add_edges` already ships the verbatim-adjacent pattern (pandas fast-path, narwhals vertical concat, legible mixed-library `TypeError` at `edges.py:299-320`), so deferral would ship an `Edges`/`NodeCollection` asymmetry in the same release that debuts the boundary, to save a one-site + one-test cost. Mixed-library semantics mirror `add_edges`'s `TypeError`; the pandas fast-path is preserved. Test: polars-marked incremental add asserting the merged `nodes.data` stays polars, plus a mixed-library legible-error case.
3. **Record for the api-critic post-impl brief (no plan change):** (a) the new `[polars]` install extra (+ `dev` inclusion); (b) `Edges.frame_library` raises a legible `ValueError` on mixed-library tags — semantics worth critic attention (error vs. a per-tag report); (c) `create_partition_variable` with `labels=None` on non-pandas engines stores Interval bins as **strings** (polars holds Intervals only as an unfilterable Object column), so axis sort order can diverge from pandas at multi-digit interval bounds (lexicographic string sort); explicit `labels` avoid it; documented in the docstring; (d) Gate Zero outcome, recorded as the pin-time checklist earning its keep: the plan's `>=1.x,<2` bound was dead at pin time (narwhals's live major is 2 as of 2026-07; 1.x ended at 1.48.1), re-derived to `>=2.0.0,<3.0.0` with the 2.0.0 floor measured (all 43 polars-marked tests green against `narwhals==2.0.0`), not assumed.

No amendment action on the log's third surfaced hole (`nodes_edges_to_networkx` on polars-built plots is pandas-only): it stays a deliberate holdout, now actually listed — the prior Holdouts bullet named only the nx→hiveplotlib direction (`networkx_to_nodes_edges`), so the bullet was widened this pass to cover both directions with a revival trigger (route the `nodes.data` reads through the internal pandas materialization helper if polars→networkx demand appears). NetworkX interop is records-mediated by design and sits downstream of the round-trip contract.

### Disposition (Workstream 4 post-impl critics): `.columns` sweep finished; datashader fast-paths restored; labels=None divergence pinned; bookkeeping folds

**Date:** 2026-07-04
**Trigger:** Adversary WS4 post-impl (2 must-fix, 2 worth-discussing, 3 low-confidence); api-critic WS4 post-impl (1 must-fix, 4 worth-discussing, 3 low-confidence, 1 test-method-coverage gap). Routed per rule 14; one entry, nine dispositions. All in-scope tweaks to WS4 (no added workstream; no new "Patterns this replaces" entries owed — item 1 records its replace-and-sweep set for qa). Items 1-7 form the **WS4 post-impl touch-up worklist**; WS4 stays commit-pending until it lands plus the notebook critics + qa.

1. **In-scope tweak (WS4 touch-up), adversary must-fix 1 adopted: finish the `.columns` sweep.** Seven user-frame `.columns` attribute reads route through `frames.hashable_columns`: node-styling membership checks at `viz/matplotlib.py:316` and `viz/plotly.py:478`; node-hover column lists at `viz/plotly.py:451`, `viz/bokeh.py:412`, `viz/holoviews.py:460`; edge-hover column lists at `viz/bokeh.py:614`, `viz/holoviews.py:743` (all seven re-verified against the working tree this pass). The helper is already imported in matplotlib/plotly/holoviews; bokeh gains the import. Behavior-identical on pandas (fast-path) and polars (`.columns` already a name list); on pyarrow it converts reads that silently resolve garbage (`.columns` is a list of ChunkedArrays) into correct name lists. **Rejected alternative:** narrowing the advertised engine set — walks back the CHANGELOG's pyarrow example and the round-trip contract's spirit. Honest residual, recorded: the verified-by-test engine set remains pandas+polars; pyarrow stays advertised on the strength of the boundary architecture (post-sweep, no known user-frame read bypasses it — this family was the ninth-site sweep's one named remainder), not a pyarrow gate; no pyarrow test dependency added. Replace-and-sweep set for qa: the seven sites, plus a grep confirming no user-frame `.columns` reads remain in `viz/` or `to_json` outside `hashable_columns`. **WS5 bearing:** the cuDF/Dask input flows through these same sites; the adversary's claim-vs-sweep decision (sweep) lands before WS5 inherits it.
2. **In-scope tweak (WS4 touch-up), adversary must-fix 2 adopted: datashader pandas fast-paths restored.** `viz/datashader.py:286` (user edge frame): the read's job is the positional single-column gather — route through the fast-pathed `column_values_at_positions` (or isinstance-branch it; implementer's judgment on the local shape). `:434`/`:903` (`node_placements`, documented ALWAYS pandas): drop the `as_eager_frame` wrap or isinstance-branch it — the wrap guards a library that cannot occur there while breaking degenerate pandas (narwhals `DuplicateError` on duplicate columns; only pandas honors non-string column names). Plus one degenerate-pandas datashader regression test (duplicate-column or int-named metadata column through a `datashade_edges_mpl` metadata reduction and a `datashade_nodes_mpl` render, hitting `:286` and `:903`) pinning the 0.28 contract where it now actually bites. Restores the done-when's "pandas ... identically to today" to literally true.
3. **In-scope tweak (WS4 touch-up), api-critic must-fix + adversary worth-discussing adopted as a pair: labels=None divergence documented AND pinned.** (a) One sentence appended to the `create_partition_variable` note at `node.py:299-306`: because these string labels sort lexicographically, downstream axis order can differ from the pandas path at multi-digit bounds; pass explicit ``labels`` to control axis order. (b) One polars-marked twin test at multi-digit default-label bounds (e.g. cutoffs `[5, 20]`) asserting the documented divergence itself — pandas twin orders axes numerically, polars twin lexicographically — so a future narwhals/polars change cannot silently morph the documented behavior into a new undocumented one. Axis order is plot content (adjacencies, which edge chunks exist), hence executable pinning, not doc-only; this is deliberately the one twin in the battery that asserts divergence rather than equality. Corrects the 2026-07-04 boundary-gap entry's item 3(c) record ("documented in the docstring" — only the mechanism half had landed). The notebook half of the api-critic's suggestion routes via item 8(b).
4. **In-scope tweak (WS4 touch-up), adversary worth-discussing adopted: `set_axes_order` stringify-on-collapse for non-pandas engines.** The non-pandas branch at `hiveplot.py:3428` hands the mixed int+collapsed-name-string object array to `with_appended_column`, where polars strict inference raises an opaque mid-method error on int partition values (the pandas branch is engineered to preserve int keys, `:3410-3416`). Fix: stringify the collapse output on non-pandas engines (consistent with the labels=None partition-label stringification precedent; the pandas branch is untouched), one docstring sentence naming the divergence, plus an int-valued polars collapse test (the existing polars collapse test at `tests/hiveplot_test.py:7919` uses string labels only). Chose fix over legible-error: erroring would ship an engineered pandas behavior with no polars counterpart on a supported surface.
5. **In-scope tweak (WS4 touch-up), api-critic worth-discussing adopted: context-aware `as_eager_frame` accepted-inputs wording** at `frames.py:44-51`. When the context is `Edges` data, the message's accepted-inputs clause appends the (n, 2) numpy-array acceptance the class docstring advertises (branch on context — arrays remain genuinely unsupported on the other surfaces); stops steering a list-of-pairs user away from the documented `np.array(...)` fix.
6. **In-scope tweak (WS4 touch-up), api-critic worth-discussing adopted per the sibling-propagation rule: one polars-marked `HivePlotMatrix.from_partition` smoke/equality test** (likely green as-is — the direct reads at `hiveplot_matrix.py:1325` etc. duck-type on polars), placed in `tests/hiveplot_matrix_test.py` or `tests/polars_test.py` (implementer's placement). Preferred over a Holdout: the surface already works, users reading "NodeCollection and Edges accept narwhals frames" will reasonably feed polars-built collections to `from_partition`, and the sibling class surface needs its own record.
7. **In-scope tweak (WS4 touch-up), mechanical folds adopted:** (a) `CLAUDE.md:19` optional-backend marker list gains `.polars`; (b) `pyproject.toml:96-98` `[polars]` extra gains one comment line, matching the file's comment conventions: polars input support itself requires only core narwhals; the extra exists as the test/dev install vehicle (the CHANGELOG correctly does not advertise the extra; keep it that way); (c) `tests/hiveplot_test.py:7866` `test_add_edge_ids_polars_matches_pandas` gains a direct `hive_plot.add_edge_ids(...)` call in its body, closing the test-name = test-body contract gap.
8. **Routed elsewhere (not in the code touch-up):** (a) adversary low-confidence narwhals-floor gap → qa brief, item 9(d). (b) Adversary low-confidence permuted-index uid pitfall + the notebook half of item 3 → the pending notebook-author revision on `examples/creating_hive_plots_from_polars.ipynb` (already awaiting editorial/viz critics; one revision dispatch): a pitfall line — pass `unique_id_column` explicitly when moving frames between engines, since `pl.from_pandas` drops a permuted pandas index and valid-looking uids can silently name different rows — and a labels=None axis-order line where the notebook calls `create_partition_variable`. (c) api-critic low-confidence `Edges.frame_library` per-tag-mapping message polish → batched to plan-end qa (no downstream bearing; auto-dispatch batching rule).
9. **Records (fixed or noted this pass, no dispatch):** (a) P2CP added to Holdouts with a revival trigger (api-critic low-confidence; the third data-ingesting class now has a plan record). (b) `frames.py` public shape ACCEPTED as-is, recorded: it is the CLAUDE.md design note's documented boundary module — public-shaped names cross-referenced from the design note, deliberately not in the user-facing API reference; revisit only if a future docs pass promotes it to autodoc. (c) Bookkeeping fixed this pass: WS4's Files list gains `src/hiveplotlib/frames.py`, `tests/frames_test.py`, `tests/polars_test.py`; the dead `>=1.x,<2` literals in the Files bullet and done-when first bullet corrected to the re-derived `>=2.0.0,<3.0.0` (the Status line already recorded the re-derivation; the normative bullets now match). (d) **qa brief additions:** re-run the full 53-test `-m polars` battery against `narwhals==2.0.0` (the measured floor covered only the 43 pre-follow-up tests) and record the outcome in the Implementation log — if red, re-derive the floor and update `pyproject.toml`; audit item 1's replace-and-sweep set as seven sites; pick up the plan-end batched item from 8(c).

No amendment action on the adversary's "dispositions held" confirmations or attack notes; the CRLF caveat is the dispatching session's standing pre-commit sweep (personal-gotchas entry), not plan content.

### Disposition (Workstream 5 measured deviations): VRAM instrument becomes RMM allocation statistics; cuDF float-reduction upstream limitation recorded; tier-1 Dask selection pre-staged at chain close; pytest.ini GPU-only ignore accepted

**Date:** 2026-07-05
**Trigger:** WS5 code-engineer's Implementation log entry (2026-07-05) surfaced two measured deviations and two records for routing rather than silently absorbing them (correctly, per rule 14). One entry, four dispositions. All in-scope tweaks / records against WS5; no added workstream; no new entry point, no new attribute read (no feasibility audit needed); no new "Patterns this replaces" entries owed (instrument/record edits, no surface rewrites).
**Workstream affected:** Workstream 5 (Status line + two done-when bullets edited in place, pointing here), plus one pre-staged line at the chain-close checkpoint under "Prior ADRs / design docs".

1. **In-scope tweak (WS5 done-when), measurement-forced instrument amendment: the GPU-VRAM hard gate's instrument becomes RMM allocation statistics (`rmm.statistics`), superseding the per-process cupy mempool pin — and the gate is DISCHARGED.** Measured reality on the canonical stack: the cupy default pool reads exactly 0 bytes across the whole workload, because none of the workload's allocations route through it (datashader's CUDA aggregates allocate via numba; cuDF allocates via RMM), while whole-device `nvidia-smi` sits at 1.28 GB of context/library noise — confirming the very isolation concern that motivated the pin. The pinned instrument was therefore vacuous, not merely inconvenient: a cupy-mempool gate would have "passed" at 0 bytes without measuring anything. RMM allocation statistics carries the pin's full rationale (per-process, exact, isolates the workload from device-level noise) and is the RAPIDS-native tracker, aligned with the maintainer's ecosystem-standard-tooling principle (cf. the 2026-07-03 instrument-selection amendment's recorded rationale). The gate's INTENT — a real measured per-process VRAM peak, not stubbed, not assumed — is satisfied: 8,155,004 bytes (8.16 MB) single-shot / 16,315,420 bytes (16.32 MB) streamed on the 2k-edge smoke, RTX 5070. WS5's done-when GPU sub-bullet is edited in place (canonical-metric sentence and hard-gate sentence now name RMM statistics); the two 2026-06-26 cupy-pin amendments above are left intact per append-only. `nvidia-smi` remains fallback / sanity-check only. *(Per-route verification 2026-07-05, closing the post-impl disposition's item 1: each route re-measured in an isolated process — single-shot 8,155,022 bytes (8.16 MB), streamed 16,315,438 bytes (16.32 MB); a fresh per-route RMM scope inside one process reproduces the original streamed figure (16,315,420 bytes), so the pair above stands as the verified, publishable per-route numbers. The streamed 2.0007x is real, not a shared-epoch artifact: at smoke scale both peaks are raster-buffer-dominated — the streamed combine holds two ~8.16 MB aggregate buffers alive at once, single-shot holds one.)*
2. **Record (known upstream limitation, with revival trigger and a notebook/docs rider):** at the pinned stack (datashader 0.19.1 `_sum_zero._append_cuda`, numba-cuda 0.28.2, sm_120), cuDF float-sum-family reductions (sum/mean/var/std) fail NVVM kernel compilation; the GPU-supported reduction set at this pin is count/any/max/min. The cudf smoke battery pins raise-not-mis-render for the unsupported set, so an upstream fix flips tests rather than shipping silent wrongness. **Revival trigger:** re-run `make test-gpu` at any datashader or numba-cuda bump — this rides the existing CONTRIBUTING cadence, which already fires on datashader-bumping releases. **Rider on WS5's documentation done-when (edited in place):** the cuDF notebook and its docs page name the supported set and that the rest raise upstream, alongside the pandas<3 pin (the 2026-07-05 notebook-facts hand-off, item 4, already carries both). The upstream issue is being filed as a separate maintainer task, outside this plan's scope.
3. **Record (chain-close input pre-staged): Dask memory instrument selection = tier-1 `measure_peak_rss`,** per the 2026-07-03 explicit-selection amendment. Deciding fact verified in-env: Dask's default local scheduler for dataframe work is threaded (`dask.base.get_scheduler(collections=[ddf]) is dask.threaded.get`), one process, so single-process exact RSS captures the whole computation; Dask-native MemorySampler is distributed-only; tier-2 is unneeded on this route. One line added at the chain-close tier-2 pruning checkpoint ("Prior ADRs / design docs") stating that WS5 practice did NOT use tier-2, so the checkpoint's input is pre-staged. This pre-stages, not pre-decides: removal-vs-retention stays a chain-close call, since tier-2's one retained justification (external-observer/ungameability, e.g. for the future autonomous-loop use) is independent of whether WS5 happened to need it.
4. **Record (reviewed-and-accepted): the new pytest.ini `NumbaPerformanceWarning` ignore is fine as shipped.** It is message-scoped (`Grid size \d+ will likely result in GPU under-utilization`, a CUDA-only numba message), so it cannot mask CPU-side numba performance warnings (e.g. from the Bézier kernel) under `filterwarnings = error`; it only ever fires in `make test-gpu` runs (never CI — no GPU there, cudf tests skip), and tiny grids are inherent to smoke-scale data, not a defect. No maintainer flag needed.

### Disposition (Workstream 5 post-impl critics): streamed-VRAM figure re-measured per route before publication; CONTRIBUTING extra + materialization docstrings must-fix; adopt-all touch-up; Files-list correction

**Date:** 2026-07-05
**Trigger:** WS5 post-impl critic sections (api-critic "Workstream 5 review (2026-07-05)"; adversary post-impl "Workstream 5 (2026-07-05)"). Rule 14 fits: post-impl `must-fix` / `worth-discussing` findings. Every item is an in-scope tweak against shipped WS5 surfaces — no new entry point, no new attribute read (no feasibility audit owed); no new "Patterns this replaces" entries (message/docstring/test sharpening on surfaces this plan already owns).
**Workstreams affected:** WS5 (status line, Files-list test paths, GPU-gate discharge parenthetical, shuffle done-when annotation — all edited in place, pointing here); API usage Example 4 (inline comment corrected in place, this amendment's cascade).

1. **Priority — adversary WD1, downstream bearing (routed now, not plan-end-batched): the streamed VRAM figure is not publication-grade as measured.** 16,315,420 bytes streamed = 2.0007× the 8,155,004 single-shot — the signature of one shared monotone RMM-statistics epoch, and tests/cudf_test.py:202-206 runs exactly that pattern (both routes back-to-back in one process, no reset between them; `rmm.statistics` peak is monotone within an epoch). The face-value reading ("streaming doubles VRAM") contradicts the streamed route's bounded-by-largest-chunk story, and nothing in-tree measures isolated per-route peaks. Disposition, three parts: **(a) re-measure each route in an isolated process (or a fresh `rmm.statistics` scope per route — `push_statistics()` / `pop_statistics()`) BEFORE any VRAM number reaches the cuDF notebook or WS6**; the notebook and blog dispatches must not quote the current pair. **(b)** Once measured, the discharged-gate record (WS5 GPU done-when sub-bullet + the 2026-07-05 instrument amendment above) is annotated with the corrected per-route numbers — whatever they measure, recorded honestly. **(c)** tests/cudf_test.py:180-210 is fixed to measure per route in isolation (fresh statistics scope per route), or keeps the shared epoch and asserts only what a shared epoch honestly supports (peak > 0, total growth), explicitly labeled as shared-epoch. **The gate's discharge STANDS** — a real, exact, per-process measurement exists and the instrument choice is right; recorded as **discharged-with-correction-in-flight** (instrument and single-shot figure trusted; the per-route split is not, pending (a)).

2. **Must-fix (api-critic): CONTRIBUTING.md:126 is not runnable as written** — `-e "<path/to/repository>[datashader]"` becomes `-e "<path/to/repository>[datashader,testing]"` (pytest.ini's addopts require the `[testing]` extra: pytest, pytest-cov, pytest-html, pytest-timeout, pytest-xdist).

3. **Must-fix (api-critic): the lazy-Dask materialization consequence lands on the public conversion surfaces.** One sentence each on `NodeCollection.to_pandas` (node.py:302-304), `Edges.to_pandas` (edges.py:281-283), `Edges.export_edge_array` (edges.py:391-396): a lazy (Dask) frame is computed into memory here — a deliberate materialization for an explicit conversion / export request. Closes the record-claims-documented gap (the notes currently live only on the private helpers).

4. **Worth-discussing: all five adopted into the same touch-up** (sentence-sized message/docstring edits; no behavior change except the warning branch):
   - `construct_curves` lazy TypeError remedy gains "and rebuild the hive plot", aligning with `_require_in_memory_edge_subsets` (hiveplot.py:1621-1630).
   - polars-LazyFrame rejection gains the collect-first clause: append "collect them to eager frames first (e.g. `.collect()` on a `polars.LazyFrame`)" (frames.py:44-57).
   - vector-backend ids-only warning goes lazy-aware: when the unconstructed subsets are lazy, lead with the datashader remedy (naming `.compute()` + rebuild) instead of `construct_curves()`, a guaranteed dead end on a lazy hive plot (input_checks.py:160-167); reorder or branch, implementer's call.
   - `_validate_edge_data` docstring rewritten to the shipped contract: `as_frame` named as the detection point (not `as_eager_frame`), lazy-Dask acceptance stated, reject list reworded to "lazy frames from other libraries", `:raises:` corrected; the `Edges` `:param data:` line gains the lazy-Dask acceptance (edges.py:170-179, :62).
   - Plan Example 4's shuffle comment: superseded story replaced (this amendment's own cascade, executed in place): no Dask sort exists inside hiveplotlib — axis placement materializes each axis's node subset into pandas, so the real call-site constraint is node data fitting in RAM (nodes are the in-memory side; edges are out-of-core). The WS5 shuffle done-when bullet is annotated resolved-no-shuffle; the notebook still answers the question visibly (Implementation log notebook fact 2).

5. **Low-confidence: five folded into the same touch-up; guard sharpening, all cheap:**
   - int-weight parametrization on the Dask equivalence twins (the NaN sentinel promotes metadata to float64 on the Dask route only, datashader.py:684-687; every current fixture is float-weighted, so the int-sum zero-fill vs float-sum NaN-fill divergence class never runs). Pass, or a documented fill-divergence note; either closes it.
   - sentinel sensitivity pinned by name: one-time local mutation run (delete the sentinel row; record which named test fails), with the failing test named in the sentinel comment; if nothing fails, add the empty-interior-partition test that does.
   - isinstance guard in the dask-native chunk loop for mixed lazy + eager same-tag subsets from different `Edges` instances, raising the existing avoid-mixing TypeError instead of AttributeError (datashader.py:733-735, :753-754).
   - cuDF moment-smoke raise-origin pin: the `except Exception` at cudf_test.py:243-249 matches NVVM / kernel-compilation in the message, so "raise" keeps meaning "upstream unsupported", not a hiveplotlib-side crash.
   - `DaskComputationError`'s quoted doc title (datashader.py:792) aligned to the shipping Dask notebook's H1; the exact title is pinned in the notebook-author brief.

6. **Batched to plan-end (no bearing on WS6/WS7 numbers or the notebooks):** the frames-module public-shape question — public docstrings cross-reference `hiveplotlib.frames.LazyEdgeSubset` with no autodoc page (hiveplot.py:1200-1202). Rides WS4's open frames-module item; resolved at the docs pass (autodoc page or underscore the module).

7. **Bookkeeping (adversary scope note, executed in place):** WS5 Files list corrected — tests landed at `tests/dask_test.py` + `tests/cudf_test.py` (flat layout, WS2 precedent), not the planned `tests/integration/` paths. Record, not creep.

**Execution shape:** one touch-up dispatch covers items 2-5 (source messages/docstrings + test sharpening + CONTRIBUTING), and the item-1 GPU re-measurement runs on the canonical host (`~/venvs/hiveplotlib-gpu`); both complete before the Dask/cuDF notebook dispatches (item 1 gates the cuDF notebook's numbers; item 4's warning wording is part of what the Dask notebook teaches). Item 6 waits for the docs pass.

### Disposition (Workstream 5 CI follow-up): stacked CI-only thread-safety fixes on the dask route; cross-kernel equivalence contract clarified as raster-level

**Date:** 2026-07-05
**Trigger:** post-chain CI failures on the WS5 dask route (Implementation log, 2026-07-05 CI-segfault entry, commit `21040d2`; the serial-kernel fix entry is being written by the concurrent code-engineer). Rule 14 fits: emergent findings against shipped WS5 surfaces. Both items are in-scope tweaks / records; no new entry point, no new attribute read (no feasibility audit owed); no new "Patterns this replaces" entries (thread-safety fixes plus a contract record on surfaces this plan already owns).
**Workstream affected:** WS5 (record only, plus a one-line equivalence-note annotation at the harness-validation done-when bullet pointing here; no done-when rewritten).

1. **Record (two stacked thread-safety bugs on the dask route, CI-only; both invisible on the 24-core canonical host across five green gate batteries).** (a) pandas-3 arrow string Index construction raced across dask worker threads; fixed in commit `21040d2` (interned shared column Index, identity fast-path, single-column gathers) and confirmed dead in subsequent CI stacks. (b) Unmasked beneath it: the numba `parallel=True` Bézier kernel invoked from multiple dask worker threads is not thread-safe on the workqueue threading layer (the only layer available in bare CI containers), and the parallel branch also mutates the process-global numba thread count. Fix in flight: the dask route forces the serial kernel. Dask's threads are the parallelism layer; nested numba parallelism was oversubscription regardless of the bug.

2. **Equivalence-contract clarification (the load-bearing record): serial and parallel kernel compilations are NOT bit-identical.** Fastmath codegen differs at the 1-ulp level (measured: f32 diverges above ~5k curves, f64 at all sizes); the library's own utils tests always pinned `allclose`, never bit-identity. Cross-kernel comparisons in the gates are therefore **raster-level equivalence**: pixel quantization absorbs 1-ulp coordinate diffs, all exact gates measured green under the forced-serial simulation, and the serial-vs-parallel comparison class was ALREADY green in CI (small partitions select serial below the 4096-point floor against parallel-built twins). **Residual risk, recorded with its remedy:** a future fixture/seed/resolution change could land a 1-ulp coordinate on a pixel boundary and flip an exact gate; the remedy is a recorded disposition (fixture/seed adjustment, or an explicit tolerance decision routed through amend-plan), never a silent tolerance bump. Where WS-level done-when language says "bit-exact" for cross-path raster comparisons, bit-exactness reads as a raster-level property contingent on quantization per this entry (annotated at the WS5 harness-validation bullet; history not rewritten).

### Execution-practice update (chain close + Workstream 6): marginal attribution downgraded; before/after evidence set suffices

**Date:** 2026-07-06
**Trigger:** Maintainer intent change: the blog post needs per-commit marginal profiling less than proof that the profiling done during execution showed each planned change worked with no functional regression, which the per-workstream gate batteries, equivalence walls, adversary passes, and the green pipeline on `44e3ab2` already established. Supersedes the named parts of the 2026-07-03 marginal-attribution amendment (annotated there); the boundary-commit mechanic and dev-mode interim numbers stand as executed history.

1. **Post-merge per-boundary-commit ASV series on ringtail: required durable record → optional.** Sufficient evidence set: the per-workstream gate measurements already recorded in this plan, the recorded perf comparison table and the two ogbn-products runs (2026-07-05 Implementation log entries), and one standard `make benchmark-capture` on master post-merge (standing practice), giving before/after against the pre-chain baseline. (A WS3-era post-impl record's "formal per-boundary ASV" reference stands as history.)
2. **No-squash on branch 53 / MR !36: hard constraint → maintainer preference.** Bisectability and CI-fix lineage remain ordinary reasons to keep the commits; nothing breaks if he squashes. During the in-flight review, edits still land as new commits (shared-MR hygiene, not attribution).
3. **WS6 data-source framing (Depends-on line edited this pass).** The blog's numbers trace to the harness gate records, the post-merge before/after capture, and the real-data OGB runs, telling a feasibility-first before/after story rather than a per-workstream marginal story. A marginal figure is bonus material iff the optional ASV series is ever run, not a dependency.

## Alternatives considered

- **Collapsing `Edges._data` from dict-of-DataFrames to single-DataFrame-with-tag-column.** Walked back during the planning conversation. The rasterization path (Workstream 2) iterates per-tag anyway; the collapse would be a breaking change to the `Edges._data` shape and the public `Edges.data` property without proportional benefit. Not pursued.
- **Polars-only adoption (drop pandas, switch internals to polars).** Polars's wins are on operations that are not bottlenecks in hiveplotlib (joins, groupbys at scale). The interop cost with matplotlib, networkx, bokeh, and datashader does not earn its keep when the dominant memory cost is the curves array, which is numpy regardless of frame backend. Narwhals at the input boundary captures the polars benefit without the cost. Not pursued.
- **Dask-only adoption (route everything through Dask).** Would slow the median pandas case (small graphs, single-machine). Optional Dask via Workstream 5 captures the benefit without the regression. Not pursued.
- **Flipping `bezier_xy_with_nans`'s default `dtype` from float64 to float32.** Cosmetic; every internal call site passes `dtype=np.float32` already. Public-API change without downstream beneficiary. Noted and left out of all workstreams.
- **Legacy `Node` / `dataframe_to_node_list` deprecation** at `src/hiveplotlib/node.py:430-450`. Worth doing, but it is scaling-unrelated tech debt and would muddy this plan. Belongs to a separate plan.

## Holdouts

Patterns deliberately left as pandas-specific (the qa-engineer should not flag these post-execution):

- `src/hiveplotlib/converters.py` — both directions (widened 2026-07-04; the bullet previously named only the first). `networkx_to_nodes_edges` (`:42`, `nx.to_pandas_edgelist`): NetworkX output is pandas; the immediate next step hands the frame to `Edges`, which (after Workstream 4) accepts pandas natively. `nodes_edges_to_networkx` (`:118-123`, pandas-idiom reads on `nodes.data`): records-mediated networkx interop by design; a polars-built plot fails here today — deliberate holdout with a revival trigger (route the reads through the internal pandas materialization helper if polars→networkx demand appears; see the 2026-07-04 boundary-gap amendment).
- `src/hiveplotlib/node.py:430-450` — `dataframe_to_node_list`. Independently flagged for deprecation; out of scope for this plan.
- `src/hiveplotlib/edges.py:151-153, 181-184` — internal `pd.DataFrame` construction from numpy arrays. The user did not bring a dataframe; we are creating a default to hold the array. No reason to route through narwhals.
- `src/hiveplotlib/viz/{matplotlib,bokeh,plotly,holoviews}.py` pandas operations on **internally produced** frames only (axis `node_placements` is the documented pandas seam; curve geometry is numpy). **Rationale corrected 2026-07-04:** the original premise "these backends do not see user dataframes" was false for column-mapped styling — the `edge_viz_kwargs` / `node_viz_kwargs` gathers (and `to_json`'s kwarg resolution + `set_index`) read user edge/node frames and are WS4 scope per the boundary-gap follow-up amendment, not holdouts. **Second correction (2026-07-04, post-impl disposition):** the user-frame `.columns` attribute-read family (node-styling membership checks + hover column lists, seven sites) also reads user frames and is WS4 scope, routed through `hashable_columns` per the WS4 post-impl disposition; after that sweep this holdout covers genuinely internally produced frames only.
- `src/hiveplotlib/p2cp.py:83` — `P2CP` remains pandas-only behind a legible assert ("`data` must be pandas DataFrame"). The third data-ingesting class, deliberately outside WS4's contract; user-facing acceptance claims are correctly scoped to `NodeCollection`/`Edges`. Revival trigger: route its `data` reads through the `frames.py` helpers if polars-P2CP demand appears (added 2026-07-04, post-impl disposition).
- All `src/hiveplotlib/datasets/` pandas operations. Internal dataset utilities; do not constitute user input boundary.
- `src/hiveplotlib/graph_features/__init__.py` pandas operations. Compute path that produces metrics; downstream of the input boundary.
- The `bezier_xy_with_nans` numpy float64 default at `src/hiveplotlib/utils.py:256`. Documented in Alternatives.

## Implementation log

- 2026-07-03: Workstream 1 complete. `BaseHivePlot.__store_edge_ids` now converts its boolean-mask parameter to integer row positions once (`np.flatnonzero(...).astype(np.int32)`), reuses them for the `"ids"` subset, and stores them in `relevant_edges[a1][a2][tag]`. **Index dtype decision: int32** — halves membership bytes vs int64 (keeping storage at or below the boolean masks it replaces at typical axis counts) while positional indexing caps at 2^31 - 1 rows, beyond any in-RAM pandas edge frame this path stores. **Dask counterpart (WS5 extends, not redesigns):** documented in the store-site code comment — lazy/out-of-core edge frames store an equivalent non-materializing membership structure at this same `__store_edge_ids` site instead of materialized positions; the boolean-mask parameter is retained so the engine-specific storage decision stays centralized at that one site. All six read sites moved to positional takes in lockstep (grep-verified via `rg -n relevant_edges src/` at dispatch): `viz/matplotlib.py` `.loc[mask, val]` → `_data[tag][val].iloc[idx]`; `viz/bokeh.py` bare-bracket mask → `_data[tag].iloc[idx]`; `viz/plotly.py` kwarg gather `.loc[mask, val]` → `_data[tag][val].iloc[idx]` and hover customdata bare-bracket mask + `.iloc[i, :]` → `_data[tag].iloc[idx[i], :]`; `viz/holoviews.py` `.loc[mask, :]` → `.iloc[idx]`; `viz/datashader.py` bare-bracket mask → `.iloc[idx]`; `hiveplot.py` `to_json` `.loc[mask, val]` → `_data[tag][val].iloc[idx]`. Verify-not-break sites confirmed value-type-agnostic (`Edges.copy()` deepcopy, docstring already index-array-worded; `hiveplot_matrix.py:693` shared reference; `hiveplot.py` reset/pop bookkeeping); `edges.py` attribute comment updated from boolean-array to integer-position wording. Tests: `test_add_edge_ids_stores_integer_indices` (multi-axis-pair; asserts int32 dtype, 1d shape, length == selected-edge count < total edges, strictly ascending order, equality with an independent `np.isin`-based recomputation, and `ids == edges[positions]`); non-RangeIndex equivalence tests parametrized over shuffled / offset-int / string edge indices for matplotlib (per-collection linewidths), bokeh (per-glyph ColumnDataSource contents), plotly (serialized figure incl. hover customdata), holoviews (per-element vdim values), datashader (raster equality under `ds.sum("weight")`), and `to_json` (string equality), each against a RangeIndex-built twin via the new `hive_plot_edge_index_factory` conftest fixture; the shuffled fixture was verified to yield divergent values under label-based `.loc` selection, so Failure mode 4 cannot pass green. Backend tests ride the module-level optional-dep markers. CHANGELOG entry added (framed as the `relevant_edges` stored-value change). ruff format/check and ty clean; full default suite 1378 passed at 100% coverage. Remaining for the gate battery: the harness-gated no-speed-regression sweep + recorded membership-storage delta (serial `performance` / `perf_harness` lane, not run in this workstream turn).
- 2026-07-03: Workstream 1 follow-up (adversary post-impl dispositions). **Membership index dtype policy changed from unconditional int32 to scale-adaptive:** new private helper `_edge_membership_index_dtype(num_edge_rows)` in `hiveplot.py` returns int32 while the edge frame's row count fits within `np.iinfo(np.int32).max` and int64 beyond it (no raise, no silent wrap, no unconditional promotion); applied at the `__store_edge_ids` store site. Store-site comment updated: the O(num_edges) storage bound is now stated via bounded multiplicity of edges across (from, to, tag) triples (repeat axes / bidirectional pairs), not "edges partition across triples." Tests: `test__edge_membership_index_dtype_scale_threshold` parametrized over typical / int32-max / int32-max+1 row counts (int64 branch covered with a plain integer, no giant frame); existing storage test unchanged, still asserting int32 at typical scale; `hive_plot_edge_index_factory` now self-asserts the shuffled/offset flavors diverge under label-based `.loc` selection (raises or misaligns), pinning Failure-mode-4 sensitivity executably. ruff format/check and ty clean; full default suite 1381 passed at 100% coverage. No new CHANGELOG entry (refinement to a change unreleased in 0.29; the existing entry's dtype-agnostic "integer array" wording remains accurate).
- 2026-07-03: Workstream 2 complete. Reduction-aware chunked rasterization shipped in `src/hiveplotlib/viz/datashader.py`: all three entry points (`datashade_edges_mpl`, `datashade_nodes_mpl`, `datashade_hive_plot_mpl`) gained `stream_chunk_threshold: Optional[int] = None` (appended last-in-signature so positional callers are unaffected; the wrapper forwards to both helpers; the `HivePlotMatrix` datashader path forwards per cell through the existing kwargs plumbing, verified by a matrix-level equality test). **Shared threshold policy (the single surface WS3 consumes):** module-level `_use_streamed_rasterization(num_items, stream_chunk_threshold, reduction)` + `_DEFAULT_STREAM_CHUNK_THRESHOLD = 1_000_000` drawn items (edges or node points) + `_streamed_combine_mode()` over the `_STREAMED_REDUCTION_COMBINE_MODES` exact-type table; call sites consume the decision, never re-derive it. Auto rationale recorded at the constant: ~0.8 GB transient per 1M edges at default num_steps, chunk overhead negligible at that scale, everything at/below the harness medium scenario stays byte-identical single-shot (interaction §1). Combine algebra: count/sum additive (NaN-aware for float rasters), `any` elementwise OR, mean accumulates per-chunk sum + count with ONE divide at the end (generalizing the density-correction divide); **max/min classification decision: SUPPORTED in the streamed path via elementwise `np.fmax`/`np.fmin`** (exact combine; verified bit-identical to single-shot); var/std and any unlisted reduction never stream (threshold ignored; stated in all three docstrings). Order of operations honored: raw per-chunk aggregates combine first, `tf.spread` + density divide exactly once at the end. Measured streamed-vs-single-shot equality: count/any/max/min bit-exact; sum/mean within ~9e-16 (float64 addition order across chunk partial sums). **Gates armed:** the three dormant gates in `tests/performance_regression_test.py` unskipped and wired via new workload pins (`FORCE_STREAMED_THRESHOLD = 0`, `FORCE_SINGLE_SHOT_THRESHOLD = 10**18`; `build_and_rasterize_large_single_shot` now pins single-shot explicitly since large sits above the auto threshold); canary test deleted (`git rm tests/performance_regression_canary_test.py`); ast constants-consistency test unaffected (stress constants deliberately outside `SCALE_SIZES`). **RSS-constant decision: `STREAMED_PEAK_RSS_MAX_FRACTION_LARGE = 0.5` KEPT and satisfied by WS2 alone** — measured on ringtail: single-shot 5.10 GB vs streamed 2.38 GB, ratio 0.466; margin is single-digit-percent (not comfortable) precisely because of interactions §2: both children still hold ~1.6 GB resident curves until WS3's fused build; decision + explicit revisit trigger recorded at the constant. **New 10M-edge stress memory gate** `test_measure_peak_rss_streamed_stress_scenario_transient_is_chunk_scale`: 500K nodes / 10M edges (the pinned 20:1 proportion, no token node counts), streamed build+render 10.28 GB vs build-only 9.28 GB = ratio 1.107 vs bound 1.35 (full-copy transient would land ~1.9); denominator is build-only rather than single-shot because single-shot at 10M approaches the host's 32 GB; skip guard keyed to TOTAL RAM >= 24 GB (foreign-env only; contention can never skip it on the canonical host). Per-mode default-suite unit tests (`@pytest.mark.datashader`): streamed-vs-single-shot parametrized over count/any/sum/mean/max/min for edges and nodes; forced-single-shot-equals-default; wrapper forwarding; var (mirroring `examples/datashading_statistical_summaries_of_metadata.ipynb`'s `ds.var`/`ds.mean` usage) + std threshold-ignored byte-equality; policy units (auto/forced boundaries, zero items, var/std/first/last never stream); streamed missing-metadata-column errors for edges and nodes. Readable-render check: 10M-edge streamed render at shipped defaults (figsize 10x10, dpi 150) saved to `/tmp/ws2_review/streamed_render_10m_edges_500k_nodes_default_raster.png` — axes, per-pair density lobes, and log-density gradients all legible. `runners/performance/README.md` canary references removed and the 10M-stress note updated. CHANGELOG entry added (new parameter vs released 0.28 behavior). ruff format/check and ty clean; full default suite 1402 passed at 100% coverage; serial perf lane (`make test-performance`) 35 passed, 0 skipped, 0 failed. Deferred with rationale: no matrix-level RSS gate (a matrix-scale memory child would roughly double the stress-lane cost while exercising the identical per-cell streamed code path; the matrix-level forwarding equality test covers the wiring).
- 2026-07-03: Workstream 2 follow-up (adversary post-impl must-fix + low-confidence #4). **Streamed `ds.mean` NaN correctness:** the streamed mean divide was using the accumulated all-rows `ds.count()` raster as denominator; single-shot `ds.mean(col)` is NaN-aware (divides by the column's non-NaN count), so any NaN in the metadata column silently shrank the streamed mean (measured up to 67% relative divergence on a 20%-NaN fixture). Fix in `src/hiveplotlib/viz/datashader.py`: both `_streamed_edge_raster` and `_streamed_node_raster` now accumulate a dedicated `mean_count_agg` per chunk via `ds.count(<mean's column>)` (non-NaN count, combined additively) and `_finalize_streamed_raster` divides the combined sum by IT; the all-rows `count_agg` is retained unchanged for the density correction, which is what single-shot divides by (`spread(mean)/spread(count_all)`), so both divides now mirror single-shot exactly. Comments/docstrings updated to say non-NaN count (combine-modes table, `_resolve_chunk_reduction`, `_finalize_streamed_raster`, both entry-point `stream_chunk_threshold` docs). Tests (`tests/datashader_test.py`): new `_example_hive_plot_with_nan_metadata` fixture (every 5th edge + every 5th placed node gets NaN `"low"`); streamed-vs-single-shot NaN cases parametrized over mean/sum/count for both `datashade_edges_mpl` and `datashade_nodes_mpl` at the existing tolerances (sum/count prove the add-combine paths NaN-consistent too); var/std carve-out NaN cases for edges and nodes (byte-identical, single-shot by construction). Sensitivity verified out-of-tree: emulating the old all-rows denominator diverges 50-67% on the fixture while the fixed path matches within float64 ulps, so the new mean tests would have caught the bug. Also reworded the stress-gate ratio comments in `tests/_performance_regression_workloads.py` and `tests/performance_regression_test.py`: the 20:1 edge:node ratio mirrors the large scenario only (tiny/small/medium are ~2.3:1 / 2.5:1 / 10:1), not "every pinned scenario". No CHANGELOG entry (fix to a feature unreleased in this version). ruff format/check and ty clean; full default suite 1412 passed at 100% coverage; serial perf lane (`make test-performance`) 35 passed, 0 skipped, 0 failed.
- 2026-07-04: Workstream 3 complete. Fused build-and-rasterize route shipped. **Where it lives:** new `BaseHivePlot._construct_edge_subset_curves(from_axis_id, to_axis_id, tag, num_steps=100, ...)` in `src/hiveplotlib/hiveplot.py` computes and RETURNS one stored `(from, to, tag)` subset's Bézier curves without persisting anything; `add_edge_curves_between_axes` now delegates its per-direction geometry to it (two-stage output preserved bit-for-bit via an explicit `control_angle_axis_ids` passthrough that keeps that method's historical `(axis_id_1, axis_id_2)` control-point anchoring for BOTH directions, since anchoring is order-symmetric mathematically but not always bit-for-bit). Datashader side (`src/hiveplotlib/viz/datashader.py`): new `_chunk_edge_curves` get-or-build seam — persisted curves returned as the stored object, missing curves built with defaults (bit-identical to `construct_curves()` defaults) and never persisted — consumed by `_streamed_edge_raster` (fused route: build chunk → WS2 per-chunk aggregator → drop with the chunk dataframe) and by the single-shot collection loop (transient build, O(total) transient as ever, still zero persistence); chunk collection now gates on stored `"ids"` rather than `"curves"`; `_edge_chunk_metadata_values` takes the chunk's curve row count so per-chunk metadata rides transiently built chunks (WS2's NaN-aware dual-denominator mean structure untouched and re-proven on the fused route). **Route selection:** the WS2 shared policy ALONE (`_use_streamed_rasterization` / `stream_chunk_threshold`; no new parameter, no second threshold); curves-persisted vs ids-only is instance state, not a selector — all four `{single-shot, streamed} × {persisted, ids-only}` cells render, fused = streamed × ids-only. **Non-persistence is datashader-only:** a datashader render never mutates `hive_plot_edges`; vector backends see unchanged state and still require `construct_curves` (interaction §5). **Discard contract (explicit, tested):** transiently built curves are never stored (no `"curves"` key appears after fused or single-shot ids-only renders), and prior persisted curves are reused as the SAME array objects, never clobbered or rebuilt; mixed persisted/ids-only states render per-chunk independently. **Gate numbers (canonical host):** large fused-vs-two-stage RSS fraction — two-stage single-shot 5.06 GB vs fused streamed 1.05 GB, ratio 0.208, new constant `FUSED_PEAK_RSS_MAX_FRACTION_LARGE = 0.30`; **`STREAMED_PEAK_RSS_MAX_FRACTION_LARGE = 0.5` deliberately NOT retuned** — its two children both still hold resident curves, so WS3 changes nothing it measures; the fused improvement interactions §2 predicted (0.466 → 0.208) is recorded in the new fused constant instead of quietly tightening WS2's (explicit decision, not silence). Stress resident-memory gate (10M edges / 500K nodes, 20:1 per Failure mode 2; always-runs policy identical to WS2's, total-RAM ≥ 24 GB skip guard for foreign hosts only): three children off one ids-only assembly — assembly-only 0.975 GB, fused streamed render 3.628 GB (ratio 3.72 vs `FUSED_STRESS_MAX_PEAK_OVER_IDS_ONLY_BUILD_RATIO = 4.5`; full materialization lands ~10), curves-resident build 8.458 GB (fused fraction 0.429 vs `FUSED_STRESS_MAX_PEAK_FRACTION_OF_RESIDENT_CURVES_BUILD = 0.55`; any O(total)-curves route lands ≥ ~1.0) — the O(largest_chunk × num_steps × 4B)-not-O(total_edges) assertion from both sides. Tiny-floor fused timing band (ids-only default policy vs two-stage forced single-shot, same assembly both sides, warmup + best-of-3) green under the shared 1.25× band; fused raster-equivalence gate green at the small scenario (exact count agreement — the fused route rebuilds bit-identical geometry). New ids-only scenario workloads in `tests/_performance_regression_workloads.py` (`build_ids_only_scenario_hive_plot` + tiny/large/stress children): same generators/seed as the two-stage scenario, manual `BaseHivePlot` assembly (3 quantile groups + repeat axes, 9 chunks capturing every edge exactly once). **Matrix-level stress assertion: REJECTED per the cheapness bound** — a `HivePlotMatrix` peak-resident assertion cannot reuse the existing stress-scale workloads/measurements (a matrix child is a second stress-scale workload, multiplying cell renders); WS2's recorded rationale stands (identical per-cell streamed code path; the matrix forwarding equality test covers the wiring). **Render canary:** `TimeRenderMatplotlib` ASV benchmark (tiny + small; matplotlib is core, canaries the shared persisted-curves contract for all four vector backends; bokeh/plotly/holoviews deliberately NOT added) in `benchmarks/benchmarks.py`, verified via `make benchmark-dev BENCH=time_render_matplotlib` (tiny 210 ms / small 299 ms dev-mode); formal capture at merge; `benchmarks/results/benchmarks.json` picked up the new benchmark definition. Unit tests (default suite, `@pytest.mark.datashader`, 19 new + 3 unmarked hiveplot tests): fused-vs-two-stage parametrized over count/any/max/min (bit-exact) + sum/mean (rtol 1e-12) per the amended exactness set, NaN-metadata fused cases (count/sum/mean), ids-only single-shot equivalence (count/mean, bit-exact), discard-contract both sides, `_chunk_edge_curves` unit, mixed-persistence render, var/std ids-only carve-out (single-shot byte-equality + nothing persisted; policy unit test already pins that var/std can never reach the streamed combine at any threshold), all-empty-ids fallback branch, `_construct_edge_subset_curves` bit-equality vs `construct_curves` at num_steps 100/50 + missing-ids KeyError. Docstring notes on resident-memory characteristics added to `construct_curves` and both datashader entry-point `stream_chunk_threshold` docs; `runners/performance/README.md` updated (two standing 10M-edge gates; canary note); CHANGELOG entry added (new capability vs released 0.28: rasterize straight from stored edge ids). ruff format/check and ty clean; full default suite 1434 passed at 100% coverage; serial perf lane (`make test-performance`) 39 passed, 0 skipped, 0 failed.
- 2026-07-04: Workstream 3 touch-up complete (disposition items 2-6). **(Item 2) Vector-backend ids-only warning:** new sibling helper `unconstructed_curves_check(hive_plot)` in `src/hiveplotlib/viz/input_checks.py` (plain `warnings.warn(..., stacklevel=3)` matching the module convention, no custom class; at most one warning per render call, firing when any `(from, to, tag)` record holds non-empty `"ids"` and no `"curves"`, naming `construct_curves()` and the `hiveplotlib.viz.datashader` alternative); called by the four vector `edge_viz` functions (matplotlib/bokeh/plotly/holoviews) immediately after `input_check` — the datashader exemption is structural (its entry points never call the helper; the shared `input_check` is untouched, so `viz/datashader.py:568` cannot fire it). Tests: matplotlib `pytest.warns` parametrized over fully-ids-only and mixed states (exactly-one-warning asserted, datashader named in message) plus a no-warn render seeding persisted-curves, empty-ids, and kwargs-only records under `simplefilter("error")`; one `pytest.warns` test each for bokeh/plotly/holoviews under their markers; datashader ids-only no-warn on both routes (threshold 0 and default) under `simplefilter("error")`. Suite sweep confirmed no existing render newly raises under `filterwarnings = error`. CHANGELOG **Changed** entry (alters released 0.28 silent edge-less-figure behavior). **(Item 3) Doc folds:** mixed-state clause ("subsets with persisted curves render as stored, including non-default geometry; the rest build at defaults") added to both entry-point `stream_chunk_threshold` docs and mirrored on `_chunk_edge_curves`'s docstring; rebuild-per-render + `construct_curves()`-once sentence appended to the wrapper's `stream_chunk_threshold` doc; `_streamed_edge_raster`'s `chunk_axis_pairs` param doc corrected to "at least one stored edge id". **(Item 4) Ambiguous-tag ValueError fix** at `hiveplot.py` both sites (`add_edge_curves_between_axes`, `add_edge_kwargs`): first message line now reads `{axis_id_1} -> {axis_id_2}` over `hive_plot_edges[axis_id_1][axis_id_2].keys()`; two new tests pin both directions appearing in the message at both call sites; CHANGELOG **Fixed** subsection added (released 0.28 bug). **(Item 5) 4.5-constant comment** extended with the false-fail drift vector (numerator-only datashader/matplotlib import stack) and the trigger (re-measure both children before any retuning). **(Item 6) Golden-value smoke** landed as one small test, no machinery needed: `test__construct_edge_subset_curves_pinned_golden_values` (2 hand-picked A->B edges, `num_steps=4`, `use_numba_curves=False`, hardcoded float32 literal, `assert_allclose` rtol 1e-5 / atol 1e-6 tolerating platform-level float variation while catching gross anchoring/direction regressions). ruff format/check and ty clean; full default suite 1444 passed at 100% coverage; perf lane not re-run (the only perf-lane touch is the comment-only constant annotation; no executable perf-lane code changed).
- 2026-07-04: Workstream 4 complete. Narwhals at the input boundary shipped. **Pin-time checklist (Gate Zero, run before any code moved):** (a) narwhals 2.23.0 publishes exactly one `py3-none-any` wheel + sdist, no platform/compiled variants, zero mandatory runtime deps; (b) narwhals ships `py.typed`, ty 0.0.20 probe with `IntoDataFrame` clean, final `make ty` green; (c) **bound RE-DERIVED: the plan's `>=1.x,<2` is stale — narwhals's live major in July 2026 is 2 (latest 2.23.0; 1.x ended at 1.48.1 mid-2025) — pinned `narwhals>=2.0.0,<3.0.0`**, floor not assumed but VERIFIED by running all 43 polars-marked tests against `narwhals==2.0.0` (green), then restoring 2.23.0; polars 1.42.1 installs cleanly beside pandas 3.0.3 (no downgrade); no deprecation warnings across the probed narwhals/polars API surface under `-W error`. **Architecture:** new `src/hiveplotlib/frames.py` boundary-helper module — `as_eager_frame` is the single engine-detection predicate (legible `TypeError` naming the received type for lists / delayed objects / lazy frames, replacing the `isinstance(val, (pd.DataFrame, np.ndarray))` guard per the promoted WS5 done-when), plus `eager_frame_to_pandas` (column-wise numpy materialization, deliberately NOT `to_pandas()` so the polars path needs no pyarrow), `with_appended_column`, `native_frame_library`, `frame_to_pandas_copy`, `normalize_partition_labels_for_library`, `hashable_columns`. Pass-through semantics with a pandas `isinstance` fast-path at every site: pandas bypasses `nw.from_native` because narwhals raises `DuplicateError` on duplicate-column pandas frames and only pandas honors the `Hashable` (non-str) column-name contract — verified degenerate pandas inputs still accepted, so "pandas path unchanged" holds literally. **Surfaces:** `NodeCollection.__init__` (narwhals clone; uid-from-row-numbers fallback for index-less libraries; dup detection via `is_duplicated`), `check_unique_ids` (`n_unique`), `copy`, `create_partition_variable` — **cut/qcut disposition: binning ALWAYS runs through pandas `cut`/`qcut` on the column's numpy values (unconditional, not a per-engine fallback — this was already the pandas implementation), and the partition column converts back to the user's frame library before assignment; `labels=None` Interval bins store as strings on non-pandas engines (polars holds Intervals only as an unfilterable Object column), documented in the docstring**; `Edges._validate_edge_data` (native frames stored per tag; ndarray→pandas exception preserved incl. inside dicts), `add_edges` (`nw.concat` vertical + legible mixed-library `TypeError`), `export_edge_array` (`select().to_numpy()`); `NodeCollection.data` converted to a property (setter kept for `graph_features` writes) and round-trip contract docstrings on both `data` properties per the naming-audit sketch (ndarray→pandas exception named); `HivePlot`: new `_partition_node_frame` (pandas `groupby` fast-path; other engines get sorted-ascending keys + null-dropped + order-preserving filters, matching pandas groupby semantics), `place_nodes_on_axis` is the documented pandas seam (`Axis.node_placements` stays pandas-backed for the geometry pipeline; non-pandas node frames materialize there column-wise), `set_axes_order` collapsed-column write via `with_appended_column`, error messages via `hashable_columns`; **`add_edge_ids` needed no edit** (it operates on the numpy `export_edge_array` output + pandas placements — the `to_numpy()+np.isin` route the plan allowed — polars equivalence pinned by test); datashader `_edge_chunk_metadata_values` re-expressed through narwhals as a positional gather (single column `to_numpy()` indexed by the stored integer row positions — no label semantics), node paths (streamed per-axis chunks + single-shot diagonal concat) via narwhals select/cast with the WS3 fused seam untouched. **Typing:** `IntoDataFrame` on input parameters; engine-dependent returns are `Any` (annotating returns as `IntoDataFrame` produced 259 ty diagnostics across src+tests; house `# noqa: ANN401` precedent followed; newly-unused `ty:ignore` comments swept). **Fold-in calls (both implemented, symmetric):** `frame_library` property + `to_pandas()` on BOTH `NodeCollection` and `Edges`; `Edges.frame_library` raises a legible `ValueError` on mixed-library tags; `Edges.to_pandas()` mirrors the `data` property's single-vs-dict shape; public `to_pandas()` uses the engine's own converter (may need that engine's interchange deps, documented) while the internal geometry seam stays dependency-free. **Equivalence gate (`tests/polars_test.py`, `polars` marker only):** polars-vs-pandas twins on identical data are BIT-EXACT (`np.testing.assert_array_equal`, no tolerance needed) for node placements (values+dtypes), `relevant_edges` membership, stored ids, constructed curves, edge rasters (count; `ds.mean("weight")` single-shot + streamed; `ds.sum` streamed; fused ids-only streamed mean), and node rasters (count; `ds.mean` single-shot + streamed); missing-metadata `ValueError` fires on polars too. **Test matrix:** polars cases for all seven done-when methods plus collapsed-axis, `hp.plot()` matplotlib render, copy/repr/subset; `polars` marker registered in pytest.ini; engine-parametrized cases use `pytest.param("polars", marks=pytest.mark.polars)`; isolation verified by collection: `-m polars` = 43 tests, `-m datashader` = 93 with zero polars-marked cases. **Config/docs:** narwhals into core deps, new `[polars]` extra + `dev` inclusion (test-dep home per the optional-backend pattern), CLAUDE.md design note (frames.py bullet + pandas trip-wire amendment), CHANGELOG Added entry. **Validation:** ruff format/check + ty clean; full default suite 1504 passed at 100% coverage; serial perf lane 39 passed / 0 skipped / 0 failed; render canary `make benchmark-dev BENCH=time_render_matplotlib` tiny 211 ms / small 203 ms (WS3 dev-mode reference 210/299 — in band). **Plan holes surfaced for amend-plan routing (not swept, per Holdouts):** the Holdout premise "vector backends do not see user dataframes" is false for column-name `edge_viz_kwargs` gathers (`_data[tag][val].iloc[idx]` at matplotlib/bokeh/plotly/holoviews + `to_json`'s kwarg resolution + `to_json`'s `nodes.data.set_index`), which break on polars edges when kwargs name metadata columns; `BaseHivePlot.add_nodes` merging a second `NodeCollection` still `pd.concat`s; `to_networkx`/converters on polars-built plots remain pandas-only (listed Holdout).
- 2026-07-04: Workstream 4 boundary-gap follow-up complete (disposition items 1-2). **Three new `frames.py` helpers, each with the pandas `isinstance` fast-path inside:** `column_values_at_positions` (single column `to_numpy()` indexed by stored integer positions — the `_edge_chunk_metadata_values` precedent; pandas branch is the verbatim old `[val].iloc[positions].to_numpy()`), `rows_at_positions_to_pandas` (positional row gather materialized into an internal pandas frame for plotting consumption; pandas branch is plain `.iloc[positions]`, other engines build column-wise through numpy — chosen over per-column lists because plotly's `.iloc[i, :]` row extraction carries pandas' cross-column dtype promotion, which the serialized-figure equality gate demands bit-for-bit), `column_values_for_keys` (unique-key equality gather for `to_json`'s id-keyed node kwarg resolution; pandas branch is the old `set_index().loc[keys, column]`). **Seven sites converted:** `viz/matplotlib.py` kwarg gather → `column_values_at_positions` (+ membership check via a per-tag `hashable_columns` hoist); `viz/bokeh.py` CDS gather and `viz/holoviews.py` metadata gather → `rows_at_positions_to_pandas(...).to_dict(orient="list")` (holoviews membership check also hoisted); `viz/plotly.py` kwarg gather → `column_values_at_positions`, hover customdata → per-chunk `rows_at_positions_to_pandas` hoist + `.iloc[i, :]` per edge (provably identical to the old full-frame `.iloc[positions[i], :]` on pandas); `hiveplot.py` `to_json` kwarg resolution → `column_values_at_positions(...).tolist()`; `to_json` node kwarg resolution → `column_values_for_keys`. **Eighth site discovered and converted (surfaced, not swept): `viz/plotly.py` hovertemplate column-position lookup** (`np.where(df.columns == from_column_name)` at the old `:645-652`) — polars `.columns` is a plain list, so `list == str` collapses to scalar `False` and the lookup IndexErrors before customdata is reached; the mandated hover twin test cannot pass without it; converted to `hashable_columns(...).index(...)` (first-match, behavior-identical on pandas). The bokeh/holoviews hover tooltip `.columns` iterations work on polars as-is (list iteration) and were left untouched. **`add_nodes` through the boundary:** mirrors `Edges.add_edges` (pandas-on-pandas fast-path preserved verbatim; `nw.concat(..., how="vertical")` keeping the new-nodes-first order; legible mixed-library `TypeError` naming both libraries via `native_frame_library`); docstring gains the `:raises TypeError:` line. **Tests (+14, all polars-marked where polars runs):** `tests/polars_test.py` styled-render twin battery mirroring the WS1 non-RangeIndex assertion shapes — matplotlib per-collection linewidths, bokeh per-glyph CDS contents entry-by-entry, plotly serialized figure equality incl. hover customdata, holoviews per-element vdim values, `to_json` string equality (edge kwarg + node kwarg naming metadata columns) — all bit-exact pandas-vs-polars; `tests/hiveplot_test.py` `add_nodes` polars incremental add (stays polars, merged content matches the pandas twin per column) + mixed-library legible-error case; `tests/frames_test.py` engine-parametrized units for the three helpers incl. a pandas positional-vs-label-index divergence case. **Validation:** ruff format/check + `make ty` clean (one `ty:ignore[too-many-positional-arguments]` on the test's `hv.extension` call — holoviews' param-typed class trips ty under a direct import, which `viz_holoviews_test.py` never sees because it uses `pytest.importorskip`); full default suite 1518 passed at 100% coverage (from 1504); serial perf lane 39 passed / 0 skipped; render canary tiny 191 ms / small 212 ms (prior dev refs 210/299 and 211/203 — in band); marker isolation re-verified (`-m polars` = 53, `-m datashader` = 93 with zero polars overlap). No CHANGELOG entry (refinement to the polars support debuting in this unreleased version).
- 2026-07-04: WS4 polars gallery notebook complete. `examples/creating_hive_plots_from_polars.ipynb` (gallery role, instructional polish): construction from polars node/edge frames via `pl.from_pandas` over the `example_*` loaders (honest stand-in note in prose), round trip shown via `type(nodes.data)` + `frame_library` on both classes, `create_partition_variable` landing on the polars frame, default render, column-mapped node color from the polars `low` column (magma + labeled colorbar; self-validating against the axis sort), required pandas/pyarrow/modin one-liner. Registered in both gallery-index blocks (pandas, polars, networkx order); llms.txt entry added under Optional (author's call: new data-source capability, both siblings carry entries). Executed end-to-end on the repo kernel, no warnings/errors. Handed back for routing (outside the author's file set): CHANGELOG Documentation entry (drafted), thumbnail + conf.py wiring, llms.txt install-extras line. Editorial-critic + viz-critic + qa pending.
- 2026-07-04: WS4 polars gallery notebook revision complete (editorial-critic + viz-critic findings + disposition item 8(b)). `examples/creating_hive_plots_from_polars.ipynb` (gallery role, instructional polish), eight items landed: (1) labels=None divergence sentence in the Create Partition Variable cell (string bin names sort alphabetically on non-pandas frames, axis order can differ from pandas at multi-digit bin bounds, explicit `labels` controls it); (2) uid pitfall line where the `pl.from_pandas` stand-in is introduced (permuted pandas index dropped, valid-looking default unique IDs can silently name different rows, pass `unique_id_column` explicitly); (3) no-own-page clause scoped to pyarrow/modin (pandas unlumped, next cell links its page); (4) to_pandas colon promise fixed by reordering so the cell's colon delivers `frame_library` (cheaper than adding a code line); (5) "two-column" reworded to "a minimum of two columns" matching the pandas sibling; (6) extras lead-in softened (core Hiveplotlib suffices when data already lives in polars; the extra installs polars for this notebook's environment); (7) thumbnail wired: magma styled figure exported to `docs/source/_static/creating_hive_plots_from_polars.jpg` (jpg, sibling-matching) + `nbsphinx_thumbnails` entry in `docs/source/conf.py` between the pandas and networkx siblings; (8) CHANGELOG 0.29 Added -> Documentation entry added (0.28 gallery-notebooks precedent). Re-executed end-to-end on the repo kernel: no error outputs, no stderr, strict JSON valid, all code lines <= 79 chars. llms.txt untouched per brief (install-extras line batched to plan-end). qa still pending for WS4 notebook surface.
- 2026-07-04: WS4 post-impl touch-up complete (disposition items 1-7). **(Item 1) `.columns` sweep finished, seven sites through `hashable_columns`:** node-styling membership checks hoisted per-axis at `viz/matplotlib.py` + `viz/plotly.py` node_viz (matching edge_viz's per-tag hoist pattern); node-hover column lists at `viz/plotly.py` hovertemplate, `viz/bokeh.py`, `viz/holoviews.py`; edge-hover column lists at `viz/bokeh.py` + `viz/holoviews.py` (hover tag hoisted to a local). Bokeh gained the `hashable_columns` import. Sweep verified: remaining `.columns` reads in `viz/` are all internally produced frames (`node_placements`, `get_hover_axis_metadata` output, `__sanitize_dataframe_columns`); `hiveplot.py` has zero raw `.columns` reads. **(Item 2) datashader pandas fast-paths restored:** `_edge_chunk_metadata_values` re-routed through `hashable_columns` + `column_values_at_positions` (pandas fast-path inside each); the `_streamed_node_raster` per-axis wrap and the `datashade_nodes_mpl` single-shot `nw.concat` reverted to the pre-narwhals pandas idioms verbatim (node_placements is the documented always-pandas seam) with a comment naming the guarantee; `narwhals`/`as_eager_frame` imports dropped from `viz/datashader.py`. **(Item 3) degenerate-pandas regression tests (+8):** `_degenerate_pandas_hive_plot(flavor)` fixture (duplicate unused column via `pd.concat`, int-named column) through `datashade_edges_mpl` `ds.mean("weight")` and `datashade_nodes_mpl` count, each parametrized over default single-shot and threshold-0 streamed routes; sensitivity verified out-of-tree — narwhals raises `DuplicateError` on the dup-column fixture (the old wraps would fail these tests), while narwhals 2.23 tolerates int-named pandas columns, so that flavor rides as a contract pin rather than the breach proof. **(Item 4)** consequence sentence appended to the `create_partition_variable` note (string labels sort lexicographically; axis order can differ at multi-digit bounds; pass explicit ``labels``). **(Item 5)** `test_create_partition_variable_default_labels_axis_order_diverges_on_polars`: cutoffs `[5, 20]` twins, pandas axes `["(-inf, 5.0]", "(5.0, 20.0]", "(20.0, inf]"]` vs polars `["(-inf, 5.0]", "(20.0, inf]", "(5.0, 20.0]"]` asserted literally. **(Item 6)** `set_axes_order` non-pandas collapse branch stringifies the collapsed column AND the axis names it feeds (pandas branch untouched); docstring note added; `test_set_axes_order_collapsed_axis_polars_int_partition_values` pins int labels `[1, 2, 3, 4]` + `axes_order=[1, 2, None]` landing as string axes `["1", "2", "Other"]`. **(Item 7)** `as_eager_frame` TypeError branches on the `` `Edges` `data` `` context prefix: Edges contexts advertise the (n, 2) numpy-array acceptance and drop "arrays" from the unsupported list; other contexts unchanged; parametrized message test added. **(Items 8-10)** `test_hive_plot_matrix_from_partition_polars_matches_pandas` (cell-by-cell node-placement equality vs the pandas twin, green as predicted) in `tests/polars_test.py`; direct `hive_plot.add_edge_ids(edges=..., tag="direct")` calls added to `test_add_edge_ids_polars_matches_pandas`'s body on both twins (test-name contract closed); CLAUDE.md marker list gained `.polars`; `pyproject.toml` `[polars]` extra gained the narwhals-only/test-vehicle comment line. No CHANGELOG entries (all refine the unreleased polars feature). Validation: ruff format/check + `make ty` clean; full default suite 1531 passed at 100% coverage (from 1518, +13 new tests); serial perf lane 39 passed / 0 skipped / 0 failed; render canary tiny 187 ms / small 196 ms (prior dev refs 210/299, 211/203, 191/212 — in band); CRLF sweep clean.
- 2026-07-04: WS4 qa release-readiness pass, narwhals-floor re-run per the post-impl disposition item 9(d). **Floor re-verified against the full battery:** `narwhals==2.0.0` installed into the venv and the complete `-m polars` battery run — **56 passed / 0 failed** (the follow-up-era 53 plus the touch-up's three polars-marked additions), so the measured `>=2.0.0` floor holds for the full surface, not just the 43 pre-follow-up tests; no `pyproject.toml` change needed. `narwhals==2.23.0` then restored and `-m polars` re-run green (56/56), confirming the working env matches the pin. qa auto-fixes this pass: CHANGELOG 0.29 Input Data entry compressed to the four-wrapped-line cap (round-trip contract, core-dep note, pandas-unchanged clause, and new members all preserved); `docs/source/conf.py` `nitpick_ignore` gains `("py:class", "IntoDataFrame")` (narwhals publishes no objects.inv, so the alias cannot be intersphinx-linked; clears the five WS4-introduced docs-build warnings, rebuild re-verified at zero warnings).
- 2026-07-05: Workstream 5 complete (code + tests + local GPU verification; notebooks are the separate notebook-author dispatch). **Boundary (Path A, zero parameters):** new `frames.as_frame` is the single detection point — eager frames from every narwhals engine plus lazy Dask frames (`nw.LazyFrame`, implementation `DASK`); other lazy engines (e.g. `polars.LazyFrame`) and non-frames keep the legible `TypeError`; `as_eager_frame` now delegates to it and rejects lazy input with an operation-needs-in-memory message, so every WS4 eager operation degrades legibly. `is_lazy_frame` / `frame_row_count` helpers added; `NodeCollection` (requires explicit `unique_id_column` for lazy input; lazy uniqueness check via one `n_unique`/`len` collect), `Edges` (lazy validation via column metadata; lazy `add_edges` concat; compute-free reprs; `export_edge_array`/`to_pandas` are documented deliberate materializations), `_partition_node_frame` (lazy unique-values collect + lazy sub-frames), and the `place_nodes_on_axis` pandas seam (per-axis collect) all lazy-aware. **Non-materializing membership (extends WS1's structure at the same site):** `__store_edge_ids` stores a new `frames.LazyEdgeSubset` (native lazy frame + narwhals membership predicate `from.is_in(axis1_ids) & to.is_in(axis2_ids)`; `is_in([])` is the empty-axis case) at BOTH sites (`hive_plot_edges[...]["ids"]` and `relevant_edges[...]`) for lazy input — no per-edge id or position array ever materializes; `add_edge_ids` routes lazy `Edges` through new `_add_edge_ids_lazy` (mirroring the eager direction/tag flow), with mixed lazy+eager multi-tag input rejected legibly rather than silently materialized. Lazy subsets never persist curves: `HivePlot` construction stores ids-only (connect_axes skips the curve build), `construct_curves`/`_construct_edge_subset_curves` raise pointing at datashader-or-`.compute()`, `to_json` raises legibly, vector backends fire the existing ids-only warning (wrapper counted as undrawn ids in `unconstructed_curves_check`). **Rendering (ecosystem delegation, no hand-rolled partition loop):** `_rasterization_route` extends the ONE shared policy — lazy input always routes `"dask-native"` (threshold not consulted; documented in both entry-point docstrings): `_dask_native_edge_raster` turns each subset into a lazy curve-points frame (`map_partitions` over the filtered native frame; per-pair builder factory calls the geometry core `_curves_from_id_pairs`, extracted verbatim from `_construct_edge_subset_curves`, at default num_steps=100), lazily concatenates, and hands ONE Dask frame to `cvs.line`, so datashader's own Dask pipeline materializes/aggregates one partition at a time and combines partials for EVERY reduction — **var/std delegation verified at pinned datashader 0.19.1** (probe + default-suite tests: count/any/max/min bit-exact vs pandas twin, sum/mean/var/std within 1e-9). One real upstream quirk found and neutralized: datashader's Dask line pipeline stitches a spurious segment across an EMPTY interior partition (reproduced minimally, 1-pixel divergence); every partition curve frame now leads with a `[NaN, NaN]` sentinel row (behavior-free; empirically restores bit-exactness; metadata columns promote to float via the sentinel). **Failure-point reraise:** the dask-native compute region wraps in new `exceptions.DaskComputationError` with repartition/partition-size guidance, `raise ... from err` (poisoned-partition test pins message + chained cause). **cuDF:** eager narwhals path as-is, plus `_collection_frame_backend`/`_collection_frame` — cuDF edge input gets cuDF transient collection frames on the single-shot AND streamed routes (CUDA path engages; `nw.from_dict(backend="cudf")`, no cudf import in src), everything else stays pandas verbatim; `_to_host_array`/`_raster_to_numpy` handle cupy-backed aggregates (fake-cupy unit tests cover the arm on CPU; all CPU-exercisable branches in the default suite at 100% coverage — no GPU-only src lines). **GPU verification (RTX 5070, `~/venvs/hiveplotlib-gpu`, cudf-cu12/dask-cudf 26.6.0, cupy 14.1.1, rmm 26.6.0, datashader 0.19.1, pandas 2.3.3; venv gained narwhals 2.23.0 + pytest stack — it predated WS4 — and the editable install refreshed):** `make test-gpu` (new target, `pytest -m cudf -n 0 --no-cov` against `GPU_VENV`, default `~/venvs/hiveplotlib-gpu`, with a legible missing-venv error) — **14 passed**: round trip, count/any/max/min bit-matched to the pandas twin on both routes through the CUDA path, RMM VRAM tracking, moment-reduction raise-or-match battery. **Measured per-process VRAM peak (hard gate): single-shot count render 8,155,004 bytes (8.16 MB), streamed 16,315,420 bytes (16.32 MB)** on the 2k-edge smoke — measured via **RMM allocation statistics** (`rmm.statistics`), NOT the plan's pinned cupy mempool, because the cupy default pool **measured exactly 0 bytes** for the whole workload: datashader's CUDA aggregates allocate via numba (cupy views over foreign memory) and cuDF allocates via RMM, so the cupy pool never sees this workload's allocations — RMM statistics is the per-process, exact, RAPIDS-native tracker with the same isolation rationale the cupy-mempool pin carried (whole-device sits at 1.28 GB of context/libraries, confirming the nvidia-smi-noise concern). **Metric deviation flagged for amend-plan routing.** Second stack finding for the record: `ds.sum/mean/var/std` over cuDF FAIL NVVM kernel compilation on this stack (datashader `_sum_zero._append_cuda`, numba-cuda 0.28.2, sm_120) — the smoke pins raise-not-mis-render; count/any/max/min are the GPU-supported set at this pin. GPU-only `NumbaPerformanceWarning` (small smoke grids) added to the pytest.ini ignore list. **Instrument selection (Dask, recorded per the 2026-07-03 amendment): tier-1 `measure_peak_rss`.** Deciding fact verified in-env: `dask.base.get_scheduler(collections=[ddf]) is dask.threaded.get` — the default local scheduler for dataframe work is threaded (one process), so single-process exact RSS captures the whole computation; Dask-native MemorySampler is distributed-only; tier-2 is unneeded for this route (bears on the chain-close tier-2 pruning decision: WS5's profiling practice does NOT use tier-2). Perf lane gains `build_small_dask_scenario_hive_plot`/`build_and_rasterize_small_dask` (small twin, 2 partitions) + two gates: tier-1 instrumentation check (instrument stated in docstrings + runner README) and `assert_raster_equivalent` dask-vs-pandas at the small scenario (exact count agreement, green). **Scheduler/hardware disclosure:** all recorded Dask numbers here are threaded-scheduler, local-machine (canonical host) results, not representative of distributed hardware. **Tests:** `tests/dask_test.py` (38, `dask` marker; marker isolation verified by collection: `-m dask`=38, `-m datashader`=116 with zero dask/cudf overlap, `-m cudf`=14) covering boundary units, structural non-materialization, membership-content equality vs pandas twin per subset, the ~10k-edge 2-partition e2e over 8 reductions, threshold-ignored, partition-count invariance (1 vs 4), empty-partition-with-metadata, single-subset manual BaseHivePlot flow, unplaced-axis empty subset, reraise, all legible-error surfaces, repeat-axes route, deepcopies; `tests/cudf_test.py` (14, `cudf` marker, GPU-gated, skips cleanly without cudf); `tests/datashader_test.py` +5 route/backend/host-array units. **Validation:** ruff format/check + `make ty` clean; full default suite **1584 passed / 14 skipped (cudf off-GPU) at 100% coverage**; serial perf lane **41 passed / 0 skipped** (all WS2/WS3 stress+timing gates green — no regression on non-Dask paths); render canary tiny 178 ms / small 203 ms (prior dev refs 187-211 / 196-299 — in band); CRLF sweep clean. CHANGELOG Added entry (4 wrapped lines). CONTRIBUTING.md gains the "GPU Verification" section (venv setup incl. the pandas<3 constraint, `GPU_VENV` override, release cadence, skipped-run-on-GPU-machine warning). **Notebook facts for the author:** (1) partition-sizing ceiling — each in-flight partition transiently holds ~`partition_rows * (num_steps + 1) * 8` bytes of float32 curve x/y (~808 bytes/edge at default num_steps=100) PLUS its metadata column, and the threaded scheduler runs up to CPU-count partitions concurrently, so the honest guideline scales inversely with thread count (e.g. ~1-2M-edge partitions on a 32 GB machine with 8+ threads; a single 10M-edge partition is ~8 GB before concurrency); density-corrected (non-count) reductions make a SECOND full pass over the lazy curve builds. (2) `place_nodes_on_axis` shuffle disposition: RESOLVED as no-shuffle — there is no Dask sort inside hiveplotlib; each axis's node subset materializes into the internal pandas placements table (one lazy filter pass per axis) and geometry runs in pandas, so "sort upstream" is unnecessary and the real constraint is node data fitting in RAM (nodes are the in-memory side; edges are out-of-core). (3) install story `pip install hiveplotlib[datashader]` (carries `dask[dataframe]`); no new extra. (4) cuDF page must name the cudf-cu12 pandas<3 pin (dedicated env, per CONTRIBUTING) and that GPU reduction support at this stack is count/any/max/min (sum/mean/var/std raise upstream). (5) Dask plots are ids-only by design: datashader renders; vector backends warn; `construct_curves`/`to_json` raise with guidance; curves rebuild per render. **Open for routing:** the VRAM-instrument refinement (cupy mempool → RMM statistics, forced by measurement) and the upstream cuDF float-sum NVVM failure are surfaced for amend-plan/critic disposition, not silently absorbed.

- 2026-07-05: Workstream 5 gallery notebooks complete (notebook-author). Two new gallery pages in "Hive Plots from Different Data Sources" (registered in both index.rst blocks, order pandas -> polars -> dask -> cudf -> networkx). **`examples/creating_hive_plots_from_dask.ipynb`** (executes end-to-end clean under the repo kernel; only stderr output is the deliberate `construct_curves` TypeError demo, house precedent): zero-ceremony construction from an explicitly partitioned in-memory synthetic (`dd.from_pandas(example_edge_data(...), npartitions=4)`, named as a stand-in for `read_parquet`); nodes kept pandas with a "Node Data Stays in Memory" section (lazy frames can't cut partition bins; nodes are the in-RAM side); datashaded render via `HivePlot(backend="datashader")` at 200 nodes / 10k edges; ids-only story (curves built and dropped per render; vector backends warn, `construct_curves` raises with guidance, demoed); "Partition Sizing" section carries the log's ceiling facts (~808 bytes/edge at default num_steps, times threaded-scheduler concurrency; guideline: 1-2M-edge partitions on a 32 GB / 8-thread machine, 10M-edge partition ~8 GB pre-concurrency; non-count reductions make a second pass) all framed as threaded-scheduler local-machine mechanics (Failure mode 3); `DaskComputationError` reraise-with-chained-cause named there; "No Dask Shuffle When Sorting Nodes" section states the no-shuffle resolution visibly; install story `pip install hiveplotlib[datashader]`. **`examples/creating_hive_plots_from_cudf.ipynb`** (all-markdown page: static fenced code blocks + captured outputs, per the GPU-doc precedent of markdown-only code for env-dependent paths, since the notebook must execute in CI/docs envs with no GPU and pandas 3; executes trivially clean): every shown output captured from a REAL run in `~/venvs/hiveplotlib-gpu` on the RTX 5070 (frame_library 'cudf', edges.data type, the rendered figure embedded as a jpeg attachment, and the live NvvmError from `ds.mean` — abbreviated honestly); both normative disclosures named as current-stack facts with version pins (cudf-cu12 26.06 pandas<3 dedicated-env story; GPU reduction set count/any/max/min with sum/mean/var/std raising from the kernel compiler, workaround `to_pandas()`); no perf claims on either page. **Thumbnails:** `docs/source/_static/creating_hive_plots_from_dask.jpg` (real Dask-pipeline render, text-free) and `creating_hive_plots_from_cudf.jpg` (real GPU render, text-free, `cmap_edges="Purples"` thumbnail-only to orthogonalize from the Dask neighbor per the viz-quality-bar thumbnail rule); both registered in `conf.py` `nbsphinx_thumbnails`. **llms.txt:** both pages judged real new data-source capabilities (out-of-core, GPU) -> two entries added to the data-sources block under `## Optional`, gallery order. CHANGELOG: two Documentation entries matching the polars precedent. No `use_dask` anywhere; no process refs in shipped prose.
- 2026-07-05 (same-day follow-up, notebook-author): coordinator-relayed critic-disposition constraints verified against the shipped notebooks. (1) No VRAM figures appear on either page (scanned for the suspect 8.16/16.32 MB pair, `VRAM`, `RMM`; zero hits) — the cuDF page was already written without memory numbers, so nothing was removed. (2) H1 titles pinned exactly as required: `# Creating Hive Plots from Dask` and `# Creating Hive Plots from cuDF` (polars-sibling convention) — the exception-message doc-title alignment touch-up can target "Creating Hive Plots from Dask" verbatim.
- 2026-07-05 (micro-pass, notebook-author): four editorial/viz residuals closed on the WS5 notebooks. (1) Dask colon-promise reworded to promise `frame_library` (what the cell shows); notebook re-executed clean. (2) Both gallery thumbnails re-exported text-free via `show_axes_labels=False` (same data/palettes; cuDF one re-rendered on the GPU). (3) cuDF page's embedded attachments now dpi-150 jpegs matching the Dask page's inline sharpness. (4) Reduction section gains a real captured `ds.max("weight")` GPU render (second jpeg attachment) showing the supported column-based set working before the `ds.mean` raise demo. VRAM embargo re-verified (zero hits); strict JSON + nbformat + ruff clean on both.
- 2026-07-05: WS5 post-impl touch-up + GPU per-route re-measure complete (2026-07-05 post-impl disposition, items 1-5). **Per-route VRAM pair (hard-gate record, verified in isolation): single-shot 8,155,022 bytes (8.16 MB), streamed 16,315,438 bytes (16.32 MB)** on the 2k-edge smoke, each measured in its own isolated process via RMM allocation statistics; a fresh per-route RMM scope inside one process reproduces the original streamed figure (16,315,420 bytes), so the prior pair was NOT a shared-epoch conflation — the streamed 2.0007x is real and raster-buffer-dominated at smoke scale (the peaks decompose to exactly two vs exactly one ~8.16 MB aggregate-raster buffer, a ~1428x1428 4-byte count grid; streamed total_bytes 64.3 MB vs single-shot 10.4 MB reflects per-chunk churn; current_bytes returns to ~0 after each route). Gate records annotated in place (WS5 status line, done-when GPU sub-bullet, 2026-07-05 instrument amendment); `tests/cudf_test.py` VRAM test rewritten to push/pop a fresh `rmm.statistics` scope per route with per-route peak>0 / total>0 assertions (item 1c). **Item 2:** CONTRIBUTING.md GPU-venv install line now `[datashader,testing]` (pytest.ini addopts need the testing extra). **Item 3:** lazy-materialization consequence sentence added to the `NodeCollection.to_pandas`, `Edges.to_pandas`, and `Edges.export_edge_array` docstrings (deliberate materialization for an explicit conversion/export request). **Item 4 (code-side four of five; the Example 4 comment was executed in the amendment):** `_construct_edge_subset_curves`'s lazy TypeError gains "and rebuild the hive plot" (matches the `to_json` twin); both `frames._unsupported_type_message` branches gain the collect-first clause (e.g. `.collect()` on a `polars.LazyFrame`); `unconstructed_curves_check` is lazy-aware — it now scans all undrawn subsets and, when any is lazy, the single warning leads with the datashader remedy plus `.compute()`-and-rebuild and drops `construct_curves()` (a guaranteed dead end there), eager message unchanged, dask warning test updated to assert the lazy message and the absence of `construct_curves`; `_validate_edge_data` docstring rewritten to the shipped contract (`as_frame` named as the detection point, lazy-Dask acceptance, "lazy frame from another library" reject wording, `:raises:` corrected) and the `Edges` `:param data:` line gains the lazy-Dask acceptance. **Item 5 (all five):** int-weight parametrization landed as `_twin_input_data(int_weight=True)` + `test_datashade_edges_mpl_dask_int_weight_matches_pandas` over max/min (bit-exact) and sum/mean (rtol 1e-9) — PASSES, both routes produce identical float64 NaN-filled rasters, so no fill-divergence note is owed; sentinel mutation probe run (sentinel rows deleted locally, dask module re-run): exactly one test fails — `test_datashade_edges_mpl_dask_empty_partition_with_metadata` — now named in the sentinel comment in `_lazy_partition_curve_frame_builder`, sentinel restored exactly; `_dask_native_edge_raster` guards every chunk subset with an `isinstance(..., LazyEdgeSubset)` check raising the legible mixing TypeError for mixed lazy+eager same-tag subsets (separate `add_edge_ids` calls from different `Edges` instances), `:raises TypeError:` documented, pinned by `test_datashade_edges_mpl_dask_mixed_lazy_and_eager_subsets_raise`; the cuDF moment-smoke `except` now walks the exception chain and asserts an NVVM / kernel-compilation marker, so a hiveplotlib-side crash fails the smoke instead of passing as "unsupported"; the `DaskComputationError` reraise message quotes "Creating Hive Plots from Dask" (the shipping notebook's H1). No CHANGELOG entries (all refine WS5 work unreleased in this version). Validation: ruff format/check + `make ty` clean; full default suite 1589 passed / 14 skipped (cudf off-GPU) at 100% coverage; serial perf lane 41 passed / 0 skipped; `make test-gpu` 14 passed (incl. the rewritten VRAM and raise-origin tests); CRLF sweep clean.
- 2026-07-05: CI segfault on the Dask route diagnosed and fixed (commit `21040d2`, post-chain follow-up). CI-only failure (6 crashed xdist workers on the Dask equivalence twins + a hard segfault in the serial perf job) with identical package versions to the green local env; root cause: pandas 3's arrow-backed string paths are not thread-safe, and the threaded Dask scheduler raced them — every partition frame built a fresh arrow string column Index and every list-label `.loc` built more, dying on the low-core CI runner inside dask's meta-enforcement `Index.equals` (24-core local timing never lost the race, which is why five green gate batteries missed it). Fix: one interned column `pd.Index` shared by all partition frames and the meta per render (identity fast-path short-circuits enforcement), integer-keyed frame construction, single-column gathers in `build_partition_curve_frame` and `_curves_from_id_pairs`; instrumented worker-thread arrow constructions under `apply_and_enforce` dropped 396 → 0, outputs probed value-and-dtype-identical. Full suite 1589/100%, perf lane 41/0 post-fix. Residual (recorded): ~24 constructions per render remain in dask-expr's own projection layer (dask-internal path, not ours); if CI ever crashes there, it is an upstream pandas/dask report, not a revision of this fix.
