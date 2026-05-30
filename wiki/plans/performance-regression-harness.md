# Plan: performance regression + numeric-equivalence gate

<!--
Working plan, tracked in the wiki submodule. Concise per mental-model rule 17.
This is the keystone gate for scaling-large-networks.md; it must land first.
-->

## Goal

Give hiveplotlib an in-repo gate that makes every downstream scaling/perf change assertable as "measurably faster (or no worse) **and** output-equivalent to the proven path, with no regression on small graphs." Concretely: a parametric synthetic-graph generator, a deterministic numeric-equivalence check (curve arrays + rasterized output match the proven single-shot datashader path within tolerance), and CI-runnable regression assertions (peak RSS sampled from a subprocess; median-of-N timing). This is the wall that protects against silent reward-hacking when `scaling-large-networks.md` Workstream 1 swaps single-shot curve concatenation for per-chunk streaming + raster aggregation (including the `ds.var`/`ds.std` streaming case). Nothing downstream can claim "faster, no regression" until this ships, so it is sequenced first.

## Prior ADRs / design docs

None — net-new design space. No `wiki/wiki/adr/` directory exists; ADR promotion has never been run. No existing perf/benchmark/regression-gating wiki page; this is net-new. Relevant concrete artifacts:

- `wiki/wiki/concepts/bezier-curves.md` — Bézier kernel, numba mode switch, and float32 curve storage; the equivalence gate compares this kernel's output across paths.
- `runners/performance/profile_hiveplot_ds.py` + `runners/performance/profiling_utils.py` — the existing in-repo profiling runner this harness extends. Establishes the "tooling in-repo, artifacts in `data/profiling/`" precedent.
- `wiki/wiki/sources/perez-2021-hype.md` — prior-tool scale comparator (weakly relevant; useful baseline framing only).
- `wiki/wiki/plans/scaling-large-networks.md` — the downstream super-plan this gate guards. Its Workstream 1 (streaming curve aggregation at `src/hiveplotlib/viz/datashader.py:181-234, 386-388`) is the first consumer of the equivalence check.

## Patterns this replaces

This **extends** `runners/performance/`; it does not supersede the existing runner. The existing single-shot datashader path stays as the regression **baseline**.

