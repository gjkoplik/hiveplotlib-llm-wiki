# Research plan: what it takes to render a billion-edge hive plot

<!--
Research-plan shape (deliberately light per the plan template). No workstream or done-when
ceremony; conventions (shallow-panel dispatch, standing lenses, bounds, grounding, durable
landing) live in the research-track skill. This plan carries only the per-plan fill-in.
Consumer: hiveplotlib. Evidence mode: literature. Requested 2026-11-02. No spec (research run).
-->

## Question

What architecture lets hiveplotlib's datashader route render a static raster on a fixed canvas for a graph of about 1B edges and 100M nodes (with 1B nodes as a stretch row), how fast can each candidate go, and does the answer live inside Python?

The answer is a ranked set of at most seven architectures. Each states what it takes (code, dependencies, API consequences against the standing rules), what it reaches (edges, nodes, hardware, time), what reading its inputs from storage costs, its exactness status against `wiki/wiki/adr/0002-performance-regression-harness.md`, and which of its figures are measured, predicted or unmeasured. The run also gives separate verdicts on the maintainer's four framings (story 9: five verdicts, since 9b splits in two).

**Scope of "architecture".** It ranges over everything upstream of the raster: storage, the join, the compute language, collapsing or sampling before drawing, the hardware and the number of machines. The 2026-09-30 ruling holds only the renderer fixed (datashader stays the at-scale answer). Verdict 9d is judged in that scope, so a "not supported" on 9d means no change upstream of the raster that large is needed, not that 9d was out of bounds.

The maintainer's words, 2026-11-02: "I don't know if there's this DuckDB story or if there's other more fundamental revisions to how we could do stuff to scale this way, or if it just has to be slow if we do things in Python and that suggests that we need to switch languages, or if there's a specific database structure that could support this, or if it would require much larger structural changes to ever be feasible fast."

**Bounds.** Literature only: lenses read and cite, run no benchmarks, write no code, and post nothing upstream. The measured inputs below are the data. The queued rows in `## Inputs` run outside the panel under the maintainer's standing "run the throwaway experiments" ruling; no lens runs them or waits for them. Nothing in the library changes. The maintainer's statements are not gospel, and the run may conclude the target or the framing is wrong.

## Inputs (attack these; they are not conclusions)

The fork's throwaway experiments, 2026-10-31 to 2026-11-02, in `~/repos/hiveplotlib-performance/synthetic_scaling/dask_partition_memory/`. Machine: ringtail, 24 cores, 30 GiB visible to WSL. 8 threads unless stated, `num_steps=16`, 3 axes, uniform random integer ids, `ds.count()`, medians of five fresh processes unless stated. Edges come from `dd.from_map` in memory; no row reads edges from disk.

**Routes.** `library_list` is today's code with the one-line tree combine: nodes in pandas, ids cast to object dtype at `hiveplot.py:1742` (`edge_ids.copy().astype("O")`, confirmed on the working branch), two pandas merges per task. `copy` is a compact id-sorted node table (33 bytes per node) held once in the process and looked up with `np.searchsorted`, one pass per edge partition. `shuffle` is two Dask disk-shuffle merges, one per endpoint. `memmap` is the copy table in files on disk, built by bucketing by id range and sorting each bucket without holding the whole table, read through the page cache.

**Equivalence.** Every route at every size produced the same raster hash. Against the library at 20k nodes / 400k edges, count and any are bit-exact and sum, mean and var agree within 4.4e-16.

| Nodes | Edges | Route | Build (s) | Edge render (s) | Peak RSS (GiB) | Live peak (GiB) | Disk (GiB) |
|---|---|---|---|---|---|---|---|
| 2M | 10M | library_list | 4.4 | 27.9 | 1.83 | | 0 |
| 2M | 10M | copy | 2.9 | 3.6 | 2.47 | | 0 |
| 2M | 10M | shuffle | 2.8 | 6.3 | 3.40 | | 0 |
| 2M | 100M | library_list | 4.7 | 153.8 | 1.97 | | 0 |
| 2M | 100M | copy | 2.8 | 17.0 | 2.68 | | 0 |
| 2M | 100M | shuffle | 2.7 | 66.8 | 5.55 | | 5.3 |
| 2M | 300M | copy | 3.8 | 50.7 | 2.81 | 2.26 | 0 |
| 2M | 300M | shuffle | 2.8 | 399.9 | 5.53 | 3.02 | 19.9 |
| 20M | 10M | library_list | 23.3 | 431.0 | 9.43 | | 0 |
| 20M | 10M | copy | 3.7 | 3.8 | 2.81 | | 0 |
| 100M | 100M | library_list | stopped at the 26 GiB cap after 215 s | | | 18.55 | |
| 100M | 100M | copy | 11.5 | 21.0 | 8.96 | 7.03 | 0 |
| 100M | 100M | shuffle | 4.2 | 121.9 | 11.17 | 9.20 | 11.4 |
| 100M | 100M | memmap | 11.0 | 31.3 | 27.4 (page cache) | 1.42 | 4.2 |
| 300M | 100M | copy | 119.7 | 33.1 | 25.70 | 23.05 | 0 |
| 300M | 100M | shuffle (unshared reads) | 7.4 | 764.8 | 15.10 | 13.74 | 28.0 |
| 300M | 100M | memmap | 107.0 | 68.9 | 77 (page cache) | 2.37 | 18.3 |
| 300M | 100M | memmap, 16 GiB cgroup cap | 126.1 | 49.3 | | 2.14 | 18.4 |
| 300M | 100M | memmap, 8 GiB cap | 113.8 | 579.3 | | 3.3 | 18.0 |
| 300M | 100M | memmap, 4 GiB cap | 125.8 | 2219.5 | | 2.96 | 18.4 |
| 1B | 100M | copy | aborted, RSS above 26 GiB at 45 s | | | | |
| 1B | 100M | shuffle | aborted, RSS above 26 GiB at 1814 s | | | | |
| 1B | 100M | memmap (n=4) | 851.5 | 7265.7 | 209 (page cache) | 7.01 | 61.5 |

Under the 16 and 8 GiB caps at 300M nodes, copy and shuffle were killed (exit 137); memmap finished under all three caps, slower as the cap fell below the table size.

**LocalCluster** (`distributed`, p2p shuffle) at 100M nodes / 100M edges, 4 workers x 6 GiB, 2 threads each: copy died (a repartition-to-fewer task gathered 9.22 GiB onto one worker against its 6 GiB limit) and memmap died (workers killed on the edge partition task, cause not isolated). Shuffle finished in 140.5 s (549.4 s at 300M nodes), total PSS peak 4.1 GiB, 13.9 GiB spilled.

**Measured earlier.** At 20M nodes the library's pandas merge takes 6.26 of 7.95 s per 1M-edge call. Tokenizing the builder closure pickles the whole hive plot per render (569 MB at 200k nodes). Today's library build costs 306 bytes per node; the copy table costs 33.

**Read, not run (orchestrator, 2026-11-02):**

- **Provenance is split.** RESULTS.md covers the copy, shuffle and library rows through 300M nodes. The memmap rows, the cgroup-cap rows, the 1B rows and the LocalCluster rows exist only as `results_2026_11_01/{memmap_fit,memmap_caps,memmap_huge,cluster}.jsonl` and are not yet written up. `results_2026_11_01/memmap_huger.jsonl` and `results_2026_11_02/bucketed_huge.jsonl` were empty at 07:14 on 2026-11-02, so a further run may be in flight or may have failed. The run does not wait on them. A lens citing a memmap or cluster figure cites the jsonl file.
- **The 1B-node "209 GiB" is not resident memory.** The row's `file_peak_gib` is 206.8 to 207.3 on a machine with 30 GiB visible, with `anon_peak_gib` near 7.1 to 7.5. It counts file-backed pages touched through the map, not a working set. Live peak (7.01 GiB) is the comparable figure. The same reading applies to the 27.4 and 77 GiB page-cache entries. The fifth 1B sample exited 143, hence n=4.
- **The 2-hour row is storage-bound, by the numbers.** Memmap renders 100M edges in 31.3 s at 100M nodes and in 7265.7 s at 1B nodes: 10x the nodes, about 230x the time. The 1B row's 61.5 GiB of files cannot fit in 30 GiB of page cache, and the lookups are random. The 8 and 4 GiB caps at 300M nodes show the same cliff. A sequential access pattern (sort edges by endpoint, merge against the sorted node file) is the obvious untested alternative.
- **The stated target is unmeasured but sits next to a measured row.** No row has 1B edges. The nearest to 1B edges / 100M nodes is copy at 100M / 100M: 21.0 s, 8.96 GiB peak, 7.03 GiB live. At 2M nodes copy's render is linear in edges (17.0 s at 100M, 50.7 s at 300M) and its peak is flat. Linear extrapolation predicts about 210 s of edge render and a peak near 9 GiB for 1B edges at 100M nodes, at `num_steps=16`, on 8 threads. A prediction, never a measurement, and it excludes reading 1B edges from storage (about 16 GB of int64 ids) and `num_steps=50`. The rows that would measure it are queued (next block).
- **`num_steps`.** Every row is at 16. The signed default is 50, which triples the curve points per edge. The lazy routes have no row at 50. The five-sample grid's only rows at 50 are on today's route at 2M nodes.
- **Installed versions** in the project venv: dask 2026.7.1, datashader 0.19.1, narwhals 2.25.0, numba 0.67.0, polars 1.43.2, pyarrow 25.0.1. DuckDB and `distributed` are not installed there; the cluster probe ran in a scratch environment.

**PENDING ROWS (queued 2026-11-02, not yet run, no figures).** Authorized by the maintainer's standing "run the throwaway experiments" ruling. Ringtail is occupied by the sibling session's own run (a 1B-node `bucketed` row with a two-hour timeout, started 07:14 on 2026-11-02), so these are queued behind its current stage:

- copy route, 1B edges, 100M nodes, 100 partitions of 10M edges, `num_steps=16`
- the same row at `num_steps=50`
- a by-mechanism breakdown of the measured 100M nodes / 100M edges copy row: `searchsorted` gather, Bézier sampling, frame build, `cvs.line` aggregation

The lens wave does not wait for them. **Constraint until these rows land in this section:** story 8 is `predicted` only, and the synth may not rank architecture A first on a predicted cell alone. If they land before synth, the dispatching session appends them here with their provenance (jsonl path, n, instrument behind each memory figure) and lifts the constraint in a dated note directly below this block. If they have not landed when the synth fires, the constraint stands for the whole run and the rows stay under `Proposed experiments`.

