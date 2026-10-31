# Plan: lower the default curve resolution (`num_steps` / `num_steps_per_edge`)

**Status: amended against the signed spec (2026-09-01); awaiting the plan-stage adversary and the
post-plan grill.** Feasibility measured 2026-08-01 (numbers and figures below). No code changed. The
number is 50 by the spec (signed 2026-08-31), so Workstream B is complete. Nothing dispatches until the
adversary's plan-stage pass and the grill have run (see "Alignment (grill)"). Workstream A is a release
gate rather than an input to the number and still runs before the default ships in a release.

## Goal

A hive plot built the usual way samples each edge at 50 points instead of 100, and the user types
nothing new to get it.

The fidelity claim is an analytic bound: how far a 50-segment polyline can sit from the true curve is
a property of the axis geometry and the canvas, not of the graph. At 50 it stays around a twentieth of
a pixel on the default canvas and under a quarter pixel even at a 4000px export, with headroom left
for the canvas-size and `control_rho_scale` multipliers that compound on top. The pixel measurements
back this up on the standalone configurations tested: there, no pixel differed from the old default
beyond 8/255 at native resolution.

What the user gets for free: rendering 1.2-1.3x faster, growing with scale, persisted curve memory
half the size, and the whole peak-memory win on the large datashader route, which plateaus below 50
anyway.

The new default reaches `HivePlot`, `HivePlotMatrix`, and `P2CP` alike, at construction and on every
rebuild (`P2CP` takes it at `build_edges`, its build step, rather than at construction). Anyone who
wants the old resolution back sets `num_steps_per_edge=100`; the docstrings say so, the way the
v0.27.0 datashader `dpi` change did.

The number is 50 and the spec commits to it. Moving it later, in either direction, is a re-sign
recorded on the spec, never a quiet adjustment.

Spec: `wiki/wiki/specs/lower-default-curve-resolution.md` (signed 2026-08-31). Transcribed
2026-09-01; the plan predates the spec convention, so this is the first transcription. The spec stays
canonical; the outcome statement above, the call shape under "API usage examples", and the top tier
of "Failure modes" are copies for the agents reading this plan.

This is a defaults-and-documentation change, not a scaling feature. It touches every backend and
every existing user's output, which is why it is scoped on its own rather than riding along with the
branch-53 scaling work.

**Brief-mode gate:** knowingly skipped at plan open (2026-08-01), when the brief arrived already
specified. Brief mode then ran at spec stage (2026-08-31), recorded on the spec's intent-extraction
line together with the failure-mode wave.

## Precedent this follows

v0.27.0 (released Apr 10, 2026) lowered the default `dpi` and the `pixel_spread_nodes` /
`pixel_spread_edges` parameters in `hiveplotlib.viz.datashader`:

> "This leads to comparable visualizations albeit at a lower resolution by default, but with
> significant speedups in plotting time. Users can still set a higher `dpi` value and adjust the
> pixel spread parameters as needed for higher-resolution visualizations."

Same trade, same shape: the default serves exploration, the docstring tells you how to buy back
fidelity when a figure is going to publication. This plan should match that documentation pattern
deliberately, not reinvent it.

## Alignment (grill)

```
Status: open decisions pending (plan-stage adversary and post-plan grill not run)
Open decisions: none left from the earlier placeholder. (1) The number is 50 by
  the spec. (2) and (3) The fused-path question is closed by unification: the
  datashader render-time num_steps defaults None and resolves to the instance's
  num_steps_per_edge, so that path inherits the default (Amendment 2026-09-01).
  The grill's agenda is whatever the plan-stage adversary raises.
Delivery shape: not yet called (default single MR)

What has run, where: brief mode and the failure-mode wave ran at spec stage
(spec "## Sign-off", intent-extraction line); the adversary's spec-stage pass
and api-critic's spec-stage walk of the call shape ran and were folded into the
spec before the 2026-08-31 signature. Not run: the adversary's plan-stage pass
on this plan, and the post-plan grill. Both precede any dispatch.
```

## Failure modes

Top tier transcribed from the spec (maintainer-named, failure-mode wave 2026-08-31). The three modes
the plan seeded on 2026-08-01 are the first three in the maintainer's words, reconciled rather than
duplicated; the fourth is new. A post-plan grill wave may add a second tier below.

- **The compromise is ugly at defaults**: faceting the maintainer's eye catches in a gallery figure,
  even if a pixel metric calls it equivalent.
- **Silently ugly on export**: unlike `dpi`, nobody sees `num_steps` change. Someone saves a
  publication figure and gets subtly polygonal curves with no signal that a knob exists. This is the
  asymmetry with the dpi precedent and the reason the number is conservative.
- **Equivalence measured only where it was easy**: everything is measured through rasters; vector
  PDF/SVG at arbitrary zoom is the case a raster metric structurally cannot see. Workstream A is the
  release gate on this mode.
- **The parameter pass-through is not clean in practice**: if changing the value on any of the
  higher-level classes does not happen cleanly (setting it at construction, or changing it on an
  existing plot), the scope expands to fix the pass-through rather than shipping a default nobody can
  move. Workstream C's widened sweep (the constant has to reach `HivePlotMatrix` and `P2CP`) is the
  first consequence of this mode; the remaining known trap is `construct_curves` silently skipping a
  built instance, documented under "API usage examples" and tested under Workstream F.

## Prior ADRs / design docs

- None as an ADR. The governing precedent is the v0.27.0 datashader `dpi` change, recorded only in
  `CHANGELOG.rst` (see "Precedent this follows" above).
- `wiki/wiki/plans/archived/adaptive-curve-sampling.md` — the measurement that produced this finding,
  and the record of why *adaptive* per-curve sampling was rejected in favour of a lower constant.
- `wiki/wiki/adr/0002-performance-regression-harness.md` binds Workstream F: old-default baselines
  and fixtures are not equivalence-comparable to the new default, so they get pinned at 100 or
  re-baselined, never compared across the change; any performance claim ships only through the
  regression harness and names the user-reachable path it was measured on; ASV history is captured at
  the merge on the canonical host so the step attributes to this change.

## Feasibility evidence (measured 2026-08-01)

### Geometric fidelity

For a quadratic Bézier, max deviation of a uniform k-segment polyline from the true curve is
`D / (4k²)` where `D = ‖P₀ - 2·P₁ + P₂‖`. Worst curve in a representative plot, converted to pixels:

| `num_steps` | max deviation @ 1500px (default canvas) | @ 4000px |
|---|---|---|
| 100 (current) | 0.011 px | 0.030 px |
| 50 | 0.046 px | 0.12 px |
| 25 | 0.194 px | 0.52 px |
| 16 | 0.496 px | 1.32 px |
| 8 | 2.276 px | 6.07 px |

The worst-case requirement is a property of the hive plot's axis geometry and the canvas, **not of
the graph**: the same maxima came back across uniform-synthetic and scale-free graphs at every scale
tested. It scales as `sqrt(canvas)`, so one constant covers a wide range of output sizes.

### Runtime

Construct + render, isolated process per cell, seed 7, `example_hive_plot` fixtures.

| scenario | n=100 | n=50 | n=25 | n=16 |
|---|---|---|---|---|
| medium (25k/250k), matplotlib | 1.98 s | 1.55 s (1.28x) | 1.37 s (1.45x) | 1.22 s (1.63x) |
| medium (25k/250k), datashader | 4.84 s | 4.07 s (1.19x) | 3.85 s (1.26x) | 3.71 s (1.30x) |
| large (100k/2M), datashader | 13.15 s | 9.98 s (1.32x) | 8.01 s (1.64x) | 6.70 s (1.96x) |

