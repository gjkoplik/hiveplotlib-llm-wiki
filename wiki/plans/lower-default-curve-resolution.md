# Plan: lower the default curve resolution (`num_steps` / `num_steps_per_edge`)

**Status: exploring.** Feasibility measured 2026-08-01 (numbers and figures below). No code changed.
Not dispatchable yet: the grill has not run, the adversary has not read it, and the target number is
an open decision.

## Goal

Every hive plot is drawn by sampling each quadratic Bézier edge at `num_steps` points; the default
has been 100 since the beginning. Measurement shows 100 is ~4-6x more than the default canvas can
resolve. Lowering it makes out-of-the-box hive plots build faster, render faster, and hold
substantially less memory, with no visible change at default output sizes. Users who need
publication-crisp output raise the number, documented the way the datashader `dpi` change was.

This is a defaults-and-documentation change, not a scaling feature. It touches every backend and
every existing user's output, which is why it is scoped on its own rather than riding along with the
branch-53 scaling work.

**Brief-mode gate:** knowingly skipped. The brief arrived already specified (maintainer named the
precedent, the concern, and the evidence he wants), so there was nothing underdetermined to
interview for.

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
Not yet run — recommended before dispatch. The target number is a taste call that
should not be settled by the model.
```

## Failure modes

Seeded from the maintainer's stated concern; the rest awaits the grill's failure-mode wave.

- **The compromise is ugly at defaults** (maintainer, in his words: "I do wanna be particular that
  the compromise isn't so ugly, particularly for the defaults"). Faceting that a maintainer's eye
  catches in a gallery figure even if a pixel metric calls it equivalent.
- **Silently ugly on export.** Unlike `dpi`, a user does not *see* `num_steps` change. Someone saves
  a publication figure at high dpi and gets subtly polygonal curves with no signal that a knob
  exists. This is the asymmetry with the dpi precedent and it is the main argument for a
  conservative number.
- **Equivalence measured only where it was easy.** Everything below is measured through the
  datashader raster and matplotlib Agg. Vector export (PDF/SVG at arbitrary zoom) is the case a
  raster metric structurally cannot see.
- *(pending: further modes from the grill wave)*

## Prior ADRs / design docs

- None as an ADR. The governing precedent is the v0.27.0 datashader `dpi` change, recorded only in
  `CHANGELOG.rst` (see "Precedent this follows" above).
- `wiki/wiki/plans/archived/adaptive-curve-sampling.md` — the measurement that produced this finding,
  and the record of why *adaptive* per-curve sampling was rejected in favour of a lower constant.

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

Figures in the session scratchpad (not committed):

`.../scratchpad/figs/`

- `A_full_view.png` — dense 2k/5k plot, full view, n ∈ {100, 50, 25, 16, 8}. All five read as the
  same figure; density hides everything. Weak test, included for completeness.
- `E_sparse_full.png` — 60 nodes / 90 edges, opaque 1.4pt lines, same candidates. **The aesthetic
  call.** 100/50/25/16 are indistinguishable; n=8 shows slight angularity on the long sweeping arcs.
- `F_worst_curve.png` — **the decisive geometry test.** The single worst curve in the plot, every
  candidate drawn over the true curve with sample points marked, plus zooms at 82x and 410x. At an
  18px-wide crop, n=8 is clearly off the true curve and 50/25/16 are not. At a 4px-wide crop, 25 and
  16 separate visibly and 50 still hugs it.
- `G_datashader_delta.png` — **the at-scale test.** 25k/250k rendered through the real datashader
  entry point, plus per-pixel difference maps against n=100. Differences are confined to a
  one-pixel-wide rim on the outer envelope and a crescent at the inner hole. Mean |Δ| 0.46-0.56 of
  255; p99 of 3-4; only 0.30-0.37% of pixels differ by more than 8/255.

An earlier version of the datashader panel appeared to show real differences in silhouette and
shading. That was an artifact of compositing into subplots, not a real effect; `G` supersedes it and
is the measurement to trust.

**Not yet shown, and needed before sign-off:** vector output. Every figure above is a raster. A
matplotlib PDF/SVG at high zoom is the case where faceting would survive that a raster metric cannot
see.

## Candidate numbers

| | fidelity @ default | fidelity @ 4000px | speed (large ds) | curve memory | read |
|---|---|---|---|---|---|
| **50** | 0.046 px | 0.12 px | 1.32x | 2.0x less | conservative; safe at any realistic export size |
| **25** | 0.194 px | 0.52 px | 1.64x | 3.9x less | aggressive; clean at default, thin margin on high-dpi export |
| 16 | 0.496 px | 1.32 px | 1.96x | 5.9x less | meets the half-pixel bound at default only, no margin |

**Recommendation: 50.** It is invisible in every test run, keeps sub-quarter-pixel fidelity even at a
4000px export, captures the whole peak-RSS win on the large route (which plateaus below 50 anyway),
and needs no caveat about high-dpi output. 25 is defensible if we are willing to lean on
documentation the way the dpi change did, but it trades away the "silently ugly on export" safety
margin for speed the plateau says we partly cannot bank.

Open for the grill. Do not treat the recommendation as settled.

## Patterns this replaces

All literal `100` curve-resolution defaults:

- `src/hiveplotlib/utils.py:75`, `:96`, `:254` — `bezier`, `bezier_all`, `bezier_xy_with_nans`
- `src/hiveplotlib/hiveplot.py:1401`, `:1570`, `:1643`, `:1810`, `:2151` — `num_steps`
- `src/hiveplotlib/hiveplot.py:2650` — `num_steps_per_edge`
- `src/hiveplotlib/hiveplot_matrix.py:1068`, `:1453`, `:1922` — `num_steps_per_edge`
- `src/hiveplotlib/p2cp.py:231` — `num_steps`
- `src/hiveplotlib/hiveplot.py:1827` — docstring citing "roughly 800 MB per 1,000,000 edges at the
  default `num_steps=100`", which becomes wrong on the same commit

Open question for the grill: should the low-level `utils.bezier*` helpers move too, or keep 100 and
let only the user-facing entry points change? They are public API but are the primitive, not the
workflow.

## Default justifications

- `num_steps=<TBD>`: the default canvas cannot resolve finer, measured; users raise it for
  publication output exactly as they raise `dpi`.

## Naming audit

- Existing inconsistency surfaced, not introduced by this work: the same concept is `num_steps` on
  `construct_curves` / `add_edge_curves_between_axes` / `P2CP` and `num_steps_per_edge` on the
  `HivePlot` / `HivePlotMatrix` constructors. Renaming is out of scope here (it is a deprecation in
  its own right) but this plan is the natural place to record it.
- No new names introduced.

## API usage examples

No API surface change: parameters, names, and types are unchanged. Only the default value moves.

```python
# Example 1: the default path, which is what changes
# Example data:
from hiveplotlib.datasets import example_hive_plot