**A reading lenses may test and must not assume.** On one machine with the node table resident, the edge side scales linearly at about 5M edges/s on 8 threads, so 1B edges is minutes, not hours. The node table is what binds (33 bytes per node compact, 306 in today's build). Past memory, the measured options are a page-cache memmap (works at 1B nodes, 2 hours, random reads) or a cluster shuffle (works under per-worker limits, slow).

## Prior rulings and records

From record-keeper's pre-task pass, 2026-11-02. Name it explicitly if a recommendation would reverse one.

Binding:

- `wiki/wiki/adr/0002-performance-regression-harness.md`: the numeric-equivalence wall. Alternate paths are compared to the single-shot path on curve arrays (elementwise, coerced to float32, NaN separators matched) and on rasters (near-exact count agreement with a validated `allowed_mismatch_fraction`). A lossy option fails it by design and must state its exactness cost.
- `wiki/wiki/specs/lower-default-curve-resolution.md` (signed 2026-08-31): default `num_steps` is 50; 16 and 25 rejected as defaults on headroom; adaptive sampling measured and rejected. The code still says 100.
- `wiki/wiki/plans/dask-route-memory-research.md`, questionnaire, maintainer rulings of 2026-10-31: this work blocks branch 53 ("settle scaling once, before it is part of the API"); the 20M-node target is rejected and nodes must scale too, node target unset (100M proposed, unconfirmed); lazy nodes get a datashader-only route beside the unchanged in-memory path; scaling means more workers or more disk; the edge-to-node lookup design is undecided. Standing rules restated there: no `use_dask` opt-in (Path A, narwhals detects Dask); narwhals pass-through with no canonical internal frame (do not relitigate); ecosystem tools over bespoke low-level code.
- `wiki/wiki/plans/scaling-large-networks.md`: polars-only and always-polars internals rejected (the dominant cost is the numpy curve array, and both break the Dask and cuDF stories); Dask-only rejected (slows small graphs); DuckDB, PySpark and Ibis as narwhals lazy backends deferred for lack of a demand signal, a deferral and not a ruling (the wiki's other DuckDB mentions are `dask-route-memory-research.md`'s exclusion of DuckDB from that run's prior-art lens, "since they combine group state and not a fixed raster", a scoping choice for a different question and not a ruling); cuDF/GPU supported through narwhals dispatch plus datashader's CUDA path, where sum/mean/var/std raise upstream and count/any/max/min work; resident curves are 808 bytes per edge at `num_steps=100`, "impossible at 1B"; the maintainer declined a second host and any multi-machine side quest for that sprint, reopenable on a genuine theoretical reason.
- `wiki/wiki/log.md` 2026-09-30: Datoviz and GPU-plotting-library integration ruled out; datashader stays the at-scale answer.
- The 2-3 axis rule (a design convention, not an ADR); more groups go through `HivePlotMatrix`.

Findings:

- `wiki/wiki/analyses/dask-route-memory-bounding.md` (validated 2026-10-30, cluster half inconclusive): option A (`Canvas.line(["x"], ["y"])` reaches datashader's fan-in-2 tree) leads for the combine, 1.8 to 1.97 GiB at 1M rows per partition from 10M to 100M edges; option F (per-pair binning, collapse before drawing) parked as power-user only, exactness unmeasured; Rübel SC08 and imMens as precedent for collapsing before drawing; no precedent found for rendering graphs on a cluster or for billions of edges drawn edge by edge (a search-scoped absence); the builder closure pickles the whole hive plot per task.
- `wiki/wiki/plans/archived/adaptive-curve-sampling.md`: the default of 100 is about 6x oversampled, the maximum required was 16 at 1500px; M4 decimation does not transfer because hive edges are not x-monotone.
- `wiki/wiki/entities/hiveplotlib.md`: stress-tested to a 62M-edge OGB network; streamed and fused datashader routes; 14 cuDF tests GPU-gated.

**Settled, do not re-research:** adaptive sampling (of curve points per edge; sampling edges is not settled and is open under story 3); 16 or 25 as the default `num_steps`; polars-only, always-polars, a canonical internal frame, Dask-only; a `use_dask` opt-in; making narwhals optional; Datoviz or GPU-plotting integration; flat versus tree combine on one machine; lazy-node lookups by copy, shuffle and memmap on one machine up to 1B nodes (measured above).

**Net new, open:** a compiled core (Rust, C++, Cython), a language switch, DuckDB or any database or query engine as the join and aggregate engine, vaex, cuGraph or Graphistry as renderers, out-of-core storage designs, sampling edges before drawing, anything past one machine's memory.

## Candidate stories / hypotheses

1. **The floor is low and Python is not the problem.** The compute shape is one scan of the edges, one join of 2B endpoint ids against the node table, a 17x to 51x expansion into curve points, and a reduction onto a fixed canvas of about 10 MiB. Compiled kernels already do the arithmetic (numba curves, numba rasterization in datashader). The 8x gain from `library_list` to `copy` came from removing object-dtype pandas merges, inside Python. A compiled core or a language switch changes the floor only if it removes a measured cost the Python stack cannot: the curve materialization between sampling and rasterizing (a fused sample-and-draw kernel, which numba could also host), or the random-access join. Its rival: Python's per-task and per-partition overheads (scheduler, GIL, pickling) compound at 1B edges and set a ceiling only a compiled pipeline clears.
2. **The join is a database problem, and the ecosystem already solves it.** DuckDB (out-of-core hash join with spilling, vectorized, multi-threaded, Arrow out), the Polars streaming engine, DataFusion, pyarrow's Acero, ClickHouse. Published 1B-row joins and group-bys on one machine, with what they need in RAM and disk. Whether datashader can consume their output batch by batch without hiveplotlib building infrastructure, and whether a user's DuckDB relation could enter through narwhals as a lazy backend under the pass-through rule.
3. **Hive-plot geometry collapses the work.** Layout is fixed and positions come from one scalar per node, so an edge's curve is a function of (axis pair, rho_from, rho_to). Three variants with different exactness. Exact deduplication groups on exact rho values and draws each distinct curve once with a weight; it is lossless for count, any, max and min, rounding-only for sum and moment-derived mean and var, and gains only when positions repeat (discrete attributes, ties). Quantization bins rho at sub-pixel resolution, so 1B edges become at most pairs x bins^2 weighted curves (about 13.5M at 1500^2 bins and 6 pairs), after which draw cost is bounded by resolution and not edge count; it is lossy, with an error that has to be bounded against the pixel-exact raster. first, last and by do not survive either collapsing variant without carrying their keys. A third variant samples edges rather than collapsing them: a Bernoulli sample at fraction p gives an unbiased count raster scaled by 1/p with a per-pixel binomial bound, and 1B edges at 1% is a 10M-edge render, which the copy route does in 3.6 s at 2M nodes (`## Inputs`). It is lossy and fails the ADR 0002 wall by design, the same class as quantization; which reductions survive is part of the claim (count and sum as unbiased estimates on the adversary's reading; max, min, first, last and by not). Precedent: Wickham's bin-summarize-smooth, imMens, Nanocubes, Falcon, Mosaic on DuckDB, datashader's own aggregate-then-shade. It is a GROUP BY, the cheapest thing a database does, so it composes with story 2.
4. **GPU, inside datashader.** cuDF, cuGraph and dask-cuda plus datashader's CUDA path, which the repo already dispatches to. Join rates and memory for 1B edges on one GPU and on several; what Graphistry and the RAPIDS cuGraph-plus-datashader demos actually render at billions. Bounded by the standing ruling (datashader stays the at-scale answer) and by sum/mean/var/std raising on cuDF today.
5. **Storage, not compute, binds past memory.** The 1B-node memmap row is random reads against files that do not fit in page cache. Candidate node-table stores: memory-mapped sorted arrays (what the probe did; vaex's approach), Arrow IPC or Feather maps, Zarr, Parquet row groups, an embedded key-value store (LMDB, RocksDB, SQLite). The claim to test: past memory, sort-merge (sort edges by endpoint id, stream against the sorted node table) beats any random-lookup store, and the choice between sort-merge, hash and broadcast joins as the node table outgrows one worker is a known result.
6. **A cluster is not required for the target.** Whether 1B edges needs more than one machine at all; prior art (datashader's billion-point demos on one machine from Parquet, GeoSparkViz, HadoopViz, sort-last compositing, where merging rasters is a sum); and the measured fact that on one machine the shuffle is the slowest route everywhere but the only one that survived per-worker limits.
7. **Precedent for feasibility and demand.** Who renders about 1B edges or points today (datashader's OpenStreetMap 2.7B points and NYC taxi 1.1B, vaex on 1B memory-mapped rows, Graphistry, cuGraph, Cosmograph, the large-graph drawing literature), on what hardware and in what time, and whether any draw edges one by one at that scale or collapse first.
8. **The target is already within reach.** The stated target (1B edges, 100M nodes) needs no new architecture: the copy route with a lazy-node table, built without today's 306-bytes-per-node overhead, fits in the 30 GiB visible to WSL on ringtail (predicted peak near 9 GiB; the copy route at 300M nodes peaked at 25.70 GiB against the same 30 GiB) and renders in minutes (prediction from `## Inputs`). The architecture question only bites past about 300M nodes, under per-worker limits, or at `num_steps=50` if the curve term binds there. Whatever bites first is the real design driver. No lens moves this story; only the pending rows in `## Inputs` can, and until they land it is `predicted` only, under the constraint stated there. The counter-evidence lens still hunts what would falsify it.
9. **The maintainer's framing: four framings, five verdicts.** His "it just has to be slow if we do things in Python and that suggests that we need to switch languages" carries two claims, so 9b splits; a "not supported" on the bundle would read as "Python is not slow" when the evidence may say "slow at the target, and a switch does not fix it".
   - 9a. There is a DuckDB story.
   - 9b-i. The Python-stack floor at the questionnaire item 1 target sits above item 2's "fast" (reported at both thresholds).
   - 9b-ii. A compiled core or a language switch lowers that floor by a named mechanism.
   - 9c. A specific database structure or storage layout can support this.
   - 9d. Feasible-fast requires much larger structural changes, judged in the scope stated under `## Question` (anything upstream of the raster; the renderer fixed).

These are not mutually exclusive. A likely answer is a combination, with a different architecture for branch 53 than for the stretch rows.

## Failure-mode rubric

**Provenance: seeded by the orchestrator, not elicited from the maintainer.** The research-branch grill is knowingly skipped (see `## Alignment (grill)`), so this is the model's standard until the maintainer says otherwise (questionnaire item 7). Consumed by the adversary at convergence.

- **Measurement artifact (first).** Peak RSS is `VmHWM` and includes memory the allocator kept after freeing. The page-cache figures (27.4, 77, 209 GiB) are counts of file-backed pages, and 209 GiB on a 30 GiB machine cannot be a working set. A finding that reads a page-cache figure as memory need, or compares peak RSS on one route to live peak on another, is killed.
- **Workload artifact.** Every row is uniform random integer ids generated in memory, 3 axes, `ds.count()`, `num_steps=16`, no disk reads of edges. Real graphs are skewed (hub nodes make the join cheaper on cache and worse on partition balance), arrive as Parquet or a database, and render at the signed default of 50. A finding extended to any of those says so.
- **Extrapolation presented as measurement.** No row has 1B edges (the pending rows in `## Inputs` would be the first). Every 1B-edge figure, including the 210 s in `## Inputs` and the 17 and 3 GiB figures from the prior run, is a prediction and is labelled as one, naming the rows it comes from.
- **"Feasible fast" stretched.** A 2-hour single-machine run is feasible, not fast. Each architecture's time cell names its row or prediction and is judged against both thresholds of questionnaire item 2 (about one minute and under 10 minutes), with its read-from-storage cell reported beside it rather than dropped.
- **Points are not segments.** Datashader draws line segments between consecutive samples. A GROUP BY over curve sample points, or a points glyph over them, is not the same raster as the line glyph. A binning or database recommendation states which glyph it feeds and why the raster matches.
- **Binning exactness asserted, not bounded.** A quantized-binning option states its error against the pixel-exact raster as a bound (in pixels or in the ADR 0002 mismatch fraction) with its derivation or a cited one, separates the exact-dedup variant from the quantized one, and names which reductions survive. An edge-sampling option states its per-pixel bound at fraction p (binomial, or a cited one) and which reductions survive as unbiased estimates. "Visually identical" is killed.
- **Language switch with no named cost.** A compiled-core or language-switch recommendation names the measured cost it removes (a row and a column of `## Inputs`) and the mechanism. One that cites general "Python is slow" sources is killed.
- **Database recommendation that breaks the standing rules.** An engine that becomes a canonical internal frame, forces user frames through a conversion the round-trip contract forbids, or adds an opt-in in the style of `use_dask` names the reversal. A bespoke join, store or rasterizer names why the ecosystem tool fails.
- **Vendor benchmarks at face value.** DuckDB Labs maintains the db-benchmark suite; Polars, ClickHouse, NVIDIA and Graphistry publish their own numbers. Record the trust judgment; a recommendation cannot rest on a vendor figure alone, and every benchmark figure carries its hardware, RAM, disk, version and date.
- **Stale numbers.** Query engines and RAPIDS move fast. A benchmark older than about two years, or with no version, cannot carry a ranking cell for a 2026 recommendation without saying so.
- **Scope laundering across hardware.** Ringtail (24 cores, 30 GiB), a cloud VM with 1 TiB, one GPU, several GPUs and a cluster are different claims. Every number carries its hardware. Cluster cells use only the three forms in `## Validation criteria`; nothing in this run is cluster-measured beyond the one LocalCluster probe on one machine, and that probe is tagged `LocalCluster, one machine`.
- **End-to-end laundered as edge render.** The table's times are edge render only; the node raster, the node-table build, reading edges from storage and pickling the builder are separate. An architecture's time cell says which parts it covers.
- **Evidence-mark laundering.** A ranking that orders a `measured here` or `predicted` cell against a `literature-supported` or `unmeasured` cell says the comparison is across marks. Architecture A's cells come free from `## Inputs` while the others share at most 14 vouchers, so a ranking that puts A first only because its cells carry numbers is ranking evidence availability, not architectures.
- **n too small, uncontrolled comparison.** The 1B memmap row is n=4, the cluster probe is one configuration, and the cgroup rows differ from the uncapped rows in more than the cap. Such comparisons are pointers.
- **Already settled.** Anything on the settled list in `## Prior rulings and records` is re-researched only with new evidence, named as such. The same holds for the prior run's searches: a claim re-found from the "Already found, unverified" block under `## Lenses + bounds` without new material is context, not a nomination.
- **Absence claimed as fact.** "Nobody renders 1B edges edge by edge" is admissible only as a search-scope statement naming what was searched.
- **Premise confirmation.** The panel treats the target, the "minutes not hours" reading or the maintainer's framing as given. Story 9's five verdicts are separate, each with evidence; one blended verdict (9b-i and 9b-ii merged included) is killed.
- **Grounding failure.** A claim without a faithful quote, a source without a version or date, or claim-maker equal to voucher.

## Lenses + bounds

Seven lenses, each disjoint and blind to the others: two standing seats, then five orthogonality slices. The boundaries below keep them disjoint; a lens that strays outside its slice hands the pointer back rather than reading it. Each brief names the ranking cells its answer moves. One file is read by one lens only: datashader 0.19.1's line glyph and its dask line path belong to the compute-floor lens, whose brief carries the other lenses' questions about that source; the rest cite published material about it.

**Common to every lens.** Literature only: docs, source, papers, issues, benchmark write-ups, talks. Run no benchmarks, write no code, post nothing anywhere. Every claim carries its supporting quote and a source with a version, tag or date. Every number carries its scope: hardware, RAM, disk, time, software version, and whether it counts points, segments or rows. Record a trust judgment per source. Nominate at most your allotment of claims for verification, each naming the ranking cell it moves; everything else is context and reaches the summary labelled `unverified`. Self-flag any nominated claim with a single quote behind it (the research-track second-vote flag). Treat `## Inputs` as things to attack, and when your material contradicts the reading at the end of `## Inputs`, say so first. Return only distilled conclusions with citations, plus what you looked for and did not find. The dispatching session hands each lens `## Question`, `## Inputs`, `## Prior rulings and records`, `## Candidate stories`, `## Failure-mode rubric`, this paragraph and its own brief (plus the "Already found, unverified" block below, for the three lenses that name it), never another lens's return.

- **[STANDING] Prior-art / counterfactual: who draws this much, and how.** Reading slice: datashader's own large-data demos and docs (OpenStreetMap, NYC taxi, the line and trajectory examples, its performance guidance), vaex's billion-row visualizations, graph-tool, Gephi and Cosmograph limits, and the large-graph drawing literature (surveys of large-graph visualization, edge bundling, graph aggregation and sampling for display, density-based edge rendering). Search strategy: start from the tools' own docs and papers and follow their citations; search by scale words ("billion edges", "billion points") plus "graph drawing" and "network visualization". Answer: what has been rendered at about 1B items, on what hardware and in what time; whether anyone draws edges one by one at that scale or collapses first; what a 1500 x 1500 canvas can show of 1B edges at all (each pixel crossed by hundreds of edges); and the counterfactual, whether a hive plot at this scale should be a binned per-pair density or a sampled raster by design (the design-judgment cell for each of D's three variants; the error bound is the binning lens's). Excludes RAPIDS, cuGraph and Graphistry (GPU lens), the visualization aggregation literature and sampling bounds (binning lens), query engines and node stores, and datashader's source (compute-floor lens). Brief carries the "Already found, unverified" block. Moves: story 7, story 3's design-judgment cell for all three variants, verdict 9d. Claims for verification: 2.

- **[STANDING] Counter-evidence: blind to the confirming lenses.** Search for what would falsify stories 1, 2, 3, 5 and 8, and the reading at the end of `## Inputs`. Hunt: out-of-core joins in DuckDB and Polars that ran out of memory, spilled badly or stalled on high-cardinality keys; datashader line-glyph throughput cliffs at large counts (antialiasing, per-segment cost, single-threaded steps, reductions that serialize), from issue trackers and published reports only, since the source read is the compute-floor lens's; numba code paths that hold the GIL or fail to scale past 8 threads; page-cache thrash and memory-mapped random access failing; cases where moving a comparable pipeline from Python to Rust or C++ gave large wins for reasons a numba kernel cannot match; arguments that binning destroys structure a hive plot is meant to show; critiques of db-benchmark, ClickBench and vendor GPU benchmarks; and whether any user has asked for 1B-edge network rasters at all. Also point at the instruments: what `VmHWM` and file-backed counters do and do not measure. Search by failure vocabulary ("OOM", "spill", "slower than", "regression", "killed", "page cache", "thrash") in issue trackers, forums and post-mortems, not by feature names. Carry every non-confirming hit labelled not-fully-confirming. For "nobody renders 1B edges edge by edge", report a search-scope statement and spend no nomination on it. Brief carries the "Already found, unverified" block. Moves: any architecture's cells a hit contradicts, the measurement-artifact rubric entry, verdicts 9a, 9b-i and 9b-ii. Claims for verification: 2.

- **Query engines for the join (story 2, architecture C).** Reading slice: DuckDB's docs and papers on out-of-core hash joins, spilling and memory limits, its Arrow and numpy interfaces and batch readers (`fetch_record_batch` and kin), and its practical join sizes on one machine; the Polars streaming engine's join and sink support; DataFusion; pyarrow Acero; published one-machine join and group-by benchmarks at about 1B rows (db-benchmark, TPC-H at SF100 and SF1000, ClickBench), each figure read with its hardware, RAM, disk, version and date; narwhals' lazy-backend support for DuckDB and Ibis and what a lazy frame can and cannot hand to `map_partitions`-style code; and the batch hand-off to datashader (whether an engine's record batches can feed `cvs.line` batch by batch, and what accumulating rasters across batches costs). Search strategy: engine docs, engine papers (CIDR, VLDB, SIGMOD), then third-party benchmark write-ups. Answer: can an embedded engine do the 2B-id join against a 100M-row and a 1B-row node table on ringtail-class hardware, in what time and memory; can its output reach datashader in batches with no hiveplotlib-built infrastructure; what a DuckDB route does to the pass-through rule and the dependency surface; and how fast an engine runs the GROUP BY of D's quantized variant (speed only). Excludes node-table storage formats and the choice of join algorithm past memory (node-table storage lens), GPU and multi-machine engines, GROUP BY exactness (binning lens), and ClickHouse as a server, which enters only through ClickBench figures. Moves: architecture C, verdict 9a. Claims for verification: 2.

- **Node-table storage (story 5, architecture B).** Reading slice: Arrow IPC and Feather memory maps; Parquet memory maps, row groups and page indexes; Zarr; LMDB, RocksDB and SQLite point-lookup and range-scan rates, each with its hardware; the database literature on sort-merge versus hash versus broadcast joins as one side outgrows memory, and random lookup versus sequential scan; page-cache behavior for random reads against files larger than RAM, and SSD random-read references. Search strategy: store docs and published benchmarks, then the join-algorithm literature (textbook treatments and VLDB/SIGMOD papers), then systems references for SSD and page-cache floors. Answer: what 2B random lookups cost on each store against a 100M-row (about 3.3 GB at 33 bytes per node) and a 1B-row (about 33 GB) node table on ringtail-class hardware (30 GiB visible to WSL, local SSD); whether a sorted, sequential layout (sort edges by endpoint, stream against the sorted node file) fixes the 1B-node memmap row, and what sorting 2B ids out of core costs; and which join the literature picks at each size. Excludes query engines' own join implementations (query-engine lens), lookups into a resident in-memory table (compute-floor lens), vaex as a renderer (prior-art lens), GPU and multi-machine. Moves: architecture B, verdict 9c. Claims for verification: 2.

- **Binning exactness and aggregate-then-render precedent (story 3).** Reading slice: Wickham's bin-summarize-smooth (2013), imMens (Liu, Jiang and Heer 2013), Nanocubes (Lins et al. 2013), Falcon (Moritz et al. 2019), Mosaic (Heer and Moritz 2023), VegaFusion, datashader's aggregate-then-shade design as documented, the Rübel SC08 and imMens precedent the prior run found, the error analysis of quantizing curve control points against a rasterized output (Lipschitz bounds on Bézier curves in their control points; rasterization's discontinuity at pixel edges), and sampling for display: a Bernoulli edge sample at fraction p, the count raster scaled by 1/p, its per-pixel binomial bound, and the published treatments of sampling large graphs or point sets for rendering. Search strategy: visualization venues (VIS, EuroVis, CHI), datashader's docs, and the sampling-for-display literature. Answer: an error bound for quantized rho binning against the pixel-exact raster, stated in pixels and translated to ADR 0002's terms; a per-pixel bound for edge sampling at fraction p, in the same terms, with the pixel count at which its relative error falls below one count step (1B edges at 1% is a 10M-edge render, 3.6 s on the copy route at 2M nodes per `## Inputs`); which reductions survive exact dedup, quantization and sampling (for sampling, which survive as unbiased estimates); analytically, whether weighted curves through `ds.sum` should reproduce `ds.count` (the source answer comes from the compute-floor lens and the synth joins the two); how many distinct curves real hive-plot layouts produce (ties in sorting variables); and whether published aggregate-then-render systems bin points only or also bin line geometry. Excludes engine benchmarks (query-engine lens), the design judgment of whether to collapse or sample (prior-art lens), and datashader's source (compute-floor lens). Moves: architecture D's exactness and reach cells for all three variants, the points-versus-segments rubric entry. Claims for verification: 2.

- **Compute floor and language (story 1).** Reading slice: numba's docs and issues on `parallel=True`, `prange`, `nogil` and threading layers; datashader 0.19.1's line glyph and its dask line path, read from source (this lens is the only one that reads that source: where it parallelizes, what it allocates per segment); numpy `searchsorted` and sort performance and their thread behavior; published per-segment line-rasterization throughput for CPU rasterizers in compiled languages; DRAM latency and bandwidth references for random lookups into resident tables of about 3 GiB and 33 GiB (past RAM is the node-table storage lens's); and documented cases of fused sample-and-rasterize kernels (curves drawn without materializing their points). Local source is in `~/repos/hiveplotlib/.venv`. Search strategy: source first, then the projects' issue trackers, then systems references for hardware floors. Answer, using `## Inputs` as the test: what fraction of the measured 21 s at 100M / 100M is lookup, curve sampling, curve materialization and rasterization (by mechanism, as a labelled estimate; the pending breakdown in `## Inputs` checks it if it lands before synth, and the lens does not wait for it); what a first-principles floor for 1B edges on 8 and 24 cores looks like; which of those costs a Rust, C++ or Cython core would remove that numba cannot; and whether a fused numba kernel would remove the curve term while staying inside the ecosystem rule (or would reimplement datashader's line drawing, which is the bespoke cost). The brief also carries three sub-questions about the same source for lenses that may not read it: for binning, how segments between samples are rasterized and weighted (which algorithm, antialiased or not, how a weight column enters through `ds.sum` and whether it reproduces `ds.count`); for counter-evidence, any throughput cliff or antialiasing cost visible in the code (per-segment allocation, single-threaded steps, reductions that serialize); for GPU, the line glyph's CUDA coverage and which reductions it supports there, including where sum, mean, var and std raise. Excludes query engines, node stores past RAM, GPU throughput and binning bounds. Moves: architecture E, the compute term of A's and B's time cells, verdicts 9b-i and 9b-ii, and the source-read cells for D (weights) and F (CUDA line coverage). Claims for verification: 2.

- **GPU and multi-machine (stories 4 and 6).** Reading slice: RAPIDS cuDF join and group-by throughput and memory at 1B rows on one GPU and with dask-cuda across several, with GPU model and memory; datashader's CUDA path at 0.19.1 as its docs, release notes and issues describe it (which glyphs and reductions run on cuDF, and the sum/mean/var/std raise; the source read of the line glyph's CUDA coverage is the compute-floor lens's); cuGraph's datashader integration and Graphistry's published scales, read for whether they rasterize edges or points; `distributed` docs at the tag matching dask 2026.7.1 on p2p shuffle, broadcast joins and worker memory limits; GeoSparkViz, HadoopViz and sort-last compositing. Search strategy: vendor docs and engineering blogs for claims, then papers and third-party reproductions for checks. Answer: whether one GPU (24 to 80 GB) holds the target and how fast; what multi-GPU or a cluster adds that one large-memory machine does not; whether a cluster is needed for the target at all; and why copy and memmap died on LocalCluster, from documented behavior (the repartition gather, worker memory thresholds). Excludes CPU query engines, node stores, binning and datashader's source. Brief carries the "Already found, unverified" block. Moves: architectures F and G, every cluster cell, verdict 9d. Claims for verification: 2.

**Already found, unverified (from `wiki/wiki/analyses/dask-route-memory-bounding.md`; start past these).** Handed to the prior-art, counter-evidence and GPU lenses. These are pointers the prior run carried beyond its voucher pool, not findings. A claim re-found here without new material is context, not a nomination.

Its "Unverified context" bullets, verbatim:

- Splitting a raster across workers, as sort-last rendering does (Rabenseifner CUG 2004; IceT SAND2011-2482C). It adds tasks and still ends in a full gather.
- GeoSparkViz at 1.7 billion points; vaex's per-thread state; MLlib histograms sized through `maxMemoryInMB`.
- Graphistry, cuGraph, Cornac and graph-tool positions on billions of edges.
- HEP accumulators (Fischer 2024); dask #9083 (tree floor is threads x (fan-in + 1) x state; a locked accumulator completed on threads); dask #9522 (default of 8 criticized for groupby); discourse 4168 (workers died at 1.6e9 pixels on the tree path); datashader #1514 (kernel speedups in progress).
- Fragility of `default`: `use_array_expr` "Eventually, this default will flip to True" against datashader's manual `da.Array` construction; compatibility commits #1317, #1335, #1341, #1390; spatialpandas #178 (points from `read_parquet`, closed 2026-10-01). The route's probes use only `from_map`, so Parquet input is untested.
- `to_delayed` surface concerns (dask #12318, `ProhibitReuse`, about `da.Array`) and uncached numba compile per new worker (datashader PR #1515).
- The scheduler event-loop chain, sum-versus-max in the memory model, and the three-term arithmetic's frames-versus-rows puzzle.

Its search scopes for 7b and 7c, verbatim:

- 7b (nobody renders graphs on a cluster, inconclusive). Searched: datashader issues and discourse, dask and distributed trackers, tool docs. Cluster datashader use exists for points and polygons (#668, #332, discourse 4168, all unverified). No report of graph or edge rendering on a cluster was found. This is an absence in these searches only.
- 7c (billions not worth drawing edge by edge, inconclusive). Searched: Graphistry's 2.53.0 and GFQL posts, the RAPIDS cuGraph with datashader post, Cornac, Zinsmaier et al. and graph-tool 3.9 docs (all unverified, several snippet only). No precedent found for drawing billions edge by edge. That those tools collapse first is unverified, and Rübel and imMens are not graph tools. Large-graph tools carry layout and interactive costs a fixed-layout static raster does not, so even a confirmed norm would make the route unusual without making it wrong.

**Model allocation (maintainer's instruction, relayed by the dispatching session, 2026-11-02).** Lenses, vouchers and the synth run on Opus. The adversary gates on Fable (the plan-stage pass and the convergence gate), as does the dispatching session's own validation. The amend-plan call disposing the plan-stage challenge ran on Opus 5.5. This closes the model half of questionnaire item 6.

**Addable pre-flight estimate (binding), counted against every Agent call in the run. Revised to 30 by Amendment 1.**

```
4  spent              record-keeper pre-task, this research-plan, plan-stage adversary,
                      amend-plan disposing the plan-stage challenge (grill skipped)
7  lenses             2 standing + 5 orthogonality
14 verify vouchers    2 per lens
2  second vouchers    generation-flagged, single-quote nominated claims
1  synth
1  adversary          convergence gate
1  record-keeper      producer path
= 30 Agent calls in all, 26 still to fire
  (25 by the research-track model: 1 scope + 7 lenses + 16 vouchers + 1 synth;
   the other 5 are the pre-task pass, both adversary passes, the amend-plan and the landing)
```

The surfaced estimate of about 27 listed 15 vouchers and no amend-plan call; the plan first held 27 by funding the amend-plan from the voucher pool. Amendment 1 splits the query-engines-and-storage lens in two, adding one lens and its two vouchers, so the binding estimate is 30. The dispatching session surfaces 30 to the maintainer in chat before the lens wave fires.

**Overrun.** Any further revision is surfaced in chat and recorded in `## Plan amendments` before the call that would exceed the standing estimate fires, and the run proceeds under the revised figure. The run halts only if a revision would take the total past 36 (the previous run's 31 plus headroom): it then parks, fires no further Agent call, and the dispatching session asks the maintainer. The consumption line is checked against the last surfaced estimate.

**Concurrency ceiling: 8.** Peak expected is 7 during the lens wave; vouchers run in waves of at most 8. Second votes go to load-bearing flagged claims first. A nominated claim beyond a lens's allotment is labelled `unverified` and treated as ungrounded at the gate.

**Token consumption** is reported, not gated, with its basis named (task-notification subagent figures, otherwise "unknown").

## Validation criteria

A validated finding is grounded (faithful quote, source version or date), independently verified (claim-maker is not the voucher, refuted if uncertain, a second voucher on self-flagged claims) and cleared by the adversary at convergence against the rubric above. Bound to this run:

- **Architectures, not stories.** The synth ranks at most seven, and a story that fails drops out with one line saying why. Seed list, which the synth may merge or cut: (A) today's stack with a lazy-node copy table on one machine; (B) A with an out-of-core node table read sequentially; (C) an embedded query engine doing the join, feeding datashader in batches; (D) collapsing or sampling before drawing, composable with A to C, in three variants: exact dedup, quantized rho binning, and Bernoulli edge sampling at fraction p; (E) a compiled core or a language switch, including a fused sample-and-draw kernel; (F) GPU, one or several; (G) a cluster.
- **Every architecture carries six cells.** What it takes (code, dependencies, API consequences against each standing rule it touches); what it reaches (edges, nodes, hardware, time, at `num_steps` 16 and 50 where they differ, judged at both thresholds of questionnaire item 2); read-from-storage (the cost of reading edges and nodes from the user's storage, reported in its own cell and never an exclusion; no row in `## Inputs` measures it, so it reads `unmeasured` wherever no lens supplies it); exactness; branch 53 or later (questionnaire item 8); and which term binds first at the item 1 target.
- **Rank within evidence class first.** The synth orders architectures within each evidence mark before ordering across marks, and says in one line wherever A leads only because its cells are `measured here` or `predicted` while a rival's are `literature-supported` or `unmeasured`. Until the pending rows land, A may not rank first on a `predicted` cell alone (constraint in `## Inputs`).
- **Exactness marks.** One of: `exact` (raster hash matches the library); `within the ADR 0002 wall` (curve arrays to float32 tolerance and raster count agreement at the validated mismatch fraction); `rounding-only` (float reductions differ by summation order, with the stated bound); `lossy, bounded` (with the bound and its source); `lossy, unbounded or unmeasured`. A binned option cannot pass the curve-array half of the wall by construction, and says so.
- **Evidence marks.** `measured here` names an `## Inputs` row or jsonl file and its scope. `predicted` names the rows the arithmetic comes from. `literature-supported` names a validated claim. `read-from-source` names version and file and needs a voucher like any lens claim. `unmeasured` covers everything else. Cells carried from `## Inputs` need no voucher. A cell whose only support is an unverified claim reads `unmeasured`.
- **Cluster cells** read exactly one of: `known mechanism of harm: <mechanism, documented or inferred>`; `no known mechanism of harm, unmeasured (searched: <what>)`; `unknown`. The LocalCluster probe is one configuration on one machine and is cited as that. "Safe on a cluster" and its synonyms are killed.
- **Ranking order** is stated, from questionnaire item 2 or its recommended answer, so the weights can be attacked at the gate. The synth reports the ranking twice, once at each "fast" threshold (about one minute; under 10 minutes), and names every architecture whose place changes between them.
- **Story 9.** Five verdicts (9a, 9b-i, 9b-ii, 9c, 9d), each `supported | not supported | inconclusive` with evidence; 9b-i is reported at both thresholds, and 9d is judged in the scope stated under `## Question`. The counter-evidence lens's hits reach the page labelled not-fully-confirming.
- **Proposed experiments.** What only a measurement can settle (a sorted-access memmap, a DuckDB join at 100M and 1B nodes, Parquet-backed edges, read-from-storage at the target, skewed degree, and the queued 1B-edge rows at `num_steps` 16 and 50 plus the 100M / 100M breakdown if they have not landed by synth) becomes a named proposal under `What's open`, carrying what it would settle. No lens runs one.
- Findings are framed as correlational pointers. "Route X is slow because of Y" needs a mechanism plus a row.

Terminal outcome, classified honestly:

- **validated finding**: a confident, grounded ranking. Lands the full page.
- **validated inconclusive**: pursued to a negative, such as "the literature does not settle whether an embedded engine beats the copy route at 1B nodes". Lands the full page as a do-not-re-research reference.
- **nothing-cohered**: the panel ran and nothing of interest cohered. Lands a minimal breadcrumb only.

A mixed page is the expected shape: the one-machine half validated, the cluster and GPU halves possibly inconclusive.

### Summary shape the run reports

```
Question: <the refined research question>
Evidence mode: literature (measured inputs carried from the fork's experiments, none produced by a lens)
Lenses run (each disjoint, blind to the others; the two standing seats floored):
  - prior-art / counterfactual (who draws this much): <distilled conclusion + citation>
  - counter-evidence: <what would falsify the stories; not-fully-confirming hits>
  - query engines: <distilled finding + citation>
  - node-table storage: <distilled finding + citation>
  - binning exactness: <distilled finding + citation>
  - compute floor and language (sole reader of datashader's line-glyph source): <distilled finding + citation>
  - GPU and multi-machine: <distilled finding + citation>
Findings (each with citation, hardware and version scope, independent-verification verdict):
  - [validated] ... (voucher re-read the cited source; default-reject cleared)
  - [killed] ... (claim-maker != voucher; cited source did not support)
  - [unverified] ... (beyond the voucher pool; ungrounded at the gate)
Ranking order used: <from questionnaire item 2, answered or recommended>
Pending-rows status: <landed (dated note in ## Inputs; constraint lifted) | not landed (story 8 predicted only; A not ranked first on a predicted cell alone)>
Architectures ranked, twice (at about one minute, and at under 10 minutes; architectures that move between the two named):
  <A-G as merged: what it takes | what it reaches | read-from-storage | exactness mark |
  branch 53 or later | binding term at the target; each cell marked measured here |
  predicted | literature-supported | read-from-source | unmeasured; cluster cell in one
  of the three permitted forms; ranked within evidence class first, with a line wherever
  A leads only because its cells are measured>
Maintainer-framing verdicts (story 9), each supported | not supported | inconclusive, with evidence:
  - 9a a DuckDB story: ...
  - 9b-i the Python-stack floor at the target sits above "fast" (at each threshold): ...
  - 9b-ii a compiled core or language switch lowers it, by a named mechanism: ...
  - 9c a database structure or storage layout supports this: ...
  - 9d feasible-fast needs much larger structural changes (upstream of the raster; renderer fixed): ...
Proposed experiments: <what each would settle; none run by this run>
Adversary verdicts (convergence gate): validated / killed, each with reason + evidence
Open questions: ...
Yield (worth-landing determination): <validated finding | validated inconclusive | nothing-cohered>
Pre-flight estimate: 30 Agent calls (25 by the research-track model; binding; revise-and-surface to exceed, park past 36)
Resources consumed: <n> agents, peak concurrency <n> (ceiling 8), ~<n> tokens (basis: task-notification subagent figures; "unknown" when unmeasured)
Destination: <full page at the path below, or the minimal breadcrumb>
```

## Destination artifact

New page at `wiki/wiki/analyses/scale-architecture.md`, created through record-keeper's producer path under maintainer approval, for a validated finding or validated inconclusive. A nothing-cohered run lands a minimal breadcrumb at that path instead. No analyses page covers this, so it is a new page, not an enrich or rebuild. Schema: What was established / What was validated, and how / What was inconclusive, and why / What's open / Sources / See Also, with `[[dask-route-memory-bounding]]` and `[[hiveplotlib]]` in See Also. The ranked architectures sit under What was established, each with its six cells. Agents do not commit; the maintainer reviews and commits. Repo code and docs must never cite this plan or the page.

This run authorizes no code, docs or notebook change.

## Minor pivots vs. amend-plan

Iteration lives inside the bounded run: refining a lens or dropping a dead angle needs no amendment. Only a fundamental change of the research question is an amend-plan.

## Alignment (grill)

```
Status: research-branch grill knowingly skipped, 2026-11-02.
The maintainer's review time is scarce, so the grill is parked rather than run live. The
failure-mode rubric above is orchestrator-seeded as a result. Open decisions are parked in
## Maintainer questionnaire; the run proceeds under each item's recommended answer unless
the maintainer says otherwise.
Delivery shape: n/a (research run; no MR).
```

With the grill skipped, nobody fights the adversary's plan-stage challenge live, so it routes to `amend-plan` for explicit disposition before the lenses fire. That call is in the estimate. Disposed 2026-11-02 in Amendment 1: all 11 items accepted, each disposition decided by the dispatching session under the async posture and open to the maintainer's re-litigation at review.

## Adversary review

### Adversary's challenge (planning mode)

```
Status: challenge (11 items)
Plan reviewed: wiki/wiki/plans/scale-architecture-research.md (plan stage, cold, 2026-11-02, before any lens fires)
Spec: none (research plan). The spec checks are out of scope. The rubric below is orchestrator-seeded (grill knowingly skipped), so every "Rubric:" line names the model's standard, not the maintainer's.
Tags: must-fix / should-fix / note, as the dispatch asked (should-fix = worth-discussing, note = low-confidence in the adversary's own grammar).
Angles worked: premise (reopened only where the plan narrows the maintainer's words: items 1, 7, 8); approach (smaller moves unmentioned: items 2, 6); size-and-maintenance against the lens set and voucher pool (items 3, 4, 5, 9, 10). Could-this-not-exist: not a kill. The prior run never searched a compiled core, DuckDB, GPU or out-of-core storage, and those are the four framings the maintainer named, so this run adds what his question requires. What it does not require is item 2's prediction standing in for a 10-minute measurement.
Files opened: agent-harness/.claude/expertise/adversary.md, agent-harness/.claude/expertise/personal-gotchas.md, .claude/skills/research-track/SKILL.md, this plan, wiki/wiki/analyses/dask-route-memory-bounding.md, wiki/wiki/plans/dask-route-memory-research.md:128-135 (ranged), greps on src/hiveplotlib/hiveplot.py, src/hiveplotlib/*.py (num_steps defaults), wiki/wiki/adr/0002-performance-regression-harness.md, wiki/wiki for "DuckDB".

Items:
  - [must-fix] Questionnaire item 2's recommended threshold sits where the plan's own prediction for architecture A lands, so the recommended answer decides the ranking and verdicts 9b and 9d before a lens runs. (at ## Maintainer questionnaire item 2; ## Inputs "The stated target is unmeasured")
    Rubric: "Feasible fast stretched" (judges against item 2, so it inherits whatever item 2 says); "Premise confirmation".
    Push: Inputs predicts about 210 s of edge render for 1B edges at 100M nodes at num_steps=16; item 2 says fast is under 10 minutes on ringtail. On that arithmetic A is "fast" at 16 and plausibly at 50, so under the recommended answer A ranks first, 9b reads "not supported" and 9d "not supported" by construction. Write that dependence into item 2's Context (quote the 210 s and say A passes at 10 minutes and fails at 1 minute), offer the threshold the maintainer may actually mean (a plot call a person waits on: about a minute), and require the synth to report the ranking at both thresholds so the gate can attack the weight. Second half: "excluding reading from the user's storage" removes the one cost where 9a and 9c (an engine, a storage layout) differ from A at all; make read-from-storage a reported cell in "what it reaches", not an exclusion from the definition of fast.

  - [must-fix] Story 8 ("the target is already within reach") has no lens that can move it, and it is the story the maintainer's question turns on. The only thing that can move it is a measurement the plan forbids itself. (at ## Candidate stories 8; ## Lenses + bounds "Moves:" lines; ## Question "Bounds")
    Rubric: "Extrapolation presented as measurement"; "End-to-end laundered as edge render".
    Push: no lens's Moves line names story 8; the counter-evidence lens is told to falsify it and nothing is told to confirm it, so it reaches the synth as the orchestrator's arithmetic. The target row (copy route, 1B edges, 100M nodes) is about 4 minutes of ringtail time at 16 and perhaps 15 at 50 on the fork's existing scripts, at 9 GiB predicted peak, and costs zero Agent calls. Same shape as the prior run's item 6 ("the cheapest route to the decision is more probes", `dask-route-memory-research.md:281`), where probes (a) and (b) ran before the lenses. Run that row at 16 and 50 (three fresh processes is enough) plus a by-phase breakdown of the measured 100M / 100M row (lookup, curve sampling, materialization, rasterize; py-spy or cProfile) before the lens wave, and add both to ## Inputs. The lenses then attack a measured target instead of a prediction, and the compute-floor lens's "what fraction of the 21 s" question checks a profile instead of guessing one. If the maintainer rules no further probes, record that on the plan, mark story 8 `predicted` only, and forbid the synth from ranking A first on a predicted cell alone.

  - [should-fix] The query-engines-and-storage lens carries two stories, two architectures, two verdicts and about fifteen bodies of material on two nominations. It is the lens the research-track skill's deep-exploration revival trigger describes. (at ## Lenses + bounds, third lens)
    Rubric: no entry; flagging anyway (lens shape is not a finding mode).
    Push: split it into (C) the engine join, story 2, verdict 9a, reading DuckDB, Polars streaming, DataFusion, Acero and narwhals lazy backends; and (B) the node-table store and sort-merge versus random lookup, story 5, verdict 9c, reading Arrow IPC, Parquet row groups, memmap, LMDB and the join-algorithm literature. N goes to 7 and the estimate to 30 (one lens, two vouchers), surfaced now rather than mid-run. If the estimate must hold at 27, cut the slice instead: ClickHouse, Zarr, SQLite and RocksDB become `unverified` context or move to counter-evidence, and say which.

  - [should-fix] Four lenses read the same file, datashader 0.19.1's line glyph: compute floor ("where it parallelizes, what it allocates per segment"), binning ("which algorithm, antialiased or not, how weights enter via ds.sum"), counter-evidence ("throughput cliffs, antialiasing, per-segment cost") and GPU ("the line glyph specifically"). Disjointness is claimed for the lens set and false for that file. (at ## Lenses + bounds, reading slices)
    Rubric: "Grounding failure" (each is a read-from-source claim needing its own voucher, so one file costs up to four vouchers); the skill's "non-overlapping material" rule is the standard.
    Push: assign the 0.19.1 line-glyph source read to the compute-floor lens and list the other three lenses' sub-questions in its brief (antialiased or not; how a weight column enters through ds.sum and whether it reproduces ds.count; which glyphs and reductions run on cuDF and the sum/mean/var/std raise; per-segment allocation and thread behavior). The other three cite published material about it and hand source questions back. The hub cannot reconcile four blind readings of one file, since it never sees the sources.

  - [should-fix] The voucher pool cannot carry the ranking's cells, and the evidence-mark rule then ranks evidence availability, not architectures. (at ## Validation criteria "Every architecture carries five cells", "Evidence marks"; ## Lenses + bounds estimate)
    Rubric: "Scope laundering across hardware" is the nearest entry; it covers hardware, not evidence mark.
    Push: seven architectures times five cells is 35 cells plus four verdicts. A's cells come free from ## Inputs; B to G share at most 12 validated claims (2 per lens), and a cell resting on an unverified claim reads `unmeasured`. So A is the only architecture with numbers in its cells, and the order "exactness, fits, time, dependencies" puts it first by construction. The prior run's item 4 chose cap-per-lens over a larger pool (`dask-route-memory-research.md:404`) and its page carried most non-A cells as `unmeasured`. Either raise the allotment to 3 on the two lenses that carry two architectures each (query engines, GPU and multi-machine), estimate 27 to 29; or add a validation criterion that the synth ranks within evidence class and says in one line when A leads only because its cells are measured, with a matching rubric entry: "Evidence-mark laundering: a ranking that orders a `measured here` or `predicted` cell against a `literature-supported` or `unmeasured` cell says the comparison is across marks."

  - [should-fix] Edge sampling is the smaller move the approach angle asks about, and it appears nowhere: not in the stories, the seed list, the settled list or the net-new list. (at ## Candidate stories 3; ## Validation criteria seed list; ## Prior rulings and records "Settled")
    Rubric: "Already settled" (it is not: the settled "adaptive sampling" is curve-point sampling, not edge sampling); "Binning exactness asserted, not bounded".
    Push: for ds.count a Bernoulli sample at fraction p gives an unbiased raster scaled by 1/p with a per-pixel binomial bound, and 1B edges at 1% is a 10M-edge render the copy route already does in 3.6 s at 2M nodes (## Inputs row 2). It is lossy and fails the ADR 0002 wall by design, the same class as D's quantized variant, and item 3 already admits lossy power-user options. Add it as a second lossy variant of D (story 3) or as architecture H, with its bound stated and which reductions survive (count and sum do; max, min, first, last, by do not), and let the prior-art lens's "what can a 1500 x 1500 canvas show of 1B edges" question feed its design-judgment cell. If it is out, say why in ## Prior rulings and records so a lens does not propose it as new.

  - [should-fix] Verdict 9b bundles two claims the maintainer said as one, and the rubric makes the second half unsupportable while the first half is a measurable floor claim. (at ## Candidate stories 9b; ## Validation criteria "Story 9")
    Rubric: "Premise confirmation" (one blended verdict is killed; the plan's own verdict form blends two); "Language switch with no named cost".
    Push: his words are "it just has to be slow if we do things in Python and that suggests that we need to switch languages." A "not supported" on the bundle reads as "Python is not slow" when the evidence may say "slow at the target, and a switch does not fix it", which is the answer that actually guides him. Split into 9b-i (the Python-stack floor at the item 1 target sits above item 2's "fast") and 9b-ii (a compiled core or language switch lowers that floor by a named mechanism), each supported / not supported / inconclusive with its own evidence.

  - [should-fix] ## Question narrows the maintainer's question in a place his words do not: "hiveplotlib's datashader route" and "a static raster on a fixed canvas". His 9d ("much larger structural changes to ever be feasible fast") is the alternative that framing excludes. (at ## Question, first sentence; ## Prior rulings and records, log 2026-09-30)
    Rubric: "Premise confirmation".
    Push: the 2026-09-30 ruling fixed the renderer (datashader stays the at-scale answer against Datoviz and GPU plotting libraries); it did not fix the pipeline upstream of the raster. State in ## Question that "architecture" ranges over everything upstream of the raster and the ruling holds only the renderer fixed, or state that 9d is judged within the datashader route so a "not supported" is read with that scope. As written, a lens can hand 9d back as out of bounds.

  - [should-fix] The binding-estimate clause under the async posture says what to write and not what the run does next. (at ## Lenses + bounds, "Exceeding 27 requires surfacing a revised estimate first; under the async posture, park it")
    Rubric: no entry; flagging anyway. The skill's "never a silent overrun" is the standard.
    Push: "park it in ## Maintainer questionnaire and record it" followed by proceeding is an overrun with a note attached; followed by halting it is a parked run nobody wakes. State which. If it proceeds, name the ceiling it proceeds to (the revised estimate, or 27 plus the number of generation-flagged claims, whichever the dispatching session will actually count against), so the consumption line can be checked against a number the plan stated.

  - [note] Three lenses re-cover the prior run's unverified-context list and its 7b / 7c search scopes, and nothing hands them that list, so they will spend reads re-finding it. (at ## Lenses + bounds, prior-art, counter-evidence and GPU briefs; ## Prior rulings and records "Findings")
    Rubric: "Already settled" (covers rulings, not prior searches; extend it).
    Push: paste the prior page's "Unverified context" bullets (Graphistry, cuGraph, Cornac, graph-tool, GeoSparkViz, vaex, sort-last, discourse 4168, datashader #1514) and its 7b / 7c "Searched" lines into the three briefs as "already found, unverified; start past these", and add to the rubric that a claim re-found from that list without new material is context, not a nomination.

  - [note] Factual drifts a lens will inherit from the plan text. (at ## Prior rulings and records, scaling-large-networks bullet; ## Inputs first paragraph; ## Maintainer questionnaire item 2)
    Rubric: "Stale numbers" in spirit; no exact entry.
    Push: (a) "the wiki's only DuckDB mention" is false in letter: `wiki/wiki/plans/dask-route-memory-research.md:132` excludes DuckDB from a lens's reading because it combines "group state and not a fixed raster"; no ruling, so "deferred, not ruled" stands, but fix the sentence so a lens citing it is not refuted on it. (b) ## Inputs says 30 GiB visible to WSL and item 2 says 32 GiB; the copy route at 300M nodes peaked at 25.70 GiB, so a "fits on target hardware" cell at the margin depends on which number the ranking uses; pick one. (c) For the record, `hiveplot.py:1742` (`edge_ids.copy().astype("O")`) and the num_steps=100 defaults (`hiveplot.py:1631,1704`, `utils.py:75,96,254`, `p2cp.py:231`) are confirmed on the working tree.
```

### Adversary post-impl

```
Status: propose (research-run convergence gate, 2026-11-02, cold)
Artifact reviewed: scratchpad research_summary.md (synth) against lens1 to lens7, voucher_verdicts.md (V1 to V16 plus tally), late_inputs.md
Dispositions held: yes. Amendment 1's constraint held in letter (story 8 predicted only; first place withheld); the ranking reports both thresholds; six cells per architecture; verdicts split five ways.
Spec-gate re-grade: n/a (no spec governs this plan)
Files opened: agent-harness/.claude/expertise/adversary.md, agent-harness/.claude/expertise/personal-gotchas.md, .claude/skills/research-track/SKILL.md, this plan (two ranged reads, whole file), the summary, lens1_prior_art.md, lens2_counter_evidence.md, lens3_query_engines.md, lens4_node_storage.md, lens5_binning.md, lens6_compute_floor.md, lens7_gpu_cluster.md, voucher_verdicts.md, late_inputs.md. No source, no spec, no jsonl.

Yield verdict: validated finding, half. Validated: the mechanism set (F1, F2, F4, F5, F7, F9, F10, F12, F14, F16, F17 bound, F18, F19 in V4's wording), verdict 9b-ii, and 9c narrowly (n=2 of 5, in flight, 100M edges). Validated inconclusive: the ranking at the target (first place withheld; C, E, F time cells unmeasured; no row at num_steps 50; no storage read), 9a, 9b-i at batch, 9d. Not nothing-cohered. The page lands in full with the split stated in its first paragraph.

Per-finding verdicts where the tag changes:
  - 9b-i at batch: downgraded, "not supported" to "inconclusive, leaning not supported at num_steps 16 excluding storage". The judged target (questionnaire item 1 recommended answer) is num_steps 50 with storage included; the 210 s is at 16, edge render only, and the summary's own A cell reads 50 and storage `unmeasured`. Rubric: Extrapolation presented as measurement; Workload artifact; Feasible fast stretched. Evidence: summary A cell; plan ## Inputs "A prediction, never a measurement, and it excludes reading 1B edges from storage and num_steps=50".
  - 9d: downgraded and split. Reported once as "not supported" when the batch half rests on the predicted A cell (the by-construction outcome the accepted plan-stage item 1 named) and the interactive half is carried by "no exact architecture meets interactive on any figure in hand", which argues for 9d or for inconclusive, not against it. Report at both thresholds: batch inconclusive (predicted only, held rows decide); interactive inconclusive (what might reach a minute, E and F, is unmeasured). Rubric: Premise confirmation; Extrapolation presented as measurement. Evidence: summary 9d line; Amendment 1 row 1.
  - E and F in the exact tiers (batch "Exact, unmeasured: D-exact, C, E, F"; interactive "E and F rise to the top, the only exact mechanisms"): killed placement. Both carry `lossy, unbounded or unmeasured` in their own exactness cells, and the stated order makes exactness a floor. E's "near a minute" rests on F21 [unverified], which the criteria say reads `unmeasured`. Rubric: Evidence-mark laundering; Ranking order; Grounding failure. Evidence: summary E and F exactness cells; F21.
  - E's fused kernel: narrowed. A kernel that reimplements clip, snap and Bresenham is a second rasterizer; the 2026-09-30 ruling keeps datashader the renderer, and the ecosystem rule needs the named reason stated in the cell (lens 6 offers the memory cliff at 50, unmeasured), with "contribute a curve glyph upstream" as the ecosystem path. Rubric: Database recommendation that breaks the standing rules (bespoke rasterizer). Evidence: summary E Takes cell; lens 6 conclusion 5.
  - D-exact `exact (analytic)`: downgraded to analytic, unvouched. The derivation joins lens 5's analysis and lens 6's source read of reductions.py:1056-1063; no voucher re-derived either, and `analytic` is not one of the plan's five evidence marks. Rubric: Grounding failure; Evidence marks. Evidence: lens 5 conclusion 1; lens 6 sub-question answer; validation criteria "Evidence marks".
  - F17's application to hiveplotlib (`hiveplot.py:1806-1808`): narrowed. The Bezier bound is validated (V5); the two code assumptions were read by the dispatching session, not a voucher. Rubric: Grounding failure (read-from-source needs a voucher). Evidence: V5 dispatcher note.
  - 9c: narrowed. State 100M edges, n=2 of a planned 5, in flight, and "feasible, not fast": the 25-minute table build plus a 58 to 77 s edge render fails both thresholds. "Settles 9c" in the Yield becomes "supports". Rubric: n too small, uncontrolled comparison; Feasible fast stretched. Evidence: late_inputs.md table.
  - Lenses-run line "the one ~1B-edge graph found collapses into glyphs (F19)": narrowed to V4's wording (buildings per maximal connected subgraph; whether bottom-level edges are drawn is unverified). The killed F20 wording re-enters here. Rubric: Grounding failure. Evidence: V4 (b).
  - Lenses-run line "all compiled and GIL-releasing except the sampler": narrowed. V2 vouches datashader's ngjit nogil; numpy searchsorted releasing the GIL is lens 6 unverified context (numpy main, not the installed tag); V1 says the pandas merges have mixed GIL behavior. Rubric: Grounding failure. Evidence: lens 6 unverified context; V1 corrections.
  - Lenses-run line "datashader: 1B OSM points in 1.98 s, 16 GB laptop": label `unverified`, version unstated, 2016 page. Rubric: Stale numbers; Vendor benchmarks at face value. Evidence: lens 1 conclusion 1 (not nominated).
  - Yield "four of five verdicts are grounded": downgraded. Grounded on validated or measured material: 9b-ii and 9c (narrow). 9a is inconclusive; 9b-i and 9d rest on the prediction. Rubric: Premise confirmation.

Checks that passed: every [validated] F-line carries its voucher's corrected wording and scope (F1, F2, F4, F5, F7, F9, F10, F12, F14, F16, F18, F19); killed wordings listed separately (F3, F6, F8, F11, F13, F15, F20); F21 to F24 labelled unverified; both thresholds reported with movers; the laundering note is present; the file-peak artifact is stated and the column omitted; N1 to N10 carry all ten counter-evidence items; the late bucketed row is scoped n=2 and 100M edges in the table and in B's cell; story 8 stays predicted and first place is withheld; no recommendation reverses pass-through, the renderer ruling or the lazy-nodes-datashader-only rule except E as narrowed above; B and C name their rule costs.

Must-fix edits before landing (record-keeper applies):
  1. Rankings: remove E and F from both exact tiers and delete "the only exact mechanisms"; rank them in an "exactness unmeasured" tier below D-exact and C; add E's renderer-ruling clause.
  2. Verdicts: 9b-i batch to inconclusive as above; 9d reported at both thresholds, inconclusive at both, naming that the batch half rests on the predicted A cell.
  3. Add the points-versus-segments resolution the rubric demands, in V8's and V2's words: D-exact, D-quantized and C feed `Canvas.line` with a weight column, segments preserved; a points glyph or a GROUP BY over sample points is out, since matching needs consecutive samples in the same or neighboring cells, a necessary and not sufficient condition (V8), and raising num_steps to reach it inflates `count()` through sub-pixel appends (V2). Each D variant and C names the glyph it feeds.
  4. Define `analytic` on the page as a lens derivation, unvouched and unmeasured, ranking below `literature-supported`, or replace it with `unmeasured`; label D-exact's exactness accordingly.
  5. Yield line: replace "four of five verdicts are grounded" and "settles 9c" with the half-finding split above.
  6. Pre-flight line reads "31 Agent calls (revised twice from 27)"; the plan records 30 by Amendment 1 and no second revision. Reconcile before landing: record the second revision under ## Plan amendments per the Overrun clause, or correct the line to 30. Fill Resources consumed with its basis.

Should-fix edits:
  - 9c verdict: add "100M edges, n=2 of 5, in flight" and "feasible, not fast (25-minute build)".
  - Stretch-row sentence "the only exact architecture with a measured 1B-node render in minutes": say an edge render of about a minute after a 25-minute table build at 100M edges; the 1B-edge scatter disk is unestimated.
  - A's Reaches cell "N1 says low": N1 says the 5M edges/s reading is not constant and 24-thread scaling is unsupported; the 210 s already uses the 100M-node row. Correct the attribution.
  - D-exact: carry lens 5's per-reduction conditions (max and min need group-carried values; by needs the category in the key; first and last need a carried row index, which ds.where may block).
  - F's Takes cell cites "project memory"; cite scaling-large-networks.md's record plus numba-cuda #384 (unverified) instead.
  - 9b-ii: name the one unmeasured candidate mechanism (prefetched, batched lookups, lens 6 conclusion 4) so "not supported" reads as "no measured mechanism".
  - Label every dispatcher-read fact on the page (sampler called by the probe; 1M-edge partitions; the F17 code assumptions) as hub reads without a voucher.
  - Carry lens 2's and lens 3's blindness disclosures (both read other lens briefs in the plan) into the page's provenance.

Notes:
  - Lens 2 item 9 also cited Nöllenburg and Wallinger (arXiv 2309.02273); N9 dropped it. Add or say why.
  - About eleven claims self-flagged for a second vote and two were funded (V12, V13); list which validated claims stand on a single vote with an unfunded flag (lens 1 A and B, lens 2 B, lens 3 claim 1, lens 4 claim 2, lens 5 claims 1 and 2, lens 6 claim 3), per the skill's no-silent-triage rule.
  - Proposed experiment 1's partition-size inconsistency (10M in the plan, 1M in a dispatcher note) is correctly surfaced; keep it.
  - The open question on the cuDF float-reduction record: the maintainer holds a machine-verified diagnosis outside the wiki; the page says "suspected toolchain fault" until he confirms it on the page.

First sentence for the maintainer: Nothing in this run shows the 1B-edge, 100M-node target needs a new architecture or a language switch, and nothing measures the target either; the one new measurement is that at 1B nodes a bucketed sequential join cuts the 100M-edge render from two hours to about a minute (n=2, in flight) behind a 25-minute table build, while the target itself stays a 210-second prediction at num_steps 16 until the held rows run.
```

## Maintainer questionnaire

Parked 2026-11-02 because the grill is skipped. Answer in chat; a one-word reply closes an item. The run does not block on any of these and proceeds under each recommended answer.

```
1. What is the target the ranking judges against?
   Context: every architecture's "reaches" and "binding term" cells are judged here. The
   1B-edge, 100M-node figure is the proposed working target from 2026-10-31, unconfirmed.
   Every measured row is at num_steps=16; the signed default is 50.
   Recommended answer: 1B edges and 100M nodes, 3 axes, the default 1500 x 1500 canvas,
   judged at num_steps=50 with 16 reported as the power-user setting. 1B nodes is a stretch
   row each architecture reports on, not the target.
   Answer (on close):

2. What counts as "feasible fast", and in what order do the cells rank?
   Context: the threshold decides the ranking before a lens runs. ## Inputs predicts about
   210 s of edge render for architecture A (copy route) at 1B edges / 100M nodes,
   num_steps=16, 8 threads. Under a 10-minute threshold A passes at 16 and sits near the
   line at 50, so A ranks first and 9b-i and 9d read "not supported" by construction; under
   a one-minute threshold A fails at both. The measured rows run from 21 s (copy, 100M
   nodes / 100M edges) to 2 hours (memmap, 1B nodes / 100M edges). No row reads edges from
   storage, and that read is the one cost where an engine (9a) or a storage layout (9c)
   differs from A at all, so excluding it from "fast" would hide the case for both.
   Recommended answer: two thresholds, and the synth reports the ranking at both.
   Interactive fast: the whole render finishes in about one minute on one workstation like
   ringtail (24 cores, 30 GiB visible to WSL, local SSD), a plot call a person waits on.
   Batch fast: the same, under 10 minutes. The whole render covers edges, nodes and reading
   both from the user's storage; read-from-storage is its own reported cell, never an
   exclusion. Feasible means it finishes within a few hours on hardware one person can
   rent (one large-memory VM or one GPU). Order: exactness is a floor for any default; then
   fits on the target hardware; then time; then dependency and maintenance surface.
   Answer (on close):

3. May a lossy architecture (story 3: the quantized variant, or edge sampling) enter the ranking?
   Context: same question as item 2 of dask-route-memory-research, still unanswered there,
   now covering edge sampling at fraction p too. The exact-dedup variant is lossless and
   needs no ruling.
   Recommended answer: yes, both, as power-user only with the error bound stated, never a default.
   Answer (on close):

4. May the ranking include a new compiled dependency such as DuckDB?
   Context: narwhals is a core dependency because it is pure Python with one wheel. DuckDB,
   a RAPIDS stack or a Rust extension each add compiled wheels and platform coverage.
   Recommended answer: yes, ranked as an optional extra behind an import guard naming the
   extra, with user frames still passing through narwhals and no canonical internal frame. A
   required dependency is ranked too, with that cost stated.
   Answer (on close):

5. Is a compiled core or a language switch inside the ranking at all?
   Context: hiveplotlib's compiled work today runs through numba and datashader. A Rust or C++
   core changes the build and the release (platform wheels, the conda-forge plan).
   Recommended answer: yes, ranked like any architecture with its build and distribution cost
   stated, and only if it names the measured cost it removes. No veto up front.
   Answer (on close):

6. Accept the revised binding estimate of 30 Agent calls, with a park-and-ask ceiling of 36?
   Context: the surfaced estimate of about 27 had 15 vouchers and no amend-plan call, which
   the skipped grill makes necessary; the plan first held 27 by funding that call from the
   voucher pool. The plan-stage adversary then pushed to split the overloaded
   query-engines-and-storage lens, and the dispatching session accepted: one more lens plus
   its two vouchers makes 30 (4 spent, 7 lenses, 16 vouchers, synth, gate, landing). The
   dispatching session surfaces 30 in chat before the lens wave fires. Model allocation is
   closed by the maintainer's instruction (lenses, vouchers, synth on Opus; the adversary
   gates on Fable).
   Recommended answer: yes. The run proceeds under 30 and under any later surfaced
   revision, and parks to ask only if a revision would pass 36.
   Answer (on close):

7. Does the orchestrator-seeded failure-mode rubric stand in for the skipped grill wave?
   Context: the rubric is normally elicited in your words; the adversary reads it at
   convergence.
   Recommended answer: yes for this run; add or strike entries at review.
   Answer (on close):

8. Does the page sort architectures into branch 53 and later?
   Context: scaling blocks branch 53 so it is settled once, before it is part of the API, and
   the lazy-node lookup design is still undecided. A language switch or a GPU stack cannot
   land on 53; a lookup choice can.
   Recommended answer: yes. Each architecture is tagged "branch 53" (reachable with today's
   dependencies or an optional extra, and decides the lazy-node lookup) or "later". The page
   also says, for each later one, whether the branch 53 API would foreclose it.
   Answer (on close):
```

## Plan amendments

### Amendment 1 (2026-11-02): plan-stage adversary challenge disposed (grill skipped)

Source: `### Adversary's challenge (planning mode)`, 11 items. Every disposition below was decided by the dispatching session under the async posture and is open to the maintainer's re-litigation at review. All 11 accepted; the edits are applied in the sections named. The research question is unchanged in substance (item 8 widens its stated scope to match the maintainer's words), so this stays an in-run amendment, not a new plan. No spec governs this plan.

| # | Tag | Bucket | Disposition and where it landed |
|---|---|---|---|
| 1 | must-fix | In-scope tweak | Accepted. Questionnaire item 2 now states that its threshold decides the ranking (quoting the 210 s), adds a second "fast" threshold of about one minute (a plot call a person waits on) beside under 10 minutes, and makes read-from-storage part of "whole render" and its own reported cell. Validation criteria: six cells, ranking reported at both thresholds with movers named. Rubric "Feasible fast stretched" and the summary shape follow. |
| 2 | must-fix | In-scope tweak | Accepted with a constraint. The probe is authorized by the maintainer's standing "run the throwaway experiments" ruling, but ringtail is occupied by the sibling session's 1B-node `bucketed` row (two-hour timeout, started 07:14), so the dispatching session has asked that session to queue: copy route at 1B edges / 100M nodes (100 partitions of 10M edges) at `num_steps` 16 and 50, plus a by-mechanism breakdown of the 100M / 100M copy row (`searchsorted` gather, Bézier sampling, frame build, `cvs.line` aggregation). The lens wave does not wait. `## Inputs` carries a marked PENDING ROWS block and the constraint: until the rows land there, story 8 is `predicted` only and the synth may not rank A first on a predicted cell alone. If they land before synth, the dispatching session appends them and lifts the constraint in a dated note. Story 8, the `## Question` bounds line, the compute-floor brief, Proposed experiments and the summary shape point at the block. |
| 3 | should-fix | In-scope tweak | Accepted, split. "Query engines" (C, story 2, 9a) and "Node-table storage" (B, story 5, 9c). N = 7, two vouchers each, estimate 27 to 30. ClickHouse enters the query-engine lens only through ClickBench figures, since it is a server and not an embedded engine. Resident-table lookup latency stays with compute floor; anything past RAM is the storage lens's. |
| 4 | should-fix | In-scope tweak | Accepted. The compute-floor lens is the only reader of datashader 0.19.1's line glyph and dask line path, and its brief carries three sub-questions: binning (how segments between samples are rasterized and weighted), counter-evidence (throughput cliffs, antialiasing cost in the code), GPU (CUDA coverage of the line glyph and its reductions). Binning, counter-evidence and GPU now cite published material and their own domains only. |
| 5 | should-fix | In-scope tweak | Accepted in the criterion form; no extra vouchers. New validation criterion "Rank within evidence class first" and new rubric entry "Evidence-mark laundering". |
| 6 | should-fix | In-scope tweak | Accepted. Bernoulli edge sampling at fraction p is a third variant of D (story 3, seed list, questionnaire item 3), owned by the binning lens (bound, surviving reductions); the prior-art lens holds its design-judgment cell. The settled list now says "adaptive sampling" means curve points per edge, and edge sampling joins the net-new list. |
| 7 | should-fix | In-scope tweak | Accepted. 9b splits into 9b-i (the Python-stack floor at the target sits above "fast", at each threshold) and 9b-ii (a compiled core or switch lowers it by a named mechanism). Story 9, rubric "Premise confirmation", validation criteria, lens Moves lines and the summary shape follow; five verdicts. |
| 8 | should-fix | In-scope tweak | Accepted. `## Question` now states that "architecture" ranges over everything upstream of the raster, with only the renderer fixed by the 2026-09-30 ruling, and that 9d is judged in that scope. |
| 9 | should-fix | In-scope tweak | Accepted. The run proceeds under a revised, surfaced estimate (now 30); it halts only if a revision would pass 36 (the previous run's 31 plus headroom), then parks and the dispatching session asks the maintainer. Written under `## Lenses + bounds` "Overrun"; questionnaire item 6 restated. |
| 10 | note | In-scope tweak | Accepted. The prior page's "Unverified context" bullets and its 7b / 7c search scopes are pasted verbatim into one "Already found, unverified; start past these" block under `## Lenses + bounds`, which the prior-art, counter-evidence and GPU briefs carry. Rubric "Already settled" extended to prior searches. |
| 11 | note | In-scope tweak | Accepted. (a) The DuckDB sentence under `## Prior rulings and records` now names `dask-route-memory-research.md`'s lens exclusion as a scoping choice, not a ruling. (b) 30 GiB visible to WSL everywhere (questionnaire item 2, story 8), with the copy route at 300M nodes peaking at 25.70 GiB against it. (c) The confirmations (`hiveplot.py:1742`; the `num_steps=100` defaults) stand. |

Also recorded: model allocation under `## Lenses + bounds` (lenses, vouchers and synth on Opus by the maintainer's instruction; the adversary gates on Fable; this amend-plan ran on Opus 5.5).

Estimate after this amendment: 30 binding (4 spent, 26 to fire), ceiling 8, peak 7, park past 36.

### Amendment 2 (2026-11-02): estimate revised from 30 to 31, recorded after the fact (dispatching session)

The dispatching session surfaced a second revision in chat before the lens wave fired: the compute-floor lens took a third voucher because it carried three other lenses' source sub-questions, so the binding estimate became 31 (30 plus one voucher). The Overrun clause asks for the revision to be recorded here before the call fires; it was surfaced in chat on time and written here only after the convergence gate flagged the omission (gate must-fix 6). The run closed at 30 calls, under the revised estimate: 4 spent before the lens wave, 7 lenses, 16 vouchers (15 single, of which lens 2's claim A and lens 7's claim B were one claim verified once, plus 2 second votes), 1 synth, 1 convergence-gate adversary, 1 record-keeper landing. Peak concurrency 8 (the eight-voucher wave), at the ceiling. A second plan-stage adversary pass on Amendment 1 was knowingly skipped: every change in it was the adversary's own push taken as pushed or in its cheaper form, and the convergence gate read Amendment 1.

### Amendment 3 (2026-11-02, after the convergence gate): the queued rows landed, the story-8 constraint is lifted (dispatching session)

The maintainer approved the two 1B-edge copy rows after the gate had run, and the experiment session ran them with the breakdown probe behind its queue. All rows below are read from the raw jsonl by the dispatching session; no voucher or gate has seen them. Scope: ringtail, 8 threads, threaded scheduler, uniform random ids, `ds.count()`, edges generated in memory (no storage read), one fresh process per sample.

**Copy route, 100M nodes, 1B edges, 1,000 partitions of 1M edges, five samples each** (`results_2026_11_02/copy_billion_edges.jsonl`):

| num_steps | Table build (s) | Edge render (s), median (range) | Peak RSS VmHWM (GiB) | Anon peak (GiB) | Live peak (GiB) | Raster |
|---|---|---|---|---|---|---|
| 16 | 8.4 to 9.1 | 180.5 (178.1 to 181.8) | 8.75 to 9.05 | 7.32 to 7.93 | 6.28 to 7.03 | a4b979dbc67a, all five |
| 50 | 8.4 to 8.6 | 334.3 (334.1 to 337.5) | 8.87 to 9.38 | 7.70 to 8.07 | 7.03 to 7.77 | fbbb91d288cb, all five |

The 210 s prediction at 16 was 16% high; at 50 the render is 1.85x the 16 figure, not 3x, which matches the compute-floor lens's claim that the pixel walk does not scale with sample count. Memory at 50 did not move. Both pass the ten-minute batch threshold on edge render alone and both fail the one-minute interactive threshold; the storage read is still unmeasured.

**Per-mechanism breakdown, one 1M-edge partition against the 100M-node table, single thread, median of five after a warm-up** (`results_2026_11_02_fork/breakdown.jsonl`, `phase_breakdown_probe.py`):

| num_steps | Two searchsorted (s) | One search on sorted ids (s) | Eight gathers (s) | Bezier plus frame (s) | Whole build (s) | cvs.line (s) | spread (s) |
|---|---|---|---|---|---|---|---|
| 16 | 0.208 | 0.122 | 0.075 | 0.115 | 0.424 | 0.750 | 0.0015 |
| 50 | 0.216 | 0.122 | 0.078 | 0.279 | 0.575 | 1.507 | 0.0016 |

Datashader's line aggregation is the largest term: 64% of single-thread time at 16 and 72% at 50; lookup plus gathers are 24% at 16; sampling plus frame build 10%. Summed over 100M edges that is about 117 core-seconds single-threaded against the 168 core-seconds the 8-thread row spent, so threading loses about 30% to contention and the GIL-held sampler. Sorting the ids first halves the search cost (0.122 against 0.208). Lens 6's split (lookup 40 to 70, raster 50 to 110 of 168 core-s) is the right order; the measured balance tilts further toward the raster.

**Also landed from the experiment session the same day** (its rows, read from jsonl): `bucketed` at 1B nodes / 100M edges reached n=5: build 1492 to 1644 s (median 1584), edge render median 77.1 s (58.1 to 134.6), peak RSS 8.15 GiB, raster 61c822ac0338 in every sample. A new `memmap_chunked` route (the memmap table read in sorted chunks) and `bucketed` both finished on the LocalCluster configurations where memmap and copy had died, with no worker restarts and every process under 1.2 GiB RSS: at 100M nodes / 100M edges on 4 workers x 6 GiB, edge render medians 37.6 s (memmap_chunked) and 28.3 s (bucketed, after a 55 to 165 s build), against the p2p shuffle's 140.5 s; at 300M nodes, 75.4 s and 24.4 s against the shuffle's 549.4 s; on 8 workers x 3 GiB at 100M, 33.2 s and 22.9 s. Bucketed at 300M nodes on one machine under a 4 GiB cgroup cap: edge render median 21.6 s (build 237 to 298 s) against 20.6 s uncapped (build 192 to 239 s).

**Constraint lifted:** story 8 ("the target is already within reach") is `measured here` for the edge render at both num_steps values, and architecture A may be ranked first at the batch threshold on measured cells. The page carries these rows in a dated post-gate section, since the gate did not read them.

**Correction to Amendment 3 and to the gate's first sentence (2026-11-02, from the experiment session's handover):** `build_seconds` in the bucketed rows includes the bucket passes (`bucket_seconds`, median 587 s at 1B nodes); the node table build itself is about 850 to 1000 s, the same build the memmap route pays (851.5 s). So the honest comparison at 1B nodes / 100M edges is lookup work of about 11 minutes (bucket passes plus a 77 s render) against memmap's 2-hour render, about 11x, and 28 minutes end to end against 2 h 15 min, about 4.9x. "A minute behind a 25-minute build" overstated it. The 1B-edge prediction for bucketed is also low: bucket-pass cost tracks node buckets times edge partitions (about 2M appends per pass at 1,000 and 1,000 against 200k measured), so about 98 minutes of passes unless the bucket count is decoupled from node partitions. Also from the handover: the memmap workers on LocalCluster died because each thread's own np.memmap counts the touched table once per mapping (two mappings per 2-thread worker, 3.1 GiB each, against a 5.7 GiB kill line), and `memmap_chunked` fixed it with no other change (workers under 1.2 GiB, no restarts); that upgrades the page's "plausible" cause to confirmed by fix, though no RssFile-at-death sample was recorded. The page is corrected at its opening paragraph, B's cells, the stretch-row line, verdict 9c, N2 and the post-gate table.
