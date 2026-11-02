# Research plan: bounding memory on hiveplotlib's Dask rendering route

<!--
Research-plan shape (deliberately light per the plan template). No workstream or done-when
ceremony; conventions (shallow-panel dispatch, standing lenses, bounds, grounding, durable
landing) live in the research-track skill. This plan carries only the per-plan fill-in.
Consumer: hiveplotlib. Evidence mode: literature. Requested 2026-10-30. No spec (research run).
-->

## Question

How do rigorous Dask-integrated libraries size work, reduce partial results and share large read-only state so memory stays bounded, on one machine and on multi-machine clusters, and what should hiveplotlib's Dask rendering route do to match or knowingly differ?

The answer is a ranked set of options for the route. Each option states its effect on single-machine memory, cluster memory and transfer, render time, raster exactness and maintenance surface, marks each effect as `## Validation criteria` defines, and names which term binds at the questionnaire item 1 target: curve buffers, aggregates or the node table. The ranking states the effect order it used (questionnaire item 6). The run also gives a separate verdict on each part of the maintainer's framing (story 7).

**Bounds from the maintainer.** Nothing is filed or posted upstream: lenses read issue trackers and never write to them. No 1B-edge run for now. The constraints are a cluster path that is not hurt, a way of running that "can do lots of work in parallel and or be collapsing things whenever possible", and a real power-user path to billions of edges even if it is not the default. Their statements are explicitly not gospel, so the run may conclude they are wrong.

**Bounds from the dispatching session's brief, not the maintainer.** Lenses run no benchmarks. This run writes no consumer code. Throwaway probes outside the lenses are not barred by any maintainer ruling: probes (a) and (b) from the adversary's challenge ran on 2026-10-30 and are in `## Inputs`, and the rest are named proposals under `### Proposed experiments`.

## Inputs (attack these; they are not conclusions)

Checked 2026-10-30 against `~/repos/hiveplotlib-performance/synthetic_scaling/dask_partition_memory/RESULTS.md` and the installed source (datashader 0.19.1, dask 2026.7.1).

**Measured, flat combine:**

| Nodes | Edges | Partitions | Output partitions (6 axis pairs) | Render (s) | Total peak (GiB) |
| --- | --- | --- | --- | --- | --- |
| 2M | 10M | 1 | 6 | 32.2 | 5.18 |
| 2M | 10M | 10 | 60 | 45.7 | 2.55 |
| 2M | 10M | 100 | 600 | 116.4 | 7.54 |
| 2M | 100M | 10 | 60 | 111.3 | 12.48 |
| 2M | 100M | 100 | 600 | 211.6 | 8.59 |
| 20M | 10M | 10 | 60 | 520.7 | 11.28 |

8 threads, `num_steps=16`, `ds.count()`, 3 axes, uniform random edges from `dd.from_map` with no disk reads, one sample each, threaded scheduler, WSL. Build peak is 0.91 GiB at 2M nodes and 6.04 GiB at 20M. Render is all of `hp.plot`, so it includes drawing the nodes.

**Measured, tree combine, fan-in 8, one run each:**

- **Internal rewrite** (`tree_combine_probe.py`, wraps `glyph_dispatch`), threaded only. 2M nodes / 10M edges / 100 partitions: 2.18 GiB, 120.0 s, same lit pixels as flat. On a 300 x 300 canvas: count, any, max, min bit-exact; sum, mean, var, std differ by rounding only (largest 1.4e-13).
- **Public API** (`public_tree_probe.py`: `to_delayed`, `Canvas.line` on pandas, rasters merged through `dask.delayed`, `mean`/`var`/`std` rebuilt from additive `count`, `sum` and sum-of-squares rasters). Same case: 2.18 GiB, 107.4 s, same lit pixels. At 20k nodes / 400k edges / 40 partitions, threaded: the same four bit-exact, the same four rounding-close (largest 4.4e-16).
- **Multi-process.** `scheduler="processes"`, 4 workers, `ds.mean`, same 20k-node case: flat and tree both complete, differing by at most 5.6e-17. The builder's closure pickles across a process boundary at this size. No memory figure was reported.

**Read fairly.** Compare any tree to flat at its best partition count. At 10M edges, flat at 10 partitions is 2.55 GiB in 45.7 s and the tree at 100 is 2.18 GiB in 107.4 to 120.0 s: 1.17x on memory and 2.3x to 2.6x slower (a pointer, since partition counts differ). The 7.54 to 2.18 drop mostly removes a penalty sizing already avoids.

**Arithmetic on the table, not measurement.** Three terms fit: `peak = build + threads x partition rows x (num_steps + 1) x 8 bytes + 6 x partitions x 9.5 MiB`. The middle term is the notebook's budget formula. It gives 1.01 GiB at 1M rows per partition and 10.13 GiB at 10M, against residuals of 1.08 and 11.01 after subtracting build and aggregates. Misses: about 1 GiB over at 600 aggregates, under at the one-partition row (4.21 against 7.6 at six concurrent tasks), 3.67 GiB over at 20M nodes. Agreement is not mechanism: each output frame holds about a ninth of a partition's rows, so why whole-partition rows fit is unexplained.

Predictions from those terms, at 2M nodes, `num_steps=16`, 8 threads. None measured except where stated:

| Edges | Flat at its best partition count (GiB) | Tree at 1M rows per partition (GiB) | Tree gain |
| --- | --- | --- | --- |
| 10M | 2.4 at about 13 (measured 2.55 at 10) | 2.18 measured, at 100k rows per partition | 1.17x, measured |
| 100M | about 6 at about 43 (measured 12.48 at 10, 8.59 at 100) | about 3 | about 2x |
| 1B | about 17 at about 135 (`RESULTS.md`: 17 to 18 at 100) | about 3 | about 5x to 6x |

The sized-flat floor grows with the square root of edges and the tree's does not, so the 1.17x is one point on a curve. At 1B edges, flat at its best is about 28 GiB at `num_steps=50` (the signed default) and about 38 at 100 (the code's current default). 20M nodes adds about 9 GiB. The curve term is also linear in threads. The tree's time at 1B (6,000 build tasks) is not predicted.

**Route.** `_dask_native_edge_raster` in `src/hiveplotlib/viz/datashader.py` builds one lazy curve frame per (from axis, to axis) pair with `map_partitions`, concatenates them, and calls `cvs.line`. Its docstring bounds the peak by the largest partition's curves, and the Dask notebook's "Partition Sizing" says "more partitions only lower the ceiling". Both are false as stated, in a two-sided way: going from 10 to 100 partitions raises the 10M-edge peak (2.55 to 7.54) and lowers the 100M-edge peak (12.48 to 8.59).

**Datashader's line path.** `data_libraries/dask.py::line` makes one canvas-sized aggregate per output partition and feeds all of them to one `(combine, keys)` task. Each line task also reads the previous partition's whole curve frame to stitch segments. Curves here are NaN-separated, and the public tree rasterizes each partition with no stitching and got the same lit pixels.

**Read, not run:**

