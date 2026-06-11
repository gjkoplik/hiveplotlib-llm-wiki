# Plan: performance regression + numeric-equivalence gate

<!--
Working plan, tracked in the wiki submodule. Concise per mental-model rule 17.
This is the keystone gate for scaling-large-networks.md; it must land first.
-->

## Goal

Give hiveplotlib an in-repo gate that makes every downstream scaling/perf change assertable as "measurably faster (or no worse) **and** output-equivalent to the proven path, with no regression on small graphs." Concretely: a parametric synthetic-graph generator, a deterministic numeric-equivalence check (curve arrays + rasterized output match the proven single-shot datashader path within tolerance), and a hybrid regression layer (2026-06-11 amendment): **pytest gates assert relative, same-run ratios** (e.g. streamed peak RSS as a fraction of single-shot peak RSS, sampled from a subprocess), while **ASV (airspeed velocity) owns timing/memory history**, the rolling baseline, and the capture-at-merge artifact store. Absolute wall-time numbers are never CI pass/fail criteria. This is the wall that protects against silent reward-hacking when `scaling-large-networks.md` Workstream 1 swaps single-shot curve concatenation for per-chunk streaming + raster aggregation (including the `ds.var`/`ds.std` streaming case). Nothing downstream can claim "faster, no regression" until this ships, so it is sequenced first.

## Prior ADRs / design docs

None — net-new design space. No `wiki/wiki/adr/` directory exists; ADR promotion has never been run. No existing perf/benchmark/regression-gating wiki page; this is net-new. Relevant concrete artifacts:

- `wiki/wiki/concepts/bezier-curves.md` — Bézier kernel, numba mode switch, and float32 curve storage; the equivalence gate compares this kernel's output across paths.
- `runners/performance/profile_hiveplot_ds.py` + `runners/performance/profiling_utils.py` — the existing in-repo profiling runner this harness extends. Establishes the "tooling in-repo, artifacts in `data/profiling/`" precedent.
- `wiki/wiki/sources/perez-2021-hype.md` — prior-tool scale comparator (weakly relevant; useful baseline framing only).
- `wiki/wiki/plans/scaling-large-networks.md` — the downstream super-plan this gate guards. Its Workstream 1 (streaming curve aggregation at `src/hiveplotlib/viz/datashader.py:181-234, 386-388`) is the first consumer of the equivalence check.

## Patterns this replaces

This **extends** `runners/performance/`; it does not supersede the existing runner. The existing single-shot datashader path stays as the regression **baseline**.

- `runners/performance/profiling_utils.py:95-100` (`time_call`) returns a single elapsed time. `time_call` stays for the existing runner's per-call use. Timing **history** is owned by ASV `time_` benchmarks (2026-06-11 amendment); the previously planned hand-rolled `time_median_of_n` helper is dropped, not replaced in-repo.
- No existing memory measurement exists in `runners/performance/`. Net-new addition (RSS-from-subprocess sampler).
- No existing equivalence check exists anywhere in `tests/` or `runners/`. Net-new addition.
- No `tests/`-level perf or equivalence assertions exist today. Net-new test module(s).

Nothing is deleted. QA Engineer's post-execution grep should find the existing `time_call` and the existing runner intact.

## Default justifications

