---
title: "ADR 0002: Performance regression + equivalence harness (pytest ratio gates + ASV history)"
type: adr
created: 2026-07-03
updated: 2026-07-03
status: Accepted
sources: [hiveplotlib-python-repo]
tags: [adr, hiveplotlib, performance, benchmarking, regression-testing, ci, datashader]
---

# ADR 0002: Performance regression + equivalence harness

**Status:** Accepted (shipped complete on GitLab #52, 2026-07-03; dev/CI infrastructure only — no user-facing API change, not tied to a release).

## Context

The scaling-large-networks super-plan (`wiki/wiki/plans/scaling-large-networks.md`, active) will replace proven code paths with streaming ones: per-chunk curve construction and raster aggregation in the datashader backend, a fused build path that discards per-chunk geometry, a membership-storage rewrite. Every one of those workstreams wants to claim "measurably faster (or no worse), and output-equivalent to the proven path, with no regression on small graphs." Before this harness, nothing in the repo could adjudicate that claim: `runners/performance/` held a profiling runner but no equivalence check, no memory measurement, and no CI-runnable regression assertion. Without a wall, a streaming refactor could silently reward-hack — optimize a false metric, drift numerically, or regress the small-graph case nobody re-measures.

This harness is that wall, sequenced ahead of the entire scaling chain. Nothing downstream ships a performance claim except through it.

## Decision

**Hybrid gating: pytest owns the per-MR gate, ASV owns history.** CI pass/fail lives in pytest assertions; ASV (airspeed velocity) owns timing/memory history, the rolling baseline across the scaling chain, and the capture-at-merge artifact store. ASV's results are committed **as-is in its native JSON format** at `benchmarks/results/` — no bespoke schema — and double as the static data source for the eventual scaling blog figures.

**CI asserts relative, same-run ratios only; absolute numbers never gate.** Gates compare two measurements taken in the same run on the same machine (e.g. streamed peak RSS as a fraction of single-shot peak RSS; streamed-vs-single-shot timing bands). Ratios are self-normalizing across machines, so the gate survives CI-runner migration by construction, and they are harder to reward-hack than a committed wall-time number. Absolute timings and absolute RSS are ASV-history material only.

**Peak RSS is kernel truth, measured two-tier.** Tier 1 (`measure_peak_rss`): the workload runs in a spawned child that measures and reports **its own** stdlib `getrusage(RUSAGE_SELF).ru_maxrss` — the exact kernel high-water mark — launched via a **two-level subprocess** (a near-empty launcher spawns the measured interpreter, resetting the Linux `ru_maxrss` floor a heavy parent would impose, and dodging the `RUSAGE_CHILDREN` cumulative-max trap that poisons sequential measurements). `tracemalloc` is rejected: it tracks only Python-level allocations and under-counts the numba/numpy C allocations that dominate at scale, so a refactor could optimize a false number while real RSS holds or grows. Tier 1 fully covers threaded (numba) work: threads share one address space, so all parallel allocation lands in one process's high-water mark. Tier 2 (`measure_peak_rss_tree`): a psutil-sampled process-tree sum (child + descendants, max over samples) — sampled-not-exact and a conservative upper bound (shared pages overcounted). Tier 2 is **provisional**: a candidate instrument for the scaling plan's Dask workstream, not the chosen one, with a hard removal-or-retention decision required at that plan's chain close. GPU VRAM is the one uncovered regime; it lands with the scaling plan's GPU workstream (per-process cupy mempool peak is the chosen metric).

**A numeric-equivalence wall, sound by construction.** Any alternate path is compared to the proven single-shot path at two levels: curve arrays (elementwise, with dtype-aware tolerance — both operands coerced to the proven path's stored float32, NaN separator rows matched positionally) and datashader rasters (near-exact count agreement, with a validated `allowed_mismatch_fraction` knob for downstream dialing). Soundness is proven, not assumed: identity comparisons must pass, deliberately perturbed paths must fail, and comparing nothing (empty-vs-empty mappings, zero-size rasters, no finite values) raises rather than passing vacuously. Tolerances are scale-independent, so fixtures stay small (low thousands of edges, generated at test time, nothing committed).

**Four pinned benchmark scenarios; tiny is a time-only floor.** tiny ≈ 34 nodes / 78 edges (karate-club scale), small ≈ 2,000 / 5,000, medium ≈ 25,000 / 250,000, large ≈ 100,000 / 2,000,000; 10M+ is a manual stress run, never a standing scenario. Tiny exists to catch fixed-overhead regressions (an accidental "always stream" bites hardest at karate-club scale, where small=2,000 would miss it) and carries `time_` benchmarks only — peak RSS there is interpreter/import-dominated noise. The scenario sizes are first-class named constants, with a consistency test keeping the pytest fixtures, workloads, and ASV benchmarks on the same seeds and sizes.

**Dormant gates arm mechanically, not by prose.** The streamed-vs-single-shot ratio gates ship skip-marked ahead of the streaming code they will judge. A canary test in the per-MR perf job fails the moment `stream_chunk_threshold` appears in the datashader entry point's signature, instructing the implementer to unskip the gates and delete the canary — skip reasons are prose, not enforcement, so the arming trigger is a signature check. The dormant timing gate warms both paths and takes best-of-N, so neither side of a same-run ratio pays first-call import/JIT cost.

**A `perf_harness` marker lane preserves the CI-semantics invariant.** The invariant: a red Python-matrix or backend-addon job means shipped library code is broken and implies a bugfix release; the perf lane red means tooling, no release. Harness-validation tests (equivalence soundness, instrument validation, constants consistency, the arming canary) carry `perf_harness`; regression gates carry `performance`; both are deselected from the default `-n 7` run and executed by a **serial** per-MR `performance_gates` job (`-m "performance or perf_harness"`) — xdist workers competing for cores scrub timing measurements.

**The perf lane installs loudly or fails loudly.** The `perf` extra self-referentially requires `hiveplotlib[datashader]` (plus asv and psutil), so the two extras cannot drift; the lane runs on `.[perf,testing]` and carries **no dependency skipifs** — a missing dependency there is a half-completed install and must ImportError, not skip. Graceful skips remain the default-suite convention for optional backends only.

**History is captured at merge, on the designated machine.** ASV history is written only by maintainer-local runs on the canonical host (ASV machine tag `ringtail`), at each scaling-workstream merge **and at dependency-bump merges** (datashader/numba/numpy pins) — the pytest gates are structurally drift-immune (same-run, same-environment), so version bumps manifest only as ASV history steps, and capturing at the bump keeps those steps attributable. CI never writes history. `make benchmark-capture` (branch + clean-tree guarded) is the canonical entry point. Retroactive re-benchmarking is **recovery-only**, never the foundation: benchmark-code API drift across the chain and environment non-reproducibility make old-commit re-runs unreliable; when used, both ends of a comparison run in one ASV session.

## Consequences

- **Enables the scaling chain.** Every scaling workstream's "faster, no worse, equivalent" claim now has a mechanical adjudicator; the equivalence wall plus its soundness tests are the anti-reward-hack guard the super-plan's grill named as the prerequisite for supervised optimization work.
- **Capture-at-merge is load-bearing evidence.** Replace-in-place changes (e.g. the membership-storage rewrite) delete their "before" on merge; the committed ASV JSON is the only surviving record for the chain's before/after story, including the blog figures.
- **One benchmark host is locked in for the chain's duration.** A mid-chain machine swap forks the ASV history and muddies cumulative-vs-original comparisons; ratios in CI are unaffected by construction.
- **The release signal stays clean.** The marker split means the matrix/addon jobs test shipped code only; harness breakage cannot masquerade as a library bug (or vice versa).
- **An obligation rides along:** the provisional tier-2 helper must get an explicit recorded removal-or-retention decision before the scaling super-plan closes (recorded in that plan's chain-close checkpoint). Retention requires justification (its unique property: an external observer that framework self-reporting cannot game); unused means removed.
- **Coverage boundary settled:** the 100% gate applies to `src/hiveplotlib` only; `runners/performance/` and `benchmarks/` are "exercised by tests," not coverage-gated.

## Alternatives considered

- **ASV as the CI gate** (`asv continuous`) — rejected: builds and benchmarks two commits per MR, too heavy for per-MR gating. ASV keeps history/reporting only.
- **Hand-rolled rolling-baseline store** — rejected: reinvents ASV's commit-indexed history; ASV's native committed JSON is the store.
- **`tracemalloc` for memory** — rejected: Python-level only, blind to the C allocations that dominate, gameable.
- **Parent-measured `getrusage(RUSAGE_CHILDREN)`** — rejected in implementation: cumulative max across all reaped children poisons sequential measurements, and a heavy parent's inherited `ru_maxrss` floor masks small workloads; hence child-reported `RUSAGE_SELF` behind a two-level subprocess.
- **Retroactive re-benchmarking as the foundation** — rejected: benchmark-code API drift (pre-WS2 commits lack the surfaces benchmarks touch) and environment non-reproducibility (old commit rebuilt under new deps). Recovery-only.
- **Absolute timing/RSS baselines committed to the repo** — rejected: unstable on shared CI runners; superseded by same-run ratios.

## References

- Working plan (verbose history, grill waves, adversary review, full amendments trail): `wiki/wiki/plans/archived/performance-regression-harness.md`.
- Consumer: `wiki/wiki/plans/scaling-large-networks.md` (active) — every workstream there is gated by this harness; its chain-close checkpoint owns the tier-2 pruning decision and the eventual umbrella scaling ADR.
- In-repo rationale record: `runners/performance/README.md` (the gate's purpose, rejected alternatives, scenario table, capture-at-merge commands). Code: `runners/performance/equivalence.py`, `runners/performance/profiling_utils.py`, `tests/performance_*`, `benchmarks/` + `asv.conf.json`.
- Wiki: [[hiveplotlib]] (entity hub), [[bezier-curves]] (the kernel whose output the equivalence wall compares).
- Issue/branch: GitLab #52 (`52-build-the-performance-regression-equivalence-harness`).

## See Also

- [[hiveplotlib]]
- [[bezier-curves]]
- [[0001-networkx-integration|ADR 0001 — NetworkX integration]]