- **The tree path is one dispatch away.** Only `LineAxis0` is registered to the flat `line` function. Every other glyph, including the `LinesAxis1` family that `cvs.line(..., axis=1)` selects, falls to `default`, which uses `da.reduction` with `split_every=2` and no previous-partition stitching. Whether one row per edge is practical for this route is unknown. `default` runs an extra pass over partitions to compute lengths for reductions that use row indexes, and its source comments call the dataframe-as-array construction "dodgy" and "Here be dragons". See story 4.
- **Public tree, shipping form.** `public_tree_probe.py` handles eight reductions (count, any, sum, mean, max, min, var, std) and its `plan_for` returns `add` for anything else (line 76), so `first`, `last` or `by` would give wrong pixels with no error. That is a probe and not a shippable surface. The shipping form is an exact-type table in the style of `_STREAMED_REDUCTION_COMBINE_MODES` (`datashader.py:70`): listed reductions take the tree, and unlisted ones stay on flat `cvs.line`, which keeps the docstring's "every reduction" promise. That table's comment says var/std have "no exact per-chunk combine", so the option must choose: var/std stay on flat with their memory unchanged, or the policy moves to exact-to-rounding and answers `wiki/wiki/adr/0002-performance-regression-harness.md`. The moment rebuild is a second implementation of datashader's reductions and needs a named justification.
- **Node table.** The 20M-node row spends 520.7 s and 11.28 GiB on the same 60 aggregates (about 0.56 GiB) that cost 45.7 s and 2.55 GiB at 2M. Each build task runs `edge_ids.copy().astype("O")`, then two `merge` plus `sort_values` calls against an axis's whole `node_placements` (`_curves_from_id_pairs`, `hiveplot.py`). One serial call under `cProfile`, 1M edges: `merge` is 0.70 of 1.64 s at 667k nodes per axis and 6.26 of 7.95 s at 6.67M, while `sort_values` and `astype` stay flat. The merge is what grows inside a call. That does not attribute the row: the row's calls are about 111k edges each, `hp.plot` also draws 20M nodes, and the profile covers time and not memory. Sixty calls at 6.26 s is at most 376 s if the threads serialize (object-dtype keys may hold the GIL, unverified) and about 47 s if they do not, against 475 s of growth. The 3.67 GiB residual is unattributed. A precomputed lookup has not been tried; it would stay inside "nodes are in memory by design". 1B edges at the densest measured ratio of 50 edges per node means 20M nodes.
- **Render time grows with partition count** at fixed edges (32, 46, 116 s at 10M; 111, 212 s at 100M), and the tree did not touch it. Each build task runs the merges above. Not profiled at row scale.
- **The route has never run on `distributed`.** `distributed` is not installed in the project venv, `tests/dask_test.py` has no hit for `scheduler`, `processes`, `pickle`, `LocalCluster` or `distributed`, `src/` and `tests/` have no `LocalCluster` or `processes` scheduler use, and `hiveplotlib-performance` has only the 20k-node processes run above. The builder closes over the entire `hive_plot` object, node tables and lazy subsets included, so each build task ships it or shares it. That it pickles at 20k nodes is settled; what that costs at 2M or 20M nodes is not. The maintainer's "don't hurt the cluster case" may be protecting an untested path.
- **A flat combine may be worse on a cluster than on one machine.** Inferred from graph shape only. One task taking all aggregates means one worker receives them all after they cross the network: about 57 GiB at 1B edges in 1M-row partitions (6,000 aggregates), about 5.6 GiB in 10M-row partitions (600).
- **Expansion depends on `num_steps`.** Row to curve is 16 to 808 bytes (50x) at `num_steps=100`, 408 bytes (25x) at 50, and 136 bytes (8.5x) at 16. These assume int64 ids and no metadata column.
- **Aggregate size depends on the reduction.** The measured 9.5 MiB per aggregate is `ds.count()` at 1500 x 1500. Mean, var and std carry more state, and column reductions make a second full pass.
- **Fan-in differs.** The probes used 8; datashader's own `default` path uses 2 with the comment "less is more".

**Prior finding.** Tripling the Dask thread cap from 8 to 24 cost 64% more peak memory (4.202 to 6.892 GiB) with no detectable speedup, five samples each (`wiki/wiki/entities/hiveplotlib.md`).

## Prior rulings and records

Respect these, and name it explicitly if a recommendation would reverse one:

- `use_dask` was dropped (Path A, 2026-06-05): Dask is detected through narwhals with no opt-in parameter (`wiki/wiki/plans/scaling-large-networks.md`, `## Alignment (grill)` Wave 3 and the "Removed parameter" amendment).
- Narwhals pass-through at the input boundary with no canonical internal frame (`CLAUDE.md`, frames.py note).
- Nodes are in memory by design.
- Design note 2026-05-17 flagged the per-partition ceiling; its disposition was notebook guidance. The notebook guidance is now one of the things under attack.
- Prefer ecosystem tools over hand-rolled low-level code: bespoke infrastructure needs a named justification (standing maintainer preference).
- `wiki/wiki/adr/0002-performance-regression-harness.md` carries the equivalence-wall precedent a raster-changing option has to answer to.
- `wiki/wiki/specs/lower-default-curve-resolution.md` (signed 2026-08-31) moves the default `num_steps` from 100 to 50 and rejected 16 as a default on headroom. The code still says 100.

Record-keeper's pre-task pass is knowingly skipped for this run. The entries above come from the brief and from direct reads, and the adversary's wiki sweep for `split_every`, `to_delayed`, `from_map`, `LineAxis0`, `treeAggregate` and "flat combine" found no collision. Weak evidence, accepted because the producer path reads the wiki again at landing.

## Candidate stories / hypotheses

1. **Tree combine is standard and the fix.** Bounded fan-in is what rigorous libraries do. Measured against sized flat at 10M edges it gains 1.17x at 2.3x to 2.6x the time, so its case is the power-user path, where the gain is predicted to grow. It has to name where sizing is unavailable. The counter-evidence lens attacks this story.
2. **Sizing is the fix, and a library defers.** Partition size has an optimum. Correct the docs and error message, state an output-byte sizing rule, and leave the graph and the user's partitioning alone. Predicted to reach 1B edges on one machine at about 17 GiB at `num_steps=16`, about 28 GiB at 50.
3. **Fold, do not tree.** The route multiplies aggregates by axis pairs for no reason, or a per-worker or sequential accumulator holds fewer aggregates than any tree, as `xgboost.dask`-style collective models do. Fan-in is then the wrong knob.
4. **Datashader's own tree.** Switch to the `axis=1` glyph family. It keeps datashader's combine for every reduction with no table, at the price of a code path its authors call "dodgy" and a one-row-per-edge frame nobody has tried here.
5. **Own the pipeline.** Build per-partition rasters through `to_delayed` or `from_map` and merge them directly. The public probe demonstrates this at 2.18 GiB with 20k-node equivalence on eight reductions. Remaining costs: a second aggregation path, the shipping form's table and fallback, the var/std policy choice, and whether `to_delayed` on a concatenated frame is a stable surface.
6. **Collapse before expanding.** Group edges at pixel resolution per axis pair and draw weighted curves, so work is bounded by output resolution and not edge count. This is the most literal reading of "collapsing things whenever possible", and it gives up exactness against the current raster.
7. **The maintainer's framing is wrong.** Four separate claims, four separate verdicts:
   - **7a. Serialization outweighs tree shape.** Shipping the node table to every worker is the bigger cluster problem. Partly answered: it pickles at 20k nodes, cost at size unknown. Process-boundary lens.
   - **7b. Nobody renders graphs on a cluster.** By the absence rule this can only end as a search-scoped inconclusive, so the page predicts that up front and no voucher chases it. Counter-evidence lens.
   - **7c. Billions of edges are not worth drawing edge by edge.** A design judgment, argued from what large-graph tools chose. Prior-art lens, with story 6.
   - **7d. Partitions are the wrong unit to expose to this user.** Sizing lens.