- `runners/performance/profiling_utils.py:95-100` (`time_call`) returns a single elapsed time. Extend with a median-of-N timing helper (new function alongside `time_call`, not a replacement; `time_call` stays for the existing runner's per-call use). Cite as extension, not removal.
- No existing memory measurement exists in `runners/performance/`. Net-new addition (RSS-from-subprocess sampler).
- No existing equivalence check exists anywhere in `tests/` or `runners/`. Net-new addition.
- No `tests/`-level perf or equivalence assertions exist today. Net-new test module(s).

Nothing is deleted. QA Engineer's post-execution grep should find the existing `time_call` and the existing runner intact.

## Default justifications

- **Peak RSS sampled from a subprocess (psutil or `/usr/bin/time -v`), NOT `tracemalloc`**: `tracemalloc` only tracks Python-level allocations and under-counts numba- and numpy-level C allocations, which dominate at scale. A streaming refactor could "optimize" against a false tracemalloc number while real RSS holds flat or grows. Measuring peak RSS of a child process that runs the workload end-to-end is the only number that can't be gamed by moving allocations into C. This is a deliberate design decision, recorded here so it is not relitigated.
- **Median-of-N timing (default N=5), fixed seeds**: a single timing sample is dominated by JIT warmup, GC, and OS scheduling jitter. Median-of-N with a fixed seed keeps the metric low-variance and reproducible across machines and CI runs; median (not mean) is robust to the occasional cold-cache outlier. N=5 matches the existing runner's `--runs` default so the two share intuition.
- **Equivalence tolerance: per-array `rtol`/`atol` chosen per the compared dtype, defaulting to float32-appropriate tolerances** (e.g. `rtol=1e-5, atol=1e-6` for float32 curve arrays; exact or near-exact integer-count match for rasters). The proven path stores curves as float32 (`src/hiveplotlib/hiveplot.py:1176-1207`), so a float64-tight tolerance would false-fail; a too-loose tolerance would let a real numeric bug through. Tolerances are scale-independent (a few thousand edges proves the algebra), so fixtures stay small.
- **Equivalence fixtures stay small (low thousands of edges), in-repo; sweep-scale artifacts stay out of git**: equivalence is scale-independent, so the gate proves correctness on small in-repo fixtures that run in CI in seconds. Large sweep CSVs/profiles/plots are gitignored under `data/` (already covered by `.gitignore:17`).

## Naming audit

This work adds developer/CI-facing test and harness entry points, not user-facing library API. Names are checked against pytest/benchmark-tooling vocabulary, not the hive-plot user vocabulary.

- New test functions: `test_equivalence_*`, `test_no_regression_*` (pytest convention; `test_<thing>_<scenario>` per CLAUDE.md test-name=body contract). OK.
- New harness helpers (in `runners/performance/`): `measure_peak_rss` (matches psutil/`time -v` "peak RSS" / "maximum resident set size" vocabulary), `time_median_of_n` (explicit about the statistic), `assert_curves_equivalent` / `assert_raster_equivalent` (pytest `assert_*` convention). OK — these mirror `pytest-benchmark` and `numpy.testing.assert_allclose` naming users of perf tooling already know.
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
- 100% coverage on `equivalence.py`; warnings-as-errors clean; `make ty` clean.

### Workstream B: measurement primitives (peak RSS + median-of-N timing)

**Status:** not started
**Files:** `runners/performance/profiling_utils.py` (extend), `tests/performance_measurement_test.py` (new).
**Done when:**
- A `measure_peak_rss(callable_or_argv) -> int` helper runs the workload in a **subprocess** and returns peak RSS in bytes, sampled via psutil (`Process.memory_info().rss` polled, or `psutil`'s peak where available) or `/usr/bin/time -v` "Maximum resident set size", with a documented fallback when neither is available. Docstring states explicitly why `tracemalloc` is rejected (under-counts numba/numpy C allocations).
- A `time_median_of_n(fn, n=5, seed=...) -> float` helper returns the median wall-clock of N runs with a fixed seed, leaving the existing `time_call` untouched.
- Tests assert: median-of-N returns the median (not mean) on a controlled sequence; peak RSS of a subprocess that allocates a known large array is meaningfully above a baseline subprocess (sanity bound, not an exact byte assertion).
- psutil/`time -v` availability is handled so the suite never errors on a missing dep (skip or fallback with a clear marker, per CLAUDE.md optional-backend convention if psutil is optional).
- 100% coverage on the new helpers; warnings-as-errors clean; `make ty` clean.

### Workstream C: CI-runnable regression assertions

**Status:** not started
**Files:** `tests/performance_regression_test.py` (new), `pyproject.toml` / `tests/conftest.py` (register `@pytest.mark.performance` if not already present), `runners/performance/README.md` (document the gate and how to run it locally vs. CI).
**Done when:**
- A regression test builds a small fixture via the parametrized `example_hive_plot` and asserts the current single-shot datashader path's median-of-N timing and peak RSS stay within a tolerance band of a recorded baseline (baseline stored in-repo as a small committed constant/JSON, NOT a swept CSV). Tolerance is generous enough to survive CI-machine variance, tight enough to catch a real regression (document the chosen band and its rationale).
- The regression assertions use Workstream B's `measure_peak_rss` and `time_median_of_n`, and the equivalence assertions from Workstream A gate any alternate path.
- Tests are marked `@pytest.mark.performance` and run in CI; small-graph regression (the "no regression on small graphs" requirement) is asserted explicitly.
- `runners/performance/README.md` documents: the gate's purpose, the RSS-subprocess-over-tracemalloc decision, the median-of-N choice, the equivalence-tolerance rationale, and the in-repo-vs-deferred-sweep boundary.
- Full suite green under `make test` (`-n 7`, 100% coverage, warnings-as-errors); `make ty` clean; CHANGELOG entry added.

## Out of scope (stated explicitly)

- **Exploratory overnight sweep rig / autonomous-research while-loop.** Deferred; homed later (possibly a sibling repo or a third submodule), not here. The in-repo/separate line is "does CI need it / does it track the API": the gate tracks the API lockstep and runs in CI, so it is in-repo; the sweep rig does neither.
- **Large artifacts** (sweep CSVs, profiles, result plots). Stay out of git under `data/profiling/` (already gitignored via `.gitignore:17`).
- **Heavy GPU/cuDF/Dask benchmarking deps.** The gate runs on the standard test deps.
- **The Bézier "Bernstein hoist" kernel tweak and numba autotuning.** Handled outside both plans (hoist = small standalone PR; autotune = deferred, already mined by the maintainer).
- **float32 storage itself.** Already shipped (`src/hiveplotlib/hiveplot.py:1176-1207`); the harness treats it as the given baseline, not a change to gate.

## Holdouts

- `runners/performance/profiling_utils.py:95-100` (`time_call`): kept as-is; the existing runner uses it per-call. The new `time_median_of_n` is additive.

## Implementation log

Append-only. One line per completed workstream.
</content>
</invoke>
