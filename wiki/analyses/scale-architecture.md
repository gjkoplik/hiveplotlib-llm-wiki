---
title: "What It Takes to Render a Billion-Edge Hive Plot"
type: analysis
created: 2026-11-02
updated: 2026-11-02
sources: [hiveplotlib-python-repo, hiveplotlib-performance-dask-partition-memory, scale-architecture-research-run-2026-11-02]
tags: [hiveplotlib, datashader, dask, scaling, out-of-core, node-table, duckdb, gpu, research-run]
---

# What It Takes to Render a Billion-Edge Hive Plot

Nothing in this run shows the 1B-edge, 100M-node target needs a new architecture or a language switch, and nothing measures the target either; the one new measurement is that at 1B nodes a bucketed sequential join cuts the 100M-edge lookup-and-render from about two hours to about eleven minutes (n=2, in flight), behind a 14 to 15 minute table build both routes share, while the target itself stays a 210-second prediction at num_steps 16 until the queued rows land. (Corrected 2026-11-02 from "a minute behind a 25-minute build": the bucket passes, about 10 minutes, are part of the lookup work, not the table build; end to end, 2 h 15 min becomes 28 min, about 4.9x.)

**Post-gate update, 2026-11-02 (dispatching session; the convergence gate did not read these rows).** The queued rows landed the same afternoon. The target's edge render is now measured on the copy route, 100M nodes, 1B edges, 1,000 partitions of 1M edges, five samples each: median 180.5 s at `num_steps=16` and 334.3 s at 50, peak RSS 8.9 to 9.4 GiB, one raster hash per setting across all five samples. Both pass the ten-minute batch threshold on edge render alone and both fail the one-minute interactive threshold; the storage read is still unmeasured. The per-mechanism breakdown shows datashader's line aggregation as the largest single-thread term (64% at 16, 72% at 50), with lookup and gathers at 24% and sampling at 10%. The `bucketed` row reached n=5 at 1B nodes: bucket passes median 587 s plus edge render median 77.1 s, about 11 minutes of lookup work against memmap's 2 hours, after a shared 14 to 15 minute table build. Story 8 is `measured here` for the edge render, and architecture A takes first place at the batch threshold below. Details in `### Landed after the gate`.

A bounded research run (2026-11-02, literature mode, working plan `wiki/wiki/plans/scale-architecture-research.md`) asked what architecture lets [[hiveplotlib]]'s datashader route draw a static 1500 x 1500 raster for about 1B edges and 100M nodes (1B nodes as a stretch row), how fast each goes, and whether the answer lives inside Python. Everything upstream of the raster was in scope; the 2026-09-30 ruling fixes only the renderer. The prior run is [[dask-route-memory-bounding]].

Verdict: **half validated finding, half validated inconclusive.** Validated: the mechanisms below, verdict 9b-ii, and verdict 9c narrowly (n=2 of a planned 5, in flight, 100M edges). Validated inconclusive: the ranking at the target (first place withheld; C, E and F times unmeasured; no lazy-route row at `num_steps=50`; no storage read), 9a, 9b-i at batch, and 9d.

**Judged against** the questionnaire's recommended answers, none yet answered: 3 axes at `num_steps=50` (the [[specs/lower-default-curve-resolution|signed default]]), 16 as power-user. Interactive fast is the whole render, storage reads included, in about a minute on ringtail; batch fast is under ten minutes; feasible is a few hours on rentable hardware. Exactness is a floor for any default, then fit, time, dependency surface. Lossy variants are power-user only.