8. **The node table is the limit.** At 50 edges per node, 1B edges means 20M nodes, the only measured row where the combine is a rounding error. A shared per-axis id lookup replaces the per-task merges and stays inside "nodes are in memory by design". The code change has no literature question in it; how a large read-only table is shared across workers does.

These are not mutually exclusive. A likely answer is a combination, with different options for the default and the power-user path.

## Failure-mode rubric

**Provenance: seeded by the orchestrator and extended from the adversary's plan-stage challenge, not elicited from the maintainer.** The research-branch grill that normally populates this was knowingly skipped (see `## Alignment (grill)`), so this is the model's standard and not the maintainer's until they say otherwise. Consumed by the adversary at convergence.

- **Result is a measurement artifact (first).** Every memory figure is one-sample process RSS on a threaded scheduler under WSL, except the prior finding's five samples. RSS high-water marks include allocator retention, so held-live memory and peak RSS can differ. Both tree probes report exactly 2.18 GiB though only one keeps the stitching read, and about 0.7 GiB of that is unexplained, so something else may set the floor. A finding that treats the partition-count term or any tree gain as established outside that setup, or rests on them without saying so, is killed.
- **Workload artifact.** Every row is uniform random edges with no disk reads ("the friendly case for axis-pair skew"), at 3 axes, `ds.count()`, 1500 x 1500, `num_steps=16`. The signed default is 50, which triples the curve term and leaves the aggregate term alone, so the optimum moves. A finding extended to skewed or Parquet-backed data, more axes, column reductions or another `num_steps` says so.
- **Wrong baseline.** Every option is compared to flat at its best partition count, never at 100 partitions of 10M edges. A tree names the case where sizing is unavailable: a user who cannot repartition, a skewed source, a worker limit below the sized floor, or an edge count where that floor exceeds the target hardware.
- **n too small, uncontrolled comparison.** One sample per row, and rows differ in more than the named variable (the 1.17x compares different partition counts). Such a comparison is a pointer and cannot carry a recommendation alone.
- **The analogy does not transfer.** Most Dask reductions combine small state: scalars, or groupby state bounded by group count. This route combines a fixed canvas-sized raster per output partition, after an 8.5x to 50x row expansion, with a read of the neighboring partition. A cited practice that does not state its own state size and input-to-output ratio does not count as support here.
- **Scope laundering across schedulers.** Threaded, single-node multi-process and cluster behave differently (shared memory versus transfer, GIL, per-worker limits, spilling). Every finding carries a scope tag. Threaded or processes evidence presented as a cluster claim, or cluster advice that worsens one machine and does not say so, is killed. Nothing in this run is cluster-measured, so the only permitted cluster cells are those listed in `## Validation criteria`, and "unhurt" is never one of them.
- **Already known or already solved.** The fix exists upstream (story 4), or a standard tool covers it, and the recommendation is a reinvention. A bespoke option must name why the ecosystem route fails.
- **Stale practice.** Dask's internals moved substantially (dask-expr, P2P shuffle, TaskSpec replacing tuple graphs). Advice about custom graphs or `split_every` with no version or date, or from before those changes, cannot carry a recommendation on the current `2026.7.1`.
- **Vendor framing.** Dask-affiliated and vendor blogs and benchmarks are useful and interested. Record the trust judgment; a recommendation cannot rest on one of these alone.
- **Absence claimed as fact.** "No library does X" is admissible only as a search-scope statement naming what was searched.
- **Premise confirmation.** The panel treats the tree result or the maintainer's constraints as given and never tests them. Stories 7a to 7d each get their own verdict, with evidence; one blended verdict is killed.
- **Silent raster change.** An option that changes pixels (story 6) or adds rounding differences (a tree on float reductions) is presented as a pure memory fix. Each option states its exactness against the current raster and against the repo's equivalence gates. Tree equivalence evidence is 20k nodes / 400k edges / eight reductions, and a claim citing it says so; "proven" is killed.
- **Prior-ruling collision.** A recommendation reintroduces a Dask opt-in parameter, a pandas-only boundary branch or an internal canonical frame without naming the reversal.
- **Literature passed off as measurement.** A literature run cannot establish memory or time numbers for this route. Only a measured row of `RESULTS.md` or the prior finding counts as a measurement. Both 1B figures in `RESULTS.md` and the arithmetic in `## Inputs` are predictions: they appear labelled as predictions and never fill a `measured here` cell. Anything else becomes a named proposal under `What's open`.
- **Grounding failure.** A claim without a faithful quote, source-code claims without a version or tag, or claim-maker equal to voucher.

## Lenses + bounds

Five lenses, each disjoint, each blind to the others: two standing seats, then three orthogonality slices. The material boundaries below are what make them disjoint; a lens that wanders outside its own hands the pointer back rather than reading it. Each brief names the ranking cells its answer moves.

**Common to every lens.** Literature only: read docs, source, papers, issues and posts; run no benchmarks; write nothing to any upstream tracker, forum or repo. Every claim carries its supporting quote and source with a version, tag or date. Tag each finding `threaded`, `multi-process single node` or `cluster`. For each practice cited, state the state size being combined and the input-to-output ratio, so transfer can be judged. Nominate at most your allotment of claims for verification, each naming the ranking cell it moves; everything else is context and reaches the summary labelled `unverified`. Self-flag any nominated claim with a single quote behind it (research-track second-vote flag). Record a trust judgment per source. Local source for dask 2026.7.1 and datashader 0.19.1 is in `~/repos/hiveplotlib/.venv`; `distributed` is not installed there, so read it at the matching upstream tag. Return only distilled conclusions with citations, plus what you looked for and did not find. Treat `## Inputs` as things to attack. The dispatching session hands each lens `## Question`, `## Inputs`, `## Prior rulings and records`, `## Candidate stories`, this paragraph and its own brief, never another lens's return.

- **[STANDING] Prior-art / counterfactual: outside Dask.** How do systems that are not Dask libraries reduce partial results of fixed size, and how does large-graph rendering collapse work before drawing? Read Spark `treeAggregate`/`treeReduce` and MLlib histogram aggregation (why they exist, how depth is chosen), vaex `binby` grids (out-of-core work with fixed-grid state), binned pre-aggregation in visualization (imMens, Nanocubes, binned aggregation for scalable vis), and the docs and papers of large-graph tools that have rendered billions of edges or declined to (graph-tool, Graphistry, cuGraph with datashader). Answer: what is the normal shape for this problem, what do you give up by collapsing before expanding (story 6), and verdict 7c. Excludes the Dask ecosystem and issue trackers. Ray, polars and DuckDB are out, since they combine group state and not a fixed raster. Moves: story 6's exactness and memory cells, story 1 and 3 cluster cells by analogy, 7c. Claims for verification: 2.