Construction time is roughly flat; the win is almost entirely in rendering. The speedup grows with
scale, which is the right direction.

### Memory

| scenario | metric | n=100 | n=50 | n=25 | n=16 |
|---|---|---|---|---|---|
| large (100k/2M) | persisted curve arrays | 1541 MiB | 778 MiB | 397 MiB | 259 MiB |
| large, datashader | peak RSS | 2471 MiB | 2158 MiB | 2161 MiB | 2160 MiB |
| medium, datashader | peak RSS | 1110 MiB | 824 MiB | 682 MiB | 630 MiB |
| medium, matplotlib | peak RSS | 943 MiB | 658 MiB | 514 MiB | 506 MiB |

Curve memory is exactly linear in `num_steps`, as expected. Note the **peak-RSS plateau on the large
datashader route**: below n=50 the curve arrays stop being the peak, so further reductions keep
buying speed but stop buying peak memory. Worth knowing before over-optimizing the number.

### Visual evidence

**Methodology note, load-bearing.** A first pass compared candidates in side-by-side subplots. That
is invalid for this question: at fixed figure dpi, each hive plot in an N-panel composite occupies
~600px, where a real standalone plot gets 1000px (matplotlib default `figsize=(10,10)` at dpi 100)
or 1500px (datashader default dpi 150). Shrinking a render hides precisely the faceting the test
exists to find, and the inline review downscaled a further ~1.5x on top. Net understatement roughly
2.5-3x. Every figure below is rendered **standalone through the library's own default canvas**, one
file per candidate, and compared at native resolution. Composite figures are not evidence here.

Files in the session scratchpad (not committed): `.../scratchpad/standalone/` and
`.../scratchpad/pixelzoom/`.

**Whole-figure difference vs n=100, matplotlib standalone (1000x1000 native), sparse 60/90 plot:**

| n | mean \|Δ\| | p99 \|Δ\| | max \|Δ\| | px >8/255 | px >32/255 |
|---|---|---|---|---|---|
| 50 | 0.045 | 2 | 7 | 0.00% | 0.00% |
| 25 | 0.208 | 7 | 29 | 0.77% | 0.00% |
| 16 | 0.548 | 20 | 69 | 2.41% | 0.16% |
| 8 | 2.555 | 84 | 202 | 4.54% | 2.90% |

**Worst 120x120 native window in that figure** (located by max deviation, same window for all
candidates, magnified 6x nearest-neighbour in `pixelzoom/zoom_n*.png`):

| n | mean \|Δ\| | max \|Δ\| | px >8/255 |
|---|---|---|---|
| 50 | 0.29 | 6 | 0.00% |
| 25 | 1.31 | 27 | 6.06% |
| 16 | 3.49 | 64 | 14.71% |
| 8 | 16.44 | 191 | 24.49% |

At 6x pixel magnification n=50 is indistinguishable from n=100. n=16 shows a small number of curves
shifted by a pixel. n=8 shows both positional shift and visible straightening on the long arcs.

**datashader standalone (1500x1500 native), 25k/250k:** barely moves at any candidate. Even n=8 has
p99 of 4/255; max |Δ| of 106-143 is confined to a one-pixel rim on the outer envelope and a crescent
at the inner hole.

| n | mean \|Δ\| | p99 \|Δ\| | px >8/255 |
|---|---|---|---|
| 50 | 0.210 | 2 | 0.17% |
| 25 | 0.236 | 2 | 0.18% |
| 16 | 0.257 | 3 | 0.22% |
| 8 | 0.429 | 4 | 0.48% |

**The inversion worth recording:** the sensitive case is the *sparse vector* plot, not the dense
at-scale one. Density rendering averages over many overlapping curves, so a single curve's deviation
washes out; a sparse plot draws each curve as itself with nothing to hide behind. Any future check of
this kind should lead with the sparse case.

**The shape of the error matters as much as its size.** The whole-figure diff maps
(`diffmaps/diff_n*_amp4x.png`) show that at 50/25/16 the error is thin ghosting along curve edges:
the curve is in nearly the same place and its antialiased boundary has shifted a fraction of a pixel.
That is the benign mode, and it is why n=16 survives eyeballing better than its numbers suggest. At
n=8 the diff thickens and picks up structure at segment joints, which is the mode a reader actually
perceives as polygonal.

### Headroom: what the fidelity numbers held fixed

Every number above was measured at `control_rho_scale=1`. That is a documented public parameter whose
job is to make edges more or less convex, i.e. to move exactly the quantity that sets required n.
Sweeping it on the default 1500px canvas:

| `control_rho_scale` | max required n | vs default | deviation @ n=50 | @ n=25 | @ n=16 |
|---|---|---|---|---|---|
| 0.5 | 20 | 1.25x | 0.069 px | 0.27 px | 0.73 px |
| 1.0 (default) | 16 | 1.00x | 0.046 px | 0.19 px | 0.49 px |
| 2.0 | 23 | 1.44x | 0.096 px | 0.40 px | 1.03 px |
| 4.0 | 26 | 1.62x | 0.126 px | 0.54 px | 1.35 px |

The default sits near a **minimum**: moving the control point either outward or inward raises the
requirement. So the shipped default has to absorb two independent multipliers the measurements held
constant, canvas size (as `sqrt`) and `control_rho_scale` (up to 1.62x), and they compound.

At n=50 the worst combination tested stays around 0.13 px, and roughly 0.35 px even at a 4000px
export. At n=25 the same combination reaches 0.54 px on the *default* canvas alone. At n=16 it is
1.35 px on the default canvas, which is visible faceting with no signal to the user that a knob
exists. Because deviation goes as `n²`, the 3x in n between 16 and 50 is ~10x in fidelity headroom.

**Not yet shown, and a release gate:** vector output. Everything above is a raster. A matplotlib
PDF/SVG at high zoom is the case where faceting would survive that a raster metric cannot see.
Workstream A compares 50 against 100 there before the default ships in a release; a failure re-signs
the number on the spec.

## Candidate numbers

| | deviation @ default | @ 4000px | worst-window px >8/255 | speed (large ds) | curve memory |
|---|---|---|---|---|---|
| **50** | 0.046 px | 0.12 px | **0.00%** | 1.32x | 2.0x less |
| **25** | 0.194 px | 0.52 px | 6.06% | 1.64x | 3.9x less |
| 16 | 0.496 px | 1.32 px | 14.71% | 1.96x | 5.9x less |
| 8 | 2.276 px | 6.07 px | 24.49% | — | 11.2x less |

**Recommendation: 50.** At true standalone resolution not a single pixel in the whole figure differs
from n=100 by more than 7/255, and nothing anywhere exceeds 8/255. It holds sub-quarter-pixel
fidelity even at a 4000px export, captures the entire peak-RSS win on the large route (which
plateaus below 50 anyway), and needs no caveat about high-dpi output.

The corrected measurement **weakened the case for 25**, which the earlier composite had flattered:
6% of pixels in the worst window differ by more than 8/255, where 50 has literally none. 25 remains
defensible if we are willing to lean on documentation the way the dpi change did, but it is a real
visual compromise rather than a free one, and part of what it buys is speed the memory plateau says
we cannot bank.

The stronger argument for 50 is **headroom, not appearance** (see "Headroom" above). 16 and 25 both
look fine in the configuration that was measured; neither has margin for two multipliers users
legitimately turn (canvas size and `control_rho_scale`), which compound and which the user gets no
warning about. 50 absorbs the worst tested combination and still lands around a tenth of a pixel.

Settled at 50 by the spec, signed 2026-08-31. The table and the argument above stay as the record of
why.

## Patterns this replaces