- **Peak RSS sampled from a subprocess (psutil or `/usr/bin/time -v`), NOT `tracemalloc`**: `tracemalloc` only tracks Python-level allocations and under-counts numba- and numpy-level C allocations, which dominate at scale. A streaming refactor could "optimize" against a false tracemalloc number while real RSS holds flat or grows. Measuring peak RSS of a child process that runs the workload end-to-end is the only number that can't be gamed by moving allocations into C. This is a deliberate design decision, recorded here so it is not relitigated.
- **Repeated-sample timing with fixed seeds, via ASV's built-in repeat/statistics machinery** (supersedes the hand-rolled median-of-N helper, 2026-06-11): the original rationale stands (a single sample is dominated by JIT warmup, GC, and scheduling jitter; a robust statistic over N runs is needed), but ASV `time_` benchmarks already implement it, so no `time_median_of_n` ships.
- **CI gates are relative, same-run ratios in pytest, NOT absolute timings vs. a committed baseline** (2026-06-11): absolute wall-time baselines are unstable on shared GitLab CI runners; ratios measured within one run on one machine (e.g. streamed peak RSS / single-shot peak RSS, streamed vs. single-shot timing at small scale) are self-normalizing across machines and more reward-hack-resistant. Absolute numbers are reported by ASV for history, never asserted in CI.
- **ASV-as-CI-gate considered and rejected** (2026-06-11): `asv continuous` must build and benchmark two commits per MR — too heavy for per-MR gating. ASV owns history/reporting only; pytest owns the gate.
- **Fully-hand-rolled rolling-baseline store considered and rejected** (2026-06-11): reinvents ASV's commit-indexed history tracking. ASV's committed results JSON is the durable per-scenario store (plain JSON, plottable without ASV's HTML dashboards) and the static data source for the scaling plan's Workstream 6 blog figures. The store is ASV's **native results format committed as-is**; no bespoke schema (2026-06-11 follow-up).
- **Retroactive re-benchmarking as the foundation considered and rejected** (2026-06-11): ASV old-commit re-runs cannot replace capture-at-merge because of benchmark-code API drift across the chain and environment non-reproducibility (old commit rebuilt under re-run-time deps). Recovery-only; details in the 2026-06-11 follow-up amendment.
- **Equivalence tolerance: per-array `rtol`/`atol` chosen per the compared dtype, defaulting to float32-appropriate tolerances** (e.g. `rtol=1e-5, atol=1e-6` for float32 curve arrays; exact or near-exact integer-count match for rasters). The proven path stores curves as float32 (`src/hiveplotlib/hiveplot.py:1176-1207`), so a float64-tight tolerance would false-fail; a too-loose tolerance would let a real numeric bug through. Tolerances are scale-independent (a few thousand edges proves the algebra), so fixtures stay small.
- **Equivalence fixtures stay small (low thousands of edges), in-repo; sweep-scale artifacts stay out of git**: equivalence is scale-independent, so the gate proves correctness on small in-repo fixtures that run in CI in seconds. Large sweep CSVs/profiles/plots are gitignored under `data/` (already covered by `.gitignore:17`).

## Naming audit

This work adds developer/CI-facing test and harness entry points, not user-facing library API. Names are checked against pytest/benchmark-tooling vocabulary, not the hive-plot user vocabulary.

- New test functions: `test_equivalence_*`, `test_no_regression_*` (pytest convention; `test_<thing>_<scenario>` per CLAUDE.md test-name=body contract). OK.
- New harness helpers (in `runners/performance/`): `measure_peak_rss` (matches psutil/`time -v` "peak RSS" / "maximum resident set size" vocabulary), `assert_curves_equivalent` / `assert_raster_equivalent` (pytest `assert_*` convention). OK — these mirror `pytest-benchmark` and `numpy.testing.assert_allclose` naming users of perf tooling already know. (`time_median_of_n` dropped per the 2026-06-11 ASV amendment.) ASV benchmark functions follow ASV's own `time_*` / `peakmem_*` naming convention.
- New synthetic-generator wrapper: parametrized call into the existing `example_hive_plot`; no new public dataset name. The generator is `example_hive_plot` already; this plan only parametrizes its existing `num_nodes`/`num_edges`/`partition_data_column`/`sorting_variables`/`seed`/`repeat_axes` kwargs. No new name.
- Pytest marker for the slower regression assertions: reuse-or-add. If a perf marker doesn't exist, add `@pytest.mark.performance` (registered in `pyproject.toml` / `conftest.py`). Vocabulary: "performance" matches the directory name and CLAUDE.md's optional-marker convention. OK.

Internal module names are out of scope.

## API usage examples

No user-facing API change. This plan adds CI/dev tooling and test fixtures only; no library class, method, or function signature changes. Skipping the three API-usage subsections per the template.

## Notebook review

No notebook change. This plan touches `runners/performance/`, `tests/`, and config only.

## Workstreams

Sequenced so the equivalence wall (A) and the measurement primitives (B) land before the CI assertions (C) that depend on them. A and B are independent and may run concurrently; C depends on both. The whole gate must complete before `scaling-large-networks.md` Workstream 1 begins.

### Workstream A: deterministic numeric-equivalence checker

**Status:** not started
**Files:** `runners/performance/equivalence.py` (new), `tests/performance_equivalence_test.py` (new), small in-repo fixtures (generated at test time from `example_hive_plot`, no committed data files).
**Done when:**
- A helper compares two curve-construction outputs (the proven single-shot `HivePlot.construct_curves` path vs. an alternate path passed as a callable) and asserts elementwise closeness with dtype-aware tolerance. It compares **like-for-like dtypes**: `bezier_xy_with_nans` defaults to float64 (`src/hiveplotlib/utils.py:256`) but every internal call site passes float32 explicitly, so the checker coerces both operands to the proven path's stored dtype (float32, `src/hiveplotlib/hiveplot.py:1176-1207`) before comparing, and treats NaN-separator rows as positionally-matched NaNs (not as inequality).
- A helper rasterizes both paths through the datashader backend (`datashade_hive_plot_mpl`, `src/hiveplotlib/viz/datashader.py:445`) at a fixed canvas size and asserts the aggregated rasters match (exact or near-exact integer-count agreement for the line/point aggregations).
- Both helpers run against a small fixture (low thousands of edges) built from `example_hive_plot(num_nodes=..., num_edges=..., partition_data_column="low", labels=["A","B","C"], cutoffs=3, sorting_variables="low", seed=<fixed>, repeat_axes=True)`.
- A passing baseline test proves the checker is sound by comparing the single-shot path to itself (identity case must pass) and a deliberately-perturbed path (must fail), so the gate can't silently always-pass.
- `equivalence.py` exercised by tests (the 100% coverage gate applies to `src/hiveplotlib` only, per the 2026-06-11 coverage-scope ruling); warnings-as-errors clean; `make ty` clean.

### Workstream B: measurement primitive (subprocess peak RSS)

Shrunk by the 2026-06-11 ASV amendment: timing and memory **history** helpers are superseded by ASV's built-in `time_` / `peakmem_` benchmarks. This workstream now delivers only the peak-RSS helper the pytest ratio gates need.

**Status:** not started
**Files:** `runners/performance/profiling_utils.py` (extend), `tests/performance_measurement_test.py` (new).
**Done when:**
- A `measure_peak_rss(callable_or_argv) -> int` helper runs the workload in a **subprocess** and returns peak RSS in bytes, sampled via psutil (`Process.memory_info().rss` polled, or `psutil`'s peak where available) or `/usr/bin/time -v` "Maximum resident set size", with a documented fallback when neither is available. Docstring states explicitly why `tracemalloc` is rejected (under-counts numba/numpy C allocations) — this rationale stands and stays documented regardless of the ASV restructure.
- No `time_median_of_n` (or any other timing-history helper) is added; ASV owns timing/memory history (2026-06-11 amendment).
- Tests assert: peak RSS of a subprocess that allocates a known large array is meaningfully above a baseline subprocess (sanity bound, not an exact byte assertion).
- psutil/`time -v` availability is handled so the suite never errors on a missing dep (skip or fallback with a clear marker, per CLAUDE.md optional-backend convention if psutil is optional).
- New helper exercised by tests (100% coverage gate applies to `src/hiveplotlib` only, per the 2026-06-11 coverage-scope ruling); warnings-as-errors clean; `make ty` clean.

### Workstream C: CI-runnable regression assertions

Restructured by the 2026-06-11 ASV amendment: pytest owns the per-MR **gate** (relative, same-run ratios); ASV owns **history and reporting** (rolling baseline, per-scenario history, marginal-vs-cumulative deltas, capture-at-merge).

**Status:** not started
**Files:** `tests/performance_regression_test.py` (new), `pytest.ini` (register `@pytest.mark.performance`; deselect it from the default run), `Makefile` (serial performance target), CI config (serial performance stage), ASV config + benchmarks (home is an implementation detail, e.g. `benchmarks/` + `asv.conf.json`; outside the coverage surface), `runners/performance/README.md` (document the gate and how to run it locally vs. CI).
**Done when:**
- **Pytest ratio gates (relative, same-run, no absolute baselines).** Regression tests build small fixtures via the parametrized `example_hive_plot` and assert **same-run ratios**, e.g.: streamed-path peak RSS ≤ a stated fraction of single-shot peak RSS at the large scenario; small-graph streamed-vs-single-shot timing within a stated band. Document each chosen ratio/band and its rationale. Absolute timings and absolute RSS are NOT CI pass/fail criteria (they're unstable on shared GitLab CI runners; ratios are self-normalizing and reward-hack-resistant).
- The ratio gates use Workstream B's `measure_peak_rss`; the equivalence assertions from Workstream A gate any alternate path.
- **Serial performance stage (2026-06-11).** Tests are marked `@pytest.mark.performance`, **deselected from the normal `-n 7` parallel run**, and executed in a separate serial pytest invocation (Makefile target + CI stage; pytest.ini marker/deselection config). Rationale: 7 xdist workers competing for cores scrubs timing measurements. Small-graph no-regression is asserted explicitly in this stage.
- **Small-scale streaming-vs-single-shot guard.** The scaling plan's WS2 (chunked rasterization) and WS3 (fused build) share one streaming-vs-single-shot threshold policy; below the threshold, chunking is slower than single-shot. So the ratio gates must cover a **small-graph scenario explicitly** where the single-shot path is the one expected to be selected, catching an accidental "always stream" policy change as a small-scale regression. Make the small-graph scenario a first-class, named entry in the benchmark scenarios (not just an incidental fixture size) so it can't be dropped silently.
- **ASV history and reporting.** ASV `time_` and `peakmem_` benchmarks cover the representative scenario set at **small / medium / large** scales. ASV owns: the rolling baseline updated as each scaling workstream merges; retained per-scenario history; **per-workstream marginal delta and cumulative joint effect** versus the original pre-chain baseline; and the capture-at-merge artifact store from the 2026-06-05 amendment. ASV's committed results JSON is the durable per-scenario store and the static data source for the scaling plan's Workstream 6 blog figures (plain JSON, plottable without ASV's HTML dashboards). Results are committed **as-is in ASV's native format** (no bespoke schema), produced by capture-at-merge runs on the **maintainer's local machine** (canonical host, ASV machine-tagged); CI never writes to the ASV history (2026-06-11 follow-up). Done-when: benchmarks parametrized over the three scales; committed results capture each merge; marginal and cumulative deltas surfaceable from the stored history.
- **Aggregation-combination language stays consistent with the scaling plan (2026-06-11).** Wherever this harness describes combining per-chunk raster aggregates, it must not assume addition: `ds.any` combines via elementwise OR/max; combination happens on raw per-chunk aggregates, with `tf.spread` and the density-correction divide applied exactly once at the end. Full correction lives in `wiki/wiki/plans/scaling-large-networks.md`.
- `runners/performance/README.md` documents: the gate's purpose, the pytest-gate / ASV-history split (including why ASV-as-CI-gate and a hand-rolled baseline store were rejected), the RSS-subprocess-over-tracemalloc decision, the ratios-over-absolutes decision, the serial performance stage, the equivalence-tolerance rationale, and the in-repo-vs-deferred-sweep boundary.
- Full suite green under `make test` (`-n 7`, 100% coverage on `src/hiveplotlib`, warnings-as-errors) with performance tests deselected; the serial performance stage green; `make ty` clean; CHANGELOG entry added. ASV benchmark code and `runners/performance/` are exempt from the coverage gate (2026-06-11 ruling; `pytest.ini` already sets `--cov=src/hiveplotlib`, so no config change needed).

## Cross-workstream interaction support

Surfaced in maintainer discussion (2026-05-29). The per-MR gate is per-workstream by construction; two interaction patterns need explicit harness support, both folded into Workstream C's benchmark scenarios:

1. **Cumulative / rolling-baseline benchmark across the chain** (implemented via ASV history per the 2026-06-11 amendment). Re-run a representative benchmark set at small / medium / large scales against a rolling baseline updated as each scaling workstream merges, reporting both the per-workstream marginal delta and the cumulative joint effect versus the original pre-chain baseline. Catches benefits masked in isolation (WS2 memory win only fully realized after WS3) and two neutral changes interacting into a regression. Benchmark-scenario / reporting requirement, not a new equivalence concern.
2. **Small-scale no-regression for the shared streaming-vs-single-shot threshold.** WS2 (chunked rasterization) and WS3 (fused build) share one threshold policy; below it, chunking is slower. The regression band must assert no-regression at small scale specifically, where single-shot should stay selected, so an accidental "always stream" policy change is caught.

The full cross-workstream interaction catalog (all interaction pairs, not just the harness-side ones) lives in `wiki/wiki/plans/scaling-large-networks.md` under "Cross-workstream performance interactions" (added in parallel). This plan records only the two harness-side requirements above.

## Out of scope (stated explicitly)

- **Exploratory overnight sweep rig / autonomous-research while-loop.** Deferred; homed later (possibly a sibling repo or a third submodule), not here. The in-repo/separate line is "does CI need it / does it track the API": the gate tracks the API lockstep and runs in CI, so it is in-repo; the sweep rig does neither.
- **Large artifacts** (sweep CSVs, profiles, result plots). Stay out of git under `data/profiling/` (already gitignored via `.gitignore:17`).
- **Heavy GPU/cuDF/Dask benchmarking deps.** The gate runs on the standard test deps.
- **The Bézier "Bernstein hoist" kernel tweak and numba autotuning.** Handled outside both plans (hoist = small standalone PR; autotune = deferred, already mined by the maintainer).
- **float32 storage itself.** Already shipped (`src/hiveplotlib/hiveplot.py:1176-1207`); the harness treats it as the given baseline, not a change to gate.

## Holdouts

- `runners/performance/profiling_utils.py:95-100` (`time_call`): kept as-is; the existing runner uses it per-call. The only addition to this module is `measure_peak_rss` (2026-06-11 amendment).

## Implementation log

Append-only. One line per completed workstream.

## Plan amendments

Append-only. Concise per mental-model rule 17.

### 2026-05-30 — Two cross-workstream interaction requirements (from 2026-05-29 maintainer discussion)

- **In-scope tweak (Workstream C): small-scale streaming-vs-single-shot guard.** Added a done-when so the regression band covers a named small-graph scenario where single-shot should stay selected, catching an accidental "always stream" change in the threshold policy WS2 and WS3 of the scaling plan share.
- **In-scope tweak (Workstream C): rolling-baseline cumulative benchmark.** Added a done-when so benchmark scenarios run at small/medium/large scales against a rolling baseline updated as each scaling workstream merges, reporting per-workstream marginal delta and cumulative joint effect. Reuses Workstream A's equivalence check unchanged; no new equivalence concern.
- Added a "Cross-workstream interaction support" section summarizing both, with a pointer to the full interaction catalog in `wiki/wiki/plans/scaling-large-networks.md` ("Cross-workstream performance interactions", added in parallel). No new workstream, no entry-point or attribute-read change (no feasibility audit needed).

### 2026-06-05 — Artifact-retention requirement (capture-at-merge for perishable wins)

- **In-scope tweak (Workstream C): durable per-scenario artifact capture at each merge.** New done-when: the harness durably stores (committed, e.g. extending the in-repo baseline JSON from the existing done-when at line 89) the **per-scenario** results (small / medium / large; peak RSS **and** timing) at each merge, rich enough to regenerate publication-quality plots for the Workstream-6 blog later, not merely a pass/fail regression verdict. This extends the existing rolling-baseline store (done-when at line 93 / "Cross-workstream interaction support" §1) from "rolling value sufficient for the no-regression gate" to "retained per-scenario history sufficient to re-plot the chain's progression."
- **Rationale to record (load-bearing, not insurance): not all wins are reconstructable at the end.** *Retained-alternative* changes keep both paths in the code (single-shot datashader and two-stage `construct_curves` both stay as defaults / equivalence baselines per the scaling plan), so their before/after is always re-runnable on demand. *Replace-in-place* changes delete the old code on merge, e.g. scaling-plan WS1 (full-length boolean-mask membership storage replaced by integer indices) and future Bezier-call reductions, so their "before" is perishable and survives **only** as what the harness captured at merge. The capture is therefore the sole evidence for the blog's replace-in-place speedup/memory claims, making it load-bearing for Workstream 6, not a nice-to-have.
- **Confirm at the harness's own pre-implementation alignment (grill) pass.** Flag this requirement explicitly to be confirmed / agreed with the maintainer at the harness's grill pass, **before** downstream workstreams build, so the capture design (schema, what fields, where committed) is validated before the chain makes replace-in-place "before" states impossible to re-benchmark. This is the next real checkpoint; nothing dispatches until the harness is built and grilled.
- No new entry point, no new attribute read (no feasibility audit needed); this is a reporting / artifact-store requirement on Workstream C, reusing Workstream A's equivalence check unchanged.

### 2026-06-11 — ASV hybrid restructure (maintainer review session, settled decision)

- **Trigger:** 2026-06-11 maintainer review; final call, not a proposal. Plan stays gated behind in-flight work; amendments capture decisions while fresh.
- **In-scope tweak (Workstream C): pytest gates become relative, same-run ratio assertions.** No absolute timings vs. a committed baseline JSON: e.g. streamed-path peak RSS ≤ a stated fraction of single-shot peak RSS at the large scenario; small-graph streamed-vs-single-shot timing within a stated band. Absolute wall-time baselines are unstable on shared GitLab CI runners; ratios are self-normalizing and more reward-hack-resistant. Absolute numbers are never CI pass/fail criteria.
- **In-scope tweak (Workstream C): ASV (airspeed velocity) owns history/reporting.** Rolling baseline, retained per-scenario history (small/medium/large), marginal-vs-cumulative delta reporting across the workstream chain, and the capture-at-merge artifact store from the 2026-06-05 amendment. ASV `time_` benchmarks replace the planned `time_median_of_n` for history; `peakmem_` benchmarks cover memory history. ASV's committed results JSON is the durable per-scenario store and the static data source for the scaling plan's Workstream 6 blog figures (plain JSON, plottable without ASV's HTML dashboards). Config/benchmarks home (e.g. `benchmarks/` + `asv.conf.json`) is an implementation detail, outside the coverage surface.
- **In-scope tweak (Workstream B): shrinks to one psutil-based subprocess peak-RSS helper** used by the pytest ratio gates. The tracemalloc-rejection rationale stands and stays documented. The rest of B's planned helpers are superseded by ASV built-ins.
- **Workstream A unchanged:** equivalence gate stays pytest; correctness checking of curve arrays/rasters is not ASV's job.
- **Recorded in Default justifications:** ASV-as-CI-gate rejected (`asv continuous` builds and benchmarks two commits per MR; too heavy for per-MR gating); fully-hand-rolled rolling-baseline store rejected (reinvents ASV's history tracking).
- **Perishability relaxation (softens, does not remove, the 2026-06-05 rationale):** ASV can check out and benchmark historical commits, so replace-in-place "before" states (e.g. scaling WS1's storage change) remain re-measurable after merge, provided benchmark code (which lives outside the package tree) guards against API drift across commits. Capture-at-merge via committed ASV results stays the primary record.
- Dev/CI tooling only; no new library entry point or attribute read (no feasibility audit needed).

### 2026-06-11 — Serial performance stage (maintainer review session, settled decision)

- **In-scope tweak (Workstream C):** `@pytest.mark.performance` tests are deselected from the normal `-n 7` parallel run and executed in a separate serial pytest invocation (Makefile target + CI stage + pytest.ini marker/deselection config, named in Workstream C Files). Rationale: 7 xdist workers competing for cores scrubs timing measurements.

### 2026-06-11 — Coverage scope ruling (maintainer decision)

- **In-scope tweak (Workstreams A and B):** the 100% coverage gate applies to `src/hiveplotlib` only (the code users hit in the deployed package). `runners/performance/`, ASV benchmark code, and harness tooling are exempt; the "100% coverage on `equivalence.py` / new helpers" done-when phrasing is relaxed to "exercised by tests." No config change needed: `pytest.ini` already sets `--cov=src/hiveplotlib`.
- **Pragma policy (recorded for downstream scaling workstreams):** GPU-gated and RAM-gated tests skip in CI and would leave their `src/` branches uncovered, so such branches (e.g. cuDF dispatch, 10M-scale paths) carry explicit `# pragma: no cover` (or equivalent) with a comment naming the gating test, so downstream workstreams don't each rediscover the coverage failure.

### 2026-06-11 — Aggregation-combination correction (cross-plan consistency)

- **In-scope tweak (Workstream C language only):** wherever this plan refers to combining per-chunk raster aggregates, align with the scaling plan's parallel amendment: `ds.any` combines via elementwise OR/max (NOT addition); combination happens on raw per-chunk aggregates, with `tf.spread` and the density-correction divide applied exactly once at the end. Audit of this plan's body found no language asserting additive combination; a guard bullet was added to Workstream C so future equivalence/regression wording can't contradict. Full correction lives in `wiki/wiki/plans/scaling-large-networks.md` (amended in parallel; not edited here).

### 2026-06-11 — Capture schema, retroactive runs, benchmark host (maintainer review session, settled decisions)

- **Trigger:** continuation of the same 2026-06-11 maintainer review; three final dispositions, settled. Plan stays gated and not executing.
- **In-scope tweak (Workstream C): capture schema = ASV's native results format; no bespoke schema design.** The capture-at-merge store is whatever `asv run` emits, committed as-is. The 2026-06-05 grill checkpoint ("schema, what fields, where committed") therefore shrinks to a five-minute confirmation: benchmark scenario naming covers small/medium/large scales with both `time_` and `peakmem_` variants, sufficient to regenerate the Workstream-6 blog figures from the committed JSON alone.
- **In-scope tweak (recorded rationale): retroactive re-benchmarking is recovery-only, never the foundation.** Maintainer asked whether capture could be dropped in favor of ASV's old-commit re-runs; no, for two reasons: (a) **benchmark-code API drift** — the chain deliberately changes the surface benchmarks touch (`stream_chunk_threshold` doesn't exist pre-WS2; the fused path doesn't exist pre-WS3), so old-commit runs need per-benchmark guards, each a chance to silently measure the wrong thing; (b) **environment non-reproducibility** — ASV rebuilds hiveplotlib at the old commit under re-run-time deps, so retroactive numbers are not the historical numbers, and an old commit may not run at all under future datashader/numba pins. Rule when retroactive runs ARE used: both ends of a comparison run in the same ASV session (same machine, same deps). Capture-at-merge remains the primary record, and is nearly free since committing ASV results is already the rolling-baseline mechanism. Refines (tightens) the prior entry's "perishability relaxation" bullet.
- **In-scope tweak (Workstream C): canonical benchmark host is the maintainer's local machine (WSL box).** Capture-at-merge ASV runs happen locally at each workstream merge and the results JSON is committed; CI never writes to the ASV history. CI's only performance role is the relative same-run pytest ratio gates. ASV's machine-tagging makes this first-class. Caveat: a mid-chain machine swap forks the history and muddies cumulative-vs-original comparisons, so aim to keep one host for the chain's duration.
- Dev/CI tooling only; no new library entry point or attribute read (no feasibility audit needed).
</content>
</invoke>