- **[STANDING] Counter-evidence: blind to the confirming lenses.** Search for what would falsify stories 1, 2 and 4 and the maintainer's framing, and carry every non-confirming hit labelled not-fully-confirming. Hunt tree reductions that failed to bound memory or made things worse on a cluster (fan-in transfer costs, memory regressions, `split_every` complaints), reports that small partitions or many tasks caused the damage instead of the combine, library maintainers who removed or declined custom graph work, and projects that left Dask for a single-machine engine and why. Datashader's maintainers' stated stance on `cvs.line` memory and the flat combine is yours: PR #926, what preceded and followed it, and issues and discussions on Dask line memory. Weigh whether a billions-of-edges path is real demand or a hypothetical, and whether one-sample threaded RSS misleads here (allocator retention, malloc arenas, `MALLOC_TRIM`), pointed at the 2.18 GiB floor. For 7b, report a search-scope statement and spend no nomination on it. Search by failure vocabulary in issue trackers, discussion forums and post-mortems, not by feature names. Moves: any option's cells where a hit contradicts them, the measurement-artifact rubric entry, 7b. Claims for verification: 3.

- **Combine practice inside Dask's collections and fixed-output libraries.** How do libraries reduce per-block results whose state size is fixed by the output? Read `dask.array.reduction`, the `apply_concat_apply` path and its `split_every` default, `dask.array.histogramdd`, `dask-histogram` with boost-histogram (the closest analog, since a raster is a 2D histogram), flox's `map-reduce`, `blockwise` and `cohorts` methods as used by xarray, and how `xgboost.dask` and `dask-ml` aggregate (collective allreduce, incremental `partial_fit`) as non-tree models. Read datashader's own `default` path as one more instance: how `LinesAxis1` behaves on Dask, which reductions force the lengths pass, what its dataframe-as-array construction risks, row-width costs and antialiasing differences. Answer: how fan-in is chosen and exposed, tree versus fold versus in-place accumulation, how multi-array state such as moments combines and to what exactness, whether the choice differs by scheduler, and whether story 4 is viable from source. Excludes how these libraries enter the graph and what they serialize. Moves: exactness and maintenance cells for stories 1, 3, 4 and 5, and the var/std policy choice. Claims for verification: 4.

- **Sizing work when output is much bigger than input.** How do libraries choose partition or chunk sizes, and do they ever change them for the user? Read `repartition(partition_size=)`, Dask's array auto-chunking and `array.chunk-size`, `blockwise` `adjust_chunks`, operations that expand rows such as `explode` and bag `flatten`, flox's cohort rechunking, the memory planning in `rechunker` and `cubed` (`allowed_mem`, projected memory, which are explicit bounded-memory designs), and Dask's best-practice guidance on partition count versus size. Answer: does any rule size by output bytes rather than input bytes, is an interior optimum in partition count documented anywhere, should a library auto-repartition or warn or defer (given the round-trip contract only binds what is returned to the user), what must a defensible sizing rule contain (attack the three-term arithmetic in `## Inputs`), and verdict 7d. Excludes scheduler internals and allocator behavior. Moves: all of story 2's cells, single-machine memory for every option, 7d. Claims for verification: 2.

- **Crossing a process boundary.** What does this workload cost once work leaves one process, and what do mature libraries send across? Read in this order, and if the lens runs out of room, return what you hold and name what went unread. (1) `distributed` docs and source at the tag matching dask 2026.7.1 on worker memory management (target, spill, pause and terminate thresholds, zict, nanny), dependency gathering when one task takes many inputs, placement and work stealing, `dask.order`, practical graph-size limits, and comm costs for 10 MiB payloads. (2) Large shared read-only state: `scatter(broadcast=True)`, `Variable`, `WorkerPlugin`, per-worker caches, what happens when a task closure captures a large object as this route's builder does, and lookup practice against such a table (broadcast merges in `dask.dataframe`, `Index.get_indexer` or array lookups inside `map_partitions`), since each build task currently merges against a whole axis table. (3) Stable surfaces: `dask-geopandas`, `spatialpandas`, `dask-ml`, `lightgbm.dask`, `stackstac` and the entry surface of `xgboost.dask`, plus Dask's stability statements on `map_partitions`, `from_map`, `to_delayed`, `HighLevelGraph`, `Expr` and TaskSpec, and whether custom graphs survive the optimizer. (4) Compiled-code threading such as numba under threaded versus multi-process schedulers. Answer: on a cluster what does a flat combine of thousands of 10 MiB aggregates do to one worker, what does a tree cost in transfer, whether node-local combining exists or helps, whether spilling rescues any of this, how a 20M-node lookup table should reach workers, and whether libraries set scheduler config or defer to the user's. This lens speaks to the maintainer's cluster constraint, so distinguish documented behavior from inference. Excludes how results are aggregated and allocator behavior. Moves: the cluster cell of every option, story 8's cluster and maintenance cells, story 5's maintenance cell, 7a. Claims for verification: 5.

**Addable pre-flight estimate (binding), counted against every Agent call in the run.**

```
3  spent              research-plan, plan-stage adversary, this amend-plan
5  lenses             2 standing + 3 orthogonality
16 verify vouchers    prior-art 2, counter-evidence 3, combine 4, sizing 2,
                      process boundary 5
4  second vouchers    generation-flagged, single-quote nominated claims
1  synth
1  adversary          convergence gate
1  record-keeper      producer path
= 31 Agent calls in all, 28 still to fire
  (27 by the research-track model: 1 scope + 5 lenses + 20 vouchers + 1 synth)
```

**Concurrency ceiling: 8.** Peak expected is 5 during the lens wave; vouchers run in waves of at most 8. Exceeding 31 requires surfacing a revised estimate first; under the async posture, park it in `## Maintainer questionnaire` and record it. Vouchers verify nominated claims only, and a nominated claim beyond a lens's allotment is labelled `unverified`.

## Validation criteria

A validated finding is grounded (faithful quote, source version), independently verified (claim-maker is not the voucher, default refuted, a second voucher on self-flagged claims) and cleared by the adversary at convergence against the rubric above. Bound to this run:

- Every finding carries its scope tag and transfer note.
- **Options, not stories.** The synth ranks at most six options, and a story that fails drops out with one line saying why. Every option states all five effects and the term that binds at the questionnaire item 1 target.
- **Cell marks.** `measured here` names a row of `RESULTS.md` and its scope. `literature-supported` names a validated claim. `read-from-source` names version and file; from a lens it needs a voucher like any claim. `unmeasured` covers everything else and may carry a labelled prediction. Cells carried from `## Inputs` are not lens claims and need no voucher. A cell whose only support is an unverified claim reads `unmeasured`.
- **Cluster cells.** Each option's cluster cell reads exactly one of: `known mechanism of harm: <mechanism, documented or inferred from graph shape>`; `no known mechanism of harm, unmeasured (searched: <what>)`; or `unknown`. A recommendation resting on the second says so in the ranking. "Unhurt", "safe on a cluster" and their synonyms are killed. Processes-scheduler evidence is tagged `multi-process single node` and never fills this cell by itself.
- **Effect order.** The ranking states the order it used, from questionnaire item 6 or its recommended answer, so the weights can be attacked at the gate.
- **Story 7.** Four verdicts, each `supported | not supported | inconclusive` with evidence. 7b is expected to read inconclusive, search-scoped. The counter-evidence lens's hits reach the page labelled not-fully-confirming.
- Effects on this route that the literature cannot settle become named proposals under `What's open`, and are not reported as findings.
- Findings frame correlational pointers as such. "The tree caused the drop" is supported by the two probes' mechanism at 10M edges and by nothing wider.