The premise that one constant governs is false today. `_DEFAULT_NUM_STEPS_PER_EDGE`
(`src/hiveplotlib/hiveplot.py:98`) reaches `HivePlot` alone: `HivePlot.__init__` takes it at `:2901`
and `BaseHivePlot` initializes the attribute from it at `:614`. Every other site below carries its own
literal `100` and forwards it explicitly, so flipping the constant by itself does not deliver the
outcome. Workstream C collapses all of them onto the constant. Line refs verified against branch 53
on 2026-09-01; they drift, so grep before citing. The 2026-08-02 list (`hiveplot.py:1401, :1570,
:1643, :1810, :2151, :2650`, `hiveplot_matrix.py:1068, :1453, :1922`) is superseded.

Signature defaults that stomp the constant (each forwards its value explicitly):

- `src/hiveplotlib/hiveplot_matrix.py:1107`, `:1501`, `:1982`: the three `HivePlotMatrix`
  constructors, `num_steps_per_edge: int = 100`, forwarded into every `HivePlot(...)` cell build
  (`:1398`, `:1427`, `:1822`, `:1871`, `:1918`, `:2262`).
- `src/hiveplotlib/p2cp.py:231`: `P2CP.build_edges(num_steps: int = 100)`, which assigns its value
  over the underlying hive plot's attribute at `:279` and passes it to `connect_axes` on every call.
  Becomes `int | None = None`, resolving to the underlying hive plot's `num_steps_per_edge` (which the
  base class already initializes from the constant), so its documented declared-rate semantics hold:
  an explicit value still becomes the P2CP's declared rate; `None` builds at the rate already
  declared.
- `src/hiveplotlib/hiveplot.py:1631` (`_construct_edge_subset_curves`) and `:1704`
  (`_curves_from_id_pairs`): private helpers with `num_steps: int = 100`. Every caller passes
  `num_steps` explicitly (`hiveplot.py:1609`, `:1687`; `viz/datashader.py:468`, `:801`), so these are
  dead literals rather than live defaults; they join the sweep so no literal survives.

Docstring literals stating the old default as a number:

- `src/hiveplotlib/hiveplot.py:537` (the `num_steps_per_edge` attribute docstring, "Default ``100``")
  and `:2812` (the `HivePlot.__init__` parameter, "Default ``100``").
