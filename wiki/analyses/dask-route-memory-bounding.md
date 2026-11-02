---
title: "Bounding Memory on the Dask Rendering Route"
type: analysis
created: 2026-10-31
updated: 2026-11-02
sources: [hiveplotlib-python-repo, hiveplotlib-performance-dask-partition-memory, dask-route-memory-research-run-2026-10-30]
tags: [hiveplotlib, dask, datashader, distributed, scaling, memory, tree-reduction, research-run]
---

# Bounding Memory on the Dask Rendering Route

A bounded research run (2026-10-30, literature mode, working plan `wiki/wiki/plans/dask-route-memory-research.md`) asked how rigorous Dask-integrated libraries size work, reduce partial results and share large read-only state so memory stays bounded, on one machine and on multi-machine clusters, and what [[hiveplotlib]]'s Dask rendering route should do to match or knowingly differ.

Verdict: **validated finding, with the cluster half validated inconclusive.** Passing `["x"], ["y"]` to `Canvas.line` (option A) is the leading candidate for the route's combine, gated on experiments (j) and (e). A per-axis node lookup shipped as one shared dependency (option B) is gated on (f). **The cluster effect of every option is unmeasured: nothing has run on `distributed`.** A is measured only at `num_steps=16`, a value the [[specs/lower-default-curve-resolution|signed curve-resolution spec]] rejected as a default. The signed default is 50, the code still says 100, and nothing about A has been measured at 50.

Everything below is an exploratory pointer from one-sample probes and source reading. None of it establishes cause beyond the named setup.

**Measurement scope, every memory figure:** one-sample process RSS, threaded scheduler, WSL, 8 threads, 3 axes, 1500 x 1500, uniform random edges from `dd.from_map`, no disk reads, `num_steps=16`, `ds.count()` unless stated. RSS includes allocator retention, and a nanny-managed `distributed` worker runs different allocator settings, so these figures do not transfer to a cluster.

**Scope tags:** T threaded, P multi-process single node, C cluster. **Cell marks:** `measured here` names a `RESULTS.md` row and its scope; `literature-supported` names a validated claim; `read-from-source` names a version and file; `unmeasured` covers the rest and may carry a labelled prediction.

**Transfer:** this route combines one canvas-sized raster per output partition after 8.5x to 50x row expansion. Practices cited below state their own state size and ratio, so the distance from that shape stays visible.

## What was established

### Measured peak and render time

Flat is today's route, datashader's `line` path with one combine task over every aggregate. A is the same route with the list-of-one call. T, `RESULTS.md`, one sample per cell. Below, `10M/100` means edges and partitions, and `20M/10M/10` adds nodes first.

| Nodes | Edges | Partitions | Rows per partition | Flat render (s) | Flat peak (GiB) | A render (s) | A peak (GiB) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 2M | 10M | 1 | 10M | 32.2 | 5.18 | not run | not run |
| 2M | 10M | 10 | 1M | 45.7 | 2.55 | 34.8 | 1.81 |
| 2M | 10M | 100 | 100k | 116.4 | 7.54 | 115.7 | 1.76 |
| 2M | 100M | 10 | 10M | 111.3 | 12.48 | 97.9 | 6.25 |
| 2M | 100M | 100 | 1M | 211.6 | 8.59 | 177.5 | 1.91 |
| 20M | 10M | 10 | 1M | 520.7 | 11.28 | not run | not run |

Build peak is 0.91 GiB at 2M nodes and 6.04 GiB at 20M. Render covers all of `hp.plot`, so it includes drawing the nodes. Baseline for every comparison is flat at its best measured partition count: 10M edges at 10 partitions, and 100M edges at 100 by memory or 10 by time.

### The combine

**1. The flat combine contributes a partition-count memory term, and `["x"], ["y"]` routes around it, but the combine does not explain A's whole gain.** T. `Canvas.line` with one-element lists selects `LineAxis0Multi`, which falls through dispatch to `default` (`da.reduction`, `split_every=2`, no override, no stitching) in datashader 0.19.1. No document promises this routing; it is a dispatch side effect. A length pass runs only on multi-partition frames, for `first`, `last`, `first_n`, `last_n`, row-index `where`, or a `where`, `by` or `summary` containing one. At 100M/10 the peak drops 6.23 GiB. Flat's 60 aggregates hold at most 0.56 GiB and stitching reaches only about 1.1 GiB across 8 threads, so about 4.5 GiB is unattributed and allocator retention can't be excluded.

**2. `var` and `std` do not require the flat combine.** All schedulers, formula only. The centred merge in `m2._combine` is closed over its own output, and datashader 0.19.1 already runs it unchanged at fan-in 2 on `default`. Dask's `moment_combine`, boost-histogram's `mean.hpp` and flox's `_var_combine` (a pairwise-equivalent form) use the same merge. Flat and tree are both floating-point merges over different groupings. Dask's `moment()` docstring cites Pebay.

**3. The flat combine makes one contiguous N x H x W allocation on the `LineAxis0` path.** All schedulers for allocation shape. `np.stack` copies all inputs (datashader 0.19.1 `compiler.py:492-493`); 600 x 1500 x 1500 x 4 bytes is 5.03 GiB. Scoped to `LineAxis0`: `default`, `histogramdd` and flox stack per tree node, and dask-histogram merges with a running sum and no stack. Input pages may be only partly resident (`np.zeros`), which is unmeasured.

