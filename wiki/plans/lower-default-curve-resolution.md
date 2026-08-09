# Plan: lower the default curve resolution (`num_steps` / `num_steps_per_edge`)

**Status: exploring.** Feasibility measured 2026-08-01 (numbers and figures below). No code changed.
No code workstream is dispatchable yet: the grill has not run, the adversary has not read it, and the
target number is an open decision. Workstream A (evidence only, no code) is the exception and can run
in parallel with both.

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
Not yet run — recommended before dispatch. Three open decisions for the agenda,
none of which should be settled by the model:
  1. The target number (see "Candidate numbers"; the plan recommends 50 on
     headroom grounds, deliberately not settled).
  2. Whether the fused rasterize-from-ids path should carry its own lower
     default, given that it never persists the curves it builds (see the
     deferred follow-up on that path under "Plan amendments"). Gated on an
     unmeasured fused-path memory sweep, recorded there.
  3. If 2 is taken up: whether it is acceptable that the same `HivePlot`
     renders at two fidelities depending on whether `construct_curves` was
     called first. The paths already differ in what they persist, so this is
     defensible, but it needs deliberate documentation rather than discovery.
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

**Not yet shown, and needed before sign-off:** vector output. Everything above is a raster. A
matplotlib PDF/SVG at high zoom is the case where faceting would survive that a raster metric cannot
see.

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

Open for the grill. Do not treat the recommendation as settled.

## Patterns this replaces

All literal `100` curve-resolution defaults:

- `src/hiveplotlib/utils.py:75`, `:96`, `:254` — `bezier`, `bezier_all`, `bezier_xy_with_nans`
- `src/hiveplotlib/hiveplot.py:1401`, `:1570`, `:1643`, `:1810`, `:2151` — `num_steps`
- `src/hiveplotlib/hiveplot.py:2650` — `num_steps_per_edge`
- `src/hiveplotlib/hiveplot_matrix.py:1068`, `:1453`, `:1922` — `num_steps_per_edge`
- `src/hiveplotlib/p2cp.py:231` — `num_steps`

Prose carrying arithmetic derived from the current default, all of which becomes wrong on the same
commit (verified 2026-08-02; these are correctness fixes, not discretionary rewording):

- `src/hiveplotlib/hiveplot.py:1824-1825` — `construct_curves` docstring citing "roughly 800 MB per
  1,000,000 edges at the default ``num_steps=100``". (Earlier drafts of this plan cited `:1827`;
  corrected against the working branch.)
- `examples/creating_hive_plots_from_dask.ipynb`, the "## Partition Sizing" markdown cell — "about
  808 bytes per edge at the default `num_steps=100`", the `n * 808` and
  `thread count * largest partition's row count * 808 bytes` formulas, and the worked "a single
  10-million-edge partition holds about 8 GB of curves" example. Owned by Workstream G; must ship
  with Workstream C.

One expected survivor of the sweep, recorded under Holdouts below: the `DaskComputationError` message
in `viz/datashader.py` states the same budget symbolically and needs no edit.

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

**Status:** not started. Runnable now, in parallel with the adversary and the grill (re-scoped
2026-08-02; see Amendments).
**Files:** none (investigation)
**Done when:** matplotlib PDF/SVG output at **each of 50, 25, and 16** is compared against n=100 at
publication zoom, and each either clears the maintainer's eye or produces a documented
counterexample. Sweeping the shortlist rather than a single number is what makes A independent of
Workstream B instead of circular with it: A produces the evidence the grill needs to settle the
number, and settles nothing itself. This is the one evidence gap remaining for the shipped default;
everything else is measured.

### Workstream B: settle the number

**Status:** blocked on the grill and on Workstream A
**Done when:** the number is chosen by the maintainer and recorded in "Default justifications" with
its justification.

### Workstream C: change the defaults

**Status:** not started
**Files:** the "Patterns this replaces" list
**Done when:** every listed default moves, `hiveplot.py:1824-1825`'s memory figure is recomputed, and
the replace-and-sweep grep finds no surviving literal. C is not complete on its own: the same sweep
reaches `examples/creating_hive_plots_from_dask.ipynb`'s "## Partition Sizing" arithmetic, which is
notebook-author's surface (Workstream G) and must land in the same change. See Amendment 2026-08-02.

### Workstream D: docstrings and the escape hatch

**Status:** not started
**Scope:** docstrings only. Notebook prose split to Workstream G (see Amendment 2026-08-02).
**Files:** `src/hiveplotlib/hiveplot.py`, `hiveplot_matrix.py`, `p2cp.py`
**Done when:** the `num_steps` / `num_steps_per_edge` docstrings present the default as the middle of
a usable range rather than a ceiling, naming both directions: raise it when a figure is going to
publication (the v0.27.0 `dpi` pattern), lower it further when scaling to large networks. Today's
text (`hiveplot.py:1831-1832`, "Larger numbers will result in smoother curves when plotting later,
but slower rendering") states the trade without telling anyone when to move it either way.

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

## Holdouts

Expected survivors of the replace-and-sweep grep:

- `src/hiveplotlib/viz/datashader.py:840`: the `DaskComputationError` message states the transient
  per-partition budget symbolically (`partition_rows * (num_steps + 1) * 8`), not as a number derived
  from the default, so it stays correct across the change. Do not "fix" it to match the two prose
  sites that do carry arithmetic.

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