Terminal outcome, classified honestly:

- **validated finding**: a confident, grounded conclusion on what the route should do. Lands the full page.
- **validated inconclusive**: pursued to a negative, such as "the literature does not settle how this workload behaves on a cluster". First-class; lands the full page as a do-not-re-research reference.
- **nothing-cohered**: the panel ran and nothing of interest cohered. Lands a minimal breadcrumb only.

A mixed page is the expected shape: practice findings validated, the cluster half possibly inconclusive.

### Proposed experiments

Ran: (a) `dask_scale_probe.py --nodes 2000000 --edges 100000000 --partition-edges 10000000`, and (b) in narrower form, one `_curves_from_id_pairs` call under `cProfile` at two node counts. The rest are proposals for `What's open`, carried with what each settles. No lens runs them. All belong in `hiveplotlib-performance`.

- **(b, remainder) Whole-render attribution** at 20M / 10 and 2M / 100: node drawing against edge build against merges, memory included. Settles how much of the 475 s and 3.67 GiB the merge explains, and why render time grows with partition count.
- **(c) `axis=1` with one row per edge.** Story 4's viability, memory and raster against current.
- **(d) `LocalCluster` with a worker memory limit,** flat against public tree, at 2M nodes. The first cluster-scope evidence and 7a's cost at size. Needs `distributed` in a scratch environment (questionnaire item 4).
- **(e) Five samples plus a held-live measure against RSS.** The 2.18 GiB floor and the first rubric entry.
- **(f) A precomputed per-axis lookup in place of the merges,** in a throwaway copy, at 2M and 20M nodes with raster comparison. Story 8's time and memory.
- **(g) Flat near the predicted optimum (about 43 partitions) against the public tree at 100M edges.** Tests whether the tree gain grows with edges, with no 1B run.
- **(h) The 2M / 100M / 10 row at `num_steps=50`.** Tests the curve term against the signed default.

### Summary shape the run reports

```
Question: <the refined research question>
Evidence mode: literature (measured inputs carried from RESULTS.md, none produced by a lens)
Lenses run (each disjoint, blind to the others; the two standing seats floored):
  - prior-art / counterfactual (outside Dask): <distilled conclusion + citation>
  - counter-evidence: <what would falsify the story; not-fully-confirming hits>
  - combine practice: <distilled finding + citation>
  - sizing when output exceeds input: <distilled finding + citation>
  - crossing a process boundary: <distilled finding + citation>
Findings (each with citation, scope tag, transfer note, independent-verification verdict):
  - [validated] ... (voucher re-read the cited source; default-reject cleared)
  - [killed] ... (claim-maker != voucher; cited source did not support)
  - [unverified] ... (beyond the voucher pool; ungrounded at the gate)
Effect order used: <from questionnaire item 6, answered or recommended>
Options ranked (default path / power-user path): <option: five effects, each marked
  measured here | literature-supported | read-from-source | unmeasured; binding term
  at the item 1 target; cluster cell in one of the three permitted forms>
Maintainer-framing verdicts (story 7), each supported | not supported | inconclusive, with evidence:
  - 7a serialization outweighs tree shape: ...
  - 7b nobody renders graphs on a cluster: ... (search scope: ...)
  - 7c billions not worth drawing edge by edge: ...
  - 7d partitions are the wrong unit: ...
Proposed experiments: <what each would settle; none run by this run>
Adversary verdicts (convergence gate): validated / killed, each with reason + evidence
Open questions: ...
Yield (worth-landing determination): <validated finding | validated inconclusive | nothing-cohered>
Pre-flight estimate: 31 Agent calls (27 by the research-track model; binding; revise-and-surface to exceed)
Resources consumed: <n> agents, peak concurrency <n> (ceiling 8), ~<n> tokens (basis: task-notification subagent figures; "unknown" when unmeasured)
Destination: <full page at the path below, or the minimal breadcrumb>
```

## Destination artifact

New page at `wiki/wiki/analyses/dask-route-memory-bounding.md`, created through record-keeper's producer path under maintainer approval, for a validated finding or validated inconclusive. A nothing-cohered run lands a minimal breadcrumb at that path instead. No existing analyses page covers this, so this is a new page and not an enrich or rebuild. Agents do not commit; the maintainer reviews and commits. Repo code and docs must never cite this plan or the page.

This run authorizes no code, docs or notebook change. The statements in the notebook, docstring and error advice that need correcting do not wait for this page; questionnaire item 7 routes them separately.

## Minor pivots vs. amend-plan

Iteration lives inside the bounded run: refining a lens or dropping a dead angle needs no amendment. Only a fundamental change of the research question is an amend-plan.

## Alignment (grill)

```
Status: research-branch grill knowingly skipped, 2026-10-30.
The maintainer directed the run to start now while they keep reviewing, so the grill is
parked rather than run live. The failure-mode rubric above is orchestrator-seeded as a
result. Open decisions are parked in ## Maintainer questionnaire; the run proceeds under
each item's recommended answer unless the maintainer says otherwise.
Delivery shape: n/a (research run; no MR).
```

With the grill skipped, nobody fights the adversary's plan-stage challenge live, so it routed to `amend-plan` for explicit disposition before the lenses fire. The dispositions are in `## Plan amendments`, were adopted under the async posture, and stay open to re-litigation at review.

## Adversary review

### Adversary's challenge (planning mode)