**10. Fan-in is a constant or config value in the libraries read, usually with a public parameter.** T, P, C. Every Dask-family library read exposes one. `dask.array` and flox convert it per reduced axis, which depends on axis count and not state size. Five of the six tree libraries are Dask-family, so they are weak independent evidence on scheduler independence.

| Library, version | Fan-in | State | Row expansion |
| --- | --- | --- | --- |
| `dask.array` and `histogramdd`, dask 2026.7.1 | 4 legacy, 16 expression | per-block; bins x storage for `histogramdd` | none |
| `dask.dataframe` and `dask.bag`, dask 2026.7.1 | 8 | per-reduction | none |
| dask-histogram, main read 2026-10-30 (latest tag 2026.2.0) | 8 | bins x storage per block | none |
| flox, main read 2026-10-30 (latest tag v0.11.2) | 4 | grouped | 3:1 at the blockwise step in its documented example |
| datashader 0.19.1 `default` | 2 | one canvas-sized raster per output partition | 8.5x to 50x before the raster step on this route |

Non-tree shapes give up something else: dask-ml's chain gives up parallelism and xgboost's collective needs `distributed`.

### Sizing

**8. Automatic work sizing, among the libraries surveyed, uses sizes known at graph time or explicit eager measurement, and row-expanding operations defer.** T, P, C, as a search-scope statement. No document of an interior peak-memory optimum was found, also search-scoped.

| Mechanism | Rule | State and ratio | Version |
| --- | --- | --- | --- |
| `einsum`, `apply_gufunc` | output chunk held to about 1.25x the largest input chunk | known shapes | dask 2026.7.1 |
| `read_parquet` | 75 MB partition floor | input side, no expansion | dask 2026.7.1 |
| Cubed, rechunker | per-task or per-worker budget | known array shapes | Cubed docs dated 2022-2025 with no version shown; rechunker master read 2026-10-30 |
| `repartition(partition_size=)` | measures input bytes eagerly, warns it may be expensive | no expansion model | dask 2026.7.1 |
| `explode`, bag `flatten`, `adjust_chunks` | left to the user | unbounded ratio, no combined state | dask 2026.7.1 |
| flox reindexing | left to the user | 3:1 per block in its example | docs, flox >= 0.9 |

**9. Published partition targets are fixed bytes or a count.** T and C for Dask's 100 MB and ten-chunks-per-core guidance (stable docs read 2026-10-30), T for datashader's `cpu_count()` (v0.19.1, "on a single machine"), single-GPU for RAPIDS's memory fraction (dask-cudf 26.08), C for Cubed's ten-times rule. `array.chunk-size` is an Array default, not a DataFrame rule. That they all assume small combined state is inferred; no source says it.

**Sizing model (our own, not a finding).** `peak = build + threads x partition rows x (num_steps + 1) x 8 bytes + 6 x partitions x 9.5 MiB` fits four of five flat rows within about 1 GiB, misses the one-partition row, and has no validation near 1B edges. Its predictions, such as a sized flat at about 17 GiB for 1B edges, stay predictions. No source read shows sizing alone failing on large raster state: the advice in hand predates the tree, and the failing 64-category run was the non-tree path.

**7d, partitions are the wrong unit: supported, with a limit.** On A's measured rows peak tracks rows per partition, and the same 10 partitions spans 1.81 to 6.25 GiB between 10M and 100M edges, so a partition count says little without a row count. Budgeted designs expose bytes. Partitions remain the only lever a Dask DataFrame user has, and no DataFrame-side library read exposes a memory budget. So document and warn in rows and bytes, keeping partitions as the mechanism.

### Node table and closure

**6. Tokenizing the closure pickles the whole `hive_plot` on the client before any task runs, on every scheduler.** All schedulers for the mechanism, dask 2026.7.1: `map_partitions` to `_normalize_pickle` to the cloudpickle fallback. `measured here` (`RESULTS.md` "Tokenization of the builder closure", T, one run per size, edges = 2 x nodes, 8 partitions): 6 tokenizations and 13 dumps per render at both sizes; 55.7 MB and 0.173 s of 3.36 s at 20k nodes; 569 MB and 0.669 s of 4.67 s at 200k. Prediction only, by 100x linear extrapolation: about 55 to 67 s and about 57 GB serialized on the client at 20M nodes. This does not attribute the 20M row's 475 s growth by itself.

### RSS as an instrument

**13. Threaded RSS has a documented reason not to transfer to a nanny worker, and may overstate held-live memory.** T versus C. The nanny sets `MALLOC_TRIM_THRESHOLD_=65536`, which disables dynamic mmap adjustment (man page, `distributed.yaml` at 2026.7.1, glibc 2.39 source). Issue #5971 shows 1.16 GiB against 151 MiB unmanaged in one small reproducer. No source sizes the gap for this route.

### Precedent outside Dask

Visualization systems fix a resolution, aggregate to it, then draw (Rübel et al. SC08; imMens, EuroVis 2013). Distributed systems reduce long fixed-size state by tree, by folding into one accumulator, or by splitting state across reducers (Spark v3.5.1; the last shape is unverified). All of this is analogy: no source measures curve rasters, Dask, or tree against flat memory.