- `src/hiveplotlib/hiveplot_matrix.py:1219`, `:1615`, `:2110` state no default at all ("number of
  Bezier curve steps per edge."); Workstream D gives them the default and the two directions.
- `src/hiveplotlib/p2cp.py:246-253` states no number; its declared-rate paragraph gains the `None`
  resolution once `build_edges` changes.

Already resolving to the constant, no edit (verified): `construct_curves` (`hiveplot.py:1871`),
`connect_axes` (`:1454`), `add_edge_curves_between_axes` (`:2280`), all `num_steps: int | None =
None`; and the datashader render-time `num_steps` on both entry points (`viz/datashader.py:996`,
`:1577`, resolved at `:427`, documented at `:1107` and `:1722`), which is why the fused
rasterize-from-ids path inherits the new default with no site of its own.

Prose carrying arithmetic derived from the default: none remain. Both sites the 2026-08-02 sweep
found have since gone symbolic on branch 53 and moved to Holdouts: the `construct_curves` note now
reads "roughly ``num_edges * (num_steps + 1) * 8`` bytes" (`hiveplot.py:1902-1903`), and
`examples/creating_hive_plots_from_dask.ipynb`'s "## Partition Sizing" cell now states
`n * (num_steps + 1) * 8` and pins its worked guideline explicitly "at `num_steps=100`", which stays
arithmetically true after the change. Re-basing that guideline at 50 is Workstream G's editorial
call, not a correctness fix.

Kept at 100, by the spec: `src/hiveplotlib/utils.py:75`, `:96`, `:254` (`bezier`, `bezier_all`,
`bezier_xy_with_nans`). Listed under Holdouts so the sweep does not flag them.

Tests and runners pinned to the old shape, owned by Workstream F: `tests/hiveplot_test.py:1769` and
`tests/p2cp_test.py:686` assert `arr.shape == (101, 2)`;
`runners/performance/profile_hiveplot_ds.py:201` sets `num_steps = 100` explicitly. Tests already
reading `_DEFAULT_NUM_STEPS_PER_EDGE` follow the constant and need nothing (listed under Holdouts).

## Default justifications

- `num_steps_per_edge=50` (and every `num_steps` that resolves to it): the default canvas cannot
  resolve finer, measured; 50 keeps about a tenth of a pixel of headroom under the two multipliers the
  measurements held fixed and users legitimately turn (canvas size as `sqrt`, `control_rho_scale` up
  to 1.62x, compounding), moved zero pixels beyond 8/255 at native resolution on the standalone
  figures, and captures the whole peak-RSS win on the large datashader route, which plateaus below 50.
  25 was rejected because a plain 4000px export already sits at 0.52 px with no rho fiddling at all.
  Users raise it for publication output exactly as they raise `dpi`. Settled by the spec, signed
  2026-08-31.
- `utils.bezier*` keep `num_steps=100`: they are the primitive, not the workflow; nobody should call
  them directly, and anyone who does is the user who knows the knob exists. The spec's call.
- `P2CP.build_edges(num_steps=None)`: `None` means "the rate this P2CP already declares", which is the
  constant unless an earlier `build_edges` set one; an explicit value keeps its documented meaning as
  the new declared rate. A sentinel rather than a restated `50` keeps the sweep at one constant.

## Naming audit

- Existing inconsistency surfaced, not introduced by this work: the same concept is `num_steps` on
  `construct_curves` / `add_edge_curves_between_axes` / `P2CP` and `num_steps_per_edge` on the
  `HivePlot` / `HivePlotMatrix` constructors. Renaming is out of scope here and tracked as GitLab
  issue #67 (both names shipped in v0.28.0, so unification is a deprecation cycle in its own right).
- No new names introduced.

## API usage examples

No API surface change: parameters, names, and types are unchanged. Only the default value moves
(`P2CP.build_edges`'s `num_steps` additionally accepts `None` as the sentinel for the declared rate,
the type shape `construct_curves` already uses).

### Proposed (planner)

Both fences transcribed from the spec's call shape (signed 2026-08-31), `Path:` lines attached.

```python
# The usual path, unchanged: this exact code gets faster and lighter, with no visible change
from hiveplotlib.datasets import example_hive_plot
from hiveplotlib.viz.matplotlib import hive_plot_viz

hp = example_hive_plot(num_nodes=2_000, num_edges=5_000, seed=7)
fig, ax = hive_plot_viz(hp)
```

Path: `HivePlot.__init__` (here via `example_hive_plot`, which forwards to it) takes
`num_steps_per_edge` from the module constant `_DEFAULT_NUM_STEPS_PER_EDGE` in `hiveplot.py`; curves
build at construction and `hive_plot_viz` draws them as stored.

Today that constant covers `HivePlot` alone: the `HivePlotMatrix` constructors, `P2CP.build_edges`,
and two private helpers in `hiveplot.py` carry their own literal 100s that stomp it. Collapsing those
literals onto the one constant is in scope for this change; flipping the constant by itself does not
deliver the outcome. "Patterns this replaces" carries the site list.

Once collapsed, `construct_curves`, `connect_axes`, and `add_edge_curves_between_axes` (default
`num_steps=None`, resolving against the instance's `num_steps_per_edge`) put every rebuild path on
the new default through that single constant. The datashader render-time `num_steps`, the fused
rasterize-from-ids path included, defaults to `None` and resolves the same way, so every render path
inherits the new default with it.

```python
# Publication output: buy the old resolution back at construction
hp = example_hive_plot(num_nodes=2_000, num_edges=5_000, seed=7, num_steps_per_edge=100)
fig, ax = hive_plot_viz(hp)
fig.savefig("figure.pdf")
```

Path: `HivePlot.__init__(num_steps_per_edge=100)`; every rebuild (`build_hive_plot`,
`connect_adjacent_axes`) redraws at that value. On an existing plot the supported path is per class:
for `HivePlot`, assign `num_steps_per_edge` and call `build_hive_plot()`; for `P2CP`, pass
`num_steps` to `build_edges(...)`, its lever, at build time rather than construction.

Two calls that look like this escape hatch and are not: `construct_curves(num_steps=100)` on an
already-built instance skips every subset that holds curves, so it runs fine and resamples nothing
(it is the build call for instances holding ids alone, e.g. `persist_curves=False`); and the
datashader render-time `num_steps` kwarg raises peak memory on a curve-persisted instance rather than
replacing what is stored. Construction or a full rebuild is the lever.

Branch-53 witnesses for the above (planner, 2026-09-01): `example_hive_plot(**hive_plot_kwargs)`
forwards into `HivePlot(...)` (`src/hiveplotlib/datasets/toy_hive_plots.py:372`, `:457-462`); the
skip is the `"curves" not in ...` guard at `hiveplot.py:1952`, and a default build holds curves
because `persist_curves: bool = True` (`:2902`); the `None` resolution is exercised by the
`*_resolves_num_steps` tests in `tests/hiveplot_test.py` (parametrized at `:2476`) and, for P2CP, by
`_p2cp_at_num_steps` in `tests/datashader_test.py` (`:2180`). The post-construction assignment path
has no test today; Workstream F adds one. The plan's earlier Example 2
(`hp.construct_curves(num_steps=100)` after a default build) was exactly the first trap and is
withdrawn (Amendment 2026-09-01).

### API Critic's take (planning mode)

Ran at spec stage (2026-08-31) against the spec's call shape, reported to the dispatching session and
folded into the spec before signature; the `build_edges` `int | None = None` shape is its suggestion.
Nothing to record here beyond that pointer.

## Adversary review

### Adversary's challenge (planning mode)

```
Status: challenge (9 items)
Plan reviewed: wiki/wiki/plans/lower-default-curve-resolution.md (plan stage, cold, 2026-09-01)
Angles worked: approach; size-and-maintenance against the workstream set A-G. Premise and
  could-this-not-exist were settled at spec stage and are not reopened; the plan's Goal
  matches the signed outcome statement verbatim. Line refs below verified on branch 53 today.
Items:
  - [must-fix] `P2CP.build_edges(num_steps: int | None = None)` contradicts the signed spec
    and changes repeat-call semantics, not just the number (at Workstream C done-when;
    "Default justifications" third bullet; "API usage examples" first paragraph)
    Rubric: The parameter pass-through is not clean in practice
    Push: the spec's "Out of scope" reads "Parameters, names, and types are unchanged;
    only the default value moves", and nothing in the spec's outcome or call shape names a
    `None` sentinel on `build_edges` (lines 11 and 39 name the method alone), so the
    "folded in at spec stage" attribution is not visible in the signed text. It is also a
    behavior change: today `build_edges(num_steps=16)` then `build_edges()` re-declares 100
    (the docstring at `p2cp.py:251`, "calling this method again replaces it"); under `None`
    the bare call keeps 16, and that sentence goes false for the bare call. Use
    `num_steps: int = _DEFAULT_NUM_STEPS_PER_EDGE`: type unchanged, replace-on-every-call
    unchanged, no second literal, spec true as signed. If `None` is wanted anyway, that is
    a re-sign of the out-of-scope bullet plus a rewrite of `p2cp.py:246-253`, and the grill
    should say which. Whichever lands, the three `HivePlotMatrix` constructors take the
    same shape (constant import or `None`); record the choice once.
  - [must-fix] `HivePlotMatrix` has no post-construction lever and the spec's per-class
    rebuild list omits it (at "Feasibility audit" under the 2026-09-01 C amendment;
    Workstreams D and F; spec call shape, second `Path:` line)
    Rubric: The parameter pass-through is not clean in practice
    Push: the mode names "any of the higher-level classes ... changing it on an existing
    plot". The matrix stores no rate (zero `self.num_steps_per_edge` in
    `hiveplot_matrix.py`; the six forwards read the constructor parameter), while its
    sibling setting has a matrix-level fan-out, `set_persist_curves` (`:966`, "the matrix
    itself stores no persistence setting of its own"). The audit's "survives every rebuild
    path enumerated" enumerates no matrix path. Decide and record one of: (a) the lever is
    per-cell assignment via `iter_populated_cells()` followed by a matrix rebuild such as
    `unify_axes`, documented in D's three matrix docstrings and tested in F next to the
    `HivePlot` hatch; (b) a `set_num_steps_per_edge` fan-out mirroring `set_persist_curves`,
    which is new surface and therefore a re-sign; (c) construction-only for the matrix,
    which the spec's `Path:` line should then say (a one-clause re-sign). Recommend (a):
    zero surface, and it turns the mode's "scope expands" clause into two docstrings and
    one test.
  - [must-fix] The sweep's test-side bookkeeping misses pins that encode "the default is
    100", and one of them fails silently (at Workstream F done-when; Workstream C
    done-when, the sweep clause; Holdouts)
    Rubric: no entry, flagging anyway
    Push: `tests/p2cp_test.py:414` (`pytest.param({}, 100, id="default_num_steps")`)
    asserts the declared rate after a bare `build_edges()` is 100; loud failure under C,
    listed nowhere. `tests/dask_test.py:550-551` ("default construction parameters on this
    route sample 100 steps per curve", `num_steps = 100`) feeds `n_curves * num_steps >=
    floor`; at a 50 default the filter is 2x too permissive and `assert
    len(above_floor_serial) > 0` stays green, so the test keeps passing on a stale premise
    with no CI signal. That is the seventh forgotten 100, in the one place a grep for
    `= 100` on `src/` cannot see. `tests/p2cp_test.py:646`, `:653`, `:658` (`(202, 2)`,
    `arr[99, :]`, `arr[101, :]`) are the same pin as `:686`, so "the two shape asserts"
    undercounts. F lists all of them with a disposition (p2cp:414 reads
    `_DEFAULT_NUM_STEPS_PER_EDGE`; dask:551 reads `hive_plot_dask.num_steps_per_edge`;
    p2cp:646/653/658 follow :686), and C's sweep clause states its pattern and that it
    runs over `tests/` and `runners/` too, e.g. `\b(100|101|202)\b` within a few lines of
    `num_steps|shape|arr\[`, rather than an unstated grep whose survivors then get waived
    ad hoc.
  - [must-fix] Workstream A has no mechanical arming trigger and no pass/fail definition
    (at Workstream A; Workstream E; "Headroom", last paragraph)
    Rubric: Equivalence measured only where it was easy; Silently ugly on export
    Push: "release gate" is prose. Nothing in C, E, or the CHANGELOG blocks a release if A
    never runs, and the house precedent (ADR 0002's arming canary) is that dormant gates
    arm mechanically. "Publication zoom" is also undefined: a polyline of any `n` facets at
    enough zoom, 100 included, so the check needs a fixed bar. A needs no code (it is
    `num_steps_per_edge=50` versus `100`, explicit, today), so sequence it before C and
    delete the gate abstraction. Its done-when names the figure (the sparse 60/90
    matplotlib plot, per the plan's own inversion), the formats (PDF and SVG), the bar
    (rasterize the vector file at 600 dpi or more, above the 4000px column, and one
    on-screen zoom to the worst window), the `control_rho_scale=4` arm the headroom table
    calls worst, and "verdict recorded in the Implementation log; a fail re-signs". If it
    must stay a release gate, E's done-when cites A's verdict line as a prerequisite for
    the CHANGELOG entry, which is the mechanical trigger.
  - [worth-discussing] C and D edit the same docstring lines in two sequential passes by
    two agents (at Workstream C done-when, the "two Default ``100``" clause; Workstream D)
    Rubric: Silently ugly on export
    Push: C moves the literals at `hiveplot.py:537` and `:2812`; D rewrites the paragraphs
    those literals sit in (`:2811-2813` and the attribute docstring) for direction. Fold D
    into C (eight docstrings, one pass, docs-engineer reviews), or C leaves every docstring
    word to D and lists the two literals as C-stage Holdouts until D lands. Recommend the
    merge: the docstrings are the only export signal the spec promises, and one author
    writes a coherent one.
  - [worth-discussing] The two private helpers should lose their default, not gain a
    constant and a drift test (at "Patterns this replaces", private helpers bullet;
    Workstream C done-when, the drift-test clause)
    Rubric: no entry, flagging anyway
    Push: every caller passes `num_steps` (`hiveplot.py:1613`, `:1691`; the datashader
    sites the plan lists), so the literal is dead. Making `num_steps` required on
    `_construct_edge_subset_curves` and `_curves_from_id_pairs` deletes the literal, the
    per-site drift test, and the next forgotten 100 in one move. For the surviving public
    sites, one parametrized signature-inspection test in the shape of
    `test_hive_plot_and_base_hive_plot_carry_the_library_num_steps_default`
    (`tests/hiveplot_test.py:2623-2635`; the plan cites `:2611`, which sits inside the
    neighboring test) covers the three matrix classmethods and `build_edges` together.
  - [worth-discussing] E re-opens a question the spec closed, and F carries a clause with
    no referent (at Workstream E done-when and Files; Workstream F done-when, the
    "tolerance question" and "performance figure" clauses)
    Rubric: no entry, flagging anyway
    Push: the spec's "Alternatives considered" already rules: "Warning machinery ahead of
    the change: rejected ... silent change, CHANGELOG entry, docstrings". E's "deprecation-
    policy question ... answered or explicitly deferred" and "possibly the deprecation-
    policy plan" re-litigate that; E is one `Changed` entry in the v0.27.0 shape
    (`CHANGELOG.rst:403-407`) and nothing else. Two consequences for F: any ratio in that
    entry ("1.2-1.3x faster") is a performance claim, which per ADR 0002 ships only through
    the harness, and the plan's figures came from a scratchpad, so either the entry carries
    no number or F re-measures on the ASV scenarios (`benchmarks/benchmarks.py` pins no
    `num_steps`, so a capture at merge shows the step for free) and names the path. And
    "the raster equivalence gate's tolerance question is resolved" names a question that
    appears nowhere in the plan; both sides of that gate run at the same default, so
    either state the question or delete the clause.
  - [worth-discussing] G's "where a reader first meets the knob" has no notebook location,
    so the item silently resolves to docstrings alone (at Workstream G, third done-when
    item; the 2026-09-01 G amendment)
    Rubric: Silently ugly on export
    Push: `examples/` names `num_steps` in exactly one notebook,
    `creating_hive_plots_from_dask.ipynb`, which teaches lowering. The
    `customizing_edge_curves.ipynb` candidate was dropped with the rho caveat, but the
    raise-it note was never tied to the rho finding. Either name the surface now (the
    large-networks section states both directions in one sentence, so G has one location;
    or `customizing_edge_curves.ipynb` gets a two-sentence raise-it note, decoupled from
    the dropped caveat) or delete the item and say plainly that docstrings alone carry the
    export signal. What should not survive is an item whose owner is "the notebook
    author's call" with no candidate left to call.
  - [low-confidence] "The post-construction assignment path has no test today" overstates,
    and F's new test should target the two paths that are actually unpinned (at "API usage
    examples", branch-53 witnesses paragraph; Workstream F done-when)
    Rubric: The parameter pass-through is not clean in practice
    Push: `test_construct_curves_honors_num_steps_per_edge_assigned_after_construction`
    (`tests/hiveplot_test.py:2599`) already pins assign-then-`construct_curves()` on
    `BaseHivePlot`. Unpinned are assign-then-`build_hive_plot()` on `HivePlot` (the spec's
    named path; `connect_adjacent_axes` reads the attribute at `hiveplot.py:4940`, `:4994`,
    `:5005`) and whichever matrix path the second item settles. Name those two as F's
    targets so the test does not duplicate the one that exists.
```

### Adversary — post-implementation

```
Pending.
```

## Notebook review

```
Pending — invoke editorial-critic in post-implementation mode after Workstream G ships.
```

## Viz review

```
Pending — viz-critic should review the sign-off figures and any regenerated gallery
figures against the viz-quality-bar skill.
```

## Workstreams

### Workstream A: vector-output fidelity check

**Status:** not started. Release gate (re-aimed 2026-09-01 by the spec; see Amendments). Runnable now,
in parallel with the adversary and the grill; no code.
**Files:** none (investigation)
**Done when:** matplotlib PDF/SVG output at 50 is compared against 100 at publication zoom and either
clears the maintainer's eye or produces a documented counterexample. Completes before the new default
ships in a release; a failure re-signs the number on the spec rather than adjusting it here. The 25 and
16 arms of the earlier sweep are dropped: A no longer informs the choice of number.

### Workstream B: settle the number

**Status:** complete, by the spec (signed 2026-08-31). The number is 50; the justification is recorded
under "Default justifications". Nothing to dispatch.

### Workstream C: change the defaults

**Status:** not started; blocked on the plan-stage adversary and the grill. Widened 2026-09-01; see
Amendments.
**Files:** the "Patterns this replaces" list (`hiveplot.py`, `hiveplot_matrix.py`, `p2cp.py`)
**Done when:** `_DEFAULT_NUM_STEPS_PER_EDGE` is 50 and every signature default listed under "Patterns
this replaces" resolves to it: the three `HivePlotMatrix` constructors and the two private helpers
take the constant; `P2CP.build_edges` becomes `num_steps: int | None = None`, resolving to the
underlying hive plot's `num_steps_per_edge` while an explicit value still becomes the declared rate;
the two "Default ``100``" docstring literals in `hiveplot.py` move; a drift test pins each collapsed
site to the constant (the existing one at `tests/hiveplot_test.py:2611` covers `HivePlot.__init__`
alone); and the replace-and-sweep grep finds no surviving curve-resolution `100` outside Holdouts. No
arithmetic prose remains to recompute (both sites went symbolic; see "Patterns this replaces"), so C
no longer waits on G's notebook item.

### Workstream D: docstrings and the escape hatch

**Status:** not started
**Scope:** docstrings only. Notebook prose split to Workstream G (see Amendment 2026-08-02).
**Files:** `src/hiveplotlib/hiveplot.py`, `hiveplot_matrix.py`, `p2cp.py`
**Done when:** the `num_steps` / `num_steps_per_edge` docstrings present the default as the middle of
a usable range rather than a ceiling, naming both directions: raise it when a figure is going to
publication (the v0.27.0 `dpi` pattern), lower it further when scaling to large networks. Today's
text ("Larger numbers will result in smoother curves when plotting later, but slower rendering", at
`hiveplot.py:1488-1489`, `:1914-1915`, `:2338-2339`, `:2811-2813`, and `p2cp.py:246-247`; the matrix
docstrings at `hiveplot_matrix.py:1219`, `:1615`, `:2110` state neither a default nor a direction)
states the trade without telling anyone when to move it either way. Every docstring that points a
user at raising the number points at construction or a full rebuild, never at `construct_curves` on a
built instance (its own note at `hiveplot.py:1891-1899` already says that call does nothing there).
Citations corrected 2026-09-01; see Amendments.

### Workstream E: changelog and release framing

**Status:** not started
**Files:** `CHANGELOG.rst`, possibly the deprecation-policy plan
**Done when:** the entry states plainly that existing plots change slightly, and the
deprecation-policy question (does a default change of this kind warrant more than a changelog entry?)
is either answered or explicitly deferred.

### Workstream F: test and gate fallout

**Status:** not started
**Files:** `tests/`, `runners/performance/`, `benchmarks/`
**Done when:** the two shape asserts pinned to the old default (`tests/hiveplot_test.py:1769` and
`tests/p2cp_test.py:686`, `(101, 2)`) either pass `num_steps_per_edge=100` / `num_steps=100`
explicitly or move to `(51, 2)`, recorded either way; `runners/performance/profile_hiveplot_ds.py:201`'s
explicit `num_steps = 100` is kept as a pin or moved to the constant, recorded; a test covers the
post-construction escape hatch (`hp.num_steps_per_edge = 100`, then `build_hive_plot()` rebuilds at
100), which no test pins today; per ADR 0002 no old-default baseline or fixture is compared for
equivalence against the new default (pin or re-baseline, never compare across); the raster equivalence
gate's tolerance question is resolved; ASV scenarios are re-captured at the merge on the canonical host
so the history step attributes to this change; and any performance figure this plan ships names the
user-reachable path it was measured on. Bound to ADR 0002 on 2026-09-01; see Amendments.

## Plan amendments

### Added workstream G: notebook prose for the resolution range

**Date:** 2026-08-02
**Trigger:** Maintainer ask — document the knob in example notebooks as well as docstrings, framed as
two opposite-direction moves from the shipped default rather than one.
**Status:** blocked on Workstream B (the number), which is itself blocked on the grill and Workstream A
**Files:** `examples/creating_hive_plots_from_dask.ipynb`,
`examples/hive_plots_for_large_networks.ipynb`, `examples/customizing_edge_curves.ipynb` (candidate)

Split out of Workstream D rather than widening it: notebook prose is notebook-author's surface and
draws an editorial-critic post-impl pass, docstrings are docs-engineer's and draw none. D and G touch
disjoint file sets (`src/` vs `examples/`) and can run in either order once the number is settled.

**Done when:**

- `examples/creating_hive_plots_from_dask.ipynb`, "## Partition Sizing": the default-derived
  arithmetic is recomputed (the 808-bytes-per-edge figure, both `808` formulas, and the
  "10-million-edge partition holds about 8 GB" worked example). This item is a correctness fix, not
  teaching prose; it ships in the same change as Workstream C and is never released with C alone.
- `examples/hive_plots_for_large_networks.ipynb` carries the lower-it-at-scale message as a sibling
  of its existing "Lower the Alpha Value" / "Thinner Lines" knob sections. It names the direction and
  the shape of the win (render time and curve memory; construction time is roughly flat) without
  reproducing this plan's benchmark tables as claims a reader cannot re-derive on their own data.
- The raise-it-for-publication direction is stated where a reader first meets the knob. Whether that
  is `customizing_edge_curves.ipynb` (genre fit: it is the curve-geometry notebook, and its
  `control_rho_scale` section is the parameter this plan's headroom analysis found interacts with
  required resolution) or docstrings alone is the notebook author's call against genre; record which,
  either way.
- `docs/source/_llms/llms.txt`: **no new entry and no edit.** Explicit call, made here so it is not
  re-adjudicated downstream: this is a tuning knob on an existing capability, not a new capability,
  class, backend, or conceptual entry point; both target notebooks already carry entries whose
  one-line descriptions ("tune the curvature of edges between axes", "scale hive plots to large edge
  counts", "Covers partition sizing and failure guidance") stay accurate after the change; and no
  page is renamed or removed.
- Only `examples/` notebooks are edited. `docs/source/notebooks/` and `docs/source/gallery_examples/`
  are generated.
- Any new render is justified against `make test-nb` runtime: prose-first, at most one new figure,
  and any new figure goes to viz-critic.
- No process references in shipped prose (no plan, workstream, or grill references, no dates).

*(2026-09-01: the first item is retired and the `customizing_edge_curves.ipynb` candidate dropped;
status moves off B. See "In-scope tweak: Workstream G's arithmetic item retired" below.)*

### In-scope tweak: Workstream D narrowed to docstrings

**Date:** 2026-08-02
**Trigger:** Same maintainer ask (the documentation split above).
**Workstream affected:** D, docstrings and the escape hatch
**Change:** Files drop "docs prose"; done-when moves from one direction ("explain the explore-vs-publish
trade and name the way to raise it") to two ("the default is the middle of a usable range: raise for
publication, lower for scale"), and now cites the specific text it supersedes
(`hiveplot.py:1831-1832`).

### In-scope tweak: the replace-and-sweep reaches into `examples/`

**Date:** 2026-08-02
**Trigger:** Amendment feasibility check for the notebook scope, which found a second prose site
carrying arithmetic derived from `num_steps=100`.
**Workstream affected:** C, change the defaults (cross-referencing G)
**Change:** "Patterns this replaces" gains `examples/creating_hive_plots_from_dask.ipynb`'s
"## Partition Sizing" cell and a holdout for the symbolic `DaskComputationError` message at
`viz/datashader.py:840`; the `hiveplot.py` citation is corrected `:1827` → `:1824-1825` against the
working branch; C's done-when now states it cannot complete without G's notebook item.

### Deferred follow-up: a distinct curve-resolution default for the fused rasterize-from-ids path

**Date:** 2026-08-02 (re-scoped the same day; see "Superseded framing" below)
**Trigger:** Maintainer ask, explicitly *not* to be formalized now.
**Target:** the grill, as an open design question. Not a workstream, no done-when, nothing to
dispatch. If adopted it becomes a plan of its own: it is net-new user-facing API surface, which this
plan by construction does not have, and that claim under "API usage examples" must keep holding.
**Rationale:** the narrowed question is well-formed where the broad one was not, but the regime it
turns on has not been measured, and whether a path-dependent default is acceptable is a taste call.

**The question, in the maintainer's framing:** once you have made the edges, you have made the edges,
so you cannot work backwards. But there is the datashader-at-scale case where you are *not* saving
the edges. In that case you could have a different default, and that is the case where it matters
most, because that is the scale where blowing up memory is the worry.

So the question is not "should the datashader *backend* use a lower number" but **should the fused
rasterize-from-ids path** (edge subsets holding stored ids with no constructed curves, whose curves
are built during rasterization and immediately discarded, never persisted) **carry its own lower
default.**

Evidence in favour, already measured above: datashader is the **insensitive** backend. Standalone at
1500x1500, even n=8 gives a p99 of 4/255, because density rendering averages over overlapping curves.
It also carries the largest speedup (1.96x at the large scale at n=16 vs 100). The sensitive case is
the sparse vector backends, which is the inversion recorded under "Visual evidence".

**The narrowing dissolves both objections the broad framing carried.**

- Keying off the viz backend was unsound because `self.backend` is mutable after construction
  (`hiveplot.py:3932-3943`, `set_viz_backend`) and read only at render dispatch (`:5027+`), so
  construct-as-datashader → switch → render-as-matplotlib reached this plan's own "silently ugly on
  export" mode through a supported call sequence. That does not apply here: nothing keys off
  `self.backend`, and nothing is decided at construction. The condition is the **path** (ids present,
  curves absent), which is observable at the moment it matters and cannot be invalidated later by a
  backend switch, because the curves it governs never outlive the rasterization that built them.
- A render-time parameter being inert when curves are already persisted was an API-coherence flaw
  under the broad framing. Under the narrow one it **is the scope definition**: the knob exists only
  on the path where it can mean anything. The documented behavior (`viz/datashader.py:338-343`,
  echoed at `:937-941`, "Curves persisted by ``construct_curves`` ... are returned as stored,
  untouched") now reads as the boundary of the feature rather than a contradiction of it.

**Concrete prerequisite, unmeasured.** Any distinct number needs a fused-path memory-vs-`num_steps`
sweep first. The peak-RSS plateau in the Memory table (large datashader route: 2471 MiB at n=100 →
2158 at n=50, then flat at ~2160 for 25 and 16) was measured with `construct_curves` called first,
i.e. on the **persisted** path. The fused path has a different profile: curves are transient and peak
is bounded by the largest chunk under `stream_chunk_threshold`. The regime the maintainer names as
mattering most is therefore exactly the one not yet measured, and the plateau may not hold there. The
sweep must run one isolated process per candidate and must not call `construct_curves` first;
peak-RSS is monotone within a process, so a persisted run preceding a fused run in the same process
would contaminate the fused figure with the persisted one's epoch.

Deliberately **not** folded into Workstream A. A is a matplotlib vector-fidelity check whose evidence
gates the shipped default (Workstream B); this is a datashader memory measurement whose evidence
gates only a deferred question. Bundling them would make A's completion depend on evidence for a
deferred item and would quietly pull that item back into the dispatchable set.

**Coherence wrinkle, for the grill** (also listed under "Alignment (grill)"): a distinct fused-path
default means the same `HivePlot` renders at two different fidelities depending on whether the user
called `construct_curves` first. Defensible, since the paths already differ in what they persist, but
it is the kind of thing that has to be documented deliberately rather than discovered.

Also note, if this is ever taken up: it would put a third spelling of the same concept in front of
users, on top of the `num_steps` / `num_steps_per_edge` inconsistency already recorded under "Naming
audit".

**Superseded framing (kept as record).** This entry was first written the same day as "backend-keyed
or render-time curve resolution for datashader," weighing two mechanisms: keying the constructor
default off `HivePlot.__init__(backend=...)`, and adding a render-time `num_steps` to the datashader
entry points. Both were recorded as problematic, for the two reasons dissolved above. The maintainer
narrowed the question to the fused path, which is why those objections no longer bind; the broad
framing is not the live question and should not be re-argued.

*(2026-09-01: closed by unification; see "In-scope tweak: the fused-path deferred follow-up is closed
by unification" below. This entry stays as the record of the question.)*

### In-scope tweak: spec transcribed (first transcription; the plan predates the convention)

**Date:** 2026-09-01
**Trigger:** Maintainer instruction to amend the plan against the signed spec
(`wiki/wiki/specs/lower-default-curve-resolution.md`, signed 2026-08-31 by Gary Koplik). No
implementation authorized; no workstream dispatches follow this amendment.
**Workstream affected:** none directly; the plan body
**Change:** "Goal" opens with the spec's outcome statement and carries the `Spec:` line; both
call-shape fences land under "API usage examples" with their `Path:` lines; the four maintainer-named
failure modes are the top tier of "Failure modes" (the plan's three seeded modes are the same three in
the maintainer's words, so they were reconciled, not duplicated; the fourth, unclean pass-through, is
new). The spec's `## Plans` list already carried this plan's path, so no backlink was written and the
spec was not touched. "Alignment (grill)" and the adversary placeholder now say what ran at spec stage
and what has not run on this plan.

### In-scope tweak: Workstream B complete, the number is 50

**Date:** 2026-09-01
**Trigger:** The spec settles the number.
**Workstream affected:** B, settle the number
**Change:** blocked on the grill and A → complete by the spec. Justification recorded under "Default
justifications": headroom under the compounding canvas-size and `control_rho_scale` multipliers
(about 0.13 px at the worst tested combination on the default canvas, about 0.35 px at 4000px), zero
measured pixels beyond 8/255 at native resolution, and the whole peak-RSS win captured. 25 rejected
because a plain 4000px export already sits at 0.52 px with no rho fiddling. "Candidate numbers" keeps
its table as the record; its "open for the grill" line is superseded.

### In-scope tweak: Workstream A re-aimed from input to release gate

**Date:** 2026-09-01
**Trigger:** The spec's open question binds vector fidelity as a gate on shipping, not as evidence
for choosing the number.
**Workstream affected:** A, vector-output fidelity check
**Change:** done-when narrows from "sweep 50/25/16 against 100 so the grill can settle the number" to
"50 against 100 at publication zoom, complete before the new default ships in a release; a failure
re-signs the number on the spec". Still runnable now, still no code.

### In-scope tweak: Workstream C widened; the one-constant premise was false

**Date:** 2026-09-01
**Trigger:** Spec drafting found that `_DEFAULT_NUM_STEPS_PER_EDGE` covers `HivePlot` alone. The
three `HivePlotMatrix` constructors (`hiveplot_matrix.py:1107`, `:1501`, `:1982`), `P2CP.build_edges`
(`p2cp.py:231`, assigning its literal over the attribute at `:279`), and two private helpers
(`hiveplot.py:1631`, `:1704`) hardcode 100 and forward it explicitly. Verified on branch 53 today;
the plan's 2026-08-02 sweep list is superseded by the verified list under "Patterns this replaces".
**Workstream affected:** C, change the defaults
**Change:** C collapses every listed site onto the constant. `P2CP.build_edges` becomes
`num_steps: int | None = None` resolving to the underlying hive plot's `num_steps_per_edge`
(api-critic's spec-stage suggestion; keeps the declared-rate semantics and adds no second literal).
The docstring sweep is corrected against the branch: the "Default ``100``" literals are at
`hiveplot.py:537` and `:2812` only; the matrix docstrings at `hiveplot_matrix.py:1219`, `:1615`,
`:2110` state no default at all, and `p2cp.py:246` states no number. The `construct_curves` memory
figure C was to recompute no longer exists (symbolic at `hiveplot.py:1902-1903`), so C's dependency
on G's notebook arithmetic is dissolved and C stands alone.

**Feasibility audit (scope d, a setting on existing objects):** the setting is `num_steps_per_edge`,
already a `BaseHivePlot` attribute initialized from the constant (`hiveplot.py:614`). Its lifecycle
after the collapse: `HivePlot.__init__` takes it from the constant (`:2901`); the `HivePlotMatrix`
constructors forward whatever they were given into each cell's `HivePlot(...)`, so they take the
constant once their own literal goes; `P2CP.build_edges` writes the declared rate over the attribute
(`:279`), which under `None` re-assigns the value already there; every rebuild (`build_hive_plot`,
`connect_adjacent_axes`, `connect_axes`, `construct_curves`, `add_edge_curves_between_axes`) resolves
`None` against the attribute; both datashader entry points resolve their render-time `None` against
it (`viz/datashader.py:427`). `to_json` serializes curves, not the rate, so a round-trip carries the
geometry and not the setting; that is today's behavior, unchanged here. The setting survives every
rebuild path enumerated; no recovery needed.

### In-scope tweak: Workstream D cites the text it supersedes at its current lines

**Date:** 2026-09-01
**Trigger:** The `hiveplot.py:1831-1832` citation drifted.
**Workstream affected:** D, docstrings and the escape hatch
**Change:** the "Larger numbers will result in smoother curves ... but slower rendering" text now
sits at `hiveplot.py:1488-1489` (`connect_axes`), `:1914-1915` (`construct_curves`), `:2338-2339`
(`add_edge_curves_between_axes`), `:2811-2813` (`HivePlot.__init__`), and `p2cp.py:246-247`; the
matrix docstrings at `hiveplot_matrix.py:1219`, `:1615`, `:2110` carry neither a default nor a
direction and get both. D also checks that every docstring pointing a user at raising the number
points at construction or a full rebuild, never at `construct_curves` on a built instance (its own
note at `hiveplot.py:1891-1899` already says that call does nothing there).

### In-scope tweak: Workstream F binds to ADR 0002 and gains the pinned-shape sites

**Date:** 2026-09-01
**Trigger:** Spec "Out of scope", last bullet; and the verified sweep.
**Workstream affected:** F, test and gate fallout
**Change:** done-when adds the two `(101, 2)` asserts (`tests/hiveplot_test.py:1769`,
`tests/p2cp_test.py:686`), the runner literal (`runners/performance/profile_hiveplot_ds.py:201`), a
test for the post-construction escape hatch (assign the attribute, then `build_hive_plot()`), the
ADR 0002 rule (old-default baselines and fixtures are pinned or re-baselined, never compared across
the change), ASV capture at the merge on the canonical host, and the requirement that any performance
figure names its measured path.

### In-scope tweak: Workstream G's arithmetic item retired; no curviness guidance needed

**Date:** 2026-09-01
**Trigger:** Verified sweep (the `808` literals are gone) and the spec's headroom finding.
**Workstream affected:** G, notebook prose for the resolution range
**Change:** G's first done-when item (recompute the 808-bytes-per-edge arithmetic) is retired: the
"## Partition Sizing" cell already states the budget symbolically and pins its worked guideline "at
`num_steps=100`", so it stays true after the change. What remains there is editorial: re-base the
worked guideline at the new default (a 10-million-edge partition holds about 4 GB at 50) or keep the
explicit 100 and say it is no longer the default; record which. G is no longer coupled to C's
release. The candidate item for `customizing_edge_curves.ipynb` (a curviness caveat next to
`control_rho_scale`) is dropped: the rho sweep at n=50 tops out at 0.126 px on the default canvas
and about 0.35 px at 4000px, so there is nothing to warn about. G's remaining items (the
lower-it-at-scale message in `hive_plots_for_large_networks.ipynb`, the raise-it-for-publication
statement where a reader first meets the knob, no llms.txt edit) stand. Status: blocked on the
plan-stage adversary and the grill, no longer on B.

### In-scope tweak: API example 2 withdrawn (silent no-op)

**Date:** 2026-09-01
**Trigger:** Spec drafting caught it.
**Workstream affected:** none (plan body)
**Change:** `hp.construct_curves(num_steps=100)` after a default build skips every subset already
holding curves (`hiveplot.py:1952`; default `persist_curves=True` at `:2902`), so it ran fine and
resampled nothing. Replaced by the spec's second fence (`num_steps_per_edge=100` at construction) and
its `Path:` line naming the per-class rebuild paths and the two look-alike calls that are not the
lever.

### In-scope tweak: the fused-path deferred follow-up is closed by unification

**Date:** 2026-09-01
**Trigger:** Spec call shape, the paragraph after the first fence.
**Workstream affected:** none; closes the 2026-08-02 deferred follow-up and "Alignment (grill)"
items 2 and 3
**Change:** the datashader render-time `num_steps` defaults `None` and resolves to the instance's
`num_steps_per_edge` (`viz/datashader.py:427`; signatures at `:996` and `:1577`; documented at
`:1107` and `:1722`), so the fused rasterize-from-ids path inherits the new default with no site of
its own. A separate lower default for that path would be a second number on top of the one the spec
commits to, and the two-fidelity coherence wrinkle the follow-up recorded is exactly its cost;
nothing is measured or planned for it. The fused-path memory sweep it named as a prerequisite is not
run and not owed.

### Deferred follow-up: `num_steps` vs `num_steps_per_edge` naming unification

**Date:** 2026-09-01
**Trigger:** Spec "Out of scope".
**Target:** GitLab issue #67 (open). Both names shipped in v0.28.0, so unification is a deprecation
cycle in its own right; not this plan.
**Rationale:** the naming audit records the inconsistency; the issue is where it gets worked.

## Holdouts

Expected survivors of the replace-and-sweep grep:

- `src/hiveplotlib/viz/datashader.py:964` (was `:840`): the `DaskComputationError` message states
  the transient per-partition budget symbolically (`partition_rows * (num_steps + 1) * 8`), not as a
  number derived from the default, so it stays correct across the change. Same for the comment at
  `:61`.
- `src/hiveplotlib/hiveplot.py:1902-1903` and `:2817`: the `construct_curves` note and the
  `persist_curves` parameter docstring state curve memory symbolically
  (`num_edges * (num_steps + 1) * 8`); both stay correct. These replaced the "800 MB per 1,000,000
  edges" figure the 2026-08-02 sweep listed.
- `examples/creating_hive_plots_from_dask.ipynb`, "## Partition Sizing": the symbolic formula plus a
  worked guideline pinned explicitly "at `num_steps=100`", arithmetically true after the change.
  Whether to re-base it at 50 is Workstream G's editorial call, not a sweep hit.
- `src/hiveplotlib/utils.py:75`, `:96`, `:254`: `bezier`, `bezier_all`, `bezier_xy_with_nans` keep
  `num_steps=100` by the spec.
- `tests/hiveplot_test.py:57`, `:2476`, `:2611` and `tests/datashader_test.py:31`, `:1827`, `:1946`,
  `:1996`, `:2180`, `:2248`, `:2491`: read `_DEFAULT_NUM_STEPS_PER_EDGE` and follow it.

## Implementation log

- 2026-08-01: Plan opened. Feasibility measured (fidelity, runtime, memory, four figure sets). No
  code changed.
- 2026-08-01: Visual evidence redone at true standalone resolution after the maintainer caught that
  side-by-side subplots undersample each plot ~2.5-3x. Numbers in "Visual evidence" replaced; the
  case for 25 weakened, the case for 50 strengthened. Recorded the sparse-vs-dense inversion.
- 2026-08-02: Amended (maintainer scope additions). Documentation split into D (docstrings) and G
  (notebook prose), both framed as a two-direction range; the datashader-specific default recorded as
  an open design question for the grill, not a workstream. Sweep found a second default-derived
  arithmetic site in `examples/creating_hive_plots_from_dask.ipynb`. Still no code changed.
- 2026-08-02: Amended again (maintainer). Workstream A re-scoped to sweep 50/25/16 rather than "the
  candidate number", making it independent of B and runnable now. The deferred design question
  narrowed from "backend-keyed resolution for datashader" to "a distinct default for the fused
  rasterize-from-ids path", which dissolves both mechanism objections; recorded its unmeasured
  prerequisite (a fused-path memory sweep) and its two-fidelity coherence wrinkle. Still deferred,
  still no workstream.
- 2026-09-01: Amended against the signed spec (first transcription; the plan predates the
  convention). Outcome statement, both call-shape fences with `Path:` lines, and the four-mode top
  tier transcribed. Number settled at 50 (B complete); A re-aimed as a release gate; C widened to
  every literal-100 site after the one-constant premise proved false, with the sweep list re-verified
  on branch 53; D and F citations corrected and F bound to ADR 0002; G's arithmetic item retired
  (both prose sites had gone symbolic); API example 2 withdrawn as a silent no-op; the fused-path
  follow-up closed by unification; naming unification deferred to GitLab #67. No spec edit (the
  backlink was already present). No tracking issue exists for this plan. Still no code changed.
  Next: plan-stage adversary, then grill; nothing dispatches before both.