```
Status: challenge (11 items)
Plan reviewed: wiki/wiki/plans/dask-route-memory-research.md (plan stage, cold)
Angles worked: all three, since a research plan has no spec-stage pass. Premise: items 2, 3, 10.
  Approach: items 1, 5, 6, 7, 9. Size-and-maintenance: items 4, 5. Considered and rejected an
  existential tag: the run survives as a smaller literature pass, because the cluster mechanism,
  surface-stability and prior-practice questions cannot be answered by a probe.
Rubric: orchestrator-seeded, not the maintainer's (questionnaire item 5). Items map to it as written.
Checked by reading, nothing run: RESULTS.md, all of public_tree_probe.py and dask_scale_probe.py,
  datashader data_libraries/dask.py, src/hiveplotlib/viz/datashader.py, hiveplot.py:1699-1804.
  These plan claims held: the LineAxis0 / default split, split_every=2, "dodgy", "Here be dragons",
  no scheduler / processes / pickle / LocalCluster / distributed hit in tests/dask_test.py, the
  64% thread-cap figure, the table, and no analyses page mentioning Dask.
Items:
  - [must-fix] `## Inputs` is stale against RESULTS.md's "Public-API tree experiment", and every lens is handed `## Inputs` as its brief. (at Inputs: "The route has apparently never left one process"; Candidate stories 4 and 5; Questionnaire 4)
    Rubric: "Already known or already solved"; "Scope laundering across schedulers"
    Push: Amend before firing. False now: "Nobody has checked whether that works", and questionnaire 4's "Only an actual multi-process or LocalCluster run would settle whether the builder closure serializes". `scheduler="processes"` ran flat and tree to completion. Add the section's numbers: tree peak 2.18 GiB against flat 7.54, render 107.4 s against 116.4, same lit pixels, count/any/max/min bit-exact, sum/mean/var/std within 4.4e-16. Story 4's selling point, "the tree through public API", is no longer unique to it. Story 5's "the var/std story" now has a demonstrated answer, and it is no longer a hypothetical cost. What must survive the edit: `distributed` is still untested; the processes run was 20k nodes / 400k edges / 40 partitions, so it says nothing about pickling cost at 2M or 20M nodes and reports no memory figure; "a flat combine may be worse on a cluster" is still graph-shape inference; the fan-in 8 bullet's "threaded only, one run" stays true of that probe. Serialization moves from "does it work" to "what does it cost at size".
  - [must-fix] The "3.5x tree gain" compares the tree to flat at its worst partition count, and the plan quotes only the worse half of RESULTS.md's 1B prediction, so story 1 is "emerging" on a baseline a user can leave with `.repartition()`. (at Inputs, "A flat combine may be worse"; Candidate story 1; Failure-mode rubric first entry)
    Rubric: no entry, flagging anyway. Nearest is "Result is a measurement artifact", which covers the instrument and not the comparison.
    Push: Against the plan's own table, flat at 10 partitions is 2.55 GiB in 45.7 s; the tree at 100 partitions is 2.18 GiB in 107.4 to 120.0 s. That is 1.17x on memory and about 2.3x to 2.6x slower (one sample each, different partition counts, so a pointer and not a speed claim). RESULTS.md's next sentence after the 57 GiB one reads: "100 partitions of 10M edges at `num_steps=16` should peak near 10-11 GiB". The Inputs bullet cites only 1,000 partitions / 6,000 aggregates. Sized flat at 1B edges holds 600 aggregates, about 5.6 GiB, so story 2 may already reach the maintainer's billions path on one machine with no graph change, as a prediction. Add that sentence to Inputs. Add a rubric entry "Wrong baseline": every option is compared to flat at its best partition count, and a tree must name the case where sizing is unavailable (a user who cannot repartition, a skewed source, a worker limit below the sized floor). Tighten "Any such number not from RESULTS.md is killed" to measured rows only, since both 1B figures are extrapolations sitting in that file and the rule currently passes them as measurements.
  - [must-fix] The largest measured effect has no story, no lens and no rubric entry: the node table. The question is aimed at the combine, while the 20M-node row spends 520.7 s and 11.28 GiB on 10M edges and only 60 aggregates (about 0.56 GiB). (at Question; Candidate stories; Lenses; Questionnaire 1)
    Rubric: no entry, flagging anyway
    Push: Holding the table's densest 2M-node density of 50 edges per node, 1B edges means 20M nodes, which is the only row where the combine is a rounding error. Questionnaire 1's recommended answer names edges and `num_steps` and no node count, so a "real path to billions" could pass at 500 edges per node, a density no row has measured. Read and not run: hiveplot.py:1742 does `edge_ids.copy().astype("O")`, then 1747-1767 runs two `merge` plus `sort_values` calls against an axis's whole `node_placements`, per output partition (datashader.py:797). Tasks scale with partitions and each join's table side scales with nodes, which fits both "render time grows with partition count" and the 11x in the 20M row by mechanism, unprofiled. This is one function with no literature question in it, and a shared per-axis id lookup would stay inside "nodes are in memory by design". Add a story for it, make Questionnaire 1 state a node count, and require each ranked option to say which term binds at that ratio. What must survive: story 7's serialization candidate, which is a different cost.
  - [must-fix] The binding estimate is wrong on firing. `## Alignment (grill)` requires this challenge to route to `amend-plan` before the lenses fire; that Agent call is in no line of the 32. (at Addable pre-flight estimate; Alignment)
    Rubric: no entry, flagging anyway. research-track binding-estimate rule.
    Push: Restate as at least 33. Separately, the voucher pool will not cover the output: 16 single votes across 7 lenses is about 2.3 claims per lens, and the required ranking is 7 stories x 5 effects = 35 cells plus practice findings. Unverified claims die at my gate per research-track, so "literature-supported" cells mostly cannot survive as budgeted. Either cap each lens at its voucher allotment of load-bearing claims in the brief, or raise the pool now. Do not leave it to a mid-run revision.
  - [worth-discussing] Seven lenses is not earned: "each disjoint" overclaims, one lens cannot move the ranking, and the prior-art reading list mostly fails the plan's own transfer rule. (at Lenses + bounds)
    Rubric: "The analogy does not transfer"
    Push: Overlaps as written: flox and `blockwise` in combine and sizing; `xgboost.dask` and `dask-ml` in combine and conventions (exclusion clauses help); cuGraph / Graphistry in prior-art against counter-evidence's "anyone runs graph rendering"; datashader issue trackers in the datashader lens and counter-evidence; `MALLOC_TRIM` in counter-evidence against the scheduler lens's worker-memory reading. Cuts: (a) Remove the datashader lens. Why the line path is flat changes no option while nothing is filed upstream, and `axis=1` viability is settled faster by a probe. (b) Merge conventions into the scheduler lens as "what crossing a process boundary costs". (c) Drop Ray, vaex, polars and DuckDB from prior-art, since bounded group state against a fixed raster after expansion is the non-transfer the rubric names. Must survive: both standing seats; maintainers' stated stance on `cvs.line` memory, moved to counter-evidence's tracker search; `to_delayed` / `from_map` stability; large shared state shipping (`scatter`, `Variable`, `WorkerPlugin`); numba under each scheduler; Spark `treeAggregate` and binned-vis pre-aggregation for story 6. Result is 5 lenses. Keep the voucher pool at 16 + 4 and spend it on fewer claims, so the total is about 31 with the amend-plan call. The gain is coverage per claim, not a smaller bill. Also add to each lens brief which ranking cell its answer moves; a lens that cannot name one should not fire.
  - [worth-discussing] The cheapest route to the decision is more probes, and the plan already predicts that. Validation caps every memory, time and transfer effect at "unmeasured" or a proposal, so the ranking's deciding cells will read "run this". (at Bounds set by the maintainer; Validation criteria; Questionnaire 4)
    Rubric: "Literature passed off as measurement"
    Push: Maintainer's call, since "runs no benchmarks" is their bound; lenses can still fire now. Between this plan's writing and this pass, one probe moved stories 4, 5 and 7 further than a lens can. Nameable today, in hiveplotlib-performance: (a) `dask_scale_probe.py --nodes 2000000 --edges 100000000 --partition-edges 10000000`, existing flags, 60 aggregates and no 1B run, which tests the term the 10-11 GiB prediction rests on; (b) one profile of the 20M / 10 and 2M / 100 rows for item 3; (c) `axis=1` with one row per edge for story 4; (d) `LocalCluster` with a worker memory limit, flat against public tree, at 2M nodes; (e) five samples plus a held-live measure against RSS, for the rubric's first entry. Ask: move questionnaire 4's "once you have read the page" to before synthesis, and let (a) to (c) run alongside the lenses so the page ranks on them. If unanswered, the run proceeds under its current bound and these become the page's `What's open` list verbatim.
  - [worth-discussing] Story 5 can land on the page as "proven" when the probe covers 8 reductions and silently adds the rest; story 4 keeps an advantage the new section does not touch. (at Candidate stories 4 and 5; rubric "Silent raster change")
    Rubric: "Silent raster change"; "Already known or already solved"
    Push: public_tree_probe.py:76 is `return {"value": (reduction, "add")}` for anything unlisted, so `first`, `last`, `by` or `mode` would give wrong pixels with no error, while the route's docstring (datashader.py:849-851) promises exactness "for every reduction `datashader` supports on Dask input". State the shipping option properly: `_STREAMED_REDUCTION_COMBINE_MODES` (datashader.py:70) already is an exact-combine table with fall-back for unlisted reductions, so listed reductions take the tree and unlisted stay on flat `cvs.line`. Its comment says var/std have "no exact per-chunk combine", and rounding-close moments contradict that policy, so the option must say which holds. The moment rebuild is also a second implementation of datashader's reductions, which is the "prefer ecosystem tools" cost. Story 4 keeps datashader's own combine for every reduction with no table, at the price of a code path its authors call "dodgy". Equivalence evidence is 20k nodes / 400k edges only; say so wherever it is cited.
  - [must-fix] My convergence gate has no runnable test for the maintainer's main constraint. The rubric kills "unhurt" with no cluster-scope evidence, Validation caps the cluster half at "literature supports this mechanism", and "cluster-scope evidence" is undefined. Story 7 also bundles four claims under one verdict. (at rubric "Scope laundering" and "Premise confirmation"; Validation criteria; Candidate story 7)
    Rubric: "Scope laundering across schedulers"; "Absence claimed as fact"; "Premise confirmation"
    Push: Define now: mechanism-level literature permits "no known mechanism of harm, unmeasured" and never "unhurt", and a recommendation resting on that says so in the ranking. Split story 7 into four verdicts. Serialization is now partly answered (item 1). "Nobody renders graphs on a cluster" can only ever be a search-scoped inconclusive under the plan's own absence rule, so predict that on the page and do not spend a voucher chasing it. "Billions not worth drawing edge by edge" is a design judgment no lens owns; assign it to prior-art with story 6 or strike it. "Partitions are the wrong unit" is unowned too. A single `supported | not supported | inconclusive` field launders all four.
  - [worth-discussing] The rubric misses workload artifacts and the floor nobody can explain. (at Failure-mode rubric first entry)
    Rubric: "Result is a measurement artifact" (extends it)
    Push: Add (a) workload: every row is uniform random edges from `dd.from_map` with no disk reads, which RESULTS.md itself calls "the friendly case for axis-pair skew", at 3 axes, `ds.count()`, 1500 x 1500. A finding extending this to skewed or Parquet-backed data must say so. (b) `num_steps`: every row is at 16 and the signed default is 50, which triples the curve term and leaves the combine term alone, so the interior optimum moves and no row shows where. Questionnaire 1's recommended "at `num_steps=16`" should be a stated choice against that default. (c) Unattributed floor: the two tree probes both report exactly 2.18 GiB though only one keeps the stitching read. That is about 0.91 GiB build plus roughly 0.6 GiB of live aggregates (8 threads x 8 rasters x 9.5 MiB), so about 0.7 GiB is unexplained. Identical figures from different mechanisms suggest something else sets the floor, which is what the allocator-retention check should be pointed at. Also absent: the research branch's named "n-too-small" and "uncontrolled comparison" modes.
  - [worth-discussing] Nothing says how five effects become a ranking, and no named user workflow is blocked today. A ranking with no stated order cannot be attacked at the gate. (at Question)
    Rubric: "Premise confirmation"
    Push: Two questions for the maintainer. First, the order among single-machine memory, cluster, render time, exactness and maintenance; is 2.18 GiB in 107 s better than 2.55 GiB in 46 s? Without an order the synth's weights are invisible. Second, why now: this route has not shipped in a release (branch 53 is unmerged), so no user has hit this. Does the run's output gate that release? If only the false statements must be right before release, those do not need this run: the notebook's "Partition Sizing", the docstring at datashader.py:851-852 and :866, and the error advice at :965-967 are wrong under every story, since each says shrinking partitions helps. Route that correction separately instead of "for whatever follows the page". Keep the byte formula's `num_steps` lever when rewriting, which stays true.
  - [low-confidence] No record-keeper pre-task pass ran, as the plan admits, so "Prior rulings and records" comes from one session's reads. (at Prior rulings and records)
    Rubric: "Prior-ruling collision"; "Already known or already solved"
    Push: My sweep of the wiki for `split_every`, `to_delayed`, `from_map`, `LineAxis0`, `treeAggregate` and "flat combine" found nothing outside this plan, and "partition ceiling" matched one scaling-plan line that records the notebook guidance. Both are weak evidence of no collision. One call would cover it, and it would add 1 to the estimate. Fine to skip if the page goes through record-keeper's producer path anyway, but record the skip as a decision.
```

