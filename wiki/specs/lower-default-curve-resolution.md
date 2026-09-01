# Spec: lower the default curve resolution to 50

## Outcome

A hive plot built the usual way samples each edge at 50 points instead of 100, and the user types nothing new to get it.

The fidelity claim is an analytic bound: how far a 50-segment polyline can sit from the true curve is a property of the axis geometry and the canvas, not of the graph. At 50 it stays around a twentieth of a pixel on the default canvas and under a quarter pixel even at a 4000px export, with headroom left for the canvas-size and `control_rho_scale` multipliers that compound on top. The pixel measurements back this up on the standalone configurations tested: there, no pixel differed from the old default beyond 8/255 at native resolution.

What the user gets for free: rendering 1.2-1.3x faster, growing with scale, persisted curve memory half the size, and the whole peak-memory win on the large datashader route, which plateaus below 50 anyway.

The new default reaches `HivePlot`, `HivePlotMatrix`, and `P2CP` alike, at construction and on every rebuild (`P2CP` takes it at `build_edges`, its build step, rather than at construction). Anyone who wants the old resolution back sets `num_steps_per_edge=100`; the docstrings say so, the way the v0.27.0 datashader `dpi` change did.

The number is 50 and this spec commits to it. Moving it later, in either direction, is a re-sign recorded here, never a quiet adjustment.

## Call shape

```python
# The usual path, unchanged: this exact code gets faster and lighter, with no visible change
from hiveplotlib.datasets import example_hive_plot
from hiveplotlib.viz.matplotlib import hive_plot_viz

hp = example_hive_plot(num_nodes=2_000, num_edges=5_000, seed=7)
fig, ax = hive_plot_viz(hp)
```

Path: `HivePlot.__init__` (here via `example_hive_plot`, which forwards to it) takes `num_steps_per_edge` from the module constant `_DEFAULT_NUM_STEPS_PER_EDGE` in `hiveplot.py`; curves build at construction and `hive_plot_viz` draws them as stored.

Today that constant covers `HivePlot` alone: the `HivePlotMatrix` constructors, `P2CP.build_edges`, and two private helpers in `hiveplot.py` carry their own literal 100s that stomp it. Collapsing those literals onto the one constant is in scope for this change; flipping the constant by itself does not deliver the outcome. The plan carries the site list.

Once collapsed, `construct_curves`, `connect_axes`, and `add_edge_curves_between_axes` (default `num_steps=None`, resolving against the instance's `num_steps_per_edge`) put every rebuild path on the new default through that single constant. The datashader render-time `num_steps`, the fused rasterize-from-ids path included, defaults to `None` and resolves the same way, so every render path inherits the new default with it.

```python
# Publication output: buy the old resolution back at construction
hp = example_hive_plot(num_nodes=2_000, num_edges=5_000, seed=7, num_steps_per_edge=100)
fig, ax = hive_plot_viz(hp)
fig.savefig("figure.pdf")
```

Path: `HivePlot.__init__(num_steps_per_edge=100)`; every rebuild (`build_hive_plot`, `connect_adjacent_axes`) redraws at that value. On an existing plot the supported path is per class: for `HivePlot`, assign `num_steps_per_edge` and call `build_hive_plot()`; for `P2CP`, pass `num_steps` to `build_edges(...)`, its lever, at build time rather than construction.

Two calls that look like this escape hatch and are not: `construct_curves(num_steps=100)` on an already-built instance skips every subset that holds curves, so it runs fine and resamples nothing (it is the build call for instances holding ids alone, e.g. `persist_curves=False`); and the datashader render-time `num_steps` kwarg raises peak memory on a curve-persisted instance rather than replacing what is stored. Construction or a full rebuild is the lever.

## Out of scope

- The low-level `utils.bezier*` primitives keep 100. Nobody should call them directly, and anyone who does is exactly the user who knows the knob exists.
- The `num_steps` vs `num_steps_per_edge` naming inconsistency. Recorded in the plan; renaming is a deprecation in its own right.
- Any new API surface. Parameters, names, and types are unchanged; only the default value moves.
- Old-default performance baselines and fixtures are not equivalence-comparable to the new default; per ADR 0002 they get pinned or re-baselined, and any performance claim ships only through the regression harness.

## Alternatives considered

Wording a re-sign displaces from the outcome statement moves here rather than being deleted, on a dated "re-sign, narrowed from:" line. Those lines are append-only: add one as each re-sign happens, and never edit or drop an earlier one. Revising the rest of this list is normal.

- Adaptive per-curve sampling: measured and rejected (archived plan `adaptive-curve-sampling`); only 1.4-2.3x over the best uniform constant, not worth the machinery.
- 25 or 16 as the default: rejected on headroom. Both look fine as measured, but neither has margin for the canvas-size and `control_rho_scale` multipliers users legitimately turn, and the corrected standalone measurement weakened 25's case.
- Warning machinery ahead of the change: rejected. This follows the v0.27.0 datashader `dpi` pole: silent change, CHANGELOG entry, docstrings tell you how to buy fidelity back.

## Failure modes

Maintainer-named (failure-mode wave, 2026-08-31).

- **The compromise is ugly at defaults**: faceting the maintainer's eye catches in a gallery figure, even if a pixel metric calls it equivalent.
- **Silently ugly on export**: unlike `dpi`, nobody sees `num_steps` change. Someone saves a publication figure and gets subtly polygonal curves with no signal that a knob exists.
- **Equivalence measured only where it was easy**: everything is measured through rasters; vector PDF/SVG at arbitrary zoom is the case a raster metric structurally cannot see.
- **The parameter pass-through is not clean in practice**: if changing the value on any of the higher-level classes does not happen cleanly (setting it at construction, or changing it on an existing plot), the scope expands to fix the pass-through rather than shipping a default nobody can move.

## Open questions

- Vector-output fidelity: matplotlib PDF/SVG at publication zoom has not been compared against n=100; every equivalence number so far is a raster. That comparison at 50 vs 100 completes before the new default ships in a release, and a failure there re-signs the number rather than quietly adjusting it.

## Sign-off

Intent extraction: brief mode ran; failure-mode wave ran.

No agent ever signs a spec change, autonomous runs included. An agent whose write can change what this spec promises marks it on an "Unapproved modifications:" line here; only the maintainer clears that marker, by re-signing. Resolving an open question above is such a change. The full governance rules live in the harness CLAUDE.md under "Specs and plans".

The fence below is append-only: one line per event, in the form `Re-signed YYYY-MM-DD by <name(s)>: <one paragraph on what changed and why>.` or, once the outcome statement above is true, `Closed YYYY-MM-DD by <name(s)>.`

```
Signed 2026-08-31 by Gary Koplik.
```

## Plans

A plan closing never closes this spec. The last plan listed here closing is the trigger to check the outcome statement above, which stays the arbiter; plan-end QA runs that check.

- wiki/wiki/plans/lower-default-curve-resolution.md
