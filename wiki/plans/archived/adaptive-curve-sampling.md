# Plan: adaptive per-curve Bézier sampling

**Status: MEASURED, IDEA REJECTED.** The gate below was run on 2026-08-01 and adaptive sampling
failed it. The measurement did surface a separate and much larger finding (the default `num_steps` is
~6x oversampled), which is written up at the bottom and is the only part worth carrying forward. This
document is commentary, not a ready-for-execution plan; nothing here has been through a grill or an
adversary pass. Do not dispatch from it.

## Where this came from

Evaluating `reflex-dev/xy` (Reflex's Rust-core charting library, Aug 2026) against the
scaling-large-networks work. xy's representation ladder uses **M4 decimation** for long ordered
lines: bucket by pixel column, keep first/last/min/max per bucket, render a pixel-identical line
from ~4x width points regardless of row count.

M4 itself does not transfer. It assumes an x-ordered series, i.e. that the data is a function of x.
Hive plot edges are Bézier curves that loop around the origin at arbitrary angles and are
emphatically not x-monotone. There is no pixel-column bucketing to do.

The transferable part was the **principle**: the number of points rendered should be bounded by what
the screen can resolve, not by the size of the input. We already apply that to *scatter* density
through datashader. We did not apply it to *curve discretization* at all.

## The current behavior

`hiveplotlib.utils.bezier` / `bezier_all` evaluate every curve at `np.linspace(0, 1, num_steps)` with
`num_steps=100`, one control point (quadratic). The flat output relies on a fixed stride of
`num_steps + 1` rows per curve (the `+ 1` is the NaN separator); the numba kernels index it as
`base = i * (num_steps + 1)`.

## The flatness bound

For a quadratic Bézier the second derivative is constant:

```
B''(t) = 2·(P₀ - 2·P₁ + P₂)
```

Max deviation between the true curve and a uniform k-segment polyline is `|B''| / (8k²) = D / (4k²)`,
where `D = ‖P₀ - 2·P₁ + P₂‖`. Solving for a pixel tolerance:

```
k ≥ sqrt( D / (4·tol) )        num_steps = k + 1
```

One vector norm per curve, fully vectorizable, no subdivision recursion. This is the right
formulation (not arc length): it goes small when a curve is short *or* nearly straight, so long
gently-curved edges correctly get few samples.

## The gate, and its result

The question was framed as a **distribution** question: adaptive sampling only earns its complexity
if the spread across curves is wide enough that no single uniform constant serves both ends.

Measured 2026-08-01 at `tol = 0.5 px`, via a spy on `bezier_xy_with_nans` capturing per-curve
(P₀, P₁, P₂) across the ASV fixtures and two scale-free graphs. Metric is
`sum(required_n) / (n_curves × max(required_n))`, i.e. adaptive cost over the best possible uniform
constant *at identical quality*.

| scenario | 800px | 1500px | 4000px |
|---|---|---|---|
| synthetic ASV tiny (34 / 78) | 0.786 | 0.775 | 0.745 |
| synthetic ASV small (2k / 5k) | 0.719 | 0.703 | 0.670 |
| synthetic ASV medium (25k / 250k) | 0.718 | 0.702 | 0.669 |
| Barabási-Albert 2k, degree-sorted | 0.549 | 0.535 | 0.497 |
| Barabási-Albert 25k, degree-sorted | 0.486 | 0.471 | 0.427 |

**Verdict: rejected.** 1.3-1.4x on the synthetic fixtures, 1.9-2.3x on realistic scale-free data.
That is not enough to justify ragged offset arrays, a numba kernel refactor, canvas-awareness
plumbing into the geometry core, and reworking the equivalence gate.

Worth recording why the scale-free rows are better: `Axis` places nodes by scaling values linearly
between `vmin`/`vmax` (**by value, not by rank**), so a heavy-tailed sorting variable like degree
bunches most nodes near one end of each axis and genuinely widens the spread. That is the realistic
best case for adaptivity, and it still only reaches ~2x.

Power-of-two bucketing (the proposed cheap middle path) is **counterproductive** at these n: ratios
of 0.89-1.15 vs the unbucketed uniform baseline, sometimes worse than no bucketing. Required counts
are small enough that rounding up to the next power of two wastes more than it saves.

## The actual finding: the default `num_steps` is ~6x oversampled

The same measurement showed today's `num_steps=100` guarantees a max deviation of **0.011 px** at the
default canvas. That is ~45x tighter than a half-pixel, on every curve.

Maximum required `num_steps` across all curves:

| canvas | max required | vs today's 100 |
|---|---|---|
| 800px | 12 | 0.12x |
| 1500px (datashader default: figsize 10x10, dpi 150) | 16 | 0.16x |
| 4000px | 26 | 0.26x |

**These maxima were identical across every scenario measured**, synthetic and scale-free, tiny
through medium. The required uniform count is a property of the hive plot's axis geometry and the
canvas, not of the graph. And because it scales as `sqrt(canvas)`, a single conservative constant
covers all realistic output sizes: 26 is safe to a 4000px render.

That last point also dissolves the "adaptive must live on the fused rasterize-from-ids path because
only it knows the canvas" argument. The sqrt makes canvas-awareness nearly unnecessary.

### Empirical raster check

Rendered via `runners/performance/equivalence.rasterize_hive_plot` at a 1500px canvas, `num_steps=100`
as reference:

| scale | n | curve rows | support Δ | dropped | total count ratio |
|---|---|---|---|---|---|
| small (2k / 5k) | 16 | 53,530 → 9,010 | 0.70% | 0.37% | 0.996 |
| small | 8 | → 4,770 | 1.26% | 0.77% | 0.996 |
| small | 4 | → 2,650 | 3.71% | 2.51% | 0.994 |
| medium (25k / 250k) | 16 | 2,802,750 → 471,750 | 0.14% | 0.10% | 0.997 |
| medium | 8 | → 249,750 | 0.26% | 0.24% | 0.996 |
| medium | 4 | → 138,750 | 1.43% | 1.37% | 0.994 |

~5.9x less geometry for sub-1% raster change, degrading gracefully well below the flatness-derived
requirement.

*Methodology gotcha, recorded because it silently voided a first attempt:* `construct_curves()` only
builds subsets where `"curves" not in ...`, so re-calling it at a different `num_steps` is a **no-op**.
The `"curves"` key must be popped first. The first run of this comparison reported a perfect 0.00%
delta at every n down to 4, which is not physically possible and was the tell.

### What would need deciding before touching the default

Not proposing the change here; recording what it would cost.

- **The counts genuinely shift.** `cvs.line` increments per segment, so an oversampled curve bumps the
  same pixel repeatedly. Total counts move ~0.3-0.4% and per-pixel means move ~0.5-2% of max. Exact
  integer raster equivalence fails; `assert_raster_equivalent` would need a stated
  `allowed_mismatch_fraction`.
- **It is a user-visible default.** Existing plots' colormap scaling shifts slightly. Changelog and
  possibly deprecation-policy territory, not a silent bump.
- **Vector backends are untested.** matplotlib / bokeh / plotly / holoviews draw real polylines; the
  fidelity check above only covers the datashader raster. Expectation is a *larger* win there (file
  size, browser geometry) at equal fidelity, but that is expectation, not evidence. A human eye on a
  zoomed-in matplotlib PDF is the missing test.
- **Users can already pass `num_steps`.** The whole finding is reachable today by anyone who sets it.
  The question is only whether the shipped default should be better.

## On xy itself

Not adopting, and not primarily because of its age (created 2026-07-09, alpha). The structural
mismatch is that hiveplotlib draws each (axis pair, tag) as one large NaN-separated polyline, and
xy's fast paths are scatter-density and x-ordered lines. Our geometry would fall into its slowest
`direct` mode. Its density surface is datashader's idea rewritten in Rust, which we already have from
the canonical implementation.

Revival trigger for reconsidering xy as a backend: a 1.0 release with a documented arbitrary
polyline / path mark. Without that there is nothing to gain.

## Salvage

- **Adaptive per-curve sampling:** dead. Do not revisit without a scenario whose gate ratio is well
  below 0.3, and note that two independent graph families landed at 0.43-0.79.
- **Lowering the default `num_steps`:** live, ~6x, one constant, no architecture change. Needs its
  own decision on the three costs above.
- **Viewport-bounded re-aggregation with exact-row refinement on zoom** (the other idea salvaged from
  the xy read): untouched by this measurement, available via holoviews + datashader `rasterize()`
  with no new dependencies. Would land near `interactive-wasm-explorer.md`.