### Adversary post-impl

```
Pending — the research-run equivalent is the convergence-gate pass; its verdicts land in the
run summary's "Adversary verdicts" block.
```

## Maintainer questionnaire

Parked 2026-10-30 because the grill is skipped. Answer in chat; a one-word reply closes an item. The run does not block on any of these and proceeds under each recommended answer.

```
1. What does "a real path to billions of edges" have to mean for a recommendation to count?
   Context: lenses and the convergence gate judge the power-user option against this. Nothing
   in the run measures it. Holding the densest measured ratio (50 edges per node), 1B edges
   means 20M nodes; 2M nodes would be 500 edges per node, a density no row has measured.
   num_steps is a real choice too: every row used 16, the signed default is 50, and the curve
   term triples at 50 (at 1B edges, 2M nodes, sized flat is predicted at about 17 and 28 GiB).
   Recommended answer: a documented way of running where a 1B-edge, 20M-node render at
   num_steps=16, set explicitly as a power-user choice against the default of 50, completes on
   multi-machine or large single-machine hardware with every worker under its memory limit.
   Hours of wall time are acceptable, and it may be non-default and need setup. Each option
   also says what 50 does to it.
   Answer (on close): 2026-10-31, maintainer, not closed: the recommended answer is rejected on
   node count. 20M came from the largest measured row, not from what a 1B-edge graph looks like.
   "A billion edges" means a very, very big graph, presumably with far more than 20M nodes, so
   nodes have to scale too, sooner rather than later. The node target is not yet set; the
   dispatching session proposed 100M nodes as a working target, unconfirmed. Also ruled
   2026-10-31: nodes get a lazy route on branch 53, added beside the in-memory path the way lazy
   edges were, and lazy nodes are datashader-only. In-memory nodes and their pandas
   node_placements stay as they are. Scaling by more workers or more disk is the goal; how edges
   look up their nodes on that route (copy per worker or shuffle) is not decided.

2. Is a lossy collapse-before-expanding option (story 6) allowed into the ranking?
   Context: it is the most literal reading of "collapsing things whenever possible", and it
   changes pixels, so it could not pass the repo's equivalence gates as a default.
   Recommended answer: yes, ranked as a power-user-only option with its exactness cost stated,
   never as a default.
   Answer (on close):

3. If the best answer is a change inside datashader, how should the page treat it?
   Context: nothing is filed or posted upstream on this run.
   Recommended answer: the page records the would-be change and any hiveplotlib-side workaround
   as facts; whether to file anything stays a later maintainer call, and no agent drafts it here.
   Answer (on close):

4. May a follow-up throwaway experiment install `distributed` in a scratch environment?
   Context: scheduler="processes" showed the builder's closure pickles at 20k nodes. Nothing
   has run on distributed, and no pickling or memory cost exists at 2M or 20M nodes. Only a
   LocalCluster run would settle what the closure and a tree cost in transfer (proposal d).
   Gates what comes after the page, not this run. Proposals b, c, e, f, g and h need no
   distributed and can run whenever you choose, before or after the page.
   Recommended answer: yes, in the hiveplotlib-performance repo and never in the project venv,
   scoped by what the page proposes, once you have read the page.
   Answer (on close):

5. Does the orchestrator-seeded failure-mode rubric stand in for the skipped grill wave?
   Context: the rubric is normally elicited in your words; the adversary reads it at convergence.
   Recommended answer: yes for this run; add or strike entries at review.
   Answer (on close):

6. In what order should the five effects rank options?
   Context: without an order the synth's weights are invisible. Case A: 10M edges, the tree at
   2.18 GiB in 107 s against sized flat at 2.55 GiB in 46 s. Case B: predicted 1B edges, sized
   flat at about 17 GiB against a tree at about 3 GiB, on a machine or worker limit under 17.
   Recommended answer: exactness and cluster are floors, memory is a fit test, then time, then
   maintenance. A default must pass the repo's equivalence gates, and a known cluster mechanism
   of harm must be stated. Among options that complete under the item 1 target, faster wins,
   then the smaller surface. So flat wins case A because both fit, and the tree wins case B
   because flat does not.
   Answer (on close):

7. Does this run's output gate the branch 53 release?
   Context: the route has not shipped, so no user has hit this. Three statements are wrong or
   one-sided whatever the ranking says: the notebook's "more partitions only lower the ceiling", the
   _dask_native_edge_raster docstring's bound by the largest partition's curves, and the
   "repartitioning to smaller partitions is the usual fix" in its :raises: line and error advice.
   The correction has to run both ways, since going from 10 to 100 partitions raised the 10M-edge
   peak (2.55 to 7.54) and lowered the 100M-edge peak (12.48 to 8.59). The byte formula stays,
   since it matches the curve term.
   Recommended answer: no. The release does not wait for this run. Correct the three statements
   before release as their own small task, and let the page drive any graph change afterward.
   Answer (on close): 2026-10-31, maintainer: recommended answer rejected. This blocks branch 53.
   The point of 53 is to settle scaling, and a half measure would mean structurally changing it
   after it is part of the API. Settle it once, on 53.
```