**Probe scope, unless stated:** ringtail (24 cores, 30 GiB visible to WSL, SSD of unknown model behind WSL2's VHDX), 8 threads, threaded scheduler, `num_steps=16`, uniform random ids, `ds.count()`, edges generated in memory, medians of five, edge render only. No lens ran a probe. VmHWM is peak RSS including resident file pages; anon is anonymous RSS; live is glibc `malloc_info` in-use bytes, excluding mmap and Arrow pools. File peak counts each thread's mapping of a file once, so the memmap rows' 27, 77 and 207 GiB are an instrument artifact, not memory need, and the column is omitted.

**Marks.** `measured here` names a probe row; `predicted`, the rows its arithmetic uses; `literature-supported`, a validated finding; `read-from-source`, a vouched version and file. `analytic` is a lens derivation no voucher re-derived and nothing measured, ranking below `literature-supported`. `unmeasured` is the rest, including cells resting on unverified claims. Exactness against [[0002-performance-regression-harness|ADR 0002]]'s wall: `exact`, `rounding-only`, `lossy, bounded`, `lossy, unbounded or unmeasured`.

## What was established

### Measured inputs

Seconds; GiB with instrument. Every route gave the same raster hash; against the library at 20k nodes / 400k edges, count and any are bit-exact and sum, mean and var agree within 4.4e-16. Today's build costs 306 bytes per node; the copy table, 33.

| Route | Nodes / edges | Build | Edge render | Memory | Disk | Source |
|---|---|---|---|---|---|---|
| library_list | 100M / 100M | stopped at a 26 GiB cap after 215 s | | live 18.55 | | RESULTS.md |
| copy | 2M / 100M; 2M / 300M | 2.8; 3.8 | 17.0; 50.7 | VmHWM 2.68; 2.81 | 0 | RESULTS.md |
| copy | 100M / 100M | 11.5 | 21.0 | VmHWM 8.96, live 7.03 | 0 | RESULTS.md |
| copy | 300M / 100M | 119.7 | 33.1 | VmHWM 25.70, live 23.05 | 0 | RESULTS.md |
| copy | 1B / 100M | aborted above 26 GiB at 45 s | | | | RESULTS.md |
| shuffle | 100M; 300M / 100M | 4.2; 7.4 | 121.9; 764.8 | VmHWM 11.17; 15.10 | 11.4; 28.0 | RESULTS.md |
| memmap | 100M / 100M | 11.0 | 31.3 | live 1.42 | 4.2 | memmap_fit.jsonl |
| memmap, cgroup cap none / 16 / 8 / 4 GiB | 300M / 100M | 107 to 126 | 68.9 / 49.3 / 579.3 / 2219.5 | live 2.1 to 3.3 | 18.0 to 18.4 | memmap_caps.jsonl |
| memmap, n=4 | 1B / 100M | 851.5 | 7265.7 | anon 7.1 to 7.5, live 7.01 | 61.5 | memmap_huge.jsonl |
| bucketed, n=2 of 5, in flight | 1B / 100M | 1492, 1584 (table bucketing 594, 574) | 58.1, 77.1 (node render 22.4, 28.3) | VmHWM 8.16; anon 7.66, 7.99; live 4.87, 7.60 | 61.47 | bucketed_huge.jsonl |
| LocalCluster shuffle, one config | 100M; 300M / 100M | | 140.5; 549.4 | total PSS peak 4.1, 13.9 spilled | | cluster.jsonl |

`copy` holds an id-sorted table once and looks up with `np.searchsorted`. `shuffle` is two Dask disk-shuffle merges. `memmap` is the copy table in files, read at random through the page cache. `bucketed` id-sorts the node table into id-range bucket files, scatters each partition's endpoints into the same buckets, and resolves each bucket in one sequential pass per endpoint; its hash matches memmap's. LocalCluster is one machine, 4 workers x 6 GiB, 2 threads each; copy and memmap died there.

### Mechanisms (validated)

In each voucher's corrected wording.

- **F1.** Neither numba Bézier sampler has `nogil=True` (`utils.py:172`, `:209`); the Dask route forces the serial one (`viz/datashader.py:797-804`) or, under `use_numba=False`, a NumPy sampler whose per-curve loop holds the GIL. Sampling serializes on the threaded scheduler, inferred from code. The pandas merges in `_curves_from_id_pairs` have mixed GIL behavior. That the copy probe calls the serial sampler is a hub read, unvouched.
- **F2.** datashader 0.19.1's default CPU line path is zero-width Bresenham in one `nogil` loop per frame, no `prange` (`glyphs/line.py:1033-1041`, `:1126-1136`); parallelism comes only from partitions. Cost is pixel-bound while segments span a pixel; each sub-pixel segment still appends once, so oversampling inflates `count()`.
- **F4.** datashader 0.19.1 has CUDA paths for sum, mean, var and std (`reductions.py:986-1330`; `reduction.csv` marks all four "yes" on GPU); cuDF input selects a CUDA line kernel; the only CUDA-conditional raise is antialiased `sum(self_intersect=False)`. Static reading.
- **F5.** distributed 2026.7.1's nanny terminates a worker above the configurable 95% of `psutil` RSS, which on Linux is VmRSS, "the sum of RssAnon, RssFile, and RssShmem" (man-pages 6.19). Nothing subtracts file pages; #4304 is open.
- **F7.** A two-pass Grace hash join costs 3(M+N) page I/Os (one partitioning pass, no overflow, output excluded) with about sqrt(build pages) buffers; hybrid hash does fewer (UNSW COMP9315 24T1; Shapiro 1986). Sequential I/O is not sourced.
- **F9.** RocksDB 7.2.2 IO-bound `readrandom` did 137K (buffered) and 189K (direct I/O) ops/s on an m5d.2xlarge (8 vCPU, 32 GB, one NVMe; vendor, 2022, stale) across about 64 readers, 2.1K to 3K/s on one thread: 2B lookups in 2.9 to 4.1 h, highly parallel only.
- **F10.** DuckDB has spilled larger-than-memory joins since 0.6.1, degrading gracefully (Kuiper et al., PVLDB 18(8), 2025) from 1.2.0; multiple blocking operators may still OOM. Its 50 GB db-benchmark join passed on a 16 GB M1, pass/fail, once, untimed.
- **F12.** DuckDB 2.0's docs give 3-4 GB per thread for join-heavy work as an approximation for ideal in-memory performance, about 7 to 10 threads at 30 GiB before spilling is likely. Not a cap.
- **F14.** As of Polars 2.1.0 (2026-10-25), group-bys and sorts spill but joins hold state in memory, so Polars cannot join against a node table larger than RAM.
- **F17.** Same-degree Bézier curves differ pointwise by at most their largest control-point difference (arXiv 2201.07834, Prop. 2), so rounding rho to h-pixel bins moves every sample at most max(1, s) h/2 px, s the control rho scale. The code assumptions (`hiveplot.py:1806-1808`) are hub reads, unvouched.
- **F18.** Novotny and Hauser (IEEE TVCG 12(5), 2006) bin straight lines per adjacent axis pair in screen space, drawing outliers as polylines, at a cost tied to display size. Abstract and snippets only.
- **F19.** Graph Cities (IEEE CG&A 42(3), 2022) shows graphs up to the 1.8B-edge Friendster as buildings, one per maximal connected subgraph; its site says about 2 hours on 64 GB with no dataset named, its abstract minutes excluding I/O. Whether bottom-level edges are drawn is unverified.

On the GIL overall: the rasterizer is compiled and `nogil` (F2); numpy `searchsorted` releasing it is unverified (numpy main, not the installed tag); the merges are mixed (F1).

### Points versus segments

**F16** (validated): datashader maps a line "into every grid cell falling between this point's location and the next" and a point to one cell (0.19.1 docs). D-exact, D-quantized and C feed `Canvas.line` with a weight column (C when it hosts D's GROUP BY), so segments are preserved. A points glyph or a GROUP BY over sample points is out: matching needs consecutive samples in the same or neighboring cells, a necessary and not sufficient condition (F16), and raising `num_steps` to get there inflates `count()` through sub-pixel appends (F2). D-sampled feeds `Canvas.line` unweighted and scales by 1/p.

### Correction to the prior record

`plans/scaling-large-networks.md` records that sum, mean, var and std "raise upstream" on cuDF. At the datashader level that is contradicted: F4 rests on two independent voucher reads (the 0.19.1 support matrix and release notes; the 0.19.1 source), neither finding a raise for those four outside antialiased `sum(self_intersect=False)`. The maintainer holds a machine-verified diagnosis outside the wiki, a CUDA 12.9 NVVM float-atomic fault (numba-cuda #384, fixed on CUDA 13 bindings; unverified here), which fits count, any, max and min working, since none needs a float atomic add. **Suspected toolchain fault until the maintainer confirms it here.**

### Ranked architectures

**A. Lazy-node copy table, one machine.**
- Takes: a 33-byte-per-node id-sorted table, `np.searchsorted` per partition, no new dependency; user frames untouched, so pass-through and no-opt-in hold (`measured here`). Factorizing object ids `unmeasured`.
- Reaches: 100M / 100M in 21.0 s at VmHWM 8.96 GiB (`measured here`). Target: 180.5 s at 16 and 334.3 s at 50, VmHWM 8.9 to 9.4 GiB, edge render only, n=5 each (`measured here`, post-gate; the 210 s prediction was 16% high). N1 adds that 8-to-24-thread scaling is unsupported. 300M nodes peak at 25.70 of 30 GiB; 1B aborted (`measured here`). Fails interactive, passes batch at 16 and at 50, on edge render alone.
- Storage: `unmeasured` (16 GB of int64 edge ids). Exactness: `exact` (`measured here`).
- Branch: 53; decides the lookup up to about 300M nodes.
- Binds: datashader's line aggregation, 64% of single-thread time at 16 and 72% at 50 (`measured here`, post-gate breakdown, one 1M-edge partition); node memory past 300M nodes.

**B. Out-of-core node table, partitioned sequential join (`bucketed`).**
- Takes: id-range-bucketed, id-sorted node files, each bucket resolved in one sequential pass; no new dependency (`measured here`). Bespoke, so the ecosystem rule needs a named reason C cannot do this join.
- Reaches: A at 100M nodes. 1B nodes / 100M edges: edge render 77.1 s median (58.1 to 134.6) after bucket passes of 587 s median (540 to 663), VmHWM 8.16 GiB (`measured here`, n=5): the lookup work is about 11x faster than memmap's 2-hour render, and end to end (table build plus passes plus render) 28 min against 2 h 15 min, about 4.9x. 1B / 1B: the bucket passes scale with node buckets times edge partitions (about 2M small appends per pass at 1,000 buckets and 1,000 partitions, against 200k measured), so a linear extrapolation gives about 98 minutes of bucket passes plus the build unless the bucket count is decoupled from the node partitions and sized to a memory budget (`predicted`, untested either way; the first knob an implementation needs). Fails both thresholds, feasible.
- Storage: 61.47 GiB of node files read sequentially (`measured here`); edge read and scatter disk at 1B edges `unmeasured`.
- Exactness: `exact` against memmap, count only (`measured here`, n=2).
- Branch: 53 for the past-memory path unless C replaces it.
- Binds: at 1B nodes, the table build (about 14 to 15 minutes, shared with memmap) and the bucket passes (about 10 minutes at 100M edges, growing with edges) (`measured here`, n=5).

**C. DuckDB join feeding datashader in batches.**
- Takes: an optional extra. narwhals' LazyFrame has no batch exit, so the hand-off needs a native branch per lazy backend, bending pass-through (`unmeasured`, F22). Pandas batches render serially (F2), needing a pool or Dask.
- Reaches: 100M nodes `unmeasured`. At 1B nodes spilled joins exist (`literature-supported`, F10), with a two-join query's OOM risk and about 7 to 10 threads before spilling (F12). Time `unmeasured`.
- Storage: `unmeasured`. Exactness: `lossy, unbounded or unmeasured`; count should stay exact (`analytic`).
- Branch: later; 53 forecloses it only if the lookup is not one replaceable internal function.
- Binds: `unmeasured`, likely spill I/O at 1B nodes.

**D-exact. Deduplicate on (ordered pair, rho_from, rho_to), draw weighted.**
- Takes: a GROUP BY after the join, weighted `Canvas.line` with `ds.sum`; composes with A to C.
- Reaches: gains only where positions repeat; continuous sorting variables give about one curve per edge (`analytic`); real counts `unmeasured`. Scan and join stay.
- Storage: its host's.
- Exactness (`analytic`, unvouched): count and any exact given one sampler for both paths, an ordered-pair key, float64 weights and NaN filled to 0. max and min need group-carried values; `by` needs the category in the key; first and last need a carried row index read back through `ds.where`, which its docstring may block. sum and mean `rounding-only`; var and std carry cancellation risk, `unmeasured`.
- Branch: 53-compatible, never foreclosed. Binds: its host's join.

**D-quantized. Bin rho at sub-pixel width.**
- Takes: the GROUP BY on binned rho, weighted `Canvas.line`; a new ADR 0002 metric.
- Reaches: at most pairs x B^2 curves (13.5M at 6 pairs, B = 1500, `analytic`); scan, join and a 1B-row GROUP BY remain, speed `unmeasured`.
- Storage: its host's.
- Exactness: `lossy, bounded` in geometry at max(1, s) L_px/(2B) px, 0.25 px at B = 1500, L_px = 750, s = 1 (`literature-supported`, F17); no per-pixel count bound (`analytic`). Fails the wall's curve-array half by construction and, `predicted`, its raster half in dense regions.
- Branch: later, power-user. Binds: scan, join, GROUP BY.

**D-sampled. Bernoulli edge sample at fraction p.**
- Takes: a filter before the join, unweighted `Canvas.line`, counts scaled by 1/p.
- Reaches: at p = 1%, about 2 s of render plus the 11.5 s build at 100M nodes (`predicted` from the 100M / 100M row); the 1B-edge scan stays.
- Storage: `unmeasured`, and it binds.
- Exactness: `lossy, bounded` (`analytic`): per-pixel relative SE sqrt((1-p)/(pC)); a pixel of C edges vanishes with probability (1-p)^C, 0.37 at C = 100. count, sum and by(count) unbiased; the rest biased or lost.
- Branch: 53 or later, power-user. Binds: reading 1B edges.

**E. Compiled core, language switch or fused kernel.**
- Takes: a fused numba kernel reimplements datashader's private clip, snap, Bresenham and appends, making it a second rasterizer. The 2026-09-30 ruling keeps datashader the renderer, so E reopens it, and the ecosystem rule needs a named reason (the curve-memory cliff at 50 is the candidate, `unmeasured`); the ecosystem path is a curve glyph contributed upstream. Rust or C++ adds platform wheels and a build chain.
- Reaches: `unmeasured`; "about a minute on 24 cores" rests on the unverified F21.
- Storage: `unmeasured`. Exactness: `lossy, unbounded or unmeasured`; exact only if it copies datashader's rasterizing bit for bit.
- Branch: later, nothing foreclosed. `nogil=True` on the samplers (F1) is a branch-53-sized fix and is not E.
- Binds: `unmeasured`.

**F. GPU in datashader's cuDF path.**
- Takes: cuDF and dask-cuda; CUDA paths exist (`read-from-source`, F4); a toolchain past the float-reduction failure recorded in `plans/scaling-large-networks.md` (suspected CUDA 12.9 NVVM fault, numba-cuda #384, unverified).
- Reaches: time `unmeasured`. 3.3 GB of table at 100M nodes fits 24 GB; 33 GB at 1B needs 48 to 80 GB (`analytic`).
- Storage: `unmeasured`. Exactness: `lossy, unbounded or unmeasured`; float atomics make sum and mean rounding-only at best (`analytic`).
- Branch: later, not foreclosed. Binds: device memory at 1B nodes (`analytic`).

**G. Cluster.**
- Takes: `distributed`; datashader's dask path already sums partition rasters.
- Reaches: shuffle 140.5 s at 100M / 100M, 549.4 s at 300M nodes (`measured here`, LocalCluster, one machine, one configuration); target about 23 minutes (`predicted`, linear), failing both thresholds.
- Storage: `unmeasured`. Exactness: `exact` (`measured here`). Branch: later, not foreclosed if the lookup runs on Dask frames. Binds: the shuffle.
- Cluster cells. copy: `known mechanism of harm: a repartition-to-fewer task gathered 9.22 GiB onto a 6 GiB worker` (LocalCluster, one machine; documented in dask 2026.7.1). memmap: `known mechanism of harm: nanny termination on RSS that includes file mappings` (documented, F5; causation inferred). shuffle: `no known mechanism of harm, unmeasured (searched: distributed worker-memory docs at 2026.7.1, the p2p blog, dask merge docs)`. bucketed: `unknown`. Raster merge: `no known mechanism of harm, unmeasured (searched: GeoSparkViz, HadoopViz, sort-last compositing)`. Multi-GPU: `unknown`.

### Ranking

At gate time the 1B-edge copy rows and the breakdown had not landed, so story 8 ("the target is already within reach") was `predicted` and first place at the target was withheld at both thresholds. They landed after the gate (see `### Landed after the gate`): story 8 is `measured here` for the edge render, and the batch ranking below is restated on measured cells. Within a tier, order is fit, then dependency surface.

**Batch (under ten minutes).**
1. Exact, `measured here` (post-gate): A first, with B identical at 100M nodes; 180.5 s at 16 and 334.3 s at 50 on edge render alone, 8.9 to 9.4 GiB; storage `unmeasured`. Then G (fails).
2. Exactness `analytic` (count), time `unmeasured`: D-exact, then C.
3. Exactness unmeasured: E, then F.
4. Power-user: D-sampled (`predicted` pass), then D-quantized.

Stretch row: B does the 100M-edge lookup and render at 1B nodes in about 11 minutes after a 14 to 15 minute table build (n=5), 28 minutes end to end; at 1B edges it fails batch on the prediction (bucket passes alone about 98 minutes with buckets tied to node partitions), feasible, scatter disk about 32 to 66 bytes per edge at peak. C is the unmeasured alternative; A cannot hold the table.

**Interactive (about one minute).**
1. Exact: nothing passes; A fails on measurement (180.5 s at 16, post-gate), then G.
2. and 3. As at batch: D-exact, then C; E, then F, neither with a measured or vouched time.
4. Power-user: D-sampled, the only `predicted` sub-minute path (about 14 s with the build, storage `unmeasured`), then D-quantized.

**Movers:** A passes batch and fails interactive; D-sampled becomes the only predicted sub-minute path. A leads the exact tier at batch on measured cells, while C, D-exact, E and F are `analytic` or `unmeasured`; across classes the order still ranks evidence availability, not architectures, and nothing shows that any of those four would beat A once measured.

### Maintainer-framing verdicts

- **9a, there is a DuckDB story: inconclusive.** DuckDB is the one vouched embedded engine whose join spills (F10, F14), but nothing times a join past memory, two joins carry the OOM risk, nothing shows a gain over A at 100M nodes, and B measures the same partitioned join without an engine.
- **9b-i, the Python-stack floor at the target sits above "fast".** Interactive: supported, now measured (post-gate): 180.5 s at 16 on 8 threads misses a minute before storage, and the sampler's GIL hold (F1) limits 24-core gains. Batch: not supported for the edge render, measured at both 16 and 50 (334.3 s); storage read still `unmeasured`, so the whole-render answer at batch stays open by that one cell. (At gate time: inconclusive.)
- **9b-ii, a compiled core lowers that floor by a named mechanism: not supported, meaning no measured mechanism.** No measured term is interpreter overhead: the rasterizer is compiled and `nogil` (F2), and the numba sampler's GIL hold is a `nogil=True` fix (F1). The one unmeasured candidate only a compiled core adds is software-prefetched, batched lookups (a lens estimate of perhaps 1.5 to 2x on the lookup term). The fused kernel named is numba.
- **9c, a storage structure or layout supports this: supported narrowly for the 1B-node stretch; not needed at the target.** B's sequential partitioned join (F7) cut the 1B-node lookup work from a 7265.7 s render to about 11 minutes of bucket passes plus render (`measured here`, 100M edges, n=5), while lookup stores price 2B lookups in hours (F9). Feasible, not fast: with the 14 to 15 minute table build it is 28 minutes end to end, failing both thresholds. A layout and a join algorithm, not a database.
- **9d, feasible-fast needs much larger structural changes (upstream of the raster, renderer fixed).** Batch: not supported for the edge render, measured post-gate (A passes at both num_steps values), with the storage read the one unmeasured cell; at gate time this rested on the predicted A cell and read inconclusive. Interactive: inconclusive; no exact architecture meets a minute on any figure in hand, and what might (E, F) is unmeasured.

## What was validated, and how

Seven disjoint lenses, blind to each other (prior art and counter-evidence as standing seats; query engines, node-table storage, binning exactness, compute floor, GPU and multi-machine), nominated claims. The compute-floor lens alone read datashader 0.19.1's line source, carrying the others' source questions. Each claim went to an independent voucher, default refuted: 16 voucher calls on 15 claims, two of them second votes. A cold adversary ran the convergence gate against the orchestrator-seeded rubric, which the maintainer has not ratified (grill knowingly skipped).

**Provenance.** The counter-evidence and query-engine lenses each read a stretch of the plan showing the other lens briefs; both report not using them. Hub reads without a voucher: the copy probe calling the serial sampler, the queued rows' 1M-edge partitions, F17's code assumptions.

- **Validated, as corrected:** F1, F2, F4, F5, F7, F9, F10, F12, F14, F16, F17, F18, F19.
- **Killed wordings:** pixel-bound cost without the sub-pixel exception (F3); the nanny killing the LocalCluster memmap workers (F6); sequential Grace I/O (F8); DuckDB joining a build side larger than RAM, or any out-of-core join time (F11); DuckDB's guidance as a thread cap (F13); "Polars group-bys do not spill" (F15); Graph Cities "rather than drawing edges individually" (F20).
- **Single vote, second-vote flag unfunded:** F18, F19, F12, F14, F9, F16, F17 and F4's source half. About eleven claims self-flagged; the two funded second votes went to F10 and F5.

**Gate edits applied.** All six must-fix: E and F moved from the exact tiers to an "exactness unmeasured" tier, with E's renderer-ruling clause; 9b-i at batch and 9d set to inconclusive; the points-versus-segments resolution; `analytic` defined; the half-finding yield; the call count reconciled at 31 through the plan's Amendment 2. All eight should-fix: 9c scoped, the stretch-row sentence restated, A's N1 attribution corrected, D-exact's per-reduction conditions, F citing the plan record and numba-cuda #384, 9b-ii's unmeasured candidate, hub reads labelled, blindness disclosures. The narrowings (F17, the GIL summary, F19, the OSM figure, E) and the four notes (Nöllenburg and Wallinger in N9, the single-vote list, experiment 1's partition sizes, the cuDF record) also landed.

### Not-fully-confirming evidence

No hit killed a story.

- **N1** (A, story 8, 9b-i): copy throughput at 100M edges falls 5.9, 4.8, 3.0M edges/s at 2M, 100M, 300M nodes, so the node table binds time too; 8-to-24-thread scaling has no support.
- **N2** (B, G memmap cells): the nanny counts file pages (F5), so the memmap death may be the instrument. Post-gate: the experiment session's `memmap_chunked` route (one mapped slice at a time) finished every LocalCluster configuration with no restarts and workers under 1.2 GiB, with no other change, which confirms the cause by fix; no RssFile-at-death sample was recorded.
- **N3, N4** (C, 9a): DuckDB's per-thread guidance and OOM caveat (F12, F10); Polars joins do not spill (F14).
- **N5** (story 5's "sort-merge wins"): Kuiper et al. say performance "degrades sharply" past RAM under external sort-merge, before 1.2.0. Abstract only.
- **N6** (memmap as a floor): in Elasticsearch #27748 (5.5.0, spinning disks), default readahead took 7,659 MiB of page cache against 1,607 under madvise, so the 2-hour row may be partly readahead.
- **N7** (C, D-quantized speed): ClickBench has no join; db-benchmark's "Not Tested" often means OOM.
- **N8** (E, 9b-ii, weak): Nixtla #1099's 3.5 to 18x numba-to-C++ gain came from JIT cold start and per-call overhead, absent here.
- **N9** (D design): Xue et al. (arXiv 2512.16017) find density plots "disrupt path continuity"; Nöllenburg and Wallinger (arXiv 2309.02273) recommend their non-datashaded method under 500 vertices, a weak transfer.
- **N10** (A, B if `line_width > 0`; F): antialiased lines are 1.3x to 14x slower (datashader 0.14.0) and CPU-only (#1304).
- **Instruments:** VmHWM includes RssFile and is documented as inaccurate; PSS is imprecise; cgroup v2 counts page cache.
- **Search-scope absences:** no user request for 1B-edge network rasters (datashader and HoloViz discourse, hiveplotlib docs); no billion-edge render drawn edge by edge (those forums, the hairball literature, Giga Graph Cities).

### Landed after the gate

Read from the raw jsonl by the dispatching session on 2026-11-02; no voucher or gate has seen these rows. Same probe scope as above unless stated.

| Row | Build (s) | Edge render (s), median (range) | Memory (GiB) | Raster | Source |
|---|---|---|---|---|---|
| copy, 100M nodes / 1B edges, 1,000 x 1M, num_steps 16, n=5 | 8.4 to 9.1 | 180.5 (178.1 to 181.8) | VmHWM 8.75 to 9.05; anon 7.3 to 7.9; live 6.3 to 7.0 | a4b979dbc67a, all five | copy_billion_edges.jsonl |
| copy, same, num_steps 50, n=5 | 8.4 to 8.6 | 334.3 (334.1 to 337.5) | VmHWM 8.87 to 9.38; anon 7.7 to 8.1 | fbbb91d288cb, all five | copy_billion_edges.jsonl |
| bucketed, 1B nodes / 100M edges, n=5 | 1492 to 1644, of which bucket passes 540 to 663 (table build about 850 to 1000, the same as memmap's 851.5) | 77.1 (58.1 to 134.6) | VmHWM 8.15 | 61c822ac0338, all five | bucketed_huge.jsonl |
| bucketed, 300M / 100M, no cap; 4 GiB cgroup cap, n=5 each | 192 to 239; 237 to 298 | 20.6; 21.6 | VmHWM 3.0 to 3.7 | 1e194c134075 | bucketed_caps.jsonl |
| LocalCluster 4 x 6 GiB, 100M / 100M, n=5: memmap_chunked; bucketed | 11 to 64; 55 to 165 | 37.6; 28.3 | every process under 1.2 GiB RSS, no restarts | d9fef4676c34 | cluster_new.jsonl |
| LocalCluster 4 x 6 GiB, 300M / 100M, n=5: memmap_chunked; bucketed | 112 to 176; 198 to 221 | 75.4; 24.4 | same | 1e194c134075 | cluster_new.jsonl |
| LocalCluster 8 x 3 GiB, 100M / 100M, n=5: memmap_chunked; bucketed | 16 to 37; 40 to 103 | 33.2; 22.9 | same | d9fef4676c34 | cluster_new.jsonl |

`memmap_chunked` reads the memmap table in sorted chunks; it and `bucketed` are the two routes that finished where copy and memmap had died on the cluster, and both beat the p2p shuffle's 140.5 s (100M nodes) and 549.4 s (300M), so the G cells for them read `no known mechanism of harm, measured on one LocalCluster configuration`.

Per-mechanism breakdown, one 1M-edge partition against the 100M-node table, single thread, median of five after a warm-up (`results_2026_11_02_fork/breakdown.jsonl`): at 16, two `searchsorted` passes 0.208 s (one pass on pre-sorted ids 0.122), eight gathers 0.075, Bézier sampling plus frame 0.115, whole build 0.424, `cvs.line` 0.750, spread 0.0015; at 50, build 0.575 (sampling plus frame 0.279) and `cvs.line` 1.507. Datashader's line aggregation is the largest term, 64% at 16 and 72% at 50; lookup and gathers 24% at 16; sampling and frame 10%. Over 100M edges that is about 117 single-thread core-seconds against the 168 the 8-thread row spent, so threading loses about 30% to contention and the GIL-held sampler (F1). The 50-step render is 1.85x the 16-step render, not 3x, consistent with F2's pixel-bound walk. Sorting ids first halves the search. Lens 6's unverified split (F21) has the right order; the measured balance tilts further toward the raster, which moves E's fused-kernel case (it removes sampling and the frame, about 10%) and leaves the rasterizer itself as the term any faster route must address.

## What was inconclusive, and why

- **A at the target:** measured after the gate for the edge render at 16 and 50 (above); the storage read and 24-thread scaling remain unmeasured, and no lazy route has a row at 50. Reading will not settle it.
- **Storage:** no row reads edges from disk.
- **C's time, 9a:** DuckDB docs and blog, db-benchmark and the VLDB 2025 abstract never time a join whose build side exceeds memory.
- **F:** RAPIDS and NVIDIA blogs, cuDF docs, db-benchmark and Graphistry give no named-GPU cuDF join at about 1B rows; the one GPU line figure is from 2019.
- **G:** one LocalCluster configuration; no nanny log for the memmap death.
- **D's gain and mismatch:** no real-graph distinct-curve counts; no published bound for quantized line endpoints against a pixel-exact raster (VIS, EuroVis).
- **E, 24-core scaling:** no compiled CPU line-rasterizer throughput, no fused sample-and-rasterize precedent (one search each).

## What's open

### Proposed experiments

None run here; all belong in `hiveplotlib-performance`.

1. Copy, 1B edges / 100M nodes, `num_steps=16` and 50: done after the gate, five samples each, see `### Landed after the gate`.
2. (merged into 1.)
3. By-mechanism breakdown at 16 and 50: done after the gate, single thread; an 8-thread breakdown that attributes the 30% threading loss is still open.
4. 1B edges read from Parquet: every Storage cell.
5. 8 to 24 threads with `nogil=True` samplers: A at interactive.
6. `bucketed` at 1B / 1B, and with a persisted table: B.
7. DuckDB two-join at 100M and 1B nodes, memory limit, threads and spill recorded: C, 9a.
8. Distinct-curve counts on the 62M-edge OGB graph: D-exact.
9. ADR 0002 mismatch fraction at B = 1500: D-quantized.
10. cuDF float reductions and line render on a CUDA 13-class toolchain: F, the cuDF correction.
11. Memmap on LocalCluster with nanny logs and RssFile at death: G.
12. Skewed degree on A and B.

### Questions

- Can the bucketed table persist across renders, and why is it 61.5 GiB against a 33 GB logical table?
- Will branch 53 keep the lookup behind one internal function, leaving B and C open?
- Does anyone want 1B-edge static network rasters?
- Does the maintainer's CUDA 12.9 diagnosis close the cuDF correction?
- Questionnaire items 1 to 8 are unanswered; item 2 decides this ranking, item 3 whether D enters.

### Unverified context

- **F21** (lens 6): the 21 s copy row as 168 core-s (lookups 40 to 70, raster 50 to 110, sampling 5 to 10); 1B-edge floors of 200 to 230 s on 8 threads, 75 to 110 s on 24 cores if the sampler releases the GIL.
- **F22** (lens 3): a 1.5 to 2 minute DuckDB join at the target; DuckDB 2.0 needs Python 3.11 against hiveplotlib's 3.10 floor; DataFusion and Acero joins do not spill.
- **F23** (lens 4): Parquet, Zarr and Arrow IPC are scan formats, not lookup formats.
- **F24** (lens 7): cuDF caps a column near 2.1B rows (cudf #13159); Graphistry's billion-edge figures are queries, not drawing.
- datashader aggregated 1B OSM points in 1.98 s on a 16 GB laptop (2016 page, version unstated; points). Edge-by-edge tools (Gephi, Cosmograph, KDEEB) stop near 1e6 to 1e7 edges. DuckDB #11752 (join OOM, pre-sorting helped); Crotty et al. (CIDR 2022) on mmap.

## Sources

Versions and locations only.

- **Measurements:** `~/repos/hiveplotlib-performance/synthetic_scaling/dask_partition_memory/`: `RESULTS.md`, `results_2026_11_01/{memmap_fit,memmap_caps,memmap_huge,cluster}.jsonl`, `results_2026_11_02/bucketed_huge.jsonl` (`lazy_node_lookup_probe.py --route bucketed`); dask 2026.7.1, datashader 0.19.1, numba 0.67.0.
- **This repo, branch 53:** `utils.py:172,209`; `viz/datashader.py:797-804`; `hiveplot.py:1742,1806-1808`.
- **datashader 0.19.1:** `glyphs/line.py`, `reductions.py`, `utils.py:52`, `doc/reduction.csv`; Pipeline docs; 0.15.2 and 0.14.0 notes; #1304.
- **distributed 2026.7.1, Linux, numba:** `worker_memory.py`, worker-memory docs, #4304; man-pages 6.19 `proc_pid_status(5)`, `proc_pid_statm(5)`; numba JIT docs.
- **Query engines:** DuckDB 2.0 docs and "Benchmarks over time" (2024-06-26); Kuiper et al., PVLDB 18(8), 2025; Polars 2.0 post (2026-10-06) and py-2.1.0 notes; ClickBench; db-benchmark (2026-08-17).
- **Joins, storage, geometry, precedent:** UNSW COMP9315 24T1 ex07; Berkeley summary of Shapiro 1986; RocksDB benchmarks wiki (2022-05-05); Elasticsearch #27748; arXiv 2201.07834, 2512.16017, 2309.02273; Novotny and Hauser 2006; Graph Cities 2022; Nixtla statsforecast #1099; numba discourse #844.

## See Also

- [[hiveplotlib]]: the library and its scaling milestone
- [[dask-route-memory-bounding]]: the prior run on the Dask route's combine and memory
- [[0002-performance-regression-harness|ADR 0002]]: the equivalence wall any raster-changing option answers to
- [[specs/lower-default-curve-resolution|Lower the default curve resolution to 50]]: the signed default the target is judged at

---
*Produced by a bounded literature-mode research run, 2026-11-02. 30 of a binding 31 Agent calls, counting this landing (4 before the lens wave, 7 lenses, 16 vouchers, 1 synth, 1 convergence gate, 1 landing); peak concurrency 8 of a ceiling of 8; lenses, vouchers and synth on Opus by the maintainer's instruction, the adversary gates on Fable; about 3.8M tokens including this landing (task-notification subagent figures, which may understate spend, because an earlier telemetry study found them closer to peak context than total spend).*