**11. Spark's tree was motivated by driver aggregation time and load.** C. SPARK-2174, PR #1110 and the 2014 Databricks blog report wall-clock numbers and give no memory motive. The later heuristic derives depth, and fan-in follows from depth. State was a model-sized vector.

**12. Per-axis-pair 2D histograms are a precedent for collapsing before drawing.** C for Rübel. Rübel et al. and imMens support bounding draw state by resolution. A first pass scanning every record holds for unconditional histograms only. Those systems have no curve expansion, and neither is a graph tool.

### Maintainer framing, outside the cluster question

- **`var` and `std` as the scaling cost:** a flat combine is not required by them, but they were a stated reason for delegating to datashader. Whether that was the deciding reason isn't established. The memory term comes from datashader's flat `line` path (findings 1 and 2).
- **Chunking as standard practice:** half right. Tree reduction is standard in the libraries read. Automatically resizing a user's partitions for expanding output is not, so that part would go beyond practice.
- **An upstream PR:** not required, since A and D need no upstream change. Filing anything stays a later maintainer call, and nothing was filed.
- **Collapsing while staying parallel:** this is what a tree does. The cluster constraint remains real, and the closure is the known mechanism that bears on it.

### Ranked options

**Effect order:** exactness and cluster are floors, memory is a fit test, then time, then maintenance. A default must pass the repo's equivalence gates, and a known cluster mechanism of harm must be stated. This is the recommended answer to questionnaire item 6, which the maintainer has not answered.