## Plan amendments

All dispositions below were adopted under the async posture with the grill skipped, and stay open to re-litigation at maintainer review. Body sections were updated in place so lens briefs read current text.

### In-scope tweak: dispose the plan-stage adversary challenge

**Date:** 2026-10-30
**Trigger:** `### Adversary's challenge` (11 items), routed here because the grill is skipped, plus `RESULTS.md` sections added after the challenge was written.
**Workstream affected:** n/a (research plan).
**Change:**

| # | Severity | Disposition | Delta |
| --- | --- | --- | --- |
| 1 | must-fix | Adopted | `## Inputs` rebuilt from `RESULTS.md`. Processes run recorded. Stories 4 and 5 and questionnaire 4 corrected. Survives: `distributed` untested, no cost at size, flat-on-cluster still inference, per-probe run counts. |
| 2 | must-fix | Adopted, figures replaced | 3.5x removed everywhere. "Wrong baseline" rubric entry added, and "measured rows only" tightening. Item's "about 5.6 GiB" for sized flat at 1B is the aggregate term alone, and `RESULTS.md` now predicts 17 to 18. Item's implied conclusion is not carried: the gain against sized flat is predicted to grow with edges, so 1.17x is stated as one point. |
| 3 | must-fix | Adopted, attribution narrowed | Story 8 added, node lookup assigned to the process-boundary lens, questionnaire 1 states a node count, each option names its binding term. Merge attributed within one call only, since the row is not shown to be merge (at most 376 s of 475 s). Survives: 7a as a separate cost. |
| 4 | must-fix | Adopted | Estimate restated at 31 with spent calls counted. Cap-per-lens chosen over a larger pool, by nomination allotments, and the ranking cut from 35 cells to at most six options, with `## Inputs` cells needing no voucher. |
| 5 | worth-discussing | Adopted with two changes | Seven lenses to five; scheduler and conventions merged. Datashader lens removed, but its "a probe settles `axis=1` faster" reason fails while no probe runs, so the `LinesAxis1` source read moves to the combine lens and maintainers' stance to counter-evidence. Ray, polars, DuckDB dropped; vaex `binby` kept, since its state is a fixed grid and the non-transfer does not describe it. Every lens brief names its cells. |
| 6 | worth-discussing | Partly adopted | Attribution of "no benchmarks" corrected. (a) and (b, narrower) ran. Remaining probes are proposals; not run alongside lenses, per dispatching-session direction. (b)'s remainder and (f) to (h) added. |
| 7 | worth-discussing | Adopted | Shipping form stated in `## Inputs`: exact-type table, unlisted reductions stay flat, var/std choice forced. Equivalence scope in rubric. |
| 8 | must-fix | Adopted | Cluster cell limited to three forms, "unhurt" killed. Story 7 split into 7a to 7d with owners and four verdicts. 7c assigned, not struck. 7b search-scoped, no voucher. |
| 9 | worth-discussing | Adopted | Workload entry, `num_steps`, the 2.18 GiB floor, n-too-small and uncontrolled comparison added to rubric. |
| 10 | worth-discussing | Parked, one reason rejected | Questionnaire 6 and 7. The statements do need correcting under every story, but not for the item's reason, "each says shrinking partitions helps": shrinking helped at 100M edges (12.48 to 8.59). The correction is two-sided and not a removal. |
| 11 | low-confidence | Skip recorded | In `## Prior rulings and records`, with the reason. |

### In-scope tweak: correct the "no benchmarks" attribution

**Date:** 2026-10-30
**Trigger:** Dispatching session: the bound came from its brief, and the maintainer only ruled out a 1B-edge run for now.
**Workstream affected:** n/a (research plan).
**Change:** one maintainer-bounds paragraph → two paragraphs in `## Question` separating maintainer bounds from brief bounds. Lenses still run no benchmarks.

### Deferred follow-up: remaining probes

**Date:** 2026-10-30
**Trigger:** Adversary item 6; dispatching-session direction.
**Target:** the page's `What's open`, from `### Proposed experiments`.
**Rationale:** Lenses cannot measure, and only (d) needs a maintainer answer first. Deciding cells on the ranking will read `unmeasured` until these run.

**Update 2026-10-31:** (e), (f), (h), (i), (k) and most of (j) ran at five samples each, plus lazy-node lookup comparisons up to 300M nodes and 300M edges. Results are in the page's `What's open` and `hiveplotlib-performance/synthetic_scaling/dask_partition_memory/RESULTS.md`. (d) still waits on questionnaire item 4.

### Deferred follow-up: false statements in notebook, docstring and error advice

**Date:** 2026-10-30
**Trigger:** Adversary item 10.
**Target:** a separate small task before the branch 53 release, on the maintainer's answer to questionnaire 7.
**Rationale:** They need correcting under every story, and this research run authorizes no docs change.