hp = example_hive_plot(
    num_nodes=2_000,
    num_edges=5_000,
    partition_data_column="low",
    labels=["A", "B", "C"],
    cutoffs=3,
    sorting_variables="low",
    seed=7,
)

# Call site: unchanged, but each edge is now sampled at the new default
from hiveplotlib.viz.matplotlib import hive_plot_viz

fig, ax = hive_plot_viz(hp)

# Example 2: buying fidelity back for a publication figure (the documented escape hatch)
hp.construct_curves(num_steps=100)
fig, ax = hive_plot_viz(hp)
```

## Adversary review

### Adversary's challenge (planning mode)

```
Pending — invoke the adversary in planning mode (cold, before grill-me).
```

### Adversary — post-implementation

```
Pending.
```

## Viz review

```
Pending — viz-critic should review the sign-off figures and any regenerated gallery
figures against the viz-quality-bar skill.
```

## Workstreams

### Workstream A: vector-output fidelity check

**Status:** not started
**Files:** none (investigation)
**Done when:** matplotlib PDF/SVG output at the candidate number is compared against n=100 at
publication zoom, and either clears the maintainer's eye or produces a documented counterexample.
This is the one evidence gap remaining; everything else is measured.

### Workstream B: settle the number

**Status:** blocked on the grill and on Workstream A
**Done when:** the number is chosen by the maintainer and recorded in "Default justifications" with
its justification.

### Workstream C: change the defaults

**Status:** not started
**Files:** the "Patterns this replaces" list
**Done when:** every listed default moves, `hiveplot.py:1827`'s memory figure is recomputed, and the
replace-and-sweep grep finds no surviving literal.

### Workstream D: docstrings and the escape hatch

**Status:** not started
**Files:** `src/hiveplotlib/hiveplot.py`, `hiveplot_matrix.py`, `p2cp.py`, docs prose
**Done when:** the `num_steps` / `num_steps_per_edge` docstrings explain the explore-vs-publish trade
and name the way to raise it, matching how the v0.27.0 `dpi` change documented the same trade.

### Workstream E: changelog and release framing

**Status:** not started
**Files:** `CHANGELOG.rst`, possibly the deprecation-policy plan
**Done when:** the entry states plainly that existing plots change slightly, and the
deprecation-policy question (does a default change of this kind warrant more than a changelog entry?)
is either answered or explicitly deferred.

### Workstream F: test and gate fallout

**Status:** not started
**Files:** `tests/`, `runners/performance/`, `benchmarks/`
**Done when:** any test asserting curve shapes at the old default is updated; the raster equivalence
gate's tolerance question is resolved; ASV scenarios are re-captured so the history step attributes
to this change rather than to the next unrelated merge.

## Implementation log

- 2026-08-01: Plan opened. Feasibility measured (fidelity, runtime, memory, four figure sets). No
  code changed.