**Target for the binding term:** the run used 1B edges, 20M nodes, `num_steps=16`, every worker under its limit, hours acceptable (item 1's recommended answer). The maintainer rejected the 20M on 2026-10-31: a 1B-edge graph presumably has far more nodes, and nodes have to scale too. The node target is not yet set, so every "binding term at target" line below understates the node side, and no lens researched scaling the node table past one machine's memory.

**Branch 53:** the maintainer ruled on 2026-10-31 that this work blocks branch `53` (item 7), overriding the recommended answer: settle scaling once, before it is part of the API.

**Ranking:** A is the leading candidate, gated on (j) and (e). B is gated on (f). D is the fallback if A's routing or fan-in proves unusable. F stays parked. A's standing on tree shape rests on `no known mechanism of harm, unmeasured`, while B's closure harm is a known mechanism A does not remove.

**How they combine:** A or D, never both, for the combine. B, C and E apply alongside either. E is a different route. F is an alternative power-user mode.

#### A. List-of-one route to datashader's tree (one line at the `cvs.line` call)

- **Single-machine memory:** `measured here`, `RESULTS.md` "One-line route", T. Peak follows rows per partition: 1.76 GiB at 100k rows, 1.81 and 1.91 at 1M, 6.25 at 10M. Against flat's best measured setting that is 1.4x at 10M edges and 4.5x at 100M. The plan's unmeasured flat optimum of about 6 GiB at 100M would make it about 3x. About 0.85 GiB above build at 100k rows is unexplained, and that floor may belong to node drawing or the allocator.
- **Where sizing is unavailable:** A's gain matters beyond a well-sized flat when a user can't repartition, when a source's partitions are fixed or skewed, when a worker limit sits below the sized flat floor, or when edge count puts that floor above the hardware (about 17 GiB predicted at 1B edges, our own model). Where flat can be sized to fit, the case is the time and memory gain at the same setting, not feasibility.
- **Cluster memory and transfer:** `known mechanism of harm: the builder closure pickled per task (finding 5), shared with today's route and not removed by this option`. Tree shape itself: `no known mechanism of harm, unmeasured (searched: dask #11026 with all comments, distributed #9077, scheduling-policies.rst and scheduler.py at 2026.7.1, flox #346 and #396)`. Fan-in 2 is the costliest tree on bytes: up to 111.3 GiB at 6,000 aggregates (arithmetic), with no single-worker gather.
- **Render time:** `measured here`, one sample, separate sessions, other agents reading sources during runs, so times are loose. Faster or equal at all four settings: 34.8 s against 45.7 at 10M/10, 115.7 against 116.4 at 10M/100 (a tie), 177.5 against 211.6 at 100M/100, 97.9 against 111.3 at 100M/10. At 100M/10, A is faster and lower-memory than flat at both 100M settings. At 10M, the only flat setting A/10 doesn't better on both axes is 1 partition (32.2 s, 5.18 GiB against 34.8 s, 1.81 GiB). Under A at 100M, going from 10 to 100 partitions cuts the peak 3.3x and costs 1.8x the time.
- **Exactness:** `measured here`, 20k nodes / 400k edges / 40 partitions, through `datashade_edges_mpl`. count, any, max, min bit-exact. sum, mean, var, std differ by at most 4.4e-16, same NaN pattern. Same lit pixel counts at all four memory rows. Those differences are absolute, after density correction, with weights in [0, 1), and they compare A to flat. The route's own tests (`tests/dask_test.py:451,491,610`) use `rtol=1e-9, atol=1e-12`; the `1e-12` and `1e-14` tolerances belong to streamed sum and mean. [[0002-performance-regression-harness|ADR 0002]]'s equivalence wall compares to single-shot, so the evidence appears to meet the route's test tolerance at this size, against flat only, and has not been run against the wall's own comparison. Pixel equality depends on whole curves per partition plus the leading NaN (`viz/datashader.py:761-764`). `first`, `last`, `by` and `where` are `unmeasured`, and the length pass adds a full curve build. This evidence stops at this size and these reductions.
- **Maintenance:** `read-from-source`, datashader 0.19.1, validated. No bespoke combine, and every reduction stays datashader's. Costs: undocumented routing, hard-coded fan-in 2, dependence on `default`'s manual array construction. `unmeasured` for future dask compatibility.
- **Binding term at target:** conditional. The node table binds at 1M rows per partition or fewer, with build alone 6.04 GiB `measured here` at 20M nodes, and a prediction from the one 20M row of about 11 GiB and hours. At 10M rows, A's curve excess (5.34 GiB above build, 2M nodes) is the same order as the 20M build.
- **At `num_steps=50`:** `unmeasured`. The curve term triples and aggregates do not change. At 10M rows, A's curve excess would pass the 20M build, a prediction of about 16 GiB.

#### B. Node table and closure change (per-axis lookup, shipped as one shared dependency)

Gated on (f), which must also cover ids missing from an axis, since today's merge is `how="left"`.

- **Single-machine memory:** `unmeasured`. 20M/10M/10 is 11.28 GiB `measured here` with only about 0.56 GiB of aggregates. 3.67 GiB is unattributed, and tokenization copies are a candidate. Nodes stay in memory by design.
- **Cluster memory and transfer:** `no known mechanism of harm, unmeasured (searched: distributed docs and source at 2026.7.1 on Delayed, scatter, active memory manager; xgboost's non-broadcast scatter)`. Replaces per-task copies (floor about 357 GiB at 2M nodes and 6,000 tasks, arithmetic; about 2.4 TiB predicted from the measured 44 MB per dump at 200k nodes) with one copy per worker process. A literal `Delayed` still travels client to scheduler to one worker once, and processes on one node each hold a copy.
- **Render time:** `measured here` for the cause inside one call only (`RESULTS.md` "Sizing and node lookup follow-up"): merge takes 6.26 of 7.95 s at 6.67M nodes per axis and 0.70 of 1.64 s at 667k. Whole-row attribution is `unmeasured`: at most 376 s of 475 s if threads serialize, about 47 s if not. Removing tokenization predicts about 55 to 67 s saved at 20M nodes.
- **Exactness:** `unmeasured`. A lookup should reproduce coordinates, and a raster comparison plus missing-id behavior is required.
- **Maintenance:** `read-from-source`, small and inside `_curves_from_id_pairs` plus the builder. No prior ruling reversed. Standard array lookup, no bespoke infrastructure.
- **Binding term at target:** addresses the node table, which binds at 1M rows per partition or fewer.
- **At `num_steps=50`:** the lookup and shipped table do not depend on `num_steps`, so the absolute saving should hold while the curve term grows around it, `unmeasured`.

#### C. Correct the statements and add a graph-time warning (sizing guidance)

- **Single-machine memory:** `measured here` that partition count moves the flat peak both ways: 10 to 100 partitions raises 10M (2.55 to 7.54) and lowers 100M (12.48 to 8.59). Under A at 100M, more partitions cut the peak 3.3x at 1.8x the time, so partition guidance survives there as a one-sided tradeoff; those two points show no interior optimum. Flat's sizing rule is `unmeasured` and model-derived.
- **Cluster memory and transfer:** `unknown`. Guidance changes nothing by itself. At 10M rows per partition, flat needs about 6 GiB per worker for the gather, arithmetic.
- **Render time:** `measured here`: larger partitions were 1.9x faster on flat at 100M and 1.8x faster under A.
- **Exactness:** no raster change, `read-from-source`.
- **Maintenance:** `literature-supported` (finding 8). The graph-time warning applies to the flat path, where aggregate bytes are free to compute from partition count, pairs, canvas and reduction. The term moving A's peak is rows per partition, which isn't free on general Dask input, so this warning doesn't cover A. Auto-repartition goes beyond surveyed practice.
- **Dependence on the A decision:** under A, measured rows fit the notebook's "more partitions only lower the ceiling", and the byte formula overpredicts about 1.9x at 10M rows (10.13 against 5.34 GiB). Under flat, that sentence is false in the 10M direction and the correction must run both ways. Corrected docs written before the A decision may not hold after it.
- **Binding term at target:** none addressed.
- **At `num_steps=50`:** the formula's curve term triples, and the docs should state it at the shipped default.

#### D. Public-API tree through `to_delayed` and `dask.delayed`

- **Single-machine memory:** `measured here`, 2.18 GiB at 10M/100 (flat at 10 partitions 2.55, a 1.17x pointer). Above A's 1.76. Predicted to matter more with edges, by our model only.
- **Cluster memory and transfer:** `known mechanism of harm: the builder closure pickled per task (finding 5), unless combined with B`. Tree shape as in A. Fan-in is ours to set: up to 63.6 GiB at fan-in 8, arithmetic.
- **Render time:** `measured here`, 107.4 s at 10M/100. 2.3x flat's 45.7 s at a different partition count, a pointer.
- **Exactness:** `measured here`, 20k nodes / 400k edges, threaded, eight reductions, at most 4.4e-16. Multi-process, `ds.mean`, at most 5.6e-17 (P, not cluster). Moment rebuild uses raw `square/counts - mean**2` with a clamp, unlike the centred merge in the four libraries read, so the result may not hold for offset columns. Shipping needs an exact-type table with unlisted reductions on flat.
- **Maintenance:** `unmeasured`. A second aggregation path needing a named justification under the ecosystem-tools preference. Reuses `_accumulate_chunk_raster`. `to_delayed` stability is unverified.
- **Binding term at target:** conditional as in A, unmeasured at 10M rows.
- **At `num_steps=50`:** `unmeasured`. The curve term triples and aggregates do not change.

#### E. Fixed-size batches on the in-memory route (`min(batch size, what's left of the pair)`)

Outside the Dask question, carried from the maintainer's stated follow-up. It composes with every option above and shares the combine loop.

- **Single-machine memory:** `unmeasured`. Prediction: 1M-edge batches cap curve memory near 808 MB at `num_steps=100`. Edge table and ids remain in RAM, at 16 bytes per edge for two 64-bit columns.
- **Cluster memory and transfer:** `unknown`; this route doesn't use a cluster.
- **Render time:** `unmeasured`.
- **Exactness:** `unmeasured`, with rounding expected on float reductions.
- **Maintenance:** `read-from-source` at branch 53. `_curves_from_id_pairs` takes arbitrary ids and the loop already combines.
- **Binding term at target:** RAM for the edge table, so this route doesn't reach the target.
- **At `num_steps=50`:** the same batch caps near 408 MB by arithmetic, `unmeasured`.

#### F. Collapse before expanding (per-pair binning, weighted curves)

Power-user only, never a default, if questionnaire item 2 allows it.

- **Single-machine memory:** `literature-supported` in shape only (finding 12); no number for this route. Node table does not move.
- **Cluster memory and transfer:** `unknown`.
- **Render time:** `unmeasured`. Still scans every edge.
- **Exactness:** `unmeasured`. Fails equivalence by design. The static-raster cost has no source and must be measured: Rübel's aliasing sentence concerns zoom on prior fixed 256 x 256 work, asserted and not measured. `first`, `last`, `by` have no obvious per-bin form (inference).
- **Maintenance:** `unmeasured`, large.
- **Binding term at target:** node table.
- **At `num_steps=50`:** draw cost scales with `num_steps` and non-empty bins, about 1.3 GiB against about 0.43 at 16 if every bin is non-empty (unverified lens arithmetic), `unmeasured`.

#### Dropped from the ranking

- **Fold, not tree.** Per-worker accumulators need `distributed`, a serial fold gives up parallelism (`literature-supported`, dask-ml and xgboost), and no measurement favors it. A lock-based threaded fold is unverified.
- **Internal graph rewrite.** Patches datashader internals, and A reaches the same shape with public arguments.
- **`axis=1` form of datashader's tree.** List-of-one reaches the same path without 2(num_steps + 1) columns or a per-row antialias clear. Unverified from source and not run.
- **An upstream PR as a prerequisite.** A and D need no upstream change.

## What was validated, and how

Outcome at the convergence gate: validated finding, with the cluster half validated inconclusive. Five disjoint lenses, blind to each other (prior art outside Dask, counter-evidence, combine practice, sizing, process boundary), nominated claims. Each nominated claim went to an independent voucher, default refuted, with a second voucher on self-flagged single-quote claims. A cold adversary then ran the gate against the failure-mode rubric, which the orchestrator seeded and the maintainer has not ratified (questionnaire item 5), since the grill was knowingly skipped.

- **Validated as corrected by their vouchers:** findings 2, 3, 4, 5, 7, 11 and 13.
- **Corrected at the gate:** finding 1 no longer implies the combine explains A's gain. Finding 6's mechanism stands and its figures are recorded one-run measurements. Findings 8, 9 and 10 gained scope tags, state sizes, ratios and versions. Findings 2 and 3 gained versions. Finding 12 reads as a precedent. Finding 14's removed single-worker gather stands, with flat's byte upper bound above both tree figures.
- **Ranking:** the gate replaced a combined default recommendation with the gated ranking above and required the statements on `num_steps=50`, on where sizing is unavailable for A, on the equivalence gates, and on the conditional binding term.
- **Verdicts:** 7c regrounded on a named search scope, 7d regrounded on A's rows, and the `var`/`std` framing restated.
- **Measured inputs:** no lens produced a measurement. A's rows came from `list_of_one_probe.py`, run outside the lenses after a lens surfaced the route from source, and the plan's `## Inputs` does not record them. The tokenization figures came from `tokenize_probe.py`.

### Not-fully-confirming evidence

No hit killed a candidate story. These cut against the ranking or the findings and stay attached to them:

- **shadeMS #34, tree against flat:** read in full, it never shows the tree doing worse. Parity holds only at chunk size 20,000, the largest working chunk, and oversized chunks exhaust RAM with or without the tree. Its author later wrote the tree is "always better or equivalent" on RAM and runtime, which conflicts with an earlier "faster but hungier" caption. The benchmarks come from the PR's author on a sometimes contested box.
- **Coarse maintainer rule:** "chunk size >> output array size" predates the tree.
- **rechunker #100 (2021):** `max_mem` is not a runtime guarantee.
- **SPARK-26881 (2019), against finding 11:** uses depth to bound result volume at the driver, and the issue's "data from each partition is big" ties time to payload.
- **#6232, against finding 13:** a trim barely helps.
- **RSS and the 100M gain:** allocator retention can't be excluded from the roughly 5 GiB of A's saving that aggregates and stitching don't account for.
- **Tree libraries as independent evidence:** five of the six are Dask-family (finding 10).

## What was inconclusive, and why

### The cluster half

**Asked:** what each option does to cluster memory and transfer, and whether a tree or the closure helps or harms there.

**Searched:** `distributed` docs and source at tag 2026.7.1, dask #11026 with all comments, distributed #9077, `scheduling-policies.rst`, `scheduler.py`, flox #346 and #396, xgboost's non-broadcast scatter, datashader issues and discourse. Trees on real clusters for fixed rasters: none found, search-scoped.

**Read from source, never run:**

- **4. A flat combine must fit in one worker's RAM at execution; spilling does not change that.** C, 2026.7.1. `worker.py:2140-2148` loads all dependencies into one dict, reading spilled ones back. The placement objective has no capacity term, except that workers paused at 80% are excluded. At 95% of the limit the nanny terminates the worker. Allowed failures is 3, so 4 attempts in total. That this workload dies on every retry is inference. Transfer: 5.57 GiB at 600 aggregates and 55.7 GiB at 6,000, both arithmetic. The final task pulls every raster onto one worker, so holding everything until the end has no support from source as a cluster advantage (inference from a graph never run).
- **5. The builder closure is pickled per task from scheduler to worker, and each worker holds its own copy.** C, 2026.7.1. No function cache sits on the task path, and `ts.run_spec` is retained until the task state is released. Client to scheduler is one pickle of the expression. Queuing bounds copies sent at once only for rootish groups, and it does not bound copies alive.
- **7. A shared dependency is a supported way to ship the node table, and `scatter(broadcast=True)` conflicts with the default active memory manager.** C, 2026.7.1. A `Delayed` argument to `map_partitions` becomes one shared task (`_expr.py:692-694`, `_blockwise_arg`). `ReduceReplicas` is on by default and documented as incompatible with `broadcast=True`. No document names this pattern for `map_partitions`.
- **14. A tree removes the single-worker gather.** C, source plus arithmetic. Line tasks fetch at most one frame each through downstream placement, and flat's combine pulls every aggregate to one worker. At 1B edges and 1M rows per partition, flat's total transfer upper bound is about 139.7 GiB (84 of frames plus a 55.7 gather), against up to 63.6 GiB at fan-in 8 and 111.3 GiB at fan-in 2 for a tree.

**Tree harm at raster scale:** no source shows or excludes it. Dask #11026 concerns about 600 MiB intermediates inflated by flox `reindex=True`; maintainers called it not a dask problem, and plain `X.sum` ran fine on the same array. Queuing caps leaf start-ahead at about nthreads + 1 per worker, read from source. No source tests about 10 MiB, and a pro-tree slide deck reports the opposite anecdote.

**Why the evidence fell short:** nothing has run on `distributed`, and no option's cluster memory is known. The `processes` runs are a 20k-node single-machine stand-in with no memory figure. Threaded RSS does not transfer (finding 13). More reading will not settle this. Experiment (d) is the next evidence.

### 7a, serialization outweighs tree shape: inconclusive

The mechanism is validated (findings 5 and 6), and on bytes the closure exceeds the combine by arithmetic: floor about 357 GiB against 55.7 GiB at 2M nodes and 6,000 tasks. The kinds of failure differ, slowdown and multiplied RSS against a hard kill, and neither has been run. Evidence is P at 20k nodes at most. The fixes are independent.

### 7b, nobody renders graphs on a cluster: inconclusive

**Searched:** datashader issues and discourse, dask and distributed trackers, tool docs. Cluster datashader use exists for points and polygons (#668, #332, discourse 4168, all unverified). No report of graph or edge rendering on a cluster was found. This is an absence in these searches only.

### 7c, billions not worth drawing edge by edge: inconclusive

**Searched:** Graphistry's 2.53.0 and GFQL posts, the RAPIDS cuGraph with datashader post, Cornac, Zinsmaier et al. and graph-tool 3.9 docs (all unverified, several snippet only). No precedent found for drawing billions edge by edge. That those tools collapse first is unverified, and Rübel and imMens are not graph tools. Large-graph tools carry layout and interactive costs a fixed-layout static raster does not, so even a confirmed norm would make the route unusual without making it wrong.

## What's open

### Experiments

None run by the lenses. All belong in `hiveplotlib-performance`.

**Run 2026-10-31, after this page was saved:** (e), (f), (h), (i), (k) and most of (j), five samples each, threaded, on one machine, recorded in `synthetic_scaling/dask_partition_memory/RESULTS.md`. A held its gates: same lit pixels as flat at every grid row, 1.77 to 1.97 GiB at 1M edges per partition from 10M to 100M edges, 3.22 GiB with every edge in one axis pair, and it finishes `num_steps=50` at 100M edges / 10 partitions, where flat passes 26 GiB. `first` and `last` are exact on A and not on flat. `by` and `where` fail on every route including single-shot, so the library's "every reduction" claim is false independently of A. The allocator setting in (e) costs 7 to 10% render time for 10 to 13% lower peak. Parquet at memory scale and (d), (g), (l), (c) remain open.

**Run 2026-11-01 and 2026-11-02, lazy-node lookup past one machine's memory, and (d) in part:** five samples each, threaded and on a `LocalCluster` with worker memory limits (`distributed` 2026.7.1 in a scratch environment), integer ids, 100M edges unless stated, same `RESULTS.md`. Every finished row matched the other routes' raster hash at its size. Three lookups were added beside copy and shuffle: the copy route's 33-byte table as files on disk, built by an external bucket sort that never holds it (`memmap`); the same lookup mapping one slice at a time (`memmap_chunked`); and a partitioned join that sorts endpoints into the table's id-range buckets and resolves each bucket against its slice with sequential reads (`bucketed`).

- **Fits in memory:** `memmap` costs 1.1 to 1.3x copy's build plus render at 100M to 300M nodes, at 2.4 to 2.7 GiB anonymous peak against copy's 9.0 and 25.7. `bucketed` costs about a third more than `memmap` (238.5 against 177.6 s at 300M).
- **Past memory:** under a 4G cap at 300M nodes (a 9.2 GiB table), `memmap` slows 13x to 2,345 s and `bucketed` slows 20% to 286 s. At 1B nodes on the full machine, `memmap` takes 2 h 15 min and `bucketed` 28 min, most of that the table build and the bucket passes. Copy and the threaded shuffle were killed at 8G and 4G and passed 26 GiB at 1B, so the shuffle is not a never-fails fallback on one machine.
- **Cluster:** `memmap` had every worker restarted, because RSS counts the mapped table once per thread and the nanny acts at 95% of the limit. `memmap_chunked` fixed that with no restarts and workers under 1.2 GiB, and was fastest in all three setups: 99 s at 100M nodes and 232 s at 300M on 4 workers of 6 GiB, 55 s on 8 workers of 3 GiB, against 144, 555 and 169 s for the p2p shuffle. `bucketed` was close at 300M. Copy failed in every setup, partly because this probe gathers the table through one worker.
- **1B edges at 100M nodes, copy:** 180.5 s at `num_steps=16` and 334.3 s at 50, peaking at 8.9 to 9.0 GiB, the node table. Per 1M-edge partition on one thread, `cvs.line` takes 0.75 s at 16 and 1.51 s at 50 against 0.42 and 0.57 s for the whole curve build, so datashader's aggregation, not the lookup, is the largest cost at this scale.

Unmeasured: a real multi-machine cluster. Every on-disk lookup needs every worker to read the same files, which holds on one machine or a shared filesystem and not on a plain cluster, where the p2p shuffle is the only route measured.

- **(j) A across `first`, `last`, `by`, `where`, offset-valued columns, Parquet input and skewed edges (all in one axis pair).** Gates A. Settles exactness and whether frames or rows set the curve term.
- **(e) Five samples plus a held-live measure, with one row rerun at `MALLOC_TRIM_THRESHOLD_=65536`.** Gates A. Settles the floor, the unattributed saving and the first rubric entry.
- **(f) Precomputed per-axis lookup in a throwaway copy,** at 2M and 20M nodes with raster comparison, including ids missing from an axis. Gates B.
- **(d) `LocalCluster` with a worker memory limit,** flat against A against D, at 2M nodes. First cluster-scope evidence and 7a's cost at size. Needs `distributed` in a scratch environment (questionnaire item 4).
- **(h) The 2M / 100M / 10 row, and A, at `num_steps=50`.** Tests the curve term at the signed default and the 16 GiB prediction.
- **(i) A's remaining rows.** A at 10M/1, plus five samples with flat and A in the same session.
- **(k) A against the 20M / 10M / 10 row.** Tests the combined memory at the binding term.
- **(b, remainder) Whole-render attribution at 20M nodes / 10 partitions and 2M nodes / 100 partitions,** memory included. Settles how much of the 475 s and 3.67 GiB the merge explains, and why time grows with partition count.
- **(g) Flat near the predicted optimum against A at 100M edges.** Tests the model and A's about 3x, with no 1B run.
- **(l) Static-raster error of per-pair binning against an exact raster.** Settles option F's exactness.
- **(c) `axis=1` with one row per edge.** Low priority now that A exists.

### Questions

- The run proceeded on each questionnaire item's recommended answer. Two have since been overruled (see "Ranked options"): (1) the 20M-node target, with the new node target still unset, and (7) this blocks branch 53, and C's docs depend on the A decision. Still open: (2) whether lossy collapse enters the ranking; (3) how to treat a best answer inside datashader, with the maintainer's direction that nothing is filed yet; (4) `distributed` in a scratch environment; (5) whether the seeded rubric stands; (6) effect order.
- How nodes scale past one machine's memory. The shipped Dask notebook says node data stays in memory, and `Axis.node_placements` is a pandas table. Build peak at 20M nodes works out to about 306 bytes per node above the 2M row (one sample, unattributed), which predicts about 29 GiB at 100M nodes by linear extrapolation. The maintainer ruled on 2026-10-31 that nodes get a lazy, datashader-only route on branch `53`, added beside the in-memory path. Unsettled: how each edge finds its two nodes on that route (axis membership at `hiveplot.py:1409`, coordinates at `hiveplot.py:1749`), by a copy on every worker, which caps at worker memory, or by a shuffle, which adds machines at the cost of moving 1B edges twice. Option B's "nodes stay in memory" no longer describes the target. Both ran on 2026-10-31 on one machine, threaded, with integer ids and no library edits, and every route matched today's raster hash. At 100M edges a 33-byte-per-node table renders 300M nodes in 33.1 s but peaks at 25.70 GiB while building a 9.2 GiB table. A disk shuffle with the nodes read separately for each merge peaks at 15.10 GiB and takes 764.8 s, growing about 29 bytes per added node for a cause not identified. Today's route passes 26 GiB at 100M nodes. So neither route showed a path past this machine's memory on 2026-10-31. The 2026-11-01 and 2026-11-02 runs above found two that do on one machine, an on-disk table and a bucketed join, and a cluster form of the on-disk table that beats the p2p shuffle on a `LocalCluster`.
- What sets the roughly 1.8 GiB floor under A, which holds only at 1M rows per partition or fewer, and why time grows with partition count.
- What explains about 4.5 GiB of A's saving at 100M/10. Aggregates and stitching account for about 1.7 GiB, and allocator retention can't be excluded. Tied to (e) and (j).
- Whether real demand for edges at this scale exists.
- Why datashader's line path stays flat. No maintainer statement specific to it was found; #464 explains why stitching exists.

### Unverified context

Beyond the voucher pool, carried as pointers and not as findings.

- Splitting a raster across workers, as sort-last rendering does (Rabenseifner CUG 2004; IceT SAND2011-2482C). It adds tasks and still ends in a full gather.
- GeoSparkViz at 1.7 billion points; vaex's per-thread state; MLlib histograms sized through `maxMemoryInMB`.
- Graphistry, cuGraph, Cornac and graph-tool positions on billions of edges.
- HEP accumulators (Fischer 2024); dask #9083 (tree floor is threads x (fan-in + 1) x state; a locked accumulator completed on threads); dask #9522 (default of 8 criticized for groupby); discourse 4168 (workers died at 1.6e9 pixels on the tree path); datashader #1514 (kernel speedups in progress).
- Fragility of `default`: `use_array_expr` "Eventually, this default will flip to True" against datashader's manual `da.Array` construction; compatibility commits #1317, #1335, #1341, #1390; spatialpandas #178 (points from `read_parquet`, closed 2026-10-01). The route's probes use only `from_map`, so Parquet input is untested.
- `to_delayed` surface concerns (dask #12318, `ProhibitReuse`, about `da.Array`) and uncached numba compile per new worker (datashader PR #1515).
- The scheduler event-loop chain, sum-versus-max in the memory model, and the three-term arithmetic's frames-versus-rows puzzle.

## Sources

Versions and locations only; the run's per-claim quotes are not reproduced here. Unverified sources sit under `What's open` with that label.

- **Measurements:** `~/repos/hiveplotlib-performance/synthetic_scaling/dask_partition_memory/RESULTS.md` (run 2026-10-30, branch 53 at `ca42c53`, datashader 0.19.1); `dask_scale_probe.py`, `public_tree_probe.py`, `list_of_one_probe.py`, `node_lookup_profile.py`, `tokenize_probe.py`.
- **datashader 0.19.1:** `core.py:418-420`, `glyphs/line.py:106`, `data_libraries/dask.py`, `reductions.py:1153-1157`, `compiler.py:478-504`, `compiler.py:492-493`; #464.
- **dask 2026.7.1:** `_expr.py:692-694`, `_blockwise_arg`, `map_partitions`, `_normalize_pickle`, `moment_combine`, `moment()` docstring; stable docs read 2026-10-30; #11026.
- **distributed 2026.7.1:** `worker.py:2140-2148`, `distributed.yaml`, `scheduler.py`, `scheduling-policies.rst`; #9077.
- **Other fixed-output and sizing libraries:** boost-histogram `mean.hpp`; flox `_var_combine`, #346, #396; dask-histogram; dask-ml; xgboost; Cubed docs; rechunker #100; dask-cudf 26.08; shadeMS #34.
- **Outside Dask:** Rübel et al. SC08; imMens, EuroVis 2013; Spark v3.5.1, SPARK-2174, PR #1110, SPARK-26881, 2014 Databricks blog.
- **Allocator:** `MALLOC_TRIM_THRESHOLD_` man page; glibc 2.39 source; issues #5971, #6232.
- **This repo, branch 53:** `tests/dask_test.py:451,491,610`; `src/hiveplotlib/viz/datashader.py:761-764`; `_curves_from_id_pairs` in `hiveplot.py`.

## See Also

- [[hiveplotlib]]: the library and its scaling milestone
- [[0002-performance-regression-harness|ADR 0002]]: the equivalence wall any raster-changing option answers to
- [[specs/lower-default-curve-resolution|Lower the default curve resolution to 50]]: the signed default that rejected `num_steps=16`

---
*Produced by a bounded literature-mode research run, 2026-10-30. 31 of a binding 31 Agent calls, counting this landing; peak concurrency 8 of a ceiling of 8; about 4.4M tokens (task-notification subagent figures, which may understate spend, because an earlier telemetry study found them closer to peak context than total spend).*
